---
title: "Tier-5: syscall 252 — ioprio_get(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`ioprio_get(2)` is **x86_64 syscall 252**, the I/O scheduling-priority reader. It returns the current packed `ioprio` value (class + hint + data) for one or more tasks selected by `(which, who)`, mirroring the selector semantics of `ioprio_set(2)`:

- `IOPRIO_WHO_PROCESS` — exact pid (or 0 = current).
- `IOPRIO_WHO_PGRP` — entire process group; returns the **highest** (numerically lowest-class, lowest-data) priority found.
- `IOPRIO_WHO_USER` — all tasks of a real uid; returns the highest priority found.

If the selected task has not been explicitly assigned via `ioprio_set` (i.e., still in `IOPRIO_CLASS_NONE`), the kernel synthesizes a return value derived from the task's nice level: `IOPRIO_CLASS_BE` with `data = (task_nice + 20) / 5`, clamped to `[0,7]`. This is read-only and lossless — it does not promote the task into BE.

Critical for: every `ionice -p`/`-P` user, every container introspection tool reading I/O class of workers, every observability dashboard surfacing per-task I/O priority, every Rookery test verifying the "highest priority in set" aggregation and the nice-derived synthesis path.

### Acceptance Criteria

- [ ] AC-1: After `ioprio_set(IOPRIO_WHO_PROCESS, 0, IOPRIO_PRIO_VALUE(BE, 4))`, `ioprio_get(IOPRIO_WHO_PROCESS, 0)` ⟹ `IOPRIO_PRIO_VALUE(BE, 4)`.
- [ ] AC-2: Task at `IOPRIO_CLASS_NONE` with nice=0 ⟹ `ioprio_get` ⟹ `IOPRIO_PRIO_VALUE(BE, 4)` (synthesized).
- [ ] AC-3: Task at `IOPRIO_CLASS_NONE` with nice=-20 ⟹ `IOPRIO_PRIO_VALUE(BE, 0)`.
- [ ] AC-4: Task at `IOPRIO_CLASS_NONE` with nice=19 ⟹ `IOPRIO_PRIO_VALUE(BE, 7)`.
- [ ] AC-5: `which = 99` ⟹ `-EINVAL`.
- [ ] AC-6: `IOPRIO_WHO_PROCESS` on nonexistent pid ⟹ `-ESRCH`.
- [ ] AC-7: `IOPRIO_WHO_PGRP` on empty pgrp ⟹ `-ESRCH`.
- [ ] AC-8: `IOPRIO_WHO_PGRP` returns *highest* across pgrp (e.g., one task at RT/2 and rest at BE/4 ⟹ RT/2).
- [ ] AC-9: `IOPRIO_WHO_USER` on uid with no tasks ⟹ `-ESRCH`.
- [ ] AC-10: `IOPRIO_WHO_USER` aggregates across all tasks of uid, returning highest.
- [ ] AC-11: No capability required: unprivileged caller can read any visible task's ioprio.
- [ ] AC-12: Return value is a non-negative `i32`; callers may receive `0` legitimately (caller must NOT use `-1` as failure marker — use errno).

### Architecture

```
struct IoprioGetArgs {
    which: i32,
    who: i32,
}
```

`Ioprio::sys_ioprio_get(args) -> i32`:

1. `match args.which { 1 | 2 | 3 => (), _ => return -EINVAL }`
2. `match args.which {`
3. `    IOPRIO_WHO_PROCESS => Ioprio::get_process(args.who),`
4. `    IOPRIO_WHO_PGRP   => Ioprio::get_pgrp(args.who),`
5. `    IOPRIO_WHO_USER   => Ioprio::get_user(args.who),`
6. `}`

`Ioprio::get_process(pid) -> i32`:

1. `let target_pid = if pid == 0 { current().pid } else { pid };`
2. `rcu_read_lock();`
3. `let task = find_task_by_vpid(target_pid).ok_or(-ESRCH)?;`
4. `let p = Ioprio::derive(&task);`
5. `rcu_read_unlock();`
6. `return p;`

`Ioprio::get_pgrp(pgrp) -> i32`:

1. `let target = if pgrp == 0 { current().task_pgrp() } else { find_vpid(pgrp).ok_or(-ESRCH)? };`
2. `let mut best: Option<i32> = None;`
3. `rcu_read_lock();`
4. `for task in do_each_pid_thread(target, PIDTYPE_PGID) {`
5. `    let p = Ioprio::derive(&task);`
6. `    best = Some(match best { None => p, Some(b) => Ioprio::pick_higher(b, p) });`
7. `}`
8. `rcu_read_unlock();`
9. `best.ok_or(-ESRCH)`

`Ioprio::get_user(uid) -> i32`:

1. `let target_uid = if uid == 0 { current().uid() } else { Kuid::from_raw(uid) };`
2. `let mut best: Option<i32> = None;`
3. `rcu_read_lock();`
4. `for task in for_each_process() {`
5. `    if task.real_cred.uid != target_uid { continue; }`
6. `    let p = Ioprio::derive(&task);`
7. `    best = Some(match best { None => p, Some(b) => Ioprio::pick_higher(b, p) });`
8. `}`
9. `rcu_read_unlock();`
10. `best.ok_or(-ESRCH)`

`Ioprio::derive(task) -> i32`:

1. `if task.ioprio_class == IOPRIO_CLASS_NONE {`
2. `    let nice = task_nice(task);`
3. `    let data = ((nice + 20) / 5).clamp(0, 7);`
4. `    return (IOPRIO_CLASS_BE << 13) | (data as i32);`
5. `}`
6. `return task.ioprio;`

`Ioprio::pick_higher(a, b) -> i32`:

1. `let (ca, cb) = (a >> 13, b >> 13);`
2. /* class ordering for "highest": RT (1) > BE (2) > IDLE (3); NONE handled via synthesis upstream */
3. `let class_rank = |c| match c { IOPRIO_CLASS_RT => 0, IOPRIO_CLASS_BE => 1, IOPRIO_CLASS_IDLE => 2, _ => 3 };`
4. `if class_rank(ca) < class_rank(cb) { return a; }`
5. `if class_rank(ca) > class_rank(cb) { return b; }`
6. `if (a & 7) < (b & 7) { a } else { b }`

### Out of Scope

- `ioprio_set(2)` — separate Tier-5 (`ioprio_set.md`).
- BFQ/mq-deadline I/O class semantics — covered in `block/bfq.md`, `block/mq-deadline.md` Tier-3.
- cgroup-v2 `io.weight` introspection — covered in `cgroup/io.md` Tier-3.
- Nice-priority subsystem — covered in `kernel/sched.md` Tier-3.
- Implementation code.

### signature

C (Linux-specific / man-pages):

```c
int ioprio_get(int which, int who);
```

glibc has no wrapper; callers use `syscall(SYS_ioprio_get, which, who)` or `ionice(1)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(ioprio_get,
                int, which,
                int, who);
```

Rookery dispatch:

```rust
pub fn sys_ioprio_get(
    which: i32,
    who: i32,
) -> SyscallResult<i32>;
```

### parameters

| name   | type | constraints                                                                                       | errno-on-bad           |
|--------|------|---------------------------------------------------------------------------------------------------|------------------------|
| which  | `int` | One of `IOPRIO_WHO_PROCESS=1`, `IOPRIO_WHO_PGRP=2`, `IOPRIO_WHO_USER=3`.                          | `EINVAL`               |
| who    | `int` | `0` means "self/caller's pgrp/caller's uid"; else interpreted per `which` as pid, pgrp, or uid.   | `ESRCH`/`EINVAL`       |

### return value

- Success: a non-negative packed `ioprio` value: `(class << 13) | (hint << 3) | data`.
- Failure: `-1` with `errno`; in-kernel: a negated errno.

Note: success values may be **zero** (`IOPRIO_CLASS_NONE | 0 | 0`); callers must distinguish via `errno == 0` (`-1` is the failure marker only because the call cannot legitimately return `-1` ≥ 32-bit; the kernel returns negated errno internally).

### errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EINVAL`       | `which` not one of the three values.                                                            |
| `ESRCH`        | No task matches `(which, who)`.                                                                 |
| `EPERM`        | Caller cannot observe the target task (`/proc`-style hiding via `GRKERNSEC_PROC_USER` etc., manifesting as `-ESRCH`/`-EPERM`). |

### abi surface (constants + flags)

From `include/uapi/linux/ioprio.h`:

- `IOPRIO_WHO_PROCESS = 1`, `IOPRIO_WHO_PGRP = 2`, `IOPRIO_WHO_USER = 3`.
- `IOPRIO_CLASS_NONE = 0`, `_RT = 1`, `_BE = 2`, `_IDLE = 3`.
- `IOPRIO_CLASS_SHIFT = 13`, `IOPRIO_NR_LEVELS = 8`.
- Decoding helpers: `IOPRIO_PRIO_CLASS(p) = (p) >> 13`, `IOPRIO_PRIO_DATA(p) = (p) & 7`, `IOPRIO_PRIO_HINT(p) = ((p) >> 3) & 0x3FF`.
- Nice-derivation rule: `task_nice_ioprio(nice) = (nice + 20) / 5` (range 0..7).

### compatibility contract

- REQ-1: Argument lowering: `%rdi=which`, `%rsi=who`.
- REQ-2: `which ∉ {IOPRIO_WHO_PROCESS, IOPRIO_WHO_PGRP, IOPRIO_WHO_USER} ⟹ -EINVAL`.
- REQ-3: For `IOPRIO_WHO_PROCESS`: `find_task_by_vpid(who or current->pid)`; not found ⟹ `-ESRCH`. Read task's stored ioprio under `task->alloc_lock` (rcu-safe read).
- REQ-4: For `IOPRIO_WHO_PGRP`: iterate `do_each_pid_thread(pid_for(who or current_pgrp), PIDTYPE_PGID)`; if empty ⟹ `-ESRCH`; else return *highest* priority across all members (lowest class index; within same class lowest data).
- REQ-5: For `IOPRIO_WHO_USER`: iterate `for_each_process(p)` filtering `p->cred->uid == target_uid`; if empty ⟹ `-ESRCH`; else return *highest* priority across all matches.
- REQ-6: Per-task derivation: if `task->ioprio_class == IOPRIO_CLASS_NONE`, synthesize `(IOPRIO_CLASS_BE << 13) | task_nice_ioprio(task_nice(task))`.
- REQ-7: No capability gate for reading: any task may read priority of any other task (subject to `/proc` visibility rules under hardening).
- REQ-8: RCU read-side critical section for task traversal (no `tasklist_lock` write).
- REQ-9: Pgrp/user aggregation order: O(N) over members; bounded by sched-statistic walk.
- REQ-10: Return value range: `[0, (3 << 13) | (0x3FF << 3) | 7]`; fits in `i32`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `which_validated` | INVARIANT | `which ∈ {1,2,3}`. |
| `synthesis_correct` | INVARIANT | `IOPRIO_CLASS_NONE` ⟹ returned class is BE, data is `(nice+20)/5` clamped. |
| `pgrp_returns_highest` | INVARIANT | `IOPRIO_WHO_PGRP` returns max-priority across members. |
| `user_returns_highest` | INVARIANT | `IOPRIO_WHO_USER` returns max-priority across uid's tasks. |
| `no_capability_required` | INVARIANT | No `capable()` check; visibility only. |
| `empty_set_errors_esrch` | INVARIANT | No task selected ⟹ `-ESRCH`. |

### Layer 2: TLA+

`uapi/syscalls/ioprio_get.tla`:
- Per-call → validate → iterate (under RCU) → derive-or-synthesize → aggregate-or-single.
- Properties:
  - `safety_synthesis_for_none_class`,
  - `safety_aggregate_max_correct`,
  - `safety_no_priority_leak_across_pid_ns`,
  - `liveness_ioprio_get_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ returns valid packed ioprio in expected range | `Ioprio::derive` |
| Post: `IOPRIO_WHO_PROCESS` ⟹ returns target task's ioprio (post-synthesis) | `Ioprio::get_process` |
| Post: aggregate calls ⟹ returns highest priority observed | `Ioprio::get_pgrp`, `Ioprio::get_user` |
| Post: any error ⟹ no kernel state changed | `Ioprio::sys_ioprio_get` |

### Layer 4: Verus/Creusot functional

`ioprio_get(which, who)` ≡ Linux `ioprio_get` per `man 2 ioprio_get` and `Documentation/block/ioprio.rst`: three selectors, nice-derived synthesis, highest-of-aggregate semantics.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`ioprio_get(2)` reinforcement:

- **Per-`which` whitelist** — defense against silent acceptance of unknown selectors.
- **Per-RCU read-side traversal** — defense against tasklist write-contention DoS.
- **Per-synthesis correctness** — defense against ambiguous report for `IOPRIO_CLASS_NONE`.
- **No-capability read** — preserved (POSIX/Linux semantics), with visibility gated by `/proc` hiding.
- **Per-empty-set `-ESRCH`** — defense against silent ambiguous return.
- **Per-pgrp/user aggregation bounded** — defense against unbounded walk on huge user/pgrp sets (subject to `RLIMIT_NPROC`).

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `ioprio_get` has no user pointers; trivially safe.
- **GRKERNSEC_PROC_GETPID** — `IOPRIO_WHO_PROCESS` on hidden pids returns `-ESRCH` rather than leaking existence of restricted tasks. **Info-leak defense**: prevents priority side-channel about other users' workloads.
- **GRKERNSEC_PROC_USER** — `IOPRIO_WHO_USER` aggregation restricted to caller's own uid unless `CAP_SYS_PTRACE` granted; cross-uid reads return `-ESRCH` for hidden tasks.
- **GRKERNSEC_AUDIT_IOPRIO_GET** — repeated `IOPRIO_WHO_USER` enumeration logged as potential info-leak probe.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_REFCOUNT** — `task_struct`, `pid` refcounts saturating under RCU.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **GRKERNSEC_PROC_IPC** — `/proc/<pid>/io` access pattern correlates with `ioprio_get` for additional rate-limit signal.
- **GRKERNSEC_CHROOT_PROC** — chrooted `IOPRIO_WHO_USER` cannot enumerate uid's tasks outside chroot.
- **GRKERNSEC_RESLOG** — high-frequency `ioprio_get` per-user enumeration logged for forensic correlation.
- **PAX_KERNEXEC** — task-list walk in read-only kernel text region; no code-flow corruption surface.

