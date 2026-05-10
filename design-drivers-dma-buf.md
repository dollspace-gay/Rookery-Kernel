---
title: "Tier-3: drivers/dma-buf/dma-buf.c — DMA-BUF buffer sharing"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **DMA-BUF subsystem** is the kernel's GPL-licensed cross-driver buffer-sharing framework: any kernel subsystem can wrap a backing storage (CPU RAM, GPU VRAM, scatter-gather list, PCI BAR, ...) in a `struct dma_buf`, hand out an fd, and another subsystem (DRM, V4L2, network, RDMA, ...) imports that fd, calls `dma_buf_attach()` to register a device, then `dma_buf_map_attachment()` to obtain an `sg_table` mapped into that device's IOMMU/DMA address space. The exporter publishes a `struct dma_buf_ops` vtable (`map_dma_buf`, `unmap_dma_buf`, `mmap`, `vmap`, `pin`/`unpin`, `begin_cpu_access`/`end_cpu_access`, `release`); the importer holds attachments. A pseudo-filesystem (`dmabuf` magic) backs anonymous inodes via `kern_mount`, and per-buffer `struct dma_resv` carries implicit fences (read/write set) so cross-device synchronization works through `poll()`, the `DMA_BUF_IOCTL_{EXPORT,IMPORT}_SYNC_FILE` ioctls, and `dma_buf_begin_cpu_access` waits. Critical for: zero-copy GPU/display/camera/video pipelines, Vulkan/EGL FD export, container-shared buffers, Wayland compositor handoff.

This Tier-3 covers `drivers/dma-buf/dma-buf.c` (~1848 lines).

### Acceptance Criteria

- [ ] AC-1: dma_buf_export with valid exp_info returns dma_buf; file refcount 1; appears in `dmabuf_list`.
- [ ] AC-2: dma_buf_export with missing required op (map_dma_buf / unmap_dma_buf / release) ⟹ ERR_PTR(-EINVAL).
- [ ] AC-3: dma_buf_fd → user fd; dma_buf_get(fd) returns same dmabuf; non-dma-buf fd ⟹ ERR_PTR(-EINVAL).
- [ ] AC-4: dma_buf_put when last reference drops ⟹ ops->release invoked, attachments-list empty (WARN-on-leak).
- [ ] AC-5: dma_buf_attach → list_add under resv lock; dma_buf_detach → list_del + ops->detach + kfree.
- [ ] AC-6: dma_buf_map_attachment static-attachment waits for DMA_RESV_USAGE_KERNEL fences before returning sg_table.
- [ ] AC-7: dma_buf_map_attachment dynamic-attachment does NOT wait for fences (importer's responsibility).
- [ ] AC-8: dma_buf_unmap_attachment unpins when dma_buf_pin_on_map was true at map time.
- [ ] AC-9: dma_buf_pin without importer_ops ⟹ WARN.
- [ ] AC-10: dma_buf_mmap with pgoff overflowing size ⟹ -EINVAL; pgoff causing u32 overflow ⟹ -EOVERFLOW.
- [ ] AC-11: dma_buf_vmap caches mapping; second call increments vmapping_counter; vunmap decrements; final vunmap calls ops->vunmap.
- [ ] AC-12: dma_buf_poll EPOLLIN/EPOLLOUT returns 0 (pending) while fences unsignaled; fires wake-up via callback chain on signal.
- [ ] AC-13: DMA_BUF_IOCTL_SYNC SYNC_START + direction-bits ⟹ dma_buf_begin_cpu_access; SYNC_END ⟹ dma_buf_end_cpu_access; waits on read/write implicit fences.
- [ ] AC-14: DMA_BUF_IOCTL_EXPORT_SYNC_FILE returns sync_file fd containing dma_resv-singleton fence for direction.
- [ ] AC-15: DMA_BUF_IOCTL_IMPORT_SYNC_FILE adds (unwrapped) fences from sync_file fd to dma_resv under reserve.
- [ ] AC-16: dma_buf_invalidate_mappings calls each attached importer_ops->invalidate_mappings under resv lock.

### Architecture

```
struct DmaBuf {
  size: usize,
  ops: &'static DmaBufOps,
  file: ArcFile,
  attachments: ListHead<DmaBufAttachment>,
  resv: NonNull<DmaResv>,               // embedded or external
  resv_embedded: bool,
  priv: AnyArc,                          // exporter opaque
  exp_name: &'static str,
  owner: ModuleRef,
  name: SpinLock<Option<KString>>,       // user-visible
  poll_wq: WaitQueueHead,
  cb_in: DmaBufPollCb,                   // EPOLLIN dcb
  cb_out: DmaBufPollCb,                  // EPOLLOUT dcb
  vmap_ptr: SpinCell<IoSysMap>,
  vmapping_counter: AtomicU32,
  list_node: ListNode,
}

struct DmaBufAttachment {
  dmabuf: NonNull<DmaBuf>,
  dev: NonNull<Device>,
  node: ListNode,
  priv: AnyArc,
  importer_ops: Option<&'static DmaBufAttachOps>,
  importer_priv: AnyArc,
  peer2peer: bool,
}

struct DmaBufOps {
  attach: Option<fn(&DmaBuf, &mut DmaBufAttachment) -> Errno<()>>,
  detach: Option<fn(&DmaBuf, &mut DmaBufAttachment)>,
  pin:    Option<fn(&DmaBufAttachment) -> Errno<()>>,
  unpin:  Option<fn(&DmaBufAttachment)>,
  map_dma_buf:   fn(&DmaBufAttachment, DmaDataDirection) -> ErrPtr<SgTable>,
  unmap_dma_buf: fn(&DmaBufAttachment, &SgTable, DmaDataDirection),
  release:       fn(&DmaBuf),
  begin_cpu_access: Option<fn(&DmaBuf, DmaDataDirection) -> Errno<()>>,
  end_cpu_access:   Option<fn(&DmaBuf, DmaDataDirection) -> Errno<()>>,
  mmap:   Option<fn(&DmaBuf, &mut VmAreaStruct) -> Errno<()>>,
  vmap:   Option<fn(&DmaBuf, &mut IoSysMap) -> Errno<()>>,
  vunmap: Option<fn(&DmaBuf, &IoSysMap)>,
}
```

`DmaBuf::export(exp_info) -> ErrPtr<&'static DmaBuf>`:
1. WARN if (!priv ∨ !ops ∨ !ops.map_dma_buf ∨ !ops.unmap_dma_buf ∨ !ops.release) ⟹ Err(EINVAL).
2. WARN if (!ops.pin) != (!ops.unpin) ⟹ Err(EINVAL).
3. owner_ref = try_module_get(exp_info.owner). None ⟹ Err(ENOENT).
4. file = DmaBuf::getfile(exp_info.size, exp_info.flags) — anon-inode under dma_buf_mnt. Err ⟹ module_put.
5. alloc_size = sizeof(DmaBuf) + (exp_info.resv.is_some() ? 1 : sizeof(DmaResv)).
6. dmabuf = kzalloc(alloc_size, GFP_KERNEL). None ⟹ fput; module_put; Err(ENOMEM).
7. Init fields (priv, ops, size, exp_name, owner, name_lock, poll_wq, cb_in.poll=&poll_wq, cb_out.poll=&poll_wq, attachments=LIST_HEAD).
8. If exp_info.resv: dmabuf.resv = exp_info.resv. Else: dmabuf.resv = &dmabuf[1]; dma_resv_init.
9. file.private_data = dmabuf; file.dentry.d_fsdata = dmabuf; dmabuf.file = file.
10. __dma_buf_list_add(dmabuf) under dmabuf_list_mutex.
11. trace_dma_buf_export; Ok(dmabuf).

`DmaBuf::install_fd(dmabuf, flags) -> Errno<i32>`:
1. if (!dmabuf ∨ !dmabuf.file): Err(EINVAL).
2. fd = FD_ADD(flags, dmabuf.file).
3. trace_dma_buf_fd; Ok(fd).

`DmaBuf::get(fd) -> ErrPtr<&'static DmaBuf>`:
1. file = fget(fd). None ⟹ Err(EBADF).
2. if !is_dma_buf_file(file): fput; Err(EINVAL).
3. Ok(file.private_data as &DmaBuf).

`DmaBuf::put(dmabuf)`:
1. WARN if !dmabuf ∨ !dmabuf.file.
2. fput(dmabuf.file).

`DmaBuf::release(dentry)`  (dentry_operations.d_release):
1. dmabuf = dentry.d_fsdata; if None return.
2. BUG_ON(dmabuf.vmapping_counter != 0).
3. BUG_ON(dmabuf.cb_in.active ∨ dmabuf.cb_out.active).
4. dmabuf.ops.release(dmabuf).
5. if dmabuf.resv_embedded: dma_resv_fini.
6. WARN_ON(!dmabuf.attachments.is_empty()).
7. module_put(dmabuf.owner); kfree(dmabuf.name); kfree(dmabuf).

`DmaBuf::dynamic_attach(dmabuf, dev, importer_ops, importer_priv) -> ErrPtr<&DmaBufAttachment>`:
1. WARN if !dmabuf ∨ !dev ⟹ Err(EINVAL).
2. attach = kzalloc(sizeof(DmaBufAttachment), GFP_KERNEL). None ⟹ Err(ENOMEM).
3. attach.dev = dev; attach.dmabuf = dmabuf.
4. if importer_ops: attach.peer2peer = importer_ops.allow_peer2peer.
5. attach.importer_ops = importer_ops; attach.importer_priv = importer_priv.
6. if dmabuf.ops.attach: dmabuf.ops.attach(dmabuf, attach). Err ⟹ kfree(attach).
7. dma_resv_lock(dmabuf.resv, NULL); list_add(&attach.node, &dmabuf.attachments); dma_resv_unlock.
8. Ok(attach).

`DmaBufAttachment::map(attach, direction) -> ErrPtr<&SgTable>`:
1. might_sleep.
2. WARN if !attach ∨ !attach.dmabuf ⟹ Err(EINVAL).
3. dma_resv_assert_held(attach.dmabuf.resv).
4. if dma_buf_pin_on_map(attach): ret = ops.pin(attach); WARN_ONCE if EBUSY; Err on non-zero.
5. sg_table = ops.map_dma_buf(attach, direction). None ⟹ Err(ENOMEM). Err ⟹ jump error_unpin.
6. if !attach.is_dynamic(): ret = dma_resv_wait_timeout(resv, DMA_RESV_USAGE_KERNEL, true, MAX_SCHEDULE_TIMEOUT). ret < 0 ⟹ error_unmap.
7. dma_buf_wrap_sg_table(&sg_table) (CONFIG_DMABUF_DEBUG only).
8. CONFIG_DMA_API_DEBUG: verify PAGE_SIZE alignment for each (dma_address, dma_len).
9. Ok(sg_table).

`DmaBuf::poll(file, poll_table) -> PollMask`:
1. dmabuf = file.private_data; if !dmabuf ∨ !resv: EPOLLERR.
2. poll_wait(file, &dmabuf.poll_wq, poll_table).
3. events = poll_requested_events & (EPOLLIN | EPOLLOUT). If empty: 0.
4. dma_resv_lock(resv, NULL).
5. For dir in (OUT, IN):
   - dcb = if OUT { &cb_out } else { &cb_in }; write = (dir == OUT).
   - spin_lock_irq(&poll_wq.lock): if dcb.active != 0 { events &= ~dir } else { dcb.active = dir }. spin_unlock_irq.
   - if events & dir:
     - get_file(dmabuf.file) (paired in poll_cb).
     - if !poll_add_cb(resv, write, dcb): poll_cb(NULL, &dcb.cb) (fast wake).
     - else: events &= ~dir (still pending).
6. dma_resv_unlock; return events.

`DmaBuf::poll_cb(fence, cb)` (fence-signal callback):
1. dcb = container_of(cb, DmaBufPollCb, cb); dmabuf = container_of(dcb.poll, DmaBuf, poll_wq).
2. spin_lock_irqsave(&dcb.poll.lock): wake_up_locked_poll(dcb.poll, dcb.active); dcb.active = 0; spin_unlock_irqrestore.
3. dma_fence_put(fence); fput(dmabuf.file).

`DmaBuf::begin_cpu_access(dmabuf, direction) -> Errno<()>`:
1. WARN if !dmabuf ⟹ Err(EINVAL).
2. might_lock(&resv.lock.base).
3. ret = ops.begin_cpu_access.map(|f| f(dmabuf, direction)).unwrap_or(Ok(()));
4. if ret == Ok: dma_resv_wait_timeout(resv, dma_resv_usage_rw(write), true, MAX_SCHEDULE_TIMEOUT).
5. Return ret.

`DmaBuf::ioctl(cmd, arg)`:
- DMA_BUF_IOCTL_SYNC: validate flags & direction → begin/end_cpu_access.
- DMA_BUF_SET_NAME_A/_B: set_name(strndup_user(buf, DMA_BUF_NAME_LEN)) under name_lock.
- DMA_BUF_IOCTL_EXPORT_SYNC_FILE (CONFIG_SYNC_FILE): dma_resv_get_singleton + sync_file_create + fd_install.
- DMA_BUF_IOCTL_IMPORT_SYNC_FILE (CONFIG_SYNC_FILE): sync_file_get_fence → dma_fence_unwrap_for_each → dma_resv_reserve_fences + dma_resv_add_fence.
- default: -ENOTTY.

### Out of Scope

- `drivers/dma-buf/dma-resv.c` reservation-object internals (Tier-3 dma-resv.md)
- `drivers/dma-buf/dma-fence.c` fence primitive (Tier-3 dma-fence.md)
- `drivers/dma-buf/dma-fence-chain.c` / `dma-fence-array.c` / `dma-fence-unwrap.c` composite fence types (separate Tier-3)
- `drivers/dma-buf/sync_file.c` userspace sync_file fd (separate Tier-3)
- `drivers/dma-buf/sw_sync.c` software fence timeline (separate Tier-3)
- `drivers/dma-buf/dma-heap.c` and `heaps/` allocator family (separate Tier-3)
- `drivers/dma-buf/udmabuf.c` userspace memfd→dma-buf bridge (separate Tier-3)
- `drivers/dma-buf/dma-buf-mapping.c` extra mapping helpers (separate Tier-3)
- Exporter-specific implementations (DRM GEM, V4L2 vb2, ION-replacement heaps, ...)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dma_buf` | per-buffer object | `DmaBuf` |
| `struct dma_buf_attachment` | per-device attachment | `DmaBufAttachment` |
| `struct dma_buf_ops` | per-exporter vtable | `DmaBufOps` |
| `struct dma_buf_export_info` | per-export descriptor | `DmaBufExportInfo` |
| `struct dma_buf_attach_ops` | per-importer vtable | `DmaBufAttachOps` |
| `dma_buf_export()` | per-create | `DmaBuf::export` |
| `dma_buf_fd()` | per-fd-install | `DmaBuf::install_fd` |
| `dma_buf_get()` | per-fd-lookup | `DmaBuf::get` |
| `dma_buf_put()` | per-refcount-drop | `DmaBuf::put` |
| `dma_buf_attach()` | per-static attach | `DmaBuf::attach` |
| `dma_buf_dynamic_attach()` | per-dynamic attach | `DmaBuf::dynamic_attach` |
| `dma_buf_detach()` | per-detach | `DmaBuf::detach` |
| `dma_buf_pin()` / `_unpin()` | per-pin | `DmaBufAttachment::pin` / `unpin` |
| `dma_buf_map_attachment()` | per-acquire sg_table | `DmaBufAttachment::map` |
| `dma_buf_map_attachment_unlocked()` | per-unlocked variant | `DmaBufAttachment::map_unlocked` |
| `dma_buf_unmap_attachment()` | per-release sg_table | `DmaBufAttachment::unmap` |
| `dma_buf_unmap_attachment_unlocked()` | per-unlocked variant | `DmaBufAttachment::unmap_unlocked` |
| `dma_buf_attach_revocable()` | per-revoke-capable query | `DmaBufAttachment::is_revocable` |
| `dma_buf_invalidate_mappings()` | per-move notification | `DmaBuf::invalidate_mappings` |
| `dma_buf_begin_cpu_access()` | per-CPU-coherency-enter | `DmaBuf::begin_cpu_access` |
| `dma_buf_end_cpu_access()` | per-CPU-coherency-exit | `DmaBuf::end_cpu_access` |
| `dma_buf_mmap()` | per-userspace-mmap | `DmaBuf::mmap` |
| `dma_buf_vmap()` / `_unlocked()` | per-kernel-vmap | `DmaBuf::vmap` / `vmap_unlocked` |
| `dma_buf_vunmap()` / `_unlocked()` | per-kernel-vunmap | `DmaBuf::vunmap` / `vunmap_unlocked` |
| `dma_buf_iter_begin()` / `_next()` | per-debugfs-iter | `DmaBuf::iter_begin` / `iter_next` |
| `dma_buf_poll` / `dma_buf_poll_cb` | per-fence-poll | `DmaBuf::poll` / `poll_cb` |
| `dma_buf_set_name()` | per-name-ioctl | `DmaBuf::set_name` |
| `dma_buf_export_sync_file()` | per-EXPORT_SYNC_FILE ioctl | `DmaBuf::export_sync_file` |
| `dma_buf_import_sync_file()` | per-IMPORT_SYNC_FILE ioctl | `DmaBuf::import_sync_file` |
| `dma_buf_ioctl()` | per-ioctl dispatch | `DmaBuf::ioctl` |
| `dma_buf_release()` (dentry op) | per-final-release | `DmaBuf::release` |
| `dma_buf_fops` | per-fd file_operations | shared |
| `dma_buf_fs_type` (kern_mount) | per-pseudo-fs | shared |
| `dmabuf_list` / `dmabuf_list_mutex` | per-global-debugfs list | `DmaBuf::LIST` |

### compatibility contract

REQ-1: struct dma_buf:
- size: per-buffer byte length (PAGE_SIZE-aligned for sg-based exporters).
- ops: const `dma_buf_ops *` (vtable: `attach`, `detach`, `pin`, `unpin`, `map_dma_buf`, `unmap_dma_buf`, `release`, `begin_cpu_access`, `end_cpu_access`, `mmap`, `vmap`, `vunmap`).
- file: backing `struct file` (anonymous inode under `dmabuf` pseudo-fs).
- attachments: per-buffer list of `dma_buf_attachment` (head + node).
- resv: per-buffer `struct dma_resv` (implicit-fence reservation set); embedded when `exp_info->resv` is NULL.
- priv: per-exporter opaque.
- exp_name / owner: per-exporter identification / module-owner.
- name / name_lock: per-buffer user-visible name + spinlock.
- poll: per-buffer waitqueue (poll fence-signal).
- cb_in / cb_out: per-direction `dma_buf_poll_cb_t` for EPOLLIN/EPOLLOUT fence callbacks.
- vmap_ptr / vmapping_counter: per-buffer cached kernel vmap + refcount.
- list_node: per-global-debugfs `dmabuf_list`.

REQ-2: struct dma_buf_attachment:
- dmabuf: per-attachment back-pointer.
- dev: per-importer device.
- node: list_head linking into dmabuf->attachments.
- priv: per-importer opaque.
- importer_ops: optional `dma_buf_attach_ops *` (NULL ⟹ static attachment).
- importer_priv: per-importer opaque (for importer_ops callbacks).
- peer2peer: per-attachment peer-to-peer permission (copied from `importer_ops->allow_peer2peer`).

REQ-3: dma_buf_export(exp_info):
- WARN if `!priv ∨ !ops ∨ !ops->map_dma_buf ∨ !ops->unmap_dma_buf ∨ !ops->release` ⟹ ERR_PTR(-EINVAL).
- WARN if `!ops->pin != !ops->unpin` (paired) ⟹ ERR_PTR(-EINVAL).
- try_module_get(exp_info->owner) — fail ⟹ ERR_PTR(-ENOENT).
- file = dma_buf_getfile(size, flags) — anon-inode under `dma_buf_mnt`.
- alloc_size = sizeof(struct dma_buf) + (resv ? 1 : sizeof(struct dma_resv)).
- dmabuf = kzalloc(alloc_size, GFP_KERNEL).
- Init: priv, ops, size, exp_name, owner, spin_lock_init(&name_lock), init_waitqueue_head(&poll), cb_in/out.poll = &poll, INIT_LIST_HEAD(&attachments).
- resv: external (passed in) OR embedded at `(struct dma_resv *)&dmabuf[1]` then `dma_resv_init`.
- file->private_data = dmabuf; file->f_path.dentry->d_fsdata = dmabuf; dmabuf->file = file.
- __dma_buf_list_add(dmabuf) under dmabuf_list_mutex.
- trace_dma_buf_export.
- Return dmabuf (errors: ERR_PTR with module_put + fput rollback).

REQ-4: dma_buf_fd(dmabuf, flags):
- if `!dmabuf ∨ !dmabuf->file` ⟹ -EINVAL.
- fd = FD_ADD(flags, dmabuf->file) — allocates fd and installs file.
- trace_dma_buf_fd.
- Return fd (negative on error).

REQ-5: dma_buf_get(fd):
- file = fget(fd) — NULL ⟹ ERR_PTR(-EBADF).
- if `!is_dma_buf_file(file)` (f_op != &dma_buf_fops): fput; ERR_PTR(-EINVAL).
- dmabuf = file->private_data.
- trace_dma_buf_get.
- Return dmabuf (file refcount transferred to caller).

REQ-6: dma_buf_put(dmabuf):
- WARN if `!dmabuf ∨ !dmabuf->file`.
- fput(dmabuf->file) — when last ref drops, dentry release calls `dma_buf_release`.

REQ-7: dma_buf_release (dentry op, called from final fput → kill_anon_super path):
- dmabuf = dentry->d_fsdata; if NULL return.
- BUG_ON(dmabuf->vmapping_counter) — caller must vunmap before release.
- BUG_ON(dmabuf->cb_in.active ∨ dmabuf->cb_out.active) — pending poll callbacks must be drained.
- dmabuf->ops->release(dmabuf) — exporter-specific destruction.
- if resv embedded: dma_resv_fini.
- WARN_ON(!list_empty(&dmabuf->attachments)).
- module_put(dmabuf->owner); kfree(dmabuf->name); kfree(dmabuf).

REQ-8: dma_buf_attach(dmabuf, dev) ⟹ dma_buf_dynamic_attach(dmabuf, dev, NULL, NULL).

REQ-9: dma_buf_dynamic_attach(dmabuf, dev, importer_ops, importer_priv):
- WARN if `!dmabuf ∨ !dev` ⟹ ERR_PTR(-EINVAL).
- attach = kzalloc(sizeof(*attach), GFP_KERNEL).
- attach->dev = dev; attach->dmabuf = dmabuf.
- if importer_ops: attach->peer2peer = importer_ops->allow_peer2peer.
- attach->importer_ops = importer_ops; attach->importer_priv = importer_priv.
- if dmabuf->ops->attach: call exporter callback (rollback kfree on error).
- dma_resv_lock(dmabuf->resv, NULL); list_add(&attach->node, &dmabuf->attachments); dma_resv_unlock.
- Return attach.

REQ-10: dma_buf_detach(dmabuf, attach):
- WARN if `!dmabuf ∨ !attach ∨ dmabuf != attach->dmabuf`.
- dma_resv_lock → list_del(&attach->node) → dma_resv_unlock.
- if dmabuf->ops->detach: call exporter callback.
- kfree(attach).

REQ-11: dma_buf_pin(attach) (importer holds dma_resv lock):
- WARN if !attach->importer_ops (only dynamic attachments may pin).
- dma_resv_assert_held(dmabuf->resv).
- if dmabuf->ops->pin: ret = ops->pin(attach); else 0.
- Return ret.

REQ-12: dma_buf_unpin(attach):
- WARN if !attach->importer_ops.
- dma_resv_assert_held; if ops->unpin: ops->unpin(attach).

REQ-13: dma_buf_map_attachment(attach, direction):
- might_sleep().
- WARN if `!attach ∨ !attach->dmabuf` ⟹ ERR_PTR(-EINVAL).
- dma_resv_assert_held(attach->dmabuf->resv).
- if `dma_buf_pin_on_map(attach)` (exporter has pin/unpin AND attachment is static): ret = ops->pin(attach). WARN_ONCE(-EBUSY); err return on failure.
- sg_table = ops->map_dma_buf(attach, direction). NULL ⟹ ERR_PTR(-ENOMEM). IS_ERR ⟹ error_unpin path.
- if attachment is static (!is_dynamic): wait `dma_resv_wait_timeout(resv, DMA_RESV_USAGE_KERNEL, true, MAX_SCHEDULE_TIMEOUT)` — static importers wait for kernel-class fences before returning sg_table.
- dma_buf_wrap_sg_table(&sg_table) — if CONFIG_DMABUF_DEBUG, clone sg_table without page_link (catches importer-page abuse).
- If CONFIG_DMA_API_DEBUG: verify each (sg_dma_address, sg_dma_len) PAGE_SIZE-aligned.
- Return sg_table (ERR_PTR with unmap_dma_buf + unpin rollback on error).

REQ-14: dma_buf_map_attachment_unlocked(attach, direction):
- might_sleep; WARN-input.
- dma_resv_lock; sg_table = dma_buf_map_attachment(attach, direction); dma_resv_unlock.

REQ-15: dma_buf_unmap_attachment(attach, sg_table, direction):
- might_sleep; WARN if `!attach ∨ !attach->dmabuf ∨ !sg_table`.
- dma_resv_assert_held.
- dma_buf_unwrap_sg_table — restore exporter's sg_table when CONFIG_DMABUF_DEBUG.
- ops->unmap_dma_buf(attach, sg_table, direction).
- if dma_buf_pin_on_map: ops->unpin(attach).

REQ-16: dma_buf_unmap_attachment_unlocked: dma_resv_lock-wrapped variant.

REQ-17: dma_buf_attach_revocable(attach):
- attach->importer_ops ∧ attach->importer_ops->invalidate_mappings.

REQ-18: dma_buf_invalidate_mappings(dmabuf):
- dma_resv_assert_held(dmabuf->resv).
- list_for_each_entry(attach, &attachments, node): if importer_ops has invalidate_mappings, call it.
- Importers must complete unmap within bounded time (revoke contract).

REQ-19: dma_buf_begin_cpu_access(dmabuf, direction):
- WARN if !dmabuf ⟹ -EINVAL.
- might_lock(&dmabuf->resv->lock.base).
- if ops->begin_cpu_access: ret = ops->begin_cpu_access(dmabuf, direction); else 0.
- if ret == 0: __dma_buf_begin_cpu_access → dma_resv_wait_timeout(resv, dma_resv_usage_rw(write), true, MAX_SCHEDULE_TIMEOUT). write = (direction == DMA_BIDIRECTIONAL ∨ DMA_TO_DEVICE).

REQ-20: dma_buf_end_cpu_access(dmabuf, direction):
- WARN if !dmabuf; might_lock(resv->lock.base).
- if ops->end_cpu_access: ret = ops->end_cpu_access; else 0.

REQ-21: dma_buf_mmap(dmabuf, vma, pgoff):
- WARN if `!dmabuf ∨ !vma` ⟹ -EINVAL.
- if !ops->mmap ⟹ -EINVAL.
- if `pgoff + vma_pages(vma) < pgoff` ⟹ -EOVERFLOW (overflow).
- if `pgoff + vma_pages(vma) > dmabuf->size >> PAGE_SHIFT` ⟹ -EINVAL.
- vma_set_file(vma, dmabuf->file); vma->vm_pgoff = pgoff.
- Return ops->mmap(dmabuf, vma).

REQ-22: dma_buf_mmap_internal (fops.mmap, user-fd mmap):
- is_dma_buf_file check ⟹ -EINVAL.
- ops->mmap required.
- vm_pgoff + vma_pages > size >> PAGE_SHIFT ⟹ -EINVAL.
- Returns ops->mmap(dmabuf, vma).

REQ-23: dma_buf_llseek(file, offset, whence):
- is_dma_buf_file ⟹ -EBADF.
- whence == SEEK_END ⟹ base = size.
- whence == SEEK_SET ⟹ base = 0.
- else ⟹ -EINVAL.
- offset != 0 ⟹ -EINVAL.
- Return base.

REQ-24: dma_buf_vmap(dmabuf, map):
- iosys_map_clear(map).
- WARN if !dmabuf ⟹ -EINVAL.
- dma_resv_assert_held(dmabuf->resv).
- if !ops->vmap ⟹ -EINVAL.
- if vmapping_counter != 0: increment; BUG_ON iosys_map_is_null(&vmap_ptr); *map = vmap_ptr; return 0.
- BUG_ON iosys_map_is_set(&vmap_ptr).
- ret = ops->vmap(dmabuf, &ptr).
- vmap_ptr = ptr; vmapping_counter = 1; *map = vmap_ptr.

REQ-25: dma_buf_vunmap(dmabuf, map):
- WARN if !dmabuf.
- dma_resv_assert_held.
- BUG_ON iosys_map_is_null(&vmap_ptr) ∨ vmapping_counter == 0 ∨ !iosys_map_is_equal(&vmap_ptr, map).
- if `--vmapping_counter == 0`: if ops->vunmap: ops->vunmap(dmabuf, map); iosys_map_clear(&vmap_ptr).

REQ-26: dma_buf_vmap_unlocked / vunmap_unlocked: dma_resv_lock wrappers.

REQ-27: dma_buf_poll(file, poll_table):
- dmabuf = file->private_data; if `!dmabuf ∨ !resv` ⟹ EPOLLERR.
- poll_wait(file, &dmabuf->poll, poll).
- events = poll_requested_events & (EPOLLIN | EPOLLOUT). If 0 return 0.
- dma_resv_lock(resv, NULL).
- For each direction (EPOLLOUT then EPOLLIN):
  - dcb = &cb_out or &cb_in.
  - Spinlock dmabuf->poll.lock: if dcb->active set, drop event; else dcb->active = direction.
  - If event still set: get_file(dmabuf->file) (paired with fput in dma_buf_poll_cb).
  - dma_buf_poll_add_cb(resv, write?, dcb): iterate dma_resv via dma_resv_for_each_fence; dma_fence_get + dma_fence_add_callback(fence, &dcb->cb, dma_buf_poll_cb). Return true on first successful add.
  - If add_cb returned false (no unsignaled fences left): dma_buf_poll_cb(NULL, &dcb->cb) immediately (wakes any other waiter).
  - Otherwise: events &= ~direction (still pending).
- dma_resv_unlock; return events.

REQ-28: dma_buf_poll_cb(fence, cb):
- dcb = container_of(cb, struct dma_buf_poll_cb_t, cb); dmabuf = container_of(dcb->poll, struct dma_buf, poll).
- spin_lock_irqsave(&dcb->poll->lock): wake_up_locked_poll(dcb->poll, dcb->active); dcb->active = 0; spin_unlock_irqrestore.
- dma_fence_put(fence); fput(dmabuf->file) (paired with get_file).

REQ-29: dma_buf_ioctl(file, cmd, arg):
- DMA_BUF_IOCTL_SYNC: copy_from_user dma_buf_sync. flags & ~DMA_BUF_SYNC_VALID_FLAGS_MASK ⟹ -EINVAL. direction = (READ⟹FROM_DEVICE, WRITE⟹TO_DEVICE, RW⟹BIDIRECTIONAL, else⟹-EINVAL). If DMA_BUF_SYNC_END: dma_buf_end_cpu_access(dmabuf, direction); else dma_buf_begin_cpu_access. Return ret.
- DMA_BUF_SET_NAME_A / _B: dma_buf_set_name.
- DMA_BUF_IOCTL_EXPORT_SYNC_FILE (CONFIG_SYNC_FILE): dma_buf_export_sync_file.
- DMA_BUF_IOCTL_IMPORT_SYNC_FILE (CONFIG_SYNC_FILE): dma_buf_import_sync_file.
- default ⟹ -ENOTTY.

REQ-30: dma_buf_set_name(dmabuf, user_buf):
- name = strndup_user(user_buf, DMA_BUF_NAME_LEN). IS_ERR ⟹ PTR_ERR.
- spin_lock(&name_lock); kfree(dmabuf->name); dmabuf->name = name; spin_unlock.

REQ-31: dma_buf_export_sync_file(dmabuf, user):
- copy_from_user arg; arg.flags & ~DMA_BUF_SYNC_RW ⟹ -EINVAL; (arg.flags & DMA_BUF_SYNC_RW) == 0 ⟹ -EINVAL.
- fd = get_unused_fd_flags(O_CLOEXEC).
- usage = dma_resv_usage_rw(arg.flags & DMA_BUF_SYNC_WRITE).
- dma_resv_get_singleton(resv, usage, &fence) — collapses set to a single fence (may be a dma_fence_chain).
- If fence is NULL: dma_fence_get_stub() (already-signaled).
- sync_file = sync_file_create(fence); dma_fence_put(fence).
- copy_to_user arg with fd; fd_install(fd, sync_file->file).

REQ-32: dma_buf_import_sync_file(dmabuf, user):
- copy_from_user; flags check (must be RW-subset, non-zero).
- fence = sync_file_get_fence(arg.fd). NULL ⟹ -EINVAL.
- usage = (WRITE ⟹ DMA_RESV_USAGE_WRITE, else READ).
- num_fences = count via dma_fence_unwrap_for_each.
- dma_resv_lock; dma_resv_reserve_fences(resv, num_fences); dma_fence_unwrap_for_each → dma_resv_add_fence(resv, f, usage); dma_resv_unlock.
- dma_fence_put(fence).

REQ-33: dma_buf_iter_begin / dma_buf_iter_next:
- Iterate `dmabuf_list` under `dmabuf_list_mutex`.
- get_file on dmabuf->file (ref-bumped iter element); previous element released by caller.

REQ-34: dma_buf_show_fdinfo(seq, file):
- size, count = file_count(file) - 1, exp_name, name (under name_lock).

REQ-35: dma_buf_fops:
- release = dma_buf_file_release (`__dma_buf_list_del` and -EINVAL guard).
- mmap = dma_buf_mmap_internal.
- llseek = dma_buf_llseek.
- poll = dma_buf_poll.
- unlocked_ioctl = dma_buf_ioctl; compat_ioctl = compat_ptr_ioctl.
- show_fdinfo = dma_buf_show_fdinfo.

REQ-36: dma_buf_fs_type:
- name "dmabuf"; init_fs_context = init_pseudo(fc, DMA_BUF_MAGIC), dops = &dma_buf_dentry_ops; kill_sb = kill_anon_super.

REQ-37: Locking convention (documented in DOC: locking convention):
- Importers MUST hold dma-buf resv lock before: dma_buf_pin / unpin / map_attachment / unmap_attachment / vmap / vunmap.
- Importers MUST NOT hold resv lock before: attach / dynamic_attach / detach / export / fd / get / put / mmap / begin_cpu_access / end_cpu_access / map_attachment_unlocked / unmap_attachment_unlocked / vmap_unlocked / vunmap_unlocked.
- Exporter callbacks invoked with resv UN-locked: attach / detach / release / begin_cpu_access / end_cpu_access / mmap (exporter may take it).
- Exporter callbacks invoked with resv LOCKED: pin / unpin / map_dma_buf / unmap_dma_buf / vmap / vunmap (exporter must NOT take it).
- Exporter MUST hold resv lock before dma_buf_invalidate_mappings.

REQ-38: dma_buf_init / dma_buf_deinit (subsys_initcall / __exitcall):
- kern_mount(&dma_buf_fs_type) → dma_buf_mnt; debugfs init/uninit.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `export_required_ops` | INVARIANT | per-export: ops.map_dma_buf ∧ ops.unmap_dma_buf ∧ ops.release ⟹ Ok, else Err(EINVAL). |
| `export_pin_unpin_paired` | INVARIANT | per-export: !ops.pin == !ops.unpin. |
| `get_typecheck` | INVARIANT | per-get: file.f_op == &dma_buf_fops ⟹ valid, else Err(EINVAL). |
| `release_no_attachments` | INVARIANT | per-release: WARN if attachments list non-empty. |
| `release_no_vmap` | INVARIANT | per-release: BUG_ON(vmapping_counter != 0). |
| `release_no_pending_poll` | INVARIANT | per-release: BUG_ON(cb_in.active ∨ cb_out.active). |
| `map_resv_lock_held` | INVARIANT | per-map_attachment: dma_resv_assert_held. |
| `unmap_resv_lock_held` | INVARIANT | per-unmap_attachment: dma_resv_assert_held. |
| `pin_unpin_resv_held` | INVARIANT | per-pin / per-unpin: dma_resv_assert_held. |
| `vmap_refcount_consistent` | INVARIANT | per-vmap: counter > 0 ⟹ vmap_ptr non-null; per-vunmap: counter == 0 ⟹ vmap_ptr cleared. |
| `mmap_size_bounds` | INVARIANT | per-mmap: pgoff + vma_pages ≤ size >> PAGE_SHIFT ∧ no overflow. |
| `poll_get_put_file_paired` | INVARIANT | per-poll: get_file paired with fput in poll_cb. |
| `dynamic_static_branch` | INVARIANT | per-map_attachment: static ⟹ wait DMA_RESV_USAGE_KERNEL; dynamic ⟹ no wait. |
| `import_unwrap_count_matches` | INVARIANT | per-import_sync_file: reserve count == unwrap count. |

### Layer 2: TLA+

`drivers/dma-buf/dma-buf.tla`:
- Per-export + per-attach + per-map + per-unmap + per-detach + per-release lifecycle.
- Properties:
  - `safety_attachment_implies_buffer_live` — attach exists ⟹ dmabuf refcount > 0.
  - `safety_no_release_with_attachments` — release ⟹ attachments list empty.
  - `safety_vmap_refcount_balanced` — sum vmap == sum vunmap before release.
  - `safety_poll_get_put_balanced` — each poll-direction-arm: get_file/fput balanced.
  - `safety_lock_convention_obeyed` — per-locking-convention table.
  - `liveness_poll_eventual_signal` — fence signaled ⟹ EPOLL{IN,OUT} delivered.
  - `liveness_map_eventual` — static map_attachment returns or times out.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DmaBuf::export` post: dmabuf in `LIST` ∧ file refcount 1 ∧ ops set | `DmaBuf::export` |
| `DmaBuf::get` post: returned dmabuf == file.private_data ∧ is_dma_buf_file | `DmaBuf::get` |
| `DmaBuf::release` pre: attachments empty ∧ vmapping_counter == 0 ∧ no pending poll | `DmaBuf::release` |
| `DmaBufAttachment::map` post: returns sg_table OR exporter callbacks rolled back | `DmaBufAttachment::map` |
| `DmaBuf::vmap` post: counter ≥ 1 ∧ vmap_ptr non-null | `DmaBuf::vmap` |
| `DmaBuf::vunmap` post: counter == 0 ⟹ ops.vunmap called | `DmaBuf::vunmap` |
| `DmaBuf::mmap` post: vma->vm_file == dmabuf.file ∧ vm_pgoff == pgoff | `DmaBuf::mmap` |
| `DmaBuf::poll` post: returned mask only contains direction with no pending callback | `DmaBuf::poll` |
| `DmaBuf::import_sync_file` post: dma_resv has ≥ pre-existing-count fences | `DmaBuf::import_sync_file` |
| `DmaBuf::export_sync_file` post: returned fd is sync_file containing singleton fence | `DmaBuf::export_sync_file` |

### Layer 4: Verus/Creusot functional

`Per-export → ((kern_mount-anon-inode pseudo-fs allocation) ∧ (priv/ops bound) ∧ (resv embedded-or-external) ∧ (dma_buf_list_add)) → per-attach (importer registers, list grows, exporter notified) → per-map (resv lock + pin? + ops.map_dma_buf + USAGE_KERNEL wait if static) → per-cpu-access (resv usage_rw wait + exporter bracket) → per-mmap (vma_set_file + ops.mmap) → per-unmap (ops.unmap_dma_buf + unpin?) → per-detach (list_del + ops.detach) → per-put (fput → dentry-release → ops.release + kfree)` semantic equivalence with `Documentation/driver-api/dma-buf.rst` and `include/linux/dma-buf.h` contracts (locking convention, refcount, sg_table mapping semantics, sync_file ioctl semantics).

### hardening

(Inherits row-1 features from `drivers/dma-buf/00-overview.md` § Hardening.)

DMA-BUF reinforcement:

- **Per-fops typecheck (is_dma_buf_file)** — defense against per-foreign-fd confusion in dma_buf_get and dma_buf_file_release.
- **Per-ops nullability checks at export** — defense against per-incomplete exporter (`map_dma_buf`/`unmap_dma_buf`/`release` mandatory).
- **Per-pin/unpin paired** — defense against per-asymmetric exporter (refcount leak / underflow at runtime).
- **Per-dma_resv lock-asserts at map / unmap / pin / unpin / vmap / vunmap / invalidate_mappings** — defense against per-importer-skipped-lock corruption.
- **Per-static-attachment USAGE_KERNEL fence-wait before sg_table** — defense against per-importer reads uninitialized device-side state.
- **Per-CONFIG_DMABUF_DEBUG sg_table wrap (no page_link)** — defense against per-importer dereferences struct page that exporter didn't expect.
- **Per-CONFIG_DMA_API_DEBUG PAGE_SIZE-alignment audit** — defense against per-exporter unaligned dma_address (broken IOMMU).
- **Per-vmap refcount + iosys_map_is_null/_set assertions** — defense against per-double-vmap or per-vunmap-without-vmap.
- **Per-mmap pgoff overflow + bounds check** — defense against per-overflowing-pgoff producing OOB mapping.
- **Per-poll get_file/fput pairing across async fence callback** — defense against per-poll-cb UAF if dmabuf released while callback queued.
- **Per-dma_buf_release BUG_ON for active cb_in/cb_out** — defense against per-release-with-pending-poll callbacks UAF.
- **Per-export module_get(owner) + module_put(owner) at release** — defense against per-exporter-module-unload while buffer live.
- **Per-DMA_BUF_NAME_LEN bound in set_name** — defense against per-unbounded-userspace-name allocation.
- **Per-DMA_BUF_SYNC_VALID_FLAGS_MASK check** — defense against per-future-flag-bits silently honored.

