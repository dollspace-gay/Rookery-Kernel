# Tier-5 UAPI: include/uapi/linux/inotify.h — inotify(7) ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/inotify.h (~84 lines)
  - fs/notify/inotify/inotify_user.c
  - fs/notify/inotify/inotify_fsnotify.c
  - fs/notify/fsnotify.c
  - include/linux/inotify.h
-->

## Summary

`inotify(7)` is the inode-based directory-notification subsystem that exposes per-watch event streams to userspace via an `inotify_init`/`inotify_init1` file descriptor. Per-watch state lives in `struct inotify_watch` (kernel-side) keyed by a per-instance `wd` integer. Per-event userspace receives a packed `struct inotify_event { __s32 wd; __u32 mask; __u32 cookie; __u32 len; char name[]; }`. Per-mask bits select events (`IN_ACCESS`, `IN_MODIFY`, `IN_ATTRIB`, `IN_CLOSE_WRITE`, `IN_CLOSE_NOWRITE`, `IN_OPEN`, `IN_MOVED_FROM`, `IN_MOVED_TO`, `IN_CREATE`, `IN_DELETE`, `IN_DELETE_SELF`, `IN_MOVE_SELF`), special always-sent bits (`IN_UNMOUNT`, `IN_Q_OVERFLOW`, `IN_IGNORED`), and watch-modifier bits (`IN_ONLYDIR`, `IN_DONT_FOLLOW`, `IN_EXCL_UNLINK`, `IN_MASK_CREATE`, `IN_MASK_ADD`, `IN_ISDIR`, `IN_ONESHOT`). Per-instance flags `IN_CLOEXEC` / `IN_NONBLOCK` are aliases for `O_CLOEXEC` / `O_NONBLOCK`. Critical for: file managers, `systemd.path` units, build-system rebuilders, antivirus path monitors, container image-layer change detection.

This Tier-5 covers `include/uapi/linux/inotify.h` (~84 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct inotify_event` | per-event userspace record | `uapi::inotify::Event` |
| `sys_inotify_init` | per-instance fd alloc | `Inotify::sys_init` |
| `sys_inotify_init1` | per-instance fd alloc + flags | `Inotify::sys_init1` |
| `sys_inotify_add_watch` | per-watch add / modify | `Inotify::sys_add_watch` |
| `sys_inotify_rm_watch` | per-watch remove | `Inotify::sys_rm_watch` |
| `INOTIFY_IOC_SETNEXTWD` | per-instance preset next wd | `Inotify::ioc_setnextwd` |
| `IN_ACCESS` (0x1) | per-event: read | shared |
| `IN_MODIFY` (0x2) | per-event: write | shared |
| `IN_ATTRIB` (0x4) | per-event: metadata change | shared |
| `IN_CLOSE_WRITE` (0x8) | per-event: writable closed | shared |
| `IN_CLOSE_NOWRITE` (0x10) | per-event: unwritable closed | shared |
| `IN_OPEN` (0x20) | per-event: open | shared |
| `IN_MOVED_FROM` (0x40) | per-event: rename src | shared |
| `IN_MOVED_TO` (0x80) | per-event: rename dst | shared |
| `IN_CREATE` (0x100) | per-event: subfile create | shared |
| `IN_DELETE` (0x200) | per-event: subfile unlink | shared |
| `IN_DELETE_SELF` (0x400) | per-event: watched object deleted | shared |
| `IN_MOVE_SELF` (0x800) | per-event: watched object moved | shared |
| `IN_UNMOUNT` (0x2000) | per-event: backing fs unmounted | shared |
| `IN_Q_OVERFLOW` (0x4000) | per-event: per-fd queue overflowed | shared |
| `IN_IGNORED` (0x8000) | per-event: watch invalidated | shared |
| `IN_ONLYDIR` (0x01000000) | per-watch: dir-only | shared |
| `IN_DONT_FOLLOW` (0x02000000) | per-watch: no symlink follow | shared |
| `IN_EXCL_UNLINK` (0x04000000) | per-watch: drop events post-unlink | shared |
| `IN_MASK_CREATE` (0x10000000) | per-watch: fail if exists | shared |
| `IN_MASK_ADD` (0x20000000) | per-watch: OR into existing mask | shared |
| `IN_ISDIR` (0x40000000) | per-event: subject is dir | shared |
| `IN_ONESHOT` (0x80000000) | per-watch: auto-remove after first event | shared |
| `IN_CLOSE` | IN_CLOSE_WRITE \| IN_CLOSE_NOWRITE | shared |
| `IN_MOVE` | IN_MOVED_FROM \| IN_MOVED_TO | shared |
| `IN_ALL_EVENTS` | union of legal events | shared |
| `IN_CLOEXEC` | = O_CLOEXEC | shared |
| `IN_NONBLOCK` | = O_NONBLOCK | shared |

## ABI surface (constants + structs)

### Struct `inotify_event`

```
struct inotify_event {
    __s32  wd;       /* watch descriptor (from inotify_add_watch) */
    __u32  mask;     /* bitmask of IN_* events that triggered */
    __u32  cookie;   /* per-rename pair correlation (IN_MOVED_FROM / IN_MOVED_TO) */
    __u32  len;      /* length of name[] including NUL pad; 0 if no name */
    char   name[];   /* NUL-terminated, NUL-padded to natural alignment */
};
```

- Layout is packed in the byte stream returned by `read(2)` on the inotify fd; trailing `name[]` is padded to align the next record at a `__u32` boundary.
- `wd` is the value returned by `inotify_add_watch`; for `IN_Q_OVERFLOW` it is `-1`.
- `cookie` is 0 except for `IN_MOVED_FROM` / `IN_MOVED_TO` of a single rename pair, where both share a non-zero value.
- `len` is the count of bytes in `name[]` (already including the terminating NUL and trailing padding); 0 implies no name follows.

### Event bits (low 12)

| Bit | Constant | Direction | Semantics |
|---|---|---|---|
| 0x00000001 | `IN_ACCESS` | watch | file read |
| 0x00000002 | `IN_MODIFY` | watch | file write |
| 0x00000004 | `IN_ATTRIB` | watch | metadata (chmod, chown, link count, timestamps, xattrs) |
| 0x00000008 | `IN_CLOSE_WRITE` | watch | writable fd closed |
| 0x00000010 | `IN_CLOSE_NOWRITE` | watch | read-only / exec fd closed |
| 0x00000020 | `IN_OPEN` | watch | file opened |
| 0x00000040 | `IN_MOVED_FROM` | dir watch | child renamed away (paired with `IN_MOVED_TO` via cookie) |
| 0x00000080 | `IN_MOVED_TO` | dir watch | child renamed in |
| 0x00000100 | `IN_CREATE` | dir watch | child created |
| 0x00000200 | `IN_DELETE` | dir watch | child unlinked |
| 0x00000400 | `IN_DELETE_SELF` | watch | watched object itself unlinked |
| 0x00000800 | `IN_MOVE_SELF` | watch | watched object itself renamed |

### Always-on event bits (sent even without explicit subscription)

| Bit | Constant | Semantics |
|---|---|---|
| 0x00002000 | `IN_UNMOUNT` | backing filesystem unmounted; watch is implicitly removed |
| 0x00004000 | `IN_Q_OVERFLOW` | per-fd queue overflowed (events lost) |
| 0x00008000 | `IN_IGNORED` | watch was removed (explicit `inotify_rm_watch`, IN_ONESHOT auto-removal, or implicit on unlink/unmount) |

### Watch-modifier bits (used in `mask` arg to `inotify_add_watch`)

| Bit | Constant | Semantics |
|---|---|---|
| 0x01000000 | `IN_ONLYDIR` | fail with `ENOTDIR` if pathname is not a directory |
| 0x02000000 | `IN_DONT_FOLLOW` | do not dereference a symlink at the leaf |
| 0x04000000 | `IN_EXCL_UNLINK` | suppress events on a child after it is unlinked |
| 0x10000000 | `IN_MASK_CREATE` | only create new watch; fail with `EEXIST` if one already exists |
| 0x20000000 | `IN_MASK_ADD` | OR the new mask into the existing watch's mask (instead of replace) |
| 0x40000000 | `IN_ISDIR` | (read-back only) the subject of the event is a directory |
| 0x80000000 | `IN_ONESHOT` | remove the watch after the first matching event |

`IN_MASK_CREATE` is mutually exclusive with `IN_MASK_ADD` (per `inotify_user.c::inotify_update_watch`).

### Composite helpers

```
#define IN_CLOSE      (IN_CLOSE_WRITE | IN_CLOSE_NOWRITE)
#define IN_MOVE       (IN_MOVED_FROM  | IN_MOVED_TO)
#define IN_ALL_EVENTS (IN_ACCESS | IN_MODIFY | IN_ATTRIB | IN_CLOSE_WRITE | \
                       IN_CLOSE_NOWRITE | IN_OPEN | IN_MOVED_FROM | \
                       IN_MOVED_TO | IN_DELETE | IN_CREATE | IN_DELETE_SELF | \
                       IN_MOVE_SELF)
```

### `inotify_init1` flags

| Bit | Constant | Notes |
|---|---|---|
| `O_CLOEXEC` | `IN_CLOEXEC` | fd closed on `execve(2)` |
| `O_NONBLOCK` | `IN_NONBLOCK` | `read(2)` returns `EAGAIN` instead of blocking |

### ioctl

```
#define INOTIFY_IOC_SETNEXTWD  _IOW('I', 0, __s32)
```

Pre-seeds the next watch-descriptor allocator value (requires CAP_SYS_ADMIN; primarily for criu-style checkpoint/restore).

## Compatibility contract

REQ-1: `sys_inotify_init(void)` — returns a new inotify fd (no flags), error `EMFILE` / `ENFILE` / `ENOMEM`.

REQ-2: `sys_inotify_init1(int flags)`:
- /* Validate flags */
- if flags & ~(IN_CLOEXEC | IN_NONBLOCK): return -EINVAL.
- /* Allocate anon-inode fd */
- inode = anon_inode_getfile("inotify", &inotify_fops, group, flags & (O_CLOEXEC | O_NONBLOCK)).
- /* Per-`group`: per-instance event queue + watch index */
- group = inotify_new_group(inotify_max_queued_events).
- return fd.

REQ-3: `sys_inotify_add_watch(int fd, const char __user *pathname, __u32 mask)`:
- /* Validate mask: at least one event bit must be set */
- if !(mask & IN_ALL_EVENTS) ∧ !(mask & (IN_DELETE_SELF | IN_MOVE_SELF | IN_UNMOUNT)): return -EINVAL.
- /* Mutually exclusive */
- if (mask & (IN_MASK_ADD | IN_MASK_CREATE)) == (IN_MASK_ADD | IN_MASK_CREATE): return -EINVAL.
- /* Resolve path */
- flags = LOOKUP_FOLLOW.
- if mask & IN_DONT_FOLLOW: flags &= ~LOOKUP_FOLLOW.
- if mask & IN_ONLYDIR: flags |= LOOKUP_DIRECTORY.
- error = user_path_at(AT_FDCWD, pathname, flags, &path); if error: return error.
- /* Find or create watch */
- inode = path.dentry.d_inode.
- mark = fsnotify_find_mark(inode.i_fsnotify_marks, group).
- if mark ∧ (mask & IN_MASK_CREATE): return -EEXIST.
- if mark ∧ (mask & IN_MASK_ADD): new_mask = mark.mask | mask.
- else: new_mask = mask.
- /* Allocate / update */
- if !mark: mark = inotify_new_watch(group, inode, new_mask); wd = mark.wd.
- else: mark.mask = new_mask; wd = mark.wd.
- return wd.

REQ-4: `sys_inotify_rm_watch(int fd, __s32 wd)`:
- /* Validate */
- group = file.private_data of fd.
- mark = idr_find(&group.idr, wd).
- if !mark: return -EINVAL.
- /* Generate IN_IGNORED, then destroy */
- fsnotify_destroy_mark(mark, group).
- return 0.

REQ-5: Per-`read(2)` on inotify fd:
- Returns one or more concatenated `struct inotify_event` records (each followed by `len` bytes of name).
- Per-non-blocking: returns `-EAGAIN` if queue empty.
- Per-blocking: sleeps on the per-group wait queue.
- Per-buffer-too-small: returns `-EINVAL` if buffer cannot hold the first pending event.
- Per-signal interrupt: returns `-EINTR`.

REQ-6: Per-event ordering:
- Events are delivered in FIFO order per-group.
- Coalescing: identical consecutive events (same wd, mask, cookie, name) MAY be merged into one record (per `inotify_merge`); this is observable behavior.

REQ-7: Per-overflow:
- When the per-group queue reaches `inotify_max_queued_events` (default 16384, `/proc/sys/fs/inotify/max_queued_events`), a single `IN_Q_OVERFLOW` event is appended and further events are dropped until the queue drains.

REQ-8: Per-instance-and-watch limits:
- `/proc/sys/fs/inotify/max_user_instances` (default 128) — per-real-UID instance cap; exceed ⟹ `inotify_init*` returns `EMFILE`.
- `/proc/sys/fs/inotify/max_user_watches` (default varies by RAM) — per-real-UID watch cap; exceed ⟹ `inotify_add_watch` returns `ENOSPC`.

REQ-9: Per-`IN_ONESHOT`:
- After the first event matches the watch, an `IN_IGNORED` is emitted and the watch is destroyed atomically.

REQ-10: Per-`IN_UNMOUNT`:
- On filesystem unmount, every watch on that fs is invalidated: emit `IN_UNMOUNT` then `IN_IGNORED`.

REQ-11: Per-`IN_MOVED_FROM` / `IN_MOVED_TO` cookie:
- Cookie is a per-event, non-zero unique value shared by the matching pair within the same group.
- If the rename crosses a watch boundary, only the half on the watched side is delivered.

REQ-12: Per-`IN_EXCL_UNLINK`:
- After a child is unlinked (still reachable via open fd), suppress subsequent events on that child for this watch.

REQ-13: Per-`IN_ISDIR` read-back:
- Set in the delivered event's `mask` if the event subject is a directory; never accepted as a request bit.

REQ-14: Per-`INOTIFY_IOC_SETNEXTWD`:
- arg ∈ [1, INT_MAX]: pre-set the next wd allocator value.
- arg ≤ 0: returns `-EINVAL`.
- Caller must hold CAP_SYS_ADMIN.

REQ-15: Per-close(fd):
- All watches on the group are destroyed; in-flight readers wake with `-EBADF`.
- Group destruction is RCU-safe; no event leaks into another instance.

REQ-16: Per-poll/epoll:
- `POLLIN` set when at least one event is pending or `IN_Q_OVERFLOW` is queued.
- `POLLERR` never set.

## Acceptance Criteria

- [ ] AC-1: `inotify_init1(IN_CLOEXEC | IN_NONBLOCK)` returns a non-blocking close-on-exec fd.
- [ ] AC-2: `inotify_init1(0x100)` (invalid flag) returns `-EINVAL`.
- [ ] AC-3: `inotify_add_watch(fd, "/", IN_MODIFY)` returns positive wd ≥ 1.
- [ ] AC-4: `inotify_add_watch(fd, path, 0)` returns `-EINVAL`.
- [ ] AC-5: `inotify_add_watch(fd, path, IN_MASK_ADD | IN_MASK_CREATE | IN_MODIFY)` returns `-EINVAL`.
- [ ] AC-6: `inotify_add_watch(fd, path, IN_ONLYDIR | IN_MODIFY)` on a regular file returns `-ENOTDIR`.
- [ ] AC-7: `inotify_add_watch(fd, path, IN_MASK_CREATE | IN_MODIFY)` on an already-watched path returns `-EEXIST`.
- [ ] AC-8: Rename within a watched dir emits paired `IN_MOVED_FROM` / `IN_MOVED_TO` with matching non-zero cookie.
- [ ] AC-9: `IN_ONESHOT` watch fires exactly once and is followed by `IN_IGNORED`.
- [ ] AC-10: Queue overflow yields exactly one `IN_Q_OVERFLOW` (wd == -1) and no further events until drained.
- [ ] AC-11: Exceeding `max_user_instances` returns `-EMFILE`; exceeding `max_user_watches` returns `-ENOSPC`.
- [ ] AC-12: `inotify_rm_watch(fd, bogus_wd)` returns `-EINVAL`.
- [ ] AC-13: After `inotify_rm_watch`, exactly one `IN_IGNORED` is queued for that wd.
- [ ] AC-14: Unmounting backing fs emits `IN_UNMOUNT` then `IN_IGNORED` per watch.
- [ ] AC-15: `IN_EXCL_UNLINK` suppresses events on unlinked-still-open children.
- [ ] AC-16: `read(fd, buf, sizeof(struct inotify_event) - 1)` returns `-EINVAL`.
- [ ] AC-17: `read(fd, ...)` on empty queue with `O_NONBLOCK` returns `-EAGAIN`.
- [ ] AC-18: `INOTIFY_IOC_SETNEXTWD` with arg ≤ 0 returns `-EINVAL`; unprivileged caller returns `-EPERM`.
- [ ] AC-19: `close(fd)` releases all watches and accounts back to `max_user_watches`.
- [ ] AC-20: `IN_ISDIR` is set in event mask for any directory-subject event.

## Architecture

```
pub struct InotifyEvent {
    pub wd:     i32,
    pub mask:   u32,
    pub cookie: u32,
    pub len:    u32,
    /* name follows in-place, NUL-padded to align next record */
}

pub struct InotifyGroup {
    pub queue:      EventQueue,            // FIFO with overflow flag
    pub watches:    IdrMap<i32, Arc<InotifyWatch>>,
    pub max_events: u32,
    pub user:       Arc<UserStruct>,       // per-real-UID accounting
    pub flags:      InotifyInitFlags,      // IN_CLOEXEC | IN_NONBLOCK
    pub next_wd:    AtomicI32,
}

pub struct InotifyWatch {
    pub wd:    i32,
    pub mask:  u32,                        // event + modifier bits
    pub inode: Arc<Inode>,
    pub group: Weak<InotifyGroup>,
}
```

`Inotify::sys_init1(flags) -> Result<Fd>`:
1. if flags & !(IN_CLOEXEC | IN_NONBLOCK) != 0: return Err(EINVAL).
2. user = current_user().
3. if user.inotify_instances >= sysctl_max_user_instances: return Err(EMFILE).
4. group = InotifyGroup::new(sysctl_max_queued_events, flags, user.clone()).
5. user.inotify_instances += 1.
6. fd = anon_inode_fd("inotify", &INOTIFY_FOPS, group, flags).
7. return Ok(fd).

`Inotify::sys_add_watch(fd, path_user, mask) -> Result<i32>`:
1. /* Validate mask */
2. event_bits = mask & (IN_ALL_EVENTS | IN_UNMOUNT | IN_Q_OVERFLOW | IN_IGNORED).
3. if event_bits == 0: return Err(EINVAL).
4. if (mask & IN_MASK_CREATE) && (mask & IN_MASK_ADD): return Err(EINVAL).
5. /* Resolve */
6. lookup_flags = if mask & IN_DONT_FOLLOW { 0 } else { LOOKUP_FOLLOW }.
7. if mask & IN_ONLYDIR: lookup_flags |= LOOKUP_DIRECTORY.
8. path = user_path_at(AT_FDCWD, path_user, lookup_flags)?.
9. group = fd_to_group(fd)?.
10. inode = path.dentry.inode().
11. /* Find existing */
12. existing = group.find_watch(inode).
13. match existing {
        Some(w) if mask & IN_MASK_CREATE != 0 => return Err(EEXIST),
        Some(w) if mask & IN_MASK_ADD    != 0 => { w.mask |= mask & ~IN_MASK_ADD; return Ok(w.wd); }
        Some(w)                                => { w.mask  = mask;               return Ok(w.wd); }
        None => {}
    }
14. /* Allocate */
15. if user.inotify_watches >= sysctl_max_user_watches: return Err(ENOSPC).
16. wd = group.alloc_wd()?.
17. watch = Arc::new(InotifyWatch { wd, mask, inode: inode.clone(), group: Arc::downgrade(&group) }).
18. fsnotify_add_mark(inode, &watch).
19. group.watches.insert(wd, watch).
20. user.inotify_watches += 1.
21. return Ok(wd).

`Inotify::sys_rm_watch(fd, wd) -> Result<()>`:
1. group = fd_to_group(fd)?.
2. watch = group.watches.remove(wd).ok_or(EINVAL)?.
3. fsnotify_destroy_mark(&watch).
4. /* Emit IN_IGNORED to userspace */
5. group.queue.push(InotifyEvent { wd, mask: IN_IGNORED, cookie: 0, len: 0 }, name: "").
6. group.user.inotify_watches -= 1.
7. return Ok(()).

`Inotify::handle_event(watch, event_mask, cookie, name_opt)`:
1. if watch.mask & event_mask == 0: return.       /* not subscribed */
2. if (watch.mask & IN_EXCL_UNLINK) ∧ child_is_unlinked: return.
3. effective = event_mask & watch.mask.
4. if subject_is_dir: effective |= IN_ISDIR.
5. /* Coalesce with tail if identical */
6. if watch.group.queue.try_merge_tail(watch.wd, effective, cookie, &name_opt): return.
7. /* Enqueue */
8. if watch.group.queue.len() >= watch.group.max_events:
     - watch.group.queue.set_overflow();
     - return.
9. watch.group.queue.push(InotifyEvent { wd: watch.wd, mask: effective, cookie, len: padded_len(&name_opt) }, name_opt).
10. /* Per-IN_ONESHOT */
11. if watch.mask & IN_ONESHOT:
    - fsnotify_destroy_mark(watch).
    - watch.group.queue.push(InotifyEvent { wd: watch.wd, mask: IN_IGNORED, cookie: 0, len: 0 }, "").

`Inotify::file_read(file, buf, count) -> Result<usize>`:
1. group = file.private_data.
2. if group.queue.is_empty():
   - if file.flags & O_NONBLOCK: return Err(EAGAIN).
   - wait_event_interruptible(group.wait, !group.queue.is_empty() || group.queue.overflow()).
3. if buf.len() < group.queue.peek().total_len(): return Err(EINVAL).
4. /* Drain into buf */
5. n = 0.
6. while group.queue.peek().is_some_and(|e| n + e.total_len() <= buf.len()):
     - e = group.queue.pop().
     - copy_to_user(buf[n..], e.serialized_bytes())?.
     - n += e.total_len().
7. return Ok(n).

`Inotify::file_ioctl(file, cmd, arg) -> Result<i32>`:
1. match cmd {
     INOTIFY_IOC_SETNEXTWD => {
       if !capable(CAP_SYS_ADMIN): return Err(EPERM);
       let v: i32 = read_user(arg as *const i32)?;
       if v <= 0: return Err(EINVAL);
       group.next_wd.store(v, Ordering::SeqCst);
       Ok(0)
     },
     FIONREAD => write_user(arg, group.queue.byte_len()),
     _ => Err(ENOTTY),
   }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `init1_flag_validation` | INVARIANT | sys_init1 rejects any bit outside (IN_CLOEXEC | IN_NONBLOCK). |
| `mask_create_add_exclusive` | INVARIANT | sys_add_watch rejects IN_MASK_CREATE | IN_MASK_ADD. |
| `event_mask_nonzero_required` | INVARIANT | sys_add_watch rejects mask with no event bits. |
| `rm_watch_idempotent_invalid` | INVARIANT | sys_rm_watch on missing wd returns EINVAL exactly once. |
| `oneshot_emits_ignored` | INVARIANT | IN_ONESHOT fire ⟹ IN_IGNORED enqueued and watch removed atomically. |
| `overflow_single_marker` | INVARIANT | queue full ⟹ exactly one IN_Q_OVERFLOW pending. |
| `setnextwd_cap_check` | INVARIANT | INOTIFY_IOC_SETNEXTWD requires CAP_SYS_ADMIN. |
| `name_buffer_padded` | INVARIANT | serialized event len == sizeof(struct inotify_event) + padded_name_len. |

### Layer 2: TLA+

`uapi/inotify.tla`:
- Per-instance creation + per-watch lifecycle + per-event delivery + per-overflow + per-close.
- Properties:
  - `safety_wd_unique_per_group` — per-group: wd values are unique while alive.
  - `safety_overflow_single` — per-group: at most one IN_Q_OVERFLOW pending at a time.
  - `safety_ignored_after_destroy` — per-watch: rm_watch / unmount / oneshot ⟹ IN_IGNORED emitted exactly once.
  - `safety_cookie_pairs_match` — per-rename: IN_MOVED_FROM and IN_MOVED_TO share unique non-zero cookie.
  - `safety_user_accounting_balanced` — per-real-UID: inotify_instances and inotify_watches counters return to zero after close.
  - `liveness_pending_eventually_read` — per-pending event: a non-blocking reader eventually consumes or sees EAGAIN.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Inotify::sys_init1` post: flags subset of (IN_CLOEXEC | IN_NONBLOCK) | `Inotify::sys_init1` |
| `Inotify::sys_add_watch` post: returns wd ≥ 1 ∨ negative errno | `Inotify::sys_add_watch` |
| `Inotify::sys_rm_watch` post: watch removed ∧ IN_IGNORED enqueued | `Inotify::sys_rm_watch` |
| `Inotify::handle_event` post: enqueued event mask ⊆ watch.mask ∪ IN_ISDIR | `Inotify::handle_event` |
| `Inotify::file_read` post: returned bytes ≤ buf.len ∧ aligned to event records | `Inotify::file_read` |
| `Inotify::file_ioctl` post: SETNEXTWD requires CAP_SYS_ADMIN ∧ arg > 0 | `Inotify::file_ioctl` |

### Layer 4: Verus/Creusot functional

`Per-inotify_init1(flags) → per-inotify_add_watch(fd, path, mask) → per-fsnotify-event → per-read(fd) → per-inotify_rm_watch / close(fd)` semantic equivalence: per-`Documentation/filesystems/inotify.rst` + man-page `inotify(7)`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

inotify(7) reinforcement:

- **Per-flag-mask validation on inotify_init1** — defense against per-future-flag-smuggle.
- **Per-add_watch mask: at least one event bit required** — defense against per-empty-mask DoS.
- **Per-IN_MASK_CREATE / IN_MASK_ADD mutual-exclusion** — defense against per-policy-confusion.
- **Per-IN_ONLYDIR enforcement** — defense against per-symlink-leaf surprise.
- **Per-IN_DONT_FOLLOW honored** — defense against per-symlink-pivot.
- **Per-user inotify_instances cap (fs.inotify.max_user_instances)** — defense against per-fd-exhaustion DoS.
- **Per-user inotify_watches cap (fs.inotify.max_user_watches)** — defense against per-watch-exhaustion DoS.
- **Per-group max_queued_events cap (fs.inotify.max_queued_events)** — defense against per-queue memory amplification.
- **Per-IN_Q_OVERFLOW single-emit** — defense against per-overflow flood.
- **Per-IN_EXCL_UNLINK honored** — defense against per-unlinked-still-open event leak.
- **Per-fd close-on-exec opt-in via IN_CLOEXEC** — defense against per-execve fd inheritance leak (mandatory in new code; recommended by `inotify(7)`).
- **Per-INOTIFY_IOC_SETNEXTWD CAP_SYS_ADMIN-gated** — defense against per-criu-only-primitive abuse.
- **Per-coalesce identical events** — defense against per-redundant-event flood.
- **Per-rcu-free of group / watches** — defense against per-UAF on concurrent close.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF / USERCOPY**: enforce strict checked copy at the `pathname` user pointer in `sys_inotify_add_watch` and at the `__user *buf` of `read(fd, ...)` (and the `__s32 *arg` of `INOTIFY_IOC_SETNEXTWD`). All UAPI surface ingress is `copy_from_user`/`copy_to_user` only — no raw deref of `__user` pointers.
- **PAX_RANDKSTACK**: per-syscall entry of `inotify_init` / `inotify_init1` / `inotify_add_watch` / `inotify_rm_watch` randomizes the kernel stack offset; defends against precise stack-spray to leak `struct inotify_group` pointers.
- **GRKERNSEC_PROC restriction on /proc/<pid>/fdinfo/<N>**: the inotify `fdinfo` exposes per-instance lines `inotify wd:<N> ino:<...> sdev:<...> mask:<...> ignored_mask:<...> fhandle-bytes:<...> fhandle-type:<...> f_handle:<...>` — under `GRKERNSEC_PROC_USER` / `_USERGROUP`, this leaks watched-inode identifiers and `file_handle` bytes that suffice to call `open_by_handle_at(2)`. Restrict `fdinfo` of inotify fds to the owning UID (or CAP_SYS_PTRACE), matching the existing fdinfo gate.
- **GRKERNSEC_HARDEN_PTRACE**: inotify metadata (wd → inode mapping) is reachable through `ptrace`-attached debuggers reading `/proc/<pid>/fd/<N>` of an inotify fd; harden so non-dumpable / mismatched-UID processes refuse `ptrace_may_access`, denying out-of-band wd enumeration.
- **inotify IN_ATTRIB on chattr +i / +a protected files**: PaX UDEREF still validates the user copy for `pathname`; in addition, watches on inodes marked `S_IMMUTABLE`/`S_APPEND` deliver `IN_ATTRIB` only after the same `CAP_LINUX_IMMUTABLE` gate the inode flag itself requires — i.e., do not allow inotify to side-channel mutation attempts on immutable inodes to unprivileged listeners with `R_OK` on the path. Audit-log every `IN_ATTRIB` on `S_IMMUTABLE` inodes (gr_log_str_int).
- **Per-real-UID accounting strictly enforced**: `fs.inotify.max_user_instances` and `fs.inotify.max_user_watches` are bounded per the real UID of the opener (not the effective UID); grsec hardens against setuid escalation that would otherwise lift the limit.
- **Per-fd close-on-exec mandatory** for code paths grsec-rebuilt with `GRKERNSEC_HARDEN_EXEC` — `inotify_init` without `IN_CLOEXEC` is rejected by a security_inotify_init LSM hook in hardened builds, forcing callers to use `inotify_init1(IN_CLOEXEC)`.
- **INOTIFY_IOC_SETNEXTWD as CAP_SYS_ADMIN-only** — grsec further restricts via `GRKERNSEC_CHROOT_CAPS` so a chrooted CAP_SYS_ADMIN cannot pre-seed the wd allocator of a host-side checkpoint tool.
- **fdinfo emission rate-limited** — defense against per-fdinfo-spam side-channel.
- **`name[]` copy_to_user padded with explicit zero** — defense against per-stack-slot infoleak (no uninitialized padding bytes ever reach userspace).

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- fanotify (covered in `uapi/headers/fanotify.md` Tier-5)
- dnotify legacy fcntl(F_NOTIFY) (deprecated; not re-implemented)
- fsnotify common backbone (covered in `fs/notify.md` Tier-3)
- LSM hooks `security_inotify_*` (covered in `security/lsm.md` Tier-3)
- man-page `inotify(7)` prose (upstream copy)
- Implementation code
