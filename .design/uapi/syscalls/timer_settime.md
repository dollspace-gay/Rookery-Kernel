# Tier-5 syscall: timer_settime(2) — syscall 223

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/posix-timers.c (sys_timer_settime, do_timer_settime, common_timer_set)
  - kernel/time/alarmtimer.c (alarm_timer_set)
  - kernel/time/hrtimer.c (hrtimer_start_range_ns)
  - include/uapi/linux/time.h (struct __kernel_itimerspec)
  - arch/*/include/generated/uapi/asm/unistd_64.h (223)
  - Documentation/core-api/timekeeping.rst, man timer_settime(2)
-->

## Summary

`timer_settime(2)` programs (arms, re-arms, or disarms) a POSIX per-process timer previously created by `timer_create(2)`, transitioning it from disarmed to armed state with an initial expiry (`it_value`) and an optional periodic reload (`it_interval`). Setting both fields to zero disarms the timer. The `flags` argument selects between **relative** (default: `it_value` interpreted as offset from now) and **absolute** (`TIMER_ABSTIME`: `it_value` interpreted as wall-time on the timer's clock). On success, the previous arming state is returned via `old_value`, enabling read-modify-write idioms. Critical for: periodic deadlines (real-time control loops, telemetry tick), one-shot deadlines (timeouts with cancellation via re-arm with zero), absolute-time alarms (cron-style "fire at 03:00 UTC"), suspend-aware wakeups (alarm-class clocks armed under `CAP_WAKE_ALARM`).

This Tier-5 covers the userspace ABI of syscall 223. Per-`hrtimer` backend and per-`alarmtimer` backend are owned by `kernel/time/hrtimer.md` and `kernel/time/alarmtimer.md` (Tier-3, planned).

## Signature

```c
int timer_settime(timer_t timerid,
                  int flags,
                  const struct itimerspec *new_value,
                  struct itimerspec *old_value);
```

Rust ABI shim:

```rust
pub fn sys_timer_settime(timerid: TimerId,
                         flags: i32,
                         new_value: *const __kernel_itimerspec,
                         old_value: *mut __kernel_itimerspec) -> isize;
```

Syscall number: **223**.
On 32-bit time-safe variant: `sys_timer_settime64` (syscall 409).

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `timerid` | `timer_t` | IN | timer handle from `timer_create` |
| `flags` | `int` | IN | `0` or `TIMER_ABSTIME` (1) |
| `new_value` | `const struct __kernel_itimerspec *` | IN | `{it_interval, it_value}` |
| `old_value` | `struct __kernel_itimerspec *` | OUT | previous arming state; may be `NULL` |

## Return

- **Success**: `0`; timer reprogrammed; `*old_value` populated if non-NULL.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `timerid` does not name a live timer in current process; `new_value->it_value.tv_nsec` or `it_interval.tv_nsec` ∉ `[0, 999_999_999]`; negative `tv_sec`; `flags` has unknown bits |
| `EFAULT` | `new_value` not readable or `old_value` not writable |

## ABI surface

`struct __kernel_itimerspec`:

```c
struct __kernel_itimerspec {
    struct __kernel_timespec it_interval;  /* reload period; zero = one-shot */
    struct __kernel_timespec it_value;     /* initial expiry */
};
```

`flags` values:

| Flag | Meaning |
|---|---|
| `0` | `it_value` is relative offset from now |
| `TIMER_ABSTIME` (1) | `it_value` is absolute time on timer's clock |

## Compatibility contract

REQ-1: `timerid` lookup:
- `posix_timer_get_locked(current.signal, timerid)` MUST find a live `k_itimer` whose `it_signal == current.signal`; else `-EINVAL`.
- Lookup taken under `current->sighand->siglock`.

REQ-2: `new_value` validation:
- `copy_from_user` failure: `-EFAULT`.
- `it_value.tv_nsec` ∉ `[0, 999_999_999]`: `-EINVAL`.
- `it_interval.tv_nsec` ∉ `[0, 999_999_999]`: `-EINVAL`.
- Negative `tv_sec`: `-EINVAL`.

REQ-3: Disarm semantics:
- `it_value.tv_sec == 0 ∧ it_value.tv_nsec == 0` ⟹ timer transitioned to disarmed; pending hrtimer cancelled; any in-flight overrun count cleared on next `timer_settime` cycle.

REQ-4: `flags` validation:
- Only `TIMER_ABSTIME` recognized; other bits ⟹ `-EINVAL`.

REQ-5: Absolute mode:
- `flags & TIMER_ABSTIME`: `it_value` is wall-time on timer's clock.
- If `it_value` is in the past, timer fires immediately (POSIX-mandated).
- For `CLOCK_REALTIME`/`CLOCK_TAI`-backed timers, the kernel re-evaluates when wall clock is stepped (see REQ-9).

REQ-6: Relative mode:
- `it_value` added to `ktime_get_clock(it_clock)`; absolute expiry recorded.
- Past values fire immediately (POSIX).

REQ-7: Old-value capture:
- Before reprogramming, kernel reads current `(it_interval, remaining_to_fire)` into a stack `itimerspec`.
- `remaining_to_fire` = `expires - now` (clamped at 0).
- Disarmed timer reports zero remaining and zero interval.

REQ-8: Periodic reload:
- `it_interval != 0`: after each expiry, kernel re-arms with `expires + N * it_interval` where N is smallest making `expires > now` (overrun handling).
- `it_interval == 0`: one-shot; after firing, timer remains in IDR but disarmed.

REQ-9: Clock-step interaction:
- For `CLOCK_REALTIME`/`CLOCK_TAI` ABSTIME timers: `clock_was_set()` propagates to per-timer requeue logic so they wake on the new wall-time basis.
- `CLOCK_MONOTONIC` / `_BOOTTIME` not affected by `clock_settime`.

REQ-10: Alarm-class:
- ALARM-class clocks dispatched to `alarm_timer_set`; uses `struct alarm` (RTC-backed wakeup-source).
- Suspend-time expiry counted as fires; consumer observes via `timer_getoverrun`.

REQ-11: Overrun:
- Re-arm fires that occur while previous fire is undelivered increment `it_overrun_last` (reported by `timer_getoverrun`).
- `timer_settime` itself does NOT clear `it_overrun` on the same call; the next signal delivery does.

REQ-12: Userspace write of old_value:
- Performed AFTER kernel commit of new arming; if `copy_to_user` fails, the new arming stands but `-EFAULT` is returned (lossy, matches POSIX behavior).

## Acceptance Criteria

- [ ] AC-1: Arm with relative `it_value = 1s`, `it_interval = 0`: timer fires once ~1s later.
- [ ] AC-2: Disarm: `it_value == 0`; subsequent `timer_gettime` returns zero remaining.
- [ ] AC-3: `TIMER_ABSTIME` past: timer fires immediately.
- [ ] AC-4: Periodic: `it_interval = 100ms`; over 1s, ~10 fires occur.
- [ ] AC-5: `tv_nsec = 10^9` ⟹ `-EINVAL`.
- [ ] AC-6: `flags = 2` ⟹ `-EINVAL`.
- [ ] AC-7: `timerid` belonging to another process ⟹ `-EINVAL`.
- [ ] AC-8: `old_value` populated with previous arming state; NULL accepted.
- [ ] AC-9: `clock_settime(REALTIME, future)`: ABSTIME REALTIME timer requeues to new basis.
- [ ] AC-10: `CLOCK_MONOTONIC` ABSTIME timer unaffected by `clock_settime`.
- [ ] AC-11: ALARM-class timer armed under suspend wakes the system via RTC.

## Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
pub fn sys_timer_settime(id: TimerId,
                         flags: i32,
                         new: *const KernelItimerspec,
                         old: *mut KernelItimerspec) -> isize {
    if flags & !TIMER_ABSTIME != 0 { return -EINVAL; }
    let nv = match copy_from_user::<KernelItimerspec>(new) {
        Ok(v) => v, Err(_) => return -EFAULT,
    };
    if !itimerspec_valid(&nv) { return -EINVAL; }
    let mut prev = KernelItimerspec::default();
    let rc = PosixTimers::do_timer_set(id, flags, &nv, &mut prev);
    if rc != 0 { return rc; }
    if !old.is_null() && copy_to_user(old, &prev).is_err() { return -EFAULT; }
    0
}
```

`PosixTimers::do_timer_set(id, flags, new, old) -> isize`:
1. let (k, guard) = posix_timer_get_locked(&current.signal, id).ok_or(-EINVAL)?;
2. /* snapshot old state */
3. let kc = &posix_clock_kclocks[k.it_clock as usize];
4. kc.timer_get(&k, old);
5. /* program new */
6. let rc = kc.timer_set(&mut k, flags, new);
7. /* release siglock */
8. drop(guard);
9. rc

`kc.timer_set` for hrtimer-backed clocks (`common_timer_set`):
1. /* cancel pending */
2. hrtimer_try_to_cancel(&k.it.real.timer).
3. /* disarm fast path */
4. if new.it_value == 0 {
   - k.it_interval = 0;
   - k.it_active = false;
   - return 0;
 }
5. /* compute expires */
6. let now = ktime_get_clock(k.it_clock);
7. let expires = if flags & TIMER_ABSTIME != 0 {
   - ktime_from_timespec(new.it_value)
 } else {
   - ktime_add(now, ktime_from_timespec(new.it_value))
 };
8. k.it_interval = ktime_from_timespec(new.it_interval);
9. k.it_active = true;
10. hrtimer_start_range_ns(&k.it.real.timer, expires, 0,
       if flags & TIMER_ABSTIME != 0 { HRTIMER_MODE_ABS } else { HRTIMER_MODE_REL });
11. 0

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `udref_on_new_old_pointers` | INVARIANT | both pointers verified by access_ok |
| `itimerspec_validated` | INVARIANT | tv_nsec ∈ [0, 1e9) for value & interval |
| `flags_whitelisted` | INVARIANT | only TIMER_ABSTIME accepted |
| `siglock_during_lookup` | INVARIANT | timer lookup under sighand siglock |
| `hrtimer_cancel_before_reprogram` | INVARIANT | old hrtimer cancelled before new arming |

### Layer 2: TLA+

`uapi/timer_settime.tla`:
- Variables: `timer_state[id]`, `clock[clkid]`, `expires[id]`.
- Properties:
  - `safety_disarm_idempotent` — disarm of disarmed timer is a no-op.
  - `safety_abstime_past_fires_immediately` — past abstime ⟹ next-tick fire.
  - `safety_overrun_counted` — missed reload firings increment overrun.
  - `liveness_armed_eventually_fires` — armed timer with future expiry fires.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_timer_set` post(0): hrtimer expires matches new value | `common_timer_set` |
| `do_timer_set` post: old populated with prior arming | `posix_timer_get` |
| disarm post: hrtimer cancelled; it_active == false | `common_timer_set` |
| abstime past post: fire scheduled immediately | `common_timer_set` |

### Layer 4: Verus/Creusot functional

Per-`timer_settime(2)` man page, glibc `sysdeps/unix/sysv/linux/timer_settime.c`, LTP `testcases/kernel/syscalls/timer_settime/*`, real-world callers (real-time control loops, `chronyd`).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

timer_settime reinforcement:

- **Lookup under siglock** — timer table never read without lock; no TOCTOU on timer_id.
- **Flags whitelist** — only TIMER_ABSTIME accepted.
- **hrtimer cancel before reprogram** — no double-arm race.
- **Validation before commit** — bad nsec/sec rejected without touching kernel state.
- **Old-state snapshot before mutation** — read-modify-write idiom safe.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF on `new_value` and `old_value`** — both copies use access_ok plus checked transfers; defense against attacker substituting a kernel-mapped address to read/write privileged data via the itimerspec marshaling path.
- **PAX_RANDKSTACK on every entry** — each call randomizes kstack layout; relevant because timer-set paths cross hrtimer/alarmtimer subsystems and randomization frustrates ROP discovery.
- **GRKERNSEC_HIDESYM on `posix_clock_kclocks`** — the per-clock dispatch table is `kallsyms`-hidden under `kptr_restrict ≥ 2`; defense against rop-gadget targeting of the indirect `kc.timer_set` call.
- **Per-uid timer_settime rate limit** — hardened policy installs a token-bucket (default: 10k arms/sec/uid); defense against unprivileged attackers programming millions of expirations to exhaust the hrtimer wheel.
- **Refuse non-monotonic interval below 100ns** — `it_interval < 100ns` (excluding zero) is rejected with `-EINVAL` under hardened policy; defense against runaway-reload denial-of-service where attacker arms a 1ns periodic timer to spin the CPU on signal delivery.
- **CAP_WAKE_ALARM re-validation on arm** — alarm-class timer may have been created when caller had CAP_WAKE_ALARM; hardened policy re-checks the capability at `timer_settime` time so loss of capability mid-life disarms the wakeup path.
- **Audit on each arm** — successful `timer_settime` logs `(uid, pid, timer_id, flags, it_value, it_interval)` to the audit subsystem; defenders can detect storm-arming attempts.
- **Block ABSTIME beyond 100-year horizon** — `it_value.tv_sec > now + 100*365*86400` rejected with `-EINVAL` under hardened policy; defense against integer-overflow attacks against per-clock expires-math in hrtimer wheel comparisons.
- **`clock_was_set` quota share** — alarm-class arming triggers a `clock_was_set`-class notification on REALTIME; hardened policy throttles per-uid to prevent cancel-on-set storms.
- **Refuse arming for foreign-process timer_id even if valid in IDR** — `it_signal != current.signal` returns `-EINVAL` BEFORE consulting hrtimer state; defense against TOCTOU where signal struct is swapped underneath an in-flight call.
- **GRKERNSEC_POSIX_TIMER_INTEGRITY** — k_itimer is allocated from a hardened slab with `SLAB_TYPESAFE_BY_RCU` plus an inline canary; defense against UAF where attacker frees and re-allocates a `k_itimer` to corrupt `it_sigev_signo` mid-call.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `timer_gettime.md`, `timer_getoverrun.md`, `timer_delete.md`, `timer_create.md` siblings.
- `kernel/time/hrtimer.md` Tier-3: hrtimer wheel.
- `kernel/time/alarmtimer.md` Tier-3: alarm-class wakeup-source.
- `kernel/time/posix_timers.md` Tier-3: k_itimer lifecycle, IDR.
- `clock_was_set` notification protocol.
- glibc / musl userspace wrappers.
- Implementation code.
