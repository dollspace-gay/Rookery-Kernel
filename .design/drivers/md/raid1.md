# Tier-3: drivers/md/raid1.c — RAID-1 (md raid1 driver: pure mirroring + per-bio multi-mirror submission)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/00-overview.md
upstream-paths:
  - drivers/md/raid1.c
  - drivers/md/raid1.h
-->

## Summary

RAID-1 is pure mirroring: every write goes to all N mirror disks; reads come from any healthy mirror. Per-bio: write splits into per-mirror clones (one per active disk); read picks best mirror via read-balance heuristic. Survives N-1 failures. Linux md raid1 also supports write-mostly disks (used only for writes, e.g., remote backup mirror) and write-behind (returning to userspace before all writes complete on slow mirrors). Bitmap-based intent log enables fast resync after power-loss (only changed-stripes resync vs full-array).

This Tier-3 covers `drivers/md/raid1.c` (~3516 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct r1conf` | per-array config | `drivers::md::raid1::R1Conf` |
| `struct r1bio` | per-bio state | `R1Bio` |
| `struct raid1_info` | per-mirror state | `Raid1Info` |
| `raid1_make_request(mddev, bio)` | per-bio dispatch | `Raid1::make_request` |
| `raid1_end_read_request(bio)` | read completion | `Raid1::end_read_request` |
| `raid1_end_write_request(bio)` | write completion | `Raid1::end_write_request` |
| `read_balance(conf, r1_bio, &max_sectors)` | per-read read-balance | `Raid1::read_balance` |
| `raid1_run(mddev)` | array startup | `Raid1::run` |
| `raid1_stop(mddev)` | array shutdown | `Raid1::stop` |
| `raid1_status(seq_file, mddev)` | proc display | `Raid1::status` |
| `raid1_resize(mddev, sectors)` | array resize | `Raid1::resize` |
| `raid1_takeover(mddev)` | takeover from raid0/raid5 | `Raid1::takeover` |
| `wait_barrier(conf, sector_nr, ...)` / `allow_barrier(conf, ...)` | resync-vs-IO sync | `Raid1::wait_barrier` / `_allow_barrier` |
| `raid1_check_read_balance(conf, r1_bio)` | per-pin read-balance check | `Raid1::check_read_balance` |
| `submit_bio_wait_atomic(...)` | per-bio submit | `Raid1::submit_bio` |
| `freeze_array(conf, ...)` / `unfreeze_array(conf)` | quiesce IO during change | `Raid1::freeze_array` / `_unfreeze_array` |
| `bio_copy_data(dest, src)` | copy bio pages | shared |
| `narrow_write_error(...)` | minimize-error-write recovery | `Raid1::narrow_write_error` |
| `setup_conf(mddev)` | per-array initial setup | `Raid1::setup_conf` |
| `raid1_handle_discard(...)` | discard propagation | `Raid1::handle_discard` |

## Compatibility contract

REQ-1: Per-array `r1conf`:
- `mddev` (back-ref).
- `mirrors` (KVec<Raid1Info>; one per disk).
- `raid_disks` (count).
- `barrier_lock` (resync sync).
- `nr_pending[BARRIER_BUCKETS_NR]` (per-bucket in-flight bio count).
- `nr_waiting[BARRIER_BUCKETS_NR]` (per-bucket waiting count).
- `pending_bio_list` (deferred bios during resync).
- `device_lock` (mirrors[] mutation lock).
- `r1bio_pool` (mempool for r1_bio).
- `r1buf_pool` (mempool for write-behind buffers).

REQ-2: Per-mirror `raid1_info`:
- `rdev` (md_rdev pointer).
- `head_position` (sector cursor for read-balance).
- `next_seq_sect` (next-sequential sector hint).
- `seq_start` (sequence start).

REQ-3: Per-r1_bio:
- `master_bio` (original bio).
- `mddev` (back-ref).
- `sector` (raid sector).
- `sectors` (length).
- `state` (R1BIO_*_FLAG).
- `read_disk` (which mirror was read for read-bio).
- `bios[N_RAID1_DISK]` (per-disk clones).
- `behind_master_bio` (write-behind copy).

REQ-4: Per-bio write flow:
1. wait_barrier(conf, bio.bi_iter.bi_sector).
2. Allocate r1_bio + per-mirror cloned bios.
3. For each mirror in 0..raid_disks:
   - If !mirror.rdev: skip.
   - If mirror.rdev.flags & FAULTY: skip.
   - clone master_bio to mirror's bdev.
   - Submit.
4. End-write per-bio: count completions; mark R1BIO_Uptodate when last completes.
5. bio_endio master_bio.

REQ-5: Per-bio read flow:
1. read_balance: pick best mirror.
2. Allocate r1_bio + cloned bio for chosen mirror.
3. Submit.
4. End-read: if uptodate: bio_endio master_bio.
5. On read-error: retry with another mirror.

REQ-6: Read-balance heuristics:
- Avoid in-resync-area mirrors.
- Prefer near head_position (sequential affinity).
- Prefer mirrors not currently busy.
- Prefer mirrors with short queue.
- Round-robin on tie.

REQ-7: Write-behind support (per-disk Write-Mostly):
- Mirror marked Write-Mostly.
- Per-write to such mirror: bio_copy_data into per-write-behind buffer.
- bio_endio master_bio after primary writes complete (don't wait for write-mostly).
- Background completion of write-mostly bio_endio releases buffer.

REQ-8: Bitmap-based intent log:
- Per-block dirty-bitmap on metadata-bdev.
- Pre-write: set bit in bitmap.
- Post-write: clear bit (after all mirror writes complete).
- Power-loss resync: only stripes with set bitmap-bits resyncd.

REQ-9: Per-mirror failure:
- Write-error: mark mirror Faulty.
- Read-error: retry on another mirror; if all fail: bio_io_error.
- narrow_write_error: try writing in smaller chunks to identify which sector specifically failed.

REQ-10: Resync (sync_request):
- Per-stripe-row reads from one mirror; writes to all others (rebuild after replace) OR compares (consistency check).
- Speed-limited via sysfs.

REQ-11: Per-array barrier buckets:
- `BARRIER_BUCKETS_NR = 1024` per-array buckets (hash by sector).
- Per-bucket nr_pending/nr_waiting reduces lock contention during resync.

REQ-12: Discard / Trim propagation:
- Per-mirror discard issued; coordinated with bitmap.

## Acceptance Criteria

- [ ] AC-1: Create raid1 with 2 disks: bio writes go to both; reads come from one.
- [ ] AC-2: Disk-fail: pull one disk; reads continue from other; writes succeed on remaining.
- [ ] AC-3: Read-balance: 2-mirror array; concurrent reads spread between disks; per-disk-IO balanced.
- [ ] AC-4: Bitmap intent log: kill -9 mid-write; reboot; bitmap-based resync only re-syncs changed stripes.
- [ ] AC-5: Write-mostly + write-behind: 1 local + 1 write-mostly remote; writes return after local complete.
- [ ] AC-6: Resync after rebuild: replace failed disk; mdadm --add; recovery copies; new disk in-sync.
- [ ] AC-7: 8-disk raid1: heavy write stress; all 8 disks consistent; periodic verify-pass passes.
- [ ] AC-8: Discard propagation: 1GB discard; both mirrors propagate.
- [ ] AC-9: narrow_write_error: simulate per-sector write fail; recovery identifies failing sector + remaps.
- [ ] AC-10: mdadm test suite raid1 passes.

## Architecture

`R1Conf`:

```
struct R1Conf {
  mddev: KArc<MdDev>,
  mirrors: KVec<Raid1Info>,
  raid_disks: u32,
  barrier_lock: SpinLock<()>,
  nr_pending: KBox<[AtomicI32; BARRIER_BUCKETS_NR]>,
  nr_waiting: KBox<[AtomicI32; BARRIER_BUCKETS_NR]>,
  pending_bio_list: ListHead,
  device_lock: SpinLock<()>,
  r1bio_pool: KArc<MempoolR1Bio>,
  r1buf_pool: KArc<MempoolBio>,
  cluster_sync_low: u64,
  cluster_sync_high: u64,
  ...
}

struct Raid1Info {
  rdev: KArc<MdRdev>,
  head_position: u64,
  next_seq_sect: u64,
  seq_start: u64,
}

struct R1Bio {
  master_bio: KArc<Bio>,
  mddev: KArc<MdDev>,
  sector: u64,
  sectors: u32,
  state: AtomicU64,                            // R1BIO_*_FLAG bits
  read_disk: u32,
  remaining: AtomicI32,                         // pending child bios
  bios: KVec<Option<KArc<Bio>>>,                // per-disk clones
  behind_master_bio: Option<KArc<Bio>>,
}
```

`Raid1::make_request(mddev, bio)`:
1. wait_barrier(conf, bio.bi_iter.bi_sector, ...).
2. r1_bio := mempool_alloc(r1bio_pool).
3. r1_bio.master_bio = bio.
4. r1_bio.sector = bio.bi_iter.bi_sector.
5. If bio.bi_op == REQ_OP_WRITE:
   - For mirror in 0..raid_disks:
     - If mirror.rdev faulty/missing: skip.
     - clone bio targeting mirror.rdev.bdev.
     - r1_bio.bios[mirror] = clone.
     - Submit.
   - r1_bio.remaining = active_mirror_count.
6. Else (READ):
   - mirror := read_balance(conf, r1_bio, &max_sectors).
   - clone bio targeting mirror.rdev.bdev.
   - r1_bio.bios[mirror] = clone.
   - Submit.

`Raid1::end_write_request(bio)`:
1. r1_bio := bio.bi_private.
2. If bio.bi_status: mark mirror failure.
3. atomic_dec(r1_bio.remaining).
4. If remaining == 0:
   - Bitmap clear bits.
   - bio_endio master_bio.
   - mempool_free(r1_bio).

`Raid1::read_balance(conf, r1_bio, &max_sectors)`:
1. best_disk := -1; best_dist := u64::MAX.
2. For each mirror in 0..raid_disks:
   - If mirror.rdev faulty/in-resync: skip.
   - dist := abs(mirror.head_position - r1_bio.sector).
   - If dist < best_dist OR mirror has shorter queue: best = mirror; best_dist = dist.
3. Return best_disk.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mirror_count_bounded` | OOB | per-array raid_disks ≤ MAX_RAID1_DISKS; defense against OOB walk. |
| `r1_bio_remaining_no_underflow` | INVARIANT | remaining ≥ 0; defense against double-decrement. |
| `barrier_bucket_isolation` | INVARIANT | per-bucket nr_pending tracks only that bucket's bios. |
| `read_disk_chosen_validated` | INVARIANT | read_balance returned mirror is non-faulty + in-sync. |
| `write_remaining_count_match` | INVARIANT | r1_bio.remaining == count of submitted child-bios at submit time. |

### Layer 2: TLA+

`drivers/md/raid1_write.tla`:
- Per-bio state ∈ {Pending, InFlight(N), Completed(success_count), Failed}.
- Properties:
  - `safety_complete_when_all_acked` — Completed when all child-bios acked.
  - `safety_failure_if_all_fail` — Failed if all child-bios failed.
  - `liveness_pending_eventually_completes` — every Pending eventually Completed/Failed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Raid1::make_request` post: per-disk-clones submitted (write) or single-mirror-clone (read) | `Raid1::make_request` |
| `Raid1::end_write_request` post: remaining decremented; on zero: bitmap-clear + master_bio endio | `Raid1::end_write_request` |
| `Raid1::read_balance` post: returned mirror is healthy + in-sync | `Raid1::read_balance` |
| Per-array nr_pending balanced across barrier buckets | invariants on barrier ops |

### Layer 4: Verus/Creusot functional

`Per-bio write: data appears on every healthy mirror; per-bio read: data matches what was written` semantic equivalence: per-bio the eventual state matches single-disk semantics for reads + multi-disk consistency for writes.

## Hardening

(Inherits row-1 features from `drivers/md/00-overview.md` § Hardening.)

raid1-specific reinforcement:

- **Per-mirror Faulty-flag honored at read** — defense against reading from known-bad disk.
- **Per-bio remaining count atomic** — defense against double-endio.
- **wait_barrier per-bucket** — defense against IO racing resync; reduces contention vs single-lock.
- **Bitmap intent log** — defense against full-resync-after-power-loss requiring O(disk-size) IO.
- **Write-behind buffer ref-count** — defense against premature buffer-free during write-mostly.
- **narrow_write_error per-sector recovery** — defense against single-sector failure causing whole-mirror-failure.
- **Per-bio mempool capped** — defense against allocation-failure under load.
- **Resync rate-limit via sysfs** — defense against resync hogging IO bandwidth.
- **Per-discard atomicity** — defense against partial-discard leaving ghost-data on subset of mirrors.
- **freeze_array/unfreeze_array discipline** — defense against config-change during active IO.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- raid10 (covered in `raid10.md` Tier-3)
- raid5/raid6 (covered separately)
- raid0 (linear striping; covered separately)
- md-core (covered in `md-core.md` future Tier-3)
- mdadm userspace
- Implementation code
