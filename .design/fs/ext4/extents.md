# Tier-3: fs/ext4/extents.c — ext4 extent tree

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/ext4/00-overview.md
upstream-paths:
  - fs/ext4/extents.c (~6308 lines)
  - fs/ext4/ext4_extents.h
  - fs/ext4/ext4.h (EXT4_GET_BLOCKS_*, EXT4_EX_*)
-->

## Summary

ext4 stores the logical-to-physical block map of a file as an **extent tree**: a depth-bounded B+-tree of `(logical, length, physical)` records that compresses contiguous runs into a single record. The tree root lives inside `ext4_inode.i_block[]` (60 bytes), where the first 12 bytes hold an `ext4_extent_header` followed by an array of up to 4 `ext4_extent` entries (depth 0) or `ext4_extent_idx` entries (depth ≥ 1). When the inode-resident root overflows, the tree grows out-of-line through `ext4_ext_grow_indepth`/`ext4_ext_split`, with a hard cap of `EXT4_MAX_EXTENT_DEPTH = 5`. Each non-root block is checksummed by a 32-bit `ext4_extent_tail.et_checksum` packed at the block's end. Per-`ext4_extent.ee_len` encodes both run length and *unwritten* state: `ee_len ≤ 0x8000` ⟹ initialized; `ee_len > 0x8000` ⟹ unwritten with actual length `ee_len - 0x8000`. Unwritten extents are how `fallocate(FALLOC_FL_KEEP_SIZE)` and DIO-write-on-allocate avoid synchronous zero-fill. Per-`ext4_ext_map_blocks` is the central map+create entry point. Per-`ext4_ext_truncate` / `ext4_ext_remove_space` / `ext4_ext_punch_hole` walk the tree to free leaf extents in a partial-cluster-aware pass. Per-`ext4_ext_convert_to_initialized` and `ext4_split_extent` perform the unwritten→initialized transition on writeback. Per-`ext4_fiemap` exposes the leaf set to userspace via `iomap_fiemap`. Critical for: large-file IO, sparse files, fallocate, DIO, holes, online resize, FIEMAP.

This Tier-3 covers `fs/ext4/extents.c` (~6308 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ext4_extent` | per-leaf record (lblk, len, pblk) | `Ext4Extent` |
| `struct ext4_extent_idx` | per-index record (lblk, child-pblk) | `Ext4ExtentIdx` |
| `struct ext4_extent_header` | per-block tree header | `Ext4ExtentHeader` |
| `struct ext4_extent_tail` | per-non-root crc32c | `Ext4ExtentTail` |
| `struct ext4_ext_path` | per-search-result path-array | `ExtPath` |
| `struct partial_cluster` | per-remove-space cluster carry | `PartialCluster` |
| `ext4_find_extent()` | per-lookup tree-walk | `Ext::find_extent` |
| `ext4_ext_binsearch_idx()` | per-index binsearch | `Ext::binsearch_idx` |
| `ext4_ext_binsearch()` | per-leaf binsearch | `Ext::binsearch` |
| `ext4_ext_insert_extent()` | per-insert + split | `Ext::insert_extent` |
| `ext4_ext_insert_index()` | per-index insert | `Ext::insert_index` |
| `ext4_ext_split()` | per-node split | `Ext::split` |
| `ext4_ext_grow_indepth()` | per-depth-grow (root → child) | `Ext::grow_indepth` |
| `ext4_ext_create_new_leaf()` | per-leaf-needed alloc + split | `Ext::create_new_leaf` |
| `ext4_ext_correct_indexes()` | per-leftmost-shift index fixup | `Ext::correct_indexes` |
| `ext4_ext_try_to_merge()` | per-adjacency merge | `Ext::try_to_merge` |
| `ext4_ext_try_to_merge_up()` | per-collapse-leaf-into-root | `Ext::try_to_merge_up` |
| `ext4_ext_remove_space()` | per-range free walk | `Ext::remove_space` |
| `ext4_ext_rm_leaf()` | per-leaf removal | `Ext::rm_leaf` |
| `ext4_ext_rm_idx()` | per-index removal | `Ext::rm_idx` |
| `ext4_remove_blocks()` | per-extent partial free | `Ext::remove_blocks` |
| `ext4_ext_truncate()` | per-inode tail-truncate | `Ext::truncate` |
| `ext4_punch_hole()` (inode.c) | per-FALLOC_FL_PUNCH_HOLE | shared with `inode.md` |
| `ext4_ext_map_blocks()` | per-map-or-create | `Ext::map_blocks` |
| `ext4_ext_convert_to_initialized()` | per-unwritten→init | `Ext::convert_to_initialized` |
| `ext4_split_extent()` / `_at()` | per-extent split | `Ext::split_extent`, `Ext::split_extent_at` |
| `ext4_ext_handle_unwritten_extents()` | per-write-into-unwritten dispatch | `Ext::handle_unwritten_extents` |
| `ext4_convert_unwritten_extents_endio()` | per-DIO endio convert | `Ext::convert_unwritten_endio` |
| `ext4_ext_search_left()` / `_right()` | per-neighbour-lookup | `Ext::search_left`, `Ext::search_right` |
| `ext4_ext_check_inode()` | per-mount tree validation | `Ext::check_inode` |
| `ext4_ext_precache()` | per-precache extents into es-cache | `Ext::precache` |
| `ext4_ext_zeroout()` | per-issue-zeroout-IO | `Ext::zeroout` |
| `ext4_fiemap()` | per-FIEMAP via iomap | `Ext::fiemap` |
| `ext4_ext_shift_extents()` | per-collapse/insert range | `Ext::shift_extents` |
| `ext4_swap_extents()` | per-defrag swap | `Ext::swap_extents` |
| `ext4_ext_calc_credits_for_single_extent()` | per-jbd2 credit estimate | `Ext::calc_credits` |
| `EXT4_MAX_EXTENT_DEPTH` (5) | per-tree-depth cap | const |
| `EXT_INIT_MAX_LEN` (0x8000) | per-initialized-extent cap | const |
| `EXT_UNWRITTEN_MAX_LEN` (0x7fff) | per-unwritten-extent cap | const |
| `EXT4_EXT_MAGIC` (0xf30a) | per-header magic | const |

## Compatibility contract

REQ-1: struct ext4_extent (12 bytes, on-disk LE):
- ee_block: u32 — first logical block this extent covers.
- ee_len: u16 — length; if le ≤ 0x8000 initialized, else unwritten with actual_len = le - 0x8000.
- ee_start_hi: u16 — high 16 bits of physical block (≥48-bit pblk).
- ee_start_lo: u32 — low 32 bits of physical block.
- ext4_ext_pblock(ex) := (ee_start_hi << 32) | ee_start_lo.
- ext4_ext_get_actual_len(ex) := if ee_len ≤ EXT_INIT_MAX_LEN then ee_len else (ee_len - EXT_INIT_MAX_LEN).
- ext4_ext_is_unwritten(ex) := ee_len > EXT_INIT_MAX_LEN.
- ext4_ext_mark_unwritten(ex): BUG_ON(actual_len == 0); ee_len |= 0x8000.
- ext4_ext_mark_initialized(ex): ee_len = actual_len.

REQ-2: struct ext4_extent_idx (12 bytes, on-disk LE):
- ei_block: u32 — first logical block covered by subtree.
- ei_leaf_lo: u32 — low 32 bits of child physical block.
- ei_leaf_hi: u16 — high 16 bits of child physical block.
- ei_unused: u16 — must be 0.

REQ-3: struct ext4_extent_header (12 bytes, on-disk LE):
- eh_magic: u16 — must equal EXT4_EXT_MAGIC (0xf30a).
- eh_entries: u16 — current populated entry count; ≤ eh_max.
- eh_max: u16 — capacity in entries (depends on container: inode-root vs block).
- eh_depth: u16 — depth of this node; 0 ⟹ leaf; > 0 ⟹ index node.
- eh_generation: u32 — generation counter, untouched by kernel.
- Non-root blocks have an `ext4_extent_tail.et_checksum = crc32c(uuid ⊕ inum ⊕ gen ⊕ block-body)` at offset EXT4_EXTENT_TAIL_OFFSET(eh).

REQ-4: Inode-resident root layout:
- `EXT4_I(inode).i_data` reused as `ext4_extent_header` followed by up to 4 entries of `ext4_extent` (leaf, depth==0) OR `ext4_extent_idx` (index, depth>0).
- `ext4_ext_space_root(inode)` := 4 (entries available in i_data).
- `ext4_ext_space_root_idx(inode)` := 4 (same, but as indexes).
- `ext4_ext_space_block(inode)` := (sb_blocksize - sizeof(header) - sizeof(tail)) / sizeof(ext4_extent).
- `ext4_ext_space_block_idx(inode)` := same denominator on ext4_extent_idx.

REQ-5: Depth bound — `EXT4_MAX_EXTENT_DEPTH == 5`:
- ext4_find_extent rejects depth > 5 with -EFSCORRUPTED.
- ext4_ext_check_inode validates root.eh_depth at mount-time.
- Tree growth via grow_indepth refuses when current depth == 5.

REQ-6: struct ext4_ext_path:
- p_block: child block ptr (index) or extent pblk (leaf).
- p_depth: current depth at this path level.
- p_maxdepth: array capacity (depth+1 entries).
- p_ext: pointer into leaf's extent array (only set at leaf level).
- p_idx: pointer into index array (only set at index level).
- p_hdr: pointer to the level's ext4_extent_header.
- p_bh: buffer_head pinning the block (NULL at depth 0 = inode-resident root).

REQ-7: ext4_find_extent(inode, block, path, flags):
- depth = ext_depth(inode); if (depth < 0 || depth > EXT4_MAX_EXTENT_DEPTH) ⟹ -EFSCORRUPTED.
- If reusable path passed: drop_refs; reuse if maxdepth ≥ depth.
- Else: kzalloc path[depth+2]; path[0].p_maxdepth = depth+1.
- path[0].p_hdr = ext_inode_hdr(inode); path[0].p_bh = NULL.
- For i = depth down to 1:
  - ext4_ext_binsearch_idx on path[ppos] for `block`.
  - bh = read_extent_tree_block(inode, path[ppos].p_idx, --i, flags). Verifies magic, eh_entries, checksum.
  - ppos++; path[ppos].p_bh = bh; path[ppos].p_hdr = bh's header.
- At ppos == depth: ext4_ext_binsearch on the leaf header.
- Return `path` (errp NULL) or ERR_PTR on failure.
- KUNIT redirect supported.

REQ-8: ext4_ext_binsearch_idx / _binsearch:
- Strict binary search on `block` against ei_block / ee_block.
- Sets p_idx (resp. p_ext) to the rightmost entry whose key ≤ block; NULL if all keys > block.
- CHECK_BINSEARCH (debug) cross-checks with linear scan.

REQ-9: ext4_ext_insert_extent(handle, inode, path, newext, gb_flags):
- /* Locate leaf-level entry in path */
- depth = ext_depth(inode); curp = path + depth.
- /* If room and adjacent, merge into existing */
- if !(gb_flags & EXT4_GET_BLOCKS_SPLIT_NOMERGE):
  - left_neighbour, right_neighbour adjacency tested by ext4_can_extents_be_merged.
  - If mergeable left: existing.ee_len += newext.ee_len; goto merge.
  - If mergeable right: shift right side into newext.
- /* If leaf has space */
- if le_entries < le_max:
  - ext4_ext_get_access(handle, inode, curp).
  - shift entries right of insert position; copy newext.
  - eh_entries++.
  - ext4_ext_correct_indexes for leftmost insert.
  - mark dirty via __ext4_ext_dirty.
- /* No room: split */
- else:
  - if EXT_HAS_FREE_INDEX(parent): ext4_ext_split(handle, inode, gb_flags, path, newext, at).
  - else: ext4_ext_create_new_leaf which may call ext4_ext_grow_indepth.
- /* Post-insert merging */
- if !(gb_flags & EXT4_GET_BLOCKS_SPLIT_NOMERGE): ext4_ext_try_to_merge.

REQ-10: ext4_ext_split(handle, inode, flags, path, newext, at):
- Allocates a new sibling block for each level ≥ `at`.
- For each level, ext4_ext_new_meta_block(inode, EXT4_MB_HINT_DATA cleared) returns metadata pblk.
- Copies right-half of source entries into sibling.
- ext4_ext_insert_index into parent to point at sibling.
- All allocations follow ext4_ext_find_goal (locality near current path).
- Recovers via brelse on partial failure; sets handle->h_aborted on jbd2 fail.

REQ-11: ext4_ext_grow_indepth(handle, inode, flags):
- Tree root in i_block[] is full; need new depth level.
- BUG-precluded if current depth == EXT4_MAX_EXTENT_DEPTH.
- Allocate one block; copy current inode-root contents into it; install single index in inode-root pointing to the new block; ++eh_depth.
- Mark inode dirty.

REQ-12: ext4_ext_create_new_leaf(handle, inode, flags, path, newext):
- Walks `path` upward to find first level with free index slot.
- If found at level k: ext4_ext_split(at=k+1).
- If reached root and root is full: ext4_ext_grow_indepth; rerun find_extent to refresh path.

REQ-13: ext4_ext_try_to_merge(handle, inode, path, ex):
- Merges ex with right neighbour if can_be_merged && pblk-contiguous && same unwritten-flag && actual_len_sum ≤ EXT_INIT_MAX_LEN.
- After merges, if leaf becomes the only one and root has space, ext4_ext_try_to_merge_up collapses leaf back into inode root.

REQ-14: ext4_ext_remove_space(inode, start, end):
- Walk extent tree from rightmost leaf downward; remove [start, end] inclusive.
- Maintains `partial_cluster` carry to free cluster-shared boundaries exactly once.
- Per-leaf calls ext4_ext_rm_leaf which calls ext4_remove_blocks per fully-or-partially-overlapping extent.
- On empty index: ext4_ext_rm_idx frees the index block and shrinks parent.
- On ENOMEM: ext4_ext_trunc_restart_fn allows JBD2 transaction restart.
- Triggers tracepoints trace_ext4_ext_remove_space / _done.

REQ-15: ext4_ext_truncate(handle, inode):
- /* Sync i_disksize before scan */
- EXT4_I(inode).i_disksize = inode.i_size.
- ext4_mark_inode_dirty(handle, inode).
- last_block = (i_size + blocksize - 1) >> blocksize_bits.
- ext4_es_remove_extent(inode, last_block, EXT_MAX_BLOCKS - last_block).
- Loop: ext4_ext_remove_space(inode, last_block, EXT_MAX_BLOCKS - 1). On -ENOMEM ⟹ memalloc_retry_wait + retry.

REQ-16: ext4_punch_hole (inode.c shim) → ext4_ext_remove_space(start_lblk, end_lblk):
- Per-FALLOC_FL_PUNCH_HOLE: must accompany FALLOC_FL_KEEP_SIZE.
- Holds i_rwsem (exclusive) + invalidate_lock; truncates page cache range first; then dio_wait; then ext4_ext_remove_space.

REQ-17: ext4_ext_map_blocks(handle, inode, map, flags):
- /* Lookup */
- path = ext4_find_extent(inode, map.m_lblk, NULL, 0).
- /* Hit */
- if path leaf has ex covering m_lblk:
  - allocated = min(ex.actual_len - (m_lblk - ex.ee_block), map.m_len).
  - if !ext4_ext_is_unwritten(ex) ∨ (flags & EXT4_GET_BLOCKS_UNWRIT_EXT):
    - m_pblk = ext4_ext_pblock(ex) + (m_lblk - ex.ee_block).
    - m_flags = EXT4_MAP_MAPPED (+ EXT4_MAP_UNWRITTEN if unwritten).
    - return allocated.
  - Unwritten + write-path:
    - ext4_ext_handle_unwritten_extents(handle, inode, map, path, flags) ⟹ convert subrange to initialized.
- /* Miss, !CREATE */
- if !(flags & EXT4_GET_BLOCKS_CREATE):
  - determine hole length via ext4_ext_determine_insert_hole; populate es-cache; return 0.
- /* Miss + CREATE: allocate */
- ar.inode = inode; ar.logical = m_lblk; ar.len = map.m_len.
- ar.goal = ext4_ext_find_goal(inode, path, m_lblk).
- ar.flags |= EXT4_MB_HINT_DATA + (UNWRIT_EXT ⟹ ext4_ext_mark_unwritten on newex).
- newblock = ext4_mb_new_blocks(handle, &ar, &err).
- newex.ee_block = m_lblk; newex.ee_len = ar.len; ext4_ext_store_pblock(&newex, newblock).
- if flags & EXT4_GET_BLOCKS_UNWRIT_EXT: ext4_ext_mark_unwritten(&newex).
- ext4_ext_insert_extent(handle, inode, path, &newex, gb_flags) ⟹ on -ENOSPC tree-grow.
- Cache via ext4_es_insert_extent; return allocated.

REQ-18: ext4_ext_convert_to_initialized(handle, inode, map, path, flags):
- /* Cheapest paths first */
- Case A: extent fully within map range ⟹ ext4_ext_mark_initialized + try_to_merge.
- Case B: extent overhangs map on one side ⟹ ext4_split_extent_at to split, then mark first half initialized.
- Case C: extent overhangs both sides ⟹ two splits via ext4_split_extent; mark middle initialized.
- Case D: small extent — zero-out + mark initialized via ext4_split_extent_zeroout to avoid creating index pressure.
- Tracks ext4_io_end_t list for DIO endio dispatch.

REQ-19: ext4_ext_handle_unwritten_extents(handle, inode, map, path, flags):
- /* Dispatch */
- if flags & EXT4_GET_BLOCKS_CONVERT: ext4_ext_convert_to_initialized.
- elif flags & EXT4_GET_BLOCKS_UNWRIT_EXT: return EXT4_MAP_UNWRITTEN (caller will write zeros via DIO).
- elif !(flags & EXT4_GET_BLOCKS_CREATE): return EXT4_MAP_UNWRITTEN with no convert.
- else: full-write-coverage convert + endio defer.

REQ-20: ext4_convert_unwritten_extents_endio(handle, inode, io_end):
- Called from DIO completion workqueue.
- For each io_end_vec: ext4_ext_map_blocks(handle, inode, map, EXT4_GET_BLOCKS_CONVERT_UNWRITTEN | EXT4_GET_BLOCKS_IO_SUBMIT).
- Refresh i_size_write if io extended.

REQ-21: ext4_fiemap(inode, fieinfo, start, len):
- iomap_fiemap dispatching ext4_iomap_xattr_begin / ext4_iomap_begin_report.
- ext4_fiemap_check_ranges clamps to inode size and EXT_MAX_BLOCKS.
- FIEMAP_FLAG_XATTR: separate iomap path for inode-resident xattrs.
- FIEMAP_FLAG_CACHE (ext4_get_es_cache): use extent-status-tree without touching disk.

REQ-22: ext4_ext_check_inode(inode):
- __ext4_ext_check on root header: magic, entries ≤ max, depth ≤ EXT4_MAX_EXTENT_DEPTH.
- ext4_valid_extent_entries: increasing ee_block/ei_block, no overlap, pblk in fs-bounds.
- Called at every iget where i_flags has EXT4_EXTENTS_FL.

REQ-23: ext4_ext_precache(inode):
- Walks the entire extent tree, populating es-cache, useful for FIEMAP/heavy-read inodes.
- Holds i_data_sem in shared mode.

REQ-24: ext4_ext_zeroout(inode, ex):
- sb_issue_zeroout(sb, ext4_ext_pblock(ex), ext4_ext_get_actual_len(ex), GFP_NOFS).
- Used by split_extent_zeroout when extent is "small enough to zero rather than split-and-track".

REQ-25: ext4_ext_calc_credits_for_single_extent / ext4_ext_index_trans_blocks:
- Returns worst-case jbd2 credits for an insert: each level may dirty 1 leaf + 1 index + up to one new metadata block; multiplied by extents needed.

REQ-26: Cluster-aware free (bigalloc):
- partial_cluster.state tracks initial / tofree / nofree across leaf transitions.
- ext4_remove_blocks defers cluster-free until last block of cluster freed (state==tofree).
- ext4_rereserve_cluster restores quota reservation when partial cluster cannot be freed.

REQ-27: Extent-tree shift (collapse_range / insert_range):
- ext4_ext_shift_extents shifts every extent left or right by `shift` blocks.
- Walks rightmost leaf back to root; each shift adjusts ee_block + ei_block.
- ext4_ext_shift_path_extents handles per-leaf shift atomically (single transaction per leaf).
- collapse_range: shift left; insert_range: shift right; both require offset+len cluster-aligned.

REQ-28: ext4_swap_extents:
- Swaps a range of extents between two inodes (used by ext4 defrag ioctl EXT4_IOC_MOVE_EXT).
- Holds both inodes' i_data_sem; preserves extent-status-tree, journal credits per swap step.

## Acceptance Criteria

- [ ] AC-1: ext4_find_extent on depth-5 tree returns leaf path of length 5; depth=6 yields -EFSCORRUPTED.
- [ ] AC-2: Insert into full root leaf with no out-of-line block grows depth via grow_indepth.
- [ ] AC-3: Insert into full leaf with parent slot calls ext4_ext_split + ext4_ext_insert_index.
- [ ] AC-4: Adjacent same-flag extents merge via ext4_ext_try_to_merge; merge respects EXT_INIT_MAX_LEN cap.
- [ ] AC-5: Cross-cluster partial free: partial_cluster=tofree leaves cluster reserved until last block freed.
- [ ] AC-6: ext4_ext_truncate(size=0) frees entire tree; tree reduces to empty inode root.
- [ ] AC-7: ext4_punch_hole(offset, len) in middle of extent splits into two initialized halves and frees blocks between.
- [ ] AC-8: ext4_ext_map_blocks(UNWRIT_EXT) on unmapped lblk creates unwritten extent (ee_len > 0x8000) and returns EXT4_MAP_UNWRITTEN.
- [ ] AC-9: ext4_ext_convert_to_initialized on fully-covered unwritten extent flips MSB without splitting.
- [ ] AC-10: DIO write into unwritten range: convert_unwritten_extents_endio runs from io_end workqueue; ee_len MSB cleared.
- [ ] AC-11: ext4_fiemap returns one record per leaf extent; xattr-only path returns FIEMAP_EXTENT_DATA_INLINE.
- [ ] AC-12: Crc32c on non-root block: corrupting et_checksum makes __ext4_ext_check fail and inode marked corrupted via EXT4_ERROR_INODE.
- [ ] AC-13: ext4_ext_check_inode flags non-monotonic ee_block as EFSCORRUPTED.
- [ ] AC-14: collapse_range with [offset, offset+len] cluster-aligned shifts all extents left by len/blocksize; sparse holes coalesce.
- [ ] AC-15: ext4_swap_extents swaps mapped ranges between two inodes and leaves both consistent on fsck.

## Architecture

```
struct Ext4ExtentHeader {        // 12 bytes, on-disk LE
  eh_magic: u16,                 // EXT4_EXT_MAGIC = 0xf30a
  eh_entries: u16,
  eh_max: u16,
  eh_depth: u16,                 // 0 == leaf; ≤ EXT4_MAX_EXTENT_DEPTH (5)
  eh_generation: u32,
}

struct Ext4Extent {              // leaf (depth==0), 12 bytes
  ee_block: u32,                 // first logical block
  ee_len: u16,                   // ≤ 0x8000 init; > 0x8000 unwritten
  ee_start_hi: u16,
  ee_start_lo: u32,              // pblk = (hi<<32) | lo
}

struct Ext4ExtentIdx {           // index (depth>0), 12 bytes
  ei_block: u32,
  ei_leaf_lo: u32,
  ei_leaf_hi: u16,
  ei_unused: u16,                // MUST be 0
}

struct Ext4ExtentTail {          // non-root only
  et_checksum: u32,              // crc32c(uuid, inum, gen, body)
}

struct ExtPath {
  p_block: ext4_fsblk_t,
  p_depth: u16,
  p_maxdepth: u16,
  p_ext: Option<*Ext4Extent>,
  p_idx: Option<*Ext4ExtentIdx>,
  p_hdr: *Ext4ExtentHeader,
  p_bh: Option<*BufferHead>,     // None at depth 0 (i_data root)
}

struct PartialCluster {
  pclu: ext4_fsblk_t,
  lblk: ext4_lblk_t,
  state: PclState,               // Initial | ToFree | NoFree
}
```

`Ext::find_extent(inode, block, path_in, flags) -> Result<*ExtPath>`:
1. /* Depth validation */
2. depth = ext_depth(inode); if depth > EXT4_MAX_EXTENT_DEPTH ⟹ EFSCORRUPTED.
3. /* Reuse or allocate path */
4. if path_in.is_some() and path_in.maxdepth ≥ depth: drop_refs(); reuse.
5. else: alloc path[depth+2]; path[0].p_maxdepth = depth+1.
6. path[0].p_hdr = ext_inode_hdr(inode); path[0].p_bh = None.
7. /* Descend */
8. for i in (depth..0).rev():
   - Ext::binsearch_idx(path[ppos], block).
   - bh = read_extent_tree_block(inode, path[ppos].p_idx, i-1, flags).
   - verify magic, eh_depth == i-1, eh_entries ≤ eh_max, crc32c.
   - ppos += 1; path[ppos] = {p_bh: Some(bh), p_hdr: bh.header()}.
9. /* Leaf */
10. Ext::binsearch(path[depth], block); set p_block if p_ext.is_some().
11. return path.

`Ext::insert_extent(handle, inode, path, newext, flags) -> Result<()>`:
1. depth = ext_depth(inode); curp = path + depth.
2. /* Try left/right neighbour merge first */
3. if !(flags & EXT4_GET_BLOCKS_SPLIT_NOMERGE) and can_be_merged(left, newext):
   - left.ee_len += newext.actual_len; mark_dirty; return.
4. if !(flags & EXT4_GET_BLOCKS_SPLIT_NOMERGE) and can_be_merged(newext, right):
   - newext.ee_len += right.actual_len; shift array; mark_dirty; return.
5. /* Insert at sorted position */
6. if curp.hdr.entries < curp.hdr.max:
   - get_access(curp); memmove right of slot; copy newext; entries += 1.
   - correct_indexes if leftmost; mark_dirty.
7. /* Split */
8. else:
   - if EXT_HAS_FREE_INDEX(path[depth-1]): Ext::split(handle, inode, flags, path, newext, depth).
   - else: Ext::create_new_leaf(handle, inode, flags, path, newext).
9. /* Post merge */
10. if !(flags & EXT4_GET_BLOCKS_SPLIT_NOMERGE): try_to_merge; try_to_merge_up.

`Ext::map_blocks(handle, inode, map, flags) -> Result<i32>`:
1. /* Cache check */
2. if ext4_es_lookup_extent(inode, m_lblk, ...) hits: return.
3. /* Tree walk */
4. path = Ext::find_extent(inode, m_lblk, None, 0)?.
5. ex = path[depth].p_ext.
6. if ex.is_some() and ex.contains(m_lblk):
   - allocated = min(ex.actual_len - (m_lblk - ex.ee_block), map.m_len).
   - pblk = ext4_ext_pblock(ex) + (m_lblk - ex.ee_block).
   - if !is_unwritten(ex): m_flags = MAPPED; cache; return allocated.
   - else: dispatch to handle_unwritten_extents.
7. /* Miss */
8. if !(flags & EXT4_GET_BLOCKS_CREATE):
   - hole_len = determine_insert_hole(inode, path, m_lblk).
   - es_insert_extent(HOLE); return 0.
9. /* Create */
10. ar = AllocationRequest { inode, logical: m_lblk, len: map.m_len, goal: Ext::find_goal(inode, path, m_lblk), flags: HINT_DATA }.
11. pblk = mb_new_blocks(handle, ar)?; (may downgrade ar.len on partial allocation).
12. newex = Ext4Extent { ee_block: m_lblk, ee_len: ar.len, pblk }.
13. if flags & EXT4_GET_BLOCKS_UNWRIT_EXT: mark_unwritten(&newex).
14. Ext::insert_extent(handle, inode, path, &newex, flags)?.
15. es_insert_extent + return allocated.

`Ext::remove_space(inode, start, end) -> Result<()>`:
1. partial = PartialCluster { state: Initial, pclu: 0, lblk: 0 }.
2. depth = ext_depth(inode); path = find_extent(inode, end, None, 0)?.
3. while depth >= 0:
   - if leaf (path[depth].p_hdr.depth == 0):
     - Ext::rm_leaf(handle, inode, path, depth, start, end, &partial).
   - if path's index node now empty: Ext::rm_idx(handle, inode, path, depth); depth -= 1.
   - if more_to_rm(path): descend leftmost rightward chain.
   - else: depth -= 1.
4. Final partial cluster: if tofree, free pclu; if nofree, rereserve_cluster.

`Ext::convert_to_initialized(handle, inode, map, path, flags) -> Result<i32>`:
1. ex = path[depth].p_ext; actual = ext4_ext_get_actual_len(ex).
2. /* Try cheap mark */
3. if (m_lblk == ex.ee_block) and (m_len ≥ actual):
   - mark_initialized(ex); try_to_merge; return actual.
4. /* Pick strategy */
5. if small-extent: split_extent_zeroout + mark_initialized.
6. else if one-side overhang: split_extent_at; mark_initialized.
7. else: split_extent at two boundaries; mark middle initialized.
8. allocated = bytes converted.

`Ext::fiemap(inode, fieinfo, start, len) -> Result<i32>`:
1. /* Range check + clamp */
2. fiemap_check_ranges(inode, start, &len).
3. /* XATTR path */
4. if fieinfo.fi_flags & FIEMAP_FLAG_XATTR:
   - iomap_fiemap with ext4_iomap_xattr_ops; emit one FIEMAP_EXTENT_DATA_INLINE record.
5. /* CACHE path */
6. else if fieinfo.fi_flags & FIEMAP_FLAG_CACHE:
   - ext4_get_es_cache(inode, fieinfo, start, len).
7. /* Default */
8. else: iomap_fiemap with ext4_iomap_report_ops.

`Ext::shift_extents(inode, handle, start, shift, direction) -> Result<()>`:
1. /* Iterate leaves right-to-left (collapse) or left-to-right (insert) */
2. path = find_extent(inode, start, None, 0)?.
3. for each leaf containing extents in [start, EXT_MAX_BLOCKS]:
   - per-leaf: ext4_ext_shift_path_extents(path, shift, inode, handle, direction).
   - adjust ee_block ± shift; if depth>0: adjust ei_block at parents via correct_indexes.
4. ext4_handle_dirty_metadata per leaf; jbd2 credits checked per iteration.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `header_magic_invariant` | INVARIANT | per-node: eh_magic == EXT4_EXT_MAGIC pre/post any RW. |
| `entries_bounded_by_max` | INVARIANT | per-node: eh_entries ≤ eh_max. |
| `depth_bounded` | INVARIANT | per-tree: eh_depth ≤ EXT4_MAX_EXTENT_DEPTH = 5. |
| `ee_block_monotonic` | INVARIANT | per-leaf: extents sorted strictly increasing by ee_block, no overlap. |
| `ei_block_monotonic` | INVARIANT | per-index: indexes sorted strictly increasing by ei_block. |
| `unwritten_zero_len_excluded` | INVARIANT | per-extent: ee_len == 0x8000 must mean initialized of length 0x8000, never unwritten of length 0. |
| `path_maxdepth_geq_depth` | INVARIANT | per-find_extent: returned path has p_maxdepth ≥ p_depth. |
| `partial_cluster_state_machine` | INVARIANT | per-remove_space: state ∈ {Initial, ToFree, NoFree}; transitions only via remove_blocks. |
| `crc32c_matches_block` | INVARIANT | per-non-root block read: et_checksum == crc32c(uuid ⊕ inum ⊕ gen ⊕ body). |
| `pblk_within_fs` | INVARIANT | per-extent: ext4_ext_pblock(ex) < sbi.s_blocks_count. |

### Layer 2: TLA+

`fs/ext4/extents.tla`:
- Per-tree-state = (depth, entries-per-node, range-coverage).
- Operations: Find, Insert, Split, Grow, Merge, Remove, Convert.
- Properties:
  - `safety_no_overlap` — per-leaf: no two extents share a logical block.
  - `safety_coverage_disjoint` — per-tree: union of leaf ranges equals mapped set, no double-mapping.
  - `safety_depth_bounded` — per-grow: depth never exceeds 5.
  - `safety_unwritten_flag_preserved_on_split` — per-split: pieces inherit unwritten bit of parent.
  - `safety_partial_cluster_freed_once` — per-remove_space: each cluster freed at most once.
  - `liveness_insert_terminates` — per-insert: returns within 5 grows or NOSPC.
  - `liveness_remove_terminates` — per-remove_space: walks finite leaf set and stops.
  - `liveness_convert_terminates` — per-convert: at most 2 splits + 1 mark per call.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ext::find_extent` post: returned path ends at depth-0 leaf OR ERR | `Ext::find_extent` |
| `Ext::binsearch_idx` post: p_idx points to greatest ei_block ≤ block | `Ext::binsearch_idx` |
| `Ext::binsearch` post: p_ext points to extent containing block, or NULL | `Ext::binsearch` |
| `Ext::insert_extent` post: tree's mapped set ∪= newext range; sorted + bounds invariants hold | `Ext::insert_extent` |
| `Ext::split` post: parent.entries += 1; sibling.entries ≥ ⌈parent.entries/2⌉ | `Ext::split` |
| `Ext::grow_indepth` post: root.eh_depth += 1; root.eh_entries == 1 (single index) | `Ext::grow_indepth` |
| `Ext::remove_space` post: mapped set \ [start,end] equals new mapped set | `Ext::remove_space` |
| `Ext::truncate` post: all extents with ee_block ≥ last_block freed | `Ext::truncate` |
| `Ext::convert_to_initialized` post: target subrange has ee_len ≤ EXT_INIT_MAX_LEN | `Ext::convert_to_initialized` |
| `Ext::map_blocks` post: m_pblk == ext4_ext_pblock(ex) + offset for hit; new extent inserted on create | `Ext::map_blocks` |
| `Ext::fiemap` post: emitted records cover all leaf extents in [start, start+len) | `Ext::fiemap` |
| `Ext::swap_extents` post: per-leaf ranges in each inode bidirectionally swapped, sizes preserved | `Ext::swap_extents` |

### Layer 4: Verus/Creusot functional

`Per-tree-lookup → per-leaf-extent → per-insert-or-create → per-merge-and-split → per-unwritten-convert → per-remove-with-partial-cluster → per-fiemap-emit` semantic equivalence: per `Documentation/filesystems/ext4/ondisk/extents.rst` + `Documentation/filesystems/ext4/dynamic.rst`.

## Hardening

(Inherits row-1 features from `fs/ext4/00-overview.md` § Hardening.)

Extent-tree reinforcement:

- **Per-eh_magic check on every read** — defense against per-stale-block or per-fs-corruption traversal into garbage.
- **Per-eh_depth bound (≤ 5) enforced** — defense against per-loop-forever in tree walk.
- **Per-eh_entries ≤ eh_max checked at __ext4_ext_check** — defense against per-OOB write into following block fields.
- **Per-extent_tail crc32c verified on every metadata read** — defense against per-silent-on-disk corruption.
- **Per-strictly-monotonic ee_block / ei_block enforced** — defense against per-double-mapping leading to UAF.
- **Per-pblk in-fs-bounds checked via ext4_valid_extent** — defense against per-OOB physical IO to neighbour fs.
- **Per-EXT4_ERROR_INODE on corruption** — defense against per-continuing-on-bad-tree.
- **Per-partial_cluster state-machine** — defense against per-double-free of bigalloc clusters.
- **Per-EXT_UNWRITTEN_MAX_LEN cap (0x7fff)** — defense against per-overflow when marking unwritten.
- **Per-EXT4_GET_BLOCKS_METADATA_NOFAIL bounded** — defense against per-unbounded retry.
- **Per-handle_t credit check before split/grow** — defense against per-journal-overflow.
- **Per-i_data_sem write-locked across mutations** — defense against per-concurrent-truncate vs writeback race.
- **Per-precache holds shared i_data_sem** — defense against per-mutator-vs-reader on tree walk.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on `FIEMAP` / `FS_IOC_FIEMAP` extent enumeration so a crafted mapping cannot drive an oversized copy_to_user.
- **PAX_KERNEXEC** — W^X for any executable mapping reachable from extent code paths.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization across `ext4_ext_map_blocks` / `ext4_ext_truncate` so crafted images cannot probe stack layout.
- **PAX_REFCOUNT** — saturating refcount on every `buffer_head` of an extent-index block and on `EXT4_I(inode)->i_es_tree` cached extents so cyclic-tree probing cannot wrap.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `ext4_ext_path` arrays and per-depth scratch buffers on every error unwind.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access on FIEMAP / FALLOCATE ioctl surface.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `ext4_ext_truncate` / `ext4_split_unwritten_extents` indirect dispatches.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding for `/proc/fs/ext4/<dev>/extents_status`.
- **GRKERNSEC_DMESG** — syslog restriction on extent-tree corruption diagnostics that otherwise expose physical block numbers.
- **Extent-tree depth bound (5)** — `EXT4_MAX_EXTENT_DEPTH = 5` is enforced in `ext4_ext_check_block`; deeper trees force `-EFSCORRUPTED` so a malicious image cannot blow the kernel stack.
- **CRC32C verify on every extent index block** — `ext4_extent_block_csum_verify` runs before any extent walk; a mismatched tail-csum aborts the I/O.
- **`EXT_UNWRITTEN_MAX_LEN = 0x7fff`** — unwritten-extent length is bounded so the high bit (the "unwritten" tag) cannot be set on a length that aliases into a written extent.
- **Unwritten-extent → written conversion atomicity** — `ext4_convert_unwritten_extents_endio` only flips the length-high-bit under journal commit so a crash never exposes uninitialized blocks as readable.
- **`partial_cluster` state-machine** — bigalloc cluster ownership tracked across truncate splits so neither end of a partial cluster is double-freed.
- **`handle_t` credit check pre-mutation** — `ext4_ext_split` / `ext4_ext_grow_indepth` verify journal credits before any tree mutation, refusing with `-ENOSPC` rather than commit-time abort.
- **`i_data_sem` write-locked across mutations** — concurrent truncate vs writeback cannot observe a half-rewritten tree.

Per-doc rationale: the extent tree is the on-disk representation of every regular-file mapping in ext4; any path that walks it (read, write, fallocate, truncate, FIEMAP) reads attacker-controllable depth/length/start fields from disk. PaX/grsec reinforcement keeps depth bounded, lengths CRC-verified, unwritten flips journal-atomic, and the entire walk under refcount/USERCOPY discipline so a malformed extent tree degrades to `-EFSCORRUPTED`, not arbitrary kernel write.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/ext4/inode.c block-mapped path (legacy indirect) — covered in `fs/ext4/inode.md`
- fs/ext4/extents_status.c es-tree caching — covered separately if expanded
- fs/ext4/mballoc.c block allocator — covered in `fs/ext4/mballoc.md`
- fs/jbd2/* transaction layer — covered in `fs/jbd2/*.md`
- fs/iomap/* generic iomap fiemap framework — covered in `fs/iomap/*.md`
- ext4 online resize (fs/ext4/resize.c) — covered separately if expanded
- ext4 defrag ioctl plumbing (fs/ext4/move_extent.c) — covered separately if expanded
- Implementation code
