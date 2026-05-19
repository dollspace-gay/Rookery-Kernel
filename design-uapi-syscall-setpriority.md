---
title: "Tier-5 syscall: setpriority(2) — syscall 141"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`setpriority(2)` sets the **nice value** of one of: a single process (PRIO_PROCESS), a process group (PRIO_PGRP), or a user (PRIO_USER) — for the latter two, every matching task is updated. Nice values range from -20 (highest priority) to +19 (lowest). Lowering nice (i.e., increasing priority) requires CAP_SYS_NICE; raising nice (decreasing priority) is free for the owner. Behind the scenes, the kernel translates nice → CFS weight via the static `sched_prio_to_weight[]` table (e.g., nice 0 → 1024, nice -5 → ~3121, nice 19 → 15). Critical for: `renice` tool, container workload deprioritization, sandbox demotion of untrusted tasks, batch-job scheduling.

### Acceptance Criteria

- [ ] AC-1: `setpriority(PRIO_PROCESS, 0, 10)` → 0; subsequent `getpriority(PRIO_PROCESS, 0)` returns 10.
- [ ] AC-2: `setpriority(PRIO_PROCESS, 0, -10)` without CAP_SYS_NICE and RLIMIT_NICE=20 → -EPERM.
- [ ] AC-3: `setpriority(PRIO_PROCESS, 0, -10)` with RLIMIT_NICE=30 → 0.
- [ ] AC-4: `setpriority(PRIO_PROCESS, 0, 100)` clamped to 19; returns 0.
- [ ] AC-5: `setpriority(PRIO_PROCESS, 0, -100)` clamped to -20; requires CAP_SYS_NICE.
- [ ] AC-6: `setpriority(3, 0, 5)` (invalid which) → -EINVAL.
- [ ] AC-7: `setpriority(PRIO_PROCESS, foreign_uid_pid, 5)` without CAP_SYS_NICE → -EPERM.
- [ ] AC-8: `setpriority(PRIO_PGRP, my_pgid, 5)` → all my group tasks niced to 5.
- [ ] AC-9: `setpriority(PRIO_USER, my_uid, 10)` → all my tasks niced to 10.
- [ ] AC-10: `setpriority(PRIO_USER, foreign_uid, 5)` without CAP_SYS_NICE → -EPERM.
- [ ] AC-11: LSM denial → -EACCES.
- [ ] AC-12: setpriority on SCHED_FIFO task: nice updated, RT priority unchanged.

### Architecture

```rust
#[syscall(nr = 141, abi = "sysv")]
pub fn sys_setpriority(which: i32, who: u32, prio: i32) -> SysResult<i32> {
    Setpriority::do_call(which, who, prio)
}
```

`Setpriority::do_call(which, who, prio) -> i32`:
1. /* Clamp prio */
2. let prio = prio.clamp(NICE_MIN, NICE_MAX);
3. let mut found = false;
4. let mut last_err = 0i32;
5. rcu_read_lock();
6. match which {
       PRIO_PROCESS => {
           let task = if who == 0 { current() } else { find_task_by_vpid(who).ok_or(-ESRCH)? };
           found = true;
           last_err = Setpriority::set_one(task, prio);
       }
       PRIO_PGRP => {
           let pgid = if who == 0 { task_pgrp_vnr(current()) } else { who };
           for_each_task_in_pgrp(pgid, |task| {
               found = true;
               let e = Setpriority::set_one(task, prio);
               if e != 0 { last_err = e; }
           });
       }
       PRIO_USER => {
           let uid = if who == 0 { current().real_cred.uid } else { make_kuid(current().user_ns, who).ok_or(-EINVAL)? };
           for_each_process(|task| {
               if task.real_cred.uid == uid {
                   found = true;
                   let e = Setpriority::set_one(task, prio);
                   if e != 0 { last_err = e; }
               }
           });
       }
       _ => { rcu_read_unlock(); return -EINVAL; }
   }
7. rcu_read_unlock();
8. if !found { return -ESRCH; }
9. if last_err < 0 { return last_err; }   // propagate first non-success
10. 0

`Setpriority::set_one(task, prio) -> i32`:
1. /* Permission gate */
2. if task.real_cred.uid != current().euid && !capable(CAP_SYS_NICE) { return -EPERM; }
3. let cur_nice = task_nice(task);
4. if prio < cur_nice {
       /* lowering nice = elevation */
       if !capable(CAP_SYS_NICE) {
           let cap = NICE_MAX - task_rlimit(task, RLIMIT_NICE) as i32 + 1;
           if prio < cap { return -EPERM; }
       }
   }
5. /* LSM */
6. security_task_setnice(task, prio)?;     // -EACCES
7. /* Apply */
8. Sched::set_user_nice(task, prio);       // dequeue/reweight/enqueue under rq lock
9. 0

`Sched::set_user_nice(task, nice)`:
1. let rq = task_rq_lock(task);
2. let was_queued = task.on_rq;
3. if was_queued { dequeue_task(rq, task, DEQUEUE_SAVE); }
4. task.static_prio = NICE_TO_PRIO(nice);
5. set_load_weight(task, true);
6. if was_queued { enqueue_task(rq, task, ENQUEUE_RESTORE); resched_curr(rq); }
7. task_rq_unlock(rq);

### Out of Scope

- `getpriority(2)` (Tier-5 separate doc).
- `nice(2)` (Tier-5 separate doc — legacy wrapper).
- CFS weight / load tracking (Tier-3 in `kernel/sched-fair.md`).
- Implementation code.

### signature

```c
int setpriority(int which, id_t who, int prio);
```

### parameters

| Parameter | Type | Direction | Description |
|---|---|---|---|
| `which` | `int` | in | PRIO_PROCESS (0), PRIO_PGRP (1), PRIO_USER (2). |
| `who` | `id_t` | in | Target ID: PID / PGID / UID (0 = self). |
| `prio` | `int` | in | New nice value in [-20, 19]. Out-of-range values are clamped. |

### return value

| Value | Meaning |
|---|---|
| 0 | Success; all matching tasks updated. |
| `-1` + errno | Error; see errors. |

### errors

| Errno | Trigger |
|---|---|
| `EINVAL` | which not in {PRIO_PROCESS, PRIO_PGRP, PRIO_USER}. |
| `ESRCH` | No matching process/group/user found. |
| `EPERM` | Lowering nice (priority elevation) without CAP_SYS_NICE / over RLIMIT_NICE; foreign-uid target without CAP_SYS_NICE. |
| `EACCES` | LSM denial (security_task_setnice). |

### abi surface

```text
__NR_setpriority (x86_64) = 141
__NR_setpriority (i386)   = 97
__NR_setpriority (arm64)  = 140  (generic-syscall)
__NR_setpriority (generic) = 140

PRIO_PROCESS = 0
PRIO_PGRP    = 1
PRIO_USER    = 2

NICE_MIN     = -20
NICE_MAX     =  19
NICE_WIDTH   =  40

/* Clamping behavior */
prio < -20  → -20
prio >  19  →  19
```

### compatibility contract

REQ-1: Syscall number is **141** on x86_64; **140** on arm64/generic.

REQ-2: which MUST be one of PRIO_PROCESS, PRIO_PGRP, PRIO_USER; otherwise -EINVAL.

REQ-3: prio is **clamped** to [-20, 19]: values outside are silently clamped (not rejected). Older kernels rejected with -EACCES; modern Linux clamps. Rookery follows modern: clamp.

REQ-4: Lowering nice (i.e., making prio more negative than current) requires CAP_SYS_NICE in the caller's user namespace, OR the new nice value ≥ -RLIMIT_NICE (the RLIMIT_NICE soft-rlimit, expressed as a positive number for "20 - allowed-min-nice").

REQ-5: Raising nice (making prio more positive) is permitted for the task owner without CAP_SYS_NICE. The owner cannot later lower it back without CAP_SYS_NICE (this is the rationale for RLIMIT_NICE — it lets unprivileged users select an initial ceiling and reach it).

REQ-6: Modifying a task owned by a different uid requires CAP_SYS_NICE.

REQ-7: PRIO_PROCESS: target single task by PID (0 = self).

REQ-8: PRIO_PGRP: target all tasks with task_pgrp == who (0 = self group). EACH task is permission-checked independently. The kernel applies the change to ALL permitted tasks, then returns 0 if at least one task was modified or -ESRCH if none.

REQ-9: PRIO_USER: target all tasks with task.real_cred.uid == who (0 = self uid). Same per-task permission model as PGRP.

REQ-10: Per-task LSM hook `security_task_setnice(task, prio)` fires; denial ⟹ -EACCES (overrides the success path).

REQ-11: For SCHED_FIFO/RR/DEADLINE tasks, setpriority changes only the nominal nice; it does NOT affect rt_priority. CFS scheduling is unaffected since the task is not on the CFS queue.

REQ-12: For CFS tasks, the change updates `task.static_prio = NICE_TO_PRIO(prio)` and re-weights the task. If runnable, the task is dequeued, reweighed via `set_user_nice`, then re-enqueued under task_rq_lock.

REQ-13: For an in-flight (running) task, the change is reflected on next scheduler tick (or sooner, via re-enqueue).

REQ-14: RLIMIT_NICE: the soft rlimit value `N` permits unprivileged nice down to `20 - N`. E.g., RLIMIT_NICE=30 permits nice -10; RLIMIT_NICE=40 permits nice -20.

REQ-15: The return value is 0 on success (any matching task updated), or -1 + errno otherwise. Note: success does NOT mean ALL tasks were updated; some may have been skipped due to per-task LSM denial.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prio_clamped` | INVARIANT | per-call: effective prio ∈ [-20, 19]. |
| `which_validated` | INVARIANT | per-call: which ∈ {0,1,2} or -EINVAL. |
| `cap_or_rlimit_for_lowering` | INVARIANT | per-task: nice decrease ⟹ CAP_SYS_NICE or RLIMIT_NICE permits. |
| `foreign_uid_cap` | INVARIANT | per-task: cross-uid ⟹ CAP_SYS_NICE. |
| `rq_lock_held` | INVARIANT | per-set_user_nice: rq lock held over dequeue→reweight→enqueue. |
| `lsm_hook_invoked` | INVARIANT | per-task: security_task_setnice called pre-commit. |
| `rcu_balanced` | INVARIANT | per-call: rcu_read_lock paired on all paths. |

### Layer 2: TLA+

`kernel/setpriority.tla`:
- States: per-task nice/static_prio, per-CFS weight.
- Properties:
  - `safety_prio_in_range` — per-task: nice ∈ [-20, 19] after any successful call.
  - `safety_no_partial_per_task` — per-task: dequeue/reweight/enqueue atomic under rq lock.
  - `safety_perm_enforced` — per-task: -EPERM cases never commit.
  - `liveness_terminates` — per-call: O(nr_matching_tasks).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: prio clamped to [-20,19]; matches per-task per-which iteration | `Setpriority::do_call` |
| `set_one` post: success ⟹ task_nice == prio | `Setpriority::set_one` |
| `set_user_nice` post: static_prio == NICE_TO_PRIO(nice); reweight done | `Sched::set_user_nice` |

### Layer 4: Verus / Creusot functional

Per-`setpriority(2)` man-page equivalence. LTP `setpriority01..04` pass. POSIX.1-2008 (XSI option) semantics verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setpriority(2)` reinforcement:

- **Per-prio clamp** — defense against per-OOB nice value injected by malicious userspace.
- **Per-task permission gating** — defense against per-cross-uid hijack.
- **Per-rq-lock atomic re-weight** — defense against per-half-weighted task observed by load balancer.
- **Per-LSM hook pre-commit** — defense against per-policy bypass.
- **Per-RLIMIT_NICE strict** — defense against per-RT-ceiling bypass.
- **Per-namespace isolation** — defense against per-cross-container task modification.

### grsecurity / pax-style reinforcement

- **PaX UDEREF (n/a for this call)** — setpriority takes no userspace pointers; included for completeness.
- **CAP_SYS_NICE strict for lowering nice** — grsec retains the upstream rule: nice-lowering (priority elevation) requires CAP_SYS_NICE or RLIMIT_NICE. Under GRKERNSEC_HARDEN, RLIMIT_NICE alone is insufficient — CAP_SYS_NICE is required.
- **GRKERNSEC_RESLOG on rlimit violations** — every -EPERM from RLIMIT_NICE ceiling is logged with task name/uid/target pid/attempted nice.
- **Sched_deadline attack-surface reduction** — setpriority on a SCHED_DEADLINE task is permitted (updates nominal nice) but audited under sandbox so RT/deadline reconnaissance is observable.
- **GRKERNSEC_CHROOT_NICE** — inside a grsec chroot, setpriority is permitted only for tasks inside the chroot's task set; PRIO_USER iteration is also confined.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset per call.
- **No_new_privs gate** — when current has PR_SET_NO_NEW_PRIVS, lowering nice (priority elevation) is denied with -EPERM regardless of capability or RLIMIT_NICE. Raising nice (deprioritizing) is permitted.
- **Nice clamping under sandbox** — under sandbox flags, the effective nice is clamped to [SANDBOX_NICE_MIN, SANDBOX_NICE_MAX] (e.g., [0, 19]) so sandboxed tasks cannot elevate priority even if capabilities or rlimits would otherwise permit.
- **PRIO_USER cross-uid lockdown** — PRIO_USER targeting a foreign uid requires CAP_SYS_NICE; under hardened mode, even CAP_SYS_NICE is insufficient — only CAP_SYS_ADMIN suffices.
- **KEEPCAPS aware** — capability check honors effective set; SECBIT_KEEP_CAPS does not bypass.
- **Bounded iteration** — PRIO_PGRP and PRIO_USER iterations are bounded by RLIMIT_NPROC; the loop is interruptible by signal so it cannot be weaponized into a kernel-side stall.

