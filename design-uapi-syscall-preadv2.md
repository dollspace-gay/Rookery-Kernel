---
title: "Tier-5 syscall: preadv2(2) — syscall 327"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`preadv2(2)` extends `preadv(2)` with a **`flags`** argument controlling per-call I/O semantics: high-priority polling, durability, non-blocking-on-uncached, and (for pwritev2) append override. It is the modern scatter-read primitive used by io_uring-fallback paths, RocksDB, and high-throughput databases that want NOWAIT semantics on cached pages.

Special-case: `offset == -1` ⟹ behave as `readv(2)` (advance f_pos). This is a kernel convenience for combining flags-controlled I/O with file-offset progression.

### Acceptance Criteria

- [ ] AC-1: `preadv2(fd, iov, 1, 0, 0)` reads at offset 0 (preadv-like).
- [ ] AC-2: `preadv2(fd, iov, 1, -1, 0)` reads at f_pos and advances it (readv-like).
- [ ] AC-3: `preadv2(fd, iov, 1, 0, RWF_NOWAIT)` on cached page: returns data.
- [ ] AC-4: `preadv2(fd, iov, 1, 0, RWF_NOWAIT)` on uncached page: `-EAGAIN`.
- [ ] AC-5: `preadv2(fd, iov, 1, 0, 0x40000000)` ⟹ `-EINVAL` (unknown flag).
- [ ] AC-6: `preadv2(fd, iov, 1, -2, 0)` ⟹ `-EINVAL`.
- [ ] AC-7: `RWF_HIPRI` on non-O_DIRECT fd: silently ignored.
- [ ] AC-8: pipe fd with offset != -1 ⟹ `-ESPIPE`.
- [ ] AC-9: pipe fd with offset == -1: behaves as readv.
- [ ] AC-10: O_DIRECT misaligned ⟹ `-EINVAL`.
- [ ] AC-11: RWF_NOWAIT does NOT trigger readahead.

### Architecture

```rust
#[syscall(nr = 327, abi = "sysv")]
pub fn sys_preadv2(fd: i32, iov: UserPtr<Iovec>, iovcnt: i32, offset: i64, flags: i32) -> isize {
    Preadv2::do_preadv2(fd, iov, iovcnt, offset, flags as u32)
}
```

`Preadv2::do_preadv2(fd, iov, iovcnt, offset, flags) -> isize`:
1. if flags & !RWF_SUPPORTED != 0 { return Err(EINVAL); }
2. if offset < -1 { return Err(EINVAL); }
3. if iovcnt < 0 || iovcnt as usize > UIO_MAXIOV { return Err(EINVAL); }
4. let file = current().files().get_file(fd).ok_or(EBADF)?;
5. if !file.has_mode(FMODE_READ) { return Err(EBADF); }
6. let use_fpos = offset == -1;
7. if !use_fpos && !file.f_op.has_llseek() { return Err(ESPIPE); }
8. let mut stack = [Iovec::zero(); UIO_FASTIOV];
9. let mut heap: Option<Vec<Iovec>> = None;
10. let kiov = Iov::import(iov, iovcnt as usize, &mut stack, &mut heap)?;
11. let total = Iov::sum_len(kiov)?;
12. /* Compute pos */
13. let pos = if use_fpos { file.f_pos.load(Ordering::Acquire) as i64 } else { offset };
14. if pos.checked_add(total as i64).is_none() { return Err(EOVERFLOW); }
15. /* Build iter + kiocb */
16. let mut iter = IovIter::from_iov(READ, kiov);
17. let mut kiocb_flags = 0u32;
18. if flags & RWF_HIPRI != 0 { kiocb_flags |= IOCB_HIPRI; }
19. if flags & RWF_NOWAIT != 0 { kiocb_flags |= IOCB_NOWAIT | IOCB_NOIO; }
20. lsm::file_permission(&file, MAY_READ)?;
21. let nr = file.read_iter_at_with_flags(&mut iter, pos, kiocb_flags)?;
22. if use_fpos && nr > 0 { file.f_pos.fetch_add(nr as u64, Ordering::AcqRel); }
23. Ok(nr)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | unknown bits ⟹ -EINVAL. |
| `offset_minus_one_uses_fpos` | INVARIANT | offset == -1 ⟹ pos = f_pos. |
| `nowait_no_disk_io` | INVARIANT | RWF_NOWAIT ⟹ no submission of bio. |
| `nowait_no_readahead` | INVARIANT | RWF_NOWAIT ⟹ readahead suppressed. |
| `hipri_only_for_direct` | INVARIANT | RWF_HIPRI without O_DIRECT ⟹ no-op. |
| `fpos_advance_iff_offset_minus_one` | INVARIANT | offset != -1 ⟹ f_pos unchanged. |

### Layer 2: TLA+

`fs/preadv2.tla`:
- States: per-call flags, per-fd cached set, per-fd f_pos.
- Properties:
  - `safety_nowait_either_cached_or_eagain` — RWF_NOWAIT: ret > 0 ⟹ all bytes from cache; else -EAGAIN.
  - `safety_offset_minus_one_advances_fpos` — offset == -1 + ret > 0 ⟹ f_pos += ret.
  - `safety_offset_positive_invariant` — offset >= 0 ⟹ f_pos unchanged.
  - `safety_unknown_flag_rejected` — flags & ~RWF_SUPPORTED ⟹ -EINVAL.
  - `liveness_blocking_terminates` — non-NOWAIT path eventually returns or signals.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_preadv2` post: flags supported ∨ -EINVAL | `Preadv2::do_preadv2` |
| `do_preadv2` post: RWF_NOWAIT ⟹ (ret > 0 ∧ cached) ∨ -EAGAIN | `Preadv2::do_preadv2` |
| `do_preadv2` post: offset >= 0 ⟹ f_pos unchanged | `Preadv2::do_preadv2` |

### Layer 4: Verus / Creusot functional

Per-`preadv2(2)` man-page semantic equivalence; LTP `preadv201..preadv203` pass; RWF_NOWAIT cached-vs-uncached test passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`preadv2(2)` reinforcement:

- **Per-flags strict validation** — defense against per-forward-compat ABI breakage from unknown bits.
- **Per-RWF_NOWAIT no-readahead** — defense against per-cache-pollution attack (NOWAIT abuse triggering readahead).
- **Per-RWF_NOWAIT no-blocking-on-uncached** — defense against per-blocked-thread DoS.
- **Per-RWF_HIPRI restricted to O_DIRECT** — defense against per-busy-poll CPU abuse.
- **Per-offset >= -1 check** — defense against per-negative-offset surprise.
- **Per-UIO_MAXIOV cap** — defense against per-iov amplification.

## Grsecurity / PaX surface

- **PaX UDEREF on iov copy_from_user** — defense against per-crafted-iov kernel deref; SMAP enforced.
- **GRKERNSEC_USERCOPY_HARDEN on copy_from_user(iov)** — whitelisted slab, bounded.
- **UIO_MAXIOV=1024 unconditional cap** — defense against per-iov amplification.
- **PAX_REFCOUNT on file->f_count** — defense against per-UAF on close+preadv2.
- **GRKERNSEC_DMESG suppresses preadv2 error logs** — `-EAGAIN` / `-EINVAL` floods suppressed.
- **PaX KERNEXEC on f_op->read_iter** — RAP/CFI protects dispatch.
- **GRKERNSEC_RESLOG on RWF_NOWAIT-spam pattern** — > 1000 EAGAIN/sec from a task audited (potential cache-state probing).
- **PaX HARDENED_USERCOPY on iov_iter** — bounds enforced.
- **GRKERNSEC_IO_POLL_RATE_LIMIT** — RWF_HIPRI polling capped per-task to prevent CPU monopolization.
- **PAX_REFCOUNT on kiocb refs** — strict during NOWAIT bail-out paths.
- **RWF_NOWAIT block-on-uncached defense** — kernel MUST NOT queue behind writeback mutex; verified by lockdep + grsec assertion.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `preadv(2)` (covered in `preadv.md`).
- `pwritev2(2)` (covered in `pwritev2.md`).
- `io_uring(7)` (covered separately).
- `RWF_*` semantics for write side (covered in `pwritev2.md`).
- Implementation code.

### signature

```c
ssize_t preadv2(int fd, const struct iovec *iov, int iovcnt,
                off_t offset, int flags);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor (must be readable). |
| `iov` | `const struct iovec *` | in | Pointer to iovec array. |
| `iovcnt` | `int` | in | Number of entries; [0, UIO_MAXIOV=1024]. |
| `offset` | `off_t` | in | Byte offset. `-1` ⟹ use f_pos (readv-like). Otherwise `>= 0`. |
| `flags` | `int` | in | RWF_* bitmask (see Flags below). |

### flags

| Flag | Value | Meaning |
|---|---|---|
| `RWF_HIPRI` | 0x00000001 | High-priority polling; only meaningful for O_DIRECT on supporting drivers. |
| `RWF_DSYNC` | 0x00000002 | Per-call data-sync (write-side only; ignored for reads). |
| `RWF_SYNC` | 0x00000004 | Per-call full sync (write-side only; ignored for reads). |
| `RWF_NOWAIT` | 0x00000008 | If page would require disk I/O ⟹ `-EAGAIN`; only proceeds for cached pages. |
| `RWF_APPEND` | 0x00000010 | Per-call append (write-side only; ignored for reads). |
| `RWF_NOAPPEND` | 0x00000020 | Override O_APPEND for this call (write-side; ignored for reads). |

For preadv2 only `RWF_HIPRI` and `RWF_NOWAIT` are meaningful; other write-side flags are silently accepted (no effect) or rejected with `-EOPNOTSUPP` depending on Kconfig.

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Bytes read. |
| `0` | EOF. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | fd invalid / not readable. |
| `EINVAL` | iovcnt out of range; offset < -1; unsupported flag bit; iov_len overflow. |
| `EAGAIN` | `RWF_NOWAIT` and page not cached. |
| `EOPNOTSUPP` | flag unsupported by filesystem (e.g., RWF_HIPRI on non-direct fs). |
| `EFAULT` | iov / iov_base bad. |
| `ESPIPE` | offset != -1 and fd unseekable. |
| `EISDIR` | fd is a directory. |
| `EINTR` | Signal before any byte read. |

### abi surface

```text
__NR_preadv2  (x86_64) = 327
__NR_preadv2  (arm64)  = 286
__NR_preadv2  (riscv)  = 286
__NR_preadv2  (i386)   = 378

RWF_HIPRI       0x01
RWF_DSYNC       0x02
RWF_SYNC        0x04
RWF_NOWAIT      0x08
RWF_APPEND      0x10
RWF_NOAPPEND    0x20

RWF_SUPPORTED = (RWF_HIPRI | RWF_DSYNC | RWF_SYNC | RWF_NOWAIT |
                 RWF_APPEND | RWF_NOAPPEND)
```

### compatibility contract

REQ-1: Syscall number is **327** on x86_64. ABI-stable.

REQ-2: `flags & ~RWF_SUPPORTED` ⟹ `-EINVAL`. Unknown bits NOT silently ignored (defense against forward-compat ABI breakage).

REQ-3: `offset == -1`: use and advance `f_pos`. Otherwise `offset >= 0`.

REQ-4: `RWF_NOWAIT`: if the read would block on disk I/O (uncached page, congested device, swap-in) ⟹ `-EAGAIN`. **Only proceed if all bytes are immediately available from page cache.** No partial completion: either all bytes returned from cache, or `-EAGAIN`.

REQ-5: `RWF_HIPRI`: only meaningful for `O_DIRECT` fds backed by drivers supporting `BLK_MQ_POLL`. Without `O_DIRECT` ⟹ silently ignored. The kernel polls the completion queue rather than waiting on interrupt.

REQ-6: Write-side flags (`RWF_DSYNC`, `RWF_SYNC`, `RWF_APPEND`, `RWF_NOAPPEND`) on preadv2: per kernel policy either silently accepted (no effect) or `-EOPNOTSUPP`. Linux currently accepts silently.

REQ-7: `iovcnt`, iov_len, UIO_MAXIOV semantics identical to `preadv(2)`.

REQ-8: `RWF_NOWAIT` + uncached + congested block layer ⟹ defense-in-depth: kernel MUST NOT trigger readahead, MUST NOT block in mempool waits, MUST NOT take any lock that may queue behind a writeback.

REQ-9: Per-32-bit: offset split into pos_l/pos_h.

REQ-10: Per-LSM: `security_file_permission(file, MAY_READ)`.

REQ-11: Per-O_DIRECT alignment: identical to preadv.

REQ-12: Per-`f_op->read_iter`: ki_flags populated from RWF_* (RWF_HIPRI → IOCB_HIPRI, RWF_NOWAIT → IOCB_NOWAIT).

REQ-13: `RWF_NOWAIT` semantics for network FS (NFS, CephFS): if the file is open with `O_NONBLOCK` and the data is not in client cache, return `-EAGAIN`; do NOT issue RPC.

REQ-14: `RWF_NOWAIT` does NOT short-circuit signal delivery: pending signals still cause `-EINTR` checks.

REQ-15: `offset == -1` with multi-threaded shared fd: f_pos race possible; same as readv (no atomicity).

