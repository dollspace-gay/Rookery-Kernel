# Tier-3: kernel/rcu/srcutree.c — SRCU (Sleepable RCU) per-srcu-struct grace-period tracking

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/rcu/tree.md
upstream-paths:
  - kernel/rcu/srcutree.c
  - kernel/rcu/srcutiny.c
  - include/linux/srcu.h
  - include/linux/srcutree.h
  - kernel/rcu/srcu.h
-->

## Summary

SRCU (Sleepable RCU) is a per-domain RCU variant where readers may sleep / block / acquire mutexes inside the read-side critical section. Each `struct srcu_struct` is an independent RCU domain with its own grace-period sequence + per-CPU reader counters. Critical for: memslot iteration in KVM (readers walk slots while doing IO/userspace-copy that may sleep), notifier-list dispatch (`mmu_notifier`, `kvm_page_track`), `tracepoint` registration. Distinct from regular RCU because (a) per-domain (no global GP), (b) sleeper-friendly (per-CPU per-cpu_idx counters instead of preempt-disable), (c) GP duration is unbounded — appropriate only when domain-scoped.

`srcutree.c` is ~2203 lines. This Tier-3 covers `srcutree.c` + `srcu.h` + `srcutree.h` (srcutiny.c is single-CPU UP variant; out-of-scope for x86_64).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct srcu_struct` | per-domain SRCU state | `kernel::rcu::srcu::SrcuStruct` |
| `struct srcu_data` | per-CPU per-domain counters + cb_list | `SrcuData` |
| `struct srcu_node` | per-tree-node aggregation | `SrcuNode` |
| `srcu_read_lock(ssp)` / `_unlock(ssp, idx)` | per-CPU reader entry/exit | `Srcu::read_lock` / `_unlock` |
| `synchronize_srcu(ssp)` | wait for all current readers | `Srcu::synchronize` |
| `synchronize_srcu_expedited(ssp)` | force fast GP | `Srcu::synchronize_expedited` |
| `call_srcu(ssp, head, func)` | enqueue per-domain callback | `Srcu::call` |
| `srcu_barrier(ssp)` | wait for all in-flight callbacks | `Srcu::barrier` |
| `init_srcu_struct(ssp)` | per-domain init | `SrcuStruct::init` |
| `cleanup_srcu_struct(ssp)` | per-domain teardown | `SrcuStruct::cleanup` |
| `srcu_gp_start(ssp)` / `_gp_end(ssp)` | per-domain GP start/end | `Srcu::gp_start` / `_gp_end` |
| `srcu_invoke_callbacks(ssp)` | per-CPU callback invocation kthread | `Srcu::invoke_callbacks` |
| `srcu_get_delay(ssp)` | per-domain GP-poll-delay scaling | `Srcu::get_delay` |
| `srcu_advance_state(ssp)` | per-domain state-machine advance | `Srcu::advance_state` |
| `srcu_readers_active(ssp)` | check no-readers (cleanup) | `Srcu::readers_active` |
| `srcu_torture_stats_print(...)` | stats dump for torture | `Srcu::print_torture_stats` |

## Compatibility contract

REQ-1: Per-domain `srcu_struct`:
- Two per-CPU reader-counter arrays (indexed by `cpu_idx`):
  - `c[0]`: lock count for current cycle.
  - `c[1]`: lock count for prev cycle.
  - `unlocks[0]` / `unlocks[1]`: unlock count.
- Per-domain `srcu_idx` (0 or 1; flipped per-GP).
- Per-domain `srcu_gp_seq` (modular GP-sequence number).
- Per-domain tree of `srcu_node` (similar topology to RCU tree).
- Per-CPU `srcu_data.cb_list` (segmented per-domain).

REQ-2: Reader semantics:
- `idx = srcu_read_lock(ssp)`:
  - idx := READ_ONCE(ssp.srcu_idx).
  - this_cpu(ssp.sda).srcu_lock_count[idx]++.
  - smp_mb() (acquire barrier).
  - return idx.
- `srcu_read_unlock(ssp, idx)`:
  - smp_mb() (release barrier).
  - this_cpu(ssp.sda).srcu_unlock_count[idx]++.
- READER MAY SLEEP between lock + unlock.

REQ-3: synchronize_srcu semantics:
1. Increment GP counter; flip srcu_idx (idx old → idx new).
2. Wait for all CPUs to report all (lock_count[old] - unlock_count[old]) reach equal (i.e., all old-cycle readers exited).
3. Increment GP counter again; flip srcu_idx back (or rotate per impl).
4. Wait again.
5. Two flips ensures any lockless-window reader observed.

REQ-4: Per-CPU `srcu_data`:
- `srcu_lock_count[2]` (lock count for each idx).
- `srcu_unlock_count[2]` (unlock count for each idx).
- `srcu_cblist` (segmented callback list).
- `srcu_gp_seq_needed` (per-CPU GP-sequence-needed for cb completion).
- `srcu_gp_seq_needed_exp` (expedited).
- `delay_work` (work-queue for per-CPU callback invocation).

REQ-5: GP state machine (per-domain `srcu_state`):
- IDLE (no GP in progress).
- DELAYED (CB present but waiting to start GP for batching).
- SCAN1 (waiting for old-idx readers).
- SCAN2 (waiting for second-flip readers).

REQ-6: Per-domain tree topology:
- Per-leaf srcu_node aggregates ≤ SRCU_NODES_PER_LEAF CPUs.
- Per-domain root srcu_node summarizes.
- Used to scale reader-count gathering for synchronize_srcu.

REQ-7: call_srcu semantics:
- `call_srcu(ssp, head, func)` — per-CPU enqueue + trigger GP if not pending.
- After GP completes: callback invoked via per-CPU work-queue.

REQ-8: Lifetime safety:
- `cleanup_srcu_struct` must be called only after all readers exited + all callbacks invoked + barrier complete.
- Static-init: `DEFINE_SRCU(name)` macro for module-scope srcu domains.

REQ-9: Per-domain `srcu_callback_priority`:
- `WORK_HIGH_PRI_FLAG` for high-priority callback domains.
- Default: per-CPU work-queue ordinary priority.

REQ-10: SRCU performance scaling:
- Per-leaf srcu_node minimizes cross-CPU cacheline ping-pong on lock_count++.
- Per-CPU reader-counter ensures lock_count++ ↔ unlock_count++ same cacheline.

REQ-11: SRCU barrier:
- `srcu_barrier(ssp)`:
  - For each CPU: call_srcu of barrier-callback that decrements per-srcu barrier-counter.
  - Wait for counter to reach zero.

## Acceptance Criteria

- [ ] AC-1: Init/cleanup: `init_srcu_struct(&ssp)` + `cleanup_srcu_struct(&ssp)` round-trip; no leak.
- [ ] AC-2: Reader sleeps under lock: `srcu_read_lock` → `mutex_lock` → `mutex_unlock` → `srcu_read_unlock` succeeds without WARN.
- [ ] AC-3: synchronize_srcu correctness: writer thread updates pointer + synchronizes; reader started before sees old value; reader after sees new.
- [ ] AC-4: Per-domain isolation: domain A synchronize_srcu does not block domain B readers.
- [ ] AC-5: rcutorture --type=srcud / srcu pass.
- [ ] AC-6: Performance: 1M srcu_read_lock/unlock pairs on 16-CPU host < 100ms total (per-CPU cacheline-local).
- [ ] AC-7: Expedited: `synchronize_srcu_expedited` < 100us with no readers vs 1ms+ for normal.
- [ ] AC-8: srcu_barrier: enqueue callback on each CPU; barrier blocks until all invoked.
- [ ] AC-9: KVM memslot use case: KVM memslot iter under srcu_read_lock; synchronize_srcu after slot-update completes correctly.

## Architecture

`SrcuStruct` per-domain:

```
struct SrcuStruct {
  srcu_idx: AtomicI32,                       // 0 or 1; flipped per GP
  sda: PerCpu<KArc<SrcuData>>,
  node: KBox<[SrcuNode; NUM_SRCU_NODES]>,
  level: [SrcuNodeRef; SRCU_NUM_LVLS],
  srcu_size_state: AtomicI32,                // SRCU_SIZE_SMALL / BIG / WAIT_CALL / WAIT_CBS
  srcu_cb_mutex: Mutex<()>,                  // GP-state-machine mutex
  srcu_gp_seq: AtomicU64,
  srcu_gp_seq_needed: AtomicU64,
  srcu_gp_seq_needed_exp: AtomicU64,
  ...
}

struct SrcuData {
  srcu_lock_count: [AtomicU64; 2],           // per-idx lock counter
  srcu_unlock_count: [AtomicU64; 2],         // per-idx unlock counter
  srcu_cblist: SrcuSegCbList,
  srcu_gp_seq_needed: u64,
  srcu_gp_seq_needed_exp: u64,
  cpu: u32,
  ssp: KWeak<SrcuStruct>,
  delay_work: DelayedWork,
  ...
}

struct SrcuNode {
  lock: SpinLock<()>,
  srcu_have_cbs: [u64; 4],                   // per-state cb-needed counts
  srcu_data_have_cbs: [u64; 4],              // per-CPU rolled-up
  srcu_gp_seq: u64,
  srcu_gp_seq_needed: u64,
  srcu_gp_seq_needed_exp: u64,
  parent: Option<SrcuNodeRef>,
}
```

`Srcu::read_lock(ssp)`:
1. idx := atomic_load(&ssp.srcu_idx);
2. cpu := smp_processor_id().
3. atomic_inc(&ssp.sda[cpu].srcu_lock_count[idx]).
4. smp_mb().
5. Return idx.

`Srcu::read_unlock(ssp, idx)`:
1. smp_mb().
2. cpu := smp_processor_id().
3. atomic_inc(&ssp.sda[cpu].srcu_unlock_count[idx]).

`Srcu::synchronize`:
1. Acquire ssp.srcu_cb_mutex.
2. start_seq = ssp.srcu_gp_seq.
3. Trigger GP advance: ssp.srcu_gp_seq += 1; ssp.srcu_idx ^= 1.
4. Wait for old-idx reader-count to balance (sum of lock - unlock per-CPU == 0).
5. ssp.srcu_gp_seq += 1; ssp.srcu_idx ^= 1.
6. Wait again.
7. Release mutex.

(In practice, synchronize_srcu uses call_srcu + completion to avoid blocking the CPU.)

`Srcu::call(ssp, head, func)`:
1. Disable preemption.
2. cpu := smp_processor_id().
3. sdp := this_cpu(ssp.sda).
4. head.func = func.
5. rcu_segcblist_enqueue(&sdp.srcu_cblist, head, RCU_NEXT_TAIL).
6. If GP not in progress for sdp.srcu_gp_seq_needed: trigger advance_state.
7. Re-enable preempt.

`Srcu::advance_state(ssp)`:
- States: IDLE → SCAN1 → SCAN2 → DONE → IDLE.
- IDLE → SCAN1: flip idx; record start-time.
- SCAN1 → SCAN2: when old-idx readers exited.
- SCAN2 → DONE: when twice-flipped readers exited; advance gp_seq; trigger callback invocation.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `srcu_data_no_uaf` | UAF | per-CPU sda accessed under preempt-disable; per-domain freed only after cleanup. |
| `lock_unlock_balance` | INVARIANT | per-(cpu, idx) lock_count - unlock_count ≥ 0 at all times. |
| `srcu_idx_two_value` | INVARIANT | srcu_idx ∈ {0, 1}; flipped via XOR. |
| `srcu_gp_seq_monotonic` | INVARIANT | srcu_gp_seq monotonically increases per advance. |

### Layer 2: TLA+

`kernel/rcu/srcu_gp.tla`:
- Per-CPU reader state ∈ {Outside, Inside(idx)}.
- Per-domain GP state ∈ {Idle, Scan1(old_idx), Scan2(old_idx), Done}.
- Transitions:
  - Reader Outside → Inside(srcu_idx) via srcu_read_lock.
  - Reader Inside(idx) → Outside via srcu_read_unlock.
  - GP Idle → Scan1(old) via synchronize trigger; flip srcu_idx.
  - Scan1(old) → Scan2(old') when no Inside(old) readers.
  - Scan2 → Done after second flip.
- Properties:
  - `safety_synchronize_after_all_old_readers_exited` — synchronize_srcu return only after every reader inside at synchronize-call exited.
  - `liveness_gp_terminates` — assuming readers eventually exit, every Scan eventually advances.
  - `safety_no_idx_reuse_during_scan` — srcu_idx may not flip back during Scan.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Srcu::read_lock` post: per-CPU lock_count[idx] += 1; smp_mb; idx returned matches counter | `Srcu::read_lock` |
| `Srcu::read_unlock` post: per-CPU unlock_count[idx] += 1 | `Srcu::read_unlock` |
| `Srcu::synchronize` post: no reader inside at call-time still inside at return | `Srcu::synchronize` |
| `Srcu::call` post: cb in cb_list; GP request issued if not pending | `Srcu::call` |
| `cleanup_srcu_struct` precondition: srcu_readers_active(ssp) == 0 | `SrcuStruct::cleanup` |

### Layer 4: Verus/Creusot functional

`Reader-Writer correctness for SRCU`: any reader that enters via `srcu_read_lock` before `synchronize_srcu` returns must see pre-synchronize values; readers entering after see post-synchronize. Two-flip GP ensures no straddling reader observed.

## Hardening

(Inherits row-1 features from `kernel/rcu/tree.md` § Hardening.)

SRCU-specific reinforcement:

- **Per-CPU reader counters cacheline-aligned** — defense against cross-CPU cacheline-ping-pong on lock_count++.
- **smp_mb() acquire/release barriers** in lock/unlock — defense against compiler/CPU reorder leaking pointer-load past lock.
- **Two-flip GP discipline** — defense against straddling-reader-window where new-idx reader sees old data.
- **cleanup_srcu_struct precondition: no active readers** — defense against use-after-free of srcu_struct mid-read.
- **Per-domain srcu_cb_mutex** — defense against concurrent GP-state advance.
- **SRCU_SIZE_SMALL → BIG transition** under workload — defense against lock-contention on hot domains; per-CPU sda grows lazily.
- **Per-srcu_node spinlock for GP-state mutation** — defense against torn state-machine update.
- **call_srcu cb head NULL after invocation** — defense against double-invoke UAF.
- **srcu_barrier wait until all CPUs barrier-cb invoked** — defense against module-unload race.
- **Reader counter overflow capped (u64)** — defense against ~10^11/sec persistent counter overflow (impossible-in-practice but worth-bound).
- **Per-domain GP latency reported** via dmesg if exceeds threshold — defense against pathologically-long readers stalling GP.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy.
- **PAX_KERNEXEC** — W^X for any executable mapping.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization.
- **PAX_REFCOUNT** — saturating refcount on subsystem structs.
- **PAX_MEMORY_SANITIZE** — zero-on-free for sensitive allocations.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on vtables.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding.
- **GRKERNSEC_DMESG** — syslog restriction.
- **call_srcu callback indirect dispatch under PAX_RAP** — `rcu_head->func` invoked through kCFI; a tampered callback function pointer fails the signature check before being executed in soft-IRQ context.
- **GP-stall detection emits via GRKERNSEC_DMESG-restricted log** — stall messages (and any pointer they contain) sanitized for unprivileged dmesg readers.
- **force_quiescent_state path under PAX_RAP** — `srcu_invoke_callbacks` workqueue dispatch kCFI-signed; cannot be hijacked to call a gadget at "GP complete" time.
- **srcu_struct->srcu_sda per-CPU bounds** — accesses indexed by `smp_processor_id()` masked against `nr_cpu_ids` so a CPU-hotplug race cannot OOB the per-CPU data.
- **PAX_MEMORY_SANITIZE on cleanup_srcu_struct** — sda counters and node spinlocks zeroed before free so a use-after-free read returns zeros rather than stale reader-counter state (which would otherwise be a powerful timing oracle).
- **Reader counter saturation** — the u64 counter is reinforced with a sanity wrap-detect that triggers a stall report rather than silently invalidating GP discipline.
- **call_srcu cb-list traversal RCU-protected** — list nodes freed via call_rcu after detach so a brief UAF reads sanitized state.

Per-doc rationale: SRCU is widely used to protect sleepable readers in subsystems that the kernel itself must trust (notifier chains, KVM, fs notification). The single highest-value attacker move would be to install a callback function pointer that runs at GP-complete time with kernel privileges. PAX_RAP on the call_srcu callback and on the srcu_invoke_callbacks workqueue dispatch turns that into a CFI fault. PAX_MEMORY_SANITIZE on cleanup closes the reader-counter timing oracle, and GRKERNSEC_DMESG hides the stall reports from unprivileged kallsyms-harvesters.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Tree-RCU (covered in `kernel/rcu/tree.md` Tier-3)
- TINY-RCU UP variant (not used on x86_64)
- srcutiny.c (UP variant; out-of-scope)
- Tasks-RCU (covered separately)
- rcu_barrier (covered in `kernel/rcu/tree.md` Tier-3)
- Implementation code
