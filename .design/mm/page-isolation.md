# Tier-3: mm/page_isolation.c — pageblock isolation for hot-remove and alloc_contig_range

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/page_isolation.c (~654 lines)
  - include/linux/page-isolation.h (enum pb_isolate_mode, MIGRATE_ISOLATE, pageblock helpers)
  - include/linux/pageblock-flags.h
  - include/trace/events/page_isolation.h (trace_test_pages_isolated)
-->

## Summary

**Page isolation** marks one or more pageblocks with `MIGRATE_ISOLATE` so the page allocator will not satisfy allocations from those blocks while migrating away or freeing their pages. It is the precondition for:
- **Memory hot-remove** (mode `PB_ISOLATE_MODE_MEM_OFFLINE`): mark a span unavailable so `offline_pages()` can migrate and free it.
- **`alloc_contig_range` / CMA allocation** (mode `PB_ISOLATE_MODE_CMA_ALLOC`): mark a span unavailable so its pages can be migrated out, leaving a contiguous physically-aligned region for the caller.

Granularity is one pageblock (`pageblock_nr_pages`). The boundary pageblocks may share a buddy-order page with adjacent (non-isolated) pageblocks; `isolate_single_pageblock()` handles this by splitting the straddling free page or migrating the straddling in-use compound page. Mid-range pageblocks are isolated by `set_migratetype_isolate()`, which probes for unmovable pages via `has_unmovable_pages()` / `page_is_unmovable()` before flipping the migratetype under `zone->lock`. Reversal is `unset_migratetype_isolate()` (single block) or `undo_isolate_page_range()` (full range). Caller checks completion via `test_pages_isolated()` — all PFNs in the range must be either free (`PageBuddy`), `PageHWPoison` (offline-only), or `PageOffline` with refcount 0 (offline-only). Critical for: memory hotplug, huge-page allocation, CMA contiguous allocator, virtio-balloon, DMA-coherent reservation.

This Tier-3 covers `mm/page_isolation.c` (~654 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum pb_isolate_mode` | per-call mode (MEM_OFFLINE / CMA_ALLOC / other) | `pb::IsolateMode` |
| `MIGRATE_ISOLATE` | per-pageblock migratetype | `MigrateType::Isolate` |
| `start_isolate_page_range()` | per-range isolate entry | `PageIsolation::start_isolate_range` |
| `undo_isolate_page_range()` | per-range undo | `PageIsolation::undo_isolate_range` |
| `test_pages_isolated()` | per-range completion test | `PageIsolation::test_pages_isolated` |
| `isolate_single_pageblock()` | per-boundary-block isolate | `PageIsolation::isolate_single_block` |
| `set_migratetype_isolate()` | per-block set ISOLATE | `PageIsolation::set_migratetype_isolate` |
| `unset_migratetype_isolate()` | per-block clear ISOLATE | `PageIsolation::unset_migratetype_isolate` |
| `has_unmovable_pages()` | per-range unmovable probe | `PageIsolation::has_unmovable_pages` |
| `page_is_unmovable()` | per-page unmovable probe | `PageIsolation::page_is_unmovable` |
| `__test_page_isolated_in_pageblock()` | per-block free-test | `PageIsolation::test_isolated_in_block` |
| `__first_valid_page()` | per-pfn-range first online page | `PageIsolation::first_valid_page` |
| `pageblock_isolate_and_move_free_pages()` | per-block move free pages to ISOLATE freelist | shared |
| `pageblock_unisolate_and_move_free_pages()` | per-block move free pages back | shared |
| `clear_pageblock_isolate()` | per-block clear flag | shared |
| `get_pageblock_isolate()` | per-block read flag | shared |
| `is_migrate_isolate_page()` | per-page query | shared |
| `is_migrate_cma_page()` | per-page CMA query | shared |
| `__isolate_free_page()` | per-page free-list removal | shared |
| `__putback_isolated_page()` | per-page free-list return | shared |
| `wait_for_freed_hugetlb_folios()` | per-test sync hugetlb-free | shared |
| `accept_page()` / `PageUnaccepted` | per-unaccepted-memory accept | shared |
| `dump_page()` | per-unmovable diag dump | shared |
| `trace_test_pages_isolated` | per-call tracepoint | shared |

## Compatibility contract

REQ-1: enum `pb_isolate_mode`:
- `PB_ISOLATE_MODE_MEM_OFFLINE` — caller is memory hot-remove path; HWPoisoned and refcount-0 PageOffline counted as movable.
- `PB_ISOLATE_MODE_CMA_ALLOC` — caller is alloc_contig_range / CMA; CMA pageblocks treated as movable.
- Other modes — strict mode: any reserved / refcounted / non-LRU / non-movable-ops page fails the probe.

REQ-2: `start_isolate_page_range(start_pfn, end_pfn, mode) -> int`:
- `isolate_start = pageblock_start_pfn(start_pfn)`.
- `isolate_end = pageblock_align(end_pfn)`.
- /* Isolate first boundary block */
- ret = `isolate_single_pageblock(isolate_start, mode, /*isolate_before=*/false, /*skip_isolation=*/false)`.
- if ret: return ret.
- /* If range is one block, second call may re-target the same block — skip re-isolation */
- if `isolate_start == isolate_end - pageblock_nr_pages`: `skip_isolation = true`.
- /* Isolate last boundary block */
- ret = `isolate_single_pageblock(isolate_end, mode, /*isolate_before=*/true, skip_isolation)`.
- if ret: `unset_migratetype_isolate(pfn_to_page(isolate_start))`; return ret.
- /* Mid-range: one pageblock at a time */
- for pfn in `[isolate_start + pageblock_nr_pages, isolate_end - pageblock_nr_pages)` step `pageblock_nr_pages`:
  - page = `__first_valid_page(pfn, pageblock_nr_pages)`.
  - if page ∧ `set_migratetype_isolate(page, mode, start_pfn, end_pfn) != 0`:
    - `undo_isolate_page_range(isolate_start, pfn)`.
    - `unset_migratetype_isolate(pfn_to_page(isolate_end - pageblock_nr_pages))`.
    - return -EBUSY.
- return 0.

REQ-3: `undo_isolate_page_range(start_pfn, end_pfn)`:
- `isolate_start = pageblock_start_pfn(start_pfn)`.
- `isolate_end = pageblock_align(end_pfn)`.
- for pfn in `[isolate_start, isolate_end)` step `pageblock_nr_pages`:
  - page = `__first_valid_page(pfn, pageblock_nr_pages)`.
  - if !page ∨ `!is_migrate_isolate_page(page)`: continue.
  - `unset_migratetype_isolate(page)`.

REQ-4: `set_migratetype_isolate(page, mode, start_pfn, end_pfn) -> int`:
- zone = `page_zone(page)`.
- if `PageUnaccepted(page)`: `accept_page(page)`.
- spin_lock_irqsave(&zone->lock, flags).
- if `is_migrate_isolate_page(page)`: unlock; return -EBUSY (already isolated by another thread).
- check_unmovable_start = `max(page_to_pfn(page), start_pfn)`.
- check_unmovable_end = `min(pageblock_end_pfn(page_to_pfn(page)), end_pfn)`.
- unmovable = `has_unmovable_pages(check_unmovable_start, check_unmovable_end, mode)`.
- if !unmovable:
  - if `!pageblock_isolate_and_move_free_pages(zone, page)`: unlock; return -EBUSY.
  - `zone->nr_isolate_pageblock++`.
  - unlock; return 0.
- /* Failure path */
- unlock.
- if mode == `PB_ISOLATE_MODE_MEM_OFFLINE`: `dump_page(unmovable, "unmovable page")` — deferred out of `zone->lock` to avoid lockdep splat.
- return -EBUSY.

REQ-5: `unset_migratetype_isolate(page)`:
- zone = `page_zone(page)`.
- spin_lock_irqsave(&zone->lock, flags).
- if `!is_migrate_isolate_page(page)`: goto out.
- isolated_page = false.
- /* Buddy-merge fixup: large free buddy straddling unisolated neighbor */
- if `PageBuddy(page)`:
  - order = `buddy_order(page)`.
  - if `order >= pageblock_order ∧ order < MAX_PAGE_ORDER`:
    - buddy = `find_buddy_page_pfn(page, page_to_pfn(page), order, NULL)`.
    - if buddy ∧ `!is_migrate_isolate_page(buddy)`:
      - isolated_page = `!!__isolate_free_page(page, order)`.
      - `VM_WARN_ON(!isolated_page)` — isolation in already-isolated block must not fail watermarks.
- if !isolated_page:
  - `WARN_ON_ONCE(!pageblock_unisolate_and_move_free_pages(zone, page))`.
- else:
  - `clear_pageblock_isolate(page)`.
  - `__putback_isolated_page(page, order, get_pageblock_migratetype(page))`.
- `zone->nr_isolate_pageblock--`.
- out: spin_unlock_irqrestore(&zone->lock, flags).

REQ-6: `has_unmovable_pages(start_pfn, end_pfn, mode) -> struct page *`:
- `VM_BUG_ON(pageblock_start_pfn(start_pfn) != pageblock_start_pfn(end_pfn - 1))` — single pageblock only.
- page = `pfn_to_page(start_pfn)`; zone = `page_zone(page)`.
- if `is_migrate_cma_page(page)`:
  - if mode == `PB_ISOLATE_MODE_CMA_ALLOC`: return NULL (CMA isolated for CMA-alloc is OK).
  - else: return page.
- while `start_pfn < end_pfn`:
  - step = 1.
  - page = `pfn_to_page(start_pfn)`.
  - if `page_is_unmovable(zone, page, mode, &step)`: return page.
  - start_pfn += step.
- return NULL.

REQ-7: `page_is_unmovable(zone, page, mode, *step) -> bool`:
- if `PageReserved(page)`: return true (bootmem / hole).
- if `zone_idx(zone) == ZONE_MOVABLE`: return false (everything else movable here).
- if `PageHuge(page) ∨ PageCompound(page)`:
  - folio = `page_folio(page)`.
  - if `folio_test_hugetlb(folio)`:
    - if `!IS_ENABLED(CONFIG_ARCH_ENABLE_HUGEPAGE_MIGRATION)`: return true.
    - h = `size_to_hstate(folio_size(folio))` (`folio_hstate()` unsafe — folio may be freed).
    - if h ∧ `!hugepage_migration_supported(h)`: return true.
  - else if `!folio_test_lru(folio)`: return true.
  - `*step = folio_nr_pages(folio) - folio_page_idx(folio, page)`.
  - return false.
- if `!page_ref_count(page)`:
  - if `PageBuddy(page)`: `*step = 1 << buddy_order(page)`.
  - return false (free page == movable).
- if mode == `PB_ISOLATE_MODE_MEM_OFFLINE`:
  - if `PageHWPoison(page)`: return false (will skip on offline).
  - if `PageOffline(page)`: return false (driver may release in MEM_GOING_OFFLINE; resolved later).
- if `PageLRU(page) ∨ page_has_movable_ops(page)`: return false.
- return true (refcounted, non-LRU, non-movable-ops, non-free, non-hugetlb).

REQ-8: `isolate_single_pageblock(boundary_pfn, mode, isolate_before, skip_isolation) -> int`:
- `VM_BUG_ON(!pageblock_aligned(boundary_pfn))`.
- if isolate_before: `isolate_pageblock = boundary_pfn - pageblock_nr_pages`.
- else: `isolate_pageblock = boundary_pfn`.
- zone = `page_zone(pfn_to_page(isolate_pageblock))`.
- start_pfn = `max(ALIGN_DOWN(isolate_pageblock, MAX_ORDER_NR_PAGES), zone->zone_start_pfn)`.
- if skip_isolation: `VM_BUG_ON(!get_pageblock_isolate(pfn_to_page(isolate_pageblock)))`.
- else: ret = `set_migratetype_isolate(pfn_to_page(isolate_pageblock), mode, isolate_pageblock, isolate_pageblock + pageblock_nr_pages)`; if ret: return ret.
- /* Bail-out: boundary does not form a straddling page */
- if isolate_before ∧ `!pfn_to_online_page(boundary_pfn)`: return 0.
- if !isolate_before ∧ `!pfn_to_online_page(boundary_pfn - 1)`: return 0.
- for pfn in [start_pfn, boundary_pfn):
  - page = `__first_valid_page(pfn, boundary_pfn - pfn)`.
  - `VM_BUG_ON(!page)`.
  - pfn = `page_to_pfn(page)`.
  - if `PageUnaccepted(page)`: pfn += `MAX_ORDER_NR_PAGES`; continue.
  - if `PageBuddy(page)`:
    - order = `buddy_order(page)`.
    - `VM_WARN_ON_ONCE(pfn + (1 << order) > boundary_pfn)` — split done in `pageblock_isolate_and_move_free_pages`.
    - pfn += `1UL << order`; continue.
  - if `PageCompound(page)`:
    - head = `compound_head(page)`; head_pfn = `page_to_pfn(head)`; nr_pages = `compound_nr(head)`.
    - if `head_pfn + nr_pages <= boundary_pfn ∨ PageHuge(page)`: pfn = head_pfn + nr_pages; continue.
    - `VM_WARN_ON_ONCE_PAGE(PageLRU(page), page)` — LRU never exceeds pageblock_order.
    - `VM_WARN_ON_ONCE_PAGE(page_has_movable_ops(page), page)` — movable-ops never exceeds pageblock_order.
    - goto failed.
  - pfn++.
- return 0.
- failed:
- if !skip_isolation: `unset_migratetype_isolate(pfn_to_page(isolate_pageblock))`.
- return -EBUSY.

REQ-9: `test_pages_isolated(start_pfn, end_pfn, mode) -> int`:
- `wait_for_freed_hugetlb_folios()` — sync hugetlb deferred-free so PageBuddy can be observed.
- for pfn in [start_pfn, end_pfn) step `pageblock_nr_pages`:
  - page = `__first_valid_page(pfn, pageblock_nr_pages)`.
  - if page ∧ `!is_migrate_isolate_page(page)`: break.
- page = `__first_valid_page(start_pfn, end_pfn - start_pfn)`.
- if pfn < end_pfn ∨ !page: ret = -EBUSY; goto out.
- zone = `page_zone(page)`.
- spin_lock_irqsave(&zone->lock, flags).
- pfn = `__test_page_isolated_in_pageblock(start_pfn, end_pfn, mode)`.
- spin_unlock_irqrestore(&zone->lock, flags).
- ret = (pfn < end_pfn) ? -EBUSY : 0.
- out: `trace_test_pages_isolated(start_pfn, end_pfn, pfn)`; return ret.

REQ-10: `__test_page_isolated_in_pageblock(pfn, end_pfn, mode) -> unsigned long`:
- while pfn < end_pfn:
  - page = `pfn_to_page(pfn)`.
  - if `PageBuddy(page)`: pfn += `1 << buddy_order(page)`.
  - else if mode == `PB_ISOLATE_MODE_MEM_OFFLINE` ∧ `PageHWPoison(page)`: pfn++.
  - else if mode == `PB_ISOLATE_MODE_MEM_OFFLINE` ∧ `PageOffline(page) ∧ !page_count(page)`: pfn++.
  - else: break.
- return pfn.

REQ-11: `__first_valid_page(pfn, nr_pages) -> struct page *`:
- for i in [0, nr_pages):
  - page = `pfn_to_online_page(pfn + i)`.
  - if !page: continue.
  - return page.
- return NULL.

REQ-12: `zone->lock` held for: set/unset isolate, test_pages_isolated inner test, free-page moves.

REQ-13: `zone->nr_isolate_pageblock` accounts isolated blocks (incremented in set, decremented in unset); consulted by allocator to short-circuit watermarks / freepage accounting.

REQ-14: Overlap discipline:
- No high-level lock prevents two threads isolating overlapping ranges.
- Conflict detected at `set_migratetype_isolate`: already-isolated returns -EBUSY.
- On any per-block error, caller unwinds via `undo_isolate_page_range` for partial progress + explicit `unset_migratetype_isolate` for the boundary block just isolated.

REQ-15: pcplist quirk:
- Pages may still leak through pcplists into isolated blocks.
- Caller may bracket with `zone_pcp_disable()` / `zone_pcp_enable()` for stronger guarantee.
- `drain_all_pages()` recommended after isolation for best-effort flush.

REQ-16: HugeTLB / Compound straddle handling:
- Boundary-block compound page that straddles the boundary triggers `failed` unless it is a HugeTLB folio (`PageHuge`) or fully contained within boundary.
- HugeTLB folios are migrated by the caller (alloc_contig_range path); they are not split in-place.

## Acceptance Criteria

- [ ] AC-1: `start_isolate_page_range` on a clean, all-movable range returns 0; every pageblock in [start, end) is `is_migrate_isolate_page`.
- [ ] AC-2: A range containing a `PageReserved` page returns -EBUSY with mode `PB_ISOLATE_MODE_CMA_ALLOC`.
- [ ] AC-3: Mode `PB_ISOLATE_MODE_MEM_OFFLINE` treats `PageHWPoison` as movable (does not fail the probe).
- [ ] AC-4: Mode `PB_ISOLATE_MODE_MEM_OFFLINE` treats `PageOffline` (count 0) as movable.
- [ ] AC-5: `is_migrate_cma_page` block with mode `PB_ISOLATE_MODE_CMA_ALLOC` is treated as movable.
- [ ] AC-6: `is_migrate_cma_page` block with mode `PB_ISOLATE_MODE_MEM_OFFLINE` fails the probe.
- [ ] AC-7: `undo_isolate_page_range` restores every previously-isolated block's migratetype; `nr_isolate_pageblock` returns to pre-call value.
- [ ] AC-8: Two threads calling `start_isolate_page_range` on overlapping ranges: one succeeds, the other gets -EBUSY; no permanent leak.
- [ ] AC-9: Boundary pageblock that shares a buddy-order-N (N ≥ pageblock_order) free page with adjacent non-isolated block: `isolate_single_pageblock` splits the buddy via `pageblock_isolate_and_move_free_pages`.
- [ ] AC-10: Boundary pageblock straddled by a `PageHuge` folio: `isolate_single_pageblock` skips past the folio without failing.
- [ ] AC-11: Boundary pageblock straddled by a non-Huge compound page that exceeds the boundary: `isolate_single_pageblock` returns -EBUSY and undoes its own boundary isolation.
- [ ] AC-12: `test_pages_isolated` returns 0 iff all PFNs in the range are PageBuddy (or HWPoison/Offline-count-0 in MEM_OFFLINE mode).
- [ ] AC-13: `test_pages_isolated` waits for hugetlb deferred-free via `wait_for_freed_hugetlb_folios` before scanning.
- [ ] AC-14: `set_migratetype_isolate` on a `PageUnaccepted` block calls `accept_page` first.
- [ ] AC-15: After successful `unset_migratetype_isolate` on a block holding an order ≥ pageblock_order free buddy that crosses into a non-isolated neighbor, the buddy is split (via `__isolate_free_page` + `__putback_isolated_page`) so freepage accounting remains coherent.

## Architecture

```
enum IsolateMode {
  Other,
  MemOffline,                 // PB_ISOLATE_MODE_MEM_OFFLINE
  CmaAlloc,                   // PB_ISOLATE_MODE_CMA_ALLOC
}

struct PageIsolation;          // unit type; all methods static
```

`PageIsolation::start_isolate_range(start_pfn, end_pfn, mode: IsolateMode) -> Result<(), Errno>`:
1. isolate_start = pageblock_start_pfn(start_pfn).
2. isolate_end = pageblock_align(end_pfn).
3. PageIsolation::isolate_single_block(isolate_start, mode, /*before=*/false, /*skip=*/false)?
4. let skip = isolate_start == isolate_end - pageblock_nr_pages.
5. match PageIsolation::isolate_single_block(isolate_end, mode, /*before=*/true, skip) {
     Ok(()) => {}
     Err(e) => { PageIsolation::unset_migratetype_isolate(pfn_to_page(isolate_start)); return Err(e); }
   }
6. let mut pfn = isolate_start + pageblock_nr_pages.
7. while pfn < isolate_end - pageblock_nr_pages:
   - if let Some(page) = first_valid_page(pfn, pageblock_nr_pages):
     - if PageIsolation::set_migratetype_isolate(page, mode, start_pfn, end_pfn).is_err():
       - PageIsolation::undo_isolate_range(isolate_start, pfn).
       - PageIsolation::unset_migratetype_isolate(pfn_to_page(isolate_end - pageblock_nr_pages)).
       - return Err(EBUSY).
   - pfn += pageblock_nr_pages.
8. Ok(()).

`PageIsolation::undo_isolate_range(start_pfn, end_pfn)`:
1. for pfn in [pageblock_start_pfn(start_pfn), pageblock_align(end_pfn)).step_by(pageblock_nr_pages):
   - let Some(page) = first_valid_page(pfn, pageblock_nr_pages) else { continue };
   - if !is_migrate_isolate_page(page): continue.
   - PageIsolation::unset_migratetype_isolate(page).

`PageIsolation::set_migratetype_isolate(page, mode, start_pfn, end_pfn) -> Result<(), Errno>`:
1. let zone = page_zone(page).
2. if PageUnaccepted(page): accept_page(page).
3. let _g = zone.lock.lock_irqsave().
4. if is_migrate_isolate_page(page): return Err(EBUSY).
5. let cu_start = max(page_to_pfn(page), start_pfn).
6. let cu_end = min(pageblock_end_pfn(page_to_pfn(page)), end_pfn).
7. let unmovable = PageIsolation::has_unmovable_pages(cu_start, cu_end, mode).
8. if unmovable.is_none():
   - if !pageblock_isolate_and_move_free_pages(zone, page): return Err(EBUSY).
   - zone.nr_isolate_pageblock += 1.
   - return Ok(()).
9. drop(_g).
10. if mode == IsolateMode::MemOffline: dump_page(unmovable, "unmovable page").
11. Err(EBUSY).

`PageIsolation::unset_migratetype_isolate(page)`:
1. let zone = page_zone(page).
2. let _g = zone.lock.lock_irqsave().
3. if !is_migrate_isolate_page(page): return.
4. let mut isolated_page = false; let mut order = 0;
5. if PageBuddy(page):
   - order = buddy_order(page).
   - if order >= pageblock_order && order < MAX_PAGE_ORDER:
     - if let Some(buddy) = find_buddy_page_pfn(page, page_to_pfn(page), order, None):
       - if !is_migrate_isolate_page(buddy):
         - isolated_page = !!__isolate_free_page(page, order).
         - VM_WARN_ON(!isolated_page).
6. if !isolated_page:
   - WARN_ON_ONCE(!pageblock_unisolate_and_move_free_pages(zone, page)).
7. else:
   - clear_pageblock_isolate(page).
   - __putback_isolated_page(page, order, get_pageblock_migratetype(page)).
8. zone.nr_isolate_pageblock -= 1.

`PageIsolation::has_unmovable_pages(start_pfn, end_pfn, mode) -> Option<*Page>`:
1. VM_BUG_ON!(pageblock_start_pfn(start_pfn) == pageblock_start_pfn(end_pfn - 1)).
2. let page = pfn_to_page(start_pfn).
3. let zone = page_zone(page).
4. if is_migrate_cma_page(page):
   - return if mode == IsolateMode::CmaAlloc { None } else { Some(page) }.
5. let mut pfn = start_pfn.
6. while pfn < end_pfn:
   - let mut step: u64 = 1.
   - let p = pfn_to_page(pfn).
   - if PageIsolation::page_is_unmovable(zone, p, mode, &mut step): return Some(p).
   - pfn += step.
7. None.

`PageIsolation::page_is_unmovable(zone, page, mode, step) -> bool`:
1. if PageReserved(page): return true.
2. if zone_idx(zone) == ZONE_MOVABLE: return false.
3. if PageHuge(page) || PageCompound(page):
   - let folio = page_folio(page).
   - if folio_test_hugetlb(folio):
     - if !cfg!(arch_enable_hugepage_migration): return true.
     - let h = size_to_hstate(folio_size(folio)).
     - if h.is_some() && !hugepage_migration_supported(h.unwrap()): return true.
   - else if !folio_test_lru(folio): return true.
   - *step = folio_nr_pages(folio) - folio_page_idx(folio, page).
   - return false.
4. if page_ref_count(page) == 0:
   - if PageBuddy(page): *step = 1 << buddy_order(page).
   - return false.
5. if mode == IsolateMode::MemOffline:
   - if PageHWPoison(page): return false.
   - if PageOffline(page): return false.
6. if PageLRU(page) || page_has_movable_ops(page): return false.
7. true.

`PageIsolation::isolate_single_block(boundary_pfn, mode, isolate_before, skip_isolation) -> Result<(), Errno>`:
1. VM_BUG_ON!(pageblock_aligned(boundary_pfn)).
2. let isolate_pageblock = if isolate_before { boundary_pfn - pageblock_nr_pages } else { boundary_pfn }.
3. let zone = page_zone(pfn_to_page(isolate_pageblock)).
4. let start_pfn = max(align_down(isolate_pageblock, MAX_ORDER_NR_PAGES), zone.zone_start_pfn).
5. if skip_isolation:
   - VM_BUG_ON!(get_pageblock_isolate(pfn_to_page(isolate_pageblock))).
6. else:
   - PageIsolation::set_migratetype_isolate(pfn_to_page(isolate_pageblock), mode, isolate_pageblock, isolate_pageblock + pageblock_nr_pages)?
7. /* Early-exit if no straddling page */
8. if isolate_before && pfn_to_online_page(boundary_pfn).is_none(): return Ok(()).
9. if !isolate_before && pfn_to_online_page(boundary_pfn - 1).is_none(): return Ok(()).
10. let mut pfn = start_pfn.
11. while pfn < boundary_pfn:
    - let page = first_valid_page(pfn, boundary_pfn - pfn).expect("VM_BUG: no valid page").
    - pfn = page_to_pfn(page).
    - if PageUnaccepted(page): pfn += MAX_ORDER_NR_PAGES; continue.
    - if PageBuddy(page):
      - let order = buddy_order(page).
      - VM_WARN_ON_ONCE!(pfn + (1 << order) <= boundary_pfn).
      - pfn += 1 << order; continue.
    - if PageCompound(page):
      - let head = compound_head(page).
      - let head_pfn = page_to_pfn(head).
      - let nr_pages = compound_nr(head).
      - if head_pfn + nr_pages <= boundary_pfn || PageHuge(page):
        - pfn = head_pfn + nr_pages; continue.
      - VM_WARN_ON_ONCE_PAGE!(PageLRU(page), page).
      - VM_WARN_ON_ONCE_PAGE!(page_has_movable_ops(page), page).
      - goto failed.
    - pfn += 1.
12. return Ok(()).
13. failed:
14. if !skip_isolation: PageIsolation::unset_migratetype_isolate(pfn_to_page(isolate_pageblock)).
15. Err(EBUSY).

`PageIsolation::test_pages_isolated(start_pfn, end_pfn, mode) -> Result<(), Errno>`:
1. wait_for_freed_hugetlb_folios().
2. let mut pfn = start_pfn.
3. while pfn < end_pfn:
   - let page = first_valid_page(pfn, pageblock_nr_pages).
   - if let Some(p) = page { if !is_migrate_isolate_page(p) { break; } }.
   - pfn += pageblock_nr_pages.
4. let head_page = first_valid_page(start_pfn, end_pfn - start_pfn).
5. if pfn < end_pfn || head_page.is_none(): trace_test_pages_isolated(start_pfn, end_pfn, pfn); return Err(EBUSY).
6. let zone = page_zone(head_page.unwrap()).
7. let _g = zone.lock.lock_irqsave().
8. let stop = PageIsolation::test_isolated_in_block(start_pfn, end_pfn, mode).
9. drop(_g).
10. trace_test_pages_isolated(start_pfn, end_pfn, stop).
11. if stop < end_pfn { Err(EBUSY) } else { Ok(()) }.

`PageIsolation::test_isolated_in_block(pfn, end_pfn, mode) -> u64`:
1. let mut pfn = pfn.
2. while pfn < end_pfn:
   - let p = pfn_to_page(pfn).
   - if PageBuddy(p): pfn += 1 << buddy_order(p); continue.
   - if mode == IsolateMode::MemOffline && PageHWPoison(p): pfn += 1; continue.
   - if mode == IsolateMode::MemOffline && PageOffline(p) && page_count(p) == 0: pfn += 1; continue.
   - break.
3. pfn.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `zone_lock_held_for_set` | INVARIANT | per-set_migratetype_isolate: `zone->lock` held during state mutation. |
| `zone_lock_held_for_unset` | INVARIANT | per-unset_migratetype_isolate: `zone->lock` held during state mutation. |
| `pageblock_aligned_boundary` | INVARIANT | per-isolate_single_block: `pageblock_aligned(boundary_pfn)`. |
| `single_block_in_has_unmovable` | INVARIANT | per-has_unmovable_pages: `start_pfn`/`end_pfn-1` in same pageblock. |
| `nr_isolate_pageblock_balanced` | INVARIANT | per-set/unset pair: `nr_isolate_pageblock` delta == 0. |
| `start_isolate_undo_on_failure` | INVARIANT | per-start_isolate_range: any per-block failure ⟹ all prior blocks restored. |
| `undo_idempotent_on_unisolated` | INVARIANT | per-undo_isolate_range: skips blocks not currently isolated. |
| `cma_mem_offline_excluded` | INVARIANT | per-has_unmovable_pages: `is_migrate_cma_page ∧ mode == MEM_OFFLINE` ⟹ unmovable. |
| `hwpoison_offline_mode_treated_movable` | INVARIANT | per-page_is_unmovable: `MEM_OFFLINE ∧ PageHWPoison` ⟹ movable. |
| `pageoffline_offline_mode_treated_movable` | INVARIANT | per-page_is_unmovable: `MEM_OFFLINE ∧ PageOffline` ⟹ movable. |
| `dump_page_outside_zone_lock` | INVARIANT | per-set_migratetype_isolate failure path: `dump_page` called after lock dropped. |
| `wait_for_hugetlb_before_test` | INVARIANT | per-test_pages_isolated: `wait_for_freed_hugetlb_folios` called before scan. |

### Layer 2: TLA+

`mm/page-isolation.tla`:
- Per-pageblock state machine: `Free → Isolating → Isolated → Unisolating → Free`.
- Per-thread concurrent isolate / undo / test.
- Properties:
  - `safety_no_isolate_within_isolate` — per-block: re-isolation attempt returns EBUSY; state unchanged.
  - `safety_undo_restores_migratetype` — per-block: post-undo, `nr_isolate_pageblock` decremented; migratetype restored.
  - `safety_overlap_two_threads` — per-overlapping isolate: one wins, other returns EBUSY; no orphan-isolated block.
  - `safety_boundary_buddy_split` — per-isolate_single_block: straddling buddy of order ≥ pageblock_order is split.
  - `safety_test_pages_isolated_strict` — `test_pages_isolated == 0` ⟹ all PFNs free / HWPoison / Offline-count-0.
  - `liveness_per_isolate_terminates` — per-start_isolate_range: terminates with 0 or EBUSY.
  - `liveness_per_test_terminates` — per-test_pages_isolated: terminates after `wait_for_freed_hugetlb_folios` quiesces.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `start_isolate_range` post: Ok ⟹ every block in [pageblock_start_pfn(start), pageblock_align(end)) is isolated | `PageIsolation::start_isolate_range` |
| `start_isolate_range` post: Err ⟹ no permanent migratetype change | `PageIsolation::start_isolate_range` |
| `undo_isolate_range` post: every previously-isolated block unisolated | `PageIsolation::undo_isolate_range` |
| `set_migratetype_isolate` post: Ok ⟹ block migratetype == ISOLATE ∧ `nr_isolate_pageblock` += 1 | `PageIsolation::set_migratetype_isolate` |
| `unset_migratetype_isolate` post: block migratetype restored ∧ `nr_isolate_pageblock` -= 1 | `PageIsolation::unset_migratetype_isolate` |
| `has_unmovable_pages` post: returns first unmovable page in range or None | `PageIsolation::has_unmovable_pages` |
| `page_is_unmovable` post: mode-dependent classification follows REQ-7 truth table | `PageIsolation::page_is_unmovable` |
| `isolate_single_block` post: failed ⟹ self-isolation undone (if !skip_isolation) | `PageIsolation::isolate_single_block` |
| `test_pages_isolated` post: Ok ⟹ scan reached end_pfn | `PageIsolation::test_pages_isolated` |

### Layer 4: Verus/Creusot functional

`Per-start_isolate_page_range (isolate boundary-start → isolate boundary-end → iterate middle blocks → on error undo + restore boundaries)` semantic equivalence: per-Documentation/admin-guide/mm/memory-hotplug.rst + `CONTIGUOUS_ALLOC` semantics. `Per-test_pages_isolated (wait hugetlb → migratetype quick-scan → zone-locked free-test)` semantic equivalence: completion criterion for hot-remove and CMA paths.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Page-isolation reinforcement:

- **Per-zone->lock held for all migratetype mutations** — defense against per-concurrent-allocator race that could allocate from a half-isolated block.
- **Per-already-isolated returns -EBUSY** — defense against per-double-isolate refcount imbalance on `nr_isolate_pageblock`.
- **Per-`has_unmovable_pages` pre-probe** — defense against per-isolate of a block containing a kernel-pinned page that cannot be migrated, which would deadlock hot-remove or CMA.
- **Per-PageReserved always unmovable** — defense against per-bootmem/hole pageblock being marked isolated and later mis-accounted.
- **Per-CMA pageblock movable only in CMA mode** — defense against per-MEM_OFFLINE attempt removing a CMA region in use by drivers.
- **Per-undo_isolate_page_range on partial failure** — defense against per-orphan-isolated pageblock leak.
- **Per-`dump_page` outside `zone->lock`** — defense against per-lockdep-splat and per-printk-while-locked deadlock.
- **Per-`wait_for_freed_hugetlb_folios` before test** — defense against per-false-negative completion when hugetlb deferred-free is in flight.
- **Per-boundary buddy-split via `pageblock_isolate_and_move_free_pages`** — defense against per-cross-block buddy that would leak free pages into adjacent migratetypes during accounting.
- **Per-non-Huge compound straddle ⟹ failed** — defense against per-large-fold being silently broken; LRU/movable-ops compound that exceeds pageblock_order is unexpected and warned (`VM_WARN_ON_ONCE_PAGE`).
- **Per-`accept_page` before set on `PageUnaccepted`** — defense against per-unaccepted-memory being isolated while still in pre-accept state (TDX/SEV-SNP).
- **Per-RCU/online checks via `pfn_to_online_page`** — defense against per-isolating-pfn-of-just-removed-section UAF.
- **Per-`zone->nr_isolate_pageblock` accounted** — defense against per-allocator misbehavior when freepages cross migratetype boundaries.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/memory_hotplug.c offline / online paths (covered in `memory-hotplug.md` Tier-3)
- mm/page_alloc.c freepage accounting + `pageblock_isolate_and_move_free_pages` internals (covered in `page-allocator.md` Tier-3)
- mm/compaction.c alloc_contig_range driver (covered in `migration-compaction.md` Tier-3)
- mm/cma.c CMA allocator (covered separately if expanded)
- mm/hugetlb.c hugepage migration support / `hugepage_migration_supported` (covered in `hugetlb.md` Tier-3)
- mm/page-flags.h `PageHWPoison`, `PageOffline`, `PageReserved`, `PageBuddy` flag semantics (covered in `mm-debug.md` Tier-3 / page-allocator)
- mm/migrate.c migration framework (covered in `migration-compaction.md` Tier-3)
- arch/x86/mm/mem_encrypt* unaccepted-memory `accept_page` (covered in arch design)
- Implementation code
