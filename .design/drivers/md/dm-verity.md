# Tier-3: drivers/md/dm-verity-target.c — authenticated read target (Merkle-tree per-block hash + signature verification + FEC)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/md/dm-core.md
upstream-paths:
  - drivers/md/dm-verity-target.c
  - drivers/md/dm-verity-fec.c
  - drivers/md/dm-verity-verify-sig.c
  - drivers/md/dm-verity-loadpin.c
  - drivers/md/dm-verity.h
-->

## Summary

dm-verity is the device-mapper target that authenticates per-block reads against a pre-computed Merkle-tree of per-block hashes — userspace builds the Merkle tree at image-creation; kernel verifies on read by walking from leaf-block hash up through tree-levels to root-hash; root-hash compared against signed/known value. Used by Android Verified Boot (AVB), ChromeOS verified-boot, immutable Linux distros (Project Atomic, Fedora Silverblue), Container OS images. dm-verity-fec.c adds optional Forward-Error-Correction (Reed-Solomon) for transient bit-flip recovery. dm-verity-verify-sig.c gates root-hash trust via per-image signature verification.

This Tier-3 covers `dm-verity-target.c` (~1838) + `dm-verity-fec.c` (~734) + `dm-verity-verify-sig.c` (~199).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dm_verity` | per-target config | `drivers::md::verity::DmVerity` |
| `struct dm_verity_io` | per-bio verification ctx | `DmVerityIo` |
| `struct dm_verity_fec` | per-target FEC config | `DmVerityFec` |
| `verity_ctr(target, argc, argv)` | target-create | `Verity::ctr` |
| `verity_dtr(target)` | dtor | `Verity::dtr` |
| `verity_map(target, bio)` | per-bio dispatch | `Verity::map_bio` |
| `verity_end_io(target, bio, err)` | bio completion | `Verity::end_io` |
| `verity_status(target, type, ...)` | dmsetup status | `Verity::status` |
| `verity_message(target, argc, argv)` | DM_TARGET_MSG | `Verity::message` |
| `verity_verify_io(io)` | per-bio verification | `DmVerityIo::verify` |
| `verity_for_io_block(...)` | per-block walk | `DmVerityIo::for_io_block` |
| `verity_for_bv_block(...)` | per-bio_vec block walk | `DmVerityIo::for_bv_block` |
| `verity_hash(...)` | per-block hash compute | `Verity::hash_block` |
| `verity_compute_hash(...)` | hash-helper wrapping shash | `Verity::compute_hash` |
| `verity_verify_zero_block(...)` | zero-block verify (special) | `Verity::verify_zero_block` |
| `verity_handle_err(...)` | per-corruption handler | `Verity::handle_err` |
| `verity_fec_init(...)` (dm-verity-fec.c) | FEC init | `DmVerityFec::init` |
| `verity_fec_decode(...)` | FEC error-correct | `DmVerityFec::decode` |
| `verity_verify_root_hash(v, opts)` (dm-verity-verify-sig.c) | per-target root-hash sig verify | `Verity::verify_root_hash_sig` |
| `verity_loadpin_dm_dev(...)` (dm-verity-loadpin.c) | per-process loadpin integration | `VerityLoadpin::dev` |

## Compatibility contract

REQ-1: Per-target `dm_verity`:
- `data_dev` (DmDev for verified data).
- `hash_dev` (DmDev for Merkle-tree hash blocks; can be same as data_dev).
- `data_blocks` (count of data blocks).
- `data_block_size` (typically 4KB).
- `hash_block_size` (typically 4KB; matches hashing-friendly granularity).
- `hash_per_block_bits` (log2(hash_block_size / digest_size)).
- `levels` (Merkle tree height).
- `hash_level_block[]` (per-level start block).
- `digest_size` (per-hash output size; e.g., 32 bytes for SHA-256).
- `salt_size` / `salt` (optional per-block salt).
- `version` (Merkle-tree variant: v0 = no salt, v1 = with salt).
- `algo_name` (hash algorithm; e.g., "sha256").
- `tfm` (per-CPU shash tfm).
- `root_digest` (Merkle-tree root hash; expected value).
- `mode` (ERROR / RESTART / IGNORE / PANIC).
- `fec` (Optional<DmVerityFec>).

REQ-2: Per-bio `dm_verity_io`:
- `v` (back-ref to dm_verity).
- `bio` (bio for data read).
- `block` (current block being verified).
- `n_blocks` (remaining blocks).
- `iter` (iterator into bio_vec).
- `result` (per-block computed hash).
- `recheck_buffer` (FEC recovery buffer).
- `error` (running error code).

REQ-3: Per-bio dispatch (`verity_map`):
1. v := target.private.
2. If !VERITY_RESTART (data dev not yet open) || bio.bi_op != REQ_OP_READ: return DM_MAPIO_REMAPPED for write/discard (data-dev pass-through).
3. Allocate dm_verity_io.
4. Submit underlying read for data blocks.
5. On read complete: queue verification on workqueue.

REQ-4: Per-block verification flow (`verity_verify_io`):
1. For each block in bio:
   - block_idx := bio.bi_iter.bi_sector / data_block_size + offset.
   - For level := 0 to levels-1:
     - hash_block_idx := block_idx >> (level * hash_per_block_bits).
     - Read hash-block from hash_dev at hash_level_block[level] + (hash_block_idx >> hash_per_block_bits).
     - Extract hash[level] = hash_block[(hash_block_idx & ((1 << hash_per_block_bits) - 1)) * digest_size].
   - leaf_hash := compute_hash(salt, block_data).
   - For level := 0 to levels-1:
     - parent_hash := compute_hash(salt, hash[level] || ...).
     - If parent_hash != hash[level+1]: corruption; verity_handle_err.
   - Final: hash[levels] == v.root_digest? else corruption.

REQ-5: Per-corruption handling (`verity_handle_err`):
- mode == ERROR: return -EILSEQ to bio.
- mode == RESTART: kernel_restart (panic-like).
- mode == IGNORE: log warning; pass through (insecure).
- mode == PANIC: panic.
- If FEC enabled: fec_decode attempts repair; only on FEC-failure escalate.

REQ-6: FEC (Forward-Error-Correction):
- Per-block Reed-Solomon code; recovers up to N corrupted bytes per block.
- Optional; configured via "verity-fec" args at create.
- Per-block FEC data stored on hash_dev or separate FEC dev.

REQ-7: Per-target signature verification:
- KVM_KCORE_*: verify root_digest via per-target signature (PKCS7).
- Used by Android Verified Boot to prevent root-hash tampering.

REQ-8: Per-target hash algorithm:
- Configurable via algo_name; typically sha256, blake2b, sha1 (deprecated).
- Per-CPU tfm pool for parallel hashing.

REQ-9: Per-target salt:
- Optional per-block salt (16-32 bytes typical).
- Mixed into per-block hash to defeat pre-computed-rainbow attacks.

REQ-10: Workqueue dispatch:
- Per-target unbound workqueue.
- Per-bio verification work is CPU-intensive (hash compute); deferred from softirq context.

REQ-11: Loadpin integration:
- Per-target verity-dev marked as "trusted source".
- Loadpin LSM restricts kernel-module loading + firmware to verity-mounted devs.

REQ-12: Per-block recheck buffer:
- For FEC + double-checking, per-bio scratch buffer.
- mempool-allocated.

## Acceptance Criteria

- [ ] AC-1: dmsetup create verity_test: data-dev + hash-dev + root_digest; subsequent reads verified.
- [ ] AC-2: Tampered data: write random bytes to underlying data-dev; subsequent dm-verity read returns -EILSEQ.
- [ ] AC-3: Tampered hash: tamper with Merkle-tree-block; verification fails.
- [ ] AC-4: Per-mode handling: RESTART mode triggers reboot on corruption (test in non-prod).
- [ ] AC-5: FEC recovery: induce N-byte corruption < FEC threshold; FEC repairs; read succeeds.
- [ ] AC-6: Per-target signature: verify-sig args; valid sig → mount succeeds; invalid sig → mount fails.
- [ ] AC-7: Performance: dm-verity sequential-read ≥ 80% of underlying-bandwidth on aes-ni-equipped CPU.
- [ ] AC-8: Multi-CPU: 32-CPU host concurrent reads; per-CPU tfm pool used; throughput scales.
- [ ] AC-9: Loadpin integration: dm-verity dev "trusted"; modules outside verity rejected.
- [ ] AC-10: dm-verity xfstests pass.

## Architecture

`DmVerity`:

```
struct DmVerity {
  data_dev: KArc<DmDev>,
  hash_dev: KArc<DmDev>,
  data_blocks: u64,
  data_block_size: u32,
  hash_block_size: u32,
  hash_per_block_bits: u32,
  levels: u32,
  hash_level_block: KVec<u64>,
  digest_size: u32,
  salt_size: u32,
  salt: KBox<[u8]>,
  version: u32,
  algo_name: KString,
  tfm: PerCpu<KArc<CryptoShash>>,
  root_digest: KBox<[u8]>,
  mode: VerityMode,
  fec: Option<DmVerityFec>,
  verify_wq: KArc<Workqueue>,
  io_pool: KArc<MempoolDmVerityIo>,
  recheck_pool: KArc<MempoolPages>,
  ...
}

struct DmVerityIo {
  v: KArc<DmVerity>,
  bio: KArc<Bio>,
  block: u64,
  n_blocks: u32,
  iter: BvecIter,
  result: KBox<[u8]>,                          // per-block computed hash
  recheck_buffer: KBox<[u8]>,
  error: AtomicI32,
  work: Work,
}
```

`Verity::map_bio(target, bio)`:
1. v := target.private.
2. If bio.bi_op != REQ_OP_READ:
   - Pass-through to data_dev (writes already validated by writer).
   - return DM_MAPIO_REMAPPED.
3. Allocate io := DmVerityIo from pool.
4. Submit underlying read for bio.
5. Queue verify work on completion.
6. return DM_MAPIO_SUBMITTED.

`Verity::verify_io(io)` (worker):
1. For each block in bio:
   - leaf_hash := compute_hash(salt, block_data).
   - For level := 0 to levels-1:
     - hash_block := read_hash_block(hash_dev, level, block_idx).
     - tree_hash := hash_block[entry_offset].
     - parent_hash := compute_hash(salt, leaf_hash || siblings).
     - If parent_hash != tree_hash: corruption; verity_handle_err.
   - Final root: tree_hash == v.root_digest? else corruption.

`Verity::compute_hash(salt, data)`:
1. shash_init(&desc, v.tfm[per-CPU]).
2. shash_update(&desc, salt, salt_size).
3. shash_update(&desc, data, block_size).
4. shash_final(&desc, hash_out).

`DmVerityFec::decode(io, block)`:
1. Read per-block FEC parity.
2. Attempt RS-decode.
3. If success: replace corrupted data; reverify.
4. Else: escalate to verity_handle_err.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `block_idx_bounded` | OOB | per-block idx < data_blocks. |
| `hash_level_block_no_oob` | OOB | per-level block index within hash_dev capacity. |
| `digest_size_bounded` | INVARIANT | digest_size ≤ MAX_DIGEST_SIZE (typically 64 bytes). |
| `verity_handle_err_terminal` | INVARIANT | corruption → bio_io_error AND no fall-through path returning success. |
| `tfm_per_cpu_unique` | INVARIANT | per-CPU tfm pool; defense against shared-state races. |

### Layer 2: TLA+

`drivers/md/verity_merkle_walk.tla`:
- Per-block walk: leaf → root.
- Properties:
  - `safety_walk_terminates_at_root` — walk eventually compares against root_digest.
  - `safety_corruption_detected` — any tampered hash anywhere in walk → corruption flagged.
  - `safety_no_false_positive` — un-tampered data passes verification.

`drivers/md/verity_fec_recovery.tla`:
- Per-block FEC-decode lifecycle.
- Properties:
  - `safety_fec_repair_within_threshold` — FEC repairs up to N corrupted bytes; beyond → escalate.
  - `safety_post_repair_verify` — repaired data re-verified before delivery.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Verity::map_bio` post: read submitted; verify queued | `Verity::map_bio` |
| `Verity::verify_io` post: per-block hash matched root_digest OR corruption flagged | `Verity::verify_io` |
| `Verity::compute_hash` post: hash_out filled with deterministic value for (salt, data) | `Verity::compute_hash` |
| `DmVerityFec::decode` post: corruption repaired OR escalate | `DmVerityFec::decode` |

### Layer 4: Verus/Creusot functional

`Per-block: read returns data iff data_block hash chain matches root_digest` semantic equivalence: per-block tampered data detected; un-tampered passes.

## Hardening

(Inherits row-1 features from `drivers/md/dm-core.md` § Hardening.)

dm-verity-specific reinforcement:

- **Per-block hash verified before bio_endio** — defense against corrupted-data return.
- **Per-target root_digest signature verified** — defense against root-hash tampering.
- **Per-CPU tfm pool** — defense against shared-cipher-state contention + timing.
- **Per-mode RESTART/PANIC for high-assurance** — defense against silent-corruption escape.
- **FEC threshold strict** — defense against FEC-overcorrection accepting tampered data.
- **Per-block salt** — defense against pre-computed rainbow-table attacks.
- **Per-target tfm validated against algo_name** — defense against attacker-supplied weak algo.
- **Hash-dev separation** — defense against logical-locality attacks.
- **Loadpin LSM gating** — defense against loading non-verified kernel modules.
- **Per-bio io_pool capped** — defense against allocation-flood under verification load.
- **Workqueue isolation** — defense against verification work blocking other targets.
- **memzero_explicit for tfm key in dtr** (when applicable) — defense against post-suspend RAM-leak.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — Merkle hash blocks are read into dm-bufio's `dm_buffer` slab (whitelisted for IO); the per-bio `verity_io` and its trailing hash-result digest live in a fixed-size `mempool_t` so `copy_to_user` paths only see exact-sized verified data, never tail-padding.
- **PAX_KERNEXEC** — `verity_target_type`, the per-format Merkle/FEC validator tables, and the chosen `crypto_ahash` `crypto_alg` vtable are all in `__ro_after_init` rodata.
- **PAX_RANDKSTACK** — `verity_map` and the workqueue handler `verity_work` cross the randomized kstack offset before invoking the async hash transform.
- **PAX_REFCOUNT** — `dm_verity->root_digest` refcount, per-bio `verity_io->in_bh` accounting, and `verity_fec_io->refs` are refcount_t; a flood of failed-block FEC recovery requests saturates rather than wraps.
- **PAX_MEMORY_SANITIZE** — completed `verity_io` slabs are zeroed before mempool return so per-bio digest material does not bleed into the next user's request.
- **PAX_UDEREF** — table constructor `verity_ctr` parses the salt, root digest, and (optional) signature blob from kmalloc'd `argv`, with each hex/byte string validated for length and charset before being copied into kernel storage.
- **PAX_RAP / kCFI** — `crypto_ahash_*` callbacks (`init`, `update`, `final`, `digest`) and `dm_target_type` callbacks are typed indirect calls; sites violating the type trap at the call edge.
- **fs-verity / Merkle-root signature verification** — when built with `CONFIG_DM_VERITY_VERIFY_ROOTHASH_SIG`, the optional `root_hash_sig_key_desc` argument forces the root hash to be verified against a PKCS#7 signature loaded into the `.builtin_trusted_keys` (or `.secondary_trusted_keys`) keyring before `verity_dtr` accepts the table; activation fails closed if the signature is missing, expired, or signed by an unknown key.
- **GRKERNSEC_TPE** — combined with Trusted Path Execution, only files under a verity-backed and signature-validated mount are exec'd by root and untrusted users; `verity_dtr` is the trust anchor for that TPE policy.
- **GRKERNSEC_HIDESYM / GRKERNSEC_DMESG** — root-hash, salt, and bufio buffer pointers are masked under `kptr_restrict ≥ 2`; corruption / FEC-correction events log at `KERN_CRIT` with `__ratelimit` and forward to audit.
- **fs-verity Merkle-root signature verify** — `dm-verity-verify-sig.c` invokes `verify_pkcs7_signature` against the kernel trusted-keys keyring before activation; signature must chain to a key in `.builtin_trusted_keys`/`.secondary_trusted_keys` or `.platform`, otherwise `verity_ctr` returns `-EKEYREJECTED`.
- **Restart-mode policy** — `restart_on_corruption` / `panic_on_corruption` modes are operator-selected at table-load time; once corruption is detected, the target either restarts the system or panics rather than serving silently corrupted data.
- **Pre-fetch hash window discipline** — the optional `prefetch_cluster` is clamped against `max_hash_block_index` so a malicious table cannot trigger out-of-bounds prefetch on the metadata device.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- dm-core (covered in `dm-core.md` Tier-3)
- dm-integrity (covered in `dm-integrity.md` Tier-3; per-write integrity vs per-read verity)
- FS-verity (per-file similar; covered in `fs/verity.md` future Tier-3)
- Crypto API (covered in `crypto/00-overview.md`)
- Implementation code
