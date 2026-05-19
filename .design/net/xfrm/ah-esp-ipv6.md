# Tier-3: net/xfrm/ah-esp-ipv6 — IPv6 AH/ESP/IPCOMP transforms + xfrm6 adapters

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/ah6.c
  - net/ipv6/esp6.c
  - net/ipv6/esp6_offload.c
  - net/ipv6/ipcomp6.c
  - net/ipv6/xfrm6_state.c
  - net/ipv6/xfrm6_protocol.c
  - net/ipv6/xfrm6_tunnel.c
  - include/net/xfrm.h
-->

## Summary
Tier-3 design for the IPv6-side IPSec transforms: mirror of `net/xfrm/ah-esp-ipv4.md` for IPv6. Implements AH (RFC 4302), ESP (RFC 4303), IPCOMP (RFC 3173) per-protocol `xfrm_type` registrations for AF_INET6, the per-AF xfrm6 adapters (xfrm6_state/protocol/tunnel) that connect generic XFRM pipelines to IPv6-specific encap/decap routines, and the IPv6 tunnel mode (per RFC 7296 § 1.3.3).

The IPv6 wire format differs from IPv4 in the way AH integrity covers extension headers: AH covers immutable/predictable extension headers (HBH, Routing, Destination Options) per-RFC 4302 § 3.3.3.1.2 with mutable fields zeroed. ESP-over-IPv6 is straightforwardly per-RFC 4303 with the IPv6-pseudo-header (cross-ref `net/ipv6/00-overview.md` for IPv6 pseudo-header layout).

ESP6's hardware-offload path (`esp6_offload.c`) integrates with `xfrm_device.c` for NIC-side ESP encrypt/decrypt over IPv6 (cross-ref `net/xfrm/device.md`).

Sub-tier-3 of `net/xfrm/00-overview.md`. Sibling of `net/xfrm/ah-esp-ipv4.md`. Pairs with `net/xfrm/state.md`, `net/xfrm/input.md`, `net/xfrm/output.md`, `net/xfrm/algo.md`.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| AH6: `xfrm_type` for AH-over-IPv6 (with extension-header handling) | `net/ipv6/ah6.c` |
| ESP6: `xfrm_type` for ESP-over-IPv6 | `net/ipv6/esp6.c` |
| ESP6 offload (TSO + GRO) | `net/ipv6/esp6_offload.c` |
| IPCOMP6: IP Payload Compression for IPv6 | `net/ipv6/ipcomp6.c` |
| xfrm6 state-AF adapter | `net/ipv6/xfrm6_state.c` |
| Per-protocol RX dispatch (ESP→IPPROTO_ESP, AH→IPPROTO_AH, IPCOMP→IPPROTO_COMP for v6) | `net/ipv6/xfrm6_protocol.c` |
| Tunnel-mode helpers for IPv6 | `net/ipv6/xfrm6_tunnel.c` |
| Public API | `include/net/xfrm.h` |

## Compatibility contract

### AH-over-IPv6 wire format (RFC 4302 § 3.3.3.1.2)

AH header inserted between outer IPv6 header (or last extension header) and inner payload (transport mode), or between outer-new IPv6 header and inner full packet (tunnel mode). Wire format:
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

ICV computed over: outer IPv6 header (with mutable fields zeroed), all immutable/predictable extension headers in the chain (HBH/RH-with-segments_left=0/Destination Options), AH header (with ICV field zeroed), inner payload.

Mutable fields per RFC 4302 § 3.3.3.1.2:
- IPv6 header: TC (Traffic Class), Flow Label (mutable except low bits), Hop Limit
- HBH options: per-option Mutable bit
- Destination Options: per-option Mutable bit
- Routing Header Type 0: SegmentsLeft (zero before ICV; result is "as if at destination")
- Routing Header Type 4 (SRv6): SegmentsLeft (zero before ICV) + per-segment fields per RFC 8754 § 5

Identical algorithm + zeroed-mutable-field set so wire compatibility holds.

### ESP-over-IPv6 wire format (RFC 4303)

Same per-RFC layout as ESP-over-IPv4 (4-byte SPI + 4-byte Seq + per-cipher IV + payload + padding + Pad Length + Next Header + ICV), but the Next Header chain handling at decap differs because of IPv6 extension headers between outer IPv6 and ESP. ESP's `Next Header` field always names the inner payload's first nexthdr (e.g., TCP, UDP, IPv6 in tunnel mode) — identical to v4.

### IPCOMP-over-IPv6 wire format (RFC 3173)

Same per-RFC layout as IPv4 IPCOMP. Uses CPI per-IANA assignment.

### Transport vs Tunnel mode for v6

- **Transport mode**: AH/ESP between outer IPv6 (+ extension headers) and inner transport
- **Tunnel mode**: outer IPv6 (new) + AH/ESP + entire inner IPv6 packet

Per `x->props.mode == XFRM_MODE_TRANSPORT` or `XFRM_MODE_TUNNEL`. Identical encapsulation choice.

### `xfrm6_protocol_register` per-protocol RX

`inet6_protos[IPPROTO_AH/ESP/IPCOMP]` → `xfrm6_rcv_cb` → walks `xfrm6_protocol` chain (per registered protocol handler with priority) → calls `handler(skb)`. Identical to v4 chain semantics.

### ESP6 hardware offload

When NIC's `dev->features & NETIF_F_HW_ESP`: ESP6 encrypt/decrypt deferred to NIC. Driver registers `xdo_dev_state_add` to validate SA + install in HW. `esp6_offload.c` implements the offload-aware xmit + GRO paths. Identical contract to v4.

### `crypto_aead_*` integration

ESP6's `esp_input` / `esp_output` use `crypto_aead_*` API (cross-ref `crypto/aead.md` Tier-3) for AEAD ciphers, or `crypto_skcipher_*` + `crypto_ahash_*` for separate cipher + authenticator. Identical to v4.

### `xfrm6_tunnel_register` for IPv6-in-IPv6 tunneling

IPv6 supports both `xfrm6_tunnel` (per-SA tunnel-mode encapsulation) and standalone `ip6_tunnel` (IP-in-IP without IPSec). The xfrm6_tunnel.c here covers the IPSec-tunneled variant only.

## Requirements

- REQ-1: AH-over-IPv6 wire format byte-identical per RFC 4302 § 3.3.3.1.2; mutable-field-zero set covers IPv6 header (TC/FL/HopLimit), HBH, RH, Dest Options per RFC.
- REQ-2: ESP-over-IPv6 wire format byte-identical per RFC 4303; per-cipher IV + per-authenticator ICV correctly placed; Next Header chain handling.
- REQ-3: ESP6 supports both AEAD mode and separate cipher+auth mode; mode dispatch identical to v4.
- REQ-4: IPCOMP-over-IPv6 wire format byte-identical per RFC 3173; CPI per-IANA assignment.
- REQ-5: Transport + Tunnel modes both supported per `x->props.mode`; outer IPv6 header construction in tunnel mode is identical to upstream.
- REQ-6: Per-protocol RX dispatch via `xfrm6_protocol_register` chain priority order; handlers registered for IPPROTO_AH/ESP/IPCOMP for AF_INET6.
- REQ-7: ESP6 hardware offload path: `dev->features & NETIF_F_HW_ESP` triggers `xdo_dev_*` driver-side install; identical contract.
- REQ-8: Crypto API integration: same as v4 (AEAD via `crypto_aead_*`, separate via `crypto_skcipher_*` + `crypto_ahash_*`, AH via `crypto_ahash_*`, IPCOMP via `crypto_acomp_*`).
- REQ-9: Replay window integration: ESP/AH input calls `xfrm_replay_check`; output calls `xfrm_replay_overflow` (cross-ref `net/xfrm/replay.md`).
- REQ-10: Per-AF state adapter (`xfrm6_state.c`): registers `xfrm_state_afinfo` for AF_INET6.
- REQ-11: Tunnel mode helpers (`xfrm6_tunnel.c`): outer IPv6 header construction (new daddr from `x->id.daddr`); identical algorithm.
- REQ-12: AH6 SR-Header (RH4) support per RFC 8754: SegmentsLeft zeroed before ICV; per-segment fields handled per RFC 8754 § 5.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: AH6 RX test: send AH-over-IPv6 with valid HMAC-SHA256 ICV → kernel verifies + delivers inner; bad ICV → drop + `XfrmInError++`. (covers REQ-1)
- [ ] AC-2: AH6 mutable-field-zero test: AH packet with HBH option marked Mutable=1 + non-Mutable=0; ICV verifies for any value of Mutable option, fails if non-Mutable option changed. (covers REQ-1)
- [ ] AC-3: AH6 with SR-Header test: receive AH+RH4 packet with `SegmentsLeft > 0`; kernel zeros `SegmentsLeft` before ICV; ICV correctly verified. (covers REQ-12)
- [ ] AC-4: ESP6 RX test: send ESP-over-IPv6 with AES-128-CBC + HMAC-SHA256 → kernel decrypts + delivers; bad ICV → `XfrmInError++`. (covers REQ-2, REQ-3)
- [ ] AC-5: ESP6 AEAD test: send ESP-over-IPv6 with rfc4106(gcm(aes)) → kernel decrypts + delivers; corrupted authenticator → drop. (covers REQ-3)
- [ ] AC-6: IPCOMP6 test: send IPCOMP-over-IPv6 with deflate-compressed payload → kernel decompresses + delivers inner. (covers REQ-4)
- [ ] AC-7: Transport-mode test: SA mode=TRANSPORT for IPv6; outer IPv6 is original; ESP/AH inserted between outer IPv6 (+ extension headers) and TCP/UDP. (covers REQ-5)
- [ ] AC-8: Tunnel-mode test: SA mode=TUNNEL for IPv6; outer IPv6 is new (saddr=`x->props.saddr`, daddr=`x->id.daddr`); inner is full IPv6 packet. (covers REQ-5, REQ-11)
- [ ] AC-9: HW-offload test on supporting NIC: install IPv6 SA with offload flag → ESP encrypt/decrypt on NIC; throughput exceeds CPU-only path. (covers REQ-7)
- [ ] AC-10: AEAD wire-format test: tcpdump on ESP-over-IPv6 with AES-GCM-128 → header matches RFC 4106 wire format; salt + per-packet IV correctly placed. (covers REQ-2, REQ-3)
- [ ] AC-11: Per-protocol register test: 2 ESP6 handlers registered with different priorities; higher-priority handler runs first. (covers REQ-6)
- [ ] AC-12: Replay integration test: per-SA replay window enforced via `xfrm_replay_check` on ESP6 RX. (covers REQ-9)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::xfrm::v6::ah::Ah6` — AH-over-IPv6 `xfrm_type`
- `kernel::net::xfrm::v6::ah::HeaderBuilder` — RFC 4302 header build for v6 + extension-header walk + mutable-field-zero
- `kernel::net::xfrm::v6::ah::IcvCompute` — `crypto_ahash_*` integration
- `kernel::net::xfrm::v6::ah::SrSupport` — RFC 8754 SR-Header AH handling
- `kernel::net::xfrm::v6::esp::Esp6` — ESP-over-IPv6 `xfrm_type`
- `kernel::net::xfrm::v6::esp::AeadPath` — AEAD-mode encrypt/decrypt
- `kernel::net::xfrm::v6::esp::SeparatePath` — cipher+auth-mode encrypt/decrypt
- `kernel::net::xfrm::v6::esp::Padding` — RFC 4303 padding + alignment
- `kernel::net::xfrm::v6::esp::offload::EspOffload` — HW offload path
- `kernel::net::xfrm::v6::ipcomp::Ipcomp6` — IPCOMP-over-IPv6 `xfrm_type`
- `kernel::net::xfrm::v6::tunnel::TunnelEncap` — outer IPv6 header build (tunnel mode)
- `kernel::net::xfrm::v6::tunnel::TunnelDecap` — inner IPv6 header extract (tunnel mode)
- `kernel::net::xfrm::v6::state::Xfrm6StateAfinfo` — per-AF state adapter
- `kernel::net::xfrm::v6::protocol::Xfrm6ProtocolChain` — per-IPPROTO RX dispatch chain

### Locking and concurrency

- (Same as v4) **No new global locks**: per-skb work; SA refcount inherited from xfrm_state lookup
- **Per-`xfrm6_protocol_handlers` rwlock**: protects per-IPPROTO handler chain mutator
- **Per-SA crypto context**: allocated at SA creation; freed at SA destroy

### Error handling

(Same set as v4)

- `Err(EBADMSG)` — ICV verification failed
- `Err(EINVAL)` — bad SPI / bad header
- `Err(EAGAIN)` — async crypto in flight
- `Err(EOVERFLOW)` — replay-seq rollover
- `Err(ENOMEM)` — alloc fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| AH6 header build (extension-header walk + mutable-field-zero) | `kani::proofs::net::xfrm::v6::ah_header_safety` |
| AH6 SR-Header handling (SegmentsLeft zero + per-segment-field zero) | `kani::proofs::net::xfrm::v6::ah_sr_safety` |
| ESP6 padding arithmetic (no overflow; alignment correct) | `kani::proofs::net::xfrm::v6::esp_pad_safety` |
| ESP6 input ICV verify path | `kani::proofs::net::xfrm::v6::esp_input_safety` |
| IPCOMP6 CPI lookup | `kani::proofs::net::xfrm::v6::ipcomp_safety` |
| Tunnel-mode outer-header build (skb_push bounds) | `kani::proofs::net::xfrm::v6::tunnel_safety` |
| Per-protocol RX chain walk | `kani::proofs::net::xfrm::v6::protocol_chain_safety` |

### Layer 2: TLA+ models

(none new owned; cross-ref `net/xfrm/00-overview.md` Layer 4 ah-esp packet-format theorem)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| AH6 ICV-buffer alignment | ICV buffer is per-`x->props.aalg.alg_icv_truncbits` length; aligned to 8 bytes (IPv6 alignment) | `kani::proofs::net::xfrm::v6::ah_icv_invariants` |
| Extension-header walk for AH6 | walk terminates on inner-payload; visits each extension exactly once; mutable-fields-zeroed-before-icv invariant | `kani::proofs::net::xfrm::v6::eh_walk_invariants` |
| Tunnel-mode outer header | `outer.payload_len` == `inner.payload_len + sizeof(esp/ah header) + per-cipher overhead` | `kani::proofs::net::xfrm::v6::tunnel_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/xfrm/00-overview.md` Layer 4)

- **AH/ESP packet-format theorem** (cross-ref `net/xfrm/00-overview.md` Layer 4) — owns proof for v6 wire format

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** | freed-after-decrypt skbs cleared; freed crypto-context buffers cleared (key material) | § Default-on configurable off |
| **SIZE_OVERFLOW** | extension-header walk + padding + outer-header arithmetic uses checked operators (CVE class: CVE-2017-class header-overflow + CVE-2024-class SRv6 underflow) | § Mandatory |
| **CONSTIFY** | per-protocol `struct xfrm_type` registrations `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, SIZE_OVERFLOW, CONSTIFY**: see above
- **USERCOPY**: skb header push/pull uses bound-checked accessors
- **KERNEXEC**: per-protocol type-handler dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

(Same as v4; LSM hooks at higher tiers — `state.md`, `policy.md`, `user.md`)

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds skb head/tail copies on the v6 transform path and any XFRM netlink crossing for SA install on this transform.
- **PAX_KERNEXEC** — keeps the `static const struct xfrm_type ah6/esp6/ipcomp6_type` registrations and per-protocol dispatch text W^X.
- **PAX_RANDKSTACK** — randomises stack per `ah6_input`/`esp6_input`/`esp6_output` entry; mitigates ROP under v6 AH/ESP floods.
- **PAX_REFCOUNT** — wraps `xfrm6_state` and tunnel-state refs taken during transform so concurrent SA delete cannot UAF an in-flight v6 ESP packet.
- **PAX_MEMORY_SANITIZE** — zeroes crypto request buffers, freed decrypted skb frags, and freed extension-header walk scratch on free.
- **PAX_UDEREF** — protects netlink-driven SA install (v6 selector) from userland deref on malformed addresses or selectors.
- **PAX_RAP / kCFI** — protects indirect dispatch through `xfrm_type->{input,output}` and crypto-completion callbacks (AEAD) on the v6 path.
- **GRKERNSEC_HIDESYM** — hides `esp6_*`, `ah6_*`, `ipcomp6_*`, `xfrm6_*` symbols from unprivileged readers.
- **GRKERNSEC_DMESG** — restricts dmesg so v6 ICV-failure / extension-header / SRv6 parse traces don't leak SPIs or peer addresses.
- **IPsec SAD/SPD CAP_NET_ADMIN** — `XFRM_MSG_NEWSA`/`UPDSA` for v6 transforms requires CAP_NET_ADMIN in the SA's user_ns.
- **XFRM key MEMORY_SANITIZE** — `XFRMA_ALG_AUTH`/`AEAD`/`CRYPT` keys on the v6 SA are wiped on destroy and on key-rollover.
- **Replay-window protected** — `x->replay`/`x->preplay` is accessed only under `x->lock`; PAX_MEMORY_SANITIZE wipes the window on SA free.
- **ICV verification bounded** — `esp6_input_done2`/`ah6_input` reject truncated trailers and ICV-length mismatch before crypto dispatch; ext-header walk uses SIZE_OVERFLOW-checked arithmetic against the SRv6/HBH/DST cases.

Rationale: IPv6 AH/ESP inherit the IPv4 risk surface and add the extension-header walker plus SRv6 — exactly where the CVE-2024-class underflows live. The same USERCOPY/UDEREF/REFCOUNT/MEMORY_SANITIZE envelope, combined with bounded ext-header arithmetic and ICV-length checks, confines the realistic surface to crypto primitives covered separately.

## Open Questions

(none — IPv6 AH/ESP/IPCOMP wire format + transforms exhaustively specified by RFCs 4302/4303/3173 + RFC 8754 SR-Header + RFC 4106/4309/4543 AEAD bindings + upstream)

## Out of Scope

- IPv4-side transforms (cross-ref `net/xfrm/ah-esp-ipv4.md`)
- Per-cipher / per-authenticator implementations (cross-ref `crypto/00-overview.md`)
- HW-offload driver-side details (cross-ref `net/xfrm/device.md`)
- Standalone IPv6-in-IPv6 tunneling without IPSec (cross-ref `net/ipv6/ip6_tunnel.md` Tier-3)
- 32-bit-only paths
- Implementation code
