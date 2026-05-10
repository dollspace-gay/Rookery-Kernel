# Tier-3: drivers/vfio/vfio_main.c — VFIO core meta-driver

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/vfio/00-overview.md
upstream-paths:
  - drivers/vfio/vfio_main.c (~1856 lines)
  - drivers/vfio/vfio.h
  - include/linux/vfio.h
  - include/uapi/linux/vfio.h
-->

## Summary

**VFIO (Virtual Function I/O)** is the kernel framework that lets unprivileged userspace own a physical device safely behind the IOMMU — the user gets the device's MMIO regions, IRQs, and configuration space, while the IOMMU isolates its DMA. The `vfio_main.c` core layer ties together: per-device objects (`struct vfio_device`), per-IOMMU-group containment (group cdev `/dev/vfio/N`), and per-container IOMMU backends (type1, spapr_tce). Per-driver registration (`vfio_register_group_dev`, `vfio_register_emulated_iommu_dev`) hooks vendor drivers (`vfio-pci`, `mlx5-vfio-pci`, mdev) into the core. Per-userspace API supports legacy group/container path and new cdev / iommufd path. Per-migration FSM (STOP/RUNNING/PRE_COPY/STOP_COPY/RESUMING/RUNNING_P2P/ERROR) supports live-migration of stateful devices. Per-DMA-logging API supports dirty-page tracking. Per-DEVICE_FEATURE multiplexes capability queries. Critical for: VM device passthrough (QEMU/KVM), DPDK userspace networking, GPU pass-through, accelerator assignment.

This Tier-3 covers `drivers/vfio/vfio_main.c` (~1856 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vfio_device` | per-device descriptor | `VfioDevice` |
| `struct vfio_device_set` | per-set (e.g. multi-function PCI) | `VfioDeviceSet` |
| `struct vfio_device_file` | per-fd state | `VfioDeviceFile` |
| `struct vfio_device_ops` | per-driver vtable | `VfioDeviceOps` |
| `_vfio_alloc_device()` | per-alloc + init | `VfioCore::alloc_device` |
| `vfio_init_device()` | per-device init | `VfioCore::init_device` |
| `vfio_register_group_dev()` | per-register (IOMMU-backed) | `VfioCore::register_group_dev` |
| `vfio_register_emulated_iommu_dev()` | per-register (emulated mdev) | `VfioCore::register_emulated` |
| `vfio_unregister_group_dev()` | per-unregister + drain | `VfioCore::unregister_group_dev` |
| `vfio_assign_device_set()` | per-attach to dev-set | `VfioCore::assign_device_set` |
| `vfio_release_device_set()` | per-detach | `VfioCore::release_device_set` |
| `vfio_device_set_open_count()` | per-set open-count | `VfioDeviceSet::open_count` |
| `vfio_find_device_in_devset()` | per-set device lookup | `VfioDeviceSet::find` |
| `vfio_device_put_registration()` | per-ref-drop | `VfioDevice::put_registration` |
| `vfio_device_try_get_registration()` | per-ref-acquire | `VfioDevice::try_get` |
| `vfio_allocate_device_file()` | per-fd allocate | `VfioDeviceFile::alloc` |
| `vfio_df_open()` / `vfio_df_close()` | per-fd open/close gate | `VfioDeviceFile::open` / `close` |
| `vfio_df_device_first_open()` | per-first-open: bind iommu | `VfioDeviceFile::first_open` |
| `vfio_df_device_last_close()` | per-last-close: unbind | `VfioDeviceFile::last_close` |
| `vfio_device_fops` | per-cdev file_operations | `VfioDeviceFops` |
| `vfio_device_fops_unl_ioctl()` | per-ioctl dispatch | `VfioDeviceFile::ioctl` |
| `vfio_ioctl_device_feature()` | per-DEVICE_FEATURE dispatch | `VfioDeviceFile::ioctl_feature` |
| `vfio_ioctl_device_feature_migration()` | per-MIGRATION feature | `VfioFeature::migration` |
| `vfio_ioctl_device_feature_mig_device_state()` | per-MIG_DEVICE_STATE | `VfioFeature::mig_state` |
| `vfio_mig_get_next_state()` | per-FSM step compute | `VfioMigration::next_state` |
| `vfio_ioctl_device_feature_logging_start/_stop/_report()` | per-DMA-logging | `VfioFeature::logging_*` |
| `vfio_combine_iova_ranges()` | per-range coalesce | `VfioLogging::combine_ranges` |
| `vfio_get_region_info()` | per-REGION_INFO | `VfioDevice::region_info` |
| `vfio_info_cap_add()` / `_shift()` | per-capability chain | `VfioInfoCap::add` / `shift` |
| `vfio_set_irqs_validate_and_prepare()` | per-IRQ_SET validate | `VfioIrq::validate` |
| `vfio_pin_pages()` / `vfio_unpin_pages()` | per-mdev page pin | `VfioCore::pin_pages` / `unpin` |
| `vfio_dma_rw()` | per-mdev virtual-DMA | `VfioCore::dma_rw` |
| `vfio_file_is_valid()` | per-fd type check | `VfioFile::is_valid` |
| `vfio_file_enforced_coherent()` | per-fd coherency probe | `VfioFile::enforced_coherent` |
| `vfio_file_set_kvm()` | per-KVM-link | `VfioFile::set_kvm` |
| `vfio_device_get_kvm_safe()` / `_put_kvm()` | per-KVM-ref (CONFIG_KVM) | `VfioDevice::get_kvm` / `put_kvm` |

## Compatibility contract

REQ-1: struct vfio_device:
- dev: per-physical struct device.
- ops: per-driver vfio_device_ops vtable.
- group: per-vfio_group (legacy IOMMU-group path).
- dev_set: per-vfio_device_set (sharing set, e.g. multifunction PCI).
- index: per-device-class minor (vfioN).
- inode: per-anon vfio-fs inode (used for dev_set lookup).
- refcount: per-registration refcount.
- comp: per-completion waited on unregister.
- open_count: per-device open count (cdev or group).
- migration_flags: per-bitmask (STOP_COPY | PRE_COPY | P2P).
- mig_ops: per-migration ops (NULL = no-migration).
- log_ops: per-dirty-logging ops (NULL = no-logging).
- iommufd_access: per-iommufd-mode access handle.
- kvm / put_kvm: per-KVM link (cleared at last_close).
- precopy_info_v2: per-flag for PRECOPY_INFO_V2 enablement.

REQ-2: vfio_register_group_dev(device) / vfio_register_emulated_iommu_dev(device):
- WARN if CONFIG_IOMMUFD ∧ !(ops->bind_iommufd ∧ unbind_iommufd ∧ attach_ioas ∧ detach_ioas) → -EINVAL.
- If !device.dev_set: assign-singleton-set self-id.
- dev_set_name(device.device, "vfio%d", device.index).
- vfio_device_set_group(device, type) — type ∈ {VFIO_IOMMU, VFIO_EMULATED_IOMMU}.
- If type == VFIO_IOMMU ∧ !vfio_device_is_noiommu(device) ∧ !device_iommu_capable(device.dev, IOMMU_CAP_CACHE_COHERENCY) → -EINVAL.
- vfio_device_add(device).
- refcount_set(&device.refcount, 1).
- vfio_device_group_register + vfio_device_debugfs_init.
- Return 0 on success; rollback chain on error.

REQ-3: vfio_unregister_group_dev(device):
- vfio_device_group_unregister(device) — prevents new VFIO_GROUP_GET_DEVICE_FD.
- vfio_device_del(device) — prevents new cdev open.
- vfio_device_put_registration(device).
- Loop: try_wait_for_completion(&device.comp).
  - If pending, ops->request(device, iter++) to signal userspace.
  - Wait HZ*10; if interrupted, switch to uninterruptible timeout wait + dev_warn.
- vfio_device_debugfs_exit + vfio_device_remove_group.

REQ-4: vfio_assign_device_set(device, set_id):
- xa_load(vfio_device_set_xa, set_id) — atomic acquire.
- If absent: alloc + xa_cmpxchg. If xa_is_err → propagate.
- dev_set.device_count++; device.dev_set = dev_set; list_add_tail(device.dev_set_list).
- Return 0.

REQ-5: vfio_release_device_set(device):
- list_del under dev_set.lock.
- xa_lock; if --device_count == 0 → __xa_erase + free.

REQ-6: vfio_df_open / _close (per device-fd lifecycle):
- df.access_granted gates ioctl/read/write/mmap (smp_load_acquire pair).
- Group path may open multiple times; cdev path: only first open is permitted.
- First open: try_module_get(driver->owner); bind iommu (iommufd or group); ops->open_device.
- Last close: ops->close_device; unbind iommu; precopy_info_v2 = 0; module_put.

REQ-7: vfio_device_fops (cdev path):
- .open = vfio_device_fops_cdev_open.
- .release = vfio_device_fops_release — calls vfio_df_group_close OR vfio_df_unbind_iommufd; vfio_device_put_registration; kfree(df).
- .read / .write / .mmap — all gated on df.access_granted; delegate to ops->{read,write,mmap}.
- .unlocked_ioctl = vfio_device_fops_unl_ioctl.

REQ-8: vfio_device_fops_unl_ioctl(cmd):
- VFIO_DEVICE_BIND_IOMMUFD — fast path before access_granted check.
- Require smp_load_acquire(df.access_granted).
- vfio_device_pm_runtime_get(device) (resume_and_get if has pm).
- If cdev ∧ !df.group:
  - VFIO_DEVICE_ATTACH_IOMMUFD_PT → vfio_df_ioctl_attach_pt.
  - VFIO_DEVICE_DETACH_IOMMUFD_PT → vfio_df_ioctl_detach_pt.
- VFIO_DEVICE_FEATURE → vfio_ioctl_device_feature.
- VFIO_DEVICE_GET_REGION_INFO → vfio_get_region_info.
- default → ops->ioctl.
- pm_runtime_put on exit.

REQ-9: vfio_ioctl_device_feature(arg):
- copy_from_user(feature, arg, minsz).
- argsz < minsz → -EINVAL.
- Reject unknown flags (mask: VFIO_DEVICE_FEATURE_MASK | _SET | _GET | _PROBE).
- SET ∧ GET (without PROBE) → -EINVAL.
- Dispatch by feature.flags & VFIO_DEVICE_FEATURE_MASK:
  - VFIO_DEVICE_FEATURE_MIGRATION → migration cap query.
  - VFIO_DEVICE_FEATURE_MIG_DEVICE_STATE → state get/set.
  - VFIO_DEVICE_FEATURE_DMA_LOGGING_START / _STOP / _REPORT.
  - VFIO_DEVICE_FEATURE_MIG_DATA_SIZE → stop_copy_length probe.
  - VFIO_DEVICE_FEATURE_MIG_PRECOPY_INFOv2 → enable PRECOPY_INFO v2.
  - default → ops->device_feature or -ENOTTY.

REQ-10: vfio_mig_get_next_state(device, cur, new, *next):
- VFIO_DEVICE_NUM_STATES = PRE_COPY_P2P + 1.
- vfio_from_fsm_table[cur][new] = next-step (skipping arcs whose state_flags requirement is unmet).
- state_flags_table maps each state → required migration_flags bits.
- Reject when cur/new states are not supported by device.migration_flags.
- Walk through unsupported intermediate steps (jump-over).
- ERROR target → -EINVAL; otherwise 0.

REQ-11: vfio_ioctl_device_feature_mig_device_state(GET):
- mig_ops->migration_get_state(device, &curr).
- mig.device_state = curr; copy_to_user (data_fd = -1).

REQ-12: vfio_ioctl_device_feature_mig_device_state(SET):
- filp = mig_ops->migration_set_state(device, mig.device_state).
- If !IS_ERR(filp) ∧ filp != NULL: vfio_ioct_mig_return_fd(filp, arg, &mig).
- Else: mig.data_fd = -1; copy_to_user; return PTR_ERR if any.

REQ-13: vfio_ioctl_device_feature_logging_start/_stop/_report:
- LOG_MAX_RANGES = PAGE_SIZE / sizeof(vfio_device_feature_dma_logging_range).
- Validate per-range alignment to control.page_size + check_add_overflow.
- Build rb_root_cached of interval_tree_node; reject overlap; sort + insert.
- log_ops->log_start(device, &root, nnodes, &control.page_size).
- _report: iova_bitmap_alloc(report.iova, length, page_size, user_bitmap); iova_bitmap_for_each → log_ops->log_read_and_clear; iova_bitmap_free.
- _stop: log_ops->log_stop(device).

REQ-14: vfio_combine_iova_ranges(root, cur, req):
- Special-case req==1: collapse all into first node, retain max .last.
- Otherwise greedy: while cur > req, find minimum-gap pair (prev.last → curr.start), merge; loop until cur == req.

REQ-15: vfio_get_region_info:
- copy_from_user info(minsz).
- argsz < minsz → -EINVAL.
- ops->get_region_info_caps(device, &info, &caps).
- If caps.size > 0: set VFIO_REGION_INFO_FLAG_CAPS; if argsz insufficient, set argsz = sizeof(info) + caps.size, cap_offset = 0; else copy capability buffer trailing the struct.

REQ-16: vfio_pin_pages / vfio_unpin_pages:
- Only meaningful for `_register_emulated_iommu_dev` devices (mdev) — non-IOMMU-backed.
- If vfio_device_has_container(device): vfio_device_container_{pin,unpin}_pages.
- Else if iommufd_access: iommufd_access_pin_pages with IOMMUFD_ACCESS_RW_WRITE flag if prot has IOMMU_WRITE.
- VFIO ignores sub-page offset — caller adds (iova % PAGE_SIZE) to pages[0].

REQ-17: vfio_dma_rw:
- Virtual-DMA via CPU on behalf of mdev device.
- If kthread (no current.mm): IOMMUFD_ACCESS_RW_KTHREAD.
- If write: IOMMUFD_ACCESS_RW_WRITE.
- Container path: vfio_device_container_dma_rw.

REQ-18: vfio_file_set_kvm / _enforced_coherent / _is_valid:
- Exported helpers used by KVM to associate a KVM with VFIO file (group or device).
- Coherent = group→vfio_group_enforced_coherent OR device→device_iommu_capable(IOMMU_CAP_ENFORCE_CACHE_COHERENCY).
- vfio_device_file_set_kvm: spin_lock(df.kvm_ref_lock); df.kvm = kvm.

## Acceptance Criteria

- [ ] AC-1: vfio_register_group_dev publishes device under `/sys/class/vfio-dev/vfioN` and registers cdev.
- [ ] AC-2: With CONFIG_IOMMUFD, registration WARNs and fails if iommufd ops absent.
- [ ] AC-3: Cache-coherency capability required for VFIO_IOMMU (non-noiommu).
- [ ] AC-4: vfio_unregister_group_dev waits for last fd close before returning; ops->request is called to signal blocked openers.
- [ ] AC-5: vfio_assign_device_set is idempotent — second call with same set_id yields the same struct.
- [ ] AC-6: Device fd ioctl gates on smp_load_acquire(df.access_granted) — racing open is rejected.
- [ ] AC-7: VFIO_DEVICE_BIND_IOMMUFD works before access_granted is set.
- [ ] AC-8: VFIO_DEVICE_FEATURE rejects (GET | SET) without PROBE.
- [ ] AC-9: vfio_mig_get_next_state refuses unsupported endpoints; walks through unsupported intermediate states.
- [ ] AC-10: DMA_LOGGING_START rejects unaligned ranges and overlapping ranges.
- [ ] AC-11: DMA_LOGGING_REPORT enforces page_size ≥ 4 KiB ∧ power-of-2.
- [ ] AC-12: vfio_combine_iova_ranges collapses N ranges to req_nodes with smallest-gap merging.
- [ ] AC-13: vfio_pin_pages on mdev w/o container ∧ w/o iommufd_access → -EINVAL.
- [ ] AC-14: vfio_dma_rw flags IOMMUFD_ACCESS_RW_KTHREAD when current.mm is NULL.

## Architecture

```
struct VfioDevice {
  dev: *Device,
  ops: *VfioDeviceOps,
  group: Option<*VfioGroup>,
  dev_set: *VfioDeviceSet,
  index: u32,
  inode: *Inode,
  device: Device,                   // /sys/class/vfio-dev/vfioN
  refcount: Refcount,
  comp: Completion,
  open_count: AtomicU32,
  migration_flags: u32,             // VFIO_MIGRATION_*
  mig_ops: Option<*VfioMigrationOps>,
  log_ops: Option<*VfioLogOps>,
  iommufd_access: Option<*IommufdAccess>,
  #[cfg(CONFIG_KVM)] kvm: Option<*Kvm>,
  #[cfg(CONFIG_KVM)] put_kvm: Option<fn(*Kvm)>,
  precopy_info_v2: u8,
}

struct VfioDeviceSet {
  set_id: *const(),
  lock: Mutex,
  device_list: List<VfioDevice.dev_set_list>,
  device_count: u32,
}

struct VfioDeviceFile {
  device: *VfioDevice,
  group: Option<*VfioGroup>,
  iommufd: Option<*IommufdCtx>,
  access_granted: AtomicBool,       // release/acquire-paired
  kvm: Option<*Kvm>,
  kvm_ref_lock: SpinLock,
}
```

`VfioCore::register_group_dev(device, type)`:
1. /* iommufd-cap check */
2. if CONFIG_IOMMUFD ∧ !(ops.bind_iommufd ∧ unbind_iommufd ∧ attach_ioas ∧ detach_ioas): return -EINVAL.
3. if device.dev_set.is_none(): VfioCore::assign_device_set(device, device as *const()).
4. dev_set_name(&device.device, "vfio{}", device.index).
5. VfioGroup::set_group(device, type).
6. /* Coherency gate */
7. if type == VFIO_IOMMU ∧ !device.is_noiommu() ∧ !device_iommu_capable(device.dev, IOMMU_CAP_CACHE_COHERENCY): return -EINVAL.
8. VfioDevice::add(device).
9. device.refcount.set(1).
10. VfioGroup::register(device).
11. VfioDebugfs::init(device).
12. return 0.

`VfioCore::unregister_group_dev(device)`:
1. VfioGroup::unregister(device).
2. VfioDevice::del(device).
3. VfioDevice::put_registration(device).
4. loop:
   - rc = device.comp.try_wait().
   - if rc > 0: break.
   - if device.ops.request.is_some(): device.ops.request(device, i++).
   - if !interrupted: rc = device.comp.wait_interruptible_timeout(HZ * 10).
     - if rc < 0: interrupted = true; dev_warn(blocked-on-task).
   - else: rc = device.comp.wait_timeout(HZ * 10).
5. VfioDebugfs::exit(device).
6. VfioGroup::remove(device).

`VfioDeviceFile::open(df)`:
1. lockdep_assert_held(device.dev_set.lock).
2. if device.open_count != 0 ∧ df.group.is_none(): return -EINVAL.   // cdev: single open
3. device.open_count += 1.
4. if device.open_count == 1:
   - ret = VfioDeviceFile::first_open(df).
   - if ret: device.open_count -= 1.
5. return ret.

`VfioDeviceFile::first_open(df)`:
1. if !try_module_get(device.dev.driver.owner): return -ENODEV.
2. if df.iommufd.is_some(): ret = vfio_df_iommufd_bind(df).
3. else: ret = vfio_device_group_use_iommu(device).
4. if ret: goto err_module_put.
5. if device.ops.open_device.is_some(): ret = device.ops.open_device(device).
6. if ret: goto err_unuse_iommu.
7. return 0.

`VfioMigration::next_state(device, cur, new) -> Result<state>`:
1. if cur >= NUM_STATES ∨ (state_flags[cur] & device.migration_flags) != state_flags[cur]: -EINVAL.
2. if new >= NUM_STATES ∨ (state_flags[new] & device.migration_flags) != state_flags[new]: -EINVAL.
3. next = vfio_from_fsm_table[cur][new].
4. while (state_flags[next] & device.migration_flags) != state_flags[next]:
   - next = vfio_from_fsm_table[next][new].
5. return if next == ERROR { Err(-EINVAL) } else { Ok(next) }.

`VfioFeature::ioctl_feature(device, user_arg)`:
1. copy_from_user(feature, user_arg, minsz).
2. if feature.argsz < minsz: -EINVAL.
3. /* Validate flags */
4. if feature.flags & ~(MASK | SET | GET | PROBE): -EINVAL.
5. if !(flags & PROBE) ∧ (flags & SET) ∧ (flags & GET): -EINVAL.
6. match feature.flags & VFIO_DEVICE_FEATURE_MASK:
   - MIGRATION → VfioFeature::migration.
   - MIG_DEVICE_STATE → VfioFeature::mig_state.
   - DMA_LOGGING_{START,STOP,REPORT} → VfioFeature::logging_*.
   - MIG_DATA_SIZE → VfioFeature::mig_data_size.
   - MIG_PRECOPY_INFOv2 → VfioFeature::precopy_v2.
   - default → device.ops.device_feature.unwrap_or(-ENOTTY).

`VfioLogging::start(device, control, user_ranges)`:
1. if !device.log_ops: -ENOTTY.
2. check_feature(SET, sizeof(control)).
3. copy_from_user(control, ranges-minsz).
4. if control.num_ranges == 0: -EINVAL.
5. if control.num_ranges > LOG_MAX_RANGES: -E2BIG.
6. nodes = kmalloc_objs(IntervalTreeNode, num_ranges).
7. root = RbRootCached::empty().
8. for i in 0..num_ranges:
   - copy_from_user(range, ranges[i]).
   - if !aligned(range.iova, control.page_size) ∨ !aligned(range.length, control.page_size): -EINVAL.
   - if check_add_overflow(range.iova, range.length, &iova_end) ∨ iova_end > ULONG_MAX: -EOVERFLOW.
   - nodes[i] = { start: iova, last: iova+length-1 }.
   - if root.iter_first(start, last).is_some(): -EINVAL.   // overlap
   - root.insert(nodes[i]).
9. ret = device.log_ops.log_start(device, &root, num_ranges, &control.page_size).
10. if copy_to_user(control): log_ops.log_stop(device); -EFAULT.
11. kfree(nodes); return ret.

`VfioLogging::combine_iova_ranges(root, cur, req)`:
1. if req == 1:
   - first = root.iter_first().
   - walk; remove all but first; set first.last = walked-max.
   - return.
2. while cur > req:
   - find min-gap pair (prev.last → curr.start).
   - comb_start.last = comb_end.last; root.remove(comb_end).
   - cur -= 1.

`VfioCore::pin_pages(device, iova, npage, prot, pages) -> Result<i32>`:
1. if !pages ∨ !npage ∨ !device.is_open(): -EINVAL.
2. if !device.ops.dma_unmap: -EINVAL.
3. if device.has_container():
   - return container_pin_pages(device, iova, npage, prot, pages).
4. if device.iommufd_access.is_some():
   - if iova > ULONG_MAX: -EINVAL.
   - flags = if prot & IOMMU_WRITE { IOMMUFD_ACCESS_RW_WRITE } else { 0 }.
   - ret = iommufd_access_pin_pages(handle, ALIGN_DOWN(iova, PAGE_SIZE), npage * PAGE_SIZE, pages, flags).
   - return npage.
5. -EINVAL.

`VfioCore::dma_rw(device, iova, data, len, write) -> Result<()>`:
1. if !data ∨ len == 0 ∨ !device.is_open(): -EINVAL.
2. if device.has_container(): container_dma_rw(device, iova, data, len, write).
3. if device.iommufd_access.is_some():
   - if iova > ULONG_MAX: -EINVAL.
   - flags = 0.
   - if current.mm.is_none(): flags |= IOMMUFD_ACCESS_RW_KTHREAD.
   - if write: flags |= IOMMUFD_ACCESS_RW_WRITE.
   - return iommufd_access_rw(handle, iova, data, len, flags).
4. -EINVAL.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `register_paths_balanced` | INVARIANT | per-register: every error path runs vfio_device_remove_group. |
| `unregister_drains_to_zero` | INVARIANT | per-unregister: returns only after device.comp completes. |
| `device_set_refcount_balanced` | INVARIANT | per-assign + per-release: dev_set.device_count get == put. |
| `access_granted_acquire_release` | INVARIANT | per-fd: ioctl reads access_granted via smp_load_acquire paired with open's smp_store_release. |
| `feature_flag_exclusion` | INVARIANT | per-VFIO_DEVICE_FEATURE: SET ∧ GET ⟹ PROBE must be set. |
| `mig_fsm_termination` | INVARIANT | per-mig_get_next_state: jump-over loop bounded by NUM_STATES. |
| `logging_range_no_overlap` | INVARIANT | per-logging_start: rb-tree insert rejects overlapping. |
| `logging_range_aligned` | INVARIANT | per-logging_start: every range aligned to control.page_size. |
| `combine_ranges_strict_decrease` | INVARIANT | per-combine_iova_ranges: each iteration strictly decreases cur_nodes. |
| `pin_pages_mdev_only` | INVARIANT | per-vfio_pin_pages: device must be mdev (has container or iommufd_access). |

### Layer 2: TLA+

`drivers/vfio/vfio-main.tla`:
- States: per-VfioDevice {Unregistered, Registered, OpenN, Unregistering, Removed}.
- Per-VfioMigration FSM: STOP, RUNNING, PRE_COPY, PRE_COPY_P2P, STOP_COPY, RESUMING, RUNNING_P2P, ERROR.
- Properties:
  - `safety_register_atomic` — per-register: either fully published or fully rolled back.
  - `safety_unregister_no_in_use_release` — per-unregister: returns only after all fds closed.
  - `safety_mig_fsm_invariant` — per-mig-FSM: state_flags requirement on every step.
  - `safety_logging_range_disjoint` — per-logging_start: in-tree ranges are pairwise disjoint.
  - `liveness_unregister_terminates` — per-unregister: eventually completes (ops->request signals userspace).
  - `liveness_pm_runtime_balanced` — per-ioctl: every get is paired with a put.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VfioCore::register_group_dev` post: success ⟺ device.refcount == 1 ∧ device added to dev_set | `VfioCore::register_group_dev` |
| `VfioCore::unregister_group_dev` post: device.refcount == 0 ∧ device.comp signaled | `VfioCore::unregister_group_dev` |
| `VfioDeviceFile::first_open` post: success ⟹ module ref taken ∧ iommu bound ∧ ops.open_device returned 0 | `VfioDeviceFile::first_open` |
| `VfioFeature::ioctl_feature` post: returns -EINVAL on unknown flag bits | `VfioFeature::ioctl_feature` |
| `VfioMigration::next_state` post: returned state is supported by device.migration_flags | `VfioMigration::next_state` |
| `VfioLogging::start` post: rb-tree contains exactly num_ranges disjoint sorted nodes | `VfioLogging::start` |
| `VfioLogging::combine_iova_ranges` post: tree has ≤ req_nodes nodes; union covers original union | `VfioLogging::combine_iova_ranges` |
| `VfioCore::pin_pages` post: returns npage on success; pages[0..npage] valid | `VfioCore::pin_pages` |

### Layer 4: Verus/Creusot functional

`Per-userspace-open → bind iommu → ops.open_device → ioctl/read/write/mmap → close → unbind` semantic equivalence: per-Documentation/driver-api/vfio.rst.

`Per-migration FSM` semantic equivalence: per-include/uapi/linux/vfio.h state_flags_table + vfio_from_fsm_table table semantics (state-graph reachability and step-skipping).

## Hardening

(Inherits row-1 features from `drivers/vfio/00-overview.md` § Hardening.)

VFIO core reinforcement:

- **Per-CONFIG_IOMMUFD bind-ops mandatory** — defense against per-register-without-iommufd-bind that would leave devices in an unbindable state.
- **Per-IOMMU_CAP_CACHE_COHERENCY required** — defense against per-non-coherent-DMA aliasing attacks (userspace would need wbinvd-like operations).
- **Per-noiommu module-param kernel taint** — defense against unintentional VFIO_NOIOMMU production deployment.
- **Per-fd smp_store_release / smp_load_acquire on access_granted** — defense against per-ioctl-before-fully-opened race.
- **Per-unregister wait-for-completion-with-ops.request** — defense against per-driver unbind-while-device-in-use silent crash.
- **Per-VFIO_DEVICE_FEATURE strict flag validation** — defense against per-future-uapi-bit smuggling.
- **Per-MIG_DEVICE_STATE state-flag enforcement** — defense against per-illegal-transition-issued device wedge.
- **Per-mig FSM jump-over loop bounded by NUM_STATES** — defense against per-FSM infinite-loop on inconsistent tables.
- **Per-DMA-logging range overlap rejection** — defense against per-double-tracked bitmap corruption.
- **Per-DMA-logging LOG_MAX_RANGES = PAGE_SIZE bounded** — defense against per-unbounded-kmalloc DoS.
- **Per-page_size ≥ 4 KiB ∧ power-of-2 for logging report** — defense against per-misaligned-bitmap-walk overflow.
- **Per-vfio_pin_pages mdev-only gate** — defense against per-direct-DMA bypass on IOMMU-backed devices.
- **Per-DMA_RW kthread-flag auto-detect** — defense against per-current.mm NULL pointer deref when called from kthreads.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/vfio/group.c` — group cdev + VFIO_GROUP_GET_DEVICE_FD (covered separately if expanded).
- `drivers/vfio/container.c` — container fd + VFIO_SET_IOMMU + VFIO_IOMMU_MAP_DMA / UNMAP (covered separately if expanded).
- `drivers/vfio/iommufd.c` — iommufd-compat bridge for vfio (covered separately if expanded).
- `drivers/vfio/iova_bitmap.c` — bitmap iterator (covered separately if expanded).
- `drivers/vfio/vfio_iommu_type1.c` — type1 IOMMU backend (covered in `vfio-iommu-type1.md` Tier-3 once written).
- `drivers/vfio/pci/` — vfio-pci bus driver (covered in `pci-core.md` Tier-3).
- KVM coupling (CONFIG_KVM kvm_get_kvm_safe + kvm_put_kvm via symbol_get/symbol_put) — covered in `virt/kvm` Tier-3 series.
- Implementation code.
