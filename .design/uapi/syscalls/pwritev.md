# Tier-5 syscall: pwritev(2) — syscall 296

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/read_write.c (SYSCALL_DEFINE5(pwritev), do_pwritev, vfs_pwritev)
  - include/linux/uio.h (struct iovec, iov_iter)
  - include/uapi/linux/uio.h (UIO_MAXIOV=1024)
  - arch/x86/entry/syscalls/syscall_64.tbl (296  common  pwritev)
-->

## Summary

`pwritev(2)` performs a **positional gather write**: it writes data from `iov[]` to a file descriptor at an explicit `offset`, **without affecting the file offset**. Equivalent to `lseek` + `writev` + `lseek-back` as one atomic operation.

Note: pwritev (like pwrite) does NOT obey `O_APPEND`. Writes with O_APPEND set bypass the explicit offset and append to end of file (Linux-specific quirk; pre-3.14 ignored the offset, post-3.14 still appends; this is per the Linux man page).

Used heavily by: Postgres heap writes, MySQL InnoDB doublewrite buffer, multithreaded log writers. Critical for: per-positional write, per-thread-safe shared-fd writes.

## Signature

```c
ssize_t pwritev(int fd, const struct iovec *iov, int iovcnt, off_t offset);
```

On x86-64 the syscall takes a split 64-bit offset (`pos_l`, `pos_h`) for 32-bit ABI compatibility.

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open seekable file descriptor (must be writable). |
| `iov` | `const struct iovec *` | in | Pointer to array of `iovcnt` iovec entries. |
| `iovcnt` | `int` | in | Number of iovec entries; range [0, UIO_MAXIOV=1024]. |
| `offset` | `off_t` | in | Byte offset from start of file. Must be >= 0. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of bytes written. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not valid / not writable. |
| `EINVAL` | `iovcnt` out of range; iov_len overflow; `offset < 0`; fd unseekable. |
| `EFAULT` | iov or iov_base out of accessible address space. |
| `EFBIG` | offset + total > RLIMIT_FSIZE or filesystem limit; SIGXFSZ raised. |
| `ENOSPC` | Device full. |
| `EDQUOT` | Quota exceeded. |
| `EIO` | Low-level I/O error. |
| `EAGAIN` | Non-blocking would block. |
| `EINTR` | Signal delivered before any byte written. |
| `ESPIPE` | fd is a pipe/socket/FIFO. |
| `EPERM` | F_SEAL_WRITE / F_SEAL_FUTURE_WRITE set. |
| `EOVERFLOW` | offset + sum(iov_len) overflows. |

## ABI surface

```text
__NR_pwritev  (x86_64) = 296
__NR_pwritev  (arm64)  = 70
__NR_pwritev  (riscv)  = 70
__NR_pwritev  (i386)   = 334

UIO_MAXIOV  = 1024
RLIMIT_FSIZE — enforced; exceed ⟹ SIGXFSZ + -EFBIG.
```

## Compatibility contract

REQ-1: Syscall number is **296** on x86_64. ABI-stable.

REQ-2: `offset < 0` ⟹ `-EINVAL`.

REQ-3: `iovcnt` and iov_len validation identical to `writev(2)`.

REQ-4: File offset (f_pos) is **NOT** modified.

REQ-5: `fd` MUST be seekable; non-seekable ⟹ `-ESPIPE`.

REQ-6: Per-O_APPEND quirk: Linux ignores the user-supplied offset and appends to end of file. POSIX says behavior is undefined; Linux chooses "append wins". Use `fcntl(F_SETFL)` to clear O_APPEND for strict positional write.

REQ-7: Per-RLIMIT_FSIZE: `pos + count > RLIMIT_FSIZE` ⟹ truncated; `pos >= RLIMIT_FSIZE` ⟹ SIGXFSZ + `-EFBIG`.

REQ-8: Per-F_SEAL_WRITE: blocks subsequent pwritev ⟹ `-EPERM`.

REQ-9: Per-O_DIRECT: `offset`, every `iov_base`, every `iov_len` MUST be block-aligned.

REQ-10: Per-f_op->write_iter: preferred; supplies kiocb.ki_pos = offset.

REQ-11: Per-32-bit ABI: offset split into pos_l/pos_h; glibc wrapper assembles.

REQ-12: Per-LSM: `security_file_permission(file, MAY_WRITE)`.

REQ-13: Per-fsync: pwritev does NOT imply sync; data may sit in page cache.

REQ-14: Per-mandatory locking (legacy): F_WRLCK conflict ⟹ `-EAGAIN`.

REQ-15: Per-concurrent pwritev on overlapping ranges: kernel does NOT serialize (no per-range mutex); ordering is page-cache implementation defined.

## Acceptance Criteria

- [ ] AC-1: `pwritev(fd, iov, 1, 100)` writes at offset 100 without moving f_pos.
- [ ] AC-2: After pwritev: `lseek(fd, 0, SEEK_CUR)` returns prior value.
- [ ] AC-3: `pwritev(fd, iov, 1, -1)` ⟹ `-EINVAL`.
- [ ] AC-4: `pwritev(pipe_wr, iov, 1, 0)` ⟹ `-ESPIPE`.
- [ ] AC-5: `pwritev(fd, NULL, 1, 0)` ⟹ `-EFAULT`.
- [ ] AC-6: O_APPEND fd: pwritev ignores offset, appends to EOF.
- [ ] AC-7: RLIMIT_FSIZE exceeded ⟹ SIGXFSZ + `-EFBIG`.
- [ ] AC-8: F_SEAL_WRITE ⟹ `-EPERM`.
- [ ] AC-9: O_DIRECT misaligned ⟹ `-EINVAL`.
- [ ] AC-10: Two threads pwritev'ing non-overlapping ranges: both succeed.
- [ ] AC-11: 32-bit compat pwritev split offset assembles correctly.

## Architecture

```rust
#[syscall(nr = 296, abi = "sysv")]
pub fn sys_pwritev(fd: i32, iov: UserPtr<Iovec>, iovcnt: i32, offset: i64) -> isize {
    Pwritev::do_pwritev(fd, iov, iovcnt, offset)
}
```

`Pwritev::do_pwritev(fd, iov, iovcnt, offset) -> isize`:
1. if offset < 0 { return Err(EINVAL); }
2. if iovcnt < 0 || iovcnt as usize > UIO_MAXIOV { return Err(EINVAL); }
3. let file = current().files().get_file(fd).ok_or(EBADF)?;
4. if !file.has_mode(FMODE_WRITE) { return Err(EBADF); }
5. if !file.f_op.has_llseek() { return Err(ESPIPE); }
6. if file.inode().seals().contains(F_SEAL_WRITE) { return Err(EPERM); }
7. let mut stack = [Iovec::zero(); UIO_FASTIOV];
8. let mut heap: Option<Vec<Iovec>> = None;
9. let kiov = Iov::import(iov, iovcnt as usize, &mut stack, &mut heap)?;
10. let total = Iov::sum_len(kiov)?;
11. /* Linux O_APPEND quirk: ignore offset, use EOF */
12. let pos = if file.has_flag(O_APPEND) {
13.   file.inode().size_locked_for_append()
14. } else {
15.   if offset.checked_add(total as i64).is_none() { return Err(EOVERFLOW); }
16.   offset
17. };
18. /* RLIMIT_FSIZE */
19. let limit = current().rlimit(RLIMIT_FSIZE);
20. if pos as u64 >= limit { send_sig(SIGXFSZ, current()); return Err(EFBIG); }
21. let count = core::cmp::min(total as u64, limit - pos as u64) as usize;
22. lsm::file_permission(&file, MAY_WRITE)?;
23. let mut iter = IovIter::from_iov(WRITE, kiov);
24. iter.truncate(count);
25. let nr = file.write_iter_at(&mut iter, pos)?;
26. /* DO NOT update f_pos */
27. Ok(nr)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `offset_nonneg` | INVARIANT | offset >= 0. |
| `f_pos_unchanged` | INVARIANT | file->f_pos identical pre/post. |
| `seal_write_blocks` | INVARIANT | F_SEAL_WRITE ⟹ -EPERM. |
| `rlimit_fsize_honored` | INVARIANT | pos + ret <= RLIMIT_FSIZE. |
| `o_append_overrides_offset` | INVARIANT | O_APPEND ⟹ pos = i_size. |
| `seekable_required` | INVARIANT | non-seekable fd ⟹ -ESPIPE. |

### Layer 2: TLA+

`fs/pwritev.tla`:
- States: per-thread offset args, per-fd f_pos, per-RLIMIT counter, per-inode seals.
- Properties:
  - `safety_f_pos_invariant` — pwritev never mutates f_pos.
  - `safety_o_append_quirk` — O_APPEND fd ⟹ pos = i_size regardless of offset arg.
  - `safety_rlimit_clamp` — never write past RLIMIT_FSIZE.
  - `safety_seal_write_blocks` — F_SEAL_WRITE ⟹ -EPERM.
  - `liveness_pwritev_terminates_or_signals` — given drain, returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pwritev` post: ret >= 0 ⟹ f_pos unchanged | `Pwritev::do_pwritev` |
| `do_pwritev` post: 0 <= ret <= sum(iov_len) | `Pwritev::do_pwritev` |
| `do_pwritev` post: O_APPEND ⟹ data appended | `Pwritev::do_pwritev` |

### Layer 4: Verus / Creusot functional

Per-`pwritev(2)` man-page semantic equivalence; LTP `pwritev01..pwritev03` pass; O_APPEND quirk test matches Linux behavior; memfd F_SEAL_WRITE test passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pwritev(2)` reinforcement:

- **Per-offset >= 0 check** — defense against per-negative-offset arithmetic surprise.
- **Per-RLIMIT_FSIZE + SIGXFSZ** — defense against per-file-size resource abuse.
- **Per-F_SEAL_WRITE honored** — defense against per-memfd tampering.
- **Per-f_pos invariant** — defense against per-shared-fd thread interference.
- **Per-O_DIRECT alignment** — defense against per-misaligned DMA.
- **Per-i64 overflow on offset+total** — defense against per-arithmetic-wrap.
- **Per-O_APPEND quirk documented + enforced** — defense against per-spec-mismatch silent overwrite.

## Grsecurity / PaX surface

- **PaX UDEREF on iov copy_from_user** — defense against per-crafted-iov kernel deref; SMAP enforced.
- **GRKERNSEC_USERCOPY_HARDEN on copy_from_user(iov)** — bounded by `iovcnt * sizeof(iovec)`.
- **UIO_MAXIOV=1024 unconditional cap** — defense against per-iov amplification.
- **PAX_REFCOUNT on file->f_count** — defense against per-UAF on concurrent close+pwritev.
- **GRKERNSEC_DMESG suppresses pwritev error logs** — `-EFBIG`, `-EPIPE`, `-EINVAL` rate-limited.
- **PaX KERNEXEC on f_op->write_iter** — RAP/CFI protects dispatch.
- **GRKERNSEC_RESLOG on F_SEAL_WRITE breach attempts** — `-EPERM` logged.
- **GRKERNSEC_RESLOG on offset > i_size + 4 GiB** — sparse-allocation amplification audited.
- **PAX_REFCOUNT on inode->i_writecount** — strict.
- **Per-iovec stack-allocated for iovcnt <= 8** — defense against per-kmalloc DoS.
- **PaX FORTIFY_SOURCE on iov_iter** — bounds enforced in `copy_page_from_iter`.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `writev(2)` (covered in `writev.md`).
- `preadv(2)` (covered in `preadv.md`).
- `pwritev2(2)` (covered in `pwritev2.md`).
- `pwrite64(2)` (covered in `pwrite64.md`).
- Implementation code.
