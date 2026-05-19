# Tier-3: net/9p/trans_virtio.c — 9P virtio transport (virtio-9p / VIRTIO_ID_9P)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/9p/00-overview.md
upstream-paths:
  - net/9p/trans_virtio.c (~837 lines)
  - net/9p/trans_common.h
  - include/net/9p/transport.h
  - include/net/9p/client.h
  - include/linux/virtio.h
  - include/linux/virtio_9p.h (VIRTIO_ID_9P, VIRTIO_9P_MOUNT_TAG, struct virtio_9p_config)
-->

## Summary

The **virtio-9p** transport binds a `struct p9_client` to a VirtIO-PCI device of class `VIRTIO_ID_9P`, exposed by the host hypervisor (QEMU `-virtfs` / `-fsdev`). Per-channel state lives in `struct virtio_chan` and owns exactly one virtqueue (`"requests"`) of depth `VIRTQUEUE_NUM=128`. Per-T-message issuance: `p9_virtio_request` packs the encoded `req->tc` (T-frame) and the empty `req->rc` (R-frame) into two `scatterlist`s, calls `virtqueue_add_sgs` then `virtqueue_kick`; the response arrives via the `req_done` callback which pulls completed descriptors via `virtqueue_get_buf` and notifies the client via `p9_client_cb(..., REQ_STATUS_RCVD)`. Per-zero-copy path (`p9_virtio_zc_request`): payload pages from a user-iov-iter are pinned (via `iov_iter_get_pages_alloc2`) and stitched into the sg list so READ/WRITE data bypasses the bounce-buffer; up to `chan->p9_max_pages = nr_free_buffer_pages()/4` pages may be pinned globally (`vp_pinned` atomic). Per-mount-tag discovery: the host advertises a `tag` string in `struct virtio_9p_config` if `VIRTIO_9P_MOUNT_TAG` is negotiated; the tag is exported via the `mount_tag` sysfs attribute and matched by `p9_virtio_create` against the user-supplied `devname` from `fs_context`. Critical for: QEMU/KVM file-passthrough, container runtime shared-fs, microvm rootfs.

This Tier-3 covers `net/9p/trans_virtio.c` (~837 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct virtio_chan` | per-instance transport state | `VirtioChan` |
| `struct p9_trans_module p9_virtio_trans` | per-vtable for client | `VIRTIO_TRANS: P9TransModule` |
| `struct virtio_driver p9_virtio_drv` | per-virtio-bus driver | `P9_VIRTIO_DRV: VirtioDriver` |
| `id_table[VIRTIO_ID_9P]` | per-device match | `ID_TABLE` |
| `p9_virtio_probe()` | per-bus probe | `VirtioChan::probe` |
| `p9_virtio_remove()` | per-bus remove | `VirtioChan::remove` |
| `p9_virtio_create()` | per-mount channel-bind | `VirtioChan::create` |
| `p9_virtio_close()` | per-unmount channel-release | `VirtioChan::close` |
| `p9_virtio_request()` | per-T-message issue | `VirtioChan::request` |
| `p9_virtio_zc_request()` | per-zero-copy T-message | `VirtioChan::zc_request` |
| `p9_virtio_cancel()` | per-cancel (returns 1 = not cancellable) | `VirtioChan::cancel` |
| `p9_virtio_cancelled()` | per-acked cancel (drop ref) | `VirtioChan::cancelled` |
| `req_done()` | per-completion virtqueue callback | `VirtioChan::req_done` |
| `pack_sg_list()` | per-linear-buffer sg pack | `VirtioChan::pack_sg_list` |
| `pack_sg_list_p()` | per-page-array sg pack | `VirtioChan::pack_sg_list_p` |
| `p9_get_mapped_pages()` | per-iov-iter pin-or-resolve | `VirtioChan::get_mapped_pages` |
| `handle_rerror()` | per-RERROR-string copy-back | `VirtioChan::handle_rerror` |
| `p9_mount_tag_show()` | per-sysfs attr show | `VirtioChan::mount_tag_show` |
| `dev_attr_mount_tag` | per-sysfs attribute | `DEV_ATTR_MOUNT_TAG` |
| `virtio_chan_list` (global) | per-driver registry | `VIRTIO_CHAN_LIST` |
| `virtio_9p_lock` (global mutex) | per-list + inuse serialization | `VIRTIO_9P_LOCK` |
| `vp_pinned` (global atomic) | per-system pinned-page accounting | `VP_PINNED` |
| `vp_wq` (global waitqueue) | per-pin-quota wakeup | `VP_WQ` |

## Compatibility contract

REQ-1: struct virtio_chan:
- inuse: bool — exclusive-mount flag, guarded by `virtio_9p_lock`.
- lock: spinlock_t — protects vq access, ring_bufs_avail, req_done path.
- client: *p9_client — backref set by create / cleared by close.
- vdev: *virtio_device — VirtIO-bus device.
- vq: *virtqueue — exactly one "requests" virtqueue.
- ring_bufs_avail: int — 1 ⟺ at least one ring slot free; 0 ⟺ wait on vc_wq.
- vc_wq: *wait_queue_head_t — sleeper on ENOSPC retry.
- p9_max_pages: ulong — per-channel ceiling = nr_free_buffer_pages()/4.
- sg[VIRTQUEUE_NUM]: scatterlist — per-channel pre-allocated sg pool.
- tag: *char — null-terminated mount-tag string from host config.
- chan_list: list_head — link in global virtio_chan_list.

REQ-2: struct p9_trans_module p9_virtio_trans:
- name = "virtio".
- create = p9_virtio_create, close = p9_virtio_close.
- request = p9_virtio_request, zc_request = p9_virtio_zc_request.
- cancel = p9_virtio_cancel, cancelled = p9_virtio_cancelled.
- maxsize = PAGE_SIZE * (VIRTQUEUE_NUM - 3) — reserve 3 sg slots (in-header / response-header / off-page slack).
- pooled_rbuffers = false (client owns rc.sdata per req).
- def = true — default transport when none specified.
- supports_vmalloc = false — sg requires physical contiguity per page; vmalloc-backed buffers rejected.
- owner = THIS_MODULE.

REQ-3: p9_virtio_probe(vdev):
- /* Verify config-access is enabled */
- if !vdev->config->get: return -EINVAL.
- chan = kmalloc(sizeof(*chan), GFP_KERNEL).
- chan->vdev = vdev.
- /* Single virtqueue: "requests" */
- chan->vq = virtio_find_single_vq(vdev, req_done, "requests").
- chan->vq->vdev->priv = chan.
- spin_lock_init(&chan->lock).
- sg_init_table(chan->sg, VIRTQUEUE_NUM).
- chan->inuse = false.
- /* MOUNT_TAG feature mandatory */
- if !virtio_has_feature(vdev, VIRTIO_9P_MOUNT_TAG): error -EINVAL.
- virtio_cread(vdev, struct virtio_9p_config, tag_len, &tag_len).
- tag = kzalloc(tag_len + 1, GFP_KERNEL).
- virtio_cread_bytes(vdev, offsetof(struct virtio_9p_config, tag), tag, tag_len).
- /* Sysfs attribute */
- sysfs_create_file(&vdev->dev.kobj, &dev_attr_mount_tag.attr).
- /* Per-channel waitqueue */
- chan->vc_wq = kmalloc(sizeof(wait_queue_head_t), GFP_KERNEL).
- init_waitqueue_head(chan->vc_wq).
- chan->ring_bufs_avail = 1.
- chan->p9_max_pages = nr_free_buffer_pages() / 4.
- virtio_device_ready(vdev).
- /* Register in global list */
- mutex_lock(&virtio_9p_lock); list_add_tail(&chan->chan_list, &virtio_chan_list); mutex_unlock.
- /* Fire udev event */
- kobject_uevent(&vdev->dev.kobj, KOBJ_CHANGE).

REQ-4: p9_virtio_create(client, fc):
- devname = fc->source (must be non-NULL → -EINVAL).
- mutex_lock(&virtio_9p_lock).
- /* Linear search by tag */
- for chan in virtio_chan_list:
  - if strcmp(devname, chan->tag) == 0:
    - if !chan->inuse: chan->inuse = true; found = 1; break.
    - else: ret = -EBUSY (continue to check for free duplicate tag).
- mutex_unlock(&virtio_9p_lock).
- if !found: return -ENOENT (or -EBUSY if tag matched but busy).
- client->trans = chan.
- client->status = Connected.
- chan->client = client.
- return 0.

REQ-5: p9_virtio_close(client):
- chan = client->trans.
- mutex_lock(&virtio_9p_lock).
- if chan: chan->inuse = false.
- mutex_unlock(&virtio_9p_lock).
- /* Does NOT free chan; chan is bus-owned and freed by p9_virtio_remove */

REQ-6: p9_virtio_request(client, req):
- chan = client->trans.
- WRITE_ONCE(req->status, REQ_STATUS_SENT).
- req_retry:
  - spin_lock_irqsave(&chan->lock, flags).
  - /* Pack out (T-frame) */
  - out = pack_sg_list(chan->sg, 0, VIRTQUEUE_NUM, req->tc.sdata, req->tc.size).
  - if out: sgs[out_sgs++] = chan->sg.
  - /* Pack in (R-frame buffer) */
  - in = pack_sg_list(chan->sg, out, VIRTQUEUE_NUM, req->rc.sdata, req->rc.capacity).
  - if in: sgs[out_sgs + in_sgs++] = chan->sg + out.
  - err = virtqueue_add_sgs(chan->vq, sgs, out_sgs, in_sgs, req, GFP_ATOMIC).
  - if err == -ENOSPC:
    - chan->ring_bufs_avail = 0.
    - spin_unlock_irqrestore.
    - io_wait_event_killable(*chan->vc_wq, chan->ring_bufs_avail).
    - if -ERESTARTSYS: return -ERESTARTSYS.
    - goto req_retry.
  - else if err < 0: spin_unlock; return -EIO.
- virtqueue_kick(chan->vq).
- spin_unlock_irqrestore.
- return 0.

REQ-7: p9_virtio_zc_request(client, req, uidata, uodata, inlen, outlen, in_hdr_len):
- /* Pin payload pages (read/write payload bypasses bounce-buffer) */
- if uodata: n = p9_get_mapped_pages(chan, &out_pages, uodata, outlen, &offs, &need_drop).
  - if n < 0: err = n; goto err_out.
  - out_nr_pages = DIV_ROUND_UP(n + offs, PAGE_SIZE).
  - /* If short pin, patch sz fields in T-frame */
  - if n != outlen: rewrite trailing __le32 length + total size header.
- else if uidata: symmetric for read direction.
- WRITE_ONCE(req->status, REQ_STATUS_SENT).
- req_retry_pinned:
  - spin_lock_irqsave(&chan->lock, flags).
  - /* sg layout: [T-frame static] [out-payload pages] [R-header static] [in-payload pages] */
  - out = pack_sg_list(chan->sg, 0, VIRTQUEUE_NUM, req->tc.sdata, req->tc.size).
  - if out: sgs[out_sgs++] = chan->sg.
  - if out_pages: sgs[out_sgs++] = chan->sg + out; out += pack_sg_list_p(...).
  - in = pack_sg_list(chan->sg, out, VIRTQUEUE_NUM, req->rc.sdata, in_hdr_len).
  - if in: sgs[out_sgs + in_sgs++] = chan->sg + out.
  - if in_pages: sgs[out_sgs + in_sgs++] = chan->sg + out + in; pack_sg_list_p(...).
  - BUG_ON(out_sgs + in_sgs > 4).
  - err = virtqueue_add_sgs(...).
  - if -ENOSPC: wait on vc_wq, goto req_retry_pinned.
  - if err < 0: spin_unlock; err = -EIO; goto err_out.
- virtqueue_kick(chan->vq).
- spin_unlock_irqrestore.
- kicked = 1.
- /* Synchronously wait for reply (zc path is blocking) */
- io_wait_event_killable(req->wq, req->status >= REQ_STATUS_RCVD).
- /* If reply was RERROR, copy variable-length error-string from pinned pages into static R-buffer */
- if req->status == REQ_STATUS_RCVD ∧ req->rc.sdata[4] == P9_RERROR:
  - handle_rerror(req, in_hdr_len, offs, in_pages).
- err_out:
  - /* Unpin */
  - if need_drop:
    - if in_pages: p9_release_pages; atomic_sub(in_nr_pages, &vp_pinned).
    - if out_pages: p9_release_pages; atomic_sub(out_nr_pages, &vp_pinned).
    - wake_up(&vp_wq).
  - kvfree(in_pages); kvfree(out_pages).
  - if !kicked: p9_req_put(client, req).  /* reply won't come */
  - return err.

REQ-8: p9_get_mapped_pages(chan, pages, data, count, offs, need_drop):
- if iov_iter_count(data) == 0: return 0.
- /* User-space iov: pin */
- if !iov_iter_is_kvec(data):
  - /* Enforce global pin ceiling */
  - if atomic_read(&vp_pinned) >= chan->p9_max_pages:
    - io_wait_event_killable(vp_wq, atomic_read(&vp_pinned) < chan->p9_max_pages).
    - if -ERESTARTSYS: return err.
  - n = iov_iter_get_pages_alloc2(data, pages, count, offs).
  - if n < 0: return n.
  - *need_drop = 1.
  - nr_pages = DIV_ROUND_UP(n + *offs, PAGE_SIZE).
  - atomic_add(nr_pages, &vp_pinned).
  - return n.
- /* Kernel kvec: resolve to page* via vmalloc_to_page / kmap_to_page */
- else:
  - p = data->kvec->iov_base + data->iov_offset.
  - nr_pages = DIV_ROUND_UP((ulong)p + len, PAGE_SIZE) - (ulong)p / PAGE_SIZE.
  - *pages = kmalloc(nr_pages * sizeof(page*), GFP_NOFS).
  - *need_drop = 0.
  - *offs = offset_in_page(p); p -= *offs.
  - for i in 0..nr_pages: (*pages)[i] = is_vmalloc_addr(p) ? vmalloc_to_page(p) : kmap_to_page(p); p += PAGE_SIZE.
  - iov_iter_advance(data, len).
  - return len.

REQ-9: req_done(vq) [virtqueue callback, interrupt context]:
- chan = vq->vdev->priv.
- spin_lock_irqsave(&chan->lock, flags).
- while (req = virtqueue_get_buf(chan->vq, &len)) != NULL:
  - if !chan->ring_bufs_avail: chan->ring_bufs_avail = 1; need_wakeup = true.
  - if len: req->rc.size = len; p9_client_cb(chan->client, req, REQ_STATUS_RCVD).
- spin_unlock_irqrestore.
- if need_wakeup: wake_up(chan->vc_wq).

REQ-10: pack_sg_list(sg, start, limit, data, count):
- /* Split linear buffer at page boundaries (DMA requires page granularity) */
- while count > 0:
  - s = min(rest_of_page(data), count).
  - BUG_ON(index >= limit).
  - sg_unmark_end(&sg[index]).
  - sg_set_buf(&sg[index++], data, s).
  - count -= s; data += s.
- if index-start > 0: sg_mark_end(&sg[index-1]).
- return index - start.

REQ-11: pack_sg_list_p(sg, start, limit, pdata, nr_pages, offs, count):
- BUG_ON(nr_pages > (limit - start)).
- data_off = offs.
- while nr_pages > 0:
  - s = min(PAGE_SIZE - data_off, count).
  - sg_unmark_end(&sg[index]).
  - sg_set_page(&sg[index++], pdata[i++], s, data_off).
  - data_off = 0; count -= s; nr_pages--.
- if index-start > 0: sg_mark_end(&sg[index-1]).
- return index - start.

REQ-12: p9_virtio_cancel: return 1 unconditionally — virtio-9p cannot rescind a posted descriptor; client must wait. (Reflects host-side serialization assumption.)

REQ-13: p9_virtio_cancelled: p9_req_put(client, req); return 0 — caller has given up; drop ref so req frees when host eventually replies.

REQ-14: handle_rerror(req, in_hdr_len, offs, pages):
- /* RERROR variable-length error-string was DMA'd into pinned pages, not static R-buf; copy back */
- if req->rc.size < in_hdr_len ∨ !pages: return.
- if req->rc.size > P9_ZC_HDR_SZ: req->rc.size = P9_ZC_HDR_SZ (truncate).
- size = req->rc.size - in_hdr_len.
- n = PAGE_SIZE - offs.
- if size > n: memcpy_from_page(to, *pages++, offs, n); offs = 0; to += n; size -= n.
- memcpy_from_page(to, *pages, offs, size).

REQ-15: Mount-tag discovery (sysfs):
- /sys/bus/virtio/devices/virtioN/mount_tag exports `chan->tag` (null-terminated).
- p9_mount_tag_show: memcpy(buf, chan->tag, strlen(chan->tag) + 1); return strlen + 1.
- Permissions: 0444 (world-readable, no write).
- udev rules consume this attribute to auto-construct `mount -t 9p -o trans=virtio <tag> /mnt`.

REQ-16: p9_virtio_remove(vdev) [bus-disconnect]:
- chan = vdev->priv.
- mutex_lock(&virtio_9p_lock).
- list_del(&chan->chan_list).  /* No new users */
- warning_time = jiffies.
- /* Wait for outstanding mounts to release */
- while chan->inuse:
  - mutex_unlock; msleep(250); mutex_lock.
  - if time_after(jiffies, warning_time + 10*HZ): dev_emerg("waiting for device in use"); warning_time = jiffies.
- mutex_unlock.
- virtio_reset_device(vdev).
- vdev->config->del_vqs(vdev).
- sysfs_remove_file(&vdev->dev.kobj, &dev_attr_mount_tag.attr).
- kobject_uevent(&vdev->dev.kobj, KOBJ_CHANGE).
- kfree(chan->tag); kfree(chan->vc_wq); kfree(chan).

REQ-17: Module init/exit:
- p9_virtio_init: INIT_LIST_HEAD(&virtio_chan_list); v9fs_register_trans(&p9_virtio_trans); register_virtio_driver(&p9_virtio_drv); rollback on failure.
- p9_virtio_cleanup: unregister_virtio_driver; v9fs_unregister_trans.

## Acceptance Criteria

- [ ] AC-1: virtio-9p device with matching `mount_tag` enumerated → mount -t 9p -o trans=virtio,<tag> succeeds.
- [ ] AC-2: Two concurrent mounts of the same tag: second returns -EBUSY.
- [ ] AC-3: Mount with unknown tag: returns -ENOENT.
- [ ] AC-4: p9_virtio_request issues a T-message and req_done delivers REQ_STATUS_RCVD with correct len.
- [ ] AC-5: ENOSPC on virtqueue_add_sgs: caller sleeps on vc_wq and retries when req_done frees slot.
- [ ] AC-6: zero-copy TREAD with user buffer: payload pages pinned, DMA'd directly, unpinned on completion.
- [ ] AC-7: zero-copy reply with RERROR: error-string copied back into static R-buf via handle_rerror.
- [ ] AC-8: vp_pinned ceiling honored: 5th concurrent zc_request blocks until prior unpin.
- [ ] AC-9: p9_virtio_cancel returns 1 (no cancel); p9_virtio_cancelled drops req ref.
- [ ] AC-10: p9_virtio_remove with active mount: blocks until inuse cleared (logs dev_emerg every 10s).
- [ ] AC-11: VIRTIO_9P_MOUNT_TAG feature absent at probe: probe fails -EINVAL.
- [ ] AC-12: kvec (kernel) iov-iter: no pin, kmap_to_page / vmalloc_to_page resolution path.
- [ ] AC-13: maxsize = PAGE_SIZE * (VIRTQUEUE_NUM - 3) exposed to client via p9_trans_module.

## Architecture

```
struct VirtioChan {
  inuse: AtomicBool,                  // guarded by VIRTIO_9P_LOCK
  lock: SpinLock<()>,                 // per-chan req-path serialization
  client: AtomicPtr<P9Client>,        // backref
  vdev: NonNull<VirtioDevice>,
  vq: NonNull<Virtqueue>,             // "requests" vq, depth 128
  ring_bufs_avail: AtomicI32,         // 0 = full, 1 = slot free
  vc_wq: NonNull<WaitQueueHead>,
  p9_max_pages: usize,                // nr_free_buffer_pages() / 4
  sg: [Scatterlist; VIRTQUEUE_NUM],   // pre-allocated, reused per req
  tag: CStringBuf,                    // null-terminated mount tag
  chan_list: ListHead,                // VIRTIO_CHAN_LIST link
}

struct P9VirtioTrans;                 // ZST implementing P9TransModule

struct P9VirtioDrv;                   // ZST implementing VirtioDriver
```

`VirtioChan::probe(vdev) -> Result<()>`:
1. require vdev.config.get != None else -EINVAL.
2. chan = Box::try_new(VirtioChan { .. })?.
3. chan.vq = virtio_find_single_vq(vdev, req_done, c"requests")?.
4. vq.vdev_mut().priv = &mut *chan.
5. sg_init_table(&mut chan.sg).
6. require virtio_has_feature(vdev, VIRTIO_9P_MOUNT_TAG) else -EINVAL.
7. tag_len = virtio_cread::<u16>(vdev, offsetof(virtio_9p_config, tag_len)).
8. tag = CStringBuf::with_capacity(tag_len + 1).
9. virtio_cread_bytes(vdev, offsetof(virtio_9p_config, tag), &mut tag, tag_len).
10. sysfs_create_file(&vdev.dev.kobj, &DEV_ATTR_MOUNT_TAG)?.
11. chan.vc_wq = WaitQueueHead::new_boxed().
12. chan.ring_bufs_avail = 1; chan.p9_max_pages = nr_free_buffer_pages() / 4.
13. virtio_device_ready(vdev).
14. with VIRTIO_9P_LOCK held: VIRTIO_CHAN_LIST.push_tail(&chan.chan_list).
15. kobject_uevent(&vdev.dev.kobj, KOBJ_CHANGE).

`VirtioChan::create(client, fc) -> Result<()>`:
1. devname = fc.source.ok_or(-EINVAL)?.
2. let mut ret = -ENOENT; let mut found = None.
3. with VIRTIO_9P_LOCK held:
   - for chan in VIRTIO_CHAN_LIST.iter():
     - if chan.tag == devname:
       - if !chan.inuse.load(): chan.inuse.store(true); found = Some(chan); break.
       - else: ret = -EBUSY.
4. let chan = found.ok_or(ret)?.
5. client.trans = chan as *mut _.
6. client.status = Connected.
7. chan.client = client.
8. Ok(()).

`VirtioChan::close(client)`:
1. let chan = client.trans as *mut VirtioChan.
2. with VIRTIO_9P_LOCK held: if !chan.is_null(): (*chan).inuse.store(false).
3. /* No free; bus-owned */.

`VirtioChan::request(client, req) -> Result<()>`:
1. let chan = client.trans.
2. req.status.store(REQ_STATUS_SENT).
3. loop {  // req_retry
   - spin_lock_irqsave(&chan.lock, flags).
   - out = pack_sg_list(&mut chan.sg, 0, VIRTQUEUE_NUM, req.tc.sdata, req.tc.size).
   - sgs[0] = if out > 0 { Some(&chan.sg[..out]) } else { None }.
   - in_ = pack_sg_list(&mut chan.sg, out, VIRTQUEUE_NUM, req.rc.sdata, req.rc.capacity).
   - sgs[..] arranged: out_sgs out-going, in_sgs in-coming.
   - match virtqueue_add_sgs(chan.vq, sgs, out_sgs, in_sgs, req, GFP_ATOMIC) {
     - Ok(_) => { virtqueue_kick(chan.vq); spin_unlock; return Ok(()); }
     - Err(-ENOSPC) => { chan.ring_bufs_avail = 0; spin_unlock; io_wait_event_killable(chan.vc_wq, chan.ring_bufs_avail)?; continue; }
     - Err(_) => { spin_unlock; return Err(-EIO); }
   - }
4. }.

`VirtioChan::zc_request(client, req, uidata, uodata, inlen, outlen, in_hdr_len) -> Result<()>`:
1. let mut in_pages = None; let mut out_pages = None.
2. let mut offs = 0; let mut need_drop = false.
3. if let Some(uo) = uodata {
   - let n = VirtioChan::get_mapped_pages(chan, &mut out_pages, uo, outlen, &mut offs, &mut need_drop)?.
   - if n != outlen: patch trailing __le32 + total-size header.
4. } else if let Some(ui) = uidata {
   - similar for read direction.
5. }.
6. req.status.store(REQ_STATUS_SENT).
7. loop {  // req_retry_pinned
   - spin_lock_irqsave(&chan.lock, flags).
   - layout sg: [T-frame] [out-pages?] [R-header] [in-pages?].
   - assert!(out_sgs + in_sgs <= 4).
   - match virtqueue_add_sgs(...) { /* same ENOSPC / EIO handling */ }.
   - virtqueue_kick(chan.vq); spin_unlock; kicked = true; break.
8. }.
9. io_wait_event_killable(req.wq, req.status >= REQ_STATUS_RCVD).
10. if req.status == REQ_STATUS_RCVD ∧ req.rc.sdata[4] == P9_RERROR:
    - VirtioChan::handle_rerror(req, in_hdr_len, offs, in_pages.as_deref()).
11. cleanup_pinned(in_pages, out_pages, need_drop).
12. if !kicked: p9_req_put(client, req).

`VirtioChan::get_mapped_pages(chan, pages, data, count, offs, need_drop) -> Result<usize>`:
1. if data.is_empty(): return Ok(0).
2. if !data.is_kvec():
   - /* User-iov: pin */
   - while vp_pinned.load() >= chan.p9_max_pages: io_wait_event_killable(VP_WQ, vp_pinned.load() < chan.p9_max_pages)?.
   - let n = iov_iter_get_pages_alloc2(data, pages, count, offs)?.
   - *need_drop = true.
   - vp_pinned.fetch_add(DIV_ROUND_UP(n + *offs, PAGE_SIZE)).
   - Ok(n).
3. else:
   - /* Kvec: resolve */.
   - let p = data.kvec.iov_base + data.iov_offset.
   - let len = data.single_seg_count().min(count).
   - let nr_pages = DIV_ROUND_UP(p + len, PAGE_SIZE) - p / PAGE_SIZE.
   - *pages = kmalloc(nr_pages * size_of::<*Page>()).
   - *need_drop = false; *offs = offset_in_page(p); p -= *offs.
   - for i in 0..nr_pages: (*pages)[i] = if is_vmalloc_addr(p) { vmalloc_to_page(p) } else { kmap_to_page(p) }; p += PAGE_SIZE.
   - data.iov_iter_advance(len).
   - Ok(len).

`VirtioChan::req_done(vq)` [vq IRQ callback]:
1. let chan = vq.vdev.priv as *mut VirtioChan.
2. let mut need_wakeup = false.
3. spin_lock_irqsave(&chan.lock, flags).
4. while let Some((req, len)) = virtqueue_get_buf(chan.vq):
   - if chan.ring_bufs_avail == 0: chan.ring_bufs_avail = 1; need_wakeup = true.
   - if len > 0: req.rc.size = len; p9_client_cb(chan.client, req, REQ_STATUS_RCVD).
5. spin_unlock_irqrestore.
6. if need_wakeup: wake_up(chan.vc_wq).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `chan_inuse_serialized` | INVARIANT | per-create/close: chan.inuse flips only under VIRTIO_9P_LOCK. |
| `ring_bufs_avail_monotone_recovery` | INVARIANT | per-ENOSPC: ring_bufs_avail = 0 ⟹ wakeup arrives via req_done ⟹ retry succeeds. |
| `vp_pinned_balanced` | INVARIANT | per-zc_request: every successful atomic_add(nr_pages, vp_pinned) is matched by atomic_sub on completion or err_out. |
| `sg_within_VIRTQUEUE_NUM` | INVARIANT | pack_sg_list / pack_sg_list_p: index < VIRTQUEUE_NUM (BUG_ON). |
| `sgs_layout_bound` | INVARIANT | zc_request: out_sgs + in_sgs <= 4. |
| `req_done_holds_chan_lock` | INVARIANT | per-IRQ: virtqueue_get_buf called under chan.lock. |
| `req_put_on_unkicked` | INVARIANT | zc_request: kicked == false ⟹ p9_req_put(client, req) at err_out. |
| `mount_tag_null_terminated` | INVARIANT | probe: tag buffer is kzalloc(tag_len + 1); last byte == 0. |

### Layer 2: TLA+

`net/9p/trans-virtio.tla`:
- Per-channel lifecycle: Probed → Listed → InUse → Closed → Removed.
- Per-request lifecycle: Submitted → InVQ → Completed → Delivered.
- Properties:
  - `safety_inuse_exclusion` — per-tag: at most one client.inuse == true.
  - `safety_no_use_after_remove` — per-chan: remove waits for inuse==false before del_vqs.
  - `safety_pin_quota_respected` — global: vp_pinned <= sum over chans of p9_max_pages (loose bound).
  - `liveness_request_completes_or_errors` — per-req: REQ_STATUS_SENT eventually reaches REQ_STATUS_RCVD ∨ REQ_STATUS_ERROR.
  - `liveness_ENOSPC_unblocks` — per-ENOSPC waiter: req_done eventually wakes vc_wq.
  - `liveness_pin_waiter_unblocks` — per-pin-quota waiter: vp_pinned eventually drops below ceiling.
  - `safety_rerror_string_copyback` — per-zc-reply: rc.sdata[4]==P9_RERROR ⟹ handle_rerror invoked before req returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VirtioChan::probe` post: chan in CHAN_LIST ∨ chan freed | `VirtioChan::probe` |
| `VirtioChan::create` post: ret == 0 ⟹ chan.inuse == true ∧ client.trans == chan | `VirtioChan::create` |
| `VirtioChan::close` post: chan.inuse == false | `VirtioChan::close` |
| `VirtioChan::request` post: ret == 0 ⟹ virtqueue_kick fired | `VirtioChan::request` |
| `VirtioChan::zc_request` post: err_out path runs unpin + maybe p9_req_put | `VirtioChan::zc_request` |
| `VirtioChan::req_done` post: every completed req delivered exactly once | `VirtioChan::req_done` |
| `pack_sg_list` post: returned count == (#sg entries written) | `pack_sg_list` |
| `pack_sg_list_p` post: BUG_ON(nr_pages > limit - start) honored | `pack_sg_list_p` |
| `get_mapped_pages` post: !is_kvec branch increments vp_pinned by exactly nr_pages | `get_mapped_pages` |

### Layer 4: Verus/Creusot functional

`Probe → list → mount (create) → request/zc_request → req_done → p9_client_cb → close → remove` semantic equivalence: per `Documentation/filesystems/9p.rst`, virtio-9p spec (`docs/specs/virtio-9p`), QEMU `hw/9pfs/9p-virtio.c` host-side handshake.

## Hardening

(Inherits row-1 features from `net/9p/00-overview.md` § Hardening.)

virtio-9p reinforcement:

- **Per-VIRTIO_9P_MOUNT_TAG mandatory** — defense against per-untagged-device confusion.
- **Per-chan->inuse exclusive mount** — defense against per-concurrent-mount race on the same channel.
- **Per-VIRTIO_9P_LOCK mutex around chan_list** — defense against per-iterator UAF during probe/remove.
- **Per-chan->lock spinlock IRQ-safe** — defense against per-req_done vs per-request contention on vq.
- **Per-vp_pinned global ceiling (nr_free_buffer_pages/4)** — defense against per-DoS via unbounded user-page pinning.
- **Per-VIRTQUEUE_NUM-3 maxsize budget** — defense against per-sg overflow (BUG_ON safety).
- **Per-supports_vmalloc = false** — defense against per-non-contiguous T-buffer crashing DMA.
- **Per-p9_virtio_remove inuse drain (10s warning)** — defense against per-mid-IO device hotplug-removal.
- **Per-virtio_reset_device + del_vqs in remove** — defense against per-late-IRQ on torn-down device.
- **Per-handle_rerror truncate at P9_ZC_HDR_SZ** — defense against per-malicious-host overflowing static R-buffer.
- **Per-req->wq killable wait** — defense against per-uninterruptible-mount hang.
- **Per-p9_virtio_cancel returns "not cancellable"** — defense against per-spurious-free of in-flight descriptor.
- **Per-kicked flag for zc_request err_out** — defense against per-double-completion or per-leak on early failure.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — virtio-9p zero-copy path uses iov_iter_get_pages with bounded msize; user pages are pinned into sg, never directly copied via a usercopy whitelist outside the standard VFS path.
- **PAX_KERNEXEC** — virtio_9p driver entry (p9_virtio_probe/remove, req_done, p9_virtio_request) executes from RX text; virtqueue callbacks point at typed static functions.
- **PAX_RANDKSTACK** — mount-time and remove-time entries honor randomized stack offset; the in-kernel mount-tag scan uses heap buffers.
- **PAX_REFCOUNT** — chan->inuse, vp_pinned (global page count), and req refcounts use hardened refcount types; pin/unpin saturate before turning a leak into a wraparound DoS.
- **PAX_MEMORY_SANITIZE** — freed virtio_chan, freed sg buffers, and freed p9_req payloads are sanitized so host-supplied bytes cannot leak across requests.
- **PAX_UDEREF** — user pages are reached only via iov_iter / pin_user_pages helpers with strict user/kernel split; no raw user pointer reaches the virtqueue layer.
- **PAX_RAP / kCFI** — p9_trans_module ops for virtio (.create, .close, .request, .zc_request, .cancel, .cancelled, .pooled_rbuffers) and virtio_driver ops are CFI-typed; vq->callback dispatch is non-pivotable.
- **GRKERNSEC_HIDESYM** — debugfs / sysfs surfaces for virtio_9p hide chan and vq addresses from non-CAP_SYSLOG readers.
- **GRKERNSEC_DMESG** — inuse-drain timeout warnings, supports_vmalloc=false rejections, and handle_rerror truncation logs gate behind dmesg_restrict.
- **Per-VIRTIO_9P_MOUNT_TAG mandatory** — defense against per-untagged-device confusion.
- **Per-vp_pinned global ceiling (nr_free_buffer_pages/4)** — defense against per-DoS via unbounded user-page pinning.
- **Per-VIRTQUEUE_NUM-3 maxsize budget** — defense against per-sg overflow.
- **Per-supports_vmalloc = false enforced** — defense against per-non-contiguous T-buffer corrupting DMA.
- **Per-virtio_reset_device + del_vqs strict pairing on remove** — defense against per-late-IRQ on torn-down device.

Rationale: virtio-9p binds untrusted host bytes to a pinned-user-page DMA path, so its grsec posture must protect both the user-page pin lifetime (REFCOUNT + global vp_pinned ceiling), the virtqueue dispatch (RAP/kCFI on trans_module + virtio_driver ops), and the freed-request slab path (MEMORY_SANITIZE). HIDESYM/DMESG ensure that probe/remove diagnostics do not turn into a host-visible kernel-address oracle, and the supports_vmalloc=false invariant keeps DMA descriptors physically contiguous as the design requires.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `net/9p/client.c` core RPC/tag/fid machinery (covered in `client.md` Tier-3).
- `net/9p/protocol.c` 9P2000.L marshalling (covered in `p9.md` Tier-3).
- `net/9p/trans_fd.c` TCP/Unix transport (covered in `trans-fd.md` Tier-3, this batch).
- `net/9p/trans_xen.c` Xen-9pfs transport (separate Tier-3 if expanded).
- `net/9p/trans_rdma.c` / `trans_usbg.c` (separate Tier-3 if expanded).
- VirtIO core (queue mgmt, MMIO/PCI bus) — covered under `drivers/virtio/*` Tier-3.
- `fs/9p/*` VFS bindings — separate Tier-3.
- Implementation code.
