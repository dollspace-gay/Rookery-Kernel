# Tier-5 syscall: vmsplice(2) — syscall 278

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/splice.c (SYSCALL_DEFINE4(vmsplice), vmsplice_to_pipe, vmsplice_to_user)
  - include/linux/pipe_fs_i.h (struct pipe_inode_info, pipe_buffer)
  - include/uapi/linux/uio.h (struct iovec)
  - include/uapi/linux/fcntl.h (SPLICE_F_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (278 common vmsplice)
-->

## Summary

`vmsplice(2)` moves data **between user-space buffers and a pipe** with reduced copying. From-user mode (`fd` is a writable pipe end): the kernel grabs references to the user pages described by `iov` and installs them as pipe buffers — no actual copy occurs unless the user later writes to those pages while the pipe still holds them. To-user mode (`fd` is a readable pipe end, requires `SPLICE_F_GIFT` semantics historically): the kernel copies pipe contents into user buffers.

Because vmsplice can install user pages directly into a kernel-shared pipe buffer, it has been a recurring source of high-severity vulnerabilities (most famously "Dirty Pipe", CVE-2022-0847). Critical for: zero-copy producer-consumer, `splice(2)` to/from pipes, high-throughput logging.

## Signature

```c
ssize_t vmsplice(int fd,
                 const struct iovec *iov,
                 unsigned long nr_segs,
                 unsigned int flags);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Pipe end. Writable end ⟹ from-user; readable end ⟹ to-user. Non-pipe ⟹ `-EBADF`. |
| `iov` | `const struct iovec *` | in | User array of `{base, len}` describing source/dest buffers. |
| `nr_segs` | `unsigned long` | in | Array length; `<= UIO_MAXIOV` (= 1024). |
| `flags` | `unsigned int` | in | `SPLICE_F_MOVE | SPLICE_F_NONBLOCK | SPLICE_F_MORE | SPLICE_F_GIFT`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Bytes spliced (may be less than total iov for partial). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` is not a pipe. |
| `EINVAL` | `nr_segs > UIO_MAXIOV`, reserved flag bit set, attempt to splice to non-writable pipe end. |
| `ENOMEM` | Page reference / pipe buffer allocation failure. |
| `EAGAIN` | `SPLICE_F_NONBLOCK` and pipe full / empty. |
| `EFAULT` | `iov` or `iov[i].iov_base` not accessible. |

## ABI surface

```text
__NR_vmsplice  (x86_64) = 278
__NR_vmsplice  (arm64)  = 75
__NR_vmsplice  (riscv)  = 75
__NR_vmsplice  (i386)   = 316

#define SPLICE_F_MOVE      (0x01)
#define SPLICE_F_NONBLOCK  (0x02)
#define SPLICE_F_MORE      (0x04)
#define SPLICE_F_GIFT      (0x08)
```

## Compatibility contract

REQ-1: Syscall number is **278** on x86_64. ABI-stable.

REQ-2: `fd` must be a pipe (`S_ISFIFO`); validated via `is_pipe(file)`. Otherwise `-EBADF`.

REQ-3: Direction inferred from `fd`'s `FMODE_*`: writable pipe end ⟹ from-user; readable end ⟹ to-user.

REQ-4: `nr_segs > UIO_MAXIOV` ⟹ `-EINVAL`.

REQ-5: `flags & ~SPLICE_F_ALL` ⟹ `-EINVAL`.

REQ-6: From-user mode (`vmsplice_to_pipe`):
- Import iovec via `import_iovec(WRITE, iov, nr_segs, ...)`.
- For each segment: `iov_iter_get_pages_alloc` pins user pages.
- Install pages into pipe buffers; refcount transfer.
- If `SPLICE_F_GIFT`, ownership donated; kernel may mark pages copy-on-write or steal.
- Without `GIFT`, pages are reference-shared; user must NOT modify until pipe consumer has drained (else memory-aliasing).
- Return bytes installed.

REQ-7: To-user mode (`vmsplice_to_user`): copy pipe buffers into user iov; bytes returned.

REQ-8: `SPLICE_F_NONBLOCK`:
- From-user: full pipe ⟹ `-EAGAIN` (instead of blocking).
- To-user: empty pipe ⟹ `-EAGAIN`.

REQ-9: `SPLICE_F_MORE` is a hint for downstream `splice(2)` to coalesce further data; vmsplice itself ignores beyond passthrough.

REQ-10: Maximum splice per call ≤ `PIPE_BUF_NR * PAGE_SIZE` (== pipe ring slots × page size).

REQ-11: Splice-from-anon-pipe-fd validation: kernel must reject if `fd` was originally anonymous pipe AND the file's `pipe_inode_info->files` is not writable in caller's cred view (closed mirror prevents the historical `Dirty Pipe` CoW shadow bug pattern).

REQ-12: Per-segment page pinning uses `pin_user_pages_fast` (GUP+pin) to ensure `MMU notifiers` see the long-term reference and `mmap` writes do not race.

REQ-13: F_SETPIPE_SZ-changed pipe capacity respected; vmsplice will not exceed current ring size.

REQ-14: O_DIRECT not applicable (pipes are not block-backed).

REQ-15: SECCOMP: vmsplice is reachable by default and is a common syscall in container escape filters; not blocked by default seccomp profile.

## Acceptance Criteria

- [ ] AC-1: `vmsplice(pipe_w, iov, n, 0)` returns bytes installed; reader sees same data.
- [ ] AC-2: `vmsplice(non_pipe, ...)` returns -EBADF.
- [ ] AC-3: `vmsplice(pipe_r, ...)` with iov in from-user direction: -EBADF (wrong end).
- [ ] AC-4: `nr_segs = 2048`: -EINVAL.
- [ ] AC-5: `flags = 0x100`: -EINVAL.
- [ ] AC-6: `SPLICE_F_NONBLOCK` and pipe full: -EAGAIN.
- [ ] AC-7: `SPLICE_F_GIFT`: writer modifying source post-vmsplice does NOT change pipe-read data.
- [ ] AC-8: Without `SPLICE_F_GIFT`: writer modifying source post-vmsplice changes pipe-read data (aliasing observed).
- [ ] AC-9: Dirty-Pipe regression: vmsplice into anon-pipe followed by `splice(pipe, file)` to read-only file does NOT mutate the file's page cache.
- [ ] AC-10: `iov_base` faulting: -EFAULT.
- [ ] AC-11: Pipe size 64KB, vmsplice 1MB: returns 64KB (short).

## Architecture

```rust
#[syscall(nr = 278, abi = "sysv")]
pub fn sys_vmsplice(fd: i32,
                    iov: UserPtr<IoVec>,
                    nr_segs: u64,
                    flags: u32) -> isize {
    Vmsplice::do_vmsplice(fd, iov, nr_segs as usize, flags)
}
```

`Vmsplice::do_vmsplice(fd, iov, nr_segs, flags) -> isize`:
1. let file = fd_to_file(fd).map_err(|_| EBADF)?;
2. if !file.is_pipe() { return -EBADF; }
3. let pipe = file.pipe().unwrap();
4. if nr_segs > UIO_MAXIOV { return -EINVAL; }
5. if (flags & !SPLICE_F_ALL) != 0 { return -EINVAL; }
6. let iter = import_iovec(/*write=*/file.is_writable(), iov, nr_segs)?;
7. /* Splice-from-anon-pipe-fd validation. */
8. Vmsplice::validate_pipe_fd_origin(&file, &pipe)?;
9. if file.is_writable() {
10.  Vmsplice::vmsplice_to_pipe(pipe, iter, flags)
11. } else if file.is_readable() {
12.  Vmsplice::vmsplice_to_user(pipe, iter, flags)
13. } else {
14.  -EBADF
15. }

`Vmsplice::vmsplice_to_pipe(pipe, iter, flags) -> isize`:
1. let mut total = 0isize;
2. while iter.has_more() && pipe.has_slots() {
3.   if (flags & SPLICE_F_NONBLOCK) != 0 && pipe.full() { break; }
4.   let (pages, off, len) = iov_iter_get_pages_alloc(iter, pipe.free_pages())?;
5.   for (i, page) in pages.iter().enumerate() {
6.     let buf = PipeBuffer { page: page.clone(), offset: off + i * PAGE_SIZE, len: ..., ops: &user_pipe_buf_ops, flags: if (flags & SPLICE_F_GIFT) != 0 { GIFT } else { 0 } };
7.     pipe.push(buf);
8.   }
9.   total += chunk_len as isize;
10.  if (flags & SPLICE_F_NONBLOCK) == 0 && pipe.full() {
11.    wait_for_slot(pipe)?;
12.  }
13. }
14. wake_up_interruptible(pipe.wait_queue());
15. return total;

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_is_pipe` | INVARIANT | non-pipe ⟹ EBADF before any user access. |
| `nr_segs_bounded` | INVARIANT | nr_segs > UIO_MAXIOV ⟹ EINVAL. |
| `flags_reserved_zero` | INVARIANT | flags & ~SPLICE_F_ALL ⟹ EINVAL. |
| `pages_pinned_for_lifetime` | INVARIANT | every page installed into pipe buf has pin_user_pages ref. |
| `gift_pages_marked_owned` | INVARIANT | SPLICE_F_GIFT ⟹ pipe_buffer.flags & GIFT. |
| `pipe_buf_ops_set` | INVARIANT | every installed pipe_buffer has user_pipe_buf_ops. |

### Layer 2: TLA+

`fs/vmsplice.tla`:
- States: per-pipe ring, per-iov segment, per-page pin count.
- Properties:
  - `safety_no_kernel_page_in_user_pipe` — no kernel-backed page installed into user-fed pipe.
  - `safety_pin_balanced` — every pin paired with a release on consumer-drop.
  - `safety_gift_releases_user_ref` — GIFT pages released from user mm.
  - `liveness_blocking_wakes` — blocking vmsplice wakes when slot frees.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_vmsplice` post: EBADF for non-pipe | `Vmsplice::do_vmsplice` |
| `vmsplice_to_pipe` post: installed bytes ≤ pipe capacity | `Vmsplice::vmsplice_to_pipe` |
| `vmsplice_to_pipe` post: every pipe_buffer.ops == user_pipe_buf_ops | `Vmsplice::vmsplice_to_pipe` |
| `validate_pipe_fd_origin` post: anon-pipe origin recorded; downstream splice cannot mutate ro file | `Vmsplice::validate_pipe_fd_origin` |

### Layer 4: Verus/Creusot functional

Per-`vmsplice(2)` man page + `tools/testing/selftests/splice/` + Dirty-Pipe regression test pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`vmsplice(2)` reinforcement:

- **Per-fd pipe-type strict check** — defense against per-non-pipe deref.
- **Per-page pin_user_pages_fast (not get_user_pages)** — defense against per-mmu-notifier race.
- **Per-pipe-buffer ops set to user_pipe_buf_ops** — defense against per-Dirty-Pipe (initial-flags-uninitialized) class.
- **Per-iov import via import_iovec** — defense against per-pointer-truncation on compat.
- **Per-SPLICE_F_GIFT mark** — defense against per-aliasing-mutation surprise.
- **Per-nr_segs ≤ UIO_MAXIOV** — defense against per-iov-array-overflow.
- **Per-pipe-ring bounded by F_SETPIPE_SZ** — defense against per-unbounded-allocation.

## Grsecurity / PaX surface

- **PaX UDEREF on iovec copy_from_user** — defense against per-iov kernel-deref bug; SMAP forced.
- **Splice-from-anon-pipe-fd validation** — grsec adds an explicit origin-check: a pipe created via `pipe(2)`/`pipe2(2)` carries an immutable "anon" tag, and a subsequent `splice(anon_pipe, file)` to a read-only or write-protected file is rejected with EPERM. This is the structural fix that obsoletes the CVE-2022-0847 / Dirty-Pipe class regardless of `PIPE_BUF_FLAG_CAN_MERGE` initial state.
- **PAX_USERCOPY_HARDEN on per-page pin** — pinned user pages must come from whitelisted slab / VMA types; PFNMAP and HUGETLB GIFT paths are rejected.
- **GRKERNSEC_PIPE_AUDIT** — vmsplice with GIFT logged to `grsec_audit` when destination pipe is later spliced to a file the caller cannot directly write.
- **PAX_REFCOUNT on pipe_buffer / page refcount** — defense against per-refcount-overflow UAF.
- **GRKERNSEC_TPE on vmsplice** — if Trusted Path Execution is enabled, vmsplice on a pipe whose other end is owned by a non-trusted UID is rejected when the caller is trusted (prevents zero-copy attacker-controlled data injection into a privileged consumer).
- **Per-pipe owner-cred snapshot** — defense against per-cred-elevation between pipe-create and vmsplice; the cred at vmsplice time is compared against the cred at pipe-create for GIFT mode.
- **PaX KERNEXEC on pipe_buf_ops vtable** — read-only; defense against per-ops-overwrite exploit.
- **Per-PIPE_BUF_FLAG_CAN_MERGE forced false on vmsplice-installed buffers** — explicit zeroing of flags rather than carrying over from prior buffer; structural Dirty-Pipe fix.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `splice(2)` between file and pipe (covered in `splice.md`).
- `tee(2)` pipe-to-pipe (covered in `tee.md`).
- Page-cache COW semantics (covered in mm Tier-3).
- Implementation code.
