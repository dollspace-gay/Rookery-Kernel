# Tier-3: drivers/virtio/virtio_pci_common.c — VirtIO PCI transport (common)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/virtio/00-overview.md
upstream-paths:
  - drivers/virtio/virtio_pci_common.c (~862 lines)
  - drivers/virtio/virtio_pci_common.h
  - drivers/virtio/virtio_pci_modern.c (probe/remove dispatched here)
  - drivers/virtio/virtio_pci_legacy.c (probe/remove dispatched here)
  - include/uapi/linux/virtio_pci.h (VIRTIO_PCI_ISR_*, VIRTIO_MSI_NO_VECTOR, VIRTIO_PCI_CAP_*)
  - include/uapi/linux/virtio_config.h (VIRTIO_CONFIG_S_*)
-->

## Summary

`virtio_pci_common.c` is the transport-agnostic core of the virtio-PCI bus driver — shared between **modern** (virtio-1.x using PCI capabilities) and **legacy** (virtio-0.95 using BAR0 I/O) transports. It owns: per-PCI-function probe/remove (`virtio_pci_probe` / `virtio_pci_remove`), per-virtqueue setup over MSI-X or INTx, per-MSI-X vector allocation (one-per-VQ / shared-slow / shared / INTx fallback), per-config-change + per-vring interrupt handlers (`vp_config_changed` / `vp_vring_interrupt` / `vp_interrupt`), per-ISR-byte read for INTx (clearing-on-read), per-SR-IOV configure, per-PCI-error reset prepare/done, per-PM suspend/resume/freeze/thaw/restore, per-VF→PF lookup. Per-device dispatch: `force_legacy=1` ⟹ try legacy first then modern; default ⟹ try modern first then legacy. Per-VirtIO 1.x capability walk happens in `virtio_pci_modern.c` (via `virtio_pci_modern_probe`); legacy I/O probe in `virtio_pci_legacy.c`. Critical for: every cloud-VM device (virtio-net, virtio-blk, virtio-scsi, virtio-gpu, virtio-fs, virtio-balloon, virtio-mem, virtio-rng, virtio-crypto, virtio-vsock, virtio-console).

This Tier-3 covers `drivers/virtio/virtio_pci_common.c` (~862 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct virtio_pci_device` | per-vp_dev (pci_dev, vdev, isr, msix state, vqs, admin_vq, ops) | `VirtioPciDevice` |
| `struct virtio_pci_vq_info` | per-VQ (vq, msix_vector, node) | `VirtioPciVqInfo` |
| `struct virtio_pci_admin_vq` | per-admin-VQ (vq_index, info, name) | `VirtioPciAdminVq` |
| `to_vp_device(vdev)` | per-cast container_of | `VirtioPciDevice::from_vdev` |
| `virtio_pci_probe()` | per-PCI-function probe | `VirtioPci::probe` |
| `virtio_pci_remove()` | per-PCI-function remove | `VirtioPci::remove` |
| `virtio_pci_release_dev()` | per-virtio_device kref release | `VirtioPci::release_dev` |
| `virtio_pci_modern_probe()` | per-modern (virtio-1.x cap-walk) probe | `VirtioPciModern::probe` |
| `virtio_pci_legacy_probe()` | per-legacy (BAR0 I/O) probe | `VirtioPciLegacy::probe` |
| `virtio_pci_modern_remove()` / `virtio_pci_legacy_remove()` | per-variant remove | `VirtioPciModern::remove` / `VirtioPciLegacy::remove` |
| `vp_notify()` | per-VQ kick (write index → notification reg) | `VirtioPci::notify_vq` |
| `vp_synchronize_vectors()` | per-vdev sync all IRQ handlers | `VirtioPci::synchronize_vectors` |
| `vp_config_changed()` | per-MSI-X config-vector ISR | `VirtioPci::config_changed_irq` |
| `vp_vring_interrupt()` | per-MSI-X shared-VQ ISR | `VirtioPci::vring_interrupt_irq` |
| `vp_vring_slow_path_interrupt()` | per-slow-path-VQ shared dispatch | `VirtioPci::vring_slow_path_irq` |
| `vp_interrupt()` | per-INTx legacy ISR (read+clear ISR byte) | `VirtioPci::intx_irq` |
| `vp_request_msix_vectors()` | per-vdev allocate MSI-X vectors + config IRQ | `VirtioPci::request_msix_vectors` |
| `vp_setup_vq()` | per-VQ alloc info + dispatch to modern/legacy setup_vq | `VirtioPci::setup_vq` |
| `vp_del_vq()` / `vp_del_vqs()` | per-VQ / per-vdev cleanup | `VirtioPci::del_vq` / `del_vqs` |
| `vp_find_one_vq_msix()` | per-VQ MSI-X vector assignment per policy | `VirtioPci::find_one_vq_msix` |
| `vp_find_vqs_msix()` | per-vdev MSI-X find-vqs (per-policy) | `VirtioPci::find_vqs_msix` |
| `vp_find_vqs_intx()` | per-vdev INTx find-vqs (fallback) | `VirtioPci::find_vqs_intx` |
| `vp_find_vqs()` | per-vdev top-level find_vqs (policy ladder) | `VirtioPci::find_vqs` |
| `vp_set_vq_affinity()` / `vp_get_vq_affinity()` | per-VQ CPU affinity | `VirtioPci::set_vq_affinity` / `get_vq_affinity` |
| `vp_is_avq()` | per-vdev is-this-the-admin-VQ predicate | `VirtioPci::is_admin_vq` |
| `vp_is_slow_path_vector()` | per-MSI-X-vec slow-path predicate | `VirtioPci::is_slow_path_vector` |
| `vp_bus_name()` | per-vdev `pci_name()` | `VirtioPci::bus_name` |
| `virtio_pci_sriov_configure()` | per-PCI-function SR-IOV VF enable/disable | `VirtioPci::sriov_configure` |
| `virtio_pci_reset_prepare()` / `_done()` | per-PCI-error reset hooks | `VirtioPci::reset_prepare` / `reset_done` |
| `virtio_pci_freeze()` / `_restore()` / `_suspend()` / `_resume()` | per-PM hooks | `VirtioPci::pm_*` |
| `vp_supports_pm_no_reset()` | per-PCI PM-no-soft-reset detect | `VirtioPci::pm_no_soft_reset` |
| `virtio_pci_vf_get_pf_dev()` | per-VF lookup-parent-PF vdev | `VirtioPci::vf_get_pf_dev` |
| `virtio_pci_id_table[]` | per-PCI-driver `{0x1AF4, PCI_ANY_ID}` (Red Hat / Qumranet) | shared |
| `force_legacy` | module-param: per-bus force-legacy mode | shared |

## Compatibility contract

REQ-1: struct virtio_pci_device (per-bus device state, allocated by `virtio_pci_probe`):
- vdev: per-virtio_device embedded (registered via `register_virtio_device`).
- pci_dev: per-backing-PCI-function.
- isr: per-modern-`__iomem *u8` ISR-byte (legacy: in BAR0; modern: per-isr-capability).
- ioaddr: per-legacy BAR0 base.
- vqs: per-array of `struct virtio_pci_vq_info *` indexed by virtqueue index.
- admin_vq: per-admin-VQ (virtio-1.2 VIRTIO_F_ADMIN_VQ; SR-IOV management).
- virtqueues / slow_virtqueues: per-list-head — list of fast-path / slow-path VQs (admin-VQ on slow path).
- lock: per-spinlock guarding virtqueues/slow_virtqueues.
- msix_enabled / intx_enabled: per-flag mutually-exclusive.
- per_vq_vectors: per-flag — one MSI-X vector per VQ.
- msix_vectors: per-count total vectors.
- msix_used_vectors: per-count assigned (config-vec at index 0; VQ-vecs after).
- msix_names: per-vector device-name string array (for `request_irq` name).
- msix_affinity_masks: per-vector cpumask array.
- is_legacy: per-flag — which probe variant succeeded.
- setup_vq / del_vq / config_vector / avq_index: per-fn-ptr filled by modern/legacy probe.

REQ-2: virtio_pci_probe(pci_dev, id):
- /* allocate */
- vp_dev = kzalloc(sizeof *vp_dev, GFP_KERNEL).
- if !vp_dev: return -ENOMEM.
- pci_set_drvdata(pci_dev, vp_dev).
- vp_dev.vdev.dev.parent = &pci_dev.dev.
- vp_dev.vdev.dev.release = virtio_pci_release_dev.
- vp_dev.pci_dev = pci_dev.
- INIT_LIST_HEAD(&vp_dev.virtqueues / .slow_virtqueues).
- spin_lock_init(&vp_dev.lock).
- /* enable */
- rc = pci_enable_device(pci_dev); if rc: goto err_enable_device.
- /* probe variant */
- if force_legacy:
  - rc = virtio_pci_legacy_probe(vp_dev).
  - if rc == -ENODEV ∨ -ENOMEM: rc = virtio_pci_modern_probe(vp_dev).
  - if rc: goto err_probe.
- else:
  - rc = virtio_pci_modern_probe(vp_dev).
  - if rc == -ENODEV: rc = virtio_pci_legacy_probe(vp_dev).
  - if rc: goto err_probe.
- pci_set_master(pci_dev).
- rc = register_virtio_device(&vp_dev.vdev); reg_dev = vp_dev.
- if rc: goto err_register.
- return 0.
- /* error paths */
- err_register: (legacy ? legacy_remove : modern_remove)(vp_dev).
- err_probe: pci_disable_device(pci_dev).
- err_enable_device: if reg_dev: put_device(&vp_dev.vdev.dev) else: kfree(vp_dev).
- return rc.

REQ-3: virtio_pci_remove(pci_dev):
- dev = get_device(&vp_dev.vdev.dev).
- /* surprise-removal: break to abort upper-layer ops */
- if !pci_device_is_present(pci_dev): virtio_break_device(&vp_dev.vdev).
- pci_disable_sriov(pci_dev).
- unregister_virtio_device(&vp_dev.vdev).
- (is_legacy ? legacy_remove : modern_remove)(vp_dev).
- pci_disable_device(pci_dev).
- put_device(dev).

REQ-4: vp_notify(vq):
- /* write VQ index to notification register (driver-supplied iomem ptr in vq.priv) */
- iowrite16(vq.index, (void __iomem *)vq.priv).
- return true.

REQ-5: vp_synchronize_vectors(vdev):
- if vp_dev.intx_enabled: synchronize_irq(vp_dev.pci_dev.irq).
- for i in 0..vp_dev.msix_vectors: synchronize_irq(pci_irq_vector(vp_dev.pci_dev, i)).

REQ-6: vp_vring_interrupt(irq, opaque) — MSI-X shared-VQ ISR:
- spin_lock_irqsave(&vp_dev.lock, flags).
- for info in vp_dev.virtqueues: if vring_interrupt(irq, info.vq) == IRQ_HANDLED: ret = IRQ_HANDLED.
- spin_unlock_irqrestore.
- return ret.

REQ-7: vp_vring_slow_path_interrupt(irq, vp_dev) — slow-path (admin-VQ) on config-vec ISR:
- spin_lock_irqsave(&vp_dev.lock, flags).
- for info in vp_dev.slow_virtqueues: vring_interrupt(irq, info.vq).
- spin_unlock_irqrestore.

REQ-8: vp_config_changed(irq, opaque):
- virtio_config_changed(&vp_dev.vdev).
- vp_vring_slow_path_interrupt(irq, vp_dev).
- return IRQ_HANDLED.

REQ-9: vp_interrupt(irq, opaque) — INTx legacy ISR:
- /* read ISR byte clears it; very important to save the value */
- isr = ioread8(vp_dev.isr).
- if !isr: return IRQ_NONE.   /* not us */
- if isr & VIRTIO_PCI_ISR_CONFIG: vp_config_changed(irq, opaque).
- return vp_vring_interrupt(irq, opaque).

REQ-10: vp_request_msix_vectors(vdev, nvectors, per_vq_vectors, desc):
- vp_dev.msix_vectors = nvectors.
- vp_dev.msix_names = kmalloc(nvectors entries).
- vp_dev.msix_affinity_masks = kzalloc(nvectors entries); alloc_cpumask_var per entry.
- if !per_vq_vectors: desc = NULL.
- flags = PCI_IRQ_MSIX; if desc: flags |= PCI_IRQ_AFFINITY; desc.pre_vectors++ /* config-vec is pre-vector */.
- err = pci_alloc_irq_vectors_affinity(pci_dev, nvectors, nvectors, flags, desc).
- if err < 0: goto error.
- vp_dev.msix_enabled = 1.
- /* config vector */
- v = vp_dev.msix_used_vectors (= 0).
- snprintf(msix_names[v], "%s-config", dev_name).
- request_irq(pci_irq_vector(pci_dev, v), vp_config_changed, 0, msix_names[v], vp_dev).
- ++msix_used_vectors.
- /* program transport with config vector */
- v = vp_dev.config_vector(vp_dev, v).
- if v == VIRTIO_MSI_NO_VECTOR: return -EBUSY.
- /* shared-VQ vector if !per_vq_vectors */
- if !per_vq_vectors:
  - v = msix_used_vectors.
  - snprintf(msix_names[v], "%s-virtqueues", dev_name).
  - request_irq(pci_irq_vector(pci_dev, v), vp_vring_interrupt, 0, msix_names[v], vp_dev).
  - ++msix_used_vectors.
- return 0.

REQ-11: enum vp_vq_vector_policy:
- VP_VQ_VECTOR_POLICY_EACH: per-VQ a dedicated MSI-X vector.
- VP_VQ_VECTOR_POLICY_SHARED_SLOW: per-VQ dedicated except slow-path which shares config-vec.
- VP_VQ_VECTOR_POLICY_SHARED: per-vdev 2 vectors total (config + shared-VQ).

REQ-12: vp_find_vqs(vdev, nvqs, vqs, vqs_info, desc) — policy ladder:
- /* L1: MSI-X EACH */ err = vp_find_vqs_msix(..., VP_VQ_VECTOR_POLICY_EACH, desc); if !err: return 0.
- /* L2: MSI-X SHARED_SLOW */ err = vp_find_vqs_msix(..., SHARED_SLOW, desc); if !err: return 0.
- /* L3: MSI-X SHARED */ err = vp_find_vqs_msix(..., SHARED, desc); if !err: return 0.
- /* No IRQ? Give up */ if !pci_dev.irq: return err.
- /* L4: INTx */ return vp_find_vqs_intx(vdev, nvqs, vqs, vqs_info).

REQ-13: vp_find_vqs_msix(vdev, nvqs, vqs, vqs_info, vector_policy, desc):
- vp_dev.vqs = kzalloc(nvqs entries).
- if vp_dev.avq_index: vp_dev.avq_index(vdev, &avq.vq_index, &avq_num) /* admin-VQ optional */.
- per_vq_vectors = (vector_policy != SHARED).
- if per_vq_vectors:
  - nvectors = 1 (config) + count(VQs with name ∧ callback).
  - if avq_num ∧ vector_policy == EACH: ++nvectors.
- else: nvectors = 2 (config + shared-VQ).
- err = vp_request_msix_vectors(vdev, nvectors, per_vq_vectors, desc).
- vp_dev.per_vq_vectors = per_vq_vectors.
- allocated_vectors = vp_dev.msix_used_vectors.
- for i in 0..nvqs: vqs[i] = vp_find_one_vq_msix(vdev, queue_idx++, vqi.callback, vqi.name, vqi.ctx, slow_path=false, &allocated_vectors, vector_policy, &vp_dev.vqs[i]).
- if avq_num: sprintf(avq.name, "avq.%u", avq.vq_index); vp_find_one_vq_msix(... slow_path=true ...).
- on err: vp_del_vqs(vdev); return err.

REQ-14: vp_find_one_vq_msix(vdev, queue_idx, cb, name, ctx, slow_path, &allocated, policy, &info):
- /* pick msix_vec */
- if !callback: msix_vec = VIRTIO_MSI_NO_VECTOR.
- else if policy == EACH ∨ (policy == SHARED_SLOW ∧ !slow_path): msix_vec = (*allocated_vectors)++.
- else if policy != EACH ∧ slow_path: msix_vec = VP_MSIX_CONFIG_VECTOR.
- else: msix_vec = VP_MSIX_VQ_VECTOR.
- vq = vp_setup_vq(...).
- if policy == SHARED ∨ msix_vec == VIRTIO_MSI_NO_VECTOR ∨ is_slow_path_vector(msix_vec): return vq /* shares an existing IRQ */.
- /* dedicated per-VQ IRQ */
- snprintf(msix_names[msix_vec], "%s-%s", dev_name, name).
- request_irq(pci_irq_vector(pci_dev, msix_vec), vring_interrupt, 0, msix_names[msix_vec], vq).
- on err: vp_del_vq(vq, *p_info); return ERR_PTR(err).

REQ-15: vp_setup_vq(vdev, index, callback, name, ctx, msix_vec, &p_info):
- info = kmalloc; if !info: return ERR_PTR(-ENOMEM).
- vq = vp_dev.setup_vq(vp_dev, info, index, callback, name, ctx, msix_vec) /* modern or legacy */.
- if IS_ERR(vq): kfree(info); return vq.
- info.vq = vq.
- if callback:
  - spin_lock_irqsave(&vp_dev.lock, flags).
  - if !vp_is_slow_path_vector(msix_vec): list_add(&info.node, &vp_dev.virtqueues).
  - else: list_add(&info.node, &vp_dev.slow_virtqueues).
  - spin_unlock_irqrestore.
- else: INIT_LIST_HEAD(&info.node).
- *p_info = info; return vq.

REQ-16: vp_del_vqs(vdev):
- /* per-VQ */
- list_for_each_entry_safe(vq, n, &vdev.vqs, list):
  - info = is_avq(vdev, vq.index) ? admin_vq.info : vp_dev.vqs[vq.index].
  - if vp_dev.per_vq_vectors:
    - v = info.msix_vector.
    - if v != VIRTIO_MSI_NO_VECTOR ∧ !is_slow_path_vector(v):
      - irq_update_affinity_hint(pci_irq_vector(pci_dev, v), NULL).
      - free_irq(pci_irq_vector(pci_dev, v), vq).
  - vp_del_vq(vq, info).
- vp_dev.per_vq_vectors = false.
- /* INTx */
- if intx_enabled: free_irq(pci_dev.irq, vp_dev); intx_enabled = 0.
- /* MSI-X management vectors (config, shared-VQ) */
- for i in 0..msix_used_vectors: free_irq(pci_irq_vector(pci_dev, i), vp_dev).
- for i in 0..msix_vectors: free_cpumask_var(msix_affinity_masks[i]).
- if msix_enabled:
  - vp_dev.config_vector(vp_dev, VIRTIO_MSI_NO_VECTOR) /* tell device */.
  - pci_free_irq_vectors(pci_dev); msix_enabled = 0.
- kfree(msix_names, msix_affinity_masks, vp_dev.vqs); zero pointers.

REQ-17: vp_set_vq_affinity(vq, cpu_mask):
- if !vq.callback: return -EINVAL.
- if msix_enabled:
  - mask = vp_dev.msix_affinity_masks[info.msix_vector].
  - irq = pci_irq_vector(pci_dev, info.msix_vector).
  - if !cpu_mask: irq_update_affinity_hint(irq, NULL).
  - else: cpumask_copy(mask, cpu_mask); irq_set_affinity_and_hint(irq, mask).
- return 0.

REQ-18: vp_is_avq(vdev, index):
- if !virtio_has_feature(vdev, VIRTIO_F_ADMIN_VQ): return false.
- return index == vp_dev.admin_vq.vq_index.

REQ-19: virtio_pci_sriov_configure(pci_dev, num_vfs):
- if !(vdev.config.get_status(vdev) & VIRTIO_CONFIG_S_DRIVER_OK): return -EBUSY.
- if !__virtio_test_bit(vdev, VIRTIO_F_SR_IOV): return -EINVAL.
- if pci_vfs_assigned(pci_dev): return -EPERM.
- if num_vfs == 0: pci_disable_sriov(pci_dev); return 0.
- ret = pci_enable_sriov(pci_dev, num_vfs); if ret < 0: return ret.
- return num_vfs.

REQ-20: virtio_pci_reset_prepare / virtio_pci_reset_done (PCI-error handler):
- prepare: virtio_device_reset_prepare(vdev); if pci_is_enabled: pci_disable_device(pci_dev).
- done: if pci_is_enabled: return; ret = pci_enable_device(pci_dev); if !ret: pci_set_master(pci_dev); virtio_device_reset_done(vdev).
- per-EOPNOTSUPP: tolerated; log warning otherwise.

REQ-21: PM hooks (CONFIG_PM_SLEEP):
- virtio_pci_freeze: virtio_device_freeze(vdev); on success: pci_disable_device(pci_dev).
- virtio_pci_restore: pci_enable_device; pci_set_master; virtio_device_restore(vdev).
- virtio_pci_suspend / virtio_pci_resume: if vp_supports_pm_no_reset(dev) ⟹ no-op; else: freeze / restore.
- vp_supports_pm_no_reset: requires pm_cap; reads PMCSR via pci_read_config_word; returns (pmcsr & PCI_PM_CTRL_NO_SOFT_RESET).
- PM op-table: suspend/resume/freeze/thaw/poweroff/restore.

REQ-22: virtio_pci_release_dev(_d):
- /* kref-release of virtio_device; safe to free now */
- kfree(vp_dev).

REQ-23: virtio_pci_id_table[]:
- { PCI_DEVICE(PCI_VENDOR_ID_REDHAT_QUMRANET = 0x1AF4, PCI_ANY_ID) }.
- Modern: device-IDs 0x1040..0x107F (transitional 0x1000..0x103F also matched here for legacy probe).

REQ-24: virtio_pci_vf_get_pf_dev(pdev):
- pf_vp_dev = pci_iov_get_pf_drvdata(pdev, &virtio_pci_driver).
- if IS_ERR: return NULL.
- return &pf_vp_dev.vdev.

REQ-25: ISR-byte semantics (legacy + per-isr-cap modern):
- VIRTIO_PCI_ISR_QUEUE bit (0x1): one or more VQ progressed.
- VIRTIO_PCI_ISR_CONFIG bit (0x2): config-space changed.
- Reading ISR atomically clears it.

REQ-26: VIRTIO_CONFIG_S_* lifecycle (transport-agnostic; modern/legacy implementations set these via cap-mapped status reg / BAR0+0x12):
- 0 → ACKNOWLEDGE → DRIVER → FEATURES_OK → DRIVER_OK; FAILED on any error; DEVICE_NEEDS_RESET signals reset required.

REQ-27: per-capability walk (delegated to virtio_pci_modern):
- VIRTIO_PCI_CAP_COMMON_CFG (1): common-config (device/driver features, status, msix-vec, num_queues, queue_select/size/notify_off).
- VIRTIO_PCI_CAP_NOTIFY_CFG (2): per-VQ notification region + notify_off_multiplier.
- VIRTIO_PCI_CAP_ISR_CFG (3): ISR-byte region.
- VIRTIO_PCI_CAP_DEVICE_CFG (4): per-device config (e.g. virtio-net mac, virtio-blk capacity).
- VIRTIO_PCI_CAP_PCI_CFG (5): cfg-window for INDIRECT BAR access.
- VIRTIO_PCI_CAP_SHARED_MEMORY_CFG (8): shared-memory regions (virtio-fs DAX).
- VIRTIO_PCI_CAP_VENDOR_CFG (9): vendor-specific.

REQ-28: Force-legacy module-param:
- /sys/module/virtio_pci/parameters/force_legacy = 0|1 (CONFIG_VIRTIO_PCI_LEGACY only).
- 1 ⟹ probe ladder: legacy → modern (for transitional virtio-1 devices).
- 0 (default) ⟹ probe ladder: modern → legacy.

## Acceptance Criteria

- [ ] AC-1: virtio_pci_probe allocates vp_dev, enables PCI, dispatches to modern/legacy, registers virtio_device.
- [ ] AC-2: Modern probe (default) tried first; legacy probe used as -ENODEV fallback.
- [ ] AC-3: force_legacy=1 reverses ladder; modern used only on -ENODEV/-ENOMEM legacy.
- [ ] AC-4: vp_find_vqs tries EACH → SHARED_SLOW → SHARED MSI-X policies, then INTx.
- [ ] AC-5: Config-vector at MSI-X index 0; VQ-vectors at 1+ when per_vq_vectors.
- [ ] AC-6: vp_interrupt returns IRQ_NONE when ISR byte reads zero.
- [ ] AC-7: vp_interrupt invokes vp_config_changed when ISR & VIRTIO_PCI_ISR_CONFIG.
- [ ] AC-8: vp_config_changed dispatches slow-path VQ interrupts too.
- [ ] AC-9: vp_notify writes vq.index to the notification iomem ptr.
- [ ] AC-10: vp_del_vqs frees all per-VQ IRQs, MSI-X vectors, info structs, and writes VIRTIO_MSI_NO_VECTOR to config-vector before disabling MSI-X.
- [ ] AC-11: sriov_configure rejects without VIRTIO_F_SR_IOV or before DRIVER_OK.
- [ ] AC-12: reset_prepare/reset_done bracket per-PCI-error reset; tolerates -EOPNOTSUPP.
- [ ] AC-13: PM no-soft-reset path (PCI_PM_CTRL_NO_SOFT_RESET) skips freeze/restore on suspend/resume.
- [ ] AC-14: virtio_pci_remove on surprise-removal flags virtio_break_device before unregister.
- [ ] AC-15: Admin-VQ (VIRTIO_F_ADMIN_VQ) placed on slow_virtqueues list, served by config-vec shared ISR.

## Architecture

```
struct VirtioPciDevice {
  vdev: VirtioDevice,                          // embedded; container_of
  pci_dev: *PciDev,
  isr: *mut u8,                                // __iomem
  ioaddr: *mut u8,                             // legacy BAR0
  vqs: Vec<*mut VirtioPciVqInfo>,              // by virtqueue index
  admin_vq: VirtioPciAdminVq,
  virtqueues: ListHead,                        // fast-path
  slow_virtqueues: ListHead,                   // admin-VQ
  lock: Spinlock,
  msix_vectors: u32,
  msix_used_vectors: u32,
  msix_names: Vec<MsixName>,                   // sized [nvectors]
  msix_affinity_masks: Vec<Cpumask>,           // sized [nvectors]
  msix_enabled: bool,
  intx_enabled: bool,
  per_vq_vectors: bool,
  is_legacy: bool,
  setup_vq: fn(...),
  del_vq: fn(...),
  config_vector: fn(*mut Self, u16) -> u16,
  avq_index: Option<fn(*mut VirtioDevice, &mut u16, &mut u16) -> i32>,
}

struct VirtioPciVqInfo {
  vq: *mut Virtqueue,
  msix_vector: u16,                            // VIRTIO_MSI_NO_VECTOR if none
  node: ListNode,                              // in virtqueues / slow_virtqueues
}

struct VirtioPciAdminVq {
  vq_index: u16,
  info: *mut VirtioPciVqInfo,
  name: [u8; 16],                              // "avq.<idx>"
}

enum VpVqVectorPolicy { Each, SharedSlow, Shared }
```

`VirtioPci::probe(pci_dev, id) -> i32`:
1. vp_dev = kzalloc.
2. pci_set_drvdata(pci_dev, vp_dev).
3. vp_dev.vdev.dev.parent = &pci_dev.dev; .release = release_dev.
4. vp_dev.pci_dev = pci_dev; INIT_LIST_HEADs; spinlock_init.
5. pci_enable_device(pci_dev)? → err_enable_device.
6. /* probe ladder per force_legacy */
7. if force_legacy:
   - rc = VirtioPciLegacy::probe(vp_dev).
   - if rc == ENODEV ∨ ENOMEM: rc = VirtioPciModern::probe(vp_dev).
8. else:
   - rc = VirtioPciModern::probe(vp_dev).
   - if rc == ENODEV: rc = VirtioPciLegacy::probe(vp_dev).
9. if rc: goto err_probe.
10. pci_set_master(pci_dev).
11. rc = register_virtio_device(&vp_dev.vdev); reg_dev = vp_dev.
12. if rc: goto err_register.
13. return 0.
14. err_register: dispatch remove(vp_dev).
15. err_probe: pci_disable_device(pci_dev).
16. err_enable_device: if reg_dev: put_device else: kfree(vp_dev).

`VirtioPci::find_vqs(vdev, nvqs, vqs, vqs_info, desc) -> i32`:
1. err = find_vqs_msix(vdev, ..., Each, desc); if !err: return 0.
2. err = find_vqs_msix(vdev, ..., SharedSlow, desc); if !err: return 0.
3. err = find_vqs_msix(vdev, ..., Shared, desc); if !err: return 0.
4. if !vp_dev.pci_dev.irq: return err.
5. return find_vqs_intx(vdev, nvqs, vqs, vqs_info).

`VirtioPci::request_msix_vectors(vdev, nvectors, per_vq_vectors, desc) -> i32`:
1. vp_dev.msix_vectors = nvectors.
2. Alloc msix_names[nvectors] and msix_affinity_masks[nvectors]; alloc_cpumask_var per.
3. flags = PCI_IRQ_MSIX; if !per_vq_vectors: desc = NULL; else if desc: flags |= PCI_IRQ_AFFINITY; ++desc.pre_vectors.
4. pci_alloc_irq_vectors_affinity(pci_dev, nvectors, nvectors, flags, desc).
5. msix_enabled = 1.
6. v = msix_used_vectors (0).
7. snprintf(msix_names[v], "%s-config", dev_name).
8. request_irq(pci_irq_vector(pci_dev, v), vp_config_changed, 0, msix_names[v], vp_dev).
9. ++msix_used_vectors.
10. v = vp_dev.config_vector(vp_dev, v); if v == VIRTIO_MSI_NO_VECTOR: return EBUSY.
11. if !per_vq_vectors:
    - v = msix_used_vectors; snprintf "%s-virtqueues".
    - request_irq(... vp_vring_interrupt ...).
    - ++msix_used_vectors.
12. return 0.

`VirtioPci::intx_irq(irq, opaque) -> IrqReturn`:
1. isr = ioread8(vp_dev.isr).   /* clears on read */
2. if !isr: return IRQ_NONE.
3. if isr & VIRTIO_PCI_ISR_CONFIG: vp_config_changed(irq, opaque).
4. return vp_vring_interrupt(irq, opaque).

`VirtioPci::vring_interrupt_irq(irq, opaque) -> IrqReturn`:
1. spin_lock_irqsave(&vp_dev.lock).
2. ret = IRQ_NONE.
3. for info in vp_dev.virtqueues: if vring_interrupt(irq, info.vq) == IRQ_HANDLED: ret = IRQ_HANDLED.
4. spin_unlock_irqrestore.
5. return ret.

`VirtioPci::config_changed_irq(irq, opaque) -> IrqReturn`:
1. virtio_config_changed(&vp_dev.vdev).
2. vp_vring_slow_path_interrupt(irq, vp_dev).
3. return IRQ_HANDLED.

`VirtioPci::del_vqs(vdev)`:
1. for (vq, n) in &vdev.vqs:
   - info = is_avq(vdev, vq.index) ? admin_vq.info : vp_dev.vqs[vq.index].
   - if per_vq_vectors ∧ info.msix_vector != NO_VECTOR ∧ !is_slow_path_vector(info.msix_vector):
     - irq = pci_irq_vector(pci_dev, info.msix_vector).
     - irq_update_affinity_hint(irq, NULL).
     - free_irq(irq, vq).
   - del_vq(vq, info).
2. per_vq_vectors = false.
3. if intx_enabled: free_irq(pci_dev.irq, vp_dev); intx_enabled = 0.
4. for i in 0..msix_used_vectors: free_irq(pci_irq_vector(pci_dev, i), vp_dev).
5. for i in 0..msix_vectors: free_cpumask_var(msix_affinity_masks[i]).
6. if msix_enabled: config_vector(vp_dev, VIRTIO_MSI_NO_VECTOR); pci_free_irq_vectors(pci_dev); msix_enabled = 0.
7. zero counters; kfree(msix_names, msix_affinity_masks, vp_dev.vqs).

`VirtioPci::sriov_configure(pci_dev, num_vfs) -> i32`:
1. if !(get_status(vdev) & DRIVER_OK): return EBUSY.
2. if !virtio_test_bit(vdev, VIRTIO_F_SR_IOV): return EINVAL.
3. if pci_vfs_assigned(pci_dev): return EPERM.
4. if num_vfs == 0: pci_disable_sriov(pci_dev); return 0.
5. ret = pci_enable_sriov(pci_dev, num_vfs); if ret < 0: return ret.
6. return num_vfs.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `probe_kfree_balanced` | INVARIANT | per-probe error path: vp_dev freed exactly once via kfree or put_device. |
| `msix_vec0_is_config` | INVARIANT | per-request_msix_vectors: config-vec at index 0 only. |
| `pervq_irq_freed_in_del_vqs` | INVARIANT | per-del_vqs: every per-VQ IRQ requested in find_one_vq_msix is free_irq'd. |
| `slow_path_no_dedicated_irq` | INVARIANT | per-slow-path-VQ: never has dedicated request_irq (shares config-vec). |
| `isr_byte_read_once` | INVARIANT | per-vp_interrupt: ioread8(isr) called exactly once; ISR not re-read in same ISR call. |
| `force_legacy_dispatch` | INVARIANT | per-probe: force_legacy=1 ⟹ legacy_probe called before modern_probe. |
| `affinity_hint_clear_before_free_irq` | INVARIANT | per-del_vqs: irq_update_affinity_hint(.., NULL) before free_irq. |
| `config_vector_cleared_before_disable_msix` | INVARIANT | per-del_vqs: config_vector(vp_dev, NO_VECTOR) before pci_free_irq_vectors. |
| `release_after_unregister` | INVARIANT | per-remove: unregister_virtio_device before pci_disable_device. |

### Layer 2: TLA+

`drivers/virtio/virtio-pci.tla`:
- Models: probe ladder (force_legacy × modern × legacy), find_vqs ladder (EACH/SHARED_SLOW/SHARED/INTX), ISR (MSI-X vs INTx), remove, suspend/freeze.
- Properties:
  - `safety_one_transport_chosen` — per-probe: exactly one of (modern, legacy) succeeds; is_legacy reflects it.
  - `safety_no_unbalanced_request_irq` — per-vdev: count(request_irq) == count(free_irq) on remove.
  - `safety_isr_clears_on_read` — per-vp_interrupt: ISR byte cleared once per read.
  - `safety_msix_and_intx_mutex` — per-vp_dev: ¬(msix_enabled ∧ intx_enabled).
  - `safety_admin_vq_on_slow_list` — per-find_vqs_msix: admin-VQ info.node in slow_virtqueues only.
  - `liveness_find_vqs_terminates` — per-vp_find_vqs: returns from one of the 4 paths.
  - `liveness_remove_completes` — per-virtio_pci_remove: all resources freed in finite steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VirtioPci::probe` post: ret == 0 ⟹ vdev registered ∧ is_legacy ∈ {0,1} | `VirtioPci::probe` |
| `VirtioPci::find_vqs` post: ret == 0 ⟹ vqs[i] != NULL ∀ i with vqi.name | `VirtioPci::find_vqs` |
| `VirtioPci::request_msix_vectors` post: msix_enabled ⟹ msix_names + msix_affinity_masks allocated for nvectors | `VirtioPci::request_msix_vectors` |
| `VirtioPci::intx_irq` post: isr==0 ⟹ IRQ_NONE; isr & CONFIG ⟹ config-changed called | `VirtioPci::intx_irq` |
| `VirtioPci::del_vqs` post: per_vq_vectors==false ∧ msix_enabled==false ∧ intx_enabled==false ∧ vp_dev.vqs == NULL | `VirtioPci::del_vqs` |
| `VirtioPci::sriov_configure` post: num_vfs>0 ⟹ pci_enable_sriov called ∧ DRIVER_OK held | `VirtioPci::sriov_configure` |

### Layer 4: Verus/Creusot functional

`Per-probe → modern_probe / legacy_probe → register_virtio_device → driver-bind → find_vqs (policy ladder) → device-OK → vp_notify (kicks) + vp_interrupt (returns)` semantic equivalence: per-VirtIO 1.2 spec §4.1 (PCI bus binding) and per-Linux drivers/virtio/virtio_pci_*.c reference behavior.

## Hardening

(Inherits row-1 features from `drivers/virtio/00-overview.md` § Hardening.)

VirtIO-PCI transport reinforcement:

- **Per-probe error path strict free** — defense against per-vp_dev leak or double-free in legacy/modern fallback.
- **Per-MSI-X vector cap == nvectors min == nvectors max** — defense against per-partial-allocation (no graceful degrade within MSI-X path).
- **Per-config-vector at index 0 invariant** — defense against per-IRQ-routing confusion (config-changed delivered to wrong handler).
- **Per-VQ IRQ name `%s-%s` unique within vdev** — defense against per-/proc/interrupts ambiguity.
- **Per-ISR-byte read clears atomically** — defense against per-IRQ-storm (re-read returns stale ISR).
- **Per-INTx IRQF_SHARED** — defense against per-mis-claimed-IRQ on shared INTx line.
- **Per-slow-path-VQ shares config-vec** — defense against per-admin-VQ-DoS (admin-VQ IRQ can't starve fast-path VQs).
- **Per-SR-IOV gated on DRIVER_OK + VIRTIO_F_SR_IOV** — defense against per-pre-init VF-enable.
- **Per-pci_vfs_assigned check on disable** — defense against per-PASSTHROUGH-VF-rip during guest-use.
- **Per-surprise-removal virtio_break_device** — defense against per-removed-device-still-doing-IO.
- **Per-PM no-soft-reset PMCSR check** — defense against per-spurious-full-reset on D3 transitions.
- **Per-affinity-hint cleared before free_irq** — defense against per-leak of affinity-hint pointer to freed cpumask.
- **Per-config_vector(NO_VECTOR) before pci_free_irq_vectors** — defense against per-device-MSI-X-write-to-freed-vector.
- **Per-force_legacy bool-param-only (CONFIG_VIRTIO_PCI_LEGACY)** — defense against per-modern-only build accidentally falling to legacy.

## Open Questions

- (none at this Tier-3 level)

## Out of Scope

- `drivers/virtio/virtio_pci_modern.c` cap-walk / common-config / notify-cap detail (covered separately if expanded)
- `drivers/virtio/virtio_pci_legacy.c` BAR0 I/O probe detail (covered separately if expanded)
- `drivers/virtio/virtio_pci_modern_dev.c` / `virtio_pci_legacy_dev.c` library helpers (covered separately if expanded)
- `drivers/virtio/virtio_pci_admin_legacy_io.c` admin-legacy-IO commands (covered separately if expanded)
- `drivers/virtio/virtio_ring.c` ring impl (covered in `virtio-ring.md` Tier-3)
- `drivers/virtio/virtio.c` bus core (covered in `virtio-core.md` Tier-3)
- vDPA bridge `virtio_vdpa.c` (covered separately)
- Implementation code
