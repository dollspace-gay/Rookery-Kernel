# Tier-3: arch/x86/kvm/hyperv.c — KVM Hyper-V emulation (per-VM Hyper-V hypercall surface + SynIC + StimerS for Windows guests)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/hyperv.c
  - arch/x86/kvm/hyperv.h
  - arch/x86/kvm/x86.c (CPUID 0x40000005-0x4000000F dispatch + Hyper-V MSRs)
  - include/asm-generic/hyperv-tlfs.h
  - arch/x86/include/asm/hyperv-tlfs.h
-->

## Summary

KVM emulates Microsoft's Hyper-V hypervisor TLFS (Top-Level Functional Specification) surface so Windows guests + Hyper-V-enlightened Linux guests can use Hyper-V's paravirt: synthetic timers (SynIC stimer), synthetic interrupt controller (SynIC), Hyper-V hypercalls (HvFlushVirtualAddressList, HvNotifyLongSpinWait, HvPostMessage), Hyper-V tsc_page (paravirt clock), VP-assist-page (per-vCPU shared state), VMBus channel emulation, etc. Critical for: running unmodified Windows + Server in qemu+kvm; allowing nested Hyper-V (KVM-on-Hyper-V or Hyper-V-on-KVM); attaining performance parity with running directly on Hyper-V. Activated per-VM via KVM_CAP_HYPERV.

This Tier-3 covers `arch/x86/kvm/hyperv.c` (~2926 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_hv` | per-VM Hyper-V state | `kernel::kvm::x86::hyperv::KvmHv` |
| `struct kvm_vcpu_hv` | per-vCPU Hyper-V state | `VcpuHv` |
| `struct kvm_vcpu_hv_synic` | per-vCPU SynIC state | `VcpuHvSynic` |
| `struct kvm_vcpu_hv_stimer` | per-vCPU stimer (4 timers per vCPU) | `VcpuHvStimer` |
| `kvm_hv_init(kvm)` / `kvm_hv_cleanup(kvm)` | per-VM lifecycle | `KvmHv::init` / `_cleanup` |
| `kvm_hv_vcpu_init(vcpu)` / `kvm_hv_vcpu_cleanup(vcpu)` | per-vCPU | `VcpuHv::init` / `_cleanup` |
| `kvm_hv_set_msr_common(vcpu, msr_info, host_initiated)` | Hyper-V MSR write dispatch | `VcpuHv::set_msr` |
| `kvm_hv_get_msr_common(vcpu, msr_info, host_initiated)` | Hyper-V MSR read dispatch | `VcpuHv::get_msr` |
| `kvm_hv_hypercall(vcpu)` | hypercall entry from VMX/SVM exit | `VcpuHv::hypercall` |
| `kvm_hv_synic_init(vcpu)` | SynIC init | `VcpuHvSynic::init` |
| `kvm_hv_synic_send_eoi(vcpu, vector)` | SynIC EOI delivery | `VcpuHvSynic::send_eoi` |
| `kvm_hv_synic_set_irq(vcpu, sint, vector)` | per-SINT assert | `VcpuHvSynic::set_irq` |
| `kvm_hv_stimer_init(vcpu, idx)` | per-stimer init | `VcpuHvStimer::init` |
| `kvm_hv_stimer_expiration(...)` | per-stimer fire | `VcpuHvStimer::expiration` |
| `kvm_hv_stimer_pending(vcpu)` | per-vCPU pending check | `VcpuHvStimer::pending` |
| `kvm_emulate_wrmsr(vcpu)` Hyper-V branch | dispatcher | covered in `Vcpu::handle_msr` |
| `kvm_hv_setup_tsc_page(vcpu)` | per-vCPU Hyper-V tsc-page | `VcpuHv::setup_tsc_page` |
| `kvm_hv_eventfd_assign(...)` | KVM_HYPERV_EVENTFD ioctl | `KvmHv::eventfd_assign` |
| `hv_post_message(...)` | HvPostMessage hypercall | `VcpuHv::post_message` |
| `hv_signal_event(...)` | HvSignalEvent hypercall | `VcpuHv::signal_event` |
| `kvm_hv_flush_tlb(...)` | HvFlushVirtualAddressList hypercall | `KvmHv::flush_tlb` |
| `kvm_hv_send_ipi(...)` | HvSendSyntheticClusterIpi hypercall | `KvmHv::send_ipi` |

## Compatibility contract

REQ-1: Per-VM Hyper-V state:
- `hv_guest_os_id` (MSR_HV_GUEST_OS_ID; identifies guest OS).
- `hv_hypercall` (MSR_HV_HYPERCALL; hypercall page).
- `hv_tsc_page` (MSR_HV_REFERENCE_TSC; paravirt-clock page).
- `hv_crash_param[5]` (Windows BSOD parameters).
- `hv_crash_ctl` (BSOD control).
- `hv_reenlightenment_control`, `hv_tsc_emulation_control`, `hv_tsc_emulation_status`.
- `eventfd_list` (per-VMBus event fd assignments).

REQ-2: Per-vCPU Hyper-V state:
- `synic` (synthetic IRQ controller; 16 SINTs per vCPU).
- `stimer[4]` (4 synthetic timers per vCPU per spec).
- `vp_assist_page` (per-vCPU shared page; nested-virt status).
- `hv_vapic` (per-vCPU virtual-APIC; Hyper-V variant).
- `hv_clock`.
- `hv_vp_index` (Hyper-V virtual processor ID).

REQ-3: Hyper-V CPUID leaves:
- 0x40000000: signature "Microsoft Hv".
- 0x40000001: interface signature "Hv#1".
- 0x40000002: hypervisor version.
- 0x40000003: feature identification (HV-X64-MSR-VP-RUNTIME, _MSR_TIME_REF_COUNT, _MSR_SYNIC, _MSR_SYNTIMER, _MSR_APIC_ACCESS, _MSR_HYPERCALL, _MSR_VP_INDEX, _MSR_REFERENCE_TSC, etc.).
- 0x40000004: implementation recommendations.
- 0x40000005: implementation limits (max_vp_count, max_lp_count).
- 0x40000006: implementation hardware features.

REQ-4: Hyper-V hypercall dispatch:
- Guest writes hypercall-input-value to RCX, GPA-of-input-buffer to RDX, GPA-of-output to R8.
- VMCALL → VMexit → KVM emulates.
- `kvm_hv_hypercall(vcpu)`:
  1. Read RCX (hypercall code + flags).
  2. Switch on code:
     - HVCALL_NOTIFY_LONG_SPIN_WAIT (1): yield-to-target (similar to KVM PV-SCHED-YIELD).
     - HVCALL_FLUSH_VIRTUAL_ADDRESS_LIST (3): TLB-shootdown across vCPUs.
     - HVCALL_FLUSH_VIRTUAL_ADDRESS_SPACE (2): full TLB flush.
     - HVCALL_SEND_IPI (0xb): send synthetic cluster IPI.
     - HVCALL_POST_MESSAGE (0x5C): post msg to a SINT.
     - HVCALL_SIGNAL_EVENT (0x5D): signal event.
     - HVCALL_GET_PARTITION_ID, HVCALL_DEPOSIT_MEMORY, etc.
  3. Write status to RAX.

REQ-5: SynIC (Synthetic Interrupt Controller):
- 16 SINTs (synthetic-interrupt) per vCPU.
- Per-SINT MSR (MSR_HV_SINT0..15): vector + masked + auto-eoi + polling.
- Message-page (MSR_HV_SIMP): per-vCPU 4KiB; per-SINT slot for message delivery.
- Event-flags-page (MSR_HV_SIEFP): per-vCPU 4KiB; per-SINT event flags.
- KVM dispatches per-SINT IRQ to vCPU LAPIC at SINT.vector.

REQ-6: SynIC stimer (synthetic timer):
- 4 per vCPU; periodic or one-shot.
- Per-stimer MSR (MSR_HV_STIMER0..3_CONFIG, _COUNT): config + count.
- Each stimer linked to a SINT for delivery.
- Backed by per-vCPU hrtimer.

REQ-7: Hyper-V tsc_page (MSR_HV_REFERENCE_TSC):
- Per-VM 4KiB; layout per-Hyper-V-TLFS: tsc_offset + tsc_scale + sequence.
- Per-VM updated by KVM on cpufreq change + live-migration.
- Read atomically by guest via sequence-counter.

REQ-8: Per-vCPU VP-assist-page (MSR_HV_VP_ASSIST_PAGE):
- 4KiB shared between KVM + guest.
- Layout: nested-virt status, virtualization-fault-info, etc.
- Used by Hyper-V's "enlightened VMCS" optimization (if guest is Hyper-V running L2).

REQ-9: KVM_HYPERV_EVENTFD ioctl:
- Bind eventfd to specific (vp, sint, flag-id) for VMBus channel emulation.
- Userspace QEMU writes eventfd → KVM signals SINT → guest wakes.

REQ-10: BSOD reporting:
- MSR_HV_CRASH_CTL = 0x40000105: when guest writes 0x80000000 (crash trigger), KVM logs guest-side BSOD parameters.
- MSR_HV_CRASH_P0..P4: crash parameters.
- KVM logs to dmesg: "Hyper-V crash: P0=... P1=..."

REQ-11: KVM_CAP_HYPERV gating:
- Per-VM enable; without this, all Hyper-V CPUID + MSRs return #GP.

REQ-12: Live migration:
- All Hyper-V MSRs included in KVM_GET_MSRS migration list.
- Per-vCPU SynIC state migrated.
- Per-vCPU stimer state migrated.

## Acceptance Criteria

- [ ] AC-1: KVM_CAP_HYPERV advertised; verify via KVM_CHECK_EXTENSION.
- [ ] AC-2: Boot Windows 10/11 guest under qemu+kvm with `-cpu Hyper-V-passthrough`: boot completes without BSOD.
- [ ] AC-3: Hyper-V CPUID: guest sees Microsoft Hv signature + advertised features.
- [ ] AC-4: SynIC: guest enables SINT0; KVM delivers SINT IRQ at configured vector.
- [ ] AC-5: stimer: guest configures stimer0 periodic 100Hz; stimer fires + delivers via linked SINT.
- [ ] AC-6: HVCALL_FLUSH_VIRTUAL_ADDRESS_LIST: 16-vCPU guest issues; KVM TLB-flushes targets.
- [ ] AC-7: KVM_HYPERV_EVENTFD: VMBus channel test; eventfd triggers SINT to guest.
- [ ] AC-8: BSOD logging: guest crash; dmesg shows Hyper-V crash parameters.
- [ ] AC-9: Live migration: Windows guest migrated; SynIC + stimer state preserved.
- [ ] AC-10: Spec-violation defense: hypercall with reserved code returns #GP.

## Architecture

`KvmHv` per-VM:

```
struct KvmHv {
  hv_guest_os_id: u64,
  hv_hypercall: u64,
  hv_tsc_page: KArc<HypervTscPage>,
  hv_crash_param: [u64; 5],
  hv_crash_ctl: u64,
  hv_reenlightenment_control: u64,
  hv_tsc_emulation_control: u64,
  hv_tsc_emulation_status: u64,
  eventfd_list: ListHead,
  pa_page_gpa: u64,                            // per-VM partition-assist page
  syndbg: HypervSyndbg,                        // synthetic-debugger
}

struct VcpuHv {
  vp_assist_page: u64,
  hv_vp_index: u32,
  vp_runtime: u64,
  synic: VcpuHvSynic,
  stimer: [VcpuHvStimer; 4],
  hv_clock: u64,
  ...
}

struct VcpuHvSynic {
  active: bool,
  scontrol: u64,                               // SynIC global control
  sversion: u64,
  sint: [u64; 16],                             // per-SINT MSR shadow
  simp: u64,                                    // message page address
  siefp: u64,                                   // event flags page address
  vec_bitmap: u64,                              // vector → SINT lookup
  auto_eoi_bitmap: u64,
}

struct VcpuHvStimer {
  index: u8,
  config: u64,
  count: u64,
  exp_time: u64,
  timer: HrTimer,                              // backing hrtimer
}
```

`VcpuHv::hypercall` flow:
1. Read RCX (hypercall code).
2. Validate: code in supported set.
3. Switch:
   - HVCALL_NOTIFY_LONG_SPIN_WAIT: kvm_vcpu_yield_to_self_or_other.
   - HVCALL_FLUSH_VIRTUAL_ADDRESS_LIST: KvmHv::flush_tlb_list.
   - HVCALL_SEND_IPI: KvmHv::send_synthetic_ipi.
   - HVCALL_POST_MESSAGE: VcpuHv::post_message.
   - HVCALL_SIGNAL_EVENT: VcpuHv::signal_event.
4. Set RAX = HV_STATUS_SUCCESS or error.

`VcpuHvSynic::set_irq(vcpu, sint_idx, vector)`:
1. Resolve sint := vcpu.synic.sint[sint_idx].
2. If sint.masked: return.
3. KVM injects vector via LAPIC.
4. If sint.auto_eoi: don't set ISR (auto-EOI).

`VcpuHvStimer::expiration(stimer)`:
1. Lookup linked SINT from stimer.config.
2. Compose message in vcpu.synic.simp page at SINT slot.
3. VcpuHvSynic::set_irq(vcpu, sint_idx, sint.vector).
4. If stimer.config.periodic: re-arm hrtimer.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `synic_sint_idx_bounded` | OOB | per-SINT idx ∈ [0, 15]; defense against guest writing OOB SINT. |
| `stimer_idx_bounded` | OOB | per-stimer idx ∈ [0, 3]. |
| `tsc_page_aligned` | INVARIANT | hv_tsc_page guest-PA 4KiB-aligned. |
| `vp_assist_page_aligned` | INVARIANT | per-vCPU vp_assist_page 4KiB-aligned. |
| `hypercall_code_validated` | INVARIANT | unrecognized hypercall returns HV_STATUS_INVALID_HYPERCALL_CODE. |

### Layer 2: TLA+

`virt/kvm/hyperv_synic.tla`:
- Per-SINT state ∈ {Masked, Asserted, Acked}.
- Transitions per set_irq, send_eoi.
- Properties:
  - `safety_masked_no_inject` — Masked SINT does not deliver IRQ.
  - `safety_eoi_clears_isr` — non-auto-eoi SINT delivery sets ISR; EOI clears.
  - `liveness_asserted_eventually_acked` — assuming guest issues EOI, every Asserted eventually Acked.

`virt/kvm/hyperv_stimer.tla`:
- Per-stimer state ∈ {Disabled, OneShot(expires), Periodic(period, expires)}.
- Properties:
  - `safety_periodic_re_arms` — Periodic stimer re-arms hrtimer after fire.
  - `safety_oneshot_disables_after_fire` — OneShot transitions to Disabled.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuHv::hypercall` post: RAX set; status reflects op outcome | `VcpuHv::hypercall` |
| `VcpuHvSynic::set_irq` post: vector injected to LAPIC iff !masked | `VcpuHvSynic::set_irq` |
| `VcpuHvStimer::expiration` post: SINT message posted; hrtimer re-armed if periodic | `VcpuHvStimer::expiration` |
| Per-vCPU SynIC + stimer state migrated atomically | `VcpuHv` migration | 
| Per-VM hv_tsc_page sequence-counter atomic update | `VcpuHv::setup_tsc_page` |

### Layer 4: Verus/Creusot functional

`Hyper-V hypercall + SynIC stimer + tsc_page → Windows guest perceives Hyper-V semantics consistent with TLFS spec`: per-feature emulated behavior matches Microsoft TLFS spec (limited to features KVM supports).

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

Hyper-V-specific reinforcement:

- **KVM_CAP_HYPERV per-VM gate** — defense against unintended Hyper-V emulation increasing attack surface.
- **Per-MSR validation** at WRMSR — defense against guest writing reserved bits causing emulator state-inconsistency.
- **Per-SINT vector range [16, 255]** — defense against reserved-vector causing CPU exception.
- **Per-stimer max-frequency cap** — defense against guest configuring 1ns-period stimer flooding LAPIC.
- **Hypercall code validated** — defense against undocumented codes reaching unimplemented paths.
- **TLB-flush hypercall vCPU-mask validated** — defense against malformed mask referring to non-existent vCPU.
- **VP-assist-page in IOMMU-protected mem** — defense against device-DMA writing nested-virt status.
- **Per-VM eventfd assignments tracked** — defense against use-after-close on eventfd.
- **BSOD param logging rate-limited** — defense against guest-induced log flood via repeated BSOD writes.
- **Per-vCPU synic.simp / siefp guest-PA validated** — defense against guest pointing message-page to host kernel memory.
- **HV_STATUS validation per-spec** — defense against non-compliant return values confusing guest.
- **Live-migrate: synic.scontrol disabled briefly during transit** — defense against post-migrate stale SINT-fire.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — Hyper-V hypercall arg / SIMP / SIEFP copies bounded by page size.
- **PAX_KERNEXEC** — Hyper-V emulation dispatch + hypercall table RO after init.
- **PAX_RANDKSTACK** — randomized kstack per KVM_RUN; hypercall entry uses fresh stack.
- **PAX_REFCOUNT** — synic / stimer / vp-assist refcounts saturating.
- **PAX_MEMORY_SANITIZE** — synic.simp, siefp, vp_assist_page zeroed on vCPU destroy.
- **PAX_UDEREF** — synic.simp/siefp guest-PA → kvm_vcpu_gfn_to_hva validated user-range.
- **PAX_RAP / kCFI** — Hyper-V hypercall dispatch type-checked.
- **GRKERNSEC_HIDESYM** — simp/siefp/vp_assist GPAs redacted.
- **GRKERNSEC_DMESG** — BSOD writes + hypercall-flood rate-limited.

Hyper-V-specific:

- **CAP_SYS_ADMIN on KVM_CAP_HYPERV_* enable** — defense against unprivileged Hyper-V emulation surface.
- **TLB-flush hypercall vCPU-mask range-checked against KVM_MAX_VCPUS** — blocks OOB vCPU access.
- **stimer-period floor (≥ 1µs)** — blocks LAPIC IPI-flood DoS.
- **Synthetic IC SINT vector [16,255] hard-enforced** — defense against reserved-IDT injection.

Rationale: Hyper-V emulation exposes a parallel hypercall ABI with shared guest pages (synic SIMP/SIEFP); sanitize + UDEREF on those pages closes the dominant cross-VM message-leak surface.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- SVM vendor (covered in `x86-svm.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- KVM PV-IPI (covered in `x86-pv-ipi.md` Tier-3)
- VMBus / synthetic-vmbus driver (separate; covered in `drivers/hv/vmbus.md` future Tier-3)
- Implementation code
