---
title: "Tier-3: net/ipv6/icmpv6 — ICMPv6 (RFC 4443) + Echo Sockets"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for ICMPv6 (RFC 4443) and the closely-related Echo Socket (`SOCK_DGRAM` IPPROTO_ICMPV6 ping socket): generation + reception of ICMPv6 messages, error generation (Destination Unreachable, Packet Too Big, Time Exceeded, Parameter Problem), Echo Request/Reply for ping, Multicast Listener Discovery v1/v2 (MLD — RFC 3810) packet snooping integration, the per-netns ICMPv6 rate-limiter (`/proc/sys/net/ipv6/icmp/*`), and the userspace Echo socket interface (gated by `/proc/sys/net/ipv4/ping_group_range`).

ND messages (RS/RA/NS/NA/Redirect — types 133-137) are handled by `net/ipv6/ndisc.md`. This Tier-3 covers the rest of ICMPv6 plus the rate-limiter / echo-socket infrastructure.

Sub-tier-3 of `net/ipv6/00-overview.md`. Cooperates with `net/ipv6/ip6-input.md` (RX entrypoint), `net/ipv6/route.md` (PMTU updates from PTB messages), `net/ipv6/ndisc.md` (carrier protocol).

### Requirements

- REQ-1: All non-ND ICMPv6 message types (per the table above) parsed/generated per RFC 4443 + extension RFCs (3810, 4286, 4620, 5006, 6275, 6550, etc.).
- REQ-2: ICMPv6 error generation: Destination Unreachable, Packet Too Big, Time Exceeded, Parameter Problem; identical per-RFC suppression heuristics.
- REQ-3: PMTU integration: receipt of Packet Too Big with MTU=N → call `rt6_update_expires` / `rt6_pmtu_discovery` to update per-flow rt6_info.rt6i_pmtu; cross-ref `net/ipv6/route.md`.
- REQ-4: Echo Request handler: respond with Echo Reply unless `echo_ignore_all` (or applicable multicast/anycast guard) is set.
- REQ-5: Echo socket interface: `SOCK_DGRAM IPPROTO_ICMPV6`; sendmsg sends Echo Request, recvmsg receives Echo Reply; identical to `net/ipv4/ping.md` IPv4 ping socket.
- REQ-6: Per-netns ICMPv6 rate-limiter: per-destination token bucket; `ratelimit` ms cooldown; `ratemask` bitmap selects which types are limited.
- REQ-7: MLDv1/v2 helper integration (`mcast_snoop.c`): bridge layer can inspect MLD packets to update per-port multicast forwarding.
- REQ-8: `struct icmp6hdr` byte-identical layout.
- REQ-9: Per-netns ICMPv6 control socket (`icmpv6_sk_per_netns`); identical creation/destruction lifecycle.
- REQ-10: TLA+ model (none mandatory at this level — covered by Layer 3 invariants).
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct icmp6hdr` byte-identical. (covers REQ-8)
- [ ] AC-2: `ping6 -c 5 ::1` succeeds; tcpdump shows Echo Request/Reply pairs with checksums computed identically. (covers REQ-1, REQ-4)
- [ ] AC-3: Send a UDP packet to a closed IPv6 port → receive ICMPv6 Type 1 Code 4 (Port Unreachable) with first-cache-line of original packet quoted (RFC 4443 § 3.1). (covers REQ-2)
- [ ] AC-4: PMTU test: a packet larger than path MTU triggers ICMPv6 PTB; subsequent socket sends to that flow respect the lower MTU; verify via `ip -6 route get`. (covers REQ-3)
- [ ] AC-5: Set `echo_ignore_all=1`; ping6 to local address times out; reset to `0`; ping6 succeeds. (covers REQ-4)
- [ ] AC-6: Echo socket test: `socket(AF_INET6, SOCK_DGRAM, IPPROTO_ICMPV6)` with appropriate gid → sendmsg/recvmsg works without root. (covers REQ-5)
- [ ] AC-7: Rate-limit test: send 1000 unroutable packets per second; ICMPv6 error rate caps at `1000 / ratelimit_ms × second / 1000` per destination. (covers REQ-6)
- [ ] AC-8: MLD-snoop test: a bridge with `mcast_snooping=1` filters MLD-Listener-Report messages and updates per-port forwarding. (covers REQ-7)
- [ ] AC-9: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::ipv6::icmpv6::Icmpv6` — top-level entrypoint
- `kernel::net::ipv6::icmpv6::recv::Icmpv6Rcv` — RX path (icmpv6_rcv)
- `kernel::net::ipv6::icmpv6::send::Icmpv6Send` — TX path (icmpv6_send_*)
- `kernel::net::ipv6::icmpv6::error::ErrorBuilder` — DestUnreach / PTB / TimeExceeded / ParamProblem
- `kernel::net::ipv6::icmpv6::echo::EchoHandler` — Echo Request → Echo Reply
- `kernel::net::ipv6::icmpv6::ratelimit::RateLimiter` — per-netns rate-limit
- `kernel::net::ipv6::icmpv6::pmtu::PmtuUpdate` — PTB → route cache update
- `kernel::net::ipv6::icmpv6::ping::PingSocket` — Echo socket (SOCK_DGRAM IPPROTO_ICMPV6)
- `kernel::net::ipv6::icmpv6::sk::PerNetnsCtrlSk` — per-netns control socket

### Locking and concurrency

- **Per-netns `icmpv6_sk_lock`** (mutex): per-CPU control-socket lock-bh
- **Per-destination rate-limit token bucket**: per-netns FIB6 + per-rt6_exception slot; lockless on read via RCU
- **Echo socket per-bucket spinlocks**: shared with `net/ipv4/ping.c` style tables

### Error handling

- `Err(EAFNOSUPPORT)` — Echo socket on non-IPv6 build
- `Err(EACCES)` — Echo socket without ping_group_range membership
- `Err(EAGAIN)` — rate-limiter blocked emission (silent in upstream; counter incremented)
- `Err(ENOMEM)` — skb alloc fail
- `Err(EHOSTUNREACH)` — error packet to unroutable source

### Out of Scope

- ND-bound ICMPv6 messages types 133-137 (cross-ref `net/ipv6/ndisc.md`)
- Multicast routing PIM-SM (cross-ref `net/ipv6/ip6mr.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| ICMPv6 generic generation + reception, rate-limiter, error-message helpers | `net/ipv6/icmp.c` |
| Generic outer ICMP API (shared with IPv4) | `net/ipv6/ip6_icmp.c` |
| ICMPv6 Echo socket interface | `net/ipv6/ping.c` |
| MLD multicast snooping helpers | `net/ipv6/mcast_snoop.c` |
| Public API | `include/net/icmp.h` |
| UAPI | `include/uapi/linux/icmpv6.h` |

### compatibility contract

### ICMPv6 message types (RFC 4443 + extensions)

| Type | Name |
|---|---|
| 1 | Destination Unreachable |
| 2 | Packet Too Big (PTB) |
| 3 | Time Exceeded |
| 4 | Parameter Problem |
| 100,101,200,201 | Private experimentation |
| 127 | Reserved for error msgs |
| 128 | Echo Request |
| 129 | Echo Reply |
| 130–132 | Multicast Listener Query/Report v1/Done |
| 133–137 | NS/NA/RS/RA/Redirect (covered in `ndisc.md`) |
| 138 | Router Renumbering |
| 139,140 | ICMP Node Information |
| 141,142 | Inverse ND |
| 143 | MLDv2 Listener Report |
| 144–147 | Mobile IPv6 |
| 148,149 | SEcure ND |
| 151,152,153 | Multicast Router Discovery |
| 155 | RPL Control |
| 200,201 | Private experimentation |

Identical type/code values + behavior (gated by feature CONFIGs where applicable).

### ICMPv6 rate limiter sysctls

`/proc/sys/net/ipv6/icmp/`:
- `ratelimit` (default 1000ms) — per-netns rate limit; if 0 → no limit
- `echo_ignore_all` — drop all echo
- `echo_ignore_multicast` — drop multicast echo
- `echo_ignore_anycast` — drop anycast echo
- `error_anycast_as_unicast` — RFC 4443 § 2.4 (e) variation
- `ratemask` — bitmap of which ICMPv6 message types are rate-limited (default: error types 1-4 + 100-127)

Format byte-identical so distro sysctl.d files work unchanged.

### Echo socket gate

`socket(AF_INET6, SOCK_DGRAM, IPPROTO_ICMPV6)` gated by being in the gid range `/proc/sys/net/ipv4/ping_group_range` (yes, IPv4 sysctl shared) — non-root unprivileged ping. Identical.

### `struct icmp6hdr` layout

`include/uapi/linux/icmpv6.h`: `icmp6_type`, `icmp6_code`, `icmp6_cksum`, then per-type union (icmp6_dataun). Layout-byte-identical.

### Error generation budget

When kernel must send an ICMPv6 error (e.g., for an unroutable IPv6 packet): rate-limited per-destination via the rate-limiter; per-RFC 4443 § 2.4 not generated for ICMPv6 errors, multicast destinations (with exceptions), or non-first fragments. Identical heuristics.

### Per-netns ICMPv6 socket

Per-netns ICMPv6 control socket used to source kernel-generated ICMPv6 messages. Uses `IPV6_FREEBIND`-style binding to allow source-address selection across all configured local addresses. Identical.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| ICMPv6 RX dispatch (type-table walk) | `kani::proofs::net::ipv6::icmpv6::rx_dispatch_safety` |
| ICMPv6 error generation (orig-packet quote bound) | `kani::proofs::net::ipv6::icmpv6::error_quote_safety` |
| Rate-limit token bucket (no negative-balance underflow) | `kani::proofs::net::ipv6::icmpv6::ratelimit_safety` |
| Echo socket sendmsg/recvmsg cmsg path | `kani::proofs::net::ipv6::icmpv6::ping_safety` |
| PTB → rt6 PMTU update (refcount sound) | `kani::proofs::net::ipv6::icmpv6::pmtu_safety` |

### Layer 2: TLA+ models

(none mandatory at this level)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns rate-limit token bucket | balance ≥ 0; refill rate ≤ configured `ratelimit` per second | `kani::proofs::net::ipv6::icmpv6::bucket_invariants` |
| Echo socket per-bucket list | each socket appears in exactly one bucket determined by `ping_hashfn` | `kani::proofs::net::ipv6::icmpv6::ping_hash_invariants` |

### Layer 4: Functional correctness (opt-in)

- **RFC 4443 § 2.4 error-suppression theorem** via Verus — proves: kernel never generates ICMPv6 error in response to (a) ICMPv6 error, (b) multicast dest (per § 2.4 (e) (3)), (c) non-first fragment, (d) source-address-selection-fails-with-no-acceptable-source.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-Echo-socket refcounts use `Refcount` (saturating) | § Mandatory |
| **MEMORY_SANITIZE** | error-quote skb's freed-after-emit cleared | § Default-on configurable off |
| **SIZE_OVERFLOW** | error-quote-length arithmetic uses checked operators (defense vs. CVE-class quote-length-overflow) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: ICMPv6 type-handler dispatch table `static const`
- **USERCOPY**: Echo socket sendmsg/recvmsg uses `copy_to_iter`/`copy_from_iter`
- **KERNEXEC**: type-handler dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_socket_create` for AF_INET6 Echo socket; GR-RBAC policy can deny per-subject (default empty).
- Useful default policy: deny ICMPv6 Echo socket outside gradm-marked `ping_users` role for stricter environments.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

