---
title: "Tier-5 syscall: gettimeofday(2) — syscall 96"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`gettimeofday(2)` is the legacy BSD/POSIX wall-clock read interface: returns the current `CLOCK_REALTIME` value with microsecond resolution into `struct timeval` and (deprecated) a `struct timezone` containing tzinfo. It has been **functionally superseded** by `clock_gettime(CLOCK_REALTIME, ...)` (nanosecond resolution, async-signal-safe, no tz baggage) but remains the canonical microsecond-resolution interface in System V derivatives, ancient C code, and almost every libc implementation as the underlying primitive for `time(2)`, `ftime(3)`, `times(3)`, and a hundred other helpers. The vDSO fastpath (`__vdso_gettimeofday`) services this entirely in userspace via the shared `vdso_data` page, never trapping into the kernel; the syscall is only the fallback. The `tz` argument is essentially vestigial: the kernel maintains a `sys_tz` struct (set once during boot, mutable via `settimeofday(NULL, &tz)`), but glibc/musl always pass `tz = NULL` and rely on `/etc/localtime` / `TZ` instead. Critical for: every microsecond timestamp in legacy code, the entire POSIX `time()` heritage chain.

This Tier-5 covers the userspace ABI of syscall 96. The vDSO update protocol and timekeeper interaction are owned by `kernel/time/timekeeping.md` and `kernel/time/vdso.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: `gettimeofday(&tv, NULL)`: `tv.tv_sec` matches `clock_gettime(CLOCK_REALTIME).tv_sec` within ±1 s.
- [ ] AC-2: `tv.tv_usec ∈ [0, 999_999]`.
- [ ] AC-3: `gettimeofday(NULL, NULL)`: returns `0` (probe).
- [ ] AC-4: `gettimeofday(NULL, &tz)`: returns `sys_tz`.
- [ ] AC-5: `gettimeofday((void*)1, NULL)`: `-EFAULT`.
- [ ] AC-6: `gettimeofday(NULL, (void*)1)`: `-EFAULT`.
- [ ] AC-7: After `settimeofday`: subsequent `gettimeofday` reflects new value.
- [ ] AC-8: vDSO `__vdso_gettimeofday` and syscall return consistent values within 1µs.
- [ ] AC-9: Two reads `t1`, `t2` with `t2` after `t1`: `t2 ≥ t1` (modulo `settimeofday` jumps).
- [ ] AC-10: Multiple concurrent vDSO readers: never observe torn time.

### Architecture

Rookery surface in `kernel/time/time.rs`:

```rust
#[repr(C)]
pub struct OldTimeval {
    pub tv_sec:  i64,  // 64-bit on x86_64/arm64/riscv64
    pub tv_usec: i64,
}

#[repr(C)]
pub struct OldTimezone {
    pub tz_minuteswest: i32,
    pub tz_dsttime:     i32,
}

static SYS_TZ: SpinLock<OldTimezone> = SpinLock::new(OldTimezone { tz_minuteswest: 0, tz_dsttime: 0 });

pub fn sys_gettimeofday(tv_user: *mut OldTimeval, tz_user: *mut OldTimezone) -> isize {
    if !tv_user.is_null() {
        let ts = Timekeeper::get_real_ts64();
        let tv = OldTimeval {
            tv_sec:  ts.tv_sec,
            tv_usec: ts.tv_nsec / 1000,
        };
        copy_to_user(tv_user, &tv).map_err(|_| -EFAULT)?;
    }
    if !tz_user.is_null() {
        let tz = *SYS_TZ.lock();
        copy_to_user(tz_user, &tz).map_err(|_| -EFAULT)?;
    }
    0
}
```

vDSO surface `__vdso_gettimeofday` lives in `arch/x86/entry/vdso/vclock_gettime.c` and mirrors:

```rust
pub extern "C" fn __vdso_gettimeofday(tv: *mut OldTimeval, tz: *mut OldTimezone) -> i32 {
    if !tv.is_null() {
        loop {
            let seq = vdso_data.seq.read();
            let now = read_clocksource_cycle();
            let nsec = ((now - vdso_data.cycle_last) * vdso_data.mult) >> vdso_data.shift;
            let sec = vdso_data.xtime_sec;
            let nsec_total = vdso_data.xtime_nsec + nsec;
            if vdso_data.seq.retry(seq) { continue; }
            unsafe {
                (*tv).tv_sec  = sec + (nsec_total / 1_000_000_000) as i64;
                (*tv).tv_usec = ((nsec_total % 1_000_000_000) / 1000) as i64;
            }
            break;
        }
    }
    if !tz.is_null() {
        unsafe {
            (*tz).tz_minuteswest = vdso_data.tz_minuteswest;
            (*tz).tz_dsttime     = vdso_data.tz_dsttime;
        }
    }
    0
}
```

### Out of Scope

- `kernel/time/timekeeping.md` Tier-3: tk_core, clocksource, xtime advance.
- `kernel/time/vdso.md` Tier-3: vDSO shared-page update protocol.
- `settimeofday.md` sibling — write path.
- `clock_gettime.md` / `clock_settime.md` siblings — modern interface.
- `time(2)` syscall — seconds-only variant.
- glibc / musl userspace `gettimeofday` wrappers.
- Implementation code.

### signature

```c
int gettimeofday(struct timeval *tv, struct timezone *tz);
```

Rust ABI shim:

```rust
pub fn sys_gettimeofday(tv: *mut __kernel_old_timeval,
                        tz: *mut __kernel_old_timezone) -> isize;
```

Syscall number: **96**.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `tv` | `struct timeval *` | OUT | `{tv_sec: time_t, tv_usec: suseconds_t}` — wall clock, microsecond resolution; may be NULL |
| `tz` | `struct timezone *` | OUT | `{tz_minuteswest: int, tz_dsttime: int}` — kernel `sys_tz`; may be NULL (almost always NULL) |

### return

- **Success**: `0`; outputs populated as requested.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `tv != NULL` and not writable userspace; `tz != NULL` and not writable userspace |

(No `EINVAL`: any pointer values are accepted, including NULL.)

### abi surface

`struct timeval` (BSD-classic):

```c
struct timeval {
    __kernel_old_time_t tv_sec;   /* seconds since 1970-01-01 */
    __kernel_suseconds_t tv_usec; /* microseconds [0, 999_999] */
};
```

On 64-bit Linux, `__kernel_old_time_t` and `__kernel_suseconds_t` are `long`. On 32-bit Linux, they are `long` (32-bit) — subject to 2038 truncation; replaced by `clock_gettime` + `struct __kernel_timespec` for time64-safe code.

`struct timezone` (vestigial):

```c
struct timezone {
    int tz_minuteswest;   /* minutes west of UTC */
    int tz_dsttime;       /* (obsolete) DST correction algorithm */
};
```

Kernel maintains a global `sys_tz` (default `{0, 0}` on most distros; settable via `settimeofday(NULL, &tz)`).

### compatibility contract

REQ-1: `tv` writeback:
- If non-NULL, copy `(xtime_sec, xtime_nsec / 1000)` truncated-to-microseconds.
- `tv_usec ∈ [0, 999_999]` — never `1_000_000`.

REQ-2: `tz` writeback:
- If non-NULL, copy the current `sys_tz` global.

REQ-3: NULL acceptance:
- Both `tv == NULL` and `tz == NULL` are valid; useful as a per-vDSO probe or no-op.

REQ-4: vDSO fastpath:
- libc MUST first attempt `__vdso_gettimeofday(tv, tz)`.
- vDSO reads `vdso_data` under seqcount; retries on torn read.
- Falls back to syscall only if vDSO unavailable (seccomp-strict, very-old kernel).

REQ-5: Clock source:
- Same `xtime` as `clock_gettime(CLOCK_REALTIME)`; consistent within nanosecond granularity (then truncated to microseconds).

REQ-6: Time-truncation:
- `tv_usec = xtime_nsec / 1000` (floor toward 0).
- Loses up to 999 nanoseconds of resolution vs `clock_gettime`.

REQ-7: Y2038:
- 32-bit `time_t`: wraps at 2038-01-19 03:14:08 UTC.
- 64-bit `time_t`: safe to year 292_277_026_596.
- Per-`time_t` size matches architecture word size; no `_time64` variant — modern code MUST use `clock_gettime` for 2038-safety on 32-bit.

REQ-8: Signal safety:
- `gettimeofday` is async-signal-safe per POSIX.1-2008.

REQ-9: Thread safety:
- Re-entrant; no shared state besides the kernel timekeeper read.

REQ-10: vDSO + syscall consistency:
- vDSO and syscall MUST return values consistent within one timekeeper update cycle.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tv_usec_in_range` | INVARIANT | `tv_usec ∈ [0, 999_999]` |
| `null_pointers_allowed` | INVARIANT | NULL `tv` or `tz` never causes EFAULT |
| `copy_to_user_efault_clean` | INVARIANT | bad userspace pointer: never leaks kernel value to non-userspace page |
| `vdso_seqcount_retry_bounded` | INVARIANT | bounded retries (with fallback to syscall) |
| `sys_tz_lock_held` | INVARIANT | SYS_TZ accessed under spinlock |

### Layer 2: TLA+

`uapi/gettimeofday.tla`:
- Variables: `vdso_data.xtime`, `vdso_data.seq`, `sys_tz`.
- Properties:
  - `safety_no_torn_read` — userspace never observes inconsistent (sec, usec) tuples.
  - `safety_truncation_consistent` — `usec` is floor(`nsec` / 1000).
  - `safety_tz_writes_under_lock` — `sys_tz` mutation serialized.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `gettimeofday` post(0): if `tv != NULL`, `tv_usec ∈ [0, 999_999]` | `sys_gettimeofday` |
| `gettimeofday` post: `tv_sec * 1e6 + tv_usec` matches `clock_gettime(REALTIME)` modulo nsec | `sys_gettimeofday` |
| `gettimeofday` post: if `tz != NULL`, `*tz == sys_tz` | `sys_gettimeofday` |
| vDSO returns same value as syscall within seqcount cycle | `__vdso_gettimeofday` |

### Layer 4: Verus/Creusot functional

Per-`gettimeofday(2)` man page, glibc `sysdeps/unix/sysv/linux/gettimeofday.c`, musl `src/time/gettimeofday.c`. Per-LTP `testcases/kernel/syscalls/gettimeofday/*` round-trip.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

gettimeofday reinforcement:

- **Per-vDSO fastpath** — userspace reads kernel timekeeper data without entering kernel; defense against syscall-flood DoS on time reads.
- **Per-vdso_data PROT_READ in userspace** — userspace cannot corrupt kernel timekeeper data via the shared mapping.
- **Per-tv_usec truncation explicit** — never returns out-of-range microseconds.
- **Per-NULL-allowed semantics** — defense against forced-EFAULT in legacy callers using NULL-probe pattern.
- **Per-SYS_TZ lock held on read** — torn-free observation of (tz_minuteswest, tz_dsttime).

### grsecurity/pax-style reinforcement

- **PAX_RANDKSTACK on every syscall entry** — `gettimeofday` is sometimes called millions of times per second in microbenchmarks that bypass the vDSO (seccomp-strict profiles); randomize kstack on each entry.
- **PaX UDEREF on `tv` and `tz` writeback** — `copy_to_user(tv_user, &tv)` and `copy_to_user(tz_user, &tz)` refuse kernel-mapped pages. Particularly important here because `tv` carries timekeeper-derived data — a kernel-write primitive into attacker-controlled memory is exploitable for ROP-chain construction.
- **GRKERNSEC_CLOCK_RESOLUTION** — under hardened policy, `gettimeofday` for suid binaries and `CAP_SYS_NICE`-less tasks returns microseconds rounded to `1/HZ` resolution (zeros out the bottom log2(HZ) bits); defense against high-precision timing channels even when the vDSO fastpath is bypassed via syscall. Tunable: `grsec_clock_resolution_ns` (default coarsens to `TICK_NSEC / 1000` µs under hardened profile).
- **vDSO ASLR per-process** — `vdso_data` mapping address randomized per task to deny stable userspace observation of kernel timekeeper structure addresses.
- **CAP_SYS_TIME N/A** — read-only; sibling `settimeofday` enforces CAP_SYS_TIME.
- **CAP_WAKE_ALARM N/A** — does not arm alarms.
- **adjtimex rate-limit interaction** — write side (`settimeofday` / `adjtimex`) shares a per-uid rate-limit token bucket so attackers cannot rapidly perturb `gettimeofday` readers via clock-mutation oracles.
- **GRKERNSEC_HIDESYM on `vdso_data` symbol** — vdso_data global address excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`; the data page itself is mapped at a random per-process address.
- **Refuse pre-epoch / 2038-wrap detection** — under hardened policy with 32-bit `time_t`, if `tv_sec` would wrap (2038 imminent), audit-log the call; defense-in-depth for legacy code.
- **Bounded vDSO retry counter** — userspace `__vdso_gettimeofday` retry loop capped at 1024; on cap-hit fall through to syscall path and audit; defense against priority-inverted writer livelock targeting realtime readers.
- **Audit suspicious patterns** — repeated calls with intentionally-bad `tv`/`tz` pointers from a single uid: audit-logged as potential EFAULT-driven kernel-pointer-leak probes.
- **`sys_tz` write rate-limited** — `settimeofday(NULL, &tz)` rate-limited per uid (sibling enforcement); rapid timezone-flip would create distinguishable observation patterns in `gettimeofday(NULL, &tz)`.

