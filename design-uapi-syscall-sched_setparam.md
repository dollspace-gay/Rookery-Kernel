---
title: "Tier-5 syscall: sched_setparam(2) — syscall 142"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sched_setparam(2)` mutates the **scheduling parameters** (i.e., the `sched_priority` field of `struct sched_param`) of a target task **without changing its policy class**. The implementation is a thin wrapper over the internal `__sched_setscheduler()` helper invoked with the task's existing policy and the new `sched_param`: this means the per-class admission rules of `sched_setscheduler(2)` apply unchanged — SCHED_FIFO/SCHED_RR require `sched_priority ∈ [1,99]`, SCHED_NORMAL/SCHED_BATCH/SCHED_IDLE require `sched_priority == 0`, and SCHED_DEADLINE tasks **cannot** be reparameterised via this syscall (must use `sched_setattr(2)`). The syscall predates `sched_setscheduler(2)` in the POSIX.1b-1993 real-time-extensions specification and survives in Linux mainly so that real-time-aware applications can re-tune RT priority during steady-state without re-asserting (and possibly re-validating) the policy. Critical for: RT priority boost/demote, audio-thread dynamic tuning (JACK, PipeWire), industrial-control priority inheritance, IRQ-thread tunables (`/sys/kernel/irq/*/threaded_priority`), sandbox privilege-drop that nudges priority while leaving policy intact.

This Tier-5 covers the userspace ABI of syscall 142. The scheduler-class admission machinery is owned by `kernel/sched/core.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: `sched_setparam(0, &{50})` on SCHED_FIFO self: succeeds; readback via `sched_getparam` returns `50`.
- [ ] AC-2: `sched_setparam(0, &{0})` on SCHED_NORMAL self: succeeds; readback returns `0`.
- [ ] AC-3: `sched_setparam(0, &{50})` on SCHED_NORMAL self: returns `-EINVAL` (priority must be 0 for fair class).
- [ ] AC-4: `sched_setparam(0, &{0})` on SCHED_FIFO self: returns `-EINVAL` (RT class requires ≥ 1).
- [ ] AC-5: `sched_setparam(0, NULL)`: returns `-EFAULT`.
- [ ] AC-6: `sched_setparam(target_other_uid, ...)` from unprivileged caller: returns `-EPERM`.
- [ ] AC-7: `sched_setparam` on a SCHED_DEADLINE task: returns `-EINVAL`.
- [ ] AC-8: Raising RT priority without `CAP_SYS_NICE`: returns `-EPERM`.
- [ ] AC-9: Lowering same-uid task's RT priority without `CAP_SYS_NICE`: succeeds (POSIX-compatible).
- [ ] AC-10: Concurrent `sched_setparam` from two threads: linearisable; final state is one of the two requests.
- [ ] AC-11: `sched_setparam(non_existent_pid, ...)`: returns `-ESRCH`.
- [ ] AC-12: `param` pointer in unmapped page: returns `-EFAULT`.

### Architecture

Rookery surface in `kernel/sched/syscalls.rs`:

```rust
pub fn sys_sched_setparam(pid: i32, param: UserPtr<SchedParam>) -> isize {
    /* fault-safe copy */
    let new = match param.copy_from_user() {
        Ok(p) => p,
        Err(_) => return -EFAULT as isize,
    };
    let target = match Task::find_in_pidns(current().pid_ns(), pid) {
        Some(t) => t,
        None    => return -ESRCH as isize,
    };
    /* Preserve current policy; rely on inner helper for full validation */
    let policy = target.policy();
    Sched::set_scheduler_inner(&target, policy, &new, /*user=*/true, /*pi=*/true)
        .map(|_| 0)
        .unwrap_or_else(|e| -(e as isize))
}
```

Validation flow (per-class):

```rust
fn validate(policy: u32, param: &SchedParam) -> Result<(), Errno> {
    match policy {
        SCHED_FIFO | SCHED_RR => {
            if !(1..=99).contains(&param.sched_priority) { Err(EINVAL) } else { Ok(()) }
        }
        SCHED_NORMAL | SCHED_BATCH | SCHED_IDLE => {
            if param.sched_priority != 0 { Err(EINVAL) } else { Ok(()) }
        }
        SCHED_DEADLINE => Err(EINVAL),
        _ => Err(EINVAL),
    }
}
```

### Out of Scope

- `kernel/sched/core.md` Tier-3: `__sched_setscheduler` admission, rq enqueue.
- `kernel/sched/rt.md` Tier-3: RT class.
- `kernel/sched/deadline.md` Tier-3: SCHED_DEADLINE admission.
- `sched_setscheduler.md` sibling.
- `sched_setattr.md` sibling (modern, full-struct).
- `sched_getparam.md` sibling.
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
int sched_setparam(pid_t pid, const struct sched_param *param);

struct sched_param {
    int sched_priority;
};
```

Rust ABI shim:

```rust
pub fn sys_sched_setparam(pid: i32, param: *const SchedParam) -> isize;
```

Syscall number: **142** (x86_64). Generic syscall table: **118**.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `pid` | `pid_t` | IN | Target task in caller's pid-ns; `0` = self. |
| `param` | `const struct sched_param *` | IN (UAPI ptr) | Pointer to `sched_param`; copied in via `copy_from_user`. |

### return

- **Success**: `0`.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `param == NULL`; `sched_priority` out of range for current policy; target's current policy is SCHED_DEADLINE. |
| `EPERM` | Caller lacks `CAP_SYS_NICE` and is raising RT priority, or target task is not owned by caller's effective uid (when not privileged). |
| `ESRCH` | No task with given pid in caller's pid-ns. |
| `EFAULT` | `param` unmapped. |

### abi surface

```text
__NR_sched_setparam (x86_64)  = 142
__NR_sched_setparam (i386)    = 154
__NR_sched_setparam (arm64)   = 118  (generic-syscall)
__NR_sched_setparam (generic) = 118

/* Priority bounds — re-applied per existing policy */
SCHED_FIFO/RR                            : 1..99
SCHED_NORMAL/BATCH/IDLE/DEADLINE         : 0 (must be 0)
SCHED_DEADLINE                           : rejected (use sched_setattr)
```

### compatibility contract

REQ-1: Syscall number is **142** on x86_64, **118** on arm64/generic. ABI-stable since 2.0.

REQ-2: `pid == 0` ⟹ target current task.

REQ-3: `param == NULL` ⟹ `-EFAULT` (per POSIX historically `-EINVAL`; Linux returns `-EFAULT` from `copy_from_user`).

REQ-4: Policy is **preserved**: implementation reads `target->policy` and re-invokes `__sched_setscheduler(target, target->policy, &new_param, true /* user */, true /* pi */)`.

REQ-5: SCHED_DEADLINE target ⟹ `-EINVAL` (deadline parameters cannot be expressed in `sched_param`).

REQ-6: Priority validation re-uses `sched_setscheduler(2)` per-class rules:
- RT classes (FIFO/RR): `sched_priority ∈ [1,99]`.
- Fair / batch / idle: `sched_priority == 0`.

REQ-7: Permission check `__sched_setscheduler`:
- Raising RT priority OR cross-uid target ⟹ require `CAP_SYS_NICE` in target's user-ns.
- Same-uid, non-RT lowering: permitted.

REQ-8: PI-boost interaction: caller may not be holding a PI-boosted priority on the target's lock chain; if the change would invert PI propagation, the scheduler walks the lock chain to re-propagate.

REQ-9: Inheritance: `fork(2)` may clear modifier per `SCHED_RESET_ON_FORK` on the policy; `execve(2)` preserves policy + parameters.

REQ-10: Concurrent `sched_setparam` / `sched_setscheduler` on same target are serialised under `target->pi_lock` + `rq->lock`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `policy_preserved` | INVARIANT | post: `target.policy` unchanged. |
| `priority_range_enforced` | INVARIANT | RT ⟹ 1..=99; fair ⟹ 0. |
| `deadline_rejected` | INVARIANT | SCHED_DEADLINE target ⟹ `-EINVAL`. |
| `cap_required_for_rt_raise` | INVARIANT | RT raise without CAP_SYS_NICE ⟹ `-EPERM`. |
| `copy_from_user_fault_safe` | INVARIANT | `EFAULT` returned on bad `param`. |

### Layer 2: TLA+

`uapi/sched-setparam.tla`:
- Variables: `target.policy`, `target.rt_priority`, `target.normal_prio`, `pi_lock`, `rq_lock`.
- Properties:
  - `safety_policy_preserved` — only `rt_priority`/`normal_prio` mutated.
  - `safety_class_priority_valid` — post: priority matches class invariants.
  - `safety_deadline_immune` — SCHED_DEADLINE never mutated by this syscall.
  - `liveness_eventually_takes_effect` — successful call enqueues to new RT runqueue slot eventually.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_sched_setparam` post: target.policy unchanged on success | `sys_sched_setparam` |
| `__sched_setscheduler` post: rt_priority == param.sched_priority on success | `Sched::set_scheduler_inner` |
| `validate` post: returns Ok ⟹ priority matches policy class rules | `validate` |

### Layer 4: Verus/Creusot functional

Per-`sched_setparam(2)` man page; per-LTP `testcases/kernel/syscalls/sched_setparam/*`; per-glibc `sysdeps/unix/sysv/linux/sched_setparam.c`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

sched_setparam reinforcement:

- **Per-class priority validation strict** — defense against per-out-of-range RT priority producing scheduler walk-tree corruption.
- **Per-policy preservation enforced by re-passing target.policy** — defense against TOCTOU between read of policy and validation.
- **Per-PI-lock-chain-walk after priority change** — defense against per-PI-inversion in lock-chain.
- **Per-SCHED_DEADLINE rejected** — defense against partial deadline-param mutation that violates CBS server admission.

### grsecurity/pax-style reinforcement

- **PaX UDEREF strict on `param` pointer** — every dereference of `param` goes through the validated UDEREF copy helper; defense against per-set-up-by-attacker kernel-space pointer aliased as user.
- **PAX_RANDKSTACK on every entry** — randomise kstack on every `sched_setparam` entry; defense against per-stack-spray RT-priority gadgets.
- **CAP_SYS_NICE strict checked in target's user-ns** — never honour caller's user-ns capability for cross-userns mutation; defense against per-userns CAP-laundering.
- **GRKERNSEC_HIDESYM on `task->dl_se`, `task->rt`** — RT-class structures excluded from `/proc/kallsyms` and `/proc/<pid>/stat` priority leak under `kptr_restrict ≥ 2`; defense against per-RT-priority-side-channel-probe of system RT topology.
- **Per-uid RT-priority quota** — grsec policy enforces per-uid ceiling on number of SCHED_FIFO threads and on max `sched_priority`; defense against per-RT-flood DoS.
- **Audit RT-priority raise** — every successful raise to `sched_priority ≥ 50` audit-logged with `AUDIT_RT_PRIO_RAISE`.
- **Refuse SCHED_FIFO/RR for sandboxed users** — under hardened policy, untrusted users get `-EPERM` regardless of CAP_SYS_NICE.
- **GRKERNSEC_RAND_USERID + CAP_SYS_NICE pinning** — sandbox profiles pin the set of uids permitted to use `sched_setparam` raising RT priority.
- **Deadline-rejection log-rate-limited** — repeated `sched_setparam` calls on SCHED_DEADLINE targets audit-logged as a possible probing pattern.
- **PI-boost cycle detection bounded** — `__sched_setscheduler` walks the lock chain with a fixed-depth bound; defense against attacker-crafted lock-chain producing unbounded recursion.
- **Per-cgroup-cpu-rt-bandwidth quota honoured** — defense against per-cgroup-bypass through `sched_setparam` raising RT priority above the cgroup's allocated bandwidth.
- **GRKERNSEC_PROC_USER restrict `/proc/<pid>/sched`** — denies side-channel observation of post-`sched_setparam` runqueue state.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `sched_setparam` to `-ENOSYS`, forcing applications onto `sched_setattr` which exposes the full sched_attr struct including deadline parameters; defense against per-legacy-API surface.

