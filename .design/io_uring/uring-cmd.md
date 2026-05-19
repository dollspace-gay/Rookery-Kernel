# Tier-3: io_uring/uring_cmd.c — IORING_OP_URING_CMD async ioctl surface

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/uring_cmd.c (~408 lines)
  - io_uring/uring_cmd.h
  - include/linux/io_uring/cmd.h
  - include/uapi/linux/io_uring.h (IORING_OP_URING_CMD, IORING_OP_URING_CMD128, IORING_URING_CMD_FIXED, IORING_URING_CMD_MULTISHOT)
-->

## Summary

`IORING_OP_URING_CMD` provides an **async ioctl-like surface** for filesystems and char devices that opt in via `struct file_operations.uring_cmd`. The SQE carries a 64-bit `cmd_op` plus an inline command payload (16 bytes at sqe->cmd[], or 80 bytes with `IORING_SETUP_SQE128` / `IORING_OP_URING_CMD128`). Drivers return `-EIOCBQUEUED` to defer completion and later call `io_uring_cmd_done()`; `-EAGAIN` triggers a reissue path through the workqueue. The submission may register itself as cancelable (`IORING_URING_CMD_CANCELABLE`), use a registered (fixed) buffer (`IORING_URING_CMD_FIXED`), or run multishot (`IORING_URING_CMD_MULTISHOT`) for many CQEs from one SQE. Used by: **NVMe passthrough** (NVME_URING_CMD_IO/_VEC/_IO_VEC/_ADMIN), **ublk** (UBLK_U_IO_*), **sound (compress_offload)**, and **xfs scrub**. Critical for: async device passthrough, zero-syscall fast paths, BPF-style retry semantics, and per-file uring_cmd_iopoll integration.

This Tier-3 covers `io_uring/uring_cmd.c` (~408 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_uring_cmd` | per-cmd context (flags, cmd_op, sqe ptr) | `IoUringCmd` |
| `struct io_async_cmd` | per-async backing (vec + sqes[2] copy) | `IoAsyncCmd` |
| `io_uring_cmd_prep()` | per-SQE parse | `UringCmd::prep` |
| `io_uring_cmd()` | per-issue dispatch | `UringCmd::issue` |
| `io_uring_cmd_sqe_copy()` | per-SQE async copy | `UringCmd::sqe_copy` |
| `io_uring_cmd_cleanup()` | per-req cleanup | `UringCmd::cleanup` |
| `io_uring_cmd_mark_cancelable()` | per-driver opt-in cancel | `UringCmd::mark_cancelable` |
| `io_uring_cmd_del_cancelable()` | per-cmd un-register | `UringCmd::del_cancelable` |
| `io_uring_try_cancel_uring_cmd()` | per-ring cancel-walk | `UringCmd::try_cancel` |
| `__io_uring_cmd_do_in_task()` | per-driver tw enqueue | `UringCmd::do_in_task` |
| `__io_uring_cmd_done()` | per-driver completion | `UringCmd::done_inner` |
| `io_uring_cmd_import_fixed()` | per-registered-buffer | `UringCmd::import_fixed` |
| `io_uring_cmd_import_fixed_vec()` | per-registered-iovec | `UringCmd::import_fixed_vec` |
| `io_uring_cmd_issue_blocking()` | per-iowq punt | `UringCmd::issue_blocking` |
| `io_cmd_poll_multishot()` | per-apoll-multishot arm | `UringCmd::poll_multishot` |
| `io_uring_cmd_buffer_select()` | per-multishot kbuf select | `UringCmd::buffer_select` |
| `io_uring_mshot_cmd_post_cqe()` | per-multishot CQE post | `UringCmd::mshot_post_cqe` |
| `io_uring_cmd_post_mshot_cqe32()` | per-multishot big-CQE post | `UringCmd::post_mshot_cqe32` |
| `io_cmd_cache_free()` | per-cmd-cache free hook | `UringCmd::cache_free` |
| `IORING_URING_CMD_FIXED` | UAPI flag | shared |
| `IORING_URING_CMD_MULTISHOT` | UAPI flag | shared |
| `IORING_URING_CMD_CANCELABLE` | internal flag | shared |
| `IORING_URING_CMD_REISSUE` | internal flag | shared |
| `IORING_URING_CMD_MASK` | UAPI flag mask | shared |

## Compatibility contract

REQ-1: struct io_uring_cmd:
- file: per-target file (req->file mirror).
- sqe: pointer to current SQE (initially user SQE; flips to copied `ac->sqes[]` after `sqe_copy`).
- task_work_cb: per-driver task-work callback.
- cmd_op: 32-bit driver opcode (from sqe->cmd_op).
- flags: per-cmd flag bits — IORING_URING_CMD_FIXED, IORING_URING_CMD_MULTISHOT, IORING_URING_CMD_CANCELABLE, IORING_URING_CMD_REISSUE (top 8 bits are kernel-private).
- pdu: per-driver opaque area for issue-side state.

REQ-2: struct io_async_cmd:
- vec: `struct iou_vec` for fixed-iovec import path.
- sqes[2]: per-SQE128 copy buffer (2 SQEs back-to-back; covers both standard and 128-byte SQEs).

REQ-3: io_uring_cmd_prep(req, sqe):
- /* Reject reserved bits */
- if sqe->__pad1: return -EINVAL.
- /* Latch user flags */
- ioucmd.flags = READ_ONCE(sqe->uring_cmd_flags).
- if ioucmd.flags & ~IORING_URING_CMD_MASK: return -EINVAL.
- /* FIXED ⇒ buf_index latched; FIXED ∧ MULTISHOT excluded */
- if ioucmd.flags & IORING_URING_CMD_FIXED:
  - if ioucmd.flags & IORING_URING_CMD_MULTISHOT: return -EINVAL.
  - req.buf_index = READ_ONCE(sqe->buf_index).
- /* MULTISHOT ⇔ REQ_F_BUFFER_SELECT */
- if !!(ioucmd.flags & IORING_URING_CMD_MULTISHOT) != !!(req.flags & REQ_F_BUFFER_SELECT): return -EINVAL.
- /* Latch cmd opcode */
- ioucmd.cmd_op = READ_ONCE(sqe->cmd_op).
- /* Allocate async backing from per-ctx cmd_cache */
- ac = io_uring_alloc_async_data(&ctx.cmd_cache, req).
- if !ac: return -ENOMEM.
- /* Initial sqe pointer aliases user SQE */
- ioucmd.sqe = sqe.
- return 0.

REQ-4: io_uring_cmd(req, issue_flags):
- /* File must export ->uring_cmd */
- if !file.f_op.uring_cmd: return -EOPNOTSUPP.
- /* LSM hook */
- ret = security_uring_cmd(ioucmd).
- if ret: return ret.
- /* Per-SQE128 propagation */
- if ctx.flags & IORING_SETUP_SQE128 ∨ req.opcode == IORING_OP_URING_CMD128:
  - issue_flags |= IO_URING_F_SQE128.
- /* Per-CQE32 propagation */
- if ctx.flags & (IORING_SETUP_CQE32 | IORING_SETUP_CQE_MIXED):
  - issue_flags |= IO_URING_F_CQE32.
- /* Compat ABI */
- if io_is_compat(ctx): issue_flags |= IO_URING_F_COMPAT.
- /* IOPOLL gate per uring_cmd_iopoll exported */
- if ctx.flags & IORING_SETUP_IOPOLL ∧ file.f_op.uring_cmd_iopoll:
  - req.flags |= REQ_F_IOPOLL.
  - issue_flags |= IO_URING_F_IOPOLL.
  - req.iopoll_completed = 0.
  - if ctx.flags & IORING_SETUP_HYBRID_IOPOLL:
    - req.flags &= ~REQ_F_IOPOLL_STATE.
    - req.iopoll_start = ktime_get_ns().
- /* Dispatch to driver */
- ret = file.f_op.uring_cmd(ioucmd, issue_flags).
- /* MULTISHOT success ⇒ stay armed */
- if ioucmd.flags & IORING_URING_CMD_MULTISHOT ∧ ret ≥ 0:
  - return IOU_ISSUE_SKIP_COMPLETE.
- /* Reissue: -EAGAIN ⇒ caller (io_uring core) will re-queue async */
- if ret == -EAGAIN:
  - ioucmd.flags |= IORING_URING_CMD_REISSUE.
  - return -EAGAIN.
- /* Deferred ⇒ driver will call io_uring_cmd_done */
- if ret == -EIOCBQUEUED: return -EIOCBQUEUED.
- /* Inline completion */
- if ret < 0: req_set_fail(req).
- io_req_uring_cleanup(req, issue_flags).
- io_req_set_res(req, ret, 0).
- return IOU_COMPLETE.

REQ-5: io_uring_cmd_sqe_copy(req):
- /* Idempotent guard: REQ_F_SQE_COPIED should preclude re-entry */
- if ioucmd.sqe == ac.sqes: WARN_ON_ONCE; return.
- /* Copy 1×SQE or 2×SQE depending on SQE128 */
- size = uring_sqe_size(req).
- memcpy(ac.sqes, ioucmd.sqe, size).
- ioucmd.sqe = ac.sqes.

REQ-6: uring_sqe_size(req):
- if ctx.flags & IORING_SETUP_SQE128 ∨ req.opcode == IORING_OP_URING_CMD128:
  - return 2 * sizeof(struct io_uring_sqe).
- return sizeof(struct io_uring_sqe).

REQ-7: io_uring_cmd_mark_cancelable(cmd, issue_flags) — EXPORT_SYMBOL_GPL:
- /* IOPOLL exclusion: hash_node aliases iopoll state */
- if req.flags & REQ_F_IOPOLL: return.
- /* Idempotent register */
- if !(cmd.flags & IORING_URING_CMD_CANCELABLE):
  - cmd.flags |= IORING_URING_CMD_CANCELABLE.
  - io_ring_submit_lock(ctx, issue_flags).
  - hlist_add_head(&req.hash_node, &ctx.cancelable_uring_cmd).
  - io_ring_submit_unlock(ctx, issue_flags).

REQ-8: io_uring_cmd_del_cancelable(cmd, issue_flags):
- if !(cmd.flags & IORING_URING_CMD_CANCELABLE): return.
- cmd.flags &= ~IORING_URING_CMD_CANCELABLE.
- io_ring_submit_lock(ctx, issue_flags).
- hlist_del(&req.hash_node).
- io_ring_submit_unlock(ctx, issue_flags).

REQ-9: io_uring_try_cancel_uring_cmd(ctx, tctx, cancel_all):
- /* Caller must hold uring_lock */
- lockdep_assert_held(&ctx.uring_lock).
- ret = false.
- hlist_for_each_entry_safe(req in ctx.cancelable_uring_cmd):
  - if !cancel_all ∧ req.tctx != tctx: continue.
  - if cmd.flags & IORING_URING_CMD_CANCELABLE:
    - file.f_op.uring_cmd(cmd, IO_URING_F_CANCEL | IO_URING_F_COMPLETE_DEFER).
    - ret = true.
- io_submit_flush_completions(ctx).
- return ret.

REQ-10: __io_uring_cmd_do_in_task(ioucmd, task_work_cb, flags) — EXPORT_SYMBOL_GPL:
- /* APOLL-multishot must not arm task_work via this path */
- if WARN_ON_ONCE(req.flags & REQ_F_APOLL_MULTISHOT): return.
- req.io_task_work.func = task_work_cb.
- __io_req_task_work_add(req, flags).

REQ-11: __io_uring_cmd_done(ioucmd, ret, res2, issue_flags, is_cqe32) — EXPORT_SYMBOL_GPL:
- /* MULTISHOT must use mshot_post_cqe instead */
- if WARN_ON_ONCE(req.flags & REQ_F_APOLL_MULTISHOT): return.
- /* Unregister cancelable */
- io_uring_cmd_del_cancelable(ioucmd, issue_flags).
- /* Set ret */
- if ret < 0: req_set_fail(req).
- io_req_set_res(req, ret, 0).
- /* Big-CQE path */
- if is_cqe32:
  - if ctx.flags & IORING_SETUP_CQE_MIXED: req.cqe.flags |= IORING_CQE_F_32.
  - req.big_cqe.extra1 = res2; req.big_cqe.extra2 = 0.
- /* Cleanup async data */
- io_req_uring_cleanup(req, issue_flags).
- /* IOPOLL: store-release iopoll_completed */
- if req.flags & REQ_F_IOPOLL: smp_store_release(&req.iopoll_completed, 1).
- /* Deferred completion from caller context */
- else if issue_flags & IO_URING_F_COMPLETE_DEFER:
  - if WARN_ON_ONCE(issue_flags & IO_URING_F_UNLOCKED): return.
  - io_req_complete_defer(req).
- /* Otherwise enqueue task-work completion */
- else:
  - req.io_task_work.func = io_req_task_complete.
  - io_req_task_work_add(req).

REQ-12: io_uring_cmd_import_fixed(ubuf, len, rw, iter, ioucmd, issue_flags) — EXPORT_SYMBOL_GPL:
- /* FIXED required */
- if WARN_ON_ONCE(!(ioucmd.flags & IORING_URING_CMD_FIXED)): return -EINVAL.
- return io_import_reg_buf(req, iter, ubuf, len, rw, issue_flags).

REQ-13: io_uring_cmd_import_fixed_vec(ioucmd, uvec, uvec_segs, ddir, iter, issue_flags) — EXPORT_SYMBOL_GPL:
- if WARN_ON_ONCE(!(ioucmd.flags & IORING_URING_CMD_FIXED)): return -EINVAL.
- /* Prep iovec into ac.vec */
- ret = io_prep_reg_iovec(req, &ac.vec, uvec, uvec_segs).
- if ret: return ret.
- return io_import_reg_vec(ddir, iter, req, &ac.vec, uvec_segs, issue_flags).

REQ-14: io_uring_cmd_issue_blocking(ioucmd):
- /* Driver requests iowq punt for blocking work */
- io_req_queue_iowq(req).

REQ-15: io_cmd_poll_multishot(cmd, issue_flags, mask):
- /* Already armed ⇒ noop success */
- if req.flags & REQ_F_APOLL_MULTISHOT: return 0.
- req.flags |= REQ_F_APOLL_MULTISHOT.
- /* Strip EPOLLONESHOT for multishot semantics */
- mask &= ~EPOLLONESHOT.
- ret = io_arm_apoll(req, issue_flags, mask).
- return ret == IO_APOLL_OK ? -EIOCBQUEUED : -ECANCELED.

REQ-16: io_uring_cmd_buffer_select(ioucmd, buf_group, *len, issue_flags) — EXPORT_SYMBOL_GPL:
- if !(ioucmd.flags & IORING_URING_CMD_MULTISHOT): return { val = -EINVAL }.
- if WARN_ON_ONCE(!io_do_buffer_select(req)): return { val = -EINVAL }.
- return io_buffer_select(req, len, buf_group, issue_flags).

REQ-17: io_uring_mshot_cmd_post_cqe(ioucmd, sel, issue_flags) — EXPORT_SYMBOL_GPL:
- /* Non-multishot: caller goes to single-shot completion */
- if !(ioucmd.flags & IORING_URING_CMD_MULTISHOT): return true.
- /* Positive: commit buffer + post CQE_F_MORE */
- if sel.val > 0:
  - cflags = io_put_kbuf(req, sel.val, sel.buf_list).
  - if io_req_post_cqe(req, sel.val, cflags | IORING_CQE_F_MORE): return false.
- /* Otherwise recycle buffer and signal terminal completion */
- io_kbuf_recycle(req, sel.buf_list, issue_flags).
- if sel.val < 0: req_set_fail(req).
- io_req_set_res(req, sel.val, cflags).
- return true.

REQ-18: io_uring_cmd_post_mshot_cqe32(cmd, issue_flags, cqe[2]):
- /* Must be in MULTISHOT context */
- if WARN_ON_ONCE(!(issue_flags & IO_URING_F_MULTISHOT)): return false.
- return io_req_post_cqe32(req, cqe).

## Acceptance Criteria

- [ ] AC-1: file_operations.uring_cmd absent: SQE returns -EOPNOTSUPP.
- [ ] AC-2: sqe->__pad1 nonzero ⇒ -EINVAL at prep.
- [ ] AC-3: uring_cmd_flags & ~IORING_URING_CMD_MASK ⇒ -EINVAL at prep.
- [ ] AC-4: IORING_URING_CMD_FIXED ∧ IORING_URING_CMD_MULTISHOT ⇒ -EINVAL at prep.
- [ ] AC-5: IORING_URING_CMD_MULTISHOT XOR REQ_F_BUFFER_SELECT ⇒ -EINVAL at prep.
- [ ] AC-6: Driver returning -EIOCBQUEUED: completion deferred to io_uring_cmd_done().
- [ ] AC-7: Driver returning -EAGAIN: IORING_URING_CMD_REISSUE set; caller reissues via iowq.
- [ ] AC-8: SQE128 / IORING_OP_URING_CMD128: 2*sizeof(sqe) copied on async copy; IO_URING_F_SQE128 propagated.
- [ ] AC-9: mark_cancelable on REQ_F_IOPOLL request: no-op (iopoll exclusion).
- [ ] AC-10: io_uring_try_cancel_uring_cmd: invokes ->uring_cmd(IO_URING_F_CANCEL | IO_URING_F_COMPLETE_DEFER) for each cancelable cmd matching tctx (or all).
- [ ] AC-11: import_fixed without IORING_URING_CMD_FIXED: WARN + -EINVAL.
- [ ] AC-12: MULTISHOT cmd success ret≥0: returns IOU_ISSUE_SKIP_COMPLETE (stays armed).
- [ ] AC-13: io_cmd_poll_multishot: IO_APOLL_OK ⇒ -EIOCBQUEUED; otherwise -ECANCELED.
- [ ] AC-14: __io_uring_cmd_done on REQ_F_APOLL_MULTISHOT: WARN_ON_ONCE + early-return (must use mshot path).
- [ ] AC-15: CQE32/CQE_MIXED + is_cqe32: extras1/2 populated; IORING_CQE_F_32 set in mixed mode.

## Architecture

```
struct IoUringCmd {
  file: *mut File,                  // mirrors req.file
  sqe: *const IoUringSqe,           // user SQE initially; flips to ac.sqes[] after sqe_copy
  task_work_cb: Option<TaskWorkFn>,
  cmd_op: u32,
  flags: u32,                       // IORING_URING_CMD_{FIXED,MULTISHOT,CANCELABLE,REISSUE}
  pdu: [u8; 32],                    // driver opaque
}

struct IoAsyncCmd {
  vec: IouVec,
  sqes: [IoUringSqe; 2],            // covers SQE64 + SQE128
}
```

`UringCmd::prep(req, sqe) -> Result<()>`:
1. if sqe.__pad1 != 0: return Err(EINVAL).
2. flags = READ_ONCE(sqe.uring_cmd_flags).
3. if flags & !IORING_URING_CMD_MASK: return Err(EINVAL).
4. if flags & IORING_URING_CMD_FIXED:
   - if flags & IORING_URING_CMD_MULTISHOT: return Err(EINVAL).
   - req.buf_index = READ_ONCE(sqe.buf_index).
5. if (flags & IORING_URING_CMD_MULTISHOT) ⊕ (req.flags & REQ_F_BUFFER_SELECT): return Err(EINVAL).
6. ioucmd.flags = flags.
7. ioucmd.cmd_op = READ_ONCE(sqe.cmd_op).
8. ac = io_uring_alloc_async_data(&ctx.cmd_cache, req).
9. if !ac: return Err(ENOMEM).
10. ioucmd.sqe = sqe.
11. Ok(()).

`UringCmd::issue(req, issue_flags) -> i32`:
1. if !file.f_op.uring_cmd: return -EOPNOTSUPP.
2. security_uring_cmd(ioucmd)?.
3. if ctx.flags & IORING_SETUP_SQE128 ∨ req.opcode == IORING_OP_URING_CMD128: issue_flags |= IO_URING_F_SQE128.
4. if ctx.flags & (IORING_SETUP_CQE32 | IORING_SETUP_CQE_MIXED): issue_flags |= IO_URING_F_CQE32.
5. if io_is_compat(ctx): issue_flags |= IO_URING_F_COMPAT.
6. if ctx.flags & IORING_SETUP_IOPOLL ∧ file.f_op.uring_cmd_iopoll:
   - req.flags |= REQ_F_IOPOLL.
   - issue_flags |= IO_URING_F_IOPOLL.
   - req.iopoll_completed = 0.
   - if ctx.flags & IORING_SETUP_HYBRID_IOPOLL:
     - req.flags &= !REQ_F_IOPOLL_STATE.
     - req.iopoll_start = ktime_get_ns().
7. ret = file.f_op.uring_cmd(ioucmd, issue_flags).
8. if ioucmd.flags & IORING_URING_CMD_MULTISHOT ∧ ret ≥ 0: return IOU_ISSUE_SKIP_COMPLETE.
9. if ret == -EAGAIN: ioucmd.flags |= IORING_URING_CMD_REISSUE; return -EAGAIN.
10. if ret == -EIOCBQUEUED: return -EIOCBQUEUED.
11. if ret < 0: req_set_fail(req).
12. UringCmd::cleanup_inner(req, issue_flags).
13. io_req_set_res(req, ret, 0).
14. return IOU_COMPLETE.

`UringCmd::done_inner(ioucmd, ret, res2, issue_flags, is_cqe32)`:
1. if WARN_ON_ONCE(req.flags & REQ_F_APOLL_MULTISHOT): return.
2. UringCmd::del_cancelable(ioucmd, issue_flags).
3. if ret < 0: req_set_fail(req).
4. io_req_set_res(req, ret, 0).
5. if is_cqe32:
   - if ctx.flags & IORING_SETUP_CQE_MIXED: req.cqe.flags |= IORING_CQE_F_32.
   - req.big_cqe.extra1 = res2; req.big_cqe.extra2 = 0.
6. UringCmd::cleanup_inner(req, issue_flags).
7. match completion-mode:
   - REQ_F_IOPOLL ⇒ smp_store_release(&req.iopoll_completed, 1).
   - IO_URING_F_COMPLETE_DEFER ⇒ io_req_complete_defer(req).
   - else ⇒ req.io_task_work.func = io_req_task_complete; io_req_task_work_add(req).

`UringCmd::mark_cancelable(cmd, issue_flags)`:
1. if req.flags & REQ_F_IOPOLL: return.
2. if !(cmd.flags & IORING_URING_CMD_CANCELABLE):
   - cmd.flags |= IORING_URING_CMD_CANCELABLE.
   - io_ring_submit_lock(ctx, issue_flags).
   - hlist_add_head(&req.hash_node, &ctx.cancelable_uring_cmd).
   - io_ring_submit_unlock(ctx, issue_flags).

`UringCmd::try_cancel(ctx, tctx, cancel_all) -> bool`:
1. lockdep_assert_held(&ctx.uring_lock).
2. ret = false.
3. for req in ctx.cancelable_uring_cmd (safe iter):
   - if !cancel_all ∧ req.tctx != tctx: continue.
   - if cmd.flags & IORING_URING_CMD_CANCELABLE:
     - file.f_op.uring_cmd(cmd, IO_URING_F_CANCEL | IO_URING_F_COMPLETE_DEFER).
     - ret = true.
4. io_submit_flush_completions(ctx).
5. return ret.

`UringCmd::sqe_copy(req)`:
1. if WARN_ON_ONCE(ioucmd.sqe == ac.sqes): return.
2. size = uring_sqe_size(req).
3. memcpy(ac.sqes, ioucmd.sqe, size).
4. ioucmd.sqe = ac.sqes.

`UringCmd::cleanup_inner(req, issue_flags)`:
1. if issue_flags & IO_URING_F_UNLOCKED: return.
2. io_alloc_cache_vec_kasan(&ac.vec).
3. if ac.vec.nr > IO_VEC_CACHE_SOFT_CAP: io_vec_free(&ac.vec).
4. if io_alloc_cache_put(&ctx.cmd_cache, ac):
   - ioucmd.sqe = NULL.
   - io_req_async_data_clear(req, REQ_F_NEED_CLEANUP).

`UringCmd::mshot_post_cqe(ioucmd, sel, issue_flags) -> bool`:
1. if !(ioucmd.flags & IORING_URING_CMD_MULTISHOT): return true.
2. if sel.val > 0:
   - cflags = io_put_kbuf(req, sel.val, sel.buf_list).
   - if io_req_post_cqe(req, sel.val, cflags | IORING_CQE_F_MORE): return false.
3. io_kbuf_recycle(req, sel.buf_list, issue_flags).
4. if sel.val < 0: req_set_fail(req).
5. io_req_set_res(req, sel.val, cflags).
6. return true.

`UringCmd::poll_multishot(cmd, issue_flags, mask) -> i32`:
1. if req.flags & REQ_F_APOLL_MULTISHOT: return 0.
2. req.flags |= REQ_F_APOLL_MULTISHOT.
3. mask &= !EPOLLONESHOT.
4. ret = io_arm_apoll(req, issue_flags, mask).
5. return if ret == IO_APOLL_OK { -EIOCBQUEUED } else { -ECANCELED }.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prep_rejects_pad1` | INVARIANT | per-prep: sqe->__pad1 nonzero ⇒ -EINVAL. |
| `prep_rejects_unknown_flags` | INVARIANT | per-prep: flags & ~IORING_URING_CMD_MASK ⇒ -EINVAL. |
| `fixed_xor_multishot` | INVARIANT | per-prep: FIXED ∧ MULTISHOT ⇒ -EINVAL. |
| `multishot_iff_buffer_select` | INVARIANT | per-prep: MULTISHOT ⇔ REQ_F_BUFFER_SELECT. |
| `cancel_iopoll_excluded` | INVARIANT | per-mark_cancelable: REQ_F_IOPOLL ⇒ no-op. |
| `cancelable_hlist_balanced` | INVARIANT | per-mark/del: add/remove balanced under submit_lock. |
| `apoll_multishot_done_excluded` | INVARIANT | per-done: REQ_F_APOLL_MULTISHOT ⇒ WARN + early-return. |
| `import_fixed_requires_fixed_flag` | INVARIANT | per-import: !FIXED ⇒ -EINVAL. |
| `try_cancel_holds_uring_lock` | INVARIANT | per-try_cancel: ctx.uring_lock held. |
| `sqe_copy_size_matches_sqe128` | INVARIANT | per-sqe_copy: SQE128 ⇒ 2*sizeof(sqe); else 1*sizeof(sqe). |
| `done_iopoll_uses_store_release` | INVARIANT | per-done: REQ_F_IOPOLL ⇒ smp_store_release on iopoll_completed. |

### Layer 2: TLA+

`io_uring/uring-cmd.tla`:
- Per-prep + per-issue + per-driver-deferred + per-done + per-cancel + per-multishot.
- Properties:
  - `safety_no_completion_without_issue` — per-CQE: a CQE is posted only after issue or driver-done.
  - `safety_eiocbqueued_implies_eventual_done` — per-driver: -EIOCBQUEUED ⇒ eventual io_uring_cmd_done call.
  - `safety_eagain_reissue_bounded` — per-EAGAIN: reissue path runs at most once per submission attempt.
  - `safety_multishot_terminal_clears_more` — per-MULTISHOT: terminal CQE has !CQE_F_MORE.
  - `safety_cancelable_set_invariant` — per-mark/del: hash_node membership ⇔ IORING_URING_CMD_CANCELABLE.
  - `safety_iopoll_no_cancelable` — per-IOPOLL: REQ_F_IOPOLL ⇒ not on cancelable_uring_cmd list.
  - `liveness_per_submission_completes_or_cancels` — per-cmd: eventually CQE posted or canceled.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `UringCmd::prep` post: ret ⟹ ac allocated ∧ flags ⊆ MASK | `UringCmd::prep` |
| `UringCmd::issue` post: ret ∈ {≥0, -EAGAIN, -EIOCBQUEUED, IOU_ISSUE_SKIP_COMPLETE, IOU_COMPLETE} | `UringCmd::issue` |
| `UringCmd::done_inner` post: CQE posted XOR deferred-tw scheduled | `UringCmd::done_inner` |
| `UringCmd::mark_cancelable` post: !REQ_F_IOPOLL ⟹ flags has CANCELABLE; on list | `UringCmd::mark_cancelable` |
| `UringCmd::del_cancelable` post: flags !CANCELABLE; off list | `UringCmd::del_cancelable` |
| `UringCmd::try_cancel` post: every cancelable matching predicate has had ->uring_cmd(IO_URING_F_CANCEL) called | `UringCmd::try_cancel` |
| `UringCmd::sqe_copy` post: ioucmd.sqe == ac.sqes ∧ bytes-equal up to uring_sqe_size | `UringCmd::sqe_copy` |
| `UringCmd::mshot_post_cqe` post: sel.val>0 ⟹ kbuf committed ∧ CQE_F_MORE; else recycled ∧ terminal | `UringCmd::mshot_post_cqe` |
| `UringCmd::poll_multishot` post: APOLL_OK ⟹ -EIOCBQUEUED ∧ REQ_F_APOLL_MULTISHOT set | `UringCmd::poll_multishot` |

### Layer 4: Verus/Creusot functional

`Per-IORING_OP_URING_CMD → prep (latch flags, cmd_op; alloc ac) → issue (LSM gate, SQE128/CQE32/IOPOLL flags, dispatch to f_op.uring_cmd) → driver-deferred (-EIOCBQUEUED) ∨ driver-reissue (-EAGAIN) ∨ inline complete → done (set_res, big_cqe extras, cleanup, tw-or-defer) ∨ multishot post_cqe loop (CQE_F_MORE until terminal) ∨ cancel-walk (per-ring iter; CMD(IO_URING_F_CANCEL))` semantic equivalence: per-Documentation/userspace-api/io_uring/io_uring_cmd.rst and per nvme/host/ioctl.c, drivers/block/ublk_drv.c, sound/core/compress_offload.c upstream consumers.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

uring_cmd reinforcement:

- **Per-LSM `security_uring_cmd` gate** — defense against per-driver-cmd-without-policy.
- **Per-`sqe->__pad1` zero-check** — defense against per-future-ABI-collision via reserved bits.
- **Per-`IORING_URING_CMD_MASK` strict whitelist** — defense against per-unknown-flag escalation.
- **Per-FIXED ⊕ MULTISHOT mutual-exclusion** — defense against per-fixed-buf semantic-conflict in multishot.
- **Per-MULTISHOT ⇔ REQ_F_BUFFER_SELECT bi-implication** — defense against per-multishot-without-kbuf misuse.
- **Per-REQ_F_IOPOLL excluded from cancelable hlist** — defense against per-hash_node aliasing with iopoll state (data corruption).
- **Per-`cancelable_uring_cmd` walk under uring_lock** — defense against per-concurrent cancel race.
- **Per-`io_uring_cmd_done` REQ_F_APOLL_MULTISHOT WARN** — defense against per-driver misuse of single-shot done on multishot req.
- **Per-`import_fixed*` WARN_ON_ONCE on !FIXED** — defense against per-driver registered-buf abuse without the flag.
- **Per-SQE128 size derivation from ctx.flags / opcode** — defense against per-truncated SQE copy.
- **Per-`io_alloc_cache_put` ac-recycle** — defense against per-cmd_cache leak.
- **Per-`-EAGAIN` REISSUE flag latch** — defense against per-driver lying about retry; bounded one-shot reissue.
- **Per-`io_submit_flush_completions` after cancel walk** — defense against per-CQE-loss on cancel-all.
- **Per-`IO_URING_F_COMPLETE_DEFER` requires !IO_URING_F_UNLOCKED** — defense against per-defer-without-lock corruption.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-check every `copy_from_user`/`copy_to_user` against `cmd->sqe`, `cmd->pdu`, and per-driver passthrough buffers to slab whitelist; reject objects that cross multi-page slab boundaries.
- **PAX_KERNEXEC** — keep `struct io_uring_cmd_ops` and per-driver `->uring_cmd` vtables in `__ro_after_init`; any rewrite (e.g., late-loaded NVMe/ublk module replacing an op) must be rejected.
- **PAX_RANDKSTACK** — re-randomize kernel stack on every uring-cmd entry to defeat stack-spray against the large `pdu[32]` scratch area.
- **PAX_REFCOUNT** — saturating refcount on `req->file`, `nvme_uring_cmd_pdu`, and `ublk_uring_cmd` so a buggy driver cannot wrap and trigger UAF on completion.
- **PAX_MEMORY_SANITIZE** — zero `cmd->pdu`, `cmd->sqe`, and any per-driver bounce buffer on free; never leak prior `struct nvme_command` bytes to a recycled SQE slot.
- **PAX_UDEREF** — enforce strict user/kernel separation when uring-cmd passes opaque `void __user *` from `sqe->addr` to driver; reject any direct kernel pointer deref.
- **PAX_RAP / kCFI** — type-check every indirect `->uring_cmd`, `->uring_cmd_iopoll`, and per-driver completion callback; mismatch panics rather than executes attacker-controlled gadget.
- **GRKERNSEC_HIDESYM** — hide driver symbol addresses (e.g., `nvme_submit_user_cmd`, `ublk_ch_uring_cmd`) from `/proc/kallsyms` for non-root to blunt passthrough exploit chains.
- **GRKERNSEC_DMESG** — restrict dmesg so per-cmd error spew (driver-leaked SGL, queue depth) cannot be harvested by unprivileged probers.
- **Per-driver CAP gating** — every `->uring_cmd` op MUST re-check the cap the corresponding `ioctl` requires (NVMe passthrough/ublk → `CAP_SYS_RAWIO`); do not trust that io_uring registration laundered the check.
- **NVMe/ublk passthrough hardening** — clamp opcode allowlist per-namespace; reject `IO_URING_F_CQE32`/`F_SQE128` if driver does not opt in; verify metadata-pointer ownership before DMA.
- **Rationale** — `uring-cmd` is a generic passthrough that turns io_uring into an ioctl multiplexer; every grsec primitive that historically hardened `ioctl(2)` MUST apply equally here, plus refcount and CFI guards because completion runs in IRQ/softirq context where exploitation is easier and recovery harder.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/nvme/host/ioctl.c NVMe passthrough (driver-side covered in `drivers/nvme/` Tier-3 if expanded)
- drivers/block/ublk_drv.c ublk (driver-side covered separately if expanded)
- sound/core/compress_offload.c (driver-side covered separately if expanded)
- io_uring/poll.c apoll machinery (covered in `poll.md` Tier-3)
- io_uring/kbuf.c buffer selection (covered in `kbuf.md` Tier-3)
- io_uring/cancel.c cancel-data path (covered in `cancel.md` Tier-3)
- io_uring/rsrc.c registered buffers (covered in `rsrc.md` Tier-3)
- Implementation code
