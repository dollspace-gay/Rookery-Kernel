---
title: "Tier-5 syscall: sched_getaffinity(2) — syscall 204"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`sched_getaffinity(2)` reads the **effective CPU affinity mask** of a target task (PID 0 = self) into a userspace bitmap. The kernel writes the intersection of the task's `cpus_mask`, the active CPUs (`cpu_active_mask`), and the cgroup cpuset bound; this is the actual set of CPUs the scheduler will pick from. The mask is written as raw little-endian-bit bitmap of `cpusetsize` bytes (must be ≥ nr_cpu_ids/8). Critical for: NUMA-aware libraries (libnuma, OpenMP), `lscpu`/`taskset` introspection, container CPU-set audit, Rookery's sandbox auditor verifying that a task didn't escape its CPU partition, and the `nproc` userspace helper.

### Acceptance Criteria

- [ ] AC-1: After `sched_setaffinity(0, sizeof(m), &m_cpu0_only)`, `sched_getaffinity(0, sizeof(out), &out)` returns out with only bit 0 set.
- [ ] AC-2: `sched_getaffinity(0, 0, &out)` → -EINVAL.
- [ ] AC-3: `sched_getaffinity(0, 7, &out)` (not multiple of sizeof(long) on 64-bit) → -EINVAL.
- [ ] AC-4: `sched_getaffinity(0, sizeof(out), NULL)` → -EFAULT.
- [ ] AC-5: `sched_getaffinity(99999, sizeof(out), &out)` → -ESRCH.
- [ ] AC-6: `sched_getaffinity(pid_in_other_ns, sizeof(out), &out)` → -ESRCH.
- [ ] AC-7: Return value is the number of bytes copied (e.g., 8 on a uniprocessor 64-bit system).
- [ ] AC-8: Mask returned ⊆ `cpu_active_mask`.
- [ ] AC-9: Mask returned ⊆ task's cgroup cpuset.
- [ ] AC-10: 1e6 concurrent calls return consistent values.
- [ ] AC-11: After CPU hot-unplug of CPU N from the task's mask, `sched_getaffinity` returns mask without bit N.
- [ ] AC-12: LSM denial → -EPERM/-EACCES.

### Architecture

```rust
#[syscall(nr = 204, abi = "sysv")]
pub fn sys_sched_getaffinity(pid: pid_t,
                             cpusetsize: usize,
                             umask: UserPtrMut<u8>) -> SysResult<i32> {
    SchedGetaffinity::do_call(pid, cpusetsize, umask)
}
```

`SchedGetaffinity::do_call(pid, cpusetsize, umask) -> i32`:
1. if cpusetsize > MAX_USER_CPUSET_BYTES { return -EINVAL; }
2. if cpusetsize % size_of::<usize>() != 0 { return -EINVAL; }
3. let need = ((nr_cpu_ids() + (8*size_of::<usize>() - 1)) / (8*size_of::<usize>())) * size_of::<usize>();
4. if cpusetsize < need { return -EINVAL; }
5. rcu_read_lock();
6. let task = if pid == 0 { current() } else {
       match find_task_by_vpid(pid) { Some(t)=>t, None=>{ rcu_read_unlock(); return -ESRCH; } }
   };
7. if let Err(e) = security_task_getscheduler(task) { rcu_read_unlock(); return e; }
8. /* Snapshot effective mask under task pi_lock */
9. let mut k_mask = CpuMask::zeroed();
10. {
        let _g = task.pi_lock();
        k_mask.copy_from(&task.cpus_mask);
    }
11. /* Intersect with cpu_active_mask (defensive; kernel keeps cpus_mask consistent) */
12. k_mask.intersect(&cpu_active_mask());
13. rcu_read_unlock();
14. let copy_n = min(cpusetsize, k_mask.bytes_len());
15. copy_to_user_slice(umask.as_bytes_mut(), k_mask.as_bytes(), copy_n)?;  // -EFAULT
16. copy_n as i32

### Out of Scope

- `sched_setaffinity(2)` (Tier-5 separate doc).
- cpuset cgroup controller (Tier-3 in `kernel/cgroup-cpuset.md`).
- Load balancer / migration internals (Tier-3 in `kernel/sched-core.md`).
- Implementation code.

### signature

```c
int sched_getaffinity(pid_t pid, size_t cpusetsize, cpu_set_t *mask);
```

### parameters

| Parameter | Type | Direction | Description |
|---|---|---|---|
| `pid` | `pid_t` | in | Target PID (caller's active pid-ns); 0 = self. |
| `cpusetsize` | `size_t` | in | Size of mask buffer in bytes. |
| `mask` | `cpu_set_t *` | out (UAPI) | Buffer to receive the affinity bitmap. |

### return value

| Value | Meaning |
|---|---|
| `> 0` | Number of bytes written (the kernel returns the number of bytes copied, i.e., `min(cpusetsize, kernel_cpumask_bytes)`). |
| `-EINVAL` | cpusetsize < nr_cpu_ids/8 (rounded up) or not a multiple of sizeof(long). |
| `-ESRCH` | PID not in caller's pid-ns. |
| `-EFAULT` | mask unwritable. |
| `-EPERM` | LSM denial. |

### errors

| Errno | Trigger |
|---|---|
| `EINVAL` | cpusetsize < required; cpusetsize not multiple of sizeof(long). |
| `ESRCH` | PID not found / cross-ns. |
| `EFAULT` | mask buffer unwritable. |
| `EPERM` | security_task_getscheduler denial. |

### abi surface

```text
__NR_sched_getaffinity (x86_64) = 204
__NR_sched_getaffinity (i386)   = 242
__NR_sched_getaffinity (arm64)  = 123  (generic-syscall)
__NR_sched_getaffinity (generic) = 123

/* glibc cpu_set_t: 128-byte / 1024-bit bitmap */
/* Kernel writes exactly nr_cpu_ids bits, rounded up to a long boundary. */
/* Tail (cpusetsize - kernel_bytes) is NOT zeroed by kernel — glibc CPU_ZERO before call. */
```

### compatibility contract

REQ-1: Syscall number is **204** on x86_64; **123** on arm64/generic.

REQ-2: pid == 0 ⟹ current.

REQ-3: cpusetsize MUST be ≥ `nr_cpu_ids / 8` (rounded up to a long); smaller ⟹ -EINVAL.

REQ-4: cpusetsize MUST be a multiple of `sizeof(unsigned long)`; otherwise -EINVAL.

REQ-5: The kernel writes its full internal cpumask (size = `BITS_TO_LONGS(nr_cpu_ids) * sizeof(long)`); if cpusetsize > kernel_bytes, the kernel does NOT clear the tail — glibc's `sched_getaffinity` wrapper zeros the buffer first.

REQ-6: The mask written is the **effective** mask: `task->cpus_mask` AS-IS (which is already the intersection of requested + cgroup + active).

REQ-7: The return value is the number of bytes the kernel actually copied (NOT 0 on success); historically this is min(cpusetsize, kernel_bytes). glibc converts to 0/-1.

REQ-8: For any visible PID, no capability is required to read.

REQ-9: LSM hook `security_task_getscheduler(task)` fires.

REQ-10: The call holds RCU read lock; mask read is performed under task->pi_lock or a stable snapshot pattern to avoid races with set_cpus_allowed.

REQ-11: cpusetsize > MAX_USER_CPUSET_BYTES (4096) ⟹ -EINVAL.

REQ-12: After `sched_setaffinity` succeeds with mask M, `sched_getaffinity` returns M ∩ active ∩ cgroup, which may be a subset of M.

REQ-13: For a task that has been migrated due to CPU hot-unplug, the mask reflects the post-hotplug effective set.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cpusetsize_alignment` | INVARIANT | per-call: cpusetsize % sizeof(long) == 0. |
| `cpusetsize_lower_bound` | INVARIANT | per-call: cpusetsize ≥ ceil(nr_cpu_ids/8) rounded to long. |
| `cpusetsize_upper_bound` | INVARIANT | per-call: cpusetsize ≤ MAX_USER_CPUSET_BYTES. |
| `rcu_balanced` | INVARIANT | per-call: rcu_read_lock paired on all paths. |
| `pi_lock_snapshot` | INVARIANT | per-call: cpus_mask read under pi_lock for stable snapshot. |
| `no_mutation` | INVARIANT | per-call: no task field written. |
| `lsm_hook_invoked` | INVARIANT | per-call: security_task_getscheduler called pre-copy. |

### Layer 2: TLA+

`kernel/sched_getaffinity.tla`:
- States: per-task cpus_mask + active set.
- Properties:
  - `safety_returns_subset_of_active` — per-call: written mask ⊆ cpu_active_mask.
  - `safety_no_mutation` — per-call: no state change.
  - `safety_lsm_pre_copy` — per-call: LSM denial ⟹ no copy_to_user.
  - `liveness_terminates` — per-call: O(BITS_TO_LONGS(nr_cpu_ids)).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_call` post: ret > 0 ⟹ ret == min(cpusetsize, kbytes) | `SchedGetaffinity::do_call` |
| `pi_lock_snapshot` post: mask read is consistent | `Sched::snapshot_cpus_mask` |
| LSM hook fires before any copy_to_user | `Sched::lsm_hook` |

### Layer 4: Verus / Creusot functional

Per-`sched_getaffinity(2)` man-page equivalence. LTP `sched_getaffinity01..02` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sched_getaffinity(2)` reinforcement:

- **Per-cpusetsize bounds** — defense against per-OOB-write and per-huge-buffer DoS.
- **Per-pi_lock snapshot** — defense against per-torn-read while another CPU calls set_cpus_allowed.
- **Per-LSM hook pre-copy** — defense against per-info-leak when LSM denies.
- **Per-RCU-balanced** — defense against per-leak on error paths.
- **Per-zeroed kernel cpumask** — defense against per-info-leak of stack residue.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `mask` (cpuset)** — copy_to_user_slice validates that mask is in userspace; UDEREF blocks per-kernel-pointer smuggling that would let an attacker overwrite kernel memory via a forged cpu_set_t pointer.
- **No CAP for read** — POSIX-compliant; grsec respects GRKERNSEC_PROC_USERGROUP so cross-user queries return -ESRCH not -EPERM to avoid PID-existence leakage.
- **GRKERNSEC_RESLOG on enumeration** — repeated affinity queries on tasks the caller does not own can be audited under GRKERNSEC_AUDIT_TASK to detect reconnaissance.
- **PAX_RANDKSTACK at syscall entry** — randomizes kernel stack offset; combined with zero-initialized k_mask hardens against speculative residue info-leaks.
- **GRKERNSEC_CHROOT_FINDTASK ns isolation** — inside a chroot, find_task_by_vpid restricted to the chroot's task set; outside returns -ESRCH.
- **Sched_deadline attack-surface reduction** — querying a SCHED_DEADLINE task's affinity is permitted but audited under sandbox.
- **No_new_privs neutral** — read-only call; NNP has no effect.
- **KEEPCAPS neutral** — no capability check.
- **Stack residue zeroing** — k_mask is zero-initialized via CpuMask::zeroed() before populating; grsec ensures the entire copy_to_user buffer comes from a freshly-zeroed kernel allocation, never from prior stack state.
- **Bounded user buffer** — cpusetsize is capped at MAX_USER_CPUSET_BYTES (4096); grsec further clamps to (nr_cpu_ids+7)/8 + 8 to minimize info-leak surface even when userspace requests a huge buffer.

