---
title: "Tier-5 syscall: sched_getscheduler(2) — syscall 145"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sched_getscheduler(2)` returns the **current scheduling policy** of a target task (PID 0 = self) as the integer constant from `<linux/sched.h>`: SCHED_NORMAL (0), SCHED_FIFO (1), SCHED_RR (2), SCHED_BATCH (3), SCHED_IDLE (5), SCHED_DEADLINE (6). The SCHED_RESET_ON_FORK bit (0x40000000) is OR'd into the result if the task was set with that flag. The call is read-only and very cheap: a single read of `task->policy` under RCU. Critical for: real-time auditing, observability tools (`chrt -p`, `ps -el`), libc helpers that adapt behavior based on caller's scheduling class, container introspection, and Rookery's policy auditor that snapshots all task policies during a tracing run.

### Acceptance Criteria

- [ ] AC-1: After `sched_setscheduler(0, SCHED_FIFO, &p)`, `sched_getscheduler(0)` returns SCHED_FIFO.
- [ ] AC-2: After `sched_setscheduler(0, SCHED_RR | SCHED_RESET_ON_FORK, &p)`, `sched_getscheduler(0) == (SCHED_RR | 0x40000000)`.
- [ ] AC-3: `sched_getscheduler(-1)` returns -EINVAL.
- [ ] AC-4: `sched_getscheduler(99999)` for unknown PID returns -ESRCH.
- [ ] AC-5: A newborn process under SCHED_NORMAL returns 0.
- [ ] AC-6: SCHED_DEADLINE tasks (set via sched_setattr) return 6.
- [ ] AC-7: Cross-pid-ns queries return -ESRCH.
- [ ] AC-8: LSM-denied query returns -EPERM (or -EACCES depending on LSM mapping).
- [ ] AC-9: 1e6 concurrent `sched_getscheduler(pid)` calls return consistent values (atomic read).
- [ ] AC-10: `sched_getscheduler(pid)` does not alter any per-task state (verified via second call).
- [ ] AC-11: PID 0 is interpreted as self (not literal kernel pid 0).
- [ ] AC-12: Strace shows correct mnemonic decoding for return value.

### Architecture

```rust
#[syscall(nr = 145, abi = "sysv")]
pub fn sys_sched_getscheduler(pid: pid_t) -> SysResult<i32> {
    SchedGetscheduler::do_call(pid)
}
```

`SchedGetscheduler::do_call(pid) -> i32`:
1. if pid < 0 { return -EINVAL; }
2. rcu_read_lock();
3. let task = if pid == 0 { current() } else {
       match find_task_by_vpid(pid) {
           Some(t) => t,
           None => { rcu_read_unlock(); return -ESRCH; }
       }
   };
4. /* LSM */
5. if let Err(e) = security_task_getscheduler(task) {
       rcu_read_unlock();
       return e;             // -EACCES / -EPERM
   }
6. let mut policy = task.policy as i32;
7. if task.sched_reset_on_fork { policy |= SCHED_RESET_ON_FORK; }
8. rcu_read_unlock();
9. policy

### Out of Scope

- `sched_setscheduler(2)` (Tier-5 separate doc).
- `sched_getattr(2)` (Tier-5 separate doc — returns deadline parameters).
- `sched_getparam(2)` (Tier-5 separate doc — returns sched_priority only).
- Per-class scheduler internals (Tier-3 in `kernel/sched-core.md`).
- Implementation code.

### signature

```c
int sched_getscheduler(pid_t pid);
```

### parameters

| Parameter | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | Target task PID (caller's active pid-ns); 0 = self. |

### return value

| Value | Meaning |
|---|---|
| ≥ 0 | Policy integer; possibly OR'd with SCHED_RESET_ON_FORK (0x40000000). |
| `-EINVAL` | pid < 0. |
| `-ESRCH` | No task with the given pid in caller's pid-ns. |
| `-EPERM` | LSM denial (rare; SELinux task_getscheduler permission). |

### errors

| Errno | Trigger |
|---|---|
| `EINVAL` | pid is negative. |
| `ESRCH` | Target PID not found, or in a different pid-ns from caller. |
| `EPERM` | security_task_getscheduler denies (e.g., SELinux). |

### abi surface

```text
__NR_sched_getscheduler (x86_64) = 145
__NR_sched_getscheduler (i386)   = 157
__NR_sched_getscheduler (arm64)  = 120  (generic-syscall)
__NR_sched_getscheduler (generic) = 120

/* Policy values returned */
SCHED_NORMAL      = 0    /* SCHED_OTHER */
SCHED_FIFO        = 1
SCHED_RR          = 2
SCHED_BATCH       = 3
SCHED_IDLE        = 5
SCHED_DEADLINE    = 6
SCHED_RESET_ON_FORK = 0x40000000   /* OR'd into return if set */
```

### compatibility contract

REQ-1: Syscall number is **145** on x86_64; **120** on arm64/generic. ABI-stable since 2.0.

REQ-2: pid == 0 ⟹ target current task.

REQ-3: pid < 0 ⟹ -EINVAL (a sanity check on userspace; man-page mandated).

REQ-4: pid > 0 with no matching task in caller's pid-ns ⟹ -ESRCH.

REQ-5: Returned value is `task->policy` bit-OR'd with `SCHED_RESET_ON_FORK` (0x40000000) if `task->sched_reset_on_fork` is set.

REQ-6: SCHED_DEADLINE tasks return 6; deadline parameters are NOT returned by this call (use sched_getattr).

REQ-7: LSM hook `security_task_getscheduler(task)` is invoked before reading policy; denial ⟹ -EPERM (SELinux maps to -EACCES which the manpage reports as -EPERM at UAPI surface).

REQ-8: The call holds RCU read lock to look up the task; it does NOT take the rq lock. The read is single-word atomic on task->policy.

REQ-9: No CAP_SYS_NICE requirement: any task may query any visible task's policy.

REQ-10: No state mutation; idempotent; async-signal-safe.

REQ-11: Bit-field interpretation: `policy & ~SCHED_RESET_ON_FORK` gives the base class; the high bit is the modifier. Userspace MUST mask before comparing with SCHED_* class constants.

REQ-12: Newly-created tasks return SCHED_NORMAL (0) until promoted. Default nice is 0; not reflected here (use getpriority).

REQ-13: The call is namespace-aware: a task in a different pid-ns is invisible (-ESRCH).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pid_negative_rejected` | INVARIANT | per-call: pid<0 ⟹ -EINVAL. |
| `rcu_read_lock_balanced` | INVARIANT | per-call: every rcu_read_lock paired with rcu_read_unlock on all paths. |
| `no_state_mutation` | INVARIANT | per-call: no task field written. |
| `reset_on_fork_bit_or` | INVARIANT | per-call: bit set ⟺ task.sched_reset_on_fork. |
| `pidns_visibility` | INVARIANT | per-call: task in other ns ⟹ -ESRCH. |
| `lsm_hook_invoked` | INVARIANT | per-call: security_task_getscheduler called before policy read returns. |

### Layer 2: TLA+

`kernel/sched_getscheduler.tla`:
- States: per-task policy + reset-on-fork flag.
- Properties:
  - `safety_returns_policy_or_error` — per-call: returns valid policy bits or known errno.
  - `safety_no_mutation` — per-call: no state change.
  - `safety_pidns_isolation` — per-call: cross-ns ⟹ -ESRCH.
  - `liveness_terminates` — per-call: O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: ret == task.policy (\| SCHED_RESET_ON_FORK if set) ∨ errno | `SchedGetscheduler::do_call` |
| `find_task_by_vpid` post: respects caller's active pid-ns | `Pid::find_task_by_vpid` |
| `security_task_getscheduler` post: hook called pre-return | `Sched::lsm_hook` |

### Layer 4: Verus / Creusot functional

Per-`sched_getscheduler(2)` man-page equivalence. LTP `sched_getscheduler01..02` pass. POSIX.1-2008 (XSI option) semantics verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sched_getscheduler(2)` reinforcement:

- **Per-RCU-balanced lookup** — defense against per-leak (rcu_read_lock without unlock on -ESRCH).
- **Per-LSM hook pre-policy-read** — defense against per-info-leak when LSM denies introspection.
- **Per-pidns isolation** — defense against per-cross-container PID enumeration.
- **Per-atomic policy read** — defense against per-half-set policy bits during concurrent setscheduler.

### grsecurity / pax-style reinforcement

- **PaX UDEREF (n/a for this call)** — sched_getscheduler takes no userspace pointers, so UDEREF has no per-arg surface; included for completeness — the return path through the syscall ABI is the only ucode interaction.
- **No CAP_SYS_NICE for read** — POSIX-compliant; grsec does not require CAP_SYS_NICE for introspection but does respect GRKERNSEC_PROC_USERGROUP: querying another user's task policy is denied with -ESRCH (not -EPERM, to avoid PID-existence leakage) when the target is not owned by caller and caller lacks CAP_SYS_PTRACE.
- **GRKERNSEC_RESLOG on policy enumeration** — repeated queries against tasks the caller does not own can be rate-logged (per-task warn) under GRKERNSEC_AUDIT_BRUTE-style policy to detect policy-fingerprinting tools.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset to defeat per-stack fingerprinting via repeated calls.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec chroot, find_task_by_vpid is restricted to tasks in the chroot's task set; queries outside return -ESRCH.
- **No_new_privs neutral** — read-only call; NNP has no effect.
- **Sandbox SCHED_DEADLINE attack-surface reduction** — while this syscall does not modify policy, querying a SCHED_DEADLINE task from inside a sandboxed task is permitted (read-only), but the query is audited so reconnaissance on RT/deadline tasks is observable.
- **KEEPCAPS neutral** — no capability check.
- **No CAP requirement** — strictly POSIX-aligned: any task may query any visible task.

