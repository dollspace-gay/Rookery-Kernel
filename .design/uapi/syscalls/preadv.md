# Tier-5 syscall: preadv(2) — syscall 295

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/read_write.c (SYSCALL_DEFINE5(preadv), do_preadv, vfs_preadv)
  - include/linux/uio.h (struct iovec, iov_iter)
  - include/uapi/linux/uio.h (UIO_MAXIOV=1024)
  - arch/x86/entry/syscalls/syscall_64.tbl (295  common  preadv)
-->

## Summary

`preadv(2)` performs a **positional scatter read**: it reads from a file descriptor at an explicit `offset` into the buffers described by `iov[]`, **without affecting the file offset**. Equivalent to `lseek` + `readv` + `lseek-back` as one atomic operation, eliminating the race between threads sharing an fd.

Used heavily by: Postgres (concurrent WAL/heap reads), MySQL InnoDB, multithreaded log indexers. Critical for: per-positional read, per-thread-safe shared-fd I/O.

## Signature

```c
ssize_t preadv(int fd, const struct iovec *iov, int iovcnt, off_t offset);
```

On x86-64 the syscall takes a split 64-bit offset (`pos_l`, `pos_h`); glibc's `preadv()` wrapper assembles them.

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open seekable file descriptor (must be readable). |
| `iov` | `const struct iovec *` | in | Pointer to array of `iovcnt` iovec entries. |
| `iovcnt` | `int` | in | Number of iovec entries; range [0, UIO_MAXIOV=1024]. |
| `offset` | `off_t` | in | Byte offset from start of file. Must be >= 0. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of bytes read. |
| `0` | EOF at requested offset. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not valid / not readable. |
| `EINVAL` | `iovcnt < 0` or `> UIO_MAXIOV`; iov_len overflow; **`offset < 0`**; fd is unseekable (pipe, socket). |
| `EFAULT` | `iov` array or any `iov_base` outside accessible address space. |
| `EAGAIN` | Non-blocking and would block (rare for seekable fd). |
| `EINTR` | Signal delivered before any byte read. |
| `EIO` | Low-level I/O error. |
| `EISDIR` | `fd` refers to a directory. |
| `ESPIPE` | fd is a pipe/socket/FIFO (unseekable). |
| `EOVERFLOW` | offset + sum(iov_len) overflows. |

## ABI surface

```text
__NR_preadv  (x86_64) = 295
__NR_preadv  (arm64)  = 69
__NR_preadv  (riscv)  = 69
__NR_preadv  (i386)   = 333  /* uses pos_l + pos_h pair */

UIO_MAXIOV = 1024

/* On 32-bit ABIs, offset passed as two longs (low, high)
   to align the 64-bit value into a single register pair. */
```

## Compatibility contract

REQ-1: Syscall number is **295** on x86_64. ABI-stable.

REQ-2: `offset < 0` ⟹ `-EINVAL` (signed comparison; off_t is signed).

REQ-3: `iovcnt` range and iov_len overflow checks identical to `readv(2)`.

REQ-4: File offset (f_pos) is **NOT** modified. Multiple threads sharing an fd can issue preadv concurrently at different offsets safely.

REQ-5: `fd` MUST be seekable (have `f_op->llseek != no_llseek`); otherwise `-ESPIPE`.

REQ-6: Per-O_NONBLOCK: rare on regular files, but if the underlying inode is on a network FS that may block, `-EAGAIN` possible.

REQ-7: Per-O_DIRECT: `offset`, every `iov_base`, every `iov_len` MUST be block-aligned (typically 512 / 4096).

REQ-8: Per-signal: identical to readv — signal before any byte ⟹ `-EINTR`; else short return.

REQ-9: Per-32-bit ABI: offset split into `pos_l, pos_h`; user code SHOULD call libc wrapper, not syscall directly.

REQ-10: Per-`f_op->read_iter`: preferred path; supplies `kiocb.ki_pos = offset` for positional read.

REQ-11: Per-RLIMIT: no FSIZE limit on read; RLIMIT_NOFILE applies to fd availability.

REQ-12: Per-locking: no per-file mutex held across read (unlike O_APPEND writes); concurrent preadv to non-overlapping ranges is parallel-safe.

REQ-13: Per-block-device fd: offset semantics use byte-addressed positions in the device.

REQ-14: Per-LSM: `security_file_permission(file, MAY_READ)` invoked.

REQ-15: Per-mandatory locking (legacy, removed in 5.15): F_RDLCK conflict ⟹ `-EAGAIN`/`-EWOULDBLOCK`.

## Acceptance Criteria

- [ ] AC-1: `preadv(fd, iov, 1, 0)` reads from offset 0 without moving f_pos.
- [ ] AC-2: After preadv: `lseek(fd, 0, SEEK_CUR)` returns prior value (unchanged).
- [ ] AC-3: `preadv(fd, iov, 1, -1)` ⟹ `-EINVAL`.
- [ ] AC-4: `preadv(pipe_rd, iov, 1, 0)` ⟹ `-ESPIPE`.
- [ ] AC-5: `preadv(fd, iov, 1025, 0)` ⟹ `-EINVAL`.
- [ ] AC-6: `preadv(fd, NULL, 1, 0)` ⟹ `-EFAULT`.
- [ ] AC-7: O_DIRECT misaligned offset ⟹ `-EINVAL`.
- [ ] AC-8: Two threads preadv'ing different offsets on same fd: both succeed.
- [ ] AC-9: Past-EOF offset ⟹ returns 0.
- [ ] AC-10: Signal before any byte ⟹ `-EINTR`.
- [ ] AC-11: 32-bit compat preadv with split pos_l/pos_h assembles correctly.

## Architecture

```rust
#[syscall(nr = 295, abi = "sysv")]
pub fn sys_preadv(fd: i32, iov: UserPtr<Iovec>, iovcnt: i32, offset: i64) -> isize {
    Preadv::do_preadv(fd, iov, iovcnt, offset)
}
```

`Preadv::do_preadv(fd, iov, iovcnt, offset) -> isize`:
1. if offset < 0 { return Err(EINVAL); }
2. if iovcnt < 0 || iovcnt as usize > UIO_MAXIOV { return Err(EINVAL); }
3. let file = current().files().get_file(fd).ok_or(EBADF)?;
4. if !file.has_mode(FMODE_READ) { return Err(EBADF); }
5. if !file.f_op.has_llseek() { return Err(ESPIPE); }
6. /* Import iov */
7. let mut stack = [Iovec::zero(); UIO_FASTIOV];
8. let mut heap: Option<Vec<Iovec>> = None;
9. let kiov = Iov::import(iov, iovcnt as usize, &mut stack, &mut heap)?;
10. let total = Iov::sum_len(kiov)?;
11. /* Overflow check */
12. if offset.checked_add(total as i64).is_none() { return Err(EOVERFLOW); }
13. /* LSM */
14. lsm::file_permission(&file, MAY_READ)?;
15. /* Build iter; dispatch */
16. let mut iter = IovIter::from_iov(READ, kiov);
17. /* kiocb with explicit pos */
18. let nr = file.read_iter_at(&mut iter, offset)?;
19. /* DO NOT update f_pos */
20. Ok(nr)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `offset_nonneg` | INVARIANT | offset >= 0. |
| `f_pos_unchanged` | INVARIANT | file->f_pos identical pre/post. |
| `iovcnt_bounded` | INVARIANT | iovcnt in [0, UIO_MAXIOV]. |
| `seekable_required` | INVARIANT | non-seekable fd ⟹ -ESPIPE. |
| `iov_sum_offset_no_overflow` | INVARIANT | offset + sum(iov_len) <= i64::MAX. |
| `fmode_read_required` | INVARIANT | fd lacks FMODE_READ ⟹ -EBADF. |

### Layer 2: TLA+

`fs/preadv.tla`:
- States: per-thread offset args, per-fd f_pos, per-fd-shared-by-N-threads.
- Properties:
  - `safety_f_pos_invariant` — preadv never mutates f_pos.
  - `safety_concurrent_threads_independent` — N threads preadv'ing at distinct offsets all succeed.
  - `safety_eof_short_or_zero` — past-EOF returns 0; partial-EOF returns short.
  - `liveness_terminates` — given data, preadv returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_preadv` post: ret >= 0 ⟹ f_pos unchanged | `Preadv::do_preadv` |
| `do_preadv` post: 0 <= ret <= sum(iov_len) | `Preadv::do_preadv` |
| `import` post: kernel slice independent of user iov | `Iov::import` |

### Layer 4: Verus / Creusot functional

Per-`preadv(2)` man-page semantic equivalence; LTP `preadv01..preadv03` pass; multithreaded shared-fd test passes (no f_pos drift).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`preadv(2)` reinforcement:

- **Per-offset >= 0 check** — defense against per-negative-offset arithmetic surprise.
- **Per-ESPIPE on unseekable fd** — defense against per-pipe positional-read misuse.
- **Per-f_pos invariant** — defense against per-shared-fd thread interference.
- **Per-UIO_MAXIOV=1024 cap** — defense against per-iov-cnt DoS.
- **Per-O_DIRECT alignment** — defense against per-misaligned DMA.
- **Per-i64 overflow check on offset+total** — defense against per-arithmetic-wrap.

## Grsecurity / PaX surface

- **PaX UDEREF on iov copy_from_user** — defense against per-crafted-iov kernel deref; SMAP enforced.
- **GRKERNSEC_USERCOPY_HARDEN on copy_from_user(iov)** — whitelisted slab, bounded by `iovcnt * sizeof(iovec)`.
- **UIO_MAXIOV=1024 unconditional cap** — defense against per-iov amplification.
- **PAX_REFCOUNT on file->f_count** — defense against per-UAF on concurrent close+preadv race.
- **GRKERNSEC_DMESG suppresses preadv error logs** — `-EINVAL`, `-EFAULT`, `-ESPIPE` rate-limited.
- **PaX KERNEXEC on f_op->read_iter** — RAP/CFI protects dispatch.
- **GRKERNSEC_RESLOG on offset > i_size + 1 GiB** — sparse-file probing attempt audited.
- **Per-fd permission re-checked on every preadv** — defense against per-fd-revocation race (LSM).
- **PaX FORTIFY_SOURCE on iov_iter copy paths** — bounds enforced in `copy_page_to_iter`.
- **GRKERNSEC_PROC_OFFSETS** — kernel-side offset hidden from `/proc/<pid>/fdinfo` for non-owner; preadv positional reads not visible in fdinfo (no f_pos change).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `readv(2)` (covered in `readv.md`).
- `pwritev(2)` (covered in `pwritev.md`).
- `preadv2(2)` (covered in `preadv2.md`).
- `pread64(2)` (covered separately).
- Implementation code.
