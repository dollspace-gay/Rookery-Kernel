# Tier-3: fs/btrfs/tree-log.c — btrfs tree log (log-tree / fsync fast-commit)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/tree-log.c (~8132 lines)
  - fs/btrfs/tree-log.h
  - include/uapi/linux/btrfs_tree.h (BTRFS_TREE_LOG_OBJECTID, BTRFS_TREE_LOG_FIXUP_OBJECTID)
-->

## Summary

The **btrfs tree log** is a per-subvolume write-ahead log (WAL) used to make `fsync(2)` and `O_SYNC` durable without paying the cost of a full transaction commit. Each fs-root may have a transient sibling root — the log root (`root->log_root`, objectid `BTRFS_TREE_LOG_OBJECTID == 7`) — into which fsync writes only the items that changed for the synced inode (and any ancestors / linkees that must be replayed correctly). At commit time the per-root log roots are themselves stitched together by a per-superblock `log_root_tree` (`fs_info->log_root_tree`) whose bytenr is then written into the superblock’s `log_root` field. Mount detects a non-zero superblock `log_root` and invokes `btrfs_recover_log_trees()`, which walks every per-root log tree in three replay stages: pin extents, replay inodes, replay everything (dir items, refs, extents, xattrs, csums). Critical state: `btrfs_log_ctx` (per-fsync caller state), `root->log_mutex` (serializes start/sync/free), `root->log_transid` and `root->log_commit[2]` (per-side rotation), `BTRFS_INODE_NEEDS_FULL_SYNC` / `BTRFS_INODE_COPY_EVERYTHING` (per-inode runtime flags driving fast vs full inode log), `LOG_INODE_ALL` / `LOG_INODE_EXISTS` (per-call inode log mode), `BTRFS_LOG_FORCE_COMMIT` (fallback to full transaction commit). Critical for: durable fsync at low latency, crash consistency, atomic rename across logged subtrees, snapshot/rename interactions.

This Tier-3 covers `fs/btrfs/tree-log.c` (~8132 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btrfs_log_ctx` | per-fsync caller state | `LogCtx` |
| `struct walk_control` | per-replay walker state | `LogWalk` |
| `btrfs_init_log_ctx` | per-fsync init | `LogCtx::init` |
| `btrfs_init_log_ctx_scratch_eb` | per-fsync scratch-eb alloc | `LogCtx::init_scratch_eb` |
| `btrfs_release_log_ctx_extents` | per-fsync drop ordered extents | `LogCtx::release_extents` |
| `start_log_trans` | per-fsync open log subtransaction | `Log::start_trans` |
| `join_running_log_trans` | per-fsync join running log | `Log::join_running_trans` |
| `btrfs_pin_log_trans` / `btrfs_end_log_trans` | per-writer refcount | `Log::pin_trans` / `Log::end_trans` |
| `btrfs_log_inode` | per-inode incremental log | `Log::log_inode` |
| `btrfs_log_inode_parent` | per-inode + ancestors log | `Log::log_inode_parent` |
| `btrfs_log_dentry_safe` | per-fsync top-level entry | `Log::log_dentry_safe` |
| `btrfs_log_new_name` | per-link/rename name-add log | `Log::log_new_name` |
| `btrfs_record_unlink_dir` | per-unlink/rename trans-id stamp | `Log::record_unlink_dir` |
| `btrfs_record_snapshot_destroy` | per-snapshot-rm full-commit hint | `Log::record_snapshot_destroy` |
| `btrfs_record_new_subvolume` | per-create-subvol full-commit hint | `Log::record_new_subvolume` |
| `btrfs_del_dir_entries_in_log` | per-unlink log update | `Log::del_dir_entries_in_log` |
| `btrfs_del_inode_ref_in_log` | per-unlink ref-log update | `Log::del_inode_ref_in_log` |
| `btrfs_sync_log` | per-fsync commit log subtransaction | `Log::sync_log` |
| `update_log_root` | per-sync update log_root_tree entry | `Log::update_log_root` |
| `walk_log_tree` / `walk_down_log_tree` / `walk_up_log_tree` | per-replay traversal | `Log::walk_tree` family |
| `process_one_buffer` | per-replay pin/free buffer callback | `Log::process_one_buffer` |
| `replay_one_buffer` | per-replay leaf callback | `Log::replay_one_buffer` |
| `overwrite_item` | per-replay item insert/overwrite | `Log::overwrite_item` |
| `replay_one_extent` | per-replay file-extent item | `Log::replay_one_extent` |
| `replay_one_name` / `replay_one_dir_item` | per-replay dir/name | `Log::replay_one_dir_item` |
| `replay_xattr_deletes` | per-replay xattr drop | `Log::replay_xattr_deletes` |
| `replay_dir_deletes` | per-replay dir-entry drop | `Log::replay_dir_deletes` |
| `add_inode_ref` | per-replay ref reconstruct | `Log::add_inode_ref` |
| `fixup_inode_link_counts` | per-replay i_nlink repair | `Log::fixup_inode_link_counts` |
| `link_to_fixup_dir` | per-replay attach to fixup dir | `Log::link_to_fixup_dir` |
| `btrfs_recover_log_trees` | mount-time multi-stage replay | `Log::recover_log_trees` |
| `btrfs_free_log` / `btrfs_free_log_root_tree` | per-commit log teardown | `Log::free_log` / `free_log_root_tree` |
| `inode_logged` | per-inode "was logged this trans" oracle | `Log::inode_logged` |
| `need_log_inode` | per-need check on ancestors | `Log::need_log_inode` |
| `btrfs_set_log_full_commit` / `btrfs_need_log_full_commit` | per-sticky fallback | shared (`log-full-commit.rs`) |
| `BTRFS_LOG_FORCE_COMMIT` constant | per-return-value sentinel | `LOG_FORCE_COMMIT` |
| `BTRFS_NO_LOG_SYNC` constant | per-skip-log return | `NO_LOG_SYNC` |
| `LOG_INODE_ALL` / `LOG_INODE_EXISTS` | per-call inode-only enum | `LogInodeMode` |
| `LOG_WALK_PIN_ONLY` / `_REPLAY_INODES` / `_REPLAY_DIR_INDEX` / `_REPLAY_ALL` | per-replay stage enum | `LogWalkStage` |

## Compatibility contract

REQ-1: struct btrfs_log_ctx (per-fsync caller state):
- log_ret: per-fsync result (set by sync path for waiters).
- log_transid: per-fsync log_transid snapshot taken on start_log_trans.
- log_new_dentries: per-fsync "new dentries seen — walk them" flag.
- logging_new_name: per-link/rename call (skip ctx-list insert).
- logging_new_delayed_dentries: per-recursive-dir flush guard.
- logged_before: per-inode "was previously logged in this trans" cache.
- inode: per-target inode.
- list: per-root log_ctxs[index] list-linkage (for wakeup on sync).
- ordered_extents: per-fast-fsync ordered extent watch-list.
- conflict_inodes / num_conflict_inodes / logging_conflict_inodes: per-fsync conflict tracking (∀ MAX_CONFLICT_INODES = 10).
- scratch_eb: per-fast-fsync scratch extent buffer (held under subvol-leaf lock).

REQ-2: btrfs_init_log_ctx(ctx, inode):
- ctx.log_ret = 0; ctx.log_transid = 0.
- ctx.log_new_dentries = false.
- ctx.logging_new_name = false.
- ctx.logging_new_delayed_dentries = false.
- ctx.logged_before = false.
- ctx.inode = inode.
- INIT_LIST_HEAD(&ctx.list).
- INIT_LIST_HEAD(&ctx.ordered_extents).
- INIT_LIST_HEAD(&ctx.conflict_inodes).
- ctx.num_conflict_inodes = 0.
- ctx.logging_conflict_inodes = false.
- ctx.scratch_eb = NULL.

REQ-3: start_log_trans(trans, root, ctx):
- /* Lazy create log_root_tree if first user */
- if !BTRFS_ROOT_HAS_LOG_TREE set on tree_root:
  - mutex_lock(tree_root.log_mutex).
  - if !fs_info.log_root_tree: btrfs_init_log_root_tree(trans, fs_info).
  - set_bit(BTRFS_ROOT_HAS_LOG_TREE, &tree_root.state).
  - mutex_unlock(tree_root.log_mutex).
- mutex_lock(root.log_mutex).
- /* Existing log root? */
- if root.log_root != NULL:
  - if btrfs_need_log_full_commit(trans): ret = BTRFS_LOG_FORCE_COMMIT.
  - /* Zoned: wait if commit on other index */
  - if zoned ∧ atomic_read(&root.log_commit[(log_transid+1) % 2]) > 0:
    - wait_log_commit; retry.
  - /* Track multi-writer concurrency */
  - if !root.log_start_pid: root.log_start_pid = current.pid.
  - else if root.log_start_pid != current.pid: set BTRFS_ROOT_MULTI_LOG_TASKS.
- else /* allocate new log_root */:
  - if zoned ∧ !created /* log_root_tree already created by another fs root in this trans */: ret = BTRFS_LOG_FORCE_COMMIT.
  - btrfs_add_log_tree(trans, root).
  - set BTRFS_ROOT_HAS_LOG_TREE; clear BTRFS_ROOT_MULTI_LOG_TASKS; root.log_start_pid = current.pid.
- atomic_inc(&root.log_writers).
- if !ctx.logging_new_name: list_add_tail(&ctx.list, &root.log_ctxs[root.log_transid % 2]); ctx.log_transid = root.log_transid.
- mutex_unlock(root.log_mutex).
- return ret.

REQ-4: btrfs_log_inode(trans, inode, inode_only, ctx):
- Modes: inode_only ∈ {LOG_INODE_ALL, LOG_INODE_EXISTS}.
- /* Symlinks always full log (inline content) */
- if S_ISLNK: inode_only = LOG_INODE_ALL.
- mutex_lock(inode.log_mutex).
- /* Determine logged-before; cache in ctx.logged_before */
- ctx.logged_before = (inode_logged(trans, inode) == 1).
- /* Force-full-commit on dir with prior unlink in this trans */
- if full_dir_logging ∧ inode.last_unlink_trans >= trans.transid: ret = BTRFS_LOG_FORCE_COMMIT; goto out.
- /* Three-way mode selection: full-sync vs copy-everything vs fast-search */
- if NEEDS_FULL_SYNC: drop_inode_items / truncate_inode_items (per logged_before).
- else if COPY_EVERYTHING ∨ inode_only == LOG_INODE_EXISTS: drop_inode_items (per logged_before).
- else: fast_search = true; goto log_extents.
- /* For DIR full-log, collect delayed items beforehand */
- if full_dir_logging ∧ !ctx.logging_new_delayed_dentries: btrfs_log_get_delayed_items.
- /* If i_nlink == 0, commit delayed inode (remove last ref item) */
- if inode.i_nlink == 0: btrfs_commit_inode_delayed_inode.
- copy_inode_items_to_log(trans, inode, ..., inode_only, ctx, &need_log_inode_item).
- btrfs_log_all_xattrs(trans, inode, path, dst_path, ctx).
- if max_key.type >= BTRFS_EXTENT_DATA_KEY ∧ !fast_search: btrfs_log_holes.
- if need_log_inode_item: log_inode_item.
- if fast_search: btrfs_log_changed_extents(trans, inode, dst_path, ctx).
- else if LOG_INODE_ALL: drain em_tree modified_extents list.
- if full_dir_logging: log_directory_changes + log_delayed_insertion_items + log_delayed_deletion_items.
- inode.logged_trans = trans.transid.
- if !ctx.logging_new_name ∧ inode_only != LOG_INODE_EXISTS: inode.last_log_commit = inode.last_sub_trans.
- if LOG_INODE_ALL: inode.last_reflink_trans = 0.
- mutex_unlock(inode.log_mutex).
- if ret: free_conflicting_inodes(ctx).
- else: log_conflicting_inodes(trans, root, ctx).
- if full_dir_logging ∧ !ctx.logging_new_delayed_dentries ∧ !ret: log_new_delayed_dentries.
- return ret.

REQ-5: btrfs_log_inode_parent(trans, inode, parent, inode_only, ctx):
- Walks dentry chain upward, logging ancestors needed for path-resolution after replay.
- For each ancestor up to a fully-committed inode: log_inode with LOG_INODE_EXISTS.
- Records new-ancestors-fast path when no rename/unlink touched ancestors.
- Returns BTRFS_LOG_FORCE_COMMIT if any ancestor required full commit.

REQ-6: btrfs_log_dentry_safe(trans, dentry, ctx):
- parent = dget_parent(dentry).
- ret = btrfs_log_inode_parent(trans, BTRFS_I(d_inode(dentry)), parent, LOG_INODE_ALL, ctx).
- dput(parent).
- return ret.

REQ-7: btrfs_sync_log(trans, root, ctx):
- /* Already committed concurrently */
- mutex_lock(root.log_mutex); log_transid = ctx.log_transid.
- if root.log_transid_committed >= log_transid: return ctx.log_ret.
- /* If a commit is already in flight on our index, wait */
- index1 = log_transid % 2.
- if atomic_read(&root.log_commit[index1]) > 0: wait_log_commit; return ctx.log_ret.
- atomic_set(&root.log_commit[index1], 1).
- /* Wait for prior commit on opposite index */
- if atomic_read(&root.log_commit[(index1 + 1) % 2]) > 0: wait_log_commit(log_transid - 1).
- /* Drain concurrent writers until log_batch quiesces */
- loop: batch = atomic_read(&root.log_batch); wait_for_writer; break if batch unchanged.
- /* Honor sticky full-commit */
- if btrfs_need_log_full_commit(trans): return BTRFS_LOG_FORCE_COMMIT.
- mark = (log_transid % 2 == 0) ? EXTENT_DIRTY_LOG1 : EXTENT_DIRTY_LOG2.
- blk_start_plug.
- btrfs_write_marked_extents(fs_info, &log.dirty_log_pages, mark).
- /* Snapshot root_item, rotate root.log_transid */
- btrfs_set_root_node(&log.root_item, log.node); memcpy(new_root_item, log.root_item).
- btrfs_set_root_log_transid(root, root.log_transid + 1). log.log_transid = old. root.log_start_pid = 0.
- mutex_unlock(root.log_mutex).
- /* Now operate on log_root_tree (per-fs log of logs) */
- mutex_lock(log_root_tree.log_mutex).
- index2 = log_root_tree.log_transid % 2. list_add_tail(&root_log_ctx.list, &log_root_tree.log_ctxs[index2]).
- update_log_root(trans, log, &new_root_item).
- /* If our entry already committed, return; else atomically claim commit slot */
- if log_root_tree.log_transid_committed >= root_log_ctx.log_transid: return root_log_ctx.log_ret.
- if atomic_read(&log_root_tree.log_commit[index2]) > 0: wait + return.
- atomic_set(&log_root_tree.log_commit[index2], 1).
- if atomic_read(&log_root_tree.log_commit[(index2 + 1) % 2]) > 0: wait_log_commit.
- /* Write log_root_tree extents */
- btrfs_write_marked_extents(fs_info, &log_root_tree.dirty_log_pages, EXTENT_DIRTY_LOG1 | EXTENT_DIRTY_LOG2).
- /* Wait per-log + per-log_root_tree extent IO */
- btrfs_wait_tree_log_extents(log, mark) + btrfs_wait_tree_log_extents(log_root_tree, ...).
- /* Update super.log_root, super.log_root_level — FLUSH to disk */
- log_root_start = log_root_tree.node.bytenr; log_root_level = btrfs_header_level(log_root_tree.node).
- btrfs_set_super_log_root(super, log_root_start); btrfs_set_super_log_root_level(super, log_root_level).
- write_dev_supers(fs_info, super, max_mirror).
- /* Bump log_root_tree.log_transid_committed; wake everyone */
- log_root_tree.log_transid_committed = root_log_ctx.log_transid; cond_wake_up_nomb(&log_root_tree.log_commit_wait[index2]).
- atomic_set(&log_root_tree.log_commit[index2], 0).
- /* Bump root.log_transid_committed; wake root waiters */
- mutex_lock(root.log_mutex); root.log_transid_committed = log_transid; mutex_unlock; cond_wake_up_nomb(&root.log_commit_wait[index1]).
- atomic_set(&root.log_commit[index1], 0).
- return 0.

REQ-8: btrfs_recover_log_trees(log_root_tree):
- /* Multi-pass replay invoked from mount when super.log_root != 0 */
- set_bit(BTRFS_FS_LOG_RECOVERING, &fs_info.flags).
- trans = btrfs_start_transaction(fs_info.tree_root, 0).
- Pass 1 (LOG_WALK_PIN_ONLY): wc.process_func = process_one_buffer; wc.pin = true; wc.log = log_root_tree. walk_log_tree(&wc). Pins every extent referenced by log so subsequent allocator does not overwrite them mid-replay.
- /* Then for each per-root log: */
- key = (BTRFS_TREE_LOG_OBJECTID, BTRFS_ROOT_ITEM_KEY, (u64)-1).
- loop:
  - search log_root_tree → found_key.
  - wc.log = btrfs_read_tree_root(log_root_tree, &found_key).
  - wc.root = btrfs_get_fs_root(fs_info, found_key.offset /* subvol-id */, true).
  - if subvol gone (ENOENT): btrfs_pin_extent_for_log_replay(wc.log.node); skip.
  - else: wc.root.log_root = wc.log; btrfs_record_root_in_trans(trans, wc.root); walk_log_tree(&wc).
  - if final stage LOG_WALK_REPLAY_ALL: fixup_inode_link_counts(&wc); btrfs_init_root_free_objectid(wc.root).
  - wc.root.log_root = NULL; put.
- /* Stage progression */
- Pass 2 (LOG_WALK_REPLAY_INODES): wc.pin = false; wc.process_func = replay_one_buffer; wc.stage = LOG_WALK_REPLAY_INODES; restart per-root loop.
- Pass 3 (LOG_WALK_REPLAY_DIR_INDEX): wc.stage++; restart.
- Pass 4 (LOG_WALK_REPLAY_ALL): wc.stage++; restart.
- btrfs_commit_transaction(trans) /* also unpins blocks */.
- clear_bit(BTRFS_FS_LOG_RECOVERING, &fs_info.flags).
- /* Error path: on any abort, set BTRFS_FS_STATE_LOG_REPLAY_ABORTED to suppress duplicate dump output */.

REQ-9: walk_control / replay stages:
- LOG_WALK_PIN_ONLY (0): only pin metadata (and data for mixed BGs); no item replay.
- LOG_WALK_REPLAY_INODES (1): replay BTRFS_INODE_ITEM_KEY only; link orphans to fixup-dir.
- LOG_WALK_REPLAY_DIR_INDEX (2): replay dir-index items.
- LOG_WALK_REPLAY_ALL (3): replay refs, extents, xattrs, csums; fixup link counts.

REQ-10: replay_one_buffer(eb, wc, gen, level):
- For each leaf slot k:
  - Skip BTRFS_ORPHAN_ITEM_KEY for cur inode if ignore_cur_inode.
  - Dispatch on key.type:
    - BTRFS_INODE_ITEM_KEY: stage>=REPLAY_INODES → overwrite_item.
    - BTRFS_INODE_REF_KEY / BTRFS_INODE_EXTREF_KEY: stage==REPLAY_ALL → add_inode_ref.
    - BTRFS_DIR_INDEX_KEY: stage>=REPLAY_DIR_INDEX → replay_one_dir_item.
    - BTRFS_DIR_ITEM_KEY: stage==REPLAY_ALL → replay_one_dir_item.
    - BTRFS_EXTENT_DATA_KEY: stage==REPLAY_ALL → replay_one_extent.
    - BTRFS_XATTR_ITEM_KEY: stage==REPLAY_ALL → overwrite_item (also replay_xattr_deletes).
    - BTRFS_EXTENT_CSUM_KEY: stage==REPLAY_ALL → overwrite_item.

REQ-11: overwrite_item(wc):
- Search subvol tree for wc.log_key.
- If found ∧ same size ∧ memcmp identical: no-op (avoid CoW).
- If size mismatch: drop + reinsert.
- Per BTRFS_INODE_ITEM_KEY: preserve dst nbytes (load old); reset i_size for dirs.
- Per new insert path: skip_release_on_error = true; insert_empty_item; truncate_item / extend_item if size mismatch.

REQ-12: add_inode_ref / link_to_fixup_dir / fixup_inode_link_counts:
- For each inode-ref / inode-extref item in log: ensure forward & back links match subvol; create missing dir entries; orphan inodes with no parent link end up under FS_TREE_OBJECTID fixup-dir.
- After all replay: fixup_inode_link_counts walks fixup-dir, reconciles i_nlink against actual references; orphans are queued via btrfs_orphan_add for delete after mount.

REQ-13: inode_logged(trans, inode):
- Returns: 1 if inode was logged in this trans; 0 if definitely not; <0 on error.
- Fast path: spin_lock(inode.lock); if inode.logged_trans == trans.transid → return 1.
- Slow path: search log_root for BTRFS_INODE_ITEM_KEY(ino).
- Caches result via mark_inode_as_not_logged for negative result.

REQ-14: btrfs_log_new_name(trans, old_dentry, old_dir, old_dir_index, parent):
- Set BTRFS_INODE_COPY_EVERYTHING on inode (force ref scan on next log).
- ctx.logging_new_name = true.
- If !S_ISDIR(inode): inode.last_unlink_trans = trans.transid.
- inode_logged(trans, inode) == 1 ∨ old_dir logged ⟹ log new name into parent's log via btrfs_log_inode_parent.
- Otherwise no log work needed (will be picked up on next fsync via COPY_EVERYTHING flag).

REQ-15: btrfs_record_unlink_dir(trans, dir, inode, for_rename):
- inode.last_unlink_trans = trans.transid (forces ancestor log on next fsync of inode).
- If for_rename ∧ dir not already logged ∧ inode not already logged: dir.last_unlink_trans = trans.transid (forces conservative full commit on subsequent fsync of dir).

REQ-16: btrfs_set_log_full_commit(trans):
- WRITE_ONCE(fs_info.last_trans_log_full_commit, trans.transid).
- Sticky for the lifetime of trans — any subsequent btrfs_sync_log returns BTRFS_LOG_FORCE_COMMIT and the caller must escalate to btrfs_commit_transaction.

REQ-17: btrfs_free_log / btrfs_free_log_root_tree:
- Per-transaction-commit: walk per-root log tree (walk_log_tree with wc.free = true) to clear dirty pages, unpin reserved extents, drop refs.
- log_root_tree dropped via btrfs_free_log_root_tree after all per-root logs released.

REQ-18: BTRFS_LOG_FORCE_COMMIT / BTRFS_NO_LOG_SYNC constants:
- BTRFS_LOG_FORCE_COMMIT = -(MAX_ERRNO + 1): per-fallback sentinel; caller commits full transaction instead of syncing log.
- BTRFS_NO_LOG_SYNC = 256: per-fsync skip-log (e.g., readonly inode or no changes).

## Acceptance Criteria

- [ ] AC-1: fsync of new file F in committed dir D writes a log entry replayable to recreate F.
- [ ] AC-2: After crash mid-fsync of file F (before log sync completes), replay does NOT see F (atomic per log_transid).
- [ ] AC-3: After crash post log-sync, replay restores F with its full data + xattrs + csums.
- [ ] AC-4: rename A → B with fsync of B replays correctly: B exists, A removed.
- [ ] AC-5: unlink + fsync of parent dir D: replay removes file from D (replay_dir_deletes).
- [ ] AC-6: Concurrent fsync of two files in same subvol both succeed; one drives log_batch wait.
- [ ] AC-7: btrfs_set_log_full_commit sticky: subsequent sync returns BTRFS_LOG_FORCE_COMMIT.
- [ ] AC-8: BTRFS_INODE_NEEDS_FULL_SYNC ⟹ btrfs_log_inode does full drop+truncate path (not fast_search).
- [ ] AC-9: LOG_INODE_EXISTS does not update inode.last_log_commit (so later explicit fsync still drives full log).
- [ ] AC-10: Replay pass 1 pins extents before pass 2 mutates subvol trees.
- [ ] AC-11: Replay handles missing subvol root by pinning log's blocks and skipping.
- [ ] AC-12: fixup_inode_link_counts reconciles orphans → btrfs_orphan_add → delete after mount.
- [ ] AC-13: Symlinks always logged with LOG_INODE_ALL (content in inline extent).
- [ ] AC-14: Snapshot-destroy + fsync(parent) ⟹ snapshot does not reappear after replay.
- [ ] AC-15: New-subvol + fsync(parent) ⟹ forces full commit (parent.last_unlink_trans bumped).

## Architecture

```
struct LogCtx {
  log_ret: i32,
  log_transid: i32,
  log_new_dentries: bool,
  logging_new_name: bool,
  logging_new_delayed_dentries: bool,
  logged_before: bool,
  inode: *BtrfsInode,
  list: ListHead,                 // root.log_ctxs[index] linkage
  ordered_extents: ListHead,
  conflict_inodes: ListHead,
  num_conflict_inodes: i32,
  logging_conflict_inodes: bool,
  scratch_eb: Option<*ExtentBuffer>,
}

struct LogWalk {
  free: bool,
  pin: bool,
  stage: LogWalkStage,             // PinOnly / ReplayInodes / ReplayDirIndex / ReplayAll
  ignore_cur_inode: bool,
  root: Option<*BtrfsRoot>,        // destination subvol (None in PinOnly)
  log: *BtrfsRoot,
  trans: *BtrfsTransHandle,
  process_func: fn(eb, wc, gen, level) -> i32,
  log_leaf: Option<*ExtentBuffer>,
  log_key: BtrfsKey,
  log_slot: i32,
  subvol_path: Option<*BtrfsPath>,
}

enum LogInodeMode { All, Exists }
enum LogWalkStage { PinOnly, ReplayInodes, ReplayDirIndex, ReplayAll }
```

`Log::log_dentry_safe(trans, dentry, ctx) -> i32`:
1. parent = dget_parent(dentry).
2. inode = BTRFS_I(d_inode(dentry)).
3. ret = Log::log_inode_parent(trans, inode, parent, LogInodeMode::All, ctx).
4. dput(parent).
5. return ret.

`Log::log_inode_parent(trans, inode, parent, mode, ctx) -> i32`:
1. /* Start log subtransaction */
2. ret = Log::start_trans(trans, inode.root, ctx).
3. if ret < 0: return ret.
4. /* Log the leaf inode first */
5. ret = Log::log_inode(trans, inode, mode, ctx).
6. if ret == BTRFS_LOG_FORCE_COMMIT: btrfs_set_log_full_commit; goto end_log_trans.
7. /* Climb ancestors logging each as LOG_INODE_EXISTS until fully-committed inode reached */
8. cur = parent; while not fully-committed ∨ name-added-this-trans:
   - ancestor = BTRFS_I(d_inode(cur)).
   - Log::log_inode(trans, ancestor, LogInodeMode::Exists, ctx).
   - cur = cur.d_parent.
9. /* Conflict + new-dentries handling */
10. Log::log_conflicting_inodes(trans, root, ctx).
11. if ctx.log_new_dentries: Log::log_new_dir_dentries(trans, inode, ctx).
12. Log::end_trans(root).
13. return ret.

`Log::log_inode(trans, inode, inode_only, ctx) -> i32`:
1. /* Symlink override */
2. if S_ISLNK(inode.mode): inode_only = LogInodeMode::All.
3. mutex_lock(&inode.log_mutex).
4. ctx.logged_before = (Log::inode_logged(trans, inode) == 1).
5. /* Dir-only force-full-commit guard */
6. if S_ISDIR(inode.mode) ∧ inode_only == All ∧ inode.last_unlink_trans >= trans.transid:
   - mutex_unlock; return BTRFS_LOG_FORCE_COMMIT.
7. /* Choose path: full-sync / copy-everything / fast-search */
8. if NEEDS_FULL_SYNC: drop_inode_items or truncate_inode_items.
9. else if COPY_EVERYTHING ∨ Exists: drop_inode_items.
10. else: fast_search = true; goto log_extents.
11. if full_dir_logging ∧ !ctx.logging_new_delayed_dentries: btrfs_log_get_delayed_items.
12. if inode.i_nlink == 0: btrfs_commit_inode_delayed_inode.
13. copy_inode_items_to_log(...).
14. btrfs_log_all_xattrs.
15. if max_key.type >= EXTENT_DATA_KEY ∧ !fast_search: btrfs_log_holes.
16. label log_extents:
17. if need_log_inode_item: log_inode_item.
18. if fast_search: btrfs_log_changed_extents.
19. else if LogInodeMode::All: drain em_tree.modified_extents.
20. if full_dir_logging: log_directory_changes + log_delayed_insertion_items + log_delayed_deletion_items.
21. spin_lock(&inode.lock); inode.logged_trans = trans.transid.
22. if !ctx.logging_new_name ∧ inode_only != Exists: inode.last_log_commit = inode.last_sub_trans.
23. if inode_only == All: inode.last_reflink_trans = 0.
24. spin_unlock(&inode.lock).
25. mutex_unlock(&inode.log_mutex).
26. if ret: free_conflicting_inodes(ctx); else log_conflicting_inodes(...).
27. if full_dir_logging ∧ !ctx.logging_new_delayed_dentries ∧ !ret: log_new_delayed_dentries.
28. return ret.

`Log::sync_log(trans, root, ctx) -> i32`:
1. mutex_lock(&root.log_mutex). log_transid = ctx.log_transid.
2. if root.log_transid_committed >= log_transid: mutex_unlock; return ctx.log_ret.
3. index1 = log_transid % 2.
4. if atomic_read(&root.log_commit[index1]) > 0: wait_log_commit; return ctx.log_ret.
5. atomic_set(&root.log_commit[index1], 1).
6. if atomic_read(&root.log_commit[(index1+1) % 2]) > 0: wait_log_commit(log_transid - 1).
7. /* Wait for in-flight writers to quiesce */
8. loop: batch = atomic_read(&root.log_batch); wait_for_writer; break if unchanged.
9. if btrfs_need_log_full_commit(trans): mutex_unlock; ret = BTRFS_LOG_FORCE_COMMIT; goto out.
10. mark = if log_transid % 2 == 0 { EXTENT_DIRTY_LOG1 } else { EXTENT_DIRTY_LOG2 }.
11. blk_start_plug. btrfs_write_marked_extents(fs_info, &log.dirty_log_pages, mark).
12. btrfs_set_root_node(&log.root_item, log.node); copy → new_root_item.
13. btrfs_set_root_log_transid(root, root.log_transid + 1). log.log_transid = old. root.log_start_pid = 0.
14. mutex_unlock(&root.log_mutex).
15. mutex_lock(&log_root_tree.log_mutex).
16. index2 = log_root_tree.log_transid % 2. list_add_tail(&root_log_ctx.list, &log_root_tree.log_ctxs[index2]).
17. update_log_root(trans, log, &new_root_item).
18. if log_root_tree.log_transid_committed >= root_log_ctx.log_transid: mutex_unlock; return root_log_ctx.log_ret.
19. if atomic_read(&log_root_tree.log_commit[index2]) > 0: wait; return root_log_ctx.log_ret.
20. atomic_set(&log_root_tree.log_commit[index2], 1).
21. if atomic_read(&log_root_tree.log_commit[(index2+1) % 2]) > 0: wait_log_commit.
22. btrfs_write_marked_extents(fs_info, &log_root_tree.dirty_log_pages, EXTENT_DIRTY_LOG1 | EXTENT_DIRTY_LOG2). blk_finish_plug.
23. btrfs_wait_tree_log_extents(log, mark). btrfs_wait_tree_log_extents(log_root_tree, ...).
24. log_root_start = log_root_tree.node.bytenr. log_root_level = btrfs_header_level(log_root_tree.node).
25. btrfs_set_super_log_root(super, log_root_start). btrfs_set_super_log_root_level(super, log_root_level).
26. write_dev_supers (flushed; barrier honored).
27. log_root_tree.log_transid_committed = root_log_ctx.log_transid. atomic_set(&log_root_tree.log_commit[index2], 0). cond_wake_up_nomb.
28. mutex_lock(&root.log_mutex). root.log_transid_committed = log_transid. mutex_unlock. atomic_set(&root.log_commit[index1], 0). cond_wake_up_nomb.
29. return 0.

`Log::recover_log_trees(log_root_tree) -> i32`:
1. set_bit(BTRFS_FS_LOG_RECOVERING, &fs_info.flags).
2. trans = btrfs_start_transaction(fs_info.tree_root, 0).
3. wc = LogWalk { process_func: process_one_buffer, stage: PinOnly, pin: true, log: log_root_tree, trans, ... }.
4. walk_log_tree(&wc). /* pins log_root_tree extents */
5. Loop over each (BTRFS_TREE_LOG_OBJECTID, ROOT_ITEM_KEY, subvol_id) entry in log_root_tree:
   - wc.log = btrfs_read_tree_root(log_root_tree, &found_key).
   - wc.root = btrfs_get_fs_root(fs_info, found_key.offset, true).
   - if subvol missing: btrfs_pin_extent_for_log_replay(wc.log.node); skip.
   - else: wc.root.log_root = wc.log; btrfs_record_root_in_trans; walk_log_tree(&wc).
   - if stage == ReplayAll: fixup_inode_link_counts(&wc); btrfs_init_root_free_objectid(wc.root).
6. Stage advance: ReplayInodes → ReplayDirIndex → ReplayAll; re-run per-root loop each stage.
7. btrfs_commit_transaction(trans).
8. clear_bit(BTRFS_FS_LOG_RECOVERING, &fs_info.flags).
9. return 0.

`Log::overwrite_item(wc) -> i32`:
1. ASSERT(btrfs_root_id(wc.root) != BTRFS_TREE_LOG_OBJECTID).
2. item_size = btrfs_item_size(wc.log_leaf, wc.log_slot).
3. btrfs_search_slot(NULL, wc.root, &wc.log_key, wc.subvol_path, 0, 0).
4. if found ∧ dst_size == item_size ∧ memcmp identical: release path; return 0 (skip cow).
5. if is_inode_item ∧ found: preserve dst nbytes into src; if S_ISDIR(mode): reset i_size = 0.
6. btrfs_release_path; wc.subvol_path.skip_release_on_error = true.
7. btrfs_insert_empty_item(trans, wc.root, wc.subvol_path, &wc.log_key, item_size).
8. if EEXIST/EOVERFLOW: btrfs_truncate_item / btrfs_extend_item to match.
9. write_extent_buffer src_ptr → dst_ptr.
10. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `log_mutex_held_during_log_inode` | INVARIANT | per-log_inode: inode.log_mutex held throughout. |
| `log_writers_balanced` | INVARIANT | per-start_trans/end_trans: log_writers refcount get/put balanced. |
| `log_commit_slot_owned_after_atomic_set` | INVARIANT | per-sync: atomic_set(log_commit[index],1) only after observing prior 0. |
| `log_transid_monotonic` | INVARIANT | per-root: log_transid strictly increases at each sync. |
| `log_root_tree_node_ptr_pinned_during_replay` | INVARIANT | per-recover: PinOnly pass pins all log nodes before ReplayInodes. |
| `force_commit_sticky` | INVARIANT | per-trans: once btrfs_set_log_full_commit, btrfs_need_log_full_commit returns true until trans ends. |
| `super_log_root_written_after_extents_flushed` | INVARIANT | per-sync: super.log_root only set after wait_tree_log_extents returned 0. |

### Layer 2: TLA+

`fs/btrfs/tree-log.tla`:
- States: per-root {No-log, Log-open, Log-committing(index), Committed(transid)}.
- Per-log_root_tree mirror states.
- Per-pid-rotation log_commit[2] semaphore.
- Properties:
  - `safety_log_transid_monotonic` — per-root: log_transid only grows.
  - `safety_two_phase_commit_atomic` — per-sync: super.log_root updated iff per-log + per-log_root_tree extents flushed.
  - `safety_replay_idempotent` — per-recover: re-running replay produces same subvol state.
  - `safety_pin_before_replay` — per-recover: every log extent pinned before any replay mutates subvol tree.
  - `safety_force_commit_drains` — per-trans: BTRFS_LOG_FORCE_COMMIT ⟹ no log sync occurs, all fsync waiters escalate to full commit.
  - `liveness_fsync_completes` — per-fsync: eventually either log sync succeeds or full commit drains.
  - `liveness_replay_terminates` — per-recover: bounded by # of per-root logs × 4 stages.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Log::start_trans` post: log_writers incremented ∧ ctx on root.log_ctxs[transid%2] (unless logging_new_name) | `Log::start_trans` |
| `Log::log_inode` post: inode.logged_trans == trans.transid (success path) | `Log::log_inode` |
| `Log::log_inode` post: NEEDS_FULL_SYNC ⟹ drop_inode_items or truncate_inode_items executed | `Log::log_inode` |
| `Log::sync_log` post: super.log_root == log_root_tree.node.bytenr ∧ disk-flushed | `Log::sync_log` |
| `Log::sync_log` post: root.log_transid_committed == log_transid (success) | `Log::sync_log` |
| `Log::recover_log_trees` post: super.log_root cleared ∧ per-root .log_root == NULL | `Log::recover_log_trees` |
| `Log::overwrite_item` post: subvol slot for log_key has identical bytes to log_leaf slot | `Log::overwrite_item` |
| `Log::fixup_inode_link_counts` post: ∀orphans → btrfs_orphan_add called | `Log::fixup_inode_link_counts` |

### Layer 4: Verus/Creusot functional

`fsync(F) → start_log_trans → log_inode(F, LOG_INODE_ALL) → log_inode_parent(ancestors, EXISTS) → sync_log → super.log_root durable` is equivalent under crash-injection model to `mount → recover_log_trees → replay PinOnly → ReplayInodes → ReplayDirIndex → ReplayAll → subvol state contains F`. Per `Documentation/filesystems/btrfs.rst` and the on-disk format reference.

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

Tree-log reinforcement:

- **Per-log_mutex serialization** — defense against per-concurrent-log corruption.
- **Per-log_commit[2] rotation** — defense against per-writer-vs-committer race on a single side.
- **Per-PinOnly precedes Replay** — defense against per-replay-overwriting-log block reuse.
- **Per-log_writers refcount strict get/put** — defense against per-log-freed-while-writing.
- **Per-set_log_full_commit sticky** — defense against per-stale-log-after-rename / unsafe replay.
- **Per-NEEDS_FULL_SYNC forced drop+truncate** — defense against per-stale-extent leaking into replay.
- **Per-symlink LOG_INODE_ALL forced** — defense against per-empty-symlink after replay.
- **Per-subvol-missing replay skip** — defense against per-deleted-subvol replay crash.
- **Per-orphan fixup-dir + btrfs_orphan_add** — defense against per-link-count drift across crash.
- **Per-zoned wait on opposite-index commit** — defense against per-zoned-OOO write (sequential-only zones).
- **Per-super.log_root cleared on successful mount-replay** — defense against per-double-replay.
- **Per-walk_log_tree level monotonic descent** — defense against per-malformed-log loop.
- **Per-skip-CoW on memcmp-identical overwrite** — defense against per-needless-write amplification.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy.
- **PAX_KERNEXEC** — W^X for any executable mapping.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization.
- **PAX_REFCOUNT** — saturating refcount on subsystem structs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for sensitive allocations.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on vtables.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding.
- **GRKERNSEC_DMESG** — syslog restriction.
- **Log-replay PAX_USERCOPY** — replay path reads on-disk log records that originated as inline-data from user buffers; per-key payload copy is bounds-checked against `eb->len` before being applied into the live subvolume.
- **log_mutex serialization** — per-root `log_mutex` is the only path mutating the log tree; PAX_REFCOUNT on `log_writers` saturates rather than wraps to prevent log-freed-while-writing UAF.
- **PAX_MEMORY_SANITIZE on log_root** — `btrfs_root` for the log is zeroed on `btrfs_free_log_root_tree` so reuse during a subsequent transaction cannot leak stale tree pointers.
- **kCFI on walk_log_tree callbacks** — `replay_one_buffer` and `process_func` dispatched via kCFI signatures so a malformed log header cannot redirect the walker.
- **GRKERNSEC_HIDESYM on log_full_commit** — sticky-fail-commit logging suppresses fs_info pointer disclosure.
- **PAX_USERCOPY on btrfs_log_inode_parent ioctl path** — bounded copy through fsync caller chain.
- **PAX_REFCOUNT on subvol root during replay** — each per-subvol `btrfs_root` referenced by a log entry is acquired through saturating refcount so a malicious log cannot decrement a freed root.

Per-doc rationale: the tree-log is the fsync-fast-path log that must survive replay through `replay_one_buffer` walkers, log_mutex-serialized writers, and PinOnly→Replay ordering; user payloads can enter the log via inline-data, so bounded usercopy on replay, refcount saturation on `log_writers` and per-subvol roots, and kCFI on the walk callbacks are required to keep a crafted log from gaining code execution at mount time.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/btrfs/transaction.c full-transaction commit semantics (covered in `transaction.md` Tier-3).
- fs/btrfs/disk-io.c log_root_tree allocation primitives (covered in `disk-io.md` Tier-3).
- fs/btrfs/extent-tree.c log-block reservation / pinning (covered in `extent-tree.md` Tier-3).
- fs/btrfs/delayed-inode.c delayed-item flushing (covered separately if expanded).
- fs/btrfs/zoned.c zoned-device sequential-write constraints (covered separately if expanded).
- Implementation code.
