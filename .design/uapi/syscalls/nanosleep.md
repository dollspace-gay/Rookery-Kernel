# Tier-5 syscall: nanosleep(2) — syscall 35

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/hrtimer.c (sys_nanosleep, common_nsleep_timens, hrtimer_nanosleep)
  - kernel/time/posix-timers.c (do_nanosleep)
  - include/uapi/linux/time.h (__kernel_timespec)
  - arch/*/include/generated/uapi/asm/unistd_64.h (35)
  - Documentation/core-api/timekeeping.rst, man nanosleep(2)
-->

## Summary

`nanosleep(2)` is the original POSIX sleep primitive: suspends the calling thread for a relative duration on `CLOCK_MONOTONIC` (Linux-specific; POSIX permitted `CLOCK_REALTIME` but Linux chose monotonic since 2.6.21 to avoid `settimeofday`-induced spurious wakes / hangs). On signal interruption it writes the residual duration into `rem` so userspace can loop the remainder. It is the syscall behind glibc/musl `sleep(3)`, `usleep(3)`, and pre-`clock_nanosleep` POSIX code; it is also frequently called directly by simple programs that don't need clock selection. The kernel implements it via the same hrtimer machinery used by `clock_nanosleep` — `nanosleep` is essentially `clock_nanosleep(CLOCK_MONOTONIC, 0, req, rem)`. Critical for: every shell `sleep` invocation, every `usleep` in pre-C11 code, every busy-loop test harness, the historical baseline for POSIX timing.

This Tier-5 covers the userspace ABI of syscall 35. hrtimer expiry, restart_block handling, and signal interaction are owned by `kernel/time/hrtimer.md` (Tier-3, planned).

## Signature

```c
int nanosleep(const struct timespec *req, struct timespec *rem);
```

Rust ABI shim:

```rust
pub fn sys_nanosleep(req: *const __kernel_timespec,
                    rem: *mut __kernel_timespec) -> isize;
```

Syscall number: **35**.
32-bit architectures supply `sys_nanosleep_time32` (deprecated for 2038-safety).

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `req` | `const struct __kernel_timespec *` | IN | relative sleep duration `{tv_sec: i64 ≥ 0, tv_nsec: i64 ∈ [0, 999_999_999]}` |
| `rem` | `struct __kernel_timespec *` | OUT (on EINTR) | residual time if interrupted; may be NULL |

## Return

- **Success**: `0` after the full duration elapsed.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `req.tv_nsec` ∉ `[0, 999_999_999]`; `req.tv_sec < 0` |
| `EFAULT` | `req` not readable; `rem != NULL` and not writable |
| `EINTR` | sleep interrupted by signal; if `rem != NULL`, residual written |

`ENOSYS` is not returned in modern kernels (>=2.6); historically signaled if hrtimer subsystem absent.

## ABI surface

`nanosleep` uses `CLOCK_MONOTONIC` implicitly. There is no clock selector and no `TIMER_ABSTIME` flag.

`struct __kernel_timespec`:

```c
struct __kernel_timespec {
    __kernel_time64_t tv_sec;     /* signed 64-bit seconds */
    long long         tv_nsec;    /* signed 64-bit nanoseconds, normalized [0, 1e9) */
};
```

Legacy 32-bit `struct timespec` (with 32-bit `tv_sec`) accepted only on pre-time64 32-bit builds via `sys_nanosleep_time32`.

## Compatibility contract

REQ-1: `req` validation:
- `copy_from_user` failure: `-EFAULT`.
- `tv_nsec ∉ [0, 999_999_999]`: `-EINVAL`.
- `tv_sec < 0`: `-EINVAL`.
- `tv_sec == 0 ∧ tv_nsec == 0`: returns `0` immediately (no sleep).

REQ-2: Clock semantics:
- Uses `CLOCK_MONOTONIC`; deadline = `now() + req`.
- Not affected by `clock_settime(CLOCK_REALTIME, ...)`.
- Not affected by NTP step adjustments to `CLOCK_REALTIME`.

REQ-3: Sleep mechanism:
- Arms `hrtimer` in `HRTIMER_MODE_REL` (relative).
- Caller blocks in `TASK_INTERRUPTIBLE` state.

REQ-4: Signal interruption:
- Pending signal (other than ignored / SIG_DFL-zeroed): wakes the sleeper.
- Returns `-EINTR`.
- If `rem != NULL`: writes residual `deadline − now()`, clamped to `≥ 0`.
- If `rem == NULL`: residual discarded.

REQ-5: SA_RESTART:
- `nanosleep` does NOT auto-restart on signal even with `SA_RESTART`.
- Kernel uses `restart_block` mechanism only for `restart_syscall(2)` machinery (used by glibc loop logic in `clock_nanosleep_restart`).
- Userspace standard loop: `while (nanosleep(&req, &rem) == -1 && errno == EINTR) req = rem;`.

REQ-6: Granularity:
- Per-hrtimer subsystem state: 1 ns if `CONFIG_HIGH_RES_TIMERS=y`, else `TICK_NSEC`.
- Actual wakeup may be later than requested (scheduling latency, lock contention); never earlier (deadline-strict).

REQ-7: Time-64 ABI:
- 64-bit architectures use `sys_nanosleep` (35) with `__kernel_timespec`.
- 32-bit architectures: `sys_nanosleep_time32` (35 legacy) with 32-bit `tv_sec` for 32-bit compat; `sys_clock_nanosleep_time64` (407) for time64-safe sleeps on `CLOCK_MONOTONIC`.

REQ-8: Signal safety:
- `nanosleep` is async-signal-safe per POSIX.1-2008.

REQ-9: Cancellation:
- Cancellation point per POSIX threads: pthread cancellation may unwind a blocked `nanosleep`.

REQ-10: Zero duration:
- `req == {0, 0}`: returns `0` immediately without entering hrtimer machinery; observable as no preemption point being inserted.

## Acceptance Criteria

- [ ] AC-1: `nanosleep({1, 0}, NULL)` returns `0` after ≥ 1 s.
- [ ] AC-2: `nanosleep({0, 100_000_000}, NULL)` returns `0` after ≥ 100 ms.
- [ ] AC-3: Interrupted by signal: returns `-EINTR`; `rem.tv_sec + rem.tv_nsec * 1e-9` ≈ unslept duration; non-negative.
- [ ] AC-4: `req == {0, 0}`: returns `0` immediately.
- [ ] AC-5: `req.tv_nsec == 1_000_000_000`: `-EINVAL`.
- [ ] AC-6: `req.tv_sec == -1`: `-EINVAL`.
- [ ] AC-7: `req == NULL`: `-EFAULT`.
- [ ] AC-8: `rem == NULL` ∧ interrupted: returns `-EINTR` with no write attempt.
- [ ] AC-9: `clock_settime(CLOCK_REALTIME, future)` does NOT wake sleeper (monotonic clock).
- [ ] AC-10: Sleep across explicit suspend: sleep duration extends by suspend time (monotonic frozen during suspend).
- [ ] AC-11: SA_RESTART signal does not auto-restart `nanosleep`; returns `-EINTR`.
- [ ] AC-12: Loop `while EINTR: nanosleep(&rem, &rem)` completes original duration exactly (within hrtimer granularity).

## Architecture

Rookery surface in `kernel/time/hrtimer.rs`:

```rust
pub fn sys_nanosleep(req_user: *const __kernel_timespec,
                    rem_user: *mut __kernel_timespec) -> isize {
    let req = copy_from_user::<KernelTimespec>(req_user).map_err(|_| -EFAULT)?;
    if !req.is_valid_duration() { return -EINVAL; }
    if req.is_zero() { return 0; }
    HrTimer::nsleep(ClockId::Monotonic as i32, /*abs=*/false, &req, rem_user)
}
```

`HrTimer::nsleep_relative(req, rem_user) -> isize`:
1. let now = Timekeeper::get_ts64();
2. let deadline = now + req;
3. let mut hrt = HrTimer::new(CLOCK_MONOTONIC, HRTIMER_MODE_ABS);
4. hrt.start(deadline);
5. /* TASK_INTERRUPTIBLE wait */
6. set_current_state(TASK_INTERRUPTIBLE);
7. let r = schedule_with_hrtimer(&hrt);
8. match r {
   - Expired => 0,
   - Signaled => {
       - hrt.cancel();
       - if !rem_user.is_null() {
           - let now2 = Timekeeper::get_ts64();
           - let rem = (deadline - now2).max(KernelTimespec::ZERO);
           - copy_to_user(rem_user, &rem).map_err(|_| -EFAULT)?;
         }
       - -EINTR
     },
 }

`KernelTimespec::is_valid_duration() -> bool`:
1. self.tv_sec ≥ 0 ∧ self.tv_nsec ∈ [0, 999_999_999]

`KernelTimespec::is_zero() -> bool`:
1. self.tv_sec == 0 ∧ self.tv_nsec == 0

The implementation is a thin layer over `clock_nanosleep` infrastructure; semantically `nanosleep(req, rem) == clock_nanosleep(CLOCK_MONOTONIC, 0, req, rem)` with errno-form return.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `req_normalized` | INVARIANT | bad `tv_nsec` / negative `tv_sec` ⟹ -EINVAL |
| `rem_null_allowed` | INVARIANT | `rem == NULL` never causes -EFAULT |
| `rem_writeback_only_on_eintr` | INVARIANT | non-EINTR returns never touch `rem` |
| `rem_nonnegative` | INVARIANT | residual is clamped to ≥ 0 |
| `zero_duration_fast_return` | INVARIANT | `{0,0}` ⟹ 0 without hrtimer arm |
| `monotonic_only` | INVARIANT | not affected by REALTIME settime |

### Layer 2: TLA+

`uapi/nanosleep.tla`:
- Variables: `now`, `signal_pending(task)`, `hrtimer_pending(deadline)`.
- Properties:
  - `safety_returns_eintr_or_zero` — terminates with EINTR or 0.
  - `safety_rem_clamped` — residual never negative.
  - `safety_monotonic_unaffected_by_settimeofday` — settimeofday does not wake nanosleep.
  - `liveness_eventually_wakes` — terminates on signal or deadline.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `nanosleep` post(0): now ≥ deadline | `HrTimer::nsleep_relative` |
| `nanosleep` post(EINTR, rem!=NULL): `rem == max(deadline - now, 0)` | `HrTimer::nsleep_relative` |
| `nanosleep` post(EINTR, rem==NULL): no userspace write | `HrTimer::nsleep_relative` |
| zero-duration fast return | `HrTimer::nsleep_relative` |

### Layer 4: Verus/Creusot functional

Per-`nanosleep(2)` man page, glibc `sysdeps/unix/sysv/linux/nanosleep.c`, musl `src/time/nanosleep.c`. Per-LTP `testcases/kernel/syscalls/nanosleep*` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

nanosleep reinforcement:

- **Per-CLOCK_MONOTONIC implicit** — defense against settimeofday-induced spurious wake-up DoS.
- **Per-rem clamped ≥ 0** — defense against per-negative residual confusing userspace EINTR loops.
- **Per-rem writeback only on EINTR** — defense against per-uninit-leak on success path.
- **Per-zero-duration fast path** — defense against pointless hrtimer arm DoS (`nanosleep(0,0)` storm).
- **Per-req validation pre-arm** — bad `req` never reaches hrtimer subsystem.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on every entry** — nanosleep is a high-frequency syscall in shell scripts and test harnesses; randomize kstack per entry.
- **PaX UDEREF on `req` and `rem`** — `copy_from_user(&req, req_user)` refuses kernel-mapped addresses; `copy_to_user(rem_user, &rem)` on EINTR write refuses the same. The EINTR-rem-write is a particularly attractive kernel-write primitive because attackers can frequently arrange to deliver signals to a controlled task; UDEREF blocks the type-confused-userspace-pointer attack class.
- **GRKERNSEC_CLOCK_RESOLUTION** — under hardened policy, sub-tick `nanosleep` requests from suid binaries are snapped to one tick (`TICK_NSEC`); defense against high-precision sleep + `clock_gettime` timing oracles (Spectre v1/v4, FLUSH+RELOAD). Tunable via `grsec_min_nanosleep_ns`.
- **CAP_SYS_TIME N/A** — read-only on clock; the sibling settime path enforces CAP_SYS_TIME.
- **CAP_WAKE_ALARM N/A** — `nanosleep` is hardcoded to `CLOCK_MONOTONIC`; alarm clocks are reachable only via `clock_nanosleep`/`timerfd_create`. No wakelock budget consumed.
- **adjtimex rate-limit irrelevant** — `nanosleep` sleeps on CLOCK_MONOTONIC which is NTP-slewed (not stepped); adjtimex cannot wake a `nanosleep` sleeper out-of-deadline.
- **GRKERNSEC_FIFO per-uid sleeper quota** — per-uid cap on concurrent armed nanosleep hrtimers; refuses arm with `-EAGAIN` (mapped to bounded sleep) past quota to break sleeper-flood DoS against hrtimer red-black-tree.
- **Refuse `nanosleep` with `tv_sec > 100 years`** — under hardened policy, durations exceeding `100 * 365 * 86400` seconds are rejected with `-EINVAL`; defense against attackers registering effectively-infinite sleeps as a covert wait-for-reboot oracle (the legitimate use case for ultra-long sleeps is `clock_nanosleep` with `TIMER_ABSTIME`, not the relative `nanosleep`).
- **Bounded restart_block on `nanosleep_restart`** — restart_block.fn validated against an allowlist; `do_nanosleep_restart` only.
- **GRKERNSEC_HIDESYM on hrtimer base addresses** — per-CPU hrtimer red-black-tree bases excluded from kallsyms.
- **Signal-delivery audit** — repeated EINTR-loop patterns from a single uid (high frequency of `nanosleep` → signal → `nanosleep` → signal) audit-logged as potential SA_RESTART abuse; the kernel does not auto-restart `nanosleep`, but attackers may use the resulting userspace tight loop as a side-channel measurement signal.
- **Zero-duration storm rate-limit** — `nanosleep({0,0})` calls per-uid rate-limited; high-rate zero-duration calls under hardened policy degrade gracefully into yield-only without entering hrtimer arm.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `kernel/time/hrtimer.md` Tier-3: hrtimer red-black-tree, expiry, restart_block protocol.
- `clock_nanosleep.md` sibling — clock-selectable / TIMER_ABSTIME variant.
- `clock_gettime.md` / `clock_settime.md` siblings.
- `pause(2)` / `select(2)` / `poll(2)` — non-deadline blocking.
- `usleep(3)` / `sleep(3)` — userspace wrappers.
- Implementation code.
