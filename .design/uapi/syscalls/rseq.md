# Tier-5 syscall: rseq(2) — syscall 334

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/rseq.c (SYSCALL_DEFINE4(rseq, ...), rseq_handle_notify_resume, __rseq_handle_notify_resume)
  - include/uapi/linux/rseq.h (struct rseq, RSEQ_FLAG_*, RSEQ_SIG)
  - include/linux/sched.h (task_struct.rseq, task_struct.rseq_sig, task_struct.rseq_event_mask)
  - arch/x86/entry/syscalls/syscall_64.tbl (334  common  rseq)
-->

## Summary

`rseq(2)` registers a **restartable sequence** descriptor for the calling thread. A restartable sequence is a user-defined critical section in a fixed virtual-address range; when the kernel detects that the thread was preempted, signal-delivered, or migrated between CPUs while the instruction pointer was inside that range, it rewinds the IP to a caller-specified abort handler on return-to-userspace. This lets glibc, jemalloc, tcmalloc, and Folly implement lock-free per-CPU data structures (per-CPU caches, sharded counters) without atomic RMW.

Critical for: glibc 2.35+ `__rseq_size`, jemalloc tcache, sched_ext scx_layered userland-driven scheduling, Folly per-CPU stats, libpthread fast paths, percpu-counter performance.

## Signature

```c
#include <linux/rseq.h>

int rseq(struct rseq *rseq, uint32_t rseq_len, int flags, uint32_t sig);

struct rseq {
    __u32 cpu_id_start;      /* user-readable hint */
    __u32 cpu_id;            /* user-readable current CPU */
    __u64 rseq_cs;           /* user-writable: address of struct rseq_cs (current critical section) */
    __u32 flags;
    __u32 node_id;           /* NUMA node */
    __u32 mm_cid;            /* concurrency ID — sched_ext */
    /* padding to 32 bytes minimum */
};

struct rseq_cs {
    __u32 version;
    __u32 flags;
    __u64 start_ip;
    __u64 post_commit_offset;
    __u64 abort_ip;
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `rseq` | `struct rseq *` | in/out | User-space descriptor in the calling thread's address space. Kernel writes cpu_id/cpu_id_start/node_id/mm_cid on context switch and migration. Aligned 32 bytes. |
| `rseq_len` | `uint32_t` | in | sizeof(*rseq); used for forward-compat extension probing. Must match the kernel's recognized layout (currently 32). |
| `flags` | `int` | in | 0 to register; `RSEQ_FLAG_UNREGISTER` to detach. Other bits reserved. |
| `sig` | `uint32_t` | in | Caller-chosen 32-bit signature. The 4 bytes **immediately preceding** `abort_ip` of every rseq_cs MUST equal this value; the kernel refuses to redirect IP to an abort handler whose signature does not match. Defense against control-flow hijacking via attacker-controlled rseq_cs. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success: rseq registered or unregistered. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | rseq_len mismatch, flags has unknown bits, sig changed across registrations, rseq pointer not 32-byte aligned, NULL rseq with flags=0. |
| `EBUSY` | Thread already has a different rseq registered (need UNREGISTER first or the same {ptr,len,sig} tuple). |
| `EPERM` | UNREGISTER with rseq pointer or sig not matching current registration. |
| `EFAULT` | rseq pointer faults on copy. |
| `ENOSYS` | Kernel built without CONFIG_RSEQ. |

## ABI surface

```text
__NR_rseq  (x86_64)   = 334
__NR_rseq  (arm64)    = 293
__NR_rseq  (riscv64)  = 293
__NR_rseq  (i386)     = 386

RSEQ_FLAG_UNREGISTER  = 1
RSEQ_SIG_DEFAULT_x86  = 0x53053053   /* arbitrary; matches glibc */

struct rseq alignment: 32 bytes.

task_struct fields:
  struct rseq __user  *rseq;
  u32                  rseq_len;
  u32                  rseq_sig;
  unsigned long        rseq_event_mask;
```

## Compatibility contract

REQ-1: Syscall number is **334** on x86_64; **293** on arm64/riscv64. ABI-stable since Linux 4.18.

REQ-2: Register: `flags == 0`, rseq != NULL, 32-byte aligned, rseq_len matches kernel layout, sig stored in task_struct.

REQ-3: Unregister: `flags == RSEQ_FLAG_UNREGISTER`, rseq pointer and sig MUST match registration tuple; mismatch ⟹ EPERM.

REQ-4: After registration, on every return-to-userspace where TIF_NOTIFY_RESUME is set, the kernel:
  - Updates `rseq->cpu_id_start`, `rseq->cpu_id`, `rseq->node_id`, `rseq->mm_cid`.
  - If `rseq->rseq_cs != 0`: dereferences the user-supplied `rseq_cs`, validates that `IP ∈ [start_ip, start_ip + post_commit_offset)`. If true: validates that the 4 bytes at `(abort_ip - 4)` equal `task->rseq_sig`. If true: rewrites IP to `abort_ip`. Then zeros `rseq->rseq_cs`.

REQ-5: Signature validation: the 4 bytes preceding `abort_ip` MUST equal `sig`. This is enforced by the kernel before any IP rewrite. Mismatch ⟹ SIGSEGV delivered to the thread (defense against attacker-supplied rseq_cs redirecting flow).

REQ-6: The kernel writes `cpu_id`/`cpu_id_start` atomically (single store) on resume; userspace reads these without a lock. The store ordering guarantees: after `cpu_id_start == cpu_id`, userspace is on that CPU and has not been preempted since the last fence; if the values differ on read, the thread was migrated.

REQ-7: `rseq_cs` field is user-writable; userspace sets it before entering a critical section and the kernel clears it after observing it (one-shot). This avoids requiring user code to clear it after success.

REQ-8: On `clone(CLONE_THREAD)`: child inherits `rseq` pointer + sig from parent ONLY if both threads explicitly opt in; otherwise child starts unregistered. Per the kernel implementation, `rseq` is inherited by default for CLONE_THREAD when the parent has it set, with child needing to confirm via its own register.

REQ-9: `fork()`: child does NOT inherit rseq registration (separate mm; pointer would be invalid anyway in a new address space).

REQ-10: `execve()`: rseq registration is cleared on successful exec (new address space).

REQ-11: Per-thread: each thread has its own rseq descriptor; pointers must lie within the thread-private TLS-or-equivalent region.

REQ-12: Per-glibc 2.35+: glibc auto-registers an rseq for every pthread during thread creation; programs read CPU id via `__rseq_size > 0` and `__rseq_offset`.

REQ-13: `RSEQ_FLAG_*` reserved bits: kernel rejects any flag outside `RSEQ_FLAG_UNREGISTER` with EINVAL.

REQ-14: rseq pointer must remain valid for the lifetime of the registration (until unregister or thread exit). Page-faulting on rseq read in resume path: kernel delivers SIGSEGV to the thread.

## Acceptance Criteria

- [ ] AC-1: rseq(rseq_ptr, 32, 0, 0x53053053): registers; subsequent reads of rseq_ptr->cpu_id return current CPU.
- [ ] AC-2: rseq(rseq_ptr, 32, RSEQ_FLAG_UNREGISTER, 0x53053053): unregisters; CPU id updates stop.
- [ ] AC-3: rseq(rseq_ptr2, 32, 0, sig) after registration with rseq_ptr1: -EBUSY.
- [ ] AC-4: rseq(NULL, 32, 0, 0): -EINVAL.
- [ ] AC-5: rseq(rseq_ptr, 16, 0, 0): -EINVAL (size mismatch).
- [ ] AC-6: rseq pointer unaligned: -EINVAL.
- [ ] AC-7: Unknown flag bit set: -EINVAL.
- [ ] AC-8: Unregister with mismatched sig: -EPERM.
- [ ] AC-9: Critical section IP-in-range + valid sig at (abort_ip - 4): preemption rewrites IP to abort_ip.
- [ ] AC-10: Critical section with sig mismatch at (abort_ip - 4): SIGSEGV delivered.
- [ ] AC-11: After execve: rseq registration cleared.
- [ ] AC-12: Read cpu_id_start, do work, read cpu_id; if equal: migration did not occur between reads.
- [ ] AC-13: Concurrent N threads each register their own rseq: all succeed.

## Architecture

```rust
#[syscall(nr = 334, abi = "sysv")]
pub fn sys_rseq(rseq: UserPtr<Rseq>, rseq_len: u32, flags: i32, sig: u32) -> isize {
    Rseq::do_rseq(rseq, rseq_len, flags, sig)
}
```

`Rseq::do_rseq(rseq_ptr, rseq_len, flags, sig) -> isize`:
1. if flags & !RSEQ_FLAG_UNREGISTER != 0 { return Err(EINVAL); }
2. if rseq_len as usize != size_of::<Rseq>() { return Err(EINVAL); }
3. if (rseq_ptr.as_u64() % 32) != 0          { return Err(EINVAL); }
4. let task = current_mut();
5. if flags & RSEQ_FLAG_UNREGISTER != 0 {
6.   if task.rseq.is_none()             { return Err(EINVAL); }
7.   if task.rseq.unwrap() != rseq_ptr   { return Err(EPERM); }
8.   if task.rseq_sig != sig             { return Err(EPERM); }
9.   task.rseq = None;
10.  task.rseq_len = 0;
11.  task.rseq_sig = 0;
12.  return 0;
13. }
14. /* Register */
15. if task.rseq.is_some() {
16.   if task.rseq.unwrap() == rseq_ptr && task.rseq_len == rseq_len && task.rseq_sig == sig {
17.     return 0;     // idempotent
18.   }
19.   return Err(EBUSY);
20. }
21. /* Probe-read to validate accessibility */
22. rseq_ptr.probe_read_user(size_of::<Rseq>())?;     // EFAULT
23. task.rseq = Some(rseq_ptr);
24. task.rseq_len = rseq_len;
25. task.rseq_sig = sig;
26. task.set_tif(TIF_NOTIFY_RESUME);                  // ensure first update
27. 0

`Rseq::handle_notify_resume(task, regs)`:
1. if task.rseq.is_none() { return; }
2. let rseq = task.rseq.unwrap();
3. /* Update CPU/node/mm_cid */
4. let cpu = smp_processor_id();
5. let node = cpu_to_node(cpu);
6. let cid  = task.mm_cid_for_cpu(cpu);
7. rseq.write_field(rseq.cpu_id_start, cpu)
8.      .and_then(|_| rseq.write_field(rseq.cpu_id, cpu))
9.      .and_then(|_| rseq.write_field(rseq.node_id, node))
10.     .and_then(|_| rseq.write_field(rseq.mm_cid, cid))
11.     .or_else(|_| { Rseq::deliver_sigsegv(task); return; });
12. /* Examine rseq_cs */
13. let cs_ptr: UserPtr<RseqCs> = rseq.read_field(rseq.rseq_cs);
14. if cs_ptr.is_null() { return; }
15. let cs: RseqCs = match cs_ptr.copy_in_struct() {
16.   Ok(v) => v,
17.   Err(_) => { Rseq::deliver_sigsegv(task); return; }
18. };
19. let ip = regs.user_ip();
20. if ip < cs.start_ip || ip >= cs.start_ip + cs.post_commit_offset { return; }
21. /* Validate signature at (abort_ip - 4) */
22. let sig_at: u32 = match UserPtr::<u32>::from(cs.abort_ip - 4).copy_in_u32() {
23.   Ok(v) => v,
24.   Err(_) => { Rseq::deliver_sigsegv(task); return; }
25. };
26. if sig_at != task.rseq_sig { Rseq::deliver_sigsegv(task); return; }
27. /* Redirect IP */
28. regs.set_user_ip(cs.abort_ip);
29. /* Clear rseq_cs (one-shot) */
30. rseq.write_field(rseq.rseq_cs, 0);

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rseq_alignment_check` | INVARIANT | rseq_ptr % 32 == 0 enforced. |
| `flags_bits_validated` | INVARIANT | unknown flags ⟹ EINVAL. |
| `signature_match_before_redirect` | INVARIANT | IP rewrite requires sig_at == task.rseq_sig. |
| `unregister_requires_match` | INVARIANT | EPERM if ptr or sig mismatch on unregister. |
| `idempotent_register` | INVARIANT | re-registering same {ptr,len,sig} returns 0. |
| `register_busy_on_change` | INVARIANT | different {ptr,len,sig} from active ⟹ EBUSY. |
| `signature_fetch_user_only` | INVARIANT | sig_at is fetched via copy_from_user, never trusted from kernel side. |

### Layer 2: TLA+

`kernel/rseq.tla`:
- States: per-register, per-notify-resume, per-rseq_cs-eval, per-IP-redirect, per-unregister.
- Properties:
  - `safety_no_redirect_without_sig` — IP rewrite ⟹ signature match.
  - `safety_register_atomic` — register set {ptr,len,sig} atomically or not at all.
  - `safety_unregister_match` — unregister ⟹ matching ptr and sig.
  - `safety_one_shot_rseq_cs` — rseq_cs zeroed after IP redirect.
  - `liveness_eventual_update` — registered ⟹ on next resume, cpu_id is written.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_rseq` post: success register ⟹ task.rseq == ptr ∧ task.rseq_sig == sig | `Rseq::do_rseq` |
| `handle_notify_resume` post: IP redirect ⟹ user u32 at (abort_ip - 4) == task.rseq_sig | `Rseq::handle_notify_resume` |
| `unregister` post: task.rseq == None | `Rseq::do_rseq` |

### Layer 4: Verus / Creusot functional

Per-`rseq(2)` man-page; tools/testing/selftests/rseq/ pass; glibc rseq tests pass. POSIX is silent; Linux-only.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`rseq(2)` reinforcement:

- **Per-signature validation** — defense against per-attacker-controlled abort_ip redirecting control flow to arbitrary userspace gadget.
- **Per-alignment enforcement** — defense against per-misaligned-struct corruption.
- **Per-rseq_cs one-shot** — defense against per-stale-cs replay.
- **Per-EBUSY on tuple change** — defense against per-rseq-takeover by injected code.
- **Per-EPERM on unregister mismatch** — defense against per-spurious-unregister.
- **Per-EFAULT on rseq-page fault** — defense against per-unmapped-rseq invariant violation.

## Grsecurity / PaX surface

- **rseq signature == stack-canary-style integrity tag** — grsec treats `task->rseq_sig` like a stack-canary: any rseq_cs whose 4 bytes preceding `abort_ip` mismatch is treated as an attempted CFI bypass and delivers SIGSEGV plus an audit record.
- **PaX UDEREF on rseq + rseq_cs copy_from_user** — SMAP-enforced; all reads of user-supplied descriptors go through the audited copy_from_user path.
- **PAX_MPROTECT integration** — the page containing `abort_ip` MUST be executable; PaX-MPROTECT W^X invariant must hold or the kernel refuses the redirect (otherwise an attacker could write a sig+gadget into RW data).
- **GRKERNSEC_KSTACKOVERFLOW on resume path** — rseq notify-resume runs on a tight stack budget; oversized signal frames or rseq pages near stack are guarded.
- **PaX KERNEXEC neutral** — rseq operates wholly on user IP; kernel-text protection unaffected.
- **GRKERNSEC_HIDESYM** — task->rseq_sig not exposed via /proc; defense against per-leak of the per-thread tag.
- **Per-CLONE_THREAD inheritance gated** — child threads inherit only if RSEQ config explicitly allows; grsec can disable inheritance via toggle.
- **PAX_REFCOUNT neutral** — rseq has no refcount; lifetime tied to task_struct.
- **No_new_privs neutral** — rseq is not a privilege boundary.
- **GRKERNSEC_AUDIT_RSEQ** — every rseq register/unregister auditable for the target group.
- **CAP requirement: none** — rseq is unprivileged; grsec preserves this but adds audit + signature integrity checks.
- **Per-rseq_len strict match** — grsec rejects size != sizeof(struct rseq) even if kernel-side forward-compat is otherwise permissive, to prevent struct-layout-confusion attacks.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- mm_cid / sched_ext concurrency-ID semantics (Tier-3 `kernel/sched/cid.md`).
- glibc auto-registration (libc-side, not kernel).
- Per-arch rseq notify-resume hooks (Tier-3 per arch).
- Implementation code.
