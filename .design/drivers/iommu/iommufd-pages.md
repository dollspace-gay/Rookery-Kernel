# Tier-3: drivers/iommu/iommufd/pages.c — IoptPages (USER/FILE/DMABUF backing + per-pages refcount + COW + page-pin lifetime)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/iommu/00-overview.md
upstream-paths:
  - drivers/iommu/iommufd/pages.c
  - drivers/iommu/iommufd/io_pagetable.h (IoptPages struct)
  - drivers/iommu/iommufd/double_span.h
-->

## Summary

`IoptPages` is the iommufd-side reference-counted backing for a contiguous range of pages — abstracts over USER (anonymous-mm-backed), FILE (inode/file-backed e.g., hugetlbfs/tmpfs), and DMABUF (cross-driver dma-buf) page sources. Critical for `IOMMU_IOAS_COPY` semantics: when one IOAS copies a mapping from another IOAS, the target IoptArea references the same `IoptPages` (refcount-bump only), allowing zero-copy sharing of pinned pages across contexts. Per-pages object holds the pinned-page array (or fault-on-demand hooks for FILE/DMABUF), per-page reference count for COW (if file-backed + writable + COW-required), and the iova_bitmap for dirty-tracking integration.

This Tier-3 covers `drivers/iommu/iommufd/pages.c` (~2500 lines) — the largest single file in `iommufd/`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct iopt_pages` | per-pages reference-counted backing | `kernel::iommu::iommufd::IoptPages` |
| `struct iopt_pages_access` | per-attached-area access record | `kernel::iommu::iommufd::PagesAccess` |
| `iopt_alloc_user_pages(uptr, npages, ...)` | alloc IoptPages from user-VA range | `IoptPages::alloc_user` |
| `iopt_alloc_file_pages(file, start, npages, ...)` | alloc IoptPages from file (tmpfs/hugetlbfs) | `IoptPages::alloc_file` |
| `iopt_alloc_dmabuf_pages(dmabuf, ...)` | alloc IoptPages from dmabuf | `IoptPages::alloc_dmabuf` |
| `iopt_pages_get(pages)` / `_put(pages)` | refcount get/put | `Arc<IoptPages>` get/put |
| `iopt_area_pages_get(area, pages)` / `_put(...)` | per-area access record | `IoptPages::area_get` / `_area_put` |
| `iopt_pages_fill_xarray(pages, start_idx, end_idx, &out_xarr)` | populate pfn xarray for installed range | `IoptPages::fill_xarray` |
| `iopt_pages_unfill_xarray(pages, start_idx, end_idx)` | inverse | `IoptPages::unfill_xarray` |
| `iopt_pages_fill_user_pages(pages, start, end, &out_npages)` | pin user pages on first access | `IoptPages::fill_user_pages` |
| `iopt_pages_unpin_user_pages(pages, start, end)` | unpin range | `IoptPages::unpin_user_pages` |
| `iopt_pages_rw_access(pages, start, length, &buf, write)` | read/write per-page (for IOMMU_IOAS_MAP_DMABUF + dirty-bitmap export) | `IoptPages::rw_access` |
| `iopt_pages_change_process(pages, new_owner)` | re-attribute pinned pages to new process | `IoptPages::change_process` |
| `iopt_pages_dmabuf_attachment(pages, dev)` | per-dev dmabuf attach + map | `IoptPages::dmabuf_attachment` |
| `iopt_pages_dmabuf_detach(pages, attach)` | inverse | `IoptPages::dmabuf_detach` |
| `iopt_pages_iova_bitmap_get(pages)` | per-pages dirty bitmap accessor | `IoptPages::iova_bitmap_get` |
| `iopt_iova_bitmap_set(bitmap, &range, dirty)` | mark range dirty | `IovaBitmap::set` |

## Compatibility contract

REQ-1: Per-pages type byte-identical:
- `IOPT_PAGES_USER`: anonymous user-mm-backed; pin via `pin_user_pages_remote(...)`.
- `IOPT_PAGES_FILE`: file-backed (tmpfs / hugetlbfs / KSM-mergeable file); pin via `pin_user_pages_remote_file(...)` or per-file inode path.
- `IOPT_PAGES_DMABUF`: cross-driver dma-buf source; per-dev `dma_buf_attach + dma_buf_map_attachment` to populate sg_table.

REQ-2: Per-IoptPages refcount: incremented on every IoptArea reference (initial alloc + IOMMU_IOAS_COPY); decremented on IoptArea destroy; final put triggers per-type cleanup.

REQ-3: Lazy per-page pinning: pin happens on `iopt_pages_fill_xarray` (called when first IoptArea installed on actual HWPT/IOMMU); unpin on last unfill.

REQ-4: Per-pages npinned counter: monotonic; bounded by per-process RLIMIT_MEMLOCK + per-cgroup `memory.lock_max`; over-cap returns -ENOMEM.

REQ-5: COW (Copy-On-Write) for file-backed + writable: per-page COW handled via VM-level COW; iopt_pages tracks per-page COW-state.

REQ-6: Per-page access tracking via `iopt_pages_access`: each IoptArea registers an access record; allows incremental fill/unfill on partial-area mapping.

REQ-7: `iopt_pages_change_process(pages, new_owner)`: re-attribute pinned pages to new process for accounting purposes (e.g., vfio process-fork-then-rebind).

REQ-8: dmabuf path: per-dev attach + map_attachment populates per-page sg_table; on iopt_pages_destroy, dma_buf_unmap_attachment + dma_buf_detach + dma_buf_put.

REQ-9: iova_bitmap integration: per-pages bitmap covers entire pages range; per-page bit set when device DMA-write detected (via per-vendor dirty-tracking from intel/amd iommu_ops); read-and-clear semantics for live-migration.

REQ-10: Per-pages double-span representation (`double_span.h`): per-pages array uses interval-tree of double-spans (covering both pin-state + access-state) for O(log N) range queries.

REQ-11: Per-pages destroy flow:
- Walk all IoptArea references; assert refcount==0.
- Per-type cleanup:
  - USER: `unpin_user_pages_dirty_lock(pages, npages, true)` for writable + dirty pages.
  - FILE: per-file `unpin_user_pages` + `fput(file)`.
  - DMABUF: dma_buf_unmap_attachment + dma_buf_detach + dma_buf_put.
- Free per-pages bitmap + double-span tree.
- Account refund against RLIMIT_MEMLOCK.

REQ-12: Concurrent map+unmap on overlapping pages serialized via per-pages mutex; defense against torn pin/unpin.

## Acceptance Criteria

- [ ] AC-1: USER-type test: `IOMMU_IOAS_MAP(uptr=anon-mmap, length=1G)` pins 256K pages; per-process RLIMIT_MEMLOCK accounting reflects 1GB.
- [ ] AC-2: FILE-type test: hugetlbfs-backed file mmap + `IOMMU_IOAS_MAP_FILE`; per-file pin count tracks correctly.
- [ ] AC-3: DMABUF-type test: GPU-allocated dma-buf imported via `IOMMU_IOAS_MAP_DMABUF`; per-dev attach + map_attachment populates pfn-list.
- [ ] AC-4: IOMMU_IOAS_COPY zero-pin test: copy 1G mapping from IOAS-A to IOAS-B; per-process RLIMIT_MEMLOCK shows 1GB total (not 2GB) — proves shared backing.
- [ ] AC-5: COW test: file-backed shared mapping + writable; first write triggers COW; per-pages tracking reflects COW pages.
- [ ] AC-6: iova_bitmap test: enable dirty-tracking on HWPT; sustained device DMA-write; read bitmap shows correct dirty bits per-page.
- [ ] AC-7: change_process test: bind device in process A; fork process B; change_process to B; cleanup on A-exit doesn't unpin.
- [ ] AC-8: kselftest iommufd pages-related subset passes.

## Architecture

`IoptPages` lives in `kernel::iommu::iommufd::IoptPages`:

```
struct IoptPages {
  refcount: Refcount,
  domains_itree: Mutex<RBTree<IoptArea>>,    // per-area back-pointer for invalidation propagation
  npages: u64,
  type_: IoptPagesType,                       // USER / FILE / DMABUF
  uptr: Option<NonNull<u8>>,                   // user_va source (USER only)
  source_user: Option<Arc<MmStruct>>,          // owning mm (USER + change_process)
  source_file: Option<Arc<File>>,              // backing file (FILE only)
  source_dmabuf: Option<Arc<DmaBuf>>,          // dmabuf (DMABUF only)
  npinned: AtomicU64,                          // count of currently-pinned pages
  account_mode: AccountMode,                   // MM_LOCK / KERNEL_LIMIT
  pinned_pfns: KBox<XArray<u64>>,              // per-pinned-page pfns
  bitmap: KBox<IovaBitmap>,                    // for dirty tracking
  access_lock: Mutex<()>,
  access_list: Mutex<Vec<KBox<PagesAccess>>>,  // per-area access records
  span_tree: Mutex<RBTree<DoubleSpan>>,         // pin-state + access-state interval tree
  type_specific: TypeSpecificData,             // per-type backing union
}

struct PagesAccess {
  iopt_area: Arc<IoptArea>,
  start_index: u64,
  end_index: u64,
  filled: bool,
}

enum TypeSpecificData {
  User { mm: Arc<MmStruct>, anon_or_file_pages: ... },
  File { file: Arc<File>, file_offset: u64, ... },
  Dmabuf { attach: Arc<DmaBufAttachment>, sg_table: NonNull<SgTable>, ... },
}
```

USER-type alloc `IoptPages::alloc_user(uptr, npages, account_mode)`:
1. Allocate IoptPages struct.
2. `pages.uptr = Some(uptr)`, `pages.npages = npages`, `pages.type_ = USER`.
3. `pages.source_user = Some(Arc::clone(&current.mm))` — hold mm refcount.
4. `pages.account_mode = account_mode`.
5. Initialize `pinned_pfns` xarray + `bitmap` + `access_list`.
6. Return Arc<IoptPages>.

FILE-type alloc `IoptPages::alloc_file(file, start, npages, ...)`:
1. Allocate IoptPages struct.
2. `pages.source_file = Some(Arc::clone(&file))`.
3. `pages.type_ = FILE`.
4. Per-file backing-specific setup (tmpfs vs hugetlbfs).

DMABUF-type alloc `IoptPages::alloc_dmabuf(dmabuf, ...)`:
1. Allocate IoptPages struct.
2. `pages.source_dmabuf = Some(Arc::clone(&dmabuf))`.
3. `pages.type_ = DMABUF`.
4. Per-importing-dev attach + map_attachment via `IoptPages::dmabuf_attachment(pages, dev)`.

Lazy fill `IoptPages::fill_xarray(pages, start_idx, end_idx, &out_xarr)`:
1. Take `pages.access_lock`.
2. Walk access_list for overlapping records; if present, return cached xarray entries.
3. Else (first install on this range):
   - Per-type pin path:
     - USER: `pin_user_pages_remote(pages.source_user.mm, pages.uptr + start_idx*PAGE_SIZE, end_idx - start_idx, FOLL_PIN | FOLL_LONGTERM, &page_array)`.
     - FILE: per-file `pin_user_pages_remote_file(...)` + `iopt_pages_fill_user_pages` for tmpfs path.
     - DMABUF: walk sg_table starting at offset; populate pfn array.
   - Update `pages.pinned_pfns` xarray with returned pfns.
   - `pages.npinned += end_idx - start_idx`.
   - Account against RLIMIT_MEMLOCK.
4. Insert PagesAccess record for this range.
5. Return out_xarr (pfn array for caller to use in iommu_map).

Per-area unfill `IoptPages::unfill_xarray(pages, start_idx, end_idx)`:
1. Take access_lock.
2. Walk access_list for matching record; remove.
3. If no other access record covers any page in [start_idx, end_idx):
   - Per-type unpin:
     - USER: `unpin_user_pages_dirty_lock(page_array, n, write)` for writable + dirty marker.
     - FILE: similar.
     - DMABUF: no per-page action (sg_table pinned for dmabuf lifetime).
   - Clear from `pages.pinned_pfns` xarray.
   - `pages.npinned -= n`.
   - Refund RLIMIT_MEMLOCK.
4. Drop access_lock.

`IoptPages::change_process(pages, new_owner)`:
1. Take access_lock.
2. Walk pinned_pfns; for each: `change_pinned_owner(pfn, old_owner, new_owner)` (per-page mm transfer).
3. Update `pages.source_user.mm = new_owner`.
4. Refund old_owner's RLIMIT_MEMLOCK + charge new_owner.

Per-pages destroy `IoptPages::Drop`:
1. Assert `domains_itree.is_empty()`.
2. Per-type cleanup (per REQ-11).
3. Free bitmap + span_tree.

iova_bitmap integration: per-pages bitmap one-bit-per-page-in-pages-range; per-vendor IOMMU dirty-tracking sets bits when device DMA-writes; consumer (live migration) reads + clears bits.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pages_no_uaf` | UAF | `Arc<IoptPages>` outlives all referencing IoptAreas; final put triggers per-type cleanup before free. |
| `pin_unpin_balanced` | INVARIANT | `pages.npinned` reflects total pinned pages; never goes negative. |
| `account_balanced` | INVARIANT | RLIMIT_MEMLOCK accounting matched per-pin/unpin; refund on destroy. |
| `cow_state_consistent` | INVARIANT | per-page COW state reflects actual VM-side COW status. |

### Layer 2: TLA+

(No specific TLA+ — pages-level semantics covered by parent's IOAS + iopt_pages_lifecycle reasoning in `iommufd-ioas.md`.)

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IoptPages::fill_xarray` post: returned xarr has pfns for [start_idx, end_idx); all pages pinned via per-type method | `IoptPages::fill_xarray` |
| `IoptPages::unfill_xarray` post: per-page refcount decremented; pages unpinned if last reference | `IoptPages::unfill_xarray` |
| `IoptPages::change_process` post: pinned pages re-attributed to new_owner; old_owner accounting refunded | `IoptPages::change_process` |
| Per-pages refcount-on-COPY invariant: IOMMU_IOAS_COPY bumps refcount but does not re-pin; both source + dest IOAS share same backing | `IoptPages::get` |

### Layer 4: Verus/Creusot functional

`IoptPages::alloc → IoptPages::fill_xarray → device DMA → IoptPages::unfill_xarray → IoptPages::Drop` round-trip equivalence: every pinned page eventually unpinned; per-process RLIMIT_MEMLOCK accounting balanced; no leaked pin.

## Hardening

(Inherits row-1 features from `drivers/iommu/00-overview.md` § Hardening.)

iommufd-pages specific reinforcement:

- **Per-process RLIMIT_MEMLOCK accounting strict** — per-pin charged + per-unpin refunded; over-cap returns -ENOMEM; defense against unprivileged process pinning all of host RAM.
- **Per-pages refcount + per-area access records** — reference shared backing across IOMMU_IOAS_COPY without re-pin; defense against double-pin-then-leak attack.
- **DMABUF cross-namespace import LSM-mediated** — `dma_buf_get(fd)` from different namespace mediated by file-LSM hook on source-fd.
- **DMABUF lifetime tied to dma_buf refcount** — per-attachment held while iopt_pages owns it; defense against UAF on dmabuf-source-driver-unload.
- **FILE-backed pin uses inode-aware pin** — defense against file truncate while pinned (proper EFAULT delivery to device).
- **change_process per-process credential check** — only the current process can claim ownership; defense against cross-process page-attribution attack.
- **Per-pages bitmap size cap** — bounded by pages.npages; defense against bitmap-OOM via huge mappings.
- **COW state tracking accurate** — per-page COW transitions observed at VM layer + reflected in iopt_pages tracking.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- IOAS internals (covered in `iommufd-ioas.md` Tier-3)
- iommufd_main top-level dispatch (covered in `iommufd-main.md` Tier-3)
- HWPT details (covered in `iommufd-hwpt.md` Tier-3)
- VM-level pin_user_pages internals (covered in `mm/00-overview.md` future Tier-3 chain)
- dma-buf reservation framework (covered in `drm-gem-core.md` Tier-3 + `kernel/dma/dmabuf-core.md` future Tier-3)
- 32-bit-only paths
- Implementation code
