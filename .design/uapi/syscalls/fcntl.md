# Tier-5 syscall: fcntl(2) — syscall 72

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/fcntl.c (SYSCALL_DEFINE3(fcntl), do_fcntl, fcntl_setlk, fcntl_getlk, fcntl_dirnotify, fcntl_rw_hint)
  - fs/locks.c (POSIX/OFD lock handling)
  - mm/memfd.c (F_ADD_SEALS, F_GET_SEALS)
  - fs/pipe.c (F_GETPIPE_SZ, F_SETPIPE_SZ)
  - include/uapi/linux/fcntl.h (F_DUPFD, F_GETFD/SETFD, F_GETFL/SETFL, F_GETLK/SETLK/SETLKW, F_OFD_*, F_SET_RW_HINT, F_ADD_SEALS, F_GET_SEALS, ...)
  - arch/x86/entry/syscalls/syscall_64.tbl (72  common  fcntl)
-->

## Summary

`fcntl(2)` is the multiplexed per-fd control syscall. A single integer `op` selects one of ~30 operations, ranging from "dup a fd" through "atomic POSIX advisory locks" to memfd-sealing and per-file write-hint hints for tiered storage. The third argument is an op-specific scalar or pointer, validated by op.

Like `bpf(2)` and `ioctl(2)`, `fcntl(2)` is a "subsystem entry point" — its single syscall number hides a sprawling op-space governed by per-op contracts, capability gates, and lock-table interactions. Critical for: shell redirection (`dup2`-style fd shuffling via F_DUPFD), database engines (POSIX/OFD locks), tmpfs / memfd_create users (seals), and io_uring file-hint workloads.

## Signature

```c
int fcntl(int fd, int op, ... /* arg */);
```

Selected ops (full table in fs/fcntl.c::do_fcntl):

```c
#define F_DUPFD          0   /* dup fd >= arg                                 */
#define F_DUPFD_CLOEXEC  1030 /* same + O_CLOEXEC                              */
#define F_GETFD          1   /* read FD_CLOEXEC                                */
#define F_SETFD          2   /* write FD_CLOEXEC                               */
#define F_GETFL          3   /* read file status flags (O_APPEND/NONBLOCK/…)   */
#define F_SETFL          4   /* write file status flags                        */
#define F_GETLK          5   /* test POSIX lock                                */
#define F_SETLK          6   /* set non-blocking POSIX lock                    */
#define F_SETLKW         7   /* set blocking POSIX lock                        */
#define F_SETOWN         8   /* set SIGIO recipient                            */
#define F_GETOWN         9   /* read SIGIO recipient                           */
#define F_SETSIG         10  /* override SIGIO with given signal               */
#define F_GETSIG         11  /* read F_SETSIG                                  */
#define F_GETOWN_EX      16
#define F_SETOWN_EX      15
#define F_OFD_GETLK      36
#define F_OFD_SETLK      37
#define F_OFD_SETLKW     38
#define F_SETPIPE_SZ     1031
#define F_GETPIPE_SZ     1032
#define F_ADD_SEALS      1033
#define F_GET_SEALS      1034
#define F_GET_RW_HINT    1035
#define F_SET_RW_HINT    1036
#define F_GET_FILE_RW_HINT 1037
#define F_SET_FILE_RW_HINT 1038
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd`  | `int` | in | Target file descriptor. |
| `op`  | `int` | in | Operation code. |
| `arg` | varies | in/out | Op-specific; integer for most, `struct flock *` for lock ops, `struct f_owner_ex *` for SETOWN_EX. |

## Return value

| Op class | Success | Failure |
|---|---|---|
| F_DUPFD / F_DUPFD_CLOEXEC | new fd | `-1` + errno |
| F_GETFD / F_GETFL / F_GETSIG / F_GETLEASE / F_GETPIPE_SZ / F_GET_SEALS / F_GET_RW_HINT | value | `-1` + errno |
| F_SETFD / F_SETFL / F_SETOWN / F_SETSIG / F_SETLK / F_SETLKW / F_SETLEASE / F_ADD_SEALS / F_SET_RW_HINT / F_SETPIPE_SZ | `0` | `-1` + errno |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not open. |
| `EINVAL` | Unknown `op`; bad flag bits; F_DUPFD `arg >= RLIMIT_NOFILE`; F_SETSIG signal > _NSIG; F_SETPIPE_SZ size > pipe-max-size; F_ADD_SEALS on non-memfd. |
| `EPERM` | F_SETOWN target outside permission; F_SETPIPE_SZ above soft cap without CAP_SYS_RESOURCE; F_ADD_SEALS adding to a file lacking F_SEAL_SEAL prereq. |
| `EAGAIN` / `EACCES` | F_SETLK conflict (POSIX semantics). |
| `EINTR` | F_SETLKW interrupted by signal. |
| `EDEADLK` | F_SETLKW would deadlock. |
| `EMFILE` | F_DUPFD: no free fd >= arg. |
| `ENOLCK` | F_SETLK / F_SETLKW: kernel lock table full. |
| `EFAULT` | `arg` user pointer faults. |
| `ENODEV` / `ESPIPE` | F_GETPIPE_SZ / F_SETPIPE_SZ on non-pipe fd. |

## ABI surface

```text
__NR_fcntl  (x86_64)  = 72
__NR_fcntl  (arm64)   = 25
__NR_fcntl  (riscv)   = 25
__NR_fcntl  (i386)    = 55   (32-bit struct flock; fcntl64 = 221 for off_t-64-bit)

/* x86_64 has no separate fcntl64 — kernel struct flock is always 64-bit off_t. */
```

## Compatibility contract

REQ-1: Syscall number is **72** on x86_64. ABI-stable.

REQ-2: `fd` is resolved via `fget(fd)`; returns `-EBADF` if not open. The file ref is dropped before return.

REQ-3: Op dispatch table in `do_fcntl(fd, op, arg, filp)`:
- F_DUPFD / F_DUPFD_CLOEXEC → `f_dupfd(arg, filp, flags)`; uses `get_unused_fd_flags(arg, flags)`.
- F_GETFD / F_SETFD → manipulate `current.files.fdt[fd].flags & FD_CLOEXEC`.
- F_GETFL → return `filp.f_flags & (O_APPEND|O_NONBLOCK|O_DIRECT|O_ASYNC|O_*)`.
- F_SETFL → mask to settable subset (only O_APPEND, O_NONBLOCK, O_DIRECT, O_ASYNC, O_NOATIME), update `filp.f_flags`.
- F_GETLK / F_SETLK / F_SETLKW → POSIX advisory locks via `fcntl_getlk` / `fcntl_setlk`; tied to (file, fl_owner = current.files).
- F_OFD_GETLK / F_OFD_SETLK / F_OFD_SETLKW → Open-File-Description locks, fl_owner = filp (per-open, not per-process).
- F_SETOWN[_EX] / F_GETOWN[_EX] → install async-IO recipient (process or pgrp) for SIGIO.
- F_SETSIG / F_GETSIG → override SIGIO signal number.
- F_SETPIPE_SZ / F_GETPIPE_SZ → pipe ring resize (pipefs-only).
- F_ADD_SEALS / F_GET_SEALS → memfd sealing (tmpfs/shmem inode-only).
- F_SET_RW_HINT / F_GET_RW_HINT → per-file or per-fd write-lifetime hint (RWH_WRITE_LIFE_*).
- F_GETLEASE / F_SETLEASE → file lease for cluster fs; CAP_LEASE for non-owner.

REQ-4: F_SETLK / F_SETLKW + struct flock: `l_type ∈ {F_RDLCK, F_WRLCK, F_UNLCK}`, `l_whence ∈ {SEEK_SET, SEEK_CUR, SEEK_END}`, `l_start`/`l_len` validated to non-negative resolved range.

REQ-5: F_OFD_* locks: `fl.l_pid` must be 0 on input. Lock is owned by the open-file-description (the `struct file *`), so dup'd fds share the lock and forks do NOT inherit it as a separate ownership.

REQ-6: F_DUPFD: minimum fd = `arg`. Searches the fdtable upward; `-EMFILE` if none < `RLIMIT_NOFILE`. F_DUPFD_CLOEXEC additionally sets FD_CLOEXEC on the new fd.

REQ-7: F_SETFL: filters arg to ONLY the modifiable bits; other O_* bits silently dropped. O_APPEND on already-O_APPEND filesystem (e.g. NFS with locking semantics) may return `-EPERM`.

REQ-8: F_SETPIPE_SZ: arg rounded up to page-sized buffer count; capped at `/proc/sys/fs/pipe-max-size`. Above the soft cap, requires CAP_SYS_RESOURCE.

REQ-9: F_ADD_SEALS: only valid on `memfd_create`-d files (anon_inode with shmem_file_operations); requires that the file does not already carry F_SEAL_SEAL (else `-EPERM`). Adds bits to inode.i_seals; mmap/write attempts now check seals.

REQ-10: F_SETSIG: signal number must be 0 (default SIGIO) or in `[1, _NSIG]`; SIGKILL/SIGSTOP rejected with `-EINVAL`.

REQ-11: F_SETOWN: target pid/pgid validated; cannot SETOWN to a process the caller cannot send signals to (`kill_ok_by_cred` check); `-EPERM` otherwise.

REQ-12: F_GETLEASE / F_SETLEASE: requires CAP_LEASE if the caller is not the file's owner. Lease break delivers SIGIO; lease-break-time = `/proc/sys/fs/lease-break-time`.

REQ-13: F_SET_RW_HINT: arg ∈ `{RWH_WRITE_LIFE_NOT_SET=0, _NONE=1, _SHORT=2, _MEDIUM=3, _LONG=4, _EXTREME=5}`; stored on either inode (F_SET_FILE_RW_HINT) or struct file (F_SET_RW_HINT); used by block layer to tier writes on supporting devices.

REQ-14: F_DUPFD return fd has FD_CLOEXEC = 0 (unless F_DUPFD_CLOEXEC); FL bits of the underlying file are unchanged (shared filp).

## Acceptance Criteria

- [ ] AC-1: F_DUPFD on open fd 3 with arg=10 returns ≥ 10.
- [ ] AC-2: F_DUPFD_CLOEXEC: new fd has FD_CLOEXEC set.
- [ ] AC-3: F_SETFL with O_NONBLOCK + O_EXEC: O_EXEC silently dropped, O_NONBLOCK applied.
- [ ] AC-4: F_SETLK conflict with another process: returns -EAGAIN.
- [ ] AC-5: F_SETLKW: blocks until lock available; interrupted by signal returns -EINTR.
- [ ] AC-6: F_OFD_SETLK: same lock seen on dup'd fd; not inherited by fork's separate process.
- [ ] AC-7: F_SETPIPE_SZ above pipe-max-size returns -EPERM without CAP_SYS_RESOURCE.
- [ ] AC-8: F_ADD_SEALS on a regular file fd returns -EINVAL.
- [ ] AC-9: F_ADD_SEALS adding F_SEAL_WRITE on a memfd with prior writers: -EBUSY.
- [ ] AC-10: F_SETSIG with SIGKILL returns -EINVAL.
- [ ] AC-11: F_SETOWN to a forbidden pid returns -EPERM.
- [ ] AC-12: F_GETFL returns flags reflecting prior F_SETFL.
- [ ] AC-13: Unknown op returns -EINVAL.

## Architecture

```rust
#[syscall(nr = 72, abi = "sysv")]
pub fn sys_fcntl(fd: i32, op: i32, arg: usize) -> isize {
    let filp = Files::fget(fd).ok_or(EBADF)?;
    let ret = Fcntl::do_fcntl(fd, op, arg, &filp);
    Files::fput(filp);
    ret
}
```

`Fcntl::do_fcntl(fd, op, arg, filp) -> isize`:
1. match FcntlOp::try_from(op).map_err(|_| EINVAL)? {
2.     F_DUPFD              => Files::dup_fd_min(filp, arg as u32, 0),
3.     F_DUPFD_CLOEXEC      => Files::dup_fd_min(filp, arg as u32, FD_CLOEXEC),
4.     F_GETFD              => Files::get_cloexec(fd),
5.     F_SETFD              => Files::set_cloexec(fd, arg as u32),
6.     F_GETFL              => Ok(filp.f_flags & FL_SETTABLE_MASK | (filp.f_flags & FL_RO_MASK)),
7.     F_SETFL              => Fcntl::set_fl(filp, arg as u32),
8.     F_GETLK              => Locks::fcntl_getlk(filp, false, arg.into()),
9.     F_SETLK              => Locks::fcntl_setlk(filp, false, false, arg.into()),
10.    F_SETLKW             => Locks::fcntl_setlk(filp, false, true,  arg.into()),
11.    F_OFD_GETLK          => Locks::fcntl_getlk(filp, true,  arg.into()),
12.    F_OFD_SETLK          => Locks::fcntl_setlk(filp, true,  false, arg.into()),
13.    F_OFD_SETLKW         => Locks::fcntl_setlk(filp, true,  true,  arg.into()),
14.    F_SETOWN             => Fown::set_owner(filp, arg as i32),
15.    F_GETOWN             => Fown::get_owner(filp),
16.    F_SETSIG             => Fown::set_sig(filp, arg as i32),
17.    F_GETSIG             => Fown::get_sig(filp),
18.    F_GETPIPE_SZ         => Pipe::get_size(filp),
19.    F_SETPIPE_SZ         => Pipe::set_size(filp, arg as u32),
20.    F_ADD_SEALS          => MemFd::add_seals(filp, arg as u32),
21.    F_GET_SEALS          => MemFd::get_seals(filp),
22.    F_SET_RW_HINT        => Fcntl::set_rw_hint(filp, false, arg as u64),
23.    F_GET_RW_HINT        => Fcntl::get_rw_hint(filp, false),
24.    F_SET_FILE_RW_HINT   => Fcntl::set_rw_hint(filp, true,  arg as u64),
25.    F_GET_FILE_RW_HINT   => Fcntl::get_rw_hint(filp, true),
26.    F_GETLEASE           => Lease::get(filp),
27.    F_SETLEASE           => Lease::set(filp, arg as u32),
28.    _                    => Err(EINVAL),
29. }

`Fcntl::set_fl(filp, new_flags) -> isize`:
1. let masked = new_flags & FL_SETTABLE_MASK;        // O_APPEND|O_NONBLOCK|O_DIRECT|O_ASYNC|O_NOATIME
2. filp.f_op.check_flags(masked)?;                   // EPERM for some FS
3. filp.f_flags = (filp.f_flags & !FL_SETTABLE_MASK) | masked;
4. Ok(0)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `op_table_total` | INVARIANT | every FcntlOp has exactly one arm. |
| `unknown_op_einval` | INVARIANT | op not in table ⟹ EINVAL. |
| `fput_balanced` | INVARIANT | sys_fcntl: fget'd ref dropped before return. |
| `setfl_mask_only` | INVARIANT | only FL_SETTABLE_MASK bits updated. |
| `setlk_owner_correct` | INVARIANT | non-OFD ⟹ fl_owner = current.files; OFD ⟹ fl_owner = filp. |
| `add_seals_memfd_only` | INVARIANT | F_ADD_SEALS rejects non-shmem inodes. |
| `setpipe_sz_capped` | INVARIANT | size ≤ pipe-max-size unless CAP_SYS_RESOURCE. |

### Layer 2: TLA+

`fs/fcntl-syscall.tla`:
- States: per-fget, per-dispatch, per-op-execute, per-fput.
- Properties:
  - `safety_no_filp_leak` — fget'd ref always fput.
  - `safety_setlk_owner_consistent` — OFD lock ownership invariant.
  - `safety_setpipe_cap` — cap honored.
  - `safety_seals_monotone` — seals never removed.
  - `liveness_setlkw_terminates` — blocking lock returns or EINTR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fcntl` post: ret matches per-op result table | `Fcntl::do_fcntl` |
| `set_fl` post: filp.f_flags & FL_SETTABLE_MASK == masked | `Fcntl::set_fl` |
| `add_seals` post: i_seals new bits subset of caller's request | `MemFd::add_seals` |
| `fcntl_setlk` post: fl_owner correct for POSIX vs OFD | `Locks::fcntl_setlk` |

### Layer 4: Verus / Creusot functional

Per-fcntl(2) POSIX-2017 + Linux extensions semantic equivalence; LTP `fcntl01..fcntl37`, glibc `tst-fcntl`, memfd self-tests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fcntl(2)` reinforcement:

- **Per-op dispatch total** — defense against per-unknown-op fall-through.
- **Per-fget/fput balanced** — defense against per-filp leak.
- **Per-F_SETFL bit-mask** — defense against per-flag-smuggling (O_TRUSTED-style bits).
- **Per-F_SETPIPE_SZ cap** — defense against per-pipe-ring inflation.
- **Per-F_ADD_SEALS monotone** — defense against per-seal-removal exploit.
- **Per-F_SETOWN credential-check** — defense against per-cross-uid SIGIO injection.
- **Per-F_SETSIG sigkill/stop reject** — defense against per-immortal-process via SIGKILL routing.
- **Per-OFD-lock owner = filp** — defense against per-fork-lock-inheritance ambiguity.
- **Per-RLIMIT_NOFILE for F_DUPFD** — defense against per-fd-amplification.

## Grsecurity / PaX surface

- **PaX UDEREF on struct flock / f_owner_ex copy_from_user / copy_to_user** — defense against per-userptr kernel-deref; SMAP forced.
- **GRKERNSEC_FORKBOMB cooperates with F_DUPFD RLIMIT_NOFILE** — F_DUPFD is the cheap path to fd-amplification; strict RLIMIT_NOFILE enforcement.
- **F_SETPIPE_SZ hard-cap = /proc/sys/fs/pipe-max-size** — grsec lowers default cap to 64KiB unless CAP_SYS_RESOURCE.
- **PAX_REFCOUNT on filp refcount during fget** — defense against per-fput-double UAF.
- **F_ADD_SEALS audit** — grsec logs every F_ADD_SEALS at GRKERNSEC_AUDIT_KERN; useful for detecting attempted seal-bypass.
- **F_SETOWN cross-uid SIGIO routing rejected** — defense against per-cross-uid signal injection via SIGIO indirection.
- **F_SETSIG SIGKILL / SIGSTOP rejected** — defense against per-immortal-task via routed SIGKILL.
- **F_SETLEASE CAP_LEASE strict** — defense against per-non-owner lease takeover.
- **GRKERNSEC_HIDESYM on file_operations vtables** — fcntl's per-fs check_flags vtables not in kallsyms.
- **PAX_USERCOPY_HARDEN on flock copy** — bounded by sizeof(struct flock), whitelisted slab.
- **F_OFD_* under audit when used inside chroot** — defense against per-lock-table-pollution by chrooted process.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-fs check_flags vtables (covered in per-fs Tier-3 docs).
- POSIX/OFD lock manager internals (covered in Tier-3 `fs/locks.md`).
- memfd sealing internals (covered in Tier-3 `mm/memfd.md`).
- ioctl(2) (separate syscall).
- Implementation code.
