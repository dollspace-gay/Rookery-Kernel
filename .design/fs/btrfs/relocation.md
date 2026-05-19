# Tier-3: fs/btrfs/relocation.c — btrfs balance / relocation (block-group + tree-block + data-extent)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/relocation.c (~6122 lines)
  - fs/btrfs/relocation.h
  - fs/btrfs/backref.h (struct btrfs_backref_node / _edge / _cache, BTRFS_MAX_LEVEL, walk helpers)
  - fs/btrfs/volumes.c (btrfs_relocate_chunk — caller wrapper)
  - include/uapi/linux/btrfs_tree.h (BTRFS_TREE_RELOC_OBJECTID = 9, BTRFS_DATA_RELOC_TREE_OBJECTID = -9ULL, BTRFS_BALANCE_ITEM_KEY)
  - include/uapi/linux/btrfs.h (struct btrfs_ioctl_balance_args, BTRFS_BALANCE_DATA, BTRFS_BALANCE_METADATA, BTRFS_BALANCE_SYSTEM, BALANCE_ARGS_*)
-->

## Summary

`fs/btrfs/relocation.c` implements the lowest layer of btrfs **balance / resize / device-replace**: moving every live extent out of a designated **block group** so the block-group can be freed. It is the only correct way to "shrink" a btrfs (`btrfs_relocate_chunk` ← `btrfs_shrink_device` and `btrfs_remove_chunk`), the engine behind `btrfs_balance_ioctl` (per-`btrfs_ioctl_balance_args`, BALANCE_DATA/METADATA/SYSTEM filters), and the metadata mover behind dev-replace.

Per-relocation context: `struct reloc_control` — one per active block-group relocation, hung off `fs_info.reloc_ctl`:

- `block_group`: per-relocation target.
- `extent_root`: per-target extent-tree (extent-tree-v2 may have multiple).
- `data_inode`: per-relocation **data-reloc inode** — a hidden inode in `BTRFS_DATA_RELOC_TREE_OBJECTID = -9ULL` (`fs_info.data_reloc_root`) used as a staging file for data extents being moved.
- `block_rsv`: per-relocation `BTRFS_BLOCK_RSV_TEMP` reservation; sized as `nodesize * RELOCATION_RESERVED_NODES`, refilled per loop iteration.
- `backref_cache`: per-relocation `struct btrfs_backref_cache` — the cache of `btrfs_backref_node` / `btrfs_backref_edge` covering snapshot-CoW shape (which fs-roots point at this extent, through which intermediate tree blocks).
- `cluster`: per-relocation `struct file_extent_cluster` — coalesces nearby data extents into a single read+write batch (`MAX_EXTENTS = 128` boundaries).
- `processed_blocks`: per-relocation `extent_io_tree` marking tree-blocks already relocated (so we don't re-walk them).
- `reloc_root_tree` + `reloc_roots`: per-relocation mapping (`struct mapping_tree { rb_root + spinlock }` of `struct mapping_node`) from each fs-root that needs CoW into its private **tree-reloc tree** (a real `BTRFS_TREE_RELOC_OBJECTID = 9`-typed btree where the rewritten metadata is staged before being merged back into the original fs-root).
- `dirty_subvol_roots`: per-relocation list of subvolume roots that received a swap and need cleanup.
- `merging_rsv_size`, `nodes_relocated`, `reserved_bytes`, `search_start`, `extents_found`: per-relocation accounting.
- `stage`: per-relocation `enum reloc_stage { MOVE_DATA_EXTENTS, UPDATE_DATA_PTRS }`.
- `create_reloc_tree`, `merge_reloc_tree`, `found_file_extent`: per-relocation lifecycle flags.

The relocation algorithm is intrinsically a **two-stage** thing because btrfs is a CoW filesystem with shared metadata (snapshots share tree blocks). To move a metadata block X out of the target block-group, every snapshot that references X must have its parent-pointer rewritten to point at the new location. The implementation:

- **Stage 1 — `MOVE_DATA_EXTENTS` (one pass over the extent tree)**: for every live extent in the block-group, take an `extent-item`:
  - If it is a **tree-block** (`BTRFS_EXTENT_FLAG_TREE_BLOCK`): build its backref tree (`build_backref_tree` — BFS over `btrfs_backref_add_tree_node` from leaf up to all fs-roots that reference it through their shape) and run `relocate_tree_blocks` → `relocate_tree_block` → either `do_relocation` (rewrite parent ptrs in tree-reloc roots) or `relocate_cowonly_block` (single-CoW shortcut for non-snapshotted roots like data-reloc-tree, free-space-tree). Each fs-root involved gets (or already has) its own **tree-reloc root** registered via `__add_reloc_root`; CoWing through that root drops the old block from the target BG and re-allocates outside it.
  - If it is a **data extent** (`BTRFS_EXTENT_FLAG_DATA`): `relocate_data_extent` queues it onto `rc.cluster`. After the extent-walk completes, `relocate_file_extent_cluster` reads each clustered range through the **data-reloc inode** (`btrfs_grab_root(fs_info.data_reloc_root)` → orphan inode created by `create_reloc_inode`), dirties the folios, and writeback re-allocates the data outside the target BG. The data-reloc inode is born with `reloc_block_group_start = bg.start`; ordered-extent completion calls `btrfs_reloc_clone_csums` to copy original csums into the new placement so checksums survive the move byte-identical.

- **Stage 2 — merge tree-reloc trees back (`prepare_to_merge` + `merge_reloc_roots`)**: each tree-reloc root is dropped back into its parent fs-root via `merge_reloc_root` → `replace_path` (key-by-key path swap with the live subvolume tree, taking care to honor active read-only or in-progress snapshots).

- After both stages, `relocate_block_group` ends: extents are now elsewhere; `block_group->used == 0`; `btrfs_remove_chunk` (caller) tears the chunk down.

The **CoW interception** that makes this work is `btrfs_reloc_cow_block` (called from `ctree.c::btrfs_cow_block` on every CoW): if `fs_info.reloc_ctl` is set and we are CoWing a tree-reloc block, attach `cow` as the backref-node's new `eb`, record `new_bytenr`, and place the node on `rc.backref_cache.pending[level]` so the parent-pointer rewrite picks it up. This means **every CoW in the kernel passes through the relocation tap** while a relocation is in flight — the price of correctness for a shared-metadata filesystem.

Snapshot interaction: `btrfs_reloc_pre_snapshot` / `btrfs_reloc_post_snapshot` reserve the extra metadata (twice the size of relocated nodes, worst case — once for CoWing the reloc tree, once for CoWing the fs tree) so creating a snapshot mid-balance does not run out of metadata.

Crash recovery: `btrfs_recover_relocation(fs_info)` is called from `open_ctree` at mount. It scans for every `BTRFS_TREE_RELOC_OBJECTID` root left over from an interrupted balance, attaches them back to their fs-roots, restarts `merge_reloc_roots` and finishes. Tree-reloc roots with `root_refs == 0` are garbage-collected via `mark_garbage_root`.

Cancellation: `btrfs_should_cancel_balance(fs_info)` is checked per-iteration; balance cancel sets `fs_info.balance_cancel_req`, which causes `relocate_block_group` to return `-ECANCELED` and the relocation to unwind cleanly.

Remap-tree path (`should_relocate_using_remap_tree(bg)` ⟺ `btrfs_fs_incompat(REMAP_TREE)` ∧ bg is data/meta-non-system): a newer faster path that bypasses backref-walking by maintaining an out-of-band remap tree (`do_remap_reloc`, `start_block_group_remapping`, `copy_remapped_data*`, `move_existing_remap*`, `mark_chunk_remapped`, `add_remap_entry`, ...). This Tier-3 includes the remap-tree surface but the legacy snapshot-CoW path is the primary contract.

This Tier-3 covers `fs/btrfs/relocation.c` (~6122 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct reloc_control` | per-relocation context | `RelocControl` |
| `struct mapping_node` | per-fs-root → reloc-root map entry | `MappingNode` |
| `struct mapping_tree` | per-rc rbtree-of-mapping-node | `MappingTree` |
| `struct tree_block` | per-tree-block enumeration entry | `TreeBlock` |
| `struct file_extent_cluster` | per-data-extent batch (MAX_EXTENTS = 128) | `FileExtentCluster` |
| `enum reloc_stage` | MOVE_DATA_EXTENTS / UPDATE_DATA_PTRS | `RelocStage` |
| `btrfs_relocate_chunk` (volumes.c) | per-balance caller wrapper | `Volumes::relocate_chunk` |
| `btrfs_relocate_block_group` | per-bg relocation entry | `Reloc::relocate_block_group` |
| `do_nonremap_reloc` | per-bg legacy-path driver | `Reloc::do_nonremap` |
| `do_remap_reloc` | per-bg remap-tree-path driver | `Reloc::do_remap` |
| `do_remap_reloc_trans` | per-remap transaction wrapper | `Reloc::do_remap_trans` |
| `start_block_group_remapping` | per-remap setup | `Reloc::start_remapping` |
| `move_existing_remap` / `move_existing_remaps` | per-remap pre-move | `Reloc::move_existing_remap` / `move_existing_remaps` |
| `create_remap_tree_entries` | per-remap tree-item builder | `Reloc::create_remap_entries` |
| `add_remap_item` / `add_remap_backref_item` / `add_remap_entry` | per-remap item insert | `Reloc::add_remap_item` / `add_remap_backref_item` / `add_remap_entry` |
| `find_next_identity_remap` | per-remap walk | `Reloc::find_next_identity_remap` |
| `remove_chunk_stripes` | per-remap chunk-stripe drop | `Reloc::remove_chunk_stripes` |
| `btrfs_last_identity_remap_gone` | per-remap teardown predicate | `Reloc::last_identity_remap_gone` |
| `mark_chunk_remapped` | per-remap finalize | `Reloc::mark_chunk_remapped` |
| `adjust_identity_remap_count` / `adjust_block_group_remap_bytes` | per-remap accounting | `Reloc::adjust_identity_remap_count` / `adjust_block_group_remap_bytes` |
| `parse_bitmap` | per-remap free-space-tree bitmap walk | `Reloc::parse_bitmap` |
| `copy_remapped_data` / `_io` / `reloc_endio` | per-remap data copy | `Reloc::copy_remapped_data` / `copy_remapped_data_io` / `reloc_endio` |
| `btrfs_translate_remap` | per-bio logical → physical translator | `Reloc::translate_remap` |
| `btrfs_remove_extent_from_remap_tree` | per-extent remap-tree cleanup | `Reloc::remove_extent_from_remap_tree` |
| `insert_remap_item` | per-trans remap-item insert | `Reloc::insert_remap_item` |
| `remove_range_from_remap_tree` | per-trans remap-range delete | `Reloc::remove_range_from_remap_tree` |
| `relocate_block_group` | per-bg legacy main loop | `Reloc::relocate_block_group_inner` |
| `prepare_to_relocate` | per-bg setup (block_rsv + reloc_root tap) | `Reloc::prepare_to_relocate` |
| `set_reloc_control` / `unset_reloc_control` | per-fs install/uninstall reloc_ctl tap | `Reloc::set_control` / `unset_control` |
| `alloc_reloc_control` / `free_reloc_control` | per-rc alloc/free | `Reloc::alloc_control` / `free_control` |
| `reloc_chunk_start` / `reloc_chunk_end` | per-bg cancellable wrapper | `Reloc::chunk_start` / `chunk_end` |
| `describe_relocation` | per-bg startup log | `Reloc::describe` |
| `stage_to_string` | per-enum-stage formatter | `Reloc::stage_to_string` |
| `find_next_extent` | per-bg extent-tree iterator | `Reloc::find_next_extent` |
| `add_tree_block` / `__add_tree_block` | per-extent enqueue (TREE_BLOCK) | `Reloc::add_tree_block` / `add_tree_block_inner` |
| `add_data_references` | per-extent enqueue (DATA refs) | `Reloc::add_data_references` |
| `free_block_list` | per-rb_root teardown | `Reloc::free_block_list` |
| `delete_block_group_cache` | per-bg v1-cache purge | `Reloc::delete_block_group_cache` |
| `delete_v1_space_cache` | per-leaf v1-cache purge | `Reloc::delete_v1_space_cache` |
| `build_backref_tree` | per-tree-block backref-cache BFS | `Reloc::build_backref_tree` |
| `walk_up_backref` / `walk_down_backref` | per-traversal upper/lower walk | `Reloc::walk_up_backref` / `walk_down_backref` |
| `handle_useless_nodes` | per-build cleanup | `Reloc::handle_useless_nodes` |
| `relocate_tree_blocks` | per-rb_root tree-block relocator | `Reloc::relocate_tree_blocks` |
| `relocate_tree_block` | per-block dispatch (shareable vs cowonly) | `Reloc::relocate_tree_block` |
| `relocate_cowonly_block` | per-block single-CoW path | `Reloc::relocate_cowonly_block` |
| `do_relocation` | per-backref-node parent-pointer rewrite | `Reloc::do_relocation` |
| `link_to_upper` | per-pending node tail-link | `Reloc::link_to_upper` |
| `finish_pending_nodes` | per-relocation pending drain | `Reloc::finish_pending_nodes` |
| `select_reloc_root` | per-edge UPPER reloc-root selector | `Reloc::select_reloc_root` |
| `select_one_root` | per-node single-root selector | `Reloc::select_one_root` |
| `record_reloc_root_in_trans` | per-trans reloc-root registration | `Reloc::record_reloc_root_in_trans` |
| `reserve_metadata_space` / `refill_metadata_space` / `calcu_metadata_size` | per-block reservation | `Reloc::reserve_metadata_space` / `refill_metadata_space` / `calcu_metadata_size` |
| `tree_block_processed` / `update_processed_blocks` / `mark_block_processed` | per-block dedup | `Reloc::tree_block_processed` / `update_processed_blocks` / `mark_block_processed` |
| `get_tree_block_key` | per-block first-key lookup | `Reloc::get_tree_block_key` |
| `find_next_key` | per-path next-key walk | `Reloc::find_next_key` |
| `invalidate_extent_cache` | per-root tree-block cache scrub | `Reloc::invalidate_extent_cache` |
| `relocate_data_extent` | per-extent data-reloc-inode preallocate-and-cluster | `Reloc::relocate_data_extent` |
| `relocate_file_extent_cluster` | per-cluster data copy + writeback | `Reloc::relocate_file_extent_cluster` |
| `prealloc_file_extent_cluster` | per-cluster prealloc on data-reloc inode | `Reloc::prealloc_file_extent_cluster` |
| `setup_relocation_extent_mapping` | per-cluster em-tree setup | `Reloc::setup_relocation_extent_mapping` |
| `relocate_one_folio` | per-folio read-modify-dirty | `Reloc::relocate_one_folio` |
| `get_cluster_boundary_end` | per-cluster boundary lookup | `Reloc::get_cluster_boundary_end` |
| `replace_file_extents` | per-CoW data-pointer rewrite | `Reloc::replace_file_extents` |
| `get_new_location` | per-data-extent old→new lookup | `Reloc::get_new_location` |
| `replace_path` | per-merge key-by-key path swap | `Reloc::replace_path` |
| `walk_up_reloc_tree` / `walk_down_reloc_tree` | per-merge tree traversal | `Reloc::walk_up_reloc_tree` / `walk_down_reloc_tree` |
| `memcmp_node_keys` | per-merge key compare | `Reloc::memcmp_node_keys` |
| `__add_reloc_root` / `__del_reloc_root` / `__update_reloc_root` | per-rc reloc-root registry | `Reloc::add_reloc_root` / `del_reloc_root` / `update_reloc_root` |
| `find_reloc_root` | per-bytenr reloc-root lookup | `Reloc::find_reloc_root` |
| `create_reloc_root` | per-fs-root tree-reloc root create | `Reloc::create_reloc_root` |
| `btrfs_init_reloc_root` | per-trans reloc-root init (callback from CoW) | `Reloc::init_reloc_root` |
| `btrfs_update_reloc_root` | per-trans reloc-root drop_progress update | `Reloc::update_reloc_root` |
| `reloc_root_is_dead` / `have_reloc_root` | per-root state predicates | `Reloc::reloc_root_is_dead` / `have_reloc_root` |
| `btrfs_should_ignore_reloc_root` | per-root snapshot-after-swap filter | `Reloc::should_ignore_reloc_root` |
| `merge_reloc_root` | per-root path-swap loop | `Reloc::merge_reloc_root` |
| `prepare_to_merge` / `merge_reloc_roots` / `free_reloc_roots` | per-merge driver | `Reloc::prepare_to_merge` / `merge_reloc_roots` / `free_reloc_roots` |
| `insert_dirty_subvol` / `clean_dirty_subvols` | per-merge dirty-subvol drain | `Reloc::insert_dirty_subvol` / `clean_dirty_subvols` |
| `create_reloc_inode` / `__insert_orphan_inode` / `delete_orphan_inode` | per-rc data-reloc inode lifecycle | `Reloc::create_reloc_inode` / `insert_orphan_inode` / `delete_orphan_inode` |
| `btrfs_reloc_cow_block` | per-CoW reloc tap (ctree.c::btrfs_cow_block hook) | `Reloc::cow_block_hook` |
| `btrfs_reloc_clone_csums` | per-ordered-extent csum-clone | `Reloc::clone_csums` |
| `btrfs_reloc_pre_snapshot` / `btrfs_reloc_post_snapshot` | per-snapshot reserve + migrate | `Reloc::pre_snapshot` / `post_snapshot` |
| `btrfs_should_cancel_balance` | per-iter cancel check | `Reloc::should_cancel_balance` |
| `mark_garbage_root` | per-recover dead-root marker | `Reloc::mark_garbage_root` |
| `btrfs_recover_relocation` | per-mount recovery driver | `Reloc::recover_relocation` |
| `btrfs_get_reloc_bg_bytenr` | per-fs in-flight bg query | `Reloc::get_reloc_bg_bytenr` |
| `RELOCATION_RESERVED_NODES` | per-rc block_rsv sizing constant | `RELOCATION_RESERVED_NODES` |
| `MAX_EXTENTS` (file_extent_cluster) | per-cluster boundary cap = 128 | `FILE_EXTENT_CLUSTER_MAX_EXTENTS` |

## Compatibility contract

REQ-1: struct reloc_control:
- block_group: per-rc target block-group pointer (refcounted).
- extent_root: per-rc extent-tree root (extent-tree-v2 may shard).
- data_inode: per-rc data-reloc inode (NULL on remap-tree path; iget'd from `fs_info.data_reloc_root`).
- block_rsv: per-rc temp reservation (`BTRFS_BLOCK_RSV_TEMP`, sized `nodesize * RELOCATION_RESERVED_NODES`).
- backref_cache: per-rc backref node/edge cache (`is_reloc = true`).
- cluster: per-rc `file_extent_cluster { start, end, boundary[MAX_EXTENTS], nr, owning_root }`.
- processed_blocks: per-rc `extent_io_tree` of already-relocated tree-blocks.
- reloc_root_tree: per-rc `mapping_tree` (rbtree + spinlock) mapping fs-tree-root.start → reloc-root.
- reloc_roots: per-rc list of active reloc-roots.
- dirty_subvol_roots: per-rc list of subvols awaiting cleanup post-merge.
- merging_rsv_size / nodes_relocated / reserved_bytes / search_start / extents_found: per-rc accounting.
- stage: per-rc {MOVE_DATA_EXTENTS, UPDATE_DATA_PTRS}.
- create_reloc_tree / merge_reloc_tree / found_file_extent: per-rc lifecycle bools.

REQ-2: btrfs_relocate_block_group(fs_info, group_start, verbose):
- /* Caller (btrfs_relocate_chunk in volumes.c) holds chunk lock. */
- extent_root = btrfs_extent_root(fs_info, group_start).
- if !extent_root: return -EUCLEAN.
- wait_on_bit(fs_info.flags, BTRFS_FS_UNFINISHED_DROPS, TASK_INTERRUPTIBLE).  /* half-deleted snapshot cleanup */
- if btrfs_fs_closing(fs_info): return -EINTR.
- bg = btrfs_lookup_block_group(fs_info, group_start).
- if !bg: return -ENOENT.
- if bg.flags & BTRFS_BLOCK_GROUP_DATA: ASSERT(sb_write_started(fs_info.sb)).  /* prevent freeze-deadlock */
- if btrfs_pinned_by_swapfile(fs_info, bg): return -ETXTBSY.
- rc = alloc_reloc_control(fs_info).
- reloc_chunk_start(fs_info).  /* test_and_set BTRFS_FS_RELOC_RUNNING; honor reloc_cancel_req */
- rc.extent_root = extent_root; rc.block_group = bg.
- btrfs_inc_block_group_ro(bg, true).  /* freeze allocator */
- inode = lookup_free_space_inode(bg, path).  /* purge v1 cache for this bg */
- if !IS_ERR(inode): delete_block_group_cache(bg, inode, 0).
- if !btrfs_fs_incompat(fs_info, REMAP_TREE): rc.data_inode = create_reloc_inode(bg).
- btrfs_wait_block_group_reservations(bg).
- btrfs_wait_nocow_writers(bg).
- btrfs_wait_ordered_roots(fs_info, U64_MAX, bg).
- btrfs_zone_finish(bg).  /* zoned-allocator finalize */
- if should_relocate_using_remap_tree(bg):
  - if bg.remap_bytes != 0: move_existing_remaps(fs_info, bg, path).
  - start_block_group_remapping(fs_info, path, bg).
  - do_remap_reloc(fs_info, path, bg).
  - btrfs_delete_unused_bgs(fs_info).
- else: do_nonremap_reloc(fs_info, verbose, rc).  /* → relocate_block_group(rc) */
- /* unwind: btrfs_dec_block_group_ro on error, iput data_inode, free path, reloc_chunk_end, btrfs_put_block_group, free_reloc_control */

REQ-3: relocate_block_group(rc) (legacy main loop):
- prepare_to_relocate(rc):
  - rc.block_rsv = btrfs_alloc_block_rsv(fs_info, BTRFS_BLOCK_RSV_TEMP).
  - rc.block_rsv.size = nodesize * RELOCATION_RESERVED_NODES.
  - btrfs_block_rsv_refill(fs_info, rc.block_rsv, rc.block_rsv.size, BTRFS_RESERVE_FLUSH_ALL).
  - rc.create_reloc_tree = true.
  - set_reloc_control(rc) — fs_info.reloc_ctl = rc (CoW tap activated).
  - trans = btrfs_join_transaction(rc.extent_root); btrfs_commit_transaction(trans).
- loop:
  - btrfs_block_rsv_refill(...).
  - trans = btrfs_start_transaction(rc.extent_root, 0).
  - if rc.backref_cache.last_trans != trans.transid: btrfs_backref_release_cache(&rc.backref_cache).
  - rc.backref_cache.last_trans = trans.transid.
  - ret = find_next_extent(rc, path, &key).
  - if ret != 0: break.
  - ei = btrfs_item_ptr(path.nodes[0], ..., struct btrfs_extent_item).
  - flags = btrfs_extent_flags(path.nodes[0], ei).
  - /* simple-quotas: stash owning_root on data-reloc subvol */
  - if btrfs_qgroup_mode(fs_info) == BTRFS_QGROUP_MODE_SIMPLE:
    - BTRFS_I(rc.data_inode).root.relocation_src_root = btrfs_get_extent_owner_root(...).
  - if flags & BTRFS_EXTENT_FLAG_TREE_BLOCK: add_tree_block(rc, &key, path, &blocks).
  - else if rc.stage == UPDATE_DATA_PTRS ∧ flags & BTRFS_EXTENT_FLAG_DATA: add_data_references(rc, &key, path, &blocks).
  - if blocks non-empty: relocate_tree_blocks(trans, rc, &blocks); on -EAGAIN: rc.extents_found--; rc.search_start = key.objectid (retry).
  - btrfs_end_transaction_throttle(trans).
  - btrfs_btree_balance_dirty(fs_info).
  - if rc.stage == MOVE_DATA_EXTENTS ∧ flags & BTRFS_EXTENT_FLAG_DATA: rc.found_file_extent = true; relocate_data_extent(rc, &key).
  - if btrfs_should_cancel_balance(fs_info): err = -ECANCELED; break.
- on -ENOSPC with progress: btrfs_force_chunk_alloc; restart.
- if !err ∧ !REMAP_TREE: relocate_file_extent_cluster(rc).  /* flush trailing data-extent cluster */
- rc.create_reloc_tree = false; set_reloc_control(rc).
- btrfs_backref_release_cache(&rc.backref_cache).
- /* Stage 2: merge */
- err = prepare_to_merge(rc, err).
- merge_reloc_roots(rc).
- rc.merge_reloc_tree = false; unset_reloc_control(rc).
- btrfs_commit_current_transaction(rc.extent_root).
- clean_dirty_subvols(rc).
- btrfs_free_block_rsv(fs_info, rc.block_rsv).

REQ-4: build_backref_tree(trans, rc, *node_key, level, bytenr) → backref_node*:
- iter = btrfs_backref_iter_alloc.
- path = btrfs_alloc_path.
- node = btrfs_backref_alloc_node(cache, bytenr, level).
- cur = node.
- /* BFS over backrefs */
- do:
  - btrfs_backref_add_tree_node(trans, cache, path, iter, node_key, cur).
  - edge = pop first cache.pending_edge entry.
  - if edge: cur = edge.node[UPPER].
- while edge.
- btrfs_backref_finish_upper_links(cache, node).
- if handle_useless_nodes(rc, node): node = NULL  /* all references resolved to detached/processed → skip */.
- ASSERT(!node || !node.detached).
- ASSERT(list_empty(cache.useless_node) ∧ list_empty(cache.pending_edge)).
- return node.

REQ-5: relocate_tree_blocks(trans, rc, *blocks) — per-rb_root tree-block driver:
- /* Readahead pass */
- rbtree_postorder_for_each_entry_safe(blocks, block): if !block.key_ready: btrfs_readahead_tree_block(fs_info, block.bytenr, block.owner, 0, block.level, NULL).
- /* Key-resolution pass */
- rbtree_postorder_for_each_entry_safe(blocks, block): if !block.key_ready: get_tree_block_key(fs_info, block).
- /* Tree-relocation pass */
- rbtree_postorder_for_each_entry_safe(blocks, block):
  - if block.owner ∧ (!btrfs_is_fstree(block.owner) ∨ block.owner == BTRFS_DATA_RELOC_TREE_OBJECTID):
    - /* cowonly tree → no snapshot sharing; one CoW does it */
    - relocate_cowonly_block(trans, rc, block, path); continue.
  - node = build_backref_tree(trans, rc, &block.key, block.level, block.bytenr).
  - if IS_ERR(node): err = PTR_ERR(node); goto out.
  - relocate_tree_block(trans, rc, node, &block.key, path).
- out: finish_pending_nodes(trans, rc, path, ret).
- free_block_list(blocks).

REQ-6: relocate_tree_block(trans, rc, *node, *key, *path):
- reserve_metadata_space(trans, rc, node).
- ASSERT(!node.processed).
- root = select_one_root(node).
- if root:
  - ASSERT(root has BTRFS_ROOT_SHAREABLE) /* else -EUCLEAN: corrupt non-shareable owner */.
  - ASSERT(node.new_bytenr == 0).
  - btrfs_record_root_in_trans(trans, root).
  - if !root.reloc_root: return -ENOENT.
  - root = root.reloc_root.
  - node.new_bytenr = root.node.start.
  - btrfs_put_root(node.root); node.root = btrfs_grab_root(root).
  - update_processed_blocks(rc, node).
- else:
  - /* multi-root: run full do_relocation pass */
  - do_relocation(trans, rc, node, key, path, /*lowest=*/1).
- if ret ∨ node.level == 0: btrfs_backref_cleanup_node(&rc.backref_cache, node).

REQ-7: do_relocation(trans, rc, *node, *key, *path, lowest) — parent-pointer rewrite:
- ASSERT(!lowest ∨ !node.eb).
- path.lowest_level = node.level + 1.
- rc.backref_cache.path[node.level] = node.  /* btrfs_reloc_cow_block keys off this */
- for edge in node.upper:
  - upper = edge.node[UPPER].
  - root = select_reloc_root(trans, rc, upper, edges).
  - /* Resolve upper.eb via path-search-or-cached */
  - if upper.eb ∧ !upper.locked: btrfs_backref_drop_node_buffer(upper).
  - if !upper.eb: btrfs_search_slot(trans, root, key, path, 0, 1); upper.eb = path.nodes[upper.level]; upper.locked = 1.
  - else: btrfs_bin_search(upper.eb, 0, key, &slot).
  - bytenr = btrfs_node_blockptr(upper.eb, slot).
  - /* Lowest node sanity check */
  - if lowest ∧ bytenr != node.bytenr: -EIO.
  - if !lowest ∧ node.eb.start == bytenr: goto next  /* already done */.
  - eb = btrfs_read_node_slot(upper.eb, slot); btrfs_tree_lock(eb).
  - if !node.eb:
    - /* CoW this block — btrfs_reloc_cow_block tap will record new_bytenr */
    - btrfs_cow_block(trans, root, eb, upper.eb, slot, &eb, BTRFS_NESTING_COW).
    - ASSERT(node.eb == eb)  /* tap set it */.
  - else:
    - /* Already CoWed: just rewrite the upper's slot to point at node.eb.start */
    - btrfs_set_node_blockptr(upper.eb, slot, node.eb.start).
    - btrfs_set_node_ptr_generation(upper.eb, slot, trans.transid).
    - btrfs_mark_buffer_dirty(trans, upper.eb).
    - /* Re-account the delayed-ref bookkeeping */
    - btrfs_inc_extent_ref(trans, &ref).
    - btrfs_drop_subtree(trans, root, eb, upper.eb).
  - if ret: btrfs_abort_transaction.
- if !ret ∧ node.pending: btrfs_backref_drop_node_buffer(node); list_del_init(&node.list); node.pending = 0.
- ASSERT(ret != -ENOSPC)  /* must have been reserved up-front */.

REQ-8: btrfs_reloc_cow_block(trans, root, *buf, *cow) — the CoW tap:
- rc = fs_info.reloc_ctl; if !rc: return 0  /* not in relocation */.
- BUG_ON(rc.stage == UPDATE_DATA_PTRS ∧ btrfs_is_data_reloc_root(root)).
- level = btrfs_header_level(buf).
- first_cow = btrfs_header_generation(buf) ≤ btrfs_root_last_snapshot(&root.root_item).
- if btrfs_root_id(root) == BTRFS_TREE_RELOC_OBJECTID ∧ rc.create_reloc_tree:
  - WARN_ON(!first_cow ∧ level == 0).
  - node = rc.backref_cache.path[level].
  - /* Validate cache coherency */
  - if node.bytenr != buf.start ∧ node.new_bytenr != buf.start: return -EUCLEAN.
  - btrfs_backref_drop_node_buffer(node); refcount_inc(&cow.refs); node.eb = cow; node.new_bytenr = cow.start.
  - if !node.pending: list_move_tail(&node.list, &rc.backref_cache.pending[level]); node.pending = 1.
  - if first_cow: mark_block_processed(rc, node).
  - if first_cow ∧ level > 0: rc.nodes_relocated += buf.len.
- if level == 0 ∧ first_cow ∧ rc.stage == UPDATE_DATA_PTRS:
  - replace_file_extents(trans, rc, root, cow)  /* rewrite EXTENT_DATA item disk_bytenr fields */.

REQ-9: data-reloc inode (`create_reloc_inode`):
- root = btrfs_grab_root(fs_info.data_reloc_root)  /* BTRFS_DATA_RELOC_TREE_OBJECTID = -9ULL */.
- trans = btrfs_start_transaction(root, 6).
- btrfs_get_free_objectid(root, &objectid).
- __insert_orphan_inode(trans, root, objectid).
- inode = btrfs_iget(objectid, root).
- inode.reloc_block_group_start = bg.start  /* used by replace_file_extents to translate logical → BG-relative */.
- btrfs_orphan_add(trans, inode)  /* survives crash; cleaned by btrfs_orphan_cleanup at mount */.

REQ-10: relocate_data_extent(rc, *key):
- /* Append [key.objectid, key.objectid+key.offset) to rc.cluster.boundary[]; flush when nr == MAX_EXTENTS or non-contiguous */
- if rc.cluster.nr == MAX_EXTENTS: relocate_file_extent_cluster(rc); rc.cluster reset.
- /* Else extend cluster */

REQ-11: relocate_file_extent_cluster(rc):
- prealloc_file_extent_cluster(rc) — prealloc data-reloc inode extents covering [cluster.start, cluster.end].
- setup_relocation_extent_mapping(rc) — extent-map-tree entries pinning logical → original-physical mapping.
- For each folio in [cluster.start - reloc_block_group_start, cluster.end - reloc_block_group_start]:
  - relocate_one_folio: btrfs_do_readpage (read original disk bytes through em), mark folio dirty.
- writeback fires through the normal data-IO path → ordered-extent → btrfs_reloc_clone_csums copies original csums.
- New disk location is allocated outside the target BG (BG is RO).

REQ-12: btrfs_reloc_clone_csums(ordered):
- /* Called from ordered-extent finalization when writing through data-reloc inode */
- Look up original-extent csums in the csum tree keyed by the cluster's original logical address; insert them at the new disk_bytenr.

REQ-13: tree-reloc roots (`__add_reloc_root`, `create_reloc_root`):
- create_reloc_root(trans, root, root_id): build a clone of `root.node` under BTRFS_TREE_RELOC_OBJECTID with root_key.offset = root_id.
- __add_reloc_root: insert (root.node.start → reloc_root) into rc.reloc_root_tree (rbtree of mapping_node) + list_add to rc.reloc_roots.
- __del_reloc_root: reverse on cleanup.
- find_reloc_root(fs_info, bytenr): rbtree lookup by node.start.
- btrfs_init_reloc_root(trans, root): called from `btrfs_record_root_in_trans` the first time a subvol is touched during relocation; allocates the per-subvol tree-reloc root.
- btrfs_update_reloc_root: per-transaction drop_progress update; if reloc_root has been emptied, clear BTRFS_ROOT_DEAD_RELOC_TREE.
- reloc_root_is_dead / have_reloc_root: state predicates with smp_rmb pairing.

REQ-14: merge_reloc_roots(rc) — Stage 2:
- For each reloc_root in rc.reloc_roots:
  - merge_reloc_root(rc, reloc_root):
    - Repeatedly btrfs_search_slot the live subvol tree and the reloc_root, calling replace_path to swap each level's blockptr atomically under a single transaction.
- free_reloc_roots: drop list.
- insert_dirty_subvol / clean_dirty_subvols: per-modified subvol, reset state + drop reloc_root reference.

REQ-15: prepare_to_merge(rc, err):
- /* Even on error path, merge so that reloc_roots are not left orphaned */
- For each rc.reloc_roots: if err: mark orphan + queue for cleanup; else: increment root_refs.
- Reservations adjusted for the merge phase (rc.merging_rsv_size += nodes_relocated).

REQ-16: btrfs_reloc_pre_snapshot(pending, *bytes_to_reserve):
- rc = root.fs_info.reloc_ctl. if !rc ∨ !have_reloc_root(root) ∨ !rc.merge_reloc_tree: return.
- /* Snapshot during merge phase doubles the worst-case metadata footprint */
- *bytes_to_reserve += rc.nodes_relocated.

REQ-17: btrfs_reloc_post_snapshot(trans, pending):
- /* After snapshot creation, migrate reservation and clone reloc-root onto the new snapshot */
- rc.merging_rsv_size += rc.nodes_relocated.
- if rc.merge_reloc_tree: btrfs_block_rsv_migrate(&pending.block_rsv, rc.block_rsv, rc.nodes_relocated, true).
- reloc_root = create_reloc_root(trans, root.reloc_root, btrfs_root_id(new_root)).
- __add_reloc_root(reloc_root).
- new_root.reloc_root = btrfs_grab_root(reloc_root).

REQ-18: btrfs_recover_relocation(fs_info):
- /* Mount-time recovery for interrupted balance */
- Scan tree_root for {objectid=BTRFS_TREE_RELOC_OBJECTID, type=BTRFS_ROOT_ITEM_KEY, offset=*}.
- For each found reloc_root: btrfs_read_tree_root; set BTRFS_ROOT_SHAREABLE; collect on reloc_roots list.
- For each: resolve its parent fs_root (key.offset). If parent missing → mark_garbage_root. Else hold a ref.
- alloc_reloc_control; rc.extent_root = btrfs_extent_root(fs_info, 0); rc.merge_reloc_tree = true.
- For each (reloc_root, fs_root): __add_reloc_root + fs_root.reloc_root = btrfs_grab_root(reloc_root).
- merge_reloc_roots(rc).
- commit.
- clean_dirty_subvols.
- After all: btrfs_orphan_cleanup(fs_info.data_reloc_root) (drop the staging-inode orphans left by interrupted MOVE_DATA_EXTENTS).

REQ-19: reloc_chunk_start / _end:
- chunk_start: test_and_set BTRFS_FS_RELOC_RUNNING (only one reloc at a time). If reloc_cancel_req > 0: return -ECANCELED.
- chunk_end: ASSERT(BTRFS_FS_RELOC_RUNNING); clear-and-wake; reset reloc_cancel_req.

REQ-20: REMAP_TREE fast path (`should_relocate_using_remap_tree`):
- Predicate: btrfs_fs_incompat(REMAP_TREE) ∧ !(bg.flags & (BTRFS_BLOCK_GROUP_SYSTEM | BTRFS_BLOCK_GROUP_METADATA_REMAP)).
- do_remap_reloc / start_block_group_remapping / do_remap_reloc_trans: build a per-fs remap-tree storing (old logical → new logical) so that subsequent metadata reads transparently translate via btrfs_translate_remap.
- copy_remapped_data + copy_remapped_data_io + reloc_endio: per-extent direct copy bypassing the data-reloc inode.
- add_remap_item / add_remap_backref_item / add_remap_entry / find_next_identity_remap / move_existing_remap(s) / adjust_identity_remap_count / adjust_block_group_remap_bytes: per-remap-tree mutators.
- mark_chunk_remapped + remove_chunk_stripes + btrfs_last_identity_remap_gone: per-remap finalize.
- btrfs_translate_remap(fs_info, *logical, *length): per-IO translator used downstream of relocation.

## Acceptance Criteria

- [ ] AC-1: `btrfs_relocate_block_group(fs_info, bg.start, true)` on an empty bg → returns 0; `bg.used == 0`; `BTRFS_FS_RELOC_RUNNING` cleared.
- [ ] AC-2: `btrfs_relocate_block_group` on a bg with N live tree-blocks → exactly N tree-blocks processed (`rc.extents_found >= N`); none of the original tree-block bytenrs remain inside `[bg.start, bg.start + bg.length)`.
- [ ] AC-3: `btrfs_relocate_block_group` on a bg with M live data-extents → all M data-extents copied through data-reloc inode (or remap-tree when REMAP_TREE); csums cloned; new disk_bytenr ∉ [bg.start, bg.start + bg.length).
- [ ] AC-4: `reloc_chunk_start` invoked twice without intervening `reloc_chunk_end` → second call returns -EINPROGRESS.
- [ ] AC-5: `btrfs_should_cancel_balance` returns true during the main loop → `relocate_block_group` returns -ECANCELED; rc + reloc-roots cleanly torn down via `prepare_to_merge(rc, -ECANCELED) + merge_reloc_roots`.
- [ ] AC-6: `btrfs_reloc_cow_block` invoked during a non-relocation transaction (`fs_info.reloc_ctl == NULL`) → returns 0 immediately (no-op tap).
- [ ] AC-7: `btrfs_reloc_cow_block` invoked on a tree-reloc root during stage MOVE_DATA_EXTENTS with cache mismatch (`node.bytenr != buf.start ∧ node.new_bytenr != buf.start`) → returns -EUCLEAN; log "bytenr ... was found but our backref cache was expecting ...".
- [ ] AC-8: `build_backref_tree(trans, rc, key, level, bytenr)` for an extent referenced by K snapshots → returns a backref_node whose upper-edge closure reaches exactly K fs-tree roots.
- [ ] AC-9: `relocate_tree_block` on a non-shareable owner → returns -EUCLEAN ("bytenr ... resolved to a non-shareable root").
- [ ] AC-10: `select_one_root(node)` for a node that is the tree-root of its owning fs-root (only one root references it directly) → returns the fs-root; node.new_bytenr set to `root.reloc_root.node.start`.
- [ ] AC-11: `select_one_root(node)` for a multi-referenced node → returns NULL → caller (`relocate_tree_block`) runs full `do_relocation`.
- [ ] AC-12: snapshot creation during MOVE_DATA_EXTENTS → `btrfs_reloc_post_snapshot` registers a new reloc_root for the snapshot and links `new_root.reloc_root`; reservation migrated.
- [ ] AC-13: mount after `relocate_block_group` crashed mid-merge → `btrfs_recover_relocation` finds the leftover `BTRFS_TREE_RELOC_OBJECTID` roots, resumes `merge_reloc_roots`, and `btrfs_orphan_cleanup(fs_info.data_reloc_root)` drains staging inodes.
- [ ] AC-14: `relocate_block_group` returns -ENOSPC after `progress > 0` → caller force-allocates a new chunk (`btrfs_force_chunk_alloc`), resets progress, restart.
- [ ] AC-15: `should_relocate_using_remap_tree(bg)` true → `do_remap_reloc` path taken; `rc.data_inode` not created; `btrfs_delete_unused_bgs` invoked post-remap.
- [ ] AC-16: `btrfs_reloc_clone_csums(ordered)` on a relocation ordered-extent → original csums looked up at `ordered.original_offset` and inserted at the new disk_bytenr.

## Architecture

```
struct RelocControl {
  block_group: *BlockGroup,
  extent_root: *BtrfsRoot,
  data_inode: Option<*Inode>,                   // None on REMAP_TREE path
  block_rsv: *BlockRsv,
  backref_cache: BtrfsBackrefCache,             // is_reloc = true
  cluster: FileExtentCluster,                   // boundary[MAX_EXTENTS = 128]
  processed_blocks: ExtentIoTree,
  reloc_root_tree: MappingTree,                 // RbRoot<MappingNode> + spinlock
  reloc_roots: ListHead,
  dirty_subvol_roots: ListHead,
  merging_rsv_size: u64,
  nodes_relocated: u64,
  reserved_bytes: u64,
  search_start: u64,
  extents_found: u64,
  stage: RelocStage,                            // MoveDataExtents | UpdateDataPtrs
  create_reloc_tree: bool,
  merge_reloc_tree: bool,
  found_file_extent: bool,
}

struct MappingNode { rb: RbNode, bytenr: u64, data: *() }
struct MappingTree { rb_root: RbRoot, lock: SpinLock<()> }

struct TreeBlock { rb: RbNode, bytenr: u64, owner: u64, key: BtrfsKey, level: u8, key_ready: bool }

struct FileExtentCluster {
  start: u64,
  end: u64,
  boundary: [u64; MAX_EXTENTS],                 // MAX_EXTENTS = 128
  nr: u32,
  owning_root: u64,
}

enum RelocStage { MoveDataExtents, UpdateDataPtrs }
```

`Reloc::relocate_block_group(fs_info, group_start, verbose) -> Result<()>`:
1. extent_root = btrfs_extent_root(fs_info, group_start).
2. wait_on_bit(fs_info.flags, BTRFS_FS_UNFINISHED_DROPS).
3. if btrfs_fs_closing(fs_info): return Err(Eintr).
4. bg = btrfs_lookup_block_group(fs_info, group_start)?
5. if bg.flags & BLOCK_GROUP_DATA: assert(sb_write_started(fs_info.sb)).
6. if btrfs_pinned_by_swapfile(fs_info, bg): return Err(Etxtbsy).
7. rc = Reloc::alloc_control(fs_info)?
8. Reloc::chunk_start(fs_info)?
9. rc.extent_root = extent_root; rc.block_group = bg.
10. btrfs_inc_block_group_ro(bg, true)?
11. inode = lookup_free_space_inode(bg, path); if Ok: delete_block_group_cache(bg, inode, 0).
12. if !btrfs_fs_incompat(REMAP_TREE): rc.data_inode = Some(Reloc::create_reloc_inode(bg)?).
13. btrfs_wait_block_group_reservations(bg).
14. btrfs_wait_nocow_writers(bg).
15. btrfs_wait_ordered_roots(fs_info, U64_MAX, bg).
16. btrfs_zone_finish(bg).
17. if should_relocate_using_remap_tree(bg):
    - if bg.remap_bytes != 0: move_existing_remaps(fs_info, bg, path)?
    - start_block_group_remapping(fs_info, path, bg)?
    - do_remap_reloc(fs_info, path, bg)?
    - btrfs_delete_unused_bgs(fs_info).
18. else: Reloc::do_nonremap(fs_info, verbose, rc).
19. unwind: btrfs_dec_block_group_ro on error; iput(rc.data_inode); reloc_chunk_end; btrfs_put_block_group; free_control.

`Reloc::relocate_block_group_inner(rc) -> Result<()>` (legacy main loop):
1. Reloc::prepare_to_relocate(rc)?
2. loop:
   - rc.reserved_bytes = 0.
   - btrfs_block_rsv_refill(fs_info, rc.block_rsv, rc.block_rsv.size, FLUSH_ALL)?
   - trans = btrfs_start_transaction(rc.extent_root, 0)?
3. restart:
   - if rc.backref_cache.last_trans != trans.transid: btrfs_backref_release_cache(rc.backref_cache).
   - rc.backref_cache.last_trans = trans.transid.
   - if Reloc::find_next_extent(rc, path, &key) != 0: break.
   - rc.extents_found += 1.
   - ei = btrfs_item_ptr(...).
   - flags = btrfs_extent_flags(ei).
   - if flags & EXTENT_FLAG_TREE_BLOCK: Reloc::add_tree_block(rc, &key, path, &blocks).
   - else if rc.stage == UpdateDataPtrs ∧ flags & EXTENT_FLAG_DATA: Reloc::add_data_references(rc, &key, path, &blocks).
   - if !blocks.is_empty(): Reloc::relocate_tree_blocks(trans, rc, &blocks)?  /* -EAGAIN: retry from key.objectid */.
   - btrfs_end_transaction_throttle(trans).
   - btrfs_btree_balance_dirty(fs_info).
   - if rc.stage == MoveDataExtents ∧ flags & EXTENT_FLAG_DATA:
     - rc.found_file_extent = true.
     - Reloc::relocate_data_extent(rc, &key)?
   - if btrfs_should_cancel_balance(fs_info): err = ECANCELED; break.
4. on -ENOSPC with progress > 0: btrfs_force_chunk_alloc(trans, bg.flags); restart.
5. if !err ∧ !REMAP_TREE: Reloc::relocate_file_extent_cluster(rc)?
6. rc.create_reloc_tree = false; Reloc::set_control(rc).
7. btrfs_backref_release_cache(rc.backref_cache).
8. err = Reloc::prepare_to_merge(rc, err).
9. Reloc::merge_reloc_roots(rc).
10. rc.merge_reloc_tree = false; Reloc::unset_control(rc).
11. btrfs_commit_current_transaction(rc.extent_root).
12. Reloc::clean_dirty_subvols(rc).
13. btrfs_free_block_rsv(fs_info, rc.block_rsv).

`Reloc::build_backref_tree(trans, rc, *node_key, level, bytenr) -> Result<*BackrefNode>`:
1. iter = btrfs_backref_iter_alloc(fs_info).
2. path = btrfs_alloc_path().
3. node = btrfs_backref_alloc_node(cache, bytenr, level).
4. cur = node.
5. loop:
   - btrfs_backref_add_tree_node(trans, cache, path, iter, node_key, cur)?
   - edge = list_first_entry_or_null(cache.pending_edge).
   - if edge.is_none(): break.
   - list_del_init(&edge.list[UPPER]).
   - cur = edge.node[UPPER].
6. btrfs_backref_finish_upper_links(cache, node)?
7. if Reloc::handle_useless_nodes(rc, node): node = NULL.
8. assert(cache.useless_node.is_empty() ∧ cache.pending_edge.is_empty()).
9. return node.

`Reloc::relocate_tree_blocks(trans, rc, *blocks) -> Result<()>`:
1. /* Readahead */
2. for block in rbtree_postorder(blocks): if !block.key_ready: btrfs_readahead_tree_block(fs_info, block.bytenr, block.owner, 0, block.level, NULL).
3. /* Key resolve */
4. for block in rbtree_postorder(blocks): if !block.key_ready: Reloc::get_tree_block_key(fs_info, block)?
5. /* Relocate */
6. for block in rbtree_postorder(blocks):
   - if block.owner ∧ (!btrfs_is_fstree(block.owner) ∨ block.owner == DATA_RELOC_TREE_OBJECTID):
     - Reloc::relocate_cowonly_block(trans, rc, block, path)?
     - continue.
   - node = Reloc::build_backref_tree(trans, rc, &block.key, block.level, block.bytenr)?
   - Reloc::relocate_tree_block(trans, rc, node, &block.key, path)?
7. Reloc::finish_pending_nodes(trans, rc, path, ret).
8. Reloc::free_block_list(blocks).

`Reloc::cow_block_hook(trans, root, *buf, *cow) -> Result<()>`:
1. rc = fs_info.reloc_ctl; if rc.is_none(): return Ok.
2. assert(!(rc.stage == UpdateDataPtrs ∧ btrfs_is_data_reloc_root(root))).
3. level = btrfs_header_level(buf).
4. first_cow = btrfs_header_generation(buf) ≤ btrfs_root_last_snapshot(root.root_item).
5. if btrfs_root_id(root) == BTRFS_TREE_RELOC_OBJECTID ∧ rc.create_reloc_tree:
   - node = rc.backref_cache.path[level].
   - if node.bytenr != buf.start ∧ node.new_bytenr != buf.start: log "backref cache was expecting ..."; return Err(Euclean).
   - btrfs_backref_drop_node_buffer(node).
   - refcount_inc(cow.refs).
   - node.eb = cow.
   - node.new_bytenr = cow.start.
   - if !node.pending: list_move_tail(node.list, rc.backref_cache.pending[level]); node.pending = 1.
   - if first_cow: Reloc::mark_block_processed(rc, node).
   - if first_cow ∧ level > 0: rc.nodes_relocated += buf.len.
6. if level == 0 ∧ first_cow ∧ rc.stage == UpdateDataPtrs:
   - Reloc::replace_file_extents(trans, rc, root, cow).

`Reloc::recover_relocation(fs_info) -> Result<()>`:
1. /* Scan tree_root for BTRFS_TREE_RELOC_OBJECTID roots */
2. key = (BTRFS_TREE_RELOC_OBJECTID, BTRFS_ROOT_ITEM_KEY, U64_MAX).
3. loop downward:
   - btrfs_search_slot(NULL, tree_root, &key, path, 0, 0)?
   - btrfs_item_key_to_cpu(leaf, &key, slot).
   - if key.objectid != BTRFS_TREE_RELOC_OBJECTID ∨ key.type != ROOT_ITEM_KEY: break.
   - reloc_root = btrfs_read_tree_root(tree_root, &key)?
   - set_bit(BTRFS_ROOT_SHAREABLE, reloc_root.state).
   - list_add(reloc_root.root_list, reloc_roots).
   - if btrfs_root_refs(reloc_root.root_item) > 0:
     - fs_root = btrfs_get_fs_root(fs_info, reloc_root.root_key.offset, false).
     - if fs_root is ENOENT: Reloc::mark_garbage_root(reloc_root)?
   - if key.offset == 0: break; else: key.offset -= 1.
4. if reloc_roots.is_empty(): return Ok.
5. rc = Reloc::alloc_control(fs_info)?
6. rc.extent_root = btrfs_extent_root(fs_info, 0).
7. Reloc::chunk_start(fs_info)?
8. Reloc::set_control(rc).
9. trans = btrfs_join_transaction(rc.extent_root)?
10. rc.merge_reloc_tree = true.
11. for reloc_root in reloc_roots:
    - if btrfs_root_refs == 0: list_add_tail(reloc_root, rc.reloc_roots); continue.
    - fs_root = btrfs_get_fs_root(fs_info, reloc_root.root_key.offset, false)?
    - Reloc::add_reloc_root(reloc_root)?
    - fs_root.reloc_root = btrfs_grab_root(reloc_root).
12. btrfs_commit_transaction(trans)?
13. Reloc::merge_reloc_roots(rc).
14. Reloc::unset_control(rc).
15. trans = btrfs_join_transaction(rc.extent_root)?
16. btrfs_commit_transaction(trans)?
17. Reloc::clean_dirty_subvols(rc).
18. Reloc::chunk_end(fs_info).
19. Reloc::free_control(rc).
20. if !btrfs_fs_incompat(REMAP_TREE): btrfs_orphan_cleanup(fs_info.data_reloc_root).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `single_reloc_at_a_time` | INVARIANT | per-fs: at most one task holds BTRFS_FS_RELOC_RUNNING ∧ fs_info.reloc_ctl non-NULL simultaneously. |
| `reloc_ctl_set_implies_running` | INVARIANT | per-fs: fs_info.reloc_ctl.is_some() ⟹ BTRFS_FS_RELOC_RUNNING is set. |
| `bg_inc_ro_balanced` | INVARIANT | per-bg: btrfs_inc_block_group_ro at relocate_block_group entry is matched by btrfs_dec_block_group_ro on every error / cancel / success return. |
| `block_rsv_balanced` | INVARIANT | per-rc: btrfs_alloc_block_rsv at prepare_to_relocate is matched by btrfs_free_block_rsv at relocate_block_group exit (success ∨ error). |
| `cow_hook_cache_coherent` | INVARIANT | per-tap: rc.create_reloc_tree ∧ root_id == TREE_RELOC_OBJECTID ⟹ node = rc.backref_cache.path[level] ∧ (node.bytenr == buf.start ∨ node.new_bytenr == buf.start). |
| `node_new_bytenr_set_once` | INVARIANT | per-relocate_tree_block: shareable-root path asserts node.new_bytenr == 0 at entry; sets it to reloc_root.node.start exactly once. |
| `do_relocation_no_enospc` | INVARIANT | per-do_relocation: ret != -ENOSPC at return (reservation already taken). |
| `data_reloc_inode_orphaned` | INVARIANT | per-create_reloc_inode: btrfs_orphan_add called under same transaction as __insert_orphan_inode; iput on error path. |
| `pending_edge_drained` | INVARIANT | per-build_backref_tree post: cache.pending_edge ∧ cache.useless_node empty. |
| `processed_blocks_idempotent` | INVARIANT | per-mark_block_processed: setting EXTENT_DIRTY on already-set range is a no-op (idempotent rbtree update). |
| `reloc_root_refcount_balanced` | INVARIANT | per-rc.reloc_roots: every btrfs_grab_root paired with btrfs_put_root on free_reloc_roots / merge_reloc_root exit. |
| `cancel_unwinds_through_merge` | INVARIANT | per-cancel: -ECANCELED path still calls prepare_to_merge + merge_reloc_roots before exit (no orphaned reloc_roots). |
| `recover_orphans_data_reloc_root` | INVARIANT | per-btrfs_recover_relocation: post-success btrfs_orphan_cleanup(data_reloc_root) called when !REMAP_TREE. |
| `chunk_start_end_paired` | INVARIANT | per-relocate_block_group: every reloc_chunk_start success is matched by reloc_chunk_end at exit. |
| `remap_path_excludes_data_inode` | INVARIANT | per-bg: should_relocate_using_remap_tree(bg) ⟹ rc.data_inode is None at do_nonremap-or-remap branch. |

### Layer 2: TLA+

`fs/btrfs/relocation.tla`:
- Per-relocate_block_group lifecycle (chunk_start → prepare_to_relocate → main loop → prepare_to_merge → merge_reloc_roots → commit → clean → chunk_end).
- Per-cow tap (btrfs_reloc_cow_block firing during BtrfsCow).
- Per-snapshot (pre_snapshot + post_snapshot) interaction during merge.
- Per-cancel (btrfs_should_cancel_balance) interleaved arbitrarily.
- Per-crash + recover (btrfs_recover_relocation reconstructs from BTRFS_TREE_RELOC_OBJECTID scan).
- Properties:
  - `safety_single_reloc` — ∀ time t: at most one rc with BTRFS_FS_RELOC_RUNNING set.
  - `safety_no_lost_extent` — for every extent E referenced before relocate_block_group, ∃ extent E' referenced after with same key/length/csum (modulo new disk_bytenr).
  - `safety_no_extent_in_bg_post` — at relocate_block_group success, no live extent has start ∈ [bg.start, bg.start + bg.length).
  - `safety_cow_tap_coherent` — every btrfs_reloc_cow_block firing finds matching backref_node in rc.backref_cache.path[level] (no orphan CoW).
  - `safety_snapshot_preserves_relocation` — snapshot created during MOVE_DATA_EXTENTS does not lose extent equivalence; post-snapshot reloc_root registered.
  - `safety_cancel_no_orphan_reloc_root` — cancel-mid-loop unwinds via prepare_to_merge → merge_reloc_roots; rc.reloc_roots empty at chunk_end.
  - `safety_recover_idempotent` — recover_relocation run twice with no relocation in between leaves filesystem in same state.
  - `liveness_relocate_block_group_terminates` — bg with finite live extents → relocate_block_group returns in finite steps.
  - `liveness_merge_completes` — finite reloc_roots → merge_reloc_roots terminates with all merged or marked-garbage.
  - `liveness_recover_terminates` — finite leftover BTRFS_TREE_RELOC_OBJECTID roots → recover_relocation terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Reloc::relocate_block_group` post: ret == Ok ⟹ bg.used == 0 ∧ no extents in [bg.start, bg.start+bg.length) | `Reloc::relocate_block_group` |
| `Reloc::relocate_block_group` exit-on-error: bg_dec_ro called ∧ reloc_chunk_end called ∧ block_rsv freed | `Reloc::relocate_block_group` |
| `Reloc::relocate_block_group_inner` post: rc.reloc_roots merged-or-garbage; rc.dirty_subvol_roots cleaned | `Reloc::relocate_block_group_inner` |
| `Reloc::build_backref_tree` post: result != NULL ⟹ result.detached == false; cache.pending_edge ∧ useless_node empty | `Reloc::build_backref_tree` |
| `Reloc::relocate_tree_block` post: ret == Ok ∨ node freed via btrfs_backref_cleanup_node | `Reloc::relocate_tree_block` |
| `Reloc::do_relocation` post: ret != -ENOSPC; node.pending → backref_drop_node_buffer + list_del | `Reloc::do_relocation` |
| `Reloc::cow_block_hook` post: rc.is_none() ⟹ no-op; else if root is TREE_RELOC ⟹ node.new_bytenr = cow.start ∧ node.pending = 1 ∧ list-moved to pending[level] | `Reloc::cow_block_hook` |
| `Reloc::create_reloc_inode` post: returns inode with reloc_block_group_start == bg.start ∧ btrfs_orphan_add called | `Reloc::create_reloc_inode` |
| `Reloc::recover_relocation` post: every BTRFS_TREE_RELOC_OBJECTID root present at entry is either merged or marked-garbage ∧ data_reloc_root orphan-cleaned (if !REMAP_TREE) | `Reloc::recover_relocation` |
| `Reloc::clone_csums` post: ordered.disk_bytenr range has csums equal to original ordered.original_offset csums | `Reloc::clone_csums` |
| `Reloc::pre_snapshot` post: *bytes_to_reserve += rc.nodes_relocated (when merge_reloc_tree active) | `Reloc::pre_snapshot` |
| `Reloc::post_snapshot` post: new_root.reloc_root.is_some() ∧ reloc_root registered in rc.reloc_root_tree | `Reloc::post_snapshot` |

### Layer 4: Verus/Creusot functional

`Per-balance request → btrfs_relocate_chunk → btrfs_relocate_block_group → (legacy: prepare_to_relocate → find_next_extent loop → build_backref_tree → relocate_tree_block / relocate_cowonly_block → btrfs_reloc_cow_block tap → relocate_data_extent → relocate_file_extent_cluster → prepare_to_merge → merge_reloc_roots → clean_dirty_subvols) | (REMAP_TREE: start_block_group_remapping → do_remap_reloc → mark_chunk_remapped → btrfs_delete_unused_bgs) → btrfs_remove_chunk` semantic equivalence: per-Documentation/filesystems/btrfs.rst, per-`fs/btrfs/volumes.c::btrfs_relocate_chunk`, per-`fs/btrfs/ioctl.c::btrfs_balance_ioctl`.

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

Relocation reinforcement:

- **Per-BTRFS_FS_RELOC_RUNNING test_and_set** — defense against per-concurrent-relocation (only one balance at a time per fs).
- **Per-reloc_cancel_req check at chunk_start** — defense against per-cancel-race-at-start (don't begin work after cancel request).
- **Per-block-group RO before relocation** — defense against per-allocator-races-with-mover.
- **Per-iteration btrfs_should_cancel_balance** — defense against per-uninterruptible-balance.
- **Per-reservation block_rsv up-front** — defense against per-relocation-ENOSPC (do_relocation never returns -ENOSPC; reservation taken in relocate_tree_block).
- **Per-CoW tap cache-coherency check** — defense against per-stale-backref-cache silently rewriting wrong block (return -EUCLEAN, abort).
- **Per-non-shareable-root rejection** — defense against per-corrupt-fs (extent owned by tree that can't legally be CoWed-via-reloc).
- **Per-snapshot reservation doubling** — defense against per-mid-balance-snapshot ENOSPC.
- **Per-mark_garbage_root for orphaned reloc-roots** — defense against per-fs-tree-already-deleted (skip merge, drop reloc-root cleanly).
- **Per-data_reloc inode orphan-on-create** — defense against per-crash leaking the staging inode (orphan-cleanup at next mount).
- **Per-csum-clone via btrfs_reloc_clone_csums** — defense against per-data-corruption (new physical location must match original csums byte-for-byte).
- **Per-zoned_finish before relocation** — defense against per-zoned-allocator-incoherence with mover.
- **Per-data-bg sb_write_started ASSERT** — defense against per-freeze-deadlock (relocation creates ordered extents; must hold sb_write).
- **Per-pinned_by_swapfile check (-ETXTBSY)** — defense against per-swap-file-on-target-bg crash.
- **Per-prepare_to_merge even on error** — defense against per-orphaned reloc-roots after cancel/error.
- **Per-recover_relocation at mount** — defense against per-interrupted-balance leaving the fs in unmountable state.

## Grsecurity/PaX-style Reinforcement

Beyond the upstream hardening above, Rookery layers the following grsec/PaX-style controls onto `fs/btrfs/relocation.c`:

- **PAX_USERCOPY** — bounds-checks every `BTRFS_IOC_BALANCE*` payload so a malformed balance argument cannot drive a slab overrun in the relocation argument parser.
- **PAX_KERNEXEC** — keeps relocation worker callbacks and the backref-cache ops in read-only memory so a memory bug cannot rewrite `relocate_tree_block`/`replace_path` dispatch.
- **PAX_RANDKSTACK** — randomizes kernel-stack offset on each balance ioctl entry so an attacker cannot deterministically probe the long relocate-tree-block path.
- **PAX_REFCOUNT** — wraps `btrfs_root.refs` for reloc-roots and `data_reloc` inode references so a torn merge/cancel sequence cannot wrap.
- **PAX_MEMORY_SANITIZE** — zero-on-free for reloc-root scratch, backref-cache nodes, and `data_reloc` staging buffers so leftover backref keys cannot be recovered from the freelist.
- **PAX_UDEREF** — enforces user/kernel pointer separation across the balance ioctl boundary.
- **PAX_RAP / kCFI** — forward-edge CFI on relocation worker callbacks so a corrupted backref-cache node cannot redirect the relocate dispatch into a gadget.
- **GRKERNSEC_HIDESYM** — strips kernel pointers from relocation EUCLEAN/cancel printks so a crafted image cannot leak kASLR offsets via dmesg.
- **GRKERNSEC_DMESG** — gates dmesg on `CAP_SYSLOG` so reloc-tree corruption diagnostics do not leak pointer material to unprivileged users.
- **CAP_SYS_ADMIN on `BTRFS_IOC_BALANCE*`** — balance ioctls (`BTRFS_IOC_BALANCE`, `BTRFS_IOC_BALANCE_V2`, `BTRFS_IOC_BALANCE_CTL`, `BTRFS_IOC_BALANCE_PROGRESS`) are hard-gated so unprivileged callers cannot initiate or cancel a balance and stress the reloc-tree code path.
- **Reloc-tree signature** — `BTRFS_DATA_RELOC_TREE_OBJECTID` reloc-roots validate the type/parent/generation triple against the live fs-tree at every backref step, treating mismatch as a hard `-EUCLEAN` abort so a crafted reloc-root cannot smuggle a stale-backref-cache attack into the mover.
- **GRKERNSEC_HIDESYM on backref-cache mismatch** — CoW-tap cache-coherency failure traces (`-EUCLEAN` paths) sanitize embedded pointers so a crafted stale-backref attack cannot extract reloc-root or backref-node addresses via dmesg.
- **PAX_MEMORY_SANITIZE on data_reloc inode buffers** — staging inode buffers are zeroed on orphan-cleanup so a crash during relocation cannot leave plaintext data-reloc payloads recoverable from the freelist.

Rationale: relocation operates with elevated rights over the entire on-disk layout (RO block-groups, snapshot-doubled reservations, csum-clone) and a bug in the backref cache or reloc-tree signature translates directly into silent on-disk corruption; capability gating + reloc-tree signature validation is layered on top of the upstream `BTRFS_FS_RELOC_RUNNING` / `prepare_to_merge` defenses.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `fs/btrfs/volumes.c::btrfs_relocate_chunk` caller wrapper + balance ioctl plumbing (covered in `volumes.md` / `ioctl.md` if expanded)
- `fs/btrfs/backref.c` backref cache implementation (covered in `backref.md` Tier-3 if expanded)
- delayed-ref engine (`btrfs_inc_extent_ref`, `btrfs_drop_subtree`) (covered in `extent-tree.md` and `delayed-ref.md` if expanded)
- snapshot creation primitives (covered in `transaction.md` / `inode-ops.md`)
- device-replace driver `fs/btrfs/dev-replace.c` (uses relocation but adds its own state machine — covered separately if expanded)
- balance ioctl + balance-resume mount (covered in `ioctl.md` / `volumes.md` if expanded)
- REMAP_TREE on-disk format details (covered in `remap-tree.md` Tier-3 if expanded)
- free-space cache load/discard for the relocating bg (covered in `free-space-cache.md` Tier-3)
- Implementation code
