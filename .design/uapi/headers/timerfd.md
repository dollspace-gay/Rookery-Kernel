# Tier-5 UAPI: include/uapi/linux/timerfd.h — timerfd(2) ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/timerfd.h (~37 lines)
  - fs/timerfd.c
  - Documentation/timers/timers-howto.rst
-->

## Summary

`timerfd_create(2)` returns a file descriptor that becomes readable when a POSIX timer (`CLOCK_REALTIME` / `CLOCK_MONOTONIC` / `CLOCK_BOOTTIME` / `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM`) expires, enabling `poll`/`select`/`epoll`-based timer multiplexing without signals. `timerfd_settime(2)` arms the timer (relative or `TFD_TIMER_ABSTIME`), optionally requesting `TFD_TIMER_CANCEL_ON_SET` so that any future discontinuous realtime-clock jump invalidates the armed timer (subsequent `read()` returns `ECANCELED`). `read(fd, &u64, 8)` returns the count of expirations since the last successful read. `TFD_IOC_SET_TICKS` ioctl injects a tick count (test-only). Critical for: event-loop timers, watchdogs, sleep-by-fd, alarm-clock daemons.

This Tier-5 covers `include/uapi/linux/timerfd.h` (~37 lines).

## ABI surface

| Constant / Type | Value | Purpose |
|---|---|---|
| `TFD_TIMER_ABSTIME` | `1 << 0` | settime: interpret it_value as absolute |
| `TFD_TIMER_CANCEL_ON_SET` | `1 << 1` | cancel timer on discontinuous RT-clock change |
| `TFD_CLOEXEC` | `O_CLOEXEC` (0x80000) | timerfd_create flag: O_CLOEXEC |
| `TFD_NONBLOCK` | `O_NONBLOCK` (0x800) | timerfd_create flag: O_NONBLOCK |
| `TFD_IOC_SET_TICKS` | `_IOW('T', 0, __u64)` | ioctl: force tick count (debug/test) |
| (syscall) `timerfd_create(int clockid, int flags)` | nr varies | create timer fd |
| (syscall) `timerfd_settime(int fd, int flags, const itimerspec *new, itimerspec *old)` | nr varies | arm/disarm |
| (syscall) `timerfd_gettime(int fd, itimerspec *cur)` | nr varies | read remaining |

Permitted clockids:
- `CLOCK_REALTIME` (0).
- `CLOCK_MONOTONIC` (1).
- `CLOCK_BOOTTIME` (7).
- `CLOCK_REALTIME_ALARM` (8) — requires `CAP_WAKE_ALARM`.
- `CLOCK_BOOTTIME_ALARM` (9) — requires `CAP_WAKE_ALARM`.

Read frame: `__u64 expirations` (host-endian, 8 bytes). EAGAIN on NONBLOCK if 0. ECANCELED if `TFD_TIMER_CANCEL_ON_SET` and RT-clock was set.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `timerfd_create(2)` | per-fd allocation | `Timerfd::create` |
| `timerfd_settime(2)` | per-arm | `Timerfd::settime` |
| `timerfd_gettime(2)` | per-query | `Timerfd::gettime` |
| `TFD_IOC_SET_TICKS` | per-test inject | `Timerfd::ioctl_set_ticks` |
| `TFD_TIMER_ABSTIME` | per-arm-mode | `TimerfdSetFlags::ABSTIME` |
| `TFD_TIMER_CANCEL_ON_SET` | per-RT-cancel | `TimerfdSetFlags::CANCEL_ON_SET` |

## Compatibility contract

REQ-1: `timerfd_create(clockid, flags)`:
- `clockid` ∈ { CLOCK_REALTIME, CLOCK_MONOTONIC, CLOCK_BOOTTIME, CLOCK_REALTIME_ALARM, CLOCK_BOOTTIME_ALARM } else EINVAL.
- `flags` MUST be subset of { TFD_CLOEXEC, TFD_NONBLOCK } else EINVAL.
- `CLOCK_*_ALARM` requires `CAP_WAKE_ALARM`; else EPERM.
- Returns fd ≥ 0 or -1/errno.

REQ-2: `timerfd_settime(fd, flags, new, old)`:
- `flags` MUST be subset of { TFD_TIMER_ABSTIME, TFD_TIMER_CANCEL_ON_SET } else EINVAL.
- `TFD_TIMER_CANCEL_ON_SET` valid only with `CLOCK_REALTIME` or `CLOCK_REALTIME_ALARM` (else EINVAL).
- `new->it_value == 0` disarms; otherwise arms.
- `new->it_interval` non-zero programs periodic; zero programs one-shot.
- Per-`old` (if non-NULL): previous setting written.
- Per-tv_nsec validation: `0 ≤ tv_nsec < 1_000_000_000` else EINVAL.

REQ-3: `timerfd_gettime(fd, cur)`:
- Writes remaining `it_value` (time until next expiry) and `it_interval`.
- Returns 0 or -1/errno.

REQ-4: `read(fd, buf, count)`:
- `count ≥ 8` else EINVAL.
- Blocks until ≥1 expiration unless O_NONBLOCK (then EAGAIN).
- Writes `__u64` count of expirations since last successful read; resets internal counter.
- ECANCELED if armed with `TFD_TIMER_CANCEL_ON_SET` and RT-clock was discontinuously set.

REQ-5: `poll(fd, POLLIN)` becomes readable iff expirations ≥ 1 (or pending cancellation).

REQ-6: `TFD_IOC_SET_TICKS` injects a u64 expiration count (test path); writes to userspace pointer required, kernel reads it. Used by selftests; not enabled in production unless `CONFIG_TIMERFD_TEST`.

REQ-7: Per-fork: timerfd inherited via O_CLOEXEC discipline; not duplicated as a separately-armed timer.

REQ-8: Per-suspend (BOOTTIME / *_ALARM): wakes system when armed against `_ALARM` clock IDs.

## Acceptance Criteria

- [ ] AC-1: `timerfd_create(CLOCK_MONOTONIC, TFD_CLOEXEC|TFD_NONBLOCK)` returns fd.
- [ ] AC-2: Invalid clockid → EINVAL.
- [ ] AC-3: `CLOCK_REALTIME_ALARM` without `CAP_WAKE_ALARM` → EPERM.
- [ ] AC-4: `timerfd_settime` with `TFD_TIMER_CANCEL_ON_SET` and CLOCK_MONOTONIC → EINVAL.
- [ ] AC-5: One-shot timer: after expiry, single read returns 1; further reads block / EAGAIN.
- [ ] AC-6: Periodic timer (it_interval=100ms): after 1s, read returns ~10.
- [ ] AC-7: `TFD_TIMER_ABSTIME` arms against absolute clock value.
- [ ] AC-8: `read` with count < 8 → EINVAL.
- [ ] AC-9: `timerfd_gettime` returns remaining + interval.
- [ ] AC-10: `clock_settime(CLOCK_REALTIME, +∞)` on CANCEL_ON_SET timer → next read ECANCELED.
- [ ] AC-11: `TFD_IOC_SET_TICKS` (test build) injects tick count.

## Architecture

Rookery surface in `kernel/timerfd/uapi.rs`:

```rust
bitflags! {
    pub struct TimerfdCreateFlags: i32 {
        const CLOEXEC  = 0x80000;
        const NONBLOCK = 0x800;
    }
    pub struct TimerfdSetFlags: i32 {
        const ABSTIME       = 1 << 0;
        const CANCEL_ON_SET = 1 << 1;
    }
}

#[repr(C)]
pub struct Itimerspec {
    pub it_interval: Timespec,
    pub it_value:    Timespec,
}
```

`Timerfd::create(clockid, flags)`:
1. /* Validate clockid */
2. if clockid ∉ allowed_set: return -EINVAL.
3. /* Validate flags */
4. if flags & !(TFD_CLOEXEC | TFD_NONBLOCK): return -EINVAL.
5. /* Check CAP_WAKE_ALARM for *_ALARM */
6. if clockid ∈ {CLOCK_REALTIME_ALARM, CLOCK_BOOTTIME_ALARM} ∧ !ns_capable(CAP_WAKE_ALARM): return -EPERM.
7. fd = anon_inode_getfd("timerfd", &Timerfd::fops, ctx, flags_to_open(flags)).
8. return fd.

`Timerfd::settime(fd, flags, new, old)`:
1. /* Validate flags */
2. if flags & !(TFD_TIMER_ABSTIME | TFD_TIMER_CANCEL_ON_SET): return -EINVAL.
3. /* CANCEL_ON_SET only on RT */
4. if (flags & CANCEL_ON_SET) ∧ ctx.clockid ∉ {CLOCK_REALTIME, CLOCK_REALTIME_ALARM}: return -EINVAL.
5. /* Save old */
6. if old: write_itimerspec(old, ctx.timer.current()).
7. /* Disarm if zero */
8. if new.it_value.is_zero(): ctx.timer.cancel(); return 0.
9. /* Arm */
10. ctx.timer.start(new, abstime = flags & ABSTIME, cancel_on_set = flags & CANCEL_ON_SET).
11. return 0.

`Timerfd::read(file, buf, count)`:
1. if count < 8: return -EINVAL.
2. /* Block / EAGAIN */
3. n = ctx.expirations.fetch_or_block(file.nonblock).
4. if ctx.cancelled.load(): return -ECANCELED.
5. copy_to_user(buf, &n, 8).
6. return 8.

`Timerfd::on_realtime_set()`:
1. for each open timerfd with cancel_on_set ∧ clockid==CLOCK_REALTIME*:
   - ctx.cancelled.store(true).
   - ctx.wq.wake_all().

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `create_clockid_in_allowed_set` | INVARIANT | clockid ∈ {0,1,7,8,9} only |
| `alarm_clock_needs_cap` | INVARIANT | *_ALARM ⟹ CAP_WAKE_ALARM |
| `cancel_on_set_only_realtime` | INVARIANT | CANCEL_ON_SET ⟹ clockid ∈ RT family |
| `read_count_ge_8` | INVARIANT | read len < 8 ⟹ EINVAL |
| `nsec_in_range` | INVARIANT | new.tv_nsec ∈ [0, 1e9) |
| `settime_flags_bits_valid` | INVARIANT | flags ⊆ {ABSTIME, CANCEL_ON_SET} |
| `create_flags_bits_valid` | INVARIANT | flags ⊆ {CLOEXEC, NONBLOCK} |

### Layer 2: TLA+

`uapi/timerfd.tla`:
- States: NEW → ARMED → (EXPIRED | CANCELLED) → READ → (ARMED | DISARMED) → CLOSED.
- Properties:
  - `safety_periodic_increments` — periodic timer: expirations strictly monotone.
  - `safety_cancel_on_realtime_jump` — CANCEL_ON_SET + RT-jump ⟹ read returns ECANCELED.
  - `liveness_armed_eventually_readable` — armed ∧ deadline ≤ now ⟹ poll(POLLIN) returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `create` post: fd ≥ 0 ∧ ctx.clockid set | `Timerfd::create` |
| `settime` post: ctx.timer reflects new ∨ disarmed | `Timerfd::settime` |
| `read` post: returns expiration count, resets counter | `Timerfd::read` |
| `on_realtime_set` post: all CANCEL_ON_SET marked cancelled | `Timerfd::on_realtime_set` |

### Layer 4: Verus/Creusot functional

Per-`timerfd_create(2)` man page semantic equivalence. Per-LTP `testcases/kernel/syscalls/timerfd/*` conformance. Per-glibc `sysdeps/unix/sysv/linux/timerfd_*.c` wrapper round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

## Grsecurity/PaX-style Reinforcement

- **GRKERNSEC_CLOCK_RESOLUTION** — coarsen `timerfd_gettime` resolution on `CLOCK_REALTIME`/`MONOTONIC` for tasks executing a suid/sgid binary, denying microarchitectural-timing side channels (e.g. Spectre-v2 / FLUSH+RELOAD-style attacks against the privileged binary).
- **CAP_WAKE_ALARM strict for *_ALARM** — Rookery enforces `ns_capable(CAP_WAKE_ALARM)` for `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM` in **every** `timerfd_create` call, including those issued under a user-namespace with capability remapping; default-deny on container hosts.
- **TFD_CLOEXEC mandatory under grsec policy** — refuse `timerfd_create` without `TFD_CLOEXEC` when the calling task has `PR_SET_NO_NEW_PRIVS` set or runs under a tight crosslink seccomp filter; closes the inherit-into-exec primitive that lets a child process schedule wake-ups it never created.
- **PAX_RANDKSTACK on timerfd entry** — randomize kernel stack on `timerfd_settime` and `read` paths, preventing stack-layout disclosure via long-pending timer states.
- **Bounded armed-timer quota per uid** — refuse with `-EMFILE` past per-uid count cap to prevent timer-flood DoS (each armed timer consumes hrtimer slots).
- **GRKERNSEC_LOG_TIMERFD_CANCEL_ON_SET** — audit-log every `TFD_TIMER_CANCEL_ON_SET` arm + matching ECANCELED read; deters covert-channel use of RT-clock-jumps as cross-uid signaling.
- **TFD_IOC_SET_TICKS gated to CAP_SYS_ADMIN** — test ioctl restricted; rejected with `-EPERM` outside test builds.
- **Reject suid binary creating CLOCK_REALTIME timerfd with CANCEL_ON_SET in seccomp-locked sandbox** — denies a hypothetical privilege-escalation primitive that observes a discontinuous RT-clock set by `ntpd`.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `fs/timerfd.c` hrtimer integration (covered in `kernel/timerfd-impl.md` Tier-3)
- `kernel/time/alarmtimer.c` *_ALARM clock backing (covered in `kernel/alarmtimer.md`)
- `clock_settime(2)` RT-clock-set machinery (covered in `uapi/time.md`)
- `epoll`/`poll`/`select` integration (covered in `uapi/headers/poll.md` if expanded)
- Implementation code
