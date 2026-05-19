# Tier-5 syscall: sched_getparam(2) — syscall 143

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sched/syscalls.c (SYSCALL_DEFINE2(sched_getparam))
  - kernel/sched/core.c (rt_task helpers; task_rt_priority)
  - include/uapi/linux/sched.h
  - include/uapi/linux/sched/types.h (struct sched_param)
  - arch/x86/entry/syscalls/syscall_64.tbl (143 common sched_getparam)
  - man sched_getparam(2)
-->

## Summary

`sched_getparam(2)` reads back the **scheduling parameters** (the single `sched_priority` field of `struct sched_param`) of a target task. For SCHED_FIFO / SCHED_RR tasks it returns the RT priority in `[1,99]`; for SCHED_NORMAL / SCHED_BATCH / SCHED_IDLE it returns `0`; for SCHED_DEADLINE it returns `0` and the caller is expected to use `sched_getattr(2)` to retrieve the full deadline parameters (runtime / deadline / period). The syscall is the read-side partner of `sched_setparam(2)`; together they form the POSIX.1b-1993 RT-extensions priority-read/write pair, predating the more general `sched_getscheduler(2)` / `sched_getattr(2)`. Critical for: RT-aware applications reading their own (or a managed task's) current RT priority before deciding to boost/demote, debug tooling (`top`, `htop`, `chrt`), priority-inheritance diagnostics, container observability of RT priority ceilings, scheduler-class introspection from constrained sandboxes (where reading is permitted but writing is not).

This Tier-5 covers the userspace ABI of syscall 143. The scheduler-class storage layout is owned by `kernel/sched/core.md` (Tier-3, planned).

## Signature

```c
int sched_getparam(pid_t pid, struct sched_param *param);

struct sched_param {
    int sched_priority;
};
```

Rust ABI shim:

```rust
pub fn sys_sched_getparam(pid: i32, param: *mut SchedParam) -> isize;
```

Syscall number: **143** (x86_64). Generic syscall table: **121**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `pid` | `pid_t` | IN | Target task in caller's pid-ns; `0` = self. |
| `param` | `struct sched_param *` | OUT (UAPI ptr) | Pointer to user buffer; receives `sched_priority` via `copy_to_user`. |

## Return

- **Success**: `0`; `param->sched_priority` written.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno; `param` not written on failure.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `param == NULL`; `pid < 0`. |
| `ESRCH` | No task with given pid in caller's pid-ns. |
| `EFAULT` | `param` points to unmapped userspace memory. |
| `EPERM` | Reserved; current implementation does not require capability to read priority of any task in caller's pid-ns. |

## ABI surface

```text
__NR_sched_getparam (x86_64)  = 143
__NR_sched_getparam (i386)    = 155
__NR_sched_getparam (arm64)   = 121  (generic-syscall)
__NR_sched_getparam (generic) = 121

/* Return mapping */
SCHED_FIFO / SCHED_RR             -> sched_priority ∈ [1,99]
SCHED_NORMAL / SCHED_BATCH / IDLE -> sched_priority == 0
SCHED_DEADLINE                    -> sched_priority == 0 (use sched_getattr for full DL params)
```

## Compatibility contract

REQ-1: Syscall number is **143** on x86_64, **121** on arm64/generic. ABI-stable since 2.0.

REQ-2: `pid == 0` ⟹ target current task.

REQ-3: `pid < 0` ⟹ `-EINVAL`.

REQ-4: `param == NULL` ⟹ `-EFAULT` from `copy_to_user`.

REQ-5: Output is unconditionally non-negative; for non-RT classes, value is `0`.

REQ-6: For SCHED_DEADLINE, this syscall reports `sched_priority = 0`. Callers needing deadline parameters MUST use `sched_getattr(2)`. This is not documented as failure but is a known information-loss.

REQ-7: No capability requirement: any task in caller's pid-ns can be queried.

REQ-8: Read is racy w.r.t. concurrent `sched_setparam` / `sched_setscheduler`: the read returns the state at the moment `task_rq_lock` is taken; subsequent state-change does not affect this return.

REQ-9: Inheritance irrelevant: this is a read-only syscall.

REQ-10: `param` write is atomic from user's perspective per single `copy_to_user` call (a single `int` write).

## Acceptance Criteria

- [ ] AC-1: `sched_getparam(0, &p)` for SCHED_NORMAL self: returns `0`; `p.sched_priority == 0`.
- [ ] AC-2: `sched_getparam(0, &p)` after `sched_setscheduler(0, SCHED_FIFO, &{42})`: returns `0`; `p.sched_priority == 42`.
- [ ] AC-3: `sched_getparam(target_pid_other_uid, &p)` from unprivileged caller: returns `0`; `p.sched_priority` reflects target.
- [ ] AC-4: `sched_getparam(-1, &p)`: returns `-EINVAL`.
- [ ] AC-5: `sched_getparam(non_existent_pid, &p)`: returns `-ESRCH`.
- [ ] AC-6: `sched_getparam(0, NULL)`: returns `-EFAULT`.
- [ ] AC-7: `sched_getparam` on SCHED_DEADLINE task: returns `0`; `p.sched_priority == 0` (deadline params not exposed).
- [ ] AC-8: Concurrent `sched_setparam` + `sched_getparam`: read sees either pre- or post-state, never a torn value.
- [ ] AC-9: `param` in unmapped page: returns `-EFAULT`; `param` not modified.
- [ ] AC-10: `sched_getparam` from inside cross-pid-ns: target in different ns ⟹ `-ESRCH`.

## Architecture

Rookery surface in `kernel/sched/syscalls.rs`:

```rust
pub fn sys_sched_getparam(pid: i32, param: UserPtrMut<SchedParam>) -> isize {
    if pid < 0 {
        return -EINVAL as isize;
    }
    let target = match Task::find_in_pidns(current().pid_ns(), pid) {
        Some(t) => t,
        None    => return -ESRCH as isize,
    };
    let prio = {
        let _g = target.pi_lock_irqsave();
        if Sched::is_dl_policy(target.policy()) {
            0
        } else {
            target.rt_priority() as i32
        }
    };
    let out = SchedParam { sched_priority: prio };
    match param.copy_to_user(&out) {
        Ok(_)  => 0,
        Err(_) => -EFAULT as isize,
    }
}
```

`task.rt_priority()` accessor:

```rust
fn rt_priority(task: &Task) -> u32 {
    match task.policy() {
        SCHED_FIFO | SCHED_RR => task.sched_rt.rt_priority,
        _                     => 0,
    }
}
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `non_negative_output` | INVARIANT | `param.sched_priority >= 0` on success. |
| `non_rt_returns_zero` | INVARIANT | policy ∉ {FIFO, RR} ⟹ output `0`. |
| `deadline_returns_zero` | INVARIANT | SCHED_DEADLINE ⟹ output `0` (no DL leak). |
| `copy_to_user_fault_safe` | INVARIANT | bad `param` ⟹ `-EFAULT`. |
| `no_torn_write` | INVARIANT | single `copy_to_user(int)` ⟹ atomic from user. |

### Layer 2: TLA+

`uapi/sched-getparam.tla`:
- Variables: `target.policy`, `target.rt_priority`, `param_out`.
- Properties:
  - `safety_value_matches_policy_class` — post: output consistent with class rules.
  - `safety_read_only` — `target` state unchanged.
  - `safety_no_dl_leak` — SCHED_DEADLINE ⟹ output 0.
  - `liveness_terminates` — call terminates without blocking.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_sched_getparam` post: `sched_priority` matches task.rt_priority for RT, else 0 | `sys_sched_getparam` |
| `sys_sched_getparam` post: target state unchanged | `sys_sched_getparam` |
| `task.rt_priority()` post: returns 0 if non-RT | `Task::rt_priority` |

### Layer 4: Verus/Creusot functional

Per-`sched_getparam(2)` man page; per-LTP `testcases/kernel/syscalls/sched_getparam/*`; per-glibc `sysdeps/unix/sysv/linux/sched_getparam.c`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

sched_getparam reinforcement:

- **Per-policy gated output** — SCHED_DEADLINE never leaks deadline params; defense against per-deadline-parameter info leak via a syscall not designed for it.
- **Per-pid-ns scoping strict** — defense against per-cross-pidns RT-priority probe.
- **Per-pi_lock taken during read** — defense against per-torn-read race vs `sched_setparam`.
- **Per-non-negative output guaranteed** — defense against per-sign-extension-confusion in callers.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF strict on `param` write** — every `copy_to_user(param)` goes through the validated UDEREF helper; defense against per-set-up-by-attacker kernel-space pointer aliased as user.
- **PAX_RANDKSTACK on every entry** — randomise kstack on each `sched_getparam` entry; defense against per-stack-spray followed by a controllable copy_to_user.
- **GRKERNSEC_HIDESYM on `task->rt`** — RT-class structures excluded from `/proc/kallsyms` and from indirect leaks via priority readback shape under `kptr_restrict ≥ 2`.
- **GRKERNSEC_PROC_USER restrict `sched_getparam` cross-user** — under hardened policy, an unprivileged user can read only own tasks' priority (rather than any task in pid-ns); defense against per-RT-priority-side-channel mapping of system RT topology by an attacker reconnaissance pass.
- **Audit per-RT-priority read above threshold** — `sched_getparam` returning ≥ 50 audit-logged at LOG_INFO; defense-in-depth observability for RT-task discovery.
- **Refuse SCHED_DEADLINE deadline-readback workaround** — repeated `sched_getparam` calls combined with timing-side-channel that infer deadline params are audit-logged.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `sched_getparam` to `-ENOSYS`, forcing applications onto `sched_getattr` which exposes a versioned struct with cleaner ABI; defense against per-legacy-API surface.
- **Per-uid sched_getparam rate-limit** — defense against per-RT-topology-scan brute-force across the pid space.
- **Pid existence enumeration mitigation** — `ESRCH` vs successful read is timing-side-channel-mitigated by constant-time path before pid lookup.
- **GRKERNSEC_PROC_GID gate `/proc/<pid>/sched` readback** — pair with `sched_getparam` to ensure a single coherent policy on RT-priority observability.
- **Sandbox profile pinning** — under SECCOMP-LSM combination, `sched_getparam` may be allowed but `sched_setparam` denied, providing read-only RT observability without privilege escalation surface.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `kernel/sched/core.md` Tier-3: rq-lock acquisition, task lookup.
- `sched_setparam.md` sibling.
- `sched_getattr.md` sibling (modern, full-struct).
- `sched_getscheduler.md` sibling (policy alone).
- glibc / musl userspace wrappers.
- Implementation code.
