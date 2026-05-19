# Tier-5 syscall: sched_getattr(2) — syscall 315

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sched/core.c (SYSCALL_DEFINE4(sched_getattr))
  - kernel/sched/syscalls.c (sched_attr_copy_to_user)
  - include/uapi/linux/sched.h (struct sched_attr)
  - arch/x86/entry/syscalls/syscall_64.tbl (315 common sched_getattr)
-->

## Summary

`sched_getattr(2)` retrieves the **full scheduling parameters** of a target task into a userspace `struct sched_attr` buffer. It is the read counterpart of `sched_setattr(2)`, returning policy, sched_flags, nice, sched_priority, the SCHED_DEADLINE triple (runtime, deadline, period), and util-clamp min/max. The buffer is sized via the `size` argument and uses the same forward-compatibility contract: kernel writes min(usize, ksize) bytes; if usize > ksize, the call returns -E2BIG. Critical for: observability (`chrt --all-tasks --print`, ps tools), SCHED_DEADLINE diagnostic dump, util-clamp monitoring, Rookery's policy auditor snapshotting every task, and libraries that adapt to caller's scheduling class.

## Signature

```c
int sched_getattr(pid_t pid, struct sched_attr *attr,
                  unsigned int size, unsigned int flags);
```

## Parameters

| Parameter | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | Target PID (caller's active pid-ns); 0 = self. |
| `attr` | `struct sched_attr *` | out (UAPI) | Buffer to receive the params; written via copy_to_user. |
| `size` | `unsigned int` | in | Userspace's view of sizeof(struct sched_attr). |
| `flags` | `unsigned int` | in | Must be 0. |

## Return value

| Value | Meaning |
|---|---|
| 0 | Success; `*attr` populated. |
| `-EINVAL` | size < SCHED_ATTR_SIZE_VER0 OR size > PAGE_SIZE OR flags != 0 OR pid < 0. |
| `-ESRCH` | PID not in caller's pid-ns. |
| `-EFAULT` | attr buffer not writable. |
| `-E2BIG` | Kernel's struct is smaller than usize; cannot represent all kernel fields. |
| `-EPERM` | LSM denies (security_task_getscheduler). |

## Errors

| Errno | Trigger |
|---|---|
| `EINVAL` | size < SCHED_ATTR_SIZE_VER0 (48); size > PAGE_SIZE; flags != 0; pid < 0. |
| `ESRCH` | PID not found, or cross-namespace. |
| `EFAULT` | attr unwritable / unaligned. |
| `E2BIG` | usize > ksize; kernel-known size insufficient. |
| `EPERM` | LSM denial via security_task_getscheduler. |

## ABI surface

```text
__NR_sched_getattr (x86_64) = 315
__NR_sched_getattr (i386)   = 352
__NR_sched_getattr (arm64)  = 275  (generic-syscall)
__NR_sched_getattr (generic) = 275

SCHED_ATTR_SIZE_VER0   = 48        /* original */
SCHED_ATTR_SIZE_VER1   = 56        /* added util_clamp_{min,max} */

struct sched_attr {
    __u32 size;
    __u32 sched_policy;
    __u64 sched_flags;
    __s32 sched_nice;
    __u32 sched_priority;
    __u64 sched_runtime;
    __u64 sched_deadline;
    __u64 sched_period;
    __u32 sched_util_min;
    __u32 sched_util_max;
};
```

## Compatibility contract

REQ-1: Syscall number is **315** on x86_64; **275** on arm64/generic.

REQ-2: pid == 0 ⟹ current.

REQ-3: pid < 0 ⟹ -EINVAL.

REQ-4: size must be in [SCHED_ATTR_SIZE_VER0, PAGE_SIZE]; outside ⟹ -EINVAL.

REQ-5: flags MUST be 0; otherwise -EINVAL.

REQ-6: Kernel populates an internal `struct sched_attr` of size ksize from the task's state, then writes min(usize, ksize) bytes to attr. The first u32 written to userspace is `attr.size = min(usize, ksize)`.

REQ-7: If usize > ksize ⟹ -E2BIG (the kernel cannot fully describe its newer view via a smaller userspace ABI — but here the inequality is reversed; semantics: kernel's struct may be smaller than userspace's, in which case userspace's extra fields would not be set ⟹ -E2BIG to flag the mismatch).

   Note: documented kernel behavior is "if user's size is bigger than kernel knows, kernel returns 0 with attr.size=ksize (leaving the rest of user's buffer untouched), and userspace is expected to check attr.size to know what was filled". The -E2BIG variant is exercised only if the kernel decides the user buffer cannot be safely populated; current upstream returns 0 in that case. Rookery follows upstream: usize > ksize ⟹ 0 with attr.size = ksize.

REQ-8: For SCHED_DEADLINE tasks, runtime/deadline/period reflect the dl_se reservation parameters (in nanoseconds).

REQ-9: For non-deadline tasks, runtime/deadline/period are zeroed.

REQ-10: For SCHED_NORMAL/BATCH/IDLE tasks, sched_priority is 0 and sched_nice contains the current nice value.

REQ-11: For SCHED_FIFO/RR tasks, sched_nice is 0 and sched_priority is 1..99.

REQ-12: sched_flags reflects flags currently set on the task: SCHED_FLAG_RESET_ON_FORK iff task.sched_reset_on_fork; SCHED_FLAG_DL_OVERRUN/RECLAIM for deadline tasks.

REQ-13: sched_util_min / sched_util_max are populated from task.uclamp_req[]; if not set, they default to 0 / SCHED_CAPACITY_SCALE (1024) respectively.

REQ-14: LSM hook `security_task_getscheduler(task)` fires; denial ⟹ -EACCES / -EPERM.

REQ-15: Holds RCU read lock to find the task; reads of policy/priority/nice are atomic per field; no rq lock.

REQ-16: The call is namespace-aware: cross-pid-ns ⟹ -ESRCH.

## Acceptance Criteria

- [ ] AC-1: After `sched_setattr(0, {policy=DEADLINE, runtime=1ms, deadline=10ms, period=10ms}, 0)`, `sched_getattr(0, &out, sizeof(out), 0)` returns deadline triple equal.
- [ ] AC-2: For a SCHED_FIFO task at prio 50, `sched_getattr` returns sched_policy=FIFO, sched_priority=50, sched_nice=0, runtime/deadline/period = 0.
- [ ] AC-3: For a SCHED_NORMAL task with nice -5, `sched_getattr` returns sched_policy=NORMAL, sched_nice=-5, sched_priority=0.
- [ ] AC-4: `sched_getattr(pid, NULL, sizeof(*attr), 0)` → -EFAULT.
- [ ] AC-5: `sched_getattr(pid, &out, 47, 0)` → -EINVAL (size < VER0).
- [ ] AC-6: `sched_getattr(pid, &out, sizeof(out), 1)` → -EINVAL (flags != 0).
- [ ] AC-7: `sched_getattr(-1, &out, sizeof(out), 0)` → -EINVAL.
- [ ] AC-8: `sched_getattr(99999, &out, sizeof(out), 0)` → -ESRCH.
- [ ] AC-9: Cross-pid-ns query → -ESRCH.
- [ ] AC-10: util-clamp set via sched_setattr is reflected in sched_util_min / sched_util_max.
- [ ] AC-11: attr.size returned equals min(usize, ksize).
- [ ] AC-12: 1e6 concurrent calls produce consistent values (atomic per-field reads).

## Architecture

```rust
#[syscall(nr = 315, abi = "sysv")]
pub fn sys_sched_getattr(pid: pid_t,
                         uattr: UserPtrMut<SchedAttr>,
                         usize_: u32,
                         flags: u32) -> SysResult<i32> {
    SchedGetattr::do_call(pid, uattr, usize_, flags)
}
```

`SchedGetattr::do_call(pid, uattr, usize_, flags) -> i32`:
1. if flags != 0 || pid < 0 { return -EINVAL; }
2. if usize_ < SCHED_ATTR_SIZE_VER0 || usize_ > PAGE_SIZE as u32 { return -EINVAL; }
3. rcu_read_lock();
4. let task = if pid == 0 { current() } else {
       match find_task_by_vpid(pid) { Some(t)=>t, None=>{ rcu_read_unlock(); return -ESRCH; } }
   };
5. if let Err(e) = security_task_getscheduler(task) { rcu_read_unlock(); return e; }
6. let mut k_attr = SchedAttr::zeroed();
7. k_attr.size = size_of::<SchedAttr>() as u32;
8. k_attr.sched_policy = task.policy;
9. k_attr.sched_flags  = SchedGetattr::flags_for(task);
10. match task.policy {
        SCHED_NORMAL|SCHED_BATCH|SCHED_IDLE => k_attr.sched_nice = task_nice(task),
        SCHED_FIFO|SCHED_RR                 => k_attr.sched_priority = task.rt_priority,
        SCHED_DEADLINE                      => {
            k_attr.sched_runtime  = task.dl.dl_runtime;
            k_attr.sched_deadline = task.dl.dl_deadline;
            k_attr.sched_period   = task.dl.dl_period;
        }
        _ => {}
    }
11. k_attr.sched_util_min = task.uclamp_req[UCLAMP_MIN].value;
12. k_attr.sched_util_max = task.uclamp_req[UCLAMP_MAX].value;
13. rcu_read_unlock();
14. /* Write back: copy min(usize, ksize) bytes; set attr.size to that */
15. let to_copy = min(usize_, k_attr.size);
16. let mut k_attr_w = k_attr; k_attr_w.size = to_copy;
17. copy_to_user_slice(uattr.as_bytes_mut(), k_attr_w.as_bytes(), to_copy as usize)?;
18. 0

`SchedGetattr::flags_for(task) -> u64`:
1. let mut f = 0u64;
2. if task.sched_reset_on_fork                       { f |= SCHED_FLAG_RESET_ON_FORK; }
3. if task.policy == SCHED_DEADLINE && task.dl.dl_overrun { f |= SCHED_FLAG_DL_OVERRUN; }
4. if task.policy == SCHED_DEADLINE && task.dl.dl_reclaim { f |= SCHED_FLAG_RECLAIM; }
5. if task.uclamp_req[UCLAMP_MIN].active             { f |= SCHED_FLAG_UTIL_CLAMP_MIN; }
6. if task.uclamp_req[UCLAMP_MAX].active             { f |= SCHED_FLAG_UTIL_CLAMP_MAX; }
7. f

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `size_bounds` | INVARIANT | per-call: usize ∈ [VER0, PAGE_SIZE]. |
| `flags_zero_only` | INVARIANT | per-call: flags != 0 ⟹ -EINVAL. |
| `rcu_read_lock_balanced` | INVARIANT | per-call: every rcu_read_lock paired with unlock on all paths. |
| `copy_to_user_bounded` | INVARIANT | per-call: writes ≤ min(usize, ksize) bytes; never overflows user buffer. |
| `no_mutation` | INVARIANT | per-call: no task field written. |
| `lsm_hook_invoked` | INVARIANT | per-call: security_task_getscheduler called pre-read. |
| `attr_size_consistent` | INVARIANT | per-call: returned attr.size == bytes actually copied. |

### Layer 2: TLA+

`kernel/sched_getattr.tla`:
- States: per-task policy/flags/params snapshot.
- Properties:
  - `safety_size_versioning` — per-call: bytes copied == min(usize, ksize).
  - `safety_no_mutation` — per-call: no task state change.
  - `safety_lsm_pre_read` — per-call: LSM denial ⟹ no copy_to_user.
  - `liveness_terminates` — per-call: O(1).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: ret == 0 ⟹ bytes_copied == min(usize, ksize) | `SchedGetattr::do_call` |
| `flags_for` post: bits match task's active flags | `SchedGetattr::flags_for` |
| LSM hook fires before any copy_to_user | `Sched::lsm_hook` |

### Layer 4: Verus / Creusot functional

Per-`sched_getattr(2)` man-page equivalence. LTP `sched_getattr01..02` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sched_getattr(2)` reinforcement:

- **Per-size-bounds** — defense against per-OOB write into user buffer.
- **Per-RCU-balanced lookup** — defense against per-leak on error paths.
- **Per-LSM hook pre-read** — defense against per-info-leak when LSM denies.
- **Per-zeroed kernel buffer** — defense against per-info-leak of stack residue (k_attr is zero-initialized).
- **Per-namespace isolation** — defense against per-cross-container PID enumeration.

## Grsecurity / PaX-style Reinforcement

- **PaX UDEREF on `attr`** — copy_to_user_slice validates the destination is in userspace; UDEREF blocks per-kernel-pointer smuggling that would let an attacker overwrite kernel memory.
- **No CAP requirement for read** — POSIX-compliant; grsec respects GRKERNSEC_PROC_USERGROUP so cross-user queries return -ESRCH not -EPERM (avoid PID-existence leak).
- **GRKERNSEC_RESLOG on policy enumeration** — repeated cross-task introspection from sandboxed tasks is logged.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset; combined with the zero-initialized k_attr this hardens against stack-residue info-leaks even under speculative reads.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a chroot, find_task_by_vpid is restricted to the chroot's task set; out-of-set queries return -ESRCH.
- **Sandbox SCHED_DEADLINE attack-surface reduction** — querying a SCHED_DEADLINE task from inside a sandbox is permitted but audited under GRKERNSEC_AUDIT_TASK so reconnaissance on RT/deadline tasks is observable.
- **No_new_privs neutral** — read-only call; NNP has no effect.
- **KEEPCAPS neutral** — no capability check.
- **Util-clamp leak control** — under sandboxed tasks, util-clamp readback is clamped to the sandbox's own ceiling so that probing an unrelated task's clamp does not expose system thermal policy.
- **Stack residue zeroing** — k_attr is zero-initialized via SchedAttr::zeroed() before populating; grsec enforces that all reserved/unused bytes remain zero across the copy_to_user.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `sched_setattr(2)` (Tier-5 separate doc).
- `sched_getscheduler(2)` (Tier-5 separate doc — legacy API).
- `sched_getparam(2)` (Tier-5 separate doc).
- SCHED_DEADLINE EDF / CBS internals (Tier-3 in `kernel/sched-deadline.md`).
- util-clamp internals (Tier-3 in `kernel/sched-util-clamp.md`).
- Implementation code.
