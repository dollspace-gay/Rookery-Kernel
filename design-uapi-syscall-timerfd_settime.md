---
title: "Tier-5 syscall: timerfd_settime(2) — syscall 286"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`timerfd_settime(2)` programs (arms, re-arms, or disarms) a timerfd previously created by `timerfd_create(2)`, transitioning it from disarmed to armed state with an initial expiry (`it_value`) and an optional periodic reload (`it_interval`). Expirations accumulate into the fd's 64-bit counter; userspace consumes the count via `read(2)`. Unlike `timer_settime(2)`, delivery is via file-descriptor readability (epoll/select/poll), not signals — making timerfds the modern primitive for event-loop deadlines. `TFD_TIMER_ABSTIME` selects absolute-time semantics on the timer's clock. `TFD_TIMER_CANCEL_ON_SET` (REALTIME / REALTIME_ALARM only) marks the timer for cancellation if the wall clock is stepped via `clock_settime(2)` / `settimeofday(2)`; `read(2)` then returns `-ECANCELED` and the timer remains disarmed until `timerfd_settime` is called again. Critical for: cron-like absolute-time alarms that survive clock skew, wall-time deadlines that must be invalidated on time-jump, periodic ticks for libuv/tokio/asyncio.

This Tier-5 covers the userspace ABI of syscall 286. Per-`hrtimer` / per-`alarmtimer` arming, per-`__timerfd_remove_cancel_list` cancel-on-set wake protocol, and `timerfd_ctx` lifecycle are owned by `kernel/timerfd.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: Arm with `it_value=1s`; after 1s, fd POLLIN; `read(8 bytes)` returns count `1`.
- [ ] AC-2: Periodic `it_interval=100ms`; over 1s, counter accrues ~10 expirations.
- [ ] AC-3: Disarm with `it_value=0`: subsequent fires do not occur.
- [ ] AC-4: `TFD_TIMER_ABSTIME` past: fires immediately on next tick.
- [ ] AC-5: `TFD_TIMER_CANCEL_ON_SET` on CLOCK_REALTIME ABSTIME: after `clock_settime` step, `read(2)` returns `-ECANCELED`.
- [ ] AC-6: `TFD_TIMER_CANCEL_ON_SET` on CLOCK_MONOTONIC ⟹ `-EINVAL`.
- [ ] AC-7: `flags` with bit 2 set ⟹ `-EINVAL`.
- [ ] AC-8: `fd` not a timerfd ⟹ `-EINVAL`.
- [ ] AC-9: `tv_nsec = 10^9` ⟹ `-EINVAL`.
- [ ] AC-10: `old_value` populated correctly; NULL accepted.
- [ ] AC-11: Counter not reset by re-arm; pending expirations still readable.

### Architecture

Rookery surface in `fs/timerfd.rs`:

```rust
pub fn sys_timerfd_settime(fd: i32,
                            flags: i32,
                            new: *const KernelItimerspec,
                            old: *mut KernelItimerspec) -> isize {
    if flags & !(TFD_TIMER_ABSTIME | TFD_TIMER_CANCEL_ON_SET) != 0 { return -EINVAL; }
    let nv = match copy_from_user::<KernelItimerspec>(new) {
        Ok(v) => v, Err(_) => return -EFAULT,
    };
    if !itimerspec_valid(&nv) { return -EINVAL; }
    let file = fdget(fd).ok_or(-EBADF)?;
    let ctx = Timerfd::from_file(&file).ok_or(-EINVAL)?;
    if (flags & TFD_TIMER_CANCEL_ON_SET) != 0 && !ctx.clock_is_realtime_class() { return -EINVAL; }
    if (flags & TFD_TIMER_CANCEL_ON_SET) != 0 && (flags & TFD_TIMER_ABSTIME) == 0 { return -EINVAL; }
    let mut prev = KernelItimerspec::default();
    Timerfd::setup(&ctx, flags, &nv, &mut prev)?;
    if !old.is_null() && copy_to_user(old, &prev).is_err() { return -EFAULT; }
    0
}
```

`Timerfd::setup(ctx, flags, new, old) -> Result<()>`:
1. let guard = ctx.lock.lock_irq();
2. /* snapshot old state */
3. Timerfd::snapshot_old(&ctx, old);
4. /* cancel pending */
5. if ctx.expires_in_use {
   - if ctx.is_alarm() { alarm_cancel(&ctx.t.alarm); }
   - else { hrtimer_cancel(&ctx.t.tmr); }
   - ctx.expires_in_use = false;
 }
6. ctx.expired = false;
7. ctx.canceled = false;
8. ctx.settime_flags = flags as u8;
9. /* update cancel-on-set list */
10. if flags & TFD_TIMER_CANCEL_ON_SET != 0 {
    - cancel_list_add(&ctx);
  } else {
    - cancel_list_remove(&ctx);
  }
11. /* fast-path disarm */
12. if new.it_value == 0 {
    - ctx.tintv = 0;
    - return Ok(());
  }
13. /* arm */
14. let now = ctx.clock_now();
15. let expires = if flags & TFD_TIMER_ABSTIME != 0 { ktime_from_ts(new.it_value) }
                else { now + ktime_from_ts(new.it_value) };
16. ctx.tintv = ktime_from_ts(new.it_interval);
17. ctx.expires_in_use = true;
18. let mode = if flags & TFD_TIMER_ABSTIME != 0 { HRTIMER_MODE_ABS } else { HRTIMER_MODE_REL };
19. if ctx.is_alarm() { alarm_start(&ctx.t.alarm, expires); }
    else { hrtimer_start(&ctx.t.tmr, expires, mode); }
20. drop(guard);
21. Ok(())

### Out of Scope

- `timerfd_create.md`, `timerfd_gettime.md` siblings.
- `kernel/timerfd.md` Tier-3: ctx lifecycle, wait-queue protocol, cancel-on-set list walk.
- `kernel/time/hrtimer.md` Tier-3: hrtimer wheel.
- `kernel/time/alarmtimer.md` Tier-3: alarm wakeup-source.
- `clock_settime.md` for wall-clock step that triggers cancel-on-set.
- glibc / musl userspace wrappers.
- libuv / tokio / asyncio integration.
- Implementation code.

### signature

```c
int timerfd_settime(int fd,
                    int flags,
                    const struct itimerspec *new_value,
                    struct itimerspec *old_value);
```

Rust ABI shim:

```rust
pub fn sys_timerfd_settime(fd: i32,
                           flags: i32,
                           new_value: *const __kernel_itimerspec,
                           old_value: *mut __kernel_itimerspec) -> isize;
```

Syscall number: **286**.
On 32-bit time-safe variant: `sys_timerfd_settime64` (syscall 410).

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `fd` | `int` | IN | timerfd from `timerfd_create` |
| `flags` | `int` | IN | bitmask: `TFD_TIMER_ABSTIME | TFD_TIMER_CANCEL_ON_SET` |
| `new_value` | `const struct __kernel_itimerspec *` | IN | `{it_interval, it_value}` |
| `old_value` | `struct __kernel_itimerspec *` | OUT | previous arming state; may be `NULL` |

### return

- **Success**: `0`; `*old_value` populated if non-NULL.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` is not a valid file descriptor |
| `EINVAL` | `fd` is not a timerfd; unknown `flags` bit; `tv_nsec` ∉ `[0, 999_999_999]`; negative `tv_sec`; `TFD_TIMER_CANCEL_ON_SET` on non-REALTIME-class clock |
| `EFAULT` | `new_value` not readable or `old_value` not writable |
| `ECANCELED` | (returned by subsequent `read(2)`, not by this call) cancel-on-set fired |

### abi surface

`flags` values:

| Flag | Value | Meaning |
|---|---|---|
| `TFD_TIMER_ABSTIME` | `1 << 0` | `it_value` is absolute time on timer's clock |
| `TFD_TIMER_CANCEL_ON_SET` | `1 << 1` | clock-step on `CLOCK_REALTIME` / `CLOCK_REALTIME_ALARM` cancels the timer |

`struct __kernel_itimerspec`: as defined in `timer_settime.md`.

Clocks supporting `TFD_TIMER_CANCEL_ON_SET`:

| Clock | Cancel-on-set |
|---|---|
| `CLOCK_REALTIME` | yes |
| `CLOCK_REALTIME_ALARM` | yes |
| `CLOCK_MONOTONIC` | no |
| `CLOCK_BOOTTIME` | no |
| `CLOCK_BOOTTIME_ALARM` | no |

### compatibility contract

REQ-1: `fd` validation:
- `fdget(fd)` returns a `struct file *` whose `f_op == &timerfd_fops`; else `-EINVAL`.

REQ-2: `flags` validation:
- Only `TFD_TIMER_ABSTIME | TFD_TIMER_CANCEL_ON_SET` accepted; other bits ⟹ `-EINVAL`.
- `TFD_TIMER_CANCEL_ON_SET` requires REALTIME-class clock; else `-EINVAL`.
- `TFD_TIMER_CANCEL_ON_SET` requires `TFD_TIMER_ABSTIME` (Linux ≥ 3.0 enforced); else `-EINVAL`.

REQ-3: `new_value` validation:
- `copy_from_user` failure: `-EFAULT`.
- `tv_nsec` ∉ `[0, 999_999_999]` for either value or interval: `-EINVAL`.
- Negative `tv_sec`: `-EINVAL`.

REQ-4: Disarm semantics:
- `it_value == 0`: cancels pending hrtimer; counter not reset (existing expirations stay readable); cancel-on-set list entry removed.

REQ-5: Arming:
- ktime computed from `now + it_value` (relative) or `it_value` (absolute).
- For hrtimer-backed clocks: `hrtimer_start(&ctx.t.tmr, expires, mode)`.
- For alarm-class: `alarm_start(&ctx.t.alarm, expires)`.

REQ-6: Old-value snapshot:
- Pre-arming state captured: remaining-to-fire and interval.
- Disarmed timer reports `{0, 0}` / preserved interval.

REQ-7: Counter behavior on re-arm:
- Re-arming does NOT reset the unread expiration counter; userspace must `read(2)` to drain.
- A `timerfd_settime` call with `new.it_value == 0` does NOT clear the counter.

REQ-8: Cancel-on-set list:
- If `TFD_TIMER_CANCEL_ON_SET` set: ctx is added to `cancel_list` (linked list scanned by `clock_was_set()`).
- If `TFD_TIMER_CANCEL_ON_SET` cleared on re-arm: ctx removed from `cancel_list`.

REQ-9: `clock_was_set()` interaction:
- On wall-clock step (via `clock_settime(REALTIME)` / `settimeofday`), kernel walks `cancel_list`, sets `ctx.expired = true`, sets `ctx.canceled = true`, wakes `ctx.wqh` so blocked `read(2)` callers return `-ECANCELED`.
- Subsequent `read(2)` on a cancelled timer: `-ECANCELED`; counter cleared; timer remains armed but inert until next `timerfd_settime`.

REQ-10: Per-class alarm gating:
- ALARM-class clock arming preserves `CAP_WAKE_ALARM` requirement (enforced at `timerfd_create`); re-arming itself does not re-check (already held at create).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_whitelisted` | INVARIANT | only TFD_TIMER_ABSTIME/CANCEL_ON_SET accepted |
| `cancel_on_set_requires_abstime` | INVARIANT | CANCEL_ON_SET without ABSTIME ⟹ -EINVAL |
| `cancel_on_set_realtime_only` | INVARIANT | CANCEL_ON_SET on non-REALTIME-class ⟹ -EINVAL |
| `udref_on_pointers` | INVARIANT | new/old via access_ok-gated copies |
| `cancel_list_consistency` | INVARIANT | CANCEL_ON_SET flag ⟺ ctx in cancel_list |
| `hrtimer_cancel_before_reprogram` | INVARIANT | old hrtimer cancelled before new arming |

### Layer 2: TLA+

`uapi/timerfd_settime.tla`:
- Variables: `ctx_armed[fd]`, `ctx_expires[fd]`, `ctx_in_cancel_list[fd]`, `wall_clock`.
- Properties:
  - `safety_cancel_on_set_on_wall_step` — wall clock step ⟹ ctx.canceled set; read returns -ECANCELED.
  - `safety_no_reset_on_rearm` — counter unread expirations preserved across re-arm.
  - `safety_disarm_cancels_hrtimer` — it_value=0 disarms cleanly.
  - `liveness_armed_fires` — armed timer with future expiry eventually wakes wait queue.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `setup` post: hrtimer/alarm armed iff new.it_value != 0 | `Timerfd::setup` |
| `setup` post: cancel_list membership matches flag | `Timerfd::setup` |
| `setup` post: old populated with prior arming | `Timerfd::snapshot_old` |
| wall-step post: all CANCEL_ON_SET ctxs marked canceled | `__timerfd_remove_cancel_list` |

### Layer 4: Verus/Creusot functional

Per-`timerfd_settime(2)` man page, glibc `sysdeps/unix/sysv/linux/timerfd_settime.c`, LTP `testcases/kernel/syscalls/timerfd/*`, real-world (libuv, tokio, asyncio).

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

timerfd_settime reinforcement:

- **Strict flag whitelist** — only the two documented bits accepted.
- **Cancel-on-set constrained** — REALTIME-class + ABSTIME only.
- **hrtimer cancel before reprogram** — no double-arm race.
- **Counter preserved across re-arm** — userspace never loses unread expirations to a re-arm.
- **Cancel-list scanning under timerfd_lock** — `clock_was_set` walks list atomically.
- **Per-fd lock** — concurrent settime/read/close serialized.

### grsecurity/pax-style reinforcement

- **PaX UDEREF on `new_value` and `old_value`** — both copies use access_ok plus checked transfers; defense against attacker forging a kernel-mapped address for the itimerspec marshaling.
- **PAX_RANDKSTACK on every entry** — random kstack on each call; relevant because timerfd_settime touches hrtimer wheel, alarmtimer wheel, and the cancel-on-set linked list — all attractive ROP targets.
- **GRKERNSEC_HIDESYM on `timerfd_fops` and `cancel_list`** — both hidden from kallsyms; defense against rop-style abuse of the `f_op` table and the cancel-on-set walker.
- **Per-uid timerfd-quota** — hardened policy installs a per-uid limit on armed timerfds (default: 256); over-quota arming returns `-EMFILE` rather than `-EINVAL` to remain forensically distinguishable. Defense against unprivileged users exhausting hrtimer/alarm wheel slots.
- **CAP_WAKE_ALARM re-check on alarm-class re-arm** — even though create-time gating exists, hardened policy re-checks at each `timerfd_settime` so loss of capability mid-life disarms the wakeup path.
- **Cancel-on-set storm throttling** — `clock_was_set()` walking the cancel_list under hardened policy is rate-limited per-uid; defense against scheduler-DoS via privileged `clock_settime` floods that cause millions of timerfd wakeups.
- **Audit on cancel-on-set arming** — `(uid, pid, fd, clockid, expires)` logged for each CANCEL_ON_SET arm; defenders correlate cron-like absolute-time alarms with subsequent clock-step attacks.
- **Reject ABSTIME beyond 100-year horizon** — `it_value.tv_sec > now + 100*365*86400` returns `-EINVAL`; defense against integer-overflow attacks against per-clock expires-math.
- **Refuse interval < 100ns** — `it_interval < 100ns ∧ ≠ 0` returns `-EINVAL`; defense against runaway-reload denial-of-service.
- **GRKERNSEC_TIMERFD_INTEGRITY** — `timerfd_ctx` allocated from hardened slab with inline canary checked on each settime; defense against buffer-overflow corruption of `ctx.t.tmr` function pointers.
- **No leakage of cancel-on-set firing via timing oracle** — the wake of cancel_list under `clock_was_set` is intentionally batched with random micro-jitter so cross-process attackers cannot use cancel-on-set arming as a high-resolution clock-step detector.
- **Refuse `TFD_TIMER_CANCEL_ON_SET` if ctx clockid is not in REALTIME-class even if originally created as such but later migrated** — paranoid re-validation against ctx field tampering through UAF.

