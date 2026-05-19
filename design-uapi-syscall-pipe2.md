---
title: "Tier-5: syscall 293 — pipe2(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pipe2(2)` is **x86_64 syscall 293**, the flagged variant of `pipe(2)`. It creates a unidirectional anonymous-inode FIFO, allocating a read-end fd and a write-end fd in `current->files->fdt`, and writes the two fds into `pipefd[0]` and `pipefd[1]`. Unlike `pipe(2)`, `pipe2(2)` accepts a `flags` mask to set `O_CLOEXEC`, `O_NONBLOCK`, `O_DIRECT` (packet-mode), and `O_NOTIFICATION_PIPE` (watch_queue notification ring; gated by `CAP_SYS_ADMIN`) atomically with fd installation, eliminating the `fcntl(F_SETFD)` race that `pipe(2)` requires for CLOEXEC. The underlying `struct pipe_inode_info` ring buffer defaults to `PIPE_DEF_BUFFERS = 16` pages but may grow to `pipe-max-size` (`/proc/sys/fs/pipe-max-size`).

Critical for: every shell pipeline using `|`, every Rookery `posix_spawn`-based child-stdio plumbing, every container-runtime stdio relay, every `keyctl_watch_key`/notify-listener (`O_NOTIFICATION_PIPE`), every io_uring `IORING_OP_SPLICE` between pipes.

### Acceptance Criteria

- [ ] AC-1: `pipe2(fds, 0) == 0`; `fds[0]` is read end, `fds[1]` is write end.
- [ ] AC-2: `write(fds[1], "x", 1); read(fds[0], buf, 1)` returns `1`, `buf == "x"`.
- [ ] AC-3: `pipe2(fds, O_CLOEXEC)` ⟹ both fds have `FD_CLOEXEC` set.
- [ ] AC-4: `pipe2(fds, O_NONBLOCK)` ⟹ `read(fds[0], buf, 1) == -EAGAIN` when empty.
- [ ] AC-5: `pipe2(fds, O_DIRECT)`: two `write(fds[1], buf, n)` calls produce two distinct `read` results.
- [ ] AC-6: `pipe2(fds, O_NOTIFICATION_PIPE)` without `CAP_SYS_ADMIN` ⟹ `-EPERM`.
- [ ] AC-7: `pipe2(fds, 0x80000000) == -EINVAL` (unknown flag).
- [ ] AC-8: `pipe2(NULL, 0) == -EFAULT`.
- [ ] AC-9: At `RLIMIT_NOFILE-1`: `pipe2` ⟹ `-EMFILE` (no partial install).
- [ ] AC-10: After `EFAULT` on `copy_to_user`, no fds are visible in `/proc/$pid/fd`.

### Architecture

```
struct Pipe2Args { pipefd: UserPtr<[i32; 2]>, flags: i32 }
```

`sys_pipe2(args) -> i32`:

1. If `args.flags & !VALID_PIPE2_FLAGS` ⟹ return `-EINVAL`.
2. If `(args.flags & O_NOTIFICATION_PIPE) && !capable(CAP_SYS_ADMIN)` ⟹ return `-EPERM`.
3. Return `do_pipe2(args.pipefd, args.flags)`.

`Fs::do_pipe2(uptr, flags) -> i32`:

1. `let (read_file, write_file) = create_pipe_files(flags)?;` — `-ENFILE`/`-ENOMEM` on failure.
2. `let read_fd = get_unused_fd_flags(flags & O_CLOEXEC)?;` else release files ⟹ `-EMFILE`.
3. `let write_fd = get_unused_fd_flags(flags & O_CLOEXEC)?;` else put `read_fd`, release files ⟹ `-EMFILE`.
4. `copy_to_user(uptr, &[read_fd, write_fd])?;` else put both fds, release files ⟹ `-EFAULT`.
5. `fd_install(read_fd, read_file);`
6. `fd_install(write_fd, write_file);`
7. Return `0`.

`Fs::create_pipe_files(flags) -> Result<(File, File), errno>`:

1. `let pipe = alloc_pipe_info()?;` — `-ENOMEM`.
2. `let inode = alloc_anon_inode(&pipe_mnt, pipe)?;`
3. `let read_file = alloc_file_pseudo(inode, O_RDONLY | (flags & ALLOW_MASK), pipefifo_fops)?;`
4. `let write_file = alloc_file_pseudo(inode, O_WRONLY | (flags & ALLOW_MASK), pipefifo_fops)?;`
5. Set `f_op = watch_queue_pipe_fops` if `O_NOTIFICATION_PIPE`.
6. Return `(read_file, write_file)`.

### Out of Scope

- `pipe(2)` (legacy 1-arg; semantically `pipe2(fds, 0)`).
- `watch_queue` / `keyctl_watch_key` event sourcing (covered in keys/ Tier-3).
- `splice`/`tee`/`vmsplice` pipe-data movement (separate Tier-5 if expanded).
- Implementation code.

### signature

C (Linux-specific):

```c
int pipe2(int pipefd[2], int flags);
```

glibc wrapper: `__libc_pipe2` → `INLINE_SYSCALL(pipe2, 2, pipefd, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(pipe2, int __user *, fildes, int, flags);
```

Rookery dispatch:

```rust
pub fn sys_pipe2(pipefd: UserPtr<[i32; 2]>, flags: i32) -> SyscallResult<i32>;
```

### parameters

| name    | type            | constraints                                                                | errno-on-bad |
|---------|-----------------|----------------------------------------------------------------------------|--------------|
| pipefd  | `int [2]`       | writable for 8 bytes; 4-byte aligned                                       | `EFAULT` |
| flags   | `int`           | subset of `{O_CLOEXEC, O_NONBLOCK, O_DIRECT, O_NOTIFICATION_PIPE}`; else `-EINVAL` | `EINVAL` |

### return value

- Success: `0`; `pipefd[0]` (read end) and `pipefd[1]` (write end) populated.
- Failure: `< 0` — negated errno; `*pipefd` unchanged.

### errors

| errno    | condition                                                                  |
|----------|----------------------------------------------------------------------------|
| `EINVAL` | `flags` contains an unrecognized bit.                                      |
| `EFAULT` | `pipefd` is not writable.                                                  |
| `ENFILE` | System-wide open-file limit reached.                                       |
| `EMFILE` | `RLIMIT_NOFILE` reached (need two free slots).                             |
| `ENOMEM` | Pipe inode / buffer allocation failed.                                     |
| `EPERM`  | `O_NOTIFICATION_PIPE` requested without `CAP_SYS_ADMIN`.                   |

### abi surface (constants + flags)

`flags` (`include/uapi/linux/fcntl.h`, `include/uapi/linux/pipe_fs_i.h`):

- `O_CLOEXEC = 0x80000` — set close-on-exec on both fds atomically.
- `O_NONBLOCK = 0x800` — set non-blocking I/O on both fds.
- `O_DIRECT = 0x4000` — packet-mode pipe; each `write` is delivered as a single `read` (boundary-preserving up to `PIPE_BUF`).
- `O_NOTIFICATION_PIPE = O_EXCL = 0x80` (reused bit, scoped to `pipe2`) — turn the pipe into a watch_queue notification ring (kernel writes notifications via `post_one_notification`; the write end is sealed); requires `CAP_SYS_ADMIN` and `CONFIG_WATCH_QUEUE`.

`PIPE_BUF` ABI guarantees:

- `PIPE_BUF = 4096` — atomicity boundary: writes of `<= PIPE_BUF` bytes to a pipe are atomic.
- `PIPE_DEF_BUFFERS = 16` — default ring-buffer slot count.
- `pipe-max-size` (sysctl) — per-pipe maximum capacity in bytes.

Related kernel symbols:

- `struct pipe_inode_info` — ring of `struct pipe_buffer`.
- `create_pipe_files(files, flags)` — allocates anonymous-inode read/write `struct file` pair sharing one `pipe_inode_info`.
- `alloc_pipe_info()` — allocates the ring buffer.
- `pipefifo_fops` — `f_op` for normal pipes; `watch_queue_pipe_fops` for `O_NOTIFICATION_PIPE`.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=pipefd ptr`, `%rsi=flags (i32)`.
- REQ-2: `flags & ~(O_CLOEXEC|O_NONBLOCK|O_DIRECT|O_NOTIFICATION_PIPE) ⟹ -EINVAL` before any allocation.
- REQ-3: `O_NOTIFICATION_PIPE` set without `CAP_SYS_ADMIN` ⟹ `-EPERM`.
- REQ-4: Allocate two `struct file` instances sharing one `pipe_inode_info` ring buffer:
  - read end: `O_RDONLY | (flags & (O_NONBLOCK | O_DIRECT))`.
  - write end: `O_WRONLY | (flags & (O_NONBLOCK | O_DIRECT))`.
- REQ-5: Both files get `f_op = pipefifo_fops` (or `watch_queue_pipe_fops` if `O_NOTIFICATION_PIPE`).
- REQ-6: Allocate two fds atomically via `__get_unused_fd_flags(O_CLOEXEC if requested)`; if either fails, release the other and the `struct file`s; return `-EMFILE`.
- REQ-7: `copy_to_user(pipefd, [read_fd, write_fd], 8)`; on failure, close both fds (which calls `filp_close` ⟹ pipe teardown) and return `-EFAULT`.
- REQ-8: `fd_install(read_fd, read_file)` then `fd_install(write_fd, write_file)` — both atomically published to fdtable.
- REQ-9: On any failure after fd alloc but before `fd_install`, both fds are returned to the unused pool via `put_unused_fd`.
- REQ-10: `O_DIRECT` pipes preserve write boundaries up to `PIPE_BUF`; reads see one packet per call.
- REQ-11: `O_NONBLOCK` propagates to both ends; toggleable later via `fcntl(F_SETFL)`.
- REQ-12: The pipe's ring buffer starts at `PIPE_DEF_BUFFERS` pages; resizable later via `fcntl(F_SETPIPE_SZ)` up to `pipe-max-size`.
- REQ-13: Notification pipes (`O_NOTIFICATION_PIPE`) reject userspace writes (`-EBADF`); only kernel-side `post_one_notification` may append.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated_first` | INVARIANT | Unknown bits ⟹ `-EINVAL` before allocation. |
| `notification_pipe_capability` | INVARIANT | `O_NOTIFICATION_PIPE ⟹ CAP_SYS_ADMIN`. |
| `no_partial_install_on_efault` | INVARIANT | `copy_to_user` failure ⟹ no fd visible in `fdt`. |
| `cloexec_atomic_both_ends` | INVARIANT | `O_CLOEXEC ⟹ both fds have FD_CLOEXEC set at fd_install time`. |
| `direct_packet_boundary` | INVARIANT | `O_DIRECT ⟹ pipe_buffer per write up to PIPE_BUF`. |

### Layer 2: TLA+

`uapi/syscalls/pipe2.tla`:
- Per-call → validate → create_pipe_files → alloc_fds → copy_to_user → fd_install.
- Properties:
  - `safety_atomic_both_or_neither`,
  - `safety_pipe_buf_preserves_write`,
  - `liveness_pipe2_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: read_file.f_mode includes FMODE_READ only; write_file.f_mode includes FMODE_WRITE only | `Fs::create_pipe_files` |
| Post: both files share the same `pipe_inode_info` | `Fs::create_pipe_files` |
| Post: success ⟹ `pipefd[0]` and `pipefd[1]` both in `fdt`; failure ⟹ neither | `Fs::do_pipe2` |

### Layer 4: Verus/Creusot functional

`pipe2(fds, flags)` ≡ Linux pipe2(2) per `man 2 pipe2`; semantics align with POSIX `pipe(2)` extended with flag mask.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pipe2(2)` reinforcement:

- **Per-flag-strict mask** — defense against silent forward-compat acceptance of unknown bits.
- **Per-`O_NOTIFICATION_PIPE` cap gate** — defense against unprivileged kernel-event subscription.
- **Per-`copy_to_user` post-allocation** — defense against fd-leak on userspace bad pointer.
- **Per-atomic-CLOEXEC** — defense against `pipe(2)` + `fcntl` race exposing fds to a concurrent `execve`.
- **Per-`alloc_anon_inode`** — defense against pipe inode appearing in mount tables.
- **Per-`pipe-max-size` sysctl ceiling** — defense against per-pipe memory-exhaustion DoS.
- **Per-`watch_queue_pipe_fops`** — defense against userspace forging kernel notifications.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `pipefd` user buffer SMAP-guarded in `copy_to_user`.
- **PAX_REFCOUNT** — `pipe_inode_info` and `struct file` refcounts saturating.
- **GRKERNSEC_FIFO** — pipe2-created FIFO inherits sticky-dir FIFO restrictions when later mounted/named.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd pipe2 stays within jail; no path escape.
- **GRKERNSEC_HIDESYM** — pipe2-error printks redact kernel pointers.
- **PAX_MEMORY_SANITIZE** — `pipe_inode_info` slab zeroed on free.
- **PAX_RANDKSTACK** — kstack offset randomized at pipe2 syscall entry.
- **GRKERNSEC_DMESG** — pipe2-error dmesg lines CAP_SYSLOG-gated.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/fd` shows pipe inode only to owner.
- **GRKERNSEC_FORKFAIL** — pipe2 ENFILE/EMFILE logged for fork-bomb / fd-exhaustion heuristics.
- **GRKERNSEC_AUDIT_IPC** — `O_NOTIFICATION_PIPE` creation auditable.

