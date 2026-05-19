# Tier-5: syscall 292 — dup3(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`292  common  dup3  sys_dup3`)
  - fs/file.c (`SYSCALL_DEFINE3(dup3, ...)`, `ksys_dup3`, `do_dup2`, `__close_fd_get_file`, `__alloc_fd`)
  - include/uapi/asm-generic/fcntl.h (`O_CLOEXEC`)
-->

## Summary

`dup3(2)` is **x86_64 syscall 292**, the explicit-newfd duplication primitive. It is the strict variant of `dup2(2)`: the caller MUST supply a `newfd` distinct from `oldfd`, and MAY pass `O_CLOEXEC` in `flags` to atomically install the duplicate with close-on-exec set. If `newfd` already refers to an open file, that file is closed atomically before `oldfd`'s `struct file` reference is installed. `dup3(oldfd, oldfd, ...)` is rejected (`EINVAL`) — unlike `dup2`, which is a no-op in that case — so that `dup3` cannot accidentally hide a logic bug that `dup2` would silently mask.

Critical for: every shell `>&` redirection using `O_CLOEXEC`, every Rookery init/exec sequence that hands a controlled fd table to a child, every container-runtime stdio setup, every io_uring `IORING_REGISTER_FILES`-using process that needs an atomic close-on-exec dup.

## Signature

C (Linux-specific):

```c
int dup3(int oldfd, int newfd, int flags);
```

glibc wrapper: `__libc_dup3` → `INLINE_SYSCALL(dup3, 3, oldfd, newfd, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(dup3, unsigned int, oldfd, unsigned int, newfd, int, flags);
```

Rookery dispatch:

```rust
pub fn sys_dup3(oldfd: u32, newfd: u32, flags: i32) -> SyscallResult<i32>;
```

## Parameters

| name   | type            | constraints                                                  | errno-on-bad |
|--------|-----------------|--------------------------------------------------------------|--------------|
| oldfd  | `unsigned int`  | open in `current->files->fdt`                                | `EBADF` |
| newfd  | `unsigned int`  | within `[0, RLIMIT_NOFILE)`; MUST differ from `oldfd`        | `EBADF` / `EINVAL` |
| flags  | `int`           | subset of `{O_CLOEXEC}`; all other bits ⟹ `-EINVAL`         | `EINVAL` |

## Return value

- Success: `newfd` (echoed back as positive integer).
- Failure: `< 0` — negated errno.

## Errors

| errno    | condition                                                                  |
|----------|----------------------------------------------------------------------------|
| `EBADF`  | `oldfd` is not open.                                                       |
| `EINVAL` | `oldfd == newfd`; or `flags` contains bits other than `O_CLOEXEC`.         |
| `EBADF`  | `newfd >= RLIMIT_NOFILE` (treated as out-of-range fd).                     |
| `EMFILE` | `newfd` is within range but expanding `fdt` would exceed `RLIMIT_NOFILE` (rare; usually `EBADF`). |
| `EBUSY`  | (Linux-rare) Race with `dup`/`close` on the same `newfd` slot; auto-retried internally — should not escape. |
| `ENOMEM` | Allocating an expanded `fdtable` failed.                                   |

## ABI surface (constants + flags)

`flags` (`include/uapi/asm-generic/fcntl.h`):

- `O_CLOEXEC = 0x80000` — set close-on-exec on the new fd atomically. No other flag bit is accepted.

Related kernel symbols:

- `struct files_struct` — per-thread-group fd table.
- `struct fdtable` — RCU-protected fd array; `close_on_exec` bitmap; `open_fds` bitmap.
- `__close_fd_get_file(newfd)` — atomically remove `newfd` from the fd table, returning its old `struct file`, so the caller can `fput` it after installing the new file.
- `__alloc_fd_with_target(files, newfd, flags)` — install `oldfd`'s `struct file` into the `newfd` slot.
- `RLIMIT_NOFILE` — soft/hard file-descriptor ceiling.
- `fput_close_sync` — synchronous reference drop used when displacing the previous `newfd`.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=oldfd (u32)`, `%rsi=newfd (u32)`, `%rdx=flags (i32)`.
- REQ-2: `flags & ~O_CLOEXEC ⟹ -EINVAL` before any fd lookup.
- REQ-3: `oldfd == newfd ⟹ -EINVAL` (this is the explicit divergence from `dup2`).
- REQ-4: `newfd >= rlimit(RLIMIT_NOFILE) ⟹ -EBADF` (consistent with `dup2`).
- REQ-5: `let f = fdget_raw(oldfd)?;` — fails with `-EBADF` if oldfd not open.
- REQ-6: Under `files->file_lock`:
  - Expand `fdtable` if `newfd >= fdt->max_fds` (may allocate; on failure ⟹ `-ENOMEM`).
  - `let prev = fdt->fd[newfd];` — saved for later `fput_close_sync`.
  - `fdt->fd[newfd] = f;`
  - Set `open_fds[newfd]`; clear `close_on_exec[newfd]`; if `O_CLOEXEC` ⟹ set `close_on_exec[newfd]`.
- REQ-7: Drop `file_lock`.
- REQ-8: If `prev.is_some()` ⟹ `filp_close(prev, current.files)` — flush + fput synchronously.
- REQ-9: `fput(f)` to balance `fdget_raw`'s ref (the slot now holds a fresh ref).
- REQ-10: On success, return `newfd`.
- REQ-11: `dup3` does NOT propagate `oldfd`'s close-on-exec; the new fd's CLOEXEC is purely controlled by `flags & O_CLOEXEC`.
- REQ-12: `dup3` shares the underlying `struct file`, so `f_pos`, `f_flags`, `f_mode`, lock owner, etc., are shared with `oldfd`.
- REQ-13: Errors leave the fd table unmodified — `newfd` retains its previous contents (or remains empty).

## Acceptance Criteria

- [ ] AC-1: `dup3(oldfd, oldfd, 0) == -EINVAL` (vs. `dup2` which returns `oldfd`).
- [ ] AC-2: `dup3(closed_fd, newfd, 0) == -EBADF`.
- [ ] AC-3: `dup3(oldfd, newfd, O_CLOEXEC)` sets `close_on_exec` on `newfd` atomically.
- [ ] AC-4: `dup3(oldfd, newfd, O_NONBLOCK) == -EINVAL`.
- [ ] AC-5: After `dup3`, `read(newfd)` shares `f_pos` with `oldfd`.
- [ ] AC-6: If `newfd` was open prior to `dup3`, its `struct file` is closed (flushed) before return.
- [ ] AC-7: `dup3(oldfd, RLIMIT_NOFILE, 0) == -EBADF`.
- [ ] AC-8: `dup3(oldfd, very_large_in_range_fd, 0)` expands the `fdtable` if needed.
- [ ] AC-9: `dup3(oldfd, newfd, 0)` clears `close_on_exec` on `newfd` (even if it was set previously).

## Architecture

```
struct Dup3Args { oldfd: u32, newfd: u32, flags: i32 }
```

`sys_dup3(args) -> i32`:

1. If `args.flags & !O_CLOEXEC` ⟹ return `-EINVAL`.
2. If `args.oldfd == args.newfd` ⟹ return `-EINVAL`.
3. Return `do_dup3(args.oldfd, args.newfd, args.flags)`.

`Fs::do_dup3(oldfd, newfd, flags) -> i32`:

1. If `newfd >= rlimit(RLIMIT_NOFILE)` ⟹ return `-EBADF`.
2. `let f = fdget_raw(oldfd).ok_or(EBADF)?;`
3. `spin_lock(&files->file_lock);`
4. `let fdt = files_fdtable(files);`
5. If `newfd >= fdt.max_fds`:
   - `expand_files(files, newfd)?;` — `-ENOMEM` if alloc fails.
   - Re-fetch `fdt`.
6. `let prev = mem::take(&mut fdt.fd[newfd]);`
7. `fdt.fd[newfd] = f.clone();` (refcount increment under lock)
8. `__set_open_fd(newfd, fdt);`
9. If `flags & O_CLOEXEC` ⟹ `__set_close_on_exec(newfd, fdt)` else `__clear_close_on_exec(newfd, fdt)`.
10. `spin_unlock(&files->file_lock);`
11. If `prev.is_some()` ⟹ `filp_close(prev, files);`
12. `fput(f);`
13. Return `newfd as i32`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `oldfd_eq_newfd_rejected` | INVARIANT | `oldfd == newfd ⟹ -EINVAL` before any side effects. |
| `flags_strict` | INVARIANT | Any bit outside `O_CLOEXEC` ⟹ `-EINVAL`. |
| `prev_closed_atomically` | INVARIANT | Previous `newfd` file is flushed/closed before return. |
| `cloexec_atomic` | INVARIANT | `close_on_exec[newfd]` set in the same critical section as fd install. |
| `no_partial_state_on_error` | INVARIANT | Errors leave fdtable bitmaps unchanged. |

### Layer 2: TLA+

`uapi/syscalls/dup3.tla`:
- Per-call → validate → fdget(oldfd) → install(newfd) → close(prev) → fput.
- Properties:
  - `safety_atomic_install_close`,
  - `safety_oldfd_neq_newfd`,
  - `liveness_dup3_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: `fdt.fd[newfd] == fdt.fd[oldfd]` (same `struct file *`) | `Fs::do_dup3` |
| Post: `close_on_exec[newfd] == (flags & O_CLOEXEC ? 1 : 0)` | `Fs::do_dup3` |
| Post: Previous occupant of `newfd` has been `filp_close`d | `Fs::do_dup3` |

### Layer 4: Verus/Creusot functional

`dup3(oldfd, newfd, flags)` ≡ Linux-specific dup3 semantics per `man 2 dup3` (POSIX has no dup3; closest is dup2).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`dup3(2)` reinforcement:

- **Per-`oldfd != newfd` enforcement** — defense against bug-masking that `dup2` allows.
- **Per-`O_CLOEXEC`-only flag mask** — defense against future flag-bit silent-accept.
- **Per-`filp_close(prev)` synchronous** — defense against leaked dnotify watches / POSIX locks on displaced fd.
- **Per-`__set_close_on_exec` under `file_lock`** — defense against exec-race leaking child-visible fd.
- **Per-`RLIMIT_NOFILE` enforcement** — defense against fdtable-exhaustion DoS.
- **Per-`fdget_raw`/`fput` discipline** — defense against `oldfd` fd-reuse race during install.

## Grsecurity/PaX-style Reinforcement

- **PAX_REFCOUNT** — `struct file` refcount saturating across dup3 install.
- **GRKERNSEC_HIDESYM** — dup3 error printks redact kernel pointers.
- **PAX_MEMORY_SANITIZE** — displaced `struct file` slab zeroed on free.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/fd` reflects dup3 install only to owner.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd process cannot escape via dup3 of a pre-chroot fd into a child.
- **GRKERNSEC_FORKFAIL** — dup3 expansion failure is logged for fork-bomb detection heuristics.
- **PAX_RANDKSTACK** — kstack offset randomized at dup3 syscall entry.
- **GRKERNSEC_DMESG** — dup3-error dmesg lines CAP_SYSLOG-gated.
- **GRKERNSEC_AUDIT_PTRACE** — dup3 by ptracer onto tracee's fd table auditable.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `dup(2)` and `dup2(2)` (separate Tier-5 docs if expanded; `dup3` is the strict modern form).
- `fcntl(F_DUPFD)` / `F_DUPFD_CLOEXEC` (covered in `fcntl.md` Tier-5 if expanded).
- `close_range(2)` mass-fd manipulation.
- Implementation code.
