# Tier-3: io_uring/sync.c — io_uring sync operations (fsync / fdatasync / sync_file_range / fallocate)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/sync.c (~114 lines)
  - io_uring/sync.h
  - include/uapi/linux/io_uring.h (IORING_OP_FSYNC, IORING_OP_SYNC_FILE_RANGE, IORING_OP_FALLOCATE, IORING_FSYNC_DATASYNC)
-->

## Summary

`io_uring/sync.c` implements the async wrappers for the sync/persistence family of file operations as io_uring SQE opcodes: `IORING_OP_FSYNC`, `IORING_OP_SYNC_FILE_RANGE`, and `IORING_OP_FALLOCATE`. All three operations share a single per-request command payload, `struct io_sync` (16 bytes inline in the io_kiocb cmd area), holding `file`, `off`, `len`, `flags`, and (for fallocate) `mode`. Each opcode follows the standard io_uring two-phase split: a `*_prep` validator that reads the SQE fields under READ_ONCE, rejects invalid sqe field combinations, and marks the request `REQ_F_FORCE_ASYNC` because every sync-family op blocks; and an issue handler that runs in worker context, delegates to the VFS primitive (`vfs_fsync_range`, `sync_file_range`, `vfs_fallocate`), and completes via `io_req_set_res` + `IOU_COMPLETE`. The DATASYNC variant of `IORING_OP_FSYNC` is requested via `IORING_FSYNC_DATASYNC` in `sqe->fsync_flags`, which `vfs_fsync_range` translates into a metadata-skipping fdatasync. `sync_file_range` consumes `sqe->sync_range_flags` (the SYNC_FILE_RANGE_WAIT_BEFORE / WRITE / WAIT_AFTER bitmask) and uses `off` + `len` to bound the range. `fallocate` aliases SQE fields unconventionally: `off` is the offset, `addr` is the length, and `len` is the mode (`FALLOC_FL_*`). Critical for: async durable write barriers (databases, journaled writers), bulk preallocation, hole-punching, and POSIX_FADV-like sparse layouts — without blocking the submission thread.

This Tier-3 covers `io_uring/sync.c` (~114 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_sync` | per-op cmd payload | `IoSync` |
| `io_sfr_prep()` | per-SQE prep for sync_file_range | `IoSync::sfr_prep` |
| `io_sync_file_range()` | per-issue: sync_file_range | `IoSync::sync_file_range` |
| `io_fsync_prep()` | per-SQE prep for FSYNC / FDATASYNC | `IoSync::fsync_prep` |
| `io_fsync()` | per-issue: vfs_fsync_range | `IoSync::fsync` |
| `io_fallocate_prep()` | per-SQE prep for FALLOCATE | `IoSync::fallocate_prep` |
| `io_fallocate()` | per-issue: vfs_fallocate | `IoSync::fallocate` |
| `IORING_OP_FSYNC` | per-opcode | shared UAPI |
| `IORING_OP_SYNC_FILE_RANGE` | per-opcode | shared UAPI |
| `IORING_OP_FALLOCATE` | per-opcode | shared UAPI |
| `IORING_FSYNC_DATASYNC` | per-flag | shared UAPI |
| `REQ_F_FORCE_ASYNC` | per-req: forces io-wq dispatch | shared req-flag |
| `IO_URING_F_NONBLOCK` | per-issue flag (must be cleared) | shared issue-flag |
| `IOU_COMPLETE` | per-issue: completion sentinel | shared issue-ret |
| `io_req_set_res()` | per-req: stamps result + cflags | shared req helper |
| `vfs_fsync_range()` | per-VFS-call: fsync subset | extern from `vfs/fsync.md` |
| `sync_file_range()` | per-VFS-call: writeback range | extern from `vfs/sync_file_range.md` |
| `vfs_fallocate()` | per-VFS-call: fs-level fallocate | extern from `vfs/fallocate.md` |
| `fsnotify_modify()` | per-fallocate notify | extern from `fsnotify/00.md` |

## Compatibility contract

REQ-1: `struct io_sync` is the per-request command payload, inline in the io_kiocb cmd area (16-byte slot via `io_kiocb_to_cmd`):
- `file: *struct file` — per-req target file (resolved by the io_uring core from `sqe->fd`).
- `len: loff_t` — per-op length (bytes for sync_file_range / fsync range end; bytes for fallocate range).
- `off: loff_t` — per-op offset.
- `flags: i32` — per-op flags (sync_range_flags / fsync_flags).
- `mode: i32` — per-fallocate FALLOC_FL_* mode.

REQ-2: `io_sfr_prep(req, sqe)` — per-SYNC_FILE_RANGE prep:
- /* Reject invalid sqe combinations */
- if `sqe->addr` ∨ `sqe->buf_index` ∨ `sqe->splice_fd_in`: return -EINVAL.
- `sync->off = READ_ONCE(sqe->off)`.
- `sync->len = READ_ONCE(sqe->len)`.
- `sync->flags = READ_ONCE(sqe->sync_range_flags)`.
- /* Always blocks — force io-wq */
- `req->flags |= REQ_F_FORCE_ASYNC`.
- return 0.

REQ-3: `io_sync_file_range(req, issue_flags)` — per-SYNC_FILE_RANGE issue:
- /* Must run in blocking context */
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- `ret = sync_file_range(req->file, sync->off, sync->len, sync->flags)`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-4: `io_fsync_prep(req, sqe)` — per-FSYNC / FDATASYNC prep:
- /* Reject invalid sqe combinations */
- if `sqe->addr` ∨ `sqe->buf_index` ∨ `sqe->splice_fd_in`: return -EINVAL.
- `sync->flags = READ_ONCE(sqe->fsync_flags)`.
- /* Reject unknown fsync flags (only DATASYNC) */
- if `sync->flags & ~IORING_FSYNC_DATASYNC`: return -EINVAL.
- `sync->off = READ_ONCE(sqe->off)`.
- /* Negative offset invalid */
- if `sync->off < 0`: return -EINVAL.
- `sync->len = READ_ONCE(sqe->len)`.
- `req->flags |= REQ_F_FORCE_ASYNC`.
- return 0.

REQ-5: `io_fsync(req, issue_flags)` — per-FSYNC / FDATASYNC issue:
- /* Must run in blocking context */
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- `end = sync->off + sync->len`.
- /* Zero-length means full file: end is LLONG_MAX */
- `ret = vfs_fsync_range(req->file, sync->off, end > 0 ? end : LLONG_MAX, sync->flags & IORING_FSYNC_DATASYNC)`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-6: `io_fallocate_prep(req, sqe)` — per-FALLOCATE prep:
- /* Reject invalid sqe combinations */
- if `sqe->buf_index` ∨ `sqe->rw_flags` ∨ `sqe->splice_fd_in`: return -EINVAL.
- /* SQE field aliasing for FALLOCATE: */
- /*   off  = sqe->off    (file offset)        */
- /*   len  = sqe->addr   (range length, bytes) */
- /*   mode = sqe->len    (FALLOC_FL_* mode)   */
- `sync->off = READ_ONCE(sqe->off)`.
- `sync->len = READ_ONCE(sqe->addr)`.
- `sync->mode = READ_ONCE(sqe->len)`.
- `req->flags |= REQ_F_FORCE_ASYNC`.
- return 0.

REQ-7: `io_fallocate(req, issue_flags)` — per-FALLOCATE issue:
- /* Must run in blocking context */
- `WARN_ON_ONCE(issue_flags & IO_URING_F_NONBLOCK)`.
- `ret = vfs_fallocate(req->file, sync->mode, sync->off, sync->len)`.
- /* On success, fire fsnotify modify */
- if `ret >= 0`: `fsnotify_modify(req->file)`.
- `io_req_set_res(req, ret, 0)`.
- return `IOU_COMPLETE`.

REQ-8: `IORING_FSYNC_DATASYNC` (UAPI flag, bit 0 of `sqe->fsync_flags`):
- If set, `vfs_fsync_range` is called with `datasync=1` ⟹ filesystem skips inode metadata flush (mtime/ctime/size unchanged-bits).
- If clear, full fsync.

REQ-9: `sync_range_flags` (`sqe->sync_range_flags`) — UAPI mirror of `sync_file_range(2)` flags:
- `SYNC_FILE_RANGE_WAIT_BEFORE`: wait on in-flight before initiating.
- `SYNC_FILE_RANGE_WRITE`: initiate writeback for dirty pages.
- `SYNC_FILE_RANGE_WAIT_AFTER`: wait on initiated writes.
- `SYNC_FILE_RANGE_WRITE_AND_WAIT`: convenience composite (BEFORE | WRITE | AFTER).

REQ-10: `REQ_F_FORCE_ASYNC` is set in all three preps:
- Per-io_uring: req is never issued inline; always dispatched to io-wq.
- Per-issue: `IO_URING_F_NONBLOCK` is guaranteed cleared (WARN_ON_ONCE otherwise).

REQ-11: Per-fallocate SQE field aliasing is permanent UAPI:
- An IORING_OP_FALLOCATE SQE that conforms to the literal SQE layout uses `addr` for `len` and `len` for `mode`. The op is named-correctly from the issuer's perspective (offset, len, mode) but the SQE field names mismatch.

REQ-12: Per-completion semantics:
- `vfs_*` return is propagated raw via `io_req_set_res(req, ret, 0)`; second arg is `cflags`, always 0 for sync-family.
- Negative return ⟹ error reported via CQE `res`; positive/zero ⟹ success.

REQ-13: Per-no-cleanup-handler:
- None of fsync / sync_file_range / fallocate sets `REQ_F_NEED_CLEANUP` — the io_sync payload owns no allocated resources.

REQ-14: Per-fsnotify on fallocate only:
- `vfs_fsync_range` and `sync_file_range` do not fsnotify (no content change visible beyond what writeback was already going to do).
- `vfs_fallocate` (FALLOC_FL_*) can change file size / allocate blocks / punch holes — counts as modify.

REQ-15: Per-off+len overflow on fsync:
- `end = off + len` — if `off + len` overflows or sums to <= 0, fall back to `LLONG_MAX` (full-file fsync from off to EOF).

## Acceptance Criteria

- [ ] AC-1: IORING_OP_FSYNC with `fsync_flags=0`: vfs_fsync_range called with datasync=0; full fsync.
- [ ] AC-2: IORING_OP_FSYNC with `fsync_flags=IORING_FSYNC_DATASYNC`: vfs_fsync_range called with datasync=1; metadata not flushed.
- [ ] AC-3: IORING_OP_FSYNC with `len=0`: full-file fsync from `off` to LLONG_MAX.
- [ ] AC-4: IORING_OP_FSYNC with negative `off`: -EINVAL at prep.
- [ ] AC-5: IORING_OP_FSYNC with unknown `fsync_flags` bits: -EINVAL at prep.
- [ ] AC-6: IORING_OP_SYNC_FILE_RANGE: sync_file_range called with off/len/sync_range_flags.
- [ ] AC-7: IORING_OP_SYNC_FILE_RANGE/FSYNC with sqe->addr or buf_index or splice_fd_in set: -EINVAL.
- [ ] AC-8: IORING_OP_FALLOCATE: vfs_fallocate(file, mode, off, len) called.
- [ ] AC-9: IORING_OP_FALLOCATE: on ret >= 0, fsnotify_modify fires; on ret < 0, no fsnotify.
- [ ] AC-10: IORING_OP_FALLOCATE SQE aliasing: `addr` → len, `len` → mode preserved.
- [ ] AC-11: All three ops set REQ_F_FORCE_ASYNC during prep.
- [ ] AC-12: All three issue handlers WARN if invoked with IO_URING_F_NONBLOCK.
- [ ] AC-13: All three complete via io_req_set_res(req, vfs-ret, 0) + IOU_COMPLETE.

## Architecture

```
struct IoSync {
  file: *File,        // resolved by io_uring core from sqe->fd
  len: i64,           // loff_t
  off: i64,           // loff_t
  flags: i32,         // sync_range_flags | fsync_flags
  mode: i32,          // FALLOC_FL_* (fallocate only)
}
```

`IoSync::sfr_prep(req, sqe) -> Result<(), Errno>`:
1. /* SQE-field validation */
2. if `sqe.addr != 0 || sqe.buf_index != 0 || sqe.splice_fd_in != 0`: return Err(EINVAL).
3. let sync = io_kiocb_to_cmd::<IoSync>(req).
4. sync.off = read_once(&sqe.off).
5. sync.len = read_once(&sqe.len).
6. sync.flags = read_once(&sqe.sync_range_flags).
7. req.flags |= REQ_F_FORCE_ASYNC.
8. Ok(()).

`IoSync::sync_file_range(req, issue_flags) -> IouRet`:
1. warn_on_once(issue_flags & IO_URING_F_NONBLOCK).
2. let sync = io_kiocb_to_cmd::<IoSync>(req).
3. let ret = vfs::sync_file_range(req.file, sync.off, sync.len, sync.flags).
4. io_req_set_res(req, ret, 0).
5. IouRet::Complete.

`IoSync::fsync_prep(req, sqe) -> Result<(), Errno>`:
1. if `sqe.addr != 0 || sqe.buf_index != 0 || sqe.splice_fd_in != 0`: return Err(EINVAL).
2. let sync = io_kiocb_to_cmd::<IoSync>(req).
3. sync.flags = read_once(&sqe.fsync_flags).
4. if sync.flags & !IORING_FSYNC_DATASYNC != 0: return Err(EINVAL).
5. sync.off = read_once(&sqe.off).
6. if sync.off < 0: return Err(EINVAL).
7. sync.len = read_once(&sqe.len).
8. req.flags |= REQ_F_FORCE_ASYNC.
9. Ok(()).

`IoSync::fsync(req, issue_flags) -> IouRet`:
1. warn_on_once(issue_flags & IO_URING_F_NONBLOCK).
2. let sync = io_kiocb_to_cmd::<IoSync>(req).
3. let end = sync.off.saturating_add(sync.len).
4. let bounded_end = if end > 0 { end } else { i64::MAX }.
5. let datasync = (sync.flags & IORING_FSYNC_DATASYNC) != 0.
6. let ret = vfs::fsync_range(req.file, sync.off, bounded_end, datasync).
7. io_req_set_res(req, ret, 0).
8. IouRet::Complete.

`IoSync::fallocate_prep(req, sqe) -> Result<(), Errno>`:
1. if `sqe.buf_index != 0 || sqe.rw_flags != 0 || sqe.splice_fd_in != 0`: return Err(EINVAL).
2. let sync = io_kiocb_to_cmd::<IoSync>(req).
3. /* SQE-field aliasing: addr=len, len=mode */
4. sync.off  = read_once(&sqe.off).
5. sync.len  = read_once(&sqe.addr) as i64.
6. sync.mode = read_once(&sqe.len) as i32.
7. req.flags |= REQ_F_FORCE_ASYNC.
8. Ok(()).

`IoSync::fallocate(req, issue_flags) -> IouRet`:
1. warn_on_once(issue_flags & IO_URING_F_NONBLOCK).
2. let sync = io_kiocb_to_cmd::<IoSync>(req).
3. let ret = vfs::fallocate(req.file, sync.mode, sync.off, sync.len).
4. if ret >= 0: fsnotify::modify(req.file).
5. io_req_set_res(req, ret, 0).
6. IouRet::Complete.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prep_rejects_addr` | INVARIANT | per-sfr_prep/fsync_prep: nonzero sqe.addr ⟹ -EINVAL. |
| `prep_rejects_buf_index` | INVARIANT | per-all-prep: nonzero sqe.buf_index ⟹ -EINVAL. |
| `prep_rejects_splice_fd_in` | INVARIANT | per-all-prep: nonzero sqe.splice_fd_in ⟹ -EINVAL. |
| `fsync_prep_rejects_unknown_flags` | INVARIANT | per-fsync_prep: any bit outside IORING_FSYNC_DATASYNC ⟹ -EINVAL. |
| `fsync_prep_rejects_neg_off` | INVARIANT | per-fsync_prep: sync.off < 0 ⟹ -EINVAL. |
| `prep_sets_force_async` | INVARIANT | per-all-prep: REQ_F_FORCE_ASYNC set on success. |
| `issue_in_blocking_context` | INVARIANT | per-all-issue: IO_URING_F_NONBLOCK never observed. |
| `fsync_end_no_overflow` | INVARIANT | per-fsync: end<=0 ⟹ LLONG_MAX used. |
| `fallocate_fsnotify_iff_ok` | INVARIANT | per-fallocate: fsnotify_modify fires ⟺ ret >= 0. |

### Layer 2: TLA+

`io_uring/sync.tla`:
- Per-prep + per-issue states for each of {FSYNC, SYNC_FILE_RANGE, FALLOCATE}.
- Properties:
  - `safety_no_nonblock_issue` — per-issue: IO_URING_F_NONBLOCK ⟹ WARN.
  - `safety_datasync_flag_propagates` — per-fsync: IORING_FSYNC_DATASYNC in sqe ⟹ vfs_fsync_range datasync arg true.
  - `safety_fallocate_aliasing_stable` — per-fallocate prep: sqe.addr→len, sqe.len→mode preserved across rounds.
  - `liveness_each_issue_completes` — per-issue: terminates with IOU_COMPLETE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IoSync::sfr_prep` post: sync.{off,len,flags} = sqe.{off,len,sync_range_flags} | `IoSync::sfr_prep` |
| `IoSync::fsync_prep` post: sync.flags & !IORING_FSYNC_DATASYNC == 0 ∧ sync.off >= 0 | `IoSync::fsync_prep` |
| `IoSync::fsync` post: vfs_fsync_range called with (off, end>0?end:LLONG_MAX, datasync) | `IoSync::fsync` |
| `IoSync::fallocate_prep` post: sync.{off,len,mode} = sqe.{off,addr,len} | `IoSync::fallocate_prep` |
| `IoSync::fallocate` post: ret >= 0 ⟹ fsnotify_modify(file) called | `IoSync::fallocate` |
| All issue post: REQ_F_FORCE_ASYNC ⟹ issue ran in worker | shared |

### Layer 4: Verus/Creusot functional

`Per-SQE submit → prep validates → REQ_F_FORCE_ASYNC ⟹ worker dispatch → vfs_*range call → io_req_set_res ⟹ CQE` semantic equivalence: per-`io_uring(7)` and `io_uring_enter(2)` man pages, per-`fsync(2)` / `sync_file_range(2)` / `fallocate(2)`.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

Sync-family reinforcement:

- **Per-sqe.{addr,buf_index,splice_fd_in} reserved-fields rejected** — defense against per-SQE field-aliasing exploits.
- **Per-fsync_flags unknown bits rejected** — defense against per-forward-flag UAPI drift.
- **Per-fsync sync.off < 0 rejected** — defense against per-negative-offset VFS pre-check bypass.
- **Per-REQ_F_FORCE_ASYNC enforced** — defense against per-blocking-in-submit-path stall.
- **Per-WARN_ON_ONCE on IO_URING_F_NONBLOCK** — defense against per-misdispatch (sync ops in submit thread).
- **Per-fsync end overflow falls back to LLONG_MAX** — defense against per-arithmetic-overflow UB.
- **Per-fallocate fsnotify_modify gated on ret >= 0** — defense against per-spurious-notify on EFBIG/ENOSPC.
- **Per-fallocate SQE field aliasing locked-in** — defense against per-UAPI drift breaking existing apps.
- **Per-vfs_*range delegation: no custom locking in io_uring layer** — defense against per-double-lock vs VFS.
- **Per-IORING_FSYNC_DATASYNC single defined bit** — defense against per-future-flag silently-ignored.
- **Per-issue handler stateless after vfs return** — defense against per-late-cancel race.
- **Per-no REQ_F_NEED_CLEANUP** — defense against per-spurious-cleanup-path bug.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `vfs_fsync_range` / `sync_file_range` / `vfs_fallocate` VFS implementations (covered in `vfs/fsync.md`, `vfs/sync_file_range.md`, `vfs/fallocate.md` Tier-3)
- io_uring core SQE→io_kiocb dispatch (covered in `io_uring/io_uring-core.md` Tier-3)
- io-wq worker pool (covered in `io_uring/io-wq.md` Tier-3 when expanded)
- Filesystem-level fallocate dispatch (`->fallocate` op, ext4_fallocate, etc.) (covered per-fs Tier-3)
- fsnotify dispatch (covered in `fsnotify/00-overview.md` Tier-3)
- Implementation code
