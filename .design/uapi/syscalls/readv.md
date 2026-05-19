# Tier-5 syscall: readv(2) — syscall 19

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/read_write.c (SYSCALL_DEFINE3(readv), do_readv, vfs_readv, do_iter_read)
  - include/linux/uio.h (struct iovec, iov_iter)
  - include/uapi/linux/uio.h (UIO_MAXIOV=1024, UIO_FASTIOV=8)
  - arch/x86/entry/syscalls/syscall_64.tbl (19  common  readv)
-->

## Summary

`readv(2)` performs a **scatter read**: it reads from a file descriptor into the buffers described by the `iov[]` vector in a single syscall, atomically with respect to other read/write calls on the same fd. The kernel returns once the contiguous bytes have filled the buffers in order (or until EOF / signal / EAGAIN). Critical for: per-scatter-gather throughput, per-userspace zero-copy, per-iovec atomicity.

This is the historical Berkeley-style scatter read; modern callers may prefer `preadv2(2)` for offset+flags control. Used heavily by: nginx (sendfile fallback), Postgres (multi-buffer WAL replay), QEMU virtio.

## Signature

```c
ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
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
| `fd` | `int` | in | Open file descriptor (must be readable). |
| `iov` | `const struct iovec *` | in | Pointer to array of `iovcnt` iovec entries (read by kernel). |
| `iovcnt` | `int` | in | Number of iovec entries; range [0, UIO_MAXIOV=1024]. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of bytes read into the buffers (sum may be less than total requested on short read / EOF / signal). |
| `0` | EOF reached before any byte read. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not a valid descriptor or not open for read. |
| `EINVAL` | `iovcnt < 0` or `iovcnt > UIO_MAXIOV`; or sum of `iov_len` overflows `ssize_t`; or fd is unsuitable (e.g., a directory). |
| `EFAULT` | `iov` array or any `iov_base` points outside accessible address space. |
| `EAGAIN` | Non-blocking fd; no data available. |
| `EINTR` | Signal delivered before any byte read. |
| `EIO` | Low-level I/O error. |
| `EISDIR` | `fd` refers to a directory. |
| `ENOMEM` | Kernel could not allocate iovec copy buffer (large iovcnt path). |

## ABI surface

```text
__NR_readv  (x86_64) = 19
__NR_readv  (arm64)  = 65
__NR_readv  (riscv)  = 65
__NR_readv  (i386)   = 145

UIO_MAXIOV    = 1024      /* hard cap on iovcnt */
UIO_FASTIOV   = 8         /* on-stack iovec count threshold */

/* struct iovec layout:
 *   64-bit ABI: 16 bytes (8 base + 8 len)
 *   32-bit ABI:  8 bytes (4 base + 4 len)
 * compat_iovec required for 32-on-64 syscalls.
 */
```

## Compatibility contract

REQ-1: Syscall number is **19** on x86_64. ABI-stable.

REQ-2: `iovcnt < 0` ⟹ `-EINVAL`. `iovcnt > UIO_MAXIOV` ⟹ `-EINVAL`. `iovcnt == 0` ⟹ returns 0 successfully.

REQ-3: Each `iov[i].iov_len` MUST be representable as `size_t`; sum across all entries MUST fit in `ssize_t` (else `-EINVAL`).

REQ-4: Kernel copies the `iov[]` array into kernel memory before walking it (defense against TOCTOU). For `iovcnt <= UIO_FASTIOV`, uses on-stack `iovec_stack[8]`; otherwise allocates via `kmalloc(iovcnt * sizeof(struct iovec), GFP_KERNEL)`.

REQ-5: Each `iov_base` is validated by `access_ok` + UDEREF on copy-back; entries with `iov_len == 0` are skipped without faulting.

REQ-6: Read is **atomic with respect to other readv/writev** on the same file position (regular files). Concurrent processes will not see interleaved partial writes appearing between two iovec slots.

REQ-7: Advances `fd` file offset by exactly the number of bytes returned (regular files, pipes that have an offset semantics — pipes do not).

REQ-8: For pipes / sockets / character devices: returns as soon as **any** byte is available; remaining iov slots stay untouched.

REQ-9: Per-O_NONBLOCK: if no data available ⟹ `-EAGAIN` (or `EWOULDBLOCK`, identical on Linux).

REQ-10: Signal handling: if a signal is delivered before any byte read, returns `-EINTR`. After at least one byte, returns the short count.

REQ-11: Per-`pipe(7)`: writes of size <= PIPE_BUF (4096) are atomic; reads via readv that cross multiple writers see writer-atomic boundaries.

REQ-12: Per-32-bit compat: `compat_sys_readv` translates `compat_iovec` (8-byte entries) into native; same UIO_MAXIOV cap.

REQ-13: Per-RLIMIT_FSIZE: not applicable to read (only writes consult).

REQ-14: Per-`F_SETFL` `O_DIRECT`: each `iov_base` and `iov_len` MUST be block-aligned; misaligned ⟹ `-EINVAL`.

REQ-15: Per-vfs hook: `f_op->read_iter` is preferred; legacy `f_op->read` is wrapped via `new_sync_read` with synthesized iov_iter.

## Acceptance Criteria

- [ ] AC-1: `readv(pipe_rd, iov={4,4,4}, 3)` reads up to 12 bytes filling slots in order.
- [ ] AC-2: `readv(fd, iov, 1025)` ⟹ `-EINVAL`.
- [ ] AC-3: `readv(-1, iov, 1)` ⟹ `-EBADF`.
- [ ] AC-4: `readv(fd, NULL, 1)` ⟹ `-EFAULT`.
- [ ] AC-5: `readv(fd, iov, 0)` ⟹ returns 0.
- [ ] AC-6: iov with one entry iov_len = SSIZE_MAX, second iov_len = 1: overflow ⟹ `-EINVAL`.
- [ ] AC-7: Directory fd ⟹ `-EISDIR`.
- [ ] AC-8: O_NONBLOCK pipe empty ⟹ `-EAGAIN`.
- [ ] AC-9: Signal during long read returns `-EINTR` if 0 bytes, short count otherwise.
- [ ] AC-10: O_DIRECT misaligned iov_base ⟹ `-EINVAL`.
- [ ] AC-11: File offset advances by exact returned count.
- [ ] AC-12: 32-bit compat readv via `compat_iovec` works identically.

## Architecture

```rust
#[syscall(nr = 19, abi = "sysv")]
pub fn sys_readv(fd: i32, iov: UserPtr<Iovec>, iovcnt: i32) -> isize {
    Readv::do_readv(fd, iov, iovcnt)
}
```

`Readv::do_readv(fd, iov, iovcnt) -> isize`:
1. if iovcnt < 0 || iovcnt as usize > UIO_MAXIOV { return Err(EINVAL); }
2. /* Resolve fd */
3. let file = current().files().get_file(fd).ok_or(EBADF)?;
4. if !file.has_mode(FMODE_READ) { return Err(EBADF); }
5. /* Import iovec array into kernel */
6. let mut stack = [Iovec::zero(); UIO_FASTIOV];
7. let mut heap: Option<Vec<Iovec>> = None;
8. let kiov: &mut [Iovec] = Readv::import_iov(iov, iovcnt as usize, &mut stack, &mut heap)?;
9. /* Validate sum of iov_len does not overflow ssize_t */
10. let total = Readv::sum_iov_len(kiov)?;
11. /* Build iov_iter */
12. let mut iter = IovIter::from_iov(READ, kiov);
13. /* Dispatch to f_op->read_iter or new_sync_read */
14. let pos = file.f_pos.load(Ordering::Acquire);
15. let nr = file.read_iter(&mut iter, pos)?;
16. if nr > 0 { file.f_pos.fetch_add(nr as u64, Ordering::AcqRel); }
17. Ok(nr)

`Readv::import_iov(uptr, count, stack, heap) -> Result<&mut [Iovec]>`:
1. if count == 0 { return Ok(&mut []); }
2. let slot: &mut [Iovec] = if count <= UIO_FASTIOV {
3.   &mut stack[..count]
4. } else {
5.   *heap = Some(vec![Iovec::zero(); count]);
6.   heap.as_mut().unwrap().as_mut_slice()
7. };
8. /* SMAP/UDEREF copy_from_user */
9. uptr.copy_array_in(slot)?;
10. /* Per-entry validation */
11. for v in slot.iter() {
12.   if v.iov_len > SSIZE_MAX as usize { return Err(EINVAL); }
13.   if v.iov_len != 0 && !access_ok(v.iov_base, v.iov_len) { return Err(EFAULT); }
14. }
15. Ok(slot)

`Readv::sum_iov_len(slot) -> Result<usize>`:
1. let mut total: usize = 0;
2. for v in slot { total = total.checked_add(v.iov_len).ok_or(EINVAL)?; if total > SSIZE_MAX as usize { return Err(EINVAL); } }
3. Ok(total)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `iovcnt_bounded` | INVARIANT | iovcnt in [0, UIO_MAXIOV]. |
| `iov_sum_no_overflow` | INVARIANT | sum of iov_len <= SSIZE_MAX. |
| `iov_copy_userspace_only` | INVARIANT | copy_from_user touches userspace addresses only. |
| `f_pos_advance_matches_return` | INVARIANT | regular file: f_pos delta == returned count. |
| `iovec_array_kernel_local` | INVARIANT | kernel walks its own copy of iov, not user memory. |
| `directory_rejected` | INVARIANT | fd S_ISDIR ⟹ -EISDIR. |

### Layer 2: TLA+

`fs/readv.tla`:
- States: per-fd offset, per-iov slot fill, signal pending.
- Properties:
  - `safety_no_partial_iov_write` — kernel never partially fills an iov slot then proceeds to the next slot without exhausting the previous.
  - `safety_signal_short_return` — signal before any byte ⟹ -EINTR; else short count.
  - `liveness_blocking_eventually_returns` — given data, readv returns.
  - `safety_offset_monotonic` — f_pos never moves backward.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_readv` post: 0 <= ret <= sum(iov_len) | `Readv::do_readv` |
| `import_iov` post: kernel slice independent of user iov | `Readv::import_iov` |
| `sum_iov_len` post: result <= SSIZE_MAX | `Readv::sum_iov_len` |

### Layer 4: Verus / Creusot functional

Per-`readv(2)` man-page semantic equivalence; LTP `readv01..readv03` pass; pipe/socket atomicity reproducer passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`readv(2)` reinforcement:

- **Per-UIO_MAXIOV=1024 hard cap** — defense against per-iov-cnt-DoS (memory amplification).
- **Per-iov array copied to kernel before walk** — defense against per-TOCTOU (user mutating iov mid-syscall).
- **Per-iov_len overflow check** — defense against per-ssize_t wraparound returning huge negative.
- **Per-FMODE_READ enforced** — defense against per-write-only fd read.
- **Per-O_DIRECT alignment checked** — defense against per-misaligned-DMA corruption.

## Grsecurity / PaX surface

- **PaX UDEREF on iov + iov_base** — defense against per-kernel-deref via crafted iovec; SMAP enforces user-only access in copy paths.
- **GRKERNSEC_USERCOPY_HARDEN on copy_from_user(iov)** — whitelisted slab, bounded `iovcnt * sizeof(iovec)`.
- **UIO_MAXIOV=1024 enforced unconditionally** — no Kconfig knob; defense against per-iovec-array memory amplification.
- **PAX_REFCOUNT on file->f_count** — defense against per-UAF on concurrent close+readv race.
- **GRKERNSEC_DMESG suppresses readv error spam** — verbose -EFAULT/-EINVAL logs visible only to CAP_SYSLOG.
- **PaX KERNEXEC on f_op->read_iter dispatch** — function pointer call protected by KERNEXEC RAP/CFI.
- **Per-fd permission re-checked on every readv** — defense against per-fd-revocation race (LSM hook).
- **Per-iovec stack-allocated for iovcnt <= 8** — defense against per-kmalloc-pressure DoS at high readv frequency.
- **GRKERNSEC_RESLOG on iovcnt > 64** — audit trail for large scatter reads (common in malware exfil).
- **PaX FORTIFY_SOURCE on iov_iter copy paths** — bounds enforced in `copy_page_to_iter`.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `writev(2)` (covered in `writev.md`).
- `preadv(2)` / `preadv2(2)` (covered separately).
- `pread64(2)` (covered separately).
- vmsplice / process_vm_readv (covered separately).
- Implementation code.
