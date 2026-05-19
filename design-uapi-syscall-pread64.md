---
title: "Tier-5 syscall: pread64(2) — syscall 17"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pread64(2)` reads up to `count` bytes from a file descriptor into a single buffer at an explicit `offset`, **without affecting the file offset**. It is the scalar (single-buffer) form of `preadv(2)`; user-visible as `pread()` in glibc, which transparently calls `pread64` on 32-bit ABIs where `off_t` would otherwise be 32 bits.

Used everywhere a thread-safe positional read is needed: Postgres `read()` paths, mmap fallback, file-backed memory database engines (LMDB, BoltDB), pread+writev pipelines. Critical for: per-positional single-buffer read, per-thread-safe shared-fd I/O.

### Acceptance Criteria

- [ ] AC-1: `pread64(fd, buf, 16, 100)` reads 16 bytes starting at offset 100.
- [ ] AC-2: After pread64: `lseek(fd, 0, SEEK_CUR)` returns prior value (unchanged).
- [ ] AC-3: `pread64(fd, buf, 16, -1)` ⟹ `-EINVAL`.
- [ ] AC-4: `pread64(pipe_rd, buf, 16, 0)` ⟹ `-ESPIPE`.
- [ ] AC-5: `pread64(-1, buf, 16, 0)` ⟹ `-EBADF`.
- [ ] AC-6: `pread64(fd, NULL, 16, 0)` ⟹ `-EFAULT`.
- [ ] AC-7: `pread64(fd, buf, 0, 0)` ⟹ returns 0.
- [ ] AC-8: `pread64(fd, buf, SIZE_MAX, S64_MAX-1)` ⟹ `-EOVERFLOW` or capped.
- [ ] AC-9: Past-EOF offset ⟹ returns 0.
- [ ] AC-10: Directory fd ⟹ `-EISDIR`.
- [ ] AC-11: O_DIRECT misaligned ⟹ `-EINVAL`.
- [ ] AC-12: Two threads pread64'ing different offsets: both succeed.
- [ ] AC-13: 32-bit compat: split pos_l/pos_h assembles correctly.

### Architecture

```rust
#[syscall(nr = 17, abi = "sysv")]
pub fn sys_pread64(fd: i32, buf: UserPtr<u8>, count: usize, offset: i64) -> isize {
    Pread64::do_pread64(fd, buf, count, offset)
}
```

`Pread64::do_pread64(fd, buf, count, offset) -> isize`:
1. if offset < 0 { return Err(EINVAL); }
2. if count == 0 { return Ok(0); }
3. let n = core::cmp::min(count, MAX_RW_COUNT);
4. if offset.checked_add(n as i64).is_none() { return Err(EOVERFLOW); }
5. let file = current().files().get_file(fd).ok_or(EBADF)?;
6. if !file.has_mode(FMODE_READ) { return Err(EBADF); }
7. if !file.f_op.has_llseek() { return Err(ESPIPE); }
8. if !access_ok(buf, n) { return Err(EFAULT); }
9. lsm::file_permission(&file, MAY_READ)?;
10. /* Build single-segment iov_iter */
11. let mut iter = IovIter::single_user_dest(buf, n);
12. let nr = file.read_iter_at(&mut iter, offset)?;
13. /* DO NOT update f_pos */
14. Ok(nr)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `offset_nonneg` | INVARIANT | offset >= 0. |
| `f_pos_unchanged` | INVARIANT | file->f_pos identical pre/post. |
| `count_capped` | INVARIANT | effective count <= MAX_RW_COUNT. |
| `overflow_rejected` | INVARIANT | offset + count <= S64_MAX. |
| `seekable_required` | INVARIANT | non-seekable fd ⟹ -ESPIPE. |
| `fmode_read_required` | INVARIANT | fd lacks FMODE_READ ⟹ -EBADF. |
| `userbuf_userspace_only` | INVARIANT | copy_to_user touches userspace only. |

### Layer 2: TLA+

`fs/pread64.tla`:
- States: per-fd f_pos, per-thread args, per-file content.
- Properties:
  - `safety_f_pos_invariant` — pread64 never mutates f_pos.
  - `safety_offset_bounded` — offset >= 0 always.
  - `safety_concurrent_threads_independent` — N threads pread64'ing different offsets all succeed.
  - `safety_eof_or_short` — past-EOF returns 0; partial returns short count.
  - `liveness_terminates` — given data, pread64 returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pread64` post: ret >= 0 ⟹ f_pos unchanged | `Pread64::do_pread64` |
| `do_pread64` post: 0 <= ret <= min(count, MAX_RW_COUNT) | `Pread64::do_pread64` |
| `do_pread64` post: count == 0 ⟹ ret == 0 without f_op call | `Pread64::do_pread64` |

### Layer 4: Verus / Creusot functional

Per-`pread(2)` man-page semantic equivalence; LTP `pread01..pread04` pass; multithreaded shared-fd pread test passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pread64(2)` reinforcement:

- **Per-offset >= 0 check** — defense against per-negative-offset surprise.
- **Per-MAX_RW_COUNT cap** — defense against per-huge-read DoS.
- **Per-ESPIPE on unseekable fd** — defense against per-pipe positional misuse.
- **Per-f_pos invariant** — defense against per-shared-fd thread interference.
- **Per-O_DIRECT alignment** — defense against per-misaligned DMA.
- **Per-i64 overflow on offset+count** — defense against per-arithmetic-wrap.

## Grsecurity / PaX surface

- **PaX UDEREF on buf copy_to_user** — defense against per-buf-pointer kernel deref; SMAP enforced.
- **GRKERNSEC_USERCOPY_HARDEN on copy_to_user(buf, count)** — bounded by `count`, whitelisted slab.
- **PAX_REFCOUNT on file->f_count** — defense against per-UAF on close+pread64 race.
- **GRKERNSEC_DMESG suppresses pread64 error logs** — `-EINVAL`, `-EFAULT`, `-ESPIPE` rate-limited.
- **PaX KERNEXEC on f_op->read_iter** — RAP/CFI protects dispatch.
- **GRKERNSEC_RESLOG on offset > i_size + 1 GiB** — sparse-file probing audited.
- **PAX_USERCOPY whitelisted slab** — buf must point into mapped user VMA.
- **Per-fd permission re-checked on every pread64** — defense against per-fd-revocation race (LSM hook).
- **PaX FORTIFY_SOURCE on iov_iter** — bounds enforced in `copy_page_to_iter`.
- **GRKERNSEC_PROC_OFFSETS** — pread64 positional reads not visible in `/proc/<pid>/fdinfo` (no f_pos change).
- **PAX_REFCOUNT on kiocb refs** — strict.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `read(2)` (covered separately).
- `preadv(2)` (covered in `preadv.md`).
- `preadv2(2)` (covered in `preadv2.md`).
- `pwrite64(2)` (covered in `pwrite64.md`).
- Implementation code.

### signature

```c
ssize_t pread64(int fd, void *buf, size_t count, off_t offset);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open seekable file descriptor (must be readable). |
| `buf` | `void *` | out | Destination userspace buffer. |
| `count` | `size_t` | in | Number of bytes to read; capped at `MAX_RW_COUNT` (~2 GiB - PAGE_SIZE). |
| `offset` | `off_t` | in | Byte offset from start of file. Must be `>= 0`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Bytes read. |
| `0` | EOF at offset. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | fd invalid / not readable. |
| `EINVAL` | offset < 0; fd unsuitable for positional read. |
| `EFAULT` | buf outside accessible address space. |
| `EAGAIN` | Non-blocking and would block. |
| `EINTR` | Signal delivered before any byte read. |
| `EIO` | I/O error. |
| `EISDIR` | fd is a directory. |
| `ESPIPE` | fd is pipe/socket/FIFO. |
| `EOVERFLOW` | offset + count overflows. |

### abi surface

```text
__NR_pread64  (x86_64) = 17
__NR_pread64  (arm64)  = 67
__NR_pread64  (riscv)  = 67
__NR_pread64  (i386)   = 180  /* split pos_l/pos_h pair */

MAX_RW_COUNT = INT_MAX & PAGE_MASK   ≈ 0x7ffff000  on 4 KiB pages

/* On 32-bit ABIs the offset arrives as two longs; SYSCALL_DEFINE
   uses pos_l + pos_h or alignment padding depending on arch. */
```

### compatibility contract

REQ-1: Syscall number is **17** on x86_64. ABI-stable.

REQ-2: `offset < 0` ⟹ `-EINVAL`.

REQ-3: `count > MAX_RW_COUNT` ⟹ kernel transparently caps to MAX_RW_COUNT (POSIX-compliant; not an error).

REQ-4: `count == 0` ⟹ kernel performs entry-side validation and returns 0 without invoking f_op->read_iter.

REQ-5: File offset (f_pos) NOT modified.

REQ-6: `fd` MUST be seekable (have `f_op->llseek != no_llseek`); otherwise `-ESPIPE`.

REQ-7: Per-O_DIRECT: `buf`, `count`, `offset` MUST be block-aligned; misaligned ⟹ `-EINVAL`.

REQ-8: Per-O_NONBLOCK: rare for regular files; network FS may return `-EAGAIN`.

REQ-9: Per-signal: signal before any byte ⟹ `-EINTR`; else short return.

REQ-10: Per-32-bit ABI: offset split into `pos_l, pos_h` on i386/ARM. glibc wrapper assembles.

REQ-11: Per-`f_op->read_iter`: preferred; kiocb.ki_pos = offset.

REQ-12: Per-LSM: `security_file_permission(file, MAY_READ)`.

REQ-13: Per-overflow: offset + count > S64_MAX ⟹ `-EOVERFLOW`.

REQ-14: Per-mandatory locking (legacy): F_RDLCK conflict ⟹ `-EAGAIN`.

REQ-15: Per-shared fd across threads: multiple pread64 to non-overlapping ranges are parallel-safe; overlapping reads see page-cache consistency at the page level (kernel does not serialize at byte level).

REQ-16: Per-block-device fd: offset is byte position; reads beyond device size return short or 0.

