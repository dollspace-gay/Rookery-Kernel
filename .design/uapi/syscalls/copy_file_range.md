# Tier-5 syscall: copy_file_range(2) — syscall 326

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/read_write.c (SYSCALL_DEFINE6(copy_file_range), do_copy_file_range, vfs_copy_file_range)
  - fs/remap_range.c (do_clone_file_range, dedupe paths)
  - include/linux/fs.h (f_op->copy_file_range, f_op->remap_file_range)
  - arch/x86/entry/syscalls/syscall_64.tbl (326  common  copy_file_range)
-->

## Summary

`copy_file_range(2)` copies up to `len` bytes from one file (`fd_in`) at offset `*off_in` to another file (`fd_out`) at offset `*off_out`, **server-side / in-kernel**. On filesystems that support reflink (btrfs, XFS with reflink, NFSv4.2 SSC, CephFS) it becomes a metadata-only copy-on-write share; on others it falls back to a kernel splice loop.

Used by: GNU `cp` (since coreutils 9.0), file managers, container layer extraction, backup tools. Critical for: per-server-side copy avoiding double user/kernel page-cache transfer, per-reflink instant-clone.

## Signature

```c
ssize_t copy_file_range(int fd_in, off64_t *off_in,
                        int fd_out, off64_t *off_out,
                        size_t len, unsigned int flags);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd_in` | `int` | in | Source fd (must be readable, seekable). |
| `off_in` | `off64_t *` | in/out | If non-NULL: read from `*off_in`, advance by bytes copied. If NULL: read from and advance `fd_in`'s f_pos. |
| `fd_out` | `int` | in | Destination fd (must be writable, seekable). |
| `off_out` | `off64_t *` | in/out | If non-NULL: write at `*off_out`, advance by bytes copied. If NULL: write at and advance `fd_out`'s f_pos. |
| `len` | `size_t` | in | Maximum bytes to copy. |
| `flags` | `unsigned int` | in | Reserved; MUST be 0. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Bytes copied; 0 indicates EOF at source. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | fd_in not readable, or fd_out not writable. |
| `EINVAL` | flags != 0; same fd with overlapping ranges; either fd not regular file. |
| `EFAULT` | off_in / off_out pointers bad. |
| `EFBIG` | Destination would exceed RLIMIT_FSIZE; SIGXFSZ raised. |
| `ENOSPC` | Destination filesystem full. |
| `EDQUOT` | Quota exceeded. |
| `EIO` | I/O error. |
| `EXDEV` | Source and dest on different filesystems AND filesystem does not support cross-fs (pre-5.3 limitation; relaxed in 5.3+ via generic splice fallback). |
| `EOPNOTSUPP` | Filesystem does not support copy_file_range. |
| `ETXTBSY` | One of the fds is a swap file or active text segment. |
| `EISDIR` | One of the fds is a directory. |
| `EOVERFLOW` | offset + len overflows. |
| `EPERM` | Source is append-only; destination has F_SEAL_WRITE. |

## ABI surface

```text
__NR_copy_file_range  (x86_64) = 326
__NR_copy_file_range  (arm64)  = 285
__NR_copy_file_range  (riscv)  = 285
__NR_copy_file_range  (i386)   = 377

flags                = 0  (reserved; non-zero ⟹ -EINVAL)

/* Filesystems with native server-side copy:
 *   btrfs    — reflink (COW share)
 *   XFS      — reflink (with mkfs -m reflink=1)
 *   NFSv4.2  — SSC (Server-Side Copy)
 *   CephFS   — copy-from object op
 *   OCFS2    — reflink
 *   nfs (v3) — falls back to splice loop in kernel
 *   ext4     — splice fallback (no native reflink)
 *   tmpfs    — splice fallback
 */

RLIMIT_FSIZE — enforced on dest; exceed ⟹ SIGXFSZ + -EFBIG
```

## Compatibility contract

REQ-1: Syscall number is **326** on x86_64. ABI-stable.

REQ-2: `flags != 0` ⟹ `-EINVAL`.

REQ-3: Both fds MUST refer to regular files; directories / pipes / sockets / char devices ⟹ `-EINVAL` or `-EISDIR`.

REQ-4: Source and destination MAY be the same file as long as ranges do not overlap; overlapping same-file ranges ⟹ `-EINVAL`. Per kernel: `(fd_in == fd_out) ∧ (off_in_range overlaps off_out_range)` ⟹ EINVAL.

REQ-5: `off_in == NULL`: use and advance `fd_in.f_pos`. `off_in != NULL`: use `*off_in`, write back updated value (NOT update f_pos).

REQ-6: `off_out == NULL`: use and advance `fd_out.f_pos`. Same semantics symmetric.

REQ-7: Per cross-mount: pre-5.3 returns `-EXDEV` for different superblocks; 5.3+ falls back to `generic_copy_file_range` (splice loop) when filesystems differ. **Even with fallback, kernel validates that both filesystems agree on user mapping (same userns, same idmap)**; mismatched idmap ⟹ `-EXDEV`.

REQ-8: Per-RLIMIT_FSIZE on destination: if `*off_out + len > limit` ⟹ truncate `len`; if `*off_out >= limit` ⟹ SIGXFSZ + `-EFBIG`.

REQ-9: Per-F_SEAL_WRITE on dest inode: ⟹ `-EPERM`.

REQ-10: Per-LSM: `security_file_permission(fd_in, MAY_READ)` AND `security_file_permission(fd_out, MAY_WRITE)`; also `security_path_*` may veto.

REQ-11: Per-`f_op->copy_file_range` if defined: filesystem driver handles natively (btrfs reflink). Else `generic_copy_file_range` does splice loop.

REQ-12: Per-`do_clone_file_range`: btrfs FICLONERANGE ioctl path is a separate clone semantic (copy_file_range MAY use it under the hood as an optimization, but only when ranges are block-aligned).

REQ-13: Per-EOF on source: returns short count (partial copy succeeds).

REQ-14: Per-signal: signal before any byte ⟹ `-EINTR`; else short return.

REQ-15: Per-immutable / append-only attr: source `S_IMMUTABLE` allowed; dest `S_IMMUTABLE` ⟹ `-EPERM`; dest `S_APPEND` ⟹ requires off_out == NULL or `*off_out == i_size`.

REQ-16: Per-len: capped at SSIZE_MAX; very large len performs internal splice in chunks of MAX_RW_COUNT.

REQ-17: Per-overflow: `*off_in + len > S64_MAX` ⟹ `-EOVERFLOW`. Same for off_out.

REQ-18: Per `*off_in` writeback: kernel writes back the post-copy offset to the user pointer BEFORE returning (so partial copies leave consistent state).

## Acceptance Criteria

- [ ] AC-1: `copy_file_range(fd_in, NULL, fd_out, NULL, 4096, 0)` copies 4 KiB from f_pos to f_pos.
- [ ] AC-2: `copy_file_range(fd_in, &in, fd_out, &out, 4096, 0)` copies at explicit offsets without moving f_pos.
- [ ] AC-3: `copy_file_range(fd, &a, fd, &b, len, 0)` with overlap ⟹ `-EINVAL`.
- [ ] AC-4: `copy_file_range(fd_in, NULL, fd_out, NULL, 4096, 1)` ⟹ `-EINVAL`.
- [ ] AC-5: `copy_file_range(pipe_rd, NULL, fd_out, NULL, 4096, 0)` ⟹ `-EINVAL`.
- [ ] AC-6: btrfs source + btrfs dest, aligned: reflink share (no data copied; same checksum, same extent).
- [ ] AC-7: Cross-mount ext4 → xfs: splice fallback succeeds (kernel >= 5.3).
- [ ] AC-8: RLIMIT_FSIZE exceeded ⟹ SIGXFSZ + `-EFBIG`.
- [ ] AC-9: F_SEAL_WRITE on dest ⟹ `-EPERM`.
- [ ] AC-10: Dest S_IMMUTABLE ⟹ `-EPERM`.
- [ ] AC-11: Source past EOF ⟹ returns 0.
- [ ] AC-12: Partial copy: *off_in / *off_out written back with advanced positions.
- [ ] AC-13: Same-mount mismatched idmap (idmapped mount) ⟹ `-EXDEV`.

## Architecture

```rust
#[syscall(nr = 326, abi = "sysv")]
pub fn sys_copy_file_range(
    fd_in: i32, off_in: UserPtr<i64>,
    fd_out: i32, off_out: UserPtr<i64>,
    len: usize, flags: u32,
) -> isize {
    CopyFileRange::do_copy(fd_in, off_in, fd_out, off_out, len, flags)
}
```

`CopyFileRange::do_copy(fd_in, off_in_uptr, fd_out, off_out_uptr, len, flags) -> isize`:
1. if flags != 0 { return Err(EINVAL); }
2. let file_in = current().files().get_file(fd_in).ok_or(EBADF)?;
3. if !file_in.has_mode(FMODE_READ) { return Err(EBADF); }
4. let file_out = current().files().get_file(fd_out).ok_or(EBADF)?;
5. if !file_out.has_mode(FMODE_WRITE) { return Err(EBADF); }
6. /* Regular-file check */
7. if !file_in.inode().is_reg() || !file_out.inode().is_reg() { return Err(EINVAL); }
8. /* Read offsets */
9. let in_use_fpos = off_in_uptr.is_null();
10. let out_use_fpos = off_out_uptr.is_null();
11. let mut pos_in = if in_use_fpos { file_in.f_pos.load(Ordering::Acquire) as i64 } else { off_in_uptr.copy_in()? };
12. let mut pos_out = if out_use_fpos { file_out.f_pos.load(Ordering::Acquire) as i64 } else { off_out_uptr.copy_in()? };
13. if pos_in < 0 || pos_out < 0 { return Err(EINVAL); }
14. let n = core::cmp::min(len, MAX_COPY_FILE_RANGE) as i64;
15. if pos_in.checked_add(n).is_none() || pos_out.checked_add(n).is_none() { return Err(EOVERFLOW); }
16. /* Same-fd overlap check */
17. if std::ptr::eq(file_in.inode(), file_out.inode()) && CopyFileRange::ranges_overlap(pos_in, pos_out, n) { return Err(EINVAL); }
18. /* Seals + immutable + append */
19. if file_out.inode().seals().contains(F_SEAL_WRITE) { return Err(EPERM); }
20. if file_out.inode().has_attr(S_IMMUTABLE) { return Err(EPERM); }
21. if file_out.inode().has_attr(S_APPEND) && pos_out != file_out.inode().size() as i64 { return Err(EPERM); }
22. /* Cross-mount validation */
23. CopyFileRange::validate_cross_mount(&file_in, &file_out)?;
24. /* RLIMIT_FSIZE */
25. let limit = current().rlimit(RLIMIT_FSIZE);
26. if (pos_out as u64) >= limit { send_sig(SIGXFSZ, current()); return Err(EFBIG); }
27. let n = core::cmp::min(n as u64, limit - pos_out as u64) as usize;
28. lsm::file_permission(&file_in, MAY_READ)?;
29. lsm::file_permission(&file_out, MAY_WRITE)?;
30. /* Dispatch: f_op->copy_file_range or generic_copy_file_range */
31. let copied = if let Some(op) = file_out.f_op.copy_file_range {
32.   op(&file_in, pos_in as u64, &file_out, pos_out as u64, n, flags)?
33. } else {
34.   generic_copy_file_range(&file_in, pos_in, &file_out, pos_out, n)?
35. };
36. /* Writeback offsets */
37. if in_use_fpos { file_in.f_pos.fetch_add(copied as u64, Ordering::AcqRel); }
38. else { off_in_uptr.copy_out(&(pos_in + copied as i64))?; }
39. if out_use_fpos { file_out.f_pos.fetch_add(copied as u64, Ordering::AcqRel); }
40. else { off_out_uptr.copy_out(&(pos_out + copied as i64))?; }
41. Ok(copied as isize)

`CopyFileRange::validate_cross_mount(fi, fo) -> Result<()>`:
1. if fi.inode().super_block() == fo.inode().super_block() { return Ok(()); }
2. /* Cross-superblock; require same userns idmap */
3. if fi.mnt_userns() != fo.mnt_userns() { return Err(EXDEV); }
4. /* Generic splice fallback supported since 5.3 */
5. Ok(())
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_zero` | INVARIANT | flags != 0 ⟹ -EINVAL. |
| `regular_files_only` | INVARIANT | both inodes S_ISREG. |
| `no_overlap_same_inode` | INVARIANT | same inode + overlapping ranges ⟹ -EINVAL. |
| `seals_immutable_append_honored` | INVARIANT | dest seals/attrs respected. |
| `rlimit_fsize_honored` | INVARIANT | pos_out + copied <= RLIMIT_FSIZE. |
| `offset_writeback` | INVARIANT | *off_in / *off_out advanced by copied count on return. |
| `cross_mount_idmap_validated` | INVARIANT | mismatched idmap ⟹ -EXDEV. |

### Layer 2: TLA+

`fs/copy-file-range.tla`:
- States: per-fd offsets, per-inode size, per-RLIMIT counter, per-mount idmap.
- Properties:
  - `safety_no_self_overlap` — same inode + overlap ⟹ -EINVAL.
  - `safety_offset_advance` — *off_in/*off_out updated to pos + copied.
  - `safety_seal_immutable_blocks` — dest seal/immutable ⟹ no bytes written.
  - `safety_rlimit_clamp` — never write past RLIMIT_FSIZE.
  - `safety_cross_mount_idmap` — different userns ⟹ -EXDEV.
  - `liveness_terminates` — given progress, copy completes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_copy` post: 0 <= ret <= min(len, MAX) | `CopyFileRange::do_copy` |
| `do_copy` post: ret > 0 ⟹ pos_in/pos_out advanced | `CopyFileRange::do_copy` |
| `validate_cross_mount` post: same userns ∨ -EXDEV | `CopyFileRange::validate_cross_mount` |

### Layer 4: Verus / Creusot functional

Per-`copy_file_range(2)` man-page semantic equivalence; xfstests `generic/430..435` pass; btrfs reflink verification (same data extent ID).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`copy_file_range(2)` reinforcement:

- **Per-flags == 0 strict** — defense against per-forward-compat ABI breakage.
- **Per-regular-file requirement** — defense against per-arbitrary-fd-type abuse.
- **Per-same-inode overlap check** — defense against per-data-corruption from self-copy.
- **Per-seal/immutable/append honored on dest** — defense against per-protected-file overwrite.
- **Per-RLIMIT_FSIZE + SIGXFSZ on dest** — defense against per-disk-fill DoS.
- **Per-cross-mount idmap validation** — defense against per-idmapped-mount privilege confusion.
- **Per-offset writeback before return** — defense against per-state-loss on signal.
- **Per-i64 overflow checks** — defense against per-arithmetic-wrap.

## Grsecurity / PaX surface

- **PaX UDEREF on off_in / off_out copy_in/copy_out** — defense against per-pointer kernel deref; SMAP enforced.
- **GRKERNSEC_USERCOPY_HARDEN on copy_to_user(off_*)** — bounded 8-byte copies, whitelisted slab.
- **PAX_REFCOUNT on both file->f_count** — defense against per-UAF on concurrent close+copy_file_range.
- **GRKERNSEC_DMESG suppresses copy_file_range error logs** — `-EXDEV`, `-EFBIG`, `-EPERM`, `-EOPNOTSUPP` rate-limited.
- **PaX KERNEXEC on f_op->copy_file_range dispatch** — RAP/CFI protects optional function pointer.
- **GRKERNSEC_RESLOG on cross-mount copy_file_range** — audit trail when fi.sb != fo.sb (potential container escape).
- **Cross-mount validation requires same userns idmap** — defense against per-idmapped-mount confusion; grsec hardens beyond upstream by also requiring matching `s_user_ns`.
- **GRKERNSEC_RESLOG on reflink-clone large copies** — > 10 GiB reflinks audited (potential dedup-bomb).
- **PAX_REFCOUNT on inode->i_writecount of dest** — strict.
- **PAX_HARDENED_USERCOPY** — bounds enforced on splice paths.
- **Per-len cap MAX_COPY_FILE_RANGE** — additional grsec cap (default INT_MAX/2) to limit single-syscall work.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `sendfile(2)` (covered separately).
- `splice(2)` / `tee(2)` / `vmsplice(2)` (covered separately).
- `FICLONERANGE` ioctl (covered separately).
- `FIDEDUPERANGE` ioctl (covered separately).
- Implementation code.
