# Tier-3: io_uring/futex.c — io_uring futex ops (FUTEX_WAIT / FUTEX_WAKE / FUTEX_WAITV)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/futex.c (~336 lines)
  - io_uring/futex.h
  - kernel/futex/futex.h
  - include/uapi/linux/io_uring.h (IORING_OP_FUTEX_WAIT / _WAKE / _WAITV)
  - include/uapi/linux/futex.h (FUTEX2_*, FUTEX_WAITV_MAX)
-->

## Summary

`io_uring/futex.c` plumbs the `kernel/futex/` primitives into io_uring so userspace can `FUTEX_WAIT`/`FUTEX_WAKE`/`FUTEX_WAITV` asynchronously instead of blocking in `sys_futex`. Per-`IORING_OP_FUTEX_WAIT` prepares a `struct futex_q` (via `futex_wait_setup`) whose `q.wake` callback is `io_futex_wake_fn`; instead of sleeping on a hashed bucket the request is parked on `ctx->futex_list` and the completion (CQE) fires from task-work when another thread does `FUTEX_WAKE` on the address. Per-`IORING_OP_FUTEX_WAKE` calls `futex_wake(uaddr, FLAGS_STRICT | flags, val, mask)` synchronously and posts the CQE immediately. Per-`IORING_OP_FUTEX_WAITV` accepts a vector of up to `FUTEX_WAITV_MAX` (= 128) `struct futex_waitv` entries via `futex_parse_waitv` + `futex_wait_multiple_setup` and completes as soon as the first wakes (race-free via `io_futexv_claim` bit-lock on `ifd->owned`). Per-cancellation iterates `ctx->futex_list` via `io_cancel_remove` and dequeues with `futex_unqueue` / `futex_unqueue_multiple`. Per-ring teardown reaps inflight requests via `io_futex_remove_all`. Critical for: lock-free async waits, futex-driven user-space schedulers, NPTL-style fast paths combined with the io_uring CQE model.

This Tier-3 covers `io_uring/futex.c` (~336 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_futex` | per-SQE state (uaddr, val, mask, flags, nr) | `IoFutex` |
| `struct io_futex_data` | per-FUTEX_WAIT async state (`futex_q` + back-ptr) | `IoFutexData` |
| `struct io_futexv_data` | per-FUTEX_WAITV async state (owned bit + `futex_vector[]`) | `IoFutexvData` |
| `io_futex_cache_init()` / `_free()` | per-ctx slab cache for `io_futex_data` | `IoUring::futex_cache_init` / `_free` |
| `io_futex_prep()` | per-SQE prep for WAIT/WAKE | `IoUring::futex_prep` |
| `io_futexv_prep()` | per-SQE prep for WAITV | `IoUring::futexv_prep` |
| `io_futex_wait()` | per-issue FUTEX_WAIT | `IoUring::futex_wait` |
| `io_futexv_wait()` | per-issue FUTEX_WAITV | `IoUring::futexv_wait` |
| `io_futex_wake()` | per-issue FUTEX_WAKE | `IoUring::futex_wake` |
| `io_futex_wake_fn()` | per-bucket wake callback (WAIT) | `IoUring::futex_wake_fn` |
| `io_futex_wakev_fn()` | per-bucket wake callback (WAITV) | `IoUring::futex_wakev_fn` |
| `io_futex_complete()` | per-task-work complete (WAIT) | `IoUring::futex_complete` |
| `io_futexv_complete()` | per-task-work complete (WAITV) | `IoUring::futexv_complete` |
| `__io_futex_complete()` | per-CQE-post + hash unlink | `IoUring::futex_complete_inner` |
| `io_futex_cancel()` | per-cancel-by-cd | `IoUring::futex_cancel` |
| `__io_futex_cancel()` | per-req cancel inner | `IoUring::futex_cancel_inner` |
| `io_futex_remove_all()` | per-ring teardown reap | `IoUring::futex_remove_all` |
| `io_futexv_claim()` | per-waitv owned-bit lock | `IoUring::futexv_claim` |
| `futex_q_init` | per-`futex_q` template | shared from `kernel/futex` |
| `futex_wait_setup()` | per-WAIT enqueue + load | shared from `kernel/futex` |
| `futex_parse_waitv()` | per-WAITV vector parse | shared from `kernel/futex` |
| `futex_wait_multiple_setup()` | per-WAITV multi-enqueue | shared from `kernel/futex` |
| `futex_unqueue()` / `_multiple()` | per-dequeue | shared from `kernel/futex` |
| `futex_wake()` | per-WAKE | shared from `kernel/futex` |
| `__futex_wake_mark()` | per-wakeup pre-claim | shared from `kernel/futex` |
| `futex2_to_flags()` / `futex_flags_valid()` | per-flag-translate / validate | shared from `kernel/futex` |
| `futex_validate_input()` | per-val/mask sanity | shared from `kernel/futex` |
| `FUTEX_WAITV_MAX` (128) | per-vector cap | UAPI |
| `FUTEX2_VALID_MASK` | per-flag mask | UAPI |
| `FLAGS_STRICT` | per-strict-wake semantics | kernel/futex |

## Compatibility contract

REQ-1: struct io_futex:
- file: per-SQE file (always NULL for futex; placeholder).
- uaddr: per-user futex address (from sqe->addr).
- futex_val: per-expected-value compared against `*uaddr` at WAIT time (from sqe->addr2).
- futex_mask: per-bitset mask for WAIT/WAKE (from sqe->addr3); zero rejected at WAIT.
- futex_flags: per-FUTEX2 flag word translated from `sqe->fd` via `futex2_to_flags`.
- futex_nr: per-WAITV count (from sqe->len), `1..=FUTEX_WAITV_MAX`.
- futexv_unqueued: per-WAITV flag indicating whether `futex_wait_multiple_setup` already unqueued (woke during setup).

REQ-2: struct io_futex_data:
- q: per-`struct futex_q` (initialized from `futex_q_init`, wake = `io_futex_wake_fn`, bitset = mask).
- req: per-back-pointer to owning `io_kiocb`.
- Allocated via per-ctx `futex_cache` (IO_FUTEX_ALLOC_CACHE_MAX = 32).

REQ-3: struct io_futexv_data:
- owned: per-claim bit-lock (bit 0; single-wake-completion arbitration).
- futexv[]: flex-array of `struct futex_vector` (length = `iof->futex_nr`).
- Allocated via `kzalloc_flex(struct io_futexv_data, futexv, iof->futex_nr, GFP_KERNEL_ACCOUNT)`; freed via `io_req_async_data_free`.

REQ-4: io_futex_cache_init(ctx) / _free(ctx):
- init: `io_alloc_cache_init(&ctx->futex_cache, 32, sizeof(io_futex_data), 0)`.
- free: `io_alloc_cache_free(&ctx->futex_cache, kfree)`.

REQ-5: io_futex_prep(req, sqe) — FUTEX_WAIT / FUTEX_WAKE:
- if sqe->len ∨ sqe->futex_flags ∨ sqe->buf_index ∨ sqe->file_index: -EINVAL.
- iof.uaddr = u64_to_user_ptr(sqe->addr).
- iof.futex_val = sqe->addr2.
- iof.futex_mask = sqe->addr3.
- flags = sqe->fd; if flags & ~FUTEX2_VALID_MASK: -EINVAL.
- iof.futex_flags = futex2_to_flags(flags).
- if !futex_flags_valid(iof.futex_flags): -EINVAL.
- if !futex_validate_input(iof.futex_flags, iof.futex_val) ∨ !futex_validate_input(iof.futex_flags, iof.futex_mask): -EINVAL.
- io_req_track_inflight(req) — so file-exit cancel finds it.
- return 0.

REQ-6: io_futexv_prep(req, sqe) — FUTEX_WAITV:
- if sqe->fd ∨ sqe->buf_index ∨ sqe->file_index ∨ sqe->addr2 ∨ sqe->futex_flags ∨ sqe->addr3: -EINVAL.
- iof.uaddr = u64_to_user_ptr(sqe->addr) (user pointer to `struct futex_waitv[]`).
- iof.futex_nr = sqe->len.
- if !iof.futex_nr ∨ iof.futex_nr > FUTEX_WAITV_MAX: -EINVAL.
- ifd = kzalloc_flex(io_futexv_data, futexv, iof.futex_nr, GFP_KERNEL_ACCOUNT).
- if !ifd: -ENOMEM.
- ret = futex_parse_waitv(ifd.futexv, iof.uaddr, iof.futex_nr, io_futex_wakev_fn, req).
- if ret: kfree(ifd); return ret.
- io_req_track_inflight(req).
- iof.futexv_unqueued = 0.
- req.flags |= REQ_F_ASYNC_DATA; req.async_data = ifd.
- return 0.

REQ-7: io_futex_wait(req, issue_flags) — FUTEX_WAIT issue:
- if !iof.futex_mask: ret = -EINVAL; goto done.
- io_ring_submit_lock(ctx, issue_flags).
- ifd = io_cache_alloc(&ctx->futex_cache, GFP_NOWAIT).
- if !ifd: ret = -ENOMEM; goto done_unlock.
- req.flags |= REQ_F_ASYNC_DATA; req.async_data = ifd.
- ifd.q = futex_q_init.
- ifd.q.bitset = iof.futex_mask.
- ifd.q.wake = io_futex_wake_fn.
- ifd.req = req.
- ret = futex_wait_setup(iof.uaddr, iof.futex_val, iof.futex_flags, &ifd.q, NULL, NULL).
- if !ret /* enqueued, will fire via wake_fn */:
  - hlist_add_head(&req.hash_node, &ctx->futex_list).
  - io_ring_submit_unlock(ctx, issue_flags).
  - return IOU_ISSUE_SKIP_COMPLETE.
- done_unlock: io_ring_submit_unlock(ctx, issue_flags).
- done: if ret < 0: req_set_fail(req).
- io_req_set_res(req, ret, 0); io_req_async_data_free(req).
- return IOU_COMPLETE.

REQ-8: io_futexv_wait(req, issue_flags) — FUTEX_WAITV issue:
- io_ring_submit_lock(ctx, issue_flags).
- ret = futex_wait_multiple_setup(ifd.futexv, iof.futex_nr, &woken).
- if ret < 0: io_ring_submit_unlock; req_set_fail; io_req_set_res(req, ret, 0); io_req_async_data_free; return IOU_COMPLETE.
- if ret == 0 /* enqueued, no wake yet */:
  - __set_current_state(TASK_RUNNING) /* async: we don't block */.
  - hlist_add_head(&req.hash_node, &ctx->futex_list).
- else /* ret == 1: woke during setup; futex_wait_multiple_setup already unqueued */:
  - iof.futexv_unqueued = 1.
  - if woken != -1: io_req_set_res(req, woken, 0).
- io_ring_submit_unlock(ctx, issue_flags).
- return IOU_ISSUE_SKIP_COMPLETE.

REQ-9: io_futex_wake(req, issue_flags) — FUTEX_WAKE issue:
- ret = futex_wake(iof.uaddr, FLAGS_STRICT | iof.futex_flags, iof.futex_val, iof.futex_mask).
- /* FLAGS_STRICT: waking 0 yields result 0 (not -EWOULDBLOCK) per commit 43adf8449510 */
- if ret < 0: req_set_fail(req).
- io_req_set_res(req, ret, 0).
- return IOU_COMPLETE.

REQ-10: io_futex_wake_fn(wake_q, q) — per-FUTEX_WAIT bucket wake:
- ifd = container_of(q, struct io_futex_data, q).
- req = ifd.req.
- if !__futex_wake_mark(q) /* lost wake-race */: return.
- io_req_set_res(req, 0, 0).
- req.io_task_work.func = io_futex_complete.
- io_req_task_work_add(req).

REQ-11: io_futex_wakev_fn(wake_q, q) — per-FUTEX_WAITV bucket wake:
- req = q.wake_data.
- ifd = req.async_data.
- if !io_futexv_claim(ifd) /* another vector slot already won */: __futex_wake_mark(q); return.
- if !__futex_wake_mark(q): return.
- io_req_set_res(req, 0, 0).
- req.io_task_work.func = io_futexv_complete.
- io_req_task_work_add(req).

REQ-12: io_futexv_claim(ifd):
- if test_bit(0, &ifd.owned) ∨ test_and_set_bit_lock(0, &ifd.owned): return false.
- return true.

REQ-13: io_futex_complete(tw_req, tw) — FUTEX_WAIT task-work:
- io_tw_lock(ctx, tw).
- io_cache_free(&ctx->futex_cache, req.async_data).
- io_req_async_data_clear(req, 0).
- __io_futex_complete(tw_req, tw).

REQ-14: io_futexv_complete(tw_req, tw) — FUTEX_WAITV task-work:
- io_tw_lock(req.ctx, tw).
- if !iof.futexv_unqueued:
  - res = futex_unqueue_multiple(ifd.futexv, iof.futex_nr).
  - if res != -1: io_req_set_res(req, res, 0).
- io_req_async_data_free(req).
- __io_futex_complete(tw_req, tw).

REQ-15: __io_futex_complete(tw_req, tw):
- hlist_del_init(&tw_req.req.hash_node) /* unlink from ctx->futex_list */.
- io_req_task_complete(tw_req, tw) /* post CQE + req release */.

REQ-16: __io_futex_cancel(req):
- if req.opcode == IORING_OP_FUTEX_WAIT:
  - ifd = req.async_data.
  - if !futex_unqueue(&ifd.q): return false /* already woken / about to wake */.
  - req.io_task_work.func = io_futex_complete.
- else /* FUTEX_WAITV */:
  - ifd = req.async_data.
  - if !io_futexv_claim(ifd): return false.
  - req.io_task_work.func = io_futexv_complete.
- hlist_del_init(&req.hash_node).
- io_req_set_res(req, -ECANCELED, 0).
- io_req_task_work_add(req).
- return true.

REQ-17: io_futex_cancel(ctx, cd, issue_flags) / io_futex_remove_all(ctx, tctx, cancel_all):
- io_cancel_remove(ctx, cd, issue_flags, &ctx->futex_list, __io_futex_cancel).
- Teardown variant: io_cancel_remove_all(ctx, tctx, &ctx->futex_list, cancel_all, __io_futex_cancel).

## Acceptance Criteria

- [ ] AC-1: SQE with IORING_OP_FUTEX_WAIT, valid uaddr, mask != 0 — issues; CQE arrives only after concurrent FUTEX_WAKE.
- [ ] AC-2: SQE with IORING_OP_FUTEX_WAIT, mask == 0 — IOU_COMPLETE with -EINVAL.
- [ ] AC-3: SQE with IORING_OP_FUTEX_WAIT, `*uaddr != val` at setup — `futex_wait_setup` returns non-zero; CQE posted immediately.
- [ ] AC-4: SQE with IORING_OP_FUTEX_WAKE — synchronous CQE; result = waker count from `futex_wake`.
- [ ] AC-5: FUTEX_WAKE wakes 0 waiters — CQE result = 0 (FLAGS_STRICT semantics), not -EAGAIN.
- [ ] AC-6: SQE with IORING_OP_FUTEX_WAITV, nr in `1..=128` — issues; CQE arrives with first-woken index.
- [ ] AC-7: SQE FUTEX_WAITV with nr == 0 or nr > 128 — prep returns -EINVAL.
- [ ] AC-8: FUTEX_WAITV race: two concurrent wakes hit different vector entries — only one CQE posted; second `io_futexv_claim` returns false.
- [ ] AC-9: Async cancel (IORING_OP_ASYNC_CANCEL) targeting pending FUTEX_WAIT — CQE -ECANCELED; futex_q dequeued.
- [ ] AC-10: Async cancel of FUTEX_WAITV — CQE -ECANCELED; `futex_unqueue_multiple` invoked in completion.
- [ ] AC-11: Ring teardown with pending FUTEX_WAIT — `io_futex_remove_all` cancels each; no leak of `io_futex_data`.
- [ ] AC-12: Prep with `sqe->fd` containing unknown FUTEX2 bits — -EINVAL.
- [ ] AC-13: Prep with futex_val violating `futex_validate_input` (e.g., FUTEX2_NUMA mis-aligned) — -EINVAL.
- [ ] AC-14: Concurrent FUTEX_WAIT and explicit cancel — `futex_unqueue` returning false yields cancel failure (-EALREADY semantics handled by `io_cancel_remove`).

## Architecture

```
struct IoFutex {            // per io_kiocb cmd payload
  uaddr: UserPtr<u32>,      // sqe->addr
  futex_val: u64,           // sqe->addr2
  futex_mask: u64,          // sqe->addr3
  futex_flags: u32,
  futex_nr: u32,            // WAITV count
  futexv_unqueued: bool,
}

struct IoFutexData {        // FUTEX_WAIT async-data
  q: FutexQ,                // wake = io_futex_wake_fn
  req: *IoKiocb,
}

struct IoFutexvData {       // FUTEX_WAITV async-data
  owned: AtomicBitSet,      // bit 0 is the claim-lock
  futexv: [FutexVector; N], // flex-array of futex_nr entries
}
```

`IoUring::futex_prep(req, sqe) -> Result<(), Errno>` (REQ-5):
1. Validate sqe fields are zero where required.
2. Copy uaddr / val / mask.
3. Translate FUTEX2 flags; validate.
4. `io_req_track_inflight(req)`.

`IoUring::futexv_prep(req, sqe) -> Result<(), Errno>` (REQ-6):
1. Validate sqe fields.
2. Range-check futex_nr against `FUTEX_WAITV_MAX`.
3. Allocate `IoFutexvData` flex-array.
4. `futex_parse_waitv(...wakev_fn..., req)`.
5. Track inflight; attach `async_data`.

`IoUring::futex_wait(req, issue_flags) -> IouCode` (REQ-7):
1. Reject mask == 0.
2. Submit-lock.
3. Allocate `IoFutexData` from `ctx.futex_cache` (GFP_NOWAIT — no sleep under lock).
4. Initialize q from `futex_q_init`; q.bitset = mask; q.wake = wake_fn; q.req-back-pointer.
5. `futex_wait_setup`: enqueues on bucket + verifies `*uaddr == val` (returns 0 ⟹ enqueued; non-zero ⟹ completed-or-error).
6. On 0: hlist-add to `ctx.futex_list`; return IOU_ISSUE_SKIP_COMPLETE.
7. On non-zero: set result; free async-data; return IOU_COMPLETE.

`IoUring::futexv_wait(req, issue_flags) -> IouCode` (REQ-8):
1. Submit-lock.
2. `futex_wait_multiple_setup(futexv, nr, &woken)`:
   - 0: enqueued on all buckets.
   - 1: woke during setup; vector already unqueued.
   - <0: error.
3. On 0: TASK_RUNNING; hlist-add.
4. On 1: mark `futexv_unqueued`; record woken index.
5. On <0: complete with error.

`IoUring::futex_wake(req, issue_flags) -> IouCode` (REQ-9):
1. `futex_wake(uaddr, FLAGS_STRICT | flags, val, mask)` (val here is the max-wakers count).
2. Set result = ret; complete synchronously.

`IoUring::futex_wake_fn(wake_q, q)` (REQ-10):
1. Recover `IoFutexData` via container_of.
2. `__futex_wake_mark(q)`: pre-claim — false ⟹ another waker won, abort.
3. Set req result = 0; queue task-work `io_futex_complete`.

`IoUring::futex_wakev_fn(wake_q, q)` (REQ-11):
1. Recover `req` from `q.wake_data`.
2. `io_futexv_claim(ifd)`: first-completion arbitration.
   - If lost: `__futex_wake_mark(q)` to consume the wake; return.
3. `__futex_wake_mark(q)`: still required to seal the wake.
4. Set result = 0; queue `io_futexv_complete`.

`IoUring::futex_complete(tw_req, tw)` (REQ-13):
1. tw-lock.
2. Return `IoFutexData` to per-ctx alloc cache.
3. Clear async-data pointer.
4. `__io_futex_complete`: unlink hash_node; post CQE.

`IoUring::futexv_complete(tw_req, tw)` (REQ-14):
1. tw-lock.
2. If not already unqueued during setup-race: `futex_unqueue_multiple(futexv, nr)` — returns index of woken slot (>=0), or -1 if none.
3. If unqueue reported a valid slot: overwrite result with that index.
4. Free `IoFutexvData`.
5. `__io_futex_complete`: unlink + CQE.

`IoUring::futex_cancel_inner(req) -> bool` (REQ-16):
1. Discriminate by opcode (FUTEX_WAIT vs FUTEX_WAITV).
2. For WAIT: `futex_unqueue(&ifd.q)` must succeed (false ⟹ wake already in flight ⟹ cancel-failed).
3. For WAITV: `io_futexv_claim(ifd)` must succeed (false ⟹ another wake won ⟹ cancel-failed).
4. On success: set task-work func to the matching complete handler, unlink hash_node, set result = -ECANCELED, schedule task-work.

`IoUring::futex_cancel(ctx, cd, flags)` (REQ-17):
1. Delegate to generic `io_cancel_remove(ctx, cd, flags, &ctx.futex_list, futex_cancel_inner)`.

`IoUring::futex_remove_all(ctx, tctx, all)`:
1. Delegate to `io_cancel_remove_all(ctx, tctx, &ctx.futex_list, all, futex_cancel_inner)` — invoked from ring fd-close and exit-handler paths.

State machine for FUTEX_WAIT:
```
prep ─► (validated, inflight tracked) ─► futex_wait ─► futex_wait_setup
                                                       │
       ┌───────────────────────────────────────────────┴───────────────────┐
       │ setup ret = 0 (enqueued)                                          │ setup ret != 0
       ▼                                                                   ▼
   parked on ctx.futex_list ─► [external FUTEX_WAKE] ─► wake_fn      complete (res = ret)
       │                                                  │
       │                                                  ▼
       │                                          io_req_task_work_add(complete)
       │                                                  │
       └───────► [cancel] ─► futex_unqueue OK ─► complete (res = -ECANCELED)
                              │
                              └─ fail ─► cancel returns false; wake will fire
```

State machine for FUTEX_WAITV: same shape; arbitration moves from kernel/futex bucket-lock to `ifd.owned` bit-lock; unqueue-multiple replaces unqueue.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prep_rejects_invalid_flags` | INVARIANT | per-futex_prep: flags & ~FUTEX2_VALID_MASK ⟹ -EINVAL. |
| `prep_rejects_zero_mask_on_wait` | INVARIANT | per-futex_wait: futex_mask == 0 ⟹ -EINVAL pre-alloc. |
| `waitv_nr_bounded` | INVARIANT | per-futexv_prep: futex_nr ∈ [1, FUTEX_WAITV_MAX]. |
| `wake_fn_single_completion` | INVARIANT | per-WAIT: at most one task-work scheduled per `futex_q`. |
| `wakev_claim_unique` | INVARIANT | per-WAITV: only first `io_futexv_claim` wins; all subsequent return false. |
| `cache_alloc_under_submit_lock` | INVARIANT | per-futex_wait: `io_cache_alloc` GFP_NOWAIT (no sleep). |
| `cancel_unqueue_consistency` | INVARIANT | per-cancel: if `futex_unqueue` fails, cancel returns false; no double task-work. |
| `inflight_tracking_set` | INVARIANT | per-prep: `io_req_track_inflight(req)` always called on success. |

### Layer 2: TLA+

`io_uring/futex.tla`:
- States: prep / armed / parked / waking / completing / cancelled / freed.
- Variables: `ctx.futex_list`, `q.lock_ptr`, `ifd.owned`, `req.async_data`.
- Actions: FutexWaitIssue / FutexWakeFire / FutexCancel / FutexComplete / FutexWaitvIssue / FutexvWakeFire / FutexvUnqueue.
- Properties:
  - `safety_one_cqe_per_req` — every WAIT/WAITV/WAKE req posts exactly one CQE.
  - `safety_no_use_after_free` — async_data freed only in `_complete` after CQE; no wake/cancel after free.
  - `safety_waitv_claim_exclusive` — at most one of {wake_fn fires, cancel succeeds} per `ifd.owned`.
  - `safety_inflight_balance` — `io_req_track_inflight` count balanced by `io_req_task_complete`.
  - `liveness_wake_propagates_to_cqe` — FUTEX_WAKE on a parked uaddr ⟹ eventually CQE posted for that WAIT.
  - `liveness_ring_teardown_reaps_all` — exit-handler with parked WAITs ⟹ all reaped via `io_futex_remove_all`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `futex_prep` post: req tracked-inflight ∧ flags valid ∨ Err | `IoUring::futex_prep` |
| `futexv_prep` post: ifd allocated ∧ futexv parsed ∨ Err (no partial state) | `IoUring::futexv_prep` |
| `futex_wait` post: ret in {IOU_ISSUE_SKIP_COMPLETE, IOU_COMPLETE}; async_data alive on SKIP, freed on COMPLETE | `IoUring::futex_wait` |
| `futex_wake_fn` post: at most one `io_req_task_work_add` per `q` | `IoUring::futex_wake_fn` |
| `futex_wakev_fn` post: `ifd.owned` bit 0 set; task-work scheduled | `IoUring::futex_wakev_fn` |
| `futex_cancel_inner` post: ret true ⟹ task-work scheduled with -ECANCELED; ret false ⟹ no side-effect | `IoUring::futex_cancel_inner` |
| `futexv_complete` post: futex_unqueue_multiple called iff !futexv_unqueued | `IoUring::futexv_complete` |
| `futex_wake` post: synchronous CQE; FLAGS_STRICT honored | `IoUring::futex_wake` |

### Layer 4: Verus/Creusot functional

`Per IORING_OP_FUTEX_WAIT issue → futex_wait_setup → bucket-park → external FUTEX_WAKE → io_futex_wake_fn → task-work → __io_futex_complete (CQE posted)` semantic equivalence: per `io_uring/futex.c:io_futex_wait/io_futex_wake_fn` and per `kernel/futex/waitwake.c:futex_wait_setup/__futex_wake_mark`.

`Per IORING_OP_FUTEX_WAITV issue → futex_wait_multiple_setup → vector-park → first FUTEX_WAKE → io_futex_wakev_fn → io_futexv_claim → task-work → __io_futex_complete (CQE = first-woken index)` semantic equivalence: per `io_uring/futex.c:io_futexv_wait/io_futex_wakev_fn/io_futexv_complete` and per `kernel/futex/waitwake.c:futex_wait_multiple_setup/futex_unqueue_multiple`.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

io_uring-futex reinforcement:

- **Per-FUTEX2_VALID_MASK flag check** — defense against per-future-bit ABI smuggling.
- **Per-futex_validate_input on val + mask** — defense against per-FUTEX2_NUMA mis-alignment / per-private-vs-shared mismatch.
- **Per-mask == 0 rejected for WAIT** — defense against per-impossible-wake (no bit could match).
- **Per-FUTEX_WAITV_MAX = 128 cap** — defense against per-unbounded vector alloc.
- **Per-`kzalloc_flex` for futexv** — defense against per-overflow-multiply of nr * sizeof(futex_vector).
- **Per-`io_futexv_claim` single-winner bit-lock** — defense against per-double-CQE on multi-bucket wake.
- **Per-`__futex_wake_mark` pre-claim** — defense against per-spurious wake-and-cancel race.
- **Per-`io_req_track_inflight` at prep** — defense against per-orphan-on-file-exit.
- **Per-`futex_unqueue` check in cancel** — defense against per-cancel-vs-wake double task-work.
- **Per-FLAGS_STRICT in WAKE** — defense against per-ambiguous zero-waker result.
- **Per-`io_cache_alloc` GFP_NOWAIT under submit-lock** — defense against per-sleep-with-lock.
- **Per-`hlist_del_init` idempotent unlink** — defense against per-double-unlink.
- **Per-`io_req_async_data_free`/`_clear` exactly-once** — defense against per-async-data UAF.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-checked copy on `struct futex_waitv` arrays for `FUTEX_WAITV` and on `uaddr2`/`val3` field reads when carried in 128-byte SQEs.
- **PAX_KERNEXEC** — write-protects the `io_op_defs[IORING_OP_FUTEX_*]` slots and the `futex_q.wake` indirect callback table.
- **PAX_RANDKSTACK** — per-call stack-offset randomization on `io_futex_wait`, `io_futex_wake`, and `io_futex_waitv`.
- **PAX_REFCOUNT** — saturating trap on `futex_q.requeue_pi_key` refcount and on `req->refs` taken across the `task_work` callback chain.
- **PAX_MEMORY_SANITIZE** — zeroes `struct io_futex_data` on free via `io_cache_free`; scrubs `futex_q` slab on `hlist_del_init`.
- **PAX_UDEREF on uaddr** — every `uaddr` SQE field is dereferenced via `futex_get_value_locked` / `get_futex_key` under explicit user-pointer fault; kernel-mode access to `uaddr` is forbidden; defends against per-kernel-pointer-aliased futex tag.
- **PAX_RAP / kCFI** — forward-edge CFI on `futex_q.wake` (`futex_wake_fn`) dispatch invoked from io_uring task-work.
- **GRKERNSEC_HIDESYM** — strips `io_futex_*` and `futex_q` helpers from `/proc/kallsyms`.
- **GRKERNSEC_DMESG** — restricts `pr_debug` futex traces and `WARN_ON_ONCE(futex_q)` shouts to CAP_SYSLOG.
- **FUTEX_PRIVATE_FLAG mandatory** — io_uring futex ops require `FUTEX_PRIVATE_FLAG` in the issued `futex_op`; cross-process / shared-mm futexes are rejected at prep with `-EINVAL`; defends against per-cross-namespace futex side-channel and per-shared-page futex-key collision attacks across rings.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `kernel/futex/` core (`waitwake.c`, `pi.c`, `core.c`, `requeue.c`, `syscalls.c`) — covered in `kernel/futex.md` Tier-3 (if expanded; otherwise Tier-2 sibling)
- `io_uring/cancel.c` (`io_cancel_remove`, `io_cancel_remove_all`) — covered in `io_uring/wait-cancel.md`
- `io_uring/io_uring.c` (`io_req_task_*`, `io_tw_lock`) — covered in `io_uring/io_uring-core.md`
- `io_uring/alloc_cache.c` (`io_cache_alloc`, `io_alloc_cache_init`) — covered in `io_uring/io_uring-core.md`
- `io_uring/opdef.c` IORING_OP_FUTEX_* registration — covered in `io_uring/00-overview.md`
- Userspace robust-futex / PI / requeue semantics
- Implementation code
