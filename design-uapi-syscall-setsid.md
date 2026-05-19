---
title: "Tier-5 syscall: setsid(2) — syscall 112"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`setsid(2)` creates a new **session** and a new **process group**, both led by the calling process. After a successful call: `getsid(0) == getpgrp() == getpid()`, the calling process has NO controlling terminal, and any prior controlling-tty association is dropped.

The canonical use is the **daemon double-fork**: parent → fork → child calls `setsid()` to detach from the parent's session and tty, then fork()s once more so the grandchild can never re-acquire a controlling tty (only session leaders can). Daemons rely on this for full detachment from interactive sessions.

`setsid()` fails with EPERM if the caller is already a process-group leader — that's because making the caller a session leader would require it to also become a new process-group leader, but it already leads a different group, which would orphan that group's other members.

Critical for: every daemon (`systemd`, `sshd`, `cron`, `init`), every container init that detaches from host tty, every shell that creates job-control session (`tmux new-session`, `screen`, login shells via PAM `pam_loginuid`), every `setsid(1)` userspace wrapper.

### Acceptance Criteria

- [ ] AC-1: Non-pgid-leader caller: `setsid()` returns a positive SID == caller's PID.
- [ ] AC-2: After successful setsid: `getsid(0) == getpgrp() == getpid()`.
- [ ] AC-3: Pgid leader caller: `setsid()` returns -1, errno == EPERM.
- [ ] AC-4: After successful setsid: caller has no controlling tty.
- [ ] AC-5: After successful setsid: writes to (formerly-controlling) tty no longer generate SIGTTOU.
- [ ] AC-6: Forked child after setsid: inherits new SID and PGID.
- [ ] AC-7: SIGHUP to session leader propagates to all session members (per kernel hangup path).
- [ ] AC-8: setsid in a pid-ns: SID confined to pid-ns; outer ns sees the host-mapped value via outer-ns getsid.
- [ ] AC-9: Concurrent setsid from sibling thread: serialized by tasklist_lock; only one wins.
- [ ] AC-10: After setsid: `/proc/<pid>/stat` session field == caller's pid.
- [ ] AC-11: Daemon double-fork idiom (fork → setsid → fork): grandchild has no controlling tty and cannot acquire one without explicit TIOCSCTTY.

### Architecture

```rust
#[syscall(nr = 112, abi = "sysv")]
pub fn sys_setsid() -> SyscallResult<pid_t> {
    Setsid::do_setsid()
}
```

`Setsid::do_setsid() -> Result<pid_t>`:
1. let task = current();
2. let group_leader = task.group_leader;
3. /* Take tasklist write-lock for atomicity vs. fork/exit */
4. write_lock(&tasklist_lock);
5. /* Refuse if already pgid leader */
6. if task_pgrp(group_leader) == task_pid(group_leader) {
7.     write_unlock(&tasklist_lock);
8.     return Err(Errno::EPERM);
9. }
10. /* Attach as PGID + SID leader */
11. set_special_pids(group_leader, task_pid(group_leader));
12. group_leader.signal.leader = 1;
13. /* Drop controlling tty atomically */
14. proc_clear_tty(group_leader);
15. let sid = pid_vnr(task_session(group_leader));
16. write_unlock(&tasklist_lock);
17. Ok(sid)

`set_special_pids(task, new_pid)`:
1. change_pid(task, PIDTYPE_PGID, new_pid);
2. change_pid(task, PIDTYPE_SID,  new_pid);

`proc_clear_tty(task)`:
1. spin_lock(&task.sighand.siglock);
2. let tty = task.signal.tty;
3. task.signal.tty = NULL;
4. spin_unlock;
5. if tty: tty_kref_put(tty);

### Out of Scope

- `getsid(2)` (Tier-5 separate doc).
- `setpgid(2)` / `getpgid(2)` / `getpgrp(2)` (Tier-5 separate docs).
- Controlling-terminal acquisition via TIOCSCTTY (Tier-3 in `drivers/tty/job-control.md`).
- Job-control signal delivery (SIGTSTP, SIGTTIN, SIGTTOU) (Tier-3 in `kernel/signal.md`).
- Implementation code.

### signature

```c
pid_t setsid(void);
```

### parameters

(none — `SYSCALL_DEFINE0(setsid)`)

### return value

| Value | Meaning |
|---|---|
| `> 0` | The new SID (equal to the caller's PID, mapped into caller's pid-ns). |
| `-1` | Error; `errno` set. |

### errors

| errno  | condition                                                              |
|--------|------------------------------------------------------------------------|
| `EPERM` | The calling process is already a process-group leader. (i.e. PID == PGID for current.) |

### abi surface

```text
__NR_setsid (x86_64)   = 112
__NR_setsid (i386)     = 66
__NR_setsid (arm64)    = 157     /* generic */
__NR_setsid (generic)  = 157

struct signal_struct {
    ...
    struct pid *pids[PIDTYPE_MAX];   /* PIDTYPE_TGID, _PGID, _SID */
    struct tty_struct *tty;          /* controlling tty (NULL after setsid) */
    int leader;                      /* 1 if session leader */
    ...
};
```

### compatibility contract

REQ-1: Syscall number is **112** on x86_64; **157** on arm64 / generic. ABI-stable since Linux 1.0.

REQ-2: If `current` is already a process-group leader (i.e. `task_pid(current) == task_pgrp(current)`): return `-EPERM`.

REQ-3: Otherwise: attach `current` as both PGID and SID leader:
- `change_pid(current, PIDTYPE_PGID, task_pid(current))`
- `change_pid(current, PIDTYPE_SID,  task_pid(current))`
- `current.signal.leader = 1`
- `disassociate_ctty(1)` — drop controlling tty atomically.

REQ-4: Returns the new SID as `pid_vnr(task_session(current))` = caller's PID in caller's pid-ns.

REQ-5: After `setsid()`: any subsequent `open(2)` of a tty by the calling process does NOT make that tty the controlling tty unless `O_NOCTTY` is omitted AND `TIOCSCTTY` is issued. The default `open()` of `/dev/tty*` only acquires controlling-tty status for session leaders **without** a controlling tty.

REQ-6: All other threads in the calling thread group continue to share `signal_struct`; setsid affects the entire thread group's SID/PGID. POSIX requires the caller to be single-threaded for setsid to be portable, but Linux permits multi-threaded callers.

REQ-7: `setsid()` does not affect child processes that have already been forked. Forked children of caller after setsid inherit the new SID.

REQ-8: After fork(): child inherits parent's SID and PGID; the child does NOT become a session leader; only an explicit `setsid()` call makes a process a session leader.

REQ-9: Failure case (EPERM): typical when caller invokes `setpgid()` on itself first and becomes pgid-leader, or when called from a shell's process-group leader. Standard daemon idiom uses fork() before setsid() precisely because the child of fork() is never a pgid leader.

REQ-10: `setsid()` is NOT async-signal-safe due to the disassociate_ctty path; it is a syscall-level operation taking `tasklist_lock` (write) and the `tty->ctrl.lock`.

REQ-11: No LSM hook on setsid path. The syscall is purely a process-state mutation.

REQ-12: After `setsid()`, sending SIGHUP to the **new** session leader propagates to all tasks in the session (per POSIX kernel-side semantics during session-leader exit/hangup).

REQ-13: Inside a pid-namespace: returned SID is in the pid-ns; the session is fully scoped within the pid-ns (sessions do not span pid-namespaces).

REQ-14: Returns the new SID on success; cannot return 0.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pgid_leader_returns_eperm` | INVARIANT | per-setsid: caller is pgid leader ⟹ -EPERM. |
| `success_sets_sid_and_pgid` | INVARIANT | per-setsid success: PIDTYPE_SID and PIDTYPE_PGID both := task_pid. |
| `controlling_tty_cleared` | INVARIANT | per-setsid success: signal.tty := NULL. |
| `tasklist_lock_held` | INVARIANT | per-setsid: tasklist write-lock held during mutation. |
| `leader_flag_set` | INVARIANT | per-setsid success: signal.leader := 1. |
| `return_eq_new_sid` | INVARIANT | per-setsid success: ret == pid_vnr(task_session(current)). |

### Layer 2: TLA+

`kernel/setsid.tla`:
- States: per-task SID, PGID, leader-flag, controlling-tty.
- Properties:
  - `safety_pgid_leader_blocked` — pre-state pgid-leader: setsid fails with EPERM.
  - `safety_atomic_state_transition` — SID + PGID + tty cleared atomically.
  - `safety_new_sid_eq_pid` — post-setsid: SID == PID.
  - `liveness_terminates` — setsid completes (no deadlock).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_setsid` pre: pgid != pid | `Setsid::do_setsid` |
| `do_setsid` post (success): SID == PGID == PID; tty == NULL; leader == 1 | `Setsid::do_setsid` |
| `do_setsid` post (failure): no mutation | `Setsid::do_setsid` |
| `set_special_pids` post: pid attached at PGID and SID type | `Setsid::set_special_pids` |

### Layer 4: Verus / Creusot functional

Per-`setsid(2)` man-page equivalence. LTP `setsid01`, `setsid02`, `setsid03` pass. POSIX.1-2008 `setsid` semantics verified. Daemon-idiom integration tests via `setsid(1)` and `nohup(1)` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`setsid(2)` reinforcement:

- **Per-tasklist-lock write** — defense against per-concurrent-fork/exit race during SID/PGID transition.
- **Per-controlling-tty atomic drop** — defense against per-stale-tty-pointer use after session detach.
- **Per-EPERM-on-pgid-leader** — defense against per-orphaned-process-group bug pattern.
- **Per-pid-ns-scoped** — defense against per-cross-ns session escape (setsid cannot create a session spanning pid-ns).
- **Per-leader-flag explicit** — defense against per-session-membership confusion (leader vs. member distinguished).

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at setsid entry** — randomizes kernel stack offset; defeats stack-layout fingerprinting probes.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — after setsid, `/proc/<pid>/stat`'s session/pgrp/tpgid fields are protected from sibling-user observation; only the owning user and CAP_SYS_ADMIN can read.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — setsid within a grsec chroot creates a session strictly confined to chroot's pid-ns; controlling tty (if any) inside chroot is dropped and cannot be reacquired across chroot boundary.
- **CAP_SETUID / CAP_SETGID strict** — setsid is not a cred-changing syscall; capabilities are not consulted; grsec does not gate on uid/gid caps.
- **GRKERNSEC_AUDIT_GROUP on cred-change** — setsid is audited as a session-creation event for audited users; the new SID is logged alongside the caller's uid/gid for forensic correlation.
- **GRKERNSEC_TIOCSTI** — separately enforced: after setsid, the new session leader cannot use TIOCSTI to inject input into another session's tty. setsid does NOT bypass this.
- **GRKERNSEC_CHROOT_CAPS** — setsid does not grant new capabilities; the post-setsid task retains its pre-setsid caps (possibly restricted by chroot caps policy).
- **PAX_NOEXEC** — setsid does not allocate executable memory; defense in depth.
- **No new privs honored** — setsid is not affected by PR_SET_NO_NEW_PRIVS; orthogonal to no_new_privs.
- **Daemon double-fork friendly** — grsec hardens but does not break the canonical fork → setsid → fork daemon idiom; setsid is permitted in the post-fork window.
- **Session-hangup propagation safe** — SIGHUP propagation on session-leader exit is gated by grsec's signal-policy hooks; cross-uid SIGHUP from a session-leader exit is permitted (POSIX-required) but logged.

