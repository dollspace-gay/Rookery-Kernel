---
title: "Tier-5 syscall: pidfd_open(2) — syscall 434"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pidfd_open(2)` allocates a **pidfd** — a process-stable file-descriptor handle to a TGID-leader task. Unlike a PID integer (which can be reused after the task exits), a pidfd refers to one specific `struct pid` for the lifetime of the fd, eliminating the entire class of PID-reuse races that plague legacy `kill(pid, sig)` / `waitpid(pid)` patterns. Added in Linux 5.3, it is the foundation of the modern Linux process API: combined with `pidfd_send_signal(2)`, `pidfd_getfd(2)`, `waitid(P_PIDFD)`, `setns(2)` on pidfd, and `clone3(CLONE_PIDFD)`, it provides race-free process management.

Critical for: container runtimes (runc, crun, youki, podman), service supervisors (systemd, s6, runit), language-runtime supervisors (Go `os.Process`, Rust `Command`, Python `asyncio.subprocess`), debuggers (gdb, lldb pidfd-based attach), security sandboxes (firejail, bubblewrap), every "wait-for-process-exit-without-zombie-race" pattern.

### Acceptance Criteria

- [ ] AC-1: `pidfd_open(getpid(), 0)` returns a valid fd.
- [ ] AC-2: `pidfd_open(1, 0)` (init) returns a valid fd or `EPERM` (cross-ns).
- [ ] AC-3: `pidfd_open(NONEXISTENT_PID, 0)` returns `ESRCH`.
- [ ] AC-4: `pidfd_open(-1, 0)` returns `EINVAL`.
- [ ] AC-5: `pidfd_open(pid, 0xDEADBEEF)` returns `EINVAL` (invalid flags).
- [ ] AC-6: `pidfd_open(non_leader_tid, 0)` returns `EINVAL`.
- [ ] AC-7: `pidfd_open(non_leader_tid, PIDFD_THREAD)` returns a valid fd (5.19+).
- [ ] AC-8: Returned fd has `FD_CLOEXEC == 1` (verified via `fcntl(F_GETFD)`).
- [ ] AC-9: `poll(pidfd, POLLIN, -1)` blocks until target exits, then returns `POLLIN`.
- [ ] AC-10: `pidfd_open(getpid(), PIDFD_NONBLOCK)` returns fd with `O_NONBLOCK` set.
- [ ] AC-11: After target exits and pidfd persists, `pidfd_send_signal(pidfd, ...)` returns `ESRCH`.
- [ ] AC-12: `PIDFD_SELF` returns equivalent fd to `pidfd_open(getpid(), 0)` (5.19+).
- [ ] AC-13: `read(pidfd, ...)` returns `EINVAL` (reserved).
- [ ] AC-14: Two `pidfd_open` calls for the same pid return distinct fds referring to the same `struct pid`.

### Architecture

```rust
#[syscall(nr = 434, abi = "sysv")]
pub fn sys_pidfd_open(pid: i32, flags: u32) -> KResult<i32> {
    PidfdOpen::do_pidfd_open(pid, flags)
}
```

`PidfdOpen::do_pidfd_open(pid, flags) -> KResult<i32>`:
1. /* Validate flags */
2. const ALLOWED: u32 = PIDFD_NONBLOCK | PIDFD_THREAD;
3. if flags & !ALLOWED != 0 { return Err(EINVAL); }
4. /* Resolve pid */
5. let target_pid: Arc<Pid> = match pid {
   - PIDFD_SELF_THREAD_GROUP => task_tgid(current()).clone(),
   - PIDFD_SELF_THREAD if flags & PIDFD_THREAD != 0 => task_pid(current()).clone(),
   - n if n > 0 => find_get_pid(n).ok_or(ESRCH)?,
   - _ => return Err(EINVAL),
6. };
7. /* Verify TGID-leader unless PIDFD_THREAD */
8. if flags & PIDFD_THREAD == 0 {
   - let task = target_pid.task_in(PIDTYPE_TGID).ok_or(ESRCH)?;
   - if task.tgid != task.pid { return Err(EINVAL); }   /* non-leader */
9. }
10. /* LSM check */
11. security_task_to_pidfd(target_pid)?;
12. /* Allocate pidfd */
13. let file = anon_inode_getfile(
   - "[pidfd]",
   - &PIDFD_FOPS,
   - target_pid.clone(),
   - O_RDWR | (flags & PIDFD_NONBLOCK),
14. )?;
15. /* PIDFD_CLOEXEC mandatory */
16. let fd = get_unused_fd_flags(O_CLOEXEC)?;
17. fd_install(fd, file);
18. Ok(fd)

`pidfd_fops::poll(file, wait) -> __poll_t`:
1. let pid: &Arc<Pid> = file.private_data;
2. poll_wait(file, &pid.wait_pidfd, wait);
3. let task = pid.task_in(PIDTYPE_TGID);
4. match task {
   - None => EPOLLIN | EPOLLRDNORM,   /* exited */
   - Some(t) if t.exit_state == EXIT_ZOMBIE => EPOLLIN | EPOLLRDNORM,
   - _ => 0,
5. }

`pidfd_fops::release(file)`:
1. let pid: Arc<Pid> = file.private_data;
2. drop(pid);   /* refcount decrement; releases struct pid if last ref */

### Out of Scope

- `pidfd_send_signal(2)` (Tier-5 separate doc — signal via pidfd).
- `pidfd_getfd(2)` (Tier-5 separate doc — steal-fd via pidfd).
- `waitid(P_PIDFD)` (Tier-5 in `waitid.md`).
- `clone3(CLONE_PIDFD)` (Tier-5 in `clone3.md`).
- `setns(2)` via pidfd (Tier-5 in `setns.md`).
- struct pid internals (Tier-3 in `kernel/pid.md`).
- Implementation code.

### signature

```c
int pidfd_open(pid_t pid, unsigned int flags);
```

### parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `pid` | `pid_t` | in | Target TGID in caller's PID namespace. MUST be > 0; `PIDFD_SELF` (`0`) for self-targeting (5.19+); `-1` reserved. |
| `flags` | `unsigned int` | in | Bitmask: `PIDFD_NONBLOCK` (sets `O_NONBLOCK`); `PIDFD_THREAD` (5.19+, allow non-TGID-leader). |

### return value

| Value | Meaning |
|---|---|
| ≥ 0 | New pidfd referring to the target process. Inherits `O_CLOEXEC` by default. |
| `-1` | Error; `errno` set. |

### errors

| `errno` | Cause |
|---|---|
| `EINVAL` | `pid <= 0` (except `PIDFD_SELF`); `flags` contains unknown bits. |
| `ESRCH` | No process with `pid` exists in caller's PID namespace. |
| `EMFILE` | Per-process fd limit reached. |
| `ENFILE` | System-wide fd table exhausted. |
| `ENOMEM` | Kernel could not allocate pidfd backing. |

### abi surface

```text
__NR_pidfd_open  (x86_64) = 434
__NR_pidfd_open  (i386)   = 434
__NR_pidfd_open  (arm64)  = 434
__NR_pidfd_open  (generic)= 434

/* Flags */
#define PIDFD_NONBLOCK   O_NONBLOCK     /* 0x800 — read does not block */
#define PIDFD_THREAD     O_EXCL         /* 0x80 — 5.19+: target may be non-leader thread */

/* Self-target (5.19+): */
#define PIDFD_SELF                 PIDFD_SELF_THREAD_GROUP
#define PIDFD_SELF_THREAD_GROUP    -10000   /* this task's TGID leader */
#define PIDFD_SELF_THREAD          -20000   /* this task itself (with PIDFD_THREAD) */

/* Default fd flags: */
/* pidfd_open ALWAYS sets FD_CLOEXEC by default (5.10+; mandatory 5.12+). */
/* This is DIFFERENT from signalfd/eventfd/epoll_create legacy behavior. */
```

### compatibility contract

REQ-1: Syscall number is **434** on all architectures (added in 5.3; uniformly numbered).

REQ-2: `pid` MUST be a positive integer representing a TGID in the caller's active PID namespace. `pid == 0`: prior to 5.19 ⇒ `EINVAL`; 5.19+ with `PIDFD_SELF` semantics ⇒ self. Negative `pid` ⇒ `EINVAL`.

REQ-3: `flags` bits permitted: `PIDFD_NONBLOCK` (5.10+), `PIDFD_THREAD` (5.19+). Other bits ⇒ `EINVAL`.

REQ-4: Default (no `PIDFD_THREAD`): `pid` MUST be a TGID-leader. Targeting a non-leader thread ⇒ `EINVAL`. With `PIDFD_THREAD`: any task is permitted.

REQ-5: Returned fd is automatically `FD_CLOEXEC`. This is mandatory; userspace MUST NOT rely on the absence of CLOEXEC.

REQ-6: `PIDFD_NONBLOCK`: makes the pidfd's poll/read semantics non-blocking. `poll(2)` will return `POLLIN` once the target has exited; without `NONBLOCK`, poll blocks until exit.

REQ-7: pidfd is `poll`-able: `POLLIN` asserts when target has exited (becomes a zombie or is reaped). `epoll`/`select` integration works.

REQ-8: pidfd is `read`-able only with `PIDFD_NONBLOCK + future-extension` (current upstream returns `EINVAL` on read; reserved for future).

REQ-9: pidfd lifetime: the fd holds a reference to `struct pid` (not `task_struct`). The fd remains valid even after the target exits (post-exit, the fd refers to the now-dead pid; `pidfd_send_signal` returns `ESRCH`).

REQ-10: `dup(2)`/`dup2(2)`/`fcntl(F_DUPFD)`: dup'd pidfd shares the underlying `struct pid`. Multiple pidfds to the same process are independent file references but the same pid-tracking.

REQ-11: Self-target via `PIDFD_SELF` (5.19+): equivalent to `pidfd_open(getpid(), 0)` for TGID, or `pidfd_open(gettid(), PIDFD_THREAD)` for self-thread. Safe shorthand that avoids the round-trip through `getpid`.

REQ-12: PaX UDEREF: not applicable (no userspace pointer args).

REQ-13: LSM hook: `security_task_to_pidfd` MAY fire (5.18+); SELinux/AppArmor MAY deny on cross-domain pidfd creation.

REQ-14: Per-PID-namespace: `pid` is resolved in caller's active PID-ns. A pidfd obtained in an inner ns refers to a task in that ns; passing the fd via SCM_RIGHTS to outer-ns process: outer process can use the fd but `pidfd_send_signal` and `waitid(P_PIDFD)` see the task in the outer-ns view (via the underlying `struct pid`).

REQ-15: `seccomp` filters see `pid` and `flags`; SHOULD be allowed for unprivileged process supervisors.

REQ-16: `clone3(CLONE_PIDFD)` is the race-free way to obtain a pidfd for a JUST-created child without a `pidfd_open` lookup race (the child PID could be reused before `pidfd_open`).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pid_positive_or_self` | INVARIANT | per-pidfd_open: pid > 0 OR PIDFD_SELF. |
| `flags_validated` | INVARIANT | per-pidfd_open: flags & ~ALLOWED == 0. |
| `tgid_leader_check` | INVARIANT | per-default: target is TGID-leader unless PIDFD_THREAD. |
| `cloexec_mandatory` | INVARIANT | per-return: O_CLOEXEC set on fd. |
| `pid_refcount_balanced` | INVARIANT | per-find_get_pid/drop: ref balanced. |
| `pid_ns_resolution` | INVARIANT | per-pid: resolved in caller's active PID-ns. |

### Layer 2: TLA+

`kernel/pidfd-open.tla`:
- States: per-pidfd struct-pid ref, per-task exit_state, per-fd-table mapping.
- Properties:
  - `safety_pidfd_stable` — per-pidfd: refers to same struct pid for fd lifetime.
  - `safety_no_pid_reuse_observation` — per-pidfd: never refers to a reused PID.
  - `safety_cloexec_always` — per-create: FD_CLOEXEC always set.
  - `liveness_pidfd_signaled_on_exit` — per-target-exit: pidfd's poll becomes POLLIN.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_pidfd_open` post: ret fd refers to anon_inode[pidfd] | `PidfdOpen::do_pidfd_open` |
| `poll` post: POLLIN iff target exited | `pidfd_fops::poll` |
| `release` post: struct pid ref decremented | `pidfd_fops::release` |

### Layer 4: Verus / Creusot functional

Per-`pidfd_open(2)` man-page equivalence. LTP `pidfd_open01..04` pass. Round-trip with `clone3(CLONE_PIDFD)`, `waitid(P_PIDFD)`, `pidfd_send_signal` semantically equivalent.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`pidfd_open(2)` reinforcement:

- **Per-FD_CLOEXEC mandatory** — defense against per-pidfd-leak via execve.
- **Per-PID-namespace resolution strict** — defense against per-cross-ns target hijack.
- **Per-struct-pid ref count** — defense against per-stale-pidfd UAF.
- **Per-TGID-leader gate** — defense against per-thread-pidfd confusion attacks.
- **Per-LSM task-to-pidfd hook** — defense against per-cross-domain pidfd creation.

### grsecurity / pax-style reinforcement

- **PIDFD_SELF for safe self-targeting** — `PIDFD_SELF` (5.19+) is the canonical safe-self pattern in grsec hardened mode; it eliminates the getpid → pidfd_open round-trip race where a fork between calls could redirect targeting. Hardened userspace SHOULD prefer `PIDFD_SELF` for self-shutdown / self-watch.
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — `pidfd_open` of a process the caller cannot `ptrace_attach` to is denied (returns EPERM) in hardened mode. This prevents cross-uid / cross-cgroup process inspection via pidfd-based side channels. Upstream allows pidfd_open of any visible task; grsec tightens to ptrace-attach-eligible only.
- **GRKERNSEC_PROC restrictions** — `pidfd_open(pid)` for a `pid` that is hidden from caller's `/proc` view returns ESRCH (consistent with /proc visibility). The set of resolvable pids matches `/proc/<pid>/` listings precisely.
- **PIDFD_THREAD CAP_SYS_PTRACE gate** — opening a non-leader thread's pidfd is treated as ptrace-equivalent; grsec requires CAP_SYS_PTRACE for cross-uid PIDFD_THREAD opens (upstream allows same-uid).
- **PaX UDEREF on syscall args** — not directly applicable (no userspace pointer args); the pidfd_send_signal/pidfd_getfd companion calls handle their own UDEREF.
- **signalfd CLOEXEC mandatory** — analogous: pidfd CLOEXEC is mandatory and grsec enforces it cannot be cleared via fcntl(F_SETFD, 0) without explicit opt-in. (Upstream allows clearing CLOEXEC post-create; grsec adds a sysctl to lock it.)
- **EPOLL fd-lifetime refcount strict** — when pidfd is watched by epoll for exit-notification (common pattern), the file ref is held by the epitem; grsec asserts proper release on EPOLL_CTL_DEL.
- **GRKERNSEC_AUDIT_GROUP** — burst-rate pidfd_open from a marked group is audited (potential process-enumeration scan, e.g. attempting to find any-pid via repeated ESRCH probes).
- **No_new_privs neutral** — NNP does not change pidfd_open semantics.
- **Anti-fingerprint hardening** — pidfd_open does not echo kernel struct pid pointer; the fd is opaque. /proc/<self>/fdinfo/<pidfd>:Pid line is filtered to render pid in caller's namespace.
- **PID-namespace mapping strict** — pidfd carries the struct pid which is ns-agnostic at the kernel level; when used cross-ns (e.g. via SCM_RIGHTS), grsec verifies the receiver has visibility of the target's PID-namespace ancestor before allowing signal/getfd operations.
- **Capability gate for setns via pidfd** — `setns(pidfd, 0)` requires CAP_SYS_ADMIN in the target namespace; grsec audits all setns attempts via pidfd in marked groups.
- **Seccomp interaction** — seccomp filters may restrict `pidfd_open` to PIDFD_SELF / known-pids only; default-deny hardening recommends permitting only self-target and clone3(CLONE_PIDFD)-derived pidfds.

