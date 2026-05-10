# Tier-3: block/io-schedulers — elevator + mq-deadline + kyber + bfq

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - block/elevator.c
  - block/mq-deadline.c
  - block/kyber-iosched.c
  - block/bfq-iosched.c
  - block/bfq-cgroup.c
  - block/bfq-wf2q.c
-->

## Summary
Tier-3 design for the block-layer I/O schedulers and the elevator framework that dispatches requests to drivers. Owns three pluggable schedulers — `mq-deadline` (default for SSDs/rotational; deadline-based fairness), `kyber` (latency-targeted; tunes by observed completion time), `bfq` (Budget Fair Queueing; cgroup-aware, weight-based, default for HDDs in many distros) — plus the per-queue elevator framework that manages scheduler attach/detach and dispatch.

Sub-tier-3 of `block/00-overview.md`. Userspace selects the scheduler via `/sys/block/<dev>/queue/scheduler`. Different workloads (latency-sensitive desktop vs. throughput-bulk batch) prefer different schedulers.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Elevator framework | `block/elevator.c` |
| mq-deadline | `block/mq-deadline.c` |
| kyber | `block/kyber-iosched.c` |
| bfq | `block/bfq-iosched.c`, `block/bfq-cgroup.c`, `block/bfq-wf2q.c` |

## Compatibility contract

### `/sys/block/<dev>/queue/scheduler` writeable knob

`echo mq-deadline > /sys/block/sda/queue/scheduler` switches active scheduler. Read returns `[active] available1 available2 …`. Format-identical.

### sysfs scheduler-specific tunables

Each scheduler exposes per-queue tunables under `/sys/block/<dev>/queue/iosched/*`:

- mq-deadline: `read_expire`, `write_expire`, `writes_starved`, `front_merges`, `fifo_batch`, `prio_aging_expire`, `async_depth`
- kyber: `read_lat_nsec`, `write_lat_nsec`
- bfq: `slice_idle`, `back_seek_max`, `back_seek_penalty`, `low_latency`, `max_budget`, `max_budget_async`, `slice_idle_us`, `idle_busy_short`, `strict_guarantees`, `timeout_async`, `timeout_sync`, `weights_per_node`

Format-identical so existing tunables (e.g., systemd's BFQ tweaks) work unchanged.

### bfq cgroup integration

bfq integrates with cgroup v1 `blkio` and v2 `io` controllers — per-cgroup weight + IO-rate limits. Cross-ref `block/cgroup-throttle.md`.

### Default selection

Per `Documentation/block/`:
- Rotational devices (`/sys/block/<dev>/queue/rotational=1`) — default to `bfq`
- Non-rotational devices (`rotational=0`) — default to `mq-deadline`
- Userspace can override via udev rules

Identical defaults.

## Requirements

- REQ-1: Three schedulers (mq-deadline, kyber, bfq) implemented; each registers via `elv_register`.
- REQ-2: Elevator framework: `elv_register` / `elv_unregister`; per-queue `elevator_init` / `elevator_exit`; dispatch hook `dispatch_request`. Identical.
- REQ-3: `/sys/block/<dev>/queue/scheduler` r/w knob format-identical.
- REQ-4: Per-scheduler sysfs tunables format + value semantics identical.
- REQ-5: mq-deadline FIFO + deadline ordering algorithm identical to upstream.
- REQ-6: kyber latency-targeting algorithm identical (observes per-bucket completion times; throttles to maintain target).
- REQ-7: bfq Budget Fair Queueing: per-cgroup weight + per-process budget + WF2Q+ scheduling identical.
- REQ-8: Default scheduler selection (per rotational flag) matches upstream.
- REQ-9: Scheduler switch atomicity: writes to /sys/block/<dev>/queue/scheduler quiesce in-flight requests, swap, resume. No leaked requests.
- REQ-10: bfq cgroup integration: per-cgroup `blkio` (v1) / `io` (v2) controller knobs work unchanged.
- REQ-11: TLA+ model `models/block/elevator.tla` (mandatory per `block/00-overview.md` Layer 2) proves I/O scheduler ordering invariants per scheduler.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `cat /sys/block/<dev>/queue/scheduler` lists `[mq-deadline] kyber bfq none` (or appropriate active marker). (covers REQ-3)
- [ ] AC-2: For each scheduler, a fio benchmark on null_blk shows latency + throughput within ±5% of upstream. (covers REQ-1, REQ-5, REQ-6, REQ-7)
- [ ] AC-3: For each scheduler's sysfs tunable, a r/w round-trip returns the written value. (covers REQ-4)
- [ ] AC-4: A scheduler-switch test (`echo kyber > /sys/block/<dev>/queue/scheduler` while in-flight) doesn't drop requests. (covers REQ-9)
- [ ] AC-5: A `rotational=1` virtual device defaults to bfq; `rotational=0` defaults to mq-deadline. (covers REQ-8)
- [ ] AC-6: A bfq + cgroup v2 weight test: two cgroups with weights 1:3 receive proportional I/O bandwidth. (covers REQ-10)
- [ ] AC-7: `make tla` passes `models/block/elevator.tla` for each of mq-deadline/kyber/bfq. (covers REQ-11)
- [ ] AC-8: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::block::elevator::Elevator` — framework
- `kernel::block::elevator::dispatch::Dispatch` — `dispatch_request` hook
- `kernel::block::sched::mq_deadline::MqDeadline` — mq-deadline impl
- `kernel::block::sched::kyber::Kyber` — kyber impl
- `kernel::block::sched::bfq::Bfq` — bfq core
- `kernel::block::sched::bfq::wf2q::Wf2qScheduler` — WF2Q+ algorithm
- `kernel::block::sched::bfq::cgroup::BfqCgroup` — bfq cgroup integration

### Locking and concurrency

- **`elv_list_lock`** (spinlock): per-system elevator-registry lock
- **Per-queue `q->mq_freeze_lock`**: held during scheduler swap
- **Per-scheduler internal locks**: each scheduler owns its own (mq-deadline FIFO list lock, kyber bucket lock, bfq tree lock)

TLA+ model `models/block/elevator.tla` proves per-scheduler ordering invariants.

### Error handling

- `Err(EINVAL)` — bad scheduler name in `/sys/block/.../scheduler` write
- `Err(ENOENT)` — scheduler not registered
- `Err(EBUSY)` — scheduler swap in progress

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Elevator dispatch hook | `kani::proofs::block::elevator::dispatch_safety` |
| mq-deadline FIFO list manipulation | `kani::proofs::block::sched::mq_deadline_safety` |
| kyber bucket-token manipulation | `kani::proofs::block::sched::kyber_safety` |
| bfq weighted tree manipulation | `kani::proofs::block::sched::bfq_tree_safety` |
| Scheduler swap (quiesce + drain + resume) | `kani::proofs::block::elevator::swap_safety` |

### Layer 2: TLA+ models

- `models/block/elevator.tla` (mandatory per `block/00-overview.md` Layer 2) — proves per-scheduler ordering invariants. Owned here (across all three schedulers).

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| mq-deadline FIFO lists | read + write FIFOs each address-ordered; deadline lists honor expire times | `kani::proofs::block::sched::mq_deadline_invariants` |
| kyber per-bucket queue | each bucket's queue ≤ bucket capacity | `kani::proofs::block::sched::kyber_invariants` |
| bfq WF2Q+ tree | tree maintains the WF2Q+ invariant: virtual time monotonically increasing | `kani::proofs::block::sched::bfq_invariants` |

### Layer 4: Functional correctness (opt-in)

- **WF2Q+ scheduling fairness theorem** via Verus — proves: under fair scheduling assumption, each cgroup receives weight-proportional I/O bandwidth on average.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

(none directly; schedulers consume hardening from queue + bio)

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-scheduler-internal-state slab caches
- **CONSTIFY**: per-scheduler ops vtable static const
- **SIZE_OVERFLOW**: budget + virtual-time arithmetic uses checked operators

### Row-2 / GR-RBAC integration

bfq cgroup integration is the LSM-relevant interaction point. GR-RBAC's policy can deny `cgroup.io.weight` writes via `security_path_chmod` on the cgroupfs file (cross-ref `kernel/cgroup/00-overview.md`).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — scheduler semantics are exhaustively specified)

## Out of Scope

- Per-cgroup throttling beyond bfq (cross-ref `block/cgroup-throttle.md`)
- 32-bit-only paths
- Implementation code
