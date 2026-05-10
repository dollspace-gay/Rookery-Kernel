---
title: "Tier-3: drivers/md/dm-snap.c — dm-snapshot (CoW snapshot of origin block-device + persistent exception store)"
tags: ["tier-3", "drivers-md", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

dm-snapshot is the device-mapper target that creates a writable copy-on-write (CoW) snapshot of an origin block-device — the snapshot starts as a "read-through to origin" view; per-write to either origin OR snapshot copies the original chunk to a separate exception-store before applying the write. Used by: LVM snapshots (legacy, pre-thin), backup tools, virt disk-snapshots. Per-snapshot `dm_exception` records (origin-chunk-idx → cow-chunk-idx) maintained in either persistent (durable across reboot, on-disk B-tree) or transient (memory-only, lost on reboot) store.

This Tier-3 covers `dm-snap.c` (~2857 lines) + `dm-snap-persistent.c` (~980) + `dm-snap-transient.c` (~158) + `dm-exception-store.c`.

### Acceptance Criteria

- [ ] AC-1: dmsetup create snap_test with snapshot of /dev/sda1; cow on /dev/sdb1; reads transparent.
- [ ] AC-2: Write to origin: chunk CoW'd; subsequent snapshot read returns pre-write data.
- [ ] AC-3: Write to snapshot: chunk CoW'd; origin unchanged; snapshot reads return new data.
- [ ] AC-4: Multiple snapshots: origin write triggers CoW for all; per-snapshot independent.
- [ ] AC-5: Persistent store: snapshot survives reboot; exception list reloaded from CoW.
- [ ] AC-6: Transient store: snapshot lost on reboot; cow data discarded.
- [ ] AC-7: Snapshot full: write to origin causes CoW; exception-store full → snapshot invalid.
- [ ] AC-8: Snapshot-merge: dmsetup snapshot-merge target; CoW data merged back into origin.
- [ ] AC-9: Concurrent IO + CoW: 100 concurrent writes; per-chunk tracked-chunk hash prevents UAF.
- [ ] AC-10: dm-snapshot xfstests pass.

### Architecture

`DmSnapshot`:

```
struct DmSnapshot {
  origin_dev: KArc<DmDev>,
  cow_dev: KArc<DmDev>,
  chunk_size: u32,
  chunk_shift: u32,
  store: KArc<dyn DmExceptionStore>,
  pending_exceptions_lock: SpinLock<()>,
  pending_exceptions: KBox<[Hlist; PENDING_HASH_SIZE]>,
  complete_exceptions: KBox<[Hlist; COMPLETE_HASH_SIZE]>,
  active_lock: Mutex<()>,
  state_bits: AtomicU64,                      // STATE_RUNNING, _ACTIVE, _INVALID
  kcopyd_client: KArc<DmKcopyd>,
  tracked_chunk_lock: SpinLock<()>,
  tracked_chunk_hash: KBox<[Hlist; TRACKED_CHUNK_HASH_SIZE]>,
  ...
}

struct DmException {
  hash_list: HlistNode,
  old_chunk: u64,
  new_chunk: u64,
}

struct DmSnapPendingException {
  e: DmException,
  list: ListNode,
  origin_bios: BioList,
  snapshot_bios: BioList,
  copy_error: i32,
  started: bool,
  full_bio: Option<KArc<Bio>>,
}
```

`DmSnapshot::map_bio(target, bio)`:
1. snap := target.private.
2. chunk := bio.bi_iter.bi_sector >> snap.chunk_shift.
3. lock(snap.pending_exceptions_lock).
4. e := complete-exceptions lookup by chunk.
5. If e:
   - bio.bi_iter.bi_sector = (e.new_chunk << snap.chunk_shift) + (sector & ((1<<snap.chunk_shift)-1)).
   - bio.bi_bdev = snap.cow_dev.bdev.
   - unlock.
   - return DM_MAPIO_REMAPPED.
6. pe := pending-exceptions lookup by chunk.
7. If pe:
   - bio_list_add(&pe.snapshot_bios, bio).
   - unlock.
   - return DM_MAPIO_SUBMITTED.
8. Else (no exception):
   - For READ: bio.bi_bdev = snap.origin_dev.bdev; unlock; return DM_MAPIO_REMAPPED.
   - For WRITE: 
     - pe := alloc pending_exception.
     - pe.e.old_chunk = chunk.
     - pe.e.new_chunk = snap.store.alloc_new_chunk.
     - bio_list_add(&pe.snapshot_bios, bio).
     - hlist_add (pending list).
     - unlock.
     - start_copy(pe).
     - return DM_MAPIO_SUBMITTED.

`DmSnapshot::start_copy(pe)`:
1. dm_kcopyd_copy(snap.kcopyd_client, source=origin chunk, dest=cow chunk, copy_callback, pe).

`DmSnapshot::copy_callback(pe, error)`:
1. lock(snap.pending_exceptions_lock).
2. If error: mark snapshot INVALID; mark pe.copy_error.
3. Else: store.commit_exception(pe.e); move pe → complete_exceptions.
4. unlock.
5. For each bio in pe.snapshot_bios + .origin_bios: re-issue with appropriate remapping.
6. Free pe.

`DmOrigin::map_bio(target, bio)`:
1. origin := target.private.
2. If bio.bi_op == REQ_OP_READ: bio.bi_bdev = origin.dev.bdev; return DM_MAPIO_REMAPPED.
3. Else (WRITE):
   - Per-snapshot of this origin:
     - chunk := bio.bi_iter.bi_sector >> snap.chunk_shift.
     - If !exception_exists(snap, chunk):
       - Allocate pe; bio_list_add(&pe.origin_bios, bio); start_copy(pe).
   - If no CoW required: bio passes through.

### Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-thin (covered in `dm-thin.md` Tier-3; modern thin-provisioning replacement)
- dm-bufio (covered separately)
- LVM2 userspace
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dm_snapshot` | per-snapshot state | `drivers::md::snap::DmSnapshot` |
| `struct dm_exception_store` | per-snapshot CoW store abstraction | `DmExceptionStore` |
| `struct dm_exception` | per-CoW-region exception record | `DmException` |
| `struct dm_origin` | origin-target state | `DmOrigin` |
| `struct dm_snap_pending_exception` | per-write CoW in-progress | `DmSnapPendingException` |
| `snapshot_ctr(target, argc, argv)` | snapshot target ctor | `DmSnapshot::ctr` |
| `snapshot_dtr(target)` | dtor | `DmSnapshot::dtr` |
| `snapshot_map(target, bio)` | per-bio dispatch | `DmSnapshot::map_bio` |
| `snapshot_endio(target, bio, err)` | bio completion | `DmSnapshot::endio` |
| `origin_ctr(target, argc, argv)` | origin target ctor | `DmOrigin::ctr` |
| `origin_map(target, bio)` | origin per-bio (intercept writes) | `DmOrigin::map_bio` |
| `__find_pending_exception(s, bio)` | pending-exception lookup | `DmSnapshot::find_pending_exception` |
| `__lookup_exception(s, chunk)` | per-chunk exception lookup | `DmSnapshot::lookup_exception` |
| `__do_origin_write(snaps, bio)` | per-write CoW dispatcher | `DmSnapshot::do_origin_write` |
| `start_copy(pe)` | per-pending exception copy-trigger | `DmSnapshot::start_copy` |
| `copy_callback(...)` | dm-kcopyd completion | `DmSnapshot::copy_callback` |
| `dm_exception_store_create(...)` | store-create dispatch | `DmExceptionStore::create` |
| `dm_exception_store_destroy(...)` | store-destroy | `DmExceptionStore::destroy` |
| `persistent_ctr(...)` (dm-snap-persistent.c) | persistent-store create | `Persistent::ctr` |
| `persistent_read_metadata(...)` | resume from disk | `Persistent::read_metadata` |
| `persistent_commit_exception(...)` | per-exception commit to disk | `Persistent::commit_exception` |
| `transient_ctr(...)` (dm-snap-transient.c) | transient-store create | `Transient::ctr` |
| `dm_kcopyd_copy(...)` | per-region async-copy | `DmKcopyd::copy` |

### compatibility contract

REQ-1: Per-snapshot `dm_snapshot`:
- `origin_dev` (DmDev for origin block-device).
- `cow_dev` (DmDev for CoW exception-store).
- `chunk_size` (sectors per CoW chunk; default 64KB).
- `chunk_shift` (log2(chunk_size)).
- `store` (KArc<DmExceptionStore>; persistent / transient).
- `pending_exceptions` (hash → DmSnapPendingException list).
- `complete_exceptions` (hash → DmException list).
- `lock` (per-snapshot spinlock).
- `kcopyd_client` (per-snapshot dm-kcopyd ref).
- `tracked_chunk_lock` / `tracked_chunk_hash` (per-bio chunk-tracking).
- `state_bits` (RUNNING / ACTIVE / INVALID).

REQ-2: Per-exception `dm_exception`:
- `hash_list` (hash bucket).
- `old_chunk` (origin chunk-index).
- `new_chunk` (cow chunk-index).

REQ-3: Per-pending-exception `dm_snap_pending_exception`:
- `e` (the exception being created).
- `origin_bios` (bios deferred until copy completes).
- `snapshot_bios` (snapshot-side bios for same chunk).
- `started` (copy-started flag).

REQ-4: Per-bio dispatch (snapshot_map):
1. chunk := bio.bi_iter.bi_sector >> chunk_shift.
2. Acquire snapshot.lock.
3. Look up exception in complete_exceptions:
   - If found: bio remapped to cow_dev at (e.new_chunk * chunk_size) + (sector % chunk_size).
   - Submit; return DM_MAPIO_REMAPPED.
4. Else look up in pending_exceptions:
   - If pending: append bio to pending.snapshot_bios; return DM_MAPIO_SUBMITTED.
5. Else (no exception):
   - Read: bio remapped to origin_dev (snapshot transparently reads from origin for un-CoW'd regions).
   - Write to snapshot: allocate new pending exception; trigger CoW copy via dm-kcopyd; defer write.

REQ-5: Per-bio dispatch on origin (`origin_map`):
1. If bio.bi_op == REQ_OP_READ: pass-through to origin_dev.
2. If bio.bi_op == REQ_OP_WRITE:
   - For each snapshot of this origin:
     - chunk := bio.bi_iter.bi_sector >> snap.chunk_shift.
     - If !exception_exists(snap, chunk):
       - Allocate pending_exception; trigger CoW.
       - Defer original-write until all snapshots' CoWs complete.

REQ-6: CoW copy flow:
1. `start_copy(pending_exception)`:
   - dm_kcopyd_copy: source = origin chunk, dest = cow next-free chunk.
   - On completion: `copy_callback`.
2. `copy_callback`:
   - Move pending_exception → complete_exceptions.
   - Per-deferred-bio in pending.origin_bios + .snapshot_bios:
     - bio remap; submit.
   - Free pending_exception.

REQ-7: Persistent exception-store layout:
- Header (magic + version + chunk-size + nr-areas).
- Per-area: bitmap of allocated chunks + array of (old_chunk, new_chunk) records.
- Exceptions persisted by area; per-area commit via barrier.

REQ-8: Transient exception-store:
- All in memory; lost on snapshot-destroy or reboot.
- Used for ephemeral snapshots (e.g., short-lived backup).

REQ-9: Snapshot-merge target:
- Reverse-of-snapshot: merge CoW-data back into origin on resume.
- Used to roll-back origin to snapshot state.

REQ-10: Multiple snapshots of same origin:
- Per-origin: dm_origin tracks list of attached snapshots.
- Per-write to origin: CoW for ALL snapshots that don't have exception.

REQ-11: Snapshot full / invalid handling:
- If exception-store full: snapshot marked INVALID; subsequent IO returns -EIO.
- Per-`dm_snap_pending_exception` failure also marks snapshot invalid.

REQ-12: Per-bio tracked-chunk:
- tracked_chunk hash tracks in-flight bios per-chunk.
- Per-CoW: wait until all in-flight bios for this chunk complete.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `chunk_idx_bounded` | OOB | per-chunk index < origin_size / chunk_size; defense against OOB exception lookup. |
| `pending_exception_no_uaf` | UAF | per-pending-exception ref-counted; freed only after copy_callback. |
| `complete_exceptions_no_dup` | INVARIANT | per-old_chunk at most one complete exception. |
| `state_invalid_terminal` | INVARIANT | once INVALID, no transitions back to ACTIVE. |
| `kcopyd_async_complete` | INVARIANT | every start_copy paired with copy_callback (assuming no infinite-fail). |

### Layer 2: TLA+

`drivers/md/snapshot_lifecycle.tla`:
- Per-chunk state ∈ {Origin, Pending, CoWed, Invalid}.
- Properties:
  - `safety_pending_eventually_done` — Pending → CoWed (success) or Invalid (failure).
  - `safety_no_origin_overwrite_without_cow` — origin write must complete CoW for all attached snapshots first.
  - `safety_snapshot_view_consistent` — per-snapshot reads return CoWed data if exception, else origin data.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DmSnapshot::map_bio` post: bio remapped (read OR cow'd) OR queued (pending OR cow-trigger) | `DmSnapshot::map_bio` |
| `DmSnapshot::start_copy` post: dm_kcopyd_copy submitted; pe.started == true | `DmSnapshot::start_copy` |
| `DmSnapshot::copy_callback` post: pe moved to complete_exceptions OR snapshot marked INVALID | `DmSnapshot::copy_callback` |
| `DmOrigin::map_bio` write post: all attached snapshots' CoW triggered before origin write | `DmOrigin::map_bio` |
| Per-snap pending_exceptions + complete_exceptions disjoint | invariants on transition |

### Layer 4: Verus/Creusot functional

`Snapshot semantics: read from snapshot returns origin-state-at-snapshot-creation-time` semantic equivalence: per-LBA per-snap-time the read result matches origin state at that snapshot's creation.

### hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-snapshot-specific reinforcement:

- **Per-pending-exception ref-counted** — defense against UAF on copy completion.
- **Per-snapshot state INVALID terminal** — defense against post-fail use causing further corruption.
- **Per-chunk tracked_chunk hash** — defense against bio active during CoW causing UAF.
- **Persistent store transactional commit** — defense against partial-state on crash.
- **Per-snapshot exception_store ref-counted** — defense against destroy-during-IO.
- **dm-kcopyd async-copy bounded** — defense against unbounded outstanding copies.
- **Per-origin attached-snapshot list under origin_lock** — defense against concurrent attach/detach during write.
- **chunk_size validated power-of-2** — defense against modulo-overhead in remap.
- **Per-snap state_bits atomic** — defense against torn state during concurrent ops.
- **kcopyd error retry policy** — defense against transient-fail invalidating snapshot.
- **Snapshot-full graceful invalidation** — defense against subsequent silent-failure.

