---
title: "Tier-3: drivers/media/v4l2-core/v4l2-dev.c — V4L2 device registration"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The V4L2 core registers each capture/output node (camera, VBI sliced/raw, radio tuner, software-defined radio, touchscreen, sub-device) as a character device under VIDEO_MAJOR (81). Per-node container `struct video_device` carries the file_operations, ioctl_ops, vfl_type/vfl_dir, v4l2_device backref, control handler, prio state, media_entity, cdev pointer, minor + per-class node number + per-v4l2_device stream index. Per-`__video_register_device(vdev, type, nr, owner)` allocates a minor in a fixed (CONFIG_VIDEO_FIXED_MINOR_RANGES) or floating range, populates a `cdev`, registers in sysfs under class `video4linux`, optionally wires media-controller entity + interface link, then atomically sets `V4L2_FL_REGISTERED`. Per-`v4l2_open` checks the flag under `videodev_lock`, takes a `get_device` reference, delegates to `vdev->fops->open(filp)`, and enforces `V4L2_FL_USES_V4L2_FH` (every driver MUST populate `struct v4l2_fh`). Per-`v4l2_release` mirror: delegate then `put_device`. Per node naming follows `video%d`, `vbi%d`, `radio%d`, `v4l-subdev%d`, `swradio%d`, `v4l-touch%d` driven by `enum vfl_devnode_type`. Critical for: stable /dev paths, hot-unplug safety, ioctl whitelist by capability, media-controller graph integration.

This Tier-3 covers `drivers/media/v4l2-core/v4l2-dev.c` (~1258 lines).

### Acceptance Criteria

- [ ] AC-1: register_chrdev_region claims [MKDEV(81,0)..MKDEV(81,256)) with name "video4linux".
- [ ] AC-2: class_register creates /sys/class/video4linux with index/name/dev_debug attrs.
- [ ] AC-3: __video_register_device errors -EINVAL if !release || !v4l2_dev || (!device_caps for non-subdev) || !fops->open || !fops->release.
- [ ] AC-4: Per fixed-minor-range build: VFL_TYPE_VIDEO ⊆ [0,64); RADIO ⊆ [64,128); VBI ⊆ [224,256); others ⊆ [128,192).
- [ ] AC-5: dev_set_name picks "video%d"/"vbi%d"/"radio%d"/"v4l-subdev%d"/"swradio%d"/"v4l-touch%d".
- [ ] AC-6: After successful register, V4L2_FL_REGISTERED set under videodev_lock; before that flag is set, /dev/video* is not openable.
- [ ] AC-7: video_unregister_device: V4L2_FL_REGISTERED cleared under videodev_lock; later open returns -ENODEV.
- [ ] AC-8: v4l2_open: WARN_ON+force-release when driver fops->open succeeds but does not set V4L2_FL_USES_V4L2_FH.
- [ ] AC-9: determine_valid_ioctls: VIDIOC_REQBUFS bit only set when V4L2_CAP_STREAMING ∧ ops->vidioc_reqbufs.
- [ ] AC-10: determine_valid_ioctls: driver pre-set bits in vdev->valid_ioctls are masked OFF (bitmap_andnot) — explicit override of valid → invalid.
- [ ] AC-11: video_register_media_controller skips when v4l2_dev->mdev==NULL or vfl_dir==VFL_DIR_M2M.
- [ ] AC-12: After unregister, v4l2_event_wake_all flushes EPOLLPRI waiters then device_unregister.
- [ ] AC-13: v4l2_release holds mdev->req_queue_mutex iff v4l2_device_supports_requests(v4l2_dev).
- [ ] AC-14: v4l2_prio_check returns -EBUSY when caller's priority < global max.
- [ ] AC-15: video_devdata returns NULL after slot reuse only if there is no concurrent open (mutex_lock-bracketed in v4l2_open).

### Architecture

```
struct VideoDevice {
    entity: MediaEntity,
    intf_devnode: Option<NonNull<MediaIntfDevnode>>,
    pipe: MediaPipeline,
    fops: &'static V4l2FileOperations,
    ctrl_handler: Option<NonNull<V4l2CtrlHandler>>,
    ioctl_ops: Option<&'static V4l2IoctlOps>,
    valid_ioctls: BitArray<BASE_VIDIOC_PRIVATE>,
    vfl_type: VflDevnodeType,         // Video / Vbi / Radio / Subdev / Sdr / Touch
    vfl_dir: VflDevnodeDirection,     // Rx / Tx / M2m
    device_caps: u32,                 // V4L2_CAP_*
    v4l2_dev: NonNull<V4l2Device>,
    dev_parent: Option<NonNull<Device>>,
    prio: Option<NonNull<V4l2PrioState>>,
    minor: i32,                       // -1 sentinel = never registered
    num: u32,
    index: u32,
    flags: AtomicU32,                 // V4L2_FL_*
    fh_list: List<V4l2Fh>,
    fh_lock: Spinlock,
    queue: Option<NonNull<Vb2Queue>>,
    release: fn(*mut VideoDevice),    // MANDATORY
    cdev: Option<NonNull<Cdev>>,
    dev: Device,
    name: [u8; 32],
    dev_debug: u16,
}
```

`VideoDev::register_inner(vdev, type, nr, warn_if_nr_in_use, owner) -> Result<()>`:
1. vdev.minor = -1.
2. /* Validation */
3. if !vdev.release ∨ !vdev.v4l2_dev ∨ (type != Subdev ∧ vdev.device_caps == 0) ∨ !vdev.fops ∨ !vdev.fops.open ∨ !vdev.fops.release: return -EINVAL.
4. /* Init fh tracking */
5. spinlock_init(&vdev.fh_lock); init_list_head(&vdev.fh_list).
6. /* Naming */
7. name_base = match type { Video => "video", Vbi => "vbi", Radio => "radio", Subdev => "v4l-subdev", Sdr => "swradio", Touch => "v4l-touch" }.
8. vdev.vfl_type = type; vdev.cdev = None.
9. if vdev.dev_parent.is_none(): vdev.dev_parent = vdev.v4l2_dev.dev.
10. if vdev.ctrl_handler.is_none(): vdev.ctrl_handler = vdev.v4l2_dev.ctrl_handler.
11. if vdev.prio.is_none(): vdev.prio = &vdev.v4l2_dev.prio.
12. /* Minor allocation */
13. mutex_lock(&VIDEODEV_LOCK).
14. (minor_offset, minor_cnt) = fixed_range_for(type) (cfg_video_fixed_minor_ranges) or (0, 256).
15. nr = devnode_find(vdev, nr.unwrap_or(0), minor_cnt). retry from 0 if none in [from..minor_cnt).
16. if nr == minor_cnt: mutex_unlock; return -ENFILE.
17. i = if FIXED_RANGES { nr } else { first free of video_devices[0..256] }.
18. vdev.minor = (i + minor_offset) as i32; vdev.num = nr.
19. if video_devices[vdev.minor].is_some(): mutex_unlock; return -ENFILE (paranoid).
20. devnode_set(vdev); vdev.index = get_index(vdev); video_devices[vdev.minor] = Some(vdev).
21. mutex_unlock.
22. /* Derive valid_ioctls */
23. if vdev.ioctl_ops.is_some(): VideoDev::derive_valid_ioctls(vdev).
24. /* cdev */
25. let cdev = cdev_alloc().ok_or(-ENOMEM)?;
26. cdev.ops = &V4L2_FOPS; cdev.owner = owner.
27. cdev_add(cdev, mkdev(VIDEO_MAJOR, vdev.minor), 1)?;
28. vdev.cdev = Some(cdev).
29. /* sysfs device */
30. vdev.dev.class = &VIDEO_CLASS; vdev.dev.devt = mkdev(VIDEO_MAJOR, vdev.minor); vdev.dev.parent = vdev.dev_parent; vdev.dev.release = VideoDev::dev_release.
31. dev_set_name(&vdev.dev, "{}{}", name_base, vdev.num).
32. v4l2_device_get(vdev.v4l2_dev).
33. mutex_lock(&VIDEODEV_LOCK).
34. device_register(&vdev.dev)?;
35. if nr != requested_nr ∧ warn_if_nr_in_use: pr_warn("requested {}{}, got {}", name_base, nr, vdev.node_name()).
36. VideoDev::register_mc(vdev).
37. set_bit(V4L2_FL_REGISTERED, &vdev.flags).
38. mutex_unlock.
39. Ok(()).

`VideoDev::unregister(vdev)`:
1. if !vdev ∨ !test_bit(V4L2_FL_REGISTERED, &vdev.flags): return.
2. mutex_lock(&VIDEODEV_LOCK); clear_bit(V4L2_FL_REGISTERED, &vdev.flags); mutex_unlock.
3. v4l2_event_wake_all(vdev).
4. device_unregister(&vdev.dev) /* triggers VideoDev::dev_release */.

`VideoDev::dev_release(cd)`:
1. vdev = container_of(cd, VideoDevice, dev).
2. mutex_lock(&VIDEODEV_LOCK).
3. WARN_ON(video_devices[vdev.minor] != vdev).
4. video_devices[vdev.minor] = None.
5. cdev_del(vdev.cdev); vdev.cdev = None.
6. devnode_clear(vdev).
7. mutex_unlock.
8. /* Media-controller teardown */
9. if cfg(media_controller) ∧ vdev.v4l2_dev.mdev.is_some() ∧ vdev.vfl_dir != M2m:
   - media_devnode_remove(vdev.intf_devnode).
   - if vdev.entity.function != Unknown: media_device_unregister_entity(&vdev.entity).
10. v4l2_dev = if vdev.v4l2_dev.release.is_some() { Some(vdev.v4l2_dev) } else { None }.
11. vdev.release(vdev).
12. if let Some(v) = v4l2_dev { v4l2_device_put(v) }.

`VideoDev::open(inode, filp) -> i32`:
1. mutex_lock(&VIDEODEV_LOCK).
2. vdev = video_devdata(filp).
3. if vdev.is_none() ∨ !video_is_registered(vdev): mutex_unlock; return -ENODEV.
4. video_get(vdev) /* get_device */.
5. mutex_unlock.
6. if !video_is_registered(vdev): ret = -ENODEV; goto done.
7. ret = vdev.fops.open(filp).
8. if ret == 0 ∧ !test_bit(V4L2_FL_USES_V4L2_FH, &vdev.flags):
   - WARN_ON; vdev.fops.release(filp); ret = -ENODEV.
9. done: if ret != 0: video_put(vdev).
10. return ret.

`VideoDev::release(inode, filp) -> i32`:
1. vdev = video_devdata(filp).
2. if v4l2_device_supports_requests(vdev.v4l2_dev): mutex_lock(&vdev.v4l2_dev.mdev.req_queue_mutex).
3. ret = vdev.fops.release(filp).
4. if requests: mutex_unlock.
5. video_put(vdev).
6. return ret.

`VideoDev::derive_valid_ioctls(vdev)`:
1. ops = vdev.ioctl_ops; local = BitArray::zero().
2. /* Always-valid (subject to op presence) */
3. set_if(ops.vidioc_querycap, VIDIOC_QUERYCAP).
4. set(VIDIOC_G_PRIORITY); set(VIDIOC_S_PRIORITY).
5. /* Control handler-aware */
6. if vdev.ctrl_handler.is_some() ∨ ops.vidioc_query_ext_ctrl.is_some(): set(_QUERYCTRL, _QUERY_EXT_CTRL).
7. if vdev.ctrl_handler.is_some() ∨ ops.vidioc_g_ext_ctrls.is_some(): set(_G_CTRL, _G_EXT_CTRLS).
8. if vdev.ctrl_handler.is_some() ∨ ops.vidioc_s_ext_ctrls.is_some(): set(_S_CTRL, _S_EXT_CTRLS).
9. if vdev.ctrl_handler.is_some() ∨ ops.vidioc_try_ext_ctrls.is_some(): set(_TRY_EXT_CTRLS).
10. if vdev.ctrl_handler.is_some() ∨ ops.vidioc_querymenu.is_some(): set(_QUERYMENU).
11. /* Per (vfl_type, vfl_dir, device_caps) */
12. is_vid = (type==Video) ∧ device_caps & vid_caps.
13. is_meta = (type==Video) ∧ device_caps & meta_caps.
14. is_rx = vfl_dir != Tx; is_tx = vfl_dir != Rx; has_streaming = caps & STREAMING; is_edid = caps & EDID; is_io_mc = caps & IO_MC.
15. (fmt/enum/try paths per is_vid/_meta/_vbi/_sdr/_tch as in REQ-9).
16. if has_streaming: set REQBUFS, QUERYBUF, QBUF, EXPBUF, DQBUF, CREATE_BUFS, PREPARE_BUF, STREAMON, STREAMOFF; if ops.vidioc_create_bufs: set REMOVE_BUFS.
17. (input/output / std / dv_timings / edid blocks per REQ-9).
18. set_if(ops.vidioc_log_status, VIDIOC_LOG_STATUS).
19. cfg(video_adv_debug): set _DBG_G_CHIP_INFO, _DBG_G_REGISTER, _DBG_S_REGISTER.
20. set(_DQEVENT) if vidioc_subscribe_event; set(_SUBSCRIBE_EVENT, _UNSUBSCRIBE_EVENT).
21. /* Driver-override: bitmap_andnot */
22. vdev.valid_ioctls = local & !vdev.valid_ioctls.

`VideoDev::register_mc(vdev) -> i32`:
1. cfg(media_controller) — else return 0.
2. if vdev.v4l2_dev.mdev.is_none() ∨ vdev.vfl_dir == M2m: return 0.
3. vdev.entity.obj_type = VideoDevice; vdev.entity.function = Unknown.
4. match vdev.vfl_type {
   - Video → intf=V4l_Video; entity.function = Io_V4l.
   - Vbi → intf=V4l_Vbi; entity.function = Io_Vbi.
   - Sdr → intf=V4l_Swradio; entity.function = Io_Swradio.
   - Touch → intf=V4l_Touch; entity.function = Io_V4l.
   - Radio → intf=V4l_Radio; entity.function = Unknown.
   - Subdev → intf=V4l_Subdev; (entity registered separately).
   - _ → return 0. }
5. if entity.function != Unknown:
   - entity.name = vdev.name; entity.info.dev.major = VIDEO_MAJOR; entity.info.dev.minor = vdev.minor.
   - media_device_register_entity(vdev.v4l2_dev.mdev, &vdev.entity)?;
6. vdev.intf_devnode = media_devnode_create(vdev.v4l2_dev.mdev, intf, 0, VIDEO_MAJOR, vdev.minor).ok_or(-ENOMEM)?;
7. if entity.function != Unknown: media_create_intf_link(&vdev.entity, &vdev.intf_devnode.intf, Enabled|Immutable).ok_or(-ENOMEM)?;
8. Ok(0).

`V4l2Prio::change(global, local, new) -> i32`:
1. if !prio_is_valid(new): return -EINVAL.
2. if *local == new: return 0.
3. atomic_inc(&global.prios[new]).
4. if prio_is_valid(*local): atomic_dec(&global.prios[*local]).
5. *local = new.
6. 0.

`V4l2Prio::max(global) -> u32`:
1. for p in [Record, Interactive, Background]: if atomic_read(&global.prios[p]) > 0: return p.
2. Unset.

`V4l2Prio::check(global, local) -> i32`:
1. return if local < V4l2Prio::max(global) { -EBUSY } else { 0 }.

### Out of Scope

- drivers/media/v4l2-core/v4l2-ioctl.c VIDIOC_* dispatch (covered in `v4l2-ioctl.md` Tier-3)
- drivers/media/v4l2-core/v4l2-subdev.c sub-device pad ops (covered in `v4l2-subdev.md` Tier-3 if expanded)
- drivers/media/v4l2-core/v4l2-fh.c file-handle management (covered in `v4l2-fh.md` if expanded)
- drivers/media/v4l2-core/v4l2-ctrls*.c control framework (covered in `v4l2-ctrls.md` if expanded)
- drivers/media/v4l2-core/v4l2-compat-ioctl32.c (covered separately)
- drivers/media/mc/* media-controller core (covered in `media-controller.md` Tier-3)
- drivers/media/common/videobuf2/* VB2 framework (covered in `videobuf2-core.md` Tier-3)
- Per-driver bridge code (uvcvideo, vivid, ...)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct video_device` | per-node container | `VideoDevice` |
| `struct v4l2_file_operations` | per-driver fops vtable | `V4l2FileOperations` |
| `struct v4l2_ioctl_ops` | per-driver ioctl vtable | `V4l2IoctlOps` |
| `enum vfl_devnode_type` | per-class enumeration | `VflDevnodeType` |
| `enum vfl_devnode_direction` | RX/TX/M2M | `VflDevnodeDirection` |
| `video_device_alloc` | kzalloc(video_device) | `VideoDev::alloc` |
| `video_device_release` | kfree | `VideoDev::release` |
| `video_device_release_empty` | static-storage no-op | `VideoDev::release_empty` |
| `__video_register_device` | per-register | `VideoDev::register_inner` |
| `video_unregister_device` | per-unregister | `VideoDev::unregister` |
| `video_devdata` | per-file → vdev | `VideoDev::from_file` |
| `video_is_registered` | per-flag test | `VideoDev::is_registered` |
| `v4l2_device_release` | per-release cb | `VideoDev::dev_release` |
| `determine_valid_ioctls` | per-vtable scan | `VideoDev::derive_valid_ioctls` |
| `video_register_media_controller` | per-MC binding | `VideoDev::register_mc` |
| `v4l2_open` / `v4l2_release` | per-open / per-release shims | `VideoDev::open` / `release` |
| `v4l2_read` / `v4l2_write` | per-rw shims | `VideoDev::read` / `write` |
| `v4l2_ioctl` | per-ioctl shim | `VideoDev::ioctl` |
| `v4l2_poll` | per-poll shim | `VideoDev::poll` |
| `v4l2_mmap` | per-mmap shim | `VideoDev::mmap` |
| `v4l2_prio_init` / `_open` / `_close` / `_change` / `_check` / `_max` | per-priority FSM | `V4l2Prio::*` |
| `videodev_init` / `videodev_exit` | subsys_initcall | `Videodev::init` / `exit` |
| `video_devices[VIDEO_NUM_DEVICES]` | per-minor table | `VIDEO_DEVICES` |
| `videodev_lock` | per-subsystem mutex | `VIDEODEV_LOCK` |
| `devnode_nums[VFL_TYPE_MAX]` | per-type bitmap | `DEVNODE_NUMS` |
| `video_class` | sysfs class | `VIDEO_CLASS` |
| `V4L2_FL_REGISTERED` | flag bit | shared |
| `V4L2_FL_USES_V4L2_FH` | flag bit | shared |

### compatibility contract

REQ-1: struct video_device essential fields:
- entity / intf_devnode / pipe (media-controller).
- fops: const *v4l2_file_operations.
- ctrl_handler: per-vdev v4l2_ctrl_handler.
- ioctl_ops: const *v4l2_ioctl_ops.
- valid_ioctls[]: bitmap; derived by `determine_valid_ioctls` + driver overrides.
- vfl_type: VFL_TYPE_VIDEO / VBI / RADIO / SUBDEV / SDR / TOUCH.
- vfl_dir: VFL_DIR_RX / VFL_DIR_TX / VFL_DIR_M2M.
- device_caps: V4L2_CAP_* superset.
- v4l2_dev: backref to `struct v4l2_device`.
- dev_parent: optional parent device.
- prio: per-vdev `v4l2_prio_state` pointer (defaults to v4l2_dev->prio).
- minor, num, index: minor (0..255), per-class node number, per-v4l2_device stream index.
- flags: bitmask (V4L2_FL_REGISTERED, V4L2_FL_USES_V4L2_FH, V4L2_FL_QUIRK_INVERTED_CROP, ...).
- fh_list + fh_lock: list of open file handles.
- queue: optional struct vb2_queue back-pointer.
- release: mandatory free-callback (video_device_release / _empty / driver-private).
- cdev: per-cdev allocation.
- dev: struct device embedded.
- dev_debug: per-vdev debug flags (read/write via sysfs dev_debug).

REQ-2: `__video_register_device(vdev, type, nr, warn_if_nr_in_use, owner)`:
- vdev->minor = -1 (sentinel for "never registered").
- WARN+EINVAL: !vdev->release, !vdev->v4l2_dev, (type!=SUBDEV ∧ !device_caps), !fops || !fops->open || !fops->release.
- spin_lock_init(&vdev->fh_lock); INIT_LIST_HEAD(&vdev->fh_list).
- name_base ∈ {"video","vbi","radio","v4l-subdev","swradio","v4l-touch"} by type.
- vdev->vfl_type = type; vdev->cdev = NULL.
- Inherit dev_parent / ctrl_handler / prio from v4l2_dev if NULL.
- CONFIG_VIDEO_FIXED_MINOR_RANGES on: VIDEO→[0,64), RADIO→[64,128), VBI→[224,256), other→[128,192).
- mutex_lock(&videodev_lock); allocate per-type devnode bitmap slot; if all taken: -ENFILE.
- vdev->minor = i + minor_offset; vdev->num = nr.
- video_devices[vdev->minor] = vdev; devnode_set(vdev); vdev->index = get_index(vdev).
- mutex_unlock; determine_valid_ioctls(vdev) if vdev->ioctl_ops.
- cdev = cdev_alloc(); cdev->ops = &v4l2_fops; cdev->owner = owner; cdev_add(cdev, MKDEV(VIDEO_MAJOR, minor), 1).
- dev.class = video_class; dev.devt = MKDEV(VIDEO_MAJOR, minor); dev.parent = dev_parent; dev.release = v4l2_device_release; dev_set_name("<name_base>%d", num).
- v4l2_device_get(v4l2_dev) (per-refcount).
- mutex_lock(&videodev_lock); device_register(&vdev->dev); video_register_media_controller(vdev); set_bit(V4L2_FL_REGISTERED, &vdev->flags); mutex_unlock.
- On any failure pre-set_bit: cdev_del; clear devnode bitmap; reset minor=-1.

REQ-3: video_unregister_device(vdev):
- if !vdev || !video_is_registered(vdev): return.
- mutex_lock(&videodev_lock); clear_bit(V4L2_FL_REGISTERED, &vdev->flags); mutex_unlock.
- v4l2_event_wake_all(vdev) (drain pollers).
- device_unregister(&vdev->dev) (drops last device ref → v4l2_device_release).

REQ-4: v4l2_device_release(cd) (per-device .release):
- vdev = to_video_device(cd); v4l2_dev = vdev->v4l2_dev.
- mutex_lock(&videodev_lock).
- WARN_ON(video_devices[vdev->minor] != vdev).
- video_devices[vdev->minor] = NULL.
- cdev_del(vdev->cdev); vdev->cdev = NULL.
- devnode_clear(vdev).
- mutex_unlock.
- CONFIG_MEDIA_CONTROLLER: if v4l2_dev->mdev ∧ vfl_dir != M2M: media_devnode_remove(intf_devnode); media_device_unregister_entity if function != UNKNOWN.
- If v4l2_dev->release == NULL: do not v4l2_device_put.
- vdev->release(vdev) (driver/static/kfree).
- v4l2_device_put(v4l2_dev) if release-cb was set.

REQ-5: v4l2_fops (top-level fops):
- .owner = THIS_MODULE.
- .read = v4l2_read; .write = v4l2_write; .open = v4l2_open; .release = v4l2_release.
- .mmap = v4l2_mmap; .poll = v4l2_poll; .unlocked_ioctl = v4l2_ioctl.
- .get_unmapped_area = v4l2_get_unmapped_area (NOMMU only).
- .compat_ioctl = v4l2_compat_ioctl32 (CONFIG_COMPAT).

REQ-6: v4l2_open(inode, filp):
- mutex_lock(&videodev_lock).
- vdev = video_devdata(filp).
- if !vdev ∨ !video_is_registered(vdev): unlock; return -ENODEV.
- video_get(vdev) (get_device(&vdev->dev)).
- mutex_unlock.
- Re-check registered (race-window after unlock).
- ret = vdev->fops->open(filp).
- if ret == 0: WARN_ON(!V4L2_FL_USES_V4L2_FH) ⇒ vdev->fops->release(filp); ret = -ENODEV.
- if ret: video_put(vdev).
- return ret.

REQ-7: v4l2_release(inode, filp):
- vdev = video_devdata(filp).
- if v4l2_device_supports_requests(v4l2_dev): mutex_lock(&v4l2_dev->mdev->req_queue_mutex).
- ret = vdev->fops->release(filp).
- if request-supported: mutex_unlock.
- video_put(vdev) (unconditional; release retval ignored for ref).

REQ-8: v4l2_read / v4l2_write / v4l2_poll / v4l2_mmap shims:
- vdev = video_devdata(filp).
- v4l2_read: !fops->read ⇒ -EINVAL; else video_is_registered ⇒ fops->read; else -ENODEV.
- v4l2_write: symmetric.
- v4l2_poll: not-registered ⇒ EPOLLERR|EPOLLHUP|EPOLLPRI; !fops->poll ⇒ DEFAULT_POLLMASK; else fops->poll.
- v4l2_mmap: !fops->mmap ⇒ -ENODEV; registered ⇒ fops->mmap; else -ENODEV.
- v4l2_ioctl: fops->unlocked_ioctl ∧ registered ⇒ fops->unlocked_ioctl; else -ENOTTY.

REQ-9: determine_valid_ioctls(vdev):
- Build local valid_ioctls[BASE_VIDIOC_PRIVATE] from `(vdev->vfl_type, vdev->vfl_dir, vdev->device_caps, vdev->ioctl_ops)`.
- Per-type ⊆: vid_caps ⊂ {CAP_VIDEO_CAPTURE, _CAPTURE_MPLANE, _OUTPUT, _OUTPUT_MPLANE, _M2M, _M2M_MPLANE}.
- Always-valid (subject to ops/handler): VIDIOC_QUERYCAP, _G_PRIORITY, _S_PRIORITY, _G_TUNER / _S_TUNER (rx∧!tch), _LOG_STATUS, _SUBSCRIBE_EVENT / _UNSUBSCRIBE_EVENT / _DQEVENT.
- CTRL ioctls (G/S/_QUERY / _MENU / _EXT_CTRLS / _TRY_EXT_CTRLS) if vdev->ctrl_handler OR ops->vidioc_*_ext_ctrls present.
- ENUM_FMT / G_FMT / S_FMT / TRY_FMT chosen by (is_vid + is_rx/tx, is_meta + is_rx/tx, is_vbi, is_tch, is_sdr).
- STREAMING-class (REQBUFS, QUERYBUF, QBUF, DQBUF, EXPBUF, CREATE_BUFS, PREPARE_BUF, STREAMON, STREAMOFF, REMOVE_BUFS) iff device_caps & V4L2_CAP_STREAMING.
- EDID-class iff device_caps & V4L2_CAP_EDID.
- ENUMSTD / S_STD / G_STD / QUERYSTD / DV_TIMINGS / INPUT/OUTPUT for is_vid ∨ is_vbi ∨ is_meta with per-direction split.
- Final: vdev->valid_ioctls = local_valid_ioctls & ~driver_override (bitmap_andnot).

REQ-10: video_register_media_controller(vdev):
- CONFIG_MEDIA_CONTROLLER guard.
- If !v4l2_dev->mdev ∨ vfl_dir == M2M: noop.
- entity.obj_type = MEDIA_ENTITY_TYPE_VIDEO_DEVICE; entity.function chosen per vfl_type:
  - VIDEO/TOUCH → MEDIA_ENT_F_IO_V4L; intf_type = MEDIA_INTF_T_V4L_VIDEO / _TOUCH.
  - VBI → _IO_VBI, _T_V4L_VBI.
  - SDR → _IO_SWRADIO, _T_V4L_SWRADIO.
  - RADIO → function UNKNOWN, _T_V4L_RADIO.
  - SUBDEV → _T_V4L_SUBDEV (entity created via v4l2_device_register_subdev).
- If function != UNKNOWN: entity.name = vdev->name; entity.info.dev.major = VIDEO_MAJOR; entity.info.dev.minor = vdev->minor; media_device_register_entity.
- intf_devnode = media_devnode_create(mdev, intf_type, 0, VIDEO_MAJOR, minor).
- media_create_intf_link(entity, &intf_devnode->intf, ENABLED|IMMUTABLE).

REQ-11: video_devdata(file):
- Return video_devices[iminor(file_inode(file))].
- Lookup is RCU-safe-against-unregister via V4L2_FL_REGISTERED check at every fop entry.

REQ-12: Priority handling (v4l2_prio_state):
- Four priorities: V4L2_PRIORITY_UNSET (0), _BACKGROUND, _INTERACTIVE (default), _RECORD.
- struct v4l2_prio_state holds atomic_t prios[4].
- v4l2_prio_open: change local from UNSET to DEFAULT.
- v4l2_prio_change: validate prio_is_valid; atomic_inc new; atomic_dec old (if valid); store local.
- v4l2_prio_close: atomic_dec local if valid.
- v4l2_prio_max: highest non-zero counter from RECORD → INTERACTIVE → BACKGROUND → UNSET.
- v4l2_prio_check: -EBUSY if local < max else 0.

REQ-13: sysfs (video_class):
- class name = "video4linux".
- Per-vdev attributes: index (RO), dev_debug (RW), name (RO).
- ATTRIBUTE_GROUPS(video_device) bound at class create.

REQ-14: V4L2_FL_REGISTERED / V4L2_FL_USES_V4L2_FH semantics:
- V4L2_FL_REGISTERED: set under videodev_lock at end of __video_register_device; cleared under same lock at unregister.
- All fop shims gate work on `video_is_registered(vdev)` (test_bit on flags).
- V4L2_FL_USES_V4L2_FH: WARN+force-close in v4l2_open if not set after fops->open returned 0 (every modern driver must populate struct v4l2_fh).

REQ-15: Subsystem init (subsys_initcall):
- videodev_init(): register_chrdev_region(MKDEV(VIDEO_MAJOR,0), 256, "video4linux"); class_register(&video_class).
- videodev_exit(): class_unregister; unregister_chrdev_region; debugfs_remove_recursive.

REQ-16: Debugfs (CONFIG_DEBUG_FS):
- v4l2_debugfs_root(): lazily create /sys/kernel/debug/v4l2/.

REQ-17: Media-pipeline helpers (CONFIG_MEDIA_CONTROLLER):
- video_device_pipeline_start / _stop / _alloc_start / __* variants: forwarders to media_pipeline_start over entity.pads[0]; require num_pads == 1.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `register_sets_flag_under_lock` | INVARIANT | per-register_inner: V4L2_FL_REGISTERED set ONLY after device_register success and inside videodev_lock crit-section. |
| `unregister_clears_flag_under_lock` | INVARIANT | per-unregister: V4L2_FL_REGISTERED cleared inside videodev_lock crit-section before device_unregister. |
| `open_get_then_unlock` | INVARIANT | per-open: video_get(&vdev.dev) executed under videodev_lock, then unlock; release ref-balanced on every error path. |
| `release_puts_unconditionally` | INVARIANT | per-release: video_put always invoked regardless of fops->release retval. |
| `subdev_skips_device_caps_check` | INVARIANT | per-register_inner: type==Subdev exempted from "!device_caps ⇒ -EINVAL". |
| `valid_ioctls_andnot` | INVARIANT | per-derive_valid_ioctls: final valid_ioctls = local & !driver_override. |
| `minor_uniqueness` | INVARIANT | per-register_inner: video_devices[minor] is None before assignment inside videodev_lock. |
| `prio_atomic_consistency` | INVARIANT | per-V4l2Prio::change: increments new before decrement old; never both zero with non-Unset local. |
| `uses_v4l2_fh_required` | INVARIANT | per-open: if V4L2_FL_USES_V4L2_FH not set after successful fops->open, force-release + -ENODEV. |
| `mdev_m2m_skip` | INVARIANT | per-register_mc: vfl_dir==M2m or mdev==NULL ⇒ no media-controller binding. |

### Layer 2: TLA+

`drivers/media/v4l2-dev.tla`:
- States: vdev ∈ {Unallocated, Allocated, Registered, Unregistering, Released}; per-fop entry guarded by Registered.
- Properties:
  - `safety_open_only_when_registered` — per-open: vdev.state == Registered.
  - `safety_minor_unique` — per-minor slot ≤ 1 vdev simultaneously.
  - `safety_unregister_drains_pollers` — per-unregister: v4l2_event_wake_all before device_unregister.
  - `liveness_open_terminates` — per-open: terminates with success or -ENODEV (no spurious -EAGAIN).
  - `liveness_unregister_completes` — per-unregister: eventually reaches Released after last fd closed.
  - `safety_class_register_before_first_register` — per-subsys: class_register precedes any __video_register_device.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VideoDev::register_inner` post: Ok ⟹ V4L2_FL_REGISTERED ∧ video_devices[minor] == Some(self) ∧ cdev attached | `VideoDev::register_inner` |
| `VideoDev::register_inner` post: Err ⟹ minor == -1 ∧ video_devices[*] unaltered | `VideoDev::register_inner` |
| `VideoDev::unregister` post: V4L2_FL_REGISTERED cleared ∧ device_unregister called | `VideoDev::unregister` |
| `VideoDev::dev_release` post: video_devices[minor] == None ∧ cdev freed ∧ devnode bit cleared | `VideoDev::dev_release` |
| `VideoDev::open` post: ret==0 ⟹ V4L2_FL_USES_V4L2_FH set ∧ get_device taken | `VideoDev::open` |
| `VideoDev::derive_valid_ioctls` post: ∀ bit: REQBUFS-class bit set ⟺ (caps & STREAMING ∧ ops.vidioc_reqbufs ∧ !driver_override) | `VideoDev::derive_valid_ioctls` |
| `V4l2Prio::max` post: returns highest p with atomic_read(prios[p]) > 0 | `V4l2Prio::max` |

### Layer 4: Verus/Creusot functional

`Per-/dev/videoN lifecycle: subsys_initcall → register_chrdev_region → class_register → driver allocates vdev → __video_register_device → V4L2_FL_REGISTERED → open/ioctl/mmap/poll → video_unregister_device → V4L2_FL_REGISTERED cleared → v4l2_event_wake_all → device_unregister → v4l2_device_release → release_cb → v4l2_device_put` semantic equivalence: per-Documentation/driver-api/media/v4l2-dev.rst and userspace ABI of /dev/videoN.

### hardening

(Inherits row-1 features from `drivers/media/00-overview.md` § Hardening.)

V4L2-dev reinforcement:

- **Per-V4L2_FL_REGISTERED gate at every fop entry** — defense against per-use-after-unregister.
- **Per-videodev_lock single-writer for video_devices[]** — defense against per-minor-race.
- **Per-V4L2_FL_USES_V4L2_FH WARN+force-close** — defense against per-driver-fh-leak.
- **Per-cdev_add error rollback (devnode bit clear, minor=-1)** — defense against per-half-registered.
- **Per-fixed-minor-range respected for legacy types** — defense against per-userspace-ABI-break.
- **Per-determine_valid_ioctls driver-override masking** — defense against per-spoofed-capability.
- **Per-v4l2_device_get/put refcount around vdev** — defense against per-parent-freed.
- **Per-req_queue_mutex held across release** — defense against per-request-cancel race.
- **Per-WARN_ON !release / !fops->open / !fops->release** — defense against per-NULL-deref.
- **Per-media_controller M2M skip** — defense against per-bogus-entity-registration.
- **Per-v4l2_event_wake_all pre-unregister** — defense against per-orphaned-poll-waiter.
- **Per-priority FSM bounded atomic_t counters** — defense against per-prio-underflow.
- **Per-class video4linux owned by THIS_MODULE** — defense against per-module-unload race.

