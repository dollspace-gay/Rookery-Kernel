# Tier-5 syscall: setitimer(2) — syscall 38

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/time/itimer.c (sys_setitimer, do_setitimer)
  - kernel/time/hrtimer.c (hrtimer_start, hrtimer_cancel — ITIMER_REAL)
  - kernel/time/posix-cpu-timers.c (set_process_cpu_timer — ITIMER_VIRTUAL/PROF)
  - kernel/signal.c (send_sig — SIGALRM/SIGVTALRM/SIGPROF delivery)
  - include/uapi/linux/time.h (struct itimerval, ITIMER_*)
  - arch/*/include/generated/uapi/asm/unistd_64.h (38)
  - Documentation/core-api/timekeeping.rst, man setitimer(2)
-->

## Summary

`setitimer(2)` arms or disarms one of three per-process **interval timers** (`ITIMER_REAL` / `ITIMER_VIRTUAL` / `ITIMER_PROF`) by writing `(it_value, it_interval)`. On expiry the kernel delivers the configured signal (`SIGALRM`/`SIGVTALRM`/`SIGPROF`) to the process; if `it_interval != {0,0}` the timer rearms with that period, otherwise it becomes one-shot and disarms. Returning the previous `(it_value, it_interval)` via the optional `old_value` argument enables save/restore patterns. It is the classic UNIX V7 interval-timer write API, behind `alarm(2)` (`ITIMER_REAL` convenience wrapper), `gprof`'s sampling logic (`ITIMER_PROF`), `ulimit -t` enforcement glue, and pre-POSIX-timer code. Critical for: gprof-style profiling, simple periodic-signal code, source compatibility with POSIX.1 and SVID.

This Tier-5 covers the userspace ABI of syscall 38. The hrtimer-backed `ITIMER_REAL` armament and posix-cpu-timer `ITIMER_VIRTUAL`/`PROF` accounting are owned by `kernel/time/itimer.md` (Tier-3, planned).

## Signature

```c
int setitimer(int which,
              const struct itimerval *new_value,
              struct itimerval *old_value);
```

Rust ABI shim:

```rust
pub fn sys_setitimer(which: i32,
                    new_value: *const __kernel_old_itimerval,
                    old_value: *mut __kernel_old_itimerval) -> isize;
```

Syscall number: **38**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `which` | `i32` | IN | `ITIMER_REAL` (0), `ITIMER_VIRTUAL` (1), `ITIMER_PROF` (2) |
| `new_value` | `const struct itimerval *` | IN | `it_value` = initial residual (`{0,0}` disarms); `it_interval` = reload after expiry (`{0,0}` = one-shot) |
| `old_value` | `struct itimerval *` | OUT (optional) | previous state if non-NULL |

## Return

- **Success**: `0`; timer reprogrammed.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `which` ∉ `{ITIMER_REAL, ITIMER_VIRTUAL, ITIMER_PROF}`; `tv_usec ∉ [0, 999_999]`; `tv_sec` negative or exceeds kernel range |
| `EFAULT` | `new_value` not readable; `old_value != NULL` and not writable |

## ABI surface

`ITIMER_*` constants and signal delivery as per `getitimer.md`:

| which | clock source | signal |
|---|---|---|
| `ITIMER_REAL` (0) | wall-clock hrtimer | `SIGALRM` |
| `ITIMER_VIRTUAL` (1) | thread-group user CPU time | `SIGVTALRM` |
| `ITIMER_PROF` (2) | thread-group user + kernel CPU time | `SIGPROF` |

`struct itimerval` as per `getitimer.md`.

Special inputs:

| Pattern | Effect |
|---|---|
| `{it_value: {0,0}, it_interval: *}` | disarm (interval is ignored) |
| `{it_value: T, it_interval: {0,0}}` | one-shot fire at T from now, then disarm |
| `{it_value: T, it_interval: P}` | fire at T, then re-fire every P |

## Compatibility contract

REQ-1: `which` validation:
- MUST be in `{0, 1, 2}` else `-EINVAL`.

REQ-2: `new_value` validation:
- `copy_from_user(&nv, new_value)` failure: `-EFAULT`.
- `nv.it_value.tv_usec ∉ [0, 999_999]` → `-EINVAL`.
- `nv.it_interval.tv_usec ∉ [0, 999_999]` → `-EINVAL`.
- `nv.it_value.tv_sec < 0` or `nv.it_interval.tv_sec < 0` → `-EINVAL`.

REQ-3: `old_value` writeback:
- If `old_value != NULL`: snapshot the prior `(it_value, it_interval)` BEFORE applying the new config, then `copy_to_user`.

REQ-4: Disarm:
- `nv.it_value == {0, 0}`: cancel any active hrtimer / posix-cpu-timer; clear `sig.it[*].expires`.

REQ-5: One-shot:
- `nv.it_interval == {0, 0}` ∧ `nv.it_value != {0, 0}`: arm to expire once at `it_value`; on expiry signal is delivered and timer auto-disarms.

REQ-6: Periodic:
- `nv.it_interval != {0, 0}`: on each expiry signal is delivered and timer rearmed to `now + it_interval`.

REQ-7: `ITIMER_REAL` arm:
- Cancel `sig.real_timer` if active.
- If new `it_value != {0,0}`:
  - Convert `it_value` (µs) to `ktime_t`.
  - `sig.it_real_incr = ktime_from(it_interval)`.
  - `hrtimer_start(&sig.real_timer, deadline, HRTIMER_MODE_REL)`.
- Expiry handler queues `SIGALRM` via `send_sig` to the process leader.

REQ-8: `ITIMER_VIRTUAL` arm:
- Update `sig.it[CPUCLOCK_VIRT].expires = thread_group_cputime(...).utime + it_value`.
- `sig.it[CPUCLOCK_VIRT].incr = it_interval`.
- Reschedule cpu-timer subsystem via `set_process_cpu_timer`.

REQ-9: `ITIMER_PROF` arm:
- Same as `ITIMER_VIRTUAL` but tracks `utime + stime`.

REQ-10: Per-process scope:
- `task->signal->...` shared by all threads; mutations require `sig.siglock`.

REQ-11: Signal-handler interaction:
- Process MUST install a handler for the corresponding signal (or accept default action) before arming; otherwise default disposition for `SIGALRM`/`SIGVTALRM`/`SIGPROF` is "terminate".

REQ-12: Inheritance:
- `fork(2)`: child inherits cleared itimers (per POSIX).
- `execve(2)`: itimers cleared.

REQ-13: Time-64 ABI:
- `struct itimerval` uses `__kernel_old_timeval`; on 32-bit `time_t` is 32-bit. Modern alternative: `timer_create(2)` with `__kernel_timespec`.

REQ-14: Signal safety:
- `setitimer` is async-signal-safe per POSIX.1-2008 (rarely used from signal handlers).

## Acceptance Criteria

- [ ] AC-1: `setitimer(ITIMER_REAL, {{0,0}, {1,0}}, NULL)`: `SIGALRM` arrives once after ≥ 1 s.
- [ ] AC-2: `setitimer(ITIMER_REAL, {{1,0}, {2,0}}, NULL)`: first `SIGALRM` at 2 s; subsequent every 1 s.
- [ ] AC-3: `setitimer(ITIMER_REAL, {{0,0}, {0,0}}, &old)`: disarms; `old` holds prior state.
- [ ] AC-4: `setitimer(3, ...)`: `-EINVAL`.
- [ ] AC-5: `new_value == NULL`: `-EFAULT`.
- [ ] AC-6: `nv.it_value.tv_usec == 1_000_000`: `-EINVAL`.
- [ ] AC-7: `setitimer(ITIMER_VIRTUAL, ...)`: under user-mode burn, `SIGVTALRM` arrives at `it_value` user-time elapsed.
- [ ] AC-8: `setitimer(ITIMER_PROF, ...)`: under user+kernel burn, `SIGPROF` arrives when accumulated CPU time matches.
- [ ] AC-9: `fork`: child has cleared itimers; parent unchanged.
- [ ] AC-10: `execve`: itimers cleared.
- [ ] AC-11: Concurrent `setitimer` on same `which` from two threads: linearizable under siglock; no torn state.
- [ ] AC-12: Disarm of disarmed: returns `0` with `old = {0,0,0,0}`.

## Architecture

Rookery surface in `kernel/time/itimer.rs`:

```rust
pub fn sys_setitimer(which: i32,
                    nv_user: *const OldItimerval,
                    ov_user: *mut OldItimerval) -> isize {
    let nv = copy_from_user::<OldItimerval>(nv_user).map_err(|_| -EFAULT)?;
    if !nv.is_valid() { return -EINVAL; }
    let old = match which {
        ITIMER_REAL    => Itimer::set_real(current, &nv),
        ITIMER_VIRTUAL => Itimer::set_cpu(current, CPUCLOCK_VIRT, &nv),
        ITIMER_PROF    => Itimer::set_cpu(current, CPUCLOCK_PROF, &nv),
        _ => return -EINVAL,
    };
    if !ov_user.is_null() {
        copy_to_user(ov_user, &old).map_err(|_| -EFAULT)?;
    }
    0
}
```

`Itimer::set_real(task, nv) -> OldItimerval`:
1. let sig = &task.signal;
2. let _g = sig.siglock.lock();
3. let old_interval = sig.it_real_incr.to_timeval();
4. let old_value = if hrtimer_active(&sig.real_timer) {
   - hrtimer_get_remaining(&sig.real_timer).to_timeval()
   - } else { OldTimeval::ZERO };
5. /* cancel */
6. hrtimer_cancel(&sig.real_timer);
7. /* arm new if non-zero */
8. if nv.it_value != OldTimeval::ZERO {
   - sig.it_real_incr = KTime::from_timeval(nv.it_interval);
   - let deadline = KTime::from_timeval(nv.it_value);
   - hrtimer_init(&sig.real_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
   - sig.real_timer.function = it_real_fn;   // signal-delivery callback
   - hrtimer_start(&sig.real_timer, deadline, HRTIMER_MODE_REL);
 } else {
   - sig.it_real_incr = 0;
 }
9. OldItimerval { it_interval: old_interval, it_value: old_value }

`Itimer::set_cpu(task, which, nv) -> OldItimerval`:
1. let sig = &task.signal;
2. let _g = sig.siglock.lock();
3. let cpu = thread_group_cputime(task);
4. let consumed = if which == CPUCLOCK_VIRT { cpu.utime } else { cpu.utime + cpu.stime };
5. let it = &mut sig.it[which];
6. let old_interval = it.incr.to_timeval();
7. let old_value = if it.expires > 0 { (it.expires - consumed).max(0).to_timeval() } else { OldTimeval::ZERO };
8. /* arm or disarm */
9. if nv.it_value != OldTimeval::ZERO {
   - it.expires = consumed + nv.it_value.to_ns();
   - it.incr = nv.it_interval.to_ns();
 } else {
   - it.expires = 0;
   - it.incr = 0;
 }
10. set_process_cpu_timer(task, which);
11. OldItimerval { it_interval: old_interval, it_value: old_value }

`it_real_fn(hrt)` (callback): walk to `signal_struct` via `container_of`, queue `SIGALRM` to thread group, re-arm if `it_real_incr != 0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `which_validated` | INVARIANT | bad `which` ⟹ -EINVAL |
| `nv_validated` | INVARIANT | bad `nv.tv_usec` ⟹ -EINVAL |
| `efault_on_bad_ptr` | INVARIANT | bad `nv_user`/`ov_user` ⟹ -EFAULT, no kernel-write to non-user page |
| `siglock_held` | INVARIANT | `sig.siglock` held during mutation |
| `old_snapshot_pre_apply` | INVARIANT | `old_value` snapshot taken before applying new config |
| `disarm_cancels_hrtimer` | INVARIANT | `nv.it_value == 0` ⟹ hrtimer_cancel called |

### Layer 2: TLA+

`uapi/setitimer.tla`:
- Variables: `sig.real_timer.{active, deadline}`, `sig.it[*].{expires, incr}`, `sig.siglock`.
- Properties:
  - `safety_disarm_cancels_pending` — disarm cancels pending expiry; no late signal.
  - `safety_old_value_pre_apply` — `old_value` reflects state BEFORE new config.
  - `safety_periodic_rearms` — `it_interval != 0` ⟹ re-arm after each expiry.
  - `liveness_signal_delivered_on_expiry` — armed timer signal eventually delivered.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `setitimer` post(0): timer state matches `nv` | `Itimer::set_real`/`set_cpu` |
| `setitimer` post: if `ov_user != NULL`, `*ov_user` == prior state | `sys_setitimer` |
| `setitimer` post: siglock released | `Itimer::set_*` |
| post-disarm: `hrtimer_active(sig.real_timer) == false` | `Itimer::set_real` |

### Layer 4: Verus/Creusot functional

Per-`setitimer(2)` man page, glibc `sysdeps/unix/sysv/linux/setitimer.c`, musl `src/time/setitimer.c`. Per-LTP `testcases/kernel/syscalls/setitimer/*` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

setitimer reinforcement:

- **Per-which whitelist** — defense against undefined backend access.
- **Per-nv validated pre-arm** — bad timeval values never reach hrtimer / cpu-timer subsystem.
- **Per-siglock-held mutation** — defense against torn state racing with `getitimer`.
- **Per-old_value snapshot pre-apply** — defense against save/restore drift.
- **Per-disarm cancels hrtimer** — defense against late-fire on disarmed timer.
- **Per-process scope strict** — cross-process arming not possible via this syscall.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on every entry** — `setitimer` is a write path with attacker-controlled `nv`; randomize kstack per entry.
- **PaX UDEREF on `nv` read and `ov` writeback** — `copy_from_user(&nv, nv_user)` refuses kernel-mapped addresses; `copy_to_user(ov_user, &old)` refuses the same. Particularly important because `old_value` writeback emits CPU-time-derived data into attacker-controlled memory.
- **GRKERNSEC_CLOCK_RESOLUTION** — under hardened policy, `it_value`/`it_interval` are quantized to `1/HZ` for suid binaries and `CAP_SYS_NICE`-less tasks; defense against using `ITIMER_VIRTUAL`/`PROF` with sub-microsecond intervals to construct CPU-time timing oracles.
- **CAP_SYS_TIME N/A** — itimers do not mutate the wall clock; sibling capability untouched.
- **CAP_WAKE_ALARM N/A** — itimers do not wake from suspend (use `CLOCK_*_ALARM` via `timer_create` instead).
- **adjtimex rate-limit interaction** — `ITIMER_REAL` uses `CLOCK_MONOTONIC`-like hrtimer; NTP slew via adjtimex changes the rate at which `ITIMER_REAL` advances, but the bounded slew prevents abrupt mass-fire of armed `ITIMER_REAL` timers.
- **Per-uid itimer arm rate-limit** — `setitimer` from a single uid rate-limited under hardened policy to defeat itimer-arm floods that destabilize the hrtimer red-black-tree or saturate the signal-delivery path.
- **Refuse sub-µs intervals from non-CAP_SYS_NICE tasks** — under hardened policy, `it_interval` ∈ `(0, 1µs)` rejected with `-EINVAL` unless caller has `CAP_SYS_NICE`; defense against using `setitimer` as a near-zero-cost signal-storm primitive that DoS-es the target process via its own SIGALRM handler.
- **Audit unusually large `it_value`** — `it_value > 365 * 86400` (one year) audit-logged; legitimate use cases are uncommon and the pattern can indicate covert long-deadline reconnaissance.
- **GRKERNSEC_HIDESYM on `signal->real_timer` address** — process signal struct excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`.
- **Bounded periodic rearm** — under hardened policy with extremely small `it_interval`, the hrtimer expiry callback enforces a minimum 1 ms gap between deliveries (back-off, not loss); defense-in-depth against runaway re-fire that pre-empts the process before its handler can install a guard.
- **GRKERNSEC_AUDIT log on `ITIMER_VIRTUAL`/`PROF` arm by non-thread-group-leader** — only the TGL legitimately mutates process-wide itimers; arms from other threads logged.
- **getitimer + setitimer co-rate-limit** — shared token bucket per uid.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `kernel/time/itimer.md` Tier-3: itimer state machine, expiry callback, signal queueing.
- `kernel/time/hrtimer.md` Tier-3: hrtimer subsystem.
- `kernel/time/posix_cpu_timers.md` Tier-3: CPU-time accounting and arming.
- `kernel/signal.md` Tier-3: signal delivery integration.
- `getitimer.md` sibling — read path.
- `alarm.md` sibling — `ITIMER_REAL` convenience wrapper.
- `timer_create(2)` family — modern POSIX interval timers.
- `timerfd_create(2)` — fd-based interval timer.
- glibc / musl userspace `setitimer` wrappers.
- Implementation code.
