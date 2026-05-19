# Tier-5 syscall: gettid(2) — syscall 186

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sys.c (SYSCALL_DEFINE0(gettid))
  - include/linux/sched.h (struct task_struct.pid)
  - include/linux/pid.h (task_pid_vnr, pid_vnr)
  - include/linux/pid_namespace.h (struct pid_namespace)
  - arch/x86/entry/syscalls/syscall_64.tbl (186  common  gettid)
-->

## Summary

`gettid(2)` returns the calling **thread's** thread ID (TID) — i.e. the kernel's per-task identifier (`task_struct.pid`) translated into the caller's pid-namespace. Distinct from `getpid(2)`, which returns the thread-group leader's pid (TGID, the POSIX "process ID"). For a single-threaded process: `gettid() == getpid()`. For a multi-threaded process: every thread has a unique TID, but all share the same TGID.

Critical for: per-thread accounting (`/proc/<pid>/task/<tid>/`), `tgkill(2)` target selection, `set_robust_list(2)` per-thread state, `pidfd_open(2)` of a sibling thread, glibc's NPTL TLS-bookkeeping (`pthread_self()` → cached TID), perf-event per-thread profiling, signal targeting (`SIGEV_THREAD_ID`), audit/log thread-correlation, futex thread-ownership recording (`FUTEX_OWNER_DIED`).

## Signature

```c
pid_t gettid(void);
```

glibc has historically NOT exposed `gettid()` as a wrapper (callers used `syscall(SYS_gettid)`); glibc 2.30+ adds a real wrapper.

## Parameters

(none — `SYSCALL_DEFINE0(gettid)`)

## Return value

| Value | Meaning |
|---|---|
| `pid_t` | The calling thread's TID mapped into `current->nsproxy->pid_ns_for_children` (caller's pid-ns). |

`gettid()` cannot fail. POSIX-mandated: never returns `-1`, never sets `errno`.

## Errors

(none)

## ABI surface

```text
__NR_gettid (x86_64)   = 186
__NR_gettid (i386)     = 224
__NR_gettid (arm64)    = 178     /* generic */
__NR_gettid (generic)  = 178

typedef __kernel_pid_t  pid_t;   /* signed int — 32-bit; positive on success */

struct task_struct {
    pid_t              pid;        /* TID — kernel-global */
    pid_t              tgid;       /* thread-group leader's pid (POSIX getpid) */
    struct pid        *thread_pid; /* per-thread pid (with ns levels) */
    struct nsproxy    *nsproxy;
    ...
};

/* Translation: kernel pid → user-visible pid in caller's pid-ns:
 *   pid_t = task_pid_vnr(task)
 *         = pid_nr_ns(task->thread_pid, task_active_pid_ns(current))
 */
```

## Compatibility contract

REQ-1: Syscall number is **186** on x86_64; **178** on arm64 / generic. ABI-stable since Linux 2.4.11.

REQ-2: Returns `task_pid_vnr(current)`. The "vnr" (virtual nr) form returns the TID as seen from the caller's active pid-namespace.

REQ-3: For a single-threaded process: `gettid() == getpid()` always. For the thread-group leader of any process: `gettid() == getpid()`.

REQ-4: For a non-leader thread: `gettid() != getpid()`; `gettid()` is unique within the pid-namespace; `getpid()` returns the leader's TID.

REQ-5: TIDs are allocated from the same pool as PIDs; after fork()/clone(): the child's TID is one TID-slot beyond the parent (modulo wrap-around at `/proc/sys/kernel/pid_max`).

REQ-6: TIDs are recycled after the task fully exits and is reaped (`wait*()` or `pidfd_send_signal()` keepalive ends). pid-namespace prevents cross-ns recycling collisions.

REQ-7: Inside a pid-namespace nested at level `L`: `gettid()` returns the TID at level `L` (the caller's view). The kernel-global pid (level 0) is NOT visible.

REQ-8: `gettid()` is async-signal-safe and re-entrant. No locks held; only an RCU read of `current->thread_pid`.

REQ-9: No LSM hook on the gettid path. The returned TID is process-public metadata.

REQ-10: `gettid()` does NOT change credentials, scheduling, or any task state. Pure read.

REQ-11: After `setns(2)` into a different pid-ns: subsequent `gettid()` calls in the original task return the same TID **as seen from the NEW active pid-ns**; if the task's thread_pid has no entry at the new ns level, the call returns 0 (per pid_nr_ns semantics for "not visible").

REQ-12: `gettid()` of pid 1 in a pid-ns returns 1.

REQ-13: Concurrent `setns()` does not perturb the returned TID for the calling thread (which itself does not migrate to a new pid-ns; only children would).

REQ-14: `gettid()` and `getpid()` are guaranteed consistent: after the syscall returns, the TID/TGID pair was a valid pairing for `current` at some point during the syscall.

REQ-15: glibc's `pthread_self()` caches the TID returned by `gettid()` at thread creation in TLS; subsequent `pthread_self()` calls do NOT re-invoke `gettid()`.

## Acceptance Criteria

- [ ] AC-1: Single-threaded process: `gettid() == getpid()`.
- [ ] AC-2: Thread-group leader: `gettid() == getpid()`.
- [ ] AC-3: Non-leader thread created via `clone(CLONE_THREAD)`: `gettid() != getpid()`.
- [ ] AC-4: All threads in a process have unique `gettid()` values.
- [ ] AC-5: After fork(): child's `gettid()` is a fresh TID, not the parent's.
- [ ] AC-6: Inside a pid-namespace: `gettid()` of pid-1-of-ns returns 1, not its host-ns TID.
- [ ] AC-7: `gettid()` is consistent with `/proc/self/task/<tid>` existence.
- [ ] AC-8: `gettid()` matches `/proc/<pid>/status:Pid` (NOT `Tgid`).
- [ ] AC-9: `gettid()` returns same value across repeated calls in same thread (TID never mutates).
- [ ] AC-10: After thread exit and PID recycling, a new task may receive the recycled TID — but never within the same `wait()`-reapable window.
- [ ] AC-11: `gettid()` never returns `<= 0` (TIDs are positive).
- [ ] AC-12: Concurrent threads calling `gettid()`: each returns its own TID atomically.

## Architecture

```rust
#[syscall(nr = 186, abi = "sysv")]
pub fn sys_gettid() -> pid_t {
    Gettid::do_gettid()
}
```

`Gettid::do_gettid() -> pid_t`:
1. let task = current();
2. /* RCU-read of thread_pid */
3. let tid = rcu::read(|| {
4.     let pid = task.thread_pid;            // *struct pid
5.     let active_ns = task_active_pid_ns(task); // current's pid-ns
6.     PidNs::pid_nr_ns(pid, active_ns)
7. });
8. /* Invariant: tid > 0 for visible task */
9. tid

`PidNs::pid_nr_ns(pid, ns) -> pid_t`:
1. /* Walk pid->numbers[] array for the entry matching ns->level */
2. if ns.level > pid.level: return 0; // not visible at this depth
3. let upid = pid.numbers[ns.level];
4. if upid.ns != ns: return 0;
5. return upid.nr;

`task_active_pid_ns(task) -> *pid_namespace`:
1. task.nsproxy.pid_ns_for_children
2. (or task.thread_pid.numbers[task.thread_pid.level].ns for the task's own ns)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tid_positive` | INVARIANT | per-gettid: returns `> 0` for live task. |
| `tid_matches_thread_pid` | INVARIANT | per-gettid: ret == pid_nr_ns(current.thread_pid, current's pid-ns). |
| `tid_distinct_per_thread` | INVARIANT | per-process: all threads' TIDs are pairwise distinct. |
| `tgid_leader_equals_tid` | INVARIANT | per-thread-group leader: gettid() == getpid(). |
| `rcu_atomic` | INVARIANT | per-gettid: concurrent setns yields consistent pid-ns snapshot. |
| `no_state_mutation` | INVARIANT | per-gettid: read-only; no task/cred field written. |

### Layer 2: TLA+

`kernel/gettid.tla`:
- States: per-task TID, TGID, pid-ns hierarchy.
- Properties:
  - `safety_tid_unique_in_ns` — within a pid-ns, no two live tasks share a TID.
  - `safety_leader_eq_tgid` — leader of thread group has TID == TGID.
  - `safety_no_mutation` — gettid never mutates state.
  - `liveness_returns` — gettid terminates in O(pid-ns-depth) ≤ O(32).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_gettid` post: ret == pid_nr_ns(current.thread_pid, active_pid_ns) | `Gettid::do_gettid` |
| `pid_nr_ns` post: returns 0 iff pid not visible at ns.level | `PidNs::pid_nr_ns` |
| `task_active_pid_ns` post: returns task's own pid-ns | `PidNs::task_active_pid_ns` |

### Layer 4: Verus / Creusot functional

Per-`gettid(2)` man-page equivalence. LTP `gettid01`, `gettid02` pass. Linux kernel selftests `tools/testing/selftests/pid_namespace` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`gettid(2)` reinforcement:

- **Per-RCU snapshot** — defense against per-torn-pid-read race with concurrent setns / pid-ns destruction.
- **Per-pid-ns-translation** — defense against per-namespace-leak (kernel-global pid would otherwise leak across ns boundaries).
- **Per-readonly semantics** — defense against per-mutation-via-gettid bug pattern.
- **Per-positive return invariant** — defense against per-TID-zero confusion (zero used as "not visible" sentinel).

## Grsecurity / PaX-style Reinforcement

- **PAX_RANDKSTACK at gettid entry** — randomizes kernel stack offset; defeats stack-layout fingerprinting via high-frequency `gettid()` probes.
- **GRKERNSEC_PROC_USERGROUP info-leak protection** — `/proc/<pid>/task/<tid>/` is restricted to owning user and CAP_SYS_ADMIN; `gettid()` of self always succeeds and never leaks sibling-task identity.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a grsec chroot, `gettid()` reflects only the chroot-pid-ns view; kernel-global pid is not leaked across chroot boundary; `find_task_by_pid_ns` rejects cross-ns lookups.
- **CAP_SETUID / CAP_SETGID strict** — gettid is a pure read and does not interact with cred-changing capabilities; grsec does not gate it.
- **GRKERNSEC_AUDIT_GROUP** — repeated `gettid()` enumeration from audited users is logged to detect thread-discovery reconnaissance (e.g. attacker scanning thread space).
- **GRKERNSEC_HIDESYM** — even with `gettid()` known, grsec hides kernel symbols (e.g. stack addresses tied to a TID) from /proc/<pid>/stack and kallsyms.
- **PAX_RANDUSTACK** — per-thread user stack randomization keyed off TID-creation order is preserved; `gettid()` does not leak entropy.
- **No LSM hook** — gettid has no security hook; grsec adds optional audit but never blocks.
- **pid_max enforcement** — grsec sysctl `kernel.pid_max` clamp prevents pid_max being raised so high that TID wraparound becomes a security issue.
- **Thread-info leak resistance** — `gettid()` returns only the TID; no kernel-pointer, no task_struct address, no scheduling info is exposed.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `getpid(2)` (Tier-5 separate doc — returns TGID).
- `getppid(2)` (Tier-5 separate doc).
- `tgkill(2)` / `tkill(2)` (Tier-5 separate docs — send signal to specific TID).
- `clone(2)` / `clone3(2)` thread creation semantics (Tier-5 separate docs).
- pid-namespace internals (Tier-3 in `kernel/pid_namespace.md`).
- Implementation code.
