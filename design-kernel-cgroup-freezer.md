---
title: "Tier-3: kernel/cgroup/freezer.c — cgroup v2 freezer (cgroup.freeze knob, recursive propagation, per-task TIF_FREEZE state machine)"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **cgroup v2 freezer** is the unified-hierarchy mechanism for atomically suspending every task in a cgroup (and its descendants), accounting how many tasks have actually reached frozen state, and propagating an aggregate "frozen" state upward when all descendants have stopped. Per-cgroup user knob `cgroup.freeze` (write `1` to freeze, `0` to thaw) sets the `cgroup->freezer.freeze` bit; the kernel then walks the subtree via `css_for_each_descendant_pre`, computes the **effective freeze** (`e_freeze = freeze || parent.e_freeze`), and for each subtree-cgroup whose effective state changes calls `cgroup_do_freeze`, which (a) sets/clears `CGRP_FREEZE` on the cgroup, (b) updates a seqcount-protected nsec accumulator (`freeze_start_nsec` / `frozen_nsec`), (c) iterates every non-kthread task in the cgroup, and (d) for each task either sets `JOBCTL_TRAP_FREEZE` + `signal_wake_up` (to freeze) or clears it + `wake_up_process` (to thaw). Each task next time it reaches a signal-handling checkpoint calls `cgroup_enter_frozen()`, marking `current->frozen = true`, bumping `cgrp->freezer.nr_frozen_tasks`, and calling `cgroup_update_frozen` which compares `nr_frozen_tasks == __cgroup_task_count(cgrp)` and — if equal AND `CGRP_FREEZE` set — flips `CGRP_FROZEN` and recursively bumps ancestors' `nr_frozen_descendants` counters via `cgroup_propagate_frozen`, eventually setting `CGRP_FROZEN` on parents whose `nr_frozen_descendants == nr_descendants`. The reverse path (`cgroup_leave_frozen`) is conditional: if the freezer is still freezing and `always_leave == false`, the task does *not* decrement the counter but instead re-arms `JOBCTL_TRAP_FREEZE` to bounce back into frozen state — preventing transient unfrozen flicker during signal handling. Migration across cgroups while frozen is handled by `cgroup_freezer_migrate_task`, which adjusts source/dest counters and force-applies the destination's freeze state. Critical for: container checkpoint/restore (CRIU), Android Doze-mode app freezing, Kubernetes Pod-suspend, systemd-Unit freeze (`systemctl freeze foo.service`).

This Tier-3 covers `kernel/cgroup/freezer.c` (~326 lines).

### Acceptance Criteria

- [ ] AC-1: `echo 1 > cgroup.freeze` on a leaf cgroup with N user tasks: within bounded delay (per task signal-handling latency), `nr_frozen_tasks == N` and `CGRP_FROZEN` flag set.
- [ ] AC-2: `echo 1 > cgroup.freeze` on an interior cgroup propagates to every descendant (`css_for_each_descendant_pre`); each descendant's `e_freeze = true`.
- [ ] AC-3: Effective-freeze short-circuit: if a descendant's old `e_freeze == new e_freeze`, the subtree is skipped via `css_rightmost_descendant`.
- [ ] AC-4: kthread in a frozen cgroup is *not* frozen; `cgroup_do_freeze` skips it.
- [ ] AC-5: Frozen task migrated to a non-freezing dst cgroup: `cgroup_freezer_migrate_task` decs src counter, incs dst, and calls `cgroup_freeze_task(task, false)` so task thaws.
- [ ] AC-6: Frozen task receives non-fatal SIGUSR1: `cgroup_leave_frozen(false)` re-arms `JOBCTL_TRAP_FREEZE`; task does not actually thaw while CGRP_FREEZE set; `nr_frozen_tasks` unchanged.
- [ ] AC-7: Frozen task receives SIGKILL: exits via `do_exit` → `cgroup_leave_frozen(true)`; `nr_frozen_tasks` decrements; task observed gone.
- [ ] AC-8: `echo 0 > cgroup.freeze` clears `CGRP_FREEZE` in subtree; each task gets `cgroup_freeze_task(task, false)` → `wake_up_process`; `frozen_nsec` accumulator increases by elapsed-frozen.
- [ ] AC-9: Aggregate frozen flag on parent: when *all* descendants are frozen (`nr_frozen_descendants == nr_descendants`) AND parent has `CGRP_FREEZE` set, parent's `CGRP_FROZEN` flips and propagates further up.
- [ ] AC-10: `cgroup.events` file emits poll notification on every `CGRP_FROZEN` flip and on no-op writes (see REQ-4 trailing notify).
- [ ] AC-11: `freeze_seq` seqcount: a reader observing `freeze_start_nsec` + `frozen_nsec` mid-flip retries and reads a consistent pair.
- [ ] AC-12: Empty cgroup with `CGRP_FREEZE` set: `cgroup_do_freeze` end-of-function reconciliation sets `CGRP_FROZEN` (0 == 0 task count match).
- [ ] AC-13: `WARN_ON_ONCE(nr_frozen_tasks < 0)` on dec — invariant violation surfaces if any path drops the counter without a matching inc.
- [ ] AC-14: `WARN_ON_ONCE(!current->frozen)` in `cgroup_leave_frozen` — invariant violation if leave called when not entered.

### Architecture

```
struct CgroupFreezer {
  freeze: bool,                       // user-set via cgroup.freeze
  e_freeze: bool,                     // effective: self.freeze || parent.e_freeze
  nr_frozen_tasks: i32,               // tasks in this cgroup currently frozen
  nr_frozen_descendants: i32,         // immediate + transitive descendants with CGRP_FROZEN set
  freeze_seq: SeqCount,               // write-seq around freeze_start_nsec / frozen_nsec
  freeze_start_nsec: u64,             // ktime at most-recent freeze entry
  frozen_nsec: u64,                   // accumulated frozen time
}

// Flags on cgroup.flags:
const CGRP_FREEZE: u32 = ...;         // pending effective-freeze on subtree
const CGRP_FROZEN: u32 = ...;         // aggregate "all tasks + descendants frozen"

// On task_struct:
struct Task {
  ...,
  frozen: bool,                       // currently counted in some cgroup.nr_frozen_tasks
  jobctl: u64,                        // JOBCTL_TRAP_FREEZE bit
}
```

`Freezer::freeze(cgrp, freeze)` (top-level cgroup.freeze writer):
1. `lockdep_assert_held(&cgroup_mutex)`.
2. If `cgrp->freezer.freeze == freeze`: return (no-op).
3. `cgrp->freezer.freeze = freeze`.
4. `ts_nsec = ktime_get_ns()`.
5. `applied = false`.
6. for css in `css_for_each_descendant_pre(&cgrp->self)`:
   - `dsct = css->cgroup`.
   - if `cgroup_is_dead(dsct)`: continue.
   - `old_e = dsct->freezer.e_freeze`.
   - `parent = cgroup_parent(dsct)`.
   - `dsct->freezer.e_freeze = dsct->freezer.freeze || parent->freezer.e_freeze`.
   - if `dsct->freezer.e_freeze == old_e`: `css = css_rightmost_descendant(css)`; continue.
   - `Freezer::do_freeze(dsct, freeze, ts_nsec)`.
   - `applied = true`.
7. if `!applied`: `TRACE_CGROUP_PATH(notify_frozen, cgrp, CGRP_FROZEN-state)`; `cgroup_file_notify(&cgrp->events_file)`.

`Freezer::do_freeze(cgrp, freeze, ts_nsec)`:
1. `lockdep_assert_held(&cgroup_mutex)`.
2. `spin_lock_irq(&css_set_lock)`.
3. `write_seqcount_begin(&cgrp->freezer.freeze_seq)`.
4. if `freeze`: `set_bit(CGRP_FREEZE, &cgrp->flags); cgrp->freezer.freeze_start_nsec = ts_nsec`.
5. else: `clear_bit(CGRP_FREEZE, &cgrp->flags); cgrp->freezer.frozen_nsec += (ts_nsec - cgrp->freezer.freeze_start_nsec)`.
6. `write_seqcount_end(&cgrp->freezer.freeze_seq)`.
7. `spin_unlock_irq(&css_set_lock)`.
8. `TRACE_CGROUP_PATH(freeze | unfreeze, cgrp)`.
9. `css_task_iter_start(&cgrp->self, 0, &it)`; per task:
   - if `task->flags & PF_KTHREAD`: skip.
   - `Freezer::freeze_task(task, freeze)`.
10. `css_task_iter_end(&it)`.
11. `spin_lock_irq(&css_set_lock)`.
12. if `cgrp->nr_descendants == cgrp->freezer.nr_frozen_descendants`: `Freezer::update_frozen(cgrp)`.
13. `spin_unlock_irq(&css_set_lock)`.

`Freezer::freeze_task(task, freeze)`:
1. `lock_task_sighand(task, &flags)` — return if task is dying.
2. if `freeze`: `task->jobctl |= JOBCTL_TRAP_FREEZE`; `signal_wake_up(task, false)`.
3. else: `task->jobctl &= ~JOBCTL_TRAP_FREEZE`; `wake_up_process(task)`.
4. `unlock_task_sighand(task, &flags)`.

`Freezer::enter_frozen()` (called from get_signal trap):
1. if `current->frozen`: return.
2. `spin_lock_irq(&css_set_lock)`.
3. `current->frozen = true`.
4. `cgrp = task_dfl_cgroup(current)`.
5. `cgrp->freezer.nr_frozen_tasks += 1`.
6. `Freezer::update_frozen(cgrp)`.
7. `spin_unlock_irq(&css_set_lock)`.

`Freezer::leave_frozen(always_leave)`:
1. `spin_lock_irq(&css_set_lock)`.
2. `cgrp = task_dfl_cgroup(current)`.
3. if `always_leave || !test_bit(CGRP_FREEZE, &cgrp->flags)`:
   - `cgrp->freezer.nr_frozen_tasks -= 1`; `WARN_ON_ONCE(< 0)`.
   - `Freezer::update_frozen(cgrp)`.
   - `WARN_ON_ONCE(!current->frozen)`.
   - `current->frozen = false`.
4. else if `!(current->jobctl & JOBCTL_TRAP_FREEZE)`:
   - `spin_lock(&current->sighand->siglock)`.
   - `current->jobctl |= JOBCTL_TRAP_FREEZE`.
   - `set_thread_flag(TIF_SIGPENDING)`.
   - `spin_unlock(&current->sighand->siglock)`.
5. `spin_unlock_irq(&css_set_lock)`.

`Freezer::update_frozen(cgrp)`:
1. `frozen = test_bit(CGRP_FREEZE, &cgrp->flags) && cgrp->freezer.nr_frozen_tasks == __cgroup_task_count(cgrp)`.
2. if `Freezer::update_frozen_flag(cgrp, frozen)`: `Freezer::propagate_frozen(cgrp, frozen)`.

`Freezer::update_frozen_flag(cgrp, frozen) -> bool`:
1. `lockdep_assert_held(&css_set_lock)`.
2. if `test_bit(CGRP_FROZEN, &cgrp->flags) == frozen`: return false.
3. if `frozen`: `set_bit(CGRP_FROZEN, &cgrp->flags)` else `clear_bit(...)`.
4. `cgroup_file_notify(&cgrp->events_file)`.
5. `TRACE_CGROUP_PATH(notify_frozen, cgrp, frozen)`.
6. return true.

`Freezer::propagate_frozen(cgrp, frozen)`:
1. `desc = 1`.
2. while `(cgrp = cgroup_parent(cgrp))`:
   - if `frozen`:
     - `cgrp->freezer.nr_frozen_descendants += desc`.
     - if `!test_bit(CGRP_FREEZE, &cgrp->flags) || cgrp->freezer.nr_frozen_descendants != cgrp->nr_descendants`: continue.
   - else:
     - `cgrp->freezer.nr_frozen_descendants -= desc`.
   - if `Freezer::update_frozen_flag(cgrp, frozen)`: `desc += 1`.

`Freezer::migrate_task(task, src, dst)`:
1. `lockdep_assert_held(&css_set_lock)`.
2. if `task->flags & PF_KTHREAD`: return.
3. if `!CGRP_FREEZE(src) && !CGRP_FREEZE(dst) && !task->frozen`: return.
4. if `task->frozen`: `dst->freezer.nr_frozen_tasks += 1; src->freezer.nr_frozen_tasks -= 1`.
5. `Freezer::update_frozen(dst); Freezer::update_frozen(src)`.
6. `Freezer::freeze_task(task, test_bit(CGRP_FREEZE, &dst->flags))`.

### Out of Scope

- Cgroup v1 legacy freezer (`legacy_freezer.c`) — separate Tier-3 if expanded.
- Kernel-wide suspend/hibernate freezer (`kernel/power/process.c`, `try_to_freeze`, `PF_FROZEN`) — separate Tier-3.
- `cgroup_can_fork` / `_post_fork` / `_cancel_fork` hooks (lifecycle plumbing in `kernel/cgroup/cgroup.c`).
- `cgroup.events` file backing + `events_file` poll wiring (in `kernel/cgroup/cgroup.c`).
- `css_for_each_descendant_pre` / `css_rightmost_descendant` / `css_task_iter_*` iterator internals (`kernel/cgroup/cgroup.c`).
- `signal_wake_up` / `wake_up_process` scheduler internals (kernel/sched/).
- CRIU / container checkpoint-restore semantics (userspace; out of kernel scope).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cgroup_freezer` (embedded in `cgroup`) | per-cgroup freezer state (freeze, e_freeze, nr_frozen_tasks, nr_frozen_descendants, freeze_seq, freeze_start_nsec, frozen_nsec) | `kernel::cgroup::freezer::CgroupFreezer` |
| `CGRP_FREEZE` (flag bit) | per-cgroup pending-freeze flag | `cgroup::Flag::Freeze` |
| `CGRP_FROZEN` (flag bit) | per-cgroup aggregate frozen flag | `cgroup::Flag::Frozen` |
| `JOBCTL_TRAP_FREEZE` (task jobctl bit) | per-task pending-freeze trap | `task::JobCtl::TrapFreeze` |
| `current->frozen` (task_struct field) | per-task is-frozen-now | `Task::frozen` |
| `cgroup_update_frozen(cgrp)` | per-cgroup state revisit | `Freezer::update_frozen` |
| `cgroup_update_frozen_flag(cgrp, frozen)` | per-cgroup CGRP_FROZEN flip + notify | `Freezer::update_frozen_flag` |
| `cgroup_propagate_frozen(cgrp, frozen)` | per-ancestor counter walk | `Freezer::propagate_frozen` |
| `cgroup_inc_frozen_cnt(cgrp)` / `_dec_frozen_cnt(cgrp)` | per-cgroup nr_frozen_tasks bump | `Freezer::inc_frozen_cnt` / `_dec_frozen_cnt` |
| `cgroup_enter_frozen()` | per-task enter (called from get_signal) | `Freezer::enter_frozen` |
| `cgroup_leave_frozen(always_leave)` | per-task conditional leave | `Freezer::leave_frozen` |
| `cgroup_freeze_task(task, freeze)` | per-task jobctl flip | `Freezer::freeze_task` |
| `cgroup_do_freeze(cgrp, freeze, ts_nsec)` | per-cgroup task iteration + flag flip | `Freezer::do_freeze` |
| `cgroup_freezer_migrate_task(task, src, dst)` | per-task migration freeze adjust | `Freezer::migrate_task` |
| `cgroup_freeze(cgrp, freeze)` | top-level cgroup.freeze write entry | `Freezer::freeze` |
| `__cgroup_task_count(cgrp)` (kernel/cgroup/cgroup.c) | per-cgroup populated task count | `cgroup::task_count` |
| `css_for_each_descendant_pre(css, &cgrp->self)` | per-subtree pre-order iterator | `cgroup::for_each_descendant_pre` |
| `css_rightmost_descendant(css)` | per-iter skip-subtree helper | `cgroup::rightmost_descendant` |
| `css_task_iter_start` / `_next` / `_end` | per-cgroup task iterator | `cgroup::task_iter` |
| `signal_wake_up(task, false)` | per-task signal-pending wake | `signal::wake_up` |
| `wake_up_process(task)` | per-task unconditional wake | `sched::wake_up_process` |
| `cgroup_file_notify(&cgrp->events_file)` | per-cgroup-events notify (poll/inotify) | `cgroup::file_notify` |
| `TRACE_CGROUP_PATH(freeze, ...)` | per-tracepoint | `cgroup::trace` |

### compatibility contract

REQ-1: `struct cgroup_freezer` shape (embedded in `struct cgroup`):
- `freeze: bool` — per-cgroup user-requested freeze (cgroup.freeze knob).
- `e_freeze: bool` — per-cgroup *effective* freeze (`freeze || parent.e_freeze`).
- `nr_frozen_tasks: i32` — count of tasks in this cgroup that have reached frozen state.
- `nr_frozen_descendants: i32` — count of descendants (immediate + transitive) whose CGRP_FROZEN is set.
- `freeze_seq: seqcount_t` — per-cgroup write-seqcount for `freeze_start_nsec` / `frozen_nsec` reader consistency.
- `freeze_start_nsec: u64` — ktime_get_ns timestamp at most-recent freeze start.
- `frozen_nsec: u64` — accumulated nsec-spent-frozen (incremented on unfreeze).

REQ-2: Per-flag bits on `struct cgroup.flags`:
- `CGRP_FREEZE` — propagated effective-freeze flag (set by `cgroup_do_freeze` when freeze argument true, cleared on unfreeze).
- `CGRP_FROZEN` — aggregate frozen flag (set by `cgroup_update_frozen_flag` when `CGRP_FREEZE && nr_frozen_tasks == __cgroup_task_count`).

REQ-3: Per-task `task_struct` fields:
- `frozen: bool` — task is currently in frozen state.
- `jobctl: unsigned long` with bit `JOBCTL_TRAP_FREEZE` — pending freeze trap (consumed by signal-handling path).

REQ-4: `cgroup_freeze(cgrp, freeze)` (top-level write to `cgroup.freeze`):
- `lockdep_assert_held(&cgroup_mutex)`.
- If `cgrp->freezer.freeze == freeze`: return (no-op).
- `cgrp->freezer.freeze = freeze`.
- `ts_nsec = ktime_get_ns()`.
- Walk subtree pre-order via `css_for_each_descendant_pre(css, &cgrp->self)`:
  - `dsct = css->cgroup`.
  - If `cgroup_is_dead(dsct)`: continue.
  - `old_e = dsct->freezer.e_freeze`.
  - `parent = cgroup_parent(dsct)`.
  - `dsct->freezer.e_freeze = (dsct->freezer.freeze || parent->freezer.e_freeze)`.
  - If `e_freeze == old_e`: `css = css_rightmost_descendant(css)` (skip subtree, no change cascades down).
  - Else: `cgroup_do_freeze(dsct, freeze, ts_nsec)`; `applied = true`.
- If `!applied`: emit `TRACE_CGROUP_PATH(notify_frozen, cgrp, CGRP_FROZEN-bit)` and `cgroup_file_notify(&cgrp->events_file)` — user wrote the knob but no actual state change occurred (e.g. already frozen by ancestor or no descendants needing change); we still notify so userspace doesn't wait forever.

REQ-5: `cgroup_do_freeze(cgrp, freeze, ts_nsec)`:
- `lockdep_assert_held(&cgroup_mutex)`.
- `spin_lock_irq(&css_set_lock)`.
- `write_seqcount_begin(&cgrp->freezer.freeze_seq)`.
- If `freeze`:
  - `set_bit(CGRP_FREEZE, &cgrp->flags)`.
  - `cgrp->freezer.freeze_start_nsec = ts_nsec`.
- Else:
  - `clear_bit(CGRP_FREEZE, &cgrp->flags)`.
  - `cgrp->freezer.frozen_nsec += (ts_nsec - cgrp->freezer.freeze_start_nsec)`.
- `write_seqcount_end(&cgrp->freezer.freeze_seq)`.
- `spin_unlock_irq(&css_set_lock)`.
- `TRACE_CGROUP_PATH(freeze, cgrp)` or `(unfreeze, cgrp)`.
- `css_task_iter_start(&cgrp->self, 0, &it)`; iterate; per task:
  - If `task->flags & PF_KTHREAD`: continue (kthreads exempt — freezing them is unsupported and would deadlock kthreadd / per-CPU workers).
  - `cgroup_freeze_task(task, freeze)`.
- `css_task_iter_end(&it)`.
- `spin_lock_irq(&css_set_lock)`; if `cgrp->nr_descendants == cgrp->freezer.nr_frozen_descendants`: `cgroup_update_frozen(cgrp)` (covers empty leaves and already-fully-frozen subtrees); `spin_unlock_irq`.

REQ-6: `cgroup_freeze_task(task, freeze)`:
- `lock_task_sighand(task, &flags)` — bail if task is about to die (sighand gone).
- If `freeze`:
  - `task->jobctl |= JOBCTL_TRAP_FREEZE`.
  - `signal_wake_up(task, false)` — `false` = not a fatal signal; sets `TIF_SIGPENDING` and wakes if sleeping interruptible.
- Else:
  - `task->jobctl &= ~JOBCTL_TRAP_FREEZE`.
  - `wake_up_process(task)` — unconditional wake (task may have been blocked in TASK_KILLABLE / interruptible inside the freeze trap).
- `unlock_task_sighand(task, &flags)`.

REQ-7: `cgroup_enter_frozen()` (called from `get_signal()` when `JOBCTL_TRAP_FREEZE` set):
- If `current->frozen`: return (already counted — idempotent on repeat trap).
- `spin_lock_irq(&css_set_lock)`.
- `current->frozen = true`.
- `cgrp = task_dfl_cgroup(current)` (default-hierarchy cgroup; v2 has single hierarchy).
- `cgroup_inc_frozen_cnt(cgrp)` → `cgrp->freezer.nr_frozen_tasks++`.
- `cgroup_update_frozen(cgrp)`.
- `spin_unlock_irq(&css_set_lock)`.

REQ-8: `cgroup_leave_frozen(always_leave)`:
- `spin_lock_irq(&css_set_lock)`.
- `cgrp = task_dfl_cgroup(current)`.
- If `always_leave || !test_bit(CGRP_FREEZE, &cgrp->flags)`:
  - `cgroup_dec_frozen_cnt(cgrp)` → `cgrp->freezer.nr_frozen_tasks--`; `WARN_ON_ONCE(< 0)`.
  - `cgroup_update_frozen(cgrp)`.
  - `WARN_ON_ONCE(!current->frozen)`.
  - `current->frozen = false`.
- Else if `!(current->jobctl & JOBCTL_TRAP_FREEZE)`:
  - /* Cgroup still freezing — re-arm trap and force re-entry. */
  - `spin_lock(&current->sighand->siglock)`.
  - `current->jobctl |= JOBCTL_TRAP_FREEZE`.
  - `set_thread_flag(TIF_SIGPENDING)`.
  - `spin_unlock(&current->sighand->siglock)`.
- `spin_unlock_irq(&css_set_lock)`.
- Note: `always_leave = false` = "leaving for signal handling"; we keep the counter and re-trap to avoid transient CGRP_FROZEN flicker. `always_leave = true` = "leaving for real (task exit / migration / unfreeze)".

REQ-9: `cgroup_update_frozen(cgrp)`:
- `frozen = test_bit(CGRP_FREEZE, &cgrp->flags) && (cgrp->freezer.nr_frozen_tasks == __cgroup_task_count(cgrp))`.
- If `cgroup_update_frozen_flag(cgrp, frozen)` returned true (flag changed): `cgroup_propagate_frozen(cgrp, frozen)`.

REQ-10: `cgroup_update_frozen_flag(cgrp, frozen)`:
- `lockdep_assert_held(&css_set_lock)`.
- If `test_bit(CGRP_FROZEN, &cgrp->flags) == frozen`: return false (no change).
- If `frozen`: `set_bit(CGRP_FROZEN, &cgrp->flags)`.
- Else: `clear_bit(CGRP_FROZEN, &cgrp->flags)`.
- `cgroup_file_notify(&cgrp->events_file)` — wakes pollers on `cgroup.events`.
- `TRACE_CGROUP_PATH(notify_frozen, cgrp, frozen)`.
- Return true.

REQ-11: `cgroup_propagate_frozen(cgrp, frozen)`:
- `desc = 1` (this cgroup just flipped).
- Walk `cgrp = cgroup_parent(cgrp)` until NULL (root):
  - If `frozen`:
    - `cgrp->freezer.nr_frozen_descendants += desc`.
    - If `!CGRP_FREEZE-set` or `nr_frozen_descendants != nr_descendants`: continue (parent not yet fully-frozen).
  - Else:
    - `cgrp->freezer.nr_frozen_descendants -= desc`.
  - If `cgroup_update_frozen_flag(cgrp, frozen)` returns true: `desc++` (carry the just-flipped count further up).

REQ-12: `cgroup_freezer_migrate_task(task, src, dst)`:
- `lockdep_assert_held(&css_set_lock)`.
- If `task->flags & PF_KTHREAD`: return (kthreads can't be frozen).
- Fast path: if `!CGRP_FREEZE` on src && !CGRP_FREEZE on dst && !task->frozen: return.
- If `task->frozen`: `cgroup_inc_frozen_cnt(dst)`; `cgroup_dec_frozen_cnt(src)` (counter-balance to keep dst's count truthful even if dst is not freezing — bookkeeping invariant).
- `cgroup_update_frozen(dst); cgroup_update_frozen(src)`.
- `cgroup_freeze_task(task, test_bit(CGRP_FREEZE, &dst->flags))` — force task into dst's required state.

REQ-13: Locking model:
- `cgroup_mutex` — held by knob writer (`cgroup_freeze`) and during `cgroup_do_freeze`. Outer lock.
- `css_set_lock` (spinlock, IRQ-safe) — held during all per-task accounting (`cgroup_enter_frozen`, `cgroup_leave_frozen`, `cgroup_freezer_migrate_task`, `cgroup_update_frozen`, `cgroup_propagate_frozen` callers).
- `task->sighand->siglock` — nested inside `css_set_lock` in `cgroup_leave_frozen` for jobctl re-arm.
- `cgrp->freezer.freeze_seq` — write-seqcount around `freeze_start_nsec` / `frozen_nsec` writes; readers (`cgroup.stat.local`-style or future per-stat) use `read_seqcount_begin` / `_retry`.

REQ-14: kthread exclusion:
- `PF_KTHREAD` tasks are skipped in `cgroup_do_freeze` task iteration.
- `cgroup_freezer_migrate_task` also returns early for kthreads.
- Rationale: freezing kthreadd, per-CPU workqueue workers, or RCU kthreads would deadlock the kernel.

REQ-15: `cgroup.freeze` knob (registered in `kernel/cgroup/cgroup.c`):
- v2-only file: appears in `/sys/fs/cgroup/<cg>/cgroup.freeze`.
- Write `"1"` → `cgroup_freeze(cgrp, true)`.
- Write `"0"` → `cgroup_freeze(cgrp, false)`.
- Read: returns `cgrp->freezer.freeze`.
- Sibling `cgroup.events` file exposes `frozen 0|1` per current `CGRP_FROZEN` and is poll/inotify-watchable.

REQ-16: Interaction with `/proc/<pid>/wchan` and signals:
- Frozen tasks are sleeping in `get_signal()` ↔ `cgroup_enter_frozen` ↔ `schedule()`; `/proc/<pid>/wchan` shows the frozen-trap address.
- SIGKILL bypasses freeze: fatal signals are delivered (frozen task wakes, exits via do_exit → `cgroup_leave_frozen(true)` from exit_signals path).
- Non-fatal signals: queued, but the frozen trap re-arms before the handler runs unless the cgroup unfreezes first.
- `ptrace_attach` to a frozen task: trap is honored at next signal-handling boundary; ptracer sees the task in frozen state.

REQ-17: nsec accounting (per `freeze_seq`):
- On freeze: `freeze_start_nsec = ktime_get_ns()`.
- On unfreeze: `frozen_nsec += (now - freeze_start_nsec)`.
- Total frozen time = `frozen_nsec` (+ if currently frozen, `now - freeze_start_nsec`).
- Readers use seqcount retry loop for atomic 64-bit pair on 32-bit kernels.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `freeze_no_op_when_unchanged` | INVARIANT | per-Freezer::freeze: if `cgrp.freeze == arg`: early-return; no state mutation. |
| `e_freeze_is_or_of_self_and_parent` | INVARIANT | per-Freezer::freeze loop: `dsct.e_freeze = dsct.freeze || parent.e_freeze`. |
| `do_freeze_skips_kthreads` | INVARIANT | per-Freezer::do_freeze iter: task with PF_KTHREAD skipped. |
| `freeze_task_under_sighand` | INVARIANT | per-Freezer::freeze_task: `lock_task_sighand` held across jobctl mutation. |
| `nr_frozen_tasks_nonneg` | INVARIANT | per-Freezer::dec_frozen_cnt: counter never negative (WARN guards). |
| `enter_frozen_idempotent` | INVARIANT | per-Freezer::enter_frozen: if `current.frozen`: no inc. |
| `leave_frozen_requires_entered` | INVARIANT | per-Freezer::leave_frozen: WARN if `!current.frozen` on leave-path. |
| `propagate_walks_to_root` | INVARIANT | per-Freezer::propagate_frozen: walks via cgroup_parent until NULL. |
| `seqcount_write_paired` | INVARIANT | per-Freezer::do_freeze: `write_seqcount_begin/_end` paired across both branches. |
| `css_set_lock_held_for_counters` | INVARIANT | per-{enter,leave}_frozen, per-update_frozen, per-propagate_frozen: `css_set_lock` held. |

### Layer 2: TLA+

`kernel/cgroup/freezer.tla`:
- Per-write-cgroup.freeze → per-css_for_each_descendant_pre → per-cgroup_do_freeze (per-task jobctl flip) → per-task `cgroup_enter_frozen` at signal-trap → per-cgroup_update_frozen → per-propagate-up. Reverse: per-write-unfreeze → per-cgroup_leave_frozen (conditional re-trap) → per-cgroup_update_frozen → per-propagate-up.
- Properties:
  - `safety_kthread_never_frozen` — PF_KTHREAD tasks never appear in nr_frozen_tasks.
  - `safety_nr_frozen_tasks_bounded` — `0 <= nr_frozen_tasks <= __cgroup_task_count(cgrp)`.
  - `safety_CGRP_FROZEN_implies_full` — `CGRP_FROZEN` set ⟹ `CGRP_FREEZE` set ∧ `nr_frozen_tasks == cgroup_task_count`.
  - `safety_descendant_counter_balanced` — `nr_frozen_descendants` matches actual count of descendants with `CGRP_FROZEN`.
  - `safety_e_freeze_monotone` — `e_freeze` propagation respects `freeze || parent.e_freeze`.
  - `safety_leave_under_freeze_re_traps` — `cgroup_leave_frozen(false)` with `CGRP_FREEZE` set: jobctl re-armed, counter unchanged.
  - `safety_migrate_preserves_freeze` — `cgroup_freezer_migrate_task`: task in CGRP_FREEZE-dst ends up with `JOBCTL_TRAP_FREEZE` set.
  - `liveness_freeze_settles` — eventually `nr_frozen_tasks == cgroup_task_count` for every cgroup with CGRP_FREEZE (assuming no kthread-only cgroup, no PF_KTHREAD blocking).
  - `liveness_unfreeze_settles` — eventually every task `current.frozen = false` after `cgroup.freeze = 0`.
  - `liveness_root_aggregate` — after freeze: eventually root's `CGRP_FROZEN` (or stopping ancestor's) flips iff every descendant fully frozen.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Freezer::freeze` post: every descendant's e_freeze recomputed; CGRP_FREEZE matches new state on each changed subtree | `Freezer::freeze` |
| `Freezer::do_freeze` post: CGRP_FREEZE flipped, freeze_seq advanced, every non-kthread task got `cgroup_freeze_task` | `Freezer::do_freeze` |
| `Freezer::freeze_task` post: jobctl bit set/cleared exactly, signal/wake delivered | `Freezer::freeze_task` |
| `Freezer::enter_frozen` post: `nr_frozen_tasks` incremented by 1 ∨ idempotent return | `Freezer::enter_frozen` |
| `Freezer::leave_frozen` post: counter dec'd ∧ `current.frozen=false` ∨ jobctl re-armed | `Freezer::leave_frozen` |
| `Freezer::update_frozen` post: CGRP_FROZEN matches `(CGRP_FREEZE ∧ nr_frozen_tasks == task_count)` | `Freezer::update_frozen` |
| `Freezer::propagate_frozen` post: ancestors' `nr_frozen_descendants` adjusted by `desc`; CGRP_FROZEN cascades when full | `Freezer::propagate_frozen` |
| `Freezer::migrate_task` post: counters balanced; task force-aligned to dst's CGRP_FREEZE | `Freezer::migrate_task` |

### Layer 4: Verus/Creusot functional

`Per-cgroup.freeze=1 write → cgroup_freeze(cgrp, true) → css_for_each_descendant_pre (compute e_freeze = self || parent.e_freeze; skip unchanged subtree via css_rightmost_descendant; else cgroup_do_freeze) → cgroup_do_freeze (set CGRP_FREEZE, capture freeze_start_nsec under freeze_seq; iterate non-kthread tasks → cgroup_freeze_task → JOBCTL_TRAP_FREEZE + signal_wake_up); per-task get_signal hits trap → cgroup_enter_frozen (current->frozen=true, nr_frozen_tasks++, cgroup_update_frozen → if CGRP_FREEZE ∧ counts match: CGRP_FROZEN set + cgroup_propagate_frozen (walk parents, bump nr_frozen_descendants, cascade CGRP_FROZEN when nr_frozen_descendants == nr_descendants)); per-cgroup.freeze=0 reverse → cgroup_do_freeze(false, ...): clear CGRP_FREEZE, accumulate frozen_nsec, per-task cgroup_freeze_task(false) → wake_up_process → next get_signal returns from trap → cgroup_leave_frozen(false) → since !CGRP_FREEZE: dec counter, current->frozen=false, propagate clear` semantic equivalence: per-Documentation/admin-guide/cgroup-v2.rst § cgroup.freeze.

### hardening

(Inherits row-1 features from `kernel/cgroup/00-overview.md` § Hardening.)

cgroup freezer reinforcement:

- **Per-PF_KTHREAD exclusion in both `cgroup_do_freeze` iter and `cgroup_freezer_migrate_task`** — defense against per-kthreadd / per-CPU-worker freeze deadlock.
- **Per-`lock_task_sighand` bail-on-dying-task in `cgroup_freeze_task`** — defense against per-task-exit UAF on jobctl write.
- **Per-`css_set_lock` IRQ-safe spin across all counter mutations** — defense against per-IRQ-context migration race on `nr_frozen_tasks`.
- **Per-`cgroup_mutex` outer hold in `cgroup_freeze` and `cgroup_do_freeze`** — defense against per-concurrent-cgroup-create / -destroy during subtree walk.
- **Per-`cgroup_is_dead(dsct)` skip in descendant walk** — defense against per-just-removed-cgroup UAF.
- **Per-`WARN_ON_ONCE(nr_frozen_tasks < 0)`** — defense against per-double-dec / per-orphan-leave invariant break.
- **Per-`WARN_ON_ONCE(!current->frozen)` on leave** — defense against per-stray-leave from un-entered task.
- **Per-`current->frozen` idempotency in `cgroup_enter_frozen`** — defense against per-repeat-trap double-inc on signal re-delivery.
- **Per-conditional re-arm in `cgroup_leave_frozen` when `CGRP_FREEZE` still set** — defense against per-signal-handling transient unfrozen flicker / userspace observing inconsistent state.
- **Per-seqcount-protected `freeze_start_nsec` / `frozen_nsec` pair** — defense against per-32-bit-tearing on time accounting read.
- **Per-`css_rightmost_descendant` skip when `e_freeze` unchanged** — defense against per-O(N) redundant subtree walk on idempotent writes.
- **Per-`signal_wake_up(task, false)` vs `wake_up_process(task)` distinction** — defense against per-freeze-during-uninterruptible-sleep livelock (freeze sets TIF_SIGPENDING so task wakes at next interruptible boundary; unfreeze unconditional-wakes any task stuck in killable freeze trap).
- **Per-`cgroup_file_notify(events_file)` on every CGRP_FROZEN flip and on no-op writes** — defense against per-userspace poll/select hanging forever on stale state.

