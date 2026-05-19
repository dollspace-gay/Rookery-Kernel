# Tier-3: fs/btrfs/block-group.c — btrfs block group lifecycle

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/block-group.c (~4997 lines)
  - fs/btrfs/block-group.h
  - include/uapi/linux/btrfs_tree.h (BTRFS_BLOCK_GROUP_*, BTRFS_BLOCK_GROUP_ITEM_KEY)
  - include/uapi/linux/btrfs.h (BTRFS_FEATURE_COMPAT_RO_BLOCK_GROUP_TREE)
-->

## Summary

A **block group** is btrfs's per-chunk runtime descriptor: a contiguous logical-address window (`start`, `length`) that maps a `BTRFS_BLOCK_GROUP_ITEM` on disk to its in-memory free-space, refcount, and dirty-list state. Per-mount `btrfs_read_block_groups` walks the extent tree (or, for `BLOCK_GROUP_TREE`-incompat filesystems, the dedicated block group tree rooted at `BTRFS_BLOCK_GROUP_TREE_OBJECTID = 11`) and rebuilds the in-core rbtree (`fs_info.block_group_cache_tree`, indexed by `start`). Per-`btrfs_make_block_group` allocates a new BG when a chunk is created; per-`btrfs_remove_block_group` tears one down at unused-bg reclaim or relocation. Per-`btrfs_caching_control` records an async caching job that loads either the on-disk free-space cache, the free-space tree, or the slow extent-tree walk into the BG's `free_space_ctl`. RO transitions (`btrfs_inc_block_group_ro` / `btrfs_dec_block_group_ro`) drain reservations and account `bytes_readonly` in the space-info. Critical for: allocator correctness, mount-time consistency between chunk map and block group items, scrub / relocation / balance.

This Tier-3 covers `fs/btrfs/block-group.c` (~4997 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btrfs_block_group` | per-BG runtime descriptor | `BtrfsBlockGroup` |
| `struct btrfs_block_group_item` | per-BG on-disk item (legacy) | `BtrfsBlockGroupItem` |
| `struct btrfs_block_group_item_v2` | per-BG on-disk item (REMAP_TREE incompat) | `BtrfsBlockGroupItemV2` |
| `struct btrfs_caching_control` | per-BG async caching job | `BtrfsCachingControl` |
| `enum btrfs_caching_type` | per-cache state machine | `BtrfsCachingType` |
| `enum btrfs_block_group_flags` | per-runtime flag bits | `BtrfsBlockGroupRuntimeFlag` |
| `btrfs_get_block_group()` / `_put_block_group()` | per-refcount | `BlockGroup::get` / `put` |
| `btrfs_lookup_block_group()` | per-rbtree-find-containing | `BlockGroup::lookup` |
| `btrfs_lookup_first_block_group()` | per-rbtree-find-first-ge | `BlockGroup::lookup_first` |
| `btrfs_next_block_group()` | per-iterate | `BlockGroup::next` |
| `btrfs_read_block_groups()` | per-mount load | `BlockGroup::read_all` |
| `read_one_block_group()` | per-item-parse | `BlockGroup::read_one` |
| `btrfs_create_block_group()` | per-alloc + init | `BlockGroup::alloc` |
| `btrfs_make_block_group()` | per-new-chunk BG creation | `BlockGroup::make` |
| `btrfs_remove_block_group()` | per-unused-bg removal | `BlockGroup::remove` |
| `btrfs_cache_block_group()` | per-kick caching | `BlockGroup::start_caching` |
| `caching_thread()` | per-work-item caching | `BlockGroup::caching_worker` |
| `load_extent_tree_free()` | per-slow extent-tree walk | `BlockGroup::cache_from_extent_tree` |
| `btrfs_inc_block_group_ro()` | per-RO transition | `BlockGroup::inc_ro` |
| `btrfs_dec_block_group_ro()` | per-RW transition | `BlockGroup::dec_ro` |
| `inc_block_group_ro()` static helper | per-account ro bytes | `BlockGroup::inc_ro_inner` |
| `btrfs_inc_nocow_writers()` / `_dec_nocow_writers()` | per-NOCOW writer ref | `BlockGroup::nocow_writers_inc` / `dec` |
| `btrfs_freeze_block_group()` / `_unfreeze_block_group()` | per-prevent-reuse | `BlockGroup::freeze` / `unfreeze` |
| `btrfs_mark_bg_unused()` | per-enqueue unused_bgs | `BlockGroup::mark_unused` |
| `btrfs_delete_unused_bgs()` | per-reclaim driver | `BlockGroup::delete_unused` |
| `btrfs_reclaim_bgs_work()` | per-async-reclaim worker | `BlockGroup::reclaim_work` |
| `btrfs_start_trans_remove_block_group()` | per-trans-join sized for removal | `BlockGroup::start_trans_remove` |
| `BTRFS_FEATURE_COMPAT_RO_BLOCK_GROUP_TREE` | per-feature bit | shared |
| `BTRFS_BLOCK_GROUP_TREE_OBJECTID = 11` | per-tree root id | shared |

## Compatibility contract

REQ-1: struct btrfs_block_group:
- fs_info: per-mount back-pointer.
- start: per-BG logical-address start (matches chunk map start).
- length: per-BG logical length.
- used / pinned / reserved / delalloc_bytes / bytes_super: per-allocator accounting.
- flags: per-BG type/profile bits (`BTRFS_BLOCK_GROUP_DATA | _METADATA | _SYSTEM | _RAID0 | _RAID1 | _RAID5 | _RAID6 | _RAID10 | _RAID1C3 | _RAID1C4 | _DUP | _REMAPPED | _METADATA_REMAP`).
- cached: per-`enum btrfs_caching_type` (NO / STARTED / FINISHED / ERROR).
- caching_ctl: per-active-caching job (or NULL).
- space_info: per-space-info back-pointer (one of data / metadata / system + mixed).
- free_space_ctl: per-free-space owner.
- cache_node: per-rbtree node in fs_info.block_group_cache_tree.
- list: per-space-info per-raid-type list.
- ro_list: per-space_info.ro_bgs membership when ro != 0.
- refs (refcount_t): per-strong-ref count; reaches 0 only at unmount / removal.
- ro: per-RO depth (0 = RW; >0 = RO; reentrant).
- nocow_writers (atomic_t): per-pending NOCOW write count.
- reservations (atomic_t): per-pending allocator reservations holding groups_sem read lock.
- frozen (atomic_t): per-`btrfs_freeze_block_group` count preventing chunk-map reuse.
- runtime_flags: per-`enum btrfs_block_group_flags` bitfield (NEW / REMOVED / NEEDS_FREE_SPACE / FREE_SPACE_ADDED / IREF / TO_COPY / RELOCATING_REPAIR / CHUNK_ITEM_INSERTED / ZONE_IS_ACTIVE / ZONED_DATA_RELOC / SEQUENTIAL_ZONE / FULLY_REMAPPED / STRIPE_REMOVAL_PENDING).
- last_used / last_remap_bytes / last_identity_remap_count / last_flags: per-committed-value snapshot used to skip BG item update.
- bg_list: per-`fs_info.unused_bgs` / `reclaim_bgs` / `trans.transaction.deleted_bgs` / `trans.new_bgs` membership.
- size_class: per-BTRFS_BG_SZ_NONE / _SMALL / _MEDIUM / _LARGE for size-class allocator.

REQ-2: struct btrfs_caching_control:
- list: per-`fs_info.caching_block_groups` membership.
- mutex: per-caching-job mutex (held by caching_thread while loading).
- wait: per-waitqueue for allocator threads blocked on caching progress.
- work: per-`btrfs_work` queued onto `fs_info.caching_workers`.
- block_group: per-target BG (back-pointer; holds a refcount on the BG).
- progress (atomic_t): per-byte-progress count; wakes waiters every `CACHING_CTL_WAKE_UP = SZ_2M`.
- count (refcount_t): per-caching-control ref; initial 2 (one for BG, one for caching list).

REQ-3: btrfs_lookup_block_group(info, bytenr) -> *BG:
- /* Per-rbtree find containing */
- read_lock(&info.block_group_cache_lock).
- node = rbtree-search-containing(info.block_group_cache_tree, bytenr).
- if found: btrfs_get_block_group(bg); read_unlock; return bg.
- else: read_unlock; return NULL.
- Result-BG references via cache_node are walked with bg.start <= bytenr < bg.start + bg.length.

REQ-4: btrfs_read_block_groups(info):
- /* Per-mount load */
- root = btrfs_block_group_root(info) (block group tree if BLOCK_GROUP_TREE; else extent tree).
- if !root OR unsupported super.compat_ro_flags:
  - return fill_dummy_bgs(info)  // fs is RO-only.
- key = {0, BTRFS_BLOCK_GROUP_ITEM_KEY, 0}.
- need_clear = (SPACE_CACHE && super.cache_generation != super.generation) OR CLEAR_CACHE.
- loop:
  - find_first_block_group(info, path, &key).
  - if 0 read items (ret > 0): break.
  - leaf = path.nodes[0]; slot = path.slots[0].
  - if REMAP_TREE incompat: read sizeof(btrfs_block_group_item_v2) bytes.
  - else: read sizeof(btrfs_block_group_item); zero-initialize v2-only fields.
  - read_one_block_group(info, &bgi, &key, need_clear).
  - key.objectid += key.offset.
- For each space_info, for each raid-index list: btrfs_sysfs_add_block_group_type(first).
- /* RAID-mixed safety */
- For RAID10 / RAID1* / RAID56 / DUP profiles: inc_block_group_ro on un-mirrored (RAID0, SINGLE) BGs to forbid unsafe allocation.
- btrfs_init_global_block_rsv(info).
- check_chunk_block_group_mappings(info)  // assert each chunk has a BG with matching {start, length, type-mask}.
- If ret && IGNOREBADROOTS: fill_dummy_bgs.

REQ-5: read_one_block_group(info, bgi, key, need_clear):
- ASSERT key.type == BTRFS_BLOCK_GROUP_ITEM_KEY.
- cache = btrfs_create_block_group(info, key.objectid).
- cache.length = key.offset.
- cache.used = bgi.used; cache.last_used = used.
- cache.flags = bgi.flags; cache.last_flags = flags.
- cache.global_root_id = bgi.chunk_objectid.
- cache.space_info = btrfs_find_space_info(info, flags).
- cache.remap_bytes / identity_remap_count from v2 fields (zero on v1).
- btrfs_set_free_space_tree_thresholds.
- If need_clear && SPACE_CACHE: cache.disk_cache_state = BTRFS_DC_CLEAR.
- If !mixed && (DATA && METADATA bits both set): -EINVAL (mixed incompat-mismatch).
- btrfs_load_block_group_zone_info(cache, false).
- exclude_super_stripes(cache).
- /* Fast paths */
- If zoned: btrfs_calc_zone_unusable(cache); free excluded extents.
- Else if cache.length == cache.used: cache.cached = BTRFS_CACHE_FINISHED.
- Else if cache.used == 0 && cache.remap_bytes == 0:
  - cache.cached = BTRFS_CACHE_FINISHED.
  - btrfs_add_new_free_space(cache, start, end, NULL).
- btrfs_add_block_group_cache(cache)  // insert into rbtree (by start) + space_info list.
- btrfs_add_bg_to_space_info(info, cache).
- set_avail_alloc_bits.
- If chunk-writeable && empty: mark for discard or unused.
- Else: inc_block_group_ro(cache, true).

REQ-6: btrfs_create_block_group(info, start) -> *BG:
- cache = kzalloc(struct btrfs_block_group).
- cache.free_space_ctl = kzalloc(struct btrfs_free_space_ctl).
- cache.start = start; cache.fs_info = info.
- cache.full_stripe_len = btrfs_full_stripe_len(info, start).
- cache.discard_index = BTRFS_DISCARD_INDEX_UNUSED.
- refcount_set(&cache.refs, 1).
- spin_lock_init, init_rwsem(data_rwsem), all INIT_LIST_HEAD.
- btrfs_init_free_space_ctl, atomic_set(frozen, 0), mutex_init(free_space_lock).

REQ-7: btrfs_make_block_group(trans, space_info, type, chunk_offset, size) -> *BG:
- /* Called from chunk allocator (phase 2) for a new chunk */
- btrfs_set_log_full_commit(trans).
- cache = btrfs_create_block_group(fs_info, chunk_offset).
- set_bit(BLOCK_GROUP_FLAG_NEW, &cache.runtime_flags)  // prevents premature mark_unused.
- cache.length = size; cache.flags = type; cache.cached = BTRFS_CACHE_FINISHED.
- cache.global_root_id = calculate_global_root_id(fs_info, cache.start).
- If FREE_SPACE_TREE compat_ro: set BLOCK_GROUP_FLAG_NEEDS_FREE_SPACE.
- btrfs_load_block_group_zone_info(cache, true).
- exclude_super_stripes(cache).
- btrfs_add_new_free_space(cache, chunk_offset, chunk_offset + size, NULL).
- cache.space_info = space_info  // caller-supplied; ASSERTed non-NULL.
- btrfs_add_block_group_cache(cache)  // rbtree insert.
- btrfs_add_bg_to_space_info; btrfs_update_global_block_rsv.
- btrfs_link_bg_list(cache, &trans.new_bgs)  // deferred BG item insertion in commit phase.
- btrfs_inc_delayed_refs_rsv_bg_inserts(fs_info).
- set_avail_alloc_bits(fs_info, type).

REQ-8: btrfs_remove_block_group(trans, map):
- block_group = btrfs_lookup_block_group(fs_info, map.start).
- if !block_group: abort, -ENOENT.
- if !block_group.ro && !(flags & BTRFS_BLOCK_GROUP_REMAPPED): abort, -EUCLEAN.
- btrfs_free_excluded_extents.
- btrfs_free_ref_tree_range(fs_info, start, length).
- /* Detach from data + metadata allocation clusters */
- btrfs_return_cluster_to_free_space(bg, &fs_info.data_alloc_cluster).
- btrfs_return_cluster_to_free_space(bg, &fs_info.meta_alloc_cluster).
- btrfs_clear_treelog_bg / btrfs_clear_data_reloc_bg.
- inode = lookup_free_space_inode(bg, path).
- /* Drain dirty / io lists */
- mutex_lock(trans.transaction.cache_write_mutex).
- if bg in io_list: list_del; btrfs_wait_cache_io(trans, bg, path); put.
- if bg in dirty_list: list_del; remove_rsv = true; put.
- btrfs_remove_free_space_inode(trans, inode, bg).
- /* rbtree erase */
- write_lock(&fs_info.block_group_cache_lock).
- rb_erase_cached(&bg.cache_node, &fs_info.block_group_cache_tree).
- RB_CLEAR_NODE; btrfs_put_block_group(bg)  // rbtree-held ref drop.
- write_unlock.
- /* space_info list detach */
- down_write(&space_info.groups_sem).
- list_del_init(&bg.list).
- if space_info.block_groups[index] empty: clear avail-alloc-bits, drop kobj.
- up_write.
- clear_incompat_bg_bits.
- if cached == BTRFS_CACHE_STARTED: btrfs_wait_block_group_cache_done.
- /* Drain any straggler caching_ctl */
- write_lock(block_group_cache_lock).
- ctl = btrfs_get_caching_control(bg).
- if !ctl: scan fs_info.caching_block_groups, pick by bg.
- if ctl: list_del_init(&ctl.list).
- write_unlock.
- if ctl: btrfs_put_caching_control twice (list + lookup).
- btrfs_remove_free_space_cache(bg).
- list_del_init(&bg.ro_list).
- if !(flags & REMAPPED): btrfs_remove_bg_from_sinfo(bg); btrfs_remove_block_group_free_space(trans, bg).
- remove_block_group_item(trans, path, bg)  // delete on-disk item from block group tree / extent tree.
- WARN if has_unwritten_metadata.
- set_bit(BLOCK_GROUP_FLAG_REMOVED, &bg.runtime_flags).
- /* Only drop chunk map if no frozen ref */
- remove_map = (atomic_read(&bg.frozen) == 0).
- if remove_map: btrfs_remove_chunk_map(fs_info, map).
- btrfs_put_block_group(bg)  // lookup-ref.
- if remove_rsv: btrfs_dec_delayed_refs_rsv_bg_updates.

REQ-9: btrfs_cache_block_group(cache, wait) -> int:
- if zoned: return 0  // allocator does not use cache.
- if cache.flags & BTRFS_BLOCK_GROUP_REMAPPED: return 0.
- caching_ctl = kzalloc; INIT_LIST_HEAD; mutex_init; init_waitqueue_head.
- caching_ctl.block_group = cache.
- refcount_set(&caching_ctl.count, 2).
- atomic_set(&caching_ctl.progress, 0).
- btrfs_init_work(&caching_ctl.work, caching_thread, NULL).
- spin_lock(&cache.lock).
- if cache.cached != BTRFS_CACHE_NO: free ctl; inherit cache.caching_ctl; spin_unlock; goto wait.
- WARN_ON(cache.caching_ctl); cache.caching_ctl = ctl; cache.cached = BTRFS_CACHE_STARTED.
- spin_unlock.
- write_lock(block_group_cache_lock); refcount_inc(&ctl.count); list_add_tail(&ctl.list, &fs_info.caching_block_groups); write_unlock.
- btrfs_get_block_group(cache)  // for worker.
- btrfs_queue_work(fs_info.caching_workers, &ctl.work).
- if wait: btrfs_caching_ctl_wait_done(cache, ctl)  // wait FINISHED or ERROR.

REQ-10: caching_thread(work):
- ctl = container_of(work, btrfs_caching_control, work).
- bg = ctl.block_group; fs_info = bg.fs_info.
- mutex_lock(&ctl.mutex); down_read(&fs_info.commit_root_sem).
- load_block_group_size_class(ctl).
- /* Source priority */
- if SPACE_CACHE: ret = load_free_space_cache(bg); if ret == 1: goto done.
- if FREE_SPACE_TREE compat_ro && !BTRFS_FS_FREE_SPACE_TREE_UNTRUSTED: ret = btrfs_load_free_space_tree(ctl).
- else: ret = load_extent_tree_free(ctl)  // slow path: walk extent tree, add gaps as free space.
- done:
  - spin_lock(&bg.lock).
  - bg.caching_ctl = NULL.
  - bg.cached = ret ? BTRFS_CACHE_ERROR : BTRFS_CACHE_FINISHED.
  - spin_unlock.
- up_read; btrfs_free_excluded_extents; mutex_unlock.
- wake_up(&ctl.wait).
- btrfs_put_caching_control(ctl); btrfs_put_block_group(bg).

REQ-11: btrfs_inc_block_group_ro(cache, do_chunk_alloc):
- /* High-level RO entry (relocation, scrub, balance, mark RO) */
- root = btrfs_block_group_root(fs_info); -EUCLEAN if NULL.
- If sb_rdonly: ro_block_group_mutex held; inner inc_block_group_ro(cache, false).
- Loop until !dirty_bg_running:
  - trans = btrfs_join_transaction(root).
  - mutex_lock(&fs_info.ro_block_group_mutex).
  - if BTRFS_TRANS_DIRTY_BG_RUN: drop mutex, end trans, wait_for_commit; retry.
- If do_chunk_alloc:
  - alloc_flags = btrfs_get_alloc_profile(fs_info, cache.flags).
  - If alloc_flags != cache.flags: btrfs_chunk_alloc(trans, space_info, alloc_flags, CHUNK_ALLOC_FORCE) (ENOSPC tolerated).
- ret = inc_block_group_ro(cache, false)  // inner.
- If 0: done.
- If -ETXTBSY: out (swap-extents present).
- If -ENOSPC and SYSTEM type and !do_chunk_alloc: out (avoid system-chunk storm).
- alloc_flags = btrfs_get_alloc_profile(fs_info, space_info.flags).
- btrfs_chunk_alloc(trans, space_info, alloc_flags, CHUNK_ALLOC_FORCE).
- btrfs_zoned_activate_one_bg.
- ret = inc_block_group_ro(cache, false)  // retry.
- If SYSTEM type: check_system_chunk(trans, alloc_flags).
- mutex_unlock(&fs_info.ro_block_group_mutex); btrfs_end_transaction(trans).

REQ-12: inc_block_group_ro(cache, force):
- spin_lock(&sinfo.lock); spin_lock(&cache.lock).
- If cache.swap_extents > 0: -ETXTBSY.
- If cache.ro: cache.ro++; ret = 0  // re-entrant.
- num_bytes = btrfs_block_group_available_space(cache).
- if force: ret = 0.
- elif DATA: ret = (sinfo_used + num_bytes <= sinfo.total_bytes) ? 0 : -ENOSPC.
- else (METADATA / SYSTEM): ret = btrfs_can_overcommit(sinfo, num_bytes, BTRFS_RESERVE_NO_FLUSH) ? 0 : -ENOSPC.
- If 0:
  - sinfo.bytes_readonly += num_bytes.
  - If zoned: sinfo.bytes_readonly += cache.zone_unusable; migrate zone_unusable; cache.zone_unusable = 0.
  - cache.ro++.
  - list_add_tail(&cache.ro_list, &sinfo.ro_bgs).
- spin_unlock both.

REQ-13: btrfs_dec_block_group_ro(cache):
- BUG_ON(!cache.ro).
- spin_lock(&sinfo.lock); spin_lock(&cache.lock).
- If --cache.ro == 0:
  - If zoned: restore zone_unusable.
  - sinfo.bytes_readonly -= btrfs_block_group_available_space(cache).
  - list_del_init(&cache.ro_list).
- spin_unlock.

REQ-14: Per-BLOCK_GROUP_TREE feature:
- super_block.compat_ro_flags & BTRFS_FEATURE_COMPAT_RO_BLOCK_GROUP_TREE (bit 3).
- btrfs_block_group_root() returns the BLOCK_GROUP_TREE root (`BTRFS_BLOCK_GROUP_TREE_OBJECTID = 11`) when set, otherwise the extent tree.
- Per-mount: tree-search hot-path moves BG items off the extent tree, reducing mount latency for large filesystems.
- Per-write: insert_block_group_item, update_block_group_item, remove_block_group_item all dispatch via btrfs_block_group_root.

REQ-15: Per-NOCOW writer protocol:
- btrfs_inc_nocow_writers(fs_info, bytenr) -> *BG:
  - bg = btrfs_lookup_block_group; if !bg: NULL.
  - spin_lock(&bg.lock).
  - If bg.ro: can_nocow = false  // drain.
  - Else atomic_inc(&bg.nocow_writers).
  - spin_unlock.
  - If !can_nocow: btrfs_put_block_group; NULL.
  - Else: keep lookup ref alive (caller pairs with dec).
- btrfs_dec_nocow_writers(bg):
  - if atomic_dec_and_test(&bg.nocow_writers): wake_up_var(&bg.nocow_writers).
  - btrfs_put_block_group(bg).
- btrfs_wait_nocow_writers(bg): wait_var_event(nocow_writers == 0).

REQ-16: Per-freeze / unfreeze:
- btrfs_freeze_block_group(cache): atomic_inc(&cache.frozen).
- btrfs_unfreeze_block_group(cache): atomic_dec_and_test → potentially drops chunk-map.
- Freeze prevents btrfs_remove_block_group from removing the chunk map mid-trim/scrub.

REQ-17: Per-unused-bg reclaim:
- btrfs_mark_bg_unused(bg): under fs_info.unused_bgs_lock, btrfs_link_bg_list(bg, &fs_info.unused_bgs).
- btrfs_delete_unused_bgs(fs_info): per-trans iterate unused_bgs; inc_block_group_ro; clean_pinned_extents; btrfs_remove_block_group + remove chunk via btrfs_start_trans_remove_block_group.

REQ-18: Per-rbtree invariants:
- block_group_cache_tree keyed by start.
- btrfs_bg_start_cmp orders by start; lookup_block_group / _first walks rb_node with cache containment.
- Erasure on REMOVED; reads under read_lock(&block_group_cache_lock); writes under write_lock.

## Acceptance Criteria

- [ ] AC-1: Mount with BLOCK_GROUP_TREE feature: btrfs_block_group_root returns block group tree, not extent tree.
- [ ] AC-2: btrfs_read_block_groups walks all BG items and populates fs_info.block_group_cache_tree; check_chunk_block_group_mappings passes for every chunk.
- [ ] AC-3: btrfs_lookup_block_group(bytenr): returns BG with start <= bytenr < start + length, with refcount incremented.
- [ ] AC-4: btrfs_make_block_group: BLOCK_GROUP_FLAG_NEW set before rbtree insert; BG appears on trans.new_bgs.
- [ ] AC-5: btrfs_remove_block_group: refuses removal when !ro && !REMAPPED (returns -EUCLEAN); succeeds for ro / REMAPPED BG.
- [ ] AC-6: btrfs_cache_block_group: caching_thread reaches CACHE_FINISHED for empty BG via fast path; for non-empty BG via free-space tree / extent-tree walk.
- [ ] AC-7: SPACE_CACHE option: caching_thread tries load_free_space_cache first; on failure falls through to FREE_SPACE_TREE or extent tree.
- [ ] AC-8: btrfs_inc_block_group_ro: bg.ro increments; sinfo.bytes_readonly increases by available_space; subsequent allocations skip this BG.
- [ ] AC-9: btrfs_inc_block_group_ro on BG with swap_extents > 0: returns -ETXTBSY.
- [ ] AC-10: btrfs_dec_block_group_ro: matched pair; bg.ro reaches 0 → bytes_readonly restored; ro_list removed.
- [ ] AC-11: btrfs_inc_nocow_writers on RO BG: returns NULL (no nocow on RO).
- [ ] AC-12: btrfs_freeze_block_group prevents btrfs_remove_block_group from removing chunk map until unfreeze.
- [ ] AC-13: BG with cached == BTRFS_CACHE_STARTED → remove waits for completion via btrfs_wait_block_group_cache_done.
- [ ] AC-14: Mixed-mode mismatch (DATA && METADATA flags without MIXED_GROUPS incompat): mount fails with -EINVAL.
- [ ] AC-15: btrfs_delete_unused_bgs reclaims empty BGs across transactions without leaking refs.

## Architecture

```
struct BtrfsBlockGroup {
  fs_info: *BtrfsFsInfo,
  inode: Option<*BtrfsInode>,
  lock: SpinLock,
  start: u64,
  length: u64,
  pinned: u64,
  reserved: u64,
  used: u64,
  delalloc_bytes: u64,
  bytes_super: u64,
  flags: u64,                           // BTRFS_BLOCK_GROUP_*
  cache_generation: u64,
  global_root_id: u64,
  remap_bytes: u64,
  identity_remap_count: u32,
  last_used: u64,
  last_remap_bytes: u64,
  last_identity_remap_count: u32,
  last_flags: u64,
  bitmap_high_thresh: u32,
  bitmap_low_thresh: u32,
  data_rwsem: RwSemaphore,
  full_stripe_len: u64,
  runtime_flags: u64,                    // enum BtrfsBlockGroupRuntimeFlag bitset
  ro: u32,
  disk_cache_state: i32,
  cached: i32,                           // BtrfsCachingType
  caching_ctl: Option<*BtrfsCachingControl>,
  space_info: *BtrfsSpaceInfo,
  free_space_ctl: *BtrfsFreeSpaceCtl,
  cache_node: RbNode,                    // fs_info.block_group_cache_tree
  list: ListHead,                        // space_info.block_groups[raid_index]
  refs: RefCount,
  cluster_list: ListHead,
  bg_list: ListHead,                     // unused_bgs / reclaim_bgs / new_bgs / deleted_bgs
  ro_list: ListHead,                     // space_info.ro_bgs
  frozen: Atomic<u32>,
  discard_list: ListHead,
  discard_index: i32,
  discard_eligible_time: u64,
  discard_cursor: u64,
  discard_state: BtrfsDiscardState,
  dirty_list: ListHead,
  io_list: ListHead,
  io_ctl: BtrfsIoCtl,
  reservations: Atomic<u32>,
  nocow_writers: Atomic<u32>,
  free_space_lock: Mutex,
  using_free_space_bitmaps: bool,
  using_free_space_bitmaps_cached: bool,
  swap_extents: i32,
  alloc_offset: u64,                     // zoned
  zone_unusable: u64,
  zone_capacity: u64,
  meta_write_pointer: u64,
  physical_map: Option<*BtrfsChunkMap>,
  active_bg_list: ListHead,
  zone_finish_work: WorkStruct,
  last_eb: Option<*ExtentBuffer>,
  size_class: BtrfsBlockGroupSizeClass,
  reclaim_mark: u64,
}

struct BtrfsCachingControl {
  list: ListHead,                        // fs_info.caching_block_groups
  mutex: Mutex,
  wait: WaitQueueHead,
  work: BtrfsWork,
  block_group: *BtrfsBlockGroup,
  progress: Atomic<u64>,                 // CACHING_CTL_WAKE_UP = 2 MiB
  count: RefCount,
}
```

`BlockGroup::lookup(info, bytenr) -> Option<Arc<BlockGroup>>`:
1. info.block_group_cache_lock.read().
2. node = rbtree_search_containing(info.block_group_cache_tree, bytenr).
3. if node.is_none(): return None.
4. bg = container_of(node, BtrfsBlockGroup, cache_node).
5. BlockGroup::get(bg).
6. Some(bg).

`BlockGroup::read_all(info) -> Result<()>`:
1. root = BlockGroup::root(info)  // block group tree if BLOCK_GROUP_TREE; else extent tree.
2. if root.is_none() || super.compat_ro_flags & !BTRFS_FEATURE_COMPAT_RO_SUPP:
   - return fill_dummy_bgs(info).
3. key = BtrfsKey { objectid: 0, type: BTRFS_BLOCK_GROUP_ITEM_KEY, offset: 0 }.
4. need_clear = (SPACE_CACHE && super.cache_generation != super.generation) || CLEAR_CACHE.
5. loop:
   - find_first_block_group(info, path, key).
   - if ret > 0: break.
   - size = if REMAP_TREE: sizeof::<BlockGroupItemV2>() else sizeof::<BlockGroupItem>().
   - read_extent_buffer(leaf, &bgi, item_ptr_offset(leaf, slot), size).
   - BlockGroup::read_one(info, &bgi, &key, need_clear)?
   - key.objectid += key.offset.
6. for sinfo in info.space_info_list():
   - for i in 0..BTRFS_NR_RAID_TYPES:
     - first = sinfo.block_groups[i].first(); if let Some(c): sysfs_add_block_group_type(c).
   - if sinfo.alloc_profile & RAID-with-mirror:
     - for c in sinfo.block_groups[RAID0]: BlockGroup::inc_ro_inner(c, force=true).
     - for c in sinfo.block_groups[SINGLE]: BlockGroup::inc_ro_inner(c, force=true).
7. btrfs_init_global_block_rsv(info).
8. BlockGroup::check_chunk_mappings(info)?
9. on error: if IGNOREBADROOTS: fill_dummy_bgs(info).

`BlockGroup::read_one(info, bgi, key, need_clear) -> Result<()>`:
1. assert key.ty == BTRFS_BLOCK_GROUP_ITEM_KEY.
2. cache = BlockGroup::alloc(info, key.objectid)?.
3. cache.length = key.offset; cache.used = bgi.used; cache.flags = bgi.flags; cache.global_root_id = bgi.chunk_objectid.
4. cache.remap_bytes / identity_remap_count = v2 fields (zeros on v1).
5. cache.space_info = find_space_info(info, cache.flags).
6. set_free_space_tree_thresholds(cache).
7. if need_clear && SPACE_CACHE: cache.disk_cache_state = BTRFS_DC_CLEAR.
8. mixed_mismatch?: bail!(EINVAL).
9. load_block_group_zone_info(cache, false)?.
10. exclude_super_stripes(cache)?.
11. /* Fast paths */
12. if is_zoned: calc_zone_unusable; free_excluded_extents.
13. elif cache.length == cache.used: cache.cached = BTRFS_CACHE_FINISHED; free_excluded_extents.
14. elif cache.used == 0 && cache.remap_bytes == 0:
    - cache.cached = BTRFS_CACHE_FINISHED.
    - add_new_free_space(cache, cache.start, end, None).
15. add_block_group_cache(cache)  // rbtree insert.
16. add_bg_to_space_info(info, cache).
17. set_avail_alloc_bits(info, cache.flags).
18. if chunk_writeable && empty: mark_bg_unused or discard_queue_work.
19. else: inc_ro_inner(cache, force=true).

`BlockGroup::make(trans, space_info, ty, chunk_offset, size) -> Result<Arc<BlockGroup>>`:
1. set_log_full_commit(trans).
2. cache = BlockGroup::alloc(fs_info, chunk_offset)?.
3. set_bit(BLOCK_GROUP_FLAG_NEW, cache.runtime_flags).
4. cache.length = size; cache.flags = ty; cache.cached = BTRFS_CACHE_FINISHED.
5. cache.global_root_id = calculate_global_root_id(fs_info, cache.start).
6. if FREE_SPACE_TREE compat_ro: set_bit(BLOCK_GROUP_FLAG_NEEDS_FREE_SPACE).
7. load_block_group_zone_info(cache, true)?.
8. exclude_super_stripes(cache)?.
9. add_new_free_space(cache, chunk_offset, chunk_offset + size, None).
10. cache.space_info = space_info; assert cache.space_info != null.
11. add_block_group_cache(cache).
12. add_bg_to_space_info; update_global_block_rsv.
13. link_bg_list(cache, &trans.new_bgs).
14. inc_delayed_refs_rsv_bg_inserts(fs_info).
15. set_avail_alloc_bits(fs_info, ty).
16. Ok(cache).

`BlockGroup::remove(trans, map) -> Result<()>`:
1. bg = BlockGroup::lookup(fs_info, map.start).ok_or(ENOENT)?.
2. if !bg.ro && !(bg.flags & BTRFS_BLOCK_GROUP_REMAPPED): abort_transaction(EUCLEAN).
3. free_excluded_extents(bg); free_ref_tree_range(fs_info, bg.start, bg.length).
4. /* Cluster detach */
5. return_cluster(bg, &fs_info.data_alloc_cluster).
6. return_cluster(bg, &fs_info.meta_alloc_cluster).
7. clear_treelog_bg; clear_data_reloc_bg.
8. inode = lookup_free_space_inode(bg, path).
9. /* Drain io_list / dirty_list */
10. mutex_lock(trans.transaction.cache_write_mutex).
11. detach_io_and_dirty(bg, &mut remove_rsv).
12. remove_free_space_inode(trans, inode, bg)?.
13. /* rbtree erase */
14. write_lock(fs_info.block_group_cache_lock).
15. rb_erase_cached(&bg.cache_node, &fs_info.block_group_cache_tree).
16. RB_CLEAR_NODE; BlockGroup::put(bg)  // rbtree-held ref.
17. write_unlock.
18. /* space_info list detach */
19. down_write(&space_info.groups_sem).
20. list_del_init(&bg.list).
21. if list empty: clear avail alloc bits + drop kobj.
22. up_write.
23. clear_incompat_bg_bits(fs_info, bg.flags).
24. if bg.cached == BTRFS_CACHE_STARTED: wait_block_group_cache_done(bg).
25. /* Caching-ctl drain */
26. write_lock(block_group_cache_lock); ctl = get_caching_control(bg).or(scan_caching_list); list_del; write_unlock; put twice.
27. remove_free_space_cache(bg).
28. list_del_init(&bg.ro_list).
29. if !(bg.flags & REMAPPED): remove_bg_from_sinfo(bg); remove_block_group_free_space(trans, bg)?.
30. remove_block_group_item(trans, path, bg)?.
31. warn if has_unwritten_metadata(bg).
32. set_bit(BLOCK_GROUP_FLAG_REMOVED, bg.runtime_flags).
33. remove_map = (atomic_read(&bg.frozen) == 0).
34. if remove_map: remove_chunk_map(fs_info, map).
35. BlockGroup::put(bg)  // lookup ref.
36. if remove_rsv: dec_delayed_refs_rsv_bg_updates(fs_info).

`BlockGroup::start_caching(cache, wait) -> Result<()>`:
1. if is_zoned(fs_info): return Ok(()).
2. if cache.flags & BTRFS_BLOCK_GROUP_REMAPPED: return Ok(()).
3. ctl = CachingControl::alloc(cache)?.
4. ctl.count = 2; ctl.progress = 0.
5. init_work(&ctl.work, BlockGroup::caching_worker).
6. spin_lock(&cache.lock).
7. if cache.cached != BTRFS_CACHE_NO:
   - drop ctl; ctl = cache.caching_ctl.map(|c| { refcount_inc(&c.count); c }).
   - spin_unlock; goto wait_arm.
8. cache.caching_ctl = Some(ctl); cache.cached = BTRFS_CACHE_STARTED.
9. spin_unlock.
10. write_lock(block_group_cache_lock); refcount_inc(ctl.count); list_add_tail(&ctl.list, &fs_info.caching_block_groups); write_unlock.
11. BlockGroup::get(cache).
12. queue_work(fs_info.caching_workers, &ctl.work).
13. wait_arm: if wait && ctl.is_some(): caching_ctl_wait_done(cache, ctl).

`BlockGroup::caching_worker(work)`:
1. ctl = container_of(work, BtrfsCachingControl, work).
2. bg = ctl.block_group.
3. mutex_lock(&ctl.mutex); down_read(&fs_info.commit_root_sem).
4. load_block_group_size_class(ctl).
5. if SPACE_CACHE:
   - ret = load_free_space_cache(bg).
   - if ret == 1: ret = 0; goto done.
   - spin_lock(&bg.lock); bg.cached = BTRFS_CACHE_STARTED; spin_unlock; wake_up(&ctl.wait).
6. if FREE_SPACE_TREE compat_ro && !BTRFS_FS_FREE_SPACE_TREE_UNTRUSTED:
   - ret = load_free_space_tree(ctl).
7. else:
   - ret = load_extent_tree_free(ctl).
8. done:
   - spin_lock(&bg.lock).
   - bg.caching_ctl = None.
   - bg.cached = if ret { BTRFS_CACHE_ERROR } else { BTRFS_CACHE_FINISHED }.
   - spin_unlock.
9. up_read; free_excluded_extents(bg); mutex_unlock.
10. wake_up(&ctl.wait); BlockGroup::put_caching_control(ctl); BlockGroup::put(bg).

`BlockGroup::inc_ro(cache, do_chunk_alloc) -> Result<()>`:
1. root = BlockGroup::root(fs_info).ok_or(EUCLEAN)?.
2. if sb_rdonly(fs_info.sb): ro_block_group_mutex held; inc_ro_inner(cache, force=false); return.
3. loop until !dirty_bg_running:
   - trans = join_transaction(root)?.
   - mutex_lock(&fs_info.ro_block_group_mutex).
   - if test_bit(BTRFS_TRANS_DIRTY_BG_RUN, trans.transaction.flags):
     - drop mutex; end_transaction(trans); wait_for_commit(transid)?; dirty_bg_running = true.
4. if do_chunk_alloc:
   - alloc_flags = get_alloc_profile(fs_info, cache.flags).
   - if alloc_flags != cache.flags: chunk_alloc(trans, space_info, alloc_flags, CHUNK_ALLOC_FORCE). ENOSPC tolerated.
5. ret = inc_ro_inner(cache, force=false).
6. match ret:
   - Ok: out.
   - Err(ETXTBSY): unlock_out.
   - Err(ENOSPC) when cache.flags & SYSTEM && !do_chunk_alloc: unlock_out.
   - Err(ENOSPC): chunk_alloc + zoned_activate_one_bg; inc_ro_inner again.
7. out: if cache.flags & SYSTEM: check_system_chunk(trans, alloc_flags).
8. unlock_out: mutex_unlock(ro_block_group_mutex); end_transaction(trans).

`BlockGroup::inc_ro_inner(cache, force) -> Result<()>`:
1. sinfo = cache.space_info; spin_lock(&sinfo.lock); spin_lock(&cache.lock).
2. if cache.swap_extents > 0: bail!(ETXTBSY).
3. if cache.ro > 0: cache.ro += 1; return Ok(()).
4. num_bytes = block_group_available_space(cache).
5. allowed = if force { true } else if cache.flags & DATA { sinfo_used(sinfo, true) + num_bytes <= sinfo.total_bytes } else { can_overcommit(sinfo, num_bytes, BTRFS_RESERVE_NO_FLUSH) }.
6. if !allowed: spin_unlock both; bail!(ENOSPC).
7. sinfo.bytes_readonly += num_bytes.
8. if is_zoned: sinfo.bytes_readonly += cache.zone_unusable; migrate_zone_unusable(sinfo, -cache.zone_unusable); cache.zone_unusable = 0.
9. cache.ro += 1.
10. list_add_tail(&cache.ro_list, &sinfo.ro_bgs).

`BlockGroup::dec_ro(cache)`:
1. assert cache.ro > 0.
2. sinfo = cache.space_info; spin_lock(&sinfo.lock); spin_lock(&cache.lock).
3. cache.ro -= 1.
4. if cache.ro == 0:
   - if is_zoned: restore cache.zone_unusable; migrate back.
   - sinfo.bytes_readonly -= block_group_available_space(cache).
   - list_del_init(&cache.ro_list).
5. spin_unlock.

`BlockGroup::nocow_writers_inc(fs_info, bytenr) -> Option<Arc<BlockGroup>>`:
1. bg = BlockGroup::lookup(fs_info, bytenr)?.
2. spin_lock(&bg.lock).
3. if bg.ro > 0: spin_unlock; BlockGroup::put(bg); return None.
4. atomic_inc(&bg.nocow_writers); spin_unlock.
5. Some(bg)  // caller pairs with nocow_writers_dec which drops lookup ref.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lookup_refcount_balanced` | INVARIANT | per-lookup: returned BG has refs incremented; caller must put. |
| `rbtree_contains_invariant` | INVARIANT | per-block_group_cache_tree: bg.start <= bytenr < bg.start + bg.length for lookup hits. |
| `make_bg_new_flag_before_rbtree` | INVARIANT | per-make: BLOCK_GROUP_FLAG_NEW set before add_block_group_cache. |
| `remove_requires_ro_or_remapped` | INVARIANT | per-remove: bg.ro > 0 OR bg.flags & REMAPPED. |
| `caching_ctl_refcount_init_2` | INVARIANT | per-start_caching: refcount_set(&ctl.count, 2); list+bg own one each. |
| `cached_transitions_monotonic` | INVARIANT | per-cached: NO → STARTED → FINISHED|ERROR; never reverses. |
| `inc_ro_swap_extents_excluded` | INVARIANT | per-inc_ro_inner: cache.swap_extents > 0 ⇒ ETXTBSY. |
| `dec_ro_underflow_forbidden` | INVARIANT | per-dec_ro: BUG_ON(!cache.ro). |
| `nocow_writers_excluded_on_ro` | INVARIANT | per-nocow_inc: bg.ro > 0 ⇒ no increment. |
| `frozen_blocks_chunk_map_removal` | INVARIANT | per-remove: atomic_read(&frozen) > 0 ⇒ chunk map retained. |
| `block_group_tree_root_dispatch` | INVARIANT | per-block_group_root: COMPAT_RO_BLOCK_GROUP_TREE ⇒ block group tree; else extent tree. |
| `bytes_readonly_balanced` | INVARIANT | per-inc/dec_ro pair: sinfo.bytes_readonly returns to baseline. |

### Layer 2: TLA+

`fs/btrfs/block-group.tla`:
- Per-mount load + per-make + per-cache + per-RO transition + per-remove.
- Properties:
  - `safety_no_remove_without_ro_or_remapped` — per-remove: bg.ro > 0 OR bg.flags & REMAPPED.
  - `safety_no_alloc_on_ro` — per-allocator: BG with ro > 0 never selected.
  - `safety_no_chunk_map_removal_while_frozen` — per-remove: frozen > 0 ⇒ chunk map persists.
  - `safety_caching_ctl_refcount_nonnegative` — per-caching: count >= 0; reaches 0 only after both put_caching_control.
  - `safety_rbtree_unique_start` — per-block_group_cache_tree: distinct BGs have distinct start values.
  - `liveness_caching_eventually_finished` — per-start_caching: cached eventually != STARTED.
  - `liveness_remove_eventually_completes` — per-remove on enqueued unused BG.
  - `safety_block_group_tree_round_trip` — per-COMPAT_RO_BLOCK_GROUP_TREE: insert/lookup/remove route through tree id 11 consistently.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `BlockGroup::lookup` post: returned bg has start <= bytenr < end, refs incremented | `BlockGroup::lookup` |
| `BlockGroup::read_all` post: every chunk has matching BG with consistent type-mask | `BlockGroup::read_all` |
| `BlockGroup::make` post: BG inserted into rbtree, on trans.new_bgs, NEW flag set | `BlockGroup::make` |
| `BlockGroup::remove` post: rbtree erased, REMOVED flag set, refs decremented, chunk map removed iff frozen == 0 | `BlockGroup::remove` |
| `BlockGroup::start_caching` post: ctl.count == 2; bg.cached transitions to STARTED | `BlockGroup::start_caching` |
| `BlockGroup::caching_worker` post: bg.cached ∈ {FINISHED, ERROR}; bg.caching_ctl = None | `BlockGroup::caching_worker` |
| `BlockGroup::inc_ro` post: bg.ro incremented; sinfo.bytes_readonly += available_space (when ro 0→1) | `BlockGroup::inc_ro` |
| `BlockGroup::dec_ro` post: bg.ro decremented; ro_list removed when reaches 0 | `BlockGroup::dec_ro` |
| `BlockGroup::nocow_writers_inc` post: ro ⇒ None; non-ro ⇒ nocow_writers incremented, lookup ref retained | `BlockGroup::nocow_writers_inc` |

### Layer 4: Verus/Creusot functional

`Per-mount(super, chunk_tree, [bg_tree | extent_tree]) → block_group_cache_tree` semantic equivalence: for every BLOCK_GROUP_ITEM in the source tree, a unique BG node exists in block_group_cache_tree with matching {start, length, flags, used, global_root_id}; for every chunk in chunk_tree there exists a BG with matching {start, length, type-mask}. Per-Documentation/filesystems/btrfs.rst BLOCK_GROUP_TREE feature description.

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

block-group reinforcement:

- **Per-refcount strict get/put** — defense against per-BG UAF (rbtree erase + lookup ref tracked separately).
- **Per-BLOCK_GROUP_FLAG_NEW before rbtree insert** — defense against per-mark_unused racing against partially-initialized BG.
- **Per-BLOCK_GROUP_FLAG_REMOVED set under cache_lock** — defense against per-allocator picking removed BG.
- **Per-remove requires ro || REMAPPED** — defense against per-mid-write removal.
- **Per-frozen prevents chunk_map removal** — defense against per-trim/scrub-vs-remove race.
- **Per-caching_ctl refcount initialized to 2** — defense against per-double-free of caching control.
- **Per-mutex(caching_ctl.mutex) + down_read(commit_root_sem)** — defense against per-commit-root mutation during caching.
- **Per-block_group_cache_lock rwlock for rbtree** — defense against per-concurrent insert vs lookup.
- **Per-groups_sem write held across list_del + clear_avail_alloc_bits** — defense against per-allocator selecting drained list.
- **Per-mixed-mode mismatch rejected at mount** — defense against per-cross-type allocation corruption.
- **Per-RAID-mixed safety: force-RO unmirrored profiles** — defense against per-data-loss on partial-redundancy mount.
- **Per-ro_block_group_mutex serialization** — defense against per-RO-vs-dirty_bg_running race.
- **Per-BLOCK_GROUP_TREE feature gated by COMPAT_RO_BLOCK_GROUP_TREE** — defense against per-mismatched on-disk layout.
- **Per-IGNOREBADROOTS dummy bgs only when explicitly enabled** — defense against per-silent-data-loss on damaged fs.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/btrfs/extent-tree.c allocator + free-extent search (covered in `extent-tree.md` Tier-3)
- fs/btrfs/free-space-cache.c + free-space-tree.c (covered in `free-space-cache.md` Tier-3)
- fs/btrfs/space-info.c overcommit + tickets (covered separately if expanded)
- fs/btrfs/chunk.c / volumes.c chunk allocation (covered separately if expanded)
- fs/btrfs/zoned.c zoned device specifics (covered separately if expanded)
- fs/btrfs/relocation.c balance / convert (covered in `relocation.md` Tier-3)
- fs/btrfs/discard.c async discard machinery (covered separately if expanded)
- Implementation code
