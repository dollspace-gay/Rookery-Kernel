# Tier-3: kernel/sched/wait.c — wait_queue (waitqueue + wake-up + bit-wait + completions)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/sched/00-overview.md
upstream-paths:
  - kernel/sched/wait.c
  - kernel/sched/wait_bit.c
  - kernel/sched/completion.c
  - include/linux/wait.h
  - include/linux/wait_bit.h
  - include/linux/completion.h
-->

## Summary

`wait_queue_head_t` is the kernel's primary blocking primitive: a list of `wait_queue_entry_t` waiters, each with task pointer + wake-up callback + flags. Every `wait_event*` macro patterns exhaustively: prepare-to-wait → check-condition → schedule → finish-wait. wait_bit.c specializes for "wait until N-th bit clears in this word" (used by buffer_head, page-flags, etc.). completion.c is the fire-once variant for one-shot signaling (replaces semaphore in many places).

This Tier-3 covers `wait.c` (~465) + `wait_bit.c` (~278) + `wait.h` (~1255) + `completion.c`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct wait_queue_head` | per-condition queue head | `kernel::sched::wait::WaitQueueHead` |
| `struct wait_queue_entry` | per-waiter entry | `WaitQueueEntry` |
| `init_waitqueue_head(wqh)` | per-instance init | `WaitQueueHead::init` |
| `init_waitqueue_entry(wqe, task)` | per-waiter init | `WaitQueueEntry::init` |
| `__wake_up(wqh, mode, nr_excl, key)` | core wake-up | `WaitQueueHead::wake_up` |
| `wake_up(wqh)` / `wake_up_all(wqh)` / `wake_up_interruptible(wqh)` | flavored wake | macros |
| `wake_up_locked(wqh)` | wake under wqh.lock-held | `WaitQueueHead::wake_up_locked` |
| `prepare_to_wait(wqh, wqe, state)` | enqueue + set-task-state | `WaitQueueHead::prepare_to_wait` |
| `prepare_to_wait_exclusive(...)` | enqueue with WQ_FLAG_EXCLUSIVE | `WaitQueueHead::prepare_to_wait_exclusive` |
| `finish_wait(wqh, wqe)` | dequeue + restore TASK_RUNNING | `WaitQueueHead::finish_wait` |
| `default_wake_function(wqe, mode, sync, key)` | per-waiter wakeup-fn | `WaitQueueEntry::default_wake_function` |
| `autoremove_wake_function(...)` | wake-fn that auto-list_del | `WaitQueueEntry::autoremove_wake_function` |
| `wake_up_bit(word, bit)` (wait_bit.c) | per-bit wake | `WaitBit::wake_up` |
| `out_of_line_wait_on_bit(...)` | wait until bit clears | `WaitBit::wait_on_bit` |
| `complete(c)` / `complete_all(c)` (completion.c) | per-completion fire | `Completion::complete` / `_all` |
| `wait_for_completion(c)` / `_timeout(c, jiffies)` / `_interruptible(c)` | wait for fire | `Completion::wait_for` |
| `__add_wait_queue(wqh, wqe)` / `__remove_wait_queue(wqh, wqe)` | low-level list ops | `WaitQueueHead::add_entry` / `_remove_entry` |
| `wake_q_add(q, task)` (sched/core.c) | batched wake-q | `WakeQ::add` |
| `wake_up_q(q)` | flush wake-q outside lock | `WakeQ::flush` |

## Compatibility contract

REQ-1: `wait_queue_head_t`:
- `lock` (raw spinlock).
- `head` (list_head; doubly-linked entries).

REQ-2: `wait_queue_entry_t`:
- `flags` (WQ_FLAG_EXCLUSIVE / WQ_FLAG_WOKEN / WQ_FLAG_BOOKMARK / WQ_FLAG_CUSTOM).
- `private` (typically task pointer; bit_wait uses `wait_bit_key`).
- `func` (wakeup callback).
- `entry` (list_node).

REQ-3: WQ_FLAG_EXCLUSIVE:
- Marks waiter as "wake one of these"; multiple non-exclusive get woken on each event but only one exclusive.
- Used by accept(2) thundering-herd-avoidance + futex-wake.

REQ-4: __wake_up flow:
1. Acquire wqh.lock.
2. Iterate head:
   - Call entry.func(entry, mode, sync, key).
   - If func returns nonzero AND entry has WQ_FLAG_EXCLUSIVE AND nr_excl == 0: stop iteration.
   - If WQ_FLAG_EXCLUSIVE: nr_excl--.
3. Release lock.
4. (default_wake_function does try_to_wake_up + sets WOKEN flag.)

REQ-5: prepare_to_wait flow:
1. Acquire wqh.lock.
2. wqe.flags |= 0; wqe.private = current; wqe.func = autoremove_wake_function (typical).
3. __add_wait_queue(wqh, wqe).
4. set_current_state(state) (TASK_INTERRUPTIBLE / _UNINTERRUPTIBLE).
5. Release lock.

REQ-6: finish_wait:
1. set_current_state(TASK_RUNNING).
2. Acquire wqh.lock.
3. If !list_empty(&wqe.entry): __remove_wait_queue.
4. Release lock.

REQ-7: wait_event(wqh, condition) macro pattern:
- For (;;): if condition: break; prepare_to_wait_exclusive(wqh, &wqe, TASK_UNINTERRUPTIBLE); if condition: break; schedule(); finish_wait(...).
- Variants: wait_event_interruptible (signal-aware), wait_event_timeout (jiffies-bounded), wait_event_killable.

REQ-8: wait_bit:
- Per-(word_addr, bit) wait-key uniqueness via `bit_wait_table[BIT_WAIT_TABLE_SIZE]` hash.
- `wake_up_bit` calls `__wake_up_bit` which dispatches to all waiters with matching key.

REQ-9: completion:
- `struct completion`: { wait: wait_queue_head_t, done: u32 }.
- `complete(c)`: c.done++; wake_up_locked(&c.wait, TASK_NORMAL, 1, NULL).
- `wait_for_completion`: if c.done > 0: c.done--; return. Else wait.
- complete_all: c.done = UINT_MAX/2; wake_up_locked all.

REQ-10: wake_q (batched wake-up):
- `wake_q_add(q, task)`: collect tasks to wake.
- `wake_up_q(q)`: flush outside lock.
- Used by futex/rt_mutex to defer wake-up until lock released (avoids preempt-while-holding-lock).

REQ-11: Wakeup-source tracking (PM):
- /sys/kernel/debug/wakeup_sources tracks per-driver wake-up causes.
- Per-waitqueue may have wakeup_source attached.

REQ-12: Per-CPU runqueue integration:
- try_to_wake_up moves task from sleeping to runnable; selects target CPU via sched_class.select_task_rq.
- Inserts into target rq.

## Acceptance Criteria

- [ ] AC-1: Basic wait_event: thread A blocks via wait_event(wqh, flag); thread B sets flag + wake_up(&wqh); A unblocks.
- [ ] AC-2: WQ_FLAG_EXCLUSIVE: 100 waiters; wake_up wakes 1; remaining 99 still blocked.
- [ ] AC-3: wait_event_interruptible: SIGKILL-delivered task returns -ERESTARTSYS even before condition true.
- [ ] AC-4: wait_event_timeout: task with 100ms timeout; returns 0 on timeout, jiffies-remaining on early-condition.
- [ ] AC-5: wait_on_bit: 32 threads waiting on different bits of same word; wake_up_bit(word, 7) wakes only bit-7 waiters.
- [ ] AC-6: completion: complete() sets done=1; wait_for_completion returns immediately for second arrival; complete_all sets done=infinite.
- [ ] AC-7: 1M wait_event cycles stress: no missed wakes (every wake_up matched by waker→sleeper transition).
- [ ] AC-8: Cross-CPU wakeup: thread A on CPU0 sleeping; thread B on CPU15 wakes; A migrates to CPU15 if rq_balance dictates.

## Architecture

`WaitQueueHead`:

```
struct WaitQueueHead {
  lock: RawSpinLock<()>,
  head: ListHead,                          // wait_queue_entry list
}

struct WaitQueueEntry {
  flags: u32,
  private: *mut TaskStruct,                // typically current task
  func: WaitFn,
  entry: ListNode,
}
```

`WaitQueueHead::wake_up(wqh, mode, nr_excl, key)`:
1. Acquire wqh.lock.
2. wake_q := WAKE_Q_INIT.
3. list_for_each_entry_safe(wqe, tmp, &wqh.head, entry):
   - ret := wqe.func(wqe, mode, 0, key).
   - If ret != 0:
     - If wqe.flags & WQ_FLAG_EXCLUSIVE:
       - nr_excl--.
       - If nr_excl == 0: break.
4. Release lock.
5. wake_up_q(&wake_q) — outside lock.

`Completion::complete`:
1. acquire c.wait.lock.
2. If c.done < UINT_MAX/2: c.done++.
3. __wake_up_locked(&c.wait, TASK_NORMAL, 1).
4. release lock.

`Completion::wait_for(c)`:
1. acquire c.wait.lock.
2. if c.done > 0: c.done--; release; return.
3. While not c.done > 0:
   - prepare_to_wait_exclusive(&c.wait, &wqe, TASK_UNINTERRUPTIBLE).
   - release lock.
   - schedule.
   - acquire lock.
   - if c.done > 0: break.
4. c.done--.
5. finish_wait.
6. release lock.

`WaitBit::wait_on_bit`:
1. wqh := bit_waitqueue(word, bit).
2. wqe with private = wait_bit_key { word, bit }.
3. Loop:
   - prepare_to_wait_exclusive(wqh, wqe, state).
   - if !test_bit(bit, word): break.
   - schedule.
4. finish_wait.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `wqh_list_no_uaf` | UAF | per-wqe stack-local; held under wqh.lock during list-walk; defense against waiter-finish-while-walking. |
| `wake_function_returns_int` | INVARIANT | wqe.func returns int matching default_wake_function semantics. |
| `exclusive_wake_count_no_underflow` | INVARIANT | nr_excl >= 0 throughout iteration. |
| `set_current_state_balanced` | INVARIANT | per-task TASK_RUNNING ↔ TASK_BLOCKED transitions paired. |
| `bit_wait_key_match` | INVARIANT | wake_up_bit only wakes waiters whose key matches (word, bit). |

### Layer 2: TLA+

`kernel/sched/wait_event.tla`:
- Per-waiter state ∈ {Outside, Prepared, Sleeping, Wakened}.
- Transitions per prepare/schedule/wake/finish.
- Properties:
  - `safety_no_lost_wake` — wakeup-after-prepare guarantees Wakened (prepare_to_wait + condition-check pattern).
  - `safety_exclusive_wake_at_most_one` — per-wake-up-event, at most one exclusive waiter wakened.
  - `liveness_pending_eventually_woken` — assuming wakers run, every Sleeping eventually Wakened.

`kernel/sched/completion_state.tla`:
- States per completion: NotDone, Done(n), Saturated.
- Transitions per complete/wait_for.
- Properties:
  - `safety_complete_all_saturates` — complete_all transitions to Saturated.
  - `safety_wait_decrements_done` — wait_for_completion observes and decrements done counter.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `WaitQueueHead::prepare_to_wait` post: wqe in wqh.head; current task in TASK_INTERRUPTIBLE/UNINTERRUPTIBLE | `WaitQueueHead::prepare_to_wait` |
| `WaitQueueHead::wake_up` post: per-wqe func called; nr_excl waited matches wake count | `WaitQueueHead::wake_up` |
| `Completion::complete` post: c.done incremented; one waiter woken | `Completion::complete` |
| `Completion::wait_for` post: c.done decremented; task unblocked | `Completion::wait_for` |
| Per-bit_waitqueue hash entry consistent across wake/wait | `WaitBit::wait_on_bit` |

### Layer 4: Verus/Creusot functional

`prepare_to_wait + condition-check + schedule = wait until condition true OR signal/timeout`: per-thread the wait-event macro produces correct outcome (condition-true / signal / timeout) without lost-wake regardless of waker timing.

## Hardening

(Inherits row-1 features from `kernel/sched/00-overview.md` § Hardening.)

wait-specific reinforcement:

- **Per-wqh.lock raw_spinlock** — defense against IRQ-context wake racing prepare.
- **set_current_state before re-checking condition** — defense against lost-wake (waker observes pre-blocked state).
- **finish_wait restores TASK_RUNNING** — defense against task stuck in TASK_INTERRUPTIBLE after exit-via-signal.
- **WQ_FLAG_BOOKMARK for long-iteration wakers** — defense against wake-up-iteration spending too long under wqh.lock.
- **wake_q deferred wake** — defense against wake-up-while-holding-lock causing PI inversion.
- **Per-bit_waitqueue hash size pow-of-2** — defense against false-key-match between unrelated bit-waiters.
- **autoremove_wake_function default** — defense against double-list-add by stale wqe.
- **Completion done counter capped at UINT_MAX/2** — defense against complete_all + complete overflowing.
- **wait_event re-checks condition under wqh.lock** — defense against TOCTOU between condition-check + wake.
- **Per-waiter exclusive bit cleared on dequeue** — defense against re-enqueue accidentally inheriting exclusive flag.
- **Per-waiter func validated as default OR explicit** — defense against attacker-controlled function-pointer in wqe.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `pollfd`/`epoll_event` copies that feed wait_queue plumbing are bounds-checked; no `wait_queue_entry` slab leakage.
- **PAX_KERNEXEC** — `default_wake_function`, `autoremove_wake_function`, and `__wake_up_common` reside in W^X kernel text; wake callbacks cannot be live-patched.
- **PAX_RANDKSTACK** — per-syscall kstack offset so `wait_event` loops cannot be groomed via predictable stack layout under contended wake races.
- **PAX_REFCOUNT** — `wait_queue_head.wq_lock`-protected counts and `completion.done` use saturating refcount; complete_all + complete overflow oopses.
- **PAX_MEMORY_SANITIZE** — freed `wait_queue_entry` slabs scrubbed so a re-enqueued waiter cannot inherit stale exclusive/flag bits.
- **PAX_UDEREF** — `wait_queue_head` and `wait_queue_entry` dereferenced via kernel mappings only; no implicit user-page reach during wake.
- **PAX_RAP / kCFI** — `wait_queue_func_t` callbacks type-signatured; mismatched `func` signature (e.g., attacker-rewritten `wq_entry->func`) is a hard CFI fault.
- **GRKERNSEC_HIDESYM** — `default_wake_function` address and per-subsystem waitqueue head addresses hidden from `/proc/kallsyms`.
- **GRKERNSEC_DMESG** — wait_queue WARN splats (unbalanced add/remove, dangling waiter) gated to CAP_SYSLOG.
- **wait_queue PAX_REFCOUNT** — `wq_head.head` list operations and `nr_exclusive` counts saturating-refcounted so a UAF on a waitqueue cannot be parlayed into wake-count corruption.
- **wake-callback PAX_RAP** — `__wake_up_common` dispatches through `curr->func(curr, mode, wake_flags, key)` under CFI; attacker-controlled `func` pointer fails the type check before invocation.
- **autoremove discipline** — `autoremove_wake_function` requires the waiter own the entry; `finish_wait` re-validates `entry->task` under `wq_head->lock` so a racing wake cannot leave a dangling entry on the list.
- **Rationale** — wait_queue underlies completion, semaphore, mutex slow paths, poll/epoll, and every IRQ→task wakeup; a corrupted `wq_entry->func` is direct kernel-mode code execution in soft-IRQ context. Grsec/PaX hardening makes the wake path the strongest CFI gate in the synchronization stack.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Sched-core (covered in `kernel/sched/core.md` Tier-3)
- CFS (covered in `kernel/sched/cfs.md` Tier-3)
- RT scheduler (covered in `kernel/sched/rt.md` Tier-3)
- swait (simple-waitqueue for IRQ-handlers; covered in `kernel/sched/swait.md` future Tier-3)
- Implementation code
