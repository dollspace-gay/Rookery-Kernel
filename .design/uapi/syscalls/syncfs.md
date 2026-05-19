# Tier-5 syscall: syncfs(2) — syscall 306

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/sync.c (SYSCALL_DEFINE1(syncfs))
  - include/linux/fs.h (super_operations::sync_fs)
  - arch/x86/entry/syscalls/syscall_64.tbl (306  common  syncfs)
-->

## Summary

`syncfs(2)` writes back ALL modified in-core data and metadata of the SINGLE filesystem identified by the open file descriptor `fd`, then issues a write-barrier / cache-flush on its backing block device(s). It is the per-filesystem variant of `sync(2)` — surgical durability rather than system-wide. The caller does not need write access to `fd`; merely having any fd open on the filesystem is sufficient.

syncfs(2) is the modern primitive for container-aware durability: a container that wishes to "sync its world" without disturbing the host (and other containers) calls syncfs on a fd rooted in its own filesystem. Critical for: container snapshots, per-filesystem fence in multi-tenant systems, image-builder commit.

## Signature

```c
int syncfs(int fd);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Any open fd on the target filesystem; access mode irrelevant. |

## Return value

| Value | Meaning |
|---|---|
| `0` | All dirty data + metadata of the filesystem containing `fd` durable. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` invalid. |
| `EIO` | Per-fs writeback or barrier failed. |
| `ENOSPC` | Delayed-alloc failed during writeback. |
| `EROFS` | (Filesystem read-only; per Linux semantics returns 0; ENOSPC/EROFS reportable in writeback errors only.) |

## ABI surface

```text
__NR_syncfs  (x86_64) = 306
__NR_syncfs  (arm64)  = 267
__NR_syncfs  (riscv)  = 267
__NR_syncfs  (i386)   = 344
```

## Compatibility contract

REQ-1: Syscall number is **306** on x86_64. Added in Linux 2.6.39.

REQ-2: syncfs(fd) is equivalent to two-pass writeback + barrier on a SINGLE superblock:
- Pass 1: `sync_inodes_sb(sb)` — writeback all dirty pages and inodes; wait for completion.
- Pass 2: `sb->s_op->sync_fs(sb, /* wait */ 1)` — commit per-fs metadata.
- Pass 3: `sync_blockdev(sb->s_bdev)` — barrier each backing block device.

REQ-3: Per-write access NOT required. Per-Linux 2.6.39 release notes: any fd accepted to allow read-only mounters to fence.

REQ-4: Per-fd-to-sb resolution: `file->f_inode->i_sb`. Bind-mounted views: each bind-mount sees the same underlying sb; syncfs syncs the underlying sb.

REQ-5: Per-overlayfs: syncfs on an overlayfs fd syncs the overlay's upper sb (where writes go) + barriers the upper-sb's bdev. The lower sb is RO and untouched.

REQ-6: Per-tmpfs / pseudo-FS: sync_fs is no-op; syncfs returns 0 quickly.

REQ-7: Per-FUSE: forwards a syncfs request to the userspace daemon (FUSE_SYNCFS opcode). Daemon returning ENOSYS is folded to success.

REQ-8: Per-error: pre-Linux 5.8, syncfs did not propagate writeback errors; post-5.8 (commit 735e4ae5ba28) syncfs propagates errseq_t errors so callers see EIO once per sb per error generation.

REQ-9: Per-CAP_DAC_OVERRIDE NOT required: open(fd) suffices; subsequent sync is on the sb, not the file.

REQ-10: Per-network FS: forwards COMMIT-RPC; latency network-bound.

REQ-11: Per-s_umount read-lock held during sync_fs to prevent unmount race.

REQ-12: Per-fd reference: `fdget(fd)` taken at entry, `fdput(fd)` at exit; sb pinned indirectly via the fd's file struct (which pins inode which pins sb).

REQ-13: Per-shutdown sb: returns -EIO; no writeback.

REQ-14: Per-bdev write-cache: blkdev_issue_flush issued; EOPNOTSUPP folded to 0.

REQ-15: Per-cgroup: writeback attributed to the calling task's memcg / blkcg.

REQ-16: syncfs(2) is interruptible only at safe sync points (filesystem-defined).

## Acceptance Criteria

- [ ] AC-1: write into a file on ext4; syncfs(fd_on_ext4): ext4 fully durable; other filesystems untouched.
- [ ] AC-2: syncfs(fd_on_tmpfs): returns 0 immediately (no backing bdev).
- [ ] AC-3: syncfs(-1): -EBADF.
- [ ] AC-4: syncfs on overlayfs: upper sb fully durable; lower untouched.
- [ ] AC-5: syncfs after a writeback EIO: returns -EIO once, then 0.
- [ ] AC-6: syncfs by unprivileged user on a fd open by them: succeeds.
- [ ] AC-7: syncfs during concurrent unmount: blocks until s_umount available; returns success or post-unmount EBADF (sb pinned).
- [ ] AC-8: syncfs on FUSE filesystem: forwards to daemon; ENOSYS folded to 0.
- [ ] AC-9: syncfs error propagation: errseq advances per sb.
- [ ] AC-10: syncfs on shutdown sb returns -EIO.

## Architecture

```rust
#[syscall(nr = 306, abi = "sysv")]
pub fn sys_syncfs(fd: i32) -> isize {
    Sync::do_syncfs(fd)
}
```

`Sync::do_syncfs(fd) -> isize`:
1. let file = fdget(fd).ok_or(EBADF)?;
2. let sb = file.f_inode().i_sb();
3. /* Take read-lock on sb umount */
4. down_read(&sb.s_umount);
5. let r = Sync::sync_filesystem(sb);
6. up_read(&sb.s_umount);
7. fdput(file);
8. /* errseq propagation */
9. let err = errseq_check_and_advance(&sb.s_wb_err, &mut file.f_sb_err_seen);
10. if r == 0 && err != 0 { return err; }
11. r

`Sync::sync_filesystem(sb) -> isize`:
1. if sb.s_flags & SB_RDONLY { return 0; }
2. /* Pass 1: data writeback */
3. let mut r = sync_inodes_sb_internal(sb, /* wait */ true);
4. /* Pass 2: per-fs metadata commit */
5. if let Some(op) = sb.s_op.sync_fs {
6.   let r2 = op(sb, /* wait */ 1);
7.   if r == 0 { r = r2; }
8. }
9. /* Pass 3: per-bdev barrier */
10. if sb.s_bdev != null {
11.   let r3 = blkdev_issue_flush(sb.s_bdev);
12.   if r3 != -EOPNOTSUPP && r == 0 { r = r3; }
13. }
14. r

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_validated_before_use` | INVARIANT | EBADF returned before sb deref. |
| `s_umount_held_during_sync` | INVARIANT | sb.s_umount read-lock held across sync_filesystem. |
| `bdev_flush_after_data` | ORDER | data writeback drains BEFORE bdev flush. |
| `errseq_advances_per_sb_per_file` | INVARIANT | per-(sb, file) errseq cursor monotone. |

### Layer 2: TLA+

`fs/syncfs.tla`:
- States: per-write-on-fs, per-syncfs, per-data-flush, per-meta-commit, per-bdev-barrier.
- Properties:
  - `safety_target_sb_durable` — only the target sb is synced.
  - `safety_no_sb_uaf` — sb pinned via fd; refcount balanced.
  - `safety_errseq_monotone` — per-sb error cursor monotone.
  - `liveness_terminates` — every syncfs returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_syncfs` post: target sb dirty count 0 | `Sync::do_syncfs` |
| `sync_filesystem` post: s_op.sync_fs called if present | `Sync::sync_filesystem` |
| `errseq` advance: at most one EIO per file per error generation | `Sync::do_syncfs` |

### Layer 4: Verus / Creusot functional

Per-`syncfs(2)` man-page, per-Linux 2.6.39 release notes, per-LWN article on syncfs error propagation (commit 735e4ae5ba28), xfstests generic/643.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`syncfs(2)` reinforcement:

- **Per-fd validation pre-sb-deref** — defense against per-bad-fd kernel UAF.
- **Per-s_umount read-lock** — defense against per-unmount race.
- **Per-errseq sb-cursor** — defense against per-silent-data-loss.
- **Per-overlayfs upper-sb-only sync** — defense against per-lower-bdev unnecessary disturbance.
- **Per-FUSE daemon timeout** — defense against per-daemon-hang.

## Grsecurity / PaX surface

- **PaX UDEREF on fd lookup** — defense against per-fd-table TOCTOU; SMAP forced.
- **GRKERNSEC_DMESG** — syncfs error klog rate-limited and CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM in syncfs klog** — sb / bdev pointers stripped.
- **PAX_REFCOUNT on file refcount and sb->s_count during syncfs** — defense against per-syncfs-induced refcount overflow.
- **PaX KERNEXEC on s_op->sync_fs dispatch** — indirect call hardened.
- **Sync data-leak prevention** — syncfs klog must not leak per-sb extent map or per-bdev geometry; klog filtered.
- **CAP_DAC_OVERRIDE strict** — syncfs does not bypass open-time DAC.
- **GRKERNSEC_CHROOT_FCHDIR adjacency** — syncfs inside a chroot is only permitted when the fd's sb is reachable from inside the chroot; fds pointing at a sb mounted only outside the chroot are rejected.
- **Per-namespace scoping** — syncfs in a user namespace can only sync sbs that the namespace can resolve via its mount namespace; cross-mntns syncfs rejected.
- **Per-FUSE daemon timeout enforced** — defense against per-userspace-hang.
- **PAX_USERCOPY_HARDEN on klog buffer** — defense against per-buffer overflow.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-filesystem ->sync_fs internals (covered in Tier-3 fs/<fs>-sync.md).
- Writeback worker plumbing (covered in Tier-3 fs/fs-writeback.md).
- Overlayfs upper/lower split semantics (covered in Tier-3 fs/overlayfs.md).
- Implementation code.
