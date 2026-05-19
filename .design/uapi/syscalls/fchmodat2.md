# Tier-5: syscall 452 — fchmodat2(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`452  common  fchmodat2  sys_fchmodat2`)
  - fs/open.c (`SYSCALL_DEFINE4(fchmodat2, ...)`, `do_fchmodat`, `vfs_fchmod`, `chmod_common`)
  - include/uapi/linux/fcntl.h (`AT_FDCWD`, `AT_EMPTY_PATH`, `AT_SYMLINK_NOFOLLOW`)
-->

## Summary

`fchmodat2(2)` is **x86_64 syscall 452** (added in Linux 6.6), the corrected directory-relative chmod primitive that finally accepts the `flags` argument glibc's `fchmodat(3)` had been *pretending* the kernel supported for a decade. Unlike `fchmodat(2)` syscall 268 — which has no `flags` parameter and silently dropped any flag bits glibc tried to forward — `fchmodat2(2)` honors `AT_SYMLINK_NOFOLLOW` (operate on the symlink itself, not its target) and `AT_EMPTY_PATH` (operate on `dirfd` directly when `pathname == ""`). It changes the file-mode bits (`S_IFMT`-clear portion) of the resolved inode if the caller has the necessary privilege (file owner or `CAP_FOWNER`).

Critical for: every modern libc `fchmodat(.., AT_SYMLINK_NOFOLLOW)`, every container runtime adjusting symlink modes (where the previous syscall silently dereferenced the link), every Rookery security-test exercise, every package-manager `install -m` that wants to operate on a symlink, every `chmod -h` equivalent.

## Signature

C (Linux-specific; matches glibc's wishful `fchmodat` prototype):

```c
int fchmodat2(int dirfd, const char *pathname, mode_t mode, int flags);
```

glibc wrapper: `fchmodat2` (preferred backend for `fchmodat(.., flags)` since glibc 2.39) → `INLINE_SYSCALL(fchmodat2, 4, dirfd, pathname, mode, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE4(fchmodat2,
                int, dfd, const char __user *, filename,
                umode_t, mode, unsigned int, flags);
```

Rookery dispatch:

```rust
pub fn sys_fchmodat2(
    dirfd: i32,
    pathname: UserPtr<u8>,
    mode: u16,
    flags: u32,
) -> SyscallResult<i32>;
```

## Parameters

| name      | type                  | constraints                                                                | errno-on-bad |
|-----------|-----------------------|----------------------------------------------------------------------------|--------------|
| dirfd     | `int`                 | open directory fd, `AT_FDCWD`, or any fd with `AT_EMPTY_PATH`              | `EBADF` / `ENOTDIR` |
| pathname  | `const char __user *` | NUL-terminated; `< PATH_MAX`; may be `""` with `AT_EMPTY_PATH`             | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| mode      | `mode_t (u16)`        | within `07777` (`S_ISUID | S_ISGID | S_ISVTX | 0777`); higher bits stripped | (silently stripped) |
| flags     | `unsigned int`        | subset of `{AT_SYMLINK_NOFOLLOW, AT_EMPTY_PATH}`                            | `EINVAL` |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno; inode mode unchanged.

## Errors

| errno         | condition                                                                              |
|---------------|----------------------------------------------------------------------------------------|
| `EACCES`      | Search permission denied on a path component.                                          |
| `EBADF`       | `dirfd` is not open and not `AT_FDCWD`.                                                |
| `EFAULT`      | `pathname` outside user address space.                                                 |
| `EINVAL`      | `flags` contains a bit outside `AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH`.                  |
| `EIO`         | Filesystem I/O error during inode update.                                              |
| `ELOOP`       | Too many symlinks in resolution.                                                       |
| `ENAMETOOLONG`| Path too long.                                                                         |
| `ENOENT`      | Component missing; empty `pathname` without `AT_EMPTY_PATH`.                           |
| `ENOMEM`      | Allocation failed.                                                                     |
| `ENOTDIR`     | Non-final component not a directory.                                                   |
| `EOPNOTSUPP`  | `AT_SYMLINK_NOFOLLOW` requested but fs cannot store symlink mode (most fs).            |
| `EPERM`       | Caller is not the file owner and lacks `CAP_FOWNER` in the file's user namespace.      |
| `EROFS`       | Read-only filesystem.                                                                  |

## ABI surface (constants + flags)

`flags` (`include/uapi/linux/fcntl.h`):

- `AT_SYMLINK_NOFOLLOW = 0x100` — do not dereference terminal symlink; chmod the link inode itself. **Most Linux filesystems do not store independent symlink modes** and return `-EOPNOTSUPP` for this case — but exposing the flag lets userspace make the request explicitly rather than silently falling through to the target.
- `AT_EMPTY_PATH = 0x1000` — if `pathname == ""`, operate on `dirfd` directly (including `O_PATH` fds, subject to ownership/`CAP_FOWNER`).

`mode` permission bits (`include/uapi/linux/stat.h`):

- `S_ISUID = 04000`, `S_ISGID = 02000`, `S_ISVTX = 01000` — setuid / setgid / sticky.
- `S_IRWXU = 00700`, `S_IRWXG = 00070`, `S_IRWXO = 00007` — owner/group/other rwx.
- `S_IALLUGO = 07777` — full chmod mask. Bits outside this mask are silently stripped.

`AT_FDCWD = -100`.

Related kernel symbols:

- `vfs_fchmod` / `chmod_common(&path, mode)` — VFS chmod entry; calls LSM, `notify_change`, `i_op->setattr`.
- `setattr_prepare` — DAC permission/`CAP_FOWNER` check, returns `-EPERM` on mismatch.
- `notify_change(idmap, dentry, attr, &delegated_inode)` — actual inode-attr update path.
- `inode_owner_or_capable(idmap, inode)` — owner-or-CAP_FOWNER predicate.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=dirfd (i32)`, `%rsi=pathname`, `%rdx=mode (u16; widened)`, `%r10=flags (u32)`.
- REQ-2: `flags & ~(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH) ⟹ -EINVAL`.
- REQ-3: `mode & ~S_IALLUGO` ⟹ silently stripped (matches `fchmod(2)`).
- REQ-4: `getname_flags(pathname, lookup_flags | (AT_EMPTY_PATH ? LOOKUP_EMPTY : 0))` ⟹ `-EFAULT/-ENAMETOOLONG/-ENOENT`.
- REQ-5: Build lookup flags:
  - `LOOKUP_FOLLOW` set unless `flags & AT_SYMLINK_NOFOLLOW`.
  - `LOOKUP_EMPTY` set iff `flags & AT_EMPTY_PATH`.
- REQ-6: `filename_lookup(dirfd, pathname, lookup_flags)` ⟹ `struct path` for the target.
- REQ-7: `mnt_want_write(path.mnt)` ⟹ `-EROFS` if RO mount.
- REQ-8: `chmod_common(&path, mode)`:
  - `setattr_prepare(idmap, dentry, attr={.ia_valid = ATTR_MODE, .ia_mode = (inode->i_mode & ~S_IALLUGO) | (mode & S_IALLUGO)})` — owner check.
  - If target is a symlink and fs does not support symlink chmod ⟹ `-EOPNOTSUPP`.
  - `security_inode_setattr(idmap, dentry, attr)`.
  - `notify_change(idmap, dentry, attr, &delegated_inode)`.
- REQ-9: NFSv4 delegation: `delegated_inode` set ⟹ `-EWOULDBLOCK` after `break_deleg_wait` retry.
- REQ-10: `mnt_drop_write(path.mnt)`; `path_put(&path)`.
- REQ-11: Setuid/setgid bit stripping: VFS does NOT strip them here; `chmod` is the canonical set-them syscall. (Contrast with `write`/`chown` which DO strip suid.)
- REQ-12: With `AT_EMPTY_PATH` and `pathname == ""`, operate on `dirfd`'s inode directly via `f_path` — caller must still own the inode or hold `CAP_FOWNER`.
- REQ-13: Mode update is atomic w.r.t. concurrent stat/access from sibling threads (under `inode_lock`).

## Acceptance Criteria

- [ ] AC-1: `fchmodat2(AT_FDCWD, "f", 0644, 0) == 0` and `stat("f").st_mode & 07777 == 0644`.
- [ ] AC-2: `fchmodat2(AT_FDCWD, "f", 0644, 0xFFFF) == -EINVAL`.
- [ ] AC-3: `fchmodat2(.., AT_SYMLINK_NOFOLLOW)` on most fs ⟹ `-EOPNOTSUPP`; the symlink target is NOT modified.
- [ ] AC-4: `fchmodat2(fd, "", 0644, AT_EMPTY_PATH) == 0` (caller is owner).
- [ ] AC-5: `fchmodat2(fd, "", 0644, AT_EMPTY_PATH)` by non-owner without `CAP_FOWNER` ⟹ `-EPERM`.
- [ ] AC-6: `fchmodat2(AT_FDCWD, "", 0644, 0) == -ENOENT` (empty without `AT_EMPTY_PATH`).
- [ ] AC-7: `fchmodat2(AT_FDCWD, "f", 0644, 0)` on `/mnt/ro/f` (RO mount) ⟹ `-EROFS`.
- [ ] AC-8: Mode bits above `07777` are silently stripped; lower bits applied as supplied.
- [ ] AC-9: `fchmodat2(closed_fd, "f", 0644, 0) == -EBADF`.
- [ ] AC-10: Concurrent `stat` observes either old or new mode, never a torn value.

## Architecture

```
struct Fchmodat2Args { dirfd: i32, pathname: UserPtr<u8>, mode: u16, flags: u32 }
```

`sys_fchmodat2(args) -> i32`:

1. If `args.flags & !(AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH)` ⟹ return `-EINVAL`.
2. Return `do_fchmodat2(args.dirfd, args.pathname, args.mode, args.flags)`.

`Fs::do_fchmodat2(dfd, name_uptr, mode, flags) -> i32`:

1. `let name = getname_flags(name_uptr, if flags & AT_EMPTY_PATH { LOOKUP_EMPTY } else { 0 })?;`
2. `let lookup = if flags & AT_SYMLINK_NOFOLLOW { 0 } else { LOOKUP_FOLLOW } | if flags & AT_EMPTY_PATH { LOOKUP_EMPTY } else { 0 };`
3. `let path = filename_lookup(dfd, name, lookup)?;`
4. `mnt_want_write(&path.mnt)?;`
5. `let res = chmod_common(&path, mode & S_IALLUGO);`
6. `mnt_drop_write(&path.mnt);`
7. `path_put(&path);`
8. Return `res`.

`Fs::chmod_common(path, mode) -> i32`:

1. `let inode = path.dentry.inode;`
2. `let attr = IAttr { ia_valid: ATTR_MODE | ATTR_CTIME, ia_mode: (inode.i_mode & !S_IALLUGO) | (mode & S_IALLUGO) };`
3. `inode_lock(&inode);`
4. `let mut delegated = None;`
5. `let res = setattr_prepare(idmap, &path.dentry, &attr).and_then(|()| notify_change(idmap, &path.dentry, &attr, &mut delegated));`
6. `inode_unlock(&inode);`
7. If `delegated.is_some()` ⟹ `break_deleg_wait(&delegated); return -EWOULDBLOCK`.
8. Return `res`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_strict` | INVARIANT | Any bit outside `AT_SYMLINK_NOFOLLOW | AT_EMPTY_PATH` ⟹ `-EINVAL`. |
| `mode_masked` | INVARIANT | `attr.ia_mode == (old_mode & !S_IALLUGO) | (mode & S_IALLUGO)`. |
| `owner_or_cap_fowner` | INVARIANT | `setattr_prepare ⟹ caller owns inode || CAP_FOWNER`. |
| `at_empty_path_requires_flag` | INVARIANT | `name == "" && !(flags & AT_EMPTY_PATH) ⟹ -ENOENT`. |
| `symlink_nofollow_explicit` | INVARIANT | Without `AT_SYMLINK_NOFOLLOW`, terminal symlink is followed (legacy `fchmodat` semantics). |
| `atomic_mode_update` | INVARIANT | Mode change under `inode_lock`. |

### Layer 2: TLA+

`uapi/syscalls/fchmodat2.tla`:
- Per-call → validate → lookup → mnt_want_write → chmod_common → mnt_drop_write → path_put.
- Properties:
  - `safety_mode_within_S_IALLUGO`,
  - `safety_owner_or_cap_required`,
  - `liveness_fchmodat2_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ `inode.i_mode & S_IALLUGO == mode & S_IALLUGO` | `Fs::chmod_common` |
| Post: error ⟹ `inode.i_mode` unchanged | `Fs::chmod_common` |
| Post: `inode.i_ctime` updated on success | `Fs::chmod_common` |

### Layer 4: Verus/Creusot functional

`fchmodat2(dirfd, path, mode, flags)` ≡ Linux fchmodat2(2) per `man 2 fchmodat2`, with the `flags` argument semantics finally honored by the kernel rather than silently dropped.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fchmodat2(2)` reinforcement:

- **Per-flag-strict mask** — defense against silent forward-compat acceptance.
- **Per-`AT_SYMLINK_NOFOLLOW` explicit** — defense against the historical glibc-vs-kernel mismatch silently following symlinks.
- **Per-`AT_EMPTY_PATH` ownership gate** — defense against chmod-via-foreign-fd.
- **Per-`mode & S_IALLUGO` mask** — defense against caller smuggling non-mode bits.
- **Per-`mnt_want_write` / `mnt_drop_write`** — defense against RO-mount bypass and rare unmount-races.
- **Per-`delegated_inode` break** — defense against NFSv4 inconsistent state.
- **Per-`inode_lock`** — defense against torn mode reads by concurrent `stat`.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `pathname` SMAP-guarded in `getname`.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd fchmodat2 cannot escape via dirfd inherited pre-chroot.
- **GRKERNSEC_LINK** — chmod on hardlinks in protected dirs gated on owner.
- **GRKERNSEC_FIFO** — FIFO mode-bits in sticky dirs gated.
- **GRKERNSEC_SYMLINKOWN** — fchmodat2 with `AT_SYMLINK_NOFOLLOW` honors owner-match on the symlink itself.
- **AT_EMPTY_PATH / AT_SYMLINK_NOFOLLOW reject** — when grsec policy denies cross-owner fd chmod or symlink mode changes, returns `-EPERM` before any inode lookup.
- **GRKERNSEC_DMESG** — chmod-error printks CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_REFCOUNT** — dentry/inode refcounts saturating across the chmod.
- **PAX_MEMORY_SANITIZE** — freed `iattr` stack scrubbed on return.
- **PAX_RANDKSTACK** — kstack offset randomized at fchmodat2 syscall entry.
- **GRKERNSEC_SETUID** — setuid/setgid bits set via chmod auditable and subject to mount-noexec policy.
- **GRKERNSEC_AUDIT_CHDIR** — fchmodat2 across chroot boundary auditable.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `fchmod(2)` (fd-only, no path; separate Tier-5 if expanded).
- `fchmodat(2)` syscall 268 (legacy; lacks `flags` arg — superseded by fchmodat2).
- `chmod(2)` legacy 2-arg wrapper.
- Filesystem-specific `i_op->setattr` (covered in fs/<fstype> Tier-3).
- Implementation code.
