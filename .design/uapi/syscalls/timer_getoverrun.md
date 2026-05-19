# Tier-5 syscall: timer_getoverrun(2) — syscall 225

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/posix-timers.c (sys_timer_getoverrun, common_timer_get_overrun)
  - kernel/signal.c (signal delivery captures it_overrun_last into siginfo->si_overrun)
  - include/uapi/asm-generic/siginfo.h (siginfo_t::si_overrun)
  - arch/*/include/generated/uapi/asm/unistd_64.h (225)
  - Documentation/core-api/timekeeping.rst, man timer_getoverrun(2)
-->

## Summary

`timer_getoverrun(2)` returns the **overrun count** for the most recently delivered expiration of a POSIX timer — i.e., the number of additional expirations that occurred between when this timer's signal was queued and when the signal handler was actually entered (signal merging in real-time-signal queues, or process-suspended periods for periodic timers). The semantic is: if a 10ms-period timer fires while its previous signal is still pending delivery, every additional fire increments the overrun count. The count is **per-delivery**, not cumulative across deliveries; each call to `timer_getoverrun` (or each siginfo-carried `si_overrun` field on signal-handler entry) reflects the most recent delivery only. Critical for: real-time loop catch-up logic (after a missed deadline, do N catch-up iterations), telemetry sample-count accounting, scheduling-anomaly detection, RT-application "did the kernel keep up?" introspection.

This Tier-5 covers the userspace ABI of syscall 225. Per-`hrtimer` overrun bookkeeping and per-`alarmtimer` overrun on suspend are owned by `kernel/time/hrtimer.md` and `kernel/time/alarmtimer.md` (Tier-3, planned).

## Signature

```c
int timer_getoverrun(timer_t timerid);
```

Rust ABI shim:

```rust
pub fn sys_timer_getoverrun(timerid: TimerId) -> isize;
```

Syscall number: **225**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `timerid` | `timer_t` | IN | timer handle |

## Return

- **Success**: non-negative overrun count (`>= 0`) — number of additional firings since the last signal was queued.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `timerid` does not name a live timer in current process |

## ABI surface

Return value is an `int`; on Linux, capped at `DELAYTIMER_MAX` (= `INT_MAX`).

Companion `siginfo_t::si_overrun` is populated on signal delivery:

```c
typedef struct {
    int      si_signo;
    int      si_errno;
    int      si_code;          /* SI_TIMER for POSIX-timer signals */
    int      si_overrun;       /* timer overrun on delivery */
    int      si_timerid;       /* timer_t */
    /* ... */
} siginfo_t;
```

So a signal handler can avoid `timer_getoverrun(2)` entirely by reading `siginfo.si_overrun` — equivalent semantics, but no extra syscall.

## Compatibility contract

REQ-1: `timerid` lookup:
- `posix_timer_get_locked(current.signal, timerid)` MUST find a live `k_itimer` whose `it_signal == current.signal`; else `-EINVAL`.

REQ-2: Overrun computation:
- For hrtimer-backed periodic timers: kernel maintains `k.it_overrun_last`, set when the prior signal was queued via `hrtimer_forward(now, k.it_interval)` (forward returns the number of periods advanced minus 1).
- For one-shot timers: overrun always `0`.
- For disarmed timers: overrun reflects last delivery (if any) or `0`.

REQ-3: Capped at INT_MAX:
- `k.it_overrun_last` is `s64`; on extreme suspend/forwarding, capped to `INT_MAX` to fit `int` return.

REQ-4: No mutation on query:
- `timer_getoverrun` does NOT reset overrun count; subsequent calls return the same value until the next signal delivery updates it.

REQ-5: Signal-delivery interaction:
- When the SIGEV_SIGNAL is delivered, `siginfo.si_overrun = k.it_overrun_last`.
- After delivery (signal-handler entered), the next periodic fire's overrun starts accumulating from zero.

REQ-6: SIGEV_NONE:
- Timers with `SIGEV_NONE`: still maintain overrun count internally (next-period-forwarding); query returns currently-accumulated count.
- No signal-handler ever runs, so the count grows monotonically up to INT_MAX and stays there.

REQ-7: Concurrency:
- Lookup under siglock; overrun read atomically (single `READ_ONCE`).

REQ-8: No capability requirement:
- Caller queries timers it owns; no capability check.

REQ-9: Lifecycle:
- A deleted timer (`timer_delete`) returns `-EINVAL` even if a signal was already queued with that timer's `si_timerid`.

REQ-10: Per-clock semantics:
- ALARM-class clocks: missed-during-suspend fires increment overrun on resume.
- CPU-time clocks: overrun rare in practice (CPU time is naturally rate-limited).

## Acceptance Criteria

- [ ] AC-1: One-shot timer fires and signal delivered: `timer_getoverrun` returns `0`.
- [ ] AC-2: Periodic 10ms timer; process blocked for 100ms with one signal queued: next signal delivery shows `si_overrun ≈ 9`.
- [ ] AC-3: After delivery, `timer_getoverrun` returns same value as `si_overrun`.
- [ ] AC-4: Next fire (post-handler): `si_overrun` starts fresh from 0.
- [ ] AC-5: SIGEV_NONE periodic timer: count grows monotonically.
- [ ] AC-6: SIGEV_NONE: count never exceeds INT_MAX (capped).
- [ ] AC-7: Disarmed timer (never fired): returns `0`.
- [ ] AC-8: Disarmed-after-firing timer: returns the last overrun count.
- [ ] AC-9: `timerid` from another process: `-EINVAL`.
- [ ] AC-10: Deleted timer: `-EINVAL`.
- [ ] AC-11: Alarm-class timer across suspend: overrun reflects missed-during-suspend periods.

## Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
pub fn sys_timer_getoverrun(id: TimerId) -> isize {
    PosixTimers::do_timer_get_overrun(id)
}
```

`PosixTimers::do_timer_get_overrun(id) -> isize`:
1. let (k, guard) = posix_timer_get_locked(&current.signal, id).ok_or(-EINVAL)?;
2. let cnt = READ_ONCE(k.it_overrun_last);
3. drop(guard);
4. /* clamp to INT_MAX */
5. let ret = cnt.min(i32::MAX as i64) as isize;
6. ret

`hrtimer_forward` overrun accounting (called when periodic timer fires):
1. let intervals = (now − k.it.real.timer.expires + k.it_interval − 1) / k.it_interval;
2. k.it.real.timer.expires += intervals * k.it_interval;
3. k.it_overrun = (intervals - 1).max(0);          // 0 if "on time"
4. k.it_overrun_last = k.it_overrun;                // snapshot at signal queueing

Signal-delivery snapshot:
1. siginfo.si_overrun = k.it_overrun_last;
2. siginfo.si_timerid = k.id;
3. /* k.it_overrun reset to 0 only after this delivery; next forward starts fresh */

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `siglock_during_lookup` | INVARIANT | k_itimer fetched under sighand siglock |
| `no_mutation_on_query` | INVARIANT | it_overrun_last unchanged after query |
| `capped_at_int_max` | INVARIANT | return value ∈ [0, INT_MAX] |
| `one_shot_overrun_is_zero` | INVARIANT | non-periodic timer ⟹ overrun == 0 |

### Layer 2: TLA+

`uapi/timer_getoverrun.tla`:
- Variables: `it_overrun_last[id]`, `signal_pending[id]`.
- Properties:
  - `safety_overrun_matches_si_overrun` — query and siginfo carry the same value within one delivery cycle.
  - `safety_one_shot_always_zero` — interval == 0 ⟹ overrun == 0.
  - `safety_monotone_until_delivery` — overrun increments per missed fire; resets on signal delivery handler entry.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_timer_get_overrun` post: ret ∈ [0, INT_MAX] | `do_timer_get_overrun` |
| `do_timer_get_overrun` post: no field mutation | `do_timer_get_overrun` |
| `hrtimer_forward` post: it_overrun ≥ 0 | hrtimer accounting |
| siginfo.si_overrun == k.it_overrun_last at delivery | `posix_timer_event` |

### Layer 4: Verus/Creusot functional

Per-`timer_getoverrun(2)` man page, glibc `sysdeps/unix/sysv/linux/timer_getoverrun.c`, LTP `testcases/kernel/syscalls/timer_getoverrun/*`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

timer_getoverrun reinforcement:

- **Read-only path** — no state mutation; signal-delivery is the only writer.
- **Clamped return** — caps at INT_MAX to avoid signed truncation surprises.
- **Lookup under siglock** — k_itimer read atomic with respect to signal delivery.
- **Per-delivery semantics** — no cumulative leak across delivery boundaries.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on every entry** — randomized kstack on each query.
- **GRKERNSEC_HIDESYM on `posix_clock_kclocks`** — function-pointer table hidden from kallsyms.
- **Per-uid overrun-query rate limit** — hardened policy throttles `timer_getoverrun` to defeat timing-oracle abuse where attackers infer scheduler decisions from rapid overrun queries.
- **Foreign-process lookup rejected pre-IDR-touch** — `it_signal != current.signal` rejected before the IDR pointer is dereferenced; defense against side-channel probing of which timer IDs are alive in sibling processes.
- **Audit on suspicious overrun patterns** — hardened policy logs queries where the returned overrun is unusually large (e.g., >1e6) which suggests SIGEV_NONE timer farming attacks designed to inflate scheduler counters.
- **Reject query of timer_id allocated in current's vfork-parent** — between `clone(CLONE_VFORK)` and exec, the child's signal struct points to parent's POSIX-timer IDR; hardened policy rejects timer_getoverrun from such windows with `-EINVAL`.
- **No leakage of suspend-skew via overrun on alarm-class** — under hardened mode, `it_overrun_last` for alarm-class is reported modulo a coarse divisor when the querying process did not originate the suspend request; defense against fingerprinting how long the system was suspended.
- **GRKERNSEC_POSIX_TIMER_OVERRUN_CAP** — hardened policy lowers the cap below INT_MAX (default: 1_000_000) to bound the magnitude attackers can amplify via long-running SIGEV_NONE timers.
- **Refuse query under PaX KERNEXEC inconsistency** — if kernel-text consistency check fails during the call (rare W^X violation), the syscall returns `-EFAULT` rather than potentially reading attacker-corrupted overrun state.
- **Audit log includes `(uid, pid, timer_id, ret)` on every query above policy threshold** — defenders reconstruct exact overrun deltas correlated with scheduler events.
- **No-cross-userns timer_id reuse** — a timer created in one userns is never queryable from a sibling userns even if signal struct sharing somehow leaks; hardened policy validates `current.user_ns == k.userns`.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `timer_create.md`, `timer_settime.md`, `timer_gettime.md`, `timer_delete.md` siblings.
- `kernel/time/hrtimer.md` Tier-3: hrtimer forward / interval math.
- `kernel/time/alarmtimer.md` Tier-3: suspend-time overrun accounting.
- `kernel/time/posix_timers.md` Tier-3: k_itimer lifecycle.
- `signalfd.md` / `rt_sigtimedwait.md` for signal delivery semantics.
- Implementation code.
