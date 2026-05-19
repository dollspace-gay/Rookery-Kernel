# Tier-3: fs/btrfs/free-space-cache.c — btrfs free-space cache (v1 inode-cache + v2 free-space-tree)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/free-space-cache.c (~4388 lines)
  - fs/btrfs/free-space-cache.h
  - fs/btrfs/free-space-tree.c (companion v2 driver — keyed serialization in the FREE_SPACE_TREE)
  - fs/btrfs/free-space-tree.h
  - include/uapi/linux/btrfs_tree.h (BTRFS_FREE_SPACE_OBJECTID, BTRFS_FREE_SPACE_TREE_OBJECTID, BTRFS_FREE_SPACE_INFO_KEY, BTRFS_FREE_SPACE_EXTENT_KEY, BTRFS_FREE_SPACE_BITMAP_KEY, BTRFS_FREE_SPACE_EXTENT, BTRFS_FREE_SPACE_BITMAP, BTRFS_FREE_SPACE_USING_BITMAPS)
-->

## Summary

Every btrfs **block group** (a contiguous logical-address chunk of disk-space carved out of `BTRFS_CHUNK_TREE_OBJECTID`) needs a fast, in-memory representation of its free extents to satisfy `find_free_extent()` on the allocation hot path. The free-space cache is that representation plus the on-disk persistence that lets it survive across mounts.

Per-block-group in-memory state lives in `struct btrfs_free_space_ctl` (hung off `block_group->free_space_ctl`):

- `free_space_offset`: per-rbtree of `struct btrfs_free_space` entries keyed by `(offset, bitmap)` — primary index used by `tree_search_offset()` for "is this offset free?".
- `free_space_bytes`: per-rbtree-cached (`rb_root_cached`) of the same entries keyed by `bytes` — secondary index used by `find_free_space()` when the caller starts at `block_group->start` and just wants "give me any extent ≥ N bytes".
- `tree_lock`: per-ctl spinlock; held over rb-tree mutation, bitmap mutation, and `free_space` accounting.
- `free_space`, `free_extents`, `total_bitmaps`, `extents_thresh`: per-ctl accounting + bitmap-promotion threshold (recomputed in `recalculate_thresholds`).
- `unit`: per-fs `sectorsize` (one bitmap bit = one `unit`).
- `start`: copy of `block_group->start` for bitmap-index arithmetic.
- `discardable_extents` / `discardable_bytes` (BTRFS_STAT_NR_ENTRIES tuple): per-ctl async-discard counters (`btrfs_discard_update_discardable`).
- `op`: per-ctl vtable (`btrfs_free_space_op::use_bitmap`) — bitmap-promotion policy.
- `block_group`: per-ctl back-pointer.
- `cache_writeout_mutex`, `trimming_ranges`: per-ctl trim/writeout serialization.

Per-extent entry: `struct btrfs_free_space { offset_index, bytes_index, offset, bytes, max_extent_size, *bitmap, list, trim_state, bitmap_extents }`. If `bitmap == NULL` it represents a contiguous run `[offset, offset+bytes)`. If `bitmap != NULL` it represents a region of `BTRFS_MAX_BITMAP_SIZE` bytes (one bit = one `unit`) used to densely encode a fragmented range — once the extent population in a region passes `extents_thresh`, the region is promoted to bitmap (`use_bitmap`); `bitmap_extents` counts the number of contiguous runs inside the bitmap. Promotion + steal-from-bitmap merging in `try_merge_free_space` / `steal_from_bitmap*`.

Per-allocator-cluster cache: `struct btrfs_free_cluster` (defined in `fs/btrfs/block-group.h`, manipulated here) — an RBT-shaped per-block-group "window" of dense free extents pre-selected for SSD / metadata workloads (`btrfs_find_space_cluster`, `btrfs_alloc_from_cluster`).

Two persistence schemes:

- **v1 inode-cache (legacy)** — one hidden inode per block-group in the **root tree**, keyed `(BTRFS_FREE_SPACE_OBJECTID = -11ULL, BTRFS_FREE_SPACE_KEY, block_group->start)`, body holding a serialized `btrfs_free_space_header` followed by extent + bitmap entries. Loaded via `load_free_space_cache → __load_free_space_cache`, written by `btrfs_write_out_cache → __btrfs_write_out_cache`, marker `btrfs_super_block::cache_generation`. Gated by mount option `space_cache` and by `btrfs_free_space_cache_v1_active()`. The v1 cache is **stale-after-crash** by design (next mount recomputes), so it carries CRC + generation + size guards (`io_ctl_check_generation`, `io_ctl_check_crc`).
- **v2 free-space-tree (`BTRFS_FREE_SPACE_TREE_OBJECTID = 10`)** — a real CoW-protected btree keyed per-block-group with three item types: `BTRFS_FREE_SPACE_INFO_KEY = 198` (per-bg summary: `extent_count + flags`, flag `BTRFS_FREE_SPACE_USING_BITMAPS = 1`), `BTRFS_FREE_SPACE_EXTENT_KEY = 199` (one item per free extent in extent-list mode), `BTRFS_FREE_SPACE_BITMAP_KEY = 200` (per-256-byte bitmap-page in bitmap mode; `BTRFS_FREE_SPACE_BITMAP_SIZE = 256`, `BTRFS_FREE_SPACE_BITMAP_BITS = 256*8`). Driven by `fs/btrfs/free-space-tree.c` (companion file); this file (`free-space-cache.c`) supplies the in-memory side, the v1 driver, and the shared add / remove / find-for-alloc / cluster / trim entry points used by both schemes.

This Tier-3 covers `fs/btrfs/free-space-cache.c` (~4388 lines) and the v1/v2 contract shared with `fs/btrfs/free-space-tree.c`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btrfs_free_space_ctl` | per-block-group in-memory ctl | `FreeSpaceCtl` |
| `struct btrfs_free_space` | per-extent / per-bitmap entry | `FreeSpaceEntry` |
| `struct btrfs_free_space_op` | per-ctl bitmap-promotion vtable | `FreeSpaceOp` |
| `struct btrfs_free_cluster` | per-allocator cluster window | `FreeCluster` |
| `struct btrfs_io_ctl` | per-v1-IO page-mapped stream | `FreeSpaceIoCtl` |
| `enum btrfs_trim_state` | UNTRIMMED / TRIMMED / TRIMMING | `TrimState` |
| `btrfs_init_free_space_ctl` | per-bg ctl init | `FreeSpaceCtl::init` |
| `btrfs_add_free_space` | per-add (untrimmed default) | `FreeSpaceCtl::add` |
| `btrfs_add_free_space_unused` | per-bg-init add (no zoned bump) | `FreeSpaceCtl::add_unused` |
| `btrfs_add_free_space_async_trimmed` | per-add marking trimmed if discard | `FreeSpaceCtl::add_async_trimmed` |
| `__btrfs_add_free_space` | per-add slow path (merge + steal-bitmap) | `FreeSpaceCtl::add_inner` |
| `__btrfs_add_free_space_zoned` | per-zoned-allocator add | `FreeSpaceCtl::add_zoned` |
| `btrfs_remove_free_space` | per-allocator remove | `FreeSpaceCtl::remove` |
| `btrfs_find_space_for_alloc` | per-allocator extract | `FreeSpaceCtl::find_for_alloc` |
| `find_free_space` | per-allocator search (offset / bytes index) | `FreeSpaceCtl::find` |
| `btrfs_find_space_cluster` | per-cluster build | `FreeSpaceCtl::find_cluster` |
| `setup_cluster_no_bitmap` / `setup_cluster_bitmap` | per-cluster builder paths | `FreeSpaceCtl::setup_cluster_no_bitmap` / `setup_cluster_bitmap` |
| `btrfs_alloc_from_cluster` | per-allocator pop-from-cluster | `FreeSpaceCtl::alloc_from_cluster` |
| `btrfs_alloc_from_bitmap` | per-cluster bitmap-pop | `FreeSpaceCtl::alloc_from_bitmap` |
| `btrfs_init_free_cluster` | per-cluster init | `FreeCluster::init` |
| `btrfs_return_cluster_to_free_space` | per-cluster teardown | `FreeSpaceCtl::return_cluster` |
| `__btrfs_return_cluster_to_free_space` | per-locked teardown | `FreeSpaceCtl::return_cluster_inner` |
| `tree_insert_offset` | per-offset-rbtree insert | `FreeSpaceCtl::tree_insert_offset` |
| `tree_search_offset` | per-offset-rbtree lookup | `FreeSpaceCtl::tree_search_offset` |
| `link_free_space` / `unlink_free_space` | per-rb maintenance | `FreeSpaceCtl::link` / `unlink` |
| `relink_bitmap_entry` | per-bytes-rb relink after bitmap mutation | `FreeSpaceCtl::relink_bitmap_entry` |
| `search_bitmap` | per-bitmap scan | `FreeSpaceCtl::search_bitmap` |
| `btrfs_bitmap_set_bits` / `bitmap_clear_bits` | per-bitmap mutate | `FreeSpaceCtl::bitmap_set_bits` / `bitmap_clear_bits` |
| `add_new_bitmap` | per-bitmap alloc + link | `FreeSpaceCtl::add_new_bitmap` |
| `free_bitmap` | per-bitmap drop | `FreeSpaceCtl::free_bitmap` |
| `insert_into_bitmap` | per-add bitmap path | `FreeSpaceCtl::insert_into_bitmap` |
| `remove_from_bitmap` | per-remove bitmap path | `FreeSpaceCtl::remove_from_bitmap` |
| `use_bitmap` / `free_space_op.use_bitmap` | per-ctl promotion policy | `FreeSpaceCtl::use_bitmap` |
| `recalculate_thresholds` | per-ctl threshold recompute | `FreeSpaceCtl::recalculate_thresholds` |
| `try_merge_free_space` | per-add adjacent-merge | `FreeSpaceCtl::try_merge_free_space` |
| `steal_from_bitmap` / `_to_end` / `_to_front` | per-add bitmap-steal | `FreeSpaceCtl::steal_from_bitmap*` |
| `__btrfs_remove_free_space_cache` | per-bg drop all | `FreeSpaceCtl::drop_all_inner` |
| `btrfs_remove_free_space_cache` | per-bg drop (locked + clusters) | `FreeSpaceCtl::drop_all` |
| `btrfs_is_free_space_trimmed` | per-bg trimmed-state predicate | `FreeSpaceCtl::is_trimmed` |
| `btrfs_dump_free_space` | per-bg klog dump | `FreeSpaceCtl::dump` |
| `lookup_free_space_inode` / `__lookup_free_space_inode` | per-bg v1 inode lookup | `FreeSpaceV1::lookup_inode` / `lookup_inode_inner` |
| `create_free_space_inode` / `__create_free_space_inode` | per-bg v1 inode create | `FreeSpaceV1::create_inode` / `create_inode_inner` |
| `btrfs_remove_free_space_inode` | per-bg v1 inode delete | `FreeSpaceV1::remove_inode` |
| `btrfs_truncate_free_space_cache` | per-bg v1 inode truncate | `FreeSpaceV1::truncate_inode` |
| `load_free_space_cache` / `__load_free_space_cache` | per-bg v1 load | `FreeSpaceV1::load` / `load_inner` |
| `copy_free_space_cache` | per-bg tmp-ctl → live-ctl | `FreeSpaceV1::copy_to_live` |
| `btrfs_write_out_cache` / `__btrfs_write_out_cache` | per-bg v1 write | `FreeSpaceV1::write_out` / `write_out_inner` |
| `write_cache_extent_entries` / `write_pinned_extent_entries` | per-bg v1 record streams | `FreeSpaceV1::write_extent_entries` / `write_pinned_entries` |
| `btrfs_wait_cache_io` / `__btrfs_wait_cache_io` | per-bg v1 IO wait | `FreeSpaceV1::wait_cache_io` / `wait_cache_io_inner` |
| `flush_dirty_cache` | per-v1 write filemap_write_and_wait | `FreeSpaceV1::flush_dirty_cache` |
| `io_ctl_init` / `_free` / `_map_page` / `_unmap_page` | per-v1 page-mapped IO | `FreeSpaceIoCtl::init` / `free` / `map_page` / `unmap_page` |
| `io_ctl_set_generation` / `_check_generation` | per-v1 generation guard | `FreeSpaceIoCtl::set_generation` / `check_generation` |
| `io_ctl_set_crc` / `_check_crc` | per-v1 per-page CRC32C | `FreeSpaceIoCtl::set_crc` / `check_crc` |
| `io_ctl_add_entry` / `_add_bitmap` / `_read_entry` / `_read_bitmap` | per-v1 record codec | `FreeSpaceIoCtl::add_entry` / `add_bitmap` / `read_entry` / `read_bitmap` |
| `btrfs_free_space_cache_v1_active` | per-fs v1-active predicate | `FreeSpaceV1::active` |
| `btrfs_set_free_space_cache_v1_active` | per-fs v1 toggle (ioctl) | `FreeSpaceV1::set_active` |
| `cleanup_free_space_cache_v1` | per-fs v1 inode purge | `FreeSpaceV1::cleanup_all` |
| `btrfs_trim_block_group` / `_extents` / `_bitmaps` | per-bg fitrim | `FreeSpaceCtl::trim_block_group` / `trim_extents` / `trim_bitmaps` |
| `trim_no_bitmap` / `trim_bitmaps` | per-trim variants | `FreeSpaceCtl::trim_no_bitmap` / `trim_bitmaps` |
| `do_trimming` | per-trim discard issue | `FreeSpaceCtl::do_trimming` |
| `reset_trimming_bitmap` / `end_trimming_bitmap` | per-trim bitmap state | `FreeSpaceCtl::reset_trimming_bitmap` / `end_trimming_bitmap` |
| `btrfs_trim_fully_remapped_block_group` | per-remap-tree shortcut | `FreeSpaceCtl::trim_fully_remapped_bg` |
| `btrfs_free_space_init` / `_exit` | per-module kmem_cache | `FreeSpace::init` / `exit` |
| `btrfs_free_space_cachep` | per-entry kmem_cache | `FREE_SPACE_CACHEP` |
| `btrfs_free_space_bitmap_cachep` | per-bitmap-page kmem_cache | `FREE_SPACE_BITMAP_CACHEP` |
| (v2) `btrfs_create_free_space_tree` | per-fs v2 create | `FreeSpaceV2::create_tree` |
| (v2) `btrfs_load_free_space_tree` | per-bg v2 load (called from caching kthread) | `FreeSpaceV2::load` |
| (v2) `btrfs_search_free_space_info` | per-bg v2 INFO_KEY lookup | `FreeSpaceV2::search_info` |
| (v2) `btrfs_add_to_free_space_tree` / `_remove_from_free_space_tree` | per-extent v2 mutate | `FreeSpaceV2::add` / `remove` |
| (v2) `btrfs_add_block_group_free_space` / `_remove_block_group_free_space` | per-bg v2 lifecycle | `FreeSpaceV2::add_bg` / `remove_bg` |
| (v2) `btrfs_convert_free_space_to_bitmaps` / `_to_extents` | per-bg v2 mode flip | `FreeSpaceV2::convert_to_bitmaps` / `convert_to_extents` |
| (v2) `BTRFS_FREE_SPACE_BITMAP_SIZE` | per-page bitmap 256B | `FREE_SPACE_BITMAP_SIZE` |
| (v2) `BTRFS_FREE_SPACE_USING_BITMAPS` | per-bg v2 mode flag | `FREE_SPACE_USING_BITMAPS` |

## Compatibility contract

REQ-1: struct btrfs_free_space (per-entry):
- offset_index: per-rb_node on `ctl.free_space_offset` (keyed by `offset`).
- bytes_index: per-rb_node on `ctl.free_space_bytes` (keyed by `bytes`, cached-leftmost ⟹ largest).
- offset: per-entry logical start.
- bytes: per-entry length-in-bytes.
- max_extent_size: per-entry hint (bitmap only; cached largest contiguous run).
- bitmap: per-entry bitmap-page pointer (NULL ⟹ pure extent; non-NULL ⟹ bitmap).
- list: per-entry temp-list link (cluster / trim / steal traversal).
- trim_state: per-entry UNTRIMMED / TRIMMED / TRIMMING.
- bitmap_extents: per-entry count of contiguous runs inside the bitmap (s32; clamped ≥ 0).

REQ-2: struct btrfs_free_space_ctl (per-block-group ctl):
- tree_lock: per-ctl spinlock; held over all rb-tree + bitmap mutation.
- free_space_offset: per-rb_root of entries by offset.
- free_space_bytes: per-rb_root_cached of entries by bytes (largest first).
- free_space: per-ctl free-byte accumulator.
- extents_thresh: per-ctl extent-count promotion threshold (recompute via `recalculate_thresholds`).
- free_extents: per-ctl count of non-bitmap entries.
- total_bitmaps: per-ctl count of bitmap entries.
- unit: per-fs sectorsize (one bitmap-bit per unit).
- start: per-bg start (copy of `block_group->start`).
- discardable_extents[BTRFS_STAT_NR_ENTRIES]: per-ctl async-discard accounting.
- discardable_bytes[BTRFS_STAT_NR_ENTRIES]: per-ctl async-discard accounting.
- op: per-ctl `&free_space_op` (use_bitmap predicate).
- block_group: per-ctl back-pointer.
- cache_writeout_mutex: per-ctl trim/writeout serialization.
- trimming_ranges: per-ctl active-trim list.

REQ-3: btrfs_init_free_space_ctl(bg, ctl):
- spin_lock_init(&ctl.tree_lock).
- ctl.unit = fs_info.sectorsize.
- ctl.start = bg.start.
- ctl.op = &free_space_op.
- INIT_LIST_HEAD(&ctl.trimming_ranges).
- mutex_init(&ctl.cache_writeout_mutex).
- ctl.free_space_offset = RB_ROOT.
- ctl.free_space_bytes = RB_ROOT_CACHED.
- /* extents_thresh initialized via recalculate_thresholds on first add */
- bg.free_space_ctl = ctl.

REQ-4: btrfs_add_free_space(bg, bytenr, size):
- /* Skip remapped block groups (REMAP_TREE: tracked elsewhere) */
- if bg.flags & BTRFS_BLOCK_GROUP_REMAPPED: return 0.
- /* Zoned filesystems use the linear allocator */
- if btrfs_is_zoned(bg.fs_info): return __btrfs_add_free_space_zoned(bg, bytenr, size, true).
- /* If DISCARD_SYNC: mark as TRIMMED — discard ran inline */
- trim_state = btrfs_test_opt(DISCARD_SYNC) ? TRIMMED : UNTRIMMED.
- return __btrfs_add_free_space(bg, bytenr, size, trim_state).

REQ-5: __btrfs_add_free_space(bg, offset, bytes, trim_state):
- info = kmem_cache_zalloc(btrfs_free_space_cachep, GFP_NOFS).
- info.offset = offset; info.bytes = bytes; info.trim_state = trim_state.
- INIT_LIST_HEAD(&info.list).
- spin_lock(&ctl.tree_lock).
- /* Merge with adjacent free entries */
- try_merge_free_space(ctl, info, true).
- /* If a neighboring bitmap-region can absorb us: steal */
- steal_from_bitmap(ctl, info, true).
- /* Insert: into a bitmap if extents_thresh exceeded, else as a pure extent */
- ret = insert_into_bitmap(ctl, info).
- if ret == 1: /* not bitmap-inserted */ link_free_space(ctl, info).
- btrfs_discard_update_discardable(bg).
- spin_unlock(&ctl.tree_lock).

REQ-6: btrfs_remove_free_space(bg, offset, bytes):
- /* Zoned: just advance alloc_offset (no rbtree) */
- if btrfs_is_zoned(bg.fs_info): adjust alloc_offset; return 0.
- spin_lock(&ctl.tree_lock).
- info = tree_search_offset(ctl, offset, 0, 0).
- if !info: info = tree_search_offset(ctl, offset_to_bitmap(ctl, offset), 1, 0).
- if !info.bitmap:
  - /* Pure-extent path: unlink, truncate, possibly split */
  - unlink_free_space(ctl, info, true).
  - if offset == info.offset:
    - to_free = min(bytes, info.bytes).
    - info.bytes -= to_free; info.offset += to_free.
    - if info.bytes: link_free_space(ctl, info); else: kmem_cache_free.
  - else:
    - /* Split: keep prefix, recurse for suffix */
    - old_end = info.bytes + info.offset.
    - info.bytes = offset - info.offset.
    - link_free_space(ctl, info).
    - if old_end > offset + bytes:
      - spin_unlock(&ctl.tree_lock).
      - __btrfs_add_free_space(bg, offset+bytes, old_end-(offset+bytes), info.trim_state).
      - return ret.
- else:
  - /* Bitmap path */
  - remove_from_bitmap(ctl, info, &offset, &bytes).  /* -EAGAIN ⟹ re-search */
- btrfs_discard_update_discardable(bg).
- spin_unlock(&ctl.tree_lock).

REQ-7: btrfs_find_space_for_alloc(bg, offset, bytes, empty_size, *max_extent_size) → u64 ret_offset:
- ctl = bg.free_space_ctl.
- bytes_search = bytes + empty_size.
- use_bytes_index = (offset == bg.start)  /* caller takes "anywhere" */.
- spin_lock(&ctl.tree_lock).
- entry = find_free_space(ctl, &offset, &bytes_search, bg.full_stripe_len, max_extent_size, use_bytes_index).
- if !entry: return 0.
- ret = offset.
- if entry.bitmap:
  - bitmap_clear_bits(ctl, entry, offset, bytes, update_stat=true).
  - if !entry.bytes: free_bitmap(ctl, entry).
- else:
  - unlink_free_space(ctl, entry, true).
  - align_gap = entry.offset; align_gap_len = offset - entry.offset.
  - entry.offset = offset + bytes; entry.bytes -= bytes + align_gap_len.
  - if !entry.bytes: kmem_cache_free(entry); else: link_free_space(ctl, entry).
- spin_unlock(&ctl.tree_lock).
- if align_gap_len > 0: __btrfs_add_free_space(bg, align_gap, align_gap_len, entry.trim_state).
- return ret.

REQ-8: per-bitmap representation:
- One `btrfs_free_space` with `bitmap != NULL` covers a region of size `BITS_PER_BITMAP * ctl.unit = 256 * 8 * sectorsize`.
- `offset_to_bitmap(ctl, offset)` rounds `offset` down to the region start.
- `offset_to_bit(bitmap_start, unit, offset)` maps to a bit-index inside the bitmap.
- `bitmap_extents` tracks the number of contiguous runs inside the bitmap, used to drive promotion / demotion + discardable_extents accounting.
- `use_bitmap` (vtable): true ⟺ `ctl.free_extents > ctl.extents_thresh`  ∨  the entry under construction already overlaps a bitmap.

REQ-9: cluster cache (`struct btrfs_free_cluster`):
- root: per-cluster rbtree of `btrfs_free_space` entries.
- window_start: per-cluster aligned start.
- max_size: per-cluster largest contiguous run (popcount of bitmap or extent length).
- block_group_list / block_group: per-cluster owner backlink.
- lock + refill_lock: per-cluster + per-refill serialization.
- fragmented: per-cluster bit (set by `setup_cluster_bitmap` when only bitmap-runs available).
- btrfs_find_space_cluster: per-{SSD_SPREAD,METADATA,DATA} compute (cont1_bytes, min_bytes); call setup_cluster_no_bitmap then _bitmap.
- btrfs_alloc_from_cluster: per-pop window_start increment; if cluster empty → `btrfs_return_cluster_to_free_space` (release entries back to ctl).
- btrfs_return_cluster_to_free_space → __btrfs_return_cluster_to_free_space: per-locked re-link of cluster entries into the ctl rb-trees.
- btrfs_init_free_cluster: per-cluster spin_lock_init + RB_ROOT + fragmented=false + INIT_LIST_HEAD(block_group_list).

REQ-10: v1 inode cache (legacy):
- One hidden inode per block-group in `fs_info.tree_root`, keyed `(BTRFS_FREE_SPACE_OBJECTID, BTRFS_FREE_SPACE_KEY, bg.start)`.
- Inode body: `btrfs_free_space_header { location, generation, num_entries, num_bitmaps }` then per-page records: { btrfs_free_space_entry × N | bitmap_page × M }.
- Per-page CRC32C: `io_ctl_set_crc(idx)` / `io_ctl_check_crc(idx)` (first 4 bytes of each page hold the CRC; `btrfs_crc32c_final` writes big-endian).
- Page 0 also carries `generation == bg.fs_info.generation` (`io_ctl_set_generation` / `_check_generation`).
- Load:
  - load_free_space_cache(bg) → if bg.disk_cache_state == BTRFS_DC_WRITTEN: __load_free_space_cache into a tmp_ctl, validate `tmp_ctl.free_space == bg.length - bg.used - bg.bytes_super`; on match: copy_free_space_cache → live ctl.
  - On mismatch / CRC fail / generation fail: discard, set bg.disk_cache_state = BTRFS_DC_CLEAR, warn-and-rebuild.
- Write:
  - btrfs_write_out_cache(trans, bg, path) → __btrfs_write_out_cache → io_ctl_init + write_cache_extent_entries + write_pinned_extent_entries + write_bitmap_entries + flush_dirty_cache.
- Wait:
  - btrfs_wait_cache_io: per-trans `__btrfs_wait_cache_io` waits + clears IO_DONE; on failure: marks DC_CLEAR.
- Cleanup:
  - btrfs_truncate_free_space_cache: per-bg truncate via i_size=0.
  - btrfs_remove_free_space_inode: per-bg unlink hidden inode.

REQ-11: v1 active toggle (sysfs / `mount -o space_cache`):
- btrfs_free_space_cache_v1_active(fs_info) = (btrfs_super_cache_generation != 0).
- btrfs_set_free_space_cache_v1_active(fs_info, active):
  - Start a transaction on tree_root.
  - if !active:
    - set BTRFS_FS_CLEANUP_SPACE_CACHE_V1.
    - cleanup_free_space_cache_v1(fs_info, trans): for each bg in block_group_cache_tree: btrfs_remove_free_space_inode(trans, NULL, bg).
  - btrfs_commit_transaction; clear BTRFS_FS_CLEANUP_SPACE_CACHE_V1.

REQ-12: v2 free-space-tree (`BTRFS_FREE_SPACE_TREE_OBJECTID = 10`):
- Per-fs single CoW btree (companion: `fs/btrfs/free-space-tree.c`).
- BTRFS_FREE_SPACE_INFO_KEY = 198: per-bg `{extent_count, flags}`; `flags & BTRFS_FREE_SPACE_USING_BITMAPS = 1` ⟺ mode is bitmap-encoded; otherwise extent-encoded.
- BTRFS_FREE_SPACE_EXTENT_KEY = 199: per-extent `(start, BTRFS_FREE_SPACE_EXTENT, length)`; no value body.
- BTRFS_FREE_SPACE_BITMAP_KEY = 200: per-page `(start, BTRFS_FREE_SPACE_BITMAP, length)` with a `BTRFS_FREE_SPACE_BITMAP_SIZE = 256`-byte value (`BTRFS_FREE_SPACE_BITMAP_BITS = 2048`).
- Per-extent type bytes (legacy in-block-group-item flags): `BTRFS_FREE_SPACE_EXTENT = 1`, `BTRFS_FREE_SPACE_BITMAP = 2`.
- v2 driver provides: `btrfs_create_free_space_tree(fs_info)` (one-time fs init), `btrfs_delete_free_space_tree(fs_info)`, `btrfs_rebuild_free_space_tree(fs_info)` (post-corruption), `btrfs_load_free_space_tree(caching_ctl)` (per-caching-kthread populate of in-memory ctl), `btrfs_add_block_group_free_space` / `_remove_block_group_free_space` (per-bg lifecycle), `btrfs_add_to_free_space_tree` / `_remove_from_free_space_tree` (per-extent on-disk mutate, called from delayed-ref + extent-tree paths), `btrfs_search_free_space_info` (per-bg INFO_KEY lookup), `btrfs_convert_free_space_to_bitmaps` / `_to_extents` (mode flip when extent_count crosses threshold).
- v2 is **crash-safe** (it's a real CoW btree → covered by tree-log + super-block transid), so unlike v1 it does not require a generation match on load.

REQ-13: zoned allocator carve-out:
- For zoned devices the free-space-cache is not used as an RBT/bitmap; `__btrfs_add_free_space_zoned` just advances `block_group.alloc_offset`.
- `btrfs_remove_free_space` on zoned just bumps `alloc_offset` to `offset + bytes - bg.start` if behind (used by tree-log replay).

REQ-14: trim (FITRIM, discard):
- btrfs_trim_block_group(bg, *trimmed, start, end, minlen) → btrfs_trim_block_group_extents + btrfs_trim_block_group_bitmaps.
- Per-trim batch lives on `ctl.trimming_ranges` (locked by `ctl.cache_writeout_mutex`).
- trim_no_bitmap: per-extent issue `do_trimming` for runs ≥ minlen, marking entries TRIMMED.
- trim_bitmaps: per-bitmap scan + slice + issue.
- reset_trimming_bitmap / end_trimming_bitmap: per-bitmap TRIMMING ↔ TRIMMED transitions, allowing async re-add while trim is in flight without re-issuing discard.
- btrfs_trim_fully_remapped_block_group(bg): per-REMAP_TREE shortcut — issue one big discard for the entire bg and mark as trimmed.

REQ-15: kmem caches:
- btrfs_free_space_cachep: KMEM_CACHE(struct btrfs_free_space, 0); per-entry slab.
- btrfs_free_space_bitmap_cachep: kmem_cache_create("btrfs_free_space_bitmap", PAGE_SIZE, PAGE_SIZE, 0, NULL); per-bitmap-page (one PAGE_SIZE allocation per bitmap entry).
- btrfs_free_space_init/_exit: per-module create/destroy.

REQ-16: locking discipline:
- ctl.tree_lock: held over rb-tree + bitmap mutation + `ctl.free_space` updates.
- cluster.lock + cluster.refill_lock: nested inside ctl.tree_lock (lock order: ctl.tree_lock → cluster.lock → cluster.refill_lock).
- ctl.cache_writeout_mutex: held over trim batch enumeration to bound the live `trimming_ranges` list.
- bg.lock: protects `disk_cache_state`; never held with ctl.tree_lock.

REQ-17: discardable accounting (per-async-discard):
- discardable_extents[BTRFS_STAT_CURR] / discardable_bytes[BTRFS_STAT_CURR]: live count.
- discardable_extents[BTRFS_STAT_PREV]: snapshot at last `btrfs_discard_update_discardable` for delta-emit.
- Updated by link/unlink/bitmap_set/bitmap_clear paths whenever `trim_state == UNTRIMMED`.

REQ-18: per-promotion policy (`free_space_op.use_bitmap`):
- Returns true if `ctl.free_extents > ctl.extents_thresh` OR if `info` already overlaps an existing bitmap region.
- `extents_thresh` initial: `(SZ_32K / sizeof(btrfs_free_space) * ctl.unit) / SZ_32K`; recalc each major add/remove.

## Acceptance Criteria

- [ ] AC-1: `btrfs_init_free_space_ctl(bg, ctl)` → ctl initialized; bg.free_space_ctl = ctl; ctl.unit == fs_info.sectorsize.
- [ ] AC-2: `btrfs_add_free_space(bg, off, n)` adjacent to existing free run → entries merged (single `btrfs_free_space`); `ctl.free_space += n`; `ctl.free_extents` unchanged.
- [ ] AC-3: Repeated `btrfs_add_free_space` exceeding `extents_thresh` → next add returns via bitmap-insertion path (entry created with `bitmap != NULL`); `ctl.total_bitmaps += 1`.
- [ ] AC-4: `btrfs_remove_free_space(bg, off, n)` middle of an extent → splits entry; suffix re-added via `__btrfs_add_free_space`; `ctl.free_extents += 1`.
- [ ] AC-5: `btrfs_find_space_for_alloc(bg, bg.start, bytes, empty_size, &max)` against empty ctl → returns 0, `max_extent_size = 0`.
- [ ] AC-6: `btrfs_find_space_for_alloc` on cluster-aligned ctl → returns aligned offset within block-group; gap bytes (alignment slop) get re-added via `__btrfs_add_free_space` with original trim_state preserved.
- [ ] AC-7: `btrfs_find_space_cluster(bg, cluster, off, bytes, empty)` on SSD_SPREAD with sufficient contiguous free → cluster populated, `cluster.block_group == bg`, `cluster.window_start` aligned, list_add_tail to `bg.cluster_list`.
- [ ] AC-8: `btrfs_alloc_from_cluster` until cluster empty → final pop triggers `__btrfs_return_cluster_to_free_space` (cluster reset, entries returned to ctl).
- [ ] AC-9: v1 `load_free_space_cache(bg)` with `bg.disk_cache_state != BTRFS_DC_WRITTEN` → returns 0 (no load); ctl untouched.
- [ ] AC-10: v1 `__load_free_space_cache` with CRC-mismatched page → returns < 0; caller sets `bg.disk_cache_state = BTRFS_DC_CLEAR`; ctl scrubbed via `__btrfs_remove_free_space_cache(&tmp_ctl)`.
- [ ] AC-11: v1 `__load_free_space_cache` with `tmp_ctl.free_space != bg.length - bg.used - bg.bytes_super` → "wrong amount of free space" warn; cache discarded; rebuild scheduled.
- [ ] AC-12: v1 `btrfs_write_out_cache → __btrfs_write_out_cache → flush_dirty_cache` → all extent entries + bitmap pages written + per-page CRC stamped + generation == fs_info.generation on page 0.
- [ ] AC-13: `btrfs_set_free_space_cache_v1_active(fs_info, false)` → per-bg `btrfs_remove_free_space_inode` invoked under one transaction; cache_generation cleared at commit.
- [ ] AC-14: v2 `BTRFS_FREE_SPACE_INFO_KEY` `extent_count > threshold` → `btrfs_convert_free_space_to_bitmaps` flips mode; subsequent v2 reads decode via `BTRFS_FREE_SPACE_BITMAP_KEY` pages.
- [ ] AC-15: Zoned: `btrfs_add_free_space` returns via `__btrfs_add_free_space_zoned` (no rbtree mutation); `btrfs_remove_free_space` may bump `bg.alloc_offset` (log-replay path).
- [ ] AC-16: `btrfs_trim_block_group(bg, &trimmed, 0, U64_MAX, minlen)` → all extents ≥ minlen issued via `do_trimming`; per-entry trim_state == TRIMMED on completion; `*trimmed` set to total discarded.

## Architecture

```
struct FreeSpaceEntry {
  offset_index: RbNode,                  // index in FreeSpaceCtl.free_space_offset
  bytes_index: RbNode,                   // index in FreeSpaceCtl.free_space_bytes (cached)
  offset: u64,                           // logical start
  bytes: u64,                            // length
  max_extent_size: u64,                  // hint (bitmap only)
  bitmap: Option<KBox<[u8; BITMAP_PAGE_BYTES]>>,
  list: ListHead,
  trim_state: TrimState,
  bitmap_extents: i32,
}

struct FreeSpaceCtl {
  tree_lock: SpinLock<()>,
  free_space_offset: RbRoot<FreeSpaceEntry>,
  free_space_bytes: RbRootCached<FreeSpaceEntry>,
  free_space: u64,
  extents_thresh: i32,
  free_extents: i32,
  total_bitmaps: i32,
  unit: u32,                             // sectorsize
  start: u64,                            // bg.start
  discardable_extents: [i32; STAT_NR_ENTRIES],
  discardable_bytes: [i64; STAT_NR_ENTRIES],
  op: &'static FreeSpaceOp,
  block_group: *BlockGroup,
  cache_writeout_mutex: Mutex<()>,
  trimming_ranges: ListHead,
}

struct FreeCluster {
  lock: SpinLock<()>,
  refill_lock: SpinLock<()>,
  root: RbRoot<FreeSpaceEntry>,
  window_start: u64,
  max_size: u64,
  fragmented: bool,
  block_group_list: ListHead,
  block_group: Option<*BlockGroup>,
}

struct FreeSpaceIoCtl {
  cur: *u8, orig: *u8,
  page: *Page,
  pages: *[*Page],
  fs_info: *FsInfo,
  inode: *Inode,
  size: usize,
  index: i32,
  num_pages: i32,
  entries: i32,
  bitmaps: i32,
}
```

`FreeSpaceCtl::add(bg, bytenr, size) -> Result<()>`:
1. /* Skip REMAP_TREE-tracked block groups + zoned-allocator shortcut */
2. if bg.flags & BLOCK_GROUP_REMAPPED: return Ok.
3. if fs_info.zoned: return add_zoned(bg, bytenr, size, true).
4. /* Trim-state inference */
5. trim_state = if fs_info.opts.DISCARD_SYNC { TRIMMED } else { UNTRIMMED }.
6. return add_inner(bg, bytenr, size, trim_state).

`FreeSpaceCtl::add_inner(bg, offset, bytes, trim_state) -> Result<()>`:
1. info = FreeSpaceEntry::new(offset, bytes, trim_state).
2. guard = ctl.tree_lock.lock().
3. try_merge_free_space(ctl, info, /*update_stat=*/true).
4. steal_from_bitmap(ctl, info, /*update_stat=*/true).
5. /* Promote or link */
6. ret = insert_into_bitmap(ctl, info).
7. if ret == 1 /* not bitmap-inserted */: link_free_space(ctl, info).
8. btrfs_discard_update_discardable(bg).
9. /* Drop guard */

`FreeSpaceCtl::remove(bg, offset, bytes) -> Result<()>`:
1. /* Zoned shortcut */
2. if fs_info.zoned: if bg.start + bg.alloc_offset < offset + bytes: bg.alloc_offset = offset + bytes - bg.start; return Ok.
3. guard = ctl.tree_lock.lock().
4. /* Recurse over an `again:` label until bytes==0 */
5. loop:
   - info = tree_search_offset(ctl, offset, exact=false, fuzzy=false).
   - if info.is_none(): info = tree_search_offset(ctl, offset_to_bitmap(ctl, offset), is_bitmap=true, fuzzy=false).
   - if info.is_none(): break /* warn on re-search */.
   - if !info.bitmap:
     - unlink_free_space(ctl, info, true).
     - if offset == info.offset:
       - to_free = min(bytes, info.bytes).
       - info.offset += to_free; info.bytes -= to_free.
       - if info.bytes > 0: link_free_space(ctl, info); else: drop info.
       - offset += to_free; bytes -= to_free.
       - if bytes == 0: break; else: continue.
     - else: /* split-prefix / recurse-suffix */ ...
   - else: /* bitmap */
     - if remove_from_bitmap(ctl, info, &mut offset, &mut bytes) == -EAGAIN: continue.
6. btrfs_discard_update_discardable(bg).

`FreeSpaceCtl::find_for_alloc(bg, offset, bytes, empty, max_extent_size) -> u64`:
1. bytes_search = bytes + empty.
2. use_bytes_index = (offset == bg.start).
3. guard = ctl.tree_lock.lock().
4. entry = find_free_space(ctl, &mut offset, &mut bytes_search, bg.full_stripe_len, max_extent_size, use_bytes_index).
5. if entry.is_none(): return 0.
6. ret = offset.
7. if entry.bitmap:
   - bitmap_clear_bits(ctl, entry, offset, bytes, true).
   - if !entry.bytes: free_bitmap(ctl, entry).
8. else:
   - unlink_free_space(ctl, entry, true).
   - align_gap = entry.offset; align_gap_len = offset - entry.offset.
   - entry.offset = offset + bytes.
   - entry.bytes -= bytes + align_gap_len.
   - if !entry.bytes: drop entry; else: link_free_space(ctl, entry).
9. drop guard.
10. if align_gap_len > 0: add_inner(bg, align_gap, align_gap_len, entry.trim_state).
11. return ret.

`FreeSpaceCtl::find_cluster(bg, cluster, offset, bytes, empty) -> Result<()>`:
1. /* Choose policy */
2. (cont1_bytes, min_bytes) = match (opts.SSD_SPREAD, bg.flags & METADATA):
   - SSD_SPREAD: (bytes + empty, bytes + empty),
   - METADATA: (bytes, sectorsize),
   - else: (max(bytes, (bytes + empty) >> 2), sectorsize).
3. guard_ctl = ctl.tree_lock.lock().
4. if ctl.free_space < bytes: return Err(NoSpace).
5. guard_cluster = cluster.lock.lock().
6. if cluster.block_group.is_some(): return Ok /* already populated */.
7. ret = setup_cluster_no_bitmap(bg, cluster, &bitmaps, offset, bytes + empty, cont1_bytes, min_bytes).
8. if ret != Ok: ret = setup_cluster_bitmap(bg, cluster, &bitmaps, offset, bytes + empty, cont1_bytes, min_bytes).
9. /* Clear temp list */
10. for entry in bitmaps: list_del_init(&entry.list).
11. if ret == Ok:
    - btrfs_get_block_group(bg).
    - list_add_tail(&cluster.block_group_list, &bg.cluster_list).
    - cluster.block_group = Some(bg).

`FreeSpaceV1::load(bg) -> i32`:
1. tmp_ctl = FreeSpaceCtl::new().
2. FreeSpaceCtl::init(bg, &mut tmp_ctl).
3. if bg.disk_cache_state != BTRFS_DC_WRITTEN: return 0.
4. path = path::alloc(); path.search_commit_root = true; path.skip_locking = true.
5. inode = lookup_free_space_inode(bg, path)?.
6. /* Re-check after iget — could have been invalidated */
7. if bg.disk_cache_state != BTRFS_DC_WRITTEN: goto out.
8. lockdep_set_class(inode.invalidate_lock, &btrfs_free_space_inode_key).
9. ret = load_inner(fs_info.tree_root, inode, &mut tmp_ctl, path, bg.start).
10. matched = (tmp_ctl.free_space == bg.length - bg.used - bg.bytes_super).
11. if matched:
    - copy_to_live(bg, &tmp_ctl).
    - ret = 1.
12. else:
    - drop_all_inner(&mut tmp_ctl).
    - warn("block group %llu has wrong amount of free space").
    - ret = -1.
13. if ret < 0:
    - bg.disk_cache_state = BTRFS_DC_CLEAR.
    - ret = 0.
14. iput(inode); return ret.

`FreeSpaceV1::set_active(fs_info, active) -> Result<()>`:
1. trans = btrfs_start_transaction(fs_info.tree_root, 0)?.
2. if !active:
   - fs_info.flags |= BTRFS_FS_CLEANUP_SPACE_CACHE_V1.
   - cleanup_all(fs_info, trans):
     - for bg in fs_info.block_group_cache_tree: btrfs_remove_free_space_inode(trans, NULL, bg)?.
3. btrfs_commit_transaction(trans).
4. fs_info.flags &= !BTRFS_FS_CLEANUP_SPACE_CACHE_V1.

`FreeSpaceV2::load(caching_ctl) -> Result<()>` (companion `fs/btrfs/free-space-tree.c`):
1. /* Driven from the per-block-group caching kthread */
2. info_item = search_info(bg, path, /*cow=*/0)?.
3. if info_item.flags & USING_BITMAPS:
   - for each BITMAP_KEY in [bg.start, bg.start + bg.length):
     - decode 256-byte bitmap page → __btrfs_add_free_space_async_trimmed(bg, run.start, run.len).
4. else:
   - for each EXTENT_KEY in [bg.start, bg.start + bg.length):
     - __btrfs_add_free_space_async_trimmed(bg, key.objectid, key.offset).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tree_lock_held_over_mutation` | INVARIANT | per-add / remove / find_for_alloc / search_bitmap: ctl.tree_lock held over rbtree + bitmap mutation. |
| `free_space_sum_equals_entries` | INVARIANT | per-ctl: ctl.free_space == Σ_entries entry.bytes (extent + popcount(bitmap) * unit). |
| `free_extents_counts_pure` | INVARIANT | per-ctl: ctl.free_extents == |{e ∈ ctl | e.bitmap.is_none()}|. |
| `total_bitmaps_counts_bitmap` | INVARIANT | per-ctl: ctl.total_bitmaps == |{e ∈ ctl | e.bitmap.is_some()}|. |
| `bitmap_extents_clamped_nonneg` | INVARIANT | per-bitmap: entry.bitmap_extents ≥ 0. |
| `find_for_alloc_returns_in_range` | INVARIANT | per-find_for_alloc result r ≠ 0 ⟹ bg.start ≤ r < bg.start + bg.length ∧ r + bytes ≤ bg.start + bg.length. |
| `align_gap_re_added` | INVARIANT | per-find_for_alloc: align_gap_len > 0 ⟹ add_inner(bg, align_gap, align_gap_len, _) called with original trim_state. |
| `cluster_window_aligned` | INVARIANT | per-find_cluster success: cluster.window_start aligned to fs.sectorsize. |
| `v1_crc_validated_before_decode` | INVARIANT | per-IoCtl read: io_ctl_check_crc(idx) precedes io_ctl_read_entry / _read_bitmap on page idx. |
| `v1_generation_validated_first` | INVARIANT | per-IoCtl read: io_ctl_check_generation precedes any io_ctl_read_entry. |
| `v1_free_space_matches_block_group` | INVARIANT | per-load: result Ok(1) ⟹ tmp_ctl.free_space == bg.length - bg.used - bg.bytes_super. |
| `kmem_cache_alloc_matched_by_free` | INVARIANT | per-entry: every kmem_cache_alloc(btrfs_free_space_cachep) paired with kmem_cache_free on drop path. |
| `bitmap_page_alloc_matched_by_free` | INVARIANT | per-bitmap: every kmem_cache_alloc(btrfs_free_space_bitmap_cachep) paired with free in free_bitmap. |
| `zoned_skips_rbtree` | INVARIANT | per-add / remove: fs_info.zoned ⟹ rbtree not mutated. |

### Layer 2: TLA+

`fs/btrfs/free-space-cache.tla`:
- Per-add + per-remove + per-find_for_alloc + per-cluster_find + per-cluster_alloc + per-v1-load + per-v1-write + per-v2-load.
- Properties:
  - `safety_no_overlapping_entries` — per-ctl: ∀ distinct entries e1, e2: ranges(e1) ∩ ranges(e2) = ∅.
  - `safety_free_space_invariant` — per-ctl: ctl.free_space == Σ ranges-bytes(e).
  - `safety_add_remove_symmetric` — per-bg: ∀ pair add(o, n) ; remove(o, n) followed by add(o, n) returns to same ctl state (modulo merge / bitmap-promotion noise).
  - `safety_cluster_disjoint_from_ctl` — per-cluster: entries in cluster.root are not simultaneously in ctl.free_space_offset (they're moved, not duplicated).
  - `safety_v1_crash_safety` — per-v1: post-crash mount with bg.disk_cache_state != DC_WRITTEN ⟹ load returns 0; cache rebuilt from extent tree.
  - `safety_v2_crash_safety` — per-v2: post-crash mount with valid free-space-tree (transid-matched) ⟹ load reconstructs identical ctl up to merge.
  - `liveness_find_for_alloc_progresses` — per-allocator: ctl.free_space ≥ bytes ⟹ find_for_alloc eventually returns ≠ 0 ∨ all candidate extents fail alignment.
  - `liveness_trim_completes` — per-trim: do_trimming returns or sets entry.trim_state = TRIMMED for at least one entry per scan.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FreeSpaceCtl::add_inner` post: ctl.free_space += size; rbtrees consistent | `FreeSpaceCtl::add_inner` |
| `FreeSpaceCtl::remove` post: ctl.free_space -= bytes (modulo split); no overlapping entries | `FreeSpaceCtl::remove` |
| `FreeSpaceCtl::find_for_alloc` post: ret ∈ {0} ∪ [bg.start, bg.end); ctl.free_space -= bytes (modulo align_gap re-add) | `FreeSpaceCtl::find_for_alloc` |
| `FreeSpaceCtl::find_cluster` post: ret = Ok ⟹ cluster non-empty ∧ cluster.block_group = Some(bg) | `FreeSpaceCtl::find_cluster` |
| `FreeSpaceCtl::alloc_from_cluster` post: ret > 0 ⟹ ret ∈ cluster's pre-pop window | `FreeSpaceCtl::alloc_from_cluster` |
| `FreeSpaceV1::load` post: ret = 1 ⟹ ctl populated to match bg's free_space; ret = 0 ⟹ ctl unchanged (rebuild scheduled) | `FreeSpaceV1::load` |
| `FreeSpaceV1::write_out` post: each page CRC-stamped; page 0 generation == fs_info.generation | `FreeSpaceV1::write_out` |
| `FreeSpaceV2::add` post: BTRFS_FREE_SPACE_EXTENT_KEY inserted (extent mode) OR BITMAP_KEY page bit set (bitmap mode); INFO_KEY extent_count adjusted | `FreeSpaceV2::add` |
| `FreeSpaceCtl::trim_block_group` post: ret = 0 ⟹ *trimmed = Σ run.bytes for discarded runs; per-entry trim_state = TRIMMED for discarded runs | `FreeSpaceCtl::trim_block_group` |
| `bitmap_extents_promotion` post: ctl.free_extents > extents_thresh ⟹ next add takes bitmap path | `use_bitmap` |

### Layer 4: Verus/Creusot functional

`Per-block-group allocator request → ctl.tree_lock → find_free_space (offset-index or bytes-index) → split + remove + align_gap re-add → return offset` semantic equivalence: per-Documentation/filesystems/btrfs.rst, `fs/btrfs/extent-tree.c::find_free_extent`, and `fs/btrfs/free-space-tree.c` v2 contract.

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

Free-space-cache reinforcement:

- **Per-ctl.tree_lock strict over mutation** — defense against per-rbtree torn-walk while allocator runs.
- **Per-tmp_ctl staging on v1 load** — defense against per-corrupt-cache leaking into live ctl (validated against `bg.length - bg.used - bg.bytes_super` before swap).
- **Per-page CRC32C on v1** — defense against per-disk-bitrot silently corrupting free-space records.
- **Per-page generation match on v1** — defense against per-replay-of-stale-cache after crash (caller falls back to rebuild).
- **Per-bg.disk_cache_state = DC_CLEAR on any load failure** — defense against per-repeated-corrupt-cache load.
- **Per-zoned-allocator bypass** — defense against per-zoned-device alignment violation (rbtree free-list is meaningless on SMR/zone-aligned storage).
- **Per-BG_REMAPPED skip** — defense against per-remap-tree double-accounting.
- **Per-DISCARD_SYNC inline-trim state propagation** — defense against per-async-discard re-issuing already-discarded runs.
- **Per-bitmap-extents clamp ≥ 0** — defense against per-int-underflow in `bitmap_extents` accounting.
- **Per-cluster.lock + cluster.refill_lock nested under ctl.tree_lock** — defense against per-deadlock with multi-bg cluster refill.
- **Per-WARN_ON(re_search) in remove** — defense against per-bitmap-vs-extent split bug (partial extent found in bitmap but other half missing).
- **Per-trimming_ranges + cache_writeout_mutex** — defense against per-trim issuing discard on a range that the allocator simultaneously re-added.
- **Per-v2 mode-flip threshold** — defense against per-INFO_KEY extent_count explosion (auto-flip to bitmap when extent_count exceeds bg.bitmap_high_thresh).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- v2 free-space-tree internals beyond the on-disk key contract (covered separately in a future `free-space-tree.md` Tier-3 if expanded)
- `find_free_extent` allocator policy (covered in `extent-tree.md` Tier-3)
- block-group caching kthread (covered in `block-group.md` if expanded)
- async-discard worker (covered in `discard.md` if expanded)
- zoned-allocator `alloc_offset` advance (covered in `zoned.md` if expanded)
- FITRIM ioctl plumbing (covered in `ioctl.md` if expanded)
- Implementation code
