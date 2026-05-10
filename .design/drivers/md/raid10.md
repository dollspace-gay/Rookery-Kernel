# Tier-3: drivers/md/raid10.c — RAID-10 (md raid10 driver: striping + mirroring + near/far/offset layouts)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/00-overview.md
upstream-paths:
  - drivers/md/raid10.c
  - drivers/md/raid10.h
  - drivers/md/md.c (md_personality registration)
-->

## Summary

RAID-10 (stripe-of-mirrors) combines RAID-1 mirroring (data duplicated to N copies) with RAID-0 striping (data distributed across drives). Linux md raid10 supports three layouts: near (default; mirror copies adjacent), far (mirror copies far apart for read-throughput), offset (hybrid with rotating offset). Per-bio: read from one of N copies (round-robin or nearest); write to all N copies. Resync rebuilds from healthy copies. Critical use case: high-performance database/VM storage where mirror gives both read-bandwidth + survive-N-1-failures protection.

This Tier-3 covers `drivers/md/raid10.c` (~5090 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct r10conf` | per-array config | `drivers::md::raid10::R10Conf` |
| `struct mirror_info` | per-mirror-disk state | `MirrorInfo` |
| `struct r10bio` | per-bio state | `R10Bio` |
| `struct geom` | layout geometry | `Geom` |
| `raid10_make_request(mddev, bio)` | per-bio dispatch | `Raid10::make_request` |
| `raid10_end_read_request(bio)` | read completion | `Raid10::end_read_request` |
| `raid10_end_write_request(bio)` | write completion | `Raid10::end_write_request` |
| `read_balance(conf, r10_bio, &max_sectors)` | per-read read-balance | `Raid10::read_balance` |
| `raid10_run(mddev)` | array startup | `Raid10::run` |
| `raid10_stop(mddev)` | array shutdown | `Raid10::stop` |
| `raid10_status(seq_file, mddev)` | proc display | `Raid10::status` |
| `raid10_resize(mddev, sectors)` | array resize | `Raid10::resize` |
| `raid10_takeover(mddev)` | takeover from raid0 etc. | `Raid10::takeover` |
| `raid10_check_reshape(mddev)` | reshape validation | `Raid10::check_reshape` |
| `raid10_start_reshape(mddev)` | reshape begin | `Raid10::start_reshape` |
| `__raid10_find_mirror(conf, sector)` | sector → device-list | `Raid10::find_mirror` |
| `raid10_handle_discard(...)` | discard propagation | `Raid10::handle_discard` |
| `raid10_size(...)` | array size in sectors | `Raid10::array_size` |
| `sync_request(mddev, sector_nr, ...)` | resync-bio submit | `Raid10::sync_request` |
| `recovery_request(mddev, sector_nr, ...)` | recovery-bio submit | `Raid10::recovery_request` |
| `wait_barrier(conf, ...)` / `allow_barrier(conf)` | per-array bio-barrier sync | `Raid10::wait_barrier` / `_allow_barrier` |
| `enough(conf, ...)` | sufficient-disks check | `Raid10::enough` |

## Compatibility contract

REQ-1: Per-array `r10conf`:
- `mddev` (back-ref).
- `mirrors` (KVec<MirrorInfo>; one per disk in array).
- `geo` (Geom: layout description).
- `prev` (Geom: previous geometry during reshape).
- `copies` (number of copies per data; e.g., 2).
- `chunk_sectors` (stripe-chunk size).
- `barrier_lock` (synchronization for resync-vs-IO).
- `nr_pending` / `nr_waiting` / `nr_queued` (in-flight bio counts).
- `device_lock` (mirrors[] mutation lock).
- `tmppage` (per-array scratch page for resync).
- `resync_lock` (resync-vs-IO arbitration).

REQ-2: Geometry (`geom`):
- `raid_disks`: total disks.
- `near_copies`: copies stored adjacent (default 2).
- `far_copies`: copies stored far apart (rotating).
- `chunk_sectors`.
- `stride`: layout stride.

REQ-3: Per-MirrorInfo:
- `rdev` (md_rdev pointer; per-disk state).
- `head_position` (sector cursor for read-balance).

REQ-4: Per-R10Bio (per-host-bio):
- `master_bio` (original bio).
- `mddev` (back-ref).
- `sector` (raid sector).
- `sectors` (length).
- `state` (R10BIO_*_FLAG: R10BIO_Uptodate, R10BIO_IsReshape, R10BIO_FailFast, etc.).
- `devs[copies]` (per-copy bio + slot).

REQ-5: Per-bio write flow:
1. raid10_make_request.
2. Wait for barriers.
3. Allocate r10_bio + per-copy device-bio.
4. For each copy: clone bio targeting mirror[i].rdev with sector translated.
5. Submit per-copy bios concurrently.
6. End-write: count completions; mark R10BIO_Uptodate when last completes.
7. bio_endio master_bio.

REQ-6: Per-bio read flow:
1. read_balance(conf, r10_bio): pick best mirror for this sector based on head-position + recent-history.
2. Allocate per-mirror bio.
3. Submit.
4. On read-fail: retry with another mirror.
5. End-read: if uptodate: bio_endio master_bio.

REQ-7: Per-mirror failure:
- Write-error: mark mirror Faulty; future writes skip.
- Read-error: retry on another mirror; if all fail: bio_io_error.
- Recovery: when mirror replaced: rebuild via sync_request from healthy mirror.

REQ-8: Resync (consistency check):
- Per-stripe sync_request reads all copies; verifies match; on mismatch: corrected.
- Configurable speed limit via sysfs.

REQ-9: Per-mirror barriers:
- raid10_make_request waits for any in-progress resync to clear.
- Resync respects nr_pending == 0 for safe-window.

REQ-10: Discard / Trim propagation:
- Per-mirror discard issued; on success: track ranges as discarded.

REQ-11: near/far/offset layouts:
- near=N: N adjacent stripes contain copies (default).
- far=N: copies stored at far_dist offsets; benefits read-bandwidth.
- offset: copies offset by chunk; allows pipelined reads.

REQ-12: Reshape (rare):
- raid10_start_reshape morphs near→far layouts; long-running.
- Per-stripe migrate via tmppage.

## Acceptance Criteria

- [ ] AC-1: Create raid10 with 4 disks, near=2: total size = 2× single-disk; bio writes go to 2 mirrors.
- [ ] AC-2: Disk-fail: pull one disk; array stays live; reads continue from other mirror.
- [ ] AC-3: Resync after rebuild: replace failed disk; mdadm --add; recovery_request copies data; new disk in-sync.
- [ ] AC-4: Read balance: 4-disk near=2 array; concurrent reads spread across both copies; verify per-disk-IO balanced.
- [ ] AC-5: Far layout: raid10 with far=2; stripe data spread far apart; read-throughput ~2× sequential.
- [ ] AC-6: Discard propagation: 1GB discard; both mirrors propagate to underlying disks.
- [ ] AC-7: Reshape near=2 → far=2: long-running reshape completes; data preserved.
- [ ] AC-8: 32-disk raid10 stress: heavy random writes; no data corruption.
- [ ] AC-9: Power-fail crash: kill -9 mid-write; reboot; resync detects unmatched copies + corrects.
- [ ] AC-10: mdadm test suite: raid10 tests pass.

## Architecture

`R10Conf`:

```
struct R10Conf {
  mddev: KArc<MdDev>,
  mirrors: KVec<MirrorInfo>,
  geo: Geom,
  prev: Geom,
  copies: u32,
  chunk_sectors: u32,
  near_copies: u32,
  far_copies: u32,
  far_offset: bool,
  barrier_lock: SpinLock<()>,
  nr_pending: AtomicI32,
  nr_waiting: AtomicI32,
  nr_queued: AtomicI32,
  pending_bio_list: ListHead,
  resync_lock: SpinLock<()>,
  tmppage: KArc<Page>,
}

struct MirrorInfo {
  rdev: KArc<MdRdev>,
  head_position: u64,                          // sector cursor for read-balance
  recovery_disabled: i32,
}

struct R10Bio {
  master_bio: KArc<Bio>,
  mddev: KArc<MdDev>,
  sector: u64,
  sectors: u32,
  state: AtomicU64,                             // R10BIO_*_FLAG bits
  devs: [R10BioDev; MAX_R10COPIES],
}

struct R10BioDev {
  bio: Option<KArc<Bio>>,
  devnum: u32,
  addr: u64,                                    // mirror-relative sector
}
```

`Raid10::make_request(mddev, bio)`:
1. wait_barrier(conf, bio).
2. r10_bio := alloc + populate.
3. If bio.bi_op == REQ_OP_WRITE:
   - For each copy in 0..copies:
     - dev := find_mirror(conf, r10_bio.sector, copy_idx).
     - Allocate r10_bio.devs[copy].bio; clone master_bio.
     - Set bi_iter.bi_sector = dev_relative_sector.
     - Set bi_bdev = mirrors[dev].rdev.bdev.
   - Submit all bios.
4. If bio.bi_op == REQ_OP_READ:
   - copy := read_balance(conf, r10_bio, &max_sectors).
   - Allocate r10_bio.devs[copy].bio; clone master_bio.
   - Set bi_iter.bi_sector + bi_bdev.
   - Submit.

`Raid10::end_write_request(bio)`:
1. r10_bio := bio.bi_private.
2. If bio.bi_status: mark r10_bio.devs[i].failure.
3. atomic_dec(r10_bio.devs_pending).
4. If devs_pending == 0:
   - if all-failed: bio_io_error master_bio.
   - else: bio_endio master_bio.
   - mempool_free(r10_bio).

`Raid10::read_balance(conf, r10_bio, &max_sectors)`:
1. For each copy in 0..copies:
   - mirror := find_mirror(conf, r10_bio.sector, copy_idx).
   - If mirror.rdev.flags & FAULTY: skip.
   - dist := abs(mirror.head_position - r10_bio.sector).
   - If dist < best_dist: best = copy_idx; best_dist = dist.
2. Return best.

`Raid10::sync_request(mddev, sector_nr, ...)`:
1. For each stripe-row at sector_nr:
   - For each copy: read mirror's data into tmppage.
   - Compare per-byte; on mismatch: write corrected to faulty copies.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mirror_idx_bounded` | OOB | per-mirror idx < raid_disks; defense against OOB walk. |
| `r10_bio_devs_count` | INVARIANT | r10_bio.devs[] count == copies; defense against torn alloc. |
| `pending_count_no_underflow` | INVARIANT | nr_pending ≥ 0; defense against double-decrement. |
| `barrier_balance` | INVARIANT | per-bio wait_barrier + allow_barrier paired. |
| `head_position_monotonic` | INVARIANT | per-mirror head_position advances with served reads. |

### Layer 2: TLA+

`drivers/md/raid10_write.tla`:
- Per-bio state ∈ {Pending, InFlight(N), Completed(success_count), Failed}.
- Properties:
  - `safety_complete_when_all_acked` — Completed only when all per-copy acks received.
  - `safety_failure_if_any_fail` — Failed if at least one copy failed AND no remaining copy succeeded.
  - `liveness_pending_eventually_completes` — assuming no infinite-fail, Pending eventually Completed/Failed.

`drivers/md/raid10_read_balance.tla`:
- Per-read state: choose best copy.
- Properties:
  - `safety_no_faulty_copy_chosen` — Faulty mirror never selected.
  - `liveness_balanced_distribution` — over time, reads distributed across copies.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Raid10::make_request` post: bios cloned + submitted per-copy (write) OR single mirror (read) | `Raid10::make_request` |
| `Raid10::end_write_request` post: completion-count decremented; on zero: master_bio endio | `Raid10::end_write_request` |
| `Raid10::read_balance` post: returned copy is non-faulty | `Raid10::read_balance` |
| Per-array nr_pending balanced across wait/allow_barrier | invariants on barrier ops |
| `Raid10::sync_request` post: per-row copies match or corrected | `Raid10::sync_request` |

### Layer 4: Verus/Creusot functional

`Per-bio: write to N copies in parallel; per-copy on disk has identical bytes` semantic equivalence: per-write the data on every active mirror is the same; per-read the result matches what was written.

## Hardening

(Inherits row-1 features from `drivers/md/00-overview.md` § Hardening.)

raid10-specific reinforcement:

- **Per-mirror Faulty-flag honored at read** — defense against reading from known-bad disk.
- **Per-bio refcount + completion-count** — defense against double-endio on multi-copy write.
- **wait_barrier + nr_pending** — defense against IO racing resync.
- **Per-mirror head_position update under lock** — defense against torn cursor.
- **Per-array layout validated at run** — defense against malformed near/far/offset config.
- **Resync mismatch correction logged** — defense against silent corruption.
- **Per-stripe write-failure tolerance** — defense against single-failed-mirror taking down array.
- **Per-bio mempool sized for max-concurrency** — defense against allocation-failure under load.
- **Reshape progress journaled** — defense against crash mid-reshape leaving inconsistent state.
- **Per-discard propagation atomicity** — defense against partial-discard leaving ghost-data.
- **mdmon userspace integration via sysfs events** — defense against silent failure detection.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- md-core (covered in `md-core.md` future Tier-3)
- raid5 / raid6 (covered separately)
- raid1 (covered separately)
- raid0 (linear striping; covered separately)
- mdadm userspace
- Implementation code
