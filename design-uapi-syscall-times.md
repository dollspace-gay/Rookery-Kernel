---
title: "Tier-5 syscall: times(2) — syscall 100"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`times(2)` is the historic POSIX syscall that returns process CPU-time accounting in a fixed four-field `struct tms` (user, system, children-user, children-system) and additionally returns "clock ticks since an arbitrary point in the past" as the syscall's return value. Times are reported in units of `clock_t` (with `sysconf(_SC_CLK_TCK)` giving the tick rate, typically 100 Hz). It is the lowest-overhead syscall for sampling cputime and is still emitted by `time(1)` and shell `times` builtin.

Critical for: shell `times` builtin, `time(1)` user-time accounting fallback, low-overhead profilers, POSIX conformance.

### Acceptance Criteria

- [ ] AC-1: `times(&buf)` returns elapsed ticks (>= 0 in practice).
- [ ] AC-2: After busy-loop, `buf.tms_utime` increments.
- [ ] AC-3: After syscall-heavy load, `buf.tms_stime` increments.
- [ ] AC-4: After a `wait4`-reaped child, `buf.tms_cutime` / `tms_cstime` accumulate.
- [ ] AC-5: `buf = NULL` returns elapsed ticks; no fault.
- [ ] AC-6: `buf` faulting → return `(clock_t)-1`, errno = EFAULT.
- [ ] AC-7: Two sequential calls: `buf2.tms_utime >= buf1.tms_utime`.
- [ ] AC-8: Return value of second call >= first within same boot.
- [ ] AC-9: `tms_utime + tms_stime` does not exceed elapsed wall-clock ticks (single-CPU bound).

### Architecture

```rust
#[syscall(nr = 100, abi = "sysv")]
pub fn sys_times(buf: UserPtrMut<Tms>) -> isize {
    Sched::do_times(buf)
}
```

`Sched::do_times(buf) -> isize`:
1. if !buf.is_null() {
   - let mut tms = Tms::zeroed();
   - Sched::fill_tms(&mut tms);
   - buf.copy_out(&tms)?;                            // EFAULT -> -EFAULT
   };
2. /* Return value: elapsed ticks since boot */
3. let elapsed = (jiffies::current() - INITIAL_JIFFIES) as i64;
4. Sched::jiffies_to_clock_t(elapsed) as isize

`Sched::fill_tms(tms) -> ()`:
1. let t = current();
2. let (utime, stime) = thread_group_cputime_adjusted(t);
3. tms.tms_utime = Sched::nsec_to_clock_t(utime);
4. tms.tms_stime = Sched::nsec_to_clock_t(stime);
5. tms.tms_cutime = Sched::nsec_to_clock_t(t.signal.cutime);
6. tms.tms_cstime = Sched::nsec_to_clock_t(t.signal.cstime);

`Sched::nsec_to_clock_t(ns) -> ClockT`:
1. /* Convert nanoseconds to user-clock-ticks */
2. /* On HZ=100 systems, divide by 10_000_000 */
3. (ns / (NSEC_PER_SEC / USER_HZ as u64)) as ClockT

`Sched::jiffies_to_clock_t(j) -> ClockT`:
1. /* When USER_HZ == HZ, identity; otherwise scale */
2. (j * USER_HZ as i64 / HZ as i64) as ClockT

### Out of Scope

- `getrusage(2)` (covered in this wave's `getrusage.md`; finer-grained accounting).
- `clock_gettime(CLOCK_PROCESS_CPUTIME_ID)` (covered separately).
- `time(2)` wall-clock syscall (covered separately).
- `wait4(2)` (covered separately; accumulates children counters).
- `sysconf(_SC_CLK_TCK)` glibc surface (userspace).
- Implementation code.

### signature

```c
clock_t times(struct tms *buf);
```

```c
struct tms {
    clock_t tms_utime;   /* user CPU time of calling process (sum across thread-group) */
    clock_t tms_stime;   /* system CPU time of calling process */
    clock_t tms_cutime;  /* user CPU time of waited-for children */
    clock_t tms_cstime;  /* system CPU time of waited-for children */
};

typedef __kernel_clock_t clock_t;   /* long on most arches */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `buf` | `struct tms *` | out | Caller buffer. `NULL` is permitted (kernel skips the copy_to_user, still returns clock ticks). |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Elapsed clock ticks since "an arbitrary point in the past" — Linux uses jiffies since boot, mapped to user-clock-ticks (`jiffies_to_clock_t`). |
| `(clock_t)-1` + `errno` | Failure (only `EFAULT`). NOTE: `(clock_t)-1` is a legitimate elapsed-tick value, so callers must clear `errno` before the call and check it after. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `buf` non-NULL and user pointer faults during copy_to_user. |

### abi surface

```text
__NR_times (x86_64) = 100
__NR_times (arm64)  = 153
__NR_times (riscv)  = 153
__NR_times (i386)   =  43

/* clock_t is `long` on Linux; on 32-bit ABIs that's 32-bit and
   wraps at ~497 days when CLK_TCK = 100. */
/* CLK_TCK is the user-visible HZ; the kernel internal HZ may differ. */
```

### compatibility contract

REQ-1: Syscall number is **100** on x86_64. ABI-stable since 1.0.

REQ-2: `tms_utime`, `tms_stime`: `thread_group_cputime_adjusted(current)` user/system components, converted from nanoseconds to user-clock-ticks via `nsec_to_clock_t`.

REQ-3: `tms_cutime`, `tms_cstime`: `current->signal->cutime`, `cstime` accumulated from `wait4(2)`-reaped children, converted similarly. Excludes still-live children.

REQ-4: Return value: `jiffies_to_clock_t(jiffies - INITIAL_JIFFIES)` — elapsed wall-clock ticks since boot. May wrap on 32-bit `clock_t`.

REQ-5: `buf == NULL`: no copy_to_user performed; return-value path unchanged.

REQ-6: Per-32-bit ABI: `clock_t` is `__kernel_old_clock_t` = `long` (32-bit signed). Wraps at `LONG_MAX / CLK_TCK` seconds ≈ 248 days (signed) when CLK_TCK = 100.

REQ-7: Per-locking: cputime read via `thread_group_cputime_adjusted` is lockless on vtime-enabled arches; siglock-protected on others.

REQ-8: Per-`compat_times`: 32-bit compat path uses `compat_clock_t` (32-bit); same fields.

REQ-9: Per-`CLK_TCK`: userspace queries via `sysconf(_SC_CLK_TCK)`; kernel exposes it as `USER_HZ` (default 100 on most arches).

REQ-10: Per-monotonicity: subsequent `times()` calls from the same process return non-decreasing `tms_utime + tms_stime`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `null_buf_ok` | INVARIANT | buf == NULL ⟹ no copy, no fault. |
| `monotonic_utime_stime` | INVARIANT | per-task: subsequent times call: tms_utime/stime non-decreasing. |
| `cutime_only_reaped` | INVARIANT | cutime/cstime only accumulate from wait4-reaped children. |
| `clock_t_overflow_safe` | INVARIANT | nsec_to_clock_t: no UB on saturating conversion. |
| `efault_no_partial_copy` | INVARIANT | partial copy_to_user failure ⟹ EFAULT, buf unchanged. |

### Layer 2: TLA+

`kernel/times.tla`:
- States: per-call cputime-read, conversion, copy_out, return-tick-compute.
- Properties:
  - `safety_cputime_monotonic` — subsequent times() call from same thread-group: tms_utime/stime non-decreasing.
  - `safety_elapsed_non_negative` — return value >= 0.
  - `safety_cutime_only_reaped` — children counters update only on wait4 reap.
  - `liveness_terminates` — call always returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_times` post: returns elapsed_ticks ∨ -EFAULT | `Sched::do_times` |
| `fill_tms` post: tms_utime/stime/cutime/cstime non-negative | `Sched::fill_tms` |
| `nsec_to_clock_t` post: conversion saturates, no UB | `Sched::nsec_to_clock_t` |

### Layer 4: Verus / Creusot functional

Per-`times(2)` man-page + POSIX-2008 semantic equivalence. Per-`time(1)` and shell-`times` builtin agreement with `getrusage` cputime within rounding.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`times(2)` reinforcement:

- **Per-`buf == NULL` permitted (skip copy)** — defense against per-fault-on-NULL.
- **Per-cputime conversion saturating** — defense against per-overflow UB on extreme uptimes.
- **Per-monotonic-cputime invariant** — defense against per-counter regression.
- **Per-`copy_to_user` all-or-nothing** — defense against per-partial-write info-leak.
- **Per-fast-path lockless** — defense against per-DoS via cheap polling.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `buf` copy_to_user** — defense against per-user-pointer kernel-deref bug; SMAP forced.
- **GRKERNSEC_HIDESYM on sched_clock-based cputime** — `tms_utime` / `tms_stime` quantized to 10ms (1 user-tick) for unprivileged callers under hardened policy; defense against per-sched_clock side-channel that could reveal crypto / branch-prediction timing via repeated `times()` polling.
- **GRKERNSEC_PROC_GETPID** — children counters (`tms_cutime`, `tms_cstime`) hidden when the calling process cannot ptrace its own descendants under hardened-userns policy; defense against per-container info-leak about external workload.
- **PAX_USERCOPY_HARDEN on `struct tms` copy_to_user** — bounded 32-byte copy uses whitelisted slab region; defense against per-overlong-copy.
- **Per-elapsed-ticks return quantized for non-root** — the elapsed-since-boot return value is rounded to 1-second granularity for unprivileged callers; defense against per-boot-time fingerprinting (boot epoch ↔ public Tor/server uptime correlation).
- **GRKERNSEC_NO_GETPID-style** — `tms_cutime` for descendants in another pidns blanked to zero; defense against per-container task-tree fingerprinting.
- **PAX_REFCOUNT on `task->signal` access** — defense against per-task UAF race when concurrent exit.
- **PaX KERNEXEC on `nsec_to_clock_t` inlined helper** — defense against per-W^X violation.
- **Per-call rate-limit (token-bucket per-UID)** — defense against per-DoS via tight times() polling that taxes per-CPU vtime updates.
- **Per-`USER_HZ` constant from .rodata** — defense against per-runtime-tampering with USER_HZ.

