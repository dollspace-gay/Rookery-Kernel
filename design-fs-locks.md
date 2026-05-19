---
title: "Tier-3: fs/locks.c — POSIX/BSD/OFD/Lease file locking"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The kernel exposes four flavors of file locking through one core engine in `fs/locks.c`: **BSD locks** (`flock(2)`), **POSIX locks** (`fcntl(F_SETLK/F_SETLKW/F_GETLK)`), **Open File Description (OFD) locks** (`fcntl(F_OFD_SETLK/_SETLKW/_GETLK)`, owned by `struct file` rather than `files_struct`), and **leases** / **delegations** (`fcntl(F_SETLEASE)`, NFS/FSCACHE delegation). Per-inode lock state lives in `struct file_lock_context` (three lists: `flc_flock`, `flc_posix`, `flc_lease`, protected by `flc_lock`). Per-lock state is `struct file_lock` / `struct file_lease` wrapping a common `struct file_lock_core` (owner, type, flags, blocker-tree links). Conflict detection is by-range overlap and lock-type matrix (`F_RDLCK` vs `F_WRLCK`); blocked waiters form a tree rooted at the applied lock, and a per-CPU `file_lock_list` plus a global `blocked_hash` (hashed by `flc_owner`) support `/proc/locks` and `posix_locks_deadlock()` cycle detection. Lease break (`__break_lease`) sends `SIGIO`, optionally downgrades or unlocks, and times out after `lease_break_time` (default 45s).

This Tier-3 covers `fs/locks.c` (~3097 lines).

### Acceptance Criteria

- [ ] AC-1: posix_lock_file: overlapping F_WRLCK by different owner: returns -EAGAIN (or FILE_LOCK_DEFERRED if FL_SLEEP).
- [ ] AC-2: posix_lock_file: same owner, overlapping ranges, same type: merges to single record.
- [ ] AC-3: posix_lock_file: F_UNLCK splits an existing range into two if it carves out the middle.
- [ ] AC-4: posix_test_lock (F_GETLK): returns first conflicting lock; F_UNLCK if none.
- [ ] AC-5: posix_locks_deadlock: A→B→A cycle returns true (deadlock detected, request fails -EDEADLK).
- [ ] AC-6: posix_locks_deadlock: FL_OFDLCK request always returns false (no per-task owner).
- [ ] AC-7: flock_lock_inode: same filp re-locks change type without conflict; different filp F_WRLCK blocks/waits.
- [ ] AC-8: __break_lease: read-only opener (LEASE_BREAK_OPEN_RDONLY) downgrades F_WRLCK lease to F_RDLCK; LEASE_BREAK_LEASE+write force-unlocks.
- [ ] AC-9: __break_lease: lease_break_time expiry: time_out_leases removes lease without owner consent.
- [ ] AC-10: fcntl_setlk(F_OFD_SETLK) with l_pid != 0: returns -EINVAL.
- [ ] AC-11: fcntl_setlk: close-race detection: lookup_fd diverges → locks_remove_posix + -EBADF.
- [ ] AC-12: locks_remove_file (final fput): removes all FLOCK, POSIX, lease records belonging to file.
- [ ] AC-13: /proc/locks: enumerates all locks via per-CPU file_lock_list under file_rwsem.
- [ ] AC-14: sys_flock(LOCK_MAND): warn-once and return 0 (legacy compat).
- [ ] AC-15: posix_lock_inode_wait: signal during wait returns -ERESTARTSYS; lock not applied; locks_delete_block removes waiter.

### Architecture

```
struct FileLockCore {
  flc_list: HlistNode,                       // per-inode flc_flock/_posix/_lease
  flc_link: HlistNode,                       // per-CPU file_lock_list + blocked_hash
  flc_blocked_requests: ListHead,            // waiters tree
  flc_blocked_member: ListNode,              // on blocker's flc_blocked_requests
  flc_blocker: Option<*FileLockCore>,        // NULL = unblocked
  flc_wait: WaitQueueHead,
  flc_file: *File,
  flc_owner: FlOwnerT,                       // files_struct* | filp | NULL
  flc_pid: i32,
  flc_link_cpu: u32,
  flc_type: u8,                              // F_RDLCK / F_WRLCK / F_UNLCK
  flc_flags: u32,                            // FL_*
}

struct FileLock {
  c: FileLockCore,
  fl_start: u64,
  fl_end: u64,                               // OFFSET_MAX = whole file
  fl_ops: Option<&FileLockOps>,
  fl_lmops: Option<&LockManagerOps>,
}

struct FileLease {
  c: FileLockCore,
  fl_break_time: u64,                        // jiffies
  fl_downgrade_time: u64,                    // jiffies
  fl_fasync: *FasyncStruct,
  fl_lmops: &LockManagerOps,
}

struct FileLockContext {
  flc_lock: SpinLock<()>,
  flc_flock: ListHead<FileLockCore>,
  flc_posix: ListHead<FileLockCore>,
  flc_lease: ListHead<FileLockCore>,
}

struct FileLockListStruct {                  // per-CPU
  lock: SpinLock<()>,
  hlist: HlistHead,
}
```

`Locks::posix_lock_inode(inode, request, conflock) -> Result<()>`:
1. ctx = locks_get_lock_context(inode, request.c.flc_type)?;
2. /* Preallocate split halves */
3. (new_fl, new_fl2) = if !FL_ACCESS ∧ (type != F_UNLCK ∨ partial range): (Some(alloc), Some(alloc)) else (None, None).
4. percpu_down_read(&file_rwsem); spin_lock(&ctx.flc_lock).
5. /* Conflict scan */
6. if type != F_UNLCK:
   - for fl in ctx.flc_posix:
     - if !posix_locks_conflict(request, fl): continue.
     - /* NFS module-callback expiration */
     - if fl.fl_lmops?.lm_lock_expirable?(fl): module_get; drop locks; lm_expire_lock(); module_put; retry.
     - if conflock.is_some(): locks_copy_conflock(conflock, fl).
     - if !FL_SLEEP: return -EAGAIN.
     - spin_lock(&blocked_lock_lock).
     - __locks_wake_up_blocks(&request.c).
     - if !posix_locks_deadlock(request, fl):
       - __locks_insert_block(&fl.c, &request.c, posix_locks_conflict).
       - error = FILE_LOCK_DEFERRED.
     - else: error = -EDEADLK.
     - spin_unlock(&blocked_lock_lock); return error.
7. /* Apply: find first same-owner lock */
8. for fl in ctx.flc_posix: if posix_same_owner(request, fl): break.
9. /* Merge / split same-owner range */
10. for (fl, tmp) in list_for_each_entry_safe_from(...):
    - if !posix_same_owner: break.
    - if same type ∧ adjacent/overlap: merge endpoints into request; if added: delete duplicate; else added=true.
    - if different type: split (uses new_fl, new_fl2 for left/right halves).
11. /* Insert new if not merged */
12. if !added ∧ !F_UNLCK: locks_copy_lock(new_fl, request); locks_insert_lock_ctx(&new_fl.c, &fl.c.flc_list).
13. /* Wake waiters on split children */
14. if right: right.fl_start = request.fl_end+1; locks_wake_up_blocks(&right.c).
15. if left: left.fl_end = request.fl_start-1; locks_wake_up_blocks(&left.c).
16. spin_unlock(&ctx.flc_lock); percpu_up_read(&file_rwsem).
17. free unused new_fl / new_fl2; dispose-list teardown.
18. return error.

`Locks::__break_lease(inode, flags) -> Result<()>`:
1. type = match flags & LEASE_BREAK_*: LEASE → FL_LEASE | DELEG → FL_DELEG | LAYOUT → FL_LAYOUT | _ → -EINVAL.
2. want_write = !(flags & LEASE_BREAK_OPEN_RDONLY).
3. new_fl = lease_alloc(NULL, type, want_write ? F_WRLCK : F_RDLCK)?;
4. ctx = locks_inode_context(inode); if ctx.is_none(): WARN; goto free.
5. percpu_down_read(&file_rwsem); spin_lock(&ctx.flc_lock).
6. time_out_leases(inode, &dispose).
7. if !any_leases_conflict(inode, new_fl): goto out.
8. break_time = lease_break_time > 0 ? jiffies + lease_break_time * HZ : 0.
9. for (fl, tmp) in ctx.flc_lease.iter_safe():
   - if !leases_conflict(&fl.c, &new_fl.c): continue.
   - if want_write:
     - if FL_UNLOCK_PENDING: continue.
     - fl.c.flc_flags |= FL_UNLOCK_PENDING; fl.fl_break_time = break_time.
   - else:
     - if lease_breaking(fl): continue.
     - fl.c.flc_flags |= FL_DOWNGRADE_PENDING; fl.fl_downgrade_time = break_time.
   - if fl.fl_lmops.lm_break(fl): locks_delete_lock_ctx(&fl.c, &dispose).
10. if list_empty(&ctx.flc_lease): goto out.
11. if LEASE_BREAK_NONBLOCK: error = -EWOULDBLOCK; goto out.
12. spin_unlock + percpu_up_read; wait_event_interruptible_timeout(new_fl.c.flc_wait, break complete, break_time - jiffies).
13. re-acquire; cleanup; locks_dispose_list.

`Locks::posix_locks_deadlock(caller_fl, block_fl) -> bool`:
1. lockdep_assert_held(&blocked_lock_lock).
2. if caller.flc_flags & FL_OFDLCK: return false.
3. blocker = &block_fl.c; i = 0.
4. loop: blocker = what_owner_is_waiting_for(blocker); if blocker.is_none(): return false.
5. if ++i > MAX_DEADLK_ITERATIONS (10): return false.
6. if posix_same_owner(caller, blocker): return true.

`Locks::flock_lock_inode(inode, request) -> Result<()>`:
1. ctx = locks_get_lock_context(inode, type)?;
2. Preallocate new_fl if !FL_ACCESS ∧ !F_UNLCK.
3. percpu_down_read(&file_rwsem); spin_lock(&ctx.flc_lock).
4. /* Match same filp */
5. for fl in ctx.flc_flock: if request.flc_file == fl.flc_file:
   - if same type: goto out.
   - locks_delete_lock_ctx(&fl.c, &dispose); found = true; break.
6. if F_UNLCK: error = (FL_EXISTS ∧ !found) ? -ENOENT : 0; goto out.
7. /* Conflict scan */
8. for fl in ctx.flc_flock:
   - if !flock_locks_conflict(request, fl): continue.
   - if !FL_SLEEP: error = -EAGAIN; goto out.
   - locks_insert_block(&fl.c, &request.c, flock_locks_conflict); error = FILE_LOCK_DEFERRED; goto out.
9. locks_copy_lock(new_fl, request); locks_move_blocks(new_fl, request); locks_insert_lock_ctx(&new_fl.c, &ctx.flc_flock).

`Locks::fcntl_setlk(fd, filp, cmd, flock) -> Result<()>`:
1. file_lock = locks_alloc_lock()?;
2. flock_to_posix_lock(filp, file_lock, flock)?;
3. check_fmode_for_setlk(file_lock)?;
4. match cmd: F_OFD_SETLK → cmd=F_SETLK + FL_OFDLCK + flc_owner=filp (require l_pid==0); F_OFD_SETLKW → cmd=F_SETLKW + FL_OFDLCK + FL_SLEEP; F_SETLKW → FL_SLEEP.
5. error = do_lock_file_wait(filp, cmd, file_lock).
6. /* Close-race detection */
7. if !error ∧ !F_UNLCK ∧ !FL_OFDLCK:
   - spin_lock(&files.file_lock); f = files_lookup_fd_locked(files, fd); spin_unlock.
   - if f != filp: locks_remove_posix(filp, files); error = -EBADF.

`Locks::sys_flock(fd, cmd) -> Result<()>`:
1. if cmd & LOCK_MAND: pr_warn_once; return 0.
2. type = flock_translate_cmd(cmd & ~LOCK_NB)?;
3. fd_lookup(fd)?; require FMODE_READ|FMODE_WRITE for non-unlock.
4. flock_make_lock(filp, &fl, type) — FL_FLOCK; owner=filp; fl_start=0; fl_end=OFFSET_MAX.
5. security_file_lock(filp, type)?;
6. if !(cmd & LOCK_NB): fl.c.flc_flags |= FL_SLEEP.
7. Dispatch: fop->flock?(F_SETLK/W, &fl) or locks_lock_file_wait(filp, &fl).

### Out of Scope

- fs/locks-related lockd/NFSv4 grace handling (covered in NFS Tier-3 if expanded)
- fs/nfs* delegation issuance (covered in `nfs/` Tier-3 set)
- fanotify permission events (covered in `notify-core.md` Tier-3 + fanotify Tier-3)
- fcntl ownership / fcntl_dirnotify (dnotify Tier-3)
- mandatory-locking (removed upstream)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct file_lock` | per-byte-range lock record | `FileLock` |
| `struct file_lease` | per-lease record | `FileLease` |
| `struct file_lock_core` | shared owner/type/flags + tree links | `FileLockCore` |
| `struct file_lock_context` | per-inode triple-list + flc_lock | `FileLockContext` |
| `locks_alloc_lock()` / `locks_alloc_lease()` | per-allocate | `Locks::alloc_lock` / `alloc_lease` |
| `locks_free_lock()` / `locks_free_lease()` | per-free | `Locks::free_lock` / `free_lease` |
| `locks_copy_conflock()` / `locks_copy_lock()` | per-copy (F_GETLK / split) | `Locks::copy_conflock` / `copy_lock` |
| `posix_lock_file()` / `posix_lock_inode()` | per-POSIX apply | `Locks::posix_lock_file` / `posix_lock_inode` |
| `posix_lock_inode_wait()` | per-blocking POSIX apply | `Locks::posix_lock_inode_wait` |
| `posix_test_lock()` | per-F_GETLK probe | `Locks::posix_test_lock` |
| `posix_locks_conflict()` / `posix_test_locks_conflict()` | per-overlap+type test | `Locks::posix_locks_conflict` / `posix_test_locks_conflict` |
| `posix_locks_deadlock()` | per-cycle test | `Locks::posix_locks_deadlock` |
| `flock_lock_inode()` / `flock_lock_inode_wait()` | per-BSD apply | `Locks::flock_lock_inode` / `flock_lock_inode_wait` |
| `flock_locks_conflict()` | per-BSD conflict | `Locks::flock_locks_conflict` |
| `flock_to_posix_lock()` / `flock64_to_posix_lock()` | per-uapi convert | `Locks::flock_to_posix_lock` / `flock64_to_posix_lock` |
| `lease_alloc()` / `lease_init()` | per-lease alloc | `Locks::lease_alloc` / `lease_init` |
| `lease_modify()` | per-lease type change | `Locks::lease_modify` |
| `__break_lease()` | per-break entry | `Locks::break_lease` |
| `time_out_leases()` | per-timeout sweep | `Locks::time_out_leases` |
| `any_leases_conflict()` / `leases_conflict()` | per-lease overlap | `Locks::any_leases_conflict` / `leases_conflict` |
| `fcntl_getlk()` / `fcntl_getlk64()` | per-F_GETLK | `Locks::fcntl_getlk` / `fcntl_getlk64` |
| `fcntl_setlk()` / `fcntl_setlk64()` | per-F_SETLK / F_OFD_SETLK | `Locks::fcntl_setlk` / `fcntl_setlk64` |
| `fcntl_setlease()` / `fcntl_getlease()` | per-lease syscall | `Locks::fcntl_setlease` / `fcntl_getlease` |
| `vfs_lock_file()` / `vfs_test_lock()` / `vfs_cancel_lock()` | per-VFS dispatch | `Locks::vfs_lock_file` / `vfs_test_lock` / `vfs_cancel_lock` |
| `locks_lock_inode_wait()` | per-FL_POSIX/FLOCK common path | `Locks::lock_inode_wait` |
| `locks_remove_posix()` / `locks_remove_flock()` / `locks_remove_lease()` / `locks_remove_file()` | per-close cleanup | `Locks::remove_posix` / `remove_flock` / `remove_lease` / `remove_file` |
| `locks_insert_block()` / `locks_delete_block()` / `__locks_wake_up_blocks()` | per-wait-tree manip | `Locks::insert_block` / `delete_block` / `wake_up_blocks` |
| `locks_insert_global_locks()` / `locks_insert_global_blocked()` | per-/proc/locks + per-deadlock hash | `Locks::insert_global_locks` / `insert_global_blocked` |
| `what_owner_is_waiting_for()` | per-deadlock walk step | `Locks::what_owner_is_waiting_for` |
| `SYSCALL_DEFINE2(flock, ...)` | per-uapi flock(2) | `Locks::sys_flock` |
| `vm.leases-enable` / `vm.lease-break-time` | per-sysctl | shared |

### compatibility contract

REQ-1: struct file_lock_core (shared head):
- flc_list: hlist node in per-inode flc_flock/_posix/_lease.
- flc_link: hlist node in per-CPU file_lock_list + global blocked_hash.
- flc_blocked_requests: list head of waiters blocked on this lock.
- flc_blocked_member: list member on blocker's flc_blocked_requests.
- flc_blocker: pointer to blocker (NULL = not blocked or wake done).
- flc_wait: wait_queue_head for sleeping waiters.
- flc_file: owning struct file.
- flc_owner: owner cookie (files_struct* for POSIX; filp for OFD/FLOCK).
- flc_pid: pid of owner (for /proc/locks; -1 for OFD).
- flc_link_cpu: per-CPU bucket index for hlist removal.
- flc_type: F_RDLCK / F_WRLCK / F_UNLCK.
- flc_flags: FL_POSIX | FL_FLOCK | FL_OFDLCK | FL_LEASE | FL_DELEG | FL_LAYOUT | FL_SLEEP | FL_ACCESS | FL_EXISTS | FL_UNLOCK_PENDING | FL_DOWNGRADE_PENDING | FL_CLOSE | ...

REQ-2: struct file_lock:
- c: file_lock_core (embedded head; container_of recovers).
- fl_start, fl_end: byte range [start, end] inclusive; end = OFFSET_MAX = whole file.
- fl_ops: per-fs callbacks (fl_copy_lock, fl_release_private).
- fl_lmops: per-lock-manager callbacks (lm_break, lm_change, lm_notify, lm_grant, lm_lock_expirable, lm_expire_lock, lm_breaker_owns_lease, lm_breaker_timedout, lm_mod_owner).
- (NFS/lockd flavor-specific tail; opaque to core).

REQ-3: struct file_lease:
- c: file_lock_core (embedded head).
- fl_break_time: jiffies when unlock-pending lease will be force-removed.
- fl_downgrade_time: jiffies when downgrade-pending lease will become F_RDLCK.
- fl_fasync: fasync_struct* for SIGIO delivery.
- fl_lmops: lease manager callbacks (lm_break required).

REQ-4: struct file_lock_context (per-inode):
- flc_lock: spinlock guarding all three lists.
- flc_flock: list of FL_FLOCK (BSD) locks.
- flc_posix: list of FL_POSIX (+ FL_OFDLCK) locks, sorted by owner then start.
- flc_lease: list of FL_LEASE / FL_DELEG / FL_LAYOUT.
- Allocated lazily by `locks_get_lock_context(inode, type)`; published via `inode->i_flctx` under `inode->i_lock` with IOP_FLCTX flag.

REQ-5: locks_alloc_lock / _alloc_lease:
- kmem_cache_alloc(filelock_cache / filelease_cache, GFP_KERNEL).
- locks_init_lock_heads(&fl.c): INIT_HLIST_NODE/list_heads + init_waitqueue_head(&flc_wait).
- Returns NULL on allocation failure.

REQ-6: locks_overlap(fl1, fl2):
- return (fl1.fl_end >= fl2.fl_start) ∧ (fl2.fl_end >= fl1.fl_start).

REQ-7: locks_conflict(caller, sys):
- if sys.flc_type == F_WRLCK: return true.
- if caller.flc_type == F_WRLCK: return true.
- return false.

REQ-8: posix_locks_conflict(caller, sys):
- /* Same owner never conflicts */
- if posix_same_owner(caller, sys): return false.
- if !locks_overlap(caller_fl, sys_fl): return false.
- return locks_conflict(caller, sys).

REQ-9: flock_locks_conflict(caller, sys):
- /* Same filp never conflicts */
- if caller.flc_file == sys.flc_file: return false.
- return locks_conflict(caller, sys).

REQ-10: posix_lock_inode(inode, request, conflock):
- ctx = locks_get_lock_context(inode, request.c.flc_type); if !ctx: return -ENOMEM (or 0 if F_UNLCK).
- Preallocate two file_lock if request may split (not FL_ACCESS, not full-file unlock).
- percpu_down_read(&file_rwsem); spin_lock(&ctx.flc_lock).
- /* Conflict scan */
- if request.c.flc_type != F_UNLCK:
  - for each fl in ctx.flc_posix:
    - if !posix_locks_conflict(request, fl): continue.
    - /* Lock-manager expiration hook (NFS) */
    - if fl.fl_lmops.lm_lock_expirable(fl): __module_get; drop locks; lm_expire_lock(); module_put; retry.
    - if conflock: locks_copy_conflock(conflock, fl).
    - error = -EAGAIN; if !(flags & FL_SLEEP) goto out.
    - error = -EDEADLK; spin_lock(&blocked_lock_lock).
    - __locks_wake_up_blocks(&request.c).
    - if !posix_locks_deadlock(request, fl):
      - error = FILE_LOCK_DEFERRED.
      - __locks_insert_block(&fl.c, &request.c, posix_locks_conflict).
    - spin_unlock(&blocked_lock_lock); goto out.
- /* Apply: scan same-owner range, merge adjacent same-type, split on type-change */
- Locate first same-owner lock; walk same-owner runs:
  - same type ∧ adjacent/overlap: merge endpoints; mark added=true.
  - different type: split if needed (uses preallocated new_fl, new_fl2 for left/right halves).
- If !added: insert new_fl at correct sorted position.
- locks_wake_up_blocks for right/left split children.
- spin_unlock; percpu_up_read; free unused preallocs; dispose-list teardown.
- return 0 / -EAGAIN / -EDEADLK / -ENOLCK / FILE_LOCK_DEFERRED.

REQ-11: posix_lock_file(filp, fl, conflock):
- return posix_lock_inode(file_inode(filp), fl, conflock).

REQ-12: posix_lock_inode_wait(inode, fl):
- might_sleep().
- loop: e = posix_lock_inode(inode, fl, NULL); if e != FILE_LOCK_DEFERRED: break.
- wait_event_interruptible(fl.c.flc_wait, list_empty(&fl.c.flc_blocked_member)).
- locks_delete_block(fl) on exit (cleans dangling waiter).

REQ-13: flock_lock_inode(inode, request):
- ctx = locks_get_lock_context.
- Preallocate new_fl if not FL_ACCESS / F_UNLCK.
- percpu_down_read(&file_rwsem); spin_lock(&ctx.flc_lock).
- Locate existing FLOCK for same filp; if same type: noop; if different: locks_delete_lock_ctx into dispose-list (found=true).
- If F_UNLCK: error = (FL_EXISTS ∧ !found) ? -ENOENT : 0; goto out.
- find_conflict: for each fl in ctx.flc_flock:
  - if !flock_locks_conflict: continue.
  - error = -EAGAIN; if !(FL_SLEEP) goto out.
  - error = FILE_LOCK_DEFERRED; locks_insert_block(&fl.c, &request.c, flock_locks_conflict); goto out.
- locks_copy_lock(new_fl, request); locks_move_blocks(new_fl, request); locks_insert_lock_ctx(&new_fl.c, &ctx.flc_flock).

REQ-14: __break_lease(inode, flags):
- type = LEASE_BREAK_LEASE→FL_LEASE / _DELEG→FL_DELEG / _LAYOUT→FL_LAYOUT; else -EINVAL.
- new_fl = lease_alloc(NULL, type, want_write ? F_WRLCK : F_RDLCK).
- ctx = locks_inode_context(inode); if !ctx: WARN; goto free.
- percpu_down_read(&file_rwsem); spin_lock(&ctx.flc_lock).
- time_out_leases(inode, &dispose).
- if !any_leases_conflict(inode, new_fl): goto out.
- break_time = lease_break_time > 0 ? jiffies + lease_break_time * HZ : 0.
- for fl in ctx.flc_lease:
  - if !leases_conflict(&fl.c, &new_fl.c): continue.
  - if want_write: skip if FL_UNLOCK_PENDING; set FL_UNLOCK_PENDING; fl.fl_break_time = break_time.
  - else: skip if lease_breaking(fl); set FL_DOWNGRADE_PENDING; fl.fl_downgrade_time = break_time.
  - if fl.fl_lmops.lm_break(fl): locks_delete_lock_ctx(&fl.c, &dispose).
- if LEASE_BREAK_NONBLOCK ∧ list non-empty: error = -EWOULDBLOCK; goto out.
- Otherwise wait_event_interruptible_timeout for break to complete.

REQ-15: posix_locks_deadlock(caller_fl, block_fl):
- lockdep_assert_held(&blocked_lock_lock).
- if caller.flc_flags & FL_OFDLCK: return false (no per-task owner).
- blocker = &block_fl.c; i = 0.
- while (blocker = what_owner_is_waiting_for(blocker)):
  - if ++i > MAX_DEADLK_ITERATIONS (=10): return false (bail).
  - if posix_same_owner(caller, blocker): return true.
- return false.

REQ-16: fcntl_setlk(fd, filp, cmd, flock):
- alloc file_lock; flock_to_posix_lock(filp, file_lock, flock).
- check_fmode_for_setlk: F_RDLCK requires FMODE_READ; F_WRLCK requires FMODE_WRITE; else -EBADF.
- Translate F_OFD_SETLK→F_SETLK + FL_OFDLCK + owner=filp (l_pid must be 0).
- Translate F_OFD_SETLKW→F_SETLKW + FL_OFDLCK + FL_SLEEP.
- Translate F_SETLKW: set FL_SLEEP.
- do_lock_file_wait(filp, cmd, file_lock): security_file_lock; loop vfs_lock_file + wait until !FILE_LOCK_DEFERRED.
- Close/fcntl race: re-lookup fd; if filp diverged ∧ non-unlock ∧ non-OFD: locks_remove_posix(filp, files); return -EBADF.

REQ-17: fcntl_getlk(filp, cmd, flock):
- alloc file_lock; require l_type ∈ {F_RDLCK, F_WRLCK} (F_OFD_GETLK exempt).
- flock_to_posix_lock(filp, fl, flock).
- if F_OFD_GETLK: require l_pid == 0; FL_OFDLCK; flc_owner = filp.
- vfs_test_lock(filp, fl): fop->lock(F_GETLK) or posix_test_lock.
- flock.l_type = fl.c.flc_type; if !F_UNLCK: posix_lock_to_flock.

REQ-18: sys_flock(fd, cmd):
- if cmd & LOCK_MAND: warn-once and return 0 (no-op; mandatory removed).
- type = flock_translate_cmd(cmd & ~LOCK_NB): LOCK_SH→F_RDLCK, LOCK_EX→F_WRLCK, LOCK_UN→F_UNLCK; else -EINVAL.
- Require FMODE_READ|FMODE_WRITE for non-unlock.
- flock_make_lock(filp, &fl, type): FL_FLOCK; owner=filp; fl_start=0; fl_end=OFFSET_MAX.
- security_file_lock; FL_SLEEP if !(cmd & LOCK_NB).
- Dispatch fop->flock(F_SETLK/W) or locks_lock_file_wait.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `posix_lock_inode_preserves_tree_property` | INVARIANT | per-apply: every waiter conflicts with all ancestors. |
| `posix_lock_inode_split_alloc_sufficient` | INVARIANT | per-apply: at most 2 preallocs consumed (left+right halves). |
| `posix_locks_deadlock_terminates` | INVARIANT | per-call: ≤MAX_DEADLK_ITERATIONS iterations then bail. |
| `posix_locks_deadlock_ofd_bypass` | INVARIANT | per-call with FL_OFDLCK: returns false. |
| `__break_lease_holds_flc_lock` | INVARIANT | per-call: flc_lock held during list mutation. |
| `posix_lock_inode_holds_file_rwsem` | INVARIANT | per-apply: file_rwsem held during global-list mutation. |
| `flc_owner_consistent_per_flavor` | INVARIANT | per-FL_POSIX: owner=files_struct*; per-FL_OFDLCK/FLOCK: owner=filp. |
| `locks_alloc_failure_returns_enomem` | INVARIANT | per-fcntl_setlk/getlk: alloc failure → -ENOLCK/-ENOMEM. |
| `close_race_detected_returns_ebadf` | INVARIANT | per-fcntl_setlk: filp diverged → -EBADF + locks_remove_posix. |

### Layer 2: TLA+

`fs/locks.tla`:
- Per-POSIX-acquire / per-FLOCK-acquire / per-OFD-acquire / per-lease-break.
- Properties:
  - `safety_no_two_writers_overlap` — per-applied: no two F_WRLCK locks from different owners overlap.
  - `safety_reader_writer_exclusive` — per-applied: F_RDLCK and F_WRLCK from different owners never overlap.
  - `safety_same_owner_merges` — per-POSIX: adjacent same-type same-owner segments merge.
  - `safety_deadlock_detected` — per-cycle A→B→A: posix_locks_deadlock returns true; -EDEADLK returned.
  - `safety_ofdlck_deadlock_bypass` — per-FL_OFDLCK: never -EDEADLK.
  - `safety_lease_break_eventually_completes` — per-__break_lease: completes within lease_break_time seconds or -EWOULDBLOCK.
  - `liveness_blocked_wakes_on_unlock` — per-unlock: all directly blocked waiters wake.
  - `liveness_fcntl_setlk_terminates` — per-call: completes (success / -EAGAIN / -EDEADLK / -ERESTARTSYS / -EBADF / -ENOLCK).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Locks::posix_lock_inode` post: ret ⟹ ctx.flc_posix tree-property preserved | `Locks::posix_lock_inode` |
| `Locks::flock_lock_inode` post: at most one FLOCK per (inode, filp) | `Locks::flock_lock_inode` |
| `Locks::__break_lease` post: list_empty(flc_lease) ∨ all conflicting marked _PENDING with break_time set | `Locks::break_lease` |
| `Locks::posix_locks_deadlock` post: returns bool; iter bound ≤ 10 | `Locks::posix_locks_deadlock` |
| `Locks::fcntl_setlk` post: success ⟹ lock recorded ∧ vfs_lock_file returned 0 | `Locks::fcntl_setlk` |
| `Locks::fcntl_getlk` post: ret 0 ⟹ flock.l_type ∈ {F_RDLCK, F_WRLCK, F_UNLCK} | `Locks::fcntl_getlk` |
| `Locks::locks_remove_file` post: zero locks for filp on inode across all three lists | `Locks::remove_file` |
| `Locks::sys_flock` post: LOCK_MAND ⟹ 0; LOCK_UN ⟹ no FLOCK for filp | `Locks::sys_flock` |

### Layer 4: Verus/Creusot functional

`Per-syscall (flock / fcntl-setlk / fcntl-setlease) → vfs dispatch → posix_lock_inode / flock_lock_inode / generic_setlease → block-tree manipulation → /proc/locks observable state` semantic equivalence: per-fcntl(2) / flock(2) / Documentation/filesystems/locks.rst.

### hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

File-locking reinforcement:

- **Per-flc_lock + file_rwsem layering** — defense against per-/proc/locks reader vs apply races.
- **Per-blocked_lock_lock strict ordering** — defense against per-deadlock-detector inconsistency.
- **Per-MAX_DEADLK_ITERATIONS bound** — defense against per-pathological-cycle CPU exhaustion (broken NFS / threads).
- **Per-FL_OFDLCK deadlock bypass** — defense against per-cross-process-owner false-positive.
- **Per-lease_break_time cap (default 45s)** — defense against per-lease-holder DoS of opener.
- **Per-LEASE_BREAK_NONBLOCK fast-fail** — defense against per-blocking-in-atomic open path.
- **Per-close-race -EBADF + locks_remove_posix** — defense against per-fd-reuse stale POSIX lock.
- **Per-FL_ACCESS dry-run mode** — defense against per-blocking-on-test path.
- **Per-LOCK_MAND warn-once-no-op** — defense against per-legacy mandatory-lock DoS.
- **Per-lm_lock_expirable retry loop** — defense against per-NFS-stuck-grace-period deadlock.
- **Per-locks_alloc preallocation before lock acquisition** — defense against per-alloc-under-lock latency.
- **Per-FMODE_READ/WRITE check** — defense against per-RDONLY-fd write-lock escalation.
- **Per-security_file_lock LSM hook** — defense against per-policy bypass of lock state.

