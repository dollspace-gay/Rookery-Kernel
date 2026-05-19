---
title: "Tier-3: fs/notify/fsnotify.c + fs/notify/notification.c — fsnotify core dispatch + event queue"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fs/notify/fsnotify.c` is the dispatch core for the **fsnotify** subsystem — the shared scaffolding under `inotify(7)`, `fanotify(7)`, `dnotify(7)`, audit, and NFS lease/delegation watchers. The VFS plants `fsnotify*()` hooks (`fsnotify_open`, `fsnotify_close`, `fsnotify_modify`, `fsnotify_create`, ...) that call into `fsnotify(mask, data, data_type, dir, file_name, inode, cookie)`. The core walks the relevant **mark connectors** (per-inode, per-mount, per-superblock, per-mnt-namespace) under `fsnotify_mark_srcu`, merges per-group masks/ignored-masks, and dispatches each watching `struct fsnotify_group` once via `send_to_group()`. Per-group event delivery flows into `fs/notify/notification.c`, which manages the per-group **notification queue** (`group->notification_list`, bounded by `group->max_events`, with `group->overflow_event` on saturation); userspace pulls events with the group-specific `read()` (e.g. `inotify_read`, `fanotify_read`) and wakeups arrive via `group->notification_waitq` + `SIGIO` (`kill_fasync`).

This Tier-3 covers `fs/notify/fsnotify.c` (~728 lines) and `fs/notify/notification.c` (~193 lines).

### Acceptance Criteria

- [ ] AC-1: fsnotify(): zero matching marks (mask & marks_mask == 0): early return 0, no SRCU acquired.
- [ ] AC-2: fsnotify(): single inode mark with mask & event: dispatched exactly once to that group.
- [ ] AC-3: fsnotify(): inode + mount + sb marks for same group: dispatched exactly once (multi-mark merge via iter_info).
- [ ] AC-4: fsnotify(): permission-event handler returns nonzero: early-out (deny propagates).
- [ ] AC-5: __fsnotify_parent(): parent_watched + child has events: parent receives with file_name + FS_EVENT_ON_CHILD.
- [ ] AC-6: send_to_group(): mark->ignore_mask covers event: not delivered.
- [ ] AC-7: send_to_group(): FS_MODIFY clears ignore_mask unless FSNOTIFY_MARK_FLAG_IGNORED_SURV_MODIFY.
- [ ] AC-8: fsnotify_insert_event(): q_len >= max_events: enqueues overflow_event (once) and returns 2.
- [ ] AC-9: fsnotify_insert_event(): merge hook returns nonzero: caller event not enqueued; ret = merge result (1).
- [ ] AC-10: fsnotify_insert_event(): wakes notification_waitq and sends SIGIO via fsn_fa.
- [ ] AC-11: fsnotify_remove_first_event(): empty queue returns NULL.
- [ ] AC-12: fsnotify_flush_notify(): drains queue and frees all events except overflow_event.
- [ ] AC-13: group->shutdown: insert_event returns 2 without enqueue.
- [ ] AC-14: fsnotify_get_cookie(): strictly increasing monotonic across calls.

### Architecture

```
struct FsnotifyGroup {
  ops: &'static FsnotifyOps,
  refcnt: Refcount,
  notification_list: ListHead<FsnotifyEvent>,
  notification_lock: SpinLock<()>,
  notification_waitq: WaitQueueHead,
  q_len: u32,
  max_events: u32,
  overflow_event: *FsnotifyEvent,
  mark_mutex: Mutex<()>,
  marks_list: ListHead<FsnotifyMark>,
  fsn_fa: *FasyncStruct,
  memcg: *MemCgroup,
  shutdown: bool,
  priority: u8,                            // FSNOTIFY_PRIO_NORMAL / _CONTENT / _PRE_CONTENT
  private: GroupPrivate,                   // inotify_data | fanotify_data | dnotify_data
}

struct FsnotifyMark {
  group: NonNull<FsnotifyGroup>,
  connector: NonNull<FsnotifyMarkConnector>,
  obj_list: HlistNode,                     // on connector->list
  g_list: ListNode,                        // on group->marks_list
  mask: u32,                               // FS_*
  ignore_mask: u32,
  flags: u32,                              // FSNOTIFY_MARK_FLAG_*
  refcnt: Refcount,
  lock: SpinLock<()>,
}

struct FsnotifyMarkConnector {
  obj: *mut (),                            // inode / sb / mount / mntns
  type_: u8,                               // FSNOTIFY_OBJ_TYPE_*
  list: HlistHead<FsnotifyMark>,           // sorted by group priority
  destroy_next: HlistNode,
  prio: u32,
}

struct FsnotifyEvent {
  list: ListNode,                          // on group->notification_list
  // subclass tail
}

struct FsnotifyIterInfo {
  marks: [*FsnotifyMark; FSNOTIFY_ITER_TYPE_COUNT],
  current_group: *FsnotifyGroup,
  report_mask: u32,                        // bitmask of selected iter_types
  srcu_idx: i32,
}
```

`Fsnotify::dispatch(mask, data, data_type, dir, file_name, inode, cookie) -> Result<()>`:
1. path = fsnotify_data_path(data, data_type); sb = fsnotify_data_sb; mnt_data = fsnotify_data_mnt; sbinfo = sb.and_then(fsnotify_sb_info).
2. mnt = path.map(|p| real_mount(p.mnt)); inode2 = None; inode2_type = 0.
3. /* Dirent event vs child event */
4. if inode.is_none(): inode = dir; if mask & FS_RENAME: moved = fsnotify_data_dentry(data, data_type); inode2 = moved.d_parent.d_inode; inode2_type = ITER_TYPE_INODE2.
5. else if mask & FS_EVENT_ON_CHILD: inode2 = dir; inode2_type = ITER_TYPE_PARENT.
6. /* Fast bail */
7. if no connector on any of {sbinfo, mnt, inode, inode2, mntns}: return Ok(()).
8. marks_mask = union of all object masks.
9. test_mask = mask & ALL_FSNOTIFY_EVENTS; if (test_mask & marks_mask) == 0: return Ok(()).
10. iter_info.srcu_idx = srcu_read_lock(&fsnotify_mark_srcu).
11. populate iter_info.marks[ITER_TYPE_SB/_VFSMOUNT/_INODE/_INODE2/_PARENT/_MNTNS] via fsnotify_first_mark.
12. loop:
    - if !fsnotify_iter_select_report_types(&iter_info): break.
    - ret = send_to_group(mask, data, data_type, dir, file_name, cookie, &iter_info).
    - if ret ∧ (mask & ALL_FSNOTIFY_PERM_EVENTS): goto out.
    - fsnotify_iter_next(&iter_info).
13. ret = 0.
14. out: srcu_read_unlock(&fsnotify_mark_srcu, iter_info.srcu_idx); return ret.

`Fsnotify::parent(dentry, mask, data, data_type) -> Result<()>`:
1. path = fsnotify_data_path(data, data_type).
2. mnt_mask = path ? READ_ONCE(real_mount(path.mnt).mnt_fsnotify_mask) : 0.
3. inode = d_inode(dentry); parent_watched = dentry.d_flags & DCACHE_FSNOTIFY_PARENT_WATCHED.
4. if !parent_watched ∧ !fsnotify_object_watched(inode, mnt_mask, mask): return Ok(()).
5. parent_needed = fsnotify_event_needs_parent(inode, mnt_mask, mask).
6. parent = dget_parent(dentry); p_inode = parent.d_inode.
7. parent_interested = fsnotify_inode_watches_children(p_inode) & mask.
8. if parent_watched ∧ !parent_interested: fsnotify_clear_child_dentry_flag(p_inode, dentry).
9. if parent_interested ∨ parent_needed: take_dentry_name_snapshot(&name, dentry); file_name = &name.name; mask |= FS_EVENT_ON_CHILD.
10. ret = fsnotify(mask, data, data_type, p_inode, file_name, inode, 0).
11. release_dentry_name_snapshot(&name) if taken; dput(parent).

`Fsnotify::send_to_group(mask, data, data_type, dir, file_name, cookie, iter_info) -> Result<()>`:
1. test_mask = mask & ALL_FSNOTIFY_EVENTS.
2. if iter_info.report_mask == 0: return Ok(()).
3. /* Clear ignore on FS_MODIFY for non-surviving marks */
4. if mask & FS_MODIFY:
   - for (mark, type) in iter_info.foreach_iter_mark_type():
     - if !(mark.flags & FSNOTIFY_MARK_FLAG_IGNORED_SURV_MODIFY): mark.ignore_mask = 0.
5. marks_mask = 0; marks_ignore_mask = 0.
6. for (mark, type) in iter_info.foreach_iter_mark_type():
   - group = mark.group; marks_mask |= mark.mask; marks_ignore_mask |= fsnotify_effective_ignore_mask(mark, is_dir, type).
7. if (test_mask & marks_mask & !marks_ignore_mask) == 0: return Ok(()).
8. if group.ops.handle_event.is_some():
   - return group.ops.handle_event(group, mask, data, data_type, dir, file_name, cookie, iter_info).
9. else: return fsnotify_handle_event(group, mask, data, data_type, dir, file_name, cookie, iter_info).

`Fsnotify::insert_event(group, event, merge, insert) -> i32`:
1. spin_lock(&group.notification_lock).
2. if group.shutdown: spin_unlock; return 2.
3. if event == group.overflow_event ∨ group.q_len >= group.max_events:
   - if !list_empty(&group.overflow_event.list): spin_unlock; return 2.
   - event = group.overflow_event; ret = 2; goto queue.
4. if !list_empty(&group.notification_list) ∧ merge:
   - ret = merge(group, event); if ret: spin_unlock; return ret.
5. queue:
   - group.q_len++; list_add_tail(&event.list, &group.notification_list); if insert: insert(group, event).
6. spin_unlock(&group.notification_lock).
7. wake_up(&group.notification_waitq); kill_fasync(&group.fsn_fa, SIGIO, POLL_IN).
8. return ret.

`Fsnotify::flush_notify(group)`:
1. spin_lock(&group.notification_lock).
2. while !fsnotify_notify_queue_is_empty(group):
   - event = fsnotify_remove_first_event(group).
   - spin_unlock; fsnotify_destroy_event(group, event); spin_lock.
3. spin_unlock.

`Fsnotify::iter_select_report_types(iter_info) -> bool`:
1. max_prio_group = None.
2. for type in 0..FSNOTIFY_ITER_TYPE_COUNT:
   - mark = iter_info.marks[type].
   - if mark.is_some() ∧ fsnotify_compare_groups(max_prio_group, mark.group) > 0: max_prio_group = Some(mark.group).
3. if max_prio_group.is_none(): return false.
4. iter_info.current_group = max_prio_group; iter_info.report_mask = 0.
5. for type in 0..FSNOTIFY_ITER_TYPE_COUNT:
   - mark = iter_info.marks[type].
   - if mark.is_some() ∧ mark.group == iter_info.current_group:
     - if type == FSNOTIFY_ITER_TYPE_PARENT ∧ !(mark.mask & FS_EVENT_ON_CHILD) ∧ !(fsnotify_ignore_mask(mark) & FS_EVENT_ON_CHILD): continue.
     - fsnotify_iter_set_report_type(iter_info, type).
6. return true.

### Out of Scope

- fs/notify/mark.c mark add/remove/connector lifecycle (covered separately as Tier-3 if expanded)
- fs/notify/inotify/* user-facing inotify (covered in `notify-inotify.md`)
- fs/notify/fanotify/* fanotify (covered in `notify-fanotify.md` Tier-3 if expanded)
- fs/notify/dnotify/dnotify.c dnotify (legacy; covered separately if expanded)
- audit_fsnotify integration (covered in audit Tier-3 if expanded)
- VFS hook plumbing (`linux/fsnotify.h` inline wrappers; covered in `vfs/` Tier-3 set)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct fsnotify_group` | per-watcher group | `FsnotifyGroup` |
| `struct fsnotify_mark` | per-(group, object) watch record | `FsnotifyMark` |
| `struct fsnotify_mark_connector` | per-object mark list head | `FsnotifyMarkConnector` |
| `struct fsnotify_event` | per-event base | `FsnotifyEvent` |
| `struct fsnotify_iter_info` | per-dispatch iterator state | `FsnotifyIterInfo` |
| `struct fsnotify_sb_info` | per-sb (super_block) fsnotify state | `FsnotifySbInfo` |
| `struct fsnotify_mnt` | per-mnt event data | `FsnotifyMntData` |
| `struct fsnotify_ops` | per-group callbacks | `FsnotifyOps` |
| `fsnotify()` | per-event dispatch | `Fsnotify::dispatch` |
| `__fsnotify_parent()` | per-child event with parent name | `Fsnotify::parent` |
| `__fsnotify_inode_delete()` / `__fsnotify_vfsmount_delete()` / `__fsnotify_mntns_delete()` | per-object teardown | `Fsnotify::inode_delete` / `vfsmount_delete` / `mntns_delete` |
| `fsnotify_sb_delete()` / `fsnotify_sb_free()` | per-superblock teardown | `Fsnotify::sb_delete` / `sb_free` |
| `fsnotify_set_children_dentry_flags()` | per-parent watches-children flag | `Fsnotify::set_children_dentry_flags` |
| `fsnotify_handle_event()` / `fsnotify_handle_inode_event()` | per-group event-handler bridge | `Fsnotify::handle_event` / `handle_inode_event` |
| `send_to_group()` | per-group merge+deliver | `Fsnotify::send_to_group` |
| `fsnotify_first_mark()` / `fsnotify_next_mark()` | per-connector SRCU walk | `Fsnotify::first_mark` / `next_mark` |
| `fsnotify_iter_select_report_types()` / `fsnotify_iter_next()` | per-multi-queue priority merge | `Fsnotify::iter_select_report_types` / `iter_next` |
| `fsnotify_attach_connector_to_object()` | per-object connector publish | `Fsnotify::attach_connector_to_object` |
| `fsnotify_init()` | per-boot init | `Fsnotify::init` |
| `fsnotify_get_cookie()` | per-rename sync cookie | `Fsnotify::get_cookie` |
| `fsnotify_destroy_event()` | per-event free | `Fsnotify::destroy_event` |
| `fsnotify_insert_event()` | per-event enqueue + merge | `Fsnotify::insert_event` |
| `fsnotify_peek_first_event()` / `fsnotify_remove_first_event()` | per-queue head | `Fsnotify::peek_first_event` / `remove_first_event` |
| `fsnotify_remove_queued_event()` | per-event unlink | `Fsnotify::remove_queued_event` |
| `fsnotify_flush_notify()` | per-group teardown drain | `Fsnotify::flush_notify` |
| `fsnotify_open_perm_and_set_mode()` | per-open permission gate | `Fsnotify::open_perm_and_set_mode` |
| `fsnotify_mnt()` | per-mnt event publish | `Fsnotify::mnt_event` |

### compatibility contract

REQ-1: struct fsnotify_group:
- ops: per-flavor &fsnotify_ops (handle_event | handle_inode_event, free_event, free_group_priv, freeing_mark).
- refcnt: get/put via fsnotify_get_group / put_group.
- notification_list: list head of queued fsnotify_event.
- notification_lock: spinlock guarding notification_list, q_len, overflow_event.list.
- notification_waitq: wait_queue for blocking read().
- q_len: current depth.
- max_events: bound.
- overflow_event: per-group sticky event used when q_len >= max_events.
- mark_mutex: serializes mark add/remove/modify within group.
- marks_list: list of all marks owned by this group.
- inotify_data / fanotify_data / dnotify_data: per-flavor private (idr, fasync, ucounts, ...).
- memcg: per-group memcg accounting target.
- shutdown: bool; true ⟹ insert_event returns 2 (drop).
- priority: FSNOTIFY_PRIO_NORMAL | _CONTENT | _PRE_CONTENT.

REQ-2: struct fsnotify_mark:
- group: owning fsnotify_group (back-ref; refcounted).
- connector: pointer to fsnotify_mark_connector (which is per-object: inode/sb/mount/mntns).
- obj_list: hlist node on connector->list.
- g_list: list node on group->marks_list.
- mask: events of interest (FS_*).
- ignore_mask / ignored_mask: events to suppress (with optional FSNOTIFY_MARK_FLAG_IGNORED_SURV_MODIFY survival).
- flags: FSNOTIFY_MARK_FLAG_ALIVE | _ATTACHED | _EXCL_UNLINK | _IGNORED_SURV_MODIFY | _NO_IREF | _HAS_IGNORE_FLAGS | _IN_ONESHOT | ...
- refcnt: refcount; group reference + idr reference (inotify) + caller ref.
- lock: spinlock for mask/flags mutation.

REQ-3: struct fsnotify_mark_connector (per-object):
- obj: pointer back to inode | super_block | mount | mnt_namespace.
- type: FSNOTIFY_OBJ_TYPE_INODE | _SB | _VFSMOUNT | _MNTNS (encoded with flags).
- list: hlist_head of fsnotify_mark (sorted by group->priority then group address).
- destroy_next: hlist node on global destroy list.
- prio: connector priority sum.

REQ-4: struct fsnotify_event:
- list: list_head on group->notification_list.
- (subclass tail: inotify_event_info, fanotify_event, ... carry mask / wd / cookie / name).

REQ-5: fsnotify(mask, data, data_type, dir, file_name, inode, cookie):
- Resolve path = fsnotify_data_path(data, data_type); sb = fsnotify_data_sb; mnt_data = fsnotify_data_mnt; sbinfo = sb ? fsnotify_sb_info(sb) : NULL.
- If !inode: inode = dir (dirent event); for FS_RENAME: inode2 = new_dir (FSNOTIFY_ITER_TYPE_INODE2).
- Else if FS_EVENT_ON_CHILD: inode2 = dir (FSNOTIFY_ITER_TYPE_PARENT).
- Fast bail: if no sb/mnt/inode/inode2/mntns connector: return 0.
- marks_mask = union(sb->s_fsnotify_mask, mnt->mnt_fsnotify_mask, inode->i_fsnotify_mask, inode2->i_fsnotify_mask, mnt_data->ns->n_fsnotify_mask).
- test_mask = mask & ALL_FSNOTIFY_EVENTS; if !(test_mask & marks_mask): return 0.
- srcu_idx = srcu_read_lock(&fsnotify_mark_srcu).
- iter_info.marks[ITER_TYPE_*] = fsnotify_first_mark(connector) for each present object.
- while fsnotify_iter_select_report_types(&iter_info):
  - send_to_group(mask, data, data_type, dir, file_name, cookie, &iter_info).
  - if ret ∧ (mask & ALL_FSNOTIFY_PERM_EVENTS): goto out (deny propagates).
  - fsnotify_iter_next(&iter_info).
- srcu_read_unlock; return ret.

REQ-6: __fsnotify_parent(dentry, mask, data, data_type):
- path = fsnotify_data_path; mnt_mask = path ? real_mount(path->mnt)->mnt_fsnotify_mask : 0.
- inode = d_inode(dentry); parent_watched = dentry->d_flags & DCACHE_FSNOTIFY_PARENT_WATCHED.
- Fast bail: if !parent_watched ∧ !fsnotify_object_watched(inode, mnt_mask, mask): return 0.
- parent_needed = fsnotify_event_needs_parent(inode, mnt_mask, mask).
- If parent_watched ∨ parent_needed: dget_parent; name_snapshot; mask |= FS_EVENT_ON_CHILD; file_name = &name.name.
- ret = fsnotify(mask, data, data_type, p_inode, file_name, inode, 0).
- release name_snapshot; dput(parent).

REQ-7: send_to_group(mask, data, data_type, dir, file_name, cookie, iter_info):
- test_mask = mask & ALL_FSNOTIFY_EVENTS.
- /* Clear ignore on FS_MODIFY */
- if mask & FS_MODIFY: for each selected mark: if !(flags & FSNOTIFY_MARK_FLAG_IGNORED_SURV_MODIFY): mark->ignore_mask = 0.
- marks_mask = OR of mark->mask; marks_ignore_mask = OR of fsnotify_effective_ignore_mask(mark, is_dir, type).
- if !(test_mask & marks_mask & ~marks_ignore_mask): return 0.
- if group->ops->handle_event: return group->ops->handle_event(group, mask, data, data_type, dir, file_name, cookie, iter_info).
- else: return fsnotify_handle_event(group, mask, ..., iter_info) → fsnotify_handle_inode_event for parent_mark + inode_mark.

REQ-8: fsnotify_iter_select_report_types(iter_info):
- max_prio_group = NULL.
- for each iter_type: m = iter_info.marks[type]; if m ∧ fsnotify_compare_groups(max_prio_group, m->group) > 0: max_prio_group = m->group.
- if !max_prio_group: return false.
- iter_info.current_group = max_prio_group; iter_info.report_mask = 0.
- for each iter_type: if mark->group == current_group: fsnotify_iter_set_report_type(iter_info, type).
- /* Skip TYPE_PARENT marks that don't watch children */
- if type == TYPE_PARENT ∧ !(mark->mask & FS_EVENT_ON_CHILD) ∧ !(ignore & FS_EVENT_ON_CHILD): continue.
- return true.

REQ-9: fsnotify_iter_next(iter_info):
- for each iter_type: if mark->group == current_group: iter_info.marks[type] = fsnotify_next_mark(iter_info.marks[type]).

REQ-10: fsnotify_attach_connector_to_object(connp, obj, obj_type) (defined in mark.c; semantically required here):
- Allocates a fsnotify_mark_connector under fsnotify_connector_caches.
- conn->obj = obj; conn->type = obj_type; INIT_HLIST_HEAD(&conn->list).
- cmpxchg-publishes &conn into *connp; loser frees its conn.
- Object side gets a watched-objects counter increment.

REQ-11: fsnotify_destroy_event(group, event):
- if !event ∨ event == group->overflow_event: return.
- if !list_empty(&event->list): spin_lock(&group->notification_lock); WARN_ON(!list_empty); spin_unlock.
- group->ops->free_event(group, event).

REQ-12: fsnotify_insert_event(group, event, merge, insert):
- spin_lock(&group->notification_lock).
- if group->shutdown: spin_unlock; return 2 (drop).
- if event == group->overflow_event ∨ group->q_len >= group->max_events:
  - ret = 2.
  - if !list_empty(&overflow_event->list): spin_unlock; return 2 (already queued).
  - event = group->overflow_event; goto queue.
- if list non-empty ∧ merge: ret = merge(group, event); if ret: spin_unlock; return ret.
- queue: group->q_len++; list_add_tail(&event->list, &group->notification_list); if insert: insert(group, event).
- spin_unlock(&group->notification_lock).
- wake_up(&group->notification_waitq); kill_fasync(&group->fsn_fa, SIGIO, POLL_IN).
- return ret (0 added | 1 merged | 2 drop/overflow).

REQ-13: fsnotify_remove_queued_event(group, event):
- assert_spin_locked(&group->notification_lock).
- list_del_init(&event->list); group->q_len--.

REQ-14: fsnotify_peek_first_event(group):
- assert_spin_locked(&group->notification_lock).
- if fsnotify_notify_queue_is_empty: return NULL.
- return list_first_entry(&group->notification_list, struct fsnotify_event, list).

REQ-15: fsnotify_remove_first_event(group):
- event = fsnotify_peek_first_event(group); if !event: return NULL.
- fsnotify_remove_queued_event(group, event); return event.

REQ-16: fsnotify_flush_notify(group) (group teardown):
- spin_lock(&notification_lock).
- while !fsnotify_notify_queue_is_empty:
  - event = fsnotify_remove_first_event(group).
  - spin_unlock; fsnotify_destroy_event(group, event); spin_lock.
- spin_unlock.

REQ-17: Mark-type matrix:
- FSNOTIFY_OBJ_TYPE_INODE: per-inode (i_fsnotify_marks); fired by fsnotify() with inode/inode2.
- FSNOTIFY_OBJ_TYPE_VFSMOUNT: per-vfsmount (mnt->mnt_fsnotify_marks); fires on any inode under mount.
- FSNOTIFY_OBJ_TYPE_SB: per-superblock (sbinfo->sb_marks); fires on any inode in fs.
- FSNOTIFY_OBJ_TYPE_MNTNS: per-mnt-namespace (ns->n_fsnotify_marks); fires for mount events (fsnotify_mnt).
- Connector lookups via fsnotify_first_mark / fsnotify_next_mark are SRCU-protected.

REQ-18: Per-event ignored / ignore_mask:
- ignore_mask: events to drop for this mark (always).
- FSNOTIFY_MARK_FLAG_IGNORED_SURV_MODIFY: keep ignore across FS_MODIFY.
- Default: FS_MODIFY clears ignore_mask (per-fanotify FAN_MARK_IGNORED_MASK behavior).

REQ-19: fsnotify_get_cookie():
- Returns atomic_inc_return(&fsnotify_sync_cookie); used to pair FS_MOVED_FROM with FS_MOVED_TO across a rename.

REQ-20: fsnotify_init() (core_initcall):
- BUILD_BUG_ON(HWEIGHT32(ALL_FSNOTIFY_BITS) != 26).
- init_srcu_struct(&fsnotify_mark_srcu).
- fsnotify_init_connector_caches().

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dispatch_srcu_balanced` | INVARIANT | per-fsnotify: srcu_read_lock balanced with srcu_read_unlock on all paths. |
| `dispatch_early_bail_no_srcu` | INVARIANT | per-fsnotify: no-connector early bail does not take SRCU. |
| `iter_select_terminates` | INVARIANT | per-iter loop: each step advances at least one iter_info.marks[type] for current_group. |
| `insert_event_q_len_bounded` | INVARIANT | per-insert: q_len ≤ max_events ∨ event == overflow_event. |
| `insert_event_overflow_once` | INVARIANT | per-overflow: overflow_event enqueued at most once concurrently. |
| `flush_notify_drains_all` | INVARIANT | per-flush: post-call queue is empty; all non-overflow events freed. |
| `parent_watched_implies_dispatch` | INVARIANT | per-parent: parent_watched ∧ event interesting ⟹ fsnotify() called. |
| `ignore_mask_modify_clear` | INVARIANT | per-FS_MODIFY: marks without SURV_MODIFY have ignore_mask cleared. |

### Layer 2: TLA+

`fs/notify-core.tla`:
- Per-event-arrival → per-connector-walk → per-group-merge → per-deliver / per-overflow.
- Properties:
  - `safety_each_group_delivered_at_most_once_per_event` — per-event: distinct groups receive disjoint copies; same group once.
  - `safety_overflow_event_singleton` — per-group: overflow_event present in queue at most once.
  - `safety_max_events_bound` — per-group: q_len ≤ max_events at all times (excluding the overflow slot).
  - `safety_perm_event_deny_short_circuits` — per-permission-event: ret != 0 breaks dispatch loop.
  - `liveness_queued_event_eventually_read_or_flushed` — per-event: reaches consumer OR is freed in flush_notify.
  - `liveness_shutdown_drops_new_events` — per-shutdown group: subsequent inserts return 2.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Fsnotify::dispatch` post: ret 0 ∨ ret = group->ops->handle_event return | `Fsnotify::dispatch` |
| `Fsnotify::send_to_group` post: dispatched ⟹ test_mask & marks_mask & ~marks_ignore_mask != 0 | `Fsnotify::send_to_group` |
| `Fsnotify::insert_event` post: ret 0 ⟹ list_contains(group.notification_list, event) ∧ q_len incremented | `Fsnotify::insert_event` |
| `Fsnotify::insert_event` post: ret 2 ⟹ overflow path OR shutdown | `Fsnotify::insert_event` |
| `Fsnotify::remove_first_event` post: list_empty ⟹ None; else q_len decremented | `Fsnotify::remove_first_event` |
| `Fsnotify::iter_select_report_types` post: false ⟹ all iter_info.marks[*] consumed | `Fsnotify::iter_select_report_types` |
| `Fsnotify::flush_notify` post: notification_list empty ∧ q_len = 0 | `Fsnotify::flush_notify` |
| `Fsnotify::get_cookie` post: returned value strictly > previous | `Fsnotify::get_cookie` |

### Layer 4: Verus/Creusot functional

`Per-VFS-hook → fsnotify() → connector-walk (sb, mount, inode, inode2, mntns) → iter merge by priority → send_to_group → group->ops->handle_event → fsnotify_insert_event → read()-side fsnotify_remove_first_event → copy_event_to_user` semantic equivalence: per-Documentation/filesystems/fsnotify.rst, per-inotify(7), per-fanotify(7).

### hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

Fsnotify-core reinforcement:

- **Per-fsnotify_mark_srcu read-side bounded** — defense against per-RCU-grace-period starvation on busy fs.
- **Per-fast-path no-mark zero-cost** — defense against per-hook overhead on unwatched VFS.
- **Per-q_len ≤ max_events strict bound** — defense against per-group memory exhaustion.
- **Per-overflow_event singleton** — defense against per-storm queue bloat.
- **Per-shutdown drop** — defense against per-teardown UAF during ongoing insert.
- **Per-group.priority ordering** — defense against per-permission-event hijack by lower-priority group.
- **Per-ALL_FSNOTIFY_PERM_EVENTS deny short-circuit** — defense against per-policy bypass after deny.
- **Per-FS_MODIFY ignore_mask reset** — defense against per-stale-ignore granting attacker future ops.
- **Per-FSNOTIFY_MARK_FLAG_EXCL_UNLINK skip on unlinked dentry** — defense against per-deleted-target spurious event.
- **Per-mark.refcnt strict get/put** — defense against per-mark UAF during dispatch.
- **Per-connector cmpxchg publish** — defense against per-attach race double-publish.
- **Per-kill_fasync SIGIO only on non-empty enqueue** — defense against per-signal flood on overflow.
- **Per-sync_cookie monotonic** — defense against per-rename pair confusion.

