---
title: "Tier-3: drivers/md/dm-log-writes.c — write-tracking target (per-bio log to journal device for fsck-injection testing)"
tags: ["tier-3", "drivers-md", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

dm-log-writes is the device-mapper target that logs every per-bio write to a separate journal device — used by xfstests + filesystem-development for testing fsck recovery + crash consistency. Per-write: bio passes through to underlying data device + simultaneously logged to journal-device with metadata (sector, size, flags, timestamp). Userspace can replay journal up to specific marker for crash-injection testing. Distinct from dm-flakey (fault-injection) — dm-log-writes captures the full write history so test can replay any prefix to simulate crash at that point.

This Tier-3 covers `drivers/md/dm-log-writes.c` (~947 lines).

### Acceptance Criteria

- [ ] AC-1: dmsetup create log_writes_test: data-dev /dev/sda1 + logdev /dev/sdb1.
- [ ] AC-2: 100 writes; verify 100 entries on logdev; visible via log-writes-replay.
- [ ] AC-3: Marker insert: dmsetup message "mark fsck_phase_1"; subsequent replay-stops-at-marker works.
- [ ] AC-4: log_only mode: writes to data-dev silenced; only logged.
- [ ] AC-5: xfstests replay-test: ext4-mkfs + write 100MB + replay+fsck consistency check.
- [ ] AC-6: Discard logged: discard 1GB; logdev shows discard entry.
- [ ] AC-7: FLUSH/FUA flags: per-flush bio logged with FLUSH flag.
- [ ] AC-8: Logdev fault: pull logdev cable; data writes continue (logging disabled); subsequent re-attach resumes logging.
- [ ] AC-9: dmsetup status shows entries-count + log-writer kthread state.
- [ ] AC-10: dm-log-writes xfstests pass.

### Architecture

`LogWritesC`:

```
struct LogWritesC {
  ti: KArc<DmTarget>,
  dev: KArc<DmDev>,
  logdev: KArc<DmDev>,
  logging_enabled: AtomicBool,
  log_lock: Mutex<()>,
  next_sector: u64,
  device_supports_discard: bool,
  device_supports_write_zeroes: bool,
  pending_blocks: ListHead,
  log_kthread: KThread,
  wait_completion: WaitQueueHead,
  entries: AtomicU64,
  super_done: AtomicBool,
}

struct PendingBlock {
  list: ListNode,
  flags: u32,
  bio: Option<KArc<Bio>>,
  sector: u64,
  nr_sectors: u64,
  data: Option<KBox<[u8]>>,
}

#[repr(C)]
struct LogWriteEntry {
  magic: u64,
  version: u32,
  flags: u32,
  nr_sectors: u64,
  sector: u64,
  data_len: u32,
  reserved: u32,
}
```

`LogWrites::map_bio(target, bio)`:
1. lc := target.private.
2. If !lc.logging_enabled OR bio.bi_op == REQ_OP_READ:
   - bio.bi_bdev = lc.dev.bdev.
   - return DM_MAPIO_REMAPPED.
3. Else:
   - Allocate PendingBlock.
   - pb.flags = (bio.bi_op-derived flags).
   - pb.sector = bio.bi_iter.bi_sector.
   - pb.nr_sectors = bio_sectors(bio).
   - For FUA/FLUSH: copy bio payload.
   - bio_list_add(&lc.pending_blocks).
   - Schedule lc.log_kthread.
4. Pass-through bio to data dev.

`LogWrites::log_writer(arg)` (kthread):
1. Loop:
   - Wait for pending_blocks.
   - lock lc.log_lock.
   - For each pb in pending_blocks:
     - Compose entry header.
     - submit_bio_sync(write entry header + payload) to logdev at next_sector.
     - next_sector += sectors_used.
     - lc.entries++.
     - Free pb.
   - log_super(lc).
   - unlock.

`LogWrites::log_super(lc)`:
1. Compose LogWriteSuper struct.
2. submit_bio_sync write to logdev sector 0.
3. Mark super_done = true; signal wait_completion.

### Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-flakey (covered in `dm-flakey.md` Tier-3; fault-injection)
- xfstests userspace (separate project)
- log-writes-replay tool (xfsprogs)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct log_writes_c` | per-target config | `drivers::md::log_writes::LogWritesC` |
| `struct pending_block` | per-write pending log entry | `PendingBlock` |
| `struct log_write_entry` | per-bio log entry header | `LogWriteEntry` |
| `struct log_write_super` | journal superblock | `LogWriteSuper` |
| `log_writes_ctr(target, argc, argv)` | target-create | `LogWrites::ctr` |
| `log_writes_dtr(target)` | dtor | `LogWrites::dtr` |
| `log_writes_map(target, bio)` | per-bio dispatch | `LogWrites::map_bio` |
| `log_writes_endio(target, bio, err)` | bio completion | `LogWrites::endio` |
| `log_writes_status(target, type, ...)` | dmsetup status | `LogWrites::status` |
| `log_writes_message(target, argc, argv)` | DM_TARGET_MSG | `LogWrites::message` |
| `log_one_block(lc, block)` | per-bio queue log entry | `LogWrites::log_one_block` |
| `log_super(lc)` | per-flush superblock update | `LogWrites::log_super` |
| `log_kthread(arg)` | per-target log-writer kthread | `LogWrites::kthread` |
| `log_writes_kthread(...)` | journal-write worker | `LogWrites::log_writer` |

### compatibility contract

REQ-1: Per-target `log_writes_c`:
- `dev` (DmDev for data; passes through unchanged).
- `logdev` (DmDev for journal; per-write entries stored here).
- `logging_enabled` (bool: per-target toggle).
- `log_lock` (per-target mutex).
- `next_sector` (next write position on logdev).
- `device_supports_discard` (per-data-dev capability).
- `device_supports_write_zeroes` (per-data-dev capability).
- `pending_blocks` (queue of pending log entries).
- `log_kthread` (per-target kthread).
- `wait_completion` (per-flush waiter).
- `entries` (atomic counter of logged entries).

REQ-2: Per-bio `pending_block`:
- `flags` (LOG_FLUSH_FLAG | LOG_FUA_FLAG | LOG_DISCARD_FLAG | LOG_MARK_FLAG).
- `bio` (original bio).
- `sector` (start sector on data-dev).
- `nr_sectors` (length).
- `data` (KBox of bio-payload-copy for log; for FUA/marker entries).

REQ-3: Per-bio log entry header (`log_write_entry`):
- `magic` (constant marker).
- `flags` (per-entry flags).
- `nr_sectors` (data length).
- `sector` (target sector on data-dev).
- `data_len` (payload bytes following header).
- `version` (entry format version).

REQ-4: Per-target superblock (`log_write_super`):
- `magic` (constant).
- `version`.
- `nr_entries` (count of logged entries).
- `sectorsize` (logdev sector size).
- `entry_size` (typical entry size).

REQ-5: Per-bio dispatch (`log_writes_map`):
1. lc := target.private.
2. If !lc.logging_enabled: pass-through to data dev.
3. Else for write/discard/flush:
   - alloc pending_block; capture bio metadata.
   - bio_list_add(&lc.pending_blocks).
   - Schedule kthread.
4. Pass-through bio to data dev.

REQ-6: Per-write log entry write (`log_writer kthread`):
1. For each pending_block:
   - Compose log_write_entry header.
   - Write header + bio payload to logdev at next_sector.
   - next_sector += entry-size.
   - lc.entries++.
   - Free pending_block.
2. After batch: log_super(lc) to checkpoint.

REQ-7: Per-target dmsetup messages:
- "mark <name>": insert marker entry; userspace can replay-up-to-marker.
- "verbose": dump log entries to dmesg.
- "log_only": only log; don't pass-through.

REQ-8: Per-marker entries:
- Special log entry with userspace-supplied name.
- Used by xfstests to delineate test-phases.

REQ-9: Discard / Trim:
- Pass-through to data + log discard-entry on logdev.

REQ-10: Replay tools (userspace):
- log-writes-replay (in xfsprogs / xfstests): reads journal, replays per-entry, stops at specified marker.
- Used to recreate crash-state for FS-test recovery validation.

REQ-11: Per-target counter:
- entries counter: monotonic; visible via dmsetup status.

REQ-12: Failure handling:
- Logdev failure: target may continue with logging disabled OR fail-shut.
- Data-dev failure: bio_io_error returned; log entry may still be written depending on order.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pending_block_no_uaf` | UAF | per-PendingBlock freed only after log entry written. |
| `next_sector_monotonic` | INVARIANT | lc.next_sector advances by sectors-used per entry; never moves backward. |
| `entries_counter_monotonic` | INVARIANT | lc.entries monotonically increases. |
| `bio_pass_through_unmodified` | INVARIANT | bio.bi_bdev = data_dev for non-log mode. |
| `pending_list_serialized` | INVARIANT | lc.pending_blocks mutated only under lc.log_lock. |

### Layer 2: TLA+

`drivers/md/log_writes_lifecycle.tla`:
- Per-bio state ∈ {Pending, Logged, Failed}.
- Properties:
  - `safety_log_entries_match_writes` — every logged entry corresponds to exactly one bio.
  - `safety_log_order_matches_bio_order` — entries written in bio-submission order.
  - `liveness_pending_eventually_logged` — assuming logdev healthy, every Pending eventually Logged.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `LogWrites::map_bio` post: bio passed-through to data_dev; pending_block queued | `LogWrites::map_bio` |
| `LogWrites::log_writer` post: per-pb entry written to logdev; pb freed | `LogWrites::log_writer` |
| `LogWrites::log_super` post: logdev sector 0 has updated superblock | `LogWrites::log_super` |
| Per-target entries count == actual entries on logdev | invariants on log_writer |

### Layer 4: Verus/Creusot functional

`Per-bio: write to data-dev + log entry on logdev` semantic equivalence: per-bio data-dev gets the write AND logdev gets matching log entry.

### hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-log-writes-specific reinforcement:

- **Per-bio passthrough preserves data-bdev semantics** — defense against logging interfering with data-IO correctness.
- **Logdev failure → logging disabled but data-IO continues** — defense against single-point-of-failure on log.
- **Per-marker user-supplied name validated** — defense against attacker-injected log control characters.
- **Per-target log_lock serialization** — defense against torn entry-write.
- **next_sector validation** against logdev capacity — defense against OOB log-write.
- **Per-PendingBlock data clone for FUA** — defense against payload-mutation between bio submit + log write.
- **Logging_enabled atomic toggle** — defense against torn state during dmsetup message.
- **Logdev superblock periodic checkpoint** — defense against incomplete-log on sudden shutdown.
- **Per-bio refcount during log-writer ref** — defense against UAF.
- **dmsetup logged_count visible** — defense against silent log-loss.

