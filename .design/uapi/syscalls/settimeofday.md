# Tier-5 syscall: settimeofday(2) — syscall 164

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/time.c (sys_settimeofday, do_sys_settimeofday64)
  - kernel/time/timekeeping.c (do_settimeofday64, timekeeping_inject_offset)
  - kernel/time/vsyscall.c (update_vsyscall)
  - include/uapi/linux/time.h (struct timeval, struct timezone)
  - arch/*/include/generated/uapi/asm/unistd_64.h (164)
  - Documentation/core-api/timekeeping.rst, man settimeofday(2)
-->

## Summary

`settimeofday(2)` is the legacy BSD/POSIX wall-clock write interface: sets `CLOCK_REALTIME` from a microsecond-resolution `struct timeval` and (optionally) the global kernel `sys_tz`. It has been **functionally superseded** by `clock_settime(CLOCK_REALTIME, ...)` (nanosecond resolution, no tz baggage) but persists for source-compat with BSD utilities, `date -s`, and the same legacy chain that pinned `gettimeofday` into perpetuity. Required capability is `CAP_SYS_TIME`. Unlike `gettimeofday`, there is **no vDSO fastpath** — every set call traps into the kernel, takes the timekeeper write lock, and notifies the cancel-on-set timerfd list. The `tz` argument is functionally the only modern reason to call `settimeofday` instead of `clock_settime`: a `settimeofday(NULL, &tz)` form mutates the kernel `sys_tz` global without touching the clock. Critical for: `ntpdate` (legacy), `hwclock --systohc-reverse`, `date -s "..."`, container start-up scripts, virtualization guest bring-up.

This Tier-5 covers the userspace ABI of syscall 164. The timekeeper write protocol, cancel-on-set notification, and NTP-state clearing are owned by `kernel/time/timekeeping.md` (Tier-3, planned).

## Signature

```c
int settimeofday(const struct timeval *tv, const struct timezone *tz);
```

Rust ABI shim:

```rust
pub fn sys_settimeofday(tv: *const __kernel_old_timeval,
                        tz: *const __kernel_old_timezone) -> isize;
```

Syscall number: **164**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `tv` | `const struct timeval *` | IN | new wall-clock value (microsecond resolution); may be NULL to skip clock set |
| `tz` | `const struct timezone *` | IN | new `sys_tz`; may be NULL to skip tz update |

If both `tv` and `tz` are NULL, the call is a no-op success.

## Return

- **Success**: `0`; kernel timekeeper / `sys_tz` updated as requested.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `tv != NULL` and `tv_usec ∉ [0, 999_999]`; `tv_sec` outside kernel-supported range; `tz_minuteswest` outside `[-1440, 1440]` (hardened policy) |
| `EFAULT` | `tv != NULL` and not readable; `tz != NULL` and not readable |
| `EPERM` | caller lacks `CAP_SYS_TIME` (in init userns) |
| `EACCES` | non-init-userns root (under hardened userns policy) |

## ABI surface

`struct timeval`:

```c
struct timeval {
    __kernel_old_time_t tv_sec;
    __kernel_suseconds_t tv_usec;
};
```

`struct timezone`:

```c
struct timezone {
    int tz_minuteswest;  /* minutes west of UTC; range typically [-1440, +1440] */
    int tz_dsttime;      /* obsolete DST algorithm code; almost always 0 */
};
```

Per `gettimeofday.md`, time-truncation is microsecond resolution; for nanosecond writes use `clock_settime`.

## Compatibility contract

REQ-1: Capability gate:
- `CAP_SYS_TIME` MUST be in effective set in init userns; else `-EPERM`.
- Non-init-userns with root: `-EPERM` under hardened userns policy.

REQ-2: Both NULL:
- `settimeofday(NULL, NULL)`: returns `0`, no-op.

REQ-3: `tv != NULL`:
- `copy_from_user(&tv, tv_user)`: failure → `-EFAULT`.
- Validate: `tv_usec ∈ [0, 999_999]`; else `-EINVAL`.
- Validate: `tv_sec` within `KTIME_SEC_MIN..KTIME_SEC_MAX`; else `-EINVAL`.
- Convert to `__kernel_timespec`: `tv_sec = tv.tv_sec, tv_nsec = tv.tv_usec * 1000`.
- Call `do_settimeofday64(&ts)` → updates `xtime`, clears NTP state.

REQ-4: `tz != NULL`:
- `copy_from_user(&tz, tz_user)`: failure → `-EFAULT`.
- Update global `sys_tz` under spinlock.
- Update `vdso_data.tz_minuteswest` and `vdso_data.tz_dsttime` under seqcount write.

REQ-5: One-shot warp:
- Per upstream historical bug-fix: if `tv != NULL` AND this is the **first** `settimeofday` AND `sys_tz.tz_minuteswest != 0`, the kernel applies a one-time "warp" subtracting `tz_minuteswest * 60` seconds from `tv` to correct for hwclock-in-localtime systems.
- Subsequent calls do NOT warp.

REQ-6: Notification:
- After successful `tv` update: `clock_was_set()` wakes blocked `clock_nanosleep(TIMER_ABSTIME, CLOCK_REALTIME)` waiters and cancels `TFD_TIMER_CANCEL_ON_SET` timerfds.

REQ-7: NTP state:
- `do_settimeofday64` invokes `ntp_clear`: pending tick adjustments discarded.

REQ-8: VDSO update:
- `update_vsyscall(&tk)` invoked under seqcount writer to publish new `xtime` to the vDSO data page.

REQ-9: Monotonic invariant:
- `CLOCK_MONOTONIC`, `_RAW`, `BOOTTIME` unchanged across `settimeofday`.
- Internal offset (`tk.monotonic - tk.real`) absorbs the wall-clock step.

REQ-10: Audit:
- Successful `settimeofday` logs `(syscall, uid, old_xtime, new_xtime, delta)` to audit.

REQ-11: Y2038:
- 32-bit `time_t`: callers passing post-2038 values trigger `-EINVAL`.
- 64-bit `time_t`: safe.

REQ-12: Signal safety:
- Async-signal-safe per POSIX (rarely needed from a signal handler).

## Acceptance Criteria

- [ ] AC-1: Unprivileged caller → `-EPERM`.
- [ ] AC-2: Both args NULL → `0`, no-op.
- [ ] AC-3: Valid `tv`, NULL `tz`: subsequent `gettimeofday` returns set value (±1µs).
- [ ] AC-4: Valid `tv`: `CLOCK_MONOTONIC` does NOT step.
- [ ] AC-5: NULL `tv`, valid `tz`: clock unchanged, `sys_tz` updated.
- [ ] AC-6: `tv.tv_usec == 1_000_000` → `-EINVAL`.
- [ ] AC-7: `tv` not readable → `-EFAULT`.
- [ ] AC-8: After set: `clock_nanosleep(TIMER_ABSTIME, CLOCK_REALTIME)` waiter past new deadline wakes.
- [ ] AC-9: After set: `TFD_TIMER_CANCEL_ON_SET` timerfd consumers see cancel.
- [ ] AC-10: Audit record emitted on success.
- [ ] AC-11: Non-init-userns root (no init-userns CAP_SYS_TIME) → `-EPERM`.
- [ ] AC-12: First-call hwclock-in-localtime warp applied correctly.

## Architecture

Rookery surface in `kernel/time/time.rs`:

```rust
pub fn sys_settimeofday(tv_user: *const OldTimeval, tz_user: *const OldTimezone) -> isize {
    if !capable(CAP_SYS_TIME) { return -EPERM; }
    let tv_opt = if !tv_user.is_null() {
        Some(copy_from_user::<OldTimeval>(tv_user).map_err(|_| -EFAULT)?)
    } else { None };
    let tz_opt = if !tz_user.is_null() {
        Some(copy_from_user::<OldTimezone>(tz_user).map_err(|_| -EFAULT)?)
    } else { None };
    if let Some(ref tz) = tz_opt {
        TimeKeeper::set_sys_tz(tz)?;
    }
    if let Some(ref tv) = tv_opt {
        if tv.tv_usec < 0 || tv.tv_usec >= 1_000_000 { return -EINVAL; }
        let ts = KernelTimespec { tv_sec: tv.tv_sec, tv_nsec: tv.tv_usec * 1000 };
        Timekeeper::do_settimeofday64(&ts)?;
        audit_log_settime(current.uid, &ts);
    }
    0
}
```

`TimeKeeper::set_sys_tz(tz) -> isize`:
1. /* validate */
2. if tz.tz_minuteswest < -24*60 || tz.tz_minuteswest > 24*60 { return -EINVAL; }
3. /* lock */
4. let mut g = SYS_TZ.lock();
5. *g = *tz;
6. /* vDSO publish */
7. let flags = raw_spin_lock_irqsave(&tk_core.lock);
8. write_seqcount_begin(&tk_core.seq);
9. vdso_data.tz_minuteswest = tz.tz_minuteswest;
10. vdso_data.tz_dsttime    = tz.tz_dsttime;
11. write_seqcount_end(&tk_core.seq);
12. raw_spin_unlock_irqrestore(&tk_core.lock, flags);
13. /* one-shot warp prep (consumed by next tv set) */
14. mark_warp_pending_if_first_ever(tz);
15. return 0;

`Timekeeper::do_settimeofday64` per `clock_settime.md` Architecture section (shared implementation).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_sys_time_enforced` | INVARIANT | no CAP_SYS_TIME ⟹ -EPERM, no kernel write |
| `tv_usec_validated` | INVARIANT | bad `tv_usec` ⟹ -EINVAL |
| `null_args_noop` | INVARIANT | both NULL ⟹ 0, no state mutation |
| `tk_core_lock_held` | INVARIANT | `do_settimeofday64` under tk_core.lock |
| `monotonic_invariant` | INVARIANT | CLOCK_MONOTONIC unchanged across set |
| `vdso_update_post_set` | INVARIANT | update_vsyscall called after tv set |

### Layer 2: TLA+

`uapi/settimeofday.tla`:
- Variables: `tk.xtime`, `tk.monotonic`, `sys_tz`, `vdso_data.tz_minuteswest`.
- Properties:
  - `safety_cap_gated` — set without CAP_SYS_TIME never reaches `inject_offset`.
  - `safety_monotonic_invariant` — `CLOCK_MONOTONIC` unchanged.
  - `safety_sys_tz_atomically_updated` — readers never observe torn `sys_tz`.
  - `liveness_cancel_on_set_wakes` — pending TFD_TIMER_CANCEL_ON_SET notified post-set.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `settimeofday` post(0) with tv: `xtime == tv * 1000` (within tick) | `Timekeeper::do_settimeofday64` |
| `settimeofday` post: `monotonic` unchanged | `Timekeeper::do_settimeofday64` |
| `settimeofday` post with tz: `sys_tz == tz`, `vdso_data.tz_*` == tz | `TimeKeeper::set_sys_tz` |
| audit emitted on success | `audit_log_settime` |

### Layer 4: Verus/Creusot functional

Per-`settimeofday(2)` man page, `Documentation/core-api/timekeeping.rst`, glibc `sysdeps/unix/sysv/linux/settimeofday.c`, `util-linux/hwclock.c`. Per-LTP `testcases/kernel/syscalls/settimeofday/*` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

settimeofday reinforcement:

- **Per-CAP_SYS_TIME strict, init-userns only** — defense against per-container clock-rewind escape.
- **Per-tv_usec normalization invariant** — defense against denormalized timestamp poisoning timekeeper.
- **Per-monotonic invariant under wall-clock set** — defense against breakage of every `CLOCK_MONOTONIC` consumer.
- **Per-NULL semantics safe** — both NULL is no-op; not a covert mutation.
- **Per-audit log** — every successful set logged.

## Grsecurity/PaX-style Reinforcement

- **CAP_SYS_TIME init-userns gating** — under hardened policy, CAP_SYS_TIME honored only in init userns; container roots with CAP_SYS_TIME on non-init userns denied with `-EPERM`. Defense against container escape vectors that rewind system time to bypass certificate expiry, replay-detection, or time-locked credentials.
- **PaX UDEREF on `tv` and `tz` read** — `copy_from_user(&tv, tv_user)` and `copy_from_user(&tz, tz_user)` refuse kernel-mapped addresses disguised as userspace; defense against type-confused read of attacker-controlled kernel pages into the timekeeper.
- **PAX_RANDKSTACK on every entry** — random kstack on each `settimeofday` denies layout discovery via repeated privileged entries.
- **GRKERNSEC_CLOCK_RESOLUTION write-side rate limit** — per-uid rate limit on `settimeofday` (default: 1 call per 1000ms under hardened profile); defense against rapid clock-mutation oracles that adversarial code uses to create timing channels observable from unprivileged tasks via `gettimeofday`. Shares a token bucket with `clock_settime` and `adjtimex`.
- **CAP_WAKE_ALARM N/A** — `settimeofday` does not arm alarms; sibling capability untouched.
- **adjtimex co-rate-limit** — `adjtimex(2)` shares a per-uid token bucket with `settimeofday` so attackers cannot fan-out clock perturbations across both syscalls.
- **GRKERNSEC_AUDIT-style log** — successful `settimeofday` logs `(syscall, uid, gid, pid, old_xtime, new_xtime, delta, old_tz, new_tz)` to audit.
- **Refuse pre-epoch sets except boot-time** — under hardened policy, `tv_sec < 0` (pre-1970) is rejected with `-EINVAL` unless system is in `SYSTEM_BOOTING` and called from PID 1.
- **Refuse large jumps without explicit confirmation** — `|new_xtime - old_xtime| > 86400 * 365` (one year) refused with `-EPERM` plus audit; legitimate operators use NTP discipline (`adjtimex(ADJ_STEP)`) for graduated correction.
- **`tz_minuteswest` range hardening** — outside `[-24*60, +24*60]` rejected with `-EINVAL`; defense against `sys_tz` corruption with absurd values that confuse downstream callers reading `gettimeofday(NULL, &tz)`.
- **clock_was_set notification rate-limit** — wakes every timerfd in the cancel-on-set list; rate-limited per uid to prevent destabilizing timerfd consumers via cancel-storms.
- **One-shot warp audit-logged** — the historical hwclock-in-localtime warp is a one-time correction; if observed multiple times audit-log a warning (indicates state corruption or attacker resetting the "first call" flag).
- **GRKERNSEC_HIDESYM on `sys_tz` address** — `sys_tz` global address excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `kernel/time/timekeeping.md` Tier-3: tk_core write protocol, NTP integration.
- `kernel/time/ntp.md` Tier-3: NTP discipline, leap-second handling, `adjtimex`.
- `gettimeofday.md` sibling — read path.
- `clock_settime.md` / `clock_gettime.md` siblings — modern interface.
- `adjtimex.md` — fine-grained NTP discipline.
- `time(2)` syscall — seconds-only read variant.
- glibc / musl userspace `settimeofday` wrappers.
- Implementation code.
