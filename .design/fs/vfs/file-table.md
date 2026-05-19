# Tier-3: fs/vfs/file-table — struct file + per-task fdtable + close_on_exec + close_range

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/file.c
  - fs/file_table.c
  - fs/file_attr.c
  - include/linux/file.h
  - include/linux/fdtable.h
  - include/linux/fs.h
  - include/uapi/linux/fcntl.h
  - include/uapi/linux/close_range.h
-->

## Summary
Tier-3 design for the per-open-file abstraction (`struct file`) and per-task file-descriptor table (`struct fdtable`). Owns `file` lifecycle (alloc on open → use → close → free), `file_operations` vtable, fd allocation + freeing within a task's `files_struct`, close-on-exec bitmap, `close_range(2)` syscall, file-attribute manipulation (`fileattr_get`/`set`), per-task file-table refcounting + clone semantics, and `dup`/`dup2`/`dup3` family.

Sub-tier-3 of `fs/vfs/00-overview.md`. The file-table is fork/exec-sensitive: clone with CLONE_FILES shares the table; clone without it copies; exec resets close-on-exec'd fds. Every fd-operation (read/write/ioctl/mmap/...) routes through here.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| File-table per-task mgmt | `fs/file.c` |
| Global file lifecycle (alloc_file / fput / put_filp) | `fs/file_table.c` |
| File attribute (fileattr_get/set ioctl backend) | `fs/file_attr.c` |
| Public types | `include/linux/file.h`, `include/linux/fdtable.h`, `include/linux/fs.h` |
| UAPI | `include/uapi/linux/fcntl.h`, `include/uapi/linux/close_range.h` |

## Compatibility contract

### `struct file` layout

`include/linux/fs.h` defines `struct file`. First-cache-line + commonly-accessed fields layout-equivalent:

- `f_u` (union: `fu_llist`, `fu_rcuhead`)
- `f_path` (path: vfsmount + dentry — cross-ref `fs/vfs/mount.md`)
- `f_inode` (inode pointer cache for fast-path)
- `f_op` (file_operations vtable)
- `f_lock` (spinlock)
- `f_flags` (O_* open flags)
- `f_mode` (FMODE_* mode flags)
- `f_count` (refcount; saturating)
- `f_version` (version counter for some FSes)
- `f_pos` (current read/write position)
- `f_cred` (credentials at open-time — for permission checks during async io)
- `f_owner` (struct fown_struct — for SIGIO/SIGURG delivery)
- `f_pos_lock` (mutex for f_pos updates)
- `f_security` (LSM blob)
- `f_mapping` (address_space)
- `f_wb_err`, `f_sb_err` (writeback/sb error sequence numbers for fsync error reporting)
- `f_ep` (eventpoll — used by epoll for back-pointer)

Layout-equivalent so out-of-tree code accessing common fields via macro works unchanged.

### `struct fdtable` layout

`include/linux/fdtable.h`:

- `max_fds` (current capacity)
- `fd[]` (array of `struct file *`)
- `close_on_exec` (bitmap)
- `open_fds` (bitmap)
- `full_fds_bits` (per-64-fd-block "full" bit)
- `rcu` (rcu_head for free)

Layout-equivalent to upstream.

### `struct files_struct` layout

`include/linux/fdtable.h`:

- `count` (refcount; CLONE_FILES shares this)
- `resize_in_progress`, `resize_wait`
- `fdt` (current fdtable pointer)
- `fdtab` (initial small fdtable)
- `file_lock`, `next_fd`
- `close_on_exec_init`, `open_fds_init`, `full_fds_bits_init` (initial bitmaps)
- `fd_array[NR_OPEN_DEFAULT]` (initial fd array — 64 entries on 64-bit)

Layout-equivalent.

### Syscalls

`open`, `openat`, `openat2`, `creat`, `close`, `close_range`, `dup`, `dup2`, `dup3`, `fcntl`, `flock`, `pipe`, `pipe2`, `socketpair` (returns 2 fds; cross-ref `net/`), `accept`, `accept4`, `eventfd`, `eventfd2`, `signalfd`, `signalfd4`, `timerfd_create`, `inotify_init`, `inotify_init1`, `fanotify_init`, `pidfd_open`, `pidfd_getfd`, `userfaultfd`, `memfd_create`, `memfd_secret`, `landlock_create_ruleset`, `bpf` (returns fd from BPF_PROG_LOAD, BPF_MAP_CREATE, etc.). Each gets a Tier-5 doc.

### `O_*` open flags

`include/uapi/asm-generic/fcntl.h` defines: `O_RDONLY=0`, `O_WRONLY=1`, `O_RDWR=2`, `O_CREAT=0x40`, `O_EXCL=0x80`, `O_NOCTTY=0x100`, `O_TRUNC=0x200`, `O_APPEND=0x400`, `O_NONBLOCK=0x800`, `O_DSYNC=0x1000`, `O_SYNC=0x101000`, `O_DIRECTORY=0x10000`, `O_NOFOLLOW=0x20000`, `O_CLOEXEC=0x80000`, `O_PATH=0x200000`, `O_TMPFILE=0x410000`, `O_LARGEFILE=0` on x86_64. Numeric values byte-identical.

### `FD_*` flags

`FD_CLOEXEC=1`. `fcntl(F_GETFD)` / `fcntl(F_SETFD)` operate on this bit.

### `close_range` flags

`include/uapi/linux/close_range.h`: `CLOSE_RANGE_UNSHARE=2`, `CLOSE_RANGE_CLOEXEC=4`. Numeric values byte-identical.

### `/proc/<pid>/fd/*` and `/proc/<pid>/fdinfo/*`

Per-fd symlinks + per-fd info (pos, flags, mnt_id, ino, eventfd-counter, etc.). Format-identical.

## Requirements

- REQ-1: `struct file` first-cache-line + commonly-accessed fields layout-equivalent.
- REQ-2: `file_operations` vtable byte-identical (per `fs/vfs/00-overview.md` REQ-1).
- REQ-3: `struct fdtable` + `struct files_struct` layouts byte-identical so existing kernel-internal code accessing fields via macros works.
- REQ-4: Open / close lifecycle: `alloc_file` → install in fdtable → use → `__close_fd` removes from fdtable + drops refcount → if 0, calls `__fput` → `f_op->release` → free.
- REQ-5: fd allocation: `__alloc_fd(files, start, end, flags)` returns the lowest fd ≥ start that's free; resize fdtable if needed.
- REQ-6: close-on-exec: per-task bitmap; on `execve`, fds with FD_CLOEXEC set are closed via `do_close_on_exec`; matches upstream.
- REQ-7: `close_range(2)` semantics: closes all fds in [first, last] inclusive; honors CLOSE_RANGE_UNSHARE + CLOSE_RANGE_CLOEXEC.
- REQ-8: dup family: `dup(oldfd)` returns lowest free fd; `dup2(oldfd, newfd)` closes newfd if open + duplicates oldfd to newfd; `dup3(oldfd, newfd, O_CLOEXEC)` ditto with cloexec.
- REQ-9: fcntl operations: F_DUPFD, F_DUPFD_CLOEXEC, F_GETFD, F_SETFD, F_GETFL, F_SETFL, F_SETLK, F_SETLKW, F_GETLK, F_SETOWN, F_GETOWN, F_SETOWN_EX, F_GETOWN_EX, F_SETSIG, F_GETSIG, F_OFD_SETLK, F_OFD_SETLKW, F_OFD_GETLK, F_NOTIFY, F_SETLEASE, F_GETLEASE, F_ADD_SEALS, F_GET_SEALS — byte-identical numeric values + semantics.
- REQ-10: Clone semantics: `clone(CLONE_FILES, ...)` shares `files_struct` (just bumps count); `clone()` without CLONE_FILES copies the fdtable. Identical.
- REQ-11: Per-fd file_struct + file ABI: `lsof` / `lsof -p <pid>` output matches upstream.
- REQ-12: `O_PATH` semantics: returned fd does NOT carry permission for read/write — only for path-resolution + linkat / openat with new flag.
- REQ-13: `O_TMPFILE` semantics: creates an unlinked file; subsequent linkat with target path + AT_EMPTY_PATH atomically links it.
- REQ-14: file-attribute ioctl (FS_IOC_GETFLAGS / SETFLAGS, FS_IOC_FSGETXATTR / FSSETXATTR) — semantic byte-identity.
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct file`, `struct fdtable`, `struct files_struct` first-cache-line layouts byte-identical vs. upstream. (covers REQ-1, REQ-3)
- [ ] AC-2: A test opens 1M files; alloc_fd returns lowest-free; resize occurs at NR_OPEN_DEFAULT boundary; refcount returns to baseline after close-all. (covers REQ-4, REQ-5)
- [ ] AC-3: A close-on-exec test: open fd with O_CLOEXEC + execve a child; child sees fd closed. (covers REQ-6)
- [ ] AC-4: `close_range(0, ~0U, CLOSE_RANGE_CLOEXEC)` marks all fds CLOEXEC; `close_range(0, ~0U, CLOSE_RANGE_UNSHARE)` unshares fdtable then closes. (covers REQ-7)
- [ ] AC-5: `dup` / `dup2` / `dup3` selftests pass byte-identically. (covers REQ-8)
- [ ] AC-6: Every `F_*` fcntl operation round-trips on a curated input set; output byte-identical. (covers REQ-9)
- [ ] AC-7: A clone(CLONE_FILES)-spawned thread shares the parent's fdtable; without CLONE_FILES, it copies. Verifiable via `/proc/<tid>/fd/`. (covers REQ-10)
- [ ] AC-8: `lsof -p <pid>` output for a curated test program byte-identical (modulo dynamic state). (covers REQ-11)
- [ ] AC-9: `O_PATH` test: read/write on the returned fd returns EBADF; openat with the fd as dirfd succeeds. (covers REQ-12)
- [ ] AC-10: `O_TMPFILE` test: `open(O_TMPFILE | O_RDWR)` returns an fd referring to an unlinked file; subsequent `linkat(fd, "", AT_FDCWD, target, AT_EMPTY_PATH)` materializes it. (covers REQ-13)
- [ ] AC-11: `FS_IOC_GETFLAGS` round-trips on a regular file. (covers REQ-14)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::fs::vfs::file::File` — `struct file` wrapper
- `kernel::fs::vfs::file::ops::FileOps` — file_operations vtable trait
- `kernel::fs::vfs::file::lifecycle` — alloc_file / fput / __fput
- `kernel::fs::vfs::file_table::FileTable` — `files_struct` wrapper
- `kernel::fs::vfs::file_table::Fdtable` — `fdtable` wrapper
- `kernel::fs::vfs::file_table::dup` — dup/dup2/dup3 helpers
- `kernel::fs::vfs::file_table::close_range` — close_range syscall
- `kernel::fs::vfs::file_table::cloexec` — close-on-exec bitmap
- `kernel::fs::vfs::file_attr` — fileattr_get/set ioctl backend

### Locking and concurrency

- **`files_struct->file_lock`** (spinlock): protects fdtable resize + fd alloc/free
- **`files_struct->fdt`** (RCU-protected pointer): reads via rcu_dereference
- **`file->f_lock`** (spinlock): atomic state-flag updates
- **`file->f_pos_lock`** (mutex): held during f_pos updates for read/write that need it (e.g., regular files; sockets/pipes don't need)

### Error handling

- `Err(EBADF)` — bad fd
- `Err(EMFILE)` — process fd limit reached
- `Err(ENFILE)` — system fd limit reached
- `Err(EBUSY)` — fdtable resize in progress
- `Err(EACCES)` — permission denied during open

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Fdtable resize (atomic swap of fdt under file_lock) | `kani::proofs::fs::vfs::file_table::resize_safety` |
| fd alloc bitmap manipulation | `kani::proofs::fs::vfs::file_table::alloc_safety` |
| dup-via-fdtable-pointer | `kani::proofs::fs::vfs::file_table::dup_safety` |
| close-on-exec walk | `kani::proofs::fs::vfs::file_table::cloexec_safety` |
| File refcount + RCU free | `kani::proofs::fs::vfs::file::refcount_safety` |

### Layer 2: TLA+ models

(none mandatory at this sub-tier; concurrency is per-task — RCU readers + per-task lock writers)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| File table | Every fd is either NULL or points to a `file` whose refcount is at least 1 (mandatory per `fs/00-overview.md`) | `kani::proofs::fs::vfs::file_table::fdtable_invariants` |
| open_fds bitmap | Every set bit corresponds to a non-NULL fd; every clear bit corresponds to a NULL fd | `kani::proofs::fs::vfs::file_table::open_fds_invariants` |
| close_on_exec bitmap | Subset of open_fds; bits cleared when fd closed | `kani::proofs::fs::vfs::file_table::cloexec_invariants` |

### Layer 4: Functional correctness (opt-in)

- **fdtable resize correctness** via Verus — proves: under any sequence of alloc + close + dup + resize, the post-resize fdtable preserves all open fds.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | `file->f_count`, `files_struct->count` use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | files_struct + file allocated via per-type slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed file + files_struct objects zeroed unless cache marked SLAB_NO_ZERO_ON_FREE | § Default-on configurable off |
| **close-on-exec hardening** | O_CLOEXEC default-on for new fd-creating syscalls (per Rookery extension); existing syscalls preserve upstream defaults | § Default-on configurable off (potential Rookery escalation noted in Q1 below) |

### Row-1 features consumed by this component

- **UDEREF**: never accepts user pointers directly; argv/envp etc. handled at exec layer (cross-ref `fs/exec-binfmt.md`)
- **SIZE_OVERFLOW**: fd-count + alloc-bitmap arithmetic uses checked operators
- **CONSTIFY**: file_operations vtables provided by per-FS / per-driver modules are `static const`

### Row-2 / GR-RBAC integration

LSM hooks dispatched:
- `security_file_alloc` / `security_file_free` — file lifecycle
- `security_file_open` — open hook
- `security_file_permission` — read/write permission check
- `security_file_ioctl` / `security_file_fcntl`
- `security_file_lock` / `security_file_set_fowner` / `security_file_send_sigiotask`
- `security_file_receive` (SCM_RIGHTS-passed file)
- `security_dentry_create_files_as` / `security_dentry_init_security`

GR-RBAC (per its loaded policy) can deny open/dup/fcntl operations as part of TPE / file-restriction policy.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy_to/from_user on `struct flock`, fown_struct, owner-ex, and fdinfo-emit buffers.
- **PAX_KERNEXEC** — W^X for file_operations vtables (open/read/write/ioctl/mmap dispatch tables).
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every fd-touching syscall (open/dup/close/close_range/fcntl).
- **PAX_REFCOUNT** — saturating refcount on `file->f_count`, `files_struct->count`; wrap-to-zero UAF closed (this was the CVE-class refcount-overflow surface).
- **PAX_MEMORY_SANITIZE** — zero-on-free for `struct file` slabs, `files_struct` slabs, fdtable arrays + cloexec/open_fds bitmaps.
- **PAX_UDEREF** — strict user-pointer access for fcntl arg, flock structs, owner-set payloads.
- **GRKERNSEC_HIDESYM** — hide file/files_struct addresses in `/proc/<pid>/fd/*` symlinks + tracepoints + lsof output.
- **GRKERNSEC_LINK** — `linkat(AT_EMPTY_PATH)` from an O_TMPFILE fd consults protected_hardlinks against the fd's `f_cred`.
- **GRKERNSEC_CHROOT_FCHDIR** — block `fchdir(2)` to a dirfd captured outside the calling task's chroot.
- **GRKERNSEC_CHROOT_FINDTASK** — block `pidfd_getfd(2)` across chroot boundary.
- **GRKERNSEC_FIFO** — pipe(2)/pipe2(2) honor FIFO-in-sticky-dir if a named pipe is materialized.
- **GRKERNSEC_TRUSTED** — F_SETLEASE / F_SETOWN_EX gated against per-fd `f_cred` capabilities.
- **GRKERNSEC_PROC** — `/proc/<pid>/fd/*` enumeration honors hidepid + restricts cross-uid visibility.
- **GRKERNSEC_SUIDDUMP** — coredumps blocked for suid-exec'd tasks; consulted during fd-table inheritance across exec.
- **O_CLOEXEC default-on** — per Open Question Q1, modern Rookery default closes legacy-syscall inheritance window (knob `kernel.legacy_fd_inherit=1` reverts).
- **PAX_SIZE_OVERFLOW** — fd-count + alloc-bitmap arithmetic uses checked operators (fdtable resize math).

Per-doc rationale: the file-table is the credential-bearing handle layer that survives fork/clone/exec; historical exploits (CVE-2016-4557, CVE-2021-4083) have all targeted refcount races or close-race UAFs on `struct file`. PAX_REFCOUNT + PAX_MEMORY_SANITIZE close those classes; GRKERNSEC_CHROOT_FCHDIR + GRKERNSEC_CHROOT_FINDTASK + O_CLOEXEC-default + GRKERNSEC_SUIDDUMP eliminate the fd-inheritance / pidfd-getfd cross-trust-domain leaks.

## Open Questions

<!-- OPEN: Q1 -->
### Q1: Default O_CLOEXEC for non-CLOEXEC-aware syscalls
Most modern syscalls have CLOEXEC variants (open with O_CLOEXEC, dup3 with O_CLOEXEC, accept4 with SOCK_CLOEXEC, etc.). Older syscalls (`open` without O_CLOEXEC, `dup`, `accept`) leave fds inherited across exec by default, which is a CVE-class issue (privileged daemons inheriting fds across exec).

**Recommendation**: Rookery considers default-on `O_CLOEXEC` for legacy non-CLOEXEC syscalls when called from non-init-task contexts. This is a userspace-visible behavior change (fds are closed across exec when previously they weren't).

**To resolve**: User confirms (yes — strict default-on; provides knob `kernel.legacy_fd_inherit=1` for legacy compat) OR rejects (preserve upstream default of inheritable fds).
<!-- /OPEN -->

## Out of Scope

- per-fd locking (cross-ref `fs/00-overview.md` § locks.c)
- mount tree (cross-ref `fs/vfs/mount.md`)
- 32-bit-only paths
- Implementation code
