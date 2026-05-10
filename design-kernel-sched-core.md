---
title: "Tier-3: kernel/sched/core.c — Scheduler core (schedule() dispatch + context switch + wakeup paths + per-CPU runqueue)"
tags: ["tier-3", "kernel-sched", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The brain of the Linux process scheduler — every context switch, every task wakeup, every timer-tick scheduler check, every preemption decision flows through this file (~11K lines, the largest single file in `kernel/sched/`). Owns: the central `__schedule(preempt)` function (the dispatch loop), per-CPU runqueue (`struct rq`) management, per-class scheduler dispatch (CFS / RT / DEADLINE / IDLE / STOP via `sched_class` vtable chain), task wakeup (`try_to_wake_up`, `wake_up_process`, `wake_up_q`), CPU hotplug hooks, sched-tick + sched-domain rebalance trigger, init_task / idle_task setup, voluntary preempt + cond_resched + schedule_timeout helpers, per-task affinity (cpus_allowed) management, scheduler-tick statistics, kernel/sched/sched.h shared definitions.

This Tier-3 covers `kernel/sched/core.c`. Per-class scheduler details (CFS, RT, DEADLINE) live in their own Tier-3s under `kernel/sched/`.

### Acceptance Criteria

- [ ] AC-1: `kernel-bench` test suite scheduler microbenchmarks (context-switch / wake-fast-path / wake-cross-cpu) within 5% of upstream baseline.
- [ ] AC-2: `chrt -f 99 yes > /dev/null` SCHED_FIFO with high prio: task pegs CPU; lower-prio tasks starved; verified via `top -H`.
- [ ] AC-3: `chrt -d -T 1000000 -P 10000000 -D 5000000 0 yes > /dev/null` SCHED_DEADLINE: per-period CPU budget enforced.
- [ ] AC-4: `taskset -c 0,2,4,6 yes > /dev/null` per-task affinity respected.
- [ ] AC-5: CPU hotplug stress: repeatedly `echo 0 > /sys/devices/system/cpu/cpu1/online` + `echo 1 > ...`; tasks pinned to CPU 1 migrated cleanly + restored on re-up.
- [ ] AC-6: stress-ng --switch 8 --timeout 30s sustained context-switch test; throughput within 5% upstream.
- [ ] AC-7: rt-tests `cyclictest -t 8 -p 99 -d 0 -i 1000 -l 100000` worst-case latency < 100µs on PREEMPT_RT-equivalent setup.
- [ ] AC-8: kselftest `tools/testing/selftests/sched/` passes.

### Architecture

`Rq` lives in `kernel::sched::Rq`:

```
struct Rq {
  lock: RawSpinLock,
  cpu: u32,
  nr_running: AtomicU32,
  cpu_load: PerCpu<u64>,
  curr: AtomicPtr<TaskStruct>,
  idle: Arc<TaskStruct>,
  stop: Arc<TaskStruct>,
  cfs: KBox<CfsRq>,                 // CFS per-rq state (cross-ref `kernel/sched/cfs.md`)
  rt: KBox<RtRq>,
  dl: KBox<DlRq>,
  online: AtomicBool,
  active_balance: AtomicBool,
  push_cpu: AtomicU32,
  cpu_capacity: AtomicU32,
  cpu_capacity_orig: AtomicU32,
  rd: Arc<RootDomain>,
  sd: AtomicPtr<SchedDomain>,
  next_balance: AtomicU64,
  prev_irq_time: AtomicU64,
  prev_steal_time: AtomicU64,
  prev_steal_time_rq: AtomicU64,
  calc_load_active: AtomicI64,
  age_stamp: AtomicU64,
  nohz_balance_kick: AtomicBool,
  ttwu_pending: AtomicPtr<TaskStruct>,  // per-CPU pending wake-list (lockless tail)
  numa_migrate_on: AtomicI32,
  rt_avg: PerCpu<u64>,
  prev_irq_time: PerCpu<u64>,
  ...
}
```

Top-level `__schedule(preempt)`:
1. Check `prev = current` and disable preemption.
2. Take `rq->lock`.
3. `prev->state` snapshot:
   - If voluntary schedule + state != TASK_RUNNING + signal_pending(prev): set state back to TASK_RUNNING (don't deschedule).
   - Else if state != TASK_RUNNING: dequeue prev (`rq->dequeue_task(prev, DEQUEUE_SLEEP)`).
4. `next = pick_next_task(rq, prev, &rf)`:
   - For each sched_class in priority order (STOP, DL, RT, fair=CFS, idle):
     - `class->pick_next_task(rq, prev, &rf)` → returns Some(task) or None.
     - First Some wins.
5. If `next != prev`: `context_switch(rq, prev, next, &rf)`:
   - `arch::switch_mm(prev->active_mm, next->mm, next)` (page table switch via per-arch).
   - `arch::switch_to(prev, next, &last)` (register state switch via per-arch — saves prev, loads next, returns prev's `last`).
   - On return from switch_to (now running as next): finish balance (idle-balance pull, etc.).
6. Drop rq->lock; re-enable preempt.

`try_to_wake_up(task, state, wake_flags)`:
1. Smp_mb__before_atomic + check `(task->__state & state) != 0`; if not, return 0 (already running or in different state).
2. `cpu = select_task_rq(task, task->prev_cpu, SD_BALANCE_WAKE, wake_flags)`:
   - per-class callback decides CPU (e.g., CFS: `select_task_rq_fair` with wake-affine, idle-CPU search, sched-domain locality).
3. `__set_task_cpu(task, cpu)`.
4. `ttwu_queue(task, cpu, wake_flags)`:
   - If target cpu == current cpu: enqueue locally (no IPI).
   - Else: append to target_rq->ttwu_pending lockless list + IPI target via `smp_call_function_single_async`.
5. Target CPU's IPI handler dequeues from ttwu_pending → `ttwu_do_activate(task)` → `enqueue_task(task)` → resched if higher prio than rq->curr.

CFS, RT, DEADLINE per-class details in respective Tier-3s.

Per-CPU IDLE: `arch::cpu_idle_loop` calls `tick_nohz_idle_enter` → check stop_handle_irq pending → `arch_cpu_idle()` (HLT / WFI / MWAIT) → on IRQ wakeup, `tick_nohz_idle_exit` → `__schedule(true)`.

Wake-Q batch: `wake_q_add(&wake_q, task)` adds task to per-caller list; later `wake_up_q(&wake_q)` walks list + invokes try_to_wake_up on each. Used by futex_unlock, mutex_unlock, etc.

CPU hotplug: `cpuhp_state_machine` invokes `migration_call(CPU_DOWN_PREPARE)` for each CPU going down; per-CPU `cpuhp_thread` migrates task at `migration_cpu_stop` step.

PSI (Pressure-Stall-Information) hooks in `__schedule`: per-cgroup CPU/IO/MEM pressure tracked via per-CPU psi_account_pcpu().

### Out of Scope

- CFS details (covered in `kernel/sched/cfs.md` Tier-3)
- RT class (covered in `kernel/sched/rt.md` Tier-3)
- DEADLINE class (covered in `kernel/sched/deadline.md` Tier-3)
- Load balance (covered in `kernel/sched/load-balance.md` Tier-3)
- Sched topology (covered in `kernel/sched/topology.md` Tier-3)
- Per-arch context switch (covered in `arch/x86/entry.md` Tier-3)
- 32-bit-only paths
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rq` | per-CPU runqueue | `kernel::sched::Rq` |
| `struct task_struct` | per-task control block | `kernel::task::TaskStruct` (cross-ref `kernel/00-overview.md`) |
| `struct sched_class` | per-scheduler-class vtable | `kernel::sched::SchedClass` (trait) |
| `__schedule(preempt)` | central dispatch — pick next task + context switch | `Scheduler::__schedule` |
| `schedule()` | voluntary schedule (no preempt) | `Scheduler::schedule` |
| `preempt_schedule()` / `_irq` / `_notrace` | involuntary preempt entry points | `Scheduler::preempt_schedule_*` |
| `try_to_wake_up(task, state, wake_flags)` | wake task that was blocked in `state` | `Scheduler::try_to_wake_up` |
| `wake_up_process(task)` | wake task in TASK_INTERRUPTIBLE / TASK_UNINTERRUPTIBLE | `Scheduler::wake_up_process` |
| `wake_up_state(task, state)` | wake task in specific state | `Scheduler::wake_up_state` |
| `wake_up_q(head)` | batch wake from a wakeup queue (built via `wake_q_add`) | `Scheduler::wake_up_q` |
| `set_user_nice(task, nice)` / `task_nice(task)` | per-task nice value | `TaskStruct::set_user_nice` / `Scheduler::task_nice` |
| `sched_setattr(task, &attr)` / `sched_getattr(task, ...)` | per-task scheduler attr (policy, nice, RT prio, deadline) | `Scheduler::sched_setattr` / `_getattr` |
| `sched_setaffinity(pid, &mask)` / `sched_getaffinity(pid, &mask)` | per-task CPU affinity | `Scheduler::sched_setaffinity` / `_getaffinity` |
| `sched_init()` | subsystem init (called from start_kernel) | `Subsystem::init` |
| `sched_init_smp()` | post-SMP init (sched_domains, RT bandwidth, etc.) | `Subsystem::init_smp` |
| `init_idle(task, cpu)` | per-CPU idle-task init | `Scheduler::init_idle` |
| `init_task` | the bootstrap PID 1 task | `kernel::task::INIT_TASK` |
| `cond_resched()` | voluntary preempt point in long kernel-mode loops | `cond_resched` macro |
| `schedule_timeout(timeout)` | sleep with hrtimer-backed wakeup | `Scheduler::schedule_timeout` |
| `io_schedule()` / `_timeout` | sleep with IO-account flag set | `Scheduler::io_schedule` |
| `update_rq_clock(rq)` | per-rq clock advance | `Rq::update_clock` |
| `enqueue_task(rq, p, flags)` / `dequeue_task(rq, p, flags)` | per-class enqueue/dequeue dispatch | `Rq::enqueue_task` / `_dequeue_task` |
| `set_next_task(rq, p, first)` | per-class set_next_task callback | `Rq::set_next_task` |
| `pick_next_task(rq, prev, &rf)` | per-class pick-next dispatch (CFS first if all CFS, else cascade through DEADLINE → RT → CFS → IDLE) | `Rq::pick_next_task` |
| `put_prev_task(rq, p)` | per-class put_prev callback | `Rq::put_prev_task` |
| `context_switch(rq, prev, next, &rf)` | actual MMU + register switch (calls `switch_to(prev, next, last)` arch-specific) | `Rq::context_switch` |
| `sched_fork(clone_flags, p)` | per-task init at fork | `Scheduler::sched_fork` |
| `sched_exec()` | per-task exec hook (CFS migration choice) | `Scheduler::sched_exec` |
| `sched_clock_*()` | per-CPU clock for accounting | `Sched::clock_*` |
| `cpu_load_update_*` family | per-CPU load tracking | `Rq::cpu_load_update_*` |
| `migration_call(...)` / `migrate_task(...)` | per-CPU hotplug migration | `Scheduler::migration_*` |

### compatibility contract

REQ-1: `__schedule(preempt)` invariant: chooses the highest-priority runnable task across all sched_classes (DEADLINE > RT > CFS > IDLE > STOP). DEADLINE+RT use absolute priority; CFS uses vruntime-min selection; STOP class only used during CPU hotplug + migration.

REQ-2: Per-CPU runqueue `struct rq` consistent invariants: `rq.nr_running == sum over per-class nr_running`; `rq.curr` always set to currently-running task on this CPU.

REQ-3: `try_to_wake_up(task, state, wake_flags)`:
- If `task.state & state == 0`: no-op (returned 0).
- Else: select target CPU via `select_task_rq(task, prev_cpu, wake_flags)` (per-class callback; CFS uses `select_task_rq_fair` w/ wake-affine + idle-balance heuristics).
- Enqueue task on target rq; if higher priority than current → IPI preempt (resched).

REQ-4: Wake-flags semantics: `WF_FORK` (newly forked), `WF_TTWU` (top-of-ttwu), `WF_SYNC` (synchronous wake e.g., pipe write→read), `WF_MIGRATED` (migrated cross-CPU), `WF_CURRENT_CPU` (wake on current CPU).

REQ-5: Voluntary preempt `cond_resched()` checks `_TIF_NEED_RESCHED` flag; if set, calls `__schedule(false)`.

REQ-6: Involuntary preempt at IRQ exit (`preempt_schedule_irq`): only when `preempt_count == 0` AND `_TIF_NEED_RESCHED` set.

REQ-7: Per-CPU IDLE thread: `init_idle(task, cpu)` creates per-CPU swapper task; runs IDLE class when nothing else runnable; halts CPU via per-arch `cpu_idle_loop` + WFI / HLT / MWAIT / wait-for-interrupt.

REQ-8: Per-task scheduler attr (policy + nice + RT prio + deadline runtime/period/deadline) byte-identical UAPI via sched_setattr(2) / sched_getattr(2) / sched_setscheduler(2) / sched_getscheduler(2) / sched_setparam(2) / sched_getparam(2).

REQ-9: Per-task affinity (cpus_allowed mask) byte-identical via sched_setaffinity(2) / sched_getaffinity(2). Per-task `cpus_ptr` enforced at every wake/migration; tasks never run on CPU not in cpus_ptr.

REQ-10: CPU hotplug: when CPU goes offline, all tasks pinned to that CPU migrated via `migration_call` to other CPUs in cpus_ptr; if no fallback CPU, fallback to `cpu_possible_mask` + warn.

REQ-11: Per-runqueue lock (`rq->lock`) acquired during enqueue / dequeue / context_switch; per-CPU runqueue lock-stealing (e.g., during balance pull) via double_rq_lock with canonical ordering (lower CPU first).

REQ-12: sched-tick (`scheduler_tick`) called from per-CPU tick (cross-ref `kernel/time/00-overview.md`); updates per-CPU clock + per-class scheduler_tick callback (CFS: vruntime + load-tracking; RT: time-slice; DEADLINE: budget).

REQ-13: Per-task accounting: `sched_info` (run-queue wait + CPU-time + last-arrival), `vtime` (per-task wall vs runtime), `psi` (Pressure-Stall-Info — per-cgroup CPU/IO/MEM pressure).

REQ-14: Wake-Q batched wake — for callers that wake multiple tasks in a critical section (e.g., release of N waiters of a futex/lock); avoids per-wake rq lock traffic.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `task_no_uaf` | UAF | `Arc<TaskStruct>` outlives any rq enqueue + per-class queue; `do_exit` defers free until task fully dequeued from all data structures. |
| `rq_lock_no_deadlock` | DEADLOCK | per-CPU rq->lock acquired in canonical order (lower CPU first) via double_rq_lock; no cycle possible. |
| `pick_next_no_starvation` | LIVENESS | every runnable task eventually picked (within bounded time per-policy: CFS via vruntime fairness, RT via priority, DEADLINE via deadline-EDF). |
| `cpus_ptr_respected` | INVARIANT | task never enqueued on CPU not in task->cpus_ptr (modulo CPU-hotplug fallback exception). |

### Layer 2: TLA+

`models/sched/runqueue_select.tla` (declared in parent's kernel/sched/00-overview): proves per-CPU runqueue selection on wake — `select_task_rq` + concurrent enqueue from N CPUs converges to a single CPU choice; no double-enqueue.
`models/sched/__schedule.tla`: proves __schedule + concurrent wake never produces stale rq->curr; context_switch atomicity.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `__schedule` post: rq->curr == returned-`next` task; prev not on this rq's runnable queue (if dequeued) | `Scheduler::__schedule` |
| `try_to_wake_up` post: returns 1 iff task transitioned from non-running to runnable + enqueued on some rq; returns 0 if no-op | `Scheduler::try_to_wake_up` |
| Per-rq nr_running invariant: equals sum over per-class nr_running | `Rq::nr_running` |
| Per-task at-most-one-rq invariant: task is on at most one rq at any time (modulo migration in-progress with both rq locks held) | enqueue / dequeue |

### Layer 4: Verus/Creusot functional

`Scheduler::__schedule + try_to_wake_up + cond_resched` round-trip equivalence: for any sequence of wakes + voluntary scheduling points, every runnable task eventually runs (no starvation), and per-task accounting (sum of cpu_time across all CPU intervals) equals wall-clock minus block-time. Encoded as Verus invariant on per-task time-tracking.

### hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

sched-core specific reinforcement:

- **Per-CPU rq lock acquired in canonical order** — defense against deadlock between CPUs.
- **Per-task `cpus_ptr` immutable during scheduling decision** — copy snapshot under per-task lock; defense against TOCTOU between affinity check + enqueue.
- **`__schedule` preempt_count check** — refuses to actually context-switch when in atomic context (preempt_count > 0); defense against scheduling-while-atomic bug; WARN + return.
- **CPU hotplug down-prepare migration of pinned tasks** — fallback to cpu_possible_mask if no in-cpus_ptr CPU available; WARN; defense against CPU hotplug deadlock.
- **`set_user_nice` / `sched_setattr` capability checks** — RT prio change requires CAP_SYS_NICE; DEADLINE attr requires CAP_SYS_NICE; cgroup-restricted via cpu controller.
- **Wake-Q batch bounded** — per-Q list length capped (default 64); over-cap auto-flushes; defense against wake-Q-flood DoS.
- **TIF_NEED_RESCHED check at preempt-irq exit** — every IRQ-exit path checks; missing check would lead to delayed preemption.
- **STOP class is privileged** — only used by per-CPU stop_machine; never settable from userspace.
- **Per-task vtime accounting** — defense against userspace time-stealing observation attacks via gettimeofday + getrusage.

