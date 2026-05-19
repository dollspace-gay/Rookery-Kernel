---
title: "Tier-5 syscall: fsync(2) — syscall 74"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fsync(2)` flushes all modified in-core data of a file referenced by `fd` to the underlying durable storage, then issues a write-barrier / cache-flush such that the file's data and metadata are recoverable across a power loss or kernel crash. It synchronously calls the filesystem's `->fsync` method, which (for most journaling filesystems) writes the file's dirty page-cache, then commits the on-disk inode (including i_size, mtime, ctime, block-map), then flushes the underlying block device write cache (via `REQ_PREFLUSH` / `REQ_FUA`).

fsync(2) is the foundational durability primitive for databases, mailers, journaled application state, package managers, and any program whose correctness depends on "this byte is on disk". Critical for: POSIX durability guarantee, atomic rename-then-fsync pattern, write-ahead-log integrity.

### Acceptance Criteria

- [ ] AC-1: write(fd, ...); fsync(fd) returns 0; pulling power immediately preserves the data on remount.
- [ ] AC-2: fsync(-1) returns -EBADF.
- [ ] AC-3: fsync on a pipe returns -EINVAL.
- [ ] AC-4: fsync on a block-device fd issues blkdev_issue_flush.
- [ ] AC-5: Underlying disk reports write error: first fsync returns -EIO, second returns 0 (errseq_t).
- [ ] AC-6: O_RDONLY fd: fsync succeeds (inode-only sync).
- [ ] AC-7: Read-only mount: -EROFS if metadata dirty, else 0.
- [ ] AC-8: Directory fd fsync after rename commits the rename atomically.
- [ ] AC-9: Filesystem shutdown: fsync returns -EIO without queueing.
- [ ] AC-10: FUSE fsync forwards to userspace; userspace ENOSYS maps to success (per kernel policy).

### Architecture

```rust
#[syscall(nr = 74, abi = "sysv")]
pub fn sys_fsync(fd: i32) -> isize {
    Sync::do_fsync(fd, /* datasync */ false)
}
```

`Sync::do_fsync(fd, datasync) -> isize`:
1. let file = fdget(fd).ok_or(EBADF)?;
2. /* Filesystems pin against unmount via file->f_path */
3. let r = Sync::vfs_fsync_range(&file, 0, LLONG_MAX, datasync);
4. fdput(file);
5. r

`Sync::vfs_fsync_range(file, start, end, datasync) -> isize`:
1. let inode = file.f_inode();
2. /* Filesystem must implement ->fsync */
3. let op = file.f_op.fsync.ok_or(EINVAL)?;
4. /* errseq_t pre-sample */
5. let err_pre = errseq_check(&file.f_wb_err, file.f_wb_err_seen);
6. /* Dispatch */
7. let r = op(file, start, end, datasync as i32);
8. /* errseq_t post-sample */
9. let err_post = errseq_check_and_advance(&file.f_wb_err, &mut file.f_wb_err_seen);
10. if r == 0 && err_post != 0 { return err_post; }
11. r

`Sync::blkdev_fsync(file, ...) -> isize`:
1. let bdev = file.f_inode().i_bdev();
2. let r = blkdev_issue_flush(bdev);
3. if r == -EOPNOTSUPP { return 0; }  /* device has no cache */
4. r

### Out of Scope

- Per-filesystem ->fsync internals (covered in Tier-3 fs/ext4-fsync.md, fs/xfs-fsync.md, fs/btrfs-fsync.md).
- Block-device queue flush implementation (covered in Tier-3 block/blk-flush.md).
- io_uring async fsync (covered in Tier-5 io_uring_enter.md).
- Implementation code.

### signature

```c
int fsync(int fd);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor (any access mode that permits the underlying inode to be sync'd; even `O_RDONLY` is accepted for inode-only sync). |

### return value

| Value | Meaning |
|---|---|
| `0` | All data + metadata for the file backing `fd` are durable. |
| `-1` + `errno` | Failure; durability NOT guaranteed; caller MUST retry or treat the file as suspect. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not a valid open file descriptor. |
| `EIO` | Lower-level I/O error during writeback or barrier. Per POSIX semantics the kernel clears the per-file dirty-error state on read; once cleared a successful fsync does NOT prove pre-error data is durable (see "errseq_t" history). |
| `EINVAL` | `fd` refers to an object that does not support sync (e.g. some sockets, pipes, anon_inode without `->fsync`). |
| `EROFS` | Underlying filesystem is read-only and an inode update was pending. |
| `ENOSPC` | Delayed allocation could not be resolved during fsync (block reservation failure). |
| `EDQUOT` | Quota exceeded during writeback. |

### abi surface

```text
__NR_fsync  (x86_64) = 74
__NR_fsync  (arm64)  = 82
__NR_fsync  (riscv)  = 82
__NR_fsync  (i386)   = 118
```

### compatibility contract

REQ-1: Syscall number is **74** on x86_64. ABI-stable since 0.99.

REQ-2: Per-POSIX: on successful return, all data written to the file via prior write(2)/pwrite(2)/writev(2)/mmap(2) (MS_SYNC equivalent) AND the file's metadata necessary to retrieve the data (block-map, i_size) are durable.

REQ-3: Per-Linux, fsync(2) is a full file sync: data + ALL metadata (not just data-retrieval metadata). For data-only-plus-size-fast-path, use `fdatasync(2)` (syscall 75).

REQ-4: Per-`file->f_op->fsync(file, start, end, datasync)`: dispatched with `start=0`, `end=LLONG_MAX`, `datasync=0`. Filesystem must:
- Write back the dirty page-cache range (filemap_write_and_wait_range).
- Commit on-disk inode (journal commit or sync metadata write).
- Issue block-device cache flush (blkdev_issue_flush) unless `wbc->sync_mode == WB_SYNC_NONE`.

REQ-5: Per-errseq_t: each file struct samples a sequence; fsync reports an EIO once per file struct per error generation. Reopening clears the cursor and may miss a prior error.

REQ-6: Per-`O_SYNC` open: every write(2) is effectively an fsync — but explicit fsync(2) is still required for crash-consistency across multi-write atomic operations.

REQ-7: Per-non-regular files:
- Socket / pipe / FIFO: -EINVAL (no `->fsync`).
- Block device fd: flushes block-device cache (calls `blkdev_fsync`).
- Character device: device-driver-defined; typically -EINVAL.
- Directory fd: per-directory `->fsync` (ext4, xfs, btrfs) commits directory entries; required for atomic-rename durability.

REQ-8: Per-`MS_SYNCHRONOUS` mount or `chattr +S`: writes already sync; fsync remains a no-op after writeback completes but still issues the barrier.

REQ-9: Per-shutdown-mode: if the filesystem is in `shutdown` state, fsync returns -EIO immediately.

REQ-10: Per-PageWriteback wait: filemap_fdatawait_range waits for all in-flight writeback before returning; cannot complete until disk-queue drains.

REQ-11: fsync(2) is interruptible at writeback-wait points only in specific filesystems (xfs); generally NOT interruptible by signals (no -EINTR per Linux semantics for the writeback path).

REQ-12: Per-FUSE: forwards to userspace `fuse_fsync_in`; the userspace daemon's response determines success. Slow daemons impose unbounded latency.

REQ-13: Per-cgroup writeback: fsync attributes I/O to the issuing memcg via `wbc->memcg_css`, enabling proportional bandwidth control.

REQ-14: Per-O_DIRECT: data already bypasses the page cache, but fsync still issues the device flush (REQ_PREFLUSH) and any metadata commit.

REQ-15: Per-AIO (io_uring with IORING_OP_FSYNC): same semantics; fully asynchronous variant.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_validated_before_use` | INVARIANT | EBADF returned before any I/O begins. |
| `errseq_advances_once` | INVARIANT | per-file errseq cursor advances on each fsync; EIO surfaced exactly once. |
| `no_op_if_no_fsync` | INVARIANT | f_op->fsync == NULL ⟹ EINVAL. |
| `barrier_after_data` | ORDER | data writeback completes BEFORE blkdev_issue_flush. |

### Layer 2: TLA+

`fs/fsync.tla`:
- States: per-write, per-fsync-call, per-writeback-complete, per-barrier-complete.
- Properties:
  - `safety_data_persistent_after_fsync` — pulling power post-fsync recovers all data.
  - `safety_errseq_monotone` — error cursor never regresses.
  - `liveness_fsync_terminates` — every fsync returns under healthy device.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fsync` post: 0 ⟹ writeback drained ∧ barrier issued | `Sync::do_fsync` |
| `vfs_fsync_range` post: errseq sampled both sides | `Sync::vfs_fsync_range` |
| `blkdev_fsync` post: EOPNOTSUPP folded to 0 | `Sync::blkdev_fsync` |

### Layer 4: Verus / Creusot functional

Per-`fsync(2)` man-page, per-POSIX.1-2017 4.18 Synchronized I/O, per-LWN errseq_t articles, per-XFS / ext4 / btrfs fsync test suites (xfstests generic/311, generic/388, generic/455).

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fsync(2)` reinforcement:

- **Per-fd validation pre-I/O** — defense against per-bad-fd kernel-pointer deref.
- **Per-errseq strict semantics** — defense against per-silent-data-loss after EIO swallow.
- **Per-blkdev_issue_flush mandatory** — defense against per-volatile-write-cache lie.
- **Per-FUSE daemon-timeout (optional)** — defense against per-malicious-daemon DoS.
- **Per-memcg writeback attribution** — defense against per-cgroup I/O starvation.

### grsecurity / pax surface

- **PaX UDEREF on fd table lookup** — defense against per-fd-table TOCTOU; SMAP forced during fdget.
- **GRKERNSEC_DMESG** — fsync EIO warnings rate-limited and gated to CAP_SYSLOG so userspace cannot drain kernel ring buffer probing for storage failure.
- **GRKERNSEC_HIDESYM on fsync klog** — block-device and inode pointers in fsync error klog stripped.
- **Per-PAX_REFCOUNT on file refcount during fsync** — defense against per-file UAF if concurrent close races writeback.
- **PaX KERNEXEC on filesystem ->fsync dispatch** — function pointer dispatched through hardened indirect-branch trampoline.
- **CAP_DAC_OVERRIDE strict** — fsync of an O_RDONLY fd is allowed only because the fd already passed the open-time DAC check; grsec does not relax CAP_DAC_OVERRIDE to permit fsync of foreign-uid files.
- **GRKERNSEC_CHROOT_FCHDIR adjacency** — directory-fd fsync inside a chroot is permitted only when the dir is rooted under the chroot.
- **Sync data-leak prevention** — fsync klog must not leak the i_size or block-map of files outside the caller's visibility scope; klog dump filtered.
- **Per-FUSE daemon timeout enforced** — defense against per-userspace-daemon hang propagating to kernel uninterruptible sleep.
- **PAX_USERCOPY_HARDEN on fsync error log copy** — defense against per-error-log-copy slab overflow.

