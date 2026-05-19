# Tier-5 syscall: writev(2) — syscall 20

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/read_write.c (SYSCALL_DEFINE3(writev), do_writev, vfs_writev, do_iter_write)
  - include/linux/uio.h (struct iovec, iov_iter)
  - include/uapi/linux/uio.h (UIO_MAXIOV=1024, UIO_FASTIOV=8)
  - arch/x86/entry/syscalls/syscall_64.tbl (20  common  writev)
-->

## Summary

`writev(2)` performs a **gather write**: it writes data from the buffers described by `iov[]` to a file descriptor in a single syscall, atomically with respect to other writers on the same fd (for regular files within a single underlying f_op->write_iter call). The kernel returns once all bytes have been queued (regular files) or as many as fit (pipes, sockets, full disk).

Used heavily by: HTTP servers building header+body in two iovecs, log writers (timestamp+payload), QEMU disk image flushing, libuv. Critical for: per-gather throughput, per-atomic-multi-buffer-write, per-iovec ordering.

## Signature

```c
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```

```c
struct iovec {
    void  *iov_base;   /* starting address */
    size_t iov_len;    /* number of bytes to transfer */
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor (must be writable). |
| `iov` | `const struct iovec *` | in | Pointer to array of `iovcnt` iovec entries (read by kernel). |
| `iovcnt` | `int` | in | Number of iovec entries; range [0, UIO_MAXIOV=1024]. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of bytes written (sum may be short on partial pipe/socket fills). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not a valid descriptor or not open for write. |
| `EINVAL` | `iovcnt < 0` or `iovcnt > UIO_MAXIOV`; or sum of `iov_len` overflows `ssize_t`; or fd unsuitable. |
| `EFAULT` | `iov` array or any `iov_base` outside accessible address space. |
| `EFBIG` | Resulting file size would exceed RLIMIT_FSIZE or filesystem limit; SIGXFSZ also raised. |
| `ENOSPC` | Device full. |
| `EDQUOT` | User/group quota exceeded. |
| `EIO` | Low-level I/O error. |
| `EAGAIN` | Non-blocking fd; would block. |
| `EINTR` | Signal delivered before any byte written. |
| `EPIPE` | Pipe/socket closed by peer; SIGPIPE also raised. |
| `EDESTADDRREQ` | Connectionless socket without prior `connect(2)`. |
| `EPERM` | File seal F_SEAL_WRITE / F_SEAL_FUTURE_WRITE blocks. |

## ABI surface

```text
__NR_writev  (x86_64) = 20
__NR_writev  (arm64)  = 66
__NR_writev  (riscv)  = 66
__NR_writev  (i386)   = 146

UIO_MAXIOV    = 1024      /* hard cap on iovcnt */
UIO_FASTIOV   = 8         /* on-stack iovec count threshold */
PIPE_BUF      = 4096      /* atomic-write boundary for pipes */
RLIMIT_FSIZE  = file size resource limit; exceed ⟹ SIGXFSZ + -EFBIG
```

## Compatibility contract

REQ-1: Syscall number is **20** on x86_64. ABI-stable.

REQ-2: `iovcnt < 0` ⟹ `-EINVAL`. `iovcnt > UIO_MAXIOV` ⟹ `-EINVAL`. `iovcnt == 0` ⟹ returns 0 successfully.

REQ-3: Each `iov[i].iov_len` MUST be representable as `size_t`; sum MUST fit in `ssize_t`.

REQ-4: Kernel copies `iov[]` into kernel buffer (UIO_FASTIOV on-stack, else kmalloc) before transferring data; user mutation of `iov[]` during syscall has no effect.

REQ-5: Per-`pipe(7)` `PIPE_BUF`: total writev <= PIPE_BUF is atomic — no interleave with other writers; > PIPE_BUF may interleave.

REQ-6: Per-O_APPEND: kernel acquires `i_rwsem` and seeks to `i_size` before writing; safe with concurrent appenders.

REQ-7: Per-RLIMIT_FSIZE: if `pos + count > RLIMIT_FSIZE` ⟹ truncate write to fit; if `pos >= RLIMIT_FSIZE` ⟹ raise `SIGXFSZ` and return `-EFBIG`.

REQ-8: Per-O_NONBLOCK: if would block ⟹ `-EAGAIN`.

REQ-9: Per-SIGPIPE: writing to broken pipe ⟹ kernel raises SIGPIPE then returns `-EPIPE` (unless SIGPIPE blocked/ignored).

REQ-10: Per-`fcntl(F_ADD_SEALS, F_SEAL_WRITE)`: subsequent writev ⟹ `-EPERM`.

REQ-11: Per-O_DIRECT: every `iov_base` and `iov_len` MUST be block-aligned; misaligned ⟹ `-EINVAL`.

REQ-12: Per-`f_op->write_iter`: preferred over legacy `f_op->write`. Synthesized iov_iter passed in.

REQ-13: Per-fsync semantics: writev does NOT imply sync; data may sit in page cache until fsync/fdatasync.

REQ-14: Per-32-bit compat: `compat_sys_writev` translates `compat_iovec` (8-byte entries) into native; same UIO_MAXIOV cap.

REQ-15: Per-LSM hook: `security_file_permission(file, MAY_WRITE)` invoked before transfer.

## Acceptance Criteria

- [ ] AC-1: `writev(fd, iov={4,4,4}, 3)` writes 12 bytes in order.
- [ ] AC-2: `writev(fd, iov, 1025)` ⟹ `-EINVAL`.
- [ ] AC-3: `writev(-1, iov, 1)` ⟹ `-EBADF`.
- [ ] AC-4: `writev(fd, NULL, 1)` ⟹ `-EFAULT`.
- [ ] AC-5: `writev(fd, iov, 0)` ⟹ returns 0.
- [ ] AC-6: O_APPEND + concurrent writev: no overlap; both writes appear contiguously.
- [ ] AC-7: Pipe writev with total <= PIPE_BUF: atomic vs other writers.
- [ ] AC-8: RLIMIT_FSIZE exceeded ⟹ SIGXFSZ + `-EFBIG`.
- [ ] AC-9: F_SEAL_WRITE set ⟹ `-EPERM`.
- [ ] AC-10: Broken pipe ⟹ SIGPIPE + `-EPIPE`.
- [ ] AC-11: O_DIRECT misaligned iov_base ⟹ `-EINVAL`.
- [ ] AC-12: 32-bit compat writev via `compat_iovec` works identically.

## Architecture

```rust
#[syscall(nr = 20, abi = "sysv")]
pub fn sys_writev(fd: i32, iov: UserPtr<Iovec>, iovcnt: i32) -> isize {
    Writev::do_writev(fd, iov, iovcnt)
}
```

`Writev::do_writev(fd, iov, iovcnt) -> isize`:
1. if iovcnt < 0 || iovcnt as usize > UIO_MAXIOV { return Err(EINVAL); }
2. let file = current().files().get_file(fd).ok_or(EBADF)?;
3. if !file.has_mode(FMODE_WRITE) { return Err(EBADF); }
4. /* Seal check */
5. if file.inode().seals().contains(F_SEAL_WRITE) { return Err(EPERM); }
6. /* Import iovec array */
7. let mut stack = [Iovec::zero(); UIO_FASTIOV];
8. let mut heap: Option<Vec<Iovec>> = None;
9. let kiov = Writev::import_iov(iov, iovcnt as usize, &mut stack, &mut heap)?;
10. let total = Writev::sum_iov_len(kiov)?;
11. /* RLIMIT_FSIZE check */
12. let pos = if file.has_flag(O_APPEND) {
13.   file.inode().size_locked_for_append()
14. } else {
15.   file.f_pos.load(Ordering::Acquire)
16. };
17. let limit = current().rlimit(RLIMIT_FSIZE);
18. if pos >= limit { send_sig(SIGXFSZ, current()); return Err(EFBIG); }
19. let count = core::cmp::min(total as u64, limit - pos) as usize;
20. /* LSM hook */
21. lsm::file_permission(&file, MAY_WRITE)?;
22. /* Build iter and dispatch */
23. let mut iter = IovIter::from_iov(WRITE, kiov);
24. iter.truncate(count);
25. let nr = file.write_iter(&mut iter, pos)?;
26. if nr > 0 && !file.has_flag(O_APPEND) {
27.   file.f_pos.store(pos + nr as u64, Ordering::Release);
28. }
29. Ok(nr)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `iovcnt_bounded` | INVARIANT | iovcnt in [0, UIO_MAXIOV]. |
| `iov_sum_no_overflow` | INVARIANT | sum of iov_len <= SSIZE_MAX. |
| `rlimit_fsize_honored` | INVARIANT | pos + ret <= RLIMIT_FSIZE. |
| `seal_write_blocks` | INVARIANT | F_SEAL_WRITE ⟹ -EPERM. |
| `o_append_atomic` | INVARIANT | O_APPEND: pos read under i_rwsem. |
| `fmode_write_required` | INVARIANT | fd lacks FMODE_WRITE ⟹ -EBADF. |

### Layer 2: TLA+

`fs/writev.tla`:
- States: per-fd offset, per-pipe buffer fill, per-RLIMIT counter.
- Properties:
  - `safety_pipe_buf_atomic` — writev with total <= PIPE_BUF: no interleave on pipe.
  - `safety_o_append_no_clobber` — concurrent O_APPEND writevs concatenate.
  - `safety_rlimit_clamp` — never write past RLIMIT_FSIZE.
  - `liveness_writev_terminates_or_signals` — given drain, writev returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_writev` post: 0 <= ret <= sum(iov_len) | `Writev::do_writev` |
| `import_iov` post: kernel slice independent of user iov | `Writev::import_iov` |
| `sum_iov_len` post: result <= SSIZE_MAX | `Writev::sum_iov_len` |

### Layer 4: Verus / Creusot functional

Per-`writev(2)` man-page semantic equivalence; LTP `writev01..writev07` pass; pipe atomicity reproducer passes; F_SEAL_WRITE memfd test passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`writev(2)` reinforcement:

- **Per-UIO_MAXIOV=1024 hard cap** — defense against per-iov-cnt-DoS.
- **Per-iov array kernel-copied** — defense against per-TOCTOU (user mutating iov mid-syscall).
- **Per-sum-overflow check** — defense against per-ssize_t wrap.
- **Per-RLIMIT_FSIZE + SIGXFSZ** — defense against per-file-size resource abuse.
- **Per-F_SEAL_WRITE honored** — defense against per-memfd tampering.
- **Per-O_DIRECT alignment checked** — defense against per-misaligned DMA corruption.

## Grsecurity / PaX surface

- **PaX UDEREF on iov + iov_base copy_from_user** — defense against per-crafted-iovec kernel deref; SMAP enforced.
- **GRKERNSEC_USERCOPY_HARDEN on copy_from_user(iov)** — bounded `iovcnt * sizeof(iovec)`, whitelisted slab.
- **UIO_MAXIOV=1024 unconditional cap** — no Kconfig knob; defense against per-iov-cnt amplification.
- **PAX_REFCOUNT on file->f_count** — defense against per-UAF on concurrent close+writev race.
- **GRKERNSEC_DMESG suppresses writev error spam** — `-EFAULT`, `-EFBIG`, `-EPIPE` errors visible only to CAP_SYSLOG.
- **PaX KERNEXEC on f_op->write_iter dispatch** — RAP/CFI protects function pointer call.
- **GRKERNSEC_PROC_FILESIZE_LOG** — write-amplification (large total > 1 GiB) logged in audit.
- **PAX_REFCOUNT on inode->i_writecount** — strict tracking of writeable references.
- **Per-iovec stack-allocated for iovcnt <= 8** — defense against per-kmalloc DoS.
- **GRKERNSEC_RESLOG on F_SEAL_WRITE breach attempts** — `-EPERM` logged with task identity.
- **PaX FORTIFY_SOURCE on iov_iter copy paths** — bounds enforced in `copy_page_from_iter`.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `readv(2)` (covered in `readv.md`).
- `pwritev(2)` / `pwritev2(2)` (covered separately).
- `pwrite64(2)` (covered separately).
- vmsplice / process_vm_writev (covered separately).
- Implementation code.
