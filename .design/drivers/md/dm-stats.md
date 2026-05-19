# Tier-3: drivers/md/dm-stats.c — per-target IO statistics (per-region histogram + latency tracking + per-cgroup accounting)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-stats.c
  - drivers/md/dm-stats.h
-->

## Summary

dm-stats is the per-dm-target IO statistics framework — userspace dmsetup messages create per-target stats-areas (sub-regions of the LBA space) tracking per-area: IO count, sectors-read/written, ticks-spent, in-flight count, optional per-bin latency histogram, optional per-cgroup accounting. Per-bio dispatch increments per-affected-area counters; userspace polls via dmsetup or /proc to extract metrics. Used for: storage performance monitoring, hot-spot identification, per-application IO accounting, regression-testing.

This Tier-3 covers `drivers/md/dm-stats.c` (~1264 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dm_stats` | per-target stats container | `drivers::md::stats::DmStats` |
| `struct dm_stat` | per-stats-area | `DmStat` |
| `struct dm_stat_shared` | per-area shared aggregate | `DmStatShared` |
| `struct dm_stat_percpu` | per-CPU per-area counters | `DmStatPercpu` |
| `dm_stats_init(stats)` | per-target init | `DmStats::init` |
| `dm_stats_cleanup(stats)` | per-target cleanup | `DmStats::cleanup` |
| `dm_stats_account_io(stats, bi_op, bi_sector, bi_sectors, &end_io_state)` | per-bio accounting | `DmStats::account_io` |
| `dm_stats_message(...)` | dmsetup message dispatcher | `DmStats::message` |
| `__dm_stats_create(...)` | create stats-area | `DmStats::create_area` |
| `__dm_stats_delete(...)` | delete stats-area | `DmStats::delete_area` |
| `__dm_stats_clear(...)` | reset counters | `DmStats::clear_area` |
| `dm_stats_print(...)` | per-area human-readable dump | `DmStats::print` |
| `dm_kvzalloc_percpu(...)` | per-CPU memory alloc | shared |
| `dm_stats_account_start(stats, &state)` / `_end(stats, &state)` | per-bio start/end pair | `DmStats::account_start` / `_end` |

## Compatibility contract

REQ-1: Per-target `dm_stats`:
- `mutex` (per-stats lock).
- `list` (list of dm_stat areas).
- `last` (last-accessed pointer for hot-cache).
- `precise_timestamps` (bool: ns vs jiffies).
- `bdi_id` (block-device backing-dev-info id; for per-bdi accounting).

REQ-2: Per-stats-area `dm_stat`:
- `id` (unique per-target).
- `start` / `end` (sector range).
- `step` (sectors per region).
- `n_entries` (number of regions).
- `program_id` (user-supplied tag).
- `aux_data` (user-supplied aux string).
- `precise_timestamps` (per-area override).
- `histogram_alloc_mask` (which bins allocated).
- `n_histogram_entries` (bin count).
- `histogram_boundaries` (KVec<u64>; per-bin upper-edge in ns).
- `stat_shared` (KVec<DmStatShared>; per-region shared aggregate).
- `stat_percpu` (PerCpu<KVec<DmStatPercpu>>; per-CPU counters).

REQ-3: Per-region `dm_stat_shared`:
- `tmp` (cumulative aggregate).
- `in_flight` (ATOMIC count).
- `last_update_ns` (last bio start).

REQ-4: Per-region per-CPU `dm_stat_percpu`:
- `sectors[NR_OPS]` (per-op sector count).
- `ios[NR_OPS]` (per-op IO count).
- `merges[NR_OPS]` (merge count).
- `ticks[NR_OPS]` (ns of accumulated time).
- `time_in_queue` (queue-depth-weighted time).
- `histogram_alloc[]` (per-bin alloc count).
- `histogram_ticks[]` (per-bin tick total).

REQ-5: Per-bio accounting flow (`dm_stats_account_io`):
1. start := now.
2. For each affected stats-area:
   - For each region in [bi_sector, bi_sector + bi_sectors) within [start, end] step:
     - per-CPU: ios[op]++; sectors[op] += affected_sectors.
     - shared.in_flight++.
     - record start-time for end_io.
3. Return end_io_state for paired call.

REQ-6: Per-bio end accounting (`dm_stats_account_end`):
1. duration := now - start.
2. For each affected region:
   - per-CPU: ticks[op] += duration; time_in_queue += duration * in_flight.
   - shared.in_flight--.
   - histogram-bin per latency.

REQ-7: dmsetup message protocol:
- "@stats_create <range> <step> [precise_timestamps] [program_id] [aux_data]": create.
- "@stats_delete <id>": delete.
- "@stats_clear <id>": reset counters.
- "@stats_list [program_id]": list active areas.
- "@stats_print <id> [start_line] [n_lines]": per-area dump.
- "@stats_set_aux <id> <aux>": update aux.

REQ-8: Per-region step:
- Sectors per region; smaller = finer-granularity stats.
- Larger = lower memory overhead.
- Default 0 = single-region covering whole [start, end].

REQ-9: Latency histogram:
- Configurable per-bin upper-edges (in ns).
- Per-CPU per-bin counter + tick total.
- Used to identify per-region tail-latency.

REQ-10: precise_timestamps:
- false: jiffies (cheap; ~1ms granularity).
- true: ktime (ns-resolution; more expensive).

REQ-11: Per-bdi accounting:
- bdi_id ties stats to backing-dev-info.
- Optional: aggregates across multiple dm-targets.

REQ-12: Per-cgroup accounting:
- /sys/fs/cgroup/io.dm-stats integration (when cgroup-v2-io enabled).
- Per-(cgroup, region) counters.

## Acceptance Criteria

- [ ] AC-1: dmsetup message stats_create on linear target: stats-area created.
- [ ] AC-2: 100K bio writes; subsequent stats_print shows correct bytes_written + ios.
- [ ] AC-3: Histogram: configure bins {1us, 10us, 100us, 1ms}; latency observed; per-bin counts populate.
- [ ] AC-4: Multi-region: stats with step=1MB across 1GB target; per-region IO distribution visible.
- [ ] AC-5: precise_timestamps: ns-resolution latency vs jiffies; histogram differs accordingly.
- [ ] AC-6: stats_clear: reset counters; subsequent IO from zero.
- [ ] AC-7: stats_delete: remove area; memory freed.
- [ ] AC-8: per-CPU scaling: 32-CPU host with concurrent IO; per-CPU counters aggregated correctly.
- [ ] AC-9: dm-stats overhead < 5% on hot-IO benchmark.
- [ ] AC-10: dmsetup test suite for dm-stats passes.

## Architecture

`DmStats`:

```
struct DmStats {
  mutex: Mutex<()>,
  list: ListHead,                              // dm_stat areas
  last: AtomicPtr<DmStat>,                     // hot-cache
  precise_timestamps: bool,
  bdi_id: u32,
}

struct DmStat {
  list: ListNode,
  id: u32,
  start: u64,                                  // sector
  end: u64,                                    // sector
  step: u32,
  n_entries: u64,
  precise_timestamps: bool,
  program_id: KString,
  aux_data: KString,
  histogram_alloc_mask: u64,
  n_histogram_entries: u32,
  histogram_boundaries: KVec<u64>,
  stat_shared: KVec<DmStatShared>,
  stat_percpu: PerCpu<KVec<DmStatPercpu>>,
}

struct DmStatShared {
  tmp: DmStatPercpu,                            // aggregated cumulative
  in_flight: AtomicI64,
  last_update_ns: u64,
}

struct DmStatPercpu {
  sectors: [AtomicU64; NR_OPS],                // per-op
  ios: [AtomicU64; NR_OPS],
  merges: [AtomicU64; NR_OPS],
  ticks: [AtomicU64; NR_OPS],
  time_in_queue: AtomicU64,
  histogram_alloc: KVec<AtomicU64>,
  histogram_ticks: KVec<AtomicU64>,
}
```

`DmStats::account_io(stats, bi_op, bi_sector, bi_sectors, &state)`:
1. Iterate areas:
   - If [bi_sector, bi_sector+bi_sectors) intersects area.[start, end]:
     - For each region in intersection:
       - cpu := smp_processor_id().
       - per_cpu := stats.stat_percpu[cpu][region].
       - per_cpu.ios[bi_op]++; per_cpu.sectors[bi_op] += affected_sectors.
       - shared.in_flight++.
2. Record state := { now_ns, bi_op, n_regions, region_indices, ... }.

`DmStats::account_end(stats, &state)`:
1. duration := now_ns - state.start_ns.
2. For each region in state:
   - per_cpu := stats.stat_percpu[cpu][region].
   - per_cpu.ticks[op] += duration.
   - bin := find_histogram_bin(stats, duration).
   - per_cpu.histogram_alloc[bin]++; per_cpu.histogram_ticks[bin] += duration.
3. shared.in_flight--.

`DmStats::message(stats, argc, argv)`:
1. Switch argv[0]:
   - "@stats_create": __dm_stats_create.
   - "@stats_delete": __dm_stats_delete.
   - "@stats_clear": __dm_stats_clear.
   - "@stats_list": list active areas.
   - "@stats_print": format counters to userspace buffer.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `region_idx_bounded` | OOB | per-region idx < n_entries; defense against OOB stat-percpu access. |
| `in_flight_no_underflow` | INVARIANT | shared.in_flight ≥ 0; defense against double-decrement. |
| `histogram_bin_idx_bounded` | OOB | per-bin idx < n_histogram_entries. |
| `area_id_unique` | INVARIANT | per-DmStat.id unique within DmStats.list. |
| `state_pair_balanced` | INVARIANT | every account_start paired with account_end. |

### Layer 2: TLA+

`drivers/md/dm_stats_accounting.tla`:
- Per-region state: counters.
- Per-bio start/end paired.
- Properties:
  - `safety_per_region_atomic_increment` — per-CPU counters via atomic-add; defense against torn count.
  - `safety_in_flight_balanced` — shared.in_flight tracks (started - ended) bios.
  - `liveness_eventual_account_end` — every account_start eventually account_end (assuming bio completes).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DmStats::account_io` post: per-region counters incremented; in_flight incremented | `DmStats::account_io` |
| `DmStats::account_end` post: ticks recorded; in_flight decremented | `DmStats::account_end` |
| `DmStats::__create_area` post: area added to list; n_entries computed; per-CPU memory allocated | `DmStats::__create_area` |
| `DmStats::__delete_area` post: area removed from list; per-CPU memory freed | `DmStats::__delete_area` |

### Layer 4: Verus/Creusot functional

`Per-IO: account_io + account_end pair → per-region counters reflect IO size + duration` semantic equivalence: per-counter the value matches sum of all attributable IOs for that (region, op).

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-stats-specific reinforcement:

- **Per-CPU counters atomic** — defense against torn count under concurrent access.
- **Per-state account_start/end paired** — defense against in_flight imbalance.
- **histogram-bin idx bounded** — defense against OOB on malformed boundaries.
- **dmsetup message privileged** — defense against unauthorized stats config.
- **Per-area memory bounded by n_entries cap** — defense against unbounded stats memory growth.
- **precise_timestamps opt-in** — defense against per-bio ktime() overhead in non-precise mode.
- **Per-CPU stat_percpu lazy-alloc** — defense against per-CPU bloat for inactive areas.
- **Per-area mutex held during create/delete** — defense against concurrent mutation.
- **last hot-cache atomic** — defense against torn pointer dereference.
- **stats_clear atomicity** — defense against partial-clear visible to readers.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `dm_stats_print` histogram and per-area latency arrays are emitted through `dm_stats_message`'s bounded `result` buffer with explicit length-clamping; the per-CPU `stat_percpu` vector is whitelisted in its own slab cache so copy_to_user cannot straddle adjacent objects.
- **PAX_KERNEXEC** — `dm_stats` helper tables (`message_stats_*` dispatch) live in `__ro_after_init`; histogram-bucket arrays are allocated with `__GFP_ZERO` from a non-executable slab.
- **PAX_RANDKSTACK** — `dm_stats_account_io` runs in the IO completion path, which hits the randomized kstack offset before recording per-area latency.
- **PAX_REFCOUNT** — `dm_stat->in_flight` and per-area `n_entries` are refcount_t / atomic_t with overflow saturation; a malicious user issuing a flood of `@stats_create` cannot wrap and free the table.
- **PAX_MEMORY_SANITIZE** — histogram buckets and per-CPU stat arrays are zeroed on `@stats_delete` before slab return so previously-recorded IO patterns don't leak between tenants.
- **PAX_UDEREF** — `@stats_create` / `@stats_print` argument parsing reads strings through `copy_from_user` into a kmalloc'd `argv` scratch, never through user-pointer chase.
- **PAX_RAP / kCFI** — dispatch from `dm_stats_message` into the per-command handler uses a typed switch, not an indirect call; the few indirect calls (e.g. `dm_io` completion) carry RAP signatures.
- **GRKERNSEC_HIDESYM** — `/sys/fs/dm-*` and `dmsetup status` output for stats objects has kernel pointers scrubbed under `kptr_restrict ≥ 2`.
- **GRKERNSEC_DMESG** — `dm_stats` does not log per-IO information; admin-only events (create/delete/clear) go at `KERN_DEBUG` with ratelimit.
- **RBAC on /sys/fs/dm** — stats objects are addressable only via the dm device-mapper ioctl plane; `@stats_*` messages require `CAP_SYS_ADMIN` in the device's userns and the device's dm-uevent group, so unprivileged userland cannot enumerate per-IO histograms.
- **Histogram PAX_USERCOPY** — the per-bucket counts emitted by `@stats_print_clear` are emitted via `DMEMIT` against a fixed `result` buffer; bucket-array tail-padding is never copied to userspace.
- **Per-area allocation cap** — `MAX_STATS_AREAS` (default `dm_stats_max_areas`) bounds the total number of areas a single dm device can host; `@stats_create` returns `-EINVAL` once the cap is hit, preventing memory-exhaustion attacks.
- **stats_clear epoch** — `stats_clear` increments a per-area sequence counter so concurrent `stats_print` sees either pre-clear or post-clear state, never a torn mix.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- Generic block-layer stats (per-bdev IO accounting; covered in `block/00-overview.md`)
- /proc/diskstats (covered separately)
- ftrace block tracepoints (covered in `kernel/trace/00-overview.md`)
- Implementation code
