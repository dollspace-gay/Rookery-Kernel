# Tier-5 syscall: pipe(2) — syscall 22

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/pipe.c (SYSCALL_DEFINE1(pipe), __do_pipe_flags, create_pipe_files, alloc_pipe_info)
  - include/linux/pipe_fs_i.h (struct pipe_inode_info, pipe_buffer)
  - include/uapi/linux/fcntl.h (no flags on pipe(2); see pipe2(2) for O_CLOEXEC/O_DIRECT/O_NONBLOCK)
  - arch/x86/entry/syscalls/syscall_64.tbl (22  common  pipe)
-->

## Summary

`pipe(2)` is the legacy interface that creates a unidirectional FIFO byte-stream kernel object and returns two file descriptors referring to it: `pipefd[0]` for the read-end and `pipefd[1]` for the write-end. It is the no-flags ancestor of `pipe2(2)`; both flags `O_CLOEXEC` and `O_NONBLOCK` are implicitly clear, so the returned fds inherit the standard blocking semantics and survive `execve(2)`.

Pipes are the oldest and simplest form of Unix IPC. The kernel object is a circular ring of page-sized `pipe_buffer` slots (default 16, capped by `/proc/sys/fs/pipe-max-size`). Writes append to the ring; reads consume from the head; both block when the ring is full / empty respectively. Critical for: shell pipelines, `popen(3)`, `posix_spawn` file-action redirections, log-buffering in inetd-style servers, and CLOEXEC-free legacy callers that pre-date 2.6.27.

## Signature

```c
int pipe(int pipefd[2]);
```

```c
#define PIPE_BUF        4096    /* atomic-write threshold; POSIX-mandated minimum */
#define PIPE_DEF_BUFFERS 16     /* default ring slots */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pipefd` | `int[2]` | out | Two-element array; on success kernel writes pipefd[0] = read-end, pipefd[1] = write-end. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; fds populated in `pipefd`. |
| `-1` + `errno` | Failure; `pipefd` unchanged. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `pipefd` user pointer faults during copy_to_user. |
| `EMFILE` | Per-process `RLIMIT_NOFILE` exhausted (need two slots). |
| `ENFILE` | System-wide open-file limit (`/proc/sys/fs/file-max`) exhausted. |
| `ENOMEM` | Kernel cannot allocate `struct pipe_inode_info` or the slot ring. |

## ABI surface

```text
__NR_pipe   (x86_64)  = 22
__NR_pipe   (arm64)   = absent — arm64 / riscv / mips-n64 expose only pipe2(2); userland glibc emulates by calling pipe2(fd, 0).
__NR_pipe   (i386)    = 42

/* pipe(2) takes no flags argument. For O_CLOEXEC / O_NONBLOCK / O_DIRECT
   callers must use pipe2(2) (syscall 293 on x86_64). */
```

## Compatibility contract

REQ-1: Syscall number is **22** on x86_64. ABI-stable since v1.

REQ-2: On entry: validate that `pipefd` is a writable two-word user buffer; defer copy_to_user until both fds are installed (atomicity).

REQ-3: Internally invokes `__do_pipe_flags(fd, 0)`; flags argument is hard-coded zero, equivalent to `pipe2(pipefd, 0)`.

REQ-4: Two file descriptors are allocated via `get_unused_fd_flags(0)`; if the second allocation fails the first is released before returning `-EMFILE`.

REQ-5: A single `struct pipe_inode_info` is shared by both fds. The struct holds the ring (`bufs[]`), head/tail indices, max_usage (= PIPE_DEF_BUFFERS by default), nr_accounted (user-mlock accounting), `rd_wait` / `wr_wait` wait-queues, and a refcount.

REQ-6: Both ends are opened against a private anonymous inode of fstype `pipefs`. The inode is invisible to /proc, has no name, and is destroyed when the last fd is closed.

REQ-7: Reading from `pipefd[0]` when no data is buffered blocks (unless O_NONBLOCK is later set via fcntl); writing to `pipefd[1]` when the ring is full blocks; writes of at most `PIPE_BUF` bytes are atomic.

REQ-8: Closing the last write-end fd causes subsequent reads on the read-end to return EOF (zero bytes). Writing to a pipe whose last read-end is closed delivers SIGPIPE to the writer and returns `-EPIPE` if the signal is ignored.

REQ-9: Ring capacity can be queried/changed post-creation via `fcntl(F_GETPIPE_SZ)` / `fcntl(F_SETPIPE_SZ)`. Increases above the soft cap require `CAP_SYS_RESOURCE`; the hard cap is `/proc/sys/fs/pipe-max-size`.

REQ-10: Per-user accounting: total per-user pipe pages is bounded by `RLIMIT_PIPESIZE` (effective on Rookery; upstream uses `user_struct->pipe_bufs`).

REQ-11: Fds returned have FD_CLOEXEC **clear** by design (legacy behavior). Callers that need close-on-exec must call pipe2(2) with O_CLOEXEC or fcntl(F_SETFD, FD_CLOEXEC) before forking.

REQ-12: Per-`/proc/sys/fs/pipe-user-pages-soft` and `pipe-user-pages-hard`: caps on aggregate pipe memory per uid.

REQ-13: Per-fork: pipe fds are inherited; their underlying `pipe_inode_info` refcount increases. Closing all read- or all write-ends across all processes triggers EOF / SIGPIPE.

REQ-14: pipe(2) does not consume entropy, does not create namespace artifacts, and does not require any capability.

## Acceptance Criteria

- [ ] AC-1: `pipe(fd)` with valid buffer returns 0 and fd[0], fd[1] are distinct positive fds.
- [ ] AC-2: `write(fd[1], "hello", 5)` followed by `read(fd[0], buf, 5)` returns "hello".
- [ ] AC-3: `pipe(NULL)` returns `-EFAULT`.
- [ ] AC-4: `RLIMIT_NOFILE` cur = current_count+1: pipe(2) returns `-EMFILE` and no fd leaked.
- [ ] AC-5: Close write-end, then read: returns 0 (EOF).
- [ ] AC-6: Close read-end, then write: writer receives SIGPIPE; if blocked returns `-EPIPE`.
- [ ] AC-7: PIPE_BUF-sized write is atomic against concurrent writes from another writer.
- [ ] AC-8: Returned fds have FD_CLOEXEC clear.
- [ ] AC-9: After `fcntl(F_SETPIPE_SZ, 1MiB)` ring grows; subsequent capacity ≥ 1MiB or `-EPERM`.
- [ ] AC-10: System file-max reached: returns `-ENFILE`.

## Architecture

```rust
#[syscall(nr = 22, abi = "sysv")]
pub fn sys_pipe(pipefd: UserPtr<[i32; 2]>) -> isize {
    Pipe::do_pipe_flags(pipefd, 0)
}
```

`Pipe::do_pipe_flags(uptr, flags) -> isize`:
1. Pipe::validate_flags(flags)?;                            // EINVAL on unknown bits
2. let (read_file, write_file) = Pipe::create_pipe_files(flags)?;   // ENOMEM
3. let fd_read = Files::get_unused_fd(flags & O_CLOEXEC)?;          // EMFILE/ENFILE
4. let fd_write = Files::get_unused_fd(flags & O_CLOEXEC).map_err(|e| {
5.     Files::put_unused_fd(fd_read); e
6. })?;
7. unsafe { uptr.copy_out(&[fd_read, fd_write]).map_err(|e| {
8.     Files::put_unused_fd(fd_read); Files::put_unused_fd(fd_write); e
9. })?; }
10. Files::install(fd_read, read_file);
11. Files::install(fd_write, write_file);
12. Ok(0)

`Pipe::create_pipe_files(flags) -> Result<(File, File)>`:
1. let inode = PipeFs::alloc_anon_inode()?;
2. let info = PipeInodeInfo::alloc(PIPE_DEF_BUFFERS)?;     // ENOMEM
3. inode.set_pipe_info(info);
4. let read_file = File::open_anon(&inode, O_RDONLY | flags, &PIPE_FOPS_R)?;
5. let write_file = File::open_anon(&inode, O_WRONLY | flags, &PIPE_FOPS_W)?;
6. Ok((read_file, write_file))

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `two_fds_or_none` | INVARIANT | partial-failure rollback releases first fd. |
| `userptr_copy_atomic` | INVARIANT | copy_to_user happens only after both fds allocated. |
| `pipefs_inode_anon` | INVARIANT | inode never enters dcache. |
| `ring_capacity_within_cap` | INVARIANT | bufs.len() ≤ pipe-max-size. |
| `no_caps_required` | INVARIANT | call succeeds for any uid given resource budget. |

### Layer 2: TLA+

`fs/pipe-syscall.tla`:
- States: per-validate, per-alloc-inode, per-alloc-info, per-fd-1, per-fd-2, per-copy-out, per-install.
- Properties:
  - `safety_no_fd_leak_on_efault` — copy_to_user fault releases both fds.
  - `safety_no_fd_leak_on_emfile` — second fd alloc fail releases first.
  - `safety_atomic_install` — fds visible to /proc only after copy_out succeeds.
  - `liveness_pipe_terminates` — every pipe() returns or faults.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pipe_flags` post: ret==0 ⟹ both fds installed & user buffer populated | `Pipe::do_pipe_flags` |
| `create_pipe_files` post: shared inode refcount == 2 | `Pipe::create_pipe_files` |
| `pipe_inode_info` invariant: head ≤ tail + max_usage | `PipeInodeInfo` |

### Layer 4: Verus / Creusot functional

Per-pipe(2) POSIX-2017 semantic equivalence; LTP `pipe01..pipe11` selftests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pipe(2)` reinforcement:

- **Per-RLIMIT_NOFILE strict** — defense against per-fd-exhaustion DoS.
- **Per-RLIMIT_PIPESIZE / pipe-user-pages-hard** — defense against per-uid pipe-page hoarding.
- **Per-fd-install-after-copy-out** — defense against per-/proc/<pid>/fd race exposure.
- **Per-anon-inode (no dcache)** — defense against per-name-collision / per-inode-enum.
- **Per-PIPE_BUF atomicity guarantee** — defense against per-interleaved-write tearing.
- **Per-SIGPIPE on writer side** — defense against per-stuck-writer livelock.
- **Per-rollback on partial failure** — defense against per-fd-leak.

## Grsecurity / PaX surface

- **PaX UDEREF on pipefd copy_to_user** — defense against per-userptr kernel-deref; SMAP forced; the two-int write is bounded and whitelisted.
- **GRKERNSEC_FORKBOMB cooperates with pipe-per-user cap** — pipe creation is the cheapest way to amplify a fork-bomb; grsec caps both fork rate and per-uid pipe-pages.
- **RLIMIT_NOFILE enforced before any allocation** — defense against per-OOM via pipe-flood (alloc inode, then fail on fd would still pin memory transiently).
- **F_SETPIPE_SZ hard-cap = `/proc/sys/fs/pipe-max-size`** — grsec lowers default cap to 64KiB unless CAP_SYS_RESOURCE; defense against per-user megabyte ring inflation.
- **PAX_REFCOUNT on pipe_inode_info.refcount** — defense against per-refcount-wrap UAF when fd-passing across SCM_RIGHTS.
- **GRKERNSEC_CHROOT_FCHDIR coordinated** — pipefs anonymous inode is unreachable via /proc/self/fd traversal escape attempts inside chroot.
- **PAX_USERCOPY_HARDEN on pipe_read / pipe_write** — bounded copy whitelisted slab for `pipe_buffer.ops`.
- **Pipefs no-suid, no-exec, no-dev** — inode mount flags hard-coded; cannot host setuid bit even via debugfs.
- **GRKERNSEC_HIDESYM on pipe_fops** — symbol not exposed via `/proc/kallsyms` to unprivileged.
- **Per-namespace pipe-pages accounting (init_user_ns)** — defense against per-userns pipe-pages bypass; sub-userns inherit init cap.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- pipe2(2) flag handling (covered in Tier-5 `pipe2.md`).
- splice(2) / vmsplice(2) / tee(2) zero-copy paths (covered in Tier-5 splice docs).
- pipe_read / pipe_write fast-path (covered in Tier-3 `fs/pipe.md`).
- Implementation code.
