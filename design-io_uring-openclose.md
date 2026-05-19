---
title: "Tier-3: io_uring/openclose.c — IORING_OP_OPENAT / OPENAT2 / CLOSE / FIXED_FD_INSTALL / FIXED_FD_REMOVE"
tags: ["tier-3", "io_uring", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

io_uring openclose implements **asynchronous open and close of file descriptors** without blocking the submitter. Per-`IORING_OP_OPENAT` mirrors legacy `openat(2)` (flags via sqe->open_flags + mode via sqe->len). Per-`IORING_OP_OPENAT2` mirrors `openat2(2)` (struct open_how copied from userspace, supports RESOLVE_*). Per-`IORING_OP_CLOSE` mirrors `close(2)` for normal fds, with fixed-table close via sqe->file_index. Per-`IORING_OP_FIXED_FD_INSTALL` installs an io_uring fixed-file into the calling task's fd table (default O_CLOEXEC; IORING_FIXED_FD_NO_CLOEXEC disables). Per-fixed-fd-remove handled internally by `io_fixed_fd_remove` invoked when sqe->file_index is set on CLOSE. Open path uses **delayed filename** (`delayed_getname`, `filename_complete_delayed`) so prep does not pin user pages while the request waits in the queue. Open path force-async for O_TRUNC / O_CREAT / __O_TMPFILE (always -EAGAIN inline). Critical for: high-throughput servers that open many files per second without per-open syscall round-trip, container-style file-table population, sandbox enforcement via OPENAT2 resolve flags.

This Tier-3 covers `io_uring/openclose.c` (~448 lines). Pipe support (`io_pipe`) is included here because it shares the file-table install machinery; deeper pipe semantics defer to `fs/pipe.c` Tier-3.

### Acceptance Criteria

- [ ] AC-1: IORING_OP_OPENAT with O_RDONLY succeeds and posts CQE with new fd (normal-fd path).
- [ ] AC-2: IORING_OP_OPENAT2 with valid open_how (len ≥ OPEN_HOW_SIZE_VER0) succeeds; len < OPEN_HOW_SIZE_VER0 fails -EINVAL.
- [ ] AC-3: O_TRUNC | O_CREAT | __O_TMPFILE: req sets REQ_F_FORCE_ASYNC and runs in worker, not inline.
- [ ] AC-4: file_slot ≠ 0 with O_CLOEXEC: prep returns -EINVAL.
- [ ] AC-5: sqe->buf_index ≠ 0 on openat prep: returns -EINVAL.
- [ ] AC-6: REQ_F_FIXED_FILE on openat prep: returns -EBADF.
- [ ] AC-7: do_file_open returns -EAGAIN, !RESOLVE_CACHED, NONBLOCK issue: req returns -EAGAIN, filename preserved via putname_to_delayed for retry.
- [ ] AC-8: Successful open with fixed slot: io_fixed_fd_install consumed slot; CQE result == slot index (or alloc'd index).
- [ ] AC-9: IORING_OP_CLOSE with fd != 0 closes task fd; CQE == 0.
- [ ] AC-10: IORING_OP_CLOSE with file_slot != 0 removes fixed-table entry under submit lock.
- [ ] AC-11: IORING_OP_CLOSE: closing the ring's own fd (io_is_uring_fops) returns -EBADF.
- [ ] AC-12: IORING_OP_CLOSE: fd has f_op.flush + nonblock issue → -EAGAIN punt to worker.
- [ ] AC-13: IORING_OP_CLOSE with both fd and file_slot set: prep returns -EINVAL.
- [ ] AC-14: IORING_OP_FIXED_FD_INSTALL with REQ_F_CREDS: prep returns -EPERM.
- [ ] AC-15: IORING_OP_FIXED_FD_INSTALL default: installs into task fdtable with O_CLOEXEC.

### Architecture

```
struct IoOpen {
  file: *File,
  dfd: i32,
  file_slot: u32,
  filename: DelayedFilename,
  how: OpenHow {
    flags: u64,
    mode: u64,
    resolve: u64,
  },
  nofile: u64,
}

struct IoClose {
  file: *File,
  fd: i32,
  file_slot: u32,
}

struct IoFixedInstall {
  file: *File,
  o_flags: u32,
}

struct IoPipe {
  file: *File,
  fds: *mut [i32; 2],   // userspace
  flags: i32,
  file_slot: i32,
  nofile: u64,
}
```

`IoUringOpenClose::openat_prep_common(req, sqe)`:
1. if sqe.buf_index != 0: return -EINVAL.
2. if req.flags.contains(REQ_F_FIXED_FILE): return -EBADF.
3. if !(open.how.flags & O_PATH) && force_o_largefile(): open.how.flags |= O_LARGEFILE.
4. open.dfd = sqe.fd as i32.
5. let fname = sqe.addr as *const c_char.
6. let ret = delayed_getname(&mut open.filename, fname); if ret != 0 { return ret; }
7. req.flags |= REQ_F_NEED_CLEANUP.
8. open.file_slot = sqe.file_index.
9. if open.file_slot != 0 && (open.how.flags & O_CLOEXEC) != 0: return -EINVAL.
10. open.nofile = rlimit(RLIMIT_NOFILE).
11. if Self::openat_force_async(&open): req.flags |= REQ_F_FORCE_ASYNC.
12. return 0.

`IoUringOpenClose::openat_prep(req, sqe)`:
1. let mode = sqe.len as u64.
2. let flags = sqe.open_flags as u64.
3. open.how = build_open_how(flags, mode).
4. Self::openat_prep_common(req, sqe).

`IoUringOpenClose::openat2_prep(req, sqe)`:
1. let how_user = sqe.addr2 as *const OpenHow.
2. let len = sqe.len as usize.
3. if len < OPEN_HOW_SIZE_VER0: return -EINVAL.
4. let ret = copy_struct_from_user(&mut open.how, size_of::<OpenHow>(), how_user, len); if ret != 0 { return ret; }
5. Self::openat_prep_common(req, sqe).

`IoUringOpenClose::openat2(req, issue_flags) -> IouResult`:
1. let mut op: OpenFlagsResolved = OpenFlagsResolved::default();
2. let ret = build_open_flags(&open.how, &mut op); if ret != 0 { goto err; }
3. let nonblock_set = (op.open_flag & O_NONBLOCK) != 0.
4. let resolve_nonblock = (open.how.resolve & RESOLVE_CACHED) != 0.
5. if issue_flags.contains(IO_URING_F_NONBLOCK):
   - debug_assert!(!Self::openat_force_async(&open)).
   - op.lookup_flags |= LOOKUP_CACHED.
   - op.open_flag |= O_NONBLOCK.
6. let fixed = open.file_slot != 0.
7. let mut ret: i32; if !fixed { ret = get_unused_fd_flags(open.how.flags, open.nofile); if ret < 0 { goto err; } }
8. let file = do_file_open(open.dfd, &name, &op).
9. if file.is_err():
   - if !fixed: put_unused_fd(ret).
   - ret = file.unwrap_err() as i32.
   - if ret == -EAGAIN && !resolve_nonblock && issue_flags.contains(IO_URING_F_NONBLOCK):
     - putname_to_delayed(&mut open.filename, no_free_ptr(name)).
     - return -EAGAIN.
   - goto err.
10. let file = file.unwrap().
11. if issue_flags.contains(IO_URING_F_NONBLOCK) && !nonblock_set: file.f_flags &= !O_NONBLOCK.
12. if !fixed: fd_install(ret, file).
13. else: ret = io_fixed_fd_install(req, issue_flags, file, open.file_slot).
14. err: req.flags &= !REQ_F_NEED_CLEANUP; if ret < 0 { req_set_fail(req); } io_req_set_res(req, ret, 0); return IouResult::Complete.

`IoUringOpenClose::open_cleanup(req)`:
1. dismiss_delayed_filename(&mut open.filename).

`IoUringOpenClose::close_prep(req, sqe)`:
1. if sqe.off | sqe.addr | sqe.len | sqe.rw_flags | sqe.buf_index != 0: return -EINVAL.
2. if req.flags.contains(REQ_F_FIXED_FILE): return -EBADF.
3. close.fd = sqe.fd as i32; close.file_slot = sqe.file_index.
4. if close.file_slot != 0 && close.fd != 0: return -EINVAL.
5. return 0.

`IoUringOpenClose::close(req, issue_flags) -> IouResult`:
1. let files = current().files.
2. let mut ret = -EBADF.
3. if close.file_slot != 0:
   - ret = Self::close_fixed_inner(req.ctx, issue_flags, close.file_slot - 1).
   - goto err.
4. let _g = files.file_lock.lock().
5. let file = files_lookup_fd_locked(&files, close.fd).
6. if file.is_none() || io_is_uring_fops(&file): goto err.
7. if file.f_op.flush.is_some() && issue_flags.contains(IO_URING_F_NONBLOCK): drop(_g); return IouResult::Eagain.
8. let file = file_close_fd_locked(&files, close.fd).
9. drop(_g).
10. if file.is_none(): goto err.
11. ret = filp_close(file.unwrap(), &current().files).
12. err: if ret < 0 { req_set_fail(req); } io_req_set_res(req, ret, 0); return IouResult::Complete.

`IoUringOpenClose::close_fixed_inner(ctx, issue_flags, offset) -> i32`:
1. ctx.submit_lock(issue_flags).
2. let ret = io_fixed_fd_remove(ctx, offset).
3. ctx.submit_unlock(issue_flags).
4. return ret.

`IoUringOpenClose::install_fixed_fd_prep(req, sqe) -> i32`:
1. if sqe.off | sqe.addr | sqe.len | sqe.buf_index | sqe.splice_fd_in | sqe.addr3 != 0: return -EINVAL.
2. if !req.flags.contains(REQ_F_FIXED_FILE): return -EBADF.
3. let flags = sqe.install_fd_flags.
4. if flags & !IORING_FIXED_FD_NO_CLOEXEC != 0: return -EINVAL.
5. if req.flags.contains(REQ_F_CREDS): return -EPERM.
6. ifi.o_flags = if flags & IORING_FIXED_FD_NO_CLOEXEC != 0 { 0 } else { O_CLOEXEC }.
7. return 0.

`IoUringOpenClose::install_fixed_fd(req, issue_flags) -> IouResult`:
1. let ret = receive_fd(req.file, None, ifi.o_flags).
2. if ret < 0 { req_set_fail(req); } io_req_set_res(req, ret, 0); return IouResult::Complete.

`IoUringOpenClose::pipe(req, issue_flags) -> IouResult`:
1. let mut files: [Option<File>; 2] = [None, None].
2. let ret = create_pipe_files(&mut files, p.flags); if ret != 0 { return ret; }
3. let ret = if p.file_slot != 0 { Self::pipe_fixed(req, &mut files, issue_flags) } else { Self::pipe_fd(req, &mut files) }.
4. io_req_set_res(req, ret, 0).
5. if ret == 0 { return IouResult::Complete; }
6. req_set_fail(req).
7. for f in files: if let Some(file) = f { fput(file); }
8. return ret.

### Out of Scope

- VFS `do_file_open`, `build_open_how`, `build_open_flags` (covered in `fs/open.md` Tier-3).
- Delayed filename machinery `delayed_getname` / `filename_complete_delayed` (covered in `fs/namei.md` Tier-3).
- Task fdtable primitives `__get_unused_fd_flags` / `fd_install` / `file_close_fd_locked` / `filp_close` (covered in `fs/filetable.md` Tier-3).
- io_uring fixed-file table internals (`io_fixed_fd_install`, `__io_fixed_fd_install`, `io_fixed_fd_remove`) covered in `io_uring/rsrc.md`.
- Pipe semantics beyond pair creation (covered in `fs/pipe.md` Tier-3 if expanded).
- BPF context populate (`io_openat_bpf_populate`) covered by io_uring BPF Tier-3.
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_open` | per-OPENAT/OPENAT2 cmd payload | `IoOpen` |
| `struct io_close` | per-CLOSE cmd payload | `IoClose` |
| `struct io_fixed_install` | per-FIXED_FD_INSTALL cmd payload | `IoFixedInstall` |
| `struct io_pipe` | per-PIPE cmd payload | `IoPipe` |
| `io_openat_prep()` | per-OPENAT prep | `IoUringOpenClose::openat_prep` |
| `io_openat2_prep()` | per-OPENAT2 prep | `IoUringOpenClose::openat2_prep` |
| `__io_openat_prep()` | per-shared prep tail | `IoUringOpenClose::openat_prep_common` |
| `io_openat()` / `io_openat2()` | per-issue (OPENAT2 is the impl) | `IoUringOpenClose::openat` / `openat2` |
| `io_openat_force_async()` | per-force-async predicate | `IoUringOpenClose::openat_force_async` |
| `io_open_cleanup()` | per-cancel/teardown cleanup | `IoUringOpenClose::open_cleanup` |
| `io_openat_bpf_populate()` | per-BPF context populate | `IoUringOpenClose::openat_bpf_populate` |
| `io_close_prep()` | per-CLOSE prep | `IoUringOpenClose::close_prep` |
| `io_close()` | per-CLOSE issue | `IoUringOpenClose::close` |
| `io_close_fixed()` | per-fixed-table close inner | `IoUringOpenClose::close_fixed` |
| `__io_close_fixed()` | per-fixed-table close (ctx-locked) | `IoUringOpenClose::close_fixed_inner` |
| `io_install_fixed_fd_prep()` | per-FIXED_FD_INSTALL prep | `IoUringOpenClose::install_fixed_fd_prep` |
| `io_install_fixed_fd()` | per-FIXED_FD_INSTALL issue | `IoUringOpenClose::install_fixed_fd` |
| `io_pipe_prep()` | per-PIPE prep | `IoUringOpenClose::pipe_prep` |
| `io_pipe()` | per-PIPE issue | `IoUringOpenClose::pipe` |
| `io_pipe_fixed()` | per-PIPE fixed-table path | `IoUringOpenClose::pipe_fixed` |
| `io_pipe_fd()` | per-PIPE normal-fd path | `IoUringOpenClose::pipe_fd` |
| `do_file_open()` | per-VFS open | shared (`vfs::do_file_open`) |
| `build_open_how()` / `build_open_flags()` | per-flag canonicalize | shared (`fs::open_flags`) |
| `delayed_getname()` / `filename_complete_delayed()` / `dismiss_delayed_filename()` / `putname_to_delayed()` | per-filename deferred resolve | shared (`fs::namei`) |
| `__get_unused_fd_flags()` / `put_unused_fd()` / `fd_install()` | per-fdtable alloc/install | shared (`fs::filetable`) |
| `files_lookup_fd_locked()` / `file_close_fd_locked()` / `filp_close()` | per-fdtable close | shared (`fs::filetable`) |
| `receive_fd()` | per-fd receive (used by install_fixed_fd) | shared (`fs::filetable`) |
| `io_fixed_fd_install()` / `__io_fixed_fd_install()` / `io_fixed_fd_remove()` | per-io_uring fixed-table | `IoUringFiletable::*` |
| `io_ring_submit_lock()` / `io_ring_submit_unlock()` | per-ctx submit lock | `IoRingCtx::submit_{lock,unlock}` |
| `create_pipe_files()` | per-pipe pair create | shared (`fs::pipe`) |
| `io_is_uring_fops()` | per-detect ring-fd | `IoRing::is_uring_fops` |
| `IORING_FIXED_FD_NO_CLOEXEC` | per-install flag | uapi constant |
| `IORING_FILE_INDEX_ALLOC` | per-slot auto-alloc sentinel | uapi constant |
| `OPEN_HOW_SIZE_VER0` | per-openat2 min size | uapi constant |
| `RESOLVE_CACHED` | per-openat2 cached-lookup hint | uapi constant |
| `REQ_F_FIXED_FILE` / `REQ_F_NEED_CLEANUP` / `REQ_F_FORCE_ASYNC` / `REQ_F_CREDS` | per-req flags | `ReqFlags` bits |
| `IO_URING_F_NONBLOCK` | per-issue nonblock hint | `IssueFlags::NONBLOCK` |
| `IOU_COMPLETE` | per-issue completion sentinel | `IouResult::Complete` |

### compatibility contract

REQ-1: struct io_open layout (in io_kiocb cmd cache):
- file: *File (held for force-async path; unused on open).
- dfd: i32 (sqe->fd; directory fd, AT_FDCWD permitted).
- file_slot: u32 (sqe->file_index; 0 = normal fd, non-zero = fixed-slot + 1).
- filename: DelayedFilename (per `delayed_getname`).
- how: struct open_how (flags, mode, resolve).
- nofile: u64 (cached RLIMIT_NOFILE at prep).

REQ-2: struct io_close layout:
- file: *File (unused on issue).
- fd: i32 (sqe->fd; only valid when file_slot == 0).
- file_slot: u32 (sqe->file_index; only valid when fd == 0).

REQ-3: struct io_fixed_install layout:
- file: *File (the fixed-file to install into task fdtable).
- o_flags: u32 (O_CLOEXEC by default; cleared if IORING_FIXED_FD_NO_CLOEXEC).

REQ-4: io_openat_force_async(open) → bool:
- Returns true iff open.how.flags & (O_TRUNC | O_CREAT | __O_TMPFILE).
- Note: uses __O_TMPFILE not O_TMPFILE because O_TMPFILE includes O_DIRECTORY which alone does not require async.

REQ-5: __io_openat_prep(req, sqe):
- /* Reject sqe->buf_index — open does not consume a registered buffer */
- if sqe->buf_index ≠ 0: return -EINVAL.
- /* Reject ring-resident fixed-file requests — openat needs a real dfd */
- if req.flags & REQ_F_FIXED_FILE: return -EBADF.
- /* O_LARGEFILE on 32-bit unless O_PATH */
- if !(open.how.flags & O_PATH) ∧ force_o_largefile(): open.how.flags |= O_LARGEFILE.
- open.dfd = READ_ONCE(sqe->fd).
- fname = u64_to_user_ptr(READ_ONCE(sqe->addr)).
- ret = delayed_getname(&open.filename, fname); if ret ≠ 0: return ret.
- req.flags |= REQ_F_NEED_CLEANUP.
- open.file_slot = READ_ONCE(sqe->file_index).
- /* Fixed-slot install incompatible with O_CLOEXEC (slot is ring-owned, no task fd to flag) */
- if open.file_slot ≠ 0 ∧ (open.how.flags & O_CLOEXEC): return -EINVAL.
- open.nofile = rlimit(RLIMIT_NOFILE).
- if io_openat_force_async(open): req.flags |= REQ_F_FORCE_ASYNC.
- return 0.

REQ-6: io_openat_prep(req, sqe):
- mode = READ_ONCE(sqe->len).
- flags = READ_ONCE(sqe->open_flags).
- open.how = build_open_how(flags, mode).
- return __io_openat_prep(req, sqe).

REQ-7: io_openat2_prep(req, sqe):
- how_user = u64_to_user_ptr(READ_ONCE(sqe->addr2)).
- len = READ_ONCE(sqe->len).
- if len < OPEN_HOW_SIZE_VER0: return -EINVAL.
- ret = copy_struct_from_user(&open.how, sizeof(open.how), how_user, len); if ret: return ret.
- return __io_openat_prep(req, sqe).

REQ-8: io_openat2(req, issue_flags) [shared issue body; io_openat() tail-calls it]:
- ret = build_open_flags(&open.how, &op); if ret: goto err.
- nonblock_set = op.open_flag & O_NONBLOCK.
- resolve_nonblock = open.how.resolve & RESOLVE_CACHED.
- if issue_flags & IO_URING_F_NONBLOCK:
  - WARN if io_openat_force_async (should have been deferred).
  - op.lookup_flags |= LOOKUP_CACHED.
  - op.open_flag |= O_NONBLOCK.
- fixed = (open.file_slot ≠ 0).
- if !fixed: ret = __get_unused_fd_flags(open.how.flags, open.nofile); if ret < 0: goto err.
- file = do_file_open(open.dfd, name, &op).
- if IS_ERR(file):
  - if !fixed: put_unused_fd(ret).
  - ret = PTR_ERR(file).
  - /* Retry path: if nonblock + EAGAIN + caller didn't already opt into cached resolve */
  - if ret == -EAGAIN ∧ !resolve_nonblock ∧ (issue_flags & IO_URING_F_NONBLOCK):
    - putname_to_delayed(&open.filename, no_free_ptr(name)).
    - return -EAGAIN.
  - goto err.
- /* Clear caller-implied O_NONBLOCK if we added it ourselves */
- if (issue_flags & IO_URING_F_NONBLOCK) ∧ !nonblock_set: file.f_flags &= ~O_NONBLOCK.
- if !fixed: fd_install(ret, file).
- else: ret = io_fixed_fd_install(req, issue_flags, file, open.file_slot).
- err: req.flags &= ~REQ_F_NEED_CLEANUP; if ret < 0: req_set_fail(req); io_req_set_res(req, ret, 0); return IOU_COMPLETE.

REQ-9: io_openat(req, issue_flags) == io_openat2(req, issue_flags) [identical body; legacy entry].

REQ-10: io_open_cleanup(req):
- dismiss_delayed_filename(&open.filename).

REQ-11: io_close_prep(req, sqe):
- if sqe->off ≠ 0 ∨ sqe->addr ≠ 0 ∨ sqe->len ≠ 0 ∨ sqe->rw_flags ≠ 0 ∨ sqe->buf_index ≠ 0: return -EINVAL.
- if req.flags & REQ_F_FIXED_FILE: return -EBADF.
- close.fd = READ_ONCE(sqe->fd).
- close.file_slot = READ_ONCE(sqe->file_index).
- /* Mutually exclusive: either close a normal fd OR a fixed slot */
- if close.file_slot ≠ 0 ∧ close.fd ≠ 0: return -EINVAL.
- return 0.

REQ-12: io_close(req, issue_flags):
- if close.file_slot ≠ 0:
  - ret = io_close_fixed(req, issue_flags) → __io_close_fixed(ctx, issue_flags, file_slot - 1):
    - io_ring_submit_lock(ctx, issue_flags).
    - ret = io_fixed_fd_remove(ctx, offset).
    - io_ring_submit_unlock(ctx, issue_flags).
  - goto err.
- spin_lock(&files.file_lock).
- file = files_lookup_fd_locked(files, close.fd).
- if !file ∨ io_is_uring_fops(file): spin_unlock; goto err with ret = -EBADF.
- /* If file has ->flush and we're nonblock, punt */
- if file.f_op.flush ∧ (issue_flags & IO_URING_F_NONBLOCK):
  - spin_unlock; return -EAGAIN.
- file = file_close_fd_locked(files, close.fd).
- spin_unlock.
- if !file: goto err.
- ret = filp_close(file, current.files).
- err: if ret < 0: req_set_fail(req); io_req_set_res(req, ret, 0); return IOU_COMPLETE.

REQ-13: io_install_fixed_fd_prep(req, sqe):
- if sqe->off ∨ sqe->addr ∨ sqe->len ∨ sqe->buf_index ∨ sqe->splice_fd_in ∨ sqe->addr3: return -EINVAL.
- /* Must already reference a fixed file via sqe->fd + IOSQE_FIXED_FILE */
- if !(req.flags & REQ_F_FIXED_FILE): return -EBADF.
- flags = READ_ONCE(sqe->install_fd_flags).
- if flags & ~IORING_FIXED_FD_NO_CLOEXEC: return -EINVAL.
- /* Personality / creds cannot be used: install must reflect task creds */
- if req.flags & REQ_F_CREDS: return -EPERM.
- ifi.o_flags = O_CLOEXEC; if flags & IORING_FIXED_FD_NO_CLOEXEC: ifi.o_flags = 0.
- return 0.

REQ-14: io_install_fixed_fd(req, issue_flags):
- ret = receive_fd(req.file, NULL, ifi.o_flags).  /* installs into current task fdtable */
- if ret < 0: req_set_fail(req).
- io_req_set_res(req, ret, 0); return IOU_COMPLETE.

REQ-15: io_pipe_prep(req, sqe):
- if sqe->fd ∨ sqe->off ∨ sqe->addr3: return -EINVAL.
- p.fds = u64_to_user_ptr(READ_ONCE(sqe->addr)).
- p.flags = READ_ONCE(sqe->pipe_flags).
- if p.flags & ~(O_CLOEXEC | O_NONBLOCK | O_DIRECT | O_NOTIFICATION_PIPE): return -EINVAL.
- p.file_slot = READ_ONCE(sqe->file_index).
- p.nofile = rlimit(RLIMIT_NOFILE).
- return 0.

REQ-16: io_pipe(req, issue_flags):
- ret = create_pipe_files(files[2], p.flags); if ret: return ret.
- if p.file_slot ≠ 0: ret = io_pipe_fixed(req, files, issue_flags).
- else: ret = io_pipe_fd(req, files).
- io_req_set_res(req, ret, 0).
- if !ret: return IOU_COMPLETE.
- /* Failure path drops unpublished file refs */
- req_set_fail(req); if files[0]: fput(files[0]); if files[1]: fput(files[1]); return ret.

REQ-17: io_pipe_fixed: per-IORING_FILE_INDEX_ALLOC auto-alloc or per-explicit-slot (read = slot, write = slot+1), copy_to_user on success; on EFAULT remove both installed slots.

REQ-18: io_pipe_fd: __get_unused_fd_flags×2, copy_to_user(p.fds); on EFAULT put both fds.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `openat_prep_rejects_buf_index` | INVARIANT | per-openat_prep_common: sqe.buf_index != 0 ⟹ -EINVAL. |
| `openat_prep_rejects_fixed_file` | INVARIANT | per-openat_prep_common: REQ_F_FIXED_FILE ⟹ -EBADF. |
| `openat_force_async_set_for_trunc_creat_tmpfile` | INVARIANT | per-prep: flags & (O_TRUNC|O_CREAT|__O_TMPFILE) ⟹ REQ_F_FORCE_ASYNC. |
| `openat_file_slot_xor_o_cloexec` | INVARIANT | per-prep: file_slot != 0 ∧ O_CLOEXEC ⟹ -EINVAL. |
| `openat2_min_size_enforced` | INVARIANT | per-openat2_prep: len < OPEN_HOW_SIZE_VER0 ⟹ -EINVAL. |
| `openat_eagain_preserves_filename` | INVARIANT | per-openat2 issue: -EAGAIN retry path stashes name via putname_to_delayed. |
| `openat_unused_fd_released_on_error` | INVARIANT | per-openat2 issue: !fixed ∧ do_file_open err ⟹ put_unused_fd(ret). |
| `close_fd_xor_file_slot` | INVARIANT | per-close_prep: fd != 0 ∧ file_slot != 0 ⟹ -EINVAL. |
| `close_uring_fop_rejected` | INVARIANT | per-close issue: io_is_uring_fops(file) ⟹ -EBADF. |
| `close_flush_punts_when_nonblock` | INVARIANT | per-close issue: f_op.flush ∧ NONBLOCK ⟹ -EAGAIN. |
| `install_fixed_fd_requires_fixed_file` | INVARIANT | per-install_fixed_fd_prep: !REQ_F_FIXED_FILE ⟹ -EBADF. |
| `install_fixed_fd_rejects_creds` | INVARIANT | per-install_fixed_fd_prep: REQ_F_CREDS ⟹ -EPERM. |
| `install_fixed_fd_unknown_flag_rejected` | INVARIANT | per-install_fixed_fd_prep: flags & !IORING_FIXED_FD_NO_CLOEXEC ⟹ -EINVAL. |
| `pipe_unknown_flag_rejected` | INVARIANT | per-pipe_prep: flags & ~(O_CLOEXEC|O_NONBLOCK|O_DIRECT|O_NOTIFICATION_PIPE) ⟹ -EINVAL. |
| `pipe_failure_releases_both_files` | INVARIANT | per-pipe: failure path fput's both file ends. |

### Layer 2: TLA+

`io_uring/openclose.tla`:
- States: prep_idle → prep_validated → queued → issued → completed | retried.
- Per-OPENAT / OPENAT2 / CLOSE / FIXED_FD_INSTALL / PIPE traces.
- Properties:
  - `safety_no_double_fd_install` — per-openat: at most one of fd_install / io_fixed_fd_install runs per success.
  - `safety_eagain_round_trip_preserves_name` — per-openat: retry path always has a valid filename.
  - `safety_close_mutex_fd_or_slot` — per-close: never both fd and slot acted upon.
  - `safety_install_fixed_fd_creds_invariant` — per-install: REQ_F_CREDS path never reaches receive_fd.
  - `liveness_open_completes_or_retries` — per-issue: every OPENAT2 terminates with CQE or REQ_F_FORCE_ASYNC reschedule.
  - `liveness_pipe_fixed_cleanup` — per-pipe-fixed: on partial install, both slots are removed under ctx submit lock.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `openat_prep_common` post: ret == 0 ⟹ REQ_F_NEED_CLEANUP set | `IoUringOpenClose::openat_prep_common` |
| `openat2_prep` post: open.how.flags / mode / resolve initialized | `IoUringOpenClose::openat2_prep` |
| `openat2` post: success ⟹ exactly-one of {fd_install, io_fixed_fd_install} | `IoUringOpenClose::openat2` |
| `openat2` post: !success ∧ !fixed ⟹ no unused fd leaked | `IoUringOpenClose::openat2` |
| `open_cleanup` post: filename dismissed | `IoUringOpenClose::open_cleanup` |
| `close_prep` post: at most one of {fd, file_slot} set | `IoUringOpenClose::close_prep` |
| `close` post: ret == -EAGAIN ⟹ files.file_lock not held | `IoUringOpenClose::close` |
| `close_fixed_inner` post: io_fixed_fd_remove ran under submit lock | `IoUringOpenClose::close_fixed_inner` |
| `install_fixed_fd_prep` post: o_flags ∈ {0, O_CLOEXEC} | `IoUringOpenClose::install_fixed_fd_prep` |
| `install_fixed_fd` post: receive_fd outcome in CQE | `IoUringOpenClose::install_fixed_fd` |
| `pipe` post: failure ⟹ both file ends fput'd | `IoUringOpenClose::pipe` |

### Layer 4: Verus/Creusot functional

`Per-OPENAT2 SQE → delayed_getname → optional REQ_F_FORCE_ASYNC → do_file_open (under task creds) → fd_install or io_fixed_fd_install → CQE` semantic equivalence: per-Documentation/io_uring.7 §IORING_OP_OPENAT2 and openat2(2). Per-CLOSE: equivalent to close(2) for normal fds with the extra rejection of io_uring ring fds; for fixed slots, equivalent to a paired IORING_REGISTER_FILES_UPDATE remove. Per-FIXED_FD_INSTALL: equivalent to a passthrough of receive_fd with task creds and configurable O_CLOEXEC.

### hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

OPENCLOSE reinforcement:

- **Per-prep REQ_F_FIXED_FILE rejection on openat** — defense against per-confusion where the ring-fixed dfd is taken as the open target.
- **Per-prep buf_index rejection** — defense against per-replay of a registered buffer through the open path.
- **Per-prep file_slot XOR O_CLOEXEC** — defense against per-uninstallable slot (ring-owned, no task fd).
- **Per-openat2 OPEN_HOW_SIZE_VER0 floor** — defense against per-short-struct UAF on copy_struct_from_user.
- **Per-force-async on O_TRUNC | O_CREAT | __O_TMPFILE** — defense against per-busy-loop EAGAIN under inline issue.
- **Per-EAGAIN retry filename preservation** — defense against per-double-getname / use-after-free of pending path.
- **Per-fd_install / put_unused_fd strict pairing** — defense against per-fd leak on do_file_open failure.
- **Per-close io_is_uring_fops rejection** — defense against per-ring-fd self-close (would unmap the ring while we're in it).
- **Per-close f_op.flush nonblock punt** — defense against per-fsync-stall on submitter thread.
- **Per-close fd XOR file_slot** — defense against per-cross-closure of unrelated fdtable entries.
- **Per-FIXED_FD_INSTALL REQ_F_CREDS rejection** — defense against per-personality-injection of fds with non-task creds.
- **Per-FIXED_FD_INSTALL unknown-flag rejection** — defense against per-ABI silent-acceptance of future flags.
- **Per-pipe failure both-ends fput** — defense against per-pipe-end leak.
- **Per-pipe_fixed full cleanup under submit lock** — defense against per-half-installed pipe pair.

