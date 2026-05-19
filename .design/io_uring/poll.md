# Tier-3: io_uring/poll.c — Poll-armed completions and async-poll

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/poll.c (~972 lines)
  - io_uring/poll.h
  - include/uapi/linux/io_uring.h (IORING_OP_POLL_ADD, IORING_OP_POLL_REMOVE, IORING_POLL_ADD_MULTI, IORING_POLL_ADD_LEVEL, IORING_POLL_UPDATE_EVENTS, IORING_POLL_UPDATE_USER_DATA)
-->

## Summary

`io_uring/poll.c` implements two related mechanisms: **(a) explicit poll requests** (`IORING_OP_POLL_ADD` / `IORING_OP_POLL_REMOVE`) — userspace asks the ring to deliver a CQE when a file becomes readable/writable; and **(b) async-poll** — when a non-poll op (e.g. recv, send, accept) would block, `io_arm_poll_handler` arms a poll wait on the file's waitqueue(s) so the op gets retried from `task_work` when the file is ready, without consuming a kernel worker thread. Per-`struct io_poll` holds `(file, head, events, retries, wait)`; per-`struct async_poll` holds `(poll, *double_poll)` where double-poll is a second `io_poll` allocated when the file uses multiple waitqueues (e.g. socket POLLIN-readers and POLLOUT-writers on separate queues). Per-`__io_arm_poll_handler`: initialize `io_poll`, invoke `vfs_poll(file, &ipt.pt)` whose `_qproc` callback (`io_poll_queue_proc` for explicit, `io_async_queue_proc` for apoll) installs the `io_poll.wait` (`wake_func = io_poll_wake`) onto each waitqueue the file calls `poll_wait(file, head, p)` against. Per-`io_poll_wake`: on wakeup, check mask vs requested events, atomically grab ownership (`poll_refs`), and enqueue `task_work` (`io_poll_task_func`). Per-`io_poll_check_events`: in task_work, re-`vfs_poll`, post CQE (one-shot ⟹ remove, multishot ⟹ keep armed via `IORING_CQE_F_MORE`). Per-`IORING_POLL_ADD_MULTI`: multishot — re-armed after each delivery without userspace resubmission (saves syscall). Per-`IORING_POLL_ADD_LEVEL`: level-trigger (default is edge `EPOLLET`). Per-`EPOLLEXCLUSIVE`: opcodes with `def->poll_exclusive` add to the file waitqueue exclusively (avoids thundering-herd) — forces `REQ_F_POLL_NO_LAZY`. Per-`IO_POLL_REF_*`: atomic state machine on `req->poll_refs` — low 30 bits ≡ refcount, bit 30 ≡ `IO_POLL_RETRY_FLAG`, bit 31 ≡ `IO_POLL_CANCEL_FLAG`. Per-`io_poll_remove`: poll-cancel/update (`IORING_OP_POLL_REMOVE` = update opcode despite legacy name; supports `IORING_POLL_UPDATE_EVENTS` / `_USER_DATA`). Critical for: efficient socket I/O, latency-bounded recv/send, multishot accept, scalable wakeups.

This Tier-3 covers `io_uring/poll.c` (~972 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_poll` | per-poll request state | `IoPoll` |
| `struct async_poll` | per-apoll wrapper | `AsyncPoll` |
| `struct io_poll_update` | per-POLL_REMOVE (update) | `IoPollUpdate` |
| `struct io_poll_table` | per-arm scratch | `IoPollTable` |
| `io_poll_add()` | per-IORING_OP_POLL_ADD issue | `Poll::add` |
| `io_poll_add_prep()` | per-POLL_ADD prep | `Poll::add_prep` |
| `io_poll_remove()` | per-POLL_REMOVE (update) issue | `Poll::remove` |
| `io_poll_remove_prep()` | per-POLL_REMOVE prep | `Poll::remove_prep` |
| `io_poll_cancel()` | per-CANCEL-by-userdata | `Poll::cancel` |
| `__io_poll_cancel()` | per-cancel inner | `Poll::cancel_inner` |
| `__io_arm_poll_handler()` | per-arm (vfs_poll) | `Poll::arm_handler` |
| `io_arm_poll_handler()` | per-non-poll-op → apoll | `Poll::arm_for_async` |
| `io_arm_apoll()` | per-apoll arm | `Poll::arm_apoll` |
| `io_req_alloc_apoll()` | per-apoll alloc | `Poll::alloc_apoll` |
| `io_poll_queue_proc()` | per-explicit qproc | `Poll::queue_proc_explicit` |
| `io_async_queue_proc()` | per-apoll qproc | `Poll::queue_proc_async` |
| `__io_queue_proc()` | per-qproc inner (single/double) | `Poll::queue_proc_inner` |
| `io_poll_double_prepare()` | per-double-arm sync | `Poll::double_prepare` |
| `io_poll_wake()` | per-wakeup func | `Poll::wake` |
| `io_pollfree_wake()` | per-POLLFREE handling | `Poll::pollfree_wake` |
| `__io_poll_execute()` | per-tw enqueue | `Poll::execute_inner` |
| `io_poll_execute()` | per-tw enqueue (with ownership) | `Poll::execute` |
| `io_poll_task_func()` | per-tw entry | `Poll::task_func` |
| `io_poll_check_events()` | per-tw events loop | `Poll::check_events` |
| `io_poll_get_ownership()` | per-poll_refs CAS | `Poll::get_ownership` |
| `io_poll_get_ownership_slowpath()` | per-poll_refs slow | `Poll::get_ownership_slow` |
| `io_poll_mark_cancelled()` | per-cancel flag set | `Poll::mark_cancelled` |
| `io_poll_remove_all()` | per-ring cleanup | `Poll::remove_all` |
| `io_poll_remove_entry()` | per-remove single waitq | `Poll::remove_entry` |
| `io_poll_remove_entries()` | per-remove both waitqs | `Poll::remove_entries` |
| `io_poll_remove_waitq()` | per-list_del + head=NULL | `Poll::remove_waitq` |
| `io_poll_disarm()` | per-disarm before update | `Poll::disarm` |
| `io_poll_req_insert()` | per-hash insert | `Poll::req_insert_hash` |
| `io_poll_find()` | per-user_data lookup | `Poll::find_by_user_data` |
| `io_poll_file_find()` | per-cancel-by-fd lookup | `Poll::find_by_file` |
| `io_poll_parse_events()` | per-sqe → events | `Poll::parse_events` |
| `io_init_poll_iocb()` | per-init wait | `Poll::init_iocb` |
| `io_poll_get_single()` / `_double()` | per-poll accessor | `Poll::get_single` / `get_double` |
| `io_poll_cancel_req()` | per-mark + kick | `Poll::cancel_req` |
| `io_poll_mark_cancelled()` | per-CANCEL_FLAG | `Poll::mark_cancelled` |
| `wqe_to_req()` / `wqe_is_double()` | per-priv encoding | shared helpers |
| `IO_POLL_REF_MASK` / `_CANCEL_FLAG` / `_RETRY_FLAG` / `_BIAS` | per-poll_refs bits | constants |
| `IO_WQE_F_DOUBLE` | per-wqe priv flag | constant |
| `IO_ASYNC_POLL_COMMON` | per-events always set | constant |
| `APOLL_MAX_RETRY` | per-apoll loop bound | constant |

## Compatibility contract

REQ-1: `struct io_poll`:
- file: per-target file*.
- head: per-current `wait_queue_head*` (NULL if not queued / removed).
- events: `__poll_t` (mask incl. EPOLLET / EPOLLONESHOT / EPOLLEXCLUSIVE).
- retries: per-apoll retry budget.
- wait: `wait_queue_entry_t` (priv encodes `req | IO_WQE_F_DOUBLE`).

REQ-2: `struct async_poll`:
- poll: per-`io_poll`.
- double_poll: per-second `io_poll*` (NULL unless file uses two waitqueues).

REQ-3: `struct io_poll_update`:
- file: per-update req file*.
- old_user_data: per-target id (lookup key).
- new_user_data: per-new id.
- events: per-replacement events (if update_events).
- update_events / update_user_data: per-flag.

REQ-4: `struct io_poll_table`:
- pt: `poll_table_struct` (with `_qproc` ≡ queue-proc).
- req: per-arming kiocb.
- nr_entries: per-add count (1 ≡ single, 2 ≡ double, >2 ⟹ -EINVAL).
- error: per-prep error.
- owning: per-arm took ownership flag (for `IO_URING_F_UNLOCKED`).
- result_mask: per-inline-complete mask.

REQ-5: `req->poll_refs` atomic state:
- `IO_POLL_REF_MASK = GENMASK(29,0)` ≡ refcount.
- `IO_POLL_RETRY_FLAG = BIT(30)` ≡ retry-set when slowpath cannot inc.
- `IO_POLL_CANCEL_FLAG = BIT(31)` ≡ poison; tw returns -ECANCELED.
- `IO_POLL_REF_BIAS = 128` ≡ slowpath threshold to avoid wraparound.

REQ-6: `io_poll_get_ownership(req) -> bool`:
- if `atomic_read(refs) >= IO_POLL_REF_BIAS`: slowpath.
- else: `return !(atomic_fetch_inc(refs) & IO_POLL_REF_MASK)` (winner saw old refcount == 0).

REQ-7: `io_poll_get_ownership_slowpath(req) -> bool`:
- `v = atomic_fetch_or(IO_POLL_RETRY_FLAG, refs)`.
- if `v & IO_POLL_REF_MASK`: return false (someone else has it; they'll see retry).
- else: `return !(atomic_fetch_inc(refs) & IO_POLL_REF_MASK)`.

REQ-8: `io_init_poll_iocb(poll, events)`:
- `poll.head = NULL`.
- `poll.events = events | IO_POLL_UNMASK` where `IO_POLL_UNMASK = EPOLLERR | EPOLLHUP | EPOLLNVAL | EPOLLRDHUP` (events always wanted/needed).
- `INIT_LIST_HEAD(&poll.wait.entry)`.
- `init_waitqueue_func_entry(&poll.wait, io_poll_wake)`.

REQ-9: `__io_queue_proc(poll, pt, head, poll_ptr)` (the qproc invoked by `vfs_poll → poll_wait`):
- wqe_private = (unsigned long)req.
- if `pt->nr_entries != 0` (this is the 2nd or later waitqueue this file adds):
  - first = poll (the single).
  - if `first->head == head`: return (double-add on same wq, ignore).
  - if `*poll_ptr != NULL`:
    - if `(*poll_ptr)->head == head`: return.
    - else: pt->error = -EINVAL (third distinct wq unsupported); return.
  - `poll = kmalloc_obj(*poll, GFP_ATOMIC)`. if !poll: pt->error = -ENOMEM; return.
  - wqe_private |= IO_WQE_F_DOUBLE.
  - `io_init_poll_iocb(poll, first->events)`.
  - if `!io_poll_double_prepare(req)`: kfree(poll); return (req already completing).
  - `*poll_ptr = poll`.
- else: req->flags |= REQ_F_SINGLE_POLL.
- pt->nr_entries++.
- poll->head = head.
- poll->wait.private = (void*) wqe_private.
- if `poll->events & EPOLLEXCLUSIVE`: `add_wait_queue_exclusive(head, &poll->wait)`.
- else: `add_wait_queue(head, &poll->wait)`.

REQ-10: `io_poll_double_prepare(req) -> bool`:
- `poll = io_poll_get_single(req)`.
- rcu_read_lock.
- head = `smp_load_acquire(&poll->head)`.
- if head:
  - `spin_lock_irq(&head->lock)`.
  - req->flags |= REQ_F_DOUBLE_POLL.
  - if opcode == IORING_OP_POLL_ADD: req->flags |= REQ_F_ASYNC_DATA.
  - `spin_unlock_irq`.
- rcu_read_unlock.
- return !!head.

REQ-11: `__io_arm_poll_handler(req, poll, ipt, mask, issue_flags) -> i32`:
- `INIT_HLIST_NODE(&req->hash_node)`.
- `io_init_poll_iocb(poll, mask)`.
- poll->file = req->file.
- req->apoll_events = poll->events.
- ipt: `_key = mask`, req = req, error = 0, nr_entries = 0.
- ipt->owning = `issue_flags & IO_URING_F_UNLOCKED`.
- `atomic_set(&req->poll_refs, (int)ipt->owning)` ≡ start with 1 ref if io-wq, 0 if task ctx.
- if `poll->events & EPOLLEXCLUSIVE`: req->flags |= REQ_F_POLL_NO_LAZY.
- mask = `vfs_poll(file, &ipt->pt) & poll->events`.
- if ipt->error ∨ !nr_entries:
  - `io_poll_remove_entries(req)`.
  - if !`io_poll_can_finish_inline(req, ipt)`:
    - `io_poll_mark_cancelled(req)`; return 0.
  - else if `mask ∧ (poll->events & EPOLLET)`:
    - ipt->result_mask = mask; return 1 (inline-complete).
  - return `ipt->error ?: -EINVAL`.
- if `mask ∧ ((poll->events & (EPOLLET|EPOLLONESHOT)) == (EPOLLET|EPOLLONESHOT))`:
  - if !can_finish_inline: `io_poll_add_hash(req, issue_flags)`; return 0.
  - `io_poll_remove_entries(req)`; ipt->result_mask = mask; return 1.
- `io_poll_add_hash(req, issue_flags)`.
- if `mask ∧ (poll->events & EPOLLET) ∧ can_finish_inline`:
  - `__io_poll_execute(req, mask)`; return 0.
- `io_napi_add(req)`.
- if ipt->owning:
  - if `atomic_cmpxchg(&req->poll_refs, 1, 0) != 1`: `__io_poll_execute(req, 0)` (state changed; queue tw).
- return 0.

REQ-12: `io_poll_can_finish_inline(req, pt) -> bool`:
- return `pt->owning ∨ io_poll_get_ownership(req)`.

REQ-13: `io_poll_add_hash(req, issue_flags)`:
- `io_ring_submit_lock(ctx, issue_flags)`; `io_poll_req_insert(req)`; `io_ring_submit_unlock(...)`.

REQ-14: `io_poll_req_insert(req)`:
- lockdep_assert_held(&ctx->uring_lock).
- index = `hash_long(req->cqe.user_data, ctx->cancel_table.hash_bits)`.
- `hlist_add_head(&req->hash_node, &table->hbs[index].list)`.

REQ-15: `io_poll_wake(wait, mode, sync, key) -> i32` (the wait_func):
- req = `wqe_to_req(wait)`.
- poll = `container_of(wait, struct io_poll, wait)`.
- mask = `key_to_poll(key)`.
- if `mask & POLLFREE`: return `io_pollfree_wake(req, poll)`.
- if `mask ∧ !(mask & (poll->events & ~IO_ASYNC_POLL_COMMON))`: return 0 (not interested).
- if `io_poll_get_ownership(req)`:
  - if `mask & EPOLL_URING_WAKE` (self-wakeup; circular avoidance):
    - poll->events |= EPOLLONESHOT.
    - req->apoll_events |= EPOLLONESHOT.
  - if `mask ∧ (poll->events & EPOLLONESHOT)`:
    - `io_poll_remove_waitq(poll)`.
    - if `wqe_is_double(wait)`: req->flags &= ~REQ_F_DOUBLE_POLL.
    - else: req->flags &= ~REQ_F_SINGLE_POLL.
  - `__io_poll_execute(req, mask)`.
- return 1.

REQ-16: `__io_poll_execute(req, mask)`:
- `io_req_set_res(req, mask, 0)`.
- req->io_task_work.func = `io_poll_task_func`.
- flags = `(req->flags & REQ_F_POLL_NO_LAZY) ? 0 : IOU_F_TWQ_LAZY_WAKE`.
- `__io_req_task_work_add(req, flags)`.

REQ-17: `io_poll_execute(req, res)`:
- if `io_poll_get_ownership(req)`: `__io_poll_execute(req, res)`.

REQ-18: `io_poll_check_events(req, tw) -> i32` (tw loop):
- if `tw.cancel`: return -ECANCELED.
- do:
  - v = `atomic_read(&req->poll_refs)`.
  - if `v != 1`:
    - WARN if `!(v & IO_POLL_REF_MASK)`; return IOU_POLL_NO_ACTION.
    - if `v & IO_POLL_CANCEL_FLAG`: return -ECANCELED.
    - if `(v & IO_POLL_REF_MASK) != 1`: req->cqe.res = 0 (stale mask; redo).
    - if `v & IO_POLL_RETRY_FLAG`:
      - req->cqe.res = 0.
      - `atomic_andnot(IO_POLL_RETRY_FLAG, refs)`.
      - v &= ~IO_POLL_RETRY_FLAG.
    - v &= IO_POLL_REF_MASK.
  - if `!req->cqe.res`:
    - events = req->apoll_events.
    - `pt = { _key = events }`; `req->cqe.res = vfs_poll(req->file, &pt) & events`.
    - if !res: if !(events & EPOLLONESHOT): continue (spurious; loop). else: return IOU_POLL_REISSUE.
  - if `events & EPOLLONESHOT`: return IOU_POLL_DONE.
  - if `!(req->flags & REQ_F_APOLL_MULTISHOT)` (explicit multishot poll):
    - mask = `mangle_poll(res & events)`.
    - if `!io_req_post_cqe(req, mask, IORING_CQE_F_MORE)`:
      - `io_req_set_res(req, mask, 0)`; return IOU_POLL_REMOVE_POLL_USE_RES.
  - else (apoll multishot):
    - if `(res & (POLLHUP | POLLRDHUP)) ∧ v != 1`: v-- (extra loop on HUP).
    - ret = `io_poll_issue(req, tw)`.
    - if ret == IOU_COMPLETE: return IOU_POLL_REMOVE_POLL_USE_RES.
    - else if ret == IOU_REQUEUE: return IOU_POLL_REQUEUE.
    - if ret != IOU_RETRY ∧ ret < 0: return ret.
  - req->cqe.res = 0.
- while `atomic_sub_return(v, refs) & IO_POLL_REF_MASK` (retry while more refs in).
- `io_napi_add(req)`.
- return IOU_POLL_NO_ACTION.

REQ-19: `io_poll_task_func(tw_req, tw)`:
- ret = `io_poll_check_events(req, tw)`.
- if ret == IOU_POLL_NO_ACTION: return.
- if ret == IOU_POLL_REQUEUE: `__io_poll_execute(req, 0)`; return.
- `io_poll_remove_entries(req)`; `hash_del(&req->hash_node)`.
- if opcode == IORING_OP_POLL_ADD:
  - if ret == IOU_POLL_DONE: req->cqe.res = `mangle_poll(res & poll->events)`.
  - else if ret == IOU_POLL_REISSUE: `io_req_task_submit(tw_req, tw)`; return.
  - else if ret != IOU_POLL_REMOVE_POLL_USE_RES: req->cqe.res = ret; req_set_fail(req).
  - `io_req_set_res(req, cqe.res, 0)`; `io_req_task_complete(tw_req, tw)`.
- else (apoll):
  - `io_tw_lock(ctx, tw)`.
  - if ret == IOU_POLL_REMOVE_POLL_USE_RES: `io_req_task_complete(tw_req, tw)`.
  - else if ret == IOU_POLL_DONE ∨ IOU_POLL_REISSUE: `io_req_task_submit(tw_req, tw)`.
  - else: `io_req_defer_failed(req, ret)`.

REQ-20: `io_poll_remove_entries(req)`:
- if `!(flags & (REQ_F_SINGLE_POLL | REQ_F_DOUBLE_POLL))`: return (fastpath; nothing armed).
- rcu_read_lock (RCU-protects waitqueue head against POLLFREE-delayed free).
- if SINGLE_POLL: `io_poll_remove_entry(io_poll_get_single(req))`.
- if DOUBLE_POLL: `io_poll_remove_entry(io_poll_get_double(req))`.
- rcu_read_unlock.

REQ-21: `io_poll_remove_entry(poll)`:
- head = `smp_load_acquire(&poll->head)`.
- if head: `spin_lock_irq(&head->lock)`; `io_poll_remove_waitq(poll)`; `spin_unlock_irq`.

REQ-22: `io_poll_remove_waitq(poll)`:
- `list_del_init(&poll->wait.entry)`.
- `smp_store_release(&poll->head, NULL)` — last (after store, req may be freed).

REQ-23: `io_pollfree_wake(req, poll) -> i32` (file waitqueue is being freed):
- `io_poll_mark_cancelled(req)`.
- `io_poll_execute(req, 0)`.
- `io_poll_remove_waitq(poll)`.
- return 1.

REQ-24: `io_arm_poll_handler(req, issue_flags) -> i32` (turn non-poll-op into apoll):
- def = `&io_issue_defs[req->opcode]`.
- mask = POLLPRI | POLLERR.
- if `!def->pollin ∧ !def->pollout`: return IO_APOLL_ABORTED.
- if `!io_file_can_poll(req)`: return IO_APOLL_ABORTED.
- if def->pollin:
  - mask |= EPOLLIN | EPOLLRDNORM.
  - if `req->flags & REQ_F_CLEAR_POLLIN`: mask &= ~EPOLLIN (recvmsg MSG_ERRQUEUE).
- else: mask |= EPOLLOUT | EPOLLWRNORM.
- if def->poll_exclusive: mask |= EPOLLEXCLUSIVE.
- return `io_arm_apoll(req, issue_flags, mask)`.

REQ-25: `io_arm_apoll(req, issue_flags, mask) -> i32`:
- mask |= EPOLLET (apoll is always edge-triggered).
- if `!io_file_can_poll(req)`: return IO_APOLL_ABORTED.
- if `!(flags & REQ_F_APOLL_MULTISHOT)`: mask |= EPOLLONESHOT.
- apoll = `io_req_alloc_apoll(req, issue_flags)`. if !apoll: return IO_APOLL_ABORTED.
- flags &= ~(REQ_F_SINGLE_POLL | REQ_F_DOUBLE_POLL); flags |= REQ_F_POLLED.
- ipt.pt._qproc = `io_async_queue_proc`.
- ret = `__io_arm_poll_handler(req, &apoll->poll, &ipt, mask, issue_flags)`.
- if ret: return `ret > 0 ? IO_APOLL_READY : IO_APOLL_ABORTED`.
- `trace_io_uring_poll_arm`. return IO_APOLL_OK.

REQ-26: `io_req_alloc_apoll(req, issue_flags) -> *async_poll`:
- if `flags & REQ_F_POLLED`: apoll = req->apoll; kfree(apoll->double_poll).
- else:
  - if `!(issue_flags & IO_URING_F_UNLOCKED)`: apoll = `io_cache_alloc(&ctx->apoll_cache, GFP_ATOMIC)`.
  - else: apoll = `kmalloc_obj(*apoll, GFP_ATOMIC)`.
  - if !apoll: return NULL.
  - apoll->poll.retries = APOLL_MAX_RETRY (128).
- apoll->double_poll = NULL.
- req->apoll = apoll.
- if `--apoll->poll.retries == 0`: return NULL (retry budget exhausted).
- return apoll.

REQ-27: `io_poll_parse_events(sqe, flags) -> __poll_t`:
- events = `READ_ONCE(sqe->poll32_events)`. `swahw32` if big-endian.
- if `!(flags & IORING_POLL_ADD_MULTI)`: events |= EPOLLONESHOT.
- if `!(flags & IORING_POLL_ADD_LEVEL)`: events |= EPOLLET (default edge).
- return `demangle_poll(events) | (events & (EPOLLEXCLUSIVE | EPOLLONESHOT | EPOLLET))`.

REQ-28: `io_poll_add_prep(req, sqe) -> i32`:
- if `sqe->buf_index ∨ sqe->off ∨ sqe->addr`: -EINVAL.
- flags = `READ_ONCE(sqe->len)`.
- if `flags & ~IORING_POLL_ADD_MULTI`: -EINVAL.
- if `(flags & IORING_POLL_ADD_MULTI) ∧ (req->flags & REQ_F_CQE_SKIP)`: -EINVAL.
- poll->events = `io_poll_parse_events(sqe, flags)`.

REQ-29: `io_poll_add(req, issue_flags) -> i32`:
- ipt.pt._qproc = `io_poll_queue_proc`.
- ret = `__io_arm_poll_handler(req, poll, &ipt, poll->events, issue_flags)`.
- if ret > 0: `io_req_set_res(req, ipt.result_mask, 0)`; return IOU_COMPLETE.
- return `ret ?: IOU_ISSUE_SKIP_COMPLETE`.

REQ-30: `io_poll_remove_prep(req, sqe) -> i32`:
- if `sqe->buf_index ∨ sqe->splice_fd_in`: -EINVAL.
- flags = `READ_ONCE(sqe->len)`.
- valid = `IORING_POLL_UPDATE_EVENTS | IORING_POLL_UPDATE_USER_DATA | IORING_POLL_ADD_MULTI`.
- if `flags & ~valid`: -EINVAL.
- if flags == IORING_POLL_ADD_MULTI: -EINVAL (meaningless without update).
- upd->old_user_data = `READ_ONCE(sqe->addr)`.
- upd->update_events = flags & IORING_POLL_UPDATE_EVENTS.
- upd->update_user_data = flags & IORING_POLL_UPDATE_USER_DATA.
- upd->new_user_data = `READ_ONCE(sqe->off)`.
- if `!upd->update_user_data ∧ upd->new_user_data`: -EINVAL.
- if upd->update_events: upd->events = `io_poll_parse_events(sqe, flags)`.
- else if sqe->poll32_events: -EINVAL.

REQ-31: `io_poll_remove(req, issue_flags) -> i32`:
- `io_ring_submit_lock(ctx, issue_flags)`.
- preq = `io_poll_find(ctx, true, &cd)` (poll_only).
- ret2 = `io_poll_disarm(preq)`.
- if ret2: ret = ret2; goto out.
- WARN if `preq->opcode != IORING_OP_POLL_ADD`: ret = -EFAULT; goto out.
- if update_events ∨ update_user_data:
  - if update_events: poll->events = (poll->events & ~0xffff) | (upd->events & 0xffff) | IO_POLL_UNMASK.
  - if update_user_data: preq->cqe.user_data = upd->new_user_data.
  - ret2 = `io_poll_add(preq, issue_flags & ~IO_URING_F_UNLOCKED)`.
  - if ret2 == IOU_ISSUE_SKIP_COMPLETE: goto out (re-armed; don't complete).
  - else if ret2 == IOU_COMPLETE: goto complete (inline-complete; complete preq).
- `io_req_set_res(preq, -ECANCELED, 0)`.
- complete: if preq->cqe.res < 0: req_set_fail(preq). preq->io_task_work.func = `io_req_task_complete`. `io_req_task_work_add(preq)`.
- out: `io_ring_submit_unlock`.
- if ret < 0: req_set_fail(req); return ret.
- `io_req_set_res(req, ret, 0)`; return IOU_COMPLETE.

REQ-32: `io_poll_disarm(req) -> i32`:
- if !req: return -ENOENT.
- if `!io_poll_get_ownership(req)`: return -EALREADY.
- `io_poll_remove_entries(req)`.
- `hash_del(&req->hash_node)`.

REQ-33: `io_poll_find(ctx, poll_only, cd) -> *req`:
- index = `hash_long(cd->data, hash_bits)`.
- iterate hb->list; match cd->data == req->cqe.user_data.
- if poll_only ∧ opcode != IORING_OP_POLL_ADD: skip.
- if `cd->flags & IORING_ASYNC_CANCEL_ALL`: check `io_cancel_match_sequence`.

REQ-34: `io_poll_file_find(ctx, cd) -> *req`:
- linear over all buckets; `io_cancel_req_match(req, cd)`.

REQ-35: `io_poll_cancel(ctx, cd, issue_flags) -> i32`:
- `io_ring_submit_lock`; `__io_poll_cancel`; unlock.

REQ-36: `__io_poll_cancel(ctx, cd) -> i32`:
- if `cd->flags & (IORING_ASYNC_CANCEL_FD | _OP | _ANY)`: `io_poll_file_find`.
- else: `io_poll_find(ctx, false, cd)`.
- if req: `io_poll_cancel_req(req)`; return 0.
- else: -ENOENT.

REQ-37: `io_poll_cancel_req(req)`:
- `io_poll_mark_cancelled(req)` ≡ `atomic_or(IO_POLL_CANCEL_FLAG, refs)`.
- `io_poll_execute(req, 0)`.

REQ-38: `io_poll_remove_all(ctx, tctx, cancel_all) -> bool`:
- lockdep uring_lock.
- for each bucket: `hlist_for_each_entry_safe(req, tmp, hb->list, hash_node)`:
  - if `io_match_task_safe(req, tctx, cancel_all)`: `hlist_del_init(hash_node)`; `io_poll_cancel_req(req)`; found = true.

REQ-39: `io_poll_get_single(req) / _double(req)`:
- single = (opcode == POLL_ADD) ? `io_kiocb_to_cmd(req, struct io_poll)` : `&req->apoll->poll`.
- double = (opcode == POLL_ADD) ? `req->async_data` : `req->apoll->double_poll`.

REQ-40: `wqe_to_req(wqe)` / `wqe_is_double(wqe)`:
- priv = (unsigned long)wqe->private.
- req = (priv & ~IO_WQE_F_DOUBLE).
- double = priv & IO_WQE_F_DOUBLE.

REQ-41: Multishot (`IORING_POLL_ADD_MULTI`):
- Don't set EPOLLONESHOT in parse_events.
- After each delivery, `io_req_post_cqe(... CQE_F_MORE)` posts a fresh CQE; req stays armed.
- If post fails (ring overflow): set IOU_POLL_REMOVE_POLL_USE_RES to convert to one-shot remove.
- self-wakeup (`EPOLL_URING_WAKE`) ⟹ disable multishot to break circular CQ-post-triggers-wake loop.

REQ-42: Level (`IORING_POLL_ADD_LEVEL`):
- If LEVEL flag clear: events |= EPOLLET (default ≡ edge).
- If LEVEL set: edge bit not set ⟹ level-trigger: vfs_poll re-checks each tw.

REQ-43: Exclusive (`EPOLLEXCLUSIVE`):
- For explicit POLL_ADD: passed in poll32_events.
- For apoll: `def->poll_exclusive` ⟹ mask |= EPOLLEXCLUSIVE.
- `add_wait_queue_exclusive(head, &wait)` ≡ only one exclusive waker awakened per wakeup.
- forces REQ_F_POLL_NO_LAZY (no batched tw wake).

REQ-44: Static configuration:
- `IO_POLL_REF_BIAS = 128`.
- `IO_POLL_REF_MASK = GENMASK(29, 0)`.
- `IO_POLL_RETRY_FLAG = BIT(30)`.
- `IO_POLL_CANCEL_FLAG = BIT(31)`.
- `IO_WQE_F_DOUBLE = 1`.
- `IO_ASYNC_POLL_COMMON = EPOLLONESHOT | EPOLLPRI`.
- `IO_POLL_UNMASK = EPOLLERR | EPOLLHUP | EPOLLNVAL | EPOLLRDHUP`.
- `APOLL_MAX_RETRY = 128`.
- `IO_POLL_ALLOC_CACHE_MAX = 32`.

## Acceptance Criteria

- [ ] AC-1: `IORING_OP_POLL_ADD` with mask EPOLLIN on a readable pipe ⟹ CQE posted with `mangle_poll(POLLIN)`; req removed from cancel hash.
- [ ] AC-2: `IORING_OP_POLL_ADD` with `IORING_POLL_ADD_MULTI` ⟹ multiple CQEs with `IORING_CQE_F_MORE` until cancel/remove; req remains hashed.
- [ ] AC-3: `IORING_OP_POLL_ADD` defaults to edge-trigger (EPOLLET set); `IORING_POLL_ADD_LEVEL` flag in `sqe->len` suppresses EPOLLET so vfs_poll re-checks in tw loop.
- [ ] AC-4: `IORING_OP_POLL_REMOVE` with `IORING_POLL_UPDATE_EVENTS` ⟹ disarm preq, change events, re-arm; original req completes with CQE 0 (update succeeded).
- [ ] AC-5: `IORING_OP_POLL_REMOVE` with `IORING_POLL_UPDATE_USER_DATA` rewrites preq->cqe.user_data so future cancels target new id.
- [ ] AC-6: `IORING_OP_POLL_REMOVE` of nonexistent id ⟹ CQE res = -ENOENT.
- [ ] AC-7: `io_arm_poll_handler` on a recv that would block ⟹ apoll allocated, EPOLLIN|EPOLLRDNORM armed on socket waitqueue, original op deferred; on POLLIN wakeup, recv reissued from tw.
- [ ] AC-8: File with two waitqueues (e.g. tty read & write) ⟹ `__io_queue_proc` allocates `double_poll`, both `add_wait_queue` calls succeed, `REQ_F_DOUBLE_POLL` set.
- [ ] AC-9: File with three+ distinct waitqueues ⟹ pt->error = -EINVAL; arming aborts.
- [ ] AC-10: `def->poll_exclusive` opcode (e.g. accept) ⟹ `EPOLLEXCLUSIVE` set, `add_wait_queue_exclusive` called, `REQ_F_POLL_NO_LAZY` set.
- [ ] AC-11: `POLLFREE` wakeup ⟹ `io_pollfree_wake` ⟹ mark cancelled + execute tw + remove waitq; subsequent tw returns -ECANCELED.
- [ ] AC-12: Concurrent wakeups (≥ 2) ⟹ second wakeup increments `poll_refs` past 1; tw loop sees stale res, resets, redoes `vfs_poll`.
- [ ] AC-13: Concurrent wakeups exceeding `IO_POLL_REF_BIAS` (128) ⟹ slowpath sets `IO_POLL_RETRY_FLAG`; tw clears it and retries.
- [ ] AC-14: `io_poll_cancel(ctx, cd)` with user_data hit ⟹ `IO_POLL_CANCEL_FLAG` set, tw runs and returns -ECANCELED.
- [ ] AC-15: `io_poll_remove_all(ctx, tctx, true)` over a ctx ⟹ every poll req across all buckets is cancelled.
- [ ] AC-16: Apoll retry budget `APOLL_MAX_RETRY = 128`: 128th retry returns NULL ⟹ apoll arm returns IO_APOLL_ABORTED.
- [ ] AC-17: `io_poll_check_events` for multishot apoll with `POLLHUP | POLLRDHUP` and v != 1 ⟹ extra loop iteration (v--), ensuring HUP delivered before exit.

## Architecture

```
struct IoPoll {
  file: *File,
  head: *WaitQueueHead,             // NULL after remove
  events: u32,                      // __poll_t with EPOLLET/EPOLLONESHOT/EPOLLEXCLUSIVE high bits
  retries: i32,
  wait: WaitQueueEntry,             // priv encodes req | IO_WQE_F_DOUBLE
}

struct AsyncPoll {
  poll: IoPoll,
  double_poll: Option<*IoPoll>,
}

struct IoPollUpdate {
  file: *File,
  old_user_data: u64,
  new_user_data: u64,
  events: u32,
  update_events: bool,
  update_user_data: bool,
}

struct IoPollTable {
  pt: PollTableStruct,              // ._qproc set per arm path
  req: *IoKiocb,
  nr_entries: i32,
  error: i32,
  owning: bool,
  result_mask: u32,
}

const IO_POLL_REF_BIAS:    i32 = 128;
const IO_POLL_REF_MASK:    u32 = 0x3FFF_FFFF;       // GENMASK(29, 0)
const IO_POLL_RETRY_FLAG:  u32 = 1 << 30;
const IO_POLL_CANCEL_FLAG: u32 = 1 << 31;
const IO_WQE_F_DOUBLE:     usize = 1;
const IO_ASYNC_POLL_COMMON: u32 = EPOLLONESHOT | EPOLLPRI;
const IO_POLL_UNMASK:      u32 = EPOLLERR | EPOLLHUP | EPOLLNVAL | EPOLLRDHUP;
const APOLL_MAX_RETRY:     i32 = 128;
```

`Poll::arm_handler(req, poll, ipt, mask, issue_flags) -> i32` [`__io_arm_poll_handler`]:
1. INIT_HLIST_NODE(req.hash_node).
2. Poll::init_iocb(poll, mask).
3. poll.file = req.file; req.apoll_events = poll.events.
4. ipt: _key = mask; req = req; error = 0; nr_entries = 0.
5. ipt.owning = issue_flags & IO_URING_F_UNLOCKED.
6. atomic_set(req.poll_refs, ipt.owning ? 1 : 0).
7. if poll.events & EPOLLEXCLUSIVE: req.flags |= REQ_F_POLL_NO_LAZY.
8. mask = vfs_poll(req.file, &ipt.pt) & poll.events.
9. if ipt.error ∨ !ipt.nr_entries:
   - Poll::remove_entries(req).
   - if !Poll::can_finish_inline(req, ipt):
     - Poll::mark_cancelled(req); return 0.
   - if mask ∧ (poll.events & EPOLLET):
     - ipt.result_mask = mask; return 1.
   - return ipt.error ?: -EINVAL.
10. if mask ∧ ((poll.events & (EPOLLET|EPOLLONESHOT)) == (EPOLLET|EPOLLONESHOT)):
    - if !Poll::can_finish_inline(req, ipt): Poll::add_hash(req, issue_flags); return 0.
    - Poll::remove_entries(req); ipt.result_mask = mask; return 1.
11. Poll::add_hash(req, issue_flags).
12. if mask ∧ (poll.events & EPOLLET) ∧ Poll::can_finish_inline: Poll::execute_inner(req, mask); return 0.
13. io_napi_add(req).
14. if ipt.owning ∧ atomic_cmpxchg(refs, 1, 0) != 1: Poll::execute_inner(req, 0).
15. return 0.

`Poll::queue_proc_inner(poll, pt, head, poll_ptr)` [`__io_queue_proc`]:
1. wqe_private = (unsigned long) pt.req.
2. if pt.nr_entries != 0:
   - first = poll.
   - if first.head == head: return.
   - if *poll_ptr != NULL:
     - if (*poll_ptr).head == head: return.
     - pt.error = -EINVAL; return.
   - poll = kmalloc_obj(*poll, GFP_ATOMIC).
   - if !poll: pt.error = -ENOMEM; return.
   - wqe_private |= IO_WQE_F_DOUBLE.
   - Poll::init_iocb(poll, first.events).
   - if !Poll::double_prepare(req): kfree(poll); return.
   - *poll_ptr = poll.
3. else: req.flags |= REQ_F_SINGLE_POLL.
4. pt.nr_entries++.
5. poll.head = head; poll.wait.private = (void*) wqe_private.
6. if poll.events & EPOLLEXCLUSIVE: add_wait_queue_exclusive(head, &poll.wait).
7. else: add_wait_queue(head, &poll.wait).

`Poll::wake(wait, mode, sync, key) -> i32`:
1. req = Poll::wqe_to_req(wait); poll = container_of(wait, IoPoll, wait); mask = key_to_poll(key).
2. if mask & POLLFREE: return Poll::pollfree_wake(req, poll).
3. if mask ∧ !(mask & (poll.events & ~IO_ASYNC_POLL_COMMON)): return 0.
4. if Poll::get_ownership(req):
   - if mask & EPOLL_URING_WAKE: poll.events |= EPOLLONESHOT; req.apoll_events |= EPOLLONESHOT.
   - if mask ∧ (poll.events & EPOLLONESHOT):
     - Poll::remove_waitq(poll).
     - if double: flags &= ~REQ_F_DOUBLE_POLL.
     - else: flags &= ~REQ_F_SINGLE_POLL.
   - Poll::execute_inner(req, mask).
5. return 1.

`Poll::check_events(req, tw) -> i32`:
1. if tw.cancel: return -ECANCELED.
2. do:
   - v = atomic_read(refs).
   - if v != 1:
     - WARN+IOU_POLL_NO_ACTION if !(v & MASK).
     - if v & CANCEL: return -ECANCELED.
     - if (v & MASK) != 1: req.cqe.res = 0.
     - if v & RETRY: req.cqe.res = 0; atomic_andnot(RETRY, refs); v &= ~RETRY.
     - v &= MASK.
   - if !req.cqe.res:
     - events = req.apoll_events; pt = { _key = events }.
     - req.cqe.res = vfs_poll(req.file, &pt) & events.
     - if !res: if !(events & EPOLLONESHOT): continue. else: return IOU_POLL_REISSUE.
   - if events & EPOLLONESHOT: return IOU_POLL_DONE.
   - if !(flags & REQ_F_APOLL_MULTISHOT):
     - mask = mangle_poll(res & events).
     - if !io_req_post_cqe(req, mask, CQE_F_MORE): io_req_set_res(req, mask, 0); return IOU_POLL_REMOVE_POLL_USE_RES.
   - else:
     - if (res & (POLLHUP|POLLRDHUP)) ∧ v != 1: v--.
     - ret = io_poll_issue(req, tw).
     - if ret == IOU_COMPLETE: return IOU_POLL_REMOVE_POLL_USE_RES.
     - else if ret == IOU_REQUEUE: return IOU_POLL_REQUEUE.
     - if ret != IOU_RETRY ∧ ret < 0: return ret.
   - req.cqe.res = 0.
3. while atomic_sub_return(v, refs) & MASK.
4. io_napi_add(req).
5. return IOU_POLL_NO_ACTION.

`Poll::task_func(tw_req, tw)`:
1. ret = Poll::check_events(req, tw).
2. match ret:
   - IOU_POLL_NO_ACTION: return.
   - IOU_POLL_REQUEUE: Poll::execute_inner(req, 0); return.
3. Poll::remove_entries(req); hash_del(req.hash_node).
4. if opcode == IORING_OP_POLL_ADD:
   - IOU_POLL_DONE ⟹ req.cqe.res = mangle_poll(res & poll.events).
   - IOU_POLL_REISSUE ⟹ io_req_task_submit; return.
   - else if ret != IOU_POLL_REMOVE_POLL_USE_RES: req.cqe.res = ret; req_set_fail.
   - io_req_set_res; io_req_task_complete.
5. else (apoll):
   - io_tw_lock.
   - IOU_POLL_REMOVE_POLL_USE_RES ⟹ io_req_task_complete.
   - IOU_POLL_DONE ∨ IOU_POLL_REISSUE ⟹ io_req_task_submit.
   - else io_req_defer_failed.

`Poll::add(req, issue_flags) -> i32`:
1. ipt.pt._qproc = `io_poll_queue_proc`.
2. ret = Poll::arm_handler(req, poll, &ipt, poll.events, issue_flags).
3. if ret > 0: io_req_set_res(req, ipt.result_mask, 0); return IOU_COMPLETE.
4. return ret ?: IOU_ISSUE_SKIP_COMPLETE.

`Poll::remove(req, issue_flags) -> i32`:
1. io_ring_submit_lock.
2. preq = Poll::find_by_user_data(ctx, true, &cd).
3. ret2 = Poll::disarm(preq); if ret2: ret = ret2; goto out.
4. WARN if preq.opcode != IORING_OP_POLL_ADD: ret = -EFAULT; goto out.
5. if update_events ∨ update_user_data:
   - if update_events: poll.events = (poll.events & ~0xffff) | (upd.events & 0xffff) | IO_POLL_UNMASK.
   - if update_user_data: preq.cqe.user_data = upd.new_user_data.
   - ret2 = Poll::add(preq, issue_flags & ~IO_URING_F_UNLOCKED).
   - if ret2 == IOU_ISSUE_SKIP_COMPLETE: goto out.
   - else if ret2 == IOU_COMPLETE: goto complete.
6. io_req_set_res(preq, -ECANCELED, 0).
7. complete: if preq.cqe.res < 0: req_set_fail(preq). preq.io_task_work.func = io_req_task_complete; io_req_task_work_add(preq).
8. out: io_ring_submit_unlock.
9. if ret < 0: req_set_fail(req); return ret.
10. io_req_set_res(req, ret, 0); return IOU_COMPLETE.

`Poll::arm_for_async(req, issue_flags) -> i32` [`io_arm_poll_handler`]:
1. def = &io_issue_defs[req.opcode].
2. mask = POLLPRI | POLLERR.
3. if !def.pollin ∧ !def.pollout: return IO_APOLL_ABORTED.
4. if !io_file_can_poll(req): return IO_APOLL_ABORTED.
5. if def.pollin:
   - mask |= EPOLLIN | EPOLLRDNORM.
   - if flags & REQ_F_CLEAR_POLLIN: mask &= ~EPOLLIN.
6. else: mask |= EPOLLOUT | EPOLLWRNORM.
7. if def.poll_exclusive: mask |= EPOLLEXCLUSIVE.
8. return Poll::arm_apoll(req, issue_flags, mask).

`Poll::arm_apoll(req, issue_flags, mask) -> i32`:
1. mask |= EPOLLET.
2. if !io_file_can_poll(req): IO_APOLL_ABORTED.
3. if !(flags & REQ_F_APOLL_MULTISHOT): mask |= EPOLLONESHOT.
4. apoll = Poll::alloc_apoll(req, issue_flags). if !apoll: IO_APOLL_ABORTED.
5. flags &= ~(REQ_F_SINGLE_POLL | REQ_F_DOUBLE_POLL); flags |= REQ_F_POLLED.
6. ipt.pt._qproc = `io_async_queue_proc`.
7. ret = Poll::arm_handler(req, &apoll.poll, &ipt, mask, issue_flags).
8. if ret: return ret > 0 ? IO_APOLL_READY : IO_APOLL_ABORTED.
9. trace_io_uring_poll_arm. return IO_APOLL_OK.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `poll_refs_state_machine` | INVARIANT | per-req: bits 30/31 set only via documented paths (RETRY=slowpath; CANCEL=cancel_req/pollfree). |
| `ownership_unique` | INVARIANT | per-req: only one tw running at a time (ownership is acquired before `__io_poll_execute`). |
| `double_poll_at_most_two` | INVARIANT | per-`__io_queue_proc`: a 3rd distinct waitqueue ⟹ pt.error = -EINVAL. |
| `wait_priv_encoding_disjoint` | INVARIANT | per-wqe.private: IO_WQE_F_DOUBLE bit disjoint from req pointer (req is at least 2-aligned ⟹ low bit free). |
| `poll_head_release_last` | INVARIANT | per-`io_poll_remove_waitq`: `list_del_init` before `smp_store_release(head, NULL)`. |
| `pollfree_marks_and_kicks` | INVARIANT | per-POLLFREE: mark_cancelled + execute + remove_waitq, in that order. |
| `multishot_no_circular_wake` | INVARIANT | per-EPOLL_URING_WAKE: forces EPOLLONESHOT after first delivery on self-wakeup. |
| `exclusive_no_lazy` | INVARIANT | per-EPOLLEXCLUSIVE arm: REQ_F_POLL_NO_LAZY set. |
| `level_default_is_edge` | INVARIANT | per-parse_events: !IORING_POLL_ADD_LEVEL ⟹ events |= EPOLLET. |
| `multishot_default_is_oneshot` | INVARIANT | per-parse_events: !IORING_POLL_ADD_MULTI ⟹ events |= EPOLLONESHOT. |
| `update_must_have_update_user_data_to_change_id` | INVARIANT | per-remove_prep: !update_user_data ∧ new_user_data ≠ 0 ⟹ -EINVAL. |
| `apoll_retries_bounded` | INVARIANT | per-arm_apoll: apoll.retries decrements; 0 ⟹ abort. |

### Layer 2: TLA+

`io_uring/poll-state.tla`:
- Vars: per-req-state ∈ {Submitted, Armed-Single, Armed-Double, Wakeup-Pending, Tw-Running, Done, Cancelled}; per-poll_refs ∈ {0..N} × {RETRY?} × {CANCEL?}.
- Actions: arm, wake (single / double / N), tw_run, cancel, pollfree, update.
- Properties:
  - `mutex_tw_ownership` — at most one tw running per req at any state.
  - `every_armed_eventually_deliver_or_cancel` — armed req eventually delivers a CQE or is cancelled.
  - `multishot_yields_at_least_one_cqe_per_wakeup` — wakeup ⟹ ≥1 CQE posted before next wakeup (modulo cancel).
  - `cancel_terminates_tw_with_ecanceled` — set CANCEL_FLAG ⟹ tw return -ECANCELED.
  - `pollfree_no_uaf` — POLLFREE wakeup before head free ⟹ head=NULL set before req freed.
  - `exclusive_wakeup_single` — EPOLLEXCLUSIVE ⟹ at most one exclusive waiter awakened per wakeup.

`io_uring/poll-update.tla`:
- Actions: POLL_REMOVE update variants (cancel-only, events-only, user_data-only, both).
- Properties:
  - `update_re_arms_with_new_events_xor_cancels` — exactly one outcome.
  - `update_user_data_visible_to_subsequent_cancel` — re-arming with new id ⟹ next cancel-by-id hits new id.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Poll::add_prep` post: poll.events ∈ valid mask; flags = sqe.len subset of POLL_ADD_MULTI | `Poll::add_prep` |
| `Poll::remove_prep` post: upd populated; flags subset of valid | `Poll::remove_prep` |
| `Poll::arm_handler` post: `ret = 0 ⟹ tw will run` ∨ `ret = 1 ⟹ result_mask set` ∨ `ret < 0` | `Poll::arm_handler` |
| `Poll::queue_proc_inner` post: nr_entries ≤ 2 ∨ pt.error = -EINVAL | `Poll::queue_proc_inner` |
| `Poll::wake` post: ownership taken ⟹ tw enqueued; not taken ⟹ no state mutation | `Poll::wake` |
| `Poll::check_events` post: ret ∈ {IOU_POLL_DONE, _NO_ACTION, _REMOVE_POLL_USE_RES, _REISSUE, _REQUEUE, -E*} | `Poll::check_events` |
| `Poll::task_func` post: hash_del called iff ret != NO_ACTION ∧ ret != REQUEUE | `Poll::task_func` |
| `Poll::remove` post: preq disarmed ⟹ either re-armed (skip-complete) or completed (-ECANCELED or 0) | `Poll::remove` |
| `Poll::arm_apoll` post: apoll allocated ∨ IO_APOLL_ABORTED returned | `Poll::arm_apoll` |
| `Poll::pollfree_wake` post: cancel flag set ∧ head = NULL before return | `Poll::pollfree_wake` |

### Layer 4: Verus/Creusot functional

`Per-IORING_OP_POLL_*` semantic equivalence relation between upstream Linux io_uring poll and Rookery `Poll` per-`io_uring(7)`:
- `IORING_OP_POLL_ADD` ↔ CQE posted with `mangle_poll(events & poll_mask)` on first matching wakeup.
- `IORING_OP_POLL_ADD | IORING_POLL_ADD_MULTI` ↔ stream of CQEs with `CQE_F_MORE`; terminates on REMOVE/CANCEL or post-failure.
- `IORING_OP_POLL_REMOVE` w/o flags ↔ cancellation (preq CQE = -ECANCELED).
- `IORING_OP_POLL_REMOVE | IORING_POLL_UPDATE_EVENTS` ↔ disarm + re-arm with new events; original req CQE = 0.
- `IORING_OP_POLL_REMOVE | IORING_POLL_UPDATE_USER_DATA` ↔ preq.user_data rewritten; original req CQE = 0.
- async-poll (`io_arm_poll_handler`) ↔ orig op deferred until file readable / writable, then reissued; failure ⟹ -EAGAIN propagated.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

Poll reinforcement:

- **Per-`poll_refs` ownership protocol (low 30 bits ≡ refcount)** — defense against per-double-tw / per-concurrent-mutation.
- **Per-`IO_POLL_REF_BIAS = 128` slowpath threshold** — defense against per-refcount-wraparound on wakeup storm.
- **Per-`IO_POLL_RETRY_FLAG` retry signaling** — defense against per-lost-event when slowpath cannot increment.
- **Per-`IO_POLL_CANCEL_FLAG` cancel poison** — defense against per-cancel race with tw.
- **Per-`smp_store_release(head, NULL)` last in `remove_waitq`** — defense against per-UAF on req free after `list_del_init`.
- **Per-RCU-bracket in `remove_entries`** — defense against per-POLLFREE race (waitqueue freed concurrently).
- **Per-`add_wait_queue_exclusive` for EPOLLEXCLUSIVE** — defense against per-thundering-herd accept.
- **Per-`REQ_F_POLL_NO_LAZY` on EPOLLEXCLUSIVE** — defense against per-missed-event on lazy tw wake.
- **Per-three-waitqueue rejection (-EINVAL)** — defense against per-ungrowable double_poll alloc / unbounded poll_table.
- **Per-self-wakeup (EPOLL_URING_WAKE) ⟹ disable multishot** — defense against per-CQ-post-triggers-wake circular dependency.
- **Per-`APOLL_MAX_RETRY = 128`** — defense against per-livelock on always-EAGAIN file.
- **Per-`io_req_post_cqe` failure ⟹ remove-poll-use-res** — defense against per-ring-overflow infinite multishot.
- **Per-cancel-by-fd / -op / -any via `io_poll_file_find`** — defense against per-orphan poll across fork/exec.
- **Per-`io_poll_remove_all` on ctx teardown** — defense against per-leak on ring destroy.
- **Per-`apoll_cache` (`io_cache_alloc`) GFP_ATOMIC + `kmalloc_obj` fallback** — defense against per-allocation under uring_lock.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-checked copy on the `events` mask / `addr` user-data fields of `IORING_OP_POLL_ADD` and on `IORING_OP_POLL_REMOVE` match key.
- **PAX_KERNEXEC** — write-protects `io_poll_wake`, `io_poll_queue_proc`, and the `wait_queue_entry_t.func` slot referenced by every armed poll.
- **PAX_RANDKSTACK** — per-call stack-offset randomization on every `io_poll_add`, `io_poll_remove`, and `io_poll_task_func` entry.
- **PAX_REFCOUNT** — saturating trap on `req->refs` taken across arm/wake/complete (multishot may bounce many times) and on the wait-queue entry refcount.
- **PAX_MEMORY_SANITIZE** — zeroes `struct io_poll` and the cached `struct async_poll` on free via `apoll_cache`; scrubs `wait_queue_entry_t` on `list_del_init`.
- **PAX_UDEREF** — `events`, `addr`, and multishot tag fields are user-faulted at prep; defends against per-kernel-aliased poll mask injection.
- **PAX_RAP / kCFI** — forward-edge CFI on `wait_queue_entry_t.func` (`io_poll_wake`) dispatched from the file's poll head and on `f_op->poll` indirect call.
- **GRKERNSEC_HIDESYM** — strips poll helpers and apoll-cache pointers from kallsyms-leaking paths.
- **GRKERNSEC_DMESG** — restricts `WARN_ON_ONCE` and `pr_debug` poll traces (multishot self-wake, APOLL_MAX_RETRY) to CAP_SYSLOG.
- **Poll-armed completion REFCOUNT** — every armed poll bumps `req->refs`; the wake handler `io_poll_wake` decrements once on hit and once on disarm; saturating trap defends against per-double-wake refcount-underflow leading to use-after-free on `req`.
- **Multishot PAX_RAP** — multishot rearm path re-uses the same `wait_queue_entry_t.func` slot through the file's poll head; kCFI tag on `io_poll_wake` ensures an attacker who repoints a wake-queue entry to a same-type function still hits a tag mismatch at the next event.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- io_uring core / SQE / CQE / ring init (covered in `io_uring-core.md` Tier-3)
- io_uring submission / completion path (covered in `io_uring-core.md` Tier-3)
- async work-queue (`io-wq.c`) (covered separately if expanded)
- task-work (`io_uring/io_uring.c::io_req_task_work_add`) (covered in `io_uring-core.md`)
- napi busy-poll (`io_uring/napi.c`) (covered separately if expanded)
- io_uring cancel framework (`io_uring/cancel.c`) (covered separately if expanded)
- `vfs_poll` / `f_op->poll` per-driver implementations (covered in respective FS/socket Tier-3s)
- eventpoll-internals (`fs/eventpoll.c`) (covered in `fs/eventpoll.md` Tier-3)
- Implementation code
