---
title: "Tier-5: syscall 1 — write(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`write(2)` is **x86_64 syscall 1**, the canonical VFS write entry point. It pushes up to `count` bytes from `buf` to the file behind `fd`, advancing `f_pos` (or honoring `O_APPEND` semantics, which atomically sets `f_pos` to `i_size` under `i_mutex` before each write). Like `read(2)`, every call funnels through `ksys_write` → `vfs_write` → either `f_op->write` or `new_sync_write` (which calls `->write_iter` with a single-segment `iov_iter`). Short writes are legal and must be retried by userspace. `O_NONBLOCK` returns `-EAGAIN` instead of blocking when the destination buffer is full (pipe/socket); `O_SYNC`/`O_DSYNC` flush data (and optionally metadata) before returning. Signals during blocking writes convert to `-EINTR` if no bytes have been transferred yet, otherwise the short byte count is returned.

Critical for: every libc `write()`/`fwrite()`/`printf` flush, every shell pipeline, every log/journal writer, every Rookery network or block I/O write path.

### Acceptance Criteria

- [ ] AC-1: `write(badfd, ...) == -EBADF`.
- [ ] AC-2: `write(fd, NULL, 1) == -EFAULT`.
- [ ] AC-3: `write(read_only_fd, ...) == -EBADF`.
- [ ] AC-4: `write(closed_peer_pipe, ...) == -EPIPE` AND raises `SIGPIPE` to caller.
- [ ] AC-5: `write(full_pipe_O_NONBLOCK, ...) == -EAGAIN`; blocking variant suspends until reader drains.
- [ ] AC-6: `O_APPEND` writers on the same file never interleave (post-condition: each write lands at the new tail).
- [ ] AC-7: `write` exceeding `RLIMIT_FSIZE` returns `-EFBIG` and raises `SIGXFSZ`.
- [ ] AC-8: `O_SYNC` write returns only after data + metadata are on stable storage.
- [ ] AC-9: Signal mid-write with 0 bytes transferred → `-EINTR`; with `n > 0` bytes transferred → returns `n`.
- [ ] AC-10: `write(O_PATH_fd, ...) == -EBADF`.

### Architecture

```
struct WriteArgs { fd: u32, buf: UserPtr<u8>, count: usize }
```

`sys_write(args) -> isize`:

1. `let f = fdget_pos(args.fd)?;`
2. If `!(f.f_mode & FMODE_WRITE)` → drop, return `-EBADF`.
3. Clamp `args.count` to `MAX_RW_COUNT`.
4. `let ppos = if f.f_mode & FMODE_STREAM { None } else { Some(&mut f.f_pos_copy) };`
5. `rw_verify_area(WRITE, f, ppos, args.count)?`.
6. Dispatch:
   - `f.f_op.write` non-NULL → `f.f_op.write(f, args.buf, args.count, ppos)`.
   - Else if `f.f_op.write_iter` → `new_sync_write(f, args.buf, args.count, ppos)`.
   - Else → `-EINVAL`.
7. If `ret >= 0 && ppos.is_some()` → `f.f_pos = *ppos`.
8. `fdput_pos(f);`
9. Return `ret`.

`Vfs::new_sync_write(f, buf, count, ppos) -> isize`:

1. Build single-segment `iov_iter` (`ITER_SOURCE`).
2. `kiocb` with `ki_pos = *ppos`.
3. Call `f.f_op.write_iter(&kiocb, &iter)`.
4. If file has `O_APPEND`, `generic_write_checks` snaps `ki_pos = i_size` under `i_rwsem` write-lock.
5. Wait on `EIOCBQUEUED` if async.
6. `*ppos = kiocb.ki_pos`.
7. Return bytes-written.

### Out of Scope

- `writev` / `pwritev` / `pwritev2` (separate Tier-5).
- `pwrite64` (separate Tier-5).
- `io_uring` `IORING_OP_WRITE` (covered in io_uring Tier-3).
- Page-cache writeback / journaling internals (covered in fs Tier-3 docs).
- Implementation code.

### signature

C (POSIX / man-pages):

```c
ssize_t write(int fd, const void *buf, size_t count);
```

glibc wrapper: `__libc_write` → `INLINE_SYSCALL(write, 3, fd, buf, count)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf, size_t, count);
```

Rookery dispatch:

```rust
pub fn sys_write(fd: u32, buf: UserPtr<u8>, count: usize) -> SyscallResult<isize>;
```

### parameters

| name  | type                  | constraints                                                | errno-on-bad |
|-------|-----------------------|------------------------------------------------------------|--------------|
| fd    | `unsigned int`        | must be open in current `files_struct`; `FMODE_WRITE` set  | `EBADF`      |
| buf   | `const char __user *` | valid userspace range of at least `count` bytes            | `EFAULT`     |
| count | `size_t`              | clamped to `MAX_RW_COUNT` (`INT_MAX & PAGE_MASK`)          | (clamped)    |

### return value

- Success: `>= 0` — number of bytes actually written. May be less than `count`.
- Failure: `< 0` — negated errno.
- A return of `0` is permitted but unusual (e.g., zero-length write on a regular file is a no-op).

### errors

| errno     | condition                                                                                |
|-----------|------------------------------------------------------------------------------------------|
| `EBADF`   | `fd` not open, or open without `FMODE_WRITE` (opened `O_RDONLY` or `O_PATH`).             |
| `EFAULT`  | `buf` (or part of `[buf, buf+count)`) is outside the caller's address space.             |
| `EINVAL`  | Buffer or offset alignment violates `O_DIRECT`, or `fd` doesn't permit `write` (e.g., `O_PATH`). |
| `EFBIG`   | Write would exceed the file size limit (RLIMIT_FSIZE) or filesystem max file size, or off_t. |
| `EPIPE`   | Pipe/socket peer is closed; `SIGPIPE` is also raised unless masked.                       |
| `EAGAIN`  | `O_NONBLOCK` is set and the destination cannot accept any bytes.                          |
| `EINTR`   | Signal delivered before any byte was transferred; no `SA_RESTART`.                        |
| `EIO`     | Low-level I/O error (block layer / device).                                               |
| `ENOSPC`  | Filesystem out of space (or quota exceeded).                                              |
| `EDQUOT`  | Disk quota exceeded.                                                                      |
| `ENOMEM`  | Kernel could not allocate transient buffer.                                               |
| `EROFS`   | File on a read-only filesystem (caught at open in most cases, but possible after remount-ro). |
| `ENXIO`   | Special file's backing object is gone.                                                    |
| `EOVERFLOW` | Position overflow on 32-bit ABI without `O_LARGEFILE`.                                 |
| `EWOULDBLOCK` | Synonym of `EAGAIN`.                                                                 |

### abi surface (constants + flags)

- File-mode bits relevant to write:
  - `FMODE_WRITE` — must be set.
  - `FMODE_CAN_WRITE` — `f_op->write` or `f_op->write_iter` is non-NULL.
  - `FMODE_STREAM` — `ppos` is NULL (no positionable file).
  - `FMODE_PWRITE` — required by `pwrite64` only.
- `MAX_RW_COUNT = INT_MAX & PAGE_MASK` — silent clamp.
- `O_APPEND` (`00002000`) — atomic seek-to-end + write under `i_mutex`.
- `O_SYNC` (`__O_SYNC|O_DSYNC`) — flush before return.
- `O_DSYNC` — data-only sync.
- `O_DIRECT` (`00040000`) — bypass page cache; strict alignment.
- `O_NONBLOCK` — non-blocking mode for non-regular files.
- `RLIMIT_FSIZE` — soft/hard cap on regular-file write extension.

### compatibility contract

- REQ-1: Argument register lowering: `%rdi=fd`, `%rsi=buf`, `%rdx=count`; return in `%rax`.
- REQ-2: `count` silently clamped to `MAX_RW_COUNT`.
- REQ-3: `fd` looked up via `CLASS(fd_pos, f)(fd)` — refcount++ and `f_pos_lock` taken if non-stream.
- REQ-4: `vfs_write` enforces:
  1. `(file->f_mode & FMODE_WRITE) != 0`,
  2. `access_ok(buf, count)`,
  3. `rw_verify_area(WRITE, file, ppos, count) >= 0`,
  4. `f_op->write` or `f_op->write_iter` is non-NULL.
- REQ-5: `O_APPEND` is honored by the underlying filesystem's `->write_iter` (typically via `generic_write_checks` which sets `ki_pos = i_size`); the seek + write is atomic relative to other `O_APPEND` writers on the same inode.
- REQ-6: Short writes are legal; userspace MUST loop.
- REQ-7: Pipe/socket with closed peer ⟹ `-EPIPE` AND `SIGPIPE` raised to current (suppressed by `MSG_NOSIGNAL`/`SIGPIPE`-blocked).
- REQ-8: `O_NONBLOCK` ⟹ `-EAGAIN` instead of blocking.
- REQ-9: `O_SYNC` / `O_DSYNC` ⟹ post-write `vfs_fsync_range`/`generic_write_sync` ensures durability before return.
- REQ-10: Signal during write: if zero bytes written ⟹ `-EINTR`; otherwise return short byte count.
- REQ-11: `write` from `O_PATH` fd → `-EBADF`.
- REQ-12: LSM (`security_file_permission`) invoked with `MAY_WRITE`; can deny with `-EACCES`.
- REQ-13: `RLIMIT_FSIZE` enforced inside `generic_write_checks`; exceeding it ⟹ `-EFBIG` and `SIGXFSZ`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_refcount_balanced` | INVARIANT | `fdget_pos` paired with `fdput_pos`. |
| `append_atomicity` | INVARIANT | Under `O_APPEND`, `i_rwsem` held across pos-snap + write. |
| `signal_partial_progress` | INVARIANT | `-EINTR` ⟺ zero bytes written. |
| `epipe_implies_sigpipe` | INVARIANT | `-EPIPE` ⟹ `SIGPIPE` queued (unless masked / `MSG_NOSIGNAL`). |

### Layer 2: TLA+

`uapi/syscalls/write.tla`:
- Per-call → fdget → verify → dispatch (append-locked) → sync-or-not → fdput.
- Properties:
  - `safety_no_write_without_FMODE_WRITE`,
  - `safety_append_writers_serialize`,
  - `liveness_blocking_write_unblocks_on_drain_or_signal`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Return `> 0` ⟹ exactly `ret` bytes consumed from `buf` | `Vfs::write` |
| `O_SYNC` ⟹ post-call `i_size` and data are durable | `Vfs::sync_after_write` |
| `RLIMIT_FSIZE` ⟹ never exceeded; otherwise `-EFBIG` | `Vfs::generic_write_checks` |

### Layer 4: Verus/Creusot functional

`write(fd, buf, count) ≡ POSIX.1-2024 write` semantic equivalence.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`write(2)` reinforcement:

- **Per-user-pointer strict validation** — `access_ok` first; `copy_from_user` reports residual.
- **Per-`FMODE_WRITE` gate** — defense against read-only fd abuse.
- **Per-`O_APPEND` atomic seek-then-write** — defense against logger interleaving.
- **Per-`RLIMIT_FSIZE` + `SIGXFSZ`** — defense against file-size DoS.
- **Per-`security_file_permission`** — defense against LSM-policy bypass.
- **Per-`-EPIPE` + `SIGPIPE`** — defense against silent loss on broken pipe.
- **Per-`O_SYNC`/`O_DSYNC` durability** — defense against post-crash data loss.
- **Per-`O_DIRECT` alignment checks** — defense against DMA-misalignment hangs.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `buf` validated against `TASK_SIZE`; SMAP guards `copy_from_user`.
- **PAX_USERCOPY** — kernel-side staging buffers slab-bounds-checked.
- **PAX_REFCOUNT** — `fdget_pos`/`fdput_pos` saturating refcounts.
- **GRKERNSEC_RWXMAP_LOG** — write to MAP_EXEC mappings (via mmap'd file) logged.
- **GRKERNSEC_HIDESYM** — block-layer pointers redacted in `EIO` logs.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — transient kernel buffers zeroed on free.
- **GRKERNSEC_DMESG** — write-failure printks rate-limited and CAP_SYSLOG-gated.
- **GRKERNSEC_TPE** — Trusted-Path-Execution may restrict writes to setuid-attributed files.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd writes blocked from escaping fd inheritance.

