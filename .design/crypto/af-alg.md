# Tier-3: crypto/af-alg — userspace AF_ALG socket interface

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - crypto/af_alg.c
  - crypto/algif_aead.c
  - crypto/algif_hash.c
  - crypto/algif_rng.c
  - crypto/algif_skcipher.c
  - include/crypto/if_alg.h
  - include/uapi/linux/if_alg.h
-->

## Summary
Tier-3 design for AF_ALG, the userspace socket interface to the kernel crypto API. Userspace creates an AF_ALG socket, calls `bind()` with the algorithm-class + name (e.g., `aead`, `gcm(aes)`), `accept()`s an operation socket, optionally sets the key via `setsockopt(ALG_SET_KEY)`, then reads/writes ciphertext/plaintext.

Sub-tier-3 of `crypto/00-overview.md`. Exposed by tools like `kcapi`, `openssl`'s `KTLS` engine, and per-algorithm test suites. Often the only userspace path to FIPS-validated kernel crypto.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| AF_ALG core: socket family, bind/accept, ALG_SET_KEY/ALG_SET_AEAD_AUTHSIZE/ALG_SET_IV/ALG_SET_OP setsockopts | `crypto/af_alg.c` |
| Per-class wrappers | `crypto/algif_aead.c`, `crypto/algif_hash.c`, `crypto/algif_rng.c`, `crypto/algif_skcipher.c` |
| Public API | `include/crypto/if_alg.h` |
| UAPI | `include/uapi/linux/if_alg.h` (ALG_SET_KEY, ALG_OP_DECRYPT/ENCRYPT, sockaddr_alg) |

## Compatibility contract

### `socket(AF_ALG, SOCK_SEQPACKET, 0)`

Creates an AF_ALG control socket. Identical to upstream.

### `bind(fd, &sockaddr_alg, ...)`

`struct sockaddr_alg`: `salg_family=AF_ALG`, `salg_type[]` (e.g., `"aead"`, `"hash"`, `"rng"`, `"skcipher"`), `salg_name[]` (e.g., `"gcm(aes)"`, `"sha256"`, `"stdrng"`, `"cbc(aes)"`). After bind, the socket is associated with that algorithm.

Identical layout + semantics.

### `accept(fd, ...)`

Returns an operation-socket fd. Reading from / writing to this fd performs the algorithm operation. The underlying tfm + state are per-operation-socket; the control-socket can be re-`accept()`ed for parallel ops.

Identical.

### `setsockopt(fd, SOL_ALG, OP, ...)`

| Option | Semantics |
|---|---|
| `ALG_SET_KEY` | set encryption / MAC key |
| `ALG_SET_AEAD_AUTHSIZE` | AEAD authtag size |
| `ALG_SET_AEAD_ASSOCLEN` (via cmsg) | AEAD AAD length |
| `ALG_SET_IV` (via cmsg) | per-op IV |
| `ALG_SET_OP` (via cmsg) | ALG_OP_ENCRYPT / ALG_OP_DECRYPT |
| `ALG_SET_KEY_BY_KEY_SERIAL` | use a kernel keyring key |
| `ALG_SET_DRBG_ENTROPY` | DRBG entropy source |

Each value byte-identical to upstream so userspace `kcapi` works unmodified.

### `recvmsg` / `sendmsg`

The SCM cmsg model carries IV + ASSOCLEN + OP per-message. Format-identical so existing senders work.

### Per-class behavior

| Class | algif | Read returns |
|---|---|---|
| `skcipher` | algif_skcipher | encrypted/decrypted block(s) |
| `aead` | algif_aead | encrypted+authtag / decrypted (or EBADMSG if auth fails) |
| `hash` | algif_hash | digest (after writes are summed) |
| `rng` | algif_rng | random bytes |

Identical semantics.

## Requirements

- REQ-1: AF_ALG socket family register identical to upstream's family number.
- REQ-2: `struct sockaddr_alg` layout-identical (salg_family / salg_type / salg_feat / salg_mask / salg_name).
- REQ-3: bind(): resolves `salg_name` via `crypto_alloc_*` for the salg_type class; binds tfm to socket. Identical.
- REQ-4: accept(): allocates per-op state from bound tfm; returns operation socket. Identical.
- REQ-5: setsockopt SOL_ALG: each documented option works identically.
- REQ-6: cmsg per-op metadata: IV / ASSOCLEN / OP / DRBG_ENTROPY parsed identically.
- REQ-7: skcipher class: write plaintext, read ciphertext (and vice versa for decrypt); supports zerocopy `vmsplice` via `splice_to_pipe`.
- REQ-8: aead class: write AAD then plaintext; read ciphertext + tag (encrypt) or write ciphertext + tag, read plaintext or EBADMSG (decrypt).
- REQ-9: hash class: write data; read digest (commit on read).
- REQ-10: rng class: read returns random bytes.
- REQ-11: ALG_SET_KEY_BY_KEY_SERIAL: looks up key via kernel keyring; cross-ref `security/keys.md`.
- REQ-12: GR-RBAC LSM hook `security_socket_create` for AF_ALG: GR-RBAC policy can deny socket creation per-subject (cross-ref `security/00-overview.md`).
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `kcapi-enc -c "cbc(aes)" -k "00…00" -i "00…00" --` round-trips bytes identically. (covers REQ-3, REQ-7)
- [ ] AC-2: A test program creating AF_ALG aead socket + setting key via setsockopt + sending AAD+plaintext via sendmsg cmsg + reading ciphertext+tag → byte-identical to in-kernel via `crypto_aead_encrypt`. (covers REQ-2, REQ-5, REQ-6, REQ-8)
- [ ] AC-3: AEAD-decrypt of corrupted ciphertext returns EBADMSG. (covers REQ-8)
- [ ] AC-4: hash-class: write 1MB; read digest matches `sha256sum`. (covers REQ-9)
- [ ] AC-5: rng-class: read 1MB; entropy ≥ 7.99 bits/byte (NIST SP800-90B-style). (covers REQ-10)
- [ ] AC-6: A keyring-based key (added via `add_key`) is settable via ALG_SET_KEY_BY_KEY_SERIAL. (covers REQ-11)
- [ ] AC-7: With GR-RBAC policy denying AF_ALG `connect` for a subject, `socket(AF_ALG, ...)` returns EACCES; without policy, succeeds. (covers REQ-12)
- [ ] AC-8: zerocopy: `vmsplice(pipe, iov, ...) → splice(pipe, alg_socket, ...)` works without copy (verifiable via /proc/self/maps + perf). (covers REQ-7)
- [ ] AC-9: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::crypto::af_alg::AfAlg` — `AF_ALG` socket family, sockaddr_alg parser, bind/accept
- `kernel::crypto::af_alg::ctrl::CtrlSocket` — control socket (post-bind, pre-accept)
- `kernel::crypto::af_alg::op::OpSocket` — operation socket (post-accept)
- `kernel::crypto::af_alg::cmsg::CmsgParser` — per-class cmsg parser
- `kernel::crypto::af_alg::algif::skcipher::AlgifSkcipher`
- `kernel::crypto::af_alg::algif::aead::AlgifAead`
- `kernel::crypto::af_alg::algif::hash::AlgifHash`
- `kernel::crypto::af_alg::algif::rng::AlgifRng`

### Locking and concurrency

- **Per-control-socket mutex**: serializes bind / setsockopt
- **Per-op-socket mutex**: serializes recvmsg / sendmsg
- **Per-tfm refcount** (Refcount): inherited from `kernel::crypto::api`

### Error handling

- `Err(EAFNOSUPPORT)` — AF_ALG disabled
- `Err(ENOENT)` — algorithm not registered
- `Err(EINVAL)` — bad sockaddr_alg
- `Err(EBADMSG)` — AEAD authentication failed
- `Err(ENOKEY)` — no key set / keyring lookup failed
- `Err(EACCES)` — LSM denial

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| AF_ALG socket-family bind path | `kani::proofs::crypto::af_alg::bind_safety` |
| accept() path | `kani::proofs::crypto::af_alg::accept_safety` |
| cmsg parser per class | `kani::proofs::crypto::af_alg::cmsg_parse_safety` |
| Per-op send/recv | `kani::proofs::crypto::af_alg::op_safety` |

### Layer 2: TLA+ models

- inherits `models/crypto/cryptd_async.tla` (owned by `crypto/api.md`) — proves async-crypto deferred-completion semantics

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-op-socket state machine | follows allowed transitions only (post-accept → key-set → in-flight → result-ready) | `kani::proofs::crypto::af_alg::state_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `crypto/api.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** | per-op socket state cleared on close (especially key + IV memory) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB**: per-op-socket state slabs
- **MEMORY_SANITIZE**: see above
- **CONSTIFY**: per-class algif ops vtables `static const`
- **USERCOPY**: send/recvmsg payload movement uses `kernel::uaccess::copy_from_user`/`copy_to_user`

### Row-2 / GR-RBAC integration

- LSM hook `security_socket_create` (PF_ALG); GR-RBAC policy can deny AF_ALG socket creation per-subject (default policy: empty / allow).
- LSM hook `security_socket_bind` for sockaddr_alg
- LSM hook `security_socket_setsockopt` for ALG_SET_KEY (deny key-setting per policy if needed)

### Userspace-visible behavior changes

None beyond upstream defaults (GR-RBAC's default empty policy permits all AF_ALG ops).

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy across `setsockopt(ALG_SET_KEY)`, `sendmsg`/`recvmsg` payload, and cmsg parsers; rejects oversize key/IV/AAD writes.
- **PAX_KERNEXEC** — W^X enforcement across algif dispatch.
- **PAX_RANDKSTACK** — kernel-stack randomization on per-op syscall entry.
- **PAX_REFCOUNT** — saturating refcount on `AfAlg` control + operation sockets and bound `crypto_tfm` references.
- **PAX_MEMORY_SANITIZE** — zero-on-free for per-op socket state including key material, IV, AAD scratch, and keystream buffers; per-op-socket close zeroes `setsockopt` key buffer.
- **PAX_UDEREF** — SMAP/SMEP user-pointer access enforced on every `copy_from_user`/`copy_to_user` in cmsg parser + sendmsg/recvmsg.
- **PAX_RAP / kCFI** — per-class algif ops vtables (`algif_skcipher`, `algif_aead`, `algif_hash`, `algif_rng`) hardened against indirect-call hijack; control + operation socket ops `static const`.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding in any `/proc`-exposed AF_ALG state.
- **GRKERNSEC_DMESG** — syslog restriction on AF_ALG warnings (key-set failures, EBADMSG floods).
- **PAX_CONSTIFY_PLUGIN** — algif per-class ops vtables `static const`.
- **GRKERNSEC_CHROOT** — AF_ALG socket creation gated inside chroot.
- **GRKERNSEC_SYSCTL** — `net.crypto.alg.*` toggles locked at boot under GRKERNSEC_SYSCTL_ON.
- **CAP_NET_RAW** + LSM `security_socket_create(PF_ALG)` — per-subject GR-RBAC denial of `socket(AF_ALG, ...)`.
- **LSM `security_socket_setsockopt`** for `ALG_SET_KEY` / `ALG_SET_KEY_BY_KEY_SERIAL` — per-subject denial of key-setting where policy requires.
- **kfree_sensitive** — every key + IV + AAD + tag scratch freed via sensitive variant.
- **PAX_SIZE_OVERFLOW** — cmsg length arithmetic + AEAD `authsize`/`assoclen` integer bounds checked.

Per-doc rationale: AF_ALG is the userspace funnel into kernel crypto; an attacker who can `socket(AF_ALG, ...)` controls key material, IV reuse, and cipher selection. PAX_USERCOPY + UDEREF block the most common abuse (oversized setsockopt + unchecked cmsg), PAX_MEMORY_SANITIZE + kfree_sensitive deny key residue post-close, and PAX_RAP + CONSTIFY pin algif vtables against ROP/JOP into per-class dispatch.

## Open Questions

(none — AF_ALG ABI is exhaustively specified by upstream)

## Out of Scope

- Per-algorithm cryptographic implementations (`crypto/aes/*.md`, `crypto/sha/*.md`)
- IPsec / kTLS userspace path (cross-ref `net/xfrm/00-overview.md`, `net/tls.md`)
- 32-bit-only paths
- Implementation code
