---
title: "Tier-3: mm/cma.c — Contiguous Memory Allocator"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

CMA (Contiguous Memory Allocator) reserves physically contiguous page ranges at boot, exposes them to the page allocator under `MIGRATE_CMA` for movable-only fallback, and allocates them on demand to consumers (DMA-coherent, hugetlb-gigantic, GPU/camera/codec buffers) on systems without IOMMU. Per-area state lives in `struct cma`; the global pool is `struct cma cma_areas[MAX_CMA_AREAS]` with `unsigned int cma_area_count`. Each area splits into 1..`CMA_MAX_RANGES` `struct cma_memrange` segments (base_pfn, count, bitmap). Per-area allocation: scan bitmap for `bitmap_count` zero bits aligned to `cma_bitmap_aligned_mask`, mark, then `alloc_contig_frozen_range(pfn, pfn+count, ACR_FLAGS_CMA, gfp)` which migrates non-movable pages out via `migrate_pages()`. Per-release: `put_page_testzero` each page, `free_contig_frozen_range`, clear bitmap. Per-zone-discipline: `cma_validate_zones` enforces every range be entirely within a single zone (`pfn_range_intersects_zones`); cross-zone ranges are rejected to keep `alloc_contig_range` correct. Per-bitmap granularity: `order_per_bit` bit ⇔ `1 << order_per_bit` pages, set via `cma_init_reserved_mem` / `cma_declare_contiguous_nid`. Critical for: DMA-coherent huge-contig on no-IOMMU platforms (ARM camera/GPU), `alloc_contig_range`-backed gigantic-hugepages, persistent reservation across boot.

This Tier-3 covers `mm/cma.c` (~1146 lines).

### Acceptance Criteria

- [ ] AC-1: cma_init_reserved_mem rejects unaligned base|size and unreserved memblock regions.
- [ ] AC-2: cma_declare_contiguous_nid with fixed=true reserves exactly [base, base+size); fixed=false honours limit and prefers HIGHMEM / above-4 GiB.
- [ ] AC-3: cma_declare_contiguous_multi falls back to ≤CMA_MAX_RANGES ranges above 4 GiB when single-range alloc fails.
- [ ] AC-4: cma_validate_zones returns false (and sets CMA_ZONES_INVALID) for any range spanning two zones; memoised on subsequent calls.
- [ ] AC-5: cma_activate_area allocates per-range bitmaps, initialises MIGRATE_CMA pageblocks, sets CMA_ACTIVATED.
- [ ] AC-6: cma_alloc returns a page* with refcount==1 on success and NULL on failure; cma_alloc_frozen returns refcount-0 pages.
- [ ] AC-7: cma_range_alloc retries (-EBUSY) on migrate failure with cleared bitmap and exits non-busy errors immediately.
- [ ] AC-8: cma_alloc honours align (in PAGE_SIZE order) via cma_bitmap_aligned_mask + cma_bitmap_aligned_offset.
- [ ] AC-9: cma_release returns false when pages do not belong to any range; true otherwise; WARNs if any page still has refs after put_page_testzero.
- [ ] AC-10: cma.available_count is decremented under cma.lock at alloc and re-incremented under cma.lock at release.
- [ ] AC-11: cma_reserve_early before activation advances cmr.early_pfn and is reflected in the bitmap at activation (pre-reserved bits marked).
- [ ] AC-12: totalcma_pages == sum of cma->count across active areas (after init).
- [ ] AC-13: ENOSPC returned when cma_area_count == MAX_CMA_AREAS.
- [ ] AC-14: trace_cma_alloc_start/finish/busy_retry/release fire as in upstream.

### Architecture

```
struct Cma {
    name: [u8; CMA_MAX_NAME],
    ranges: [CmaMemrange; CMA_MAX_RANGES],
    nranges: u32,
    count: u64,                         // total pages
    available_count: u64,               // free pages
    order_per_bit: u32,                 // bit ⇔ 1<<order_per_bit pages
    flags: AtomicU32,                   // CMA_ACTIVATED | CMA_ZONES_VALID | ...
    nid: i32,
    lock: SpinLockIrq,                  // protects bitmap + available_count
    alloc_mutex: Mutex,                 // serialises alloc_contig_frozen_range
    #[cfg(CONFIG_CMA_DEBUGFS)]
    mem_head: HListHead,
    #[cfg(CONFIG_CMA_DEBUGFS)]
    mem_head_lock: SpinLock,
}

struct CmaMemrange {
    base_pfn: u64,
    early_pfn: u64,                     // bump cursor for cma_reserve_early
    count: u64,
    bitmap: *mut usize,                 // bitmap_alloc'd at activation
}

static mut CMA_AREAS: [Cma; MAX_CMA_AREAS] = ...;
static mut CMA_AREA_COUNT: u32 = 0;
```

`Cma::init_reserved_mem(base, size, order_per_bit, name, res_cma) -> i32`:
1. if !size ∨ !memblock_is_region_reserved(base, size): return -EINVAL.
2. if !pageblock_order: return -EINVAL.
3. if !IS_ALIGNED(base | size, CMA_MIN_ALIGNMENT_BYTES): return -EINVAL.
4. cma_new_area(name, size, order_per_bit, &cma)?.
5. cma.ranges[0] = { base_pfn=PFN_DOWN(base), early_pfn=PFN_DOWN(base), count=cma.count, bitmap=NULL }.
6. cma.nranges = 1; cma.nid = NUMA_NO_NODE.
7. *res_cma = cma; return 0.

`Cma::declare_contiguous_nid(base, size, limit, alignment, order_per_bit, fixed, name, &res_cma, nid) -> i32`:
1. Calls __declare_contiguous_nid(&base, ...) (see REQ-5).
2. Logs success / failure with size and base.

`Cma::activate_area(cma)`:
1. For each range r in 0..nranges:
   - early_pfn_local[r] = cmr.early_pfn.
   - cmr.bitmap = bitmap_zalloc(cma_bitmap_maxno(cma, cmr), GFP_KERNEL). On NULL: goto cleanup.
2. if !cma_validate_zones(cma): goto cleanup.
3. For each range:
   - if early_pfn_local[r] != cmr.base_pfn: bitmap_set(cmr.bitmap, 0, cma_bitmap_pages_to_bits(cma, early_pfn_local[r] - cmr.base_pfn)).
   - for pfn = early_pfn_local[r]; pfn < cmr.base_pfn+cmr.count; pfn += pageblock_nr_pages: init_cma_reserved_pageblock(pfn_to_page(pfn)).
4. spin_lock_init(&cma.lock); mutex_init(&cma.alloc_mutex).
5. #ifdef CONFIG_CMA_DEBUGFS: INIT_HLIST_HEAD + spin_lock_init.
6. set_bit(CMA_ACTIVATED, &cma.flags).
7. cleanup: bitmap_free for r in 0..allocrange; if !CMA_RESERVE_PAGES_ON_ERROR: free_reserved_page for every pfn; totalcma_pages -= cma.count; available_count=count=0; pr_err.

`Cma::validate_zones(cma) -> bool`:
1. if CMA_ZONES_VALID: return true. if CMA_ZONES_INVALID: return false.
2. For r: WARN_ON_ONCE(!pfn_valid(base_pfn)); if pfn_range_intersects_zones(cma.nid, base_pfn, count): set CMA_ZONES_INVALID; return false.
3. set CMA_ZONES_VALID; return true.

`Cma::range_alloc(cma, cmr, count, align, &page, gfp) -> i32`:
1. mask = bitmap_aligned_mask; offset = bitmap_aligned_offset; bitmap_maxno; bitmap_count.
2. if bitmap_count > bitmap_maxno: return -EBUSY.
3. start = 0.
4. loop:
   a. spin_lock_irq(&cma.lock).
   b. if count > available_count: unlock; break.
   c. bitmap_no = bitmap_find_next_zero_area_off(bitmap, bitmap_maxno, start, bitmap_count, mask, offset).
   d. if bitmap_no >= bitmap_maxno: unlock; break.
   e. pfn = cmr.base_pfn + (bitmap_no << order_per_bit); page = pfn_to_page(pfn).
   f. if !page_range_contiguous(page, count): unlock; pr_warn_ratelimited; start = bitmap_no + mask + 1; continue.
   g. bitmap_set; available_count -= count.
   h. spin_unlock_irq.
   i. mutex_lock(&alloc_mutex); ret = alloc_contig_frozen_range(pfn, pfn+count, ACR_FLAGS_CMA, gfp); mutex_unlock.
   j. if !ret: break (success).
   k. cma_clear_bitmap(cma, cmr, pfn, count).
   l. if ret != -EBUSY: break.
   m. trace_cma_alloc_busy_retry; start = bitmap_no + mask + 1; loop.
5. if !ret: *page = page_local; return 0. Else return ret.

`Cma::alloc(cma, count, align, no_warn) -> *mut Page`:
1. page = __alloc_frozen(cma, count, align, GFP_KERNEL | (no_warn?__GFP_NOWARN:0)).
2. if page: set_pages_refcounted(page, count).
3. return page.

`Cma::release(cma, pages, count) -> bool`:
1. cmr = find_memrange(cma, pages, count).
2. if !cmr: return false.
3. pfn = page_to_pfn(pages).
4. ret = 0; for i in 0..count: ret += !put_page_testzero(pfn_to_page(pfn+i)).
5. WARN(ret, "%lu pages still in use").
6. free_contig_frozen_range(pfn, count); cma_clear_bitmap; cma_sysfs_account_release_pages; trace_cma_release.
7. return true.

`Cma::reserve_early(cma, size) -> *mut c_void`:
1. if !cma ∨ !cma.count ∨ test_bit(CMA_ACTIVATED): return NULL.
2. if !IS_ALIGNED(size, CMA_MIN_ALIGNMENT_BYTES) ∨ !IS_ALIGNED(size, PAGE_SIZE<<order_per_bit): return NULL.
3. size >>= PAGE_SHIFT.
4. if size > available_count: return NULL.
5. For each range: available = cmr.count - (cmr.early_pfn - cmr.base_pfn). If size ≤ available: ret = phys_to_virt(PFN_PHYS(cmr.early_pfn)); cmr.early_pfn += size; available_count -= size; return ret.
6. return NULL.

### Out of Scope

- mm/cma_debug.c CONFIG_CMA_DEBUGFS dentries (covered separately if expanded)
- mm/cma_sysfs.c per-area sysfs counters (covered separately if expanded)
- mm/page_isolation.c alloc_contig_range / alloc_contig_frozen_range (covered in `page-isolation.md` Tier-3)
- mm/migrate.c migrate_pages (covered in `migration-compaction.md` Tier-3)
- mm/page_alloc.c MIGRATE_CMA fallback semantics (covered in `page-allocator.md` Tier-3)
- mm/hugetlb.c gigantic-page allocation via cma (covered in `hugetlb.md` Tier-3)
- dma-contiguous.c DMA-CMA glue (covered separately if expanded)
- arch/*/mm cma= command-line parsing (covered separately if expanded)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cma` | per-area descriptor | `Cma` |
| `struct cma_memrange` | per-range (base_pfn, count, bitmap) | `CmaMemrange` |
| `cma_areas[MAX_CMA_AREAS]` | global pool | `cma::areas` static array |
| `cma_area_count` | populated slot count | `cma::area_count` |
| `cma_init_reserved_mem()` | per-fixed-range create | `Cma::init_reserved_mem` |
| `cma_declare_contiguous_nid()` | per-memblock alloc + create | `Cma::declare_contiguous_nid` |
| `cma_declare_contiguous_multi()` | per-multi-range create | `Cma::declare_contiguous_multi` |
| `cma_activate_area()` | per-bitmap alloc + pageblock init | `Cma::activate_area` |
| `cma_init_reserved_areas()` | core_initcall driver | `Cma::init_reserved_areas` |
| `cma_validate_zones()` | per-area single-zone check | `Cma::validate_zones` |
| `cma_alloc()` / `cma_alloc_frozen()` | per-request alloc | `Cma::alloc` / `alloc_frozen` |
| `cma_alloc_frozen_compound()` | per-order compound alloc | `Cma::alloc_frozen_compound` |
| `cma_release()` / `cma_release_frozen()` | per-free | `Cma::release` / `release_frozen` |
| `cma_range_alloc()` | per-range bitmap scan + migrate | `Cma::range_alloc` |
| `__cma_release_frozen()` | per-range free path | `Cma::release_inner` |
| `find_cma_memrange()` | per-pfn ⇒ range lookup | `Cma::find_memrange` |
| `cma_for_each_area()` | per-iter | `Cma::for_each_area` |
| `cma_intersects()` | per-pfn-range overlap | `Cma::intersects` |
| `cma_reserve_early()` | per-early-bootstrap reserve | `Cma::reserve_early` |
| `cma_reserve_pages_on_error()` | per-keep-on-fail flag | `Cma::reserve_pages_on_error` |
| `cma_get_base()` / `_size()` / `_name()` | per-accessor | `Cma::base` / `size` / `name` |
| `cma_bitmap_aligned_mask()` | per-align mask | `Cma::bitmap_aligned_mask` |
| `cma_bitmap_aligned_offset()` | per-range bitmap offset | `Cma::bitmap_aligned_offset` |
| `cma_bitmap_pages_to_bits()` | pages → bits | `Cma::bitmap_pages_to_bits` |
| `cma_clear_bitmap()` | per-release bitmap clear | `Cma::clear_bitmap` |
| `cma_debug_show_areas()` | per-OOM-CMA diag | `Cma::debug_show_areas` |
| `cma_drop_area()` | per-init-fail rollback | `Cma::drop_area` |
| `cma_new_area()` | per-slot alloc | `Cma::new_area` |
| `cma_fixed_reserve()` | per-fixed memblock_reserve | `Cma::fixed_reserve` |
| `cma_alloc_mem()` | per-memblock_alloc_range_nid | `Cma::alloc_mem` |
| `init_cma_reserved_pageblock()` | per-pageblock to MIGRATE_CMA | `mm::init_cma_reserved_pageblock` (page_alloc) |
| `alloc_contig_frozen_range()` | per-migrate non-movable out | `mm::alloc_contig_frozen_range` (page_isolation) |
| `free_contig_frozen_range()` | per-release back to ranges | `mm::free_contig_frozen_range` |
| `MIGRATE_CMA` | migratetype | shared with page-allocator |
| `CMA_MAX_AREAS` / `CMA_MAX_RANGES` / `CMA_MAX_NAME` | per-tunables | `cma::limits` |
| `CMA_MIN_ALIGNMENT_BYTES` (pageblock_nr_pages << PAGE_SHIFT) | per-min align | shared |
| `CMA_ACTIVATED` / `CMA_ZONES_VALID` / `CMA_ZONES_INVALID` / `CMA_RESERVE_PAGES_ON_ERROR` | per-flag bits | `CmaFlags` |
| `totalcma_pages` | global accounting | shared with vmstat |
| `trace_cma_alloc_start` / `_finish` / `_busy_retry` / `_release` | tracepoints | `trace::cma::*` |
| `count_vm_event(CMA_ALLOC_SUCCESS / _FAIL)` | per-stat | shared vmstat |
| `cma_sysfs_account_*` | per-sysfs stat | sysfs glue |

### compatibility contract

REQ-1: struct cma:
- name: char[CMA_MAX_NAME] — area label ("cmaN" default).
- ranges: cma_memrange[CMA_MAX_RANGES] — physical ranges.
- nranges: number of populated ranges (1 for single-range, ≤CMA_MAX_RANGES for multi).
- count: total pages in area (sum of range counts).
- available_count: free pages (decremented on alloc, incremented on release).
- order_per_bit: bit-to-page exponent (one bit ⇔ 1<<order_per_bit pages).
- flags: bitfield (CMA_ACTIVATED, CMA_ZONES_VALID, CMA_ZONES_INVALID, CMA_RESERVE_PAGES_ON_ERROR).
- nid: NUMA node id (NUMA_NO_NODE if any).
- lock: spinlock_t — protects bitmap + available_count.
- alloc_mutex: mutex — serialises alloc_contig_frozen_range invocations.
- mem_head / mem_head_lock: per-CONFIG_CMA_DEBUGFS allocation tracking list.

REQ-2: struct cma_memrange:
- base_pfn: starting PFN.
- early_pfn: next unreserved PFN for cma_reserve_early.
- count: pages in range.
- bitmap: dynamically-allocated unsigned-long array of cma_bitmap_maxno bits.

REQ-3: cma_areas / cma_area_count:
- Static array sized MAX_CMA_AREAS.
- Indexed 0..cma_area_count-1.
- Slot returned by cma_new_area; reverted by cma_drop_area on init failure.
- ENOSPC if cma_area_count == MAX_CMA_AREAS.

REQ-4: cma_init_reserved_mem(base, size, order_per_bit, name, &res_cma):
- /* Sanity */
- if !size ∨ !memblock_is_region_reserved(base, size): return -EINVAL.
- /* Pageblock order must be set */
- if !pageblock_order: return -EINVAL.
- /* Alignment */
- if !IS_ALIGNED(base | size, CMA_MIN_ALIGNMENT_BYTES): return -EINVAL.
- ret = cma_new_area(name, size, order_per_bit, &cma).
- cma.ranges[0] = { base_pfn = PFN_DOWN(base), early_pfn = PFN_DOWN(base), count = cma.count }.
- cma.nranges = 1.
- cma.nid = NUMA_NO_NODE.
- *res_cma = cma.
- return 0.

REQ-5: cma_declare_contiguous_nid(base, size, limit, alignment, order_per_bit, fixed, name, &res_cma, nid):
- Wraps __cma_declare_contiguous_nid(&base, ...).
- /* Validate */
- ENOSPC if cma_area_count == MAX_CMA_AREAS.
- EINVAL if !size; EINVAL if alignment ∧ !is_power_of_2(alignment).
- /* nid = NUMA_NO_NODE if !CONFIG_NUMA */
- /* alignment = max(alignment, CMA_MIN_ALIGNMENT_BYTES) */
- /* If fixed: base must be aligned, else error */
- /* limit defaults to memblock_end_of_DRAM() */
- if base+size > limit: EINVAL.
- if fixed: cma_fixed_reserve(base, size) — checks HIGHMEM boundary, calls memblock_reserve.
- else: base = cma_alloc_mem(base, size, alignment, limit, nid):
  - Prefer bottom-up above 4 GiB if CONFIG_PHYS_ADDR_T_64BIT.
  - Prefer HIGHMEM if CONFIG_HIGHMEM.
  - Fallback memblock_alloc_range_nid in [base, limit].
- kmemleak_ignore_phys(base) for non-fixed.
- ret = cma_init_reserved_mem(base, size, order_per_bit, name, res_cma).
- on fail: memblock_phys_free(base, size).
- (*res_cma)->nid = nid.

REQ-6: cma_declare_contiguous_multi(total_size, align, order_per_bit, name, &res_cma, nid):
- /* Try single-range first */
- ret = __cma_declare_contiguous_nid(start=0, total_size, ..., fixed=false).
- if ret != -ENOMEM: return ret.
- /* Multi-range fallback: scan free memblocks above 4 GiB */
- cma_new_area(name, total_size, order_per_bit, &cma).
- align = max(align, CMA_MIN_ALIGNMENT_BYTES).
- for_each_free_mem_range(i, nid, MEMBLOCK_NONE, &start, &end, NULL):
  - skip if upper_32_bits(start) == 0.
  - align start up, end down, size down to (PAGE_SIZE << order_per_bit).
  - keep at most CMA_MAX_RANGES largest; sorted by size descending.
- if sizesum < total_size: cma_drop_area(cma); return -ENOMEM.
- Re-sort selected ranges by base ascending; for each, memblock_reserve and append cma->ranges[].
- cma.nranges = nr; cma.nid = nid.
- on memblock_reserve failure: free preceding reservations, cma_drop_area, return -ENOMEM.

REQ-7: cma_new_area(name, size, order_per_bit, &res_cma):
- if cma_area_count == ARRAY_SIZE(cma_areas): return -ENOSPC.
- cma = &cma_areas[cma_area_count++].
- name: strscpy if provided, else snprintf("cma%d\n", cma_area_count).
- cma.available_count = cma.count = size >> PAGE_SHIFT.
- cma.order_per_bit = order_per_bit.
- totalcma_pages += cma.count.
- return 0.

REQ-8: cma_drop_area(cma):
- totalcma_pages -= cma.count.
- cma_area_count--.
- (Caller responsible for clearing slot fields and any reservations.)

REQ-9: cma_validate_zones(cma):
- Memoised via flags:
  - CMA_ZONES_VALID set ⟹ return true.
  - CMA_ZONES_INVALID set ⟹ return false.
- For r in 0..cma.nranges:
  - WARN_ON_ONCE(!pfn_valid(base_pfn)).
  - if pfn_range_intersects_zones(cma.nid, base_pfn, cmr.count): set CMA_ZONES_INVALID; return false.
- set CMA_ZONES_VALID; return true.

REQ-10: cma_activate_area(cma):
- /* core_initcall via cma_init_reserved_areas */
- For each range: cmr.bitmap = bitmap_zalloc(cma_bitmap_maxno(cma, cmr), GFP_KERNEL). On fail: goto cleanup.
- if !cma_validate_zones(cma): goto cleanup.
- For each range:
  - If early_pfn != base_pfn (pages consumed by cma_reserve_early): bitmap_set first count bits.
  - for pfn = early_pfn; pfn < base_pfn+count; pfn += pageblock_nr_pages: init_cma_reserved_pageblock(pfn_to_page(pfn)) (marks MIGRATE_CMA and frees to buddy).
- spin_lock_init(&cma.lock).
- mutex_init(&cma.alloc_mutex).
- INIT_HLIST_HEAD(&cma.mem_head) ∧ spin_lock_init(&cma.mem_head_lock) under CONFIG_CMA_DEBUGFS.
- set_bit(CMA_ACTIVATED, &cma.flags).
- cleanup: bitmap_free per range; unless CMA_RESERVE_PAGES_ON_ERROR: free_reserved_page each page back to buddy; totalcma_pages -= cma.count; cma.available_count = cma.count = 0; pr_err.

REQ-11: cma_init_reserved_areas():
- core_initcall.
- for i in 0..cma_area_count: cma_activate_area(&cma_areas[i]).

REQ-12: cma_reserve_pages_on_error(cma):
- set_bit(CMA_RESERVE_PAGES_ON_ERROR, &cma.flags).
- Activation failure must keep pages out of buddy (caller retains responsibility).

REQ-13: cma_alloc(cma, count, align, no_warn) → struct page *:
- gfp = GFP_KERNEL | (no_warn ? __GFP_NOWARN : 0).
- page = __cma_alloc_frozen(cma, count, align, gfp).
- if page: set_pages_refcounted(page, count).
- return page.

REQ-14: cma_alloc_frozen(cma, count, align, no_warn) → frozen pages (refcount-0):
- Same alloc path as cma_alloc but does not call set_pages_refcounted.

REQ-15: cma_alloc_frozen_compound(cma, order) → struct page *:
- gfp = GFP_KERNEL | __GFP_COMP | __GFP_NOWARN.
- __cma_alloc_frozen(cma, 1<<order, order, gfp).

REQ-16: __cma_alloc_frozen(cma, count, align, gfp):
- if !cma ∨ !cma.count ∨ !count: return NULL.
- trace_cma_alloc_start(name, count, available_count, total, align).
- For r in 0..cma.nranges:
  - ret = cma_range_alloc(cma, &cma.ranges[r], count, align, &page, gfp).
  - if ret != -EBUSY ∨ page: break.
- If page allocated: for each of count pages, page_kasan_tag_reset.
- On failure ∧ !__GFP_NOWARN: pr_err_ratelimited + cma_debug_show_areas.
- count_vm_event(CMA_ALLOC_SUCCESS|CMA_ALLOC_FAIL).
- cma_sysfs_account_success_pages / _fail_pages.
- trace_cma_alloc_finish.

REQ-17: cma_range_alloc(cma, cmr, count, align, &pagep, gfp):
- mask = cma_bitmap_aligned_mask(cma, align).
- offset = cma_bitmap_aligned_offset(cma, cmr, align).
- bitmap_maxno = cma_bitmap_maxno(cma, cmr); bitmap_count = cma_bitmap_pages_to_bits(cma, count).
- if bitmap_count > bitmap_maxno: return -EBUSY.
- Loop start=0; on retry start = bitmap_no + mask + 1:
  - spin_lock_irq(&cma.lock).
  - if count > cma.available_count: unlock; break.
  - bitmap_no = bitmap_find_next_zero_area_off(cmr.bitmap, bitmap_maxno, start, bitmap_count, mask, offset).
  - if bitmap_no >= bitmap_maxno: unlock; break.
  - pfn = cmr.base_pfn + (bitmap_no << cma.order_per_bit).
  - page = pfn_to_page(pfn).
  - if !page_range_contiguous(page, count): unlock; pr_warn_ratelimited; continue (skip this slot).
  - bitmap_set(cmr.bitmap, bitmap_no, bitmap_count); cma.available_count -= count.
  - spin_unlock_irq(&cma.lock).
  - mutex_lock(&cma.alloc_mutex); ret = alloc_contig_frozen_range(pfn, pfn+count, ACR_FLAGS_CMA, gfp); mutex_unlock(&cma.alloc_mutex).
  - if !ret: break (success).
  - cma_clear_bitmap(cma, cmr, pfn, count); if ret != -EBUSY: break.
  - trace_cma_alloc_busy_retry; loop.
- on success: *pagep = page; return 0. Otherwise return ret (-EBUSY/-ENOMEM/etc.).

REQ-18: cma_release(cma, pages, count) → bool:
- cmr = find_cma_memrange(cma, pages, count).
- if !cmr: return false (pages not in this CMA).
- pfn = page_to_pfn(pages).
- for i in 0..count: ret += !put_page_testzero(pfn_to_page(pfn+i)) (counts pages still in use).
- WARN(ret, "%lu pages are still in use!").
- __cma_release_frozen(cma, cmr, pages, count): free_contig_frozen_range; cma_clear_bitmap; cma_sysfs_account_release_pages; trace_cma_release.
- return true.

REQ-19: cma_release_frozen(cma, pages, count) → bool:
- Same as cma_release minus the put_page_testzero loop (frozen pages have refcount 0).

REQ-20: find_cma_memrange(cma, pages, count):
- pfn = page_to_pfn(pages).
- For r in 0..cma.nranges: if pfn ∈ [base_pfn, base_pfn+count): break (and VM_WARN_ON_ONCE if pfn+count exceeds range).
- Return matching cmr or NULL.

REQ-21: cma_clear_bitmap(cma, cmr, pfn, count):
- bitmap_no = (pfn - cmr.base_pfn) >> cma.order_per_bit.
- bitmap_count = cma_bitmap_pages_to_bits(cma, count).
- spin_lock_irqsave; bitmap_clear(cmr.bitmap, bitmap_no, bitmap_count); cma.available_count += count; spin_unlock_irqrestore.

REQ-22: cma_for_each_area(it, data):
- for i in 0..cma_area_count: ret = it(&cma_areas[i], data); if ret: return ret. Return 0.

REQ-23: cma_intersects(cma, start, end) (physical address range):
- For each range cmr: rstart=PFN_PHYS(base_pfn); rend=PFN_PHYS(base_pfn+count). Return true if !(end<rstart ∨ start≥rend).

REQ-24: cma_reserve_early(cma, size):
- Early-boot bumper allocator, single-threaded, no locking.
- if !cma ∨ !cma.count ∨ test_bit(CMA_ACTIVATED): return NULL.
- if !IS_ALIGNED(size, CMA_MIN_ALIGNMENT_BYTES) ∨ !IS_ALIGNED(size, PAGE_SIZE << order_per_bit): return NULL.
- size >>= PAGE_SHIFT.
- if size > cma.available_count: return NULL.
- For each range: available = cmr.count - (cmr.early_pfn - cmr.base_pfn). If size ≤ available: ret = phys_to_virt(PFN_PHYS(cmr.early_pfn)); cmr.early_pfn += size; cma.available_count -= size; return ret.

REQ-25: Accessors:
- cma_get_base(cma): WARN_ON_ONCE(cma.nranges != 1); PFN_PHYS(cma.ranges[0].base_pfn).
- cma_get_size(cma): cma.count << PAGE_SHIFT.
- cma_get_name(cma): cma.name.

REQ-26: Bitmap helpers:
- cma_bitmap_aligned_mask(cma, align_order): align_order ≤ order_per_bit ⟹ 0, else (1<<(align_order-order_per_bit))-1.
- cma_bitmap_aligned_offset(cma, cmr, align_order): (cmr.base_pfn & ((1<<align_order)-1)) >> order_per_bit.
- cma_bitmap_pages_to_bits(cma, pages): ALIGN(pages, 1<<order_per_bit) >> order_per_bit.
- cma_bitmap_maxno(cma, cmr): cmr.count >> order_per_bit (from cma.h header).

REQ-27: MIGRATE_CMA semantics (consumed by page-allocator):
- init_cma_reserved_pageblock marks the pageblock MIGRATE_CMA at activation.
- Allocator may dispense MIGRATE_CMA pages only to movable allocations (and via the explicit cma_alloc path); MIGRATE_CMA pageblocks fall back from MIGRATE_MOVABLE on contention.
- alloc_contig_range / alloc_contig_frozen_range with ACR_FLAGS_CMA migrates non-movable squatters out of the requested range using migrate_pages.

REQ-28: cma_debug_show_areas(cma):
- Under spin_lock_irq(&cma.lock).
- For each range: iterate clear bit-ranges, print "+%lu@%lu" (nr_part, start_bit).
- Trailing "=> %lu free of %lu total pages\n".

REQ-29: Concurrency model:
- cma.lock (spinlock_t, irq-disabling): protects bitmap (per-range) + available_count.
- cma.alloc_mutex: serialises alloc_contig_frozen_range so concurrent migrations on the same area do not contend on isolation state.
- cma_areas + cma_area_count: written only at boot (single-threaded init); read everywhere else.

REQ-30: Error / accounting paths:
- count_vm_event(CMA_ALLOC_SUCCESS) / (CMA_ALLOC_FAIL).
- cma_sysfs_account_success_pages / _fail_pages / _release_pages (sysfs counters per area).
- totalcma_pages global mirrors sum of cma->count across all areas (adjusted by cma_new_area / cma_drop_area / activation cleanup).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `init_alignment_required` | INVARIANT | per-init_reserved_mem: base ∧ size aligned to CMA_MIN_ALIGNMENT_BYTES. |
| `cma_area_count_bounded` | INVARIANT | cma_area_count ≤ MAX_CMA_AREAS. |
| `nranges_bounded` | INVARIANT | cma.nranges ≤ CMA_MAX_RANGES. |
| `available_count_never_exceeds_count` | INVARIANT | per-alloc / release: 0 ≤ available_count ≤ count. |
| `bitmap_alloc_size_eq_maxno` | INVARIANT | per-activate: bitmap_zalloc size == cma_bitmap_maxno(cma, cmr). |
| `zone_invariant_memoised` | INVARIANT | per-validate_zones: at most one of CMA_ZONES_VALID / _INVALID set. |
| `alloc_lock_protects_bitmap` | INVARIANT | per-range_alloc: bitmap_set / available_count modified under cma.lock. |
| `alloc_contig_under_mutex` | INVARIANT | per-range_alloc: alloc_contig_frozen_range called under alloc_mutex. |
| `release_lookup_required` | INVARIANT | per-release: cmr := find_memrange must succeed before clear_bitmap. |
| `reserve_early_pre_activation` | INVARIANT | per-reserve_early: CMA_ACTIVATED unset ⟹ early_pfn advance allowed. |

### Layer 2: TLA+

`mm/cma.tla`:
- Per-init-reserved + per-declare-contiguous + per-activate + per-alloc + per-release + per-multi-range + per-reserve-early.
- Properties:
  - `safety_no_overlap` — per-activate: ranges within a single cma area are disjoint.
  - `safety_single_zone_per_range` — per-validate_zones: each range entirely in one zone.
  - `safety_bitmap_accounts_available` — per-alloc/release: popcount(bitmap)*1<<order_per_bit + available_count == count.
  - `safety_alloc_mutex_exclusive` — per-area: at most one in-flight alloc_contig_frozen_range.
  - `safety_release_idempotent_only_on_owned` — per-release: pfn not in any range ⟹ no state change.
  - `liveness_alloc_retries_bounded` — per-alloc: -EBUSY retries bounded by bitmap_maxno.
  - `liveness_activate_or_returns_pages_to_buddy` — per-activate-fail: free_reserved_page for every pfn unless CMA_RESERVE_PAGES_ON_ERROR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cma::init_reserved_mem` post: nranges=1, range[0]=(PFN_DOWN(base), count=size>>PAGE_SHIFT) | `Cma::init_reserved_mem` |
| `Cma::activate_area` post: CMA_ACTIVATED set ⟺ all bitmap_zalloc succeeded ∧ validate_zones | `Cma::activate_area` |
| `Cma::range_alloc` post: ret=0 ⟹ bitmap[bitmap_no..+bitmap_count] all set ∧ alloc_contig_frozen_range succeeded | `Cma::range_alloc` |
| `Cma::release` post: bitmap cleared ∧ available_count += count | `Cma::release` |
| `Cma::reserve_early` post: cmr.early_pfn advanced by size ∧ available_count -= size | `Cma::reserve_early` |
| `Cma::validate_zones` post: CMA_ZONES_VALID ⊕ CMA_ZONES_INVALID set after first call | `Cma::validate_zones` |
| `Cma::declare_contiguous_multi` post: sum(range.count) ≥ total_size ∧ nr ≤ CMA_MAX_RANGES | `Cma::declare_contiguous_multi` |

### Layer 4: Verus/Creusot functional

`Per-cma_init_reserved_mem → core_initcall cma_activate_area → cma_alloc (bitmap_find_next_zero_area_off + alloc_contig_frozen_range) → cma_release (put_page_testzero + free_contig_frozen_range + cma_clear_bitmap)` semantic equivalence: per-Documentation/admin-guide/mm/cma_debugfs.rst and per-mm/Kconfig (CMA, CMA_AREAS, CMA_ALIGNMENT).

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

CMA reinforcement:

- **Per-init alignment enforced (CMA_MIN_ALIGNMENT_BYTES)** — defense against per-misaligned pageblock that breaks alloc_contig_range.
- **Per-area zone single-zone invariant (cma_validate_zones)** — defense against per-cross-zone allocation corrupting buddy.
- **Per-MAX_CMA_AREAS / CMA_MAX_RANGES static bound** — defense against per-unbounded boot-time slot exhaustion.
- **Per-cma.lock IRQ-safe spinlock on bitmap** — defense against per-IRQ-during-alloc race.
- **Per-alloc_mutex serialising migrate** — defense against per-double-migrate on overlapping isolations.
- **Per-page_range_contiguous skip** — defense against per-handing-out-discontiguous PFN to a caller that assumes contiguity.
- **Per-cma_release WARN on still-referenced pages** — defense against per-double-free of caller-pinned pages.
- **Per-find_cma_memrange bounds check + VM_WARN_ON_ONCE** — defense against per-caller passing pages outside the area.
- **Per-pageblock_order pre-init check** — defense against per-too-early init before mm core ready.
- **Per-CMA_RESERVE_PAGES_ON_ERROR opt-in** — defense against per-buddy ingesting reserved-for-firmware pages on activate failure.
- **Per-trace_cma_alloc_busy_retry telemetry** — defense against per-silent migration thrash.
- **Per-CMA_ACTIVATED check in cma_reserve_early** — defense against per-bump-allocator-after-bitmap-init corruption.
- **Per-kmemleak_ignore_phys on non-fixed reservations** — defense against per-false-positive leak report on unmapped physical memory.

