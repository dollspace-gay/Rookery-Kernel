---
title: "Tier-2: kernel/time — timekeeping, hrtimer, posix-timers, clocksource, tick"
tags: ["design-doc", "tier-2", "kernel", "time"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Tier-2 overview for the kernel time subsystem — every clock + timer + tick mechanism that other subsystems depend on. Heavily cross-referenced from `kernel/sched/*`, `net/sched/sch-fq-codel.md`, `net/sched/sch-fq.md`, `net/sched/sch-taprio.md`, `net/xfrm/state.md` (lifetime timers), `net/netfilter/conntrack-core.md` (timeout GC), `kernel/cgroup/freezer.md`.

Components:
- **Timekeeping** (`timekeeping.c`): wall clock, CLOCK_MONOTONIC / _BOOTTIME / _RAW / _TAI / _REALTIME maintenance; NTP discipline integration; per-CPU sched_clock fast path
- **hrtimer** (`hrtimer.c`): high-resolution timer subsystem; per-CPU rb-tree of pending timers; clockid-keyed dispatch (CLOCK_MONOTONIC / _REALTIME / _BOOTTIME / _TAI)
- **timer** (`timer.c`): legacy jiffy-resolution timer wheel (still used for sched-tick + per-task itimers)
- **posix-timers** (`posix-timers.c`, `posix-cpu-timers.c`): POSIX `timer_create(2)` + per-process/per-task CPU-time timers (RLIMIT_CPU + ITIMER_VIRTUAL/PROF)
- **clocksource** (`clocksource.c`): per-arch clocksource registration (TSC / HPET / ACPI-PMT / KVM-clock / etc.) + watchdog
- **clockevents** (`clockevents.c`): per-CPU clock-event-device registration (LAPIC-timer / HPET / per-arch)
- **tick** (`tick-common.c`, `tick-sched.c`, `tick-oneshot.c`, `tick-broadcast.c`): per-CPU periodic / one-shot / nohz_idle / nohz_full tick mode
- **alarmtimer** (`alarmtimer.c`): RTC-wakeup-capable timers for `clock_*(2)` with CLOCK_BOOTTIME_ALARM / _REALTIME_ALARM
- **time-namespace** (`namespace.c`, `namespace_vdso.c`): per-task CLOCK_MONOTONIC + CLOCK_BOOTTIME offsets (cross-ref `fs/proc/proc-namespaces.md`)
- **NTP** (`ntp.c`): kernel-side NTP discipline (`adjtimex(2)` + per-tick frequency adjustments)
- **sched_clock** (`sched_clock.c`): per-CPU monotonic counter (used by tracing + per-CPU runtime accounting)
- **timer_migration** (`timer_migration.c`): per-CPU timer-migration to least-loaded CPU (CONFIG_TIMER_MIGRATION)
- **timer_list** (`timer_list.c`): /proc/timer_list reporter

Sub-tier-2 of `kernel/00-overview.md`. Pairs with `kernel/sched/*` (timekeeping consumed by sched-clock), `arch/x86/kernel-platform.md` (TSC + HPET + LAPIC), `drivers/clocksource/` + `drivers/clk/` once added.

### Out of Scope

- Per-Tier-3 (Phase D)
- 32-bit-only paths
- Implementation code

### scope

This Tier-2 governs all of `/home/doll/linux-src/kernel/time/` (~25 source files) plus public API + UAPI headers.

### compatibility contract — outline

### `clock_gettime(2)` + `clock_settime(2)` + `clock_nanosleep(2)` + `clock_getres(2)`

Standard POSIX clock IDs:
- `CLOCK_REALTIME` (wall clock; settable; subject to NTP discipline)
- `CLOCK_MONOTONIC` (post-boot monotonic; subject to NTP-rate adjustment)
- `CLOCK_BOOTTIME` (post-boot monotonic INCLUDING suspend time)
- `CLOCK_REALTIME_COARSE` / `_MONOTONIC_COARSE` (low-precision via vDSO)
- `CLOCK_MONOTONIC_RAW` (post-boot monotonic; no NTP rate-adjust)
- `CLOCK_TAI` (atomic time; offset from REALTIME by leap-second count)
- `CLOCK_PROCESS_CPUTIME_ID` (per-process CPU time)
- `CLOCK_THREAD_CPUTIME_ID` (per-thread CPU time)
- `CLOCK_BOOTTIME_ALARM` / `_REALTIME_ALARM` (wakeup-capable alarmtimer-backed)

Wire format byte-identical for all syscalls.

### `timer_create(2)` + `timer_settime(2)` + `timer_gettime(2)` + `timer_delete(2)` + `timer_getoverrun(2)` (POSIX timers)

Per-process timer creation; backed by hrtimer for CLOCK_MONOTONIC/_REALTIME/_TAI/_BOOTTIME or by alarmtimer for CLOCK_*_ALARM. Per-timer ID via per-process `signal->posix_timers` IDR.

### `setitimer(2)` + `getitimer(2)` (legacy interval timers)

3 timer types: `ITIMER_REAL` (CLOCK_REALTIME-backed), `ITIMER_VIRTUAL` (per-thread user-CPU time), `ITIMER_PROF` (per-thread user+kernel CPU time). Fires SIGALRM / SIGVTALRM / SIGPROF.

### `nanosleep(2)` + `clock_nanosleep(2)`

Sleep with hrtimer; supports absolute (`TIMER_ABSTIME`) or relative; signal-interruptible.

### `getrlimit(2)` / `setrlimit(2)` `RLIMIT_CPU` enforcement

Per-process CPU-time hard cap; enforced by posix-cpu-timers with SIGXCPU at soft limit then SIGKILL at hard limit.

### `adjtimex(2)`

NTP discipline — userspace ntpd / chronyd adjusts kernel time-tracking via this syscall. Modes: ADJ_OFFSET / ADJ_FREQUENCY / ADJ_MAXERROR / ADJ_ESTERROR / ADJ_STATUS / ADJ_TIMECONST / ADJ_TICK / ADJ_TAI / ADJ_SETOFFSET / ADJ_NANO / ADJ_MICRO.

### Per-clocksource sysfs

`/sys/devices/system/clocksource/clocksource0/`:
- `available_clocksource` — readable list
- `current_clocksource` — current selected (writable: switch)
- `unbind_clocksource` — disable current

### Per-CPU tick state

`/proc/timer_list` shows per-CPU clockevent + per-cpu hrtimer + clocksource state. Format byte-identical so `timer-list-parse`-style tools work.

### `/proc/sys/kernel/`

- `timer_migration` — enable/disable per-CPU timer migration
- `nmi_watchdog`, `hung_task_*` — related but lives in `kernel/watchdog.c` (cross-ref future Tier-3)

### `/proc/timer_list`

Per-CPU enumeration of pending hrtimers + clockevent devices. Format byte-identical.

### Time namespace (per-task offsets)

Per-time-namespace `offsets[]` indexed by `[CLOCK_MONOTONIC, CLOCK_BOOTTIME]`. `clock_gettime(CLOCK_MONOTONIC)` from a task in a non-root time-ns returns kernel-time + ns-offset. Used by `unshare --time` for container time-virtualization.

Cross-ref `fs/proc/proc-namespaces.md` REQ-11.

### vDSO integration

CLOCK_GETTIME / CLOCK_GETRES / GETTIMEOFDAY exposed to userspace via vDSO (cross-ref `arch/x86/vdso.md`); per-arch vDSO function reads kernel timekeeper data via lockless seqcount.

### tier-3 docs governed by this tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `kernel/time/timekeeping.md` | wall clock + monotonic clocks + NTP discipline |
| `kernel/time/hrtimer.md` | high-resolution timer subsystem |
| `kernel/time/timer.md` | legacy timer wheel |
| `kernel/time/posix-timers.md` | POSIX timer_create/_settime |
| `kernel/time/posix-cpu-timers.md` | per-task CPU-time timers + RLIMIT_CPU |
| `kernel/time/clocksource.md` | clocksource registration + watchdog |
| `kernel/time/clockevents.md` | clockevent device registration |
| `kernel/time/tick.md` | tick (periodic / one-shot / nohz) |
| `kernel/time/alarmtimer.md` | RTC-wakeup-capable timers |
| `kernel/time/time-namespace.md` | per-task time offsets |
| `kernel/time/ntp.md` | adjtimex + NTP discipline |

### compatibility outline (top-level)

- REQ-O1: All POSIX clock_* syscalls + clock IDs identical wire format.
- REQ-O2: timer_* + setitimer/getitimer / nanosleep / adjtimex syscalls identical.
- REQ-O3: Per-clocksource sysfs (`/sys/devices/system/clocksource/`) byte-identical content + writable knobs.
- REQ-O4: `/proc/timer_list` format byte-identical.
- REQ-O5: vDSO-via-kernel coordination for fast clock_gettime path identical.
- REQ-O6: Time namespace per-clock offsets identical.
- REQ-O7: TLA+ models declared at this Tier-2 (timekeeping seqcount safety, hrtimer rb-tree integrity).
- REQ-O8: Hardening: row-1 features applied per `00-security-principles.md`.

### acceptance criteria (top-level)

- [ ] AC-O1: clock_gettime / settime / nanosleep round-trip equivalent to upstream. (covers REQ-O1, REQ-O2)
- [ ] AC-O2: chronyd / ntpd discipline kernel time correctly via adjtimex. (covers REQ-O2)
- [ ] AC-O3: `cat /sys/devices/system/clocksource/clocksource0/available_clocksource` lists TSC / HPET / KVM-clock as expected. (covers REQ-O3)
- [ ] AC-O4: `/proc/timer_list` byte-identical content. (covers REQ-O4)
- [ ] AC-O5: vDSO clock_gettime is < 100ns on x86_64 with TSC. (covers REQ-O5)
- [ ] AC-O6: time-ns test: `unshare --time --boottime-offset=86400`; clock_gettime(CLOCK_BOOTTIME) returns ns-offset value. (covers REQ-O6)

### verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/time/timekeeping_seqcount.tla` | `kernel/time/timekeeping.md` (proves: vDSO seqcount-based readers never see torn state; writer ↔ reader concurrency is correct) |
| `models/time/hrtimer_rbtree.tla` | `kernel/time/hrtimer.md` (proves: per-CPU rb-tree maintains earliest-expiry-leftmost invariant; concurrent insert/cancel under per-base lock) |

### hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-posix_timer + per-alarmtimer refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-clocksource + per-clockevent registrations `static const` | § Mandatory |
| **SIZE_OVERFLOW** | timekeeping arithmetic + hrtimer expires-comparison uses checked operators | § Mandatory |
| **MEMORY_SANITIZE** | freed posix-timer state cleared (carries per-task signal target) | § Default-on configurable off |

