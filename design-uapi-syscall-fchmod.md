---
title: "Tier-5 syscall: fchmod(2) — syscall 91"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fchmod(2)` changes the file mode bits of the file referred to by the
open file descriptor `fd`. It is the fd-based counterpart of
`chmod(2)`: identical semantics on the underlying inode, but no path
resolution. Because the fd was already obtained (with whatever
`open(2)` permission was required at that time), `fchmod` skips path
walking and operates directly on `file->f_inode`.

`fchmod` is preferred over `chmod` in race-free programs: the caller
captured a reference to the inode via the fd, so a TOCTOU swap between
lookup and metadata change is impossible. Used heavily by tar/cpio
extractors, runtimes that materialize executables via `memfd_create`,
and atomic-update flows (`open` → `fchmod` → `rename`).

Same permission rules as `chmod`: owner or `CAP_FOWNER`. Same setgid
filtering. Same LSM, audit, fsnotify, and ctime semantics. The fd does
NOT need to have been opened with write access; `O_RDONLY` is enough.
`O_PATH` fds are accepted (kernel commit `bf3c4d4`).

Critical for: race-free metadata change on freshly-created files,
`memfd` workflows, container image extraction, `mkstemp` finishers.

### Acceptance Criteria

- [ ] AC-1: `fchmod(fd, 0644)` on owned regular file: succeeds; mode 0100644.
- [ ] AC-2: `fchmod(-1, 0644)`: `-EBADF`.
- [ ] AC-3: `fchmod(fd_to_/etc/shadow, 0777)` as non-root: `-EPERM`.
- [ ] AC-4: `fchmod` on O_PATH fd: succeeds with same owner check.
- [ ] AC-5: `fchmod` on memfd: succeeds; subsequent fstat shows new mode.
- [ ] AC-6: `fchmod(fd, 02755)` with non-matching group, no CAP_FSETID: setgid cleared.
- [ ] AC-7: `fchmod(fd, 0644)` on RO-mount fd: `-EROFS`.
- [ ] AC-8: `fchmod` on pipe fd: succeeds; pipe's anonymous inode mode updated.
- [ ] AC-9: ctime touched even if new mode == old mode.
- [ ] AC-10: Audit captures fd → resolved path + old/new mode.
- [ ] AC-11: LSM (SELinux) policy denial: `-EACCES`.
- [ ] AC-12: Bind-mounted RO-overlay: `-EROFS` from mnt_want_write_file.

### Architecture

```rust
#[syscall(nr = 91, abi = "sysv")]
pub fn sys_fchmod(fd: c_int, mode: u16) -> isize {
    Fs::do_fchmod(fd, mode)
}
```

`Fs::do_fchmod(fd, mode) -> isize`:
1. let f = fdget(fd)?;                                       /* EBADF */
2. let r = Fs::chmod_common(&f.file.f_path, mode & 0o7777);  /* shared with chmod */
3. fdput(f);
4. r

`Fs::chmod_common(path, mode) -> isize`: identical to chmod (see chmod.md
REQ-section + Architecture). Notable per-fchmod points:
- `mnt_want_write_file(f.file)` instead of `mnt_want_write(path.mnt)` —
  preserves "write to mount currently exposed by this fd" semantics.
- Inode obtained via `f.file.f_inode`, NOT via dentry lookup — TOCTOU
  immunity.

### Out of Scope

- `chmod(2)` (separate Tier-5 — path-based).
- `fchmodat(2)` (separate Tier-5 — at-relative).
- `fchown(2)` (separate Tier-5 — ownership counterpart).
- `memfd_create(2)` (separate Tier-5).
- Implementation code.

### signature

```c
int fchmod(int fd, mode_t mode);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor. May be O_RDONLY, O_RDWR, or O_PATH. |
| `mode` | `mode_t` | in | New mode bits — low 12 bits used (setuid/setgid/sticky/rwx). |

### return value

| Value | Meaning |
|---|---|
| `0` | mode updated. |
| `-1` + `errno` | Failure; mode unchanged. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` is not a valid open file descriptor. |
| `EPERM` | Caller is not file owner and lacks `CAP_FOWNER`; or file is immutable. |
| `EROFS` | File's filesystem is read-only (or fd's mount is RO). |
| `EIO`  | I/O error writing inode metadata. |
| `EACCES` | LSM (SELinux/AppArmor) denial. |

### abi surface

```text
__NR_fchmod (x86_64)  = 91
__NR_fchmod (i386)    = 94
__NR_fchmod (arm64)   = 52
__NR_fchmod (riscv)   = 52
__NR_fchmod (generic) = 52
```

### compatibility contract

REQ-1: Syscall number is **91** on x86_64; **52** on generic. ABI-stable.

REQ-2: Per-fd resolution: `fdget(fd)` → struct file. EBADF if invalid.

REQ-3: Per-O_PATH: fd opened with `O_PATH` is acceptable; fchmod uses
the inode pinned by the fd regardless of read/write mode (kernel
commit `bf3c4d4`).

REQ-4: Per-mode masking: low 12 bits only (`mode & 07777`); upper bits
ignored.

REQ-5: Per-owner-check: `current.fsuid == inode.uid` OR `CAP_FOWNER`
in inode's user_ns. Else `-EPERM`.

REQ-6: Per-setgid-restrict: setgid bit silently cleared if inode.gid
not in caller's group set and caller lacks `CAP_FSETID`. Same rule as
chmod.

REQ-7: Per-RO-mount: `mnt_want_write_file(file)` → `-EROFS` on RO mount.
The file's mount, not the inode's filesystem, gates this.

REQ-8: Per-immutable: inode `S_IMMUTABLE` → `-EPERM` unless
`CAP_LINUX_IMMUTABLE`.

REQ-9: Per-notify_change with `ATTR_MODE | ATTR_CTIME`. Filesystem
`setattr` op invoked.

REQ-10: Per-ctime-update: ctime touched even if mode unchanged.

REQ-11: Per-LSM: `security_inode_setattr` before mode change.

REQ-12: Per-fsnotify: `FS_ATTRIB` emitted post-success.

REQ-13: Per-audit: `AUDIT_SYSCALL` records fd, path-by-fd, old mode,
new mode.

REQ-14: Per-mount-uid/gid idmap: ownership check uses idmapped IDs.

REQ-15: Per-memfd: `memfd_create` fds are anonymous inodes; fchmod
on them is permitted (owner = caller).

REQ-16: Per-pipefd / sockfd: fchmod on a pipe or socket fd: succeeds
(POSIX explicitly allows). Mode bits stored on the anonymous inode.

REQ-17: Per-procfs fd: fchmod on `/proc/self/fd/N` indirects to the
underlying inode of fd N.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_valid_before_inode` | ORDER | EBADF returned BEFORE inode dereference. |
| `mode_masked_07777` | INVARIANT | Only low 12 bits applied. |
| `owner_check_pre_setattr` | ORDER | EPERM BEFORE notify_change. |
| `mnt_write_file_balanced` | INVARIANT | mnt_want_write_file balanced by mnt_drop_write_file. |
| `o_path_accepted` | INVARIANT | O_PATH fd does not return -EBADF. |

### Layer 2: TLA+

`fs/fchmod.tla`:
- States: FDGET → OWNER_CHECK → MNT_WRITE_FILE → SETGID_FILTER → LSM → SETATTR.
- Properties:
  - `safety_owner_check` — only owner or CAP_FOWNER may chmod.
  - `safety_no_toctou` — inode bound at fdget; no path race.
  - `safety_setgid_drop` — setgid silently dropped under REQ-6.
  - `liveness_terminates` — fchmod terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_fchmod` post: inode.mode low-12 = arg & 07777 (modulo filter) | `sys_fchmod` |
| `do_fchmod` post: f.refcount balanced (fdget/fdput) | `Fs::do_fchmod` |
| `chmod_common` post: ctime touched on 0 ret | `Fs::chmod_common` |

### Layer 4: Verus / Creusot functional

Per-`fchmod(2)` man-page, per-POSIX, per-Linux fs/open.c, LTP `fchmod01..07`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fchmod(2)` reinforcement:

- **Per-fdget atomic** — defense against per-fd-table race.
- **Per-O_PATH inclusion** — defense against per-attempt to forge fd type.
- **Per-mnt_want_write_file** — defense against per-RO-mount-overlay bypass.
- **Per-owner / CAP_FOWNER strict** — defense against per-DAC bypass.
- **Per-setgid silent-drop** — defense against per-stealthy-sgid escalation.
- **Per-LSM hook** — defense against per-MAC bypass.
- **Per-TOCTOU immunity (fd-bound inode)** — defense against per-rename race.

### grsecurity / pax-style reinforcement

- **PaX UDEREF** — fchmod has no user-pointer dereference in its main
  path; UDEREF policy still applies uniformly. The `mode` argument
  passes through register only.
- **CAP_FOWNER strict** — gradm RBAC may forbid `CAP_FOWNER` for non-
  policy subjects; default deny for new suid-bit creation.
- **GRKERNSEC_SUIDDUMP on suid-mode-change** — adding or removing
  `S_ISUID` / `S_ISGID` via fchmod logs sender uid, fd-resolved path,
  old mode, new mode. Coredump suppression flag recomputed.
- **GRKERNSEC_CHROOT_CHMOD** — chrooted process cannot fchmod-set
  `S_ISUID` / `S_ISGID` on any file in its chroot, regardless of the
  fd's origin (even if the fd was opened pre-chroot). The bit is
  silently cleared from the iattr.
- **GRKERNSEC_TPE on setuid-mode set** — fchmod setting `S_ISUID` on a
  binary whose containing directory is world/group-writable returns
  `-EPERM` under TPE.
- **File-cap-strip on chown (cross-reference)** — fchown after fchmod
  triggers cap-strip; rate-limited.
- **PaX MEMORY_SANITIZE on Iattr** — zero-initialized; unused arms not
  leaked.
- **GRKERNSEC_PROC restrictions** — fd-by-path resolution via
  `/proc/<pid>/fd/N` is uid-filtered; foreign fds cannot be fchmod'd
  via that route.
- **PaX KERNEXEC** — `chmod_common`, `notify_change`, and the fdget
  fast path are read-only-after-init kernel text.
- **GRKERNSEC_DMESG** — fchmod failure klog (e.g. chroot-suid attempt)
  rate-limited, CAP_SYSLOG-gated.
- **GRKERNSEC_HIDESYM** — pointer leaks scrubbed from any fchmod log
  message (inode, file, dentry).
- **PaX RANDSTRUCT on `struct file`** — randomized layout; corruption
  cannot deterministically swap `f_inode` to mis-target fchmod.
- **PaX RANDSTRUCT on `struct iattr`** — same protection for `ia_mode`.
- **gradm policy: fchmod-suid as discrete capability** — RBAC `O` mode
  for setuid-bit setting; subjects without policy cannot create new
  suid binaries even via fchmod-on-memfd workflows.
- **GRKERNSEC_HARDEN_PTRACE** — a tracer cannot use the tracee's fd
  table to fchmod across credential boundaries; PTRACE_MODE_ATTACH_FSCREDS
  enforced before fd table access.
- **memfd-suid policy** — combined with `MFD_NOEXEC_SEAL` (recent
  upstream), grsec gradm forbids `S_ISUID` on any memfd, closing
  the "create-suid-via-anonymous-inode" pattern.

