# Tier-3: crypto/api — kernel crypto API core (registration + alloc + dispatch)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - crypto/api.c
  - crypto/algapi.c
  - crypto/algboss.c
  - crypto/proc.c
  - crypto/cryptd.c
  - crypto/crypto_engine.c
  - crypto/crypto_user.c
  - include/linux/crypto.h
  - include/crypto/algapi.h
  - include/uapi/linux/cryptouser.h
-->

## Summary
Tier-3 design for the kernel crypto API core: the registration framework that lets cryptographic algorithms be plugged in (`crypto_register_alg` family), the algorithm-instance allocator (`crypto_alloc_*`), the algorithm-name parser + template instantiation (e.g., `cbc(aes)` instantiates from `cbc` template + `aes` cipher), `cryptd` (thread-deferred crypto), `crypto_engine` (driver-side request queueing), `/proc/crypto` (registered-algorithm enumeration), and `crypto_user` netlink interface for inspection.

Sub-tier-3 of `crypto/00-overview.md`. Every fs-crypto, IPsec, dm-crypt, dm-verity, kTLS, AF_ALG, BPF-skcipher, and module-signing operation routes through this Tier-3.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Algorithm registration + alloc + dispatch | `crypto/api.c`, `crypto/algapi.c` |
| Late-binding algorithm boss (kthread that resolves driver-name strings) | `crypto/algboss.c` |
| `/proc/crypto` enumeration | `crypto/proc.c` |
| cryptd (thread-deferred crypto for async-incapable callers) | `crypto/cryptd.c` |
| crypto_engine (driver-side request queueing for HW accelerators) | `crypto/crypto_engine.c` |
| crypto_user netlink (inspect via NETLINK_CRYPTO) | `crypto/crypto_user.c` |
| Public types | `include/linux/crypto.h`, `include/crypto/algapi.h` |
| UAPI | `include/uapi/linux/cryptouser.h` |

## Compatibility contract

### `/proc/crypto`

Format-identical: per-registered-alg block with `name`, `driver`, `module`, `priority`, `refcnt`, `selftest`, `internal`, `type`, `blocksize`, `min keysize`, `max keysize`, `ivsize`, `geniv`, `chunksize`, `walksize` (and per-alg-type-specific fields). Existing tools (kernel-cap-utils, openssl-via-AF_ALG) parse identically.

### Algorithm-name resolution

Userspace + kernel callers request algorithms by **template-name strings**: `cbc(aes)`, `gcm(aes)`, `hmac(sha256)`, `xts(aes)`, `chacha20poly1305(rfc7539)`, `authenc(hmac(sha256),cbc(aes))`. The crypto API resolves these by:
1. Look up the literal name in the registered-alg list
2. If not found, parse as `template(inner)`, recursively resolve inner
3. Instantiate template with inner

Identical to upstream so existing callers' name strings work unchanged.

### Priority + selection

When multiple algorithms register the same name (e.g., generic `aes-generic` priority 100 vs. AES-NI `aes-aesni` priority 300), the highest-priority is selected. Identical priority assignment so AES-NI on capable CPUs always wins over generic.

### `crypto_user` netlink

NETLINK_CRYPTO message types per `include/uapi/linux/cryptouser.h`: `CRYPTO_MSG_NEWALG`, `CRYPTO_MSG_DELALG`, `CRYPTO_MSG_UPDATEALG`, `CRYPTO_MSG_GETALG`, `CRYPTO_MSG_DELRNG`, `CRYPTO_MSG_GETSTAT`. Wire format byte-identical so userspace `crconf` works unmodified.

### `struct crypto_alg` layout

`include/linux/crypto.h`: per-registered-alg state. First-cache-line + commonly-accessed fields layout-equivalent. Fields: `cra_list`, `cra_users`, `cra_flags`, `cra_blocksize`, `cra_ctxsize`, `cra_alignmask`, `cra_priority`, `cra_refcnt`, `cra_name[CRYPTO_MAX_ALG_NAME]`, `cra_driver_name[CRYPTO_MAX_ALG_NAME]`, `cra_type`, `cra_init`, `cra_exit`, `cra_destroy`, `cra_module`, `cra_u` (union of per-type ops).

## Requirements

- REQ-1: `struct crypto_alg` first-cache-line layout-equivalent.
- REQ-2: Algorithm registration: `crypto_register_alg`, `crypto_register_aead`, `crypto_register_skcipher`, `crypto_register_shash`, `crypto_register_ahash`, `crypto_register_akcipher`, `crypto_register_kpp`, `crypto_register_rng`, `crypto_register_acomp`, `crypto_register_scomp` semantics identical.
- REQ-3: Algorithm-name resolution: literal lookup + template-instantiation recursive parse identical to upstream.
- REQ-4: Priority + selection: highest-priority alg wins on `crypto_alloc_*`; ties broken per upstream's tiebreaker (registration order).
- REQ-5: Algorithm allocation: `crypto_alloc_aead`, `crypto_alloc_skcipher`, `crypto_alloc_shash`, `crypto_alloc_ahash`, `crypto_alloc_akcipher`, `crypto_alloc_kpp`, `crypto_alloc_rng`, `crypto_alloc_acomp` semantics identical.
- REQ-6: Self-tests: `CRYPTO_ALG_NEED_FALLBACK`, `CRYPTO_ALG_TESTED` flags + per-alg test-vector machinery (`tcrypt`) identical.
- REQ-7: cryptd: per-CPU `cryptd_queue` for thread-deferred sync→async wrapping. Identical.
- REQ-8: crypto_engine: per-driver request queueing for HW accelerators; semantics identical.
- REQ-9: `/proc/crypto` content format-identical for any equivalent algorithm set.
- REQ-10: NETLINK_CRYPTO wire format byte-identical.
- REQ-11: FIPS mode (`fips=1`): when enabled, alg registration honors FIPS-compliance flag; non-compliant algs fail registration with EFIPSALG (cross-ref `crypto/fips.md` Tier-3).
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A diff of `cat /proc/crypto` between Rookery + upstream after equivalent boot is empty (modulo dynamically-loaded modules). (covers REQ-9)
- [ ] AC-2: `pahole struct crypto_alg` byte-identical first-cache-line. (covers REQ-1)
- [ ] AC-3: Every documented `crypto_register_*` API succeeds for a representative algorithm; its priority + selection matches upstream's. (covers REQ-2, REQ-4)
- [ ] AC-4: A name-resolution test: `crypto_alloc_skcipher("cbc(aes)", 0, 0)` resolves via cbc template + aes cipher; `crypto_alloc_aead("authenc(hmac(sha256),cbc(aes))", ...)` resolves recursively. (covers REQ-3)
- [ ] AC-5: An AES-NI-capable CPU + an AES-NI-registered alg + a generic AES alg → `crypto_alloc_skcipher("aes", ...)` returns the AES-NI driver (verifiable via `crypto_tfm_alg_driver_name`). (covers REQ-4)
- [ ] AC-6: tcrypt self-test runs every documented test vector; pass/fail matches upstream. (covers REQ-6)
- [ ] AC-7: A cryptd-wrapped sync skcipher invoked from atomic context defers to a cryptd kthread; result is correct. (covers REQ-7)
- [ ] AC-8: A crypto_engine-using HW accelerator (e.g., emulated via test driver) queues + completes requests in order. (covers REQ-8)
- [ ] AC-9: `crconf` netlink-list output byte-identical. (covers REQ-10)
- [ ] AC-10: Boot with `fips=1`; non-FIPS-compliant alg registration returns EFIPSALG. (covers REQ-11)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::crypto::alg::Alg` — `struct crypto_alg` wrapper
- `kernel::crypto::api::Registry` — alg registration + lookup
- `kernel::crypto::api::resolve::NameResolver` — template-instantiation parser
- `kernel::crypto::api::alloc::Allocator` — per-type alloc helpers
- `kernel::crypto::cryptd::Cryptd` — thread-deferred crypto
- `kernel::crypto::engine::CryptoEngine` — HW accelerator request queue
- `kernel::crypto::user::CryptoUserNetlink` — NETLINK_CRYPTO
- `kernel::crypto::proc::ProcCrypto` — /proc/crypto
- `kernel::crypto::test::TestVectors` — tcrypt + per-alg self-test

### Locking and concurrency

- **`crypto_alg_sem`** (rwsem): per-system alg-list mutator; writers register/unregister; readers walk
- **Per-alg `cra_refcnt`** (saturating Refcount): per-alg refcount on instances
- **Per-cryptd-queue spinlock**: cryptd dispatch
- **Per-engine queue spinlock**: crypto_engine

### Error handling

- `Err(ENOENT)` — alg not found
- `Err(ENOMEM)` — alloc failed
- `Err(EINVAL)` — bad alg name / bad priority
- `Err(EAGAIN)` — async crypto in flight
- `Err(EBUSY)` — alg in use during deregister
- `Err(EBADMSG)` — bad ciphertext / authentication failure (AEAD)
- `Err(ENOKEY)` — no key set

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Alg registration list manipulation under crypto_alg_sem | `kani::proofs::crypto::api::register_safety` |
| Template-instantiation parser | `kani::proofs::crypto::api::resolve_safety` |
| Per-alg refcount transitions | `kani::proofs::crypto::api::refcount_safety` |
| cryptd dispatch | `kani::proofs::crypto::cryptd::dispatch_safety` |
| crypto_user netlink message parse | `kani::proofs::crypto::user::netlink_parse_safety` |

### Layer 2: TLA+ models

- `models/crypto/cryptd_async.tla` (mandatory per `crypto/00-overview.md` Layer 2) — proves async-crypto deferred-completion semantics: requester always observes either result OR cancel/error, never inconsistent state. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Algorithm priority ordering | /proc/crypto listing matches priority sort within same alg-name group | `kani::proofs::crypto::api::priority_invariants` |
| Template-instantiation cache | Each instantiated template appears at most once in the registry per (template, inner) pair | `kani::proofs::crypto::api::template_cache_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `crypto/00-overview.md` Layer 4)

- **Algorithm name parser** via Creusot — proves: parser yields correct (template, inner) tree for valid inputs; rejects invalid syntax with EINVAL.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | cra_refcnt + per-template refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-template + per-instance allocation via per-type slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed alg objects zeroed (especially relevant for caches that hold key state) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: per-alg ops vtables provided by alg modules are `static const`
- **SIZE_OVERFLOW**: alg-priority + name-length arithmetic uses checked operators

### Row-2 / GR-RBAC integration

LSM hooks: `security_socket_bind` for AF_ALG (cross-ref `crypto/af-alg.md`); `security_keyctl` for asymmetric-key access. GR-RBAC's policy can deny crypto API operations.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — crypto API semantics are exhaustively specified by upstream)

## Out of Scope

- Per-algorithm implementations (cross-ref `crypto/skcipher/`, `crypto/aead.md`, `crypto/hash.md`, etc.)
- 32-bit-only paths
- Implementation code
