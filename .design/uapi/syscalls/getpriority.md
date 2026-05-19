# Tier-5 syscall: getpriority(2) — syscall 140

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE2(getpriority))
  - kernel/sched/core.c (task_nice helper)
  - include/uapi/linux/resource.h (PRIO_PROCESS, PRIO_PGRP, PRIO_USER)
  - arch/x86/entry/syscalls/syscall_64.tbl (140 common getpriority)
-->

## Summary

`getpriority(2)` returns the **nice value** of one of: a single process (PRIO_PROCESS), a process group (PRIO_PGRP), or a user (PRIO_USER) — for the latter two, the **minimum** nice value (numerically lowest, meaning highest scheduling priority) across all matching tasks is returned. The kernel's nice values internally range from -20 (highest priority) to +19 (lowest); the syscall returns them **offset by +20** to make the range 0..39 to avoid -1 colliding with the error return — so the user MUST first clear errno, call getpriority, and check errno on a -1 return. Critical for: `nice` / `renice` userspace tools, `top` and `ps` priority columns, per-user load detection, container resource auditing, Rookery's sandbox auditor.

## Signature

```c
int getpriority(int which, id_t who);
```

## Parameters

| Parameter | Type | Direction | Description |
|---|---|---|---|
| `which` | `int` | in | PRIO_PROCESS (0), PRIO_PGRP (1), PRIO_USER (2). |
| `who` | `id_t` | in | Target ID: PID for PROCESS (0 = self), PGID for PGRP (0 = self group), UID for USER (0 = self uid). |

## Return value

| Value | Meaning |
|---|---|
| -20..19 | The minimum nice value among matching tasks. Userspace MUST clear errno before the call, since -1 is a valid nice value (interpreted as "high priority"). |
| `-1` + errno set | Error; check errno. |

The errno-conflict is a UAPI quirk: the underlying kernel ABI returns `(20 - nice)`, so the wire value is 0..40 (never -1); glibc subtracts 20 to present the conventional range. The kernel itself returns 0..40 on success and a negative errno on error; glibc translates.

## Errors

| Errno | Trigger |
|---|---|
| `EINVAL` | which not in {PRIO_PROCESS, PRIO_PGRP, PRIO_USER}. |
| `ESRCH` | No matching process/group/user found. |

## ABI surface

```text
__NR_getpriority (x86_64) = 140
__NR_getpriority (i386)   = 96
__NR_getpriority (arm64)  = 141  (generic-syscall)
__NR_getpriority (generic) = 141

/* which values */
PRIO_PROCESS = 0
PRIO_PGRP    = 1
PRIO_USER    = 2

/* Nice value range */
NICE_MIN     = -20
NICE_MAX     =  19

/* Kernel wire ABI: returns (20 - nice), range 0..40; -errno on failure. */
/* Glibc translates: subtracts 20 → -20..19. */
```

## Compatibility contract

REQ-1: Syscall number is **140** on x86_64; **141** on arm64/generic.

REQ-2: which MUST be one of PRIO_PROCESS, PRIO_PGRP, PRIO_USER; otherwise -EINVAL.

REQ-3: PRIO_PROCESS:
- who == 0 ⟹ target current process.
- who > 0 ⟹ target task by PID in caller's pid-ns.
- PID not found ⟹ -ESRCH.
- Returns the task's nice value (`task_nice(task)`).

REQ-4: PRIO_PGRP:
- who == 0 ⟹ target current process's PGID.
- who > 0 ⟹ target by PGID in caller's pid-ns.
- Iterates all tasks with task_pgrp == target; returns the minimum (most-prio) nice.
- No matching tasks ⟹ -ESRCH.

REQ-5: PRIO_USER:
- who == 0 ⟹ target current's real UID.
- who > 0 ⟹ target by UID in caller's user-ns.
- Iterates all tasks with task_uid == target; returns the minimum nice.
- No matching tasks ⟹ -ESRCH.

REQ-6: No capability required to read.

REQ-7: LSM hook `security_task_getpriority(task)` fires for each task examined (PRIO_PGRP and PRIO_USER iterate; PRIO_PROCESS hooks once).

REQ-8: For SCHED_FIFO/RR/DEADLINE tasks, the returned "nice" is meaningless (RT priority is stored separately). The kernel returns the task's nominal nice (usually 0); userspace should use sched_getparam/sched_getattr for RT info.

REQ-9: The kernel return value is 0..40 on success (never -1 on the wire); glibc converts.

REQ-10: rcu_read_lock held while iterating tasks; no rq lock needed (nice is read with READ_ONCE).

REQ-11: User and group iteration is namespace-aware: only tasks visible in the caller's pid-ns are considered.

REQ-12: PRIO_USER iterates via the user_struct hash table; PRIO_PGRP via the PIDTYPE_PGID hlist.

## Acceptance Criteria

- [ ] AC-1: `getpriority(PRIO_PROCESS, 0)` returns current's nice.
- [ ] AC-2: After `setpriority(PRIO_PROCESS, 0, 5)`, `getpriority` returns 5.
- [ ] AC-3: `getpriority(PRIO_PROCESS, 99999)` for unknown PID returns -1 with errno=ESRCH.
- [ ] AC-4: `getpriority(3, 0)` (invalid which) returns -1 with errno=EINVAL.
- [ ] AC-5: `getpriority(PRIO_PGRP, my_pgid)` returns minimum nice across the process group.
- [ ] AC-6: `getpriority(PRIO_USER, my_uid)` returns minimum nice across all my processes.
- [ ] AC-7: For an RT task (SCHED_FIFO), `getpriority(PRIO_PROCESS, rt_pid)` returns 0 (or the task's nominal nice).
- [ ] AC-8: Cross-pid-ns query returns -ESRCH.
- [ ] AC-9: 1e6 concurrent calls return consistent values.
- [ ] AC-10: getpriority does not mutate any state.
- [ ] AC-11: `getpriority(PRIO_USER, 0)` for a uid with no tasks returns -ESRCH.
- [ ] AC-12: Strace shows correct wire decoding (kernel returns 0..40, displayed as -20..19).

## Architecture

```rust
#[syscall(nr = 140, abi = "sysv")]
pub fn sys_getpriority(which: i32, who: u32) -> SysResult<i32> {
    Getpriority::do_call(which, who)
}
```

`Getpriority::do_call(which, who) -> i32`:
1. let mut best = NICE_MAX + 1;   // sentinel
2. let mut found = false;
3. rcu_read_lock();
4. match which {
       PRIO_PROCESS => {
           let task = if who == 0 { current() } else { find_task_by_vpid(who).ok_or(-ESRCH)? };
           if security_task_getpriority(task).is_ok() {
               best = task_nice(task);
               found = true;
           }
       }
       PRIO_PGRP => {
           let pgid = if who == 0 { task_pgrp_vnr(current()) } else { who };
           for_each_task_in_pgrp(pgid, |task| {
               if security_task_getpriority(task).is_ok() {
                   best = min(best, task_nice(task));
                   found = true;
               }
           });
       }
       PRIO_USER => {
           let uid = if who == 0 { current().real_cred.uid } else { make_kuid(current().user_ns, who).ok_or(-EINVAL)? };
           for_each_process(|task| {
               if task.real_cred.uid == uid && security_task_getpriority(task).is_ok() {
                   best = min(best, task_nice(task));
                   found = true;
               }
           });
       }
       _ => { rcu_read_unlock(); return -EINVAL; }
   }
5. rcu_read_unlock();
6. if !found { return -ESRCH; }
7. /* Wire ABI: return 20 - nice */
8. (20 - best) as i32

`task_nice(task) -> i32`:
1. /* PRIO_TO_NICE conversion */
2. (task.static_prio - MAX_RT_PRIO - NICE_WIDTH/2) as i32

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `which_validated` | INVARIANT | per-call: which ∈ {0,1,2} or -EINVAL. |
| `rcu_balanced` | INVARIANT | per-call: every rcu_read_lock paired with unlock on all paths. |
| `no_mutation` | INVARIANT | per-call: no task state written. |
| `lsm_hook_invoked` | INVARIANT | per-task examined: security_task_getpriority called. |
| `wire_abi_offset` | INVARIANT | per-call: kernel return = 20 - nice ∈ [1, 40]. |
| `pgrp_user_min` | INVARIANT | per-PRIO_PGRP/USER: result is min over matching tasks. |
| `pidns_visibility` | INVARIANT | per-call: cross-ns tasks invisible. |

### Layer 2: TLA+

`kernel/getpriority.tla`:
- States: per-task nice + per-process-group + per-user mapping.
- Properties:
  - `safety_min_over_matching` — per-call: result minimum over visible matching set.
  - `safety_no_mutation` — per-call: no state change.
  - `safety_esrch_on_empty` — per-call: empty matching set ⟹ -ESRCH.
  - `liveness_terminates` — per-call: O(nr_visible_tasks).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: PRIO_PROCESS → ret == 20 - task_nice(target) ∨ errno | `Getpriority::do_call` |
| `do_call` post: PRIO_PGRP/USER → ret == 20 - min(nice over matching) ∨ -ESRCH | `Getpriority::do_call` |
| `task_nice` post: result ∈ [-20, 19] | `Sched::task_nice` |

### Layer 4: Verus / Creusot functional

Per-`getpriority(2)` man-page equivalence. LTP `getpriority01..02` pass. POSIX.1-2008 (XSI option) semantics verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getpriority(2)` reinforcement:

- **Per-which validation** — defense against per-OOB enum value.
- **Per-RCU-balanced iteration** — defense against per-leak on partial-iteration error paths.
- **Per-LSM hook per task** — defense against per-info-leak when LSM denies introspection.
- **Per-namespace isolation** — defense against per-cross-container task enumeration.
- **Per-no-mutation** — defense against per-side-effect-on-read.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF (n/a for this call)** — getpriority takes no userspace pointers; included for completeness.
- **No CAP requirement** — POSIX-compliant; grsec respects GRKERNSEC_PROC_USERGROUP so cross-user iteration is restricted: only tasks visible to the caller are examined. Cross-user/group tasks are silently skipped (treated as if security_task_getpriority denied).
- **GRKERNSEC_RESLOG on policy enumeration** — repeated PRIO_USER iterations from sandboxed tasks are rate-logged to detect reconnaissance / load-fingerprinting.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a chroot, PRIO_PGRP and PRIO_USER iterate only tasks inside the chroot's task set; outside tasks are invisible.
- **Sched_deadline attack-surface reduction** — querying PRIO_USER/PGRP from inside a sandbox is permitted but audited; SCHED_DEADLINE tasks return their nominal nice (0).
- **GRKERNSEC_PROC_USERGROUP for PRIO_USER** — querying another uid's tasks returns -ESRCH (not the minimum nice) under hardened mode, to prevent cross-user load fingerprinting.
- **No_new_privs neutral** — read-only call; NNP has no effect.
- **KEEPCAPS neutral** — no capability check.
- **Nice clamping** — under sandbox flags, the kernel may report a clamped nice (max of actual nice and sandbox-floor) so that userspace observability does not leak true scheduling intent.
- **Bounded iteration** — PRIO_USER iteration via for_each_process is bounded by RLIMIT_NPROC for the target user; grsec ensures the loop is interruptible by signal so it cannot be weaponized into a kernel-side stall.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `setpriority(2)` (Tier-5 separate doc).
- `nice(2)` (Tier-5 separate doc — legacy wrapper).
- CFS internals / nice-to-weight conversion (Tier-3 in `kernel/sched-fair.md`).
- Implementation code.
