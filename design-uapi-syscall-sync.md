---
title: "Tier-5 syscall: sync(2) — syscall 162"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sync(2)` writes back ALL modified in-core data and metadata of EVERY filesystem mounted on the system, then issues a write-barrier / cache-flush so that the entire system state on every mounted writable filesystem is durable. Historically (pre-2.6.35) Linux made `sync(2)` non-blocking — it merely scheduled writeback and returned. Modern Linux makes it fully synchronous: it returns only after every dirty page on every filesystem has been written and every block-device cache has been flushed.

sync(2) is the system-wide durability barrier used by shutdown scripts, container snapshotting, `sync` shell command, and large-scale crash-consistency probes. Critical for: ordered shutdown, multi-filesystem snapshot consistency, bulk durability fence.

### Acceptance Criteria

- [ ] AC-1: dirty pages on tmpfs and ext4 simultaneously; sync(); both writeback complete on return.
- [ ] AC-2: sync() under heavy write load: returns only after queues drain.
- [ ] AC-3: sync() with FUSE filesystem hung: kernel respects grsec timeout (default 30s) and logs WARN.
- [ ] AC-4: sync() under root and under nobody: both return successfully (default policy).
- [ ] AC-5: grsec sysctl `gr_sync_priv = 1`: sync() under nobody returns -EPERM.
- [ ] AC-6: sync() with read-only mounts: completes; no writeback issued for ro filesystems.
- [ ] AC-7: sync() during ongoing unmount: ordered via s_umount; no UAF.
- [ ] AC-8: sync() twice in succession: second call ~no-op latency.

### Architecture

```rust
#[syscall(nr = 162, abi = "sysv")]
pub fn sys_sync() -> isize {
    Sync::do_sync()
}
```

`Sync::do_sync() -> isize`:
1. /* Pass 1: enqueue writeback */
2. Sync::iterate_supers(|sb| Sync::sync_inodes_one_sb(sb, /* wait */ false));
3. /* Pass 2: commit per-fs metadata */
4. Sync::iterate_supers(|sb| Sync::sync_fs_one_sb(sb, /* wait */ true));
5. /* Pass 3: wait for block-device queue drain */
6. Sync::iterate_bdevs(|bdev| Sync::sync_blockdev_one(bdev));
7. /* Pass 4: per-bdev barrier */
8. Sync::iterate_bdevs(|bdev| Sync::sync_blockdev_barrier(bdev));
9. /* Final accounting */
10. 0

`Sync::iterate_supers(callback)`:
1. spin_lock(&sb_lock);
2. for sb in &super_blocks {
3.   if sb.s_flags & SB_BORN == 0 { continue; }
4.   sb.s_count += 1;
5.   spin_unlock(&sb_lock);
6.   down_read(&sb.s_umount);
7.   if sb.s_root != NULL && (sb.s_flags & SB_ACTIVE) {
8.     callback(sb);
9.   }
10.  up_read(&sb.s_umount);
11.  spin_lock(&sb_lock);
12.  sb.s_count -= 1;
13. }
14. spin_unlock(&sb_lock);

`Sync::sync_inodes_one_sb(sb, wait) -> ()`:
1. let wbc = WritebackControl { sync_mode: if wait { WB_SYNC_ALL } else { WB_SYNC_NONE }, nr_to_write: LLONG_MAX, range_start: 0, range_end: LLONG_MAX, .. };
2. sync_inodes_sb_internal(sb, &mut wbc);

`Sync::sync_fs_one_sb(sb, wait) -> ()`:
1. if sb.s_flags & SB_RDONLY { return; }
2. if let Some(op) = sb.s_op.sync_fs { op(sb, wait); }

`Sync::sync_blockdev_one(bdev) -> ()`:
1. let mapping = bdev_mapping(bdev);
2. filemap_fdatawait(mapping);

`Sync::sync_blockdev_barrier(bdev) -> ()`:
1. let r = blkdev_issue_flush(bdev);
2. if r != 0 && r != -EOPNOTSUPP { pr_warn("sync: bdev %s flush=%d", bdev.name, r); }

### Out of Scope

- Per-filesystem ->sync_fs implementations (covered in Tier-3 fs/<fs>-sync.md).
- Writeback worker thread internals (covered in Tier-3 fs/fs-writeback.md).
- Block-device flush plumbing (covered in Tier-3 block/blk-flush.md).
- Implementation code.

### signature

```c
void sync(void);
```

Returns no value, although the underlying Linux syscall returns 0.

### parameters

None.

### return value

| Value | Meaning |
|---|---|
| `0` | All dirty data on all mounted filesystems durable. |
| (POSIX defines no failure path; Linux kernel always returns 0; partial-failures are logged.) |

### errors

| errno | Trigger |
|---|---|
| (none surfaced to userspace) | Per-filesystem failures are logged to dmesg; the syscall itself does not propagate. |

### abi surface

```text
__NR_sync  (x86_64) = 162
__NR_sync  (arm64)  = 81
__NR_sync  (riscv)  = 81
__NR_sync  (i386)   = 36
```

### compatibility contract

REQ-1: Syscall number is **162** on x86_64. ABI-stable since 0.99.

REQ-2: Per-POSIX: sync schedules writeback. Linux strengthens to: sync returns only when writeback is complete and barriers issued. (Per Documentation/filesystems/sync.rst, post-2.6.35.)

REQ-3: Two-pass algorithm:
- Pass 1: `iterate_supers(sync_inodes_one_sb, NULL)` — initiate writeback on every superblock (WB_SYNC_NONE).
- Pass 2: `iterate_supers(sync_fs_one_sb, NULL)` — wait for completion and commit metadata.
- Pass 3: `iterate_bdevs(fdatawait_one_bdev, NULL)` — wait for block-device queues to drain.
- Pass 4: `iterate_bdevs(sync_blockdev_one, NULL)` — issue barrier on every block device.

REQ-4: Per-cred check: ANY user may call sync(2) — historically. But grsec restricts to CAP_SYS_ADMIN to prevent per-user disk-storm DoS.

REQ-5: Per-filesystem `->sync_fs(sb, wait)` callback dispatched on each writable superblock during pass 2 with `wait=1`.

REQ-6: Per-pseudo filesystem: tmpfs, procfs, sysfs, debugfs — sync_fs is no-op (no backing device).

REQ-7: Per-O_NOATIME / lazytime: atime / mtime / ctime updates pending in-memory MAY be lost on crash if sync was not called; sync(2) commits them.

REQ-8: Per-MS_RDONLY mount: still iterated, but writeback is a no-op.

REQ-9: Per-FUSE: forwards a sync request to each mounted FUSE filesystem's userspace daemon; the syscall blocks until every daemon responds (or grsec-bounded timeout).

REQ-10: sync(2) is interruptible only at superblock boundaries (between filesystems); cannot be interrupted mid-writeback of a single filesystem.

REQ-11: Per-disk-quota: sync forces quota flush on every quota-enabled filesystem.

REQ-12: Per-`SYNC_OUTER_LOCK`: each superblock taken via `s_umount` read-lock to prevent unmount race.

REQ-13: Per-`shutdown` mount state: shutdown filesystems skipped (already non-writable).

REQ-14: Per-network filesystem (NFS, CIFS, SMB): sync forwards COMMIT RPC; latency dominated by network round-trip.

REQ-15: sync(2) is monotone — calling it twice in a row: second call has no work to do, returns quickly.

REQ-16: Per-process accounting: I/O work attributed to the calling task's blkcg / memcg via wbc.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `s_umount_held_during_sync_fs` | INVARIANT | sync_fs called with s_umount read-lock held. |
| `s_count_balanced` | INVARIANT | per-sb refcount get/put balanced across iterate_supers. |
| `four_pass_order` | ORDER | enqueue ⟶ commit-meta ⟶ wait ⟶ barrier (strict). |
| `unmounted_sb_skipped` | INVARIANT | sb without SB_ACTIVE skipped. |

### Layer 2: TLA+

`fs/sync.tla`:
- States: per-sb-enqueue, per-sb-meta-commit, per-bdev-wait, per-bdev-barrier.
- Properties:
  - `safety_all_dirty_persisted` — every dirty page at entry is durable at exit.
  - `safety_no_sb_uaf` — sb refcount strictly balanced; unmount blocked.
  - `safety_four_phase_order` — phases strictly ordered.
  - `liveness_sync_terminates` — every sync returns (under healthy bdev).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_sync` post: every sb visited; every bdev flushed | `Sync::do_sync` |
| `iterate_supers` post: every active sb visited exactly once | `Sync::iterate_supers` |
| `sync_blockdev_barrier` post: EOPNOTSUPP folded silently | `Sync::sync_blockdev_barrier` |

### Layer 4: Verus / Creusot functional

Per-`sync(2)` man-page, per-POSIX, per-Documentation/filesystems/sync.rst, xfstests generic/445.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sync(2)` reinforcement:

- **Per-superblock refcount balanced** — defense against per-sb UAF during concurrent unmount.
- **Per-s_umount read-lock** — defense against per-unmount race.
- **Per-bdev refcount** — defense against per-bdev UAF.
- **Per-four-pass strict order** — defense against per-pre-meta-barrier ordering bug.
- **Per-FUSE daemon timeout** — defense against per-malicious-daemon hang.

### grsecurity / pax surface

- **PaX UDEREF in sync klog path** — defense against per-klog kernel-deref bug.
- **GRKERNSEC_DMESG** — sync error klog rate-limited and CAP_SYSLOG-gated; unprivileged callers cannot observe per-filesystem failure messages.
- **GRKERNSEC_HIDESYM in sync klog** — superblock and bdev pointers stripped from any sync-emitted klog.
- **PAX_REFCOUNT on sb->s_count + bdev refcount during sync** — defense against per-sync-induced refcount overflow.
- **PaX KERNEXEC on s_op->sync_fs dispatch** — indirect-call hardened.
- **Sync data-leak prevention** — sync klog must not leak per-superblock layout or per-bdev geometry; klog filtered.
- **gr_sync_priv sysctl** — when set, sync(2) requires CAP_SYS_ADMIN. Defense against per-unprivileged disk-storm DoS (e.g. forking task spamming sync to stall I/O subsystem).
- **GRKERNSEC_CHROOT_FCHDIR ortho** — sync inside a chroot only flushes superblocks reachable from inside the chroot's mount namespace (no global flush of host filesystems).
- **CAP_DAC_OVERRIDE strict** — sync does not bypass DAC since no per-file access is needed, but grsec still gates the elevated-iteration path through capability check.
- **Per-FUSE daemon timeout enforced (default 30s)** — defense against per-userspace-hang lifting kernel into uninterruptible.
- **PAX_USERCOPY_HARDEN on any sync-related copy_to_user** — defense against per-buffer overflow.

