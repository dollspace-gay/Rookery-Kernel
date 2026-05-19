# Tier-3: drivers/md/dm-integrity.c — per-sector data-integrity (HMAC tags + journal + recovery)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-integrity.c
  - drivers/md/persistent-data/dm-bufio.c (used)
  - include/linux/crypto.h (used for hmac)
-->

## Summary

dm-integrity is the device-mapper target that adds per-sector data-integrity tags (per-write computes HMAC + stores in tag-area; per-read recomputes + compares; mismatch returns -EILSEQ). Used as backing layer for dm-crypt's authenticated-encryption (LUKS2 with --integrity), or standalone for tamper-detect on plain ext4. Per-sector tag stored in dedicated tag-area on data-bdev (interleaved with data sectors per "tag-blocks-per-data-block" config). Journal area handles write-atomicity (write tag + data atomically); on crash mid-write, journal replay either applies or discards. Per-tag size 4-32 bytes (HMAC-SHA256 truncated; or AEAD auth-tag).

This Tier-3 covers `drivers/md/dm-integrity.c` (~5475 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dm_integrity_c` | per-target config | `drivers::md::integrity::DmIntegrityC` |
| `struct dm_integrity_io` | per-bio integrity context | `DmIntegrityIo` |
| `struct journal_entry` | per-journal-slot ABI | `JournalEntry` |
| `struct journal_sector` | per-512-byte journal sector | `JournalSector` |
| `dm_integrity_ctr(target, argc, argv)` | target-create | `DmIntegrity::ctr` |
| `dm_integrity_dtr(target)` | destroy | `DmIntegrity::dtr` |
| `dm_integrity_map(target, bio)` | bio dispatch | `DmIntegrity::map_bio` |
| `dm_integrity_endio(...)` | bio completion | `DmIntegrity::endio` |
| `integrity_metadata_io_complete(...)` | per-tag-block completion | `DmIntegrity::metadata_io_complete` |
| `integrity_data_io_complete(...)` | per-data completion | `DmIntegrity::data_io_complete` |
| `integrity_recheck(...)` | per-bio recheck | `DmIntegrity::recheck` |
| `do_endio_flush(...)` | bio_endio with flush | `DmIntegrity::do_endio_flush` |
| `journal_read_write(...)` | per-journal-slot I/O | `Journal::read_write` |
| `replay_journal(ic)` | crash-recovery replay | `Journal::replay` |
| `commit_journal(ic)` | journal commit | `Journal::commit` |
| `recalc_thread(arg)` | bg recalculate-tags worker | `DmIntegrity::recalc_thread` |
| `dm_integrity_alloc_tag(...)` | per-tag alloc | `DmIntegrity::alloc_tag` |
| `dm_integrity_compute_tag(...)` | HMAC compute | `DmIntegrity::compute_tag` |
| `dm_integrity_compare_tag(...)` | HMAC verify | `DmIntegrity::compare_tag` |
| `journal_section_init(...)` / `journal_section_close(...)` | per-section transaction | `Journal::section_init` / `_close` |

## Compatibility contract

REQ-1: Per-target `dm_integrity_c`:
- `dev` (block-device).
- `meta_dev` (separate metadata device; optional).
- `start` / `provided_data_sectors` (data range).
- `tag_size` (bytes per tag; e.g., 32 for HMAC-SHA256).
- `tag_alg` (HMAC / per-bio mode).
- `mode` (DM_INTEGRITY_MODE_DIRECT_WRITE / _BITMAP_MODE / _JOURNAL_MODE).
- `journal` (per-target journal-area metadata).
- `recalc_lock` / `recalc_workqueue` (background tag recalculation).
- `bufio` (dm-bufio for tag-area block I/O).
- `internal_hash` (per-target HMAC key).
- `legacy_recalc` (legacy compatibility).

REQ-2: Per-bio `dm_integrity_io`:
- `ic` (back-ref).
- `bio_details` (cloned bio info).
- `range_lock` (range-locked sectors).
- `payload` (per-bio scratch for tag computation).

REQ-3: Per-tag layout:
- Tag = HMAC(internal_hash, sector_number || sector_data) truncated to tag_size.
- Per-data-block has 1 tag (1 sector default; or larger).
- Tag-area placement:
  - Direct mode: separate metadata device or inline interleaving.
  - Journal mode: tags written to journal first, then to tag-area on commit.

REQ-4: Per-bio write flow (journal mode):
1. Acquire range-lock for bio's sector range.
2. compute_tag(sector, data) → tag.
3. Allocate journal-section.
4. Write (sector, data, tag) tuple to journal.
5. journal-flush (force barrier).
6. Update tag-area in-memory; mark dirty.
7. Submit data write to underlying-dev.
8. On data-write complete: bio_endio.
9. Background flush tag-area updates to disk asynchronously.

REQ-5: Per-bio read flow:
1. Submit underlying read.
2. On read complete: read corresponding tag from tag-area.
3. compare_tag(sector, data, expected_tag).
4. If mismatch: bio_io_error with EILSEQ.
5. Else: bio_endio with success.

REQ-6: Journal replay:
1. On mount: scan journal sections.
2. Per-uncommitted section: re-apply (write tag-area + data).
3. On committed section: skip.
4. Mark journal empty after replay.

REQ-7: Bitmap mode (no journal; faster):
- Per-data-block dirty-bitmap on metadata-dev.
- On crash: tag may be inconsistent; subsequent read returns -EILSEQ → caller must retry write.

REQ-8: Background recalc:
- For shrinking / extending integrity-protected range: recalc_thread reads existing data + writes new tag.
- Tracks progress via metadata.

REQ-9: AEAD mode integration with dm-crypt:
- dm-crypt provides AEAD cipher (e.g., aes-gcm).
- AEAD-tag from cipher serves as integrity tag.
- dm-integrity stores AEAD tag in tag-area.

REQ-10: discard-handling:
- Discard zero out tag-area for affected blocks.
- Subsequent read returns zeros + valid tag (or -EILSEQ if not zeroed).

## Acceptance Criteria

- [ ] AC-1: dmsetup create test_integrity → integrity target with HMAC-SHA256 + 32-byte tag.
- [ ] AC-2: Write 100MB; per-block tag stored in tag-area.
- [ ] AC-3: Read back: tag verified; data identical.
- [ ] AC-4: Tamper test: write random bytes directly to underlying data sectors; read via dm-integrity returns -EILSEQ.
- [ ] AC-5: Crash recovery: kill -9 mid-write; reboot; journal replay completes; data consistent.
- [ ] AC-6: --integrity HMAC + dm-crypt AEAD: stack works; tamper detected.
- [ ] AC-7: Discard: range discard; subsequent read returns zeros without -EILSEQ.
- [ ] AC-8: Performance: linear write ≥ 50% of underlying-bw with HMAC-SHA256.
- [ ] AC-9: Background recalc on extend: integrity range extended via dmsetup message; recalc_thread populates tags.
- [ ] AC-10: dm-integrity xfstests pass.

## Architecture

`DmIntegrityC`:

```
struct DmIntegrityC {
  dev: KArc<DmDev>,
  meta_dev: Option<KArc<DmDev>>,
  start: u64,
  provided_data_sectors: u64,
  tag_size: u32,
  tag_alg: TagAlg,                              // HMAC, AEAD
  mode: IntegrityMode,                          // DirectWrite, Bitmap, Journal
  journal: Option<KArc<Journal>>,
  bufio: KArc<DmBufio>,                          // for tag-area block I/O
  internal_hash: KArc<CryptoShash>,             // HMAC key + tfm
  recalc_lock: Mutex<()>,
  recalc_workqueue: KArc<Workqueue>,
  recalc_buffer: Option<KBox<[u8]>>,
  ...
}

struct DmIntegrityIo {
  ic: KArc<DmIntegrityC>,
  bio_details: BioDetails,
  range_lock: ListNode,
  payload: KBox<[u8; INTEGRITY_PAYLOAD_SIZE]>,
}
```

`DmIntegrity::map_bio(target, bio)`:
1. ic := target.private.
2. io := DmIntegrityIo::alloc(ic, bio).
3. Acquire range-lock for bio sector range.
4. If bio.bi_op == REQ_OP_WRITE:
   - compute_tag for each sector.
   - If journal mode: queue to journal-thread.
   - Else: write tag-area then data.
5. If bio.bi_op == REQ_OP_READ:
   - submit data read.
   - On completion: read tag-area; compare.
6. return DM_MAPIO_SUBMITTED.

`DmIntegrity::compute_tag(ic, sector, data, tag_out)`:
1. shash_digest_init(&desc, ic.internal_hash).
2. shash_update(&desc, &sector_be64, 8).
3. shash_update(&desc, data, sector_size).
4. shash_final(&desc, tag_out).

`DmIntegrity::compare_tag(ic, sector, data, expected_tag)`:
1. compute_tag(...) → computed_tag.
2. If memcmp(computed_tag, expected_tag, tag_size) != 0: return -EILSEQ.
3. Return 0.

`Journal::commit(ic)`:
1. Mark current journal-section as committed.
2. Submit journal-section write to underlying-dev.
3. On flush: write tag-area updates from journal to tag-area.
4. Mark journal-section as empty.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tag_compute_no_oob` | OOB | per-sector compute_tag bounded by sector_size; defense against tag-buffer overrun. |
| `range_lock_balanced` | INVARIANT | per-bio range-lock acquire/release paired; defense against deadlock. |
| `journal_section_state_valid` | INVARIANT | per-section state ∈ {Empty, Filling, Committing, Committed}. |
| `tag_size_bounded` | INVARIANT | tag_size ≤ MAX_TAG_SIZE; defense against per-tag buffer overflow. |
| `recalc_progress_monotonic` | INVARIANT | recalc-progress field monotonically advances. |

### Layer 2: TLA+

`drivers/md/integrity_journal.tla`:
- Per-write state ∈ {RangeLocked, JournalWritten, JournalCommitted, DataWritten, Done}.
- Per-section state ∈ {Empty, Active, Committing, Committed}.
- Properties:
  - `safety_no_data_without_tag` — DataWritten only after JournalCommitted (so post-crash replay valid).
  - `safety_replay_idempotent` — replaying same journal-section produces same result.
  - `liveness_pending_eventually_committed` — every Active section eventually committed.

`drivers/md/integrity_tamper.tla`:
- Per-bio read state with tampering model.
- Properties:
  - `safety_tamper_detected` — modification of data on underlying-dev → read returns -EILSEQ.
  - `safety_no_false_positive` — un-tampered data passes verification.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `DmIntegrity::map_bio` post: range-locked; submitted to underlying or journal | `DmIntegrity::map_bio` |
| `DmIntegrity::compute_tag` post: tag_out filled with HMAC; deterministic for given (sector, data) | `DmIntegrity::compute_tag` |
| `DmIntegrity::compare_tag` post: returns 0 iff computed == expected | `DmIntegrity::compare_tag` |
| `Journal::commit` post: section marked committed; persisted to disk before next op uses tag-area | `Journal::commit` |
| Per-bio range-lock held throughout I/O | invariants on map_bio + endio |

### Layer 4: Verus/Creusot functional

`Per-write: data + tag persisted atomically; per-read: tag verified; tamper-detect rate = 100% modulo HMAC collision` semantic equivalence: per-sector the tag uniquely binds (sector_number, data, internal_hash); any modification of data without rewriting tag detected on read.

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-integrity-specific reinforcement:

- **Per-target internal_hash key wiped on dtr** — defense against post-suspend RAM-leak of HMAC key.
- **HMAC-SHA256 default** — defense against cryptanalytic-attack on weaker MACs.
- **Sector-number in HMAC input** — defense against block-relocation attack where attacker swaps two valid (data, tag) pairs.
- **Journal commit before tag-area update** — defense against post-crash inconsistent (data, tag) pair.
- **Journal replay idempotent** — defense against double-commit on multi-mount.
- **Range-lock per-bio** — defense against concurrent write-then-read seeing partial state.
- **AEAD-mode tag stored separately from cipher** — defense against AEAD-tag-leak via tag-area dump.
- **Discard zeroes tag-area** — defense against subsequent read returning random tag.
- **Per-target legacy_recalc gating** — defense against silent recalc on incompatible-version tag.
- **Per-bio dm_integrity_io.payload mempool** — defense against unbounded allocation under load.
- **bg recalc-thread bounded by recalc_buffer size** — defense against recalc consuming all CPU.
- **HMAC-output-truncation only to ≥ 16 bytes** — defense against forge-tag via short truncation.

## Grsecurity/PaX-style Reinforcement

Rationale: dm-integrity stores per-sector HMAC tags + a write-ahead journal beneath dm-crypt, providing the only authenticator that detects ciphertext tampering on LUKS2-with-integrity volumes. A leaked HMAC key, journal replay corruption, or sector-tag aliasing here directly defeats authenticated FDE. The reinforcement below restates baseline PaX/grsec coverage applied to `drivers/md/dm-integrity.c` plus dm-integrity-specific reinforcement.

Baseline (cross-ref `drivers/md/dm-core.md` § Hardening):
- **PAX_USERCOPY**: status output through dm-core whitelisted slab; ctr param parsing whitelisted.
- **PAX_KERNEXEC**: per-`dm_target_type` vtable, per-`crypto_shash`/`crypto_ahash` vtable, journal-mode dispatch table all `__ro_after_init`.
- **PAX_RANDKSTACK**: every entry to `dm_integrity_map` / `dm_integrity_end_io` / journal commit randomises stack.
- **PAX_REFCOUNT**: per-`dm_integrity_c` refcount, per-`dm_integrity_io.payload` refcount, journal-section refcount saturating.
- **PAX_MEMORY_SANITIZE**: HMAC keys + per-sector tag scratch + journal-replay scratch zeroed on free — see integrity-specific subsection.
- **PAX_UDEREF**: ctr key parsing via dm-core's guarded path.
- **PAX_RAP / kCFI**: hash/cipher callbacks + journal-recover callback kCFI-tagged.
- **GRKERNSEC_HIDESYM**: `&ic`, `&ic->journal_xor`, per-key buffers never rendered; `%pK` only.
- **GRKERNSEC_DMESG**: integrity-failure warnings ratelimited (but always emit at least one — see below); CAP_SYSLOG.

dm-integrity-specific reinforcement:
- **HMAC-SHA-2 keys PAX_MEMORY_SANITIZE** — `ic->journal_mac_key`, `ic->internal_hash_key`, `ic->journal_crypt_key` allocated via `kvzalloc` and `memzero_explicit`-cleared on dtr; defends key residue identical to dm-crypt's posture.
- **HMAC-output-truncation lower bound 16 bytes** — already in row-2; reinforced with grsec audit refusal when activate args request `internal_hash:hmac(sha256):8` (8 bytes is forge-able).
- **Journal replay PAX_USERCOPY** — journal sections replay-read into a slab-allocated, USERCOPY-whitelisted buffer; per-section commit-tag verified BEFORE the buffer is consumed for sector dispatch. Defends against crafted journal sections leaking past tag verify.
- **Journal sector signature verify** — every journal commit section magic `DM_INTEGRITY_JOURNAL_MAGIC` + per-section HMAC validated; refusal triggers integrity-mode flag (target enters `FAILED` and refuses further writes).
- **Recalc-thread bounded by recalc_sectors** — extension of row-2; recalc thread cgroup-pinned so it cannot starve other tenants.
- **Per-bio `dm_integrity_io.payload` mempool reservation** — defends OOM-on-flood by reserving N_CPU * 8 payloads at activate.
- **CSPRNG-only HMAC key derive on `--integrity-no-journal-bitmap` recovery** — refuses fallback to `prandom`.
- **Activate-time integrity-mode lockdown** — refuses `bitmap` journal mode (lower assurance) when `kernel_is_locked_down(LOCKDOWN_INTEGRITY_MAX)`; defends downgrade attack at activate.
- **`integrity_metadata_size_per_sector` clamped to [8, 64] bytes** — defends ctr-string-injection driving heap-bounds confusion.
- **Tag mismatch always-emit (override ratelimit)** — first three tag mismatches per device emit one un-ratelimited dmesg + audit entry per the grsec policy; subsequent are ratelimited but counted via per-device counter exposed under CAP_SYS_ADMIN.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-crypt (covered in `dm-crypt.md` Tier-3)
- Crypto API (covered in `crypto/00-overview.md`)
- LUKS2 + integrity userspace (cryptsetup concern)
- Implementation code
