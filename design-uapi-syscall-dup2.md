---
title: "Tier-5: syscall 33 — dup2(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`dup2(2)` is **x86_64 syscall 33**, the explicit-newfd file-descriptor duplication primitive. It is the targeted variant of `dup(2)`: the caller supplies the desired `newfd`, and the kernel installs the duplicate at exactly that slot, atomically closing whatever file `newfd` previously referred to (if any). The defining quirk: if `oldfd == newfd` AND `oldfd` is valid, `dup2` is a **no-op** that returns `newfd` — unlike `dup3` which returns `-EINVAL` in that case. This silent-success behavior is the classic POSIX shape but is also the historical source of countless bugs where callers expected `dup2(oldfd, oldfd)` to imply "make this fd's CLOEXEC fresh" or similar; the call simply returns the fd untouched. The new fd's `FD_CLOEXEC` is **cleared** (POSIX-required), matching `dup(2)`'s semantics; to set CLOEXEC atomically use `dup3(... O_CLOEXEC)`.

Critical for: shell `>&` redirection (`exec 2>&1` lowers to `dup2(1, 2)`), every `popen(3)`'s child-side fd plumbing, every `fork`+`exec` init pattern, every Rookery service-runner's stdio redirection, every `pty(7)` master/slave handoff.

### Acceptance Criteria

- [ ] AC-1: `dup2(oldfd, oldfd) == oldfd` when oldfd is valid (no-op).
- [ ] AC-2: `dup2(closed_fd, newfd) == -EBADF`.
- [ ] AC-3: `dup2(oldfd, RLIMIT_NOFILE) == -EBADF`.
- [ ] AC-4: After `dup2(oldfd, newfd)`, `read(newfd)` shares `f_pos` with `oldfd`.
- [ ] AC-5: If `newfd` was open prior, its `struct file` is closed (flushed) before return.
- [ ] AC-6: `fcntl(newfd, F_GETFD) & FD_CLOEXEC == 0` after `dup2`.
- [ ] AC-7: `dup2(oldfd, very_large_in_range_fd)` expands fdtable if needed.
- [ ] AC-8: `dup2(oldfd, oldfd)` does NOT close `newfd` (the no-op semantic distinguishes from `dup3`).
- [ ] AC-9: `dup2(oldfd, oldfd)` when `oldfd` invalid ⟹ `-EBADF` (short-circuit requires valid oldfd).

### Architecture

```
struct Dup2Args { oldfd: u32, newfd: u32 }
```

`sys_dup2(args) -> i32`:

1. `let f = fdget_raw(args.oldfd).ok_or(EBADF)?;`
2. If `args.oldfd == args.newfd`:
   - `fput(f);`
   - Return `args.newfd as i32`.
3. `fput(f);` — drop temporary; the install path will re-fdget.
4. Return `Fs::do_dup2(args.oldfd, args.newfd)`.

`Fs::do_dup2(oldfd, newfd) -> i32`:

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
9. `__clear_close_on_exec(newfd, fdt);` — CLOEXEC explicitly cleared.
10. `spin_unlock(&files->file_lock);`
11. If `prev.is_some()` ⟹ `filp_close(prev, files);`
12. `fput(f);`
13. Return `newfd as i32`.

### Out of Scope

- `dup(2)` — covered in `dup.md`.
- `dup3(2)` — covered in `dup3.md` (strict modern form with `O_CLOEXEC`).
- `fcntl(F_DUPFD)` / `F_DUPFD_CLOEXEC` (covered in `fcntl.md` Tier-5 if expanded).
- `close_range(2)` mass-fd manipulation.
- Implementation code.

### signature

C (POSIX-1.2008):

```c
int dup2(int oldfd, int newfd);
```

glibc wrapper: `__dup2` → `INLINE_SYSCALL(dup2, 2, oldfd, newfd)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(dup2, unsigned int, oldfd, unsigned int, newfd);
```

Rookery dispatch:

```rust
pub fn sys_dup2(oldfd: u32, newfd: u32) -> SyscallResult<i32>;
```

### parameters

| name   | type           | constraints                                                  | errno-on-bad |
|--------|----------------|--------------------------------------------------------------|--------------|
| oldfd  | `unsigned int` | open in `current->files->fdt`                                | `EBADF` |
| newfd  | `unsigned int` | within `[0, RLIMIT_NOFILE)`                                  | `EBADF` |

### return value

- Success: `newfd` (echoed back as positive integer).
- Failure: `< 0` — negated errno.

### errors

| errno     | condition                                                                  |
|-----------|----------------------------------------------------------------------------|
| `EBADF`   | `oldfd` is not open; or `newfd` is outside the allowed fd range.           |
| `EBUSY`   | (Linux-rare) Race with `dup`/`close` on the same `newfd` slot; internally retried — should not escape. |
| `EINTR`   | `dup2` interrupted by a signal during the implicit `close` of previous `newfd` (uncommon). |
| `ENOMEM`  | Allocating an expanded `fdtable` failed.                                   |

### abi surface (constants + flags)

`dup2(2)` is flagless. Internally implemented as `ksys_dup3(oldfd, newfd, 0)` with a `oldfd == newfd` short-circuit that bypasses the install path entirely.

Related kernel symbols:

- `struct files_struct` — per-thread-group fd table.
- `struct fdtable` — RCU-protected fd array; `close_on_exec` bitmap; `open_fds` bitmap.
- `__close_fd_get_file(newfd)` — atomically remove `newfd` from the fd table, returning its old `struct file`, so the caller can `fput`/`filp_close` it after installing the new file.
- `__alloc_fd_with_target(files, newfd, flags)` — install `oldfd`'s `struct file` into the `newfd` slot.
- `expand_files(files, newfd)` — grow fdtable if `newfd >= fdt.max_fds`.
- `RLIMIT_NOFILE` — soft/hard file-descriptor ceiling.
- `filp_close(file, files)` — synchronous flush + fput used when displacing the previous `newfd`.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=oldfd (u32)`, `%rsi=newfd (u32)`.
- REQ-2: Validate `oldfd`: `let f = fdget_raw(oldfd).ok_or(EBADF)?;`
- REQ-3: `oldfd == newfd` short-circuit:
  - Since we already validated `oldfd` via `fdget_raw`, the slot is occupied.
  - `fput(f);` return `newfd`. — **NO mutation; no close; no CLOEXEC change**.
- REQ-4: `newfd >= rlimit(RLIMIT_NOFILE) ⟹ -EBADF` (note: NOT `EMFILE`, matching historical POSIX).
- REQ-5: Under `files->file_lock`:
  - Expand fdtable if `newfd >= fdt.max_fds` ⟹ `-ENOMEM` on failure.
  - `let prev = mem::take(&mut fdt.fd[newfd]);` — saved for later `filp_close`.
  - `fdt.fd[newfd] = f;`
  - `__set_open_fd(newfd, fdt); __clear_close_on_exec(newfd, fdt);` — CLOEXEC explicitly cleared.
- REQ-6: Drop `file_lock`.
- REQ-7: If `prev.is_some()` ⟹ `filp_close(prev, current.files);` — flush + fput synchronously.
- REQ-8: `fput(f)` to balance `fdget_raw`'s ref (the slot now holds a fresh ref).
- REQ-9: On success, return `newfd`.
- REQ-10: `dup2` does NOT propagate `oldfd`'s CLOEXEC; the new fd's CLOEXEC is unconditionally cleared.
- REQ-11: `dup2` shares the underlying `struct file`, so `f_pos`, `f_flags`, `f_mode`, etc., are shared with `oldfd`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `oldfd_eq_newfd_noop` | INVARIANT | `oldfd == newfd` AND `oldfd` valid ⟹ returns `newfd`, no mutation. |
| `oldfd_eq_newfd_invalid` | INVARIANT | `oldfd == newfd` AND `oldfd` invalid ⟹ `-EBADF`. |
| `prev_closed_atomically` | INVARIANT | Previous `newfd` file flushed before return. |
| `cloexec_cleared` | INVARIANT | `close_on_exec[newfd] == 0` after install. |
| `rlimit_enforced` | INVARIANT | `newfd >= RLIMIT_NOFILE ⟹ -EBADF`. |

### Layer 2: TLA+

`uapi/syscalls/dup2.tla`:
- Per-call → validate(oldfd) → branch(oldfd==newfd) → install(newfd) → close(prev) → fput.
- Properties:
  - `safety_atomic_install_close`,
  - `safety_oldfd_eq_newfd_noop`,
  - `safety_cloexec_cleared`,
  - `liveness_dup2_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): `fdt.fd[newfd] == fdt.fd[oldfd]` (same `struct file *`) | `Fs::do_dup2` |
| Post (success): `close_on_exec[newfd] == 0` | `Fs::do_dup2` |
| Post (success): previous occupant of `newfd` has been `filp_close`d | `Fs::do_dup2` |
| Post (no-op): when `oldfd == newfd`, no state change | `sys_dup2` |

### Layer 4: Verus/Creusot functional

`dup2(oldfd, newfd)` ≡ `dup3(oldfd, newfd, 0)` when `oldfd != newfd`; no-op return when `oldfd == newfd` (deviates from `dup3`'s `-EINVAL`), per POSIX-1.2008.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`dup2(2)` reinforcement:

- **Per-`oldfd == newfd` no-op semantics** — POSIX-required, but caller MUST treat this as a potential bug-masking path (use `dup3` for strict diagnostic).
- **Per-`filp_close(prev)` synchronous** — defense against leaked dnotify watches / POSIX locks on displaced fd.
- **Per-`__clear_close_on_exec` under `file_lock`** — defense against exec-race leaking child-visible fd.
- **Per-`RLIMIT_NOFILE` enforcement** — defense against fdtable-exhaustion DoS.
- **Per-`fdget_raw`/`fput` discipline** — defense against `oldfd` fd-reuse race during install.

### grsecurity/pax-style reinforcement

- **fd-cap-inheritance on dup2** — capability rights attached to `oldfd` (per-mount, per-namespace restrictions) carry to `newfd` unchanged. The displaced previous `newfd`'s capabilities are dropped on `filp_close`.
- **CLOEXEC propagation** — `dup2(2)` clears CLOEXEC unconditionally (POSIX-required). Callers expecting CLOEXEC carryover from `oldfd` are buggy; use `dup3(... O_CLOEXEC)` for explicit control. Rookery's optional `fs.dup_cloexec_default` sysctl can flip the default for defense-in-depth.
- **PAX_REFCOUNT** — saturating `struct file` refcount across dup2 install + previous-fd-close.
- **GRKERNSEC_HIDESYM** — dup2-error dmesg redacts kernel pointers.
- **PAX_MEMORY_SANITIZE** — displaced `struct file` slab zeroed on free.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/fd` reflects dup2 install only to owner.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd process cannot escape via dup2 of a pre-chroot fd into a child.
- **GRKERNSEC_FORKFAIL** — dup2 expansion failure logged for fork-bomb / fd-exhaustion detection.
- **PAX_RANDKSTACK** — kstack offset randomized at dup2 syscall entry.
- **GRKERNSEC_DMESG** — dup2-error dmesg lines CAP_SYSLOG-gated.
- **GRKERNSEC_AUDIT_PTRACE** — dup2 by ptracer onto tracee fd table auditable.

