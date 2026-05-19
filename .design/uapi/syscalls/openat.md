# Tier-5: syscall 257 — openat(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`257  common  openat  sys_openat`)
  - fs/open.c (`SYSCALL_DEFINE4(openat, ...)`, `do_sys_open`, `do_sys_openat2`, `build_open_how`, `build_open_flags`, `do_file_open`)
  - fs/namei.c (`path_openat`, `do_filp_open`, `path_init`, `nameidata.dfd`)
  - include/uapi/linux/fcntl.h (`AT_FDCWD = -100`, `AT_EMPTY_PATH`, `AT_NO_AUTOMOUNT`, etc.)
-->

## Summary

`openat(2)` is **x86_64 syscall 257**, the directory-relative open. It is identical to `open(2)` except:

- An additional first argument `dirfd` resolves relative `pathname` against either:
  - the open directory referred to by `dirfd` (must be `O_DIRECTORY`-opened or `O_PATH` on a directory), or
  - the calling process's current working directory if `dirfd == AT_FDCWD = -100`.
- Absolute `pathname` ignores `dirfd` entirely (but `dirfd` is still validated unless `pathname` is non-empty absolute).

The internal implementation reuses `do_sys_open(dirfd, ...)` exactly — `open(2)` is just `openat(AT_FDCWD, ...)` under the hood. `openat` enables race-free directory-relative file access patterns (every `*at` family member shares this design), e.g., `mkdirat` + `openat` with the same dirfd avoids `..`/symlink races between the two calls. With `AT_EMPTY_PATH`, an empty `pathname` re-opens `dirfd` itself (essentially `dup`-with-different-flags).

Critical for: every libc `openat`, every container-runtime sandbox setup, every Rookery `openat`-based VFS test, every `O_TMPFILE` creator targeting an open directory fd, every chroot-resistant path resolver.

## Signature

C (POSIX / man-pages):

```c
int openat(int dirfd, const char *pathname, int flags, ... /* mode_t mode */ );
```

glibc wrapper: `__libc_openat` → `INLINE_SYSCALL(openat, 4, dirfd, pathname, flags, mode)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(openat, int, dfd, const char __user *, filename, int, flags, umode_t, mode);
```

Rookery dispatch:

```rust
pub fn sys_openat(dirfd: i32, filename: UserPtr<u8>, flags: i32, mode: u16) -> SyscallResult<i32>;
```

## Parameters

| name     | type                  | constraints                                                        | errno-on-bad |
|----------|-----------------------|--------------------------------------------------------------------|--------------|
| dirfd    | `int`                 | open fd (must be directory, or `O_PATH` directory), or `AT_FDCWD` | `EBADF` / `ENOTDIR` |
| filename | `const char __user *` | NUL-terminated; length `< PATH_MAX`. May be `""` with `AT_EMPTY_PATH`. | `EFAULT` / `ENAMETOOLONG` |
| flags    | `int`                 | subset of `VALID_OPEN_FLAGS` ∪ `{AT_EMPTY_PATH, AT_NO_AUTOMOUNT}` (silently stripped otherwise) | (silently stripped) |
| mode     | `umode_t`             | `S_IALLUGO` (`07777`); honored only if `O_CREAT|O_TMPFILE`         | (silently stripped) |

## Return value

- Success: `>= 0` — new file descriptor.
- Failure: `< 0` — negated errno.

## Errors

Same as `open(2)` plus:

| errno      | condition                                                                              |
|------------|----------------------------------------------------------------------------------------|
| `EBADF`    | `dirfd` is not open and not `AT_FDCWD`; or fd lookup race.                             |
| `ENOTDIR`  | `dirfd` refers to a non-directory and `pathname` is relative (and not empty / not `AT_EMPTY_PATH`). |
| `ENOENT`   | Empty `pathname` without `AT_EMPTY_PATH`.                                              |
| All `open(2)` errors | `EFAULT`, `EACCES`, `ELOOP`, `ENAMETOOLONG`, `ENXIO`, `EISDIR`, `EROFS`, `ETXTBSY`, `ENOMEM`, `ENOSPC`, `EFBIG`, `EOVERFLOW`, `EOPNOTSUPP`, `EPERM`, `EBUSY`, `EEXIST`, `EMFILE`, `ENFILE`, `EWOULDBLOCK`. |

## ABI surface (constants + flags)

All `open(2)` flags apply (see `open.md`). Additional `AT_*` constants from `include/uapi/linux/fcntl.h`:

- `AT_FDCWD = -100` — sentinel directory fd meaning "use current working directory".
- `AT_EMPTY_PATH = 0x1000` — if `pathname` is `""`, operate on `dirfd` itself (effectively re-open via dirfd). Capability-gated: requires `CAP_DAC_READ_SEARCH` for unprivileged use on fds the caller didn't open.
- `AT_NO_AUTOMOUNT = 0x800` — suppress automount triggering during path resolution (don't traverse autofs / NFS automount points).
- `AT_SYMLINK_NOFOLLOW = 0x100` — *not* honored by `openat(2)` (use `O_NOFOLLOW` instead). The flag is documented as overloaded across the `*at` syscalls.

Mode bits and `O_*` constants are identical to `open(2)`; see `open.md` for full table.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=dirfd (i32)`, `%rsi=filename`, `%rdx=flags (i32)`, `%rcx`/`%r10=mode (umode_t)` (Linux uses `%r10` for arg 4; glibc rewrites `%rcx → %r10`).
- REQ-2: On 64-bit, `force_o_largefile()` ORs `O_LARGEFILE` into `flags`.
- REQ-3: `do_sys_open(dirfd, filename, flags, mode)` is identical to the `open(2)` path with `AT_FDCWD` replaced by `dirfd`.
- REQ-4: Path-resolution starts at:
  - `current->fs->pwd` if `dirfd == AT_FDCWD`,
  - else `fdget(dirfd)` — must succeed and the file must be `O_DIRECTORY` or `O_PATH` directory, else `-ENOTDIR`.
- REQ-5: Absolute `pathname` (starting with `/`) ignores `dirfd` — but `dirfd` validity is **not** re-checked when path is absolute (matches POSIX `openat` semantics).
- REQ-6: With `AT_EMPTY_PATH` and empty `pathname`, the operation re-opens `dirfd` itself with the requested flags. The new fd has independent `f_pos`, `f_flags`, etc.; this is a more general `dup` for converting `O_PATH` → real read/write fd (subject to LSM checks).
- REQ-7: All `open(2)` flag validation (`VALID_OPEN_FLAGS`, `O_PATH` strip, `O_TMPFILE` constraints, `O_DIRECTORY|O_CREAT` reject) applies identically.
- REQ-8: `AT_NO_AUTOMOUNT` translates to `LOOKUP_NO_AUTOMOUNT` in `nameidata`.
- REQ-9: `O_TMPFILE` honors `dirfd` as the parent directory for the anonymous inode; if `dirfd` is `AT_FDCWD`, the cwd is used.
- REQ-10: `getname` returns `-EFAULT` if `filename` is invalid; `-ENAMETOOLONG` if `> PATH_MAX-1`; `-ENOENT` if empty without `AT_EMPTY_PATH`.
- REQ-11: Returned fd is allocated from `current->files` with `O_CLOEXEC` honored atomically.
- REQ-12: `RLIMIT_NOFILE` enforced; exceeding ⟹ `-EMFILE`.

## Acceptance Criteria

- [ ] AC-1: `openat(AT_FDCWD, "foo", O_RDONLY)` ≡ `open("foo", O_RDONLY)`.
- [ ] AC-2: `openat(dirfd, "foo", ...)` resolves `foo` under `dirfd`'s directory, not cwd.
- [ ] AC-3: `openat(non_dir_fd, "foo", ...) == -ENOTDIR`.
- [ ] AC-4: `openat(closed_fd, "foo", ...) == -EBADF`.
- [ ] AC-5: `openat(dirfd, "/abs/path", ...)` ignores `dirfd` and resolves absolute.
- [ ] AC-6: `openat(o_path_dirfd, "", O_RDWR|AT_EMPTY_PATH)` re-opens with read-write (subject to LSM).
- [ ] AC-7: `openat(dirfd, "", O_RDONLY)` (without `AT_EMPTY_PATH`) → `-ENOENT`.
- [ ] AC-8: `O_TMPFILE` with `dirfd` creates anonymous inode under `dirfd`'s filesystem.
- [ ] AC-9: Returned fd respects `O_CLOEXEC` atomically.
- [ ] AC-10: Unknown bits in `flags` silently stripped.

## Architecture

```
struct OpenatArgs { dirfd: i32, filename: UserPtr<u8>, flags: i32, mode: u16 }
```

`sys_openat(args) -> i32`:

1. `let mut flags = args.flags as u32;`
2. If `force_o_largefile()` ⟹ `flags |= O_LARGEFILE;`
3. Return `do_sys_open(args.dirfd, args.filename, flags, args.mode)`.

`Fs::do_sys_open(dfd, filename, flags, mode) -> i32`:

(Same as `open.md` `Architecture`; the only difference is `dfd` is the user-supplied value, not `AT_FDCWD`.)

`Path::init_nameidata_with_dfd(dfd, name) -> Result<Nameidata, errno>`:

1. If `dfd == AT_FDCWD`:
   - `nd.path = current.fs.pwd_path.clone();`
   - Return `Ok(nd)`.
2. `let f = fdget(dfd)?;`
3. If `!(f.f_mode & FMODE_PATH || f.f_op.iterate_shared.is_some())` and path is relative ⟹ `return -ENOTDIR`.
4. `nd.path = f.f_path.clone();`
5. Return `Ok(nd)`.

(`AT_EMPTY_PATH` short-circuits inside `do_file_open` to clone `dfd`'s `struct file` semantics with new flags.)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dirfd_validated_for_relative` | INVARIANT | Relative path ⟹ `fdget(dirfd)` succeeded and is directory. |
| `at_empty_path_requires_flag` | INVARIANT | Empty path without `AT_EMPTY_PATH` ⟹ `-ENOENT`. |
| `absolute_path_ignores_dirfd` | INVARIANT | `pathname[0] == '/' ⟹ dfd not dereferenced`. |
| `at_no_automount_to_lookup` | INVARIANT | `AT_NO_AUTOMOUNT ⟹ LOOKUP_NO_AUTOMOUNT`. |

### Layer 2: TLA+

`uapi/syscalls/openat.tla`:
- Per-call → init_nameidata → path_openat → fd_install.
- Properties:
  - `safety_dirfd_or_AT_FDCWD`,
  - `safety_AT_EMPTY_PATH_gated`,
  - `liveness_openat_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Resolution start = `dirfd.path` (relative) or `current.cwd` (`AT_FDCWD`) or absolute root | `Path::init_nameidata_with_dfd` |
| `AT_EMPTY_PATH ⟹ pathname == ""` and capability check passes | `Fs::do_file_open` |
| All `open(2)` invariants apply | `Fs::do_sys_openat2` |

### Layer 4: Verus/Creusot functional

`openat(dirfd, path, flags, mode) ≡ POSIX.1-2024 openat` semantic equivalence.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening; inherits all `open(2)` hardening.)

`openat(2)` reinforcement:

- **Per-`dirfd` validation strict** — defense against using non-directory fd for relative lookup.
- **Per-`AT_EMPTY_PATH` CAP_DAC_READ_SEARCH gate** — defense against re-opening foreign fds.
- **Per-`AT_NO_AUTOMOUNT`** — defense against accidental autofs trigger leaking remote-mount probing.
- **Per-`AT_FDCWD` sentinel handling** — defense against negative-fd misuse (fdget(AT_FDCWD) would fail).
- **Per-`fdget(dirfd)` refcount discipline** — defense against fd reuse race during long path walks.
- **Per-`getname` length cap** — same as `open(2)`.

## Grsecurity/PaX-style Reinforcement

- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd processes cannot escape via dirfd inherited before chroot.
- **GRKERNSEC_CHROOT_FINDTASK** — `dirfd` rooted outside chroot is rejected.
- **GRKERNSEC_SYMLINKOWN / FIFO / LINK** — symlink/FIFO/hardlink protections apply during `openat` lookup.
- **GRKERNSEC_TPE** — Trusted-Path-Execution applies to `openat`-loaded executables.
- **PAX_UDEREF** — `filename` SMAP-guarded in `getname`.
- **PAX_REFCOUNT** — `dirfd` refcount and `struct file` refcount saturating.
- **PAX_MEMORY_SANITIZE** — `struct file` slab zeroed on free.
- **GRKERNSEC_AUDIT_MOUNT** — `openat` traversing a mount boundary logged.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at openat syscall entry.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `openat2(2)` (separate Tier-5 — `openat2.md`).
- LSM internals (`security/00-overview.md`).
- Path-resolution internals (covered in `fs/namei.md` Tier-3 if expanded).
- Implementation code.
