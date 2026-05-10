# Tier-3: net/xfrm/ah-esp-ipv4 — IPv4 AH/ESP/IPCOMP transforms + xfrm4 adapters

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv4/ah4.c
  - net/ipv4/esp4.c
  - net/ipv4/esp4_offload.c
  - net/ipv4/ipcomp.c
  - net/ipv4/xfrm4_state.c
  - net/ipv4/xfrm4_protocol.c
  - net/ipv4/xfrm4_tunnel.c
  - include/net/xfrm.h
-->

## Summary
Tier-3 design for the IPv4-side IPSec transforms: AH (RFC 4302), ESP (RFC 4303), IPCOMP (RFC 3173) per-protocol `xfrm_type` registrations, the per-AF xfrm4 adapters that connect the generic XFRM pipelines to IPv4-specific encap/decap routines, and the IPv4 xfrm tunnel mode (RFC 3884 / RFC 7296 § 1.3.3) helpers. Each transform implements the RFC-specified wire format + cryptographic operations, plugging into `net/xfrm/state.md` (`x->type->input/output` callbacks) and the kernel crypto API (cross-ref `crypto/00-overview.md`).

ESP's hardware-offload path (`esp4_offload.c`) integrates with `xfrm_device.c` for NIC-side ESP encrypt/decrypt.

Sub-tier-3 of `net/xfrm/00-overview.md`. Mirror Tier-3 `net/xfrm/ah-esp-ipv6.md` covers the IPv6 side. Pairs with `net/xfrm/state.md`, `net/xfrm/input.md`, `net/xfrm/output.md`, `net/xfrm/algo.md`.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| AH4: `xfrm_type` for AH-over-IPv4 | `net/ipv4/ah4.c` |
| ESP4: `xfrm_type` for ESP-over-IPv4 | `net/ipv4/esp4.c` |
| ESP4 offload (TSO + GRO) | `net/ipv4/esp4_offload.c` |
| IPCOMP: IP Payload Compression for IPv4 | `net/ipv4/ipcomp.c` |
| xfrm4 state-AF adapter | `net/ipv4/xfrm4_state.c` |
| Per-protocol RX dispatch (ESP→IPPROTO_ESP, AH→IPPROTO_AH, IPCOMP→IPPROTO_COMP) | `net/ipv4/xfrm4_protocol.c` |
| Tunnel-mode helpers (encapsulate inner IPv4 in outer IPv4) | `net/ipv4/xfrm4_tunnel.c` |
| Public API | `include/net/xfrm.h` (struct xfrm_type) |

## Compatibility contract

### AH-over-IPv4 wire format (RFC 4302)

AH header inserted between outer IP header and inner payload (or original transport header in transport mode):
```
+---------------+---------------+---------------+---------------+
| Next Header   | Payload Len   |          Reserved             |
+---------------+---------------+---------------+---------------+
|                  Security Parameters Index (SPI)              |
+---------------+---------------+---------------+---------------+
|                  Sequence Number Field                        |
+---------------+---------------+---------------+---------------+
|                  Integrity Check Value (ICV) (variable)       |
~                                                               ~
+---------------------------------------------------------------+
```

ICV computed over: outer IP header (with mutable fields zeroed), AH header (with ICV field zeroed), inner payload. Algorithm + zeroed-mutable-field set per RFC 4302 § 3.3.3.1 — identical.

### ESP-over-IPv4 wire format (RFC 4303)

ESP header (4 bytes SPI + 4 bytes Seq) followed by encrypted payload + ESP trailer + ICV:
```
+---------------------------------------------------------------+
|                Security Parameters Index (SPI)                |
+---------------------------------------------------------------+
|                    Sequence Number                            |
+---------------------------------------------------------------+
|         Initialization Vector (IV) (per-cipher)               |
~                                                               ~
+---------------------------------------------------------------+
|                  Encrypted Payload Data                       |
~                                                               ~
+---------------------------------------------------------------+
|              Padding (0–255 bytes)              | Pad Length  |
+---------------------------------------------------------------+
|              Padding (cont)                     | Next Header |
+---------------------------------------------------------------+
|                  Integrity Check Value (ICV)                  |
~                                                               ~
+---------------------------------------------------------------+
```

Per-cipher IV size; per-authenticator ICV size; padding to align to cipher block. Identical wire format.

### IPCOMP-over-IPv4 wire format (RFC 3173)

IPCOMP header:
```
+-------+-------+-------+-------+
|NextHdr| Flags |     CPI       |
+-------+-------+-------+-------+
```

Followed by compressed payload. CPI (Compression Parameter Index) per-RFC reserved + IANA-assigned (deflate=2, lzs=3, lzjh=4). Bypass shorter-than-savings via `IPCOMP_SCRATCH_SIZE` heuristic — identical.

### Transport vs Tunnel mode

- **Transport mode**: AH/ESP inserted between outer IP header and inner transport (TCP/UDP) header
- **Tunnel mode**: outer IP header (new) + AH/ESP + entire inner IP packet (encapsulated)

Per `x->props.mode == XFRM_MODE_TRANSPORT` or `XFRM_MODE_TUNNEL`. Identical encapsulation choice.

### `xfrm4_protocol_register` per-protocol RX

`inet_protos[IPPROTO_AH/ESP/IPCOMP]` → `xfrm4_rcv_cb` → walks `xfrm4_protocol` chain (per registered protocol handler with priority) → calls `handler(skb)`. Identical chain.

### ESP4 hardware offload

When NIC's `dev->features & NETIF_F_HW_ESP`: ESP encrypt/decrypt deferred to NIC. Driver registers `xdo_dev_state_add` to validate SA + install in HW. `esp4_offload.c` implements the offload-aware xmit + GRO paths.

### `crypto_aead_*` integration

ESP4's `esp_input` / `esp_output` use `crypto_aead_*` API (cross-ref `crypto/aead.md` Tier-3) for AEAD ciphers (rfc4106-gcm-aes etc.) or `crypto_skcipher_*` + `crypto_ahash_*` for separate cipher + authenticator (e.g., aes-cbc + hmac-sha256). Identical crypto API usage.

### `xfrm_replay_*` integration

ESP/AH input calls `xfrm_replay_check` after SPI extraction; output calls `xfrm_replay_overflow` for seq allocation (cross-ref `net/xfrm/replay.md`).

## Requirements

- REQ-1: AH-over-IPv4 wire format byte-identical per RFC 4302; mutable-field-zero set identical; ICV algorithm dispatch via `crypto_ahash_*`.
- REQ-2: ESP-over-IPv4 wire format byte-identical per RFC 4303; per-cipher IV-size + per-authenticator ICV-size correctly inserted; padding to cipher block alignment.
- REQ-3: ESP4 supports both AEAD mode (single `crypto_aead`) and separate cipher+auth mode; mode dispatch identical.
- REQ-4: IPCOMP-over-IPv4 wire format byte-identical per RFC 3173; CPI per-IANA assignment; bypass-shorter-than-savings heuristic identical.
- REQ-5: Transport + Tunnel modes both supported per `x->props.mode`; identical encap choice.
- REQ-6: Per-protocol RX dispatch via `xfrm4_protocol_register` chain priority order; handlers registered for IPPROTO_AH/ESP/IPCOMP.
- REQ-7: ESP4 hardware offload path: `dev->features & NETIF_F_HW_ESP` triggers `xdo_dev_*` driver-side install; identical contract.
- REQ-8: Crypto API integration: ESP uses `crypto_aead_*` for AEAD, `crypto_skcipher_*` + `crypto_ahash_*` for separate cipher+auth; AH uses `crypto_ahash_*`; IPCOMP uses `crypto_acomp_*`.
- REQ-9: Replay window integration: ESP/AH input calls `xfrm_replay_check`; output calls `xfrm_replay_overflow`.
- REQ-10: Per-AF state adapter (`xfrm4_state.c`): registers `xfrm_state_afinfo` for AF_INET; identical fields.
- REQ-11: Tunnel mode helpers (`xfrm4_tunnel.c`): outer IP header construction (new daddr from `x->id.daddr`); identical algorithm.
- REQ-12: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: AH4 RX test: send AH-over-IPv4 packet with valid HMAC-SHA256 ICV → kernel verifies + delivers inner; bad ICV → drop + `XfrmInError++`. (covers REQ-1)
- [ ] AC-2: ESP4 RX test: send ESP-over-IPv4 with AES-128-CBC + HMAC-SHA256 → kernel decrypts + delivers; bad ICV → `XfrmInError++`. (covers REQ-2, REQ-3)
- [ ] AC-3: ESP4 AEAD test: send ESP-over-IPv4 with rfc4106(gcm(aes)) AEAD → kernel decrypts + delivers; corrupted authenticator → drop. (covers REQ-3)
- [ ] AC-4: IPCOMP4 test: send IPCOMP-over-IPv4 with deflate-compressed payload → kernel decompresses + delivers inner; uncompressible too-short payload → bypass; pcap shows uncompressed. (covers REQ-4)
- [ ] AC-5: Transport-mode test: SA mode=TRANSPORT; outer IP header is original (saddr/daddr from socket); ESP/AH inserted between outer IP and TCP/UDP. (covers REQ-5)
- [ ] AC-6: Tunnel-mode test: SA mode=TUNNEL; outer IP header is new (saddr=`x->props.saddr`, daddr=`x->id.daddr`); inner is full IP packet. (covers REQ-5, REQ-11)
- [ ] AC-7: HW-offload test on supporting NIC: install SA with offload flag → ESP encrypt/decrypt on NIC; throughput exceeds CPU-only path. (covers REQ-7)
- [ ] AC-8: AEAD wire-format test: tcpdump on ESP-over-IPv4 with AES-GCM-128 → header matches RFC 4106 wire format; salt + per-packet IV correctly placed. (covers REQ-2, REQ-3)
- [ ] AC-9: Per-protocol register test: 2 ESP handlers registered with different priorities; higher-priority handler runs first; ACCEPT result delivered to upper layer. (covers REQ-6)
- [ ] AC-10: Replay integration test: per-SA replay window enforced via `xfrm_replay_check` on ESP RX; replay → `XfrmInStateSeqError++`. (covers REQ-9)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-12)

## Architecture

### Rust module organization

- `kernel::net::xfrm::v4::ah::Ah4` — AH-over-IPv4 `xfrm_type`
- `kernel::net::xfrm::v4::ah::HeaderBuilder` — RFC 4302 header build + mutable-field-zero
- `kernel::net::xfrm::v4::ah::IcvCompute` — `crypto_ahash_*` integration
- `kernel::net::xfrm::v4::esp::Esp4` — ESP-over-IPv4 `xfrm_type`
- `kernel::net::xfrm::v4::esp::AeadPath` — AEAD-mode encrypt/decrypt
- `kernel::net::xfrm::v4::esp::SeparatePath` — cipher+auth-mode encrypt/decrypt
- `kernel::net::xfrm::v4::esp::Padding` — RFC 4303 padding + alignment
- `kernel::net::xfrm::v4::esp::offload::EspOffload` — HW offload path
- `kernel::net::xfrm::v4::ipcomp::Ipcomp4` — IPCOMP-over-IPv4 `xfrm_type`
- `kernel::net::xfrm::v4::ipcomp::Compress`, `Decompress` — `crypto_acomp_*` integration
- `kernel::net::xfrm::v4::tunnel::TunnelEncap` — outer IP header build (tunnel mode)
- `kernel::net::xfrm::v4::tunnel::TunnelDecap` — inner IP header extract (tunnel mode)
- `kernel::net::xfrm::v4::state::Xfrm4StateAfinfo` — per-AF state adapter
- `kernel::net::xfrm::v4::protocol::Xfrm4ProtocolChain` — per-IPPROTO RX dispatch chain

### Locking and concurrency

- **No new global locks**: all per-skb work; SA refcount inherited from xfrm_state lookup
- **Per-`xfrm4_protocol_handlers` rwlock**: protects per-IPPROTO handler chain mutator
- **Per-SA crypto context**: `crypto_aead_*` / `crypto_skcipher_*` / `crypto_ahash_*` instances allocated at SA creation; freed at SA destroy

### Error handling

- `Err(EBADMSG)` — ICV verification failed (drop + counter)
- `Err(EINVAL)` — bad SPI / bad header
- `Err(EAGAIN)` — async crypto in flight
- `Err(EOVERFLOW)` — replay-seq rollover
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| AH header build (mutable-field-zero set; ICV-field zero before compute) | `kani::proofs::net::xfrm::v4::ah_header_safety` |
| ESP padding arithmetic (no overflow; alignment correct) | `kani::proofs::net::xfrm::v4::esp_pad_safety` |
| ESP4 input ICV verify path | `kani::proofs::net::xfrm::v4::esp_input_safety` |
| IPCOMP CPI lookup | `kani::proofs::net::xfrm::v4::ipcomp_safety` |
| Tunnel-mode outer-header build (skb_push bounds) | `kani::proofs::net::xfrm::v4::tunnel_safety` |
| Per-protocol RX chain walk | `kani::proofs::net::xfrm::v4::protocol_chain_safety` |

### Layer 2: TLA+ models

(none new owned; cross-ref `net/xfrm/00-overview.md` Layer 4 ah-esp packet-format theorem)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| AH ICV-buffer alignment | ICV buffer is per-`x->props.aalg.alg_icv_truncbits` length; aligned to 4 bytes | `kani::proofs::net::xfrm::v4::ah_icv_invariants` |
| ESP padding | pad_len ∈ [0, 255]; total skb len after pad is multiple of cipher block-size | `kani::proofs::net::xfrm::v4::esp_pad_invariants` |
| Per-protocol handler chain | handlers in chain ordered by `priority` strictly descending | `kani::proofs::net::xfrm::v4::chain_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/xfrm/00-overview.md` Layer 4)

- **AH/ESP packet-format theorem** (cross-ref `net/xfrm/00-overview.md` Layer 4) — owns proof for v4 wire format

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** | freed-after-decrypt skbs cleared; freed crypto-context buffers cleared (key material) | § Default-on configurable off |
| **SIZE_OVERFLOW** | header-build + padding-arithmetic uses checked operators (CVE class: CVE-2017-class header-overflow) | § Mandatory |
| **CONSTIFY** | per-protocol `struct xfrm_type` registrations `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, SIZE_OVERFLOW, CONSTIFY**: see above
- **USERCOPY**: skb header push/pull uses bound-checked accessors
- **KERNEXEC**: per-protocol type-handler dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: none directly (transforms are kernel-internal); LSM hooks for SA/policy mutation already at `state.md` / `policy.md` / `user.md` level.
- Default GR-RBAC policy: empty so behavior matches upstream.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — AH/ESP/IPCOMP wire format + transforms exhaustively specified by RFCs 4302/4303/3173 + RFC 4106/4309/4543 AEAD bindings + upstream)

## Out of Scope

- IPv6-side transforms (cross-ref `net/xfrm/ah-esp-ipv6.md`)
- Per-cipher / per-authenticator implementations (cross-ref `crypto/00-overview.md`)
- HW-offload driver-side details (cross-ref `net/xfrm/device.md`)
- 32-bit-only paths
- Implementation code
