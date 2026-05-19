---
title: "Tier-5 syscall: getppid(2) — syscall 110"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getppid(2)` returns the **parent process ID** of the calling task — specifically, the TGID of `current->real_parent`, mapped through the caller's active PID namespace. After the parent dies, the kernel re-parents the calling task to a subreaper (or PID-1 of the namespace), and `getppid()` returns the new parent's PID — observable proof that orphan reparenting has occurred.

Critical for: orphan detection (`getppid() == 1` idiom), `PR_SET_PDEATHSIG` parent-death notification, daemonization sequences, `init` / `systemd` reaper logic, `PR_SET_CHILD_SUBREAPER` subreaper testing, container PID-1 semantics, double-fork daemon trick (`waitpid(child); child2's getppid() == 1`).

### Acceptance Criteria

- [ ] AC-1: Child of `fork()` reports `getppid() == parent_pid`.
- [ ] AC-2: After parent exits while child runs, child's `getppid()` becomes 1 (in root PIDNS, absent subreapers).
- [ ] AC-3: PID-1 of host namespace returns `getppid() == 0`.
- [ ] AC-4: `unshare(CLONE_NEWPID); fork()` — child observes `getppid() == 0` (parent invisible in inner ns).
- [ ] AC-5: `PR_SET_CHILD_SUBREAPER` ancestor: grandchild whose parent dies is reparented to subreaper, not pid-1.
- [ ] AC-6: `ptrace(PTRACE_ATTACH, child)` does NOT alter child's `getppid()` return.
- [ ] AC-7: All threads of a multi-threaded process return the same `getppid()` value.
- [ ] AC-8: `execve()` preserves `getppid()` return.
- [ ] AC-9: Double-fork daemon idiom: grandchild's `getppid() == 1` after intermediate exits.
- [ ] AC-10: `getppid()` never returns -1 or any value < 0.
- [ ] AC-11: Concurrent parent-exit + `getppid()`: result is consistent (pre- or post-reparent, never an inconsistent value).
- [ ] AC-12: PR_SET_PDEATHSIG handler observes `getppid() != original_parent` after SIGNAL delivery.

### Architecture

```rust
#[syscall(nr = 110, abi = "sysv")]
pub fn sys_getppid() -> pid_t {
    Getppid::do_getppid()
}
```

`Getppid::do_getppid() -> pid_t`:
1. let task = current();
2. /* RCU-protected read of real_parent */
3. let ppid = rcu::read(|| {
4.     let parent = task.real_parent;       // RCU-protected pointer
5.     /* Translate parent's TGID through THIS task's active pid namespace */
6.     let our_ns = task_active_pid_ns(task);
7.     task_tgid_nr_ns(parent, our_ns)      // returns 0 if not visible at our_ns level
8. });
9. /* Invariant: ppid ≥ 0 */
10. debug_assert!(ppid >= 0);
11. ppid

`task_tgid_nr_ns(parent, ns) -> pid_t`:
1. let pid = parent.signal.pids[PIDTYPE_TGID];
2. if ns.level > pid.level { return 0; }   // parent invisible at our level
3. pid.numbers[ns.level].nr

`task_active_pid_ns(task) -> *pid_namespace`:
1. let leader_pid = task.thread_pid;
2. leader_pid.numbers[leader_pid.level].ns

### Out of Scope

- `getpid(2)` (Tier-5 separate doc — returns own TGID).
- `prctl(PR_SET_PDEATHSIG)` (Tier-5 covered in `prctl.md`).
- `prctl(PR_SET_CHILD_SUBREAPER)` (Tier-5 covered in `prctl.md`).
- Reparenting machinery, `exit_notify`, `forget_original_parent` (Tier-3 in `kernel/exit.md`).
- PID namespace lifecycle (Tier-3 in `kernel/pid_namespace.md`).
- `ptrace(2)` parent-pointer manipulation (Tier-5 in `ptrace.md`).
- Implementation code.

### signature

```c
pid_t getppid(void);
```

### parameters

(none — `SYSCALL_DEFINE0(getppid)`)

### return value

| Value | Meaning |
|---|---|
| `pid_t` ≥ 0 | The PID of the calling task's current parent in `task_active_pid_ns(current)`. |

`getppid()` cannot fail. POSIX-mandated: never returns `-1`, never sets `errno`.

A return value of **0** indicates the parent is not visible in the caller's PID namespace (e.g., the caller is PID-1 of a child PID namespace whose creator lives outside).

### errors

(none)

### abi surface

```text
__NR_getppid (x86_64)    = 110
__NR_getppid (i386)      = 64
__NR_getppid (arm64)     = 173  (generic-syscall)
__NR_getppid (generic)   = 173

typedef __kernel_pid_t  pid_t;  /* int — 32-bit signed */

struct task_struct {
    ...
    struct task_struct __rcu *real_parent;   /* biological parent; survives ptrace */
    struct task_struct __rcu *parent;        /* current reporting parent; ptrace-modifiable */
    ...
};
```

### compatibility contract

REQ-1: Syscall number is **110** on x86_64; **173** on arm64 / generic. ABI-stable since the earliest Linux.

REQ-2: `getppid()` returns the TGID of `current->real_parent` (biological parent), NOT `current->parent` (which can be temporarily a tracer under `ptrace(2)`).

REQ-3: After parent exits, kernel `exit_notify()` reparents children:
  - First to the nearest ancestor with `PR_SET_CHILD_SUBREAPER` (signal->is_child_subreaper).
  - Else to PID-1 of the calling task's PID namespace.
  - `getppid()` after reparenting returns the NEW parent's PID.

REQ-4: For PID-1 of a PID namespace whose parent task lives in an enclosing PID namespace, `getppid()` returns 0 (parent invisible at the active level).

REQ-5: For PID-1 of the **root** PID namespace (the kernel-spawned init), `getppid() == 0` (no parent; `swapper`/PID-0 is not exposed).

REQ-6: Multi-threaded program: all threads return the same value (parent is per-thread-group via `signal_struct`, not per-thread).

REQ-7: `getppid()` is async-signal-safe and re-entrant. Implementation samples `real_parent` under RCU.

REQ-8: `setpgid(2)`, `setsid(2)`, `setpgrp(2)` do NOT change `real_parent`; `getppid()` is unaffected by session/process-group changes.

REQ-9: `ptrace(PTRACE_ATTACH)`: tracer becomes `current->parent`, but `real_parent` is unchanged → `getppid()` unchanged. On `PTRACE_DETACH`, `parent` reverts; `getppid()` still unchanged.

REQ-10: `execve(2)`: parent is preserved across exec. `getppid()` returns the same value pre- and post-exec.

REQ-11: After the calling task itself has been reparented (parent died, calling task re-parented to subreaper), `getppid()` returns the subreaper's TGID in the active namespace.

REQ-12: PR_SET_PDEATHSIG signal is delivered to the calling task on real_parent's exit — independently of subreaper reparenting. The sequence is observable: SIGNAL_PDEATH → next `getppid()` → reparented PID.

REQ-13: Concurrent reparent: `getppid()` reads under RCU; observes either pre- or post-reparent snapshot atomically. Never tears.

REQ-14: `getppid()` from a kernel thread returns the kthread parent (typically `kthreadd`'s TGID = 2 in root PIDNS). Userspace cannot reach this path.

REQ-15: Returns NEVER < 0. Returns ≥ 0 always. 0 is reserved for "not visible".

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `never_returns_negative` | INVARIANT | per-getppid: return ≥ 0 for any valid `current`. |
| `real_parent_followed` | INVARIANT | per-getppid: uses `real_parent`, not `parent`; ptrace transparent. |
| `rcu_atomic_snapshot` | INVARIANT | per-getppid: concurrent reparent yields consistent snapshot. |
| `ns_visibility_zero` | INVARIANT | per-getppid: parent invisible at our ns level ⟹ return 0. |
| `no_state_mutation` | INVARIANT | per-getppid: read-only; no task field written. |
| `thread_group_equality` | INVARIANT | per-thread-group: all threads return same value. |

### Layer 2: TLA+

`kernel/getppid.tla`:
- States: per-task real_parent map, reparent transitions, subreaper table.
- Properties:
  - `safety_reparent_invariant` — after parent-exit, getppid returns nearest live ancestor.
  - `safety_subreaper_priority` — subreaper preferred over pid-1.
  - `safety_ptrace_transparent` — ptrace attach does not alter getppid().
  - `safety_ns_isolation` — parent in outer ns ⟹ getppid() == 0 in inner ns.
  - `liveness_returns` — getppid terminates in O(1).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getppid` post: ret == task_tgid_nr_ns(task.real_parent, task_active_pid_ns(task)) | `Getppid::do_getppid` |
| `task_tgid_nr_ns` post: returns 0 iff ns.level > pid.level | `task_tgid_nr_ns` |
| `reparent` pre: target subreaper is alive at chosen time | `Exit::reparent_children` |

### Layer 4: Verus / Creusot functional

Per-`getppid(2)` man-page equivalence. LTP `getppid01..getppid02` pass. POSIX.1-2008 `getppid` semantics verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getppid(2)` reinforcement:

- **Per-RCU snapshot** — defense against per-torn-read race with concurrent parent-exit / reparent.
- **Per-real_parent (not parent)** — defense against per-ptrace confusion bug pattern.
- **Per-ns-visibility-zero** — defense against per-cross-ns PID leak.
- **Per-never-negative return** — defense against per-userspace assumption violation (POSIX guarantee).
- **Per-readonly semantics** — defense against per-mutation-via-getppid bug pattern.

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at getppid entry** — randomizes kernel stack offset per call; defeats stack-layout fingerprinting via repeated `getppid()` probes.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` PPid line is consistent with `getppid()` for the owning user; non-owners see hidden PIDs depending on policy. `getppid()` of self always works (you cannot hide your own parent from yourself, but the visibility of the parent in the namespace is governed by ns rules).
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec chroot, `getppid()` is restricted to parents that are also in the same chroot/task-set; otherwise returns 0 (rather than leak the existence of a parent outside the chroot).
- **PID namespace isolation reinforced** — parent in outer ns ⟹ return 0; grsec validates this invariant at every read.
- **GRKERNSEC_AUDIT_GROUP correlation** — high-frequency `getppid()` calls from audited groups are logged to detect orphan-detection / daemonization fingerprinting.
- **Anti-fingerprint hardening** — combined with PID-randomization, repeated `getppid()` does not leak parent-allocation timing relative to child.
- **No CAP requirement** — `getppid()` is unprivileged by POSIX; grsec does not gate.
- **No_new_privs neutral** — `getppid()` is read-only; NNP has no effect.
- **KEEPCAPS neutral** — capability preservation flags do not affect parent reporting.
- **Subreaper transparency** — `PR_SET_CHILD_SUBREAPER` reparenting is honored; grsec does not hide subreaper from `getppid()` (it is the legitimate parent post-reparent).
- **No LSM hook** — `getppid()` has no security hook; grsec adds an optional audit logger but never blocks the call.

