# Tier-3: drivers/vhost/vdpa.c — vhost-vdpa: vDPA frontend for hardware data-path acceleration

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/vhost/00-overview.md
upstream-paths:
  - drivers/vhost/vdpa.c (~1683 lines)
  - drivers/vhost/vhost.h
  - include/linux/vdpa.h (struct vdpa_device, vdpa_config_ops, vdpa_iova_range, vdpa_map_file)
  - include/uapi/linux/vhost.h (VHOST_VDPA_GET_DEVICE_ID, _GET_STATUS, _SET_STATUS, _GET_CONFIG, _SET_CONFIG, _SET_VRING_ENABLE, _GET_VRING_NUM, _GET_GROUP_NUM, _GET_AS_NUM, _GET_VRING_GROUP, _GET_VRING_DESC_GROUP, _SET_GROUP_ASID, _GET_IOVA_RANGE, _GET_CONFIG_SIZE, _GET_VQS_COUNT, _SUSPEND, _RESUME, _SET_CONFIG_CALL, _GET_VRING_SIZE)
  - include/uapi/linux/vhost_types.h (struct vhost_iotlb_msg, vhost_vdpa_config, vhost_vdpa_iova_range, VHOST_IOTLB_UPDATE, _INVALIDATE, _BATCH_BEGIN, _BATCH_END)
-->

## Summary

**vhost-vdpa** is a vhost-style frontend over the vDPA (virtio Data Path Accelerator) framework — it exposes a `/dev/vhost-vdpa-N` chardev that mirrors the `/dev/vhost-{net,scsi,vsock}` UAPI shape, but instead of running the data path in a host kernel worker, it forwards control operations to a `struct vdpa_device` (NIC, block, crypto, or other vendor-provided device whose vendor driver implements `struct vdpa_config_ops`). The userspace VMM (qemu, cloud-hypervisor, dpdk vhost-user-vdpa) drives the device just like a vhost backend — `VHOST_SET_OWNER`, `VHOST_GET_FEATURES`/`SET_FEATURES`, `VHOST_SET_MEM_TABLE` (via the write_iter `vhost_chr_write_iter` IOTLB message channel), `VHOST_SET_VRING_NUM`/`_ADDR`/`_BASE`/`_KICK`/`_CALL`, and `VHOST_VDPA_SET_STATUS(VIRTIO_CONFIG_S_DRIVER_OK)` — but each operation is bridged to the corresponding `ops->set_vq_*`/`set_features`/`set_status`/`get_status` callback on the vDPA device. Per-DMA mapping: vhost-vdpa carries a per-Address-Space-IDentifier (ASID) `struct vhost_iotlb` (hashed in `as[VHOST_VDPA_IOTLB_BUCKETS = 16]`); userspace pushes IOTLB messages (`VHOST_IOTLB_UPDATE` / `_INVALIDATE` / `_BATCH_BEGIN` / `_BATCH_END`) into the `write_iter` fd channel, which lands in `vhost_vdpa_process_iotlb_msg`. Mappings flow either to (a) `ops->dma_map`/`dma_unmap` (per-device IOMMU; SmartNIC vendor handles its own translation), (b) `ops->set_map` (whole-IOTLB shadow push for devices that prefer batched updates), or (c) `iommu_map`/`iommu_unmap` on `v->domain` allocated via `iommu_paging_domain_alloc(dma_dev)` (platform IOMMU fallback). Pages are pinned via `pin_user_pages(FOLL_LONGTERM)` for PA-mode mappings; or `get_file` + the VMA `vm_file` is recorded as a `vdpa_map_file` opaque cookie for VA-mode (`vdpa->use_va` — SVA-capable devices). Per-irq: `ops->get_vq_irq` returns the device's MSI vector for VQ `qid`; vhost-vdpa registers an `irq_bypass_producer` so that KVM (consumer on the guest side) can inject the IRQ directly into the guest without a host VMM hop. Per-vDPA device probe: vhost-vdpa registers a `vdpa_driver { .probe = vhost_vdpa_probe, .remove = vhost_vdpa_remove }`; when a vDPA bus device is bound, `probe` allocates a `vhost_vdpa`, allocates a minor via `vhost_vdpa_ida`, creates the cdev at `(major,minor)`, and calls `cdev_device_add`. Per-suspend/resume: `VHOST_VDPA_SUSPEND`/`_RESUME` ioctls invoke `ops->suspend`/`ops->resume` so live-migration can quiesce + snapshot vring state via `ops->get_vq_state`/`set_vq_state`. Critical for: zero-overhead virtio data path on smartNICs/DPUs (Mellanox BlueField, Intel IPU), passthrough-grade throughput while keeping the virtio control plane, live-migration of hardware-accelerated VMs.

This Tier-3 covers `drivers/vhost/vdpa.c` (~1683 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vhost_vdpa` | per-/dev/vhost-vdpa-N fd device | `VhostVdpa` |
| `struct vhost_vdpa_as` | per-ASID IOTLB bucket | `VhostVdpaAs` |
| `struct vhost_iotlb` | per-ASID interval-tree IOTLB | `VhostIotlb` (shared) |
| `vhost_vdpa_probe()` | per-vDPA device bind | `VhostVdpa::probe` |
| `vhost_vdpa_remove()` | per-vDPA device unbind | `VhostVdpa::remove` |
| `vhost_vdpa_open()` | per-fd open | `VhostVdpa::open` |
| `vhost_vdpa_release()` | per-fd release | `VhostVdpa::release` |
| `vhost_vdpa_unlocked_ioctl()` | per-ioctl dispatch | `VhostVdpa::ioctl` |
| `vhost_vdpa_vring_ioctl()` | per-vring ioctl bridge | `VhostVdpa::vring_ioctl` |
| `vhost_vdpa_chr_write_iter()` | per-IOTLB-message channel | `VhostVdpa::chr_write_iter` |
| `vhost_vdpa_mmap()` / `vhost_vdpa_fault()` | per-doorbell page mmap | `VhostVdpa::mmap` / `fault` |
| `vhost_vdpa_get_device_id()` | per-`VHOST_VDPA_GET_DEVICE_ID` | `VhostVdpa::get_device_id` |
| `vhost_vdpa_get_status()` / `set_status()` | per-virtio status get/set | `VhostVdpa::get_status` / `set_status` |
| `vhost_vdpa_get_config()` / `set_config()` | per-virtio config-space get/set | `VhostVdpa::get_config` / `set_config` |
| `vhost_vdpa_config_validate()` | per-off+len bounds check | `VhostVdpa::config_validate` |
| `vhost_vdpa_get_features()` / `set_features()` | per-virtio device feature negotiation | `VhostVdpa::get_features` / `set_features` |
| `vhost_vdpa_get_backend_features()` | per-backend feature bits | `VhostVdpa::get_backend_features` |
| `vhost_vdpa_get_vring_num()` | per-max VQ depth | `VhostVdpa::get_vring_num` |
| `vhost_vdpa_get_iova_range()` | per-vDPA-IOVA aperture | `VhostVdpa::get_iova_range` |
| `vhost_vdpa_get_config_size()` | per-config-space size | `VhostVdpa::get_config_size` |
| `vhost_vdpa_get_vqs_count()` | per-VQ count | `VhostVdpa::get_vqs_count` |
| `vhost_vdpa_set_config_call()` | per-config eventfd binding | `VhostVdpa::set_config_call` |
| `vhost_vdpa_suspend()` / `vhost_vdpa_resume()` | per-live-mig quiesce | `VhostVdpa::suspend` / `resume` |
| `vhost_vdpa_reset()` / `_compat_vdpa_reset()` | per-virtio device reset | `VhostVdpa::reset` |
| `vhost_vdpa_bind_mm()` / `unbind_mm()` | per-SVA-mode mm binding | `VhostVdpa::bind_mm` / `unbind_mm` |
| `vhost_vdpa_can_suspend()` / `can_resume()` / `has_desc_group()` / `has_persistent_map()` | per-feature gating | `VhostVdpa::can_suspend` etc. |
| `vhost_vdpa_setup_vq_irq()` / `unsetup_vq_irq()` | per-VQ irq-bypass producer setup | `VhostVdpa::setup_vq_irq` / `unsetup_vq_irq` |
| `vhost_vdpa_virtqueue_cb()` / `_config_cb()` | per-vDPA callback → eventfd_signal | `VhostVdpa::virtqueue_cb` / `config_cb` |
| `handle_vq_kick()` | per-VQ kick → `ops->kick_vq` | `VhostVdpa::handle_vq_kick` |
| `vhost_vdpa_process_iotlb_msg()` | per-IOTLB-message dispatch | `VhostVdpa::process_iotlb_msg` |
| `vhost_vdpa_process_iotlb_update()` | per-`VHOST_IOTLB_UPDATE` validate + map | `VhostVdpa::process_iotlb_update` |
| `vhost_vdpa_map()` / `vhost_vdpa_unmap()` | per-IOTLB map/unmap (route dma_map vs set_map vs iommu_map) | `VhostVdpa::map` / `unmap` |
| `vhost_vdpa_pa_map()` | per-PA-mode GUP-pin + map | `VhostVdpa::pa_map` |
| `vhost_vdpa_va_map()` | per-VA-mode (SVA) map | `VhostVdpa::va_map` |
| `vhost_vdpa_pa_unmap()` | per-PA-mode unpin + unmap | `VhostVdpa::pa_unmap` |
| `vhost_vdpa_va_unmap()` | per-VA-mode fput + unmap | `VhostVdpa::va_unmap` |
| `vhost_vdpa_iotlb_unmap()` | per-ASID range unmap (routes PA vs VA) | `VhostVdpa::iotlb_unmap` |
| `vhost_vdpa_general_unmap()` | per-map dma_unmap vs iommu_unmap | `VhostVdpa::general_unmap` |
| `vhost_vdpa_alloc_domain()` / `free_domain()` | per-IOMMU domain alloc+attach | `VhostVdpa::alloc_domain` / `free_domain` |
| `vhost_vdpa_set_iova_range()` | per-IOVA aperture init | `VhostVdpa::set_iova_range` |
| `vhost_vdpa_cleanup()` | per-fd-release cleanup | `VhostVdpa::cleanup` |
| `vhost_vdpa_clean_irq()` | per-fd-release IRQ teardown | `VhostVdpa::clean_irq` |
| `vhost_vdpa_alloc_as()` / `_find_alloc_as()` / `_remove_as()` / `asid_to_as()` / `asid_to_iotlb()` / `iotlb_to_asid()` | per-ASID hash-bucket lookup/alloc/remove | `VhostVdpa::*_as` |
| `vhost_vdpa_reset_map()` | per-vendor reset_map callback | `VhostVdpa::reset_map` |
| `perm_to_iommu_flags()` | per-`VHOST_ACCESS_{RO,WO,RW}` → IOMMU_{READ,WRITE,CACHE} | `VhostVdpa::perm_to_iommu_flags` |
| `vhost_vdpa_config_put()` | per-config_ctx eventfd put | `VhostVdpa::config_put` |
| `vhost_vdpa_release_dev()` | per-`device.release` (after ida_free + kfree) | `VhostVdpa::release_dev` |
| `vhost_vdpa_driver` | per-vdpa-bus driver registration | `VHOST_VDPA_DRIVER` |
| `vhost_vdpa_init()` / `_exit()` | module init/exit | `VhostVdpa::init` / `exit` |

## Compatibility contract

REQ-1: struct vhost_vdpa:
- `vdev`: per-`vhost_dev` (embedded; vqs allocated separately at `vqs`).
- `domain`: per-`iommu_domain *` (NULL if device-managed IOMMU via `ops->dma_map` or `ops->set_map`).
- `vqs`: per-`vhost_virtqueue` array of length `nvqs`.
- `completion`: per-release-vs-remove race rendezvous.
- `vdpa`: per-back-pointer to `vdpa_device`.
- `as[VHOST_VDPA_IOTLB_BUCKETS = 16]`: per-ASID hash table.
- `dev`: per-`struct device` (driver model).
- `cdev`: per-cdev (`vhost_vdpa_fops`).
- `opened`: per-atomic open-counter (mutex of {0,1}).
- `nvqs`: per-`vdpa->nvqs`.
- `virtio_id`: per-`ops->get_device_id(vdpa)`.
- `minor`: per-allocated minor (`vhost_vdpa_ida`).
- `config_ctx`: per-`eventfd_ctx *` for config-change AEN.
- `in_batch`: per-`VHOST_IOTLB_BATCH_*` in-flight flag.
- `range`: per-`vdpa_iova_range` aperture.
- `batch_asid`: per-active-batch ASID.
- `suspended`: per-`VHOST_VDPA_SUSPEND` state.

REQ-2: struct vhost_vdpa_as:
- `hash_link`: per-`as[]` bucket hlist node.
- `iotlb`: per-ASID `vhost_iotlb` (interval tree).
- `id`: per-ASID id (`0 ≤ id < vdpa->nas`).

REQ-3: VhostVdpa::probe(vdpa):
- Reject if `!ops->set_map ∧ !ops->dma_map ∧ (vdpa->ngroups > 1 ∨ vdpa->nas > 1)` — platform-IOMMU mode supports only 1 group / 1 AS.
- Alloc `v = kzalloc(GFP_KERNEL | __GFP_RETRY_MAYFAIL)`.
- `minor = ida_alloc_max(&vhost_vdpa_ida, VHOST_VDPA_DEV_MAX - 1 = (1<<MINORBITS) - 1)`.
- `atomic_set(&v->opened, 0)`; `v->vdpa = vdpa`; `v->nvqs = vdpa->nvqs`; `v->virtio_id = ops->get_device_id(vdpa)`.
- `device_initialize(&v->dev)`; `v->dev.release = vhost_vdpa_release_dev`; `v->dev.parent = &vdpa->dev`; `v->dev.devt = MKDEV(MAJOR(vhost_vdpa_major), minor)`.
- `v->vqs = kmalloc_array(struct vhost_virtqueue, v->nvqs)`.
- `dev_set_name(&v->dev, "vhost-vdpa-%u", minor)`.
- `cdev_init(&v->cdev, &vhost_vdpa_fops)`; `v->cdev.owner = THIS_MODULE`.
- `cdev_device_add(&v->cdev, &v->dev)`.
- `init_completion(&v->completion)`.
- `vdpa_set_drvdata(vdpa, v)`.
- For each `as[]` bucket: `INIT_HLIST_HEAD`.

REQ-4: VhostVdpa::remove(vdpa):
- `cdev_device_del(&v->cdev, &v->dev)`.
- Wait loop: `atomic_cmpxchg(&v->opened, 0, 1)`; if previously 0 ⟹ break; else `wait_for_completion(&v->completion)` (set by release) and retry. Guarantees no fd still open.
- `put_device(&v->dev)` ⟹ triggers `vhost_vdpa_release_dev` which `ida_free`s minor + `kfree`s `v->vqs` + `v`.

REQ-5: VhostVdpa::open(file):
- `v = container_of(inode->i_cdev, struct vhost_vdpa, cdev)`.
- `opened = atomic_cmpxchg(&v->opened, 0, 1)`; if non-zero ⟹ -EBUSY.
- `vhost_vdpa_reset(v)` — `v->in_batch = 0` + `_compat_vdpa_reset` which calls `vdpa_reset(vdpa, VDPA_RESET_F_CLEAN_MAP)` unless persistent-map feature acked.
- Alloc `vqs = kmalloc_array(nvqs)`.
- For each VQ: `vqs[i] = &v->vqs[i]`; `vqs[i]->handle_kick = handle_vq_kick`; `vqs[i]->call_ctx.ctx = NULL`.
- `vhost_dev_init(&v->vdev, vqs, nvqs, iov_limit=0, weight=0, byte_weight=0, use_worker=false, msg_handler=vhost_vdpa_process_iotlb_msg)`.
- `vhost_vdpa_alloc_domain(v)`.
- `vhost_vdpa_set_iova_range(v)`.
- `file->private_data = v`.

REQ-6: VhostVdpa::release(file):
- Lock `vdev.mutex`.
- `file->private_data = NULL`.
- `vhost_vdpa_clean_irq(v)` — for each VQ: `irq_bypass_unregister_producer`.
- `vhost_vdpa_reset(v)` — wipe device state.
- `vhost_dev_stop(&v->vdev)`.
- `vhost_vdpa_unbind_mm(v)`.
- `vhost_vdpa_config_put(v)` — drop `config_ctx` eventfd.
- `vhost_vdpa_cleanup(v)` — for each ASID: `vhost_vdpa_remove_as`; `vhost_vdpa_free_domain`; `vhost_dev_cleanup`.
- Unlock; `atomic_dec(&v->opened)`; `complete(&v->completion)` (wakes `remove`).

REQ-7: VhostVdpa::alloc_domain(v):
- `map = vdpa_get_map(vdpa)`; `dma_dev = map.dma_dev`.
- If `ops->set_map ∨ ops->dma_map`: device-managed; return 0.
- Require `device_iommu_capable(dma_dev, IOMMU_CAP_CACHE_COHERENCY)` else -ENOTSUPP.
- `v->domain = iommu_paging_domain_alloc(dma_dev)`.
- `iommu_attach_device(v->domain, dma_dev)`.

REQ-8: VhostVdpa::set_iova_range(v):
- If `ops->get_iova_range`: `v->range = ops->get_iova_range(vdpa)`.
- Else if `v->domain ∧ v->domain->geometry.force_aperture`: `range = (aperture_start, aperture_end)`.
- Else: `range = (0, ULLONG_MAX)`.

REQ-9: VhostVdpa::ioctl(file, cmd, arg) dispatch:
- `VHOST_SET_BACKEND_FEATURES`:
  - Validate `features ⊆ VHOST_VDPA_BACKEND_FEATURES | F_DESC_ASID | F_IOTLB_PERSIST | F_SUSPEND | F_RESUME | F_ENABLE_AFTER_DRIVER_OK` else -EOPNOTSUPP.
  - Validate `F_SUSPEND` ⟹ `can_suspend`; `F_RESUME` ⟹ `can_resume`; `F_DESC_ASID` ⟹ `F_IOTLB_ASID` ∧ `has_desc_group`; `F_IOTLB_PERSIST` ⟹ `has_persistent_map`.
  - `vhost_set_backend_features(&v->vdev, features)`.
- Else under `dev.mutex`:
  - `VHOST_VDPA_GET_DEVICE_ID` ⟹ `ops->get_device_id`.
  - `VHOST_VDPA_GET_STATUS` / `_SET_STATUS` ⟹ `ops->get_status` / `set_status` (with VRING_ENABLE-AFTER-DRIVER_OK irq setup/teardown).
  - `VHOST_VDPA_GET_CONFIG` / `_SET_CONFIG` ⟹ `vdpa_get_config` / `vdpa_set_config` after `vhost_vdpa_config_validate`.
  - `VHOST_GET_FEATURES` ⟹ `ops->get_device_features`.
  - `VHOST_SET_FEATURES` ⟹ reject if `S_FEATURES_OK` set; `vdpa_set_features`; broadcast `acked_features` to all VQs.
  - `VHOST_VDPA_GET_VRING_NUM` ⟹ `ops->get_vq_num_max`.
  - `VHOST_VDPA_GET_GROUP_NUM` ⟹ `vdpa->ngroups`.
  - `VHOST_VDPA_GET_AS_NUM` ⟹ `vdpa->nas`.
  - `VHOST_VDPA_GET_IOVA_RANGE` ⟹ `v->range`.
  - `VHOST_VDPA_GET_CONFIG_SIZE` ⟹ `ops->get_config_size`.
  - `VHOST_VDPA_GET_VQS_COUNT` ⟹ `vdpa->nvqs`.
  - `VHOST_VDPA_SUSPEND` / `_RESUME` ⟹ `vhost_vdpa_suspend` / `resume`.
  - `VHOST_VDPA_SET_CONFIG_CALL` ⟹ `vhost_vdpa_set_config_call`.
  - `VHOST_GET_BACKEND_FEATURES` ⟹ `VHOST_VDPA_BACKEND_FEATURES | (can_suspend ? F_SUSPEND : 0) | (can_resume ? F_RESUME : 0) | (has_desc_group ? F_DESC_ASID : 0) | (has_persistent_map ? F_IOTLB_PERSIST : 0) | ops->get_backend_features()`.
  - `VHOST_SET_LOG_BASE` / `VHOST_SET_LOG_FD` ⟹ -ENOIOCTLCMD (no dirty logging in vdpa).
  - default ⟹ `vhost_dev_ioctl`; if -ENOIOCTLCMD ⟹ `vhost_vdpa_vring_ioctl`.
- Post-`SET_OWNER` hook: `vhost_vdpa_bind_mm(v)`; on fail ⟹ `vhost_dev_reset_owner(d, NULL)`.

REQ-10: VhostVdpa::vring_ioctl(cmd, argp):
- `idx = get_user(argp)`; bounds-check `idx < v->nvqs` (-ENOBUFS) and `array_index_nospec`.
- `vq = &v->vqs[idx]`.
- Pre-switch (no vhost-core):
  - `VHOST_VDPA_SET_VRING_ENABLE` ⟹ `ops->set_vq_ready(vdpa, idx, s.num)`.
  - `VHOST_VDPA_GET_VRING_GROUP` ⟹ `ops->get_vq_group(vdpa, idx)` (bounds-check vs `vdpa->ngroups`).
  - `VHOST_VDPA_GET_VRING_DESC_GROUP` ⟹ requires `F_DESC_ASID`; `ops->get_vq_desc_group(vdpa, idx)`.
  - `VHOST_VDPA_SET_GROUP_ASID` ⟹ reject if `S_DRIVER_OK`; `ops->set_group_asid(vdpa, idx, s.num)`.
  - `VHOST_VDPA_GET_VRING_SIZE` ⟹ `ops->get_vq_size(vdpa, idx)`.
  - `VHOST_GET_VRING_BASE` ⟹ `ops->get_vq_state` then translate packed/split into `vq.last_avail_idx` / `last_used_idx`.
  - `VHOST_SET_VRING_CALL` (pre) ⟹ if old `call_ctx.ctx ∧ S_DRIVER_OK` ⟹ `unsetup_vq_irq`.
- `vhost_vring_ioctl(&v->vdev, cmd, argp)` — common framework path.
- Post-switch:
  - `VHOST_SET_VRING_ADDR` ⟹ reject if `S_DRIVER_OK ∧ !suspended`; `ops->set_vq_address(vdpa, idx, desc, avail, used)`.
  - `VHOST_SET_VRING_BASE` ⟹ reject same; pack split/packed → `vq_state` → `ops->set_vq_state`.
  - `VHOST_SET_VRING_CALL` ⟹ build `vdpa_callback { vhost_vdpa_virtqueue_cb, vq, vq.call_ctx.ctx }` and `ops->set_vq_cb`; if `S_DRIVER_OK` ⟹ `setup_vq_irq(v, idx)`.
  - `VHOST_SET_VRING_NUM` ⟹ `ops->set_vq_num(vdpa, idx, vq.num)`.

REQ-11: VhostVdpa::process_iotlb_msg(dev, asid, msg):
- `vhost_dev_check_owner(dev)` else error.
- If `msg.type ∈ {UPDATE, BATCH_BEGIN}`: `as = vhost_vdpa_find_alloc_as(v, asid)` (allocates ASID + `vhost_iotlb_init` if absent); else `iotlb = asid_to_iotlb(v, asid)`.
- Reject if `v->in_batch ∧ v->batch_asid != asid` (-EINVAL).
- Switch `msg.type`:
  - `VHOST_IOTLB_UPDATE` ⟹ `vhost_vdpa_process_iotlb_update(v, iotlb, msg)`.
  - `VHOST_IOTLB_INVALIDATE` ⟹ `vhost_vdpa_unmap(v, iotlb, msg.iova, msg.size)`.
  - `VHOST_IOTLB_BATCH_BEGIN` ⟹ `v->batch_asid = asid`; `v->in_batch = true`.
  - `VHOST_IOTLB_BATCH_END` ⟹ if `in_batch ∧ ops->set_map`: `ops->set_map(vdpa, asid, iotlb)`; clear `in_batch`.
  - default ⟹ -EINVAL.

REQ-12: VhostVdpa::process_iotlb_update(iotlb, msg):
- Validate `msg.iova ≥ v->range.first`; `msg.size != 0`; no integer overflow; `msg.iova + msg.size - 1 ≤ v->range.last` else -EINVAL.
- Reject if overlapping mapping in `iotlb` (`vhost_iotlb_itree_first` non-NULL) (-EEXIST).
- If `vdpa->use_va` ⟹ `vhost_vdpa_va_map(v, iotlb, iova, size, uaddr, perm)`.
- Else ⟹ `vhost_vdpa_pa_map(v, iotlb, iova, size, uaddr, perm)`.

REQ-13: VhostVdpa::map(iotlb, iova, size, pa, perm, opaque):
- `vhost_iotlb_add_range_ctx(iotlb, iova, iova+size-1, pa, perm, opaque)`.
- If `ops->dma_map` ⟹ `ops->dma_map(vdpa, asid, iova, size, pa, perm, opaque)` (vendor-managed).
- Else if `ops->set_map` ⟹ if `!v->in_batch`: `ops->set_map(vdpa, asid, iotlb)` (whole-IOTLB shadow push).
- Else ⟹ `iommu_map(v->domain, iova, pa, size, perm_to_iommu_flags(perm), GFP_KERNEL_ACCOUNT)` (platform IOMMU).
- On error: `vhost_iotlb_del_range(iotlb, iova, iova+size-1)`.
- On success and `!vdpa->use_va`: `atomic64_add(PFN_DOWN(size), &dev->mm->pinned_vm)`.

REQ-14: VhostVdpa::pa_map(iotlb, iova, size, uaddr, perm) — PA-mode:
- `page_list = __get_free_page` (scratch).
- `gup_flags = FOLL_LONGTERM`; `(perm & WO) ⟹ gup_flags |= FOLL_WRITE`.
- `npages = PFN_UP(size + (iova & ~PAGE_MASK))`.
- `mmap_read_lock(dev->mm)`.
- Reject if `npages + pinned_vm > rlimit(RLIMIT_MEMLOCK)` (-ENOMEM).
- Loop `pin_user_pages(cur_base, sz2pin, gup_flags, page_list)`:
  - On short pin or error ⟹ `unpin_user_pages(page_list, pinned)`; -ENOMEM.
  - Walk pinned PFNs; on PFN-discontinuity ⟹ `vhost_vdpa_map(iotlb, iova, csize, PFN_PHYS(map_pfn), perm, NULL)`; advance `iova`.
- Final chunk: `vhost_vdpa_map(...)`.
- On any failure: unpin outstanding pages of current page_list + `vhost_vdpa_unmap(iotlb, start, size)` (which unwinds prior chunks).
- `mmap_read_unlock`.

REQ-15: VhostVdpa::va_map(iotlb, iova, size, uaddr, perm) — VA-mode (SVA):
- `mmap_read_lock(dev->mm)`.
- Loop while `size > 0`:
  - `vma = find_vma(dev->mm, uaddr)`; NULL ⟹ -EINVAL.
  - `map_size = min(size, vma->vm_end - uaddr)`.
  - Require `vma->vm_file ∧ (vm_flags & VM_SHARED) ∧ !(vm_flags & (VM_IO | VM_PFNMAP))` else skip.
  - Alloc `map_file: vdpa_map_file { offset, file = get_file(vma->vm_file) }`.
  - `vhost_vdpa_map(iotlb, map_iova, map_size, uaddr, perm, map_file)`; on fail ⟹ `fput(file)` + `kfree`.
- On any error: `vhost_vdpa_unmap(iotlb, iova, map_iova - iova)`.

REQ-16: VhostVdpa::pa_unmap(iotlb, start, last, asid):
- For each `map = vhost_iotlb_itree_first(iotlb, start, last)`:
  - `pinned = PFN_DOWN(map.size)`.
  - For each PFN: `page = pfn_to_page(pfn)`; if `WO` ⟹ `set_page_dirty_lock`; `unpin_user_page(page)`.
  - `atomic64_sub(PFN_DOWN(map.size), &dev->mm->pinned_vm)`.
  - `vhost_vdpa_general_unmap(v, map, asid)` ⟹ `ops->dma_unmap` or `iommu_unmap`.
  - `vhost_iotlb_map_free(iotlb, map)`.

REQ-17: VhostVdpa::va_unmap(iotlb, start, last, asid):
- For each map: `map_file = map->opaque`; `fput(map_file->file)`; `kfree(map_file)`; `vhost_vdpa_general_unmap`; `vhost_iotlb_map_free`.

REQ-18: VhostVdpa::setup_vq_irq(qid):
- If `!ops->get_vq_irq` ⟹ return.
- `irq = ops->get_vq_irq(vdpa, qid)`; if `< 0` ⟹ return.
- If `!vq->call_ctx.ctx` ⟹ return.
- `irq_bypass_register_producer(&vq->call_ctx.producer, vq->call_ctx.ctx, irq)` — KVM (consumer side) connects this producer to the guest IRQ injection path, bypassing host VMM.

REQ-19: VhostVdpa::handle_vq_kick(work):
- `vq = container_of(work, vhost_virtqueue, poll.work)`.
- `v = container_of(vq->dev, vhost_vdpa, vdev)`.
- `ops->kick_vq(v->vdpa, vq - v->vqs)` — forward to vDPA device.

REQ-20: VhostVdpa::set_status(statusp):
- `status = copy_from_user`.
- `status_old = ops->get_status(vdpa)`.
- Reject removing bits: `status != 0 ∧ (status_old & ~status) != 0` ⟹ -EINVAL.
- If `S_DRIVER_OK` cleared: for each VQ ⟹ `unsetup_vq_irq`.
- If `status == 0` ⟹ `_compat_vdpa_reset(v)`.
- Else ⟹ `vdpa_set_status(vdpa, status)`.
- If `S_DRIVER_OK` newly set: for each VQ ⟹ `setup_vq_irq`.

REQ-21: VhostVdpa::suspend(v) / resume(v):
- suspend: if `!(S_DRIVER_OK)` ⟹ 0; if `!ops->suspend` ⟹ -EOPNOTSUPP; else `ret = ops->suspend(vdpa)`; on success `v->suspended = true`.
- resume: requires `S_DRIVER_OK`; `ops->resume(vdpa)`; clear `v->suspended` on success.

REQ-22: VhostVdpa::set_features(featurep):
- Reject if `S_FEATURES_OK` set (-EBUSY — already negotiated).
- `features = copy_from_user`.
- `vdpa_set_features(vdpa, features)`.
- Read back `actual_features = ops->get_driver_features(vdpa)`.
- Broadcast `actual_features` to `vq.acked_features` under each `vq.mutex`.

REQ-23: VhostVdpa::set_config_call(argp):
- `fd = copy_from_user`.
- `ctx = fd == VHOST_FILE_UNBIND ? NULL : eventfd_ctx_fdget(fd)`.
- `swap(ctx, v->config_ctx)`; drop old `ctx` if non-NULL.
- `vdpa->config->set_config_cb(vdpa, &cb)` with `cb.callback = vhost_vdpa_config_cb`.

REQ-24: VhostVdpa::config_cb(private):
- `v = private`; if `v->config_ctx` ⟹ `eventfd_signal(v->config_ctx)`; return `IRQ_HANDLED`.

REQ-25: VhostVdpa::virtqueue_cb(private):
- `vq = private`; `call_ctx = vq->call_ctx.ctx`; if non-NULL ⟹ `eventfd_signal(call_ctx)`; return `IRQ_HANDLED`.

REQ-26: VhostVdpa::mmap(file, vma) — doorbell mmap:
- Require `vma.size == PAGE_SIZE`; `VM_SHARED`; not `VM_READ`; `index ≤ 65535`; `ops->get_vq_notification != NULL` else -ENOTSUPP.
- `notify = ops->get_vq_notification(vdpa, index)`.
- `notify.addr` must be PAGE_SIZE-aligned; `notify.size == vma.size` else -EINVAL/-ENOTSUPP.
- `vma.vm_page_prot = pgprot_noncached(...)`; set `VM_IO | VM_PFNMAP | VM_DONTEXPAND | VM_DONTDUMP`.
- `vma.vm_ops = &vhost_vdpa_vm_ops` (fault ⟹ `vmf_insert_pfn(vma, addr & PAGE_MASK, PFN_DOWN(notify.addr))`).

REQ-27: VhostVdpa::chr_write_iter(iocb, from):
- `vhost_chr_write_iter(&v->vdev, from)` — vhost core decodes `vhost_iotlb_msg` and calls back into `process_iotlb_msg`.

REQ-28: vhost_vdpa_driver:
- `.driver.name = "vhost_vdpa"`.
- `.probe = vhost_vdpa_probe`.
- `.remove = vhost_vdpa_remove`.
- Registered on `vdpa_bus` via `vdpa_register_driver(&vhost_vdpa_driver)` in `vhost_vdpa_init`.

REQ-29: Module init/exit:
- `alloc_chrdev_region(&vhost_vdpa_major, 0, VHOST_VDPA_DEV_MAX = 1 << MINORBITS, "vhost-vdpa")`.
- `vdpa_register_driver(&vhost_vdpa_driver)`.
- exit: `vdpa_unregister_driver`; `unregister_chrdev_region`.

REQ-30: Backend features supported:
- `VHOST_BACKEND_F_IOTLB_MSG_V2`, `F_IOTLB_BATCH`, `F_IOTLB_ASID` (always);
- `F_SUSPEND` (if `ops->suspend`);
- `F_RESUME` (if `ops->resume`);
- `F_DESC_ASID` (if `ops->get_vq_desc_group`);
- `F_IOTLB_PERSIST` (if `(!set_map ∧ !dma_map) ∨ ops->reset_map ∨ get_backend_features() & F_IOTLB_PERSIST`);
- `F_ENABLE_AFTER_DRIVER_OK`.

## Acceptance Criteria

- [ ] AC-1: `vhost_vdpa_probe`: binding a `vdpa_device` creates `/dev/vhost-vdpa-N` (N from `vhost_vdpa_ida`).
- [ ] AC-2: `open` is exclusive: second concurrent open returns -EBUSY.
- [ ] AC-3: `VHOST_VDPA_GET_DEVICE_ID` returns `ops->get_device_id(vdpa)` byte-identical.
- [ ] AC-4: `VHOST_GET_FEATURES` returns `ops->get_device_features(vdpa)`; `VHOST_SET_FEATURES` rejected after `S_FEATURES_OK` set.
- [ ] AC-5: `VHOST_SET_VRING_ADDR/_NUM/_BASE/_KICK/_CALL` bridge to `ops->set_vq_address/_num/_state/handle_kick/_cb`.
- [ ] AC-6: `VHOST_IOTLB_UPDATE` via `write_iter` produces `iommu_map` (platform-IOMMU mode) or `ops->dma_map` (device-managed mode) or queued `ops->set_map` (batched mode).
- [ ] AC-7: `VHOST_IOTLB_BATCH_BEGIN ... UPDATE* ... BATCH_END` issues one `ops->set_map` at end (batched-mode device).
- [ ] AC-8: `VHOST_IOTLB_INVALIDATE` unpins user pages (`unpin_user_page` + `set_page_dirty_lock` for WO mappings) and updates `dev->mm->pinned_vm`.
- [ ] AC-9: `VHOST_VDPA_SUSPEND`/`_RESUME` toggles `ops->suspend`/`resume`; `v->suspended` reflects.
- [ ] AC-10: `VHOST_VDPA_SET_STATUS(S_DRIVER_OK)` post: each VQ has `setup_vq_irq` called and `irq_bypass_producer` registered.
- [ ] AC-11: `VHOST_VDPA_SET_STATUS(0)` triggers `_compat_vdpa_reset` and clears `v->suspended`.
- [ ] AC-12: vDPA device hot-remove (`vhost_vdpa_remove`): waits for fd closed via `completion`; `ida_free` minor.
- [ ] AC-13: ASID isolation: `VHOST_IOTLB_UPDATE` to ASID X is invisible to ASID Y; `VHOST_VDPA_SET_GROUP_ASID` rejected if `S_DRIVER_OK`.
- [ ] AC-14: Pinned-VM accounting: `dev->mm->pinned_vm` incremented on map, decremented on unmap; never exceeds `RLIMIT_MEMLOCK`.
- [ ] AC-15: `vhost_vdpa_mmap` of doorbell page returns aligned non-cached PFN-only VMA; only `qid ≤ 65535`.
- [ ] AC-16: `VHOST_VDPA_GET_IOVA_RANGE` returns `(first,last)` matching `ops->get_iova_range` or IOMMU geometry.
- [ ] AC-17: `VHOST_SET_BACKEND_FEATURES`: invalid combos rejected (`F_DESC_ASID` without `F_IOTLB_ASID`, `F_SUSPEND` without `can_suspend`, etc.).
- [ ] AC-18: VA-mode (`vdpa->use_va`): mappings track `vma->vm_file` via `get_file`; release via `fput` + `kfree(map_file)`.

## Architecture

```
struct VhostVdpa {
  vdev: VhostDev,
  domain: Option<*IommuDomain>,
  vqs: Vec<VhostVirtqueue>,
  completion: Completion,
  vdpa: *VdpaDevice,
  as_buckets: [HlistHead; VHOST_VDPA_IOTLB_BUCKETS],
  dev: Device,
  cdev: Cdev,
  opened: AtomicI32,
  nvqs: u32,
  virtio_id: i32,
  minor: i32,
  config_ctx: Option<*EventfdCtx>,
  in_batch: i32,
  range: VdpaIovaRange,
  batch_asid: u32,
  suspended: bool,
}

struct VhostVdpaAs {
  hash_link: HlistNode,
  iotlb: VhostIotlb,
  id: u32,
}
```

`VhostVdpa::open(file)`:
1. `v = container_of(inode.i_cdev, VhostVdpa, cdev)`.
2. `opened = atomic_cmpxchg(&v.opened, 0, 1)`; non-zero ⟹ -EBUSY.
3. `nvqs = v.nvqs`.
4. `vhost_vdpa_reset(v)` — `v.in_batch = 0`, `_compat_vdpa_reset` ⟹ `vdpa_reset(v.vdpa, flags)` where `flags = (acked F_IOTLB_PERSIST ? 0 : VDPA_RESET_F_CLEAN_MAP)`.
5. Alloc `vqs = kmalloc_array(nvqs)`; for each VQ: `vqs[i] = &v.vqs[i]; vqs[i].handle_kick = handle_vq_kick; vqs[i].call_ctx.ctx = NULL`.
6. `vhost_dev_init(&v.vdev, vqs, nvqs, 0, 0, 0, false, vhost_vdpa_process_iotlb_msg)`.
7. `vhost_vdpa_alloc_domain(v)`.
8. `vhost_vdpa_set_iova_range(v)`.
9. `file.private_data = v`; return 0.
10. On error: `vhost_vdpa_cleanup(v)`; `atomic_dec(&v.opened)`.

`VhostVdpa::ioctl(file, cmd, arg)` — see REQ-9.

`VhostVdpa::vring_ioctl(cmd, argp)` — see REQ-10.

`VhostVdpa::set_status(statusp)`:
1. `copy_from_user(&status, statusp)`.
2. `status_old = ops.get_status(vdpa)`.
3. If `status != 0 ∧ (status_old & ~status) != 0` ⟹ -EINVAL.
4. If `(status_old & DRIVER_OK) ∧ !(status & DRIVER_OK)`: for each VQ ⟹ `unsetup_vq_irq(i)`.
5. If `status == 0`: `_compat_vdpa_reset(v)`.
6. Else: `vdpa_set_status(vdpa, status)`.
7. If `(status & DRIVER_OK) ∧ !(status_old & DRIVER_OK)`: for each VQ ⟹ `setup_vq_irq(i)`.

`VhostVdpa::process_iotlb_msg(dev, asid, msg)`:
1. lock `dev.mutex`.
2. `vhost_dev_check_owner(dev)` — must be in owner-set state.
3. If `msg.type ∈ {UPDATE, BATCH_BEGIN}`: `as = vhost_vdpa_find_alloc_as(v, asid)`; `iotlb = &as.iotlb`.
4. Else: `iotlb = asid_to_iotlb(v, asid)`.
5. If `in_batch ∧ batch_asid != asid` ⟹ -EINVAL.
6. If `!iotlb` ⟹ -EINVAL.
7. Switch:
   - `UPDATE` ⟹ `vhost_vdpa_process_iotlb_update(v, iotlb, msg)`.
   - `INVALIDATE` ⟹ `vhost_vdpa_unmap(v, iotlb, msg.iova, msg.size)`.
   - `BATCH_BEGIN` ⟹ `batch_asid = asid; in_batch = true`.
   - `BATCH_END` ⟹ if `in_batch ∧ ops.set_map`: `ops.set_map(vdpa, asid, iotlb)`; `in_batch = false`.
   - default ⟹ -EINVAL.
8. unlock.

`VhostVdpa::process_iotlb_update(iotlb, msg)`:
1. Validate aperture: `msg.iova ≥ v.range.first ∧ msg.size ≠ 0 ∧ msg.iova ≤ U64_MAX - msg.size + 1 ∧ msg.iova + msg.size - 1 ≤ v.range.last`.
2. Reject overlap: `vhost_iotlb_itree_first(iotlb, msg.iova, msg.iova + msg.size - 1) != NULL` ⟹ -EEXIST.
3. Dispatch: `vdpa.use_va ⟹ va_map(v, iotlb, iova, size, uaddr, perm)`; else `pa_map(...)`.

`VhostVdpa::map(iotlb, iova, size, pa, perm, opaque)`:
1. `vhost_iotlb_add_range_ctx(iotlb, iova, iova+size-1, pa, perm, opaque)`.
2. Branch by ops:
   - `ops.dma_map` ⟹ `ops.dma_map(vdpa, asid, iova, size, pa, perm, opaque)`.
   - `ops.set_map` ⟹ if `!v.in_batch`: `ops.set_map(vdpa, asid, iotlb)` (immediate); else defer to BATCH_END.
   - else ⟹ `iommu_map(v.domain, iova, pa, size, perm_to_iommu_flags(perm), GFP_KERNEL_ACCOUNT)`.
3. On error: `vhost_iotlb_del_range`; return error.
4. If `!vdpa.use_va`: `atomic64_add(PFN_DOWN(size), &dev.mm.pinned_vm)`.

`VhostVdpa::pa_map(iotlb, iova, size, uaddr, perm)` — see REQ-14.

`VhostVdpa::va_map(iotlb, iova, size, uaddr, perm)` — see REQ-15.

`VhostVdpa::unmap(iotlb, iova, size)`:
1. `vhost_vdpa_iotlb_unmap(v, iotlb, iova, iova+size-1, asid)` — dispatches to `va_unmap` (use_va) or `pa_unmap`.
2. If `ops.set_map ∧ !v.in_batch`: `ops.set_map(vdpa, asid, iotlb)` to push shadow.

`VhostVdpa::setup_vq_irq(qid)` — see REQ-18.

`VhostVdpa::probe(vdpa)` — see REQ-3.

`VhostVdpa::remove(vdpa)` — see REQ-4.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `iotlb_msg_aperture_bounded` | INVARIANT | per-process_iotlb_update: `msg.iova + msg.size - 1 ≤ v.range.last`. |
| `iotlb_no_overlap` | INVARIANT | per-process_iotlb_update: `vhost_iotlb_itree_first` returns NULL before add. |
| `asid_index_bounded` | INVARIANT | per-alloc_as: `asid < v.vdpa.nas`. |
| `vring_idx_bounded` | INVARIANT | per-vring_ioctl: `idx < v.nvqs` (uses `array_index_nospec`). |
| `pin_user_pages_rlimit` | INVARIANT | per-pa_map: `npages + pinned_vm ≤ rlimit(RLIMIT_MEMLOCK)`. |
| `gup_unwind_on_short_pin` | INVARIANT | per-pa_map: short pin ⟹ `unpin_user_pages(page_list, pinned)` + return -ENOMEM. |
| `set_vring_addr_requires_quiesce` | INVARIANT | per-vring_ioctl SET_VRING_ADDR: `S_DRIVER_OK ⟹ v.suspended` else -EINVAL. |
| `set_features_pre_features_ok` | INVARIANT | per-set_features: `!(S_FEATURES_OK)` else -EBUSY. |
| `open_exclusive` | INVARIANT | per-open: `atomic_cmpxchg(opened, 0, 1) == 0` else -EBUSY. |
| `remove_waits_release` | INVARIANT | per-remove: waits for `complete(&v.completion)` issued by release. |
| `batch_asid_consistent` | INVARIANT | per-process_iotlb_msg: `in_batch ⟹ msg.asid == batch_asid`. |
| `va_map_file_refcount_paired` | INVARIANT | per-va_map / va_unmap: `get_file` matched by `fput`. |
| `pa_unmap_unpin_with_dirty` | INVARIANT | per-pa_unmap: WO mappings ⟹ `set_page_dirty_lock` before `unpin_user_page`. |
| `domain_alloc_requires_cache_coherent` | INVARIANT | per-alloc_domain: `device_iommu_capable(IOMMU_CAP_CACHE_COHERENCY)` else -ENOTSUPP. |
| `desc_group_requires_iotlb_asid` | INVARIANT | per-SET_BACKEND_FEATURES: `F_DESC_ASID ⟹ F_IOTLB_ASID`. |

### Layer 2: TLA+

`drivers/vhost/vdpa.tla`:
- Per-fd lifecycle: probe ⟶ open(exclusive) ⟶ {ioctl,write_iter}* ⟶ release ⟶ remove.
- Per-IOTLB lifecycle: write_iter ⟶ process_iotlb_msg ⟶ {BATCH_BEGIN ⟶ UPDATE* ⟶ BATCH_END} | UPDATE | INVALIDATE.
- Per-virtio-status FSM: 0 ⟶ ACKNOWLEDGE ⟶ DRIVER ⟶ FEATURES_OK ⟶ DRIVER_OK ⟶ {DEVICE_NEEDS_RESET | FAILED} ⟶ 0.
- Per-suspend: DRIVER_OK ⟶ SUSPEND(suspended=true) ⟶ RESUME(suspended=false).
- Properties:
  - `safety_one_open` — at most one open fd at any time per `v`.
  - `safety_no_features_change_after_features_ok` — `S_FEATURES_OK ⟹ set_features rejected`.
  - `safety_iotlb_no_overlap` — ASID's IOTLB is a partial function on IOVA.
  - `safety_pinned_within_rlimit` — `pinned_vm ≤ RLIMIT_MEMLOCK` invariant.
  - `safety_irq_bypass_paired` — `setup_vq_irq` matched by `unsetup_vq_irq` exactly per fd-lifetime + status transitions.
  - `safety_remove_waits_release` — `remove` blocks until fd's `release` has signalled `completion`.
  - `liveness_kick_reaches_device` — every guest kick eventually invokes `ops.kick_vq` on the vDPA device.
  - `liveness_iotlb_update_visible` — after `UPDATE` (non-batch) or `BATCH_END` (batch), `ops.set_map` reflects the IOTLB.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VhostVdpa::open` post: ret == 0 ⟹ `opened == 1 ∧ vdev.vqs != NULL ∧ alloc_domain succeeded ∧ range set` | `VhostVdpa::open` |
| `VhostVdpa::release` post: `opened == 0 ∧ vdev cleaned ∧ all ASIDs removed ∧ config_ctx put ∧ completion signalled` | `VhostVdpa::release` |
| `VhostVdpa::set_status` post: `DRIVER_OK` newly set ⟹ each VQ has `setup_vq_irq` called; cleared ⟹ each VQ has `unsetup_vq_irq` called | `VhostVdpa::set_status` |
| `VhostVdpa::process_iotlb_msg` post: UPDATE/INVALIDATE/BATCH_*/default branches mutually exclusive | `VhostVdpa::process_iotlb_msg` |
| `VhostVdpa::process_iotlb_update` post: ret == 0 ⟹ `iotlb` contains the mapping ∧ device backing has been pushed (dma_map / set_map / iommu_map) | `VhostVdpa::process_iotlb_update` |
| `VhostVdpa::pa_map` post: ret == 0 ⟹ every page in `[uaddr, uaddr+size)` pinned; ret ≠ 0 ⟹ no leaks | `VhostVdpa::pa_map` |
| `VhostVdpa::pa_unmap` post: every map in `[start,last]` removed; `pinned_vm` decremented by `PFN_DOWN(size)` per map | `VhostVdpa::pa_unmap` |
| `VhostVdpa::set_features` post: ret == 0 ⟹ each VQ's `acked_features == ops.get_driver_features(vdpa)` | `VhostVdpa::set_features` |
| `VhostVdpa::vring_ioctl SET_VRING_ADDR` post: `S_DRIVER_OK ∧ !suspended` ⟹ -EINVAL; success ⟹ `ops.set_vq_address` called | `VhostVdpa::vring_ioctl` |
| `VhostVdpa::probe` post: ret == 0 ⟹ `cdev_device_add` succeeded ∧ `vdpa_set_drvdata(vdpa, v)` ∧ all `as[]` buckets initialized | `VhostVdpa::probe` |
| `VhostVdpa::remove` post: `cdev_device_del` issued ∧ waited for `completion` ∧ `put_device(&v.dev)` issued | `VhostVdpa::remove` |
| `VhostVdpa::alloc_domain` post: `ops.set_map ∨ ops.dma_map ⟹ v.domain == NULL`; else `v.domain` allocated + attached | `VhostVdpa::alloc_domain` |

### Layer 4: Verus/Creusot functional

`Per-vhost-vdpa control flow → vDPA control ops → device IOMMU mapping → guest virtio-data-path` semantic equivalence: per-Documentation/networking/device_drivers/ethernet/intel/idpf-vdpa.rst + Documentation/userspace-api/vduse.rst + virtio 1.x spec.

`Per-IOTLB-message channel (write_iter ⟶ vhost_chr_write_iter ⟶ process_iotlb_msg)` semantic equivalence: per-`include/uapi/linux/vhost_types.h` `struct vhost_iotlb_msg` wire format + `VHOST_BACKEND_F_IOTLB_MSG_V2`/`F_IOTLB_BATCH`/`F_IOTLB_ASID` negotiation.

## Hardening

(Inherits row-1 features from `drivers/vhost/00-overview.md` § Hardening.)

vhost-vdpa reinforcement:

- **Per-`CAP_SYS_ADMIN` on `/dev/vhost-vdpa-N`** — defense against per-unprivileged-DMA-mapping-attack (any user with this fd can pin host pages and program device DMA).
- **Per-`atomic_cmpxchg(opened, 0, 1)` exclusive open** — defense against per-concurrent-fd state corruption.
- **Per-`pin_user_pages(FOLL_LONGTERM)` + `RLIMIT_MEMLOCK` check** — defense against per-memlock-DoS (cannot pin more than userspace's memlock budget).
- **Per-IOVA aperture bounds checked (`v.range.first` / `v.range.last`)** — defense against per-out-of-aperture DMA into unintended device memory.
- **Per-IOTLB overlap rejection (`vhost_iotlb_itree_first` non-NULL ⟹ -EEXIST)** — defense against per-DMA-aliasing (two IOVAs ⟶ same PA).
- **Per-ASID isolation enforced via separate `vhost_iotlb` per `vhost_vdpa_as`** — defense against per-cross-ASID DMA.
- **Per-`array_index_nospec(idx, nvqs)` in vring_ioctl** — defense against per-Spectre-v1 speculative array OOB.
- **Per-`SET_VRING_ADDR` rejected when `S_DRIVER_OK ∧ !suspended`** — defense against per-live-vring relocation race.
- **Per-`SET_FEATURES` rejected after `S_FEATURES_OK`** — defense against per-feature-renegotiation.
- **Per-`SET_GROUP_ASID` rejected when `S_DRIVER_OK`** — defense against per-running-device-ASID-swap (would orphan in-flight descriptors).
- **Per-`device_iommu_capable(IOMMU_CAP_CACHE_COHERENCY)` requirement in `alloc_domain`** — defense against per-non-cache-coherent-DMA seeing stale data.
- **Per-`irq_bypass_register_producer` paired with `_unregister_producer` across DRIVER_OK transitions** — defense against per-stale-irq-injection after device tear-down.
- **Per-completion-wait in `remove` until `release` signals** — defense against per-vDPA-device-yanked while fd open (would UAF `vdpa` pointer).
- **Per-`set_page_dirty_lock` before `unpin_user_page` for WO mappings** — defense against per-lost-write-during-DMA-unmap.
- **Per-`fput(map_file.file)` + `kfree(map_file)` on VA-mode unmap** — defense against per-vm-file-refcount leak.
- **Per-mmap doorbell: `VM_IO | VM_PFNMAP | VM_DONTEXPAND | VM_DONTDUMP` + `pgprot_noncached` + `qid ≤ 65535`** — defense against per-coredump-leak-of-MMIO + per-incoherent-write to device register.
- **Per-`vhost_dev_check_owner` gate on IOTLB messages** — defense against per-non-owner DMA programming.
- **Per-backend-feature bit dependency checks (`F_DESC_ASID ⟹ F_IOTLB_ASID`; `F_SUSPEND ⟹ can_suspend`; ...)** — defense against per-inconsistent-negotiation.

## Open Questions

- Per-VA-mode (`use_va = true`) is currently exercised by vduse and a few SVA-capable smartNIC vendor drivers — Rookery should ensure the `vdpa_map_file` opaque cookie lifetime matches mainline exactly (notably `get_file` is taken at map time and released at unmap time; vDPA vendor driver may dereference between).
- Per-`F_ENABLE_AFTER_DRIVER_OK` backend feature changes the order of `SET_VRING_ENABLE` vs `SET_STATUS(DRIVER_OK)`. Verify our state machine accepts both orderings as mainline does.

## Out of Scope

- vDPA bus core + vendor drivers (`drivers/vdpa/`) — covered separately under `drivers/vdpa/00-overview.md` if expanded.
- `drivers/vhost/vhost.c` (vhost framework: `vhost_dev`, `vhost_virtqueue`, IOTLB-message decoder `vhost_chr_write_iter`) — covered under `drivers/vhost/core.md` Tier-3.
- `drivers/vhost/iotlb.c` (`vhost_iotlb` interval tree) — covered under `drivers/vhost/iotlb.md` Tier-3.
- `drivers/vhost/vdpa_user/` (vduse — userspace vDPA backend) — covered under `drivers/vhost/vduse.md` Tier-3.
- KVM-side `irq_bypass_consumer` (KVM PIC/IOAPIC/POSTED_INTR wiring) — covered under `virt/kvm/` Tier-3.
- Implementation code.
