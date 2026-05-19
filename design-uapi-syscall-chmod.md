---
title: "Tier-5 syscall: chmod(2) — syscall 90"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`chmod(2)` changes the file mode bits (the rwx permission triplet plus
setuid/setgid/sticky) of the file identified by `path`. Symbolic links
are dereferenced; the mode of the target is changed, never of the link
itself (Linux does not support changing link modes, since link mode bits
are unused). The new mode replaces the existing mode bits (the upper
inode-type bits S_IFMT are preserved).

Permission to chmod is restricted to the owner (effective uid equals
inode uid) or a holder of `CAP_FOWNER` in the inode's user_ns. Setting
the setgid bit additionally requires that the caller's group set contain
the file's gid (or `CAP_FSETID`); otherwise the setgid bit is silently
cleared. Setting any sticky bit on a non-directory likewise requires
filesystem permission.

`chmod(2)` is the path-based wrapper around `chmod_common`, the shared
helper also used by `fchmod(2)` and `fchmodat(2)`. All three converge
on `notify_change(dentry, ATTR_MODE)` which invokes the filesystem's
`inode->setattr` (or `simple_setattr` for tmpfs-style FS).

Critical for: package installers (`install -m 0755`), shell `chmod`,
setuid-binary creation, lock-down hardening scripts.

### Acceptance Criteria

- [ ] AC-1: `chmod("/tmp/f", 0644)` on owned regular file: succeeds; `stat` shows mode 0100644.
- [ ] AC-2: `chmod("/etc/shadow", 0777)` as non-root: `-EPERM`.
- [ ] AC-3: `chmod("/nonexistent", 0644)`: `-ENOENT`.
- [ ] AC-4: `chmod` on symlink: dereferences; target's mode changes.
- [ ] AC-5: `chmod(path, 04777)` on file in group caller does not have, no CAP_FSETID: setgid bit cleared; final mode `00777`.
- [ ] AC-6: `chmod(path, 02755)` with matching group: setgid preserved.
- [ ] AC-7: `chmod(path, 0644)` on read-only FS: `-EROFS`.
- [ ] AC-8: `chmod` on immutable file: `-EPERM` (no CAP_LINUX_IMMUTABLE).
- [ ] AC-9: ctime updated even when new mode == old mode.
- [ ] AC-10: `FS_ATTRIB` inotify event observed by watchers.
- [ ] AC-11: Audit record contains old + new mode.
- [ ] AC-12: Upper bits of `mode` argument (e.g. 0x40000) silently dropped.

### Architecture

```rust
#[syscall(nr = 90, abi = "sysv")]
pub fn sys_chmod(path: UserPtr<u8>, mode: u16) -> isize {
    Fs::do_fchmodat(AT_FDCWD, path, mode, 0)
}
```

`Fs::do_fchmodat(dfd, path, mode, flags) -> isize`:
1. let name = getname(path)?;                                /* EFAULT, ENAMETOOLONG */
2. let p = user_path_at(dfd, &name, LOOKUP_FOLLOW)?;
3. let r = Fs::chmod_common(&p, mode & 0o7777);
4. path_put(p); putname(name);
5. r

`Fs::chmod_common(path, mode) -> isize`:
1. let inode = path.dentry.d_inode();
2. let mnt_w = mnt_want_write(path.mnt)?;                    /* EROFS */
3. let iattr = Iattr { ia_valid: ATTR_MODE | ATTR_CTIME, ia_mode: (inode.mode & S_IFMT) | (mode & 0o7777), ia_ctime: now() };
4. /* Setgid restriction */
5. if (iattr.ia_mode & S_ISGID) && !in_group_p(inode.gid) && !capable_wrt(inode, CAP_FSETID) { iattr.ia_mode &= !S_ISGID; }
6. /* LSM */
7. security_inode_setattr(path.dentry, &iattr)?;
8. inode_lock(inode);
9. let r = notify_change(path.mnt_idmap, path.dentry, &iattr, &delegated);
10. inode_unlock(inode);
11. mnt_drop_write(mnt_w);
12. /* fsnotify */
13. if r == 0 { fsnotify_change(path.dentry, FS_ATTRIB); }
14. r

`Fs::check_owner(inode) -> bool`:
1. current.fsuid == inode.uid OR capable(CAP_FOWNER in inode.user_ns)

### Out of Scope

- `fchmod(2)` (separate Tier-5 — fd-based variant).
- `fchmodat(2)` (separate Tier-5 — at-relative variant with AT_SYMLINK_NOFOLLOW).
- `chown(2)` (separate Tier-5 — ownership counterpart).
- POSIX ACL `setxattr(system.posix_acl_access)` (separate Tier-5).
- Implementation code.

### signature

```c
int chmod(const char *path, mode_t mode);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | NUL-terminated path string. Symlinks followed. |
| `mode` | `mode_t` | in | New mode bits — low 12 bits used: setuid (04000), setgid (02000), sticky (01000), rwxrwxrwx (0777). Higher bits silently ignored. |

### return value

| Value | Meaning |
|---|---|
| `0` | mode updated. |
| `-1` + `errno` | Failure; mode unchanged. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `path` outside caller's address space. |
| `ENAMETOOLONG` | `path` exceeds PATH_MAX or component exceeds NAME_MAX. |
| `ENOENT` | `path` does not exist. |
| `ENOTDIR` | A component of `path` is not a directory. |
| `EACCES` | Search permission denied on a component. |
| `ELOOP` | Symlink chain too long (>40). |
| `EPERM` | Caller is not the file owner and lacks `CAP_FOWNER`. |
| `EROFS` | Target is on a read-only filesystem. |
| `EIO` | I/O error writing inode metadata. |
| `ENOMEM` | Out of memory during lookup. |
| `EBADF` | (Internal) — not returned by chmod (relevant to fchmod). |

### abi surface

```text
__NR_chmod  (x86_64) = 90
__NR_chmod  (i386)   = 15
__NR_chmod  (arm64)  — n/a, removed; use fchmodat (268)
__NR_chmod  (riscv)  — n/a, removed; use fchmodat
```

Note: arm64 / riscv expose only `fchmodat(2)`; glibc emulates `chmod()`
on those architectures via `fchmodat(AT_FDCWD, path, mode, 0)`.

### compatibility contract

REQ-1: Syscall number is **90** on x86_64. ABI-stable since v1.

REQ-2: Per-path resolution: `user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW)`
— symlinks dereferenced; final inode targeted.

REQ-3: Per-mode masking: only low 12 bits used (`mode & 07777`); upper
bits silently ignored — including `S_IFMT`. Mode bits NEVER change file
type.

REQ-4: Per-owner-check: `current.fsuid == inode.uid` OR
`CAP_FOWNER` in inode's user_ns. Else `-EPERM`.

REQ-5: Per-setgid-restrict: if `mode & S_ISGID` and inode.gid not in
caller's group set, and caller lacks `CAP_FSETID`, the setgid bit is
silently cleared (not -EPERM). Per kernel commit `c5dc7a9a` semantics.

REQ-6: Per-suid-clear-on-chmod: chmod itself does NOT clear suid/sgid
(that happens on chown/write of file data). chmod sets exactly what the
caller requested (modulo REQ-5 masking).

REQ-7: Per-RO-fs: filesystem mounted MS_RDONLY → `-EROFS` from
`mnt_want_write`.

REQ-8: Per-immutable: inode with `S_IMMUTABLE` attribute → `-EPERM`
even for root (modulo `CAP_LINUX_IMMUTABLE`).

REQ-9: Per-notify_change: filesystem's `inode_operations->setattr`
invoked with `iattr { ia_valid = ATTR_MODE | ATTR_CTIME, ia_mode, ia_ctime }`.

REQ-10: Per-ctime-update: inode ctime touched even if mode value unchanged
(POSIX behavior — chmod implies metadata "modified").

REQ-11: Per-inotify/fsnotify: `FS_ATTRIB` event fired post-success.

REQ-12: Per-audit: `AUDIT_SYSCALL` records syscall, path, old mode,
new mode, success/errno.

REQ-13: Per-LSM: `security_inode_setattr` invoked pre-change; SELinux,
Smack, AppArmor may veto with `-EACCES` or `-EPERM`.

REQ-14: Per-mount-uid/gid mapping: if mounted with `uidmap`/`gidmap`,
ownership check uses mapped IDs.

REQ-15: Per-tmpfs / FUSE: `simple_setattr` or FUSE `setattr` op called;
same iattr semantics.

REQ-16: Per-CHMOD-on-bind-mount: chmod applies to the underlying inode;
all bind-mounts observe the change.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mode_masked_07777` | INVARIANT | Only low 12 bits of `mode` reach setattr. |
| `setgid_drops_if_group_mismatch` | INVARIANT | No CAP_FSETID + non-member ⟹ S_ISGID cleared. |
| `owner_check_before_setattr` | ORDER | EPERM returned BEFORE notify_change. |
| `mnt_write_dropped` | INVARIANT | mnt_want_write balanced by mnt_drop_write. |
| `ctime_updated` | INVARIANT | ctime touched on success. |

### Layer 2: TLA+

`fs/chmod.tla`:
- States: LOOKUP → OWNER_CHECK → MNT_WRITE → SETGID_FILTER → LSM → SETATTR → FSNOTIFY.
- Properties:
  - `safety_owner_or_capable` — chmod only by owner or CAP_FOWNER.
  - `safety_setgid_drop` — setgid silently dropped per REQ-5.
  - `safety_mode_masked` — high bits never persisted.
  - `liveness_chmod_terminates` — every chmod returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fchmodat` post: inode.mode low-12 == arg & 07777 (modulo setgid filter) | `Fs::do_fchmodat` |
| `chmod_common` post: ctime updated on 0 ret | `Fs::chmod_common` |
| `setgid_filter` post: per-REQ-5 holds | `Fs::chmod_common` |

### Layer 4: Verus / Creusot functional

Per-`chmod(2)` man-page, per-POSIX, per-Linux fs/open.c, LTP `chmod01..07`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`chmod(2)` reinforcement:

- **Per-getname PAX_USERCOPY** — defense against per-user-pointer overflow.
- **Per-LOOKUP_FOLLOW + mnt_want_write** — defense against per-RO-fs bypass.
- **Per-owner / CAP_FOWNER strict** — defense against per-DAC bypass.
- **Per-setgid silent-drop** — defense against per-stealthy-sgid escalation.
- **Per-LSM hook setattr** — defense against per-MAC bypass.
- **Per-notify_change under inode_lock** — defense against per-mode-tearing.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `path`** — `getname` runs with SMAP+UDEREF on; the
  user pointer cannot point into kernel memory; `path` always copied
  through the user-segment.
- **CAP_FOWNER strict** — gradm RBAC may forbid `CAP_FOWNER` from being
  granted to non-policy subjects; default policy denies setuid-binary
  creation to non-system roles regardless of ambient capability set.
- **GRKERNSEC_SUIDDUMP on suid-mode-change** — when `chmod` adds or
  removes `S_ISUID` / `S_ISGID` on a regular file, the change is logged
  with sender uid, target path, old mode, new mode. Coredump suppression
  for the affected file recomputed (coredump-of-newly-suid file inherits
  the suppression flag).
- **GRKERNSEC_CHROOT_CHMOD** — a chrooted process cannot use `chmod` to
  set `S_ISUID` or `S_ISGID` on any file inside its chroot; the bit is
  forcibly cleared from the iattr before `notify_change`. Closes the
  "chroot-escape-via-suid-binary" pattern.
- **GRKERNSEC_TPE on setuid-mode set** — Trusted Path Execution: if TPE
  is enabled, a `chmod` that sets `S_ISUID` on a binary whose directory
  is group-writable or world-writable returns `-EPERM`. Combined with
  TPE_GID, only the trusted group may chmod-suid binaries.
- **File-cap-strip on chown (cross-reference)** — `chmod` does not strip
  file capabilities by itself, but a subsequent `chown` will via the
  `setattr_should_drop_suidgid` path. Grsec logs the cap-strip event
  immediately and rate-limits per-uid.
- **PaX MEMORY_SANITIZE on Iattr** — `iattr` struct zero-initialized so
  unused arms (ATTR_UID/GID) are not leaked into the filesystem driver.
- **GRKERNSEC_LINK on symlink chmod** — `chmod` through a symlink owned
  by a different uid is rejected per gr_acl_handle_symlink_owner unless
  the destination is owned by the same uid as the symlink.
- **PaX KERNEXEC** — `chmod_common` and `notify_change` in read-only-
  after-init kernel text; cannot be patched to skip owner check or
  setgid filter.
- **GRKERNSEC_DMESG** — chmod permission failures rate-limited and
  CAP_SYSLOG-gated in kernel ring-buffer output.
- **GRKERNSEC_HIDESYM in chmod klog** — kernel pointers (inode, dentry)
  stripped from any log message.
- **PaX RANDSTRUCT on `struct iattr`** — randomized field layout; memory
  corruption primitives cannot deterministically overwrite `ia_mode` to
  smuggle a setuid bit through the LSM hook.
- **gradm policy: chmod-suid REQUIRES policy** — under RBAC, the right
  to chmod-add S_ISUID/S_ISGID is a discrete capability (`O` mode on
  policy object); subjects without it cannot create new suid binaries
  even as root.

