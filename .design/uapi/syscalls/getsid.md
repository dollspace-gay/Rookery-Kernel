# Tier-5 syscall: getsid(2) — syscall 124

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE1(getsid))
  - include/linux/sched.h (struct task_struct.signal->__session)
  - include/linux/sched/signal.h (task_session, task_session_vnr)
  - include/linux/pid.h (pid_vnr, find_task_by_vpid)
  - arch/x86/entry/syscalls/syscall_64.tbl (124  common  getsid)
-->

## Summary

`getsid(2)` returns the **session ID** (SID) of the process identified by `pid`, mapped into the caller's pid-namespace. A session is a collection of process groups, typically associated with a controlling terminal; it is created by `setsid(2)`, which makes the calling process a session leader and a new process-group leader. The SID is the PID of the session leader.

Sessions form the outer of the three-level POSIX job-control hierarchy: **session → process group → process**. Within a session, exactly one process group can be the **foreground** group (eligible to read from the controlling tty); others are background. The session leader is the only process allowed to acquire a controlling tty (`TIOCSCTTY`).

Critical for: shell job control (`bash`, `zsh`), `setsid(1)` / daemon double-fork, `nohup`, `tmux` / `screen` session detach, `systemd-logind` session bookkeeping, container init `setsid()` to detach from host tty, audit trail correlation by session.

## Signature

```c
pid_t getsid(pid_t pid);
```

## Parameters

| name | type    | constraints                                                                          | errno-on-bad |
|------|---------|--------------------------------------------------------------------------------------|--------------|
| pid  | `pid_t` | `0` = caller; otherwise must refer to an existing task visible in caller's pid-ns.    | `ESRCH`      |

## Return value

| Value | Meaning |
|---|---|
| `> 0` | The SID of the target process, mapped into caller's pid-namespace. |
| `-1` | Error; `errno` set. |

## Errors

| errno   | condition                                                                  |
|---------|----------------------------------------------------------------------------|
| `ESRCH` | No task exists with the specified `pid` in the caller's pid-namespace.     |
| `EPERM` | (Linux historical / per-POSIX option) The target's session is in a different session and the caller is not the session leader; current Linux returns the SID regardless. |

## ABI surface

```text
__NR_getsid (x86_64)   = 124
__NR_getsid (i386)     = 147
__NR_getsid (arm64)    = 156     /* generic */
__NR_getsid (generic)  = 156

typedef __kernel_pid_t  pid_t;

struct signal_struct {
    ...
    struct pid *pids[PIDTYPE_MAX];   /* PIDTYPE_TGID, _PGID, _SID */
    ...
};

/* PIDTYPE_SID indexes the session-leader pid for the process. */
```

## Compatibility contract

REQ-1: Syscall number is **124** on x86_64; **156** on arm64 / generic. ABI-stable since Linux 2.0.

REQ-2: When `pid == 0`: returns `task_session_vnr(current)`. When `pid != 0`: looks up task by `find_task_by_vpid(pid)`; returns its SID.

REQ-3: If lookup fails: `-ESRCH`.

REQ-4: Linux does NOT enforce the POSIX optional EPERM (cross-session inquiry restriction). Any process can query any visible task's SID.

REQ-5: SID is returned via `task_session_vnr(task)` which translates `task.signal.pids[PIDTYPE_SID]` to the caller's pid-namespace via `pid_vnr`. Returns 0 if the session-leader pid is not visible in the caller's pid-ns.

REQ-6: Multiple processes can share a session; their SIDs are all equal to the session leader's PID.

REQ-7: After `setsid(2)`: the calling process becomes session leader, its new SID equals its own TGID, and `getsid(0)` returns that value.

REQ-8: After `fork(2)`: child inherits parent's SID. After `execve(2)`: SID is preserved.

REQ-9: `getsid()` is async-signal-safe and re-entrant; RCU-read of `task.signal->pids[PIDTYPE_SID]`.

REQ-10: No LSM hook on the getsid path.

REQ-11: `getsid()` does NOT mutate any task state.

REQ-12: SID is returned as `pid_t` (positive value on success); 0 indicates the session leader's pid is not visible in the caller's pid-ns (cross-ns reach-up).

REQ-13: Inside a pid-namespace where session leader was created in a parent ns: `getsid()` returns 0 (session leader pid not visible at this depth) — caller cannot reach upward.

## Acceptance Criteria

- [ ] AC-1: `getsid(0)` returns current's SID.
- [ ] AC-2: `getsid(getpid())` returns same value as `getsid(0)`.
- [ ] AC-3: After `setsid()`: `getsid(0) == getpid()`.
- [ ] AC-4: Child after fork: `getsid(0)` matches parent's pre-fork SID.
- [ ] AC-5: Sibling process in same session: `getsid(sibling_pid) == getsid(0)`.
- [ ] AC-6: Process in different session: `getsid(other_pid) != getsid(0)`.
- [ ] AC-7: `getsid(invalid_pid)` returns -1 with `errno == ESRCH`.
- [ ] AC-8: After `execve()`: `getsid(0)` unchanged.
- [ ] AC-9: Inside a pid-ns, `getsid()` of a task whose session-leader is in an outer ns returns 0.
- [ ] AC-10: `getsid()` consistent with `/proc/<pid>/stat` field `session`.
- [ ] AC-11: `getsid()` does not modify any task.

## Architecture

```rust
#[syscall(nr = 124, abi = "sysv")]
pub fn sys_getsid(pid: pid_t) -> SyscallResult<pid_t> {
    Getsid::do_getsid(pid)
}
```

`Getsid::do_getsid(pid) -> Result<pid_t>`:
1. let task = if pid == 0 {
2.     current()
3. } else {
4.     rcu::read(|| find_task_by_vpid(pid))
5.         .ok_or(Errno::ESRCH)?
6. };
7. /* RCU-read task.signal.pids[PIDTYPE_SID] */
8. let sid = rcu::read(|| {
9.     PidNs::pid_vnr(task.signal.pids[PIDTYPE_SID])
10. });
11. Ok(sid)

`PidNs::pid_vnr(pid) -> pid_t`:
1. if pid.is_null() { return 0; }
2. let active = task_active_pid_ns(current());
3. pid_nr_ns(pid, active)

`find_task_by_vpid(vpid) -> Option<*task_struct>`:
1. let active = task_active_pid_ns(current());
2. let pid = find_pid_ns(vpid, active)?;
3. pid_task(pid, PIDTYPE_PID)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pid_zero_returns_self_sid` | INVARIANT | per-getsid: pid == 0 ⟹ ret == task_session_vnr(current). |
| `unknown_pid_returns_esrch` | INVARIANT | per-getsid: lookup fails ⟹ -ESRCH. |
| `sid_in_pid_ns` | INVARIANT | per-getsid: ret is the session pid in caller's pid-ns or 0. |
| `rcu_atomic` | INVARIANT | per-getsid: concurrent setsid/exit yields consistent snapshot. |
| `no_state_mutation` | INVARIANT | per-getsid: read-only. |

### Layer 2: TLA+

`kernel/getsid.tla`:
- States: per-task SID, pid-ns hierarchy.
- Properties:
  - `safety_self_lookup` — pid==0 path returns current's SID.
  - `safety_unknown_pid_esrch` — invalid pid path returns ESRCH.
  - `safety_no_mutation` — getsid never mutates state.
  - `liveness_returns` — getsid terminates in O(pid-ns-depth + hash-lookup) ≤ O(32).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getsid(0)` post: ret == task_session_vnr(current) | `Getsid::do_getsid` |
| `do_getsid(pid)` post (lookup success): ret == task_session_vnr(target) | `Getsid::do_getsid` |
| `do_getsid(pid)` post (lookup fail): ret == Err(ESRCH) | `Getsid::do_getsid` |

### Layer 4: Verus / Creusot functional

Per-`getsid(2)` man-page equivalence. LTP `getsid01`, `getsid02` pass. POSIX.1-2008 `getsid` semantics verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getsid(2)` reinforcement:

- **Per-RCU snapshot** — defense against per-torn-pid-read race with concurrent setsid/exit.
- **Per-pid-ns-translation** — defense against per-namespace-leak (kernel-global session leader pid would otherwise leak across ns).
- **Per-readonly semantics** — defense against per-mutation-via-getsid bug pattern.
- **Per-find_task_by_vpid scoped to caller's ns** — defense against per-cross-ns pid-discovery.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at getsid entry** — randomizes kernel stack offset; defeats stack-layout fingerprinting via high-frequency `getsid()` probes.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/stat`'s session field is restricted to owning user and CAP_SYS_ADMIN under grsec; `getsid(0)` on self always works. Querying other users' SIDs is gated separately from this syscall.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec chroot, `getsid(pid)` for a task outside the chroot pid-ns returns ESRCH (find_task scoped to chroot's pid-ns); session leader pids never leak across chroot.
- **CAP_SETUID / CAP_SETGID strict** — getsid is a pure read; not gated by cred-change capabilities.
- **GRKERNSEC_AUDIT_GROUP on cred-change** — getsid is not itself a cred-change but is logged when followed by setsid on audited users (correlates session-leader-creation events).
- **GRKERNSEC_HIDESYM** — getsid returns only the SID; no kernel pointers exposed.
- **No LSM hook** — getsid has no security hook; grsec adds optional audit but never blocks.
- **Cross-session enumeration limit** — grsec can rate-limit `getsid()` calls per uid to detect attacker enumerating sessions.
- **Session-leader liveness** — if session leader has exited but session is still referenced (zombie leader): getsid returns the pid in caller's ns (still allocated until reaped); grsec ensures no stale info-leak via recycled pid (kuid+pid pair validated).
- **No info-leak on ESRCH** — grsec ensures ESRCH is returned uniformly whether pid does not exist or exists but is not visible to caller's ns; prevents probing for hidden tasks.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `setsid(2)` (Tier-5 separate doc — creates new session).
- `getpgid(2)` / `setpgid(2)` (Tier-5 separate docs — process-group level).
- `getpgrp(2)` (Tier-5 separate doc).
- Job-control / controlling-terminal semantics (Tier-3 in `drivers/tty/job-control.md`).
- pid-namespace internals (Tier-3 in `kernel/pid_namespace.md`).
- Implementation code.
