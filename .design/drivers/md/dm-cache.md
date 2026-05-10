# Tier-3: drivers/md/dm-cache-target.c — block cache target (fast/slow tier + writeback policy + per-block migration)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-cache-target.c
  - drivers/md/dm-cache-metadata.c
  - drivers/md/dm-cache-policy-smq.c
  - drivers/md/dm-cache-policy.c
  - drivers/md/dm-cache-policy-internal.h
-->

## Summary

dm-cache is the device-mapper target that combines a small fast block-device (cache, e.g., NVMe SSD) with a large slow block-device (origin, e.g., HDD) — promoting frequently-accessed origin blocks to cache + writing back dirty cache blocks. Per-block 4-KiB-default tracked in cache via a hashmap (cache-block-index → origin-block-index, dirty-bit, hit-count). Per-policy plug-in (SMQ = Stochastic Multi-Queue is default) decides per-bio whether to promote / evict / passthrough. Useful for desktop SSD-as-HDD-cache (dm-bcache equivalent) + cloud-tiering. dm-cache-metadata.c persists policy state across reboots.

This Tier-3 covers `dm-cache-target.c` (~3612 lines) + `dm-cache-metadata.c` (~1827) + `dm-cache-policy-smq.c` (~1960).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cache` | per-target cache state | `drivers::md::cache::Cache` |
| `struct cache_features` | per-cache config flags | `CacheFeatures` |
| `struct dm_cache_metadata` | per-cache persistent state | `DmCacheMetadata` |
| `struct dm_cache_policy` | policy vtable | `DmCachePolicy` |
| `struct smq_policy` | SMQ implementation state | `SmqPolicy` |
| `struct entry` (smq) | per-block hit-count entry | `SmqEntry` |
| `cache_ctr(target, argc, argv)` | target-create | `Cache::ctr` |
| `cache_dtr(target)` | destroy | `Cache::dtr` |
| `cache_map(target, bio)` | bio dispatch | `Cache::map_bio` |
| `cache_endio(target, bio, err)` | bio completion | `Cache::endio` |
| `cache_status(...)` | dmsetup status | `Cache::status` |
| `cache_message(...)` | DM_TARGET_MSG | `Cache::message` |
| `process_bio(cache, bio)` | per-bio lookup + dispatch | `Cache::process_bio` |
| `migration_init(...)` / `migration_success(...)` / `migration_failure(...)` | per-block migration | `CacheMigration::init` / `_success` / `_failure` |
| `do_writeback(cache)` | dirty-block writeback worker | `Cache::do_writeback` |
| `policy_create(name, ...)` | per-policy registry lookup + create | `DmCachePolicy::create` |
| `policy_lookup(p, oblock, &cblock, ...)` | per-policy block-lookup | `DmCachePolicy::lookup` |
| `policy_get_background_work(p, ...)` | per-policy promote/demote/writeback decision | `DmCachePolicy::get_background_work` |
| `policy_get_hint(p, cblock, &hint)` | per-policy persistence hint | `DmCachePolicy::get_hint` |
| `dm_cache_metadata_open(...)` (dm-cache-metadata.c) | open persistent state | `DmCacheMetadata::open` |
| `dm_cache_commit(metadata)` | commit metadata | `DmCacheMetadata::commit` |
| `smq_lookup(p, oblock)` (dm-cache-policy-smq.c) | SMQ-specific lookup | `SmqPolicy::lookup` |
| `smq_set_clear_dirty(...)` | SMQ dirty-bit update | `SmqPolicy::set_clear_dirty` |

## Compatibility contract

REQ-1: Per-target `cache`:
- `cache_dev` (fast block-device).
- `origin_dev` (slow block-device).
- `metadata_dev` (separate or carved from cache-dev).
- `cache_size` (sectors).
- `origin_blocks` (cache block count for origin).
- `cache_blocks` (cache block count for cache-dev).
- `block_size` (sectors per block; default 64 sectors = 32KB; configurable up to 1MB).
- `policy` (KArc<DmCachePolicy>).
- `migration_threshold` (max in-flight migrations).
- `mode` (DM_CACHE_FEATURE_WRITEBACK / _PASSTHROUGH / _WRITETHROUGH).
- `cmd` (KArc<DmCacheMetadata>; persistent state).
- `bio_prison` (per-block-cell locking).
- `worker` (background worker thread).
- `quiescing` (suspend/resume coordination).
- `migration_count` (active migrations).
- `mg_in_flight` / `mg_done` (per-status counts).

REQ-2: Per-policy vtable:
- `lookup(p, oblock, &result)`: returns CacheLookupResult { cblock, dirty, valid }.
- `set_clear_dirty(p, oblock, dirty)`.
- `get_background_work(p, &op)`: returns next promote/demote/writeback op.
- `complete_background_work(p, op, success)`.
- `get_hint(p, cblock, &hint)`: per-policy persistence hint.
- `set_hint(p, cblock, hint)`: restore hint at mount.
- `tick(p)`: per-tick policy update.
- `lock_metadata(p)` / `_unlock_metadata(p)`.

REQ-3: Per-bio dispatch (`cache_map` → `process_bio`):
1. Acquire bio_prison cell for origin-block.
2. policy.lookup(origin_block, &result):
   - HIT(cblock, dirty, valid): bio remap to cache_dev at cblock.
   - MISS_BACKGROUND_PROMOTE: promote-to-cache (write-back); meanwhile remap to origin.
   - MISS_NO_PROMOTE: passthrough (no caching this access).
   - INVALIDATE: cache invalidated; force-fetch from origin.
3. Submit to underlying.

REQ-4: Per-bio writeback semantics (DM_CACHE_FEATURE_WRITEBACK):
- Writes go to cache; dirty-bit set.
- Reads from cache when present; from origin otherwise.
- Background worker writes back dirty blocks to origin.

REQ-5: Per-bio passthrough (DM_CACHE_FEATURE_PASSTHROUGH):
- All bios go directly to origin.
- Cache stale; userspace-controlled refresh required.
- Used for safe-degrade / failover.

REQ-6: Per-bio writethrough (DM_CACHE_FEATURE_WRITETHROUGH):
- Writes go to BOTH cache and origin.
- Reads from cache when present; from origin otherwise.
- No dirty blocks; safe even if cache lost.

REQ-7: SMQ (Stochastic Multi-Queue) policy:
- Per-block hit-count tracked; promotion-policy randomized for fairness.
- Multi-level LRU (clean / dirty / hot).
- Per-block per-queue level adjustment via Stochastic decay.
- Defense against pathological access pattern starving cache.

REQ-8: Per-cache migration:
- Migration-init: alloc cache-block; pin; submit copy bio (origin → cache for promote, cache → origin for writeback).
- Migration-success: install in policy; release.
- Migration-failure: rollback; un-pin.

REQ-9: Per-policy persistent hints:
- Per-cblock hint stored in metadata at suspend.
- At resume: hints restored to policy.
- Used by SMQ for per-block-hit-count persistence.

REQ-10: Per-cache metadata layout:
- Mapping table: cblock → oblock + dirty-bit.
- Per-cblock hint area.
- Discard bitmap (track unmapped origin blocks).
- Persistent via dm-bufio + transactional commit.

REQ-11: Discard support:
- Discard origin block: invalidate cache entry + propagate to origin.
- Discard bitmap tracks valid-data presence.

REQ-12: Per-suspend dirty flush:
- DM_CACHE_FEATURE_WRITEBACK: suspend triggers do_writeback for all dirty blocks.
- Otherwise: simple quiesce.

## Acceptance Criteria

- [ ] AC-1: dmsetup create test_cache: cache-dev (10GB SSD) + origin-dev (1TB HDD); cache mode writeback.
- [ ] AC-2: Sequential read 100GB through cache: hot-data promoted; subsequent re-read served from cache.
- [ ] AC-3: Write-back: writes go to cache; dirty-bit set; bg writeback flushes to origin.
- [ ] AC-4: Suspend in writeback mode: dirty blocks flushed before suspend completes.
- [ ] AC-5: Crash recovery: kill -9 mid-IO; reboot; metadata commit boundary preserved; no data loss for committed-state.
- [ ] AC-6: Mode switch: dmsetup message switch writeback → writethrough; pending writes drained.
- [ ] AC-7: Discard propagation: discard 1GB on cached volume; cache + origin both freed.
- [ ] AC-8: 1M-block cache: stress test 1M-cblock cache; SMQ policy prevents pathological starvation.
- [ ] AC-9: Replacement policy: feed fixed-pattern; verify SMQ replaces oldest-cold first.
- [ ] AC-10: dm-cache xfstests pass.

## Architecture

`Cache`:

```
struct Cache {
  cache_dev: KArc<DmDev>,
  origin_dev: KArc<DmDev>,
  metadata_dev: KArc<DmDev>,
  cmd: KArc<DmCacheMetadata>,
  policy: KArc<dyn DmCachePolicy>,
  features: CacheFeatures,
  block_size: u32,
  cache_size: u64,                              // sectors
  origin_blocks: u64,
  cache_blocks: u64,
  bio_prison: KArc<BioPrison>,
  worker_lock: SpinLock<()>,
  worker: Worker,
  ...
}

struct CacheMigration {
  list: ListNode,
  cache: KArc<Cache>,
  oblock: u64,
  cblock: u64,
  op: BackgroundOp,
  bio: Option<KArc<Bio>>,
  status: AtomicI32,
}

struct CacheFeatures {
  io_mode: CacheIoMode,                         // Writeback, Writethrough, Passthrough
  metadata_version: u32,
  discard_passdown: bool,
}
```

`Cache::map_bio(target, bio)`:
1. cache := target.private.
2. oblock := bio.bi_iter.bi_sector / block_size.
3. cell := bio_prison_isolate(cache, oblock, bio).
4. If !cell: defer; return DM_MAPIO_SUBMITTED.
5. Schedule process_bio on worker.

`Cache::process_bio(cache, bio, oblock, cell)`:
1. result := cache.policy.lookup(oblock).
2. Switch result:
   - HIT(cblock): bio remapped to cache_dev at cblock; submit; return.
   - PROMOTE(cblock): start migration origin→cache; meanwhile bio remapped to origin.
   - PASSTHROUGH: bio remapped to origin; submit.
   - INVALIDATE: cache invalidated; bio remapped to origin.

`Cache::do_writeback(cache)` (worker):
1. policy.get_background_work(&op).
2. If op.kind == WRITEBACK:
   - alloc CacheMigration.
   - Submit copy: cache_dev[cblock] → origin_dev[oblock].
   - On complete: policy.complete_background_work(op, success).
   - clear dirty-bit.

`SmqPolicy::lookup(p, oblock)`:
1. hash := hash(oblock).
2. entry := hash-bucket-walk for matching oblock.
3. If found: increment hit-count; return HIT(cblock, dirty, valid).
4. Else: stochastic decision based on cache fill + access pattern → MISS_PROMOTE or MISS_NO_PROMOTE.

`DmCacheMetadata::commit(metadata)`:
1. dm-bufio flush all dirty buffers.
2. Persistent superblock written.
3. Transaction sequence advanced.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cblock_idx_bounded` | OOB | per-cblock < cache.cache_blocks; defense against policy returning OOB index. |
| `bio_prison_no_uaf` | UAF | per-cell ref-count under bio_prison.lock. |
| `migration_state_valid` | INVARIANT | per-migration state ∈ {Init, InFlight, Done, Failed}. |
| `dirty_bit_persistent` | INVARIANT | per-block dirty-bit persisted before bio_endio for writeback writes. |
| `metadata_commit_atomic` | INVARIANT | metadata commit transactional via dm-bufio; partial-commit not visible. |

### Layer 2: TLA+

`drivers/md/cache_promotion.tla`:
- Per-block state ∈ {NotCached, Migrating, Cached(clean/dirty), Invalidated}.
- Properties:
  - `safety_no_dirty_writeback_during_origin_read` — mutual exclusion via bio_prison.
  - `safety_writeback_clears_dirty` — successful writeback transitions to Clean.
  - `liveness_dirty_eventually_writeback` — assuming bg-worker runs, every Dirty eventually Clean.

`drivers/md/cache_metadata_commit.tla`:
- Commit state: Dirty, Committing, Clean.
- Properties:
  - `safety_recovered_state_at_last_commit` — post-crash state = state at last commit.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cache::map_bio` post: bio queued for processing; cell acquired | `Cache::map_bio` |
| `Cache::process_bio` post: bio remapped per policy result | `Cache::process_bio` |
| `Cache::do_writeback` post: migrations queued; dirty-bits cleared on success | `Cache::do_writeback` |
| Per-policy.lookup returns valid cblock or PROMOTE-decision | invariants on policy result |
| Per-cache.cmd.commit persists policy hints | invariants on commit |

### Layer 4: Verus/Creusot functional

`Cached read returns same data as origin read for that block` semantic equivalence: per-block per-bio read result matches what origin would have returned at last clean state.

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-cache-specific reinforcement:

- **Per-block bio_prison cell** — defense against concurrent migration + read.
- **Metadata commit before bio_endio** for writeback dirty — defense against post-crash data loss.
- **Per-mode writeback dirty-flush at suspend** — defense against losing dirty state.
- **policy lookup returns bounded cblock** — defense against malicious policy plug-in returning OOB.
- **Per-cache.cmd transactional commit** — defense against partial-state on crash.
- **DM_CACHE_FEATURE_PASSTHROUGH safe-default** during failure-modes — defense against cache-stale access.
- **Per-migration in-flight count cap** — defense against unbounded background-work consuming all IO bandwidth.
- **Per-block discard propagation** — defense against orphan-data after origin discard.
- **dm-cache-metadata superblock checksum** — defense against torn-write at superblock during crash.
- **Per-policy hint persistence** — defense against SMQ resetting per-block hit-count on every reboot.
- **Per-bio_prison cell ref-count** — defense against double-release on requeue.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-bufio (covered separately)
- Per-policy plug-in details beyond SMQ (out-of-scope)
- bcache (different stackable cache; out-of-scope)
- Implementation code
