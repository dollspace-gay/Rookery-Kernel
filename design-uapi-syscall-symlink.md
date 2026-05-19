---
title: "Tier-5: syscall 88 — symlink(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`symlink(2)` is **x86_64 syscall 88**, the POSIX symbolic-link creation primitive. It creates a new filesystem entry at `linkpath` of type `S_IFLNK` whose payload is the **literal byte string** `target`. The kernel performs **no** validation that `target` resolves to an existing file — symlinks are lazily evaluated at lookup time. Unlike `link(2)`, the source `target` is treated as an opaque string and may cross filesystem boundaries, name nonexistent files, or be arbitrarily long up to `PATH_MAX - 1`. The new symlink's inode is owned by the caller's effective UID/GID, modulo the parent directory's SGID bit.

Critical for: every `/etc/alternatives` chain, every `/proc/self` magic-link, every container overlay's tombstone symlinks, every `ln -s` invocation, every `pkg-config` style symlink farm, every Rookery init-system unit `Install.WantedBy` symlink.

### Acceptance Criteria

- [ ] AC-1: `symlink("foo", "bar")` ⟹ `lstat("bar").st_mode & S_IFMT == S_IFLNK`.
- [ ] AC-2: `readlink("bar", buf, n)` ⟹ buf contains `"foo"`.
- [ ] AC-3: `symlink("", "bar")` ⟹ `-ENOENT`.
- [ ] AC-4: `symlink("/nonexistent/path", "bar")` ⟹ success (no target validation).
- [ ] AC-5: Existing `linkpath` ⟹ `-EEXIST`.
- [ ] AC-6: NULL pointers ⟹ `-EFAULT`.
- [ ] AC-7: Target of `PATH_MAX` length ⟹ `-ENAMETOOLONG`.
- [ ] AC-8: Read-only mount ⟹ `-EROFS`.
- [ ] AC-9: Filesystem without symlink support (FAT) ⟹ `-EPERM`.
- [ ] AC-10: Resulting inode `i_uid == current.fsuid`, `i_gid == current.fsgid` (or parent's gid if SGID).

### Architecture

```
struct SymlinkArgs { target: UserPtr<u8>, linkpath: UserPtr<u8> }
```

`sys_symlink(args) -> i32`:

1. `let target = strncpy_from_user_pax(args.target, PATH_MAX)?;`
2. If `target.is_empty()` ⟹ `-ENOENT`.
3. `let linkname = strncpy_from_user_pax(args.linkpath, PATH_MAX)?;`
4. Return `Fs::do_symlinkat(target, AT_FDCWD, linkname)`.

`Fs::do_symlinkat(target, newdfd, newname) -> i32`:

1. `let (parent, dentry) = filename_create(newdfd, newname, LookupFlags::DIRECTORY)?;`
2. `mnt_want_write(parent.mnt)?;` — `-EROFS`.
3. `security_inode_symlink(parent.dentry.inode(), &dentry, &target)?;`
4. `inode_lock(parent.dentry.inode());`
5. `vfs_symlink(idmap, parent.dentry.inode(), &dentry, &target)?;`
6. `fsnotify_create(parent.dentry.inode(), &dentry);`
7. `inode_unlock(parent.dentry.inode());`
8. `mnt_drop_write(parent.mnt);`
9. Return `0`.

### Out of Scope

- `symlinkat(2)` (separate Tier-5 doc; `symlink` is the AT_FDCWD-only ancestor).
- `readlink(2)` / `readlinkat(2)` — covered in `readlink.md` / `readlinkat.md`.
- `link(2)` — covered in `link.md`.
- VFS `i_op->symlink` per-filesystem (Tier-3 per FS).
- Implementation code.

### signature

C (POSIX-1.2008):

```c
int symlink(const char *target, const char *linkpath);
```

glibc wrapper: `__symlink` → `INLINE_SYSCALL(symlink, 2, target, linkpath)` on architectures exporting the bare syscall; otherwise forwarded to `symlinkat(target, AT_FDCWD, linkpath)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(symlink, const char __user *, oldname, const char __user *, newname);
```

Rookery dispatch:

```rust
pub fn sys_symlink(target: UserPtr<u8>, linkpath: UserPtr<u8>) -> SyscallResult<i32>;
```

### parameters

| name     | type           | constraints                                                        | errno-on-bad |
|----------|----------------|--------------------------------------------------------------------|--------------|
| target   | `const char *` | NUL-terminated; `≤ PATH_MAX - 1` (the payload, not a path lookup)  | `EFAULT` / `ENAMETOOLONG` / `EINVAL` (empty) |
| linkpath | `const char *` | NUL-terminated path; ≤ `PATH_MAX`; parent must exist and be writable | `EFAULT` / `ENAMETOOLONG` / `ENOENT` / `EACCES` |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno.

### errors

| errno          | condition                                                                |
|----------------|--------------------------------------------------------------------------|
| `EACCES`       | Write access to parent of `linkpath` denied, or search permission denied on a component of either prefix. |
| `EDQUOT`       | Quota exhausted on the filesystem containing `linkpath`.                |
| `EEXIST`       | `linkpath` already exists.                                              |
| `EFAULT`       | One of the pointers is outside the caller's address space.              |
| `EIO`          | I/O error on the filesystem.                                            |
| `ELOOP`        | Too many symlinks resolved during `linkpath` parent traversal.          |
| `ENAMETOOLONG` | `target` exceeds `PATH_MAX - 1`, or a component of `linkpath` exceeds `NAME_MAX`. |
| `ENOENT`       | A component of `linkpath`'s prefix does not exist; or `target` is the empty string. |
| `ENOMEM`       | Kernel memory exhausted.                                                |
| `ENOSPC`       | No space on the filesystem.                                             |
| `ENOTDIR`      | A component of `linkpath`'s prefix is not a directory.                  |
| `EPERM`        | Filesystem does not support symlinks (e.g., FAT32).                     |
| `EROFS`        | `linkpath` would reside on a read-only filesystem.                      |

### abi surface (constants + flags)

`symlink(2)` is flagless. Internally the kernel calls `do_symlinkat(target, AT_FDCWD, linkpath)`. The symlink-content layer:

- **Fast symlinks**: targets ≤ `i_inode_inline_data_size` (per-FS; ~60 bytes for ext4) stored inline in the inode.
- **Slow symlinks**: longer targets allocate a separate data block; readable via `inode->i_op->get_link`.

Related kernel symbols:

- `vfs_symlink(idmap, dir, dentry, oldname)` — VFS layer; takes `i_mutex` on parent.
- `security_inode_symlink(dir, dentry, oldname)` — LSM hook.
- `fsnotify_create(dir, new_dentry)` — inotify/fanotify creation event.
- `mnt_want_write` / `mnt_drop_write` — freeze-aware write reservation.
- `dquot_alloc_inode` — quota allocation for the new inode.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=oldname (const char __user *)`, `%rsi=newname (const char __user *)`.
- REQ-2: Both pointers MUST pass UDEREF; kernel pointers in `argv` ⟹ `-EFAULT` (PAX_UDEREF enforced).
- REQ-3: `strncpy_from_user_with_pax_check(target, PATH_MAX)`; empty target ⟹ `-ENOENT` (POSIX-required).
- REQ-4: `strncpy_from_user_with_pax_check(linkpath, PATH_MAX)`.
- REQ-5: Resolve `linkpath` parent (`LOOKUP_PARENT | LOOKUP_DIRECTORY`); last component must NOT exist.
- REQ-6: `mnt_want_write(parent.mnt)?;` — `-EROFS` if frozen/read-only.
- REQ-7: `security_inode_symlink(dir, dentry, target)`.
- REQ-8: `vfs_symlink(idmap, dir, dentry, target)`:
  - allocate inode of type `S_IFLNK`,
  - copy `target` into inline-data area or allocate symlink block,
  - set `i_mode = S_IFLNK | 0777` (POSIX symlinks have full permissions; access uses target's perms),
  - assign `i_uid = current_fsuid`, `i_gid = current_fsgid` (SGID-on-parent overrides),
  - `inc_nlink(dir); fsnotify_create(dir, dentry);`.
- REQ-9: On failure, no inode allocated; refcounts balanced.
- REQ-10: SUID/SGID are NOT meaningful for symlinks; mode 0777 is canonical.
- REQ-11: Symlink content is opaque bytes; no UTF-8 validation; embedded NULs forbidden (string is C-string).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pax_uderef_target_link` | INVARIANT | Both pointers UDEREF-validated before deref. |
| `empty_target_rejected` | INVARIANT | Empty `target` ⟹ `-ENOENT`. |
| `lookup_excl_enforced` | INVARIANT | `linkpath` must not exist; `EEXIST` otherwise. |
| `parent_lock_held` | INVARIANT | parent `i_mutex` held during `vfs_symlink`. |
| `no_target_validation` | INVARIANT | Target is stored verbatim; not resolved. |

### Layer 2: TLA+

`uapi/syscalls/symlink.tla`:
- Per-call → copy(target) → copy(linkpath) → filename_create → vfs_symlink → fsnotify.
- Properties:
  - `safety_target_byte_preserved`,
  - `safety_empty_target_errno`,
  - `liveness_symlink_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): new dentry's inode is `S_IFLNK`, mode `0777` | `vfs_symlink` |
| Post (success): `readlink(linkpath)` returns exact bytes of `target` | `vfs_symlink` |
| Post (error): no inode allocated; quota/disk untouched | `Fs::do_symlinkat` |

### Layer 4: Verus/Creusot functional

`symlink(target, linkpath)` ≡ `symlinkat(target, AT_FDCWD, linkpath)` per POSIX-1.2008.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`symlink(2)` reinforcement:

- **Per-`mnt_want_write`** — defense against freeze-time symlink injection.
- **Per-`security_inode_symlink` LSM** — SELinux/AppArmor mediates symlink creation (denies symlinks pointing outside profile-allowed scope).
- **Per-`fsnotify_create`** — auditable inotify event for forensic trail.
- **Per-PATH_MAX target ceiling** — defense against unbounded symlink-data allocation DoS.
- **Per-`dquot_alloc_inode`** — quota accounting prevents per-user symlink-storm DoS.
- **Per-parent `i_mutex`** — defense against race-creating-symlinks-with-same-name.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF on `target`/`linkpath`** — strict user/kernel pointer split; kernel pointer in argv rejected.
- **GRKERNSEC_SYMLINKOWN** — when symlink would land in a sticky-bit directory (e.g., `/tmp`), the caller's UID is recorded; later `follow_link` by a different UID returns `-EACCES`. Defends `/tmp` symlink-race attacks (CVE-1996-class).
- **GRKERNSEC_LINK** — uniform with `link(2)`: symlinks inherit the cross-uid restriction in tmpdirs.
- **GRKERNSEC_FIFO** — symlinks targeting FIFOs/sockets across UID boundaries in sticky-bit dirs blocked.
- **GRKERNSEC_CHROOT_NICE** — symlink content treated as a string in chroot, but follow-link inside chroot cannot escape (per chroot semantics).
- **PAX_MEMORY_SANITIZE** — failed-symlink transient buffers zeroed on free.
- **GRKERNSEC_HIDESYM** — symlink-error dmesg redacts kernel pointers.
- **GRKERNSEC_AUDIT_PTRACE** — symlink by ptracer onto tracee FS auditable.
- **GRKERNSEC_DMESG** — symlink-error dmesg lines CAP_SYSLOG-gated.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/maps` and `/proc/$pid/fd` referencing the new symlink are owner-only.

