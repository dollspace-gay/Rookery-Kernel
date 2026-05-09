---
title: "Tier-3: mm/reclaim — vmscan, LRU, MGLRU, shrinker, kswapd, oom_kill"
tags: ["design-doc", "tier-3", "mm"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for memory reclaim: the path that frees memory under pressure. Owns vmscan (the LRU-aware page-eviction algorithm — both classic and MGLRU variants), the per-memcg per-node LRU lists, the shrinker framework (fs/slab/etc. caches register reclaim callbacks), kswapd (the per-NUMA-node async reclaim thread), and oom_kill (last-resort task-killing when reclaim cannot recover memory).

Reclaim runs alongside every allocation: most allocs check watermarks first; if low, they wake kswapd; if critically low, they direct-reclaim themselves. This Tier-3 is the keystone of memory-pressure handling — bugs here mean the system either gets killed prematurely (false OOM) or allocates indefinitely past safe limits (system hang).

### Requirements

- REQ-1: vmscan algorithm (classic LRU + MGLRU when CONFIG_LRU_GEN=y) preserves upstream's eviction policy: page-aging via reference + dirty bits, two-list (active + inactive) per memcg per node, refault detection via shadow entries.
- REQ-2: kswapd per-node thread: woken when zone free count drops below `low watermark`; sleeps when `high watermark` reached; respects `nr_zone_inactive_anon` etc. balancing.
- REQ-3: Direct reclaim: invoked by allocation path when watermark + kswapd insufficient; respects GFP flags (NOFAIL forces, NORETRY backs off, NOIO bounds reclaim depth).
- REQ-4: Shrinker framework: registered shrinkers called per `nr_to_scan` requests; reclaim priority adjusted per recent slab-pressure history; `count_objects` + `scan_objects` callbacks invoked per upstream contract.
- REQ-5: OOM killer: `oom_score_adj` honored (-1000 = OOM-immune); selected victim's `signal->oom_score_adj` checked; SIGKILL delivered via `mark_oom_victim`; `oom_reaper` thread reclaims the victim's memory asynchronously.
- REQ-6: Memcg integration: per-memcg LRU lists; `memory.high` triggers active reclaim; `memory.max` triggers OOM within the memcg before global OOM.
- REQ-7: NUMA-balancing: when CONFIG_NUMA_BALANCING=y, page-migration triggered on remote-node access; identical heuristics + sysctl knobs (`numa_balancing_*`).
- REQ-8: Compaction integration: under fragmentation pressure, defrag pages are migrated to consolidate free runs (cross-ref `mm/migration-compaction.md`).
- REQ-9: `/proc/meminfo` + `/proc/vmstat` content format-identical for equivalent kernel state.
- REQ-10: OOM panic mode (`/proc/sys/vm/panic_on_oom=1`): reaches kernel panic instead of selecting a victim; matches upstream.
- REQ-11: Layer-2 TLA+ model `models/mm/lru.tla` proves LRU list ordering under concurrent reclaim + add + touch operations, including memcg hierarchical accounting.
- REQ-12: Layer-3 invariant harness for LRU + memcg consistency MANDATORY per `mm/00-overview.md` REQ-14 / `00-overview.md` D4.
- REQ-13: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: A memory-pressure benchmark (allocate N% of system memory, then access in stride) produces upstream-comparable eviction decisions; `/proc/vmstat pgscan_*` counters within ±10%. (covers REQ-1)
- [ ] AC-2: kswapd test: under sustained pressure, kswapd is the primary reclaim path (not direct reclaim); verifiable via `/proc/<kswapd-pid>/stat` CPU time vs. `pgsteal_direct`. (covers REQ-2)
- [ ] AC-3: A `__GFP_NOFAIL` allocation succeeds even under extreme pressure; `__GFP_NORETRY` backs off after a single attempt. (covers REQ-3)
- [ ] AC-4: A shrinker test (e.g., dcache shrinker via `echo 2 > /proc/sys/vm/drop_caches`) frees the expected fraction of registered shrinkable objects. (covers REQ-4)
- [ ] AC-5: An OOM scenario (`stress --vm 8 --vm-bytes 8G` on 4G system) selects the same victim as upstream on identical config; oom_score_adj=-1000 process is exempt. (covers REQ-5)
- [ ] AC-6: cgroup v2 memcg test: memory.high triggers reclaim; memory.max triggers in-memcg OOM before global OOM. (covers REQ-6)
- [ ] AC-7: NUMA-balancing test on multi-socket: pages migrate toward accessing CPU per upstream policy; `/proc/<pid>/numa_maps` shows migration. (covers REQ-7)
- [ ] AC-8: Fragmentation test: high-order allocation under low-order-only fragmentation triggers compaction; success rate matches upstream. (covers REQ-8)
- [ ] AC-9: `cat /proc/meminfo` reclaim fields byte-identical (modulo dynamic counts) on equivalent boot. (covers REQ-9)
- [ ] AC-10: Boot with `panic_on_oom=1`; trigger OOM; kernel panics with the upstream-format panic message. (covers REQ-10)
- [ ] AC-11: `make tla` passes `models/mm/lru.tla`. (covers REQ-11)
- [ ] AC-12: `make verify` passes `kani::proofs::mm::reclaim::lru_invariants`. (covers REQ-12)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-13)

### Architecture

### Rust module organization

- `kernel::mm::reclaim::vmscan` — vmscan core (eviction algorithm)
- `kernel::mm::reclaim::mglru` — MGLRU when CONFIG_LRU_GEN=y
- `kernel::mm::reclaim::lru` — per-memcg per-node LRU lists
- `kernel::mm::reclaim::pcp_lru` — per-CPU LRU pageset (mm/swap.c)
- `kernel::mm::reclaim::shrinker::Shrinker` — shrinker registration + dispatch
- `kernel::mm::reclaim::kswapd` — per-node async reclaim thread
- `kernel::mm::reclaim::oom::OomKiller` — task-selection algorithm + SIGKILL delivery
- `kernel::mm::reclaim::oom::Reaper` — async memory-reclaim thread for OOM victims
- `kernel::mm::reclaim::watermarks` — per-zone watermark management
- `kernel::mm::reclaim::page_writeback` — dirty-page writeback orchestration
- `kernel::mm::reclaim::show_mem` — memory-state dump (sysrq, dmesg)

### Key data structures

- `lruvec` — per-memcg per-node LRU set (active/inactive × anon/file × evictable/unevictable)
- `lru_gen_struct` (when CONFIG_LRU_GEN=y) — per-memcg generation counters + per-folio age tags
- `shrink_control` — request packet for shrinker callbacks
- `oom_control` — OOM-decision context (scope: memcg vs. global, memory pressure level)

### Locking and concurrency

- **lru_lock** (per-memcg per-node spinlock): protects LRU list manipulation
- **lru_gen_struct lock** (per-memcg): protects MGLRU generation transitions
- **shrinker_rwsem**: rwsem; readers walk shrinker list during reclaim; writers register/unregister
- **kswapd waitqueue**: per-node `pgdat->kswapd_wait`
- **oom_lock**: global mutex; serializes OOM victim selection
- **oom_reaper_lock**: protects the OOM-reaper task list

TLA+ model `models/mm/lru.tla` (mandatory per `mm/00-overview.md` Layer 2) covers the multi-CPU + multi-memcg LRU concurrency; this Tier-3 owns the model.

### Error handling

Reclaim is a "best effort" path; most callers don't propagate `Result`:
- Watermark recovery: returns `unsigned long pages_reclaimed`; 0 = nothing reclaimed
- Direct reclaim: returns same; caller decides whether to retry alloc
- Shrinker: each shrinker callback returns `unsigned long objects_freed`
- OOM killer: returns `bool oom_killed`; if false, caller (allocator) returns ENOMEM

### Out of Scope

- Swap subsystem (cross-ref `mm/swap.md`)
- Zswap / zsmalloc (cross-ref `mm/zsmalloc.md`)
- Per-FS shrinker implementations (cross-ref individual fs/* docs)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Reclaim core (vmscan, MGLRU) | `mm/vmscan.c` |
| Per-CPU LRU mgmt | `mm/swap.c` |
| Shrinker framework | `mm/shrinker.c`, `include/linux/shrinker.h` |
| OOM killer | `mm/oom_kill.c`, `include/linux/oom.h` |
| Memory-state dump (sysrq + dmesg) | `mm/show_mem.c` |
| Per-LRU helper | `mm/list_lru.c`, `include/linux/list_lru.h` |
| Page writeback orchestration (dirty-page eviction) | `mm/page-writeback.c` |
| Page migration (anti-fragmentation, NUMA-balancing) | `mm/migrate.c` |
| LRU-state inline helpers | `include/linux/mm_inline.h` |
| Swap interface (cross-ref `mm/swap.md`) | `include/linux/swap.h` |
| Zone watermarks | `include/linux/mmzone.h` |

### compatibility contract

### `/proc` surfaces

- `/proc/meminfo` reclaim-relevant fields: `Active`, `Inactive`, `Active(anon)`, `Inactive(anon)`, `Active(file)`, `Inactive(file)`, `Unevictable`, `Mlocked`, `KReclaimable`, `Slab`, `SReclaimable`, `SUnreclaim`. Format-identical.
- `/proc/vmstat` per-zone counters: `pgactivate`, `pgdeactivate`, `pgmajfault`, `pgrotated`, `pgrefill`, `pgscan_kswapd`, `pgscan_direct`, `pgsteal_kswapd`, `pgsteal_direct`, `oom_kill`, `compact_*`. Format-identical.
- `/proc/<pid>/oom_score`, `/proc/<pid>/oom_score_adj`, `/proc/<pid>/oom_adj` — per-task OOM accounting. Numeric values + range identical.
- `/proc/sys/vm/{swappiness, vfs_cache_pressure, watermark_*, dirty_*, oom_dump_tasks, panic_on_oom, overcommit_*}` — every documented sysctl preserved.

### `/sys/kernel/mm/lru_gen/*` (MGLRU)

When CONFIG_LRU_GEN=y, exposes:
- `enabled` — bitmask: bit 0 enable; bit 1 mm_walk; bit 2 non-leaf-young
- `min_ttl_ms` — eviction protection window
- `seq` (per-cgroup) — generation counters

Identical to upstream.

### Shrinker registration

`register_shrinker` / `unregister_shrinker` ABI preserved. Per-shrinker `nr_to_scan` callback semantics + reclaim-priority bucketing identical so out-of-tree shrinkers (e.g., zfs ARC, fuse cache) work unchanged.

### OOM-kill scoring

`oom_badness(task) = oom_score_adj_normalize + rss_pages` (per upstream's algorithm). The selected victim is the highest-scoring eligible task. Identical so existing systemd `OOMScoreAdjust=` settings + container memory-pressure handlers reproduce upstream's victim selection.

### vmstat counters

Every counter exposed via `/proc/vmstat` preserved with identical semantics.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| LRU list manipulation under lru_lock | `kani::proofs::mm::reclaim::lru_link_safety` |
| Per-CPU LRU pageset push/pop | `kani::proofs::mm::reclaim::pcp_lru_safety` |
| MGLRU generation transitions | `kani::proofs::mm::reclaim::mglru_gen_safety` |
| Shrinker callback dispatch | `kani::proofs::mm::reclaim::shrinker_dispatch_safety` |
| OOM victim list manipulation | `kani::proofs::mm::reclaim::oom_victims_safety` |

### Layer 2: TLA+ models

- `models/mm/lru.tla` (mandatory per `mm/00-overview.md` Layer 2) — proves LRU list ordering under concurrent reclaim + add + touch operations, including memcg hierarchical accounting. Owned here.
- `models/mm/oom_serialization.tla` (NEW) — proves OOM victim selection serializes globally so two simultaneous OOM events don't kill two victims when one suffices.
- `models/mm/kswapd_balance.tla` (NEW) — proves kswapd's wake/sleep coordination with allocator threads: every alloc that drops below watermark wakes kswapd; kswapd doesn't oversleep past wake events.

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY per `00-overview.md` D4)

| Data structure | Invariant | Harness |
|---|---|---|
| LRU lists (per memcg per node) | "Every page on an LRU list has the corresponding LRU page-flag set" + "Memcg accounting is consistent: sum-of-charges == sum-of-folio-references" | `kani::proofs::mm::reclaim::lru_invariants` |
| MGLRU generations | "Every folio's generation tag is in [0, max_gen]"; "Generation counters monotonically increase" | `kani::proofs::mm::reclaim::mglru_gen_invariants` |
| Shrinker list | "Each registered shrinker appears exactly once; iteration order matches registration order modulo seqcount" | `kani::proofs::mm::reclaim::shrinker_list_invariants` |

### Layer 4: Functional correctness (opt-in)

- **MGLRU generation accounting** via Verus — proves: under any sequence of LRU operations (add, touch, evict), the generation counts remain consistent with the ground-truth folio set. (Higher difficulty; tracked but not v0.)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

(none directly; reclaim consumes hardening from page-allocator + slab)

### Row-1 features consumed by this component

- **MEMORY_SANITIZE**: reclaimed folios go through page-allocator's free path; zeroed if `vm.zero_on_free=1`
- **REFCOUNT**: lruvec counters use `Refcount`-equivalent saturating arithmetic
- **SIZE_OVERFLOW**: page-count + memory-size arithmetic uses checked operators
- **CONSTIFY**: shrinker default-priority table is `static const`

### Row-2 / GR-RBAC integration

OOM-kill is LSM-relevant: `security_task_kill(victim, SIGKILL, ...)` invoked. GR-RBAC (per its loaded policy) can deny OOM-kill for protected processes — adds defense-in-depth above the existing oom_score_adj=-1000 mechanism.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

