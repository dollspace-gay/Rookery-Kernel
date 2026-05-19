# Tier-5 syscall: membarrier(2) — syscall 324

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/sched/membarrier.c (SYSCALL_DEFINE3(membarrier), sync_runqueues_membarrier_state)
  - kernel/sched/core.c (smp_call_function_many, IPI dispatch)
  - include/uapi/linux/membarrier.h (enum membarrier_cmd, MEMBARRIER_CMD_*)
  - include/linux/mm_types.h (mm_struct::membarrier_state)
  - arch/x86/entry/syscalls/syscall_64.tbl (324 common membarrier)
-->

## Summary

`membarrier(2)` is a low-cost, kernel-mediated memory barrier primitive: it issues a memory barrier on every CPU running threads of the caller's mm (or a chosen subset), without the userspace needing to issue an `mfence` itself on each access. It is intended for asymmetric synchronisation patterns where one side runs frequently in a hot path (and wants to omit fences) and the other side runs rarely (and can pay the cost of a system call that broadcasts an IPI). Canonical use: garbage-collected userspace runtimes (Go, Java) and `librseq` rseq-fast-path readers.

The command set is queried (`MEMBARRIER_CMD_QUERY`) before use. Concrete barrier commands include global, private (mm-scoped), expedited, and sync-core variants. Critical for: rseq fast paths, Go runtime preemption signals, JIT/userspace lockless reclaim.

## Signature

```c
int membarrier(int cmd, unsigned int flags, int cpu_id);
```

```c
enum membarrier_cmd {
    MEMBARRIER_CMD_QUERY                                       = 0,
    MEMBARRIER_CMD_GLOBAL                                      = (1 << 0),
    MEMBARRIER_CMD_GLOBAL_EXPEDITED                            = (1 << 1),
    MEMBARRIER_CMD_REGISTER_GLOBAL_EXPEDITED                   = (1 << 2),
    MEMBARRIER_CMD_PRIVATE_EXPEDITED                           = (1 << 3),
    MEMBARRIER_CMD_REGISTER_PRIVATE_EXPEDITED                  = (1 << 4),
    MEMBARRIER_CMD_PRIVATE_EXPEDITED_SYNC_CORE                 = (1 << 5),
    MEMBARRIER_CMD_REGISTER_PRIVATE_EXPEDITED_SYNC_CORE        = (1 << 6),
    MEMBARRIER_CMD_PRIVATE_EXPEDITED_RSEQ                      = (1 << 7),
    MEMBARRIER_CMD_REGISTER_PRIVATE_EXPEDITED_RSEQ             = (1 << 8),
    MEMBARRIER_CMD_GET_REGISTRATIONS                           = (1 << 9),
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `cmd` | `int` | in | One of `MEMBARRIER_CMD_*`. |
| `flags` | `unsigned int` | in | Must be 0 unless `cmd` is `MEMBARRIER_CMD_*_EXPEDITED` and using `MEMBARRIER_CMD_FLAG_CPU` (1 << 0). |
| `cpu_id` | `int` | in | Target CPU when `flags & MEMBARRIER_CMD_FLAG_CPU`; else ignored. |

## Return value

| Value | Meaning |
|---|---|
| `MEMBARRIER_CMD_QUERY` | Bitmask of supported commands. |
| `MEMBARRIER_CMD_GET_REGISTRATIONS` | Bitmask of registrations on current mm. |
| `0` | Barrier issued / registration succeeded. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | Unknown `cmd`, non-zero reserved bits in `flags`, expedited-variant called without prior registration, `cpu_id < 0` while flag set, `cpu_id` not online. |
| `EPERM` | `MEMBARRIER_CMD_GLOBAL` issued while sysctl `kernel.numa_balancing` or similar gate disables membarrier (rare). |

## ABI surface

```text
__NR_membarrier  (x86_64) = 324
__NR_membarrier  (arm64)  = 283
__NR_membarrier  (riscv)  = 283
__NR_membarrier  (i386)   = 375

#define MEMBARRIER_CMD_FLAG_CPU  (1U << 0)
```

## Compatibility contract

REQ-1: Syscall number is **324** on x86_64. ABI-stable.

REQ-2: `MEMBARRIER_CMD_QUERY` returns the kernel-supported command bitmask without issuing a barrier.

REQ-3: `MEMBARRIER_CMD_GLOBAL` issues a full memory barrier (`smp_mb()`) effective on **all** running threads system-wide. Requires no registration. May use IPI broadcast or scheduler synchronization.

REQ-4: Expedited variants require prior `*_REGISTER_*` on the same mm. Calling expedited without registration returns `-EINVAL`. Registration is idempotent and persists for the lifetime of the mm.

REQ-5: `MEMBARRIER_CMD_PRIVATE_EXPEDITED` issues `smp_mb()` only on CPUs running threads of the caller's mm. Implementation: walks `cpu_online_mask`, sends IPI to CPUs whose `rq->curr->mm == current->mm`.

REQ-6: `MEMBARRIER_CMD_PRIVATE_EXPEDITED_SYNC_CORE` issues a core-serializing instruction (e.g. `iret` / `cpuid` on x86, ISB on arm64) on each targeted CPU; needed for JITed code to safely observe newly-written instructions.

REQ-7: `MEMBARRIER_CMD_PRIVATE_EXPEDITED_RSEQ` issues a barrier AND invalidates any rseq critical section currently running on each targeted CPU, allowing rseq commit semantics.

REQ-8: With `MEMBARRIER_CMD_FLAG_CPU`, the command is restricted to a single CPU (`cpu_id`); `cpu_id` must be `online` and the running thread there must belong to caller's mm (else no-op for that CPU). Without the flag, `cpu_id` is ignored.

REQ-9: `MEMBARRIER_CMD_GET_REGISTRATIONS` returns the bitmask of `*_REGISTER_*` commands previously issued on the current mm.

REQ-10: Registration updates `mm->membarrier_state` atomically. The scheduler reads this state on context switch to decide whether to issue a barrier on switch-in.

REQ-11: On context switch into a task whose mm has `MEMBARRIER_STATE_PRIVATE_EXPEDITED` set, the scheduler issues `smp_mb__after_switch_mm()` to ensure ordering between switch and any pending barrier-request.

REQ-12: Calls are RCU-safe (`rcu_read_lock` over the CPU iteration); no allocation in the IPI path.

REQ-13: `membarrier` is a sleeper from the caller's view but uses synchronous IPI; latency scales with number of targeted CPUs.

## Acceptance Criteria

- [ ] AC-1: `membarrier(MEMBARRIER_CMD_QUERY, 0, 0)` returns a non-zero bitmask containing at least `MEMBARRIER_CMD_GLOBAL`.
- [ ] AC-2: `membarrier(MEMBARRIER_CMD_PRIVATE_EXPEDITED, 0, 0)` without prior register: `-EINVAL`.
- [ ] AC-3: After `MEMBARRIER_CMD_REGISTER_PRIVATE_EXPEDITED`, the next `MEMBARRIER_CMD_PRIVATE_EXPEDITED` returns 0.
- [ ] AC-4: Test pattern: writer issues `store; membarrier(PRIVATE_EXPEDITED); store2`; reader on another CPU sees store before store2 even without its own fence — verified by litmus test.
- [ ] AC-5: `membarrier(0x12345678, 0, 0)`: `-EINVAL`.
- [ ] AC-6: `membarrier(MEMBARRIER_CMD_GLOBAL, 1, 0)`: `-EINVAL` (flags reserved).
- [ ] AC-7: With `MEMBARRIER_CMD_FLAG_CPU` and `cpu_id = -1`: `-EINVAL`.
- [ ] AC-8: `MEMBARRIER_CMD_PRIVATE_EXPEDITED_SYNC_CORE` issues ISB / cpuid on each CPU running an mm-thread.
- [ ] AC-9: `MEMBARRIER_CMD_PRIVATE_EXPEDITED_RSEQ` aborts rseq critical sections on targeted CPUs.
- [ ] AC-10: `MEMBARRIER_CMD_GET_REGISTRATIONS` returns bitmask matching prior register calls.
- [ ] AC-11: Context-switch into mm with PRIVATE_EXPEDITED registered: `smp_mb` issued by scheduler.

## Architecture

```rust
#[syscall(nr = 324, abi = "sysv")]
pub fn sys_membarrier(cmd: i32, flags: u32, cpu_id: i32) -> isize {
    Membarrier::do_membarrier(cmd, flags, cpu_id)
}
```

`Membarrier::do_membarrier(cmd, flags, cpu_id) -> isize`:
1. let cmd = MembarrierCmd::try_from(cmd).map_err(|_| EINVAL)?;
2. /* Validate flags. */
3. if (flags & !MEMBARRIER_CMD_FLAG_CPU) != 0 { return -EINVAL; }
4. if (flags & MEMBARRIER_CMD_FLAG_CPU) != 0 {
5.   if cpu_id < 0 || !cpu_online(cpu_id) { return -EINVAL; }
6. }
7. /* Dispatch. */
8. match cmd {
9.   Query => return Membarrier::supported_mask() as isize,
10.  GetRegistrations => return current().mm().membarrier_state.load(Acquire) as isize,
11.  Global => return Membarrier::do_global(),
12.  GlobalExpedited => Membarrier::require_registered(REG_GLOBAL_EXPEDITED)?; return Membarrier::do_global_expedited(),
13.  RegisterGlobalExpedited => return Membarrier::register(REG_GLOBAL_EXPEDITED),
14.  PrivateExpedited => Membarrier::require_registered(REG_PRIVATE_EXPEDITED)?; return Membarrier::do_private_expedited(flags, cpu_id, /*sync_core=*/false, /*rseq=*/false),
15.  RegisterPrivateExpedited => return Membarrier::register(REG_PRIVATE_EXPEDITED),
16.  PrivateExpeditedSyncCore => Membarrier::require_registered(REG_PRIVATE_EXPEDITED_SYNC_CORE)?; return Membarrier::do_private_expedited(flags, cpu_id, true, false),
17.  RegisterPrivateExpeditedSyncCore => return Membarrier::register(REG_PRIVATE_EXPEDITED_SYNC_CORE),
18.  PrivateExpeditedRseq => Membarrier::require_rseq_registered()?; Membarrier::require_registered(REG_PRIVATE_EXPEDITED_RSEQ)?; return Membarrier::do_private_expedited(flags, cpu_id, false, true),
19.  RegisterPrivateExpeditedRseq => Membarrier::require_rseq_registered()?; return Membarrier::register(REG_PRIVATE_EXPEDITED_RSEQ),
20. }

`Membarrier::do_private_expedited(flags, cpu_id, sync_core, rseq) -> isize`:
1. let mm = current().mm();
2. let mask = if (flags & MEMBARRIER_CMD_FLAG_CPU) != 0 {
3.   single_cpu_mask(cpu_id)
4. } else {
5.   /* Build mask of CPUs whose rq->curr->mm == mm. */
6.   Membarrier::cpus_running_mm(mm)
7. };
8. /* IPI handler issues smp_mb() (+ ISB/cpuid for sync_core; + rseq abort for rseq). */
9. smp_call_function_many(&mask, ipi_membarrier, IpiArgs { sync_core, rseq }, /*wait=*/1);
10. /* Local barrier */
11. smp_mb();
12. return 0;

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cmd_dispatch_total` | INVARIANT | every MembarrierCmd variant has one arm. |
| `flags_reserved_zero` | INVARIANT | flags & ~MEMBARRIER_CMD_FLAG_CPU ⟹ EINVAL. |
| `expedited_requires_register` | INVARIANT | expedited cmd without registered state ⟹ EINVAL. |
| `ipi_targets_only_mm_threads` | INVARIANT | private expedited: IPI mask ⊆ {CPUs with rq.curr.mm == mm}. |
| `barrier_after_ipi` | INVARIANT | local smp_mb issued after smp_call_function_many returns. |

### Layer 2: TLA+

`kernel/membarrier.tla`:
- States: per-mm registration set, per-CPU rq.curr.mm, per-IPI delivery, per-barrier issue.
- Properties:
  - `safety_register_before_expedited` — every expedited call has a corresponding prior register.
  - `safety_ipi_target_subset` — IPI mask ⊆ mm-running CPUs (∪ requested cpu_id).
  - `safety_barrier_paired` — every successful expedited call ⟹ smp_mb on each target CPU.
  - `liveness_ipi_completes` — bounded delivery.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_membarrier` post: Query returns bitmask | `Membarrier::do_membarrier` |
| `do_membarrier` post: expedited ⟹ registered | `Membarrier::do_membarrier` |
| `do_private_expedited` post: smp_mb on every target CPU | `Membarrier::do_private_expedited` |
| `register` post: mm.membarrier_state has bit set | `Membarrier::register` |

### Layer 4: Verus/Creusot functional

Per-`membarrier(2)` man page + Documentation/scheduler/membarrier.rst semantic equivalence. Go runtime preemption test passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`membarrier(2)` reinforcement:

- **Per-flags reserved-bits check** — defense against per-extension-bit smuggling.
- **Per-registration gate on expedited** — defense against per-uregistered DoS via constant IPI flood.
- **Per-IPI mask scoped to caller's mm** — defense against per-cross-mm IPI noise (other workloads not perturbed).
- **Per-IPI handler in interrupt context, allocation-free** — defense against per-IPI memory exhaustion.
- **Per-cpu_id online check** — defense against per-offline-CPU IPI hang.
- **Per-rseq-registration requirement for RSEQ variant** — defense against per-untracked rseq abort.

## Grsecurity / PaX surface

- **PaX UDEREF on syscall entry** — defense against per-arg kernel-deref bug (no user pointer in args, but SMAP enforced).
- **Membarrier rseq-required** — RSEQ-variant commands require that the calling thread has a registered rseq via `rseq(2)`; grsec enforces this strictly so an attacker cannot abuse the rseq-abort path on a thread without an rseq mapping (which would otherwise be a no-op crater for DoS amplification).
- **GRKERNSEC_PROC_HIDE on `mm->membarrier_state`** — not exposed via /proc; only the syscall returns it.
- **PAX_REFCOUNT on mm refcount during IPI dispatch** — defense against mm vanish mid-IPI.
- **GRKERNSEC_IPI_RATELIMIT** — per-task rate-limit on expedited membarrier invocations (default 1000/s) to prevent IPI-storm DoS against shared cores; kernel emits a `grsec_audit` log on burst.
- **Per-cpu_id capability check** — when `MEMBARRIER_CMD_FLAG_CPU` targets a CPU running a task in a different cgroup, the call is rate-bucketed separately; cross-cgroup IPI floods are detected and capped.
- **Per-CAP_SYS_NICE not required** — membarrier is intentionally available to unprivileged users; grsec compensates with the rate-limit above rather than blanket-blocking.
- **PaX KERNEXEC on IPI handler** — handler in .text, no W^X violation.
- **Per-CONFIG_RSEQ dependency strict** — RSEQ commands return EINVAL when RSEQ disabled at kernel config; grsec also disables them when grsec_disable_rseq=1.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- rseq(2) registration mechanics (covered in `rseq.md`).
- Per-arch IPI dispatch (covered in arch Tier-3 docs).
- Scheduler smp_mb__after_switch_mm hook (covered in `sched.md`).
- Implementation code.
