---
title: "Tier-3: kernel/workqueue.c — concurrency-managed workqueue (per-CPU + unbound + ordered + WQ pools)"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Workqueues are kernel deferred-execution primitives: a `work_struct` queued via `queue_work` runs in a per-pool worker kthread context (sleep-allowed, can acquire mutexes, can do IO). The CMWQ (Concurrency Managed Workqueue) infrastructure organizes worker threads into per-CPU `worker_pool`s + per-NUMA-node unbound pools; each pool grows or shrinks workers based on load + scheduler signals (worker becomes runnable triggers wake of additional worker if first is sleeping). Critical for: every async-IO completion path in the kernel, per-device async-probe, RCU callback offload (NOCB), block-layer device probe, suspend/resume, eth-driver TX-completion, USB driver, etc.

`kernel/workqueue.c` is ~8450 lines, the third-largest single kernel file.

This Tier-3 covers `workqueue.c` + `workqueue_internal.h` + `workqueue.h`.

### Acceptance Criteria

- [ ] AC-1: Boot init: `dmesg | grep "workqueue"` shows per-CPU worker_pools created.
- [ ] AC-2: queue_work + work invocation: `INIT_WORK(&w, fn); queue_work(system_wq, &w);` results in fn being invoked in worker context.
- [ ] AC-3: delayed_work: queue_delayed_work with 100ms; verify fn runs ~100ms later.
- [ ] AC-4: cancel_work_sync: cancel pending work that has not started; cancel work that has started → wait for completion.
- [ ] AC-5: flush_workqueue: queue 1000 works; flush_workqueue blocks until all complete.
- [ ] AC-6: WQ_MEM_RECLAIM: simulate worker stuck in IO; rescuer kicks in to drain queue.
- [ ] AC-7: Multi-CPU stress: queue 1M works distributed across 32-CPU host; all execute; no work lost.
- [ ] AC-8: __WQ_ORDERED: queue 100 works on ordered WQ; verify execution order matches queue order.
- [ ] AC-9: WQ_UNBOUND NUMA: queue work on NUMA-attrs unbound WQ; worker runs on correct NUMA-node.
- [ ] AC-10: cgroup-cpuset constraint: WQ workers respect per-cgroup cpuset mask.

### Architecture

`Workqueue` per-WQ:

```
struct Workqueue {
  pwqs: KArc<RcuPointer<KVec<KArc<PoolWorkqueue>>>>,  // per-CPU or per-NUMA-node
  list: ListHead,                                       // global wq-list link
  flush_color: AtomicU32,
  flush_mutex: Mutex<()>,
  saved_max_active: u32,
  unbound_attrs: Option<KBox<WorkqueueAttrs>>,
  rescuer: Option<KArc<Worker>>,
  flags: u32,
  ...
}

struct WorkerPool {
  cpu: i32,                                             // -1 if unbound
  node: i32,
  id: u32,
  flags: u32,
  workers: ListHead,                                    // all workers
  idle_list: ListHead,
  busy_workers_hashtable: KBox<[Hlist; BUSY_WORKER_HASH_SIZE]>,
  worklist: ListHead,                                   // pending work
  nr_running: AtomicU32,                                 // running workers count
  nr_idle: u32,
  watchdog_ts: u64,
  manager_arb: Mutex<()>,
  attrs: WorkqueueAttrs,
  ...
}

struct Worker {
  task: KArc<TaskStruct>,
  pool: KArc<WorkerPool>,
  current_work: Option<KWeak<Work>>,
  current_func: Option<WorkFn>,
  current_pwq: Option<KArc<PoolWorkqueue>>,
  scheduled: ListHead,
  flags: u32,
  ...
}
```

`Workqueue::queue(wq, work)`:
1. preempt_disable.
2. cpu := smp_processor_id() (or work-target).
3. pwq := wq.per-cpu or per-numa pwq.
4. Atomic compare-exchange WORK_STRUCT_PENDING in work.data:
   - If was PENDING: skip (already queued); return false.
5. Set work.data = pwq | PENDING.
6. If pwq.nr_active < pwq.max_active:
   - list_add_tail(&work.entry, &pool.worklist).
   - pwq.nr_active++.
   - wake_up_worker(pool).
7. Else:
   - list_add_tail(&work.entry, &pwq.inactive_works).
8. preempt_enable.
9. Return true.

`Worker::main_loop`:
1. Loop:
   - prepare_to_wait_exclusive(&pool.idle_wait).
   - if pool.worklist.empty():
     - schedule (sleep).
     - continue.
   - finish_wait.
   - work := list_first_entry(&pool.worklist, struct work_struct, entry).
   - process_one_work(worker, work).

`Worker::process_one(worker, work)`:
1. list_del_init(&work.entry).
2. busy-hashtable insert worker.
3. set_work_pwq(work, NULL); clear PENDING bit (allows re-queue).
4. work.func(work).
5. set_worker_idle.

`scheduler_hook` (wq_worker_sleeping):
1. worker := task->worker.
2. pool.nr_running--.
3. If pool.nr_running == 0 && !pool.worklist.empty():
   - wake_up_worker(pool).

### Out of Scope

- Async (async.c covered in `kernel/async.md` future Tier-3)
- Tasklet (tasklet.c covered in `kernel/softirq.md` future Tier-3)
- Threaded IRQ (irqthread.c covered in `kernel/irq/irqdesc.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct work_struct` | per-work primitive | `kernel::workqueue::Work` |
| `struct delayed_work` | work + timer | `DelayedWork` |
| `struct workqueue_struct` | per-WQ descriptor | `Workqueue` |
| `struct worker_pool` | per-pool worker mgmt | `WorkerPool` |
| `struct worker` | per-thread state | `Worker` |
| `struct pool_workqueue` (`pwq`) | per-(WQ, pool) link | `PoolWorkqueue` |
| `INIT_WORK(work, func)` | init work_struct | `Work::init` |
| `INIT_DELAYED_WORK(dwork, func)` | init delayed_work | `DelayedWork::init` |
| `queue_work(wq, work)` / `queue_work_on(cpu, wq, work)` | enqueue per-WQ | `Workqueue::queue` / `_queue_on` |
| `queue_delayed_work(wq, dwork, delay)` | delay-then-enqueue | `Workqueue::queue_delayed` |
| `mod_delayed_work(...)` | reschedule delayed-work | `Workqueue::mod_delayed` |
| `cancel_work(work)` / `cancel_delayed_work(dwork)` | cancel pending | `Work::cancel` / `DelayedWork::cancel` |
| `cancel_work_sync(work)` | cancel + wait | `Work::cancel_sync` |
| `flush_workqueue(wq)` | wait for queued+executing | `Workqueue::flush` |
| `flush_work(work)` | wait for specific work | `Work::flush` |
| `drain_workqueue(wq)` | flush + reject new | `Workqueue::drain` |
| `alloc_workqueue(fmt, flags, max_active, ...)` | create custom WQ | `Workqueue::alloc` |
| `destroy_workqueue(wq)` | teardown | `Workqueue::destroy` |
| `system_wq` / `system_long_wq` / `system_unbound_wq` / etc. | global default WQs | `Workqueue::SYSTEM_*` |
| `worker_thread(worker)` | per-thread main | `Worker::main_loop` |
| `process_one_work(worker, work)` | per-work invoke | `Worker::process_one` |
| `pwq_activate_inactive_work(pwq)` | activate from inactive list | `PoolWorkqueue::activate` |
| `wq_worker_running(...)` / `_sleeping(...)` | scheduler hooks | `Worker::scheduler_hook` |

### compatibility contract

REQ-1: `work_struct` (24 bytes typical):
- `data` (atomic_long; encodes pwq pointer + flags PENDING/PWQ_STATS/etc.).
- `entry` (list-node).
- `func` (function-pointer).

REQ-2: `delayed_work` (work + timer):
- `work` (embedded work_struct).
- `timer` (timer_list).
- `wq` (workqueue back-reference).
- `cpu` (target CPU).

REQ-3: Per-WQ flags:
- `WQ_UNBOUND` (workers not pinned to specific CPU).
- `WQ_FREEZABLE` (freeze during system suspend).
- `WQ_MEM_RECLAIM` (rescuer thread reserved for memory-reclaim path).
- `WQ_HIGHPRI` (worker_pool with elevated nice).
- `WQ_CPU_INTENSIVE` (worker excluded from concurrency-management on its CPU).
- `WQ_POWER_EFFICIENT` (hint to scheduler to preserve power).
- `WQ_SYSFS` (expose via sysfs).
- `__WQ_ORDERED` (single-thread sequential WQ).

REQ-4: Per-CPU per-pool worker management:
- Per-CPU normal-priority pool (one worker pool per CPU).
- Per-CPU high-priority pool (WQ_HIGHPRI).
- Per-NUMA-node unbound pools (one per (node, attrs)).
- Per-pool: list of workers, list of pending works, list of busy workers, idle-list with timeout-based death.

REQ-5: Worker concurrency management:
- Pool tracks "manage workers needed" via per-CPU running-count from scheduler hooks.
- Worker becoming sleeping → trigger wake_up_worker if pool has pending work.
- Worker becoming running → no-op (already executing).
- Goal: at most one running worker per CPU per pool unless work is CPU-bound.

REQ-6: queue_work flow:
1. Disable preemption.
2. Determine target pwq (per-(wq, target-pool)).
3. Atomic-set work.data with WORK_STRUCT_PENDING + pwq pointer.
4. If pwq.nr_active < max_active: insert work in pool.worklist; wake-up worker.
5. Else: insert work in pwq.inactive_works (delayed activation).
6. Re-enable preempt.

REQ-7: queue_delayed_work:
1. Setup timer with `delayed_work_timer_fn` callback.
2. Timer expiration: wq-internally call queue_work.

REQ-8: Worker main loop:
1. Sleep until pool has work.
2. Wake; consume head of pool.worklist.
3. process_one_work:
   - `set_work_pwq(work, NULL)` (clear PENDING bit; allow re-queue).
   - Mark worker busy.
   - Call work.func(work).
   - Mark worker idle.
4. Loop.

REQ-9: max_active:
- Per-WQ + per-CPU max number of works simultaneously executing.
- For WQ_UNBOUND: per-NUMA-node max.
- Default: 256 per-CPU bound; 4 * num_cpus for unbound.
- Excess work waits in pwq.inactive_works.

REQ-10: __WQ_ORDERED:
- max_active forced to 1; works execute in queue-order.
- Used for: device-probe per-bus, FS journal-commit, etc. that require strict serialization.

REQ-11: WQ_MEM_RECLAIM rescuer:
- Per-WQ kthread that runs only when pool has stuck work (workers all blocked in IO).
- Used for memory-reclaim path: prevents PI deadlock where pool waits for memory + memory waits for IO that needs WQ.

REQ-12: cancel_work_sync semantics:
- Atomic clear of WORK_STRUCT_PENDING.
- If work is currently executing: wait via `flush_work`.
- After return: work is not pending and not executing.

REQ-13: drain_workqueue + destroy_workqueue:
- drain: continuously flush until no new work queued (defense against work-queues-self).
- destroy: drain + free per-WQ structures.

REQ-14: Scheduler integration:
- `wq_worker_sleeping(task)` called on schedule-out: pool.nr_running--.
- `wq_worker_running(task)` called on schedule-in: pool.nr_running++.
- If nr_running drops to 0 + pending work: wake additional worker.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `worklist_no_uaf` | UAF | per-work.entry list-node managed via PENDING-bit-protected ownership. |
| `pwq_max_active_respected` | INVARIANT | per-pwq.nr_active ≤ pwq.max_active at all times. |
| `worker_pool_per_cpu_unique` | INVARIANT | per-CPU normal-pool + high-pool registered exactly once. |
| `nr_running_no_underflow` | INVARIANT | pool.nr_running ≥ 0; defense against double-sleep-hook. |
| `pending_bit_serialization` | INVARIANT | WORK_STRUCT_PENDING atomic ↔ exactly one pwq owns work at a time. |

### Layer 2: TLA+

`kernel/workqueue/cmwq_concurrency.tla`:
- Per-CPU per-pool state ∈ {Idle, OneRunning, OneRunningOneSleepingWithWork}.
- Transitions per scheduler hooks + work-process events.
- Properties:
  - `safety_concurrency_management` — at most one worker running per pool unless CPU-INTENSIVE work.
  - `liveness_pending_eventually_dispatched` — every pending work eventually matches a worker (assuming pool not poisoned).

`kernel/workqueue/cancel_sync.tla`:
- Per-work state ∈ {Idle, Pending, Executing, Cancelled}.
- Transitions per queue/start-execute/cancel.
- Properties:
  - `safety_cancel_sync_excludes_executing` — cancel_work_sync return implies state == Idle.
  - `liveness_executing_terminates` — every Executing eventually Idle (assuming work.func returns).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Workqueue::queue` post: work.data has PENDING bit + pwq encoded; work in either pool.worklist or pwq.inactive_works | `Workqueue::queue` |
| `Worker::process_one` post: worker.busy bit cleared; work.PENDING cleared before func called | `Worker::process_one` |
| `cancel_work_sync` post: work.PENDING cleared; if was executing, work.func returned | `Work::cancel_sync` |
| `flush_workqueue` post: every work queued before flush has been processed | `Workqueue::flush` |
| Per-WQ.list integrity: insertion + removal under wq_pool_mutex | `Workqueue::alloc` / `_destroy` |

### Layer 4: Verus/Creusot functional

`queue_work + worker dequeue + process_one_work → work.func invoked exactly once per queue` semantic equivalence: per-queue-of-(work, func), exactly one invocation of func happens unless cancel intervenes before execution.

### hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

workqueue-specific reinforcement:

- **WORK_STRUCT_PENDING bit atomic** — defense against concurrent queue_work double-enqueue.
- **WQ_MEM_RECLAIM rescuer thread** — defense against memory-reclaim PI-deadlock.
- **drain_workqueue rejects new work** — defense against work-queues-self preventing destroy.
- **Per-pool watchdog** — detects stuck pool (no worker progress for ~30s) + dumps stack trace.
- **Per-WQ flush_color counter** — defense against premature flush_workqueue return when work re-queued.
- **scheduler hook serialization** under task->pi_lock — defense against torn nr_running.
- **Per-pool worker death timeout (5min idle)** — defense against unbounded worker-pool growth on transient burst.
- **Per-CPU worker affinity enforced** — defense against worker migrating to different CPU mid-work breaking per-CPU-state assumptions.
- **__WQ_ORDERED max_active=1 strict** — defense against undefined ordering when ordered-WQ misconfigured.
- **destroy_workqueue WARN_ON if pwq.nr_active != 0** — defense against premature destroy.
- **Per-work data encoding validates aligned pwq pointer** — defense against torn data field reading half-pointer.
- **Per-pool busy_workers_hashtable** — defense against per-work double-execution across pools (ABA via re-queue).

