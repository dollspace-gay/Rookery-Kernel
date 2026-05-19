---
title: "Tier-5 syscall: io_cancel(2) — syscall 210"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`io_cancel(2)` requests that a previously submitted Linux AIO operation (an `iocb`) be cancelled before it completes. The kernel looks up the iocb in the context's active-requests list, calls the underlying file-op's `aio_cancel` method (if any), and on success copies the resulting completion event into the user-supplied `io_event`. Most cancellation requests fail with `-EINVAL` because the vast majority of file operations — notably block-device direct I/O — do not implement `aio_cancel` (they hand work to the block layer which cannot retract submitted requests).

Where `io_cancel` does work: socket I/O via certain async-poll paths, USB userspace I/O. In practice the syscall is largely cosmetic; its primary role is part of `io_destroy`'s cleanup path. The newer `io_uring` family includes `IORING_OP_ASYNC_CANCEL` which is far more useful.

Critical for: best-effort cancellation in databases that retry on timeout, AIO test suites, and as the in-kernel primitive used by `io_destroy`.

This Tier-5 covers `io_cancel(2)` on x86_64; the syscall is generic-unistd-equivalent.

### Acceptance Criteria

- [ ] AC-1: `io_submit` an iocb on a cancellable fd (e.g., timerfd async poll); `io_cancel(ctx, iocb, &ev)` returns 0; `ev.res == -ECANCELED`.
- [ ] AC-2: `io_submit` an iocb on a non-cancellable fd (block direct I/O); `io_cancel(ctx, iocb, &ev)` returns `-EAGAIN`.
- [ ] AC-3: `io_cancel` on a completed iocb returns `-EINVAL`.
- [ ] AC-4: `io_cancel` on an unknown iocb returns `-EINVAL`.
- [ ] AC-5: `io_cancel` with bad ctx returns `-EINVAL`.
- [ ] AC-6: `io_cancel(ctx, iocb, NULL)` returns `-EFAULT`.
- [ ] AC-7: `io_cancel(ctx, NULL, &ev)` returns `-EFAULT`.
- [ ] AC-8: Successful cancel does NOT duplicate the event to the ring buffer.
- [ ] AC-9: Failed cancel (`-EAGAIN`) leaves iocb in active_reqs; subsequent natural completion still publishes a ring event.
- [ ] AC-10: Concurrent `io_destroy` and `io_cancel` on the same iocb: one returns 0, the other `-EINVAL`; no corruption.
- [ ] AC-11: Per-`CONFIG_AIO=n`: `io_cancel(any)` returns `-ENOSYS`.
- [ ] AC-12: Per-LTP `io_cancel01..02` pass.

### Architecture

```rust
#[syscall(nr = 210, abi = "sysv")]
pub fn sys_io_cancel(
    ctx_id: u64,
    iocb_ptr: UserPtr<UserIocb>,
    result: UserPtr<IoEvent>,
) -> isize {
    IoCancel::dispatch(ctx_id, iocb_ptr, result)
}
```

`IoCancel::dispatch(ctx_id, iocb_ptr, result) -> isize`:
1. if iocb_ptr.is_null() || result.is_null() { return Err(EFAULT); }
2. let mm = current.mm();
3. let ctx = mm.ioctx_table.lookup(ctx_id).ok_or(EINVAL)?;
4. /* Find by user pointer */
5. let kiocb = ctx.active_reqs.remove_by_user_ptr(iocb_ptr.as_u64()).ok_or(EINVAL)?;
6. /* Invoke aio_cancel */
7. let cancel_fn = kiocb.file.f_op.aio_cancel;
8. let rc = match cancel_fn {
9.   Some(f) => f(kiocb.as_ref()),
10.  None    => Err(EAGAIN),
11. };
12. match rc {
13.   Ok(()) => {
14.     /* Build event */
15.     let ev = IoEvent {
16.       data: kiocb.user_data,
17.       obj:  iocb_ptr.as_u64(),
18.       res:  -ECANCELED as i64,
19.       res2: 0,
20.     };
21.     result.write(&ev)?;
22.     ctx.reqs_active.fetch_sub(1, SeqCst);
23.     drop(kiocb);
24.     Ok(0)
25.   }
26.   Err(EAGAIN) => {
27.     /* Cannot cancel — DO NOT re-insert; natural completion path retains a reference */
28.     /* Modern kernels: simply don't re-add to active_reqs; the completion path checks dead-flag. */
29.     ctx.active_reqs.insert(kiocb);     /* historical re-insert; preserved here */
30.     Err(EAGAIN)
31.   }
32.   Err(e) => Err(e),
33. }

### Out of Scope

- `io_setup(2)`, `io_submit(2)`, `io_getevents(2)`, `io_destroy(2)` (separate Tier-5 docs).
- `io_uring` family with proper async-cancel (Tier-3 in `fs/io_uring/00-overview.md`).
- Per-file-operations `aio_cancel` semantics (Tier-3 in each driver / fs).
- Block-layer cancellation (it does not exist; that's why most cancels fail).
- Implementation code.

### signature

```c
int io_cancel(aio_context_t ctx_id, struct iocb *iocb, struct io_event *result);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ctx_id` | `aio_context_t` | in | AIO context returned by `io_setup`. |
| `iocb` | `struct iocb *` | in | User pointer matching the `iocb*` previously passed to `io_submit`. |
| `result` | `struct io_event *` | out | Completion event populated on successful cancel. |

### return value

| Value | Meaning |
|---|---|
| `0` | Cancellation succeeded; `*result` populated with the cancellation event. |
| `-1` + `errno` | Failure (either lookup or cancel rejected). |

### errors

| errno | Trigger |
|---|---|
| `EINVAL`  | `ctx_id` does not refer to a context in current mm; `iocb` is not an active request in this context; `iocb` already completed. |
| `EFAULT`  | `iocb` or `result` not a valid user pointer. |
| `EAGAIN`  | The operation does not support cancellation (most common outcome). |
| `ENOSYS`  | Kernel compiled with `CONFIG_AIO=n`. |

### abi surface

```text
__NR_io_cancel (x86_64) = 210
__NR_io_cancel (i386)   = 249

struct iocb {                            /* user-supplied; opaque to kernel beyond user-VA matching */
    __u64 aio_data;
    __u32 aio_key;
    __u32 aio_rw_flags;
    __u16 aio_lio_opcode;
    __s16 aio_reqprio;
    __u32 aio_fildes;
    __u64 aio_buf;
    __u64 aio_nbytes;
    __s64 aio_offset;
    __u64 aio_reserved2;
    __u32 aio_flags;
    __u32 aio_resfd;
};

struct io_event {                        /* populated by kernel on cancel-success */
    __u64 data;                          /* iocb->aio_data */
    __u64 obj;                           /* iocb user pointer (recast __u64) */
    __s64 res;                           /* -EINTR or -ECANCELED */
    __s64 res2;                          /* 0 */
};
```

### Limits

```text
ctx_id      : valid aio_context_t from prior io_setup
iocb        : must match an in-flight submission in this context
cancellable : depends entirely on file_operations.aio_cancel — rare
```

### compatibility contract

REQ-1: Syscall number is **210** on x86_64. ABI-stable since 2.5.32.

REQ-2: `ctx_id` looked up in `current->mm->ioctx_table`. Missing ⟹ `-EINVAL`. No mutation.

REQ-3: `iocb` (user pointer) is matched against the `obj` field of every entry in `ctx->active_reqs`. Match key: the user-VA identity. No match ⟹ `-EINVAL`.

REQ-4: If found, the in-kernel `aio_kiocb` is removed from `active_reqs` (so a racing completion does NOT also produce a ring event) and its `ki_filp->f_op->aio_cancel` is called.

REQ-5: If `aio_cancel` is `NULL` for the file's `f_op`, `io_cancel` returns `-EAGAIN`. The iocb is RE-INSERTED into `active_reqs` so its natural completion still produces a ring event. (This re-insertion is the historically-buggy path; modern kernels treat the absence of `aio_cancel` as immediate failure without removal.)

REQ-6: If `aio_cancel` returns success, the kernel:
- populates `*result` with `io_event{ data = iocb.aio_data, obj = iocb_uptr, res = -ECANCELED, res2 = 0 }`;
- frees the `aio_kiocb`;
- decrements `ctx->reqs_active`;
- returns 0.

REQ-7: If `aio_cancel` returns `-EAGAIN` ("operation already past cancellation point"), `io_cancel` returns `-EAGAIN` and the iocb remains scheduled for natural completion.

REQ-8: The cancellation event is written ONLY to `*result`. It is NOT also pushed to the ring buffer (would be a double-completion).

REQ-9: Per-`result == NULL`: returns `-EFAULT` before any cancellation attempt.

REQ-10: Per-`iocb == NULL`: returns `-EFAULT`.

REQ-11: Per-CONFIG_AIO=n: `io_cancel` returns `-ENOSYS`.

REQ-12: Per-mm scoping: only the calling task's mm's contexts are reachable. A different process (no CLONE_VM) cannot cancel another's iocb (lookup fails).

REQ-13: Per-thread sharing same mm: any thread may cancel any iocb in any context owned by that mm. There is no per-thread submission-tracking.

REQ-14: Concurrency with `io_destroy`: if `io_destroy` is running concurrently, `io_cancel` either succeeds (race won) or returns `-EINVAL` (context already removed). No torn state.

REQ-15: Concurrency with natural completion: if the iocb completes between lookup and `aio_cancel`, the call returns `-EINVAL` (iocb no longer in active_reqs) and the completion event has already been pushed to the ring.

REQ-16: Best-effort semantics: man-page explicitly states this. Userspace MUST treat `-EAGAIN` as "I/O will complete naturally; consult ring".

REQ-17: Per-audit: cancellation success NOT audited by default (high-volume during io_destroy); failures NOT audited (purely diagnostic).

REQ-18: x32 / i386: same semantics, different numbers.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `userptr_fault_safe` | INVARIANT | per-dispatch: NULL iocb / NULL result ⟹ -EFAULT. |
| `lookup_in_current_mm` | INVARIANT | per-dispatch: ctx lookup via current.mm only. |
| `not_found_einval` | INVARIANT | per-dispatch: missing iocb ⟹ -EINVAL, no event written. |
| `no_double_event` | INVARIANT | per-cancel-ok: ring not touched; only *result written. |
| `aio_cancel_null_eagain` | INVARIANT | per-dispatch: f_op.aio_cancel == NULL ⟹ -EAGAIN. |
| `reqs_active_dec_on_success` | INVARIANT | per-cancel-ok: ctx.reqs_active -= 1. |

### Layer 2: TLA+

`fs/aio_cancel.tla`:
- States: per-ctx active_reqs set; per-ctx ring buffer (FIFO); per-iocb (pending|cancelled|complete).
- Properties:
  - `safety_no_double_complete` — cancel-ok ⟹ no ring event AND user gets event in *result.
  - `safety_cancel_or_natural` — every iocb terminates in exactly one of {cancel-ok, natural-completion, destroy}.
  - `safety_reqs_active_consistent` — reqs_active == |active_reqs|.
  - `liveness_terminates` — io_cancel returns in bounded time.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `dispatch` post: ok ⟹ *result.res == -ECANCELED | `IoCancel::dispatch` |
| `dispatch` post: ok ⟹ reqs_active -= 1 | `IoCancel::dispatch` |
| `dispatch` post: EAGAIN ⟹ iocb remains in active_reqs | `IoCancel::dispatch` |
| `dispatch` post: err ⟹ *result untouched | `IoCancel::dispatch` |

### Layer 4: Verus/Creusot functional

Per-`io_cancel(2)` man-page semantic equivalence. LTP `io_cancel01..02` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`io_cancel(2)` reinforcement:

- **Per-mm scoping of `ioctx_table`** — defense against per-cross-process iocb cancellation.
- **Per-user-pointer lookup** — defense against per-stale-kernel-pointer reuse (kiocb keyed by user VA).
- **Per-no-double-event** — defense against per-userspace ring-corruption via duplicate event.
- **Per-`reqs_active` exact accounting** — defense against per-mismatched-refcount UAF.
- **Per-RCU-protected active_reqs** — defense against per-concurrent-destroy data race.
- **Per-result-pointer pre-check** — defense against per-cancel-with-NULL-result silent loss.
- **Per-aio_cancel best-effort** — defense against per-misuse-as-block-cancel (returns -EAGAIN cleanly).
- **Per-destroy-race resolution** — exactly-one-winner via active_reqs.remove atomic.

### grsecurity / pax surface

- **PaX UDEREF on `iocb` and `result`** — defense against per-malicious-user-pointer kernel deref bug.
- **PAX_RANDKSTACK at io_cancel entry** — randomizes kernel stack offset per call.
- **GRKERNSEC_KMEM** — `aio_kiocb` internals never exposed via `/dev/kmem` or `/proc/<pid>/mem`.
- **GRKERNSEC_CHROOT_NO_AIO** — inside grsec chroot, io_cancel returns `-EPERM` if policy denies AIO entirely (matches io_setup denial).
- **Per-grsec RBAC** — `+I` flag (aio-allow) gates per-subject; default policy permits.
- **Per-grsec audit** — io_cancel NOT audited by default (low forensic value, high call rate during io_destroy); opt-in via per-subject policy.
- **PaX REFCOUNT** — `kiocb.refcount` and `ctx.reqs_active` PAX_REFCOUNT-protected; under/over-flow panics.
- **GRKERNSEC_HIDESYM** — `aio_kiocb` struct symbols not leaked via kallsyms or BPF introspection.
- **Per-grsec aio_cancel-validate** — the file's `aio_cancel` function pointer is validated against the kernel text section to defeat function-pointer hijacks before invocation.

