---
title: "Tier-3: drivers/md/dm-thin.c — thin-provisioning target (overcommit + COW snapshots + per-pool data/metadata + thin-pool deferred-bio)"
tags: ["tier-3", "drivers-md", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

dm-thin is the device-mapper target backing thin-provisioning + writable snapshots — a single thin-pool (one pair of metadata + data block-devices) hosts many thin-volumes that allocate-on-write data blocks lazily. Per-thin: B-tree mapping LBA → data-block-index. Per-snapshot: copy-on-write reference to source thin's mapping; first-write COWs the block. Used by Docker overlay storage, LVM2 thin-pools, k8s persistent-volume thin-snapshots. Backing store: dm-thin-metadata.c persists B-trees + space-maps via dm-bufio (transactional + crash-safe).

This Tier-3 covers `dm-thin.c` (~4558 lines) + `dm-thin-metadata.c` (~2160 lines).

### Acceptance Criteria

- [ ] AC-1: dmsetup create thin-pool: 1GB metadata + 100GB data; pool active.
- [ ] AC-2: dmsetup create thin1 (10GB virtual): writes only allocate as needed; df shows actual usage growing.
- [ ] AC-3: Snapshot: dmsetup create snap1 from thin1; both shared; first write to snap1 COWs that block.
- [ ] AC-4: 1000-thin-volume scale: pool with 1000 thin-volumes; mappings indexed via B-tree; lookup ≤ ~10us.
- [ ] AC-5: Out-of-space: thin-volume writes hit pool-full; -ENOSPC returned.
- [ ] AC-6: Discard: discard thin-volume range; data-blocks freed back to pool.
- [ ] AC-7: Crash recovery: kill -9 during write; mount; previously-uncommitted writes lost cleanly; pool consistent.
- [ ] AC-8: dm-thin xfstests pass.
- [ ] AC-9: 4K block-size pool: thin-volume with 4KB block-size; verify mapping-block-overhead acceptable.
- [ ] AC-10: Performance: linear write throughput ≥ 80% of underlying data-dev bandwidth.

### Architecture

`Pool`:

```
struct Pool {
  pool_md: KArc<MappedDevice>,
  metadata: KArc<DmPoolMetadata>,
  data_dev: KArc<DmDev>,
  metadata_dev: KArc<DmDev>,
  block_size: u32,
  block_size_mask: u64,
  block_size_shift: u32,
  low_water_blocks: u64,
  bio_prison: KArc<BioPrison>,
  worker: Worker,
  wq: KArc<Workqueue>,
  notify_fn: PoolNotifyFn,
  read_only: AtomicBool,
  out_of_data_space: AtomicBool,
  ...
}

struct ThinC {
  pool: KArc<Pool>,
  td: KArc<DmPoolThinDevice>,
  dev_id: u32,
  requeue_lock: SpinLock<ListHead>,
  requeue_list: ListHead,
}

struct ThinNewMapping {
  list: ListNode,
  thin: KArc<ThinC>,
  virt_block: u64,                            // thin LBA-block
  data_block: u64,                            // pool data-block
  prepared: ThinPrepareKind,                  // none, partial, complete
  bio: Option<KArc<Bio>>,                     // original bio
  cell: Option<DmBioPrisonCell>,
}
```

`Thin::map_bio(target, bio)`:
1. thin := target.private.
2. virt_block := bio.bi_iter.bi_sector / pool.block_size.
3. If pool.read_only && bio.bi_op == WRITE: bio_io_error.
4. dm_pool_lookup_block(thin.td, virt_block, &lookup_result):
   - found && !shared: bio.bi_iter.bi_sector = data_block * pool.block_size + (bio.bi_iter.bi_sector % pool.block_size); return DM_MAPIO_REMAPPED.
   - found && shared: process_shared_bio(thin, bio); return DM_MAPIO_SUBMITTED.
   - !found && bio.bi_op == READ: zero-fill bio range; bio_endio.
   - !found && bio.bi_op == WRITE: process_bio(thin, bio); return DM_MAPIO_SUBMITTED.

`Thin::process_bio(thin, bio)`:
1. cell := bio_prison_isolate(thin, virt_block, bio).
2. If !cell (already isolated): defer to existing chain.
3. Else: schedule provision_block(thin, virt_block, cell, bio) on worker.

`Pool::provision_block_worker(thin, virt_block, cell, bio)`:
1. dm_pool_alloc_data_block(pool.metadata, &new_block).
2. If alloc fails: bio_io_error; cell release.
3. Allocate ThinNewMapping; populate.
4. zero_new_blocks → blkdev_issue_zeroout(data_block).
5. dm_pool_insert_block(thin.td, virt_block, new_block, NOT_SHARED).
6. Re-issue bio remapped to data_dev at new_block.
7. cell release on bio_endio.

`DmPoolMetadata::lookup_block(td, virt_block, &result)`:
1. Acquire metadata.root_lock.
2. dm_btree_lookup(metadata.details_root, &lookup_key, &lookup_value).
3. result.found = (return == 0).
4. result.data_block = lookup_value.block.
5. result.shared = lookup_value.flags & SHARED.
6. Release lock.

### Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-btree implementation details (covered in `drivers/md/persistent-data.md` future Tier-3)
- dm-bufio block manager (covered separately)
- thinp-tools userspace utilities
- dm-cache (covered in `dm-cache.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pool` | per-thin-pool state | `drivers::md::thin::Pool` |
| `struct thin_c` | per-thin-volume target ctx | `ThinC` |
| `struct dm_thin_new_mapping` | per-COW operation | `ThinNewMapping` |
| `struct dm_pool_metadata` (dm-thin-metadata.c) | per-pool persistent metadata | `DmPoolMetadata` |
| `pool_ctr(target, argc, argv)` | thin-pool target-create | `Pool::ctr` |
| `pool_dtr(target)` | destroy | `Pool::dtr` |
| `pool_map(target, bio)` | per-pool bio dispatch | `Pool::map_bio` |
| `pool_endio(...)` | bio completion | `Pool::endio` |
| `pool_status(...)` | dmsetup status | `Pool::status` |
| `pool_message(...)` | DM_TARGET_MSG dispatch | `Pool::message` |
| `thin_ctr(target, argc, argv)` | thin-target create | `Thin::ctr` |
| `thin_dtr(target)` | destroy | `Thin::dtr` |
| `thin_map(target, bio)` | per-thin bio dispatch | `Thin::map_bio` |
| `thin_endio(...)` | bio completion | `Thin::endio` |
| `process_bio(thin, bio)` | per-thin bio decision | `Thin::process_bio` |
| `process_cell(...)` | deferred-cell-list processing | `Thin::process_cell` |
| `provision_block(...)` | alloc new data block | `Pool::provision_block` |
| `process_shared_bio(...)` | COW shared block | `Thin::process_shared_bio` |
| `dm_pool_metadata_open(...)` | open pool metadata | `DmPoolMetadata::open` |
| `dm_pool_metadata_close(...)` | close + commit | `DmPoolMetadata::close` |
| `dm_pool_create_thin(...)` | create thin-volume | `DmPoolMetadata::create_thin` |
| `dm_pool_create_snap(...)` | create snapshot of thin | `DmPoolMetadata::create_snap` |
| `dm_pool_alloc_data_block(...)` | space-map alloc | `DmPoolMetadata::alloc_data_block` |
| `dm_pool_lookup_block(...)` | LBA → data-block lookup | `DmPoolMetadata::lookup_block` |
| `dm_pool_insert_block(...)` | mapping insert | `DmPoolMetadata::insert_block` |
| `dm_pool_remove_block(...)` | mapping unmap (discard) | `DmPoolMetadata::remove_block` |
| `dm_pool_commit_metadata(...)` | flush B-tree to data-bdev | `DmPoolMetadata::commit` |

### compatibility contract

REQ-1: Per-pool `pool` struct:
- `pool_md` (mapped_device).
- `metadata` (KArc<DmPoolMetadata>).
- `data_dev` (block_device for data).
- `metadata_dev` (block_device for metadata).
- `block_size` (sectors per data block; typically 64KB-1MB).
- `low_water_blocks` (low-water-mark; alert at this point).
- `cell_sort_array` (per-block cell-sort).
- `bio_prison` (per-pool dm_bio_prison; serializes per-block ops).
- `notify_fn` (callback on out-of-data-space / metadata-error).
- `wq` (workqueue for deferred bio processing).
- `worker_thread` (thread pool).
- `out_of_data_space` (state).
- `read_only` (read-only after metadata error).

REQ-2: Per-thin-volume `thin_c`:
- `pool` (back-ref).
- `td` (per-thin metadata handle).
- `dev_id` (per-thin numeric ID, unique within pool).
- `requeue` (per-volume bio requeue list during pool fail).
- `mapping_lock` (serializes mapping ops).
- `endio_hook_pool` (bio_endio per-thin slab).

REQ-3: Per-pool data-bdev partitioned into fixed-size blocks (block_size):
- Block #0..max_data_block.
- Per-block usage tracked in space-map (allocated / free).

REQ-4: Per-pool metadata-bdev contains:
- B-tree mapping (thin_id, LBA-block) → data-block-index.
- Per-thin device list.
- Per-snapshot-relation tracking.
- Space-map for data + metadata blocks.

REQ-5: Per-thin bio dispatch (`thin_map`):
1. Look up thin-block (bio.bi_iter.bi_sector / pool.block_size).
2. dm_pool_lookup_block(td, thin_block, &lookup_result):
   - lookup_result.found: data_block, shared.
   - !found: hole.
3. If hole + write: defer via process_bio → provision_block.
4. If found + !shared: remap bio to data_dev at data_block.
5. If found + shared + write: defer for COW via process_shared_bio.
6. If found + read: remap to data_dev (shared OK for read).

REQ-6: provision_block flow (per-write to hole):
1. Acquire bio_prison cell for thin-block.
2. dm_pool_alloc_data_block(metadata, &new_block) — space-map allocates.
3. dm_pool_insert_block(td, thin_block, new_block, ...) — update B-tree.
4. Allocate dm_thin_new_mapping; queue to worker thread.
5. Worker:
   - Zero new data block (per-pool zero_new_blocks setting).
   - Re-issue original bio remapped to data_dev at new_block.
   - Release bio_prison cell.

REQ-7: process_shared_bio (COW for snapshot-on-write):
1. Acquire bio_prison cell.
2. dm_pool_alloc_data_block(metadata, &new_block).
3. Allocate dm_thin_new_mapping with copy_op = src_block → new_block.
4. Worker:
   - Read src_block; write to new_block (deep-copy).
   - dm_pool_insert_block(td, thin_block, new_block, NOT_SHARED).
   - Re-issue original bio remapped to data_dev at new_block.
   - Release cell.

REQ-8: dm_pool_create_snap:
- Create thin with same dev_id'th metadata as source.
- All blocks marked SHARED.
- Source thin also marked SHARED for those blocks.
- First write to either thin triggers COW.

REQ-9: Per-pool transactional commit:
- Worker thread periodically commits metadata changes (every ~1 second or after Nth update).
- dm_pool_commit_metadata flushes B-tree via dm-bufio.
- On crash before commit: lost mappings → next-mount sees pre-commit state.

REQ-10: Out-of-space handling:
- low-water-mark trigger: notify_fn called; userspace can extend pool.
- 100%-full: writes return -ENOSPC; reads OK.
- Configurable via "no_space_timeout" message.

REQ-11: dm_bio_prison per-block locking:
- Per-block "cell" tracks active bios for that block.
- Per-cell first bio admitted; subsequent queued.
- On completion: dequeue + release cell + admit next.

REQ-12: Persistent-data layer (drivers/md/persistent-data/):
- dm-btree.c: copy-on-write B-tree.
- dm-block-manager.c: bufio-backed block I/O.
- dm-space-map.c: persistent space-map (free-list as bitmap).
- dm-transaction-manager.c: rollback-safe transaction wrapping.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `data_block_idx_bounded` | OOB | per-data-block index < pool.max_data_blocks; defense against B-tree returning stale OOB index. |
| `bio_prison_no_uaf` | UAF | per-cell ref-count under bio_prison.lock; defense against cell-free during dispatch. |
| `metadata_commit_atomic` | INVARIANT | metadata commit transactional via dm-transaction-manager; either fully applied or fully rolled back. |
| `space_map_no_double_alloc` | INVARIANT | per-data-block alloc'd at most once until freed; defense against double-allocation corrupting B-tree. |
| `cow_preserves_old_block` | INVARIANT | shared-block COW writes to NEW block; old block remains valid for source thin. |

### Layer 2: TLA+

`drivers/md/thin_provisioning.tla`:
- Per-thin-LBA-block state ∈ {Hole, AllocatedExclusive(data_block), AllocatedShared(data_block), Snapshotting}.
- Transitions: write to Hole → AllocatedExclusive (provision); write to Shared → COW → new AllocatedExclusive; snapshot-create → all blocks transition Shared.
- Properties:
  - `safety_no_data_loss_on_cow` — COW writes new block; old block preserved for snapshot.
  - `safety_per_block_atomic_alloc` — provision atomic (alloc + insert).
  - `liveness_provision_eventually_completes` — assuming pool not OOS, hole-write eventually transitions to Allocated.

`drivers/md/thin_metadata_commit.tla`:
- Metadata commit state: Dirty, Committing, Clean.
- Properties:
  - `safety_recovered_state_at_last_commit` — post-crash state == state at last commit.
  - `safety_no_partial_commit_visible` — Committing state never visible at next mount.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Thin::map_bio` post: bio either remapped to data_dev or queued for deferred processing | `Thin::map_bio` |
| `Pool::provision_block` post: B-tree updated; data-block alloc'd; bio resubmitted | `Pool::provision_block` |
| `Thin::process_shared_bio` post: COW issued; new block allocated; B-tree updated to NOT_SHARED | `Thin::process_shared_bio` |
| Per-pool metadata.commit synchronized with worker via metadata.commit_lock | invariants on commit |
| Per-data-block alloc'd in space-map iff in some thin's B-tree | invariants on space-map vs B-tree consistency |

### Layer 4: Verus/Creusot functional

`Snapshot semantics: snap1 = snapshot of thin1; per-LBA read from snap1 returns same data as read from thin1 at snapshot-time`: per-LBA per-(thin, snap) the read result reflects the source-state at snapshot-creation, regardless of subsequent thin writes (which COW).

### hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-thin-specific reinforcement:

- **Metadata transactional commit** — defense against partial-state on crash.
- **Per-data-block bio_prison cell** — defense against concurrent provision-races on same block.
- **COW deep-copy before B-tree update** — defense against snapshot-divergence.
- **Per-pool low-water-mark notify_fn** — defense against silent out-of-space surprise.
- **read-only mode on metadata-error** — defense against further corruption after detected fault.
- **zero_new_blocks** option — defense against data-block-reuse leaking previous-tenant data to new owner.
- **Per-thin dev_id unique within pool** — defense against B-tree key collision.
- **Per-snapshot ref-count on shared blocks** — defense against premature-free.
- **dm-bufio + dm-transaction-manager rollback** — defense against partial-update on commit failure.
- **discard threshold rate-limit** — defense against discard-flood corrupting space-map.
- **per-pool space-map balance check** at commit — defense against B-tree drift causing space-map mismatch.
- **Per-thin bio_prison.cell ref-count** — defense against double-release on requeue.
- **Pool out_of_data_space flag atomic** — defense against torn-state during alloc race.

