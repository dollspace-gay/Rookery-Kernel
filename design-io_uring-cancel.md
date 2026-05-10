---
title: "Tier-3: io_uring/cancel.c — io_uring request cancellation"
tags: ["tier-3", "io_uring", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

io_uring cancellation lets userspace tear down submitted-but-not-yet-completed requests by descriptor (user_data), backing fd, opcode, or unconditionally. Two entry points exist: **async** via `IORING_OP_ASYNC_CANCEL` SQEs and **synchronous** via `io_uring_register(IORING_REGISTER_SYNC_CANCEL)`. Per-cancel context: `struct io_cancel_data` (ctx, user_data, file, opcode, flags, cancel_seq). Per-target routing in `io_try_cancel`: io-wq queue → poll → waitid → futex → timeout. Per-iter `cancel_seq` dedup prevents the same request being matched twice in an `IORING_ASYNC_CANCEL_ALL`/`_ANY` walk. Task-exit cancellation (`__io_uring_cancel`, `io_uring_cancel_generic`) iterates every ctx the task touched and drains inflight via `io_uring_try_cancel_requests`. Critical for: graceful shutdown, timeout-driven I/O retraction, exec/exit hygiene, deterministic teardown.

This Tier-3 covers `io_uring/cancel.c` (~667 lines).

### Acceptance Criteria

- [ ] AC-1: SQE with cancel_flags & ~CANCEL_FLAGS ⟹ async_prep returns -EINVAL.
- [ ] AC-2: _FD + _ANY in same SQE ⟹ async_prep returns -EINVAL.
- [ ] AC-3: _OP + _ANY in same SQE ⟹ async_prep returns -EINVAL.
- [ ] AC-4: _OP with opcode >= IORING_OP_LAST ⟹ async_prep returns -EINVAL.
- [ ] AC-5: Default cancel (no _FD/_OP) matches on user_data only.
- [ ] AC-6: _FD without _FD_FIXED ⟹ io_file_get_normal; with _FD_FIXED ⟹ io_file_get_fixed.
- [ ] AC-7: _ALL with multiple matches returns count; non-_ALL returns first match status.
- [ ] AC-8: io_try_cancel routes io-wq → poll → waitid → futex → timeout (in order); -ENOENT falls through.
- [ ] AC-9: Cancellation of running work returns -EALREADY (still falls through to poll-disarm).
- [ ] AC-10: IORING_REGISTER_SYNC_CANCEL pads/pad2 nonzero ⟹ -EINVAL.
- [ ] AC-11: Sync-cancel with timeout 0 / unmatched ⟹ -ETIME after schedule_hrtimeout fires.
- [ ] AC-12: Task exit (__io_uring_cancel(true)): all inflight reqs drained, in_cancel decremented, tctx freed.
- [ ] AC-13: SQPOLL drain: io_uring_cancel_generic(true, sqd) walks sqd.ctx_list (not tctx.xa).
- [ ] AC-14: cancel_seq dedup: same request not matched twice in _ALL/_ANY pass.
- [ ] AC-15: REQ_F_LINK_TIMEOUT match uses timeout_lock to avoid races.

### Architecture

```
struct IoCancel {                 // per-SQE prep state
  file: Option<*File>,
  addr: u64,
  flags: u32,
  fd: i32,
  opcode: u8,
}

struct IoCancelData {             // per-cancel match descriptor
  ctx: *IoRingCtx,
  data: u64,                      // user_data
  file: Option<*File>,
  opcode: u8,
  flags: u32,
  seq: u32,                       // cancel_seq snapshot
}

struct IoTaskCancel {
  tctx: *IoUringTask,
  all: bool,
}

const CANCEL_FLAGS: u32 = IORING_ASYNC_CANCEL_ALL
  | IORING_ASYNC_CANCEL_FD
  | IORING_ASYNC_CANCEL_ANY
  | IORING_ASYNC_CANCEL_FD_FIXED
  | IORING_ASYNC_CANCEL_USERDATA
  | IORING_ASYNC_CANCEL_OP;
```

`Cancel::req_match(req, cd) -> bool`:
1. let mut match_user_data = (cd.flags & USERDATA) != 0.
2. if req.ctx != cd.ctx: return false.
3. if (cd.flags & (FD | OP)) == 0: match_user_data = true.
4. if cd.flags & ANY: goto check_seq.
5. if (cd.flags & FD) ∧ req.file != cd.file: return false.
6. if (cd.flags & OP) ∧ req.opcode != cd.opcode: return false.
7. if match_user_data ∧ req.cqe.user_data != cd.data: return false.
8. if cd.flags & ALL:
   - check_seq: if Cancel::match_sequence(req, cd.seq): return false.
9. return true.

`Cancel::async_one(tctx, cd) -> i32`:
1. if tctx.is_none() ∨ tctx.io_wq.is_none(): return -ENOENT.
2. let all = cd.flags & (ALL | ANY) != 0.
3. match io_wq_cancel_cb(tctx.io_wq, Cancel::wq_match_cb, cd, all):
   - OK ⟹ 0; RUNNING ⟹ -EALREADY; NOTFOUND ⟹ -ENOENT.

`Cancel::try_cancel(tctx, cd, issue_flags) -> i32`:
1. WARN_ON_ONCE(!io_wq_current_is_worker() ∧ tctx != current.io_uring).
2. ret = Cancel::async_one(tctx, cd). if ret == 0: return 0.
3. ret = io_poll_cancel(ctx, cd, issue_flags). if ret != -ENOENT: return ret.
4. ret = io_waitid_cancel(ctx, cd, issue_flags). if ret != -ENOENT: return ret.
5. ret = io_futex_cancel(ctx, cd, issue_flags). if ret != -ENOENT: return ret.
6. /* completion_lock for timeout */
7. spin_lock(&ctx.completion_lock).
8. if (cd.flags & FD) == 0: ret = io_timeout_cancel(ctx, cd).
9. spin_unlock(&ctx.completion_lock).
10. return ret.

`Cancel::async_prep(req, sqe) -> i32`:
1. if req.flags & REQ_F_BUFFER_SELECT: return -EINVAL.
2. if sqe.off ∨ sqe.splice_fd_in: return -EINVAL.
3. cancel.addr = READ_ONCE(sqe.addr).
4. cancel.flags = READ_ONCE(sqe.cancel_flags).
5. if cancel.flags & !CANCEL_FLAGS: return -EINVAL.
6. if cancel.flags & FD:
   - if cancel.flags & ANY: return -EINVAL.
   - cancel.fd = READ_ONCE(sqe.fd).
7. if cancel.flags & OP:
   - if cancel.flags & ANY: return -EINVAL.
   - op = READ_ONCE(sqe.len). if op >= IORING_OP_LAST: return -EINVAL.
   - cancel.opcode = op as u8.
8. return 0.

`Cancel::async_inner(cd, tctx, issue_flags) -> i32`:
1. let all = cd.flags & (ALL | ANY) != 0.
2. let mut nr = 0.
3. loop:
   - ret = Cancel::try_cancel(tctx, cd, issue_flags).
   - if ret == -ENOENT: break.
   - if !all: return ret.
   - nr += 1.
4. /* Slow path */
5. __set_current_state(TASK_RUNNING).
6. io_ring_submit_lock(ctx, issue_flags).
7. mutex_lock(&ctx.tctx_lock).
8. ret = -ENOENT.
9. for node in ctx.tctx_list:
   - r = Cancel::async_one(node.task.io_uring, cd).
   - if r != -ENOENT: { ret = r; if !all: break; nr += 1. }
10. mutex_unlock(&ctx.tctx_lock).
11. io_ring_submit_unlock(ctx, issue_flags).
12. return if all { nr } else { ret }.

`Cancel::async(req, issue_flags) -> i32`:
1. let mut cd = IoCancelData {
     ctx: req.ctx, data: cancel.addr, flags: cancel.flags,
     opcode: cancel.opcode, seq: atomic_inc_return(&ctx.cancel_seq), file: None,
   }.
2. if cd.flags & FD:
   - if req.flags & REQ_F_FIXED_FILE ∨ cd.flags & FD_FIXED:
     - req.flags |= REQ_F_FIXED_FILE.
     - req.file = io_file_get_fixed(req, cancel.fd, issue_flags).
   - else: req.file = io_file_get_normal(req, cancel.fd).
   - if !req.file: ret = -EBADF; goto done.
   - cd.file = req.file.
3. ret = Cancel::async_inner(&cd, req.tctx, issue_flags).
4. done: if ret < 0: req_set_fail(req). io_req_set_res(req, ret, 0). return IOU_COMPLETE.

`Cancel::sync(ctx, arg) -> i32`:
1. /* Holds ctx.uring_lock on entry */
2. copy_from_user(&sc, arg, sizeof(IoUringSyncCancelReg))?.
3. if sc.flags & !CANCEL_FLAGS: return -EINVAL.
4. for pad in sc.pad.iter().chain(sc.pad2.iter()): if pad != 0: return -EINVAL.
5. let mut cd = IoCancelData { ctx, data: sc.addr, flags: sc.flags, opcode: sc.opcode, seq: atomic_inc_return(&ctx.cancel_seq), file: None }.
6. let file = if (cd.flags & FD) ∧ !(cd.flags & FD_FIXED) { fget(sc.fd) } else { None }.
   - if expected ∧ file.is_none(): return -EBADF.
   - cd.file = file.
7. ret = Cancel::sync_inner(current.io_uring, &cd, sc.fd).
8. if ret != -EALREADY: goto out.
9. /* Timeout wait loop */
10. let timeout = if sc.timeout != (-1UL, -1UL) { ktime_add_ns(...) } else { KTIME_MAX }.
11. loop:
    - cd.seq = atomic_inc_return(&ctx.cancel_seq).
    - prepare_to_wait(&ctx.cq_wait, &wait, TASK_INTERRUPTIBLE).
    - ret = Cancel::sync_inner(current.io_uring, &cd, sc.fd).
    - mutex_unlock(&ctx.uring_lock).
    - if ret != -EALREADY: break.
    - if io_run_task_work_sig(ctx) < 0: break.
    - if schedule_hrtimeout(&timeout, HRTIMER_MODE_ABS) == 0: ret = -ETIME; break.
    - mutex_lock(&ctx.uring_lock).
12. finish_wait(&ctx.cq_wait, &wait). mutex_lock(&ctx.uring_lock).
13. if ret == -ENOENT ∨ ret > 0: ret = 0.
14. out: if let Some(f) = file: fput(f). return ret.

`Cancel::generic_drain(cancel_all, sqd)`:
1. let tctx = current.io_uring. if tctx.is_none(): return.
2. if let Some(wq) = tctx.io_wq: io_wq_exit_start(wq).
3. atomic_inc(&tctx.in_cancel).
4. loop:
   - io_uring_drop_tctx_refs(current).
   - if !Cancel::tctx_inflight(tctx, !cancel_all): break.
   - let inflight = Cancel::tctx_inflight(tctx, false). if inflight == 0: break.
   - if sqd.is_none():
     - for (_, node) in tctx.xa.iter(): if node.ctx.sq_data.is_none(): loop |= Cancel::try_requests(node.ctx, tctx, cancel_all, false).
   - else:
     - for ctx in sqd.ctx_list: loop |= Cancel::try_requests(ctx, tctx, cancel_all, true).
   - if loop: cond_resched(); continue.
   - prepare_to_wait(&tctx.wait, &wait, TASK_INTERRUPTIBLE).
   - io_run_task_work(). io_uring_drop_tctx_refs(current).
   - if any node.ctx has local_work_pending: goto end_wait.
   - if inflight == Cancel::tctx_inflight(tctx, !cancel_all): schedule().
   - end_wait: finish_wait(&tctx.wait, &wait).
5. io_uring_clean_tctx(tctx).
6. if cancel_all: atomic_dec(&tctx.in_cancel). __io_uring_free(current).

### Out of Scope

- io_uring/poll.c io_poll_cancel implementation (covered in `poll.md` Tier-3)
- io_uring/timeout.c io_timeout_cancel / io_kill_timeouts (covered separately if expanded)
- io_uring/waitid.c io_waitid_cancel (covered separately if expanded)
- io_uring/futex.c io_futex_cancel (covered separately if expanded)
- io_uring/io-wq.c io_wq_cancel_cb internals (covered separately if expanded)
- io_uring/uring_cmd.c io_uring_try_cancel_uring_cmd (covered separately if expanded)
- io_uring/tctx.c io_uring_clean_tctx / __io_uring_free (covered separately if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_cancel` | per-SQE prep state (addr, flags, fd, opcode) | `IoCancel` |
| `struct io_cancel_data` | per-cancel match descriptor | `IoCancelData` |
| `struct io_task_cancel` | per-task-cancel pair (tctx, all) | `IoTaskCancel` |
| `io_cancel_req_match()` | per-request match predicate | `Cancel::req_match` |
| `io_cancel_cb()` | per-io-wq match callback | `Cancel::wq_match_cb` |
| `io_async_cancel_one()` | per-io-wq cancel-one | `Cancel::async_one` |
| `io_try_cancel()` | per-subsystem routing | `Cancel::try_cancel` |
| `io_async_cancel_prep()` | per-SQE validation | `Cancel::async_prep` |
| `__io_async_cancel()` | per-loop + slow-path tctx_list walk | `Cancel::async_inner` |
| `io_async_cancel()` | per-OP issue | `Cancel::async` |
| `__io_sync_cancel()` | per-sync-call core | `Cancel::sync_inner` |
| `io_sync_cancel()` | per-IORING_REGISTER_SYNC_CANCEL | `Cancel::sync` |
| `io_cancel_remove_all()` | per-hlist remove-all | `Cancel::remove_all_in_list` |
| `io_cancel_remove()` | per-hlist match-and-remove | `Cancel::remove_in_list` |
| `io_match_linked()` | per-link-chain REQ_F_INFLIGHT scan | `Cancel::match_linked` |
| `io_match_task_safe()` | per-task match with linked-timeout race guard | `Cancel::match_task_safe` |
| `__io_uring_cancel()` | per-exit/exec entry | `Cancel::uring_cancel` |
| `io_cancel_task_cb()` | per-io-wq task-match callback | `Cancel::task_match_cb` |
| `io_cancel_defer_files()` | per-defer-list reaper | `Cancel::defer_files` |
| `io_cancel_ctx_cb()` | per-ctx io-wq match | `Cancel::ctx_match_cb` |
| `io_uring_try_cancel_iowq()` | per-ctx all-tctx io-wq walk | `Cancel::try_iowq` |
| `io_uring_try_cancel_requests()` | per-ctx full subsystem sweep | `Cancel::try_requests` |
| `tctx_inflight()` | per-tctx inflight count | `Cancel::tctx_inflight` |
| `io_uring_cancel_generic()` | per-task drain loop | `Cancel::generic_drain` |
| `io_cancel_match_sequence()` | per-cancel_seq dedup helper | `Cancel::match_sequence` |
| `IORING_ASYNC_CANCEL_*` flags | per-UAPI flag set | shared UAPI |
| `CANCEL_FLAGS` (mask) | per-validated flag union | `CANCEL_FLAGS` const |
| `struct io_uring_sync_cancel_reg` | per-UAPI sync-cancel arg | `IoUringSyncCancelReg` |

### compatibility contract

REQ-1: struct io_cancel (per-SQE):
- file: per-fd-cancel target file pointer (set by io_async_cancel during issue).
- addr: per-user_data value to match (from sqe->addr).
- flags: per-IORING_ASYNC_CANCEL_* bitmask.
- fd: per-IORING_ASYNC_CANCEL_FD raw / fixed fd index (from sqe->fd).
- opcode: per-IORING_ASYNC_CANCEL_OP target opcode (from sqe->len).

REQ-2: struct io_cancel_data (per-cancel match descriptor):
- ctx: per-ring io_ring_ctx (matches must share ctx).
- data: per-user_data target.
- file: per-fd-cancel target struct file.
- opcode: per-opcode target (u8).
- flags: per-IORING_ASYNC_CANCEL_* bitmask.
- seq: per-cancel cancel_seq (atomic_inc_return on ctx->cancel_seq).

REQ-3: CANCEL_FLAGS mask (per-validation):
- IORING_ASYNC_CANCEL_ALL | _FD | _ANY | _FD_FIXED | _USERDATA | _OP.
- Any other bit in cancel->flags ⟹ -EINVAL.

REQ-4: io_cancel_req_match(req, cd) — per-request predicate:
- match_user_data = cd.flags & IORING_ASYNC_CANCEL_USERDATA.
- /* Per-ring scoping */
- if req.ctx != cd.ctx: return false.
- /* Implicit-user_data: if neither _FD nor _OP specified, match on user_data */
- if !(cd.flags & (IORING_ASYNC_CANCEL_FD | IORING_ASYNC_CANCEL_OP)): match_user_data = true.
- /* Per-_ANY: jump to seq dedup */
- if cd.flags & IORING_ASYNC_CANCEL_ANY: goto check_seq.
- /* Per-_FD */
- if cd.flags & IORING_ASYNC_CANCEL_FD ∧ req.file != cd.file: return false.
- /* Per-_OP */
- if cd.flags & IORING_ASYNC_CANCEL_OP ∧ req.opcode != cd.opcode: return false.
- /* Per-user_data */
- if match_user_data ∧ req.cqe.user_data != cd.data: return false.
- /* Per-_ALL: cancel_seq dedup */
- if cd.flags & IORING_ASYNC_CANCEL_ALL:
  - check_seq: if io_cancel_match_sequence(req, cd.seq): return false.
- return true.

REQ-5: io_async_cancel_one(tctx, cd) — per-io-wq attempt:
- if !tctx ∨ !tctx.io_wq: return -ENOENT.
- all = cd.flags & (IORING_ASYNC_CANCEL_ALL | IORING_ASYNC_CANCEL_ANY).
- cancel_ret = io_wq_cancel_cb(tctx.io_wq, io_cancel_cb, cd, all).
- Map: IO_WQ_CANCEL_OK ⟹ 0; IO_WQ_CANCEL_RUNNING ⟹ -EALREADY; IO_WQ_CANCEL_NOTFOUND ⟹ -ENOENT.

REQ-6: io_try_cancel(tctx, cd, issue_flags) — per-subsystem routing:
- WARN_ON_ONCE(!io_wq_current_is_worker() ∧ tctx != current.io_uring).
- /* 1. Try io-wq queue */
- ret = io_async_cancel_one(tctx, cd).
- if ret == 0: return 0.
- /* Even -EALREADY falls through: poll arming may need un-arm */
- /* 2. Poll subsystem */
- ret = io_poll_cancel(ctx, cd, issue_flags). if ret != -ENOENT: return ret.
- /* 3. waitid */
- ret = io_waitid_cancel(ctx, cd, issue_flags). if ret != -ENOENT: return ret.
- /* 4. futex */
- ret = io_futex_cancel(ctx, cd, issue_flags). if ret != -ENOENT: return ret.
- /* 5. timeout (under completion_lock; skipped for _FD) */
- spin_lock(&ctx.completion_lock).
- if !(cd.flags & IORING_ASYNC_CANCEL_FD): ret = io_timeout_cancel(ctx, cd).
- spin_unlock(&ctx.completion_lock).
- return ret.

REQ-7: io_async_cancel_prep(req, sqe) — per-SQE validation:
- if req.flags & REQ_F_BUFFER_SELECT: return -EINVAL.
- if sqe.off ∨ sqe.splice_fd_in: return -EINVAL.
- cancel.addr = READ_ONCE(sqe.addr).
- cancel.flags = READ_ONCE(sqe.cancel_flags).
- if cancel.flags & ~CANCEL_FLAGS: return -EINVAL.
- if cancel.flags & IORING_ASYNC_CANCEL_FD:
  - if cancel.flags & IORING_ASYNC_CANCEL_ANY: return -EINVAL (_FD + _ANY illegal).
  - cancel.fd = READ_ONCE(sqe.fd).
- if cancel.flags & IORING_ASYNC_CANCEL_OP:
  - if cancel.flags & IORING_ASYNC_CANCEL_ANY: return -EINVAL (_OP + _ANY illegal).
  - op = READ_ONCE(sqe.len). if op >= IORING_OP_LAST: return -EINVAL.
  - cancel.opcode = op.
- return 0.

REQ-8: __io_async_cancel(cd, tctx, issue_flags) — per-loop + slow-path:
- all = cd.flags & (IORING_ASYNC_CANCEL_ALL | IORING_ASYNC_CANCEL_ANY).
- /* Fast loop on current tctx */
- nr = 0.
- do:
  - ret = io_try_cancel(tctx, cd, issue_flags).
  - if ret == -ENOENT: break.
  - if !all: return ret.
  - nr++.
- while (1).
- /* Slow path: walk ctx.tctx_list under tctx_lock + submit_lock */
- __set_current_state(TASK_RUNNING).
- io_ring_submit_lock(ctx, issue_flags).
- mutex_lock(&ctx.tctx_lock).
- ret = -ENOENT.
- for node in ctx.tctx_list:
  - r = io_async_cancel_one(node.task.io_uring, cd).
  - if r != -ENOENT: { ret = r; if !all: break; nr++. }
- mutex_unlock(&ctx.tctx_lock).
- io_ring_submit_unlock(ctx, issue_flags).
- return all ? nr : ret.

REQ-9: io_async_cancel(req, issue_flags) — IORING_OP_ASYNC_CANCEL issue:
- Build io_cancel_data: ctx, data=cancel.addr, flags=cancel.flags, opcode=cancel.opcode,
  seq = atomic_inc_return(&ctx.cancel_seq).
- if cd.flags & IORING_ASYNC_CANCEL_FD:
  - if req.flags & REQ_F_FIXED_FILE ∨ cd.flags & IORING_ASYNC_CANCEL_FD_FIXED:
    - req.flags |= REQ_F_FIXED_FILE.
    - req.file = io_file_get_fixed(req, cancel.fd, issue_flags).
  - else: req.file = io_file_get_normal(req, cancel.fd).
  - if !req.file: ret = -EBADF; goto done.
  - cd.file = req.file.
- ret = __io_async_cancel(&cd, req.tctx, issue_flags).
- done: if ret < 0: req_set_fail(req). io_req_set_res(req, ret, 0). return IOU_COMPLETE.

REQ-10: __io_sync_cancel(tctx, cd, fd) — per-sync core:
- /* Re-grab fixed file each iteration (uring_lock is dropped between waits) */
- if (cd.flags & IORING_ASYNC_CANCEL_FD) ∧ (cd.flags & IORING_ASYNC_CANCEL_FD_FIXED):
  - node = io_rsrc_node_lookup(&ctx.file_table.data, fd). if !node: return -EBADF.
  - cd.file = io_slot_file(node). if !cd.file: return -EBADF.
- return __io_async_cancel(cd, tctx, 0).

REQ-11: io_sync_cancel(ctx, arg) — IORING_REGISTER_SYNC_CANCEL entry:
- __must_hold(&ctx.uring_lock).
- copy_from_user(&sc, arg, sizeof(io_uring_sync_cancel_reg)). on EFAULT: return -EFAULT.
- if sc.flags & ~CANCEL_FLAGS: return -EINVAL.
- for pad ∈ sc.pad, pad2: if pad != 0: return -EINVAL.
- Initialize cd { ctx, data=sc.addr, flags=sc.flags, opcode=sc.opcode, seq=atomic_inc_return(&ctx.cancel_seq) }.
- /* Normal-fd path: grab once up front */
- if (cd.flags & IORING_ASYNC_CANCEL_FD) ∧ !(cd.flags & IORING_ASYNC_CANCEL_FD_FIXED):
  - file = fget(sc.fd). if !file: return -EBADF. cd.file = file.
- ret = __io_sync_cancel(current.io_uring, &cd, sc.fd).
- if ret != -EALREADY: goto out.
- /* Loop-with-timeout until -ENOENT */
- timeout = (sc.timeout.tv_sec != -1UL ∨ sc.timeout.tv_nsec != -1UL) ? ktime_add_ns(ts, ktime_get_ns()) : KTIME_MAX.
- do:
  - cd.seq = atomic_inc_return(&ctx.cancel_seq).
  - prepare_to_wait(&ctx.cq_wait, &wait, TASK_INTERRUPTIBLE).
  - ret = __io_sync_cancel(current.io_uring, &cd, sc.fd).
  - mutex_unlock(&ctx.uring_lock).
  - if ret != -EALREADY: break.
  - ret = io_run_task_work_sig(ctx). if ret < 0: break.
  - ret = schedule_hrtimeout(&timeout, HRTIMER_MODE_ABS). if !ret: ret = -ETIME; break.
  - mutex_lock(&ctx.uring_lock).
- while (1).
- finish_wait(&ctx.cq_wait, &wait). mutex_lock(&ctx.uring_lock).
- if ret == -ENOENT ∨ ret > 0: ret = 0.
- out: if file: fput(file). return ret.

REQ-12: io_cancel_remove_all(ctx, tctx, list, cancel_all, cancel) — per-hlist:
- lockdep_assert_held(&ctx.uring_lock).
- For each req in list:
  - if !io_match_task_safe(req, tctx, cancel_all): continue.
  - hlist_del_init(&req.hash_node).
  - if cancel(req): found = true.
- return found.

REQ-13: io_cancel_remove(ctx, cd, issue_flags, list, cancel) — per-cd hlist:
- io_ring_submit_lock(ctx, issue_flags).
- For each req in list:
  - if !io_cancel_req_match(req, cd): continue.
  - if cancel(req): nr++.
  - if !(cd.flags & IORING_ASYNC_CANCEL_ALL): break.
- io_ring_submit_unlock(ctx, issue_flags).
- return nr ?: -ENOENT.

REQ-14: io_match_task_safe(head, tctx, cancel_all) — per-link race-guarded predicate:
- if tctx ∧ head.tctx != tctx: return false.
- if cancel_all: return true.
- if head.flags & REQ_F_LINK_TIMEOUT:
  - raw_spin_lock_irq(&ctx.timeout_lock).
  - matched = io_match_linked(head).
  - raw_spin_unlock_irq(&ctx.timeout_lock).
- else: matched = io_match_linked(head).
- return matched.

REQ-15: __io_uring_cancel(cancel_all) — task exit/exec entry:
- io_uring_unreg_ringfd().
- io_uring_cancel_generic(cancel_all, NULL).

REQ-16: io_uring_try_cancel_requests(ctx, tctx, cancel_all, is_sqpoll_thread):
- /* DEFER_TASKRUN wakeup arm */
- if ctx.flags & IORING_SETUP_DEFER_TASKRUN: atomic_set(&ctx.cq_wait_nr, 1); smp_mb().
- if !ctx.rings: return false (init failure, no requests).
- /* io-wq sweep: per-ctx if !tctx, per-tctx if tctx.io_wq present */
- if !tctx: ret |= io_uring_try_cancel_iowq(ctx).
- else if tctx.io_wq: ret |= io_wq_cancel_cb(tctx.io_wq, io_cancel_task_cb, &{tctx, cancel_all}, true) != IO_WQ_CANCEL_NOTFOUND.
- /* iopoll reap (SQPOLL handles its own otherwise) */
- if (!(ctx.flags & IORING_SETUP_SQPOLL) ∧ cancel_all) ∨ is_sqpoll_thread:
  - while !list_empty(&ctx.iopoll_list): io_iopoll_try_reap_events(ctx); ret = true; cond_resched().
- /* DEFER_TASKRUN local work flush */
- if (ctx.flags & IORING_SETUP_DEFER_TASKRUN) ∧ io_allowed_defer_tw_run(ctx):
  - ret |= io_run_local_work(ctx, INT_MAX, INT_MAX) > 0.
- /* Under uring_lock: deferred, poll, waitid, futex, uring_cmd */
- mutex_lock(&ctx.uring_lock).
- ret |= io_cancel_defer_files(ctx, tctx, cancel_all).
- ret |= io_poll_remove_all(ctx, tctx, cancel_all).
- ret |= io_waitid_remove_all(ctx, tctx, cancel_all).
- ret |= io_futex_remove_all(ctx, tctx, cancel_all).
- ret |= io_uring_try_cancel_uring_cmd(ctx, tctx, cancel_all).
- mutex_unlock(&ctx.uring_lock).
- ret |= io_kill_timeouts(ctx, tctx, cancel_all).
- if tctx: ret |= io_run_task_work() > 0.
- else: ret |= flush_delayed_work(&ctx.fallback_work).
- return ret.

REQ-17: io_uring_cancel_generic(cancel_all, sqd) — per-task drain loop:
- tctx = current.io_uring. if !tctx: return.
- WARN_ON_ONCE(sqd ∧ sqpoll_task_locked(sqd) != current).
- if tctx.io_wq: io_wq_exit_start(tctx.io_wq).
- atomic_inc(&tctx.in_cancel).
- do:
  - io_uring_drop_tctx_refs(current).
  - if !tctx_inflight(tctx, !cancel_all): break.
  - inflight = tctx_inflight(tctx, false). if !inflight: break.
  - if !sqd: for node ∈ tctx.xa: if !node.ctx.sq_data: loop |= io_uring_try_cancel_requests(node.ctx, tctx, cancel_all, false).
  - else: for ctx ∈ sqd.ctx_list: loop |= io_uring_try_cancel_requests(ctx, tctx, cancel_all, true).
  - if loop: cond_resched(); continue.
  - prepare_to_wait(&tctx.wait, &wait, TASK_INTERRUPTIBLE).
  - io_run_task_work(). io_uring_drop_tctx_refs(current).
  - for node ∈ tctx.xa: if io_local_work_pending(node.ctx): goto end_wait.
  - if inflight == tctx_inflight(tctx, !cancel_all): schedule().
  - end_wait: finish_wait(&tctx.wait, &wait).
- while (1).
- io_uring_clean_tctx(tctx).
- if cancel_all: atomic_dec(&tctx.in_cancel). __io_uring_free(current).

REQ-18: Per-cancel_seq dedup:
- Each cancel iteration advances ctx.cancel_seq via atomic_inc_return.
- io_cancel_match_sequence(req, seq): if req.work.cancel_seq == seq, request already considered; skip.
- Ensures an _ALL or _ANY walk visits each request at most once.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cancel_flags_validated` | INVARIANT | per-async_prep: cancel.flags ⊆ CANCEL_FLAGS or -EINVAL. |
| `fd_and_any_exclusive` | INVARIANT | per-async_prep: (FD ∧ ANY) ⟹ -EINVAL. |
| `op_and_any_exclusive` | INVARIANT | per-async_prep: (OP ∧ ANY) ⟹ -EINVAL. |
| `opcode_in_range` | INVARIANT | per-async_prep: opcode < IORING_OP_LAST. |
| `try_cancel_routing_order` | INVARIANT | per-try_cancel: io-wq → poll → waitid → futex → timeout. |
| `timeout_under_completion_lock` | INVARIANT | per-try_cancel: io_timeout_cancel called with completion_lock held. |
| `cancel_seq_monotonic` | INVARIANT | per-issue: cd.seq strictly increasing via atomic_inc_return. |
| `link_timeout_uses_timeout_lock` | INVARIANT | per-match_task_safe: REQ_F_LINK_TIMEOUT ⟹ timeout_lock held during match. |
| `in_cancel_paired` | INVARIANT | per-generic_drain: atomic_inc/dec on in_cancel paired except cancel_all path. |
| `file_refcount_balanced` | INVARIANT | per-sync: fget/fput balanced; -EBADF short-circuits before fput. |

### Layer 2: TLA+

`io_uring/cancel.tla`:
- Per-request lifecycle × per-cancel-flag × per-subsystem routing.
- Properties:
  - `safety_cancel_seq_dedup` — per-_ALL: req never matched twice in same pass.
  - `safety_routing_monotone` — per-try_cancel: stops at first non-ENOENT (modulo io-wq fall-through for poll-disarm).
  - `safety_no_cross_ctx_match` — per-req_match: req.ctx == cd.ctx required.
  - `safety_fd_any_mutex` — per-prep: FD ∧ ANY rejected.
  - `safety_sync_cancel_eventually_terminates` — per-sync: returns 0 ∨ -ETIME ∨ -ENOENT in finite time.
  - `liveness_generic_drain_terminates` — per-task-exit: tctx_inflight reaches 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cancel::async_prep` post: ret 0 ⟹ flags ⊆ CANCEL_FLAGS ∧ no exclusive-pair violation | `Cancel::async_prep` |
| `Cancel::req_match` post: false unless ctx match ∧ flag-specific predicates pass | `Cancel::req_match` |
| `Cancel::try_cancel` post: returns -ENOENT only if none of {io-wq, poll, waitid, futex, timeout} matched | `Cancel::try_cancel` |
| `Cancel::async_inner` post (all): returns nr ≥ 0; otherwise per-cancel ret | `Cancel::async_inner` |
| `Cancel::sync` post: ret == 0 ∨ -EINVAL ∨ -EBADF ∨ -EFAULT ∨ -ETIME ∨ -EALREADY ∨ -ENOENT | `Cancel::sync` |
| `Cancel::generic_drain` post: tctx.in_cancel decremented in cancel_all mode | `Cancel::generic_drain` |
| `Cancel::try_requests` post: returns true ⟹ ≥1 subsystem matched | `Cancel::try_requests` |

### Layer 4: Verus/Creusot functional

`Per-async-cancel SQE → __io_async_cancel → io_try_cancel (io-wq → poll → waitid → futex → timeout) → CQE; Per-IORING_REGISTER_SYNC_CANCEL → loop until -ENOENT or -ETIME; Per-task-exit __io_uring_cancel → io_uring_cancel_generic → drain inflight` semantic equivalence: per-include/uapi/linux/io_uring.h and io_uring(7) manpage cancel semantics.

### hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

Cancel reinforcement:

- **Per-CANCEL_FLAGS strict validation** — defense against per-undefined-flag injection on future kernel/library versions.
- **Per-_FD/_OP-with-_ANY rejection at prep** — defense against per-ambiguous-target semantic confusion.
- **Per-opcode range check (< IORING_OP_LAST)** — defense against per-op-id OOB.
- **Per-cancel_seq atomic dedup** — defense against per-_ALL infinite loop / double-cancel on the same req.
- **Per-req.ctx == cd.ctx mandatory** — defense against per-cross-ring cancellation.
- **Per-completion_lock around io_timeout_cancel** — defense against per-timeout-race in cancel path.
- **Per-timeout_lock for REQ_F_LINK_TIMEOUT match** — defense against per-link-chain mutation during cancel match.
- **Per-tctx_lock + submit_lock for tctx_list walk** — defense against per-tctx mutation during slow path.
- **Per-fget/fput balanced; -EBADF short-circuits** — defense against per-fd refcount leak.
- **Per-sync-cancel pad/pad2 zero-check** — defense against per-uapi-extension UB / silent flag injection.
- **Per-fixed-file re-grab each retry (uring_lock dropped)** — defense against per-stale-file UAF.
- **Per-IO_WQ_CANCEL_RUNNING fall-through to poll-disarm** — defense against per-poll-armed-running zombie.
- **Per-in_cancel atomic counter** — defense against per-task-work re-entering cancel after exec/exit.

