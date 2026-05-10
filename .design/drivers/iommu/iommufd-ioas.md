# Tier-3: drivers/iommu/iommufd/{ioas,io_pagetable,pages}.c — IOAS abstraction (per-IOAS pagetable + page-pin lifetime + iova_bitmap)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iommufd/ioas.c
  - drivers/iommu/iommufd/io_pagetable.c
  - drivers/iommu/iommufd/io_pagetable.h
  - drivers/iommu/iommufd/pages.c
  - drivers/iommu/iommufd/iova_bitmap.c
  - drivers/iommu/iommufd/double_span.h
-->

## Summary

The IOAS (IO Address Space) is the modern userspace-visible abstraction for an IOMMU pagetable in iommufd (cross-ref `iommufd-main.md`). Owns: per-IOAS pagetable (a software representation of the iova→backing-page mapping), per-page-pin lifetime tracking via `iopt_pages` reference-counted page lists (allowing IOAS_COPY between IOASes to share underlying pinned pages without re-pinning), per-IOVA-range double-span tracking for partial-mapping discovery, iova-bitmap for dirty-page tracking compatible with VFIO live-migration.

This Tier-3 covers `ioas.c` (~660 lines: per-IOAS lifecycle + map/unmap/copy ops), `io_pagetable.c` (~1550 lines: software pagetable rb-tree), `pages.c` (~2530 lines: per-pages object + page-pin reference tracking + COW + dmabuf integration), `iova_bitmap.c` (per-IOVA bitmap for dirty tracking).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iommufd_ioas` | per-IOAS object | `kernel::iommu::iommufd::Ioas` |
| `struct io_pagetable` | per-IOAS software pagetable | `kernel::iommu::iommufd::IoPagetable` |
| `struct iopt_area` | per-mapped-IOVA-range area (rb-tree node) | `kernel::iommu::iommufd::IoptArea` |
| `struct iopt_pages` | per-source-page-array refcounted backing | `kernel::iommu::iommufd::IoptPages` |
| `iommufd_ioas_alloc(ictx)` | alloc new IOAS object | `Ioas::alloc` |
| `iommufd_ioas_destroy(obj)` | destroy IOAS (auto-unmap + pin-release) | `Ioas::destroy` (Drop) |
| `iommufd_ioas_iova_ranges(ictx, cmd)` | enumerate IOAS aperture(s) | `Ioas::iova_ranges` |
| `iommufd_ioas_allow_iovas(ictx, cmd)` | constrain IOAS aperture | `Ioas::allow_iovas` |
| `iommufd_ioas_map(ictx, cmd)` / `_map_file(...)` | map iova range | `Ioas::map` / `_map_file` |
| `iommufd_ioas_copy(ictx, cmd)` | copy iova range from src IOAS to dst IOAS (sharing pages) | `Ioas::copy_from` |
| `iommufd_ioas_unmap(ictx, cmd)` | unmap iova range | `Ioas::unmap` |
| `iommufd_ioas_change_process(ictx, cmd)` | change owning process for backing pages | `Ioas::change_process` |
| `iommufd_ioas_map_dmabuf(ictx, cmd)` | map dma-buf into IOAS | `Ioas::map_dmabuf` |
| `iopt_init_table(iopt)` | per-IOAS pagetable init | `IoPagetable::init` |
| `iopt_destroy_table(iopt)` | inverse | `IoPagetable::destroy` |
| `iopt_alloc_iova(iopt, &iova, length, flags)` | allocate iova range from aperture | `IoPagetable::alloc_iova` |
| `iopt_map_pages(iopt, pages, length, iova_in, iova_alignment, ...)` | map per-page list at iova | `IoPagetable::map_pages` |
| `iopt_unmap_iova(iopt, iova, length, &unmapped)` | unmap iova range | `IoPagetable::unmap_iova` |
| `iopt_copy_pages_for_area(iopt, area, src_iopt, src_area, src_start_iova, src_iova_alignment)` | per-area page copy | `IoPagetable::copy_pages_for_area` |
| `iopt_set_dirty_tracking(iopt, hwpt, enable)` | enable/disable per-area dirty tracking | `IoPagetable::set_dirty_tracking` |
| `iopt_read_and_clear_dirty_data(iopt, hwpt, &iova_bitmap, range, flags)` | read + clear dirty bitmap (for live migration) | `IoPagetable::read_and_clear_dirty_data` |
| `iopt_pages_init(pages, type, npages)` | per-pages init (USER / FILE / DMABUF type) | `IoptPages::init` |
| `iopt_pages_get(pages)` / `_put(pages)` | refcount get/put | `IoptPages::get` / `_put` |
| `iopt_pages_fill_area(pages, &area, &freelist)` | populate per-area's page-array | `IoptPages::fill_area` |
| `iopt_pages_unmap_area(pages, &area, npages)` | per-area unmap | `IoptPages::unmap_area` |
| `iopt_get_pages(iopt, iova, length, &pages_out, &start_byte_out)` | per-iova lookup → backing pages | `IoPagetable::get_pages` |
| `iova_bitmap_init(bitmap, ...)` | per-bitmap init | `IovaBitmap::init` |
| `iova_bitmap_for_each(bitmap, cb, opaque)` | iterate set bits | `IovaBitmap::for_each` |
| `iova_bitmap_set(bitmap, iova, length)` | mark range dirty | `IovaBitmap::set` |

## Compatibility contract

REQ-1: `IOMMU_IOAS_ALLOC` UAPI byte-identical: returns IOAS object id; per-IOAS empty mapping table.

REQ-2: `IOMMU_IOAS_IOVA_RANGES` returns per-IOAS aperture range list (typically reflects per-attached-HWPT aperture intersection); used by userspace to validate iova choices.

REQ-3: `IOMMU_IOAS_ALLOW_IOVAS` constrains future allocs to a smaller subset of the aperture; useful for partition-IOVA between consumers.

REQ-4: `IOMMU_IOAS_MAP[2]` semantics:
- `iova` (input/output): if user-supplied (FIXED_IOVA flag), use as-is; else iopt allocates from free aperture.
- `length`: bytes to map (must be page-aligned).
- `user_va`: source userspace virtual address.
- `flags`: `READABLE`, `WRITEABLE`, `FIXED_IOVA`, `MAP_FILE`.
- Pin user pages via `pin_user_pages_remote(...)` → per-page refcount + per-pages object.
- For each attached HWPT (RCU-protected list): `iommu_map(domain, iova, paddr, length, prot)`.
- Insert IoptArea into rb-tree.
- Account against current process's RLIMIT_MEMLOCK + per-cgroup `memory.lock_max`.

REQ-5: `IOMMU_IOAS_UNMAP`:
- Look up IoptArea(s) covering [iova, iova+length).
- For each attached HWPT: `iommu_unmap(domain, iova, length)`.
- Walk per-pages refcount; decrement; if refcount==0, `unpin_user_pages(...)`.
- Free IoptArea + remove from rb-tree.

REQ-6: `IOMMU_IOAS_COPY`:
- Locate IoptArea(s) in src IOAS for src_iova range.
- For each src area: get its IoptPages; bump refcount.
- Allocate IoptArea in dst IOAS; install pointer to same IoptPages (sharing).
- For dst's attached HWPTs: `iommu_map(...)`.

REQ-7: `IOMMU_IOAS_MAP_FILE` semantics: same as MAP but page source is an inode (typically tmpfs / hugetlbfs); pin via `pin_user_pages_for_file(...)`; useful for guest-RAM-backed-by-file scenarios.

REQ-8: `IOMMU_IOAS_MAP_DMABUF` semantics: page source is a dma-buf; per-attachment via `dma_buf_attach + dma_buf_map_attachment`; sg_table → per-page list.

REQ-9: `IOMMU_IOAS_CHANGE_PROCESS` re-attributes pinned pages to a new owning process (used for vfio process-fork-then-rebind scenarios).

REQ-10: Dirty-page tracking: per-attached-HWPT `set_dirty_tracking(true)` enables; subsequent `read_and_clear_dirty_data(iova_bitmap, range)` walks dirty bits + clears; used by VFIO live-migration `IOMMU_HWPT_GET_DIRTY_BITMAP`.

REQ-11: Per-IoptPages refcount: incremented on copy + map; decremented on unmap; final put triggers `unpin_user_pages` and frees structure.

REQ-12: Per-process RLIMIT_MEMLOCK accounting: pinned pages count against per-process locked-memory cap; over-cap returns -ENOMEM.

## Acceptance Criteria

- [ ] AC-1: qemu vfio-pci passthrough w/ iommufd-mode: `IOMMU_IOAS_ALLOC` + `IOMMU_IOAS_MAP` for guest RAM + `IOMMU_HWPT_ALLOC` + attach VFIO-device → guest sees device + can DMA.
- [ ] AC-2: IOAS_COPY test: alloc IOAS-A, map 1GB; alloc IOAS-B, copy from A; both reference same backing pages (verified via /proc/<pid>/status MemLocked).
- [ ] AC-3: IOAS_MAP_FILE test: allocate hugetlbfs-backed file; map into IOAS via MAP_FILE; subsequent device DMA works at expected throughput.
- [ ] AC-4: IOAS_MAP_DMABUF test: GPU-allocated dma-buf imported into IOAS; cross-driver DMA path works.
- [ ] AC-5: Dirty-tracking test: IOAS w/ HWPT having dirty-tracking enabled; sustained device DMA writes; `read_and_clear_dirty_data` returns correct dirty bitmap.
- [ ] AC-6: RLIMIT_MEMLOCK test: process w/ ulimit -l 100MB attempts IOAS_MAP 200MB → -ENOMEM (after 100MB pinned).
- [ ] AC-7: kselftest iommufd subset passes (incl. iommufd_dirty_tracking test).

## Architecture

`Ioas` lives in `kernel::iommu::iommufd::Ioas`:

```
struct Ioas {
  obj: Object,                          // base iommufd object
  iopt: IoPagetable,
  hwpt_list: Mutex<Vec<Arc<HwPagetable>>>,  // attached HWPTs
  mutex: Mutex<()>,
}

struct IoPagetable {
  iova_alignment: u32,
  domain_alignment: u32,
  iova_max: u64,
  area_list: Mutex<RBTree<IoptArea>>,    // per-iova-range areas
  reserved_iovas: Mutex<RBTree<IoptReservedIova>>,
  next_iova: AtomicU64,                  // hint for first-fit alloc
}

struct IoptArea {
  iova_start: u64,
  iova_last: u64,
  iommu_prot: IommuProt,                  // READ / WRITE / EXEC / etc.
  num_accesses: AtomicU64,                // ref count for in-use map_attachments
  pages: Arc<IoptPages>,
  page_offset: u64,                        // start within pages
  storage_domain: AtomicPtr<IommuDomain>,
}

struct IoptPages {
  refcount: Refcount,
  domains_itree: Mutex<RBTree<IoptArea>>,  // back-pointer to areas referencing these pages
  npages: u64,
  type_: IoptPagesType,                    // USER / FILE / DMABUF
  uptr: Option<NonNull<u8>>,                // user_va source (USER type)
  npinned: AtomicU64,                       // count of pinned pages
  account_mode: AccountMode,
  pinned_pfns: KBox<XArray<u64>>,           // per-pinned-page pfns
  bitmap: KBox<IovaBitmap>,                 // for dirty tracking
  type_specific: TypeSpecificData,          // per-type backing (file struct, dmabuf attach, etc.)
}
```

Map flow `Ioas::map(cmd)`:
1. Validate cmd: length non-zero, page-aligned, user_va non-zero.
2. Acquire `ioas.mutex`.
3. `IoPagetable::alloc_iova(&ioas.iopt, &iova_out, length, flags)`:
   - If FIXED_IOVA: validate `cmd.iova` is in aperture + non-overlap; reserve.
   - Else: scan rb-tree for free hole; reserve.
4. Allocate or look up IoptPages:
   - For USER type: `IoptPages::init(npages, USER, cmd.user_va, account_mode)`.
   - `pin_user_pages_remote(mm, user_va, npages, flags, &pages_array)` → per-page refcount.
   - Account against current's RLIMIT_MEMLOCK.
5. Allocate IoptArea: `iova_start=iova_out`, `iova_last=iova_out+length-1`, `pages=Arc::clone(&pages)`, `page_offset=0`.
6. Insert into `ioas.iopt.area_list` rb-tree.
7. For each `hwpt` in `ioas.hwpt_list`:
   - `iopt_pages_fill_xarray(...)` retrieves backing pfn array.
   - `iommu_map_pages(hwpt.domain, iova, pfns, length, prot)` (cross-ref `iommu-core.md`).
8. Drop ioas.mutex.

Unmap flow `Ioas::unmap(cmd)`:
1. Acquire ioas.mutex.
2. Walk `ioas.iopt.area_list` for areas overlapping [cmd.iova, cmd.iova+cmd.length).
3. For each area:
   - For each attached HWPT: `iommu_unmap(hwpt.domain, area.iova_start, area.iova_last - area.iova_start + 1)`.
   - `IoptPages::unmap_area(area.pages, &area)` → decrement per-pages refcount; if refcount==0, `unpin_user_pages(...)`.
   - Remove area from rb-tree.
4. Drop ioas.mutex.

Copy flow `Ioas::copy_from(cmd)`:
1. Acquire src + dst ioas.mutex (in canonical order).
2. Walk src.iopt.area_list for areas in src_iova range.
3. For each src_area:
   - `IoPagetable::alloc_iova(&dst.iopt, &dst_iova_out, src_area.length, ...)`.
   - Allocate dst_area: `pages=Arc::clone(&src_area.pages)` (refcount bumped, no re-pin).
   - Insert dst_area.
   - For each dst's attached HWPT: `iommu_map(dst_hwpt.domain, dst_iova, ...)`.
4. Drop mutexes.

Dirty-tracking `IoPagetable::read_and_clear_dirty_data(iopt, hwpt, &bitmap, range)`:
1. For each iopt_area in [range.iova, range.iova+range.length):
   - Per-attached-HWPT: per-vendor `domain_ops->read_and_clear_dirty(domain, area.iova_start, area.length, &bitmap)` (Intel SLADE / AMD HWDirty / software-tracked-via-write-protect-then-fault).
   - Bitmap bits set per-page that was dirty.
2. Return bitmap to userspace via cmd buffer.

Dmabuf path `Ioas::map_dmabuf(cmd)`:
1. `dma_buf = dma_buf_get(cmd.dma_buf_fd)`.
2. `attach = dma_buf_attach(dma_buf, &iommu_dev)`.
3. `sg_table = dma_buf_map_attachment(attach, DMA_BIDIRECTIONAL)`.
4. Allocate IoptPages with type DMABUF; populate pfn-list from sg_table.
5. Continue with normal map flow.

Per-IoptPages free `IoptPages::Drop`:
1. Per-type cleanup:
   - USER: unpin_user_pages.
   - FILE: per-file unpin + fput.
   - DMABUF: dma_buf_unmap_attachment + dma_buf_detach + dma_buf_put.
2. Release pfns xarray.
3. Account refund against RLIMIT_MEMLOCK.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ioas_no_uaf` | UAF | `Ioas` outlives all attached HWPTs + in-flight map operations; iommufd object table refcount enforces. |
| `iopt_pages_no_uaf` | UAF | `Arc<IoptPages>` refcount tracks all referencing IoptAreas; final put triggers unpin. |
| `iova_alloc_no_collision` | UNIQUENESS | per-IOAS iova alloc returns unique non-overlapping range; rb-tree invariants enforce. |
| `pinned_npages_no_overflow` | OVERFLOW | per-IoptPages npinned counter saturating; per-process RLIMIT_MEMLOCK accounting checked. |

### Layer 2: TLA+

(Specific to iommufd-IOAS — declared in parent's `drivers/iommu/iommufd-main.md` section.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ioas::map` post: requested iova range covered by exactly one IoptArea; per-page pfn xarray fully populated; per-attached HWPT has matching iommu_map | `Ioas::map` |
| `Ioas::unmap` post: requested iova range no longer covered by any IoptArea; per-page refcount decremented; freed pfns un-pinned | `Ioas::unmap` |
| `Ioas::copy_from` post: dst IoptArea references same IoptPages as src (refcount bumped); both can DMA to same backing memory | `Ioas::copy_from` |
| Per-IoptArea invariant: `iova_start <= iova_last`; `iova_last - iova_start + 1 == pages.npages * PAGE_SIZE - page_offset` | per-area constructor |

### Layer 4: Verus/Creusot functional

`Ioas::map(iova, len, user_va) → device DMA → Ioas::unmap(iova, len)` round-trip equivalence: post-map, every iova in [iova, iova+len) translates to corresponding user_va offset; post-unmap, no iova in range translates.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iommufd-ioas specific reinforcement:

- **Per-IOAS page-pin RLIMIT_MEMLOCK accounting** enforced; defense against unprivileged process pinning all of host RAM via iommufd.
- **Per-IoptPages refcount + RCU-safe access** during copy/map/unmap; defense against UAF during concurrent ops.
- **IOAS_COPY non-overlapping enforcement** — copy from area A to area B in same IOAS rejected if ranges overlap.
- **Per-IoptArea reserved-region check** — IOAS_MAP rejected for IOVA in MSI-X window or RMRR ranges (cross-ref `iommu-core.md` § Reserved-region MSI-X enforcement).
- **Dmabuf cross-namespace import LSM mediation** — `dma_buf_get(fd)` from a different process namespace mediated.
- **Per-IOAS aperture cap** — IOAS aperture bounded by intersection of attached HWPTs' apertures; defense against IOMMU-feature-mismatch exposing larger range than HW supports.
- **`change_process` per-process credential check** — only the current process can claim ownership; defense against cross-process page-attribution attack.
- **Dirty-tracking per-area cap** — per-IOAS dirty-bit tracking memory bounded by aperture/page-size; defense against unbounded dirty-bitmap growth.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- iommufd_main.c top-level dispatch (covered in `iommufd-main.md` Tier-3)
- HWPT alloc + invalidate (covered in `iommufd-hwpt.md` future Tier-3)
- VIOMMU + VDEVICE (covered in `iommufd-viommu.md` future Tier-3)
- VFIO compat shim (covered in `iommufd-vfio-compat.md` future Tier-3)
- iommu_map/unmap (covered in `iommu-core.md` Tier-3)
- 32-bit-only paths
- Implementation code
