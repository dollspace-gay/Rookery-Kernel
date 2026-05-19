# Tier-5 UAPI: include/uapi/linux/mount.h — mount(2) / fsopen(2) / fsmount(2) ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/mount.h (~245 lines)
-->

## Summary

The mount UAPI is the syscall-edge contract for the entire VFS mount surface: the legacy `mount(2)` plus the post-5.2 "new mount API" (`fsopen(2)`, `fsconfig(2)`, `fsmount(2)`, `fspick(2)`, `move_mount(2)`, `open_tree(2)`, `mount_setattr(2)`) plus the `statmount(2)` / `listmount(2)` introspection pair. It defines:

- **Legacy `mount(2)` flags `MS_*`** — `MS_RDONLY`, `MS_NOSUID`, `MS_NODEV`, `MS_NOEXEC`, `MS_SYNCHRONOUS`, `MS_REMOUNT`, `MS_MANDLOCK`, `MS_DIRSYNC`, `MS_NOSYMFOLLOW`, `MS_NOATIME`, `MS_NODIRATIME`, `MS_BIND`, `MS_MOVE`, `MS_REC`, `MS_VERBOSE`/`MS_SILENT`, `MS_POSIXACL`, `MS_UNBINDABLE`, `MS_PRIVATE`, `MS_SLAVE`, `MS_SHARED`, `MS_RELATIME`, `MS_KERNMOUNT`, `MS_I_VERSION`, `MS_STRICTATIME`, `MS_LAZYTIME`, plus the kernel-internal `MS_SUBMOUNT`, `MS_NOREMOTELOCK`, `MS_NOSEC`, `MS_BORN`, `MS_ACTIVE`, `MS_NOUSER`. Plus the remount mask `MS_RMT_MASK` and the obsolete-but-still-recognized magic `MS_MGC_VAL` / `MS_MGC_MSK`.
- **`open_tree(2)` flags** `OPEN_TREE_CLONE`, `OPEN_TREE_NAMESPACE`, `OPEN_TREE_CLOEXEC`.
- **`move_mount(2)` flags** `MOVE_MOUNT_F_SYMLINKS`/`F_AUTOMOUNTS`/`F_EMPTY_PATH` (source-side), `MOVE_MOUNT_T_SYMLINKS`/`T_AUTOMOUNTS`/`T_EMPTY_PATH` (destination-side), `MOVE_MOUNT_SET_GROUP`, `MOVE_MOUNT_BENEATH`, and the validation mask `MOVE_MOUNT__MASK`.
- **`fsopen(2)` flags** `FSOPEN_CLOEXEC`.
- **`fspick(2)` flags** `FSPICK_CLOEXEC`, `FSPICK_SYMLINK_NOFOLLOW`, `FSPICK_NO_AUTOMOUNT`, `FSPICK_EMPTY_PATH`.
- **`fsconfig(2)` commands** `enum fsconfig_command` — `FSCONFIG_SET_FLAG`, `FSCONFIG_SET_STRING`, `FSCONFIG_SET_BINARY`, `FSCONFIG_SET_PATH`, `FSCONFIG_SET_PATH_EMPTY`, `FSCONFIG_SET_FD`, `FSCONFIG_CMD_CREATE`, `FSCONFIG_CMD_RECONFIGURE`, `FSCONFIG_CMD_CREATE_EXCL`.
- **`fsmount(2)` flags** `FSMOUNT_CLOEXEC`, `FSMOUNT_NAMESPACE`.
- **`mount_setattr(2)` attribute bits `MOUNT_ATTR_*`** — `RDONLY`, `NOSUID`, `NODEV`, `NOEXEC`, `NOSYMFOLLOW`, `IDMAP`, the `_ATIME` mask covering `RELATIME`/`NOATIME`/`STRICTATIME`/`NODIRATIME`, and the corresponding `struct mount_attr` (`attr_set`, `attr_clr`, `propagation`, `userns_fd`).
- **`statmount(2)` / `listmount(2)` introspection** — `struct statmount`, `struct mnt_id_req`, the `STATMOUNT_*` mask bits, `LSMT_ROOT`, `LISTMOUNT_REVERSE`, `STATMOUNT_BY_FD`.

Critical for: `mount(8)`, `umount(8)`, `findmnt(8)`, `systemd` (which uses the new mount API exclusively), every container-runtime mount setup (Docker, runc, podman, LXC, `systemd-nspawn`), every user-namespace mount-API consumer (`unshare(1)`, `fuse-overlayfs`), every initramfs `pivot_root` flow. The `MS_*` flag values are frozen ABI dating from 1992; the new `MOUNT_ATTR_*` / `FSCONFIG_*` are post-5.2 ABI that is also stabilized.

This Tier-5 covers `include/uapi/linux/mount.h` (~245 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `MS_RDONLY..MS_NOUSER` | per-legacy-mount(2) flag | `MountFlag` |
| `MS_RMT_MASK` | per-remount-allowed mask | `Mount::RMT_MASK` |
| `MS_MGC_VAL` / `MS_MGC_MSK` | per-historical-magic prefix | `Mount::{MGC_VAL,MGC_MSK}` |
| `OPEN_TREE_CLONE/NAMESPACE/CLOEXEC` | per-open_tree(2) flag | `OpenTreeFlag` |
| `MOVE_MOUNT_F_*` / `T_*` / `SET_GROUP` / `BENEATH` / `__MASK` | per-move_mount(2) flag | `MoveMountFlag` |
| `FSOPEN_CLOEXEC` | per-fsopen(2) flag | `FsopenFlag` |
| `FSPICK_*` | per-fspick(2) flag | `FspickFlag` |
| `enum fsconfig_command` | per-fsconfig(2) cmd | `FsconfigCommand` |
| `FSMOUNT_CLOEXEC/NAMESPACE` | per-fsmount(2) flag | `FsmountFlag` |
| `MOUNT_ATTR_*` | per-mount_setattr(2) attr | `MountAttr` |
| `MOUNT_ATTR__ATIME` | per-atime sub-mask | `MountAttr::ATIME_MASK` |
| `struct mount_attr` | per-mount_setattr(2) blob | `MountAttrBlob` |
| `MOUNT_ATTR_SIZE_VER0` | per-blob-version | `Mount::ATTR_SIZE_VER0` |
| `struct statmount` | per-statmount(2) result | `Statmount` |
| `struct mnt_id_req` | per-statmount/listmount req | `MntIdReq` |
| `STATMOUNT_*` | per-mask bit | `StatmountMask` |
| `LSMT_ROOT` / `LISTMOUNT_REVERSE` | per-listmount sentinels | `Listmount::*` |
| `MNT_ID_REQ_SIZE_VER{0,1}` | per-req-version | `MntIdReq::*` |
| `do_mount` / `path_mount` | per-mount(2) core | `Mount::do_mount` |
| `do_fsopen` / `do_fsconfig` / `do_fsmount` | per-new-API core | `Mount::{fsopen,fsconfig,fsmount}` |
| `do_move_mount` / `do_open_tree` | per-new-API core | `Mount::{move_mount,open_tree}` |
| `do_mount_setattr` | per-mount_setattr(2) | `Mount::mount_setattr` |

## ABI surface (constants + structs)

### Legacy `mount(2)` flags `MS_*` (`mount.h:13-47`)

```text
MS_RDONLY        = 1          /* read-only */
MS_NOSUID        = 2          /* ignore suid/sgid bits */
MS_NODEV         = 4          /* disallow device-special access */
MS_NOEXEC        = 8          /* disallow program execution */
MS_SYNCHRONOUS   = 16         /* writes are synced at once */
MS_REMOUNT       = 32         /* alter flags of mounted FS */
MS_MANDLOCK      = 64         /* allow mandatory locks */
MS_DIRSYNC       = 128        /* directory modifications are synchronous */
MS_NOSYMFOLLOW   = 256        /* do not follow symlinks */
MS_NOATIME       = 1024       /* do not update atime */
MS_NODIRATIME    = 2048       /* do not update directory atime */
MS_BIND          = 4096       /* bind-mount */
MS_MOVE          = 8192       /* move-mount */
MS_REC           = 16384      /* recursive (with BIND/MOVE/SLAVE/SHARED/etc) */
MS_VERBOSE       = 32768      /* deprecated alias for MS_SILENT */
MS_SILENT        = 32768      /* suppress printk on success */
MS_POSIXACL      = (1<<16)    /* VFS does not apply umask */
MS_UNBINDABLE    = (1<<17)    /* change to unbindable */
MS_PRIVATE       = (1<<18)    /* change to private */
MS_SLAVE         = (1<<19)    /* change to slave */
MS_SHARED        = (1<<20)    /* change to shared */
MS_RELATIME      = (1<<21)    /* update atime relative to mtime/ctime */
MS_KERNMOUNT     = (1<<22)    /* this is a kern_mount call */
MS_I_VERSION     = (1<<23)    /* update inode I_version */
MS_STRICTATIME   = (1<<24)    /* always update atime */
MS_LAZYTIME      = (1<<25)    /* update on-disk [acm]times lazily */

/* kernel-internal; userspace MUST NOT set: */
MS_SUBMOUNT      = (1<<26)
MS_NOREMOTELOCK  = (1<<27)
MS_NOSEC         = (1<<28)
MS_BORN          = (1<<29)
MS_ACTIVE        = (1<<30)
MS_NOUSER        = (1<<31)

MS_RMT_MASK      = (MS_RDONLY | MS_SYNCHRONOUS | MS_MANDLOCK |
                    MS_I_VERSION | MS_LAZYTIME)

MS_MGC_VAL       = 0xC0ED0000    /* legacy magic prefix */
MS_MGC_MSK       = 0xffff0000    /* mask to detect magic */
```

`MS_VERBOSE` and `MS_SILENT` share bit `32768`; `MS_VERBOSE` is deprecated. The header's comment ("War is peace. Verbosity is silence.") preserves the historical aliasing.

`MS_NOATIME | MS_RELATIME | MS_STRICTATIME` are mutually exclusive (kernel picks the most-specific). `MS_BIND` + `MS_REC` is the canonical recursive bind-mount. `MS_PRIVATE | MS_SLAVE | MS_SHARED | MS_UNBINDABLE` are mount-propagation-type-change ops; mutually exclusive.

### `open_tree(2)` flags (`mount.h:64-66`)

```text
OPEN_TREE_CLONE     = 1 << 0       /* clone the tree, attach the clone */
OPEN_TREE_NAMESPACE = 1 << 1       /* clone into a new mount namespace */
OPEN_TREE_CLOEXEC   = O_CLOEXEC    /* set FD_CLOEXEC on the resulting fd */
```

### `move_mount(2)` flags (`mount.h:71-79`)

```text
MOVE_MOUNT_F_SYMLINKS    = 0x00000001   /* from: follow symlinks */
MOVE_MOUNT_F_AUTOMOUNTS  = 0x00000002   /* from: follow automounts */
MOVE_MOUNT_F_EMPTY_PATH  = 0x00000004   /* from: empty path permitted */
MOVE_MOUNT_T_SYMLINKS    = 0x00000010   /* to:   follow symlinks */
MOVE_MOUNT_T_AUTOMOUNTS  = 0x00000020   /* to:   follow automounts */
MOVE_MOUNT_T_EMPTY_PATH  = 0x00000040   /* to:   empty path permitted */
MOVE_MOUNT_SET_GROUP     = 0x00000100   /* set sharing group instead */
MOVE_MOUNT_BENEATH       = 0x00000200   /* mount beneath top mount */
MOVE_MOUNT__MASK         = 0x00000377   /* validation mask */
```

### `fsopen(2)` flags (`mount.h:84`)

```text
FSOPEN_CLOEXEC = 0x00000001
```

### `fspick(2)` flags (`mount.h:89-92`)

```text
FSPICK_CLOEXEC          = 0x00000001
FSPICK_SYMLINK_NOFOLLOW = 0x00000002
FSPICK_NO_AUTOMOUNT     = 0x00000004
FSPICK_EMPTY_PATH       = 0x00000008
```

### `enum fsconfig_command` (`mount.h:97-107`)

```text
FSCONFIG_SET_FLAG         = 0   /* set param, no value */
FSCONFIG_SET_STRING       = 1   /* set param, string value */
FSCONFIG_SET_BINARY       = 2   /* set param, binary blob */
FSCONFIG_SET_PATH         = 3   /* set param, object-by-path */
FSCONFIG_SET_PATH_EMPTY   = 4   /* set param, object-by-(empty)-path */
FSCONFIG_SET_FD           = 5   /* set param, object-by-fd */
FSCONFIG_CMD_CREATE       = 6   /* create new or reuse existing sb */
FSCONFIG_CMD_RECONFIGURE  = 7   /* invoke sb reconfiguration */
FSCONFIG_CMD_CREATE_EXCL  = 8   /* create new sb, fail if reusing */
```

### `fsmount(2)` flags (`mount.h:112-113`)

```text
FSMOUNT_CLOEXEC   = 0x00000001
FSMOUNT_NAMESPACE = 0x00000002   /* mount in a new mount namespace */
```

### `mount_setattr(2)` attribute bits (`mount.h:118-128`)

```text
MOUNT_ATTR_RDONLY      = 0x00000001   /* mount read-only */
MOUNT_ATTR_NOSUID      = 0x00000002   /* ignore suid/sgid */
MOUNT_ATTR_NODEV       = 0x00000004   /* disallow device-special */
MOUNT_ATTR_NOEXEC      = 0x00000008   /* disallow program exec */
MOUNT_ATTR__ATIME      = 0x00000070   /* sub-field mask (3 bits) */
MOUNT_ATTR_RELATIME    = 0x00000000   /*  - relative atime */
MOUNT_ATTR_NOATIME     = 0x00000010   /*  - no atime updates */
MOUNT_ATTR_STRICTATIME = 0x00000020   /*  - always atime updates */
MOUNT_ATTR_NODIRATIME  = 0x00000080   /* no directory atime */
MOUNT_ATTR_IDMAP       = 0x00100000   /* idmap mount to userns_fd */
MOUNT_ATTR_NOSYMFOLLOW = 0x00200000   /* no symlink-follow on this mount */
```

`MOUNT_ATTR__ATIME` is a **3-bit sub-field**, not a flag — it covers exactly one of `RELATIME` (0), `NOATIME` (0x10), or `STRICTATIME` (0x20). Userspace MUST set `attr_set & MOUNT_ATTR__ATIME` to exactly one of these values (or zero for relatime).

### `struct mount_attr` (`mount.h:133-141`)

```text
struct mount_attr {
    __u64 attr_set;       /* MOUNT_ATTR_* bits to set */
    __u64 attr_clr;       /* MOUNT_ATTR_* bits to clear */
    __u64 propagation;    /* propagation type: MS_PRIVATE/SLAVE/SHARED/UNBINDABLE */
    __u64 userns_fd;      /* fd of user namespace (with MOUNT_ATTR_IDMAP) */
};

MOUNT_ATTR_SIZE_VER0 = 32   /* sizeof the first published struct */
```

### `struct statmount` (`mount.h:157-191`)

```text
struct statmount {
    __u32 size;             /* total size including strings */
    __u32 mnt_opts;         /* [str-offset] options string */
    __u64 mask;             /* what fields were written */
    __u32 sb_dev_major;
    __u32 sb_dev_minor;
    __u64 sb_magic;         /* *_SUPER_MAGIC */
    __u32 sb_flags;         /* SB_{RDONLY,SYNCHRONOUS,DIRSYNC,LAZYTIME} */
    __u32 fs_type;          /* [str-offset] fstype */
    __u64 mnt_id;           /* unique mount id */
    __u64 mnt_parent_id;
    __u32 mnt_id_old;       /* legacy /proc/<pid>/mountinfo id */
    __u32 mnt_parent_id_old;
    __u64 mnt_attr;         /* MOUNT_ATTR_* */
    __u64 mnt_propagation;  /* MS_{SHARED,SLAVE,PRIVATE,UNBINDABLE} */
    __u64 mnt_peer_group;
    __u64 mnt_master;
    __u64 propagate_from;
    __u32 mnt_root;         /* [str-offset] root-of-mount relative to root-of-fs */
    __u32 mnt_point;        /* [str-offset] mountpoint relative to caller's root */
    __u64 mnt_ns_id;
    __u32 fs_subtype;       /* [str-offset] subtype if any */
    __u32 sb_source;        /* [str-offset] source string */
    __u32 opt_num;
    __u32 opt_array;        /* [str-offset] NUL-array of options */
    __u32 opt_sec_num;
    __u32 opt_sec_array;    /* [str-offset] NUL-array of security options */
    __u64 supported_mask;
    __u32 mnt_uidmap_num;
    __u32 mnt_uidmap;       /* [str-offset] */
    __u32 mnt_gidmap_num;
    __u32 mnt_gidmap;       /* [str-offset] */
    __u64 __spare2[43];
    char  str[];            /* variable-size strings region */
};
```

### `struct mnt_id_req` (`mount.h:200-212`)

```text
struct mnt_id_req {
    __u32 size;
    union {
        __u32 mnt_ns_fd;
        __u32 mnt_fd;
    };
    __u64 mnt_id;
    __u64 param;          /* statmount: mask; listmount: last id (or 0) */
    __u64 mnt_ns_id;
};

MNT_ID_REQ_SIZE_VER0 = 24
MNT_ID_REQ_SIZE_VER1 = 32
```

### `STATMOUNT_*` mask bits (`mount.h:218-232`)

```text
STATMOUNT_SB_BASIC       = 0x00000001U
STATMOUNT_MNT_BASIC      = 0x00000002U
STATMOUNT_PROPAGATE_FROM = 0x00000004U
STATMOUNT_MNT_ROOT       = 0x00000008U
STATMOUNT_MNT_POINT      = 0x00000010U
STATMOUNT_FS_TYPE        = 0x00000020U
STATMOUNT_MNT_NS_ID      = 0x00000040U
STATMOUNT_MNT_OPTS       = 0x00000080U
STATMOUNT_FS_SUBTYPE     = 0x00000100U
STATMOUNT_SB_SOURCE      = 0x00000200U
STATMOUNT_OPT_ARRAY      = 0x00000400U
STATMOUNT_OPT_SEC_ARRAY  = 0x00000800U
STATMOUNT_SUPPORTED_MASK = 0x00001000U
STATMOUNT_MNT_UIDMAP     = 0x00002000U
STATMOUNT_MNT_GIDMAP     = 0x00004000U

LSMT_ROOT          = 0xffffffffffffffff   /* listmount: root mount */
LISTMOUNT_REVERSE  = 1 << 0               /* list later mounts first */
STATMOUNT_BY_FD    = 0x00000001U          /* statmount @flag: by-fd */
```

## Compatibility contract

REQ-1: All `MS_*` numeric values MUST match Linux exactly. `MS_RDONLY=1, MS_NOSUID=2, MS_NODEV=4, MS_NOEXEC=8, MS_SYNCHRONOUS=16, MS_REMOUNT=32, MS_MANDLOCK=64, MS_DIRSYNC=128, MS_NOSYMFOLLOW=256, MS_NOATIME=1024, MS_NODIRATIME=2048, MS_BIND=4096, MS_MOVE=8192, MS_REC=16384, MS_SILENT=32768, MS_POSIXACL=0x10000, MS_UNBINDABLE=0x20000, MS_PRIVATE=0x40000, MS_SLAVE=0x80000, MS_SHARED=0x100000, MS_RELATIME=0x200000, MS_KERNMOUNT=0x400000, MS_I_VERSION=0x800000, MS_STRICTATIME=0x1000000, MS_LAZYTIME=0x2000000`.

REQ-2: Kernel-internal flags `MS_SUBMOUNT (1<<26)`, `MS_NOREMOTELOCK (1<<27)`, `MS_NOSEC (1<<28)`, `MS_BORN (1<<29)`, `MS_ACTIVE (1<<30)`, `MS_NOUSER (1<<31)` MUST be silently cleared from `mount(2)` user-supplied `mountflags` at syscall entry. Userspace setting them is silently ignored (not an error to preserve ABI).

REQ-3: `MS_RMT_MASK == (MS_RDONLY | MS_SYNCHRONOUS | MS_MANDLOCK | MS_I_VERSION | MS_LAZYTIME) == 0x2800041`. On `mount(MS_REMOUNT, ...)`, only bits in this mask may change the superblock state; other bits are preserved.

REQ-4: `MS_MGC_VAL == 0xC0ED0000`, `MS_MGC_MSK == 0xffff0000`. If `(mountflags & MS_MGC_MSK) == MS_MGC_VAL` on a legacy `mount(2)`, the kernel strips the magic and processes the remaining low-16-bit flags. Modern callers MUST NOT supply the magic.

REQ-5: `MS_BIND | MS_REC` is the canonical recursive bind-mount. `MS_MOVE` MUST NOT be combined with `MS_BIND` or `MS_REMOUNT` (returns `-EINVAL`).

REQ-6: `MS_PRIVATE`, `MS_SLAVE`, `MS_SHARED`, `MS_UNBINDABLE` are mutually exclusive propagation-type-change ops. Combined with `MS_REC`, they apply recursively. They require no source/fstype arguments to `mount(2)`.

REQ-7: `MS_NOSYMFOLLOW` is enforced at path-walk time for **opens via this mount** — files reached through the mount are looked up with `LOOKUP_NO_SYMLINKS` semantics. Note: this applies to symlinks **starting from the mount**, not to symlinks within file contents.

REQ-8: `MS_NOSUID` MUST cause `setuid`/`setgid` bits on binaries reached via this mount to be ignored during `execve(2)`. Defense-in-depth: the kernel still preserves the bits in `i_mode`; the suppression is at exec-time.

REQ-9: `MS_NODEV` MUST cause `open(2)` of device-special files (S_IFCHR, S_IFBLK) reached via this mount to return `-EACCES`.

REQ-10: `MS_NOEXEC` MUST cause `execve(2)` of files reached via this mount to return `-EACCES`.

REQ-11: `MS_NOATIME` precludes any `[ac]time` updates. `MS_NODIRATIME` precludes only directory atime. `MS_RELATIME` (default since 2.6.30) updates `atime` only if `atime < mtime` or `atime < ctime` or atime > 24 hours old. `MS_STRICTATIME` always updates. These four are mutually exclusive.

REQ-12: `MS_KERNMOUNT (1<<22)` is **NEVER** set by userspace; it marks a `kern_mount()` call internally. `mount(2)` MUST strip it from user flags.

REQ-13: `OPEN_TREE_CLONE | OPEN_TREE_NAMESPACE` requires `CAP_SYS_ADMIN` in the caller's user-namespace. `OPEN_TREE_NAMESPACE` without `OPEN_TREE_CLONE` returns `-EINVAL`.

REQ-14: `move_mount(2)` MUST validate `flags & MOVE_MOUNT__MASK == flags` — unknown bits return `-EINVAL`. `MOVE_MOUNT_BENEATH` requires that the target mount support beneath-mounting (mountpoint mount, not a leaf).

REQ-15: `fsconfig(2)` cmd MUST be one of the enum values 0..8. Unknown cmd returns `-EOPNOTSUPP`. Per-cmd argument constraints: `SET_FLAG` takes no value (param `value`/`aux` ignored); `SET_STRING` takes a `char *`; `SET_BINARY` takes a `(void *, size)` pair via `value`/`aux`; `SET_PATH`/`SET_PATH_EMPTY` take a path string in `value` and a dirfd in `aux`; `SET_FD` takes an fd in `aux`. `CMD_CREATE`/`CMD_RECONFIGURE`/`CMD_CREATE_EXCL` take no parameters.

REQ-16: `fsmount(2)` flags MUST be a subset of `FSMOUNT_CLOEXEC | FSMOUNT_NAMESPACE`. Unknown bits return `-EINVAL`. `attr_flags` argument (low 32 bits) MUST be a subset of `MOUNT_ATTR_RDONLY | NOSUID | NODEV | NOEXEC | __ATIME | NODIRATIME | NOSYMFOLLOW`.

REQ-17: `mount_setattr(2)` operates on `struct mount_attr { attr_set, attr_clr, propagation, userns_fd }`. `MOUNT_ATTR_SIZE_VER0 == 32`. The kernel MUST accept `size == 32` and reject smaller sizes; larger sizes are forward-compat (kernel zero-extends).

REQ-18: `attr_set & MOUNT_ATTR__ATIME` MUST be exactly one of `MOUNT_ATTR_RELATIME` (0), `MOUNT_ATTR_NOATIME` (0x10), `MOUNT_ATTR_STRICTATIME` (0x20). Setting two simultaneously returns `-EINVAL`.

REQ-19: `attr_set & MOUNT_ATTR_IDMAP` requires `userns_fd` to refer to a valid user namespace; the kernel does NOT apply idmap if the userns is the caller's own (no-op idmap returns `-EINVAL`). Idmap is permanent on the mount — `attr_clr & MOUNT_ATTR_IDMAP` returns `-EINVAL` (cannot un-idmap).

REQ-20: `propagation` field MUST be exactly one of `MS_PRIVATE | MS_SLAVE | MS_SHARED | MS_UNBINDABLE` or zero (no change). Multiple bits set returns `-EINVAL`.

REQ-21: `statmount(2)` MUST set `mask` on output to indicate which fields were filled. Caller-supplied `mask` is the **requested** subset; kernel-returned `mask` is the actually-filled subset (∩ supported).

REQ-22: `struct statmount` strings live in the trailing `char str[]`; integer fields hold byte-offsets relative to `&statmount.str[0]`. The buffer must include enough space; insufficient buffer returns `-EOVERFLOW` and the kernel sets `size` to required-bytes.

REQ-23: `mnt_id_req.size` MUST be one of `MNT_ID_REQ_SIZE_VER0 (24)` or `MNT_ID_REQ_SIZE_VER1 (32)` or larger (forward-compat). Smaller returns `-EINVAL`.

REQ-24: `listmount(2)` with `mnt_id == LSMT_ROOT (~0)` lists the root mount of the caller's mount namespace. `param` is the last-listed mount id (0 to start). `LISTMOUNT_REVERSE` reverses iteration order.

REQ-25: All write operations (mount, umount, move_mount, mount_setattr, open_tree with CLONE) MUST be performed under `CAP_SYS_ADMIN` in **both** the calling user-namespace AND the namespace owning the source/target mount. User-namespace-confined mounts are permitted only for vfs types marked `FS_USERNS_MOUNT`.

## Acceptance Criteria

- [ ] AC-1: All `MS_*` values match the Linux table (REQ-1 / REQ-2).
- [ ] AC-2: `MS_RMT_MASK == 0x2800041` and `MS_REMOUNT` only changes bits within this mask.
- [ ] AC-3: `MS_MGC_VAL == 0xC0ED0000`, `MS_MGC_MSK == 0xffff0000`; legacy magic stripped at syscall entry.
- [ ] AC-4: Kernel-internal `MS_*` (bits 26..31) silently cleared from user `mountflags`.
- [ ] AC-5: `MS_NOSUID` mount: `execve(2)` of suid binary via this mount: suid bits ignored.
- [ ] AC-6: `MS_NODEV` mount: `open(2)` of S_IFCHR file via this mount returns `-EACCES`.
- [ ] AC-7: `MS_NOEXEC` mount: `execve(2)` of any file via this mount returns `-EACCES`.
- [ ] AC-8: `MS_NOSYMFOLLOW` mount: `open(2)` traversing a symlink located **at** this mount returns `-ELOOP`.
- [ ] AC-9: `MS_BIND | MS_REC` clones the whole subtree of mounts.
- [ ] AC-10: `MS_PRIVATE | MS_SHARED` in one call returns `-EINVAL`.
- [ ] AC-11: `OPEN_TREE_CLONE` without `CAP_SYS_ADMIN` returns `-EPERM`; with it, returns a clonefd.
- [ ] AC-12: `OPEN_TREE_NAMESPACE` without `OPEN_TREE_CLONE` returns `-EINVAL`.
- [ ] AC-13: `move_mount(2)` with `flags & ~MOVE_MOUNT__MASK != 0` returns `-EINVAL`.
- [ ] AC-14: `fsconfig(FSCONFIG_CMD_CREATE_EXCL)` on existing reusable sb returns `-EBUSY`.
- [ ] AC-15: `fsmount(2)` with `attr_flags` containing unsupported bits returns `-EINVAL`.
- [ ] AC-16: `mount_setattr(2)` with `attr_set & MOUNT_ATTR__ATIME == 0x30` (two bits) returns `-EINVAL`.
- [ ] AC-17: `mount_setattr(2)` clearing `MOUNT_ATTR_IDMAP` returns `-EINVAL` (idmap not removable).
- [ ] AC-18: `mount_setattr(2)` with `propagation == MS_SHARED | MS_SLAVE` returns `-EINVAL`.
- [ ] AC-19: `statmount(2)` with insufficient buffer returns `-EOVERFLOW` and sets `size` to required-bytes.
- [ ] AC-20: `mnt_id_req.size < 24` returns `-EINVAL`; `== 24` or `== 32` succeeds.
- [ ] AC-21: `listmount(LSMT_ROOT, ...)` lists from the namespace root; `LISTMOUNT_REVERSE` reverses order.
- [ ] AC-22: Any mount op without `CAP_SYS_ADMIN` in the relevant userns returns `-EPERM`.

## Architecture

```
bitflags! {
    pub struct MsFlag: u32 {
        const RDONLY        = 1;
        const NOSUID        = 2;
        const NODEV         = 4;
        const NOEXEC        = 8;
        const SYNCHRONOUS   = 16;
        const REMOUNT       = 32;
        const MANDLOCK      = 64;
        const DIRSYNC       = 128;
        const NOSYMFOLLOW   = 256;
        const NOATIME       = 1024;
        const NODIRATIME    = 2048;
        const BIND          = 4096;
        const MOVE          = 8192;
        const REC           = 16384;
        const SILENT        = 32768;
        const POSIXACL      = 1 << 16;
        const UNBINDABLE    = 1 << 17;
        const PRIVATE       = 1 << 18;
        const SLAVE         = 1 << 19;
        const SHARED        = 1 << 20;
        const RELATIME      = 1 << 21;
        const KERNMOUNT     = 1 << 22;
        const I_VERSION     = 1 << 23;
        const STRICTATIME   = 1 << 24;
        const LAZYTIME      = 1 << 25;
        /* kernel-internal */
        const SUBMOUNT      = 1 << 26;
        const NOREMOTELOCK  = 1 << 27;
        const NOSEC         = 1 << 28;
        const BORN          = 1 << 29;
        const ACTIVE        = 1 << 30;
        const NOUSER        = 1 << 31;

        const RMT_MASK = Self::RDONLY.bits | Self::SYNCHRONOUS.bits |
                         Self::MANDLOCK.bits | Self::I_VERSION.bits |
                         Self::LAZYTIME.bits;
        const KERNEL_INTERNAL = Self::SUBMOUNT.bits | Self::NOREMOTELOCK.bits |
                                Self::NOSEC.bits | Self::BORN.bits |
                                Self::ACTIVE.bits | Self::NOUSER.bits |
                                Self::KERNMOUNT.bits;
    }
}

pub const MS_MGC_VAL: u32 = 0xC0ED_0000;
pub const MS_MGC_MSK: u32 = 0xFFFF_0000;

bitflags! {
    pub struct MountAttr: u64 {
        const RDONLY      = 0x0000_0001;
        const NOSUID      = 0x0000_0002;
        const NODEV       = 0x0000_0004;
        const NOEXEC      = 0x0000_0008;
        const RELATIME    = 0x0000_0000;   /* sub-field; default */
        const NOATIME     = 0x0000_0010;
        const STRICTATIME = 0x0000_0020;
        const ATIME_MASK  = 0x0000_0070;
        const NODIRATIME  = 0x0000_0080;
        const IDMAP       = 0x0010_0000;
        const NOSYMFOLLOW = 0x0020_0000;
    }
}

#[repr(C)]
pub struct MountAttrBlob {
    pub attr_set:    u64,
    pub attr_clr:    u64,
    pub propagation: u64,
    pub userns_fd:   u64,
}
pub const MOUNT_ATTR_SIZE_VER0: u32 = 32;

#[repr(u32)]
pub enum FsconfigCommand {
    SetFlag        = 0,
    SetString      = 1,
    SetBinary      = 2,
    SetPath        = 3,
    SetPathEmpty   = 4,
    SetFd          = 5,
    CmdCreate      = 6,
    CmdReconfigure = 7,
    CmdCreateExcl  = 8,
}

bitflags! {
    pub struct OpenTreeFlag: u32 {
        const CLONE     = 1 << 0;
        const NAMESPACE = 1 << 1;
        const CLOEXEC   = O_CLOEXEC;
    }
}

bitflags! {
    pub struct MoveMountFlag: u32 {
        const F_SYMLINKS   = 0x01;
        const F_AUTOMOUNTS = 0x02;
        const F_EMPTY_PATH = 0x04;
        const T_SYMLINKS   = 0x10;
        const T_AUTOMOUNTS = 0x20;
        const T_EMPTY_PATH = 0x40;
        const SET_GROUP    = 0x100;
        const BENEATH      = 0x200;
        const MASK         = 0x377;   /* MOVE_MOUNT__MASK */
    }
}

bitflags! {
    pub struct FsopenFlag:  u32 { const CLOEXEC = 0x01; }
    pub struct FsmountFlag: u32 { const CLOEXEC = 0x01; const NAMESPACE = 0x02; }
    pub struct FspickFlag:  u32 {
        const CLOEXEC          = 0x01;
        const SYMLINK_NOFOLLOW = 0x02;
        const NO_AUTOMOUNT     = 0x04;
        const EMPTY_PATH       = 0x08;
    }
}
```

`Mount::do_mount(dev: &Path, dir: &Path, fstype: &str, mut flags: u32, data: UserPtr<u8>) -> Result<()>`:
1. /* Strip legacy magic */
2. if flags & MS_MGC_MSK == MS_MGC_VAL: flags &= !MS_MGC_MSK.
3. /* Strip kernel-internal bits */
4. flags &= !MsFlag::KERNEL_INTERNAL.bits.
5. /* CAP gate */
6. capable(CAP_SYS_ADMIN)?.   /* in userns owning dir.mnt_ns */
7. /* Propagation-type-change ops have no source/fstype */
8. let prop = flags & (MsFlag::PRIVATE | SLAVE | SHARED | UNBINDABLE).bits.
9. if prop != 0:
   - return Mount::change_propagation(dir, prop, flags & MsFlag::REC.bits).
10. /* Bind */
11. if flags & MsFlag::BIND.bits != 0:
    - return Mount::bind(dev, dir, flags & MsFlag::REC.bits).
12. /* Move */
13. if flags & MsFlag::MOVE.bits != 0:
    - return Mount::move_(dev, dir).
14. /* Remount */
15. if flags & MsFlag::REMOUNT.bits != 0:
    - return Mount::remount(dir, flags & MsFlag::RMT_MASK.bits, data).
16. /* New superblock mount */
17. return Mount::new_sb(dev, dir, fstype, flags, data).

`Mount::mount_setattr(target: &Path, attr: &MountAttrBlob, flags: u32) -> Result<()>`:
1. /* Validate size — caller supplies via syscall; we assume blob is canonical here */
2. /* Validate atime sub-field is one-of-three */
3. let atime = attr.attr_set & MountAttr::ATIME_MASK.bits;
4. if atime != 0 && atime != MountAttr::NOATIME.bits && atime != MountAttr::STRICTATIME.bits:
   - return Err(EINVAL).
5. /* Validate propagation */
6. let prop = attr.propagation;
7. if prop.count_ones() > 1: return Err(EINVAL).
8. if prop != 0 && prop & !(MS_PRIVATE|SLAVE|SHARED|UNBINDABLE) != 0: return Err(EINVAL).
9. /* IDMAP not removable */
10. if attr.attr_clr & MountAttr::IDMAP.bits != 0: return Err(EINVAL).
11. /* CAP */
12. capable(CAP_SYS_ADMIN)?.
13. /* Apply */
14. Mount::apply_attr(target, attr.attr_set, attr.attr_clr, flags & AT_RECURSIVE)?.
15. if prop != 0: Mount::change_propagation(target, prop as u32, flags & AT_RECURSIVE != 0)?.
16. if attr.attr_set & MountAttr::IDMAP.bits != 0:
    - Mount::apply_idmap(target, attr.userns_fd as i32)?.
17. return Ok(()).

`Mount::fsmount(fs_fd: i32, fsmount_flags: u32, attr_flags: u32) -> Result<i32>`:
1. /* Validate flags */
2. if fsmount_flags & !(FsmountFlag::CLOEXEC | NAMESPACE).bits != 0:
   - return Err(EINVAL).
3. let allowed_attr = MountAttr::RDONLY | NOSUID | NODEV | NOEXEC |
                     ATIME_MASK | NODIRATIME | NOSYMFOLLOW;
4. if attr_flags & !allowed_attr.bits != 0:
   - return Err(EINVAL).
5. let atime = attr_flags & MountAttr::ATIME_MASK.bits;
6. if atime != 0 && atime != MountAttr::NOATIME.bits && atime != MountAttr::STRICTATIME.bits:
   - return Err(EINVAL).
7. let fc = FsContext::from_fd(fs_fd)?.
8. capable(CAP_SYS_ADMIN)?.
9. let new_mount = Mount::create_from_fc(fc, attr_flags)?.
10. let new_fd = install_fd(new_mount, fsmount_flags & FsmountFlag::CLOEXEC.bits != 0)?.
11. return Ok(new_fd).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ms_flag_table` | INVARIANT | Every `MS_*` matches the Linux numeric table. |
| `ms_rmt_mask_value` | INVARIANT | `MS_RMT_MASK == 0x2800041`. |
| `kernel_internal_stripped` | INVARIANT | per-do_mount: bits 26..31 cleared from user flags. |
| `mgc_magic_stripped` | INVARIANT | per-do_mount: `(flags & MGC_MSK) == MGC_VAL ⟹ MGC bits stripped`. |
| `mount_attr_size_ver0` | INVARIANT | `MountAttrBlob` is 32 bytes; `MOUNT_ATTR_SIZE_VER0 == 32`. |
| `atime_one_of_three` | INVARIANT | per-mount_setattr: `attr_set & ATIME_MASK` ∈ {0, NOATIME, STRICTATIME}. |
| `idmap_not_removable` | INVARIANT | per-mount_setattr: `attr_clr & IDMAP` ⟹ EINVAL. |
| `propagation_at_most_one` | INVARIANT | per-mount_setattr: `popcount(propagation) ≤ 1`. |
| `open_tree_namespace_needs_clone` | INVARIANT | per-open_tree: NAMESPACE without CLONE ⟹ EINVAL. |
| `move_mount_mask_validated` | INVARIANT | per-move_mount: `flags & ~__MASK ⟹ EINVAL`. |
| `cap_sys_admin_on_every_writer` | INVARIANT | per-mount/umount/move_mount/mount_setattr/open_tree-clone: caller has CAP_SYS_ADMIN. |
| `fsmount_attr_subset` | INVARIANT | per-fsmount: attr_flags ⊆ {RDONLY,NOSUID,NODEV,NOEXEC,_ATIME,NODIRATIME,NOSYMFOLLOW}. |
| `fsconfig_cmd_range` | INVARIANT | per-fsconfig: cmd ∈ {0..8}. |
| `statmount_overflow_returns_eoverflow` | INVARIANT | per-statmount: insufficient buffer ⟹ EOVERFLOW with size = required. |
| `mnt_id_req_size_versioned` | INVARIANT | per-statmount/listmount: req.size ∈ {24, 32, ≥ 32}. |

### Layer 2: TLA+

`uapi/headers/mount.tla`:
- Per-`fsopen(2)` → `fsconfig(2)*` → `fsmount(2)` → `move_mount(2)` → `umount(2)` lifecycle.
- Per-`mount(2)` legacy entry.
- Per-`open_tree(2)` → `mount_setattr(2)` → `move_mount(2)` flow.
- Properties:
  - `safety_kernel_internal_unforgeable` — user-supplied `MS_KERNMOUNT`/`SUBMOUNT`/`NOREMOTELOCK`/`NOSEC`/`BORN`/`ACTIVE`/`NOUSER` never reach the VFS.
  - `safety_remount_only_in_rmt_mask` — `MS_REMOUNT` cannot change bits outside `MS_RMT_MASK`.
  - `safety_atime_mutex` — at most one of NOATIME/RELATIME/STRICTATIME on a mount at any time.
  - `safety_propagation_mutex` — at most one of PRIVATE/SLAVE/SHARED/UNBINDABLE per change op.
  - `safety_idmap_immutable_once_set` — IDMAP cannot be cleared via mount_setattr.
  - `safety_cap_sys_admin_on_writers` — every mount-tree-mutating syscall requires CAP_SYS_ADMIN.
  - `liveness_fsmount_chain_eventually_mounts` — fsopen → fsconfig(CREATE) → fsmount eventually attaches a mount.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MsFlag` bit table post: every const matches Linux | `MsFlag` |
| `MountAttrBlob` layout post: 32 bytes, 4×u64 | `MountAttrBlob` |
| `Mount::do_mount` post: kernel-internal flags stripped; CAP_SYS_ADMIN verified | `Mount::do_mount` |
| `Mount::mount_setattr` post: atime/prop validated; idmap-clear rejected | `Mount::mount_setattr` |
| `Mount::fsmount` post: attr_flags ⊆ allowed; CAP_SYS_ADMIN verified | `Mount::fsmount` |
| `Mount::open_tree` post: NAMESPACE ⟹ CLONE; CAP_SYS_ADMIN verified | `Mount::open_tree` |
| `Mount::move_mount` post: flags subset of `__MASK` | `Mount::move_mount` |

### Layer 4: Verus/Creusot functional

`Per mount(2) / fsopen(2) → fsconfig(2)* → fsmount(2) / move_mount(2) / mount_setattr(2) / open_tree(2) / umount(2)` semantic equivalence: per-`Documentation/filesystems/mount_api.rst`, per-`Documentation/filesystems/shared-subtrees.txt`, per-`Documentation/filesystems/idmappings.rst`, per-`man 2 mount`, per-`man 2 fsopen`, per-`man 2 mount_setattr`, per-`man 2 statmount`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

mount UAPI reinforcement:

- **Kernel-internal `MS_*` bits (KERNMOUNT, SUBMOUNT, NOREMOTELOCK, NOSEC, BORN, ACTIVE, NOUSER) stripped at syscall entry** — defense against per-user-space spoofing of internal flags.
- **`MS_MGC_VAL` legacy magic stripped** — defense against per-bit-pattern collision with new flags.
- **`MS_REMOUNT` only affects `MS_RMT_MASK` bits** — defense against per-userland surprise toggle of, e.g., MS_BIND on a live mount.
- **Propagation-type ops mutually exclusive** — defense against per-PROP-confusion (e.g., simultaneously SHARED and PRIVATE).
- **`MS_NOSUID`/`NODEV`/`NOEXEC`/`NOSYMFOLLOW` enforced at exec/open/path-walk** — defense in depth against per-mount untrusted-content escalation.
- **`MOUNT_ATTR_IDMAP` immutable once set** — defense against per-idmap-clear privilege-confusion.
- **`MOUNT_ATTR__ATIME` validated as one-of-three** — defense against per-multi-bit confusion (kernel previously had bugs accepting NOATIME | STRICTATIME).
- **`fsconfig` cmd switch is exhaustive** — defense against per-unknown-cmd undefined behavior.
- **`open_tree(NAMESPACE)` requires `CLONE`** — defense against per-namespace-leak via open_tree without explicit clone.
- **`statmount` overflow returns size-required** — defense against per-truncated-buffer info-leak.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY** on every `copy_from_user` of `struct mount_attr`, `struct mnt_id_req`, the `fsconfig` `value` blob, the `fsmount` `attr_flags` word, and the mount/umount path-string buffers — defense against per-kernel-pointer dereference of user-controlled bytes during multi-step mount setup. The variable-length `struct statmount` `str[]` region is a particular `copy_to_user` hot-spot.
- **PAX_RANDKSTACK on syscall entry** — randomize kernel-stack offset on every `mount(2)`, `umount(2)`, `umount2(2)`, `fsopen(2)`, `fsconfig(2)`, `fsmount(2)`, `fspick(2)`, `move_mount(2)`, `open_tree(2)`, `mount_setattr(2)`, `statmount(2)`, `listmount(2)` entry. `struct fs_context` and `struct mount_attr` are stack-resident for the duration of these calls; randomization defeats heap/stack-grooming for adjacent-allocation disclosure.
- **GRKERNSEC_CHROOT_MOUNT** — chroot'd processes are denied any mount operation regardless of capability: `mount(2)`, `umount(2)`, `move_mount(2)`, `open_tree(CLONE)`, `mount_setattr(2)`, `pivot_root(2)` all return `-EPERM`. Defense against per-chroot-escape via mount-tree mutation (classic chroot-escape via bind-mount-over `/proc/self/root`).
- **GRKERNSEC_NOMOUNT for non-root** — even outside chroots, unprivileged user-namespaces are denied mount syscalls beyond the documented `FS_USERNS_MOUNT` set (tmpfs, devpts, sysfs read-only, proc read-only, fuse, overlayfs in user-ns). Defense against per-FS-bug-driven privilege escalation through niche filesystems.
- **MOUNT_ATTR_NOSYMFOLLOW + MOUNT_ATTR_NOEXEC enforcement** — Rookery's container-default mount profile sets `NOSYMFOLLOW | NOEXEC | NOSUID | NODEV` on every bind-mounted user-data directory. The kernel verifies enforcement at exec/open time regardless of whether the symlink/binary is "trusted" from userspace's perspective. Defense against per-untrusted-bind-mount escalation (the "shadow `/etc/passwd`" class).
- **CAP_SYS_ADMIN gating in BOTH source AND target user-namespace** — `move_mount(2)` between user-namespaces requires the capability in both; defense against per-userns-asymmetry escalation.
- **`MS_BIND` recursive depth bounded** — `MS_BIND | MS_REC` over a deeply nested mount-tree is bounded by a kernel limit (`MAX_MOUNT_TREE_DEPTH`); defense against per-mount-tree DoS / kernel-stack exhaustion in propagation code.
- **`fsconfig(FSCONFIG_SET_PATH)` path-resolution under caller's user-namespace** — paths supplied to fsconfig are resolved relative to the caller's namespace, not the kernel root; defense against per-userns-confused-deputy where a less-privileged process tricks the kernel into resolving a path in a more-privileged namespace.
- **`MOUNT_ATTR_IDMAP` requires distinct user-namespace** — idmap to caller's own userns is rejected (`-EINVAL`); defense against per-trivial-noop-idmap masking a real escalation.
- **`MOUNT_ATTR_IDMAP` immutable once applied** — once an idmap is on a mount, `mount_setattr(2)` cannot clear it; the only path to remove is umount + re-mount. Defense against per-toggling-idmap to bypass per-path access checks.
- **`open_tree(2)` CLOEXEC default-on under grsec** — for `open_tree(OPEN_TREE_CLONE)`, the resulting fd is **always** opened with `FD_CLOEXEC` under grsec policy regardless of whether userspace requested it. Defense against per-clone-tree-fd-leak-via-execve.
- **`statmount(2)` / `listmount(2)` audit-logged on suid binaries** — record (uid, pid, mnt_id, mask) on every call from suid context; defense against per-stealth-enumeration of host mount-tree topology by privileged-but-suspect binaries.
- **`MS_MANDLOCK` denied unprivileged** — mandatory locking is a known DoS vector (a process can wedge an entire filesystem); requires `CAP_SYS_ADMIN` in initial userns under grsec. Defense against per-userns-mandlock-DoS.

## Open Questions

(none at this Tier-5 level — `fs/namespace.c`, `fs/fs_context.c`, `fs/mount.c` Tier-3 docs cover implementation)

## Out of Scope

- `include/uapi/linux/fs.h` superblock-flag `SB_*` mirror (covered in `uapi/headers/fs.md`).
- `include/uapi/linux/fanotify.h` mount-watch ABI (covered in `uapi/headers/fanotify.md`).
- `include/uapi/linux/magic.h` filesystem magic numbers used in `statmount.sb_magic` (covered separately).
- `fs/namespace.c` mount-namespace plumbing (covered in `fs/namespace.md` Tier-3).
- `fs/fs_context.c` new mount API plumbing (covered in `fs/fs_context.md` Tier-3).
- `fs/mount.c` core mount-tree mutation (covered in `fs/mount.md` Tier-3).
- Per-filesystem `FS_USERNS_MOUNT` policies (covered per-fs).
- Implementation code.
