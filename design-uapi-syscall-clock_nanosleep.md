---
title: "Tier-5 syscall: clock_nanosleep(2) — syscall 230"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`clock_nanosleep(2)` suspends the calling thread for a duration measured against a caller-selected clock, with optional absolute deadline semantics via `TIMER_ABSTIME`. It supersedes the original `nanosleep(2)` (which is hardcoded to `CLOCK_REALTIME` relative) by exposing every settable clock plus alarm clocks for suspend-aware wakeups. Critical for: `pthread_cond_timedwait` (clock-aware deadlines), `sleep(3)`/glibc `nanosleep` implementation (delegates here when CLOCK_MONOTONIC is wanted), realtime scheduling loops using `TIMER_ABSTIME + CLOCK_MONOTONIC` to avoid drift across iterations, suspend-survivable alarms via `CLOCK_BOOTTIME_ALARM`. Per-`TIMER_ABSTIME` does NOT write back `remain` on signal interruption (caller already has the deadline); per-relative writes residual time on EINTR for restart loops.

This Tier-5 covers the userspace ABI of syscall 230. hrtimer sleep state machine and signal-restart protocol are owned by `kernel/time/hrtimer.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: Relative `{1, 0}` (1 second): returns `0` after ≥ 1 s elapsed.
- [ ] AC-2: Relative interrupted by signal: returns `EINTR`, `remain.tv_sec + remain.tv_nsec * 1e-9` ≈ unslept duration.
- [ ] AC-3: Absolute deadline in past: returns `0` immediately.
- [ ] AC-4: Absolute deadline + signal: returns `EINTR`, `remain` NOT modified.
- [ ] AC-5: `clock_id == CLOCK_THREAD_CPUTIME_ID`: `EINVAL`.
- [ ] AC-6: `tv_nsec == 1_000_000_000`: `EINVAL`.
- [ ] AC-7: `tv_sec < 0` (relative): `EINVAL`.
- [ ] AC-8: `CLOCK_REALTIME_ALARM` without `CAP_WAKE_ALARM`: `EPERM`.
- [ ] AC-9: `CLOCK_REALTIME` + `TIMER_ABSTIME`: `clock_settime` jump past deadline wakes sleeper.
- [ ] AC-10: `CLOCK_BOOTTIME`: sleep across explicit suspend resumes at correct delta.
- [ ] AC-11: `flags == 2`: `EINVAL` (unknown flag bit).
- [ ] AC-12: `request == NULL`: `EFAULT`.

### Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
pub fn sys_clock_nanosleep(clock_id: i32, flags: i32,
                           req_user: *const __kernel_timespec,
                           rem_user: *mut __kernel_timespec) -> isize {
    if (flags & !TIMER_ABSTIME) != 0 { return -EINVAL; }
    let req = copy_from_user::<KernelTimespec>(req_user).map_err(|_| -EFAULT)?;
    if !req.is_valid_duration() { return -EINVAL; }
    PosixTimers::clock_nanosleep(clock_id, flags, req, rem_user)
}
```

`PosixTimers::clock_nanosleep(clock_id, flags, req, rem_user) -> isize`:
1. let abs = (flags & TIMER_ABSTIME) != 0;
2. match ClockId::try_from(clock_id) {
   - Realtime | Monotonic | Boottime | Tai => HrTimer::nsleep(clock_id, abs, &req, rem_user),
   - RealtimeAlarm | BoottimeAlarm => {
       - if !capable(CAP_WAKE_ALARM) { return -EPERM; }
       - AlarmTimer::nsleep(clock_id, abs, &req, rem_user)
     },
   - ProcessCpuTimeId => PosixCpuTimer::nsleep_process(abs, &req, rem_user),
   - ThreadCpuTimeId => -EINVAL,
   - MonotonicRaw | RealtimeCoarse | MonotonicCoarse => -EOPNOTSUPP,
   - _ if (clock_id as u32 & CLOCKFD_MASK) == CLOCKFD =>
       PosixClock::dispatch_nsleep(clock_id, abs, &req, rem_user),
   - _ => -EINVAL,
 }

`HrTimer::nsleep(clock, abs, req, rem_user) -> isize`:
1. let now = Timekeeper::now(clock);
2. let deadline = if abs { *req } else { now + *req };
3. if deadline <= now { return 0; }
4. let mut hrt = HrTimer::new(clock, HRTIMER_MODE_ABS);
5. hrt.start(deadline);
6. /* wait */
7. let r = wait_for_signal_or_expiry(&hrt);
8. match r {
   - Expired => { hrt.cancel(); 0 },
   - Signaled => {
       - hrt.cancel();
       - if !abs ∧ !rem_user.is_null() {
           - let rem = deadline - Timekeeper::now(clock);
           - copy_to_user(rem_user, &rem.max(ZERO)).map_err(|_| -EFAULT)?;
         }
       - -EINTR
     },
 }

### Out of Scope

- `kernel/time/hrtimer.md` Tier-3: hrtimer red-black-tree, expiry, restart_block.
- `kernel/time/alarmtimer.md` Tier-3: RTC wakeup-source alarm-class clocks.
- `kernel/time/posix_cpu_timers.md` Tier-3: process CPU-time-based sleep.
- `nanosleep.md` sibling — relative CLOCK_REALTIME-only variant.
- `clock_gettime.md` / `clock_settime.md` / `clock_getres.md` siblings.
- `timer_create(2)` / `timerfd_settime(2)` — interval timers, not sleep.
- glibc / musl userspace `clock_nanosleep` wrappers.
- Implementation code.

### signature

```c
int clock_nanosleep(clockid_t clock_id, int flags,
                    const struct timespec *request,
                    struct timespec *remain);
```

Rust ABI shim:

```rust
pub fn sys_clock_nanosleep(clock_id: i32, flags: i32,
                           request: *const __kernel_timespec,
                           remain: *mut __kernel_timespec) -> isize;
```

Syscall number: **230**.
On 32-bit architectures the time-64-safe variant is `sys_clock_nanosleep_time64` (syscall 407).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `clock_id` | `clockid_t` (`i32`) | IN | clock to sleep against |
| `flags` | `i32` | IN | `0` (relative) or `TIMER_ABSTIME` (`1`) — absolute deadline |
| `request` | `const struct __kernel_timespec *` | IN | sleep duration (relative) or deadline (absolute) |
| `remain` | `struct __kernel_timespec *` | OUT | (relative only) residual time on EINTR; ignored if `TIMER_ABSTIME` set |

### return

- **Success**: `0` after the full interval / deadline.
- **Failure**: positive errno-equivalent (note: returns errno value directly per POSIX, not `-1 + errno`); Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `request.tv_nsec` ∉ `[0, 999_999_999]`; `request.tv_sec < 0`; unknown `clock_id`; `flags` has bits beyond `TIMER_ABSTIME`; `clock_id == CLOCK_THREAD_CPUTIME_ID` (self-CPU-sleep undefined) |
| `EFAULT` | `request` or (writable) `remain` not in valid userspace |
| `EINTR` | sleep interrupted by signal; per-relative `*remain` populated |
| `ENOTSUP` | `clock_id` is a dynamic POSIX clock that does not implement `clock_nsleep` |
| `EPERM` | `clock_id ∈ {CLOCK_REALTIME_ALARM, CLOCK_BOOTTIME_ALARM}` without `CAP_WAKE_ALARM` |
| `ECANCELED` | (since Linux 5.10 for `CLOCK_REALTIME` with `TIMER_ABSTIME`) clock was set during the sleep and TFD_TIMER_CANCEL_ON_SET semantics request cancellation (only via timerfd; raw clock_nanosleep does not surface) |

### abi surface

Acceptable `clock_id` values:

| Constant | Value | TIMER_ABSTIME meaningful | CAP_WAKE_ALARM |
|---|---|---|---|
| `CLOCK_REALTIME` | `0` | yes (jump-aware) | no |
| `CLOCK_MONOTONIC` | `1` | yes | no |
| `CLOCK_PROCESS_CPUTIME_ID` | `2` | yes | no |
| `CLOCK_THREAD_CPUTIME_ID` | `3` | **invalid** (cannot sleep on own CPU time) | — |
| `CLOCK_BOOTTIME` | `7` | yes (suspend-aware accounting) | no |
| `CLOCK_REALTIME_ALARM` | `8` | yes (wakes from suspend) | required |
| `CLOCK_BOOTTIME_ALARM` | `9` | yes (wakes from suspend) | required |
| `CLOCK_TAI` | `11` | yes | no |
| dynamic posix clock (CLOCKFD) | varies | per backend | per backend |

`flags`:

| Flag | Value | Meaning |
|---|---|---|
| `0` | `0` | `request` is a relative duration |
| `TIMER_ABSTIME` | `1` | `request` is an absolute time on `clock_id` |

`struct __kernel_timespec` layout matches `clock_gettime.md`.

### compatibility contract

REQ-1: `request` validation:
- `copy_from_user` failure: `-EFAULT`.
- `tv_nsec ∉ [0, 999_999_999]`: `-EINVAL`.
- `tv_sec < 0`: `-EINVAL`.

REQ-2: `flags` validation:
- Bits outside `TIMER_ABSTIME` set: `-EINVAL`.

REQ-3: `clock_id` validation:
- `CLOCK_THREAD_CPUTIME_ID`: `-EINVAL` (sleeping on your own CPU time would never advance during sleep).
- Other unrecognized: `-EINVAL`.

REQ-4: Capability gate:
- `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM`: `CAP_WAKE_ALARM` required else `-EPERM`.

REQ-5: Relative sleep:
- Computes deadline = `now(clock_id) + request`.
- Sleeps on `hrtimer` armed against deadline.
- On signal interruption: `-EINTR` and (if `remain != NULL`) writes residual.

REQ-6: Absolute sleep (`TIMER_ABSTIME`):
- Treats `request` directly as deadline on `clock_id`.
- On signal interruption: `-EINTR`, `remain` NOT written.
- Past deadline (`request < now`): returns `0` immediately.

REQ-7: `CLOCK_REALTIME` + `TIMER_ABSTIME`:
- Wakes if `clock_settime(CLOCK_REALTIME, t)` moves time past deadline.
- Does NOT cancel-on-set by default (only timerfd `TFD_TIMER_CANCEL_ON_SET` does).

REQ-8: `CLOCK_BOOTTIME` semantics:
- Sleep continues across suspend (boottime accounts for suspend duration).

REQ-9: `CLOCK_*_ALARM`:
- Sleep wakes the device from suspend at the deadline.
- Uses RTC wakeup-source under the hood.

REQ-10: `remain` writeback (relative only):
- If signal interrupts: `*remain = deadline - now`.
- Caller loop pattern: `while (clock_nanosleep(clk, 0, &req, &rem) == EINTR) req = rem;`.

REQ-11: Dynamic POSIX clock:
- Dispatched to `posix_clock_ops::nsleep`.
- Backend may not implement: `-ENOTSUP`.

REQ-12: Restart semantics:
- Kernel uses `restart_block` for SA_RESTART-equivalent on relative sleeps; the userspace contract is that the syscall returns errno (not `-1`) and the libc wrapper translates.

REQ-13: Time-64 ABI:
- 32-bit architectures use `sys_clock_nanosleep_time64` (407) with `__kernel_timespec` (64-bit `tv_sec`).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `req_normalized` | INVARIANT | bad tv_nsec ⟹ -EINVAL |
| `flags_validated` | INVARIANT | flags & ~TIMER_ABSTIME ⟹ -EINVAL |
| `thread_cputime_rejected` | INVARIANT | CLOCK_THREAD_CPUTIME_ID ⟹ -EINVAL |
| `cap_wake_alarm_enforced` | INVARIANT | alarm clock w/o CAP_WAKE_ALARM ⟹ -EPERM |
| `remain_writeback_relative_only` | INVARIANT | TIMER_ABSTIME: never writes remain |
| `past_deadline_returns_zero` | INVARIANT | abs deadline ≤ now ⟹ 0 |

### Layer 2: TLA+

`uapi/clock_nanosleep.tla`:
- Variables: `clock(c)`, `signal_pending(task)`, `hrtimer_pending(deadline)`.
- Properties:
  - `safety_relative_returns_eintr_or_zero` — every relative sleep terminates with EINTR (and updated remain) or 0.
  - `safety_abs_no_remain_writeback` — TIMER_ABSTIME path never writes remain.
  - `safety_settime_wakes_abstime_realtime` — clock_settime past deadline wakes blocked sleeper.
  - `liveness_eventually_wakes` — sleep terminates (signal, deadline, or settime jump).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `clock_nanosleep` post(0): now(clock) ≥ deadline | `HrTimer::nsleep` |
| `clock_nanosleep` post(EINTR rel): remain == deadline − now (≥ 0) | `HrTimer::nsleep` |
| `clock_nanosleep` post(EINTR abs): remain memory unchanged | `HrTimer::nsleep` |
| CAP_WAKE_ALARM enforced pre-arm | `AlarmTimer::nsleep` |

### Layer 4: Verus/Creusot functional

Per-`clock_nanosleep(2)` man page, `Documentation/core-api/timekeeping.rst`, glibc `sysdeps/unix/sysv/linux/clock_nanosleep.c`, musl `src/time/clock_nanosleep.c`. Per-LTP `testcases/kernel/syscalls/clock_nanosleep/*` round-trip.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

clock_nanosleep reinforcement:

- **Per-TIMER_ABSTIME never writes remain** — defense against per-uninit-leak of kernel pointer to remain buffer.
- **Per-CAP_WAKE_ALARM enforced before hrtimer arm** — defense against unprivileged wakeup-source consumption.
- **Per-CLOCK_THREAD_CPUTIME_ID refused** — defense against logical sleep deadlock (self-CPU clock).
- **Per-signal restart_block bounded** — restart block sanitized; cannot reuse with mutated clock_id.
- **Per-deadline-in-past zero-return** — never arms a timer that immediately expires.

### grsecurity/pax-style reinforcement

- **PAX_RANDKSTACK on every entry** — clock_nanosleep is the kernel sleep primitive for high-precision scheduling loops; layout-randomize each entry.
- **PaX UDEREF on `request` and `remain`** — `copy_from_user(&req, request)` and `copy_to_user(remain, &rem)` refuse kernel-mapped addresses. The `remain` writeback in particular is a kernel-write primitive (writes `deadline - now`, a timekeeper-derived value) — UDEREF prevents type-confused writes into the kernel.
- **GRKERNSEC_CLOCK_RESOLUTION** — under hardened policy, `clock_nanosleep` for suid binaries enforces a minimum sleep granularity of `TICK_NSEC` (snaps sleeps shorter than one tick to one tick); defense against using sub-microsecond sleep + `clock_gettime` to construct timing oracles for Spectre-class side-channel attacks. Tunable: `grsec_min_nanosleep_ns` (default TICK_NSEC under hardened profile, 1 ns under permissive).
- **CAP_WAKE_ALARM strict** — alarm clocks require CAP_WAKE_ALARM at sleep-arm time; under userns-hardened policy, CAP_WAKE_ALARM honored only in init userns to prevent containers from consuming wakelock budget.
- **CAP_SYS_TIME N/A** — read of clock for deadline computation only; sibling `clock_settime` enforces CAP_SYS_TIME.
- **adjtimex rate-limit interaction** — `CLOCK_REALTIME + TIMER_ABSTIME` sleepers can be woken via clock-jump (`clock_settime` or `adjtimex(ADJ_STEP)`); the sibling rate-limit on `adjtimex` indirectly bounds the rate at which a privileged attacker can wake batches of sleeping tasks (defense against scheduling-DoS via cancel-storms).
- **GRKERNSEC_FIFO per-uid sleep-arm quota** — per-uid cap on concurrent armed alarm timers; refuses arm with `-EPERM` past quota to break alarm-flood DoS.
- **Refuse `CLOCK_REALTIME_ALARM` + `TIMER_ABSTIME` with deadline > 1 year in future** — under hardened policy, alarm sleeps with deadlines > 365 days are refused with `-EINVAL` to prevent attackers from registering long-deadline alarms that survive multiple suspend cycles and act as covert wakeup oracles.
- **Per-clock_id whitelist** — unknown clock IDs always rejected; never falls through to undefined backend.
- **GRKERNSEC_HIDESYM on hrtimer node addresses** — hrtimer red-black-tree nodes excluded from kallsyms; high-value target for race-window discovery.
- **Bounded restart_block reuse** — under hardened policy, restart_block.fn pointer validated against an allowlist (only `do_restart_poll`, `hrtimer_nanosleep_restart`, etc.); defense against attacks that mutate restart_block to call arbitrary kernel functions on signal return.

