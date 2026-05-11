# Tier-3: fs/btrfs/inode.c — Btrfs inode + file I/O operations

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/inode.c (~10775 lines)
  - fs/btrfs/btrfs_inode.h (struct btrfs_inode)
  - fs/btrfs/file.c (btrfs_buffered_write, btrfs_file_write_iter, btrfs_do_write_iter)
  - fs/btrfs/extent_io.c (btrfs_writepages, btrfs_read_folio, btrfs_readahead, subpage I/O)
  - fs/btrfs/ordered-data.c (struct btrfs_ordered_extent, *_ordered_extent helpers)
-->

## Summary

`fs/btrfs/inode.c` is the **per-file I/O + inode-lifecycle core** of btrfs. It owns: per-`struct btrfs_inode` (VFS `struct inode` wrapper) lifecycle (`btrfs_alloc_inode` / `btrfs_destroy_inode` / `btrfs_free_inode` / `btrfs_drop_inode` / `btrfs_evict_inode`); per-inode lookup-from-disk (`btrfs_iget` / `btrfs_iget_path` / `btrfs_iget_locked` + `btrfs_read_locked_inode`); per-extent lookup with extent_map cache fall-through to disk (`btrfs_get_extent` → reads `BTRFS_EXTENT_DATA_KEY` from fs-tree, populates `inode->extent_tree`); per-page/folio read (`btrfs_read_folio` / `btrfs_readahead` in `extent_io.c`) including inline-extent decode (`read_inline_extent` / `uncompress_inline`); per-write submission (`btrfs_writepages` / `btrfs_run_delalloc_range` → COW or NOCOW or inline or compressed); per-buffered-write `file_write_iter` → `btrfs_buffered_write` (per-iov copy_one_range + `btrfs_set_extent_delalloc` + delalloc-bytes accounting); per-ordered-extent (`struct btrfs_ordered_extent`) tracking + `btrfs_finish_one_ordered` (insert file_extent_item, add csums, drop reserved bytes, clear range); subpage-blocksize-aware page IO. Critical for: per-data integrity (data csums via `btrfs_calculate_block_csum_*` + `btrfs_data_csum_ok`), per-COW correctness, per-NOCOW preallocation path, per-inline-small-file space efficiency, per-delalloc accounting.

This Tier-3 covers `fs/btrfs/inode.c` (~10775 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btrfs_inode` | per-inode state | `BtrfsInode` |
| `btrfs_alloc_inode()` | per-VFS `alloc_inode` | `InodeOps::alloc_inode` |
| `btrfs_destroy_inode()` | per-VFS `destroy_inode` | `InodeOps::destroy_inode` |
| `btrfs_free_inode()` | per-RCU free | `InodeOps::free_inode` |
| `btrfs_evict_inode()` | per-VFS `evict_inode` | `InodeOps::evict_inode` |
| `btrfs_drop_inode()` | per-VFS `drop_inode` | `InodeOps::drop_inode` |
| `btrfs_iget()` | per-ino lookup | `InodeOps::iget` |
| `btrfs_iget_path()` | per-ino lookup with preallocated path | `InodeOps::iget_path` |
| `btrfs_iget_locked()` | per-iget5_locked | `InodeOps::iget_locked` |
| `btrfs_init_locked_inode()` | per-iget5 init callback | `InodeOps::init_locked_inode` |
| `btrfs_find_actor()` | per-iget5 match callback | `InodeOps::find_actor` |
| `btrfs_read_locked_inode()` | per-disk read inode_item | `InodeOps::read_locked_inode` |
| `btrfs_add_inode_to_root()` | per-root xarray insert | `InodeOps::add_inode_to_root` |
| `btrfs_del_inode_from_root()` | per-root xarray remove | `InodeOps::del_inode_from_root` |
| `btrfs_init_file_extent_tree()` | per-inode file_extent_tree alloc | `InodeOps::init_file_extent_tree` |
| `btrfs_get_extent()` | per-range extent_map (cache + disk) | `InodeOps::get_extent` |
| `read_inline_extent()` | per-inline read into folio | `InodeOps::read_inline_extent` |
| `uncompress_inline()` | per-inline decompress | `InodeOps::uncompress_inline` |
| `btrfs_read_folio()` (extent_io.c) | per-folio read | `InodeOps::read_folio` |
| `btrfs_readahead()` (extent_io.c) | per-rac readahead | `InodeOps::readahead` |
| `btrfs_writepages()` (extent_io.c) | per-mapping writeback | `InodeOps::writepages` |
| `btrfs_run_delalloc_range()` | per-range dispatch: inline ∨ NOCOW ∨ compressed ∨ COW | `InodeOps::run_delalloc_range` |
| `cow_file_range()` | per-range COW alloc + ordered | `InodeOps::cow_file_range` |
| `run_delalloc_nocow()` | per-range NOCOW (PREALLOC/NODATACOW) | `InodeOps::run_delalloc_nocow` |
| `run_delalloc_compressed()` / `compress_file_range()` | per-range compressed | `InodeOps::run_delalloc_compressed` |
| `run_delalloc_inline()` / `__cow_file_range_inline()` | per-range inline-extent | `InodeOps::run_delalloc_inline` |
| `btrfs_buffered_write()` (file.c) | per-iov buffered write | `InodeOps::buffered_write` |
| `btrfs_do_write_iter()` / `btrfs_file_write_iter()` (file.c) | per-`file_write_iter` | `InodeOps::do_write_iter` / `_file_write_iter` |
| `btrfs_set_extent_delalloc()` | per-range mark EXTENT_DELALLOC + accounting | `InodeOps::set_extent_delalloc` |
| `btrfs_set_delalloc_extent()` | per-extent_state set hook | `InodeOps::set_delalloc_extent` |
| `btrfs_clear_delalloc_extent()` | per-extent_state clear hook | `InodeOps::clear_delalloc_extent` |
| `btrfs_split_delalloc_extent()` / `btrfs_merge_delalloc_extent()` | per-range split/merge accounting | `InodeOps::split/merge_delalloc_extent` |
| `btrfs_finish_one_ordered()` | per-ordered-extent completion | `InodeOps::finish_one_ordered` |
| `btrfs_finish_ordered_io()` | per-bio completion handler | `InodeOps::finish_ordered_io` |
| `btrfs_calculate_block_csum_*()` / `btrfs_check_block_csum()` / `btrfs_data_csum_ok()` | per-data csum compute/verify | `InodeOps::data_csum_*` |
| `btrfs_add_delayed_iput()` / `btrfs_run_delayed_iputs()` | per-iput deferred | `InodeOps::delayed_iput_*` |
| `btrfs_writepage_cow_fixup()` / `btrfs_writepage_fixup_worker()` | per-page-mkwrite fixup | `InodeOps::writepage_cow_fixup` |
| `btrfs_aops` | per-data-inode address_space_operations | `InodeOps::BTRFS_AOPS` |

## Compatibility contract

REQ-1: struct btrfs_inode (key per-inode state):
- vfs_inode: per-VFS `struct inode` (must be last for `BTRFS_I()` container_of upward / downward use).
- root: *BtrfsRoot (subvolume root).
- generation: per-create generation.
- last_trans / last_sub_trans / logged_trans / last_log_commit / last_reflink_trans / last_unlink_trans: per-fsync + per-reflink tracking.
- io_tree: per-inode `extent_io_tree` (IO_TREE_INODE_IO) — per-byte-range bits EXTENT_DELALLOC, EXTENT_LOCKED, EXTENT_DELALLOC_NEW, EXTENT_DEFRAG, EXTENT_DEFRAG_RANGE, EXTENT_QGROUP_RESERVED, EXTENT_FINISHING_ORDERED.
- file_extent_tree: per-inode `extent_io_tree *` (IO_TREE_INODE_FILE_EXTENT) — tracks file-extent-item ranges (only when !NO_HOLES + S_ISREG + !free-space-inode).
- extent_tree: per-inode `extent_map_tree` — per-range extent_map cache (logical→physical mapping).
- ordered_tree (RB_ROOT) + ordered_tree_last + ordered_tree_lock (spin): per-ordered-extent in-flight set.
- delalloc_inodes (list_head, anchored on root->delalloc_inodes): per-inode-pending-delalloc tracking.
- delalloc_bytes (i64): per-inode total EXTENT_DELALLOC bytes.
- new_delalloc_bytes (i64) — union with last_dir_index_offset for dirs.
- defrag_bytes (i64), defrag_compress (u8).
- csum_bytes (i64): per-inode bytes covered by pending csums.
- outstanding_extents (u32): per-inode reserved-metadata extents counter.
- disk_i_size (u64): per-inode last-on-disk i_size.
- block_rsv (`btrfs_block_rsv`, BTRFS_BLOCK_RSV_DELALLOC).
- runtime_flags (unsigned long): BTRFS_INODE_DUMMY, BTRFS_INODE_FREE_SPACE_INODE, BTRFS_INODE_ROOT_STUB, BTRFS_INODE_HAS_ORPHAN_ITEM, BTRFS_INODE_ORDERED_DATA_CLOSE, BTRFS_INODE_NO_DELALLOC_FLUSH, ...
- flags (u32): BTRFS_INODE_NODATASUM, BTRFS_INODE_NODATACOW, BTRFS_INODE_PREALLOC, BTRFS_INODE_COMPRESS, BTRFS_INODE_NOCOMPRESS, BTRFS_INODE_SYNC, BTRFS_INODE_IMMUTABLE, BTRFS_INODE_APPEND, ...
- ro_flags (u32).
- prop_compress (u8).
- delayed_node (`btrfs_delayed_node *`).
- delayed_iput (list_head).
- log_mutex / i_mmap_lock / lock (spinlock).
- i_otime_sec, i_otime_nsec: birth time.
- dir_index (u64), index_cnt (u64), ref_root_id (u64).

REQ-2: btrfs_alloc_inode(sb) per-VFS callback:
- ei = alloc_inode_sb(sb, btrfs_inode_cachep, GFP_KERNEL).
- Zero counters: generation, last_trans, last_sub_trans, logged_trans, delalloc_bytes, new_delalloc_bytes, defrag_bytes, disk_i_size, flags, ro_flags, csum_bytes, dir_index, last_unlink_trans, last_reflink_trans, last_log_commit.
- spin_lock_init(&ei->lock); outstanding_extents = 0.
- if sb->s_magic != BTRFS_TEST_MAGIC: `btrfs_init_metadata_block_rsv(fs_info, &ei->block_rsv, BTRFS_BLOCK_RSV_DELALLOC)`.
- runtime_flags = 0; prop_compress = NONE; defrag_compress = NONE; delayed_node = NULL.
- i_otime_sec = 0; i_otime_nsec = 0.
- `btrfs_extent_map_tree_init(&ei->extent_tree)`.
- `btrfs_extent_io_tree_init(fs_info, &ei->io_tree, IO_TREE_INODE_IO)`; ei->io_tree.inode = ei.
- ei->file_extent_tree = NULL (lazy alloc in `btrfs_init_file_extent_tree`).
- mutex_init(&ei->log_mutex); spin_lock_init(&ei->ordered_tree_lock); ei->ordered_tree = RB_ROOT; ordered_tree_last = NULL.
- INIT_LIST_HEAD(&ei->delalloc_inodes); INIT_LIST_HEAD(&ei->delayed_iput); init_rwsem(&ei->i_mmap_lock).
- return &ei->vfs_inode.

REQ-3: btrfs_destroy_inode(vfs_inode) per-VFS callback:
- WARN_ON !hlist_empty(i_dentry), nrpages, block_rsv.reserved/size, outstanding_extents.
- !S_ISDIR ⟹ WARN_ON delalloc_bytes, new_delalloc_bytes, csum_bytes.
- if !data_reloc_root: WARN_ON defrag_bytes.
- if !root: return (early-fail path during create).
- Loop: while (ordered = `btrfs_lookup_first_ordered_extent(inode, (u64)-1)`): WARN + `btrfs_remove_ordered_extent`; double `btrfs_put_ordered_extent`.
- `btrfs_qgroup_check_reserved_leak(inode)`.
- `btrfs_del_inode_from_root(inode)`.
- `btrfs_drop_extent_map_range(inode, 0, (u64)-1, false)`.
- `btrfs_inode_clear_file_extent_range(inode, 0, (u64)-1)`.
- `btrfs_put_root(inode->root)`.

REQ-4: btrfs_evict_inode(inode):
- if !root: clear_inode; return.
- `evict_inode_truncate_pages(inode)`.
- Skip eviction if i_nlink > 0 ∧ (root_refs ≠ 0 ∨ free-space-inode) ∨ is_bad_inode ∨ FS_LOG_RECOVERING.
- `btrfs_commit_inode_delayed_inode(inode)` (sync the inode item).
- `btrfs_kill_delayed_inode_items(inode)` (drop pending dir-index ops — covered by truncate).
- Reserve block_rsv BTRFS_BLOCK_RSV_TEMP sized for 1-meta-extent; failfast=true.
- `btrfs_i_size_write(inode, 0)`.
- Loop: trans = evict_refill_and_join(root, &rsv); set trans.block_rsv = &rsv; `btrfs_truncate_inode_items(trans, root, control={new_size=0, min_type=0})`; trans.block_rsv = trans_block_rsv; `btrfs_end_transaction`; `btrfs_btree_balance_dirty_nodelay`. Break on ret==0; bail on err != -ENOSPC/-EAGAIN.
- Final pass: trans = evict_refill_and_join(...); `btrfs_orphan_del(trans, inode)`; end_trans.
- `btrfs_block_rsv_release(fs_info, &rsv, (u64)-1, NULL)`; `btrfs_remove_delayed_node(inode)`; clear_inode.

REQ-5: btrfs_iget(ino, root):
- inode = btrfs_iget_locked(ino, root). if !inode: ERR_PTR(-ENOMEM).
- if !(I_NEW): return inode (cache hit).
- path = btrfs_alloc_path; if !path: iget_failed; ERR_PTR(-ENOMEM).
- ret = `btrfs_read_locked_inode(inode, path)`; free_path; if ret: ERR_PTR(ret).
- if S_ISDIR: i_opflags |= IOP_FASTPERM_MAY_EXEC.
- unlock_new_inode; return inode.

REQ-6: btrfs_iget_locked(ino, root):
- args.ino = ino; args.root = root.
- inode = `iget5_locked_rcu(root->fs_info->sb, hashval=btrfs_inode_hash(ino, root), btrfs_find_actor, btrfs_init_locked_inode, &args)`.
- btrfs_init_locked_inode callback: `btrfs_set_inode_number`; `BTRFS_I(inode)->root = btrfs_grab_root(args->root)`; if args.root == tree_root ∧ ino != BTREE_INODE_OBJECTID: set BTRFS_INODE_FREE_SPACE_INODE runtime_flag.
- btrfs_find_actor: matches (args->ino == btrfs_ino) ∧ (args->root == BTRFS_I(inode)->root).

REQ-7: btrfs_read_locked_inode(inode, path):
- Build `key = {objectid=ino, type=BTRFS_INODE_ITEM_KEY, offset=0}`.
- `btrfs_search_slot(NULL, root, &key, path, 0, 0)`; on -ENOENT: cleanup + return -ESTALE/-ENOENT.
- Read inode_item via `btrfs_item_ptr`; copy generation, mode, uid, gid, nlink, size, blocks, atime, mtime, ctime, otime, flags, sequence, rdev.
- Set runtime fields: vfs_inode->i_op + i_fop + i_mapping->a_ops per S_ISREG/_ISDIR/_ISLNK/_ISCHR/_ISBLK/_ISFIFO/_ISSOCK.
- `btrfs_init_file_extent_tree(inode)`.
- `btrfs_add_inode_to_root(inode, false)`.
- Read inline ACL/xattrs cached on leaf when possible (`acls_after_inode_item`).
- Cache `disk_i_size = vfs_inode->i_size`.

REQ-8: btrfs_init_file_extent_tree(inode):
- if file_extent_tree already set: WARN_ON.
- if NO_HOLES incompat: return 0 (tree-checker covers).
- if !S_ISREG: return 0.
- if free-space-inode: return 0.
- file_extent_tree = kmalloc_obj(struct extent_io_tree).
- `btrfs_extent_io_tree_init(fs_info, file_extent_tree, IO_TREE_INODE_FILE_EXTENT)`.
- lockdep_set_class on lock.

REQ-9: btrfs_get_extent(inode, folio, start, len) — extent_map cache + fall-through to disk:
- read_lock(em_tree.lock); em = `btrfs_lookup_extent_mapping(em_tree, start, len)`; read_unlock.
- if em:
  - if em.start > start ∨ end(em) <= start: free.
  - else if em.disk_bytenr == EXTENT_MAP_INLINE ∧ folio: free (need fresh read into folio).
  - else: return em (cache hit).
- em = btrfs_alloc_extent_map; em.start = EXTENT_MAP_HOLE; em.disk_bytenr = EXTENT_MAP_HOLE; em.len = (u64)-1.
- path = btrfs_alloc_path; path->reada = READA_FORWARD.
- if free-space-inode: path->search_commit_root = true; path->skip_locking = true.
- `btrfs_lookup_file_extent(NULL, root, path, objectid=btrfs_ino(inode), start, 0)`.
- If ret > 0: path->slots[0]-- (back up one slot).
- item = btrfs_item_ptr(leaf, slot, struct btrfs_file_extent_item).
- found_key = btrfs_item_key_to_cpu(leaf, slot).
- if found_key.objectid ≠ ino ∨ found_key.type ≠ BTRFS_EXTENT_DATA_KEY: goto next (hole).
- extent_type = btrfs_file_extent_type(leaf, item).
- extent_end = `btrfs_file_extent_end(path)`.
- if REG ∨ PREALLOC: WARN if !S_ISREG (-EUCLEAN).
- next: advance slot/leaf; if outside our range: set em as hole (start=start, len=len, disk_bytenr=HOLE).
- `btrfs_extent_item_to_extent_map(inode, path, item, em)`.
- if INLINE: ASSERT extent_start==0 ∧ em.start==0 ∧ em.disk_bytenr==INLINE ∧ em.len==sectorsize; `read_inline_extent(path, folio)`.
- insert: write_lock(em_tree); `btrfs_add_extent_mapping(inode, &em, start, len)`; write_unlock.
- return em.

REQ-10: read_inline_extent(path, folio):
- if !folio ∨ folio_test_uptodate: return 0.
- ASSERT folio_pos(folio) == 0.
- If compression != BTRFS_COMPRESS_NONE: `uncompress_inline`.
- Else: copy_size = min(blocksize, ram_bytes); kmap_local; `read_extent_buffer(leaf, kaddr, file_extent_inline_start, copy_size)`; kunmap_local.
- If copy_size < blocksize: folio_zero_range(folio, copy_size, blocksize - copy_size).

REQ-11: btrfs_aops (data inode address_space_operations):
- read_folio = btrfs_read_folio.
- writepages = btrfs_writepages.
- readahead = btrfs_readahead.
- invalidate_folio = btrfs_invalidate_folio.
- launder_folio = btrfs_launder_folio.
- release_folio = btrfs_release_folio.
- migrate_folio = btrfs_migrate_folio.
- dirty_folio = filemap_dirty_folio.
- error_remove_folio = generic_error_remove_folio.
- swap_activate = btrfs_swap_activate.
- swap_deactivate = btrfs_swap_deactivate.

REQ-12: btrfs_read_folio(file, folio) (extent_io.c):
- per-bio_ctrl: build read bio via `submit_extent_folio`, range = [folio_pos, folio_pos + folio_size).
- per-sector subpage: per-`btrfs_subpage` bitmaps (uptodate, dirty, locked, writeback).
- per-inline-extent: detected via `btrfs_get_extent` returning EXTENT_MAP_INLINE; data read in-place from leaf.
- per-hole: zero-fill folio range; mark uptodate.
- per-disk: submit btrfs_bio to disk_bytenr; on completion `btrfs_data_csum_ok` verifies csum.

REQ-13: btrfs_writepages(mapping, wbc) (extent_io.c):
- iterate dirty folios via write_cache_pages.
- per-folio: find_lock_delalloc_range → `btrfs_run_delalloc_range(inode, locked_folio, start, end, wbc)`.
- per-range: submit bio after delalloc plumbing returns.
- after submit, unlocks folio; ordered-extent owns post-IO completion.

REQ-14: btrfs_run_delalloc_range(inode, locked_folio, start, end, wbc):
- if start == 0 ∧ end+1 <= sectorsize ∧ end+1 >= disk_i_size:
  - ret = `run_delalloc_inline(inode, locked_folio)`.
  - if 0: return 1 (locked_folio handled inline; nothing more).
- if `should_nocow(inode, start, end)`: return `run_delalloc_nocow(inode, locked_folio, start, end)`.
- if can_compress ∧ inode_need_compress ∧ `run_delalloc_compressed`: return 1.
- if zoned: return `run_delalloc_cow(inode, locked_folio, start, end, wbc, true)`.
- else: return `cow_file_range(inode, locked_folio, start, end, NULL, 0)`.

REQ-15: cow_file_range(inode, locked_folio, start, end, done_offset, flags):
- num_bytes = ALIGN(end - start + 1, sectorsize).
- alloc_hint = `btrfs_get_extent_allocation_hint(inode, start, num_bytes)`.
- min_alloc_size = data_reloc ? num_bytes : sectorsize (relocation must keep extent boundaries).
- Loop while num_bytes > 0:
  - `cow_one_range(inode, locked_folio, &ins, &cached_state, start, num_bytes, min_alloc_size, alloc_hint, &cur_alloc_size)`:
    1. `btrfs_reserve_extent` (returns &ins {objectid=bytenr, offset=len}).
    2. `btrfs_create_io_em` (extent_map for COW).
    3. `btrfs_alloc_ordered_extent` (BTRFS_ORDERED_REGULAR).
    4. `btrfs_extent_item_to_extent_map` add to inode->extent_tree.
    5. PAGE_UNLOCK (unless KEEP_LOCKED) + PAGE_SET_ORDERED on locked_folio range.
  - start += cur_alloc_size; num_bytes -= cur_alloc_size.
- on -EAGAIN (zoned): wait BTRFS_FS_NEED_ZONE_FINISH then retry; eventually -ENOSPC.

REQ-16: run_delalloc_nocow(inode, locked_folio, start, end) — NOCOW / PREALLOC:
- ASSERT !zoned ∨ data_reloc.
- Walk `BTRFS_EXTENT_DATA_KEY` items in fs-tree for [start, end].
- For each item:
  - if PREALLOC/REG ∧ `can_nocow_file_extent(path, &nocow_args, ...)`: `nocow_one_range`:
    1. `btrfs_create_io_em` (extent_map for existing physical extent).
    2. `btrfs_alloc_ordered_extent` (BTRFS_ORDERED_NOCOW / BTRFS_ORDERED_PREALLOC).
    3. Skip allocator (re-use disk_bytenr from existing file_extent_item).
  - else: `fallback_to_cow(...)` for that sub-range.
- Tracks oe_cleanup_start/len + untouched_start/len for error recovery.

REQ-17: btrfs_set_extent_delalloc(inode, start, end, extra_bits, *cached_state):
- WARN_ON(PAGE_ALIGNED(end)).
- if start >= i_size_read(inode) ∧ !BTRFS_INODE_PREALLOC: extra_bits |= EXTENT_DELALLOC_NEW.
- else: `btrfs_find_new_delalloc_bytes(inode, start, end+1-start, cached_state)` (accounts for partial-fresh bytes).
- return `btrfs_set_extent_bit(&inode->io_tree, start, end, EXTENT_DELALLOC | extra_bits, cached_state)`.

REQ-18: btrfs_set_delalloc_extent(inode, state, bits) — set hook called under io_tree.lock:
- if (EXTENT_DEFRAG ∧ !EXTENT_DELALLOC): WARN.
- if !state.EXTENT_DELALLOC ∧ bits.EXTENT_DELALLOC:
  - len = state.end + 1 - state.start.
  - num_extents = `count_max_extents(fs_info, len)`.
  - spin_lock(inode.lock); `btrfs_mod_outstanding_extents(inode, num_extents)`; spin_unlock.
  - if testing: return.
  - `percpu_counter_add_batch(&fs_info->delalloc_bytes, len, delalloc_batch)`.
  - spin_lock; prev = inode.delalloc_bytes; inode.delalloc_bytes += len; if EXTENT_DEFRAG: defrag_bytes += len; spin_unlock.
  - if !free-space-inode ∧ prev == 0: `btrfs_add_delalloc_inode(inode)` (per-root delalloc_inodes list).
- if !state.EXTENT_DELALLOC_NEW ∧ bits.EXTENT_DELALLOC_NEW: spin_lock; new_delalloc_bytes += len; spin_unlock.

REQ-19: btrfs_clear_delalloc_extent(inode, state, bits) — clear hook (under io_tree.lock):
- Mirror of REQ-18: decrements outstanding_extents, delalloc_bytes, new_delalloc_bytes.
- if bits.EXTENT_CLEAR_META_RESV ∧ root ≠ tree_root: `btrfs_delalloc_release_metadata(inode, len, true)`.
- if not data_reloc/free-space/NORESERVE ∧ bits.EXTENT_CLEAR_DATA_RESV: `btrfs_free_reserved_data_space_noquota(inode, len)`.
- if !free-space-inode ∧ new_delalloc_bytes == 0: spin_lock(root.delalloc_lock); `btrfs_del_delalloc_inode`.
- if bits.EXTENT_ADD_INODE_BYTES: `inode_add_bytes(vfs_inode, len)`.

REQ-20: btrfs_split_delalloc_extent / btrfs_merge_delalloc_extent:
- Adjust outstanding_extents when an extent_state is split or merged, comparing `count_max_extents(fs_info, total_size)` before vs after.

REQ-21: btrfs_buffered_write(iocb, iter) (file.c):
- if NOWAIT: ilock_flags |= BTRFS_ILOCK_TRY.
- `btrfs_inode_lock(BTRFS_I(inode), ilock_flags)`.
- old_isize = i_size_read.
- `generic_write_checks(iocb, iter)`; `btrfs_write_check(iocb, ret)`.
- Loop while iov_iter_count > 0:
  - `copy_one_range(BTRFS_I(inode), iter, &data_reserved, pos, nowait)`:
    1. Reserve data space + meta via `btrfs_check_data_free_space` + `btrfs_delalloc_reserve_metadata`.
    2. `prepare_pages` (grab folios, dirty_for_io).
    3. `lock_and_cleanup_extent_if_need` (lock io_tree range, drop overlapping ordered).
    4. iov_iter_copy_from_user_atomic into folios.
    5. `btrfs_dirty_pages` → `btrfs_set_extent_delalloc` + folio_mark_dirty.
    6. unlock io_tree range.
  - pos += ret; num_written += ret; cond_resched.
- `extent_changeset_free(data_reserved)`.
- if num_written > 0: `pagecache_isize_extended(inode, old_isize, iocb->ki_pos)`; iocb.ki_pos += num_written.
- `btrfs_inode_unlock`; return num_written ?: ret.

REQ-22: btrfs_do_write_iter(iocb, from, encoded):
- if shutdown: -EIO.
- if FS_ERROR: -EROFS.
- if encoded ∧ NOWAIT: -EOPNOTSUPP.
- Dispatch:
  - encoded ⟹ `btrfs_encoded_write`.
  - IOCB_DIRECT ⟹ `btrfs_direct_write`.
  - else ⟹ `btrfs_buffered_write`.
- `btrfs_set_inode_last_sub_trans(inode)`.
- if num_sync > 0: `generic_write_sync(iocb, num_sync)`.

REQ-23: btrfs_finish_one_ordered(ordered_extent):
- Calculate file_offset, num_bytes.
- if !NOCOW ∧ !PREALLOC ∧ !DIRECT ∧ !ENCODED: clear_bits |= EXTENT_DELALLOC_NEW.
- if !NOCOW: clear_bits |= EXTENT_DEFRAG.
- if BTRFS_ORDERED_IOERR: ret = -EIO; goto out (cleanup).
- `btrfs_zone_finish_endio(fs_info, disk_bytenr, disk_num_bytes)`.
- if BTRFS_ORDERED_TRUNCATED: logical_len = truncated_len; if 0 goto out.
- if !NOCOW: clear_bits |= EXTENT_LOCKED|EXTENT_FINISHING_ORDERED; `btrfs_lock_extent_bits(io_tree, start, end, ...)`.
- trans = `btrfs_join_transaction_spacecache(root)` (free-space) ∨ `btrfs_join_transaction(root)`.
- trans.block_rsv = &inode.block_rsv.
- `btrfs_insert_raid_extent(trans, ordered_extent)`.
- if NOCOW: ASSERT csum_list empty; `btrfs_inode_safe_disk_i_size_write(inode, 0)`; `btrfs_update_inode_fallback(trans, inode)`; goto out.
- if PREALLOC: `btrfs_mark_extent_written(trans, inode, file_offset, file_offset+logical_len)`; `btrfs_zoned_release_data_reloc_bg`.
- else: `insert_ordered_extent_file_extent(trans, ordered_extent)`; clear_reserved_extent = false; `btrfs_release_delalloc_bytes(fs_info, disk_bytenr, disk_num_bytes)`.
- `btrfs_unpin_extent_cache(inode, file_offset, ...)`.
- `add_pending_csums(trans, ordered_extent)`.
- Clear io_tree bits in [start, end].
- `btrfs_inode_safe_disk_i_size_write` (extend disk_i_size).
- `btrfs_update_inode_fallback`.
- `btrfs_end_transaction(trans)`.
- on err: `btrfs_unwrite_dirty_pages_for_io_error` + `btrfs_cleanup_ordered_extents` cleanup.

REQ-24: btrfs_calculate_block_csum_folio/_pages + btrfs_check_block_csum + btrfs_data_csum_ok:
- Per-block (sectorsize) csum compute with `fs_info->csum_type`.
- `btrfs_data_csum_ok(bbio, dev, bio_offset, bio_vec)`:
  - For each bvec block: lookup csum from `bbio->csum`; compute; compare.
  - On mismatch: log via `__btrfs_print_data_csum_error`; `btrfs_repair_io_failure(fs_info, bbio->inode, bbio->file_offset+...)`.
- Return true if all blocks pass.

REQ-25: btrfs_add_delayed_iput / btrfs_run_delayed_iputs / btrfs_wait_on_delayed_iputs:
- `btrfs_add_delayed_iput(inode)`: list_add_tail(&inode.delayed_iput, &fs_info->delayed_iputs); `atomic_inc(nr_delayed_iputs)`.
- `btrfs_run_delayed_iput(fs_info, inode)`: `iput(&inode->vfs_inode)`; `atomic_dec_and_wakeup(nr_delayed_iputs)`.
- `btrfs_run_delayed_iputs(fs_info)`: spin-lock-pop loop with cleaner-kthread participation.

REQ-26: Subpage-blocksize support:
- When sectorsize < PAGE_SIZE: `btrfs_subpage` attached to folio (folio.private), per-sector bitmaps for {uptodate, dirty, locked, writeback, error}.
- Per-sector ops gated by `subpage->lock`.
- `wait_subpage_spinlock(folio)` ensures atomic per-sector state transitions.

REQ-27: Inline extent path:
- Predicated by `can_cow_file_range_inline(inode, offset, size, compressed_size)`: offset==0 ∧ size<=PAGE_SIZE ∧ size<=sectorsize ∧ data_len<BTRFS_MAX_INLINE_DATA_SIZE ∧ data_len<=max_inline ∧ size==i_size ∧ !IS_ENCRYPTED.
- `__cow_file_range_inline`: drop_extents[0..sectorsize] + `insert_inline_extent` + `btrfs_update_inode_bytes(size, drop_args.bytes_found)` + `btrfs_update_inode`.

REQ-28: NOCOW path — `should_nocow(inode, start, end)`:
- inode.flags & BTRFS_INODE_PREALLOC for the range, OR
- inode.flags & BTRFS_INODE_NODATACOW (mount with `nodatacow` or `chattr +C`).
- + range has existing extent that allows in-place rewrite (`can_nocow_extent` / `can_nocow_file_extent`).

REQ-29: Ordered-extent state machine (set in `btrfs_alloc_ordered_extent`):
- Flags: REGULAR | NOCOW | PREALLOC | COMPRESSED | DIRECT | ENCODED.
- Per-tree node in inode.ordered_tree (RB tree by file_offset).
- `btrfs_finish_ordered_io` → `btrfs_finish_one_ordered` → `btrfs_remove_ordered_extent` → `btrfs_put_ordered_extent`.

## Acceptance Criteria

- [ ] AC-1: btrfs_alloc_inode initializes all per-inode state (io_tree, extent_tree, ordered_tree, delayed_iput, i_mmap_lock).
- [ ] AC-2: btrfs_iget cache-hit on second lookup of same (ino, root); no disk read.
- [ ] AC-3: btrfs_iget cache-miss → btrfs_read_locked_inode populates fields from `BTRFS_INODE_ITEM_KEY`.
- [ ] AC-4: btrfs_get_extent cache hit → returns em without disk read.
- [ ] AC-5: btrfs_get_extent miss → walks `BTRFS_EXTENT_DATA_KEY`; adds em to inode.extent_tree.
- [ ] AC-6: btrfs_get_extent hole-after-extent → em.disk_bytenr == EXTENT_MAP_HOLE.
- [ ] AC-7: btrfs_get_extent inline-extent with folio → read_inline_extent reads data into folio.
- [ ] AC-8: btrfs_run_delalloc_range dispatches inline path when start==0 ∧ size<=sectorsize ∧ disk_i_size<=sectorsize.
- [ ] AC-9: btrfs_run_delalloc_range dispatches NOCOW when should_nocow==true.
- [ ] AC-10: btrfs_run_delalloc_range dispatches COW path otherwise (zoned ⟹ run_delalloc_cow, else cow_file_range).
- [ ] AC-11: btrfs_buffered_write: writes propagate to delalloc_bytes counter; new_delalloc_bytes for past-EOF.
- [ ] AC-12: btrfs_buffered_write: pagecache_isize_extended called when num_written > 0.
- [ ] AC-13: btrfs_finish_one_ordered: REG path → insert_ordered_extent_file_extent + release_delalloc_bytes + add_pending_csums + update_inode.
- [ ] AC-14: btrfs_finish_one_ordered: PREALLOC path → mark_extent_written; no extent insert.
- [ ] AC-15: btrfs_finish_one_ordered: NOCOW path → safe_disk_i_size_write + update_inode_fallback; no csum insert.
- [ ] AC-16: btrfs_destroy_inode: no leftover ordered_extents (WARN if present).
- [ ] AC-17: btrfs_evict_inode: i_nlink==0 ∧ root_refs>0 → truncate_inode_items to size 0 + orphan_del.
- [ ] AC-18: subpage write: per-sector dirty/writeback transitions atomic under subpage.lock.

## Architecture

```
struct BtrfsInode {
  vfs_inode: VfsInode,                  // must be at end / via container_of
  root: *BtrfsRoot,
  generation: u64,
  last_trans: u64,
  last_sub_trans: u64,
  logged_trans: u64,
  last_log_commit: u64,
  last_unlink_trans: u64,
  last_reflink_trans: u64,
  io_tree: ExtentIoTree,                // IO_TREE_INODE_IO
  file_extent_tree: Option<*ExtentIoTree>,  // IO_TREE_INODE_FILE_EXTENT, lazy
  extent_tree: ExtentMapTree,           // per-range extent_map cache
  ordered_tree: RbRoot,                 // by file_offset
  ordered_tree_last: Option<*OrderedExtent>,
  ordered_tree_lock: Spinlock,
  delalloc_inodes: ListHead,
  delalloc_bytes: i64,
  new_delalloc_bytes: i64,              // union with last_dir_index_offset
  defrag_bytes: i64,
  csum_bytes: i64,
  outstanding_extents: AtomicU32,
  disk_i_size: u64,
  block_rsv: BtrfsBlockRsv,             // BTRFS_BLOCK_RSV_DELALLOC
  runtime_flags: AtomicBitFlags<RuntimeFlag>,
  flags: u32,                           // BTRFS_INODE_*
  ro_flags: u32,
  prop_compress: u8,
  defrag_compress: u8,
  delayed_node: Option<*DelayedNode>,
  delayed_iput: ListHead,
  log_mutex: Mutex,
  i_mmap_lock: RwSemaphore,
  lock: Spinlock,
  i_otime_sec: i64,
  i_otime_nsec: u32,
  dir_index: u64,
  index_cnt: u64,
  ref_root_id: u64,
}
```

`InodeOps::iget(ino, root) -> Result<*BtrfsInode>`:
1. inode = InodeOps::iget_locked(ino, root).
2. if inode is_none: return Err(-ENOMEM).
3. if !(inode.state.contains(I_NEW)): return Ok(inode).
4. path = btrfs_alloc_path; if !path: iget_failed; return Err(-ENOMEM).
5. ret = InodeOps::read_locked_inode(inode, path).
6. free_path; if ret: return Err(ret).
7. if S_ISDIR(inode.vfs_inode.i_mode): inode.vfs_inode.i_opflags |= IOP_FASTPERM_MAY_EXEC.
8. unlock_new_inode(&inode.vfs_inode); return Ok(inode).

`InodeOps::get_extent(inode, folio, start, len) -> Result<*ExtentMap>`:
1. read_lock(extent_tree.lock).
2. em = btrfs_lookup_extent_mapping(extent_tree, start, len).
3. read_unlock(extent_tree.lock).
4. if em.is_some():
   - if em.start > start ∨ em_end(em) <= start: free em; em = None.
   - else if em.disk_bytenr == EXTENT_MAP_INLINE ∧ folio.is_some(): free em; em = None.
   - else: return Ok(em).
5. em = btrfs_alloc_extent_map; em.start = HOLE; em.disk_bytenr = HOLE; em.len = (u64)-1.
6. path = btrfs_alloc_path; path.reada = READA_FORWARD.
7. if btrfs_is_free_space_inode(inode): path.search_commit_root = true; path.skip_locking = true.
8. btrfs_lookup_file_extent(NULL, root, path, btrfs_ino(inode), start, 0).
9. /* Decode found_key + extent_type + extent_end */.
10. if !found ∨ outside range: em.start=start; em.len=len; em.disk_bytenr=HOLE.
11. else: btrfs_extent_item_to_extent_map(inode, path, item, em).
12. if INLINE ∧ folio: InodeOps::read_inline_extent(path, folio).
13. write_lock(extent_tree.lock); btrfs_add_extent_mapping(inode, &em, start, len); write_unlock.
14. return Ok(em).

`InodeOps::run_delalloc_range(inode, locked_folio, start, end, wbc) -> Result<i32>`:
1. if start == 0 ∧ end+1 <= sectorsize ∧ end+1 >= disk_i_size:
   - ret = InodeOps::run_delalloc_inline(inode, locked_folio).
   - if Ok(0): return 1.
2. if should_nocow(inode, start, end): return InodeOps::run_delalloc_nocow(inode, locked_folio, start, end).
3. if can_compress(inode) ∧ inode_need_compress(inode, start, end) ∧ run_delalloc_compressed: return 1.
4. if zoned: return InodeOps::run_delalloc_cow(inode, locked_folio, start, end, wbc, true).
5. return InodeOps::cow_file_range(inode, locked_folio, start, end, None, 0).

`InodeOps::buffered_write(iocb, iter) -> ssize_t`:
1. ilock_flags = NOWAIT ? BTRFS_ILOCK_TRY : 0.
2. btrfs_inode_lock(inode, ilock_flags).
3. old_isize = i_size_read(inode).
4. generic_write_checks(iocb, iter); btrfs_write_check(iocb, ret).
5. while iov_iter_count(iter) > 0:
   - n = copy_one_range(inode, iter, &data_reserved, pos, nowait):
     a. btrfs_check_data_free_space + btrfs_delalloc_reserve_metadata.
     b. prepare_pages (lock + dirty_for_io).
     c. lock_and_cleanup_extent_if_need (lock io_tree + drop overlapping ordered).
     d. iov_iter_copy_from_user_atomic.
     e. btrfs_dirty_pages → InodeOps::set_extent_delalloc → mark_dirty.
     f. unlock io_tree.
   - pos += n; num_written += n; cond_resched.
6. extent_changeset_free(data_reserved).
7. if num_written > 0: pagecache_isize_extended(inode, old_isize, iocb.ki_pos); iocb.ki_pos += num_written.
8. btrfs_inode_unlock; return num_written ?: ret.

`InodeOps::finish_one_ordered(ordered_extent) -> Result<()>`:
1. start = ordered.file_offset; end = start + num_bytes - 1.
2. compute clear_bits per ordered.flags.
3. if BTRFS_ORDERED_IOERR: ret = -EIO; cleanup.
4. btrfs_zone_finish_endio(fs_info, disk_bytenr, disk_num_bytes).
5. if TRUNCATED: logical_len = truncated_len; if 0 goto out.
6. if !NOCOW: lock io_tree on [start, end] with EXTENT_LOCKED | EXTENT_FINISHING_ORDERED.
7. trans = btrfs_join_transaction(root) (or _spacecache for free-space).
8. trans.block_rsv = &inode.block_rsv.
9. btrfs_insert_raid_extent(trans, ordered).
10. Branch:
    - NOCOW: ASSERT csum_list empty; btrfs_inode_safe_disk_i_size_write(inode, 0); btrfs_update_inode_fallback(trans, inode).
    - PREALLOC: btrfs_mark_extent_written(trans, inode, file_offset, file_offset+logical_len); btrfs_zoned_release_data_reloc_bg.
    - REG/DIRECT/ENCODED: insert_ordered_extent_file_extent(trans, ordered); btrfs_release_delalloc_bytes.
11. btrfs_unpin_extent_cache(inode, file_offset, num_bytes, generation).
12. add_pending_csums(trans, ordered) (insert into csum tree).
13. btrfs_clear_extent_bit(io_tree, start, end, clear_bits, &cached_state).
14. btrfs_inode_safe_disk_i_size_write(inode, new_size).
15. btrfs_update_inode_fallback(trans, inode).
16. btrfs_end_transaction(trans).

`InodeOps::set_delalloc_extent(inode, state, bits)`:
1. lockdep_assert_held(&inode.io_tree.lock).
2. if (bits.EXTENT_DEFRAG ∧ !bits.EXTENT_DELALLOC): WARN.
3. if !state.EXTENT_DELALLOC ∧ bits.EXTENT_DELALLOC:
   - len = state.end + 1 - state.start.
   - num_extents = count_max_extents(fs_info, len).
   - spin_lock(inode.lock); btrfs_mod_outstanding_extents(inode, num_extents); spin_unlock.
   - if testing: return.
   - percpu_counter_add_batch(&fs_info.delalloc_bytes, len, delalloc_batch).
   - prev = inode.delalloc_bytes; inode.delalloc_bytes += len; (if EXTENT_DEFRAG: defrag_bytes += len).
   - if !free-space-inode ∧ prev == 0: btrfs_add_delalloc_inode(inode).
4. if !state.EXTENT_DELALLOC_NEW ∧ bits.EXTENT_DELALLOC_NEW: inode.new_delalloc_bytes += len.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `iget_cache_hit_no_disk` | INVARIANT | per-iget: cache hit (!I_NEW) returns without read_locked_inode. |
| `get_extent_cache_hit_no_disk` | INVARIANT | per-get_extent: em valid for [start, len) ∧ !(inline ∧ folio) ⟹ return cached em. |
| `delalloc_bytes_balanced` | INVARIANT | per-set/clear pair: delalloc_bytes per-inode + percpu_counter both balanced. |
| `outstanding_extents_balanced` | INVARIANT | per-set/clear pair: outstanding_extents non-negative across split/merge. |
| `run_delalloc_dispatch_disjoint` | INVARIANT | per-run_delalloc_range: at most one of inline/NOCOW/compressed/COW chosen. |
| `nocow_no_csums` | INVARIANT | per-finish_one_ordered(NOCOW): csum_list empty. |
| `prealloc_no_extent_insert` | INVARIANT | per-finish_one_ordered(PREALLOC): mark_extent_written only; no insert_file_extent_item. |
| `inline_offset_zero` | INVARIANT | per-inline-extent: offset==0 ∧ size<=sectorsize ∧ size==i_size. |
| `buffered_write_isize_extended` | INVARIANT | per-buffered_write: num_written > 0 ⟹ pagecache_isize_extended called. |
| `evict_inode_iput_balanced` | INVARIANT | per-evict_inode: orphan_del + remove_delayed_node + clear_inode all reached. |

### Layer 2: TLA+

`fs/btrfs/inode-ops.tla`:
- Per-inode lifecycle: ALLOC → READ_LOCKED → IN_CACHE → (DIRTY ↔ CLEAN) → EVICT → FREED.
- Per-extent lifecycle: DELALLOC_RESERVED → DELALLOC_DIRTY → ORDERED_INFLIGHT → ORDERED_COMPLETING → FILE_EXTENT_ITEM_INSERTED → DELALLOC_BYTES_DROPPED.
- Per-buffered-write step: RESERVE_DATA → RESERVE_META → LOCK_IO_TREE → COPY → SET_DELALLOC → UNLOCK_IO_TREE → MAYBE_EXTEND_I_SIZE.
- Properties:
  - `safety_no_double_csum_on_nocow` — per-NOCOW ordered: no csum insertion.
  - `safety_inline_only_at_offset_zero` — per-inline-extent: offset == 0.
  - `safety_delalloc_bytes_never_negative` — per-set/clear: inode.delalloc_bytes >= 0.
  - `safety_outstanding_extents_never_negative` — per-set/clear/split/merge.
  - `safety_disk_i_size_monotonic` — per-finish_one_ordered: disk_i_size grows monotonically (within trans).
  - `safety_evict_no_orphan_after_complete` — per-evict: orphan_del invoked iff i_nlink == 0 path taken.
  - `liveness_ordered_extent_completes` — per-ordered: eventually finish_one_ordered called.
  - `liveness_buffered_write_returns` — per-write: returns after iov_iter exhausted ∨ error.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `InodeOps::alloc_inode` post: io_tree, extent_tree, ordered_tree, delayed_iput, i_mmap_lock initialized | `InodeOps::alloc_inode` |
| `InodeOps::iget` post: I_NEW cleared ∨ Err returned | `InodeOps::iget` |
| `InodeOps::read_locked_inode` post: i_mode/i_uid/i_gid/i_size/disk_i_size populated; root xarray updated | `InodeOps::read_locked_inode` |
| `InodeOps::get_extent` post: em covers [start, ..); em added to extent_tree iff miss | `InodeOps::get_extent` |
| `InodeOps::run_delalloc_range` post: exactly one of {inline, NOCOW, compressed, COW} executed | `InodeOps::run_delalloc_range` |
| `InodeOps::buffered_write` post: num_written ≤ iov_iter_count(initial); ki_pos advanced | `InodeOps::buffered_write` |
| `InodeOps::set_extent_delalloc` post: EXTENT_DELALLOC set in [start, end] | `InodeOps::set_extent_delalloc` |
| `InodeOps::finish_one_ordered` post: extent removed from ordered_tree; csum/file_extent_item per-flag-class inserted; disk_i_size extended | `InodeOps::finish_one_ordered` |
| `InodeOps::set_delalloc_extent` post: delalloc_bytes += len ∧ outstanding_extents += num_extents | `InodeOps::set_delalloc_extent` |
| `InodeOps::clear_delalloc_extent` post: delalloc_bytes -= len ∧ outstanding_extents -= num_extents | `InodeOps::clear_delalloc_extent` |
| `InodeOps::evict_inode` post: i_nlink==0 → truncate_inode_items + orphan_del | `InodeOps::evict_inode` |
| `InodeOps::destroy_inode` post: no ordered_extent leak; extent_map_range dropped; root put | `InodeOps::destroy_inode` |

### Layer 4: Verus/Creusot functional

`Per-buffered-write: lock_inode → check → loop {reserve_data + reserve_meta + lock_io_tree + copy + set_delalloc + unlock_io_tree} → extend_i_size → unlock_inode` semantic equivalence: per-Documentation/filesystems/btrfs.rst.

`Per-extent path: btrfs_run_delalloc_range → (inline ⨁ NOCOW ⨁ compressed ⨁ COW) → btrfs_alloc_ordered_extent → bio submission → btrfs_finish_one_ordered → file_extent_item + csum + delalloc release` semantic equivalence to upstream + per-`btrfs check`.

`Per-iget: iget5_locked_rcu → if I_NEW → read inode_item via BTRFS_INODE_ITEM_KEY → populate vfs_inode → unlock_new_inode` semantic equivalence to upstream + per-`btrfs inspect-internal`.

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

Inode-ops reinforcement:

- **Per-extent_map cache invalidation on inline read** — defense against per-stale-inline em masking real data.
- **Per-can_cow_file_range_inline strict predicate** — defense against per-encrypted-or-oversized inline extent.
- **Per-NOCOW path csum_list empty assertion** — defense against per-stale-csum reuse on rewrite.
- **Per-PREALLOC path no insert_file_extent_item** — defense against per-double-insert of file_extent_item.
- **Per-IS_ENCRYPTED inline rejection** — defense against per-leak of plaintext into metadata.
- **Per-data-reloc min_alloc_size == num_bytes** — defense against per-extent-fragmentation breaking reloc.
- **Per-zoned wait_on_bit BTRFS_FS_NEED_ZONE_FINISH** — defense against per-zoned-allocator-stall.
- **Per-ordered-extent IOERR flag propagation** — defense against per-silent-corruption (-EIO surfaced).
- **Per-disk_i_size extended after finish_one_ordered** — defense against per-i_size-regression after crash.
- **Per-EXTENT_FINISHING_ORDERED + EXTENT_LOCKED bracket** — defense against per-concurrent-truncate-vs-finish race.
- **Per-fixup-worker on writepage_cow_fixup** — defense against per-page-mkwrite-without-delalloc-reserve.
- **Per-delalloc_bytes percpu_counter + per-inode counter** — defense against per-flush-decision-error under load.
- **Per-iget5 actor (ino, root)** — defense against per-cross-subvol inode-cache collision.
- **Per-evict iput sequence + orphan_del** — defense against per-orphaned-inode leak after crash.
- **Per-destroy_inode ordered_extent leak check** — defense against per-UAF on remove_ordered_extent skipped.
- **Per-i_mmap_lock rw separation** — defense against per-fallocate vs mmap-write race.
- **Per-subpage spinlock + bitmaps** — defense against per-cross-sector torn writeback.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/btrfs/file.c per-llseek + per-fsync + per-fallocate (covered separately if expanded)
- fs/btrfs/extent_io.c per-bio_ctrl + per-readahead expand logic (covered separately as `extent-io.md` Tier-3 candidate)
- fs/btrfs/ordered-data.c per-ordered_extent alloc + RB tree details (covered separately)
- fs/btrfs/compression.c per-zlib/lzo/zstd codec (covered separately)
- fs/btrfs/delayed-inode.c per-delayed-item plumbing (covered separately)
- fs/btrfs/relocation.c per-balance + per-data-reloc-root (covered separately)
- fs/btrfs/send.c per-send-receive (covered separately)
- fs/btrfs/qgroup.c per-quota accounting (covered separately)
- Direct-I/O path (`btrfs_direct_write` / `btrfs_dio_read`) — covered separately
- Encoded-I/O path (`btrfs_encoded_read` / `_write`) — covered separately
- Implementation code
