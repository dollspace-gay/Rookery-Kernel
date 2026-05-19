# Tier-5 UAPI: include/uapi/linux/fcntl.h — fcntl(2) ABI + *at(2) flags

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/fcntl.h (~193 lines)
  - include/uapi/asm-generic/fcntl.h (struct flock, struct flock64, struct f_owner_ex, F_DUPFD .. F_OFD_*, F_RDLCK/F_WRLCK/F_UNLCK, F_LINUX_SPECIFIC_BASE)
  - include/uapi/linux/fs.h (struct file_clone_range, struct fsxattr, struct file_dedupe_range, struct file_dedupe_range_info)
-->

## Summary

`linux/fcntl.h` is the **`fcntl(2)` ABI** seen by every userspace `fcntl(2)`, `open(2)`, `openat(2)`, `openat2(2)`, `linkat(2)`, `unlinkat(2)`, `renameat2(2)`, `faccessat(2)`, `statx(2)`, `execveat(2)`, `name_to_handle_at(2)` caller. The header layers Linux-specific `F_*` operations on top of `asm-generic/fcntl.h` (which provides the POSIX core: `F_DUPFD`, `F_GETFD`, `F_SETFD`, `F_GETFL`, `F_SETFL`, `F_GETLK`, `F_SETLK`, `F_SETLKW`, `F_SETOWN`, `F_GETOWN`, `F_SETOWN_EX`, `F_GETOWN_EX`, `F_GETOWNER_UIDS`, `F_OFD_GETLK`, `F_OFD_SETLK`, `F_OFD_SETLKW`, `struct flock`, `struct flock64`, `struct f_owner_ex`, `F_RDLCK`, `F_WRLCK`, `F_UNLCK`).

Linux-specific commands live at `F_LINUX_SPECIFIC_BASE` (`1024`) + offset:
- `F_SETLEASE` / `F_GETLEASE` (lease lifecycle)
- `F_NOTIFY` (legacy dnotify, deprecated in favor of inotify/fanotify)
- `F_DUPFD_QUERY`, `F_CREATED_QUERY` (introspection)
- `F_CANCELLK` (cancel blocking POSIX lock, internal)
- `F_DUPFD_CLOEXEC` (dup with `O_CLOEXEC`)
- `F_SETPIPE_SZ` / `F_GETPIPE_SZ` (pipe-buffer sizing)
- `F_ADD_SEALS` / `F_GET_SEALS` (memfd write-once sealing)
- `F_GET_RW_HINT` / `F_SET_RW_HINT` / `F_GET_FILE_RW_HINT` / `F_SET_FILE_RW_HINT` (write-life-time hints)
- `F_GETDELEG` / `F_SETDELEG` (NFSv4-style delegations)

The header also pins the **`AT_*` flag namespace** consumed by every `*at(2)` syscall: `AT_FDCWD = -100`, `AT_SYMLINK_NOFOLLOW`, `AT_SYMLINK_FOLLOW`, `AT_NO_AUTOMOUNT`, `AT_EMPTY_PATH`, `AT_RECURSIVE`, `AT_STATX_SYNC_TYPE` / `AT_STATX_FORCE_SYNC` / `AT_STATX_DONT_SYNC` / `AT_STATX_SYNC_AS_STAT`, the overloaded-per-syscall flags `AT_EACCESS` and `AT_REMOVEDIR` (both `0x200`, mutually exclusive by syscall context), `AT_HANDLE_FID` / `AT_HANDLE_MNT_ID_UNIQUE` / `AT_HANDLE_CONNECTABLE` (name_to_handle_at), `AT_RENAME_NOREPLACE` / `AT_RENAME_EXCHANGE` / `AT_RENAME_WHITEOUT` (renameat2), `AT_EXECVE_CHECK` (execveat2), and the per-syscall special pidfds (`PIDFD_SELF_THREAD = -10000`, `PIDFD_SELF_THREAD_GROUP = -10001`, `FD_PIDFS_ROOT = -10002`, `FD_NSFS_ROOT = -10003`, `FD_INVALID = -10009`).

Auxiliary userspace-facing structures referenced through fcntl-adjacent syscalls and ioctls are also part of the wire ABI: `struct flock`, `struct flock64`, `struct f_owner_ex`, `struct delegation`, `struct file_clone_range` (FICLONERANGE), `struct fsxattr` (FS_IOC_FSGETXATTR / FS_IOC_FSSETXATTR), and `struct file_dedupe_range` + `struct file_dedupe_range_info` (FIDEDUPERANGE).

Critical for: every `open()` / `dup()` / `dup2()` / `dup3()` / `pipe()` / `pipe2()` / `flock()` / `fcntl()` caller, libc `posix_spawn()`, sandboxed builds, container runtimes (memfd seals), io_uring fixed-fd registration, NFSv4 clients, ftruncate/fallocate sequences relying on F_SEAL_GROW.

This Tier-5 covers `include/uapi/linux/fcntl.h` (~193 lines) plus the ABI structures that fcntl operations consume.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `F_DUPFD` (0)               | per-dup | `uapi::fcntl::cmd::DUPFD` |
| `F_GETFD` (1)               | per-get-FD_CLOEXEC | `uapi::fcntl::cmd::GETFD` |
| `F_SETFD` (2)               | per-set-FD_CLOEXEC | `uapi::fcntl::cmd::SETFD` |
| `F_GETFL` (3)               | per-get-file-status-flags | `uapi::fcntl::cmd::GETFL` |
| `F_SETFL` (4)               | per-set-file-status-flags | `uapi::fcntl::cmd::SETFL` |
| `F_GETLK` (5)               | per-query-POSIX-lock | `uapi::fcntl::cmd::GETLK` |
| `F_SETLK` (6)               | per-acquire-POSIX-lock-nonblock | `uapi::fcntl::cmd::SETLK` |
| `F_SETLKW` (7)              | per-acquire-POSIX-lock-blocking | `uapi::fcntl::cmd::SETLKW` |
| `F_SETOWN` (8)              | per-set-async-signal-owner | `uapi::fcntl::cmd::SETOWN` |
| `F_GETOWN` (9)              | per-get-async-signal-owner | `uapi::fcntl::cmd::GETOWN` |
| `F_SETOWN_EX` (15)          | per-set-owner-with-type | `uapi::fcntl::cmd::SETOWN_EX` |
| `F_GETOWN_EX` (16)          | per-get-owner-with-type | `uapi::fcntl::cmd::GETOWN_EX` |
| `F_GETOWNER_UIDS` (17)      | per-get-owner-UID-pair | `uapi::fcntl::cmd::GETOWNER_UIDS` |
| `F_OFD_GETLK` (36)          | per-OFD-lock-query | `uapi::fcntl::cmd::OFD_GETLK` |
| `F_OFD_SETLK` (37)          | per-OFD-lock-nonblock | `uapi::fcntl::cmd::OFD_SETLK` |
| `F_OFD_SETLKW` (38)         | per-OFD-lock-blocking | `uapi::fcntl::cmd::OFD_SETLKW` |
| `F_SETLEASE` (1024+0)       | per-acquire-lease | `uapi::fcntl::cmd::SETLEASE` |
| `F_GETLEASE` (1024+1)       | per-get-lease | `uapi::fcntl::cmd::GETLEASE` |
| `F_NOTIFY` (1024+2)         | per-dnotify (legacy) | `uapi::fcntl::cmd::NOTIFY` |
| `F_DUPFD_QUERY` (1024+3)    | per-dup-introspect | `uapi::fcntl::cmd::DUPFD_QUERY` |
| `F_CREATED_QUERY` (1024+4)  | per-file-just-created-query | `uapi::fcntl::cmd::CREATED_QUERY` |
| `F_CANCELLK` (1024+5)       | per-cancel-blocking-POSIX-lock (internal) | `uapi::fcntl::cmd::CANCELLK` |
| `F_DUPFD_CLOEXEC` (1024+6)  | per-dup-with-O_CLOEXEC | `uapi::fcntl::cmd::DUPFD_CLOEXEC` |
| `F_SETPIPE_SZ` (1024+7)     | per-set-pipe-buffer-size | `uapi::fcntl::cmd::SETPIPE_SZ` |
| `F_GETPIPE_SZ` (1024+8)     | per-get-pipe-buffer-size | `uapi::fcntl::cmd::GETPIPE_SZ` |
| `F_ADD_SEALS` (1024+9)      | per-memfd-add-seal | `uapi::fcntl::cmd::ADD_SEALS` |
| `F_GET_SEALS` (1024+10)     | per-memfd-get-seals | `uapi::fcntl::cmd::GET_SEALS` |
| `F_GET_RW_HINT` (1024+11)   | per-get-inode-rw-hint | `uapi::fcntl::cmd::GET_RW_HINT` |
| `F_SET_RW_HINT` (1024+12)   | per-set-inode-rw-hint | `uapi::fcntl::cmd::SET_RW_HINT` |
| `F_GET_FILE_RW_HINT` (1024+13) | per-get-file-rw-hint | `uapi::fcntl::cmd::GET_FILE_RW_HINT` |
| `F_SET_FILE_RW_HINT` (1024+14) | per-set-file-rw-hint | `uapi::fcntl::cmd::SET_FILE_RW_HINT` |
| `F_GETDELEG` (1024+15)      | per-get-delegation | `uapi::fcntl::cmd::GETDELEG` |
| `F_SETDELEG` (1024+16)      | per-set-delegation | `uapi::fcntl::cmd::SETDELEG` |
| `struct flock`              | per-POSIX-record-lock | `uapi::fcntl::Flock` |
| `struct flock64`            | per-POSIX-record-lock-64bit | `uapi::fcntl::Flock64` |
| `struct f_owner_ex`         | per-owner-with-type | `uapi::fcntl::FOwnerEx` |
| `struct delegation`         | per-NFSv4-delegation | `uapi::fcntl::Delegation` |
| `struct file_clone_range`   | per-FICLONERANGE | `uapi::fs::FileCloneRange` |
| `struct fsxattr`            | per-FS_IOC_FSxXATTR | `uapi::fs::Fsxattr` |
| `struct file_dedupe_range`  | per-FIDEDUPERANGE | `uapi::fs::FileDedupeRange` |
| `struct file_dedupe_range_info` | per-FIDEDUPERANGE-entry | `uapi::fs::FileDedupeRangeInfo` |
| `AT_FDCWD` (-100)           | per-cwd-relative-dirfd | `uapi::fcntl::at::FDCWD` |
| `AT_SYMLINK_NOFOLLOW` (0x100) | per-no-follow-symlink | `uapi::fcntl::at::SYMLINK_NOFOLLOW` |
| `AT_EACCESS` (0x200)        | per-faccessat-effective-uid | `uapi::fcntl::at::EACCESS` |
| `AT_REMOVEDIR` (0x200)      | per-unlinkat-as-rmdir | `uapi::fcntl::at::REMOVEDIR` |
| `AT_HANDLE_FID` (0x200)     | per-name_to_handle_at-FID | `uapi::fcntl::at::HANDLE_FID` |
| `AT_SYMLINK_FOLLOW` (0x400) | per-follow-symlink | `uapi::fcntl::at::SYMLINK_FOLLOW` |
| `AT_NO_AUTOMOUNT` (0x800)   | per-suppress-automount | `uapi::fcntl::at::NO_AUTOMOUNT` |
| `AT_EMPTY_PATH` (0x1000)    | per-empty-path-on-dirfd | `uapi::fcntl::at::EMPTY_PATH` |
| `AT_STATX_SYNC_TYPE` (0x6000) | per-statx-sync-mask | `uapi::fcntl::at::STATX_SYNC_TYPE` |
| `AT_STATX_SYNC_AS_STAT` (0x0000) | per-statx-sync-as-stat | `uapi::fcntl::at::STATX_SYNC_AS_STAT` |
| `AT_STATX_FORCE_SYNC` (0x2000)   | per-statx-force-sync | `uapi::fcntl::at::STATX_FORCE_SYNC` |
| `AT_STATX_DONT_SYNC` (0x4000)    | per-statx-dont-sync | `uapi::fcntl::at::STATX_DONT_SYNC` |
| `AT_RECURSIVE` (0x8000)     | per-apply-subtree | `uapi::fcntl::at::RECURSIVE` |
| `AT_RENAME_NOREPLACE` (0x0001) | per-renameat2-no-replace | `uapi::fcntl::at::RENAME_NOREPLACE` |
| `AT_RENAME_EXCHANGE` (0x0002)  | per-renameat2-exchange | `uapi::fcntl::at::RENAME_EXCHANGE` |
| `AT_RENAME_WHITEOUT` (0x0004)  | per-renameat2-whiteout | `uapi::fcntl::at::RENAME_WHITEOUT` |
| `AT_HANDLE_MNT_ID_UNIQUE` (0x001) | per-name_to_handle_at-unique-mntid | `uapi::fcntl::at::HANDLE_MNT_ID_UNIQUE` |
| `AT_HANDLE_CONNECTABLE` (0x002)   | per-name_to_handle_at-connectable | `uapi::fcntl::at::HANDLE_CONNECTABLE` |
| `AT_EXECVE_CHECK` (0x10000) | per-execveat2-check-only | `uapi::fcntl::at::EXECVE_CHECK` |
| `RWH_WRITE_LIFE_*` (0..5)   | per-write-life-time-hint | `uapi::fcntl::rwh` constants |
| `F_SEAL_SEAL`, `F_SEAL_SHRINK`, `F_SEAL_GROW`, `F_SEAL_WRITE`, `F_SEAL_FUTURE_WRITE`, `F_SEAL_EXEC` | per-memfd seal bits | `uapi::fcntl::seal` constants |
| `F_RDLCK` (0), `F_WRLCK` (1), `F_UNLCK` (2) | per-lock-type | `uapi::fcntl::lock_type` constants |
| `F_OWNER_TID`, `F_OWNER_PID`, `F_OWNER_PGRP` | per-owner-type | `uapi::fcntl::owner_type` constants |
| `DN_ACCESS`, `DN_MODIFY`, `DN_CREATE`, `DN_DELETE`, `DN_RENAME`, `DN_ATTRIB`, `DN_MULTISHOT` | per-dnotify event bits | `uapi::fcntl::dn` constants |
| `FD_CLOEXEC` (1)            | per-close-on-exec bit | `uapi::fcntl::FD_CLOEXEC` |
| `PIDFD_SELF_THREAD` (-10000), `PIDFD_SELF_THREAD_GROUP` (-10001) | per-self pidfd | `uapi::fcntl::pidfd_self` |
| `FD_PIDFS_ROOT` (-10002), `FD_NSFS_ROOT` (-10003), `FD_INVALID` (-10009) | per-special dirfd | `uapi::fcntl::dirfd_special` |

## ABI surface (constants + structs)

### Linux-specific `F_*` commands (base = `F_LINUX_SPECIFIC_BASE = 1024`)

| Cmd | Value | Arg type | Purpose |
|---|---|---|---|
| `F_SETLEASE`       | 1024+0  | `int`               | Acquire/release lease (`F_RDLCK`/`F_WRLCK`/`F_UNLCK`) |
| `F_GETLEASE`       | 1024+1  | -                   | Query current lease type |
| `F_NOTIFY`         | 1024+2  | `int` (DN_* mask)   | Legacy dnotify (deprecated; prefer inotify/fanotify) |
| `F_DUPFD_QUERY`    | 1024+3  | `int`               | Ask whether two fds share the same open file description |
| `F_CREATED_QUERY`  | 1024+4  | -                   | Was the file just created by `open(O_CREAT)`? |
| `F_CANCELLK`       | 1024+5  | -                   | Cancel blocking POSIX lock (internal-only) |
| `F_DUPFD_CLOEXEC`  | 1024+6  | `int`               | Dup with FD_CLOEXEC set |
| `F_SETPIPE_SZ`     | 1024+7  | `int` (bytes)       | Set pipe buffer size; clamped to `[PAGE_SIZE, pipe-max-size sysctl]` |
| `F_GETPIPE_SZ`     | 1024+8  | -                   | Get pipe buffer size |
| `F_ADD_SEALS`      | 1024+9  | `int` (seal mask)   | Add seals to memfd (one-way; rejected if `F_SEAL_SEAL` already set) |
| `F_GET_SEALS`      | 1024+10 | -                   | Get current seal mask |
| `F_GET_RW_HINT`    | 1024+11 | `__u64*`            | Get inode-level write-life-time hint |
| `F_SET_RW_HINT`    | 1024+12 | `__u64*`            | Set inode-level write-life-time hint |
| `F_GET_FILE_RW_HINT` | 1024+13 | `__u64*`          | Get fd-level (per-`struct file`) hint |
| `F_SET_FILE_RW_HINT` | 1024+14 | `__u64*`          | Set fd-level hint |
| `F_GETDELEG`       | 1024+15 | `struct delegation*`| Get NFSv4 delegation |
| `F_SETDELEG`       | 1024+16 | `struct delegation*`| Set NFSv4 delegation |

### POSIX commands inherited from `<asm-generic/fcntl.h>`

| Cmd | Value | Arg | Purpose |
|---|---|---|---|
| `F_DUPFD`   | 0 | `int` | dup, lowest fd ≥ arg |
| `F_GETFD`   | 1 | -     | get FD flags (only `FD_CLOEXEC`) |
| `F_SETFD`   | 2 | `int` | set FD flags |
| `F_GETFL`   | 3 | -     | get file status flags (`O_*`) |
| `F_SETFL`   | 4 | `int` | set file status flags (only `O_APPEND` / `O_NONBLOCK` / etc.) |
| `F_GETLK`   | 5 | `struct flock*` | query POSIX record lock |
| `F_SETLK`   | 6 | `struct flock*` | acquire POSIX record lock (non-blocking) |
| `F_SETLKW`  | 7 | `struct flock*` | acquire POSIX record lock (blocking, interruptible) |
| `F_SETOWN`  | 8 | `int` | set owner for SIGIO/SIGURG delivery |
| `F_GETOWN`  | 9 | -     | get owner |
| `F_SETSIG`  | 10| `int` | set delivered signal (default SIGIO) |
| `F_GETSIG`  | 11| -     | get delivered signal |
| `F_GETLK64`/`F_SETLK64`/`F_SETLKW64` | 12/13/14 | `struct flock64*` | 32-bit-arch wide-offset variants |
| `F_SETOWN_EX` | 15 | `struct f_owner_ex*` | set owner with explicit type (TID/PID/PGRP) |
| `F_GETOWN_EX` | 16 | `struct f_owner_ex*` | get owner with type |
| `F_GETOWNER_UIDS` | 17 | `uid_t[2]*` | get owner's UIDs (uid, euid) |
| `F_OFD_GETLK`  | 36 | `struct flock*` | Open File Description lock query |
| `F_OFD_SETLK`  | 37 | `struct flock*` | OFD lock (non-blocking) |
| `F_OFD_SETLKW` | 38 | `struct flock*` | OFD lock (blocking) |

### Seal bits (`F_ADD_SEALS` / `F_GET_SEALS`)

| Constant | Hex | Meaning |
|---|---|---|
| `F_SEAL_SEAL`         | `0x0001` | prevent further seals from being set |
| `F_SEAL_SHRINK`       | `0x0002` | prevent file from shrinking |
| `F_SEAL_GROW`         | `0x0004` | prevent file from growing |
| `F_SEAL_WRITE`        | `0x0008` | prevent writes |
| `F_SEAL_FUTURE_WRITE` | `0x0010` | prevent future writes while existing mappings stay writable |
| `F_SEAL_EXEC`         | `0x0020` | prevent chmod from modifying the exec bits |
| (reserved)            | `1<<31`  | reserved for signed error codes |

### Write-life-time hint values (`F_{GET,SET}_RW_HINT`, `F_{GET,SET}_FILE_RW_HINT`)

| Constant | Value | Meaning |
|---|---|---|
| `RWH_WRITE_LIFE_NOT_SET`  | 0 | no hint (or clear) |
| `RWH_WRITE_LIFE_NONE`     | 1 | no useful temporal hint |
| `RWH_WRITE_LIFE_SHORT`    | 2 | short-lived |
| `RWH_WRITE_LIFE_MEDIUM`   | 3 | medium-lived |
| `RWH_WRITE_LIFE_LONG`     | 4 | long-lived |
| `RWH_WRITE_LIFE_EXTREME`  | 5 | extremely long-lived |
| `RWF_WRITE_LIFE_NOT_SET`  | (alias of `RWH_WRITE_LIFE_NOT_SET`, for legacy spelling) |

### Lock types (used in `struct flock.l_type` and lease commands)

| Constant | Value | Meaning |
|---|---|---|
| `F_RDLCK` | 0 | shared read lock |
| `F_WRLCK` | 1 | exclusive write lock |
| `F_UNLCK` | 2 | release lock |
| `F_EXLCK` | 4 | (BSD legacy) exclusive flock |
| `F_SHLCK` | 8 | (BSD legacy) shared flock |

### Owner types (used in `struct f_owner_ex.type`)

| Constant | Value | Meaning |
|---|---|---|
| `F_OWNER_TID`  | 0 | per-thread (TID) |
| `F_OWNER_PID`  | 1 | per-process |
| `F_OWNER_PGRP` | 2 | per-process-group |

### Dnotify event bits (`F_NOTIFY`)

| Constant | Hex | Meaning |
|---|---|---|
| `DN_ACCESS`     | `0x00000001` | file accessed |
| `DN_MODIFY`     | `0x00000002` | file modified |
| `DN_CREATE`     | `0x00000004` | file created |
| `DN_DELETE`     | `0x00000008` | file removed |
| `DN_RENAME`     | `0x00000010` | file renamed |
| `DN_ATTRIB`     | `0x00000020` | attribute changed |
| `DN_MULTISHOT`  | `0x80000000` | persistent notifier (don't auto-remove) |

### Structures

```
/* asm-generic */
struct flock {
    short          l_type;        /* F_RDLCK / F_WRLCK / F_UNLCK */
    short          l_whence;      /* SEEK_SET / SEEK_CUR / SEEK_END */
    __kernel_off_t l_start;
    __kernel_off_t l_len;
    __kernel_pid_t l_pid;
    /* optional arch-specific __ARCH_FLOCK_EXTRA_SYSID / __ARCH_FLOCK_PAD */
};

struct flock64 {
    short            l_type;
    short            l_whence;
    __kernel_loff_t  l_start;
    __kernel_loff_t  l_len;
    __kernel_pid_t   l_pid;
    /* optional __ARCH_FLOCK64_PAD */
};

struct f_owner_ex {
    int             type;         /* F_OWNER_TID / F_OWNER_PID / F_OWNER_PGRP */
    __kernel_pid_t  pid;
};

/* linux/fcntl.h */
struct delegation {
    __u32 d_flags;                /* must be 0 (reserved) */
    __u16 d_type;                 /* F_RDLCK / F_WRLCK / F_UNLCK */
    __u16 __pad;                  /* must be 0 */
};

/* linux/fs.h, used by FICLONERANGE / FS_IOC_FSGETXATTR / FIDEDUPERANGE */
struct file_clone_range {
    __s64 src_fd;
    __u64 src_offset;
    __u64 src_length;
    __u64 dest_offset;
};

struct fsxattr {
    __u32         fsx_xflags;     /* XFS-derived attribute flags */
    __u32         fsx_extsize;    /* extent size hint */
    __u32         fsx_nextents;   /* count of extents (get-only) */
    __u32         fsx_projid;     /* project quota id */
    __u32         fsx_cowextsize; /* CoW extent size hint */
    unsigned char fsx_pad[8];
};

struct file_dedupe_range_info {
    __s64 dest_fd;                /* in */
    __u64 dest_offset;            /* in */
    __u64 bytes_deduped;          /* out */
    __s32 status;                 /* out: < 0 err / SAME / DIFFERS */
    __u32 reserved;               /* must be zero */
};

struct file_dedupe_range {
    __u64 src_offset;             /* in */
    __u64 src_length;             /* in */
    __u16 dest_count;             /* in */
    __u16 reserved1;              /* must be zero */
    __u32 reserved2;              /* must be zero */
    struct file_dedupe_range_info info[];
};
```

### `AT_*` flag namespace

| Flag | Hex | Consumed by |
|---|---|---|
| `AT_FDCWD`                 | `-100`   | every `*at(2)` (dirfd-relative-to-cwd sentinel) |
| `AT_SYMLINK_NOFOLLOW`      | `0x0100` | fstatat, fchownat, fchmodat, fexecveat, ... |
| `AT_EACCESS`               | `0x0200` | faccessat (effective-uid test) |
| `AT_REMOVEDIR`             | `0x0200` | unlinkat (overloaded: rmdir-mode) |
| `AT_HANDLE_FID`            | `0x0200` | name_to_handle_at (file-id-only handle) |
| `AT_SYMLINK_FOLLOW`        | `0x0400` | linkat (force-follow) |
| `AT_NO_AUTOMOUNT`          | `0x0800` | fstatat, statx, name_to_handle_at, ... |
| `AT_EMPTY_PATH`            | `0x1000` | fstatat, openat, name_to_handle_at, linkat, execveat |
| `AT_STATX_SYNC_TYPE`       | `0x6000` | statx (mask of below) |
| `AT_STATX_SYNC_AS_STAT`    | `0x0000` | statx |
| `AT_STATX_FORCE_SYNC`      | `0x2000` | statx |
| `AT_STATX_DONT_SYNC`       | `0x4000` | statx |
| `AT_RECURSIVE`             | `0x8000` | open_tree / mount_setattr |
| `AT_RENAME_NOREPLACE`      | `0x0001` | renameat2 (alias of RENAME_NOREPLACE) |
| `AT_RENAME_EXCHANGE`       | `0x0002` | renameat2 (alias of RENAME_EXCHANGE) |
| `AT_RENAME_WHITEOUT`       | `0x0004` | renameat2 (alias of RENAME_WHITEOUT) |
| `AT_HANDLE_MNT_ID_UNIQUE`  | `0x0001` | name_to_handle_at (return u64 unique mnt-id) |
| `AT_HANDLE_CONNECTABLE`    | `0x0002` | name_to_handle_at (request connectable handle) |
| `AT_EXECVE_CHECK`          | `0x10000`| execveat2 (check-only, no exec) |

### Reserved dirfd sentinels

| Constant | Value | Meaning |
|---|---|---|
| `AT_FDCWD`                | `-100`   | per-syscall: dirfd refers to cwd |
| `PIDFD_SELF_THREAD`       | `-10000` | current thread (kernel-internal usage) |
| `PIDFD_SELF_THREAD_GROUP` | `-10001` | current thread group leader |
| `FD_PIDFS_ROOT`           | `-10002` | root of pidfs |
| `FD_NSFS_ROOT`            | `-10003` | root of nsfs |
| `FD_INVALID`              | `-10009` | invalid fd sentinel (-10000 - EBADF) |

Reserved kernel ranges: `[-100]` and `[-10000, -40000]`.

### FD flag

| Constant | Value | Meaning |
|---|---|---|
| `FD_CLOEXEC` | `1` | close-on-exec (low bit; reserved bits MAY be added in future) |

## Compatibility contract

REQ-1: Linux-specific F_* numbering:
- `F_LINUX_SPECIFIC_BASE == 1024`.
- Each Linux command MUST be `F_LINUX_SPECIFIC_BASE + N` with the exact `N` from the table.
- Commands MUST NOT collide with POSIX commands (which live in `[0, 38]`).

REQ-2: `struct flock` layout:
- Field order: `l_type` (short), `l_whence` (short), `l_start` (off_t), `l_len` (off_t), `l_pid` (pid_t).
- Per-arch overrides (`__ARCH_FLOCK_EXTRA_SYSID`, `__ARCH_FLOCK_PAD`) MUST be honoured where the upstream kernel sets them; absence MUST also match (most arches).
- `l_whence ∈ {SEEK_SET, SEEK_CUR, SEEK_END}`; other values return `-EINVAL`.

REQ-3: `struct flock64` layout:
- Same layout as `flock` but with `__kernel_loff_t` (`__s64`) for `l_start` and `l_len`.

REQ-4: `struct f_owner_ex` layout:
- `type` (int) MUST be one of `F_OWNER_TID`, `F_OWNER_PID`, `F_OWNER_PGRP`; other values return `-EINVAL`.
- `pid` (pid_t) is the TID/PID/PGID per `type`.

REQ-5: `struct delegation` layout:
- `d_flags` MUST be 0 (currently reserved); kernel returns `-EINVAL` if non-zero.
- `d_type ∈ {F_RDLCK, F_WRLCK, F_UNLCK}`.
- `__pad` MUST be zero (must-be-zero-on-write contract).

REQ-6: `F_DUPFD` / `F_DUPFD_CLOEXEC` semantics:
- Returns the lowest available fd ≥ `arg`.
- `F_DUPFD_CLOEXEC` sets `FD_CLOEXEC` atomically (race-free vs concurrent fork/exec).
- New fd refers to the SAME open file description (file offset + flags shared).
- Returns `-EINVAL` if `arg < 0`; `-EMFILE` if no fd ≤ `RLIMIT_NOFILE` is available.

REQ-7: `F_GETFD` / `F_SETFD` semantics:
- Only `FD_CLOEXEC` (= 1) is exposed today; the kernel ignores higher bits on `F_SETFD` (forward-compatibility) but MUST mask the return value of `F_GETFD` to the bits it knows.

REQ-8: `F_GETFL` / `F_SETFL` semantics:
- `F_GETFL` returns the full `O_*` flags as they appear in `struct file::f_flags`.
- `F_SETFL` is restricted: only `O_APPEND`, `O_NONBLOCK`, `O_ASYNC`, `O_DIRECT`, `O_NOATIME` (where supported) MAY be modified; other bits are silently ignored.

REQ-9: `F_GETLK` / `F_SETLK` / `F_SETLKW` semantics:
- Process-associated POSIX advisory locks.
- `F_GETLK` returns `l_type = F_UNLCK` if no conflict, otherwise overwrites with the conflicting lock's details.
- `F_SETLK` returns `-EAGAIN` (or `-EACCES`) when a conflicting lock is held; non-blocking.
- `F_SETLKW` blocks; interruptible by signal (returns `-EINTR`).
- All process-locks are released on ANY `close()` of any fd referring to the file (POSIX historical wart).

REQ-10: `F_OFD_*` semantics:
- Open File Description locks: tied to the open file description, not the process.
- NOT released on `close()` of other fds for the same file; ARE released when the last fd referring to the OFD is closed.
- Inherited across `fork()` (BSD-flock-like).

REQ-11: `F_SETOWN` / `F_GETOWN`:
- Sets/gets the recipient of SIGIO/SIGURG for the fd.
- Positive arg = pid; negative arg = process-group `-pgid` (legacy encoding).

REQ-12: `F_SETOWN_EX` / `F_GETOWN_EX`:
- Use `struct f_owner_ex` to avoid the sign-overload of `F_SETOWN`.

REQ-13: `F_GETOWNER_UIDS`:
- Writes a `[uid, euid]` pair into the user buffer; supports security daemons inspecting which UID will receive notifications.

REQ-14: `F_NOTIFY`:
- Legacy dnotify on a directory fd; arg = OR of `DN_*` bits.
- Deprecated in favor of inotify(7)/fanotify(7); MUST continue to work for ABI but new code SHOULD NOT use it.
- `DN_MULTISHOT` keeps the notifier registered across delivery.

REQ-15: `F_DUPFD_QUERY`:
- Compares two fds; returns 1 if both refer to the same open file description, 0 otherwise.

REQ-16: `F_CREATED_QUERY`:
- Reports whether the file was just created by the `open(O_CREAT)` that produced this fd (single-shot; reset by `dup`).

REQ-17: `F_CANCELLK`:
- Internal-use-only cancel of a blocking POSIX lock; not documented for general userspace.

REQ-18: `F_SETPIPE_SZ` / `F_GETPIPE_SZ`:
- `arg` is rounded up to the next power of two and clamped to `[PAGE_SIZE, /proc/sys/fs/pipe-max-size]`.
- Unprivileged callers cannot exceed `pipe-max-size`; `CAP_SYS_RESOURCE` lifts the upper bound.
- `F_GETPIPE_SZ` returns the current capacity in bytes.

REQ-19: `F_ADD_SEALS` / `F_GET_SEALS`:
- Operates ONLY on memfd-backed inodes (created via `memfd_create(2)` with `MFD_ALLOW_SEALING`).
- `F_ADD_SEALS` is monotonic: bits can only be added, never removed.
- Returns `-EPERM` if `F_SEAL_SEAL` is already set.
- Returns `-EBUSY` if existing writable mappings prevent applying `F_SEAL_WRITE` (use `F_SEAL_FUTURE_WRITE` if they should remain).
- Returns `-EINVAL` for unknown seal bits.

REQ-20: `F_{GET,SET}_RW_HINT` / `F_{GET,SET}_FILE_RW_HINT`:
- Arg is `__u64*` (5-bit value space `0..5`); reserved values return `-EINVAL`.
- `_RW_HINT` modifies the underlying inode (shared across all opens).
- `_FILE_RW_HINT` modifies the `struct file` only.

REQ-21: `F_GETDELEG` / `F_SETDELEG`:
- Argument is `struct delegation*`; `d_flags` MUST be 0, `__pad` MUST be 0.
- `d_type` indicates read/write/none.

REQ-22: AT_FDCWD value:
- MUST be exactly `-100` (signed-int).
- Userspace and the kernel agree to use this magic value to indicate "use the calling process's cwd".

REQ-23: `AT_EACCESS` / `AT_REMOVEDIR` / `AT_HANDLE_FID` overload:
- All three are `0x200` but disambiguate by syscall:
  - faccessat: `AT_EACCESS`.
  - unlinkat: `AT_REMOVEDIR`.
  - name_to_handle_at: `AT_HANDLE_FID`.
- Passing the wrong one to a syscall is "clearly wrong" undefined behaviour and the kernel MAY treat it as the syscall's interpretation.

REQ-24: `AT_STATX_SYNC_TYPE` mask = `0x6000`:
- Mutually exclusive bits `AT_STATX_FORCE_SYNC` and `AT_STATX_DONT_SYNC`.
- Both set ⟹ `-EINVAL`.

REQ-25: `AT_EMPTY_PATH`:
- When set, the kernel accepts `pathname = ""` and operates on `dirfd` directly.
- Requires `dirfd` not be `AT_FDCWD` (since cwd has no associated fd).

REQ-26: `AT_NO_AUTOMOUNT`:
- Suppress the automount trigger when traversing a path that crosses a Direct mountpoint (autofs).

REQ-27: `AT_RECURSIVE`:
- Apply the operation to the entire subtree (used by `open_tree(2)`, `mount_setattr(2)`).

REQ-28: `AT_RENAME_*` flags = legacy `RENAME_*`:
- `AT_RENAME_NOREPLACE` = `0x01`, `AT_RENAME_EXCHANGE` = `0x02`, `AT_RENAME_WHITEOUT` = `0x04`.
- NOREPLACE and EXCHANGE are mutually exclusive.

REQ-29: `AT_HANDLE_*` flags for name_to_handle_at:
- `AT_HANDLE_MNT_ID_UNIQUE` = `0x001` (return u64 unique mount ID).
- `AT_HANDLE_CONNECTABLE` = `0x002` (request connectable file handle).
- `AT_HANDLE_FID` = `0x200` (file-handle is fid-only; cannot be reopened).

REQ-30: `AT_EXECVE_CHECK`:
- For `execveat2(2)`: only test whether exec would be permitted; do not actually replace the image.

REQ-31: `struct file_clone_range`, `struct fsxattr`, `struct file_dedupe_range` layouts:
- Pinned by ioctl ABI; sizes and field offsets MUST match upstream.
- `file_dedupe_range::info` is a flexible array of `file_dedupe_range_info`; the caller specifies `dest_count`.

REQ-32: Reserved-must-be-zero fields:
- `delegation::__pad`, `delegation::d_flags` (currently), `file_dedupe_range::reserved1`, `file_dedupe_range::reserved2`, `file_dedupe_range_info::reserved`, `fsxattr::fsx_pad[8]` MUST be zero on entry; kernel rejects with `-EINVAL` if non-zero (or silently zeroes on return).

## Acceptance Criteria

- [ ] AC-1: All `F_*` cmd values match upstream exactly (table verified by `static_assert` / `const _: () = assert!`).
- [ ] AC-2: `F_LINUX_SPECIFIC_BASE == 1024`.
- [ ] AC-3: `sizeof(struct flock)`, `sizeof(struct flock64)`, `sizeof(struct f_owner_ex)`, `sizeof(struct delegation)`, `sizeof(struct file_clone_range)`, `sizeof(struct fsxattr)`, `sizeof(struct file_dedupe_range)`, `sizeof(struct file_dedupe_range_info)` match upstream on every arch.
- [ ] AC-4: `F_DUPFD_CLOEXEC` produces an fd with `FD_CLOEXEC` set (verified atomically vs `fork()` race).
- [ ] AC-5: `F_GETFD` returns only documented bits (today: only `FD_CLOEXEC`).
- [ ] AC-6: `F_SETFL` silently ignores bits other than `O_APPEND | O_NONBLOCK | O_ASYNC | O_DIRECT | O_NOATIME`.
- [ ] AC-7: `F_SETLK` returns `-EAGAIN`/`-EACCES` on conflict; `F_SETLKW` blocks and returns `-EINTR` on signal.
- [ ] AC-8: POSIX lock released on ANY `close()` of any fd for the file; OFD lock released only when last fd to the OFD closes.
- [ ] AC-9: `F_SETPIPE_SZ` clamps to `[PAGE_SIZE, pipe-max-size]`; rounds up to power of two; unprivileged cannot exceed `pipe-max-size`.
- [ ] AC-10: `F_ADD_SEALS` is monotonic; rejects unknown bits with `-EINVAL`; rejects after `F_SEAL_SEAL` with `-EPERM`.
- [ ] AC-11: `F_SET_RW_HINT` rejects values > `RWH_WRITE_LIFE_EXTREME` with `-EINVAL`.
- [ ] AC-12: `AT_FDCWD == -100`.
- [ ] AC-13: `AT_STATX_SYNC_TYPE == 0x6000`; passing `AT_STATX_FORCE_SYNC | AT_STATX_DONT_SYNC` returns `-EINVAL`.
- [ ] AC-14: `AT_EMPTY_PATH` with `dirfd == AT_FDCWD` returns `-EBADF`.
- [ ] AC-15: `renameat2(_, _, _, _, AT_RENAME_NOREPLACE | AT_RENAME_EXCHANGE)` returns `-EINVAL`.
- [ ] AC-16: `delegation::__pad` and `delegation::d_flags` non-zero ⟹ `-EINVAL`.
- [ ] AC-17: Round-trip: glibc-static `flock(2)` / `fcntl(2)` test suite passes against Rookery byte-identical to upstream.

## Architecture

```
pub mod uapi::fcntl {

    pub const F_LINUX_SPECIFIC_BASE: i32 = 1024;

    pub mod cmd {
        // POSIX (from asm-generic/fcntl.h)
        pub const DUPFD:          i32 = 0;
        pub const GETFD:          i32 = 1;
        pub const SETFD:          i32 = 2;
        pub const GETFL:          i32 = 3;
        pub const SETFL:          i32 = 4;
        pub const GETLK:          i32 = 5;
        pub const SETLK:          i32 = 6;
        pub const SETLKW:         i32 = 7;
        pub const SETOWN:         i32 = 8;
        pub const GETOWN:         i32 = 9;
        pub const SETSIG:         i32 = 10;
        pub const GETSIG:         i32 = 11;
        pub const GETLK64:        i32 = 12;
        pub const SETLK64:        i32 = 13;
        pub const SETLKW64:       i32 = 14;
        pub const SETOWN_EX:      i32 = 15;
        pub const GETOWN_EX:      i32 = 16;
        pub const GETOWNER_UIDS:  i32 = 17;
        pub const OFD_GETLK:      i32 = 36;
        pub const OFD_SETLK:      i32 = 37;
        pub const OFD_SETLKW:     i32 = 38;

        // Linux-specific
        pub const SETLEASE:           i32 = super::F_LINUX_SPECIFIC_BASE + 0;
        pub const GETLEASE:           i32 = super::F_LINUX_SPECIFIC_BASE + 1;
        pub const NOTIFY:             i32 = super::F_LINUX_SPECIFIC_BASE + 2;
        pub const DUPFD_QUERY:        i32 = super::F_LINUX_SPECIFIC_BASE + 3;
        pub const CREATED_QUERY:      i32 = super::F_LINUX_SPECIFIC_BASE + 4;
        pub const CANCELLK:           i32 = super::F_LINUX_SPECIFIC_BASE + 5;
        pub const DUPFD_CLOEXEC:      i32 = super::F_LINUX_SPECIFIC_BASE + 6;
        pub const SETPIPE_SZ:         i32 = super::F_LINUX_SPECIFIC_BASE + 7;
        pub const GETPIPE_SZ:         i32 = super::F_LINUX_SPECIFIC_BASE + 8;
        pub const ADD_SEALS:          i32 = super::F_LINUX_SPECIFIC_BASE + 9;
        pub const GET_SEALS:          i32 = super::F_LINUX_SPECIFIC_BASE + 10;
        pub const GET_RW_HINT:        i32 = super::F_LINUX_SPECIFIC_BASE + 11;
        pub const SET_RW_HINT:        i32 = super::F_LINUX_SPECIFIC_BASE + 12;
        pub const GET_FILE_RW_HINT:   i32 = super::F_LINUX_SPECIFIC_BASE + 13;
        pub const SET_FILE_RW_HINT:   i32 = super::F_LINUX_SPECIFIC_BASE + 14;
        pub const GETDELEG:           i32 = super::F_LINUX_SPECIFIC_BASE + 15;
        pub const SETDELEG:           i32 = super::F_LINUX_SPECIFIC_BASE + 16;
    }

    pub mod lock_type {
        pub const F_RDLCK: i16 = 0;
        pub const F_WRLCK: i16 = 1;
        pub const F_UNLCK: i16 = 2;
        pub const F_EXLCK: i16 = 4;
        pub const F_SHLCK: i16 = 8;
    }

    pub mod owner_type {
        pub const F_OWNER_TID:  i32 = 0;
        pub const F_OWNER_PID:  i32 = 1;
        pub const F_OWNER_PGRP: i32 = 2;
    }

    pub mod seal {
        pub const F_SEAL_SEAL:         u32 = 0x0001;
        pub const F_SEAL_SHRINK:       u32 = 0x0002;
        pub const F_SEAL_GROW:         u32 = 0x0004;
        pub const F_SEAL_WRITE:        u32 = 0x0008;
        pub const F_SEAL_FUTURE_WRITE: u32 = 0x0010;
        pub const F_SEAL_EXEC:         u32 = 0x0020;
    }

    pub mod rwh {
        pub const NOT_SET:  u64 = 0;
        pub const NONE:     u64 = 1;
        pub const SHORT:    u64 = 2;
        pub const MEDIUM:   u64 = 3;
        pub const LONG:     u64 = 4;
        pub const EXTREME:  u64 = 5;
    }

    pub mod dn {
        pub const ACCESS:    u32 = 0x0000_0001;
        pub const MODIFY:    u32 = 0x0000_0002;
        pub const CREATE:    u32 = 0x0000_0004;
        pub const DELETE:    u32 = 0x0000_0008;
        pub const RENAME:    u32 = 0x0000_0010;
        pub const ATTRIB:    u32 = 0x0000_0020;
        pub const MULTISHOT: u32 = 0x8000_0000;
    }

    pub const FD_CLOEXEC: i32 = 1;

    pub mod at {
        pub const FDCWD:                 i32 = -100;
        pub const SYMLINK_NOFOLLOW:      i32 = 0x0100;
        pub const EACCESS:               i32 = 0x0200;
        pub const REMOVEDIR:             i32 = 0x0200;  // overloaded
        pub const HANDLE_FID:            i32 = 0x0200;  // overloaded
        pub const SYMLINK_FOLLOW:        i32 = 0x0400;
        pub const NO_AUTOMOUNT:          i32 = 0x0800;
        pub const EMPTY_PATH:            i32 = 0x1000;
        pub const STATX_SYNC_TYPE:       i32 = 0x6000;
        pub const STATX_SYNC_AS_STAT:    i32 = 0x0000;
        pub const STATX_FORCE_SYNC:      i32 = 0x2000;
        pub const STATX_DONT_SYNC:       i32 = 0x4000;
        pub const RECURSIVE:             i32 = 0x8000;
        pub const RENAME_NOREPLACE:      i32 = 0x0001;
        pub const RENAME_EXCHANGE:       i32 = 0x0002;
        pub const RENAME_WHITEOUT:       i32 = 0x0004;
        pub const HANDLE_MNT_ID_UNIQUE:  i32 = 0x0001;
        pub const HANDLE_CONNECTABLE:    i32 = 0x0002;
        pub const EXECVE_CHECK:          i32 = 0x10000;
    }

    pub mod dirfd_special {
        pub const PIDFD_SELF_THREAD:       i32 = -10000;
        pub const PIDFD_SELF_THREAD_GROUP: i32 = -10001;
        pub const FD_PIDFS_ROOT:           i32 = -10002;
        pub const FD_NSFS_ROOT:            i32 = -10003;
        pub const FD_INVALID:              i32 = -10009;
    }

    // Structures ---------------------------------------------------------

    #[repr(C)]
    pub struct Flock {
        pub l_type:   i16,
        pub l_whence: i16,
        pub l_start:  i64,   // __kernel_off_t (LP64) or i32 on LP32 (per-arch)
        pub l_len:    i64,
        pub l_pid:    i32,
    }

    #[repr(C)]
    pub struct Flock64 {
        pub l_type:   i16,
        pub l_whence: i16,
        pub l_start:  i64,   // __kernel_loff_t = __s64
        pub l_len:    i64,
        pub l_pid:    i32,
    }

    #[repr(C)]
    pub struct FOwnerEx {
        pub type_: i32,      // F_OWNER_TID/PID/PGRP
        pub pid:   i32,      // __kernel_pid_t
    }

    #[repr(C)]
    pub struct Delegation {
        pub d_flags: u32,    // must be 0
        pub d_type:  u16,    // F_RDLCK/F_WRLCK/F_UNLCK
        pub __pad:   u16,    // must be 0
    }
}

pub mod uapi::fs {
    #[repr(C)]
    pub struct FileCloneRange {
        pub src_fd:      i64,
        pub src_offset:  u64,
        pub src_length:  u64,
        pub dest_offset: u64,
    }

    #[repr(C)]
    pub struct Fsxattr {
        pub fsx_xflags:     u32,
        pub fsx_extsize:    u32,
        pub fsx_nextents:   u32,
        pub fsx_projid:     u32,
        pub fsx_cowextsize: u32,
        pub fsx_pad:        [u8; 8],
    }

    #[repr(C)]
    pub struct FileDedupeRangeInfo {
        pub dest_fd:       i64,
        pub dest_offset:   u64,
        pub bytes_deduped: u64,
        pub status:        i32,
        pub reserved:      u32,
    }

    #[repr(C)]
    pub struct FileDedupeRange {
        pub src_offset:  u64,
        pub src_length:  u64,
        pub dest_count:  u16,
        pub reserved1:   u16,
        pub reserved2:   u32,
        // followed by `info[dest_count]`
    }
}
```

`uapi::fcntl::Flock::validate(&self) -> Result<(), Errno>`:
1. if self.l_type ∉ { F_RDLCK, F_WRLCK, F_UNLCK }: return Err(EINVAL).
2. if self.l_whence ∉ { SEEK_SET, SEEK_CUR, SEEK_END }: return Err(EINVAL).
3. if self.l_len < 0 ∧ self.l_start.checked_add(self.l_len).is_none(): return Err(EOVERFLOW).
4. return Ok(()).

`uapi::fcntl::FOwnerEx::validate(&self) -> Result<(), Errno>`:
1. if self.type_ ∉ { F_OWNER_TID, F_OWNER_PID, F_OWNER_PGRP }: return Err(EINVAL).
2. return Ok(()).

`uapi::fcntl::Delegation::validate(&self) -> Result<(), Errno>`:
1. if self.d_flags != 0: return Err(EINVAL).
2. if self.__pad   != 0: return Err(EINVAL).
3. if self.d_type ∉ { F_RDLCK as u16, F_WRLCK as u16, F_UNLCK as u16 }: return Err(EINVAL).
4. return Ok(()).

`uapi::fcntl::rwh::validate(v: u64) -> Result<(), Errno>`:
1. if v > rwh::EXTREME: return Err(EINVAL).
2. return Ok(()).

`uapi::fcntl::at::validate_statx_sync(flags: i32) -> Result<(), Errno>`:
1. let sync = flags & at::STATX_SYNC_TYPE;
2. if sync == at::STATX_SYNC_TYPE: return Err(EINVAL).
3. return Ok(()).

`uapi::fcntl::at::validate_rename(flags: i32) -> Result<(), Errno>`:
1. if (flags & at::RENAME_NOREPLACE != 0) ∧ (flags & at::RENAME_EXCHANGE != 0):
   - return Err(EINVAL).
2. return Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `f_cmd_values_pinned`              | LAYOUT     | every `cmd::*` constant matches upstream numeric value. |
| `f_linux_specific_base_is_1024`    | LAYOUT     | `F_LINUX_SPECIFIC_BASE == 1024`. |
| `flock_layout_pinned`              | LAYOUT     | `sizeof(Flock)`, `sizeof(Flock64)`, `sizeof(FOwnerEx)`, `sizeof(Delegation)` match upstream. |
| `flock_validate_rejects_bad_type`  | INVARIANT  | `l_type ∉ {RDLCK,WRLCK,UNLCK} ⟹ EINVAL`. |
| `flock_validate_rejects_bad_whence`| INVARIANT  | `l_whence ∉ {SET,CUR,END} ⟹ EINVAL`. |
| `f_owner_ex_validate`              | INVARIANT  | `type_ ∉ {TID,PID,PGRP} ⟹ EINVAL`. |
| `delegation_pad_must_be_zero`      | INVARIANT  | `d_flags != 0 ∨ __pad != 0 ⟹ EINVAL`. |
| `seal_monotonic`                   | INVARIANT  | seal mask after `F_ADD_SEALS` is a superset of mask before. |
| `seal_seal_freezes`                | INVARIANT  | `F_SEAL_SEAL set ⟹ further F_ADD_SEALS returns EPERM`. |
| `rwh_bounded`                      | INVARIANT  | `v > rwh::EXTREME ⟹ EINVAL`. |
| `at_statx_sync_mutual_exclusion`   | INVARIANT  | `flags & STATX_SYNC_TYPE == STATX_SYNC_TYPE ⟹ EINVAL`. |
| `at_rename_exclusion`              | INVARIANT  | `NOREPLACE ∧ EXCHANGE ⟹ EINVAL`. |
| `dupfd_cloexec_atomic`             | INVARIANT  | `F_DUPFD_CLOEXEC ⟹ FD_CLOEXEC set before any concurrent fork can observe the new fd`. |
| `setpipe_sz_clamped`               | INVARIANT  | resulting size ∈ `[PAGE_SIZE, pipe-max-size]`, power-of-two. |
| `posix_lock_released_on_any_close` | INVARIANT  | per-POSIX-lock: any close of any fd to inode releases lock. |
| `ofd_lock_per_open_file_description` | INVARIANT | per-OFD-lock: released only when last fd referring to the OFD closes. |

### Layer 2: TLA+

`uapi/fcntl-locks.tla`:
- Models POSIX vs OFD lock acquisition / release across fork+close interleavings.
- States: per-(file, range) lock owner set, per-process fd set, per-OFD fd set.
- Properties:
  - `safety_posix_lock_released_on_any_close` — POSIX lock invariant.
  - `safety_ofd_lock_survives_other_close` — OFD lock invariant.
  - `safety_no_double_grant` — no two conflicting locks overlap.
  - `liveness_setlkw_completes_or_eintr` — blocking lock terminates.

`uapi/fcntl-seals.tla`:
- Models memfd seal monotonicity.
- States: per-memfd seal mask, per-mapping writability.
- Properties:
  - `safety_seal_monotonic` — mask is monotonically non-decreasing.
  - `safety_seal_seal_freezes` — after `F_SEAL_SEAL`, no further seals.
  - `safety_seal_write_blocks_new_writes` — `F_SEAL_WRITE ⟹ EPERM on write`.
  - `safety_seal_future_write_keeps_old_maps` — `F_SEAL_FUTURE_WRITE ⟹ existing writable maps remain`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Flock::validate` post: `ret.is_ok() ⟹ l_type ∈ {RDLCK,WRLCK,UNLCK} ∧ l_whence ∈ {SET,CUR,END}` | `Flock::validate` |
| `FOwnerEx::validate` post: `ret.is_ok() ⟹ type_ ∈ {TID,PID,PGRP}` | `FOwnerEx::validate` |
| `Delegation::validate` post: `ret.is_ok() ⟹ d_flags == 0 ∧ __pad == 0 ∧ d_type ∈ {RDLCK,WRLCK,UNLCK}` | `Delegation::validate` |
| `rwh::validate` post: `ret.is_ok() ⟹ v ≤ EXTREME` | `rwh::validate` |
| `at::validate_statx_sync` post: `ret.is_ok() ⟹ (flags & STATX_SYNC_TYPE) != STATX_SYNC_TYPE` | `at::validate_statx_sync` |
| `at::validate_rename` post: `ret.is_ok() ⟹ !(NOREPLACE ∧ EXCHANGE)` | `at::validate_rename` |
| `F_ADD_SEALS` post: new mask ⊇ old mask; rejects unknown bits; rejects after F_SEAL_SEAL | `Fcntl::add_seals` |
| `F_DUPFD_CLOEXEC` post: result fd has FD_CLOEXEC set | `Fcntl::dupfd_cloexec` |
| `F_SETPIPE_SZ` post: result_size ∈ `[PAGE_SIZE, pipe-max-size]` ∧ pow2 | `Fcntl::set_pipe_sz` |

### Layer 4: Verus/Creusot functional

`Per-fcntl call → cmd dispatch → arg validate → per-cmd handler (lock/seal/owner/pipe/hint/deleg/...) → copy_to_user (if applicable) → return rc` byte-identical to upstream Linux at the same baseline, including:
- POSIX vs OFD lock semantics interleaved with fork/close.
- memfd seal monotonicity and `F_SEAL_FUTURE_WRITE` mapping survival.
- `F_DUPFD_CLOEXEC` race-free atomic FD_CLOEXEC.
- `F_SETPIPE_SZ` clamping, rounding, capability gating.
- `*at(2)` flag dispatch across faccessat, unlinkat, name_to_handle_at, renameat2, statx, execveat2.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

fcntl ABI reinforcement:

- **Per-`F_DUPFD_CLOEXEC` atomic install** — defense against per-fork-leak-of-pre-CLOEXEC-fd.
- **Per-`F_ADD_SEALS` monotonic + `F_SEAL_SEAL` freeze** — defense against per-seal-rollback / re-mutation.
- **Per-`struct flock` `l_type`/`l_whence` validated** — defense against per-OOB-lock or per-EOVERFLOW.
- **Per-`struct f_owner_ex.type` validated** — defense against per-bogus-owner-type SIGIO mis-delivery.
- **Per-`struct delegation` `__pad` / `d_flags` zero-check** — defense against per-uninitialised-struct probe.
- **Per-`F_SETPIPE_SZ` clamped to `pipe-max-size`** — defense against per-unprivileged-pipe-memory-DoS.
- **Per-`F_SETLKW` signal-interruptible** — defense against per-uninterruptible-blocking-lock starvation.
- **Per-POSIX-lock released on ANY close** — defense against per-stuck-lock-on-fork-and-close.
- **Per-OFD-lock tied to open file description** — defense against per-spurious-lock-loss.
- **Per-write-life-time-hint range-checked** — defense against per-out-of-range-hint scheduler confusion.
- **Per-`AT_*` flag mutual-exclusion validation** — defense against per-undefined-syscall behaviour.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF — `struct flock` / `flock64` / `f_owner_ex` / `delegation` range validation** — every copy_from_user of a lock or owner struct MUST pass through UDEREF; `l_start` and `l_len` MUST be checked for `l_start >= 0 ∧ l_start + l_len` does not overflow `off_t` BEFORE any locking-tree mutation. Out-of-range returns `-EOVERFLOW` rather than wrapping.
- **GRKERNSEC_FIFO — `F_SETPIPE_SZ` capacity limit + `CAP_SYS_ADMIN` enforcement** — unprivileged callers MUST NOT exceed `/proc/sys/fs/pipe-max-size`; raising past that limit requires `CAP_SYS_ADMIN` in the calling user-namespace AND the init user-namespace, mirroring grsec's restriction on FIFO buffer growth. Also: a per-user accounting ceiling (sum of pipe buffers owned by the UID) is enforced via `RLIMIT_PIPESIZE` (Rookery-ext) so a single user cannot allocate gigabytes of pipe buffers.
- **GRKERNSEC_FORKBOMB — `F_SETPIPE_SZ` memory accounting against `RLIMIT_AS` / memcg** — pipe-buffer pages charged to the calling task's memcg and counted against the per-user pipe-buffer total; over-limit returns `-EPERM` not silent truncation. Combined with `F_GETPIPE_SZ` reporting the effective (post-clamp) size so userspace cannot be fooled.
- **`F_ADD_SEALS` for memfd hardening** — Rookery REQUIRES that any memfd handed to a less-privileged process via SCM_RIGHTS or `pidfd_getfd` carry a non-trivial seal mask (at minimum `F_SEAL_SHRINK | F_SEAL_GROW | F_SEAL_WRITE`) when used as a shared-memory IPC primitive; the LSM hook `lsm_memfd_post_share` enforces this in privileged-sender mode.
- **Per-fd `CAP_SETFCAP` gate on `F_DUPFD_CLOEXEC` across user-namespace** — when a `pidfd_getfd`-acquired fd is duped into a process in a different user-namespace, `F_DUPFD_CLOEXEC` MUST verify the destination namespace has `CAP_SETFCAP` (or equivalent) before duplicating an fd that carries file capabilities or LSM labels.
- **`F_NOTIFY` (legacy dnotify) deprecated — inotify(7)/fanotify(7) preferred** — Rookery keeps the ABI working for legacy programs but emits a one-shot warn-on-first-use dmesg per-pid (rate-limited) AND a `prctl(PR_SET_NO_DNOTIFY)` knob that turns the command into `-ENOSYS` for that process tree. CONFIG_FCNTL_DNOTIFY can be disabled at build time.
- **Per-fcntl audit hook** — every `fcntl(fd, cmd, arg)` is logged via `audit_fcntl(fd, cmd, arg)` before per-cmd handler dispatch; LSM modules can rate-limit, deny, or fingerprint per-cmd usage (e.g. detect a process that suddenly issues `F_ADD_SEALS` on an unexpected memfd).
- **PaX USERCOPY — `struct flock` / `flock64` / `f_owner_ex` are slab/stack whitelisted** — staging buffers for these structs live in size-pinned slab caches (`flock_cache`, `flock64_cache`, `f_owner_ex_cache`) or on the kernel stack with `sizeof(...)` as a compile-time constant so PAX_USERCOPY's whitelist check fires on any mismatched length.
- **`F_OFD_*` lock-owner sanitised on fork** — child process inherits OFD locks but the kernel MUST NOT leak parent-only metadata via `F_OFD_GETLK` (the returned `l_pid` is the owning PID of the lock when first acquired; in a user-namespace, it's translated to that ns's pid view, returning 0 if untranslatable).
- **`F_SETOWN` / `F_SETOWN_EX` SIGIO target capability check** — the caller MUST have permission to send `SIGIO` to the target task (UID/EUID match OR `CAP_KILL`); otherwise `-EPERM`, defense against unauthorised cross-process signal channels.
- **`F_GETOWNER_UIDS` user-namespace translation** — UID and EUID returned MUST be translated through the caller's user-namespace; out-of-range UIDs return `OVERFLOW_UID` (65534), not raw kernel UIDs.
- **`F_SETLKW` cancellable on user-namespace shutdown / cgroup-freeze** — blocking lock waits MUST be cancellable when the calling task's user-namespace is destroyed or its freezer cgroup is frozen, preventing per-orphaned-blocking-lock DoS.
- **`AT_EMPTY_PATH` requires source fd reference accounting** — when used (e.g. `linkat(AT_FDCWD, fd, target_dirfd, "newname", AT_EMPTY_PATH)`), the fd MUST be referenced via `fget_raw_pos` so that a concurrent `close(fd)` cannot race the `*at` syscall into operating on a recycled fd.
- **Per-`AT_RECURSIVE` capability check** — recursive mount-setattr / open_tree require `CAP_SYS_ADMIN` in the mount-namespace; cross-namespace recursion is rejected to prevent per-subtree privilege smuggling.

## Open Questions

- Should Rookery add a `prctl(PR_DENY_F_NOTIFY)` that hard-fails dnotify for the process tree (in addition to the per-fd opt-out)? Pending input from LSM consumers.
- Whether to expose a `F_GET_PIPE_BUDGET` introspection cmd showing the per-user pipe-buffer ceiling, so userspace tooling can adapt before `F_SETPIPE_SZ` returns `-EPERM`.
- Whether `F_OFD_GETLK` should expose a "kernel-internal lock cookie" for tracing / fairness analysis (currently no equivalent in upstream).

## Out of Scope

- `fs/locks.c` POSIX/OFD lock implementation (covered in `fs/locks.md` Tier-3).
- `mm/memfd.c` memfd seal enforcement (covered in `mm/memfd.md` Tier-3).
- `fs/pipe.c` pipe buffer sizing (covered in `fs/pipe.md` Tier-3).
- `fs/notify/dnotify/dnotify.c` legacy dnotify driver (covered in `fs/notify/dnotify.md` Tier-3).
- `fs/open.c` `dup` / `dup2` / `dup3` implementation (covered in `fs/open.md` Tier-3).
- `<asm/fcntl.h>` per-architecture overrides (covered in `uapi/headers/asm-fcntl.md` Tier-5 once written).
- `<linux/openat2.h>` and the `RESOLVE_*` flags (covered in `uapi/headers/openat2.md` Tier-5 once written).
- Implementation code.
