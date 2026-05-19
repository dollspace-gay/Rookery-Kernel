# Tier-3: fs/proc/proc-fd — /proc/<pid>/fd/ + /proc/<pid>/fdinfo/

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/proc/fd.c
  - fs/proc/fd.h
  - fs/proc/internal.h
-->

## Summary
Tier-3 design for the `/proc/<pid>/fd/` symlink directory + `/proc/<pid>/fdinfo/` per-fd metadata directory. The `fd/` subdirectory exposes per-task open file-descriptors as symlinks resolving to the underlying `<scheme>:[<id>]` (or filesystem path); `fdinfo/` exposes per-fd metadata (offset, flags, mount-id, plus per-fd-type extras like eventfd-counter or epoll-watched-fds).

Used by:
- `lsof` (parses `/proc/<pid>/fd/` to enumerate open files per process)
- `ls -l /proc/<pid>/fd` for ad-hoc debugging
- Container-runtime introspection (verify which file descriptors the container has open)
- BPF / pidfd / eventfd debug tooling
- systemd's per-service fd-passing introspection
- Per-FD memfd_secret debugging

Sub-tier-3 of `fs/proc/00-overview.md`. Pairs with `fs/proc/proc-task.md` (parent /proc/<pid>/ infrastructure).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| /proc/<pid>/fd/ + /fdinfo/ implementation | `fs/proc/fd.c` |
| Internal helpers | `fs/proc/fd.h`, `fs/proc/internal.h` |

## Compatibility contract

### `/proc/<pid>/fd/<n>` symlink format

Each numbered entry resolves to one of:

| Underlying type | Symlink target |
|---|---|
| Regular file | absolute path (e.g., `/etc/passwd`) |
| Directory | absolute path |
| Symlink (followed at open) | resolved-to path |
| Pipe | `pipe:[<inode>]` |
| Socket | `socket:[<inode>]` |
| anon_inode | `anon_inode:<name>` (e.g., `anon_inode:[eventfd]`, `anon_inode:[signalfd]`, `anon_inode:[timerfd]`, `anon_inode:[eventpoll]`, `anon_inode:[fanotify]`, `anon_inode:[inotify]`, `anon_inode:[bpf-prog]`, `anon_inode:[bpf-map]`, `anon_inode:[bpf-link]`, `anon_inode:[pidfd]`, `anon_inode:[userfaultfd]`, `anon_inode:[secretmem]`, `anon_inode:[mctp]`, `anon_inode:[io_uring]`, `anon_inode:[memfd]`, `anon_inode:[memfd_secret]`, `anon_inode:[uffd]`) |
| memfd | `memfd:<name> (deleted)` |
| /dev/* | underlying device path |

Format byte-identical so `lsof` output matches.

### `/proc/<pid>/fdinfo/<n>` per-fd metadata

Common fields (all FD types):
```
pos:	<file-offset>
flags:	0<octal-O_*-flags>
mnt_id:	<mount-id>
ino:	<inode-number>
```

Plus per-fd-type extras:

#### eventfd
```
eventfd-count: <hex value>
eventfd-id: <eventfd-id>
eventfd-semaphore: 0|1
```

#### signalfd
```
sigmask: <hex bitmask>
```

#### epoll
Per-watched fd:
```
tfd: <target-fd> events: <hex events bitmap> data: <hex u64> pos:<file-pos> ino:<sub-inode> sdev:<sub-dev>
```

#### inotify
Per-watch:
```
inotify wd:<watch-id> ino:<inode> sdev:<dev> mask:<events> ignored_mask:<events> fhandle-bytes:<n> fhandle-type:<n> f_handle:<hex>
```

#### fanotify
```
fanotify ino:<inode> sdev:<dev> mflags:<flags> mask:<events> ignored_mask:<events> fhandle-bytes:<n> fhandle-type:<n> f_handle:<hex>
```

#### pidfd
```
Pid: <pid-in-pidns>
NSpid: <chain of pids per nested pidns>
```

#### BPF prog / map / link
```
prog_type:	<type>
prog_jited:	<bool>
prog_tag:	<hex>
prog_id:	<id>
... (per-type extras like map_id, btf_id, etc.)
```

#### io_uring
```
SqMask:		<hex>
SqHead:		<n>
SqTail:		<n>
CachedSqHead:	<n>
CqMask:		<hex>
CqHead:		<n>
CqTail:		<n>
CachedCqTail:	<n>
SQEs:		<entries>
CQEs:		<entries>
SqThreadCpu:	<n>
SqThreadIdle:	<ms>
```

#### memfd
Per-shmem inode:
```
size:	<bytes>
seals:	<F_SEAL_* flags>
```

#### userfaultfd
```
pending:	<count>
total:		<count>
API:		<hex version>
```

Format byte-identical for each fd-type; `lsof -i :<port>` and FD-aware tools work.

### Per-fd permissions

Each `fd/<n>` has the underlying file's open-mode (O_RDONLY/O_WRONLY/O_RDWR) reflected in `i_mode`. Symlink read returns target path; `O_PATH` open of a symlink works for re-opening the underlying file via `procfs-bind-mount` trick.

### `fdinfo/<n>` reads — `proc_fdinfo_show`

Per-fd-type emit via `f_op->show_fdinfo(seq, f)` callback. Per-fd-type modules export their own `show_fdinfo` implementation. Format-byte-identical for each fd-type.

### Permission model

Reading `/proc/<pid>/fd/<n>` symlinks normally requires:
- Same uid as task OR CAP_DAC_READ_SEARCH OR CAP_SYS_PTRACE

`fdinfo/<n>` reads require:
- Same uid as task OR CAP_SYS_ADMIN

These are checked via `ptrace_may_access` (PTRACE_MODE_READ).

### Subdirectory entries

`/proc/<pid>/fd/` directory listing returns entries `0`, `1`, `2`, ..., for each open FD. Per-task `task->files->fdt` bitmap walked to enumerate.

`/proc/<pid>/fdinfo/` parallel structure with same FD numbers.

## Requirements

- REQ-1: `/proc/<pid>/fd/<n>` symlink resolves per-FD-type to canonical scheme/path; format byte-identical.
- REQ-2: `/proc/<pid>/fdinfo/<n>` common fields (`pos`/`flags`/`mnt_id`/`ino`) byte-identical; per-fd-type `show_fdinfo` callback dispatched per-FD type.
- REQ-3: Per-fd-type extras: eventfd / signalfd / epoll / inotify / fanotify / pidfd / BPF / io_uring / memfd / userfaultfd byte-identical format.
- REQ-4: anon_inode-class FDs dispatched per their `anon_inode_inode` `i_op->get_link` which returns `<class>:[<id>]`-style target.
- REQ-5: Permission gate via `ptrace_may_access(task, PTRACE_MODE_READ)`; same-uid OR CAP_DAC_READ_SEARCH OR CAP_SYS_PTRACE.
- REQ-6: Per-fd subdirectory listing walks `task->files->fdt` bitmap; entries match.
- REQ-7: O_PATH open of /proc/<pid>/fd/<n> returns valid fd referencing same underlying file (proc-fd-bind-mount-equivalent).
- REQ-8: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `ls -l /proc/$$/fd/` shows entries 0/1/2 as `/dev/pts/X` (or pipe:[...] for redirection); each symlink readable. (covers REQ-1, REQ-6)
- [ ] AC-2: pipe test: parent `mkfifo /tmp/p; exec 3</tmp/p`; `readlink /proc/$$/fd/3` returns `/tmp/p`. (covers REQ-1)
- [ ] AC-3: socket test: open AF_UNIX socket; `readlink /proc/$$/fd/N` returns `socket:[<inode>]`. (covers REQ-1)
- [ ] AC-4: eventfd test: `int fd = eventfd(0, 0)`; `cat /proc/$$/fdinfo/N` shows `eventfd-count:`, `eventfd-id:`. (covers REQ-2, REQ-3)
- [ ] AC-5: epoll test: `int fd = epoll_create(1); epoll_ctl(fd, EPOLL_CTL_ADD, sock, &ev)`; `cat /proc/$$/fdinfo/N` lists `tfd:` for each watched fd. (covers REQ-2, REQ-3)
- [ ] AC-6: pidfd test: `int fd = pidfd_open(<child-pid>, 0)`; `cat /proc/$$/fdinfo/N` shows `Pid: <pid>` + `NSpid:`. (covers REQ-3)
- [ ] AC-7: BPF prog test: load BPF program; `cat /proc/$$/fdinfo/<bpf-fd>` shows `prog_type:`, `prog_id:`, etc. (covers REQ-3)
- [ ] AC-8: io_uring test: io_uring_setup; `cat /proc/$$/fdinfo/<ring-fd>` shows SQ/CQ ring metadata. (covers REQ-3)
- [ ] AC-9: Permission test: process A's /proc/A/fd readable by A and root; not readable by other unprivileged user (without CAP_SYS_PTRACE). (covers REQ-5)
- [ ] AC-10: O_PATH bind test: `int fd = open("/proc/$$/fd/0", O_PATH); fstat(fd, ...)` returns same dev/ino as fd 0. (covers REQ-7)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-8)

## Architecture

### Rust module organization

- `kernel::fs::proc::fd::ProcFd` — `/proc/<pid>/fd/` directory generator
- `kernel::fs::proc::fd::FdSymlink` — per-fd symlink generator (resolves to scheme/path)
- `kernel::fs::proc::fd::FdInfo` — `/proc/<pid>/fdinfo/` directory + per-fd metadata
- `kernel::fs::proc::fd::ShowFdinfo` — per-fd-type callback dispatch
- `kernel::fs::proc::fd::Permission` — `ptrace_may_access` gate
- `kernel::fs::proc::fd::AnonInode` — anon_inode class symlink resolution

### Locking and concurrency

- **Per-task `task->files->file_lock`** (spinlock): held during fd-table walk for directory listing
- **Per-fd refcount**: held during `fdinfo` readback to prevent close-during-read
- **RCU**: per-fd lookup hot path RCU-side via `rcu_dereference(fdt)`

### Error handling

- `Err(ESRCH)` — task gone
- `Err(EACCES)` — permission denied per ptrace_may_access
- `Err(ENOENT)` — fd closed mid-read
- `Err(EFAULT)` — userspace buffer fault

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-fd-table walk under file_lock (no double-acquire) | `kani::proofs::fs::proc::fd::walk_safety` |
| Per-fd refcount-acquire during fdinfo read (no use-after-free on close) | `kani::proofs::fs::proc::fd::ref_safety` |
| Per-fd-type `show_fdinfo` dispatch (no NULL fn-ptr deref) | `kani::proofs::fs::proc::fd::show_safety` |
| Permission gate via ptrace_may_access | `kani::proofs::fs::proc::fd::perm_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-task fd-table walk | every entry in fd-table bitmap corresponds to a non-NULL `struct file`; no orphan bits | `kani::proofs::fs::proc::fd::table_invariants` |
| Per-fd metadata format | per-fd-type `show_fdinfo` output is well-formed (terminating newlines, valid escape) | `kani::proofs::fs::proc::fd::format_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `fs/proc/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-task ref + per-file-table ref + per-file refcount use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | `proc_fd_inode_operations` + `proc_fd_dir_operations` `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed seq_file buffers cleared (carry per-fd path strings, possibly sensitive) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, CONSTIFY, MEMORY_SANITIZE**: see above
- **AUTOSLAB**: per-seq_file slab cache
- **USERCOPY**: per-symlink readlink + per-fdinfo seq_file uses bound-checked accessors
- **SIZE_OVERFLOW**: per-fd-table index arithmetic uses checked operators
- **KERNEXEC**: per-fd-type `show_fdinfo` dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_file_open` for /proc/<pid>/fd/<n> accesses (already standard).
- LSM hook `security_inode_permission` per-file gate.
- Default useful GR-RBAC policy: deny `/proc/<pid>/fd/` reads outside gradm-marked `debugger` role for non-self pids; arbitrary fd-introspection of other tasks reveals significant attack info.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

/proc/<pid>/fd is a cross-uid disclosure surface (fd numbers, target paths, dup-source inodes) that grsec historically restricted to gradm `debugger` role; Rookery encodes the equivalent contract:

- **PAX_USERCOPY** — readlink + fdinfo seq buffers bounded against per-allocation slab size; no copy_to_user past `PATH_MAX`.
- **PAX_KERNEXEC** — `proc_fd_inode_operations` and `proc_fd_dir_operations` are `static const`; per-fd-type `show_fdinfo` dispatch table is `__ro_after_init`.
- **PAX_RANDKSTACK** — kernel-stack offset randomized so per-fd open paths cannot anchor a kstack-grooming primitive.
- **PAX_REFCOUNT** — `get_task_struct`, `get_files_struct`/`put_files_struct`, and per-`struct file` refs use saturating refcount.
- **PAX_MEMORY_SANITIZE** — freed fdinfo seq buffers zeroed; per-fd path strings (potentially sensitive ipc/anon_inode names) cannot bleed.
- **PAX_UDEREF** — readlink and fdinfo emitters treat seq buffers as user-domain on copy_to_user; no kernel-pointer dereference smuggled via fd-path formatting.
- **PAX_RAP/kCFI** — `proc_fdinfo_inode_operations.show_fdinfo` function pointer is CFI-typed; per-fd-type vtable cannot be hijacked.
- **GRKERNSEC_HIDESYM** — `/proc/<pid>/fdinfo/<n>` strips kernel addresses (per-driver private state, anon_inode private_data ptrs) for non-CAP_SYSLOG readers.
- **GRKERNSEC_DMESG** — fd-walk denial paths never log target paths to dmesg.
- **/proc/<pid>/fd CAP_SYS_PTRACE for cross-uid** — fd readdir + readlink for `pid != current` require `ptrace_may_access(target, PTRACE_MODE_READ_FSCREDS)`; mirrors grsec `GRKERNSEC_PROC_USER` lockdown for fd introspection.
- **Per-fdinfo CAP_SYS_PTRACE gate** — fdinfo seq_show for cross-uid targets follows the same gate, so `pos`/`flags`/`mnt_id` of another process's fd is unreachable to unprivileged readers.
- **PR_SET_DUMPABLE=0** — when the target has cleared dumpable, `/proc/<pid>/fd` directory access is denied even to same-uid readers (matches grsec `GRKERNSEC_PROC_USERGROUP` interaction).

Rationale: fd-table enumeration is the cheapest oracle for "what is process X talking to" — sockets, pipes, perf_event, io_uring rings, ebpf maps — and grsec specifically targeted this with `GRKERNSEC_PROC_ADD` because lsof-style reconnaissance feeds nearly every local LPE chain.

## Open Questions

(none — /proc/<pid>/fd + /fdinfo semantics exhaustively specified by upstream + lsof + systemd test corpus)

## Out of Scope

- Per-fd-type underlying subsystems (cross-ref `kernel/eventfd.md`, `kernel/signalfd.md`, `fs/eventpoll.md`, `fs/inotify_user.md`, `fs/fanotify/`, `kernel/pidfd.md`, `kernel/bpf/00-overview.md`, `io_uring/`, etc. — many deferred)
- 32-bit-only paths
- Implementation code
