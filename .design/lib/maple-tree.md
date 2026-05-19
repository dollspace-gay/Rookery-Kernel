# Tier-3: lib/maple_tree.c — Maple tree (RCU-safe range-keyed B+tree)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: lib/00-overview.md
upstream-paths:
  - lib/maple_tree.c (~7042 lines)
  - include/linux/maple_tree.h (~936 lines)
  - include/trace/events/maple_tree.h
  - tools/testing/radix-tree/maple.c (userspace test harness)
  - Documentation/core-api/maple_tree.rst
-->

## Summary

The Maple Tree is an **RCU-safe adaptive B+tree keyed on `unsigned long` ranges**, designed by Liam Howlett and Matthew Wilcox to replace the rbtree-of-VMAs in `struct mm_struct` (`mm->mm_mt`) and to serve as a general range-indexed associative container. Each node is 256 bytes and 256-byte aligned, holds either 16 (64-bit) or 32 (32-bit) slots, and stores **pivots** — keys that are inclusive upper bounds on contiguous ranges. Internal nodes (`maple_range_64`, `maple_arange_64`) store pointers to child nodes plus pivots; leaf nodes (`maple_leaf_64`) store user entries plus pivots. `maple_arange_64` additionally tracks per-slot `gap[]` arrays (largest free-range size in that subtree) to allow fast empty-area allocation. Per-tree state lives in `struct maple_tree { ma_lock, ma_flags, ma_root }`. Per-walk state lives in `struct ma_state` (the MA_STATE), which carries `index`/`last` range, current `node`, implied `min`/`max`, status (`ma_start`/`ma_active`/`ma_none`/`ma_pause`/`ma_overflow`/`ma_underflow`/`ma_error`), and a per-walk allocation cache. Per-walk safety is via either internal spinlock (`ma_lock`) or an externally-declared lock for embedded trees, plus optional RCU mode (`MT_FLAGS_USE_RCU`). The simple API (`mtree_load`, `mtree_insert`, `mtree_store`, `mtree_erase`, `mtree_destroy`) covers single-index operations; the advanced API (`mas_walk`, `mas_store`, `mas_erase`, `mas_find`, `mas_next`, `mas_prev`, `mas_preallocate`, `mas_empty_area`) is used by `mm/mmap.c` and `mm/vma.c` to drive bulk VMA operations. Critical for: VMA lookup (`find_vma`), VMA insertion / merging / splitting, anon_vma reverse-map walks, file mmap, fork/exec address-space duplication, and ranged page-cache operations. Replaces rbtree for VMAs (4.x era code) — substantially better cache locality, reduced height, and RCU-safe lookups in fast paths.

This Tier-3 covers `lib/maple_tree.c` (~7042 lines) + `include/linux/maple_tree.h` (~936 lines) — by line count the largest data-structure library in the kernel.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct maple_tree` | per-tree root + lock + flags | `MapleTree` |
| `struct maple_node` | per-node (256-byte) union: range64 / arange64 / leaf / topiary / alloc / copy | `MapleNode` |
| `struct maple_range_64` | per-internal-range-node {parent, pivot[15], slot[16], meta} | `MapleRange64` |
| `struct maple_arange_64` | per-allocation-range-node {parent, pivot[9], slot[10], gap[10], meta} | `MapleARange64` |
| `struct maple_metadata` | per-{end, gap} optimization metadata | `MapleMetadata` |
| `struct maple_topiary` | per-pruned-subtree-list | `MapleTopiary` |
| `struct maple_copy` | per-rebalance-bulk-copy scratch | `MapleCopy` |
| `enum maple_type` | maple_dense / leaf_64 / range_64 / arange_64 / copy | `MapleType` |
| `enum maple_status` | ma_active / ma_start / ma_root / ma_none / ma_pause / ma_overflow / ma_underflow / ma_error | `MaStatus` |
| `enum store_type` | wr_invalid / wr_new_root / wr_store_root / wr_exact_fit / wr_spanning_store / wr_split_store / wr_rebalance / wr_append / wr_node_store / wr_slot_store | `StoreType` |
| `struct ma_state` | per-walk state (tree, index, last, node, min, max, status, depth, offset, ...) | `MaState` |
| `struct ma_wr_state` | per-write state (mas, node, r_min, r_max, pivots, slots, entry, content) | `MaWrState` |
| `MA_STATE(name, mt, first, end)` macro | per-stack-init MA_STATE | `MaState::new` |
| `MA_WR_STATE(name, mas, entry)` macro | per-stack-init ma_wr_state | `MaWrState::new` |
| `MTREE_INIT(name, flags)` macro | per-static-init MapleTree | `MapleTree::INIT` |
| `MTREE_INIT_EXT(name, flags, lock)` macro | per-static-init with external lock | `MapleTree::INIT_EXT` |
| `DEFINE_MTREE(name)` macro | per-flat-flag static init | `MapleTree::define` |
| `mt_init()` / `mt_init_flags()` (inline) | per-runtime init | `MapleTree::init` / `init_flags` |
| `maple_tree_init()` | per-boot kmem_cache_create for maple_node_cache | `MapleTree::boot_init` |
| `mtree_destroy()` / `__mt_destroy()` | per-tree teardown (locked / unlocked) | `MapleTree::destroy` / `destroy_unlocked` |
| `mtree_load(mt, index)` | per-point query | `MapleTree::load` |
| `mtree_store(mt, index, entry, gfp)` | per-point overwrite-store | `MapleTree::store` |
| `mtree_store_range(mt, first, last, entry, gfp)` | per-range overwrite-store | `MapleTree::store_range` |
| `mtree_insert(mt, index, entry, gfp)` | per-point insert (EEXIST if occupied) | `MapleTree::insert` |
| `mtree_insert_range(mt, first, last, entry, gfp)` | per-range insert | `MapleTree::insert_range` |
| `mtree_erase(mt, index)` | per-point delete, return removed entry | `MapleTree::erase` |
| `mtree_alloc_range(mt, *startp, entry, size, min, max, gfp)` | per-allocation-tree bottom-up empty-range alloc | `MapleTree::alloc_range` |
| `mtree_alloc_rrange(...)` | per-allocation-tree top-down (reverse) | `MapleTree::alloc_rrange` |
| `mtree_alloc_cyclic(mt, *startp, entry, lo, hi, *next, gfp)` | per-cyclic IDA-style alloc | `MapleTree::alloc_cyclic` |
| `mtree_dup(mt, new, gfp)` / `__mt_dup` | per-tree deep-copy (used by fork) | `MapleTree::dup` / `dup_unlocked` |
| `mtree_empty(mt)` (inline) | per-tree emptiness check | `MapleTree::is_empty` |
| `mas_walk(mas)` | per-MA_STATE walk to mas->index | `MaState::walk` |
| `mas_store(mas, entry)` | per-MA_STATE store at [index..last] | `MaState::store` |
| `mas_store_gfp(mas, entry, gfp)` | per-MA_STATE store with allocation | `MaState::store_gfp` |
| `mas_store_prealloc(mas, entry)` | per-MA_STATE store using preallocated nodes | `MaState::store_prealloc` |
| `mas_erase(mas)` | per-MA_STATE erase current entry | `MaState::erase` |
| `mas_find(mas, max)` | per-MA_STATE forward find next entry | `MaState::find` |
| `mas_find_range(mas, max)` | per-MA_STATE forward find next range | `MaState::find_range` |
| `mas_find_rev(mas, min)` | per-MA_STATE reverse find | `MaState::find_rev` |
| `mas_find_range_rev(mas, min)` | per-MA_STATE reverse find range | `MaState::find_range_rev` |
| `mas_next(mas, max)` | per-MA_STATE step to next slot | `MaState::next` |
| `mas_next_range(mas, max)` | per-MA_STATE step to next range | `MaState::next_range` |
| `mas_prev(mas, min)` | per-MA_STATE step to previous slot | `MaState::prev` |
| `mas_prev_range(mas, min)` | per-MA_STATE step to previous range | `MaState::prev_range` |
| `mas_empty_area(mas, min, max, size)` | per-allocation tree: find empty-range of `size` bottom-up | `MaState::empty_area` |
| `mas_empty_area_rev(mas, min, max, size)` | per-allocation tree: find empty-range top-down | `MaState::empty_area_rev` |
| `mas_preallocate(mas, entry, gfp)` | per-preallocate worst-case nodes | `MaState::preallocate` |
| `mas_alloc_cyclic(mas, ...)` | per-cyclic alloc via MA_STATE | `MaState::alloc_cyclic` |
| `mas_nomem(mas, gfp)` | per-OOM handling: drop lock, allocate, retry | `MaState::nomem` |
| `mas_pause(mas)` | per-mark stale-and-restart | `MaState::pause` |
| `mas_destroy(mas)` | per-walk teardown / free reservation | `MaState::destroy` |
| `mas_init(mas, tree, addr)` (inline) | per-walk reset | `MaState::init` |
| `mas_reset(mas)` (inline) | per-walk status reset to ma_start | `MaState::reset` |
| `mas_set(mas, idx)` / `mas_set_range(mas, start, last)` | per-walk reposition | `MaState::set` / `set_range` |
| `mas_for_each(mas, entry, max)` macro | per-walk iterator | `MaState::for_each` |
| `mas_for_each_rev(mas, entry, min)` macro | per-walk reverse iterator | `MaState::for_each_rev` |
| `mt_find(mt, *idx, max)` / `mt_find_after` / `mt_next` / `mt_prev` | per-simple-API range walkers | `MapleTree::find` / `find_after` / `next` / `prev` |
| `mt_for_each(tree, entry, idx, max)` macro | per-simple-API iterator | `MapleTree::for_each` |
| `mt_set_in_rcu` / `mt_clear_in_rcu` / `mt_in_rcu` (inline) | per-RCU-mode switch | `MapleTree::set_in_rcu` / `clear_in_rcu` / `in_rcu` |
| `mt_height(mt)` (inline) | per-tree height (from ma_flags) | `MapleTree::height` |
| `mt_lock_is_held(mt)` / `mt_write_lock_is_held(mt)` | per-LOCKDEP assertion | shared lockdep-side |
| `MT_FLAGS_ALLOC_RANGE` / `_USE_RCU` / `_HEIGHT_OFFSET` / `_HEIGHT_MASK` / `_LOCK_MASK` / `_LOCK_IRQ` / `_LOCK_BH` / `_LOCK_EXTERN` / `_ALLOC_WRAPPED` | per-flag bits | `MapleTreeFlags` |
| `MAPLE_NODE_SLOTS` (31 on 64-bit) | per-node slot count | `LayoutConst` |
| `MAPLE_RANGE64_SLOTS` (16 on 64-bit) | per-range64 slot count | `LayoutConst` |
| `MAPLE_ARANGE64_SLOTS` (10 on 64-bit) | per-arange64 slot count | `LayoutConst` |
| `MAPLE_HEIGHT_MAX` (31) | per-tree max height | `LayoutConst` |
| `MAPLE_RESERVED_RANGE` (4096) | per-reserved-low-range for internal use | `LayoutConst` |
| `MA_ERROR(err)` macro | per-encoded-error pseudo-node | `MaState::encode_error` |
| `mas_is_err(mas)` / `mas_is_active(mas)` (inline) | per-state predicates | `MaState::is_err` / `is_active` |
| `maple_node_cache` (static kmem_cache) | per-node SLAB cache | shared slab-side |

## Compatibility contract

REQ-1: Node layout (struct maple_node, 256 bytes, 256-byte aligned):
- Union over: { parent + slot[MAPLE_NODE_SLOTS] }, { pad + rcu_head + piv_parent + parent_slot + type + slot_len + ma_flags } (used post-removal), `maple_range_64 mr64`, `maple_arange_64 ma64`, `maple_copy cp`.
- Allocated from `maple_node_cache` (kmem_cache_create with 256-byte alignment).
- Bottom 8 bits of node pointer used for type-tagging via `enum maple_type` encoded in bits 3-6; root tag in bit 0; bit 2 reserved.

REQ-2: maple_range_64 layout (internal range node):
- `struct maple_pnode *parent`.
- `unsigned long pivot[MAPLE_RANGE64_SLOTS - 1]` (15 pivots on 64-bit).
- Union: `void __rcu *slot[MAPLE_RANGE64_SLOTS]` (16 slots) OR `void __rcu *pad[15] + struct maple_metadata meta` (when meta is encoded into the last slot).
- Pivots are inclusive upper bounds of ranges; slot[i] holds entry/child for range `(pivot[i-1]+1, pivot[i])`.

REQ-3: maple_arange_64 layout (allocation-range node — tracks free gaps):
- `struct maple_pnode *parent`.
- `unsigned long pivot[MAPLE_ARANGE64_SLOTS - 1]` (9 pivots on 64-bit).
- `void __rcu *slot[MAPLE_ARANGE64_SLOTS]` (10 slots).
- `unsigned long gap[MAPLE_ARANGE64_SLOTS]` (10 gaps — largest free range in each subtree).
- `struct maple_metadata meta`.

REQ-4: struct maple_tree:
- Union: `spinlock_t ma_lock` OR `struct lockdep_map *ma_external_lock` (when CONFIG_LOCKDEP and using MT_FLAGS_LOCK_EXTERN).
- `unsigned int ma_flags` — packs ALLOC_RANGE bit, USE_RCU bit, height (bits 2-6 via HEIGHT_MASK), LOCK_MASK (bits 8-9), ALLOC_WRAPPED.
- `void __rcu *ma_root` — single root pointer; may directly hold a small entry (singleton optimization), MA_ERROR-encoded errno, or a tagged maple_enode.

REQ-5: struct ma_state (MA_STATE) fields:
- tree: per-tree.
- index, last: per-range being operated on (inclusive).
- node: current maple_enode (NULL on start; tagged pointer with type+offset bits).
- min, max: per-implied bounds of `node`.
- sheaf: per-allocated-node-cache for this operation (slab_sheaf).
- alloc: per-single fast-path node allocation.
- node_request: per-needed-allocation count.
- status: enum maple_status (ma_active / ma_start / ma_root / ma_none / ma_pause / ma_overflow / ma_underflow / ma_error).
- depth: tree-descent depth.
- offset: per-slot/pivot of interest.
- mas_flags: per-walk flags (e.g. MA_STATE_PREALLOC).
- end: per-end-of-node offset.
- store_type: per-write enum (selected during mas_wr_*).

REQ-6: MA_STATE(name, mt, first, end) initializer:
- `.tree = mt, .index = first, .last = end, .node = NULL, .status = ma_start, .min = 0, .max = ULONG_MAX, .sheaf = NULL, .alloc = NULL, .node_request = 0, .mas_flags = 0, .store_type = wr_invalid`.

REQ-7: Locking modes (MT_FLAGS_LOCK_MASK):
- `MT_FLAGS_LOCK_IRQ` (0x100) — caller takes spin_lock_irq on ma_lock.
- `MT_FLAGS_LOCK_BH` (0x200) — caller takes spin_lock_bh on ma_lock.
- `MT_FLAGS_LOCK_EXTERN` (0x300) — caller uses external lock (e.g. mmap_lock for mm->mm_mt), tree's ma_lock unused.
- LOCK == 0 — default: spin_lock on ma_lock.
- All mutations require the chosen lock held in write mode; lookups in RCU mode may proceed without the lock.

REQ-8: RCU mode (MT_FLAGS_USE_RCU):
- Set via `mt_set_in_rcu`; cleared via `mt_clear_in_rcu` (both require lock for transition).
- Reads (`mtree_load`, `mt_find`, `mas_walk` if not in write mode) can run under rcu_read_lock.
- Writes always require the write lock.
- Removed nodes are freed via `call_rcu` (rcu_head reused from slot[0..1] post-removal).
- `node->parent` self-pointer indicates a removed node (readers re-check).

REQ-9: `mtree_load(mt, index)`:
- /* RCU-safe single-index lookup */
- rcu_read_lock if MT_FLAGS_USE_RCU.
- Walk root → leaf following pivots; on retry-required (parent-self-pointer): restart from root.
- Return entry at `index` or NULL.
- rcu_read_unlock.

REQ-10: `mtree_store(mt, index, entry, gfp)`:
- /* Single-index overwrite */
- MA_STATE(mas, mt, index, index).
- mtree_lock(mt) (or appropriate variant).
- mas_store_gfp(&mas, entry, gfp).
- mtree_unlock(mt).
- Return 0 or -ENOMEM / -EEXIST as appropriate.

REQ-11: `mtree_store_range(mt, first, last, entry, gfp)`:
- /* Range overwrite [first..last] */
- MA_STATE(mas, mt, first, last).
- mtree_lock(mt).
- mas_store_gfp(&mas, entry, gfp).
- mtree_unlock(mt).

REQ-12: `mtree_insert(mt, index, entry, gfp)`:
- /* Insert only if currently empty at index; -EEXIST else */
- MA_STATE(mas, mt, index, index).
- mtree_lock(mt).
- Walk; if existing entry: ret = -EEXIST; else: store.
- mtree_unlock(mt).

REQ-13: `mtree_erase(mt, index)`:
- /* Remove and return entry at index */
- MA_STATE(mas, mt, index, index).
- mtree_lock(mt).
- ret = mas_erase(&mas).
- mtree_unlock(mt).
- Return ret (may be NULL if no entry).

REQ-14: `mtree_alloc_range(mt, *startp, entry, size, min, max, gfp)`:
- /* MT_FLAGS_ALLOC_RANGE tree required; find an empty range [start, start+size-1] in [min, max], bottom-up */
- MA_STATE(mas, mt, min, max).
- mtree_lock(mt).
- ret = mas_empty_area(&mas, min, max, size).
- If ret == 0: mas_store_gfp(&mas, entry, gfp); *startp = mas.index.
- mtree_unlock(mt).

REQ-15: `mtree_alloc_rrange(mt, *startp, entry, size, min, max, gfp)`:
- Mirror of alloc_range using mas_empty_area_rev (top-down).

REQ-16: `mtree_alloc_cyclic(mt, *startp, entry, lo, hi, *next, gfp)`:
- /* IDA-style: alloc from *next upward, wrapping to lo when exhausted; track wrap via MT_FLAGS_ALLOC_WRAPPED */
- MA_STATE.
- mtree_lock.
- mas_alloc_cyclic.
- mtree_unlock.

REQ-17: `mas_walk(mas) -> *void`:
- /* Walk to mas->index */
- If status == ma_start: descend from root.
- Else: refine from current node based on (offset, pivots).
- Return entry or NULL.
- Post: mas.status ∈ {ma_active, ma_root, ma_none, ma_overflow, ma_underflow, ma_error}.

REQ-18: `mas_store(mas, entry) -> *void`:
- /* Store at [mas.index..mas.last] using preallocated nodes (assumes mas_preallocate called) */
- Determine `store_type` via mas_wr_* state machine.
- Perform the per-store_type path: wr_new_root / wr_store_root / wr_exact_fit / wr_spanning_store / wr_split_store / wr_rebalance / wr_append / wr_node_store / wr_slot_store.
- Return previous entry (overwritten content).

REQ-19: `mas_store_gfp(mas, entry, gfp) -> int`:
- Loop:
  - mas_preallocate(mas, entry, gfp).
  - if mas_is_err(mas): return PTR_ERR.
  - mas_store(mas, entry).
  - if not mas_nomem(mas, gfp): break.
- Return 0 or -ENOMEM.

REQ-20: `mas_erase(mas) -> *void`:
- Walk to mas.index.
- If entry found: mas_store(mas, NULL); return entry.
- Else: return NULL.

REQ-21: `mas_find(mas, max) -> *void`:
- /* Find next non-NULL entry from mas.last+1 up to max */
- Walk forward; update mas.index, mas.last to entry's range.
- Return entry or NULL on max overrun.

REQ-22: `mas_find_range(mas, max) -> *void`:
- Like mas_find but mas.index/mas.last reflect the full range slot.

REQ-23: `mas_next(mas, max) -> *void`:
- /* Advance to next slot regardless of NULL */
- May return NULL entry for "hole" range.

REQ-24: `mas_prev(mas, min) -> *void`:
- /* Step backward to previous slot */

REQ-25: `mas_empty_area(mas, min, max, size) -> int`:
- /* MT_FLAGS_ALLOC_RANGE required */
- Walk using arange64 gap[] metadata to locate a [a, a+size-1] within [min, max] where a+size-1 <= max.
- On success: mas.index = a; mas.last = a+size-1; status ma_active; return 0.
- Return -EBUSY if no fit.

REQ-26: `mas_preallocate(mas, entry, gfp) -> int`:
- /* Reserve worst-case nodes for a store */
- Compute upper bound on nodes needed (depth × spillover factor).
- Allocate via kmem_cache_alloc from maple_node_cache into mas.sheaf / mas.alloc.
- Return 0 or -ENOMEM.

REQ-27: `mas_nomem(mas, gfp) -> bool`:
- /* OOM recovery: if mas in error-ENOMEM state and gfp allows blocking, drop lock + alloc + retry */
- If mas.status == ma_error and error == -ENOMEM and !(gfp & __GFP_NOWARN-only fast):
  - Drop external lock if any.
  - Allocate.
  - Re-take lock.
  - mas_reset(mas).
  - return true (retry).
- Else: return false.

REQ-28: `mas_pause(mas)`:
- /* Mark state stale so next op restarts from root */
- mas.status = ma_pause.

REQ-29: `mas_destroy(mas)`:
- /* Free unused reservation */
- If mas.alloc: kmem_cache_free.
- If mas.sheaf: per-sheaf free.

REQ-30: `mtree_destroy(mt)` / `__mt_destroy(mt)`:
- /* Locked / unlocked teardown */
- Walk all internal+leaf nodes; kmem_cache_free each.
- ma_root = NULL.
- ma_flags &= keep-bits.

REQ-31: `mtree_dup(mt, new, gfp)` / `__mt_dup`:
- /* Copy a tree (used by dup_mmap during fork to clone mm->mm_mt) */
- Deep-traverse `mt`; allocate parallel nodes; copy slots+pivots+gaps.
- Return 0 or -ENOMEM.

REQ-32: Singleton-root optimization:
- A single entry at index 0 (with bottom two bits not `10`) may be stored directly in `mt->ma_root`.
- `MAPLE_RESERVED_RANGE` (4096): values 2,6,10,...,4094 (bottom two bits `10`, value < 4096) reserved for internal encoding (errno-shifted, etc.).
- `mas_is_err(mas)` checks the `0b10` low-bits sentinel + the encoded value.

REQ-33: Per-flag accessors:
- `mt_height(mt) = (ma_flags & MT_FLAGS_HEIGHT_MASK) >> MT_FLAGS_HEIGHT_OFFSET`.
- `mt_in_rcu(mt) = ma_flags & MT_FLAGS_USE_RCU` (false if CONFIG_MAPLE_RCU_DISABLED).
- `mt_external_lock(mt) = (ma_flags & MT_FLAGS_LOCK_MASK) == MT_FLAGS_LOCK_EXTERN`.

REQ-34: `maple_tree_init()` (per-boot):
- `maple_node_cache = kmem_cache_create("maple_node", sizeof(struct maple_node), sizeof(struct maple_node), SLAB_PANIC, NULL)` — 256-byte alignment via size==align.

REQ-35: Iterator macros:
- `mas_for_each(mas, entry, max)`: while (entry = mas_find(mas, max)) != NULL.
- `mas_for_each_rev(mas, entry, min)`: while (entry = mas_find_rev(mas, min)) != NULL.
- `mt_for_each(tree, entry, idx, max)`: entry = mt_find(tree, &idx, max); while entry; entry = mt_find_after(tree, &idx, max).

REQ-36: Per-LOCKDEP assertion macros:
- `mt_lock_is_held(mt) = !ext_lock || lock_is_held(ext_lock)`.
- `mt_write_lock_is_held(mt) = !ext_lock || lock_is_held_type(ext_lock, 0 /* write */)`.
- `mas_lock(mas)` / `mas_unlock(mas)` = `spin_lock(&mas->tree->ma_lock)` / `spin_unlock(...)`.
- `mt_set_external_lock(mt, lock)` records the lockdep map.

REQ-37: Per-consumer set (canonical):
- `mm->mm_mt` (struct mm_struct, MT_FLAGS_LOCK_EXTERN with mmap_lock) — VMA range storage; primary consumer.
- `dup_mmap()` (kernel/fork.c) — uses `mtree_dup`/`__mt_dup` to clone VMAs on fork.
- `find_vma()` / `find_vma_intersection()` / `find_vma_prev()` — use `mas_walk`/`mas_find`/`mas_prev`.
- `vma_iter_*` family (mm/vma_iter.h) — thin wrappers over MA_STATE.
- `vm_unmapped_area()` — uses `mas_empty_area`/`_rev` to find gaps for mmap.
- `mmap_region()` / `vma_merge()` / `__split_vma()` — use `mas_store_prealloc`.
- IDR replacement candidate (lib/idr.c → maple-tree-backed in progress upstream).

## Acceptance Criteria

- [ ] AC-1: `mtree_load` returns the entry stored at `index`, or NULL if no range covers `index`.
- [ ] AC-2: `mtree_store(mt, i, e, gfp)` then `mtree_load(mt, i)` returns `e`.
- [ ] AC-3: `mtree_store_range(mt, a, b, e, gfp)` then `mtree_load(mt, x)` returns `e` for all a ≤ x ≤ b.
- [ ] AC-4: `mtree_insert` returns -EEXIST when slot non-NULL.
- [ ] AC-5: `mtree_erase` returns the previously-stored entry and leaves NULL at the index.
- [ ] AC-6: `mas_find(&mas, max)` enumerates non-NULL entries in [mas.index..max] in ascending order.
- [ ] AC-7: `mas_find_rev(&mas, min)` enumerates in descending order.
- [ ] AC-8: `mas_empty_area(&mas, min, max, size)` returns 0 with mas.index..mas.last spanning `size` consecutive free indices in [min..max] when one exists, else -EBUSY.
- [ ] AC-9: `mas_nomem` recovers from ENOMEM under blocking gfp by dropping lock + allocating + retrying once.
- [ ] AC-10: `mt_in_rcu(mt)` reads do not require ma_lock; `mas_walk` in RCU mode under `rcu_read_lock` returns a consistent snapshot.
- [ ] AC-11: `mtree_dup(src, dst, gfp)` produces a tree where for every index, `mtree_load(dst, i) == mtree_load(src, i)`.
- [ ] AC-12: `mtree_destroy` frees every internal+leaf node; subsequent `mtree_empty(mt)` returns true.
- [ ] AC-13: Concurrent reader under RCU mode never observes a partially-mutated node (parent-self-pointer check on stale-load).
- [ ] AC-14: `MA_STATE` initializer produces status ma_start, min=0, max=ULONG_MAX, node=NULL.
- [ ] AC-15: Storing a range that splits an existing entry produces correctly-bounded slots in the leaf (no overlap, no gap).

## Architecture

```
struct MapleTree {
    lock: SpinLock,              // or external lockdep map under LOCK_EXTERN
    flags: u32,                  // ALLOC_RANGE | USE_RCU | height | LOCK_* | ALLOC_WRAPPED
    root: AtomicPtr<MapleNode>,  // RCU-published; may hold tagged singleton entry
}

struct MapleNode {                 // 256 bytes, 256-byte aligned (kmem_cache)
    // union of these reps:
    Internal {
        parent: PNodePtr,
        slots: [RcuPtr<MapleNode>; MAPLE_NODE_SLOTS], // 31 on 64-bit
    },
    PostRemoval { rcu: RcuHead, type: MapleType, slot_len: u8, ma_flags: u32, ... },
    Range64 { parent: PNodePtr, pivot: [u64; 15], slot: [RcuPtr<MapleNode>; 16], meta: Metadata },
    ARange64 { parent: PNodePtr, pivot: [u64; 9], slot: [RcuPtr<MapleNode>; 10], gap: [u64; 10], meta: Metadata },
    Copy(MapleCopy),
}

enum MaStatus { Active, Start, Root, None, Pause, Overflow, Underflow, Error }
enum StoreType { Invalid, NewRoot, StoreRoot, ExactFit, SpanningStore, SplitStore, Rebalance, Append, NodeStore, SlotStore }

struct MaState {
    tree: *MapleTree,
    index: u64,
    last: u64,
    node: *MapleEnode,             // tagged: low bits encode type+offset
    min: u64,
    max: u64,
    sheaf: *SlabSheaf,
    alloc: *MapleNode,
    node_request: u32,
    status: MaStatus,
    depth: u8,
    offset: u8,
    mas_flags: u8,
    end: u8,
    store_type: StoreType,
}

struct MaWrState {
    mas: *MaState,
    node: *MapleNode,
    r_min: u64,
    r_max: u64,
    node_type: MapleType,
    offset_end: u8,
    pivots: *u64,
    end_piv: u64,
    slots: *RcuPtr<MapleNode>,
    entry: *void,
    content: *void,
    vacant_height: u8,
    sufficient_height: u8,
}
```

`MapleTree::init_flags(&mut self, flags)`:
1. self.flags = flags.
2. if !mt_external_lock(self): SpinLock::init(&mut self.lock).
3. rcu_assign_pointer(self.root, NULL).

`MapleTree::load(&self, index) -> *void`:
1. rcu_read_lock if mt_in_rcu(self).
2. MA_STATE(mas, self, index, index).
3. loop:
   - entry = mas_walk(&mas).
   - if !mas_is_err(&mas): break.
   - if !mas_nomem(&mas, GFP_KERNEL): return PTR_ERR.
4. rcu_read_unlock if mt_in_rcu(self).
5. return entry.

`MapleTree::store(&self, index, entry, gfp) -> i32`:
1. MA_STATE(mas, self, index, index).
2. mtree_lock(self).
3. ret = mas_store_gfp(&mas, entry, gfp).
4. mtree_unlock(self).
5. return ret.

`MapleTree::store_range(&self, first, last, entry, gfp) -> i32`:
1. MA_STATE(mas, self, first, last).
2. mtree_lock; ret = mas_store_gfp; mtree_unlock.

`MapleTree::insert_range(&self, first, last, entry, gfp) -> i32`:
1. MA_STATE(mas, self, first, last).
2. mtree_lock.
3. ret = mas_insert(&mas, entry) /* -EEXIST if any slot in [first..last] is non-NULL */.
4. mtree_unlock.

`MapleTree::erase(&self, index) -> *void`:
1. MA_STATE(mas, self, index, index).
2. mtree_lock.
3. ret = mas_erase(&mas).
4. mtree_unlock.
5. return ret.

`MapleTree::alloc_range(&self, startp, entry, size, min, max, gfp) -> i32`:
1. MA_STATE(mas, self, min, max).
2. mtree_lock.
3. ret = mas_empty_area(&mas, min, max, size).
4. if ret == 0: mas_store_gfp(&mas, entry, gfp); *startp = mas.index.
5. mtree_unlock.

`MapleTree::destroy(&self)`:
1. mtree_lock.
2. mt_destroy_walk(self.root, self) — recursive kmem_cache_free.
3. rcu_assign_pointer(self.root, NULL).
4. mtree_unlock.

`MaState::walk(&mut self) -> *void`:
1. If status == ma_start: descend from self.tree.root.
2. While not leaf:
   - Pick child via pivots[].
   - Update min, max, node.
3. At leaf: locate offset by pivot[]; return slot[offset].

`MaState::store(&mut self, entry) -> *void`:
1. MA_WR_STATE wr_mas with entry.
2. Run write-state-machine to pick store_type.
3. Per store_type:
   - SlotStore: in-place slot update.
   - NodeStore: rebuild leaf node.
   - SplitStore: split leaf, propagate up.
   - Rebalance: pull from siblings.
   - SpanningStore: multi-node store across siblings.
   - Append: append to end.
   - NewRoot / StoreRoot: special-case root.
4. Return prior entry.

`MaState::store_gfp(&mut self, entry, gfp) -> i32`:
1. loop:
   - mas_preallocate(self, entry, gfp).
   - if mas_is_err(self): return -ENOMEM.
   - mas_store(self, entry).
   - if !mas_nomem(self, gfp): break.
2. return 0.

`MaState::erase(&mut self) -> *void`:
1. walk = mas_walk(self).
2. if walk: mas_store(self, NULL); return walk.
3. return NULL.

`MaState::find(&mut self, max) -> *void`:
1. loop:
   - if self.last >= max: return NULL.
   - advance offset; potentially climb+descend across boundary.
   - if slot non-NULL: update index/last to slot range; return slot.

`MaState::empty_area(&mut self, min, max, size) -> i32`:
1. Requires ALLOC_RANGE tree.
2. Walk arange64 nodes consulting gap[]; descend into subtree whose largest_gap ≥ size and fits in [min..max].
3. On leaf: scan for first NULL run of length `size`.
4. mas.index = a; mas.last = a+size-1; return 0.
5. Return -EBUSY if no fit.

`MaState::preallocate(&mut self, entry, gfp) -> i32`:
1. Compute worst-case nodes per the store-type analysis.
2. Allocate into self.sheaf / self.alloc.
3. self.mas_flags |= MA_STATE_PREALLOC.
4. Return 0 / -ENOMEM.

`MaState::nomem(&mut self, gfp) -> bool`:
1. If self.status != ma_error or err != -ENOMEM: return false.
2. If gfp does not permit blocking + retry: return false.
3. Drop external lock if any; allocate; re-take; mas_reset(self); return true.

`MaState::pause(&mut self)`: self.status = ma_pause.

`MaState::destroy(&mut self)`:
1. If self.alloc: kmem_cache_free(maple_node_cache, self.alloc).
2. If self.sheaf: slab_sheaf_release(self.sheaf).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `node_size_256_bytes` | INVARIANT | per-MapleNode: size_of == 256 on both 32-bit and 64-bit. |
| `pivot_monotonic` | INVARIANT | per-node: pivot[0] < pivot[1] < ... < pivot[end-1] (strict ascending, last <= node.max). |
| `slot_range_contiguous` | INVARIANT | per-node: slot[i] owns range (pivot[i-1]+1 .. pivot[i]); no gap, no overlap. |
| `arange_gap_correct` | INVARIANT | per-arange64-node: gap[i] == max contiguous NULL run in slot[i]'s subtree. |
| `height_bounded` | INVARIANT | per-tree: mt_height(self) <= MAPLE_HEIGHT_MAX (31). |
| `rcu_removed_self_parent` | INVARIANT | per-removed-node: node.parent == self (signals readers to retry). |
| `ma_root_singleton_safe_bits` | INVARIANT | per-root-stored entry: bottom two bits != `10` (else stored in node). |
| `mas_walk_terminates` | INVARIANT | per-mas_walk: at most MAPLE_HEIGHT_MAX descend steps. |
| `mas_nomem_drops_lock_only_if_external` | INVARIANT | per-mas_nomem: ma_lock dropped only when external; internal lock retained. |
| `store_gfp_loop_progress` | INVARIANT | per-mas_store_gfp: mas_nomem returns false in finite retries (no infinite loop). |
| `preallocate_worst_case_bound` | INVARIANT | per-mas_preallocate: allocated nodes <= 2*height+constant. |

### Layer 2: TLA+

`lib/maple-tree.tla`:
- Per-N-CPU concurrent model: M readers, K writers, write-lock held during all mutations; readers use RCU mode.
- Actions: `Load(c, idx)`, `Store(c, idx, e)`, `StoreRange(c, a, b, e)`, `Erase(c, idx)`, `Find(c, max)`, `EmptyArea(c, min, max, size)`, `Destroy(c)`, `Dup(src, dst)`.
- Properties:
  - `safety_no_overlap` — per-leaf-state: no two slots' (pivot_lo, pivot_hi) ranges overlap.
  - `safety_no_gap` — per-leaf-state: union of all slot ranges == [node.min..node.max].
  - `safety_rcu_reader_sees_consistent` — per-reader-under-rcu_read_lock: observed entry corresponds to some pre-CAS-or-post-CAS atomic version of the tree.
  - `safety_dup_isomorphism` — per-mtree_dup: for all indices, src.load(i) == dst.load(i).
  - `safety_arange_gap_invariant` — per-mutate: gap[] arrays stay consistent.
  - `liveness_store_gfp_terminates` — per-mas_store_gfp: under bounded ENOMEM, terminates within finite retries.
  - `liveness_empty_area_finds_or_fails` — per-mas_empty_area: returns 0 with valid range or -EBUSY within bounded steps.
  - `liveness_destroy_frees_all` — per-mtree_destroy: post-state ma_root == NULL.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MapleTree::init_flags` post: flags == arg, root NULL, lock unlocked (or external configured) | `MapleTree::init_flags` |
| `MapleTree::load` post: returns entry covering index, or NULL | `MapleTree::load` |
| `MapleTree::store` post: load(index) == entry; load(other) unchanged | `MapleTree::store` |
| `MapleTree::store_range` post: ∀x ∈ [first..last]: load(x) == entry | `MapleTree::store_range` |
| `MapleTree::insert` post: returns -EEXIST iff prior slot non-NULL; else as store | `MapleTree::insert` |
| `MapleTree::erase` post: load(index) == NULL; returns prior entry | `MapleTree::erase` |
| `MapleTree::alloc_range` post: *startp ∈ [min..max - size + 1] ∧ load is_NULL prior, == entry after | `MapleTree::alloc_range` |
| `MapleTree::dup` post: ∀i: src.load(i) == dst.load(i) | `MapleTree::dup` |
| `MapleTree::destroy` post: ma_root == NULL; all nodes freed | `MapleTree::destroy` |
| `MaState::walk` post: status ∈ {active, root, none, overflow, underflow, error}; node consistent with index | `MaState::walk` |
| `MaState::store` post: tree leaf at [index..last] holds entry; surrounding slots split as needed | `MaState::store` |
| `MaState::empty_area` post: ret==0 ⟹ mas.index..mas.last spans `size` free indices in [min..max] | `MaState::empty_area` |
| `MaState::preallocate` post: mas.alloc and/or mas.sheaf have sufficient capacity for the next store | `MaState::preallocate` |
| `MaState::nomem` post: ret==true ⟹ status reset to ma_start; ret==false ⟹ status unchanged | `MaState::nomem` |

### Layer 4: Verus/Creusot functional

`Per-MapleTree API trace` semantic equivalence: per-Documentation/core-api/maple_tree.rst. The Maple Tree must be observationally indistinguishable from an abstract `Map<Range<u64>, *void>` with range-partitioning semantics (every index maps to exactly one entry; range stores split prior ranges; range erases remove without leaving gaps in node coverage).

## Hardening

(Inherits row-1 features from `lib/00-overview.md` § Hardening.)

maple-tree reinforcement:

- **Per-256-byte node alignment** — defense against per-tag-bit-collision (the bottom 8 bits of node pointers carry type + slot info; misalignment would corrupt this encoding).
- **Per-node-self-parent sentinel for removed nodes** — defense against per-RCU-reader-following-dead-pointer; readers re-verify parent before trusting slot reads.
- **Per-MA_ERROR encoding (low bits `0b10`, shifted errno)** — defense against per-NULL-vs-error confusion; reserved value range MAPLE_RESERVED_RANGE (< 4096 with `10` bits) cannot be stored by users.
- **Per-`BUILD_BUG_ON`-style size checks for MAPLE_*_SLOTS** — defense against per-arch-layout-drift (32-bit vs 64-bit slot counts diverge).
- **Per-LOCKDEP integration via `mt_lock_is_held`/`mt_write_lock_is_held`** — defense against per-unlocked-mutation; every mas_store path asserts the write lock is held (external or internal).
- **Per-MT_FLAGS_LOCK_EXTERN for mm->mm_mt** — defense against per-double-locking with mmap_lock; the tree's ma_lock is unused and the caller's mmap_write_lock provides exclusion.
- **Per-RCU mode read-side without lock** — defense against per-find_vma-scalability bottleneck; readers proceed under rcu_read_lock with no spinlock.
- **Per-call_rcu for removed nodes** — defense against per-UAF on concurrent readers; freeing deferred until grace period.
- **Per-preallocation strategy** — defense against per-mid-mutation-OOM corruption; `mas_preallocate` reserves worst-case nodes before any in-place change.
- **Per-`mas_nomem` retry under blocking gfp** — defense against per-spurious-ENOMEM under transient memory pressure.
- **Per-pivot monotonicity invariant** — defense against per-malformed-node yielding out-of-order or overlapping ranges; CONFIG_DEBUG_MAPLE_TREE adds `mt_validate` checks.
- **Per-`maple_node_cache` SLAB_PANIC** — defense against per-boot-OOM yielding a broken maple infrastructure (boot fails loudly rather than silently).
- **Per-`mas_pause` for safe long walks** — defense against per-blocking-walker-holding-references-to-stale-nodes; pause forces a restart.
- **Per-`__mt_dup` deep-copy under source lock** — defense against per-fork-time tearing of source tree while target tree is built.
- **Per-`mt_set_external_lock` lockdep linkage** — defense against per-cross-tree lock-class confusion; every embedded tree declares its lockdep parent.

## Grsecurity/PaX-style Reinforcement

Baseline (apply to every Tier-3 surface):

- **PAX_USERCOPY** — maple-tree itself does no user copies; values it stores (e.g. VMA pointers) are kernel-only.
- **PAX_KERNEXEC** — no writable text; iterator callbacks (`mtree_load`/`mas_walk`) reside in `.text`.
- **PAX_RANDKSTACK** — caller's syscall entry randomizes stack base.
- **PAX_REFCOUNT** — `ma_state` debug refcounts (under `CONFIG_DEBUG_MAPLE_TREE`) saturate.
- **PAX_MEMORY_SANITIZE** — `maple_node` slab cache zero-on-free; node-reuse never leaks prior range/value data.
- **PAX_UDEREF** — no user-pointer deref.
- **PAX_RAP / kCFI** — augmented-tree callbacks (split/merge ops) type-tagged.
- **GRKERNSEC_HIDESYM** — `mas_*`, `mtree_*`, `mt_*` internals hidden from non-CAP_SYSLOG kallsyms.
- **GRKERNSEC_DMESG** — `MT_BUG_ON` / `MAS_WARN_ON` gated behind CAP_SYSLOG.

Subsystem-specific reinforcement:

- **Range-keyed B+tree traversal integrity** — every node stores its `min`/`max` pivots; `mas_walk` asserts `min <= index <= max` on descent and bails with `mas_set_err(mas, -EINVAL)` rather than continuing into a corrupted child. PaX promotes this to BUG().
- **Slot/pivot invariants under `CONFIG_DEBUG_MAPLE_TREE`** — `mt_validate` walks the entire tree checking pivot monotonicity, slot/pivot pairing, child parent-back-pointers, and gap accounting; runs at every write under DEBUG, sampled under production-grsec.
- **`mas_set_external_lock` lockdep linkage** — embedded trees (mm `vma` tree, `kernfs`) declare their lockdep parent; cross-tree lock-class confusion (e.g. taking `mm->mmap_lock` while holding `i_mmap_rwsem`) detected as a class violation.
- **`__mt_dup` deep-copy under source lock** — fork-time VMA-tree duplication snapshots under `mmap_write_lock(src)`; PaX additionally asserts target tree is empty (refusing partial copy onto a non-empty destination).
- **Rationale** — maple-tree is the new mm VMA backing structure; corruption here is corruption of every process's address-space mapping. Mandatory pivot-monotonicity checks + lockdep-tracked external locks + DEBUG validation make tree corruption immediately fatal rather than slowly exploitable.

## Open Questions

- `lib/idr.c` migration: upstream is investigating replacing IDR/IDA with maple-tree-backed implementations; Rookery should track this and decide whether to mirror the migration or maintain the legacy radix-tree IDA.
- Per-CONFIG_DEBUG_MAPLE_TREE: `mt_dump` / `mt_validate` / `MT_BUG_ON` / `MAS_WARN_ON` are heavy debug aids; should Rookery enable them by default in development builds or gate behind a kernel param?
- Per-CONFIG_MAPLE_RCU_DISABLED: rarely enabled (test harness only); should Rookery support disabling RCU mode at all, or hard-require RCU?
- Per-`MA_STATE_PREALLOC` semantics under nested writes (e.g. split-then-rebalance): are pre-allocated nodes correctly accounted across multi-step writes?
- Per-`mt_height(mt) <= MAPLE_HEIGHT_MAX (31)`: with ULONG_MAX range and 16-fanout per level, the theoretical worst-case height is ≤ 16; the 31 bound is generous but consumes 5 flag bits — could be reduced if we add other flags.
- Per-`maple_arange_64` gap[] vs `maple_range_64` (no gap[]): the choice is made at tree creation via MT_FLAGS_ALLOC_RANGE; cross-conversion is not supported. Document this constraint.

## Out of Scope

- `mm/mmap.md` / `mm/vma.md` — owns `find_vma`, `vma_iter_*`, `vm_unmapped_area`, `__split_vma`, `vma_merge` consumer code.
- `kernel/fork.c` `dup_mmap` — uses `mtree_dup`; covered in `kernel/fork.md` if added.
- `lib/idr.c` / `lib/radix-tree.c` — alternative range/ID-allocation data structures; covered in `lib/data-structures.md`.
- `lib/xarray.c` — sibling structure (page-cache indices); covered separately.
- `mm/slab.md` — owns `kmem_cache_create` / `kmem_cache_alloc` semantics for `maple_node_cache`.
- `include/linux/rcupdate.h` — owns `rcu_read_lock`, `rcu_assign_pointer`, `call_rcu` semantics.
- `kernel/locking/spinlock.md` — owns `spin_lock` / `spin_unlock` semantics for ma_lock.
- `kernel/locking/lockdep.md` (if added) — owns `lockdep_map` / `lock_is_held` semantics for the external-lock integration.
- CONFIG_DEBUG_MAPLE_TREE harness (`mt_dump`, `mas_dump`, `MT_BUG_ON`, ...) — covered as a Tier-4 debug appendix if added.
- Userspace test harness (`tools/testing/radix-tree/maple.c`) — not part of kernel build.
- Implementation code.
