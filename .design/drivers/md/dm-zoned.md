# Tier-3: drivers/md/dm-zoned-target.c — host-managed SMR (Shingled-Magnetic-Recording) zoned-block-device target

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-zoned-target.c
  - drivers/md/dm-zoned-metadata.c
  - drivers/md/dm-zoned-reclaim.c
  - drivers/md/dm-zoned.h
-->

## Summary

dm-zoned exposes host-managed SMR (Shingled-Magnetic-Recording) zoned block-devices as a regular random-write target — SMR drives have sequential-write-only zones that must be reset to write again; dm-zoned maps random-write LBAs onto sequential zones via per-chunk indirection (chunk → zone mapping) + per-zone state (conventional / sequential / random). Background reclaim moves data between zones to free old sequential zones. Used to allow legacy filesystems (ext4, etc.) on SMR drives that otherwise require zoned-aware FS (zonefs, F2FS-zoned, btrfs-zoned).

This Tier-3 covers `dm-zoned-target.c` (~1160) + `dm-zoned-metadata.c` (~3001) + `dm-zoned-reclaim.c` (~640).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dmz_target` | per-target state | `drivers::md::zoned::DmzTarget` |
| `struct dmz_metadata` | per-target persistent metadata | `DmzMetadata` |
| `struct dmz_dev` | per-zoned-bdev descriptor | `DmzDev` |
| `struct dmz_zone` | per-zone state | `DmzZone` |
| `struct dmz_mblock` | per-metadata-block in cache | `DmzMblock` |
| `struct dm_zoned_reclaim` | per-target reclaim worker | `DmZonedReclaim` |
| `dmz_ctr(target, argc, argv)` | target-create | `Dmz::ctr` |
| `dmz_dtr(target)` | dtor | `Dmz::dtr` |
| `dmz_map(target, bio)` | per-bio dispatch | `Dmz::map_bio` |
| `dmz_handle_read(...)` / `_write(...)` / `_discard(...)` | per-op dispatch | `Dmz::handle_read` / `_write` / `_discard` |
| `dmz_get_chunk_zone(...)` | per-chunk-idx zone lookup | `Dmz::get_chunk_zone` |
| `dmz_get_zone_for_reclaim(...)` | reclaim-source selection | `Dmz::get_zone_for_reclaim` |
| `dmz_load_metadata(...)` (dm-zoned-metadata.c) | metadata-load | `DmzMetadata::load` |
| `dmz_flush_metadata(...)` | metadata-flush | `DmzMetadata::flush` |
| `dmz_get_mblock(...)` / `_put_mblock(...)` | per-block cache | `DmzMetadata::get_mblock` / `_put_mblock` |
| `dmz_reclaim_run(...)` (dm-zoned-reclaim.c) | reclaim worker thread | `DmZonedReclaim::run` |
| `dmz_alloc_zone(...)` / `_free_zone(...)` | per-zone alloc | `Dmz::alloc_zone` / `_free_zone` |
| `dmz_reset_zone(...)` | SMR reset-zone command | `DmzDev::reset_zone` |

## Compatibility contract

REQ-1: SMR zone types:
- Conventional: random R/W (small region; ~10% of drive).
- Sequential-write-required (SWR): writes must go in increasing LBA; reset-zone to write again.
- Sequential-write-preferred (SWP): random-write allowed but not optimized.

REQ-2: Per-target `dmz_target`:
- `ti` (back-ref).
- `dev` (DmzDev for underlying zoned bdev).
- `metadata` (KArc<DmzMetadata>).
- `reclaim` (KArc<DmZonedReclaim>).
- `bio_pool` (mempool).
- `chunk_size` (sectors; matches zone-size).
- `dmz_wq` (workqueue).
- `flags` (DMZ_*: BAD_DEVICE, ON_DEMAND_FLUSH, etc.).

REQ-3: Per-zone `dmz_zone`:
- `dev` (back-ref to DmzDev).
- `id` (zone index).
- `flags` (DMZ_RND, DMZ_SEQ, DMZ_RECLAIM, etc.).
- `chunk` (currently mapped chunk-id).
- `wp_block` (write pointer; sequential-zone next-write block).
- `link` (per-state list-node).
- `mblk_node` (metadata-block link).
- `bzone` (per-buffer zone link for write-buffering).

REQ-4: Per-chunk indirection:
- Per-target: chunk_size sectors per chunk.
- Per-chunk → primary zone mapping in metadata.
- Per-chunk → optional buffer zone (random-write zone for write-buffer).
- Reclaim moves chunks between zones to free sequential zones.

REQ-5: Per-bio write flow:
1. chunk := bio.bi_iter.bi_sector / chunk_size.
2. zone := dmz_get_chunk_zone(chunk).
3. If zone is sequential + write-pointer mismatch:
   - Write goes to buffer zone (random) instead.
   - Buffer zone tracks dirty-blocks.
4. Submit bio to zone.

REQ-6: Per-bio read flow:
1. chunk := bio.bi_iter.bi_sector / chunk_size.
2. zone := dmz_get_chunk_zone(chunk).
3. If buffer zone has dirty data: serve from buffer.
4. Else: serve from primary zone.

REQ-7: Background reclaim:
- Identifies "best" sequential zone to reclaim (most invalid blocks).
- Copies valid blocks to a fresh sequential zone.
- Reset old zone via SMR-zone-reset command.
- Free zone returned to alloc pool.

REQ-8: Per-target metadata:
- Persistent on dedicated metadata zones.
- Layout:
  - Superblock (per-zone state, version, magic).
  - Bitmap zones (dirty-block-bitmap per chunk).
  - Mapping zones (chunk → zone).
  - Random zones reserved for buffer-zone allocation.

REQ-9: dmz_mblock cache:
- Per-block 4KiB metadata cache with LRU eviction.
- Per-mblock dirty bit; flushed at metadata-checkpoint.

REQ-10: SMR zone-reset:
- BLKRESETZONE ioctl issued to underlying device.
- After reset: write-pointer back to zone start.

REQ-11: Failure handling:
- Per-zone error count; mark zone bad on threshold.
- Per-target metadata corruption: target marked DMZ_BAD_DEVICE; subsequent IO returns -EIO.

REQ-12: Modern alternatives:
- For new deployments, native zoned-FS (F2FS-zoned, btrfs-zoned) preferred.
- dm-zoned for legacy ext4/xfs on SMR.

## Acceptance Criteria

- [ ] AC-1: Create dm-zoned over /dev/sde (SMR drive); /dev/dm-N reports as random-write block device.
- [ ] AC-2: ext4 mkfs + mount on /dev/dm-N: filesystem operations succeed despite SMR backing.
- [ ] AC-3: Background reclaim: sustained writes; reclaim runs; old zones reset; new zones allocated.
- [ ] AC-4: Crash recovery: kill -9 mid-write; reboot; metadata-replay restores consistent state.
- [ ] AC-5: Per-zone error: simulate zone-fault; per-zone error counter increments; zone marked bad after threshold.
- [ ] AC-6: 1TB SMR drive stress: heavy write workload; sustained throughput within 50% of native bandwidth.
- [ ] AC-7: Buffer zone fallback: random write on sequential zone goes to buffer; subsequent read serves correct data.
- [ ] AC-8: dm-zoned xfstests pass.
- [ ] AC-9: Mode change: read-only mount works; write-mount enables reclaim worker.
- [ ] AC-10: Performance: reclaim does not overlap with foreground IO causing pathological latency.

## Architecture

`DmzTarget`:

```
struct DmzTarget {
  ti: KArc<DmTarget>,
  dev: KArc<DmzDev>,
  metadata: KArc<DmzMetadata>,
  reclaim: KArc<DmZonedReclaim>,
  bio_pool: KArc<MempoolBio>,
  chunk_size: u32,
  zone_size: u64,                              // sectors
  zone_size_shift: u32,
  zone_nr_blocks: u64,
  zone_nr_blocks_shift: u32,
  zone_bits_per_mblk: u32,
  flags: u32,
  dmz_wq: KArc<Workqueue>,
  ...
}

struct DmzDev {
  bdev: KArc<BlockDevice>,
  nr_zones: u32,
  zones: KVec<DmzZone>,
  flags: u32,
  zone_bitmap_size: u64,
}

struct DmzZone {
  dev: KWeak<DmzDev>,
  id: u32,
  flags: u32,                                  // DMZ_RND, _SEQ, _OFFLINE, _READONLY
  chunk: u32,                                   // mapped chunk-id, or DMZ_MAP_UNMAPPED
  wp_block: u64,                                // write pointer
  weight: u32,                                  // valid blocks
  bzone: Option<KArc<DmzZone>>,                 // buffer zone, if any
  link: ListNode,
  mblk_node: HlistNode,
  ...
}

struct DmzMetadata {
  dev: KArc<DmzDev>,
  superblock: KBox<DmzSuperblock>,
  mblk_lru_lock: SpinLock<()>,
  mblk_lru: ListHead,
  nr_mblks: u32,
  ...
}
```

`Dmz::map_bio(target, bio)`:
1. dmz := target.private.
2. chunk := bio.bi_iter.bi_sector >> dmz.chunk_size_shift.
3. lock dmz.metadata.lock.
4. zone := dmz_get_chunk_zone(metadata, chunk, op).
5. Switch bio.bi_op:
   - READ: handle_read(zone, bio).
   - WRITE: handle_write(zone, bio).
   - DISCARD: handle_discard(zone, bio).
6. unlock.

`Dmz::handle_write(zone, bio)`:
1. If zone.flags & DMZ_SEQ:
   - Check bio.bi_iter.bi_sector aligns with zone.wp_block.
   - If mismatch: route to buffer zone (random).
2. Submit bio to zone.dev.bdev at zone.id * zone_size + offset.
3. On completion: update zone.wp_block += bio_sectors; metadata mark-dirty.

`DmZonedReclaim::run(reclaim)` (worker):
1. Loop:
   - Wait for trigger condition (free zones < threshold).
   - target_zone := dmz_get_zone_for_reclaim (most invalid blocks).
   - For each valid chunk in target_zone:
     - Allocate dest zone.
     - Copy chunk via dm_kcopyd.
     - Update metadata mapping.
   - Reset target_zone via BLKRESETZONE.
   - Free zone to pool.
2. Sleep until next trigger.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `zone_id_bounded` | OOB | per-zone idx < dev.nr_zones; defense against OOB walk. |
| `chunk_idx_bounded` | OOB | per-chunk idx < dmz.nr_chunks. |
| `wp_block_monotonic` | INVARIANT | per-sequential-zone wp_block monotonically advances except on reset. |
| `metadata_commit_atomic` | INVARIANT | metadata commit transactional. |
| `zone_state_valid` | INVARIANT | per-zone flags ∈ valid bitmap (DMZ_RND, _SEQ, _OFFLINE, _READONLY, etc.). |

### Layer 2: TLA+

`drivers/md/zoned_lifecycle.tla`:
- Per-zone state ∈ {Free, Allocated, Active(wp), FullActive, Reclaiming, Reset, Failed}.
- Properties:
  - `safety_no_random_write_on_seq_zone` — sequential zone writes only at wp_block (or via buffer-zone redirect).
  - `safety_reset_clears_wp` — Reset transitions wp_block → 0.
  - `liveness_full_zone_eventually_reclaimed` — assuming reclaim runs, FullActive eventually Free.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Dmz::map_bio` post: bio remapped to underlying zone OR redirected to buffer zone | `Dmz::map_bio` |
| `DmZonedReclaim::run` post: target zone reclaimed; valid chunks copied; metadata updated | `DmZonedReclaim::run` |
| Per-chunk mapping consistent with metadata | invariants on metadata commit |
| `DmzDev::reset_zone` post: zone wp_block == 0; flags allow new write | `DmzDev::reset_zone` |

### Layer 4: Verus/Creusot functional

`Per-chunk read/write: data through dm-zoned matches single-disk semantics modulo sequential-write redirection` semantic equivalence: per-bio data observable through dm-zoned matches what bare-metal random-write would be (including buffer-zone redirect).

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-zoned-specific reinforcement:

- **Per-zone wp_block validation** — defense against write before wp causing SMR write-error.
- **Buffer zone fallback** — defense against random-write on SMR causing IO failure.
- **Metadata transactional commit** — defense against partial-state on crash.
- **Per-zone error_count threshold** — defense against transient-error flapping.
- **DMZ_BAD_DEVICE terminal state** — defense against post-fail IO causing further corruption.
- **Background reclaim rate-limited** — defense against reclaim hogging IO bandwidth.
- **dm_kcopyd async-copy bounded** — defense against unbounded outstanding copies.
- **Per-zone refcount during reclaim** — defense against reclaim-during-IO UAF.
- **BLKRESETZONE wait-for-quiesce** — defense against reset-during-active-write.
- **Per-mblock LRU bounded** — defense against unbounded metadata-cache growth.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- Native zoned-FS (zonefs, F2FS-zoned, btrfs-zoned; covered in respective FS Tier-3s)
- ZBC/ZAC SCSI command-set (covered in `drivers/scsi/sd_zbc.md` future Tier-3)
- NVMe-ZNS (covered in `drivers/nvme/host-pci.md` Tier-3)
- Implementation code
