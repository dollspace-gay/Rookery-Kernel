# Tier-3: kernel/time/hrtimer.c — high-resolution timer (per-CPU per-base RB-tree + ns-resolution + clock-base discrimination)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/time/00-overview.md
upstream-paths:
  - kernel/time/hrtimer.c
  - include/linux/hrtimer.h
  - include/linux/hrtimer_types.h
  - kernel/time/timekeeping.c (clock-base read)
  - kernel/time/clockevents.c (clock-event-device hook)
-->

## Summary

`hrtimer` is the kernel high-resolution timer primitive (~ns resolution vs jiffies-resolution `timer_list`). Each per-CPU base maintains a red-black-tree of pending timers ordered by absolute expiry-time; the leftmost timer determines the next clock-event-device program. Distinct from timer_list: per-CPU per-clock-base (CLOCK_MONOTONIC, CLOCK_REALTIME, CLOCK_BOOTTIME, CLOCK_TAI), no jiffies-quantization, no cascade. Backs: nanosleep(2), clock_nanosleep(2), itimer (POSIX timer), TCP retransmit, hrtick scheduler tick, kvm-pit emulation, perf_event sample-period, eventpoll-timeout.

This Tier-3 covers `kernel/time/hrtimer.c` (~2503 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hrtimer` | per-timer | `kernel::time::hrtimer::HrTimer` |
| `struct hrtimer_clock_base` | per-CPU per-clock-base | `HrtimerClockBase` |
| `struct hrtimer_cpu_base` | per-CPU rollup of all clock-bases | `HrtimerCpuBase` |
| `enum hrtimer_mode` | abs/rel + soft/hard + pinned | `HrtimerMode` |
| `enum hrtimer_restart` | callback return: NORESTART / RESTART | `HrtimerRestart` |
| `hrtimer_init(timer, clk, mode)` | init | `HrTimer::init` |
| `hrtimer_setup(timer, fn, clk, mode)` | init + bind callback | `HrTimer::setup` |
| `hrtimer_start(timer, time, mode)` / `_start_range_ns(timer, ...)` | enqueue | `HrTimer::start` / `_start_range` |
| `hrtimer_cancel(timer)` / `hrtimer_try_to_cancel(timer)` | dequeue | `HrTimer::cancel` / `_try_cancel` |
| `hrtimer_forward(timer, now, interval)` / `_forward_now(...)` | re-arm relative-to-now | `HrTimer::forward` |
| `hrtimer_active(timer)` | query | `HrTimer::active` |
| `__hrtimer_run_queues(cpu_base, now, flags)` | per-CPU per-base run | `HrtimerCpuBase::run_queues` |
| `enqueue_hrtimer(timer, base, mode)` | per-tree insert | `HrtimerClockBase::enqueue` |
| `__remove_hrtimer(...)` | per-tree remove | `HrtimerClockBase::remove` |
| `hrtimer_reprogram(timer, reprogram)` | re-program clock-event-device | `HrTimer::reprogram` |
| `hrtimer_interrupt(dev)` | clockevent-irq entry | `HrtimerCpuBase::interrupt` |
| `hrtimer_run_softirq` | softirq dispatch (HRTIMER_SOFTIRQ) | `HrtimerCpuBase::run_softirq` |
| `schedule_hrtimeout_range(...)` | task-sleep for nsec | `HrTimer::schedule_hrtimeout` |
| `hrtimer_init_sleeper(...)` | per-task wake-on-expiry timer | `HrTimer::init_sleeper` |

## Compatibility contract

REQ-1: Per-CPU `hrtimer_cpu_base`:
- Per-clock-base array (typically 8: 4 hard + 4 soft):
  - HRTIMER_BASE_MONOTONIC.
  - HRTIMER_BASE_REALTIME.
  - HRTIMER_BASE_BOOTTIME.
  - HRTIMER_BASE_TAI.
  - + soft variants (HRTIMER_BASE_*_SOFT).
- Per-base RB-tree of pending timers.
- Per-base running-timer pointer (currently-executing).
- Per-CPU-base nr_active, hres_active, in_hrtirq, expires_next.

REQ-2: Per-timer `hrtimer`:
- `node` (timerqueue_node embedded; combines RB-tree + linked-list-leftmost).
- `_softexpires` (range-end if hrtimer_start_range_ns).
- `function` (callback returning HRTIMER_NORESTART / _RESTART).
- `base` (back-ref to per-CPU clock-base).
- `state` (HRTIMER_STATE_INACTIVE / _ENQUEUED).
- `is_rel` (relative vs absolute time).
- `is_soft` (HRTIMER_SOFTIRQ context).
- `is_hard` (HARDIRQ context, default).

REQ-3: Per-CPU clock-event-device integration:
- Programmable via `clockevents_program_event`.
- One-shot or periodic; KVM/x86 typically uses TSC-deadline or LAPIC-timer.
- Programmed for next-leftmost-timer's expiry-ns.

REQ-4: hrtimer_start flow:
1. lock_hrtimer_base(timer); base := timer.base.
2. if state == ENQUEUED: __remove_hrtimer.
3. timer.expires = time (or hrtimer_get_softexpires for range).
4. enqueue_hrtimer:
   - timerqueue_add(&base.active, &timer.node) — RB-tree insert ordered by softexpires.
5. timer.state = ENQUEUED.
6. unlock.
7. If newly-leftmost: hrtimer_reprogram(timer) — program clock-event-device for new earliest.

REQ-5: hrtimer_interrupt (IRQ) flow:
1. Read clock-base nows (per-base latest time).
2. Per-base RB-tree:
   - While leftmost.expires ≤ now:
     - Remove leftmost.
     - If timer.is_soft: enqueue to softirq queue.
     - Else: call timer.function(timer) inline.
     - On RESTART: timer.expires = (callback-set new time); re-enqueue.
3. Compute next-leftmost across all bases.
4. Reprogram clock-event-device for next-expiry.

REQ-6: Soft vs hard:
- HRTIMER_MODE_HARD: callback runs from clockevent-IRQ (hardirq context).
- HRTIMER_MODE_SOFT: callback runs from HRTIMER_SOFTIRQ.
- Default: hard for short-callback (BBR retransmit, sched-rt-runtime); soft for longer (QoS).

REQ-7: Pinned vs non-pinned:
- HRTIMER_MODE_PINNED: timer fires on its add-time CPU; no migration.
- Non-pinned: NOHZ may migrate to wakeful CPU.

REQ-8: Range-ns (hrtimer_start_range_ns):
- timer fires anytime in [expires .. expires+slack-ns].
- Allows kernel to coalesce with other timers in window; reduces wake-ups.

REQ-9: hrtimer_cancel semantics:
- Remove from RB-tree.
- If currently-executing on some CPU: spin-wait for completion.
- After return: state == INACTIVE.

REQ-10: hrtimer_forward(timer, now, interval):
- Used in periodic-timer pattern: callback computes next-expires = old_expires + N*interval where N is smallest making next > now.
- Returns N (number of intervals skipped).

REQ-11: schedule_hrtimeout_range(expires, slack, mode):
- Per-task: arm hrtimer; sleep TASK_INTERRUPTIBLE; on expiry/signal/wakeup: cancel; return remaining.
- Used by: nanosleep, clock_nanosleep, futex-with-timeout, eventpoll, etc.

REQ-12: Live-migration / suspend-resume:
- On suspend: per-CPU base.expires_next saved.
- On resume: clock-base read fresh; reprogram clock-events.

## Acceptance Criteria

- [ ] AC-1: nanosleep(100ns): sleeps ~100ns (within clock-event-device resolution); resolves better than jiffies.
- [ ] AC-2: hrtimer_start CLOCK_MONOTONIC + 1ms expiry: callback runs ≤ 1.1ms after start.
- [ ] AC-3: hrtimer_cancel mid-execution: waits for running callback; returns 1 if was active.
- [ ] AC-4: HRTIMER_MODE_PINNED: timer added on CPU0 fires on CPU0 even if CPU0 idle.
- [ ] AC-5: hrtimer_forward: periodic 100ms timer; if callback is delayed by 50ms, forward returns 1; if delayed by 250ms, returns 2.
- [ ] AC-6: schedule_hrtimeout: timer-armed sleep; signal-interrupt returns -ERESTARTSYS; timeout returns 0.
- [ ] AC-7: 1M hrtimer stress: distributed across 32 CPUs at random ns-intervals; all fire within ±50us.
- [ ] AC-8: clock_nanosleep CLOCK_REALTIME absolute: timer fires when wallclock reaches target (across host-clock-step).
- [ ] AC-9: KVM PIT timer emulation: hrtimer-driven IRQ0 advancement matches host-jiffies precision.

## Architecture

`HrTimer` typed wrapper:

```
struct HrTimer<F: FnMut(&mut HrTimer<F>) -> HrtimerRestart + 'static> {
  inner: HrtimerC,
  callback: F,
}

struct HrtimerC {
  node: TimerqueueNode,
  _softexpires: ktime_t,
  function: HrtimerFn,
  base: KArc<HrtimerClockBase>,
  state: u8,
  is_rel: bool,
  is_soft: bool,
  is_hard: bool,
}

struct HrtimerClockBase {
  cpu_base: KArc<HrtimerCpuBase>,
  index: u8,
  clockid: ClockId,
  active: TimerqueueHead,                    // RB-tree + leftmost-cache
  get_time: GetTimeFn,                       // ktime_get_*
  offset: ktime_t,                            // wallclock offset for REALTIME / TAI
  ...
}

struct HrtimerCpuBase {
  lock: RawSpinLock<()>,
  cpu: u32,
  active_bases: u32,
  clock_was_set_seq: u32,
  hres_active: bool,
  in_hrtirq: bool,
  hang_detected: bool,
  expires_next: ktime_t,
  next_timer: Option<KArc<HrtimerC>>,
  softirq_expires_next: ktime_t,
  softirq_next_timer: Option<KArc<HrtimerC>>,
  clock_base: [HrtimerClockBase; HRTIMER_MAX_CLOCK_BASES],
}
```

`HrTimer::start(timer, time, mode)`:
1. base := lock_hrtimer_base(timer).
2. If state == ENQUEUED: __remove_hrtimer.
3. timer.expires = time (or _softexpires if range).
4. enqueue_hrtimer(timer, base, mode):
   - timerqueue_add(&base.active, &timer.node).
   - state = ENQUEUED.
5. unlock(base).
6. If newly-leftmost: hrtimer_reprogram(timer).

`HrtimerCpuBase::interrupt(dev)`:
1. Lock cpu_base.
2. now := per-base get_time.
3. For each active base:
   - leftmost := timerqueue_getnext(&base.active).
   - While leftmost && leftmost.expires ≤ now:
     - __remove_hrtimer(leftmost).
     - If leftmost.is_soft: enqueue to softirq.
     - Else: state := EXECUTING; unlock; ret := leftmost.function(leftmost); lock.
     - If ret == RESTART: leftmost.expires set by callback; enqueue_hrtimer.
4. Compute next-expires across all bases.
5. cpu_base.expires_next = next.
6. unlock.
7. clockevents_program_event(dev, next-expires).

`HrTimer::cancel(timer)`:
1. Loop:
   - lock_hrtimer_base.
   - If state == INACTIVE: unlock; return 0.
   - If state == ENQUEUED: __remove_hrtimer; unlock; return 1.
   - If state == EXECUTING: unlock; cpu_relax; spin-wait for callback to complete; retry.

`schedule_hrtimeout_range(expires, slack, mode)`:
1. hrtimer_init_sleeper_on_stack(t, current).
2. hrtimer_set_expires_range_ns(&t.timer, *expires, slack).
3. hrtimer_sleeper_start_expires(&t, mode).
4. set_current_state(TASK_INTERRUPTIBLE).
5. schedule().
6. hrtimer_cancel(&t.timer).
7. Return -EINTR / 0 / -ETIMEDOUT.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rb_tree_invariants` | INVARIANT | per-base.active RB-tree always sorted by expires; leftmost-cache up-to-date. |
| `state_transitions` | INVARIANT | timer.state ∈ {INACTIVE, ENQUEUED, EXECUTING}; transitions follow state-machine. |
| `cpu_base_per_cpu_unique` | INVARIANT | per-CPU base distinct; defense against cross-CPU sharing. |
| `cancel_no_uaf` | UAF | hrtimer_cancel waits for executing; defense against module-unload UAF. |
| `expires_next_consistent` | INVARIANT | cpu_base.expires_next == min over all base leftmost expires. |

### Layer 2: TLA+

`kernel/time/hrtimer_state.tla`:
- Per-timer state ∈ {Inactive, Enqueued, Executing}.
- Transitions per start/cancel/expire/restart.
- Properties:
  - `safety_at_most_one_base` — per-Enqueued timer in exactly one base.active tree.
  - `safety_executing_then_dequeued` — once Executing, not Enqueued (has been removed from tree).
  - `liveness_enqueued_eventually_executes` — assuming clock advances, every Enqueued eventually Executing.

`kernel/time/clockevents_program.tla`:
- Per-CPU clock-event-device program-state ∈ {Idle, Programmed(expires), Triggered}.
- Properties:
  - `safety_program_matches_leftmost` — Programmed.expires == leftmost-timer expires across all bases.
  - `safety_no_lost_irq` — Triggered → re-program after handling.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `HrTimer::start` post: timer in base.active tree; state == ENQUEUED | `HrTimer::start` |
| `HrTimer::cancel` post: timer not in tree; state == INACTIVE | `HrTimer::cancel` |
| `HrtimerCpuBase::interrupt` post: all expired timers fired; expires_next reprogrammed | `HrtimerCpuBase::interrupt` |
| `HrTimer::forward` post: returns N where new expires = old + N*interval, smallest > now | `HrTimer::forward` |
| Per-base lock held during enqueue/dequeue | invariants on all ops |

### Layer 4: Verus/Creusot functional

`hrtimer_start(t, expires=N) → callback fires at clock_get(clk) ≥ N` semantic equivalence: per-timer the callback is invoked when clock-base time first reaches expires (within clockevent-resolution).

## Hardening

(Inherits row-1 features from `kernel/time/00-overview.md` § Hardening.)

hrtimer-specific reinforcement:

- **Per-base raw_spinlock_t** — defense against IRQ-context start racing tree mutation.
- **hrtimer_cancel wait-for-executing** — defense against module-unload UAF.
- **HRTIMER_MODE_PINNED honored** — defense against per-CPU timer migrating away.
- **RB-tree leftmost-cache** updated atomically — defense against stale leftmost causing missed-expiry.
- **hang-detection per-CPU** — detects clockevent-irq hang (no progress for ~5s) + recovery via reset.
- **Per-clock-base offset cache** — defense against torn realtime-offset read.
- **schedule_hrtimeout uses on-stack sleeper** — defense against per-task sleeper-state outliving stack frame.
- **hrtimer_forward returns >= 1 always** — defense against zero-return causing infinite-restart on RESTART path.
- **clockevents_program_event delta clamped** — defense against negative-delta or far-future-delta causing clockevent driver crash.
- **Per-timer is_soft/is_hard set at init** — defense against context-mode confusion in dispatch.
- **hrtimer_run_queues bounded by lock-contention** — defense against single-base hogging IRQ time.
- **Suspend/resume per-base.expires_next saved/restored** — defense against post-resume infinite-now-next causing miss-fire.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `itimerspec`/`timespec` copies for `timer_settime`/`nanosleep` bounds-checked; no `hrtimer` slab leakage to userspace.
- **PAX_KERNEXEC** — `hrtimer_interrupt`, `__hrtimer_run_queues`, and `hrtimer_setup` reside in W^X kernel text; expiry dispatch cannot be live-patched.
- **PAX_RANDKSTACK** — per-syscall kstack offset so `nanosleep`/`clock_nanosleep` waiters cannot be groomed via predictable kstack layout.
- **PAX_REFCOUNT** — `hrtimer_cpu_base->active` count, per-clock-base `nr_active`, and `hrtimer.state` transitions saturating-refcounted.
- **PAX_MEMORY_SANITIZE** — freed `hrtimer` slabs scrubbed so a reused timer slot cannot inherit stale `function`/`expires`.
- **PAX_UDEREF** — hrtimer callbacks dereference `hrtimer` and `hrtimer_cpu_base` via per-CPU kernel mappings only; no implicit user-page reach.
- **PAX_RAP / kCFI** — `hrtimer.function` (enum `hrtimer_restart (*)(struct hrtimer *)`) type-signatured; attacker-rewritten function pointer is a hard CFI fault before dispatch.
- **GRKERNSEC_HIDESYM** — `hrtimer_bases`, per-CPU `hrtimer_cpu_base`, and tick-device addresses hidden from `/proc/kallsyms`.
- **GRKERNSEC_DMESG** — hrtimer WARN splats (negative delta, callback ran past deadline) gated to CAP_SYSLOG.
- **hrtimer_setup PAX_RAP** — `hrtimer_setup` records `function` only via the type-checked initializer; later mutation of `timer->function` outside the initializer trips CFI on dispatch.
- **hrtimer_callback signature** — every callback returns `enum hrtimer_restart` and takes `struct hrtimer *`; the CFI hash bound at compile time is what `__hrtimer_run_queues` checks before calling.
- **Expiry-handler discipline** — callbacks must not sleep, must complete bounded work, and must re-arm or return `HRTIMER_NORESTART` cleanly; `WARN_ON(in_atomic())` (and lockdep) catches callbacks that violate the contract, denying "expiry-as-soft-IRQ-pivot" tricks.
- **Rationale** — hrtimer drives nanosleep, posix-timers, sched_clock-tick, RCU expedited paths, and netdev watchdogs; corrupted `timer->function` is direct kernel-mode code-execution in soft-IRQ context. Grsec/PaX hardening keeps every expiry through an RX-only, CFI-checked, refcounted dispatch.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- timer-wheel `timer_list` (covered in `kernel/time/timer.md` Tier-3)
- POSIX timers (covered in `kernel/time/posix-timers.md` future Tier-3)
- itimer (covered in `kernel/time/itimer.md` future Tier-3)
- alarmtimer (covered in `kernel/time/alarmtimer.md` future Tier-3)
- timekeeping core (covered in `kernel/time/timekeeping.md` future Tier-3)
- clockevents driver layer (covered in `kernel/time/clockevents.md` future Tier-3)
- Implementation code
