# Tier-5: syscall 83 — mkdir(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`83  common  mkdir  sys_mkdir`)
  - fs/namei.c (`SYSCALL_DEFINE2(mkdir, ...)`, `do_mkdirat`, `vfs_mkdir`)
  - include/uapi/linux/stat.h (`S_IFDIR`, `S_IRWXU`, ...)
-->

## Summary

`mkdir(2)` is **x86_64 syscall 83**, the POSIX directory creation primitive. It creates a new directory `pathname` with permission bits derived from `mode & ~umask & 0777` (the SUID/SGID/sticky high bits are filtered out for normal callers; the SGID bit is then re-applied if the parent directory is itself SGID, enabling group-inheritance semantics). The new directory contains two entries: `.` (pointing to itself) and `..` (pointing to its parent). The parent's `i_nlink` is incremented by one to account for the new directory's `..` entry; this is the kernel's tracking of "how many subdirectories have me as parent" used to determine directory deletability. `mkdir` is `mkdirat(AT_FDCWD, pathname, mode)` internally.

Critical for: every install script's tree creation, every `mkdir -p` invocation (which retries with `EEXIST` to detect race-create), every Rookery service's working-dir setup, every container image extraction creating layer directories, every `~/.config/<app>` first-run.

## Signature

C (POSIX-1.2008):

```c
int mkdir(const char *pathname, mode_t mode);
```

glibc wrapper: `__mkdir` → `INLINE_SYSCALL(mkdir, 2, pathname, mode)` on architectures exporting the bare syscall; otherwise forwarded to `mkdirat(AT_FDCWD, pathname, mode)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(mkdir, const char __user *, pathname, umode_t, mode);
```

Rookery dispatch:

```rust
pub fn sys_mkdir(pathname: UserPtr<u8>, mode: u16) -> SyscallResult<i32>;
```

## Parameters

| name     | type           | constraints                                                   | errno-on-bad |
|----------|----------------|---------------------------------------------------------------|--------------|
| pathname | `const char *` | NUL-terminated; ≤ `PATH_MAX`; parent must exist and be writable | `EFAULT` / `ENAMETOOLONG` / `EACCES` / `ENOENT` |
| mode     | `mode_t`       | Permission bits `0-0777`; SUID/SGID/sticky high bits ignored for non-root | (silently masked) |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.

## Errors

| errno          | condition                                                                  |
|----------------|----------------------------------------------------------------------------|
| `EACCES`       | Write or search permission denied on a prefix.                            |
| `EDQUOT`       | Quota exhausted on the FS.                                                |
| `EEXIST`       | `pathname` already exists.                                                |
| `EFAULT`       | `pathname` outside user address space.                                    |
| `EINVAL`       | Final component is `.` or `..`.                                           |
| `ELOOP`        | Too many symlinks resolved on prefix.                                     |
| `EMLINK`       | Parent dir's `i_nlink` cannot be incremented (subdir limit reached).      |
| `ENAMETOOLONG` | Path > `PATH_MAX` or component > `NAME_MAX`.                              |
| `ENOENT`       | A prefix component does not exist.                                        |
| `ENOMEM`       | Kernel memory exhausted.                                                  |
| `ENOSPC`       | No space on the FS.                                                       |
| `ENOTDIR`      | A prefix component is not a directory.                                    |
| `EPERM`        | FS doesn't support directories.                                           |
| `EROFS`        | Read-only FS.                                                             |

## ABI surface (constants + flags)

`mkdir(2)` has no flags. The effective mode is `(mode & ~current->umask) & S_IRWXUGO` plus the inherited SGID bit if the parent has SGID set (POSIX semantics).

Related kernel symbols:

- `vfs_mkdir(idmap, dir, dentry, mode)` — VFS layer; takes parent `i_mutex`, calls `dir->i_op->mkdir`.
- `may_create(idmap, dir, dentry, &lookup)` — DAC permission check.
- `security_inode_mkdir(dir, dentry, mode)` — LSM hook.
- `fsnotify_mkdir(dir, dentry)` — inotify/fanotify event emission.
- `inc_nlink(dir)` — parent's `i_nlink++` for new subdir's `..` entry.
- `mnt_want_write` / `mnt_drop_write` — freeze-aware write reservation.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=pathname (const char __user *)`, `%rsi=mode (u16)`.
- REQ-2: UDEREF on `pathname`.
- REQ-3: `strncpy_from_user_with_pax_check(pathname, PATH_MAX)`.
- REQ-4: Resolve parent (`LOOKUP_PARENT | LOOKUP_DIRECTORY`); last component must NOT exist.
- REQ-5: Final component must NOT be `.` or `..` ⟹ `-EINVAL` (the path-walk already catches this, but `vfs_mkdir` re-asserts).
- REQ-6: Mode masking: `effective_mode = (mode & ~current.umask) & 0777;` plus SGID inheritance.
- REQ-7: `mnt_want_write(parent.mnt)?;` — `-EROFS`.
- REQ-8: `may_create(idmap, parent.inode, &dentry, &Lookup::dir());`
- REQ-9: `security_inode_mkdir(parent.inode, &dentry, effective_mode);`
- REQ-10: `inode_lock(parent.inode);`
- REQ-11: `vfs_mkdir(idmap, parent.inode, &dentry, effective_mode);`:
  - allocate new inode of type `S_IFDIR`,
  - set `i_mode = S_IFDIR | effective_mode`,
  - set `i_uid = current.fsuid`, `i_gid` per SGID-inheritance,
  - call `dir->i_op->mkdir(dir, dentry, mode)`,
  - `inc_nlink(parent.inode); inc_nlink(new_inode);` (new inode starts at 2: self + `.`).
- REQ-12: `fsnotify_mkdir(parent.inode, &dentry);`
- REQ-13: `inode_unlock(parent.inode);`
- REQ-14: `mnt_drop_write(parent.mnt);`
- REQ-15: On failure, no inode allocation; refcounts balanced.

## Acceptance Criteria

- [ ] AC-1: `mkdir("foo", 0755)` ⟹ `0`; `stat("foo").st_mode == S_IFDIR | 0755` (modulo umask).
- [ ] AC-2: `mkdir("/existing", 0755)` ⟹ `-EEXIST`.
- [ ] AC-3: `mkdir("/nonexistent/foo", 0755)` ⟹ `-ENOENT`.
- [ ] AC-4: `mkdir("dir/.", 0755)` ⟹ `-EINVAL`.
- [ ] AC-5: `mkdir("foo", 07777)` with umask 022 ⟹ effective mode 0755.
- [ ] AC-6: Parent SGID set ⟹ new dir inherits parent's GID.
- [ ] AC-7: New dir starts with `i_nlink == 2`.
- [ ] AC-8: Parent dir's `i_nlink` increments by 1.
- [ ] AC-9: Read-only FS ⟹ `-EROFS`.
- [ ] AC-10: NULL pathname ⟹ `-EFAULT`.

## Architecture

```
struct MkdirArgs { pathname: UserPtr<u8>, mode: u16 }
```

`sys_mkdir(args) -> i32`:

1. `let name = strncpy_from_user_pax(args.pathname, PATH_MAX)?;`
2. Return `Fs::do_mkdirat(AT_FDCWD, name, args.mode)`.

`Fs::do_mkdirat(dfd, name, mode) -> i32`:

1. `let (parent, dentry) = filename_create(dfd, name, LookupFlags::DIRECTORY)?;`
2. `let effective_mode = (mode & !current.umask()) & 0o777;`
3. `mnt_want_write(parent.mnt)?;`
4. `may_create(idmap, parent.dentry.inode(), &dentry, &Lookup::dir())?;`
5. `security_inode_mkdir(parent.dentry.inode(), &dentry, effective_mode)?;`
6. `inode_lock(parent.dentry.inode());`
7. `vfs_mkdir(idmap, parent.dentry.inode(), &dentry, effective_mode)?;`
8. `fsnotify_mkdir(parent.dentry.inode(), &dentry);`
9. `inode_unlock(parent.dentry.inode());`
10. `mnt_drop_write(parent.mnt);`
11. Return `0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pax_uderef_pathname` | INVARIANT | pathname UDEREF-validated. |
| `mode_masked` | INVARIANT | effective mode = `(mode & ~umask) & 0777`; high bits stripped. |
| `parent_lock_held` | INVARIANT | parent `i_mutex` held during `vfs_mkdir`. |
| `nlink_increment_atomic` | INVARIANT | parent `i_nlink` increment happens inside `vfs_mkdir`. |
| `dot_dotdot_rejected` | INVARIANT | trailing `.`/`..` ⟹ `-EINVAL` before mutation. |

### Layer 2: TLA+

`uapi/syscalls/mkdir.tla`:
- Per-call → resolve(parent) → may_create → vfs_mkdir → fsnotify_mkdir.
- Properties:
  - `safety_no_duplicate_mkdir`,
  - `safety_parent_nlink_balanced`,
  - `safety_sgid_inheritance`,
  - `liveness_mkdir_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post (success): `dentry.inode().i_mode_type() == S_IFDIR`, `i_nlink == 2` | `vfs_mkdir` |
| Post (success): parent `i_nlink` += 1 | `vfs_mkdir` |
| Post (success): if parent SGID, new dir `i_gid == parent.i_gid` | `vfs_mkdir` |
| Post (error): no inode allocated; refcounts balanced | `Fs::do_mkdirat` |

### Layer 4: Verus/Creusot functional

`mkdir(pathname, mode)` ≡ `mkdirat(AT_FDCWD, pathname, mode)` per POSIX-1.2008.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mkdir(2)` reinforcement:

- **Per-`umask` application** — defense against world-writable directory creation by accident.
- **Per-SGID-inheritance** — POSIX group-collaboration semantics enforced consistently.
- **Per-`may_create` DAC** — defense against unauthorized directory injection.
- **Per-`security_inode_mkdir` LSM** — SELinux/AppArmor scope-enforcement.
- **Per-`fsnotify_mkdir`** — auditable inotify event.
- **Per-`mnt_want_write`** — defense against directory creation during freeze.
- **Per-parent `i_mutex`** — defense against race-creating same-name directories.
- **Per-`vfs_mkdir` SUID/sticky filter** — defense against accidental SUID-on-directory (a misfeature on Linux).

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF on `pathname`** — strict user/kernel pointer split; kernel pointer in argv rejected.
- **GRKERNSEC_LINK** — newly-created dir's parent must be writable by caller per stricter cross-uid rules than DAC.
- **GRKERNSEC_SYMLINKOWN** — when the path traverses a symlink in a sticky-bit dir owned by another UID, mkdir returns `-EACCES`. Defends `/tmp` race attacks.
- **GRKERNSEC_FIFO** — sticky-bit dir rules apply uniformly to dir-creation inside such dirs.
- **PAX_MEMORY_SANITIZE** — failed-mkdir transient inode buffer zeroed on free.
- **GRKERNSEC_PROC_USER** — `/proc/$pid/cwd` accurately reflects new dir to owner only.
- **GRKERNSEC_CHROOT_NICE** — chroot'd process cannot mkdir outside chroot via residual fd-walk.
- **GRKERNSEC_CHROOT_MKNOD** — companion policy: mknod inside chroot blocked by default, mkdir likewise scrutinized.
- **GRKERNSEC_HIDESYM** — mkdir-error dmesg redacts kernel pointers.
- **GRKERNSEC_AUDIT_PTRACE** — mkdir by ptracer on tracee FS auditable.
- **GRKERNSEC_DMESG** — mkdir-error dmesg lines CAP_SYSLOG-gated.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `mkdirat(2)` (separate Tier-5 doc; this is the AT_FDCWD-only ancestor).
- `rmdir(2)` — covered in `rmdir.md`.
- `mknod(2)`, `mknodat(2)` (separate Tier-5 docs).
- VFS `i_op->mkdir` per-filesystem (Tier-3 per FS).
- Implementation code.
