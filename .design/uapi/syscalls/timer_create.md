# Tier-5 syscall: timer_create(2) — syscall 222

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/posix-timers.c (sys_timer_create, do_timer_create)
  - kernel/time/alarmtimer.c (alarm_timer_create)
  - include/uapi/linux/time.h (clockid_t, sigevent, timer_t)
  - include/uapi/asm-generic/siginfo.h (sigevent)
  - arch/*/include/generated/uapi/asm/unistd_64.h (222)
  - Documentation/core-api/timekeeping.rst, man timer_create(2)
-->

## Summary

`timer_create(2)` allocates a kernel-resident POSIX per-process interval timer bound to a clock source and a delivery mechanism (signal, thread-targeted signal, or thread function callback expressed via `sigevent`). The returned opaque `timer_t` identifier names the timer for subsequent `timer_settime(2)`/`timer_gettime(2)`/`timer_getoverrun(2)`/`timer_delete(2)` calls. Unlike `setitimer(2)` which provides three fixed `ITIMER_*` slots per process, `timer_create` permits an unbounded (resource-limit-gated) collection of named timers, each independently armed. Delivery via `SIGEV_SIGNAL` queues a real-time signal carrying the timer ID in `siginfo`; `SIGEV_THREAD_ID` targets a specific TID; `SIGEV_NONE` defers delivery (state only readable via `timer_gettime`); `SIGEV_THREAD` is userspace-only (glibc helper thread) and the kernel sees `SIGEV_SIGNAL`. Critical for: glibc `pthread_cond_timedwait`, real-time scheduler deadline tracking, telecom/industrial-control deterministic periodic actions, container runtimes implementing per-cgroup soft deadlines, `cron`-class single-shot scheduling.

This Tier-5 covers the userspace ABI of syscall 222 (creation only); programming, querying, overrun, and deletion are siblings 223-226. Per-`alarmtimer` backend for `CLOCK_REALTIME_ALARM`/`CLOCK_BOOTTIME_ALARM` is owned by `kernel/time/alarmtimer.md` (Tier-3, planned).

## Signature

```c
int timer_create(clockid_t clockid, struct sigevent *sevp, timer_t *timerid);
```

Rust ABI shim:

```rust
pub fn sys_timer_create(clockid: i32,
                        sevp: *mut sigevent,
                        timerid: *mut timer_t) -> isize;
```

Syscall number: **222**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `clockid` | `clockid_t` (`i32`) | IN | clock source (REALTIME, MONOTONIC, BOOTTIME, REALTIME_ALARM, BOOTTIME_ALARM, CPUTIME variants, dynamic CLOCKFD) |
| `sevp` | `struct sigevent *` | IN/OUT | delivery descriptor; `NULL` ⟹ default `SIGEV_SIGNAL`/`SIGALRM` with `sival_int = timer_id` |
| `timerid` | `timer_t *` | OUT | newly-allocated opaque timer handle |

## Return

- **Success**: `0`; `*timerid` populated.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno; `*timerid` untouched on failure.

## Errors

| errno | Trigger |
|---|---|
| `EAGAIN` | per-uid `RLIMIT_SIGPENDING` or per-process POSIX-timer slot exhausted |
| `EINVAL` | unsupported `clockid`; `sigev_notify` not in {`SIGEV_SIGNAL`, `SIGEV_NONE`, `SIGEV_THREAD_ID`}; `sigev_signo` ∉ `[1, _NSIG]`; `SIGEV_THREAD_ID` with non-existent TID |
| `EFAULT` | `sevp` or `timerid` outside writable userspace |
| `ENOMEM` | kernel cannot allocate `struct k_itimer` |
| `EPERM` | alarm-class clock requested without `CAP_WAKE_ALARM` |
| `ENOTSUP` | dynamic POSIX clock backend refuses timer creation |

## ABI surface

`struct sigevent` layout (UAPI):

```c
struct sigevent {
    sigval_t  sigev_value;          /* int / void* payload */
    int       sigev_signo;          /* signal number (SIGEV_SIGNAL/_THREAD_ID) */
    int       sigev_notify;         /* SIGEV_SIGNAL/_NONE/_THREAD/_THREAD_ID */
    union {
        int   _pad[12];
        int   _tid;                  /* SIGEV_THREAD_ID target TID */
        struct { void (*function)(sigval_t); void *attribute; } _sigev_thread;
    } _sigev_un;
};
```

Recognized `sigev_notify` values:

| Value | Kernel behavior |
|---|---|
| `SIGEV_SIGNAL` (0) | queue real-time signal `sigev_signo` to process |
| `SIGEV_NONE` (1) | no notification; state polled via `timer_gettime` |
| `SIGEV_THREAD` (2) | kernel treats as `SIGEV_SIGNAL`; glibc spawns helper thread |
| `SIGEV_THREAD_ID` (4) | queue signal to specific TID (must be in process) |

`timer_t` is opaque (typically `int` index into `current->signal->posix_timers` IDR).

## Compatibility contract

REQ-1: `clockid` validation:
- Accepted: `CLOCK_REALTIME`, `CLOCK_MONOTONIC`, `CLOCK_BOOTTIME`, `CLOCK_REALTIME_ALARM`, `CLOCK_BOOTTIME_ALARM`, `CLOCK_PROCESS_CPUTIME_ID`, `CLOCK_THREAD_CPUTIME_ID`, CLOCKFD-encoded dynamic clocks.
- Coarse variants (`_COARSE`, `_RAW`) rejected with `-EINVAL`.

REQ-2: Alarm-clock gating:
- `CLOCK_REALTIME_ALARM` / `CLOCK_BOOTTIME_ALARM` require `CAP_WAKE_ALARM` in current userns; else `-EPERM`.

REQ-3: `sigev_notify` validation:
- `sigev_notify == SIGEV_THREAD_ID`: `_tid` MUST identify a live task sharing `current->signal`; else `-EINVAL`.
- `sigev_notify` not in supported set: `-EINVAL`.

REQ-4: Signal number validation:
- For `SIGEV_SIGNAL`/`SIGEV_THREAD_ID`: `sigev_signo ∈ [1, _NSIG]` and not `SIGKILL`/`SIGSTOP`; else `-EINVAL`.
- Real-time signals (`SIGRTMIN..SIGRTMAX`) are conventional but not required.

REQ-5: NULL `sevp`:
- Default `sigevent`: `sigev_notify = SIGEV_SIGNAL`, `sigev_signo = SIGALRM`, `sigev_value.sival_int = timer_id`.

REQ-6: Allocation:
- `posix_timer_add()` allocates `struct k_itimer` and assigns a free `timer_id` via `current->signal->posix_timers` IDR.
- Per-process slot count gated by `RLIMIT_SIGPENDING` (shared budget).
- `k_itimer` references the chosen `clock_kclock` (`posix_clock_kclocks[]` table) and ALARM-class is dispatched through `alarm_timer_create`.

REQ-7: IDR uniqueness:
- Returned `timer_id` MUST be unique within the process; preserved across `fork()` only when child inherits (POSIX timers are NOT inherited; child starts with empty IDR).

REQ-8: Initial state:
- Timer is created in **disarmed** state; first `timer_settime` arms it.
- `it_interval` and `it_value` zero-initialized.
- `it_overrun_last` zero; `it_requeue_pending` clear.

REQ-9: `timerid` write:
- Write to userspace MUST occur only after all kernel state is committed.
- `copy_to_user` failure after allocation: kernel frees `k_itimer` and returns `-EFAULT`.

REQ-10: Fork/exec semantics:
- POSIX timers are NOT inherited across `fork`; `exec` deletes all timers via `exit_itimers`.

## Acceptance Criteria

- [ ] AC-1: `timer_create(CLOCK_MONOTONIC, NULL, &tid)` returns 0; subsequent `timer_settime(tid)` arms timer.
- [ ] AC-2: `clockid == CLOCK_REALTIME_COARSE` returns `-EINVAL`.
- [ ] AC-3: `CLOCK_REALTIME_ALARM` without `CAP_WAKE_ALARM` returns `-EPERM`.
- [ ] AC-4: `sigev_notify = 99` returns `-EINVAL`.
- [ ] AC-5: `SIGEV_THREAD_ID` with foreign-process TID returns `-EINVAL`.
- [ ] AC-6: `sigev_signo = SIGKILL` returns `-EINVAL`.
- [ ] AC-7: NULL `timerid` returns `-EFAULT`.
- [ ] AC-8: Per-process timer count up to RLIMIT-bound; (count+1)th returns `-EAGAIN`.
- [ ] AC-9: `fork`: child has empty timer set; parent timers unaffected.
- [ ] AC-10: `exec`: all timers deleted prior to new image start.
- [ ] AC-11: NULL `sevp`: default SIGEV_SIGNAL/SIGALRM with sival_int = timer_id.

## Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
pub fn sys_timer_create(clockid: i32,
                        sevp: *mut Sigevent,
                        timerid: *mut TimerId) -> isize {
    let sev = if sevp.is_null() {
        Sigevent::default_sigalrm()
    } else {
        match copy_from_user::<Sigevent>(sevp) {
            Ok(s) => s,
            Err(_) => return -EFAULT,
        }
    };
    if !Sigevent::is_supported_notify(sev.sigev_notify) { return -EINVAL; }
    if matches!(clockid, CLOCK_REALTIME_ALARM | CLOCK_BOOTTIME_ALARM)
        && !capable(CAP_WAKE_ALARM) { return -EPERM; }
    PosixTimers::do_timer_create(clockid, sev, timerid)
}
```

`PosixTimers::do_timer_create(clk, sev, out) -> isize`:
1. /* dispatch on clock */
2. let kc = match clk {
   - REALTIME | MONOTONIC | BOOTTIME | PROCESS_CPUTIME_ID | THREAD_CPUTIME_ID
       => &posix_clock_kclocks[clk as usize],
   - REALTIME_ALARM | BOOTTIME_ALARM => &alarm_clock_kclock,
   - _ if (clk as u32 & CLOCKFD_MASK) == CLOCKFD => return PosixClock::dispatch_create(clk, sev, out),
   - _ => return -EINVAL,
 };
3. /* allocate timer slot */
4. let mut k = KItimer::alloc().ok_or(-ENOMEM)?;
5. k.it_clock     = clk;
6. k.it_signal    = current.signal.clone();
7. k.it_sigev     = sev;
8. k.it_overrun   = 0;
9. k.it_requeue_pending = 0;
10. /* per-class init */
11. if let Err(e) = kc.timer_create(&mut k) { KItimer::free(k); return e; }
12. /* IDR insert */
13. let id = match posix_timer_add(&current.signal, &mut k) {
    - Ok(id) => id,
    - Err(_) => { kc.timer_del(&mut k); KItimer::free(k); return -EAGAIN; }
  };
14. /* publish */
15. if copy_to_user(out, &id).is_err() {
    - posix_timer_del(&current.signal, id);
    - kc.timer_del(&mut k);
    - KItimer::free(k);
    - return -EFAULT;
  }
16. 0

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `udref_on_sigevent_copy` | INVARIANT | sevp read uses access_ok + checked copy |
| `cap_wake_alarm_enforced` | INVARIANT | alarm-clock without CAP_WAKE_ALARM ⟹ -EPERM |
| `idr_unique_per_process` | INVARIANT | per-process: returned ids never collide |
| `kitimer_freed_on_failure` | INVARIANT | post-allocation failure ⟹ no leak |
| `default_sigevent_well_formed` | INVARIANT | NULL sevp ⟹ SIGALRM/SIGEV_SIGNAL |

### Layer 2: TLA+

`uapi/timer_create.tla`:
- Variables: `posix_timers[uid]`, `idr_max`, `cap_set`, `rlimit_sigpending`.
- Properties:
  - `safety_no_kid_leak_on_efault` — copy_to_user failure path frees k_itimer.
  - `safety_alarm_requires_cap_wake_alarm` — ALARM clock + ¬CAP_WAKE_ALARM ⟹ -EPERM.
  - `safety_rlimit_honored` — beyond RLIMIT_SIGPENDING ⟹ -EAGAIN.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_timer_create` post(0): IDR contains new id; `*out == id` | `PosixTimers::do_timer_create` |
| `do_timer_create` post(err): no IDR entry; no k_itimer | `PosixTimers::do_timer_create` |
| alarm path post: dispatches via `alarm_timer_create` | dispatch |
| default sevp post: `sigev_signo == SIGALRM` | `Sigevent::default_sigalrm` |

### Layer 4: Verus/Creusot functional

Per-`timer_create(2)` man page, glibc `sysdeps/unix/sysv/linux/timer_create.c`, LTP `testcases/kernel/syscalls/timer_create/*`, real-world callers (`pthread_cond_timedwait`, `chronyd`).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

timer_create reinforcement:

- **Per-RLIMIT_SIGPENDING strict** — timer slot allocation gated; cannot exceed per-uid quota.
- **CAP_WAKE_ALARM for alarm clocks** — strictly checked in current userns.
- **IDR per-signal-struct** — timer IDs scoped to thread group; no cross-process spoofing.
- **`exit_itimers` on exec/exit** — every k_itimer freed; no dangling delivery.
- **Default sevp well-formed** — NULL sevp yields a sane SIGEV_SIGNAL/SIGALRM record.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF on `sevp`** — `copy_from_user(&sev, sevp)` refuses kernel-mapped addresses; defense against type-confused write where attacker tricks the kernel into reading a forged sigevent placing `sigev_signo` outside the process or pointing `_sigev_thread.function` into kernel text.
- **PaX UDEREF on `timerid`** — `copy_to_user(timerid, &id)` validates the destination is a true userspace mapping; defense against probe-confusion of `timer_t` writes overwriting kernel state.
- **GRKERNSEC_POSIX_TIMER_QUOTA per-uid** — hardened policy installs a per-uid POSIX-timer cap (default: 1024) below RLIMIT_SIGPENDING; defense against unprivileged users exhausting the global signal pending budget by creating tens of thousands of timers.
- **CAP_WAKE_ALARM init-userns gating** — under hardened policy, alarm-class clocks require CAP_WAKE_ALARM in the **init** user namespace; container roots with CAP_WAKE_ALARM in a non-init userns are denied. Prevents abuse of `CLOCK_BOOTTIME_ALARM` to keep the system awake from suspend.
- **PAX_RANDKSTACK on every entry** — random kstack on each `timer_create` denies layout discovery; relevant because POSIX-timer paths interleave with signal delivery and kallsyms-adjacent helpers.
- **GRKERNSEC_HIDESYM on `posix_clock_kclocks`** — function pointer table excluded from `kallsyms`; defense against rop-style abuse of the per-clock `kclock_ops` indirect calls.
- **Audit on timer creation** — successful `timer_create` logs `(uid, pid, clockid, sigev_notify, sigev_signo, tid)` to the audit subsystem; defenders can detect timer-storm attempts and trace which TID a SIGEV_THREAD_ID targets.
- **Reject `sigev_notify == SIGEV_THREAD_ID` targeting init/PID-1 even within same signal struct** — hardened policy rejects any SIGEV_THREAD_ID where target task has `PF_KTHREAD` set or is PID 1 of the init userns, even if `current->signal` matches; defense against signal-storm attacks on system-critical threads.
- **Refuse SIGEV_SIGNAL with SIGRTMIN..SIGRTMIN+3** — these signals are reserved for glibc/musl threading internals; hardened policy denies them with `-EINVAL` to prevent malicious processes from poisoning the libc real-time-signal pipeline.
- **Per-clockid audit-route partition** — alarm-clock timers logged under a distinct audit subtype so SIEM can flag "CLOCK_REALTIME_ALARM" allocations independently from MONOTONIC timers, which are far more common.
- **No-cross-userns sigev_value pointer payload validation** — `sigev_value.sival_ptr` is a userspace cookie only; hardened policy zeros the high bits of `sival_ptr` if userspace falsely sets kernel-range bits to prevent pointer-leak fingerprinting via siginfo.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `timer_settime.md`, `timer_gettime.md`, `timer_getoverrun.md`, `timer_delete.md` siblings.
- `kernel/time/alarmtimer.md` Tier-3: alarm-class timer backend.
- `kernel/time/posix_timers.md` Tier-3: IDR management, k_itimer lifecycle, dispatch tables.
- `signalfd.md` / `rt_sigtimedwait.md` consumer-side signal delivery.
- glibc `pthread_cond_timedwait` integration.
- Implementation code.
