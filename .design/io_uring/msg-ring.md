# Tier-3: io_uring/msg_ring.c — IORING_OP_MSG_RING inter-ring messaging

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/msg_ring.c (~338 lines)
  - io_uring/msg_ring.h
  - include/uapi/linux/io_uring.h (IORING_OP_MSG_RING, IORING_MSG_DATA, IORING_MSG_SEND_FD, IORING_MSG_RING_CQE_SKIP, IORING_MSG_RING_FLAGS_PASS)
-->

## Summary

**`IORING_OP_MSG_RING`** is the io_uring opcode that lets one ring deliver a message (a CQE payload, or a file descriptor) into another ring atomically. Per-message kinds: `IORING_MSG_DATA` (post a CQE with `user_data` + `len` + optional `cqe_flags`) and `IORING_MSG_SEND_FD` (install a `struct file` into the target ring's fixed-file table at `dst_fd`, then post a CQE). Per-target-locking the source ring uses `mutex_trylock` on the target's `uring_lock` to avoid AB-BA ordering; if it fails ∧ the source already holds its own lock the op is punted to `io-wq`. Per-IO_RING_F_TASK_COMPLETE rings (the target is task-local) data delivery uses a fresh `io_kiocb` + `io_req_task_work_add_remote`; FD delivery uses `task_work_add` on `ctx->submitter_task`. Per-source `IORING_MSG_RING_CQE_SKIP` flag suppresses the sender-side CQE for "fire-and-forget" semantics. Per-`io_uring_sync_msg_ring()` synchronous variant (no source ring needed) is exposed to `io_uring_enter()` for direct DATA-only delivery. Critical for: per-worker fan-out, NIC RX-distribution, and zero-syscall fd hand-off.

This Tier-3 covers `io_uring/msg_ring.c` (~338 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_msg` | per-op state | `IoMsg` |
| `io_msg_ring_prep()` | per-SQE prep | `MsgRing::prep` |
| `io_msg_ring()` | per-issue dispatch | `MsgRing::issue` |
| `io_msg_ring_cleanup()` | per-failure cleanup (fput src_file) | `MsgRing::cleanup` |
| `io_uring_sync_msg_ring()` | per-sync-DATA path | `MsgRing::sync` |
| `__io_msg_ring_prep()` | per-prep internal | `MsgRing::prep_inner` |
| `io_msg_ring_data()` / `__io_msg_ring_data()` | per-DATA delivery | `MsgRing::data` / `data_inner` |
| `io_msg_send_fd()` | per-FD delivery dispatch | `MsgRing::send_fd` |
| `io_msg_install_complete()` | per-FD install + complete | `MsgRing::install_complete` |
| `io_msg_grab_file()` | per-source-fd lookup + get_file | `MsgRing::grab_file` |
| `io_msg_data_remote()` | per-remote-DATA queue | `MsgRing::data_remote` |
| `io_msg_fd_remote()` | per-remote-FD queue | `MsgRing::fd_remote` |
| `io_msg_remote_post()` | per-remote-CQE prepare | `MsgRing::remote_post` |
| `io_msg_tw_complete()` | per-remote-DATA tw callback | `MsgRing::tw_complete` |
| `io_msg_tw_fd_complete()` | per-remote-FD tw callback | `MsgRing::tw_fd_complete` |
| `io_msg_need_remote()` | per-target task-complete test | `MsgRing::need_remote` |
| `io_lock_external_ctx()` | per-target uring_lock acquire | `MsgRing::lock_external` |
| `io_double_unlock_ctx()` | per-target uring_lock release | `MsgRing::double_unlock` |

## Compatibility contract

REQ-1: struct io_msg (per-cmd state in io_kiocb):
- file: per-req target ring file (req->file).
- src_file: per-source file (set when SEND_FD; refcounted via get_file).
- tw: per-task_work callback_head (used by io_msg_tw_fd_complete).
- user_data: per-CQE user_data field (from sqe.off).
- len: per-CQE res field (from sqe.len).
- cmd: per-IORING_MSG_* discriminant (from sqe.addr).
- src_fd: per-source fixed-file index (from sqe.addr3).
- dst_fd: per-target fixed-file slot (from sqe.file_index).
- cqe_flags: union with dst_fd; per-IORING_MSG_RING_FLAGS_PASS cflags (from sqe.file_index).
- flags: per-IORING_MSG_RING_MASK validated (sqe.msg_ring_flags).

REQ-2: IORING_MSG_RING_MASK:
- = IORING_MSG_RING_CQE_SKIP | IORING_MSG_RING_FLAGS_PASS.
- Any other bit in msg.flags ⟹ -EINVAL.

REQ-3: io_msg_ring_prep(req, sqe):
- Reject sqe.buf_index ∨ sqe.personality → -EINVAL.
- msg.src_file = NULL.
- msg.user_data = sqe.off.
- msg.len = sqe.len.
- msg.cmd = sqe.addr.
- msg.src_fd = sqe.addr3.
- msg.dst_fd = sqe.file_index (aliases cqe_flags).
- msg.flags = sqe.msg_ring_flags.
- Reject msg.flags & ~IORING_MSG_RING_MASK → -EINVAL.

REQ-4: io_msg_ring(req, issue_flags):
- Default ret = -EBADFD.
- If !io_is_uring_fops(req.file): goto done.
- Dispatch msg.cmd:
  - IORING_MSG_DATA → io_msg_ring_data(req, issue_flags).
  - IORING_MSG_SEND_FD → io_msg_send_fd(req, issue_flags).
  - default → -EINVAL.
- done:
  - If ret < 0 ∧ ret != -EAGAIN ∧ ret != IOU_ISSUE_SKIP_COMPLETE:
    - req_set_fail(req).
  - io_req_set_res(req, ret, 0).
  - return IOU_COMPLETE.
- -EAGAIN / IOU_ISSUE_SKIP_COMPLETE return as-is (re-queue / async).

REQ-5: __io_msg_ring_data(target_ctx, msg, issue_flags) — DATA path:
- Reject msg.src_fd != 0 ∨ msg.flags & ~IORING_MSG_RING_FLAGS_PASS → -EINVAL.
- Reject !(flags & FLAGS_PASS) ∧ msg.dst_fd != 0 → -EINVAL  (dst_fd / cqe_flags is the union — must be 0 when no FLAGS_PASS).
- /* IORING_SETUP_R_DISABLED ordering */
- smp_load_acquire(target_ctx.flags) & IORING_SETUP_R_DISABLED ⟹ -EBADFD.
- If io_msg_need_remote(target_ctx): return io_msg_data_remote(target_ctx, msg).
- flags = msg.flags & FLAGS_PASS ? msg.cqe_flags : 0.
- ret = -EOVERFLOW.
- /* SETUP_IOPOLL needs target lock */
- If target_ctx.flags & IORING_SETUP_IOPOLL:
  - if io_lock_external_ctx(target_ctx, issue_flags) → -EAGAIN.
- if io_post_aux_cqe(target_ctx, msg.user_data, msg.len, flags): ret = 0.
- If IORING_SETUP_IOPOLL: io_double_unlock_ctx(target_ctx).
- return ret.

REQ-6: io_msg_ring_data(req, issue_flags):
- target_ctx = req.file.private_data (target io_ring_ctx).
- msg = io_kiocb_to_cmd(req, struct io_msg).
- return __io_msg_ring_data(target_ctx, msg, issue_flags).

REQ-7: io_msg_send_fd(req, issue_flags):
- target_ctx = req.file.private_data.
- ctx = req.ctx.
- Reject msg.len != 0 → -EINVAL.
- Reject target_ctx == ctx → -EINVAL  (no self-send).
- smp_load_acquire(target_ctx.flags) & IORING_SETUP_R_DISABLED ⟹ -EBADFD.
- If msg.src_file is NULL: io_msg_grab_file(req, issue_flags); on error return.
- If io_msg_need_remote(target_ctx): return io_msg_fd_remote(req).
- Else: return io_msg_install_complete(req, issue_flags).

REQ-8: io_msg_grab_file(req, issue_flags):
- ctx = req.ctx.
- io_ring_submit_lock(ctx, issue_flags).
- node = io_rsrc_node_lookup(&ctx.file_table.data, msg.src_fd).
- If node:
  - msg.src_file = io_slot_file(node).
  - if msg.src_file: get_file(msg.src_file).
  - req.flags |= REQ_F_NEED_CLEANUP.
  - ret = 0.
- Else ret = -EBADF.
- io_ring_submit_unlock(ctx, issue_flags).

REQ-9: io_msg_install_complete(req, issue_flags):
- io_lock_external_ctx(target_ctx, issue_flags); on -EAGAIN return.
- ret = __io_fixed_fd_install(target_ctx, msg.src_file, msg.dst_fd).
- If ret >= 0: msg.src_file = NULL; req.flags &= ~REQ_F_NEED_CLEANUP.
- If msg.flags & IORING_MSG_RING_CQE_SKIP: goto out_unlock (no target CQE).
- Else if !io_post_aux_cqe(target_ctx, msg.user_data, ret, 0): ret = -EOVERFLOW.
- out_unlock: io_double_unlock_ctx(target_ctx); return ret.

REQ-10: io_msg_need_remote(target_ctx):
- = target_ctx.int_flags & IO_RING_F_TASK_COMPLETE.
- IO_RING_F_TASK_COMPLETE is set when the target ring requires its submitter task to drive completion.

REQ-11: io_msg_data_remote(target_ctx, msg):
- Allocate target io_kiocb from req_cachep (GFP_KERNEL | __GFP_NOWARN | __GFP_ZERO).
- -ENOMEM on alloc failure.
- flags = msg.flags & FLAGS_PASS ? msg.cqe_flags : 0.
- io_msg_remote_post(target_ctx, target, msg.len, flags, msg.user_data).
- return 0.

REQ-12: io_msg_remote_post(ctx, req, res, cflags, user_data):
- req.opcode = IORING_OP_NOP.
- req.cqe.user_data = user_data.
- io_req_set_res(req, res, cflags).
- percpu_ref_get(&ctx.refs).
- req.ctx = ctx; req.tctx = NULL.
- req.io_task_work.func = io_msg_tw_complete.
- io_req_task_work_add_remote(req, IOU_F_TWQ_LAZY_WAKE).

REQ-13: io_msg_tw_complete(tw_req, tw):
- ctx = req.ctx.
- io_add_aux_cqe(ctx, req.cqe.user_data, req.cqe.res, req.cqe.flags).
- kfree_rcu(req, rcu_head).
- percpu_ref_put(&ctx.refs).

REQ-14: io_msg_fd_remote(req):
- target ctx = req.file.private_data.
- task = target_ctx.submitter_task.
- init_task_work(&msg.tw, io_msg_tw_fd_complete).
- if task_work_add(task, &msg.tw, TWA_SIGNAL) fails → -EOWNERDEAD.
- return IOU_ISSUE_SKIP_COMPLETE (req completes from task_work).

REQ-15: io_msg_tw_fd_complete(head):
- msg = container_of(head, struct io_msg, tw).
- req = cmd_to_io_kiocb(msg).
- ret = -EOWNERDEAD.
- If !(current.flags & PF_EXITING): ret = io_msg_install_complete(req, IO_URING_F_UNLOCKED).
- if ret < 0: req_set_fail(req).
- io_req_queue_tw_complete(req, ret).

REQ-16: io_lock_external_ctx(octx, issue_flags) (target-uring_lock acquire):
- If !(issue_flags & IO_URING_F_UNLOCKED) (source ctx is locked):
  - mutex_trylock(octx.uring_lock); on failure return -EAGAIN  (op punted to io-wq with UNLOCKED set).
  - return 0.
- Else (UNLOCKED): mutex_lock(octx.uring_lock); return 0.

REQ-17: io_msg_ring_cleanup(req):
- WARN if !msg.src_file.
- fput(msg.src_file).
- msg.src_file = NULL.
- Called when REQ_F_NEED_CLEANUP set ∧ request fails before src_file is moved to target.

REQ-18: io_uring_sync_msg_ring(sqe) — direct ENTER syscall variant:
- io_msg = {} on stack.
- __io_msg_ring_prep(&io_msg, sqe).
- if io_msg.cmd != IORING_MSG_DATA → -EINVAL  (no source-ring → no SEND_FD).
- CLASS(fd, f)(sqe.fd) — get target ring fd.
- if fd_empty(f) → -EBADF.
- if !io_is_uring_fops(fd_file(f)) → -EBADFD.
- return __io_msg_ring_data(fd_file(f).private_data, &io_msg, IO_URING_F_UNLOCKED).

## Acceptance Criteria

- [ ] AC-1: io_msg_ring_prep rejects sqe.buf_index ∨ sqe.personality.
- [ ] AC-2: io_msg_ring_prep rejects msg.flags & ~(CQE_SKIP | FLAGS_PASS).
- [ ] AC-3: io_msg_ring(req, ...) with non-io_uring target file → -EBADFD.
- [ ] AC-4: IORING_MSG_DATA path delivers a CQE with .user_data = msg.user_data, .res = msg.len, .flags = (msg.cqe_flags if FLAGS_PASS else 0).
- [ ] AC-5: With IORING_SETUP_R_DISABLED on the target → -EBADFD.
- [ ] AC-6: __io_msg_ring_data rejects msg.src_fd != 0.
- [ ] AC-7: __io_msg_ring_data without FLAGS_PASS rejects msg.dst_fd != 0.
- [ ] AC-8: IORING_MSG_SEND_FD on same-ring (target_ctx == ctx) → -EINVAL.
- [ ] AC-9: IORING_MSG_SEND_FD with msg.len != 0 → -EINVAL.
- [ ] AC-10: SEND_FD: src_file fput'd via io_msg_ring_cleanup on failure path.
- [ ] AC-11: SEND_FD: with IORING_MSG_RING_CQE_SKIP on target, no target CQE is posted.
- [ ] AC-12: target IORING_SETUP_IOPOLL → io_msg_ring_data acquires target uring_lock via trylock; -EAGAIN punts to io-wq.
- [ ] AC-13: IO_RING_F_TASK_COMPLETE target: DATA path uses fresh io_kiocb + io_req_task_work_add_remote.
- [ ] AC-14: IO_RING_F_TASK_COMPLETE target: FD path uses task_work_add(target_ctx.submitter_task) with TWA_SIGNAL; returns IOU_ISSUE_SKIP_COMPLETE.
- [ ] AC-15: io_uring_sync_msg_ring rejects IORING_MSG_SEND_FD → -EINVAL.

## Architecture

```
struct IoMsg {
  file: *File,                      // target ring file
  src_file: Option<*File>,
  tw: CallbackHead,
  user_data: u64,
  len: u32,
  cmd: u32,                         // IORING_MSG_*
  src_fd: u32,
  // union {
  dst_fd: u32,
  // cqe_flags: u32,
  // }
  flags: u32,                       // IORING_MSG_RING_*
}

const IORING_MSG_RING_MASK: u32 =
    IORING_MSG_RING_CQE_SKIP | IORING_MSG_RING_FLAGS_PASS;
```

`MsgRing::prep(req, sqe)`:
1. if sqe.buf_index != 0 ∨ sqe.personality != 0: return -EINVAL.
2. msg.src_file = None.
3. msg.user_data = READ_ONCE(sqe.off).
4. msg.len = READ_ONCE(sqe.len).
5. msg.cmd = READ_ONCE(sqe.addr).
6. msg.src_fd = READ_ONCE(sqe.addr3).
7. msg.dst_fd = READ_ONCE(sqe.file_index).
8. msg.flags = READ_ONCE(sqe.msg_ring_flags).
9. if msg.flags & ~IORING_MSG_RING_MASK: return -EINVAL.
10. return 0.

`MsgRing::issue(req, issue_flags)`:
1. ret = -EBADFD.
2. if !io_is_uring_fops(req.file): goto done.
3. match msg.cmd:
   - IORING_MSG_DATA → ret = MsgRing::data(req, issue_flags).
   - IORING_MSG_SEND_FD → ret = MsgRing::send_fd(req, issue_flags).
   - _ → ret = -EINVAL.
4. done:
   - if ret == -EAGAIN ∨ ret == IOU_ISSUE_SKIP_COMPLETE: return ret.
   - if ret < 0: req_set_fail(req).
   - io_req_set_res(req, ret, 0).
   - return IOU_COMPLETE.

`MsgRing::data_inner(target_ctx, msg, issue_flags)`:
1. if msg.src_fd != 0 ∨ msg.flags & ~IORING_MSG_RING_FLAGS_PASS: return -EINVAL.
2. if !(msg.flags & FLAGS_PASS) ∧ msg.dst_fd != 0: return -EINVAL.
3. /* R_DISABLED ordering */
4. if smp_load_acquire(target_ctx.flags) & IORING_SETUP_R_DISABLED: return -EBADFD.
5. if MsgRing::need_remote(target_ctx): return MsgRing::data_remote(target_ctx, msg).
6. flags = if msg.flags & FLAGS_PASS { msg.cqe_flags } else { 0 }.
7. ret = -EOVERFLOW.
8. if target_ctx.flags & IORING_SETUP_IOPOLL:
   - if MsgRing::lock_external(target_ctx, issue_flags) == -EAGAIN: return -EAGAIN.
9. if io_post_aux_cqe(target_ctx, msg.user_data, msg.len, flags): ret = 0.
10. if target_ctx.flags & IORING_SETUP_IOPOLL: io_double_unlock_ctx(target_ctx).
11. return ret.

`MsgRing::send_fd(req, issue_flags)`:
1. target_ctx = req.file.private_data.
2. ctx = req.ctx.
3. if msg.len != 0: return -EINVAL.
4. if target_ctx == ctx: return -EINVAL.
5. if smp_load_acquire(target_ctx.flags) & IORING_SETUP_R_DISABLED: return -EBADFD.
6. if msg.src_file.is_none():
   - ret = MsgRing::grab_file(req, issue_flags).
   - if ret: return ret.
7. if MsgRing::need_remote(target_ctx): return MsgRing::fd_remote(req).
8. return MsgRing::install_complete(req, issue_flags).

`MsgRing::install_complete(req, issue_flags)`:
1. if MsgRing::lock_external(target_ctx, issue_flags) == -EAGAIN: return -EAGAIN.
2. ret = __io_fixed_fd_install(target_ctx, msg.src_file, msg.dst_fd).
3. if ret < 0: goto out_unlock.
4. msg.src_file = None.
5. req.flags &= ~REQ_F_NEED_CLEANUP.
6. if msg.flags & IORING_MSG_RING_CQE_SKIP: goto out_unlock.
7. if !io_post_aux_cqe(target_ctx, msg.user_data, ret, 0): ret = -EOVERFLOW.
8. out_unlock: io_double_unlock_ctx(target_ctx); return ret.

`MsgRing::data_remote(target_ctx, msg)`:
1. target = req_cachep.alloc(GFP_KERNEL | __GFP_NOWARN | __GFP_ZERO).
2. if target.is_null(): return -ENOMEM.
3. flags = if msg.flags & FLAGS_PASS { msg.cqe_flags } else { 0 }.
4. MsgRing::remote_post(target_ctx, target, msg.len, flags, msg.user_data).
5. return 0.

`MsgRing::remote_post(ctx, req, res, cflags, user_data)`:
1. req.opcode = IORING_OP_NOP.
2. req.cqe.user_data = user_data.
3. io_req_set_res(req, res, cflags).
4. percpu_ref_get(&ctx.refs).
5. req.ctx = ctx; req.tctx = None.
6. req.io_task_work.func = MsgRing::tw_complete.
7. io_req_task_work_add_remote(req, IOU_F_TWQ_LAZY_WAKE).

`MsgRing::tw_complete(tw_req, tw)`:
1. ctx = req.ctx.
2. io_add_aux_cqe(ctx, req.cqe.user_data, req.cqe.res, req.cqe.flags).
3. kfree_rcu(req, rcu_head).
4. percpu_ref_put(&ctx.refs).

`MsgRing::fd_remote(req)`:
1. task = target_ctx.submitter_task.
2. init_task_work(&msg.tw, MsgRing::tw_fd_complete).
3. if task_work_add(task, &msg.tw, TWA_SIGNAL) != 0: return -EOWNERDEAD.
4. return IOU_ISSUE_SKIP_COMPLETE.

`MsgRing::tw_fd_complete(head)`:
1. msg = container_of(head, IoMsg, tw).
2. req = cmd_to_io_kiocb(msg).
3. ret = -EOWNERDEAD.
4. if !(current.flags & PF_EXITING):
   - ret = MsgRing::install_complete(req, IO_URING_F_UNLOCKED).
5. if ret < 0: req_set_fail(req).
6. io_req_queue_tw_complete(req, ret).

`MsgRing::lock_external(octx, issue_flags)`:
1. if !(issue_flags & IO_URING_F_UNLOCKED):
   - if !mutex_trylock(octx.uring_lock): return -EAGAIN.
   - return 0.
2. mutex_lock(octx.uring_lock).
3. return 0.

`MsgRing::sync(sqe)`:
1. let io_msg = IoMsg::default().
2. MsgRing::prep_inner(&io_msg, sqe)?.
3. if io_msg.cmd != IORING_MSG_DATA: return -EINVAL.
4. CLASS(fd, f)(sqe.fd).
5. if fd_empty(f): return -EBADF.
6. if !io_is_uring_fops(fd_file(f)): return -EBADFD.
7. return MsgRing::data_inner(fd_file(f).private_data, &io_msg, IO_URING_F_UNLOCKED).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_mask_validated` | INVARIANT | per-prep: msg.flags ⊆ IORING_MSG_RING_MASK. |
| `data_src_fd_zero` | INVARIANT | per-DATA: msg.src_fd == 0. |
| `data_dst_fd_zero_no_pass` | INVARIANT | per-DATA: !FLAGS_PASS ⟹ msg.dst_fd == 0. |
| `send_fd_no_self` | INVARIANT | per-SEND_FD: target_ctx != ctx. |
| `send_fd_len_zero` | INVARIANT | per-SEND_FD: msg.len == 0. |
| `r_disabled_acquire` | INVARIANT | per-data/send_fd: smp_load_acquire on target_ctx.flags. |
| `iopoll_lock_paired` | INVARIANT | per-data: SETUP_IOPOLL acquire/release of target uring_lock balanced. |
| `lock_external_no_deadlock` | INVARIANT | per-lock_external: when source-locked, only trylock (no blocking acquire). |
| `grab_file_refcount` | INVARIANT | per-grab_file: success ⟹ get_file ∧ REQ_F_NEED_CLEANUP. |
| `cleanup_drops_src_file` | INVARIANT | per-cleanup: src_file fput'd exactly once. |
| `remote_post_percpu_ref_balanced` | INVARIANT | per-remote-DATA: percpu_ref_get on enqueue, _put on tw_complete. |
| `fd_remote_task_work_skip_complete` | INVARIANT | per-fd_remote: task_work_add success ⟹ IOU_ISSUE_SKIP_COMPLETE. |

### Layer 2: TLA+

`io_uring/msg-ring.tla`:
- Per-source ring + target ring with separate uring_lock mutexes; per-target IO_RING_F_TASK_COMPLETE optional.
- Operations: PrepMsg, IssueData, IssueSendFD, RemoteData, RemoteFD, TwComplete, TwFdComplete, SyncMsg, Cleanup.
- Properties:
  - `safety_no_ab_ba` — per-issue: source-locked path uses trylock on target; no nested blocking acquire.
  - `safety_self_send_rejected` — per-SEND_FD: target_ctx == ctx ⟹ -EINVAL.
  - `safety_no_cqe_when_skip` — per-CQE_SKIP install: no target CQE posted on success.
  - `safety_remote_post_no_cqring_lock` — per-remote: no acquire of target uring_lock (delivery via task_work).
  - `safety_fd_install_atomic` — per-FD: ret >= 0 ⟹ msg.src_file == NULL (ownership transferred).
  - `liveness_remote_data_completes` — per-remote-DATA: eventually io_msg_tw_complete fires (assuming target ring stays alive).
  - `liveness_remote_fd_completes_or_owner_dead` — per-remote-FD: tw_fd_complete posts result or -EOWNERDEAD (PF_EXITING).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MsgRing::prep` post: msg.flags ⊆ IORING_MSG_RING_MASK ∧ src_file = None | `MsgRing::prep` |
| `MsgRing::issue` post: return ∈ {IOU_COMPLETE, -EAGAIN, IOU_ISSUE_SKIP_COMPLETE} | `MsgRing::issue` |
| `MsgRing::data_inner` post: success ⟹ CQE posted on target_ctx | `MsgRing::data_inner` |
| `MsgRing::install_complete` post: success ⟹ fd installed in target_ctx.fixed_file_table[dst_fd] | `MsgRing::install_complete` |
| `MsgRing::grab_file` post: success ⟹ src_file refcount ≥ 1 ∧ REQ_F_NEED_CLEANUP set | `MsgRing::grab_file` |
| `MsgRing::data_remote` post: success ⟹ task_work queued on target_ctx | `MsgRing::data_remote` |
| `MsgRing::fd_remote` post: success ⟹ task_work queued on target_ctx.submitter_task ∧ ret == IOU_ISSUE_SKIP_COMPLETE | `MsgRing::fd_remote` |
| `MsgRing::tw_complete` post: ctx percpu_ref balanced; req kfree_rcu'd | `MsgRing::tw_complete` |
| `MsgRing::sync` post: cmd == IORING_MSG_DATA enforced; target fd is uring fops | `MsgRing::sync` |

### Layer 4: Verus/Creusot functional

`Per-source-SQE → IORING_OP_MSG_RING (DATA | SEND_FD) → target-CQE (or fd install) → optional source-CQE-skip` semantic equivalence: per-include/uapi/linux/io_uring.h IORING_MSG_RING + IORING_MSG_DATA + IORING_MSG_SEND_FD + IORING_MSG_RING_CQE_SKIP + IORING_MSG_RING_FLAGS_PASS contract.

`Per-IO_RING_F_TASK_COMPLETE target → remote delivery via io_req_task_work_add_remote / task_work_add(submitter_task, TWA_SIGNAL)` semantic equivalence: per-target-task-context completion model.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

MSG_RING reinforcement:

- **Per-IORING_MSG_RING_MASK strict flag validation** — defense against per-future-uapi-bit smuggling.
- **Per-prep buf_index / personality rejection** — defense against per-SQE-field misuse silently inheriting non-msg-ring semantics.
- **Per-non-uring target file rejection (-EBADFD)** — defense against per-arbitrary-fd msg-ring crash.
- **Per-self-send (target_ctx == ctx) -EINVAL on SEND_FD** — defense against per-self-loop fd-install corruption.
- **Per-IORING_SETUP_R_DISABLED smp_load_acquire** — defense against per-race-with-target-disable submitter_task tear-off.
- **Per-mutex_trylock-only when source-locked** — defense against per-AB-BA deadlock across two rings.
- **Per-io_msg_ring_cleanup REQ_F_NEED_CLEANUP** — defense against per-src_file refcount leak on failure path.
- **Per-grab_file under source ring submit_lock** — defense against per-src_fd table mutation race.
- **Per-percpu_ref_get on remote-post + _put on tw_complete** — defense against per-target_ctx UAF when target rings closes mid-delivery.
- **Per-IO_RING_F_TASK_COMPLETE remote-path** — defense against per-cross-task completion on rings that require submitter-task semantics.
- **Per-PF_EXITING guard in tw_fd_complete** — defense against per-task-exit fd-install crash.
- **Per-io_uring_sync_msg_ring SEND_FD reject** — defense against per-no-source-ring fd-grab undefined behavior.
- **Per-IORING_SETUP_IOPOLL target trylock with -EAGAIN punt** — defense against per-IOPOLL ring lock contention stall.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY on MSG_DATA** — bounds-checked copy on the `user_data` / `len` / `flags` fields and on the SEND_FD `src_fd` interpreted via the source ring's fixed-file table.
- **PAX_KERNEXEC** — write-protects `io_msg_ring_data`, `io_msg_send_fd`, and the cross-ring task-work entry points after init.
- **PAX_RANDKSTACK** — per-call stack-offset randomization on every `io_msg_ring` issue path, on the remote-post task-work, and on the SEND_FD `fd_install` callback.
- **PAX_REFCOUNT** — saturating trap on the `percpu_ref_get` taken on the target ctx and on `tw_complete` `_put` symmetry; on the source ring's `file->f_count` taken by `grab_file`.
- **PAX_MEMORY_SANITIZE** — zeroes `io_msg` slab/stack frame on tw_complete; scrubs the source-side `req` once posted to the target.
- **PAX_UDEREF** — the SEND_FD path dereferences source-ring file table entries via kernel pointer only; user payload (`addr3`) is faulted at prep.
- **PAX_RAP / kCFI** — forward-edge CFI on the target-side `tw_complete` callback (`io_msg_tw_fd_complete`, `io_msg_tw_complete`).
- **GRKERNSEC_HIDESYM** — strips msg-ring symbols and source/target ctx pointers from `/proc/kallsyms`.
- **GRKERNSEC_DMESG** — restricts cross-ring delivery `pr_debug` traces to CAP_SYSLOG.
- **Cross-ring CAP_SYS_NICE** — issuing IORING_OP_MSG_RING against a target ring owned by a different task is gated by CAP_SYS_NICE; defends against per-target-ring CQE-storm DoS and per-cross-uid completion injection without explicit privilege.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `io_uring/io_uring.c` core SQE/CQE pipeline (covered in `io_uring-core.md` Tier-3).
- `io_uring/rsrc.c` fixed-file table + io_rsrc_node lookup (covered in `rsrc.md` Tier-3).
- `io_uring/filetable.c` __io_fixed_fd_install internals (covered separately if expanded).
- `io_uring/sqpoll.c` SQPOLL kthread (covered in `sqpoll.md` Tier-3).
- `io_uring/poll.c` poll (covered in `poll.md` Tier-3).
- `io_uring/cancel.c` cancel (covered in `cancel.md` Tier-3).
- Implementation code.
