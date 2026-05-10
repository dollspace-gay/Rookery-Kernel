# Tier-3: drivers/iommu/iommufd/hw_pagetable.c — HWPT (Hardware Page Table) object (paging + nested + dirty-tracking + per-vendor data)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iommufd/hw_pagetable.c
  - drivers/iommu/iommufd/iommufd_private.h
  - include/uapi/linux/iommufd.h (IOMMU_HWPT_*)
-->

## Summary

The HWPT (Hardware Page Table) object is the iommufd-side representation of an actual hardware IOMMU domain — what the per-vendor IOMMU driver instantiates as `struct iommu_domain` (cross-ref `iommu-core.md`). Two flavors:

- **HWPT-PAGING**: standard paging domain backed by an IOAS (cross-ref `iommufd-ioas.md`); userspace creates one via `IOMMU_HWPT_ALLOC` with `pt_id = ioas_id` + `data_type = IOMMU_HWPT_DATA_NONE`. When attached to a device, the device DMA goes through this domain's IOMMU pagetable populated with mappings from the linked IOAS.
- **HWPT-NESTED**: nested IOMMU page table for guest IOMMU virtualization (qemu/cloud-hypervisor exposing virtual IOMMU to guest); userspace creates one via `IOMMU_HWPT_ALLOC` with `pt_id = parent_hwpt_id` + `data_type = IOMMU_HWPT_DATA_VTD_S1` / `_ARM_SMMUV3` / `_AMD_V2`. Per-vendor `data_uptr` blob describes the guest-managed L1 pagetable; HWPT-paging parent provides the L2 (host-managed) pagetable.

Owns: HWPT lifecycle (alloc/destroy), per-vendor `iommu_ops->domain_alloc(...)` dispatch, dirty-tracking enable/disable + read-and-clear bitmap, selective TLB invalidation for nested mode (IOMMU_HWPT_INVALIDATE), per-HWPT device attach/detach list, IOAS attach for paging HWPTs.

This Tier-3 covers `drivers/iommu/iommufd/hw_pagetable.c` (~550 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommufd_hwpt_paging` | per-paging-HWPT object | `kernel::iommu::iommufd::HwptPaging` |
| `struct iommufd_hwpt_nested` | per-nested-HWPT object | `kernel::iommu::iommufd::HwptNested` |
| `iommufd_hwpt_paging_alloc(ictx, ioas, idev, flags, immediate_attach, user_data)` | alloc paging HWPT | `HwptPaging::alloc` |
| `iommufd_hwpt_nested_alloc(ictx, parent, idev, flags, user_data)` | alloc nested HWPT | `HwptNested::alloc` |
| `iommufd_hwpt_paging_destroy(obj)` | destroy paging HWPT | `HwptPaging::destroy` (Drop) |
| `iommufd_hwpt_nested_destroy(obj)` | destroy nested HWPT | `HwptNested::destroy` (Drop) |
| `iommufd_hwpt_paging_attach_ioas(hwpt, ioas)` | attach IOAS (install all IOAS mappings into HWPT's domain) | `HwptPaging::attach_ioas` |
| `iommufd_hwpt_paging_detach_ioas(hwpt)` | detach IOAS | `HwptPaging::detach_ioas` |
| `iommufd_hwpt_alloc(ictx, cmd)` | top-level IOMMU_HWPT_ALLOC ioctl handler | `Ctx::handle_hwpt_alloc` |
| `iommufd_hwpt_invalidate(ictx, cmd)` | IOMMU_HWPT_INVALIDATE ioctl (selective TLB invalidate for nested) | `Ctx::handle_hwpt_invalidate` |
| `iommufd_hwpt_set_dirty_tracking(ictx, cmd)` | IOMMU_HWPT_SET_DIRTY_TRACKING ioctl | `Ctx::handle_hwpt_set_dirty_tracking` |
| `iommufd_hwpt_get_dirty_bitmap(ictx, cmd)` | IOMMU_HWPT_GET_DIRTY_BITMAP ioctl | `Ctx::handle_hwpt_get_dirty_bitmap` |
| `iommufd_hwpt_replace(ictx, cmd)` | atomically replace one HWPT with another for a device (live migration) | `Ctx::handle_hwpt_replace` |
| `iommufd_hwpt_paging_iova_to_phys(hwpt, iova)` | translate via paging HWPT (debugging) | `HwptPaging::iova_to_phys` |

## Compatibility contract

REQ-1: `IOMMU_HWPT_ALLOC` UAPI byte-identical:
- Input: `dev_id`, `pt_id` (IOAS for paging OR parent HWPT for nested), `flags`, `data_type` (NONE / VTD_S1 / ARM_SMMUV3 / AMD_V2 / etc.), `data_len`, `data_uptr` (per-vendor blob for nested).
- Output: `out_hwpt_id`.
- Validate per-vendor data: per-vendor `iommu_ops->domain_alloc_user(...)` reads + validates `data_uptr` against per-vendor expected layout; rejects malformed.

REQ-2: HWPT_PAGING attach to IOAS: walk all IoptArea(s) in IOAS; for each, `iommu_map(hwpt.domain, iova, paddr, length, prot)`; future IOAS_MAP also propagates to this HWPT.

REQ-3: HWPT_NESTED parent must be HWPT_PAGING (provides the L2 pagetable); per-vendor data describes guest-managed L1 (e.g., Intel VT-d 1st-stage PT root for SVM-style guest IOMMU).

REQ-4: `IOMMU_HWPT_INVALIDATE`: per-vendor descriptor format; processed in batch via per-IOMMU QI for Intel, per-vendor invalidate-cmd for AMD.

REQ-5: `IOMMU_HWPT_SET_DIRTY_TRACKING(true|false)`: per-domain `iommu_ops->set_dirty_tracking(domain, enable)`; per-vendor SLADE (Intel) or HWDirty (AMD) bit set on each leaf PTE.

REQ-6: `IOMMU_HWPT_GET_DIRTY_BITMAP`: walk per-IOAS area-list (parent IOAS for paging, parent's parent IOAS for nested); per-vendor `iommu_ops->read_and_clear_dirty(domain, iova_range, &bitmap)` populates user-supplied bitmap; bits cleared per-page after read.

REQ-7: `IOMMU_HWPT_REPLACE`: atomically swap one HWPT for another for a single attached device; used for live-migration handoff (source qemu detaches device from source HWPT, dest qemu attaches to dest HWPT, all without device DMA pause).

REQ-8: Per-HWPT attached-device list (RCU-protected): `attach_device` adds dev to list; `detach_device` removes; map/unmap on the parent IOAS propagates to all HWPTs attached.

REQ-9: HWPT_NESTED invalidate-only-once: invalidate cmd marked completed via QI Wait descriptor; userspace sees synchronous completion via ioctl return.

REQ-10: HWPT destroy invariants: HWPT cannot be destroyed if still attached to any device; HWPT-paging cannot be destroyed if attached as parent by HWPT-nested.

## Acceptance Criteria

- [ ] AC-1: qemu vfio-pci passthrough w/ iommufd: `IOMMU_HWPT_ALLOC(ioas, NONE)` + attach VFIO-device → guest sees device + DMA works.
- [ ] AC-2: Nested HWPT test on Intel HW: qemu `-device intel-iommu` exposes virtual IOMMU to guest; guest installs guest L1 pagetable; passes via `data_uptr=VTD_S1`; HW walks L1 (guest) + L2 (host) tables for guest's vfio-pci device DMA.
- [ ] AC-3: Dirty-tracking test on Sapphire Rapids+: enable via `IOMMU_HWPT_SET_DIRTY_TRACKING`; sustained device DMA writes; `IOMMU_HWPT_GET_DIRTY_BITMAP` returns correct dirty page bits.
- [ ] AC-4: Replace test: live-migration scenario: src qemu detaches device from src HWPT; dst qemu attaches to dst HWPT (different IOAS); device DMA continues without observed downtime.
- [ ] AC-5: HWPT destroy refused when attached: attempt `IOMMU_DESTROY` on attached HWPT returns -EBUSY.
- [ ] AC-6: Per-vendor data validation: malformed `data_uptr` for VTD_S1 nested → -EINVAL with no driver crash.
- [ ] AC-7: kselftest iommufd nested + dirty-tracking subset passes.

## Architecture

`HwptPaging` lives in `kernel::iommu::iommufd::HwptPaging`:

```
struct HwptPaging {
  obj: Object,                      // base iommufd object
  domain: Arc<IommuDomain>,         // generic iommu_domain backed by per-vendor
  ioas: AtomicPtr<Ioas>,             // attached IOAS (None until first attach)
  enforce_cache_coherency: bool,
  attach_lock: Mutex<()>,
  attached_devs: Mutex<Vec<Arc<Device>>>,
  hwpt_item: ListEntry,              // links into ioas.hwpt_list
  msi_cookie: Option<KBox<MsiCookie>>,
  dirty_tracking_enabled: AtomicBool,
}

struct HwptNested {
  obj: Object,
  domain: Arc<IommuDomain>,           // per-vendor nested domain (1st-level user-managed + 2nd-level inherited from parent)
  parent: Arc<HwptPaging>,            // parent HWPT-paging providing L2
  attach_lock: Mutex<()>,
  attached_devs: Mutex<Vec<Arc<Device>>>,
}
```

Alloc flow `HwptPaging::alloc(ictx, ioas, idev, flags, immediate_attach, user_data)`:
1. Per-vendor `iommu_ops->domain_alloc_paging(...)` returns `iommu_domain *`.
2. Allocate HwptPaging struct.
3. If `IOMMU_HWPT_ALLOC_DIRTY_TRACKING` flag: per-vendor `iommu_ops->set_dirty_tracking(domain, true)`.
4. If `immediate_attach`: bind to IOAS via `attach_ioas(hwpt, ioas)`.
5. Insert into ictx.objects → return out_hwpt_id.

Alloc nested flow `HwptNested::alloc(ictx, parent, idev, flags, user_data)`:
1. Validate parent is HWPT-paging.
2. Per-vendor `iommu_ops->domain_alloc_nested(parent.domain, user_data)` reads per-vendor blob:
   - Intel VTD_S1: 1st-stage PT root from guest pagetable (gpa space, validated against parent's IOAS aperture).
   - AMD V2: similar.
   - ARM SMMUV3: STE entry layout from guest.
3. Allocate HwptNested struct + Arc::clone(&parent).
4. Insert into ictx.objects.

Attach IOAS `HwptPaging::attach_ioas(hwpt, ioas)`:
1. Take `ioas.mutex` + `hwpt.attach_lock`.
2. For each IoptArea in ioas.iopt.area_list:
   - `iopt_pages_fill_xarray(area.pages, &xarr)` → pfn array.
   - `iommu_map_pages(hwpt.domain, area.iova_start, &xarr, area.length, area.iommu_prot, GFP_ATOMIC)`.
3. Add hwpt to ioas.hwpt_list (RCU-protected).
4. `hwpt.ioas = Some(ioas)`.

`Ctx::handle_hwpt_invalidate(cmd)` (nested HWPT):
1. Look up nested HWPT.
2. Parse `data_uptr` blob: per-vendor invalidate descriptor list.
3. Per-vendor `iommu_ops->cache_invalidate_user(hwpt.domain, &uptr_descriptors)`:
   - Intel: per-descriptor build QI invalidate descriptor + submit to per-IOMMU QI.
   - AMD: per-descriptor invalidate command via dedicated per-IOMMU command queue.
4. Wait for completion (Intel: QI Wait descriptor; AMD: per-cmd completion bit).
5. Return success.

`Ctx::handle_hwpt_set_dirty_tracking(cmd)`:
1. Look up paging HWPT.
2. `iommu_ops->set_dirty_tracking(hwpt.domain, cmd.flags & IOMMU_HWPT_DIRTY_TRACKING_ENABLE)`.
3. Per-vendor: walk all SPTEs in domain's pagetable + set/clear D-bit-tracking-enable.

`Ctx::handle_hwpt_get_dirty_bitmap(cmd)`:
1. Look up paging HWPT.
2. Validate user-supplied bitmap shape (page-size, length, base IOVA).
3. `iopt_read_and_clear_dirty_data(&hwpt.ioas.iopt, hwpt, &bitmap, range, flags)`:
   - For each IoptArea in range: per-vendor `iommu_ops->read_and_clear_dirty(domain, area.iova, area.length, &bitmap)`.
4. Copy bitmap back to userspace.

`Ctx::handle_hwpt_replace(cmd)`:
1. Look up old + new HWPT + device.
2. `iommu_attach_device_pasid(new.domain, dev, pasid)` (replaces old atomically per-spec).
3. Update device's per-iommufd-context attach record.
4. Detach from old.

Destroy `HwptPaging::Drop`:
1. `attach_lock` held: assert `attached_devs.is_empty()`; assert no nested HWPT references this as parent.
2. Detach from IOAS via `detach_ioas`:
   - Walk ioas.iopt.area_list; iommu_unmap from hwpt.domain.
   - Remove from ioas.hwpt_list (RCU-defer).
3. `iommu_domain_free(hwpt.domain)` (per-vendor `iommu_ops->free`).

Destroy `HwptNested::Drop`:
1. `attach_lock` held: assert `attached_devs.is_empty()`.
2. `iommu_domain_free(hwpt.domain)`.
3. `Arc::drop(&hwpt.parent)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `hwpt_no_uaf` | UAF | `Arc<HwptPaging>` outlives all attached devices + nested-children; destroy refused while refs exist. |
| `data_uptr_validated` | INVARIANT | per-vendor `domain_alloc_nested` validates entire `data_uptr` against expected layout + spec-required fields. |
| `replace_atomicity` | ATOMICITY | `IOMMU_HWPT_REPLACE` swaps device's HWPT atomically — device DMA either uses old or new, never partial. |
| `dirty_bitmap_no_oob` | OOB | user-supplied bitmap range bounds-checked against IOAS aperture; per-page bit indexed within bitmap_len. |

### Layer 2: TLA+

`models/iommu/nested_invalidate.tla` (parent-declared): proves nested HWPT — guest-managed L1 pagetable + host-managed L2 pagetable + IOMMU_HWPT_INVALIDATE propagation: every guest-issued invalidate eventually reaches IOMMU TLB before bound device's next DMA.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `HwptPaging::alloc` post: returned hwpt has hwpt.domain valid; hwpt.ioas == ioas if immediate_attach else None | `HwptPaging::alloc` |
| `HwptNested::alloc` post: hwpt.parent is HWPT-paging; per-vendor data validated against domain's spec | `HwptNested::alloc` |
| `IOMMU_HWPT_INVALIDATE` post: per-vendor cache-invalidate completed; subsequent device DMA observes invalidation | `Ctx::handle_hwpt_invalidate` |
| `IOMMU_HWPT_GET_DIRTY_BITMAP` post: returned bitmap reflects per-page dirty state at time of call; bits cleared in HW (read-and-clear semantics) | `Ctx::handle_hwpt_get_dirty_bitmap` |

### Layer 4: Verus/Creusot functional

`HwptPaging::alloc(ioas) → attach VFIO device → device DMA at iova → guest accesses iova` round-trip equivalence: per-IOAS mapping → HWPT installs → device-DMA-translated-by-HWPT-domain → host-physical-page accessed.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iommufd-hwpt specific reinforcement:

- **Per-vendor `data_uptr` validation strict** — every byte of guest-supplied nested-HWPT description validated against per-vendor expected layout BEFORE per-vendor domain_alloc_nested invoked; defense against driver crash from invalid nested data.
- **HWPT destroy with attached refs refused** — defense against UAF on dependent device.
- **Per-HWPT attached-device list under attach_lock** — defense against concurrent attach + replace racing.
- **Dirty-tracking enable/disable per-vendor atomic** — per-vendor SLADE/HWDirty bit transition observed atomically by HW; defense against partial-enable causing missed dirties.
- **Replace operation atomic** — single device's HWPT swap visible in single instant to HW; defense against partial-replace exposing device DMA to two domains.
- **Per-vendor cache-invalidate completion-wait** — IOMMU_HWPT_INVALIDATE ioctl returns only after per-vendor invalidate fully completes; defense against userspace assuming invalidate done before HW actually drained.
- **HWPT-nested parent reference held via Arc** — parent HwptPaging cannot be destroyed while child HwptNested references it.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- iommufd_main.c top-level dispatch (covered in `iommufd-main.md` Tier-3)
- IOAS internals (covered in `iommufd-ioas.md` Tier-3)
- Per-vendor domain_alloc_user impl (covered in `intel-nested.md` + `amd-nested.md` future Tier-3s)
- VIOMMU + VDEVICE (covered in `iommufd-viommu.md` future Tier-3)
- Eventq for PRI faults (covered in `iommufd-eventq.md` future Tier-3)
- 32-bit-only paths
- Implementation code
