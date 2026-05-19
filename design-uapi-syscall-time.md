---
title: "Tier-5 syscall: time(2) — syscall 201"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`time(2)` returns the **number of seconds since the Epoch** (1970-01-01 00:00:00 UTC) as a `time_t`. It is the oldest POSIX time API and is functionally equivalent to `clock_gettime(CLOCK_REALTIME, &ts) ; return ts.tv_sec;`, but with a half-millennium of legacy code that still calls it directly. The optional `time_t *t` argument, if non-NULL, also receives the value via `put_user`. The syscall is **second-granularity only**: any sub-second jitter that callers would observe from `clock_gettime` is invisible here. On 32-bit architectures the return value is a 32-bit signed integer, leading to the well-known **Y2038 overflow** at 2038-01-19 03:14:07 UTC; on 64-bit kernels `time_t` is 64-bit and the syscall is Y2038-safe. Critical for: legacy shell-out programs that parse `date +%s` equivalent, daemons logging coarse timestamps, RNG-seeding code (`srand(time(NULL))` antipattern), `make` mtime comparisons.

This Tier-5 covers the userspace ABI of syscall 201. The wall-clock timekeeper is owned by `kernel/time/timekeeping.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: `time(NULL)` returns a positive `time_t` close to wallclock-now (within ±5 s of `date +%s`).
- [ ] AC-2: `time(&tt)` writes the same value to `tt` and returns it.
- [ ] AC-3: `time((time_t *)0x1)` (unmapped): returns `-1` with `errno == EFAULT`.
- [ ] AC-4: Two consecutive `time(NULL)` calls within the same wall-second: return identical values.
- [ ] AC-5: After `clock_settime(CLOCK_REALTIME, &new_ts)` advancing the clock by 60 s: subsequent `time(NULL)` returns 60 s higher.
- [ ] AC-6: After `clock_settime` setting the clock backward by 60 s: subsequent `time(NULL)` returns 60 s lower (wall clock can step backward).
- [ ] AC-7: `time(NULL)` after NTP slew (small `adjtimex`): returns smoothly slewed value, not stepped.
- [ ] AC-8: vDSO path is functionally identical to syscall path.
- [ ] AC-9: 64-bit kernel `time(NULL)`: returned value matches first 32 bits of `clock_gettime(CLOCK_REALTIME, ...).tv_sec` (low 32 bits in common range).
- [ ] AC-10: 32-bit kernel `time(NULL)` after 2038-01-19 03:14:07: returns negative `time_t` (Y2038 sign bit).

### Architecture

Rookery surface in `kernel/time/syscalls.rs`:

```rust
pub fn sys_time(t: Option<UserPtrMut<Time_t>>) -> isize {
    let now = Timekeeping::get_real_seconds();
    if let Some(p) = t {
        if p.copy_to_user(&now).is_err() {
            return -EFAULT as isize;
        }
    }
    now as isize
}
```

`Timekeeping::get_real_seconds()` reads the wall-clock component:

```rust
pub fn get_real_seconds() -> i64 {
    let tk = TIMEKEEPER.read();
    /* CLOCK_REALTIME = monotonic + offset_realtime */
    tk.xtime_sec + (tk.cycle_now() - tk.cycle_last) * tk.mult >> tk.shift / NSEC_PER_SEC
}
```

(For Y2038-safety, internal storage is `i64`. The truncation to 32-bit on i386 occurs only at the syscall return-register marshal layer.)

### Out of Scope

- `kernel/time/timekeeping.md` Tier-3: `xtime` machinery, NTP slew, `clocksource` selection.
- `clock_gettime.md` / `clock_settime.md` siblings (modern, full struct timespec).
- `gettimeofday.md` / `settimeofday.md` siblings.
- `adjtimex.md` (NTP slew interface).
- vDSO `__vdso_time` implementation.
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
time_t time(time_t *t);
```

Rust ABI shim:

```rust
pub fn sys_time(t: *mut Time_t) -> isize;
```

Syscall number: **201** (x86_64). Generic syscall table: not present on arm64 / generic — those architectures must use `clock_gettime(CLOCK_REALTIME, ...)` (glibc emulates `time(2)` via that). x32 / i386 have **time** at 13.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `t` | `time_t *` | OUT (UAPI ptr, may be NULL) | If non-NULL, the resulting `time_t` is also written here via `put_user`. |

### return

- **Success**: a non-negative `time_t` = current wall-clock seconds since Epoch.
- **Failure**: only on `t != NULL` + bad pointer: `-1` and `errno == EFAULT`.

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `t` is non-NULL and points to unmapped userspace memory. |

(There is no time-source error: the kernel always has a wall clock, although it may be wildly wrong before NTP sync.)

### abi surface

```text
__NR_time (x86_64) = 201
__NR_time (i386)   = 13
__NR_time (arm64 / generic) = unavailable
                              (emulated by glibc via clock_gettime(CLOCK_REALTIME, ...))

/* Type */
time_t = signed long          (64-bit on x86_64; 32-bit on i386)

/* Y2038 risk */
i386:    overflows at 2038-01-19 03:14:07 UTC (32-bit signed)
x86_64:  Y2038-safe (64-bit signed; overflows at year 292277026596)

/* Equivalence */
time(t) === { time_t now = clock_gettime(CLOCK_REALTIME, &ts).tv_sec;
               if (t) *t = now;
               return now; }
```

### compatibility contract

REQ-1: Syscall number is **201** on x86_64; **13** on i386; not present on arm64/generic. ABI-stable since 1.0 on x86.

REQ-2: Return value is `ktime_get_real_seconds()`: the wall-clock seconds component of `CLOCK_REALTIME`.

REQ-3: If `t != NULL`, write the same value via `put_user(now, t)`.

REQ-4: `t == NULL` is **valid** and common — `time(NULL)` is the canonical idiom.

REQ-5: `t` non-NULL but unmapped ⟹ `-EFAULT`. The function does NOT return the time in this case (the kernel's return register is set to `-EFAULT`).

REQ-6: Second granularity only: sub-second fractional time NOT returned. Callers needing finer resolution must use `clock_gettime`, `gettimeofday`, or `clock_getres`.

REQ-7: Y2038 behavior:
- 64-bit: `time_t` is 64-bit; no overflow until far future.
- 32-bit: `time_t` is 32-bit; overflows to negative at 2038-01-19 03:14:07 UTC.

REQ-8: Time-source is the wall clock (`CLOCK_REALTIME`), affected by NTP adjustments and `settimeofday(2)`. Not monotonic; can step backward.

REQ-9: No capability check.

REQ-10: Async-signal-safe per POSIX.1-2008.

REQ-11: vDSO acceleration: on x86_64, glibc dispatches via vDSO `__vdso_clock_gettime` for `time(2)` to avoid the syscall trap; the kernel syscall path remains the canonical implementation.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `null_t_ok` | INVARIANT | `t == NULL` ⟹ no put_user; only return. |
| `non_null_t_writes` | INVARIANT | `t != NULL` ⟹ `put_user(now, t)` attempted exactly once. |
| `bad_t_efault` | INVARIANT | bad `t` ⟹ `-EFAULT`. |
| `monotonic_within_second` | INVARIANT | two calls in same wall-second return identical values. |
| `equivalence_with_clock_gettime` | INVARIANT | `time(NULL) == clock_gettime(CLOCK_REALTIME, ...).tv_sec`. |
| `y2038_safe_64bit` | INVARIANT | on 64-bit, no overflow for `time_t < 2^63`. |

### Layer 2: TLA+

`uapi/time.tla`:
- Variables: `wallclock.sec`, `wallclock.nsec`, `t_arg`, `out`.
- Properties:
  - `safety_returns_wallclock_sec` — return == `wallclock.sec`.
  - `safety_null_no_write` — `t == NULL` ⟹ user memory unchanged.
  - `safety_efault_no_partial_write` — bad `t` ⟹ `-EFAULT`; no partial write observable to user.
  - `liveness_terminates` — call returns synchronously.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_time` post: return == ktime_get_real_seconds | `sys_time` |
| `sys_time(NULL)` post: no put_user issued | `sys_time` |
| `sys_time(t)` post: `*t == return` on success | `sys_time` |
| `Timekeeping::get_real_seconds` post: returns wall-clock seconds | `Timekeeping::get_real_seconds` |

### Layer 4: Verus/Creusot functional

Per-`time(2)` man page; per-LTP `testcases/kernel/syscalls/time/*`; per-glibc `sysdeps/unix/sysv/linux/time.c`; per-POSIX.1-2008; per-Y2038 transition plan in `Documentation/admin-guide/abi-stable.rst`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

time reinforcement:

- **Per-second-granularity strict** — defense against per-sub-second-info-leak: `time(2)` deliberately does NOT expose nanoseconds; callers wanting precision must use `clock_gettime`.
- **Per-NULL-t legal** — defense against per-spurious-EFAULT: `time(NULL)` is the canonical idiom.
- **Per-Y2038-aware (64-bit safe)** — defense against per-32-bit-overflow on modern kernels.
- **Per-wall-clock-step honored** — `time` follows `settimeofday`; defense against per-monotonic-confusion: callers needing monotonic must use `clock_gettime(CLOCK_MONOTONIC, ...)`.
- **Per-vDSO equivalence** — defense against per-vDSO-divergence from syscall path; both must return identical values modulo race.

### grsecurity/pax-style reinforcement

- **PaX UDEREF strict on `t` pointer** — every `put_user(now, t)` goes through validated UDEREF helper; defense against per-attacker kernel-pointer aliased as user.
- **PAX_RANDKSTACK on every entry** — randomise kstack on every `time` entry; even though arguments are scalar, defense against per-stack-spray of the timekeeper read path.
- **GRKERNSEC_RANDPID — time(2) precision-coarsen on suid** — under hardened policy, `time(2)` from a suid process is coarsened to the nearest 5-second boundary; defense against per-PRNG-seed-prediction in setuid programs that do `srand(time(NULL))`.
- **GRKERNSEC_HIDESYM on `TIMEKEEPER` address** — timekeeper struct excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`; defense against per-timekeeper layout reconnaissance.
- **GRKERNSEC_CLOCK_RESOLUTION** — `time(2)` already at second granularity; defense-in-depth: even the vDSO path is forced to second granularity under hardened policy (no nanosecond leak via vDSO debug variable).
- **CAP_SYS_TIME N/A for `time(2)` read** — but pair with `settimeofday`/`clock_settime` capability gating; defense against per-wallclock-mutation outside `CAP_SYS_TIME`.
- **Per-uid time rate-limit** — `time` calls above 100k/s per uid audit-logged as a possible timing-side-channel probe (unusual frequency).
- **Audit per-Y2038 boundary cross** — on 32-bit, the kernel audit-logs the first `time` call returning negative `time_t` post-2038, alerting userspace.
- **Refuse high-resolution sub-second derivation via `time` paired with retries** — under hardened policy, attempts to busy-loop `time` to detect second-boundary transitions (a covert-channel pattern) audit-logged.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `time(2)` to `-ENOSYS`, forcing applications onto `clock_gettime(CLOCK_REALTIME, ...)` which has explicit nanosecond resolution and Y2038-safe `struct timespec64`; defense against per-legacy-API surface.
- **GRKERNSEC_PROC_USER + `/proc/uptime` coarsening** — pair with `time(2)` to ensure all wall-clock readers agree on the coarsening policy; defense against per-side-channel disagreement.
- **PAX_USERCOPY strict on `put_user(time_t)`** — defense against per-copy-helper buffer overflow on architectures with non-aligned `time_t`.

