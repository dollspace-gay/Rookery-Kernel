---
title: "Tier-5 syscall: clock_settime(2) — syscall 227"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`clock_settime(2)` mutates the kernel's wall-clock time (and a small subset of related clocks) to the value supplied by userspace, applying a step (not slew) to the timekeeper. It is the privileged complement of `clock_gettime(2)`: only `CLOCK_REALTIME`, `CLOCK_TAI`, and dynamic POSIX clocks accept writes; `CLOCK_MONOTONIC`, `CLOCK_BOOTTIME`, `CLOCK_MONOTONIC_RAW`, and the COARSE variants are read-only and return `-EINVAL`. The monotonic family advances only via NTP slew or suspend-time accumulation, never via direct step. Used by: `ntpd`/`chronyd` on initial sync, `hwclock --systohc-reverse`, container start-up scripts injecting a baseline time, virtualization guests on first boot. Critical for: bringing a freshly-booted system into sync; preserving the monotonic-invariant for everything else.

This Tier-5 covers the userspace ABI of syscall 227. NTP discipline, leap-second handling, and timekeeper write protocol are owned by `kernel/time/timekeeping.md` and `kernel/time/ntp.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: Unprivileged caller (no `CAP_SYS_TIME`) → `-EPERM`.
- [ ] AC-2: `CLOCK_MONOTONIC` write → `-EINVAL`.
- [ ] AC-3: `CLOCK_BOOTTIME` write → `-EINVAL`.
- [ ] AC-4: Valid `CLOCK_REALTIME` set: subsequent `clock_gettime(CLOCK_REALTIME)` returns the set value (±tick).
- [ ] AC-5: Valid `CLOCK_REALTIME` set: `CLOCK_MONOTONIC` does NOT step.
- [ ] AC-6: `tv_nsec == 1_000_000_000` → `-EINVAL`.
- [ ] AC-7: `tp == NULL` → `-EFAULT`.
- [ ] AC-8: `CLOCK_TAI` set: `tai_offset` updated; `CLOCK_TAI − CLOCK_REALTIME` reflects new offset.
- [ ] AC-9: Set under blocked `clock_nanosleep(TIMER_ABSTIME, CLOCK_REALTIME)`: waiter wakes if armed past new time.
- [ ] AC-10: Non-init-userns root (no init-userns CAP_SYS_TIME) → `-EPERM`.
- [ ] AC-11: Audit record emitted on success.

### Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
pub fn sys_clock_settime(clk_id: i32, tp_user: *const __kernel_timespec) -> isize {
    if !capable(CAP_SYS_TIME) { return -EPERM; }
    let ts = copy_from_user::<KernelTimespec>(tp_user).map_err(|_| -EFAULT)?;
    if ts.tv_nsec < 0 || ts.tv_nsec >= 1_000_000_000 { return -EINVAL; }
    PosixTimers::clock_settime(clk_id, ts)
}
```

`PosixTimers::clock_settime(clk_id, ts) -> isize`:
1. /* dynamic clock */
2. if (clk_id as u32 & CLOCKFD_MASK) == CLOCKFD:
   - return PosixClock::dispatch_settime(clk_id, ts).
3. match ClockId::try_from(clk_id) {
   - Realtime => Timekeeper::do_settimeofday64(ts),
   - Tai      => Timekeeper::do_settime_tai(ts),
   - _        => return -EINVAL,
 }

`Timekeeper::do_settimeofday64(ts) -> isize`:
1. /* range check */
2. if !ts_valid_settod(&ts) { return -EINVAL; }
3. /* write-side lock */
4. let flags = raw_spin_lock_irqsave(&tk_core.lock);
5. write_seqcount_begin(&tk_core.seq);
6. /* compute delta */
7. let old = tk_xtime(&tk_core.tk);
8. let delta = timespec64_sub(ts, old);
9. /* apply offset */
10. timekeeping_inject_offset(&mut tk_core.tk, delta);
11. ntp_clear(&mut tk_core.tk);
12. update_pvclock_gtod(&tk_core.tk);
13. update_vsyscall(&tk_core.tk);          // vDSO data page
14. write_seqcount_end(&tk_core.seq);
15. raw_spin_unlock_irqrestore(&tk_core.lock, flags);
16. /* wake cancel-on-set timerfds & abstime clock_nanosleep waiters */
17. clock_was_set();
18. audit_log_settime(current.uid, &ts);
19. return 0;

### Out of Scope

- `kernel/time/timekeeping.md` Tier-3: tk_core write protocol, NTP integration.
- `kernel/time/ntp.md` Tier-3: NTP discipline, leap-second handling, `adjtimex`.
- `clock_gettime.md` / `clock_getres.md` / `clock_nanosleep.md` siblings.
- `settimeofday.md` / `adjtimex.md` siblings.
- `timerfd_settime` cancel-on-set wake protocol.
- glibc / musl userspace `clock_settime` wrappers.
- Implementation code.

### signature

```c
int clock_settime(clockid_t clk_id, const struct timespec *tp);
```

Rust ABI shim:

```rust
pub fn sys_clock_settime(clk_id: i32, tp: *const __kernel_timespec) -> isize;
```

Syscall number: **227**.
On 32-bit architectures the time-64-safe variant is `sys_clock_settime64` (syscall 404).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `clk_id` | `clockid_t` (`i32`) | IN | clock to set (`CLOCK_REALTIME`, `CLOCK_TAI`, or dynamic posix clock) |
| `tp` | `const struct __kernel_timespec *` | IN | target value `{tv_sec: i64, tv_nsec: i64}` |

### return

- **Success**: `0`; kernel timekeeper updated.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `clk_id` not settable (`CLOCK_MONOTONIC`, `_RAW`, `_COARSE`, `BOOTTIME`, CPU clocks); `tv_nsec` ∉ `[0, 999_999_999]`; `tv_sec` negative beyond kernel-supported range |
| `EFAULT` | `tp` not in readable userspace |
| `EPERM` | caller lacks `CAP_SYS_TIME` (or `CAP_WAKE_ALARM` for alarm clocks) |
| `EOPNOTSUPP` | dynamic POSIX clock backend (`/dev/ptp*`) returns ENOTSUP |
| `EACCES` | per-userns: not init userns and not CAP_SYS_TIME-in-init-ns |

### abi surface

Settable `clockid_t` values:

| Constant | Value | Settable | Notes |
|---|---|---|---|
| `CLOCK_REALTIME` | `0` | yes | step wall-clock |
| `CLOCK_TAI` | `11` | yes | step TAI; updates `tai_offset = tai − utc` |
| `CLOCK_MONOTONIC` | `1` | **no** | `-EINVAL` |
| `CLOCK_MONOTONIC_RAW` | `4` | **no** | `-EINVAL` |
| `CLOCK_REALTIME_COARSE` | `5` | **no** | `-EINVAL` |
| `CLOCK_MONOTONIC_COARSE` | `6` | **no** | `-EINVAL` |
| `CLOCK_BOOTTIME` | `7` | **no** | `-EINVAL` |
| `CLOCK_PROCESS_CPUTIME_ID` | `2` | **no** | `-EINVAL` |
| `CLOCK_THREAD_CPUTIME_ID` | `3` | **no** | `-EINVAL` |
| dynamic posix clock (`CLOCKFD` encoded) | varies | maybe | dispatches to backend `posix_clock_ops::clock_settime` |

`struct __kernel_timespec` layout matches `clock_gettime.md`.

### compatibility contract

REQ-1: Capability gate:
- `CAP_SYS_TIME` MUST be present in effective set; else `-EPERM`.
- Per-userns: capability MUST hold in **init user namespace**; in non-init userns, even root-equivalent uid is denied.

REQ-2: `clk_id` validation:
- Only `CLOCK_REALTIME`, `CLOCK_TAI`, and CLOCKFD-encoded dynamic clocks are settable.
- Read-only clocks: `-EINVAL`.

REQ-3: `tp` validation:
- `copy_from_user` failure: `-EFAULT`.
- `tv_nsec` ∉ `[0, 999_999_999]`: `-EINVAL`.
- `tv_sec < KTIME_SEC_MIN` or `tv_sec > KTIME_SEC_MAX`: `-EINVAL`.

REQ-4: `CLOCK_REALTIME` step:
- Computes delta from current `xtime` to requested `tp`.
- Calls `timekeeping_inject_offset(delta)` under `tk_core.lock` write-side.
- Updates `vdso_data` seqcount on completion.
- NTP state cleared (`ntp_clear`): pending kerneluser tick adjustments discarded.

REQ-5: `CLOCK_TAI` step:
- Updates `tk.tai_offset = tp - xtime_real` (UTC delta).
- Does NOT modify `CLOCK_REALTIME` directly.

REQ-6: Notification:
- After successful set: `clock_was_set()` invoked → wakes blocked `clock_nanosleep(TIMER_ABSTIME, CLOCK_REALTIME, ...)` waiters per `TFD_TIMER_CANCEL_ON_SET` semantics on `timerfd_settime`.
- `__timerfd_remove_cancel_list` walks cancel-on-set timerfds and signals them.

REQ-7: Dynamic POSIX clock:
- `clk_id` decoded to `posix_clock`; calls `posix_clock_ops::clock_settime(clk, tp)`.
- Errors propagate (e.g., `-EOPNOTSUPP` for read-only PHCs).

REQ-8: Monotonic-family invariant:
- Setting `CLOCK_REALTIME` MUST NOT step `CLOCK_MONOTONIC`/`_RAW`/`BOOTTIME`.
- `tk.monotonic - tk.real` offset is updated to absorb the step.

REQ-9: Audit:
- All successful `clock_settime` calls audit-logged (syscall + uid + new value).

REQ-10: Signal safety:
- `clock_settime` is async-signal-safe per POSIX; in practice rarely called from signal handlers.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_sys_time_enforced` | INVARIANT | no CAP_SYS_TIME ⟹ -EPERM, no kernel write |
| `monotonic_not_settable` | INVARIANT | settable clocks ∈ {REALTIME, TAI, CLOCKFD} only |
| `tv_nsec_in_range` | INVARIANT | bad `tv_nsec` ⟹ -EINVAL |
| `tk_core_lock_held_during_write` | INVARIANT | inject_offset under tk_core.lock |
| `vdso_data_updated_post_set` | INVARIANT | update_vsyscall called within seqcount writer |

### Layer 2: TLA+

`uapi/clock_settime.tla`:
- Variables: `tk.xtime`, `tk.monotonic`, `tk.tai_offset`, `tk.seq`.
- Properties:
  - `safety_monotonic_invariant_under_set` — `CLOCK_REALTIME` step does not change `tk.monotonic`.
  - `safety_cap_gated` — set-attempt without CAP_SYS_TIME never reaches `inject_offset`.
  - `liveness_cancel_on_set_wakes` — pending TFD_TIMER_CANCEL_ON_SET timers signaled after set.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `clock_settime` post(0): `xtime == ts` (within tick) | `Timekeeper::do_settimeofday64` |
| `clock_settime` post: `monotonic` unchanged | `Timekeeper::do_settimeofday64` |
| `tai` set post: `tai_offset == ts − xtime` | `Timekeeper::do_settime_tai` |
| audit record emitted on success | `audit_log_settime` |

### Layer 4: Verus/Creusot functional

Per-`clock_settime(2)` man page, `Documentation/core-api/timekeeping.rst`, glibc `sysdeps/unix/sysv/linux/clock_settime.c`, `chronyd` source (sys_linux.c). Per-LTP `testcases/kernel/syscalls/clock_settime/*` round-trip.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

clock_settime reinforcement:

- **Per-CAP_SYS_TIME strict** — checked in init userns; non-init-userns calls denied regardless of in-ns root.
- **Per-clock whitelist for write** — only REALTIME, TAI, dynamic CLOCKFD accept writes.
- **Per-monotonic invariant** — write to REALTIME never disturbs MONOTONIC/_RAW/BOOTTIME.
- **Per-vdso_data update under seqcount** — userspace vDSO readers never observe torn time after set.
- **Per-audit log** — every successful set logged for forensic replay.

### grsecurity/pax-style reinforcement

- **CAP_SYS_TIME init-userns gating** — under hardened policy, CAP_SYS_TIME is honored only in the **init** user namespace; container roots with CAP_SYS_TIME on a non-init userns are denied with `-EPERM`. Prevents container escape vectors that successfully obtained CAP_SYS_TIME in a userns from rolling system time to bypass certificate expiry, replay attack detection, or time-locked credentials.
- **PaX UDEREF on `tp` read** — `copy_from_user(&ts, tp)` refuses kernel-mapped addresses disguised as userspace; defense against type-confused write of attacker-controlled kernel pages into the timekeeper.
- **GRKERNSEC_CLOCK_RESOLUTION write-side rate limit** — per-uid rate limit on `clock_settime` (default: 1 call per 1000ms under hardened profile); defense against rapid clock-mutation oracles that adversarial code uses to create timing channels observable from unprivileged tasks via `clock_gettime`.
- **CAP_WAKE_ALARM N/A here** — alarm-class clocks (`CLOCK_REALTIME_ALARM`, `CLOCK_BOOTTIME_ALARM`) are not settable via this syscall; the capability is consumed by the timer creation paths (`timerfd_create`, `timer_create`).
- **adjtimex co-rate-limit** — `adjtimex(2)` (sibling) shares a per-uid token bucket with `clock_settime` so attackers cannot fan-out clock perturbations across both syscalls; both consume the same hardened-policy quota.
- **PAX_RANDKSTACK on every entry** — random kstack on each `clock_settime` denies layout discovery via repeated privileged entries.
- **GRKERNSEC_AUDIT_GRPADD-style audit** — successful `clock_settime` logs `(syscall, uid, gid, pid, old_xtime, new_xtime, delta)` to the audit subsystem; defenders can reconstruct any "time rewind" attack post-incident.
- **Refuse pre-epoch sets except boot-time** — under hardened policy, `tv_sec < 0` (pre-1970) is rejected with `-EINVAL` unless the system is in `system_state == SYSTEM_BOOTING` and the call originates from PID 1; defense against time-rewind attacks that intentionally roll into the pre-epoch range to corrupt comparison logic in cron/at/systemd-timer.
- **Block CLOCK_TAI step > 8s without explicit confirmation** — large TAI steps are atypical (leap-second handling does ±1s only); hardened policy rejects |delta| > 8s on TAI with `-EPERM` plus audit; legitimate operators set TAI via NTP discipline (`adjtimex(ADJ_TAI)`) instead.
- **clock_was_set notification rate-limit** — `clock_was_set()` wakes every timerfd in the cancel-on-set list; under hardened policy this notification is rate-limited per uid to prevent a privileged caller from generating storms that destabilize timerfd consumers (defense-in-depth against scheduler-DoS via cancel-storms).
- **GRKERNSEC_HIDESYM on tk_core address** — `tk_core` global address is excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`; the timekeeper is a high-value target for kernel write primitives.

