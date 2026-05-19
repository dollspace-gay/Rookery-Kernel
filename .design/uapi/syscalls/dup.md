# Tier-5: syscall 32 — dup(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`32  common  dup  sys_dup`)
  - fs/file.c (`SYSCALL_DEFINE1(dup, ...)`, `f_dupfd`, `alloc_fd`)
  - include/uapi/asm-generic/fcntl.h
-->

## Summary

`dup(2)` is **x86_64 syscall 32**, the classic Unix file-descriptor duplication primitive. It allocates the **lowest-numbered** unused file descriptor in the caller's fd table and points it at the same `struct file` as `oldfd`, incrementing the file's refcount. The new fd shares all `struct file`-level state with the original: file offset (`f_pos`), open-file flags (`f_flags` — `O_APPEND`, `O_NONBLOCK`, `O_DIRECT`, ...), file-position lock, owner, etc. Per-fd state — specifically the close-on-exec flag (`FD_CLOEXEC`) — is **cleared** on the new fd. This is the original POSIX behavior and explicitly differs from `dup3(... O_CLOEXEC)` which can opt in. The "lowest-unused-fd" guarantee is a millennia-old Unix invariant exploited by shell redirection (`0<&-; cmd` produces stdin = newly-allocated lowest fd).

Critical for: shell I/O redirection (the `>&` operator's reduce-to-fd-3 hack), every fork-then-redirect plumbing in init systems, every `popen(3)`/`pipe(2)`+`fork(2)` pattern, every Rookery service-runner's child fd plumbing, every `fexecve(3)`-style binary opener.

## Signature

C (POSIX-1.2008):

```c
int dup(int oldfd);
```

glibc wrapper: `__dup` → `INLINE_SYSCALL(dup, 1, oldfd)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE1(dup, unsigned int, fildes);
```

Rookery dispatch:

```rust
pub fn sys_dup(oldfd: u32) -> SyscallResult<i32>;
```

## Parameters

| name  | type           | constraints                            | errno-on-bad |
|-------|----------------|----------------------------------------|--------------|
| oldfd | `unsigned int` | open in `current->files->fdt`         | `EBADF` |

## Return value

- Success: the new fd (a positive integer, the lowest unused).
- Failure: `< 0` — negated errno.

## Errors

| errno     | condition                                                                |
|-----------|--------------------------------------------------------------------------|
| `EBADF`   | `oldfd` is not an open fd.                                              |
| `EMFILE`  | The process has reached `RLIMIT_NOFILE` (no fd available).              |
| `ENOMEM`  | Allocating an expanded `fdtable` failed.                                |

## ABI surface (constants + flags)

`dup(2)` is flagless. Internally calls `f_dupfd(0, file, 0)` to allocate the lowest-unused fd starting from 0.

Related kernel symbols:

- `struct files_struct` — per-thread-group fd table.
- `struct fdtable` — RCU-protected fd array; `close_on_exec` bitmap; `open_fds` bitmap; `full_fds_bits` accelerator.
- `f_dupfd(from, file, flags)` — allocate fd ≥ `from`, install file, set/clear CLOEXEC per `flags`.
- `alloc_fd(start, end, flags)` — bitmap scan helper.
- `__fd_install(files, fd, file)` — install into table under `file_lock`.
- `RLIMIT_NOFILE` — soft/hard file-descriptor ceiling.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=fildes (u32)`.
- REQ-2: `let file = fdget_raw(oldfd).ok_or(EBADF)?;` — increments file refcount.
- REQ-3: `let ret = f_dupfd(0, file, 0);` — bitmap-scan for lowest unused fd ≥ 0:
  - Scan `open_fds` bitmap from index 0 to first 0-bit; or expand fdtable if necessary.
  - On expansion failure ⟹ `-ENOMEM`.
  - On bitmap miss past `rlimit(RLIMIT_NOFILE)` ⟹ `-EMFILE`.
- REQ-4: Install file into `fdt.fd[newfd]` under `files->file_lock`.
- REQ-5: Set `open_fds[newfd] = 1; close_on_exec[newfd] = 0;` (CLOEXEC explicitly cleared — POSIX-required).
- REQ-6: Drop `file_lock`.
- REQ-7: Return `newfd`.
- REQ-8: The new fd shares the underlying `struct file` ⟹ `f_pos`, `f_flags`, lock owner, dnotify watches, owner pid, sigown ALL shared.
- REQ-9: The new fd's `FD_CLOEXEC` is per-fd state, NOT shared with `oldfd`.
- REQ-10: On error, `fput(file)` to balance `fdget_raw`; fd table unchanged.

## Acceptance Criteria

- [ ] AC-1: `dup(closed_fd) == -EBADF`.
- [ ] AC-2: `dup(0)` when fd 3, 4, 5 are in-use and 1 is unused ⟹ returns 1 (lowest unused).
- [ ] AC-3: After `dup`, `read(newfd)` shares `f_pos` with `oldfd`.
- [ ] AC-4: `fcntl(newfd, F_GETFD) & FD_CLOEXEC == 0`, regardless of `oldfd`'s CLOEXEC.
- [ ] AC-5: `dup` when fdtable full and at `RLIMIT_NOFILE` ⟹ `-EMFILE`.
- [ ] AC-6: `close(oldfd)` after `dup` ⟹ underlying file still accessible via newfd.
- [ ] AC-7: Two threads racing `dup` from same `files_struct` get distinct `newfd`s.
- [ ] AC-8: `fcntl(newfd, F_GETFL) == fcntl(oldfd, F_GETFL)` — file-level flags shared.
- [ ] AC-9: `dup` succeeds when fdtable must be expanded but `RLIMIT_NOFILE` permits it.

## Architecture

```
struct DupArgs { oldfd: u32 }
```

`sys_dup(args) -> i32`:

1. `let file = fdget_raw(args.oldfd).ok_or(EBADF)?;`
2. `let ret = Fs::f_dupfd(0, &file, 0);`
3. If `ret < 0`: `fput(file); return ret;`
4. Return `ret`.

`Fs::f_dupfd(from, file, flags) -> i32`:

1. `let newfd = alloc_fd(from, rlimit(RLIMIT_NOFILE), flags)?;` — `-EMFILE` or `-ENOMEM`.
2. `spin_lock(&files->file_lock);`
3. `let fdt = files_fdtable(files);`
4. `fdt.fd[newfd] = file.clone();` — refcount increment.
5. `__set_open_fd(newfd, fdt);`
6. `__clear_close_on_exec(newfd, fdt);` — CLOEXEC explicitly cleared.
7. `spin_unlock(&files->file_lock);`
8. Return `newfd as i32`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `oldfd_validated` | INVARIANT | `oldfd` validated via `fdget_raw`; `EBADF` if absent. |
| `lowest_unused_selected` | INVARIANT | `newfd` is the lowest index where `open_fds[i] == 0`. |
| `cloexec_cleared` | INVARIANT | `close_on_exec[newfd] == 0` after install. |
| `file_refcount_balanced` | INVARIANT | On error: `fput(file)` called; on success: file's refcount increased by exactly 1. |
| `rlimit_enforced` | INVARIANT | `newfd < rlimit(RLIMIT_NOFILE)`. |

### Layer 2: TLA+

`uapi/syscalls/dup.tla`:
- Per-call → fdget_raw(oldfd) → alloc_fd → install → set_open_fd → clear_cloexec.
- Properties:
  - `safety_lowest_unused_fd`,
  - `safety_cloexec_cleared`,
  - `safety_refcount_balanced`,
  - `liveness_dup_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): `fdt.fd[newfd] == fdt.fd[oldfd]` (same `struct file *`) | `Fs::f_dupfd` |
| Post (success): `close_on_exec[newfd] == 0` | `Fs::f_dupfd` |
| Post (success): `open_fds[newfd] == 1` | `Fs::f_dupfd` |
| Post (error): fdtable unchanged | `Fs::f_dupfd` |

### Layer 4: Verus/Creusot functional

`dup(oldfd)` ≡ `fcntl(oldfd, F_DUPFD, 0)` per POSIX-1.2008.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`dup(2)` reinforcement:

- **Per-`RLIMIT_NOFILE` enforcement** — defense against fdtable-exhaustion DoS.
- **Per-`fdget_raw` ref discipline** — defense against fd-reuse race during install.
- **Per-`file_lock` install** — defense against torn fdtable observation by another thread.
- **Per-CLOEXEC-cleared semantics** — POSIX-required, prevents leaking exec-suppression across dup chains accidentally.
- **Per-`alloc_fd` bitmap scan** — defense against duplicate-allocation race (atomic via `open_fds` bitmap test-and-set).

## Grsecurity/PaX-style Reinforcement

- **fd-cap-inheritance** — when an fd carries capability rights (e.g., per-mount or per-namespace restricted), `dup(2)` propagates those rights unchanged to the new fd. The new fd cannot escalate beyond the source.
- **CLOEXEC propagation** — `dup(2)` explicitly clears CLOEXEC; this is POSIX-required, but grsec-style policies recommend that callers immediately follow `dup` with `fcntl(F_SETFD, FD_CLOEXEC)` to avoid exec-leak. The kernel does not auto-set CLOEXEC on dup, but Rookery exposes an opt-in sysctl `fs.dup_cloexec_default` to flip this for defense-in-depth (default off for POSIX compatibility).
- **PAX_REFCOUNT** — saturating `struct file` refcount across dup install; prevents wrap-to-UAF.
- **GRKERNSEC_HIDESYM** — dup-error dmesg redacts kernel pointers.
- **PAX_MEMORY_SANITIZE** — displaced `struct file` slab (if any) zeroed on free.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/fd` reflects dup install only to owner.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd process cannot escape via dup of a pre-chroot fd into a child.
- **GRKERNSEC_FORKFAIL** — `EMFILE` from dup is logged for fork-bomb / fd-exhaustion detection.
- **PAX_RANDKSTACK** — kstack offset randomized at dup syscall entry.
- **GRKERNSEC_DMESG** — dup-error dmesg lines CAP_SYSLOG-gated.
- **GRKERNSEC_AUDIT_PTRACE** — dup by ptracer onto tracee fd table auditable.
- **GRKERNSEC_BRUTE** — repeated `dup` failures from the same task contribute to brute-force-detection heuristics that may apply a per-task slow-down on subsequent fd-allocation attempts.
- **PAX_USERCOPY** — although `dup` does not copy from/to user memory, its tightening of the file refcount path is checked by PAX_USERCOPY-adjacent slab-bound assertions when fdtables are expanded.
- **Per-namespace dup denial** — when a per-fd `mnt_namespace` capability is restricted, dup cannot widen access; rookery's namespace-aware fd-cap layer intersects the source fd's rights with the namespace policy before install.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `dup2(2)` — covered in `dup2.md`.
- `dup3(2)` — covered in `dup3.md`.
- `fcntl(F_DUPFD)` / `F_DUPFD_CLOEXEC` (covered in `fcntl.md` Tier-5 if expanded).
- `close_range(2)` mass-fd manipulation.
- Implementation code.
