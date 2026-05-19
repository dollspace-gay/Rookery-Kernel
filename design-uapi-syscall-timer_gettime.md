---
title: "Tier-5 syscall: timer_gettime(2) — syscall 224"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`timer_gettime(2)` reports the current arming state of a POSIX per-process timer identified by `timerid` — specifically the time **remaining** until the next expiration (`it_value`) and the **reload interval** (`it_interval`). A disarmed timer reports `it_value == {0, 0}`. The reported `it_value` is always in **relative** form (offset from now), regardless of how the timer was armed: an `TIMER_ABSTIME`-armed timer that expires in 2.5 seconds reports `{2, 500_000_000}`. The interval is reported as set. This syscall does NOT mutate the timer; it is the read-only sibling of `timer_settime(2)`. Critical for: deadline introspection (time-remaining displays, scheduler diagnostics), idempotent timer-state queries (poll a timer without disarming it), reconstructing absolute expiry by composing the result with `clock_gettime`.

This Tier-5 covers the userspace ABI of syscall 224. Per-`hrtimer` remaining-time computation and per-`alarmtimer` query are owned by `kernel/time/hrtimer.md` and `kernel/time/alarmtimer.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: Disarmed timer: `{0,0} / {0,0}`.
- [ ] AC-2: Arm with `it_value=2s` (rel); immediately query: `it_value ≈ 2s` (±tick).
- [ ] AC-3: 1s later: `it_value ≈ 1s`.
- [ ] AC-4: Periodic with interval `100ms`: `it_interval == {0, 100_000_000}`.
- [ ] AC-5: Arm with `TIMER_ABSTIME` future; query: `it_value = future − now`.
- [ ] AC-6: Past-fire absolute: `it_value` reports remaining-to-next-period (0 if one-shot already fired).
- [ ] AC-7: `timerid` belonging to another process ⟹ `-EINVAL`.
- [ ] AC-8: NULL `curr_value` ⟹ `-EFAULT`.
- [ ] AC-9: Query does not consume overrun count; subsequent `timer_getoverrun` still returns count.
- [ ] AC-10: Alarm timer across suspend: remaining advances during suspend.
- [ ] AC-11: CPU-time timer: remaining decreases only when task accrues CPU.

### Architecture

Rookery surface in `kernel/time/posix_timers.rs`:

```rust
pub fn sys_timer_gettime(id: TimerId,
                         out: *mut KernelItimerspec) -> isize {
    let mut snap = KernelItimerspec::default();
    let rc = PosixTimers::do_timer_get(id, &mut snap);
    if rc != 0 { return rc; }
    if copy_to_user(out, &snap).is_err() { return -EFAULT; }
    0
}
```

`PosixTimers::do_timer_get(id, snap) -> isize`:
1. let (k, guard) = posix_timer_get_locked(&current.signal, id).ok_or(-EINVAL)?;
2. let kc = &posix_clock_kclocks[k.it_clock as usize];
3. kc.timer_get(&k, snap);
4. drop(guard);
5. 0

`common_timer_get(k, snap)` for hrtimer-backed clocks:
1. snap.it_interval = ktime_to_timespec(k.it_interval);
2. if !k.it_active {
   - snap.it_value = Timespec::ZERO;
   - return;
 }
3. let rem = hrtimer_get_remaining(&k.it.real.timer);
4. snap.it_value = if rem.is_negative() { Timespec::ZERO } else { ktime_to_timespec(rem) };

`alarm_timer_get(k, snap)` for alarm-class clocks:
1. snap.it_interval = ktime_to_timespec(k.it_interval);
2. if !k.it_active { snap.it_value = Timespec::ZERO; return; }
3. let rem = alarm_get_remaining(&k.it.alarm.alarm);
4. snap.it_value = if rem.is_negative() { Timespec::ZERO } else { ktime_to_timespec(rem) };

### Out of Scope

- `timer_create.md`, `timer_settime.md`, `timer_getoverrun.md`, `timer_delete.md` siblings.
- `kernel/time/hrtimer.md` Tier-3.
- `kernel/time/alarmtimer.md` Tier-3.
- `kernel/time/posix_timers.md` Tier-3.
- `clock_gettime.md` for clock-now query.
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
int timer_gettime(timer_t timerid, struct itimerspec *curr_value);
```

Rust ABI shim:

```rust
pub fn sys_timer_gettime(timerid: TimerId,
                         curr_value: *mut __kernel_itimerspec) -> isize;
```

Syscall number: **224**.
On 32-bit time-safe variant: `sys_timer_gettime64` (syscall 408).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `timerid` | `timer_t` | IN | timer handle from `timer_create` |
| `curr_value` | `struct __kernel_itimerspec *` | OUT | `{it_interval, it_value}` snapshot |

### return

- **Success**: `0`; `*curr_value` populated.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno; `*curr_value` undefined on failure.

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | `timerid` does not name a live timer in current process |
| `EFAULT` | `curr_value` not in writable userspace |

### abi surface

Output structure layout:

```c
struct __kernel_itimerspec {
    struct __kernel_timespec it_interval;  /* reload (zero if one-shot) */
    struct __kernel_timespec it_value;     /* time until next expiry */
};
```

Reported semantics:

| Timer state | Reported `it_value` | Reported `it_interval` |
|---|---|---|
| Disarmed | `{0, 0}` | `{0, 0}` |
| One-shot armed, t s until fire | `{t.s, t.ns}` | `{0, 0}` |
| Periodic, t s until next, period p | `{t.s, t.ns}` | `{p.s, p.ns}` |
| Periodic that has already fired (overrun pending) | `{remaining-to-next-fire ≥ 0}` | `{p.s, p.ns}` |

### compatibility contract

REQ-1: `timerid` lookup:
- `posix_timer_get_locked(current.signal, timerid)` MUST find a live `k_itimer` whose `it_signal == current.signal`; else `-EINVAL`.
- Lookup taken under `current->sighand->siglock`.

REQ-2: Remaining-time computation:
- For hrtimer-backed clocks: `remaining = hrtimer_get_remaining(&k.it.real.timer)`.
- For CPU-time clocks: `remaining = k.it.cpu.expires − current_cpu_time(k.target)`.
- For alarm-class clocks: `alarm_get_remaining(&k.it.alarm)`.
- Negative remaining (timer fired but not yet delivered/requeued): clamp to `0`.

REQ-3: Disarmed timer:
- `k.it_active == false`: `it_value = {0, 0}`, `it_interval` preserved as last-set.

REQ-4: Interval reporting:
- `k.it_interval` (ktime_t) converted back to timespec; reported as-is.
- A one-shot timer (interval zero) reports `{0, 0}`.

REQ-5: Output write:
- Snapshot taken under lock; lock released; `copy_to_user(curr_value, &snapshot)` performed outside lock.
- `copy_to_user` failure: `-EFAULT`.

REQ-6: No mutation:
- `timer_gettime` MUST NOT clear `it_overrun`, reset arming, or otherwise touch state.

REQ-7: Concurrent expiry:
- A timer firing concurrently with `timer_gettime` may report `it_value == 0` (in the brief window between "expired" and "requeued for next interval"); next call observes the new remaining-to-next-period.

REQ-8: CPU-time clocks:
- `CLOCK_PROCESS_CPUTIME_ID` / `CLOCK_THREAD_CPUTIME_ID`-backed timers: remaining depends on accumulated cputime; query is consistent only across no-task-migration windows.

REQ-9: Suspend behavior:
- For alarm-class timers, suspend time counts as elapsed; remaining reported relative to post-resume `now`.

REQ-10: No capability requirement:
- Caller does NOT need `CAP_SYS_TIME` or `CAP_WAKE_ALARM` to query a timer it owns.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `udref_on_out_pointer` | INVARIANT | curr_value access_ok-checked before copy_to_user |
| `siglock_during_lookup` | INVARIANT | k_itimer fetched under sighand siglock |
| `no_mutation_on_query` | INVARIANT | `do_timer_get` does not touch k.it_interval, it_active, it_overrun |
| `negative_remaining_clamped` | INVARIANT | negative ktime ⟹ reported as 0 |
| `disarmed_yields_zero_value` | INVARIANT | k.it_active == false ⟹ snap.it_value == 0 |

### Layer 2: TLA+

`uapi/timer_gettime.tla`:
- Variables: `timer_state[id]`, `now`, `snapshot[id]`.
- Properties:
  - `safety_no_mutation` — k_itimer fields unchanged after gettime.
  - `safety_remaining_nonnegative` — reported it_value ≥ 0.
  - `safety_interval_invariant` — reported interval equals last-armed interval.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_timer_get` post: snap.it_interval == ktime_to_ts(k.it_interval) | `common_timer_get` |
| `do_timer_get` post: armed ⟹ snap.it_value == max(0, hrtimer_get_remaining) | `common_timer_get` |
| `do_timer_get` post: disarmed ⟹ snap.it_value == 0 | `common_timer_get` |
| no overrun mutation | `do_timer_get` |

### Layer 4: Verus/Creusot functional

Per-`timer_gettime(2)` man page, glibc `sysdeps/unix/sysv/linux/timer_gettime.c`, LTP `testcases/kernel/syscalls/timer_gettime/*`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

timer_gettime reinforcement:

- **Read-only path** — no mutation of k_itimer fields; defense against accidental overrun-clear.
- **Lock-bounded snapshot** — sighand siglock held only for the snapshot, not for copy_to_user.
- **Clamp on negative remaining** — never report negative durations.
- **No capability needed** — owner of the timer can query freely.

### grsecurity/pax-style reinforcement

- **PaX UDEREF on `curr_value`** — `copy_to_user(curr_value, &snap)` refuses kernel-mapped addresses; defense against forged out-pointer that would leak `k_itimer` state into kernel memory or trigger oops via type-confused write.
- **PAX_RANDKSTACK on every entry** — random kstack on each call; relevant because POSIX-timer accessors share locking primitives with signal delivery.
- **GRKERNSEC_HIDESYM on `posix_clock_kclocks`** — `kc.timer_get` indirect-call table hidden from kallsyms.
- **Per-uid query rate limit** — hardened policy installs a per-uid bucket (default: 100k queries/sec) to prevent abuse of timer_gettime as a high-resolution timing oracle that leaks hrtimer wheel scheduling decisions.
- **Foreign-process timer query rejected pre-lookup** — `it_signal != current.signal` returns `-EINVAL` without revealing whether the ID existed in any process; defense against IDR-probing for cross-process timer existence.
- **Audit on bulk queries** — hardened policy logs `(uid, pid, timer_id)` for query bursts exceeding a threshold; defenders detect timing-oracle exploitation patterns.
- **Snapshot under SLAB_TYPESAFE_BY_RCU** — k_itimer protected against UAF during the brief gettime path; even if a sibling thread races `timer_delete`, the snapshot remains type-safe.
- **Defense against information disclosure via remaining-time precision** — under hardened mode, `it_value.tv_nsec` is quantized to the nearest 1us before reporting to userspace if the timer was armed by a different uid than the querying task (does not normally occur; defense-in-depth against shared-credential abuses).
- **Reject query of timer_id from a vfork'd child mid-window** — between `clone(CLONE_VFORK)` and exec, the child's signal struct is shared; hardened policy rejects timer_gettime from such windows with `-EINVAL` to prevent state escape from parent.
- **Audit timer_get on alarm-class** — alarm timers query the RTC-backed wakeup source; hardened policy logs each query so attackers cannot quietly map the alarmtimer wheel.
- **No leak of suspend-time skew** — `it_value` after resume is computed against current wall-clock, not raw RTC offset, so suspend-time delta cannot be inferred from gettime.

