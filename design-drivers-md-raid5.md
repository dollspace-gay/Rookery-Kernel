---
title: "Tier-3: drivers/md/raid5.c — RAID-5/6 (parity-striped: stripe-cache + xor compute + recovery)"
tags: ["tier-3", "drivers-md", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

RAID-5/6 is parity-striped with N+1 (RAID-5) or N+2 (RAID-6) parity disks per row — survives 1 (R5) or 2 (R6) failures. Per-stripe operations require reading existing data + computing XOR (R5) or Reed-Solomon (R6) parity + writing data + parity. md raid5 implements via per-stripe `stripe_head` cache (default 256 stripe-heads, 4KB each per device per stripe) protected by reference counting + state machine. Per-bio: split into stripe-sized chunks; per-chunk: lookup stripe-head in cache, run state machine through Reading/Computing/Writing transitions. Optional journal device (raid5-cache.c) protects against write-hole on power-loss. PPL (Partial-Parity-Log) is alternative to journal for in-place recovery.

This Tier-3 covers `drivers/md/raid5.c` (~9173 lines) + `raid5-cache.c` (~3187) + `raid5-ppl.c` (~1521).

### Acceptance Criteria

- [ ] AC-1: Create raid5 with 4 disks (3 data + 1 parity): bio writes go to data + parity stripes.
- [ ] AC-2: Disk-fail in RAID-5: pull one disk; reads continue; per-stripe parity-recovery via XOR.
- [ ] AC-3: Disk-fail in RAID-6: pull two disks; reads + writes continue.
- [ ] AC-4: Resync after rebuild: replace failed disk; recovery via XOR (R5) or RS (R6).
- [ ] AC-5: Journal device: power-loss mid-write; on reboot journal replays.
- [ ] AC-6: PPL: power-loss mid-write; on reboot PPL replay-from-metadata.
- [ ] AC-7: 32-disk RAID-6 stress: heavy random writes; full-verify pass.
- [ ] AC-8: Reshape: grow 4-disk RAID-5 to 5-disk; data preserved.
- [ ] AC-9: Stripe-cache pressure: large IO load; cache LRU-evicts inactive; no stripe leak.
- [ ] AC-10: mdadm raid5/raid6 test suite passes.

### Architecture

`R5Conf`:

```
struct R5Conf {
  mddev: KArc<MdDev>,
  disks: KVec<R5Disk>,
  raid_disks: u32,
  chunk_sectors: u32,
  level: u8,
  algorithm: RaidAlgorithm,
  stripe_hashtbl: KBox<[Hlist; NR_STRIPE_HASH_LOCKS]>,
  hash_locks: KBox<[SpinLock<()>; NR_STRIPE_HASH_LOCKS]>,
  active_stripes: AtomicI32,
  inactive_blocked: AtomicI32,
  pending_full_writes: ListHead,
  delayed_list: ListHead,
  bypass_count: u32,
  bypass_threshold: u32,
  cache_size_mutex: Mutex<()>,
  min_nr_stripes: u32,
  max_nr_stripes: u32,
  worker_groups: KArc<[WorkerGroup; NR_WORKER_GROUPS]>,
  log: Option<KArc<R5Log>>,
  ppl_log: Option<KArc<PplLog>>,
  ...
}

struct StripeHead {
  sector: u64,
  pd_idx: i32,
  qd_idx: i32,
  state: AtomicU64,                            // STRIPE_*_FLAG
  count: AtomicI32,                            // refcount
  dev: KVec<R5Dev>,
  lru: ListNode,
  hash: HlistNode,
  ops_request: u32,
  ...
}

struct R5Dev {
  req: Option<KArc<Bio>>,
  vec: BioVec,
  flags: u32,                                   // R5_*_FLAG
  page: KArc<Page>,
  orig_page: Option<KArc<Page>>,
  read_error: u32,
  write_error: u32,
}
```

`Raid5::make_request(mddev, bio)`:
1. wait_barrier.
2. For each chunk in bio:
   - stripe_sector := bio.bi_iter.bi_sector / chunk_sectors.
   - sh := R5Conf::get_active_stripe(conf, stripe_sector).
   - add_stripe_bio(sh, bio, dd_idx, &dd_idx).
   - release_stripe(sh).
3. Async wait for all stripes to complete; bio_endio.

`StripeHead::handle(sh)` (worker):
1. s := analyse_stripe(sh).
2. If s.toread > 0 && !uptodate: handle_stripe_fill (submit reads).
3. Else if s.towrite > 0 && uptodate: handle_stripe_dirtying (submit writes via RMW or RCW).
4. Else if s.locked > 0: wait for in-flight to complete.
5. Else if s.compute > 0: handle_stripe_compute (xor).
6. Etc. through state machine.

`Parity::compute_5(stripe)`:
- New parity = XOR of all data blocks in stripe.
- Async-tx dispatched; sync-xor as fallback.

`Parity::compute_6(stripe)`:
- New P = XOR of data blocks.
- New Q = Galois-Field RS-encoded XOR of data blocks.
- Lookup tables for GF mult.

`R5Log::handle_flush(log, sh)` (journal):
1. Write per-stripe-data + parity to journal first.
2. Mark stripe LOG_RUNNING.
3. After journal-flush complete: queue stripe for actual data-write.
4. After data-write complete: mark stripe LOG_COMMITTED.

### Out of Scope

- raid1 (covered in `raid1.md` Tier-3)
- raid10 (covered in `raid10.md` Tier-3)
- md-core (covered in `md-core.md` future Tier-3)
- async-tx subsystem (covered in `crypto/async_tx.md` future Tier-3)
- mdadm userspace
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct r5conf` | per-array config | `drivers::md::raid5::R5Conf` |
| `struct stripe_head` | per-stripe cache entry | `StripeHead` |
| `struct r5dev` | per-stripe-disk state | `R5Dev` |
| `raid5_make_request(mddev, bio)` | per-bio dispatch | `Raid5::make_request` |
| `add_stripe_bio(sh, bi, dd_idx, ...)` | per-stripe queue bio | `StripeHead::add_bio` |
| `raid5_get_active_stripe(...)` | per-stripe lookup-or-alloc | `R5Conf::get_active_stripe` |
| `release_stripe_plug(...)` / `release_stripe(sh)` | per-stripe ref-decrement | `StripeHead::release` |
| `handle_stripe(sh)` | per-stripe state-machine main | `StripeHead::handle` |
| `handle_stripe_dirtying(...)` | per-stripe write-prep | `StripeHead::handle_dirtying` |
| `handle_stripe_clean_event(...)` | post-write cleanup | `StripeHead::handle_clean_event` |
| `handle_stripe_fill(...)` | per-stripe partial-read fill | `StripeHead::handle_fill` |
| `handle_parity_checks(...)` | resync parity-verify | `StripeHead::handle_parity_checks` |
| `handle_stripe_expansion(...)` | reshape-related stripe handling | `StripeHead::handle_expansion` |
| `analyse_stripe(sh, &s)` | per-stripe state computation | `StripeHead::analyse` |
| `compute_parity6(...)` / `compute_parity5(...)` | RAID-6 / RAID-5 parity compute | `Parity::compute_5` / `_compute_6` |
| `ops_run_compute5(sh, ...)` / `ops_run_compute6(sh, ...)` | async-tx compute dispatch | `StripeHead::run_compute_5` / `_compute_6` |
| `raid5_remove_disk(mddev, rdev)` | per-disk failure handling | `Raid5::remove_disk` |
| `raid5_add_disk(mddev, rdev)` | per-disk add | `Raid5::add_disk` |
| `r5l_log_init(conf, ...)` (raid5-cache.c) | journal initialization | `R5Log::init` |
| `r5l_init_log(...)` | per-journal init | `R5Log::init_log` |
| `r5l_handle_flush_request(...)` | journal-flush request | `R5Log::handle_flush` |
| `r5l_log_run(...)` | journal commit thread | `R5Log::run` |
| `ppl_init_log(conf, ...)` (raid5-ppl.c) | PPL init | `PplLog::init` |
| `ppl_log_stripe(log, sh)` | PPL log-stripe | `PplLog::log_stripe` |
| `setup_conf(mddev)` | array setup | `Raid5::setup_conf` |
| `raid5_resize(mddev, sectors)` | resize | `Raid5::resize` |
| `raid5_takeover(mddev)` | takeover from raid0/4 | `Raid5::takeover` |
| `raid5_check_reshape(mddev)` | reshape check | `Raid5::check_reshape` |
| `raid5_start_reshape(mddev)` | begin reshape | `Raid5::start_reshape` |

### compatibility contract

REQ-1: Per-array `r5conf`:
- `mddev` (back-ref).
- `disks` (KVec<R5Disk>).
- `raid_disks` (count including parity).
- `chunk_sectors` (data-block per disk per stripe; default 64KB).
- `level` (5 or 6).
- `algorithm` (RAID-5: LEFT_ASYMMETRIC / LEFT_SYMMETRIC / RIGHT_*; RAID-6: similar + Q).
- `stripe_hashtbl` (hash for stripe_head lookup).
- `inactive_blocked` (state for cache pressure).
- `active_stripes` / `inactive_stripes` (counter / list).
- `pending_full_writes` / `bypass_count` (write-back queue).
- `delayed_list` (delayed-stripes for batching).
- `temp_inactive_list` (per-CPU per-hash inactive cache).
- `cache_size_mutex` (lock for cache-size adjustment).
- `min_nr_stripes` / `max_nr_stripes` (cache size limits).
- `bypass_threshold` (controls cache bypass).
- `worker_groups` (per-CPU worker thread groups).
- `log` (Optional<KArc<R5Log>>; journal device).
- `ppl_log` (Optional<KArc<PplLog>>; PPL alternative).

REQ-2: Per-stripe `stripe_head`:
- `sector` (raid stripe sector).
- `pd_idx` (parity disk index).
- `qd_idx` (Q disk index for RAID-6, else -1).
- `state` (STRIPE_*_FLAG bitmap: ACTIVE, INSYNC, COMPUTE_RUN, BIODRAIN, OPSREQ_*).
- `count` (refcount).
- `dev[N]` (per-disk R5Dev state).
- `lru` (LRU list-node).
- `hash` (hash list-node).
- `ops_request` (pending async-tx ops bitmap).
- `submitting_thread` (thread submitting bios for this stripe).

REQ-3: Per-stripe-disk `r5dev`:
- `req` (request bio for this disk slot).
- `vec` (bio_vec).
- `flags` (R5_*_FLAG: UPTODATE, LOCKED, OVERWRITE, INSYNC, READRROR).
- `page` (per-disk-slot 4KB page in cache).
- `orig_page` (for reshape; original page if mid-migration).
- `read_error` (count for retry policy).
- `write_error` (similar).

REQ-4: Per-bio dispatch (`raid5_make_request`):
1. wait_barrier.
2. For each chunk in bio:
   - sh := raid5_get_active_stripe(stripe_sector, ...).
   - add_stripe_bio(sh, bio, dd_idx, ...).
3. release_stripe.
4. Worker thread runs handle_stripe → state machine → submits per-disk bios.

REQ-5: Per-stripe state machine (handle_stripe):
- analyse_stripe: compute current state (uptodate count, locked count, missing count).
- Decide next action:
  - need-fill (some disks not uptodate): submit reads.
  - need-compute (parity dirty, missing data): xor compute.
  - need-write (stripe ready): submit data + parity writes.
  - need-recovery (failed disk): recompute via remaining disks.
  - need-replace (replace device): copy from other disks.

REQ-6: Read-modify-write (RMW) vs reconstruct-write (RCW):
- RMW: read existing data + parity → compute new parity from differences → write changed-data + new-parity. Used when modifying few disks.
- RCW: read all other-disks → compute new parity → write data + new-parity. Used when modifying many disks.
- Heuristic: chooses lower-IO approach.

REQ-7: RAID-6 dual parity:
- P parity = XOR of data blocks.
- Q parity = Reed-Solomon (Galois-Field multiplication).
- Recovery: 2-disk failure recoverable from remaining + P + Q.
- compute_parity6 generates Q via lookup-tables.

REQ-8: Async-tx integration:
- Stripe ops dispatched to async-tx subsystem (uses DMA-engine if available, else sync XOR).
- Reduces CPU usage on RAID-5/6.

REQ-9: Stripe cache management:
- `stripe_hashtbl` chained hash (per-bucket list).
- LRU eviction when cache full.
- Per-cache cgroup-limit via memcg.

REQ-10: Journal device (raid5-cache.c):
- Optional dedicated SSD as journal.
- Per-write logged to journal first; data + parity written to main array later (write-back).
- Recovers from write-hole: power-loss mid-write replays from journal.

REQ-11: PPL (Partial-Parity-Log; raid5-ppl.c):
- Alternative to dedicated journal: stores partial-parity in metadata-area.
- Smaller overhead but slower than dedicated journal.

REQ-12: Reshape (rare):
- raid5_start_reshape grows/shrinks number of disks.
- Per-stripe migrate via temporary buffers.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `stripe_count_no_underflow` | INVARIANT | per-stripe count refcount ≥ 0; defense against double-decrement. |
| `stripe_state_valid` | INVARIANT | per-stripe state-flag transitions follow per-FSM legality. |
| `parity_compute_correctness` | INVARIANT | new parity bits computed via XOR/RS match expected. |
| `cache_active_count_balanced` | INVARIANT | active_stripes incremented at get_active_stripe, decremented at release. |
| `journal_committed_before_overwrite` | INVARIANT | journal-replay covers any in-flight stripe at crash. |

### Layer 2: TLA+

`drivers/md/raid5_stripe_fsm.tla`:
- Per-stripe state ∈ {Idle, Reading, Computing, Writing, Done}.
- Properties:
  - `safety_no_overwrite_without_uptodate` — Writing only after Reading+Computing complete.
  - `safety_parity_after_data` — parity write paired with data write.
  - `liveness_pending_eventually_done` — assuming workers run, every Idle eventually Done.

`drivers/md/raid5_journal.tla`:
- Per-stripe journal state ∈ {Pending, JournalLogged, JournalCommitted, DataWritten, Final}.
- Properties:
  - `safety_no_data_write_before_journal_commit` — DataWritten requires JournalCommitted.
  - `safety_replay_idempotent` — re-applying same journal-record produces same result.
  - `liveness_journal_eventually_committed` — every Pending eventually Final.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `R5Conf::get_active_stripe` post: stripe found-or-alloc'd; refcount += 1 | `R5Conf::get_active_stripe` |
| `StripeHead::handle` post: state-machine advances per analyse_stripe outcome | `StripeHead::handle` |
| `Parity::compute_5` post: parity-disk page contains XOR of data-disks | `Parity::compute_5` |
| Per-stripe.dev[i].page valid for i ∈ [0, raid_disks) | invariants on stripe_head |
| `R5Log::handle_flush` post: journal entry persisted before stripe transitions to writing | `R5Log::handle_flush` |

### Layer 4: Verus/Creusot functional

`Per-stripe write: data + parity persisted such that any single disk failure (R5) or two-disk failure (R6) is recoverable from remaining` semantic equivalence: per-stripe the parity invariant holds (P = XOR(data); Q = RS(data)).

### hardening

(Inherits row-1 features from `drivers/md/00-overview.md` § Hardening.)

raid5-specific reinforcement:

- **Per-stripe refcount + cache LRU** — defense against stripe-leak under high IO load.
- **Per-stripe state-machine atomic transitions** — defense against torn FSM during concurrent access.
- **Journal device write-hole-defense** — defense against power-loss mid-stripe-write causing parity-mismatch.
- **PPL alternative for journal-less arrays** — defense against in-place-recovery requiring full-array-read.
- **Per-disk read_error/write_error counters** — defense against transient-fault disabling permanently.
- **RMW vs RCW heuristic** — defense against pathological write-pattern causing IO-amplification.
- **Async-tx + DMA-engine integration** — defense against XOR-CPU-saturation.
- **Per-stripe-cache cap (max_nr_stripes)** — defense against cache-blow-up under sustained pressure.
- **Per-stripe wait-for-in-flight** before reuse — defense against UAF on per-disk-page.
- **Reshape progress journaled** — defense against crash mid-reshape leaving inconsistent state.
- **Per-stripe parity-correctness verify** during sync — defense against silent bit-rot.
- **Per-disk-page sg-list bounded** — defense against pathological large bio.

