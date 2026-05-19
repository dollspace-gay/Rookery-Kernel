# Tier-3: drivers/gpu/drm/drm_drv.c — DRM driver core (top-level)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
upstream-paths:
  - drivers/gpu/drm/drm_drv.c (~1281 lines)
  - include/drm/drm_drv.h (struct drm_driver, DRIVER_* feature bits)
  - include/drm/drm_device.h (struct drm_device)
  - include/drm/drm_file.h (struct drm_minor)
  - drivers/gpu/drm/drm_internal.h
-->

## Summary

`drm_drv.c` is the **top-level lifecycle and registration layer** of the Direct Rendering Manager. Per-GPU driver: `struct drm_driver` (feature bits, `fops`, ioctl table, lifecycle callbacks). Per-device: `struct drm_device` (kref-managed; embeds `managed.resources` list, file lists, mode-config, anonymous inode for VRAM mmap address-space). Per-instance: zero or more `struct drm_minor` records — `primary` (`/dev/dri/card<N>`), `render` (`/dev/dri/renderD<N>`), `accel` (`/dev/accel/accelN`) — keyed by minor-number in `xarray drm_minors_xa`. Per-allocation path: drivers call `devm_drm_dev_alloc()` (recommended; embeds `drm_device` in a driver-private struct via `__devm_drm_dev_alloc(parent, driver, size, offset)`) — internal `drm_dev_init(dev, driver, parent)` runs `drm_minor_alloc` for PRIMARY (and optionally RENDER or ACCEL) and `drm_gem_init` if `DRIVER_GEM` is set. Per-publish path: `drm_dev_register(dev, flags)` calls `drm_minor_register` for each allocated minor (debugfs, sysfs `device_add`, `xa_store` so `drm_stub_open` lookup finds the minor) and `drm_modeset_register_all` if `DRIVER_MODESET`. Per-unpublish: `drm_dev_unregister` reverses; `drm_dev_unplug` adds an SRCU-coordinated `dev->unplugged = true` so hot-unplug-safe paths can use `drm_dev_enter`/`drm_dev_exit`. Per-fops: `drm_stub_fops` (.open = `drm_stub_open`) is the char-dev open hook on DRM_MAJOR — it routes to the driver's `fops` via `replace_fops` after `drm_minor_acquire` resolves the minor. IOCTL dispatch is delegated to `drm_ioctl` (lives in `drm_ioctl.c`, called via `driver.fops.unlocked_ioctl`). Critical for: every DRM-class char-dev (graphics card, render-only, compute-accel), hot-unplug correctness (USB DisplayLink / GPU-eject), in-kernel clients (fbcon, vt-console).

This Tier-3 covers `drivers/gpu/drm/drm_drv.c` (~1281 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct drm_driver` | per-driver descriptor | `DrmDriver` (trait + descriptor) |
| `struct drm_device` | per-instance state | `DrmDevice` |
| `struct drm_minor` | per-char-dev minor | `DrmMinor` |
| `enum drm_minor_type` | PRIMARY / RENDER / ACCEL | `DrmMinorType` |
| `drm_minors_xa` (xarray) | per-major lookup | `DRM_MINORS_XA` |
| `drm_minor_get_xa()` | per-type xarray lookup | `Drm::minor_xa_for_type` |
| `drm_minor_get_slot()` | per-type slot in drm_device | `Drm::minor_slot` |
| `drm_minor_alloc()` | per-minor numbering + sysfs alloc | `Drm::minor_alloc` |
| `drm_minor_register()` | per-publish | `Drm::minor_register` |
| `drm_minor_unregister()` | per-unpublish | `Drm::minor_unregister` |
| `drm_minor_acquire()` / `_release()` | per-open lookup | `Drm::minor_acquire` / `Drm::minor_release` |
| `drm_dev_init()` | per-construct | `Drm::dev_init` |
| `__drm_dev_alloc()` | per-alloc-Rust-callable | `Drm::dev_alloc` |
| `__devm_drm_dev_alloc()` | per-alloc-devres | `Drm::devm_dev_alloc` |
| `drm_dev_alloc()` | per-alloc-deprecated | `Drm::dev_alloc_deprecated` |
| `drm_dev_get()` / `_put()` | per-refcount | `DrmDevice::get` / `put` |
| `drm_dev_release()` | per-kref-release | `Drm::dev_release` |
| `drm_dev_register()` | per-publish | `Drm::dev_register` |
| `drm_dev_unregister()` | per-unpublish | `Drm::dev_unregister` |
| `drm_dev_unplug()` | per-hot-unplug | `Drm::dev_unplug` |
| `drm_dev_enter()` / `_exit()` | per-SRCU section | `Drm::dev_enter` / `dev_exit` |
| `drm_dev_set_dma_dev()` | per-DMA-device | `Drm::set_dma_dev` |
| `drm_dev_wedged_event()` | per-wedge uevent | `Drm::wedged_event` |
| `drm_put_dev()` (deprecated) | per-legacy-release | `Drm::put_dev_deprecated` |
| `drm_fs_inode_new()` / `_free()` | per-anon-inode for mmap | `Drm::fs_inode_new` / `fs_inode_free` |
| `create_compat_control_link()` / `remove_compat_control_link()` | per-controlD symlink | `Drm::compat_control_link` |
| `drm_stub_open()` / `drm_stub_fops` | per-major char-dev open dispatcher | `Drm::stub_open` |
| `drm_core_init()` / `drm_core_exit()` | per-module-load | `Drm::core_init` / `core_exit` |
| `drmm_cgroup_register_region()` | per-DMEM-cgroup region | `Drm::cgroup_register_region` |
| `drm_core_check_feature()` | per-feature-bit query | `DrmDevice::has_feature` |
| `DRIVER_GEM / _MODESET / _RENDER / _ATOMIC / _SYNCOBJ / _SYNCOBJ_TIMELINE / _COMPUTE_ACCEL / _GEM_GPUVA / _CURSOR_HOTSPOT` | per-feature bits | `DriverFeatures` (bitflags) |

## Compatibility contract

REQ-1: struct drm_minor:
- index: per-minor-number (allocated from `drm_minors_xa` / `accel_minors_xa`).
- type: DRM_MINOR_PRIMARY / DRM_MINOR_RENDER / DRM_MINOR_ACCEL.
- dev: per-`drm_device` back-pointer.
- kdev: per-sysfs device (allocated by `drm_sysfs_minor_alloc`).
- debugfs_root: per-`/sys/kernel/debug/dri/<index>` dentry.

REQ-2: drm_minor_get_xa(type):
- PRIMARY ∨ RENDER → `&drm_minors_xa` (shared, range 64×t..64×t+63 fast-path; 192..MINORBITS extended).
- ACCEL (if CONFIG_DRM_ACCEL) → `&accel_minors_xa` (range 0..ACCEL_MAX_MINORS).
- else → ERR_PTR(-EOPNOTSUPP).

REQ-3: DRM_MINOR_LIMIT(t):
- PRIMARY (t=0) → XA_LIMIT(0, 63).
- (legacy CONTROL t=1 reserved by historic kernel; gap 64-127.)
- RENDER (t=2) → XA_LIMIT(128, 191).
- ACCEL → XA_LIMIT(0, ACCEL_MAX_MINORS).
- After fast-path exhaustion (-EBUSY) for PRIMARY/RENDER: retry with DRM_EXTENDED_MINOR_LIMIT = XA_LIMIT(192, (1<<MINORBITS) - 1).

REQ-4: drm_minor_alloc(dev, type):
- minor = drmm_kzalloc(dev, sizeof(*minor), GFP_KERNEL).
- minor.type = type; minor.dev = dev.
- xa_alloc into drm_minor_get_xa(type) with DRM_MINOR_LIMIT(type); on -EBUSY (PRIMARY/RENDER) retry DRM_EXTENDED_MINOR_LIMIT.
- drmm_add_action_or_reset(dev, drm_minor_alloc_release, minor) → on dev release, put_device + xa_erase.
- minor.kdev = drm_sysfs_minor_alloc(minor).
- *drm_minor_get_slot(dev, type) = minor.

REQ-5: drm_minor_register(dev, type):
- minor = *drm_minor_get_slot(dev, type).
- if !minor: return 0 (driver does not expose this minor).
- if type != DRM_MINOR_ACCEL: drm_debugfs_register(minor, minor.index).
- device_add(minor.kdev).
- xa_store(drm_minor_get_xa(type), minor.index, minor, GFP_KERNEL) — replaces the NULL placeholder so `drm_stub_open` lookups succeed from here on.

REQ-6: drm_minor_unregister(dev, type):
- minor = slot.
- if !minor ∨ !device_is_registered(minor.kdev): return.
- xa_store(drm_minor_get_xa(type), minor.index, NULL, GFP_KERNEL) — lookups fail from here on.
- device_del(minor.kdev); dev_set_drvdata(minor.kdev, NULL); drm_debugfs_unregister(minor).

REQ-7: drm_minor_acquire(minor_xa, minor_id) → drm_minor *:
- xa_lock(minor_xa); minor = xa_load(minor_xa, minor_id); if minor: drm_dev_get(minor.dev); xa_unlock.
- if !minor: return ERR_PTR(-ENODEV).
- if drm_dev_is_unplugged(minor.dev): drm_dev_put(minor.dev); return ERR_PTR(-ENODEV).
- return minor.

REQ-8: drm_dev_init(dev, driver, parent):
- if !drm_core_init_complete: return -ENODEV.
- if !parent (WARN_ON): return -EINVAL.
- kref_init(&dev.ref); dev.dev = get_device(parent); dev.driver = driver.
- INIT_LIST_HEAD(&dev.managed.resources); spin_lock_init(&dev.managed.lock).
- dev.driver_features = ~0u (full).
- if driver_features include DRIVER_COMPUTE_ACCEL ∧ (DRIVER_RENDER ∨ DRIVER_MODESET): return -EINVAL.
- INIT_LIST_HEAD(filelist / filelist_internal / clientlist / client_sysrq_list / vblank_event_list).
- spin_lock_init(&dev.event_lock); mutex_init(&dev.filelist_mutex); mutex_init(&dev.clientlist_mutex); mutex_init(&dev.master_mutex).
- raw_spin_lock_init(&dev.mode_config.panic_lock).
- drmm_add_action_or_reset(dev, drm_dev_init_release, NULL).
- dev.anon_inode = drm_fs_inode_new() (anonymous inode rooted in pseudo-FS `drm`).
- if DRIVER_COMPUTE_ACCEL: drm_minor_alloc(DRM_MINOR_ACCEL).
- else: optionally drm_minor_alloc(DRM_MINOR_RENDER); drm_minor_alloc(DRM_MINOR_PRIMARY).
- if DRIVER_GEM: drm_gem_init(dev).
- dev.unique = drmm_kstrdup(dev_name(parent)).
- drm_debugfs_dev_init(dev).
- on err: drm_managed_release(dev); return ret.

REQ-9: __drm_dev_alloc(parent, driver, size, offset) → void *:
- container = kzalloc(size, GFP_KERNEL).
- drm = container + offset.
- drm_dev_init(drm, driver, parent).
- drmm_add_final_kfree(drm, container) — when last kref drops, kfree(container) fires.
- return container.
- /* Distinguished from `__devm_drm_dev_alloc` (also runs `devm_add_action_or_reset(parent, devm_drm_dev_init_release, dev)` so the parent's devres tears the device down). */

REQ-10: drm_dev_alloc(driver, parent):
- /* Deprecated; for non-subclass-embedding drivers. */
- return __drm_dev_alloc(parent, driver, sizeof(struct drm_device), 0).

REQ-11: drm_dev_get / drm_dev_put:
- get: if dev: kref_get(&dev.ref). Caller must already hold a reference.
- put: if dev: kref_put(&dev.ref, drm_dev_release).
- release: drm_debugfs_dev_fini; if driver.release: driver.release(dev); drm_managed_release(dev); kfree(dev.managed.final_kfree).

REQ-12: drm_dev_register(dev, flags):
- driver = dev.driver.
- if !driver.load: drm_mode_config_validate(dev).
- WARN_ON(!dev.managed.final_kfree).
- if drm_dev_needs_global_mutex(dev): mutex_lock(&drm_global_mutex).
- if DRIVER_COMPUTE_ACCEL: accel_debugfs_register(dev) else drm_debugfs_dev_register(dev).
- drm_minor_register(dev, DRM_MINOR_RENDER) — no-op if minor wasn't allocated.
- drm_minor_register(dev, DRM_MINOR_PRIMARY).
- drm_minor_register(dev, DRM_MINOR_ACCEL).
- create_compat_control_link(dev) — for DRIVER_MODESET, symlink `controlD<index+64>` → primary kdev (legacy userspace probe heuristic).
- dev.registered = true.
- if driver.load: driver.load(dev, flags). /* deprecated; race-prone */
- if DRIVER_MODESET: drm_modeset_register_all(dev).
- drm_panic_register(dev) (DRM-panic screen).
- drm_client_sysrq_register(dev) (SysRq-mediated client teardown).
- on err_unload: driver.unload(dev) if present.
- on err_minors: remove_compat_control_link + minor_unregister for all three types.
- mutex_unlock(&drm_global_mutex) if held.
- return 0.

REQ-13: drm_dev_unregister(dev):
- dev.registered = false.
- drm_client_sysrq_unregister; drm_panic_unregister; drm_client_dev_unregister.
- if DRIVER_MODESET: drm_modeset_unregister_all.
- if driver.unload: driver.unload(dev).
- remove_compat_control_link.
- drm_minor_unregister for ACCEL, PRIMARY, RENDER.
- drm_debugfs_dev_fini.

REQ-14: drm_dev_unplug(dev):
- dev.unplugged = true.
- synchronize_srcu(&drm_unplug_srcu) — wait for in-flight drm_dev_enter sections.
- drm_dev_unregister(dev).
- unmap_mapping_range(dev.anon_inode.i_mapping, 0, 0, 1) — drop all userspace VRAM mmaps.

REQ-15: drm_dev_enter(dev, &idx) → bool:
- *idx = srcu_read_lock(&drm_unplug_srcu).
- if dev.unplugged: srcu_read_unlock; return false.
- return true.

REQ-16: drm_dev_exit(idx):
- srcu_read_unlock(&drm_unplug_srcu, idx).

REQ-17: drm_put_dev (deprecated):
- drm_dev_unregister(dev); drm_dev_put(dev).

REQ-18: drm_dev_set_dma_dev(dev, dma_dev):
- dma_dev = get_device(dma_dev).
- put_device(dev.dma_dev); dev.dma_dev = dma_dev.

REQ-19: drm_dev_wedged_event(dev, method, info):
- envp[] = { "WEDGED=…" [, "PID=…", "TASK=…"] }.
- For each set bit in method: append drm_get_wedge_recovery(opt) ∈ { "none", "rebind", "bus-reset", "vendor-specific" }; unknown → "WEDGED=unknown".
- kobject_uevent_env(&dev.primary.kdev.kobj, KOBJ_CHANGE, envp).

REQ-20: drm_fs_inode_new:
- simple_pin_fs(&drm_fs_type, &drm_fs_mnt, &drm_fs_cnt).
- alloc_anon_inode(drm_fs_mnt.mnt_sb).
- drm_fs_type: pseudo filesystem (`init_pseudo`) named "drm", super magic 0x010203ff.

REQ-21: drm_stub_open(inode, filp):
- minor = drm_minor_acquire(&drm_minors_xa, iminor(inode)).
- new_fops = fops_get(minor.dev.driver.fops); if !new_fops: -ENODEV.
- replace_fops(filp, new_fops).
- if filp.f_op.open: err = filp.f_op.open(inode, filp); else err = 0.
- drm_minor_release(minor) — `drm_dev_put` (balances `drm_dev_get` in `drm_minor_acquire`).

REQ-22: drm_stub_fops:
- .owner = THIS_MODULE.
- .open = drm_stub_open.
- .llseek = noop_llseek.

REQ-23: drm_core_init (module_init):
- drm_connector_ida_init(); drm_memcpy_init_early().
- drm_sysfs_init() — `/sys/class/drm`.
- drm_debugfs_init_root() — `/sys/kernel/debug/dri`.
- drm_debugfs_bridge_params().
- register_chrdev(DRM_MAJOR, "drm", &drm_stub_fops).
- accel_core_init() (if CONFIG_DRM_ACCEL).
- drm_panic_init().
- drm_privacy_screen_lookup_init().
- drm_ras_genl_family_register().
- drm_core_init_complete = true.

REQ-24: drm_core_exit:
- drm_ras_genl_family_unregister; drm_privacy_screen_lookup_exit; drm_panic_exit; accel_core_exit; unregister_chrdev(DRM_MAJOR); drm_debugfs_remove_root; drm_sysfs_destroy; WARN_ON(!xa_empty(&drm_minors_xa)); drm_connector_ida_destroy.

REQ-25: DRIVER_* feature bits (driver_features = bitmask, default ~0u):
- DRIVER_GEM (BIT 0): driver uses GEM buffer-object manager.
- DRIVER_MODESET (BIT 1): driver supports KMS modesetting.
- DRIVER_RENDER (BIT 3): driver supports a render-only node (`renderD<N>`).
- DRIVER_ATOMIC (BIT 4): driver implements atomic modesetting.
- DRIVER_SYNCOBJ (BIT 5): driver supports drm_syncobj.
- DRIVER_SYNCOBJ_TIMELINE (BIT 6): driver supports timeline syncobjs.
- DRIVER_COMPUTE_ACCEL (BIT 7): compute-acceleration class (mutually exclusive with RENDER ∧ MODESET).
- DRIVER_GEM_GPUVA (BIT 8): GEM with GPU-VA manager.
- DRIVER_CURSOR_HOTSPOT (BIT 9): driver reports KMS cursor hotspot in atomic plane state.

REQ-26: Client init / in-kernel users:
- `drm_client_dev_register` (lives in drm_client.c) is reached transitively from external users; `drm_drv.c` simply unregisters via `drm_client_dev_unregister` in `drm_dev_unregister`.
- `drm_client_sysrq_register` / `_unregister` wire the device into SysRq-based client teardown.

REQ-27: IOCTL dispatch:
- The driver's `.fops.unlocked_ioctl` is wired by the driver to `drm_ioctl` (drm_ioctl.c).
- drm_drv.c is **not** the dispatcher itself — it is the registration plumbing that publishes the cdev whose `f_op.open` is `drm_stub_open`, which then `replace_fops` to the driver's fops table. From that point all IOCTL traffic goes through the driver's fops (typically `drm_ioctl` from `drm_ioctl.c`).

## Acceptance Criteria

- [ ] AC-1: Load a DRM driver (e.g. virtio-gpu): `drm_dev_register` succeeds; `/dev/dri/card0` appears.
- [ ] AC-2: If driver.driver_features has DRIVER_RENDER: `/dev/dri/renderD128` (minor 128) also appears.
- [ ] AC-3: If driver.driver_features has DRIVER_COMPUTE_ACCEL: `/dev/accel/accel0` appears; PRIMARY / RENDER minors are not allocated.
- [ ] AC-4: Mutual-exclusion check: driver with DRIVER_COMPUTE_ACCEL | DRIVER_MODESET fails `drm_dev_init` with -EINVAL.
- [ ] AC-5: `open("/dev/dri/card0")`: `drm_stub_open` resolves the minor, `replace_fops` switches the file to the driver's fops; driver's `.open` runs once.
- [ ] AC-6: Hot-unplug: `drm_dev_unplug` flips `dev.unplugged`; subsequent `drm_dev_enter` returns false; userspace mmap regions are unmapped (`unmap_mapping_range`).
- [ ] AC-7: `drm_dev_get` after `drm_dev_unregister` but before `drm_dev_put`: minor lookup via `drm_minor_acquire` returns -ENODEV (entry replaced with NULL by `drm_minor_unregister`).
- [ ] AC-8: PRIMARY minor exhaustion in 0..63: fallback xa_alloc in 192..MINORBITS succeeds; userspace gets a card65 etc.
- [ ] AC-9: `drm_dev_wedged_event(dev, DRM_WEDGE_RECOVERY_BUS_RESET | DRM_WEDGE_RECOVERY_REBIND, NULL)`: emits uevent `WEDGED=rebind,bus-reset` on primary minor's kobj.
- [ ] AC-10: DRIVER_MODESET driver: `controlD<index+64>` symlink under `/sys/class/drm/` points to primary kdev; removed by `drm_dev_unregister`.
- [ ] AC-11: `drm_core_init` failure path: `drm_core_exit` cleans up partially-initialized core; `drm_core_init_complete` stays false; subsequent `drm_dev_init` returns -ENODEV.
- [ ] AC-12: `drm_put_dev(NULL)`: logs error, returns; no crash.
- [ ] AC-13: Final `drm_dev_put`: `drm_dev_release` calls `driver.release` if non-NULL; `drm_managed_release` runs all `drmm_add_action` callbacks in LIFO order; `kfree(dev->managed.final_kfree)` releases the container.

## Architecture

```
struct DrmDevice {
  ref: KRef,
  dev: *Device,                              // parent (PCI / platform)
  dma_dev: Option<*Device>,
  driver: *DrmDriver,
  driver_features: u32,                      // bitfield of DRIVER_*
  primary: Option<*DrmMinor>,
  render: Option<*DrmMinor>,
  accel: Option<*DrmMinor>,
  anon_inode: *Inode,                        // for mmap address_space
  unique: *str,                              // dev_name(parent) copy
  managed: DrmManaged {
    resources: ListHead,                     // LIFO of drmm_action callbacks
    lock: SpinLock<()>,
    final_kfree: Option<*c_void>,
  },
  filelist: ListHead,
  filelist_internal: ListHead,
  clientlist: ListHead,
  client_sysrq_list: ListHead,
  vblank_event_list: ListHead,
  event_lock: SpinLock<()>,
  filelist_mutex: Mutex<()>,
  clientlist_mutex: Mutex<()>,
  master_mutex: Mutex<()>,
  mode_config: DrmModeConfig { panic_lock: RawSpinLock<()>, ... },
  registered: bool,
  unplugged: bool,
}

struct DrmMinor {
  index: u32,
  type: DrmMinorType,                        // Primary / Render / Accel
  dev: *DrmDevice,
  kdev: *Device,                             // sysfs device
  debugfs_root: Option<*Dentry>,
}
```

`Drm::dev_init(dev, driver, parent) -> Result<(), i32>`:
1. Check drm_core_init_complete.
2. Validate non-NULL parent.
3. kref_init; dev.dev = get_device(parent); dev.driver = driver.
4. Init managed list + lock; driver_features = ~0u.
5. Reject COMPUTE_ACCEL combined with RENDER / MODESET.
6. Init filelist, clientlist, vblank_event_list; init mutexes and event_lock; init mode_config.panic_lock.
7. Register drm_dev_init_release via drmm_add_action_or_reset.
8. Allocate anon_inode via drm_fs_inode_new.
9. If COMPUTE_ACCEL: minor_alloc(ACCEL). Else: optional minor_alloc(RENDER); minor_alloc(PRIMARY).
10. If DRIVER_GEM: drm_gem_init.
11. dev.unique = drmm_kstrdup(dev_name(parent)).
12. drm_debugfs_dev_init.

`Drm::dev_alloc(parent, driver, size, offset) -> *c_void`:
1. container = kzalloc(size).
2. drm = container + offset.
3. Drm::dev_init(drm, driver, parent).
4. drmm_add_final_kfree(drm, container).
5. return container.

`Drm::dev_register(dev, flags) -> Result<(), i32>`:
1. Optional drm_global_mutex if needs_global_mutex.
2. accel_debugfs_register or drm_debugfs_dev_register.
3. minor_register(RENDER), (PRIMARY), (ACCEL).
4. create_compat_control_link if DRIVER_MODESET.
5. dev.registered = true.
6. If driver.load (deprecated): driver.load(dev, flags).
7. If DRIVER_MODESET: drm_modeset_register_all.
8. drm_panic_register; drm_client_sysrq_register.
9. On err_unload / err_minors: unwind in reverse.

`Drm::dev_unregister(dev)`:
1. dev.registered = false.
2. drm_client_sysrq_unregister; drm_panic_unregister; drm_client_dev_unregister.
3. If DRIVER_MODESET: drm_modeset_unregister_all.
4. If driver.unload: driver.unload(dev).
5. remove_compat_control_link.
6. minor_unregister(ACCEL, PRIMARY, RENDER).
7. drm_debugfs_dev_fini.

`Drm::dev_unplug(dev)`:
1. dev.unplugged = true.
2. synchronize_srcu(&drm_unplug_srcu).
3. Drm::dev_unregister(dev).
4. unmap_mapping_range(dev.anon_inode.i_mapping, 0, 0, 1).

`Drm::dev_release(ref) (kref-release)`:
1. dev = container_of(ref, DrmDevice, ref).
2. drm_debugfs_dev_fini (safety-belt).
3. If driver.release: driver.release(dev).
4. drm_managed_release(dev) — LIFO unwind of drmm actions; releases minors, anon_inode, dma_dev, parent_dev refs.
5. kfree(dev.managed.final_kfree).

`Drm::stub_open(inode, filp) -> Result<(), i32>`:
1. minor = Drm::minor_acquire(&DRM_MINORS_XA, iminor(inode)) (holds drm_dev_get).
2. new_fops = fops_get(minor.dev.driver.fops); if NULL: -ENODEV.
3. replace_fops(filp, new_fops).
4. If filp.f_op.open: call it.
5. Drm::minor_release(minor) (drm_dev_put).

`Drm::minor_register(dev, type) -> Result<(), i32>`:
1. minor = drm_minor_get_slot(dev, type); if NULL: return 0.
2. If type != ACCEL: drm_debugfs_register(minor, minor.index).
3. device_add(minor.kdev).
4. xa_store(drm_minor_get_xa(type), minor.index, minor, GFP_KERNEL).

`Drm::minor_acquire(xa, id) -> *DrmMinor`:
1. xa_lock; minor = xa_load(xa, id); if minor: drm_dev_get(minor.dev); xa_unlock.
2. If !minor: ERR_PTR(-ENODEV).
3. If drm_dev_is_unplugged(minor.dev): drm_dev_put; ERR_PTR(-ENODEV).
4. return minor.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `kref_balanced_alloc_to_put` | INVARIANT | per-`Drm::dev_alloc` → drm_managed_release path: every kref_get matches a kref_put; final_kfree freed at last put. |
| `accel_excludes_render_modeset` | INVARIANT | per-`Drm::dev_init`: feature bits COMPUTE_ACCEL ∧ (RENDER ∨ MODESET) ⟹ -EINVAL. |
| `minor_xa_consistency` | INVARIANT | per-`Drm::minor_register`/`_unregister`: xa entry == minor (registered) or NULL (unregistered) — never stale pointer. |
| `unplug_srcu_synchronized_before_unregister` | INVARIANT | per-`Drm::dev_unplug`: synchronize_srcu precedes `Drm::dev_unregister`. |
| `core_init_complete_gate` | INVARIANT | per-`Drm::dev_init`: returns -ENODEV if `drm_core_init_complete == false`. |
| `parent_get_put_balanced` | INVARIANT | per-`Drm::dev_init` + `Drm::dev_init_release`: get_device(parent) matched by put_device(dev.dev). |
| `dma_dev_get_put_balanced` | INVARIANT | per-`Drm::set_dma_dev`: get_device + put_device balanced. |
| `stub_open_minor_release_on_error` | INVARIANT | per-`Drm::stub_open`: minor_release called whether or not driver-open succeeds (refcount drop guaranteed). |
| `feature_check_no_post_register_mutate` | INVARIANT | per-driver_features: not mutated after `Drm::dev_register` returns. |
| `register_unwind_on_failure` | INVARIANT | per-`Drm::dev_register` err paths: every successful step before failure is unwound in reverse. |

### Layer 2: TLA+

`drivers/gpu/drm/drm-drv.tla`:
- States per DrmDevice: Uninit, Initialized, Registered, Unregistered, Unplugged, Released.
- Per minor: Allocated, Stored (xa entry == minor), Cleared (xa entry == NULL).
- Per file: Closed, StubOpen, FopsReplaced, DriverOpen.
- Properties:
  - `safety_lookup_only_after_register` — per-minor: drm_minor_acquire returns non-error only when xa entry == minor (i.e. between drm_minor_register and drm_minor_unregister).
  - `safety_no_open_after_unplug` — per-dev: dev.unplugged ⟹ subsequent drm_minor_acquire returns -ENODEV.
  - `safety_release_only_at_refcount_zero` — per-dev: drm_dev_release fires exactly when kref drops to zero.
  - `safety_register_idempotent_within_unwind` — per-dev: failed drm_dev_register restores the pre-call state exactly.
  - `liveness_unplug_terminates` — per-`Drm::dev_unplug`: synchronize_srcu returns in bounded time after all `Drm::dev_enter` sections complete.
  - `liveness_open_eventually_routes_to_driver` — per-stub_open: replace_fops + driver.open executes within one syscall.
  - `safety_accel_exclusive_minor` — per-dev: if DRIVER_COMPUTE_ACCEL, only accel minor allocated; never primary / render.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Drm::dev_init` post (ret == 0): dev.ref == 1 ∧ dev.driver == driver ∧ dev.dev == parent ∧ dev.anon_inode != NULL | `Drm::dev_init` |
| `Drm::dev_register` post (ret == 0): dev.registered = true ∧ each allocated minor.index has xa entry == minor | `Drm::dev_register` |
| `Drm::dev_unregister` post: dev.registered = false ∧ each minor.index has xa entry == NULL | `Drm::dev_unregister` |
| `Drm::dev_unplug` post: dev.unplugged = true ∧ Drm::dev_enter returns false ∧ all in-flight enter sections drained | `Drm::dev_unplug` |
| `Drm::dev_release` post: managed.resources empty ∧ final_kfree freed ∧ parent put | `Drm::dev_release` |
| `Drm::minor_alloc` post (ret == 0): minor.index allocated within DRM_MINOR_LIMIT(type) ∨ DRM_EXTENDED_MINOR_LIMIT | `Drm::minor_alloc` |
| `Drm::minor_register` post: device_is_registered(minor.kdev) ∧ xa entry == minor | `Drm::minor_register` |
| `Drm::stub_open` post (ret == 0): filp.f_op == driver.fops ∧ filp may now serve driver IOCTLs | `Drm::stub_open` |
| `Drm::wedged_event` post: kobject_uevent_env returned ∧ envp[0] starts with "WEDGED=" | `Drm::wedged_event` |
| `Drm::core_init` post (ret == 0): drm_core_init_complete = true ∧ register_chrdev(DRM_MAJOR, "drm", &drm_stub_fops) succeeded | `Drm::core_init` |

### Layer 4: Verus/Creusot functional

`Per-load (drm_core_init) → drm_dev_alloc → drm_dev_init → drm_dev_register → minor_register (xa_store) → user-space open → drm_stub_open → replace_fops → driver-open → drm_dev_unregister (xa_store NULL) → drm_dev_put → drm_dev_release` semantic equivalence: per-Documentation/gpu/drm-internals.rst and Documentation/gpu/drm-uapi.rst. Per-unplug path: per-Documentation/gpu/drm-uapi.rst "Device Hot-Unplug". Per-wedged: per-Documentation/gpu/drm-uapi.rst "Device Wedging".

## Hardening

(Inherits row-1 features from `drivers/gpu/drm/00-overview.md` § Hardening.)

DRM driver-core reinforcement:

- **Per-`drm_core_init_complete` gate** — defense against per-driver-probe-before-DRM-core-ready partial-init.
- **Per-DRIVER_COMPUTE_ACCEL exclusivity check** — defense against per-policy violation (accel devices must not appear under `/dev/dri/`).
- **Per-`drm_unplug_srcu` SRCU barrier** — defense against per-use-after-unplug across HW MMIO reads (drivers wrap MMIO with `drm_dev_enter`/`exit`).
- **Per-minor xa NULL placeholder swap** — defense against per-race where userspace `open` hits a half-torn-down minor (xa returns NULL → -ENODEV).
- **Per-`drm_dev_is_unplugged` in `drm_minor_acquire`** — defense against per-stale-fd opening a removed device's driver fops.
- **Per-`drmm_add_action_or_reset` LIFO** — defense against per-leak when partial init fails; release fires in reverse alloc order.
- **Per-`final_kfree` distinct from `dev`** — defense against per-double-free when `drm_device` is embedded in a driver-private container.
- **Per-`anon_inode` per-device address_space** — defense against per-cross-device VRAM mmap collision; `unmap_mapping_range` on unplug evicts every mapping in one call.
- **Per-`get_device(parent)` / `put_device(dev.dev)`** — defense against per-parent-bus-device freed while DRM still references it.
- **Per-DMA-dev separate get/put** — defense against per-DMA-coherent-device disappearing while buffers still bound.
- **Per-`WARN_ON(!managed.final_kfree)` at register** — defense against per-incorrect-alloc-path that skipped final_kfree registration.
- **Per-`drm_global_mutex` for legacy `driver.load`** — defense against per-double-registration when deprecated path is used.
- **Per-`WARN_ON(!xa_empty)` at core_exit** — defense against per-orphan-minor module-unload bug.
- **Per-`device_is_registered` check before device_del** — defense against per-double-`device_del` on a never-published minor.

## Grsecurity/PaX-style Reinforcement

This driver inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on every DRM ioctl (`drm_ioctl` arg-len computed from per-ioctl table); GEM/PRIME ioctl buffer-handle arrays bounds-checked.
- **PAX_KERNEXEC** — W^X for GPU command-stream buffers and any driver-loaded GPU microcode/firmware blob; userspace-submitted IBs validated/parsed before HW fetch.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every `/dev/dri/card*` and `/dev/dri/renderD*` ioctl entry.
- **PAX_REFCOUNT** — saturating refcount on `drm_device`, `drm_minor`, `drm_file`, `drm_gem_object`, `drm_framebuffer`, `dma_fence`, per-driver private device.
- **PAX_MEMORY_SANITIZE** — zero-on-free for VRAM-eviction shmem pages, GEM scratch buffers, command-stream staging buffers, indirect-buffer (IB) shadow buffers.
- **PAX_UDEREF / PAX_MEMORY_UDEREF** — strict user-pointer access on GEM `userptr` mappings; pin-page accounting via `mmu_notifier`; userptr buffers never blindly DMA'd.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `drm_driver`, `drm_ioctl_desc`, `file_operations` (drm_fops), `dma_fence_ops`, `drm_gem_object_funcs`, `drm_framebuffer_funcs`, `drm_mode_config_funcs`.
- **GRKERNSEC_IO** — DRM driver MMIO via `pci_iomap` / `devm_ioremap`; no `iopl/ioperm`; VGA-arbiter access gated.
- **GRKERNSEC_HIDESYM** — `/sys/class/drm/card*/device/` kernel pointers (`drm_dev`, `drm_minor`, GPU MMIO base) masked from non-CAP_SYSLOG; debugfs `/sys/kernel/debug/dri/*` restricted to CAP_SYS_ADMIN.
- **GRKERNSEC_DMESG** — GPU-reset, GPU-hang, page-fault, MMU-fault log lines restricted to CAP_SYSLOG.
- **GRKERNSEC_TPE** — Trusted Path Execution for `/dev/dri/card*` (full-mode-setting + display ownership) and `/dev/dri/renderD*` (compute/render-node).
- **GRKERNSEC_KMOD** — DRM driver auto-bind (amdgpu, i915, nouveau, radeon, xe) gated by CAP_SYS_MODULE.
- **GRKERNSEC_MODHARDEN** — module signature required for `drm`, `drm_kms_helper`, all per-vendor DRM drivers; firmware blobs (`amdgpu/*`, `i915/*`, `nvidia/*`) verified via `request_firmware_into_buf` + IMA appraisal.
- **GRKERNSEC_NO_FBSPLASH** — raw framebuffer access via fbdev compat layer (`/dev/fb*`) gated; `drm_fbdev_generic_setup` policy-controlled.
- **CAP_SYS_RAWIO** strict for legacy mode-setting ioctls (`DRM_IOCTL_MODE_*` on `/dev/dri/card*` from non-DRM-master), GPU-debugging ioctls (umr, amdgpu-debug, i915-gem-context-setparam-priority elevation), debugfs writes.
- **Render-node CAP_SYS_RAWIO bypass policy** — `/dev/dri/renderD*` is intentionally accessible to unprivileged users for compute; grsec policy enforces that *only* render-only ioctls (GEM/exec/syncobj) reach those nodes and any mode-setting ioctl returns -EACCES even for the GPU's render-node minor.
- **GRKERNSEC_BRUTE** — per-uid GPU-hang / VM-fault rate-limit; defense against compute-shader-DoS that wedges the scheduler.
- **PAX-anti-DMA-from-userland** — GEM `userptr` requires `CAP_SYS_RAWIO` for kernel-allocated pages mode; PRIME-import from foreign DMA-buf verified against IOMMU group of the importing device.

Per-driver rationale: DRM exposes two distinct chardev classes — primary nodes (`card*`, full display ownership including mode-setting and DMA-buf import) and render nodes (`renderD*`, intentionally world-accessible for compute and rendering); a compromised userspace process can submit arbitrary GPU command streams that read/write any GPU-visible memory if the driver fails to parse/validate IBs, and userptr GEM lets userspace pin its own pages into the GPU's IOMMU domain. Grsec reinforces by signing every DRM ops vtable (kCFI), gating legacy mode-setting + debugfs behind CAP_SYS_RAWIO even on the primary node, enforcing the render-node "no mode-setting" boundary, sanitizing VRAM-evict pages on free (cross-tenant data leak vector), and verifying firmware blobs through IMA before `request_firmware`.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/gpu/drm/drm_ioctl.c IOCTL dispatch table + drm_ioctl entry (covered in `drm-ioctl.md` Tier-3 when expanded)
- drivers/gpu/drm/drm_file.c struct drm_file lifecycle (covered in `drm-file.md` Tier-3 when expanded)
- drivers/gpu/drm/drm_sysfs.c sysfs minor class (covered in `drm-sysfs.md` Tier-3 when expanded)
- drivers/gpu/drm/drm_debugfs.c debugfs dri/<index> tree (covered separately)
- drivers/gpu/drm/drm_client.c in-kernel client framework (covered in `drm-client.md` Tier-3 when expanded)
- drivers/gpu/drm/drm_managed.c drmm_* devres-like helpers (covered in `drm-managed.md` Tier-3 when expanded)
- drivers/gpu/drm/drm_gem.c GEM core (covered in `gem-core.md` Tier-3)
- drivers/gpu/drm/drm_atomic.c atomic modesetting core (covered in `atomic.md` Tier-3)
- drivers/accel/* accel core (separately)
- Implementation code
