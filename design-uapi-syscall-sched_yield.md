---
title: "Tier-5 syscall: sched_yield(2) — syscall 24"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sched_yield(2)` relinquishes the CPU **back to the run queue at the end of its scheduling class**, allowing other runnable tasks of equal-or-higher priority to run. For SCHED_FIFO/RR, the task is moved to the tail of its priority list. For SCHED_NORMAL/BATCH/IDLE (CFS), the task's vruntime is bumped to ensure it is not picked again immediately; semantics are intentionally weak — the kernel may schedule the yielding task again immediately if it is the only runnable task. For SCHED_DEADLINE, sched_yield sets `dl_yielded=1` and causes the task to throttle for the remainder of its current period (effectively donating its remaining runtime). Critical for: spinlocks-with-yield in user space, priority-inheritance schedulers in glibc, cooperative-multitasking emulation, RT-FIFO/RR round-robin reset. Linus famously called sched_yield "broken for anything other than RT" and Rookery enforces strict semantics matching POSIX intent.

### Acceptance Criteria

- [ ] AC-1: `sched_yield()` returns 0.
- [ ] AC-2: SCHED_FIFO/RR task A at prio 50 calling yield while task B at prio 50 is runnable: B runs next.
- [ ] AC-3: SCHED_FIFO task A at prio 90 yielding while only B at prio 50 runnable: A runs next (same priority needed to displace).
- [ ] AC-4: SCHED_NORMAL task yielding with N runnable peers: another task picked next (probability ~1 with large N).
- [ ] AC-5: SCHED_DEADLINE task yielding mid-period: task does not run again until next replenishment.
- [ ] AC-6: Yield in a tight loop completes O(1) per call (does not deadlock or starve).
- [ ] AC-7: No LSM hook fires (verified via SELinux audit).
- [ ] AC-8: schedstat nr_yields counter increments.
- [ ] AC-9: PI-boosted task yielding does NOT lose its inherited priority.
- [ ] AC-10: sched_yield in a single-task system returns 0 with no scheduling change.
- [ ] AC-11: sched_yield is interruptible by signals (signal delivered before return if pending).
- [ ] AC-12: SCHED_IDLE task yielding behaves as CFS yield (lowest weight remains).

### Architecture

```rust
#[syscall(nr = 24, abi = "sysv")]
pub fn sys_sched_yield() -> SysResult<i32> {
    SchedYield::do_call()
}
```

`SchedYield::do_call() -> i32`:
1. /* Sandbox gate (Rookery extension): may deny under no_new_privs or seccomp-yield-banned */
2. SchedYield::sandbox_gate(current())?;        // -EPERM under sandbox-deny
3. let rq = task_rq_lock(current());
4. schedstat_inc(rq.yld_count);
5. /* Dispatch to class hook */
6. match current().sched_class {
       Class::Fair      => Sched::yield_task_fair(&rq, current()),
       Class::Rt        => Sched::yield_task_rt(&rq, current()),
       Class::Deadline  => Sched::yield_task_dl(&rq, current()),
       Class::Idle      => { /* no-op */ }
       _ => {}
   };
7. /* Re-mark need-resched */
8. set_tsk_need_resched(current());
9. task_rq_unlock(rq);
10. preempt_enable_and_schedule();
11. 0

`Sched::yield_task_fair(rq, p)`:
1. /* Bump vruntime so that another CFS task with smaller vruntime preempts */
2. let cfs = rq.cfs_rq_of(p);
3. cfs.skip = Some(p.se);                       // CFS_FEAT_GENTLE_FAIR_SLEEPERS-aware
4. clear_buddies(cfs, p.se);

`Sched::yield_task_rt(rq, p)`:
1. /* Move to tail of priority list */
2. let rt = rq.rt_rq_of(p);
3. requeue_task_rt(rq, p, /*head=*/false);

`Sched::yield_task_dl(rq, p)`:
1. /* Throttle: forfeit remaining runtime until next period */
2. p.dl.dl_yielded = true;
3. p.dl.runtime = 0;
4. update_curr_dl(rq);                          // triggers throttle if runtime ≤ 0

### Out of Scope

- CFS / RT / DEADLINE class internals (Tier-3 in `kernel/sched-core.md`).
- Priority-inheritance mutex (Tier-3 in `kernel/locking/rtmutex.md`).
- futex FUTEX_WAKE / FUTEX_REQUEUE (Tier-5 separate docs).
- Implementation code.

### signature

```c
int sched_yield(void);
```

### parameters

(none — `SYSCALL_DEFINE0(sched_yield)`)

### return value

| Value | Meaning |
|---|---|
| 0 | Success; control returns after yielding. |

`sched_yield` cannot fail under POSIX. Always returns 0.

### errors

(none — POSIX-mandated never-fail)

### abi surface

```text
__NR_sched_yield (x86_64) = 24
__NR_sched_yield (i386)   = 158
__NR_sched_yield (arm64)  = 124  (generic-syscall)
__NR_sched_yield (generic) = 124

/* No userspace structures. */
```

### compatibility contract

REQ-1: Syscall number is **24** on x86_64; **124** on arm64/generic. ABI-stable since 2.0.

REQ-2: Always returns 0. POSIX mandates no error path.

REQ-3: For SCHED_FIFO/SCHED_RR: the task is moved to the **tail** of its priority list. If other tasks of equal priority exist, the next pick goes to one of them; otherwise the same task is rescheduled.

REQ-4: For SCHED_NORMAL/SCHED_BATCH/SCHED_IDLE (CFS): the task's vruntime is set so that another CFS task with smaller-or-equal vruntime gets selected. If no such task exists, the yielding task may be rescheduled immediately.

REQ-5: For SCHED_DEADLINE: dl_yielded=1; the task is throttled for the remainder of its replenishment period (the remaining runtime is forfeited). The next replenishment fires at the period boundary.

REQ-6: The kernel does not provide a hard guarantee that another task will be selected — only that the current task is moved to a position where it is not preferred.

REQ-7: The call invokes `schedule()` after marking; if `current` remains the highest-priority runnable task, the same task may be picked immediately.

REQ-8: No locks held across schedule(); preemption is enabled around the yield point.

REQ-9: Not async-signal-safe in glibc (libc wraps but the syscall itself has no signal-handling effects).

REQ-10: No LSM hook fires; no capability gate.

REQ-11: PI-mutex with priority inheritance: sched_yield does NOT unwind boosted priority. The boosted task at a higher priority still benefits.

REQ-12: For SCHED_DEADLINE tasks: yielding does NOT cause -EBUSY or any error; bandwidth is consumed-but-throttled.

REQ-13: The schedstat counter `nr_yields` is incremented per call (per-task and per-rq).

REQ-14: When the entire system has only one runnable task at the highest priority, sched_yield is effectively a no-op (modulo the rescheduling round-trip cost).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `always_returns_zero` | INVARIANT | per-call: return == 0 on POSIX path. |
| `rq_lock_held_during_dispatch` | INVARIANT | per-call: rq lock held over class hook invocation. |
| `preempt_re_enabled` | INVARIANT | per-call: preemption re-enabled before return. |
| `no_state_corruption` | INVARIANT | per-call: only documented fields updated (vruntime, dl_yielded, RT list pos). |
| `dl_yield_throttles` | INVARIANT | per-DEADLINE: dl_yielded ⟹ runtime forced ≤ 0 ⟹ throttle on next tick. |
| `rt_tail_requeue` | INVARIANT | per-FIFO/RR: tail-requeue preserves prio-list invariants. |

### Layer 2: TLA+

`kernel/sched_yield.tla`:
- States: per-CPU rq with per-class queues; per-task position.
- Transitions: per-yield → class hook → schedule.
- Properties:
  - `safety_fifo_tail_requeue` — per-FIFO/RR yield: yielding task at tail of its prio list.
  - `safety_dl_throttle` — per-DEADLINE yield: runtime ≤ 0, dl_yielded set.
  - `safety_returns_after_schedule` — per-yield: control returns to caller after schedule().
  - `liveness_eventual_progress` — per-yield: yielding task eventually runs again (within bounded time given fairness).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: returns 0 | `SchedYield::do_call` |
| `yield_task_rt` post: task at tail of prio[task.rt_priority] list | `Sched::yield_task_rt` |
| `yield_task_dl` post: dl_yielded ∧ runtime ≤ 0 | `Sched::yield_task_dl` |
| `yield_task_fair` post: cfs.skip == Some(p.se) | `Sched::yield_task_fair` |

### Layer 4: Verus / Creusot functional

Per-`sched_yield(2)` man-page equivalence. LTP `sched_yield01..02` pass. POSIX.1-2008 (XSI option) semantics verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sched_yield(2)` reinforcement:

- **Per-class dispatch via sched_class vtable** — defense against per-wrong-hook on policy change race.
- **Per-rq-lock atomic state update** — defense against per-half-updated yield state.
- **Per-DEADLINE throttle correctness** — defense against per-deadline-bandwidth leak on yield.
- **Per-RT-list invariant preservation** — defense against per-broken-prio-list bisect.
- **Per-no-LSM no-overhead path** — POSIX-mandated minimum-cost.

### grsecurity / pax-style reinforcement

- **PaX UDEREF (n/a for this call)** — sched_yield takes no userspace pointers; included for completeness.
- **No CAP requirement** — POSIX mandates unprivileged.
- **GRKERNSEC_RESLOG on rlimit violations (n/a here)** — sched_yield does not interact with rlimits; included for completeness with the rest of the scheduling family.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset per call; sched_yield is a hot path so RANDKSTACK uses a low-entropy fast path that doesn't dominate yield latency.
- **GRKERNSEC_SCHED_DENY_YIELD_UNDER_SANDBOX (Rookery extension)** — under sandbox flags (seccomp filter, no_new_privs), sched_yield can be configured to return -EPERM rather than yield. Rationale: in adversarial sandboxes, sched_yield is a known side-channel (yield-then-measure timing) and a DoS amplifier (rapid yield loops). Default policy: deny under hardened sandbox; permit under default.
- **GRKERNSEC_SCHED_HARDEN nice clamping** — when yield is denied under sandbox, the syscall returns -EPERM and increments a GRKERNSEC audit counter (rate-limited log).
- **Sched_deadline attack-surface reduction** — under sandbox flags, SCHED_DEADLINE tasks calling sched_yield trigger an audit log; the yield is permitted but the audit catches RT-bandwidth abuse patterns.
- **Per-task yield rate limit** — Rookery extension: per-task counter caps yields/sec at YIELD_RATE_LIMIT (e.g., 1e5/s); exceeding triggers a back-off (the task is forced into a 1ms sleep). This defeats spinlock-with-yield DoS.
- **No_new_privs interaction** — sched_yield itself doesn't elevate privileges, but under NNP combined with sandbox-deny the syscall returns -EPERM.
- **KEEPCAPS neutral** — no capability check on POSIX path.
- **Schedstat hardening** — yld_count is incremented under the rq lock; grsec adds a per-uid yld_count aggregation that feeds into the rate limiter.

