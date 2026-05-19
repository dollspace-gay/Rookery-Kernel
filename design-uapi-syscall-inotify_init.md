---
title: "Tier-5 syscall: inotify_init(2) — syscall 253"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`inotify_init(2)` is the **flagless legacy variant** of `inotify_init1(2)`: it allocates a kernel-resident inotify instance (an `fsnotify_group` plus an event queue) and returns a file descriptor on it. Behaviorally it is equivalent to `inotify_init1(0)` — no `IN_NONBLOCK`, no `IN_CLOEXEC` — meaning the returned fd is **blocking** on `read(2)` and is **inherited across `execve(2)`**. The latter property is a long-standing fd-leak footgun: a process that opens an inotify instance and then `exec`s without explicitly closing the fd hands the inotify instance to the new image. `inotify_init1(2)` was introduced (along with the `IN_CLOEXEC` flag conventions for `socket2`, `pipe2`, `dup3`, `epoll_create1`, `signalfd`, `timerfd`, `eventfd`, etc.) in 2.6.27 precisely to close this loophole. The legacy syscall is preserved for ABI compatibility. Critical for: pre-2.6.27 applications, archaic build systems, distros maintaining backward compatibility, statically-linked binaries built against pre-`inotify_init1` headers.

This Tier-5 covers the userspace ABI of syscall 253.

### Acceptance Criteria

- [ ] AC-1: `inotify_init()` returns a non-negative fd; `fstat(fd)` reports `S_IFREG`-like with `st_mode & S_IFMT` matching the inotify magic device.
- [ ] AC-2: `read(fd, ...)` on a freshly created inotify instance with no watches **blocks** indefinitely (no IN_NONBLOCK).
- [ ] AC-3: After `inotify_init()` + `execve`: the new image inherits the fd (no IN_CLOEXEC).
- [ ] AC-4: `fcntl(fd, F_GETFD)` returns `0` (no FD_CLOEXEC).
- [ ] AC-5: `fcntl(fd, F_GETFL) & O_NONBLOCK == 0`.
- [ ] AC-6: After 128 `inotify_init()` calls from a single uid: 129th call returns `-EMFILE`.
- [ ] AC-7: Exhausting `RLIMIT_NOFILE`: returns `-EMFILE`.
- [ ] AC-8: `inotify_init()` followed by `inotify_add_watch(fd, "/tmp", IN_CREATE)`: returns a valid watch descriptor.
- [ ] AC-9: `inotify_init()` followed by `close(fd)`: per-user instance count decremented.
- [ ] AC-10: `inotify_init()` followed by `fork`: both parent and child can `read(fd)` and see events from the same queue.

### Architecture

Rookery surface in `fs/notify/inotify/syscalls.rs`:

```rust
pub fn sys_inotify_init() -> isize {
    /* Delegate to the flagged variant with flags = 0 */
    Inotify::do_init(0 /* flags */)
}
```

`Inotify::do_init(flags)` is the shared implementation between `inotify_init` and `inotify_init1`:

```rust
pub fn do_init(flags: i32) -> isize {
    /* Validate flags (only IN_NONBLOCK, IN_CLOEXEC accepted; 0 always OK) */
    if (flags & !(IN_NONBLOCK | IN_CLOEXEC)) != 0 {
        return -EINVAL as isize;
    }
    /* Per-user instance cap */
    let uid = current().real_uid();
    if UserCounters::inotify_instances(uid).inc_if_below(sysctl::INOTIFY_MAX_USER_INSTANCES.load()) == false {
        return -EMFILE as isize;
    }
    /* Allocate fsnotify_group */
    let grp = match FsnotifyGroup::alloc_inotify() {
        Ok(g)  => g,
        Err(_) => {
            UserCounters::inotify_instances(uid).dec();
            return -ENOMEM as isize;
        }
    };
    /* Anon file */
    let file = match Anon::create("inotify", &INOTIFY_FOPS, grp.clone(), o_flags_from(flags)) {
        Ok(f)  => f,
        Err(e) => {
            UserCounters::inotify_instances(uid).dec();
            return -(e as isize);
        }
    };
    /* fd-table install */
    let fd_flags = if flags & IN_CLOEXEC != 0 { FD_CLOEXEC } else { 0 };
    match FdTable::install(file, fd_flags) {
        Ok(fd) => fd as isize,
        Err(e) => -(e as isize),
    }
}
```

Note that `inotify_init` always calls with `flags == 0`, which means:
- `o_flags_from(0)` = `0` (no `O_NONBLOCK`).
- `fd_flags = 0` (no `FD_CLOEXEC`).

### Out of Scope

- `fs/notify/inotify/00-overview.md` Tier-3: inotify subsystem internals.
- `fs/notify/fsnotify.md` Tier-3: fsnotify framework.
- `inotify_init1.md` modern sibling.
- `inotify_add_watch.md` / `inotify_rm_watch.md` siblings.
- `dup3.md` / `pipe2.md` / `epoll_create1.md` / `signalfd.md` / `eventfd2.md` (CLOEXEC family).
- glibc / musl userspace wrappers.
- Implementation code.

### signature

```c
int inotify_init(void);
```

Rust ABI shim:

```rust
pub fn sys_inotify_init() -> isize;
```

Syscall number: **253** (x86_64). Generic syscall table: not present on arm64 / generic (those use `inotify_init1` exclusively).

### parameters

(none)

### return

- **Success**: non-negative file descriptor on a new inotify instance.
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

### errors

| errno | Trigger |
|---|---|
| `EMFILE` | per-process fd table full (`RLIMIT_NOFILE`), or per-user inotify-instance limit hit (`fs.inotify.max_user_instances`, default 128). |
| `ENFILE` | system-wide fd table full. |
| `ENOMEM` | kernel cannot allocate `fsnotify_group`. |

### abi surface

```text
__NR_inotify_init (x86_64) = 253
__NR_inotify_init (i386)   = 291
__NR_inotify_init (arm64 / generic) = unavailable (use inotify_init1)

/* Equivalence */
inotify_init() === inotify_init1(0);

/* Returned fd properties */
- O_NONBLOCK: NOT set; read(2) blocks until events available
- FD_CLOEXEC: NOT set; fd inherited across execve(2)
- File-table backing: fsnotify_group + event_queue
- Per-user instance count incremented; checked against fs.inotify.max_user_instances

/* Sysctls */
fs.inotify.max_user_instances = 128       (default)
fs.inotify.max_queued_events  = 16384     (default; per-instance queue cap)
fs.inotify.max_user_watches   = 65536     (default; per-user, sum across instances)
```

### compatibility contract

REQ-1: Syscall number is **253** on x86_64; **291** on i386; not present on arm64/generic.

REQ-2: Equivalent to `inotify_init1(0)`: no flags applied; returned fd is blocking and inherits across exec.

REQ-3: Per-user `fs.inotify.max_user_instances` (default 128) is decremented; exceeding the cap returns `-EMFILE`.

REQ-4: Per-process `RLIMIT_NOFILE` is decremented; exceeding returns `-EMFILE`.

REQ-5: System-wide fd table exhaustion returns `-ENFILE`.

REQ-6: Kernel-side allocation failure of `fsnotify_group` returns `-ENOMEM`.

REQ-7: Returned fd is readable via `read(2)` for `struct inotify_event` records; poll-able via `POLLIN`; closable via `close(2)`.

REQ-8: Returned fd is the substrate for `inotify_add_watch(2)` and `inotify_rm_watch(2)`; both syscalls operate on this fd.

REQ-9: Inheritance:
- `fork(2)`: child inherits fd (refcounted); both parent and child can read events.
- `execve(2)`: fd inherited unless caller explicitly sets `FD_CLOEXEC` via `fcntl(fd, F_SETFD, FD_CLOEXEC)` before exec. (This is the footgun `inotify_init1(IN_CLOEXEC)` was designed to close.)
- `dup(2)` / `dup2(2)` / `dup3(2)`: standard fd dup semantics.

REQ-10: No capability check; any unprivileged user can call.

REQ-11: Per-user instance count is **per-real-uid**, not per-user-ns; tracked in `inotify_init_user(real_uid)`.

REQ-12: Per-instance event queue capped at `fs.inotify.max_queued_events`; overflow events are silently dropped after marking the queue with `IN_Q_OVERFLOW`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flagless_blocking_fd` | INVARIANT | returned fd has `O_NONBLOCK == 0`. |
| `flagless_not_cloexec` | INVARIANT | returned fd has `FD_CLOEXEC == 0`. |
| `per_user_cap_enforced` | INVARIANT | exceeding `fs.inotify.max_user_instances` ⟹ `-EMFILE` AND instance counter not incremented. |
| `rollback_on_error` | INVARIANT | partial failure (anon-file create, fd install) ⟹ user counter and group resources released. |
| `equivalence_with_init1_0` | INVARIANT | `sys_inotify_init()` ≡ `sys_inotify_init1(0)`. |

### Layer 2: TLA+

`uapi/inotify-init.tla`:
- Variables: `user_instance_count[uid]`, `fd_table`, `inotify_groups`.
- Properties:
  - `safety_flag_equivalence` — `inotify_init()` always produces same fd-properties as `inotify_init1(0)`.
  - `safety_cap_enforced` — `user_instance_count[uid] ≤ max_user_instances` post.
  - `safety_rollback_on_error` — error path leaves no leaked group / counter.
  - `liveness_eventually_terminates` — call terminates in bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_inotify_init` post: returned fd has flags `O_NONBLOCK == 0, FD_CLOEXEC == 0` | `sys_inotify_init` |
| `Inotify::do_init(0)` post: equivalent to do_init(0) for flagged variant | shared impl |
| `Inotify::do_init` post: error path balances user counter | shared impl |

### Layer 4: Verus/Creusot functional

Per-`inotify_init(2)` man page; per-LTP `testcases/kernel/syscalls/inotify_init/*`; per-glibc `sysdeps/unix/syscalls.list`; per-`Documentation/filesystems/inotify.rst`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

inotify_init reinforcement:

- **Per-user instance cap strict** — defense against per-fd-flood DoS via inotify-instance creation.
- **Per-FD_CLOEXEC NOT set is documented** — defense-in-depth: users explicitly aware that they must set CLOEXEC themselves before exec, or use `inotify_init1`.
- **Per-O_NONBLOCK NOT set is documented** — defense against per-blocking-read-stuck readers.
- **Per-rollback on partial failure** — defense against per-leaked-fsnotify_group / per-leaked-user-counter.

### grsecurity/pax-style reinforcement

- **PaX UDEREF irrelevant** — `inotify_init` takes no arguments; no userspace pointer dereference.
- **PAX_RANDKSTACK on every entry** — randomise kstack on every `inotify_init` entry; defense against per-stack-spray of anon-file creation path.
- **GRKERNSEC_HIDESYM on `fsnotify_group`, `INOTIFY_FOPS` symbol** — fsnotify group struct and file operations table excluded from `/proc/kallsyms` under `kptr_restrict ≥ 2`; defense against per-fsnotify layout reconnaissance.
- **Per-user inotify-instance quota MUCH stricter under hardened policy** — `fs.inotify.max_user_instances` may be set to 8 (rather than 128) for untrusted uids; defense against per-fd-table-flood.
- **Audit cross-user inotify-watch from same fsnotify-group** — defense against per-group-aliased-watch leakage.
- **Refuse `inotify_init` from setuid programs** — under hardened policy, suid programs MUST use `inotify_init1(IN_CLOEXEC)`; defense against per-fd-inheritance leak across `execve` post-setuid.
- **Deprecated-syscall ENOSYS gate (off by default)** — grsec hardened policy may force `inotify_init(2)` to `-ENOSYS`, forcing applications onto `inotify_init1(IN_CLOEXEC | IN_NONBLOCK)` which has explicit CLOEXEC; defense against per-legacy-API surface, AND defense against per-fd-leak-across-exec footgun.
- **Audit FD_CLOEXEC-not-set on inotify fd at exec boundary** — every `execve` from a task that holds a non-CLOEXEC inotify fd audit-logged; defense against per-fd-leak reconnaissance.
- **GRKERNSEC_CHROOT restrict inotify-init under chroot** — chrooted process attempting `inotify_init` denied unless explicitly permitted; defense against per-chroot escape via `/proc`-watching.
- **Per-user-ns scoping strict** — `user_instance_count` indexed by real-uid in caller's user-ns; defense against per-userns CAP-laundering.
- **GRKERNSEC_PROC_GID restrict /proc/<pid>/fd inotify entries** — `/proc/<pid>/fd/N` symlink target "anon_inode:inotify" coarsened to hide whether the fd is inotify under hardened policy; defense against per-process surveillance reconnaissance.
- **CAP_SYS_RESOURCE pre-check for cap-raise** — under hardened policy, lifting `fs.inotify.max_user_instances` requires CAP_SYS_RESOURCE in target user-ns; defense against per-userns CAP-laundering.

