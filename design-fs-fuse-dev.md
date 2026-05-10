---
title: "Tier-3: fs/fuse/dev.c — FUSE userspace-fs character device"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`/dev/fuse` is the character device through which a userspace filesystem daemon ("FUSE server") services VFS operations. The kernel side of a FUSE mount queues requests; the daemon `read(2)`s them, executes the operation, and `write(2)`s the reply. Per-`struct fuse_conn` ties a `/dev/fuse` open file (`struct fuse_dev` — at least one, more via `FUSE_DEV_IOC_CLONE`) to a kernel-side mount (`struct fuse_mount`). Per-request lifecycle: PENDING (on `fiq->pending`) → SENT (on `fpq->processing[hash]`, FR_SENT set) → FINISHED (`fuse_request_end`, waiter woken). Per-`FUSE_INIT` handshake: synchronous request issued by `fuse_send_init` at mount; negotiates protocol version, capability flags (FUSE_BIG_WRITES, FUSE_SPLICE_*, FUSE_ASYNC_READ, FUSE_DIO_RELAX, …), `max_read`, `max_write`, `max_background`, `congestion_threshold`. Per-background-queue: async requests gate at `fc->max_background` (default 12), drained into the pending queue by `flush_bg_queue`. Per-`fuse_copy_state`: bytewise copy engine for kiocb / splice paths over `fuse_in_header` + per-arg payloads, with optional folio-move-on-copy fast path (`fuse_try_move_folio`). Per-`FUSE_INTERRUPT`: synthesized request with opcode FUSE_INTERRUPT + reply-unique = original-unique | FUSE_INT_REQ_BIT, sent ahead of any other queued request. Per-`fuse_dev_release`: `/dev/fuse` close auto-aborts the connection on last-dev close. Critical for: container userspace filesystems (gocryptfs, sshfs, GlusterFS, virtio-fs), unprivileged FUSE mounts, libfuse compatibility.

This Tier-3 covers `fs/fuse/dev.c` (~2755 lines).

### Acceptance Criteria

- [ ] AC-1: Daemon read on empty pending blocks; O_NONBLOCK ⟹ -EAGAIN.
- [ ] AC-2: Daemon read returns at least one full fuse_in_header + per-arg payload, len == req->in.h.len.
- [ ] AC-3: Daemon read with nbytes < FUSE_MIN_READ_BUFFER returns -EINVAL.
- [ ] AC-4: FR_PENDING → FR_SENT transition observed via FR_SENT after copy; FR_ISREPLY=0 ⟹ never on processing list.
- [ ] AC-5: Daemon write with valid (unique, error) on a processed req: matched, FR_LOCKED set, fuse_request_end called, waiter woken.
- [ ] AC-6: Daemon write with unique=0 invokes fuse_notify dispatching on oh.error as enum fuse_notify_code.
- [ ] AC-7: Daemon write with unique unmatched ⟹ -ENOENT.
- [ ] AC-8: Daemon write with oh.error not in (-511..0] ⟹ -EINVAL.
- [ ] AC-9: Interrupt: signal during request_wait_answer ⟹ FR_INTERRUPTED, FUSE_INTERRUPT enqueued ahead of pending; daemon replies on |FUSE_INT_REQ_BIT unique.
- [ ] AC-10: -ENOSYS interrupt reply ⟹ fc->no_interrupt = 1 (no further interrupts queued).
- [ ] AC-11: FUSE_INIT exchange completes: fc->initialized set, max_background/max_write/flags reflect negotiated values.
- [ ] AC-12: Background queue: num_background == max_background ⟹ next fuse_request_queue_background blocks (or non-blocking caller stalls).
- [ ] AC-13: Background queue: completion of in-flight bg request wakes blocked submitter; num_background decremented.
- [ ] AC-14: Last /dev/fuse close ⟹ fuse_abort_conn drained all pending/processing with -ECONNABORTED.
- [ ] AC-15: poll: pending ⟹ EPOLLIN|EPOLLRDNORM; aborted ⟹ EPOLLERR; idle ⟹ EPOLLOUT|EPOLLWRNORM only.
- [ ] AC-16: fuse_dev_splice_read transfers request bytes into pipe; fuse_dev_splice_write consumes from pipe.
- [ ] AC-17: fuse_try_move_folio on aligned page-aligned arg with ref==1 ⟹ folio replaced; no byte copy.
- [ ] AC-18: FUSE_DEV_IOC_CLONE on a connected /dev/fuse fd installs new fud sharing same fc.
- [ ] AC-19: fuse_request_end frees one ref; refcount underflow asserted on get/put pairing.
- [ ] AC-20: fuse_check_timeout on expired pending req triggers fuse_abort_conn.

### Architecture

```
struct FuseReq {
  list: ListNode,
  intr_entry: ListNode,
  args: *FuseArgs,
  count: RefCount,
  flags: AtomicBitmask,               // FR_*
  in_h: FuseInHeader,
  out_h: FuseOutHeader,
  waitq: WaitQueueHead,
  fm: *FuseMount,
  create_time: u64,                   // jiffies
}

struct FuseDev {
  ref: RefCount,
  sync_init: bool,
  fc: Atomic<*FuseConn>,              // FUSE_DEV_FC_DISCONNECTED sentinel after release
  pq: FuseProcessingQueue,
  entry: ListNode,                    // on fc.devices
}

struct FuseProcessingQueue {
  connected: bool,
  lock: SpinLock,
  processing: [ListHead; FUSE_PQ_HASH_SIZE],
  io: ListHead,
}

struct FuseInputQueue {
  connected: bool,
  reqctr: u64,                        // FUSE_REQ_ID_STEP increments
  pending: ListHead,
  interrupts: ListHead,
  forget_list_head: ListHead,
  forget_list_tail: *ListNode,
  forget_batch: i32,
  waitq: WaitQueueHead,
  fasync: *FasyncStruct,
  ops: &'static FuseIQueueOps,
  lock: SpinLock,
}

struct FuseConn {
  lock: SpinLock,
  count: RefCount,
  iq: FuseInputQueue,
  devices: ListHead,
  max_read: u32, max_write: u32, max_pages: u32, max_pages_limit: u32,
  max_background: u32, congestion_threshold: u32,
  num_background: u32, active_background: u32,
  bg_queue: ListHead,
  bg_lock: SpinLock,
  blocked: bool, blocked_waitq: WaitQueueHead, num_waiting: AtomicI32,
  connected: bool, aborted: bool, initialized: bool,
  no_interrupt: bool, no_open: bool, /* ... per-capability negate bits */
  epoch: AtomicI32, epoch_work: WorkStruct,
  polled_files: RbRoot,
  timeout: FuseTimeout,
}

struct FuseCopyState {
  write: bool,
  iter: *IovIter,
  pipe: *PipeInodeInfo,
  pipebufs: *[PipeBuffer],
  req: *FuseReq,
  move_folios: bool,
  offset: usize,
  len: usize,
  currbuf: *PipeBuffer,
  pg: *Folio,
  /* ... */
}
```

`FuseDev::do_read(fud, file, cs, nbytes) -> isize`:
1. /* Buffer-size sanity */
2. min = max(FUSE_MIN_READ_BUFFER, sizeof(FuseInHeader) + sizeof(FuseWriteIn) + fc.max_write).
3. if nbytes < min: return -EINVAL.
4. /* Wait for work */
5. loop:
   - lock fiq.
   - if !fiq.connected ∨ request_pending(fiq): break.
   - unlock fiq.
   - if file.f_flags & O_NONBLOCK: return -EAGAIN.
   - wait_event_interruptible_exclusive(fiq.waitq, !fiq.connected ∨ request_pending(fiq))? — propagate -ERESTARTSYS.
6. if !fiq.connected: return fc.aborted ? -ECONNABORTED : -ENODEV.
7. /* Interrupts first */
8. if !list_empty(fiq.interrupts): return read_interrupt(fiq, cs, nbytes, head).
9. /* Forgets second (batched) */
10. if forget_pending(fiq) ∧ (list_empty(pending) ∨ fiq.forget_batch-- > 0):
    - return read_forget(fc, fiq, cs, nbytes).
11. /* Normal request */
12. req = list_first(fiq.pending, FuseReq, list).
13. clear FR_PENDING; list_del_init.
14. unlock fiq.
15. if nbytes < req.in_h.len:
    - req.out_h.error = (args.opcode == FUSE_SETXATTR) ? -E2BIG : -EIO.
    - FuseReq::end(req).
    - goto wait_for_work.
16. lock fpq; if !fpq.connected: return -ECONNABORTED.
17. list_add(req → fpq.io); unlock fpq.
18. cs.req = req.
19. FuseCopy::one(cs, &req.in_h, sizeof(FuseInHeader))?
20. FuseCopy::args(cs, args.in_numargs, args.in_pages, args.in_args, 0)?
21. FuseCopy::finish(cs).
22. lock fpq; clear FR_LOCKED; if !fpq.connected: -ECONNABORTED.
23. if !FR_ISREPLY: return reqsize (one-shot).
24. hash = fuse_req_hash(req.in_h.unique).
25. list_move_tail(req → fpq.processing[hash]).
26. FuseReq::get(req); set FR_SENT.
27. smp_mb__after_atomic.
28. if FR_INTERRUPTED: queue_interrupt(req).
29. FuseReq::put(req).
30. return reqsize.

`FuseDev::do_write(fud, cs, nbytes) -> isize`:
1. if nbytes < sizeof(FuseOutHeader): return -EINVAL.
2. FuseCopy::one(cs, &oh, sizeof(oh))?
3. if oh.len != nbytes: return -EINVAL.
4. /* Async notification */
5. if oh.unique == 0:
   - return FuseNotify::dispatch(fc, oh.error as FuseNotifyCode, nbytes - sizeof(oh), cs).
6. /* Bounds on error */
7. if oh.error <= -512 ∨ oh.error > 0: return -EINVAL.
8. lock fpq.
9. req = fuse_request_find(fpq, oh.unique & ~FUSE_INT_REQ_BIT).
10. if !req: unlock; return -ENOENT.
11. /* Interrupt reply */
12. if oh.unique & FUSE_INT_REQ_BIT:
    - FuseReq::get(req); unlock fpq.
    - if nbytes != sizeof(FuseOutHeader): err = -EINVAL.
    - else if oh.error == -ENOSYS: fc.no_interrupt = 1.
    - else if oh.error == -EAGAIN: queue_interrupt(req).
    - FuseReq::put(req); return err ? err : nbytes.
13. /* Normal reply */
14. clear FR_SENT; list_move(req → fpq.io); req.out_h = oh; set FR_LOCKED.
15. unlock fpq; cs.req = req; if !args.page_replace: cs.move_folios = false.
16. if oh.error: err = (nbytes != sizeof(oh)) ? -EINVAL : 0.
17. else: err = FuseCopy::out_args(cs, req.args, nbytes).
18. FuseCopy::finish(cs).
19. lock fpq; clear FR_LOCKED; if !fpq.connected: err = -ENOENT;
    else if err: req.out_h.error = -EIO.
20. if !FR_PRIVATE: list_del_init.
21. unlock fpq.
22. FuseReq::end(req).
23. return err ? err : nbytes.

`FuseReq::end(req)`:
1. set FR_FINISHED on req.flags.
2. clear FR_SENT.
3. if FR_BACKGROUND:
   - lock fc.bg_lock; fc.num_background--.
   - if fc.num_background < fc.max_background ∧ fc.blocked: fc.blocked = false; wake fc.blocked_waitq.
   - if fc.num_background == fc.congestion_threshold ∧ fm.sb.bdi: clear_bdi_congested.
   - fc.active_background--.
   - flush_bg_queue(fc).
   - unlock bg_lock.
4. wake req.waitq (synchronous waiter).
5. if args.end: args.end(fm, args, req.out_h.error).
6. FuseReq::put(req).

`FuseConn::flush_bg_queue(fc)`:
1. lock bg_lock (caller holds).
2. while fc.active_background < fc.max_background ∧ !list_empty(fc.bg_queue):
   - req = pop_front(fc.bg_queue).
   - fc.active_background++.
   - fuse_send_one(&fc.iq, req).
3. (caller releases lock.)

`FuseInputQueue::send_one(fiq, req)`:
1. set FR_PENDING; req.in_h.unique = fiq.reqctr; fiq.reqctr += FUSE_REQ_ID_STEP.
2. fiq.ops.send_req(fiq, req) → fuse_dev_queue_req:
   - lock fiq.lock.
   - list_add_tail(req → fiq.pending).
   - fuse_dev_wake_and_unlock(fiq):
     - if fiq.connected: wake fiq.waitq; kill_fasync(fiq.fasync, SIGIO, POLL_IN).
     - unlock.

`FuseConn::send_init(fm) -> i32` (fs/fuse/inode.c):
1. ia = kzalloc(FuseInitArgs).
2. ia.in = { major=FUSE_KERNEL_VERSION, minor=FUSE_KERNEL_MINOR_VERSION, max_readahead, flags = FUSE_ASYNC_READ|FUSE_POSIX_LOCKS|…, flags2 }.
3. args.opcode = FUSE_INIT; args.in_numargs = 1; args.out_numargs = 1; args.force = true; args.end = fuse_process_init_reply.
4. if sync_init: fuse_simple_request(fm, args).
5. else: fuse_simple_background(fm, args, GFP_KERNEL).
6. on reply: fc.minor = arg.minor; fc.max_write; fc.max_pages; fc.max_background; fc.congestion_threshold; mirror capability negotiation; fuse_set_initialized; wake fc.blocked_waitq.

`FuseConn::abort(fc)`:
1. lock fc.lock; if !fc.connected: unlock + return.
2. cancel timeout work.
3. lock bg_lock; fc.connected = false; unlock.
4. fuse_set_initialized(fc).
5. ∀ fud in fc.devices:
   - lock fud.pq.lock; fud.pq.connected = false.
   - iterate fud.pq.io / processing: mark out_h.error = -ECONNABORTED; set FR_ABORTED.
   - if !FR_LOCKED: set FR_PRIVATE; FuseReq::get; splice to to_end.
6. lock fiq.lock; fiq.connected = false; splice pending → to_end (each -ECONNABORTED); clear interrupts; end_polls; unlock.
7. wake fiq.waitq; kill_fasync(fiq.fasync, SIGIO, POLL_IN).
8. wake fc.blocked_waitq.
9. unlock fc.lock.
10. fuse_dev_end_requests(&to_end) → each: clear FR_SENT, list_del_init, FuseReq::end.
11. wake fc.blocked_waitq (final).

`FuseCopy::try_move_folio(cs, foliop) -> i32`:
1. if !cs.pipebufs ∨ !cs.req.args.page_replace ∨ !cs.move_folios: return fallback.
2. buf = cs.currbuf (next pipe_buffer).
3. if buf.flags & (PIPE_BUF_FLAG_GIFT inverse) ∨ folio_ref_count(buf.page.folio) != 1: return fallback.
4. fuse_check_folio(folio): no PageWriteback ∨ Dirty ∨ Mlocked ∨ Anon ∨ LRU pinned ∨ Slab; mapping == NULL.
5. /* Steal: replace *foliop pointer */
6. *foliop = page_folio(buf.page); cs.offset += PAGE_SIZE; cs.len -= PAGE_SIZE.
7. return PAGE_SIZE.

`FuseDev::release(inode, file)`:
1. fud = fuse_file_to_fud(file).
2. fc = xchg(&fud.fc, FUSE_DEV_FC_DISCONNECTED).
3. if fc:
   - lock fpq.lock; WARN_ON(!list_empty(fpq.io)).
   - ∀ i in 0..FUSE_PQ_HASH_SIZE: splice fpq.processing[i] → to_end.
   - unlock fpq.lock.
   - fuse_dev_end_requests(&to_end).
   - lock fc.lock; list_del(&fud.entry); last = list_empty(fc.devices); unlock.
   - if last: WARN_ON(fc.iq.fasync); FuseConn::abort(fc).
   - fuse_conn_put(fc).
4. fuse_dev_put(fud).
5. return 0.

### Out of Scope

- fs/fuse/inode.c VFS mount / lookup / sb_ops (covered in `fuse/inode.md` Tier-3)
- fs/fuse/dir.c entry-cache + d_revalidate (covered separately)
- fs/fuse/file.c read/write/mmap (covered separately)
- fs/fuse/virtio_fs.c virtio-fs queue-ops (covered separately)
- fs/fuse/dev_uring.c io_uring command interface (covered separately)
- fs/fuse/passthrough.c FUSE_PASSTHROUGH backing-file (covered separately)
- libfuse userspace library (out of kernel scope)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct fuse_conn` | per-mount kernel-side context | `FuseConn` |
| `struct fuse_dev` | per-open-/dev/fuse-fd state | `FuseDev` |
| `struct fuse_iqueue` | per-conn input queue (pending, interrupts, forgets) | `FuseInputQueue` |
| `struct fuse_pqueue` | per-dev processing queue (io list + hashed processing) | `FuseProcessingQueue` |
| `struct fuse_req` | per-request | `FuseReq` |
| `struct fuse_copy_state` | per-copy state for kiocb / splice / folio | `FuseCopyState` |
| `FR_PENDING / FR_SENT / FR_FINISHED / FR_INTERRUPTED / FR_BACKGROUND / FR_ISREPLY / FR_FORCE / FR_LOCKED / FR_ABORTED / FR_PRIVATE` | per-request flags | `FuseReqFlag` bits |
| `fuse_dev_read()` / `fuse_dev_do_read()` | per-iov dequeue + copy | `FuseDev::read` / `do_read` |
| `fuse_dev_write()` / `fuse_dev_do_write()` | per-iov reply ingest | `FuseDev::write` / `do_write` |
| `fuse_dev_splice_read()` / `fuse_dev_splice_write()` | per-pipe paths | `FuseDev::splice_read` / `splice_write` |
| `fuse_dev_poll()` | per-poll | `FuseDev::poll` |
| `fuse_dev_release()` | per-close | `FuseDev::release` |
| `fuse_dev_ioctl()` + `_clone` / `_backing_open` / `_backing_close` / `_sync_init` | per-ioctl | `FuseDev::ioctl_*` |
| `fuse_copy_init()` / `_finish()` / `_fill()` / `_do()` / `_one()` / `_args()` / `_folio()` / `_folios()` | per-copy primitives | `FuseCopy::*` |
| `fuse_try_move_folio()` | per-folio-move fast path | `FuseCopy::try_move_folio` |
| `fuse_request_init()` / `_free()` / `__fuse_get_request()` / `__fuse_put_request()` / `fuse_put_request()` | per-refcount | `FuseReq::init` / `free` / `get` / `put` |
| `fuse_request_end()` | per-finish | `FuseReq::end` |
| `request_wait_answer()` | per-wait | `FuseReq::wait_answer` |
| `queue_interrupt()` / `fuse_dev_queue_interrupt()` | per-interrupt | `FuseReq::queue_interrupt` |
| `fuse_read_interrupt()` / `_forget()` / `_single_forget()` / `_batch_forget()` | per-special-read | `FuseDev::read_interrupt` / `read_forget` |
| `flush_bg_queue()` | per-bg-drain | `FuseConn::flush_bg_queue` |
| `fuse_request_queue_background()` | per-bg-enqueue | `FuseConn::queue_background` |
| `fuse_simple_background()` | per-bg public API | `FuseConn::simple_background` |
| `fuse_dev_queue_req()` / `fuse_send_one()` | per-iqueue-send | `FuseInputQueue::send_one` |
| `fuse_dev_wake_and_unlock()` | per-waitq wake | `FuseInputQueue::wake_and_unlock` |
| `fuse_dev_queue_forget()` | per-forget-enqueue | `FuseInputQueue::queue_forget` |
| `fuse_notify()` + `_poll` / `_inval_inode` / `_inval_entry` / `_delete` / `_store` / `_retrieve` / `_resend` / `_inc_epoch` / `_prune` | per-async-notify (unique == 0) | `FuseNotify::*` |
| `fuse_abort_conn()` / `fuse_wait_aborted()` / `fuse_dev_end_requests()` | per-abort | `FuseConn::abort` / `wait_aborted` |
| `fuse_check_timeout()` / `fuse_request_expired()` | per-timeout work | `FuseConn::check_timeout` |
| `fuse_send_init()` (inode.c) | per-FUSE_INIT handshake | `FuseConn::send_init` |
| `fuse_dev_operations` (file_operations) | per-fop vtable | `FuseDev::file_ops` |
| `FUSE_DEV_IOC_CLONE` / `_BACKING_OPEN` / `_BACKING_CLOSE` / `_SYNC_INIT` | per-ioctl uapi | shared uapi |
| `FUSE_INT_REQ_BIT` | per-interrupt-unique bit | shared |
| `FUSE_MIN_READ_BUFFER` | per-min-buf-size | shared |
| `FUSE_PQ_HASH_BITS` / `_SIZE` | per-pq-hash table | shared |

### compatibility contract

REQ-1: struct fuse_req:
- list: per-iqueue-or-pqueue entry.
- intr_entry: per-fiq->interrupts entry.
- args: *fuse_args (in/out numargs, in_args, out_args, opcode, force, nocreds, page_*).
- count: refcount_t.
- flags: bitmask of FR_*.
- in.h: fuse_in_header (len, opcode, unique, nodeid, uid, gid, pid, padding).
- out.h: fuse_out_header (len, error, unique).
- waitq: per-completion wait queue.
- fm: *fuse_mount.
- create_time: per-jiffies stamp for timeout.

REQ-2: struct fuse_dev:
- ref: refcount.
- sync_init: per-bool, run FUSE_INIT synchronously.
- fc: *fuse_conn (or FUSE_DEV_FC_DISCONNECTED sentinel after release).
- pq: per-dev fuse_pqueue (connected flag, lock, processing[FUSE_PQ_HASH_SIZE], io list).
- entry: list-link in fc->devices.

REQ-3: struct fuse_pqueue:
- connected: per-dev liveness.
- lock: spinlock.
- processing: per-hash bucket array of FUSE_PQ_HASH_SIZE list_heads keyed by `hash_long(unique & ~FUSE_INT_REQ_BIT, FUSE_PQ_HASH_BITS)`.
- io: per-list of requests currently being copied to/from userspace (FR_LOCKED).

REQ-4: struct fuse_iqueue:
- connected: per-conn liveness.
- reqctr: per-monotonic-unique counter (FUSE_REQ_ID_STEP increments; 0 reserved for notifications).
- pending: list of requests not yet read by userspace (FR_PENDING).
- interrupts: list of synthesized FUSE_INTERRUPT requests.
- forget_list_head / _tail / forget_batch: per-batched forget queue.
- waitq: per-readers wait queue.
- fasync: per-SIGIO fasync_struct list.
- ops: per-iqueue-ops vtable (send_forget, send_interrupt, send_req, release) — `fuse_dev_fiq_ops` for /dev/fuse, distinct for virtio-fs.

REQ-5: struct fuse_conn (subset):
- count: refcount.
- iq: fuse_iqueue.
- devices: list of fuse_dev open fds.
- max_read, max_write, max_pages, max_pages_limit: per-negotiation.
- max_background: per-conn limit (default FUSE_DEFAULT_MAX_BACKGROUND = 12, capped at max_user_bgreq for non-CAP_SYS_ADMIN).
- congestion_threshold: per-bdi congestion gate.
- num_background, active_background: per-counters under bg_lock.
- bg_queue: per-list of FR_BACKGROUND not yet dispatched.
- blocked, blocked_waitq, num_waiting: per-flow-control (blocked while num_background >= max_background).
- bg_lock: spinlock for max_background/num_background/active_background/bg_queue/blocked.
- connected, aborted, initialized, conn_init, conn_error: per-state.
- no_interrupt, no_open, no_release, no_…: per-negotiated-capability "not supported" memo.
- timeout: per-deferred-work hung-FUSE detector.
- polled_files: per-rbtree of fuse_files awaiting poll-notifications.
- epoch: per-mount dentry epoch (incremented on `FUSE_NOTIFY_INC_EPOCH`).

REQ-6: fuse_dev_open(inode, file):
- fud = fuse_dev_alloc() — initializes refcount/sync_init/pq, fc = NULL.
- file->private_data = fud.

REQ-7: fuse_dev_read(iocb, to):
- fud = fuse_get_dev(file) — waits for `fuse_dev_install` to set fud->fc (interruptible).
- must be user_backed_iter; else -EINVAL.
- fuse_copy_init(&cs, /*write=*/true, to).
- return fuse_dev_do_read(fud, file, &cs, iov_iter_count(to)).

REQ-8: fuse_dev_do_read(fud, file, cs, nbytes):
- /* Sanity: minimum buffer */
- if nbytes < max(FUSE_MIN_READ_BUFFER, sizeof(fuse_in_header)+sizeof(fuse_write_in)+fc->max_write): return -EINVAL.
- /* Wait for any work */
- loop: lock fiq; if !fiq->connected ∨ request_pending(fiq) break; unlock; (O_NONBLOCK ⟹ -EAGAIN); wait_event_interruptible_exclusive(fiq->waitq, !fiq->connected ∨ request_pending(fiq)).
- if !fiq->connected: return fc->aborted ? -ECONNABORTED : -ENODEV.
- /* Priority: interrupts > forgets > regular */
- if !list_empty(&fiq->interrupts): return fuse_read_interrupt(fiq, cs, nbytes, head).
- if forget_pending(fiq) ∧ (list_empty(pending) ∨ forget_batch-- > 0): return fuse_read_forget(...).
- /* Dequeue head of pending */
- req = list_first(&fiq->pending, fuse_req, list); clear_bit(FR_PENDING); list_del_init.
- unlock fiq.
- /* Bounce too-large request as -EIO (FUSE_SETXATTR ⟹ -E2BIG) */
- if nbytes < req->in.h.len: req->out.h.error = -EIO (or -E2BIG); fuse_request_end; restart.
- /* Move to fpq->io (FR_LOCKED set) */
- lock fpq; if !fpq->connected: -ECONNABORTED. list_add(req → fpq->io). unlock.
- cs->req = req.
- fuse_copy_one(cs, &req->in.h, sizeof(fuse_in_header)).
- fuse_copy_args(cs, args->in_numargs, args->in_pages, args->in_args, 0).
- fuse_copy_finish(cs).
- /* Decide post-copy state */
- if !req->args->test FR_ISREPLY: return reqsize (one-shot — release, free).
- /* Move to fpq->processing[hash] (FR_SENT set) */
- hash = fuse_req_hash(req->in.h.unique).
- list_move_tail(&req->list, &fpq->processing[hash]).
- __fuse_get_request(req); set FR_SENT.
- if FR_INTERRUPTED set: queue_interrupt(req).
- fuse_put_request(req).
- return reqsize.

REQ-9: fuse_dev_splice_read(in, ppos, pipe, len, flags):
- fud = fuse_get_dev.
- bufs = kvmalloc_objs(pipe_buffer, pipe->max_usage).
- fuse_copy_init(&cs, true, NULL). cs.pipebufs = bufs. cs.pipe = pipe.
- ret = fuse_dev_do_read(fud, in, &cs, len).
- if ret < 0: kvfree(bufs); return ret.
- splice_grow_spd → splice_to_pipe with the captured page_nr buffers; kvfree(bufs).

REQ-10: fuse_dev_write(iocb, from):
- fud = __fuse_get_dev(iocb->ki_filp). if !fud: -EPERM.
- must be user_backed_iter.
- fuse_copy_init(&cs, /*write=*/false, from).
- return fuse_dev_do_write(fud, &cs, iov_iter_count(from)).

REQ-11: fuse_dev_do_write(fud, cs, nbytes):
- if nbytes < sizeof(fuse_out_header): -EINVAL.
- fuse_copy_one(cs, &oh, sizeof(oh)).
- if oh.len != nbytes: -EINVAL.
- /* unique == 0 ⟹ async notification (server → kernel) */
- if !oh.unique: return fuse_notify(fc, oh.error /* code */, nbytes - sizeof(oh), cs).
- if oh.error <= -512 ∨ oh.error > 0: -EINVAL.
- lock fpq; req = fuse_request_find(fpq, oh.unique & ~FUSE_INT_REQ_BIT); if !req: -ENOENT.
- if oh.unique & FUSE_INT_REQ_BIT:
  - /* interrupt reply */
  - if oh.error == -ENOSYS: fc->no_interrupt = 1.
  - else if oh.error == -EAGAIN: queue_interrupt(req).
  - fuse_put_request(req); return.
- /* Normal reply */
- clear FR_SENT; list_move(req → fpq->io); req->out.h = oh; set FR_LOCKED.
- unlock fpq.
- cs->req = req; if !args->page_replace: cs->move_folios = false.
- if oh.error: err = (nbytes != sizeof(oh)) ? -EINVAL : 0.
- else: err = fuse_copy_out_args(cs, req->args, nbytes).
- fuse_copy_finish.
- lock fpq; clear FR_LOCKED; if !fpq->connected: -ENOENT; else if err: req->out.h.error = -EIO; if !FR_PRIVATE: list_del_init. unlock.
- fuse_request_end(req).
- return nbytes.

REQ-12: fuse_dev_splice_write(pipe, out, ppos, len, flags):
- Atomically capture pipe head/tail/count snapshot.
- Build fuse_copy_state over the captured pipe_buffer[] with cs.pipebufs.
- Call fuse_dev_do_write.

REQ-13: fuse_dev_poll(file, wait):
- mask = EPOLLOUT | EPOLLWRNORM (write side always ready).
- poll_wait(file, &fiq->waitq, wait).
- if !fiq->connected: mask = EPOLLERR.
- else if request_pending(fiq): mask |= EPOLLIN | EPOLLRDNORM.
- return mask.

REQ-14: fuse_dev_release(inode, file):
- fud = fuse_file_to_fud(file).
- fc = xchg(&fud->fc, FUSE_DEV_FC_DISCONNECTED) — pairs with cmpxchg in fuse_dev_install.
- if fc:
  - splice fpq->processing[i] (∀ i) → to_end; WARN if fpq->io non-empty.
  - fuse_dev_end_requests(&to_end) — ECONNABORTED + fuse_request_end on each.
  - list_del(&fud->entry).
  - last = list_empty(&fc->devices).
  - if last: WARN_ON(fc->iq.fasync); fuse_abort_conn(fc).
  - fuse_conn_put(fc).
- fuse_dev_put(fud).

REQ-15: Request lifecycle:
- /* Submit (kernel → daemon) */
- fuse_simple_request / fuse_simple_background fills fuse_req (args, in.h.unique = fiq->reqctr; reqctr += FUSE_REQ_ID_STEP).
- fuse_dev_queue_req: set FR_PENDING; list_add_tail(req → fiq->pending); fuse_dev_wake_and_unlock(fiq) — wakes waitq + signals fasync.
- daemon reads via fuse_dev_read; moves to fpq->processing[hash]; FR_SENT.
- /* Reply */
- daemon writes via fuse_dev_write; matches by unique; moves to fpq->io; FR_LOCKED.
- After copy: fuse_request_end(req) — clear FR_SENT, list_del_init, wake_up(waitq), drop refs.
- /* Synchronous waiter */
- request_wait_answer: wait_event_interruptible(req->waitq, FR_FINISHED set) — if interrupted ∧ !FR_FORCE: set FR_INTERRUPTED, queue_interrupt, wait_event_killable.

REQ-16: Interrupt request:
- queue_interrupt(req): if FR_FINISHED, no-op; else fiq->ops->send_interrupt(fiq, req).
- fuse_dev_queue_interrupt(fiq, req): lock fiq; if !FR_FINISHED ∧ !FR_INTERRUPTED already on intr list: list_add(&req->intr_entry, &fiq->interrupts), fuse_dev_wake_and_unlock.
- Reader serves interrupts ahead of any pending/forget: fuse_read_interrupt synthesizes a fuse_in_header { opcode=FUSE_INTERRUPT, unique=req->in.h.unique | FUSE_INT_REQ_BIT, len=sizeof(ih)+sizeof(struct fuse_interrupt_in) }.
- Daemon replies on the | FUSE_INT_REQ_BIT unique: -ENOSYS ⟹ fc->no_interrupt = 1; -EAGAIN ⟹ requeue.

REQ-17: FUSE_INIT handshake (`fuse_send_init` in fs/fuse/inode.c):
- args.opcode = FUSE_INIT; args.in_args = fuse_init_in { major=FUSE_KERNEL_VERSION, minor=FUSE_KERNEL_MINOR_VERSION, max_readahead, flags, flags2 }.
- args.out_args = fuse_init_out { major, minor, max_readahead, flags, flags2, max_background, congestion_threshold, max_write, max_pages, … }.
- Either FORCE-async (FR_FORCE) or synchronous (sync_init dev): on reply, fuse_process_init_reply sets fc->minor, fc->max_write, fc->max_pages, fc->max_background, congestion_threshold, capability flag mirrors (no_open, no_force_umount, async_read, big_writes, splice_*, dont_mask, posix_locks, atomic_o_trunc, …); finally fuse_set_initialized(fc) — wakes blocked_waitq.

REQ-18: Background queue (fc->max_background):
- fuse_simple_background: sets FR_BACKGROUND; fuse_request_queue_background:
  - lock bg_lock; if !connected: -ENOTCONN.
  - num_background++.
  - if num_background == max_background: fc->blocked = 1.
  - if num_background == congestion_threshold ∧ bdi: clear_bdi_congested.
  - list_add_tail(&req->list, &fc->bg_queue).
  - flush_bg_queue(fc).
- flush_bg_queue: while active_background < max_background ∧ !list_empty(bg_queue): pop head; active_background++; fuse_send_one(fiq, req).
- on completion: num_background--; active_background--; if blocked ∧ num_background < max_background: blocked = 0 + wake blocked_waitq.

REQ-19: fuse_copy_init / _fill / _do / _one / _args / _folio / _folios:
- fuse_copy_init(cs, write, iter): zeroes cs; cs->write = write; cs->iter = iter.
- fuse_copy_finish(cs): kunmap_local(cs->currbuf if any); pipe_buf_release on any leftover; iov_iter_advance(cs->iter, copied) where applicable.
- fuse_copy_fill(cs): if cs->pipe: fetch next pipe_buffer; else copy_page_from_iter / copy_page_to_iter; on dispatch leftover buffer.
- fuse_copy_do(cs, &val, &size): per-byte-or-folio inner copy (read or write direction).
- fuse_copy_one(cs, val, size): loop fuse_copy_do until size bytes copied; error propagates -EFAULT / -EINVAL.
- fuse_copy_args(cs, numargs, argpages, args, zeroing): per-arg fuse_copy_one (or fuse_copy_folios for paged args).
- fuse_copy_folios(cs, nbytes, args, zeroing): walks args->folios, copies via fuse_copy_folio (or fuse_try_move_folio fast path).
- fuse_try_move_folio: if cs->pipebufs ∧ page_replace ∧ ref==1 ∧ aligned ∧ checked by fuse_check_folio: replace dest folio in place (zero-copy steal), increment cs->offset, return success — failure path falls back to byte copy.

REQ-20: fuse_copy_out_args(cs, args, nbytes):
- count = nbytes - sizeof(fuse_out_header).
- For each args->out_args[i]: bounded by remaining count; final out arg may be short (FUSE_ARG_LAZY semantics).
- if args->out_pages: fuse_copy_folios for the paged-output region.

REQ-21: fuse_notify(fc, code, size, cs):
- Dispatch on enum fuse_notify_code: FUSE_NOTIFY_POLL → fuse_notify_poll; FUSE_NOTIFY_INVAL_INODE → fuse_notify_inval_inode; FUSE_NOTIFY_INVAL_ENTRY; FUSE_NOTIFY_STORE; FUSE_NOTIFY_RETRIEVE (kernel pulls pages, server gets reply with FUSE_NOTIFY_REPLY-tagged in-header); FUSE_NOTIFY_DELETE; FUSE_NOTIFY_RESEND; FUSE_NOTIFY_INC_EPOCH; FUSE_NOTIFY_PRUNE.
- All execute under fuse_dev_do_write context and may rate-limit, lock specific inodes, or fan out work to inode.c / dir.c helpers.

REQ-22: fuse_abort_conn(fc):
- /* Emergency teardown */
- lock fc->lock; if !fc->connected: return.
- cancel timeout work.
- lock bg_lock; fc->connected = 0; unlock bg_lock.
- fuse_set_initialized(fc) — release any waiters blocked in FUSE_INIT.
- For each fud in fc->devices:
  - lock fpq; fpq->connected = 0; iterate fpq->io and fpq->processing:
    - mark req->out.h.error = -ECONNABORTED; set FR_ABORTED.
    - if !FR_LOCKED: set FR_PRIVATE; __fuse_get_request; splice to to_end.
- lock fiq; fiq->connected = 0; splice fiq->pending into to_end with -ECONNABORTED; clear interrupts; end_polls; unlock.
- wake fiq->waitq; kill_fasync(fc->iq.fasync, SIGIO, POLL_IN).
- wake fc->blocked_waitq.
- fuse_dev_end_requests(&to_end).
- wake fc->blocked_waitq again post-end so fuse_wait_aborted unblocks.

REQ-23: fuse_wait_aborted(fc):
- smp_mb (pairs with fuse_drop_waiting).
- wait_event(blocked_waitq, atomic_read(&fc->num_waiting) == 0).
- fuse_uring_wait_stopped_queues(fc) (if FUSE_IO_URING).

REQ-24: FUSE_RELEASE auto-cleanup on /dev/fuse close:
- Per-REQ-14 above: fuse_dev_release on last open fd triggers fuse_abort_conn.
- The kernel-side fuse_mount synthesizes FUSE_RELEASE / FUSE_RELEASEDIR / FUSE_FORGET as VFS releases its references; they queue normally, then drain via fuse_abort_conn into ECONNABORTED if no daemon left to service them.

REQ-25: fuse_dev_ioctl:
- FUSE_DEV_IOC_CLONE: clone existing /dev/fuse fd onto new fd (per-cpu queue scaling) — both share fc, get independent fpq.
- FUSE_DEV_IOC_BACKING_OPEN: install a passthrough backing file (FUSE_PASSTHROUGH).
- FUSE_DEV_IOC_BACKING_CLOSE: close a backing-file id.
- FUSE_DEV_IOC_SYNC_INIT: switch this dev to synchronous FUSE_INIT mode.

REQ-26: fuse_check_timeout:
- delayed_work that fires every fc->timeout.req_timeout jiffies.
- fuse_request_expired(fc, list): walks list, compares (jiffies - req->create_time) against req_timeout.
- fuse_fpq_processing_expired similarly across processing buckets.
- On timeout: WARN + force fuse_abort_conn (configurable).

REQ-27: fuse_dev_show_fdinfo (CONFIG_PROC_FS):
- seq_printf "fuse_connection:\t%u\n" with fc->dev (minor number).

REQ-28: fuse_dev_operations vtable:
- .owner, .open=fuse_dev_open, .read_iter=fuse_dev_read, .splice_read=fuse_dev_splice_read, .write_iter=fuse_dev_write, .splice_write=fuse_dev_splice_write, .poll=fuse_dev_poll, .release=fuse_dev_release, .fasync=fuse_dev_fasync, .unlocked_ioctl=fuse_dev_ioctl, .compat_ioctl=compat_ptr_ioctl, .uring_cmd=fuse_uring_cmd (FUSE_IO_URING), .show_fdinfo=fuse_dev_show_fdinfo (PROC_FS).

REQ-29: fuse_miscdevice registration:
- .minor = FUSE_MINOR; .name = "fuse"; .fops = &fuse_dev_operations.
- misc_register at module init (fuse_dev_init), misc_deregister + kmem_cache_destroy at exit.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `req_unique_monotonic_step` | INVARIANT | per-send_one: reqctr += FUSE_REQ_ID_STEP; unique strictly monotone within a fiq epoch. |
| `req_state_pending_to_sent` | INVARIANT | per-do_read: FR_PENDING cleared before FR_SENT set; req on at most one fpq list. |
| `req_state_sent_to_finished` | INVARIANT | per-do_write or end: FR_SENT cleared before FR_FINISHED set. |
| `read_min_buffer_enforced` | INVARIANT | per-do_read: nbytes < min ⟹ -EINVAL (no partial header). |
| `write_oh_len_matches_nbytes` | INVARIANT | per-do_write: oh.len != nbytes ⟹ -EINVAL. |
| `write_oh_error_bounded` | INVARIANT | per-do_write: oh.error ∈ (-511..0] for non-notification, non-interrupt. |
| `interrupt_unique_bit_set` | INVARIANT | per-read_interrupt: emitted ih.unique has FUSE_INT_REQ_BIT. |
| `interrupt_reply_matches_orig` | INVARIANT | per-do_write: oh.unique & FUSE_INT_REQ_BIT ⟹ matches existing req by lower bits. |
| `max_background_blocks_submitters` | INVARIANT | per-queue_background: num_background == max_background ⟹ fc.blocked = true. |
| `flush_bg_queue_under_bg_lock` | INVARIANT | per-flush_bg_queue: bg_lock held; active_background ≤ max_background. |
| `request_refcount_get_put_balanced` | INVARIANT | per-do_read/do_write: every __fuse_get_request paired with fuse_put_request / fuse_request_end. |
| `last_dev_release_aborts` | INVARIANT | per-release: list_empty(fc.devices) ⟹ fuse_abort_conn invoked. |
| `notify_unique_zero` | INVARIANT | per-do_write: oh.unique == 0 ⟹ fuse_notify path; never matches a req. |
| `splice_read_no_iter` | INVARIANT | per-splice_read: cs.iter == NULL; cs.pipebufs ≠ NULL. |
| `try_move_folio_safe` | INVARIANT | per-try_move_folio: folio passes fuse_check_folio (no writeback, no anon, no LRU, ref==1) before steal. |
| `init_synchronizes` | INVARIANT | per-process_init_reply: fc.initialized true ⟹ blocked_waitq woken. |
| `fpq_disconnect_drains_processing` | INVARIANT | per-release: fpq.processing buckets emptied to to_end before fuse_dev_end_requests. |

### Layer 2: TLA+

`fs/fuse/dev.tla`:
- Per-/dev/fuse-open + per-FUSE_INIT-handshake + per-request-submit + per-daemon-read + per-daemon-write + per-interrupt + per-background-queue + per-abort.
- Properties:
  - `safety_request_unique_id_unique_per_fiq_epoch` — per-send_one: distinct requests get distinct unique within a fiq.
  - `safety_state_machine_pending_sent_finished` — per-req: state transitions only along PENDING→SENT→FINISHED (or PENDING→FINISHED for one-shot / aborted).
  - `safety_interrupt_priority` — per-do_read: list_empty(interrupts) false ⟹ next byte stream serves the interrupt request.
  - `safety_max_background_gate` — per-queue_background: invariant num_background ≤ max_background after blocked-flag set.
  - `safety_abort_drains_all` — per-abort: after abort, every fuse_req on iq/pq lists has out.h.error == -ECONNABORTED and FR_FINISHED set.
  - `liveness_pending_eventually_drained` — per-pending-req: with live daemon ∧ buffer-large-enough, eventually FR_SENT set (or daemon disconnected ⟹ ECONNABORTED).
  - `liveness_bg_drains_when_unblocked` — per-bg_queue: eventually flushed once active_background < max_background.
  - `liveness_init_handshake_completes` — per-FUSE_INIT: sent ⟹ eventually fc.initialized ∨ fc.aborted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FuseDev::do_read` post: ret >= sizeof(FuseInHeader) ∨ ret < 0 | `FuseDev::do_read` |
| `FuseDev::do_read` post: ret > 0 ⟹ either FR_SENT set ∨ !FR_ISREPLY | `FuseDev::do_read` |
| `FuseDev::do_write` post: ret > 0 ⟹ matching FuseReq::end called | `FuseDev::do_write` |
| `FuseReq::end` post: FR_FINISHED set, waiter woken | `FuseReq::end` |
| `FuseConn::flush_bg_queue` post: active_background ≤ max_background | `FuseConn::flush_bg_queue` |
| `FuseConn::queue_background` post: num_background incremented atomically under bg_lock | `FuseConn::queue_background` |
| `FuseInputQueue::send_one` post: req.in_h.unique != 0 ∧ FR_PENDING set | `FuseInputQueue::send_one` |
| `FuseConn::abort` post: ∀ req on lists ⟹ out.h.error == -ECONNABORTED | `FuseConn::abort` |
| `FuseDev::release` post: fud.fc == FUSE_DEV_FC_DISCONNECTED | `FuseDev::release` |
| `FuseCopy::try_move_folio` post: success ⟹ folio refcount preserved, mapping null | `FuseCopy::try_move_folio` |

### Layer 4: Verus/Creusot functional

`Per-/dev/fuse open → fuse_send_init → fuse_dev_read (PENDING→SENT) → daemon-process → fuse_dev_write (SENT→FINISHED + reply data) → waiter wake → fuse_request_end → refcount drop → close ⟹ fuse_abort_conn ⟹ ECONNABORTED on residuals` semantic equivalence: per-Documentation/filesystems/fuse.rst and include/uapi/linux/fuse.h ABI (protocol version 7.x).

### hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

FUSE-device reinforcement:

- **Per-FUSE_MIN_READ_BUFFER enforced** — defense against per-truncated-header DoS.
- **Per-oh.len == nbytes strict** — defense against per-malformed-reply parsing.
- **Per-oh.error bounded (-511..0]** — defense against per-arbitrary-errno injection.
- **Per-unique match under fpq.lock** — defense against per-reply-race (TOCTOU on req lookup).
- **Per-FR_LOCKED during copy** — defense against per-mid-copy mutation by abort.
- **Per-FUSE_INT_REQ_BIT segregation** — defense against per-interrupt-reply confusion with normal reply.
- **Per-fc.no_interrupt sticky after ENOSYS** — defense against per-interrupt-flood on filesystem-not-supporting.
- **Per-max_background cap (with max_user_bgreq for non-CAP_SYS_ADMIN)** — defense against per-FUSE-bg-DoS from unprivileged daemon.
- **Per-blocked_waitq throttling** — defense against per-runaway-submitter (caller blocks on bg saturation).
- **Per-congestion_threshold + bdi-congested** — defense against per-write-storm to /dev/fuse.
- **Per-fuse_check_folio gate on try_move_folio** — defense against per-folio-steal-corruption (writeback/dirty/anon/Slab/LRU rejected).
- **Per-fpq.connected drain on release** — defense against per-stale-req after daemon-close.
- **Per-last-dev abort drains pending** — defense against per-orphaned-FUSE-mount hanging VFS.
- **Per-FUSE_DEV_FC_DISCONNECTED sentinel via xchg** — defense against per-fud-after-release UAF.
- **Per-fuse_check_timeout watchdog** — defense against per-malicious-or-hung daemon.
- **Per-FR_FORCE for FUSE_INIT / FUSE_DESTROY** — defense against per-shutdown-deadlock (cannot be interrupted).
- **Per-fuse_simple_background GFP_KERNEL with __GFP_HIGH** — defense against per-OOM-during-FUSE-init.
- **Per-FUSE_NOTIFY_INVAL_ENTRY input length validated** — defense against per-malicious-server-spoofed-name (size and uniqueness).
- **Per-CAP_SYS_ADMIN required for max_background > max_user_bgreq override** — defense against per-unprivileged-resource-exhaustion.
- **Per-fasync only on the fuse_iqueue, not per-fud** — defense against per-SIGIO-storm after CLONE.
- **Per-FUSE_DEV_IOC_CLONE fd-checked f_op == file->f_op** — defense against per-cross-driver fd-injection.
- **Per-args->page_replace required for cs->move_folios** — defense against per-folio-steal where caller never opted in.
- **Per-recursive-mount guard via super_block flags (covered in inode.c)** — defense against per-stack-overflow on cross-mount FUSE.

