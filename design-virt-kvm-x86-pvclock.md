---
title: "Tier-3: arch/x86/kvm/x86.c — kvmclock paravirt clock (per-vCPU pvclock-page + MSR_KVM_SYSTEM_TIME + WALL_CLOCK)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

kvmclock is the KVM paravirt clock that gives guest a stable, non-jittery monotonic clocksource: KVM publishes per-vCPU time-keeping data (TSC frequency, current TSC value, system_time accumulator) to a guest-mapped 4KiB page; guest reads atomically with sequence-counter and computes monotonic-time as `system_time + (rdtsc - tsc_timestamp) * tsc_to_system_mul`. Critical for VMs because TSC alone is insufficient (per-vCPU TSC-offset on migration, host-paused-while-guest-running, rdtsc precision differences across host CPUs); kvmclock abstracts these. Host updates pvclock-page on: vCPU-create, host-CPU-frequency change (CPU-freq governor), VM-pause/resume, live-migration. Guest detects via CPUID 0x40000001:EAX bit KVM_FEATURE_CLOCKSOURCE2.

This Tier-3 covers kvmclock-related code in `arch/x86/kvm/x86.c` (~14571 total; kvmclock relevant sections totalling ~600-800 lines).

### Acceptance Criteria

- [ ] AC-1: Boot Linux guest with `clocksource=kvm-clock`: dmesg shows kvmclock active.
- [ ] AC-2: Guest `clock_gettime(CLOCK_MONOTONIC)` accuracy: < 100ns drift over 60s vs host (TSC-stable).
- [ ] AC-3: Live migration: post-migrate, guest CLOCK_MONOTONIC continuous (no negative jumps).
- [ ] AC-4: Host CPU-freq change: per-vCPU regen detected; guest sees consistent rate.
- [ ] AC-5: VM pause/resume: PVCLOCK_GUEST_STOPPED bit toggles; guest absorbs pause-duration.
- [ ] AC-6: PVCLOCK_TSC_STABLE: TSC-stable host advertises bit; TSC-unstable host does not.
- [ ] AC-7: kvm-unit-tests `pvclock` test passes.
- [ ] AC-8: 16-vCPU guest: all vCPUs see identical CLOCK_MONOTONIC offsets (within nanosecond tolerance).
- [ ] AC-9: Spec-violation defense: WRMSR(MSR_KVM_SYSTEM_TIME) with non-page-aligned addr → no enable; KVM logs.

### Architecture

`PvclockVcpuTimeInfo` packed struct (UAPI; per-Intel KVM PV ABI):

```
#[repr(C, packed)]
struct PvclockVcpuTimeInfo {
  version: u32,
  pad0: u32,
  tsc_timestamp: u64,
  system_time: u64,
  tsc_to_system_mul: u32,
  tsc_shift: i8,
  flags: u8,
  pad1: [u8; 2],
}
```

Per-vCPU additions to `VcpuArch`:

```
struct VcpuArch {
  ...
  time: u64,                                  // pvclock-page guest-PA + enable-bit
  pv_time_enabled: bool,
  hv_clock: PvclockVcpuTimeInfo,
  pvclock_seq: AtomicU32,
  this_tsc_nsec: u64,                         // last-update system_time
  this_tsc_write: u64,                        // last-update tsc_timestamp
  last_guest_tsc: u64,
  tsc_offset: u64,
  ...
}
```

`Vcpu::guest_time_update` flow:
1. now_ns := ktime_get_boot_ns().
2. now_tsc := rdtsc().
3. host_tsc_clocksource := use vsyscall-tsc if available (for nanosecond-precision via TSC).
4. Compute system_time = now_ns relative to vm_start_ns.
5. Compose hv_clock fields:
   - tsc_timestamp = now_tsc.
   - system_time = system_time.
   - tsc_to_system_mul + tsc_shift from per-VM masterclock.
   - flags |= PVCLOCK_TSC_STABLE if host clocksource stable.
6. seq := atomic_inc(&pvclock_seq) | 1.
7. hv_clock.version = seq.
8. memcpy_to_guest(vcpu, time-addr, &hv_clock, sizeof).
9. atomic_inc(&pvclock_seq); seq is now even.
10. memcpy_to_guest(vcpu, time-addr + offset_of(version), &seq, sizeof(u32)).

`KvmClock::do_base(kvm, &system_time, &tsc_timestamp)`:
1. Acquire kvm.arch.tsc_lock.
2. tsc_timestamp = rdtsc() relative to kvm.arch.master_cycle_now.
3. system_time = ktime_get_boot_ns() relative to kvm.arch.master_kernel_ns.
4. Release lock.

`KvmClock::cpufreq_notifier(action, freq, cpu)`:
1. CPUFREQ_PRECHANGE: per-vCPU on this CPU: kvm_make_request(KVM_REQ_CLOCK_UPDATE, vcpu).
2. CPUFREQ_POSTCHANGE: re-init per-VM tsc_to_system_mul; mark all vCPUs for regen.

`Vcpu::handle_msr` MSR_KVM_SYSTEM_TIME case:
1. addr := data & ~1; enable := data & 1.
2. If !enable: vcpu.arch.pv_time_enabled = false.
3. If enable:
   - Validate addr aligned + within memslot.
   - vcpu.arch.time = data.
   - vcpu.arch.pv_time_enabled = true.
   - kvm_make_request(KVM_REQ_CLOCK_UPDATE, vcpu).

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- VMX TSC offset details (covered in `x86-vmx.md` Tier-3)
- SVM TSC offset details (covered in `x86-svm.md` Tier-3)
- LAPIC timer (covered in `x86-lapic.md` Tier-3)
- Hyper-V REFERENCE_TSC (covered in `x86-hyperv.md` future Tier-3)
- Xen pvclock (covered in `x86-xen.md` future Tier-3)
- Guest-side clocksource integration (guest-driver concern)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pvclock_vcpu_time_info` | per-vCPU 32-byte pvclock-page payload | `kernel::kvm::x86::pvclock::PvclockVcpuTimeInfo` |
| `struct pvclock_wall_clock` | per-VM wallclock-snapshot | `PvclockWallClock` |
| `MSR_KVM_SYSTEM_TIME` (0x12) / `_NEW` (0x4b564d01) | guest writes pvclock-page address | (handled in `Vcpu::handle_msr`) |
| `MSR_KVM_WALL_CLOCK` (0x11) / `_NEW` (0x4b564d00) | guest writes wallclock-page address | same |
| `KVM_FEATURE_CLOCKSOURCE` / `_CLOCKSOURCE2` | CPUID feature bits | covered in `x86-cpuid.md` |
| `kvm_setup_pvclock_page(vcpu)` | per-vCPU compute + write pvclock-page | `Vcpu::setup_pvclock_page` |
| `kvm_guest_time_update(vcpu)` | per-vCPU pvclock recompute | `Vcpu::guest_time_update` |
| `do_kvmclock_base(t, tsc_timestamp)` | per-VM compute system_time + tsc | `KvmClock::do_base` |
| `__get_kvmclock(kvm, data)` / `get_kvmclock(...)` | per-VM kvm_clock_data snapshot | `KvmClock::get_data` |
| `kvm_gen_kvmclock_update(vcpu)` | mark per-vCPU pvclock-page dirty | `Vcpu::gen_kvmclock_update` |
| `kvmclock_reset(vcpu)` | per-vCPU reset on INIT | `Vcpu::kvmclock_reset` |
| `kvm_set_msr_common` MSR_KVM_SYSTEM_TIME case | MSR write handler | covered in vcpu MSR handler |
| `kvm_get_msr_common` MSR_KVM_WALL_CLOCK case | MSR read handler | same |
| `kvm_arch_para_features(vcpu)` | KVM-paravirt feature bitmap | `Vcpu::kvm_para_features` |
| `kvmclock_cpufreq_notifier` | host CPU-freq change notify | `KvmClock::cpufreq_notifier` |
| `pvclock_clocksource_read(src)` (guest-side) | guest atomic-read helper | (guest-only; out-of-scope) |
| `KVM_REQ_CLOCK_UPDATE` / `KVM_REQ_GLOBAL_CLOCK_UPDATE` | per-vCPU request markers | `Vcpu::REQ_CLOCK_UPDATE` |

### compatibility contract

REQ-1: Per-vCPU pvclock-page (4KiB, guest-physical-addr from MSR_KVM_SYSTEM_TIME):
- 32-byte `pvclock_vcpu_time_info` at offset 0:
  - version (u32; even = stable, odd = update-in-progress).
  - pad0 (u32).
  - tsc_timestamp (u64; host TSC at last update).
  - system_time (u64; ns since boot at last update).
  - tsc_to_system_mul (u32; TSC freq → ns scaling).
  - tsc_shift (i8).
  - flags (u8 bitmap: PVCLOCK_TSC_STABLE / _GUEST_STOPPED).
  - pad1 (u16).
- Rest of page: reserved.

REQ-2: Per-VM wallclock-page:
- 4KiB at MSR_KVM_WALL_CLOCK addr.
- 12-byte `pvclock_wall_clock` at offset 0:
  - version (u32).
  - sec (u32).
  - nsec (u32).

REQ-3: Guest WRMSR(MSR_KVM_SYSTEM_TIME{,_NEW}) handler:
1. Validate addr 4KiB-aligned + accessible per memslot.
2. vcpu.arch.time = addr | enable-bit.
3. Mark KVM_REQ_CLOCK_UPDATE; next vmenter rebuild.

REQ-4: Per-vCPU `kvm_setup_pvclock_page` flow:
1. Compute current system_time + tsc_timestamp.
2. Compose pvclock_vcpu_time_info struct.
3. seq = atomic_inc(&vcpu.arch.pvclock_seq) | 1 (set odd bit).
4. Write `version = seq` first.
5. Write rest of fields.
6. seq++ (now even).
7. Write `version = seq`.
8. Guest reads atomically via:
   - Loop: v = version; if v & 1: retry; read fields; v2 = version; if v != v2: retry; OK.

REQ-5: tsc_to_system_mul + tsc_shift compute:
- nanos_per_tsc_tick = 10^9 / tsc_khz.
- mul + shift such that `(tsc * mul) >> shift == nanos`.
- Specifically: shift chosen so mul fits 32-bit; mul = (10^9 << shift) / tsc_khz / 1000.

REQ-6: Host CPU-freq change handling (`kvmclock_cpufreq_notifier`):
1. CPUFREQ_PRECHANGE: per-vCPU mark KVM_REQ_CLOCK_UPDATE.
2. CPUFREQ_POSTCHANGE: re-compute tsc_to_system_mul; per-vCPU regen.

REQ-7: Live migration:
- Pre-migrate: KVM_KVMCLOCK_CTRL ioctl signals guests to pause kvmclock.
- During-migrate: KVM_GET_CLOCK + KVM_SET_CLOCK move per-VM time origin.
- Post-migrate: per-vCPU regen pvclock-page with new tsc_offset.

REQ-8: TSC-stable vs unstable:
- TSC-stable host: PVCLOCK_TSC_STABLE bit set; guest may use rdtsc directly.
- TSC-unstable: bit cleared; guest must read pvclock-page on every clocksource access (slower).
- Determined per-VM at vCPU init based on host clocksource state.

REQ-9: Hyper-V time interop:
- Hyper-V time-stamp-counter MSR (0x40000020) related path; KVM may emulate Hyper-V REFERENCE_TSC alongside kvmclock if KVM_CAP_HYPERV.

REQ-10: Per-vCPU TSC offset (vmcs.tsc_offset for VMX; vmcb.tsc_offset for SVM):
- guest_tsc = host_tsc + tsc_offset.
- KVM adjusts tsc_offset per-vCPU on VM-load to make guest see consistent virtual TSC.

REQ-11: Guest-stopped bit (PVCLOCK_GUEST_STOPPED):
- Set when host pauses VM; guest sees and adjusts.
- Cleared on resume.

REQ-12: KVM_REQ_GLOBAL_CLOCK_UPDATE per-VM:
- After cross-VM config change (e.g., master_kernel_ns_offset), all vCPUs need pvclock regen.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pvclock_addr_aligned` | INVARIANT | per-vCPU pvclock-page addr 4KiB-aligned; defense against straddling write. |
| `version_seq_odd_during_update` | INVARIANT | version always odd during update; even when stable; per-Intel-KVM-spec atomicity. |
| `tsc_to_system_mul_no_overflow` | INVARIANT | (tsc * mul) >> shift fits u64 for any tsc within u64-half range. |
| `pvclock_page_in_memslot` | INVARIANT | per-vCPU pvclock-page guest-PA in valid memslot. |

### Layer 2: TLA+

`virt/kvm/pvclock_atomic_update.tla`:
- States per per-vCPU pvclock: Stable, UpdateInProgress.
- Properties:
  - `safety_seq_odd_iff_in_progress` — version-bit-0 is 1 iff state == UpdateInProgress.
  - `safety_guest_atomic_read` — guest seq-loop terminates with consistent fields if host updates eventually quiesce.
  - `liveness_eventual_quiescence` — assuming bounded host-updates, guest read eventually succeeds.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::guest_time_update` post: pvclock-page version even after final write; tsc_timestamp matches captured rdtsc | `Vcpu::guest_time_update` |
| Per-VM master_kernel_ns + master_cycle_now consistent across vCPUs after global-clock-update | `Kvm::sync_global_clock` |
| `KvmClock::cpufreq_notifier` post: all online vCPUs marked KVM_REQ_CLOCK_UPDATE | `KvmClock::cpufreq_notifier` |
| Per-vCPU tsc_offset adjusted post-migrate so guest TSC continues monotonic | `Vcpu::adjust_tsc_offset` |

### Layer 4: Verus/Creusot functional

`Guest CLOCK_MONOTONIC = (rdtsc - tsc_timestamp) * tsc_to_system_mul / 2^tsc_shift + system_time → matches host CLOCK_BOOTTIME during run` semantic equivalence: per-vCPU virtual clock progresses linearly with host time during run; pauses correctly during VM-pause.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

kvmclock-specific reinforcement:

- **Version sequence-counter atomic update** — defense against guest reading torn pvclock fields.
- **Per-vCPU pvclock-page addr validated** at WRMSR — defense against guest pointing to host kernel memory.
- **Per-VM tsc_to_system_mul recomputed on cpufreq change** — defense against rate skew on dynamic-cpufreq host.
- **PVCLOCK_TSC_STABLE bit gated on host clocksource state** — defense against guest using rdtsc directly when host TSC unstable.
- **PVCLOCK_GUEST_STOPPED set during VM-pause** — defense against guest computing wrong elapsed-time across host-pause.
- **Per-vCPU tsc_offset adjusted on cross-CPU migrate** — defense against TSC-deviation between host-CPUs causing guest TSC backwards-jump.
- **Per-VM kvm.arch.tsc_lock** for master_*-clock sync — defense against concurrent global-clock-update racing per-vCPU update.
- **KVM_KVMCLOCK_CTRL ioctl for graceful pause** — defense against migrate causing guest unexpected time-jump.
- **memcpy_to_guest under page-pinning** — defense against guest unmapping pvclock-page mid-update.
- **MSR_KVM_SYSTEM_TIME enable-bit** — defense against half-configured pvclock causing incoherent state.
- **Per-vCPU pvclock_seq odd-during-update** — defense against guest reading mid-update value.

