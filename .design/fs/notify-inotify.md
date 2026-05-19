# Tier-3: fs/notify/inotify/inotify_user.c — inotify(7) user-facing implementation

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/notify/inotify/inotify_user.c (~861 lines)
  - fs/notify/inotify/inotify_fsnotify.c
  - fs/notify/inotify/inotify.h
  - include/linux/inotify.h
  - include/uapi/linux/inotify.h (IN_ACCESS / IN_MODIFY / IN_ATTRIB / IN_CLOSE_WRITE / IN_CLOSE_NOWRITE / IN_OPEN / IN_MOVED_FROM / IN_MOVED_TO / IN_CREATE / IN_DELETE / IN_DELETE_SELF / IN_MOVE_SELF / IN_Q_OVERFLOW / IN_IGNORED / IN_ISDIR / IN_CLOEXEC / IN_NONBLOCK / IN_MASK_ADD / IN_MASK_CREATE / IN_ONESHOT / IN_EXCL_UNLINK / IN_DONT_FOLLOW / IN_ONLYDIR)
-->

## Summary

`fs/notify/inotify/inotify_user.c` is the user-facing entry point for the **inotify(7)** filesystem-event API. It registers three syscalls (`inotify_init`, `inotify_init1`, `inotify_add_watch`, `inotify_rm_watch`), provides the anon-inode file with `inotify_fops` (`.poll`, `.read`, `.fasync`, `.release`, `.unlocked_ioctl`, `.show_fdinfo`), and bridges per-fd state into the generic fsnotify-core. Each `inotify_init` call obtains a new `struct fsnotify_group` (`inotify_new_group`) with an integer-IDR mapping **watch descriptors** (`wd`) to `struct inotify_inode_mark` wrappers around `struct fsnotify_mark`. Events accumulate as `struct inotify_event_info` on `group->notification_list`; `inotify_read` blocks on `notification_waitq`, dequeues one event at a time via `get_one_event`, and emits the variable-length `struct inotify_event { wd, mask, cookie, len, name[0] }` to userspace. Mask bits (`IN_*`) map 1:1 to fsnotify (`FS_*`) via compile-time `BUILD_BUG_ON` equivalences. Per-user quotas: `UCOUNT_INOTIFY_INSTANCES` and `UCOUNT_INOTIFY_WATCHES` (sysctls `fs.inotify.max_user_instances` / `max_user_watches` / `max_queued_events`).

This Tier-3 covers `fs/notify/inotify/inotify_user.c` (~861 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct inotify_inode_mark` | inotify-side mark wrapper (fsn_mark + wd) | `InotifyInodeMark` |
| `struct inotify_event_info` | inotify-side event payload (mask, wd, cookie, name_len, name[]) | `InotifyEventInfo` |
| `INOTIFY_E(fsn_event)` | container_of from fsnotify_event | `InotifyEventInfo::from_fsn_event` |
| `inotify_fops` | per-fd ops table | `Inotify::FOPS` |
| `inotify_poll()` | per-poll readiness | `Inotify::poll` |
| `inotify_read()` | per-read blocking dequeue | `Inotify::read` |
| `inotify_release()` | per-close fsnotify_destroy_group | `Inotify::release` |
| `inotify_ioctl()` | per-FIONREAD / per-INOTIFY_IOC_SETNEXTWD | `Inotify::ioctl` |
| `get_one_event()` | per-read peek+remove with size check | `Inotify::get_one_event` |
| `copy_event_to_user()` | per-emit inotify_event + name | `Inotify::copy_event_to_user` |
| `round_event_name_len()` | per-name pad to sizeof(inotify_event) | `Inotify::round_event_name_len` |
| `inotify_arg_to_mask()` | per-IN_* → FS_* + FS_UNMOUNT + FS_EVENT_ON_CHILD | `Inotify::arg_to_mask` |
| `inotify_arg_to_flags()` | per-IN_EXCL_UNLINK / IN_ONESHOT → mark flags | `Inotify::arg_to_flags` |
| `inotify_mask_to_arg()` | per-FS_* → IN_* for emit | `Inotify::mask_to_arg` |
| `inotify_find_inode()` | per-path resolve + MAY_READ + LSM | `Inotify::find_inode` |
| `inotify_add_to_idr()` / `inotify_remove_from_idr()` / `inotify_idr_find()` / `inotify_idr_find_locked()` | per-wd↔mark idr management | `Inotify::add_to_idr` / `remove_from_idr` / `idr_find` / `idr_find_locked` |
| `inotify_update_existing_watch()` | per-IN_MASK_ADD / replace | `Inotify::update_existing_watch` |
| `inotify_new_watch()` | per-new mark create | `Inotify::new_watch` |
| `inotify_update_watch()` | per-add_watch dispatch | `Inotify::update_watch` |
| `inotify_new_group()` | per-init group alloc | `Inotify::new_group` |
| `inotify_ignored_and_remove_idr()` | per-rm queue IN_IGNORED + idr drop | `Inotify::ignored_and_remove_idr` |
| `do_inotify_init()` | per-syscall common | `Inotify::do_init` |
| `SYSCALL_DEFINE1(inotify_init1)` / `SYSCALL_DEFINE0(inotify_init)` | per-uapi init | `Inotify::sys_init` / `sys_init1` |
| `SYSCALL_DEFINE3(inotify_add_watch)` / `SYSCALL_DEFINE2(inotify_rm_watch)` | per-uapi watch | `Inotify::sys_add_watch` / `sys_rm_watch` |
| `inotify_inode_mark_cachep` | per-inotify mark slab | `Inotify::INODE_MARK_CACHE` |
| `inotify_max_queued_events` | per-init max_events default (=16384) | `Inotify::MAX_QUEUED_EVENTS` |
| `fs.inotify.max_user_instances` / `_user_watches` / `_queued_events` | per-sysctl | shared |

## Compatibility contract

REQ-1: struct inotify_inode_mark:
- fsn_mark: struct fsnotify_mark (embedded).
- wd: int (watch descriptor; -1 if not on idr).

REQ-2: struct inotify_event_info:
- fse: struct fsnotify_event (embedded list head).
- mask: __u32 (FS_* event mask).
- wd: __s32.
- sync_cookie: __u32 (paired across rename pair via fsnotify_get_cookie).
- name_len: int (0 if no name).
- name: char[] (NUL-terminated; flexible array).

REQ-3: struct inotify_event (uapi):
- wd: __s32.
- mask: __u32 (IN_*).
- cookie: __u32.
- len: __u32 (padded name field length; multiple of sizeof(struct inotify_event)).
- name: __u8[] (len bytes; NUL-padded to multiple of sizeof(struct inotify_event)).

REQ-4: Event bit equivalence (compile-time BUILD_BUG_ON in inotify_user_setup):
- IN_ACCESS == FS_ACCESS.
- IN_MODIFY == FS_MODIFY.
- IN_ATTRIB == FS_ATTRIB.
- IN_CLOSE_WRITE == FS_CLOSE_WRITE.
- IN_CLOSE_NOWRITE == FS_CLOSE_NOWRITE.
- IN_OPEN == FS_OPEN.
- IN_MOVED_FROM == FS_MOVED_FROM.
- IN_MOVED_TO == FS_MOVED_TO.
- IN_CREATE == FS_CREATE.
- IN_DELETE == FS_DELETE.
- IN_DELETE_SELF == FS_DELETE_SELF.
- IN_MOVE_SELF == FS_MOVE_SELF.
- IN_UNMOUNT == FS_UNMOUNT.
- IN_Q_OVERFLOW == FS_Q_OVERFLOW.
- IN_IGNORED == FS_IN_IGNORED.
- IN_ISDIR == FS_ISDIR.
- HWEIGHT32(ALL_INOTIFY_BITS) == 22.

REQ-5: inotify_arg_to_mask(inode, arg):
- mask = FS_UNMOUNT.
- if S_ISDIR(inode.i_mode): mask |= FS_EVENT_ON_CHILD.
- mask |= (arg & INOTIFY_USER_MASK).
- return mask.

REQ-6: inotify_arg_to_flags(arg):
- flags = 0.
- if arg & IN_EXCL_UNLINK: flags |= FSNOTIFY_MARK_FLAG_EXCL_UNLINK.
- if arg & IN_ONESHOT: flags |= FSNOTIFY_MARK_FLAG_IN_ONESHOT.
- return flags.

REQ-7: inotify_mask_to_arg(mask):
- return mask & (IN_ALL_EVENTS | IN_ISDIR | IN_UNMOUNT | IN_IGNORED | IN_Q_OVERFLOW).

REQ-8: do_inotify_init(flags):
- BUILD_BUG_ON(IN_CLOEXEC != O_CLOEXEC); BUILD_BUG_ON(IN_NONBLOCK != O_NONBLOCK).
- if flags & ~(IN_CLOEXEC | IN_NONBLOCK): return -EINVAL.
- group = inotify_new_group(inotify_max_queued_events); if IS_ERR: return PTR_ERR.
- ret = anon_inode_getfd("inotify", &inotify_fops, group, O_RDONLY | flags).
- if ret < 0: fsnotify_destroy_group(group).
- return ret.

REQ-9: SYSCALL_DEFINE0(inotify_init): return do_inotify_init(0).
SYSCALL_DEFINE1(inotify_init1, int, flags): return do_inotify_init(flags).

REQ-10: SYSCALL_DEFINE3(inotify_add_watch, fd, pathname, mask):
- if mask & ~ALL_INOTIFY_BITS: return -EINVAL.
- if !(mask & ALL_INOTIFY_BITS): return -EINVAL.
- fd → f via CLASS(fd, f)(fd); if fd_empty: -EBADF.
- if (mask & IN_MASK_ADD) ∧ (mask & IN_MASK_CREATE): -EINVAL.
- if f.f_op != &inotify_fops: -EINVAL.
- flags = 0; if !(mask & IN_DONT_FOLLOW): flags |= LOOKUP_FOLLOW; if mask & IN_ONLYDIR: flags |= LOOKUP_DIRECTORY.
- inotify_find_inode(pathname, &path, flags, mask & IN_ALL_EVENTS):
  - user_path_at(AT_FDCWD, dirname, flags, path).
  - path_permission(path, MAY_READ).
  - security_path_notify(path, mask, FSNOTIFY_OBJ_TYPE_INODE).
- inode = path.dentry.d_inode; group = f.private_data.
- ret = inotify_update_watch(group, inode, mask); path_put(&path); return ret.

REQ-11: SYSCALL_DEFINE2(inotify_rm_watch, fd, wd):
- CLASS(fd, f)(fd); if fd_empty: -EBADF.
- if f.f_op != &inotify_fops: -EINVAL.
- group = f.private_data.
- i_mark = inotify_idr_find(group, wd); if !i_mark: -EINVAL.
- fsnotify_destroy_mark(&i_mark.fsn_mark, group). (queues IN_IGNORED via inotify_freeing_mark → inotify_ignored_and_remove_idr; removes from idr; decs ucounts).
- fsnotify_put_mark(&i_mark.fsn_mark) (match get from inotify_idr_find).
- return 0.

REQ-12: inotify_update_watch(group, inode, arg):
- fsnotify_group_lock(group) (group.mark_mutex).
- ret = inotify_update_existing_watch(group, inode, arg).
- if ret == -ENOENT: ret = inotify_new_watch(group, inode, arg).
- fsnotify_group_unlock(group).
- return ret.

REQ-13: inotify_update_existing_watch(group, inode, arg):
- replace = !(arg & IN_MASK_ADD); create = (arg & IN_MASK_CREATE).
- fsn_mark = fsnotify_find_inode_mark(inode, group); if !fsn_mark: return -ENOENT.
- if create: -EEXIST.
- i_mark = container_of(fsn_mark, struct inotify_inode_mark, fsn_mark).
- spin_lock(&fsn_mark.lock).
- old_mask = fsn_mark.mask.
- if replace: fsn_mark.mask = 0; fsn_mark.flags &= ~INOTIFY_MARK_FLAGS.
- fsn_mark.mask |= inotify_arg_to_mask(inode, arg).
- fsn_mark.flags |= inotify_arg_to_flags(arg).
- new_mask = fsn_mark.mask; spin_unlock.
- if old_mask != new_mask: if (old_mask & ~new_mask) ∨ (new_mask & ~READ_ONCE(inode.i_fsnotify_mask)): fsnotify_recalc_mask(fsn_mark.connector).
- ret = i_mark.wd; fsnotify_put_mark(fsn_mark); return ret.

REQ-14: inotify_new_watch(group, inode, arg):
- tmp_i_mark = kmem_cache_alloc(inotify_inode_mark_cachep, GFP_KERNEL); if !tmp_i_mark: -ENOMEM.
- fsnotify_init_mark(&tmp_i_mark.fsn_mark, group).
- tmp_i_mark.fsn_mark.mask = inotify_arg_to_mask(inode, arg).
- tmp_i_mark.fsn_mark.flags = inotify_arg_to_flags(arg).
- tmp_i_mark.wd = -1.
- ret = inotify_add_to_idr(idr, idr_lock, tmp_i_mark); if ret: goto out_err.
- if !inc_inotify_watches(group.inotify_data.ucounts): inotify_remove_from_idr; ret = -ENOSPC; goto out_err.
- ret = fsnotify_add_inode_mark_locked(&tmp_i_mark.fsn_mark, inode, 0); if ret: remove_from_idr + dec_inotify_watches; goto out_err.
- ret = tmp_i_mark.wd.
- out_err: fsnotify_put_mark(&tmp_i_mark.fsn_mark) (matches init).
- return ret.

REQ-15: inotify_add_to_idr(idr, idr_lock, i_mark):
- idr_preload(GFP_KERNEL); spin_lock(idr_lock).
- ret = idr_alloc_cyclic(idr, i_mark, 1, 0, GFP_NOWAIT).
- if ret >= 0: i_mark.wd = ret; fsnotify_get_mark(&i_mark.fsn_mark).
- spin_unlock(idr_lock); idr_preload_end().
- return ret < 0 ? ret : 0.

REQ-16: inotify_remove_from_idr(group, i_mark):
- spin_lock(idr_lock); wd = i_mark.wd.
- if wd == -1: WARN_ONCE; goto out.
- found_i_mark = inotify_idr_find_locked(group, wd).
- WARN if !found or found != i_mark.
- BUG_ON refcount < 2 (idr ref + locked-find ref).
- idr_remove(idr, wd); fsnotify_put_mark(&i_mark.fsn_mark) (drops idr ref).
- out: i_mark.wd = -1; spin_unlock(idr_lock).
- if found_i_mark: fsnotify_put_mark(&found_i_mark.fsn_mark) (drops locked-find ref).

REQ-17: inotify_ignored_and_remove_idr(fsn_mark, group):
- inotify_handle_inode_event(fsn_mark, FS_IN_IGNORED, NULL, NULL, NULL, 0).
- i_mark = container_of(fsn_mark, struct inotify_inode_mark, fsn_mark).
- inotify_remove_from_idr(group, i_mark).
- dec_inotify_watches(group.inotify_data.ucounts).

REQ-18: inotify_new_group(max_events):
- group = fsnotify_alloc_group(&inotify_fsnotify_ops, FSNOTIFY_GROUP_USER); if IS_ERR: return.
- oevent = kmalloc_obj(struct inotify_event_info, GFP_KERNEL_ACCOUNT); if !oevent: destroy + -ENOMEM.
- group.overflow_event = &oevent.fse; fsnotify_init_event; oevent.mask = FS_Q_OVERFLOW; oevent.wd = -1; oevent.sync_cookie = 0; oevent.name_len = 0.
- group.max_events = max_events; group.memcg = get_mem_cgroup_from_mm(current.mm).
- spin_lock_init(&group.inotify_data.idr_lock); idr_init(&group.inotify_data.idr).
- group.inotify_data.ucounts = inc_ucount(current_user_ns, current_euid, UCOUNT_INOTIFY_INSTANCES).
- if !ucounts: destroy + -EMFILE.
- return group.

REQ-19: inotify_read(file, buf, count, pos):
- start = buf; group = file.private_data.
- add_wait_queue(&group.notification_waitq, &wait).
- loop:
  - spin_lock(&group.notification_lock); kevent = get_one_event(group, count); spin_unlock.
  - if kevent:
    - if IS_ERR(kevent): ret = PTR_ERR(kevent); break.
    - ret = copy_event_to_user(group, kevent, buf).
    - fsnotify_destroy_event(group, kevent).
    - if ret < 0: break.
    - buf += ret; count -= ret; continue.
  - ret = -EAGAIN; if O_NONBLOCK: break.
  - ret = -ERESTARTSYS; if signal_pending(current): break.
  - if start != buf: break.
  - wait_woken(&wait, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT).
- remove_wait_queue.
- if start != buf ∧ ret != -EFAULT: ret = buf - start.
- return ret.

REQ-20: get_one_event(group, count):
- event_size = sizeof(struct inotify_event).
- event = fsnotify_peek_first_event(group); if !event: return NULL.
- event_size += round_event_name_len(event).
- if event_size > count: return ERR_PTR(-EINVAL).
- fsnotify_remove_first_event(group).
- return event.

REQ-21: copy_event_to_user(group, fsn_event, buf):
- event = INOTIFY_E(fsn_event); name_len = event.name_len; pad_name_len = round_event_name_len(fsn_event).
- inotify_event.len = pad_name_len; .mask = inotify_mask_to_arg(event.mask); .wd = event.wd; .cookie = event.sync_cookie.
- copy_to_user(buf, &inotify_event, event_size); buf += event_size.
- if pad_name_len: copy_to_user(buf, event.name, name_len); buf += name_len; clear_user(buf, pad_name_len - name_len); event_size += pad_name_len.
- return event_size.

REQ-22: round_event_name_len(fsn_event):
- event = INOTIFY_E(fsn_event); if !event.name_len: return 0.
- return roundup(event.name_len + 1, sizeof(struct inotify_event)). /* +1 for NUL */

REQ-23: inotify_poll(file, wait):
- group = file.private_data.
- poll_wait(file, &group.notification_waitq, wait).
- spin_lock(&group.notification_lock); ready = !fsnotify_notify_queue_is_empty(group); spin_unlock.
- return ready ? (EPOLLIN | EPOLLRDNORM) : 0.

REQ-24: inotify_release(inode, file):
- group = file.private_data.
- fsnotify_destroy_group(group). (puts last reference; runs ops.free_group_priv → free overflow_event + idr_destroy + put ucounts + put memcg.)
- return 0.

REQ-25: inotify_ioctl(file, cmd, arg):
- group = file.private_data.
- FIONREAD: walk notification_list summing sizeof(inotify_event) + round_event_name_len; put_user to *p.
- INOTIFY_IOC_SETNEXTWD (CONFIG_CHECKPOINT_RESTORE): if 1 ≤ arg ≤ INT_MAX: idr_set_cursor(&data.idr, arg).
- default: -ENOTTY.

REQ-26: inotify_fops:
- .show_fdinfo = inotify_show_fdinfo (per-wd dump for /proc/PID/fdinfo).
- .poll = inotify_poll.
- .read = inotify_read.
- .fasync = fsnotify_fasync.
- .release = inotify_release.
- .unlocked_ioctl = inotify_ioctl.
- .compat_ioctl = inotify_ioctl.
- .llseek = noop_llseek.

REQ-27: Per-user quotas (inotify_user_setup):
- inotify_max_queued_events = 16384 (per-instance default).
- init_user_ns.ucount_max[UCOUNT_INOTIFY_INSTANCES] = 128 (per-uid instances).
- init_user_ns.ucount_max[UCOUNT_INOTIFY_WATCHES] = clamp((totalram - totalhigh)/100 << PAGE_SHIFT / INOTIFY_WATCH_COST, 8192, 1048576).
- INOTIFY_WATCH_COST = sizeof(struct inotify_inode_mark) + 2*sizeof(struct inode).
- Sysctls: fs.inotify.max_user_instances / _user_watches / _queued_events.

## Acceptance Criteria

- [ ] AC-1: inotify_init(0) returns fd backed by anon_inode "inotify" with O_RDONLY.
- [ ] AC-2: inotify_init1(IN_CLOEXEC | IN_NONBLOCK): fd has FD_CLOEXEC ∧ O_NONBLOCK; any other flag → -EINVAL.
- [ ] AC-3: inotify_add_watch on non-inotify fd: -EINVAL.
- [ ] AC-4: inotify_add_watch with mask & ~ALL_INOTIFY_BITS or mask == 0: -EINVAL.
- [ ] AC-5: inotify_add_watch with both IN_MASK_ADD and IN_MASK_CREATE: -EINVAL.
- [ ] AC-6: inotify_add_watch new mark: returns positive wd; subsequent rm_watch(wd) succeeds.
- [ ] AC-7: inotify_add_watch existing mark (no IN_MASK_ADD): replaces mask; returns same wd.
- [ ] AC-8: inotify_add_watch existing mark + IN_MASK_ADD: ORs mask; returns same wd.
- [ ] AC-9: inotify_add_watch existing mark + IN_MASK_CREATE: -EEXIST.
- [ ] AC-10: inotify_rm_watch invalid wd: -EINVAL; valid wd: queues IN_IGNORED event then removes mark.
- [ ] AC-11: inotify_read: empty queue + O_NONBLOCK: -EAGAIN; otherwise blocks until event or signal.
- [ ] AC-12: inotify_read: buffer smaller than next event: -EINVAL (if no bytes copied) else partial bytes returned.
- [ ] AC-13: inotify_read: emits {wd, mask, cookie, len, name[]} with len = roundup(name_len+1, sizeof(inotify_event)) and NUL-padded.
- [ ] AC-14: IN_ACCESS/MODIFY/ATTRIB/CLOSE_*/OPEN/MOVED_FROM/TO/CREATE/DELETE/DELETE_SELF/MOVE_SELF emitted with corresponding FS_* events from VFS.
- [ ] AC-15: Queue saturation (q_len ≥ max_events): subsequent events drop; IN_Q_OVERFLOW with wd=-1 enqueued once.
- [ ] AC-16: ucount limit exceeded: inotify_init → -EMFILE; inotify_add_watch → -ENOSPC.
- [ ] AC-17: inotify_poll: empty queue → 0; non-empty → EPOLLIN|EPOLLRDNORM.
- [ ] AC-18: FIONREAD ioctl returns total bytes pending in queue.

## Architecture

```
struct InotifyInodeMark {
  fsn_mark: FsnotifyMark,                   // embedded; container_of recovers
  wd: i32,                                  // -1 if not on idr
}

struct InotifyEventInfo {
  fse: FsnotifyEvent,                       // embedded; on group.notification_list
  mask: u32,                                // FS_*
  wd: i32,
  sync_cookie: u32,
  name_len: usize,
  name: [u8; _],                            // flexible array; NUL-terminated
}

#[repr(C)]
struct InotifyEventUapi {                   // wire format
  wd: i32,
  mask: u32,
  cookie: u32,
  len: u32,                                 // padded name field length
  // name: [u8; len]
}

const INOTIFY_USER_MASK: u32 = IN_ALL_EVENTS | IN_MASK_ADD | IN_MASK_CREATE | IN_ONESHOT
                              | IN_EXCL_UNLINK | IN_DONT_FOLLOW | IN_ONLYDIR;
const ALL_INOTIFY_BITS: u32 = /* 22 bits per BUILD_BUG_ON(HWEIGHT32 == 22) */;
const INOTIFY_WATCH_COST: usize = size_of::<InotifyInodeMark>() + 2 * size_of::<Inode>();
```

`Inotify::do_init(flags) -> Result<RawFd>`:
1. const_assert!(IN_CLOEXEC == O_CLOEXEC && IN_NONBLOCK == O_NONBLOCK).
2. if flags & !(IN_CLOEXEC | IN_NONBLOCK) != 0: return Err(EINVAL).
3. group = Inotify::new_group(inotify_max_queued_events)?;
4. fd = anon_inode_getfd("inotify", &INOTIFY_FOPS, group, O_RDONLY | flags);
5. if fd < 0: fsnotify_destroy_group(group); return Err(...).
6. return Ok(fd).

`Inotify::sys_add_watch(fd, pathname, mask) -> Result<i32>`:
1. if mask & !ALL_INOTIFY_BITS != 0: return Err(EINVAL).
2. if mask & ALL_INOTIFY_BITS == 0: return Err(EINVAL).
3. f = FdRef::from(fd)?; if f.is_empty(): return Err(EBADF).
4. if (mask & IN_MASK_ADD) != 0 && (mask & IN_MASK_CREATE) != 0: return Err(EINVAL).
5. if f.f_op != &INOTIFY_FOPS: return Err(EINVAL).
6. lookup_flags = 0; if !(mask & IN_DONT_FOLLOW): lookup_flags |= LOOKUP_FOLLOW.
7. if mask & IN_ONLYDIR: lookup_flags |= LOOKUP_DIRECTORY.
8. path = Inotify::find_inode(pathname, lookup_flags, mask & IN_ALL_EVENTS)?;
9. inode = path.dentry.d_inode; group = f.private_data::<FsnotifyGroup>().
10. ret = Inotify::update_watch(group, inode, mask).
11. path_put(path); return ret.

`Inotify::update_watch(group, inode, arg) -> Result<i32>`:
1. fsnotify_group_lock(group).
2. ret = Inotify::update_existing_watch(group, inode, arg).
3. if ret == Err(ENOENT): ret = Inotify::new_watch(group, inode, arg).
4. fsnotify_group_unlock(group).
5. return ret.

`Inotify::new_watch(group, inode, arg) -> Result<i32>`:
1. tmp = kmem_cache_alloc(INODE_MARK_CACHE, GFP_KERNEL)?; — InotifyInodeMark.
2. fsnotify_init_mark(&tmp.fsn_mark, group);
3. tmp.fsn_mark.mask = Inotify::arg_to_mask(inode, arg);
4. tmp.fsn_mark.flags = Inotify::arg_to_flags(arg);
5. tmp.wd = -1.
6. ret = Inotify::add_to_idr(&group.inotify_data.idr, &group.inotify_data.idr_lock, tmp);
7. if ret.is_err(): goto out_err.
8. if !inc_inotify_watches(&group.inotify_data.ucounts): Inotify::remove_from_idr(group, tmp); ret = Err(ENOSPC); goto out_err.
9. ret = fsnotify_add_inode_mark_locked(&tmp.fsn_mark, inode, 0).
10. if ret.is_err(): Inotify::remove_from_idr(group, tmp); dec_inotify_watches; goto out_err.
11. ret = Ok(tmp.wd).
12. out_err: fsnotify_put_mark(&tmp.fsn_mark); return ret.

`Inotify::sys_rm_watch(fd, wd) -> Result<()>`:
1. f = FdRef::from(fd)?; if f.is_empty(): return Err(EBADF).
2. if f.f_op != &INOTIFY_FOPS: return Err(EINVAL).
3. group = f.private_data::<FsnotifyGroup>().
4. i_mark = Inotify::idr_find(group, wd); if i_mark.is_none(): return Err(EINVAL).
5. fsnotify_destroy_mark(&i_mark.fsn_mark, group).
6. fsnotify_put_mark(&i_mark.fsn_mark).
7. return Ok(()).

`Inotify::read(file, buf, count) -> Result<isize>`:
1. start = buf; group = file.private_data::<FsnotifyGroup>().
2. wait = DEFINE_WAIT_FUNC(woken_wake_function).
3. add_wait_queue(&group.notification_waitq, &wait).
4. loop:
   - spin_lock(&group.notification_lock).
   - kevent = Inotify::get_one_event(group, count).
   - spin_unlock(&group.notification_lock).
   - match kevent:
     - Some(Ok(ev)):
       - ret = Inotify::copy_event_to_user(group, ev, buf).
       - fsnotify_destroy_event(group, ev).
       - if ret < 0: break.
       - buf += ret; count -= ret; continue.
     - Some(Err(e)): ret = e; break.
     - None:
       - if file.f_flags & O_NONBLOCK: ret = -EAGAIN; break.
       - if signal_pending(current): ret = -ERESTARTSYS; break.
       - if start != buf: break.
       - wait_woken(&wait, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT).
5. remove_wait_queue(&group.notification_waitq, &wait).
6. if start != buf ∧ ret != -EFAULT: ret = buf - start.
7. return ret.

`Inotify::copy_event_to_user(group, fsn_event, buf) -> Result<usize>`:
1. event = INOTIFY_E(fsn_event); name_len = event.name_len.
2. pad_name_len = Inotify::round_event_name_len(fsn_event).
3. ue = InotifyEventUapi { wd: event.wd, mask: Inotify::mask_to_arg(event.mask), cookie: event.sync_cookie, len: pad_name_len }.
4. copy_to_user(buf, &ue, size_of::<InotifyEventUapi>())?;
5. buf += size_of::<InotifyEventUapi>().
6. if pad_name_len > 0:
   - copy_to_user(buf, event.name, name_len)?;
   - buf += name_len.
   - clear_user(buf, pad_name_len - name_len)?;
   - event_size += pad_name_len.
7. return Ok(size_of::<InotifyEventUapi>() + pad_name_len).

`Inotify::round_event_name_len(fsn_event) -> usize`:
1. ev = INOTIFY_E(fsn_event); if ev.name_len == 0: return 0.
2. return roundup(ev.name_len + 1, size_of::<InotifyEventUapi>()).

`Inotify::arg_to_mask(inode, arg) -> u32`:
1. mask = FS_UNMOUNT.
2. if S_ISDIR(inode.i_mode): mask |= FS_EVENT_ON_CHILD.
3. mask |= arg & INOTIFY_USER_MASK.
4. return mask.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `event_bit_equivalence` | INVARIANT | per-init: IN_* == FS_* for all 16 events (compile-time BUILD_BUG_ON). |
| `init_flags_validated` | INVARIANT | per-do_init: only IN_CLOEXEC | IN_NONBLOCK accepted. |
| `add_watch_mask_validated` | INVARIANT | per-add_watch: mask subset of ALL_INOTIFY_BITS ∧ mask != 0. |
| `add_watch_excl_xor_create` | INVARIANT | per-add_watch: IN_MASK_ADD ∧ IN_MASK_CREATE rejected. |
| `idr_refcount_consistent` | INVARIANT | per-add_to_idr: success ⟹ +1 mark ref (idr ref). |
| `remove_from_idr_balanced` | INVARIANT | per-remove_from_idr: -1 idr ref via fsnotify_put_mark; locked-find ref dropped. |
| `read_no_underrun` | INVARIANT | per-read: copy_event_to_user bytes ≤ count. |
| `round_name_len_aligned` | INVARIANT | per-emit: pad_name_len % sizeof(InotifyEventUapi) == 0. |
| `release_destroys_group` | INVARIANT | per-release: fsnotify_destroy_group called exactly once. |
| `q_overflow_event_singleton` | INVARIANT | per-group: overflow_event present at most once in queue. |

### Layer 2: TLA+

`fs/notify-inotify.tla`:
- Per-init-syscall / per-add-watch / per-event-arrival / per-read / per-rm-watch / per-close.
- Properties:
  - `safety_wd_unique_per_group` — per-group: each (inode) maps to ≤1 wd; each wd maps to ≤1 mark.
  - `safety_in_ignored_emitted_on_rm` — per-rm_watch: IN_IGNORED enqueued before mark dropped.
  - `safety_q_overflow_on_saturation` — per-fill: q_len ≥ max_events triggers IN_Q_OVERFLOW (wd = -1).
  - `safety_oneshot_self_destructs` — per-IN_ONESHOT: mark removed after first matching event.
  - `safety_excl_unlink_skips_unlinked` — per-IN_EXCL_UNLINK: events on d_unlinked dentry suppressed.
  - `liveness_event_eventually_readable` — per-event: poll/read returns EPOLLIN within bounded delay (queue non-empty).
  - `liveness_blocking_read_wakes` — per-event-arrival: blocked readers on notification_waitq wake.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Inotify::do_init` post: fd ≥ 0 ⟹ group attached to anon_inode ∧ ucounts incremented | `Inotify::do_init` |
| `Inotify::sys_add_watch` post: Ok(wd) ⟹ wd > 0 ∧ idr_find(wd) == mark | `Inotify::sys_add_watch` |
| `Inotify::sys_rm_watch` post: Ok ⟹ idr_find(wd) == NULL ∧ IN_IGNORED enqueued | `Inotify::sys_rm_watch` |
| `Inotify::update_existing_watch` post: !IN_MASK_ADD ⟹ replaces mask; IN_MASK_ADD ⟹ ORs | `Inotify::update_existing_watch` |
| `Inotify::new_watch` post: Ok ⟹ idr ref + ucount + connector attachment all held | `Inotify::new_watch` |
| `Inotify::read` post: Ok(n) ⟹ n is sum of full inotify_event records emitted | `Inotify::read` |
| `Inotify::copy_event_to_user` post: returns size_of<InotifyEventUapi>() + pad_name_len | `Inotify::copy_event_to_user` |
| `Inotify::arg_to_mask` post: result superset of (arg & INOTIFY_USER_MASK) | `Inotify::arg_to_mask` |
| `Inotify::mask_to_arg` post: emit subset of (IN_ALL_EVENTS | IN_ISDIR | IN_UNMOUNT | IN_IGNORED | IN_Q_OVERFLOW) | `Inotify::mask_to_arg` |

### Layer 4: Verus/Creusot functional

`Per-inotify_init1 → anon_inode_getfd + fsnotify_alloc_group → per-inotify_add_watch (path lookup + LSM + idr + connector) → per-VFS hook → fsnotify() → inotify_handle_inode_event → fsnotify_insert_event → per-inotify_read → copy_event_to_user (wd / mask / cookie / len / name)` semantic equivalence: per-inotify(7) man page + per-Documentation/filesystems/inotify.rst.

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

Inotify-user reinforcement:

- **Per-ucount UCOUNT_INOTIFY_INSTANCES + _WATCHES** — defense against per-uid DoS via watch flooding.
- **Per-max_queued_events bound (=16384 default)** — defense against per-fd queue exhaustion.
- **Per-IN_Q_OVERFLOW singleton** — defense against per-overflow storm.
- **Per-MAY_READ permission check + security_path_notify LSM** — defense against per-watch-without-perm info-leak.
- **Per-IN_CLOEXEC/IN_NONBLOCK == O_CLOEXEC/O_NONBLOCK assertion** — defense against per-libc flag-mismatch.
- **Per-IN_MASK_ADD xor IN_MASK_CREATE rejection** — defense against per-ambiguous-update semantics.
- **Per-IN_DONT_FOLLOW honored in user_path_at** — defense against per-symlink-target inotify spoof.
- **Per-IN_EXCL_UNLINK skip on d_unlinked** — defense against per-stale-watch on deleted file.
- **Per-IN_ONESHOT auto-remove** — defense against per-fd-leak on one-shot watchers.
- **Per-idr_alloc_cyclic** — defense against per-wd reuse soon after rm_watch (collision with stale userland refs).
- **Per-fsnotify_group_lock around update_existing/new_watch** — defense against per-concurrent add race.
- **Per-INOTIFY_WATCH_COST clamp [8192, 1048576]** — defense against per-totalram OOM-class watch budgeting.
- **Per-anon_inode O_RDONLY only** — defense against per-write-on-inotify-fd misuse.
- **Per-INOTIFY_IOC_SETNEXTWD gated on CONFIG_CHECKPOINT_RESTORE** — defense against per-CRIU-only path attack surface.
- **Per-FIONREAD walks queue under notification_lock** — defense against per-FIONREAD-vs-read race.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on `inotify_read` `copy_event_to_user` so a crafted name-length on an oversize watch path cannot overrun a user buffer.
- **PAX_KERNEXEC** — W^X for any executable mapping reachable from inotify dispatch.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization across `inotify_init1` / `inotify_add_watch` / `inotify_rm_watch` so timing channels do not leak stack layout.
- **PAX_REFCOUNT** — saturating refcount on `struct fsnotify_group`, on the per-mark `inotify_inode_mark`, and on `wd → mark` IDR mappings.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `inotify_event_info` slab so leftover name/path bytes do not survive into the next event.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access on read path and on `INOTIFY_IOC_SETNEXTWD`.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `inotify_fsnotify_ops` (`handle_inode_event`, `free_event`, `freeing_mark`).
- **GRKERNSEC_HIDESYM** — kernel pointer hiding in `/proc/<pid>/fdinfo/N` for inotify fds (`inotify wd:`).
- **GRKERNSEC_DMESG** — syslog restriction on overflow / mark-leak diagnostics.
- **`max_user_watches` per-uid quota** — `inotify_new_watch` charges a per-uid counter via `inc_inotify_watches`; quota is enforced regardless of user-namespace traversal so a user inside a container cannot exceed the host policy.
- **`max_user_instances` per-uid quota** — `inotify_init1` charges `inc_inotify_instances` so fork-bomb-style instance creation cannot exhaust group slab.
- **`IN_DONT_FOLLOW`** — symlink targets are not auto-resolved when `IN_DONT_FOLLOW` is set; this is required for trusted-path inotify because the path component might be attacker-writable.
- **`IN_ONESHOT` auto-remove** — single-shot watches drop the mark on first event so a watcher cannot leak wd entries.
- **`idr_alloc_cyclic`** — wd numbers are not reused soon after `rm_watch`, eliminating collisions with stale userland wd references.
- **`anon_inode` O_RDONLY only** — inotify fd cannot be `write()`-targeted by mistake; write attempts fail with `-EBADF`.
- **`INOTIFY_IOC_SETNEXTWD` `CONFIG_CHECKPOINT_RESTORE` gate** — wd-number injection is restricted to CRIU builds; non-CRIU production kernels reject this ioctl outright.

Per-doc rationale: inotify is an unprivileged user-namespace-traversable subsystem with per-uid watch quotas as its only resource bound; PaX/grsec reinforcement keeps watch and instance counters under saturating arithmetic, keeps wd allocation collision-resistant, keeps event copy-out USERCOPY-bounded, and keeps the watch-symlink-following semantics under explicit `IN_DONT_FOLLOW` opt-in so trusted-path monitoring is actually trusted.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/notify/inotify/inotify_fsnotify.c handle_inode_event + free_event + freeing_mark (covered as Tier-3 sibling if expanded)
- fs/notify/fsnotify.c dispatch core (covered in `notify-core.md`)
- fs/notify/notification.c queue (covered in `notify-core.md`)
- fs/notify/mark.c mark/connector lifecycle (covered separately if expanded)
- fs/notify/fanotify/* user API (covered in `notify-fanotify.md` if expanded)
- fs/notify/dnotify/dnotify.c (legacy; covered separately if expanded)
- /proc/PID/fdinfo dumper (`inotify_show_fdinfo`; covered in proc Tier-3 if expanded)
- Implementation code
