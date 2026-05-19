# Tier-5: syscall 267 — readlinkat(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`267  common  readlinkat  sys_readlinkat`)
  - fs/stat.c (`SYSCALL_DEFINE4(readlinkat, ...)`, `do_readlinkat`)
  - fs/namei.c (`vfs_readlink`, `__vfs_get_link`)
  - include/linux/namei.h (`LOOKUP_EMPTY`)
-->

## Summary

`readlinkat(2)` is **x86_64 syscall 267**, the directory-fd-relative + empty-path-capable variant of `readlink(2)`. It reads up to `bufsiz` bytes of the symbolic link `pathname` interpreted relative to `dirfd` (or `AT_FDCWD`) into the user buffer `buf`. Distinct from `readlink(2)`: if `pathname` is the empty string AND `dirfd` itself refers to a symlink opened with `O_PATH | O_NOFOLLOW`, the kernel reads the symlink content of `dirfd` directly — this is the **only** way to read a symlink referenced solely by file descriptor, and it underlies modern `realpath`/`canonicalize_file_name` implementations that want to avoid path-walk TOCTOU. The buffer is **not NUL-terminated**.

Critical for: every container runtime that resolves `/proc/$pid/exe` via fd, every secure realpath implementation, every Rookery init that holds an `O_PATH` reference to a config symlink and needs to inspect it without re-walking the path, every `fexecve`-style flow that wants to read the target of an executable symlink.

## Signature

C (Linux 2.6.16+, glibc 2.4+):

```c
ssize_t readlinkat(int dirfd, const char *pathname, char *buf, size_t bufsiz);
```

glibc wrapper: `__readlinkat` → `INLINE_SYSCALL(readlinkat, 4, dirfd, pathname, buf, bufsiz)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(readlinkat, int, dfd, const char __user *, pathname, char __user *, buf, int, bufsiz);
```

Rookery dispatch:

```rust
pub fn sys_readlinkat(dirfd: i32, pathname: UserPtr<u8>, buf: UserPtr<u8>, bufsiz: i32) -> SyscallResult<isize>;
```

## Parameters

| name     | type           | constraints                                                       | errno-on-bad |
|----------|----------------|-------------------------------------------------------------------|--------------|
| dirfd    | `int`          | open dirfd, `AT_FDCWD`, or `O_PATH|O_NOFOLLOW` fd to a symlink    | `EBADF` / `ENOTDIR` |
| pathname | `const char *` | NUL-terminated; ≤ `PATH_MAX`; empty allowed iff dirfd is a symlink fd | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| buf      | `char *`       | writable for at least `bufsiz` bytes                              | `EFAULT` |
| bufsiz   | `int`          | MUST be `> 0`                                                     | `EINVAL` if `≤ 0` |

## Return value

- Success: number of bytes copied (`> 0`, `≤ bufsiz`). NOT NUL-terminated.
- Failure: `< 0` — negated errno.

## Errors

| errno          | condition                                                                  |
|----------------|----------------------------------------------------------------------------|
| `EACCES`       | Search permission denied on a prefix component.                            |
| `EBADF`        | `dirfd` is not a valid open fd and is not `AT_FDCWD`.                      |
| `EFAULT`       | `pathname` or `buf` outside the caller's address space.                    |
| `EINVAL`       | `bufsiz <= 0`; or `pathname` non-symlink; or empty `pathname` with non-symlink `dirfd`. |
| `EIO`          | I/O error reading symlink content.                                         |
| `ELOOP`        | Too many symlinks in prefix.                                               |
| `ENAMETOOLONG` | A component exceeds `NAME_MAX`, or path > `PATH_MAX`.                      |
| `ENOENT`       | A prefix component does not exist; or empty `pathname` and `dirfd` does not designate a symlink. |
| `ENOMEM`       | Kernel memory exhausted.                                                   |
| `ENOTDIR`      | `dirfd` is not `AT_FDCWD`, is non-empty `pathname`, and `dirfd` is not a directory. |

## ABI surface (constants + flags)

`readlinkat(2)` has no explicit flags argument; the empty-path behaviour is implicit when `pathname == ""`. Internally `do_readlinkat` sets `LOOKUP_EMPTY` when `pathname` is empty so that the dirfd's dentry itself is used as the target.

`AT_FDCWD = -100` — magic dirfd meaning "current working directory".

Related kernel symbols:

- `vfs_readlink(dentry, buf, bufsiz)` — VFS entry; calls `inode->i_op->readlink`.
- `getname_flags(pathname, LOOKUP_EMPTY)` — path-name copy supporting empty path.
- `path_lookupat` — name walker.
- `security_inode_readlink(dentry)` — LSM.
- `audit_inode(name, dentry, flags)` — audit.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=dfd (i32)`, `%rsi=pathname (const char __user *)`, `%rdx=buf (char __user *)`, `%r10=bufsiz (i32)`.
- REQ-2: `bufsiz <= 0 ⟹ -EINVAL` before any path walk.
- REQ-3: UDEREF: `pathname` and `buf` must be user pointers.
- REQ-4: `getname_flags(pathname, LOOKUP_EMPTY)` — empty allowed.
- REQ-5: If `pathname.is_empty()` AND `dfd != AT_FDCWD`:
  - `let dirfd_path = fdget_raw(dfd)?;`
  - If `dirfd_path.dentry.inode().i_mode & S_IFMT != S_IFLNK ⟹ -EINVAL`.
  - Read directly from `dirfd_path.dentry`.
- REQ-6: Otherwise `filename_lookup(dfd, pathname, LookupFlags::empty())` — does not follow trailing symlink.
- REQ-7: If resolved dentry's inode is not `S_IFLNK ⟹ -EINVAL`.
- REQ-8: `security_inode_readlink(dentry)`.
- REQ-9: `vfs_readlink(dentry, buf, bufsiz)` — `min(strlen(link), bufsiz)` bytes copied.
- REQ-10: Truncation is silent.
- REQ-11: On error, no bytes copied to `buf`.

## Acceptance Criteria

- [ ] AC-1: `readlinkat(dirfd, "symlink", buf, 1024)` ⟹ link content copied.
- [ ] AC-2: `readlinkat(symlink_o_path_nofollow_fd, "", buf, 1024)` ⟹ content of the symlink referenced by fd.
- [ ] AC-3: `readlinkat(regular_file_fd, "", buf, 1024)` ⟹ `-EINVAL`.
- [ ] AC-4: `readlinkat(AT_FDCWD, "", buf, 1024)` ⟹ `-ENOENT`.
- [ ] AC-5: `readlinkat(-99 /* bad */, "x", buf, 1024)` ⟹ `-EBADF`.
- [ ] AC-6: `readlinkat(dirfd, "regular_file", buf, 1024)` ⟹ `-EINVAL`.
- [ ] AC-7: `readlinkat(dirfd, "path", buf, 0)` ⟹ `-EINVAL`.
- [ ] AC-8: Truncation: target length 100, bufsiz 50 ⟹ returns 50.
- [ ] AC-9: NULL `buf` ⟹ `-EFAULT`.
- [ ] AC-10: Resolving link content via the fd path does NOT re-traverse the original directory chain.

## Architecture

```
struct ReadlinkatArgs {
  dirfd: i32, pathname: UserPtr<u8>, buf: UserPtr<u8>, bufsiz: i32,
}
```

`sys_readlinkat(args) -> isize`:

1. If `args.bufsiz <= 0` ⟹ return `-EINVAL`.
2. `let name = getname_flags_pax(args.pathname, LookupFlags::EMPTY)?;`
3. Return `Fs::do_readlinkat(args.dirfd, name, args.buf, args.bufsiz as usize)`.

`Fs::do_readlinkat(dfd, name, buf, bufsiz) -> isize`:

1. If `name.is_empty()`:
   - `let fd_path = fdget_raw(dfd).ok_or(EBADF)?;`
   - `if fd_path.dentry.inode().i_mode_type() != FileType::Symlink ⟹ -EINVAL;`
   - dentry = fd_path.dentry.
2. Else:
   - `let dentry = filename_lookup(dfd, name, LookupFlags::empty())?.dentry;`
   - `if dentry.inode().i_mode_type() != FileType::Symlink ⟹ -EINVAL;`
3. `security_inode_readlink(&dentry)?;`
4. `let n = vfs_readlink(&dentry, buf, bufsiz)?;`
5. `audit_inode(name, &dentry, 0);`
6. Return `n as isize`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bufsiz_positive` | INVARIANT | `bufsiz <= 0 ⟹ -EINVAL` before lookup. |
| `empty_path_needs_symlink_fd` | INVARIANT | empty `pathname` only valid if `dirfd` is symlink. |
| `not_symlink_rejected` | INVARIANT | non-symlink dentry ⟹ `-EINVAL`. |
| `truncation_silent` | INVARIANT | `bytes_copied = min(strlen(link), bufsiz)`. |
| `pax_uderef_paths_buf` | INVARIANT | UDEREF on all user pointers. |

### Layer 2: TLA+

`uapi/syscalls/readlinkat.tla`:
- Per-call → resolve(dirfd, pathname, EMPTY) → check IFLNK → vfs_readlink → copy_to_user.
- Properties:
  - `safety_empty_path_requires_symlink_fd`,
  - `safety_non_symlink_returns_einval`,
  - `liveness_readlinkat_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): bytes copied == `min(strlen(link_text), bufsiz)` | `vfs_readlink` |
| Post (error): no bytes copied | `Fs::do_readlinkat` |
| Post: no fd refcount leak when `dirfd` used | `Fs::do_readlinkat` |

### Layer 4: Verus/Creusot functional

`readlinkat(dirfd, pathname, buf, bufsiz)` ≡ Linux-specific semantics per `man 2 readlinkat`. POSIX-2008 equivalent.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`readlinkat(2)` reinforcement:

- **Per-`dirfd` fd-cap inheritance** — capability-bound dirfd cannot be escalated by readlinkat to gain content access outside scope.
- **Per-`O_PATH|O_NOFOLLOW` empty-path mode** — defense against TOCTOU between path-walk and readlink.
- **Per-`security_inode_readlink`** — LSM mediates.
- **Per-`-EINVAL` non-symlink check** — defense against type-confusion.
- **Per-`bufsiz > 0`** — defense against negative-length copy.
- **Per-`audit_inode`** — auditable trail.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF on `pathname`/`buf`** — strict user/kernel pointer split.
- **GRKERNSEC_SYMLINKOWN** — reading a symlink in a sticky-bit directory owned by another UID ⟹ `-EACCES`. Defends `/tmp` race; uniform with `readlink(2)`.
- **GRKERNSEC_PROC_USER** — readlinkat against `/proc/$other_uid_pid/exe` requires `CAP_SYS_PTRACE` or ownership.
- **GRKERNSEC_CHROOT_FINDTASK** — chroot'd process cannot readlinkat against a `/proc/$pid` outside chroot's pid namespace.
- **PAX_MEMORY_SANITIZE** — failed-readlinkat transient buffers zeroed on free.
- **GRKERNSEC_HIDESYM** — error dmesg redacts kernel pointers.
- **fd-cap-inheritance on dirfd** — capability rights on `dirfd` (e.g., `CAP_LOOKUP`, `CAP_READ`) are intersected with the readlinkat operation.
- **GRKERNSEC_AUDIT_PTRACE** — readlinkat by ptracer on tracee fd auditable.
- **GRKERNSEC_DMESG** — error dmesg lines CAP_SYSLOG-gated.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `readlink(2)` — covered in `readlink.md`.
- `symlink(2)` / `symlinkat(2)` — covered in `symlink.md`.
- `realpath(3)` (libc).
- VFS `i_op->readlink` per-filesystem (Tier-3 per FS).
- Implementation code.
