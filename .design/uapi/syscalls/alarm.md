# Tier-5 syscall: alarm(2) — syscall 37

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/itimer.c (sys_alarm, alarm_setitimer)
  - kernel/time/hrtimer.c (hrtimer_start, hrtimer_cancel)
  - include/uapi/linux/time.h (struct itimerval, ITIMER_REAL)
  - arch/*/include/generated/uapi/asm/unistd_64.h (37)
  - Documentation/core-api/timekeeping.rst, man alarm(2)
-->

## Summary

`alarm(2)` is a convenience wrapper over `setitimer(ITIMER_REAL, ...)` with second-granularity: arms (or, if `seconds == 0`, disarms) the per-process `ITIMER_REAL` timer to deliver a single `SIGALRM` to the process `seconds` from now. The previous remaining time (in seconds, rounded up) is returned. There is no interval / periodic variant — `alarm` always creates a one-shot. It is the most primitive timing API in UNIX, dating to V7; it shares the same `task->signal->real_timer` hrtimer slot as `setitimer(ITIMER_REAL, ...)` so the two APIs **interlock**: arming via `alarm` cancels any prior `setitimer(ITIMER_REAL, ...)` configuration and vice versa. Critical for: shell-script-level timeouts, fork-and-wait-with-deadline patterns, libc `sleep(3)` historical implementation (modern libc uses `nanosleep`), pre-POSIX timing code.

This Tier-5 covers the userspace ABI of syscall 37. The hrtimer-backed `ITIMER_REAL` armament is owned by `kernel/time/itimer.md` (Tier-3, planned).

## Signature

```c
unsigned int alarm(unsigned int seconds);
```

Rust ABI shim:

```rust
pub fn sys_alarm(seconds: u32) -> isize;   // returns prior remaining seconds, never negative
```

Syscall number: **37**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `seconds` | `u32` | IN | duration in whole seconds from now to next `SIGALRM` delivery; `0` disarms |

## Return

- **Always returns** the number of seconds remaining on any prior `ITIMER_REAL` alarm (rounded up to the next whole second), or `0` if no prior alarm was set.
- Never fails in the traditional sense (no `errno`).
- The return value is `unsigned int`; Rust internal returns `isize` of the same non-negative value.

## Errors

None observable as errno. The syscall **always succeeds**:

| Outcome | Cause |
|---|---|
| Return `0` | no prior alarm; or prior alarm already expired before this call |
| Return `n` | `n` whole seconds remained on prior alarm at call time |

(Pathological inputs like `seconds == UINT_MAX` are accepted and stored; the kernel clamps internally to representable `ktime_t` range.)

## ABI surface

`alarm(0)` semantics:

- Disarms `ITIMER_REAL`.
- Returns prior remaining seconds.

`alarm(n)` semantics:

- Cancels any prior `ITIMER_REAL` (whether armed via `alarm` or `setitimer`).
- Arms `ITIMER_REAL` with `it_value = {n, 0}` and `it_interval = {0, 0}` (one-shot).
- Returns prior remaining seconds.

Interlock with `setitimer`:

| Sequence | Effect |
|---|---|
| `setitimer(ITIMER_REAL, {{1,0},{5,0}}, NULL)` then `alarm(10)` | periodic timer cancelled; `alarm` one-shot at 10 s; `alarm` returns `5` |
| `alarm(5)` then `setitimer(ITIMER_REAL, &nv, &ov)` | `ov.it_value ≈ {5,0}`, `ov.it_interval == {0,0}` |
| `alarm(5)` then `alarm(0)` | disarms; second call returns `5` |
| `alarm(5)` then `alarm(10)` | second call returns `5`; new alarm at 10 s |

Signal delivery: `SIGALRM` to the calling process (default action: terminate; usually handled by signal handler installed via `sigaction`).

## Compatibility contract

REQ-1: `seconds == 0` disarms:
- Cancel `sig.real_timer`.
- `sig.it_real_incr = 0`.
- Return prior remaining seconds (rounded up).

REQ-2: `seconds > 0` arms:
- Cancel prior `sig.real_timer` if active.
- Capture prior remaining as return value.
- Set `sig.it_real_incr = 0` (one-shot).
- Arm `sig.real_timer` with `deadline = now + seconds * NSEC_PER_SEC` in `HRTIMER_MODE_REL`.

REQ-3: Return-value rounding:
- Remainder rounded UP to next whole second (per POSIX: `alarm` returns "the number of seconds until the previously requested alarm would have generated the SIGALRM signal").
- Sub-second residual (e.g., 4.3 s) reports as `5`.

REQ-4: Shared state with `setitimer`:
- `alarm` and `setitimer(ITIMER_REAL, ...)` mutate the same `task->signal->real_timer`.
- Cross-API cancel honored.

REQ-5: Per-process scope:
- All threads share via `task->signal`.
- Concurrent `alarm` from multiple threads is linearizable under `sig.siglock`.

REQ-6: Inheritance:
- `fork(2)`: child has alarm cleared.
- `execve(2)`: alarm cleared in new image.

REQ-7: Saturation:
- `seconds * NSEC_PER_SEC` checked against `ktime_t` range; on overflow, deadline is `KTIME_MAX`.

REQ-8: Signal safety:
- `alarm` is async-signal-safe per POSIX.1-2008.

REQ-9: Cancellation point:
- Not a pthread cancellation point.

REQ-10: SIGALRM disposition:
- If no handler installed, default action is "Term" — process killed on first expiry.

## Acceptance Criteria

- [ ] AC-1: `alarm(0)` with no prior alarm: returns `0`.
- [ ] AC-2: `alarm(0)` with `n` seconds remaining: returns `n` (rounded up); subsequent `alarm(0)` returns `0`.
- [ ] AC-3: `alarm(5)`: SIGALRM arrives after ≥ 5 s.
- [ ] AC-4: `alarm(5)` then `alarm(0)` after 1 s: second call returns `4` (or `5`); no SIGALRM arrives.
- [ ] AC-5: `alarm(5)` then `setitimer(ITIMER_REAL, &disarm, &ov)`: `ov.it_value ≈ {5,0}`; SIGALRM cancelled.
- [ ] AC-6: `setitimer(ITIMER_REAL, periodic, NULL)` then `alarm(10)`: periodic cancelled; `alarm` returns first-expiry remainder.
- [ ] AC-7: `alarm(UINT_MAX)`: accepted; SIGALRM scheduled for `UINT_MAX` seconds (~136 years).
- [ ] AC-8: `fork`: parent retains alarm; child has none.
- [ ] AC-9: `execve`: alarm cleared in new image.
- [ ] AC-10: Concurrent `alarm` from two threads: linearizable; second-armer's value wins; second caller's return value is first-armer's remaining.
- [ ] AC-11: Rounding: 4.3 s residual reports as `5`.
- [ ] AC-12: SIGALRM default disposition without handler: process terminated on expiry.

## Architecture

Rookery surface in `kernel/time/itimer.rs`:

```rust
pub fn sys_alarm(seconds: u32) -> isize {
    /* model as setitimer(ITIMER_REAL, ...) wrapper */
    let nv = OldItimerval {
        it_interval: OldTimeval::ZERO,
        it_value:    OldTimeval { tv_sec: seconds as i64, tv_usec: 0 },
    };
    let old = Itimer::set_real(current, &nv);
    /* return prior remaining seconds, rounded up */
    let rem_ns = old.it_value.tv_sec * NSEC_PER_SEC + old.it_value.tv_usec * 1000;
    let rem_sec = (rem_ns + NSEC_PER_SEC - 1) / NSEC_PER_SEC;   // ceil
    rem_sec as isize
}
```

`Itimer::set_real(task, nv)` is the shared implementation from `setitimer.md`. The only `alarm`-specific logic is the rounding-up of the returned residual.

Round-up rule expressed explicitly:

```rust
fn ceil_to_seconds(tv: OldTimeval) -> u32 {
    let sec = tv.tv_sec as u32;
    if tv.tv_usec > 0 { sec.saturating_add(1) } else { sec }
}
```

Saturation on `seconds == UINT_MAX`:

```rust
let deadline_ns = (seconds as u64).saturating_mul(NSEC_PER_SEC as u64);
let deadline = ktime_from_ns(deadline_ns.min(KTIME_MAX as u64));
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `zero_disarms` | INVARIANT | `seconds == 0` ⟹ `hrtimer_active(sig.real_timer) == false` post |
| `prior_remaining_returned` | INVARIANT | return == ceil(prior remainder / 1s) |
| `siglock_held` | INVARIANT | sig.siglock held during state mutation |
| `cross_api_cancel_honored` | INVARIANT | prior `setitimer(ITIMER_REAL, periodic)` state cancelled by `alarm` arm |
| `saturation_safe` | INVARIANT | `seconds == UINT_MAX` ⟹ deadline ≤ KTIME_MAX (no overflow) |
| `never_returns_negative` | INVARIANT | return value ≥ 0 |

### Layer 2: TLA+

`uapi/alarm.tla`:
- Variables: `sig.real_timer.{active, deadline}`, `sig.it_real_incr`, `sig.siglock`.
- Properties:
  - `safety_zero_disarms` — `alarm(0)` ⟹ no pending SIGALRM.
  - `safety_arm_cancels_prior` — new `alarm(n)` cancels any prior `ITIMER_REAL` armament.
  - `safety_return_matches_prior` — return = ceil(prior_remaining_seconds).
  - `liveness_sigalrm_eventually_delivered` — armed alarm eventually delivers SIGALRM.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `alarm` post: return == ceil(prior `it_value` in seconds) | `sys_alarm` |
| `alarm(0)` post: `hrtimer_active(sig.real_timer) == false` | `Itimer::set_real` |
| `alarm(n)` post: `hrtimer_active(sig.real_timer) == true`, `deadline ≈ now + n*1s` | `Itimer::set_real` |
| `alarm` post: `sig.it_real_incr == 0` (always one-shot) | `Itimer::set_real` |

### Layer 4: Verus/Creusot functional

Per-`alarm(2)` man page, glibc `sysdeps/unix/sysv/linux/alarm.c`, musl `src/time/alarm.c`. Per-LTP `testcases/kernel/syscalls/alarm/*` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

alarm reinforcement:

- **Per-rounding-up of prior remaining seconds** — defense against per-zero-return on truncation-to-0 race (e.g., 0.4s residual incorrectly reported as 0).
- **Per-shared-with-setitimer interlock** — defense against per-stale-alarm-fires after `setitimer` reprogramming.
- **Per-siglock-held mutation** — defense against torn state racing with `getitimer`/`setitimer`.
- **Per-saturation on UINT_MAX** — defense against per-ktime_t overflow producing immediate-fire on large input.
- **Per-process-scope strict** — cross-process arming not possible via this syscall.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on every entry** — `alarm` is callable from any task with no privilege; randomize kstack per entry.
- **PaX UDEREF irrelevant** — `alarm` takes a scalar `unsigned int` argument; no userspace pointer dereference. The companion `setitimer`/`getitimer` paths enforce UDEREF on their pointer arguments.
- **GRKERNSEC_CLOCK_RESOLUTION** — `alarm` is second-granularity by definition; coarsening is naturally enforced. The return-value rounding up to whole seconds means `alarm` does not leak sub-second timekeeper state to userspace.
- **CAP_SYS_TIME N/A** — `alarm` does not mutate the wall clock.
- **CAP_WAKE_ALARM N/A** — `ITIMER_REAL` does not wake from suspend; use `timer_create(CLOCK_*_ALARM, ...)` for that.
- **adjtimex rate-limit interaction** — `ITIMER_REAL` uses hrtimer on monotonic time; NTP slew via `adjtimex` changes the rate `alarm` advances at, but bounded slew prevents abrupt mass-fire of armed alarms.
- **Per-uid alarm-arm rate-limit** — `alarm(n)` calls per uid rate-limited under hardened policy to defeat alarm-arm floods that destabilize the hrtimer red-black-tree.
- **Refuse `alarm(0)` storm** — repeated `alarm(0)` calls at >1000 Hz from a single uid audit-logged as a potential `getitimer`-via-`alarm` side-channel probe (the return value of `alarm(0)` leaks the remaining time on a prior alarm, which can be used to measure CPU-time elapsed between arming and reading).
- **Audit `alarm(UINT_MAX)` — UINT_MAX (~136 years) is implausible legitimate use; audit-logged.
- **Saturation strict** — never compute `seconds * NSEC_PER_SEC` without explicit `saturating_mul`; defense against integer-overflow producing nonsensical deadlines.
- **GRKERNSEC_HIDESYM on `signal->real_timer` address** — process signal struct excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`.
- **GRKERNSEC_FIFO scaled to alarm flood** — per-uid quota on simultaneous armed `ITIMER_REAL` timers; refuses arm past quota (the timer becomes a no-op with the prior return value preserved) under hardened policy.
- **SIGALRM delivery to controlled handlers only** — under SECCOMP-loose / SECCOMP-strict combination, the audit subsystem records the disposition of SIGALRM at arm time; defense-in-depth against attacker patterns that rely on SIGALRM default-action killing a target process.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `kernel/time/itimer.md` Tier-3: itimer state machine, expiry callback.
- `kernel/time/hrtimer.md` Tier-3: hrtimer subsystem.
- `kernel/signal.md` Tier-3: signal delivery integration.
- `getitimer.md` / `setitimer.md` siblings.
- `timer_create(2)` family — modern POSIX interval timers.
- `timerfd_create(2)` — fd-based interval timer.
- `sleep(3)` / `usleep(3)` — userspace wrappers (modern libc uses `nanosleep`).
- glibc / musl userspace `alarm` wrappers.
- Implementation code.
