# Tier-5 syscall: timerfd_create(2) — syscall 283

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/timerfd.c (sys_timerfd_create, do_timerfd_create)
  - include/uapi/linux/timerfd.h
  - include/uapi/linux/time.h (clockid_t values)
  - arch/*/include/generated/uapi/asm/unistd_64.h (283)
  - Documentation/admin-guide/timerfd.rst, man timerfd_create(2)
-->

## Summary

`timerfd_create(2)` allocates a kernel-resident POSIX-style interval timer and returns a file descriptor that fires the timer's expirations into a 64-bit counter readable via `read(2)`. Companion syscalls `timerfd_settime(2)` (syscall 286) and `timerfd_gettime(2)` (syscall 287) program and inspect the timer. The fd is poll/select/epoll-friendly: `POLLIN` when one or more expirations have occurred since the last read. Supersedes the legacy `signal`-based POSIX timer (`timer_create(2)`) for event-loop integration, eliminating signal-handler-context complexity. Critical for: every modern event-loop deadline (libuv timers, Go time.Ticker, asyncio scheduling, Rust tokio time, Java ScheduledExecutorService backends), wall-clock alarm wakeups (cron-equivalent via `TFD_TIMER_ABSTIME | TFD_TIMER_CANCEL_ON_SET`), suspend-aware alarms (`CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM`).

This Tier-5 covers the userspace ABI of syscall 283 (creation only); `timerfd_settime` / `timerfd_gettime` are sibling syscalls in `uapi/headers/timerfd.md` and `kernel/timerfd.md` (Tier-3, planned).

## Signature

```c
int timerfd_create(int clockid, int flags);
```

Rust ABI shim:

```rust
pub fn sys_timerfd_create(clockid: i32, flags: i32) -> isize;
```

Syscall number: **283**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `clockid` | `i32` | IN | clock source: `CLOCK_REALTIME` / `CLOCK_MONOTONIC` / `CLOCK_BOOTTIME` / `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM` |
| `flags` | `i32` | IN | bitmask: `TFD_CLOEXEC | TFD_NONBLOCK` |

## Return

- **Success**: non-negative file descriptor.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | unsupported `clockid`; unknown `flags` bit |
| `EMFILE` | per-process fd table full |
| `ENFILE` | system-wide fd table full |
| `ENOMEM` | kernel cannot allocate `struct timerfd_ctx` |
| `ENODEV` | anon_inode mount unavailable |
| `EPERM` | `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM` without `CAP_WAKE_ALARM` |

## ABI surface

`clockid_t` values accepted:

| Constant | Value | Purpose |
|---|---|---|
| `CLOCK_REALTIME` | `0` | wall-clock time; jumps on settimeofday |
| `CLOCK_MONOTONIC` | `1` | monotonic since boot; no jumps |
| `CLOCK_BOOTTIME` | `7` | monotonic plus suspend time |
| `CLOCK_REALTIME_ALARM` | `8` | REALTIME plus wakes from suspend (requires `CAP_WAKE_ALARM`) |
| `CLOCK_BOOTTIME_ALARM` | `9` | BOOTTIME plus wakes from suspend (requires `CAP_WAKE_ALARM`) |

Other clockids (`CLOCK_PROCESS_CPUTIME_ID = 2`, `CLOCK_THREAD_CPUTIME_ID = 3`, `CLOCK_MONOTONIC_RAW = 4`, `CLOCK_REALTIME_COARSE = 5`, `CLOCK_MONOTONIC_COARSE = 6`, `CLOCK_TAI = 11`) are **not** valid for `timerfd_create` → `-EINVAL`.

`TFD_*` flags:

| Constant | Value | Purpose |
|---|---|---|
| `TFD_CLOEXEC` | `O_CLOEXEC` (`0x80000`) | close-on-exec |
| `TFD_NONBLOCK` | `O_NONBLOCK` (`0x800`) | non-blocking read |

(Note: `TFD_TIMER_ABSTIME` and `TFD_TIMER_CANCEL_ON_SET` are `timerfd_settime` flags, not creation flags.)

Read frame: 8 bytes (`__u64`), counting expirations since the previous successful read; counter resets to zero on read.

## Compatibility contract

REQ-1: `clockid` validation:
- Accept `{0, 1, 7, 8, 9}` only.
- Any other value → `-EINVAL`.

REQ-2: `flags` validation:
- Subset of `{TFD_CLOEXEC, TFD_NONBLOCK}` else `-EINVAL`.

REQ-3: `CAP_WAKE_ALARM`:
- `CLOCK_REALTIME_ALARM` and `CLOCK_BOOTTIME_ALARM` require `CAP_WAKE_ALARM` in the calling task's effective set; else `-EPERM`.

REQ-4: Returned fd:
- `O_RDONLY`-equivalent (read-only from userspace; kernel-side write of counter only).
- `O_CLOEXEC` set iff `TFD_CLOEXEC`.
- `O_NONBLOCK` set iff `TFD_NONBLOCK`.
- `fcntl(F_GETFL)` reflects.
- `read(2)` returns 8 bytes: count of expirations since last read; if no expirations: block (or `-EAGAIN` if `O_NONBLOCK`).
- `write(2)` → `-EINVAL`.
- Poll: `POLLIN` iff counter > 0.

REQ-5: Initial state:
- Timer disarmed (`it_value == {0,0}`).
- Counter == 0.
- No expirations pending.
- Subsequent `timerfd_settime(2)` programs.

REQ-6: anon_inode:
- Inode is anon_inode `[timerfd]`.
- Visible in `/proc/<pid>/fd/<N>` as `anon_inode:[timerfd]`.
- `/proc/<pid>/fdinfo/<N>` exposes `clockid: <N>`, `ticks: <count>`, `settime flags: <flags>`, `it_value: <ts>`, `it_interval: <ts>`.

REQ-7: Lifetime:
- `close(2)` triggers timer cancellation + ctx free.
- `dup(2)` / `fork(2)`: shared `struct file`; both holders see same timer.
- `execve(2)`: closed iff `O_CLOEXEC`.

REQ-8: Per-process fd accounting:
- Consumes one `RLIMIT_NOFILE` slot.
- Underlying hrtimer / alarmtimer charges no quota beyond standard timer accounting.

REQ-9: Concurrency:
- `timerfd_create` is reentrant; multiple timers per task supported up to `RLIMIT_NOFILE`.

REQ-10: Suspend semantics by clockid:
- `MONOTONIC`: pauses during suspend (effectively); does not wake.
- `BOOTTIME`: counts during suspend; does not wake.
- `REALTIME`: counts wall-clock; does not wake (but is reactive to wall-clock jumps).
- `REALTIME_ALARM` / `BOOTTIME_ALARM`: counts and wakes from suspend.

REQ-11: Wall-clock-jump sensitivity:
- `CLOCK_REALTIME[_ALARM]`: settimeofday / NTP-step affects the absolute deadline; `timerfd_settime` with `TFD_TIMER_CANCEL_ON_SET` (set-time flag) signals cancellation.

## Acceptance Criteria

- [ ] AC-1: `timerfd_create(CLOCK_MONOTONIC, 0)` returns fd ≥ 0; clockid reflected in `/proc/<pid>/fdinfo`.
- [ ] AC-2: `timerfd_create(CLOCK_REALTIME, TFD_CLOEXEC)` returns fd; `FD_CLOEXEC` set.
- [ ] AC-3: `timerfd_create(CLOCK_REALTIME, TFD_CLOEXEC|TFD_NONBLOCK)` returns fd; both flags set.
- [ ] AC-4: `timerfd_create(CLOCK_BOOTTIME, 0)` returns fd.
- [ ] AC-5: `timerfd_create(CLOCK_REALTIME_ALARM, 0)` without `CAP_WAKE_ALARM` → `-EPERM`.
- [ ] AC-6: `timerfd_create(CLOCK_REALTIME_ALARM, 0)` with `CAP_WAKE_ALARM` → fd.
- [ ] AC-7: `timerfd_create(CLOCK_BOOTTIME_ALARM, 0)` without `CAP_WAKE_ALARM` → `-EPERM`.
- [ ] AC-8: `timerfd_create(CLOCK_THREAD_CPUTIME_ID, 0)` → `-EINVAL`.
- [ ] AC-9: `timerfd_create(CLOCK_MONOTONIC, 0x10)` → `-EINVAL`.
- [ ] AC-10: `timerfd_create(CLOCK_MONOTONIC, -1)` → `-EINVAL`.
- [ ] AC-11: `timerfd_create(100, 0)` → `-EINVAL`.
- [ ] AC-12: Created fd: `write(fd, &x, 8)` → `-EINVAL`.
- [ ] AC-13: Created fd: `read(fd, &v, 4)` (count < 8) → `-EINVAL`.
- [ ] AC-14: Created fd with `O_NONBLOCK`: `read(fd)` immediately after create (no `settime`) → `-EAGAIN`.
- [ ] AC-15: After `execve` with `TFD_CLOEXEC`: fd absent.

## Architecture

Rookery surface in `kernel/timerfd/create.rs`:

```rust
#[repr(i32)]
pub enum TimerfdClock {
    Realtime      = 0,
    Monotonic     = 1,
    Boottime      = 7,
    RealtimeAlarm = 8,
    BoottimeAlarm = 9,
}

bitflags! {
    pub struct TimerfdCreateFlags: i32 {
        const CLOEXEC  = 0x80000;
        const NONBLOCK = 0x800;
    }
}

pub struct TimerfdCtx {
    pub clockid:        TimerfdClock,
    pub flags:          TimerfdCreateFlags,
    pub ticks:          AtomicU64,
    pub it_value:       Mutex<KernelTimespec>,
    pub it_interval:    Mutex<KernelTimespec>,
    pub settime_flags:  AtomicI32,
    pub hrtimer:        Mutex<Option<HrTimer>>,
    pub alarmtimer:     Mutex<Option<AlarmTimer>>,
    pub wq:             WaitQueue,
    pub cancel_on_set:  AtomicBool,
}
```

`Timerfd::create(clockid_raw, flags_raw) -> isize`:
1. /* clockid validation */
2. let clockid = match clockid_raw {
     - 0 => TimerfdClock::Realtime,
     - 1 => TimerfdClock::Monotonic,
     - 7 => TimerfdClock::Boottime,
     - 8 => TimerfdClock::RealtimeAlarm,
     - 9 => TimerfdClock::BoottimeAlarm,
     - _ => return -EINVAL,
   };
3. /* flag validation */
4. let f = TimerfdCreateFlags::from_bits(flags_raw).ok_or(-EINVAL)?;
5. /* CAP_WAKE_ALARM */
6. if matches!(clockid, TimerfdClock::RealtimeAlarm | TimerfdClock::BoottimeAlarm) {
     - if !current.has_cap(CAP_WAKE_ALARM) { return -EPERM; }
   }
7. /* allocate ctx */
8. let ctx = Arc::new(TimerfdCtx {
     - clockid,
     - flags: f,
     - ticks: AtomicU64::new(0),
     - it_value: Mutex::new(KernelTimespec::zero()),
     - it_interval: Mutex::new(KernelTimespec::zero()),
     - settime_flags: AtomicI32::new(0),
     - hrtimer: Mutex::new(None),
     - alarmtimer: Mutex::new(None),
     - wq: WaitQueue::new(),
     - cancel_on_set: AtomicBool::new(false),
   });
9. /* open flags translation */
10. let open_flags = O_RDONLY
      | if f.contains(TimerfdCreateFlags::CLOEXEC) { O_CLOEXEC } else { 0 }
      | if f.contains(TimerfdCreateFlags::NONBLOCK) { O_NONBLOCK } else { 0 };
11. let fd = anon_inode_getfd("[timerfd]", &TIMERFD_FOPS, ctx, open_flags)?;
12. return fd as isize;

`TIMERFD_FOPS`:
1. read         ⟹ Timerfd::read (8-byte counter; reset to 0).
2. write        ⟹ -EINVAL.
3. poll         ⟹ Timerfd::poll (POLLIN ⟺ ticks > 0).
4. release      ⟹ cancel hrtimer/alarmtimer; free ctx.
5. show_fdinfo  ⟹ emit clockid/ticks/settime_flags/it_value/it_interval.
6. ioctl        ⟹ TFD_IOC_SET_TICKS (CAP_SYS_ADMIN).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `clockid_subset` | INVARIANT | clockid ∉ {0,1,7,8,9} ⟹ -EINVAL |
| `flags_subset` | INVARIANT | flags & ~{CLOEXEC,NONBLOCK} ⟹ -EINVAL |
| `alarm_requires_cap` | INVARIANT | alarm clockids without CAP_WAKE_ALARM ⟹ -EPERM |
| `cloexec_propagates` | INVARIANT | TFD_CLOEXEC ⟹ FD_CLOEXEC |
| `nonblock_propagates` | INVARIANT | TFD_NONBLOCK ⟹ O_NONBLOCK |
| `initial_disarmed` | INVARIANT | post-create: it_value == 0, ticks == 0 |
| `fd_install_atomic` | INVARIANT | fd visible iff ctx fully initialized |

### Layer 2: TLA+

`uapi/timerfd_create.tla`:
- States: `validating`, `cap_check`, `allocating`, `installing_fd`, `returned`, `failed`.
- Properties:
  - `safety_clockid_supported` — accepted iff in supported set.
  - `safety_alarm_capability` — alarm clockids ⟹ CAP_WAKE_ALARM held.
  - `safety_no_partial_install` — fd visible iff ctx fully initialized.
  - `liveness_terminate` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `create` post(ok): `0 ≤ ret ∧ ctx.clockid == requested ∧ ctx.ticks == 0` | `Timerfd::create` |
| `create` post: ctx.flags == requested flags | `Timerfd::create` |
| `create` post: ctx.hrtimer == None ∧ ctx.alarmtimer == None | `Timerfd::create` |

### Layer 4: Verus/Creusot functional

Per-`timerfd_create(2)` man page, `Documentation/admin-guide/timerfd.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/timerfd/*.c` and glibc `sysdeps/unix/sysv/linux/timerfd_create.c` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

timerfd_create reinforcement:

- **CLOEXEC default under restrictive policy** — defense against per-exec leak of timer fd to setuid child.
- **`CAP_WAKE_ALARM` strictly enforced** — defense against per-unprivileged suspend-wake DoS.
- **Clockid strict subset** — defense against per-clockid-confusion exploits (e.g., requesting CPUTIME on timerfd).
- **anon_inode source isolation** — defense against per-namespace cross-leak.
- **Per-user-namespace open count limit (sysctl)** — defense against per-user fd exhaustion.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on timerfd_create entry** — randomize kernel stack; timerfd_create is low-frequency but establishes a long-lived kernel timer object, and grsec doctrine randomizes the entry that allocates the persistent ctx.
- **PaX UDEREF not directly applicable** — the syscall takes only scalars. UDEREF is inherited from generic syscall entry.
- **GRKERNSEC_BPF_HARDEN considerations** — timerfd is not a VM but is a kernel-resident scheduled-callback resource. The same audit doctrine applies for the alarm variants (`CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM`) which wake the system from suspend: log every alarm-timerfd creation with `current->comm`, `pid`, clockid.
- **CAP_BPF gating not appropriate** — timerfd is a baseline event-loop primitive. However alarm variants are gated by `CAP_WAKE_ALARM` (canonical) which under hardened policy is upgraded to require `CAP_BPF` OR `CAP_WAKE_ALARM` (allowing only the explicit alarm-capability holders or BPF-trusted admins).
- **GRKERNSEC_FIFO scaled to timerfd flood (parallel to eventfd flood)** — per-uid quota on simultaneous open timerfds and per-uid rate-limit on `timerfd_create`; refuses with `-EMFILE` past quota. Defense against per-user timer-table exhaustion. Particularly important for alarm timerfds which consume RTC programming slots and can prevent the system from suspending.
- **TFD_CLOEXEC mandatory under restrictive policy** — when calling task has `PR_SET_NO_NEW_PRIVS=1`, runs under crosslink seccomp lockdown, or invoked across setuid boundary, refuse `timerfd_create` without `TFD_CLOEXEC`; eliminates a cross-exec covert channel where parent leaks a programmed timer to a less-privileged child.
- **GRKERNSEC_HIDESYM on `/proc/<pid>/fdinfo/<N>` timerfd entries** — `it_value` / `it_interval` reveal scheduling state across the calling task's fd namespace; under `kernel.kptr_restrict ≥ 2`, fdinfo restricted to `CAP_SYS_ADMIN` or same-uid + same-pidns observers. This is particularly important because the absolute `it_value` of a `CLOCK_REALTIME` timer can be correlated with kernel internals.
- **Alarm-timerfd per-uid hard cap** — under hardened policy, cap the number of armed `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM` timerfds per uid; refuse creation past cap with `-EPERM` and audit-log. Defense against suspend-prevention DoS via alarm flood.
- **Audit log on every alarm timerfd creation** — alarm-class clockids are sensitive (wake suspended system); each creation logged at `LOGLEVEL_INFO` with clockid + comm + pid.
- **Cross-namespace alarm refusal** — under hardened policy, refuse `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM` from a task in a non-init user_ns even with `CAP_WAKE_ALARM` capability set; suspend-wake is a system-global resource.
- **Settime_flags audit for `TFD_TIMER_ABSTIME` past-due** — absolute deadlines in the past trigger immediate expiration; pattern of "create + settime-past-due + read" is characteristic of timing-side-channel probes; audit at `LOGLEVEL_INFO`.
- **Clockid strict whitelist enforced even with privileged caller** — even `CAP_SYS_ADMIN` cannot pass `CLOCK_THREAD_CPUTIME_ID` etc. to timerfd_create; closes per-clockid-confusion class.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `timerfd_settime(2)` (syscall 286) — programming the timer
- `timerfd_gettime(2)` (syscall 287) — inspecting the timer
- `uapi/headers/timerfd.md` Tier-5 — header-level read/poll/settime semantics
- `kernel/timerfd.md` Tier-3 — hrtimer / alarmtimer integration, suspend handling
- `kernel/time/alarmtimer.md` Tier-3 — RTC-backed alarm path
- Implementation code
