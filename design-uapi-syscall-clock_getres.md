---
title: "Tier-5 syscall: clock_getres(2) — syscall 229"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`clock_getres(2)` reports the resolution (smallest representable interval between two distinct readings) of a named clock. The value is determined by: the active hardware clocksource, the high-resolution timer subsystem state (`hrtimer_resolution` is `KTIME_LOW_RES` ≈ `1/HZ` if hrtimers are disabled, else 1 ns), and whether the clock is `_COARSE` (always `1/HZ` resolution). Userspace consumers — `pthread_condattr_setclock`-aware libraries, scientific code computing measurement noise floors, `clock_nanosleep` callers selecting between coarse and fine clocks — branch on the reported resolution to pick efficient strategies. The function does NOT measure clock jitter or accuracy; it only reports the granularity. Critical for: portable sleep loops, tick-tuned scheduling, sane defaults for sub-millisecond timing libraries.

This Tier-5 covers the userspace ABI of syscall 229. Clocksource selection, hrtimer subsystem state, and per-CPU tick infrastructure are owned by `kernel/time/clocksource.md` and `kernel/time/hrtimer.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: `clock_getres(CLOCK_MONOTONIC, &r)`: `r.tv_nsec == 1` under `CONFIG_HIGH_RES_TIMERS=y`.
- [ ] AC-2: `clock_getres(CLOCK_REALTIME_COARSE, &r)`: `r.tv_nsec == TICK_NSEC`.
- [ ] AC-3: `clock_getres(99999, NULL)`: `-EINVAL`.
- [ ] AC-4: `clock_getres(CLOCK_MONOTONIC, NULL)`: returns `0` (probe).
- [ ] AC-5: `clock_getres(CLOCK_MONOTONIC, (struct timespec*)1)`: `-EFAULT`.
- [ ] AC-6: `clock_getres(CLOCK_THREAD_CPUTIME_ID, &r)`: `r.tv_nsec == 1`.
- [ ] AC-7: Per-pid CPU clock for non-existent pid: `-ESRCH`.
- [ ] AC-8: Coarse clock resolution matches `1/HZ` under all `HZ` values.
- [ ] AC-9: `CLOCK_REALTIME_ALARM` getres succeeds without `CAP_WAKE_ALARM`.
- [ ] AC-10: Resolution reported is monotonic across kernel versions for same clocksource.

### Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
pub fn sys_clock_getres(clk_id: i32, res_user: *mut __kernel_timespec) -> isize {
    let res = PosixTimers::clock_getres(clk_id)?;
    if !res_user.is_null() {
        copy_to_user(res_user, &res).map_err(|_| -EFAULT)?;
    }
    0
}
```

`PosixTimers::clock_getres(clk_id) -> Result<KernelTimespec, isize>`:
1. /* dynamic */
2. if (clk_id as u32 & CLOCKFD_MASK) == CLOCKFD:
   - return PosixClock::dispatch_getres(clk_id);
3. /* CPU-encoded */
4. if (clk_id as i32) < 0 {
   - return PosixCpuClock::getres(clk_id);
   - }
5. let nsec = match ClockId::try_from(clk_id) {
   - Realtime | Monotonic | MonotonicRaw | Boottime | Tai
   - | RealtimeAlarm | BoottimeAlarm
   - | ProcessCpuTimeId | ThreadCpuTimeId => hrtimer_resolution(),
   - RealtimeCoarse | MonotonicCoarse => TICK_NSEC,
   - _ => return Err(-EINVAL),
 };
6. Ok(KernelTimespec { tv_sec: 0, tv_nsec: nsec as i64 })

`hrtimer_resolution() -> u64`:
1. if hrtimer_hres_active() { 1 } else { TICK_NSEC }

`PosixCpuClock::getres(clk_id) -> Result<KernelTimespec, isize>`:
1. let (pid, which) = decode_cpu_clockid(clk_id);
2. let task = find_task_by_pid(pid).ok_or(-ESRCH)?;
3. if task.pidns != current.pidns ∧ !capable(CAP_SYS_PTRACE) { return Err(-EPERM); }
4. Ok(KernelTimespec { tv_sec: 0, tv_nsec: hrtimer_resolution() as i64 })

### Out of Scope

- `kernel/time/clocksource.md` Tier-3: clocksource selection, TSC vs HPET vs ACPI_PM.
- `kernel/time/hrtimer.md` Tier-3: hrtimer subsystem state machine.
- `kernel/time/posix_cpu_timers.md` Tier-3: per-thread / per-process CPU-time accounting.
- `clock_gettime.md` / `clock_settime.md` / `clock_nanosleep.md` siblings.
- Dynamic POSIX clock (`/dev/ptp*`) backends.
- glibc / musl userspace `clock_getres` wrappers.
- Implementation code.

### signature

```c
int clock_getres(clockid_t clk_id, struct timespec *res);
```

Rust ABI shim:

```rust
pub fn sys_clock_getres(clk_id: i32, res: *mut __kernel_timespec) -> isize;
```

Syscall number: **229**.
On 32-bit architectures the time-64-safe variant is `sys_clock_getres_time64` (syscall 406).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `clk_id` | `clockid_t` (`i32`) | IN | clock whose resolution is queried |
| `res` | `struct __kernel_timespec *` | OUT | smallest representable interval; may be NULL (validates `clk_id` only) |

### return

- **Success**: `0`; if `res != NULL`, `*res` populated.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | unrecognized `clk_id` |
| `EFAULT` | `res != NULL` but not in writable userspace |
| `EOPNOTSUPP` | dynamic POSIX clock returns ENOTSUP from `clock_getres` |
| `ESRCH` | per-pid CPU clock targeting non-existent pid |
| `EPERM` | cross-pidns per-pid CPU clock without `CAP_SYS_PTRACE` |

### abi surface

Reported resolutions per clock (typical x86_64 with TSC clocksource and `CONFIG_HIGH_RES_TIMERS=y`, `HZ=1000`):

| Clock | Typical `tv_sec` | Typical `tv_nsec` | Source |
|---|---|---|---|
| `CLOCK_REALTIME` | `0` | `1` | hrtimer (1 ns) |
| `CLOCK_MONOTONIC` | `0` | `1` | hrtimer (1 ns) |
| `CLOCK_MONOTONIC_RAW` | `0` | `1` | hrtimer (1 ns) |
| `CLOCK_BOOTTIME` | `0` | `1` | hrtimer (1 ns) |
| `CLOCK_TAI` | `0` | `1` | hrtimer (1 ns) |
| `CLOCK_REALTIME_COARSE` | `0` | `1_000_000` (1 ms @ HZ=1000) | `TICK_NSEC` |
| `CLOCK_MONOTONIC_COARSE` | `0` | `1_000_000` | `TICK_NSEC` |
| `CLOCK_PROCESS_CPUTIME_ID` | `0` | `1` | sched_clock |
| `CLOCK_THREAD_CPUTIME_ID` | `0` | `1` | sched_clock |
| `CLOCK_REALTIME_ALARM` | `0` | `1` | alarm-timer hrtimer |
| `CLOCK_BOOTTIME_ALARM` | `0` | `1` | alarm-timer hrtimer |

Under `CONFIG_HIGH_RES_TIMERS=n`: all fine-grain clocks report `TICK_NSEC` instead of 1 ns.

`struct __kernel_timespec` layout matches `clock_gettime.md`.

### compatibility contract

REQ-1: `clk_id` validation:
- Recognized values match `clock_gettime` set.
- Per-pid / per-tid CPU clocks: target task validity checked.
- Unknown: `-EINVAL`.

REQ-2: `res == NULL`:
- Allowed; serves as cheap "is this clk_id known?" probe.
- No `EFAULT` for NULL `res`.

REQ-3: Hi-res clocks:
- If `hrtimer_resolution == HIGH_RES_NSEC` (1): report `{0, 1}`.
- Else: report `{0, TICK_NSEC}`.

REQ-4: Coarse clocks:
- Always report `{0, TICK_NSEC}` regardless of hrtimer state.

REQ-5: CPU-time clocks:
- Report `{0, hrtimer_resolution}` (sched_clock granularity).

REQ-6: Alarm clocks:
- Report hrtimer resolution; require `CAP_WAKE_ALARM` only for `_settime`/`_nanosleep`, NOT for `_getres`.

REQ-7: Dynamic POSIX clocks:
- Dispatch to `posix_clock_ops::clock_getres`.
- Backend errors propagate.

REQ-8: Per-pid CPU clock:
- Target pid validated; missing → `-ESRCH`.
- Cross-pidns: `CAP_SYS_PTRACE` required else `-EPERM`.

REQ-9: Time-64 ABI:
- 32-bit architectures use `sys_clock_getres_time64` (406) returning `__kernel_timespec`.

REQ-10: Signal safety:
- `clock_getres` is async-signal-safe per POSIX.1-2008.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `clk_id_known_or_einval` | INVARIANT | unrecognized clk_id ⟹ -EINVAL |
| `res_null_allowed` | INVARIANT | `res == NULL` ⟹ no EFAULT |
| `coarse_returns_tick_nsec` | INVARIANT | COARSE clocks always TICK_NSEC |
| `fine_returns_hrtimer_res` | INVARIANT | non-COARSE: hrtimer_resolution |
| `crosspidns_cap_enforced` | INVARIANT | per-pid CPU clock cross-pidns ⟹ CAP_SYS_PTRACE |

### Layer 2: TLA+

`uapi/clock_getres.tla`:
- Variables: `hrtimer_hres_active`, `TICK_NSEC`, `posix_clock_table[fd]`.
- Properties:
  - `safety_coarse_constant` — COARSE clocks always report TICK_NSEC.
  - `safety_unknown_clk_einval` — out-of-range clk_id always returns EINVAL.
  - `liveness_returns` — terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `clock_getres` post(0): if non-null, `*res = {0, ns}` with `ns ∈ {1, TICK_NSEC}` | `PosixTimers::clock_getres` |
| `hrtimer_resolution` returns ∈ {1, TICK_NSEC} | `hrtimer_resolution` |
| per-pid getres: ESRCH if pid absent | `PosixCpuClock::getres` |

### Layer 4: Verus/Creusot functional

Per-`clock_getres(2)` man page, `Documentation/core-api/timekeeping.rst`, glibc `sysdeps/unix/sysv/linux/clock_getres.c`, musl `src/time/clock_getres.c`. Per-LTP `testcases/kernel/syscalls/clock_getres/*` round-trip.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

clock_getres reinforcement:

- **Per-clk_id whitelist** — unknown values rejected with `-EINVAL`; never reads kernel state for unknown clocks.
- **Per-NULL probe allowed** — userspace can validate `clk_id` cheaply without exposing kernel pointers.
- **Per-copy_to_user EFAULT-safe** — bad `res` pointer never causes kernel pointer leak.
- **Per-cross-pidns CAP_SYS_PTRACE gating** — per-pid CPU clock access constrained.
- **Per-dynamic-clock backend isolation** — `posix_clock_ops` dispatch table immutable post-registration.

### grsecurity/pax-style reinforcement

- **PAX_RANDKSTACK on every entry** — clock_getres is callable from any task; randomize kstack on each entry.
- **PaX UDEREF on `res` writeback** — `copy_to_user(res, &ts)` refuses kernel-mapped pages; the writeback writes hrtimer_resolution which is a useful (1 vs TICK_NSEC) signal about kernel config to attackers planning timing-side-channel attacks.
- **GRKERNSEC_CLOCK_RESOLUTION** — under hardened policy, `clock_getres` for any non-COARSE clock returns `{0, TICK_NSEC}` (lying about resolution) when caller is suid, has no `CAP_SYS_NICE`, or runs in a non-init userns. Defense against high-precision timing side channels: an attacker confirming the system has nanosecond-resolution `CLOCK_MONOTONIC` knows that `clock_gettime` is a viable cache-timing oracle, whereas a TICK_NSEC-resolution answer encourages the attacker to assume the timing channel is too noisy. Tunable: `grsec_clock_resolution_ns` matches the value coarsened in `clock_gettime`.
- **CAP_SYS_TIME N/A** — read-only; no privileges enforced beyond the cross-pidns CPU-clock case.
- **CAP_WAKE_ALARM N/A** — alarm clocks are queryable via `clock_getres` without the capability; only their `_settime`/`_nanosleep` paths gate on CAP_WAKE_ALARM.
- **adjtimex rate-limit irrelevant** — `clock_getres` returns a clock-source / hrtimer constant that does not depend on NTP discipline; adjtimex rate-limiting (sibling syscall) does not affect `clock_getres`.
- **GRKERNSEC_PROC_USERGROUP cross-pidns CPU clock denied** — per-pid CPU clock targeting tasks outside caller's pidns refused with `-EPERM` even with `CAP_SYS_PTRACE` under hardened userns policy.
- **GRKERNSEC_HIDESYM on hrtimer subsystem addresses** — `hrtimer_hres_active` global flag excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`.
- **Audit non-init-userns hrtimer probing** — repeated `clock_getres` calls from a single non-init-userns uid against many clocks is logged as a potential timing-oracle fingerprint probe.
- **Refuse CLOCK_TAI getres outside CAP_SYS_TIME if leap pending** — defense-in-depth against attackers using TAI resolution as a leap-second-pending oracle.
- **Per-NULL probe rate-limit** — `clock_getres(any, NULL)` is callable by any task as a clk_id enumerator; under hardened policy rate-limited per uid to discourage fingerprinting.

