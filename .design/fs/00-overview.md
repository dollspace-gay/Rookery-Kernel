# Subsystem: fs/ — virtual filesystem and filesystems

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - fs/
  - fs/proc/
  - fs/sysfs/
  - fs/devpts/
  - fs/ramfs/
  - fs/hugetlbfs/
  - fs/debugfs/
  - fs/configfs/
  - fs/tracefs/
  - fs/efivarfs/
  - fs/pstore/
  - fs/iomap/
  - fs/netfs/
  - fs/jbd2/
  - fs/crypto/
  - fs/verity/
  - fs/notify/
  - fs/quota/
  - fs/exportfs/
  - fs/kernfs/
  - fs/unicode/
  - fs/nls/
  - fs/ext4/
  - fs/ext2/
  - fs/btrfs/
  - fs/xfs/
  - fs/f2fs/
  - fs/fat/
  - fs/exfat/
  - fs/ntfs3/
  - fs/squashfs/
  - fs/erofs/
  - fs/iso9660/
  - fs/isofs/
  - fs/overlayfs/
  - fs/fuse/
  - fs/autofs/
  - fs/ecryptfs/
  - fs/nfs/
  - fs/nfsd/
  - fs/nfs_common/
  - fs/lockd/
  - fs/ceph/
  - fs/smb/
  - fs/9p/
  - fs/afs/
  - fs/cachefiles/
  - fs/orangefs/
  - fs/coda/
  - fs/dax.c
  - include/linux/fs.h
  - include/linux/dcache.h
  - include/linux/path.h
  - include/linux/file.h
  - include/linux/mount.h
  - include/linux/namei.h
  - include/linux/buffer_head.h
  - include/linux/iomap.h
  - include/linux/posix_acl.h
  - include/linux/fsnotify.h
  - include/linux/fscrypt.h
  - include/linux/fsverity.h
  - include/uapi/linux/fs.h
  - include/uapi/linux/fcntl.h
  - include/uapi/linux/stat.h
  - include/uapi/linux/mount.h
-->

## Summary
Tier-2 overview for `fs/` — the virtual filesystem (VFS) layer plus the in-tree filesystems. Owns the inode/dentry/file/superblock/vfsmount data model, path resolution (namei), file-table management, the entire posix-filesystem syscall surface (`open`, `read`, `write`, `stat`, `chmod`, `mount`, `unlink`, `rename`, ...), the file-locking subsystem (POSIX, BSD, OFD, leases, lockd), exec + binfmt loaders, the iomap I/O abstraction, the netfs network-filesystem helper layer, jbd2 (journaling block device for ext4), fs-crypto + fs-verity, fsnotify (inotify, fanotify, dnotify), the binfmt_elf userspace loader, and 78 individual filesystem implementations.

`fs/` is the second-largest subsystem after `kernel/` by surface area. The overall strategy for the per-filesystem coverage is **class-tiered with a named v0 port set**: not every filesystem gets a per-FS Tier-3 design doc, only those in the v0 port set; the rest are covered by class overviews and rely on the C implementations during transition (per the rust-for-linux interop pattern).

## Upstream references in scope

`fs/` (73 top-level files + 78 subdirectories at baseline). Mapping to Tier-3 docs:

| Category | Upstream paths (selection) | Planned Tier-3 doc |
|---|---|---|
| VFS core: inode, dentry, file, superblock, mount | `fs/inode.c`, `fs/dcache.c`, `fs/file.c`, `fs/file_table.c`, `fs/super.c`, `fs/namespace.c`, `fs/mount.h`, `fs/pnode.c`, `fs/d_path.c`, `fs/libfs.c`, `fs/bad_inode.c`, `fs/anon_inodes.c`, `fs/mnt_idmapping.c`, `fs/file_attr.c`, `include/linux/{fs,dcache,path,file,mount,namei}.h` | `vfs/00-overview.md` (Tier 3 hub) spawning `vfs/inode.md`, `vfs/dcache.md`, `vfs/super.md`, `vfs/file-table.md`, `vfs/mount.md` |
| Path resolution + nameidata | `fs/namei.c`, `include/linux/namei.h` | `vfs/path-resolution.md` (under vfs/) |
| Filesystem syscalls (open/read/write/stat/...) | `fs/open.c`, `fs/read_write.c`, `fs/stat.c`, `fs/statfs.c`, `fs/fcntl.c`, `fs/select.c`, `fs/sync.c`, `fs/utimes.c`, `fs/xattr.c`, `fs/ioctl.c`, `fs/readdir.c`, `fs/sync.c`, `fs/seq_file.c`, `fs/d_path.c`, `fs/fhandle.c`, `fs/locks.c`, `fs/posix_acl.c`, `fs/remap_range.c`, `fs/attr.c`, `fs/init.c`, `include/uapi/linux/{fs,fcntl,stat,ioctl,xattr}.h` | `syscalls.md` |
| Filesystem context + parameters (modern mount API) | `fs/fs_context.c`, `fs/fs_parser.c`, `fs/fs_pin.c`, `fs/fs_struct.c`, `fs/fsopen.c`, `fs/filesystems.c` | `vfs/fs-context.md` (under vfs/) |
| exec + binfmt | `fs/exec.c`, `fs/binfmt_elf.c`, `fs/binfmt_misc.c`, `fs/binfmt_script.c`, `fs/binfmt_flat.c`, `fs/binfmt_elf_fdpic.c`, `fs/compat_binfmt_elf.c`, `fs/coredump.c`, `fs/kernel_read_file.c` | `exec-binfmt.md` |
| iomap (modern I/O abstraction) | `fs/iomap/` | `iomap.md` |
| netfs (network-FS helper layer) | `fs/netfs/` | `netfs.md` |
| jbd2 (journaling block device, used by ext4) | `fs/jbd2/` | `jbd2.md` |
| fs-crypto (ext4 / f2fs encryption) | `fs/crypto/` | `fs-crypto.md` |
| fs-verity (per-file authenticated read) | `fs/verity/` | `fs-verity.md` |
| fsnotify (inotify, fanotify, dnotify, audit) | `fs/notify/` (inotify, fanotify, dnotify subdirs) | `fsnotify.md` |
| Quota | `fs/quota/` | `quota.md` |
| Buffer cache (legacy block-FS helper) | `fs/buffer.c`, `fs/mbcache.c`, `fs/mpage.c`, `include/linux/buffer_head.h` | `buffer-cache.md` |
| Direct I/O | `fs/direct-io.c`, `fs/dax.c` | `direct-io-dax.md` |
| Pipes | `fs/pipe.c`, `fs/splice.c`, `include/linux/pipe_fs_i.h` | `pipe-splice.md` |
| File-event FDs | `fs/eventpoll.c`, `fs/eventfd.c`, `fs/signalfd.c`, `fs/timerfd.c`, `fs/userfaultfd.c` (mm-owned but ABI here) | `event-fds.md` (cross-references mm/userfaultfd.md, kernel/task-lifecycle.md for signalfd) |
| AIO | `fs/aio.c` | `aio.md` |
| Background flushing / writeback | `fs/fs-writeback.c` | folded into `iomap.md` and per-FS docs |
| Pseudo-filesystems (compat-critical) | `fs/proc/`, `fs/sysfs/`, `fs/devpts/`, `fs/ramfs/`, `fs/hugetlbfs/`, `fs/debugfs/`, `fs/configfs/`, `fs/tracefs/`, `fs/efivarfs/`, `fs/pstore/`, `fs/nsfs.c`, `fs/anon_inodes.c`, `fs/kernfs/` | `pseudo-fs/00-overview.md` (Tier 3 hub) spawning `pseudo-fs/{proc,sysfs,devpts,ramfs-tmpfs,hugetlbfs,debugfs,configfs,tracefs,efivarfs,pstore,kernfs}.md` |
| Filesystem charset / encoding helpers | `fs/unicode/`, `fs/nls/` | `charset.md` |
| Disk filesystems (CLASS overview) | `fs/ext4/`, `fs/ext2/`, `fs/btrfs/`, `fs/xfs/`, `fs/f2fs/`, `fs/fat/`, `fs/exfat/`, `fs/ntfs3/`, `fs/squashfs/`, `fs/erofs/`, `fs/isofs/`, `fs/udf/`, `fs/jfs/`, `fs/nilfs2/`, `fs/ocfs2/`, `fs/gfs2/`, `fs/hpfs/`, `fs/affs/`, `fs/befs/`, `fs/bfs/`, `fs/cramfs/`, `fs/efs/`, `fs/freevxfs/`, `fs/hfs/`, `fs/hfsplus/`, `fs/jffs2/`, `fs/minix/`, `fs/omfs/`, `fs/qnx4/`, `fs/qnx6/`, `fs/romfs/`, `fs/ubifs/`, `fs/ufs/`, `fs/zonefs/`, `fs/openpromfs/` | `disk-fs/00-overview.md` + per-FS docs for the v0 port set (see Q1 below) |
| Network filesystems (CLASS overview) | `fs/nfs/`, `fs/nfsd/`, `fs/nfs_common/`, `fs/lockd/`, `fs/smb/` (cifs), `fs/ceph/`, `fs/9p/`, `fs/afs/`, `fs/cachefiles/`, `fs/orangefs/`, `fs/coda/` | `network-fs/00-overview.md` + per-FS docs for the v0 port set |
| Stacked / virtual filesystems | `fs/overlayfs/`, `fs/ecryptfs/`, `fs/fuse/`, `fs/autofs/` | `stacked-fs/00-overview.md` + per-FS docs |
| Misc | `fs/bpf_fs_kfuncs.c` (BPF kfuncs scoped to fs/), `fs/backing-file.c`, `fs/char_dev.c`, `fs/d_path.c`, `fs/drop_caches.c`, `fs/fserror.c`, `fs/fs_dirent.c`, `fs/internal.h` | folded into vfs/ Tier 3 docs and `syscalls.md` |
| Tests | `fs/tests/` | (not separately documented) |

## Compatibility contract

`fs/` owns one of the two largest slices of the userspace-visible compat surface (alongside `mm/` and `kernel/`). Because filesystem semantics are documented in POSIX and Linux extensions are documented in `Documentation/filesystems/`, the compat target is unusually well-defined.

### Syscall surface

Filesystem syscalls — every entry below maps to a Tier-5 `uapi/syscalls/<name>.md`:

`open`, `openat`, `openat2`, `creat`, `close`, `close_range`, `read`, `write`, `pread64`, `pwrite64`, `readv`, `writev`, `preadv`, `pwritev`, `preadv2`, `pwritev2`, `lseek`, `llseek`, `tee`, `splice`, `vmsplice`, `copy_file_range`, `sendfile`, `sendfile64`, `truncate`, `ftruncate`, `fallocate`, `fadvise64`, `readahead`, `posix_fadvise`, `madvise` (mm-owned), `dup`, `dup2`, `dup3`, `fcntl`, `flock`, `mkdir`, `mkdirat`, `rmdir`, `mknod`, `mknodat`, `link`, `linkat`, `unlink`, `unlinkat`, `symlink`, `symlinkat`, `readlink`, `readlinkat`, `rename`, `renameat`, `renameat2`, `chmod`, `fchmod`, `fchmodat`, `fchmodat2`, `chown`, `fchown`, `lchown`, `fchownat`, `chroot`, `chdir`, `fchdir`, `getcwd`, `umask`, `mount`, `umount`, `umount2`, `fsmount`, `fsopen`, `fsconfig`, `fspick`, `move_mount`, `open_tree`, `mount_setattr`, `pivot_root`, `swapon`/`swapoff` (mm-owned), `mkdir`, `getdents`, `getdents64`, `lookup_dcookie`, `quotactl`, `quotactl_fd`, `name_to_handle_at`, `open_by_handle_at`, `setxattr`, `lsetxattr`, `fsetxattr`, `getxattr`, `lgetxattr`, `fgetxattr`, `listxattr`, `llistxattr`, `flistxattr`, `removexattr`, `lremovexattr`, `fremovexattr`, `acct`, `sync`, `syncfs`, `fsync`, `fdatasync`, `sync_file_range`, `mlock` (mm), `inotify_init`, `inotify_init1`, `inotify_add_watch`, `inotify_rm_watch`, `fanotify_init`, `fanotify_mark`, `signalfd`, `signalfd4`, `timerfd_create`, `timerfd_gettime`, `timerfd_settime`, `eventfd`, `eventfd2`, `epoll_create`, `epoll_create1`, `epoll_ctl`, `epoll_wait`, `epoll_pwait`, `epoll_pwait2`, `userfaultfd` (mm), `pselect6`, `select`, `poll`, `ppoll`, `pidfd_open`, `pidfd_send_signal`, `pidfd_getfd`, `landlock_create_ruleset`, `landlock_add_rule`, `landlock_restrict_self`, `mq_open`, `mq_unlink`, `mq_timedsend`, `mq_timedreceive`, `mq_notify`, `mq_getsetattr`, `io_uring_setup`/`io_uring_register`/`io_uring_enter` (io_uring-owned but consume fs ABI), `aio_*`, `getuid`/etc. (kernel/-owned), `stat`, `lstat`, `fstat`, `fstatat`, `statx`, `statfs`, `fstatfs`, `ustat`, `init_module`/`finit_module`/`delete_module` (kernel/-owned but kernel module .ko is loaded via fs path), `process_madvise` (mm), `cachestat` (mm), `process_mrelease` (mm), `pread64`/`pwrite64` (above), `sendfile`/`sendfile64` (above)

(That's ~140 syscalls. Each gets a Tier-5 doc in Phase D.)

### `/proc` surfaces (fs-owned subset)

| Path | Owner doc | Compat level |
|---|---|---|
| `/proc/` mountpoint structure (a procfs implementation) | `pseudo-fs/proc.md` | Format-identical |
| `/proc/<pid>/{fd,fdinfo}/*` | `pseudo-fs/proc.md` (cross-ref `vfs/file-table.md`) | Format-identical |
| `/proc/<pid>/mountinfo`, `/proc/<pid>/mounts`, `/proc/<pid>/mountstats` | `vfs/mount.md` | Format-identical |
| `/proc/{filesystems,mounts,partitions,locks,self}` | `pseudo-fs/proc.md` | Format-identical |
| `/proc/sys/fs/{file-max,file-nr,nr_open,inode-nr,inode-state,dentry-state,...}` | `vfs/file-table.md`, `vfs/inode.md`, `vfs/dcache.md` | Identical |

### `/sys` surfaces (fs-owned subset)

| Path | Owner doc | Compat level |
|---|---|---|
| `/sys/` mountpoint structure (a sysfs implementation) | `pseudo-fs/sysfs.md` | Format-identical |
| `/sys/fs/<fs>/<various>` (per-FS sysfs entries) | per-FS Tier-3 docs (ext4, btrfs, xfs each have their own) | Identical |
| `/sys/fs/cgroup/*` (cgroupfs mount; cgroup framework owns content) | `kernel/cgroup/00-overview.md` (delegate) | Identical |
| `/sys/fs/bpf/*` (BPF pin filesystem) | `kernel/bpf/00-overview.md` (delegate) | Identical |
| `/sys/fs/fuse/*` | `stacked-fs/fuse.md` | Identical |
| `/sys/fs/{ext4,btrfs,xfs,f2fs}/*` | per-FS docs | Identical |
| `/sys/fs/fscrypt/*` | `fs-crypto.md` | Identical |
| `/sys/fs/verity/*` | `fs-verity.md` | Identical |

### Other userspace-visible interfaces

- **`mount(8)` syntax + `/proc/mounts` format** — preserved.
- **fanotify, inotify, dnotify event format** — preserved.
- **POSIX file-locking semantics** (advisory + mandatory + leases + OFD) — preserved.
- **`statx(2)` `STATX_*` mask bits and `struct statx` layout** — preserved.
- **`open_tree(2)` + `mount_setattr(2)` + `move_mount(2)` "new mount API"** — preserved.
- **fs-crypto policy format and `fscrypt(2)` ioctls** — preserved.
- **fs-verity Merkle tree format** — preserved.
- **Per-filesystem on-disk formats** (ext4 superblock, btrfs B-trees, xfs allocation groups) — preserved EXACTLY for in-port-set FSes; not relevant for FSes Rookery doesn't re-implement.
- **netlink-based filesystem events** (e.g., dm netlink) — preserved.
- **NFS RPC wire format** (NFSv3, NFSv4, NFSv4.1, pNFS) — preserved.
- **CIFS/SMB wire format** — preserved.
- **9P wire format** — preserved.

### binfmt + ELF loader interface

- **`binfmt_elf` consumes ELF binaries** with `e_machine = EM_X86_64`, sets up the `auxv`, mmap segments, and stack arguments. Owned by `exec-binfmt.md`. Compat-critical — every userspace ELF must execute correctly.
- **`binfmt_misc`** allows registering interpreters via `/proc/sys/fs/binfmt_misc/`. Preserved to support distros' qemu-user-static + Java-class-loader registrations.
- **`binfmt_script`** handles `#!` shebang. Preserved.
- **`binfmt_flat`, `binfmt_elf_fdpic`** — non-MMU formats; out of v0 scope for x86_64.

## Requirements

- REQ-1: Every filesystem syscall is implemented byte-identically with upstream — entry/exit conventions, register usage, errno returns, path-resolution semantics, struct layouts (`struct stat`, `struct statx`, `struct fsxattr`, `struct file_dedupe_range`, etc.).
- REQ-2: VFS path resolution (`namei.c` semantics) is byte-identical: chroot/chdir handling, symlink expansion, `..` handling, mountpoint crossing, namespace handling, idmapped-mount handling, RCU-walk + ref-walk transition.
- REQ-3: VFS data model (`inode`, `dentry`, `file`, `superblock`, `vfsmount`) preserves the userspace-observable invariants — multiple `file`s can share an `inode`, multiple `dentry`s can share an `inode` (hardlinks), the dcache is RCU-readable.
- REQ-4: `/proc`, `/sys`, `/dev/pts`, `/sys/fs/cgroup`, `/sys/kernel/debug`, `/sys/kernel/tracing` (the compat-critical pseudo-FSes) are byte-identical in path layout and content format. (See `mm/00-overview.md` for /proc fields owned by mm; this overview owns the procfs IMPLEMENTATION mechanics in `pseudo-fs/proc.md`.)
- REQ-5: tmpfs and ramfs implement `tmpfs(5)` semantics identically: extended-attribute support, mount options (size, nr_inodes, mode, uid, gid, huge), POSIX acl support.
- REQ-6: hugetlbfs preserves its mount options, page-size-per-mount, and the `/sys/kernel/mm/hugepages/` interaction.
- REQ-7: ELF loading (binfmt_elf): every userspace ELF executable that loads under upstream loads identically under Rookery. Identical `auxv` content, same stack setup, same vDSO mapping.
- REQ-8: For each disk-FS in the v0 port set (decided per Q1 below): on-disk format compat is byte-identical (existing on-disk filesystems mount cleanly); userspace-visible behavior (allowed mount options, ioctls) is identical.
- REQ-9: For each FS NOT in the v0 port set: those filesystems either continue using the upstream C implementation (via rust-for-linux interop) or are unsupported in Rookery (CONFIG_<FS>=n by default). The default-on / default-off list is enumerated in Q1 resolution.
- REQ-10: fsnotify (inotify, fanotify, dnotify) event format is byte-identical; existing tools (inotifywait, auditd consumer of fsnotify, gio's monitor) work unmodified.
- REQ-11: POSIX file-locking semantics (`flock`, `fcntl(F_SETLK*)`, `fcntl(F_OFD_SETLK*)`, leases) match upstream exactly.
- REQ-12: fs-crypto policy storage and key-handling ioctl ABI match upstream.
- REQ-13: fs-verity Merkle tree format and `FS_IOC_ENABLE_VERITY` / `FS_IOC_MEASURE_VERITY` ABI match upstream.
- REQ-14: io_uring's filesystem-operations cohabit with the VFS path correctly: fixed-files, registered-buffers, async-read/write/openat semantics match upstream.
- REQ-15: All Tier-3 docs spawned by this overview each declare their unsafe-block clusters, TLA+ models (where novel concurrency primitives are introduced), and Kani harnesses (mandatory Layer-3 for `vfs/dcache.md` invariants).
- REQ-16: `fs/dcache` is one of the four MANDATORY Layer-3 subsystems per `00-overview.md` D4 — Kani invariant harnesses for the dcache are required.
- REQ-17: `coredump.c` produces ELF core files with byte-identical layout to upstream (`gdb`, `crash`, `drgn`, `lldb` consume them).

## Acceptance Criteria

- [ ] AC-1: `strace -e trace=file,desc,fsmount,mount` of a kernel-build test produces byte-identical traces on Rookery and upstream. (covers REQ-1)
- [ ] AC-2: VFS namei test suite (`tools/testing/selftests/filesystems/`) passes with the same pass/fail set as upstream. (covers REQ-2, REQ-3)
- [ ] AC-3: A boot under Rookery vs. upstream with identical initramfs produces a diff-clean `/proc` and `/sys` content listing (modulo addresses/timestamps). (covers REQ-4)
- [ ] AC-4: tmpfs selftests (`tools/testing/selftests/tmpfs/`) pass. (covers REQ-5)
- [ ] AC-5: hugetlbfs selftests pass; mount/unmount with various pagesizes succeeds; `/sys/kernel/mm/hugepages/` knobs round-trip. (covers REQ-6)
- [ ] AC-6: A 4 MiB-50 GiB sample of real-world ELF binaries from `/usr/bin/` and `/usr/lib/` execute correctly under Rookery. (covers REQ-7)
- [ ] AC-7: For each disk-FS in the v0 port set: `mkfs.<fs>` / `mount` / random-write workload / `fsck.<fs>` round-trip succeeds against an upstream-formatted image and a Rookery-formatted image read by upstream. (covers REQ-8)
- [ ] AC-8: For each FS NOT in v0 port set, CONFIG_<FS> defaults match a documented project decision (most are CONFIG=n; some are CONFIG=m with C-impl); a smoke test mounts a sample image with each `m`-default FS. (covers REQ-9)
- [ ] AC-9: `inotifywait` against a churning directory produces byte-identical event streams (modulo timestamps) on Rookery vs. upstream. (covers REQ-10)
- [ ] AC-10: `flock(2)` and `fcntl(F_SETLK)` selftests pass. (covers REQ-11)
- [ ] AC-11: fscrypt selftest mounts an fscrypt-encrypted volume created on upstream and reads the decrypted content correctly. (covers REQ-12)
- [ ] AC-12: fs-verity test verifies an upstream-generated Merkle tree against Rookery and vice versa. (covers REQ-13)
- [ ] AC-13: `io_uring` selftests under `tools/testing/selftests/io_uring/` pass identically. (covers REQ-14)
- [ ] AC-14: `make verify` runs all `kernel/fs/<area>/proofs/` Kani harnesses; `make tla` runs all models. Both pass. (covers REQ-15, REQ-16)
- [ ] AC-15: A core file produced from a SIGSEGV under Rookery is consumable by `gdb`, `lldb`, `drgn`, and `crash` against the same `vmlinux`. (covers REQ-17)

## Architecture

### Layout map (Tier-3 docs spawned from this overview)

```
.design/fs/
  00-overview.md              ← this document
  vfs/
    00-overview.md            ← VFS hub
    inode.md                  ← struct inode + inode operations vtable
    dcache.md                 ← struct dentry + RCU-walk + dcache hash
    super.md                  ← struct super_block + super_operations
    file-table.md             ← struct file + fdtable + close_on_exec
    mount.md                  ← vfsmount + mount namespace + propagation
    path-resolution.md        ← namei.c + LOOKUP_* flags + RCU-walk vs. ref-walk
    fs-context.md             ← modern mount API (fsopen/fsmount/fsconfig/move_mount)
  syscalls.md                 ← Tier-2 cross-reference: fs syscalls map to Tier-5 docs
  exec-binfmt.md              ← exec.c + binfmt_elf + binfmt_misc + binfmt_script + coredump
  iomap.md                    ← iomap I/O abstraction
  netfs.md                    ← network-FS helper layer
  jbd2.md                     ← journaling block device (used by ext4)
  fs-crypto.md                ← per-file encryption framework
  fs-verity.md                ← per-file authenticated read framework
  fsnotify.md                 ← inotify + fanotify + dnotify
  quota.md                    ← disk quota enforcement
  buffer-cache.md             ← buffer_head / mbcache / mpage (legacy)
  direct-io-dax.md            ← direct I/O + DAX (Direct Access for persistent memory)
  pipe-splice.md              ← pipe + splice + tee + vmsplice
  event-fds.md                ← eventfd + epoll + signalfd + timerfd
  aio.md                      ← Linux AIO (separate from io_uring)
  charset.md                  ← unicode + nls (encoding helpers)
  pseudo-fs/
    00-overview.md            ← pseudo-FS hub
    proc.md                   ← procfs implementation (compat-critical)
    sysfs.md                  ← sysfs implementation (compat-critical)
    devpts.md                 ← /dev/pts pty filesystem
    ramfs-tmpfs.md            ← ramfs + tmpfs
    hugetlbfs.md              ← explicit-huge-pages FS
    debugfs.md                ← /sys/kernel/debug/
    configfs.md               ← /sys/kernel/config/
    tracefs.md                ← /sys/kernel/tracing/
    efivarfs.md               ← UEFI variable filesystem
    pstore.md                 ← persistent-storage event log
    kernfs.md                 ← in-memory hierarchical kernel filesystem (substrate for sysfs/cgroupfs)
  disk-fs/
    00-overview.md            ← disk-FS class hub
    ext4.md                   ← (v0 port set candidate)
    btrfs.md                  ← (v0 port set candidate)
    xfs.md                    ← (v0 port set candidate)
    f2fs.md                   ← (v0 port set candidate, per Q1)
    fat-vfat.md               ← (compat with EFI partition; in v0)
    exfat.md                  ← (per Q1)
    ntfs3.md                  ← (per Q1)
    iso9660.md                ← (CD/DVD; minor)
    squashfs.md               ← (read-only; in v0 for Snap/AppImage compat)
    erofs.md                  ← (read-only)
  network-fs/
    00-overview.md            ← network-FS class hub
    nfs-client.md             ← (NFSv3/v4 client; v0 port set per Q1)
    nfs-server.md             ← (server; per Q1)
    cifs-smb.md               ← (per Q1)
    ceph.md                   ← (per Q1)
    9p.md                     ← (lightweight; useful for VMs)
    afs.md                    ← (rare; out of v0 likely)
  stacked-fs/
    00-overview.md            ← stacked-FS class hub
    overlayfs.md              ← (REQUIRED — container substrate)
    fuse.md                   ← (REQUIRED — userspace-FS interface, used by sshfs/ntfs-3g/etc.)
    ecryptfs.md               ← (per Q1)
    autofs.md                 ← (per Q1)
```

### Cross-references

- `mm/00-overview.md` — page cache (filemap.c) lives in mm/, but is consumed extensively by fs/. Cross-ref from `iomap.md` and per-FS docs.
- `kernel/00-overview.md` — fs syscalls share entry path with kernel-/cgroup-/perf- syscalls; cgroupfs is a fs/ implementation but cgroup framework is in kernel/.
- `block/00-overview.md` (Phase B) — disk filesystems consume the block layer.
- `arch/x86/00-overview.md` — `binfmt_elf` is x86-specific in its `e_machine = EM_X86_64` recognition.
- `crypto/00-overview.md` (Phase B) — fs-crypto consumes the kernel crypto API.
- `io_uring/00-overview.md` (Phase B) — io_uring is a sibling subsystem but heavily uses fs/.
- `security/00-overview.md` (Phase B) — LSM hooks instrument every `fs/` operation.
- `00-glossary.md` — `inode`, `dentry`, `file`, `super_block`, `vfsmount`, `path`, `page cache`.

### Rust module organization (informative)

Some upstream coverage exists; Rookery extends:

- `kernel::fs` — exists in upstream rust/kernel/fs/; Rookery extends with VFS-core abstractions
- `kernel::block` — exists (`rust/kernel/block.rs` + `rust/kernel/block/`); cross-ref to block/00-overview.md
- `kernel::cred` — exists; consumed by fs for permissions
- `kernel::mm` — page-cache interaction (mm-owned)

Rookery to author for fs/:
- `kernel::fs::dcache` — wrapper around dcache + RCU-walk
- `kernel::fs::namei` — path resolution
- `kernel::fs::file` — file table + fdtable
- `kernel::fs::mount` — mount namespace
- `kernel::fs::iomap` — iomap I/O abstraction
- `kernel::fs::pseudo::{proc, sysfs, kernfs}` — pseudo-FS infrastructure
- `kernel::fs::binfmt::elf` — ELF loader
- `kernel::fs::fsnotify` — fsnotify substrate
- `kernel::fs::lock` — POSIX/OFD/lease locking

### Locking and concurrency

VFS is heavily concurrent. Locking landscape:
- **inode locks**: `i_rwsem` (sleepable rw), `i_lock` (spinlock for atomic field updates), `i_pages` (xarray lock for page cache).
- **dentry locks**: `d_lock` (spinlock per dentry); RCU for the RCU-walk fast path.
- **superblock locks**: `s_umount` (rwsem); `sb->s_inode_list_lock`.
- **mount tree**: `namespace_sem` (rwsem); `mount_lock` (seqlock).
- **file lock**: `flc_lock` (spinlock) for the file's `file_lock_context`.
- **fdtable**: `files->file_lock` (spinlock); RCU for read.
- **fsnotify**: per-watch refcount + RCU for traversal.

`vfs/dcache.md` Tier 3 details the RCU-walk algorithm with TLA+ model (`models/fs/dcache_rcu_walk.tla`).

### Error handling

`fs/` returns `Result<T, KernelError>` everywhere fallible. Specific returns:
- `Err(EACCES)` — permission denied (DAC, MAC, capabilities)
- `Err(ENOENT)` — path component missing
- `Err(EISDIR)` / `Err(ENOTDIR)` — type mismatch
- `Err(EXDEV)` — cross-device link / rename
- `Err(EBUSY)` — busy (mount, unmount, etc.)
- `Err(ENOSPC)` / `Err(EDQUOT)` — out of space / quota
- `Err(EROFS)` — read-only filesystem
- `Err(ESTALE)` — NFS stale handle
- `Err(ENAMETOOLONG)` — path too long
- `Err(ELOOP)` — too many symlinks
- `Err(EFBIG)` — file too large

## Verification

### Layer 1: Kani SAFETY proofs

Anticipated `unsafe` clusters:
- dcache RCU-walk: `kernel::fs::dcache::*` — pointer-chase across unprotected memory under RCU read-side
- file-table dup/swap on fork/exec: `kernel::fs::file::*`
- buffer_head ↔ folio interactions: `kernel::fs::buffer::*`
- ELF loader segment mapping: `kernel::fs::binfmt::elf::*`
- iomap raw block-device interactions: `kernel::fs::iomap::*`
- fsnotify mark traversal: `kernel::fs::fsnotify::*`
- namei symlink resolution: `kernel::fs::namei::*`

### Layer 2: TLA+ models (mandatory for novel concurrency)

- `models/fs/dcache_rcu_walk.tla` — proves RCU-walk's "no torn read" invariant: even when a parent is renamed during the walk, the walker either succeeds with a coherent path or fails to ENOENT (never reads garbage).
- `models/fs/mount_propagation.tla` — proves shared/private/slave mount-propagation rules preserve the mount-namespace tree's invariants.
- `models/fs/inode_concurrent_link.tla` — proves concurrent link/unlink on the same target inode doesn't corrupt the i_link counter.
- `models/fs/file_lock_compat.tla` — proves POSIX/OFD locks coexist with BSD flock without deadlock or starvation.
- `models/fs/fsnotify_event_order.tla` — proves event ordering semantics: events for a single file appear in the order operations completed (not started).
- `models/fs/jbd2_journal.tla` — proves journal-replay produces a state equivalent to a successful commit, even after crash.

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY for `vfs/dcache.md`)

Per `00-overview.md` D4, fs/dcache is one of the four mandatory Layer-3 subsystems:

| Data structure | Invariant | Harness |
|---|---|---|
| dcache hash | "Every dentry's parent is reachable from a fs root, and the parent's child list contains the dentry" | `kani::proofs::fs::dcache::tree_invariants` |
| dcache LRU | "Every dentry on the LRU list has refcount==0 and the corresponding LRU flag set" | `kani::proofs::fs::dcache::lru_invariants` |
| inode list (per-superblock) | "Every linked inode has its superblock's `s_inodes` list containing it" | `kani::proofs::fs::inode::sb_list_invariants` |
| file table | "Every fd is either NULL or points to a `file` whose refcount is at least 1" | `kani::proofs::fs::file::fdtable_invariants` |
| mount tree | "Every mountpoint dentry has the corresponding `vfsmount->mnt_mountpoint` field" + "Mount tree is acyclic" | `kani::proofs::fs::mount::tree_invariants` |
| posix locks per inode | "No two locks of conflicting types overlap on the same inode in a way disallowed by POSIX semantics" | `kani::proofs::fs::lock::posix_invariants` |

### Layer 4: Functional correctness via Creusot / Verus / Prusti (opt-in)

Strong opt-in candidates:
- `binfmt_elf.md` — ELF parser correctness via Creusot. Parser is well-defined and a frequent attack surface; provable correctness substantially reduces CVE risk.
- `fs-crypto.md` — crypto primitives are Verus targets (per `lib/lightweight-crypto.md`); the fs-side glue is harder.
- `jbd2.md` — journaling correctness theorem (after replay, on-disk state matches a serializable transaction history) is a major formal-methods opportunity. May slip past v0.
- `iomap.md` — I/O ordering invariants under concurrent reads/writes; lower priority.

## Hardening

Placeholder per `00-overview.md` D6. fs/ owns implementation of:

- **GRKERNSEC_CHROOT_*-equivalent** suite: chroot hardening (no double-chroot, no mknod-in-chroot, etc.) — many of these are already upstream Linux config flags; Rookery preserves them.
- **fs-crypto**: per-file encryption.
- **fs-verity**: tamper-evident reads.
- **GRKERNSEC_TPE-equivalent** (Trusted Path Execution): exec restriction by path/uid — to be evaluated in `00-security-principles.md`.
- **DAC + POSIX-ACL** + **xattrs**: standard Linux file permissions; preserved.
- **Idmapped mounts**: namespace-isolated permission models; preserved.
- **fanotify perm hooks**: AV scanning support; preserved.

## Resolved Decisions

### D1 (2026-05-09): Filesystem v0 port set — 3-bucket split locked

- **Full Rust port + Tier-3 doc** (compat-critical and reasonably scoped): ext4, btrfs, xfs (dominant Linux disk FSes); tmpfs (used everywhere); proc, sysfs, devpts, debugfs, tracefs, configfs, kernfs, hugetlbfs, efivarfs, pstore (compat-critical pseudo-FSes); overlayfs (containers); fuse (userspace-FS interface); 9p (VMs); iso9660 (CD/DVD); squashfs (Snap/AppImage); vfat (EFI partition).
- **FFI to upstream C — no Rust port in v0**: nfs client + nfsd server, cifs/smb, f2fs, exfat, ntfs3, ecryptfs, autofs, ceph, jbd2.
- **CONFIG=n by default**: afs, coda, orangefs, befs, bfs, cramfs, efs, freevxfs, hfs, hfsplus, hpfs, jfs, jffs2, minix, omfs, qnx4, qnx6, romfs, ubifs, ufs, ocfs2, gfs2, nilfs2, openpromfs, vboxsf, hostfs, adfs, affs. (exportfs IS in v0 but folded into vfs/ Tier-3.)

### D2 (2026-05-09): jbd2 stays as upstream C in v0
ext4's Rust implementation depends on jbd2 via FFI. `jbd2.md` Tier 3 documents the FFI contract; full Rust port of jbd2 deferred to v1+ (Layer-4-verification candidate). Q1's port-set list above assumes this.

### D3 (2026-05-09): NFS server (nfsd) FFI in v0
`fs/nfsd/` stays as upstream C with FFI shim. CONFIG=m by default; not a default-on subsystem. v1+ may revisit.

### D4 (2026-05-09): tmpfs huge-pages IN v0
`huge=always|never|within_size|advise` mount option supported per upstream. `pseudo-fs/ramfs-tmpfs.md` (Tier 3) covers integration with `mm/00-overview.md` § thp.md.

## Open Questions

(none — all open questions for this subsystem document are resolved above)

## Out of Scope

- 32-bit-only filesystem paths (consistent with `arch/x86/00-overview.md` D1).
- Filesystems explicitly listed in Q1 bucket C (CONFIG=n by default).
- jbd2 Rust port in v0 (per Q2 recommendation; deferred to v1+).
- AIO (`fs/aio.c`) is in scope but lower priority than io_uring; Tier-3 doc included but treated as legacy.
- Test fixtures (`fs/tests/`).
- Implementation code — `.design/` contains specs only.
