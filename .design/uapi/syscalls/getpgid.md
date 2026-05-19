# Tier-5 syscall: getpgid(2) — syscall 121

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE1(getpgid))
  - include/linux/sched/signal.h (task_pgrp, task_pgrp_vnr)
  - include/linux/pid.h (find_task_by_vpid, pid_vnr)
  - arch/x86/entry/syscalls/syscall_64.tbl (121  common  getpgid)
-->

## Summary

`getpgid(2)` returns the **process-group ID** (PGID) of the process identified by `pid`, mapped into the caller's pid-namespace. With `pid == 0`, returns the caller's own PGID. Distinct from `getpgrp(2)`, which always returns the caller's PGID with no argument (legacy convenience).

The PGID identifies a process group — a set of processes that receive job-control signals together. Shells assemble pipelines into a process group via `setpgid(2)`; the kernel routes `kill(-pgid, sig)` and tty signals (SIGINT on Ctrl-C, SIGTSTP on Ctrl-Z) to all group members. The PGID is the PID of the group leader.

Critical for: shell job-control state tracking, `waitpid(-pgid, ...)` filtering, container runtime's per-container pgroup tracking, audit subsystem's group-correlation, `ps -o pgid` and `htop` group display, `kill -- -pgid` user-space patterns.

## Signature

```c
pid_t getpgid(pid_t pid);
```

## Parameters

| name | type    | constraints                                                              | errno-on-bad |
|------|---------|--------------------------------------------------------------------------|--------------|
| pid  | `pid_t` | `0` = caller; otherwise must refer to a task visible in caller's pid-ns. | `ESRCH`      |

## Return value

| Value | Meaning |
|---|---|
| `> 0` | The PGID of the target process, mapped into caller's pid-namespace. |
| `-1` | Error; `errno` set. |

## Errors

| errno   | condition                                                                       |
|---------|---------------------------------------------------------------------------------|
| `ESRCH` | No task exists with the specified `pid` in caller's pid-namespace.              |
| `EPERM` | (POSIX-optional / Linux historical) Cross-session inquiry restricted. Linux does NOT enforce; never returns EPERM here on current kernels. |

## ABI surface

```text
__NR_getpgid (x86_64)   = 121
__NR_getpgid (i386)     = 132
__NR_getpgid (arm64)    = 155    /* generic */
__NR_getpgid (generic)  = 155

typedef __kernel_pid_t  pid_t;   /* signed int */

struct signal_struct {
    ...
    struct pid *pids[PIDTYPE_MAX];   /* PIDTYPE_PGID */
    ...
};

/* Translation: kernel pid → caller's pid-ns:
 *   pid_t = task_pgrp_vnr(task)
 *         = pid_nr_ns(task->signal->pids[PIDTYPE_PGID], task_active_pid_ns(current))
 */
```

## Compatibility contract

REQ-1: Syscall number is **121** on x86_64; **155** on arm64 / generic. ABI-stable since Linux 2.0.

REQ-2: When `pid == 0`: returns `task_pgrp_vnr(current)`. When `pid != 0`: looks up task by `find_task_by_vpid(pid)`; returns its PGID.

REQ-3: If lookup fails: `-ESRCH`.

REQ-4: Linux does NOT enforce the POSIX optional EPERM (cross-session inquiry restriction). Any process can query any visible task's PGID.

REQ-5: PGID is returned as `task_pgrp_vnr(task)` which translates `task.signal.pids[PIDTYPE_PGID]` to caller's pid-ns via `pid_vnr`. Returns 0 if the group leader's pid is not visible in caller's pid-ns (cross-ns reach-up).

REQ-6: After fork: child inherits parent's PGID. After execve: PGID preserved.

REQ-7: After `setpgid(target, X)`: `getpgid(target)` returns X.

REQ-8: After `setsid()`: caller's new PGID == caller's PID; `getpgid(0)` returns that value.

REQ-9: `getpgid()` is async-signal-safe and re-entrant; RCU-read of `task.signal->pids[PIDTYPE_PGID]`.

REQ-10: No LSM hook on the getpgid path.

REQ-11: `getpgid()` does NOT mutate any task state.

REQ-12: Inside a pid-namespace where the group leader was created in a parent ns: `getpgid()` returns 0 (group leader pid not visible at this depth).

REQ-13: PGID is the PID of the process-group leader. If the group leader has exited but other group members remain (orphaned group): the PGID is still valid as long as the leader's pid has not been reaped (zombie reservation keeps the pid).

REQ-14: All threads in a thread group share a PGID (since signal_struct is per-tgid). Calling `getpgid(tid)` for any thread returns the thread group's PGID.

## Acceptance Criteria

- [ ] AC-1: `getpgid(0)` returns current's PGID.
- [ ] AC-2: `getpgid(getpid())` returns same as `getpgid(0)`.
- [ ] AC-3: After `setpgid(0, 0)`: `getpgid(0) == getpid()`.
- [ ] AC-4: Child after fork: `getpgid(0)` matches parent's pre-fork PGID.
- [ ] AC-5: Sibling in same group: `getpgid(sibling) == getpgid(0)`.
- [ ] AC-6: After `setsid()`: `getpgid(0) == getpid()`.
- [ ] AC-7: `getpgid(invalid_pid)` returns -1 with errno == ESRCH.
- [ ] AC-8: Across pid-namespaces: returns 0 if group leader not visible.
- [ ] AC-9: `getpgid()` consistent with `/proc/<pid>/stat` pgrp field.
- [ ] AC-10: All threads of a process return same `getpgid(0)`.
- [ ] AC-11: `getpgid()` does not modify any task.
- [ ] AC-12: After execve: PGID preserved; `getpgid(0)` unchanged.

## Architecture

```rust
#[syscall(nr = 121, abi = "sysv")]
pub fn sys_getpgid(pid: pid_t) -> SyscallResult<pid_t> {
    Getpgid::do_getpgid(pid)
}
```

`Getpgid::do_getpgid(pid) -> Result<pid_t>`:
1. let task = if pid == 0 {
2.     current()
3. } else {
4.     rcu::read(|| find_task_by_vpid(pid))
5.         .ok_or(Errno::ESRCH)?
6. };
7. /* RCU-read PIDTYPE_PGID slot */
8. let pgid = rcu::read(|| {
9.     PidNs::pid_vnr(task.signal.pids[PIDTYPE_PGID])
10. });
11. Ok(pgid)

`task_pgrp_vnr(task) -> pid_t`:
1. pid_nr_ns(task.signal.pids[PIDTYPE_PGID], task_active_pid_ns(current()))

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pid_zero_returns_self_pgid` | INVARIANT | per-getpgid: pid == 0 ⟹ ret == task_pgrp_vnr(current). |
| `unknown_pid_returns_esrch` | INVARIANT | per-getpgid: lookup fails ⟹ -ESRCH. |
| `pgid_in_pid_ns` | INVARIANT | per-getpgid: ret is PGID in caller's pid-ns or 0. |
| `rcu_atomic` | INVARIANT | per-getpgid: concurrent setpgid yields consistent snapshot. |
| `no_state_mutation` | INVARIANT | per-getpgid: read-only. |
| `same_session_same_visibility` | INVARIANT | per-getpgid: targets in caller's session have non-zero PGID return. |

### Layer 2: TLA+

`kernel/getpgid.tla`:
- States: per-task PGID, pid-ns hierarchy.
- Properties:
  - `safety_self_lookup` — pid==0 path returns current's PGID.
  - `safety_unknown_pid_esrch` — invalid pid path returns ESRCH.
  - `safety_no_mutation` — getpgid never mutates state.
  - `liveness_returns` — getpgid terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getpgid(0)` post: ret == task_pgrp_vnr(current) | `Getpgid::do_getpgid` |
| `do_getpgid(pid)` post (success): ret == task_pgrp_vnr(target) | `Getpgid::do_getpgid` |
| `do_getpgid(pid)` post (fail): ret == Err(ESRCH) | `Getpgid::do_getpgid` |

### Layer 4: Verus / Creusot functional

Per-`getpgid(2)` man-page equivalence. LTP `getpgid01`, `getpgid02` pass. POSIX.1-2008 `getpgid` semantics verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getpgid(2)` reinforcement:

- **Per-RCU snapshot** — defense against per-torn-pid-read race with concurrent setpgid / exit.
- **Per-pid-ns-translation** — defense against per-namespace-leak (kernel-global group leader pid would otherwise leak across ns boundaries).
- **Per-readonly semantics** — defense against per-mutation-via-getpgid bug pattern.
- **Per-find_task_by_vpid scoped to caller's ns** — defense against per-cross-ns pid-discovery.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at getpgid entry** — randomizes kernel stack offset; defeats stack-layout fingerprinting via high-frequency `getpgid()` probes.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/stat`'s pgrp field is restricted to owning user and CAP_SYS_ADMIN under grsec; `getpgid(0)` on self always succeeds.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec chroot, `getpgid(pid)` returns ESRCH for pids outside the chroot pid-ns; group-leader pids never leak across chroot.
- **CAP_SETUID / CAP_SETGID strict** — getpgid is a pure read; not gated by cred-change caps.
- **GRKERNSEC_AUDIT_GROUP on cred-change** — getpgid is not itself a cred-change; logged when followed by setpgid on audited users.
- **GRKERNSEC_HIDESYM** — getpgid returns only the PGID; no kernel pointers exposed.
- **No LSM hook** — getpgid has no security hook; grsec adds optional audit but never blocks.
- **PGID enumeration limit** — grsec can rate-limit `getpgid()` per uid to detect attacker enumerating process groups (e.g. reconnaissance prior to targeted kill).
- **PGID-leader zombie protection** — if group leader has exited but pid not yet reaped: getpgid returns the (zombie) pid in caller's ns; grsec validates the pid+uid pair to prevent stale info-leak.
- **Uniform ESRCH** — grsec ensures ESRCH return does not distinguish "no such pid" from "exists but cross-ns hidden"; prevents probing for hidden tasks.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `getpgrp(2)` (Tier-5 separate doc — caller's PGID with no arg).
- `setpgid(2)` (Tier-5 separate doc — set PGID).
- `getsid(2)` / `setsid(2)` (Tier-5 separate docs — session level).
- Job-control / signal-to-pgroup routing (Tier-3 in `kernel/signal.md`).
- pid-namespace internals (Tier-3 in `kernel/pid_namespace.md`).
- Implementation code.
