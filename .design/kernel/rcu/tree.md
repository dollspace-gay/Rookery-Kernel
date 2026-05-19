# Tier-3: kernel/rcu/tree.c — Tree-RCU implementation (multi-level rcu_node tree + grace-period state machine)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/rcu/tree.c
  - kernel/rcu/tree.h
  - kernel/rcu/tree_plugin.h
  - kernel/rcu/tree_exp.h
  - kernel/rcu/tree_nocb.h
  - kernel/rcu/tree_stall.h
  - kernel/rcu/rcu.h
  - include/linux/rcupdate.h (public API)
  - include/linux/rcutree.h
-->

## Summary

Tree-RCU is the Linux RCU implementation that scales to 1000s of CPUs by organizing per-CPU quiescent-state reporting into a tree of `rcu_node` structures. Each level fans out 16-64 CPUs (RCU_FANOUT); the root rcu_node sees aggregated grace-period state. Per-CPU `rcu_data` tracks per-CPU callback list + quiescent-state status; per-CPU softirq + per-CPU kthread (rcuc) batches callback invocations. Critical for all kernel readers using `rcu_read_lock()/_unlock()`: ensures any object freed via `call_rcu` is reachable until every existing reader completes. Tree-RCU is the default; preemptible-RCU additionally supports preempting RCU readers (for PREEMPT_RT).

`tree.c` is ~4931 lines, the second-largest single file in the kernel tree (after kvm/x86/mmu/mmu.c).

This Tier-3 covers `tree.c` + `tree.h` + `tree_plugin.h` + `tree_exp.h` + `tree_nocb.h` + `tree_stall.h`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rcu_state` | global RCU state | `kernel::rcu::tree::RcuState` |
| `struct rcu_node` | per-tree-node aggregation | `RcuNode` |
| `struct rcu_data` | per-CPU RCU data | `RcuData` |
| `call_rcu(head, func)` | enqueue per-CPU callback | `Rcu::call` |
| `synchronize_rcu()` | block until grace period elapses | `Rcu::synchronize` |
| `rcu_read_lock()` / `rcu_read_unlock()` | enter/exit reader (compile-noop in CONFIG_PREEMPT=n) | `Rcu::read_lock` / `_unlock` |
| `rcu_read_lock_sched()` / `rcu_read_unlock_sched()` | sched-RCU reader (disable preempt) | `Rcu::read_lock_sched` / `_unlock_sched` |
| `rcu_read_lock_bh()` / `rcu_read_unlock_bh()` | BH-RCU reader | `Rcu::read_lock_bh` / `_unlock_bh` |
| `rcu_qs()` / `rcu_note_context_switch(...)` | per-CPU quiescent-state report | `Rcu::qs` / `_note_context_switch` |
| `rcu_check_callbacks(...)` (timer-tick) | per-tick callback check | `Rcu::check_callbacks` |
| `rcu_core(...)` | softirq/kthread main | `Rcu::core` |
| `rcu_gp_init(...)` / `rcu_gp_cleanup(...)` | grace-period state-machine | `Rcu::gp_init` / `_gp_cleanup` |
| `rcu_gp_kthread(...)` | per-state grace-period kthread | `Rcu::gp_kthread` |
| `rcu_report_qs_rnp(...)` | propagate per-CPU qs up tree | `Rcu::report_qs_rnp` |
| `rcu_advance_cbs(...)` | advance callback-list per grace-period | `Rcu::advance_cbs` |
| `rcu_do_batch(...)` | invoke per-CPU callback batch | `Rcu::do_batch` |
| `rcu_barrier()` | wait until all in-flight callbacks invoked | `Rcu::barrier` |
| `synchronize_rcu_expedited()` | force grace-period via IPI | `Rcu::synchronize_expedited` |
| `rcu_offload_callbacks(...)` (tree_nocb.h) | NOCB callback offload | `Rcu::offload_callbacks` |
| `print_other_cpu_stall(...)` (tree_stall.h) | RCU CPU-stall warnings | `Rcu::print_stall` |

## Compatibility contract

REQ-1: Tree topology (compile-time):
- `RCU_FANOUT` (typically 16-64 per level).
- Levels: 1 to 4 depending on `nr_cpu_ids`.
- Per-leaf rcu_node aggregates ≤ RCU_FANOUT CPUs.
- Root rcu_node summarizes entire tree.

REQ-2: Per-CPU rcu_data:
- `gp_seq` (last-observed grace-period sequence).
- `cb_list` (RcuSegcbList — segmented callback list partitioned by destination grace period).
- `passed_quiesce` (bool, per-CPU qs report status).
- `cpu_no_qs` (per-CPU qs-pending status).
- `core_needs_qs` (per-CPU need-qs flag).

REQ-3: Per-rcu_node state:
- `level` / `grpnum` / `grpmask` (tree position).
- `qsmask` (bitmap of children/CPUs that have not yet reported qs for current GP).
- `qsmaskinit` (initial qsmask at GP start).
- `expmask` (expedited-GP qsmask).
- `gp_seq` (GP-sequence number).

REQ-4: Per-VM grace-period sequence (`gp_seq`):
- Monotonic counter incremented at GP-start + GP-cleanup.
- Per-CPU rcu_data.gp_seq tracks last-acknowledged GP.
- Wraparound-safe via 64-bit + sign-correct comparison helpers.

REQ-5: `call_rcu(head, func)` semantics:
- Per-CPU enqueue: `head` linked into `rcu_data.cb_list` segmented by destination GP.
- After current GP + 1 elapses: callback invoked via softirq/kthread.

REQ-6: `synchronize_rcu()` semantics:
- Block calling task until at least one full GP elapses since call.
- Implementation: `call_rcu(&completion, complete)` + `wait_for_completion`.
- Or `synchronize_rcu_expedited` for faster (but more disruptive) variant.

REQ-7: Per-CPU quiescent-state reporting:
- Schedule-out (preemption / sleep): per-CPU rcu_data.passed_quiesce = 1.
- User-mode entry: idle: same.
- Per-tick (timer interrupt): rcu_check_callbacks; if rcu_data.passed_quiesce: report_qs.
- Per-context-switch: rcu_note_context_switch records CPU as quiescent.

REQ-8: GP propagation:
- Per-CPU qs → leaf rcu_node clear qsmask bit.
- Leaf rcu_node qsmask == 0 → propagate to parent.
- Root rcu_node qsmask == 0 → GP complete; rcu_state.gp_seq advanced.

REQ-9: Callback batching:
- Per-CPU `cb_list` segmented: DONE (callbacks ready to invoke), WAIT (waiting for current GP), NEXT_READY (waiting for GP+1), NEXT (waiting for next-next).
- Per-tick: advance segments based on completed GPs.
- Per-tick or per-kthread: `do_batch` invokes ≤ blimit callbacks (default 10).

REQ-10: NOCB (No-Callback offload):
- For RT systems: per-CPU callbacks offloaded to dedicated rcuog/rcuop kthreads to avoid invoking callbacks on time-critical CPU.
- Configured per-CPU at boot via `nohz_full=N` + `rcu_nocbs=N`.

REQ-11: Expedited-GP (rapid GP):
- `synchronize_rcu_expedited`: send IPI to all online CPUs to force fast qs report.
- Used for memory-pressure code paths that cannot afford 100ms GP latency.

REQ-12: RCU CPU-stall detection (tree_stall.h):
- Per-CPU jiffies-since-GP-start watchdog.
- If GP has not advanced after `rcu_cpu_stall_timeout` (default 21s): print stall warnings + per-CPU stack trace.
- Detect deadlock or livelocked CPU.

REQ-13: SRCU (Sleepable-RCU):
- Variant where readers may sleep; tracked via per-srcu-struct counters (covered in srcu Tier-3 separately).

REQ-14: Per-VM `tree_plugin.h` provides preemptible-RCU additions:
- `rcu_preempt_*` variants tracking blocked readers via per-rcu_node `blkd_tasks` list.
- `rcu_preempt_qs` / `rcu_preempt_note_context_switch` for preempt-aware qs.

## Acceptance Criteria

- [ ] AC-1: Boot-time RCU init: `dmesg | grep "rcu"` shows tree-RCU init with N levels.
- [ ] AC-2: Basic call_rcu: enqueue 1000 callbacks; verify all invoked within 100ms after synchronize_rcu.
- [ ] AC-3: synchronize_rcu correctness: writer thread updates pointer + synchronizes; reader thread that started before synchronize sees old value; reader after sees new value.
- [ ] AC-4: Multi-CPU stress: rcutorture stress test 1h on 32-CPU host without RCU stall.
- [ ] AC-5: Expedited-GP: synchronize_rcu_expedited completes < 1ms vs 100ms+ for normal.
- [ ] AC-6: NOCB callback offload: rcu_nocbs=0-7 cmdline; per-CPU 0-7 callbacks visible on rcuog/rcuop kthreads not on CPU0-7 ksoftirqd.
- [ ] AC-7: rcu_barrier: enqueue callbacks on all CPUs; rcu_barrier blocks until all invoked.
- [ ] AC-8: CPU-stall detection: simulate stuck CPU (set bit in test); rcu stall warning prints after 21s.
- [ ] AC-9: rcutorture full pass.

## Architecture

`RcuState` is the global per-flavor state singleton:

```
struct RcuState {
  node: KBox<[RcuNode; NUM_RCU_NODES]>,     // tree array; root at index 0
  level_nodes: [RcuNodeRef; RCU_NUM_LVLS],  // per-level start-of-array
  rcu_data: PerCpu<RcuData>,
  gp_seq: AtomicU64,
  gp_kthread: Option<KThread>,
  gp_state: AtomicU32,                       // GpState enum
  gp_wq: WaitQueueHead,
  expedited_sequence: AtomicU64,
  expedited_workdone: AtomicU64,
  ...
}

struct RcuNode {
  lock: RawSpinLock,
  level: u8,
  grpnum: u8,
  grpmask: u64,
  qsmask: u64,
  qsmaskinit: u64,
  expmask: u64,
  expmaskinit: u64,
  gp_seq: AtomicU64,
  parent: Option<RcuNodeRef>,
  blkd_tasks: ListHead,                      // preemptible-RCU
  ...
}

struct RcuData {
  gp_seq: u64,
  passed_quiesce: bool,
  cpu_no_qs: BoolArray,                      // per-context (sched, bh)
  core_needs_qs: bool,
  cb_list: RcuSegCbList,                     // segmented callback list
  qlen: u32,                                 // callback count
  blimit: u32,                               // max callbacks per do_batch
  rcu_node: RcuNodeRef,
  ...
}
```

`Rcu::call(head, func)`:
1. Disable preemption.
2. rcu_data := this_cpu_ptr(rcu_state.rcu_data).
3. head.func = func; head.next = NULL.
4. rcu_segcblist_enqueue(&rcu_data.cb_list, head, RCU_NEXT_TAIL).
5. rcu_data.qlen++.
6. If callback list crosses watermark: trigger softirq.
7. Re-enable preemption.

`Rcu::synchronize`:
1. completion := init_completion().
2. cb := { head, .func = wakeme_after_rcu, .completion_ptr = &completion }.
3. call_rcu(&cb.head, wakeme_after_rcu).
4. wait_for_completion(&completion).

`Rcu::gp_kthread` main loop:
1. wait_event(rcu_state.gp_wq, rcu_state.gp_state == GpState::ReadyToStart).
2. `gp_init(rs)`:
   - Per-rcu_node: snapshot online-CPU bitmap to qsmaskinit.
   - rs.gp_seq = (rs.gp_seq + 1) | 1; (mark in-progress).
3. Wait until all CPUs report qs (root.qsmask == 0):
   - Periodic timeouts to check forced-qs (FQS).
   - On stall-detection threshold: invoke print_stall.
4. `gp_cleanup(rs)`:
   - Per-rcu_node: gp_seq = rs.gp_seq.
   - rs.gp_seq = (rs.gp_seq + 1) & ~1; (mark complete).
   - Wake waiters.
5. Loop.

`Rcu::report_qs_rnp(rnp, gp_seq, mask)`:
1. Acquire rnp.lock.
2. If gp_seq != rnp.gp_seq: stale; release + return.
3. rnp.qsmask &= ~mask.
4. If rnp.qsmask != 0: release; done.
5. mask = rnp.grpmask; rnp = rnp.parent.
6. If rnp == NULL: GP complete; rs.gp_state = ReadyToCleanup; wake gp_kthread.
7. Goto step 1.

`Rcu::do_batch(rdp)`:
1. Snapshot rdp.cb_list.DONE segment to local list.
2. blimit = rdp.blimit.
3. For each cb in local list (up to blimit):
   - cb.func(cb).
   - count++.
4. If count == blimit: re-trigger softirq for remainder.
5. rdp.qlen -= count.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rcu_node_tree_no_oob` | OOB | per-CPU rcu_node lookup bounded by NUM_RCU_NODES. |
| `cb_list_no_uaf` | UAF | per-callback enqueue/invoke discipline; head freed only by callback owner after invocation. |
| `gp_seq_monotonic` | INVARIANT | rs.gp_seq monotonically increases (modular order). |
| `qsmask_subset_of_qsmaskinit` | INVARIANT | per-rcu_node qsmask ⊆ qsmaskinit at all times. |

### Layer 2: TLA+

`kernel/rcu/tree_gp.tla` models per-flavor GP state-machine:
- States: Idle, Init, Active(qsmask), Cleanup.
- Transitions:
  - Idle → Init via gp_kthread wake.
  - Init → Active via gp_init.
  - Active(qsmask) → Active(qsmask') via per-CPU qs-report (qsmask' = qsmask \ reported).
  - Active(0) → Cleanup via report_qs_rnp(root) reaching empty.
  - Cleanup → Idle via gp_cleanup.
- Properties:
  - `safety_no_qs_report_outside_active` — qs-report only valid in Active.
  - `safety_gp_seq_advances_only_at_cleanup` — gp_seq even↔odd transitions paired.
  - `liveness_gp_eventually_completes` — assuming all online CPUs eventually qs, every Active eventually reaches Cleanup.

`kernel/rcu/cb_list_segment.tla` models per-CPU callback segment-advance:
- Segment ∈ {DONE, WAIT, NEXT_READY, NEXT}.
- Per-GP-completion: WAIT → DONE; NEXT_READY → WAIT; NEXT → NEXT_READY.
- Properties: `liveness_eventual_invocation` — every NEXT cb eventually reaches DONE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Rcu::call` post: cb enqueued in NEXT_TAIL segment of per-CPU cb_list | `Rcu::call` |
| `Rcu::synchronize` post: completion signaled iff at least one full GP has elapsed since call | `Rcu::synchronize` |
| `Rcu::report_qs_rnp` post: rnp.qsmask cleared by mask; parent reached if rnp.qsmask == 0 | `Rcu::report_qs_rnp` |
| `Rcu::do_batch` post: per-CPU qlen decreased by exactly count-invoked | `Rcu::do_batch` |
| Per-rcu_node.lock held during qsmask mutation | all qs-report paths |

### Layer 4: Verus/Creusot functional

`Reader-Writer correctness`: any reader that begins `rcu_read_lock` before `synchronize_rcu` returns must see pre-synchronize pointer values; readers that begin after see post-synchronize values. Formally: GP semantics ensure happens-before edges between writer + readers.

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

RCU-specific reinforcement:

- **Per-rcu_node.lock raw spinlock** — defense against priority-inversion delaying GP advancement.
- **GP-stall detection** with per-CPU stack-trace dump — defense against silent CPU lockup blocking GP.
- **Per-CPU cb_list segmented** — defense against unbounded callback invocation latency.
- **blimit cap on do_batch** — defense against single-batch consuming too much CPU; preserves softirq latency budget.
- **Per-CPU qlen counter under-flow check** — defense against do_batch double-decrement causing wrap.
- **Forced-quiescent-state (FQS)** at jiffies threshold — defense against stuck CPU not reporting qs causing infinite GP.
- **NOCB offload separate kthreads** — defense against RT-CPU latency blow from RCU-callback invocation.
- **Per-callback head.next NULL after invocation** — defense against double-enqueue UAF.
- **rcu_barrier waits for all in-flight cbs** — defense against module-unload race with pending callbacks.
- **Expedited-GP IPI bounded scope** — defense against expedited-storm causing IPI flood.
- **Per-CPU per-context (sched/bh/preempt) qs tracking** — defense against missed qs in hardirq path.
- **PREEMPT_RT preemptible-RCU per-rcu_node blkd_tasks** — defense against preemption losing qs context.

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
- **call_rcu callback invocation under PAX_RAP** — every `rcu_head->func` invoked from `rcu_do_batch` is dispatched through a kCFI-signed indirect call; a tampered callback function pointer (the classic UAF-into-arbitrary-call gadget) fails the signature check before executing in soft-IRQ context.
- **GP-stall detection emits via GRKERNSEC_DMESG** — `rcu_check_gp_kthread_starvation` and related stall messages routed to the syslog-restricted stream so the addresses they print are not exposed to unprivileged dmesg readers.
- **force_quiescent_state IPI under PAX_RAP** — the per-CPU `rcu_implicit_dynticks_qs` callback fired by FQS dispatched through kCFI.
- **rnp->gp_seq saturation** — sequence numbers monitored for wrap; a wrap triggers a stall report rather than silently re-using a sequence.
- **NOCB offload kthread wakeups PAX_RAP** — `wake_nocb_gp` indirect dispatch into the offload kthread is kCFI-signed.
- **rcu_barrier callback PAX_RAP** — barrier-completion callback function pointer kCFI-signed; cannot be hijacked at module-unload-completion time.
- **PAX_MEMORY_SANITIZE on per-CPU rcu_data tear-down** — segcblist state zeroed when a CPU is offlined so a stale cb pointer cannot be observed transiently by an attacker watching the per-CPU page.
- **expedited GP IPI scope bounded to nr_cpu_ids** — `cpumask_t` index validated; no OOB cpumask write through expedited GP triggers.

Per-doc rationale: tree RCU is the universal "wait until safe to free" primitive in the kernel, and its callback list is one of the most attractive UAF-to-arbitrary-call gadgets — corrupt a `rcu_head->func` and you control the CPU at GP-complete time, in soft-IRQ context, with all locks dropped. PAX_RAP/kCFI on every call_rcu callback and on the rcu_barrier completion is therefore the load-bearing hardening. PAX_MEMORY_SANITIZE on offlined per-CPU segcblist and GRKERNSEC_DMESG on stall reports close the remaining side channels.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- SRCU (covered in `kernel/rcu/srcu.md` future Tier-3)
- Tasks-RCU + Tasks-Trace-RCU (covered in `kernel/rcu/tasks.md` future Tier-3)
- rcuref + rcuref_t (covered in `kernel/rcu/refscale.md` future Tier-3)
- rcutorture (test code; covered in `kernel/rcu/rcutorture.md` future Tier-3)
- TINY-RCU (single-CPU UP variant; not used on x86_64)
- Implementation code
