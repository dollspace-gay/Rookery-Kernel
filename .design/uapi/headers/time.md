# Tier-5 UAPI: include/uapi/linux/time.h — time(2) family ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/time.h (~86 lines)
  - include/uapi/linux/time_types.h (struct __kernel_timespec / itimerspec)
  - kernel/time/posix-timers.c
  - kernel/time/timekeeping.c
  - kernel/time/itimer.c
  - kernel/time/ntp.c (adjtimex)
-->

## Summary

`include/uapi/linux/time.h` declares the kernel-side time ABI: the user-visible `struct timespec` / `timeval` / `itimerspec` / `itimerval` / `timezone` structures, the y2038-safe `__kernel_timespec`/`__kernel_itimerspec` (in `time_types.h`), the POSIX.1b clock IDs (`CLOCK_REALTIME` ... `CLOCK_TAI`, auxiliary `CLOCK_AUX` block), the interval-timer IDs (`ITIMER_REAL` / `_VIRTUAL` / `_PROF`), and the `TIMER_ABSTIME` flag for `clock_nanosleep`/`timer_settime`. This ABI is the surface for `clock_gettime(2)`, `clock_settime(2)`, `clock_getres(2)`, `clock_nanosleep(2)`, `clock_adjtime(2)`, `timer_create(2)`/`timer_settime(2)`/`timer_gettime(2)`/`timer_getoverrun(2)`/`timer_delete(2)`, `getitimer(2)`/`setitimer(2)`, `gettimeofday(2)`/`settimeofday(2)`, `adjtimex(2)` and is shared with VDSO fast-paths. Critical for: POSIX timers, hrtimer scheduling, NTP / chrony / PTP synchronization, alarm clocks, auxiliary clock domains.

This Tier-5 covers `include/uapi/linux/time.h` (~86 lines).

## ABI surface

| Constant / Type | Value | Purpose |
|---|---|---|
| `struct timespec` | `{ __kernel_old_time_t tv_sec; long tv_nsec; }` | POSIX nanosecond timestamp (legacy `time_t`; y2038-unsafe on 32-bit) |
| `struct __kernel_timespec` | `{ __kernel_time64_t tv_sec; long long tv_nsec; }` (from `time_types.h`) | y2038-safe 64-bit timestamp |
| `struct timeval` | `{ __kernel_old_time_t tv_sec; __kernel_suseconds_t tv_usec; }` | POSIX microsecond timestamp |
| `struct itimerspec` | `{ timespec it_interval; timespec it_value; }` | timer_settime/timerfd_settime arg |
| `struct __kernel_itimerspec` | `{ __kernel_timespec it_interval; it_value; }` | y2038-safe |
| `struct itimerval` | `{ timeval it_interval; timeval it_value; }` | setitimer/getitimer arg |
| `struct timezone` | `{ int tz_minuteswest; int tz_dsttime; }` | legacy gettimeofday tz arg (obsolete) |
| `ITIMER_REAL` | `0` | setitimer: SIGALRM on expiry, wall-clock |
| `ITIMER_VIRTUAL` | `1` | SIGVTALRM, user CPU time |
| `ITIMER_PROF` | `2` | SIGPROF, user + system CPU time |
| `CLOCK_REALTIME` | `0` | wall-clock, settable by `clock_settime` (NTP-disciplined) |
| `CLOCK_MONOTONIC` | `1` | monotonic since boot, NTP-adjusted, no jumps backward |
| `CLOCK_PROCESS_CPUTIME_ID` | `2` | per-process CPU time |
| `CLOCK_THREAD_CPUTIME_ID` | `3` | per-thread CPU time |
| `CLOCK_MONOTONIC_RAW` | `4` | raw hardware monotonic, no NTP slew |
| `CLOCK_REALTIME_COARSE` | `5` | low-resolution REALTIME (no HW read) |
| `CLOCK_MONOTONIC_COARSE` | `6` | low-resolution MONOTONIC |
| `CLOCK_BOOTTIME` | `7` | MONOTONIC + suspend time |
| `CLOCK_REALTIME_ALARM` | `8` | REALTIME + wake-from-suspend (CAP_WAKE_ALARM) |
| `CLOCK_BOOTTIME_ALARM` | `9` | BOOTTIME + wake-from-suspend (CAP_WAKE_ALARM) |
| `CLOCK_SGI_CYCLE` | `10` | reserved (driver removed; ID not reused) |
| `CLOCK_TAI` | `11` | International Atomic Time (REALTIME + leap-seconds; CAP_SYS_TIME to set) |
| `MAX_CLOCKS` | `16` | POSIX clock ID space ceiling |
| `CLOCK_AUX` | `MAX_CLOCKS` (16) | auxiliary clock block base |
| `MAX_AUX_CLOCKS` | `8` | max dynamically configured aux clocks |
| `CLOCK_AUX_LAST` | `CLOCK_AUX + MAX_AUX_CLOCKS - 1` (23) | aux clock block ceiling |
| `CLOCKS_MASK` | `CLOCK_REALTIME \| CLOCK_MONOTONIC` (1) | (legacy mask; bitwise convention) |
| `CLOCKS_MONO` | `CLOCK_MONOTONIC` (1) | (legacy alias) |
| `TIMER_ABSTIME` | `0x01` | clock_nanosleep / timer_settime: absolute deadline |
| (NTP-API constants) | from `include/uapi/linux/timex.h` | `adjtimex` modes / `ntp_adjtime` ABI (NTP_API series) |

NTP_API surface (referenced from `linux/timex.h`, kept here for completeness):
- `NTP_API` = `4` — current `adjtimex(2)` API version.
- `MOD_OFFSET` / `MOD_FREQUENCY` / `MOD_MAXERROR` / `MOD_ESTERROR` / `MOD_STATUS` / `MOD_TIMECONST` / `MOD_PPSMAX` / `MOD_TAI` / `MOD_MICRO` / `MOD_NANO` / `MOD_CLKB` / `MOD_CLKA` — adjtimex mode bits.
- `STA_PLL` / `STA_PPSFREQ` / `STA_PPSTIME` / `STA_FLL` / `STA_INS` / `STA_DEL` / `STA_UNSYNC` / `STA_FREQHOLD` / `STA_PPSSIGNAL` / `STA_PPSJITTER` / `STA_PPSWANDER` / `STA_PPSERROR` / `STA_CLOCKERR` / `STA_NANO` / `STA_MODE` / `STA_CLK` — adjtimex status bits.
- `TIME_OK` / `TIME_INS` / `TIME_DEL` / `TIME_OOP` / `TIME_WAIT` / `TIME_ERROR` / `TIME_BAD` — adjtimex return states.
- `ADJ_OFFSET_SINGLESHOT` / `ADJ_OFFSET_SS_READ` — legacy single-shot adjustments.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `clock_gettime(2)` | per-clock read | `Time::clock_gettime` |
| `clock_settime(2)` | per-clock write | `Time::clock_settime` |
| `clock_getres(2)` | per-clock resolution | `Time::clock_getres` |
| `clock_nanosleep(2)` | per-clock sleep | `Time::clock_nanosleep` |
| `clock_adjtime(2)` | per-clock NTP adjust | `Time::clock_adjtime` |
| `timer_create(2)` | per-POSIX timer alloc | `Time::timer_create` |
| `timer_settime(2)` | per-POSIX timer arm | `Time::timer_settime` |
| `timer_gettime(2)` | per-POSIX timer query | `Time::timer_gettime` |
| `timer_getoverrun(2)` | per-POSIX overrun count | `Time::timer_getoverrun` |
| `timer_delete(2)` | per-POSIX timer free | `Time::timer_delete` |
| `getitimer(2)` / `setitimer(2)` | per-process itimer | `Time::getitimer` / `setitimer` |
| `gettimeofday(2)` / `settimeofday(2)` | legacy wall-clock | `Time::gettimeofday` / `settimeofday` |
| `adjtimex(2)` | NTP discipline | `Time::adjtimex` |
| `CLOCK_AUX` block | aux clock dynamic config | `Time::aux_register` / `Time::aux_set` |

## Compatibility contract

REQ-1: `struct timespec` ABI (kernel-supplied to userspace):
- `tv_sec`: `__kernel_old_time_t` (32-bit on legacy 32-bit ABI; 64-bit on 64-bit). y2038-unsafe on 32-bit.
- `tv_nsec`: `long`, must be in `[0, 999_999_999]`.

REQ-2: `struct __kernel_timespec` ABI (y2038-safe):
- `tv_sec`: `__kernel_time64_t` (64-bit on every arch).
- `tv_nsec`: `long long`, must be in `[0, 999_999_999]`.

REQ-3: `struct timeval` ABI:
- `tv_sec`: `__kernel_old_time_t`.
- `tv_usec`: `__kernel_suseconds_t`, must be in `[0, 999_999]`.

REQ-4: `struct itimerspec` / `struct itimerval` ABI:
- `it_interval`: period (zero ⟹ one-shot).
- `it_value`: time until next expiry (zero ⟹ disarmed).

REQ-5: `struct timezone` ABI:
- `tz_minuteswest` and `tz_dsttime` retained for ABI compat; per-POSIX they are obsolete and should be NULL on `settimeofday`. Non-NULL `tz` with non-zero `tz_dsttime` is rejected.

REQ-6: `ITIMER_*` selectors:
- `ITIMER_REAL`: counts wall-clock; sends SIGALRM on expiry.
- `ITIMER_VIRTUAL`: counts only user-mode CPU time; sends SIGVTALRM.
- `ITIMER_PROF`: counts user + system CPU time; sends SIGPROF.

REQ-7: `CLOCK_*` selector behavior:
- `CLOCK_REALTIME`: NTP-disciplined wall-clock; settable with `clock_settime` (CAP_SYS_TIME).
- `CLOCK_MONOTONIC`: NTP-slewed monotonic; not settable.
- `CLOCK_PROCESS_CPUTIME_ID`: per-process CPU time. Read-only in `clock_gettime`; "settable" only via dynamic clockid encoded from PID (older convention).
- `CLOCK_THREAD_CPUTIME_ID`: per-thread CPU time. Read-only.
- `CLOCK_MONOTONIC_RAW`: raw hardware monotonic, immune to NTP slew/freq adjust.
- `CLOCK_REALTIME_COARSE` / `CLOCK_MONOTONIC_COARSE`: low-cost, jiffy-resolution; do NOT read hardware.
- `CLOCK_BOOTTIME`: like `CLOCK_MONOTONIC` but ALSO counts time spent suspended.
- `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM`: timer-arm-only clock IDs; wake from suspend; require `CAP_WAKE_ALARM`.
- `CLOCK_SGI_CYCLE`: reserved place-holder; `clock_gettime`/`clock_settime` return `-EINVAL`.
- `CLOCK_TAI`: TAI (UTC + 37s in 2026); leap-second-free. `clock_settime` requires `CAP_SYS_TIME`; TAI offset adjusted via `adjtimex(MOD_TAI)`.

REQ-8: `CLOCK_AUX` block (`[16, 23]`):
- Auxiliary clock IDs dynamically registered by drivers (PTP-aux, multi-domain TSN).
- Up to `MAX_AUX_CLOCKS` (8) simultaneously.
- Each is steered independently of the core timekeeper.
- VDSO support depends on architecture.

REQ-9: `TIMER_ABSTIME` flag:
- `clock_nanosleep(clockid, TIMER_ABSTIME, &abs, NULL)`: sleep until absolute `abs`.
- `timer_settime(tid, TIMER_ABSTIME, &new, &old)`: arm timer against absolute `it_value`.
- Without flag: relative semantics.

REQ-10: NTP_API surface (full set in `linux/timex.h`):
- `adjtimex(2)` reads/writes the NTP discipline state.
- `clock_adjtime(2)` is per-clock adjtimex (for CLOCK_REALTIME / CLOCK_TAI / aux clocks).
- Setting requires CAP_SYS_TIME.

REQ-11: VDSO fast-path:
- `CLOCK_REALTIME`, `CLOCK_MONOTONIC`, `CLOCK_BOOTTIME`, `CLOCK_REALTIME_COARSE`, `CLOCK_MONOTONIC_COARSE`, `CLOCK_TAI` served by VDSO without syscall on supported archs.
- Coarse variants always served by VDSO (read jiffies state directly).

REQ-12: Per-clock_settime CAP_SYS_TIME requirement:
- `CLOCK_REALTIME` / `CLOCK_REALTIME_ALARM` / `CLOCK_TAI` / `CLOCK_AUX_*` require `CAP_SYS_TIME`.
- `CLOCK_MONOTONIC*` / `CLOCK_BOOTTIME` / `CLOCK_*_COARSE` / CPU-time clocks reject `clock_settime` with `-EINVAL`.

REQ-13: Per-`clock_getres`:
- Returns a `timespec` of the minimum tick of the clock.
- COARSE clocks: `CONFIG_HZ` jiffy.
- Hi-res clocks: hardware ns resolution.

## Acceptance Criteria

- [ ] AC-1: `clock_gettime(CLOCK_REALTIME, &ts)` returns wall-clock; `ts.tv_nsec ∈ [0, 999_999_999]`.
- [ ] AC-2: `clock_gettime(CLOCK_MONOTONIC, ...)` is non-decreasing across calls.
- [ ] AC-3: `clock_gettime(CLOCK_SGI_CYCLE, ...)` → EINVAL.
- [ ] AC-4: `clock_settime(CLOCK_REALTIME, ...)` without CAP_SYS_TIME → EPERM.
- [ ] AC-5: `clock_settime(CLOCK_MONOTONIC, ...)` → EINVAL.
- [ ] AC-6: `clock_nanosleep(CLOCK_MONOTONIC, TIMER_ABSTIME, &abs, NULL)` sleeps until `abs`.
- [ ] AC-7: `clock_getres(CLOCK_REALTIME_COARSE)` returns CONFIG_HZ jiffy.
- [ ] AC-8: `setitimer(ITIMER_VIRTUAL, ...)` sends SIGVTALRM on user-CPU expiration.
- [ ] AC-9: `timer_create(CLOCK_BOOTTIME_ALARM, ...)` without CAP_WAKE_ALARM → EPERM.
- [ ] AC-10: `clock_gettime(CLOCK_TAI, ...)` returns REALTIME + current TAI offset.
- [ ] AC-11: `timerfd_settime` with TIMER_ABSTIME against CLOCK_BOOTTIME wakes after suspend resume.
- [ ] AC-12: `adjtimex` without CAP_SYS_TIME but modes==0 succeeds (read-only); with modes!=0 → EPERM.
- [ ] AC-13: `tv_nsec` out of range in `clock_settime`/`timer_settime` → EINVAL.

## Architecture

Rookery surface in `kernel/time/uapi.rs`:

```rust
#[repr(C)]
pub struct Timespec {
    pub tv_sec: i64,    // __kernel_old_time_t on legacy 32-bit; widened in Rookery
    pub tv_nsec: i64,
}

#[repr(C)]
pub struct KernelTimespec64 {
    pub tv_sec: i64,    // y2038-safe everywhere
    pub tv_nsec: i64,
}

#[repr(C)]
pub struct Timeval {
    pub tv_sec: i64,
    pub tv_usec: i64,
}

#[repr(C)]
pub struct Itimerspec {
    pub it_interval: Timespec,
    pub it_value:    Timespec,
}

#[repr(C)]
pub struct Itimerval {
    pub it_interval: Timeval,
    pub it_value:    Timeval,
}

#[repr(C)]
pub struct Timezone {
    pub tz_minuteswest: i32,
    pub tz_dsttime:     i32,
}

#[repr(i32)]
pub enum ClockId {
    Realtime           = 0,
    Monotonic          = 1,
    ProcessCpuTimeId   = 2,
    ThreadCpuTimeId    = 3,
    MonotonicRaw       = 4,
    RealtimeCoarse     = 5,
    MonotonicCoarse    = 6,
    Boottime           = 7,
    RealtimeAlarm      = 8,
    BoottimeAlarm      = 9,
    SgiCycle           = 10,  // reserved; rejected
    Tai                = 11,
    // CLOCK_AUX..CLOCK_AUX_LAST resolved dynamically
}

pub const TIMER_ABSTIME: i32 = 0x01;
pub const MAX_CLOCKS:    i32 = 16;
pub const CLOCK_AUX:     i32 = MAX_CLOCKS;
pub const MAX_AUX_CLOCKS:i32 = 8;
pub const CLOCK_AUX_LAST:i32 = CLOCK_AUX + MAX_AUX_CLOCKS - 1;
```

`Time::clock_gettime(clockid, &mut ts)`:
1. /* Validate clockid */
2. match clockid {
     0..=4 | 7 => read_timekeeper(clockid, &mut ts),
     5 | 6     => read_coarse(clockid, &mut ts),
     8 | 9     => read_alarm_clock(clockid, &mut ts),
     10        => return -EINVAL, /* SGI_CYCLE reserved */
     11        => read_tai(&mut ts),
     CLOCK_AUX..=CLOCK_AUX_LAST => Time::aux_get(clockid, &mut ts),
     _         => return Time::resolve_dynamic_clockid(clockid, &mut ts), /* per-PID CPU clocks */
   }.
3. assert ts.tv_nsec ∈ [0, 999_999_999].
4. return 0.

`Time::clock_settime(clockid, ts)`:
1. /* Validate tv_nsec */
2. if !(0 ≤ ts.tv_nsec < 1_000_000_000): return -EINVAL.
3. /* Per-clock CAP requirement */
4. match clockid {
     0 | 8        => if !ns_capable(CAP_SYS_TIME): return -EPERM; timekeeper_set_realtime(ts),
     1 | 4..=7    => return -EINVAL, /* not settable */
     2 | 3        => return -EINVAL, /* CPU-time read-only */
     9            => if !ns_capable(CAP_WAKE_ALARM): return -EPERM; alarm_clock_set(ts),
     10           => return -EINVAL,
     11           => if !ns_capable(CAP_SYS_TIME): return -EPERM; tai_set(ts),
     CLOCK_AUX..=CLOCK_AUX_LAST => if !ns_capable(CAP_SYS_TIME): return -EPERM; Time::aux_set(clockid, ts),
     _            => return -EINVAL,
   }.
5. return 0.

`Time::clock_nanosleep(clockid, flags, &req, rem)`:
1. abs = (flags & TIMER_ABSTIME) != 0.
2. deadline = if abs { req } else { Time::clock_now(clockid) + req }.
3. /* hrtimer schedule */
4. wait_until(clockid, deadline, &mut interrupted).
5. if interrupted ∧ !abs ∧ rem: write_remaining(rem, deadline - Time::clock_now(clockid)); return -ERESTART_RESTARTBLOCK.
6. return 0.

`Time::setitimer(which, new, old)`:
1. match which { 0 | 1 | 2 => ok, _ => return -EINVAL }.
2. if old: write_itimerval(old, current.itimer[which]).
3. current.itimer[which] = new.
4. arm_itimer_timer(which, new).
5. return 0.

`Time::aux_register(idx) -> ClockId`:
1. if idx ≥ MAX_AUX_CLOCKS: return -EINVAL.
2. ctx = aux_clocks[idx].init().
3. return CLOCK_AUX + idx.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tv_nsec_in_range` | INVARIANT | clock_gettime / settime / nanosleep: tv_nsec ∈ [0, 1e9) |
| `tv_usec_in_range` | INVARIANT | timeval: tv_usec ∈ [0, 1e6) |
| `clockid_in_universe` | INVARIANT | clockid ∈ {0..11} ∪ [CLOCK_AUX, CLOCK_AUX_LAST] ∪ dynamic-CPU-clockid |
| `sgi_cycle_rejected` | INVARIANT | clockid == 10 ⟹ -EINVAL |
| `monotonic_non_decreasing` | INVARIANT | clock_gettime(MONOTONIC) sequence is non-decreasing |
| `realtime_set_requires_cap` | INVARIANT | clock_settime(REALTIME) ⟹ CAP_SYS_TIME |
| `tai_set_requires_cap` | INVARIANT | clock_settime(TAI) ⟹ CAP_SYS_TIME |
| `alarm_create_requires_cap` | INVARIANT | timer_create on *_ALARM ⟹ CAP_WAKE_ALARM |
| `timer_abstime_only_bit` | INVARIANT | flags == 0 ∨ flags == TIMER_ABSTIME |

### Layer 2: TLA+

`uapi/time.tla`:
- Variables: `realtime`, `monotonic`, `boottime`, `tai`, `aux[i]`.
- Properties:
  - `safety_monotonic_no_jump` — monotonic clock never decreases.
  - `safety_boottime_geq_monotonic` — boottime ≥ monotonic ∀ time.
  - `safety_tai_offset_invariant` — tai = realtime + tai_offset.
  - `safety_realtime_jump_visible_to_cancel_on_set` — clock_settime(REALTIME, ...) wakes all CANCEL_ON_SET timerfds.
  - `liveness_clock_nanosleep_completes` — every clock_nanosleep returns at-or-after deadline or with EINTR.
  - `liveness_timer_armed_eventually_fires` — armed POSIX timer with deadline ≤ now eventually delivers SIGEV_*.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `clock_gettime` post: ts is normalized | `Time::clock_gettime` |
| `clock_settime` post: timekeeper.realtime == ts (or aux variant) | `Time::clock_settime` |
| `clock_nanosleep` post: now ≥ deadline ∨ -EINTR | `Time::clock_nanosleep` |
| `clock_getres` post: res ≥ ns(1) | `Time::clock_getres` |
| `setitimer` post: current.itimer[which] reflects new | `Time::setitimer` |
| `aux_register` post: returns id ∈ [CLOCK_AUX, CLOCK_AUX_LAST] | `Time::aux_register` |

### Layer 4: Verus/Creusot functional

Per-POSIX.1-2024 §`<time.h>` clock surface equivalence. Per-LTP `testcases/kernel/syscalls/clock_*` and `glibc` `sysdeps/unix/sysv/linux/clock_*.c` round-trip. Per-`Documentation/timers/timekeeping.rst` and `Documentation/userspace-api/vdso.rst` ABI consistency.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

## Grsecurity/PaX-style Reinforcement

- **GRKERNSEC_CLOCK_RESOLUTION randomization** — coarsen `clock_gettime(CLOCK_MONOTONIC | CLOCK_MONOTONIC_RAW | CLOCK_REALTIME)` to microsecond resolution (or coarse-jiffy) when the calling task is unprivileged and the system has `kernel.unprivileged_clock_precision=0`; defeats high-resolution-timer side channels (FLUSH+RELOAD, branch-predictor probing, Spectre-v2 gadgets) that rely on ns-level rdtsc-equivalent reads from userspace. VDSO fast-path mirrors the policy (no syscall bypass).
- **PAX_RANDKSTACK on time syscall entry** — randomize kernel stack on every `clock_gettime`/`clock_settime`/`clock_nanosleep`/`timer_settime` entry; prevents stack-layout disclosure via long-pending nanosleep timing.
- **CLOCK_TAI requires CAP_SYS_TIME (no namespace loophole)** — `clock_settime(CLOCK_TAI, ...)` and `adjtimex(MOD_TAI, ...)` require `ns_capable(init_user_ns, CAP_SYS_TIME)`; refuse the operation even when the calling task has `CAP_SYS_TIME` in a nested userns. TAI-offset manipulation is host-wide and would otherwise allow a privileged container to drift the host's TAI-OFFSET.
- **CLOCK_REALTIME_ALARM / CLOCK_BOOTTIME_ALARM strict CAP_WAKE_ALARM** — `timer_create`/`timerfd_create` against `*_ALARM` clock IDs require CAP_WAKE_ALARM in the **alarm namespace** (init_user_ns by policy); refuses the elevation-by-container-CAP loophole that lets a container wake the host from suspend.
- **adjtimex steering rate-limited** — limit cumulative `MOD_FREQUENCY` adjustments to a tunable ppm/sec ceiling per uid; prevents a privileged adversary from radically destabilizing the timekeeper to mask covert-channel signaling or to defeat anti-replay nonces.
- **CLOCK_SGI_CYCLE permanently reserved** — return `-EINVAL` unconditionally (no future driver re-use without bumping the kernel ABI epoch); avoids the historical pattern of a stale clockid being silently rebound to a new driver and breaking sandbox audit assumptions.
- **CLOCK_AUX block sandboxed** — auxiliary clock writes restricted to a single "owner" uid + ns recorded at register time; cross-namespace `clock_settime(CLOCK_AUX+N, ...)` returns `-EPERM` even with CAP_SYS_TIME.
- **GRKERNSEC_LOG_TIME_SET** — audit-log every `clock_settime` / `settimeofday` / `adjtimex(MOD_*)` mutation with uid, clockid, and delta; deters covert use of CLOCK_REALTIME jumps to defeat audit timestamping or TOTP.
- **Reject `settimeofday` with non-NULL timezone** — `tz` is obsolete per POSIX; refuse `tz != NULL ∧ tz->tz_dsttime != 0` with `-EINVAL`; defense against legacy ABI being used as a side-channel scratch register.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `kernel/time/timekeeping.c` core timekeeper (covered in `kernel/timekeeping.md` Tier-3)
- `kernel/time/hrtimer.c` high-resolution timer wheel (covered in `kernel/hrtimer.md`)
- `kernel/time/posix-timers.c` POSIX-timer implementation (covered in `kernel/posix-timers.md`)
- `kernel/time/ntp.c` NTP discipline + `adjtimex` modes (covered in `uapi/timex.md`)
- `kernel/time/alarmtimer.c` `*_ALARM` clock backing (covered in `kernel/alarmtimer.md`)
- VDSO clock_gettime fast-path arch wiring (covered in `arch/vdso.md`)
- `Documentation/userspace-api/vdso.rst` ABI listing (cross-referenced; not duplicated)
- Implementation code
