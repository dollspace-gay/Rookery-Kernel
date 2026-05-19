# Tier-5 syscall: clock_gettime(2) — syscall 228

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/posix-timers.c (sys_clock_gettime, do_clock_gettime)
  - kernel/time/posix-stubs.c (compat path)
  - kernel/time/timekeeping.c (ktime_get_real_ts64, ktime_get_ts64, ktime_get_coarse_real_ts64)
  - kernel/time/posix-cpu-timers.c (process / thread CPU time)
  - kernel/time/vsyscall.c (vDSO update_vsyscall)
  - arch/x86/entry/vdso/vclock_gettime.c (vDSO fastpath)
  - include/uapi/linux/time.h (clockid_t values, struct __kernel_timespec)
  - arch/*/include/generated/uapi/asm/unistd_64.h (228)
  - Documentation/core-api/timekeeping.rst, man clock_gettime(2)
-->

## Summary

`clock_gettime(2)` reads the current value of one of nine kernel-tracked clocks into a userspace `struct timespec` with nanosecond resolution. It is the canonical timestamping primitive on Linux: every modern timestamp (libc `time()`, glibc `clock()`, C++ `steady_clock::now()`, Rust `Instant::now()`, Go `time.Now()`, Java `System.nanoTime()`) reduces to `clock_gettime` with a specific `clockid_t`. The vDSO fastpath services `CLOCK_REALTIME`, `CLOCK_MONOTONIC`, `CLOCK_REALTIME_COARSE`, `CLOCK_MONOTONIC_COARSE`, `CLOCK_BOOTTIME`, `CLOCK_TAI`, and `CLOCK_MONOTONIC_RAW` entirely in userspace through a seqcount-protected shared page, never trapping into the kernel; CPU-time clocks and dynamic POSIX clocks (`/dev/ptp*`, dynamic per-fd timers) fall through to the syscall. Critical for: every event loop, latency measurement, scheduler runtime accounting, RCU stall detection, every `gettimeofday`-style consumer.

This Tier-5 covers the userspace ABI of syscall 228. vDSO shared-page layout and timekeeping accumulator are owned by `kernel/time/timekeeping.md` and `kernel/time/vdso.md` (Tier-3, planned).

## Signature

```c
int clock_gettime(clockid_t clk_id, struct timespec *tp);
```

Rust ABI shim:

```rust
pub fn sys_clock_gettime(clk_id: i32, tp: *mut __kernel_timespec) -> isize;
```

Syscall number: **228** (`__NR_clock_gettime` on `x86_64`/`arm64`/`riscv64`).
On 32-bit architectures the time-64-safe variant is `sys_clock_gettime64` (syscall 403).

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `clk_id` | `clockid_t` (`i32`) | IN | clock source identifier — see ABI surface |
| `tp` | `struct __kernel_timespec *` | OUT | writeback of `{tv_sec: i64, tv_nsec: i64}` (`tv_nsec` ∈ `[0, 999_999_999]`) |

## Return

- **Success**: `0` and `*tp` populated.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `clk_id` not a recognized clock; per-CPU clock referencing a dead pid; dynamic posix clock fd closed |
| `EFAULT` | `tp` not in writable userspace |
| `EOPNOTSUPP` | dynamic POSIX clock backend (`/dev/ptp*`) returns ENOTSUP |
| `EPERM` | per-pid CPU clock targeting a task in a different user-ns without `CAP_SYS_PTRACE` |
| `ESRCH` | per-pid CPU clock: target task exited |

## ABI surface

Recognized `clockid_t` values:

| Constant | Value | Source | vDSO? |
|---|---|---|---|
| `CLOCK_REALTIME` | `0` | wall-clock, adjustable by `settimeofday(2)`/`clock_settime(2)`/`adjtimex(2)`, may jump | yes |
| `CLOCK_MONOTONIC` | `1` | monotonic since arbitrary epoch, NTP-disciplined slew, no jumps (excludes suspend) | yes |
| `CLOCK_PROCESS_CPUTIME_ID` | `2` | sum of CPU time consumed by all threads in the calling process | no |
| `CLOCK_THREAD_CPUTIME_ID` | `3` | CPU time consumed by the calling thread | no |
| `CLOCK_MONOTONIC_RAW` | `4` | monotonic, NTP-undisciplined (raw hardware counter) | yes |
| `CLOCK_REALTIME_COARSE` | `5` | last tick's `CLOCK_REALTIME` snapshot (jiffies resolution) | yes |
| `CLOCK_MONOTONIC_COARSE` | `6` | last tick's `CLOCK_MONOTONIC` snapshot (jiffies resolution) | yes |
| `CLOCK_BOOTTIME` | `7` | `CLOCK_MONOTONIC` plus accumulated suspend time | yes |
| `CLOCK_REALTIME_ALARM` | `8` | `CLOCK_REALTIME` that wakes from suspend (alarm timers only) | no |
| `CLOCK_BOOTTIME_ALARM` | `9` | `CLOCK_BOOTTIME` that wakes from suspend (alarm timers only) | no |
| `CLOCK_TAI` | `11` | International Atomic Time (UTC + leap seconds count) | yes |

Per-pid / per-tid CPU clocks: encoded as negative `clockid_t`:

| Encoding | Meaning |
|---|---|
| `MAKE_PROCESS_CPUCLOCK(pid, CPUCLOCK_SCHED)` | per-pid process-wide CPU time |
| `MAKE_THREAD_CPUCLOCK(tid, CPUCLOCK_SCHED)` | per-tid thread CPU time |
| low 3 bits = clock-type: `CPUCLOCK_PROF=0`, `CPUCLOCK_VIRT=1`, `CPUCLOCK_SCHED=2` | |

Dynamic POSIX clocks: `clk_id` derived from `((unsigned int)(~fd) << 3) | CLOCKFD` where `CLOCKFD == 3`; backend is the `posix_clock` registered on the open fd (`/dev/ptp0`, etc.).

`struct __kernel_timespec`:

```c
struct __kernel_timespec {
    __kernel_time64_t tv_sec;     /* signed 64-bit seconds */
    long long         tv_nsec;    /* signed 64-bit nanoseconds, normalized [0, 1e9) */
};
```

## Compatibility contract

REQ-1: `clk_id` validation:
- Reject negative non-CPU-clock encodings outside the per-pid / per-tid format with `-EINVAL`.
- Reject unknown positive `clk_id` values with `-EINVAL`.
- Per-pid / per-tid CPU clock with target task gone: `-ESRCH` (per `posix_cpu_clock_get`).

REQ-2: `tp` writeback:
- MUST be in caller's writable userspace mapping at the moment of writeback, else `-EFAULT`.
- `tv_nsec` MUST be normalized into `[0, 999_999_999]`.
- `tv_sec` is signed; pre-1970 values are valid only for `CLOCK_REALTIME` after a `clock_settime` to before-epoch.

REQ-3: vDSO fastpath:
- For `CLOCK_REALTIME`, `CLOCK_MONOTONIC`, `CLOCK_BOOTTIME`, `CLOCK_TAI`, `CLOCK_MONOTONIC_RAW`, `CLOCK_REALTIME_COARSE`, `CLOCK_MONOTONIC_COARSE`: the libc `clock_gettime` MUST first attempt the vDSO entry; the syscall path is a fallback for forced-CLOEXEC seccomp profiles or unsupported clocks.
- vDSO shared page `struct vdso_data` is updated under seqcount; readers retry on torn read.

REQ-4: `CLOCK_REALTIME` semantics:
- Returns seconds + nanoseconds since 1970-01-01 00:00:00 UTC.
- May jump under `settimeofday(2)`, `clock_settime(2)`, NTP `adjtimex(2)` with step, or hibernation resume.
- Subject to leap-second handling (POSIX UTC: leap second is repeated 23:59:59).

REQ-5: `CLOCK_MONOTONIC` semantics:
- Strictly non-decreasing across reads.
- Slewed (not stepped) by NTP `adjtimex` to track real time.
- Frozen during system suspend; resumes from same value on wakeup.

REQ-6: `CLOCK_MONOTONIC_RAW` semantics:
- Raw hardware clocksource ticks, not NTP-adjusted.
- Strictly non-decreasing.

REQ-7: `CLOCK_BOOTTIME` semantics:
- `CLOCK_MONOTONIC` plus accumulated suspend time delta.
- Strictly non-decreasing.

REQ-8: `CLOCK_TAI` semantics:
- `CLOCK_REALTIME` plus current TAI-UTC offset (typically 37 seconds in 2026).
- Updated via `adjtimex(2)` `ADJ_TAI`.

REQ-9: `CLOCK_REALTIME_COARSE` / `CLOCK_MONOTONIC_COARSE`:
- Returns the most recent tick-time snapshot; resolution is `1/HZ`-bounded.
- Cheap: no hardware-clock read.

REQ-10: CPU-time clocks:
- `CLOCK_PROCESS_CPUTIME_ID`: sum of `utime + stime` across all threads in current's tgid (via `thread_group_cputime`).
- `CLOCK_THREAD_CPUTIME_ID`: sum of `utime + stime` for `current` task, updated per tick.

REQ-11: Per-pid CPU clock:
- `clk_id` encodes `(pid, CPUCLOCK_SCHED/PROF/VIRT)`.
- Cross-pidns pid: requires `CAP_SYS_PTRACE`.

REQ-12: Dynamic POSIX clocks:
- Open `/dev/ptp0`; `clk_id = ((unsigned)~fd << 3) | CLOCKFD`; dispatch to `posix_clock_ops::clock_gettime`. Backend errors propagate.

REQ-13: Time-64 ABI:
- 32-bit architectures use `sys_clock_gettime64` (403) with `__kernel_timespec`; legacy `sys_clock_gettime` truncates to 32-bit `time_t` (deprecated).

REQ-14: Signal safety: async-signal-safe per POSIX.1-2008.

## Acceptance Criteria

- [ ] AC-1: `CLOCK_REALTIME` returns wall-clock matching `gettimeofday(2)` within 1µs.
- [ ] AC-2: `CLOCK_MONOTONIC`: two reads `t1`, `t2` with `t2` after `t1` satisfy `t2 >= t1` strictly.
- [ ] AC-3: `CLOCK_MONOTONIC_RAW` advances at hardware-clocksource rate (no NTP slew).
- [ ] AC-4: `CLOCK_BOOTTIME` increases by suspend duration on resume.
- [ ] AC-5: `CLOCK_TAI` − `CLOCK_REALTIME` equals current TAI-UTC offset.
- [ ] AC-6: `CLOCK_THREAD_CPUTIME_ID` returns 0 immediately at thread spawn; increases under busy loop.
- [ ] AC-7: `CLOCK_PROCESS_CPUTIME_ID` ≥ max thread CPU time within the process.
- [ ] AC-8: `CLOCK_REALTIME_COARSE` resolution ≥ `1/HZ`.
- [ ] AC-9: Unknown `clk_id` (e.g., `42`) returns `-EINVAL`.
- [ ] AC-10: `tp == NULL` returns `-EFAULT`.
- [ ] AC-11: Per-pid CPU clock targeting non-existent pid returns `-ESRCH`.
- [ ] AC-12: vDSO `CLOCK_MONOTONIC` matches syscall path within 1µs.
- [ ] AC-13: Cross-pidns per-pid CPU clock without `CAP_SYS_PTRACE` returns `-EPERM`.

## Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
#[repr(i32)]
pub enum ClockId {
    Realtime         = 0,
    Monotonic        = 1,
    ProcessCpuTimeId = 2,
    ThreadCpuTimeId  = 3,
    MonotonicRaw     = 4,
    RealtimeCoarse   = 5,
    MonotonicCoarse  = 6,
    Boottime         = 7,
    RealtimeAlarm    = 8,
    BoottimeAlarm    = 9,
    Tai              = 11,
}

pub const CPUCLOCK_PROF:  u32 = 0;
pub const CPUCLOCK_VIRT:  u32 = 1;
pub const CPUCLOCK_SCHED: u32 = 2;
pub const CPUCLOCK_PERTHREAD_MASK: u32 = 4;
pub const CLOCKFD:        u32 = 3;
pub const CLOCKFD_MASK:   u32 = 0x7;
```

`PosixTimers::clock_gettime(clk_id, tp_user) -> isize`:
1. /* CPU-clock encoding */
2. if (clk_id as i32) < 0 || (clk_id as u32 & CLOCKFD_MASK) == CLOCKFD:
   - return posix_cpu_clock_get_or_dynamic(clk_id, tp_user).
3. /* Normal clock dispatch */
4. let ts = match ClockId::try_from(clk_id) {
   - Realtime         => Timekeeper::get_real_ts64(),
   - Monotonic        => Timekeeper::get_ts64(),
   - MonotonicRaw     => Timekeeper::get_raw_ts64(),
   - RealtimeCoarse   => Timekeeper::get_coarse_real_ts64(),
   - MonotonicCoarse  => Timekeeper::get_coarse_ts64(),
   - Boottime         => Timekeeper::get_boottime_ts64(),
   - Tai              => Timekeeper::get_clocktai_ts64(),
   - ProcessCpuTimeId => Cpu::thread_group_cputime(current.tgid),
   - ThreadCpuTimeId  => Cpu::task_cputime(current),
   - RealtimeAlarm    => return -EINVAL,
   - BoottimeAlarm    => return -EINVAL,
   - _ => return -EINVAL,
 };
5. /* writeback */
6. copy_to_user(tp_user, &ts).map_err(|_| -EFAULT)?;
7. return 0;

`Timekeeper::get_real_ts64() -> KernelTimespec` reads `tk_core.tk` under `read_seqcount_begin`/`_retry`, computes `sec + (cyc_delta * mult >> shift) + xtime_nsec`, normalizes `tv_nsec` into `[0, 1e9)`, and returns the timespec. The vDSO entry `__vdso_clock_gettime` (in `arch/x86/entry/vdso/vclock_gettime.c`) mirrors this formula against `vdso_data` — a `PROT_READ` shared page mapped into every process.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `clk_id_known` | INVARIANT | unknown `clk_id` ⟹ -EINVAL |
| `tp_user_writable` | INVARIANT | per-copy_to_user: bad `tp` ⟹ -EFAULT, no kernel write to non-userspace |
| `monotonic_nondecreasing` | INVARIANT | two `get_ts64` reads under same boot: `t2 >= t1` |
| `nsec_normalized` | INVARIANT | post: `tv_nsec ∈ [0, 999_999_999]` |
| `seqcount_retry_terminates` | INVARIANT | bounded retries under writer pressure (configurable cap) |
| `cputime_perpid_ns_check` | INVARIANT | per-pid clock cross-ns ⟹ CAP_SYS_PTRACE enforced |

### Layer 2: TLA+

`uapi/clock_gettime.tla`:
- Variables: `tk_core.tk` (real epoch, monotonic offset, mult/shift), `cycle_counter`, `seq`.
- Properties:
  - `safety_monotonic_nondecreasing` — successive `CLOCK_MONOTONIC` reads are non-decreasing.
  - `safety_boottime_includes_suspend` — per-suspend delta accumulator strict.
  - `safety_tai_realtime_offset_constant_modulo_adjtimex` — `TAI − REALTIME` changes only on `adjtimex(ADJ_TAI)`.
  - `liveness_seqcount_eventually_quiescent` — writer eventually stops; readers terminate.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `clock_gettime` post(0): `*tp` normalized | `PosixTimers::clock_gettime` |
| `get_real_ts64` post: matches `xtime_sec + xtime_nsec + cyc_delta * mult >> shift` | `Timekeeper::get_real_ts64` |
| `posix_cpu_clock_get` post: returns task or thread-group cputime sum | `PosixCpuClock::get` |
| dynamic posix clock dispatches to `posix_clock_ops` | `PosixClock::dispatch` |

### Layer 4: Verus/Creusot functional

Per-`clock_gettime(2)` man page, `Documentation/core-api/timekeeping.rst`, glibc `sysdeps/unix/sysv/linux/clock_gettime.c`, Rust std `sys::unix::time::Instant::now`, Go runtime `time·now`. Per-LTP `testcases/kernel/syscalls/clock_gettime/*` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

clock_gettime reinforcement:

- **Per-clk_id whitelist** — unknown values rejected with `-EINVAL`; no fall-through.
- **Per-CPU-clock pidns gating** — cross-pidns per-pid clocks enforce `CAP_SYS_PTRACE`.
- **Per-seqcount writer-side fence** — `xtime_update` under `raw_spin_lock_irqsave(&tk_core.lock)`; readers never see torn `(sec, nsec)`.
- **Per-vDSO read-only page + tv_nsec normalization** — `vdso_data` mapped `PROT_READ`; output `tv_nsec ∈ [0, 1e9)` invariant.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on every `clock_gettime` syscall** — clock_gettime is one of the highest-frequency syscalls in modern workloads (every `Instant::now`, every `time.Now()`); randomizing kernel stack on each entry mitigates layout-discovery attacks that rely on stable stack pointer values across many syscalls.
- **PaX UDEREF on the `tp` writeback** — `copy_to_user(tp, &ts)` MUST refuse kernel-mapped pages disguised as userspace pointers; type-confused writes of timekeeper-derived data are a known kernel-write primitive.
- **GRKERNSEC_CLOCK_RESOLUTION** — under hardened policy, suid-binaries and `CAP_SYS_NICE`-less processes see coarsened `CLOCK_MONOTONIC` / `CLOCK_REALTIME` (rounded to `1/HZ` instead of nanosecond resolution); defense against high-precision timing side channels (Spectre v1/v4 measurement, cache timing attacks, FLUSH+RELOAD). Tunable: `grsec_clock_resolution_ns` (default 4096 ns under hardened profile, 1 ns under permissive).
- **CAP_SYS_TIME orthogonal here** — `clock_gettime` reads but never writes; CAP_SYS_TIME is enforced only on the `_settime`/`_adjtime` siblings, never on read paths.
- **CAP_WAKE_ALARM N/A for read** — `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM` are rejected from `clock_gettime` itself (`-EINVAL`); they exist only as `timer_create`/`timerfd_create` clock sources. The capability gate applies there, not here.
- **adjtimex rate-limit** — companion-syscall rate-limiting (in `adjtimex.md`) prevents a privileged attacker from rapidly mutating clock parameters to create timing oracles observable via `clock_gettime`; per-uid rate-limit of clock-mutating syscalls applied at write side keeps read-side timing predictable.
- **Per-pidns CPU-clock denial** — per-pid CPU clock targeting tasks outside caller's pidns: refused with `-EPERM` regardless of CAP_SYS_PTRACE under `GRKERNSEC_PROC_USERGROUP` to prevent CPU-time fingerprinting of victims across namespaces.
- **GRKERNSEC_HIDESYM + vDSO ASLR** — `CLOCKFD`-encoded `posix_clock_ops` pointers opaque to userspace; `vdso_data` mapped at a per-process randomized address (complements PAX_RANDMMAP).
- **Seqcount bounded retry counter** — `read_seqcount_retry` capped at 1024 iterations; on cap-hit falls through to syscall path and audit-logs the retry storm (defense against priority-inverted writer livelock against high-RT readers).

## Open Questions

(none)

## Out of Scope

- `kernel/time/timekeeping.md` / `kernel/time/vdso.md` / `kernel/time/posix_cpu_timers.md` Tier-3.
- `clock_settime.md` / `clock_getres.md` / `clock_nanosleep.md` / `adjtimex.md` siblings.
- glibc / musl userspace `clock_gettime` wrappers.
- Implementation code.
