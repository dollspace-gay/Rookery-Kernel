# Tier-5 syscall: sync_file_range(2) — syscall 277

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/sync.c (SYSCALL_DEFINE4(sync_file_range), ksys_sync_file_range)
  - include/linux/fs.h (SYNC_FILE_RANGE_WAIT_BEFORE / WRITE / WAIT_AFTER)
  - arch/x86/entry/syscalls/syscall_64.tbl (277  common  sync_file_range)
-->

## Summary

`sync_file_range(2)` flushes a byte range of a file's page cache to disk with explicit, fine-grained control over the writeback / wait phases. Unlike fsync/fdatasync, it does NOT commit any filesystem metadata and does NOT issue a block-device cache flush — it is a Linux-specific tuning primitive intended for buffered-I/O hot paths that want to hand-roll write streaming without taking the cost of a full file or fs sync.

Three flags compose a four-state phase machine: WAIT_BEFORE (wait for any prior in-flight writeback on the range), WRITE (start writeback on the dirty pages), WAIT_AFTER (wait for the newly started writeback to complete). Critical for: high-throughput streamers, write-pacing, write-and-forget bulk loaders.

## Signature

```c
int sync_file_range(int fd, off_t offset, off_t nbytes, unsigned int flags);
```

```c
#define SYNC_FILE_RANGE_WAIT_BEFORE  1
#define SYNC_FILE_RANGE_WRITE        2
#define SYNC_FILE_RANGE_WAIT_AFTER   4
#define SYNC_FILE_RANGE_WRITE_AND_WAIT (SYNC_FILE_RANGE_WAIT_BEFORE | SYNC_FILE_RANGE_WRITE | SYNC_FILE_RANGE_WAIT_AFTER)
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open writable fd. |
| `offset` | `off_t` | in | Byte start of the range. |
| `nbytes` | `off_t` | in | Length in bytes. `0` means "to end-of-file". |
| `flags` | `unsigned int` | in | Bitwise OR of the three SYNC_FILE_RANGE_* flags. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Per-phase semantics completed (i.e., requested phases satisfied; no metadata commit; no barrier). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` invalid or not open for write (some kernel versions). |
| `EINVAL` | `flags` contains bits outside the three defined; offset / nbytes overflow off_t arithmetic; `fd` refers to a non-regular file (pipe, fifo, socket, dir). |
| `ESPIPE` | `fd` refers to a pipe or FIFO. |
| `EIO` | Underlying writeback error. |
| `ENOMEM` | Could not allocate writeback control / per-page state. |
| `ENOSPC` | Delayed-alloc failed for a dirty page during writeback. |

## ABI surface

```text
__NR_sync_file_range  (x86_64) = 277
__NR_sync_file_range  (arm64)  = 84
__NR_sync_file_range  (riscv)  = 84
__NR_sync_file_range  (i386)   = 314
```

Note: the i386/ARM ABI uses a 7-argument variant `sync_file_range2(fd, flags, offset_lo, offset_hi, nbytes_lo, nbytes_hi)` to avoid 64-bit-args register alignment issues.

## Compatibility contract

REQ-1: Syscall number is **277** on x86_64. Introduced in Linux 2.6.17.

REQ-2: Flag combinations and their semantic phases (per Documentation/admin-guide/sync_file_range.rst):
- `0`: invalid (EINVAL).
- `WAIT_BEFORE`: wait for any in-flight writeback on the range; do not initiate any new writeback.
- `WRITE`: initiate writeback on dirty pages in the range; do not wait.
- `WAIT_BEFORE | WRITE`: wait for prior writeback, then start new writeback; return without waiting for the new writeback.
- `WAIT_AFTER`: wait for in-flight writeback (no semantic distinction from WAIT_BEFORE when used alone; in practice they map to the same internal phase).
- `WRITE | WAIT_AFTER`: start new writeback, wait for it to complete.
- `WAIT_BEFORE | WRITE | WAIT_AFTER` (= WRITE_AND_WAIT): full pipeline — wait, start, wait.

REQ-3: Does NOT commit filesystem metadata; on power loss, the data may have been written but the inode-block-map may not be updated, leaving data unreachable. **sync_file_range is NOT a substitute for fsync/fdatasync.** Per the man-page: "Warning: this system call is extremely dangerous and should not be used in portable programs."

REQ-4: Does NOT issue a block-device cache flush; data may sit in the disk's volatile write cache after return.

REQ-5: Range arithmetic: `endbyte = offset + nbytes - 1`; if `nbytes == 0`, `endbyte = LLONG_MAX`. The kernel rounds to page boundaries internally for the writeback control.

REQ-6: Per-`fd` must be a regular file or a block device. Pipes / FIFOs / sockets / character devices return ESPIPE or EINVAL.

REQ-7: Per-O_DIRECT: data already bypasses the page cache. sync_file_range is a no-op for purely O_DIRECT writes; mixed buffered + O_DIRECT files may still have dirty page-cache pages and benefit.

REQ-8: Per-write-only fd: pre-3.9 the kernel rejected RDONLY fds with EBADF; current kernels accept any open fd (no fd-mode check beyond fdget).

REQ-9: Per-FUSE: writeback is forwarded to the userspace daemon via FUSE_FSYNC with the range subset; meta commit suppressed.

REQ-10: Per-cgroup writeback: I/O attributed to the calling task's memcg.

REQ-11: Per-error: writeback errors propagate via errseq_t per file struct.

REQ-12: Per-shutdown sb: returns -EIO.

REQ-13: Per-MAP_PRIVATE pages: not eligible for sync_file_range (private dirty); kernel ignores them.

REQ-14: Per-PageWriteback wait: WAIT_BEFORE / WAIT_AFTER use `filemap_fdatawait_range(mapping, offset, endbyte)`.

REQ-15: Per-WRITE phase: uses `__filemap_fdatawrite_range(mapping, offset, endbyte, WB_SYNC_NONE)`.

REQ-16: sync_file_range is interruptible at writeback-wait points only in some filesystems.

## Acceptance Criteria

- [ ] AC-1: write 1 MiB; sync_file_range(fd, 0, 1 MiB, WRITE | WAIT_AFTER); writeback complete, no metadata commit.
- [ ] AC-2: sync_file_range(fd, 0, 0, WRITE): writeback to EOF.
- [ ] AC-3: sync_file_range(fd, 0, 1, 0): -EINVAL.
- [ ] AC-4: sync_file_range(fd, 0, 1, 0xFFFFFFFF): -EINVAL (unknown bits).
- [ ] AC-5: sync_file_range on a pipe: -ESPIPE.
- [ ] AC-6: sync_file_range on a tmpfs file: returns 0 (no backing bdev writeback).
- [ ] AC-7: sync_file_range followed by power loss with newly-allocated extents: data may be lost (documented hazard).
- [ ] AC-8: WAIT_BEFORE: blocks until prior writeback drains.
- [ ] AC-9: WRITE: returns before writeback complete.
- [ ] AC-10: WRITE | WAIT_AFTER: returns only after the started writeback drains.
- [ ] AC-11: errseq EIO propagated once per file.
- [ ] AC-12: O_DIRECT file with no buffered dirty pages: returns 0 quickly.

## Architecture

```rust
#[syscall(nr = 277, abi = "sysv")]
pub fn sys_sync_file_range(fd: i32, offset: i64, nbytes: i64, flags: u32) -> isize {
    Sync::do_sync_file_range(fd, offset, nbytes, flags)
}
```

`Sync::do_sync_file_range(fd, offset, nbytes, flags) -> isize`:
1. /* Validate flags */
2. if flags & !VALID_FLAGS { return -EINVAL; }
3. if offset < 0 || nbytes < 0 { return -EINVAL; }
4. /* Compute endbyte */
5. let endbyte = if nbytes == 0 { i64::MAX } else { offset.checked_add(nbytes - 1).ok_or(EINVAL)? };
6. /* Lookup fd */
7. let file = fdget(fd).ok_or(EBADF)?;
8. let inode = file.f_inode();
9. /* Reject non-regular */
10. if !inode.is_reg() && !inode.is_blk() { fdput(file); return -EINVAL; }
11. if inode.is_fifo() || inode.is_sock() { fdput(file); return -ESPIPE; }
12. /* Per-phase dispatch */
13. let mapping = file.f_mapping;
14. let mut r: isize = 0;
15. if flags & SYNC_FILE_RANGE_WAIT_BEFORE {
16.   r = filemap_fdatawait_range(mapping, offset, endbyte);
17.   if r != 0 { fdput(file); return r; }
18. }
19. if flags & SYNC_FILE_RANGE_WRITE {
20.   r = __filemap_fdatawrite_range(mapping, offset, endbyte, WB_SYNC_NONE);
21.   if r != 0 { fdput(file); return r; }
22. }
23. if flags & SYNC_FILE_RANGE_WAIT_AFTER {
24.   r = filemap_fdatawait_range(mapping, offset, endbyte);
25. }
26. /* errseq advance */
27. let e = errseq_check_and_advance(&file.f_wb_err, &mut file.f_wb_err_seen);
28. fdput(file);
29. if r == 0 && e != 0 { return e; }
30. r

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated_first` | INVARIANT | unknown bits ⟹ EINVAL before any work. |
| `phase_order` | ORDER | WAIT_BEFORE phase strictly before WRITE phase strictly before WAIT_AFTER phase. |
| `no_metadata_commit` | INVARIANT | no s_op->sync_fs invoked. |
| `no_bdev_flush` | INVARIANT | no blkdev_issue_flush invoked. |
| `range_arith_no_overflow` | INVARIANT | offset + nbytes overflow ⟹ EINVAL. |

### Layer 2: TLA+

`fs/sync-file-range.tla`:
- States: per-write, per-sfr-call, per-wait-before, per-write-phase, per-wait-after.
- Properties:
  - `safety_phase_strict_order` — phases composed in strict order.
  - `safety_no_meta_commit` — metadata never committed by this call.
  - `safety_no_bdev_flush` — block-device flush never issued.
  - `liveness_terminates` — every sfr returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_sync_file_range` post: requested phases applied | `Sync::do_sync_file_range` |
| `flags` validation: VALID_FLAGS subset enforced | `Sync::do_sync_file_range` |
| `range arith`: i64::MAX upper bound respected | `Sync::do_sync_file_range` |

### Layer 4: Verus / Creusot functional

Per-`sync_file_range(2)` man-page, per-Documentation/admin-guide/sync_file_range.rst, xfstests generic/466.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sync_file_range(2)` reinforcement:

- **Per-fd validation pre-I/O** — defense against per-bad-fd kernel UAF.
- **Per-flag whitelist** — defense against per-future-flag privilege confusion.
- **Per-range-arith overflow check** — defense against per-i64-wrap kernel oops.
- **Per-non-regular reject** — defense against per-pipe/socket pseudo-dispatch.
- **Per-errseq strict** — defense against per-silent-data-loss.

## Grsecurity / PaX surface

- **PaX UDEREF on fd lookup and writeback bookkeeping** — defense against per-fd-table TOCTOU; SMAP forced.
- **GRKERNSEC_DMESG** — sync_file_range error klog rate-limited and CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM in sfr klog** — file mapping and bdev pointer stripped.
- **PAX_REFCOUNT on file refcount during sfr** — defense against per-file UAF.
- **PaX KERNEXEC on writeback dispatch** — indirect call hardened.
- **Sync data-leak prevention** — sfr error klog must not leak per-file extent layout.
- **CAP_DAC_OVERRIDE strict** — sfr does not bypass open-time DAC.
- **GRKERNSEC_CHROOT_FCHDIR adjacency** — sfr inside a chroot operates only on fds whose underlying sb is reachable inside the chroot.
- **Per-FUSE daemon timeout enforced** — defense against per-userspace-hang.
- **PAX_USERCOPY_HARDEN on klog buffer** — defense against per-buffer overflow.
- **Per-namespace scoping** — sfr cannot synchronize an fd opened in a sibling mount namespace if that namespace has been hidden by grsec policy.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-filesystem writeback worker internals (covered in Tier-3 fs/fs-writeback.md).
- io_uring SYNC_FILE_RANGE async variant (covered in Tier-5 io_uring_enter.md).
- 32-bit sync_file_range2 ABI (covered in arch Tier-5 docs).
- Implementation code.
