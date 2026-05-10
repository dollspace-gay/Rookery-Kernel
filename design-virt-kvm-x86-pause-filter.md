---
title: "Tier-3: arch/x86/kvm/{vmx/vmx.c, svm/svm.c} (PLE / Pause-Filter subset) — KVM lock-holder yield via PAUSE-loop-exiting"
tags: ["tier-3", "virt-kvm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Per-vCPU PAUSE-loop-exiting (Intel VT-x SDM-Vol3 §25.1.3 `PAUSE_LOOP_EXITING`) and SVM Pause-Filter (AMD APM-Vol2 `pause_filter_count` + `pause_filter_thresh`) detect lock-holder-pre-emption: when a guest spinlock attempt fires N consecutive PAUSE instructions within W cycles, vmexit triggers `kvm_vcpu_on_spin` which yields the host-CPU to another vCPU on the same VM (especially the lock-holder). Reduces lock-holder-pre-emption (LHP) effect under over-committed CPU. Critical for: SMP guest performance under host CPU contention.

This Tier-3 covers PLE/Pause-Filter integration in vmx.c + svm.c (~80 lines combined).

### Acceptance Criteria

- [ ] AC-1: Boot Linux SMP guest with `kvm_intel ple_gap=128 ple_window=4096`: PLE active.
- [ ] AC-2: Stress: spinlock contention; tracepoint `kvm_ple_window_update` fires.
- [ ] AC-3: Per-vmexit on PAUSE: ple_window grows toward ple_window_max.
- [ ] AC-4: Per-vmexit on non-PAUSE: ple_window resets to default.
- [ ] AC-5: Per-VM with 4 vCPUs over 2 host CPUs: yield_to fires under contention.
- [ ] AC-6: ple_gap=0: PLE disabled; vmcs PAUSE_LOOP_EXITING bit clear.
- [ ] AC-7: SVM: pause_filter_count decrements per PAUSE within thresh.
- [ ] AC-8: SVM vmexit reason PAUSE: kvm_vcpu_on_spin invoked.
- [ ] AC-9: Per-VM kvm_vcpu_yield_to fails on already-yielding target: returns 0.
- [ ] AC-10: kvm-unit-tests `pause_loop` passes.

### Architecture

Per-VMX module-state:

```
struct VmxOps {
  ple_gap: u32,                                  // module-param
  ple_window: u32,                               // module-param
  ple_window_grow: u32,
  ple_window_shrink: u32,
  ple_window_max: u32,
}
```

Per-VMX-vCPU state:

```
struct VmxVcpu {
  ...
  ple_window: u32,                               // current window
  ple_window_dirty: bool,
}
```

Per-SVM-vCPU state:

```
// Embedded in vmcb.control:
//   pause_filter_count: u16
//   pause_filter_thresh: u16
```

`VmxOps::grow_ple_window(vcpu)`:
1. old = vcpu.ple_window.
2. new = min(old * ple_window_grow, ple_window_max).
3. If new > old: vcpu.ple_window = new; ple_window_dirty = true.

`VmxOps::shrink_ple_window(vcpu)`:
1. old = vcpu.ple_window.
2. new = max(old / ple_window_shrink, ple_window).
3. If new < old: vcpu.ple_window = new; ple_window_dirty = true.

`VmxOps::handle_pause(vcpu)`:
1. If !current_in_kernel: shrink_ple_window(vcpu).
2. kvm_vcpu_on_spin(vcpu, current_kernel_mode()).

`SvmOps::handle_pause(vcpu)`:
1. kvm_vcpu_on_spin(vcpu, current_kernel_mode()).

`Vcpu::on_spin(me, in_kernel)`:
1. start_idx = atomic_add_return(1, &me.kvm.last_boosted_vcpu) % me.kvm.nr_vcpus.
2. for pass in 0..2:
   - for i in 0..me.kvm.nr_vcpus:
     - target = me.kvm.vcpus[(start_idx + i) % nr_vcpus].
     - if target == me: continue.
     - if !target.vcpu_dy_runnable(): continue.
     - if pass == 0 ∧ in_kernel ∧ !target.in_kernel: continue.
     - if Vcpu::yield_to(target): break-pass.

`Vcpu::yield_to(target)`:
1. If !mutex_trylock(&target.mutex): return false.
2. yielded = sched_yield_to(target.thread).
3. mutex_unlock.
4. Return yielded.

### Out of Scope

- KVM core (covered in `kvm-core.md` Tier-3)
- KVM scheduling (covered separately)
- Per-arch PAUSE instruction semantics (out-of-tree)
- Linux scheduler `sched_yield_to` (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `ple_gap` | per-VMX module-param: PAUSE-gap upper bound | `VmxOps::ple_gap` |
| `ple_window` | per-VMX module-param: cycles-per-window | `VmxOps::ple_window` |
| `ple_window_grow/shrink/max` | per-VMX adaptive window | `VmxOps::ple_window_*` |
| `vmx->ple_window` | per-vCPU current window | `VmxVcpu::ple_window` |
| `vmx->ple_window_dirty` | per-vCPU dirty flag | `VmxVcpu::ple_window_dirty` |
| `grow_ple_window()` / `shrink_ple_window()` | per-vCPU adaptive | `VmxOps::grow_ple_window` / `shrink_ple_window` |
| `handle_pause()` | per-VMX vmexit handler | `VmxOps::handle_pause` |
| `vmcb.control.pause_filter_count` | per-SVM count | `Vmcb::pause_filter_count` |
| `vmcb.control.pause_filter_thresh` | per-SVM cycle threshold | `Vmcb::pause_filter_thresh` |
| `kvm_vcpu_on_spin()` | per-vCPU yield-to handler | `Vcpu::on_spin` |
| `kvm_vcpu_yield_to()` | per-vCPU yield to other | `Vcpu::yield_to` |
| `kvm_vcpu_sched_yield_to_holder` | best-fit yield-target select | `Vcpu::sched_yield_to_holder` |

### compatibility contract

REQ-1: VMX `PAUSE_LOOP_EXITING` (vmcs.secondary-controls bit 10):
- Per-vCPU PLE_GAP (vmcs read-only) and PLE_WINDOW (vmcs.read-write).
- HW counts consecutive PAUSE instructions within ple_gap cycles of each other.
- After ple_window cycles in same loop: vmexit reason 40 (PAUSE).

REQ-2: VMX defaults:
- ple_gap = KVM_DEFAULT_PLE_GAP (typically 128).
- ple_window = KVM_VMX_DEFAULT_PLE_WINDOW (typically 4096).
- ple_window_grow = KVM_DEFAULT_PLE_WINDOW_GROW (typically 2).
- ple_window_shrink = KVM_DEFAULT_PLE_WINDOW_SHRINK (typically 0; reset-to-default per-exit).
- ple_window_max = KVM_VMX_DEFAULT_PLE_WINDOW_MAX (typically u32-max-clamped).

REQ-3: VMX adaptive window:
- Per-vmexit on PAUSE: grow_ple_window(vcpu).
- new = min(old × ple_window_grow, ple_window_max).
- If new > old: vcpu.ple_window = new; ple_window_dirty = true.
- Per-vmenter: vmcs.guest_PLE_WINDOW updated if dirty.
- Per-vmexit on non-PAUSE: shrink_ple_window(vcpu) (typically reset-to-ple_window).

REQ-4: SVM Pause-Filter:
- vmcb.control.pause_filter_count: counts consecutive PAUSEs.
- vmcb.control.pause_filter_thresh: cycle gap.
- HW decrements pause_filter_count per PAUSE within thresh; vmexit when 0.
- Per-vmenter: pause_filter_count reset to KVM_PAUSE_FILTER_COUNT (typically 3000).

REQ-5: handle_pause flow:
- Vmexit reason: PAUSE / EXIT_PAUSE.
- Call kvm_vcpu_on_spin(vcpu, in_kernel).

REQ-6: kvm_vcpu_on_spin:
- Iterate per-VM vCPUs.
- Skip self + scheduling-out vCPUs.
- Per-candidate: if vcpu_dy_runnable(target): kvm_vcpu_yield_to(target).
- If yield_to succeeds: vcpu->stat.dy_yields++.

REQ-7: kvm_vcpu_yield_to:
- Acquire target.mutex (try_lock).
- yield_to(target.thread, true).
- Per-yield_to is a sched_yield_to wrapper.

REQ-8: Per-VM CPU.dirty.last_boosted_vcpu:
- Tracks last yield-target to round-robin selection.
- Avoids per-vCPU starvation.

REQ-9: Per-vCPU vcpu_dy_runnable:
- True iff target is in_kernel ∨ vcpu.preempted ∨ guest-spinlock.

REQ-10: Module params:
- /sys/module/kvm_intel/parameters/ple_gap.
- /sys/module/kvm_intel/parameters/ple_window.
- ple_gap=0: disable PLE.
- /sys/module/kvm_amd/parameters/pause_filter_count.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ple_window_within_range` | INVARIANT | per-vcpu.ple_window ∈ [ple_window, ple_window_max]. |
| `ple_grow_monotonic` | INVARIANT | post-grow: new ≥ old. |
| `ple_shrink_monotonic` | INVARIANT | post-shrink: new ≤ old. |
| `yield_target_neq_self` | INVARIANT | per-yield_to target != current vcpu. |
| `last_boosted_vcpu_lt_nr_vcpus` | INVARIANT | per-VM last_boosted_vcpu < nr_vcpus. |

### Layer 2: TLA+

`virt/kvm/pause_filter.tla`:
- Per-vCPU PAUSE → vmexit → on_spin → yield-target → resume.
- Properties:
  - `safety_yield_to_distinct` — per-yield_to: target != source.
  - `safety_no_yield_to_unrunnable` — per-yield: target.runnable.
  - `liveness_progress_under_contention` — per-VM with N vCPUs over M < N CPUs ⟹ each vCPU eventually scheduled.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VmxOps::grow_ple_window` post: vcpu.ple_window ≤ ple_window_max | `VmxOps::grow_ple_window` |
| `VmxOps::handle_pause` post: vcpu.ple_window_dirty cleared at vmenter | `VmxOps::handle_pause` |
| `Vcpu::on_spin` post: yielded-to-target runnable; round-robin start tracked | `Vcpu::on_spin` |
| `Vcpu::yield_to` post: target.mutex released | `Vcpu::yield_to` |

### Layer 4: Verus/Creusot functional

`Per-PAUSE-loop in guest spinlock → vmexit → yield_to(lock-holder vCPU) → guest spinlock unblocks` semantic equivalence: per-LHP-mitigation matches Intel SDM PAUSE_LOOP_EXITING + AMD APM Pause-Filter intent.

### hardening

(Inherits row-1 features from `virt/kvm/kvm-core.md` § Hardening.)

Pause-Filter-specific reinforcement:

- **Per-vCPU ple_window upper-bounded by ple_window_max** — defense against window-grow runaway.
- **Per-vCPU ple_window lower-bounded by ple_window** — defense against window-shrink to 0.
- **Per-VM last_boosted_vcpu round-robin** — defense against per-vCPU starvation.
- **Per-yield_to target.mutex try-lock** — defense against deadlock with concurrent yield.
- **Per-yield_to target ≠ self enforced** — defense against self-yield NOP-loop.
- **Per-yield_to target.dy_runnable check** — defense against yielding to halted target wasting cycles.
- **Per-VM ple_gap = 0 disables PLE** — defense against unwanted PLE-vmexit overhead.
- **Per-vCPU ple_window_dirty atomic flush at vmenter** — defense against stale window after grow.
- **Per-vCPU pause_filter_count reset at vmenter (SVM)** — defense against guest pre-loading false-positive count.
- **Per-yield-pass distinguishes in_kernel vs user** — defense against yielding to user-mode vCPU when host expected kernel-mode lock-holder.

