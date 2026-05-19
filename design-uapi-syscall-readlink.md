---
title: "Tier-5: syscall 89 — readlink(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`readlink(2)` is **x86_64 syscall 89**, the POSIX symbolic-link content-reading primitive. It reads up to `bufsiz` bytes of the symbolic link `pathname` into the user buffer `buf`, returning the number of bytes copied. The buffer is **not NUL-terminated**; the caller is responsible for either limiting the read to `bufsiz - 1` and appending a `'\0'`, or using `lstat(2)`'s `st_size` to size the buffer correctly. If the link is longer than `bufsiz`, the result is silently truncated — there is no `ENAMETOOLONG`. On non-symlink targets the call returns `-EINVAL`. Magic links under `/proc/self/{fd,exe,cwd,root}` etc. are served by per-link `i_op->get_link` callbacks that synthesize the content from kernel state rather than reading a stored payload.

Critical for: every container runtime that resolves `/proc/$pid/exe`, every `realpath(3)`/`canonicalize_file_name(3)` walker, every package manager checking `/etc/alternatives/*`, every Rookery init reading `/proc/self/exe` to locate itself.

### Acceptance Criteria

- [ ] AC-1: `readlink("symlink", buf, 1024)` ⟹ bytes copied, no NUL.
- [ ] AC-2: `readlink("regular_file", buf, 1024)` ⟹ `-EINVAL`.
- [ ] AC-3: `readlink("path", buf, 0)` ⟹ `-EINVAL`.
- [ ] AC-4: `readlink("path", buf, -1)` ⟹ `-EINVAL`.
- [ ] AC-5: Truncation: target length 100, `bufsiz = 50` ⟹ returns 50, `buf` holds first 50 bytes.
- [ ] AC-6: `readlink("nonexistent", buf, 1024)` ⟹ `-ENOENT`.
- [ ] AC-7: `readlink("/proc/self/exe", buf, 1024)` ⟹ path of current binary.
- [ ] AC-8: `readlink("dir/x", buf, 1024)` where prefix has `ELOOP` ⟹ `-ELOOP`.
- [ ] AC-9: NULL `buf` ⟹ `-EFAULT`.
- [ ] AC-10: Empty pathname ⟹ `-ENOENT`.

### Architecture

```
struct ReadlinkArgs { pathname: UserPtr<u8>, buf: UserPtr<u8>, bufsiz: i32 }
```

`sys_readlink(args) -> isize`:

1. If `args.bufsiz <= 0` ⟹ return `-EINVAL`.
2. `let path = strncpy_from_user_pax(args.pathname, PATH_MAX)?;`
3. If `path.is_empty()` ⟹ return `-ENOENT`.
4. Return `Fs::do_readlinkat(AT_FDCWD, path, args.buf, args.bufsiz as usize)`.

`Fs::do_readlinkat(dfd, path, buf, bufsiz) -> isize`:

1. `let dentry = filename_lookup(dfd, path, LookupFlags::empty())?.dentry;` — does not follow trailing symlink.
2. `if dentry.inode().i_mode_type() != FileType::Symlink ⟹ -EINVAL;`
3. `security_inode_readlink(&dentry)?;`
4. `let len = vfs_readlink(&dentry, buf, bufsiz)?;`
5. `audit_inode(name, &dentry, 0);`
6. Return `len as isize`.

`vfs_readlink(dentry, ubuf, bufsiz) -> isize`:

1. Acquire link text:
   - If `inode.i_link != NULL` ⟹ `link = inode.i_link; len = strlen(link);`
   - Else ⟹ call `inode.i_op.readlink(dentry, ubuf, bufsiz)` directly (page_readlink / magic).
2. `let n = min(len, bufsiz);`
3. `copy_to_user_pax(ubuf, link, n)?;` — `-EFAULT` if buf invalid.
4. Return `n`.

### Out of Scope

- `readlinkat(2)` (separate Tier-5 doc; this is the AT_FDCWD-only ancestor).
- `symlink(2)` — covered in `symlink.md`.
- `realpath(3)` (libc routine combining `readlink` and lstat).
- VFS `i_op->readlink` per-filesystem implementations (Tier-3 per FS).
- Implementation code.

### signature

C (POSIX-1.2008):

```c
ssize_t readlink(const char *pathname, char *buf, size_t bufsiz);
```

glibc wrapper: `__readlink` → `INLINE_SYSCALL(readlink, 3, pathname, buf, bufsiz)` on architectures exporting the bare syscall; otherwise forwarded to `readlinkat(AT_FDCWD, pathname, buf, bufsiz)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(readlink, const char __user *, path, char __user *, buf, int, bufsiz);
```

Rookery dispatch:

```rust
pub fn sys_readlink(pathname: UserPtr<u8>, buf: UserPtr<u8>, bufsiz: i32) -> SyscallResult<isize>;
```

### parameters

| name     | type           | constraints                                                       | errno-on-bad |
|----------|----------------|-------------------------------------------------------------------|--------------|
| pathname | `const char *` | NUL-terminated; ≤ `PATH_MAX`; final component MUST be `S_IFLNK`   | `EFAULT` / `ENAMETOOLONG` / `EINVAL` (not symlink) |
| buf      | `char *`       | writable for at least `bufsiz` bytes                              | `EFAULT` |
| bufsiz   | `int`          | MUST be `> 0`                                                     | `EINVAL` if `≤ 0` |

### return value

- Success: number of bytes placed in `buf` (`> 0`, `≤ bufsiz`). NOT NUL-terminated.
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                  |
|----------------|----------------------------------------------------------------------------|
| `EACCES`       | Search permission denied on a component of the path prefix.                |
| `EFAULT`       | `pathname` or `buf` outside the caller's address space.                    |
| `EINVAL`       | `bufsiz ≤ 0`; or `pathname` is not a symbolic link.                        |
| `EIO`          | An I/O error occurred reading the symlink content.                         |
| `ELOOP`        | Too many symlinks in the prefix.                                           |
| `ENAMETOOLONG` | A component of `pathname` exceeds `NAME_MAX`, or full path > `PATH_MAX`.   |
| `ENOENT`       | A component of `pathname` does not exist.                                  |
| `ENOMEM`       | Kernel memory exhausted.                                                   |
| `ENOTDIR`      | A component of the prefix is not a directory.                              |

### abi surface (constants + flags)

`readlink(2)` is flagless. Internally the kernel calls `do_readlinkat(AT_FDCWD, pathname, buf, bufsiz)`. The lookup uses `LOOKUP_EMPTY` is OFF — empty `pathname` is `-ENOENT` (unlike `readlinkat(AT_EMPTY_PATH)` which permits empty path with an fd).

Symlink content source:

- **Inline (fast) symlinks**: stored in `inode->i_link` directly — single page-uncached copy via `readlink_copy`.
- **Block (slow) symlinks**: read via `page_readlink` from the symlink's data block; uses page cache.
- **Magic symlinks**: per-link `i_op->get_link(dentry, inode, &done)` synthesizes content (e.g., `/proc/self/exe` returns the current task's exe path).

Related kernel symbols:

- `vfs_readlink(dentry, buffer, buflen)` — VFS entry; calls `inode->i_op->readlink`.
- `readlink_copy(buffer, buflen, link)` — bounded `copy_to_user` of symlink bytes.
- `security_inode_readlink(dentry)` — LSM hook.
- `audit_inode(name, dentry, flags)` — audit record.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=path (const char __user *)`, `%rsi=buf (char __user *)`, `%rdx=bufsiz (i32)`.
- REQ-2: `bufsiz <= 0 ⟹ -EINVAL` before any path walk.
- REQ-3: UDEREF: both `pathname` and `buf` must be user pointers.
- REQ-4: `strncpy_from_user_with_pax_check(path, PATH_MAX)`.
- REQ-5: Empty `path` ⟹ `-ENOENT` (in contrast to `readlinkat` which permits empty path with `AT_EMPTY_PATH`).
- REQ-6: `filename_lookup(AT_FDCWD, path, LookupFlags::empty())` — does NOT follow trailing symlink (we want to read it, not its target).
- REQ-7: If `dentry.inode().i_mode & S_IFMT != S_IFLNK ⟹ -EINVAL`.
- REQ-8: `security_inode_readlink(dentry)` — LSM gate.
- REQ-9: `vfs_readlink(dentry, buf, bufsiz)`:
  - Call `inode->i_op->readlink(dentry, buffer, buflen)`,
  - returns `min(len(link_text), bufsiz)` copied.
- REQ-10: On success return bytes copied (`≥ 0`).
- REQ-11: Truncation is silent; no errno.
- REQ-12: `buf` MUST be writable per `access_ok(USER_DS, buf, bufsiz)`; failure ⟹ `-EFAULT`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bufsiz_positive` | INVARIANT | `bufsiz <= 0 ⟹ -EINVAL` before any deref. |
| `not_symlink_rejected` | INVARIANT | non-symlink target ⟹ `-EINVAL`. |
| `truncation_silent` | INVARIANT | If `link_text_len > bufsiz`, returns `bufsiz`, no errno. |
| `no_nul_terminator` | INVARIANT | `buf` not NUL-terminated. |
| `pax_uderef_buffers` | INVARIANT | `pathname` and `buf` UDEREF-validated. |

### Layer 2: TLA+

`uapi/syscalls/readlink.tla`:
- Per-call → resolve(no-follow) → check IFLNK → vfs_readlink → copy_to_user.
- Properties:
  - `safety_non_symlink_returns_einval`,
  - `safety_truncation_does_not_corrupt`,
  - `liveness_readlink_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): bytes copied == `min(strlen(link_text), bufsiz)` | `vfs_readlink` |
| Post (error EINVAL): zero bytes touched in `buf` | `Fs::do_readlinkat` |
| Post: no inode/dentry refcount leaks | `Fs::do_readlinkat` |

### Layer 4: Verus/Creusot functional

`readlink(pathname, buf, bufsiz)` ≡ `readlinkat(AT_FDCWD, pathname, buf, bufsiz)` per POSIX-1.2008.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`readlink(2)` reinforcement:

- **Per-`security_inode_readlink`** — LSM mediates reading of sensitive symlinks (e.g., AppArmor blocks `/proc/$other_pid/exe` per profile).
- **Per-`-EINVAL` on non-symlink** — defense against type-confusion (returning regular-file content via readlink).
- **Per-`bufsiz > 0`** — defense against arithmetic underflow + negative length copy.
- **Per-`copy_to_user_pax` bounded** — defense against kernel-buffer overread.
- **Per-no-follow-trailing-symlink** — defense against TOCTOU between `readlink` and earlier `lstat`.
- **Per-`audit_inode`** — auditable read trail.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF on `path`/`buf`** — kernel pointer in argv rejected before deref.
- **GRKERNSEC_SYMLINKOWN** — when reading a symlink in a sticky-bit dir owned by a different UID, `readlink` returns `-EACCES`. Defends `/tmp` race attacks where attacker plants a symlink victim then reads it.
- **GRKERNSEC_PROC_USER** — `readlink("/proc/$other_uid_pid/exe")` returns `-EACCES` unless caller is owner or `CAP_SYS_PTRACE`.
- **GRKERNSEC_HIDESYM** — readlink-error dmesg redacts kernel pointers.
- **PAX_MEMORY_SANITIZE** — failed-readlink transient buffer zeroed on free.
- **GRKERNSEC_CHROOT_FINDTASK** — chroot'd process cannot `readlink("/proc/$pid_outside_chroot/exe")`.
- **GRKERNSEC_AUDIT_PTRACE** — readlink by ptracer on tracee FS auditable.
- **GRKERNSEC_DMESG** — readlink-error dmesg lines CAP_SYSLOG-gated.

