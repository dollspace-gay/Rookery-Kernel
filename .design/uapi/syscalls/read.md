# Tier-5: syscall 0 — read(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`0  common  read  sys_read`)
  - fs/read_write.c (`SYSCALL_DEFINE3(read, ...)`, `ksys_read`, `vfs_read`, `rw_verify_area`, `new_sync_read`, `file_ppos`)
  - include/linux/fs.h (`struct file`, `FMODE_READ`, `FMODE_STREAM`, `FMODE_CAN_READ`, `FMODE_PREAD`)
  - include/linux/syscalls.h (`asmlinkage long sys_read(unsigned int, char __user *, size_t)`)
-->

## Summary

`read(2)` is **x86_64 syscall 0** and the canonical VFS read entry point. It pulls up to `count` bytes from the file referenced by `fd` into the userspace buffer `buf`, advancing the per-`struct file` position `f_pos` (unless `FMODE_STREAM` is set, in which case the file has no kernel-tracked position). The kernel funnels every call through `ksys_read` → `vfs_read` → either `file->f_op->read` or `new_sync_read` (which calls `->read_iter` with a single-segment `iov_iter`). Result semantics match POSIX: a short read (including zero at EOF) is success, `-1` with `errno` set indicates failure. `O_NONBLOCK` files return `-EAGAIN` instead of blocking when no data is immediately available; signals during blocking I/O convert to `-EINTR` (or are restarted by `SA_RESTART`).

Critical for: every libc `read()`/`fread()`/`getline()` caller, every shell pipeline stage, every Rookery storage / network read path, every userspace `select`/`poll`/`epoll`-driven event loop.

## Signature

C (POSIX / man-pages):

```c
ssize_t read(int fd, void *buf, size_t count);
```

glibc wrapper: `__libc_read` → `INLINE_SYSCALL(read, 3, fd, buf, count)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count);
```

Rookery dispatch:

```rust
pub fn sys_read(fd: u32, buf: UserPtr<u8>, count: usize) -> SyscallResult<isize>;
```

## Parameters

| name  | type             | constraints                                              | errno-on-bad |
|-------|------------------|----------------------------------------------------------|--------------|
| fd    | `unsigned int`   | must be open in current `files_struct`; `FMODE_READ` set | `EBADF`      |
| buf   | `char __user *`  | must be a valid userspace region of at least `count`     | `EFAULT`     |
| count | `size_t`         | clamped to `MAX_RW_COUNT` (`INT_MAX & PAGE_MASK`)        | (clamped, not error) |

## Return value

- Success: `>= 0` — number of bytes actually read (0 = EOF for regular files, or end-of-stream for pipes/sockets with peer closed).
- Failure: `< 0` — negated errno per `include/uapi/asm-generic/errno-base.h` / `errno.h`.

## Errors

| errno     | condition                                                                                |
|-----------|------------------------------------------------------------------------------------------|
| `EBADF`   | `fd` not open, or open without `FMODE_READ` (e.g., opened `O_WRONLY` or `O_PATH`).       |
| `EFAULT`  | `buf` (or part of `[buf, buf+count)`) is outside the caller's address space.             |
| `EINVAL`  | `fd` is attached to an object that does not permit `read(2)` (e.g., `O_DIRECTORY`), or offset/buffer unaligned for `O_DIRECT`. |
| `EISDIR`  | `fd` refers to a directory (regular `read` is rejected, must use `getdents64`).          |
| `EAGAIN`  | `O_NONBLOCK` is set and no data is currently available (pipe empty, socket buffer empty, etc.). |
| `EINTR`   | A signal was delivered before any data was transferred and the handler did not restart. |
| `EIO`     | A low-level I/O error (block layer ENOMEDIUM/EREMOTEIO/etc.).                            |
| `ENOMEM`  | Kernel could not allocate transient buffers (rare; mostly cgroup-OOM).                  |
| `ESPIPE`  | (`pread64` cousin only — not raised by plain `read`).                                    |
| `ENXIO`   | Read of a special file (e.g., `/dev/...`) whose backing object has gone away.            |
| `EOVERFLOW` | Position would overflow `off_t` (32-bit ABI without `O_LARGEFILE`).                    |
| `EWOULDBLOCK` | Synonym of `EAGAIN`.                                                                 |

## ABI surface (constants + flags)

- File-mode bits checked in `vfs_read`:
  - `FMODE_READ` — must be set, else `-EBADF`.
  - `FMODE_CAN_READ` — must be set (file_operations has `read` or `read_iter`).
  - `FMODE_STREAM` — file has no positionable `f_pos`; `ppos` passed as `NULL`.
  - `FMODE_PREAD` — required by `pread64` only.
- `MAX_RW_COUNT = INT_MAX & PAGE_MASK` — count is silently clamped to this.
- Per-`O_NONBLOCK` (`00004000`) — non-blocking mode for character/socket/pipe file types.
- Per-`O_DIRECT` (`00040000`) — bypasses page cache; imposes alignment requirements on `buf`, `count`, and `f_pos`.
- Per-`O_APPEND` — irrelevant on read.
- Per-`O_PATH` — read is rejected (`EBADF`).

## Compatibility contract

- REQ-1: Argument register lowering follows the SysV AMD64 syscall ABI: `%rdi=fd`, `%rsi=buf`, `%rdx=count`; return in `%rax`. Errors are negative `int` values (negated errno) that glibc converts to `-1 + errno`.
- REQ-2: `count` is silently truncated to `MAX_RW_COUNT`; no error is returned even for `count > SSIZE_MAX`.
- REQ-3: `fd` is looked up via `CLASS(fd_pos, f)(fd)` which both increments the refcount and locks `f_pos_lock` (so concurrent stream reads serialize on the position).
- REQ-4: `vfs_read` enforces:
  1. `(file->f_mode & FMODE_READ) != 0`,
  2. `access_ok(buf, count)`,
  3. `rw_verify_area(READ, file, ppos, count) >= 0` (mandatory locks, file-range LSM hooks),
  4. either `f_op->read` or `f_op->read_iter` is non-NULL.
- REQ-5: When `FMODE_STREAM` is set, the operation receives `ppos == NULL`; otherwise a local `pos` copy is passed and written back to `file->f_pos` on success.
- REQ-6: Short reads are legal at any layer; the caller MUST loop. EOF is signalled by a return of 0 (regular files) or by 0 with no peer (sockets/pipes).
- REQ-7: For `O_NONBLOCK` file types where data is not ready, the operation returns `-EAGAIN` and does not block.
- REQ-8: Blocking reads are restartable: `ERESTARTSYS` propagates to the signal-delivery code; with `SA_RESTART` the syscall replays, otherwise it returns `-EINTR`.
- REQ-9: A signed-to-unsigned cast on the return value MUST NOT lose information: kernel returns `ssize_t` ≤ `MAX_RW_COUNT` ≤ `INT_MAX`.
- REQ-10: `read` on a directory unconditionally returns `-EISDIR` (POSIX-violating Linux extension).
- REQ-11: `read` from an `O_PATH` fd returns `-EBADF` (the open file has no read capability).
- REQ-12: LSM (`security_file_permission`) is invoked with `MAY_READ`; SELinux / AppArmor / Smack may deny with `-EACCES`.

## Acceptance Criteria

- [ ] AC-1: `read(badfd, ...) == -EBADF` for an fd not in `current->files`.
- [ ] AC-2: `read(fd, NULL, 1) == -EFAULT`; `read(fd, kernel_addr, 1) == -EFAULT`.
- [ ] AC-3: `read(write_only_fd, ...) == -EBADF`.
- [ ] AC-4: `read(dir_fd, ...) == -EISDIR`.
- [ ] AC-5: `read(empty_pipe_O_NONBLOCK, ...) == -EAGAIN`; same fd blocking → suspends until peer writes.
- [ ] AC-6: Signal during blocking `read` → `-EINTR` unless `SA_RESTART`.
- [ ] AC-7: Reading `count > MAX_RW_COUNT` succeeds and returns at most `MAX_RW_COUNT` bytes (no overflow error).
- [ ] AC-8: After short read on a regular file, `f_pos` advances by exactly `ret` bytes.
- [ ] AC-9: Two concurrent `read` calls on the same fd serialize on `f_pos_lock` (stream invariant: positions never interleave).
- [ ] AC-10: Reading from `O_PATH` fd → `-EBADF`.

## Architecture

```
struct ReadArgs { fd: u32, buf: UserPtr<u8>, count: usize }
```

`sys_read(args) -> isize`:

1. `let f = fdget_pos(args.fd)?;` — refcount++; if `FMODE_STREAM` clear, take `f_pos_lock`.
2. If `!(f.f_mode & FMODE_READ)` → drop `f`, return `-EBADF`.
3. If `args.count > MAX_RW_COUNT` → `args.count = MAX_RW_COUNT`.
4. `let ppos = if f.f_mode & FMODE_STREAM { None } else { Some(&mut f.f_pos_copy) };`
5. `rw_verify_area(READ, f, ppos, args.count)?` — short-circuits on mandatory-lock conflict or LSM denial.
6. `let ret = if let Some(rd) = f.f_op.read { rd(f, args.buf, args.count, ppos) } else if f.f_op.read_iter.is_some() { new_sync_read(f, args.buf, args.count, ppos) } else { -EINVAL };`
7. On `ret >= 0 && ppos.is_some()` → `f.f_pos = *ppos`.
8. `fdput_pos(f);`
9. Return `ret`.

`Vfs::new_sync_read(f, buf, count, ppos) -> isize`:

1. Build single-segment `iov_iter` (`ITER_DEST`).
2. Construct `kiocb` with `f.f_pos`.
3. Call `f.f_op.read_iter(&kiocb, &iter)`.
4. If `-EIOCBQUEUED` returned (async path) → wait on completion.
5. Update `*ppos = kiocb.ki_pos`.
6. Return bytes-read.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_refcount_balanced` | INVARIANT | `fdget_pos` paired with `fdput_pos` on every path. |
| `f_pos_lock_held_for_stream` | INVARIANT | When `!FMODE_STREAM`, `f_pos_lock` is held across `read_iter`. |
| `count_clamped` | INVARIANT | Post-clamp `count <= MAX_RW_COUNT`. |
| `user_buf_validated` | INVARIANT | `access_ok(buf, count)` checked before any copy. |

### Layer 2: TLA+

`uapi/syscalls/read.tla`:
- Per-call → fdget → verify → dispatch → fdput.
- Properties:
  - `safety_no_read_without_FMODE_READ`,
  - `safety_no_dir_read`,
  - `liveness_blocking_read_unblocks_on_data_or_signal`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Return `>= 0` ⟹ exactly `ret` bytes copied to `buf` | `Vfs::read` |
| Return `< 0` ⟹ no bytes copied, `f_pos` unchanged | `Vfs::read` |
| `EAGAIN` ⟹ `O_NONBLOCK` was set | `Vfs::nonblock_dispatch` |

### Layer 4: Verus/Creusot functional

`read(fd, buf, count) ≡ POSIX.1-2024 read` semantic equivalence: per `man 2 read`, per fs/read_write.c control flow.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`read(2)` reinforcement:

- **Per-user-pointer strict validation** — `access_ok` first, then `copy_to_user` returns the unwritten residual; never trust `buf`.
- **Per-`MAX_RW_COUNT` clamp** — defense against `count > INT_MAX` integer-conversion bugs.
- **Per-`FMODE_READ` gate** — defense against using write-only fds for data exfiltration.
- **Per-`f_pos_lock`** — defense against TOCTOU on the file position between concurrent reads.
- **Per-`O_PATH` strict reject** — defense against `O_PATH`-to-data confusion.
- **Per-mandatory-lock observance via `rw_verify_area`** — defense against bypassing filesystem locking.
- **Per-`security_file_permission` LSM hook** — defense against policy bypass.
- **Per-signal-restart classification** — defense against silent EOF when a signal interrupts (return `-EINTR`, not 0).

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `buf` must live above `TASK_SIZE` enforcement boundary; `copy_to_user` uses SMAP-guarded primitives.
- **PAX_USERCOPY** — bounce-buffer / slab-allocation sanity checks for kernel-side staging during `read_iter`.
- **PAX_REFCOUNT** — `fdget_pos`/`fdput_pos` use saturating refcounts; underflow ⟹ kernel `BUG()`.
- **GRKERNSEC_HIDESYM** — `dmesg` / klog suppress kernel pointers on `EIO` printouts.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/fd` view restricted; userspace cannot probe other tasks' fd tables.
- **GRKERNSEC_CHROOT_FINDTASK** — chroot'd tasks cannot read from fds inherited beyond their root.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry so on-stack buffers used during `read_iter` are unpredictable.
- **PAX_MEMORY_SANITIZE** — slab pages freed by transient read buffers are sanitized.
- **GRKERNSEC_DMESG** — block-layer error messages from `EIO` paths are CAP_SYSLOG-gated.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `readv` / `preadv` / `preadv2` (separate Tier-5 docs).
- `pread64` (separate Tier-5).
- `io_uring` `IORING_OP_READ` (covered in io_uring Tier-3).
- Block-layer / page-cache backing path (covered in `fs/buffered-io.md` Tier-3).
- Implementation code.
