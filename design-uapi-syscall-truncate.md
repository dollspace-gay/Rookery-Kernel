---
title: "Tier-5 syscall: truncate(2) — syscall 76"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`truncate(2)` resizes the regular file at `path` to exactly `length`
bytes. Shrink discards data beyond the new end (page cache invalidated);
grow extends the file with a sparse hole that reads as zeros (no blocks
allocated until written).

The file must be writable by the caller in the conventional DAC sense
(or `CAP_DAC_OVERRIDE`). Unlike `chmod`, write permission on the file
content is what is checked — not ownership. Truncate on a non-regular
file: directories `EISDIR`; symlinks dereferenced; FIFOs/sockets `EINVAL`.

As a security side-effect, a non-CAP_FSETID truncate of a setuid binary
clears the setuid bit (and setgid for executables) via
`setattr_should_drop_suidgid` — same rule as chown. Closes the
"truncate-then-rewrite-suid-binary" race.

`RLIMIT_FSIZE` is enforced on grow: truncate that would exceed
RLIMIT_FSIZE returns `-EFBIG` and delivers `SIGXFSZ`.

Critical for: log rotation, atomic image-file creation, sparse-extension
preallocation, container image copy-up.

### Acceptance Criteria

- [ ] AC-1: `truncate("f", 100)` on 50-byte file: file grows to 100, last 50 bytes are zero (sparse hole).
- [ ] AC-2: `truncate("f", 100)` on 200-byte file: file shrinks; bytes 100..199 discarded.
- [ ] AC-3: `truncate("f", -1)`: `-EINVAL`.
- [ ] AC-4: `truncate("/dir", 0)`: `-EISDIR`.
- [ ] AC-5: truncate of fifo: `-EINVAL`.
- [ ] AC-6: truncate of file not writable by caller: `-EACCES`.
- [ ] AC-7: truncate on RO-fs: `-EROFS`.
- [ ] AC-8: truncate on immutable file: `-EPERM`.
- [ ] AC-9: truncate on append-only file: `-EPERM`.
- [ ] AC-10: truncate of running executable (text-busy): `-ETXTBSY`.
- [ ] AC-11: truncate that grows beyond RLIMIT_FSIZE: `-EFBIG` + SIGXFSZ delivered.
- [ ] AC-12: truncate of suid binary by non-CAP_FSETID: setuid cleared.
- [ ] AC-13: mtime + ctime updated; page cache pages beyond new length invalidated.
- [ ] AC-14: mmap'd page beyond new length: SIGBUS on access.

### Architecture

```rust
#[syscall(nr = 76, abi = "sysv")]
pub fn sys_truncate(path: UserPtr<u8>, length: i64) -> isize {
    Fs::do_sys_truncate(path, length, AT_FDCWD)
}
```

`Fs::do_sys_truncate(path, length, dfd) -> isize`:
1. if length < 0 { return -EINVAL; }
2. let p = user_path_at(dfd, getname(path)?, LOOKUP_FOLLOW)?;
3. let inode = p.dentry.d_inode();
4. if !inode.is_reg() { return if inode.is_dir() { -EISDIR } else { -EINVAL }; }
5. let mnt_w = mnt_want_write(p.mnt)?;
6. inode_permission(inode, MAY_WRITE)?;                       /* EACCES */
7. if IS_APPEND(inode) || IS_IMMUTABLE(inode) { return -EPERM; }
8. deny_write_access(inode)?;                                 /* ETXTBSY */
9. if length > inode.size && length > current.rlim[RLIMIT_FSIZE].rlim_cur && !capable(CAP_SYS_RESOURCE) { send_sig(SIGXFSZ, current, 0); return -EFBIG; }
10. let mut ia = Iattr { ia_valid: ATTR_SIZE | ATTR_CTIME | ATTR_MTIME, ia_size: length, ia_ctime: now(), ia_mtime: now(), .. Default::default() };
11. if setattr_should_drop_suidgid(inode, &ia) { ia.ia_valid |= ATTR_KILL_SUID; if inode.mode & S_IXGRP != 0 { ia.ia_valid |= ATTR_KILL_SGID; } }
12. security_path_truncate(&p)?; security_inode_setattr(p.dentry, &ia)?;
13. inode_lock(inode); let r = Fs::do_truncate(p.mnt_idmap, p.dentry, length, ia.ia_valid, NULL); inode_unlock(inode);
14. allow_write_access(inode); mnt_drop_write(mnt_w);
15. r

`Fs::do_truncate(idmap, dentry, length, attrs, file)`: notify_change(...)
then `vmtruncate` / `truncate_pagecache_range` / `truncate_inode_pages`.

### Out of Scope

- `ftruncate(2)` / `truncate64(2)` / `fallocate(2)` / `open(O_TRUNC)` (separate Tier-5 docs).
- Implementation code.

### signature

```c
int truncate(const char *path, off_t length);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | Path to regular file. Symlinks followed. |
| `length` | `off_t` | in | New file length in bytes. Must be `>= 0`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Truncated successfully. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EFAULT` | `path` outside caller's address space. |
| `EACCES` | Write permission denied on file content, or search denied on a path component. |
| `EFBIG` | `length` exceeds filesystem max file size OR exceeds caller's `RLIMIT_FSIZE`. |
| `EINTR` | Wait for in-flight I/O interrupted by signal. |
| `EINVAL` | `length` < 0; or filesystem does not support truncate on this inode type. |
| `EIO`  | Low-level I/O error. |
| `EISDIR` | `path` is a directory. |
| `ELOOP` | Symlink chain > 40. |
| `ENAMETOOLONG` | Path too long. |
| `ENOENT` | Path does not exist. |
| `ENOTDIR` | Component not a directory. |
| `EROFS` | Read-only filesystem. |
| `ETXTBSY` | File is an executable currently being executed (busy text segment). |
| `EPERM` | Append-only (`S_APPEND`) or immutable (`S_IMMUTABLE`) attribute. |

### abi surface

```text
__NR_truncate (x86_64)  = 76
__NR_truncate (i386)    = 92    /* + truncate64 for off64 */
__NR_truncate (arm64)   = 45
__NR_truncate (riscv)   = 45
__NR_truncate (generic) = 45
```

On 32-bit architectures, `truncate64(2)` (separate syscall) handles
64-bit offsets; on 64-bit architectures `truncate(2)` already accepts
64-bit `off_t`.

### compatibility contract

REQ-1: Syscall number is **76** on x86_64; **45** on generic. ABI-stable.

REQ-2: Per-path resolution: `user_path_at(AT_FDCWD, path, LOOKUP_FOLLOW)`.
Symlinks dereferenced.

REQ-3: Per-length-non-negative: `length < 0` ⟹ `-EINVAL`.

REQ-4: Per-file-type: regular file only. Directory: `-EISDIR`. Special
file (FIFO, socket, device): `-EINVAL`, filesystem-dependent.

REQ-5: Per-DAC: caller must have write (`MAY_WRITE`) on the inode. NOT
owner check; write permission alone is sufficient.

REQ-6: Per-RO-fs: `mnt_want_write` ⟹ `-EROFS`.

REQ-7: Per-immutable/append-only: `S_IMMUTABLE` ⟹ `-EPERM`;
`S_APPEND` ⟹ `-EPERM` (cannot shrink an append-only file).

REQ-8: Per-text-busy: file is text segment of a running executable ⟹
`-ETXTBSY` from `deny_write_access`.

REQ-9: Per-RLIMIT_FSIZE on grow: `length > current_size` AND
`length > rlim[RLIMIT_FSIZE].rlim_cur` AND no `CAP_SYS_RESOURCE` ⟹
`-EFBIG` AND deliver `SIGXFSZ`.

REQ-10: Per-suid/sgid clear: non-CAP_FSETID caller ⟹ ATTR_KILL_SUID
(and ATTR_KILL_SGID for executables) via `setattr_should_drop_suidgid`.

REQ-11: Per-mtime/ctime: BOTH updated on success.

REQ-12: Per-LSM: `security_path_truncate` and `security_inode_setattr`
hooks invoked.

REQ-13: Per-page-cache: shrink ⟹ `truncate_pagecache(inode, new_size)`
invalidates pages beyond new_size, dropping dirty bits.

REQ-14: Per-block-allocation: shrink frees blocks beyond new EOF; grow
creates a sparse hole. Per-mmap: pages beyond new length remain in
process VAS but SIGBUS on access (POSIX).

REQ-15: Per-fsnotify: `FS_MODIFY` on inode. Per-audit: AUDIT_SYSCALL
records path and new length. Per-notify_change with
`ATTR_SIZE | ATTR_CTIME | ATTR_MTIME`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `length_non_negative` | INVARIANT | length < 0 ⟹ -EINVAL. |
| `regular_file_only` | INVARIANT | non-reg ⟹ EISDIR or EINVAL. |
| `dac_write_check` | INVARIANT | EACCES if MAY_WRITE denied. |
| `append_immutable_eperm` | INVARIANT | IS_APPEND or IS_IMMUTABLE ⟹ EPERM. |
| `rlimit_fsize_on_grow` | INVARIANT | grow > RLIMIT_FSIZE ⟹ EFBIG + SIGXFSZ. |
| `suid_cleared_no_fsetid` | INVARIANT | non-CAP_FSETID ⟹ ATTR_KILL_SUID. |
| `pagecache_invalidated_on_shrink` | INVARIANT | new < old ⟹ truncate_pagecache. |
| `mtime_ctime_updated` | INVARIANT | success ⟹ both touched. |

### Layer 2: TLA+

`fs/truncate.tla`:
- States: VALIDATE → LOOKUP → TYPE_CHECK → MNT_WRITE → DAC → APPEND/IMM → TEXT_BUSY → RLIMIT → BUILD_IATTR → SUID_FILTER → LSM → NOTIFY_CHANGE → PAGECACHE_TRUNC.
- Properties:
  - `safety_regular_only` — non-reg never reaches notify_change.
  - `safety_dac_write` — write permission strict.
  - `safety_append_imm` — never truncates immutable.
  - `safety_rlimit_grow` — RLIMIT_FSIZE enforced on grow.
  - `safety_suid_cleared` — non-CAP_FSETID truncate strips setuid.
  - `safety_pagecache_consistent` — pages beyond new length invalidated.
  - `liveness_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_truncate` post: inode.size == length on success | `sys_truncate` |
| `do_sys_truncate` post: length validated BEFORE iattr build | `Fs::do_sys_truncate` |
| `do_truncate` post: pagecache truncated on shrink | `Fs::do_truncate` |
| `setattr_should_drop_suidgid` post: ATTR_KILL_SUID set per REQ-10 | `setattr_should_drop_suidgid` |

### Layer 4: Verus / Creusot functional

Per-`truncate(2)` man-page, per-POSIX, per-Linux fs/open.c + fs/attr.c
+ mm/truncate.c, LTP `truncate01..04`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`truncate(2)` reinforcement:

- **Per-getname PAX_USERCOPY** — defense against per-string overflow.
- **Per-length-non-negative** — defense against per-signed-overflow on
  iattr build.
- **Per-regular-file-only** — defense against per-directory / per-special
  truncate confusion.
- **Per-DAC MAY_WRITE strict** — defense against per-content modification
  without write permission.
- **Per-mnt_want_write** — defense against per-RO-mount bypass.
- **Per-append/immutable check** — defense against per-attribute bypass.
- **Per-text-busy** — defense against per-running-binary truncation
  (the classic "modify code under execution" attack).
- **Per-RLIMIT_FSIZE on grow** — defense against per-disk-exhaustion
  via sparse-extension.
- **Per-suid/sgid drop** — defense against per-suid-preservation
  through content-replace.
- **Per-LSM hooks** — defense against per-MAC bypass.
- **Per-pagecache invalidation on shrink** — defense against per-stale-
  data leak.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `path`** — getname under SMAP+UDEREF.
- **CAP_FOWNER / CHOWN / SETID strict (cross-reference)** — suid-drop
  path consumes CAP_FSETID's absence.
- **GRKERNSEC_SUIDDUMP on suid-mode-change** — truncate of a setuid
  binary that triggers ATTR_KILL_SUID logs sender uid, target path,
  before/after mode. Coredump suppression flag recomputed.
- **GRKERNSEC_CHROOT_CHMOD (cross-reference)** — chroot-confined
  truncate's suid-strip permitted (it REMOVES privilege, never adds).
- **GRKERNSEC_TPE on setuid-mode set (cross-reference)** — N/A;
  truncate cannot ADD setuid bits.
- **File-cap-strip on chown (cross-reference)** — truncate also strips
  `security.capability` xattr per `setattr_should_drop_suidgid` for
  non-CAP_FSETID callers. Logged.
- **RLIMIT_FSIZE on truncate-grow ENFORCED** — `length > rlim_cur`
  delivers `-EFBIG` AND `SIGXFSZ` BEFORE any filesystem-side
  allocation. Grsec rate-limits SIGXFSZ delivery per-uid to prevent
  self-DoS-via-signal. CAP_SYS_RESOURCE escape policy-gated.
- **GRKERNSEC_LINK on symlink truncate** — truncate via a symlink owned
  by a different uid (sticky directory) denied per
  gr_acl_handle_symlink_owner unless RBAC grants explicit policy.
- **PaX MEMORY_SANITIZE on truncated pages** — pages beyond new EOF
  zeroed before release to page allocator. Closes cross-process
  page-reuse disclosure window.
- **GRKERNSEC_DMESG / GRKERNSEC_HIDESYM** — failure klog rate-limited,
  CAP_SYSLOG-gated; pointer leaks scrubbed.
- **PaX KERNEXEC** — `sys_truncate`, `do_truncate`, `notify_change`,
  `truncate_pagecache` in read-only-after-init text.
- **PaX RANDSTRUCT on `struct iattr`** — randomized layout; corruption
  cannot deterministically swap `ia_size`.
- **gradm policy: truncate REQUIRES write `w` mode** — RBAC may require
  explicit write-policy regardless of DAC MAY_WRITE; closes the
  "truncate as DoS on shared logs" pattern.
- **GRKERNSEC_HARDEN_PTRACE** — tracee-driven truncate via cwd retargeting
  requires PTRACE_MODE_ATTACH_FSCREDS.
- **PaX RANDKSTACK** — random kernel stack at sys_truncate entry;
  RLIMIT/SUID-drop control flow not stack-layout-targetable.

