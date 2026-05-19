---
title: "Tier-5 syscall: io_uring_enter(2) — syscall 426"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`io_uring_enter(2)` is the submission and completion-wait entry point of the io_uring subsystem. The syscall hands the kernel the right to consume up to `to_submit` SQEs from the caller's submission queue, optionally waits for at least `min_complete` CQEs to land, and atomically swaps the calling task's signal mask for the duration of the wait. Calling with `to_submit=0, min_complete=0, flags=0` is a no-op and used for cancellation polling. Calling with `to_submit>0, min_complete=0` is a pure-submit fast path. Calling with `min_complete>0` is a wait. For SQPOLL rings most submits skip the syscall entirely; `io_uring_enter` is then used only to wake the SQ kthread (`IORING_ENTER_SQ_WAKEUP`) or wait on completions (`IORING_ENTER_GETEVENTS`). Critical for: hot-path I/O submission, IOPOLL completion harvesting, signal-safe wait windows, DEFER_TASKRUN draining.

This Tier-5 covers the userspace ABI of syscall 426; runtime dispatch is owned by `io_uring/io_uring-core.md` (Tier-3, planned).

### Acceptance Criteria

- [ ] AC-1: `enter(fd, 0, 0, 0, NULL, 0)` on fresh ring returns 0.
- [ ] AC-2: Submit one nop SQE; `enter(fd, 1, 1, GETEVENTS, NULL, 0)` returns 1; CQ has one CQE.
- [ ] AC-3: Unknown flag bit → `-EINVAL`.
- [ ] AC-4: `min_complete > cq_entries` → `-EINVAL`.
- [ ] AC-5: `sigsetsize != _NSIG/8` → `-EINVAL`.
- [ ] AC-6: `EXT_ARG` with `argsz` != struct size → `-EINVAL`.
- [ ] AC-7: SQPOLL ring with `SQ_NEED_WAKEUP` set: `enter(SQ_WAKEUP)` wakes kthread; observed via SQ progress.
- [ ] AC-8: IOPOLL ring with no work + `GETEVENTS` + `min_complete=1` + no timeout → blocks; signal → `-EINTR`.
- [ ] AC-9: DEFER_TASKRUN ring with pending task work + pure-submit (no GETEVENTS) → `-EAGAIN`.
- [ ] AC-10: `R_DISABLED` ring + `to_submit > 0` → `-EBADFD`.
- [ ] AC-11: Wait with `ts` of 10ms elapsed without completions → `-ETIME`.
- [ ] AC-12: Concurrent `close(fd)` mid-wait → `-ECANCELED`.
- [ ] AC-13: `SINGLE_ISSUER` ring, second thread enters → `-EEXIST`.
- [ ] AC-14: Sigmask installed during wait; signal in mask is queued; restored on return.
- [ ] AC-15: `REGISTERED_RING` with valid slot index → resolves; invalid slot → `-EINVAL`.

### Architecture

Rookery surface in `kernel/io_uring/syscall/enter.rs`:

```rust
bitflags! {
    pub struct IoUringEnterFlags: u32 {
        const GETEVENTS         = 1 << 0;
        const SQ_WAKEUP         = 1 << 1;
        const SQ_WAIT           = 1 << 2;
        const EXT_ARG           = 1 << 3;
        const REGISTERED_RING   = 1 << 4;
        const ABS_TIMER         = 1 << 5;
        const EXT_ARG_REG       = 1 << 6;
        const NO_IOWAIT         = 1 << 7;
    }
}

#[repr(C)]
pub struct IoUringGeteventsArg {
    pub sigmask:        u64,
    pub sigmask_sz:     u32,
    pub min_wait_usec:  u32,
    pub ts:             u64,
}
```

`IoUring::enter(fd, to_submit, min_complete, flags, arg, argsz) -> isize`:
1. /* flag validation */
2. let f = IoUringEnterFlags::from_bits(flags).ok_or(-EINVAL)?;
3. /* resolve fd */
4. let ctx = if f.contains(REGISTERED_RING) {
     - current.io_uring_reg_rings.get(fd).ok_or(-EINVAL)?
   } else {
     - current.fdtable.get(fd).ok_or(-EBADF)?.as_io_uring().ok_or(-EBADFD)?
   };
5. /* enable-state */
6. if ctx.is_disabled() && to_submit > 0 { return -EBADFD; }
7. /* single-issuer claim */
8. if ctx.flags.contains(SINGLE_ISSUER) {
     - ctx.acquire_issuer(current).map_err(|_| -EEXIST)?;
   }
9. /* parse ext-arg */
10. let (sigmask, sigsetsize, min_wait_usec, ts_uptr) = if f.contains(EXT_ARG) {
      - if argsz != sizeof::<IoUringGeteventsArg>() { return -EINVAL; }
      - let ea = copy_from_user::<IoUringGeteventsArg>(arg)?;
      - (ea.sigmask as *const u8, ea.sigmask_sz as usize, ea.min_wait_usec, ea.ts as *const u8)
    } else {
      - (arg, argsz, 0, ptr::null())
    };
11. /* sigsetsize */
12. if !sigmask.is_null() && sigsetsize != NSIG_BYTES { return -EINVAL; }
13. /* SQPOLL: wake / wait */
14. if ctx.flags.contains(SQPOLL) {
      - if f.contains(SQ_WAKEUP) { ctx.sqpoll.wake(); }
      - if f.contains(SQ_WAIT) { ctx.sqpoll.wait_for_room(ts_uptr)?; }
    } else {
      - /* userspace-issuer path */
      - submitted = ctx.submit_sqes(to_submit, f);
    }
15. /* DEFER_TASKRUN gate */
16. if ctx.flags.contains(DEFER_TASKRUN) && !f.contains(GETEVENTS) && ctx.has_task_work() {
      - return -EAGAIN;
    }
17. /* wait */
18. if f.contains(GETEVENTS) || min_complete > 0 {
      - let old_mask = if !sigmask.is_null() {
          - let m = copy_from_user::<SigSet>(sigmask)?;
          - current.swap_sigmask(m & !KILL_STOP_MASK)
        } else { None };
      - let res = ctx.wait_cqe(min_complete, ts_uptr, min_wait_usec, f);
      - if let Some(om) = old_mask { current.restore_sigmask(om); }
      - res.map_err(|e| return e as isize)?;
    }
19. /* NO_IOWAIT accounting handled by wait_cqe */
20. /* release issuer */
21. if ctx.flags.contains(SINGLE_ISSUER) { ctx.release_issuer(current); }
22. return submitted as isize;

`ctx.submit_sqes(to_submit, f)`:
1. for i in 0..to_submit:
   - sqe = sq.pop_head().ok_or(break)?;
   - dispatch(sqe);
2. return i.

`ctx.wait_cqe(min_complete, ts, min_wait_usec, f)`:
1. start = clock();
2. while ctx.cq.harvested < min_complete:
   - if signal_pending() { return -EINTR; }
   - if ctx.is_torn_down() { return -ECANCELED; }
   - if ts != NULL && elapsed(start) ≥ ts && ctx.cq.harvested == 0 { return -ETIME; }
   - park(min_wait_usec_or_remaining_ts).
3. return 0.

### Out of Scope

- `io_uring/io_uring-core.md` Tier-3 dispatch loop
- `io_uring/sqpoll.md` SQPOLL kthread mechanics
- `io_uring/iopoll.md` IOPOLL ring busy-poll loop
- `io_uring/cancel.md` IORING_OP_ASYNC_CANCEL semantics
- `io_uring_setup.md` sibling
- `io_uring_register.md` sibling
- Implementation code

### signature

```c
int io_uring_enter(unsigned int fd,
                   unsigned int to_submit,
                   unsigned int min_complete,
                   unsigned int flags,
                   const sigset_t *sig,
                   size_t sigsetsize);
```

Rust ABI shim:

```rust
pub fn sys_io_uring_enter(fd: u32,
                          to_submit: u32,
                          min_complete: u32,
                          flags: u32,
                          arg: *const u8,
                          argsz: usize) -> isize;
```

When `IORING_ENTER_EXT_ARG` is set in `flags`, `(arg, argsz)` is reinterpreted as `(struct io_uring_getevents_arg *, sizeof(io_uring_getevents_arg))` providing minimum wait time, sigmask, and timeout (`io_uring_enter2`).

Syscall number: **426**.

### parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `fd` | `u32` | IN | ring fd from `io_uring_setup(2)`; or registered-ring index when `IORING_ENTER_REGISTERED_RING` |
| `to_submit` | `u32` | IN | upper bound on SQEs to consume from the SQ this call |
| `min_complete` | `u32` | IN | minimum CQEs to wait for before returning |
| `flags` | `u32` | IN | `IORING_ENTER_*` bitmask |
| `sig` / `arg` | `const sigset_t *` / `const void *` | IN | sigmask, or extended arg when `EXT_ARG` |
| `sigsetsize` / `argsz` | `size_t` | IN | size of sigmask, or size of extended-arg struct |

`struct io_uring_getevents_arg`:

```c
struct io_uring_getevents_arg {
    __u64 sigmask;       /* user ptr to sigset_t */
    __u32 sigmask_sz;    /* must be _NSIG/8 */
    __u32 min_wait_usec; /* MIN_TIMEOUT */
    __u64 ts;            /* user ptr to struct __kernel_timespec or 0 */
};
```

### return

- **Success**: non-negative count of SQEs successfully consumed by the kernel for submission (regardless of whether they completed). Wait-only calls return `0`.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EAGAIN` | `IORING_ENTER_SQ_WAIT`: SQ remained full past timeout; or DEFER_TASKRUN ring with no task work and no `GETEVENTS` |
| `EINTR` | wait interrupted by signal before `min_complete` reached |
| `EINVAL` | unknown `flags` bit; `min_complete > cq_entries`; `EXT_ARG` set but `argsz != sizeof(io_uring_getevents_arg)`; `IOPOLL` + `min_complete=0` invariant; `sigsetsize != _NSIG/8`; `to_submit` non-zero on `R_DISABLED` ring |
| `EBADF` | `fd` not a valid file descriptor |
| `EBADFD` | `fd` is not an io_uring fd; or ring is `R_DISABLED` and not yet enabled |
| `EBUSY` | IOPOLL ring with completions still pending and userspace hasn't reaped CQ before re-entering with `GETEVENTS` (legacy) |
| `EFAULT` | `sig`/`arg` not in user-space-readable address range |
| `EOPNOTSUPP` | flag bit valid in UAPI but unsupported on this build (e.g., `EXT_ARG` on legacy build) |
| `ETIME` | `EXT_ARG` `ts` timeout elapsed before `min_complete` reached |
| `ECANCELED` | ring is being torn down; concurrent `close(fd)` raced |

### abi surface

`IORING_ENTER_*` flags:

| Constant | Value | Purpose |
|---|---|---|
| `IORING_ENTER_GETEVENTS` | `1 << 0` | wait for `min_complete` CQEs |
| `IORING_ENTER_SQ_WAKEUP` | `1 << 1` | wake SQPOLL kthread |
| `IORING_ENTER_SQ_WAIT` | `1 << 2` | wait until SQ has room |
| `IORING_ENTER_EXT_ARG` | `1 << 3` | `arg` is `io_uring_getevents_arg *` |
| `IORING_ENTER_REGISTERED_RING` | `1 << 4` | `fd` is registered-ring slot index |
| `IORING_ENTER_ABS_TIMER` | `1 << 5` | `ts` in EXT_ARG is absolute clock time |
| `IORING_ENTER_EXT_ARG_REG` | `1 << 6` | EXT_ARG points to a registered region |
| `IORING_ENTER_NO_IOWAIT` | `1 << 7` | suppress iowait accounting |

Per-ring runtime constants observed via SQ ring flags (set by kernel, read by userspace):

| Flag | Value | Set when |
|---|---|---|
| `IORING_SQ_NEED_WAKEUP` | `1 << 0` | SQPOLL is asleep; userspace must `enter(SQ_WAKEUP)` |
| `IORING_SQ_CQ_OVERFLOW` | `1 << 1` | CQ overflow list non-empty |
| `IORING_SQ_TASKRUN` | `1 << 2` | task work pending (with `TASKRUN_FLAG` setup) |

Per-CQ ring flags:

| Flag | Value | Set when |
|---|---|---|
| `IORING_CQ_EVENTFD_DISABLED` | `1 << 0` | suppress eventfd notification this poll |

### compatibility contract

REQ-1: `fd` resolution:
- Default: `fd` is a process fd index.
- With `IORING_ENTER_REGISTERED_RING`: `fd` is the slot index from `IORING_REGISTER_RING_FDS`.

REQ-2: Pure-submit path (`min_complete == 0 ∧ !GETEVENTS`):
- Drain up to `to_submit` SQEs.
- Return submitted count.
- No wait.

REQ-3: Submit-and-wait (`min_complete > 0 ∨ GETEVENTS`):
- Drain up to `to_submit` SQEs.
- Then wait until CQE count since-entry ≥ `min_complete`, or signal, or timeout, or cancellation.
- Sigmask if provided is installed before wait, restored before return.

REQ-4: SQPOLL behavior:
- With `IORING_ENTER_SQ_WAKEUP`: wake SQ kthread.
- With `IORING_ENTER_SQ_WAIT`: block until SQ has room for new SQEs.
- `to_submit` is **not** consulted by kernel — SQ kthread owns SQ — but `min_complete`/`GETEVENTS` still apply.

REQ-5: IOPOLL ring:
- Completions only via in-kernel polling.
- `IORING_ENTER_GETEVENTS` triggers `io_iopoll_check`; busy-polls until `min_complete` reached.

REQ-6: DEFER_TASKRUN ring:
- Task work runs only on `IORING_ENTER_GETEVENTS`.
- Pure-submit without `GETEVENTS`: returns `-EAGAIN` if any task work is queued.
- SQE submission is always permitted.

REQ-7: `EXT_ARG` (`io_uring_enter2`):
- `argsz == sizeof(io_uring_getevents_arg)` strict.
- `sigmask_sz == _NSIG/8` strict.
- `ts == NULL` ⟹ no timeout; otherwise `__kernel_timespec` is wait timeout (relative or absolute per `ABS_TIMER`).
- `min_wait_usec > 0` ⟹ minimum wait even if completions arrive earlier (batching).

REQ-8: Sigmask semantics:
- During wait, current sigmask is `(sigmask & ~SIGKILL_MASK & ~SIGSTOP_MASK)`.
- Restored on return regardless of error.
- `sigsetsize` MUST equal kernel `_NSIG/8` (typically 8) else `-EINVAL`.

REQ-9: `R_DISABLED`:
- Any `to_submit > 0` ⟹ `-EBADFD`.
- `to_submit == 0` with `GETEVENTS` permitted (for cancellation drain).

REQ-10: Cancellation on close:
- If another thread `close(fd)`s mid-wait, woken with `-ECANCELED`.

REQ-11: Concurrency:
- Multi-issuer permitted unless `SINGLE_ISSUER` (then second issuer ⟹ `-EEXIST`).
- Within an issuer, calls are serialized on the SQ-tail update; CQ-head update is per-consumer.

REQ-12: `IORING_ENTER_NO_IOWAIT`:
- Excludes wait from `/proc/loadavg` iowait counter.
- Recommended for application-driven event loops to avoid skewing load average.

REQ-13: Returned submit count:
- `ret ≤ to_submit`.
- `ret < to_submit` is normal (e.g., link chain blocked, SQE invalid mid-batch with `SUBMIT_ALL` cleared).
- A CQE for each non-submitted SQE may still be posted with the per-op error (when validation fails after partial accept).

REQ-14: Spurious wakes:
- Wait may return with `ret == 0` and not `EINTR`/`ETIME` if completion arrived but `min_complete` not yet reached (legitimate when `min_complete > 1` and the wait deadline was set externally). Userspace MUST recheck CQ.

REQ-15: Timeout precision: `ts` is a `__kernel_timespec` (64-bit `tv_sec` on all architectures).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_known` | INVARIANT | `flags & !known ⟹ -EINVAL` |
| `sigsetsize_strict` | INVARIANT | `sig != NULL ∧ sigsetsize != NSIG/8 ⟹ -EINVAL` |
| `ext_arg_size_strict` | INVARIANT | `EXT_ARG ∧ argsz != sizeof(io_uring_getevents_arg) ⟹ -EINVAL` |
| `submit_bound` | INVARIANT | `ret ≤ to_submit` |
| `r_disabled_blocks_submit` | INVARIANT | `R_DISABLED ∧ to_submit > 0 ⟹ -EBADFD` |
| `sigmask_restored` | INVARIANT | sigmask restored on any return path |
| `single_issuer_excl` | INVARIANT | second issuer ⟹ -EEXIST |
| `defer_taskrun_eagain` | INVARIANT | DEFER_TASKRUN ∧ task work ∧ !GETEVENTS ⟹ -EAGAIN |

### Layer 2: TLA+

`uapi/io_uring_enter.tla`:
- States: `validating`, `submitting`, `waiting`, `returned`, `failed`.
- Variables: `sq_head`, `sq_tail`, `cq_head`, `cq_tail`, `sigmask`, `issuer_held`.
- Properties:
  - `safety_sigmask_restored` — sigmask restored on every termination.
  - `safety_issuer_excl` — at most one concurrent issuer when SINGLE_ISSUER.
  - `safety_submit_bound` — `ret ≤ to_submit`.
  - `liveness_eventually_returns` — every call terminates (submit-only is finite; wait is bounded by signal/timeout/cqe).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `enter` post(ok): `0 ≤ ret ≤ to_submit` | `IoUring::enter` |
| `enter` post: sigmask unchanged on entry/return | `IoUring::enter` |
| `submit_sqes` post: `submitted ≤ to_submit ∧ submitted ≤ sq.available_at_entry` | `ctx.submit_sqes` |
| `wait_cqe` post(0): `cq.harvested_since_entry ≥ min_complete` | `ctx.wait_cqe` |

### Layer 4: Verus/Creusot functional

Per-`io_uring_enter(2)` and `io_uring_enter2(2)` man pages, `Documentation/userspace-api/io_uring.rst`, and liburing `__io_uring_submit_and_wait_timeout` semantic equivalence.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

io_uring enter reinforcement:

- **Sigmask install/restore strictly paired** — defense against per-leak of sigmask across syscall boundary.
- **SINGLE_ISSUER claim enforced atomically** — defense against per-issuer race producing CQE-ordering corruption.
- **`to_submit` per-call upper-bound clamp** — defense against per-task SQ-grooming DoS.
- **`R_DISABLED` strict block on `to_submit > 0`** — defense against per-bypass-restrictions submission.
- **DEFER_TASKRUN ⟹ task work runs only under GETEVENTS** — defense against per-task-work leaking onto unrelated syscall return.

### grsecurity/pax-style reinforcement

- **PAX_RANDKSTACK on io_uring_enter entry** — randomize kernel stack at syscall entry; `io_uring_enter` is the highest-frequency io_uring syscall and a known target for stack-layout discovery via timing.
- **PaX UDEREF on every `copy_from_user` of `sigmask`, `ts`, `io_uring_getevents_arg`** — refuses kernel pointers through the extended-arg surface; the `IORING_ENTER_EXT_ARG_REG` path is a particularly attractive primitive because it points at a kernel-registered region and demands careful userspace-deref policy.
- **GRKERNSEC_BPF_HARDEN extended to io_uring runtime** — runtime ops dispatched via `io_uring_enter` form a kernel mini-VM equivalent to BPF; the same anti-tamper doctrine applies — per-uid rate-limit on `enter` calls under `kernel.io_uring_disabled ≥ 1`, audit-log any `enter` that touches `IORING_OP_URING_CMD` / `IORING_OP_NVME_PASSTHRU` / `IORING_OP_FUTEX_WAIT` / `IORING_OP_BPF` opcodes.
- **CAP_BPF gating for privileged enter modes** — `IORING_ENTER_EXT_ARG_REG` and `IORING_ENTER_NO_IOWAIT` require `CAP_BPF` under hardened policy; mirrors grsec's BPF-cap restriction model.
- **GRKERNSEC_FIFO scaled to enter-flood** — per-uid rate-limit on `io_uring_enter` calls per second; under flood (busy-loop SQPOLL wake or IOPOLL hammer) refuse with `-EAGAIN` and log; breaks the "spam enter to win a race window" exploitation pattern.
- **GRKERNSEC_HIDESYM on ring flags reflection** — SQ/CQ-ring flag fields (`IORING_SQ_NEED_WAKEUP`, `IORING_SQ_CQ_OVERFLOW`, `IORING_SQ_TASKRUN`) reveal scheduler state; under `kernel.kptr_restrict ≥ 2` mask `IORING_SQ_TASKRUN` to deny task-work fingerprinting.
- **PAX_USERCOPY on `io_uring_getevents_arg`** — validate exact-size, no-overlap copy of EXT_ARG struct; refuse copies whose source spans a sensitive SLAB boundary.
- **Audit-log every `EINTR` from a wait carrying a custom sigmask** — abnormal sigmask + EINTR pattern is associated with race-window probing; log at `LOGLEVEL_INFO` for forensic correlation.
- **Refuse `IORING_ENTER_NO_IOWAIT` from unprivileged tasks under restrictive policy** — iowait accounting is part of the kernel observability contract; allow only with `CAP_SYS_NICE` or in a privileged crosslink profile.
- **Strict serialization on `R_DISABLED` enable transition** — between `IORING_REGISTER_ENABLE_RINGS` and the first `enter`, refuse cross-thread enters that observed the pre-enable state.

