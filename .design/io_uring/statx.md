# Tier-3: io_uring/statx.c — IORING_OP_STATX

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/statx.c (~68 lines)
  - io_uring/statx.h
  - fs/stat.c (do_statx, vfs_statx)
  - fs/internal.h (delayed_filename, delayed_getname_uflags, filename_complete_delayed, dismiss_delayed_filename)
  - include/uapi/linux/io_uring.h (IORING_OP_STATX)
  - include/uapi/linux/stat.h (struct statx, STATX_*, AT_*)
-->

## Summary

io_uring statx implements **asynchronous statx(2)** so userspace can probe a path's metadata without blocking the submitter thread. Per-`IORING_OP_STATX` reads dfd from sqe->fd, path from sqe->addr, statx buffer from sqe->addr2, mask from sqe->len, AT_* flags from sqe->statx_flags. Prep uses **delayed filename** (`delayed_getname_uflags` — the uflags variant honors AT_EMPTY_PATH semantics) so the user path is materialised only when the worker runs. Issue path always runs **force-async** (REQ_F_FORCE_ASYNC set unconditionally at prep) because do_statx walks the filesystem and may block on disk I/O or LOOKUP_FOLLOW symlink-traversal; a WARN_ON_ONCE asserts the issue never reaches inline NONBLOCK context. Cleanup dismisses the delayed filename on cancellation. Critical for: high-throughput stat-storms (mail spools, `ls -l` over millions of files), prefetch / readahead daemons, fanotify-style watch builders that need bulk metadata.

This Tier-3 covers `io_uring/statx.c` (~68 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_statx` | per-STATX cmd payload | `IoStatx` |
| `io_statx_prep()` | per-prep | `IoUringStatx::statx_prep` |
| `io_statx()` | per-issue | `IoUringStatx::statx` |
| `io_statx_cleanup()` | per-cancel/teardown cleanup | `IoUringStatx::statx_cleanup` |
| `do_statx()` | per-VFS stat (resolves path, calls vfs_statx, copies to user) | shared (`fs::stat::do_statx`) |
| `delayed_getname_uflags()` | per-deferred path resolve honoring AT_* | shared (`fs::namei`) |
| `filename_complete_delayed` (CLASS) | per-issue path materialise | shared (`fs::namei`) |
| `dismiss_delayed_filename()` | per-cleanup drop | shared (`fs::namei`) |
| `REQ_F_FIXED_FILE` | per-req flag (incompatible) | `ReqFlags` bit |
| `REQ_F_NEED_CLEANUP` | per-req flag | `ReqFlags` bit |
| `REQ_F_FORCE_ASYNC` | per-req flag (always set) | `ReqFlags` bit |
| `IO_URING_F_NONBLOCK` | per-issue flag (must not be set) | `IssueFlags::NONBLOCK` |
| `IOU_COMPLETE` | per-issue completion sentinel | `IouResult::Complete` |

## Compatibility contract

REQ-1: struct io_statx layout (in io_kiocb cmd cache):
- file: *File (held but unused; statx is path-based).
- dfd: i32 (sqe->fd; AT_FDCWD permitted).
- mask: u32 (sqe->len; STATX_* mask).
- flags: u32 (sqe->statx_flags; AT_SYMLINK_NOFOLLOW, AT_NO_AUTOMOUNT, AT_EMPTY_PATH, AT_STATX_*_SYNC).
- filename: DelayedFilename (per `delayed_getname_uflags`).
- buffer: *mut Statx (user pointer to statx struct).

REQ-2: io_statx_prep(req, sqe):
- /* sqe->buf_index unused — statx does not consume a registered buffer */
- if sqe->buf_index ≠ 0 ∨ sqe->splice_fd_in ≠ 0: return -EINVAL.
- /* IOSQE_FIXED_FILE incompatible — statx needs a real dfd not a ring-fixed fd */
- if req.flags & REQ_F_FIXED_FILE: return -EBADF.
- sx.dfd = READ_ONCE(sqe->fd).
- sx.mask = READ_ONCE(sqe->len).
- path = u64_to_user_ptr(READ_ONCE(sqe->addr)).
- sx.buffer = u64_to_user_ptr(READ_ONCE(sqe->addr2)).
- sx.flags = READ_ONCE(sqe->statx_flags).
- /* uflags variant validates AT_EMPTY_PATH / AT_SYMLINK_NOFOLLOW at getname time */
- ret = delayed_getname_uflags(&sx.filename, path, sx.flags); if ret: return ret.
- req.flags |= REQ_F_NEED_CLEANUP.
- req.flags |= REQ_F_FORCE_ASYNC.   /* unconditional: must run in worker */
- return 0.

REQ-3: io_statx(req, issue_flags):
- /* Worker context only — never inline */
- WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK).
- name = filename_complete_delayed(&sx.filename).  /* CLASS-scoped; auto-putname on drop */
- ret = do_statx(sx.dfd, name, sx.flags, sx.mask, sx.buffer).
- io_req_set_res(req, ret, 0).
- return IOU_COMPLETE.

REQ-4: io_statx_cleanup(req):
- dismiss_delayed_filename(&sx.filename).
- /* Called when req is cancelled or torn down before issue completes */

REQ-5: do_statx contract (referenced from `fs/stat.c`):
- Resolves (dfd, name, AT_*) into a path.
- Calls vfs_getattr / vfs_statx to fill struct kstat.
- Copies kstat → user `struct statx` honoring requested mask.
- Returns 0 on success, -errno otherwise.

REQ-6: REQ_F_FORCE_ASYNC unconditional set at prep:
- Implication: io_statx_prep guarantees the op is queued to io-wq, never tried inline.
- Implication: io_statx() body may assume non-NONBLOCK issue context.

REQ-7: Filename lifetime:
- prep: delayed_getname_uflags reserves a `struct filename` token but does not pin user pages.
- issue: filename_complete_delayed CLASS-scoped variable materialises the name; auto-putname on drop.
- cancel: dismiss_delayed_filename releases the reservation without materialise.

REQ-8: REQ_F_NEED_CLEANUP set at prep ⟹ io_statx_cleanup runs on cancellation paths.

REQ-9: sqe->splice_fd_in is rejected at prep — that sqe field belongs to splice ops and must not be aliased.

REQ-10: sqe->buf_index is rejected — statx does not bind a registered buffer; reusing this field would silently drop into another op's registration.

## Acceptance Criteria

- [ ] AC-1: IORING_OP_STATX with valid (dfd, path, statx buffer, mask): CQE result == 0 and userspace statx struct populated.
- [ ] AC-2: AT_FDCWD as dfd + relative path: resolves under cwd.
- [ ] AC-3: AT_SYMLINK_NOFOLLOW: do_statx returns metadata of symlink itself, not target.
- [ ] AC-4: AT_EMPTY_PATH with empty user path: stats the dfd itself (when supported by VFS).
- [ ] AC-5: REQ_F_FIXED_FILE on prep: returns -EBADF.
- [ ] AC-6: sqe->buf_index ≠ 0: prep returns -EINVAL.
- [ ] AC-7: sqe->splice_fd_in ≠ 0: prep returns -EINVAL.
- [ ] AC-8: Prep always sets REQ_F_FORCE_ASYNC and REQ_F_NEED_CLEANUP.
- [ ] AC-9: Cancel before issue: io_statx_cleanup releases delayed filename (no leak).
- [ ] AC-10: do_statx returns -ENOENT for non-existent path: CQE carries -ENOENT, req_set_fail not required (negative result is valid for stat).
- [ ] AC-11: Mask = 0: do_statx still succeeds with minimal kstat fields populated; CQE == 0.
- [ ] AC-12: Bad user buffer pointer at copy_to_user time: CQE == -EFAULT.

## Architecture

```
struct IoStatx {
  file: *File,
  dfd: i32,
  mask: u32,
  flags: u32,             // AT_* flags
  filename: DelayedFilename,
  buffer: *mut Statx,     // user pointer
}
```

`IoUringStatx::statx_prep(req, sqe) -> i32`:
1. if sqe.buf_index != 0 || sqe.splice_fd_in != 0: return -EINVAL.
2. if req.flags.contains(REQ_F_FIXED_FILE): return -EBADF.
3. sx.dfd = sqe.fd as i32.
4. sx.mask = sqe.len.
5. let path = sqe.addr as *const c_char.
6. sx.buffer = sqe.addr2 as *mut Statx.
7. sx.flags = sqe.statx_flags.
8. let ret = delayed_getname_uflags(&mut sx.filename, path, sx.flags); if ret != 0 { return ret; }
9. req.flags |= REQ_F_NEED_CLEANUP.
10. req.flags |= REQ_F_FORCE_ASYNC.
11. return 0.

`IoUringStatx::statx(req, issue_flags) -> IouResult`:
1. debug_assert!(!issue_flags.contains(IO_URING_F_NONBLOCK)).
2. let name = filename_complete_delayed(&mut sx.filename).  // RAII guard; putname on drop
3. let ret = do_statx(sx.dfd, &name, sx.flags, sx.mask, sx.buffer).
4. io_req_set_res(req, ret, 0).
5. return IouResult::Complete.

`IoUringStatx::statx_cleanup(req)`:
1. dismiss_delayed_filename(&mut sx.filename).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `statx_prep_rejects_buf_index` | INVARIANT | per-statx_prep: sqe.buf_index != 0 ⟹ -EINVAL. |
| `statx_prep_rejects_splice_fd_in` | INVARIANT | per-statx_prep: sqe.splice_fd_in != 0 ⟹ -EINVAL. |
| `statx_prep_rejects_fixed_file` | INVARIANT | per-statx_prep: REQ_F_FIXED_FILE ⟹ -EBADF. |
| `statx_prep_sets_force_async` | INVARIANT | per-statx_prep: success ⟹ REQ_F_FORCE_ASYNC set. |
| `statx_prep_sets_need_cleanup` | INVARIANT | per-statx_prep: success ⟹ REQ_F_NEED_CLEANUP set. |
| `statx_issue_never_nonblock` | INVARIANT | per-statx: WARN_ON_ONCE on IO_URING_F_NONBLOCK; do_statx never called in atomic context. |
| `statx_cleanup_dismisses_filename` | INVARIANT | per-statx_cleanup: filename reservation released. |

### Layer 2: TLA+

`io_uring/statx.tla`:
- States: prep_idle → prep_validated → queued → issued → completed | cancelled.
- Per-IORING_OP_STATX traces.
- Properties:
  - `safety_force_async_always` — per-prep: REQ_F_FORCE_ASYNC always set; inline path unreachable.
  - `safety_filename_balanced` — per-trace: every successful prep matches exactly one of {filename_complete_delayed, dismiss_delayed_filename}.
  - `safety_no_fixed_file_path` — per-prep: REQ_F_FIXED_FILE never reaches do_statx.
  - `liveness_statx_completes` — per-issue: every queued STATX terminates with CQE.
  - `liveness_cancelled_statx_releases` — per-cancel: cleanup runs and filename is freed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `statx_prep` post: ret == 0 ⟹ REQ_F_NEED_CLEANUP ∧ REQ_F_FORCE_ASYNC set | `IoUringStatx::statx_prep` |
| `statx_prep` post: sx.dfd / mask / flags / buffer / filename initialized | `IoUringStatx::statx_prep` |
| `statx` pre: !IO_URING_F_NONBLOCK | `IoUringStatx::statx` |
| `statx` post: CQE result ∈ {0, -errno from do_statx} | `IoUringStatx::statx` |
| `statx_cleanup` post: delayed filename reservation == None | `IoUringStatx::statx_cleanup` |

### Layer 4: Verus/Creusot functional

`Per-STATX SQE → delayed_getname_uflags → REQ_F_FORCE_ASYNC enqueue → worker thread → filename_complete_delayed → do_statx(dfd, name, flags, mask, buffer) → CQE` semantic equivalence: per-statx(2) man page and Documentation/io_uring.7 §IORING_OP_STATX. The op result is identical to that of a userspace `statx(dfd, path, flags, mask, &out)` call with the calling task's credentials.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

STATX reinforcement:

- **Per-prep REQ_F_FIXED_FILE rejection** — defense against per-confusion where a ring-fixed dfd is passed in IOSQE_FIXED_FILE meaning.
- **Per-prep buf_index rejection** — defense against per-cross-op-field-aliasing where a registered buffer ID is silently consumed.
- **Per-prep splice_fd_in rejection** — defense against per-cross-op-field-aliasing with splice ops.
- **Per-prep unconditional REQ_F_FORCE_ASYNC** — defense against per-blocking-path on submitter thread (do_statx may sleep on filesystem I/O).
- **Per-issue WARN_ON_ONCE on NONBLOCK** — defense-in-depth assertion that the force-async invariant held.
- **Per-delayed_getname_uflags AT_* validation at prep** — defense against per-late-error where bad AT_* flags only surface in the worker.
- **Per-CLASS-scoped filename_complete_delayed** — defense against per-leak of struct filename on early return from do_statx.
- **Per-REQ_F_NEED_CLEANUP set ⟹ cancel runs dismiss_delayed_filename** — defense against per-leak on cancellation.
- **Per-do_statx credential invariant** — runs under submitter task creds (io-wq inherits), defense against per-privilege-escalation through ring polling.
- **Per-user buffer copy_to_user EFAULT bounded** — defense against per-kernel-OOPS on stat buffer fault.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- VFS `do_statx`, `vfs_statx`, `vfs_getattr` (covered in `fs/stat.md` Tier-3).
- Delayed-filename machinery `delayed_getname_uflags` / `filename_complete_delayed` (covered in `fs/namei.md` Tier-3).
- struct statx UAPI layout (covered in uapi tier).
- io-wq worker selection / scheduling (covered in `io_uring/io-wq.md`).
- Implementation code.
