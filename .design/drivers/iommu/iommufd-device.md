# Tier-3: drivers/iommu/iommufd/device.c — Per-device iommufd binding (bind / attach / replace / unbind for VFIO + VDPA backends)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iommufd/device.c
  - drivers/iommu/iommufd/iommufd_private.h
  - include/linux/iommufd.h (iommufd_device_*)
-->

## Summary

The per-device iommufd-binding subsystem — when a VFIO or VDPA backend wants to put a device under iommufd's control (instead of using legacy VFIO type1 container), it calls `iommufd_device_bind()` to allocate a per-device `iommufd_device` object representing this device under a particular `iommufd_ctx`. Then `iommufd_device_attach()` attaches the device to a HWPT (Hardware Page Table), wiring the per-device IOMMU domain to translate the device's DMA. `iommufd_device_replace()` atomically swaps HWPTs (used for live migration). `iommufd_device_unbind()` reverses everything.

This Tier-3 covers `drivers/iommu/iommufd/device.c` (~1700 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommufd_device` | per-device iommufd state | `kernel::iommu::iommufd::Device` |
| `iommufd_device_bind(ictx, dev, &id)` | bind device to ictx, return id | `Device::bind` |
| `iommufd_device_unbind(idev)` | unbind | `Device::unbind` (Drop) |
| `iommufd_device_attach(idev, hwpt)` | attach to HWPT | `Device::attach` |
| `iommufd_device_detach(idev)` | detach | `Device::detach` |
| `iommufd_device_replace(idev, new_hwpt)` | atomic swap from current HWPT to new | `Device::replace` |
| `iommufd_device_iommufd_set_pasid(idev, pasid, hwpt)` | per-PASID attach (VDEVICE companion) | `Device::set_pasid` |
| `iommufd_device_iommufd_remove_pasid(idev, pasid, hwpt)` | per-PASID detach | `Device::remove_pasid` |
| `iommufd_device_alloc_pasid(idev, &pasid)` | per-device PASID alloc | `Device::alloc_pasid` |
| `iommufd_device_free_pasid(idev, pasid)` | per-device PASID free | `Device::free_pasid` |
| `iommufd_get_hw_info(ictx, cmd)` | IOMMU_GET_HW_INFO ioctl handler | `Ctx::handle_get_hw_info` |
| `iommufd_device_attach_reserved_iova(idev, iova, length)` | reserve specific IOVA range | `Device::attach_reserved_iova` |
| `iommufd_device_get_resv_regions(idev, list)` | per-device reserved region enumeration | `Device::get_resv_regions` |
| `iommufd_device_destroy(obj)` | destroy IDevice (releases bind) | `Device::destroy` (object Drop) |

## Compatibility contract

REQ-1: `iommufd_device_bind(ictx, dev, &id)` UAPI semantics:
- Validate dev has IOMMU coverage (per-`iommu_group` membership; cross-ref `iommu-core.md`).
- Verify exclusive ownership (no other ictx has this device bound; -EBUSY if so).
- Allocate per-device `iommufd_device` object.
- Insert into ictx.objects; assign id.
- Return id to caller.

REQ-2: `iommufd_device_attach(idev, hwpt)` semantics:
- Validate hwpt is HWPT-PAGING or HWPT-NESTED (cross-ref `iommufd-hwpt.md`).
- Per-vendor `iommu_attach_group(hwpt.domain, idev.iommu_group)` (cross-ref `iommu-core.md`).
- Add idev to hwpt.attached_devs (RCU-protected).
- Set idev.attached = Some(hwpt).

REQ-3: `iommufd_device_detach(idev)` semantics:
- Per-vendor `iommu_detach_group(hwpt.domain, idev.iommu_group)`.
- Remove idev from hwpt.attached_devs.
- Clear idev.attached = None.

REQ-4: `iommufd_device_replace(idev, new_hwpt)` semantics:
- Atomic swap from current HWPT to new — single device's translation transitions in single HW instant.
- Per-vendor `iommu_attach_device_pasid(...)` (uses HW-side replace mechanism per spec).
- Update idev's HWPT reference + RCU-list memberships.

REQ-5: `iommufd_get_hw_info(ictx, cmd)` UAPI:
- Input: `dev_id`, `data_type` (per-vendor: VTD / SMMUV3 / AMD).
- Output: per-vendor capabilities struct (e.g., Intel: CAP_REG + ECAP_REG decoded; SMMUV3: IDR registers; AMD: capability blob).
- Used by qemu to determine guest IOMMU capability advertisement.

REQ-6: Per-device PASID alloc per-device `pci_max_pasids(dev)`; per-device PASID space partitioned per-IOAS for nested use.

REQ-7: Per-device exclusive ownership: same device cannot be bound to two ictx simultaneously; second bind returns -EBUSY.

REQ-8: Per-VFIO/VDPA backend integration: each backend (vfio-pci-core / vfio-mdev / vhost-vdpa) calls `iommufd_device_bind` from its bind path; backend keeps `idev` handle for subsequent attach/detach.

REQ-9: Per-device reserved-region enumeration: `iommufd_device_get_resv_regions(idev, list)` returns per-IOMMU + per-device reserved IOVA ranges (MSI-X window, RMRR, IOMMU-internal); used by IOAS to refuse map of those ranges (cross-ref `iommufd-ioas.md`).

REQ-10: Bind/attach/replace operations atomic w.r.t. concurrent device DMA — per-device DMA either uses old HWPT or new, never partial state.

## Acceptance Criteria

- [ ] AC-1: vfio-pci-core test: bind PCIe device to iommufd via `vfio_pci_core_iommufd_bind` → `iommufd_device_bind`; subsequent attach to HWPT-PAGING + IOAS_MAP + DMA works.
- [ ] AC-2: Exclusive ownership test: bind device to ictx-A; second bind to ictx-B returns -EBUSY.
- [ ] AC-3: Replace test: live-migration scenario — src qemu detaches device from src HWPT; dst qemu attaches to dst HWPT (different IOAS); device DMA continues without observed downtime.
- [ ] AC-4: PASID attach test: `iommufd_device_alloc_pasid` + `iommufd_device_iommufd_set_pasid` for nested HWPT; per-PASID DMA works.
- [ ] AC-5: GET_HW_INFO test: `IOMMU_GET_HW_INFO(dev_id, VTD)` returns Intel VT-d cap+ecap fields; userspace decodes correctly.
- [ ] AC-6: Reserved-region enforcement: per-device resv-regions enumerate MSI-X window; subsequent IOAS_MAP onto MSI-X-window range rejected with -EINVAL.
- [ ] AC-7: VDPA bind test: vhost-vdpa device binds via iommufd; per-device IOMMU translation works for VDPA virtqueue DMA.
- [ ] AC-8: kselftest iommufd device-bind subset passes.

## Architecture

`Device` lives in `kernel::iommu::iommufd::Device`:

```
struct Device {
  obj: Object,                          // base iommufd object
  ictx: Arc<Ctx>,
  dev: Arc<KernelDevice>,               // backing kernel struct device
  iommu_dev: Arc<IommuDevice>,          // per-IOMMU instance
  group: Arc<IommuGroup>,               // per-iommu_group
  attached: Mutex<AttachedHwpt>,        // None or Some(hwpt)
  igate: Mutex<()>,
  evict_iopf: AtomicBool,
  ...
}

enum AttachedHwpt {
  None,
  Paging(Arc<HwptPaging>),
  Nested(Arc<HwptNested>),
}
```

Bind flow `Device::bind(ictx, dev, &out_id)`:
1. `iommu_probe_device(dev)` ensures device has iommu_group (cross-ref `iommu-core.md`).
2. Verify exclusive ownership: walk ictx.objects for IDevice with same `dev`; if found, return -EEXIST.
3. Cross-ictx check: per-device `dev->iommufd_dev` ptr atomic-cmpxchg from null to this idev (else -EBUSY from cross-ictx contention).
4. Allocate `iommufd_device` struct.
5. Initialize ref to dev, iommu_dev, group.
6. Insert into ictx.objects → assign id.
7. Return out_id.

Attach flow `Device::attach(idev, hwpt)`:
1. Take idev.igate.
2. Validate idev.attached == None.
3. `iommu_attach_group(hwpt.domain, idev.group)` — per-vendor IOMMU op binds device to domain.
4. Add idev to hwpt.attached_devs (RCU-protected).
5. Set idev.attached = Some(hwpt).
6. Drop idev.igate.

Detach flow `Device::detach(idev)`:
1. Take idev.igate.
2. Validate idev.attached.is_some().
3. `iommu_detach_group(hwpt.domain, idev.group)`.
4. Remove idev from hwpt.attached_devs.
5. Set idev.attached = None.
6. Drop idev.igate.

Replace flow `Device::replace(idev, new_hwpt)`:
1. Take idev.igate.
2. Validate idev.attached.is_some().
3. Per-vendor `iommu_replace_group_handle(idev.group, new_hwpt.domain, ...)` — atomic-replace per spec; HW-side replace mechanism (e.g., Intel: PASID-table-entry replace; ARM SMMUV3: STE replace).
4. Remove idev from old_hwpt.attached_devs; add to new_hwpt.attached_devs.
5. Update idev.attached = Some(new_hwpt).
6. Drop idev.igate.

Per-PASID attach `Device::set_pasid(idev, pasid, hwpt)`:
1. Take idev.igate.
2. Validate dev supports PASID + cross-ref `intel-pasid.md` for vendor-specific install.
3. Per-vendor `iommu_attach_device_pasid(hwpt.domain, idev.dev, pasid)` (cross-ref `iommu-core.md`).
4. Track per-PASID attachment in idev.
5. Drop idev.igate.

`Ctx::handle_get_hw_info(cmd)`:
1. Look up dev_id → idev.
2. Per-vendor `iommu_ops->hw_info(idev.dev, &length, &data_type)`:
   - Intel: copy CAP_REG + ECAP_REG + EXT_CAP_REG decoded into struct iommu_hw_info_vtd.
   - SMMUV3: copy IDR0-5 + IIDR + AIDR registers.
   - AMD: copy capability + extended-capability blob.
3. Copy data + length to userspace.

Unbind/destroy `Device::Drop`:
1. Take idev.igate.
2. If attached: detach via `Device::detach`.
3. Walk per-PASID attachments; detach each.
4. Atomic-CAS dev->iommufd_dev from this idev to null (releases exclusive ownership).
5. Drop refs to dev, iommu_dev, group, ictx.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `idev_no_uaf` | UAF | `Arc<Device>` outlives all in-flight attach/detach + per-PASID operations; destroy waits for refcount==0. |
| `exclusive_ownership` | INVARIANT | per-kernel-device `dev->iommufd_dev` atomic-CAS enforces single-binding-at-a-time. |
| `attach_atomicity` | ATOMICITY | `Device::replace` HW-side atomic replace; observer sees old or new HWPT, never partial. |
| `pasid_no_collision` | UNIQUENESS | per-device PASID alloc bounded by per-IOMMU PASID-bitmap; never collides with concurrent PASID alloc. |

### Layer 2: TLA+

(No specific TLA+ — semantics covered by parent's iommu-core + iommufd-main TLA+ models for per-device bind/attach orchestration.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Device::bind` post: idev in ictx.objects with refcount=1; dev->iommufd_dev set | `Device::bind` |
| `Device::attach` post: idev.attached == Some(hwpt); hwpt.attached_devs contains idev | `Device::attach` |
| `Device::replace` post: idev.attached == Some(new_hwpt); old_hwpt.attached_devs does not contain idev; new_hwpt.attached_devs does | `Device::replace` |
| `Device::Drop` post: dev->iommufd_dev cleared; HWPT detached if attached | `Device::Drop` |

### Layer 4: Verus/Creusot functional

`Device::bind(ictx, dev) → Device::attach(idev, hwpt) → IOMMU_IOAS_MAP(ioas, iova, ...) → device DMA at iova → access succeeds` round-trip equivalence: bound + attached device's DMA reaches userspace pages via configured HWPT translation.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iommufd-device specific reinforcement:

- **Exclusive ownership enforced via atomic-CAS** — defense against cross-ictx device-state confusion + per-device split-brain.
- **Per-vendor `iommu_replace_group_handle` atomic** — defense against partial-replace exposing device DMA to two domains simultaneously.
- **Per-PASID alloc bounded** by per-device pci_max_pasids + per-IOMMU PASID-table size; defense against PASID-exhaustion via single device.
- **GET_HW_INFO data length validated** — per-vendor data length checked against expected struct size before copy; defense against userspace-buffer overflow.
- **Per-device reserved-region enumeration** — IOAS_MAP validates against per-device resv-regions (cross-ref `iommufd-ioas.md`); defense against device-MSI-window overlap.
- **Detach during in-flight DMA safe** — per-vendor IOMMU ops drain HW IOTLB before returning from detach; subsequent device DMA faults cleanly.
- **Per-device VDPA + VFIO bind path validated** — backend identifies itself; cross-backend bind rejected at iommu_attach_group.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- iommufd_main top-level dispatch (covered in `iommufd-main.md` Tier-3)
- IOAS internals (covered in `iommufd-ioas.md` Tier-3)
- HWPT internals (covered in `iommufd-hwpt.md` Tier-3)
- VIOMMU + VDEVICE (covered in `iommufd-viommu.md` Tier-3)
- Eventq for PRI faults (covered in `iommufd-eventq.md` Tier-3)
- Per-vendor IOMMU drivers (covered in `intel-iommu.md` + `amd-iommu.md` Tier-3s)
- VFIO bind path (covered in `drivers/vfio/iommufd-backend.md` future Tier-3)
- VDPA bind path (covered in `drivers/vhost/vdpa.md` future Tier-3)
- 32-bit-only paths
- Implementation code
