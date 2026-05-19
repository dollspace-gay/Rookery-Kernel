---
title: "Tier-5 syscall: flock(2) — syscall 73"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`flock(2)` applies or removes a 4.2BSD-style advisory lock on the open file referred to by `fd`. The lock is whole-file (no byte range), associated with the **open-file-description** (the `struct file *`), and held until the last reference to that description is closed. Unlike `fcntl(2)` POSIX locks, flock locks **do** survive fork — both parent and child share the same `filp` and thus the same lock — but do **not** survive `dup`-then-`close` of the original fd when a second open-file-description exists.

flock is the lock interface used by Debian package tools (`/var/lib/dpkg/lock`), GNU `lockfile-progs`, mail user agents (`mbox` locking), and most "shell-level" locking idioms (`flock -n /tmp/foo cmd`). Critical for: file-as-mutex idioms, init-script serialization, single-instance daemon checks, and atomic config-update locks.

### Acceptance Criteria

- [ ] AC-1: flock(fd, LOCK_EX) on freshly-open file returns 0.
- [ ] AC-2: Second process flock(LOCK_EX | LOCK_NB) on same path: returns -1, EWOULDBLOCK.
- [ ] AC-3: Second process flock(LOCK_EX) without NB: blocks until first unlocks.
- [ ] AC-4: flock(fd, LOCK_SH) ∧ another process flock(fd2, LOCK_SH): both succeed.
- [ ] AC-5: flock(fd, LOCK_UN) on never-locked fd: returns 0 (idempotent).
- [ ] AC-6: flock(fd, 0): returns -EINVAL.
- [ ] AC-7: flock(fd, LOCK_MAND): returns -EINVAL (since 5.15).
- [ ] AC-8: dup(fd) ⟹ same open-file-description ⟹ same lock; lock held until both fds closed.
- [ ] AC-9: fork() ⟹ child shares lock with parent (same filp).
- [ ] AC-10: Blocking acquire interrupted by SIGINT returns -EINTR; SA_RESTART ignored.
- [ ] AC-11: flock on a pipe fd: returns -EINVAL.
- [ ] AC-12: /proc/locks shows the lock with type FLOCK and ADVISORY kind.

### Architecture

```rust
#[syscall(nr = 73, abi = "sysv")]
pub fn sys_flock(fd: i32, op: i32) -> isize {
    let filp = Files::fget(fd).ok_or(EBADF)?;
    let ret = Flock::do_flock(&filp, op as u32);
    Files::fput(filp);
    ret
}
```

`Flock::do_flock(filp, op) -> isize`:
1. let cmd = op & ~LOCK_NB;
2. let nb = (op & LOCK_NB) != 0;
3. if cmd != LOCK_SH && cmd != LOCK_EX && cmd != LOCK_UN { return Err(EINVAL); }
4. if op & LOCK_MAND != 0 { return Err(EINVAL); }
5. /* Reject fds without inode (anon_inode, eventfd, etc.). */
6. if !filp.has_inode() { return Err(EINVAL); }
7. /* Build the file_lock. */
8. let mut fl = FileLock::new();
9. fl.fl_type    = match cmd { LOCK_SH => F_RDLCK, LOCK_EX => F_WRLCK, LOCK_UN => F_UNLCK, _ => unreachable!() };
10. fl.fl_flags  = FL_FLOCK | (if nb { 0 } else { FL_SLEEP });
11. fl.fl_owner  = filp as *const _ as usize;       // owner = open-file-description
12. fl.fl_pid    = current().tgid;
13. /* Delegate to per-fs op or generic. */
14. if let Some(op) = filp.f_op.flock {
15.     return op(filp, cmd, &mut fl);
16. }
17. Locks::flock_lock_file_wait(filp, &mut fl)
18. /* This routine handles SH/EX conflicts, returns EWOULDBLOCK if NB & blocked,
19.    EINTR if signal during blocking wait, ENOLCK if lock table full. */

`Locks::flock_lock_file_wait(filp, fl) -> isize`:
1. loop {
2.     match Locks::flock_lock_inode(filp.inode(), fl) {
3.         Ok(()) => return Ok(0),
4.         Err(EWOULDBLOCK) if (fl.fl_flags & FL_SLEEP) == 0 => return Err(EWOULDBLOCK),
5.         Err(EWOULDBLOCK) => Locks::wait_for_flock(filp, fl)?,   // returns EINTR on signal
6.         Err(e) => return Err(e),
7.     }
8. }

### Out of Scope

- fcntl(F_SETLK) POSIX locks (covered in Tier-5 `fcntl.md`).
- OFD locks via fcntl (covered in Tier-5 `fcntl.md`).
- NFS-lock-state-recovery (covered in Tier-3 `fs/nfs/locking.md`).
- FUSE flock routing (covered in Tier-3 `fs/fuse/locking.md`).
- Implementation code.

### signature

```c
int flock(int fd, int operation);
```

```c
#define LOCK_SH      1   /* shared lock */
#define LOCK_EX      2   /* exclusive lock */
#define LOCK_UN      8   /* unlock */
#define LOCK_NB      4   /* non-blocking (or-able with SH/EX) */
/* LOCK_MAND  = 32  mandatory locking — removed; -EINVAL */
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `int` | in | Open file descriptor. |
| `operation` | `int` | in | One of LOCK_SH / LOCK_EX / LOCK_UN, optionally or'd with LOCK_NB. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EBADF` | `fd` not open. |
| `EINTR` | Blocking acquire interrupted by signal. |
| `EINVAL` | `operation` not in {LOCK_SH, LOCK_EX, LOCK_UN} (optionally or'd with LOCK_NB); LOCK_MAND attempts; fd on file with no inode (anon pipe with no flock support). |
| `ENOLCK` | Kernel lock table full. |
| `EWOULDBLOCK` | LOCK_NB set and lock would block. |

### abi surface

```text
__NR_flock   (x86_64)  = 73
__NR_flock   (arm64)   = 32
__NR_flock   (riscv)   = 32
__NR_flock   (i386)    = 143

/* flock(2) is whole-file, advisory, and tied to struct file.       */
/* It is NOT the same lock-space as fcntl(F_SETLK) POSIX locks.      */
/* On NFSv2: flock is emulated on the client; NFSv3+: server-side.   */
```

### compatibility contract

REQ-1: Syscall number is **73** on x86_64. ABI-stable.

REQ-2: `operation` must be exactly one of LOCK_SH / LOCK_EX / LOCK_UN, optionally or'd with LOCK_NB. Any other bit set → `-EINVAL`. LOCK_MAND (mandatory locking) was removed in 5.15; attempts return `-EINVAL`.

REQ-3: `fd` resolved via `fget(fd)`; flock requires the file to have a backing inode that supports flock. Pipes, sockets, eventfd, signalfd, etc. return `-EINVAL`.

REQ-4: Lock ownership is per-open-file-description (`fl_owner = filp`). dup'd fds and forked children share the lock. Independent `open()` calls produce independent open-file-descriptions and thus independent locks (which may conflict).

REQ-5: LOCK_SH allows multiple holders simultaneously; LOCK_EX requires exclusive. Upgrades and downgrades:
- SH → EX: requested as a fresh LOCK_EX; succeeds only when no other SH holders exist (or LOCK_NB returns EWOULDBLOCK).
- EX → SH: requested as LOCK_SH; demoted atomically.
- → UN: drops the lock regardless of previous state.

REQ-6: LOCK_NB makes both acquire and downgrade non-blocking; if the operation would block, return `-EWOULDBLOCK` (== `-EAGAIN`).

REQ-7: Blocking acquire (no LOCK_NB) is interruptible by signal; returns `-EINTR`; SA_RESTART is **not** honored (BSD compat) — the syscall does NOT auto-restart.

REQ-8: Lock is released on:
- Explicit LOCK_UN.
- Last close of the same open-file-description (via `locks_remove_flock` in `__fput`).
- Process exit (via files-cleanup tear-down).

REQ-9: Per-filesystem `f_op->flock` may override the default. NFS mounts route to nfsv3-lockd or nfsv4 client lock state; FUSE routes to userspace.

REQ-10: Lock state is observable via `/proc/locks`, columns: lock-id, type (FLOCK), kind (ADVISORY), mode (READ/WRITE), pid, dev:ino, start, end.

REQ-11: Per-RLIMIT_LOCKS (if compiled): bounds per-user lock count.

REQ-12: flock(2) and fcntl(F_SETLK) are in **separate** lock-spaces on Linux. A flock LOCK_EX and a fcntl F_WRLCK on the same inode do not interact (they may both succeed simultaneously). Code mixing the two is a known footgun.

REQ-13: flock on a directory fd: kernel rejects with `-EINVAL` since 5.15? — actually allowed: flock on dirfd is permitted; advisory whole-fd lock.

REQ-14: flock(2) does not consume entropy and does not require any capability.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `op_table_total` | INVARIANT | only LOCK_SH/EX/UN ± LOCK_NB accepted. |
| `lock_mand_rejected` | INVARIANT | LOCK_MAND bit ⟹ EINVAL. |
| `fl_owner_is_filp` | INVARIANT | fl_owner = filp pointer (open-file-description). |
| `filp_must_have_inode` | INVARIANT | no-inode fd ⟹ EINVAL. |
| `nb_returns_ewouldblock` | INVARIANT | LOCK_NB set & blocked ⟹ EWOULDBLOCK. |
| `eintr_no_restart` | INVARIANT | signal during wait ⟹ EINTR, no auto-restart. |

### Layer 2: TLA+

`fs/flock-syscall.tla`:
- States: per-fget, per-validate-op, per-build-fl, per-acquire, per-wait, per-release.
- Properties:
  - `safety_sh_ex_exclusion` — no concurrent EX and any other lock; multiple SH allowed.
  - `safety_owner_is_filp` — fl_owner stable across fork / dup.
  - `safety_release_on_last_filp_close` — last fput releases lock.
  - `liveness_blocking_acquire_terminates` — either gets lock or EINTR.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_flock` post: ret == 0 ⟹ lock recorded in inode.flctx | `Flock::do_flock` |
| `flock_lock_file_wait` post: blocking caller acquires or returns EINTR | `Locks::flock_lock_file_wait` |
| `locks_remove_flock` post: on last fput, lock removed | `Locks::locks_remove_flock` |

### Layer 4: Verus / Creusot functional

Per-flock(2) BSD-4.2 + Linux semantics; LTP `flock01..flock06`, util-linux `flock(1)` selftests pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`flock(2)` reinforcement:

- **Per-op-mask strict (LOCK_SH/EX/UN ± LOCK_NB)** — defense against per-bit-smuggling.
- **Per-LOCK_MAND rejected** — defense against per-deprecated-mandatory-lock abuse.
- **Per-fl_owner = filp** — defense against per-fork-lock-confusion (correctness invariant).
- **Per-RLIMIT_LOCKS** — defense against per-lock-table-exhaustion DoS.
- **Per-fget/fput balanced** — defense against per-filp-leak.
- **Per-locks_remove_flock at last fput** — defense against per-zombie-lock.
- **Per-EINTR no SA_RESTART** — defense against per-signal-loss across rearm (BSD-correct).

### grsecurity / pax surface

- **PaX UDEREF on file_lock copy paths** — although flock takes only ints, the internal file_lock struct copy-out via /proc/locks is SMAP-guarded.
- **GRKERNSEC_CHROOT_FLOCK** — defense against per-flock-table-pollution from chroot: chrooted callers may only flock files within their root; flock attempts on dirfd above root return -EACCES.
- **RLIMIT_LOCKS strict** — grsec lowers default user lock cap to 1024 for non-CAP_SYS_RESOURCE; defense against per-lock-table-blow DoS.
- **PAX_REFCOUNT on filp refcount during fget** — defense against per-fput-double UAF leaking lock.
- **GRKERNSEC_PROC_LOCKS_READABLE** — `/proc/locks` filtered per-process namespace and uid; non-root sees only own.
- **LOCK_MAND removal enforced (= EINVAL)** — defense against per-deprecated-mandatory-lock cohort.
- **PAX_USERCOPY_HARDEN on /proc/locks read path** — bounded usercopy whitelisted slab.
- **GRKERNSEC_HIDESYM on flock_lock_file_wait** — symbol not in kallsyms.
- **flock on bind-mounted /proc denied** — defense against per-procfs-flock-DoS (procfs inodes ignore flock; grsec audits the attempt).
- **Flock + fcntl-lock interaction audited** — grsec logs at GRKERNSEC_AUDIT_KERN when a process applies both flock and fcntl POSIX locks to the same inode (footgun detector).

