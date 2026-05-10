---
title: "Tier-3: arch/x86/kvm/x86.c (TSC subset) — KVM TSC virtualization (per-vCPU TSC offset + scaling + adjustment + clocksource)"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

KVM TSC (TimeStamp Counter) virtualization gives each guest a stable virtual TSC even across host TSC differences (per-CPU TSC drift, live-migration, host-CPU-frequency change). Per-vCPU `tsc_offset` (vmcs.TSC_OFFSET on VMX; vmcb.tsc_offset on SVM) added to host RDTSC by CPU before delivery to guest. Per-vCPU `tsc_scaling_ratio` (vmcs.TSC_MULTIPLIER) lets guest see a frequency-scaled TSC vs host. KVM tracks per-vCPU per-VM master TSC for synchronization across vCPUs + adjusts on cross-CPU migration. MSR_IA32_TSC writes accepted via emulation; MSR_IA32_TSC_ADJUST per-vCPU offset for host-side resync. Critical for: kvmclock paravirt accuracy, guest TSC-deadline timer, live migration time-continuity.

This Tier-3 covers TSC code in `arch/x86/kvm/x86.c` (~500-700 lines).

### Acceptance Criteria

- [ ] AC-1: Boot Linux guest: rdtsc returns offset+host_tsc; subsequent rdtsc reads monotonically increase.
- [ ] AC-2: TSC-stable host: KVM advertises PVCLOCK_TSC_STABLE; guest kvmclock uses TSC directly.
- [ ] AC-3: TSC scaling: configure vCPU at 2GHz on 3GHz host; guest perceives 2GHz TSC rate.
- [ ] AC-4: MSR_IA32_TSC write: guest sets TSC=0; subsequent reads start near 0.
- [ ] AC-5: Live migration: pre-migrate TSC=X; post-migrate guest reads TSC continues from X (no negative jump).
- [ ] AC-6: Cross-CPU migrate: vCPU moves CPU; TSC adjustment compensates per-CPU drift.
- [ ] AC-7: KVM_GET_CLOCK / KVM_SET_CLOCK round-trip preserves master_clock state.
- [ ] AC-8: Multi-vCPU TSC-sync: 16-vCPU guest writes MSR_IA32_TSC to same value; KVM matches them as synchronized.
- [ ] AC-9: TSC-deadline timer: guest writes MSR_IA32_TSC_DEADLINE; LAPIC fires when guest TSC reaches.
- [ ] AC-10: kvm-unit-tests `tsc` test passes.

### Architecture

`Vcpu::compute_l1_tsc_offset(vcpu, target_tsc)`:
1. host_tsc := rdtsc_ordered().
2. If !tsc_scaling_active: return target_tsc - host_tsc.
3. Else: scaled_host := scale(host_tsc, tsc_scaling_ratio); return target_tsc - scaled_host.

`Vcpu::write_tsc_offset(vcpu, offset)`:
1. vcpu.arch.l1_tsc_offset = offset.
2. trace_kvm_write_tsc_offset(vcpu_id, prev_offset, new_offset).
3. If is_guest_mode (L2): vcpu.arch.tsc_offset = calc_nested_tsc_offset(offset, l2_offset, l2_multiplier).
4. Else: vcpu.arch.tsc_offset = offset.
5. Vendor-specific: vmx_write_tsc_offset(vcpu, vcpu.arch.tsc_offset) OR svm_write_tsc_offset.

`VcpuVmx::write_tsc_offset(vcpu, offset)`:
1. vmcs_write64(TSC_OFFSET, offset).

`VcpuSvm::write_tsc_offset(vcpu, offset)`:
1. svm.vmcb01.control.tsc_offset = offset.
2. vmcb_mark_dirty(svm.vmcb01, INTERCEPTS).

`Vcpu::set_tsc_khz(vcpu, user_khz)`:
1. host_tsc_khz := host_clock_tsc_khz.
2. ratio := compute_scaling(user_khz, host_tsc_khz).
3. vcpu.arch.tsc_scaling_ratio = ratio.
4. vendor: vmx_write_tsc_multiplier(vcpu, ratio) OR svm.vmcb.control.tsc_scale = ratio.

`Vcpu::synchronize_tsc(vcpu, src_vcpu, data, &ns, &elapsed)`:
1. Now := ktime_get_ns().
2. delta_ns := Now - src_vcpu.this_tsc_nsec.
3. If delta_ns < SYNC_THRESHOLD AND data near src_vcpu.this_tsc_write:
   - Treat as synchronized; update vcpu.this_tsc_generation = src.this_tsc_generation.
   - kvm.arch.nr_vcpus_matched_tsc++.
4. Else:
   - New generation; vcpu.this_tsc_generation = ++kvm.arch.cur_tsc_generation.

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- kvmclock paravirt (covered in `x86-pvclock.md` Tier-3)
- LAPIC TSC-deadline timer (covered in `x86-lapic.md` Tier-3)
- VMX vendor (covered in `x86-vmx.md` Tier-3)
- SVM vendor (covered in `x86-svm.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `kvm_compute_l1_tsc_offset(vcpu, target_tsc)` | per-vCPU offset compute | `Vcpu::compute_l1_tsc_offset` |
| `kvm_calc_nested_tsc_offset(...)` | nested L2 offset compute | `Vcpu::calc_nested_tsc_offset` |
| `kvm_vcpu_write_tsc_offset(vcpu, offset)` | per-vCPU offset write to vmcs/vmcb | `Vcpu::write_tsc_offset` |
| `vmx_write_tsc_offset(vcpu, offset)` (vmx.c) | VMX vmcs.TSC_OFFSET write | `VcpuVmx::write_tsc_offset` |
| `svm_write_tsc_offset(vcpu, offset)` (svm.c) | SVM vmcb.tsc_offset write | `VcpuSvm::write_tsc_offset` |
| `vmx_write_tsc_multiplier(vcpu, multiplier)` | VMX TSC scaling | `VcpuVmx::write_tsc_multiplier` |
| `kvm_set_tsc_khz(vcpu, user_khz)` | per-vCPU TSC kHz set | `Vcpu::set_tsc_khz` |
| `kvm_get_tsc_khz(vcpu)` | per-vCPU TSC kHz get | `Vcpu::get_tsc_khz` |
| `kvm_arch_vcpu_load(vcpu, cpu)` (TSC adjust on cross-CPU) | per-vCPU load TSC offset | covered in vendor |
| `adjust_tsc_offset_host(vcpu, adjustment)` | per-vCPU offset adjust | `Vcpu::adjust_tsc_offset_host` |
| `adjust_tsc_offset_guest(vcpu, adjustment)` | per-vCPU TSC_ADJUST guest write | `Vcpu::adjust_tsc_offset_guest` |
| `kvm_synchronize_tsc(vcpu, src_vcpu, data, &ns, &elapsed)` | per-VM TSC-master coordination | `Vcpu::synchronize_tsc` |
| `kvm_track_tsc_matching(vcpu, ...)` | per-VM TSC-match tracking | `Kvm::track_tsc_matching` |
| `MSR_IA32_TSC` (0x10) | guest TSC MSR | UAPI |
| `MSR_IA32_TSC_ADJUST` (0x3B) | per-vCPU TSC_ADJUST MSR | UAPI |
| `MSR_IA32_TSC_DEADLINE` (0x6E0) | TSC-deadline timer | (covered in lapic Tier-3) |

### compatibility contract

REQ-1: Per-vCPU TSC state in `vcpu.arch`:
- `l1_tsc_offset` (per-L1 TSC offset; passed to vmcs/vmcb).
- `tsc_offset` (effective offset for current mode; L1 OR L1+L2 nested combine).
- `tsc_scaling_ratio` (vmcs.TSC_MULTIPLIER; 64-bit fixed-point).
- `tsc_offset_adjustment` (per-vCPU adjustment from host TSC drift).
- `tsc_catchup` (bool: catch up after pause).
- `tsc_always_catchup` (bool: aggressive catchup).
- `last_guest_tsc` (last-observed guest TSC).
- `last_host_tsc` (last-observed host TSC).
- `this_tsc_generation` / `this_tsc_nsec` / `this_tsc_write` (per-VM master tracking).
- `arch_capabilities` (TSC-related cap bits).

REQ-2: Per-VM TSC state in `kvm.arch`:
- `master_kernel_ns` (master ns at last-update).
- `master_cycle_now` (master TSC at last-update).
- `cur_tsc_generation` / `cur_tsc_nsec` / `cur_tsc_write` (master tracking).
- `nr_vcpus_matched_tsc` (count of vCPUs with matched TSC).

REQ-3: Per-vCPU TSC offset compute (`kvm_compute_l1_tsc_offset`):
1. host_tsc := rdtsc().
2. Compute desired guest_tsc := target_tsc.
3. offset := guest_tsc - host_tsc.
4. Apply scaling: if tsc_scaling_ratio != default: scale offset.
5. Return offset.

REQ-4: Per-vCPU offset write:
- VMX: VMWRITE vmcs.TSC_OFFSET = offset.
- SVM: vmcb.control.tsc_offset = offset.
- Per-vmenter: CPU adds offset to host TSC before guest sees.

REQ-5: TSC scaling:
- Per-vCPU tsc_scaling_ratio (Q48-Q16 fixed-point).
- VMX: VMWRITE vmcs.TSC_MULTIPLIER.
- SVM: vmcb.control.tsc_scale.
- Hardware-supported (CPUID feature); if not: KVM emulates via offset-recompute.

REQ-6: MSR_IA32_TSC handling:
- Guest WRMSR(MSR_IA32_TSC, X): set offset such that guest sees X starting from now.
- Guest RDMSR(MSR_IA32_TSC) / RDTSC: pass-through (with offset/scaling applied by HW).

REQ-7: MSR_IA32_TSC_ADJUST:
- Per-vCPU adjustment counter (per Intel arch-spec).
- Tracks cumulative TSC adjustments.
- Used by guest scheduler to detect drift.

REQ-8: Per-VM TSC synchronization:
- When 2+ vCPUs write same TSC value within small window: KVM treats as "synchronized" (likely host syncing).
- nr_vcpus_matched_tsc tracks; when all match: kvm.arch.cur_tsc_* updated.
- Used to keep master_clock TSC-stable assumption valid.

REQ-9: Cross-CPU vCPU migration TSC adjust:
- Different host CPUs may have small TSC delta.
- adjust_tsc_offset_host accumulates into vcpu.arch.tsc_offset_adjustment.
- Per-vmenter: offset += adjustment.

REQ-10: kvmclock dependence:
- kvmclock uses guest TSC + per-VM master_kernel_ns/master_cycle_now to derive monotonic-time.
- Stable iff PVCLOCK_TSC_STABLE_BIT set.
- kvmclock + TSC-virtualization tightly coupled (covered in pvclock Tier-3).

REQ-11: Live migration:
- Per-vCPU TSC + offset migrated.
- Destination KVM adjusts offset to maintain TSC continuity on resume.

REQ-12: KVM_GET_CLOCK / KVM_SET_CLOCK ioctl:
- Per-VM master_clock state migrated.
- Used by qemu live-migration.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tsc_offset_arithmetic` | INVARIANT | offset = guest_tsc - host_tsc * scaling; bounded u64 wrap-safe. |
| `tsc_scaling_ratio_valid` | INVARIANT | ratio in valid range (e.g., max 16x scale per Intel); defense against arithmetic-overflow. |
| `nested_tsc_offset_compute` | INVARIANT | L2 offset combines L1 offset + L2 multiplier per AMD/Intel spec. |
| `master_clock_consistent` | INVARIANT | per-VM master_kernel_ns + master_cycle_now updated atomically. |
| `tsc_adjust_atomic` | INVARIANT | per-vCPU TSC adjustment via cmpxchg; defense against cross-CPU race. |

### Layer 2: TLA+

`virt/kvm/tsc_offset_compute.tla`:
- Per-vCPU offset state: { l1_offset, tsc_offset, scaling_ratio }.
- Properties:
  - `safety_offset_correct_for_target_tsc` — offset chosen yields target_tsc on next guest RDTSC.
  - `safety_scaling_consistent` — scaling applied consistently across L1/L2.
  - `liveness_synchronize_tsc_eventually_matches` — assuming bounded host-clock-drift, sync eventually achieves match.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vcpu::compute_l1_tsc_offset` post: returned offset matches target_tsc - host_tsc relationship | `Vcpu::compute_l1_tsc_offset` |
| `Vcpu::write_tsc_offset` post: vmcs/vmcb written; vendor-specific applied | `Vcpu::write_tsc_offset` |
| `Vcpu::set_tsc_khz` post: scaling_ratio derived from kHz + applied to vendor | `Vcpu::set_tsc_khz` |
| `Vcpu::synchronize_tsc` post: per-vCPU generation incremented OR matched-set | `Vcpu::synchronize_tsc` |

### Layer 4: Verus/Creusot functional

`Per-vCPU: guest RDTSC = host_tsc * scaling + offset` semantic equivalence: per-RDTSC the guest-observed value matches scaling*host + offset deterministically.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

TSC-virtualization-specific reinforcement:

- **Per-vCPU offset arithmetic wrap-safe** — defense against u64-overflow on extreme offsets.
- **TSC scaling ratio bounded** — defense against arithmetic-overflow in scale-multiply.
- **Per-VM master_clock under tsc_lock** — defense against cross-vCPU torn master state.
- **Per-CPU rdtsc_ordered()** — defense against speculative RDTSC reordering.
- **Synchronize_tsc threshold tuned** — defense against false-positive sync detection.
- **Live-migrate TSC continuity** — defense against post-migrate negative-jump breaking guest scheduler.
- **Cross-CPU migrate TSC adjust** — defense against TSC-skew across host CPUs.
- **MSR_IA32_TSC_ADJUST tracking** — defense against guest detecting silent drift.
- **TSC scaling fallback to offset-recompute** — defense against missing-CPU-feature.
- **Per-vCPU tsc_catchup gating** — defense against pathological catchup-loop after pause.
- **Nested-virt L2 offset combined correctly** — defense against L2 reading wrong TSC.

