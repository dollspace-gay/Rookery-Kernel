---
title: "Tier-3: kernel/sched/rt — POSIX real-time scheduling (SCHED_FIFO + SCHED_RR)"
tags: ["design-doc", "tier-3", "scheduler"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the Linux real-time scheduling class — `SCHED_FIFO` (no time slice; preempted only by higher-priority RT or SCHED_DEADLINE) and `SCHED_RR` (FIFO with per-priority round-robin time slice). Implements POSIX 1003.1b + 1003.1c real-time semantics. RT class outranks CFS; SCHED_DEADLINE outranks RT.

Sub-tier-3 of `kernel/sched/00-overview.md`. Pairs with `kernel/sched/cfs.md` (regular tasks) and `kernel/sched/deadline.md` (EDF/CBS). Used by audio servers (PulseAudio/PipeWire RT threads), industrial-control kernels, networking dataplanes (DPDK PMDs sometimes pin-+-RT), and any latency-bounded workload.

### Requirements

- REQ-1: SCHED_FIFO + SCHED_RR semantics identical: priority outranks CFS; FIFO has no time slice, RR has 100ms-default round-robin.
- REQ-2: Priority range 1..99 for both classes; sched_setscheduler validates.
- REQ-3: Per-CPU `rt_rq`: per-prio `struct list_head` array + active bitmap; pick-next-task walks bitmap in O(1).
- REQ-4: RT throttling: per-1s period, 95% RT runtime default; cgroup `rt_period_us` / `rt_runtime_us` per-cgroup override identical.
- REQ-5: `sched_rr_get_interval` returns time slice; tunable via `/proc/sys/kernel/sched_rr_timeslice_ms`.
- REQ-6: cpupri (per-CPU priority tracking): O(1) `cpupri_find` returns CPU mask of CPUs running tasks at-or-below given prio. Identical algorithm.
- REQ-7: push/pull migration: when local rq gains a task with prio higher than remote-running task, push migrates; symmetrically pull. Heuristics identical.
- REQ-8: Priority inheritance for futex+mutex: RT mutex inherits priority of waiter (covered in `kernel/locking/rtmutex.md` Tier-3); RT class consumes that interface.
- REQ-9: Boost interaction: SCHED_RR/FIFO tasks blocked on rt_mutex held by lower-prio task → lender boosted; rt class re-queues post-boost identically.
- REQ-10: stop_task class above RT (kernel/sched/stop_task.c) — RT class never preempts a stop_task. Identical.
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `chrt -f 50 -p $$ ; chrt -p $$` returns `policy: SCHED_FIFO`, `priority: 50`. (covers REQ-1, REQ-2)
- [ ] AC-2: A SCHED_FIFO prio-50 task spinning a tight loop on CPU N, while a CFS-class task on the same CPU is starved → CFS task receives ≥ 5% of CPU time per second (RT throttle preserves 5%). (covers REQ-4)
- [ ] AC-3: Two SCHED_RR tasks at the same priority alternate every 100ms ± kernel-tick error. (covers REQ-1, REQ-5)
- [ ] AC-4: A 4-CPU machine with 4 SCHED_FIFO prio-99 tasks spawned on CPU 0 → tasks distribute across all 4 CPUs (push-migrate working). (covers REQ-7)
- [ ] AC-5: cpupri_find unit-test: registers tasks at varied prios across CPUs; for each query prio, returns correct mask. (covers REQ-6)
- [ ] AC-6: A SCHED_FIFO prio-50 task blocked on rt_mutex held by a SCHED_OTHER (CFS) task → CFS task boosted to prio 50; rt class queues it; on unlock, CFS task de-boosts and is back in CFS. (covers REQ-8, REQ-9)
- [ ] AC-7: A cgroup with `rt_runtime_us=500000` (50%) running an RT-50 hog → only 500ms / period of CPU consumed. (covers REQ-4)
- [ ] AC-8: TLA+ model `models/sched/rt_class.tla` proves: at every instant, the running task on any CPU is the highest-prio RT-runnable task on that CPU (modulo push/pull in flight + active throttling). (covers REQ-3)
- [ ] AC-9: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::sched::rt::RtClass` — main class entrypoint
- `kernel::sched::rt::queue::RtRq` — per-CPU per-prio runqueue
- `kernel::sched::rt::pick::PickNextRt` — bitmap-walk highest-prio
- `kernel::sched::rt::push_pull::PushPull` — migration
- `kernel::sched::rt::throttle::RtThrottle` — period/runtime accounting
- `kernel::sched::rt::cpupri::CpuPri` — per-CPU prio tracker

### Locking and concurrency

- **Per-CPU rq lock** (already held by core scheduler): protects rt_rq state
- **`cpupri_lock`** (per-cpupri-vector spinlock): cpupri vector mutator
- **rt_b->rt_runtime_lock** (per-cgroup throttle accounting): protects rt_runtime / rt_time

The push/pull paths use `double_lock_balance` to acquire both source + destination rq locks; deadlock-free because of always-ordered-by-CPU-id locking.

### Error handling

- `Err(EINVAL)` — bad priority
- `Err(EPERM)` — non-CAP_SYS_NICE attempting RT for non-self target (per RLIMIT_RTPRIO)
- `Err(EBUSY)` — task in invalid state for prio change

### Out of Scope

- SCHED_DEADLINE (cross-ref `kernel/sched/deadline.md`)
- SCHED_OTHER / SCHED_BATCH / SCHED_IDLE (cross-ref `kernel/sched/cfs.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| RT scheduling class (per-prio runqueues, push/pull, throttle) | `kernel/sched/rt.c` |
| Per-CPU priority tracking (`cpupri`) for push-decision | `kernel/sched/cpupri.c`, `kernel/sched/cpupri.h` |
| Public API | `include/linux/sched/rt.h` |
| UAPI | `include/uapi/linux/sched.h` (SCHED_FIFO=1, SCHED_RR=2, MAX_RT_PRIO=100) |

### compatibility contract

### `sched_setscheduler(pid, policy, param)` semantics

| Policy | Constants | Priority range | Behavior |
|---|---|---|---|
| `SCHED_FIFO` | 1 | 1..99 | Runs until blocks or yields; only preempted by higher-prio RT/DL or stop_task |
| `SCHED_RR` | 2 | 1..99 | Like FIFO but per-prio round-robin time slice (default 100ms via `sched_rr_get_interval`) |

Identical priority semantics: priority 99 = highest; priority 1 = lowest RT (still above any CFS task).

### `sched_rr_get_interval(pid, &timespec)`

Returns RR time slice. Default 100ms; tunable via `/proc/sys/kernel/sched_rr_timeslice_ms`. Identical.

### RT throttling — `/proc/sys/kernel/sched_rt_period_us` + `sched_rt_runtime_us`

Default: `1000000` (1s) period, `950000` (950ms) runtime. RT tasks run at most 95% of any 1s period — preserves 5% to non-RT to prevent priority inversion deadlock.

`sched_rt_runtime_us = -1` disables throttling. cgroup `rt_period_us` / `rt_runtime_us` per-cgroup override.

Identical defaults and tunables. Userspace `chrt -p`, `taskset`, `systemd` RT-thread setups work unchanged.

### `struct sched_rt_entity` layout

`include/linux/sched.h`: per-task RT entity. Contains `run_list` (per-prio listhead linkage), `timeout`, `watchdog_stamp`, `rt_priority`, `time_slice`, `on_rq`. Layout-equivalent.

### `struct rt_rq` layout

`kernel/sched/sched.h`: per-CPU per-prio array of `struct list_head` (`MAX_RT_PRIO=100` entries) + `bitmap` (DECLARE_BITMAP(MAX_RT_PRIO+1)). The bitmap supports O(1) `find_first_bit` to pick highest-prio queued task. Layout-equivalent.

### Push / pull migration

When a CPU's RT runqueue gains a task whose priority ≤ a remote CPU's currently-running task, `push_rt_tasks` migrates the task off; conversely `pull_rt_tasks` pulls higher-prio tasks from overloaded CPUs onto idle ones. Heuristics identical so RT-pinned setups behave identically.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `dequeue_task_rt` per-prio list_head removal | `kani::proofs::sched::rt::dequeue_safety` |
| `enqueue_task_rt` per-prio list_head insertion + bitmap update | `kani::proofs::sched::rt::enqueue_safety` |
| `pick_next_rt_entity` bitmap-walk (no off-by-one) | `kani::proofs::sched::rt::pick_safety` |
| push/pull double-lock | `kani::proofs::sched::rt::push_pull_safety` |
| cpupri vector update | `kani::proofs::sched::rt::cpupri_safety` |

### Layer 2: TLA+ models

- `models/sched/rt_class.tla` (mandatory per `kernel/sched/00-overview.md` Layer 2 — RT-fairness invariant) — proves: at every step, on every CPU, the running task is the highest-prio runnable RT/DL/stop_task task (or CFS-task if no higher-class runnable). Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-CPU rt_rq prio bitmap | `bit i` set ⇔ `rt_rq->active.queue[i]` non-empty (consistent at all times outside locked critical sections) | `kani::proofs::sched::rt::bitmap_invariants` |
| cpupri vector | each CPU's recorded prio matches actual rq->highest_prio at every visible state | `kani::proofs::sched::rt::cpupri_invariants` |

### Layer 4: Functional correctness (opt-in)

- **RT priority inversion bound theorem** via Verus — proves: with priority inheritance enabled (rt_mutex), the maximum blocking time of a SCHED_FIFO/RR task is bounded by Σ critical-section lengths of lower-prio holders, not unbounded.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | rt_bandwidth structures use `Refcount` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-task sched_rt_entity allocations
- **CONSTIFY**: `sched_class` ops vtable for `rt_sched_class` is `static const`
- **SIZE_OVERFLOW**: prio + bitmap-index arithmetic uses checked operators

### Row-2 / GR-RBAC integration

- LSM hook `security_task_setscheduler` (already standard in upstream); GR-RBAC policy can deny `sched_setscheduler` for SCHED_FIFO/RR per-subject.
- A useful default GR-RBAC policy fragment (defined in `security/grsec/00-overview.md`): deny SCHED_FIFO/RR for processes outside the gradm-marked-RT-allowed role; default empty so userspace not surprised.
- DoS-prevention: RLIMIT_RTPRIO + RLIMIT_RTTIME apply identically; `sched_rt_runtime_us` 95%-cap defends against stuck-RT-loop livelocking the system.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

