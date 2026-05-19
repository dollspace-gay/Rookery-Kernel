# Tier-3: drivers/iommu/iommufd/viommu.c — Virtual IOMMU object + virtual device for nested-guest IOMMU emulation

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iommufd/viommu.c
  - drivers/iommu/iommufd/iommufd_private.h
  - include/uapi/linux/iommufd.h (IOMMU_VIOMMU_*, IOMMU_VDEVICE_*)
-->

## Summary

The VIOMMU (Virtual IOMMU) object is iommufd's representation of a virtual IOMMU instance exposed to a guest VM. Used by qemu/cloud-hypervisor when guest sees a virtual IOMMU (e.g., guest Linux booting with `intel_iommu=on` even though running in a VM): qemu allocates VIOMMU object → guest's virtual IOMMU driver issues commands → qemu intercepts → translates → VIOMMU bridges to host iommufd ops on actual HW.

VDEVICE (Virtual Device) is the per-virtual-PCIe-device companion — represents a guest-visible PCIe device with virtual BDF (bus/device/function) coordinates that map to a real underlying iommufd device. Used for PCIe-PASID virtualization: guest assigns PASID X to its virtual device → VDEVICE translates to real-PASID X' on real device for HW IOMMU programming.

This Tier-3 covers `drivers/iommu/iommufd/viommu.c` (~430 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommufd_viommu` | per-VIOMMU object | `kernel::iommu::iommufd::Viommu` |
| `struct iommufd_vdevice` | per-virtual-device object | `kernel::iommu::iommufd::Vdevice` |
| `iommufd_viommu_alloc_ioctl(ictx, cmd)` | IOMMU_VIOMMU_ALLOC ioctl | `Ctx::handle_viommu_alloc` |
| `iommufd_viommu_destroy(obj)` | destroy VIOMMU | `Viommu::destroy` (Drop) |
| `iommufd_vdevice_alloc_ioctl(ictx, cmd)` | IOMMU_VDEVICE_ALLOC ioctl | `Ctx::handle_vdevice_alloc` |
| `iommufd_vdevice_destroy(obj)` | destroy VDEVICE | `Vdevice::destroy` (Drop) |
| `iommufd_viommu_set_vdev_id(viommu, vdev, virt_id)` | per-VIOMMU vdev-id table install | `Viommu::set_vdev_id` |
| `iommufd_viommu_unset_vdev_id(viommu, vdev)` | uninstall | `Viommu::unset_vdev_id` |
| `iommufd_vdevice_lookup_id(viommu, virt_id, &vdev)` | vdev lookup by virt_id (used during cmd intercept) | `Viommu::lookup_vdev_id` |
| `iommufd_viommu_invalidate_user(viommu, cmd)` | per-vIOMMU TLB invalidate (forward to real HW HWPT) | `Viommu::invalidate_user` |
| `iommufd_viommu_pri_response(viommu, cmd)` | guest-supplied PRI response forward to real device | `Viommu::pri_response` |

## Compatibility contract

REQ-1: `IOMMU_VIOMMU_ALLOC` UAPI byte-identical:
- Input: `flags`, `viommu_type` (per-vendor: VTD_S2 / ARM_SMMUV3 / AMD_V2), `dev_id` (real device backing), `hwpt_id` (parent HWPT-paging providing L2 translation).
- Output: `out_viommu_id`.

REQ-2: VIOMMU is per-vendor; the `viommu_type` matches the host IOMMU's vendor. Cross-vendor virtualization (e.g., AMD vIOMMU on Intel HW) not supported.

REQ-3: VIOMMU's parent HWPT-paging provides the L2 (host-level) IOMMU translation; guest-managed pagetables become L1 (nested) HWPTs allocated under this VIOMMU.

REQ-4: `IOMMU_VDEVICE_ALLOC` UAPI byte-identical:
- Input: `viommu_id`, `dev_id` (real device), `virt_id` (guest-visible BDF or guest-visible PASID).
- Output: `out_vdevice_id`.
- Allocates VDEVICE; installs in viommu's vdev-id table indexed by virt_id.

REQ-5: Per-VIOMMU vdev-id table: hash-map from virt_id (32-bit; encodes guest BDF or guest PASID) → Vdevice. Lookup at command-intercept time.

REQ-6: `IOMMU_VIOMMU_INVALIDATE` UAPI: per-vendor invalidation descriptor list from guest; iommufd validates each descriptor's virt_id, looks up real device, translates virt_id → real_id, forwards to per-vendor HWPT invalidate.

REQ-7: `IOMMU_VIOMMU_PRI_RESPONSE` UAPI: guest-supplied page-response for PRI fault forwarded by VIOMMU eventq; iommufd validates virt_id, translates to real device, calls per-vendor `iommu_ops->page_response` on real HW.

REQ-8: VIOMMU destroy invariants: cannot be destroyed if any VDEVICE still attached to it; cannot if any nested HWPT still references it as parent.

REQ-9: VDEVICE destroy invariants: removed from VIOMMU's vdev-id table atomically; any in-flight PRI fault for this virt_id replied with INVALID before drop.

REQ-10: Per-vendor `iommu_ops->viommu_alloc(...)` callback: per-vendor type-specific VIOMMU init (e.g., Intel installs per-VIOMMU descriptor table; AMD configures nested-translation context).

## Acceptance Criteria

- [ ] AC-1: qemu nested-IOMMU test on Intel: guest with `intel_iommu=on` boots, sees virtual IOMMU; guest assigns PASID 5 to its virtual device; iommufd VDEVICE translates to real PASID; guest device DMA via PASID 5 routes correctly via nested L1 (guest) + L2 (host) translation.
- [ ] AC-2: `IOMMU_VDEVICE_ALLOC` per-virt-id installs correctly; lookup returns expected real device.
- [ ] AC-3: `IOMMU_VIOMMU_INVALIDATE` translation: guest issues invalidate for virt_id X; real-device invalidate observable on HW for real_id X' before guest device's next DMA.
- [ ] AC-4: VIOMMU destroy refused with attached VDEVICE: -EBUSY until all VDEVICEs explicitly destroyed.
- [ ] AC-5: PRI response forwarding: guest receives PRI fault for virt-PASID, sends response → host forwards to real device.
- [ ] AC-6: Cross-vendor rejection: attempt VIOMMU_ALLOC with VTD_S2 type on AMD HW returns -EOPNOTSUPP.
- [ ] AC-7: kselftest iommufd nested + viommu subset passes.

## Architecture

`Viommu` lives in `kernel::iommu::iommufd::Viommu`:

```
struct Viommu {
  obj: Object,
  type_: ViommuType,                    // VTD_S2 / ARM_SMMUV3 / AMD_V2
  iommu_dev: Arc<IommuDevice>,           // real backing IOMMU
  hwpt: Arc<HwptPaging>,                 // parent HWPT-paging providing L2
  vdevs: Mutex<XArray<Arc<Vdevice>>>,    // virt_id → Vdevice
  ops: &'static dyn ViommuOps,           // per-vendor vtable
  vdev_lock: SpinLock<()>,
}

struct Vdevice {
  obj: Object,
  viommu: Arc<Viommu>,
  dev: Arc<Device>,                      // real backing device
  virt_id: u32,                          // guest-visible BDF or PASID
  vdev_lock: SpinLock<()>,
}
```

`Ctx::handle_viommu_alloc(cmd)`:
1. Validate cmd.viommu_type matches host IOMMU's vendor.
2. Look up parent HWPT-paging by cmd.hwpt_id.
3. Look up dev_id → IommuDevice.
4. Per-vendor `iommu_ops->viommu_alloc(iommu_dev, hwpt.domain, viommu_type)` → returns viommu domain or per-vendor state.
5. Allocate Viommu struct.
6. Insert into ictx.objects → return out_viommu_id.

`Ctx::handle_vdevice_alloc(cmd)`:
1. Look up viommu_id, dev_id.
2. Validate dev backed by viommu's same iommu_dev.
3. Allocate Vdevice struct: `viommu = Arc::clone(&viommu)`, `dev = Arc::clone(&dev)`, `virt_id = cmd.virt_id`.
4. Take viommu.vdev_lock.
5. `viommu.vdevs.alloc(virt_id, vdev)`: insert into xarray; if virt_id already present, return -EEXIST.
6. Drop vdev_lock.
7. Insert into ictx.objects → return out_vdevice_id.

`Ctx::handle_viommu_invalidate(cmd)`:
1. Look up Viommu.
2. Parse cmd.uptr's invalidate descriptor list per-vendor format.
3. For each descriptor:
   - Extract virt_id (per-vendor encoding).
   - `viommu.lookup_vdev_id(virt_id, &vdev)` → real device.
   - Translate descriptor's virt_id field to real device's source-id (per-vendor).
   - Forward to per-vendor `iommu_ops->cache_invalidate_user(viommu.hwpt.domain, &translated_descriptor)`.
4. Wait for per-vendor completion.
5. Return success / -EINVAL on malformed.

`Ctx::handle_viommu_pri_response(cmd)`:
1. Look up Viommu.
2. Validate cmd.virt_id against vdev table.
3. Translate to real device.
4. Per-vendor `iommu_ops->page_response(real_dev, &cmd.response)`.

VIOMMU destroy `Viommu::Drop`:
1. Assert `vdevs.is_empty()` (else -EBUSY at destroy ioctl).
2. Per-vendor `iommu_ops->viommu_destroy(...)` cleans up per-VIOMMU state.
3. Drop Arc<HwptPaging> + Arc<IommuDevice>.

VDEVICE destroy `Vdevice::Drop`:
1. Take viommu.vdev_lock.
2. `viommu.vdevs.remove(virt_id)`.
3. Drop vdev_lock.
4. Drop Arc<Viommu> + Arc<Device>.

Per-vendor backends:
- Intel VTD: VIOMMU_TYPE_VTD_S2 — guest's 1st-stage pagetable installed as nested VTD-S1 HWPT; VIOMMU manages PASID-table-entry allocations for guest-PASID space.
- ARM SMMUV3: Stream Table Entry (STE) virtualization; per-virt-RID maps to real-RID via vdev table.
- AMD V2: similar to Intel via guest-managed L1 + host-managed L2.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `viommu_no_uaf` | UAF | `Arc<Viommu>` outlives all attached VDEVICEs + nested HWPTs; destroy refused while refs exist. |
| `vdev_id_no_collision` | UNIQUENESS | per-VIOMMU vdev-id xarray ensures unique virt_id; duplicate alloc returns -EEXIST. |
| `cross_vendor_rejected` | TYPE-CHECK | viommu_type must match host IOMMU vendor; mismatch rejected at alloc. |
| `invalidate_translation_no_oob` | OOB | per-vendor descriptor field bounds-checked before virt_id lookup. |

### Layer 2: TLA+

`models/iommu/nested_invalidate.tla` (parent-declared): proves nested HWPT — guest-managed L1 pagetable + host-managed L2 pagetable + invalidate propagation: every guest-issued invalidate eventually reaches IOMMU TLB before bound device's next DMA.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ctx::handle_viommu_invalidate` post: per-vendor cache-invalidate completed for all translated descriptors; subsequent device DMA observes invalidation | `Ctx::handle_viommu_invalidate` |
| Per-VIOMMU vdev-id table consistent with VDEVICE objects: every Vdevice in ictx.objects with viommu pointer is in viommu.vdevs xarray | VDEVICE alloc/destroy |
| `Viommu::lookup_vdev_id` post: returned vdev (if Some) has matching virt_id + still attached to viommu | `Viommu::lookup_vdev_id` |

### Layer 4: Verus/Creusot functional

`Guest VIOMMU command → qemu intercept → IOMMU_VIOMMU_INVALIDATE / IOMMU_VIOMMU_PRI_RESPONSE → virt_id translation → real device → HW IOMMU` round-trip equivalence: any guest-side IOMMU command results in equivalent real-HW operation within bounded time.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iommufd-viommu specific reinforcement:

- **Per-VIOMMU vdev-id table size cap** — bounded (default 64K vdevs per VIOMMU); defense against per-VIOMMU vdev-flood DoS.
- **Cross-vendor VIOMMU rejected at alloc** — defense against driver crash from invalid vendor-specific data interpreted by wrong vendor backend.
- **Per-virt-id translation strict** — every invalidate descriptor's virt_id validated against vdev table before forward; defense against guest forging invalidates for non-owned devices.
- **VIOMMU/VDEVICE destroy refused with attached refs** — defense against UAF.
- **Per-VIOMMU pagetable parent immutable** — parent HWPT-paging cannot be replaced after VIOMMU alloc; defense against parent-swap inconsistency.
- **PRI response source validation** — per-cmd virt_id matched against vdev's real device's actual PRI source; defense against forging PRI response for arbitrary device.
- **Per-VIOMMU invalidate descriptor count cap** — bounded (default 256/batch); defense against invalidate-flood DoS.
- **Cross-context VDEVICE attach refused** — VDEVICE bound to viommu inherits viommu's ictx; cross-ctx attempts rejected.

## Grsecurity/PaX-style Reinforcement

Hardened-policy supplement above baseline `## Hardening`. The VIOMMU/VDEVICE objects are the iommufd interface by which a guest VM directly drives a host IOMMU (via qemu) — they are the highest-privilege virtualization surface in the iommu subsystem and receive the strictest grsec/PaX layering.

- **PAX_USERCOPY** on per-vendor invalidate-descriptor blobs and PRI-response copies from userspace.
- **PAX_KERNEXEC** on VIOMMU/VDEVICE alloc/destroy/invalidate paths and per-vendor `ViommuOps` dispatch (RO post-init).
- **PAX_RANDKSTACK** on `IOMMU_VIOMMU_*` and `IOMMU_VDEVICE_*` ioctl entry chains.
- **PAX_REFCOUNT** on `Viommu`, `Vdevice`, attached HWPT-paging parent, and vdev-id xarray entries.
- **PAX_MEMORY_SANITIZE** zeroes `Vdevice` and `Viommu` storage on destroy + clears vdev-id xarray slots.
- **PAX_UDEREF** on per-vendor descriptor-array copy_from_user (one of the largest blob ingest paths in iommufd).
- **PAX_RAP/kCFI** on `ViommuOps` vtable + per-vendor `viommu_alloc`/`viommu_destroy`/`cache_invalidate_user`/`page_response` indirect calls.
- **GRKERNSEC_HIDESYM** hides `Viommu` pointers, vdev-id xarray base, and per-vendor backing state.
- **GRKERNSEC_DMESG** restricts viommu-related decoded printouts to CAP_SYSLOG.
- **CAP_SYS_ADMIN-in-init-userns strict** for nested-translation VIOMMU alloc — nested-userns categorically blocked from VIOMMU/VDEVICE creation.
- **GRKERNSEC_DMA strict-mode** — VIOMMU vdev-id table starts empty; every entry requires explicit VDEVICE_ALLOC ioctl.
- **ATS/PASID gating** — VDEVICE refused for backing real devices missing verified PCIe ATS+PASID caps.
- **VT-d PRQ rate-limit** and per-VIOMMU invalidate-batch rate-limit layered to defang guest-driven QI saturation.
- **DMAR/IVRS ACPI signature verify** required before honoring per-vendor nested capability for VIOMMU alloc.
- **VFIO-compat path refused** for VIOMMU consumers under hardened policy.
- **Cross-vendor VIOMMU type refused** at alloc + module init (no AMD VIOMMU on Intel host, etc.).
- **Per-VIOMMU vdev-id cap** enforced per-userns (not just per-VIOMMU) to defang multi-guest exhaustion.

Rationale: VIOMMU + VDEVICE expose direct guest control over real IOMMU invalidation and PRI response, mediated only by virt_id translation. A misvalidated descriptor lets a guest invalidate the host's TLB or spoof PRI for non-owned devices. Hardened Rookery enforces every cap-revalidation, namespace gate, and rate-limit independently.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-vendor VIOMMU implementation (covered in `intel-iommu.md` + `amd-iommu.md` + `arm-smmuv3.md` future Tier-3s)
- Eventq for VIOMMU vEVENTs (covered in `iommufd-eventq.md` Tier-3 § VEVENT path)
- iommufd_main top-level dispatch (covered in `iommufd-main.md` Tier-3)
- 32-bit-only paths
- Implementation code
