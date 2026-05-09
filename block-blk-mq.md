---
title: "Tier-3: block/blk-mq — multi-queue request layer"
tags: ["design-doc", "tier-3", "block"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for blk-mq — the multi-queue block request layer. Owns per-CPU software-queue (`ctx`) → per-hardware-queue (`hctx`) submission machinery, the per-hctx tag bitmap allocator, request lifecycle (alloc → dispatch → complete → free), I/O-scheduler integration (mq-deadline / kyber / bfq dispatched per-hctx), DMA-mapping integration, sysfs + debugfs surfaces, and the rq-completion path with timeout handling.

Sub-tier-3 of `block/00-overview.md`. blk-mq replaced the legacy single-queue request_fn API in upstream's 5.0 era; Rookery does NOT preserve the legacy API per `block/00-overview.md` REQ-1.

### Requirements

- REQ-1: blk-mq is the only block request layer (no legacy single-queue per `block/00-overview.md` REQ-1).
- REQ-2: `struct request` first-cache-line layout-equivalent.
- REQ-3: Per-CPU software-queue (`ctx`) → per-hardware-queue (`hctx`) mapping per upstream's `blk-mq-cpumap.c`.
- REQ-4: Per-hctx tag bitmap: lock-free atomic-set bitmap; per-tag fast-allocation. Reserved tags + normal tags partitioned per upstream's `blk_mq_tags`.
- REQ-5: Request lifecycle: `blk_mq_alloc_request` → submit → driver `queue_rq` callback → driver completion → `blk_mq_end_request` → free. State machine matches upstream's `enum mq_rq_state`.
- REQ-6: I/O scheduler dispatch: when CONFIG_MQ_IOSCHED_*, scheduler holds requests; on `__blk_mq_run_hw_queue`, scheduler chooses + dispatches. Identical hooks (`elevator_*`).
- REQ-7: Timeout handling: per-request deadline; on expiry, `blk_mq_check_expired` reaps + retries. Identical state machine.
- REQ-8: DMA-mapping integration: blk-mq builds scatter-list per request; `blk_rq_map_sg` produces upstream-equivalent SG list.
- REQ-9: Per-hctx srcu: protects in-flight requests during scheduler / device transitions.
- REQ-10: blk-mq sysfs + debugfs format-identical.
- REQ-11: TLA+ models (per `block/00-overview.md` Layer 2): `blk_mq_tag_alloc.tla` (no double-allocation), `request_state_machine.tla` (state transitions honor invariants).
- REQ-12: Layer-3 mandatory invariants (per `block/00-overview.md`): tag bitmap exclusivity (each tag in exactly one of free/allocated/reserved); request list non-duplication.
- REQ-13: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: A grep over Rookery for `blk_init_queue` (legacy) finds zero matches. (covers REQ-1)
- [ ] AC-2: `pahole struct request` first-cache-line byte-identical. (covers REQ-2)
- [ ] AC-3: A 16-CPU concurrent fio benchmark on null_blk shows even per-CPU ctx → hctx distribution. (covers REQ-3, REQ-4)
- [ ] AC-4: A request lifecycle test (allocate, submit, driver completes, free) traces through every state transition; no leaked tags. (covers REQ-5)
- [ ] AC-5: With `mq-deadline`, `kyber`, or `bfq` selected via `/sys/block/<dev>/queue/scheduler`, requests dispatch through the scheduler hook. (covers REQ-6)
- [ ] AC-6: A timeout-injection test: driver doesn't complete a request within timeout; `blk_mq_check_expired` reaps; subsequent submission succeeds. (covers REQ-7)
- [ ] AC-7: A scatter-list test on a multi-page bio produces identical SG list vs. upstream. (covers REQ-8)
- [ ] AC-8: A scheduler-switch test (`echo none > /sys/block/.../scheduler` while in-flight) doesn't lose requests; srcu protects. (covers REQ-9)
- [ ] AC-9: Diff of `cat /sys/block/null_blk0/mq/0/*` byte-identical (modulo dynamic counts). (covers REQ-10)
- [ ] AC-10: `make tla` passes `models/block/blk_mq_tag_alloc.tla` and `models/block/request_state_machine.tla`. (covers REQ-11)
- [ ] AC-11: `make verify` passes mandatory L3 harnesses. (covers REQ-12)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

### Architecture

### Rust module organization

- `kernel::block::mq::Request` — `struct request` wrapper
- `kernel::block::mq::Hctx` — per-hardware-queue
- `kernel::block::mq::Ctx` — per-CPU software queue
- `kernel::block::mq::Tag` — per-hctx tag bitmap
- `kernel::block::mq::Dispatch` — `__blk_mq_run_hw_queue`
- `kernel::block::mq::Sched` — scheduler hook integration
- `kernel::block::mq::Timeout` — request timeout handling
- `kernel::block::mq::Cpumap` — per-CPU → hctx mapping
- `kernel::block::mq::Dma` — DMA-mapping integration

### Locking and concurrency

- **Per-`request_queue` `q->queue_lock`** (raw_spinlock; legacy; rare in mq path)
- **Per-hctx `hctx->lock`** (spinlock): protects hctx state transitions
- **Per-hctx tag bitmap**: lock-free atomic ops on bitmap word + per-tag waitqueue
- **Per-hctx srcu**: in-flight request protection during scheduler/device transitions

TLA+ models cover blk_mq_tag_alloc + request_state_machine.

### Error handling

- `Err(EAGAIN)` — out of tags; caller retries
- `Err(ETIMEDOUT)` — request timed out
- `Err(EIO)` — driver returned I/O error
- `Err(EREMOTEIO)` — driver-side specific error
- `Err(ENOMEM)` — request alloc failed
- `Err(EOPNOTSUPP)` — driver doesn't support op

### Out of Scope

- Legacy single-queue layer (removed)
- Per-driver block driver (cross-ref `drivers/00-overview.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| blk-mq core (alloc + dispatch + complete) | `block/blk-mq.c`, `block/blk-mq.h` |
| Per-hctx tag bitmap | `block/blk-mq-tag.c` |
| I/O-scheduler integration | `block/blk-mq-sched.c` |
| sysfs + debugfs | `block/blk-mq-sysfs.c`, `block/blk-mq-debugfs.c` |
| Per-CPU → hctx mapping | `block/blk-mq-cpumap.c` |
| DMA-mapping for blk-mq | `block/blk-mq-dma.c` |
| Public types | `include/linux/blk-mq.h`, `include/linux/blkdev.h`, `include/linux/blk_types.h` |

### compatibility contract

### `struct request` layout

`include/linux/blk-mq.h` defines `struct request`. First-cache-line + commonly-accessed fields layout-equivalent. Fields: `q`, `mq_ctx`, `mq_hctx`, `cmd_flags`, `rq_flags`, `tag`, `internal_tag`, `start_time_ns`, `io_start_time_ns`, `wbt_flags`, `nr_phys_segments`, `nr_integrity_segments`, `__data_len`, `__sector`, `bio`, `biotail`, `rq_disk`, `part`, `start_time`, `state`, `ref` (refcount; saturating), `timeout`, `errors`, `csd`, `end_io`, `end_io_data`, `cmd`, `cmd_size`, `cmd_type`, `__deadline`, `_lookups`, `tagset_data`, `prio`, `cmd_flags`.

### `struct blk_mq_hw_ctx` layout

Per-hctx state. Layout-equivalent. Fields: `ctxs`, `tags`, `sched_tags`, `queued`, `run`, `numa_node`, `tx_queue`, `dispatch`, `state`, `lock`, `cpumask`, `queue`, `tagset_data`, `next_cpu`, `next_cpu_batch`, `flags`, `srcu`, `nr_active`, `wait`, `delay_work`, `cpuhp_*`.

### blk-mq sysfs

`/sys/block/<dev>/mq/<hctx_id>/*`: `cpu_list`, `nr_tags`, `nr_reserved_tags`, `tags`, `sched_tags`, `active`, `dispatch`, `flags`, `state`, ... Format-identical.

### blk-mq debugfs

`/sys/kernel/debug/block/<dev>/hctx<id>/*`: `dispatch`, `flags`, `state`, `cpu_list`, `tags`, `tags_bitmap`, `sched_tags`, `sched_tags_bitmap`, `run`, `active`. Format-identical.

### Userspace tooling

`fio` (with `ioengine=io_uring` or `libaio`), `dd`, `iostat`, `iotop` all work unmodified.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Tag bitmap atomic set/clear | `kani::proofs::block::mq::tag_bitmap_safety` |
| Per-CPU ctx → hctx mapping | `kani::proofs::block::mq::cpumap_safety` |
| Request state-transition atomic CAS | `kani::proofs::block::mq::state_transition_safety` |
| Timeout list manipulation | `kani::proofs::block::mq::timeout_list_safety` |
| Scatter-list build | `kani::proofs::block::mq::sg_build_safety` |

### Layer 2: TLA+ models

- `models/block/blk_mq_tag_alloc.tla` (mandatory per `block/00-overview.md` Layer 2) — proves tag allocation under concurrent submission across hctxs never double-allocates. Owned here.
- `models/block/request_state_machine.tla` (mandatory per `block/00-overview.md` Layer 2) — proves request state transitions (IDLE → SUBMITTED → IN_FLIGHT → DONE / TIMEOUT / FAILED) honor invariants under race between dispatch, timeout, completion. Owned here.
- `models/block/elevator.tla` (mandatory) — proves I/O scheduler ordering invariants for each of mq-deadline / kyber / bfq.

### Layer 3: Kani harnesses for data-structure invariants (mandatory per `block/00-overview.md`)

| Data structure | Invariant | Harness |
|---|---|---|
| Tag bitmap (per hctx) | Each tag is in exactly one of: free, allocated, reserved | `kani::proofs::block::mq::tag_invariants` |
| Request linked list | No request appears twice in any list (free, dispatch, timeout, complete) | `kani::proofs::block::mq::request_list_invariants` |
| Per-CPU ctx | Each CPU has exactly one ctx; each ctx maps to exactly one hctx | `kani::proofs::block::mq::ctx_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Request state machine** via Verus — proves: under any sequence of (submit, complete, timeout, retry), state transitions follow the upstream-defined diagram.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | request->ref + per-hctx refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | request allocated via per-hctx tagset slab cache; per-driver type-tagged | § Mandatory |
| **MEMORY_SANITIZE** | freed request objects zeroed (especially relevant for command + integrity-data fields) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **UDEREF**: never accepts user pointers directly; bio is the consumer-side abstraction (cross-ref `block/bio.md`)
- **SIZE_OVERFLOW**: tag-index + sector arithmetic uses checked operators
- **CONSTIFY**: `blk_mq_ops` vtable per-driver `static const`

### Row-2 / GR-RBAC integration

blk-mq is below the LSM-hook layer; LSM hooks fire at fs/syscall level (cross-ref `fs/00-overview.md`).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

