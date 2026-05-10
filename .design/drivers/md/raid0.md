# Tier-3: drivers/md/raid0.c — RAID-0 (linear striping: per-bio sector remap to per-disk via stripe-offset)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/00-overview.md
upstream-paths:
  - drivers/md/raid0.c
  - drivers/md/raid0.h
-->

## Summary

RAID-0 is pure striping: data spread across N disks as fixed-size chunks; no parity / no mirroring. Per-bio: split into per-chunk segments; per-segment route to corresponding disk at stripe-offset. Throughput scales with N for sequential workloads; no redundancy (any disk failure loses all data). Linux md raid0 also supports zoning with non-uniform disk sizes via "zones" — disks grouped by smallest-disk-size; inner zones use only larger disks. Smallest md driver (~841 lines).

This Tier-3 covers `drivers/md/raid0.c` (~841 lines) + `raid0.h` (~33).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct r0conf` | per-array config | `drivers::md::raid0::R0Conf` |
| `struct strip_zone` | per-zone descriptor | `StripZone` |
| `raid0_make_request(mddev, bio)` | per-bio dispatch | `Raid0::make_request` |
| `raid0_run(mddev)` | array startup | `Raid0::run` |
| `raid0_stop(mddev)` | array shutdown | `Raid0::stop` |
| `raid0_status(seq_file, mddev)` | proc display | `Raid0::status` |
| `raid0_size(mddev, sectors, raid_disks)` | array size in sectors | `Raid0::array_size` |
| `find_zone(conf, sector)` | sector → zone lookup | `R0Conf::find_zone` |
| `map_sector(mddev, zone, sector, &offset)` | sector → disk + offset | `R0Conf::map_sector` |
| `dump_zones(mddev)` | debug dump | `Raid0::dump_zones` |
| `is_io_in_chunk_boundary(...)` | chunk-boundary check | `Raid0::in_chunk_boundary` |
| `raid0_make_request_split(mddev, bio)` | bio split for cross-chunk | `Raid0::split_bio` |
| `create_strip_zones(mddev, &priv_conf)` | zone-list creation | `R0Conf::create_strip_zones` |

## Compatibility contract

REQ-1: Per-array `r0conf`:
- `mddev` (back-ref).
- `strip_zone` (KVec<StripZone>).
- `nr_strip_zones` (count).
- `devlist` (per-zone disk-list flat array).
- `layout` (raid0 layout: original / Alternative).

REQ-2: Per-zone `strip_zone`:
- `zone_end` (end-sector of this zone in raid).
- `dev_start` (start-sector on each disk of this zone).
- `nb_dev` (count of disks in this zone).

REQ-3: Per-bio dispatch (`raid0_make_request`):
1. zone := find_zone(conf, bio.bi_iter.bi_sector).
2. If bio crosses zone boundary or chunk boundary: split.
3. (sector, offset) := map_sector(mddev, zone, bio.bi_iter.bi_sector).
4. Update bio.bi_iter.bi_sector = offset; bio.bi_bdev = devlist[zone-disk-idx].bdev.
5. submit_bio_noacct.

REQ-4: Sector → (zone, disk, offset) mapping:
1. zone := find_zone (linear search or binary-search if many zones).
2. relative_sector := sector - zone.start.
3. chunk_idx := relative_sector / chunk_sectors.
4. disk_idx := chunk_idx % zone.nb_dev.
5. stripe_idx := chunk_idx / zone.nb_dev.
6. offset := zone.dev_start + stripe_idx * chunk_sectors + (relative_sector % chunk_sectors).

REQ-5: Per-disk size mismatch handling:
- Smallest-first zone: all disks contribute (size = N × smallest_disk).
- Subsequent zone: only larger disks contribute (size += (N-1) × (next_smallest - smallest)).
- Inner zones use fewer disks; reduces stripe-width.

REQ-6: Layout types:
- ORIGINAL: chunk_idx = relative_sector / chunk_sectors per zone.
- ALTERNATIVE: chunk-rotation across zones.

REQ-7: Failure semantics:
- Any disk failure → array fails (no redundancy).
- Linux md raid0 returns -EIO immediately on any disk failure for any IO touching that disk's stripes.

REQ-8: Discard / Trim:
- Per-disk discard issued; per-bio split mirrors data-IO path.

REQ-9: Reshape (rare):
- raid0 → raid4/5 takeover supported (other-direction takes data preservation).

REQ-10: Per-zone validation at run:
- All zones non-overlapping.
- Per-zone disk-count > 0.
- Cumulative coverage matches array size.

## Acceptance Criteria

- [ ] AC-1: Create raid0 with 4 equal-sized disks: array size = 4 × disk-size; bio writes spread across 4 disks.
- [ ] AC-2: Sector mapping: sector S → disk (S/chunk) % 4; verifiable via dump_zones.
- [ ] AC-3: Throughput: linear-write 4-disk raid0 ≥ 3.5× single-disk bandwidth (assuming chunked-IO).
- [ ] AC-4: Chunk boundary split: bio spanning chunk boundary correctly split into per-chunk sub-bios.
- [ ] AC-5: Non-uniform disks: 1×100GB + 3×200GB; zones correctly created (zone-1: 4×100GB; zone-2: 3×100GB).
- [ ] AC-6: Disk failure: pull one disk; subsequent IO returns -EIO; array marked failed.
- [ ] AC-7: Discard propagation: 1GB discard; per-bio split + propagated to each disk.
- [ ] AC-8: Takeover raid0→raid4: existing raid0 converted to raid4 + parity disk added; data preserved.
- [ ] AC-9: 16-disk raid0 stress: heavy random IO; throughput scales; no data corruption.
- [ ] AC-10: mdadm raid0 test suite passes.

## Architecture

`R0Conf`:

```
struct R0Conf {
  mddev: KArc<MdDev>,
  strip_zone: KVec<StripZone>,
  nr_strip_zones: u32,
  devlist: KVec<KArc<MdRdev>>,                  // flat array; per-zone slice
  layout: u32,
}

struct StripZone {
  zone_end: u64,                                // end sector in raid (exclusive)
  dev_start: u64,                                // start sector on each disk in this zone
  nb_dev: u32,                                   // disks contributing to this zone
}
```

`Raid0::make_request(mddev, bio)`:
1. conf := mddev.private.
2. orig_sector := bio.bi_iter.bi_sector.
3. orig_size := bio.bi_iter.bi_size.
4. If !is_io_in_chunk_boundary(mddev, conf, bio):
   - raid0_make_request_split → split into per-chunk bios; recurse.
   - return.
5. zone := find_zone(conf, orig_sector).
6. (disk_idx, offset) := map_sector(mddev, zone, orig_sector).
7. bio.bi_iter.bi_sector = offset.
8. bio.bi_bdev = devlist[zone disk-idx].bdev.
9. submit_bio_noacct(bio).

`R0Conf::map_sector(mddev, zone, sector, &offset)`:
1. relative := sector - zone_start.
2. chunk_idx := relative / chunk_sectors.
3. disk_idx := chunk_idx % zone.nb_dev.
4. stripe_idx := chunk_idx / zone.nb_dev.
5. offset := zone.dev_start + stripe_idx * chunk_sectors + (relative % chunk_sectors).
6. Return disk_idx.

`R0Conf::find_zone(conf, sector)`:
1. for i in 0..nr_strip_zones:
   - if sector < strip_zone[i].zone_end: return &strip_zone[i].
2. Return NULL (out-of-range).

`R0Conf::create_strip_zones(mddev, &priv_conf)`:
1. Sort disks by size.
2. Create zones in order: smallest-first.
3. Per-zone: nb_dev := disks-with-≥-this-size; dev_start := previous-zone-end.
4. Validate cumulative size matches mddev.array_sectors.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `zone_idx_bounded` | OOB | per-zone idx < nr_strip_zones; defense against find_zone returning OOB. |
| `disk_idx_bounded` | OOB | per-disk idx < zone.nb_dev. |
| `chunk_boundary_check` | INVARIANT | bios crossing chunk-boundary are split before dispatch. |
| `cumulative_zone_size_matches` | INVARIANT | Σ(zone.zone_end - zone.zone_start) × nb_dev matches array size. |
| `zone_non_overlapping` | INVARIANT | zone[i].zone_end ≤ zone[i+1].zone_start. |

### Layer 2: TLA+

`drivers/md/raid0_zone_map.tla`:
- Per-sector deterministic mapping to (disk, offset).
- Properties:
  - `safety_unique_per_sector` — for each raid-sector, exactly one (disk, offset) pair.
  - `safety_within_disk_bounds` — offset always within target-disk capacity.
  - `safety_chunk_atomic` — bio not split inside a chunk; only at chunk-boundary.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Raid0::make_request` post: bio remapped (single-chunk) or split (multi-chunk) | `Raid0::make_request` |
| `R0Conf::find_zone` post: returned zone contains the sector | `R0Conf::find_zone` |
| `R0Conf::map_sector` post: offset < zone.dev_start + zone.nb_sectors_per_disk | `R0Conf::map_sector` |
| `R0Conf::create_strip_zones` post: nr_strip_zones ≥ 1; cumulative coverage matches array_sectors | `R0Conf::create_strip_zones` |

### Layer 4: Verus/Creusot functional

`raid-sector S → disk D, offset O: read disk D at offset O returns same data as if S were a single-disk sector` semantic equivalence: per-sector the (disk, offset) tuple deterministically encodes the sector.

## Hardening

(Inherits row-1 features from `drivers/md/00-overview.md` § Hardening.)

raid0-specific reinforcement:

- **Per-zone overlap check** — defense against malformed config causing duplicate-mapping.
- **Per-bio chunk-boundary split** — defense against per-disk straddling causing partial-write.
- **Per-disk failure → array-fail strict** — defense against silent-failure on missing disk.
- **Per-zone validation at run** — defense against incomplete coverage.
- **Per-disk size-validated** — defense against zone with disk smaller than zone.nb_sectors.
- **Chunked-IO required by spec** — defense against non-power-of-2 chunk-size causing modulo-overhead.
- **Disk-list immutable post-init** — defense against concurrent disk-add corrupting zone-table.
- **Per-bio refcount** if multi-chunk — defense against partial-bio_endio.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- raid1 / raid5 / raid6 / raid10 (covered separately)
- md-core (covered in `md-core.md` future Tier-3)
- mdadm userspace
- Implementation code
