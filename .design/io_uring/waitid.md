# Tier-3: io_uring/waitid.c — IORING_OP_WAITID async wait4/waitid

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/waitid.c (~348 lines)
  - io_uring/waitid.h
  - kernel/exit.h (struct wait_opts, __do_wait, kernel_waitid_prepare, pid_child_should_wake)
  - include/uapi/linux/wait.h (P_ALL, P_PID, P_PGID, P_PIDFD)
  - include/uapi/linux/io_uring.h (IORING_OP_WAITID)
-->

## Summary

`IORING_OP_WAITID` provides **async waitid(2)** through io_uring. Per-submission: SQE carries `which` (P_ALL / P_PID / P_PGID / P_PIDFD) in `sqe->len`, `upid` in `sqe->fd`, `options` (WEXITED / WSTOPPED / WCONTINUED / WNOHANG / WNOWAIT) in `sqe->file_index`, and a user `struct siginfo *infop` in `sqe->addr2`. Per-issue: the request piggy-backs on `wait_chldexit` waitqueue under `current->signal->wait_chldexit` via a custom `wait_queue_entry_t` callback (`io_waitid_wait`). When a child changes state, the callback invokes `pid_child_should_wake()` and if matched, queues task work (`io_waitid_cb`) which re-runs `__do_wait()` to harvest the child state. On `-ERESTARTSYS`, the request re-arms on the waitqueue (one retry). Per-cancellation: a refcount with `IO_WAITID_CANCEL_FLAG` (bit 31) and a 31-bit ref mask (`IO_WAITID_REF_MASK`) coordinates wakeup ↔ cancel races. Per-`siginfo` copy: writes `si_signo / si_errno / si_code / si_pid / si_uid / si_status` (or compat 32-bit variant) into userspace via `user_write_access_begin`.

This Tier-3 covers `io_uring/waitid.c` (~348 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_waitid` | per-cmd context | `IoWaitid` |
| `struct io_waitid_async` | per-async backing (req + wait_opts) | `IoWaitidAsync` |
| `io_waitid_prep()` | per-SQE parse | `Waitid::prep` |
| `io_waitid()` | per-issue dispatch | `Waitid::issue` |
| `io_waitid_cancel()` | per-cancel lookup | `Waitid::cancel` |
| `io_waitid_remove_all()` | per-ring teardown | `Waitid::remove_all` |
| `io_waitid_wait()` | per-wakeup callback | `Waitid::wait_callback` |
| `io_waitid_cb()` | per-task-work harvest | `Waitid::tw_callback` |
| `io_waitid_complete()` | per-completion | `Waitid::complete` |
| `io_waitid_finish()` | per-finish copy_si | `Waitid::finish` |
| `io_waitid_free()` | per-req-async-data free | `Waitid::free` |
| `io_waitid_remove_wq()` | per-waitqueue unlink | `Waitid::remove_wq` |
| `io_waitid_copy_si()` | per-userspace siginfo copy | `Waitid::copy_si` |
| `io_waitid_compat_copy_si()` | per-compat siginfo copy | `Waitid::compat_copy_si` |
| `io_waitid_drop_issue_ref()` | per-issue ref drop | `Waitid::drop_issue_ref` |
| `__io_waitid_cancel()` | per-individual cancel | `Waitid::cancel_one` |
| `IO_WAITID_CANCEL_FLAG` | refs bit 31 | shared |
| `IO_WAITID_REF_MASK` | refs bits 0..30 | shared |
| `struct wait_opts` (kernel/exit.h) | upstream waitid context | shared |
| `kernel_waitid_prepare()` | per-options parse | shared |
| `__do_wait()` | per-task-state harvest | shared |
| `pid_child_should_wake()` | per-callback filter | shared |
| `struct waitid_info` | per-result block | shared |

## Compatibility contract

REQ-1: struct io_waitid:
- file: per-req file (unused; required by io_kiocb_to_cmd).
- which: P_ALL / P_PID / P_PGID / P_PIDFD (enum pid_type-equivalent, per `sqe->len`).
- upid: pid value (per `sqe->fd`).
- options: WEXITED | WSTOPPED | WCONTINUED | WNOHANG | WNOWAIT | __WALL | __WCLONE | __WNOTHREAD (per `sqe->file_index`).
- refs: atomic_t — bit 31 = IO_WAITID_CANCEL_FLAG; bits 0..30 = ref count.
- head: pointer to `current->signal->wait_chldexit` waitqueue head while armed; NULL otherwise (release-ordered).
- infop: userspace `struct siginfo __user *` (per `sqe->addr2`).
- info: `struct waitid_info` result block (pid, uid, status, cause).

REQ-2: struct io_waitid_async (async_data backing):
- req: back-pointer to `struct io_kiocb`.
- wo: embedded `struct wait_opts` (wo_type, wo_flags, wo_pid, wo_info, wo_stat, wo_rusage, child_wait wait-queue entry, notask_error).

REQ-3: io_waitid_prep(req, sqe):
- /* Reserved fields must be zero */
- if sqe->addr ∨ sqe->buf_index ∨ sqe->addr3 ∨ sqe->waitid_flags: return -EINVAL.
- /* Allocate async backing (no alloc_cache; NULL) */
- iwa = io_uring_alloc_async_data(NULL, req).
- if !iwa: return -ENOMEM.
- iwa.req = req.
- /* Latch UAPI fields */
- iw.which = READ_ONCE(sqe->len).
- iw.upid = READ_ONCE(sqe->fd).
- iw.options = READ_ONCE(sqe->file_index).
- iw.head = NULL.
- iw.infop = u64_to_user_ptr(READ_ONCE(sqe->addr2)).
- return 0.

REQ-4: io_waitid(req, issue_flags):
- /* Reuse kernel waitid options-validation + wo init */
- ret = kernel_waitid_prepare(&iwa.wo, iw.which, iw.upid, &iw.info, iw.options, NULL).
- if ret: goto done.
- /* Initial ref = 1 (issuer holds) */
- atomic_set(&iw.refs, 1).
- /* Linkage under ring lock */
- io_ring_submit_lock(ctx, issue_flags).
- iw.head = &current.signal.wait_chldexit.
- hlist_add_head(&req.hash_node, &ctx.waitid_list).
- /* Custom waitqueue entry */
- init_waitqueue_func_entry(&iwa.wo.child_wait, io_waitid_wait).
- iwa.wo.child_wait.private = req.tctx.task.
- add_wait_queue(iw.head, &iwa.wo.child_wait).
- /* First harvest attempt */
- ret = __do_wait(&iwa.wo).
- if ret == -ERESTARTSYS:
  - if !io_waitid_drop_issue_ref(req):
    - io_ring_submit_unlock(ctx, issue_flags).
    - return IOU_ISSUE_SKIP_COMPLETE.
  - /* Wakeup raced ⇒ tw was queued; we still skip-complete */
  - io_ring_submit_unlock(ctx, issue_flags).
  - return IOU_ISSUE_SKIP_COMPLETE.
- /* Synchronous completion path (NOHANG, or already-reaped child) */
- hlist_del_init(&req.hash_node).
- io_waitid_remove_wq(req).
- ret = io_waitid_finish(req, ret).
- io_ring_submit_unlock(ctx, issue_flags).
- done:
- if ret < 0: req_set_fail(req).
- io_req_set_res(req, ret, 0).
- return IOU_COMPLETE.

REQ-5: io_waitid_wait(wait, mode, sync, key) — wait_queue callback:
- /* key is the task that changed state */
- p = key (task_struct).
- if !pid_child_should_wake(wo, p): return 0.
- /* Match: detach from waitqueue inline */
- list_del_init(&wait.entry).
- smp_store_release(&iw.head, NULL).
- /* Claim ownership via ref-inc; if prior nonzero ref ⇒ canceler/issuer holds */
- if atomic_fetch_inc(&iw.refs) & IO_WAITID_REF_MASK: return 1.
- /* Queue task-work to harvest */
- req.io_task_work.func = io_waitid_cb.
- io_req_task_work_add(req).
- return 1.

REQ-6: io_waitid_cb(tw_req, tw) — task work:
- io_tw_lock(ctx, tw).
- ret = __do_wait(&iwa.wo).
- /* Spurious wakeup ⇒ one re-arm */
- if ret == -ERESTARTSYS:
  - ret = -ECANCELED.
  - if !(atomic_read(&iw.refs) & IO_WAITID_CANCEL_FLAG):
    - iw.head = &current.signal.wait_chldexit.
    - add_wait_queue(iw.head, &iwa.wo.child_wait).
    - ret = __do_wait(&iwa.wo).
    - if ret == -ERESTARTSYS:
      - /* Retry armed ⇒ drop our ref and bail */
      - io_waitid_drop_issue_ref(req).
      - return.
- io_waitid_complete(req, ret).
- io_req_task_complete(tw_req, tw).

REQ-7: io_waitid_complete(req, ret):
- /* Caller must hold a ref */
- WARN_ON_ONCE(!(atomic_read(&iw.refs) & IO_WAITID_REF_MASK)).
- lockdep_assert_held(&ctx.uring_lock).
- hlist_del_init(&req.hash_node).
- io_waitid_remove_wq(req).
- ret = io_waitid_finish(req, ret).
- if ret < 0: req_set_fail(req).
- io_req_set_res(req, ret, 0).

REQ-8: io_waitid_finish(req, ret):
- signo = 0.
- /* ret > 0 ⇒ a child was reaped */
- if ret > 0:
  - signo = SIGCHLD.
  - ret = 0.
- if !io_waitid_copy_si(req, signo): ret = -EFAULT.
- io_waitid_free(req).
- return ret.

REQ-9: io_waitid_copy_si(req, signo):
- if !iw.infop: return true (no userspace siginfo requested).
- if io_is_compat(ctx): return io_waitid_compat_copy_si(iw, signo).
- if !user_write_access_begin(iw.infop, sizeof(siginfo)): return false.
- unsafe_put_user(signo, &infop->si_signo).
- unsafe_put_user(0, &infop->si_errno).
- unsafe_put_user(iw.info.cause, &infop->si_code).
- unsafe_put_user(iw.info.pid, &infop->si_pid).
- unsafe_put_user(iw.info.uid, &infop->si_uid).
- unsafe_put_user(iw.info.status, &infop->si_status).
- user_write_access_end.
- return ok.

REQ-10: io_waitid_compat_copy_si — same as REQ-9 but `struct compat_siginfo` (32-bit pid_t / uid_t fields per UAPI compat layout).

REQ-11: io_waitid_remove_wq(req):
- head = smp_load_acquire(&iw.head).
- if head:
  - smp_store_release(&iw.head, NULL).
  - spin_lock_irq(&head.lock).
  - list_del_init(&iwa.wo.child_wait.entry).
  - spin_unlock_irq(&head.lock).

REQ-12: io_waitid_free(req):
- put_pid(iwa.wo.wo_pid).
- io_req_async_data_free(req).

REQ-13: io_waitid_drop_issue_ref(req):
- /* Issuer drops its ref; if no other party has incremented, return false */
- if atomic_sub_return(1, &iw.refs) == 0: return false.
- /* Wakeup raced ⇒ deferred path; cleanup wq + arm tw */
- io_waitid_remove_wq(req).
- req.io_task_work.func = io_waitid_cb.
- io_req_task_work_add(req).
- return true.

REQ-14: __io_waitid_cancel(req):
- lockdep_assert_held(&ctx.uring_lock).
- /* Set CANCEL flag regardless of ref ownership */
- atomic_or(IO_WAITID_CANCEL_FLAG, &iw.refs).
- /* Claim ownership: if prior ref nonzero, someone else owns */
- if atomic_fetch_inc(&iw.refs) & IO_WAITID_REF_MASK: return false.
- io_waitid_complete(req, -ECANCELED).
- io_req_queue_tw_complete(req, -ECANCELED).
- return true.

REQ-15: io_waitid_cancel(ctx, cd, issue_flags):
- return io_cancel_remove(ctx, cd, issue_flags, &ctx.waitid_list, __io_waitid_cancel).

REQ-16: io_waitid_remove_all(ctx, tctx, cancel_all):
- return io_cancel_remove_all(ctx, tctx, &ctx.waitid_list, cancel_all, __io_waitid_cancel).

REQ-17: Per-multi-shot vs single-shot:
- Upstream waitid.c implementation is single-shot per submission; multi-shot semantics (many children from one SQE) are achieved by userspace re-submitting after each CQE, **not** by a kernel multi-shot loop. (The framework supports multishot via `IOSQE_BUFFER_SELECT` / `REQ_F_APOLL_MULTISHOT` patterns elsewhere; waitid does not opt in.)

REQ-18: Per-idtype:
- P_ALL (0): wait for any child.
- P_PID (1): wait for exact pid.
- P_PGID (2): wait for any child in process group.
- P_PIDFD (3): wait for child identified by pidfd (upid is a pidfd in this mode).

## Acceptance Criteria

- [ ] AC-1: sqe->addr / buf_index / addr3 / waitid_flags nonzero ⇒ -EINVAL at prep.
- [ ] AC-2: kernel_waitid_prepare returning error ⇒ ret latched, completes via done path with -err in CQE.
- [ ] AC-3: WNOHANG with no pending child: synchronous CQE result 0 (no kids ready).
- [ ] AC-4: Child already reaped before issue: synchronous CQE with siginfo populated.
- [ ] AC-5: Child state change after issue: io_waitid_wait fires; tw harvests; CQE posted.
- [ ] AC-6: -ERESTARTSYS on initial __do_wait: returns IOU_ISSUE_SKIP_COMPLETE (armed on waitqueue).
- [ ] AC-7: Cancel via io_uring_register cancel: __io_waitid_cancel sets CANCEL_FLAG; CQE -ECANCELED.
- [ ] AC-8: io_waitid_remove_all on ring exit: every queued waitid request torn down.
- [ ] AC-9: siginfo copy: si_signo=SIGCHLD on success; si_code = info.cause (CLD_EXITED/CLD_KILLED/CLD_DUMPED/CLD_TRAPPED/CLD_STOPPED/CLD_CONTINUED).
- [ ] AC-10: Compat ABI: io_waitid_compat_copy_si used; struct compat_siginfo layout written.
- [ ] AC-11: infop NULL: no userspace copy attempted; CQE ret intact.
- [ ] AC-12: Wakeup ↔ cancel race: at most one of (complete, cancel-complete) issues CQE; refs counted.
- [ ] AC-13: head store/load uses acquire/release ordering (smp_load_acquire / smp_store_release).
- [ ] AC-14: P_PIDFD: kernel_waitid_prepare resolves pidfd into wo.wo_pid; put_pid in free.

## Architecture

```
struct IoWaitid {
  file: *mut File,
  which: i32,              // P_ALL | P_PID | P_PGID | P_PIDFD
  upid: pid_t,
  options: i32,            // WEXITED | WSTOPPED | WCONTINUED | WNOHANG | WNOWAIT | ...
  refs: AtomicU32,         // bit 31 = IO_WAITID_CANCEL_FLAG; bits 0..30 = ref count
  head: *mut WaitQueueHead, // current->signal->wait_chldexit (acquire/release)
  infop: *mut Siginfo,     // user
  info: WaitidInfo,        // pid, uid, status, cause
}

struct IoWaitidAsync {
  req: *mut IoKiocb,
  wo: WaitOpts {            // upstream struct wait_opts (kernel/exit.h)
    wo_type: PidType,
    wo_flags: i32,
    wo_pid: *mut Pid,
    wo_info: *mut WaitidInfo,
    wo_stat: i32,
    wo_rusage: *mut Rusage,
    child_wait: WaitQueueEntry,
    notask_error: i32,
  },
}
```

`Waitid::prep(req, sqe) -> Result<()>`:
1. if sqe.addr ∨ sqe.buf_index ∨ sqe.addr3 ∨ sqe.waitid_flags: return Err(EINVAL).
2. iwa = io_uring_alloc_async_data(None, req).
3. if !iwa: return Err(ENOMEM).
4. iwa.req = req.
5. iw.which = READ_ONCE(sqe.len).
6. iw.upid = READ_ONCE(sqe.fd).
7. iw.options = READ_ONCE(sqe.file_index).
8. iw.head = ptr::null_mut().
9. iw.infop = u64_to_user_ptr(READ_ONCE(sqe.addr2)).
10. Ok(()).

`Waitid::issue(req, issue_flags) -> i32`:
1. ret = kernel_waitid_prepare(&iwa.wo, iw.which, iw.upid, &iw.info, iw.options, None).
2. if ret: goto done.
3. atomic_set(&iw.refs, 1).
4. io_ring_submit_lock(ctx, issue_flags).
5. iw.head = &current.signal.wait_chldexit.
6. hlist_add_head(&req.hash_node, &ctx.waitid_list).
7. init_waitqueue_func_entry(&iwa.wo.child_wait, Waitid::wait_callback).
8. iwa.wo.child_wait.private = req.tctx.task.
9. add_wait_queue(iw.head, &iwa.wo.child_wait).
10. ret = __do_wait(&iwa.wo).
11. if ret == -ERESTARTSYS:
    - if !Waitid::drop_issue_ref(req):
      - io_ring_submit_unlock(ctx, issue_flags).
      - return IOU_ISSUE_SKIP_COMPLETE.
    - io_ring_submit_unlock(ctx, issue_flags).
    - return IOU_ISSUE_SKIP_COMPLETE.
12. hlist_del_init(&req.hash_node).
13. Waitid::remove_wq(req).
14. ret = Waitid::finish(req, ret).
15. io_ring_submit_unlock(ctx, issue_flags).
16. done:
17. if ret < 0: req_set_fail(req).
18. io_req_set_res(req, ret, 0).
19. return IOU_COMPLETE.

`Waitid::wait_callback(wait, mode, sync, key) -> i32`:
1. wo = container_of(wait, struct wait_opts, child_wait).
2. iwa = container_of(wo, struct io_waitid_async, wo).
3. req = iwa.req.
4. p = key (task_struct).
5. if !pid_child_should_wake(wo, p): return 0.
6. list_del_init(&wait.entry).
7. smp_store_release(&iw.head, NULL).
8. if atomic_fetch_inc(&iw.refs) & IO_WAITID_REF_MASK: return 1.
9. req.io_task_work.func = Waitid::tw_callback.
10. io_req_task_work_add(req).
11. return 1.

`Waitid::tw_callback(tw_req, tw)`:
1. io_tw_lock(ctx, tw).
2. ret = __do_wait(&iwa.wo).
3. if ret == -ERESTARTSYS:
   - ret = -ECANCELED.
   - if !(atomic_read(&iw.refs) & IO_WAITID_CANCEL_FLAG):
     - iw.head = &current.signal.wait_chldexit.
     - add_wait_queue(iw.head, &iwa.wo.child_wait).
     - ret = __do_wait(&iwa.wo).
     - if ret == -ERESTARTSYS:
       - Waitid::drop_issue_ref(req).
       - return.
4. Waitid::complete(req, ret).
5. io_req_task_complete(tw_req, tw).

`Waitid::complete(req, ret)`:
1. WARN_ON_ONCE(!(atomic_read(&iw.refs) & IO_WAITID_REF_MASK)).
2. lockdep_assert_held(&ctx.uring_lock).
3. hlist_del_init(&req.hash_node).
4. Waitid::remove_wq(req).
5. ret = Waitid::finish(req, ret).
6. if ret < 0: req_set_fail(req).
7. io_req_set_res(req, ret, 0).

`Waitid::finish(req, ret) -> i32`:
1. signo = 0.
2. if ret > 0: signo = SIGCHLD; ret = 0.
3. if !Waitid::copy_si(req, signo): ret = -EFAULT.
4. Waitid::free(req).
5. return ret.

`Waitid::copy_si(req, signo) -> bool`:
1. if !iw.infop: return true.
2. if io_is_compat(ctx): return Waitid::compat_copy_si(iw, signo).
3. user_write_access_begin(iw.infop, sizeof(siginfo)) or return false.
4. unsafe_put_user(signo, &infop.si_signo).
5. unsafe_put_user(0, &infop.si_errno).
6. unsafe_put_user(iw.info.cause, &infop.si_code).
7. unsafe_put_user(iw.info.pid, &infop.si_pid).
8. unsafe_put_user(iw.info.uid, &infop.si_uid).
9. unsafe_put_user(iw.info.status, &infop.si_status).
10. user_write_access_end. return true.

`Waitid::remove_wq(req)`:
1. head = smp_load_acquire(&iw.head).
2. if head:
   - smp_store_release(&iw.head, NULL).
   - spin_lock_irq(&head.lock).
   - list_del_init(&iwa.wo.child_wait.entry).
   - spin_unlock_irq(&head.lock).

`Waitid::drop_issue_ref(req) -> bool`:
1. if atomic_sub_return(1, &iw.refs) == 0: return false.
2. Waitid::remove_wq(req).
3. req.io_task_work.func = Waitid::tw_callback.
4. io_req_task_work_add(req).
5. return true.

`Waitid::cancel_one(req) -> bool`:
1. lockdep_assert_held(&ctx.uring_lock).
2. atomic_or(IO_WAITID_CANCEL_FLAG, &iw.refs).
3. if atomic_fetch_inc(&iw.refs) & IO_WAITID_REF_MASK: return false.
4. Waitid::complete(req, -ECANCELED).
5. io_req_queue_tw_complete(req, -ECANCELED).
6. Ok(true).

`Waitid::cancel(ctx, cd, issue_flags) -> i32`:
1. return io_cancel_remove(ctx, cd, issue_flags, &ctx.waitid_list, Waitid::cancel_one).

`Waitid::remove_all(ctx, tctx, cancel_all) -> bool`:
1. return io_cancel_remove_all(ctx, tctx, &ctx.waitid_list, cancel_all, Waitid::cancel_one).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prep_rejects_reserved` | INVARIANT | per-prep: addr / buf_index / addr3 / waitid_flags nonzero ⇒ -EINVAL. |
| `refs_initial_one` | INVARIANT | per-issue: after kernel_waitid_prepare success, refs == 1. |
| `head_acquire_release` | INVARIANT | per-head: store uses release; load uses acquire. |
| `cancel_flag_set_atomic` | INVARIANT | per-cancel: atomic_or(IO_WAITID_CANCEL_FLAG) ordered before claim. |
| `claim_via_fetch_inc` | INVARIANT | per-wait_callback / per-cancel: ownership via atomic_fetch_inc test against IO_WAITID_REF_MASK. |
| `complete_under_uring_lock` | INVARIANT | per-complete: ctx.uring_lock held (lockdep). |
| `tw_one_retry_only` | INVARIANT | per-tw_callback: at most one re-arm on -ERESTARTSYS. |
| `infop_null_skips_copy` | INVARIANT | per-copy_si: !iw.infop ⇒ true (no UAF). |
| `compat_path_selected_by_io_is_compat` | INVARIANT | per-copy_si: io_is_compat ⇒ compat_copy_si. |
| `remove_wq_idempotent` | INVARIANT | per-remove_wq: head NULL ⇒ no-op. |
| `wo_pid_put_in_free` | INVARIANT | per-free: put_pid(iwa.wo.wo_pid) called exactly once. |
| `hash_node_del_on_complete_or_cancel` | INVARIANT | per-terminal: hlist_del_init called on completion or cancel. |

### Layer 2: TLA+

`io_uring/waitid.tla`:
- States: ISSUED, ARMED, FIRED, COMPLETED, CANCELED.
- Per-issue ARMED on add_wait_queue; per-callback FIRED; per-tw COMPLETED; per-cancel CANCELED.
- Properties:
  - `safety_one_terminal_cqe` — per-request: exactly one CQE posted (success / -ECANCELED / -EFAULT / -err).
  - `safety_no_wq_use_after_complete` — per-remove_wq: no list ops after hlist_del_init.
  - `safety_cancel_flag_blocks_retry` — per-tw: CANCEL_FLAG set ⇒ no re-arm in tw_callback.
  - `safety_ref_inc_per_owner` — per-refs: at most one of {issuer, waker, canceler} holds non-zero count concurrently after claim.
  - `safety_pidfd_pid_released` — per-P_PIDFD: put_pid in free regardless of completion path.
  - `safety_compat_path_for_compat_ctx` — per-CTX compat: writes compat_siginfo layout.
  - `liveness_armed_eventually_completes` — per-request: armed ⇒ eventually CQE (assuming child state-change or cancel).
  - `liveness_ring_exit_drains_waitid_list` — per-ring exit: remove_all empties waitid_list.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Waitid::prep` post: iw fields latched; iwa.req == req | `Waitid::prep` |
| `Waitid::issue` post: ret ∈ {IOU_ISSUE_SKIP_COMPLETE, IOU_COMPLETE} | `Waitid::issue` |
| `Waitid::wait_callback` post: return 0 if not matched; return 1 if matched (with tw queued or canceler-owns) | `Waitid::wait_callback` |
| `Waitid::tw_callback` post: req completed ∨ re-armed exactly once | `Waitid::tw_callback` |
| `Waitid::complete` post: CQE set; hash_node removed; wq detached | `Waitid::complete` |
| `Waitid::finish` post: copy_si called; iwa freed | `Waitid::finish` |
| `Waitid::copy_si` post: returns true iff user_write_access succeeded for all 6 fields | `Waitid::copy_si` |
| `Waitid::drop_issue_ref` post: refs==0 ⇒ false (issuer-only owns); refs>0 ⇒ tw armed | `Waitid::drop_issue_ref` |
| `Waitid::cancel_one` post: CANCEL_FLAG set; ownership claimed ⇒ -ECANCELED CQE | `Waitid::cancel_one` |
| `Waitid::remove_wq` post: iw.head == NULL ∧ child_wait.entry detached | `Waitid::remove_wq` |

### Layer 4: Verus/Creusot functional

`Per-IORING_OP_WAITID → prep (latch which/upid/options/infop) → issue (kernel_waitid_prepare for options validation ∧ wo_pid resolution; refs=1; arm on current.signal.wait_chldexit via custom callback; first __do_wait attempt) → either synchronous complete (NOHANG / already-reaped) ∨ -ERESTARTSYS armed → wakeup callback (pid_child_should_wake filter; detach + queue tw) → tw harvest (__do_wait; optional one-shot re-arm) → complete (copy_si user_write_access; finish; free; CQE) ∨ cancel (CANCEL_FLAG; -ECANCELED CQE)` semantic equivalence: per `man 2 waitid` POSIX semantics, per kernel/exit.c::kernel_waitid + do_wait, per liburing test/waitid.c.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

waitid reinforcement:

- **Per-reserved-fields strict-zero (`sqe->addr`, `buf_index`, `addr3`, `waitid_flags`)** — defense against per-future-ABI-collision via reserved bytes.
- **Per-`kernel_waitid_prepare` reuse** — defense against per-divergent-options-validation between syscall waitid(2) and IORING_OP_WAITID.
- **Per-`iw.head` smp_load_acquire / smp_store_release** — defense against per-torn-pointer when concurrent wakeup observes the head field.
- **Per-`IO_WAITID_CANCEL_FLAG` ordered atomic_or before claim** — defense against per-cancel-races-with-completion that could double-post a CQE.
- **Per-`atomic_fetch_inc` claim test against `IO_WAITID_REF_MASK`** — defense against per-double-complete from racing waker + canceler + issuer.
- **Per-`pid_child_should_wake` filter inside callback** — defense against per-wrong-child wakeup (P_PID mismatch / WCLONE filter / __WALL).
- **Per-`io_tw_lock` taken in `io_waitid_cb`** — defense against per-tw racing with submission and cancel.
- **Per-`lockdep_assert_held(&ctx.uring_lock)` in `__io_waitid_cancel` and `io_waitid_complete`** — defense against per-unsynchronized list mutation.
- **Per-one-retry cap on -ERESTARTSYS in tw** — defense against per-livelock spinning on spurious wakeups.
- **Per-`user_write_access_begin / unsafe_put_user / user_write_access_end` envelope** — defense against per-uaccess-without-stac/clac (SMAP / pkey).
- **Per-`io_waitid_compat_copy_si` for compat ABI** — defense against per-32-on-64 siginfo layout corruption.
- **Per-`put_pid(iwa.wo.wo_pid)` in free** — defense against per-pid refcount leak (especially P_PIDFD).
- **Per-`hlist_del_init` on both terminal paths** — defense against per-stale waitid_list entry after free.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-check `copy_siginfo_to_user` and `__put_user(&iwa.wo.wo_stat)` against the userspace `siginfo_t __user *` slab whitelist; reject any partial copy that could leak kernel `struct waitid_async` bytes.
- **PAX_KERNEXEC** — keep the `io_op_def[IORING_OP_WAITID]` table entry and `io_waitid_wait` notifier ops in `__ro_after_init`; reject late patching of `wo_pid`/`wo_info` callbacks.
- **PAX_RANDKSTACK** — re-randomize stack on each `io_waitid_prep`/`io_waitid` entry; the embedded `struct wait_opts` is large and a stack-disclosure target.
- **PAX_REFCOUNT** — saturating `get_pid`/`put_pid` on `wo_pid` so a malicious task cannot wrap the pid refcount via repeated `P_PIDFD` submissions and force UAF in `__wake_up`.
- **PAX_MEMORY_SANITIZE** — zero `struct io_waitid_async` on free (it carries `siginfo_t` with potentially sensitive `si_uid`/`si_pid`/`si_status`); never recycle bytes into a fresh waitid op.
- **PAX_UDEREF** — strict user/kernel split when reading `infop __user *` and writing back `siginfo_t`; the per-syscall stub MUST NOT deref a userspace pointer in kernel mode.
- **PAX_RAP / kCFI** — type-check the `io_waitid_complete` callback chain and `wait_queue_func_t` slots; mismatch panics rather than chains.
- **GRKERNSEC_HIDESYM** — hide `io_waitid_*`, `kernel_waitid`, and `do_wait` from non-root `/proc/kallsyms` to deny gadget discovery for waitid-based exploits.
- **GRKERNSEC_DMESG** — restrict dmesg so wait-queue debug spew cannot reveal pid layout or task struct addresses.
- **WNOWAIT + GRKERNSEC_HARDEN_PTRACE** — under HARDEN_PTRACE, treat `WNOWAIT` like a ptrace-attach probe: deny cross-uid `WNOWAIT` and audit; otherwise WNOWAIT becomes an unmetered task-state oracle for sibling processes.
- **P_PIDFD lifecycle** — re-validate `pidfd_get_pid(fd)` against current creds at completion (not just at prep); a SCM_RIGHTS-passed pidfd MUST NOT escalate the waiter's view of a different user's child.
- **Rationale** — `io_uring` waitid runs the wake callback in softirq with the target task's refcount; without PAX_REFCOUNT + PAX_RAP an attacker who wins the cancel-vs-wake race can both UAF the `io_waitid_async` and divert the indirect call. The WNOWAIT and pidfd cases additionally turn waitid into a cross-namespace info leak that must inherit ptrace-hardening rules.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/exit.c::kernel_waitid / do_wait / wait_consider_task (covered in `kernel/exit.md` Tier-3 if expanded)
- kernel/pid.c / pidfd lifecycle (covered separately if expanded)
- io_uring/cancel.c io_cancel_remove framework (covered in `cancel.md` Tier-3)
- io_uring/io_uring.c task-work plumbing (covered in `io_uring-core.md` Tier-3)
- userspace siginfo / compat_siginfo UAPI layout (UAPI surface; not a Tier-3 design target)
- signal delivery (SIGCHLD generation) (covered in `kernel/signal.md` Tier-3 if expanded)
- Implementation code
