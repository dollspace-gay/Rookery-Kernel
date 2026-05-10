# Tier-3: kernel/time/timer.c — timer-wheel (cascading-buckets timer-list expiration + per-CPU run-queue + deferrable timers)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/time/00-overview.md
upstream-paths:
  - kernel/time/timer.c
  - include/linux/timer.h
  - include/linux/timer_types.h
  - kernel/time/timer_migration.c
-->

## Summary

The classic timer-list (`struct timer_list`) primitive backs the most common kernel deferred-call patterns where high-precision is not needed (~jiffies-resolution): per-socket retransmit timers, per-device probe timeout, per-process kill-on-timeout, BH-context one-shot callbacks. Implementation: per-CPU cascading timer-wheel with 8 levels (each level's bucket is 8x coarser than previous; total range ~6 years at HZ=1000); timer expiration runs from per-CPU TIMER_SOFTIRQ. NOHZ-aware: per-CPU detects pure-deferrable wheels at idle and skips waking host.

This Tier-3 covers `kernel/time/timer.c` (~2580 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct timer_list` | per-timer | `kernel::time::timer::Timer` |
| `struct timer_base` | per-CPU per-bucket-arena | `TimerBase` |
| `struct hlist_head wheel[WHEEL_SIZE]` | per-CPU 512-bucket wheel | `TimerBase::wheel` |
| `init_timer(timer)` / `timer_setup(timer, fn, flags)` | init | `Timer::init` / `_setup` |
| `add_timer(timer)` / `add_timer_on(timer, cpu)` | enqueue | `Timer::add` / `_add_on` |
| `mod_timer(timer, expires)` / `mod_timer_pending(timer, expires)` | reschedule | `Timer::mod` |
| `del_timer(timer)` / `del_timer_sync(timer)` | cancel | `Timer::del` / `_del_sync` |
| `timer_pending(timer)` | query | `Timer::pending` |
| `run_local_timers()` | softirq entry | `TimerBase::run_local_timers` |
| `__run_timers(base)` | per-base run | `TimerBase::run_timers` |
| `expire_timers(base, head)` | per-bucket fire | `TimerBase::expire_bucket` |
| `cascade(base, level)` | per-level cascade-down to next-finer | `TimerBase::cascade` |
| `next_pending_bucket(base)` | bucket lookup for ETA | `TimerBase::next_pending_bucket` |
| `__get_next_timer_interrupt(basej, basem)` | per-CPU next-timer (NOHZ) | `TimerBase::next_timer_interrupt` |
| `enqueue_timer(base, timer, idx, bucket_expiry)` | per-bucket insert | `TimerBase::enqueue` |
| `timer_migrate(timer, new_cpu)` | per-CPU migrate | `Timer::migrate` |
| `lock_timer_base(timer, &flags)` | per-base spinlock | `Timer::lock_base` |
| `do_init_timer(timer, fn, flags, name, key)` (debug) | lockdep-aware init | `Timer::do_init` |
| `process_timeout(...)` (schedule_timeout) | schedule integration | `Timer::process_timeout` |
| `tmigr_*` (timer_migration.c) | NOHZ timer migration to wakeful CPU | `TimerMigr::*` |

## Compatibility contract

REQ-1: Per-CPU `timer_base`:
- 9 levels × 64 buckets = 576 buckets; each level's bucket period 8x previous.
- Level 0: bucket period = 1 jiffy (64 jiffies range).
- Level 1: 8 jiffies × 64 = 512 jiffies range.
- ...
- Level 8: 8^8 jiffies × 64 = 32-bit jiffy range.
- Per-base lock (raw_spinlock).

REQ-2: Per-timer `timer_list`:
- `entry` (hlist_node).
- `expires` (jiffies; absolute).
- `function` (callback).
- `flags` (TIMER_DEFERRABLE / TIMER_PINNED / TIMER_IRQSAFE / TIMER_SOFTIRQ_LATCH).
- `base` (back-ref to per-CPU base).

REQ-3: TIMER_DEFERRABLE flag:
- timer expiration NOT urgent; per-base second-arena (`bases[BASE_DEF]`).
- NOHZ-aware: idle CPU may skip waking just-for-deferrable.

REQ-4: TIMER_PINNED flag:
- timer must run on its add_timer-target CPU (no NOHZ migration).
- Used for per-CPU monitoring / per-CPU clock-event-managed.

REQ-5: TIMER_IRQSAFE flag:
- callback may run from IRQ context (no preempt-disable / no sleep).
- Default: callback in TIMER_SOFTIRQ context.

REQ-6: Per-bucket-list (hlist):
- Insertion: `hlist_add_head` (prepend; ordering within bucket not strict).
- Removal: `hlist_del`.

REQ-7: Per-tick scheduler hook (`run_local_timers`):
1. Mark TIMER_SOFTIRQ pending on per-CPU.
2. Softirq runs `__run_timers(base)`.

REQ-8: `__run_timers` flow:
1. base.timer_jiffies advances to current jiffies.
2. For each tick from base.timer_jiffies to current:
   - idx = base.timer_jiffies & WHEEL_BUCKETS_MASK.
   - If level-0-bucket non-empty:
     - base.timer_jiffies++.
     - Move list-head to expired_list.
     - cascade level-1 if needed (when level-0 wheel-wraps).
3. Per-expired-timer: fire callback.
   - If TIMER_IRQSAFE: hardirq-context (rare).
   - Else: softirq-context.

REQ-9: Cascade:
- When level-N wheel wraps (every 64 ticks at level-N's period):
  - cascade(base, N+1): drain head of level-N+1 bucket; per-timer-in-bucket: re-insert at appropriate level-N bucket per its expires.

REQ-10: NOHZ integration (`__get_next_timer_interrupt`):
- Per-CPU finds nearest pending timer across all 9 levels.
- Returns next ETA jiffies for tick-stop calculation.
- NOHZ_FULL: per-CPU may have no timers; defer to other CPU (timer_migration).

REQ-11: del_timer_sync:
- Cancels timer; if timer is currently executing, waits for callback to return.
- Required for module unload + kfree-of-callback-data.

REQ-12: Timer-migration (timer_migration.c):
- For NOHZ_FULL: idle CPUs offload non-pinned non-IRQSAFE timers to wakeful CPU's base.
- Per-tmigr-group hierarchical balancing.

REQ-13: schedule_timeout(timeout):
- Queue per-task timer for `timeout` jiffies; schedule(); on wakeup: timeout reached or signal.
- Common pattern for sleeping with bounded duration.

## Acceptance Criteria

- [ ] AC-1: add_timer + 100ms expiry: callback runs ~100ms later (within 1-jiffy tolerance).
- [ ] AC-2: del_timer_sync: cancel pending timer; cancel currently-executing timer waits for callback completion.
- [ ] AC-3: mod_timer: reschedule pending timer; old expiry-bucket cleared; new bucket gets timer.
- [ ] AC-4: TIMER_DEFERRABLE: idle 100ms test; deferrable-only base does not wake CPU; non-deferrable bases do.
- [ ] AC-5: TIMER_PINNED: timer added on CPU0; never migrates to CPU1 even on NOHZ idle.
- [ ] AC-6: NOHZ_FULL CPU: pinned timer fires on its CPU; non-pinned migrates to wakeful CPU.
- [ ] AC-7: 1M timers stress: distributed across 32 CPUs; all fire within ±1 jiffy of expected.
- [ ] AC-8: Cascade test: timer at expires=1000jiffies (level-3 bucket); after 64 ticks wraps level-1; cascades to level-0 correctly.

## Architecture

`Timer<F>` typed wrapper (F = callback closure):

```
struct Timer<F: FnMut() + 'static> {
  inner: TimerList,                        // C-side struct timer_list
  func: F,
}

struct TimerList {
  entry: HlistNode,
  expires: u64,                             // absolute jiffies
  function: TimerFn,
  flags: u32,
}

struct TimerBase {
  cpu: u32,
  next_expiry: u64,                         // cached nearest-pending across all wheels
  timer_jiffies: u64,                       // last-processed
  pending_map: u64,                         // per-bucket pending bitmap
  vectors: [HlistHead; WHEEL_SIZE],         // 576 buckets
  is_idle: bool,
  must_forward_clk: bool,
  lock: RawSpinLock<()>,
  ...
}
```

`Timer::add(timer)`:
1. base := per-CPU timer_base (or migrated target).
2. lock(base).
3. expires := timer.expires; idx := calc_wheel_index(expires, base.timer_jiffies).
4. enqueue_timer(base, timer, idx, bucket_expiry).
5. Update base.next_expiry if needed.
6. unlock(base).

`Timer::mod(timer, expires)`:
1. lock_timer_base(timer); base := timer.base.
2. If timer pending: hlist_del(&timer.entry); update bucket pending bit.
3. timer.expires = expires.
4. Re-enqueue at new bucket.
5. unlock(base).

`Timer::del_sync(timer)`:
1. Loop:
   - lock_timer_base.
   - If !timer pending: unlock; check executing-state.
   - Else: hlist_del; unlock; return.
2. If currently executing on some CPU: spin-wait for callback to return.

`TimerBase::run_timers` (softirq):
1. lock(base).
2. now := jiffies.
3. While base.timer_jiffies < now:
   - idx := base.timer_jiffies & WHEEL_BUCKETS_MASK.
   - hlist_move_list(&base.vectors[idx], &expired_list).
   - base.timer_jiffies++.
   - If wheel-level-N wraps: cascade(base, N+1).
4. unlock(base).
5. Per-timer in expired_list:
   - timer.function() (callback in softirq ctx unless IRQSAFE).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `wheel_idx_no_oob` | OOB | per-bucket index ∈ [0, 575]; calc_wheel_index returns bounded value. |
| `cascade_level_bounded` | INVARIANT | level ∈ [0, 8]; cascade never recurses beyond. |
| `timer_base_per_cpu_unique` | INVARIANT | per-CPU base distinct; defense against cross-CPU base sharing. |
| `expire_bucket_no_uaf` | UAF | per-timer hlist_del + callback under timer-base lock; cancel waits for executing. |

### Layer 2: TLA+

`kernel/time/timer_wheel.tla`:
- Per-timer state ∈ {Free, Queued, Executing, Done}.
- Transitions per add/mod/del/run.
- Properties:
  - `safety_at_most_one_bucket` — per-Queued timer in exactly one bucket at a time.
  - `safety_no_executing_after_del_sync` — del_sync return implies state ∈ {Free, Done}.
  - `liveness_queued_eventually_executes` — assuming jiffies advance, every Queued eventually Executing.

`kernel/time/timer_cascade.tla`:
- Per-level wheel-wrap-cascade preserves per-timer absolute-expiry semantics.
- Properties:
  - `safety_cascade_preserves_expiry` — post-cascade timer still fires at original expires.
  - `safety_cascade_terminates` — cascade chain bounded by level-count.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Timer::add` post: timer in correct bucket per expires; pending bit set | `Timer::add` |
| `Timer::mod` post: old bucket entry removed; new bucket entry added | `Timer::mod` |
| `Timer::del_sync` post: timer not pending and not executing | `Timer::del_sync` |
| `TimerBase::run_timers` post: base.timer_jiffies advanced to current jiffies; all expired-list timers fired | `TimerBase::run_timers` |
| Per-base lock held during enqueue/dequeue | invariants on all ops |

### Layer 4: Verus/Creusot functional

`add_timer(t, expires=N) → callback fires at jiffies ≥ N` semantic equivalence: per-timer the callback is invoked at first softirq run after jiffies reaches expires (within bounded error of softirq-latency).

## Hardening

(Inherits row-1 features from `kernel/time/00-overview.md` § Hardening.)

timer-specific reinforcement:

- **Per-base raw_spinlock_t** — defense against IRQ-context add_timer racing softirq-context run.
- **del_timer_sync wait-for-executing** — defense against module-unload UAF on callback-data.
- **TIMER_PINNED honored across NOHZ migration** — defense against per-CPU monitoring-timer migrating away.
- **WHEEL_BUCKETS bound checked** — defense against calc_wheel_index OOB on wraparound expires.
- **Per-cascade-level bound** — defense against infinite cascade on pathological expires.
- **Per-base must_forward_clk** discipline — defense against jiffies-skew across NOHZ ↔ active.
- **TIMER_DEFERRABLE arena separate** — defense against deferrable wake interfering with NOHZ idle.
- **timer.flags atomic** at lock_base re-check — defense against TOCTOU on flags-mutation.
- **Per-task `process_timeout` linked-list** — defense against torn-update during signal-wake.
- **timer_migrate per-tmigr-group** — defense against incorrect timer migration target.
- **schedule_timeout return-value** distinguishes timeout vs signal — defense against caller misinterpretation.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- hrtimer (high-resolution timer; covered in `kernel/time/hrtimer.md` future Tier-3)
- posix-timers (covered in `kernel/time/posix-timers.md` future Tier-3)
- itimer (covered in `kernel/time/itimer.md` future Tier-3)
- alarmtimer (covered in `kernel/time/alarmtimer.md` future Tier-3)
- timer_migration.c full details (covered separately if needed)
- Implementation code
