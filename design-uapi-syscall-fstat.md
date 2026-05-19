---
title: "Tier-5 syscall: fstat(2) — syscall 5"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`fstat(2)` retrieves the file-metadata block of the file referenced by file descriptor `fd`. The kernel calls `fdget(fd)` to obtain the underlying `struct file`, then `vfs_getattr(&file->f_path, &kstat, STATX_BASIC_STATS, AT_NO_AUTOMOUNT_DISABLED)`, and translates `kstat` into the ABI-frozen `struct stat`. Because `fd` already names a resolved open-file, no path walk, symlink chase, or permission check on path components occurs — the call inherits the access already granted at `open(2)` time. Critical for: libc `fstat()`, mmap setup (size discovery), seek validation, /proc tooling.

`fstat(2)` is the file-descriptor sibling of `stat(2)` and `lstat(2)`.

### Acceptance Criteria

- [ ] AC-1: `open(O_RDONLY) + fstat` on a regular file: returns 0, `S_ISREG`.
- [ ] AC-2: `fstat` on directory fd: `S_ISDIR`.
- [ ] AC-3: `fstat` on pipe[0]: `S_ISFIFO`.
- [ ] AC-4: `fstat` on socketpair fd: `S_ISSOCK`.
- [ ] AC-5: `fstat(eventfd_fd, &sb)` succeeds; `S_ISREG(sb.st_mode) == 1` (anon_inode marker).
- [ ] AC-6: `fstat(-1, &sb)` returns -1 with `EBADF`.
- [ ] AC-7: `fstat(fd, (void*)0x1)` returns -1 with `EFAULT`.
- [ ] AC-8: `fstat` on `O_PATH` fd: success (metadata-only access).
- [ ] AC-9: After `close(fd)`, `fstat(fd, ...)` returns `EBADF`.
- [ ] AC-10: `fstat` never modifies `st_atim` of the file (no access timestamp update).

### Architecture

```rust
#[syscall(nr = 5, abi = "sysv")]
pub fn sys_fstat(fd: i32, statbuf: UserPtr<UapiStat>) -> isize {
    Stat::do_fstat(fd, statbuf)
}
```

`Stat::do_fstat(fd, statbuf) -> isize`:
1. let file = FdTable::get(fd)?;                          // EBADF
2. let kstat = Stat::vfs_getattr_path(&file.f_path, STATX_BASIC_STATS)?;
3. let ustat = Stat::cp_new_stat(&kstat)?;                // EOVERFLOW
4. statbuf.write_to_user(&ustat)?;                        // EFAULT
5. FdTable::put(file);
6. Ok(0)

`Stat::vfs_getattr_path(path, req_mask) -> Result<KStat>`:
1. let inode = path.dentry.inode;
2. let mut k = KStat::zeroed();
3. if let Some(op) = inode.i_op.getattr {
4.   op(path, &mut k, req_mask, AT_NO_AUTOMOUNT_DISABLED)?;
5. } else {
6.   Stat::generic_fillattr(inode, &mut k);
7. }
8. Ok(k)

`Stat::generic_fillattr(inode, kstat)`:
1. kstat.dev      = inode.i_sb.s_dev;
2. kstat.ino      = inode.i_ino;
3. kstat.mode     = inode.i_mode;
4. kstat.nlink    = inode.i_nlink;
5. kstat.uid      = inode.i_uid;
6. kstat.gid      = inode.i_gid;
7. kstat.rdev     = inode.i_rdev;
8. kstat.size     = i_size_read(inode);
9. kstat.atime    = inode.i_atime;
10. kstat.mtime   = inode.i_mtime;
11. kstat.ctime   = inode.i_ctime;
12. kstat.blksize = i_blocksize(inode);
13. kstat.blocks  = inode.i_blocks;

### Out of Scope

- `statx(2)` extended interface (covered in `statx.md` Tier-5).
- VFS getattr per-fs implementation (covered in Tier-3 `fs/stat.md`).
- 32-bit `fstat64` ABI quirks (covered in arch-compat docs).
- Implementation code.

### signature

```c
int fstat(int fd, struct stat *statbuf);
```

```c
struct stat {
    dev_t      st_dev;
    ino_t      st_ino;
    mode_t     st_mode;
    nlink_t    st_nlink;
    uid_t      st_uid;
    gid_t      st_gid;
    dev_t      st_rdev;
    off_t      st_size;
    blksize_t  st_blksize;
    blkcnt_t   st_blocks;
    struct timespec st_atim;
    struct timespec st_mtim;
    struct timespec st_ctim;
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor; may name any object that has an inode (file, dir, pipe, socket, anon_inode). |
| `statbuf` | `struct stat *` | out | User-space buffer of `sizeof(struct stat)` bytes; populated on success. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; `*statbuf` written. |
| `-1` + `errno` | Failure; `*statbuf` unchanged. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` is not a valid open file descriptor for the calling process. |
| `EFAULT` | `statbuf` points outside accessible user memory. |
| `ENOMEM` | Out of kernel memory. |
| `EOVERFLOW` | `st_size`, `st_blocks`, or `st_ino` does not fit in the legacy struct on a 32-bit ABI. |

### abi surface

```text
__NR_fstat   (x86_64)  = 5
__NR_fstat   (i386)    = 108   (legacy 32-bit struct)
__NR_fstat64 (i386)    = 197   (LFS variant)
/* arm64 / riscv: no __NR_fstat; userspace uses statx or fstatat64. */
```

### compatibility contract

REQ-1: Syscall number is **5** on x86_64; ABI-stable.

REQ-2: `fd` lookup uses `fdget(fd)` for fast-path (RCU + bias-ref); released via `fdput()`.

REQ-3: No path walk; no symlink semantics; no LOOKUP_FOLLOW vs NOFOLLOW distinction.

REQ-4: `vfs_getattr` dispatches to the `i_op->getattr` of the inode underlying `file->f_path.dentry->d_inode`.

REQ-5: For `O_PATH` file descriptors, `fstat` still succeeds — `O_PATH` permits metadata operations.

REQ-6: For anon_inode descriptors (eventfd, signalfd, timerfd, epoll, bpf-prog, bpf-map, ...): `st_mode` reports the file's intrinsic mode (`S_IFREG | 0600` typical), `st_size = 0`, `st_dev = 0`, `st_ino` = a private kernfs-style ino.

REQ-7: For sockets: `st_mode = S_IFSOCK | 0777`, fields mostly zero.

REQ-8: For pipes / FIFOs (anon_pipe): `st_mode = S_IFIFO | 0600`, `st_size` = bytes currently in pipe.

REQ-9: Permission to *call* `fstat` is gated solely by ownership of `fd`; no further DAC check applies.

REQ-10: `statbuf` is written **only** on success; on error, the buffer is untouched (the kernel-side intermediate `kstat` is filled, then translated and copied as the last step).

REQ-11: All `copy_to_user` is bounded by `sizeof(struct stat)`; PaX UDEREF / SMAP enforced.

REQ-12: On 64-bit ABI, `EOVERFLOW` is structurally impossible.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_get_balanced` | INVARIANT | every `FdTable::get` is paired with `FdTable::put`. |
| `no_statbuf_write_on_error` | INVARIANT | error ⟹ statbuf untouched. |
| `no_path_walk` | INVARIANT | fstat never invokes path-resolution. |
| `kstat_translation_total` | INVARIANT | every kstat field has translation or EOVERFLOW. |

### Layer 2: TLA+

`fs/fstat-syscall.tla`:
- States: per-fdget, per-vfs_getattr, per-cp_new_stat, per-copy_to_user, per-fdput.
- Properties:
  - `safety_fdput_after_get` — every fdget eventually fdput.
  - `safety_statbuf_write_post_success` — write only after success.
  - `liveness_fstat_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fstat` post: ok ⟹ statbuf valid; fd ref-balanced | `Stat::do_fstat` |
| `vfs_getattr_path` post: kstat populated | `Stat::vfs_getattr_path` |

### Layer 4: Verus / Creusot functional

Per-`fstat(2)` POSIX semantics; selftests in `tools/testing/selftests/filesystems/`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fstat(2)` reinforcement:

- **Per-fdget RCU-safe reference acquisition** — defense against per-fd-UAF.
- **Per-no-path-walk** — defense against per-symlink-race.
- **Per-statbuf write last** — defense against per-partial-write info-leak.
- **Per-O_PATH metadata-only access** — defense against per-bypass of full open privs.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `statbuf` copy_to_user** — defense against per-kernel-pointer destination; SMAP forced.
- **GRKERNSEC_HIDESYM on st_ino / st_dev / st_rdev** — pseudo-filesystem inode / device numbers (procfs, sysfs, kernfs, bpffs anon_inode) are masked for unprivileged callers; only CAP_SYS_ADMIN sees raw values.
- **GRKERNSEC_PROC_USERGROUP info-leak guard** — `st_uid` / `st_gid` of `/proc/<pid>/` files are reported as the calling uid/gid when `proc=hidepid=4` and caller is not owner.
- **PAX_USERCOPY_HARDEN on cp_new_stat copy_to_user** — bounded, whitelisted slab; size is a compile-time constant.
- **Per-anon_inode st_dev = 0 enforced** — defense against per-cross-anon-inode device-ID leak that would otherwise help fingerprint kernel internal pointers.
- **Per-fd ownership table strict** — defense against per-cross-process fd injection (anti-CVE-like classes where a closed fd is briefly visible).
- **Per-fdget RCU bias-ref strict** — defense against per-fdtable-resize UAF.
- **Per-O_PATH fstat permitted but no I/O** — defense against per-O_PATH-priv-escalation.

