# Tier-3: fs/btrfs/qgroup.c — btrfs quota groups (qgroup)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/00-overview.md
upstream-paths:
  - fs/btrfs/qgroup.c (~4959 lines)
  - fs/btrfs/qgroup.h
  - include/uapi/linux/btrfs_tree.h (BTRFS_QGROUP_LEVEL_SHIFT, QGROUP_STATUS/INFO/LIMIT/RELATION keys, status flags)
  - include/uapi/linux/btrfs.h (struct btrfs_ioctl_quota_ctl_args, struct btrfs_qgroup_inherit, btrfs_qgroup_limit)
-->

## Summary

The btrfs **quota groups** (qgroups) subsystem accounts referenced / exclusive / compressed / reserved bytes per subvolume (and across hierarchies of subvolumes) and optionally enforces per-qgroup limits at `EDQUOT`. qgroup IDs are 64-bit and shaped: the high 16 bits carry the level (`qgroupid >> BTRFS_QGROUP_LEVEL_SHIFT`, shift = 48) and the low 48 bits are the subvol-id; level-0 qgroups have `qgroupid == subvol_id` and there is exactly one level-0 qgroup per subvolume. Higher levels (1..N) are aggregator qgroups configured by the admin. Per-qgroup state: `struct btrfs_qgroup { qgroupid, rfer, excl, rfer_cmpr, excl_cmpr, rsv (BTRFS_QGROUP_RSV_DATA/META_PERTRANS/META_PREALLOC), lim_flags, max_rfer, max_excl, ... }` indexed by `fs_info->qgroup_tree` rbtree; parent↔child wiring via `struct btrfs_qgroup_list { group, member, next_group, next_member }`. On-disk layout: a dedicated **quota tree** (`BTRFS_QUOTA_TREE_OBJECTID = 8`) holding `BTRFS_QGROUP_STATUS_KEY`, per-qgroup `_INFO_KEY` and `_LIMIT_KEY`, and parent↔child `_RELATION_KEY`. Two operating modes selected at enable-time: **full quotas** (delta-accounting via backref-walk in `btrfs_qgroup_account_extents` at commit) and **simple quotas** (decoupled, append-only accounting via per-extent owner stamp, no backref walk, no rescan); selected by `BTRFS_QGROUP_STATUS_FLAG_SIMPLE_MODE` and `BTRFS_FEATURE_INCOMPAT_SIMPLE_QUOTA`. Critical for: container/tenant accounting, snapshot accounting (shared vs exclusive), enforced multi-tenant disk-usage caps.

This Tier-3 covers `fs/btrfs/qgroup.c` (~4959 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct btrfs_qgroup` | per-qgroup state | `Qgroup` |
| `struct btrfs_qgroup_list` | per-relation parent↔member | `QgroupList` |
| `struct btrfs_qgroup_rsv` | per-qgroup reservation tuple | `QgroupRsv` |
| `struct btrfs_qgroup_extent_record` | per-dirty-extent record | `QgroupExtentRecord` |
| `struct btrfs_qgroup_swapped_block` | per-relocation swap | `QgroupSwappedBlock` |
| `struct btrfs_squota_delta` | per-simple-quota delta | `SquotaDelta` |
| `enum btrfs_qgroup_mode` | DISABLED/FULL/SIMPLE | `QgroupMode` |
| `btrfs_qgroup_mode` / `_enabled` / `_full_accounting` | per-mode predicates | `Qgroup::mode` / `enabled` / `full_accounting` |
| `btrfs_qgroup_subvolid` | per-id low-48 extractor | `Qgroup::subvolid_of` |
| `btrfs_quota_enable` | per-mount enable | `Qgroup::quota_enable` |
| `btrfs_quota_disable` | per-mount disable | `Qgroup::quota_disable` |
| `btrfs_read_qgroup_config` | per-mount load from quota tree | `Qgroup::read_config` |
| `btrfs_free_qgroup_config` | per-umount drop | `Qgroup::free_config` |
| `find_qgroup_rb` / `add_qgroup_rb` / `del_qgroup_rb` | per-rbtree lookup/add/del | `Qgroup::find_rb` / `add_rb` / `del_rb` |
| `add_relation_rb` / `del_relation_rb` | per-relation rbtree update | `Qgroup::add_relation_rb` / `del_relation_rb` |
| `btrfs_create_qgroup` / `btrfs_remove_qgroup` | per-ioctl create/remove | `Qgroup::create` / `remove` |
| `btrfs_add_qgroup_relation` / `btrfs_del_qgroup_relation` | per-ioctl parent↔child | `Qgroup::add_relation` / `del_relation` |
| `btrfs_limit_qgroup` | per-ioctl limit set | `Qgroup::limit` |
| `btrfs_qgroup_trace_extent` | per-delayed-ref hook | `Qgroup::trace_extent` |
| `btrfs_qgroup_trace_extent_nolock` / `_post` | per-record alloc + backref-search | `Qgroup::trace_extent_nolock` / `post` |
| `btrfs_qgroup_trace_leaf_items` | per-CoW leaf scan | `Qgroup::trace_leaf_items` |
| `btrfs_qgroup_trace_subtree` | per-snapshot subtree scan | `Qgroup::trace_subtree` |
| `btrfs_qgroup_account_extent` | per-extent accounting (old/new roots) | `Qgroup::account_extent` |
| `btrfs_qgroup_account_extents` | per-commit drain dirty_extents xa | `Qgroup::account_extents` |
| `qgroup_update_refcnt` | per-refcnt scan over qgroup tree | `Qgroup::update_refcnt` |
| `qgroup_update_counters` | per-rfer/excl delta engine | `Qgroup::update_counters` |
| `btrfs_run_qgroups` | per-commit write dirty qgroups | `Qgroup::run` |
| `update_qgroup_info_item` / `_limit_item` / `_status_item` | per-disk update | `Qgroup::update_info_item` / `_limit_item` / `_status_item` |
| `btrfs_qgroup_inherit` | per-snapshot inherit | `Qgroup::inherit` |
| `qgroup_auto_inherit` | per-create auto-inherit policy | `Qgroup::auto_inherit` |
| `qgroup_snapshot_quick_inherit` | per-simple-quota fast inherit | `Qgroup::snapshot_quick_inherit` |
| `qgroup_reserve` | per-write reservation w/ enforce | `Qgroup::reserve` |
| `btrfs_qgroup_reserve_data` / `_release_data` / `_free_data` | per-write data reservation | `Qgroup::reserve_data` / `release_data` / `free_data` |
| `btrfs_qgroup_reserve_meta_prealloc` / `_free_meta_prealloc` / `_convert_reserved_meta` | per-meta reservation | `Qgroup::reserve_meta_prealloc` / `free_meta_prealloc` / `convert_reserved_meta` |
| `btrfs_qgroup_free_meta_all_pertrans` | per-trans-end drain | `Qgroup::free_meta_all_pertrans` |
| `btrfs_qgroup_free_refroot` | per-root rsv release | `Qgroup::free_refroot` |
| `btrfs_qgroup_check_reserved_leak` | per-inode leak audit | `Qgroup::check_reserved_leak` |
| `qgroup_rescan_init` / `_worker` / `qgroup_rescan_leaf` | per-rescan kthread | `Qgroup::rescan_*` |
| `btrfs_qgroup_rescan` / `_resume` / `_wait_for_completion` | per-ioctl rescan | `Qgroup::rescan` / `resume` / `wait` |
| `btrfs_qgroup_check_inherit` | per-inherit validation | `Qgroup::check_inherit` |
| `btrfs_record_squota_delta` | per-simple-quota delta apply | `Qgroup::record_squota_delta` |
| `btrfs_qgroup_init_swapped_blocks` / `_clean_swapped_blocks` / `_add_swapped_blocks` | per-relocation tracking | `Qgroup::swapped_*` |
| `btrfs_qgroup_trace_subtree_after_cow` | per-relocation CoW completion | `Qgroup::trace_subtree_after_cow` |
| `qgroup_mark_inconsistent` | per-error mark INCONSISTENT | `Qgroup::mark_inconsistent` |
| `BTRFS_QGROUP_LEVEL_SHIFT` (= 48) | per-id level extractor shift | `LEVEL_SHIFT` const |
| `BTRFS_QGROUP_STATUS_FLAG_ON / _RESCAN / _INCONSISTENT / _SIMPLE_MODE` | per-status flag bits | `QgroupStatusFlag` |
| `BTRFS_QUOTA_TREE_OBJECTID` (= 8) | per-on-disk quota tree | shared |

## Compatibility contract

REQ-1: qgroup ID shape (per BTRFS_QGROUP_LEVEL_SHIFT = 48):
- High 16 bits: level (0..65535).
- Low 48 bits: subvol-id.
- Level-0 qgroup: qgroupid == subvol-id, exactly one per subvolume, automatically created at enable-time and at subvol/snapshot creation.
- Helper: btrfs_qgroup_subvolid(qid) = qid & ((1ULL << 48) - 1).
- Helper: btrfs_qgroup_level(qid) = (u16)(qid >> 48).

REQ-2: struct btrfs_qgroup (per-qgroup state):
- qgroupid: u64 (level<<48 | subvol_id).
- rfer / rfer_cmpr: referenced bytes (and compressed equivalent).
- excl / excl_cmpr: exclusive bytes (bytes referenced by exactly one fs root through this qgroup).
- rsv: struct btrfs_qgroup_rsv { values[BTRFS_QGROUP_RSV_LAST] } per-type reservation (DATA, META_PERTRANS, META_PREALLOC).
- lim_flags: per-bitmap of which limits set (MAX_RFER, MAX_EXCL, RSV_RFER, RSV_EXCL).
- max_rfer / max_excl / rsv_rfer / rsv_excl: per-limit values.
- groups: list of btrfs_qgroup_list this qgroup is a *member* of (parents).
- members: list of btrfs_qgroup_list this qgroup is a *group* of (children).
- dirty: per-qgroup linkage on fs_info.dirty_qgroups.
- iterator / nested_iterator: per-walk scratch list-head.
- node: rbnode in fs_info.qgroup_tree.
- old_refcnt / new_refcnt: per-account-extent scratch.
- kobj: per-sysfs entry.

REQ-3: struct btrfs_qgroup_list (per-edge):
- next_group: linkage on parent.members.
- next_member: linkage on member.groups.
- group: parent qgroup.
- member: child qgroup.

REQ-4: enum btrfs_qgroup_mode:
- BTRFS_QGROUP_MODE_DISABLED: quotas off; no quota_root.
- BTRFS_QGROUP_MODE_FULL: full accounting via backref-walk; rescan on enable.
- BTRFS_QGROUP_MODE_SIMPLE: simple quotas; per-extent owner stamp; no rescan; INCOMPAT_SIMPLE_QUOTA bit set.

REQ-5: btrfs_quota_enable(fs_info, args):
- lockdep_assert_held_write(&fs_info.subvol_sem).
- /* extent-tree-v2 incompatible */
- if btrfs_fs_incompat(fs_info, EXTENT_TREE_V2): return -EINVAL.
- mutex_lock(&fs_info.qgroup_ioctl_lock).
- if fs_info.quota_root != NULL: goto out (already on).
- btrfs_sysfs_add_qgroups(fs_info).
- mutex_unlock(&fs_info.qgroup_ioctl_lock) /* avoid lock-inv with vfs freeze sem */.
- trans = btrfs_start_transaction(fs_info.tree_root, 2).
- mutex_lock(&fs_info.qgroup_ioctl_lock).
- /* Create the quota tree */
- quota_root = btrfs_create_tree(trans, BTRFS_QUOTA_TREE_OBJECTID).
- /* Insert BTRFS_QGROUP_STATUS_KEY (0, 240, 0) */
- btrfs_insert_empty_item(trans, quota_root, &(0, STATUS_KEY, 0), sizeof(qgroup_status_item)).
- btrfs_set_qgroup_status_generation(...trans.transid).
- btrfs_set_qgroup_status_version(BTRFS_QGROUP_STATUS_VERSION = 1).
- fs_info.qgroup_flags = BTRFS_QGROUP_STATUS_FLAG_ON.
- if simple: fs_info.qgroup_flags |= SIMPLE_MODE; btrfs_set_fs_incompat(fs_info, SIMPLE_QUOTA); set_qgroup_status_enable_gen(trans.transid).
- else: fs_info.qgroup_flags |= INCONSISTENT.
- btrfs_set_qgroup_status_flags(... & STATUS_FLAGS_MASK).
- btrfs_set_qgroup_status_rescan(...0).
- /* Walk tree_root's BTRFS_ROOT_REF_KEY entries: for each existing subvolume add a qgroup_item + insert in rbtree */
- for_each (found_key.type == BTRFS_ROOT_REF_KEY): add_qgroup_item(trans, quota_root, found_key.offset); add_qgroup_rb(fs_info, prealloc, found_key.offset); btrfs_sysfs_add_one_qgroup.
- /* Always add FS_TREE_OBJECTID (the default subvol) */
- add_qgroup_item(trans, quota_root, BTRFS_FS_TREE_OBJECTID); add_qgroup_rb.
- fs_info.qgroup_enable_gen = trans.transid.
- mutex_unlock(&fs_info.qgroup_ioctl_lock).
- btrfs_commit_transaction(trans).
- mutex_lock(&fs_info.qgroup_ioctl_lock).
- spin_lock(&fs_info.qgroup_lock); fs_info.quota_root = quota_root; set_bit(BTRFS_FS_QUOTA_ENABLED, &fs_info.flags); spin_unlock.
- /* Simple-quotas: no rescan */
- if mode == SIMPLE: goto out.
- qgroup_rescan_init(fs_info, 0, 1); qgroup_rescan_zero_tracking; fs_info.qgroup_rescan_running = true; btrfs_queue_work(qgroup_rescan_workers, &qgroup_rescan_work).
- mutex_unlock(&fs_info.qgroup_ioctl_lock).
- return 0.

REQ-6: btrfs_quota_disable(fs_info):
- lockdep_assert_held_write(&fs_info.subvol_sem); lockdep_assert_held(&fs_info.cleaner_mutex).
- mutex_lock(&fs_info.qgroup_ioctl_lock); if !fs_info.quota_root: goto out; mutex_unlock.
- clear_bit(BTRFS_FS_QUOTA_ENABLED, &fs_info.flags).
- btrfs_qgroup_wait_for_completion(fs_info, false) /* drain rescan worker */.
- flush_reservations(fs_info) /* start_delalloc_roots + wait_ordered_roots + commit */.
- trans = btrfs_start_transaction(fs_info.tree_root, 1).
- mutex_lock(&fs_info.qgroup_ioctl_lock).
- spin_lock(&fs_info.qgroup_lock); quota_root = fs_info.quota_root; fs_info.quota_root = NULL; fs_info.qgroup_flags &= ~(FLAG_ON | FLAG_SIMPLE_MODE); fs_info.qgroup_drop_subtree_thres = BTRFS_QGROUP_DROP_SUBTREE_THRES_DEFAULT; spin_unlock.
- btrfs_free_qgroup_config(fs_info) /* drop rbtree + relations */.
- btrfs_clean_quota_tree(trans, quota_root) /* iterate all keys, delete */.
- btrfs_del_root(trans, &quota_root.root_key).
- spin_lock(&fs_info.trans_lock); list_del(&quota_root.dirty_list); spin_unlock.
- btrfs_tree_lock(quota_root.node); btrfs_clear_buffer_dirty; btrfs_tree_unlock.
- btrfs_free_tree_block(trans, btrfs_root_id(quota_root), quota_root.node, 0, 1).
- btrfs_commit_transaction(trans).
- return 0.

REQ-7: btrfs_read_qgroup_config(fs_info):
- /* Called at mount when BTRFS_FS_QUOTA_ENABLED suggested by super or quota_root present */
- Walk quota_root reading:
  - STATUS_KEY: set fs_info.qgroup_flags ∧ MASK; if simple, populate enable_gen.
  - INFO_KEY: add_qgroup_rb(qgroupid); populate rfer/excl/cmpr.
  - LIMIT_KEY: populate lim_flags + max_rfer/max_excl/rsv_rfer/rsv_excl.
  - RELATION_KEY: add_relation_rb(child=key.objectid, parent=key.offset).
- Mark INCONSISTENT if any structural issue.

REQ-8: btrfs_qgroup_account_extent(trans, bytenr, num_bytes, old_roots, new_roots):
- Only when mode == FULL ∧ !RUNTIME_FLAG_NO_ACCOUNTING.
- maybe_fs_roots(new_roots) ∧ maybe_fs_roots(old_roots) — else skip.
- nr_old_roots = old_roots->nnodes; nr_new_roots = new_roots->nnodes.
- Quick-exit on nr_old + nr_new == 0.
- Honor in-flight rescan: if RESCAN ∧ rescan_progress.objectid <= bytenr: skip (will be picked up by rescan).
- spin_lock(&fs_info.qgroup_lock). seq = fs_info.qgroup_seq.
- qgroup_update_refcnt(fs_info, old_roots, &qgroups, seq, update_old=true) /* sets old_refcnt on every qgroup reachable upward */.
- qgroup_update_refcnt(fs_info, new_roots, &qgroups, seq, update_old=false) /* sets new_refcnt */.
- qgroup_update_counters(fs_info, &qgroups, nr_old_roots, nr_new_roots, num_bytes, seq).
- qgroup_iterator_nested_clean.
- fs_info.qgroup_seq += max(nr_old_roots, nr_new_roots) + 1.
- spin_unlock(&fs_info.qgroup_lock).
- ulist_free(old_roots); ulist_free(new_roots).

REQ-9: qgroup_update_counters (per-qg update engine):
- Per qgroup qg in nested_iterator list:
  - cur_old = btrfs_qgroup_get_old_refcnt(qg, seq); cur_new = btrfs_qgroup_get_new_refcnt(qg, seq).
  - /* RFER update */
  - if cur_old == 0 ∧ cur_new > 0: qg.rfer += num_bytes; qg.rfer_cmpr += num_bytes; dirty = true.
  - if cur_old > 0 ∧ cur_new == 0: qg.rfer -= num_bytes; qg.rfer_cmpr -= num_bytes; dirty = true.
  - /* EXCL update (per-truth-table per A/B) */
  - if cur_old == nr_old_roots ∧ cur_new < nr_new_roots ∧ cur_old != 0: qg.excl -= num_bytes; qg.excl_cmpr -= num_bytes; dirty = true.
  - if cur_old < nr_old_roots ∧ cur_new == nr_new_roots ∧ cur_new != 0: qg.excl += num_bytes; qg.excl_cmpr += num_bytes; dirty = true.
  - if cur_old == nr_old_roots ∧ cur_new == nr_new_roots:
    - if cur_old == 0 ∧ cur_new != 0: qg.excl += num_bytes; dirty.
    - if cur_old != 0 ∧ cur_new == 0: qg.excl -= num_bytes; dirty.
  - if dirty: qgroup_dirty(fs_info, qg) /* link on dirty_qgroups */.

REQ-10: btrfs_qgroup_account_extents(trans):
- mode == SIMPLE ⟹ return 0 (simple-quotas accounted at delayed-ref-run time).
- For each record in trans.transaction.delayed_refs.dirty_extents (xarray):
  - bytenr = index << fs_info.sectorsize_bits.
  - Search current roots: btrfs_find_all_roots(&ctx, time_seq=BTRFS_SEQ_LAST) → new_roots.
  - If !record.old_roots: search commit_root via btrfs_find_all_roots(&ctx, false) → old_roots.
  - If qgroup_to_skip is set on trans (e.g., per-snapshot exclusion): ulist_del(qgroup_to_skip).
  - btrfs_qgroup_account_extent(trans, bytenr, record.num_bytes, record.old_roots, new_roots).
  - btrfs_qgroup_free_refroot(fs_info, record.data_rsv_refroot, record.data_rsv, BTRFS_QGROUP_RSV_DATA).
  - xa_erase + kfree(record).

REQ-11: btrfs_run_qgroups(trans):
- Called in commit path (and from qgroup ioctl) — must hold qgroup_ioctl_lock if not in TRANS_STATE_COMMIT_DOING.
- if !fs_info.quota_root: return 0.
- spin_lock(&fs_info.qgroup_lock).
- Drain fs_info.dirty_qgroups: for each qg: spin_unlock; update_qgroup_info_item(trans, qg); update_qgroup_limit_item(trans, qg); spin_lock.
- Reconcile fs_info.qgroup_flags & FLAG_ON to enabled().
- spin_unlock.
- update_qgroup_status_item(trans).
- On any error: qgroup_mark_inconsistent(fs_info, "...") → sets INCONSISTENT flag.

REQ-12: qgroup_reserve(root, num_bytes, enforce, type):
- ref_root = btrfs_root_id(root).
- if !btrfs_is_fstree(ref_root): return 0 (only fs trees).
- if num_bytes == 0: return 0.
- if BTRFS_FS_QUOTA_OVERRIDE ∧ capable(CAP_SYS_RESOURCE): enforce = false.
- spin_lock(&fs_info.qgroup_lock). if !fs_info.quota_root: return 0.
- qgroup = find_qgroup_rb(fs_info, ref_root); if !qgroup: return 0.
- /* Iterate upward across parents collecting all qgroups */
- list_for_each_entry(qgroup, qgroup_list, iterator):
  - if enforce ∧ !qgroup_check_limits(qgroup, num_bytes): ret = -EDQUOT; goto out.
  - list_for_each_entry(glist, &qgroup.groups, next_group): qgroup_iterator_add(&qgroup_list, glist.group).
- /* No limits exceeded — apply */
- list_for_each_entry(qgroup, qgroup_list, iterator): qgroup_rsv_add(fs_info, qgroup, num_bytes, type).
- spin_unlock(&fs_info.qgroup_lock).
- return 0.

REQ-13: qgroup_check_limits(qg, num_bytes):
- if (lim_flags & MAX_RFER) ∧ rsv_total(qg) + (s64)qg.rfer + num_bytes > qg.max_rfer: return false.
- if (lim_flags & MAX_EXCL) ∧ rsv_total(qg) + (s64)qg.excl + num_bytes > qg.max_excl: return false.
- return true.

REQ-14: btrfs_qgroup_inherit(trans, srcid, objectid, inode_rootid, inherit):
- Called from snapshot-create / subvolume-create paths.
- mode == DISABLED ⟹ return 0.
- Create new level-0 qgroup for objectid (the new subvol-id).
- if inherit supplied: apply explicit parent list via add_qgroup_relation_item + add_relation_rb for each (objectid → inherit.qgroups[i]).
- else: qgroup_auto_inherit(fs_info, srcid, objectid) — copy src's parents.
- For full quotas: copy src qgroup's rfer/excl into new qgroup (snapshot-time exclusive becomes shared).
- For simple quotas: qgroup_snapshot_quick_inherit — no backref walk; new qgroup starts with src.rfer copied to rfer; new excl = 0.
- Mark dirty; qgroup_run later writes to disk.

REQ-15: simple quotas vs full quotas:
- Full: BTRFS_QGROUP_MODE_FULL; account on every commit by walking backrefs (btrfs_find_all_roots) for every dirty extent; supports nested qgroup hierarchy with rfer/excl truth-table; requires rescan after enable; can become INCONSISTENT under tree-reloc / extent-tree-v2.
- Simple: BTRFS_QGROUP_MODE_SIMPLE + INCOMPAT_SIMPLE_QUOTA; on every extent allocation, owner subvol is stamped on the extent item itself; accounting is append-only (deltas applied at delayed-ref-run via btrfs_record_squota_delta); no backref walk; no rescan; no excl tracking (only rfer). Defined by `BTRFS_QGROUP_STATUS_FLAG_SIMPLE_MODE`.
- Mode transitions: enable-time only (per quota_ctl_args.cmd ∈ {ENABLE, ENABLE_SIMPLE_QUOTA}). Disabling and re-enabling in a different mode is supported but requires full rescan in FULL mode.

REQ-16: btrfs_qgroup_rescan (worker):
- qgroup_rescan_init: validate state; set RESCAN flag; clear INCONSISTENT.
- qgroup_rescan_zero_tracking: reset all rfer/excl to 0.
- qgroup_rescan_worker (workqueue): iterate extent-tree leaves; for each extent: btrfs_find_all_roots(&ctx, false) → call btrfs_qgroup_account_extent(NULL_old, computed_new) under a fresh transaction.
- qgroup_rescan_leaf: process one leaf, advance rescan_progress.objectid.
- Stop on rescan_should_stop / unmount / error.
- After successful completion: clear RESCAN flag; commit.

REQ-17: btrfs_qgroup_create / _remove / _add_relation / _del_relation:
- create(trans, qgroupid): require quota enabled; add_qgroup_item to quota_tree; add_qgroup_rb in memory; sysfs_add_one_qgroup.
- remove(trans, qgroupid): can_delete_qgroup gate (no children, no level-0 of live subvol unless dropped); del_qgroup_item; del_qgroup_rb; sysfs remove.
- add_relation(trans, src, dst, prealloc): add_qgroup_relation_item × 2 (both directions); __add_relation_rb.
- del_relation(trans, src, dst): del_qgroup_relation_item × 2; del_relation_rb.

REQ-18: BTRFS_QGROUP_STATUS_FLAG_*:
- ON (1<<0): quotas enabled.
- RESCAN (1<<1): rescan in progress.
- INCONSISTENT (1<<2): accounting may be off; rescan recommended.
- SIMPLE_MODE (1<<3): simple-quota mode.
- MASK = ON | RESCAN | INCONSISTENT | SIMPLE_MODE.
- VERSION = 1.

## Acceptance Criteria

- [ ] AC-1: btrfs_quota_enable(FULL): quota_root created; STATUS_KEY persisted; level-0 qgroups added for every subvol; rescan queued.
- [ ] AC-2: btrfs_quota_enable(SIMPLE): INCOMPAT_SIMPLE_QUOTA set; SIMPLE_MODE bit set; no rescan queued.
- [ ] AC-3: btrfs_quota_disable: quota_root removed; rbtree freed; all rsv accounting flushed; FS_QUOTA_ENABLED bit cleared.
- [ ] AC-4: subvolid_of(qid) == qid & ((1<<48)-1); level-0 qgroupid == subvolid.
- [ ] AC-5: New write to subvol S consumes from S's level-0 qgroup rsv via qgroup_reserve(RSV_DATA).
- [ ] AC-6: Limit max_rfer exceeded: qgroup_reserve returns -EDQUOT (enforce=true).
- [ ] AC-7: CAP_SYS_RESOURCE + FS_QUOTA_OVERRIDE: enforce silently disabled in qgroup_reserve.
- [ ] AC-8: btrfs_qgroup_account_extent for shared-to-exclusive transition: excl += num_bytes; rfer unchanged.
- [ ] AC-9: btrfs_qgroup_account_extent for exclusive-to-shared transition: excl -= num_bytes; rfer unchanged.
- [ ] AC-10: snapshot of S: new level-0 qgroup inherits S's parents (auto-inherit); both subvols' excl appropriately recomputed.
- [ ] AC-11: btrfs_run_qgroups at commit: dirty_qgroups drained; STATUS_KEY rewritten.
- [ ] AC-12: Simple-quotas: no backref walk in commit; btrfs_record_squota_delta updates owner-stamped qgroup only.
- [ ] AC-13: Rescan iterates entire extent tree; final rfer/excl reconstructed; INCONSISTENT cleared on completion.
- [ ] AC-14: qgroup_mark_inconsistent on update_qgroup_info_item failure sets INCONSISTENT flag.
- [ ] AC-15: Adding a relation cycle is rejected (cycle detection during add_relation_rb gate).

## Architecture

```
struct Qgroup {
  qgroupid: u64,                          // level<<48 | subvol_id
  rfer: u64, rfer_cmpr: u64,
  excl: u64, excl_cmpr: u64,
  lim_flags: u64,
  max_rfer: u64, max_excl: u64,
  rsv_rfer: u64, rsv_excl: u64,
  rsv: QgroupRsv { values: [u64; 3] },    // DATA, META_PERTRANS, META_PREALLOC
  groups: ListHead,                       // members of this qgroup -> parents
  members: ListHead,                      // children
  dirty: ListHead,                        // fs_info.dirty_qgroups linkage
  iterator: ListHead,
  nested_iterator: ListHead,
  node: RbNode,                           // fs_info.qgroup_tree
  old_refcnt: u64, new_refcnt: u64,
  kobj: Kobject,
}

struct QgroupList {
  next_group: ListHead,   // parent.members
  next_member: ListHead,  // member.groups
  group: *Qgroup,
  member: *Qgroup,
}

enum QgroupMode { Disabled, Full, Simple }
enum QgroupRsvType { Data, MetaPertrans, MetaPrealloc }
```

`Qgroup::quota_enable(fs_info, args) -> i32`:
1. lockdep_assert_held_write(&fs_info.subvol_sem).
2. if btrfs_fs_incompat(fs_info, EXTENT_TREE_V2): return -EINVAL.
3. mutex_lock(&fs_info.qgroup_ioctl_lock).
4. if fs_info.quota_root: goto out (idempotent).
5. btrfs_sysfs_add_qgroups(fs_info).
6. mutex_unlock(&fs_info.qgroup_ioctl_lock).
7. trans = btrfs_start_transaction(fs_info.tree_root, 2).
8. mutex_lock(&fs_info.qgroup_ioctl_lock).
9. quota_root = btrfs_create_tree(trans, BTRFS_QUOTA_TREE_OBJECTID).
10. btrfs_insert_empty_item(quota_root, (0, STATUS_KEY, 0)).
11. set status_generation = trans.transid; status_version = 1; flags = ON; if simple: flags |= SIMPLE_MODE + set_fs_incompat(SIMPLE_QUOTA); else flags |= INCONSISTENT.
12. set_qgroup_status_flags(...& MASK). set_qgroup_status_rescan(0).
13. for_each tree_root entry with type == ROOT_REF_KEY: add_qgroup_item(quota_root, subvol_id); add_qgroup_rb; sysfs_add_one_qgroup.
14. add_qgroup_item(quota_root, BTRFS_FS_TREE_OBJECTID); add_qgroup_rb.
15. fs_info.qgroup_enable_gen = trans.transid.
16. mutex_unlock(&fs_info.qgroup_ioctl_lock).
17. btrfs_commit_transaction(trans).
18. mutex_lock(&fs_info.qgroup_ioctl_lock).
19. spin_lock(&fs_info.qgroup_lock); fs_info.quota_root = quota_root; set_bit(BTRFS_FS_QUOTA_ENABLED, &fs_info.flags); spin_unlock.
20. if Qgroup::mode(fs_info) == Simple: goto out (no rescan).
21. qgroup_rescan_init(fs_info, 0, 1); qgroup_rescan_zero_tracking; fs_info.qgroup_rescan_running = true; btrfs_queue_work(qgroup_rescan_workers, &qgroup_rescan_work).
22. mutex_unlock(&fs_info.qgroup_ioctl_lock).
23. return 0.

`Qgroup::quota_disable(fs_info) -> i32`:
1. lockdep_assert_held_write(&fs_info.subvol_sem). lockdep_assert_held(&fs_info.cleaner_mutex).
2. mutex_lock(&fs_info.qgroup_ioctl_lock).
3. if !fs_info.quota_root: mutex_unlock; return 0.
4. mutex_unlock(&fs_info.qgroup_ioctl_lock).
5. clear_bit(BTRFS_FS_QUOTA_ENABLED, &fs_info.flags).
6. btrfs_qgroup_wait_for_completion(fs_info, false).
7. flush_reservations(fs_info).
8. trans = btrfs_start_transaction(fs_info.tree_root, 1).
9. mutex_lock(&fs_info.qgroup_ioctl_lock).
10. spin_lock(&fs_info.qgroup_lock); quota_root = fs_info.quota_root; fs_info.quota_root = NULL; flags &= ~(ON | SIMPLE_MODE); spin_unlock.
11. btrfs_free_qgroup_config(fs_info).
12. btrfs_clean_quota_tree(trans, quota_root).
13. btrfs_del_root(trans, &quota_root.root_key).
14. spin_lock(&fs_info.trans_lock); list_del(&quota_root.dirty_list); spin_unlock.
15. btrfs_tree_lock(quota_root.node); btrfs_clear_buffer_dirty; btrfs_tree_unlock.
16. btrfs_free_tree_block(trans, btrfs_root_id(quota_root), quota_root.node, 0, 1).
17. btrfs_commit_transaction(trans).
18. return 0.

`Qgroup::account_extent(trans, bytenr, num_bytes, old_roots, new_roots) -> i32`:
1. if !Qgroup::full_accounting(fs_info) ∨ (fs_info.qgroup_flags & RUNTIME_FLAG_NO_ACCOUNTING): drop ulists; return 0.
2. if new_roots ∧ !maybe_fs_roots(new_roots): drop; return 0.
3. if old_roots ∧ !maybe_fs_roots(old_roots): drop; return 0.
4. nr_old = old_roots?.nnodes ?? 0; nr_new = new_roots?.nnodes ?? 0.
5. if nr_old + nr_new == 0: drop; return 0.
6. mutex_lock(&fs_info.qgroup_rescan_lock); if RESCAN ∧ rescan_progress.objectid <= bytenr: mutex_unlock; drop; return 0; mutex_unlock.
7. spin_lock(&fs_info.qgroup_lock). seq = fs_info.qgroup_seq.
8. Qgroup::update_refcnt(fs_info, old_roots, &qgroups, seq, update_old=true).
9. Qgroup::update_refcnt(fs_info, new_roots, &qgroups, seq, update_old=false).
10. Qgroup::update_counters(fs_info, &qgroups, nr_old, nr_new, num_bytes, seq).
11. qgroup_iterator_nested_clean(&qgroups).
12. fs_info.qgroup_seq += max(nr_old, nr_new) + 1.
13. spin_unlock(&fs_info.qgroup_lock).
14. ulist_free(old_roots); ulist_free(new_roots); return 0.

`Qgroup::account_extents(trans) -> i32`:
1. if Qgroup::mode(fs_info) == Simple: return 0.
2. delayed_refs = &trans.transaction.delayed_refs. qgroup_to_skip = delayed_refs.qgroup_to_skip.
3. xa_for_each(&delayed_refs.dirty_extents, index, record):
   - bytenr = (index << fs_info.sectorsize_bits).
   - if !(fs_info.qgroup_flags & RUNTIME_FLAG_NO_ACCOUNTING):
     - ctx = { bytenr, fs_info }.
     - if !record.old_roots: btrfs_find_all_roots(&ctx, false); record.old_roots = ctx.roots.
     - ctx.trans = trans; ctx.time_seq = BTRFS_SEQ_LAST.
     - btrfs_find_all_roots(&ctx, false); new_roots = ctx.roots.
     - if qgroup_to_skip != 0: ulist_del(new_roots, qgroup_to_skip, 0); ulist_del(record.old_roots, qgroup_to_skip, 0).
     - Qgroup::account_extent(trans, bytenr, record.num_bytes, record.old_roots, new_roots).
     - record.old_roots = NULL; new_roots = NULL.
   - btrfs_qgroup_free_refroot(fs_info, record.data_rsv_refroot, record.data_rsv, RSV_DATA).
   - ulist_free(record.old_roots); ulist_free(new_roots).
   - xa_erase(&delayed_refs.dirty_extents, index); kfree(record).
4. return ret.

`Qgroup::reserve(root, num_bytes, enforce, type) -> i32`:
1. ref_root = btrfs_root_id(root). if !btrfs_is_fstree(ref_root): return 0.
2. if num_bytes == 0: return 0.
3. if test_bit(BTRFS_FS_QUOTA_OVERRIDE, &fs_info.flags) ∧ capable(CAP_SYS_RESOURCE): enforce = false.
4. spin_lock(&fs_info.qgroup_lock). if !fs_info.quota_root: ret = 0; goto out.
5. qgroup = Qgroup::find_rb(fs_info, ref_root). if !qgroup: ret = 0; goto out.
6. qgroup_iterator_add(&qgroup_list, qgroup).
7. for each qg in qgroup_list:
   - if enforce ∧ !qgroup_check_limits(qg, num_bytes): ret = -EDQUOT; goto out.
   - for each glist in qg.groups: qgroup_iterator_add(&qgroup_list, glist.group).
8. for each qg in qgroup_list: qgroup_rsv_add(fs_info, qg, num_bytes, type).
9. ret = 0.
10. out: qgroup_iterator_clean(&qgroup_list); spin_unlock; return ret.

`Qgroup::inherit(trans, srcid, objectid, inode_rootid, inherit) -> i32`:
1. if Qgroup::mode(fs_info) == Disabled: return 0.
2. /* Always create level-0 qgroup for new subvol */
3. add_qgroup_item(trans, fs_info.quota_root, objectid); add_qgroup_rb.
4. if inherit != NULL: validate via btrfs_qgroup_check_inherit; for each parent_qid in inherit.qgroups: add_relation_item × 2; __add_relation_rb.
5. else: qgroup_auto_inherit(fs_info, srcid, objectid) — copy src's parent set.
6. mode == Simple: qgroup_snapshot_quick_inherit — set new.rfer = src.rfer; new.excl = 0; mark dirty.
7. mode == Full: copy src.rfer / src.excl into new; mark all affected qgroups (including parents) dirty.
8. return 0.

`Qgroup::run(trans) -> i32`:
1. if trans.transaction.state != TRANS_STATE_COMMIT_DOING: lockdep_assert_held(&fs_info.qgroup_ioctl_lock).
2. if !fs_info.quota_root: return 0.
3. spin_lock(&fs_info.qgroup_lock).
4. while !list_empty(&fs_info.dirty_qgroups):
   - qgroup = list_first_entry(...). list_del_init(&qgroup.dirty).
   - spin_unlock.
   - update_qgroup_info_item(trans, qgroup). on err: qgroup_mark_inconsistent.
   - update_qgroup_limit_item(trans, qgroup). on err: qgroup_mark_inconsistent.
   - spin_lock.
5. if Qgroup::enabled(fs_info): fs_info.qgroup_flags |= STATUS_FLAG_ON.
6. else: fs_info.qgroup_flags &= ~STATUS_FLAG_ON.
7. spin_unlock.
8. update_qgroup_status_item(trans). on err: qgroup_mark_inconsistent.
9. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `qgroupid_level_shape` | INVARIANT | per-id: btrfs_qgroup_subvolid(qid) | (qid & ~((1<<48)-1)) == qid (lossless split). |
| `level0_qgroupid_eq_subvolid` | INVARIANT | per-subvol: level-0 qgroup.qgroupid == subvol-id. |
| `quota_enable_idempotent` | INVARIANT | per-fs_info: quota_enable called twice ⟹ second is no-op (quota_root != NULL check). |
| `quota_root_consistency` | INVARIANT | per-fs_info: fs_info.quota_root != NULL ⟺ BTRFS_FS_QUOTA_ENABLED bit set ⟺ qgroup_flags & FLAG_ON. |
| `dirty_qgroups_drained_on_run` | INVARIANT | per-run: after btrfs_run_qgroups returns 0, dirty_qgroups list is empty. |
| `qgroup_lock_held_for_counters` | INVARIANT | per-account_extent: qgroup_lock held throughout update_refcnt + update_counters. |
| `rsv_total_le_max_when_enforced` | INVARIANT | per-reserve: enforce ⟹ rsv_total(qg) + counter + num_bytes <= max for every parent. |
| `simple_mode_no_rescan` | INVARIANT | per-enable(SIMPLE): qgroup_rescan_init never called. |

### Layer 2: TLA+

`fs/btrfs/qgroup.tla`:
- States: per-fs_info {Disabled, Enabling, Enabled(Full|Simple), Rescanning, Disabling}.
- Per-qgroup: {rfer, excl, rsv} counters.
- Per-extent: {bytenr, num_bytes, old_roots, new_roots}.
- Properties:
  - `safety_disabled_no_reservations` — Disabled ⟹ all rsv.values == 0.
  - `safety_simple_no_excl` — Simple ⟹ qg.excl unchanged on account.
  - `safety_full_excl_invariant` — Full: Σ excl ≤ Σ rfer for every parent chain.
  - `safety_inherit_preserves_parents` — Per-snapshot: new qgroup parents ⊇ src qgroup parents (under auto_inherit).
  - `safety_enforce_bounds` — Per-reserve(enforce=true): success ⟹ ∀ parent qg: rsv_total + rfer/excl + num_bytes ≤ max.
  - `liveness_rescan_terminates` — Per-rescan: bounded by extent-tree size.
  - `liveness_account_drains` — Per-commit: dirty_extents xarray emptied after btrfs_qgroup_account_extents.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Qgroup::quota_enable` post: quota_root != NULL ∧ FS_QUOTA_ENABLED set ∧ status_item persisted | `Qgroup::quota_enable` |
| `Qgroup::quota_disable` post: quota_root == NULL ∧ FS_QUOTA_ENABLED cleared ∧ tree freed | `Qgroup::quota_disable` |
| `Qgroup::account_extent` post: Σ(qg.rfer Δ) per old_roots/new_roots truth-table | `Qgroup::account_extent` |
| `Qgroup::reserve` post: success ⟹ qg.rsv.values[type] += num_bytes for every parent in chain | `Qgroup::reserve` |
| `Qgroup::reserve` post: -EDQUOT ⟹ no qg.rsv mutated | `Qgroup::reserve` |
| `Qgroup::inherit` post: level-0 qgroup for objectid exists ∧ has expected parent set | `Qgroup::inherit` |
| `Qgroup::run` post: ∀qg ∈ dirty: INFO_KEY + LIMIT_KEY rewritten ∧ qg removed from dirty list | `Qgroup::run` |
| `Qgroup::record_squota_delta` post: Simple-mode only mutates qg.rfer (not excl) | `Qgroup::record_squota_delta` |

### Layer 4: Verus/Creusot functional

`Per-write reserve → write → commit account_extents` semantic equivalence: at commit boundary, fs_info.dirty_qgroups drained ∧ on-disk INFO_KEY values for every level-0 qgroup reflect (sum of writes within trans) + initial values, modulo backref-walk identifying shared vs exclusive extents per `Documentation/filesystems/btrfs.rst` and on-disk format. Simple-quotas equivalence: every extent allocation records owner-subvol stamp; btrfs_record_squota_delta(is_inc=true|false) sums to net rfer change per subvol; no excl tracked.

## Hardening

(Inherits row-1 features from `fs/btrfs/00-overview.md` § Hardening.)

Qgroup reinforcement:

- **Per-qgroup_ioctl_lock serialization** — defense against per-concurrent-ioctl (enable/disable/create/remove) race.
- **Per-qgroup_lock spinlock** — defense against per-counter torn-write (rfer/excl/rsv updates atomic).
- **Per-rescan vs commit coordination** — defense against per-double-account (rescan-progress check in account_extent).
- **Per-EXTENT_TREE_V2 explicit reject in quota_enable** — defense against per-unsupported-combination corruption.
- **Per-INCONSISTENT sticky flag on update error** — defense against per-silent-quota-drift.
- **Per-EDQUOT before rsv_add** — defense against per-over-allocation; reservation atomic w/ check.
- **Per-FS_QUOTA_OVERRIDE + CAP_SYS_RESOURCE for emergency** — defense against per-root-lockout when filesystem fills.
- **Per-qgroup_to_skip drop in account_extents** — defense against per-snapshot self-counting double-charge.
- **Per-simple-mode INCOMPAT_SIMPLE_QUOTA bit** — defense against per-old-kernel mount corrupting simple-quota stamps.
- **Per-flush_reservations before quota_disable** — defense against per-dangling-rsv leak across disable.
- **Per-qgroup_check_reserved_leak on inode evict** — defense against per-rsv leak / underflow on cleanup.
- **Per-cycle-detection in add_relation** — defense against per-quota-tree cyclic parent chain.
- **Per-maybe_fs_roots gate in account_extent** — defense against per-non-fs-tree (reloc, csum) accounting pollution.

## Grsecurity/PaX-style Reinforcement

Beyond the upstream hardening above, Rookery layers the following grsec/PaX-style controls onto `fs/btrfs/qgroup.c`:

- **PAX_USERCOPY** — bounds-checks every `BTRFS_IOC_QUOTA_*` / `BTRFS_IOC_QGROUP_ASSIGN` / `BTRFS_IOC_QGROUP_CREATE` payload so a malformed ioctl cannot drive a slab overrun in the qgroup parser.
- **PAX_KERNEXEC** — keeps the qgroup ops tables and rescan worker callbacks in read-only memory so a memory bug cannot rewrite the account/rescan dispatch.
- **PAX_RANDKSTACK** — randomizes kernel-stack offset on each qgroup ioctl/transaction entry so an attacker cannot deterministically probe the long account-extent path.
- **PAX_REFCOUNT** — wraps `btrfs_qgroup.refcnt` (held across relation/account paths) so a torn relation/account cycle cannot wrap.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `btrfs_qgroup` and rescan-progress scratch so leftover quota stamps and rfer/excl counters cannot be recovered from the freelist.
- **PAX_UDEREF** — enforces user/kernel pointer separation across the qgroup ioctl boundary.
- **PAX_RAP / kCFI** — forward-edge CFI on the rescan worker callback so a corrupted worker pointer cannot redirect rescan into a userspace gadget.
- **GRKERNSEC_HIDESYM** — strips kernel pointers from qgroup INCONSISTENT/EDQUOT printks so a crafted image cannot leak kASLR offsets via dmesg.
- **GRKERNSEC_DMESG** — gates dmesg on `CAP_SYSLOG` so qgroup-corruption diagnostics do not leak pointer material to unprivileged users.
- **CAP_SYS_ADMIN on qgroup quota ioctls** — `BTRFS_IOC_QUOTA_CTL`, `BTRFS_IOC_QGROUP_ASSIGN`, `BTRFS_IOC_QGROUP_CREATE`, and `BTRFS_IOC_QGROUP_LIMIT` are hard-gated so unprivileged callers cannot enable/disable quotas or assign cross-subvol relations.
- **qgroup-tree PAX_REFCOUNT** — `btrfs_qgroup.refcnt` is wrapped with the hardened-refcount layer so a malicious workload driving extreme relation/account churn cannot wrap the counter and produce an early-free UAF on a qgroup node.
- **GRKERNSEC_HIDESYM on cycle-detection failures** — `add_relation` cycle-detection failure traces sanitize embedded pointers so a crafted quota-tree cycle attack cannot extract qgroup-node addresses via dmesg.
- **FS_QUOTA_OVERRIDE + CAP_SYS_RESOURCE audit** — any use of the emergency override path is logged via `audit_log` so an unexpected override (which bypasses EDQUOT) is observable post-hoc.

Rationale: qgroup accounting is a privilege-relevant resource policy; INCONSISTENT drift translates directly into a quota bypass, and the rescan worker plus cross-subvol relation graph are easy targets for refcount/usercopy bugs, so capability gating + refcount hardening are layered on top of the upstream qgroup_ioctl_lock/qgroup_lock defenses.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/btrfs/qgroup-tree-walk subtree-swap during relocation (`btrfs_qgroup_trace_subtree_after_cow` mechanics covered separately if expanded).
- fs/btrfs/backref.c `btrfs_find_all_roots` backref walker (covered separately if expanded).
- fs/btrfs/extent-tree.c delayed-ref running (covered in `extent-tree.md` Tier-3).
- fs/btrfs/transaction.c commit machinery (covered in `transaction.md` Tier-3).
- fs/btrfs/ioctl.c quota ioctl plumbing (covered separately if expanded).
- fs/btrfs/sysfs.c qgroup sysfs surface (covered separately if expanded).
- Implementation code.
