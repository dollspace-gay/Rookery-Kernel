# Tier-3: mm/migrate.c + mm/compaction.c — Page migration + compaction

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/migrate.c (~2768 lines)
  - mm/compaction.c (~3351 lines)
  - include/linux/migrate.h
  - include/linux/compaction.h
-->

## Summary

Page migration moves folios between physical pages atomically (e.g., NUMA balancing, CMA, hot-unplug, THP collapse). Per-folio: isolate from LRU; alloc new folio at target; per-mapping `migrate_folio()` callback copies + remaps PTEs/PMDs; old folio freed. Compaction is **per-zone defragmenter** that uses migration: scans zone for movable pages → migrates them out of high-fragmentation regions into compacted regions, freeing contiguous chunks for high-order allocations (THP, hugetlb, vmalloc-large). Per-zone kcompactd background daemon. Per-direct compaction triggered from page-allocator on high-order alloc-fail. Critical for: THP availability, NUMA balancing, contiguous allocations.

This Tier-3 covers `migrate.c` (~2768 lines) + `compaction.c` (~3351 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `migrate_pages()` | per-list batch-migrate | `Migration::migrate_pages` |
| `migrate_folio()` | per-folio default migrate-fn | `Migration::migrate_folio` |
| `__migrate_folio()` | per-folio backend | `Migration::__migrate_folio` |
| `move_to_new_folio()` | per-folio move ptr | `Migration::move_to_new_folio` |
| `unmap_and_move_huge_folio()` | per-huge migrate | `Migration::unmap_and_move_huge_folio` |
| `buffer_migrate_folio()` | per-buffer-head migrate | `Migration::buffer_migrate_folio` |
| `filemap_migrate_folio()` | per-page-cache migrate | `Migration::filemap_migrate_folio` |
| `isolate_migratepages_range()` | per-pageblock isolate | `Compact::isolate_migratepages_range` |
| `isolate_freepages()` | per-zone isolate freepages | `Compact::isolate_freepages` |
| `compact_zone()` | per-zone compaction loop | `Compact::compact_zone` |
| `compaction_suitable()` | per-zone fragmentation check | `Compact::compaction_suitable` |
| `try_to_compact_pages()` | per-direct-compaction entry | `Compact::try_to_compact_pages` |
| `kcompactd_init()` | per-node bg daemon init | `Kcompactd::init` |
| `migrate_balanced_pgdat()` | per-pgdat balance check | `Migration::balanced_pgdat` |
| `numa_migrate_prep()` | per-NUMA-balancing setup | `Migration::numa_migrate_prep` |
| `MIGRATE_*` enum | per-page migration-type | shared |
| `migration target alloc` | per-target-page alloc fn | shared |

## Compatibility contract

REQ-1: Migration entrypoint `migrate_pages(from, get_new_page, put_new_page, private, mode, reason)`:
- from: list of folios to migrate.
- get_new_page: callback to alloc destination folio.
- mode: MIGRATE_ASYNC / MIGRATE_SYNC_LIGHT / MIGRATE_SYNC.
- reason: MR_COMPACTION / MR_NUMA_MISPLACED / MR_CONTIG_RANGE / MR_MEMORY_HOTPLUG / etc.
- Returns: # pages successfully migrated; -ENOMEM on alloc-fail.

REQ-2: Per-folio __migrate_folio:
- folio_lock(src).
- alloc dst via get_new_page.
- Per-mapping: per-mapping.migrate_folio(mapping, dst, src, mode).
- if success: unmap_and_move_huge_folio (huge) / move_to_new_folio (4KB) updates rmap.
- folio_unlock; folio_put_after_mig.

REQ-3: Per-mapping callback contract:
- mapping_migrate_folio: backend-specific (filemap_migrate_folio / buffer_migrate_folio / aops.migrate_folio).
- Default migrate_folio: simple-copy + xa_replace.
- Per-buffer-head: per-bh ref-locked move + bh_migrate.
- Per-fs custom: e.g., btrfs.

REQ-4: Per-rmap remap:
- src is reverse-mapped via vma_rmap_walk; per-VMA: pte_at = pte_modify(src) → migration-PTE.
- After dst-install: pte_modify per-VMA to pfn(dst).

REQ-5: Per-migration-PTE:
- pte_to_swap_entry; SWP_MIGRATION_READ / _WRITE.
- Faulting task waits on dst alloc complete.

REQ-6: Compaction `compact_zone(zone, cc)`:
- cc: compact_control { migrate_pfn, free_pfn, mode, ... }.
- migrate_pfn cursor: scans up; isolates LRU folios.
- free_pfn cursor: scans down; isolates free pageblocks.
- Migrate isolated → free; advance cursors.
- Continue until cursors meet.

REQ-7: Per-zone isolate_migratepages_range:
- Per-pageblock: scan PFNs in [start, end).
- Per-LRU-folio: isolate to cc.migratepages list.
- Per-PageBuddy: skip (not movable).
- Per-PageMovable: optional callback isolate_movable_page.

REQ-8: Per-zone isolate_freepages:
- Per-pageblock: per-PageBuddy: __isolate_free_page.

REQ-9: Per-pageblock migrate-type:
- MIGRATE_UNMOVABLE / MIGRATE_MOVABLE / MIGRATE_RECLAIMABLE / MIGRATE_PCPTYPES / MIGRATE_HIGHATOMIC / MIGRATE_CMA / MIGRATE_ISOLATE.
- Set/get via set_pageblock_migratetype.

REQ-10: kcompactd daemon:
- Per-node kthread; wakes per-pgdat-imbalanced.
- Background compaction.
- /proc/sys/vm/compact_unevictable_allowed.

REQ-11: try_to_compact_pages (direct):
- Per-page-allocator: high-order alloc fail → try direct compaction.
- gfp passed; mode = MIGRATE_ASYNC | _SYNC_LIGHT | _SYNC depending.
- Per-success: re-attempt allocation.

REQ-12: NUMA balancing:
- Per-task scan of VMAs; mark PTE with PROT_NONE → next access NUMA-fault.
- migrate_misplaced_folio: alloc dst on faulting-CPU's node.

REQ-13: Per-userspace ABI:
- /proc/sys/vm/compaction_proactiveness.
- /proc/sys/vm/extfrag_threshold.
- /proc/<pid>/numa_maps.
- /sys/kernel/mm/numa/balancing.
- migrate_pages syscall (MPOL_*).

## Acceptance Criteria

- [ ] AC-1: migrate_pages list of 4 folios: returns 4 if all moved.
- [ ] AC-2: Per-folio with mapping.migrate_folio: callback invoked.
- [ ] AC-3: migration-PTE marker installed during in-flight; faulters wait.
- [ ] AC-4: After move: VMA PTEs point to dst.
- [ ] AC-5: Per-LRU-folio after migration: isolated then re-LRU'd at dst.
- [ ] AC-6: Compaction: scan zone; migratable pages migrated; free contig increases.
- [ ] AC-7: Direct compaction on high-order alloc fail: contig page found post-compact.
- [ ] AC-8: kcompactd: wakes when zone fragmented; runs compact_zone.
- [ ] AC-9: NUMA balance fault: migrate_misplaced_folio to faulter's node.
- [ ] AC-10: CMA alloc: migrates non-CMA pages out of CMA region.
- [ ] AC-11: hugetlb migrate: unmap_and_move_huge_folio.

## Architecture

Per-migration state:

```
struct CompactControl {
  freepages: ListHead<Folio>,                    // isolated free
  migratepages: ListHead<Folio>,                 // isolated to migrate
  free_pfn: u64,                                 // top-down cursor
  migrate_pfn: u64,                              // bottom-up cursor
  mode: MigrateMode,
  zone: *Zone,
  reason: u8,
  ignore_skip_hint: bool,
  nr_freepages: u64,
  nr_migratepages: u64,
  total_migrate_scanned: u64,
  total_free_scanned: u64,
}

enum MigrateMode {
  MIGRATE_ASYNC,
  MIGRATE_SYNC_LIGHT,
  MIGRATE_SYNC,
  MIGRATE_SYNC_NO_COPY,
}
```

`Migration::migrate_pages(from, get_new_page, mode, reason) -> i32`:
1. nr_succeeded = 0; nr_failed = 0.
2. for folio in from:
   - dst = get_new_page(...).
   - rc = Migration::__migrate_folio(folio, dst, mode).
   - if rc == 0: nr_succeeded++.
   - else: nr_failed++.
3. Return nr_succeeded.

`Migration::__migrate_folio(src, dst, mode) -> i32`:
1. folio_lock(src).
2. /* Setup migration PTEs */
3. if folio_is_anon: try_to_unmap(src).
4. if mapping = src.mapping ∧ mapping.migrate_folio:
   - rc = mapping.migrate_folio(mapping, dst, src, mode).
5. else:
   - rc = Migration::migrate_folio(NULL, dst, src, mode).
6. if rc == 0:
   - Migration::move_to_new_folio(dst, src, mode).
7. folio_unlock(src).
8. Return rc.

`Migration::migrate_folio(mapping, dst, src, mode) -> i32`:
1. folio_copy(dst, src).
2. xa_lock(mapping.i_pages).
3. xas_store(mapping.i_pages, src.index, dst).
4. xa_unlock.
5. Return 0.

`Compact::isolate_migratepages_range(cc, start, end) -> u64`:
1. cur = start.
2. for pfn in start..end:
   - page = pfn_to_page(pfn).
   - if PageBuddy(page): pfn += 1 << order; continue.
   - if PageLRU(page) ∨ PageMovable(page):
     - folio = page_folio(page).
     - if folio_trylock(folio) ∧ isolate_lru_folio(folio, isolate_mode):
       - list_add(&folio.lru, &cc.migratepages).
       - cc.nr_migratepages++.
3. Return scanned-count.

`Compact::isolate_freepages(cc)`:
1. /* Top-down scan from cc.free_pfn */
2. while cc.free_pfn > cc.migrate_pfn ∧ cc.nr_freepages < cc.nr_migratepages:
   - block_pfn = aligned-down(cc.free_pfn, pageblock_nr_pages).
   - per-PageBuddy in block: __isolate_free_page; list_add(&folio.lru, &cc.freepages).
   - cc.nr_freepages++.

`Compact::compact_zone(zone, cc) -> CompactResult`:
1. cc.migrate_pfn = zone.zone_start_pfn; cc.free_pfn = zone_end_pfn(zone).
2. while !compact_finished(zone, cc):
   - Compact::isolate_migratepages_range(cc, ...).
   - if cc.nr_freepages < cc.nr_migratepages: Compact::isolate_freepages(cc).
   - Migration::migrate_pages(&cc.migratepages, compaction_alloc, ..., cc.mode, MR_COMPACTION).
3. Return COMPACT_COMPLETE / COMPACT_PARTIAL_SKIPPED.

`Compact::try_to_compact_pages(gfp_mask, order, alloc_flags, ac) -> Page`:
1. for zone in ac.zonelist:
   - rc = Compact::compact_zone(zone, cc).
   - if rc == COMPACT_COMPLETE: return zone_alloc_pages(zone, order, ...).
2. Return NULL.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `migrate_pte_during_in_flight` | INVARIANT | per-folio in-flight: VMA-PTE has migration-marker. |
| `cursors_dont_overlap` | INVARIANT | cc.migrate_pfn ≤ cc.free_pfn. |
| `nr_migratepages_consistent` | INVARIANT | cc.nr_migratepages == #folios in cc.migratepages. |
| `mode_async_no_sleep` | INVARIANT | per-MIGRATE_ASYNC: no sleep allowed in callbacks. |
| `pageblock_migratetype_consistent` | INVARIANT | per-pageblock: migrate-type consistent across PFNs. |

### Layer 2: TLA+

`mm/migration.tla`:
- Per-folio migrate (lock, unmap, alloc, copy, remap, free) sequence.
- Per-zone compaction cursors meeting.
- Properties:
  - `safety_no_lost_data` — per-migrate: post-state pages have src bytes.
  - `safety_no_concurrent_double_migrate` — per-folio: at most one in-flight migrate.
  - `liveness_compact_eventually_finds_contig` — per-zone with movable pages: compaction yields high-order block.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Migration::__migrate_folio` post: rc==0 ⟹ dst contains src bytes; src freed | `Migration::__migrate_folio` |
| `Compact::isolate_migratepages_range` post: per-folio in cc.migratepages locked + isolated | `Compact::isolate_migratepages_range` |
| `Compact::compact_zone` post: nr_freepages converges; cursors meet | `Compact::compact_zone` |
| `Migration::migrate_pages` post: returned count = succeeded; failed re-list'd | `Migration::migrate_pages` |

### Layer 4: Verus/Creusot functional

`Per-folio migrate atomically + per-zone compaction yields contiguous blocks` semantic equivalence: per-Linux mm migration model (migrate.h doc).

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Migration/compaction reinforcement:

- **Per-folio_lock** — defense against per-concurrent-migrate UAF.
- **Per-migration-PTE marker** — defense against per-fault read-stale src after migrate.
- **Per-rmap_walk during migrate** — defense against per-VMA orphaned mapping.
- **Per-mode determines sleep-allowed** — defense against per-MIGRATE_ASYNC sleep oops.
- **Per-pageblock migrate-type checked** — defense against per-UNMOVABLE wrongful migrate.
- **Per-cursors don't cross** — defense against per-compaction infinite loop.
- **Per-CMA region migrate-out** — defense against per-CMA-page leak preventing alloc.
- **Per-NUMA-fault rate-limit** — defense against per-NUMA-thrash.
- **Per-direct-compaction backoff** — defense against per-alloc-fail livelock.
- **Per-isolate count caps** — defense against per-batch unbounded list.
- **Per-folio_set_movable not removed mid-isolate** — defense against per-isolate inconsistent.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/00-overview (Tier-2)
- Page allocator (covered in `page-allocator.md` Tier-3)
- THP (covered in `thp.md` Tier-3)
- KSM (covered separately if added)
- Hugetlb (covered separately if added)
- NUMA mempolicy (covered separately if added)
- Implementation code
