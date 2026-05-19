---
title: "Tier-5 syscall: ftruncate(2) — syscall 77"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`ftruncate(2)` resizes the regular file referenced by fd `fd` to
exactly `length` bytes. It is the fd-based counterpart of `truncate(2)`:
same shrink/grow semantics, same `RLIMIT_FSIZE` enforcement, same
setuid/sgid stripping, same `security.capability` xattr removal for
executables.

Key difference from `truncate`: `ftruncate` REQUIRES the fd to have
been opened with WRITE access (`O_RDWR`/`O_WRONLY`). DAC check at
lookup time is replaced by an fd-mode check — `open()` already
validated write permission. `O_PATH` fds NOT accepted (no R/W mode).

`ftruncate` is TOCTOU-immune in the inode-binding sense: fd pinned the
inode at open; no path race can swap a different file. Preferred for
atomic create-then-resize flows (`memfd_create` + `ftruncate`,
`open(O_CREAT|O_RDWR)` + `ftruncate` for preallocated images).

Critical for: shared memory (`shm_open` → `ftruncate`), `memfd_create` +
ftruncate for sealed-memory IPC, image preallocation, log rotation via
stdout-fd, container layer build tools.

### Acceptance Criteria

- [ ] AC-1: `ftruncate(fd, 100)` on 50-byte file opened O_RDWR: grows to 100, sparse hole.
- [ ] AC-2: `ftruncate(fd, 30)` on 100-byte file: shrinks; bytes 30..99 discarded.
- [ ] AC-3: `ftruncate(-1, 0)`: `-EBADF`.
- [ ] AC-4: `ftruncate(fd_O_RDONLY, 0)`: `-EINVAL`.
- [ ] AC-5: `ftruncate(fd_O_PATH, 0)`: `-EBADF`.
- [ ] AC-6: `ftruncate(fd, -1)`: `-EINVAL`.
- [ ] AC-7: `ftruncate(fd_to_dir_O_RDWR, 0)`: `-EINVAL` (no truncate on directory).
- [ ] AC-8: ftruncate on RO-mount fd: `-EROFS`.
- [ ] AC-9: ftruncate on immutable file: `-EPERM`.
- [ ] AC-10: ftruncate on running-executable text fd: `-ETXTBSY`.
- [ ] AC-11: ftruncate grow beyond RLIMIT_FSIZE: `-EFBIG` + SIGXFSZ.
- [ ] AC-12: ftruncate of suid binary fd by non-CAP_FSETID: setuid cleared.
- [ ] AC-13: ftruncate on memfd: succeeds; mmap sees new size.
- [ ] AC-14: F_SEAL_GROW + grow ⟹ `-EPERM`; F_SEAL_SHRINK + shrink ⟹ `-EPERM`.
- [ ] AC-15: mtime + ctime updated.

### Architecture

```rust
#[syscall(nr = 77, abi = "sysv")]
pub fn sys_ftruncate(fd: c_int, length: i64) -> isize {
    Fs::do_sys_ftruncate(fd, length, true /* small_arg=true means off_t fits */)
}
```

`Fs::do_sys_ftruncate(fd, length) -> isize`:
1. if length < 0 { return -EINVAL; }
2. let f = fdget(fd)?;                                       /* EBADF */
3. if f.file.f_mode & FMODE_PATH != 0 { return -EBADF; }
4. if f.file.f_mode & FMODE_WRITE == 0 { return -EINVAL; }
5. let inode = f.file.f_inode;
6. if !inode.is_reg() { return -EINVAL; }
7. if let Some(seals) = memfd_get_seals(f.file) {
8.     if (seals & F_SEAL_GROW) != 0 && length > inode.size { return -EPERM; }
9.     if (seals & F_SEAL_SHRINK) != 0 && length < inode.size { return -EPERM; }
10. }
11. let mnt_w = mnt_want_write_file(f.file)?;
12. if IS_APPEND(inode) || IS_IMMUTABLE(inode) { return -EPERM; }
13. deny_write_access(inode)?;                                /* ETXTBSY */
14. if length > inode.size && length > current.rlim[RLIMIT_FSIZE].rlim_cur && !capable(CAP_SYS_RESOURCE) { send_sig(SIGXFSZ, current, 0); return -EFBIG; }
15. let mut ia = Iattr { ia_valid: ATTR_SIZE | ATTR_CTIME | ATTR_MTIME, ia_size: length, ia_ctime: now(), ia_mtime: now(), ia_file: Some(f.file), .. Default::default() };
16. if setattr_should_drop_suidgid(inode, &ia) { ia.ia_valid |= ATTR_KILL_SUID; if inode.mode & S_IXGRP != 0 { ia.ia_valid |= ATTR_KILL_SGID; } }
17. security_inode_setattr(f.file.f_path.dentry, &ia)?;
18. inode_lock(inode); let r = Fs::do_truncate(f.file.f_path.mnt_idmap, f.file.f_path.dentry, length, ia.ia_valid, Some(f.file)); inode_unlock(inode);
19. mnt_drop_write_file(mnt_w);
20. r

### Out of Scope

- `truncate(2)` / `ftruncate64(2)` / `fallocate(2)` / `memfd_create(2)` / `shm_open(3)` (separate Tier-5 docs).
- Implementation code.

### signature

```c
int ftruncate(int fd, off_t length);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor with write access. |
| `length` | `off_t` | in | New file length. Must be `>= 0`. |

### return value

| Value | Meaning |
|---|---|
| `0` | Truncated successfully. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` is not a valid file descriptor; or fd not opened for write. |
| `EINVAL` | `length` < 0; or fd cannot be truncated (not a regular file / not writable / O_PATH); or filesystem does not support truncate. |
| `EFBIG` | `length` exceeds filesystem max OR exceeds caller's `RLIMIT_FSIZE`. |
| `EINTR` | I/O wait interrupted by signal. |
| `EIO`  | I/O error. |
| `EPERM` | Append-only or immutable inode. |
| `EROFS` | Filesystem mounted read-only. |
| `ETXTBSY` | File is the text of a running executable. |

### abi surface

```text
__NR_ftruncate (x86_64)  = 77
__NR_ftruncate (i386)    = 93    /* + ftruncate64 for off64 */
__NR_ftruncate (arm64)   = 46
__NR_ftruncate (riscv)   = 46
__NR_ftruncate (generic) = 46
```

### compatibility contract

REQ-1: Syscall **77** on x86_64; **46** on generic. ABI-stable.

REQ-2: Per-fd: `fdget(fd)` → struct file. EBADF if invalid.

REQ-3: Per-write-mode: `f.file.f_mode & FMODE_WRITE` required. If
absent ⟹ `-EINVAL` (NOT EBADF — historical Linux behavior; POSIX
spec is `EINVAL`).

REQ-4: Per-O_PATH: `f.file.f_mode & FMODE_PATH` ⟹ `-EBADF`. O_PATH
fds carry no read/write mode.

REQ-5: Per-regular-file: `f.file.f_inode.is_reg()` required. Non-reg
⟹ `-EINVAL`. (Even with write access to a directory fd, ftruncate
fails — POSIX prohibits.)

REQ-6: Per-length-non-negative: `length < 0` ⟹ `-EINVAL`.

REQ-7: Per-RO-mount: `mnt_want_write_file(f.file)` ⟹ `-EROFS`. Uses
the fd's mount (relevant for bind-mount RO overlays).

REQ-8: Per-immutable/append-only: `S_IMMUTABLE` ⟹ `-EPERM` always.
`S_APPEND` ⟹ `-EPERM` (truncate cannot shrink an append-only file —
also cannot extend, by POSIX clarification).

REQ-9: Per-text-busy: file is text of a running executable ⟹
`-ETXTBSY`.

REQ-10: Per-RLIMIT_FSIZE on grow: `length > current_size` AND
`length > current.rlim[RLIMIT_FSIZE].rlim_cur` AND no CAP_SYS_RESOURCE
⟹ `-EFBIG` + deliver `SIGXFSZ`.

REQ-11: Per-suid/sgid clear: non-CAP_FSETID caller ⟹ ATTR_KILL_SUID
+ (executable) ATTR_KILL_SGID via `setattr_should_drop_suidgid`.

REQ-12: Per-mtime/ctime: BOTH updated on success.

REQ-13: Per-LSM: `security_file_truncate` and `security_inode_setattr`
hooks invoked.

REQ-14: Per-page-cache: shrink invalidates pages beyond new EOF; grow
leaves sparse hole. Per-mmap: pages beyond new EOF SIGBUS on access.
Per-block-allocation: shrink frees blocks; grow does not allocate.

REQ-16: Per-fsnotify: `FS_MODIFY`. Per-audit: AUDIT_SYSCALL records fd,
resolved path, new length. Per-notify_change with
`ATTR_SIZE | ATTR_CTIME | ATTR_MTIME` + optional ATTR_KILL_SUID/SGID.

REQ-17: Per-memfd: ftruncate on memfd succeeds; tmpfs-backed.

REQ-18: Per-sealed-memfd: `F_SEAL_GROW` + grow ⟹ `-EPERM`.
`F_SEAL_SHRINK` + shrink ⟹ `-EPERM`. Seals enforced here.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_valid_first` | ORDER | EBADF before inode deref. |
| `o_path_rejected` | INVARIANT | FMODE_PATH ⟹ -EBADF. |
| `write_mode_required` | INVARIANT | !FMODE_WRITE ⟹ -EINVAL. |
| `regular_file_only` | INVARIANT | !is_reg ⟹ -EINVAL. |
| `length_non_negative` | INVARIANT | length < 0 ⟹ -EINVAL. |
| `memfd_seal_grow` | INVARIANT | F_SEAL_GROW + grow ⟹ -EPERM. |
| `memfd_seal_shrink` | INVARIANT | F_SEAL_SHRINK + shrink ⟹ -EPERM. |
| `append_immutable_eperm` | INVARIANT | IS_APPEND/IMMUTABLE ⟹ -EPERM. |
| `rlimit_fsize_on_grow` | INVARIANT | grow > RLIMIT_FSIZE ⟹ -EFBIG + SIGXFSZ. |
| `suid_cleared_no_fsetid` | INVARIANT | non-CAP_FSETID ⟹ ATTR_KILL_SUID. |

### Layer 2: TLA+

`fs/ftruncate.tla`:
- States: FDGET → MODE_CHECK → TYPE_CHECK → SEAL_CHECK → MNT_WRITE → APPEND/IMM → TEXT_BUSY → RLIMIT → IATTR → SUID_FILTER → LSM → NOTIFY_CHANGE → PAGECACHE_TRUNC.
- Properties:
  - `safety_write_mode_strict` — !FMODE_WRITE never reaches notify_change.
  - `safety_regular_only` — non-reg never truncated.
  - `safety_seal_grow_shrink` — memfd seals enforced.
  - `safety_rlimit_grow` — RLIMIT_FSIZE strict on grow.
  - `safety_suid_cleared` — non-CAP_FSETID strips setuid.
  - `safety_no_toctou` — inode pinned by fd.
  - `liveness_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_ftruncate` post: inode.size == length on success | `sys_ftruncate` |
| `do_sys_ftruncate` post: O_PATH rejected | `Fs::do_sys_ftruncate` |
| `do_sys_ftruncate` post: !FMODE_WRITE ⟹ -EINVAL | `Fs::do_sys_ftruncate` |
| `do_sys_ftruncate` post: memfd seals enforced | `Fs::do_sys_ftruncate` |
| `do_truncate` post: pagecache truncated on shrink | `Fs::do_truncate` |

### Layer 4: Verus / Creusot functional

Per-`ftruncate(2)` man-page, per-POSIX, per-Linux fs/open.c + fs/attr.c
+ mm/truncate.c, LTP `ftruncate01..04`, memfd `tools/testing/selftests/memfd`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`ftruncate(2)` reinforcement:

- **Per-fdget atomic** — defense against per-fd-table race.
- **Per-O_PATH rejection** — defense against per-empty-mode confusion.
- **Per-FMODE_WRITE strict** — defense against per-read-only-fd truncate.
- **Per-regular-file-only** — defense against per-directory/special truncate.
- **Per-length-non-negative** — defense against per-signed-overflow.
- **Per-memfd seal enforcement** — defense against per-sealed-region bypass.
- **Per-mnt_want_write_file** — defense against per-RO-mount overlay bypass.
- **Per-append/immutable check** — defense against per-attribute bypass.
- **Per-text-busy** — defense against per-running-binary truncation.
- **Per-RLIMIT_FSIZE on grow** — defense against per-disk-exhaustion.
- **Per-suid/sgid drop + LSM hooks + TOCTOU immunity** — defense against
  per-suid-preservation, per-MAC bypass, per-rename race.

### grsecurity / pax-style reinforcement

- **PaX UDEREF** — ftruncate has no user pointer in its main path.
- **CAP_FOWNER / CHOWN / SETID strict (cross-reference)** — suid-drop
  path consumes CAP_FSETID's absence.
- **GRKERNSEC_SUIDDUMP on suid-mode-change** — ftruncate that triggers
  ATTR_KILL_SUID logs sender uid, fd-resolved path, before/after mode.
  Coredump suppression flag recomputed.
- **GRKERNSEC_CHROOT_CHMOD (cross-reference)** — chroot-confined
  ftruncate's suid-strip permitted (it REMOVES privilege).
- **GRKERNSEC_TPE on setuid-mode set (cross-reference)** — N/A;
  ftruncate cannot ADD setuid bits.
- **File-cap-strip on chown (cross-reference)** — ftruncate strips
  `security.capability` per setattr_should_drop_suidgid for
  non-CAP_FSETID callers. Logged.
- **RLIMIT_FSIZE on truncate-grow ENFORCED** — `length > rlim_cur`
  delivers `-EFBIG` AND `SIGXFSZ` BEFORE any filesystem-side
  allocation. Grsec rate-limits SIGXFSZ delivery per-uid.
  CAP_SYS_RESOURCE escape policy-gated by RBAC.
- **PaX MEMORY_SANITIZE on truncated pages** — pages beyond new EOF
  zeroed before release back to page allocator. Closes cross-process
  page-reuse disclosure window. Especially important on shrink of
  memfd-shared regions where the truncated tail may have contained
  IPC data from a higher-privileged peer.
- **memfd seal enforcement HARDENED** — F_SEAL_GROW / F_SEAL_SHRINK
  rejection happens BEFORE mnt_want_write_file so attempts to bypass
  via overlay/idmap remounts cannot reach the truncate path.
  F_SEAL_WRITE additionally prevents content modification.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/fd/N` resolution
  uid-filtered; foreign fds cannot be retargeted into ftruncate.
- **PaX KERNEXEC** — `sys_ftruncate`, `do_truncate`, `notify_change`,
  `truncate_pagecache` in read-only-after-init text.
- **PaX RANDSTRUCT on `struct iattr` and `struct file`** — randomized
  layout; corruption cannot deterministically swap ia_size or f_inode.
- **GRKERNSEC_DMESG / GRKERNSEC_HIDESYM** — ftruncate failure klog
  rate-limited, CAP_SYSLOG-gated; pointer leaks scrubbed.
- **gradm policy: ftruncate REQUIRES `w` mode** — RBAC write-policy on
  the target file regardless of fd's open-mode; closes the "open via
  /proc/<pid>/fd then ftruncate" laundering pattern.
- **GRKERNSEC_HARDEN_PTRACE** — tracee-driven ftruncate via fd table
  requires PTRACE_MODE_ATTACH_FSCREDS.
- **PaX RANDKSTACK** — random kernel stack at sys_ftruncate entry;
  RLIMIT/suid-drop control flow not stack-layout-targetable.

