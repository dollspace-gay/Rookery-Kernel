# Tier-3: kernel/audit_watch.c — Audit filesystem watches

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/audit/00-overview.md
upstream-paths:
  - kernel/audit_watch.c (~547 lines)
  - kernel/audit.h
  - include/linux/audit.h
  - include/linux/fsnotify_backend.h
-->

## Summary

The audit watch subsystem translates `auditctl -w <path> -p <perms> -k <key>` watches into kernel fsnotify marks. Per-watch consists of an `audit_parent` (one fsnotify mark on the *parent directory* inode listening for AUDIT_FS_WATCH = `FS_MOVE | FS_CREATE | FS_DELETE | FS_DELETE_SELF | FS_MOVE_SELF | FS_UNMOUNT`) and an `audit_watch` (per-name child watch attached to that parent). Per-`audit_to_watch` parses an `AUDIT_WATCH` filter field into a `struct audit_watch` attached to a krule. Per-`audit_add_watch` resolves the path via `audit_get_nd` -> `kern_path_parent`, finds-or-creates an `audit_parent` via `audit_find_parent` / `audit_init_parent` + `fsnotify_init_mark` + `fsnotify_add_inode_mark`, and inserts the krule on the watch's rules list + the audit_inode_hash bucket determined by `audit_hash_ino(watch->ino)`. Per-`audit_watch_handle_event` is the fsnotify callback that, on each FS event, mutates the watch's stored (dev, ino) by `audit_update_watch` (CREATE / MOVED_TO -> bind new inode; DELETE / MOVED_FROM -> AUDIT_INO_UNSET + invalidate matching cached audit names; DELETE_SELF / UNMOUNT / MOVE_SELF -> tear down the parent watch via `audit_remove_parent_watches`). Critical for: per-PCI-DSS-/-STIG file-watch compliance, per-`auditctl -w`-byte-equivalence, per-path-watch survival across rename and atomic-replace (editor write-via-temp-then-rename pattern).

This Tier-3 covers `kernel/audit_watch.c` (~547 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct audit_watch` | per-watch state (refcount, dev/ino, path, parent, wlist, rules) | `AuditWatch` |
| `struct audit_parent` | per-parent-directory mark (watches list + fsnotify_mark) | `AuditParent` |
| `audit_watch_group` | per-subsystem fsnotify group | `AuditWatchGroup` |
| `AUDIT_FS_WATCH` | per-mask of events monitored | `AUDIT_FS_WATCH_MASK` |
| `audit_get_watch()` / `audit_put_watch()` | per-refcount | `AuditWatch::get` / `put` |
| `audit_get_parent()` / `audit_put_parent()` | per-mark refcount via `fsnotify_get_mark` / `_put_mark` | `AuditParent::get` / `put` |
| `audit_init_watch()` | per-allocate audit_watch (kzalloc + INIT_LIST_HEAD + refcount_set 1) | `AuditWatch::init` |
| `audit_init_parent()` | per-allocate + fsnotify_init_mark + fsnotify_add_inode_mark | `AuditParent::init` |
| `audit_alloc_watch()` (alias of audit_init_watch in `audit.h`) | per-allocate fresh watch | `AuditWatch::alloc` |
| `audit_find_parent()` | per-lookup via fsnotify_find_inode_mark | `AuditParent::find` |
| `audit_free_parent()` | per-kfree post empty watches | `AuditParent::free` |
| `audit_watch_free_mark()` | per-fsnotify_ops .free_mark callback | `AuditParent::free_mark` |
| `audit_to_watch()` | per-parse AUDIT_WATCH filter field | `AuditWatch::to_watch` |
| `audit_dupe_watch()` | per-deep-copy watch on rule update | `AuditWatch::dupe` |
| `audit_add_watch()` | per-attach krule to parent + audit_inode_hash bucket | `AuditWatch::add` |
| `audit_add_to_parent()` | per-link watch into parent->watches | `AuditWatch::add_to_parent` |
| `audit_remove_watch()` | per-remove from parent + drop ref | `AuditWatch::remove` |
| `audit_remove_watch_rule()` | per-remove krule + maybe destroy parent | `AuditWatch::remove_rule` |
| `audit_remove_parent_watches()` | per-tear-down on parent unmount/delete | `AuditParent::remove_watches` |
| `audit_update_watch()` | per-rebind (dev,ino) on FS event | `AuditWatch::update` |
| `audit_watch_handle_event()` | per-fsnotify .handle_inode_event | `AuditWatch::handle_event` |
| `audit_watch_fsnotify_ops` | per-fsnotify_ops vtable | `AUDIT_WATCH_FSNOTIFY_OPS` |
| `audit_watch_init()` (device_initcall) | per-allocate audit_watch_group | `AuditWatch::module_init` |
| `audit_get_nd()` | per-path-resolve via kern_path_parent | `AuditWatch::get_nd` |
| `audit_watch_path()` | per-getter for path | `AuditWatch::path` |
| `audit_watch_compare()` | per-(ino,dev) compare for filter match | `AuditWatch::compare` |
| `audit_watch_log_rule_change()` | per-AUDIT_CONFIG_CHANGE record | `AuditWatch::log_rule_change` |
| `audit_dupe_exe()` | per-deep-copy AUDIT_EXE mark on rule update | `AuditWatch::dupe_exe` |
| `audit_exe_compare()` | per-match current's exe inode against AUDIT_EXE mark | `AuditWatch::exe_compare` |
| `AUDIT_DEV_UNSET` / `AUDIT_INO_UNSET` | per-sentinel for negative-cache | shared constants |

## Compatibility contract

REQ-1: struct audit_watch fields preserved (binary irrelevant — internal):
- refcount_t count: per-reference count; init = 1 in `audit_init_watch`.
- dev_t dev: per-associated-superblock device; AUDIT_DEV_UNSET pre-bind.
- char *path: per-insertion path; kstrdup of the user-supplied watch path (NUL-terminated).
- u64 ino: per-associated-inode number; AUDIT_INO_UNSET pre-bind or post-DELETE/MOVED_FROM.
- struct audit_parent *parent: per-back-pointer to parent directory's mark.
- struct list_head wlist: per-entry in parent->watches list.
- struct list_head rules: per-anchor for `audit_krule->rlist` (every krule sharing this watch).

REQ-2: struct audit_parent fields preserved:
- struct list_head watches: per-anchor for audit_watch->wlist.
- struct fsnotify_mark mark: per-mark on the parent directory inode; lifetime owned by fsnotify (refcounted via `fsnotify_get_mark` / `_put_mark`).

REQ-3: AUDIT_FS_WATCH mask preserved:
- AUDIT_FS_WATCH = FS_MOVE | FS_CREATE | FS_DELETE | FS_DELETE_SELF | FS_MOVE_SELF | FS_UNMOUNT.
- mark.mask set to AUDIT_FS_WATCH in `audit_init_parent`.

REQ-4: audit_to_watch(krule, path, len, op):
- /* Reject if subsystem not initialised */
- if !audit_watch_group: return -EOPNOTSUPP.
- /* Path must be absolute, must not end in `/` */
- if path[0] != '/' || path[len-1] == '/': return -EINVAL.
- /* Watch is only valid on exit-filter lists */
- if krule->listnr != AUDIT_FILTER_EXIT ∧ krule->listnr != AUDIT_FILTER_URING_EXIT: return -EINVAL.
- /* Watch op must be `=` */
- if op != Audit_equal: return -EINVAL.
- /* Mutually-exclusive fields */
- if krule->inode_f || krule->watch || krule->tree: return -EINVAL.
- watch = audit_init_watch(path); if IS_ERR(watch): return PTR_ERR.
- krule->watch = watch.
- return 0.
- /* Caller (audit_data_to_entry's AUDIT_WATCH arm) owns `path` until success; on success, the watch owns it. */

REQ-5: audit_init_watch(path):
- watch = kzalloc_obj(*watch); ENOMEM-on-fail.
- INIT_LIST_HEAD(&watch->rules).
- refcount_set(&watch->count, 1).
- watch->path = path. /* takes ownership */
- watch->dev = AUDIT_DEV_UNSET; watch->ino = AUDIT_INO_UNSET.
- return watch.

REQ-6: audit_init_parent(path):
- inode = d_backing_inode(path->dentry).
- parent = kzalloc_obj(*parent); ENOMEM-on-fail.
- INIT_LIST_HEAD(&parent->watches).
- fsnotify_init_mark(&parent->mark, audit_watch_group). /* mark.refcnt = 1 */
- parent->mark.mask = AUDIT_FS_WATCH.
- ret = fsnotify_add_inode_mark(&parent->mark, inode, 0).
- if ret < 0: audit_free_parent(parent); return ERR_PTR(ret).
- return parent.

REQ-7: audit_get_nd(watch, parent_path):
- d = kern_path_parent(watch->path, parent_path). /* resolves parent dir; out-param `parent_path` filled with parent's struct path; returns leaf dentry */
- if IS_ERR(d): return PTR_ERR(d).
- if d_is_positive(d):
  - watch->dev = d->d_sb->s_dev.
  - watch->ino = d_backing_inode(d)->i_ino.
- dput(d).
- return 0.
- /* Negative dentry case: watch->{dev,ino} remain AUDIT_*_UNSET; will get bound later by FS_CREATE handler. */

REQ-8: audit_add_watch(krule, list_out) (caller holds audit_filter_mutex):
- watch = krule->watch.
- audit_get_watch(watch). /* extra ref because audit_add_to_parent may swap krule->watch with an existing one */
- mutex_unlock(audit_filter_mutex). /* path lookup cannot run under mutex */
- ret = audit_get_nd(watch, &parent_path).
- mutex_lock(audit_filter_mutex).
- if ret: audit_put_watch(watch); return ret.
- parent = audit_find_parent(d_backing_inode(parent_path.dentry)).
- if !parent: parent = audit_init_parent(&parent_path); if IS_ERR(parent): ret = PTR_ERR; goto error.
- audit_add_to_parent(krule, parent).
- *list_out = &audit_inode_hash[audit_hash_ino(watch->ino)].
- error: path_put(&parent_path); audit_put_watch(watch); return ret.

REQ-9: audit_add_to_parent(krule, parent) (caller holds audit_filter_mutex):
- BUG_ON(!mutex_is_locked(&audit_filter_mutex)).
- For each existing watch w in parent->watches:
  - if strcmp(watch->path, w->path) == 0: /* duplicate path: collapse onto existing */
    - audit_put_watch(watch). /* drop krule's ref to temporary watch (init_watch's count=1) */
    - audit_get_watch(w). /* take ref to surviving watch */
    - krule->watch = watch = w.
    - audit_put_parent(parent). /* parent ref returned by audit_find_parent / audit_init_parent is no longer ours */
    - break.
- if not found:
  - watch->parent = parent. /* inherits the parent ref */
  - audit_get_watch(watch). /* parent->watches list holds an additional ref */
  - list_add(&watch->wlist, &parent->watches).
- list_add(&krule->rlist, &watch->rules). /* link this krule into the watch */

REQ-10: audit_watch_handle_event(inode_mark, mask, inode, dir, dname, cookie):
- parent = container_of(inode_mark, struct audit_parent, mark).
- if WARN_ON_ONCE(inode_mark->group != audit_watch_group): return 0.
- /* CREATE or MOVED_TO with new child inode -> bind */
- if (mask & (FS_CREATE | FS_MOVED_TO)) ∧ inode:
  - audit_update_watch(parent, dname, inode->i_sb->s_dev, inode->i_ino, 0).
- /* DELETE or MOVED_FROM -> clear dev/ino; invalidating=1 forces audit_filter_inodes on current */
- else if mask & (FS_DELETE | FS_MOVED_FROM):
  - audit_update_watch(parent, dname, AUDIT_DEV_UNSET, AUDIT_INO_UNSET, 1).
- /* Self-vanish events -> tear down parent + all child watches */
- else if mask & (FS_DELETE_SELF | FS_UNMOUNT | FS_MOVE_SELF):
  - audit_remove_parent_watches(parent).
- return 0.

REQ-11: audit_update_watch(parent, dname, dev, ino, invalidating):
- mutex_lock(audit_filter_mutex).
- For each owatch in parent->watches matching audit_compare_dname_path(dname, owatch->path, AUDIT_NAME_FULL):
  - if invalidating ∧ !audit_dummy_context(): audit_filter_inodes(current, audit_context()). /* avoid losing records about the just-deleted child */
  - nwatch = audit_dupe_watch(owatch); on ENOMEM: mutex_unlock; audit_panic("error updating watch, skipping"); return.
  - nwatch->dev = dev; nwatch->ino = ino.
  - For each r in owatch->rules:
    - oentry = container_of(r, struct audit_entry, rule).
    - list_del(&oentry->rule.rlist); list_del_rcu(&oentry->list).
    - nentry = audit_dupe_rule(&oentry->rule).
    - if IS_ERR(nentry): list_del(&oentry->rule.list); audit_panic("error updating watch, removing").
    - else:
      - h = audit_hash_ino(ino).
      - audit_put_watch(nentry->rule.watch). /* drop the dup'd reference */
      - audit_get_watch(nwatch); nentry->rule.watch = nwatch.
      - list_add(&nentry->rule.rlist, &nwatch->rules).
      - list_add_rcu(&nentry->list, &audit_inode_hash[h]).
      - list_replace(&oentry->rule.list, &nentry->rule.list).
    - if oentry->rule.exe: audit_remove_mark(oentry->rule.exe).
    - call_rcu(&oentry->rcu, audit_free_rule_rcu).
  - audit_remove_watch(owatch).
  - goto add_watch_to_parent. /* event applies to a single watch by exact name */
- mutex_unlock; return.
- add_watch_to_parent: list_add(&nwatch->wlist, &parent->watches); mutex_unlock; return.

REQ-12: audit_remove_watch_rule(krule):
- watch = krule->watch; parent = watch->parent.
- list_del(&krule->rlist).
- if list_empty(&watch->rules):
  - audit_get_parent(parent). /* audit_remove_watch drops parent ref, but we still need it for `list_empty(&parent->watches)` below */
  - audit_remove_watch(watch). /* parent ref consumed inside */
  - if list_empty(&parent->watches): fsnotify_destroy_mark(&parent->mark, audit_watch_group). /* mark .free_mark callback frees the parent */
  - audit_put_parent(parent).

REQ-13: audit_remove_parent_watches(parent):
- mutex_lock(audit_filter_mutex).
- For each (w, nextw) in parent->watches:
  - For each (r, nextr) in w->rules:
    - e = container_of(r, struct audit_entry, rule).
    - audit_watch_log_rule_change(r, w, "remove_rule").
    - if e->rule.exe: audit_remove_mark(e->rule.exe).
    - list_del(&r->rlist); list_del(&r->list); list_del_rcu(&e->list).
    - call_rcu(&e->rcu, audit_free_rule_rcu).
  - audit_remove_watch(w).
- mutex_unlock.
- fsnotify_destroy_mark(&parent->mark, audit_watch_group). /* triggers .free_mark -> audit_watch_free_mark -> audit_free_parent */

REQ-14: audit_dupe_watch(old):
- path = kstrdup(old->path, GFP_KERNEL); ENOMEM-on-fail.
- new = audit_init_watch(path); on err: kfree(path); goto out.
- new->dev = old->dev; new->ino = old->ino.
- audit_get_parent(old->parent); new->parent = old->parent.
- /* new->rules empty; new->wlist undefined until caller list_adds */
- return new.

REQ-15: audit_watch_compare(watch, ino, dev):
- return (watch->ino != AUDIT_INO_UNSET) ∧ (watch->ino == ino) ∧ (watch->dev == dev).
- /* Negative-cache: AUDIT_INO_UNSET never matches; rule remains inert until first FS_CREATE rebinds. */

REQ-16: audit_watch_log_rule_change(krule, watch, op):
- if !audit_enabled: return.
- ab = audit_log_start(audit_context(), GFP_NOFS, AUDIT_CONFIG_CHANGE). /* GFP_NOFS — runs from fsnotify-event context which may be on the writeback path */
- audit_log_session_info(ab); audit_log_format(ab, "op=%s path=", op); audit_log_untrustedstring(ab, w->path); audit_log_key(ab, r->filterkey); audit_log_format(ab, " list=%d res=1", r->listnr).
- audit_log_end(ab).

REQ-17: audit_dupe_exe(new_krule, old_krule):
- pathname = kstrdup(audit_mark_path(old->exe), GFP_KERNEL); ENOMEM-on-fail.
- audit_mark = audit_alloc_mark(new, pathname, strlen(pathname)).
- if IS_ERR(audit_mark): kfree(pathname); return PTR_ERR.
- new->exe = audit_mark.
- return 0.

REQ-18: audit_exe_compare(tsk, mark):
- if tsk != current: return 0. /* exe filtering only applies to the recording task */
- if !current->mm: return 0.
- exe_file = get_mm_exe_file(current->mm); if !exe_file: return 0.
- ino = file_inode(exe_file)->i_ino; dev = file_inode(exe_file)->i_sb->s_dev.
- fput(exe_file).
- return audit_mark_compare(mark, ino, dev).

REQ-19: audit_watch_init (device_initcall):
- audit_watch_group = fsnotify_alloc_group(&audit_watch_fsnotify_ops, 0).
- if IS_ERR(audit_watch_group): audit_watch_group = NULL; audit_panic("cannot create audit fsnotify group").
- audit_watch_fsnotify_ops = { .handle_inode_event = audit_watch_handle_event, .free_mark = audit_watch_free_mark }.

## Acceptance Criteria

- [ ] AC-1: `auditctl -w /etc/passwd -p wa -k passwd_changes` → kernel creates one audit_parent on `/etc/` inode, one audit_watch on `passwd` name; rule is on AUDIT_FILTER_EXIT list; krule->watch->{dev,ino} bound to passwd's inode at insertion time.
- [ ] AC-2: `auditctl -w nonabs -p wa` (relative path) or `auditctl -w /etc/` (trailing slash) → audit_to_watch returns -EINVAL.
- [ ] AC-3: `auditctl -A user,always -w /tmp/x -p wa` (watch on user filter, not exit) → audit_to_watch returns -EINVAL.
- [ ] AC-4: cat > /etc/passwd via tmp+rename (atomic replace) → FS_MOVED_FROM on old inode clears (dev,ino) to UNSET; FS_MOVED_TO on new inode rebinds; new audit_entry replaces old; subsequent syscalls match new inode.
- [ ] AC-5: Delete watched file → FS_DELETE clears (dev,ino) to UNSET; rule survives, dormant.
- [ ] AC-6: Recreate the previously-watched file (same parent, same name) → FS_CREATE rebinds new inode; subsequent syscalls match.
- [ ] AC-7: `umount` the filesystem containing a watched path → FS_UNMOUNT on the parent triggers audit_remove_parent_watches; all child watches torn down; AUDIT_CONFIG_CHANGE op=remove_rule logged for each rule.
- [ ] AC-8: Delete watched parent directory → FS_DELETE_SELF same teardown as AC-7.
- [ ] AC-9: Two krules with the same path → share one audit_watch (audit_add_to_parent path-strcmp dedup); refcount on watch reflects both rules.
- [ ] AC-10: `auditctl -D` while file is being deleted concurrently → no UAF (refcount on watch + parent strict).
- [ ] AC-11: audit_filter_mutex never held across kern_path_parent (audit_add_watch's drop-relock pattern).
- [ ] AC-12: audit_watch_handle_event runs under fsnotify-event context with GFP_NOFS for any allocation it triggers (audit_watch_log_rule_change uses GFP_NOFS).
- [ ] AC-13: kthread / kernel-only path: audit_watch_init runs at device_initcall; failure path sets audit_watch_group = NULL and panics via audit_panic (continues unaudited).
- [ ] AC-14: `auditctl -w /usr/bin/foo -F exe=/usr/bin/foo` → audit_exe_compare matches when /proc/self/exe maps to that inode.

## Architecture

```
struct AuditWatch {
  count:   AtomicU32,                 // refcount_t
  dev:     DevT,                      // AUDIT_DEV_UNSET when unbound
  path:    KBox<KStr>,                // owned; from kstrdup of user input
  ino:     u64,                       // AUDIT_INO_UNSET when unbound
  parent:  Option<KArc<AuditParent>>,
  wlist:   ListLink,                  // -> parent.watches
  rules:   ListHead,                  // -> krule.rlist
}

struct AuditParent {
  watches: ListHead,                  // -> AuditWatch.wlist
  mark:    FsnotifyMark,              // embedded; refcount via fsnotify
}
```

`AuditWatch::to_watch(krule, path, len, op) -> Result<()>`:
1. if !AUDIT_WATCH_GROUP.get(): return Err(EOPNOTSUPP).
2. if path[0] != b'/' || path[len-1] == b'/': return Err(EINVAL).
3. if krule.listnr ∉ {AUDIT_FILTER_EXIT, AUDIT_FILTER_URING_EXIT}: return Err(EINVAL).
4. if op != Audit_equal: return Err(EINVAL).
5. if krule.inode_f.is_some() || krule.watch.is_some() || krule.tree.is_some(): return Err(EINVAL).
6. watch = AuditWatch::init(path)?.
7. krule.watch = Some(watch).
8. Ok(()).

`AuditWatch::add(krule, list_out) -> Result<()>` (caller holds audit_filter_mutex):
1. watch = krule.watch.clone(). /* extra ref */
2. audit_filter_mutex.unlock(). /* path lookup cannot run under mutex */
3. ret = AuditWatch::get_nd(&watch, &mut parent_path).
4. audit_filter_mutex.lock().
5. if ret.is_err(): drop(watch); return ret.
6. parent = AuditParent::find(d_backing_inode(parent_path.dentry)).
7. if parent.is_none(): parent = AuditParent::init(&parent_path)?.
8. AuditWatch::add_to_parent(krule, parent).
9. *list_out = &AUDIT_INODE_HASH[audit_hash_ino(watch.ino)].
10. path_put(parent_path); drop(watch); Ok(()).

`AuditWatch::get_nd(watch, parent_path_out) -> Result<()>`:
1. d = kern_path_parent(&watch.path, parent_path_out)?.
2. if d_is_positive(d):
   - watch.dev = d.d_sb.s_dev.
   - watch.ino = d_backing_inode(d).i_ino.
3. dput(d).
4. Ok(()).

`AuditWatch::handle_event(mark, mask, inode_opt, dir_opt, dname, cookie) -> i32`:
1. parent = container_of(mark, AuditParent, mark).
2. if mark.group != AUDIT_WATCH_GROUP: WARN_ON_ONCE; return 0.
3. if mask.contains(FS_CREATE | FS_MOVED_TO) ∧ inode_opt.is_some():
   - AuditWatch::update(parent, dname, inode.i_sb.s_dev, inode.i_ino, /*invalidating=*/0).
4. else if mask.contains(FS_DELETE | FS_MOVED_FROM):
   - AuditWatch::update(parent, dname, AUDIT_DEV_UNSET, AUDIT_INO_UNSET, /*invalidating=*/1).
5. else if mask.contains(FS_DELETE_SELF | FS_UNMOUNT | FS_MOVE_SELF):
   - AuditParent::remove_watches(parent).
6. 0.

`AuditWatch::update(parent, dname, dev, ino, invalidating)`:
1. audit_filter_mutex.lock().
2. for owatch in parent.watches.iter_safe():
   - if audit_compare_dname_path(dname, &owatch.path, AUDIT_NAME_FULL) != 0: continue.
   - if invalidating ∧ !audit_dummy_context(): audit_filter_inodes(current(), audit_context()).
   - nwatch = AuditWatch::dupe(owatch).map_err(panic("error updating watch, skipping"))?.
   - nwatch.dev = dev; nwatch.ino = ino.
   - for r in owatch.rules.iter_safe():
     - oentry = container_of(r, AuditEntry, rule).
     - list_del(&oentry.rule.rlist); list_del_rcu(&oentry.list).
     - nentry = audit_dupe_rule(&oentry.rule).
     - if Ok(nentry):
       - h = audit_hash_ino(ino).
       - audit_put_watch(nentry.rule.watch). audit_get_watch(&nwatch). nentry.rule.watch = nwatch.clone().
       - list_add(&nentry.rule.rlist, &nwatch.rules).
       - list_add_rcu(&nentry.list, &AUDIT_INODE_HASH[h]).
       - list_replace(&oentry.rule.list, &nentry.rule.list).
     - else: list_del(&oentry.rule.list); audit_panic("error updating watch, removing").
     - if oentry.rule.exe.is_some(): audit_remove_mark(oentry.rule.exe).
     - call_rcu(&oentry.rcu, audit_free_rule_rcu).
   - AuditWatch::remove(owatch).
   - list_add(&nwatch.wlist, &parent.watches).
   - audit_filter_mutex.unlock(); return. /* event applies to one watch */
3. audit_filter_mutex.unlock().

`AuditWatch::remove_rule(krule)`:
1. watch = krule.watch.clone(); parent = watch.parent.clone().
2. list_del(&krule.rlist).
3. if watch.rules.is_empty():
   - AuditParent::get(parent). /* survive audit_remove_watch */
   - AuditWatch::remove(watch). /* drops watch->parent ref */
   - if parent.watches.is_empty(): fsnotify_destroy_mark(&parent.mark, AUDIT_WATCH_GROUP). /* .free_mark -> audit_watch_free_mark -> AuditParent::free */
   - AuditParent::put(parent).

`AuditParent::remove_watches(parent)`:
1. audit_filter_mutex.lock().
2. for w in parent.watches.iter_safe():
   - for r in w.rules.iter_safe():
     - e = container_of(r, AuditEntry, rule).
     - AuditWatch::log_rule_change(r, w, "remove_rule").
     - if e.rule.exe.is_some(): audit_remove_mark(e.rule.exe).
     - list_del(&r.rlist); list_del(&r.list); list_del_rcu(&e.list).
     - call_rcu(&e.rcu, audit_free_rule_rcu).
   - AuditWatch::remove(w).
3. audit_filter_mutex.unlock().
4. fsnotify_destroy_mark(&parent.mark, AUDIT_WATCH_GROUP).

`AuditWatch::module_init()` (device_initcall):
1. group = fsnotify_alloc_group(&AUDIT_WATCH_FSNOTIFY_OPS, 0).
2. if Err: AUDIT_WATCH_GROUP = None; audit_panic("cannot create audit fsnotify group"); return.
3. AUDIT_WATCH_GROUP = Some(group).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `watch_refcount_balanced` | INVARIANT | per-audit_init_watch: count=1; per-audit_put_watch on count=0: kfree path + watch. |
| `parent_refcount_via_fsnotify` | INVARIANT | per-audit_get_parent/_put_parent forward to fsnotify_get_mark/_put_mark; no parallel counter. |
| `mutex_dropped_for_path_lookup` | INVARIANT | per-audit_add_watch: audit_filter_mutex released before kern_path_parent, reacquired after. |
| `audit_filter_mutex_held_in_add_to_parent` | INVARIANT | per-audit_add_to_parent: BUG_ON(!mutex_is_locked(&audit_filter_mutex)). |
| `audit_to_watch_validates_path` | INVARIANT | per-audit_to_watch: path[0]=='/' ∧ path[len-1]!='/' ∧ listnr ∈ {EXIT, URING_EXIT} ∧ op==Audit_equal. |
| `unbound_watch_never_matches` | INVARIANT | per-audit_watch_compare: watch->ino == AUDIT_INO_UNSET ⟹ return 0. |
| `move_self_tears_down_parent` | INVARIANT | per-audit_watch_handle_event: FS_DELETE_SELF/UNMOUNT/MOVE_SELF ⟹ audit_remove_parent_watches. |
| `log_rule_change_gfp_nofs` | INVARIANT | per-audit_watch_log_rule_change: audit_log_start GFP_NOFS (fsnotify callback context). |
| `path_string_owned_by_watch_on_success` | INVARIANT | per-audit_to_watch: on success, watch->path = caller's pointer; caller no longer frees. |
| `exe_compare_only_for_current` | INVARIANT | per-audit_exe_compare: tsk != current ⟹ return 0 (cannot deref another task's mm). |

### Layer 2: TLA+

`kernel/audit/audit-watch.tla`:
- States: rule-insert / path-resolve / parent-find-or-create / watch-attach / fs-event / rule-replace / rule-remove / parent-teardown.
- Properties:
  - `safety_no_concurrent_path_lookup_under_mutex` — per-audit_add_watch: mutex never held during kern_path_parent.
  - `safety_watch_freed_iff_refcount_zero` — per-audit_put_watch: kfree iff refcount_dec_and_test.
  - `safety_parent_freed_iff_watches_empty` — per-audit_remove_watch_rule: parent destroyed iff list_empty(&parent->watches).
  - `safety_rule_replace_atomic_under_rcu` — per-audit_update_watch: list_replace_rcu(oentry, nentry); readers see exactly one.
  - `safety_dupe_inherits_parent_ref` — per-audit_dupe_watch: audit_get_parent before storing.
  - `liveness_fs_event_eventually_handled` — per-fsnotify-delivery: every queued event yields one handle_inode_event call.
  - `liveness_parent_teardown_completes` — per-audit_remove_parent_watches: all child watches removed; fsnotify_destroy_mark issued.
  - `safety_dev_ino_unset_after_invalidating_event` — per-FS_DELETE/FS_MOVED_FROM: watch->{dev,ino} = AUDIT_*_UNSET.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `AuditWatch::init` post: count==1 ∧ rules.empty() ∧ dev/ino == UNSET | `AuditWatch::init` |
| `AuditWatch::to_watch` post: ret==0 ⟹ krule.watch.is_some() ∧ owns path | `AuditWatch::to_watch` |
| `AuditWatch::add` post: ret==0 ⟹ krule.rlist linked into watch.rules ∧ watch linked into parent.watches ∧ list_out points at audit_inode_hash bucket | `AuditWatch::add` |
| `AuditWatch::add_to_parent` post: dup-path collapse ⟹ krule.watch = existing; new-path ⟹ watch.parent set ∧ refcount++ | `AuditWatch::add_to_parent` |
| `AuditWatch::remove_rule` post: list_empty(watch.rules) ⟹ watch freed; list_empty(parent.watches) ⟹ fsnotify_destroy_mark | `AuditWatch::remove_rule` |
| `AuditWatch::update` post: matched owatch replaced by nwatch with new (dev,ino); per-rule audit_entry replaced under RCU | `AuditWatch::update` |
| `AuditWatch::handle_event` post: per-mask dispatched to update/remove/teardown | `AuditWatch::handle_event` |
| `AuditWatch::get_nd` post: ret==0 ∧ d_is_positive ⟹ watch.{dev,ino} bound; ret==0 ∧ negative dentry ⟹ watch.{dev,ino} == UNSET | `AuditWatch::get_nd` |
| `AuditWatch::log_rule_change` post: audit_log_start uses GFP_NOFS | `AuditWatch::log_rule_change` |
| `AuditWatch::exe_compare` post: tsk != current ⟹ 0; otherwise calls audit_mark_compare with current.mm.exe inode | `AuditWatch::exe_compare` |

### Layer 4: Verus/Creusot functional

`Per-auditctl -w <path> -p <perms> -k <key>` netlink message → audit_data_to_entry parses AUDIT_WATCH field → audit_to_watch + audit_add_watch → fsnotify mark on parent dir → mutation events (CREATE / DELETE / MOVE / DELETE_SELF / UNMOUNT) drive `audit_update_watch` / `audit_remove_parent_watches` → semantic equivalence with upstream per Documentation/audit/audit.txt + audit-userspace `auditctl -l` round-trip byte-identical.

## Hardening

(Inherits row-1 features from `kernel/audit/00-overview.md` § Hardening.)

audit_watch reinforcement:

- **Per-absolute-path-required (path[0]=='/')** — defense against per-cwd-relative watch ambiguity and per-chroot escape.
- **Per-no-trailing-slash (path[len-1]!='/')** — defense against per-directory-vs-file watch confusion (use audit_tree.c for directories).
- **Per-listnr-restricted-to-{EXIT, URING_EXIT}** — defense against per-watch-on-user-or-task-filter which cannot evaluate path.
- **Per-op==Audit_equal-only** — defense against per-non-sensical-comparators on a string field.
- **Per-mutual-exclusion-{inode_f, watch, tree}** — defense against per-conflicting-path-resolvers.
- **Per-refcount strict on audit_watch (refcount_t)** — defense against per-watch UAF when fsnotify event races with auditctl -D.
- **Per-fsnotify_get/put_mark for audit_parent** — defense against per-parent UAF when last watch leaves before mark teardown completes.
- **Per-audit_filter_mutex never held during kern_path_parent** — defense against per-deadlock on path-walk that may take fs-internal locks.
- **Per-audit_log_start GFP_NOFS in audit_watch_log_rule_change** — defense against per-FS-deadlock from log allocations in fsnotify callback.
- **Per-AUDIT_DEV_UNSET / AUDIT_INO_UNSET sentinel** — defense against per-watch-firing-on-stale-inode after delete.
- **Per-audit_panic-on-init-failure** — defense against per-silently-disabled-watch-subsystem (audit_watch_group NULL is checked by audit_to_watch with EOPNOTSUPP).
- **Per-audit_exe_compare tsk==current guard** — defense against per-cross-task mm dereference.
- **Per-call_rcu for entry free** — defense against per-RCU-reader UAF during filter list traversal.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `kernel/auditfilter.c` rule filter engine (covered in `auditfilter.md`).
- `kernel/audit_tree.c` recursive directory watches (covered in `audit-tree.md`).
- `kernel/audit_fsnotify.c` AUDIT_EXE mark management (covered in `audit-fsnotify.md`).
- `fs/notify/` fsnotify core (covered separately).
- `kernel/auditsc.c` syscall context (covered in `auditsc.md`).
- Implementation code.
