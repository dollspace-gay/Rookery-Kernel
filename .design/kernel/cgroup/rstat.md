# Tier-3: kernel/cgroup/rstat — recursive statistics framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/cgroup/rstat.c
  - include/linux/cgroup.h
-->

## Summary
Tier-3 design for the cgroup recursive statistics (rstat) framework — the kernel's flush-on-read aggregation system that lets per-cgroup stats efficiently support per-CPU updates while presenting consistent recursive (cgroup-tree-aggregated) views on read. Without rstat, every per-cpu counter update would need to take a per-cgroup atomic; with rstat, updates are lockless per-cpu, and aggregation only happens on read (when userspace queries `memory.stat` / `cpu.stat` / `io.stat` / etc.).

The trick: each per-cpu counter increment also pushes the (cgroup, cpu) pair onto a per-cpu "updated cgroups" list (lockless). On read, kernel walks this list for the queried cgroup + ancestors, flushing per-cpu values into the global aggregate. Subsequent reads from the same cgroup hit cached aggregate.

Owns the mandatory `models/cgroup/rstat_consistency.tla` TLA+ model.

Sub-tier-3 of `kernel/cgroup/00-overview.md`. Pairs with `kernel/cgroup/cgroup-core.md` (consumer of per-cgroup state); per-controller Tier-3s (memcontrol, blkcg, cpuctl) consume rstat for stat aggregation.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| rstat framework: per-cpu update lists, flush-on-read tree-walk, per-controller css_rstat_flush | `kernel/cgroup/rstat.c` |
| Public API | `include/linux/cgroup.h` (cgroup_rstat_*) |

## Compatibility contract

### Per-cpu rstat data

Per-cgroup per-cpu state:
```c
struct cgroup_rstat_cpu {
    struct u64_stats_sync bsync;
    struct cgroup_base_stat bstat;
    struct cgroup_base_stat last_bstat;
    struct cgroup_base_stat subtree_bstat;
    struct cgroup_base_stat last_subtree_bstat;
    struct cgroup *updated_children;
    struct cgroup *updated_next;
};

struct cgroup_rstat_base_cpu {
    struct cgroup_base_stat bstat;
    struct cgroup_base_stat last_bstat;
    struct cgroup_base_stat subtree_bstat;
    struct cgroup_base_stat last_subtree_bstat;
};
```

Layout-equivalent.

### `struct cgroup_base_stat` (CPU-time accounting)

```c
struct cgroup_base_stat {
    struct task_cputime cputime;
    u64 forceidle_sum;
    u64 ntime;
};
```

Per-cgroup base CPU stats: user / system / sum_exec_runtime + force-idle sum + nice-time. Per-controller can extend via `css_rstat_flush`.

### `cgroup_rstat_updated(cgrp, cpu)` — mark cgroup-cpu dirty

Called when a per-cpu counter is updated. Push `(cgrp, cpu)` onto per-cpu updated list:
```c
void cgroup_rstat_updated(struct cgroup *cgrp, int cpu);
```

Lockless via per-cpu data + atomic-cmpxchg insertion. Identical algorithm.

### `cgroup_rstat_flush(cgrp)` — flush + aggregate

Called on read of cgroup's stat file (e.g., `memory.stat`):
1. Acquire per-cgroup `cgroup_rstat_lock` (rwsem)
2. Walk per-cpu updated list for cgrp + descendants
3. Per-(cgrp, cpu) on list: read per-cpu delta, add to global aggregate, then call per-controller `css_rstat_flush(css, cpu)` to flush per-controller per-cpu state
4. Drain updated list (atomic clear after flush)
5. Release lock

Identical algorithm.

### `css_rstat_flush(css, cpu)` per-controller callback

Per-controller's `cgroup_subsys.css_rstat_flush` callback:
```c
void (*css_rstat_flush)(struct cgroup_subsys_state *css, int cpu);
```

Per-controller reads per-cpu delta from controller-private state (e.g., `mem_cgroup_per_node->lruvec_stats`), adds to aggregate, then propagates upward to parent css via per-controller hook.

### Per-cpu lockless updated-list

Per-cpu `updated_children` linked-list head; per-cgroup `updated_next` is the per-cpu next-pointer. On `cgroup_rstat_updated`:
1. Atomic compare-exchange to insert at head
2. If already on list (next != NULL), no-op (deduplication)

Identical lockless insertion.

### Tree-walk via `for_each_cpu_for_rstat`

Per-flush, kernel walks all CPUs for rstat-updated cgroups under target. Walk is O(updated_cgroups × cpus); drained per-flush.

### Per-cgroup `rstat_lock` rwsem

Held during flush (writer = flush; readers don't exist — flush is the only path that touches the global aggregate).

### `cgroup_base_stat_add` arithmetic

Per-base-stat accumulator. Uses per-`u64_stats_sync` for 32-bit-safe 64-bit reads (lockless writers via seqcount-style update).

### `cgroup_rstat_exit` per-cgroup teardown

Called from cgroup destruction path. Drains rstat list + frees per-cpu data.

### Per-cpu base + per-controller layered state

Each cgroup has:
- Per-cpu `cgroup_rstat_base_cpu` (bstat — CPU-time)
- Per-cpu `cgroup_rstat_cpu` (extended state for per-controller flush)
- Per-controller per-cpu state lives in controller-private structs (e.g., `mem_cgroup_per_node`)

## Requirements

- REQ-1: Per-cgroup per-cpu state: `cgroup_rstat_cpu` + `cgroup_rstat_base_cpu` first-cache-line layout-equivalent.
- REQ-2: `cgroup_rstat_updated(cgrp, cpu)` lockless per-cpu list-insert via atomic-cmpxchg; deduplication on already-on-list.
- REQ-3: `cgroup_rstat_flush(cgrp)` walks per-cpu updated list + flushes + drains; per-controller `css_rstat_flush` invoked per-(css, cpu).
- REQ-4: Per-controller `css_rstat_flush` callback: per-css per-cpu delta-read + aggregate + propagate-to-parent.
- REQ-5: `cgroup_base_stat_add` arithmetic with `u64_stats_sync` for 32-bit-safe 64-bit reads (lockless writers).
- REQ-6: Per-cgroup `cgroup_rstat_lock` rwsem held during flush; concurrent updaters lockless.
- REQ-7: Tree-walk `for_each_cpu_for_rstat` traverses all rstat-updated descendants under target cgroup.
- REQ-8: `cgroup_rstat_exit` drains + frees per-cpu data on cgroup destroy.
- REQ-9: Aggregate consistency: post-flush, global aggregate = Σ all per-cpu deltas (modulo concurrent updates after flush).
- REQ-10: TLA+ model `models/cgroup/rstat_consistency.tla` (mandatory per `kernel/cgroup/00-overview.md` Layer 2) — proves: rstat tree-walk produces consistent aggregate values; no lost increments; flush-on-read safe under concurrent writers.
- REQ-11: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct cgroup_rstat_cpu` + `cgroup_rstat_base_cpu` byte-identical first cache-line. (covers REQ-1)
- [ ] AC-2: Update test: synthetic test increments per-cpu counter via `cgroup_rstat_updated`; per-cpu updated list contains the (cgrp, cpu) entry. (covers REQ-2)
- [ ] AC-3: Flush test: trigger flush via `cat /sys/fs/cgroup/<cg>/cpu.stat`; per-cpu values aggregate into global; updated list drained. (covers REQ-3)
- [ ] AC-4: Per-controller flush test: memcontrol `css_rstat_flush` invoked during flush; per-cgroup `memory.stat` shows aggregated deltas. (covers REQ-4)
- [ ] AC-5: 32-bit-safe read test: on 32-bit hardware (or via 32-bit emulation), 64-bit `cputime` values read consistently under concurrent writes (no torn read). (covers REQ-5)
- [ ] AC-6: Concurrent update test: 1000 threads incrementing per-cpu counters concurrently with 100 readers calling flush; aggregated result equals total increment count (modulo race window for very recent updates). (covers REQ-6, REQ-9)
- [ ] AC-7: Tree-walk test: cgroup hierarchy with 100 leaves under root; flush root aggregates all leaf updates. (covers REQ-7)
- [ ] AC-8: Destroy test: cgroup with active rstat updates is destroyed; rstat data freed cleanly; no UAF. (covers REQ-8)
- [ ] AC-9: Aggregation consistency theorem: post-flush, global aggregate = Σ per-cpu deltas (verified via Verus proof). (covers REQ-9)
- [ ] AC-10: TLA+ `models/cgroup/rstat_consistency.tla` proves: tree-walk is consistent; no lost increments; flush-on-read safe under concurrent writers. (covers REQ-10)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-11)

## Architecture

### Rust module organization

- `kernel::cgroup::rstat::PerCpu` — `struct cgroup_rstat_cpu` + `cgroup_rstat_base_cpu`
- `kernel::cgroup::rstat::Updated` — per-cpu updated-list mgmt
- `kernel::cgroup::rstat::Flush` — `cgroup_rstat_flush` tree-walk
- `kernel::cgroup::rstat::CssFlush` — per-controller `css_rstat_flush` dispatch
- `kernel::cgroup::rstat::BaseStat` — `cgroup_base_stat_add` arithmetic + u64_stats_sync
- `kernel::cgroup::rstat::Lock` — per-cgroup `cgroup_rstat_lock`
- `kernel::cgroup::rstat::Exit` — `cgroup_rstat_exit`

### Locking and concurrency

- **Per-cgroup `cgroup_rstat_lock`** (rwsem): held during flush
- **Per-cpu updated-list**: lockless via atomic-cmpxchg insertion
- **`u64_stats_sync`**: per-cpu seqcount-style for 32-bit-safe 64-bit reads
- **RCU**: per-cgroup state read RCU-side from controller hot-path

### Error handling

- No userspace-visible errors; all internal state mgmt
- `cgroup_rstat_flush` may detect rstat-disabled cgroups (early-return)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-cpu list-insert (atomic-cmpxchg correctness; no lost insertions under concurrent updaters) | `kani::proofs::kernel::cgroup::rstat::insert_safety` |
| `cgroup_rstat_flush` tree-walk under rstat_lock (no use-after-free of just-destroyed cgroups) | `kani::proofs::kernel::cgroup::rstat::flush_safety` |
| Per-controller `css_rstat_flush` invocation (RCU-side per-css ref-acquire) | `kani::proofs::kernel::cgroup::rstat::css_flush_safety` |
| `u64_stats_sync` 32-bit-safe write/read pairing | `kani::proofs::kernel::cgroup::rstat::u64sync_safety` |

### Layer 2: TLA+ models

- `models/cgroup/rstat_consistency.tla` (mandatory per `kernel/cgroup/00-overview.md`) — proves rstat aggregation soundness. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-cpu updated-list | acyclic; every entry's cgroup pointer non-NULL; deduplication invariant (each cgroup appears at most once per-cpu) | `kani::proofs::kernel::cgroup::rstat::list_invariants` |
| Per-cgroup aggregate | post-flush bstat = previous-bstat + Σ per-cpu deltas | `kani::proofs::kernel::cgroup::rstat::aggregate_invariants` |

### Layer 4: Functional correctness (declared in `kernel/cgroup/00-overview.md` Layer 4)

- **rstat aggregate-consistency theorem** via TLA+ refinement — proves: post-flush global aggregate equals Σ all per-cpu updates (modulo concurrent updates that happen after flush start); no lost increments. Owned here.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **AUTOSLAB** | per-cgroup_rstat_cpu + per-cgroup_rstat_base_cpu per-cpu allocations | § Mandatory |
| **SIZE_OVERFLOW** | per-counter accumulator arithmetic uses checked operators (CVE class: counter-overflow leading to incorrect aggregation) | § Mandatory |
| **MEMORY_SANITIZE** | freed per-cpu rstat data cleared (carry per-cgroup accounting) | § Default-on configurable off |

### Row-1 features consumed by this component

- **AUTOSLAB, SIZE_OVERFLOW, MEMORY_SANITIZE**: see above
- **CONSTIFY**: per-controller `css_rstat_flush` fn-ptr in `cgroup_subsys` `static const`
- **REFCOUNT**: per-cgroup ref (cross-ref `cgroup-core.md`)

### Row-2 / GR-RBAC integration

- LSM hook: read-only via cgroup-file kernfs; permission gate via `security_inode_permission` already standard.
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — rstat readers (`cpu.stat`, `memory.stat`, `io.stat`, `cgroup.stat`) `seq_printf` per-cgroup aggregated values into a user-readable buffer; whitelist the per-controller output schema so a kernel-write primitive cannot widen the seq_file extent to drag adjacent per-cpu rstat state across the user boundary.
- **PAX_KERNEXEC** — `css_rstat_flush` per-controller dispatch (`cgroup_subsys.css_rstat_flush`) is indirect-called from `cgroup_rstat_flush_locked`; require W^X on the subsys ops table and refuse rwx for the per-controller flush trampoline.
- **PAX_RANDKSTACK** — `cgroup_rstat_flush` runs from `read(2)` with attacker-driven cgroup depth; randomized kstack offset disrupts ROP through the `css_rstat_updated_list` walk.
- **PAX_REFCOUNT** — `cgroup->rstat_flush_next` chain count and per-CPU `cgroup_rstat_cpu->updated_next` lifetime are protected by the `cgroup_rstat_lock` + per-cpu `updated_lock`; harden the css refcount surrounding `cgroup_rstat_exit` with saturating overflow trap.
- **PAX_MEMORY_SANITIZE** — `cgroup_rstat_exit` must zero the per-cpu `cgroup_rstat_cpu` array (`bsync`, `bstat`, `subtree_bstat`, `last_bstat`) before kfree; defense against post-free residual exposing CPU-time / IO-byte telemetry to a fresh cgroup at the same per-cpu offset.
- **PAX_UDEREF** — rstat `seq_show` callbacks never touch userspace directly (seq_file mediates), but the seq_buf write barrier must keep SMAP/PAN engaged through the controller's `printf` formatting.
- **PAX_RAP/kCFI** — per-controller `css_rstat_flush` indirect call must verify kCFI tag at the cgroup-core dispatch site; defense against a corrupted subsys vector routing flush through an attacker-chosen stub.
- **GRKERNSEC_HIDESYM** — `cgroup.stat` and per-controller stat readers must expose aggregated counters only; refuse to leak per-cpu `cgroup_rstat_cpu *` pointers or `updated_next` linked-list addresses through any error path.
- **GRKERNSEC_DMESG** — `WARN_ON` on `cgroup_rstat_lock` contention or `cgroup_rstat_flush_irqsafe` reentry must not splat raw cgroup / per-cpu pointers into dmesg readable by non-CAP_SYSLOG.
- **Per-cgroup stats CAP_SYS_NICE gate** — read of high-resolution per-cgroup stats (sub-second `cpu.stat` deltas, ns-resolution `io.stat`) should require `ns_capable(user_ns, CAP_SYS_NICE)`; defense against side-channel attacks using rstat granularity to fingerprint sibling-cgroup workloads.
- **Lock-free per-cpu update discipline** — `cgroup_rstat_updated` uses per-cpu `updated_lock` + lockless `updated_next` chain; any extension that introduces cross-cpu lock acquisition during the update fast path is forbidden under grsec mode (would create a per-cgroup soft-DoS primitive).
- **Per-cgroup flush ceiling** — `cgroup_rstat_flush` walk must be bounded by `cgroup_max_depth` so an attacker creating a deeply-nested hierarchy cannot stall a stat read under `cgroup_rstat_lock`.
- **`rstat_flush_irqsafe` irq-disable window minimal** — the irq-disabled section in `cgroup_rstat_flush_locked` must release the lock periodically (every N cgroups walked); defense against attacker driving long flushes that starve unrelated IRQ handlers.
- **Rationale** — rstat is the kernel's central per-cgroup statistics aggregator; high-resolution access from an attacker grants both a side-channel (fingerprint sibling workloads) and a soft-DoS (stall flushes under contended cgroup_rstat_lock). The grsec regime forces an attacker to defeat W^X on subsys flush + kCFI on flush dispatch + CAP_SYS_NICE on high-res read + memory sanitize on css_free + bounded-walk depth before reaching either a fingerprint or a stall primitive.

## Open Questions

(none — rstat framework exhaustively specified by upstream)

## Out of Scope

- Per-controller `css_rstat_flush` implementations (cross-ref `mm/memcontrol.md`, `block/blk-cgroup.md`, etc.)
- 32-bit-only paths
- Implementation code
