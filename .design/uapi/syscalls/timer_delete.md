# Tier-5 syscall: timer_delete(2) — syscall 226

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/posix-timers.c (sys_timer_delete, do_timer_delete, common_timer_del)
  - kernel/time/alarmtimer.c (alarm_timer_del)
  - kernel/time/hrtimer.c (hrtimer_cancel)
  - kernel/exit.c (exit_itimers)
  - arch/*/include/generated/uapi/asm/unistd_64.h (226)
  - Documentation/core-api/timekeeping.rst, man timer_delete(2)
-->

## Summary

`timer_delete(2)` releases a POSIX per-process timer previously created by `timer_create(2)`: cancels any pending expiration, removes the timer from the per-process IDR, and frees the underlying `k_itimer`. Pending but undelivered signals already enqueued for this timer are dequeued from the per-process / per-thread signal queue, ensuring that no `SIGEV_SIGNAL` carrying a stale `si_timerid` is observed after the call returns successfully. The freed `timer_t` value is available for reuse by future `timer_create` calls. All timers owned by a process are automatically deleted on `execve(2)` and on the last thread's exit via `exit_itimers()`. Critical for: timer lifecycle hygiene (event-loops releasing per-connection deadlines), shutdown ordering (free all timers before close-on-exec'd fds), preventing `si_overrun` blow-up on long-lived SIGEV_NONE timers, releasing the `RLIMIT_SIGPENDING` slot.

This Tier-5 covers the userspace ABI of syscall 226. Per-`hrtimer_cancel` synchronous semantics and per-`exit_itimers` exit-time bulk-delete are owned by `kernel/time/hrtimer.md` and `kernel/exit.md` (Tier-3, planned).

## Signature

```c
int timer_delete(timer_t timerid);
```

Rust ABI shim:

```rust
pub fn sys_timer_delete(timerid: TimerId) -> isize;
```

Syscall number: **226**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `timerid` | `timer_t` | IN | timer handle from `timer_create` |

## Return

- **Success**: `0`; timer cancelled, IDR slot freed, pending signals dequeued.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `timerid` does not name a live timer in current process |

## ABI surface

No structures; just the `timer_t`.

## Compatibility contract

REQ-1: `timerid` lookup:
- `posix_timer_get_locked(current.signal, timerid)` MUST find a live `k_itimer` whose `it_signal == current.signal`; else `-EINVAL`.
- Lookup taken under `current->sighand->siglock`.

REQ-2: Cancellation:
- `kc.timer_del(&mut k)` cancels the underlying timekeeping primitive:
  - hrtimer-backed: `hrtimer_cancel` (synchronous wait if callback executing on another CPU).
  - alarm-backed: `alarm_cancel` (synchronous).
  - CPU-time-backed: `posix_cpu_timer_del` removes from per-task list.
- After cancel, no further firings can occur.

REQ-3: Pending-signal cleanup:
- After cancel, the kernel walks the per-thread-group signal queue and removes any queued `siginfo` with `si_code == SI_TIMER ∧ si_timerid == this`.
- This prevents handlers from observing a stale `si_timerid` after delete returns.

REQ-4: IDR release:
- `posix_timer_unhash(current.signal, k.id)` removes the entry; the timer_t value becomes available for reuse.

REQ-5: Memory release:
- `kfree(k)` (RCU-deferred via `SLAB_TYPESAFE_BY_RCU`).

REQ-6: Quota release:
- The slot count against `RLIMIT_SIGPENDING` is released.

REQ-7: Concurrent delivery:
- A signal queued just before `timer_delete` MAY still be delivered (race window between hrtimer-callback and siglock acquisition).
- After successful return, no new signal CAN be queued; the cleanup pass dequeues any queued but undelivered ones.

REQ-8: `exit_itimers`:
- Called at last-thread exit and at `execve` (post-fork-cleanup): walks `current.signal.posix_timers` IDR, calls `timer_delete` semantics on each.
- Ordering: `exit_itimers` runs BEFORE signal queue final drain, so per-timer cleanup is safe.

REQ-9: No capability requirement:
- Caller deletes timers they own.

REQ-10: Idempotency:
- A successfully-deleted `timer_t` returns `-EINVAL` on subsequent `timer_delete` (or any `timer_*` syscall).

## Acceptance Criteria

- [ ] AC-1: Arm a timer; delete; subsequent `timer_gettime` returns `-EINVAL`.
- [ ] AC-2: Arm with 10ms interval; delete; no further SIGEV_SIGNAL signals delivered.
- [ ] AC-3: Already-pending signal at delete-time is dequeued before return.
- [ ] AC-4: Deleted timer_t may be reused by a subsequent `timer_create` (same int value possible).
- [ ] AC-5: Per-process timer count decremented; new create succeeds within RLIMIT.
- [ ] AC-6: Double-delete returns `-EINVAL`.
- [ ] AC-7: `timerid` from another process: `-EINVAL`.
- [ ] AC-8: `execve` deletes all timers; child of fork starts with empty IDR.
- [ ] AC-9: Last-thread exit deletes all timers (no signal leaks to next process via PID reuse).
- [ ] AC-10: hrtimer callback executing concurrently: timer_delete waits for it (synchronous cancel).
- [ ] AC-11: Alarm-class timer delete cancels RTC wakeup source.

## Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
pub fn sys_timer_delete(id: TimerId) -> isize {
    PosixTimers::do_timer_delete(id)
}
```

`PosixTimers::do_timer_delete(id) -> isize`:
1. let (k, guard) = posix_timer_get_locked(&current.signal, id).ok_or(-EINVAL)?;
2. let kc = &posix_clock_kclocks[k.it_clock as usize];
3. /* mark deleted so concurrent fires no-op */
4. k.it_deleted = true;
5. drop(guard);
6. /* synchronous cancel (may sleep waiting on remote CPU callback) */
7. kc.timer_del(&mut k);
8. /* re-acquire siglock for IDR removal + signal cleanup */
9. let guard = current.sighand.siglock.lock_irq();
10. posix_timer_unhash(&current.signal, id);
11. dequeue_pending_signals_for_timer(&current.signal, id);
12. drop(guard);
13. /* RCU-deferred free */
14. KItimer::free_rcu(k);
15. 0

`common_timer_del(k)` (hrtimer-backed):
1. hrtimer_cancel(&k.it.real.timer);
2. k.it_active = false;
3. k.it_interval = 0;

`alarm_timer_del(k)` (alarm-class):
1. alarm_cancel(&k.it.alarm.alarm);
2. k.it_active = false;

`exit_itimers(signal)` (called at exec/exit):
1. for k in signal.posix_timers.drain():
   - kc = &posix_clock_kclocks[k.it_clock as usize];
   - kc.timer_del(&mut k);
   - KItimer::free_rcu(k);
2. signal.posix_timers.clear();

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `siglock_during_lookup_and_unhash` | INVARIANT | both lookup and IDR removal under sighand siglock |
| `synchronous_cancel_before_free` | INVARIANT | hrtimer_cancel completes before KItimer::free |
| `idr_slot_released_on_success` | INVARIANT | posix_timer_unhash called iff timer_del succeeded |
| `no_signal_delivery_post_return` | INVARIANT | pending SI_TIMER signals dequeued before return |
| `rcu_grace_period_for_free` | INVARIANT | k_itimer freed via call_rcu/free_rcu |

### Layer 2: TLA+

`uapi/timer_delete.tla`:
- Variables: `posix_timers[uid]`, `signal_queue`, `it_active[id]`.
- Properties:
  - `safety_no_post_delete_delivery` — no SIGEV_SIGNAL with si_timerid == deleted-id observed by handler after timer_delete returns.
  - `safety_idr_release` — id available for reuse post-success.
  - `liveness_timer_delete_terminates` — even with concurrent hrtimer callback, timer_delete eventually returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_timer_delete` post(0): IDR has no id; k_itimer freed | `do_timer_delete` |
| `do_timer_delete` post(0): signal queue has no SI_TIMER for id | `dequeue_pending_signals_for_timer` |
| `do_timer_delete` post(0): RLIMIT slot released | `do_timer_delete` |
| `exit_itimers` post: signal.posix_timers empty | `exit_itimers` |

### Layer 4: Verus/Creusot functional

Per-`timer_delete(2)` man page, glibc `sysdeps/unix/sysv/linux/timer_delete.c`, LTP `testcases/kernel/syscalls/timer_delete/*`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

timer_delete reinforcement:

- **Synchronous hrtimer cancel** — waits for in-flight callback; no UAF if callback re-arms.
- **Pending-signal dequeue** — no stale si_timerid observed by handler post-delete.
- **RCU-deferred free** — concurrent reader observes type-safe memory.
- **Idempotent rejection** — double-delete safely errors.
- **`exit_itimers` covers exec/exit** — no leak across image boundaries.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on every entry** — randomized kstack on each call; relevant because timer_delete cross-cuts hrtimer wheel, signal queue, and slab subsystems.
- **GRKERNSEC_HIDESYM on `posix_clock_kclocks`** — function-pointer table hidden from kallsyms; defense against indirect-call abuse during `kc.timer_del` dispatch.
- **`SLAB_TYPESAFE_BY_RCU` for k_itimer slab** — UAF defense; even if attacker times a free/realloc race against a concurrent hrtimer callback, the slab remains type-safe and field offsets validated.
- **Per-uid timer_delete audit** — hardened policy logs `(uid, pid, timer_id)` on each delete; defenders correlate delete patterns with subsequent timer_create floods.
- **Foreign-process delete attempt rejected pre-cancel** — `it_signal != current.signal` rejected without invoking `hrtimer_cancel`; defense against cross-process timer-thrashing.
- **Reject delete of a timer mid-fire when its sigev_signo is queued to PID 1 of init userns** — hardened policy enforces a final-delivery-or-reject barrier so that a privileged subsystem cannot be tricked into a missed-signal state.
- **GRKERNSEC_POSIX_TIMER_INTEGRITY canary** — k_itimer holds an inline canary checked at delete; corruption (e.g., from buffer overflow into the timer slab) triggers a hardened-mode OOPS rather than silently freeing arbitrary memory.
- **`exit_itimers` runs under per-task `signal_struct.cred` snapshot** — even if credentials change during exit teardown, timer deletion sees a consistent uid for audit.
- **Block double-delete via marker bit `it_deleted`** — set early in delete path; subsequent lookups treat the timer as gone even before unhash completes; defense against TOCTOU.
- **No leakage of `timer_t` reuse pattern** — hardened policy randomizes (via xorshift seed per-signal-struct) the next-id selection so the freed id is not immediately reused, frustrating fingerprinting of timer-table allocation by adversarial userspace.
- **Pending-signal dequeue audited** — if dequeue removes a non-trivial number of pending SI_TIMER signals (e.g., > 16), hardened policy logs the count to alert defenders to potential SIGEV_NONE-overrun amplification setups.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `timer_create.md`, `timer_settime.md`, `timer_gettime.md`, `timer_getoverrun.md` siblings.
- `kernel/time/hrtimer.md` Tier-3: hrtimer_cancel synchronization.
- `kernel/time/alarmtimer.md` Tier-3: alarm_cancel.
- `kernel/exit.md` Tier-3: exit_itimers ordering during teardown.
- `kernel/time/posix_timers.md` Tier-3: IDR lifecycle.
- `signalfd.md` / `rt_sigtimedwait.md` consumers.
- Implementation code.
