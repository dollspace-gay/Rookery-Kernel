---
title: "Tier-3: drivers/md/dm-flakey.c — fault-injection target (configurable up/down intervals + per-direction read/write/error/drop policy)"
tags: ["tier-3", "drivers-md", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

dm-flakey is the device-mapper fault-injection target — periodically degrades pass-through to underlying device based on configurable up/down intervals (e.g., "20s up, 10s down"). During down-intervals: bios may be returned with -EIO, dropped silently, or have data scrambled (random byte flips, corrupt write). Used by xfstests + filesystem-recovery testing to validate filesystems against transient block-device faults. dm-flakey also provides "drop_writes" mode (writes silently succeed but data not actually written) for log-replay testing. Tier-3 batches dm-flakey + adjacent simple dm-targets (dm-delay, dm-zero, dm-linear, dm-stripe).

This Tier-3 covers `dm-flakey.c` (~709 lines) + brief coverage of `dm-delay.c`, `dm-zero.c`, `dm-linear.c`, `dm-stripe.c`.

### Acceptance Criteria

- [ ] AC-1: Create flakey "20 10": up 20jiffies, down 10jiffies; verify alternating periods.
- [ ] AC-2: ERROR_WRITES: writes during down-period return -EIO; reads pass through.
- [ ] AC-3: DROP_WRITES: writes during down-period silently lost; reads from same range return stale data.
- [ ] AC-4: Corrupt_bio_byte: writes during down corrupt N-th byte to value V; reads see corruption.
- [ ] AC-5: dm-delay: 100ms read-delay; bio completion latency ≥ 100ms.
- [ ] AC-6: dm-linear: linear at offset 1000; bio sector S → underlying sector (S + 1000).
- [ ] AC-7: dm-zero: read returns zeros; write silently succeeds.
- [ ] AC-8: dm-stripe with 4 disks: bios distributed; per-disk-IO approximately balanced.
- [ ] AC-9: xfstests fs/transient_failure tests pass with dm-flakey.
- [ ] AC-10: dm-flakey corruption_count counter increments per corruption event.

### Architecture

`FlakeyC`:

```
struct FlakeyC {
  dev: KArc<DmDev>,
  start: u64,
  up_interval: u32,                            // jiffies
  down_interval: u32,
  corrupt_bio_byte: u32,                       // byte offset; 0 = disabled
  corrupt_bio_rw: u32,                         // READ or WRITE
  corrupt_bio_value: u8,
  corrupt_bio_flags: u32,
  flags: u32,                                  // DROP_WRITES, ERROR_WRITES, ERROR_READS, RANDOM_*
  random_read_corrupt_pct: u32,
  random_write_corrupt_pct: u32,
  start_time: u64,                             // jiffies at ctr
  ...
}
```

`Flakey::map_bio(target, bio)`:
1. fc := target.private.
2. elapsed := jiffies - fc.start_time.
3. cycle := elapsed % (fc.up_interval + fc.down_interval).
4. is_up := cycle < fc.up_interval.
5. bio.bi_iter.bi_sector += fc.start.
6. bio.bi_bdev = fc.dev.bdev.
7. If is_up: return DM_MAPIO_REMAPPED.
8. Else (DOWN):
   - If bio.bi_op == REQ_OP_WRITE && (fc.flags & DROP_WRITES): bio_endio(bio); return DM_MAPIO_SUBMITTED.
   - If bio.bi_op == REQ_OP_WRITE && (fc.flags & ERROR_WRITES): bio_io_error(bio); return DM_MAPIO_SUBMITTED.
   - If bio.bi_op == REQ_OP_READ && (fc.flags & ERROR_READS): bio_io_error(bio); return DM_MAPIO_SUBMITTED.
   - Else if should_corrupt_bio(fc, bio.bi_op):
     - corrupt_bio_data(bio, fc).
   - return DM_MAPIO_REMAPPED.

`Flakey::corrupt_bio_data(bio, fc)`:
1. Iterate bio_vec:
   - If fc.corrupt_bio_byte within page: page[byte] = fc.corrupt_bio_value.

`Linear::map_bio(target, bio)`:
1. lc := target.private.
2. bio.bi_iter.bi_sector = bio.bi_iter.bi_sector - target.begin + lc.start.
3. bio.bi_bdev = lc.dev.bdev.
4. return DM_MAPIO_REMAPPED.

`Zero::map_bio(target, bio)`:
1. switch bio.bi_op:
   - REQ_OP_READ: zero_fill_bio(bio); bio_endio; return DM_MAPIO_SUBMITTED.
   - REQ_OP_WRITE: bio_endio (no-op); return DM_MAPIO_SUBMITTED.
   - other: -EOPNOTSUPP.

### Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-thin / dm-cache / dm-snapshot / dm-mirror / dm-crypt / dm-integrity (covered separately)
- dm-error (per-bio always-error; trivial)
- dm-multipath (covered in `dm-multipath.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct flakey_c` | per-target config | `drivers::md::flakey::FlakeyC` |
| `flakey_ctr(target, argc, argv)` | target-create | `Flakey::ctr` |
| `flakey_dtr(target)` | dtor | `Flakey::dtr` |
| `flakey_map(target, bio)` | per-bio dispatch | `Flakey::map_bio` |
| `flakey_end_io(target, bio, err)` | bio completion | `Flakey::end_io` |
| `flakey_status(target, type, ...)` | dmsetup status | `Flakey::status` |
| `flakey_message(target, argc, argv)` | DM_TARGET_MSG | `Flakey::message` |
| `corrupt_bio_data(bio, fc)` | per-bio corruption helper | `Flakey::corrupt_bio_data` |
| `parse_features(...)` | per-target feature parsing | `Flakey::parse_features` |
| `should_corrupt_bio(fc, bio_op)` | per-bio corruption decision | `Flakey::should_corrupt_bio` |
| `delay_ctr(target, argc, argv)` (dm-delay.c) | dm-delay setup | `Delay::ctr` |
| `delay_map(target, bio)` | per-bio defer-via-timer | `Delay::map_bio` |
| `linear_map(target, bio)` (dm-linear.c) | per-bio simple-remap | `Linear::map_bio` |
| `stripe_map(target, bio)` (dm-stripe.c) | per-bio stripe-distribute | `Stripe::map_bio` |
| `zero_map(target, bio)` (dm-zero.c) | per-bio zero-on-read / discard-on-write | `Zero::map_bio` |

### compatibility contract

REQ-1: Per-target `flakey_c`:
- `dev` (DmDev for underlying block-device).
- `start` (sector offset on dev).
- `up_interval` / `down_interval` (jiffies).
- `corrupt_bio_byte` (which byte to corrupt; 0 = disabled).
- `corrupt_bio_rw` (READ / WRITE direction).
- `corrupt_bio_value` (byte value to write).
- `corrupt_bio_flags` (bio_op mask).
- `flags` (DROP_WRITES / ERROR_WRITES / ERROR_READS / RANDOM_READ_CORRUPT / RANDOM_WRITE_CORRUPT).
- `random_read_corrupt_pct` / `random_write_corrupt_pct` (0-100).

REQ-2: Per-bio dispatch (`flakey_map`):
1. fc := target.private.
2. now := jiffies.
3. up := (now / (up_interval + down_interval)) * (up_interval + down_interval) + up_interval.
4. If now < up: bio passes through (UP interval).
5. Else (DOWN interval):
   - If flags & ERROR_WRITES && bio.bi_op == WRITE: bio_io_error.
   - If flags & ERROR_READS && bio.bi_op == READ: bio_io_error.
   - If flags & DROP_WRITES && bio.bi_op == WRITE: bio_endio without submitting.
   - Else: corrupt_bio_data + pass-through.

REQ-3: corrupt_bio_data (controlled corruption):
1. If corrupt_bio_byte == 0: skip.
2. Per-bio iter pages:
   - If page-byte at corrupt_bio_byte matches direction: write corrupt_bio_value.

REQ-4: random_corrupt:
- Per-bio: random < random_read_corrupt_pct? corrupt random byte to random value.

REQ-5: dm-delay:
- Per-bio: defer via hrtimer for configurable jiffies (e.g., 100ms).
- Used to test timeout-handling.
- Per-target separate read/write delay.

REQ-6: dm-linear (simple pass-through with offset):
- target.private = { dev, start_sector }.
- Per-bio: bio.bi_iter.bi_sector += target.start_sector; bio.bi_bdev = target.dev.bdev; submit.
- Trivial; ~234 lines.

REQ-7: dm-zero:
- Per-bio:
  - READ: zero-fill bio pages; bio_endio.
  - WRITE: discard (don't write).

REQ-8: dm-stripe:
- target.private = { stripes[], chunk_size }.
- Per-bio: split into per-chunk; per-chunk to designated stripe-disk.
- Same logic as raid0 but as a dm-target (vs as md personality).

REQ-9: Statistics for testing:
- /sys/dm-flakey/N/ exposes counters: corruption_count, error_count, drop_count.
- Used by xfstests assertions.

REQ-10: Per-feature dynamic toggle via DM_TARGET_MSG:
- "flakey set_corrupt_bio_byte N V": runtime corrupt setup.
- "flakey clear": disable.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `interval_period_nonzero` | INVARIANT | per-target up_interval + down_interval > 0; defense against div-by-zero. |
| `corrupt_byte_within_bio` | OOB | per-corrupt-bio-byte index < bio total length. |
| `flags_bitmask_valid` | INVARIANT | flags ⊆ DROP_WRITES | ERROR_WRITES | ERROR_READS | RANDOM_*. |
| `random_pct_bounded` | INVARIANT | random_*_corrupt_pct ∈ [0, 100]. |

### Layer 2: TLA+

`drivers/md/flakey_lifecycle.tla`:
- Per-target state ∈ {Up, Down}.
- Periodic transition based on jiffies.
- Per-bio in Down state: dispatch decision tree.
- Properties:
  - `safety_up_iff_in_up_window` — bio passes through iff cycle position < up_interval.
  - `safety_drop_silently` — DROP_WRITES decisions don't generate -EIO; bio_endio with success.
  - `liveness_period_oscillates` — assuming jiffies advance, state oscillates Up ↔ Down.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Flakey::map_bio` post: bio remapped (UP), error'd (ERROR), endio'd (DROP), or corrupted-then-remapped | `Flakey::map_bio` |
| `Flakey::corrupt_bio_data` post: per-bio bytes at corrupt_bio_byte match corrupt_bio_value | `Flakey::corrupt_bio_data` |
| Per-target jiffies-cycle deterministic | invariants on map_bio |

### Layer 4: Verus/Creusot functional

`Flakey behavior over time: per-period UP delivers reliable IO, DOWN delivers configured failures` semantic equivalence: per-period the bio outcome matches the configured (DROP / ERROR / CORRUPT / pass-through) policy.

### hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-flakey-specific reinforcement:

- **interval > 0 enforced** — defense against div-by-zero in cycle compute.
- **corrupt_bio_byte bounded** — defense against bio-page OOB write.
- **Per-direction policy enforced** — defense against confusion of read-vs-write semantics.
- **DROP_WRITES vs ERROR_WRITES distinct** — defense against silent vs noisy failure conflation.
- **dm-flakey privileged at create** — defense against unauthorized fault-injection.
- **Per-feature flag bitmap whitelist** — defense against unknown flag bits causing future-incompat.
- **dm-zero discard semantics** — defense against silent data acceptance interfering with FS journal.
- **dm-delay hrtimer per-bio** — defense against per-bio timer-overflow on flood.
- **dm-stripe chunk-size validated power-of-2** — defense against modulo-overhead.
- **dm-linear offset + size ≤ dev capacity** — defense against OOB device access.

