---
title: "Subsystem: crypto/ — kernel cryptographic API"
tags: ["design-doc", "subsystem"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-2 overview for `crypto/` — the kernel cryptographic API. Owns the algorithm registration framework (`crypto_register_*` / `crypto_alloc_*`), every kernel-side cryptographic primitive (block ciphers, AEAD, hashes, MACs, KDFs, public-key, RNG, compression), the userspace-accessible `AF_ALG` socket interface, asymmetric-key handling (X.509 + PKCS#7 + signature verification), and the modular hardware-acceleration plumbing.

Adjacent: `lib/crypto/` (covered in `lib/00-overview.md`) holds *standalone* lightweight crypto primitives consumed by other lib code — there's no full crypto-API registration. `arch/x86/crypto/` (`arch/x86/00-overview.md` § crypto-accel.md) holds the x86 SIMD-accelerated implementations that register *into* this subsystem. The boundary is: `crypto/` is the API + generic implementations; `arch/<arch>/crypto/` are the hardware-accelerated variants that register at runtime.

### Requirements

- REQ-1: Every cryptographic algorithm upstream registers must be registered by Rookery under the same name + with the same priority.
- REQ-2: For each algorithm, output bytes are bit-identical given the same input + key + IV. (Algorithms are spec-defined — this is a strong invariant.)
- REQ-3: AF_ALG socket protocol preserves wire format and sockopts; existing userspace (cryptsetup, kcapi-tool, custom AF_ALG consumers) works unmodified.
- REQ-4: `/proc/crypto` format is byte-identical.
- REQ-5: Asymmetric key handling: X.509 / PKCS#7 / DER / PEM parsing preserves byte-identity for signature verification; module signing (kernel/module-loading.md) and kexec verification (kernel/crash-kexec.md) work unmodified.
- REQ-6: FIPS mode (`fips=1`) restricts the algorithm set identically and runs identical self-tests at boot.
- REQ-7: PQC (post-quantum) algorithms (ML-KEM-512/768/1024, ML-DSA-44/65/87, SLH-DSA-128/192/256-{S,F}) are registered if upstream registers them at the baseline.
- REQ-8: HW-accelerated implementations registered by `arch/x86/crypto/` (AES-NI, SHA-NI, AVX assists) are preferred (higher priority) over generic implementations identically to upstream.
- REQ-9: All Tier-3 docs declare verification-stack details. **`lightweight-crypto.md` of `lib/00-overview.md` (chacha20, poly1305, blake2s, libsha256) AND `crypto/00-overview.md` opt into Layer-4 functional-correctness proofs (Verus or Creusot)** — algorithms are textbook-defined, proofs of "this Rust implementation computes this RFC's algorithm" are tractable and high-value.
- REQ-10: The implementation reuses upstream rust-for-linux's existing crypto abstractions where they exist (`rust/kernel/digest.rs` and similar).

### Acceptance Criteria

- [ ] AC-1: A diff between Rookery's `/proc/crypto` and upstream's on the same `.config` is empty (modulo priorities of arch-accel implementations on different hardware). (covers REQ-1, REQ-4)
- [ ] AC-2: A test-vector harness exercises every registered algorithm against NIST-CAVP / IETF-test-vectors / per-algorithm reference vectors and reports byte-identical outputs. (covers REQ-2)
- [ ] AC-3: `cryptsetup` open/close cycle on a LUKS2 volume formatted under upstream succeeds on Rookery. (covers REQ-3)
- [ ] AC-4: `kcapi-tool` AF_ALG smoke tests pass identically. (covers REQ-3)
- [ ] AC-5: Module signing test: a kernel module signed with a key registered via X.509 loads correctly. (covers REQ-5)
- [ ] AC-6: A FIPS-mode boot (`fips=1`) registers only FIPS-approved algorithms and runs the same self-test set as upstream. (covers REQ-6)
- [ ] AC-7: PQC test vectors (ML-KEM, ML-DSA, SLH-DSA) round-trip correctly. (covers REQ-7)
- [ ] AC-8: AES-NI and SHA-NI implementations (registered by arch/x86/crypto/) appear in `/proc/crypto` with the same higher priority as upstream. (covers REQ-8)
- [ ] AC-9: Layer-4 proofs for at least chacha20, poly1305, blake2s, libsha256, hmac(sha256), aes (the v0 critical-path algorithms) compile and verify under Verus or Creusot. (covers REQ-9)
- [ ] AC-10: A grep over Rookery for `kernel::crypto::*` shows reuse of upstream rust-for-linux abstractions. (covers REQ-10)

### Architecture

### Layout map

```
.design/crypto/
  00-overview.md           ← this document
  api.md                   ← registration + alloc + cryptd + crypto_engine + /proc/crypto
  af-alg.md                ← AF_ALG userspace interface
  skcipher/
    00-overview.md         ← block ciphers hub
    aes.md, des.md, chacha.md, blowfish.md, cast.md, camellia.md, serpent.md, twofish.md, aria.md, sm4.md
    modes.md               ← cbc, ctr, ecb, ofb, cfb, xts, lrw, adiantum, hctr2
  aead.md                  ← AEAD: gcm, ccm, chacha20poly1305, authenc, authencesn
  hash.md                  ← shash + ahash + sha1/sha256/sha512/sha3/md4/md5/blake2/sm3/streebog/hmac/cmac + polyval/ghash/poly1305 + KDFs
  asymmetric.md            ← akcipher + sig + kpp + dh + ecdh + rsa + ecdsa + curve25519 + asymmetric_keys/ + PQC
  compression.md           ← acompress + scompress + lz4 + lzo + deflate + zstd + 842
  rng.md                   ← drbg + jitterentropy
  async-tx.md              ← async_tx/ DMA-engine offload (legacy)
  fips.md                  ← FIPS mode
```

### Cross-references

- `lib/00-overview.md` § `lightweight-crypto.md` — lightweight primitives (chacha20, poly1305, blake2s, libsha256) used internally; the corresponding crypto/ alg registrations are this subsystem's wrappers around the lib/crypto implementations.
- `arch/x86/00-overview.md` § `crypto-accel.md` — x86 SIMD accelerators (AES-NI, SHA-NI, AVX) register into this subsystem at higher priorities.
- `fs/00-overview.md` § `fs-crypto.md`, `fs-verity.md` — consume the crypto API.
- `net/00-overview.md` § `tls.md`, `xfrm.md` — consume the crypto API.
- `kernel/00-overview.md` § `module-loading.md`, `crash-kexec.md` — consume asymmetric-keys for signature verification.
- `security/00-overview.md` § `keys.md` (when written) — keyring infrastructure.
- `00-glossary.md` — `crypto_alg`, `LSM`, `kernel/sched`.

### Rust module organization (informative)

- `kernel::crypto::api` — registration core
- `kernel::crypto::skcipher`, `kernel::crypto::aead`, `kernel::crypto::hash`, `kernel::crypto::akcipher`, `kernel::crypto::compression`, `kernel::crypto::rng` — Rookery to author
- `kernel::crypto::af_alg` — Rookery to author

### Locking and concurrency

`crypto/` registers algorithms into a global `crypto_alg_list` protected by `crypto_alg_sem` (rwsem). Algorithm allocation/use after registration is mostly RCU-protected reads + per-instance refcounts.

### Error handling

Crypto errors: EINVAL (bad parameter), EBADMSG (bad ciphertext / authentication failure for AEAD), ENOKEY, EAGAIN (async crypto in flight), EOPNOTSUPP, EBUSY.

### Out of Scope

- Userspace TLS (OpenSSL etc.) — the kernel only offloads TLS record-layer; handshake stays userspace.
- 32-bit-only paths.
- Algorithms removed from upstream during v0 (e.g., deprecated MD4 may be dropped).
- Implementation code — `.design/` contains specs only.

### upstream references in scope

`crypto/` (~150 .c files + 2 subdirs at baseline). Categorized:

| Category | Upstream paths | Planned Tier-3 doc |
|---|---|---|
| API core: registration + alloc + dispatch | `crypto/api.c`, `crypto/algapi.c`, `crypto/algboss.c`, `crypto/proc.c` (`/proc/crypto`) | `api.md` |
| AF_ALG userspace interface | `crypto/af_alg.c`, `crypto/algif_aead.c`, `crypto/algif_hash.c`, `crypto/algif_rng.c`, `crypto/algif_skcipher.c`, `crypto/cryptouser.c`, `include/uapi/linux/cryptouser.h` | `af-alg.md` |
| Block ciphers (skcipher) | `crypto/skcipher.c`, `crypto/cipher.c`, plus impls (cbc, ctr, ecb, ofb, cfb, xts, lrw, adiantum, hctr2, salsa20, serpent, twofish, blowfish, cast{5,6}, camellia, aria, arc4, des, aes, sm4, anubis, fcrypt, khazad, seed, tea, xtea) | `skcipher/00-overview.md` (Tier 3 hub) |
| AEAD | `crypto/aead.c`, `crypto/authenc.c`, `crypto/authencesn.c`, `crypto/ccm.c`, `crypto/gcm.c`, `crypto/chacha20poly1305.c` | `aead.md` |
| Hashes (shash + ahash) | `crypto/shash.c`, `crypto/ahash.c`, plus impls (sha1, sha256, sha512, sha3, md4, md5, hmac, cmac, xcbc, vmac, crc{32,32c}, blake2b, blake2s, sm3, streebog, ghash, polyval, poly1305) | `hash.md` |
| Public-key + asymmetric | `crypto/asymmetric_keys/`, `crypto/akcipher.c`, `crypto/sig.c`, `crypto/kpp.c` (key pair primitive: DH, ECDH), `crypto/dh.c`, `crypto/ecdh.c`, `crypto/rsa.c`, `crypto/ecdsa.c`, `crypto/curve25519.c`, plus PQC (ML-KEM, ML-DSA, SLH-DSA in 7.x) | `asymmetric.md` |
| Compression | `crypto/acompress.c`, `crypto/scompress.c`, `crypto/lz4.c`, `crypto/lzo.c`, `crypto/deflate.c`, `crypto/zstd.c`, `crypto/842.c` | `compression.md` |
| RNG | `crypto/rng.c`, `crypto/drbg.c` (NIST SP 800-90A DRBG), `crypto/jitterentropy.c` | `rng.md` |
| Async TX (legacy DMA-engine offload) | `crypto/async_tx/` | `async-tx.md` |
| Cryptd (deferred-execution wrapper) | `crypto/cryptd.c`, `crypto/crypto_engine.c` | folded into `api.md` |
| BPF crypto | `crypto/bpf_crypto_skcipher.c` (BPF program access to skcipher) | folded into `bpf-net.md` cross-ref |
| KDF | `crypto/hkdf.c`, `crypto/kdf_sp800108.c`, `crypto/skcipher_kdf.c` | folded into `hash.md` |
| FIPS-mode discipline | `crypto/fips.c` | `fips.md` |

### compatibility contract

### `/proc/crypto` and AF_ALG

| Surface | Owner doc | Compat level |
|---|---|---|
| `/proc/crypto` content (every registered alg) | `api.md` | Format-identical |
| `AF_ALG` socket protocol | `af-alg.md` | Wire-identical |
| `AF_ALG` `setsockopt(ALG_SET_KEY/AUTH_SIZE)` etc. | `af-alg.md` | Identical |

### Userspace-visible algorithm names

The set of registered cryptographic algorithm names (`aes`, `cbc(aes)`, `gcm(aes)`, `hmac(sha256)`, `xts(aes)`, etc.) and their priorities determine what userspace via AF_ALG, fscrypt, tcrypt, IPsec/xfrm, dm-crypt, dm-verity see. **Every algorithm-name string upstream registers MUST be registered identically by Rookery.** Removing an alg breaks compat for any tool that requested it by name.

### Asymmetric keys API

X.509 + PKCS#7 + signature verification ABI consumed by:
- Module signing (`kernel/module/`)
- Kexec image verification (`kernel/kexec_*`)
- IMA / EVM (`security/integrity/`)
- DM-verity (`drivers/md/`)

Wire formats (DER, PEM) are RFC-defined; preserved.

### FIPS mode

`fips=1` boot parameter: limits algorithm set to FIPS-approved + adds self-tests. Preserved.

### verification

### Layer 1: Kani SAFETY proofs
- Cipher block-loop pointer arithmetic
- Hash buffer-bound writes
- AEAD authentication tag construction (constant-time-equal comparison)

### Layer 2: TLA+ models
- `models/crypto/cryptd_async.tla` — proves async-crypto deferred-completion semantics: the requester always observes either the result OR a cancel/error, never an inconsistent state.

### Layer 3: Kani harnesses
- Algorithm-priority ordering invariants (`/proc/crypto` listing matches priority sort)
- AF_ALG socket state machine

### Layer 4: Functional correctness (MANDATORY for v0 critical-path algorithms per REQ-9)

- chacha20 — Verus
- poly1305 — Verus (constant-time aspects via Creusot)
- chacha20poly1305 (combined AEAD) — Verus
- blake2s — Verus
- libsha256 — Verus
- hmac(sha256) — Verus
- aes-128/256 (in `lib/crypto/` if present, or this subsystem's generic AES) — Verus
- ecdsa P-256, curve25519 X25519 — Creusot

These six are the hot path for fscrypt + WireGuard + dm-verity + module signing. Proving them is high-leverage.

### hardening

Placeholder per `00-overview.md` D6. Notable: constant-time discipline is mandated by the deferred `00-security-principles.md`. Per-algorithm Rust implementations must avoid data-dependent branches and table lookups (use bitslicing or hardware-accelerated code paths).

