# Tier-3: drivers/md/dm-crypt.c — encrypted block device target (LUKS / per-sector AES-XTS / authenticated-encryption)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-crypt.c
  - include/crypto/skcipher.h (used)
  - include/crypto/aead.h (authenticated encryption)
-->

## Summary

dm-crypt is the device-mapper target backing LUKS (Linux Unified Key Setup) full-disk encryption — every sector is encrypted in-place via AES-XTS (default) or other registered cipher, with per-sector IV derived from sector-number (essiv / plain64). Userspace cryptsetup parses LUKS header (containing key-slots + master-key wrapped with passphrase-derived KEK) + activates dm-crypt target with master-key. Per-bio: encrypt/decrypt happens in worker-thread (cipher operations may take 100s of microseconds; not safe in atomic ctx). Supports authenticated-encryption variants (--integrity for dm-integrity stack). Fundamental block-encryption primitive on Linux.

This Tier-3 covers `drivers/md/dm-crypt.c` (~3723 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct crypt_config` | per-target config | `drivers::md::crypt::CryptConfig` |
| `struct dm_crypt_io` | per-bio crypt context | `DmCryptIo` |
| `struct dm_crypt_request` | per-cipher-request descriptor | `DmCryptRequest` |
| `struct iv_essiv_private` | ESSIV-mode IV state | `IvEssivPrivate` |
| `struct iv_lmk_private` | LMK-mode IV state | `IvLmkPrivate` |
| `struct iv_tcw_private` | TCW-mode IV state | `IvTcwPrivate` |
| `struct iv_eboiv_private` | EBOIV-mode IV state | `IvEboivPrivate` |
| `crypt_ctr(target, argc, argv)` | target-create | `Crypt::ctr` |
| `crypt_dtr(target)` | destroy | `Crypt::dtr` |
| `crypt_map(target, bio)` | per-bio dispatch | `Crypt::map_bio` |
| `crypt_endio(target, bio, error)` | bio completion | `Crypt::endio` |
| `crypt_status(...)` | dmsetup status | `Crypt::status` |
| `crypt_message(...)` | DM_TARGET_MSG (e.g., key-changes) | `Crypt::message` |
| `kcryptd_queue_crypt(io)` | enqueue io to worker | `Crypt::queue_crypt` |
| `kcryptd_crypt(work)` | per-bio worker entry | `Crypt::worker` |
| `kcryptd_async_done(...)` | async-cipher completion callback | `Crypt::async_done` |
| `crypt_alloc_buffer(io, size)` | per-bio alloc encrypted-buffer | `DmCryptIo::alloc_buffer` |
| `crypt_dec_pending(...)` / `crypt_inc_pending(...)` | per-bio refcount | `DmCryptIo::dec_pending` / `_inc_pending` |
| `crypt_iv_*_gen(cc, iv, dmreq)` (per-mode IV gen) | per-IV-mode IV computation | `Crypt::iv_<mode>_gen` |
| `crypt_alloc_tfms(...)` | per-CPU per-target tfm pool | `Crypt::alloc_tfms` |
| `crypt_setkey(...)` | install master key | `Crypt::setkey` |
| `crypt_wipe_key(...)` | wipe key from memory | `Crypt::wipe_key` |
| `crypt_iv_init(...)` | per-mode IV init | `Crypt::iv_init` |
| `crypt_per_request_data(...)` | per-request scratch buffer | `Crypt::per_request_data` |
| `crypt_convert_block_skcipher(...)` | per-block sk-cipher invoke | `Crypt::convert_block_skcipher` |
| `crypt_convert_block_aead(...)` | per-block AEAD invoke | `Crypt::convert_block_aead` |

## Compatibility contract

REQ-1: Per-target `crypt_config`:
- `dev` (underlying block-device).
- `start` (offset on dev for first sector).
- `tfms` (per-CPU array of crypto-tfm; for parallel encryption).
- `tfms_count`.
- `key_size` (bytes of master key).
- `key_parts` (count of key fragments; default 1 unless XTS-with-different-key-per-half).
- `key_extra_size` (extra key bytes for AEAD mode).
- `key` (master key bytes).
- `cipher` (e.g., "aes").
- `mode` (e.g., "xts", "cbc", "gcm").
- `iv_size` (cipher IV size).
- `iv_offset` (sector offset added to sector-IV; for shared-bdev variants).
- `iv_gen_ops` (per-IV-mode vtable).
- `cipher_string` (full cipher spec; e.g., "aes-xts-plain64").
- `req_pool` (mempool for dm_crypt_request).
- `bs_pool` (per-cpu sk-cipher buffer pool).
- `crypt_queue` (workqueue).
- `flags` (dm-crypt flags: SAME_CPU_CRYPT, NO_WORKQUEUE, INTEGRITY_AEAD, INTEGRITY_HMAC, etc.).

REQ-2: Per-bio `dm_crypt_io`:
- `cc` (back-ref to crypt_config).
- `base_bio` (original bio).
- `ctx` (current dm_crypt_request being processed).
- `error` (running error code).
- `sector` (current sector being processed).
- `pending` (atomic refcount).
- `submit_io` (sub-bio for encrypted version).

REQ-3: AES-XTS mode (default for LUKS2):
- Two AES keys (key_part_a, key_part_b) total 256 or 512 bits.
- Per-sector IV: 64-bit sector number (via plain64 IV-gen).
- Per-block tweaked AES.

REQ-4: Per-bio flow on write:
1. crypt_map(target, bio):
   - If bio.bi_op == REQ_OP_WRITE:
     - allocate new bio (sub_bio) for ciphertext output.
     - allocate ciphertext pages.
     - kcryptd_queue_crypt(io) → worker thread.
   - If bio.bi_op == REQ_OP_READ:
     - submit underlying read first; on completion, queue decryption.
2. Worker:
   - Per-block: gen IV (sector-based); cipher encrypt(plaintext, iv) → ciphertext.
   - Submit sub_bio to underlying device.
3. On sub_bio completion: bio_endio original bio.

REQ-5: Per-bio flow on read:
1. crypt_map: submit underlying read for original bio location.
2. crypt_endio: queue decrypt-worker.
3. Worker: per-block decrypt; populate plaintext into bio's pages.
4. bio_endio.

REQ-6: IV generation modes:
- plain: 32-bit sector number (truncated).
- plain64: 64-bit sector number.
- essiv (Encrypted Salt-Sector IV): hash(key) used to encrypt sector-number → IV.
- benbi: 64-bit big-endian sector number.
- null: zero IV (insecure; only for special test cases).
- lmk (loop-AES-compatible).
- tcw (TrueCrypt-compatible).
- random: random IV (must be stored alongside ciphertext; rare).
- eboiv (Encrypted Block Number IV): tweaked-encrypt of sector for AES-XTS variants.

REQ-7: Per-CPU tfm pool:
- N CPUs × tfms_count crypto-tfm instances allocated at ctr.
- Worker selects per-CPU tfm to avoid lock contention on shared cipher state.

REQ-8: Authenticated encryption (--integrity):
- Cipher = aes-gcm or aes-xts + chained-hash.
- Per-sector tag stored in dm-integrity layer below dm-crypt.
- Detects + rejects tampered ciphertext on read.

REQ-9: Key wipe on dtr:
- crypt_wipe_key zeros master key + per-tfm keys via memzero_explicit.
- Defense against post-suspend RAM dump exposing keys.

REQ-10: Suspend/resume coordination:
- DM suspend waits for in-flight encrypt/decrypt; safe-quiesce.
- Resume: re-init worker; re-attach key (via DM_TARGET_MSG "key set" if temporarily wiped on suspend).

REQ-11: Workqueue dispatch:
- Default: per-target unbound workqueue.
- SAME_CPU_CRYPT flag: bound workqueue keeping per-bio on submitting CPU.
- NO_WORKQUEUE flag: synchronous decrypt in atomic context (small bios; reads).

REQ-12: dm_crypt_request scratch:
- Per-request: SG-list for source + dest + scratch IV space.
- mempool-allocated; released on completion.

## Acceptance Criteria

- [ ] AC-1: cryptsetup luksFormat /dev/sda1 + luksOpen test_crypt; dm-crypt target active.
- [ ] AC-2: Read-write through dm-crypt: write 100MB; read back; verifies bytes match (round-trip).
- [ ] AC-3: AES-XTS-plain64: write known-plaintext via dm-crypt; verify ciphertext on underlying dev differs (encrypted).
- [ ] AC-4: Multi-CPU: 32-CPU host with concurrent dm-crypt IO; per-CPU tfm pool used; throughput scales.
- [ ] AC-5: --integrity option: aes-xts + integrity hmac; tamper test (write random bytes to underlying dev) detected on read.
- [ ] AC-6: Suspend/resume: dmsetup suspend; in-flight bios drain; resume; subsequent IO works.
- [ ] AC-7: Key wipe: dmsetup remove; verify key bytes zeroed via /dev/kmem inspection (test build).
- [ ] AC-8: Performance: dm-crypt linear write ≥ 50% of underlying-device throughput on aes-ni-equipped CPU.
- [ ] AC-9: cryptsetup-tests pass.
- [ ] AC-10: tcrypt module (kernel test): all dm-crypt path tests pass.

## Architecture

`CryptConfig`:

```
struct CryptConfig {
  dev: KArc<DmDev>,
  start: u64,                                   // sector offset
  tfms: PerCpu<KArc<CryptoSkCipher>>,
  tfms_count: u32,
  key_size: u32,
  key_parts: u32,
  key: KBox<[u8; MAX_KEY_SIZE]>,
  cipher_string: KString,
  iv_size: u8,
  iv_offset: u64,
  iv_gen_ops: KArc<dyn IvGenOps>,
  iv_private: Option<KBox<dyn IvPrivate>>,
  req_pool: KArc<MempoolDmCryptRequest>,
  bs_pool: PerCpu<KArc<MempoolBio>>,
  crypt_queue: KArc<Workqueue>,
  flags: u32,
  sector_size: u32,                              // 512 or 4096
  sector_shift: u32,
  on_disk_tag_size: u32,                         // for AEAD/integrity
  ...
}

struct DmCryptIo {
  cc: KArc<CryptConfig>,
  base_bio: KArc<Bio>,
  ctx: DmCryptRequestCtx,
  error: AtomicI32,
  sector: AtomicU64,
  pending: AtomicI32,
  submit_io: Option<KArc<Bio>>,
  cipher_finished: bool,
}
```

`Crypt::map_bio(target, bio)`:
1. cc := target.private.
2. io := DmCryptIo::alloc(cc, bio, target.begin).
3. If bio.bi_op == REQ_OP_WRITE:
   - allocate clone bio with ciphertext pages.
   - kcryptd_queue_crypt(io) → submit to worker for encrypt.
   - return DM_MAPIO_SUBMITTED.
4. If bio.bi_op == REQ_OP_READ:
   - bio.bi_iter.bi_sector += cc.start.
   - bio.bi_bdev = cc.dev.bdev.
   - return DM_MAPIO_REMAPPED (with endio hooked for decrypt).

`Crypt::worker(work)`:
1. io := container_of(work, DmCryptIo, work).
2. cc := io.cc.
3. Loop per-block in bio:
   - sector := io.sector.
   - sg-build per-block source + dest.
   - iv := iv_gen_ops.generate(cc, dmreq).
   - skcipher_request_set_crypt(...).
   - On write: skcipher_encrypt(req).
   - On read: skcipher_decrypt(req).
   - On async-completion: kcryptd_async_done called from softirq.
4. After all blocks: submit clone-bio (write) or bio_endio (read).

`Crypt::iv_essiv_gen(cc, iv, dmreq)`:
1. block-encrypt(hash(cc.key) used as essiv-key, sector_buf) → iv.

`Crypt::ctr(target, argc, argv)`:
1. Parse cipher_string (e.g., "aes-xts-plain64").
2. Allocate cc.
3. crypt_alloc_tfms(cc, cipher_name) — per-CPU sk-cipher tfm.
4. Validate key_size matches cipher.
5. crypt_setkey(cc, key) — install on all per-CPU tfms.
6. iv_gen_ops_init(cc, mode) — per-IV-mode private state.
7. Allocate workqueue + mempools.
8. target.private = cc.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `key_wipe_on_destroy` | INVARIANT | crypt_wipe_key zeros key bytes via memzero_explicit; defense against compiler-eliding zero. |
| `per_cpu_tfm_count_match` | INVARIANT | tfms[] alloc count == num_possible_cpus; defense against per-CPU OOB. |
| `iv_size_matches_cipher` | INVARIANT | per-cipher iv_size matches expected; defense against IV truncation. |
| `bio_pending_no_underflow` | INVARIANT | per-DmCryptIo pending refcount ≥ 0; defense against double-decrement causing premature endio. |
| `clone_bio_pages_owned` | UAF | per-clone-bio pages from per-CPU bs_pool; defense against double-free. |

### Layer 2: TLA+

`drivers/md/crypt_io_lifecycle.tla`:
- Per-bio state ∈ {Queued, EncryptingPlaintext, SubmittingCipher, ReadingCipher, DecryptingCipher, Completed, Errored}.
- Properties:
  - `safety_decrypt_only_after_read` — DecryptingCipher only after underlying-read complete.
  - `safety_no_double_endio` — bio_endio called exactly once per bio.
  - `liveness_pending_eventually_completes` — every Queued eventually Completed or Errored.

`drivers/md/crypt_iv_uniqueness.tla`:
- Per-sector IV uniqueness across underlying device.
- Properties:
  - `safety_iv_unique_per_sector` — for plain64/essiv/etc., IVs deterministic + per-sector.
  - `safety_no_iv_repeat_under_same_key` — defense against XTS cipher-reuse-attack.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Crypt::map_bio` post: bio remapped (read) or queued for crypt (write) | `Crypt::map_bio` |
| `Crypt::worker` post: per-block encrypt/decrypt complete; bio submitted/endio | `Crypt::worker` |
| `Crypt::ctr` post: per-CPU tfms allocated; key installed; cc populated | `Crypt::ctr` |
| `Crypt::dtr` post: key wiped; tfms freed; mempools released | `Crypt::dtr` |
| Per-bio io.pending balanced between inc/dec | invariants on bio refcount |

### Layer 4: Verus/Creusot functional

`Per-sector: encrypt(plaintext, iv(sector)) → underlying-write; underlying-read → decrypt(ciphertext, iv(sector)) → plaintext`: per-sector round-trip preserves data; ciphertext on underlying device hides plaintext.

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-crypt-specific reinforcement:

- **memzero_explicit on key wipe** — defense against compiler-eliding zeroing causing key-leak on dtr.
- **Per-CPU tfm pool** — defense against single-tfm shared-state contention + cipher-key-leak via timing.
- **Sector-based IV deterministic** — defense against IV-reuse attack on AES-XTS.
- **Cipher-spec parsed via crypto-API** — defense against attacker-supplied cipher-name causing crypto subsystem misuse.
- **Per-tfm key set under tfm.lock** — defense against torn key during concurrent set+use.
- **Workqueue isolation per-target** — defense against cross-target work interference.
- **bio_pending atomic refcount** — defense against double-endio causing UAF.
- **Per-block sg-list bounded** — defense against sg-list-overflow on pathological large bio.
- **Async-cipher completion serialization** — defense against torn-state on async cipher.
- **dm-integrity layer required for tamper-detect** — defense against silent ciphertext tampering.
- **NO_WORKQUEUE only for read-fast-path** — defense against atomic-context cipher-call sleeping.
- **Per-CPU bs_pool bounded** — defense against unbounded bio-clone allocation under load.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-integrity (covered in `dm-integrity.md` future Tier-3)
- LUKS header parsing (cryptsetup userspace concern)
- Crypto API (covered in `crypto/00-overview.md`)
- Implementation code
