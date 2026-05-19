# Tier-5: syscall 251 — ioprio_set(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`251  common  ioprio_set  sys_ioprio_set`)
  - block/ioprio.c (`SYSCALL_DEFINE3(ioprio_set, ...)`, `set_task_ioprio`)
  - include/uapi/linux/ioprio.h (`IOPRIO_WHO_*`, `IOPRIO_CLASS_*`, `IOPRIO_PRIO_VALUE`, `IOPRIO_HINT_*`)
  - kernel/sched/core.c (`task_set_ioprio`)
  - include/linux/sched/user.h (`struct user_struct`)
-->

## Summary

`ioprio_set(2)` is **x86_64 syscall 251**, the I/O scheduling-priority setter. It assigns the encoded I/O priority `ioprio` to one or more tasks selected by `(which, who)`:

- `IOPRIO_WHO_PROCESS` — `who` is a `pid` (or 0 = current task), sets a single task.
- `IOPRIO_WHO_PGRP` — `who` is a process-group id (or 0 = caller's pgrp), sets every member.
- `IOPRIO_WHO_USER` — `who` is a real `uid` (or 0 = caller's uid), sets every task owned by that user.

The `ioprio` value is a packed 16-bit encoding: high 3 bits = **class** (`IOPRIO_CLASS_NONE`, `_RT`, `_BE`, `_IDLE`), low 13 bits split into **data** (low 3 bits, the priority level 0–7) and **hint** (10 bits, advisory scheduler hints like `IOPRIO_HINT_DEV_DURATION`). The block-layer I/O scheduler (BFQ, mq-deadline, none) consults these on every BIO submission.

`IOPRIO_CLASS_RT` is a real-time class that may *starve* other I/O; setting it (or moving a task into it from another class) requires `CAP_SYS_NICE` or `CAP_SYS_ADMIN`. `IOPRIO_CLASS_IDLE` is also gated by `CAP_SYS_NICE`. Cross-user/cross-pgrp updates require either matching real uid or `CAP_SYS_NICE`/`CAP_SYS_ADMIN`.

Critical for: every container-runtime I/O isolation, every cgroup-aware throttling helper, every `ionice(1)` user, every database/back-end attempting to deprioritize background I/O, every Rookery test asserting class-gate enforcement and `(which, who)` permission semantics.

## Signature

C (Linux-specific / man-pages):

```c
int ioprio_set(int which, int who, int ioprio);
```

glibc has no wrapper; callers use `syscall(SYS_ioprio_set, which, who, ioprio)` or `ionice(1)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(ioprio_set,
                int, which,
                int, who,
                int, ioprio);
```

Rookery dispatch:

```rust
pub fn sys_ioprio_set(
    which: i32,
    who: i32,
    ioprio: i32,
) -> SyscallResult<i32>;
```

## Parameters

| name   | type | constraints                                                                                       | errno-on-bad           |
|--------|------|---------------------------------------------------------------------------------------------------|------------------------|
| which  | `int` | One of `IOPRIO_WHO_PROCESS=1`, `IOPRIO_WHO_PGRP=2`, `IOPRIO_WHO_USER=3`.                          | `EINVAL`               |
| who    | `int` | `0` means "self/caller's pgrp/caller's uid"; else interpreted per `which` as pid, pgrp, or uid.   | `ESRCH`/`EINVAL`       |
| ioprio | `int` | Packed `IOPRIO_PRIO_VALUE(class, hint, data)` with `class ∈ {NONE,RT,BE,IDLE}`, `data ∈ [0,7]`, hint validated per kernel. | `EINVAL`/`EPERM` |

## Return value

- Success: `0`.
- Failure: `-1` with `errno`; in-kernel: a negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EINVAL`       | `which` not one of the three values; or `IOPRIO_PRIO_CLASS(ioprio)` not in `{NONE,RT,BE,IDLE}`; or `IOPRIO_PRIO_DATA(ioprio) > 7`. |
| `ESRCH`        | No task matches `(which, who)` (no such pid, empty pgrp, no tasks owned by uid).                |
| `EPERM`        | Caller lacks `CAP_SYS_NICE`/`CAP_SYS_ADMIN` when required: setting `IOPRIO_CLASS_RT` or `IOPRIO_CLASS_IDLE`, or changing priority of a task whose real uid differs from caller's. |

## ABI surface (constants + flags)

From `include/uapi/linux/ioprio.h`:

- `IOPRIO_CLASS_NONE = 0` — kernel chooses default (typically maps to BE/4 from nice).
- `IOPRIO_CLASS_RT = 1` — real-time, may starve others; **CAP_SYS_NICE** required.
- `IOPRIO_CLASS_BE = 2` — best-effort (default).
- `IOPRIO_CLASS_IDLE = 3` — runs only when device idle; **CAP_SYS_NICE** required.
- `IOPRIO_CLASS_SHIFT = 13`; `IOPRIO_NR_LEVELS = 8` (data 0..7); hint bits cover `[3..13)`.
- `IOPRIO_WHO_PROCESS = 1`, `IOPRIO_WHO_PGRP = 2`, `IOPRIO_WHO_USER = 3`.
- Hint constants: `IOPRIO_HINT_NONE = 0`, `IOPRIO_HINT_DEV_DURATION_LIMIT_1..7` (validated by kernel; unknown bits ⟹ `-EINVAL`).
- Encoding helpers: `IOPRIO_PRIO_VALUE(class, data) = (class << 13) | data`; with hint: `IOPRIO_PRIO_VALUE_HINT(class, hint, data) = (class << 13) | (hint << 3) | data`.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=which`, `%rsi=who`, `%rdx=ioprio`.
- REQ-2: `which ∉ {IOPRIO_WHO_PROCESS, IOPRIO_WHO_PGRP, IOPRIO_WHO_USER} ⟹ -EINVAL`.
- REQ-3: `class = IOPRIO_PRIO_CLASS(ioprio); data = IOPRIO_PRIO_DATA(ioprio); hint = IOPRIO_PRIO_HINT(ioprio)`.
- REQ-4: `class ∉ {NONE,RT,BE,IDLE} ⟹ -EINVAL`.
- REQ-5: `data > IOPRIO_NR_LEVELS - 1 (=7) ⟹ -EINVAL`.
- REQ-6: `hint` value validated against kernel-known hint set; unknown bits ⟹ `-EINVAL`.
- REQ-7: `class == RT || class == IDLE` ⟹ `capable(CAP_SYS_NICE)` required; else `-EPERM`.
- REQ-8: For each task selected by `(which, who)`: cross-uid update requires `task->real_cred->uid == current_uid()` OR `capable(CAP_SYS_NICE)`/`CAP_SYS_ADMIN`.
- REQ-9: For `IOPRIO_WHO_PROCESS`: `find_task_by_vpid(who or current->pid)`; not found ⟹ `-ESRCH`.
- REQ-10: For `IOPRIO_WHO_PGRP`: iterate `do_each_pid_thread(find_vpid(who or current->pgrp), PIDTYPE_PGID, t)`; empty ⟹ `-ESRCH`.
- REQ-11: For `IOPRIO_WHO_USER`: iterate `for_each_process(p)` filtering `p->cred->uid == target_uid`; empty ⟹ `-ESRCH`.
- REQ-12: Atomic per-task update: acquire `task->alloc_lock`, store `task->io_context->ioprio = ioprio` (allocating an io_context if needed), or store `task->io_uring->ioprio` for io_uring contexts.
- REQ-13: `class == NONE` resets to nice-derived default at next BIO submission.
- REQ-14: Audit hook: `audit_log_io_priority(target_pid, ioprio)`.

## Acceptance Criteria

- [ ] AC-1: `ioprio_set(IOPRIO_WHO_PROCESS, 0, IOPRIO_PRIO_VALUE(IOPRIO_CLASS_BE, 4))` ⟹ `0`; `ioprio_get` returns same value.
- [ ] AC-2: `IOPRIO_CLASS_RT` without `CAP_SYS_NICE` ⟹ `-EPERM`.
- [ ] AC-3: `IOPRIO_CLASS_IDLE` without `CAP_SYS_NICE` ⟹ `-EPERM`.
- [ ] AC-4: `data = 8` ⟹ `-EINVAL`.
- [ ] AC-5: `class = 5` (unknown) ⟹ `-EINVAL`.
- [ ] AC-6: `which = 99` ⟹ `-EINVAL`.
- [ ] AC-7: `IOPRIO_WHO_PROCESS` on nonexistent pid ⟹ `-ESRCH`.
- [ ] AC-8: `IOPRIO_WHO_PGRP` on empty pgrp ⟹ `-ESRCH`.
- [ ] AC-9: `IOPRIO_WHO_USER` on nonexistent uid (no tasks) ⟹ `-ESRCH`.
- [ ] AC-10: Cross-uid update without `CAP_SYS_NICE` ⟹ `-EPERM`.
- [ ] AC-11: Setting `IOPRIO_CLASS_NONE` ⟹ subsequent BIO submission derives priority from nice.
- [ ] AC-12: Update is per-task immediate: subsequent `ioprio_get` on same target reflects new value.
- [ ] AC-13: Unknown hint bits ⟹ `-EINVAL`.

## Architecture

```
struct IoprioSetArgs {
    which: i32,
    who: i32,
    ioprio: i32,
}
```

`Ioprio::sys_ioprio_set(args) -> i32`:

1. /* Validate which */ `match args.which { 1 | 2 | 3 => (), _ => return -EINVAL }`
2. /* Decode ioprio */ `let class = (args.ioprio >> 13) & 0x7; let data = args.ioprio & 0x7; let hint = (args.ioprio >> 3) & 0x3FF;`
3. `if !(0..=3).contains(&class) || data > 7 { return -EINVAL; }`
4. `if !Ioprio::hint_supported(hint) { return -EINVAL; }`
5. /* Capability gate for RT and IDLE */
6. `if (class == IOPRIO_CLASS_RT || class == IOPRIO_CLASS_IDLE) && !capable(CAP_SYS_NICE) { return -EPERM; }`
7. /* Dispatch on which */
8. `match args.which {`
9. `    IOPRIO_WHO_PROCESS => Ioprio::set_process(args.who, args.ioprio),`
10. `    IOPRIO_WHO_PGRP   => Ioprio::set_pgrp(args.who, args.ioprio),`
11. `    IOPRIO_WHO_USER   => Ioprio::set_user(args.who, args.ioprio),`
12. `}`

`Ioprio::set_process(pid, ioprio) -> i32`:

1. `let target_pid = if pid == 0 { current().pid } else { pid };`
2. `let task = find_task_by_vpid(target_pid).ok_or(-ESRCH)?;`
3. `Ioprio::check_task_cred(&task)?;`
4. `Ioprio::set_task_ioprio(&task, ioprio)`

`Ioprio::set_pgrp(pgrp, ioprio) -> i32`:

1. `let target = if pgrp == 0 { current().task_pgrp() } else { find_vpid(pgrp).ok_or(-ESRCH)? };`
2. `let mut any = false; let mut err = 0;`
3. `for task in do_each_pid_thread(target, PIDTYPE_PGID) {`
4. `    any = true;`
5. `    Ioprio::check_task_cred(&task)?;`
6. `    err = Ioprio::set_task_ioprio(&task, ioprio);`
7. `    if err != 0 { return err; }`
8. `}`
9. `if !any { return -ESRCH; }`
10. `return 0;`

`Ioprio::set_user(uid, ioprio) -> i32`:

1. `let target_uid = if uid == 0 { current().uid() } else { Kuid::from_raw(uid) };`
2. `let mut any = false;`
3. `rcu_read_lock();`
4. `for task in for_each_process() {`
5. `    if task.real_cred.uid != target_uid { continue; }`
6. `    any = true;`
7. `    Ioprio::check_task_cred(&task)?;`
8. `    Ioprio::set_task_ioprio(&task, ioprio)?;`
9. `}`
10. `rcu_read_unlock();`
11. `if !any { return -ESRCH; }`
12. `return 0;`

`Ioprio::check_task_cred(task) -> Result<(), i32>`:

1. `if task.real_cred.uid == current().uid() { return Ok(()); }`
2. `if capable(CAP_SYS_NICE) || capable(CAP_SYS_ADMIN) { return Ok(()); }`
3. `Err(-EPERM)`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `which_validated` | INVARIANT | `which ∈ {1,2,3}`. |
| `class_validated` | INVARIANT | `class ∈ {NONE,RT,BE,IDLE}`. |
| `data_validated` | INVARIANT | `data ≤ 7`. |
| `rt_idle_capability_gate` | INVARIANT | Setting RT or IDLE requires `CAP_SYS_NICE`. |
| `cross_uid_capability_gate` | INVARIANT | Target uid ≠ caller uid ⟹ `CAP_SYS_NICE`/`ADMIN` required. |
| `empty_target_set_errors_esrch` | INVARIANT | No task selected ⟹ `-ESRCH`. |

### Layer 2: TLA+

`uapi/syscalls/ioprio_set.tla`:
- Per-call → validate-encoding → capability-gate → iterate-targets → per-task-update.
- Properties:
  - `safety_only_authorized_class_changes`,
  - `safety_per_task_update_atomic`,
  - `safety_no_partial_pgrp_update_on_eperm`,
  - `liveness_ioprio_set_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ every selected task's ioprio == requested value | `Ioprio::set_*` |
| Post: `-EPERM` ⟹ no task modified (atomic on first deny) | `Ioprio::set_pgrp`, `Ioprio::set_user` |
| Post: `-ESRCH` ⟹ no task modified | `Ioprio::set_*` |
| Post: `IOPRIO_CLASS_NONE` ⟹ next BIO submission derives from nice | `BlockLayer::derive_ioprio` |

### Layer 4: Verus/Creusot functional

`ioprio_set(which, who, ioprio)` ≡ Linux ioprio_set per `man 2 ioprio_set` and `Documentation/block/ioprio.rst`: class gating, three target selectors, per-task atomic update.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`ioprio_set(2)` reinforcement:

- **Per-`which` whitelist** — defense against silent acceptance of unknown selectors.
- **Per-class whitelist** — defense against undefined-class injection.
- **Per-data bounds (≤ 7)** — defense against array out-of-bound priority indexing.
- **Per-hint validation** — defense against silent acceptance of unknown hint bits (future-proof).
- **Per-RT/IDLE capability gate** — defense against unprivileged starvation of system I/O.
- **Per-cross-uid capability gate** — defense against unprivileged interference with other users' I/O.
- **Per-empty-target `-ESRCH`** — defense against silent no-op semantics confusing callers.
- **Atomic first-deny in batch** — defense against partial pgrp/user updates.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `ioprio_set` has no user pointers; trivially safe.
- **GRKERNSEC_PROC_GETPID** — `IOPRIO_WHO_PROCESS` cannot enumerate pids hidden by `GRKERNSEC_PROC_USER`; `find_task_by_vpid` returns NULL ⟹ `-ESRCH` for hidden pids.
- **GRKERNSEC_RLIMIT** — RT class respects per-uid hard limit on count of RT-class tasks (analogous to `RLIMIT_RTPRIO`).
- **GRKERNSEC_TRUSTED** — `CAP_SYS_NICE` and `CAP_SYS_ADMIN` checked under capability namespace rules; container-confined caps cannot escape.
- **GRKERNSEC_AUDIT_IOPRIO** — every `ioprio_set` (especially RT/IDLE/cross-uid) logged with caller uid/exe and target pid/uid.
- **GRKERNSEC_HIDESYM** — `-EPERM` printks redact kernel pointers.
- **PAX_REFCOUNT** — `task_struct`, `io_context`, `user_struct` refcounts saturating.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **GRKERNSEC_PROC_IPC** — RT-class assignments visible only to owner/CAP_SYS_ADMIN in `/proc/<pid>/io`.
- **GRKERNSEC_CHROOT_NICE** — chrooted task cannot escalate to RT class even with `CAP_SYS_NICE` inside chroot.
- **GRKERNSEC_RESLOG** — repeated cross-uid `-EPERM` denials logged for forensic analysis.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `ioprio_get(2)` — separate Tier-5 (`ioprio_get.md`).
- BFQ/mq-deadline scheduler internals — covered in `block/bfq.md`, `block/mq-deadline.md` Tier-3.
- cgroup-v2 `io.weight` — covered in `cgroup/io.md` Tier-3.
- io_uring per-request priority — covered in `io_uring/00-overview.md` Tier-3.
- Implementation code.
