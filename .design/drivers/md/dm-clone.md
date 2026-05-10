# Tier-3: drivers/md/dm-clone-target.c — efficient block-level cloning (origin → destination CoW + bg copy + persistent metadata)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-clone-target.c
  - drivers/md/dm-clone-metadata.c
  - drivers/md/dm-clone-metadata.h
-->

## Summary

dm-clone is the device-mapper target for efficient online block-level cloning of an immutable source (origin) to a destination — per-region (default 64KB) on-demand copy as bios touch un-hydrated regions; meanwhile background hydration copies un-touched regions; persistent metadata tracks per-region hydrated/dirty state. Unlike dm-snapshot (which is CoW-on-origin-write), dm-clone copies origin TO destination on access, leaving origin untouched. Used for: VM disk-image clone (qemu image-fork), backup-restore, immutable-image deploy. Provides immediate-availability with deferred copy.

This Tier-3 covers `dm-clone-target.c` (~2208) + `dm-clone-metadata.c` (~1020).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct clone` | per-target state | `drivers::md::clone::Clone` |
| `struct dm_clone_metadata` | per-target persistent state | `DmCloneMetadata` |
| `clone_ctr(target, argc, argv)` | target-create | `Clone::ctr` |
| `clone_dtr(target)` | dtor | `Clone::dtr` |
| `clone_map(target, bio)` | per-bio dispatch | `Clone::map_bio` |
| `clone_endio(target, bio, err)` | bio completion | `Clone::endio` |
| `clone_status(...)` | dmsetup status | `Clone::status` |
| `clone_message(...)` | DM_TARGET_MSG | `Clone::message` |
| `dm_clone_metadata_open(...)` | metadata-open | `DmCloneMetadata::open` |
| `dm_clone_metadata_close(...)` | metadata-close + commit | `DmCloneMetadata::close` |
| `dm_clone_set_region_hydrated(...)` | per-region mark hydrated | `DmCloneMetadata::set_region_hydrated` |
| `dm_clone_is_region_hydrated(...)` | per-region query | `DmCloneMetadata::is_region_hydrated` |
| `dm_clone_get_dirty_bitmap(...)` | per-bitmap iter | `DmCloneMetadata::get_dirty_bitmap` |
| `do_hydration_work(work)` | background hydration worker | `Clone::do_hydration_work` |
| `process_bio(clone, bio)` | per-bio process | `Clone::process_bio` |
| `process_discard_bio(clone, bio)` | per-discard process | `Clone::process_discard_bio` |
| `hydrate_bio_region(...)` | per-region hydrate | `Clone::hydrate_bio_region` |
| `commit_metadata(clone)` | persistent commit | `Clone::commit_metadata` |

## Compatibility contract

REQ-1: Per-target `clone`:
- `ti` (back-ref).
- `origin_dev` (DmDev for source; immutable).
- `dest_dev` (DmDev for destination).
- `meta_dev` (DmDev for metadata).
- `region_size` / `region_shift` (sectors per region; default 64KB).
- `nr_regions` (count).
- `cmd` (KArc<DmCloneMetadata>).
- `hydration_threshold` (per-target work-queue threshold).
- `hydration_batch_size` (sectors per hydrate work).
- `kmcc_wq` / `kmcc_thread` (background hydrate kthread).
- `wq` (per-target unbound workqueue for IO).
- `kcopyd_client` (for region-copy).
- `bio_prison` (per-region cell-locking).
- `flags` (DM_CLONE_*: DISCARD_PASSDOWN, NO_HYDRATION, READ_ONLY, etc.).
- `mode` (DM_CLONE_MODE_RW / _RO / _FAIL).

REQ-2: Per-region state:
- HYDRATED: copied from origin to dest; reads from dest.
- NOT_HYDRATED: not yet copied; reads from origin.
- HYDRATING: copy in progress; bios deferred.
- DIRTY: dest has been written; origin no longer authoritative.

REQ-3: Per-bio dispatch (`clone_map`):
1. region := bio.bi_iter.bi_sector >> clone.region_shift.
2. lock bio_prison cell.
3. If is_region_hydrated(region):
   - bio remap to dest_dev; submit.
4. Else:
   - For READ: bio remap to origin_dev (read-from-source).
   - For WRITE: hydrate region first; defer bio.

REQ-4: Per-region hydration:
- start_copy: dm_kcopyd from origin to dest at region offset.
- On complete: mark hydrated; deferred bios re-submitted.

REQ-5: Per-bio write to non-hydrated:
1. Allocate hydrate_work.
2. dm_kcopyd_copy origin→dest for region.
3. On complete: set_region_hydrated; resubmit bio.

REQ-6: Background hydration (kmcc_thread):
- Walk non-hydrated regions in metadata.
- Per-region: hydrate_bio_region (kcopyd).
- Configurable rate via `hydration_batch_size` per pass.

REQ-7: Discard handling:
- Per-discard region: clear hydration metadata + propagate to origin (if DISCARD_PASSDOWN) or just dest.

REQ-8: Persistent metadata:
- Per-region 1-bit hydrated state.
- Persistent on meta_dev via dm-bufio.
- Transactional commit on metadata_close + periodic checkpoint.

REQ-9: Per-target modes:
- RW: writes hydrate-then-write.
- RO: read-only; no hydration; reads from origin/dest as appropriate.
- FAIL: any IO returns -EIO (used after fatal error).

REQ-10: dmsetup messages:
- "hydration_threshold N": rate-limit bg hydration.
- "hydration_batch_size N": per-pass batch.
- "set_mode rw|ro|fail": mode change.
- "shutdown_metadata": flush + close metadata.
- "delete_old_origin": discard origin reference (after full hydration).

REQ-11: Per-region bio_prison cell:
- Per-region serializes access during hydration.
- Per-cell wait-list for deferred bios.

REQ-12: Crash recovery:
- Metadata on meta_dev with transactional commit.
- Replay-on-mount: re-confirm per-region state.

## Acceptance Criteria

- [ ] AC-1: dmsetup create dm-clone test_clone: origin /dev/sda1 + dest /dev/sdb1 + meta /dev/sdc1.
- [ ] AC-2: Read non-hydrated region: serves from origin.
- [ ] AC-3: Write non-hydrated region: triggers hydration; subsequent reads serve from dest.
- [ ] AC-4: Background hydration: progress visible via dmsetup status; completes asynchronously.
- [ ] AC-5: Discard: per-region discard propagates to dest + origin (if passdown).
- [ ] AC-6: Crash recovery: kill -9 mid-hydration; reboot; metadata-replay restores; resume.
- [ ] AC-7: Mode change: RW → RO; subsequent writes return -EROFS.
- [ ] AC-8: 1TB clone stress: heavy random writes; bg hydration completes; full clone consistency verified.
- [ ] AC-9: delete_old_origin: after 100% hydrated; origin reference dropped; dm-clone now functions as standalone dest.
- [ ] AC-10: dm-clone xfstests pass.

## Architecture

`Clone`:

```
struct Clone {
  ti: KArc<DmTarget>,
  origin_dev: KArc<DmDev>,
  dest_dev: KArc<DmDev>,
  meta_dev: KArc<DmDev>,
  cmd: KArc<DmCloneMetadata>,
  region_size: u32,
  region_shift: u32,
  nr_regions: u64,
  hydration_threshold: u32,
  hydration_batch_size: u32,
  kmcc_wq: KArc<Workqueue>,
  kmcc_thread: KThread,
  wq: KArc<Workqueue>,
  kcopyd_client: KArc<DmKcopyd>,
  bio_prison: KArc<BioPrison>,
  flags: u32,
  mode: AtomicI32,
  ...
}

struct CloneHydrationWork {
  list: ListNode,
  clone: KArc<Clone>,
  region: u64,
  bio: Option<KArc<Bio>>,                       // deferred bio (if any)
  cell: BioPrisonCell,
}
```

`Clone::map_bio(target, bio)`:
1. clone := target.private.
2. region := bio.bi_iter.bi_sector >> clone.region_shift.
3. cell := bio_prison_isolate(clone, region, bio).
4. If !cell: defer; return DM_MAPIO_SUBMITTED.
5. If clone.cmd.is_region_hydrated(region):
   - bio.bi_iter.bi_sector += clone.dest_offset; bio.bi_bdev = clone.dest_dev.bdev.
   - submit_bio_noacct(bio).
   - return DM_MAPIO_REMAPPED.
6. Else (not hydrated):
   - If bio.bi_op == REQ_OP_READ:
     - bio.bi_iter.bi_sector += clone.origin_offset; bio.bi_bdev = clone.origin_dev.bdev.
     - return DM_MAPIO_REMAPPED.
   - Else (WRITE/DISCARD):
     - Allocate hydrate_work; queue to clone.wq.
     - return DM_MAPIO_SUBMITTED.

`Clone::hydrate_bio_region(clone, region, bio)` (worker):
1. dm_kcopyd_copy(origin → dest, region).
2. On complete:
   - clone.cmd.set_region_hydrated(region).
   - If bio: re-submit (now hydrated path).
   - Release bio_prison cell.

`Clone::do_hydration_work(work)` (kmcc_thread):
1. Walk dirty-bitmap of non-hydrated regions.
2. For each region (up to hydration_batch_size):
   - hydrate_bio_region(clone, region, NULL).

`DmCloneMetadata::set_region_hydrated(cmd, region)`:
1. lock cmd.lock.
2. mark dirty bitmap[region] = HYDRATED.
3. periodic commit if dirty count > threshold.
4. unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `region_idx_bounded` | OOB | per-region idx < nr_regions; defense against OOB metadata access. |
| `bio_prison_no_uaf` | UAF | per-cell ref-counted; defense against concurrent hydrate + io. |
| `mode_state_terminal` | INVARIANT | DM_CLONE_MODE_FAIL terminal; no transitions back. |
| `hydration_atomic_marker` | INVARIANT | per-region hydrated state set atomically per metadata commit boundary. |
| `metadata_commit_transactional` | INVARIANT | metadata commits transactional; partial-state not visible. |

### Layer 2: TLA+

`drivers/md/clone_lifecycle.tla`:
- Per-region state ∈ {NotHydrated, Hydrating, Hydrated, Dirty, Discarded}.
- Properties:
  - `safety_no_dest_read_before_hydrate` — non-hydrated read goes to origin.
  - `safety_write_hydrates_region` — Write to NotHydrated triggers Hydrating.
  - `liveness_hydrating_eventually_hydrated` — Hydrating → Hydrated assuming kcopyd succeeds.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Clone::map_bio` post: bio remapped (read-from-origin OR read-from-dest OR queued for hydrate) | `Clone::map_bio` |
| `Clone::hydrate_bio_region` post: per-region kcopyd submitted; on-complete metadata updated | `Clone::hydrate_bio_region` |
| `DmCloneMetadata::set_region_hydrated` post: bitmap set; commit pending | `DmCloneMetadata::set_region_hydrated` |
| Per-region hydrate state monotonic: NotHydrated → Hydrated (no transition back) | invariants on metadata |

### Layer 4: Verus/Creusot functional

`Per-region: read returns origin-data if NotHydrated; dest-data if Hydrated/Dirty` semantic equivalence: per-LBA per-region the read result reflects origin-state-at-clone-creation-OR-modified-dest-state.

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-clone-specific reinforcement:

- **Per-region bio_prison serialization** — defense against concurrent hydrate + IO causing UAF.
- **Metadata transactional commit** — defense against partial hydrate state on crash.
- **Per-region hydrate state monotonic** — defense against bug-induced regression.
- **DM_CLONE_MODE_FAIL terminal** — defense against post-fail IO causing further corruption.
- **Background hydration rate-limited** — defense against hydration hogging IO bandwidth.
- **dm_kcopyd async-copy bounded** — defense against unbounded outstanding copies.
- **Per-cell ref-counted** — defense against double-release on requeue.
- **Discard validated against region boundary** — defense against partial-region inconsistency.
- **Origin immutability assumed** — KVM doesn't enforce; userspace responsibility.
- **delete_old_origin only after 100% hydrated** — defense against premature origin-loss.
- **Per-bio refcount during hydrate** — defense against double-endio on requeue.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-snapshot (covered in `dm-snapshot.md` Tier-3; CoW-on-origin-write semantics)
- dm-thin (covered in `dm-thin.md` Tier-3; thin-pool with snapshots)
- Generic block-layer (covered in `block/00-overview.md`)
- Implementation code
