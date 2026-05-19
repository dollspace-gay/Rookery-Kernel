# Tier-5 syscall: fallocate(2) — syscall 285

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/open.c (SYSCALL_DEFINE4(fallocate), do_sys_fallocate, vfs_fallocate)
  - include/linux/falloc.h (FALLOC_FL_* flags)
  - include/uapi/linux/falloc.h
  - arch/x86/entry/syscalls/syscall_64.tbl (285  common  fallocate)
-->

## Summary

`fallocate(2)` manipulates the disk-space allocation of a file: preallocate space, punch holes, collapse / insert ranges, zero ranges, or unshare reflinked extents — all without writing user data through the page cache. Atomic at the filesystem-extent level.

Used widely by: qemu-img (disk image preallocation), Postgres (WAL preallocation), virtualization (sparse disk), torrent clients, log rotation (PUNCH_HOLE), video editors (INSERT_RANGE). Critical for: per-extent control, per-sparse-file management, per-reflink-unshare.

## Signature

```c
int fallocate(int fd, int mode, off_t offset, off_t len);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open writable file descriptor. |
| `mode` | `int` | in | Bitmask of `FALLOC_FL_*` flags. |
| `offset` | `off_t` | in | Starting byte offset. Must be `>= 0`. |
| `len` | `off_t` | in | Number of bytes. Must be `> 0`. |

## Flags (`mode`)

| Flag | Value | Meaning |
|---|---|---|
| (default = 0) | 0 | Preallocate; extends `i_size` only if `offset+len > i_size` AND `KEEP_SIZE` not set. |
| `FALLOC_FL_KEEP_SIZE` | 0x01 | Preallocate without growing `i_size`. |
| `FALLOC_FL_PUNCH_HOLE` | 0x02 | Deallocate range (must be combined with `KEEP_SIZE`); subsequent reads return zeros. |
| `FALLOC_FL_NO_HIDE_STALE` | 0x04 | Preallocate without zeroing (security-sensitive; usually rejected without privilege). |
| `FALLOC_FL_COLLAPSE_RANGE` | 0x08 | Remove range; subsequent bytes shift down; `i_size` shrinks by `len`. |
| `FALLOC_FL_ZERO_RANGE` | 0x10 | Zero the range (may use UNWRITTEN extents under the hood). |
| `FALLOC_FL_INSERT_RANGE` | 0x20 | Insert a hole at offset; subsequent bytes shift up; `i_size` grows by `len`. |
| `FALLOC_FL_UNSHARE_RANGE` | 0x40 | COW-break: copy reflinked extents so the range is owned exclusively. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | fd invalid / not writable. |
| `EINVAL` | offset < 0; len <= 0; mode unknown / unsupported combination; range not block-aligned for COLLAPSE/INSERT. |
| `EFBIG` | offset + len exceeds RLIMIT_FSIZE / fs limit; SIGXFSZ raised. |
| `ENOSPC` | Device full. |
| `EDQUOT` | Quota exceeded. |
| `EIO` | I/O error. |
| `ENODEV` | fd not on a filesystem (special files). |
| `EOPNOTSUPP` | Filesystem does not support requested mode. |
| `EPERM` | F_SEAL_WRITE / F_SEAL_GROW / F_SEAL_SHRINK blocks; or NO_HIDE_STALE without privilege. |
| `ETXTBSY` | fd is an active swap file or memory-mapped executable. |
| `ESPIPE` | fd is pipe/socket. |
| `EISDIR` | fd is a directory. |

## ABI surface

```text
__NR_fallocate  (x86_64) = 285
__NR_fallocate  (arm64)  = 47
__NR_fallocate  (riscv)  = 47
__NR_fallocate  (i386)   = 324

FALLOC_FL_KEEP_SIZE       0x01
FALLOC_FL_PUNCH_HOLE      0x02   /* requires KEEP_SIZE */
FALLOC_FL_NO_HIDE_STALE   0x04
FALLOC_FL_COLLAPSE_RANGE  0x08
FALLOC_FL_ZERO_RANGE      0x10
FALLOC_FL_INSERT_RANGE    0x20
FALLOC_FL_UNSHARE_RANGE   0x40

FALLOC_FL_SUPPORTED = KEEP_SIZE | PUNCH_HOLE | NO_HIDE_STALE |
                      COLLAPSE_RANGE | ZERO_RANGE |
                      INSERT_RANGE | UNSHARE_RANGE

RLIMIT_FSIZE — enforced for grow paths.
```

## Compatibility contract

REQ-1: Syscall number is **285** on x86_64. ABI-stable.

REQ-2: `offset < 0` or `len <= 0` ⟹ `-EINVAL`.

REQ-3: `mode & ~FALLOC_FL_SUPPORTED` ⟹ `-EINVAL`. Unknown bits NOT silently ignored.

REQ-4: `FALLOC_FL_PUNCH_HOLE` MUST be combined with `FALLOC_FL_KEEP_SIZE`; otherwise `-EINVAL`.

REQ-5: `FALLOC_FL_COLLAPSE_RANGE` MUST NOT be combined with any other mode bit (except `KEEP_SIZE` is implied/ignored); range MUST be filesystem-block-aligned; offset+len MUST NOT span EOF.

REQ-6: `FALLOC_FL_INSERT_RANGE` MUST NOT be combined with other mode bits; range block-aligned; offset MUST be < `i_size`.

REQ-7: `FALLOC_FL_ZERO_RANGE` MAY combine with `KEEP_SIZE`.

REQ-8: `FALLOC_FL_UNSHARE_RANGE` MAY combine with `KEEP_SIZE`.

REQ-9: `FALLOC_FL_NO_HIDE_STALE`: requires `CAP_SYS_RAWIO` (or grsec strict policy: requires `CAP_SYS_ADMIN`). Otherwise `-EPERM`.

REQ-10: Per-RLIMIT_FSIZE: if grow (offset+len > i_size and !KEEP_SIZE) and (offset+len > RLIMIT_FSIZE) ⟹ SIGXFSZ + `-EFBIG`. Punch / collapse / zero / unshare on existing extents do NOT grow file size and bypass RLIMIT_FSIZE.

REQ-11: Per-F_SEAL_WRITE: blocks any mode that modifies data (PUNCH_HOLE, ZERO_RANGE, COLLAPSE, INSERT, UNSHARE, plain allocate) ⟹ `-EPERM`.

REQ-12: Per-F_SEAL_GROW: blocks grow paths (default allocate beyond i_size, INSERT_RANGE) ⟹ `-EPERM`.

REQ-13: Per-F_SEAL_SHRINK: blocks COLLAPSE_RANGE ⟹ `-EPERM`.

REQ-14: Per-S_IMMUTABLE / S_APPEND attr: blocks all fallocate modes ⟹ `-EPERM`.

REQ-15: Per-fd S_ISREG check: directory ⟹ `-EISDIR`; non-regular ⟹ `-ENODEV` / `-ESPIPE`.

REQ-16: Per-active-swap file: `-ETXTBSY`.

REQ-17: Per-`f_op->fallocate` dispatch: filesystem must implement. Filesystems that do not ⟹ `-EOPNOTSUPP`.

REQ-18: Per-LSM: `security_file_permission(file, MAY_WRITE)`.

REQ-19: Per-overflow: `offset + len > S64_MAX` ⟹ `-EINVAL` (or `-EFBIG`).

REQ-20: Atomicity: fallocate is atomic at the extent level; partial failure leaves file in a state consistent with the underlying filesystem's journal.

## Acceptance Criteria

- [ ] AC-1: `fallocate(fd, 0, 0, 4096)` preallocates 4 KiB; i_size grows to 4096.
- [ ] AC-2: `fallocate(fd, KEEP_SIZE, 0, 4096)` preallocates; i_size unchanged.
- [ ] AC-3: `fallocate(fd, PUNCH_HOLE|KEEP_SIZE, 4096, 4096)` zeros block; subsequent reads return zeros.
- [ ] AC-4: `fallocate(fd, PUNCH_HOLE, 4096, 4096)` (no KEEP_SIZE) ⟹ `-EINVAL`.
- [ ] AC-5: `fallocate(fd, COLLAPSE_RANGE, 4096, 4096)` removes range; i_size shrinks.
- [ ] AC-6: `fallocate(fd, INSERT_RANGE, 4096, 4096)` inserts hole; bytes shift up.
- [ ] AC-7: COLLAPSE / INSERT misaligned ⟹ `-EINVAL`.
- [ ] AC-8: `fallocate(fd, ZERO_RANGE, 0, 4096)` zeros range.
- [ ] AC-9: `fallocate(fd, UNSHARE_RANGE, 0, 4096)` COW-breaks reflinked extent.
- [ ] AC-10: `fallocate(fd, 0x80, 0, 4096)` ⟹ `-EINVAL` (unknown flag).
- [ ] AC-11: F_SEAL_WRITE memfd: any fallocate ⟹ `-EPERM`.
- [ ] AC-12: F_SEAL_SHRINK: COLLAPSE ⟹ `-EPERM`.
- [ ] AC-13: F_SEAL_GROW: grow allocate ⟹ `-EPERM`.
- [ ] AC-14: RLIMIT_FSIZE exceeded on grow ⟹ SIGXFSZ + `-EFBIG`.
- [ ] AC-15: NO_HIDE_STALE without CAP_SYS_RAWIO ⟹ `-EPERM`.
- [ ] AC-16: Directory fd ⟹ `-EISDIR`.
- [ ] AC-17: Pipe fd ⟹ `-ESPIPE`.

## Architecture

```rust
#[syscall(nr = 285, abi = "sysv")]
pub fn sys_fallocate(fd: i32, mode: i32, offset: i64, len: i64) -> isize {
    Fallocate::do_fallocate(fd, mode as u32, offset, len)
}
```

`Fallocate::do_fallocate(fd, mode, offset, len) -> isize`:
1. if offset < 0 || len <= 0 { return Err(EINVAL); }
2. if offset.checked_add(len).is_none() { return Err(EINVAL); }
3. if mode & !FALLOC_FL_SUPPORTED != 0 { return Err(EINVAL); }
4. /* PUNCH_HOLE requires KEEP_SIZE */
5. if mode & FALLOC_FL_PUNCH_HOLE != 0 && mode & FALLOC_FL_KEEP_SIZE == 0 { return Err(EINVAL); }
6. /* COLLAPSE / INSERT mutually exclusive with most others */
7. if mode & FALLOC_FL_COLLAPSE_RANGE != 0 && mode & !(FALLOC_FL_COLLAPSE_RANGE) != 0 { return Err(EINVAL); }
8. if mode & FALLOC_FL_INSERT_RANGE != 0 && mode & !(FALLOC_FL_INSERT_RANGE) != 0 { return Err(EINVAL); }
9. /* Privileges for NO_HIDE_STALE */
10. if mode & FALLOC_FL_NO_HIDE_STALE != 0 && !current().has_cap(CAP_SYS_RAWIO) { return Err(EPERM); }
11. let file = current().files().get_file(fd).ok_or(EBADF)?;
12. if !file.has_mode(FMODE_WRITE) { return Err(EBADF); }
13. let inode = file.inode();
14. if inode.is_dir() { return Err(EISDIR); }
15. if !inode.is_reg() && !inode.is_blk() { return Err(ENODEV); }
16. /* Seals */
17. let seals = inode.seals();
18. let modifies = mode & (FALLOC_FL_PUNCH_HOLE | FALLOC_FL_ZERO_RANGE | FALLOC_FL_COLLAPSE_RANGE | FALLOC_FL_INSERT_RANGE | FALLOC_FL_UNSHARE_RANGE) != 0 || mode == 0;
19. if modifies && seals.contains(F_SEAL_WRITE) { return Err(EPERM); }
20. if mode & FALLOC_FL_COLLAPSE_RANGE != 0 && seals.contains(F_SEAL_SHRINK) { return Err(EPERM); }
21. let grows = (mode & FALLOC_FL_KEEP_SIZE == 0 && offset + len > inode.size() as i64) || mode & FALLOC_FL_INSERT_RANGE != 0;
22. if grows && seals.contains(F_SEAL_GROW) { return Err(EPERM); }
23. /* Immutable / append */
24. if inode.has_attr(S_IMMUTABLE) || inode.has_attr(S_APPEND) { return Err(EPERM); }
25. /* Active swap */
26. if inode.is_swap_active() { return Err(ETXTBSY); }
27. /* RLIMIT_FSIZE on grow */
28. if grows {
29.   let limit = current().rlimit(RLIMIT_FSIZE);
30.   if (offset + len) as u64 > limit { send_sig(SIGXFSZ, current()); return Err(EFBIG); }
31. }
32. lsm::file_permission(&file, MAY_WRITE)?;
33. /* Dispatch */
34. let op = file.f_op.fallocate.ok_or(EOPNOTSUPP)?;
35. op(&file, mode, offset, len)?;
36. Ok(0)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mode_validated` | INVARIANT | unknown bits ⟹ -EINVAL. |
| `punch_requires_keep_size` | INVARIANT | PUNCH_HOLE without KEEP_SIZE ⟹ -EINVAL. |
| `collapse_insert_exclusive` | INVARIANT | COLLAPSE/INSERT with other bits ⟹ -EINVAL. |
| `seal_write_blocks` | INVARIANT | F_SEAL_WRITE ⟹ -EPERM for modifying modes. |
| `seal_shrink_blocks_collapse` | INVARIANT | F_SEAL_SHRINK + COLLAPSE ⟹ -EPERM. |
| `seal_grow_blocks_grow` | INVARIANT | F_SEAL_GROW + grow path ⟹ -EPERM. |
| `rlimit_fsize_on_grow` | INVARIANT | grow + offset+len > RLIMIT_FSIZE ⟹ -EFBIG + SIGXFSZ. |
| `nohidestale_requires_cap` | INVARIANT | NO_HIDE_STALE without CAP_SYS_RAWIO ⟹ -EPERM. |
| `immutable_blocks_all` | INVARIANT | S_IMMUTABLE ⟹ -EPERM. |

### Layer 2: TLA+

`fs/fallocate.tla`:
- States: per-inode extent map, per-inode i_size, per-RLIMIT counter, per-inode seals.
- Properties:
  - `safety_mode_combinations` — illegal mode combos ⟹ -EINVAL.
  - `safety_seal_invariants` — seals respected per mode.
  - `safety_isize_grow_iff_no_keep_size_and_no_punch` — i_size only grows for default-allocate.
  - `safety_isize_shrink_iff_collapse` — i_size only shrinks for COLLAPSE.
  - `safety_rlimit_clamp` — never grow past RLIMIT_FSIZE.
  - `safety_no_hide_stale_priv` — NO_HIDE_STALE only with CAP_SYS_RAWIO.
  - `liveness_terminates` — fallocate returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fallocate` post: mode valid ∨ -EINVAL | `Fallocate::do_fallocate` |
| `do_fallocate` post: ret = 0 ⟹ extent map updated per mode | `Fallocate::do_fallocate` |
| `do_fallocate` post: seal violations ⟹ -EPERM with no state change | `Fallocate::do_fallocate` |

### Layer 4: Verus / Creusot functional

Per-`fallocate(2)` man-page semantic equivalence; xfstests `generic/263..275` (PUNCH/ZERO/COLLAPSE/INSERT/UNSHARE) pass; memfd seal tests pass; ext4/xfs/btrfs filesystem-specific tests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fallocate(2)` reinforcement:

- **Per-mode strict validation** — defense against per-forward-compat ABI breakage.
- **Per-mode combination rules** — defense against per-undefined-semantic flag combinations.
- **Per-seal respect (WRITE/GROW/SHRINK)** — defense against per-memfd tampering.
- **Per-RLIMIT_FSIZE + SIGXFSZ on grow** — defense against per-disk-fill DoS.
- **Per-NO_HIDE_STALE privileged** — defense against per-data-leak via unzeroed-extent disclosure.
- **Per-S_IMMUTABLE / S_APPEND blocks** — defense against per-protected-file tampering.
- **Per-ETXTBSY on swap** — defense against per-swap-file corruption.
- **Per-i64 overflow on offset+len** — defense against per-arithmetic-wrap.
- **Per-COLLAPSE/INSERT block-aligned** — defense against per-non-aligned-extent corruption.

## Grsecurity / PaX surface

- **PaX UDEREF on syscall args** — not applicable (scalar args), but mode/offset/len fed into f_op->fallocate with validated bounds.
- **GRKERNSEC_USERCOPY_HARDEN** — not directly applicable to fallocate (no user buffer); inherited via underlying fs paths.
- **PAX_REFCOUNT on file->f_count** — defense against per-UAF on concurrent close+fallocate.
- **GRKERNSEC_DMESG suppresses fallocate error logs** — `-EFBIG`, `-EPERM`, `-EOPNOTSUPP`, `-ENOSPC` rate-limited.
- **PaX KERNEXEC on f_op->fallocate dispatch** — RAP/CFI protects function pointer call.
- **GRKERNSEC_RESLOG on FALLOC_FL_NO_HIDE_STALE use** — every use audited (data-leak vector).
- **GRKERNSEC_RESLOG on large allocate (> 100 GiB)** — disk-fill DoS pattern audited.
- **NO_HIDE_STALE requires CAP_SYS_ADMIN under grsec strict policy** — tighter than upstream's CAP_SYS_RAWIO.
- **PAX_REFCOUNT on inode->i_writecount** — strict during COLLAPSE/INSERT exclusive locks.
- **GRKERNSEC_RESLOG on COLLAPSE/INSERT** — schema-changing ops audited (potential corruption vector).
- **PaX HARDENED_USERCOPY** — inherited via fs paths.
- **RLIMIT_FSIZE + SIGXFSZ on grow** — grsec enforces; no privileged bypass.
- **Per-fd S_ISREG / S_ISBLK strict** — defense against per-special-fd-type misuse.
- **PaX FORCED_SHRINK on PUNCH_HOLE excess** — under memory pressure, PUNCH_HOLE invalidates page cache first.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `posix_fallocate(3)` glibc wrapper (covered separately if expanded).
- `truncate(2)` / `ftruncate(2)` (covered separately).
- `FICLONERANGE` ioctl (covered separately).
- `madvise(MADV_DONTNEED)` (covered separately).
- Implementation code.
