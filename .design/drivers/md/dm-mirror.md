# Tier-3: drivers/md/dm-raid1.c — dm-mirror (legacy mirror target with dm-log dirty region tracking + recovery)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-raid1.c
  - drivers/md/dm-log.c
  - drivers/md/dm-region-hash.c
  - drivers/md/dm-bio-record.h
-->

## Summary

dm-mirror (`mirror` target) is the legacy device-mapper mirror implementation (separate from md raid1 — used via dmsetup before LVM mirrored-LVs migrated to dm-raid integration). Per-region dirty-bitmap (dm-log) tracks which regions need resync on recovery; per-bio splits across all mirrors with read-balancing among healthy mirrors. dm-log abstracts dirty-state persistence (core = in-memory, disk = on-disk). dm-region-hash hashes per-region active state for in-flight tracking. Used as legacy backend for LVM mirrored volumes.

This Tier-3 covers `drivers/md/dm-raid1.c` (~1531 lines) + `dm-log.c` (~916 lines) + `dm-region-hash.c`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mirror_set` | per-target mirror config | `drivers::md::mirror::MirrorSet` |
| `struct mirror` | per-mirror disk state | `Mirror` |
| `struct dm_dirty_log` | dm-log abstraction | `DmDirtyLog` |
| `struct dm_dirty_log_type` | log-type vtable | `DmDirtyLogType` |
| `struct dm_region_hash` | per-region hash | `DmRegionHash` |
| `mirror_ctr(target, argc, argv)` | mirror target ctor | `Mirror::ctr` |
| `mirror_dtr(target)` | dtor | `Mirror::dtr` |
| `mirror_map(target, bio)` | per-bio dispatch | `Mirror::map_bio` |
| `mirror_endio(target, bio, err)` | bio completion | `Mirror::endio` |
| `do_mirror(work)` | worker thread main | `Mirror::do_mirror` |
| `do_writes(ms, &writes)` | per-write list dispatch | `Mirror::do_writes` |
| `do_reads(ms, &reads)` | per-read list dispatch | `Mirror::do_reads` |
| `do_recovery(ms)` | per-region resync | `Mirror::do_recovery` |
| `dm_dirty_log_create(name, ti, ...)` (dm-log.c) | log-create | `DmDirtyLog::create` |
| `dm_dirty_log_destroy(log)` | log-destroy | `DmDirtyLog::destroy` |
| `dm_dirty_log_get_type(name)` | per-log-type lookup | `DmDirtyLogType::lookup` |
| `dm_dirty_log_register_type(...)` | per-log-type register | `DmDirtyLogType::register_type` |
| `dm_rh_init(rh, ...)` (dm-region-hash.c) | region-hash init | `DmRegionHash::init` |
| `dm_rh_get_state(rh, region)` | per-region state query | `DmRegionHash::get_state` |
| `dm_rh_inc_pending(rh, &bio_list)` | per-region inc-pending | `DmRegionHash::inc_pending` |
| `dm_rh_dec(rh, region)` | per-region dec | `DmRegionHash::dec` |
| `recover(ms, reg)` | per-region recovery | `Mirror::recover` |
| `read_balance(ms, bio)` | per-bio read-balance | `Mirror::read_balance` |
| `bio_record(...)` (dm-bio-record.h) | per-bio state save/restore | `BioRecord::save` / `_restore` |

## Compatibility contract

REQ-1: Per-target `mirror_set`:
- `ti` (back-ref to dm_target).
- `nr_mirrors`.
- `mirror[N]` (per-disk Mirror struct).
- `region_size` / `region_shift` (sectors per region; default 512KB).
- `nr_regions` (count).
- `rh` (DmRegionHash).
- `log` (DmDirtyLog).
- `kmirrord_wq` (workqueue).
- `kcopyd_client` (for resync).
- `pages_pool` (mempool for read pages).
- `error_type` (DM_RAID1_HANDLE_ERRORS / FAULTY).
- `default_mirror` (read-balance root).

REQ-2: Per-mirror `mirror`:
- `ms` (back-ref).
- `error_count`.
- `error_type`.
- `dev` (DmDev).
- `offset` (per-mirror sector offset on bdev).

REQ-3: Per-region state:
- DM_RH_CLEAN: in sync; reads/writes proceed normally.
- DM_RH_DIRTY: write in progress on this region; subsequent reads need to wait.
- DM_RH_NOSYNC: not yet synced (e.g., new region after disk-add); mirror.dev hidden from reads.
- DM_RH_RECOVERING: per-region copy in flight.

REQ-4: Per-bio dispatch (`mirror_map`):
1. ms := target.private.
2. region := bio.bi_iter.bi_sector >> region_shift.
3. If bio.bi_op == REQ_OP_READ:
   - mirror := read_balance(ms, bio).
   - bio.bi_iter.bi_sector += mirror.offset; bio.bi_bdev = mirror.dev.bdev.
   - return DM_MAPIO_REMAPPED.
4. Else (WRITE):
   - dm_rh_inc_pending(rh, &bio_list).
   - Mark region DIRTY in log if not already.
   - Queue to worker via bio_list_add(&ms.writes, bio).
   - Return DM_MAPIO_SUBMITTED.

REQ-5: Per-write worker (`do_writes`):
1. For each bio in &ms.writes:
   - Per-mirror: clone bio; submit to mirror.dev.
2. On all-mirrors complete:
   - Per-region: dec_pending; if last: dirty-log clear region bit.
   - bio_endio.

REQ-6: Read-balance (`read_balance`):
1. Default mirror first; round-robin if multi.
2. Skip faulted mirrors.
3. Skip mirrors in NOSYNC state (data not yet copied).

REQ-7: Per-recovery (`do_recovery`):
1. Walk dm-log for DIRTY/NOSYNC regions.
2. For each: dm_kcopyd_copy from default_mirror to others.
3. On completion: log clear region bit; mark CLEAN.

REQ-8: dm-log types:
- `core`: in-memory only; lost on reboot.
- `disk`: persistent on dedicated bdev.
- `clustered_*`: shared-storage cluster-aware.
- `userspace`: userspace daemon (dmeventd).

REQ-9: dm-region-hash:
- Per-region hash bucket lookup.
- Per-bucket list of active regions.
- Per-region in-flight bio count + state.

REQ-10: Per-mirror failure:
- Write-error: mark mirror FAULTY; future reads/writes skip.
- Read-error: retry on another mirror; if all fail: bio_io_error.

REQ-11: Per-pin recovery:
- Replace failed mirror: mdadm --add or dmsetup message.
- Recovery scans NOSYNC regions; copies; updates log.

REQ-12: dm-mirror vs md-raid1:
- dm-mirror: legacy; uses dm-log + dm-region-hash; LVM2 backend pre-2010.
- md-raid1: modern; native md; preferred for new deployments.
- dm-raid (drivers/md/dm-raid.c) bridges md-raid backends to dm framework.

## Acceptance Criteria

- [ ] AC-1: Create mirror with 2 disks + dm-log core: bios writes go to both; reads load-balanced.
- [ ] AC-2: Disk-fail: pull one disk; reads continue from other; writes succeed on remaining.
- [ ] AC-3: Persistent log: power-off; reboot; only dirty regions resync (vs full-array).
- [ ] AC-4: Recovery: replace failed disk + dmsetup message; per-region copy completes.
- [ ] AC-5: Read-balance: 2-mirror; concurrent reads spread between disks.
- [ ] AC-6: 1M-region stress: heavy random writes; per-region tracked correctly.
- [ ] AC-7: Cluster-log: shared-storage cluster; cross-node mirror writes coordinated.
- [ ] AC-8: dm-mirror legacy LVM2 mirror activate works.
- [ ] AC-9: dm-raid1 xfstests pass.

## Architecture

`MirrorSet`:

```
struct MirrorSet {
  ti: KArc<DmTarget>,
  nr_mirrors: u32,
  mirror: KVec<Mirror>,
  region_size: u32,
  region_shift: u32,
  nr_regions: u64,
  rh: KArc<DmRegionHash>,
  log: KArc<DmDirtyLog>,
  kmirrord_wq: KArc<Workqueue>,
  kcopyd_client: KArc<DmKcopyd>,
  pages_pool: KArc<MempoolPages>,
  error_type: u32,
  default_mirror: AtomicI32,
  reads: BioList,
  writes: BioList,
  failures: BioList,
  ...
}

struct Mirror {
  ms: KWeak<MirrorSet>,
  error_count: AtomicI32,
  error_type: u32,
  dev: KArc<DmDev>,
  offset: u64,
}

struct DmDirtyLog {
  type_: KArc<DmDirtyLogType>,
  context: KBox<dyn DmDirtyLogContext>,
}

struct DmDirtyLogType {
  name: KStr,
  ctr: DmDirtyLogCtrFn,
  dtr: DmDirtyLogDtrFn,
  is_clean: DmDirtyLogIsCleanFn,
  in_sync: DmDirtyLogInSyncFn,
  flush: DmDirtyLogFlushFn,
  mark_region: DmDirtyLogMarkRegionFn,
  clear_region: DmDirtyLogClearRegionFn,
  get_resync_work: DmDirtyLogGetResyncFn,
  ...
}
```

`Mirror::map_bio(target, bio)`:
1. ms := target.private.
2. If bio.bi_op == REQ_OP_READ:
   - region := bio.bi_iter.bi_sector >> ms.region_shift.
   - state := dm_rh_get_state(ms.rh, region, true).
   - If state == DM_RH_NOSYNC: queue for recovery; return SUBMITTED.
   - mirror := read_balance(ms, bio).
   - bio.bi_iter.bi_sector += mirror.offset; bio.bi_bdev = mirror.dev.bdev.
   - return DM_MAPIO_REMAPPED.
3. Else (WRITE):
   - dm_rh_inc_pending(ms.rh, bio_list_one(bio)).
   - mark log region DIRTY.
   - bio_list_add(&ms.writes, bio).
   - queue_work(ms.kmirrord_wq, &ms.kmirrord_work).
   - return DM_MAPIO_SUBMITTED.

`Mirror::do_writes(ms, writes)`:
1. For each bio in writes:
   - Per-mirror: clone bio targeting mirror; submit.
2. Track per-mirror completion via per-bio remaining count.
3. On all complete:
   - dec_pending region.
   - dirty-log clear region bit.
   - bio_endio master_bio.

`Mirror::recover(ms, reg)`:
1. dm_kcopyd_copy from default_mirror to all-other-mirrors for region.
2. On completion: log.clear_region(region); rh state CLEAN.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mirror_count_bounded` | OOB | per-set nr_mirrors ≤ MAX_DM_MIRRORS; defense against OOB walk. |
| `region_idx_bounded` | OOB | per-region idx < nr_regions; defense against OOB log access. |
| `dm_rh_state_valid` | INVARIANT | per-region state ∈ {CLEAN, DIRTY, NOSYNC, RECOVERING}. |
| `log_persistence_atomic` | INVARIANT | dirty-log marks atomic via per-log-type flush(). |
| `mirror_offset_within_dev` | INVARIANT | per-mirror.offset + bio-len ≤ mirror.dev capacity. |

### Layer 2: TLA+

`drivers/md/mirror_region_state.tla`:
- Per-region state ∈ {Clean, Dirty, NoSync, Recovering}.
- Properties:
  - `safety_dirty_persists_until_clean` — Dirty → Clean only after all writes complete + log flushed.
  - `safety_recovery_completes_to_clean` — Recovering → Clean after kcopyd success.
  - `liveness_pending_eventually_clean` — every Dirty eventually Clean.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mirror::map_bio` post: bio remapped (read healthy mirror) OR queued (write/recovery) | `Mirror::map_bio` |
| `Mirror::do_writes` post: per-mirror clones submitted; log marked DIRTY | `Mirror::do_writes` |
| `Mirror::recover` post: kcopyd copy submitted; on-complete log cleared | `Mirror::recover` |
| Per-region in_flight count balanced across inc/dec | invariants on do_writes |

### Layer 4: Verus/Creusot functional

`Per-bio: write completes on all healthy mirrors; per-bio: read returns same data as written` semantic equivalence: per-bio per-region the eventual state matches single-disk semantics + multi-disk consistency.

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-mirror-specific reinforcement:

- **Per-region dirty-log persisted before write-ack** — defense against post-crash silent data divergence.
- **Per-mirror error_count cap** — defense against transient-error flapping.
- **Per-region in-flight tracking** — defense against UAF on rapid region-state-change.
- **Read-balance skips faulty mirrors** — defense against reading bad data.
- **Per-region recovery before allowing read on NOSYNC** — defense against reading uninitialized.
- **Per-log persistent commit synchronous** — defense against pre-crash unflushed log losing dirty state.
- **dm_kcopyd async-copy bounded** — defense against unbounded outstanding copies.
- **Per-mirror.offset validated against dev capacity** — defense against OOB device-write.
- **Cluster-log coordination** — defense against split-brain.

## Grsecurity/PaX-style Reinforcement

Rationale: dm-mirror's recovery / kcopyd path reads from one mirror leg and writes to another to bring sync regions current. A corrupted per-region recovery buffer or mis-counted leg refcount produces silent data substitution between mirror legs, including substituting attacker-controlled content during recovery. The reinforcement below restates baseline PaX/grsec coverage applied to `drivers/md/dm-raid1.c` (+ `dm-region-hash.c`) plus dm-mirror-specific reinforcement.

Baseline (cross-ref `drivers/md/dm-core.md` § Hardening):
- **PAX_USERCOPY**: status output via dm-core whitelisted slab; per-leg ctr params whitelisted.
- **PAX_KERNEXEC**: per-`dm_target_type` vtable + per-`dm_dirty_log_type` vtable (`core` / `disk` / `clustered_disk`) all `__ro_after_init`.
- **PAX_RANDKSTACK**: per-bio entry to `mirror_map` / `mirror_end_io` / `do_recovery` re-randomises stack.
- **PAX_REFCOUNT**: per-leg refcount, per-region cell refcount, kcopyd job refcount saturating — see mirror-specific bullet.
- **PAX_MEMORY_SANITIZE**: per-region recovery buffer zeroed on completion; per-mirror context zeroed on dtr.
- **PAX_UDEREF**: no direct user pointer.
- **PAX_RAP / kCFI**: ctr/dtr/map/end_io/presuspend/postsuspend/preresume/resume/status/message + per-dirty-log ops + kcopyd done callback kCFI-tagged.
- **GRKERNSEC_HIDESYM**: per-mirror addr, per-region addr, kcopyd job addr never rendered; `%pK`.
- **GRKERNSEC_DMESG**: mirror-error / recovery-progress warnings ratelimited; CAP_SYSLOG to read full progress.

dm-mirror-specific reinforcement:
- **Per-disk PAX_REFCOUNT** — `struct mirror.dev` count + per-region `dm_region.pending` count + `kcopyd_job.ref` saturating; defends double-release on requeue during recovery.
- **Recovery PAX_USERCOPY** — kcopyd reads a region into a slab-allocated buffer carrying the USERCOPY whitelist tag (even though no user copy happens) so a future evolution that exports recovery buffers (e.g., via status/debug) can't accidentally bypass the slab/USERCOPY whitelist.
- **PAX_MEMORY_SANITIZE on region buffer** — kcopyd buffer pages zeroed before return to free pool; defends source-leg data residue.
- **Per-mirror.offset validated against `bdev_nr_sectors`** — already row-2; extended with overflow-checked `add_safe` against `sector_t`.
- **Per-region recovery before read on NOSYNC** — row-2; extended with grsec audit log entry per region marked SYNC.
- **Per-log persistent commit synchronous** — row-2; extended with checksum (CRC32C) over the per-log header on every commit.
- **Cluster-log coordination** — extends row-2 (split-brain defense) with a quorum check requiring 2-of-N node ack before declaring a region SYNC in clustered_disk mode.
- **kcopyd outstanding-job cap** — `dm_kcopyd_client` capped at `max(64, num_legs * 16)` outstanding jobs; defends recovery-storm OOM.
- **Per-mirror leg checksum on read-verify** — when CONFIG_DM_MIRROR_VERIFY is set, recovered region reads verify CRC32C against a per-region signature; mismatch escalates leg to failed.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- md-raid1 (covered in `raid1.md` Tier-3; modern alternative)
- dm-raid (covered in `dm-raid.md` future Tier-3; bridge to md backends)
- LVM2 userspace
- Implementation code
