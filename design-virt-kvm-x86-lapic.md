---
title: "Tier-3: arch/x86/kvm/lapic.c — In-kernel LAPIC + x2APIC + posted-interrupts wiring"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The in-kernel local APIC (Advanced Programmable Interrupt Controller) emulation for KVM x86 — every guest interrupt injection (IPI between vCPUs, HW MSI/MSI-X from passed-through PCI device, virtual timer tick, software-generated NMI/SMI) routes through here. Critical KVM hot path: avoids the ~10× slower userspace-LAPIC mode (where every interrupt would VMexit to userspace qemu) by handling APIC register access + IPI delivery + EOI in-kernel.

Supports: legacy xAPIC mode (mmio @ 0xfee00000), x2APIC mode (msr-based), Hyper-V extensions (auto-EOI), AVIC (AMD Advanced Virtual Interrupt Controller for direct-IPI-no-vmexit), Intel posted-interrupt descriptors (PI for direct-MSI-injection-no-vmexit), TSC-deadline timer mode, save/restore via `KVM_GET_LAPIC` / `KVM_SET_LAPIC`.

This Tier-3 covers `arch/x86/kvm/lapic.c` (~3600 lines) + `lapic.h` + `irq.c` (per-vCPU interrupt-pending tracking).

### Acceptance Criteria

- [ ] AC-1: qemu KVM-x86 boots Linux guest with default in-kernel APIC; guest sees IPIs delivered (verified via `cat /proc/interrupts | grep IPI`).
- [ ] AC-2: x2APIC test: guest with `x2apic` enabled (read CPUID.0x1F.ECX bit 21) → MSR-based APIC access works; xAPIC MMIO no longer trapped.
- [ ] AC-3: TSC-deadline timer test: guest with `clocksource=tsc` + tsc-deadline LAPIC timer → high-resolution sleep accuracy < 1µs jitter.
- [ ] AC-4: AVIC test on AMD HW: guest IPI between vCPUs delivered without vmexit (verified via `perf kvm stat` showing 0 EXIT_REASON_IO_INSTRUCTION for ICR write).
- [ ] AC-5: Posted-Interrupt test on Intel HW: vfio-pci passthrough device's MSI delivered to guest vCPU without vmexit (verified via `perf kvm stat` showing 0 EXIT_REASON_EXTERNAL_INTERRUPT for that vector).
- [ ] AC-6: Live-migration test: KVM_GET_LAPIC + KVM_SET_LAPIC round-trip preserves all 1024 bytes; migrated guest resumes interrupt delivery correctly.
- [ ] AC-7: Lowest-priority delivery test: cluster of vCPUs with different TPR values → IRR delivered to vCPU with lowest TPR (within cluster).
- [ ] AC-8: kvm-unit-tests `apic_timer` + `tsc_adjust_timer` pass.

### Architecture

`Lapic` lives in `kernel::kvm::x86::Lapic`:

```
struct Lapic {
  refcount: Refcount,
  vcpu: Arc<Vcpu>,
  base_address: u64,                 // typically 0xfee00000
  regs: KBox<[u8; 4096]>,            // mmio'd register space
  apic_timer: HrTimer,
  pending_events: AtomicU32,
  vapic_addr: AtomicU64,             // virtual-APIC-page guest gpa
  vapic_cache: AtomicPtr<u8>,        // host-mapped vapic page
  isr_count: AtomicI8,
  highest_isr_cache: AtomicI8,
  apic_hw_disabled: AtomicBool,
  apic_sw_disabled: AtomicBool,
  divide_count: u32,
  lapic_timer: KBox<KvmTimer>,
  pending_eoi: AtomicI8,
}

struct LapicIrq {
  vector: u8,
  delivery_mode: u8,                  // FIXED / LOWEST / SMI / NMI / INIT / SIPI / EXTINT
  dest_mode: u8,                      // PHYSICAL / LOGICAL
  level: u8,
  trig_mode: u8,                      // EDGE / LEVEL
  shorthand: u8,                      // NO_SHORTHAND / SELF / ALL / ALL_BUT_SELF
  dest_id: u32,
  msi_redir_hint: bool,
}
```

Per-vCPU LAPIC create `Lapic::create(vcpu)`:
1. Allocate Lapic struct.
2. Allocate 4 KB MMIO register page.
3. `Lapic::reset(lapic, false)` → init register defaults (ID, VER, SVR, etc.).
4. Initialize hrtimer for LAPIC timer.

Per-vCPU LAPIC reset `Lapic::reset(lapic, init_event)`:
1. Set ID, VER from per-vCPU vCPU id.
2. SVR = 0xff (all-vectors-allowed default).
3. TPR = 0.
4. APR = 0, PPR = 0.
5. ESR = 0.
6. LVT entries set to MASKED (default).
7. Clear ICR + ICR2.
8. Clear ISR + IRR + TMR.

xAPIC MMIO trap path (`apic_mmio_write` / `apic_mmio_read`):
- Per-register-offset dispatch for each access.
- Writes to ICR low: extract dest_field, delivery_mode, level, trig_mode, shorthand → call `Lapic::send_ipi`.
- Writes to TPR: re-evaluate pending interrupts (lower TPR may unmask previously-suppressed).
- Writes to EOI: call `Lapic::set_eoi(vcpu)`.
- Writes to LVT_TIMER / TMR / DCR: re-arm timer hrtimer.

x2APIC MSR access path (`x2apic_msr_write` / `_read`):
- Per-MSR-offset (0x800 + register_index) dispatch.
- Faster than xAPIC because MSR access is faster than MMIO + no instruction emulation needed for MSR access.

`Lapic::send_ipi(apic, icr_low, icr_high)`:
1. Build `LapicIrq` from ICR fields.
2. `Vm::irq_delivery_to_apic(kvm, &this_apic, &irq, dest_map)`:
   - For each vCPU matching destination: `Lapic::set_irq(target_vcpu, &irq)` → set vector bit in IRR.
   - For LOWEST_PRIORITY: walk all matching vCPUs, pick one with lowest PPR (TPR-related) via `kvm_apic_compare_prio`.
3. Wake target vCPU (kvm_vcpu_kick) so VMENTRY picks up new IRR.

`Lapic::set_irq(vcpu, irq, dest_map)`:
1. If apic disabled: no-op.
2. Set bit `irq.vector` in IRR.
3. If trig_mode == LEVEL: set TMR bit (level-triggered).
4. Mark vcpu->apicv_active.
5. Wake vCPU.

VMENTRY interrupt-injection path:
1. `Lapic::has_interrupt(vcpu)` returns highest-pending vector if PPR < vector.
2. `Lapic::get_apic_interrupt(vcpu)` dequeues from IRR + sets in ISR.
3. Inject vector via VMX RFLAGS.IF + VM-Entry-Interruption-Information field (or SVM equivalent).

Posted-Interrupt mode (Intel):
1. Per-vCPU PID (Posted-Interrupt Descriptor) in physical RAM.
2. PID has 256-bit PIR + ON (Outstanding Notification) bit + SN (Suppress Notification) bit.
3. HW IRTE programmed to write to PID on MSI from PCI device.
4. ASIC posts interrupts directly (writes PIR bit + sets ON if SN clear).
5. On next VMENTRY: HW reads PID.PIR + clears it + ORs into VPID's IRR (no software intervention).

AVIC mode (AMD):
1. Per-vCPU AVIC backing page in nested-pgtable.
2. IPI from one vCPU to another → AVIC HW directly delivers (writes target's backing page).
3. No vmexit on common case.
4. Only vmexits for: cross-physical-CPU IPIs, NMI, SMI, x2APIC-access (legacy mode).

LAPIC timer hrtimer callback:
1. On expire: `Lapic::local_deliver(apic, APIC_LVTT)` → set IRR bit for LVTT vector.
2. Wake vCPU.
3. If TSC-deadline mode: re-arm based on guest's TSC-deadline MSR value translated to host TSC via per-vCPU TSC offset.

### Out of Scope

- IOAPIC + PIC + PIT emulation (covered in `virt/kvm/x86-ioapic.md` future Tier-3)
- VMX vmenter/vmexit (covered in `virt/kvm/vmx-core.md` future Tier-3)
- SVM vmrun/vmexit (covered in `virt/kvm/svm-core.md` future Tier-3)
- AVIC details (covered in `virt/kvm/svm-avic.md` future Tier-3)
- Posted-Interrupt details (covered in `virt/kvm/vmx-posted-intr.md` future Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_lapic` | per-vCPU LAPIC control block | `kernel::kvm::x86::Lapic` |
| `struct kvm_lapic_irq` | per-IRQ delivery descriptor | `kernel::kvm::x86::LapicIrq` |
| `kvm_create_lapic(vcpu, ...)` | per-vCPU LAPIC alloc | `Lapic::create` |
| `kvm_free_lapic(vcpu)` | per-vCPU LAPIC release | `Lapic::free` (Drop) |
| `kvm_lapic_reset(vcpu, init_event)` | per-vCPU LAPIC reset (INIT or RESET event) | `Lapic::reset` |
| `kvm_apic_post_state_restore(vcpu, s)` | post-load state apply (after KVM_SET_LAPIC) | `Lapic::post_state_restore` |
| `kvm_lapic_get_cr8(vcpu)` / `_set_cr8` | TPR/CR8 access | `Lapic::get_cr8` / `_set_cr8` |
| `kvm_apic_update_apicv(vcpu)` | per-vCPU APICv (AVIC/PI) enable/disable | `Lapic::update_apicv` |
| `kvm_lapic_set_eoi(vcpu)` | software EOI | `Lapic::set_eoi` |
| `kvm_apic_set_state(vcpu, s)` / `_get_state(vcpu, s)` | KVM_SET/GET_LAPIC ioctl handler | `Lapic::set_state` / `_get_state` |
| `kvm_apic_local_deliver(apic, lvt_type)` | local LVT (timer / thermal / perfmon / lint0/1 / error) deliver | `Lapic::local_deliver` |
| `kvm_apic_pending_eoi(vcpu, vector)` | check if vector has pending EOI | `Lapic::pending_eoi` |
| `kvm_apic_match_dest(vcpu, source, shorthand, dest, mode)` | per-IPI destination match | `Lapic::match_dest` |
| `kvm_apic_send_ipi(apic, icr_low, icr_high)` | IPI delivery via ICR write | `Lapic::send_ipi` |
| `kvm_irq_delivery_to_apic(kvm, src, irq, &dest_map)` | top-level IRQ routing to vCPU's APIC | `Vm::irq_delivery_to_apic` |
| `kvm_irq_delivery_to_apic_fast(kvm, src, irq, &r, &dest_map)` | fast-path delivery (hash lookup) | `Vm::irq_delivery_to_apic_fast` |
| `kvm_apic_set_irq(vcpu, irq, &dest_map)` | per-vCPU IRQ inject (set pending in IRR) | `Lapic::set_irq` |
| `kvm_apic_compare_prio(vcpu1, vcpu2)` | per-vCPU priority compare for lowest-priority delivery | `Vcpu::lapic_compare_prio` |
| `kvm_apic_has_interrupt(vcpu)` | check if vCPU has pending interrupt | `Lapic::has_interrupt` |
| `kvm_apic_accept_pic_intr(vcpu)` | check if vCPU accepting PIC interrupt | `Lapic::accept_pic_intr` |
| `kvm_get_apic_interrupt(vcpu)` | dequeue highest-pending interrupt for delivery | `Lapic::get_apic_interrupt` |
| `__kvm_set_lapic(vcpu, ...)` / `__kvm_apic_update_irr(...)` | per-vCPU IRR update | `Lapic::__set_irr` |
| `kvm_lapic_expired_hv_timer(vcpu)` | hyperv-stimer expired callback | `Lapic::expired_hv_timer` |
| `start_apic_timer(apic)` / `cancel_apic_timer(apic)` | hrtimer-driven timer | `Lapic::start_timer` / `_cancel_timer` |
| `kvm_apic_load_xapic_msr(vcpu, msr_info)` / `_store_xapic_msr(...)` | xAPIC MMIO trap | `Lapic::load_xapic_msr` / `_store_xapic_msr` |
| `kvm_x2apic_msr_write(vcpu, msr, data)` / `_read(vcpu, msr, &data)` | x2APIC MSR access | `Lapic::x2apic_write_msr` / `_read_msr` |

### compatibility contract

REQ-1: Per-vCPU LAPIC emulates per-Intel-SDM:
- ID, VER, TPR, APR, PPR, EOI, RRD, LDR, DFR, SVR, ISR (256 bits), TMR (256 bits), IRR (256 bits), ESR, ICR (low/high), LVT (Timer / Thermal / PerfMon / LINT0 / LINT1 / Error), ICR Timer, CCR Timer, DCR Timer.
- xAPIC MMIO range 0xfee00000-0xfee00fff.
- x2APIC MSR range 0x800-0x83f (mode + register layout per SDM).

REQ-2: IPI delivery: ICR write triggers IPI to destination(s) per shorthand (NoShortHand / SelfOnly / All / AllButSelf) + delivery mode (Fixed / LowestPriority / SMI / NMI / INIT / SIPI / ExtINT) + dest mode (Physical / Logical) + dest field.

REQ-3: Per-vCPU IRR (Interrupt Request Register) tracks pending interrupts; ISR (In-Service Register) tracks currently-being-serviced; both 256-bit per-vector bitmaps.

REQ-4: Per-vCPU TPR (Task Priority Register) controls minimum-priority interrupt delivery; updates trigger immediate re-evaluation of pending IRR.

REQ-5: Auto-EOI mode (Hyper-V extension): no software EOI required for level-triggered interrupts; reduces vmexits for guest doing many EOIs.

REQ-6: TSC-deadline timer mode: per-vCPU absolute-TSC-value timer; expires when guest TSC reaches deadline; backed by host hrtimer.

REQ-7: AVIC (AMD): when enabled, IPIs between vCPUs delivered HW-direct via AVIC backing-page (no vmexit); per-vCPU AVIC backing page in physical-domain page table.

REQ-8: Intel posted-interrupts (PI): per-vCPU posted-interrupt descriptor (PID); HW MSI from PCI device routed via IRTE → ASIC writes to PID's PIR (Posted Interrupt Request) bitmap → on next VMENTRY, vCPU's APIC sees pending interrupts without vmexit.

REQ-9: KVM_GET_LAPIC / KVM_SET_LAPIC ioctl wire format byte-identical: 1024-byte snapshot of LAPIC register space; used by qemu live-migration.

REQ-10: KVM_IRQFD integration: eventfd write triggers IRQ injection via this LAPIC code (cross-ref `virt/kvm/eventfd.md` future Tier-3).

REQ-11: GSI routing table maps userspace-defined GSIs to per-vCPU vector + delivery-mode + dest; consumed by KVM_IRQFD + KVM_SIGNAL_MSI.

REQ-12: APICv (auto-Posted-Interrupt + Virtual-APIC-Page) mode toggles per-vCPU based on CPU support + KVM_CAP_APICV cmdline.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lapic_no_uaf` | UAF | `Arc<Lapic>` outlives all in-flight set_irq + timer-fire; vCPU destroy waits for refcount==0. |
| `irr_isr_no_oob` | OOB | per-vector bitmap operations bounded by 256 vectors (8 dwords); per-bit ops bounds-checked. |
| `vector_priority_consistent` | INVARIANT | PPR (Processor Priority Register) computed correctly from TPR + highest-ISR; never delivers vector below TPR. |
| `pid_no_uaf` | UAF | per-vCPU PID page lifetime tied to vCPU lifetime; HW write to freed page impossible. |

### Layer 2: TLA+

`models/kvm/lapic_injection.tla` (parent-declared): proves in-kernel APIC interrupt injection + EOI ordering matches `KVM_GET_LAPIC` snapshot semantics; lowest-priority delivery picks one and only one destination.
`models/kvm/posted_intr.tla` (parent-declared): proves PID descriptor PIR/SN/ON bits race-free under concurrent IPI from host + remote vCPU.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lapic::set_irq` post: vector bit set in target IRR; vCPU kicked if delivery_mode != EXTINT | `Lapic::set_irq` |
| `Lapic::get_apic_interrupt` post: returned vector (if any) cleared from IRR + set in ISR; PPR updated | `Lapic::get_apic_interrupt` |
| `KVM_GET_LAPIC` post: returned 1024-byte snapshot fully reflects current state; KVM_SET_LAPIC restores identically | `Lapic::get_state` / `_set_state` |
| Posted-interrupt: PIR bit set by HW always observable on next VMENTRY (per-spec) | per-arch HW contract |

### Layer 4: Verus/Creusot functional

`Lapic::send_ipi(src, irq) → Lapic::set_irq(dst, irq) → vCPU kicked → VMENTRY → guest sees vector` round-trip equivalence: any IPI sent in guest is eventually delivered to target vCPU's IDT entry.

### hardening

(Inherits row-1 features from `virt/kvm/00-overview.md` § Hardening.)

lapic-specific reinforcement:

- **Per-vCPU LAPIC refcount saturating** — defense against ref-count-overflow.
- **xAPIC MMIO range fixed at 0xfee00000** — IOMMU reserved-region (cross-ref `iommu-core.md`); defense against MSI overlap.
- **PID per-vCPU page in physical-domain pgtable** — programmed via IRTE; defense against guest writing to PID directly (would require guest-physical-to-host-physical mapping which it doesn't have).
- **APICv enable/disable via dynamic apicv_active flag** — defense against runtime APICv-state inconsistency between vCPUs.
- **TPR write triggers re-evaluation** under per-vCPU mutex; defense against TPR + IRR race exposing vector to guest before TPR check.
- **Lowest-priority destination tie-break deterministic** — first vCPU in vCPU-list with min PPR; defense against non-determinism causing live-migration divergence.
- **Per-vCPU LAPIC state validated at KVM_SET_LAPIC** — register fields range-checked; reserved bits zeroed; defense against malformed userspace state corrupting in-kernel APIC.
- **TSC-deadline + TSC-offset coordination** — guest TSC-deadline translated to host hrtimer expiration via per-vCPU TSC offset; defense against TSC-skew confusing timer expiry.

