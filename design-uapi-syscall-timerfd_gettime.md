---
title: "Tier-5 syscall: timerfd_gettime(2) — syscall 287"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`timerfd_gettime(2)` reports the current arming state of a timerfd: the time **remaining** until the next expiration (`it_value`) and the reload **interval** (`it_interval`). Like `timer_gettime(2)`, the reported `it_value` is always relative (offset from now), even if the timer was armed with `TFD_TIMER_ABSTIME`. A disarmed timerfd reports `{0, 0} / {0, 0}` (interval also zero after a cancel-on-set fire that cleared `tintv`). Critical for: deadline introspection in event loops, periodic-tick diagnostics, debugging "did my timer arm correctly?" questions, reconstructing absolute expiry by composing with `clock_gettime`.

This Tier-5 covers the userspace ABI of syscall 287. Per-`hrtimer_get_remaining` / per-`alarm_get_remaining` computation is owned by `kernel/time/hrtimer.md` / `kernel/time/alarmtimer.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: Disarmed timerfd: `{0,0} / {0,0}`.
- [ ] AC-2: Arm with relative `it_value=2s`; immediately query: `it_value ≈ 2s` (±tick).
- [ ] AC-3: 1s later: `it_value ≈ 1s`.
- [ ] AC-4: Periodic `it_interval=100ms`: `it_interval == {0, 100_000_000}`.
- [ ] AC-5: Arm with `TFD_TIMER_ABSTIME` future; query: `it_value = future − now`.
- [ ] AC-6: After cancel-on-set fire and `-ECANCELED` read: query returns `{0,0}/{0,0}`.
- [ ] AC-7: `fd` not a timerfd: `-EINVAL`.
- [ ] AC-8: Invalid `fd`: `-EBADF`.
- [ ] AC-9: NULL `curr_value`: `-EFAULT`.
- [ ] AC-10: Query does not drain `ctx.ticks`; subsequent `read` still returns count.
- [ ] AC-11: BOOTTIME timerfd across suspend: remaining advances during sleep.

### Architecture

Rookery surface in `fs/timerfd.rs`:

```rust
pub fn sys_timerfd_gettime(fd: i32,
                            out: *mut KernelItimerspec) -> isize {
    let file = fdget(fd).ok_or(-EBADF)?;
    let ctx = Timerfd::from_file(&file).ok_or(-EINVAL)?;
    let mut snap = KernelItimerspec::default();
    Timerfd::query(&ctx, &mut snap);
    if copy_to_user(out, &snap).is_err() { return -EFAULT; }
    0
}
```

`Timerfd::query(ctx, snap)`:
1. let guard = ctx.lock.lock_irq();
2. snap.it_interval = ktime_to_ts(ctx.tintv);
3. if !ctx.expires_in_use {
   - snap.it_value = Timespec::ZERO;
 } else {
   - let rem = if ctx.is_alarm() {
       - alarm_get_remaining(&ctx.t.alarm)
     } else {
       - hrtimer_get_remaining(&ctx.t.tmr)
     };
   - snap.it_value = if rem.is_negative() { Timespec::ZERO } else { ktime_to_ts(rem) };
 }
4. drop(guard);

### Out of Scope

- `timerfd_create.md`, `timerfd_settime.md` siblings.
- `kernel/timerfd.md` Tier-3: ctx lifecycle, wait-queue protocol.
- `kernel/time/hrtimer.md` Tier-3: hrtimer_get_remaining math.
- `kernel/time/alarmtimer.md` Tier-3: alarm_get_remaining math.
- `clock_gettime.md` for clock-now query.
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
int timerfd_gettime(int fd, struct itimerspec *curr_value);
```

Rust ABI shim:

```rust
pub fn sys_timerfd_gettime(fd: i32,
                            curr_value: *mut __kernel_itimerspec) -> isize;
```

Syscall number: **287**.
On 32-bit time-safe variant: `sys_timerfd_gettime64` (syscall 411).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `fd` | `int` | IN | timerfd from `timerfd_create` |
| `curr_value` | `struct __kernel_itimerspec *` | OUT | `{it_interval, it_value}` snapshot |

### return

- **Success**: `0`; `*curr_value` populated.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` is not a valid file descriptor |
| `EINVAL` | `fd` is not a timerfd |
| `EFAULT` | `curr_value` not in writable userspace |

### abi surface

Output layout: `struct __kernel_itimerspec` (see `timer_gettime.md`).

| Timer state | Reported `it_value` | Reported `it_interval` |
|---|---|---|
| Disarmed | `{0, 0}` | `{0, 0}` |
| One-shot armed, t s until fire | `{t.s, t.ns}` | `{0, 0}` |
| Periodic, t s until next, period p | `{t.s, t.ns}` | `{p.s, p.ns}` |
| Cancel-on-set fired (canceled, drained) | `{0, 0}` | `{0, 0}` (tintv cleared on the fire path; read returned -ECANCELED) |

### compatibility contract

REQ-1: `fd` validation:
- `fdget(fd)` returns a `struct file *` with `f_op == &timerfd_fops`; else `-EINVAL`.

REQ-2: Snapshot under ctx lock:
- `spin_lock_irq(&ctx.lock)` taken.
- Remaining computed: `hrtimer_get_remaining(&ctx.t.tmr)` or `alarm_get_remaining(&ctx.t.alarm)`.
- Interval = `ctx.tintv` converted to timespec.
- Negative remaining clamped to `{0, 0}`.

REQ-3: Disarmed timerfd:
- `!ctx.expires_in_use`: `it_value == 0`, `it_interval == ktime_to_ts(ctx.tintv)`.

REQ-4: Cancel-on-set fired:
- After cancel-on-set, `ctx.tintv` is cleared and the next `read` returns `-ECANCELED`; subsequent `timerfd_gettime` reports `{0, 0} / {0, 0}`.

REQ-5: No mutation:
- `timerfd_gettime` MUST NOT clear `ctx.ticks` (unread expiration count), arm/disarm, or modify any other state.

REQ-6: Output write:
- Snapshot taken under lock; lock released; `copy_to_user` outside the lock.
- Failure: `-EFAULT`.

REQ-7: No capability requirement:
- Owner of the fd may query freely.

REQ-8: Per-class semantics:
- ALARM-class: remaining reflects RTC-backed wakeup-source target.
- MONOTONIC-class: remaining decreases monotonically (excluding suspend).
- BOOTTIME-class: remaining advances during suspend.

REQ-9: Concurrent expiry:
- If hrtimer callback is firing concurrently, snapshot may catch the transition; report remaining as 0 if expires has passed but interval has not yet been added.

REQ-10: Interval reporting:
- `ctx.tintv == 0`: one-shot or disarmed; interval reported as `{0, 0}`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `udref_on_out_pointer` | INVARIANT | curr_value access_ok-checked before copy_to_user |
| `ctx_lock_during_snapshot` | INVARIANT | tintv + remaining read under ctx.lock |
| `no_mutation_on_query` | INVARIANT | ctx.ticks, ctx.expires_in_use unchanged after query |
| `negative_remaining_clamped` | INVARIANT | negative ktime ⟹ reported as 0 |
| `disarmed_yields_zero_value` | INVARIANT | !expires_in_use ⟹ it_value == 0 |

### Layer 2: TLA+

`uapi/timerfd_gettime.tla`:
- Variables: `ctx_armed[fd]`, `ctx_tintv[fd]`, `now`.
- Properties:
  - `safety_no_mutation` — ctx fields unchanged after query.
  - `safety_remaining_nonnegative` — reported it_value ≥ 0.
  - `safety_interval_invariant` — reported interval equals `ctx.tintv`.
  - `safety_disarmed_post_cancel_on_set` — after cancel-on-set, snapshot = {0,0}/{0,0}.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `query` post: snap.it_interval == ktime_to_ts(ctx.tintv) | `Timerfd::query` |
| `query` post: armed ⟹ snap.it_value == max(0, remaining) | `Timerfd::query` |
| `query` post: disarmed ⟹ snap.it_value == 0 | `Timerfd::query` |
| no ctx.ticks mutation | `Timerfd::query` |

### Layer 4: Verus/Creusot functional

Per-`timerfd_gettime(2)` man page, glibc `sysdeps/unix/sysv/linux/timerfd_gettime.c`, LTP `testcases/kernel/syscalls/timerfd/*`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

timerfd_gettime reinforcement:

- **Read-only path** — no mutation; ticks counter not drained.
- **Lock-bounded snapshot** — ctx.lock held only for the snapshot, not for copy_to_user.
- **Clamp on negative remaining** — never report negative durations.
- **No capability needed** — owner of the fd queries freely.

### grsecurity/pax-style reinforcement

- **PaX UDEREF on `curr_value`** — `copy_to_user(curr_value, &snap)` refuses kernel-mapped addresses; defense against forged out-pointer that would leak ctx state or trigger oops via type-confused write.
- **PAX_RANDKSTACK on every entry** — randomized kstack; relevant because timerfd accessors share locking primitives with hrtimer/alarm wheels.
- **GRKERNSEC_HIDESYM on `timerfd_fops`** — function-pointer table hidden from kallsyms; defense against rop-style abuse of `f_op` indirect calls during `from_file` dispatch.
- **Per-uid query rate limit** — hardened policy throttles `timerfd_gettime` (default: 100k/sec/uid) to defeat timing-oracle abuse where attackers infer hrtimer wheel scheduling from rapid remaining-time queries.
- **Foreign-process fd query rejected pre-`from_file`** — `f_op != timerfd_fops` returns `-EINVAL` without revealing whether the fd existed; defense against cross-process fd-table probing.
- **No leakage of suspend-skew via remaining-time on BOOTTIME-class** — under hardened mode, BOOTTIME timerfd remaining advances during suspend is reported via quantized step to prevent fingerprinting of suspend duration by unprivileged tasks.
- **Audit on suspicious query patterns** — bursts > 10k queries/sec to the same fd logged as potential timing-oracle exploitation.
- **GRKERNSEC_TIMERFD_INTEGRITY** — ctx slab canary validated at query time; corruption (e.g., from buffer overflow into the timerfd_ctx slab) triggers a hardened-mode OOPS rather than silently reading attacker-corrupted data.
- **Reject query under PaX KERNEXEC inconsistency** — if kernel-text consistency check fails during the call, syscall returns `-EFAULT` rather than potentially reading attacker-corrupted ctx state.
- **No-cross-userns ctx access** — even if fd table sharing leaks a timerfd across userns boundaries, hardened policy validates `current.user_ns == ctx.userns` and returns `-EINVAL` otherwise.
- **Snapshot under SLAB_TYPESAFE_BY_RCU** — ctx protected against UAF during the brief query path; even if a sibling thread races `close(fd)`, the snapshot remains type-safe.
- **Cancel-on-set state disclosure quantized** — `it_value == 0` after cancel-on-set is intentionally indistinguishable from a freshly-disarmed timer (no separate "canceled" code) so attackers cannot use timerfd_gettime as a high-resolution wall-step detector.

