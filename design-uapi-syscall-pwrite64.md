---
title: "Tier-5 syscall: pwrite64(2) — syscall 18"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pwrite64(2)` writes up to `count` bytes from a single buffer to a file descriptor at an explicit `offset`, **without affecting the file offset**. Scalar form of `pwritev(2)`; glibc exposes it as `pwrite()`.

Like `pwritev`, pwrite64 has the **Linux O_APPEND quirk**: if the fd has `O_APPEND`, the offset is ignored and data is appended at end-of-file.

Used widely by: Postgres heap and WAL, MySQL InnoDB doublewrite buffer, LMDB, LevelDB. Critical for: per-positional single-buffer write, per-thread-safe shared-fd writes.

### Acceptance Criteria

- [ ] AC-1: `pwrite64(fd, "hello", 5, 100)` writes 5 bytes at offset 100; f_pos unchanged.
- [ ] AC-2: `pwrite64(fd, buf, 5, -1)` ⟹ `-EINVAL`.
- [ ] AC-3: `pwrite64(pipe_wr, buf, 5, 0)` ⟹ `-ESPIPE`.
- [ ] AC-4: `pwrite64(-1, buf, 5, 0)` ⟹ `-EBADF`.
- [ ] AC-5: `pwrite64(fd, NULL, 5, 0)` ⟹ `-EFAULT`.
- [ ] AC-6: `pwrite64(fd, buf, 0, 0)` ⟹ returns 0.
- [ ] AC-7: O_APPEND fd: pwrite64 ignores offset, appends to EOF.
- [ ] AC-8: RLIMIT_FSIZE exceeded ⟹ SIGXFSZ + `-EFBIG`.
- [ ] AC-9: F_SEAL_WRITE ⟹ `-EPERM`.
- [ ] AC-10: O_DIRECT misaligned ⟹ `-EINVAL`.
- [ ] AC-11: Two threads pwrite64'ing non-overlapping ranges: both succeed.
- [ ] AC-12: pwrite64(fd, buf, SIZE_MAX, S64_MAX-1) ⟹ `-EOVERFLOW` or capped.
- [ ] AC-13: 32-bit compat: split pos_l/pos_h assembles correctly.

### Architecture

```rust
#[syscall(nr = 18, abi = "sysv")]
pub fn sys_pwrite64(fd: i32, buf: UserPtr<u8>, count: usize, offset: i64) -> isize {
    Pwrite64::do_pwrite64(fd, buf, count, offset)
}
```

`Pwrite64::do_pwrite64(fd, buf, count, offset) -> isize`:
1. if offset < 0 { return Err(EINVAL); }
2. if count == 0 { return Ok(0); }
3. let n = core::cmp::min(count, MAX_RW_COUNT);
4. if offset.checked_add(n as i64).is_none() { return Err(EOVERFLOW); }
5. let file = current().files().get_file(fd).ok_or(EBADF)?;
6. if !file.has_mode(FMODE_WRITE) { return Err(EBADF); }
7. if !file.f_op.has_llseek() { return Err(ESPIPE); }
8. if file.inode().seals().contains(F_SEAL_WRITE) { return Err(EPERM); }
9. if !access_ok(buf, n) { return Err(EFAULT); }
10. /* O_APPEND quirk */
11. let pos = if file.has_flag(O_APPEND) {
12.   file.inode().size_locked_for_append()
13. } else {
14.   offset
15. };
16. /* RLIMIT_FSIZE */
17. let limit = current().rlimit(RLIMIT_FSIZE);
18. if (pos as u64) >= limit { send_sig(SIGXFSZ, current()); return Err(EFBIG); }
19. let count_clamped = core::cmp::min(n as u64, limit - pos as u64) as usize;
20. lsm::file_permission(&file, MAY_WRITE)?;
21. let mut iter = IovIter::single_user_src(buf, count_clamped);
22. let nr = file.write_iter_at(&mut iter, pos)?;
23. /* DO NOT update f_pos */
24. Ok(nr)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `offset_nonneg` | INVARIANT | offset >= 0. |
| `f_pos_unchanged` | INVARIANT | file->f_pos identical pre/post. |
| `count_capped` | INVARIANT | effective count <= MAX_RW_COUNT. |
| `rlimit_fsize_honored` | INVARIANT | pos + ret <= RLIMIT_FSIZE. |
| `seal_write_blocks` | INVARIANT | F_SEAL_WRITE ⟹ -EPERM. |
| `o_append_overrides_offset` | INVARIANT | O_APPEND ⟹ pos = i_size. |
| `seekable_required` | INVARIANT | non-seekable fd ⟹ -ESPIPE. |
| `fmode_write_required` | INVARIANT | fd lacks FMODE_WRITE ⟹ -EBADF. |
| `userbuf_userspace_only` | INVARIANT | copy_from_user touches userspace only. |

### Layer 2: TLA+

`fs/pwrite64.tla`:
- States: per-thread args, per-fd f_pos, per-inode i_size, per-RLIMIT counter.
- Properties:
  - `safety_f_pos_invariant` — pwrite64 never mutates f_pos.
  - `safety_o_append_quirk` — O_APPEND fd ⟹ pos = i_size.
  - `safety_rlimit_clamp` — never write past RLIMIT_FSIZE.
  - `safety_seal_write_blocks` — F_SEAL_WRITE ⟹ -EPERM.
  - `liveness_terminates_or_signals` — given drain, pwrite64 returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pwrite64` post: ret >= 0 ⟹ f_pos unchanged | `Pwrite64::do_pwrite64` |
| `do_pwrite64` post: 0 <= ret <= min(count, MAX_RW_COUNT) | `Pwrite64::do_pwrite64` |
| `do_pwrite64` post: O_APPEND ⟹ data appended at i_size | `Pwrite64::do_pwrite64` |

### Layer 4: Verus / Creusot functional

Per-`pwrite(2)` man-page semantic equivalence; LTP `pwrite01..pwrite04` pass; O_APPEND quirk test matches Linux behavior; memfd F_SEAL_WRITE test passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pwrite64(2)` reinforcement:

- **Per-offset >= 0 check** — defense against per-negative-offset surprise.
- **Per-MAX_RW_COUNT cap** — defense against per-huge-write DoS.
- **Per-RLIMIT_FSIZE + SIGXFSZ** — defense against per-file-size resource abuse.
- **Per-F_SEAL_WRITE honored** — defense against per-memfd tampering.
- **Per-f_pos invariant** — defense against per-shared-fd thread interference.
- **Per-O_DIRECT alignment** — defense against per-misaligned DMA.
- **Per-i64 overflow** — defense against per-arithmetic-wrap.
- **Per-O_APPEND quirk documented + enforced** — defense against per-spec-mismatch silent overwrite.

## Grsecurity / PaX surface

- **PaX UDEREF on buf copy_from_user** — defense against per-buf-pointer kernel deref; SMAP enforced.
- **GRKERNSEC_USERCOPY_HARDEN on copy_from_user(buf, count)** — bounded by `count`, whitelisted slab.
- **PAX_REFCOUNT on file->f_count** — defense against per-UAF on close+pwrite64 race.
- **GRKERNSEC_DMESG suppresses pwrite64 error logs** — `-EFBIG`, `-EPIPE`, `-EINVAL`, `-EPERM` rate-limited.
- **PaX KERNEXEC on f_op->write_iter** — RAP/CFI protects dispatch.
- **GRKERNSEC_RESLOG on F_SEAL_WRITE breach attempts** — `-EPERM` logged.
- **GRKERNSEC_RESLOG on offset > i_size + 4 GiB** — sparse-allocation amplification audited.
- **PAX_REFCOUNT on inode->i_writecount** — strict.
- **PAX_USERCOPY whitelisted slab** — buf must point into mapped user VMA.
- **PaX FORTIFY_SOURCE on iov_iter** — bounds enforced.
- **GRKERNSEC_PROC_OFFSETS** — pwrite64 positional writes not visible in `/proc/<pid>/fdinfo`.
- **PAX_REFCOUNT on kiocb refs** — strict.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `write(2)` (covered separately).
- `pwritev(2)` (covered in `pwritev.md`).
- `pwritev2(2)` (covered in `pwritev2.md`).
- `pread64(2)` (covered in `pread64.md`).
- Implementation code.

### signature

```c
ssize_t pwrite64(int fd, const void *buf, size_t count, off_t offset);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open seekable file descriptor (must be writable). |
| `buf` | `const void *` | in | Source userspace buffer. |
| `count` | `size_t` | in | Number of bytes to write; capped at `MAX_RW_COUNT`. |
| `offset` | `off_t` | in | Byte offset from start of file. Must be `>= 0`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Bytes written. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | fd invalid / not writable. |
| `EINVAL` | offset < 0; fd unsuitable. |
| `EFAULT` | buf outside accessible address space. |
| `EFBIG` | RLIMIT_FSIZE exceeded; SIGXFSZ raised. |
| `ENOSPC` | Device full. |
| `EDQUOT` | Quota exceeded. |
| `EIO` | I/O error. |
| `EAGAIN` | Non-blocking and would block. |
| `EINTR` | Signal delivered before any byte written. |
| `ESPIPE` | fd is pipe/socket/FIFO. |
| `EPERM` | F_SEAL_WRITE / F_SEAL_FUTURE_WRITE set. |
| `EPIPE` | Broken pipe (rare; pwrite64 normally for seekable fds); SIGPIPE raised. |
| `EOVERFLOW` | offset + count overflows. |

### abi surface

```text
__NR_pwrite64  (x86_64) = 18
__NR_pwrite64  (arm64)  = 68
__NR_pwrite64  (riscv)  = 68
__NR_pwrite64  (i386)   = 181  /* split pos_l/pos_h pair */

MAX_RW_COUNT  = INT_MAX & PAGE_MASK
RLIMIT_FSIZE  enforced.
```

### compatibility contract

REQ-1: Syscall number is **18** on x86_64. ABI-stable.

REQ-2: `offset < 0` ⟹ `-EINVAL`.

REQ-3: `count > MAX_RW_COUNT` ⟹ capped (not an error).

REQ-4: `count == 0` ⟹ kernel returns 0 without invoking f_op->write_iter (but still runs LSM check; matches Linux).

REQ-5: File offset (f_pos) NOT modified (except O_APPEND quirk does not modify f_pos either; it modifies i_size).

REQ-6: `fd` MUST be seekable; non-seekable ⟹ `-ESPIPE`.

REQ-7: Per-O_APPEND quirk: if fd has O_APPEND, kernel ignores the supplied offset and appends at `i_size` (Linux behavior; POSIX says undefined).

REQ-8: Per-RLIMIT_FSIZE: `pos + count > limit` ⟹ truncate; `pos >= limit` ⟹ SIGXFSZ + `-EFBIG`.

REQ-9: Per-F_SEAL_WRITE: ⟹ `-EPERM`.

REQ-10: Per-O_DIRECT: `buf`, `count`, `offset` MUST be block-aligned.

REQ-11: Per-`f_op->write_iter`: preferred; kiocb.ki_pos = pos.

REQ-12: Per-32-bit ABI: offset split into pos_l/pos_h; glibc wrapper assembles.

REQ-13: Per-LSM: `security_file_permission(file, MAY_WRITE)`.

REQ-14: Per-overflow: offset + count > S64_MAX ⟹ `-EOVERFLOW`.

REQ-15: Per-fsync: pwrite64 does NOT imply sync; data may sit in page cache.

REQ-16: Per-shared fd across threads: pwrite64 to non-overlapping ranges parallel-safe; overlapping writes have no kernel-defined ordering (last writer wins per page).

