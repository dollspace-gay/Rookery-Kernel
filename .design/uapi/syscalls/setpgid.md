# Tier-5 syscall: setpgid(2) — syscall 109

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE2(setpgid), ksys_setpgid)
  - kernel/pid.c (attach_pid, change_pid)
  - include/linux/sched/signal.h (task_pgrp, PIDTYPE_PGID)
  - include/linux/pid.h (find_task_by_vpid)
  - arch/x86/entry/syscalls/syscall_64.tbl (109  common  setpgid)
-->

## Summary

`setpgid(2)` sets the **process-group ID** (PGID) of the process identified by `pid` to `pgid`. Used by shells to assemble pipelines: each command in a pipeline is placed into a single process group so that job-control signals (SIGTSTP, SIGINT) target the entire pipeline as a unit, and the shell can move the group between foreground/background.

The constraints are restrictive: the target process must be the caller or one of the caller's not-yet-execve'd children, the caller and target must be in the same session, the new PGID must either equal the target's PID (creating a new group) or refer to an existing group within the same session, and a session leader cannot change its own PGID.

Critical for: shell pipeline construction (`bash`, `zsh`, `fish`), `tcsetpgrp(3)` foreground-group selection, job-control bring-to-foreground (`fg`/`bg`), `tmux`/`screen` window-group placement, container runtime's per-container pgid grouping.

## Signature

```c
int setpgid(pid_t pid, pid_t pgid);
```

## Parameters

| name | type    | constraints                                                                                                                   | errno-on-bad |
|------|---------|-------------------------------------------------------------------------------------------------------------------------------|--------------|
| pid  | `pid_t` | `0` = caller; otherwise must be caller or descendant (in same session, pre-execve).                                            | `ESRCH`/`EPERM`/`EACCES` |
| pgid | `pid_t` | `0` = target's pid; otherwise must be ≥ 0 and either target's pid (new group) or existing group in same session.               | `EINVAL`/`EPERM` |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `-1` | Error; `errno` set. |

## Errors

| errno    | condition                                                                            |
|----------|--------------------------------------------------------------------------------------|
| `EACCES` | Target is caller's child but has already called `execve(2)`.                          |
| `EINVAL` | `pgid < 0`, or `pgid` does not match `pid` and no group with `pgid` exists in session.|
| `EPERM`  | Target is a session leader (cannot change own pgid). Or target's session differs from caller's. Or `pgid` refers to a group in a different session. |
| `ESRCH`  | No task exists with `pid` in caller's pid-ns; or `pid` is not caller or child.        |

## ABI surface

```text
__NR_setpgid (x86_64)   = 109
__NR_setpgid (i386)     = 57
__NR_setpgid (arm64)    = 154    /* generic */
__NR_setpgid (generic)  = 154

struct signal_struct {
    ...
    struct pid *pids[PIDTYPE_MAX];   /* PIDTYPE_PGID at index PIDTYPE_PGID */
    int leader;                       /* 1 if session leader */
    ...
};
```

## Compatibility contract

REQ-1: Syscall number is **109** on x86_64; **154** on arm64 / generic. ABI-stable since Linux 1.0.

REQ-2: `pid == 0` is shorthand for `current->tgid`. `pgid == 0` is shorthand for `pid` (after resolution).

REQ-3: Validation sequence (in order):
- (a) Resolve target via `find_task_by_vpid(pid)` (or current if pid==0). Failure: ESRCH.
- (b) Target must be caller or child-of-caller (`target.real_parent == current` ∨ target == current). Failure: ESRCH.
- (c) If target is a child: target must not have called execve (no PF_EXEC flag set on signal). Failure: EACCES.
- (d) Target must be in same session as caller. Failure: EPERM.
- (e) Target must not be a session leader. Failure: EPERM.
- (f) If `pgid != pid_vnr(target)`: a process group with that PGID must already exist in the same session. Failure: EPERM.

REQ-4: On success: `change_pid(target, PIDTYPE_PGID, find_vpid(pgid))`.

REQ-5: `setpgid` is permitted on a child process between `fork()` and `execve()`. Shells use this race-resistantly by calling `setpgid` in BOTH parent (after fork, possibly fails harmlessly if child already did it) AND child (immediately after fork) — ensuring the pgid is established before any subsequent waitpid or tcsetpgrp.

REQ-6: Session leader's own PGID is locked to its own pid: a session leader cannot call setpgid on itself with any pgid != its own pid; explicit EPERM.

REQ-7: After successful setpgid, the target's PGID is updated atomically under tasklist_lock (write).

REQ-8: setpgid is async-signal-unsafe (takes tasklist write-lock).

REQ-9: No LSM hook on setpgid path.

REQ-10: Inside a pid-namespace: pid and pgid are resolved in caller's active pid-ns. Cross-ns setpgid is impossible (find_task_by_vpid returns NULL).

REQ-11: Concurrent setpgid on same target from different threads: serialized by tasklist_lock; only one ultimate state.

REQ-12: After execve on a child: the child's PGID is preserved across exec; only the parent's ability to call setpgid on the child is revoked (EACCES).

REQ-13: setpgid does NOT affect controlling-tty foreground-group (tcsetpgrp is a separate ioctl on /dev/tty).

## Acceptance Criteria

- [ ] AC-1: `setpgid(0, 0)` on non-session-leader sets PGID := PID and returns 0.
- [ ] AC-2: Parent calls `setpgid(child_pid, child_pid)` before child execve: returns 0; child's PGID == child_pid.
- [ ] AC-3: Parent calls `setpgid(child_pid, ...)` after child execve: returns -1, errno == EACCES.
- [ ] AC-4: Session leader calls `setpgid(0, X)` for X != own pid: returns -1, errno == EPERM.
- [ ] AC-5: setpgid to a pgid in a different session: -1, EPERM.
- [ ] AC-6: setpgid to a non-existent pgid (not == target pid): -1, EPERM.
- [ ] AC-7: pid not visible in caller's pid-ns: -1, ESRCH.
- [ ] AC-8: pgid negative: -1, EINVAL.
- [ ] AC-9: After successful setpgid: `/proc/<pid>/stat` pgrp field == new pgid.
- [ ] AC-10: After successful setpgid: `getpgid(target)` returns new pgid.
- [ ] AC-11: Sending signal to -pgid reaches target: validates pgid attachment.

## Architecture

```rust
#[syscall(nr = 109, abi = "sysv")]
pub fn sys_setpgid(pid: pid_t, pgid: pid_t) -> SyscallResult<i32> {
    Setpgid::do_setpgid(pid, pgid)
}
```

`Setpgid::do_setpgid(pid, pgid) -> Result<i32>`:
1. let caller = current();
2. let target = if pid == 0 { caller.group_leader } else {
3.     rcu::read(|| find_task_by_vpid(pid)).ok_or(Errno::ESRCH)?
4. };
5. /* Must be caller or child-of-caller */
6. if target != caller.group_leader && target.real_parent != caller.group_leader {
7.     return Err(Errno::ESRCH);
8. }
9. /* Child must not have execve'd */
10. if target.real_parent == caller.group_leader && (target.signal.flags & SIGNAL_GROUP_EXEC) != 0 {
11.     return Err(Errno::EACCES);
12. }
13. /* Same session */
14. if !same_thread_group(target, caller) && pid_vnr(task_session(target)) != pid_vnr(task_session(caller)) {
15.     return Err(Errno::EPERM);
16. }
17. /* Not session leader */
18. if target.signal.leader != 0 {
19.     return Err(Errno::EPERM);
20. }
21. /* Resolve pgid */
22. let resolved_pgid = if pgid == 0 { pid_vnr(task_pid(target)) } else { pgid };
23. if resolved_pgid < 0 {
24.     return Err(Errno::EINVAL);
25. }
26. /* If pgid != target's pid: must be existing group in same session */
27. let new_pgrp = if resolved_pgid == pid_vnr(task_pid(target)) {
28.     task_pid(target)
29. } else {
30.     let g = find_vpid(resolved_pgid).ok_or(Errno::EPERM)?;
31.     /* Validate same session */
32.     let leader = pid_task(g, PIDTYPE_PID).ok_or(Errno::EPERM)?;
33.     if pid_vnr(task_session(leader)) != pid_vnr(task_session(caller)) {
34.         return Err(Errno::EPERM);
35.     }
36.     g
37. };
38. /* Apply atomically */
39. write_lock(&tasklist_lock);
40. change_pid(target, PIDTYPE_PGID, new_pgrp);
41. write_unlock(&tasklist_lock);
42. Ok(0)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `target_is_self_or_child` | INVARIANT | per-setpgid: target == caller ∨ target.real_parent == caller, else ESRCH. |
| `post_execve_blocks_parent` | INVARIANT | per-setpgid: target child + execve'd ⟹ EACCES. |
| `session_leader_locked` | INVARIANT | per-setpgid: target is session leader ⟹ EPERM. |
| `same_session_required` | INVARIANT | per-setpgid: target session != caller session ⟹ EPERM. |
| `pgid_must_exist_in_session` | INVARIANT | per-setpgid: pgid != target.pid ⟹ pgid is existing group in same session. |
| `tasklist_lock_held` | INVARIANT | per-setpgid: tasklist write-lock during change_pid. |

### Layer 2: TLA+

`kernel/setpgid.tla`:
- States: per-task PGID, session, parent, execve-flag.
- Properties:
  - `safety_self_or_child` — setpgid succeeds only on caller or child.
  - `safety_post_exec_immutable` — after execve, parent cannot change child's PGID.
  - `safety_session_invariant` — PGID always refers to a group in same session.
  - `liveness_terminates` — setpgid terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setpgid` pre-check chain | `Setpgid::do_setpgid` |
| `do_setpgid` post (success): target.PGID == new_pgrp | `Setpgid::do_setpgid` |
| `do_setpgid` post (failure): no mutation | `Setpgid::do_setpgid` |

### Layer 4: Verus / Creusot functional

Per-`setpgid(2)` man-page equivalence. LTP `setpgid01..setpgid03` pass. POSIX.1-2008 `setpgid` semantics verified.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setpgid(2)` reinforcement:

- **Per-tasklist-lock write** — defense against per-concurrent-PGID-mutation races.
- **Per-post-execve-immutability** — defense against per-shell-pipeline disruption via late setpgid.
- **Per-same-session-invariant** — defense against per-cross-session group escape.
- **Per-session-leader-locked** — defense against per-session-leader PGID change (would break job-control invariants).
- **Per-pid-ns-scoped** — defense against per-cross-ns process-group escape.

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at setpgid entry** — randomizes kernel stack offset.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/stat`'s pgrp field is restricted; queries of other users' pgids are gated separately.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec chroot, setpgid cannot reference tasks/pgids outside the chroot pid-ns; find_task is scoped.
- **CAP_SETUID / CAP_SETGID strict** — setpgid is not a cred-changing syscall; capabilities not consulted.
- **GRKERNSEC_AUDIT_GROUP** — setpgid calls from audited users are logged with target pid and new pgid for forensic group-membership tracking (used to detect anti-debugging attempts to detach from group).
- **GRKERNSEC_CHROOT_NICE** — not directly relevant; setpgid is not nice/sched.
- **No-info-leak on EPERM/ESRCH** — grsec ensures the errno does not distinguish "target doesn't exist" from "target visible but in different session" beyond what POSIX requires; prevents probing for hidden processes.
- **GRKERNSEC_TPE / GRKERNSEC_TIOCSTI** — orthogonal; setpgid does not bypass these.
- **Job-control consistency** — grsec ensures setpgid + tcsetpgrp + signal-delivery pipeline maintains foreground-group invariants; no race to escape a foreground SIGINT.
- **No LSM hook** — setpgid has no security hook; grsec adds audit + ns-scope checks only.
- **PAX_USERCOPY honored** — setpgid takes scalars, no user buffer; not directly applicable but kernel-stack canaries protect the syscall frame.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `getpgid(2)` / `getpgrp(2)` (Tier-5 separate docs — read PGID).
- `setsid(2)` (Tier-5 separate doc — create session).
- `tcsetpgrp(3)` / `tcgetpgrp(3)` (Tier-3 in `drivers/tty/job-control.md`).
- Signal-to-pgroup delivery (`kill(-pgid, ...)`) (Tier-5 in `kill.md`).
- Implementation code.
