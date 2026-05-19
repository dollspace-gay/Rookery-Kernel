---
title: "Tier-3: io_uring/fs.c — io_uring filesystem path operations (renameat / unlinkat / mkdirat / symlinkat / linkat)"
tags: ["tier-3", "io_uring", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`io_uring/fs.c` implements the async wrappers for the path-manipulation family of filesystem syscalls as io_uring SQE opcodes: `IORING_OP_RENAMEAT`, `IORING_OP_UNLINKAT`, `IORING_OP_MKDIRAT`, `IORING_OP_SYMLINKAT`, and `IORING_OP_LINKAT`. Each op uses a dedicated command payload (`struct io_rename`, `struct io_unlink`, `struct io_mkdir`, `struct io_link` — the latter shared between symlink and hardlink) inline in the io_kiocb cmd area. Each op pulls user pathnames from `sqe->addr` and `sqe->addr2` and stores them in `struct delayed_filename` slots so that the actual `getname_flags` copy-from-user happens in the io-wq worker context (not the submission path). The two-phase split is: a `*_prep` validator that captures `dfd`, mode/flags, calls `delayed_getname[_uflags]` to seed the path slot, and marks the request `REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC`; an issue handler that runs in worker context, uses the `CLASS(filename_complete_delayed, ...)` RAII guard to complete the delayed getname (resolving the actual `struct filename`), dispatches to the `filename_*at` VFS primitives (`filename_renameat2`, `filename_rmdir` / `filename_unlinkat`, `filename_mkdirat`, `filename_symlinkat`, `filename_linkat`), and completes via `io_req_set_res` + `IOU_COMPLETE`. The cleanup hook dismisses any delayed_filename slots not yet consumed. All five ops reject `REQ_F_FIXED_FILE` (none use a registered fd as the dir-fd; `dfd` is a regular fd or `AT_FDCWD`). Critical for: async directory operations during high-throughput file ingest, archive expansion, container image unpack, atomic-swap deploys, and fanout-pattern path remap.

This Tier-3 covers `io_uring/fs.c` (~305 lines).

### Acceptance Criteria

- [ ] AC-1: IORING_OP_RENAMEAT: filename_renameat2(old_dfd, old, new_dfd, new, flags) called with prep-stored values.
- [ ] AC-2: IORING_OP_RENAMEAT with REQ_F_FIXED_FILE: -EBADF at prep.
- [ ] AC-3: IORING_OP_UNLINKAT with flags=0: filename_unlinkat called.
- [ ] AC-4: IORING_OP_UNLINKAT with flags=AT_REMOVEDIR: filename_rmdir called.
- [ ] AC-5: IORING_OP_UNLINKAT with flags=unknown: -EINVAL at prep.
- [ ] AC-6: IORING_OP_MKDIRAT: filename_mkdirat(dfd, name, mode) called; mode read from sqe->len.
- [ ] AC-7: IORING_OP_SYMLINKAT: filename_symlinkat(old, new_dfd, new) — note new_dfd from sqe->fd.
- [ ] AC-8: IORING_OP_LINKAT: filename_linkat with hardlink_flags; delayed_getname_uflags used for source (AT_EMPTY_PATH-aware).
- [ ] AC-9: All five ops set REQ_F_NEED_CLEANUP and REQ_F_FORCE_ASYNC during prep.
- [ ] AC-10: All five issue handlers clear REQ_F_NEED_CLEANUP after the RAII slot guard scope ends.
- [ ] AC-11: Prep failure mid-sequence (second delayed_getname fails) dismisses the first slot.
- [ ] AC-12: Cleanup handler dismisses any still-held delayed_filename slots.
- [ ] AC-13: All five issue handlers WARN_ON_ONCE if dispatched with IO_URING_F_NONBLOCK.
- [ ] AC-14: All five complete via io_req_set_res(req, vfs-ret, 0) + IOU_COMPLETE.

### Architecture

```
struct IoRename {
  file: *File,
  old_dfd: i32,
  new_dfd: i32,
  oldpath: DelayedFilename,
  newpath: DelayedFilename,
  flags: i32,
}
struct IoUnlink {
  file: *File,
  dfd: i32,
  flags: i32,                 // AT_REMOVEDIR only
  filename: DelayedFilename,
}
struct IoMkdir {
  file: *File,
  dfd: i32,
  mode: u16,                  // umode_t
  filename: DelayedFilename,
}
struct IoLink {
  file: *File,
  old_dfd: i32,
  new_dfd: i32,
  oldpath: DelayedFilename,
  newpath: DelayedFilename,
  flags: i32,
}
```

`IoRename::prep(req, sqe) -> Result<(), Errno>`:
1. if sqe.buf_index != 0 || sqe.splice_fd_in != 0: return Err(EINVAL).
2. if req.flags & REQ_F_FIXED_FILE: return Err(EBADF).
3. ren.old_dfd = read_once(&sqe.fd).
4. ren.new_dfd = read_once(&sqe.len) as i32.
5. ren.flags   = read_once(&sqe.rename_flags) as i32.
6. delayed_getname(&mut ren.oldpath, user_ptr(sqe.addr))?.
7. delayed_getname(&mut ren.newpath, user_ptr(sqe.addr2)).map_err(|e| { dismiss(&mut ren.oldpath); e })?.
8. req.flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC.
9. Ok(()).

`IoRename::issue(req, issue_flags) -> IouRet`:
1. warn_on_once(issue_flags & IO_URING_F_NONBLOCK).
2. with_delayed_filename(&mut ren.oldpath) as old => {
3.   with_delayed_filename(&mut ren.newpath) as new => {
4.     let ret = vfs::filename_renameat2(ren.old_dfd, old, ren.new_dfd, new, ren.flags).
5.     req.flags &= !REQ_F_NEED_CLEANUP.
6.     io_req_set_res(req, ret, 0).
7.     IouRet::Complete.
8. } }.

`IoUnlink::prep(req, sqe) -> Result<(), Errno>`:
1. if sqe.off || sqe.len || sqe.buf_index || sqe.splice_fd_in: return Err(EINVAL).
2. if req.flags & REQ_F_FIXED_FILE: return Err(EBADF).
3. un.dfd   = read_once(&sqe.fd).
4. un.flags = read_once(&sqe.unlink_flags) as i32.
5. if un.flags & !AT_REMOVEDIR != 0: return Err(EINVAL).
6. delayed_getname(&mut un.filename, user_ptr(sqe.addr))?.
7. req.flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC.
8. Ok(()).

`IoUnlink::issue(req, issue_flags) -> IouRet`:
1. warn_on_once(issue_flags & IO_URING_F_NONBLOCK).
2. with_delayed_filename(&mut un.filename) as name => {
3.   let ret = if un.flags & AT_REMOVEDIR != 0 {
4.       vfs::filename_rmdir(un.dfd, name)
5.   } else {
6.       vfs::filename_unlinkat(un.dfd, name)
7.   }.
8.   req.flags &= !REQ_F_NEED_CLEANUP.
9.   io_req_set_res(req, ret, 0).
10.  IouRet::Complete.
11. }.

`IoMkdir::prep(req, sqe) -> Result<(), Errno>`:
1. if sqe.off || sqe.rw_flags || sqe.buf_index || sqe.splice_fd_in: return Err(EINVAL).
2. if req.flags & REQ_F_FIXED_FILE: return Err(EBADF).
3. mkd.dfd  = read_once(&sqe.fd).
4. mkd.mode = read_once(&sqe.len) as u16.
5. delayed_getname(&mut mkd.filename, user_ptr(sqe.addr))?.
6. req.flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC.
7. Ok(()).

`IoLink::symlink_prep(req, sqe) -> Result<(), Errno>`:
1. if sqe.len || sqe.rw_flags || sqe.buf_index || sqe.splice_fd_in: return Err(EINVAL).
2. if req.flags & REQ_F_FIXED_FILE: return Err(EBADF).
3. sl.new_dfd = read_once(&sqe.fd).
4. delayed_getname(&mut sl.oldpath, user_ptr(sqe.addr))?.   /* symlink text */
5. delayed_getname(&mut sl.newpath, user_ptr(sqe.addr2)).map_err(|e| { dismiss(&mut sl.oldpath); e })?.
6. req.flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC.
7. Ok(()).

`IoLink::link_prep(req, sqe) -> Result<(), Errno>`:
1. if sqe.buf_index || sqe.splice_fd_in: return Err(EINVAL).
2. if req.flags & REQ_F_FIXED_FILE: return Err(EBADF).
3. lnk.old_dfd = read_once(&sqe.fd).
4. lnk.new_dfd = read_once(&sqe.len) as i32.
5. lnk.flags   = read_once(&sqe.hardlink_flags) as i32.
6. delayed_getname_uflags(&mut lnk.oldpath, user_ptr(sqe.addr), lnk.flags)?.
7. delayed_getname(&mut lnk.newpath, user_ptr(sqe.addr2)).map_err(|e| { dismiss(&mut lnk.oldpath); e })?.
8. req.flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC.
9. Ok(()).

`IoRename::cleanup(req)` / `IoUnlink::cleanup(req)` / `IoMkdir::cleanup(req)` / `IoLink::cleanup(req)`:
- dismiss_delayed_filename on all surviving slots.

### Out of Scope

- `filename_renameat2` / `filename_unlinkat` / `filename_rmdir` / `filename_mkdirat` / `filename_symlinkat` / `filename_linkat` (covered in `fs/namei.md` Tier-3)
- `delayed_getname` / `getname_flags` / `struct filename` lifecycle (covered in `fs/namei.md` Tier-3)
- io_uring core SQE→io_kiocb dispatch (covered in `io_uring/io_uring-core.md` Tier-3)
- io-wq worker pool (covered separately)
- `IORING_OP_OPENAT` / `OPENAT2` / `CLOSE` (covered in `io_uring/openclose.md` Tier-3)
- `IORING_OP_STATX` (covered in `io_uring/statx.md` Tier-3)
- `IORING_OP_FGETXATTR` / `FSETXATTR` / `GETXATTR` / `SETXATTR` (covered in `io_uring/xattr.md` Tier-3)
- Filesystem-level vfs_rename/unlink/etc dispatch (covered per-fs Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_rename` | per-RENAMEAT cmd payload | `IoRename` |
| `struct io_unlink` | per-UNLINKAT cmd payload | `IoUnlink` |
| `struct io_mkdir` | per-MKDIRAT cmd payload | `IoMkdir` |
| `struct io_link` | per-SYMLINKAT / LINKAT cmd payload | `IoLink` |
| `io_renameat_prep()` | per-prep | `IoRename::prep` |
| `io_renameat()` | per-issue | `IoRename::issue` |
| `io_renameat_cleanup()` | per-cleanup | `IoRename::cleanup` |
| `io_unlinkat_prep()` | per-prep | `IoUnlink::prep` |
| `io_unlinkat()` | per-issue | `IoUnlink::issue` |
| `io_unlinkat_cleanup()` | per-cleanup | `IoUnlink::cleanup` |
| `io_mkdirat_prep()` | per-prep | `IoMkdir::prep` |
| `io_mkdirat()` | per-issue | `IoMkdir::issue` |
| `io_mkdirat_cleanup()` | per-cleanup | `IoMkdir::cleanup` |
| `io_symlinkat_prep()` | per-prep | `IoLink::symlink_prep` |
| `io_symlinkat()` | per-issue | `IoLink::symlink_issue` |
| `io_linkat_prep()` | per-prep | `IoLink::link_prep` |
| `io_linkat()` | per-issue | `IoLink::link_issue` |
| `io_link_cleanup()` | per-cleanup | `IoLink::cleanup` |
| `struct delayed_filename` | per-pending getname slot | `DelayedFilename` |
| `delayed_getname()` | per-defer getname capture | extern from `fs/namei.md` |
| `delayed_getname_uflags()` | per-defer getname with at-flags | extern from `fs/namei.md` |
| `dismiss_delayed_filename()` | per-release pending slot | extern from `fs/namei.md` |
| `CLASS(filename_complete_delayed, x)(...)` | RAII guard: resolves slot ⟹ struct filename | extern from `fs/namei.md` |
| `filename_renameat2()` | per-VFS rename(at2) by filename | extern from `fs/namei.md` |
| `filename_rmdir()` | per-VFS rmdir by filename | extern from `fs/namei.md` |
| `filename_unlinkat()` | per-VFS unlinkat by filename | extern from `fs/namei.md` |
| `filename_mkdirat()` | per-VFS mkdirat by filename | extern from `fs/namei.md` |
| `filename_symlinkat()` | per-VFS symlinkat by filename | extern from `fs/namei.md` |
| `filename_linkat()` | per-VFS linkat by filename | extern from `fs/namei.md` |
| `REQ_F_FIXED_FILE` | per-req: dfd is registered slot (rejected here) | shared req-flag |
| `REQ_F_NEED_CLEANUP` | per-req: cleanup callback required | shared req-flag |
| `REQ_F_FORCE_ASYNC` | per-req: forces io-wq dispatch | shared req-flag |
| `IO_URING_F_NONBLOCK` | per-issue flag (must be cleared) | shared issue-flag |
| `IOU_COMPLETE` | per-issue: completion sentinel | shared issue-ret |
| `io_req_set_res()` | per-req: stamps result + cflags | shared req helper |
| `AT_REMOVEDIR` | per-UNLINKAT flag: rmdir vs unlink | shared UAPI |

### compatibility contract

REQ-1: `struct io_rename` — per-RENAMEAT payload:
- `file: *struct file` — unused (renameat does not bind to an open file).
- `old_dfd: i32` — per-source-directory fd.
- `new_dfd: i32` — per-target-directory fd.
- `oldpath: struct delayed_filename` — per-source path slot.
- `newpath: struct delayed_filename` — per-target path slot.
- `flags: i32` — per-`renameat2(2)` flags (`RENAME_EXCHANGE`, `RENAME_NOREPLACE`, `RENAME_WHITEOUT`).

REQ-2: `struct io_unlink` — per-UNLINKAT payload:
- `file: *struct file` — unused.
- `dfd: i32` — per-directory fd.
- `flags: i32` — `AT_REMOVEDIR` only (rejected otherwise).
- `filename: struct delayed_filename` — per-target path slot.

REQ-3: `struct io_mkdir` — per-MKDIRAT payload:
- `file: *struct file` — unused.
- `dfd: i32` — per-parent-directory fd.
- `mode: umode_t` — per-new-dir mode.
- `filename: struct delayed_filename` — per-new-name slot.

REQ-4: `struct io_link` — per-SYMLINKAT / LINKAT shared payload:
- `file: *struct file` — unused.
- `old_dfd: i32` — per-LINKAT source dir fd (unused by symlink prep; only the link target string is stored).
- `new_dfd: i32` — per-target dir fd.
- `oldpath: struct delayed_filename` — per-source slot (or per-symlink-text for symlinkat).
- `newpath: struct delayed_filename` — per-new-link slot.
- `flags: i32` — per-LINKAT flags (`AT_EMPTY_PATH`, `AT_SYMLINK_FOLLOW`).

REQ-5: `io_renameat_prep(req, sqe)` — per-RENAMEAT prep:
- /* SQE-field validation */
- if `sqe->buf_index` ∨ `sqe->splice_fd_in`: return -EINVAL.
- /* RENAMEAT is path-only; registered dfd not supported */
- if `req->flags & REQ_F_FIXED_FILE`: return -EBADF.
- `ren->old_dfd = READ_ONCE(sqe->fd)`.
- `oldf = u64_to_user_ptr(READ_ONCE(sqe->addr))`.
- `newf = u64_to_user_ptr(READ_ONCE(sqe->addr2))`.
- `ren->new_dfd = READ_ONCE(sqe->len)`.
- `ren->flags = READ_ONCE(sqe->rename_flags)`.
- /* Defer getname; copy-from-user happens at issue */
- `err = delayed_getname(&ren->oldpath, oldf)`. if err: return err.
- `err = delayed_getname(&ren->newpath, newf)`. if err: dismiss_delayed_filename(&oldpath); return err.
- `req->flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC`.
- return 0.

REQ-6: `io_renameat(req, issue_flags)` — per-RENAMEAT issue:
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- /* RAII guards complete delayed getname; auto-dismiss on early return */
- `CLASS(filename_complete_delayed, old)(&ren->oldpath)`.
- `CLASS(filename_complete_delayed, new)(&ren->newpath)`.
- `ret = filename_renameat2(ren->old_dfd, old, ren->new_dfd, new, ren->flags)`.
- /* Clear cleanup: RAII guards already consumed slots */
- `req->flags &= ~REQ_F_NEED_CLEANUP`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-7: `io_renameat_cleanup(req)` — per-RENAMEAT cleanup:
- `dismiss_delayed_filename(&ren->oldpath)`.
- `dismiss_delayed_filename(&ren->newpath)`.

REQ-8: `io_unlinkat_prep(req, sqe)` — per-UNLINKAT prep:
- if `sqe->off` ∨ `sqe->len` ∨ `sqe->buf_index` ∨ `sqe->splice_fd_in`: return -EINVAL.
- if `req->flags & REQ_F_FIXED_FILE`: return -EBADF.
- `un->dfd = READ_ONCE(sqe->fd)`.
- `un->flags = READ_ONCE(sqe->unlink_flags)`.
- if `un->flags & ~AT_REMOVEDIR`: return -EINVAL.
- `fname = u64_to_user_ptr(READ_ONCE(sqe->addr))`.
- `err = delayed_getname(&un->filename, fname)`. if err: return err.
- `req->flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC`.
- return 0.

REQ-9: `io_unlinkat(req, issue_flags)` — per-UNLINKAT issue:
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- `CLASS(filename_complete_delayed, name)(&un->filename)`.
- if `un->flags & AT_REMOVEDIR`: `ret = filename_rmdir(un->dfd, name)`.
- else: `ret = filename_unlinkat(un->dfd, name)`.
- `req->flags &= ~REQ_F_NEED_CLEANUP`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-10: `io_unlinkat_cleanup(req)` — per-UNLINKAT cleanup:
- `dismiss_delayed_filename(&ul->filename)`.

REQ-11: `io_mkdirat_prep(req, sqe)` — per-MKDIRAT prep:
- if `sqe->off` ∨ `sqe->rw_flags` ∨ `sqe->buf_index` ∨ `sqe->splice_fd_in`: return -EINVAL.
- if `req->flags & REQ_F_FIXED_FILE`: return -EBADF.
- `mkd->dfd = READ_ONCE(sqe->fd)`.
- `mkd->mode = READ_ONCE(sqe->len)`.
- `fname = u64_to_user_ptr(READ_ONCE(sqe->addr))`.
- `err = delayed_getname(&mkd->filename, fname)`. if err: return err.
- `req->flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC`.
- return 0.

REQ-12: `io_mkdirat(req, issue_flags)` — per-MKDIRAT issue:
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- `CLASS(filename_complete_delayed, name)(&mkd->filename)`.
- `ret = filename_mkdirat(mkd->dfd, name, mkd->mode)`.
- `req->flags &= ~REQ_F_NEED_CLEANUP`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-13: `io_symlinkat_prep(req, sqe)` — per-SYMLINKAT prep:
- if `sqe->len` ∨ `sqe->rw_flags` ∨ `sqe->buf_index` ∨ `sqe->splice_fd_in`: return -EINVAL.
- if `req->flags & REQ_F_FIXED_FILE`: return -EBADF.
- `sl->new_dfd = READ_ONCE(sqe->fd)`.
- `oldpath = u64_to_user_ptr(READ_ONCE(sqe->addr))`. /* symlink target text */
- `newpath = u64_to_user_ptr(READ_ONCE(sqe->addr2))`. /* new link path */
- `err = delayed_getname(&sl->oldpath, oldpath)`. if err: return err.
- `err = delayed_getname(&sl->newpath, newpath)`. if err: dismiss(&sl->oldpath); return err.
- `req->flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC`.
- return 0.

REQ-14: `io_symlinkat(req, issue_flags)` — per-SYMLINKAT issue:
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- `CLASS(filename_complete_delayed, old)(&sl->oldpath)`.
- `CLASS(filename_complete_delayed, new)(&sl->newpath)`.
- `ret = filename_symlinkat(old, sl->new_dfd, new)`.
- `req->flags &= ~REQ_F_NEED_CLEANUP`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-15: `io_linkat_prep(req, sqe)` — per-LINKAT prep:
- if `sqe->buf_index` ∨ `sqe->splice_fd_in`: return -EINVAL.
- if `req->flags & REQ_F_FIXED_FILE`: return -EBADF.
- `lnk->old_dfd = READ_ONCE(sqe->fd)`.
- `lnk->new_dfd = READ_ONCE(sqe->len)`.
- `oldf = u64_to_user_ptr(READ_ONCE(sqe->addr))`.
- `newf = u64_to_user_ptr(READ_ONCE(sqe->addr2))`.
- `lnk->flags = READ_ONCE(sqe->hardlink_flags)`.
- /* Uses _uflags variant so AT_EMPTY_PATH is honored */
- `err = delayed_getname_uflags(&lnk->oldpath, oldf, lnk->flags)`. if err: return err.
- `err = delayed_getname(&lnk->newpath, newf)`. if err: dismiss(&lnk->oldpath); return err.
- `req->flags |= REQ_F_NEED_CLEANUP | REQ_F_FORCE_ASYNC`.
- return 0.

REQ-16: `io_linkat(req, issue_flags)` — per-LINKAT issue:
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- `CLASS(filename_complete_delayed, old)(&lnk->oldpath)`.
- `CLASS(filename_complete_delayed, new)(&lnk->newpath)`.
- `ret = filename_linkat(lnk->old_dfd, old, lnk->new_dfd, new, lnk->flags)`.
- `req->flags &= ~REQ_F_NEED_CLEANUP`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-17: `io_link_cleanup(req)` — per-symlink/link cleanup (shared):
- `dismiss_delayed_filename(&sl->oldpath)`.
- `dismiss_delayed_filename(&sl->newpath)`.

REQ-18: Per-delayed-getname pattern (shared across all five ops):
- `delayed_getname(slot, user_ptr)` captures the user pointer into the slot without copying.
- At issue, `CLASS(filename_complete_delayed, name)(slot)` performs the actual `getname_flags` (copy-from-user, struct filename alloc).
- The RAII destructor at scope-exit releases the struct filename.
- On prep error mid-sequence, the partial slot must be `dismiss_delayed_filename`'d explicitly.
- On issue success, `REQ_F_NEED_CLEANUP` is cleared because RAII consumed the slots.
- On any error after prep but before issue, the cleanup handler runs and dismisses both slots.

REQ-19: Per-REQ_F_FIXED_FILE rejection (all five ops):
- io_uring path ops do not accept a registered file as `dfd`. `dfd` must be an AT_FDCWD-style raw fd or AT_FDCWD itself.
- If REQ_F_FIXED_FILE is set, prep returns -EBADF.

REQ-20: Per-REQ_F_FORCE_ASYNC + IO_URING_F_NONBLOCK invariant:
- All prep handlers set REQ_F_FORCE_ASYNC.
- All issue handlers WARN_ON_ONCE if invoked with IO_URING_F_NONBLOCK — they always need blocking context (path walk, dentry locks, journal commits).

REQ-21: Per-completion semantics:
- `filename_*` return is propagated raw via `io_req_set_res(req, ret, 0)`.
- 0 = success; negative = errno.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prep_rejects_fixed_file` | INVARIANT | per-all-prep: REQ_F_FIXED_FILE ⟹ -EBADF. |
| `prep_rejects_buf_index` | INVARIANT | per-all-prep: nonzero sqe.buf_index ⟹ -EINVAL. |
| `prep_rejects_splice_fd_in` | INVARIANT | per-all-prep: nonzero sqe.splice_fd_in ⟹ -EINVAL. |
| `unlink_prep_rejects_unknown_flags` | INVARIANT | per-unlink_prep: flags & ~AT_REMOVEDIR ⟹ -EINVAL. |
| `prep_dismisses_first_slot_on_second_err` | INVARIANT | per-rename/symlink/link_prep: 2nd delayed_getname err ⟹ 1st dismissed. |
| `prep_sets_cleanup_and_force_async` | INVARIANT | per-all-prep: REQ_F_NEED_CLEANUP and REQ_F_FORCE_ASYNC set on success. |
| `issue_in_blocking_context` | INVARIANT | per-all-issue: IO_URING_F_NONBLOCK never observed. |
| `issue_clears_cleanup_after_raii` | INVARIANT | per-all-issue: REQ_F_NEED_CLEANUP cleared after CLASS guards run. |
| `unlinkat_dispatches_correct_op` | INVARIANT | per-unlink_issue: AT_REMOVEDIR ⟹ filename_rmdir, else filename_unlinkat. |
| `linkat_uflags_path` | INVARIANT | per-link_prep: delayed_getname_uflags (not plain) used for source. |

### Layer 2: TLA+

`io_uring/fs.tla`:
- Per-prep + per-issue + per-cleanup states for each of {RENAMEAT, UNLINKAT, MKDIRAT, SYMLINKAT, LINKAT}.
- Properties:
  - `safety_no_fixed_file` — per-prep: REQ_F_FIXED_FILE ⟹ -EBADF.
  - `safety_slot_leak_free` — per-all-paths: every delayed_filename either becomes a struct filename then freed, or is dismissed.
  - `safety_unlinkat_flags_strict` — per-unlinkat_prep: only AT_REMOVEDIR accepted.
  - `safety_cleanup_idempotent` — per-cleanup: dismiss on already-completed slot is a no-op.
  - `liveness_each_issue_completes` — per-issue: terminates with IOU_COMPLETE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IoRename::prep` post: oldpath, newpath both `Captured` ∨ both `Empty` | `IoRename::prep` |
| `IoRename::issue` post: both slots consumed; REQ_F_NEED_CLEANUP cleared | `IoRename::issue` |
| `IoUnlink::prep` post: un.flags ⊆ {0, AT_REMOVEDIR} | `IoUnlink::prep` |
| `IoUnlink::issue` post: AT_REMOVEDIR ⟹ filename_rmdir called | `IoUnlink::issue` |
| `IoMkdir::prep` post: mkd.mode = sqe.len, mkd.dfd = sqe.fd | `IoMkdir::prep` |
| `IoLink::link_prep` post: lnk.{old_dfd,new_dfd,flags} = sqe.{fd,len,hardlink_flags} | `IoLink::link_prep` |
| `IoLink::cleanup` post: both slots dismissed | `IoLink::cleanup` |

### Layer 4: Verus/Creusot functional

`Per-SQE submit → prep validates + captures user-ptr in delayed slots → REQ_F_FORCE_ASYNC ⟹ worker dispatch → CLASS guard resolves getname + struct filename → filename_*at(dfd, name, ...) VFS call → io_req_set_res ⟹ CQE → RAII releases filename` semantic equivalence: per-`renameat2(2)`, `unlinkat(2)`, `mkdirat(2)`, `symlinkat(2)`, `linkat(2)` man pages.

### hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

FS-op reinforcement:

- **Per-REQ_F_FIXED_FILE rejection on dfd** — defense against per-registered-fd-as-directory ambiguity.
- **Per-sqe.{buf_index,splice_fd_in} reserved-field rejection** — defense against per-SQE field-aliasing exploits.
- **Per-unlinkat flag whitelist (AT_REMOVEDIR only)** — defense against per-forward-flag UAPI drift.
- **Per-delayed_getname defers copy_from_user to worker context** — defense against per-submit-thread blocking on a faulting page.
- **Per-second-getname-failure dismisses first slot** — defense against per-leak of pending getname slot.
- **Per-RAII CLASS guards for struct filename** — defense against per-error-path filename leak.
- **Per-cleanup handler runs on any failed-path** — defense against per-residual delayed_filename leak.
- **Per-REQ_F_FORCE_ASYNC enforced** — defense against per-blocking-in-submit-path stall.
- **Per-WARN_ON_ONCE on IO_URING_F_NONBLOCK** — defense against per-misdispatch (path ops in submit thread).
- **Per-linkat delayed_getname_uflags honors AT_EMPTY_PATH** — defense against per-link-from-fd silently failing.
- **Per-VFS delegation: filename_* primitives carry locking + capability checks** — defense against per-bypass of namei security.
- **Per-mkdirat mode passed unmodified** — defense against per-umask-already-applied double-mask.
- **Per-symlinkat new_dfd from sqe.fd (not sqe.len)** — defense against per-SQE-field-confusion bug.

