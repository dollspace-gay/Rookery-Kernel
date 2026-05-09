---
title: "Tier-3: net/ipv6/udp-v6 — UDP-over-IPv6 (v6-specific socket layer)"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the IPv6-specific UDP socket layer: `udpv6_protocol` registration, `udp6_sock` (struct that contains `udp_sock` + `ipv6_pinfo`), v6-aware sendmsg/recvmsg, dual-stack v4-mapped redirection, v6-aware checksum (with optional zero-checksum permitted only over tunnel encapsulation per RFC 6935 / 6936), UDPv6 GRO/GSO segmentation offload, UDPv6 tunnel encapsulation (used by VXLAN-over-IPv6, GTP-over-IPv6, GENEVE-over-IPv6, etc.), and the v6 hashtable-lookup helpers `udp6_lib_lookup`.

The UDP datagram path + sendmsg/recvmsg buffer mgmt + corking + checksum offload are inherited from `net/ipv4/udp.md`. This Tier-3 covers the v6 entry-points + dual-stack adapter + IPv6-checksum-zero rules only.

UDP-Lite (CONFIG_IP_UDPLITE) is folded into `udp.c` upstream; this Tier-3 covers the v6 UDP-Lite path within udp.c via `udplite_protocol` second protosw entry.

Sub-tier-3 of `net/ipv6/00-overview.md`. Pairs with `net/ipv4/udp.md` (parent abstraction), `net/ipv6/af-inet6.md` (family-level dispatch), `net/ipv6/ip6-output.md` (TX path).

### Requirements

- REQ-1: `udpv6_protocol` registered in inet6_protosw for `(SOCK_DGRAM, IPPROTO_UDP)` and `(SOCK_DGRAM, IPPROTO_UDPLITE)`.
- REQ-2: `struct udp6_sock` first-cache-line layout-equivalent.
- REQ-3: Dual-stack: AF_INET6 UDP socket sends to v4-mapped destination → routed via IPv4 datapath; per-message determination since UDP is connectionless.
- REQ-4: v6-aware UDP checksum: pseudo-header per RFC 8200 § 8.1; zero-checksum permitted only when `udp6_zero_csum_allowed(skb)` returns true (tunnel encap on whitelisted protos per RFC 6935/6936); otherwise rejected.
- REQ-5: UDPv6 GRO: aggregate consecutive same-5-tuple UDP-over-IPv6 datagrams; per `dev->features` and per-socket `UDP_GRO`.
- REQ-6: UDPv6 GSO: `UDP_SEGMENT` setsockopt opts in; `udp_send_skb` segments oversize datagrams; per-driver TSO offload via `dev->features`.
- REQ-7: UDP tunnel encap (VXLAN/GENEVE/GTP-over-IPv6): `setup_udp_tunnel_sock` creates kernel-side UDP socket with tunnel-encap-callback; identical UAPI for tunnel modules.
- REQ-8: UDP-Lite (`IPPROTO_UDPLITE`): full implementation per RFC 3828; UDPLITE_SEND_CSCOV / UDPLITE_RECV_CSCOV setsockopt.
- REQ-9: cmsg surface for `recvmsg` (IPV6_PKTINFO / IPV6_HOPLIMIT / IPV6_TCLASS / IPV6_RTHDR / IPV6_HOPOPTS / IPV6_DSTOPTS / IPV6_FLOWINFO) — shared with `datagram.c`; identical layout + ordering.
- REQ-10: `__udp6_lib_lookup` hashtable lookup; identical bucket sizing + key construction.
- REQ-11: `/proc/net/udp6` + `/proc/net/udplite6` content format byte-identical.
- REQ-12: NETLINK_SOCK_DIAG IPv6 wire format byte-identical for both UDP and UDP-Lite.
- REQ-13: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct udp6_sock` byte-identical first cache-line. (covers REQ-2)
- [ ] AC-2: UDP-over-IPv6 send-recv test: `nc -6 -u -l ::1 8080` + `nc -6 -u ::1 8080` round-trips bytes; tcpdump shows correct pseudo-header checksum. (covers REQ-1, REQ-4)
- [ ] AC-3: Dual-stack UDP test: AF_INET6 UDP socket bound `:::8080` (V6ONLY=0) receives from IPv4 sender to `127.0.0.1:8080`; the recvfrom-returned sin6_addr is `::ffff:127.0.0.1`. (covers REQ-3)
- [ ] AC-4: Zero-checksum test: VXLAN-over-IPv6 with zero-checksum mode succeeds (whitelisted); other proto with zero-checksum is dropped + counter increment. (covers REQ-4)
- [ ] AC-5: UDPv6 GSO: `setsockopt(UDP_SEGMENT, &MTU)`; sendto a 64KB buffer → driver receives MTU-sized segments (or single skb with GSO offload if NIC supports). (covers REQ-6)
- [ ] AC-6: UDPv6 GRO: receive 4 same-5-tuple UDPv6 datagrams; per-CPU aggregator builds a single skb_frag_list. (covers REQ-5)
- [ ] AC-7: VXLAN-over-IPv6 test: configure VXLAN with v6 VTEP, traffic encaps in UDPv6 with vxlan port; tunnel UDP socket receives + decap callback fires. (covers REQ-7)
- [ ] AC-8: UDP-Lite test: UDPLITE_SEND_CSCOV=8 (header-only checksum); receiver with UDPLITE_RECV_CSCOV=8 accepts; with UDPLITE_RECV_CSCOV=16 (stricter) rejects since coverage too short. (covers REQ-8)
- [ ] AC-9: cmsg test: with IPV6_RECVPKTINFO=1, recvmsg returns IPV6_PKTINFO cmsg with correct ifindex + dst. (covers REQ-9)
- [ ] AC-10: `cat /proc/net/udp6` + `cat /proc/net/udplite6` byte-identical. (covers REQ-11)
- [ ] AC-11: `ss -6 -u -i` byte-identical. (covers REQ-12)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

### Architecture

### Rust module organization

- `kernel::net::ipv6::udp_v6::UdpV6Protocol` — protocol register + ops vtable
- `kernel::net::ipv6::udp_v6::Udp6Sock` — `struct udp6_sock` wrapper
- `kernel::net::ipv6::udp_v6::dual_stack::DualStackDatagram` — per-msg v4-mapped redirect
- `kernel::net::ipv6::udp_v6::checksum::PseudoHeader` — RFC 8200 § 8.1 checksum
- `kernel::net::ipv6::udp_v6::checksum::ZeroCsumPolicy` — RFC 6935/6936 whitelist
- `kernel::net::ipv6::udp_v6::offload::UdpV6Gro`, `UdpV6Gso` — segmentation offload
- `kernel::net::ipv6::udp_v6::tunnel::Udp6Tunnel` — tunnel encap helper
- `kernel::net::ipv6::udp_v6::lookup::UdpV6Lookup` — hashtable lookup
- `kernel::net::ipv6::udp_v6::datagram::DatagramCmsg` — cmsg builder (shared with RAWv6, ICMPv6 echo)
- `kernel::net::ipv6::udp_v6::udplite::UdpLiteV6` — UDP-Lite v6 entrypoints
- `kernel::net::ipv6::udp_v6::proc::ProcUdp6` — `/proc/net/udp6` + `udplite6`
- `kernel::net::ipv6::udp_v6::diag::Udp6Diag` — NETLINK_SOCK_DIAG

### Locking and concurrency

- (Inherits UDP locking from `net/ipv4/udp.md`)
- **Per-udp6_sock socket-lock**: shared with IPv4 path
- **udp_table bucket spinlocks**: per-bucket; shared with IPv4 (different table instance)

### Error handling

(Inherits UDP error model from `net/ipv4/udp.md` — EAGAIN, EMSGSIZE, ECONNREFUSED on ICMP echo, ENOBUFS, etc.)

### Out of Scope

- UDP datagram path + sendmsg/recvmsg buffer mgmt + corking (cross-ref `net/ipv4/udp.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| UDP-over-IPv6 socket layer (incl. UDP-Lite) | `net/ipv6/udp.c` |
| UDPv6 GRO/GSO offload | `net/ipv6/udp_offload.c` |
| UDPv6 tunnel encap helpers (VXLAN/GENEVE/etc. over IPv6) | `net/ipv6/ip6_udp_tunnel.c` |
| Generic v6 datagram helpers (used by UDPv6 + RAWv6 + ICMPv6 echo) | `net/ipv6/datagram.c` |
| Public API | `include/net/transp_v6.h`, `include/net/udp.h` |

### compatibility contract

### `udpv6_protocol` registration

Two entries: `(SOCK_DGRAM, IPPROTO_UDP)` for regular UDP, and `(SOCK_DGRAM, IPPROTO_UDPLITE)` for UDP-Lite. Identical setsockopt surfaces.

### `struct udp6_sock` layout

```c
struct udp6_sock {
    struct udp_sock      udp;
    struct ipv6_pinfo    inet6;  /* via inet_sock embedding */
};
```

(Layout via `inet_sock.pinet6 = &udp6_sock.inet6` for AF_INET6 sockets.)

Layout-equivalent.

### Dual-stack v4-mapped redirection

When an AF_INET6 UDP socket (without `IPV6_V6ONLY=1`) sends to / receives from an IPv4-mapped destination, the kernel routes via the IPv4 UDP path identically. Per-message redirection (vs. per-socket for TCP) since UDP is connectionless.

### v6-aware checksum (RFC 8200 § 8.1)

UDP-over-IPv6 pseudo-header includes the same fields as TCP-over-IPv6 (saddr/daddr/upper-layer-len/nexthdr). RFC 8200 § 8.1 originally mandated non-zero UDP checksum on IPv6 (vs. optional on IPv4); RFC 6935 + 6936 permit zero checksum for tunnel encapsulation under conditions: tunnel proto is in the kernel's `udpv6_zero_csum_allowed` list (VXLAN/GENEVE/etc.) AND tunnel-specific integrity is provided.

Identical algorithm.

### UDPv6 GRO/GSO offload

Per `dev->features`, `NETIF_F_GRO_FRAGLIST` + `NETIF_F_GSO_UDP_L4`: the kernel can aggregate consecutive same-5-tuple UDP datagrams into a single skb on RX (GRO) or segment a giant UDP send into MTU-sized fragments (GSO). UDP_SEGMENT setsockopt opts in.

### `recvmsg` cmsg surface (shared with `datagram.c`)

When `IPV6_RECVPKTINFO` set: cmsg `IPV6_PKTINFO` with `struct in6_pktinfo` (ifindex + dst).
When `IPV6_RECVHOPLIMIT` set: cmsg `IPV6_HOPLIMIT` with received hoplimit.
When `IPV6_RECVTCLASS` set: cmsg `IPV6_TCLASS` with received TCLASS.
When `IPV6_RECVRTHDR` set: cmsg `IPV6_RTHDR` with received Routing Header.
When `IPV6_RECVHOPOPTS`/`IPV6_RECVDSTOPTS` set: cmsg with HBH/DestOpt.
When `IPV6_RECVFLOWINFO` set: cmsg with received flow info.

Identical layout + ordering.

### UDP-Lite (RFC 3828)

`SOCK_DGRAM IPPROTO_UDPLITE` socket; `UDPLITE_SEND_CSCOV` / `UDPLITE_RECV_CSCOV` setsockopt sets coverage; checksum covers only the first `cscov` bytes of payload (header always covered). Identical UAPI.

### UDPv6 hashtable

Per-netns udphash + udplitehash (independent table from IPv4 since v6 keys differ); shared infrastructure with IPv4 via `udp_table`. Lookup helper `__udp6_lib_lookup`.

### `/proc/net/udp6` + `/proc/net/udplite6`

Per-socket line: `sl local_address remote_address st tx_queue rx_queue tr tm->when retrnsmt uid timeout inode ref pointer drops`. Format-byte-identical.

### NETLINK_SOCK_DIAG `INET_DIAG_REQ_FAMILY=AF_INET6 + INET_DIAG_REQ_PROTOCOL=IPPROTO_UDP/UDPLITE`

Wire format byte-identical (cross-ref `net/ipv4/udp.md` for diag wire schema).

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Pseudo-header checksum bounds | `kani::proofs::net::ipv6::udp_v6::pseudo_safety` |
| Zero-checksum policy whitelist check | `kani::proofs::net::ipv6::udp_v6::zerocsum_safety` |
| GRO aggregator skb_frag_list build | `kani::proofs::net::ipv6::udp_v6::gro_safety` |
| GSO segmentation arithmetic | `kani::proofs::net::ipv6::udp_v6::gso_safety` |
| UDP-Lite cscov enforcement | `kani::proofs::net::ipv6::udp_v6::udplite_safety` |
| Tunnel encap-callback registration | `kani::proofs::net::ipv6::udp_v6::tunnel_safety` |

### Layer 2: TLA+ models

(none owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-udp6_sock zero-csum permission | `udp_no_check6_tx` is set ⇔ `tunnel_id` is registered in zero-csum-allowed list | `kani::proofs::net::ipv6::udp_v6::zerocsum_invariants` |
| UDP-Lite cscov | `pcsicv ≤ skb->len`; both send and recv cscovs ≥ UDPLITE_MIN_CSCOV (8) | `kani::proofs::net::ipv6::udp_v6::udplite_invariants` |
| GRO frag-list | aggregated skbs all share 5-tuple; ordering preserved | `kani::proofs::net::ipv6::udp_v6::gro_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/ipv6/00-overview.md`)

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | udp6_sock + per-tunnel-callback refcounts use `Refcount` (saturating) | § Mandatory |
| **MEMORY_SANITIZE** | freed udp6_sock + per-recv-queue skbs cleared (datagram payload sensitive) | § Default-on configurable off |
| **SIZE_OVERFLOW** | UDP length + cscov arithmetic uses checked operators (CVE class: CVE-2019-class UDP-len underflow) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, SIZE_OVERFLOW**: see above
- **CONSTIFY**: `udpv6_prot` proto-ops + per-protosw entries `static const`
- **USERCOPY**: sendmsg/recvmsg payload via `copy_to_iter`/`copy_from_iter`
- **KERNEXEC**: tunnel-callback dispatch via `static const fn-ptr` array per registered tunnel proto

### Row-2 / GR-RBAC integration

- LSM hook `security_socket_create` for AF_INET6 SOCK_DGRAM
- LSM hook `security_socket_setsockopt` for `UDP_SEGMENT` / `UDPLITE_*_CSCOV`
- Default GR-RBAC policy: empty so behavior matches upstream

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

