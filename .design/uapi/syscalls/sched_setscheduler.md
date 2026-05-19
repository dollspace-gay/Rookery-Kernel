# Tier-5 syscall: sched_setscheduler(2) — syscall 144

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sched/core.c (SYSCALL_DEFINE3(sched_setscheduler))
  - kernel/sched/syscalls.c (__sched_setscheduler, _sched_setscheduler)
  - kernel/sched/rt.c (RT class hooks)
  - kernel/sched/fair.c (CFS hooks for SCHED_NORMAL/BATCH/IDLE)
  - kernel/sched/deadline.c (SCHED_DEADLINE policy)
  - include/uapi/linux/sched.h (SCHED_* constants)
  - include/uapi/linux/sched/types.h (struct sched_param)
  - arch/x86/entry/syscalls/syscall_64.tbl (144 common sched_setscheduler)
-->

## Summary

`sched_setscheduler(2)` changes the **scheduling policy** and **priority** of a target task (PID 0 = self) to one of the kernel's per-class policies: SCHED_NORMAL/SCHED_OTHER (CFS-default), SCHED_FIFO (real-time first-in-first-out), SCHED_RR (real-time round-robin), SCHED_BATCH (CPU-bound non-interactive), SCHED_IDLE (very-low-priority background), or SCHED_DEADLINE (EDF / CBS — set via sched_setattr, NOT via this syscall). The user-supplied `struct sched_param` carries a single field, `sched_priority`, valid 1..99 for SCHED_FIFO/RR and required to be 0 for the other policies. Critical for: real-time audio (JACK, PipeWire), industrial control loops, IRQ-thread tuning, latency-sensitive servers, container CPU-class enforcement, sandbox policy demotion (drop privileges by forcing SCHED_IDLE / SCHED_BATCH on untrusted workers).

## Signature

```c
int sched_setscheduler(pid_t pid, int policy,
                       const struct sched_param *param);

struct sched_param {
    int sched_priority;
};
```

## Parameters

| Parameter | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | Target task PID (caller's active pid-ns); 0 = self. |
| `policy` | `int` | in | SCHED_NORMAL (0), SCHED_FIFO (1), SCHED_RR (2), SCHED_BATCH (3), SCHED_IDLE (5), SCHED_DEADLINE (6, rejected here), or any of these OR'd with SCHED_RESET_ON_FORK (0x40000000). |
| `param` | `const struct sched_param *` | in (UAPI) | Pointer to sched_priority; copied in via copy_from_user. |

## Return value

| Value | Meaning |
|---|---|
| 0 | Success; task now runs under the requested policy at the requested priority. |
| `-EINVAL` | Invalid policy, priority out-of-range, NULL param, or SCHED_DEADLINE requested (must use sched_setattr). |
| `-EPERM` | Caller lacks CAP_SYS_NICE for a real-time policy or for a non-owned target. |
| `-ESRCH` | No task with the given pid in caller's pid-ns. |
| `-EFAULT` | param points to inaccessible userspace memory. |
| `-EBUSY` | SCHED_DEADLINE bandwidth oversubscription (only via sched_setattr path; included for completeness). |

## Errors

| Errno | Trigger |
|---|---|
| `EINVAL` | Unknown policy bit; sched_priority out of [0,99]; SCHED_DEADLINE via this syscall; SCHED_FIFO/RR with priority 0. |
| `EPERM` | Non-CAP_SYS_NICE caller asking for SCHED_FIFO/RR/DEADLINE, raising RT priority of a task it doesn't own, or modifying another user's task. |
| `ESRCH` | Target PID not found, or in a different pid-ns from caller. |
| `EFAULT` | param NULL or unmapped. |
| `EBUSY` | Bandwidth-allocator rejects (deadline only; on this syscall it's reported as EINVAL because deadline is refused). |

## ABI surface

```text
__NR_sched_setscheduler (x86_64) = 144
__NR_sched_setscheduler (i386)   = 156
__NR_sched_setscheduler (arm64)  = 119  (generic-syscall)
__NR_sched_setscheduler (generic) = 119

/* Policies */
SCHED_NORMAL      = 0    /* a.k.a. SCHED_OTHER (CFS default) */
SCHED_FIFO        = 1    /* RT: run until block / yield */
SCHED_RR          = 2    /* RT: time-sliced FIFO */
SCHED_BATCH       = 3    /* CPU-bound, non-interactive */
SCHED_IDLE        = 5    /* very low priority */
SCHED_DEADLINE    = 6    /* EDF / CBS; rejected by this syscall */
SCHED_RESET_ON_FORK = 0x40000000  /* policy modifier */

/* Priority bounds (sched_get_priority_min/max) */
SCHED_FIFO/RR    : 1..99
SCHED_NORMAL/BATCH/IDLE/DEADLINE : 0 (must be 0)

struct sched_param {
    int sched_priority;
};
```

## Compatibility contract

REQ-1: Syscall number is **144** on x86_64; **119** on arm64/generic. ABI-stable since 2.0.

REQ-2: pid == 0 ⟹ target current task.

REQ-3: SCHED_DEADLINE via this syscall **must** return -EINVAL; deadline parameters require sched_setattr (which carries runtime/deadline/period).

REQ-4: Policy bits are validated as (policy & ~SCHED_RESET_ON_FORK) ∈ {NORMAL, FIFO, RR, BATCH, IDLE}.

REQ-5: SCHED_FIFO and SCHED_RR require sched_priority ∈ [1,99]. SCHED_NORMAL/BATCH/IDLE require sched_priority == 0. Any mismatch ⟹ -EINVAL.

REQ-6: Raising a task to SCHED_FIFO/RR requires CAP_SYS_NICE in the caller's user namespace; without CAP_SYS_NICE, the caller may only set SCHED_FIFO/RR if `rlimit(RLIMIT_RTPRIO)` permits the chosen priority AND the target task is owned by the caller.

REQ-7: Setting policy on a task owned by another user requires CAP_SYS_NICE.

REQ-8: The change is effective on return: the task's `sched_class`, `policy`, and (for RT classes) `rt_priority` fields are updated atomically under `task_rq_lock`. If the target was running, it is dequeued, requeued under the new class, and possibly preempted.

REQ-9: SCHED_RESET_ON_FORK modifier: children created via fork/clone reset policy to SCHED_NORMAL and any negative nice value to 0. Cleared on the target itself after first reset propagation if requested via SCHED_FLAG_RESET_ON_FORK semantics — but for sched_setscheduler it persists as a policy bit.

REQ-10: Lowering a task from a real-time policy to SCHED_NORMAL requires CAP_SYS_NICE if the target is not owned by caller; owners can demote themselves freely.

REQ-11: SCHED_DEADLINE tasks cannot be retargeted via this syscall to a non-deadline policy except by passing SCHED_DEADLINE as policy (which is rejected) OR by sched_setattr. To leave deadline, userspace MUST call sched_setattr with the new policy.

REQ-12: The syscall is equivalent to `sched_setattr(pid, &attr, 0)` where `attr.sched_policy = policy & ~SCHED_RESET_ON_FORK`, `attr.sched_priority = param->sched_priority`, `attr.sched_flags = (policy & SCHED_RESET_ON_FORK) ? SCHED_FLAG_RESET_ON_FORK : 0`, and all deadline fields are zero.

REQ-13: LSM hook `security_task_setscheduler(task)` is invoked; SELinux / AppArmor / Tomoyo may deny. seccomp may filter the syscall entry.

REQ-14: `copy_from_user(&lparam, param, sizeof(*param))` validates the param pointer; failure ⟹ -EFAULT. The buffer is treated as untrusted.

REQ-15: Result observable via `sched_getscheduler(pid)` (returns policy bits, including SCHED_RESET_ON_FORK) and `sched_getparam(pid)` (priority field).

## Acceptance Criteria

- [ ] AC-1: `sched_setscheduler(0, SCHED_FIFO, {.sched_priority=50})` as CAP_SYS_NICE → 0; subsequent `sched_getscheduler(0)` returns SCHED_FIFO.
- [ ] AC-2: Same call without CAP_SYS_NICE and without RLIMIT_RTPRIO → -EPERM.
- [ ] AC-3: `sched_setscheduler(0, SCHED_NORMAL, {.sched_priority=10})` → -EINVAL (priority must be 0).
- [ ] AC-4: `sched_setscheduler(0, SCHED_DEADLINE, &p)` → -EINVAL regardless of priority.
- [ ] AC-5: `sched_setscheduler(99999, SCHED_NORMAL, &p)` for unknown PID → -ESRCH.
- [ ] AC-6: `sched_setscheduler(pid, SCHED_FIFO, NULL)` → -EFAULT.
- [ ] AC-7: `sched_setscheduler(0, SCHED_RR | SCHED_RESET_ON_FORK, {.sched_priority=20})` succeeds; child of subsequent fork inherits SCHED_NORMAL.
- [ ] AC-8: An RT task at SCHED_FIFO/50 set to SCHED_NORMAL by its owner (no CAP_SYS_NICE needed for self-demotion) → 0.
- [ ] AC-9: Cross-user target without CAP_SYS_NICE → -EPERM.
- [ ] AC-10: After setting SCHED_FIFO, the task preempts SCHED_NORMAL tasks immediately on the same CPU.
- [ ] AC-11: LSM denial path returns -EACCES (mapped from security_task_setscheduler).
- [ ] AC-12: SCHED_DEADLINE-resident task subject to sched_setscheduler(_, SCHED_NORMAL, _) → -EINVAL (must use sched_setattr).

## Architecture

```rust
#[syscall(nr = 144, abi = "sysv")]
pub fn sys_sched_setscheduler(pid: pid_t, policy: i32,
                              param: UserPtr<SchedParam>) -> SysResult<i32> {
    SchedSetscheduler::do_call(pid, policy, param)
}
```

`SchedSetscheduler::do_call(pid, policy, param) -> i32`:
1. if policy & ~SCHED_RESET_ON_FORK == SCHED_DEADLINE { return -EINVAL; }
2. if param.is_null() { return -EFAULT; }
3. let lparam = copy_from_user::<SchedParam>(param)?;     // -EFAULT on bad addr
4. let attr = SchedAttr {
       size: size_of::<SchedAttr>() as u32,
       sched_policy: (policy & !SCHED_RESET_ON_FORK) as u32,
       sched_flags:  if (policy & SCHED_RESET_ON_FORK) != 0 { SCHED_FLAG_RESET_ON_FORK } else { 0 },
       sched_nice:   0,
       sched_priority: lparam.sched_priority as u32,
       sched_runtime: 0, sched_deadline: 0, sched_period: 0,
       ..Default::default()
   };
5. Sched::__sched_setscheduler(pid, &attr, /*user=*/true, /*pi=*/true)

`Sched::__sched_setscheduler(pid, attr, user, pi) -> i32`:
1. /* policy validation */
2. validate_policy(attr.sched_policy)?;             // -EINVAL on unknown
3. validate_priority(attr.sched_policy, attr.sched_priority)?;
4. /* resolve task */
5. let task = if pid == 0 { current() } else { find_task_by_vpid(pid).ok_or(-ESRCH)? };
6. /* permission gating */
7. Sched::check_setscheduler_perm(task, attr, user)?; // -EPERM
8. /* LSM */
9. security_task_setscheduler(task)?;               // -EACCES
10. /* atomic switch under rq lock */
11. let rq = task_rq_lock(task);
12. Sched::__setscheduler(rq, task, attr, pi);
13. task_rq_unlock(rq);
14. 0

`Sched::check_setscheduler_perm(task, attr, user)`:
1. /* Self-demotion to NORMAL/BATCH/IDLE always allowed */
2. if attr.sched_policy ∈ {NORMAL,BATCH,IDLE} && task == current() && current().euid == task.euid { return Ok; }
3. /* RT class needs CAP_SYS_NICE or RLIMIT_RTPRIO + ownership */
4. if attr.sched_policy ∈ {FIFO,RR} {
       if !capable(CAP_SYS_NICE) {
           if task.real_cred.uid != current().euid { return Err(-EPERM); }
           if attr.sched_priority > task_rlimit(task, RLIMIT_RTPRIO) { return Err(-EPERM); }
       }
   }
5. /* Cross-user gating */
6. if task.real_cred.euid != current().euid && !capable(CAP_SYS_NICE) { return Err(-EPERM); }
7. Ok

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `param_copied_via_uaccess` | INVARIANT | per-call: param is read via copy_from_user; no direct deref. |
| `deadline_rejected` | INVARIANT | per-call: SCHED_DEADLINE ⟹ -EINVAL. |
| `priority_bounds` | INVARIANT | per-call: priority validated for class before any state change. |
| `rq_lock_held_during_switch` | INVARIANT | per-__setscheduler: task_rq_lock held over dequeue→class-change→enqueue. |
| `cap_sys_nice_for_rt` | INVARIANT | per-FIFO/RR: capable(CAP_SYS_NICE) OR (own task ∧ priority ≤ RLIMIT_RTPRIO). |
| `cross_user_capability` | INVARIANT | per-foreign-uid: capable(CAP_SYS_NICE) required. |
| `lsm_hook_invoked` | INVARIANT | per-call: security_task_setscheduler called before commit. |

### Layer 2: TLA+

`kernel/sched_setscheduler.tla`:
- States: per-task policy, priority, class, run-queue residency.
- Transitions: per-validated-call → atomic dequeue→reclass→enqueue.
- Properties:
  - `safety_policy_consistent` — per-task: (policy, priority) always matches class rules.
  - `safety_no_partial_state` — per-call: failure leaves task unchanged.
  - `safety_perm_enforced` — per-call: -EPERM cases never commit.
  - `liveness_terminates` — per-call: returns in O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: SCHED_DEADLINE input ⟹ ret == -EINVAL | `SchedSetscheduler::do_call` |
| `validate_priority` post: matches policy class | `SchedSetscheduler::validate_priority` |
| `check_setscheduler_perm` post: returns Ok ⟹ caps satisfied | `SchedSetscheduler::check_setscheduler_perm` |
| `__setscheduler` post: task.policy/sched_class/rt_priority updated under rq lock | `Sched::__setscheduler` |

### Layer 4: Verus / Creusot functional

Per-`sched_setscheduler(2)` man-page equivalence. LTP `sched_setscheduler01..04` pass. POSIX.1-2008 (XSI option) semantics verified for FIFO/RR/NORMAL.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sched_setscheduler(2)` reinforcement:

- **Per-policy-then-priority validation order** — defense against per-EINVAL-after-side-effect bugs.
- **Per-rq-lock atomic class switch** — defense against per-half-reclassed task observed by another CPU.
- **Per-LSM hook pre-commit** — defense against per-policy bypass.
- **Per-RLIMIT_RTPRIO fallback** — defense against per-uncapped real-time elevation.
- **Per-cross-user capability** — defense against per-task-hijacking by neighbor uid.
- **Per-SCHED_DEADLINE rejection** — defense against per-deadline-via-priority-API misuse.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `param`** — the copy_from_user call validates that `param` resides in userspace; UDEREF prevents per-kernel-pointer smuggling that would let an attacker pass a kernel address.
- **CAP_SYS_NICE strict for SCHED_FIFO/RR/DEADLINE** — grsec elevates RLIMIT_RTPRIO bypass to a stricter check: under GRKERNSEC_HARDEN, RLIMIT_RTPRIO alone is insufficient — CAP_SYS_NICE is required even for self-promotion to RT.
- **GRKERNSEC_RESLOG on RLIMIT_RTPRIO violations** — every -EPERM from RTPRIO ceiling is logged at KERN_WARNING with task name, uid, target pid, attempted priority.
- **GRKERNSEC_SCHED_HARDEN** (Rookery extension) — under sandboxed task flags (no_new_privs ∨ seccomp-filter active), SCHED_FIFO/RR self-promotion is denied even with CAP_SYS_NICE; only demotion to NORMAL/BATCH/IDLE is permitted.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset to defeat per-stack fingerprinting via repeated calls.
- **No_new_privs gate** — when current has PR_SET_NO_NEW_PRIVS, any policy that would elevate scheduling priority (SCHED_FIFO/RR or nice < target's current) is denied with -EPERM.
- **Sandbox SCHED_DEADLINE attack-surface reduction** — under sandbox flags, the deadline-rejection path is supplemented with a GRKERNSEC audit log of any deadline-policy attempt, since deadline bandwidth is a known RT-DoS vector.
- **Per-target-pid namespace confinement** — find_task_by_vpid restricts to caller's active pid-ns; grsec adds an ESRCH-not-EPERM rule so cross-ns attempts cannot leak existence of out-of-ns PIDs.
- **KEEPCAPS aware** — capability check uses current's effective set; SECBIT_KEEP_CAPS does not bypass the CAP_SYS_NICE gate.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `sched_setattr(2)` (Tier-5 separate doc — covers SCHED_DEADLINE).
- `sched_getscheduler(2)` (Tier-5 separate doc).
- `setpriority(2)` / `getpriority(2)` (Tier-5 separate docs — nice-value API).
- CFS / RT / DEADLINE class internals (Tier-3 in `kernel/sched-core.md`).
- RT bandwidth controller (Tier-3 in `kernel/sched-rt-bandwidth.md`).
- Implementation code.
