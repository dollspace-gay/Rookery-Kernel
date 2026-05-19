---
title: "Tier-5 syscall: pwritev2(2) — syscall 328"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pwritev2(2)` extends `pwritev(2)` with a **`flags`** argument controlling per-call write semantics: durability (`RWF_DSYNC` / `RWF_SYNC`), polling (`RWF_HIPRI`), non-blocking (`RWF_NOWAIT`), per-call append (`RWF_APPEND`), per-call non-append override (`RWF_NOAPPEND`).

Special-case: `offset == -1` ⟹ behave as `writev(2)` (advance f_pos).

Used heavily by: RocksDB, Postgres for selective fsync per write, journal-writers needing per-record DSYNC without setting O_DSYNC globally. Critical for: per-record durability, per-flag NOWAIT writes from io_uring fallbacks.

### Acceptance Criteria

- [ ] AC-1: `pwritev2(fd, iov, 1, 0, 0)` writes at offset 0 without moving f_pos.
- [ ] AC-2: `pwritev2(fd, iov, 1, -1, 0)` writes at f_pos and advances it.
- [ ] AC-3: `pwritev2(fd, iov, 1, 0, RWF_DSYNC)` writes + fdatasync atomically.
- [ ] AC-4: `pwritev2(fd, iov, 1, 0, RWF_APPEND)` writes at i_size; offset ignored.
- [ ] AC-5: `pwritev2(fd_with_O_APPEND, iov, 1, 100, RWF_NOAPPEND)` writes at offset 100.
- [ ] AC-6: `RWF_APPEND | RWF_NOAPPEND` ⟹ `-EINVAL`.
- [ ] AC-7: `RWF_NOWAIT` on ext4 page-cache write needing allocation ⟹ `-EAGAIN`.
- [ ] AC-8: `RWF_NOWAIT` on filesystem without FMODE_NOWAIT ⟹ `-EOPNOTSUPP`.
- [ ] AC-9: Unknown flag bit ⟹ `-EINVAL`.
- [ ] AC-10: `pwritev2(pipe_wr, iov, 1, 0, 0)` ⟹ `-ESPIPE`.
- [ ] AC-11: `pwritev2(pipe_wr, iov, 1, -1, 0)` succeeds (pipe writev-like).
- [ ] AC-12: F_SEAL_WRITE ⟹ `-EPERM`.

### Architecture

```rust
#[syscall(nr = 328, abi = "sysv")]
pub fn sys_pwritev2(fd: i32, iov: UserPtr<Iovec>, iovcnt: i32, offset: i64, flags: i32) -> isize {
    Pwritev2::do_pwritev2(fd, iov, iovcnt, offset, flags as u32)
}
```

`Pwritev2::do_pwritev2(fd, iov, iovcnt, offset, flags) -> isize`:
1. if flags & !RWF_SUPPORTED != 0 { return Err(EINVAL); }
2. if flags & RWF_APPEND != 0 && flags & RWF_NOAPPEND != 0 { return Err(EINVAL); }
3. if offset < -1 { return Err(EINVAL); }
4. if iovcnt < 0 || iovcnt as usize > UIO_MAXIOV { return Err(EINVAL); }
5. let file = current().files().get_file(fd).ok_or(EBADF)?;
6. if !file.has_mode(FMODE_WRITE) { return Err(EBADF); }
7. if flags & RWF_NOWAIT != 0 && !file.has_mode(FMODE_NOWAIT) { return Err(EOPNOTSUPP); }
8. if file.inode().seals().contains(F_SEAL_WRITE) { return Err(EPERM); }
9. let use_fpos = offset == -1;
10. let force_append = flags & RWF_APPEND != 0;
11. if !use_fpos && !force_append && !file.f_op.has_llseek() { return Err(ESPIPE); }
12. let mut stack = [Iovec::zero(); UIO_FASTIOV];
13. let mut heap: Option<Vec<Iovec>> = None;
14. let kiov = Iov::import(iov, iovcnt as usize, &mut stack, &mut heap)?;
15. let total = Iov::sum_len(kiov)?;
16. /* Compute pos */
17. let pos = if force_append || (file.has_flag(O_APPEND) && flags & RWF_NOAPPEND == 0) {
18.   file.inode().size_locked_for_append()
19. } else if use_fpos {
20.   file.f_pos.load(Ordering::Acquire) as i64
21. } else {
22.   offset
23. };
24. /* RLIMIT_FSIZE */
25. let limit = current().rlimit(RLIMIT_FSIZE);
26. if (pos as u64) >= limit { send_sig(SIGXFSZ, current()); return Err(EFBIG); }
27. let count = core::cmp::min(total as u64, limit - pos as u64) as usize;
28. lsm::file_permission(&file, MAY_WRITE)?;
29. /* Build iter + kiocb */
30. let mut iter = IovIter::from_iov(WRITE, kiov);
31. iter.truncate(count);
32. let mut kiocb_flags = 0u32;
33. if flags & RWF_HIPRI != 0 { kiocb_flags |= IOCB_HIPRI; }
34. if flags & RWF_DSYNC != 0 { kiocb_flags |= IOCB_DSYNC; }
35. if flags & RWF_SYNC  != 0 { kiocb_flags |= IOCB_SYNC; }
36. if flags & RWF_NOWAIT != 0 { kiocb_flags |= IOCB_NOWAIT; }
37. if force_append { kiocb_flags |= IOCB_APPEND; }
38. let nr = file.write_iter_at_with_flags(&mut iter, pos, kiocb_flags)?;
39. if use_fpos && nr > 0 { file.f_pos.fetch_add(nr as u64, Ordering::AcqRel); }
40. Ok(nr)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | unknown bits ⟹ -EINVAL. |
| `append_noappend_mutex` | INVARIANT | both set ⟹ -EINVAL. |
| `nowait_no_block` | INVARIANT | RWF_NOWAIT ⟹ no extent-alloc block. |
| `dsync_means_fdatasync` | INVARIANT | RWF_DSYNC ⟹ writeback completes before return. |
| `rlimit_fsize_honored` | INVARIANT | pos + ret <= RLIMIT_FSIZE. |
| `seal_write_blocks` | INVARIANT | F_SEAL_WRITE ⟹ -EPERM. |
| `fpos_advance_iff_offset_minus_one_and_not_append` | INVARIANT | f_pos changes only for writev-like path. |

### Layer 2: TLA+

`fs/pwritev2.tla`:
- States: per-call flags, per-fd f_pos, per-inode i_size, per-RLIMIT.
- Properties:
  - `safety_dsync_implies_durable` — RWF_DSYNC ⟹ post-return: data on stable storage.
  - `safety_nowait_no_alloc` — RWF_NOWAIT + extent-alloc needed ⟹ -EAGAIN.
  - `safety_append_writes_at_isize` — RWF_APPEND ⟹ pos == i_size at write time.
  - `safety_noappend_writes_at_offset` — O_APPEND + RWF_NOAPPEND + offset >= 0 ⟹ pos == offset.
  - `safety_mutex_append_noappend` — both flags ⟹ -EINVAL.
  - `liveness_nowait_terminates_fast` — RWF_NOWAIT never blocks; returns within bounded time.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pwritev2` post: flags valid ∨ -EINVAL | `Pwritev2::do_pwritev2` |
| `do_pwritev2` post: RWF_DSYNC ⟹ writeback done | `Pwritev2::do_pwritev2` |
| `do_pwritev2` post: RWF_NOWAIT ⟹ (ret > 0 ∨ -EAGAIN) | `Pwritev2::do_pwritev2` |
| `do_pwritev2` post: pos correctness for APPEND/NOAPPEND/-1 cases | `Pwritev2::do_pwritev2` |

### Layer 4: Verus / Creusot functional

Per-`pwritev2(2)` man-page semantic equivalence; LTP `pwritev201..pwritev203` pass; RWF_APPEND atomicity reproducer passes; RWF_DSYNC durability test on ext4/xfs passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pwritev2(2)` reinforcement:

- **Per-flags strict validation** — defense against per-forward-compat ABI breakage.
- **Per-APPEND/NOAPPEND mutual exclusion** — defense against per-ambiguous-semantic write.
- **Per-RWF_NOWAIT no-extent-alloc** — defense against per-blocked-thread DoS in io_uring fallback.
- **Per-RWF_DSYNC atomic with write** — defense against per-crash-window data loss.
- **Per-RLIMIT_FSIZE + SIGXFSZ** — defense against per-file-size resource abuse.
- **Per-F_SEAL_WRITE honored** — defense against per-memfd tampering.
- **Per-RWF_HIPRI restricted to O_DIRECT** — defense against per-busy-poll CPU abuse.

## Grsecurity / PaX surface

- **PaX UDEREF on iov copy_from_user** — defense against per-crafted-iov kernel deref; SMAP enforced.
- **GRKERNSEC_USERCOPY_HARDEN on copy_from_user(iov)** — bounded by `iovcnt * sizeof(iovec)`.
- **UIO_MAXIOV=1024 unconditional cap** — defense against per-iov amplification.
- **PAX_REFCOUNT on file->f_count** — defense against per-UAF on close+pwritev2.
- **GRKERNSEC_DMESG suppresses pwritev2 error logs** — `-EAGAIN`, `-EFBIG`, `-EPIPE`, `-EPERM` rate-limited.
- **PaX KERNEXEC on f_op->write_iter** — RAP/CFI protects dispatch.
- **GRKERNSEC_IO_POLL_RATE_LIMIT** — RWF_HIPRI polling capped per-task.
- **GRKERNSEC_RESLOG on RWF_DSYNC frequency** — > 10000 DSYNC/sec from a task audited (potential journal-thrash DoS).
- **GRKERNSEC_RESLOG on F_SEAL_WRITE breach attempts** — `-EPERM` logged.
- **PAX_REFCOUNT on inode->i_writecount** — strict.
- **PaX HARDENED_USERCOPY on iov_iter** — bounds enforced.
- **RWF_NOWAIT block-on-uncached defense** — kernel must not queue behind writeback mutex; verified by lockdep + grsec assertion.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `pwritev(2)` (covered in `pwritev.md`).
- `preadv2(2)` (covered in `preadv2.md`).
- `pwrite64(2)` (covered in `pwrite64.md`).
- `io_uring(7)` (covered separately).
- Implementation code.

### signature

```c
ssize_t pwritev2(int fd, const struct iovec *iov, int iovcnt,
                 off_t offset, int flags);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor (must be writable). |
| `iov` | `const struct iovec *` | in | Pointer to iovec array. |
| `iovcnt` | `int` | in | Number of entries; [0, UIO_MAXIOV=1024]. |
| `offset` | `off_t` | in | Byte offset. `-1` ⟹ use f_pos (writev-like). Otherwise `>= 0`. |
| `flags` | `int` | in | RWF_* bitmask. |

### flags

| Flag | Value | Meaning |
|---|---|---|
| `RWF_HIPRI` | 0x01 | Busy-poll completion; O_DIRECT only. |
| `RWF_DSYNC` | 0x02 | Per-call data-sync (equivalent to fdatasync after write). |
| `RWF_SYNC` | 0x04 | Per-call full sync (equivalent to fsync after write). |
| `RWF_NOWAIT` | 0x08 | Fail with -EAGAIN if write would block (filesystem-specific support). |
| `RWF_APPEND` | 0x10 | Atomically append regardless of O_APPEND flag; offset arg ignored. |
| `RWF_NOAPPEND` | 0x20 | Override O_APPEND: write at the given offset even though fd has O_APPEND. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Bytes written. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | fd invalid / not writable. |
| `EINVAL` | iovcnt out of range; offset < -1; unsupported flag combination; iov_len overflow. |
| `EAGAIN` | RWF_NOWAIT and would block (e.g., extent allocation needed). |
| `EOPNOTSUPP` | RWF_NOWAIT/RWF_HIPRI on filesystem that does not support. |
| `EFAULT` | iov / iov_base bad. |
| `EFBIG` | RLIMIT_FSIZE exceeded; SIGXFSZ raised. |
| `ENOSPC` | Device full. |
| `EDQUOT` | Quota exceeded. |
| `EIO` | I/O error. |
| `EPIPE` | Pipe closed; SIGPIPE raised. |
| `EPERM` | F_SEAL_WRITE / F_SEAL_FUTURE_WRITE blocks. |
| `ESPIPE` | offset != -1 and fd unseekable. |

### abi surface

```text
__NR_pwritev2  (x86_64) = 328
__NR_pwritev2  (arm64)  = 287
__NR_pwritev2  (riscv)  = 287
__NR_pwritev2  (i386)   = 379

RWF_SUPPORTED = RWF_HIPRI | RWF_DSYNC | RWF_SYNC | RWF_NOWAIT |
                RWF_APPEND | RWF_NOAPPEND
```

### compatibility contract

REQ-1: Syscall number is **328** on x86_64. ABI-stable.

REQ-2: `flags & ~RWF_SUPPORTED` ⟹ `-EINVAL`. Unknown bits NOT silently ignored.

REQ-3: `flags & (RWF_APPEND | RWF_NOAPPEND) == (RWF_APPEND | RWF_NOAPPEND)` ⟹ `-EINVAL` (mutual exclusion).

REQ-4: `flags & (RWF_DSYNC | RWF_SYNC)`: after the write, kernel issues per-range writeback equivalent to `fdatasync` (DSYNC) or `fsync` (SYNC). Atomic with the write itself.

REQ-5: `RWF_NOWAIT`: write returns `-EAGAIN` if it would block on extent allocation, fsync, or any locked range. Only filesystems advertising `FMODE_NOWAIT` support this; otherwise `-EOPNOTSUPP`.

REQ-6: `RWF_APPEND`: per-call append; ignores `offset` and writes at `i_size`. Holds `i_rwsem` exclusively for the duration.

REQ-7: `RWF_NOAPPEND`: per-call positional write; even if fd has `O_APPEND`, kernel uses `offset` (must be `>= 0`).

REQ-8: `offset == -1`: use and advance `f_pos` (writev-like).

REQ-9: `RWF_HIPRI`: O_DIRECT only; polls completion queue rather than waiting on irq.

REQ-10: Per-RLIMIT_FSIZE: enforced identically to `pwritev`/`writev`. Exceed ⟹ SIGXFSZ + `-EFBIG`.

REQ-11: Per-F_SEAL_WRITE: blocks ⟹ `-EPERM`.

REQ-12: `iovcnt`, iov_len, UIO_MAXIOV semantics identical to `pwritev`/`writev`.

REQ-13: Per-`f_op->write_iter`: ki_flags populated:
- RWF_HIPRI → IOCB_HIPRI
- RWF_DSYNC → IOCB_DSYNC
- RWF_SYNC → IOCB_SYNC
- RWF_NOWAIT → IOCB_NOWAIT
- RWF_APPEND → IOCB_APPEND
- RWF_NOAPPEND clears IOCB_APPEND if O_APPEND was set on fd.

REQ-14: Per-32-bit ABI: offset split into pos_l/pos_h.

REQ-15: Per-LSM: `security_file_permission(file, MAY_WRITE)`.

REQ-16: `RWF_NOWAIT` does NOT bypass mandatory locks (legacy) — F_WRLCK conflict ⟹ `-EAGAIN`.

