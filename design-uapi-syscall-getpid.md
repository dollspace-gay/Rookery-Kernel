---
title: "Tier-5 syscall: getpid(2) — syscall 39"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`getpid(2)` returns the calling thread's **thread-group ID** (TGID) — what userspace programs call "the process ID" — as observed from the caller's PID namespace. It is the most-called metadata syscall on a Linux system after `gettimeofday`; glibc historically cached its result across calls, then dropped the cache after `clone(CLONE_PARENT)` and `fork()` ambiguities (2.25+). The kernel implementation is a single dereference: `current->signal->tgid` mapped through `task_active_pid_ns(current)` to obtain the namespace-relative virtual PID.

Critical for: every `getpid()`-using program, `printf("%d", getpid())` boilerplate, PID-namespace-isolated containers, glibc `__getpid` wrapper, audit subsystem caller-identification, signal-self delivery (`kill(getpid(), ...)`), `/proc/self` resolution, fork-safety state in libc heap arenas.

### Acceptance Criteria

- [ ] AC-1: `getpid()` from a single-threaded process returns its TGID (same as `/proc/self/status:Tgid`).
- [ ] AC-2: All threads in a multi-threaded program return the same value from `getpid()`.
- [ ] AC-3: `gettid() != getpid()` for all non-leader threads; equal for the leader.
- [ ] AC-4: After `unshare(CLONE_NEWPID); fork()`, child's `getpid() == 1`.
- [ ] AC-5: After `unshare(CLONE_NEWPID); fork()`, outer-ns reader's `/proc/<child>/status:Tgid` shows outer-ns PID; inner observes 1.
- [ ] AC-6: PID 1 of host namespace always returns 1.
- [ ] AC-7: `execve` does not change the reported PID.
- [ ] AC-8: Reading `/proc/self/status` shows `Tgid:` matching `getpid()`.
- [ ] AC-9: Strace-traced `getpid` returns the same value as the syscall return register.
- [ ] AC-10: `getpid()` returns the same value across 1e6 calls within one task (no reallocation/migration).
- [ ] AC-11: `getpid()` never returns 0 or a negative number.
- [ ] AC-12: Concurrent `getpid()` from N threads of the same process all return the same TGID.

### Architecture

```rust
#[syscall(nr = 39, abi = "sysv")]
pub fn sys_getpid() -> pid_t {
    Getpid::do_getpid()
}
```

`Getpid::do_getpid() -> pid_t`:
1. let task = current();
2. /* Look up the TGID-typed `struct pid` for the task's thread group leader */
3. let pid = task_tgid(task);
4. /* Translate to the caller's active PID namespace */
5. let ns = task_active_pid_ns(task);
6. /* Walk pid.numbers[] to find the entry where upid.ns == ns */
7. let vnr = Pid::vnr_in_ns(pid, ns);
8. /* Invariant: ns level is ≤ pid.level by construction */
9. debug_assert!(vnr >= 1);
10. vnr

`Pid::vnr_in_ns(pid, ns) -> pid_t`:
1. /* numbers[] is ordered root → leaf; level field caches depth */
2. if ns.level > pid.level { return 0; }   // unreachable for current task
3. pid.numbers[ns.level].nr

`task_tgid(task) -> *struct pid`:
1. /* TGID lookup walks PIDTYPE_TGID hlist link on signal->pids[] */
2. task.signal.pids[PIDTYPE_TGID]

`task_active_pid_ns(task) -> *pid_namespace`:
1. /* The active PID namespace is the namespace at the TASK's pid level */
2. let leader_pid = task.thread_pid;   // PIDTYPE_PID
3. leader_pid.numbers[leader_pid.level].ns

### Out of Scope

- `gettid(2)` (Tier-5 separate doc — returns kernel PID per-thread).
- `getppid(2)` (Tier-5 separate doc — returns parent's TGID).
- PID namespace lifecycle and `clone(CLONE_NEWPID)` semantics (Tier-3 in `kernel/pid_namespace.md`).
- PID allocator (`alloc_pid`, IDR) (Tier-3 in `kernel/pid.md`).
- `/proc/self` resolution (Tier-3 in `fs/proc/self.md`).
- Implementation code.

### signature

```c
pid_t getpid(void);
```

### parameters

(none — `SYSCALL_DEFINE0(getpid)`)

### return value

| Value | Meaning |
|---|---|
| `pid_t` ≥ 1 | The calling task's TGID in its active PID namespace. |

`getpid()` cannot fail. No error path exists: the kernel always has a valid `current` task with a non-zero TGID. POSIX-mandated: never returns `-1`, never sets `errno`.

### errors

(none)

### abi surface

```text
__NR_getpid (x86_64)    = 39
__NR_getpid (i386)      = 20
__NR_getpid (arm64)     = 172  (generic-syscall via __NR3264_getpid alias)
__NR_getpid (generic)   = 172

typedef __kernel_pid_t  pid_t;  /* int — 32-bit signed */

PID_MAX_LIMIT_64  = 4 * 1024 * 1024   /* 4M PIDs on 64-bit */
PID_MAX_LIMIT_32  = 32 * 1024         /* 32K PIDs on 32-bit; rare */
DEFAULT_PID_MAX   = 32768

struct pid {
    refcount_t          count;
    unsigned int        level;          /* namespace nesting depth */
    spinlock_t          lock;
    struct hlist_head   tasks[PIDTYPE_MAX];   /* PIDTYPE_PID, _TGID, _PGID, _SID */
    struct upid         numbers[];       /* one per ns level */
};

struct upid {
    int                 nr;              /* virtual PID at this level */
    struct pid_namespace *ns;
};
```

### compatibility contract

REQ-1: Syscall number is **39** on x86_64; **172** on arm64 / generic. ABI-stable since the earliest Linux.

REQ-2: Return value is the caller's **thread-group leader's PID** in `task_active_pid_ns(current)`, NOT the kernel-global PID.

REQ-3: All threads of a process return the same value (the TGID). Distinct from `gettid(2)` which returns per-thread `pid` (the kernel PID, i.e. `current->pid`).

REQ-4: After `unshare(CLONE_NEWPID)` + `fork()`, the child observes `getpid() == 1` (PID 1 of the new namespace). The parent in the outer namespace still observes the child as `tgid_in_outer_ns`.

REQ-5: `getpid()` of pid-1 of any PID namespace returns 1. Multiple processes can return 1 system-wide.

REQ-6: After `setns(CLONE_NEWPID)`: subsequent `fork()` children see the new namespace; the calling task itself does NOT change its active PID namespace (setns to PIDNS only affects descendants). `getpid()` for the calling task continues to return its TGID in the OLD namespace.

REQ-7: After `execve(2)`: TGID is preserved. Thread group is reset (other threads die, leader inherits leader's old `pid`). `getpid()` returns the same value pre- and post-exec.

REQ-8: After `vfork(2)` parent stall: parent and child have distinct TGIDs; child's TGID is allocated at vfork-time; parent's `getpid()` is unchanged.

REQ-9: `getpid()` is async-signal-safe and re-entrant. Kernel implementation holds NO locks (RCU is implicit through `current`).

REQ-10: `getpid()` MUST NOT be cached across `fork()` / `clone()` boundaries by libc (since glibc 2.25). The kernel guarantees that two distinct tasks never observe the same TGID at the same time, but PID reuse occurs over the lifetime of the namespace.

REQ-11: `current->signal->tgid` is set at `clone()`/`fork()` time and never changes thereafter for that task. The mapping to a virtual PID is constant for the lifetime of the task within its namespace.

REQ-12: `getpid()` is NOT affected by `prctl(PR_SET_NAME)`, `prctl(PR_SET_DUMPABLE)`, capability changes, `seccomp`, or any LSM hook. No LSM hook fires.

REQ-13: When `current` is a kernel thread (`PF_KTHREAD`), `getpid()` returns the kthread's TGID. Userspace never calls this path (kthreads cannot enter the syscall layer), but the kernel path is well-defined.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `never_returns_zero` | INVARIANT | per-getpid: return ≥ 1 for any valid `current`. |
| `tgid_stable_post_clone` | INVARIANT | per-task lifetime: TGID does not change after task creation. |
| `ns_level_in_bounds` | INVARIANT | per-vnr lookup: ns.level ≤ pid.level. |
| `no_state_mutation` | INVARIANT | per-getpid: read-only; no task/pid/ns field written. |
| `rcu_implicit_safety` | INVARIANT | per-current deref: current is always valid in syscall context. |
| `thread_group_equality` | INVARIANT | per-task: all threads sharing signal_struct return same TGID. |

### Layer 2: TLA+

`kernel/getpid.tla`:
- States: per-task pid-namespace map, TGID assignment table.
- Properties:
  - `safety_pid_unique_per_ns` — per-namespace: TGID is unique among live tasks.
  - `safety_pid_one_for_init` — per-namespace: pid-1 task returns 1.
  - `safety_no_mutation` — per-getpid: no state transition.
  - `liveness_returns` — per-getpid: terminates in O(1) steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getpid` post: ret == task.signal.tgid mapped through task_active_pid_ns | `Getpid::do_getpid` |
| `vnr_in_ns` post: returns pid.numbers[ns.level].nr | `Pid::vnr_in_ns` |
| `task_active_pid_ns` post: returns the ns associated with task.thread_pid at its level | `task_active_pid_ns` |

### Layer 4: Verus / Creusot functional

Per-`getpid(2)` man-page equivalence. LTP `getpid01..getpid02` pass. POSIX.1-2008 `getpid` semantics verified.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getpid(2)` reinforcement:

- **Per-RCU-implicit current** — defense against per-stale-task-pointer use-after-task-exit (impossible by construction, but verified).
- **Per-ns-level-bounds check** — defense against per-malformed pid_namespace hierarchy.
- **Per-readonly semantics** — defense against per-mutation-via-getpid bug pattern.
- **Per-never-zero return** — defense against per-userspace assumption violation (POSIX guarantee).

### grsecurity / pax-style reinforcement

- **PAX_RANDKSTACK at getpid entry** — randomizes kernel stack offset per call; defeats per-syscall stack-layout fingerprinting that uses high-frequency `getpid()` as a probe.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/status` Tgid line is consistent with `getpid()` but the existence of `<pid>` is hidden from non-owners; `getpid()` of self always permitted (you cannot hide your own PID from yourself).
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec-hardened chroot, `getpid()` is unaffected (returns own TGID), but `/proc/<other-pid>/status` is filtered so userspace cannot correlate `getpid()` of self with sibling PIDs outside the chroot's task set.
- **PID namespace isolation reinforced** — `getpid()` MUST honor `task_active_pid_ns(current)`; grsec adds an assertion that the returned vnr is ≥ 1 and ≤ pid_max for the namespace.
- **Anti-fingerprint hardening** — combined with PID-randomization (PID-randomize policy), high-volume `getpid()` reconnaissance does not reveal PID-allocation ordering.
- **No CAP requirement** — `getpid()` is unprivileged by POSIX; grsec does NOT add a cap gate, but it DOES audit (optionally) sequences of `getpid()` from suspicious tasks via GRKERNSEC_AUDIT_GROUP if the task is marked.
- **No_new_privs neutral** — `getpid()` is read-only; NNP has no effect on its semantics.
- **KEEPCAPS neutral** — capability preservation flags are unrelated to PID reporting.

