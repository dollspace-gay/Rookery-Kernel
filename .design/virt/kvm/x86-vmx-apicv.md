# Tier-3: arch/x86/kvm/vmx/vmx.c (APICv subset) — Intel APICv (APIC Virtualization) IPI bypass + virtual-APIC-page

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-vmx.md
upstream-paths:
  - arch/x86/kvm/vmx/vmx.c (APICv-related sections; see grep "apicv|APICv|virtual_apic_page|apic_access_page")
  - arch/x86/kvm/vmx/posted_intr.c (posted-interrupt support)
  - arch/x86/kvm/lapic.c (APICv-aware LAPIC)
-->

## Summary

Intel APICv (APIC Virtualization) is the Intel-VMX hardware feature that bypasses VM-exit on guest LAPIC operations — analog to AMD AVIC. Per-vCPU `virtual_apic_page` is mapped IO-coherent so HW writes IRR/ISR directly; `apic_access_page` is a per-VM 4KiB MMIO-trapped page at xfee0_0000 (LAPIC base) that triggers APIC-write vmexit only for state-changing register writes. APICv comes with sub-features: virtualize-APIC-accesses, virtualize-x2APIC-mode, APIC-register-virtualization, virtual-interrupt-delivery, posted-interrupt-processing. Posted-IRQ from IOMMU IRTE (or cross-vCPU sender) writes IRR-bit + posted-interrupt-notification (PIN) vector → CPU vmexit on PIN OR direct delivery if vCPU running.

This Tier-3 covers APICv-related code in `vmx.c` (~500-700 lines) + `posted_intr.c`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enable_apicv` | module param | `Vmx::ENABLE_APICV` |
| `vmx_refresh_apicv_exec_ctrl(vcpu)` | per-vCPU APICv exec-ctrl refresh | `VcpuVmx::refresh_apicv_exec_ctrl` |
| `vmx_apicv_pre_state_restore(vcpu)` | per-vCPU pre-vmenter setup | `VcpuVmx::apicv_pre_state_restore` |
| `vmx_load_eoi_exitmap(vcpu, eoi_exit_bitmap)` | per-vCPU EOI exitmap update | `VcpuVmx::load_eoi_exitmap` |
| `vmx_set_virtual_apic_mode(vcpu)` | per-vCPU vAPIC mode set (xAPIC vs x2APIC vs disabled) | `VcpuVmx::set_virtual_apic_mode` |
| `vmx_set_apic_access_page_addr(vcpu)` | per-VM apic-access-page setup | `VcpuVmx::set_apic_access_page_addr` |
| `vmx_apicv_pre_state_restore(vcpu)` | per-vCPU pre-state-restore | `VcpuVmx::apicv_pre_state_restore` |
| `vmx_inject_irq_into_guest(vcpu, ...)` | APICv-fast-path IRQ inject | `VcpuVmx::inject_irq_into_guest` |
| `vmx_set_rvi(vec)` | RVI (Requesting Virtual Interrupt) set | `VcpuVmx::set_rvi` |
| `vmx_get_rvi()` | RVI get | `VcpuVmx::get_rvi` |
| `vmx_complete_atomic_exit(vcpu)` | per-vmexit IRR+ISR sync | `VcpuVmx::complete_atomic_exit` |
| `vmx_pi_update_irte(...)` (posted_intr.c) | posted-IRQ IRTE update | `VcpuVmx::pi_update_irte` |
| `vmx_pi_send_ipi(...)` | per-vCPU posted-IPI send | `VcpuVmx::pi_send_ipi` |
| `pi_test_and_set_pir(vec, pi_desc)` | per-PIR atomic-set | `PostedIntrDesc::test_and_set_pir` |
| `pi_clear_on(pi_desc)` | per-PIR clear-on bit | `PostedIntrDesc::clear_on` |
| `pi_test_pending(pi_desc)` | per-PIR pending check | `PostedIntrDesc::test_pending` |
| `vmx_apicv_inhibit_set(...)` / `_clear` | per-VM APICv-inhibit | `VcpuVmx::apicv_inhibit_set` / `_clear` |

## Compatibility contract

REQ-1: APICv sub-features (vmcs.secondary_exec_control bits):
- `VIRTUALIZE_APIC_ACCESSES` (bit 0): trap MMIO to apic-access-page only on state-changing writes.
- `VIRTUALIZE_X2APIC_MODE` (bit 4): pass-through x2APIC MSR access.
- `APIC_REGISTER_VIRTUALIZATION` (bit 8): hardware-managed vAPIC register access.
- `VIRTUAL_INTERRUPT_DELIVERY` (bit 9): hardware-injects pending IRQ at vmenter.
- `POSTED_INTERRUPT_PROCESSING` (vmcs.pin_based_exec bit): hardware-handles posted-IRQ on guest.

REQ-2: Per-vCPU `virtual_apic_page` (4KiB):
- HW-mapped vAPIC page; IRR / ISR / TPR / EOI / etc. registers.
- IO-coherent for IOMMU-direct writes.
- Per-vCPU separate page.

REQ-3: Per-VM `apic_access_page` (4KiB at GPA 0xFEE00000):
- Per-VM single page; MMIO-trapped.
- Guest writes to vAPIC MMIO range (e.g., APIC.ICR write) trap to KVM via APIC-WRITE vmexit.
- Reads pass-through to virtual_apic_page directly.

REQ-4: Per-vCPU posted-interrupt descriptor (PI desc, 64 bytes):
- PIR (Posted-Interrupt Request bitmap; 256 bits = 1 per vector).
- Notification Vector (NV; configurable; typically 0xF2).
- ON (Outstanding Notification bit).
- SN (Suppress Notification bit).
- NDST (Notification Destination; APIC ID of target vCPU's host-CPU).

REQ-5: Posted-IRQ delivery flow:
1. Source (cross-vCPU sender / IOMMU IRTE.posted=1):
   - test_and_set_pir(vec, pi_desc).
   - If was-clear: send NV-IPI to NDST (host CPU running target vCPU).
2. Target host-CPU receives NV-IPI:
   - If guest is running on this CPU: HW processes; clears NV-IPI; sets vmcs IRR-bit; injects on next instruction boundary.
   - If guest not running: vmexit reason POSTED_INTR_NV; KVM picks up.

REQ-6: Per-vCPU APIC mode:
- xAPIC: virtual_apic_page used; 8-bit APIC ID.
- x2APIC: virtual_apic_page used differently (some MSR offsets); 32-bit APIC ID.
- Disabled: legacy LAPIC fully emulated.

REQ-7: APIC-WRITE vmexit handler:
- Per-write-trapped: VMEXIT_REASON_APIC_WRITE.
- Read VMCS exit qualification: register offset.
- Switch on register:
  - APIC_TASKPRI: TPR write; recompute pending IRQ.
  - APIC_EOI: EOI write; clear ISR top.
  - APIC_LDR / _DFR: logical-id update; refresh APICv tables.
  - APIC_ICR: IPI dispatch.
- Resume guest.

REQ-8: APICv inhibit reasons:
- DISABLED (unconditional).
- BLOCKIRQ.
- ABSENT (no in-kernel apic).
- NESTED.
- LOGICAL_ID_ALIASED.
- IRQBALANCE_FORCED (host irqbalance pinning issue).

REQ-9: Per-vCPU EOI exit-bitmap:
- 256-bit bitmap; per-vector trap-on-EOI.
- Used for level-triggered IRQs needing EOI side-effect (IOAPIC remote-IRR clear).

REQ-10: virtualize-x2APIC-mode + APIC-register-virtualization:
- Combined: most x2APIC MSR accesses pass-through (no vmexit).
- ICR / EOI / SELF_IPI still trap.

REQ-11: Live migration:
- vAPIC state migrated via KVM_GET_LAPIC.
- PI desc state included in vCPU events.

REQ-12: Per-vmexit complete_atomic_exit:
- After vmexit, sync vAPIC RVI/SVI to LAPIC IRR/ISR.
- Used so KVM-side LAPIC consistent with HW state.

## Acceptance Criteria

- [ ] AC-1: KVM-VMX with APICv: 16-vCPU guest IPI-stress; per-IPI vmexit count near zero.
- [ ] AC-2: Posted-IRQ + SR-IOV: passthrough VF; per-VF MSI delivered via IOMMU posted-IRQ; PIR populated.
- [ ] AC-3: APIC-WRITE intercept: ICR / EOI traps work correctly.
- [ ] AC-4: virtualize-x2APIC-mode: x2APIC MSRs (0x800-0x83F) most pass-through.
- [ ] AC-5: APICv inhibit on legacy-PIC mode: inhibit set; vmexit-on-IPI restored.
- [ ] AC-6: EOI exitmap: level-IRQ EOI traps; remote-IRR clear correct.
- [ ] AC-7: Live migration: APICv state preserved.
- [ ] AC-8: Cross-vCPU posted-IPI: source vCPU writes ICR; target vCPU sees IRR-bit set; no vmexit if running.
- [ ] AC-9: kvm-unit-tests `apicv` test passes.
- [ ] AC-10: Performance: APICv reduces IPI-heavy workload runtime by ≥ 30% vs no-APICv baseline.

## Architecture

`KvmVmxApicv` per-VM (in kvm.arch):

```
struct KvmVmxApicv {
  apic_access_page: KArc<Page>,                  // shared 4KiB at xfee0_0000
  apic_access_page_pa: u64,
  apicv_inhibit_reasons: AtomicU64,
  pid_table_page: Option<KArc<Page>>,            // posted-interrupt descriptor table (IPI-virt)
}
```

`VcpuVmxApicv` per-vCPU (in vcpu_vmx; see x86-vmx.md):

```
struct VcpuVmx {
  ...
  pi_desc: KArc<PostedIntrDesc>,                  // 64-byte aligned
  pi_desc_addr: u64,
  posted_intr_nv: u8,                             // notification vector
  apicv_active: AtomicBool,
}

#[repr(C, align(64))]
struct PostedIntrDesc {
  pir: [AtomicU64; 4],                            // 256 bits PIR
  control: AtomicU64,                              // [ON | SN | NV | NDST]
}
```

`VcpuVmx::refresh_apicv_exec_ctrl(vcpu)` flow:
1. apicv_active := !apicv_inhibit_reasons && lapic_in_kernel.
2. exec_ctrl := vmcs.secondary_exec_control.
3. If apicv_active:
   - exec_ctrl |= VIRTUALIZE_APIC_ACCESSES | APIC_REGISTER_VIRTUALIZATION | VIRTUAL_INTERRUPT_DELIVERY.
   - If x2apic: exec_ctrl |= VIRTUALIZE_X2APIC_MODE.
4. Else: exec_ctrl &= ~all-APICv-bits.
5. VMWRITE secondary_exec_control.

`VcpuVmx::pi_send_ipi(target_vcpu, vec)`:
1. pi_desc := target_vcpu.pi_desc.
2. was_set := pi_test_and_set_pir(vec, pi_desc).
3. If !was_set:
   - Set ON (Outstanding Notification) bit in pi_desc.control.
   - cpu := target_vcpu.cpu (last-running CPU).
   - apic_send_IPI(cpu, target_vcpu.posted_intr_nv).

`VcpuVmx::pi_update_irte(vcpu, host_irq, guest_irq, set)` (IOMMU posted):
1. If set: IRTE.posted = 1; IRTE.pda = vcpu.pi_desc_addr; IRTE.notification_vector = vcpu.posted_intr_nv.
2. Else: IRTE.posted = 0.

`VcpuVmx::complete_atomic_exit(vcpu)` (post-vmexit):
1. Read vmcs.GUEST_INTR_STATUS field: RVI (low 8 bits), SVI (high 8 bits).
2. Sync vAPIC IRR / ISR with KVM-side lapic state.
3. Used so KVM understands current state for non-APICv code paths.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pi_desc_aligned` | INVARIANT | per-vCPU pi_desc 64-byte aligned. |
| `pir_atomic_setbit` | INVARIANT | pi_test_and_set_pir uses atomic-test-and-set; defense against torn-update. |
| `apicv_active_consistent` | INVARIANT | apicv_active flag matches vmcs.secondary_exec_control APICv-bits. |
| `apic_access_page_aligned` | INVARIANT | apic_access_page guest-PA 4KiB-aligned at 0xFEE00000. |
| `posted_nv_validated` | INVARIANT | posted_intr_nv ∈ [16, 255]; defense against reserved-vector. |

### Layer 2: TLA+

`virt/kvm/posted_intr.tla` (already exists from earlier Tier-3 collection) covers posted-IRQ delivery semantics.

`virt/kvm/apicv_inhibit.tla` (analog to AVIC inhibit):
- Per-VM APICv state ∈ {Active, Inhibited(reasons)}.
- Properties:
  - `safety_inhibit_disables_apicv` — Inhibited implies APICv-bits clear in vmcs.
  - `safety_inhibit_clear_re_enables` — clearing all reasons re-enables APICv.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuVmx::refresh_apicv_exec_ctrl` post: vmcs.secondary_exec_control reflects apicv_active state | `VcpuVmx::refresh_apicv_exec_ctrl` |
| `VcpuVmx::pi_send_ipi` post: PIR bit set; ON set; NV-IPI sent to NDST | `VcpuVmx::pi_send_ipi` |
| `VcpuVmx::complete_atomic_exit` post: KVM lapic IRR/ISR synced with vmcs RVI/SVI | `VcpuVmx::complete_atomic_exit` |
| `VcpuVmx::pi_update_irte` post: IRTE posted-bit + PDA atomically updated | `VcpuVmx::pi_update_irte` |

### Layer 4: Verus/Creusot functional

`Per-IPI: source writes ICR; target's PIR-bit set + delivered via posted-IPI / vmexit` semantic equivalence: per-IPI the target vCPU eventually sees IRQ at correct vector, regardless of running-on-CPU vs not-running.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-vmx.md` § Hardening.)

APICv-specific reinforcement:

- **Per-vCPU virtual_apic_page IO-coherent** — defense against device-DMA tearing.
- **Per-vCPU PI desc 64-byte aligned** — defense against split-cacheline access.
- **Per-vCPU PIR atomic test-and-set** — defense against torn IRQ-bit set.
- **Per-VM apic_access_page 4KiB-aligned at xfee0_0000** — defense against incorrect MMIO-trap.
- **APICv inhibit reasons bitmap** — defense against partial-inhibit.
- **Per-vCPU posted_intr_nv vector validated** — defense against reserved-vector causing CPU exception.
- **EOI exitmap per-vector** — defense against level-IRQ losing remote-IRR clear.
- **virtualize-x2APIC + APIC-register-virtualization combined** — defense against guest direct-write to read-only registers.
- **Live-migrate clears APICv state** — defense against post-migrate stale tables.
- **Per-vCPU complete_atomic_exit** — defense against KVM lapic out-of-sync with HW.
- **Per-VM IPI virtualization PID-table** (newer feature) — defense against IPI-source-vmexit leak.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — virtual-APIC page kernel-allocated; KVM_GET/SET_LAPIC bounded by `struct kvm_lapic_state` slab size.
- **PAX_KERNEXEC** — `vmx_apicv_post_state_restore`, `vmx_hwapic_isr_update`, `vmx_hwapic_irr_update`, `vmx_set_apic_access_page_addr` resolve through RX-only kernel text.
- **PAX_RANDKSTACK** — APICv inhibit refresh and IPI virtualization paths inherit RANDKSTACK.
- **PAX_REFCOUNT** — per-VM apicv_inhibit_reasons bitmap, per-vCPU virtual-APIC page refcount saturating.
- **PAX_MEMORY_SANITIZE** — virtual-APIC page, posted-interrupt descriptor, EOI exitmap zeroed on alloc and free; APIC-access page zeroed before successor VM mmap.
- **PAX_UDEREF** — KVM_SET_LAPIC copy_from_user STAC/CLAC bracketed.
- **PAX_RAP / kCFI** — `kvm_x86_ops.refresh_apicv_exec_ctrl`, `hwapic_isr_update`, `hwapic_irr_update` slots RAP-signed.
- **GRKERNSEC_HIDESYM** — virtual-APIC page HPA, posted-int descriptor kaddr redacted unless CAP_SYSLOG + gr-rbac.
- **GRKERNSEC_DMESG** — APICv inhibit-reason transitions, EOI-exitmap-update warnings rate-limited and gated by `dmesg_restrict`.
- **KVM ioctl CAP_SYS_ADMIN strict** — KVM_CREATE_IRQCHIP + APICv enable gated to CAP_SYS_ADMIN + gr-rbac VM-create role.
- **VMCS virtual-APIC page validated** — `VIRTUAL_APIC_PAGE_ADDR` field set from kernel-allocated HPA; never user-controlled.
- **EOI exitmap per-vector** — level-IRQ exitmap programmed under `apic->lock`; remote-IRR clear cannot be lost.
- **virtualize-x2APIC + register-virtualization combined** — guest direct-write to read-only x2APIC registers trapped per Intel-spec.
- **APIC-access page protection** — APIC-access HPA mapped read-only in EPT for non-APICv vCPUs; APICv vCPUs see virtualized version.
- **Nested-virt strict** — L2 APICv inhibited when L0 has no nested-APICv support; L1 cannot accidentally expose L0 APIC state to L2.

Per-doc rationale: APICv (Intel APICv = virtualize-APIC + virtual-int-delivery + posted-int) accelerates LAPIC by mapping a per-vCPU APIC register page that the guest writes directly; the grsec reinforcement here keeps the virtual-APIC page kernel-validated, SANITIZEs APIC tables on free, atomic-locks EOI exitmap to defeat remote-IRR loss, and inhibits APICv on feature-mismatch rather than running inconsistent.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VMX core (covered in `x86-vmx.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- AMD AVIC (covered in `x86-svm-avic.md` Tier-3; analog feature)
- IOMMU IR (covered in `drivers/iommu/intel-irq-remap.md` Tier-3)
- Implementation code
