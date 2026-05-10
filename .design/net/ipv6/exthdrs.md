# Tier-3: net/ipv6/exthdrs — extension header semantics (HBH, RH, DestOpt, SR, Calipso)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/exthdrs.c
  - net/ipv6/exthdrs_core.c
  - net/ipv6/exthdrs_offload.c
  - net/ipv6/seg6.c
  - net/ipv6/seg6_iptunnel.c
  - net/ipv6/seg6_local.c
  - net/ipv6/calipso.c
-->

## Summary
Tier-3 design for IPv6 extension-header semantics: per-EH-type TLV option dispatch + behavior. Handles Hop-by-Hop Options (RFC 8200 § 4.3 — Pad1, PadN, Router Alert, Jumbo Payload, IOAM, Calipso), Routing Header (RFC 8200 § 4.4 — Type 0 dropped per RFC 5095, Type 2 Mobile IPv6, Type 4 Segment Routing per RFC 8754, Type 1/3 deprecated), Destination Options (RFC 8200 § 4.6 — Tunnel Encapsulation Limit, Home Address Option), Segment Routing (SRv6 — `seg6.c`, `seg6_iptunnel.c`, `seg6_local.c`), Calipso (RFC 5570 IP-Security labeling), and EH offload to NIC's encapsulation engines.

The walker itself + RX/TX integration are in `net/ipv6/ip6-input.md` / `ip6-output.md`. This Tier-3 covers the per-option processing + per-EH-type protocol handlers.

Sub-tier-3 of `net/ipv6/00-overview.md`.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Per-EH-type RX handlers + per-option TLV dispatch | `net/ipv6/exthdrs.c` |
| EH skip helpers (used by GRO + checksum) | `net/ipv6/exthdrs_core.c` |
| EH offload (GRO/GSO) | `net/ipv6/exthdrs_offload.c` |
| Segment Routing core | `net/ipv6/seg6.c` |
| Segment Routing iptunnel encap | `net/ipv6/seg6_iptunnel.c` |
| Segment Routing local-action processing | `net/ipv6/seg6_local.c` |
| Calipso (CIPSO equivalent for IPv6) | `net/ipv6/calipso.c` |

## Compatibility contract

### TLV option encoding (HBH + DestOpt)

Each option:
```
+-+-+-+-+-+-+-+-+
| Option Type   |
+-+-+-+-+-+-+-+-+
| Opt Data Len  |
+-+-+-+-+-+-+-+-+
| Option Data   |
| (variable)    |
+-+-+-+-+-+-+-+-+
```

Option Type's high-2-bits dictate unrecognized-type behavior (RFC 8200 § 4.2):
- `00`: skip
- `01`: discard packet
- `10`: discard + send ICMP Parameter Problem (Code 2)
- `11`: discard + send ICMP Parameter Problem (Code 2) only if dst is unicast

Wire format byte-identical so packet-format-checking tools work.

### Recognized HBH options

| Type | Constant | Semantic |
|---|---|---|
| 0 | Pad1 | 1-byte pad |
| 1 | PadN | N-byte pad |
| 5 | Router Alert | (Type/Length/Value with 16-bit value); per RFC 2711 |
| 194 | Jumbo Payload | for >65535-byte packets |
| 49 | Calipso | CIPSO-equivalent label |
| 73 | IOAM | per RFC 9197 |

Identical recognized set + handlers.

### Recognized Routing Header types

| Type | Status | Handler |
|---|---|---|
| 0 | RFC 5095 — DROPPED | reject + ICMP Parameter Problem |
| 1 | Nimrod — RESERVED | reject + ICMP Parameter Problem |
| 2 | Mobile IPv6 (RFC 6275) | accept; swap addrs per Mobile IPv6 |
| 3 | RPL (RFC 6554) | accept; per-RPL processing |
| 4 | Segment Routing (RFC 8754) | accept; per-SRv6 processing |

Identical statuses + handlers.

### Segment Routing (SRv6)

Per-iface `/sys/devices/virtual/net/<iface>/queues/seg6_*` knobs. SRv6 actions: `End`, `End.X`, `End.T`, `End.DX2/4/6`, `End.DT4/6`, `End.B6`, `End.B6.Encaps`, `End.BM`, `End.S`, `End.AS`, `End.AM`, `End.BPF`. Per-RFC 8986 + extensions. Identical action set + per-action behavior.

### Calipso

Per-RFC 5570 IPv6 IP-Security labeling — used by SELinux/Smack via `security/selinux/netlabel.c`. Wire format byte-identical so existing CIPSO-using deployments interoperate.

### `IPV6_HOPOPTS`/`IPV6_DSTOPTS`/`IPV6_RTHDR` setsockopts

Per-socket setsockopt installs an option chain in `np->opt`; injected into TX (cross-ref `net/ipv6/ip6-output.md` REQ-2). Wire format byte-identical so the receiver sees the option chain as upstream.

### Extension-header walk bound

`MAX_HEADER_OPT_PARSED` (typically 255 options) caps the per-packet TLV walk; rejects extensions that exceed it with ICMP Parameter Problem. Defense vs. CVE-2017-7542-class extension-header-explosion attacks.

## Requirements

- REQ-1: HBH option TLV dispatch: Pad1, PadN, Router Alert, Jumbo Payload, Calipso, IOAM recognized; unknown types dispatch per high-2-bits per RFC 8200 § 4.2.
- REQ-2: DestOpt option TLV dispatch: Pad1, PadN, Tunnel Encapsulation Limit (RFC 2473), Home Address Option (RFC 6275); unknown types dispatch per RFC 8200 § 4.2.
- REQ-3: Routing Header Type 0: per RFC 5095, dropped + ICMP Parameter Problem. Identical.
- REQ-4: Routing Header Type 2 (Mobile IPv6): per RFC 6275, swap addrs per `addresses_left`. Identical.
- REQ-5: Routing Header Type 4 (SRv6): per RFC 8754, walk segment list; perform per-segment action.
- REQ-6: SRv6 actions per RFC 8986: End, End.X, End.T, End.DX2/4/6, End.DT4/6, End.B6, End.B6.Encaps, End.BM, End.S, End.AS, End.AM, End.BPF — implemented identically.
- REQ-7: SRv6 iptunnel encap: lwtunnel-style; identical iproute2 wire format (`ip route add encap seg6 mode encap segs <list> dev eth0`).
- REQ-8: SRv6 sysctls: `/proc/sys/net/ipv6/seg6_enabled` per-iface; `/proc/sys/net/ipv6/seg6_require_hmac` per-iface; HMAC keys via genetlink.
- REQ-9: Calipso labeling: RX validates label per per-iface CIPSO domain DOI mapping; TX inserts label from sock's CIPSO state. Cross-ref `security/00-overview.md`.
- REQ-10: Per-socket EH installation via `IPV6_HOPOPTS` / `IPV6_DSTOPTS` / `IPV6_RTHDR` setsockopts; identical UAPI + injection at TX.
- REQ-11: Extension-header walk cap `MAX_HEADER_OPT_PARSED` enforced; over-cap rejected with ICMP Parameter Problem; per-netns counter `Ip6InHdrErrors` bumped.
- REQ-12: EH offload (GRO/GSO): per-iface `dev->features` may indicate EH offload capability; `exthdrs_offload.c` integrates with GRO/GSO callbacks.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: HBH unknown-type-with-bits-`11`-`unicast`-only test: send to multicast → no ICMP; send to unicast → ICMP Parameter Problem Code 2. (covers REQ-1)
- [ ] AC-2: PadN with `Opt Data Len = N`: parser advances exactly `N+2` bytes; validates with `N <= remaining`. (covers REQ-1)
- [ ] AC-3: Router Alert option (Type 5, value 0): kernel processes packet (e.g., MLD); without alert, packet routed without alert-handler. (covers REQ-1)
- [ ] AC-4: RH0 test: receive IPv6 packet with RH Type=0 → drop + ICMP Parameter Problem; counter `Ip6InHdrErrors++`. (covers REQ-3)
- [ ] AC-5: SRv6 End action test: configure SRv6 SID, receive packet with RH Type=4 + SID match, `addresses_left > 0` → kernel decrements `addresses_left`, sets dst to next segment, continues. (covers REQ-5, REQ-6)
- [ ] AC-6: SRv6 iptunnel test: `ip route add 2001:db8::/64 encap seg6 mode encap segs 2001::1,2001::2 dev eth0`; verify TX prefixes RH4 + segment list. (covers REQ-7)
- [ ] AC-7: SRv6 HMAC test: enable `seg6_require_hmac`; reject packet without HMAC TLV; accept with valid HMAC. (covers REQ-8)
- [ ] AC-8: Calipso label test: assign CIPSO domain to socket; tcpdump shows HBH option Type=49 with proper label fields. (covers REQ-9)
- [ ] AC-9: `setsockopt(IPV6_RTHDR, ...)` per-socket test: TX from socket has RH injected per setsockopt buffer. (covers REQ-10)
- [ ] AC-10: Walk-cap test: send packet with 256+ TLV options → reject + ICMP Parameter Problem; CPU bounded. (covers REQ-11)
- [ ] AC-11: GRO test: 2 IPv6 packets with identical 5-tuple + EH chain are aggregated; without identical EH, kept separate. (covers REQ-12)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::ipv6::exthdrs::TlvWalker` — bounded TLV walker
- `kernel::net::ipv6::exthdrs::hbh::Hbh` — HBH dispatcher
- `kernel::net::ipv6::exthdrs::hbh::Pad1`, `PadN`, `RouterAlert`, `Jumbo`, `Calipso`, `Ioam`
- `kernel::net::ipv6::exthdrs::dest::Dest` — DestOpt dispatcher
- `kernel::net::ipv6::exthdrs::dest::TunnelEncapLimit`, `HomeAddress`
- `kernel::net::ipv6::exthdrs::rh::RoutingHeader` — RH dispatcher
- `kernel::net::ipv6::exthdrs::rh::Rh0`, `Rh2Mip6`, `Rh4Srv6`
- `kernel::net::ipv6::seg6::Seg6` — SRv6 core
- `kernel::net::ipv6::seg6::actions::*` — per-action handlers (End, End.X, …)
- `kernel::net::ipv6::seg6::iptunnel::Seg6IpTunnel`
- `kernel::net::ipv6::seg6::hmac::Seg6Hmac`
- `kernel::net::ipv6::calipso::Calipso`
- `kernel::net::ipv6::exthdrs::offload::Offload`

### Locking and concurrency

- **Per-EH-type handler dispatch**: lockless RCU read-side; static const dispatch table
- **`seg6_pernet` (per-netns) HMAC-key registry**: rwlock for read-side
- **`calipso_doi_list_lock` (per-netns)**: per-DOI mapping registry

### Error handling

- ICMP Parameter Problem (Type 4 Code 0/1/2) generated on EH parse failure
- Counter `Ip6InHdrErrors++` per-netns / per-idev
- skb dropped via kfree_skb_reason for telemetry
- Walk-cap exhaustion → drop + ICMP Code 2

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| TLV walker (no infinite loop on 0-length option, bounded by walk-cap) | `kani::proofs::net::ipv6::exthdrs::tlv_safety` |
| Per-HBH-option handler bounds checking | `kani::proofs::net::ipv6::exthdrs::hbh_safety` |
| RH segment-list walk (no out-of-bounds index) | `kani::proofs::net::ipv6::exthdrs::rh_walk_safety` |
| SRv6 End / End.X / End.T action arithmetic | `kani::proofs::net::ipv6::seg6::action_safety` |
| Calipso label parser | `kani::proofs::net::ipv6::calipso::parse_safety` |

### Layer 2: TLA+ models

(none mandatory at this level — covered by `net/ipv6/00-overview.md` Layer 4 ext-header walk theorem)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| TLV-option walk | Σ option-bytes-consumed ≤ EH length; remaining-bytes ≥ 0 throughout | `kani::proofs::net::ipv6::exthdrs::walk_invariants` |
| RH segment list | `addresses_left` ≤ `segments_left`; `segments_left` ≤ array bound from `Hdr Ext Len` | `kani::proofs::net::ipv6::exthdrs::rh_invariants` |

### Layer 4: Functional correctness (opt-in; declared in `net/ipv6/00-overview.md` Layer 4)

- **Extension-header bounded-walk theorem** via Verus (cross-ref `net/ipv6/ip6-input.md` Layer 4)
- **SRv6 End-action correctness theorem** via Verus — proves: ∀ End-class action `a`, post-action skb has dst = `segments[segments_left-1]` and `segments_left -= 1`, no other state change.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **SIZE_OVERFLOW** | TLV-option-length + RH-segments_left arithmetic uses checked operators (CVE class: CVE-2017-7542 + CVE-2024-class SRv6 underflow) | § Mandatory |
| **CONSTIFY** | per-EH-type handler dispatch table `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed seg6 / Calipso DOI state cleared (sensitive: HMAC keys, security labels) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB**: per-DOI + per-HMAC-key + per-iptunnel slabs
- **SIZE_OVERFLOW**: see above
- **CONSTIFY**: see above
- **KERNEXEC**: TLV-option dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for SRv6 SID configuration; GR-RBAC policy can deny per-subject (default empty).
- LSM hooks for Calipso label evaluation (cross-ref `security/00-overview.md`); these are integral to MLS-style policies.
- Useful default policy: deny SRv6 endpoint configuration outside gradm-marked `srv6_admin` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — IPv6 EH semantics are exhaustively specified by RFCs 8200, 5095, 6275, 8754, 8986 + upstream)

## Out of Scope

- IPv6 RX/TX path integration (cross-ref `net/ipv6/ip6-input.md`, `ip6-output.md`)
- AH/ESP (cross-ref `net/xfrm/00-overview.md`)
- 32-bit-only paths
- Implementation code
