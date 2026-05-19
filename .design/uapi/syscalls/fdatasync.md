# Tier-5 syscall: fdatasync(2) — syscall 75

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/sync.c (SYSCALL_DEFINE1(fdatasync), do_fsync(... datasync=1))
  - include/linux/fs.h (file_operations::fsync with datasync flag)
  - arch/x86/entry/syscalls/syscall_64.tbl (75  common  fdatasync)
-->

## Summary

`fdatasync(2)` is the "data + retrieval-metadata only" variant of `fsync(2)`. It flushes the file's modified data pages and the strict subset of metadata required to retrieve those data on a subsequent read after a crash (notably i_size for new allocations and block-map entries), but it does NOT necessarily commit metadata-only changes (mtime, ctime, atime, mode, ownership).

For a database that writes redo-log records into a pre-extended (already at full i_size, already allocated) file, fdatasync is dramatically faster than fsync because no journal commit is required — only data-pages and the device cache flush. Critical for: PostgreSQL/MySQL wal/log durability, log-structured stores, hot-path append-only journals.

## Signature

```c
int fdatasync(int fd);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor; any access mode acceptable (even O_RDONLY for the data-already-clean fast path). |

## Return value

| Value | Meaning |
|---|---|
| `0` | Data + retrieval-metadata durable. |
| `-1` + `errno` | Failure; durability NOT guaranteed. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` invalid. |
| `EIO` | Writeback / barrier failed. |
| `EINVAL` | `fd` not sync-capable (no `->fsync`). |
| `EROFS` | Read-only mount + pending allocation. |
| `ENOSPC` | Delayed-allocation block reservation failed. |

## ABI surface

```text
__NR_fdatasync  (x86_64) = 75
__NR_fdatasync  (arm64)  = 83
__NR_fdatasync  (riscv)  = 83
__NR_fdatasync  (i386)   = 148
```

## Compatibility contract

REQ-1: Syscall number is **75** on x86_64. ABI-stable.

REQ-2: Per-POSIX: data sync sufficient to retrieve the file's data after a crash. Metadata-only fields (mtime, ctime, mode, perm bits) need NOT be durable on return.

REQ-3: Internally implemented via `vfs_fsync_range(file, 0, LLONG_MAX, datasync=1)`. The `datasync=1` flag is passed to `file->f_op->fsync`; the filesystem MAY skip metadata commit if no data-retrieval-relevant metadata is dirty.

REQ-4: Per-i_size growth: if the file size increased since the last commit (i.e., new data extends EOF), the filesystem MUST commit i_size — otherwise the data is unreachable on recovery.

REQ-5: Per-block-map / extent allocation: any newly allocated extents covering the file MUST be committed; the data is otherwise unreachable.

REQ-6: Per-overwrite-in-place: writing into already-allocated blocks of an already-EOF-stable file: fdatasync commits ONLY the data pages and the device cache flush; the filesystem journal is NOT touched (ext4 with `data=ordered` for in-place overwrite).

REQ-7: Per-mtime/ctime not flushed: the inode timestamps may differ between in-memory and on-disk after fdatasync; subsequent power loss may roll back the timestamps but NOT the data. Applications that depend on timestamp durability must use fsync(2).

REQ-8: Per-block-device fd: blkdev_fsync(datasync=1) is equivalent to blkdev_fsync(datasync=0) — both issue blkdev_issue_flush.

REQ-9: Per-errseq_t: same per-file error reporting as fsync(2); EIO reported once per file struct per error generation.

REQ-10: Per-FUSE: forwards `fuse_fsync_in.fsync_flags |= FUSE_FSYNC_FDATASYNC`; userspace daemon distinguishes.

REQ-11: Per-OPEN_FLAGS_NEEDS_SYNC (O_DSYNC): every write(2) implies an internal fdatasync; explicit fdatasync remains useful for groups of writes.

REQ-12: Per-XFS: `xfs_file_fsync` checks `XFS_IFLAG_NEEDS_FSYNC` — if only timestamps are dirty AND datasync=1, the journal commit is skipped.

REQ-13: Per-ext4 with `data=journal`: every write is journaled; fdatasync still issues the barrier but the journal commit may already be done.

REQ-14: Per-btrfs: fdatasync uses the dedicated log-tree (`fs/btrfs/tree-log.c`); only the log-tree commit + barrier are required (no full transaction).

REQ-15: Per-cgroup-writeback: same memcg attribution as fsync.

## Acceptance Criteria

- [ ] AC-1: write into a pre-extended file; fdatasync(fd) returns 0; data recoverable after power-loss.
- [ ] AC-2: append write (extending i_size); fdatasync commits new i_size; data recoverable.
- [ ] AC-3: fdatasync(-1) returns -EBADF.
- [ ] AC-4: fdatasync on a pipe returns -EINVAL.
- [ ] AC-5: utimensat alone + fdatasync: timestamps may not be on disk; data path unaffected.
- [ ] AC-6: fdatasync benchmark < fsync benchmark for in-place overwrite workload (XFS, ext4, btrfs).
- [ ] AC-7: O_DSYNC open: write returns only after data + retrieval-metadata durable; explicit fdatasync is no-op-but-correct.
- [ ] AC-8: ENOSPC during delayed-alloc resolution surfaces.
- [ ] AC-9: errseq EIO surfaced once.
- [ ] AC-10: FUSE FUSE_FSYNC_FDATASYNC bit propagated.

## Architecture

```rust
#[syscall(nr = 75, abi = "sysv")]
pub fn sys_fdatasync(fd: i32) -> isize {
    Sync::do_fsync(fd, /* datasync */ true)
}
```

`Sync::do_fsync(fd, datasync) -> isize`:
1. let file = fdget(fd).ok_or(EBADF)?;
2. let r = Sync::vfs_fsync_range(&file, 0, LLONG_MAX, datasync);
3. fdput(file);
4. r

`Sync::vfs_fsync_range(file, start, end, datasync) -> isize`:
1. let op = file.f_op.fsync.ok_or(EINVAL)?;
2. let err_pre = errseq_check(&file.f_wb_err, file.f_wb_err_seen);
3. let r = op(file, start, end, datasync as i32);
4. let err_post = errseq_check_and_advance(&file.f_wb_err, &mut file.f_wb_err_seen);
5. if r == 0 && err_post != 0 { return err_post; }
6. r

`Xfs::file_fsync(file, start, end, datasync) -> isize`:
1. let ip = XFS_I(file.f_inode());
2. /* Flush data pages */
3. let r = filemap_write_and_wait_range(file.f_mapping, start, end);
4. if r != 0 { return r; }
5. /* Datasync: skip journal commit if only timestamps dirty */
6. if datasync && (ip.i_flags & XFS_IFLAG_NEEDS_FSYNC == 0) {
7.   return blkdev_issue_flush(ip.i_bdev);
8. }
9. /* Else: force log to current LSN */
10. let lsn = xfs_log_force_inode(ip);
11. xfs_log_force_lsn(ip.i_mount, lsn, XFS_LOG_SYNC);
12. blkdev_issue_flush(ip.i_bdev)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `datasync_flag_propagated` | INVARIANT | fdatasync passes datasync=1 into f_op->fsync. |
| `isize_growth_implies_metadata_commit` | INVARIANT | if i_size changed, metadata-commit path taken. |
| `barrier_after_data` | ORDER | data writeback drains BEFORE block-device flush. |

### Layer 2: TLA+

`fs/fdatasync.tla`:
- States: per-write, per-fdatasync, per-data-flush, per-meta-cond-commit, per-barrier.
- Properties:
  - `safety_data_recoverable` — power loss after fdatasync recovers data.
  - `safety_no_unnecessary_meta_commit` — overwrite-in-place skips meta commit.
  - `liveness_terminates` — every fdatasync returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fsync(datasync=true)` post: data durable; meta-not-required | `Sync::do_fsync` |
| `xfs_file_fsync(datasync=1)` post: skips log force if NEEDS_FSYNC clear | `Xfs::file_fsync` |

### Layer 4: Verus / Creusot functional

Per-`fdatasync(2)` man-page, per-POSIX.1-2017 Synchronized I/O Data Integrity Completion, xfstests generic/311.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fdatasync(2)` reinforcement:

- **Per-fd validation pre-I/O** — defense against per-bad-fd kernel deref.
- **Per-i_size-change forces metadata commit** — defense against per-data-island-unreachable bug.
- **Per-errseq strict** — defense against per-silent-data-loss.
- **Per-block-device flush mandatory** — defense against per-volatile-cache lie.
- **Per-FUSE flag propagation** — defense against per-daemon ambiguity.

## Grsecurity / PaX surface

- **PaX UDEREF on fd lookup** — defense against per-fd-table TOCTOU.
- **GRKERNSEC_DMESG** — fdatasync error klog rate-limited and CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM in fdatasync klog** — block-device pointer scrubbed.
- **PAX_REFCOUNT on file refcount during fdatasync** — defense against per-file UAF if close races writeback.
- **PaX KERNEXEC on f_op->fsync dispatch** — indirect call hardened via Function CFI.
- **Sync data-leak prevention** — per-file extent layout not leaked via fdatasync error path; klog filtered.
- **CAP_DAC_OVERRIDE strict** — same open-time DAC gate as fsync; grsec does not loosen.
- **GRKERNSEC_CHROOT_FCHDIR adjacency** — directory-fd fdatasync inside chroot restricted to chroot subtree (no upward escape).
- **Per-FUSE daemon timeout enforced** — defense against per-userspace-hang.
- **PAX_USERCOPY_HARDEN on klog buffer** — defense against per-klog-copy overflow.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-filesystem datasync internals (covered in Tier-3 fs/<fs>-fsync.md).
- Block-device flush plumbing (covered in Tier-3 block/blk-flush.md).
- O_DSYNC / O_SYNC open-flag semantics (covered in Tier-5 open.md).
- Implementation code.
