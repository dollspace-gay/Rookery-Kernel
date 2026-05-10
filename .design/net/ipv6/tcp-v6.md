# Tier-3: net/ipv6/tcp-v6 — TCP-over-IPv6 (v6-specific socket layer)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/tcp_ipv6.c
  - include/net/transp_v6.h
  - include/net/tcp.h
-->

## Summary
Tier-3 design for the IPv6-specific TCP socket layer: `tcpv6_protocol` registration, `tcp6_sock` (struct that contains `tcp_sock` + `ipv6_pinfo`), v6-aware connect/accept/listen/sendmsg/recvmsg, dual-stack v4-mapped redirection (when an IPv4-mapped destination is connected on an AF_INET6 socket), v6-aware checksum (pseudo-header includes IPv6 saddr/daddr), MD5/AO TCP authentication for IPv6, and the v6 hashtable-lookup helpers `tcp6_lookup_listener` / `__inet6_lookup_established`.

The TCP state machine + congestion control + TIME_WAIT handling + sendmsg-driven segmentation are inherited from `net/ipv4/tcp.md`. This Tier-3 covers the v6 entry-points + dual-stack adapter only.

Sub-tier-3 of `net/ipv6/00-overview.md`. Pairs with `net/ipv4/tcp.md` (parent abstraction), `net/ipv6/af-inet6.md` (family-level dispatch), `net/ipv6/ip6-output.md` (TX path).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| TCP-over-IPv6 socket layer | `net/ipv6/tcp_ipv6.c` |
| Public API | `include/net/transp_v6.h`, `include/net/tcp.h` |

## Compatibility contract

### `tcpv6_protocol` registration

Registered in inet6_protosw with type=SOCK_STREAM, protocol=IPPROTO_TCP. Identical setsockopt surface (SOL_TCP) shared with IPv4 TCP path; v6-specific surface lives in SOL_IPV6 (cross-ref `net/ipv6/af-inet6.md`).

### `struct tcp6_sock` layout

```c
struct tcp6_sock {
    struct tcp_sock      tcp;
    struct ipv6_pinfo    inet6;  /* via inet_sock embedding */
};
```

(In upstream: `tcp_sock` embeds `inet_connection_sock` which embeds `inet_sock` with `pinet6` pointer; for AF_INET6 sockets, `pinet6 = &tcp6_sock.inet6`.)

Layout-equivalent — `tcp_sock` first, then `ipv6_pinfo`.

### Dual-stack v4-mapped redirection

When an AF_INET6 TCP socket (without `IPV6_V6ONLY=1`) `connect()`s to an IPv4-mapped IPv6 address (`::ffff:0:0/96`), the kernel redirects to the IPv4 TCP path:
1. Detect v4-mapped via `ipv6_addr_v4mapped(&saddr)` / `(&daddr)`
2. Switch `sk->sk_prot` to `tcp_prot` (IPv4) for this socket's lifetime
3. Subsequent operations use IPv4 datapath transparently

Identical mechanism so dual-stack listeners see IPv4 connections.

### v6-aware pseudo-header checksum

TCP-over-IPv6 pseudo-header (RFC 8200 § 8.1):
```
+---------------------+
| Source IPv6 Address | (16 octets)
+---------------------+
|  Dest IPv6 Address  | (16 octets)
+---------------------+
| Upper-Layer Length  | (4 octets)
+---------------------+
| Zero | Next Header  | (3 octets zero, 1 octet "TCP"=6)
+---------------------+
```

Identical layout. Checksum computed over pseudo-header + TCP header + payload.

### TCP MD5 / TCP-AO for IPv6

Per `RFC 5925` (TCP-AO) and per `RFC 2385` (TCP-MD5): TCP-MD5/AO option carries authentication; per-socket key store via `TCP_MD5SIG_EXT` setsockopt with `tcpm_addr` as IPv6 sockaddr_in6. Identical UAPI.

### Hashtable lookups

IPv6 TCP socket lookup via `__inet6_lookup_listener` (for listening) + `__inet6_lookup_established` (for ESTABLISHED), shared infrastructure with IPv4. Algorithm + bucket sizing identical so `ss -6 -t` performance matches.

### `tcp6_seq_show` `/proc/net/tcp6`

Per-TCP6-socket line: `sl local_address remote_address st tx_queue rx_queue tr tm->when retrnsmt uid timeout inode ref pointer drops opt_size sk_drops`. Format-byte-identical so `ss -6 -t` listing parses correctly.

### NETLINK_SOCK_DIAG `INET_DIAG_REQ_FAMILY=AF_INET6 + INET_DIAG_REQ_PROTOCOL=IPPROTO_TCP`

Wire format byte-identical (cross-ref `net/ipv4/tcp.md` for the diag wire schema).

## Requirements

- REQ-1: `tcpv6_protocol` registered in inet6_protosw (SOCK_STREAM, IPPROTO_TCP).
- REQ-2: `struct tcp6_sock` first-cache-line layout-equivalent.
- REQ-3: Dual-stack v4-mapped redirection: AF_INET6 socket + non-V6ONLY + connect to ::ffff:0:0/96 → switch sk_prot to tcp_prot; subsequent operations on IPv4 datapath.
- REQ-4: TCP-over-IPv6 pseudo-header checksum: per RFC 8200 § 8.1; identical computation.
- REQ-5: `__inet6_lookup_listener` + `__inet6_lookup_established` semantics identical (shared infra with IPv4).
- REQ-6: `tcp_v6_connect`: builds flowi6 + ip6_route_output → tcp_connect; identical sequence.
- REQ-7: `tcp_v6_do_rcv` per-state RX dispatch: identical state machine entry.
- REQ-8: TCP-MD5 (`RFC 2385`) for IPv6: per-socket key store with v6 keys; setsockopt UAPI identical.
- REQ-9: TCP-AO (`RFC 5925`) for IPv6: per-socket KMP key store; identical UAPI.
- REQ-10: `tcpv6_send_reset`, `tcpv6_send_check` helpers identical.
- REQ-11: `/proc/net/tcp6` content format byte-identical.
- REQ-12: NETLINK_SOCK_DIAG `INET_DIAG_REQ_PROTOCOL=IPPROTO_TCP` + `INET_DIAG_REQ_FAMILY=AF_INET6` wire-format byte-identical.
- REQ-13: SYN cookies for IPv6 (when synflood detected): `tcp_v6_syn_recv_sock` integrates with the syncookies module identically.
- REQ-14: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct tcp6_sock` byte-identical first cache-line. (covers REQ-2)
- [ ] AC-2: TCP-over-IPv6 connect test: `nc -6 ::1 8080` to `nc -6 -l ::1 8080` round-trips bytes; pseudo-header checksum verifiable via tcpdump. (covers REQ-1, REQ-4, REQ-6)
- [ ] AC-3: Dual-stack test: AF_INET6 listener (V6ONLY=0) on `:::8080` accepts IPv4 connections to `0.0.0.0:8080`; the AF_INET6 accept'd socket has v4-mapped `::ffff:127.0.0.1` peer; bytes round-trip. (covers REQ-3)
- [ ] AC-4: TCP-MD5 test: setsockopt `TCP_MD5SIG_EXT` with v6 sockaddr_in6 key; subsequent SYN carries TCP-MD5 option; mismatched key on receiver rejects. (covers REQ-8)
- [ ] AC-5: TCP-AO test: setsockopt `TCP_AO_ADD_KEY` with v6 sockaddr; subsequent SYN carries TCP-AO option; mismatched MAC rejects. (covers REQ-9)
- [ ] AC-6: SYN flood test: enable `tcp_syncookies=1`; flood SYN to listener; legitimate connection succeeds via syncookie path. (covers REQ-13)
- [ ] AC-7: `cat /proc/net/tcp6` byte-identical content. (covers REQ-11)
- [ ] AC-8: `ss -6 -t -i` byte-identical content (sock_diag wire format). (covers REQ-12)
- [ ] AC-9: Hardening section present and follows template. (covers REQ-14)

## Architecture

### Rust module organization

- `kernel::net::ipv6::tcp_v6::TcpV6Protocol` — protocol register + ops vtable
- `kernel::net::ipv6::tcp_v6::Tcp6Sock` — `struct tcp6_sock` wrapper
- `kernel::net::ipv6::tcp_v6::dual_stack::DualStackRedirect` — v4-mapped switching
- `kernel::net::ipv6::tcp_v6::checksum::PseudoHeader` — RFC 8200 § 8.1 checksum
- `kernel::net::ipv6::tcp_v6::lookup::Inet6Lookup` — listener + established lookup
- `kernel::net::ipv6::tcp_v6::md5::TcpMd5V6` — TCP-MD5 v6 wrapper
- `kernel::net::ipv6::tcp_v6::ao::TcpAoV6` — TCP-AO v6 wrapper
- `kernel::net::ipv6::tcp_v6::syncookies::Syncookies6` — v6 SYN cookies
- `kernel::net::ipv6::tcp_v6::proc::ProcTcp6` — `/proc/net/tcp6`
- `kernel::net::ipv6::tcp_v6::diag::Tcp6Diag` — NETLINK_SOCK_DIAG

### Locking and concurrency

- (Inherits TCP locking from `net/ipv4/tcp.md`)
- **Per-tcp6_sock socket-lock**: shared with IPv4 path
- **Inet6 hashtables bucket spinlocks**: shared with IPv4

### Error handling

(Inherits TCP error model from `net/ipv4/tcp.md` — ECONNREFUSED, EAGAIN, EPIPE, ENOTCONN, EISCONN, etc.)

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Pseudo-header checksum bounds (saddr+daddr+upper-layer-len+nexthdr fits buffer) | `kani::proofs::net::ipv6::tcp_v6::pseudo_safety` |
| Dual-stack sk_prot switch (no use-after-free of v6 state) | `kani::proofs::net::ipv6::tcp_v6::dual_stack_safety` |
| Inet6 lookup bucket walk under bucket lock | `kani::proofs::net::ipv6::tcp_v6::lookup_safety` |
| TCP-MD5 v6 key-store insert/lookup | `kani::proofs::net::ipv6::tcp_v6::md5_safety` |

### Layer 2: TLA+ models

(none owned at this granularity; TCP state machine is modeled in `net/ipv4/tcp.md`)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-socket dual-stack state | sk_prot is either tcpv6_prot OR tcp_prot; never both observable concurrently | `kani::proofs::net::ipv6::tcp_v6::dual_stack_invariants` |
| TCP-MD5 v6 key store | every key has either a sockaddr_in or sockaddr_in6 family-tagged | `kani::proofs::net::ipv6::tcp_v6::md5_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/ipv6/00-overview.md`; cross-ref `net/ipv4/tcp.md` for TCP state-machine theorem)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | tcp6_sock + per-MD5/AO key refcounts use `Refcount` (saturating) | § Mandatory |
| **MEMORY_SANITIZE** | freed tcp6_sock + freed MD5/AO keys cleared (key material is sensitive) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above (per-tcp6_sock slab cache)
- **CONSTIFY**: `tcpv6_prot` proto-ops + `tcpv6_protocol` inet6_protosw `static const`
- **USERCOPY**: setsockopt buffer copy via `copy_from_user`
- **SIZE_OVERFLOW**: pseudo-header upper-layer-len arithmetic uses checked operators

### Row-2 / GR-RBAC integration

- LSM hook `security_socket_create` for AF_INET6 SOCK_STREAM
- LSM hook `security_inet_conn_request` (for SYN)
- LSM hook `security_inet_csk_clone` (for accept)
- Default GR-RBAC policy: empty so behavior matches upstream

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — TCP-over-IPv6 ABI is exhaustively specified by upstream + RFCs)

## Out of Scope

- TCP state machine, congestion control, TIME_WAIT handling, sendmsg/recvmsg buffer mgmt (cross-ref `net/ipv4/tcp.md`)
- 32-bit-only paths
- Implementation code
