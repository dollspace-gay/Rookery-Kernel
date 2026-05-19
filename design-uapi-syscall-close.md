---
title: "Tier-5: syscall 3 — close(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`close(2)` is **x86_64 syscall 3**, the file-descriptor release entry point. It detaches `fd` from `current->files->fdt`, invokes `filp_flush` (which calls `f_op->flush`, drops dnotify watches, releases POSIX record locks owned by `current->files`), and then decrements the underlying `struct file` refcount — synchronously via `fput_close_sync` (no delayed work). The fd slot becomes available for reuse immediately on syscall entry, even if flush returns an error; for that reason errors from `close(2)` carry **flushed semantics only** — they do not unwind the close, and userspace MUST NOT retry `close(fd)` (the fd is already gone, and on a subsequent retry might refer to a freshly opened file in a different thread). Critical errno values are normalized — `ERESTART*` are folded to `-EINTR`.

Critical for: every libc `fclose`, every shell pipeline teardown, every Rookery resource finalizer, every container shutdown, every PID-1 reaping path.

### Acceptance Criteria

- [ ] AC-1: `close(invalid_fd) == -EBADF`.
- [ ] AC-2: `close(fd)` removes the fd from `current->files->fdt` before flush runs.
- [ ] AC-3: Subsequent `read/write/...` on the closed fd returns `-EBADF`.
- [ ] AC-4: Concurrent `close(fd)` from two threads: one returns `0`, the other `-EBADF`.
- [ ] AC-5: `f_op->flush` returning `-ERESTARTSYS` ⟹ syscall returns `-EINTR`, fd still released.
- [ ] AC-6: `close` of `O_PATH` fd does not invoke `dnotify_flush` or `locks_remove_posix`.
- [ ] AC-7: `close` releases all POSIX record locks held by `current->files` on the closed file.
- [ ] AC-8: Last-reference close destroys the `struct file` synchronously inside the syscall (no deferred work).
- [ ] AC-9: After close, the fd number is eligible for immediate reuse by `open`/`dup`.

### Architecture

```
struct CloseArgs { fd: u32 }
```

`sys_close(args) -> i32`:

1. `let file = file_close_fd(args.fd)?;` — returns `Option<Arc<File>>`; `None` ⟹ return `-EBADF`.
2. `let owner = current.files;`
3. `let mut retval = filp_flush(&file, owner);`
4. `fput_close_sync(file);` — synchronous final put.
5. If `retval == 0` ⟹ return `0`.
6. Normalize `ERESTART*` ⟹ `-EINTR`.
7. Return `retval`.

`Fs::file_close_fd(fd) -> Option<Arc<File>>`:

1. `spin_lock(&files->file_lock);`
2. `let fdt = files_fdtable(files);`
3. If `fd >= fdt.max_fds || fdt.fd[fd].is_none()` ⟹ unlock, return `None`.
4. `let file = mem::take(&mut fdt.fd[fd]);`
5. `__clear_close_on_exec(fd, fdt);`
6. `__put_unused_fd(files, fd);`  // marks fd reusable
7. `spin_unlock(&files->file_lock);`
8. Return `Some(file)`.

`Fs::filp_flush(file, owner) -> i32`:

1. If `file_count(file) == 0` ⟹ `CHECK_DATA_CORRUPTION; return 0;`.
2. `let retval = if let Some(flush) = file.f_op.flush { flush(file, owner) } else { 0 };`
3. If `!(file.f_mode & FMODE_PATH)`:
   - `dnotify_flush(file, owner);`
   - `locks_remove_posix(file, owner);`
4. Return `retval`.

### Out of Scope

- `close_range(2)` (separate Tier-5 if expanded).
- Deferred `fput` (`task_work`) used by other callers — this syscall uses synchronous `fput_close_sync`.
- `epoll`/`io_uring` fd-cleanup interactions (covered in respective Tier-3 docs).
- Implementation code.

### signature

C (POSIX / man-pages):

```c
int close(int fd);
```

glibc wrapper: `__libc_close` → `INLINE_SYSCALL(close, 1, fd)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE1(close, unsigned int, fd);
```

Rookery dispatch:

```rust
pub fn sys_close(fd: u32) -> SyscallResult<i32>;
```

### parameters

| name | type           | constraints                                        | errno-on-bad |
|------|----------------|----------------------------------------------------|--------------|
| fd   | `unsigned int` | must currently be open in `current->files->fdt`    | `EBADF`      |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno. **Fd is released regardless.**

### errors

| errno     | condition                                                                          |
|-----------|------------------------------------------------------------------------------------|
| `EBADF`   | `fd` is not currently open in the calling thread's file-descriptor table.          |
| `EINTR`   | A signal interrupted `f_op->flush` (or the flush returned `ERESTART*` which is normalized to `EINTR` since the fd cannot be restored). |
| `EIO`     | Lower-level flush returned an I/O error (e.g., dirty-page writeback flush failed for `O_SYNC` or NFS close-to-open coherence). |
| `ENOSPC`  | Flush flushed pending writes that hit a full filesystem.                           |
| `EDQUOT`  | Same, but quota-driven.                                                            |

Note: errors other than `EBADF` indicate the *flush* of buffered/cached state failed; the fd itself is **always** released.

### abi surface (constants + flags)

`close(2)` has no flags. The fd slot is the only ABI element. Related kernel symbols:

- `struct files_struct` — per-thread-group fd table.
- `struct fdtable` — RCU-protected fd array.
- `current->files` — owner for record locks released by `filp_close`.
- `FMODE_PATH` — `O_PATH`-opened files skip dnotify/POSIX-lock cleanup.
- `f_op->flush` — fs-supplied flush callback (NFS, FUSE, ceph, etc.).
- `ERESTART_RESTARTBLOCK` / `ERESTARTSYS` / `ERESTARTNOHAND` / `ERESTARTNOINTR` — all collapse to `-EINTR` post-close.

### compatibility contract

- REQ-1: Argument lowering: `%rdi = fd`. Return in `%rax`.
- REQ-2: `file_close_fd(fd)` atomically (under `files->file_lock`) removes the entry from `current->files->fdt` and returns the previously installed `struct file *`, or `NULL`.
- REQ-3: If `file_close_fd` returns `NULL` ⟹ return `-EBADF`. The fd slot is unchanged (it was already empty).
- REQ-4: Otherwise the fd is released **before** `filp_flush` runs; a concurrent thread re-opening the same fd number will see the slot reused.
- REQ-5: `filp_flush(file, owner)`:
  1. `CHECK_DATA_CORRUPTION(file_count(file) == 0)` — kernel `WARN` (treats as already-freed).
  2. If `f_op->flush` ⟹ call it; capture return.
  3. If `!(file->f_mode & FMODE_PATH)` ⟹ `dnotify_flush` + `locks_remove_posix(file, owner)`.
- REQ-6: After flush, `fput_close_sync(file)` performs an immediate (non-deferred) reference drop; if last ref, the file is destroyed in syscall context (no `task_work_add`).
- REQ-7: Restart-style errnos are normalized: `ERESTARTSYS|ERESTARTNOINTR|ERESTARTNOHAND|ERESTART_RESTARTBLOCK ⟹ EINTR`. Rationale: the fd is gone, the syscall **cannot** be restarted.
- REQ-8: `close(2)` of an `O_PATH` fd skips dnotify and POSIX-lock cleanup (those subsystems were never engaged).
- REQ-9: `close(2)` may legitimately return success while data has not yet hit stable storage (for non-`O_SYNC` regular files); userspace MUST call `fsync`/`fdatasync` for durability.
- REQ-10: Userspace MUST NOT retry `close(fd)` after any error — fd is gone.
- REQ-11: Concurrent `close(fd)` from sibling threads: at most one returns `0`, the others return `-EBADF`.
- REQ-12: `close(2)` releases the file slot atomically with respect to `dup`/`dup2`/`fcntl(F_DUPFD)`/`open` racing for the same slot.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_released_before_flush` | INVARIANT | `file_close_fd` returns before `filp_flush` runs. |
| `errno_normalized` | INVARIANT | `ERESTART*` never escapes to userspace; folded to `EINTR`. |
| `fd_reuse_safe` | INVARIANT | After close, fd slot in `fdt` is `None` under `file_lock`. |
| `o_path_skips_dnotify` | INVARIANT | `FMODE_PATH ⟹ no dnotify_flush, no locks_remove_posix`. |
| `concurrent_close_safe` | INVARIANT | At most one thread observes `Some(file)` from `file_close_fd`. |

### Layer 2: TLA+

`uapi/syscalls/close.tla`:
- Per-call → file_close_fd → filp_flush → fput_close_sync.
- Properties:
  - `safety_concurrent_close_unique_winner`,
  - `safety_fd_unconditionally_released`,
  - `liveness_fput_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `fdt.fd[fd] is None` for closed fd | `Fs::file_close_fd` |
| Post: caller's POSIX locks on file are released | `Fs::filp_flush` |
| Post: `ERESTART* ⟹ EINTR` | `Fs::sys_close` |

### Layer 4: Verus/Creusot functional

`close(fd) ≡ POSIX.1-2024 close` semantic equivalence, with Linux-specific fd-release-first semantics documented in `man 2 close` NOTES.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`close(2)` reinforcement:

- **Per-fd-release-first** — defense against `close` then `open` race where caller assumes fd is still valid.
- **Per-`CHECK_DATA_CORRUPTION` on `file_count == 0`** — defense against double-free of `struct file`.
- **Per-`locks_remove_posix(file, current->files)`** — defense against lock leak after fd close.
- **Per-`dnotify_flush`** — defense against zombie dnotify watch.
- **Per-`ERESTART*` normalization** — defense against userspace looping on a phantom fd.
- **Per-`fput_close_sync`** — defense against deferred-work resource leak.
- **Per-`O_PATH` skip-flush** — defense against unnecessary dnotify/lock-cleanup on path-only fd.
- **Per-`spinlock files->file_lock`** — defense against TOCTOU during fd-table mutation.

### grsecurity/pax-style reinforcement

- **PAX_REFCOUNT** — `file->f_count` saturating; underflow at last fput ⟹ `BUG()`.
- **GRKERNSEC_HIDESYM** — flush-error printks redact kernel pointers.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/fd` cleanup visible only to owner.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd close behaves normally; cannot un-chroot.
- **PAX_MEMORY_SANITIZE** — freed `struct file` slab is zeroed.
- **PAX_RANDKSTACK** — kstack offset randomized at close syscall entry.
- **GRKERNSEC_DMESG** — flush-error dmesg lines CAP_SYSLOG-gated.
- **GRKERNSEC_AUDIT_CHDIR** — chdir/close on chroot boundary auditable.

