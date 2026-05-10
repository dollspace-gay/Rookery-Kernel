---
title: "Tier-3: drivers/iommu/iommufd/{main,driver,device}.c — iommufd UAPI dispatch + per-context object table + per-device bind"
tags: ["tier-3", "drivers-iommu", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The iommufd subsystem — `/dev/iommu` chardev replacing the legacy VFIO type1 container interface. Designed for: granular IOAS (IO Address Space) creation, nested HWPT (Hardware Page Table) for guest IOMMU virtualization (qemu/cloud-hypervisor exposing virtual IOMMU to guest), per-device fine-grained binding (cross-VM device passthrough), per-IOAS dirty-page tracking for live migration. Modern userspace (qemu w/ `iommufd=on`, cloud-hypervisor, dpdk vfio-pci backend) prefers iommufd over legacy VFIO; legacy compat shim (`vfio_compat.c`) preserves old code paths during transition.

This Tier-3 covers `iommufd/main.c` (~800 lines: chardev fops + ioctl dispatch + per-fd object table), `driver.c` (~300 lines: per-vendor IOMMU driver registration), `device.c` (~1700 lines: per-device bind/attach lifecycle).

### Acceptance Criteria

- [ ] AC-1: qemu `-object iommufd,id=iommufd0 -device vfio-pci,host=0000:01:00.0,iommufd=iommufd0` boots Linux guest with NVMe passthrough.
- [ ] AC-2: cloud-hypervisor with iommufd backend works.
- [ ] AC-3: `IOMMU_IOAS_MAP` + DMA via attached vfio-device + `IOMMU_IOAS_UNMAP` round-trip works without leaks.
- [ ] AC-4: Nested HWPT test: qemu with `-device intel-iommu` + L1 guest with assigned vfio-pci device + L2 guest sees device with own IOMMU.
- [ ] AC-5: Dirty-page tracking test: live-migration with `IOMMU_HWPT_GET_DIRTY_BITMAP` shows correct dirty bits during sustained DMA.
- [ ] AC-6: VFIO compat test: legacy qemu code path with VFIO_IOMMU_MAP_DMA on iommufd-bound fd works.
- [ ] AC-7: `IOMMU_IOAS_COPY` test: copy mapping from IOAS-A to IOAS-B; both work; underlying page-pin shared.
- [ ] AC-8: kselftest `tools/testing/selftests/iommu/` passes.

### Architecture

`Ctx` lives in `kernel::iommu::iommufd::Ctx`:

```
struct Ctx {
  refcount: Refcount,
  file: NonNull<File>,
  objects: Mutex<xa::XArray<Box<dyn Object>>>,  // object id → object
  ioctl_lock: Mutex<()>,
  vfio_compat_state: AtomicPtr<VfioCompat>,      // for legacy compat
  account_mode: AccountMode,                      // IOMMU_IOAS_MM_LOCK / KERNEL_LIMIT
}
```

Generic `Object` trait:

```
trait Object: Send + Sync {
  fn type_id(&self) -> ObjectType;  // IOAS / HWPT_PAGING / HWPT_NESTED / DEVICE / VIOMMU / VDEVICE / FAULT
  fn destroy(self: Box<Self>) -> Result<(), Errno>;
  // ...
}
```

`/dev/iommu` open flow:
1. `iommufd_fops::open(inode, file)` → `Ctx::new(file)`:
   - Allocate Ctx struct.
   - Init empty `objects` xarray.
   - file->private_data = Ctx pointer.

`Ctx::ioctl(file, cmd, arg)` dispatch:
1. Look up cmd in ioctl-table (sized by `_IOC_NR(cmd)`).
2. Each handler validates per-ioctl struct size + flags + zero-pad enforcement (forward-compat).
3. Execute handler under per-ctx ioctl_lock (most ioctls serialize; some allow concurrent).
4. Return result code.

`IOMMU_IOAS_ALLOC` handler:
1. Validate user-arg struct.
2. `Object::alloc(ictx, ObjectType::IOAS, sizeof(IoasObject))`.
3. Initialize IoasObject: empty io_pagetable + per-IOAS access mode.
4. Insert into ictx.objects via xa_alloc → returns 32-bit id.
5. Copy id back to user-arg.

`IOMMU_HWPT_ALLOC` handler:
1. Validate user-arg.
2. Look up `pt_id`:
   - If type == NONE: pt_id is IOAS id; HWPT type = paging.
   - Else: pt_id is parent HWPT id; HWPT type = nested.
3. Look up `dev_id`: per-device object owns the bind state.
4. `Object::alloc(ictx, HWPT_PAGING or HWPT_NESTED, sizeof(HwptObject))`.
5. Per-vendor `iommu_ops->domain_alloc(...)` → real `iommu_domain`.
6. For nested: per-vendor `domain_alloc_nested(...)` reads vendor-specific `data_uptr` blob.
7. Insert HwptObject into objects table.
8. Return hwpt_id.

`IOMMU_IOAS_MAP[2]` handler:
1. Validate user-arg + iova-bounds.
2. Look up IOAS object via ioas_id.
3. Pin user pages via `pin_user_pages(...)` → per-page refcount.
4. For each attached HWPT (RCU-protected list): `iommu_map(domain, iova, paddr, length, prot)`.
5. Insert pinned-page list into IOAS pagetable for tracking.
6. Account against current process's RLIMIT_MEMLOCK or per-ctx kernel-limit.

`IOMMU_IOAS_UNMAP`:
1. Look up IOAS.
2. For each attached HWPT: `iommu_unmap(domain, iova, length)`.
3. Walk page-pin list for iova range; `unpin_user_pages(...)`.
4. Free pagetable entries.

Per-device bind `Device::bind_iommufd(ictx, dev, &out_id)`:
1. Allocate per-device object.
2. `iommu_probe_device(...)` ensures device has iommu_group.
3. Verify exclusive ownership (no other ictx has this device bound).
4. Insert into ictx.objects.
5. Return out_id.

Per-device attach `Device::attach_hwpt(idev, hwpt)`:
1. Look up device + HWPT.
2. `iommu_attach_group(hwpt.domain, idev.group)` — IOMMU API call.
3. Add idev to hwpt's device-list (RCU-protected).

Per-fd close `iommufd_fops::release`:
1. Walk objects in topological-sort order: vdevice → viommu → device → hwpt → ioas.
2. For each: `Object::destroy(obj)` → IOAS unmap walks pages + unpins; HWPT detach all devices + free domain; device unbind from iommu_group.
3. Free ctx struct.

### Out of Scope

- IOAS internals (covered in `drivers/iommu/iommufd-ioas.md` future Tier-3)
- HWPT internals (covered in `drivers/iommu/iommufd-hwpt.md` future Tier-3)
- Eventq impl (covered in `drivers/iommu/iommufd-eventq.md` future Tier-3)
- VIOMMU object (covered in `drivers/iommu/iommufd-viommu.md` future Tier-3)
- VFIO compat shim (covered in `drivers/iommu/iommufd-vfio-compat.md` future Tier-3)
- Per-vendor IOMMU drivers (covered in `intel-iommu.md`, `amd-iommu.md` future Tier-3s)
- Per-driver iommu_ops impl (covered in `iommu-core.md` parent)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommufd_ctx` | per-fd context (object table + per-fd state) | `kernel::iommu::iommufd::Ctx` |
| `struct iommufd_object` | base type for every iommufd object (IOAS / HWPT / device / vdevice / viommu / ...) | `kernel::iommu::iommufd::Object` (trait) |
| `iommufd_fops` | `/dev/iommu` chardev file_operations | `iommufd::Fops` |
| `iommufd_ioctl(file, cmd, arg)` | top-level ioctl dispatch | `Ctx::ioctl` |
| `IOMMU_DESTROY` ioctl | generic object destroy | `Ctx::handle_destroy` |
| `IOMMU_IOAS_ALLOC` | alloc new IOAS | `Ctx::handle_ioas_alloc` |
| `IOMMU_IOAS_IOVA_RANGES` | enumerate per-IOAS aperture | `Ctx::handle_ioas_iova_ranges` |
| `IOMMU_IOAS_ALLOW_IOVAS` | constrain per-IOAS aperture | `Ctx::handle_ioas_allow_iovas` |
| `IOMMU_IOAS_MAP[2]` | map iova→pages into IOAS | `Ctx::handle_ioas_map` |
| `IOMMU_IOAS_COPY` | copy iova range from one IOAS to another | `Ctx::handle_ioas_copy` |
| `IOMMU_IOAS_UNMAP` | unmap iova range | `Ctx::handle_ioas_unmap` |
| `IOMMU_IOAS_MAP_FILE` | map iova → file (for inode-backed IOAS) | `Ctx::handle_ioas_map_file` |
| `IOMMU_IOAS_CHANGE_PROCESS` | change owning process for backing pages | `Ctx::handle_ioas_change_process` |
| `IOMMU_IOAS_MAP_DMABUF` | map dmabuf into IOAS | `Ctx::handle_ioas_map_dmabuf` |
| `IOMMU_GET_HW_INFO` | per-device IOMMU hw info | `Ctx::handle_get_hw_info` |
| `IOMMU_HWPT_ALLOC` | alloc hardware page table (paging or nested) | `Ctx::handle_hwpt_alloc` |
| `IOMMU_HWPT_INVALIDATE` | selective TLB invalidate (nested) | `Ctx::handle_hwpt_invalidate` |
| `IOMMU_HWPT_GET_DIRTY_BITMAP` / `_SET_DIRTY_TRACKING` | dirty-page tracking | `Ctx::handle_hwpt_dirty_*` |
| `IOMMU_VFIO_IOAS` | legacy VFIO ↔ iommufd IOAS bridge | `Ctx::handle_vfio_ioas` |
| `IOMMU_VIOMMU_ALLOC` | alloc virtual IOMMU object (nested guest IOMMU emulation) | `Ctx::handle_viommu_alloc` |
| `IOMMU_VDEVICE_ALLOC` | alloc virtual device object | `Ctx::handle_vdevice_alloc` |
| `IOMMU_FAULT_QUEUE_ALLOC` | alloc PRI fault eventq | `Ctx::handle_fault_queue_alloc` |
| `IOMMU_OPTION` | per-context option set | `Ctx::handle_option` |
| `iommufd_object_alloc(ictx, type, size)` | allocate generic object + insert into per-ctx table | `Object::alloc` |
| `iommufd_object_destroy_user(ictx, obj_id)` | destroy from userspace ioctl | `Object::destroy_user` |
| `iommufd_get_object(ictx, obj_id, type)` / `_put_object` | per-ctx object lookup w/ refcount | `Ctx::get_object` / `_put_object` |
| `iommufd_device_bind(ictx, dev, &id)` | per-device bind to ictx | `Device::bind_iommufd` |
| `iommufd_device_unbind(idev)` | inverse | `Device::unbind_iommufd` |
| `iommufd_device_attach(idev, hwpt)` / `_detach(idev)` | per-device attach to HWPT | `Device::attach_hwpt` / `_detach_hwpt` |

### compatibility contract

REQ-1: `/dev/iommu` chardev w/ `iommufd_fops` registered; mode 0660 owned by root:root (or root:kvm); ioctl `IOMMU_*` dispatch byte-identical to upstream UAPI.

REQ-2: Per-fd object table: every alloc returns 32-bit object id assigned via xarray; lookup-by-id under per-ctx mutex; refcount per object via `Refcount`.

REQ-3: `IOMMU_DESTROY` accepts any object id; verifies type-correct destroy (HWPT cannot be destroyed if still attached to devices, IOAS cannot be destroyed if HWPT references it).

REQ-4: `IOMMU_IOAS_ALLOC` returns IOAS object id; per-IOAS empty mapping table.

REQ-5: `IOMMU_IOAS_MAP[2]` post: `[iova, iova+length)` mapped to user-virtual range starting at `user_va`; subsequent device DMA via attached HWPT translates correctly.

REQ-6: `IOMMU_IOAS_UNMAP` post: `[iova, iova+length)` no longer mapped; subsequent device DMA produces fault delivered via IOMMU fault eventq.

REQ-7: `IOMMU_IOAS_COPY` semantics: copies mapping from src IOAS to dst IOAS (sharing the underlying page-pin); useful for cross-IOAS sharing without re-pinning userspace pages.

REQ-8: `IOMMU_HWPT_ALLOC` accepts `data_type`: NONE (default paging) / VTD_S1 (Intel nested 1st stage) / ARM_SMMUV3 / AMD_V2; `pt_id` references an IOAS for paging or another HWPT for nested.

REQ-9: `IOMMU_HWPT_INVALIDATE` for nested HWPT: per-vendor invalidation descriptor format; processed in batch.

REQ-10: `IOMMU_HWPT_GET_DIRTY_BITMAP` / `_SET_DIRTY_TRACKING`: per-IOAS dirty-page tracking for live migration; per-vendor IOMMU dirty-bit (Intel SLADE, AMD HWDirty) used when available, software-tracked otherwise.

REQ-11: `IOMMU_VFIO_IOAS` legacy compat: VFIO_IOMMU_GET_INFO / _MAP_DMA / _UNMAP_DMA on iommufd-fd works via the compat shim (cross-ref `vfio_compat.c`).

REQ-12: Per-device bind: `iommufd_device_bind(ictx, dev, &id)` allocates per-device object, assigns id; subsequent attach to HWPT installs the device into HWPT's per-domain device list.

REQ-13: Per-fd close (`iommufd_fops::release`): walks object table, destroys all in dependency order (vdevices → viommus → devices → HWPTs → IOAS); guarantees no orphaned pinned pages.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `obj_no_uaf` | UAF | per-object Refcount; destroy waits for refcount==0; per-fd close walks in topo-sort order. |
| `obj_id_no_collision` | UNIQUENESS | xa_alloc guarantees unique id; per-ctx object table mediated. |
| `iova_no_overflow` | OVERFLOW | iova + length checked against u64 + per-IOAS aperture max. |
| `page_pin_no_leak` | LEAK | every IOMMU_IOAS_MAP page-pin matched by either IOMMU_IOAS_UNMAP or per-fd close-time auto-unmap. |

### Layer 2: TLA+

`models/iommu/nested_invalidate.tla` (parent-declared): proves nested HWPT — guest-managed L1 pgtable + host-managed L2 pgtable + IOMMU_HWPT_INVALIDATE propagation: every guest-issued invalidate eventually reaches IOMMU TLB before bound device's next DMA.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Object::alloc` post: object inserted into ctx.objects with refcount==1; type field correctly set | `Object::alloc` |
| `IOMMU_DESTROY` invariant: HWPT cannot be destroyed if still attached to any device; IOAS cannot be destroyed if any HWPT references it | `Ctx::handle_destroy` |
| `IOMMU_IOAS_MAP` post: every iova in `[iova, iova+length)` translates to mapped paddr in every attached HWPT | `Ctx::handle_ioas_map` |
| Per-fd close invariant: post-close, no pinned pages remain attributed to this ctx's IOASes | `iommufd_fops::release` |

### Layer 4: Verus/Creusot functional

`IOMMU_IOAS_MAP(ioas, iova, len, va) → device-DMA(iova) → IOMMU_IOAS_UNMAP(ioas, iova, len)` round-trip equivalence: after MAP, device DMA to iova reads/writes the user-VA pages; after UNMAP, device DMA to iova produces fault. Encoded as Verus invariant on the IOAS abstract page-table state.

### hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iommufd-main specific reinforcement:

- **Per-ctx object table size cap = 64K** in v0 (configurable via Kconfig); defense against per-fd object-flood DoS.
- **Per-IOAS page-pin RLIMIT_MEMLOCK accounting** — per-process locked-memory cap enforced; defense against unprivileged process pinning all of host RAM via iommufd.
- **`/dev/iommu` open requires CAP_SYS_RAWIO + LSM mediation** — passthrough surface; gate it.
- **Per-IOAS reserved-region MSI-X enforcement** — IOMMU_IOAS_MAP rejected for IOVA in MSI-X window [0xFEE00000, 0xFEEFFFFF] (cross-ref `iommu-core.md` § Hardening).
- **Per-device bind exclusive** — only one iommufd ctx can bind a device at a time; defense against cross-ctx device-state confusion.
- **Per-HWPT nested data validated against vendor capabilities** — `IOMMU_HWPT_ALLOC` with `data_type=VTD_S1` requires Intel host w/ VT-d nested support advertised; defense against driver crash from invalid nested data.
- **Per-fd close robust to partial object-graph** — failure during close-time destroy of one object doesn't abort cleanup of others; ensures no leaked pin/refcount.
- **IOMMU_VFIO_IOAS legacy shim rate-limited** — defense against legacy VFIO-API-flood from misbehaved userspace.
- **IOMMU_HWPT_INVALIDATE descriptor count cap** — per-batch invalidate descriptor count capped; defense against invalidate-flood DoS.

