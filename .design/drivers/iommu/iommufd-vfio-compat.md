# Tier-3: drivers/iommu/iommufd/vfio_compat.c — Legacy VFIO type1 container shim (transition window)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iommufd/vfio_compat.c
  - drivers/iommu/iommufd/iommufd_private.h
  - include/uapi/linux/vfio.h (VFIO_IOMMU_*)
-->

## Summary

The iommufd-side VFIO compatibility shim — when an existing `/dev/vfio/vfio` legacy container fd or `/dev/vfio/<group>` legacy group fd is used by qemu (with old "vfio-iommu-type1" code paths still active in distro builds), the kernel can transparently route those legacy ioctls to an iommufd-backed implementation under the hood. Two activation modes:

- **Implicit**: when CONFIG_IOMMUFD_VFIO_CONTAINER=y AND user opens `/dev/vfio/vfio`, kernel allocates an iommufd `iommufd_ctx` + auto-binds it as the legacy container. All legacy `VFIO_IOMMU_*` ioctls translate to iommufd-side IOAS operations.
- **Explicit**: user opens `/dev/iommu` (modern path), then issues `IOMMU_VFIO_IOAS` ioctl to associate an existing IOAS as the "default container" for legacy `VFIO_IOMMU_GET_INFO` / `_MAP_DMA` / `_UNMAP_DMA` calls on that fd.

This Tier-3 covers `drivers/iommu/iommufd/vfio_compat.c` (~540 lines): legacy ioctl translation + per-VFIO-API parameter mapping to modern iommufd internals.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `iommufd_vfio_ioctl(file, cmd, arg)` | top-level legacy VFIO ioctl dispatch | `Ctx::vfio_ioctl` |
| `iommufd_vfio_set_iommu(ictx, type)` | VFIO_SET_IOMMU translation | `Ctx::vfio_set_iommu` |
| `iommufd_vfio_check_extension(ictx, val)` | VFIO_CHECK_EXTENSION translation | `Ctx::vfio_check_extension` |
| `iommufd_vfio_get_api_version(...)` | VFIO_GET_API_VERSION (returns 0) | `Ctx::vfio_get_api_version` |
| `iommufd_vfio_iommu_get_info(ictx, arg)` | VFIO_IOMMU_GET_INFO translation | `Ctx::vfio_iommu_get_info` |
| `iommufd_vfio_map_dma(ictx, arg)` | VFIO_IOMMU_MAP_DMA translation | `Ctx::vfio_map_dma` |
| `iommufd_vfio_unmap_dma(ictx, arg)` | VFIO_IOMMU_UNMAP_DMA translation | `Ctx::vfio_unmap_dma` |
| `iommufd_vfio_dirty_pages(ictx, arg)` | VFIO_IOMMU_DIRTY_PAGES translation | `Ctx::vfio_dirty_pages` |
| `iommufd_get_compat_ioas(ictx)` | get-or-create per-ctx default IOAS for legacy ops | `Ctx::get_compat_ioas` |
| `iommufd_compat_set_ioas(ictx, ioas_id)` | set existing IOAS as compat target (IOMMU_VFIO_IOAS) | `Ctx::compat_set_ioas` |
| `iommufd_compat_clear_ioas(ictx)` | unset compat IOAS | `Ctx::compat_clear_ioas` |
| `iommufd_compat_get_ioas(ictx)` | get current compat IOAS | `Ctx::compat_get_ioas` |

## Compatibility contract

REQ-1: Legacy `/dev/vfio/vfio` chardev opens still work — `iommufd_vfio_ioctl` recognizes legacy ioctl numbers AND translates to modern iommufd ops.

REQ-2: `VFIO_GET_API_VERSION` returns `VFIO_API_VERSION` (0); userspace using legacy API sees expected version.

REQ-3: `VFIO_CHECK_EXTENSION(ext)` returns 1 for: `VFIO_TYPE1_IOMMU`, `VFIO_TYPE1v2_IOMMU`, `VFIO_NOIOMMU_IOMMU`, `VFIO_UNMAP_ALL`, `VFIO_UPDATE_VADDR` (when supported by underlying iommufd backend).

REQ-4: `VFIO_SET_IOMMU(VFIO_TYPE1_IOMMU)`: bound the legacy container to type1 IOMMU; iommufd lazily allocates compat IOAS on first MAP_DMA.

REQ-5: `VFIO_IOMMU_GET_INFO`: returns per-iommu info struct (`vfio_iommu_type1_info`) with iova_pgsizes from per-vendor `iommu_ops->pgsize_bitmap`, plus extended-info caps for IOVA range + dirty-tracking + migration-state.

REQ-6: `VFIO_IOMMU_MAP_DMA(struct vfio_iommu_type1_dma_map)` translation:
- Validate flags (READ / WRITE).
- Get-or-create compat IOAS via `iommufd_get_compat_ioas`.
- Translate to `Ioas::map`: `iova=arg.iova`, `length=arg.size`, `user_va=arg.vaddr`, `flags=READ|WRITE per arg.flags`.
- Return same status codes legacy expects (-ENOMEM / -EINVAL / -EBUSY).

REQ-7: `VFIO_IOMMU_UNMAP_DMA(struct vfio_iommu_type1_dma_unmap)` translation:
- Same as `Ioas::unmap` with arg.iova + arg.size.
- For VFIO_DMA_UNMAP_FLAG_GET_DIRTY_BITMAP: bitmap returned via dirty-tracking infrastructure.
- For VFIO_DMA_UNMAP_FLAG_ALL: walk all areas in compat IOAS.

REQ-8: `VFIO_IOMMU_DIRTY_PAGES(arg)` translation to `IOMMU_HWPT_GET_DIRTY_BITMAP` on compat IOAS's first attached HWPT.

REQ-9: `IOMMU_VFIO_IOAS` ioctl on `/dev/iommu` fd:
- `op=IOMMU_VFIO_IOAS_GET`: returns current compat IOAS id (or -ENOENT).
- `op=IOMMU_VFIO_IOAS_SET`: install given IOAS as compat.
- `op=IOMMU_VFIO_IOAS_CLEAR`: detach.

REQ-10: VFIO container attach/detach: per-vendor `vfio_iommu_driver_ops_*` hooks bridged to per-IOAS attach via underlying iommufd HWPT.

REQ-11: Legacy NOIOMMU mode: `VFIO_NOIOMMU_IOMMU` declines to install a real IOMMU domain; userspace uses CAP_SYS_RAWIO + DMA without isolation. Bridged via iommufd by allocating a special "noiommu" pseudo-IOAS that performs no-op map/unmap.

REQ-12: Performance: legacy translation overhead ≤ 2% vs native iommufd path; defense against pessimistic compat shim slowing critical workloads.

## Acceptance Criteria

- [ ] AC-1: Legacy qemu (no `iommufd=` arg) using `/dev/vfio/vfio` boots Linux guest with vfio-pci passthrough on Rookery — kernel transparently routes via vfio_compat.
- [ ] AC-2: `vfio-test` or libvfio-user test program: `VFIO_IOMMU_MAP_DMA` + DMA + `VFIO_IOMMU_UNMAP_DMA` round-trip works.
- [ ] AC-3: `VFIO_CHECK_EXTENSION(VFIO_TYPE1v2_IOMMU)` returns 1.
- [ ] AC-4: Mixed mode test: same qemu uses both legacy `/dev/vfio/vfio` and modern `/dev/iommu` for different devices simultaneously.
- [ ] AC-5: NOIOMMU test: `enable_unsafe_noiommu_mode=1` + `VFIO_NOIOMMU_IOMMU`; DPDK testpmd against vfio-pci-bound device works.
- [ ] AC-6: Performance test: legacy MAP_DMA throughput within 2% of native IOMMU_IOAS_MAP throughput.
- [ ] AC-7: kselftest vfio + iommufd compat tests pass.

## Architecture

VFIO compat state lives in `Ctx`:

```
struct Ctx {
  ...
  vfio_compat: AtomicPtr<VfioCompat>,
}

struct VfioCompat {
  type_: VfioIommuType,                  // TYPE1 / TYPE1v2 / NOIOMMU / UNSET
  ioas: AtomicPtr<Ioas>,                  // current compat IOAS
  group_fd: AtomicPtr<File>,              // legacy /dev/vfio/<N> group fd
}
```

Open path `/dev/vfio/vfio`:
1. If CONFIG_IOMMUFD_VFIO_CONTAINER=y: redirect to iommufd `Ctx::open` (single shared dispatch).
2. ioctl table includes both `IOMMU_*` (modern) AND `VFIO_*` (legacy) numbers.

`iommufd_vfio_ioctl(file, cmd, arg)` dispatch:
```
match cmd {
  VFIO_GET_API_VERSION => 0,
  VFIO_CHECK_EXTENSION => Ctx::vfio_check_extension(ictx, arg as u32),
  VFIO_SET_IOMMU => Ctx::vfio_set_iommu(ictx, arg as i32),
  VFIO_IOMMU_GET_INFO => Ctx::vfio_iommu_get_info(ictx, arg),
  VFIO_IOMMU_MAP_DMA => Ctx::vfio_map_dma(ictx, arg),
  VFIO_IOMMU_UNMAP_DMA => Ctx::vfio_unmap_dma(ictx, arg),
  VFIO_IOMMU_DIRTY_PAGES => Ctx::vfio_dirty_pages(ictx, arg),
  _ => Ctx::ioctl(ictx, cmd, arg),  // fall through to modern dispatch
}
```

`Ctx::vfio_set_iommu(type)`:
1. Validate `type ∈ {TYPE1, TYPE1v2, NOIOMMU}`.
2. Allocate `VfioCompat` if not yet allocated.
3. `vfio_compat.type_ = type`.
4. Compat IOAS allocated lazily on first MAP_DMA (avoids unnecessary allocation if userspace just queries info).

`Ctx::vfio_map_dma(arg)`:
1. Copy `vfio_iommu_type1_dma_map` from userspace.
2. Validate `arg.argsz`, `arg.flags`.
3. `ioas = Ctx::get_compat_ioas(ictx)` (lazy alloc on first call).
4. Translate to internal call: `Ioas::map(ioas, iova=arg.iova, length=arg.size, user_va=arg.vaddr, flags=...)`.
5. Map errors back to legacy errno: `-ENOMEM` for OOM, `-EINVAL` for bad args, `-EBUSY` for overlap.

`Ctx::vfio_unmap_dma(arg)`:
1. Copy struct from userspace.
2. If `arg.flags & VFIO_DMA_UNMAP_FLAG_GET_DIRTY_BITMAP`: capture dirty bitmap via per-HWPT dirty-tracking before unmap.
3. `Ioas::unmap(ioas, iova=arg.iova, length=arg.size)`.
4. Update `arg.size` with actual unmapped length (may be larger than requested if area overlaps).
5. Copy back to userspace.

`Ctx::vfio_iommu_get_info(arg)`:
1. Copy `vfio_iommu_type1_info` from userspace.
2. Populate:
   - `iova_pgsizes`: per-vendor `iommu_ops->pgsize_bitmap` from compat IOAS's HWPT.
   - `cap_offset`: chained extended-info caps.
3. Append per-cap structs:
   - `VFIO_IOMMU_TYPE1_INFO_CAP_IOVA_RANGE`: aperture from IOAS's iopt.
   - `VFIO_IOMMU_TYPE1_INFO_CAP_MIGRATION`: dirty-tracking support.
   - `VFIO_IOMMU_TYPE1_INFO_DMA_AVAIL`: max dma-available accounting.
4. Copy back to userspace.

`Ctx::vfio_dirty_pages(arg)`:
1. Copy `vfio_iommu_type1_dirty_bitmap` from userspace.
2. Per `arg.flags`:
   - `VFIO_IOMMU_DIRTY_PAGES_FLAG_START`: enable dirty tracking on compat IOAS's HWPTs (call `iommufd_hwpt_set_dirty_tracking(true)`).
   - `VFIO_IOMMU_DIRTY_PAGES_FLAG_STOP`: disable.
   - `VFIO_IOMMU_DIRTY_PAGES_FLAG_GET_BITMAP`: copy out dirty bitmap from `iommufd_hwpt_get_dirty_bitmap`.

`IOMMU_VFIO_IOAS` (modern path):
1. `IOMMU_VFIO_IOAS_GET`: return ictx.vfio_compat.ioas's id.
2. `IOMMU_VFIO_IOAS_SET`: look up IOAS by id; replace ictx.vfio_compat.ioas via cmpxchg.
3. `IOMMU_VFIO_IOAS_CLEAR`: clear ictx.vfio_compat.ioas.

NOIOMMU mode: special-case path bypassing IOMMU_HWPT_ALLOC; per-vendor `iommu_ops` returns NULL domain; map/unmap pin pages but don't program any IOMMU hardware. CAP_SYS_RAWIO check per legacy semantics.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `compat_ioas_no_uaf` | UAF | `Arc<Ioas>` ref held by VfioCompat outlives compat-ops; ctx close drops refs in correct order. |
| `arg_size_no_oob` | OOB | every legacy arg's `argsz` validated against expected struct size before copy_from_user. |
| `errno_translation_consistent` | INVARIANT | every internal -E* maps to expected legacy -E* per VFIO API spec. |

### Layer 2: TLA+

(No specific TLA+ — compatibility shim semantics covered by parent's IOAS + HWPT models.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ctx::vfio_map_dma` post: equivalent to `Ioas::map(compat_ioas, ...)`; same final state | `Ctx::vfio_map_dma` |
| `Ctx::vfio_unmap_dma` post: equivalent to `Ioas::unmap(compat_ioas, ...)` + dirty-bitmap-copyout if requested | `Ctx::vfio_unmap_dma` |
| `Ctx::vfio_iommu_get_info` post: returned struct matches per-iommu actual capabilities | `Ctx::vfio_iommu_get_info` |

### Layer 4: Verus/Creusot functional

`VFIO_IOMMU_MAP_DMA(iova, size, vaddr) → device DMA → VFIO_IOMMU_UNMAP_DMA(iova, size)` via legacy interface produces same observable state as equivalent native `IOMMU_IOAS_MAP` + `IOMMU_IOAS_UNMAP` sequence — no behavioral difference visible to qemu / DPDK / SPDK using legacy API.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

vfio-compat specific reinforcement:

- **NOIOMMU mode requires CAP_SYS_RAWIO + module param `enable_unsafe_noiommu_mode=1`** (cross-ref `drivers/vfio/00-overview.md`); defense against unprivileged DMA-capable code path.
- **Per-ioctl `argsz` validated** before copy; oversized argsz capped at sizeof(known-struct); defense against userspace-supplied `argsz` causing kernel OOB read.
- **Compat IOAS allocation per-ctx** — single per-ctx default IOAS for legacy ops; defense against per-call IOAS leak.
- **Errno translation strict per VFIO API spec** — unexpected internal errno mapped to -EINVAL (not silently propagated as random errno); defense against userspace API confusion.
- **Per-fd VFIO ioctl rate-limit** inherited from per-fd modern dispatch rate-limit; defense against legacy-VFIO-API-flood DoS.
- **CONFIG_IOMMUFD_VFIO_CONTAINER opt-in** — distros that don't want compat shim can disable; defense against unintended legacy code paths in security-hardened builds.
- **Dirty-tracking flags enforced same as modern path** — `IOMMU_HWPT_DIRTY_TRACKING_ENABLE` capability validated against per-vendor HW support.

## Open Questions

- **Q1**: Deprecation timeline. Upstream plans to phase out CONFIG_IOMMUFD_VFIO_CONTAINER after qemu transition completes (~2-3 years). Rookery v0 keeps it on; v1 evaluates. **Resolution**: enabled in v0; deprecation gated on userspace migration metrics.

## Out of Scope

- Modern `/dev/iommu` ioctl dispatch (covered in `iommufd-main.md` Tier-3)
- IOAS internals (covered in `iommufd-ioas.md` Tier-3)
- HWPT internals (covered in `iommufd-hwpt.md` Tier-3)
- vfio-pci core (covered in `drivers/vfio/pci-core.md` Tier-3)
- vfio_iommu_type1.c legacy backend (cross-ref `drivers/vfio/00-overview.md`; eventually deprecated)
- 32-bit-only paths
- Implementation code
