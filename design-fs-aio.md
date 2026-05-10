---
title: "Tier-3: fs/aio.c — POSIX-AIO / kernel-aio"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The kernel-AIO subsystem implements the legacy Linux native asynchronous-I/O ABI: per-process **kioctx** ring buffers established by `io_setup()`, per-iocb commands (PREAD/PWRITE/PREADV/PWRITEV/FSYNC/FDSYNC/POLL) submitted via `io_submit()`, completion events drained via `io_getevents()` (and time-pgetevents variants), in-flight cancellation via `io_cancel()`, and teardown via `io_destroy()`. The completion ring is an anonymous pseudo-file backed by `aio_ring_fops`, **mmap'd into user-space** so user-space can poll the head/tail pointers without a syscall. Per-iocb backing types: read/write reuse the VFS `kiocb` + `iov_iter` path, FSYNC schedules a `work_struct`, POLL hooks into the file's wait-queue. Completion (`aio_complete`) writes an `io_event` to the ring, increments tail, optionally signals an eventfd, and wakes any `io_getevents` waiters. Critical for: glibc POSIX-AIO, databases that pre-date io_uring (Oracle, MySQL InnoDB), userspace block-cache servers.

This Tier-3 covers `fs/aio.c` (~2499 lines).

### Acceptance Criteria

- [ ] AC-1: io_setup(N, &ctx) returns 0 with valid aio_context_t (= mmap_base); ring magic == AIO_RING_MAGIC visible to user-space.
- [ ] AC-2: io_setup with nr_events==0 OR initial *ctxp != 0 returns -EINVAL.
- [ ] AC-3: io_setup beyond aio_max_nr returns -EAGAIN; aio_nr decremented on io_destroy.
- [ ] AC-4: io_submit(IOCB_CMD_PREAD) on regular file returns 1; eventually aio_complete writes io_event with res = bytes-read.
- [ ] AC-5: io_submit(IOCB_CMD_PWRITE) issues async write; res == nbytes on success; aio_complete signals eventfd if IOCB_FLAG_RESFD.
- [ ] AC-6: io_submit(IOCB_CMD_FSYNC / FDSYNC) schedules workqueue; res == 0 after fsync_range.
- [ ] AC-7: io_submit(IOCB_CMD_POLL) arms wait queue; aio_complete on first wake matching events mask.
- [ ] AC-8: io_submit(invalid opcode) returns -EINVAL for that iocb; partial-success returns count of successful submissions.
- [ ] AC-9: io_getevents min_nr=N waits until ≥N completions OR timeout; returns -EINTR on signal.
- [ ] AC-10: io_cancel of POLL iocb succeeds; ring delivers cancellation result; return == -EINPROGRESS when handler cancels.
- [ ] AC-11: io_cancel with bad aio_key returns -EINVAL (no KIOCB_KEY).
- [ ] AC-12: io_destroy drains in-flight iocbs; wait_for_completion returns after free_ioctx.
- [ ] AC-13: mm exit (exit_aio) cleans up all kioctxs without leak.
- [ ] AC-14: User-space ring head update visible to kernel under ring_lock; tail update visible to user-space after smp_wmb.
- [ ] AC-15: aio_ring folio migration via aio_migrate_folio preserves user mappings.

### Architecture

```
struct AioRing {                       // user-visible mmap layout (no padding)
  id: u32,
  nr: u32,
  head: u32,                           // userspace cursor
  tail: u32,                           // kernel cursor
  magic: u32,                          // AIO_RING_MAGIC
  compat_features: u32,
  incompat_features: u32,
  header_length: u32,
  // io_events[]: flex array follows
}

struct IoEvent {                       // user-visible per-completion
  data: u64,
  obj: u64,
  res: i64,
  res2: i64,
}

struct Kioctx {
  users: PercpuRef,
  dead: AtomicU32,
  reqs: PercpuRef,
  user_id: u64,                        // == mmap_base
  cpu: PerCpu<KioctxCpu>,
  req_batch: u32,
  max_reqs: u32,
  nr_events: u32,
  mmap_base: u64,
  mmap_size: u64,
  ring_folios: Vec<*Folio>,
  nr_pages: i64,
  free_rwork: RcuWork,
  rq_wait: Option<*CtxRqWait>,
  reqs_available: CacheAligned<AtomicI32>,
  cancel: CacheAligned<{ ctx_lock: Spinlock, active_reqs: List<AioKiocb> }>,
  reader: CacheAligned<{ ring_lock: Mutex, wait: WaitQueueHead }>,
  producer: CacheAligned<{ tail: u32, completed_events: u32, completion_lock: Spinlock }>,
  internal_folios: [*Folio; AIO_RING_PAGES /* 8 */],
  aio_ring_file: *File,
  id: u32,                             // kioctx_table index
}

struct AioKiocb {
  // union { ki_filp, rw: Kiocb, fsync: FsyncIocb, poll: PollIocb }
  ki: AioKiocbBacking,
  ki_ctx: *Kioctx,
  ki_cancel: Option<KiocbCancelFn>,
  ki_res: IoEvent,                     // template for completion
  ki_list: ListNode,                   // on Kioctx.cancel.active_reqs
  ki_refcnt: RefCount,
  ki_eventfd: Option<*EventfdCtx>,
}

enum AioKiocbBacking {
  Rw(Kiocb),
  Fsync(FsyncIocb),
  Poll(PollIocb),
}

enum IocbCmd {
  Pread   = 0,
  Pwrite  = 1,
  Fsync   = 2,
  Fdsync  = 3,
  Poll    = 5,
  Noop    = 6,
  Preadv  = 7,
  Pwritev = 8,
}
```

`Aio::io_setup(nr_events, ctxp) -> i64`:
1. ctx = get_user(ctxp).
2. if ctx != 0 ∨ nr_events == 0: return -EINVAL.
3. ioctx = Aio::ioctx_alloc(nr_events).
4. if ioctx.is_err: return ioctx.err.
5. put_user(ioctx.user_id, ctxp).
6. if put_user fails: Aio::kill_ioctx(current.mm, ioctx, None); return -EFAULT.
7. percpu_ref_put(&ioctx.users).
8. return 0.

`Aio::ioctx_alloc(nr_events) -> Result<*Kioctx>`:
1. /* Round-up + 2 for header slots */
2. nr_events = nr_events.checked_mul(2).ok_or(-EINVAL)?.
3. nr_events = nr_events.checked_add(2).ok_or(-EINVAL)?.
4. /* System-wide quota under aio_nr_lock */
5. lock aio_nr_lock:
   - if aio_nr + nr_events > aio_max_nr ∨ overflow: return -EAGAIN.
   - reserve aio_nr += nr_events.
6. ctx = kmem_cache_zalloc(kioctx_cachep).
7. ctx.max_reqs = nr_events / 2 - 1.
8. /* Setup ring (allocates ring_folios + mmap into current.mm) */
9. Aio::setup_ring(ctx, nr_events) — assigns nr_events, mmap_base, ring_folios.
10. ctx.user_id = ctx.mmap_base.
11. /* percpu refs */
12. percpu_ref_init(&ctx.users, free_ioctx_users).
13. percpu_ref_init(&ctx.reqs, free_ioctx_reqs).
14. ctx.cpu = alloc_percpu(KioctxCpu).
15. /* RCU-publish */
16. Aio::ioctx_add_table(ctx, current.mm).
17. return Ok(ctx).

`Aio::setup_ring(ctx, nr_events)`:
1. /* compute number of ring pages */
2. nr_pages = (sizeof(AioRing) + nr_events * sizeof(IoEvent) + PAGE_SIZE - 1) / PAGE_SIZE.
3. ctx.aio_ring_file = aio_private_file(ctx, nr_pages).
4. /* Populate folios */
5. for i in 0..nr_pages:
   - folio = filemap_grab_folio(aio_ring_file.f_mapping, i).
   - ctx.ring_folios[i] = folio.
6. /* mmap into user-space */
7. ctx.mmap_base = vm_mmap(aio_ring_file, 0, nr_pages * PAGE_SIZE, PROT_READ|PROT_WRITE, MAP_SHARED, 0).
8. ring = folio_address(ring_folios[0]) as *AioRing.
9. ring.nr = nr_events.
10. ring.head = 0; ring.tail = 0.
11. ring.magic = AIO_RING_MAGIC.
12. ring.compat_features = 1; ring.incompat_features = 0.
13. ring.header_length = sizeof(AioRing).
14. flush_dcache_folio(ring_folios[0]).

`Aio::io_submit(ctx_id, nr, iocbpp) -> i64`:
1. if nr < 0: return -EINVAL.
2. ctx = Aio::lookup_ioctx(ctx_id); if !ctx: return -EINVAL.
3. nr = min(nr, ctx.nr_events).
4. if nr > AIO_PLUG_THRESHOLD: blk_start_plug(&plug).
5. for i in 0..nr:
   - user_iocb = get_user(iocbpp + i).if_err { ret = -EFAULT; break }.
   - ret = Aio::submit_one(ctx, user_iocb, /*compat=*/false).
   - if ret != 0: break.
6. if nr > AIO_PLUG_THRESHOLD: blk_finish_plug.
7. percpu_ref_put(&ctx.users).
8. return if i > 0 { i } else { ret }.

`Aio::submit_one(ctx, user_iocb, compat) -> i64`:
1. iocb = copy_from_user(user_iocb).map_err(|_| -EFAULT)?.
2. if iocb.aio_reserved2 != 0: return -EINVAL.
3. /* overflow guard on aio_buf / aio_nbytes */
4. req = Aio::get_req(ctx).ok_or(-EAGAIN)?.
5. err = Aio::submit_one_inner(ctx, &iocb, user_iocb, req, compat).
6. Aio::iocb_put(req) — synchronous ref drop.
7. if err != 0: Aio::iocb_destroy(req); Aio::put_reqs_avail(ctx, 1).
8. return err.

`Aio::submit_one_inner(ctx, iocb, user_iocb, req, compat) -> i64`:
1. req.ki_filp = fget(iocb.aio_fildes); if !filp: return -EBADF.
2. if iocb.aio_flags & IOCB_FLAG_RESFD:
   - req.ki_eventfd = eventfd_ctx_fdget(iocb.aio_resfd).map_err(...)?.
3. put_user(KIOCB_KEY, &user_iocb.aio_key).map_err(|_| -EFAULT)?.
4. req.ki_res = IoEvent { obj: user_iocb as u64, data: iocb.aio_data, res: 0, res2: 0 }.
5. match iocb.aio_lio_opcode:
   - PREAD => Aio::read(&req.rw, iocb, false, compat),
   - PWRITE => Aio::write(&req.rw, iocb, false, compat),
   - PREADV => Aio::read(&req.rw, iocb, true, compat),
   - PWRITEV => Aio::write(&req.rw, iocb, true, compat),
   - FSYNC => Aio::fsync(&req.fsync, iocb, false),
   - FDSYNC => Aio::fsync(&req.fsync, iocb, true),
   - POLL => Aio::poll(req, iocb),
   - _ => -EINVAL.

`Aio::complete(iocb)`:
1. ctx = iocb.ki_ctx.
2. spin_lock_irqsave(&ctx.producer.completion_lock).
3. tail = ctx.producer.tail.
4. pos = tail + AIO_EVENTS_OFFSET.
5. if (tail+1) >= ctx.nr_events { tail = 0 } else { tail += 1 }.
6. ev_page = folio_address(ctx.ring_folios[pos / events_per_page]).
7. event = ev_page + pos % events_per_page.
8. *event = iocb.ki_res.
9. flush_dcache_folio(ctx.ring_folios[pos / events_per_page]).
10. smp_wmb() — event visible before tail update.
11. ctx.producer.tail = tail.
12. ring = folio_address(ctx.ring_folios[0]) as *AioRing.
13. head = ring.head.
14. ring.tail = tail.
15. flush_dcache_folio(ctx.ring_folios[0]).
16. ctx.producer.completed_events += 1.
17. if completed_events > 1: Aio::refill_reqs_avail(ctx, head, tail).
18. avail = if tail > head { tail - head } else { tail + ctx.nr_events - head }.
19. spin_unlock_irqrestore.
20. if iocb.ki_eventfd: eventfd_signal(iocb.ki_eventfd).
21. smp_mb().
22. if waitqueue_active(&ctx.reader.wait):
    - lock wait.lock; for each waiter where avail ≥ waiter.min_nr: wake + list_del; unlock.

`Aio::io_cancel(ctx_id, user_iocb, result) -> i64`:
1. key = get_user(&user_iocb.aio_key).map_err(|_| -EFAULT)?.
2. if key != KIOCB_KEY: return -EINVAL.
3. ctx = Aio::lookup_ioctx(ctx_id).ok_or(-EINVAL)?.
4. obj = user_iocb as u64.
5. ret = -EINVAL.
6. spin_lock_irq(&ctx.cancel.ctx_lock).
7. for kiocb in ctx.cancel.active_reqs:
   - if kiocb.ki_res.obj == obj:
     - ret = (kiocb.ki_cancel.unwrap())(&kiocb.rw).
     - list_del_init(&kiocb.ki_list).
     - break.
8. spin_unlock_irq.
9. if ret == 0: ret = -EINPROGRESS.
10. percpu_ref_put(&ctx.users).
11. return ret.

`Aio::io_getevents(ctx_id, min_nr, nr, events, timeout) -> i64`:
1. ts = if timeout { get_timespec64(timeout)? } else { None }.
2. until = ts.map(timespec64_to_ktime).unwrap_or(KTIME_MAX).
3. ioctx = Aio::lookup_ioctx(ctx_id).ok_or(-EINVAL)?.
4. ret = if min_nr <= nr ∧ min_nr ≥ 0 {
     Aio::read_events_wait(ioctx, min_nr, nr, events, until)
   } else { -EINVAL }.
5. percpu_ref_put(&ioctx.users).
6. if ret == 0 ∧ signal_pending(current): ret = -EINTR.
7. return ret.

`Aio::read_events_wait(ctx, min_nr, nr, events, until) -> i64`:
1. ret = Aio::read_events_ring(ctx, events, nr).
2. if ret ≥ min_nr: return ret.
3. Loop:
   - prepare_to_wait_exclusive(&ctx.reader.wait, &waiter, TASK_INTERRUPTIBLE).
   - waiter.min_nr = min_nr - ret.
   - schedule_hrtimeout_range_clock(&until, ...).
   - finish_wait.
   - if signal_pending: return -EINTR.
   - if hrtimeout expired: break.
   - ret += Aio::read_events_ring(ctx, events + ret, nr - ret).
   - if ret ≥ min_nr: return ret.
4. return ret.

### Out of Scope

- io_uring (fs/io_uring/* — separate Tier-3 `io-uring.md`)
- eventfd backing (covered in `eventfd.md` Tier-3)
- VFS `kiocb` + iov_iter primitives (covered in `vfs/file-rw.md` Tier-3)
- Block-layer `blk_plug` (covered in `block/plug.md` Tier-3)
- Workqueue infrastructure (covered in `kernel/workqueue.md` Tier-3)
- POSIX-AIO glibc shim semantics (user-space, out of kernel scope)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct aio_ring` | per-mmap ring header + io_events[] | `AioRing` |
| `struct kioctx` | per-context state | `Kioctx` |
| `struct kioctx_table` | per-mm RCU array of kioctxs | `KioctxTable` |
| `struct kioctx_cpu` | per-cpu reqs_available cache | `KioctxCpu` |
| `struct aio_kiocb` | per-iocb state (union rw/fsync/poll) | `AioKiocb` |
| `struct fsync_iocb` | per-FSYNC iocb backing | `FsyncIocb` |
| `struct poll_iocb` | per-POLL iocb backing | `PollIocb` |
| `struct aio_inode_info` | per-pseudo-inode kioctx backref | `AioInodeInfo` |
| `ioctx_alloc()` | per-io_setup allocator | `Aio::ioctx_alloc` |
| `aio_setup_ring()` | per-ring page allocation + mmap | `Aio::setup_ring` |
| `kill_ioctx()` | per-io_destroy teardown | `Aio::kill_ioctx` |
| `exit_aio()` | per-mm-exit teardown all ctxs | `Aio::exit_aio` |
| `lookup_ioctx()` | per-id → kioctx + ref-bump | `Aio::lookup_ioctx` |
| `aio_get_req()` | per-iocb slab alloc | `Aio::get_req` |
| `iocb_destroy()` | per-iocb teardown | `Aio::iocb_destroy` |
| `iocb_put()` | per-iocb ref-drop | `Aio::iocb_put` |
| `__io_submit_one()` | per-iocb opcode dispatch | `Aio::submit_one_inner` |
| `io_submit_one()` | per-iocb copy-in + alloc + dispatch | `Aio::submit_one` |
| `aio_read()` / `aio_write()` | per-PREAD(V)/PWRITE(V) | `Aio::read` / `Aio::write` |
| `aio_prep_rw()` | per-RW kiocb prep (fd, pos, flags) | `Aio::prep_rw` |
| `aio_setup_rw()` | per-RW iov_iter import | `Aio::setup_rw` |
| `aio_rw_done()` | per-RW submit-or-defer | `Aio::rw_done` |
| `aio_complete_rw()` | per-RW completion callback | `Aio::complete_rw` |
| `aio_fsync()` | per-(F)(D)SYNC schedule work | `Aio::fsync` |
| `aio_fsync_work()` | per-FSYNC workqueue | `Aio::fsync_work` |
| `aio_poll()` | per-POLL arm wait | `Aio::poll` |
| `aio_poll_wake()` | per-POLL wakeup | `Aio::poll_wake` |
| `aio_poll_cancel()` | per-POLL cancel | `Aio::poll_cancel` |
| `aio_poll_complete_work()` | per-POLL completion path | `Aio::poll_complete_work` |
| `aio_complete()` | per-iocb ring-event publish | `Aio::complete` |
| `kiocb_set_cancel_fn()` | per-driver cancel registration | `Aio::set_cancel_fn` |
| `aio_remove_iocb()` | per-iocb active-list unlink | `Aio::remove_iocb` |
| `aio_read_events_ring()` | per-getevents drain | `Aio::read_events_ring` |
| `aio_read_events()` | per-getevents wrapper | `Aio::read_events` |
| `read_events()` | per-getevents waitable | `Aio::read_events_wait` |
| `do_io_getevents()` | per-getevents kernel-internal | `Aio::do_getevents` |
| `aio_ring_mmap_prepare()` | per-mmap install vm_ops | `Aio::ring_mmap_prepare` |
| `aio_ring_mremap()` | per-mremap fixup user_id | `Aio::ring_mremap` |
| `aio_migrate_folio()` | per-folio migration of ring page | `Aio::migrate_folio` |
| `get_reqs_available()` / `put_reqs_available()` | per-percpu req credits | `Aio::get_reqs_avail` / `put_reqs_avail` |
| `refill_reqs_available()` | per-completion refill | `Aio::refill_reqs_avail` |
| `aio_nr_lock` / `aio_nr` / `aio_max_nr` | per-system request quota | shared |
| `sys_io_setup` / `sys_io_destroy` / `sys_io_submit` / `sys_io_cancel` / `sys_io_getevents` / `sys_io_pgetevents` | per-syscall entry | `SyscallDispatch::*` |

### compatibility contract

REQ-1: struct aio_ring (mmap'd, user-visible):
- id: per-kernel kioctx index (also stored as kioctx.user_id).
- nr: per-ring io_events slots (= kioctx.nr_events).
- head: per-userland write-or-kernel-ring_lock-write; consumer cursor.
- tail: per-kernel-completion-lock write; producer cursor.
- magic = AIO_RING_MAGIC (0xa10a10a1).
- compat_features = AIO_RING_COMPAT_FEATURES (1).
- incompat_features = AIO_RING_INCOMPAT_FEATURES (0).
- header_length = sizeof(struct aio_ring).
- io_events[]: flex-array of struct io_event (per-completion record).

REQ-2: struct kioctx:
- users: percpu_ref — per-context user refcount (drops to 0 → free_ioctx_users).
- dead: atomic_t — per-io_destroy marker.
- reqs: percpu_ref — per-iocb refcount aggregator.
- user_id: per-context id (= kioctx.mmap_base, returned to user).
- cpu: per-cpu kioctx_cpu (reqs_available cache).
- req_batch: per-cpu batch size for reqs_available migration.
- max_reqs: per-user-requested capacity.
- nr_events: per-ring actual capacity (≥ max_reqs+1).
- mmap_base / mmap_size: per-user-mapping address + size.
- ring_folios / nr_pages / internal_folios[AIO_RING_PAGES=8]: per-ring page array.
- aio_ring_file: per-pseudo-file (aio_ring_fops).
- rq_wait: per-io_destroy completion for in-flight drain.
- reqs_available (cacheline): per-ctx atomic credit counter.
- ctx_lock + active_reqs: per-cancellation linked list.
- ring_lock + wait: per-getevents wait-queue.
- tail + completed_events + completion_lock (cacheline): per-aio_complete producer state.
- id: per-mm kioctx_table index.

REQ-3: struct aio_kiocb (union per opcode):
- ki_filp: per-iocb file* (common first member of all union arms).
- rw: per-PREAD/PWRITE struct kiocb (VFS).
- fsync: per-FSYNC fsync_iocb (file + work_struct + datasync + creds).
- poll: per-POLL poll_iocb (file + wait_queue_head + events + cancel-flags + wait_entry + work).
- ki_ctx: per-kioctx backref.
- ki_cancel: per-driver kiocb_cancel_fn pointer (NULL = uncancelable).
- ki_res: per-iocb io_event template (obj/data/res/res2).
- ki_list: per-active_reqs linkage.
- ki_refcnt: per-iocb async refcount.
- ki_eventfd: per-IOCB_FLAG_RESFD eventfd_ctx.

REQ-4: io_setup(nr_events, ctxp):
- /* Validate */
- get_user(ctx, ctxp); if ctx != 0 ∨ nr_events == 0: return -EINVAL.
- /* Allocate kioctx */
- ioctx = ioctx_alloc(nr_events).
- if IS_ERR: return PTR_ERR.
- /* Publish id to user */
- put_user(ioctx.user_id, ctxp).
- if put_user fails: kill_ioctx(current.mm, ioctx, NULL); return -EFAULT.
- percpu_ref_put(&ioctx.users).
- return 0.

REQ-5: ioctx_alloc(nr_events):
- /* Round up to ring-page capacity */
- nr_events *= 2; nr_events += 2.
- /* System quota */
- if aio_nr + nr_events > aio_max_nr ∨ aio_nr + nr_events < aio_nr: return -EAGAIN.
- /* kmem_cache_zalloc kioctx */
- /* aio_setup_ring(ctx, nr_events) — allocates ring_folios, mmap's into current.mm */
- /* percpu_ref_init users + reqs */
- /* alloc_percpu kioctx_cpu */
- /* ioctx_add_table(ctx, current.mm) — RCU-publish into kioctx_table */
- /* aio_nr += ctx.nr_events */
- /* ctx.user_id = ctx.mmap_base */
- return ctx.

REQ-6: aio_setup_ring(ctx, nr_events):
- nr_pages = (sizeof(aio_ring) + nr_events*sizeof(io_event) + PAGE_SIZE-1) / PAGE_SIZE.
- ctx.aio_ring_file = aio_private_file(ctx, nr_pages) — pseudo-fs file with `aio_ctx_aops`.
- For each page: allocate folio, attach to aio_ring_file.f_mapping, store in ring_folios.
- vm_mmap(aio_ring_file, 0, nr_pages * PAGE_SIZE, PROT_READ|PROT_WRITE, MAP_SHARED, 0) → mmap_base.
- Initialize aio_ring header: magic = AIO_RING_MAGIC, nr, head=0, tail=0, etc.

REQ-7: io_destroy(ctx_id):
- ioctx = lookup_ioctx(ctx_id); if !ioctx: return -EINVAL.
- init_completion(&wait.comp); atomic_set(&wait.count, 1).
- ret = kill_ioctx(current.mm, ioctx, &wait).
- percpu_ref_put(&ioctx.users).
- if !ret: wait_for_completion(&wait.comp).
- return ret.

REQ-8: kill_ioctx(mm, ctx, wait):
- /* CAS dead 0→1 */
- if atomic_xchg(&ctx.dead, 1): return -EINVAL.
- /* Remove from kioctx_table (RCU) */
- /* Subtract from aio_nr */
- /* Drop user-ring mmap (munmap mmap_base) */
- ctx.rq_wait = wait.
- percpu_ref_kill(&ctx.users).
- return 0.
- /* When users.percpu_ref → 0: free_ioctx_users → percpu_ref_kill(&reqs) → free_ioctx_reqs → RCU work → free_ioctx (aio_free_ring + complete(wait)) */

REQ-9: io_submit(ctx_id, nr, iocbpp):
- if nr < 0: return -EINVAL.
- ctx = lookup_ioctx(ctx_id); if !ctx: return -EINVAL.
- if nr > ctx.nr_events: nr = ctx.nr_events.
- if nr > AIO_PLUG_THRESHOLD: blk_start_plug(&plug).
- For i in 0..nr:
  - get_user(user_iocb, iocbpp+i); if EFAULT: break.
  - ret = io_submit_one(ctx, user_iocb, false).
  - if ret: break.
- if nr > AIO_PLUG_THRESHOLD: blk_finish_plug(&plug).
- percpu_ref_put(&ctx.users).
- return i ? i : ret.

REQ-10: io_submit_one(ctx, user_iocb, compat):
- copy_from_user(&iocb, user_iocb, sizeof iocb) — fault → -EFAULT.
- if iocb.aio_reserved2 != 0: return -EINVAL.
- /* overflow checks: aio_buf, aio_nbytes */
- req = aio_get_req(ctx); if !req: return -EAGAIN.
- /* __io_submit_one */
- req.ki_filp = fget(iocb.aio_fildes); if !filp: return -EBADF.
- if iocb.aio_flags & IOCB_FLAG_RESFD: req.ki_eventfd = eventfd_ctx_fdget(iocb.aio_resfd).
- put_user(KIOCB_KEY, &user_iocb.aio_key) — fault → -EFAULT.
- Initialize req.ki_res = { .obj = user_iocb, .data = iocb.aio_data, .res = 0, .res2 = 0 }.
- switch iocb.aio_lio_opcode:
  - IOCB_CMD_PREAD: aio_read(&req.rw, &iocb, false, compat).
  - IOCB_CMD_PWRITE: aio_write(&req.rw, &iocb, false, compat).
  - IOCB_CMD_PREADV: aio_read(&req.rw, &iocb, true, compat).
  - IOCB_CMD_PWRITEV: aio_write(&req.rw, &iocb, true, compat).
  - IOCB_CMD_FSYNC: aio_fsync(&req.fsync, &iocb, false).
  - IOCB_CMD_FDSYNC: aio_fsync(&req.fsync, &iocb, true).
  - IOCB_CMD_POLL: aio_poll(req, &iocb).
  - IOCB_CMD_NOOP: ignored (return 0, no completion).
  - default: -EINVAL.
- iocb_put(req) — drop synchronous reference.
- If err: iocb_destroy(req); put_reqs_available(ctx, 1).

REQ-11: aio_complete(iocb):
- /* Producer: write event into ring under completion_lock */
- spin_lock_irqsave(&ctx.completion_lock).
- tail = ctx.tail.
- pos = tail + AIO_EVENTS_OFFSET (skip ring-header slots).
- if ++tail >= ctx.nr_events: tail = 0.
- event = ring_folios[pos/per_page] + pos%per_page.
- *event = iocb.ki_res (copy obj/data/res/res2).
- flush_dcache_folio.
- smp_wmb() — event visible before tail bump.
- ctx.tail = tail.
- ring (folio_address ring_folios[0]).tail = tail.
- ctx.completed_events++.
- if completed_events > 1: refill_reqs_available(ctx, ring.head, tail).
- spin_unlock_irqrestore.
- if iocb.ki_eventfd: eventfd_signal.
- smp_mb().
- if waitqueue_active(&ctx.wait): wake matching aio_waiter entries (min_nr satisfied).

REQ-12: io_cancel(ctx_id, iocb, result):
- get_user(key, &iocb.aio_key); if key != KIOCB_KEY: return -EINVAL.
- ctx = lookup_ioctx(ctx_id); if !ctx: return -EINVAL.
- obj = (u64)iocb.
- spin_lock_irq(&ctx.ctx_lock).
- list_for_each_entry(kiocb, &ctx.active_reqs, ki_list):
  - if kiocb.ki_res.obj == obj:
    - ret = kiocb.ki_cancel(&kiocb.rw).
    - list_del_init(&kiocb.ki_list).
    - break.
- spin_unlock_irq.
- if !ret: ret = -EINPROGRESS (event still delivered via ring).
- percpu_ref_put(&ctx.users).
- return ret.

REQ-13: io_getevents(ctx_id, min_nr, nr, events, timeout):
- if timeout: get_timespec64(&ts, timeout) → -EFAULT.
- ret = do_io_getevents(ctx_id, min_nr, nr, events, ts ? &ts : NULL).
  - ioctx = lookup_ioctx(ctx_id); if !ioctx: -EINVAL.
  - if min_nr <= nr ∧ min_nr ≥ 0: ret = read_events(ioctx, min_nr, nr, events, until).
  - percpu_ref_put(&ioctx.users).
- if !ret ∧ signal_pending(current): ret = -EINTR.
- return ret.

REQ-14: read_events / aio_read_events_ring:
- ring_lock (mutex).
- Read ring.head, ctx.tail.
- avail = events ready = (tail - head) mod nr_events.
- For each consumable: copy_to_user io_event to user `events` buffer, advance head.
- ring.head = head; flush_dcache_folio.
- If still < min_nr: prepare aio_waiter on ctx.wait, schedule_hrtimeout_range, retry.

REQ-15: aio_ring mmap:
- aio_ring_fops.mmap_prepare = aio_ring_mmap_prepare:
  - desc.vm_ops = &aio_ring_vm_ops (with .mremap = aio_ring_mremap).
- aio_ring_mremap: on mremap, rewrite kioctx.user_id, kioctx.mmap_base to new address; update kioctx_table index.
- aio_ctx_aops.migrate_folio = aio_migrate_folio: per-folio migration uses migrate_lock + remap-pages-via-kioctx.

REQ-16: IOCB_FLAG_RESFD eventfd delivery:
- On submit: req.ki_eventfd = eventfd_ctx_fdget(iocb.aio_resfd).
- On completion: eventfd_signal(req.ki_eventfd) after ring publish.

REQ-17: Per-system quota:
- aio_nr_lock + aio_nr + aio_max_nr (sysctl fs.aio-nr / fs.aio-max-nr; default 0x10000).
- Per-ioctx_alloc must atomically check `aio_nr + nr_events > aio_max_nr` before commit.
- On kill_ioctx: aio_nr_sub(nr_events).

REQ-18: KIOCB_KEY:
- Sentinel u32 written to user_iocb.aio_key on submit; checked on cancel.
- Protects against userspace passing arbitrary iocb pointers to io_cancel.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ring_magic_constant` | INVARIANT | per-setup_ring: ring.magic == AIO_RING_MAGIC after initialization. |
| `ring_head_tail_bounded` | INVARIANT | per-aio_complete: tail ∈ [0, nr_events); per-aio_read_events_ring: head ∈ [0, nr_events). |
| `aio_nr_quota_not_exceeded` | INVARIANT | per-ioctx_alloc: aio_nr + reserved ≤ aio_max_nr at commit. |
| `kiocb_refcnt_balanced` | INVARIANT | per-submit_one: refcnt get → eventual put (sync or async) — no leak. |
| `completion_lock_held_in_aio_complete` | INVARIANT | per-aio_complete: completion_lock held when writing tail. |
| `kiocb_key_validated_in_cancel` | INVARIANT | per-io_cancel: rejects iocb where aio_key != KIOCB_KEY. |
| `eventfd_signaled_after_ring_publish` | ORDERING | per-aio_complete: eventfd_signal happens after smp_wmb + tail update. |

### Layer 2: TLA+

`fs/aio.tla`:
- Per-ring producer (aio_complete) + per-ring consumer (aio_read_events_ring).
- Properties:
  - `safety_no_overwrite_unconsumed` — per-aio_complete: refuses to advance tail when head == (tail+1) mod nr_events (ring full).
  - `safety_no_phantom_event` — per-aio_read_events_ring: never returns an event slot not yet written.
  - `safety_user_id_unique_per_mm` — per-ioctx_alloc: user_id (mmap_base) unique within mm.
  - `safety_killed_ctx_cannot_submit` — per-ctx.dead==1: lookup_ioctx returns NULL, all subsequent submits/cancels fail with -EINVAL.
  - `liveness_io_destroy_completes` — per-kill_ioctx: percpu_ref_kill eventually drains → free_ioctx → completion fired.
  - `liveness_getevents_terminates` — per-read_events_wait: signal OR ≥ min_nr completions OR timeout.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Aio::io_setup` post: ret==0 ⟹ ctxp.read == ioctx.user_id ∈ current.mm.kioctx_table | `Aio::io_setup` |
| `Aio::ioctx_alloc` post: aio_nr advanced; users refcnt = 2 (caller + table) | `Aio::ioctx_alloc` |
| `Aio::submit_one` post: ret==0 ⟹ iocb dispatched OR completion already queued | `Aio::submit_one` |
| `Aio::complete` post: ring.tail advanced by exactly 1; event written before tail visible | `Aio::complete` |
| `Aio::io_cancel` post: ret==-EINPROGRESS ⟹ removed from active_reqs | `Aio::io_cancel` |
| `Aio::read_events_ring` post: ret events copied to user; ring.head advanced by ret | `Aio::read_events_ring` |
| `Aio::kill_ioctx` post: ctx.dead==1; aio_nr decreased; wait queued to fire after free | `Aio::kill_ioctx` |

### Layer 4: Verus/Creusot functional

`Per-syscall sequence: io_setup → io_submit → aio_complete → io_getevents → io_destroy` semantic equivalence: per-`io_setup(2)` / `io_submit(2)` / `io_getevents(2)` / `io_cancel(2)` / `io_destroy(2)` man-pages + LTP `testcases/kernel/syscalls/io_*` reference traces.

### hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

AIO reinforcement:

- **Per-KIOCB_KEY sentinel validated in io_cancel** — defense against per-user-supplied bogus iocb pointer used to cancel arbitrary kernel iocbs.
- **Per-aio_nr global quota enforced under aio_nr_lock** — defense against per-user OOM via spammed io_setup.
- **Per-ring mmap MAP_SHARED PROT_READ|WRITE only** — defense against per-mremap to executable region; aio_ring_mremap fixes up user_id rather than allowing free relocation.
- **Per-completion-lock IRQ-safe (spin_lock_irqsave)** — defense against per-aio_complete-from-IRQ vs reader race.
- **Per-percpu_ref users/reqs strict** — defense against per-use-after-free during concurrent io_destroy + io_submit.
- **Per-ctx.dead atomic gate** — defense against per-double-destroy.
- **Per-active_reqs list under ctx_lock** — defense against per-cancel during completion race.
- **Per-iocb refcnt 2-stage (sync ref + async ref)** — defense against per-iocb destroyed mid-submission.
- **Per-aio_reserved2 must be zero** — forward-compat fence; defense against per-future-flag silently honored on older kernel.
- **Per-overflow check on aio_buf / aio_nbytes** — defense against per-ssize_t overflow into negative.
- **Per-eventfd_signal after smp_wmb** — defense against per-stale-event visible to eventfd reader before ring publish.
- **Per-folio migration via aio_migrate_folio under migrate_lock** — defense against per-page-migration corrupting user mapping.
- **Per-exit_aio walks kioctx_table on mm teardown** — defense against per-leak when process dies with open kioctxs.

