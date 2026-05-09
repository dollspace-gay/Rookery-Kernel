---
title: "Tier-3: net/xfrm/algo — XFRM cipher / authenticator / compression algorithm registry"
tags: ["design-doc", "tier-3", "net", "ipsec"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the XFRM algorithm registry: catalog of all IPSec ciphers (`xfrm_ealg_list`), authenticators (`xfrm_aalg_list`), AEAD ciphers (`xfrm_aead_list`), and compressors (`xfrm_calg_list`) recognized by the IKE-userspace daemons + kernel crypto API. Each entry maps an algorithm string name (e.g., `"aes"`, `"hmac(sha256)"`, `"rfc4106(gcm(aes))"`, `"deflate"`) and SADB algorithm number (per RFC 4305 + RFC 8221 + RFC 7321 + IANA registry) to per-algorithm capability descriptors (key-size range, IV-size, ICV-size, block-size, status — VALID/EXPIRED/DEPRECATED).

Provides the helper functions `xfrm_ealg_get_byname` / `xfrm_aalg_get_byid` / etc. that NETLINK_XFRM and PF_KEYv2 use to validate algorithm references in NEWSA / GETSA messages.

Sub-tier-3 of `net/xfrm/00-overview.md`. The data layer; per-algorithm crypto implementations live in `crypto/` (cross-ref `crypto/00-overview.md`). Pairs with `net/xfrm/state.md` (consumes `xfrm_aalg`/`xfrm_ealg`/`xfrm_aead`/`xfrm_calg` per-SA), `net/xfrm/user.md` (validates user-supplied algorithm refs).

### Requirements

- REQ-1: `struct xfrm_algo_desc` byte-identical layout.
- REQ-2: Catalog content (per the table above) byte-identical to upstream's `xfrm_ealg_list` / `xfrm_aalg_list` / `xfrm_aead_list` / `xfrm_calg_list`.
- REQ-3: Per-class `_get_byname` / `_get_byid` / `_get_byidx` semantics identical.
- REQ-4: SADB-num ↔ name mapping per RFC 4305 + RFC 8221 + IANA registry.
- REQ-5: `available` flag computed from kernel crypto API's `crypto_has_*` at registry init + on module load/unload.
- REQ-6: `pfkey_supported` flag set per IANA SADB-num assignment.
- REQ-7: FIPS mode (`fips=1`): DEPRECATED algorithms marked `available = 0`; subsequent `_get_*` returns the descriptor but caller observes 0 → returns -EINVAL.
- REQ-8: Algorithm-name parser strips template syntax (e.g., `"hmac(sha256)"` parsed identically to upstream so kernel crypto API lookup matches).
- REQ-9: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct xfrm_algo_desc` byte-identical. (covers REQ-1)
- [ ] AC-2: Catalog content test: enumerate all 4 lists; algorithm count + names byte-identical to upstream's. (covers REQ-2)
- [ ] AC-3: SADB-num lookup test: `xfrm_aalg_get_byid(2)` returns the descriptor for `hmac(sha1)`; reverse `_get_byname("hmac(sha1)")` returns the descriptor with `desc.sadb_alg_id == 2`. (covers REQ-3, REQ-4)
- [ ] AC-4: `available` flag test: `aes-cbc` has `available = 1` on a kernel where `crypto/aes-generic.ko` is loadable; an unsupported algorithm has `available = 0`. (covers REQ-5)
- [ ] AC-5: PF_KEYv2 supported test: `rfc4106(gcm(aes))` AEAD has `pfkey_supported = 0` (post-PF_KEYv2 addition); `aes-cbc` has `pfkey_supported = 1`. (covers REQ-6)
- [ ] AC-6: FIPS mode test: boot with `fips=1`; `hmac(md5)` has `available = 0`; user attempt to install SA with `auth-trunc 'hmac(md5)' ...` returns EINVAL. (covers REQ-7)
- [ ] AC-7: Template-name parse test: `"hmac(sha256)"` resolves to the same descriptor whether queried via setkey (PF_KEYv2 SADB-id 5) or `ip xfrm` (NETLINK_XFRM by name). (covers REQ-8)
- [ ] AC-8: Hardening section present and follows template. (covers REQ-9)

### Architecture

### Rust module organization

- `kernel::net::xfrm::algo::AlgoRegistry` — registry root
- `kernel::net::xfrm::algo::EalgList` — encryption algorithm list
- `kernel::net::xfrm::algo::AalgList` — authentication algorithm list
- `kernel::net::xfrm::algo::AeadList` — AEAD algorithm list
- `kernel::net::xfrm::algo::CalgList` — compression algorithm list
- `kernel::net::xfrm::algo::AlgoDesc` — `struct xfrm_algo_desc`
- `kernel::net::xfrm::algo::Lookup` — `_get_byname/byid/byidx` helpers
- `kernel::net::xfrm::algo::Available` — `available` flag computation (consults kernel crypto API)
- `kernel::net::xfrm::algo::Fips` — FIPS-mode gate

### Locking and concurrency

- **`xfrm_algo_lock`** (mutex): held during `available` flag recomputation on crypto module load/unload
- **RCU**: lookup hot path is RCU-side
- **Lists are `static const` (almost)**: catalog itself is immutable; only `available` field mutable

### Error handling

- `Err(EINVAL)` — algorithm not in catalog (returned by callers when `_get_byname` returns NULL)
- `Err(EAGAIN)` — algorithm in catalog but `available = 0`

### Out of Scope

- Per-algorithm crypto implementations (cross-ref `crypto/00-overview.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Algorithm catalog: ealg/aalg/aead/calg lists + by-name/by-id helpers + SADB-num ↔ name mapping | `net/xfrm/xfrm_algo.c` |
| Public API | `include/net/xfrm.h` (struct xfrm_algo_desc) |
| UAPI | `include/uapi/linux/xfrm.h` (struct xfrm_algo, struct xfrm_algo_aead, struct xfrm_algo_auth, struct xfrm_algo_auth_dump_compat) |

### compatibility contract

### `struct xfrm_algo_desc`

```c
struct xfrm_algo_desc {
    char        name[CRYPTO_MAX_ALG_NAME];      /* kernel-crypto-API name */
    char        compat[CRYPTO_MAX_ALG_NAME];    /* PF_KEYv2 compat name */
    u8          available;
    u8          pfkey_supported;
    union {
        struct sadb_alg desc;                   /* PF_KEYv2 algorithm-descriptor (id, ivlen, minbits, maxbits) */
    } uinfo;
};
```

(Plus per-class extra fields: encryption descriptors carry `ealg.icv_truncbits`, AEAD `aead.icv_truncbits`, etc.)

Layout-byte-identical so iproute2 + setkey + strongSwan algorithm-name parsing work.

### Catalog content (current upstream)

Encryption (`xfrm_ealg_list`):
- `cipher_null` (id 11), `des` (id 2 — DEPRECATED RFC 8221), `des3_ede` (id 3 — LEGACY), `cast5` (id 6), `bf` (id 7 — Blowfish, LEGACY), `aes-cbc` (id 12), `aes-ctr` (id 13), `aes-ccm-icv8/12/16`, `aes-gcm-icv8/12/16`, `serpent` (id 252), `camellia-cbc/ctr/ccm/gcm` (id 22-29), `camellia-cbc-rfc3686`, `null-ecb`, `chacha20poly1305` (id 28).

Authentication (`xfrm_aalg_list`):
- `digest_null`, `hmac(md5)` (id 2 — DEPRECATED), `hmac(sha1)` (id 2 — also DEPRECATED for crypto, KEY-based PRF still used), `hmac(sha256)`, `hmac(sha384)`, `hmac(sha512)`, `cmac(aes)` (id 8), `xcbc(aes)`, `rmd160` (LEGACY), `aes-128-gmac`, `aes-192-gmac`, `aes-256-gmac`.

AEAD (`xfrm_aead_list`):
- `rfc4106(gcm(aes))` (RFC 4106 — AES-GCM for ESP), `rfc4309(ccm(aes))` (RFC 4309 — AES-CCM), `rfc4543(gcm(aes))` (RFC 4543 — AES-GMAC for ESP-AAD).

Compression (`xfrm_calg_list`):
- `deflate` (id 2), `lzs` (id 3 — LEGACY), `lzjh` (id 4 — LEGACY).

Identical catalog so existing strongSwan + libreswan + iproute2 algorithm references resolve.

### `xfrm_*_get_byname` / `xfrm_*_get_byidx` / `xfrm_*_get_byid`

| Helper | Returns |
|---|---|
| `xfrm_ealg_get_byname(name)` | descriptor by kernel-crypto-API name |
| `xfrm_ealg_get_byid(id)` | descriptor by SADB-num (RFC 4305) |
| `xfrm_ealg_get_byidx(idx)` | descriptor by walking-index |
| `xfrm_aalg_get_*`, `xfrm_aead_get_*`, `xfrm_calg_get_*` | analogous |

Each returns `NULL` on miss + sets `err` for the caller. Identical semantics so user-space algorithm validation succeeds/fails identically.

### Algorithm `available` flag

Per descriptor: `available = 1` when the underlying crypto module is registered + can allocate (per `crypto_has_*`). Otherwise the algorithm is enumerable (so userspace sees it in dumps) but not allocable.

### `pfkey_supported` flag

`pfkey_supported = 1` when the algorithm has a SADB-num assignment (per RFC 4305 + IANA); otherwise NETLINK_XFRM-only (e.g., AEAD ciphers were added post PF_KEYv2).

### Status flags (FIPS / DEPRECATED)

When CONFIG_FIPS is enabled (or `fips=1` boot param), DEPRECATED algorithms are marked `available = 0`. RFC 8221 retires `hmac(md5)`, `des`, `des3_ede`, `bf`, `cast5` (deprecation differs by class). Identical FIPS-mode gating.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Catalog list-walk (terminates, no out-of-bounds) | `kani::proofs::net::xfrm::algo::list_walk_safety` |
| `_get_byname` string-compare bounds (CRYPTO_MAX_ALG_NAME-clamped) | `kani::proofs::net::xfrm::algo::byname_safety` |
| `_get_byid` SADB-num lookup (no integer-promotion sign issues) | `kani::proofs::net::xfrm::algo::byid_safety` |
| `available` flag mutation under xfrm_algo_lock | `kani::proofs::net::xfrm::algo::available_safety` |

### Layer 2: TLA+ models

(none mandatory at this level)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Catalog lists | every entry has unique (name, compat) tuple within its list | `kani::proofs::net::xfrm::algo::uniqueness_invariants` |
| `pfkey_supported` flag | when `pfkey_supported = 1`, `desc.sadb_alg_id` is a valid IANA SADB-num | `kani::proofs::net::xfrm::algo::pfkey_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/xfrm/00-overview.md`)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **CONSTIFY** | catalog lists are `static const` (except `available` which is bookkeeping) | § Mandatory |

### Row-1 features consumed by this component

- **CONSTIFY**: see above
- **USERCOPY**: NLA parsing of algorithm names uses bound-checked accessors; CRYPTO_MAX_ALG_NAME-clamped string comparison
- **SIZE_OVERFLOW**: no — catalog has no arithmetic surface; key-size range checks happen in `net/xfrm/state.md`

### Row-2 / GR-RBAC integration

- LSM hook: none directly; algorithm catalog is read-only for userspace.
- Useful default GR-RBAC policy: empty so behavior matches upstream.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

