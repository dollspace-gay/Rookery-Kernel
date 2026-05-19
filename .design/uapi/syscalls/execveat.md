# Tier-5 syscall: execveat(2) — syscall 322

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/exec.c (sys_execveat, do_execveat_common)
  - include/uapi/linux/fcntl.h (AT_EMPTY_PATH, AT_SYMLINK_NOFOLLOW, AT_FDCWD)
  - arch/x86/entry/syscalls/syscall_64.tbl (322  64  execveat)
-->

## Summary

`execveat(2)` is the dirfd-relative + flags-aware variant of `execve(2)`.
It allows callers to:
1. Resolve `path` relative to a directory file-descriptor (`dirfd`) rather
   than the cwd — fundamental for sandboxes and `*at`-family path safety.
2. Specify `AT_EMPTY_PATH` to exec the file referenced by `dirfd` itself —
   the only way to `execve` a memfd / O_PATH fd / non-pathname-bearing fd.
3. Specify `AT_SYMLINK_NOFOLLOW` to refuse to follow a final-component
   symlink — defense against TOCTOU symlink-swap attacks on the binary.

All credential, NNP, AT_SECURE, MAX_ARG_STRLEN/STRINGS, binfmt, and
ptrace-event semantics are IDENTICAL to `execve(2)`. The only ABI surface
unique to `execveat` is the `(dirfd, flags)` pair.

`execveat` is the syscall used by `fexecve(3)` (glibc), by container
runtimes that pre-open binaries before chroot (e.g. `runc init`), by
`memfd_create`-backed binaries (often used for in-memory exec, where
there is no path), and by sandbox brokers that hand a pre-vetted `O_PATH`
fd to the sandboxed process.

This Tier-5 covers `fs/exec.c::SYSCALL_DEFINE5(execveat, ...)` (~30 lines
of entry code; everything else is `do_execveat_common`, shared with
`execve(2)` — see `execve.md`).

## Signature

```c
long execveat(int dirfd,
              const char *path,
              char *const argv[],
              char *const envp[],
              int flags);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Directory fd for relative path resolution, or `AT_FDCWD = -100` for cwd-relative, or any fd combined with `AT_EMPTY_PATH`. |
| `path` | `const char *` | in | NUL-terminated pathname, OR `""` (empty) when `AT_EMPTY_PATH` is set. Absolute paths ignore `dirfd`. |
| `argv` | `char *const []` | in | NULL-terminated argv array (same as `execve`). |
| `envp` | `char *const []` | in | NULL-terminated envp array (same as `execve`). |
| `flags` | `int` | in | Bit-OR of `AT_EMPTY_PATH`, `AT_SYMLINK_NOFOLLOW`. Other bits MUST be zero. |

## Return value

| Value | Meaning |
|---|---|
| (does not return) | Success. |
| `-1` + `errno`    | Failure. |

## Errors

All errors of `execve(2)` apply, plus:

| errno | Trigger |
|---|---|
| `EBADF`  | `dirfd` is not a valid open fd (and `!AT_FDCWD`). |
| `EINVAL` | `flags` contains a bit other than `AT_EMPTY_PATH` or `AT_SYMLINK_NOFOLLOW`. |
| `EINVAL` | `AT_EMPTY_PATH` set but `path != ""`. (Strict mode; mainline accepts both but Rookery follows the documented contract.) |
| `ENOENT` | `path` empty and `!AT_EMPTY_PATH`. |
| `ENOTDIR`| `dirfd` is not a directory and `path` is relative. |
| `ELOOP`  | `AT_SYMLINK_NOFOLLOW` set AND final component is a symlink. |
| `EACCES` | `dirfd` is opened `O_PATH` but caller cannot search its directory (`F_GETFL` denies). |

## ABI surface

```text
__NR_execveat (x86_64)    = 322
__NR_execveat (i386)      = 358
__NR_execveat (generic)   = 281    (arm64/riscv/loongarch via asm-generic/unistd.h)
__NR_execveat (powerpc)   = 362
__NR_execveat (s390x)     = 354
__NR_execveat (sparc)     = 350
__NR_execveat (mips O32)  = 4356
__NR_execveat (mips N64)  = 5316

AT_FDCWD              = -100        /* dirfd sentinel: resolve relative to cwd */
AT_SYMLINK_NOFOLLOW   = 0x100       /* refuse symlink follow on final component */
AT_EMPTY_PATH         = 0x1000      /* exec the file referenced by dirfd if path is "" */
```

(The `AT_*` flag values are stable across all `*at(2)` syscalls and live
in `uapi/linux/fcntl.h`.)

## Compatibility contract

REQ-1: The syscall number is **322** on x86_64, **281** on generic-syscall
archs. ABI-stable forever.

REQ-2: Path resolution semantics:
- `path` is absolute (starts with `/`): `dirfd` ignored. Resolved from
  filesystem root.
- `path` is relative AND `dirfd == AT_FDCWD`: resolved relative to caller's
  cwd. Equivalent to `execve(2)`.
- `path` is relative AND `dirfd >= 0`: resolved relative to the directory
  referenced by `dirfd`. `dirfd` MUST be open and refer to a directory
  (or `O_PATH`-opened directory).
- `path == ""` AND `AT_EMPTY_PATH` set: the file referenced by `dirfd`
  is the binary. `dirfd` MUST be open. May be a regular file, a memfd, an
  `O_PATH`-opened file.
- `path == ""` AND `!AT_EMPTY_PATH`: `-ENOENT` (matches mainline 3.19+).

REQ-3: `flags` MUST be a bitwise-OR of `AT_EMPTY_PATH` and/or
`AT_SYMLINK_NOFOLLOW`. Any other bit set returns `-EINVAL`.

REQ-4: `AT_SYMLINK_NOFOLLOW`: if the final-component name in `path`
(after resolving intermediate components) is a symlink, return `-ELOOP`
(not `-EACCES`, not silent follow). Intermediate symlinks are still
followed normally — only the final component is gated.

REQ-5: When `AT_EMPTY_PATH` is used, there is no final-component name —
`AT_SYMLINK_NOFOLLOW` is meaningless in combination and SHOULD be omitted
(but is not an error to include).

REQ-6: `AT_EMPTY_PATH` AND `dirfd` opened `O_PATH`: permitted. The
`O_PATH` fd does not need read permission on the file; permission to
execute is checked at exec time independently.

REQ-7: `AT_EMPTY_PATH` AND `dirfd` opened `O_RDONLY` / `O_RDWR` etc.:
permitted. The fd's open-mode is irrelevant; execute permission is checked.

REQ-8: `AT_EMPTY_PATH` AND `dirfd` is a directory fd: `-EACCES` (cannot
execute a directory).

REQ-9: Special procfs symlink behavior: `/proc/self/fd/<n>` accessed via
`execveat` with `AT_EMPTY_PATH` is a common pattern for "exec the
already-open fd"; equivalent to passing `dirfd = n, path = ""`,
`AT_EMPTY_PATH`. The procfs path produces `/proc/<pid>/exe` symlink
pointing at the original file (or "memfd:<name>" for memfd).

REQ-10: `/proc/<pid>/exe` symlink contents:
- Binary loaded from named-path: target is the canonicalized path.
- Binary loaded via `AT_EMPTY_PATH` from regular file: target is path of
  the file at exec time.
- Binary loaded via `AT_EMPTY_PATH` from memfd: target is
  `/memfd:<name>(deleted)`.
- Binary loaded via `AT_EMPTY_PATH` from O_PATH fd: target is the
  recorded path at fd-open time.

REQ-11: `dirfd` is a regular file (NOT a directory) AND `path != ""`:
`-ENOTDIR`. Path-relative resolution requires a directory.

REQ-12: Permission checks:
- `dirfd` directory needs `MAY_EXEC` (search bit) for the caller.
- `path` traversal checks `MAY_EXEC` on each intermediate component.
- Final file needs `MAY_EXEC` for the caller's creds AND `MAY_READ` if the
  binfmt requires reading the file (almost always true; binfmt_misc with
  `O` flag may skip read).

REQ-13: All credential, NNP, AT_SECURE, MAX_ARG_STRLEN/STRINGS, binfmt,
ptrace-event, audit semantics are IDENTICAL to `execve(2)` (see
`execve.md`). The only entry-point divergence is the resolution path.

REQ-14: Audit: AUDIT_EXECVE record includes `dirfd`, `path`, and `flags`
in addition to the resolved final path.

REQ-15: When loaded via `AT_EMPTY_PATH` from a memfd that has F_SEAL_EXEC
NOT set (sealed against further write/grow), the binary content is
guaranteed immutable for the lifetime of the new process — preferred for
JIT-then-exec patterns.

## Acceptance Criteria

- [ ] AC-1: `execveat(AT_FDCWD, "/bin/true", argv, envp, 0)` ≡ `execve("/bin/true", argv, envp)`.
- [ ] AC-2: `execveat(dirfd, "true", argv, envp, 0)` where `dirfd` opens `/bin` succeeds.
- [ ] AC-3: `execveat(AT_FDCWD, "", argv, envp, AT_EMPTY_PATH)` → `-ENOENT` (no dirfd to exec).
- [ ] AC-4: `execveat(fd_of_memfd, "", argv, envp, AT_EMPTY_PATH)` succeeds; `/proc/self/exe` reads `/memfd:<name>`.
- [ ] AC-5: `execveat(opath_fd, "", argv, envp, AT_EMPTY_PATH)` succeeds (O_PATH fd permitted).
- [ ] AC-6: `execveat(dir_fd, "", argv, envp, AT_EMPTY_PATH)` → `-EACCES` (cannot exec directory).
- [ ] AC-7: `execveat(reg_file_fd, "x", argv, envp, 0)` → `-ENOTDIR`.
- [ ] AC-8: `execveat(AT_FDCWD, "symlink_to_binary", argv, envp, AT_SYMLINK_NOFOLLOW)` → `-ELOOP`.
- [ ] AC-9: `execveat(AT_FDCWD, "symlink_to_binary", argv, envp, 0)` follows symlink and succeeds.
- [ ] AC-10: `execveat(..., flags = 0x4000 /* unknown */)` → `-EINVAL`.
- [ ] AC-11: `execveat(bad_fd, "x", argv, envp, 0)` → `-EBADF`.
- [ ] AC-12: setuid bit on memfd-exec'd binary: euid upgrade NOT applied (memfd is on tmpfs which is `MS_NOSUID`-treated for setuid? — per kernel rule: memfd preserves setuid only if memfd mount allows it; Rookery treats memfd as `MS_NOSUID` by default).
- [ ] AC-13: ptrace `PTRACE_EVENT_EXEC` fired with new pc (same as execve).
- [ ] AC-14: AUDIT_EXECVE record includes `dirfd`, `flags`, resolved path.
- [ ] AC-15: x86_64 syscall number is 322; generic is 281.
- [ ] AC-16: `AT_EMPTY_PATH | AT_SYMLINK_NOFOLLOW` together: accepted (no-op for the latter when path is empty).

## Architecture

```rust
#[syscall(nr = 322, abi = "sysv")]
pub fn sys_execveat(
    dirfd: i32,
    path:  UserPtr<u8>,
    argv:  UserPtr<UserPtr<u8>>,
    envp:  UserPtr<UserPtr<u8>>,
    flags: i32,
) -> isize {
    // REQ-3: validate flags bitmask
    let valid_flags = AT_EMPTY_PATH | AT_SYMLINK_NOFOLLOW;
    if flags & !valid_flags != 0 { return -EINVAL; }

    Exec::do_execveat_common(dirfd, path, argv, envp, flags)
}
```

`Exec::do_execveat_common(dfd, path, argv, envp, flags) -> isize`:
1. /* Path resolution */
2. let lookup_flags = LOOKUP_FOLLOW
3.     | if flags & AT_SYMLINK_NOFOLLOW != 0 { 0 } else { LOOKUP_FOLLOW }
4.     | if flags & AT_EMPTY_PATH != 0 { LOOKUP_EMPTY } else { 0 };
5. let file = Vfs::do_open_execat(dfd, path, lookup_flags)?;
6. /* Permission */
7. Vfs::inode_permission(&file.inode(), MAY_EXEC)?;
8. /* Shared with execve(2): binfmt + creds + dispatch */
9. ... see execve.md architecture section ...

`Vfs::do_open_execat(dfd, path, lookup_flags)`:
1. if lookup_flags & LOOKUP_EMPTY && user_path_is_empty(path):
2.   /* AT_EMPTY_PATH branch: take the file from dfd directly */
3.   let f = fdget(dfd)?;
4.   if f.inode().is_dir() { return Err(-EACCES); }
5.   return Ok(f);
6. else:
7.   /* Standard relative-or-absolute path resolution */
8.   path_openat(dfd, path, lookup_flags | LOOKUP_EXEC)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_bitmask_sealed` | TOTAL | Any `flags` with unknown bit → `-EINVAL`. |
| `at_empty_path_requires_dirfd` | INVARIANT | `AT_EMPTY_PATH | path==""` ⟹ dfd is open file. |
| `nofollow_blocks_final_symlink` | INVARIANT | `AT_SYMLINK_NOFOLLOW` ⟹ final-component symlink rejected. |
| `dirfd_dir_required_for_relative` | INVARIANT | relative path ⟹ dfd is directory. |

### Layer 2: TLA+

`fs/execveat.tla`:
- States: VALIDATE_FLAGS → RESOLVE_PATH → OPEN_FILE → (SHARED_EXEC_PATH).
- Properties:
  - `safety_at_empty_path_or_resolve` — exactly one resolution path taken per call.
  - `safety_nofollow_terminal_symlink` — final symlink with AT_SYMLINK_NOFOLLOW ⟹ ELOOP.
  - `liveness_eventual_dispatch` — every accepted call enters shared exec path.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_execveat` post: bad flags ⟹ -EINVAL, no fd touched | `sys_execveat` |
| `do_open_execat` post: AT_EMPTY_PATH branch returns dfd's file | `Vfs::do_open_execat` |
| `lookup_flags` post: AT_SYMLINK_NOFOLLOW ⟹ ¬LOOKUP_FOLLOW on final | `Vfs::do_open_execat` |

### Layer 4: Verus / Creusot functional

`execveat(2)` man page semantic equivalence. LTP `execveat01..execveat03`,
`kselftests/exec/execveat*` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`execveat(2)` reinforcement (in addition to all `execve(2)` reinforcements):

- **Per-`AT_SYMLINK_NOFOLLOW` available** — defense against per-symlink-swap TOCTOU.
- **Per-dirfd resolution** — defense against per-cwd-race during sandbox setup.
- **Per-`AT_EMPTY_PATH` memfd-only ergonomic** — defense against per-stale-pathname load (memfd is anonymous).
- **Per-flags bitmask sealed** — defense against per-future-bit-leak.
- **Per-`AT_EMPTY_PATH` rejects directory fd** — defense against per-dir-fd exec.
- **Per-O_PATH fd permitted** — defense against per-secrecy-leak (no read permission needed to vet a binary).
- **Per-memfd default `MS_NOSUID`** — defense against per-memfd-setuid escalation.

## Grsecurity / PaX surface

- **PAX_RANDKSTACK at syscall entry** — same as `execve`.
- **PaX UDEREF on user buffers** — same as `execve`; additionally
  `dirfd` value (an integer) is validated via `fdget` against the
  caller's fd table under UDEREF.
- **PaX ASLR / NOEXEC / SEGMEXEC** — applied to the loaded PT_LOAD
  segments exactly as `execve`.
- **GRKERNSEC_HARDEN_PTRACE** — `PTRACE_EVENT_EXEC` honors yama+grsec
  policy; identical to `execve`.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/exe` for an
  AT_EMPTY_PATH-loaded binary leaks the original fd's recorded path; the
  per-uid hide-filter applies to this readback too.
- **GRKERNSEC_CHROOT** — inside chroot:
  - `dirfd` MUST be a fd inside the chroot subtree;
    `chroot_findtask`-style check enforces this on every `fdget`.
  - `AT_EMPTY_PATH` execve of a memfd is gated by
    `chroot_deny_anything-other-than-suid-binaries`; if the policy is
    strict, returns `-EACCES`.
  - All `execve` chroot restrictions (`chroot_deny_suid`,
    `chroot_caps`) apply identically.
- **GRKERNSEC_BRUTE** — execveat failures (e.g. `-ELOOP` from
  `AT_SYMLINK_NOFOLLOW`) increment the per-uid failure counter.
- **GRKERNSEC_SIGNALS spoof prevention** — same as `execve`.
- **Per-grsec `tpe` (trusted-path-execution)** — `AT_EMPTY_PATH` via
  memfd treated as untrusted-path → `-EACCES` unless caller is in
  TPE-trusted group; this CLOSES the "memfd-exec backdoor" used by
  in-memory droppers.
- **Per-grsec `gradm` learning** — `execveat` records `(dirfd-path,
  flags)` triple against RBAC.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `execve(2)` syscall (separate Tier-5 doc — most semantics live there).
- ELF / script / binfmt_misc loaders (Tier-3 `fs/binfmt_*.c`).
- Credential-recomputation internals (`kernel/cred.c` Tier-3).
- memfd_create semantics (Tier-5 `uapi/syscalls/memfd_create.md` if/when written).
- O_PATH semantics (Tier-5 `uapi/headers/fcntl.md` — already covered).
- Implementation code.
