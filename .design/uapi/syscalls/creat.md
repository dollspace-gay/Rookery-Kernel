# Tier-5 syscall: creat(2) — syscall 85

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/open.c (SYSCALL_DEFINE2(creat), do_sys_open)
  - include/uapi/asm-generic/fcntl.h (O_CREAT, O_WRONLY, O_TRUNC)
  - arch/x86/entry/syscalls/syscall_64.tbl (85  common  creat)
-->

## Summary

`creat(2)` is a historical pre-POSIX shorthand equivalent to `open(path, O_CREAT | O_WRONLY | O_TRUNC, mode)`. It creates a new file at `path` (or truncates an existing regular file to zero length) and opens it for writing only. The kernel literally implements `creat` as a thin wrapper that calls `do_sys_open(AT_FDCWD, path, O_CREAT | O_WRONLY | O_TRUNC, mode)`. Critical for: legacy shell scripts, ancient C code, POSIX conformance test suites, and certain shell built-ins (`>`).

`creat(2)` is the limit case of `open(2)` and shares all its policy with that syscall; it remains in the ABI for backward compatibility.

## Signature

```c
int creat(const char *path, mode_t mode);

/* Strictly equivalent to: */
int open(const char *path, O_CREAT | O_WRONLY | O_TRUNC, mode);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | NUL-terminated user-space pathname; intermediate symlinks followed. |
| `mode` | `mode_t` | in | Permission bits for the new file; masked by the calling process's umask. Bits outside `0777`+SUID/SGID/sticky are ignored. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | A new file descriptor open for writing (O_WRONLY). |
| `-1` + `errno` | Failure; no fd created. |

## Errors

| errno | Trigger |
|---|---|
| `EACCES` | Write permission denied on parent directory, or search denied on a component. |
| `EDQUOT` | Filesystem quota exhausted. |
| `EEXIST` | Path exists and is not a regular file (kernel still opens regular files via O_CREAT|O_TRUNC semantics). |
| `EFAULT` | `path` outside user memory. |
| `EFBIG` | (Open succeeds but later large-file checks may fail.) |
| `EINTR` | Open interrupted by signal during slow path. |
| `EISDIR` | `path` resolved to a directory. |
| `ELOOP` | Too many symbolic links. |
| `EMFILE` | Per-process file-descriptor table full. |
| `ENAMETOOLONG` | `path` exceeds `PATH_MAX`. |
| `ENFILE` | System-wide open-file table full. |
| `ENOENT` | A directory component of `path` does not exist. |
| `ENOMEM` | Out of kernel memory. |
| `ENOSPC` | No space on filesystem. |
| `ENOTDIR` | A component of the path prefix is not a directory. |
| `EROFS` | Read-only filesystem. |
| `ETXTBSY` | Path is currently being executed and would be truncated. |
| `EOVERFLOW` | Existing file is too large for off_t (32-bit ABI without LFS). |

## ABI surface

```text
__NR_creat (x86_64)  = 85
__NR_creat (i386)    = 8
/* arm64 / riscv: no __NR_creat; userspace uses openat. */
```

## Compatibility contract

REQ-1: Syscall number is **85** on x86_64; ABI-stable.

REQ-2: Semantic equivalence: `creat(path, mode) == open(path, O_CREAT | O_WRONLY | O_TRUNC, mode)` with `dirfd = AT_FDCWD`.

REQ-3: New-file creation: if path does not exist, a regular file is created with mode `mode & ~umask` and ownership `(euid, egid)` (or sgid-from-dir for SGID-on-directory).

REQ-4: Existing-file truncation: if path exists and resolves to a regular file with sufficient write permission, the file is truncated to length 0; ctime/mtime updated.

REQ-5: Returned fd has `O_WRONLY` mode; no read access.

REQ-6: Symlink-following: `creat` follows symbolic links (unlike `O_NOFOLLOW`-augmented opens).

REQ-7: Cannot open directories (returns `EISDIR`).

REQ-8: Path-string copy_from_user bounded by `PATH_MAX`.

REQ-9: New-file inherits sgid bit from parent dir if the parent has g+s; sticky bit honored on parent for delete protection (not relevant at creation).

REQ-10: Honors `MS_RDONLY`/read-only mounts: `EROFS`.

REQ-11: Honors mount option `noexec` (no effect here; `creat` doesn't grant exec).

REQ-12: Honors LSM hooks (SELinux file-create, AppArmor, Yama): may deny with `EACCES`.

REQ-13: `mode` is masked by `umask` BEFORE permission checks; `S_ISUID` / `S_ISGID` / `S_ISVTX` bits in `mode` may be silently stripped depending on filesystem policy.

## Acceptance Criteria

- [ ] AC-1: `creat("/tmp/x", 0644)` returns a positive fd; file exists with mode `0644 & ~umask`.
- [ ] AC-2: `creat` on existing regular file: file truncated to 0.
- [ ] AC-3: `creat` on existing directory: `EISDIR`.
- [ ] AC-4: `creat` on read-only fs: `EROFS`.
- [ ] AC-5: `creat` with full disk: `ENOSPC`.
- [ ] AC-6: `creat(NULL, 0)`: `EFAULT`.
- [ ] AC-7: `creat(path_too_long, 0)`: `ENAMETOOLONG`.
- [ ] AC-8: Returned fd is writable: `write(fd, "x", 1) == 1`.
- [ ] AC-9: Returned fd is NOT readable: `read(fd, buf, 1)` returns `-1 EBADF`.
- [ ] AC-10: `creat("symlink_to_/tmp/x", 0644)`: follows symlink; creates/truncates target.
- [ ] AC-11: `creat` on running executable: `ETXTBSY`.
- [ ] AC-12: Per-process fd-table full: `EMFILE`.

## Architecture

```rust
#[syscall(nr = 85, abi = "sysv")]
pub fn sys_creat(path: UserPtr<u8>, mode: u32) -> isize {
    Open::do_sys_open(
        AT_FDCWD,
        path,
        OpenFlags::O_CREAT | OpenFlags::O_WRONLY | OpenFlags::O_TRUNC,
        mode,
    )
}
```

`Open::do_sys_open(dirfd, path_uptr, flags, mode) -> isize`:
1. let kpath = Open::copy_path_from_user(path_uptr)?;        // EFAULT, ENAMETOOLONG
2. let fd = FdTable::reserve_fd()?;                          // EMFILE
3. let open_how = OpenHow {
4.   flags: flags.bits(),
5.   mode: mode & 07777,                                     // strip non-perm bits
6.   resolve: 0,
7. };
8. let file = Open::path_openat(dirfd, &kpath, &open_how)?;  // ENOENT, EISDIR, EACCES, ELOOP, ENOSPC, EDQUOT, EROFS, ETXTBSY, ...
9. FdTable::install(fd, file);
10. Ok(fd as isize)

`Open::path_openat(dirfd, kpath, how) -> Result<Arc<File>>`:
1. Path lookup with LOOKUP_FOLLOW | LOOKUP_CREATE.
2. If exists ∧ S_ISDIR(inode): Err(EISDIR).
3. If exists ∧ S_ISREG(inode):
4.   may_open(inode, MAY_WRITE)?;                            // EACCES
5.   do_truncate(inode, 0)?;                                 // EROFS, EFBIG
6. Else: vfs_create(parent, kpath_leaf, how.mode & ~umask)?;
7. file = alloc_file(inode, O_WRONLY | O_TRUNC);
8. lsm_file_open(file)?;                                     // EACCES per-LSM
9. Ok(file)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_force_o_wronly` | INVARIANT | creat always supplies O_WRONLY; never read-capable. |
| `flags_force_o_creat` | INVARIANT | creat always supplies O_CREAT. |
| `flags_force_o_trunc` | INVARIANT | creat always supplies O_TRUNC. |
| `mode_masked_by_umask` | INVARIANT | applied mode == (mode & ~umask) & 07777. |
| `no_fd_install_on_error` | INVARIANT | error ⟹ FdTable unchanged. |

### Layer 2: TLA+

`fs/creat-syscall.tla`:
- States: per-copy_path, per-reserve_fd, per-path_openat, per-install.
- Properties:
  - `safety_fd_reservation_atomic` — reserved fd is either installed or freed.
  - `safety_o_wronly_only` — returned fd has write-only access.
  - `safety_truncation_under_write_perm` — truncate only if MAY_WRITE granted.
  - `liveness_creat_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_creat` post: ok ⟹ fd is O_WRONLY, file exists | `Open::do_sys_open` (creat path) |
| `path_openat(O_CREAT|O_TRUNC|O_WRONLY)` post: regular file, length 0 | `Open::path_openat` |

### Layer 4: Verus / Creusot functional

Per-`creat(2)` POSIX semantics; selftests in `tools/testing/selftests/filesystems/`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`creat(2)` reinforcement:

- **Per-O_WRONLY enforced** — defense against per-creat-grants-read-too misuse.
- **Per-O_TRUNC under MAY_WRITE check** — defense against per-truncate-without-perm.
- **Per-umask masking** — defense against per-permissive-default file leak.
- **Per-LSM hook on file_create** — defense against per-MAC-bypass.
- **Per-EISDIR for directories** — defense against per-directory truncate.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `path` copy_from_user** — defense against per-kernel-pointer path; SMAP forced.
- **GRKERNSEC_FIFO restrictive FIFO/regular-file create in sticky dirs** — defense against per-/tmp-symlink-race that would let an attacker pre-create a victim path.
- **GRKERNSEC_LINK restricted symlink follow** — defense against per-creat-via-symlink in world-writable sticky dirs (matches `fs.protected_symlinks`).
- **GRKERNSEC_TPE (Trusted Path Execution)** — defense against per-creat in non-trusted dirs by non-trusted users (gid-based exclusion).
- **GRKERNSEC_CHROOT_CHMOD restricts S_ISUID / S_ISGID inside chroot** — SUID bit silently stripped on creat under chroot.
- **GRKERNSEC_AUDIT_CHDIR / AUDIT_MOUNT not directly relevant** but per-creat events optionally logged when caller is in `audit_gid`.
- **Per-O_TRUNC of running executable: ETXTBSY enforced** — defense against per-running-binary-corruption.
- **Per-fd-installation atomic** — defense against per-partial-fd-create info-leak.
- **PAX_USERCOPY_HARDEN on path copy** — bounded by PATH_MAX; whitelisted slab.
- **Per-LSM file_open hook fired** — SELinux/AppArmor/Yama integration enforced before fd visible.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Full `open(2)` flag matrix (covered in `open.md` Tier-5).
- `openat(2)` / `openat2(2)` (covered in `openat.md`, `openat2.md`).
- VFS create per-fs (covered in Tier-3 `fs/namei.md`).
- Implementation code.
