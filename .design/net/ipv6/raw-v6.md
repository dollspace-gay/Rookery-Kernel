# Tier-3: net/ipv6/raw-v6 — RAW IPv6 sockets + ICMPv6 filtering

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/raw.c
  - net/ipv6/datagram.c
  - include/net/ipv6.h
  - include/net/raw.h
  - include/uapi/linux/in6.h
-->

## Summary
Tier-3 design for RAW IPv6 sockets (`socket(AF_INET6, SOCK_RAW, proto)`): the kernel datapath that delivers raw IPv6 datagrams (typically ICMPv6, OSPFv3, custom protocols, or `IPPROTO_RAW` for full IPv6-header injection) to userspace without per-protocol stack processing. Supports the ICMPv6 type filter (`ICMPV6_FILTER` setsockopt — RFC 3542 § 3.2), per-socket checksum offset (`IPV6_CHECKSUM`), per-recvmsg cmsg surface (shared with UDPv6), and `/proc/net/raw6` enumeration.

CAP_NET_RAW gated. Used by tcpdump's BSD-side fallback, dhclient-v6 (when not using NL), netcat raw mode, traceroute6, OSPFv3 daemons (frr, bird), and custom IPv6 transports.

Sub-tier-3 of `net/ipv6/00-overview.md`. Pairs with `net/ipv4/raw.md` (parent abstraction), `net/ipv6/af-inet6.md` (family-level dispatch), `net/ipv6/icmpv6.md` (RAW ICMPv6 socket alternative to SOCK_DGRAM ping socket).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| RAW IPv6 socket family + per-socket state + RX delivery + TX | `net/ipv6/raw.c` |
| Generic v6 datagram helpers (cmsg builder shared with UDPv6, ICMPv6 echo) | `net/ipv6/datagram.c` |
| Public API | `include/net/ipv6.h`, `include/net/raw.h` |
| UAPI | `include/uapi/linux/in6.h` (struct icmp6_filter, IPV6_CHECKSUM constants) |

## Compatibility contract

### `socket(AF_INET6, SOCK_RAW, proto)`

CAP_NET_RAW required. `proto` is the IPv6 nexthdr type (e.g., IPPROTO_ICMPV6, IPPROTO_OSPF=89, IPPROTO_RAW=255 for IPv6-header injection mode).

Identical gating + dispatch.

### `ICMPV6_FILTER` setsockopt (RFC 3542 § 3.2)

```c
struct icmp6_filter {
    __u32 data[8];   /* 256-bit bitmap, 1 bit per ICMPv6 type */
};
```

Userspace builds a 256-bit type filter via `ICMP6_FILTER_SETPASS / SETBLOCK / SETPASSALL / SETBLOCKALL / WILLPASS / WILLBLOCK` macros; setsockopt installs. RX delivery checks per-socket filter; types not matching are dropped silently (counter only).

Identical wire format + macros so libc + ndp daemons + ping6 work unchanged.

### `IPV6_CHECKSUM` setsockopt (RFC 3542 § 3.1)

For non-ICMPv6 RAW sockets: per-socket integer offset where kernel computes + writes upper-layer checksum on TX (and verifies on RX). `-1` disables.

For ICMPv6 RAW sockets: kernel always computes ICMPv6 checksum (RFC 3542 § 3.1 mandates); IPV6_CHECKSUM setsockopt rejected with EINVAL.

Identical semantics.

### TX with `IPPROTO_RAW`

When `proto == IPPROTO_RAW`: sendmsg payload includes the full IPv6 header (kernel does not insert one); used for IPv6-header-injection tools (e.g., scapy, raw fuzzers). Requires CAP_NET_RAW.

Identical semantics.

### RX delivery

Per-netns hash bucket of RAW sockets keyed on (proto, daddr, ifindex). On packet arrival in `ipv6_raw_deliver` (called from `ip6_input.c` after EH walk):
1. Walk per-bucket list of RAW sockets matching nexthdr
2. For each socket: filter by daddr (if bound) + ifindex (if SO_BINDTODEVICE)
3. For ICMPv6 RAW: apply ICMPV6_FILTER bitmap
4. Clone skb + queue to socket's recv queue

Multiple RAW sockets matching same packet each receive a clone. Identical algorithm.

### `/proc/net/raw6`

Per-socket line: `sl local_address remote_address st tx_queue rx_queue tr tm->when retrnsmt uid timeout inode ref pointer drops`. Format-byte-identical so existing tools work.

### NETLINK_SOCK_DIAG `INET_DIAG_REQ_FAMILY=AF_INET6 + INET_DIAG_REQ_PROTOCOL=<proto>`

Wire format byte-identical (cross-ref `net/ipv4/raw.md`).

### cmsg surface (shared with `datagram.c`)

When the appropriate `IPV6_RECV*` setsockopt is set: cmsg with received PKTINFO / HOPLIMIT / TCLASS / RTHDR / HOPOPTS / DSTOPTS / FLOWINFO / TUNNEL_INFO. Identical layout + ordering.

### sysctl `/proc/sys/net/ipv4/ping_group_range`

Despite the name, also gates SOCK_DGRAM IPPROTO_ICMPV6 ("unprivileged ICMPv6 ping socket") — not a RAW socket. RAW sockets always require CAP_NET_RAW.

## Requirements

- REQ-1: `socket(AF_INET6, SOCK_RAW, proto)` gated by CAP_NET_RAW; rejection ACL'd via LSM hooks.
- REQ-2: ICMPV6_FILTER setsockopt: 256-bit-per-type bitmap; per-RFC-3542 § 3.2 macro semantics; reject types not matching at RX delivery.
- REQ-3: IPV6_CHECKSUM setsockopt: per-socket integer offset for TX checksum + RX verify; rejected (EINVAL) on ICMPv6 RAW sockets.
- REQ-4: IPPROTO_RAW TX: payload includes full IPv6 header; kernel does not insert; identical semantics.
- REQ-5: Per-netns RX bucket walk: deliver to all matching sockets via skb_clone; per-socket queue.
- REQ-6: cmsg surface for `recvmsg` (PKTINFO / HOPLIMIT / TCLASS / RTHDR / HOPOPTS / DSTOPTS / FLOWINFO) — shared with `datagram.c`.
- REQ-7: `/proc/net/raw6` content format byte-identical.
- REQ-8: NETLINK_SOCK_DIAG IPv6 RAW wire format byte-identical.
- REQ-9: SO_BINDTODEVICE filtering: bound socket only receives packets from the specified ifindex.
- REQ-10: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `socket(AF_INET6, SOCK_RAW, IPPROTO_ICMPV6)` without CAP_NET_RAW → EPERM. (covers REQ-1)
- [ ] AC-2: ICMPV6_FILTER test: filter passes ECHO_REQUEST + ECHO_REPLY only; sending Type 1 (DestUnreach) → not delivered to filtered socket; ECHO_REPLY → delivered. (covers REQ-2)
- [ ] AC-3: IPV6_CHECKSUM test on `IPPROTO_OSPF` socket: setsockopt offset=12; sendmsg with payload → kernel computes checksum at offset 12. (covers REQ-3)
- [ ] AC-4: IPV6_CHECKSUM on ICMPv6 socket → EINVAL. (covers REQ-3)
- [ ] AC-5: IPPROTO_RAW TX test: send a buffer with full IPv6 header + UDP payload → tcpdump sees byte-identical packet (no kernel header insertion). (covers REQ-4)
- [ ] AC-6: Multi-socket RX test: 2 RAW sockets both bound to IPPROTO_OSPF → an OSPFv3 packet delivered to both. (covers REQ-5)
- [ ] AC-7: SO_BINDTODEVICE test: bind RAW socket to ifindex N; packet arriving on ifindex M → not delivered. (covers REQ-9)
- [ ] AC-8: cmsg test: with IPV6_RECVPKTINFO=1, recvmsg returns IPV6_PKTINFO cmsg with received-side ifindex + dst. (covers REQ-6)
- [ ] AC-9: `cat /proc/net/raw6` byte-identical. (covers REQ-7)
- [ ] AC-10: `ss -6 -w` byte-identical. (covers REQ-8)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-10)

## Architecture

### Rust module organization

- `kernel::net::ipv6::raw_v6::RawV6Protocol` — `IPPROTO_RAW` family + per-proto register
- `kernel::net::ipv6::raw_v6::RawV6Sock` — `struct raw6_sock`
- `kernel::net::ipv6::raw_v6::Icmpv6Filter` — 256-bit-per-type bitmap
- `kernel::net::ipv6::raw_v6::ChecksumOffset` — per-socket IPV6_CHECKSUM
- `kernel::net::ipv6::raw_v6::Deliver` — RX bucket walk + clone-to-each-matching
- `kernel::net::ipv6::raw_v6::SendIpv6Raw` — TX (header insertion vs. IPPROTO_RAW)
- `kernel::net::ipv6::raw_v6::proc::ProcRaw6` — `/proc/net/raw6`
- `kernel::net::ipv6::raw_v6::diag::Raw6Diag` — NETLINK_SOCK_DIAG

### Locking and concurrency

- **Per-netns `raw_v6_hashinfo` per-bucket spinlock**: protects per-bucket socket list
- **Per-`raw6_sock` socket-lock**: serializes setsockopt + sendmsg/recvmsg
- **RCU**: bucket-list walk in RX delivery is RCU-side

### Error handling

- `Err(EPERM)` — non-CAP_NET_RAW
- `Err(EINVAL)` — bad ICMPV6_FILTER, bad IPV6_CHECKSUM offset
- `Err(EAGAIN)` — non-blocking recvmsg with empty queue
- `Err(EMSGSIZE)` — payload exceeds MTU on IPPROTO_RAW
- `Err(EFAULT)` — bad userspace ptr

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-netns bucket walk under bucket spinlock | `kani::proofs::net::ipv6::raw_v6::deliver_safety` |
| ICMPV6_FILTER bitmap test (no out-of-bounds bit lookup) | `kani::proofs::net::ipv6::raw_v6::filter_safety` |
| IPV6_CHECKSUM offset write (skb-bounds-checked) | `kani::proofs::net::ipv6::raw_v6::checksum_safety` |
| IPPROTO_RAW header-injection bounds | `kani::proofs::net::ipv6::raw_v6::ipproto_raw_safety` |

### Layer 2: TLA+ models

(none mandatory at this level)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns RAW v6 hashinfo | every socket appears in exactly one bucket determined by `inet6_ehashfn(proto, daddr, ifindex)` | `kani::proofs::net::ipv6::raw_v6::hash_invariants` |
| ICMPv6 filter bitmap | bitmap covers exactly 256 ICMPv6 types (0-255) | `kani::proofs::net::ipv6::raw_v6::filter_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/ipv6/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | raw6_sock + per-skb-clone refcounts use `Refcount` (saturating) | § Mandatory |
| **MEMORY_SANITIZE** | freed raw6_sock + per-recv-queue skbs cleared | § Default-on configurable off |
| **SIZE_OVERFLOW** | IPV6_CHECKSUM offset arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, SIZE_OVERFLOW**: see above
- **CONSTIFY**: `rawv6_prot` proto-ops `static const`
- **USERCOPY**: sendmsg/recvmsg payload via `copy_to_iter`/`copy_from_iter`
- **KERNEXEC**: per-proto register table dispatched via `static const fn-ptr`

### Row-2 / GR-RBAC integration

- LSM hook `security_socket_create` for AF_INET6 SOCK_RAW (CAP_NET_RAW gate already standard); GR-RBAC policy can deny per-subject (default empty).
- LSM hook `security_socket_setsockopt` for ICMPV6_FILTER + IPV6_CHECKSUM.
- Useful default GR-RBAC policy: deny SOCK_RAW outside gradm-marked `raw_socket_users` role; CAP_NET_RAW is a frequent capability-escape target.

### Userspace-visible behavior changes

None beyond upstream defaults. CAP_NET_RAW gate identical.

### Verification

(See § Verification above.)

## Open Questions

(none — RAW IPv6 socket ABI is exhaustively specified by RFC 3542 + upstream)

## Out of Scope

- ICMPv6 SOCK_DGRAM ping socket (cross-ref `net/ipv6/icmpv6.md` REQ-5)
- 32-bit-only paths
- Implementation code
