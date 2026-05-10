# Tier-3: drivers/gpu/drm/scheduler/sched_*.c — drm_sched generic GPU job scheduler (entity + fence-chain + per-rq priority queues)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/gpu/drm/00-overview.md
upstream-paths:
  - drivers/gpu/drm/scheduler/sched_main.c
  - drivers/gpu/drm/scheduler/sched_entity.c
  - drivers/gpu/drm/scheduler/sched_fence.c
  - include/drm/gpu_scheduler.h
-->

## Summary

`drm_sched` is the generic GPU command-submission scheduler — used by amdgpu (largest consumer), etnaviv, lima, msm, panfrost, panthor, v3d, and increasingly other drivers that previously rolled their own. Provides: per-GPU-engine `drm_gpu_scheduler` with N priority levels (NORMAL, HIGH, KERNEL by default + dynamic per-driver), per-context `drm_sched_entity` representing a single sequence of dependent jobs from one client, per-job dependency-fence chaining (job-A's output fence becomes job-B's input fence), kthread-per-scheduler workqueue execution, per-job timeout + GPU-reset, per-job complete callbacks, hardware-fence integration (driver-side scheduling thread waits for HW fence then signals job-finished fence).

This Tier-3 covers `drivers/gpu/drm/scheduler/` (3 files, ~2400 lines): `sched_main.c` (per-scheduler kthread + per-priority pick), `sched_entity.c` (per-entity job submission + fence-dependency tracking), `sched_fence.c` (per-job in-flight + finished fence pair).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct drm_gpu_scheduler` | per-engine scheduler | `drm::sched::Scheduler` |
| `struct drm_sched_entity` | per-context entity | `drm::sched::Entity` |
| `struct drm_sched_job` | per-job control block | `drm::sched::Job` |
| `struct drm_sched_fence` | per-job in-flight + finished fence pair | `drm::sched::SchedFence` |
| `struct drm_sched_rq` | per-priority sub-queue (one per scheduler per priority) | `drm::sched::Rq` |
| `drm_sched_init(sched, ops, name, num_rqs, ...)` | init scheduler | `Scheduler::init` |
| `drm_sched_fini(sched)` | finalize | `Scheduler::fini` |
| `drm_sched_entity_init(entity, priority, sched_list, num_sched_list, guilty)` | init entity | `Entity::init` |
| `drm_sched_entity_destroy(entity)` | finalize entity | `Entity::destroy` |
| `drm_sched_entity_set_priority(entity, priority)` | per-entity prio change | `Entity::set_priority` |
| `drm_sched_job_init(job, entity, owner)` | init per-job | `Job::init` |
| `drm_sched_job_arm(job)` | arm job (assign sequence + scheduled-fence) | `Job::arm` |
| `drm_sched_job_add_dependency(job, fence)` | add dependency | `Job::add_dependency` |
| `drm_sched_job_add_resv_dependencies(job, resv, usage)` | add all fences from a dma_resv | `Job::add_resv_dependencies` |
| `drm_sched_job_add_implicit_dependencies(job, obj, write)` | add per-GEM-obj implicit fences | `Job::add_implicit_dependencies` |
| `drm_sched_entity_push_job(job)` | submit job to entity (queues for scheduling) | `Entity::push_job` |
| `drm_sched_main(arg)` | per-scheduler kthread main loop | `Scheduler::main_loop` |
| `drm_sched_entity_pop_job(entity)` | pick next runnable job from entity | `Entity::pop_job` |
| `drm_sched_select_entity(sched)` | per-priority round-robin entity pick | `Scheduler::select_entity` |
| `drm_sched_resubmit_jobs(sched)` | re-queue all jobs after GPU reset | `Scheduler::resubmit_jobs` |
| `drm_sched_stop(sched, bad_job)` / `_start(sched, full_recovery)` | per-reset stop/start | `Scheduler::stop` / `_start` |
| `drm_sched_increase_karma(bad_job)` | karma per-context-fault tracking | `Scheduler::increase_karma` |
| `drm_sched_reset_karma(bad_job)` | reset karma | `Scheduler::reset_karma` |
| `drm_sched_fence_create(entity, owner)` | alloc SchedFence pair | `SchedFence::create` |
| `drm_sched_fence_scheduled(fence)` | signal scheduled fence | `SchedFence::signal_scheduled` |
| `drm_sched_fence_finished(fence)` | signal finished fence | `SchedFence::signal_finished` |
| `drm_sched_job_recovery(sched)` | post-reset job recovery callback | per-driver `Ops::recovery` |
| `drm_sched_backend_ops` | per-driver vtable (`prepare_job`, `run_job`, `timedout_job`, `free_job`) | `drm::sched::BackendOps` (trait) |

## Compatibility contract

REQ-1: Per-scheduler `kthread` `drm/sched-<name>` runs the scheduling main loop; per-scheduler 1 kthread.

REQ-2: Per-priority round-robin: scheduler picks entity from highest-priority rq with runnable jobs first; round-robin within a priority level.

REQ-3: Per-entity job ordering: jobs pushed via `drm_sched_entity_push_job` execute in submission order (FIFO per-entity); preserves per-context determinism.

REQ-4: Per-job dependency-fence chain: `drm_sched_job_add_dependency(job, fence)` adds fence to per-job dep-list; scheduler waits until all deps signal before invoking `run_job` callback.

REQ-5: Per-job in-flight fence (scheduled-fence) signals when scheduler picks job + invokes `run_job`; per-job finished-fence signals when HW fence (returned from `run_job`) signals.

REQ-6: Per-driver `drm_sched_backend_ops` vtable:
- `prepare_job(sched_job, entity)` → returns optional dep fence (e.g., GPU page-fault fence).
- `run_job(sched_job)` → submit to HW; returns HW fence.
- `timedout_job(sched_job)` → called when job timeout fires; per-driver does GPU reset OR per-job retry.
- `free_job(sched_job)` → release per-driver job resources after finished-fence signals.

REQ-7: Per-job timeout: scheduler arms timeout-timer when invoking `run_job`; expiry triggers `timedout_job` callback; karma incremented per-context (used to detect persistent-faulting client).

REQ-8: GPU-reset support: per-driver invokes `drm_sched_stop(sched, bad_job)` to halt scheduling + cancel in-flight jobs; resets HW; calls `drm_sched_resubmit_jobs(sched)` to re-issue still-pending jobs; `drm_sched_start(sched, full_recovery)` resumes scheduling.

REQ-9: Per-entity migration: `drm_sched_entity_modify_sched(entity, sched_list, num_sched_list)` re-binds entity to new scheduler set; used when GPU context migrates between engines.

REQ-10: Per-entity guilty-flag: when scheduler detects GPU hang caused by this entity, `entity->guilty = true` set; subsequent job submissions return -ECANCELED.

REQ-11: Per-job RT priority: `drm_sched_entity_init(priority=...)`. Up to MAX_DRM_SCHED_PRIORITY (default 4: KERNEL + HIGH + NORMAL + LOW).

REQ-12: Fence handle UAPI: per-job finished-fence exported via syncobj or sync-file fd to userspace for dependency-tracking across processes (cross-ref `drm-syncobj.md` future Tier-3).

## Acceptance Criteria

- [ ] AC-1: amdgpu workload (Vulkan compute or 3D) submits jobs through drm_sched; per-job timeline visible via `/sys/kernel/debug/dri/0/amdgpu_*` debugfs.
- [ ] AC-2: panfrost (Mali) workload — Vulkan via Panvk submits jobs through drm_sched.
- [ ] AC-3: GPU-reset stress: artificial GPU hang via debugfs → drm_sched detects timeout → per-driver `timedout_job` reset → still-pending jobs re-issued; subsequent submissions succeed.
- [ ] AC-4: Per-priority test: KERNEL-prio + HIGH-prio + NORMAL-prio entities; KERNEL preempts; observed via tracepoints.
- [ ] AC-5: Dependency-chain test: 3-deep job dependency (A → B → C); B's run_job invoked after A's HW fence signals; C's run_job invoked after B's signals.
- [ ] AC-6: kselftest drm_sched-related tests pass.

## Architecture

`Scheduler` lives in `drm::sched::Scheduler`:

```
struct Scheduler {
  ops: &'static dyn BackendOps,
  hw_submission_limit: u32,
  hang_limit: u32,                   // karma threshold
  timeout: Duration,
  name: KString,
  num_rqs: u32,
  sched_rq: VarLenArray<KBox<Rq>>,    // per-priority rq
  job_scheduled: WaitQueue,           // signal when job scheduled
  hw_rq_count: AtomicU32,
  job_id_count: AtomicU64,
  thread: Arc<KernelThread>,           // per-scheduler kthread
  pending_list: Mutex<LinkedList<Arc<Job>>>,
  job_list_lock: SpinLock<()>,
  free_guilty: AtomicBool,
  ready: AtomicBool,
  dev: Arc<DrmDevice>,
}

struct Entity {
  refcount: Refcount,
  sched_list: KBox<[NonNull<Scheduler>]>,
  num_sched_list: u32,
  priority: SchedPriority,
  current_sched_idx: AtomicU32,
  guilty: AtomicBool,
  fence_context: u64,
  job_queue: SpscQueue<Arc<Job>>,
  dependency: Mutex<XArray<Arc<DmaFence>>>,
  fini_status: AtomicI32,
  last_scheduled: AtomicPtr<DmaFence>,
  last_user: AtomicPtr<()>,
  rq: AtomicPtr<Rq>,
  rq_lock: SpinLock<()>,
  finished_cb: DmaFenceCallback,
}

struct Job {
  refcount: Refcount,
  sched: Arc<Scheduler>,
  s_fence: KBox<SchedFence>,           // scheduled-fence + finished-fence pair
  deps: Mutex<XArray<Arc<DmaFence>>>,
  cb: DmaFenceCallback,
  list: ListEntry,
  entity: Arc<Entity>,
  guilty: bool,
  s_priority: SchedPriority,
  id: u64,
  karma: u32,
  hw_fence: AtomicPtr<DmaFence>,        // returned from run_job
}

struct SchedFence {
  scheduled: DmaFence,
  finished: DmaFence,
  parent: AtomicPtr<DmaFence>,         // hw fence
  sched: Arc<Scheduler>,
  owner: NonNull<()>,
}
```

Init `Scheduler::init(sched, ops, name, num_rqs, hw_submission, hang_limit, timeout)`:
1. Initialize sched_rq array (one per priority).
2. Spawn per-scheduler kthread `drm/sched-<name>` running `Scheduler::main_loop`.

Job lifecycle:
1. Driver calls `Job::init(job, entity, owner)`:
   - `SchedFence::create(entity, owner)` → alloc scheduled+finished fence pair.
   - Init deps xarray.
2. Driver calls `Job::add_dependency(job, fence)` for each upstream fence.
3. Driver calls `Job::arm(job)` to assign sequence number + finalize fences.
4. Driver calls `Entity::push_job(job)`:
   - Append to entity's job_queue (SPSC).
   - If entity not on any rq: `Entity::set_rq(rq)` + add to rq's runnable_list.
   - Wake `Scheduler.job_scheduled`.

`Scheduler::main_loop` (kthread):
```
loop {
  wait_event_interruptible(job_scheduled, hw_rq_count < hw_submission_limit && entity = select_entity());
  if kthread_should_stop() break;
  let entity = Scheduler::select_entity(sched);  // per-priority round-robin
  let job = Entity::pop_job(entity);
  if let Some(job) = job {
    // Wait for all deps to signal (or arm callbacks)
    if let Some(dep) = Entity::dependency(entity) {
      // dep is a not-yet-signaled fence; install callback to retry when it signals
      job.s_fence.scheduled.signal();  // signal scheduled-fence
      install_callback(dep, &job);     // wakeup main_loop when dep signals
      continue;
    }
    // All deps signaled
    SchedFence::signal_scheduled(&job.s_fence);
    let hw_fence = ops.run_job(job);   // per-driver submits to HW
    job.hw_fence.store(hw_fence);
    install_callback(hw_fence, |fence| SchedFence::signal_finished(&job.s_fence));
    pending_list.push_back(job);
    arm_timeout(job, sched.timeout);
  }
}
```

`Entity::pop_job(entity)`:
1. Peek entity.job_queue.head().
2. If all deps signaled: pop + return job.
3. Else: return None (caller installs callback to wake when dep signals).

GPU reset flow:
1. Driver detects GPU hang.
2. `drm_sched_stop(sched, bad_job)`:
   - Cancel pending timeout for bad_job.
   - For each pending job: detach hw_fence callback (HW will be reset).
3. Driver does HW reset (per-driver `recovery` callback).
4. `drm_sched_resubmit_jobs(sched)`:
   - For each pending job: re-call `ops.run_job(job)` to re-issue.
5. `drm_sched_start(sched, full_recovery=true)`: resume scheduling.

Karma: `drm_sched_increase_karma(bad_job)` walks all entities w/ jobs in pending_list; if entity's job count contributing to bad batch > threshold, mark entity guilty. Subsequent job pushes return -ECANCELED.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `job_no_uaf` | UAF | `Arc<Job>` outlives all dma_fence callbacks; release waits for finished-fence callback chain to complete. |
| `entity_no_uaf` | UAF | `Arc<Entity>` outlives all in-flight jobs; destroy waits for entity's pending jobs to drain. |
| `dep_chain_no_cycle` | TERMINATION | dep-fence install bounded; no infinite-deferral via cyclic dep (per-DAG). |
| `hw_rq_count_no_overflow` | OVERFLOW | hw_submission_limit enforced; hw_rq_count never exceeds. |

### Layer 2: TLA+

`models/drm/sched_fence_chain.tla` (parent-declared): proves drm_sched job-dependency chain — concurrent enqueue on N entities under scheduler thread; job-fence-signal triggers dependent jobs in FIFO order per priority; in-flight cancellation always completes pending fences with -ECANCELED.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Scheduler::main_loop` invariant: every job has scheduled-fence signal preceding finished-fence signal | `Scheduler::main_loop` |
| Per-entity FIFO: pop_job returns jobs in push_job order | `Entity::pop_job` |
| Karma update: entity.guilty set iff bad-batch karma > hang_limit | `Scheduler::increase_karma` |
| `pending_list` invariant: every job in pending_list has hw_fence != NULL (run_job returned) | `Scheduler::main_loop` |

### Layer 4: Verus/Creusot functional

`Job::push → main_loop pick → run_job → hw_fence signal → finished_fence signal → free_job` round-trip: every successfully-scheduled job's finished-fence eventually signals (modulo GPU hang + reset path). Encoded as Verus liveness invariant.

## Hardening

(Inherits row-1 features from `drivers/gpu/drm/00-overview.md` § Hardening.)

drm_sched specific reinforcement:

- **Per-job timeout default-on** — every job armed with timeout when run_job invoked; defense against silent GPU-hang freezing scheduler.
- **Karma-based entity-guilty marking** — persistent-faulting entities marked guilty + reject new submissions; defense against GPU-bomb client repeatedly hanging GPU.
- **Per-scheduler hw_submission_limit** — bounded in-flight jobs per scheduler; defense against single client flooding scheduler causing OOM via deferred-completion accumulation.
- **Per-job dep-fence install callback bounded** — per-fence callback list cap (default 64); defense against fence-callback-flood DoS.
- **GPU reset path drains pending callbacks** — on stop_sched, all hw-fence callbacks detached safely; defense against UAF on reset.
- **Per-entity fence_context unique** — no two entities share fence_context (defense against cross-entity fence-confusion).
- **Per-priority strict ordering** — KERNEL never starved by HIGH/NORMAL; defense against userspace DoS via low-prio flood blocking system processes.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-driver run_job + timedout_job impl (covered in `amdgpu.md`, `panfrost.md`, `lima.md`, `etnaviv.md`, `msm.md`, `v3d.md`, `panthor.md` future Tier-3s)
- dma_fence + dma_resv impl (covered in `kernel/dma/dmabuf-core.md` future Tier-3)
- syncobj (covered in `drm-syncobj.md` future Tier-3)
- 32-bit-only paths
- Implementation code
