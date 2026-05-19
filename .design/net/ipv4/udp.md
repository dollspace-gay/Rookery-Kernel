# Tier-3: net/ipv4/udp — User Datagram Protocol (RFC 768)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv4/udp.c
  - net/ipv4/udp_diag.c
  - net/ipv4/udp_offload.c
  - net/ipv4/udp_bpf.c
  - net/ipv4/udp_tunnel_core.c
  - net/ipv4/udp_tunnel_nic.c
  - include/net/udp.h
  - include/net/udp_tunnel.h
  - include/uapi/linux/udp.h
-->

## Summary
Tier-3 design for UDP (User Datagram Protocol) on IPv4. Owns UDP socket sendmsg/recvmsg paths, the UDP-style hash table for fast `(saddr, sport, daddr, dport)` lookup, GRO + GSO for UDP, the udp_tunnel framework (used by VxLAN, GENEVE, FOU, GUE encapsulations), inet_diag dump, BPF integration (sockmap), and UDP-Lite per RFC 3828 (now folded into udp.c).

Sub-tier-3 of `net/ipv4/00-overview.md`. UDP is the second most-used transport — DNS, NTP, DHCP, QUIC (over UDP), every game, every video conference, every container-overlay tunnel.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| UDP core | `net/ipv4/udp.c` |
| inet_diag dump | `net/ipv4/udp_diag.c` |
| UDP GSO + GRO | `net/ipv4/udp_offload.c` |
| BPF integration (sockmap, sk_msg) | `net/ipv4/udp_bpf.c` |
| UDP tunnel framework | `net/ipv4/udp_tunnel_core.c`, `net/ipv4/udp_tunnel_nic.c` |
| Public types | `include/net/udp.h`, `include/net/udp_tunnel.h` |
| UAPI | `include/uapi/linux/udp.h` |

## Compatibility contract

### Wire format

UDP wire format per RFC 768: 8-byte header (src port, dst port, length, checksum) + payload. Byte-identical.

### `struct udp_sock` layout

`include/linux/udp.h` defines `struct udp_sock`. First-cache-line + commonly-accessed fields layout-equivalent to upstream.

### UDP options

- `UDP_CORK` (TCP_CORK analog; bundles small writes)
- `UDP_NO_CHECK6_TX` / `UDP_NO_CHECK6_RX` (IPv6 over IPv4 zero-checksum)
- `UDP_SEGMENT` (UFO via UDP-GSO)
- `UDP_GRO` (enable UDP receive aggregation)
- `UDP_ENCAP_*` (encap-protocol identifiers for udp_tunnel)
- `UDP_LITE_RECV_CSCOV` / `UDP_LITE_SEND_CSCOV` (UDP-Lite checksum coverage)

Numeric values byte-identical.

### `/proc/net/udp` + `/proc/net/udp6`

Format: per-socket `<sl> <local_address>:<port> <remote_address>:<port> <state> <tx_queue>:<rx_queue> <tr>:<tm_when> <retrnsmt> <uid> <timeout> <inode>`. Format-identical so `ss -u` / `netstat -un` work unmodified.

### UDP encap (used by VxLAN, GENEVE, FoU)

`udp_tunnel_setup_udp_sock_via_socket` registers a UDP socket as an encap endpoint. Each encap protocol (VxLAN, GENEVE, FoU, GUE, MPLS-over-UDP) sets `udp_sk->encap_rcv` callback. Identical.

### Sysctls (`net.ipv4.udp_*`)

`udp_mem`, `udp_rmem_min`, `udp_wmem_min`, `udp_l3mdev_accept`. Format + value semantics identical.

## Requirements

- REQ-1: UDP wire format byte-identical to RFC 768.
- REQ-2: `struct udp_sock` layout-equivalent for first-cache-line.
- REQ-3: UDP socket lifecycle: bind (port allocation via udp_lib_get_port + ephemeral-port range honored) → use → close. Identical behavior.
- REQ-4: UDP receive: skb routes through netfilter (cross-ref `net/netfilter.md`) → encap-rx if applicable → udp_lib_rcv → enqueue on per-sock receive queue → wake `sk_data_ready`.
- REQ-5: UDP send: app writes via sendmsg → udp_sendmsg builds skb → ip_make_skb → ip_send_skb → udp_send_skb → IP-output (cross-ref `net/ipv4/00-overview.md`).
- REQ-6: UDP GSO + GRO: tx-side GSO via UDP_SEGMENT sockopt; rx-side GRO via udp_gro_receive. Identical heuristics.
- REQ-7: UDP-Lite (RFC 3828): partial-checksum UDP variant; CSCOV sockopts. Identical.
- REQ-8: UDP encap framework: VxLAN, GENEVE, FoU, GUE all register via udp_tunnel_*. Identical.
- REQ-9: BPF integration: sockmap + sk_msg + sk_skb via udp_bpf.c. Identical.
- REQ-10: inet_diag dump for UDP: `ss -u -i` produces identical output.
- REQ-11: UDP fragmentation: outgoing UDP packets > MTU fragment per IP-output rules; PMTU discovery integration.
- REQ-12: UDP-corking: UDP_CORK + uncork bundles writes into single send.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: A `dnsperf` workload over Rookery achieves throughput within ±5% of upstream. (covers REQ-1, REQ-3, REQ-4, REQ-5)
- [ ] AC-2: `pahole struct udp_sock` first-cache-line byte-identical. (covers REQ-2)
- [ ] AC-3: A `bind(0)` + `getsockname` test verifies ephemeral port range honored. (covers REQ-3)
- [ ] AC-4: A UDP-GSO test (UDP_SEGMENT sockopt) sends 64KB frames; receiver assembles via UDP-GRO; data integrity preserved. (covers REQ-6)
- [ ] AC-5: A UDP-Lite test (UDP_LITE_SEND_CSCOV=8) produces packets with partial checksum; receiver accepts. (covers REQ-7)
- [ ] AC-6: A VxLAN test creates a vxlan netdev (cross-ref `net/netdev.md` + per-tunnel docs); UDP encap routing works. (covers REQ-8)
- [ ] AC-7: A sockmap-based UDP redirect test routes packets between two UDP sockets via BPF. (covers REQ-9)
- [ ] AC-8: `ss -u` output for active UDP sockets byte-identical. (covers REQ-10)
- [ ] AC-9: A UDP fragmentation test (`sendto` of 8K with MTU 1500) produces 6 fragmented IP packets per upstream. (covers REQ-11)
- [ ] AC-10: A `setsockopt(UDP_CORK)` + 100 small writes + `setsockopt(UDP_CORK=0)` produces a single bundled UDP packet. (covers REQ-12)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::ipv4::udp::UdpSock` — `struct udp_sock` wrapper
- `kernel::net::ipv4::udp::sendmsg` — TX path
- `kernel::net::ipv4::udp::recvmsg` — RX path
- `kernel::net::ipv4::udp::hash::UdpHash` — port-hash + 4-tuple-hash tables
- `kernel::net::ipv4::udp::offload::Gso` / `Gro` — UDP-GSO / UDP-GRO
- `kernel::net::ipv4::udp::tunnel::UdpTunnel` — encap framework
- `kernel::net::ipv4::udp::lite::UdpLite` — UDP-Lite (RFC 3828)
- `kernel::net::ipv4::udp::diag::UdpDiag` — inet_diag dump
- `kernel::net::ipv4::udp::bpf::UdpBpf` — sockmap + sk_msg

### Locking and concurrency

- **Per-port hash buckets** (spinlock per bucket): protect bind-list
- **Per-`udp_sock` lock** (cross-ref `net/struct-sock.md`): protects per-sock state
- **Per-receive-queue lock** (cross-ref `net/skbuff.md`): protects receive queue

### Error handling

- `Err(EADDRINUSE)` — port already bound
- `Err(EAFNOSUPPORT)` — bad family
- `Err(EMSGSIZE)` — packet too large
- `Err(EAGAIN)` — would-block on non-blocking
- `Err(ECONNREFUSED)` — port-unreachable on connected UDP

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| UDP header parse + write | `kani::proofs::net::udp::hdr_safety` |
| UDP-GSO/GRO segmentation arithmetic | `kani::proofs::net::udp::offload_safety` |
| Tunnel-encap rx callback dispatch | `kani::proofs::net::udp::tunnel_safety` |

### Layer 2: TLA+ models

(none mandatory — UDP is mostly stateless; concurrency at the sock layer)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Port hash | Each bound port appears in exactly one bucket | `kani::proofs::net::udp::port_hash_invariants` |
| 4-tuple hash | Connected sockets indexed by 4-tuple | `kani::proofs::net::udp::4tuple_hash_invariants` |

### Layer 4: Functional correctness (opt-in)

- **UDP checksum + UDP-Lite checksum** correctness via Verus — proves: tx + rx checksum computations match RFC 768 + RFC 3828.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | udp_sock refcount uses `Refcount` | § Mandatory |
| **AUTOSLAB** | udp_sock allocated via per-protocol slab cache | § Mandatory |
| **rp_filter** (reverse-path filter; defends against IP spoof) | sysctl `net.ipv4.conf.*.rp_filter` default-on; matches upstream | (existing) |

### Row-1 features consumed by this component

- **REFCOUNT**, **AUTOSLAB**, **MEMORY_SANITIZE**: per-sock + per-skb (cross-ref struct-sock + skbuff)
- **UDEREF**: sendmsg/recvmsg via iov_iter
- **CONSTIFY**: udp_protocol vtable static const
- **SIZE_OVERFLOW**: UDP-len + offset arithmetic uses checked operators

### Row-2 / GR-RBAC integration

LSM hooks (cross-ref `net/socket-api.md`): `security_socket_bind`, `security_socket_sendmsg`, `security_socket_recvmsg`. GR-RBAC's policy can deny UDP send/recv for specific src/dst combinations.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on every sendmsg iovec, recvmsg user-buffer, and SOL_UDP getsockopt/setsockopt blob; UDP-Lite CSCOV-bounded copies validated.
- **PAX_KERNEXEC** — W^X on JIT'd sockmap/sk_msg BPF programs, per-encap `udp_sk->encap_rcv` callbacks (VxLAN, GENEVE, FoU), and udp_tunnel offload vtables.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every UDP recvmsg/sendmsg entry; UDP is the densest stateless syscall path (DNS, QUIC, NTP).
- **PAX_REFCOUNT** — saturating refcount on `udp_sock`, per-port hash bucket, encap binding entries, and udp_tunnel sock references.
- **PAX_MEMORY_SANITIZE** — zero-on-free for udp_sock cache (especially encap private state in `udp_sk->encap_*` for VxLAN/GENEVE/FoU/GUE).
- **PAX_UDEREF / PAX_MEMORY_UDEREF** — sendmsg/recvmsg via iov_iter; setsockopt optval (especially `UDP_LITE_*_CSCOV`, `UDP_SEGMENT`, `UDP_GRO`) via typed `UserPtr<...>`.
- **GRKERNSEC_HIDESYM** — hide kernel pointers in `/proc/net/udp{,6}`, `ss -u -i` output, `inet_diag` netlink dumps.
- **GRKERNSEC_NO_SIMULT_CONNECT** — rate-limit same-uid UDP-connected sockets to identical dst:port (mitigates UDP-reflection amplification).
- **GRKERNSEC_BLACKHOLE** — silent drop of probe packets (ICMP port-unreachable suppression on closed UDP ports); rate-limited per-source.
- **GRKERNSEC_RANDNET** — randomize UDP ephemeral source port allocation, UDP-tunnel VNI/encap source ports, and per-flow GRO/GSO hash seeds.
- **GRKERNSEC_NETFILTER** — UDP conntrack interaction; `sk->sk_mark`/`sk->sk_priority` mutation restricted to CAP_NET_ADMIN.
- **GRKERNSEC_SOCK_PRIV** — udp_tunnel encap-sock registration audited; privileged-port bind (< 1024) gated.
- **PAX_SIZE_OVERFLOW** — UDP-len + cmsg-length + per-flow offset arithmetic uses checked operators; UDP-GSO segment-count saturating.
- **CAP_NET_RAW / CAP_NET_BIND_SERVICE** strict — UDP-Lite raw-mode requires CAP_NET_RAW in the binding userns; port < 1024 requires CAP_NET_BIND_SERVICE.

Per-doc rationale: UDP is stateless but bridges to QUIC, DNS, NTP, DHCP, VxLAN, GENEVE, FoU, GUE — every container-overlay and every video conference rides UDP. PaX/Grsecurity reinforcement focuses on (a) saturating refs on udp_tunnel encap sockets (long-lived; shared across many flows), (b) zero-on-free for encap private state which may carry tunnel-key material, (c) GRKERNSEC_BLACKHOLE for the well-known UDP-reflection amplification class (NTP monlist, DNS open-resolver, memcached), and (d) GRKERNSEC_RANDNET for ephemeral port allocation since QUIC connection identifiers ride alongside.

## Open Questions

(none — UDP semantics are exhaustively specified by RFC + Linux extensions)

## Out of Scope

- UDP over IPv6 (cross-ref `net/ipv6/udp6.md` Tier-3; mostly shares this Tier-3's logic)
- VxLAN/GENEVE/FoU per-tunnel detail (cross-ref individual tunnel docs)
- 32-bit-only paths
- Implementation code
