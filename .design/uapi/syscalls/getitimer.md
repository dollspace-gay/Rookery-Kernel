# Tier-5 syscall: getitimer(2) — syscall 36

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/itimer.c (sys_getitimer, do_getitimer)
  - kernel/time/posix-cpu-timers.c (cpu_clock_sample_group — ITIMER_VIRTUAL/PROF source)
  - kernel/time/hrtimer.c (hrtimer_get_remaining — ITIMER_REAL source)
  - include/uapi/linux/time.h (struct itimerval, ITIMER_*)
  - arch/*/include/generated/uapi/asm/unistd_64.h (36)
  - Documentation/core-api/timekeeping.rst, man getitimer(2)
-->

## Summary

`getitimer(2)` reads the current value (residual time until next expiry plus configured interval) of one of three per-process **interval timers**: `ITIMER_REAL` (wall-clock; delivers `SIGALRM`), `ITIMER_VIRTUAL` (user-mode CPU time; delivers `SIGVTALRM`), `ITIMER_PROF` (user + kernel CPU time; delivers `SIGPROF`). These are the original UNIX V7 interval-timer interface, predating POSIX `timer_create(2)` by decades. They remain useful for: shell `ulimit -t`-style CPU-time soft limits, gprof-style profiling (`ITIMER_PROF`), simple periodic signal-driven alarms (though `timerfd` is preferred in new code). Per-process scope: only one timer of each kind exists per process; modifications are racy across threads unless serialized by the caller. Critical for: profiling tools (`gprof`, `gcov`), simple periodic-signal C code, source compatibility with POSIX.1 and SVID.

This Tier-5 covers the userspace ABI of syscall 36. The hrtimer-backed `ITIMER_REAL` armament and posix-cpu-timer `ITIMER_VIRTUAL`/`PROF` accounting are owned by `kernel/time/itimer.md` (Tier-3, planned).

## Signature

```c
int getitimer(int which, struct itimerval *curr_value);
```

Rust ABI shim:

```rust
pub fn sys_getitimer(which: i32, curr_value: *mut __kernel_old_itimerval) -> isize;
```

Syscall number: **36**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `which` | `i32` | IN | `ITIMER_REAL` (0), `ITIMER_VIRTUAL` (1), or `ITIMER_PROF` (2) |
| `curr_value` | `struct itimerval *` | OUT | current state: `it_value` (residual until next expiry) + `it_interval` (reload interval after expiry) |

## Return

- **Success**: `0`; `*curr_value` populated.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `which` not in `{ITIMER_REAL, ITIMER_VIRTUAL, ITIMER_PROF}` |
| `EFAULT` | `curr_value` not in writable userspace |

## ABI surface

`ITIMER_*` values:

| Constant | Value | Source clock | Signal on expiry |
|---|---|---|---|
| `ITIMER_REAL` | `0` | wall-clock (hrtimer on `CLOCK_MONOTONIC` historically; `CLOCK_REALTIME` semantics for delivery) | `SIGALRM` |
| `ITIMER_VIRTUAL` | `1` | per-process user CPU time | `SIGVTALRM` |
| `ITIMER_PROF` | `2` | per-process user + kernel CPU time | `SIGPROF` |

`struct itimerval`:

```c
struct itimerval {
    struct timeval it_interval;  /* reload value after expiry; {0,0} = one-shot */
    struct timeval it_value;     /* residual until next expiry; {0,0} = disarmed */
};
```

`struct timeval` matches `gettimeofday.md`.

## Compatibility contract

REQ-1: `which` validation:
- MUST be in `{0, 1, 2}` else `-EINVAL`.

REQ-2: `curr_value` writeback:
- `copy_to_user(curr_value, &val)` failure: `-EFAULT`.

REQ-3: `ITIMER_REAL` semantics:
- `it_value`: residual derived from `hrtimer_get_remaining(&task->signal->real_timer)`.
- `it_interval`: stored `task->signal->it_real_incr`.
- Disarmed: `it_value == {0, 0}`.

REQ-4: `ITIMER_VIRTUAL` semantics:
- `it_value`: derived from `task->signal->it[CPUCLOCK_VIRT].expires - thread_group_cputime(...).utime`, clamped ≥ 0.
- `it_interval`: stored `task->signal->it[CPUCLOCK_VIRT].incr`.

REQ-5: `ITIMER_PROF` semantics:
- `it_value`: derived from `task->signal->it[CPUCLOCK_PROF].expires - thread_group_cputime(...).utime+stime`, clamped ≥ 0.
- `it_interval`: stored `task->signal->it[CPUCLOCK_PROF].incr`.

REQ-6: Truncation:
- All values in microseconds (`tv_usec ∈ [0, 999_999]`); nanosecond residual rounded down.

REQ-7: Disarmed timer:
- Returns `{it_interval: {0,0}, it_value: {0,0}}`.

REQ-8: Per-process scope:
- All threads share the same itimer state via `task->signal->...`.
- Read is atomic w.r.t. expiry (uses signal->siglock or RCU-protected snapshot).

REQ-9: Signal safety:
- `getitimer` is async-signal-safe per POSIX.1-2008.

REQ-10: Thread safety:
- Concurrent `getitimer` from multiple threads sees a consistent snapshot per call.

## Acceptance Criteria

- [ ] AC-1: Disarmed timer: `it_value == {0,0}` and `it_interval == {0,0}`.
- [ ] AC-2: After `setitimer(ITIMER_REAL, {{1,0}, {5,0}}, NULL)`: `getitimer` returns `it_value ≈ {5,0}` and `it_interval == {1,0}`.
- [ ] AC-3: `which == 3`: `-EINVAL`.
- [ ] AC-4: `curr_value == NULL`: `-EFAULT`.
- [ ] AC-5: `ITIMER_VIRTUAL`: residual decreases under user-mode CPU burn.
- [ ] AC-6: `ITIMER_PROF`: residual decreases under user-mode AND kernel-mode CPU burn.
- [ ] AC-7: `ITIMER_REAL` after one-shot expiry: `it_value == {0,0}`.
- [ ] AC-8: `ITIMER_VIRTUAL` survives `fork` in parent but is reset in child.
- [ ] AC-9: `it_value.tv_usec ∈ [0, 999_999]`.
- [ ] AC-10: Per-thread `getitimer` returns process-wide value (shared signal).

## Architecture

Rookery surface in `kernel/time/itimer.rs`:

```rust
#[repr(C)]
pub struct OldItimerval {
    pub it_interval: OldTimeval,
    pub it_value:    OldTimeval,
}

pub const ITIMER_REAL:    i32 = 0;
pub const ITIMER_VIRTUAL: i32 = 1;
pub const ITIMER_PROF:    i32 = 2;

pub fn sys_getitimer(which: i32, cv_user: *mut OldItimerval) -> isize {
    let val = match which {
        ITIMER_REAL    => Itimer::get_real(current),
        ITIMER_VIRTUAL => Itimer::get_cpu(current, CPUCLOCK_VIRT),
        ITIMER_PROF    => Itimer::get_cpu(current, CPUCLOCK_PROF),
        _ => return -EINVAL,
    };
    copy_to_user(cv_user, &val).map_err(|_| -EFAULT)?;
    0
}
```

`Itimer::get_real(task) -> OldItimerval`:
1. let sig = &task.signal;
2. let _g = sig.siglock.lock();
3. let interval = sig.it_real_incr.to_timeval();
4. let value = if hrtimer_active(&sig.real_timer) {
   - hrtimer_get_remaining(&sig.real_timer).to_timeval()
   - } else { OldTimeval::ZERO };
5. OldItimerval { it_interval: interval, it_value: value }

`Itimer::get_cpu(task, which) -> OldItimerval`:
1. let sig = &task.signal;
2. let _g = sig.siglock.lock();
3. let cpu = thread_group_cputime(task);   // sum across threads
4. let consumed = match which { VIRT => cpu.utime, PROF => cpu.utime + cpu.stime, _ => unreachable!() };
5. let it = &sig.it[which];
6. let value = if it.expires > 0 {
   - (it.expires - consumed).max(0).to_timeval()
   - } else { OldTimeval::ZERO };
7. let interval = it.incr.to_timeval();
8. OldItimerval { it_interval: interval, it_value: value }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `which_validated` | INVARIANT | which ∉ {0,1,2} ⟹ -EINVAL |
| `efault_on_bad_user_ptr` | INVARIANT | bad `curr_value` ⟹ -EFAULT, no kernel-write to non-user page |
| `siglock_held_for_read` | INVARIANT | sig.siglock held during read |
| `value_nonnegative` | INVARIANT | clamped residual ≥ 0 |
| `tv_usec_in_range` | INVARIANT | output `tv_usec ∈ [0, 999_999]` |

### Layer 2: TLA+

`uapi/getitimer.tla`:
- Variables: `sig.real_timer.expires`, `sig.it[VIRT/PROF].expires`, `sig.it_*.incr`.
- Properties:
  - `safety_disarmed_returns_zero` — disarmed timer reports {0,0}.
  - `safety_snapshot_consistent` — `it_value` and `it_interval` come from same locked snapshot.
  - `liveness_returns` — terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `getitimer` post(0): if armed, `it_value ≥ {0,0}` and ≤ `it_interval` (when periodic) | `Itimer::get_real`/`get_cpu` |
| `getitimer` post: snapshot under siglock | `Itimer::get_*` |
| `getitimer` post: `tv_usec ∈ [0, 999_999]` | `OldTimeval::from` |

### Layer 4: Verus/Creusot functional

Per-`getitimer(2)` man page, glibc `sysdeps/unix/sysv/linux/getitimer.c`, musl `src/time/getitimer.c`. Per-LTP `testcases/kernel/syscalls/getitimer/*` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

getitimer reinforcement:

- **Per-which whitelist** — defense against undefined backend access via out-of-range value.
- **Per-siglock-held read** — defense against torn-snapshot return racing with `setitimer`.
- **Per-EFAULT clean** — bad user pointer never leaks kernel value.
- **Per-value clamped ≥ 0** — defense against post-expiry negative residual confusing userspace.
- **Per-process-wide scope honored** — defense against cross-thread inconsistency.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on every entry** — `getitimer` is in profiling hot-paths; randomize kstack per entry.
- **PaX UDEREF on `curr_value` writeback** — `copy_to_user(cv_user, &val)` refuses kernel-mapped addresses; the write of `it_value` and `it_interval` is a kernel-write primitive of CPU-time-derived data into attacker-controlled memory.
- **GRKERNSEC_CLOCK_RESOLUTION** — under hardened policy, residual `it_value` reported by `getitimer` for suid binaries is coarsened to `1/HZ` microseconds; defense against using `ITIMER_VIRTUAL`/`ITIMER_PROF` residual + tight loop as a CPU-time-burn timing oracle. The leak class is "how much CPU time did this task burn since arming?" with microsecond precision; under hardened policy this is rounded to one tick.
- **CAP_SYS_TIME N/A** — read-only, no clock-mutation. No capability gating beyond inheritance from `signal->siglock` (which is per-process, not capability-checked).
- **CAP_WAKE_ALARM N/A** — itimers do not arm wakelocks.
- **adjtimex rate-limit irrelevant** — `ITIMER_REAL` uses hrtimer on `CLOCK_MONOTONIC`-like advance (not affected by clock_settime). NTP slew via adjtimex affects `CLOCK_MONOTONIC` but the slewing rate is bounded.
- **GRKERNSEC_PROC_USERGROUP scope honored** — `getitimer` operates only on the calling task's `signal->`; cross-process inspection not possible via this syscall.
- **GRKERNSEC_HIDESYM on `signal->it[*]` array address** — process signal struct address excluded from kallsyms under `kptr_restrict ≥ 2`.
- **Refuse `ITIMER_VIRTUAL`/`PROF` from non-thread-group-leader context** — under hardened policy, threads that are NOT the thread-group leader may read but not mutate itimers (mutation reserved for the leader); this read remains permitted but is audit-logged when called from a thread that is not the TGL, to detect lateral-movement reconnaissance patterns.
- **Audit storm detection** — repeated `getitimer` calls at >1000 Hz from a single uid against any one `which` value: audit-logged as potential CPU-time-side-channel measurement.
- **Per-tv_usec range strict** — output `tv_usec` saturated to `[0, 999_999]` under all paths; defense-in-depth against arithmetic-overflow producing `tv_usec == 1_000_000` that downstream parsers misinterpret.
- **getitimer + setitimer co-rate-limit** — together with `setitimer`, per-uid token-bucket-limited under hardened policy to defeat itimer-flood attempts that destabilize the signal-delivery path.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `kernel/time/itimer.md` Tier-3: itimer state machine, signal delivery integration.
- `kernel/time/hrtimer.md` Tier-3: hrtimer subsystem.
- `kernel/time/posix_cpu_timers.md` Tier-3: CPU-time accounting.
- `setitimer.md` sibling — write path.
- `alarm.md` sibling — `ITIMER_REAL` convenience wrapper.
- `timer_create(2)` family — modern POSIX interval timers.
- `timerfd_create(2)` — fd-based interval timer.
- glibc / musl userspace `getitimer` wrappers.
- Implementation code.
