---
title: "Tier-5 syscall: sched_setaffinity(2) — syscall 203"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sched_setaffinity(2)` sets the **CPU affinity mask** of a target task (PID 0 = self) — the bitmap of CPUs on which the task is allowed to run. The kernel uses this both as a hard constraint on the load balancer and as a CPU-set boundary: when the task is migrated, the destination CPU is chosen only from the intersection of (caller-supplied mask) ∧ (cpus_allowed_by_cgroup_cpuset) ∧ (active CPUs). If the intersection is empty, the call returns -EINVAL (Linux ≥ 4.5; older returned -EINVAL only for empty user mask). The userspace mask is an opaque bitmap of length `cpusetsize` bytes; glibc wraps this with `cpu_set_t`. Critical for: NUMA-pinning, IRQ-thread placement, real-time CPU isolation (`isolcpus=`), Rookery's sandbox CPU subset enforcement, container CPU shares.

### Acceptance Criteria

- [ ] AC-1: `sched_setaffinity(0, sizeof(mask), &mask_with_cpu0_only)` → 0; task pinned to CPU 0.
- [ ] AC-2: `sched_setaffinity(0, sizeof(mask), &empty_mask)` → -EINVAL.
- [ ] AC-3: `sched_setaffinity(0, 0, &mask)` → -EINVAL.
- [ ] AC-4: `sched_setaffinity(0, sizeof(mask), NULL)` → -EFAULT.
- [ ] AC-5: Setting affinity to a CPU not in cgroup cpuset → -EINVAL (empty intersection) or masked.
- [ ] AC-6: Cross-user target without CAP_SYS_NICE → -EPERM.
- [ ] AC-7: Foreign task with CAP_SYS_NICE → 0.
- [ ] AC-8: SCHED_DEADLINE task with shrunk mask insufficient for admission → -EBUSY.
- [ ] AC-9: After setaffinity, running task is migrated to permitted CPU before return.
- [ ] AC-10: `sched_getaffinity(pid)` returns exactly the effective mask (post-intersection).
- [ ] AC-11: Tail bits beyond nr_cpu_ids set → -EINVAL (strict mode).
- [ ] AC-12: cpusetsize > 4096 → -EINVAL.

### Architecture

```rust
#[syscall(nr = 203, abi = "sysv")]
pub fn sys_sched_setaffinity(pid: pid_t,
                             cpusetsize: usize,
                             umask: UserPtr<u8>) -> SysResult<i32> {
    SchedSetaffinity::do_call(pid, cpusetsize, umask)
}
```

`SchedSetaffinity::do_call(pid, cpusetsize, umask) -> i32`:
1. if cpusetsize > MAX_USER_CPUSET_BYTES { return -EINVAL; }
2. let need = (nr_cpu_ids() + 7) / 8;
3. if cpusetsize < need { return -EINVAL; }
4. /* Allocate kernel cpumask, copy in, validate tail */
5. let mut cpumask = CpuMask::alloc().ok_or(-ENOMEM)?;
6. let nbytes = min(cpusetsize, cpumask.bytes_len());
7. copy_from_user_slice(cpumask.as_bytes_mut(), umask.as_bytes(), nbytes)?;  // -EFAULT
8. if cpusetsize > cpumask.bytes_len() {
       /* require tail bytes to be zero */
       let mut tail = [0u8; 64];
       for off in (cpumask.bytes_len()..cpusetsize).step_by(tail.len()) {
           let n = min(tail.len(), cpusetsize - off);
           copy_from_user_slice(&mut tail[..n], umask.as_bytes() + off, n)?;
           if tail[..n].iter().any(|&b| b != 0) { return -EINVAL; }
       }
   }
9. /* Mask off bits beyond nr_cpu_ids (defense in depth) */
10. cpumask.clear_above(nr_cpu_ids());
11. /* Resolve task */
12. let task = if pid == 0 { current() } else { find_task_by_vpid(pid).ok_or(-ESRCH)? };
13. /* Permission */
14. SchedSetaffinity::check_perm(task)?;
15. security_task_setscheduler(task)?;
16. /* Apply */
17. Sched::__set_cpus_allowed(task, &cpumask)

`SchedSetaffinity::check_perm(task) -> Result<()>`:
1. if task.real_cred.uid == current().euid || task.real_cred.uid == current().uid { return Ok(()); }
2. if capable(CAP_SYS_NICE) { return Ok(()); }
3. Err(-EPERM)

`Sched::__set_cpus_allowed(task, mask) -> i32`:
1. let mut effective = mask.clone();
2. effective.intersect(&task.cpuset_cpus_allowed());
3. effective.intersect(&cpu_active_mask());
4. if effective.is_empty() { return -EINVAL; }
5. /* For SCHED_DEADLINE: validate admission against effective */
6. if task.policy == SCHED_DEADLINE && !dl_admission_test(task, &effective) { return -EBUSY; }
7. /* Store requested + effective */
8. task.cpus_requested = mask.clone();
9. /* Atomic switch under task_rq_lock */
10. let rq = task_rq_lock(task);
11. task.cpus_mask = effective;
12. task.cpus_ptr = &task.cpus_mask;
13. if !task.cpus_mask.is_set(task_cpu(task)) {
        Sched::migrate_task_to_allowed_cpu(task);  // queues a stop_one_cpu work
    }
14. task_rq_unlock(rq);
15. 0

### Out of Scope

- `sched_getaffinity(2)` (Tier-5 separate doc).
- cpuset cgroup controller (Tier-3 in `kernel/cgroup-cpuset.md`).
- Load balancer / migration internals (Tier-3 in `kernel/sched-core.md`).
- IRQ-thread affinity (`/proc/irq/N/smp_affinity`) (Tier-3 in `kernel/irq.md`).
- Implementation code.

### signature

```c
int sched_setaffinity(pid_t pid, size_t cpusetsize,
                      const cpu_set_t *mask);
```

### parameters

| Parameter | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | Target task PID; 0 = self. |
| `cpusetsize` | `size_t` | in | Size of the mask buffer in bytes (must cover at least nr_cpu_ids bits). |
| `mask` | `const cpu_set_t *` | in (UAPI) | Bitmap of allowed CPUs; read via copy_from_user. |

### return value

| Value | Meaning |
|---|---|
| 0 | Success; task's cpus_allowed updated. |
| `-EINVAL` | cpusetsize < required, mask empty after intersection with cpu_active_mask. |
| `-EPERM` | Caller lacks CAP_SYS_NICE for foreign task / forbidden CPUs. |
| `-ESRCH` | PID not in caller's pid-ns. |
| `-EFAULT` | mask unreadable. |
| `-ENOMEM` | Kernel cpumask alloc failure. |

### errors

| Errno | Trigger |
|---|---|
| `EINVAL` | cpusetsize too small; mask & cpu_possible_mask is empty; mask & (cpu_active_mask ∩ cpus_allowed_by_cgroup) is empty. |
| `EPERM` | Foreign-uid task without CAP_SYS_NICE; CPUs forbidden by cpuset hierarchy. |
| `ESRCH` | PID not found / cross-ns. |
| `EFAULT` | mask unreadable. |
| `ENOMEM` | alloc_cpumask_var failed. |

### abi surface

```text
__NR_sched_setaffinity (x86_64) = 203
__NR_sched_setaffinity (i386)   = 241
__NR_sched_setaffinity (arm64)  = 122  (generic-syscall)
__NR_sched_setaffinity (generic) = 122

/* glibc: */
typedef struct {
    __cpu_mask __bits[CPU_SETSIZE / NCPUBITS];   /* 1024 bits default = 128 bytes */
} cpu_set_t;

/* Kernel reads at most min(cpusetsize, nr_cpu_ids/8 + 1) bytes. */
/* If cpusetsize > kernel's nr_cpu_ids/8, the kernel checks that the tail bits beyond nr_cpu_ids are all zero — else -EINVAL. */
```

### compatibility contract

REQ-1: Syscall number is **203** on x86_64; **122** on arm64/generic.

REQ-2: pid == 0 ⟹ current.

REQ-3: cpusetsize must be ≥ `nr_cpu_ids / 8` (rounded up); smaller ⟹ -EINVAL.

REQ-4: The kernel copies in min(cpusetsize, max_cpumask_bytes) bytes. Bits beyond nr_cpu_ids MUST be zero — non-zero ⟹ -EINVAL (some kernels are lenient and zero-ignore; Rookery enforces strict).

REQ-5: After validating the user mask, the kernel intersects with `cpu_active_mask` AND with the task's `cpuset->cpus_allowed`. If the result is empty ⟹ -EINVAL.

REQ-6: The new effective mask becomes `task->cpus_ptr` (and `task->cpus_mask`). The "requested" mask is stored separately so cpuset changes can re-derive the effective mask.

REQ-7: If the task is currently running on a CPU not in the new mask, it is migrated to a permitted CPU before the syscall returns. Migration uses the standard load-balancer hooks.

REQ-8: Permission: CAP_SYS_NICE in the caller's user namespace OR (caller's euid == target's euid OR target's real_cred uid) — i.e., task-owner may set own affinity without CAP_SYS_NICE.

REQ-9: For SCHED_DEADLINE tasks, sched_setaffinity must NOT shrink the mask below what the deadline admission requires; if doing so would cause -EBUSY, the kernel returns -EBUSY (kernel ≥ 4.5).

REQ-10: LSM hook `security_task_setscheduler(task)` fires (yes, even for affinity).

REQ-11: cpuset cgroup intersection: the kernel honors `cpuset.cpus` of the task's cgroup. User cannot grant access to CPUs outside the cgroup — they are silently masked off (Rookery follows upstream which returns -EINVAL only if the intersection is empty).

REQ-12: For isolated CPUs (`isolcpus=`), userspace MAY set affinity to them; this is a feature not a bug.

REQ-13: cpusetsize > MAX_USER_CPUSET_BYTES (4096) ⟹ -EINVAL.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cpusetsize_lower_bound` | INVARIANT | per-call: cpusetsize ≥ nr_cpu_ids/8 or -EINVAL. |
| `cpusetsize_upper_bound` | INVARIANT | per-call: cpusetsize ≤ 4096 or -EINVAL. |
| `tail_zero_strict` | INVARIANT | per-call: bytes beyond kernel cpumask must be zero. |
| `mask_intersection_nonempty` | INVARIANT | per-call: effective mask non-empty before commit. |
| `rq_lock_held_during_switch` | INVARIANT | per-set_cpus_allowed: rq lock held over field write. |
| `dl_admission_atomic` | INVARIANT | per-DEADLINE: admission re-tested under new mask. |
| `lsm_hook_invoked` | INVARIANT | per-call: security_task_setscheduler called. |

### Layer 2: TLA+

`kernel/sched_setaffinity.tla`:
- States: per-task cpus_mask, per-CPU active set, per-cgroup cpuset.
- Properties:
  - `safety_effective_subset_of_active` — per-task: cpus_mask ⊆ cpu_active_mask.
  - `safety_effective_subset_of_cgroup` — per-task: cpus_mask ⊆ cpuset_cpus_allowed.
  - `safety_no_partial_state` — per-call: failure leaves task unchanged.
  - `liveness_migration_completes` — per-call: if running CPU outside new mask, task migrates eventually.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: cpusetsize bounds enforced before any alloc | `SchedSetaffinity::do_call` |
| `__set_cpus_allowed` post: effective = req ∩ cpuset ∩ active; non-empty | `Sched::__set_cpus_allowed` |
| `dl_admission_test` post: deadline tasks always admissible on new mask | `Sched::dl_admission_test` |

### Layer 4: Verus / Creusot functional

Per-`sched_setaffinity(2)` man-page equivalence. LTP `sched_setaffinity01..03` pass. cpuset cgroup interaction matches `Documentation/admin-guide/cgroup-v1/cpusets.rst`.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sched_setaffinity(2)` reinforcement:

- **Per-cpusetsize bounds** — defense against per-OOB-read and per-huge-alloc DoS.
- **Per-tail-zero strict** — defense against per-stale-bit-set hidden in oversized user buffer.
- **Per-intersection-non-empty pre-commit** — defense against per-task-stranded-on-no-CPU.
- **Per-rq-lock atomic switch** — defense against per-half-updated mask raced by load balancer.
- **Per-DEADLINE admission re-check** — defense against per-deadline task admitted under one mask running on incompatible new mask.
- **Per-LSM hook pre-commit** — defense against per-affinity bypass of MAC policy.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `mask` (cpuset)** — copy_from_user_slice validates the source is in userspace; UDEREF blocks per-kernel-pointer smuggling that would let an attacker pass a kernel address as the cpumask.
- **CAP_SYS_NICE strict for foreign tasks** — grsec retains the upstream rule: foreign-uid target requires CAP_SYS_NICE. Under GRKERNSEC_HARDEN, even own-task pinning to isolated CPUs (`isolcpus=`) requires CAP_SYS_NICE to prevent RT-isolation breakouts.
- **GRKERNSEC_RESLOG on rlimit-like violations** — denied affinity changes (EPERM or empty-intersection EINVAL) are logged with task name/uid/target pid/requested mask.
- **Sched_deadline attack-surface reduction** — under sandbox flags, sched_setaffinity on a SCHED_DEADLINE task is denied with -EPERM regardless of capability; the task must demote first.
- **GRKERNSEC_CHROOT_CHMOD/SCHED** — inside a grsec chroot, sched_setaffinity is permitted only for tasks inside the same chroot's task set; cross-chroot returns -ESRCH.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset per call; the tail-zero validation loop has bounded iteration so RANDKSTACK is not destabilized.
- **No_new_privs interaction** — when current has PR_SET_NO_NEW_PRIVS, sched_setaffinity is permitted (no privilege elevation) but the effective mask is intersected with the sandbox's per-task allowed CPU set (a Rookery extension that survives no_new_privs).
- **CPU-set restriction under sandbox** — grsec adds a per-uid CPU-set quota: unprivileged users cannot pin to more than N CPUs at once (configurable), preventing per-uid hot-spot DoS on isolated CPU partitions.
- **KEEPCAPS aware** — capability check honors effective set.
- **Bounded user buffer** — cpusetsize is capped at MAX_USER_CPUSET_BYTES (4096) so the tail-zero validation cannot be weaponized into a long copy_from_user.

