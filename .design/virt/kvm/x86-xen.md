# Tier-3: arch/x86/kvm/xen.c — KVM Xen guest emulation (per-VM Xen hypercall surface + shared_info + grant_table + evtchn)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/kvm-core.md
upstream-paths:
  - arch/x86/kvm/xen.c
  - arch/x86/kvm/xen.h
  - include/xen/interface/* (Xen ABI structs - external)
  - include/uapi/linux/kvm.h (KVM_CAP_XEN_HVM + KVM_XEN_HVM_*)
-->

## Summary

KVM emulates a small subset of the Xen hypervisor PV interface so unmodified Xen-PVH guests (or Xen-DOMU using Xen-PV drivers) can run under KVM. This includes: Xen hypercalls (HYPERVISOR_sched_op, HYPERVISOR_event_channel_op, HYPERVISOR_vcpu_op, etc.), per-vCPU vcpu_info shared page, per-VM shared_info page, event-channel emulation (2-bit per-evtchn pending+masked state in shared_info), grant-table emulation (memory sharing for PV drivers), Xen pvclock, IDT vector for Xen callback, etc. Use case: cloud platforms that historically ran Xen workloads + want to migrate to KVM-host without rebuilding guests.

This Tier-3 covers `arch/x86/kvm/xen.c` (~2348 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_xen` | per-VM Xen state | `kernel::kvm::x86::xen::KvmXen` |
| `struct kvm_vcpu_xen` | per-vCPU Xen state | `VcpuXen` |
| `kvm_xen_init_vm(kvm)` / `kvm_xen_destroy_vm(kvm)` | per-VM lifecycle | `KvmXen::init` / `_destroy` |
| `kvm_xen_init_vcpu(vcpu)` / `kvm_xen_destroy_vcpu(vcpu)` | per-vCPU | `VcpuXen::init` / `_destroy` |
| `kvm_xen_hvm_get_attr(...)` / `_set_attr(...)` | KVM_XEN_HVM_GET_ATTR / _SET_ATTR ioctl | `KvmXen::get_attr` / `_set_attr` |
| `kvm_xen_vcpu_get_attr(...)` / `_set_attr(...)` | per-vCPU attr ioctl | `VcpuXen::get_attr` / `_set_attr` |
| `kvm_xen_hypercall(vcpu)` | hypercall dispatcher | `VcpuXen::hypercall` |
| `kvm_xen_setup_pvclock(vcpu)` | per-vCPU Xen pvclock | `VcpuXen::setup_pvclock` |
| `kvm_xen_runstate_set_running(vcpu)` / `_runstate_set_preempted(vcpu)` | runstate update | `VcpuXen::set_running` / `_set_preempted` |
| `kvm_xen_inject_vcpu_vector(...)` | per-vCPU upcall vector inject | `VcpuXen::inject_vector` |
| `kvm_xen_assign_evtchn(...)` | evtchn assignment ioctl | `KvmXen::assign_evtchn` |
| `kvm_xen_set_evtchn_fast(...)` / `_set_evtchn(...)` | evtchn fire | `KvmXen::set_evtchn_fast` / `_set_evtchn` |
| `kvm_xen_handle_eventfd(...)` | eventfd → evtchn dispatch | `KvmXen::handle_eventfd` |
| `xen_vcpu_runstate_running` / `_runnable` / `_blocked` / `_offline` | runstate enum | `XenRunstate` |
| `KVM_XEN_HVM_CONFIG_INTERCEPT_HCALL` | per-VM bit: KVM intercept hypercalls | UAPI |
| `KVM_XEN_HVM_CONFIG_SHARED_INFO` | per-VM bit: KVM-side shared_info | UAPI |
| `KVM_XEN_HVM_CONFIG_RUNSTATE` | per-VM bit: KVM-side runstate updates | UAPI |
| `KVM_XEN_HVM_CONFIG_EVTCHN_2LEVEL_*` | per-VM evtchn enable | UAPI |

## Compatibility contract

REQ-1: KVM_CAP_XEN_HVM gating:
- KVM_XEN_HVM_CONFIG ioctl enables Xen emulation; without it, all Xen MSRs/CPUID return #GP.
- Per-VM flags select sub-features (intercept_hcall, shared_info, runstate, evtchn_2level, evtchn_send, upcall_vector).

REQ-2: Per-VM Xen state:
- `shared_info` (per-VM 4KiB page; layout per Xen ABI).
- `xen_msr` (Xen MSR address; default 0x40000000).
- `xen_version_*` (Xen version reported via CPUID 0x40000xxx leaves).
- `evtchn_pending_lock` (per-VM evtchn-bitmap update lock).
- `evtchn_redirect` (per-VM redirect table for evtchn-to-real-IRQ).

REQ-3: Per-vCPU Xen state:
- `vcpu_info` (per-vCPU 64-byte chunk in shared_info or separate page; per-vCPU pending+masked evtchn-bitmap copy).
- `runstate` (XenVcpuRunstate { state, state_entry_time, time[4] }).
- `runstate_entry_time` last-update timestamp.
- `pvclock` (per-vCPU; same struct as kvmclock).
- `upcall_vector` (per-vCPU IDT vector for Xen callback).
- `evtchn_received` (per-vCPU bitmap of pending evtchns for this vCPU).

REQ-4: Xen CPUID leaves:
- 0x40000000: signature "XenVMMXenVMM".
- 0x40000001: Xen-version-major + minor.
- 0x40000002: msr-base + nr_msrs.
- 0x40000003: features.

REQ-5: Hypercall dispatch (`kvm_xen_hypercall`):
- Guest hypercall via direct CALL instruction to Xen hypercall page (mapped per HYPERVISOR_sched_op MSR).
- KVM intercepts via #UD on hypercall page or VMCALL emulation.
- Read RAX (hypercall number); switch:
  - HYPERVISOR_sched_op (29): SCHEDOP_yield, _block, _shutdown, _poll.
  - HYPERVISOR_event_channel_op (32): EVTCHNOP_send, _bind_*, _close.
  - HYPERVISOR_vcpu_op (24): VCPUOP_register_vcpu_info, _runstate_memory_area, _stop_periodic_timer, etc.
  - HYPERVISOR_xen_version (17).
  - HYPERVISOR_grant_table_op (20): GNTTABOP_setup_table, _query_size, _copy.
  - HYPERVISOR_set_timer_op (15).

REQ-6: Event-channel emulation:
- 2-bit-per-evtchn state in shared_info: pending-bit + masked-bit.
- Total 4096 evtchns by default (2-level evtchn extension supports up to 131072).
- Per-evtchn delivery target: vCPU + upcall_vector.
- KVM_XEN_EVTCHN ioctl from userspace fires per-evtchn.

REQ-7: Runstate accumulator (per-vCPU):
- 4 states: Running, Runnable, Blocked, Offline.
- Per-state ns counter accumulated on transitions.
- Updated on vmenter/vmexit + halt/run.
- Guest reads to compute steal-time-equivalent.

REQ-8: Xen pvclock:
- Same struct as kvmclock (`pvclock_vcpu_time_info`).
- Per-vCPU time-info page at vcpu_info offset.
- Updated on cpufreq / live-migrate same as kvmclock.

REQ-9: Upcall vector:
- Per-vCPU vector configured via HVM_PARAM_CALLBACK_IRQ.
- KVM injects this vector when evtchn-pending bit-set transitions 0→1.

REQ-10: Per-VM evtchn → eventfd binding:
- Userspace QEMU binds eventfd to evtchn via KVM_XEN_HVM_EVTCHN_SEND ioctl + reverse direction.
- Used for IPC between qemu device-emulator + guest.

REQ-11: Schedop-poll:
- Guest issues SCHEDOP_poll with timeout + evtchn list; vCPU blocks until any evtchn fires or timeout.
- Implementation: hrtimer + evtchn-pending check.

REQ-12: KVM_GET_*_ATTR / _SET_ATTR for migration:
- Per-VM + per-vCPU snapshot of all Xen state for migration.

## Acceptance Criteria

- [ ] AC-1: KVM_CAP_XEN_HVM advertised; KVM_XEN_HVM_CONFIG ioctl succeeds.
- [ ] AC-2: Boot Xen-PVH guest (or Linux-with-Xen-PV-drivers) under qemu+kvm: guest boots; shared_info + vcpu_info populated.
- [ ] AC-3: Xen CPUID: guest sees XenVMM signature.
- [ ] AC-4: Xen pvclock: guest /proc/clocksource shows xen-pvclock; CLOCK_MONOTONIC accuracy < 100ns drift.
- [ ] AC-5: Event-channel: per-evtchn-fire from userspace QEMU; guest upcall handler runs.
- [ ] AC-6: Runstate: 16-vCPU guest under host contention; per-vCPU runstate accumulator advances.
- [ ] AC-7: SCHEDOP_yield: guest yields; co-vCPU runs; round-robin observed.
- [ ] AC-8: Live migration: Xen-PV guest migrated; shared_info + per-vCPU state preserved.
- [ ] AC-9: Spec-violation defense: malformed hypercall returns -ENOSYS via RAX.
- [ ] AC-10: KVM_XEN_HVM_EVTCHN_SEND from userspace: bind eventfd → evtchn; signal eventfd → guest sees evtchn-pending.

## Architecture

`KvmXen` per-VM:

```
struct KvmXen {
  hvm_config: KvmXenHvmConfig,                 // KVM_XEN_HVM_CONFIG snapshot
  long_mode: bool,
  xen_version: u32,
  shared_info_pa: u64,                          // 4KiB-aligned guest-PA
  shared_info_cache: GfnToHvaCache,
  evtchn_pending_lock: SpinLock<()>,
  evtchn_2level: bool,
  evtchn_max: u32,                              // 4096 or 131072
  evtchn_eventfd_list: ListHead,                // per-evtchn → eventfd binding
  upcall_vector: u8,
}
```

`VcpuXen` per-vCPU:

```
struct VcpuXen {
  vcpu_info_pa: u64,                            // 4KiB-aligned guest-PA
  vcpu_info_cache: GfnToHvaCache,
  vcpu_time_info_pa: u64,                       // separate pvclock page
  vcpu_time_info_cache: GfnToHvaCache,
  runstate_pa: u64,                             // separate runstate page
  runstate_cache: GfnToHvaCache,
  runstate: u8,                                  // 0=Running, 1=Runnable, 2=Blocked, 3=Offline
  runstate_entry_time: u64,
  runstate_times: [u64; 4],                     // accumulator per state
  upcall_vector: u8,
  evtchn_received: KBitmap,                     // per-vCPU pending evtchns
  pv_eoi_pending: bool,
  next_runstate: u8,                            // queued transition
}
```

`VcpuXen::hypercall(vcpu)`:
1. Read RAX (hypercall number), RDI/RSI/RDX/R10/R8/R9 (args).
2. Switch:
   - SCHED_OP (29): per-subop (yield, block, etc.).
   - EVENT_CHANNEL_OP (32): per-subop (send, bind, close).
   - VCPU_OP (24): per-subop (register_vcpu_info, runstate_memory_area).
   - XEN_VERSION (17): return version.
   - SET_TIMER_OP (15): arm hrtimer.
3. Set RAX = result.

`KvmXen::set_evtchn_fast(kvm, vm_id, port)`:
1. Acquire evtchn_pending_lock.
2. Set pending-bit in shared_info.evtchn_pending[port / 64] bit (port % 64).
3. If !masked-bit set: 
   - target_vcpu := evtchn-port-target-vcpu.
   - target_vcpu.evtchn_received.set(port).
   - kvm_xen_inject_vcpu_vector(target_vcpu, target_vcpu.upcall_vector).
4. Release lock.

`VcpuXen::runstate_set_running(vcpu)`:
1. now := ktime_get_ns().
2. delta := now - vcpu.runstate_entry_time.
3. vcpu.runstate_times[vcpu.runstate] += delta.
4. vcpu.runstate = XEN_RUNSTATE_RUNNING (0).
5. vcpu.runstate_entry_time = now.
6. Write runstate-page (atomic via version-counter).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `evtchn_port_bounded` | OOB | per-evtchn port < evtchn_max; defense against port-OOB in shared_info. |
| `vcpu_info_aligned` | INVARIANT | per-vCPU vcpu_info guest-PA aligned. |
| `runstate_state_valid` | INVARIANT | per-vCPU runstate ∈ {Running, Runnable, Blocked, Offline}; defense against guest writing reserved value. |
| `hypercall_returns_eax` | INVARIANT | every dispatched hypercall sets RAX even on error. |
| `shared_info_mapping_pinned` | UAF | gfn_to_hva_cache invalidated on memslot delete. |

### Layer 2: TLA+

`virt/kvm/xen_evtchn.tla`:
- Per-evtchn state ∈ {NotPending, Pending, PendingMasked}.
- Transitions per evtchn-send + EOI + mask/unmask.
- Properties:
  - `safety_masked_no_inject` — Pending+Masked does not inject upcall vector.
  - `liveness_unmasked_pending_eventually_injects` — Pending+!Masked eventually upcall delivered.

`virt/kvm/xen_runstate.tla`:
- Per-vCPU runstate transitions (Running ↔ Runnable ↔ Blocked).
- Properties:
  - `safety_runstate_times_monotonic` — per-state accumulator monotonically increases.
  - `safety_one_state_at_a_time` — runstate single-valued per moment.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VcpuXen::hypercall` post: RAX set; subop dispatched per spec | `VcpuXen::hypercall` |
| `KvmXen::set_evtchn_fast` post: evtchn-pending bit set in shared_info; upcall queued if !masked | `KvmXen::set_evtchn_fast` |
| `VcpuXen::runstate_set_*` post: per-state accumulator advanced; runstate-page written | `VcpuXen::runstate_set_*` |
| Per-VM gfn_to_hva_cache validated for shared_info + vcpu_info + runstate pages | invariants on cache use |

### Layer 4: Verus/Creusot functional

`Xen-PV guest sees Xen ABI semantics within KVM-emulated subset`: per-feature emulation matches Xen public-headers ABI for emulated subset (limited to what KVM implements).

## Hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

Xen-specific reinforcement:

- **KVM_CAP_XEN_HVM per-VM gate** — defense against unintended Xen attack surface enabled.
- **Per-feature flag in KVM_XEN_HVM_CONFIG** — defense against guest using sub-feature (evtchn / runstate / shared_info) without explicit enable.
- **Per-evtchn port range** validated — defense against guest evtchn_send to OOB port.
- **Per-vCPU vcpu_info_pa validated** at register — defense against guest pointing to host kernel memory.
- **Hypercall code/subop validated** — defense against unsupported codes reaching unimplemented paths.
- **Per-vCPU evtchn_received bitmap bounded** — defense against malicious evtchn flood causing per-vCPU memory exhaustion.
- **shared_info gfn_to_hva_cache invalidation** on memslot delete — defense against UAF.
- **Per-VM evtchn_pending_lock spinlock** — defense against concurrent evtchn-fire torn shared_info update.
- **Per-vCPU runstate atomic update via version-counter** — defense against guest reading torn runstate.
- **Per-VM evtchn_eventfd binding ref-counted** — defense against eventfd close-while-active causing UAF.
- **xen-pvclock atomic update via same version-counter scheme as kvmclock** — defense against guest reading torn fields.
- **Long-mode flag** validated against guest CR0 — defense against pointer-size mismatch in hypercall args.
- **Live-migrate clears caches + re-validates on first vmenter** — defense against post-migrate stale HVA.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX / SVM (covered in `x86-vmx.md` / `x86-svm.md` Tier-3s)
- KVM PV-IPI (covered in `x86-pv-ipi.md` Tier-3)
- Hyper-V (covered in `x86-hyperv.md` Tier-3)
- Xen-PV grant-table full mapping (out-of-scope at this Tier-3)
- Xen-PV netfront / blkfront drivers (separate driver concern)
- Implementation code
