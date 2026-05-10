# Tier-3: mm/zsmalloc.c — zsmalloc compressed-page slab-style allocator

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/zsmalloc.c (~2271 lines)
  - include/linux/zsmalloc.h
-->

## Summary

zsmalloc is a slab-style allocator optimized for storing **variable-size compressed pages** (compressed via lz4/zstd/lzo) coming from zswap and zram. Per-zspage = N consecutive 4KB pages forming one virtual `zspage` containing M same-class objects (per-class size 32-bytes-step from 32 to ~3KB). Per-class has lists: ZS_EMPTY / ZS_ALMOST_EMPTY / ZS_ALMOST_FULL / ZS_FULL. Per-`zs_malloc(size)` returns u64 handle (encodes class+zspage+obj-idx); per-`zs_free(handle)` releases. Per-handle dereferenced via `zs_map_object` (returns kernel-virtual ptr w/ kmap-protected-2-page-window). Per-zspage migration (compaction) consolidates fragmented zspages. Critical for: zram/zswap memory efficiency.

This Tier-3 covers `zsmalloc.c` (~2271 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct zs_pool` | per-pool state | `ZsPool` |
| `struct size_class` | per-(size, num-pages) class | `SizeClass` |
| `struct zspage` | per-(class, N-pages) collection | `ZsPage` |
| `zs_create_pool()` | per-pool create | `Zs::create_pool` |
| `zs_destroy_pool()` | per-pool destroy | `Zs::destroy_pool` |
| `zs_malloc()` | per-handle alloc | `Zs::malloc` |
| `zs_free()` | per-handle free | `Zs::free` |
| `zs_map_object()` / `zs_unmap_object()` | per-handle map | `Zs::map_object` / `unmap_object` |
| `zspage_alloc()` | per-class alloc-zspage | `Zs::zspage_alloc` |
| `migrate_zspage()` | per-zspage migrate (compact) | `Zs::migrate_zspage` |
| `MAX_PHYSMEM_BITS` | per-handle bit-budget | shared |
| `ZS_OBJ_ALLOCATED` / `ZS_HANDLE_ALLOCATED` | per-bit | shared |
| `ZS_HUGE_OBJECT_SIZE` (~3KB) | per-class max | shared |
| `__GFP_*` interaction | per-alloc gfp-mask | shared |

## Compatibility contract

REQ-1: Per-zs_pool:
- name: pool name (e.g. "zswap", "zram0").
- size_class[]: per-class (size_class).
- migrate_lock: rw-lock for compaction.
- objs_allocated: AtomicLong.
- pages_allocated: AtomicLong.

REQ-2: Per-size_class:
- size: object size (32, 64, 96, ..., max).
- pages_per_zspage: 1, 2, 3, 4 (max 4 contiguous pages).
- objs_per_zspage: pages_per_zspage × PAGE_SIZE / size.
- fullness_list[]: ZS_EMPTY / ALMOST_EMPTY / ALMOST_FULL / FULL.
- isolated_zspages: per-migration isolated.

REQ-3: Per-zspage:
- N contiguous 4KB pages (1..4) virtual-mapped contiguous.
- magic + class + first_obj + ...:
- objs are linked-list internally (free-list).
- inuse: count of allocated objects.
- freeobj: next free idx.

REQ-4: zs_malloc(pool, size, gfp, mode):
- Cap size at ZS_HUGE_OBJECT_SIZE.
- class_idx = get_size_class_index(size).
- class = pool.size_class[class_idx].
- per-pool spinlock_class.
- zspage = find-zspage with free obj OR alloc new.
- obj_idx = pop-from-zspage-freelist.
- handle = (zspage << OBJ_TAG_BITS) | obj_idx.
- zspage.inuse++.
- pool.objs_allocated++.
- Return handle.

REQ-5: zs_free(pool, handle):
- per-handle decode → (zspage, obj_idx).
- spin_lock(class.lock).
- push obj back to zspage freelist.
- zspage.inuse--.
- per-fullness-transition: relink to appropriate fullness_list.
- if ZS_EMPTY: free_zspage(pool, class, zspage).

REQ-6: zs_map_object(pool, handle):
- per-handle → (zspage, obj_idx).
- per-zspage may span 2 pages.
- kmap-window-2-page (zsmalloc_map_window) → contiguous-virtual.
- Return ptr (within kmap-window).

REQ-7: zs_unmap_object:
- kunmap-window.

REQ-8: Per-class fullness state machine:
- ZS_EMPTY: zspage just freed.
- ZS_ALMOST_EMPTY: < 25% obj-allocated.
- ZS_ALMOST_FULL: 25-75%.
- ZS_FULL: 100% allocated.

REQ-9: Per-migration / compaction:
- Identify ALMOST_EMPTY zspages.
- Migrate objs to other zspages (consolidate).
- Free emptied zspage.
- Per-handle remapped via per-handle-stable-pointer.

REQ-10: Per-handle stability:
- handle is u64; encodes pfn + obj_idx.
- Per-migration: handle table (per-pool) maps handle → (zspage, obj_idx) — handle stays constant.

REQ-11: Per-class N-pages-per-zspage:
- Optimize based on objects fit:
  - 1 page: per-256-byte object → 16 obj/zspage, 0 wasted.
  - 2 pages: per-768-byte object → 10 obj/zspage, ~256 wasted.
  - Per-pages_per_zspage minimizes wastage.

## Acceptance Criteria

- [ ] AC-1: zs_create_pool("zswap"): pool with default classes.
- [ ] AC-2: zs_malloc(pool, 256, GFP_KERNEL): handle returned; pool.objs_allocated++.
- [ ] AC-3: zs_map_object(pool, handle): returns kernel-virtual ptr.
- [ ] AC-4: Write to ptr; zs_unmap_object: data persisted.
- [ ] AC-5: zs_free(pool, handle): obj returned to freelist.
- [ ] AC-6: 100% alloc class: zspage in FULL list.
- [ ] AC-7: After free: zspage transitions back through ALMOST_FULL → ALMOST_EMPTY → EMPTY.
- [ ] AC-8: Compaction migrates ALMOST_EMPTY zspages: emptied freed.
- [ ] AC-9: Per-pool destroy: all zspages freed.
- [ ] AC-10: zswap reports per-pool zsmalloc-stats.

## Architecture

Per-pool state:

```
struct ZsPool {
  name: String,
  size_class: [Option<&SizeClass>; ZS_SIZE_CLASSES], // ~256 classes
  zspage_migrate_lock: RwLock,
  objs_allocated: AtomicLong,
  pages_allocated: AtomicLong,
  pool_lock: SpinLock,
  inc_compress: AtomicLong,
  ...
}

struct SizeClass {
  size: u32,
  objs_per_zspage: u32,
  pages_per_zspage: u8,
  index: u8,
  fullness_list: [ListHead<ZsPage>; NR_FULLNESS_GROUPS],
  isolated_zspages: ListHead<ZsPage>,
  lock: SpinLock,
  stats: ZsPoolStats,
}

struct ZsPage {
  list: ListLink,                                 // fullness_list
  magic: u32,                                     // ZSPAGE_MAGIC
  inuse: u32,
  freeobj: u32,                                   // head of free-list
  class: u8,
  fullness: u8,                                   // ZS_*
  isolated: u8,
  first_page: *Page,
  pool: *ZsPool,
}
```

Constants:

```
const ZS_HUGE_OBJECT_SIZE: usize = 3 * 1024 - 64; // ~3KB
const ZS_SIZE_CLASSES: usize = 255;
const NR_FULLNESS_GROUPS: usize = 4;             // EMPTY / ALMOST_EMPTY / ALMOST_FULL / FULL
const ZS_MAX_PAGES_PER_ZSPAGE: usize = 4;
```

`Zs::create_pool(name) -> *ZsPool`:
1. pool = kzalloc(sizeof(ZsPool)).
2. pool.name = name.
3. for size from 32 to ZS_HUGE_OBJECT_SIZE step 16/32/...:
   - class = init_size_class(size, &pool).
   - pool.size_class[idx] = class.
4. atomic_set(&pool.objs_allocated, 0).
5. spin_lock_init(&pool.pool_lock).

`Zs::malloc(pool, size, gfp, ...) -> u64`:
1. class_idx = get_size_class_index(size).
2. class = pool.size_class[class_idx].
3. spin_lock(&class.lock).
4. zspage = pop-from-fullness-list[ZS_ALMOST_FULL] ∨ [ALMOST_EMPTY] ∨ [EMPTY].
5. if !zspage: spin_unlock; zspage = Zs::zspage_alloc(class, gfp, ...); spin_lock.
6. obj_idx = zspage.freeobj.
7. zspage.freeobj = obj-internal-link[zspage.freeobj].
8. zspage.inuse++.
9. fullness_post = compute_fullness(class, zspage).
10. if fullness_post != fullness_pre: relink to new fullness_list.
11. handle = (zspage_to_handle_id(zspage) << OBJ_TAG_BITS) | obj_idx.
12. spin_unlock.
13. atomic_inc(&pool.objs_allocated).
14. Return handle.

`Zs::zspage_alloc(class, gfp, mode) -> *ZsPage`:
1. for i in 0..class.pages_per_zspage:
   - page = alloc_page(gfp).
   - if !page: free pages so far; return NULL.
2. /* Link N pages contiguously */
3. zspage = init_zspage(pages, class).
4. atomic_add(class.pages_per_zspage, &pool.pages_allocated).
5. Return zspage.

`Zs::free(pool, handle)`:
1. (zspage, obj_idx) = decode_handle(handle).
2. class = pool.size_class[zspage.class].
3. spin_lock(&class.lock).
4. /* Push back on freelist */
5. obj-internal-link[obj_idx] = zspage.freeobj.
6. zspage.freeobj = obj_idx.
7. zspage.inuse--.
8. fullness_post = compute_fullness(class, zspage).
9. relink to new fullness_list.
10. if zspage.inuse == 0: free_zspage(pool, class, zspage).
11. spin_unlock.
12. atomic_dec(&pool.objs_allocated).

`Zs::map_object(pool, handle, mm) -> *u8`:
1. (zspage, obj_idx) = decode_handle(handle).
2. class = pool.size_class[zspage.class].
3. obj_offset = obj_idx × class.size.
4. /* Per-cpu map-window (2 pages contiguously kmapped) */
5. kmap_atomic per-cpu zsmalloc_map_window pages → window_ptr.
6. Per-zspage spans 2 pages: copy bytes from spanning region into window.
7. Return window_ptr + obj_offset.

`Zs::unmap_object(pool, handle)`:
1. kunmap_atomic.

`Zs::migrate_zspage(pool, src_zspage, dst_zspage)`:
1. for obj_idx in src_zspage.alloc-bitmap:
   - copy obj-bytes src → dst.
   - update handle-table to point to dst.
2. dst_zspage.inuse += migrated_count.
3. src_zspage.inuse = 0; free.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `obj_idx_lt_objs_per_zspage` | INVARIANT | per-handle: obj_idx < zspage.objs_per_zspage. |
| `fullness_in_set` | INVARIANT | zspage.fullness ∈ {EMPTY, ALMOST_EMPTY, ALMOST_FULL, FULL}. |
| `inuse_le_objs_per_zspage` | INVARIANT | zspage.inuse ≤ class.objs_per_zspage. |
| `pages_per_zspage_le_max` | INVARIANT | class.pages_per_zspage ≤ ZS_MAX_PAGES_PER_ZSPAGE. |
| `magic_validates` | INVARIANT | per-zspage.magic == ZSPAGE_MAGIC. |

### Layer 2: TLA+

`mm/zsmalloc.tla`:
- Per-pool alloc/free + per-zspage fullness-transitions + per-migration.
- Properties:
  - `safety_obj_unique_per_handle` — per-handle decodes to unique (zspage, obj_idx).
  - `safety_no_double_free` — per-zs_free called once per zs_malloc.
  - `liveness_compact_emptys` — per-empty-zspage eventually freed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Zs::malloc` post: handle decodes; objs_allocated++ | `Zs::malloc` |
| `Zs::free` post: obj on freelist; inuse--; fullness updated | `Zs::free` |
| `Zs::map_object` post: ptr within zsmalloc_map_window | `Zs::map_object` |
| `Zs::zspage_alloc` post: N pages allocated; pages_allocated += N | `Zs::zspage_alloc` |
| `Zs::migrate_zspage` post: per-obj copied; src.inuse=0 | `Zs::migrate_zspage` |

### Layer 4: Verus/Creusot functional

`Per-handle compressed-data persisted in per-class zspage; per-fragmentation compaction frees emptied` semantic equivalence: per-zsmalloc spec from `Documentation/mm/zsmalloc.rst`.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

zsmalloc-specific reinforcement:

- **Per-class spinlock** — defense against per-class concurrent-alloc race.
- **Per-zspage magic-validation** — defense against per-handle decode UB.
- **Per-handle obj_idx bounded** — defense against per-handle OOB obj-access.
- **Per-zspage migration with handle-table** — defense against per-handle stale post-migrate.
- **Per-pages_per_zspage cap** — defense against per-class over-large zspage.
- **Per-fullness_list state machine** — defense against per-zspage list-corruption.
- **Per-rw-lock for migration** — defense against per-walk during compact race.
- **Per-pool atomic counters** — defense against per-stat race.
- **Per-zspage pages alloc-fail rollback** — defense against per-partial zspage alloc.
- **Per-zsmalloc_map_window per-cpu** — defense against per-CPU concurrent map race.
- **Per-pool destroy assert all freed** — defense against per-pool memory-leak.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/00-overview (Tier-2)
- zswap (covered in `swap.md` Tier-3 if expanded)
- zram (covered separately)
- Implementation code
