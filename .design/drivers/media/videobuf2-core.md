# Tier-3: drivers/media/common/videobuf2/videobuf2-core.c — VB2 framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/media/00-overview.md
upstream-paths:
  - drivers/media/common/videobuf2/videobuf2-core.c (~3308 lines)
  - include/media/videobuf2-core.h
  - include/media/videobuf2-v4l2.h
  - include/media/videobuf2-dma-contig.h
  - include/media/videobuf2-dma-sg.h
  - include/media/videobuf2-vmalloc.h
  - include/media/videobuf2-memops.h
-->

## Summary

**Videobuf2 (VB2)** is the V4L2 buffer-queue framework. Per-`struct vb2_queue` owns a fixed-size array of `struct vb2_buffer *` (q.bufs[]) up to q.max_num_buffers indexed by a bitmap (q.bufs_bitmap). Per-`vb2_core_reqbufs` / `vb2_core_create_bufs` ask the driver via `q.ops->queue_setup` for (num_buffers, num_planes, plane_sizes[]) then allocate per-plane mem_priv via `q.mem_ops->alloc / get_userptr / attach_dmabuf`. Per-`vb2_buffer.state` machine: DEQUEUED → PREPARING → PREPARED → QUEUED → ACTIVE → DONE | ERROR → DEQUEUED (DQBUF closes the loop). Per-`vb2_core_qbuf` validates the request-API mode and either binds the buffer to a media_request or appends to q.queued_list, then `__enqueue_in_driver` if q.start_streaming_called. Per-`vb2_core_streamon` calls `prepare_streaming` then `vb2_start_streaming` (calls driver `start_streaming(q, count)` and on failure recycles all queued buffers back to QUEUED). Per-`vb2_core_streamoff` invokes `__vb2_queue_cancel`: stop_streaming on driver, drain owned_by_drv_count, wipe queued_list + done_list, mark every buffer DEQUEUED. Per-`vb2_buffer_done` flips ACTIVE → DONE/ERROR/QUEUED under q.done_lock and wakes q.done_wq pollers + DQBUF waiters. Per-three memory models: MMAP (alloc via mem_ops.alloc, exposed via vb2_mmap), USERPTR (get_userptr pins user pages), DMABUF (attach_dmabuf + map_dmabuf). Per-`vb2_core_expbuf` exports a plane as a dma-buf fd; per-`vb2_core_poll` integrates with file->poll. Critical for: capture/playback throughput, zero-copy DMA-buf interop, stream restart correctness, fileio fallback (read/write emulation), m2m codecs.

This Tier-3 covers `drivers/media/common/videobuf2/videobuf2-core.c` (~3308 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vb2_queue` | per-queue context | `Vb2Queue` |
| `struct vb2_buffer` | per-buffer instance | `Vb2Buffer` |
| `struct vb2_plane` | per-plane storage | `Vb2Plane` |
| `enum vb2_buffer_state` | state machine | `Vb2BufferState` |
| `enum vb2_memory` | MMAP/USERPTR/DMABUF | `Vb2Memory` |
| `struct vb2_mem_ops` | mem-vtable | `Vb2MemOps` |
| `struct vb2_ops` | driver-vtable | `Vb2Ops` |
| `struct vb2_buf_ops` | buf-ops vtable | `Vb2BufOps` |
| `vb2_core_queue_init` | per-init | `Vb2::queue_init` |
| `vb2_core_queue_release` | per-release | `Vb2::queue_release` |
| `vb2_core_reqbufs` | per-REQBUFS | `Vb2::reqbufs` |
| `vb2_core_create_bufs` | per-CREATE_BUFS | `Vb2::create_bufs` |
| `vb2_core_remove_bufs` | per-REMOVE_BUFS | `Vb2::remove_bufs` |
| `vb2_core_querybuf` | per-QUERYBUF | `Vb2::querybuf` |
| `vb2_core_prepare_buf` | per-PREPARE_BUF | `Vb2::prepare_buf` |
| `vb2_core_qbuf` | per-QBUF | `Vb2::qbuf` |
| `vb2_core_dqbuf` | per-DQBUF | `Vb2::dqbuf` |
| `vb2_core_streamon` | per-STREAMON | `Vb2::streamon` |
| `vb2_core_streamoff` | per-STREAMOFF | `Vb2::streamoff` |
| `vb2_core_expbuf` | per-EXPBUF | `Vb2::expbuf` |
| `vb2_core_poll` | per-poll | `Vb2::poll` |
| `vb2_mmap` | per-mmap | `Vb2::mmap` |
| `vb2_get_unmapped_area` | NOMMU helper | `Vb2::get_unmapped_area` |
| `vb2_buffer_done` | per-buffer-completion | `Vb2::buffer_done` |
| `vb2_discard_done` | per-flush done-list | `Vb2::discard_done` |
| `vb2_wait_for_all_buffers` | per-drain | `Vb2::wait_for_all_buffers` |
| `vb2_plane_vaddr` | kvaddr accessor | `Vb2::plane_vaddr` |
| `vb2_plane_cookie` | per-cookie accessor | `Vb2::plane_cookie` |
| `vb2_buffer_in_use` | per-mmap-pinned test | `Vb2::buffer_in_use` |
| `vb2_verify_memory_type` | per-IO-mode verify | `Vb2::verify_memory_type` |
| `vb2_queue_error` | per-fatal-mark | `Vb2::queue_error` |
| `vb2_start_streaming` | inner streamon | `Vb2::start_streaming` |
| `__vb2_queue_cancel` | per-cancel | `Vb2::queue_cancel` |
| `__vb2_queue_alloc` | per-alloc | `Vb2::queue_alloc` |
| `__vb2_queue_free` | per-free | `Vb2::queue_free` |
| `__buf_prepare` | per-prepare | `Vb2::buf_prepare` |
| `__enqueue_in_driver` | per-driver-handoff | `Vb2::enqueue_in_driver` |
| `vb2_thread_start` / `_stop` | kthread fileio | `Vb2::thread_start` / `stop` |
| `vb2_request_object_is_buffer` | per-req-type test | `Vb2::request_object_is_buffer` |
| `vb2_core_req_ops` | per-media-req vtable | `VB2_CORE_REQ_OPS` |

## Compatibility contract

REQ-1: struct vb2_queue (queue-wide state):
- type: V4L2_BUF_TYPE_*.
- io_modes: VB2_MMAP | VB2_USERPTR | VB2_DMABUF | VB2_READ | VB2_WRITE.
- dev: struct device * (DMA-mapping owner).
- dma_dir: DMA_TO_DEVICE / FROM_DEVICE / BIDIRECTIONAL (derived from is_output + bidirectional).
- ops: *vb2_ops (driver vtable: queue_setup, buf_init, buf_prepare, buf_finish, buf_cleanup, buf_queue, buf_request_complete, prepare_streaming, start_streaming, stop_streaming, unprepare_streaming).
- mem_ops: *vb2_mem_ops (alloc, put, get_userptr, put_userptr, attach_dmabuf, detach_dmabuf, map_dmabuf, unmap_dmabuf, prepare, finish, vaddr, cookie, mmap, get_dmabuf).
- buf_ops: *vb2_buf_ops (verify_planes_array, init_buffer, fill_user_buffer, fill_vb2_buffer, copy_timestamp).
- drv_priv, lock (caller-side serialization mutex), mmap_lock, done_lock (spinlock for done_list).
- queued_list, done_list, done_wq (wait_queue_head_t).
- bufs: array (kzalloc_objs); bufs_bitmap: bitmap; max_num_buffers; min_queued_buffers; min_reqbufs_allocation.
- num_buffers (counter), queued_count, owned_by_drv_count (atomic_t).
- memory: enum vb2_memory.
- is_output, is_busy, streaming, error, waiting_for_buffers, last_buffer_dequeued, uses_qbuf, uses_requests, requires_requests, supports_requests, start_streaming_called, waiting_in_dqbuf, allow_cache_hints, non_coherent_mem, bidirectional.
- buf_struct_size (>= sizeof(vb2_buffer)).
- alloc_devs[VB2_MAX_PLANES].
- name[32].

REQ-2: struct vb2_buffer (per-buffer):
- vb2_queue (back-pointer), index, type, memory, num_planes.
- planes[VB2_MAX_PLANES]: {bytesused, length, m.{offset,userptr,fd}, data_offset, mem_priv, dbuf, dbuf_mapped, dbuf_duplicated}.
- state: enum vb2_buffer_state.
- queued_entry, done_entry: list_head.
- timestamp (ktime), request_fd.
- req_obj (media_request_object), request (back-pointer).
- prepared, synced, skip_cache_sync_on_prepare, skip_cache_sync_on_finish, copied_timestamp.
- cnt_* (CONFIG_VIDEO_ADV_DEBUG counters: cnt_mem_alloc / put / mmap, cnt_buf_init / prepare / queue / finish / cleanup / done).

REQ-3: enum vb2_buffer_state:
- VB2_BUF_STATE_DEQUEUED (0) — userspace owns.
- VB2_BUF_STATE_IN_REQUEST — queued via media-request, not yet active.
- VB2_BUF_STATE_PREPARING — __buf_prepare in flight.
- VB2_BUF_STATE_QUEUED — VB2 owns, awaiting driver.
- VB2_BUF_STATE_ACTIVE — driver owns (between buf_queue and buffer_done).
- VB2_BUF_STATE_DONE — VB2 owns, awaiting dequeue.
- VB2_BUF_STATE_ERROR — VB2 owns, errored, awaiting dequeue.

REQ-4: vb2_core_queue_init(q):
- max_num_buffers ||= VB2_MAX_FRAME; clamp to MAX_BUFFER_INDEX.
- WARN+EINVAL: !q | !q->ops | !q->mem_ops | !q->type | !q->io_modes | !ops->queue_setup | !ops->buf_queue.
- WARN+EINVAL: max_num_buffers < VB2_MAX_FRAME | min_queued_buffers > max_num_buffers.
- WARN+EINVAL: requires_requests ∧ !supports_requests.
- WARN+EINVAL: supports_requests ∧ min_queued_buffers (requests must always succeed on queue).
- min_reqbufs_allocation = max(min_reqbufs_allocation, min_queued_buffers + 1).
- WARN+EINVAL: min_reqbufs_allocation > max_num_buffers.
- WARN+EINVAL: !q->lock.
- INIT_LIST_HEAD(&queued_list); INIT_LIST_HEAD(&done_list); spin_lock_init(&done_lock); mutex_init(&mmap_lock); init_waitqueue_head(&done_wq).
- memory = VB2_MEMORY_UNKNOWN.
- buf_struct_size ||= sizeof(vb2_buffer).
- dma_dir = bidirectional ? BIDIRECTIONAL : (is_output ? TO_DEVICE : FROM_DEVICE).
- name ||= snprintf("cap-%p" or "out-%p").

REQ-5: vb2_verify_memory_type(q, memory, type):
- type must equal q->type; memory ∈ {MMAP, USERPTR, DMABUF} ∧ q->io_modes must support it.
- VB2_MEMORY_MMAP ⟹ q->mem_ops must implement {alloc, put, mmap}.
- VB2_MEMORY_USERPTR ⟹ {get_userptr, put_userptr}.
- VB2_MEMORY_DMABUF ⟹ {attach_dmabuf, detach_dmabuf, map_dmabuf, unmap_dmabuf}.
- -EBUSY if vb2_fileio_is_active(q).

REQ-6: vb2_core_reqbufs(q, memory, flags, *count):
- -EBUSY if streaming or another dup()'d fd is waiting_in_dqbuf with *count > 0.
- If *count==0 or num_bufs!=0 or memory model changing or coherency mismatch:
  - mutex_lock(mmap_lock); __vb2_queue_cancel(q); __vb2_queue_free(q, 0, max_num_buffers); mutex_unlock.
  - q->is_busy = 0; if *count==0 return 0.
- num_buffers = clamp(*count, min_reqbufs_allocation, max_num_buffers).
- mutex_lock(mmap_lock); allocate bufs[] + bufs_bitmap; q->memory = memory; mutex_unlock.
- set_queue_coherency(q, non_coherent_mem).
- call_qop(queue_setup, q, &num_buffers, &num_planes, plane_sizes, alloc_devs).
- WARN_ON+EINVAL: !num_planes; ∀i plane_sizes[i] != 0.
- allocated = __vb2_queue_alloc(...); if allocated < min_reqbufs_allocation: -ENOMEM rollback.
- If allocated < num_buffers: re-call queue_setup with num_planes=0; ret -ENOMEM if still short.
- *count = allocated; waiting_for_buffers = !is_output; is_busy = 1.

REQ-7: vb2_core_create_bufs(q, memory, flags, *count, requested_planes, requested_sizes[], *first_index):
- -ENOBUFS if at max_num_buffers.
- If no_previous_buffers: same memory-model init as reqbufs; else verify memory model + coherency match.
- num_buffers = min(*count, max - num).
- call_qop(queue_setup); __vb2_queue_alloc; if short retry once.
- *count = allocated; is_busy = 1.

REQ-8: __vb2_queue_alloc(q, memory, num_buffers, num_planes, plane_sizes, *first_index):
- For each new vb:
  - vb = kzalloc(q->buf_struct_size).
  - vb->state = VB2_BUF_STATE_DEQUEUED.
  - vb->num_planes = num_planes.
  - vb->type = q->type; vb->memory = memory.
  - For each plane: planes[i].length = plane_sizes[i].
  - init_buffer_cache_hints(q, vb).
  - vb2_queue_add_buffer(q, vb, next_free_index in q->bufs_bitmap).
  - call_vb_qop(buf_init, vb).
  - if MMAP: __vb2_buf_mem_alloc (per-plane alloc via mem_ops.alloc).
  - __setup_offsets(vb): plane.m.offset = (index << PLANE_INDEX_SHIFT) | (plane << PAGE_SHIFT).
- On failure: free pre-allocated planes (in reverse), free vb.

REQ-9: vb2_core_qbuf(q, vb, pb, req) — state transitions:
- if q->error: -EIO.
- if !req ∧ vb->state != IN_REQUEST ∧ q->requires_requests: -EBADR.
- Reject mixed-mode: req+uses_qbuf or !req+uses_requests after first call.
- If req != NULL:
  - q->uses_requests = 1.
  - require vb->state == DEQUEUED; else -EINVAL.
  - if is_output ∧ !vb->prepared: call_vb_qop(buf_out_validate, vb).
  - media_request_object_init(&vb->req_obj); media_request_lock_for_update(req); media_request_object_bind(req, &vb2_core_req_ops, q, true, &vb->req_obj); media_request_unlock_for_update.
  - vb->state = IN_REQUEST; media_request_get(req); vb->request = req.
  - if pb: copy_timestamp; fill_user_buffer.
  - return 0.
- Non-request path: q->uses_qbuf = 1 (if not already IN_REQUEST).
- switch vb->state:
  - DEQUEUED | IN_REQUEST: if !prepared: __buf_prepare(vb).
  - PREPARING: -EINVAL.
  - default: -EINVAL.
- orig_state = vb->state; list_add_tail(&vb->queued_entry, &q->queued_list); q->queued_count++; waiting_for_buffers = false; vb->state = QUEUED.
- if pb: copy_timestamp; trace_vb2_qbuf.
- if q->start_streaming_called: __enqueue_in_driver(vb).
- if pb: fill_user_buffer.
- if q->streaming ∧ !start_streaming_called ∧ queued_count >= min_queued_buffers: vb2_start_streaming(q); on failure: rollback queued_entry/state.

REQ-10: __buf_prepare(vb) — transitions to PREPARED:
- if q->error: -EIO.
- if vb->prepared: return 0.
- vb->state = PREPARING.
- switch memory:
  - MMAP: __prepare_mmap(vb).
  - USERPTR: __prepare_userptr(vb) (may release+reacquire pages if userptr / length changed; resets mem_priv via get_userptr).
  - DMABUF: __prepare_dmabuf(vb) (acquire fd via dma_buf_get, attach + map if not already; validate length).
- __vb2_buf_mem_prepare(vb): per-plane mem_ops.prepare (cache sync) unless skip_cache_sync_on_prepare.
- call_vb_qop(buf_prepare, vb); on failure: state = DEQUEUED; return -EINVAL.
- vb->state = DEQUEUED (caller transitions to QUEUED via qbuf).
- vb->prepared = 1.

REQ-11: __enqueue_in_driver(vb):
- atomic_inc(&q->owned_by_drv_count).
- trace_vb2_buf_queue.
- vb->state = ACTIVE.
- call_void_vb_qop(buf_queue, vb).

REQ-12: vb2_start_streaming(q):
- /* Queue all already-pending bufs into driver */
- list_for_each_entry(vb, &q->queued_list, queued_entry): __enqueue_in_driver(vb).
- q->start_streaming_called = 1.
- ret = call_qop(start_streaming, q, queued_count).
- On success: return 0.
- On failure: q->start_streaming_called = 0.
  - Forcefully reclaim every ACTIVE buffer via vb2_buffer_done(vb, QUEUED).
  - WARN_ON(atomic_read(&q->owned_by_drv_count)).
  - WARN_ON(!list_empty(&q->done_list)).
- return ret.

REQ-13: vb2_core_streamon(q, type):
- type must equal q->type; else -EINVAL.
- Already streaming: 0.
- q_num_bufs == 0: -EINVAL.
- q_num_bufs < min_queued_buffers: -EINVAL.
- call_qop(prepare_streaming, q).
- if queued_count >= min_queued_buffers: vb2_start_streaming(q); on err: call_void_qop(unprepare_streaming); return err.
- q->streaming = 1; return 0.

REQ-14: vb2_core_streamoff(q, type):
- type must equal q->type.
- __vb2_queue_cancel(q).
- waiting_for_buffers = !is_output; last_buffer_dequeued = false.

REQ-15: __vb2_queue_cancel(q):
- if start_streaming_called: call_void_qop(stop_streaming, q).
- if streaming: call_void_qop(unprepare_streaming, q).
- WARN if any ACTIVE buffer remains; vb2_buffer_done(vb, ERROR) to recycle.
- q->streaming = 0; start_streaming_called = 0; queued_count = 0; error = 0; uses_requests = 0; uses_qbuf = 0.
- INIT_LIST_HEAD(&queued_list); INIT_LIST_HEAD(&done_list).
- atomic_set(&owned_by_drv_count, 0); wake_up_all(&done_wq).
- For each vb in bufs: __vb2_buf_mem_finish; if prepared: buf_finish; __vb2_dqbuf; clear req_obj/request; copied_timestamp=0.

REQ-16: vb2_buffer_done(vb, state):
- WARN_ON if vb->state != ACTIVE.
- WARN+coerce-to-ERROR if state ∉ {DONE, ERROR, QUEUED}.
- spin_lock_irqsave(&q->done_lock).
- if state != QUEUED: __vb2_buf_mem_finish; list_add_tail(&vb->done_entry, &q->done_list); vb->state = state.
- else: list_add(&vb->queued_entry, &q->queued_list_head /* prepend */); vb->state = QUEUED.
- atomic_dec(&owned_by_drv_count).
- trace_vb2_buf_done.
- spin_unlock_irqrestore.
- if state ∈ {DONE, ERROR}: wake_up(&q->done_wq).

REQ-17: vb2_core_dqbuf(q, *pindex, pb, nonblocking):
- __vb2_get_done_vb(q, &vb, pb, nonblocking) — waits on q->done_wq if blocking; returns -EAGAIN if nonblocking and empty; -EPIPE if last_buffer_dequeued; -EIO if q->error; -EBUSY if waiting_in_dqbuf.
- valid vb->state ∈ {DONE, ERROR}; else -EINVAL.
- call_void_vb_qop(buf_finish, vb); vb->prepared = 0.
- *pindex = vb->index; if pb: fill_user_buffer.
- list_del(&vb->queued_entry); queued_count--.
- __vb2_dqbuf(vb): vb->state = DEQUEUED; init_buffer.
- WARN_ON(vb->req_obj.req); unbind+put req_obj; media_request_put(vb->request); vb->request = NULL.

REQ-18: vb2_mmap(q, vma):
- VM_SHARED required; VM_WRITE if is_output, VM_READ otherwise.
- mutex_lock(&mmap_lock).
- __find_plane_by_offset(q, vma->vm_pgoff << PAGE_SHIFT, &vb, &plane): validate MMAP memory; not fileio; decode buffer index + plane index from offset cookie.
- length = PAGE_ALIGN(plane.length); reject if vma length > buffer length.
- vma->vm_pgoff = 0 (mmap always at offset 0 of plane).
- call_memop(mmap, vb, plane.mem_priv, vma).

REQ-19: vb2_core_expbuf(q, *fd, type, vb, plane, flags):
- q->memory must be MMAP; mem_ops.get_dmabuf required.
- flags ⊆ {O_CLOEXEC | O_ACCMODE}.
- plane < vb->num_planes.
- !vb2_fileio_is_active(q).
- dbuf = mem_ops.get_dmabuf(vb, plane.mem_priv, flags & O_ACCMODE).
- *fd = dma_buf_fd(dbuf, flags & ~O_ACCMODE); on err: dma_buf_put.

REQ-20: vb2_core_poll(q, file, wait):
- poll_wait(file, &q->done_wq, wait).
- Direction-filter: !is_output ∧ !(events & (POLLIN|POLLRDNORM)) ⟹ 0; symmetric for is_output + POLLOUT.
- If no buffers + !fileio + matching io_mode: __vb2_init_fileio(q, read=!is_output) — initialize fileio emulation; OUTPUT write returns EPOLLOUT|EPOLLWRNORM immediately.
- If !q->streaming or q->error: EPOLLERR.
- Spin done_lock; if any DONE/ERROR vb at head: return (is_output ? POLLOUT|POLLWRNORM : POLLIN|POLLRDNORM).

REQ-21: Memory models (mem_ops):
- dma-contig (videobuf2-dma-contig.c): contiguous DMA buffer (dma_alloc_attrs); good for ARM/IOMMU-bypass; mmap via dma_mmap_attrs.
- dma-sg (videobuf2-dma-sg.c): scatter-gather; per-plane sg_table; mmap via remap_pfn_range over each page.
- vmalloc (videobuf2-vmalloc.c): vmalloc() backed; mmap via vm_iomap_memory.
- All three implement the full vb2_mem_ops vtable (alloc, put, get_userptr, put_userptr, attach_dmabuf, ...).

REQ-22: Media-request integration (vb2_core_req_ops):
- prepare: validate q ∧ vb state == IN_REQUEST.
- unprepare: __vb2_dqbuf.
- queue: hand off to __enqueue_in_driver (state = ACTIVE).
- unbind: clear vb->req_obj.req.
- release: media_request_object cleanup.
- vb2_request_object_is_buffer / vb2_request_buffer_cnt: introspection helpers.

REQ-23: Plane offset cookie encoding:
- Bits [PLANE_INDEX_BITS-1:0] = plane index (3 bits ⟹ ≤ 8 planes).
- Bits [29:PLANE_INDEX_SHIFT] = buffer index (PLANE_INDEX_SHIFT = PAGE_SHIFT + 3).
- Bit 30 reserved (DST_QUEUE_OFF_BASE distinguishes M2M source vs destination).
- MAX_BUFFER_INDEX = BIT_MASK(30 - PLANE_INDEX_SHIFT).

REQ-24: fileio emulation (__vb2_init_fileio / __vb2_perform_fileio / __vb2_cleanup_fileio):
- Allocates internal vb2_fileio_data with vb2_fileio_buf[VB2_MAX_FRAME].
- Issues REQBUFS internally then synchronous QBUF/DQBUF cycles around read()/write() copies.
- Disabled once any other streaming-class ioctl is issued.

REQ-25: vb2_thread (kthread fileio):
- vb2_thread_start(q, fnc, priv, thread_name): kthread that loops dqbuf → fnc → qbuf for the lifetime of the queue.
- vb2_thread_stop(q): signal thread; wait; cleanup.

## Acceptance Criteria

- [ ] AC-1: vb2_core_queue_init rejects q with !ops->queue_setup or !ops->buf_queue.
- [ ] AC-2: vb2_core_reqbufs returns -EBUSY when q->streaming.
- [ ] AC-3: REQBUFS(0) followed by REQBUFS(N>0): N buffers allocated, *count == N (or min_reqbufs_allocation).
- [ ] AC-4: CREATE_BUFS at max_num_buffers returns -ENOBUFS.
- [ ] AC-5: vb2_core_qbuf with q->error returns -EIO.
- [ ] AC-6: vb2_core_qbuf without prior request, when q->requires_requests, returns -EBADR.
- [ ] AC-7: vb2_core_qbuf transitions state: DEQUEUED → PREPARED (via __buf_prepare) → QUEUED → ACTIVE (after __enqueue_in_driver if streaming).
- [ ] AC-8: vb2_core_streamon with q_num_bufs == 0 returns -EINVAL.
- [ ] AC-9: vb2_core_streamoff drains owned_by_drv_count to 0 and wakes done_wq.
- [ ] AC-10: vb2_buffer_done WARN+coerces invalid state to VB2_BUF_STATE_ERROR.
- [ ] AC-11: vb2_buffer_done(QUEUED) puts vb back on queued_list_head, does NOT wake done_wq.
- [ ] AC-12: vb2_buffer_done(DONE | ERROR) wakes done_wq pollers and DQBUF waiters.
- [ ] AC-13: vb2_core_dqbuf nonblocking with empty done_list returns -EAGAIN.
- [ ] AC-14: vb2_core_dqbuf clears vb->prepared after buf_finish.
- [ ] AC-15: vb2_mmap requires VM_SHARED ∧ direction-matched VM_READ/VM_WRITE.
- [ ] AC-16: vb2_core_expbuf rejects non-MMAP queue with -EINVAL.
- [ ] AC-17: vb2_start_streaming failure recycles every ACTIVE buffer to QUEUED via vb2_buffer_done.
- [ ] AC-18: __vb2_queue_cancel zeros owned_by_drv_count and empties queued_list + done_list.
- [ ] AC-19: vb2_core_poll integrates poll_wait with q->done_wq on first call.

## Architecture

```
struct Vb2Queue {
    typ: u32,                          // V4L2_BUF_TYPE_*
    io_modes: u32,                     // VB2_MMAP | _USERPTR | _DMABUF | _READ | _WRITE
    dev: NonNull<Device>,
    dma_dir: DmaDataDirection,
    ops: &'static Vb2Ops,
    mem_ops: &'static Vb2MemOps,
    buf_ops: Option<&'static Vb2BufOps>,
    drv_priv: *mut u8,
    lock: NonNull<Mutex>,              // caller-side serialization
    mmap_lock: Mutex,
    done_lock: Spinlock,
    queued_list: List<Vb2Buffer>,
    queued_count: u32,
    done_list: List<Vb2Buffer>,
    done_wq: WaitQueueHead,
    bufs: Option<Box<[Option<Box<Vb2Buffer>>]>>,
    bufs_bitmap: Option<Box<BitArray>>,
    max_num_buffers: u32,
    min_queued_buffers: u32,
    min_reqbufs_allocation: u32,
    owned_by_drv_count: AtomicU32,
    memory: Vb2Memory,                 // Unknown / Mmap / Userptr / Dmabuf
    flags: Vb2QueueFlags {
        is_output, is_busy, streaming, error, waiting_for_buffers,
        last_buffer_dequeued, uses_qbuf, uses_requests, requires_requests,
        supports_requests, start_streaming_called, waiting_in_dqbuf,
        allow_cache_hints, non_coherent_mem, bidirectional, ...
    },
    buf_struct_size: usize,
    alloc_devs: [Option<NonNull<Device>>; VB2_MAX_PLANES],
    name: [u8; 32],
}

struct Vb2Buffer {
    vb2_queue: NonNull<Vb2Queue>,
    index: u32,
    typ: u32,
    memory: Vb2Memory,
    num_planes: u32,
    planes: [Vb2Plane; VB2_MAX_PLANES],
    state: Vb2BufferState,
    queued_entry: ListLink,
    done_entry: ListLink,
    timestamp: ktime_t,
    request_fd: i32,
    req_obj: MediaRequestObject,
    request: Option<NonNull<MediaRequest>>,
    prepared: bool,
    synced: bool,
    skip_cache_sync_on_prepare: bool,
    skip_cache_sync_on_finish: bool,
    copied_timestamp: bool,
}

struct Vb2Plane {
    bytesused: u32,
    length: u32,
    m: Vb2PlaneM { offset: u32, userptr: u64, fd: i32 },
    data_offset: u32,
    mem_priv: *mut u8,
    dbuf: Option<NonNull<DmaBuf>>,
    dbuf_mapped: bool,
    dbuf_duplicated: bool,
}
```

`Vb2::queue_init(q) -> i32`:
1. q.max_num_buffers ||= VB2_MAX_FRAME; q.max_num_buffers = min(q.max_num_buffers, MAX_BUFFER_INDEX).
2. /* Validation */
3. if !q | !q.ops | !q.mem_ops | !q.typ | !q.io_modes | !q.ops.queue_setup | !q.ops.buf_queue: return -EINVAL.
4. if q.max_num_buffers < VB2_MAX_FRAME | q.min_queued_buffers > q.max_num_buffers: return -EINVAL.
5. if q.requires_requests ∧ !q.supports_requests: return -EINVAL.
6. if q.supports_requests ∧ q.min_queued_buffers != 0: return -EINVAL.
7. q.min_reqbufs_allocation = max(q.min_reqbufs_allocation, q.min_queued_buffers + 1).
8. if q.min_reqbufs_allocation > q.max_num_buffers: return -EINVAL.
9. if !q.lock: return -EINVAL.
10. INIT_LIST_HEAD; spin_lock_init; mutex_init; init_waitqueue_head.
11. q.memory = Vb2Memory::Unknown.
12. q.buf_struct_size ||= size_of::<Vb2Buffer>().
13. q.dma_dir = if q.bidirectional { Bidirectional } else if q.is_output { ToDevice } else { FromDevice }.
14. if q.name == "": snprintf "{}-{:p}", (if is_output { "out" } else { "cap" }), q.
15. Ok(0).

`Vb2::reqbufs(q, memory, flags, *count) -> i32`:
1. if q.streaming: return -EBUSY.
2. if q.waiting_in_dqbuf ∧ *count > 0: return -EBUSY.
3. non_coherent = flags & V4L2_MEMORY_FLAG_NON_COHERENT.
4. if *count==0 ∨ vb2_get_num_buffers(q) != 0 ∨ (q.memory != Unknown ∧ q.memory != memory) ∨ !verify_coherency_flags(q, non_coherent):
   - mutex_lock(mmap_lock); Vb2::queue_cancel(q); Vb2::queue_free(q, 0, max_num_buffers); mutex_unlock.
   - q.is_busy = 0.
   - if *count == 0: return 0.
5. num_buffers = clamp(*count, q.min_reqbufs_allocation, q.max_num_buffers).
6. memset(alloc_devs, 0).
7. mutex_lock(mmap_lock); allocate bufs[] + bufs_bitmap; q.memory = memory; mutex_unlock.
8. set_queue_coherency(q, non_coherent).
9. call_qop(queue_setup, q, &num_buffers, &num_planes, plane_sizes, alloc_devs).
10. WARN_ON+EINVAL: !num_planes; ∀i in 0..num_planes: !plane_sizes[i].
11. allocated = Vb2::queue_alloc(q, memory, num_buffers, num_planes, plane_sizes, &first_index).
12. if allocated == 0: -ENOMEM.
13. if allocated < q.min_reqbufs_allocation: ret = -ENOMEM.
14. if !ret ∧ allocated < num_buffers: num_buffers = allocated; num_planes = 0; call_qop(queue_setup, ...).
15. mutex_lock(mmap_lock); if ret < 0: Vb2::queue_free(q, first_index, allocated); mutex_unlock; return ret.
16. mutex_unlock; *count = allocated; q.waiting_for_buffers = !q.is_output; q.is_busy = 1; return 0.

`Vb2::qbuf(q, vb, pb, req) -> i32`:
1. if q.error: return -EIO.
2. if !req ∧ vb.state != IN_REQUEST ∧ q.requires_requests: return -EBADR.
3. /* Reject mixed mode */
4. if (req ∧ q.uses_qbuf) ∨ (!req ∧ vb.state != IN_REQUEST ∧ q.uses_requests): return -EBUSY.
5. if let Some(req) = req {
   - q.uses_requests = 1.
   - if vb.state != DEQUEUED: return -EINVAL.
   - if q.is_output ∧ !vb.prepared: call_vb_qop(buf_out_validate, vb)?;
   - media_request_object_init(&vb.req_obj); media_request_lock_for_update(req)?;
   - media_request_object_bind(req, &VB2_CORE_REQ_OPS, q, true, &vb.req_obj)?;
   - media_request_unlock_for_update.
   - vb.state = IN_REQUEST; media_request_get(req); vb.request = Some(req).
   - if pb: copy_timestamp; fill_user_buffer.
   - return 0.
6. }
7. if vb.state != IN_REQUEST: q.uses_qbuf = 1.
8. match vb.state {
   - DEQUEUED | IN_REQUEST: if !vb.prepared: Vb2::buf_prepare(vb)?;
   - PREPARING: return -EINVAL.
   - _ => return -EINVAL. }
9. orig_state = vb.state; list_add_tail(&vb.queued_entry, &q.queued_list); q.queued_count += 1; q.waiting_for_buffers = false; vb.state = QUEUED.
10. if pb: copy_timestamp.
11. if q.start_streaming_called: Vb2::enqueue_in_driver(vb).
12. if pb: fill_user_buffer.
13. if q.streaming ∧ !q.start_streaming_called ∧ q.queued_count >= q.min_queued_buffers:
   - ret = Vb2::start_streaming(q).
   - if ret: list_del(&vb.queued_entry); q.queued_count -= 1; vb.state = orig_state; return ret.
14. Ok(0).

`Vb2::dqbuf(q, *pindex, pb, nonblocking) -> i32`:
1. Vb2::get_done_vb(q, &vb, pb, nonblocking)?.
2. match vb.state {
   - DONE | ERROR: ok.
   - _ => return -EINVAL. }
3. call_void_vb_qop(buf_finish, vb); vb.prepared = 0.
4. if pindex: *pindex = vb.index.
5. if pb: fill_user_buffer.
6. list_del(&vb.queued_entry); q.queued_count -= 1.
7. Vb2::dqbuf_inner(vb): vb.state = DEQUEUED; init_buffer.
8. WARN_ON(vb.req_obj.req); unbind + put req_obj; media_request_put(vb.request); vb.request = None.
9. Ok(0).

`Vb2::start_streaming(q) -> i32`:
1. for vb in q.queued_list: Vb2::enqueue_in_driver(vb).
2. q.start_streaming_called = 1.
3. ret = call_qop(start_streaming, q, q.queued_count).
4. if ret != 0:
   - q.start_streaming_called = 0.
   - for i in 0..q.max_num_buffers: vb = vb2_get_buffer(q, i); if vb ∧ vb.state == ACTIVE: Vb2::buffer_done(vb, QUEUED).
   - WARN_ON(atomic_read(&q.owned_by_drv_count)); WARN_ON(!q.done_list.empty()).
5. ret.

`Vb2::streamon(q, typ) -> i32`:
1. if typ != q.typ: -EINVAL.
2. if q.streaming: return 0.
3. q_num = vb2_get_num_buffers(q); if q_num == 0: -EINVAL.
4. if q_num < q.min_queued_buffers: -EINVAL.
5. call_qop(prepare_streaming, q)?;
6. if q.queued_count >= q.min_queued_buffers: ret = Vb2::start_streaming(q); if ret: call_void_qop(unprepare_streaming, q); return ret.
7. q.streaming = 1; Ok(0).

`Vb2::streamoff(q, typ) -> i32`:
1. if typ != q.typ: -EINVAL.
2. Vb2::queue_cancel(q).
3. q.waiting_for_buffers = !q.is_output; q.last_buffer_dequeued = false; Ok(0).

`Vb2::queue_cancel(q)`:
1. if q.start_streaming_called: call_void_qop(stop_streaming, q).
2. if q.streaming: call_void_qop(unprepare_streaming, q).
3. WARN_ON(atomic_read(&q.owned_by_drv_count)).
4. For any vb still ACTIVE: pr_warn(); Vb2::buffer_done(vb, ERROR).
5. q.streaming = 0; q.start_streaming_called = 0; q.queued_count = 0; q.error = 0; q.uses_requests = 0; q.uses_qbuf = 0.
6. INIT_LIST_HEAD(&q.queued_list); INIT_LIST_HEAD(&q.done_list).
7. atomic_set(&q.owned_by_drv_count, 0); wake_up_all(&q.done_wq).
8. For each vb in q.bufs: __vb2_buf_mem_finish; if vb.prepared: buf_finish; Vb2::dqbuf_inner; clear req_obj/request; copied_timestamp=0.

`Vb2::buffer_done(vb, state)`:
1. q = vb.vb2_queue.
2. WARN_ON if vb.state != ACTIVE: return.
3. if state ∉ {DONE, ERROR, QUEUED}: WARN; state = ERROR.
4. spin_lock_irqsave(&q.done_lock).
5. if state != QUEUED:
   - __vb2_buf_mem_finish(vb).
   - list_add_tail(&vb.done_entry, &q.done_list).
   - vb.state = state.
6. else: list_add(&vb.queued_entry, head of queued_list); vb.state = QUEUED.
7. atomic_dec(&q.owned_by_drv_count).
8. spin_unlock_irqrestore.
9. if state ∈ {DONE, ERROR}: wake_up(&q.done_wq).

`Vb2::mmap(q, vma) -> i32`:
1. if !(vma.vm_flags & VM_SHARED): -EINVAL.
2. if q.is_output ∧ !(vma.vm_flags & VM_WRITE): -EINVAL.
3. if !q.is_output ∧ !(vma.vm_flags & VM_READ): -EINVAL.
4. mutex_lock(&q.mmap_lock).
5. offset = vma.vm_pgoff << PAGE_SHIFT.
6. find_plane_by_offset(q, offset, &vb, &plane)?;
7. length = PAGE_ALIGN(vb.planes[plane].length).
8. if length < (vma.vm_end - vma.vm_start): -EINVAL.
9. vma.vm_pgoff = 0.
10. ret = call_memop(mmap, vb, vb.planes[plane].mem_priv, vma).
11. mutex_unlock; ret.

`Vb2::expbuf(q, *fd, typ, vb, plane, flags) -> i32`:
1. if q.memory != Mmap: -EINVAL.
2. if !q.mem_ops.get_dmabuf: -EINVAL.
3. if flags & !(O_CLOEXEC | O_ACCMODE): -EINVAL.
4. if typ != q.typ: -EINVAL.
5. if plane >= vb.num_planes: -EINVAL.
6. if vb2_fileio_is_active(q): -EBUSY.
7. dbuf = call_ptr_memop(get_dmabuf, vb, vb.planes[plane].mem_priv, flags & O_ACCMODE).
8. if IS_ERR_OR_NULL(dbuf): -EINVAL.
9. *fd = dma_buf_fd(dbuf, flags & !O_ACCMODE); on err: dma_buf_put.

`Vb2::poll(q, file, wait) -> u32`:
1. poll_wait(file, &q.done_wq, wait).
2. req_events = poll_requested_events(wait).
3. if !q.is_output ∧ !(req_events & (POLLIN | POLLRDNORM)): return 0.
4. if q.is_output ∧ !(req_events & (POLLOUT | POLLWRNORM)): return 0.
5. if vb2_get_num_buffers(q) == 0 ∧ !vb2_fileio_is_active(q):
   - if !q.is_output ∧ (q.io_modes & VB2_READ) ∧ (req_events & (POLLIN | POLLRDNORM)): __vb2_init_fileio(q, 1)?;
   - if q.is_output ∧ (q.io_modes & VB2_WRITE) ∧ (req_events & (POLLOUT | POLLWRNORM)): __vb2_init_fileio(q, 0)?; return POLLOUT | POLLWRNORM.
6. if !q.streaming ∨ q.error: return POLLERR.
7. spin_lock_irqsave(&q.done_lock).
8. if !q.done_list.empty():
   - vb = head; if vb.state ∈ {DONE, ERROR}: ret = (if q.is_output { POLLOUT|POLLWRNORM } else { POLLIN|POLLRDNORM }).
9. spin_unlock_irqrestore.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `state_machine_well_formed` | INVARIANT | per-buffer-state: only legal transitions DEQUEUED→PREPARING→DEQUEUED→QUEUED→ACTIVE→{DONE,ERROR,QUEUED}→DEQUEUED. |
| `owned_by_drv_balanced` | INVARIANT | per-enqueue_in_driver vs buffer_done: atomic_inc / atomic_dec paired; reaches 0 after stop. |
| `queue_cancel_resets_counters` | INVARIANT | per-queue_cancel: queued_count=0 ∧ owned_by_drv_count=0 ∧ queued_list/done_list empty. |
| `done_lock_protects_done_list` | INVARIANT | per-buffer_done / get_done_vb: done_list mutated only under done_lock. |
| `mmap_requires_vm_shared` | INVARIANT | per-vb2_mmap: VM_SHARED + direction-matched read/write enforced. |
| `expbuf_requires_mmap` | INVARIANT | per-vb2_core_expbuf: q.memory == MMAP ∧ mem_ops.get_dmabuf != NULL. |
| `qbuf_in_request_requires_dequeued` | INVARIANT | per-qbuf(req): vb.state == DEQUEUED at entry. |
| `start_streaming_rollback_to_queued` | INVARIANT | per-vb2_start_streaming failure: every ACTIVE buffer recycled via buffer_done(QUEUED). |
| `streamon_requires_min_queued` | INVARIANT | per-streamon: rejects when q_num_bufs < min_queued_buffers. |
| `dqbuf_clears_prepared` | INVARIANT | per-dqbuf: vb.prepared cleared after buf_finish. |
| `plane_offset_cookie_invertible` | INVARIANT | per-__setup_offsets / __find_plane_by_offset: round-trips for buffer<MAX_BUFFER_INDEX ∧ plane<num_planes. |

### Layer 2: TLA+

`drivers/media/videobuf2-core.tla`:
- States per-vb ∈ {DEQUEUED, IN_REQUEST, PREPARING, QUEUED, ACTIVE, DONE, ERROR}.
- Per-queue: {ALLOCATED, STREAMING_PREPARED, STREAMING, ERROR, CANCELED}.
- Actions: reqbufs, create_bufs, prepare_buf, qbuf, qbuf_request, streamon, start_streaming, buf_queue, buffer_done, dqbuf, streamoff, queue_cancel, queue_release.
- Properties:
  - `safety_state_machine` — per-vb: state transitions only on allowed action.
  - `safety_owned_count_consistency` — per-owned_by_drv_count: equals |{vb: state==ACTIVE}|.
  - `safety_done_list_only_done_error` — per-done_list: all members have state DONE or ERROR.
  - `safety_streamoff_drains` — per-streamoff: all vb end in DEQUEUED.
  - `safety_start_streaming_rollback` — per-start_streaming failure: state ∈ {QUEUED, DEQUEUED}.
  - `liveness_qbuf_eventually_dequeues` — per-streaming-OK: every qbuf'd buffer reaches DONE/ERROR and is dequeueable.
  - `safety_requests_excl_qbuf` — per-queue: uses_requests ∧ uses_qbuf cannot both be true.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vb2::queue_init` post: Ok ⟹ all lists/locks initialized ∧ q.memory==Unknown ∧ q.dma_dir set | `Vb2::queue_init` |
| `Vb2::reqbufs` post: Ok ⟹ q.is_busy ∧ allocated == *count ∈ [min_reqbufs, max] | `Vb2::reqbufs` |
| `Vb2::qbuf` post: Ok ⟹ vb.state ∈ {QUEUED, ACTIVE, IN_REQUEST} ∧ list-membership consistent | `Vb2::qbuf` |
| `Vb2::dqbuf` post: Ok ⟹ vb.state == DEQUEUED ∧ vb.prepared == false ∧ vb.request == None | `Vb2::dqbuf` |
| `Vb2::streamon` post: Ok ⟹ q.streaming == true ∧ prepare_streaming called | `Vb2::streamon` |
| `Vb2::streamoff` post: Ok ⟹ q.streaming == false ∧ owned_by_drv_count == 0 ∧ both lists empty | `Vb2::streamoff` |
| `Vb2::buffer_done` post: state ∈ {DONE, ERROR} ⟹ vb on done_list ∧ done_wq waked; QUEUED ⟹ vb on queued_list head | `Vb2::buffer_done` |
| `Vb2::start_streaming` post: Err ⟹ every prior ACTIVE vb on queued_list with state QUEUED | `Vb2::start_streaming` |
| `Vb2::mmap` post: Ok ⟹ vma.vm_pgoff == 0 ∧ mem_ops.mmap returned 0 | `Vb2::mmap` |
| `Vb2::expbuf` post: Ok ⟹ *fd ≥ 0 ∧ dma_buf_get refcount taken | `Vb2::expbuf` |

### Layer 4: Verus/Creusot functional

`Per-capture lifecycle: queue_init → reqbufs(MMAP, N) → mmap(plane0..M) → streamon → loop(qbuf → poll → dqbuf) → streamoff → queue_release` semantic equivalence: per-Documentation/userspace-api/media/v4l/buffer.rst and Documentation/driver-api/media/v4l2-videobuf2.rst.

`Per-DMABUF import lifecycle: queue_init → reqbufs(DMABUF, N) → for each buffer fill m.fd → qbuf → streamon → dqbuf → release fd` semantic equivalence: per-dma-buf API contract.

`Per-USERPTR lifecycle: queue_init → reqbufs(USERPTR, N) → for each buffer fill m.userptr+length → qbuf → streamon → dqbuf` semantic equivalence: per-userspace pin/unpin contract.

## Hardening

(Inherits row-1 features from `drivers/media/00-overview.md` § Hardening.)

VB2 reinforcement:

- **Per-state machine enforced at every transition** — defense against per-driver-misuse (out-of-order buf_queue / buffer_done).
- **Per-owned_by_drv_count atomic_t** — defense against per-driver-leak of ACTIVE buffers across stop_streaming.
- **Per-WARN_ON in vb2_buffer_done(state != ACTIVE)** — defense against per-double-completion.
- **Per-state coercion to ERROR for invalid completion state** — defense against per-driver-passing-bogus-state.
- **Per-q.error sticky flag** — defense against per-transient-recovery hiding fatal HW state.
- **Per-mmap_lock + done_lock + caller-side q.lock layering** — defense against per-mmap-vs-streamon race.
- **Per-mode lockout uses_qbuf ⊕ uses_requests** — defense against per-mixed-mode use-after-bind.
- **Per-vb2_start_streaming rollback to QUEUED** — defense against per-driver-fail-leak.
- **Per-min_queued_buffers gate before start_streaming** — defense against per-driver-underflow at HW level.
- **Per-fileio mutually exclusive with streaming ioctls** — defense against per-read/write+QBUF concurrent use.
- **Per-plane offset cookie validated against MAX_BUFFER_INDEX** — defense against per-userspace-mmap escape.
- **Per-VM_SHARED + direction-matched VM_READ/WRITE required** — defense against per-COW alias of DMA pages.
- **Per-request-API media_request_object refcount paired** — defense against per-request-UAF.
- **Per-DMABUF dbuf_duplicated guard** — defense against per-double-attach of the same dma-buf across planes.
- **Per-min_reqbufs_allocation ≥ min_queued_buffers + 1** — defense against per-streamon-immediately-after-reqbufs starvation.
- **Per-coherency-flag verification on second REQBUFS** — defense against per-coherency-model-mismatch corruption.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/media/common/videobuf2/videobuf2-v4l2.c V4L2-specific wrappers (covered in `videobuf2-v4l2.md` Tier-3 if expanded)
- drivers/media/common/videobuf2/videobuf2-dma-contig.c (covered in `videobuf2-dma-contig.md` Tier-3 if expanded)
- drivers/media/common/videobuf2/videobuf2-dma-sg.c (covered in `videobuf2-dma-sg.md` if expanded)
- drivers/media/common/videobuf2/videobuf2-vmalloc.c (covered in `videobuf2-vmalloc.md` if expanded)
- drivers/media/common/videobuf2/videobuf2-memops.c (vm_ops shared helpers)
- drivers/dma-buf/dma-buf.c dma-buf core (covered in `dma-buf.md` Tier-3)
- drivers/media/mc/media-request.c media-request core (covered in `media-controller.md` Tier-3)
- drivers/media/v4l2-core/v4l2-mem2mem.c m2m glue (covered separately)
- Per-driver vb2_queue setup (uvcvideo, vivid, ...)
- Implementation code
