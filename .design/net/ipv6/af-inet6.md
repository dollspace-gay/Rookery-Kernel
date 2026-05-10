# Tier-3: net/ipv6/af-inet6 — AF_INET6 socket family + dual-stack

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/af_inet6.c
  - net/ipv6/ipv6_sockglue.c
  - net/ipv6/inet6_connection_sock.c
  - net/ipv6/inet6_hashtables.c
  - include/net/ipv6.h
  - include/uapi/linux/in6.h
-->

## Summary
Tier-3 design for the AF_INET6 socket family: registration of the address family, per-IPPROTO socket-type dispatch (TCPv6/UDPv6/RAWv6/ICMPv6/SCTPv6), `struct ipv6_pinfo` (per-AF_INET6-socket state), the dual-stack mechanism (IPv4-mapped IPv6 addresses on AF_INET6 sockets), per-socket setsockopt SOL_IPV6 surface, the IPv6 ehash/bhash hashtables for socket lookup, and the inet6_connection_sock helpers shared between connection-oriented IPv6 protocols.

Sub-tier-3 of `net/ipv6/00-overview.md`. The "front door" of the IPv6 stack. Pairs with `net/ipv4/inet.md`-style structure on the IPv4 side.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| AF_INET6 family register, inet6_create, per-IPPROTO dispatch | `net/ipv6/af_inet6.c` |
| SOL_IPV6 setsockopt/getsockopt + ipv6_pinfo init | `net/ipv6/ipv6_sockglue.c` |
| Connection-oriented helpers (accept, sk lookup) | `net/ipv6/inet6_connection_sock.c` |
| ehash + bhash for IPv6 socket lookup | `net/ipv6/inet6_hashtables.c` |
| Public API | `include/net/ipv6.h` |
| UAPI | `include/uapi/linux/in6.h` (struct sockaddr_in6, struct in6_addr, IN6_IS_ADDR_*, IPV6_*) |

## Compatibility contract

### `socket(AF_INET6, type, protocol)`

`type` ∈ `{SOCK_STREAM, SOCK_DGRAM, SOCK_RAW, SOCK_SEQPACKET}`. `protocol`:
- `IPPROTO_TCP` (with SOCK_STREAM)
- `IPPROTO_UDP` / `IPPROTO_UDPLITE` (with SOCK_DGRAM)
- `IPPROTO_ICMPV6` (with SOCK_DGRAM, "ICMPv6 echo socket")
- `IPPROTO_RAW` / per-proto for SOCK_RAW

Identical dispatch semantics; SOCK_RAW gated by CAP_NET_RAW.

### `struct sockaddr_in6` layout

```c
struct sockaddr_in6 {
    sa_family_t     sin6_family;       /* AF_INET6 */
    __be16          sin6_port;
    __be32          sin6_flowinfo;     /* upper 4 bits TC, lower 20 bits flow label */
    struct in6_addr sin6_addr;         /* 16 bytes */
    __u32           sin6_scope_id;     /* link-local scope */
};
```

Layout-byte-identical so existing /proc/-parsing tools and tcpdump headers see identical wire format.

### `struct in6_addr`

```c
struct in6_addr {
    union {
        __u8   u6_addr8[16];
        __be16 u6_addr16[8];
        __be32 u6_addr32[4];
    } in6_u;
};
```

Layout-byte-identical.

### Dual-stack: IPv4-mapped IPv6 addresses

`::ffff:0:0/96` (IPv4-mapped) addresses on AF_INET6 sockets transparently dispatch to the IPv4 path. Default: enabled. Disabled via `setsockopt(IPPROTO_IPV6, IPV6_V6ONLY, &one)`. Sysctl `/proc/sys/net/ipv6/bindv6only` sets per-netns default.

Identical semantics so existing dual-stack listeners (Apache `Listen *:80` resolves to AF_INET6 + IPV6_V6ONLY=0 by default in modern distros; nginx `listen [::]:80` similar) work unchanged.

### SOL_IPV6 setsockopt surface

| Option | Semantics |
|---|---|
| `IPV6_V6ONLY` | disable dual-stack on this socket |
| `IPV6_RECVPKTINFO` / `IPV6_PKTINFO` | per-recvmsg cmsg with src/dst + ifindex |
| `IPV6_RECVHOPLIMIT` / `IPV6_HOPLIMIT` | per-recvmsg cmsg with hoplimit |
| `IPV6_RECVTCLASS` / `IPV6_TCLASS` | per-recvmsg cmsg with traffic class |
| `IPV6_RECVRTHDR` / `IPV6_RTHDR` | per-recvmsg cmsg with routing header |
| `IPV6_RECVHOPOPTS` / `IPV6_HOPOPTS` | per-recvmsg cmsg with hop-by-hop options |
| `IPV6_RECVDSTOPTS` / `IPV6_DSTOPTS` | per-recvmsg cmsg with destination options |
| `IPV6_UNICAST_HOPS` | per-socket hoplimit |
| `IPV6_MULTICAST_HOPS` | per-socket multicast hoplimit |
| `IPV6_MULTICAST_IF` | outgoing multicast interface |
| `IPV6_MULTICAST_LOOP` | multicast loopback |
| `IPV6_JOIN_GROUP` / `IPV6_LEAVE_GROUP` | MLD group membership |
| `IPV6_MTU_DISCOVER` / `IPV6_MTU` | path MTU discovery |
| `IPV6_FLOWLABEL_MGR` / `IPV6_FLOWINFO_SEND` | flow-label management (cross-ref `net/ipv6/flowlabel.md`) |
| `IPV6_TRANSPARENT` | bind to non-local (TPROXY) |
| `IPV6_FREEBIND` | bind to non-existent local addr |
| `IPV6_AUTOFLOWLABEL` | kernel-generated flow labels |
| `IPV6_ADDR_PREFERENCES` | RFC 5014 source-address selection prefs |
| `IPV6_MINHOPCOUNT` | TCP-AO/MD5 GTSM-style minimum-hop-count check |

Each option's value byte-identical so existing setsockopt-using code works.

### `struct ipv6_pinfo` layout

`include/net/ipv6.h`: per-AF_INET6-socket state, embedded in `struct tcp6_sock`/`udp6_sock`/etc. Contains `saddr`, `daddr_cache`, `mcast_oif`, `ucast_oif`, `flow_label`, `hop_limit`, `mcast_hops`, `min_hopcount`, `flow_label_lock`, `pktoptions`, `recverr`, `mc_loop`, `recvtclass`, `tclass`, `srcprefs`, `cgroup_bpf`, etc.

Layout-equivalent for the first cache-line (offsets used by hot paths) so `bpftrace` / kgdb introspection works unchanged.

### IPv6 ehash + bhash

Per-netns `inet_hashinfo`-style hashtables shared with IPv4 (the lookup keys are ip-version-tagged), accessed via `inet6_lookup_listener`, `__inet6_lookup_established`, `inet6_bind_bucket_create`. Algorithm + sizing identical so `ss -6 -t` listing performance matches.

## Requirements

- REQ-1: AF_INET6 family registered with `sock_register(&inet6_family_ops)`.
- REQ-2: `inet6_create` dispatch table maps (type, IPPROTO) to per-protocol `inet6_protosw`; identical entries to upstream.
- REQ-3: `struct sockaddr_in6` byte-identical layout (sin6_family, sin6_port, sin6_flowinfo, sin6_addr, sin6_scope_id).
- REQ-4: `struct in6_addr` byte-identical layout.
- REQ-5: Dual-stack: an AF_INET6 socket without `IPV6_V6ONLY` accepts/sends IPv4-mapped IPv6 addresses (`::ffff:a.b.c.d`); kernel transparently dispatches via IPv4 path.
- REQ-6: `IPV6_V6ONLY` setsockopt: per-socket; default value taken from per-netns `bindv6only` sysctl (default `0`).
- REQ-7: SOL_IPV6 setsockopt/getsockopt: each option in the table above implemented identically.
- REQ-8: `struct ipv6_pinfo` first-cache-line layout-equivalent.
- REQ-9: ehash + bhash for IPv6: shared infrastructure with IPv4; lookups via `inet6_lookup_*` family identical.
- REQ-10: Per-recvmsg cmsg generation: when corresponding RECV* option is set, recvmsg attaches the appropriate cmsg(s) — identical layout + ordering as upstream.
- REQ-11: SOCK_RAW gated by CAP_NET_RAW (or capable in netns); identical.
- REQ-12: ICMPv6 SOCK_DGRAM "ping socket": gated by `/proc/sys/net/ipv4/ping_group_range` (yes, IPv4 sysctl shared); identical.
- REQ-13: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct sockaddr_in6` byte-identical. (covers REQ-3)
- [ ] AC-2: `pahole struct in6_addr` byte-identical. (covers REQ-4)
- [ ] AC-3: `pahole struct ipv6_pinfo` byte-identical first cache-line. (covers REQ-8)
- [ ] AC-4: `socket(AF_INET6, SOCK_STREAM, 0)` returns an FD; `bind` to `::1:8080` succeeds; `listen` + `accept` work. (covers REQ-1, REQ-2)
- [ ] AC-5: Dual-stack: AF_INET6 socket without IPV6_V6ONLY bound to `:::80` accepts IPv4 connections to `0.0.0.0:80` (verifiable via `tcpdump -i lo`). (covers REQ-5, REQ-6)
- [ ] AC-6: With IPV6_V6ONLY=1, an AF_INET socket can bind `0.0.0.0:80` simultaneously without EADDRINUSE. (covers REQ-6)
- [ ] AC-7: For each SOL_IPV6 setsockopt/getsockopt option in the table, a r/w round-trip returns the written value (or for boolean/cmsg-enable, the post-write get reads as set). (covers REQ-7)
- [ ] AC-8: With `IPV6_RECVPKTINFO=1`, recvmsg returns a cmsg of type `IPV6_PKTINFO` carrying the receiving in6_pktinfo (ifindex + dst). (covers REQ-10)
- [ ] AC-9: Without CAP_NET_RAW, `socket(AF_INET6, SOCK_RAW, IPPROTO_RAW)` returns EPERM. (covers REQ-11)
- [ ] AC-10: ICMPv6 ping socket with no `ping_group_range` privilege returns EPERM; with privilege returns success. (covers REQ-12)
- [ ] AC-11: `ss -6 -t` byte-identical to upstream after equivalent boot. (covers REQ-9)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

## Architecture

### Rust module organization

- `kernel::net::ipv6::af::AfInet6` — family register + `inet6_create`
- `kernel::net::ipv6::sock::Inet6Sock` — `struct ipv6_pinfo` wrapper
- `kernel::net::ipv6::sockglue::SolIpv6` — SOL_IPV6 setsockopt/getsockopt dispatch
- `kernel::net::ipv6::sockglue::DualStack` — IPv4-mapped redirection
- `kernel::net::ipv6::hashtables::Inet6Hashtables` — ehash/bhash IPv6 lookup
- `kernel::net::ipv6::conn_sock::Inet6ConnSock` — connection-oriented helpers (accept, listen)
- `kernel::net::ipv6::cmsg::Ipv6Cmsg` — per-recvmsg cmsg builder

### Locking and concurrency

- **`inet6_protosw_lock`** (rwlock): per-system protosw registry
- **Per-`ipv6_pinfo` `lock`** (spinlock): protects ip6 socket-level state
- **ehash/bhash bucket spinlocks**: per-bucket
- **Per-netns `inet6_addr_lst_lock`**: bind-conflict resolution

### Error handling

- `Err(EAFNOSUPPORT)` — AF_INET6 disabled (CONFIG)
- `Err(EPROTONOSUPPORT)` — bad (type, IPPROTO) combination
- `Err(EPERM)` — SOCK_RAW or ping-group violation
- `Err(EADDRINUSE)` — bind conflict
- `Err(EINVAL)` — bad sockaddr_in6 / bad setsockopt value
- `Err(ENETUNREACH)` — connect to address with no route

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| `inet6_create` dispatch table walk | `kani::proofs::net::ipv6::af_create_safety` |
| `setsockopt` SOL_IPV6 option-table walk | `kani::proofs::net::ipv6::sockglue_safety` |
| Dual-stack IPv4-mapped redirection (no use-after-free of v6 sk) | `kani::proofs::net::ipv6::dual_stack_safety` |
| ehash/bhash bucket insert/erase | `kani::proofs::net::ipv6::hashtables_safety` |
| Per-recvmsg cmsg builder | `kani::proofs::net::ipv6::cmsg_safety` |

### Layer 2: TLA+ models

(none owned here; cross-ref `net/ipv6/00-overview.md` Layer 2 list — none of those models is owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| ehash bucket per IPv6 socket | each socket appears in exactly one bucket determined by its `inet_ehashfn` hash | `kani::proofs::net::ipv6::ehash_invariants` |
| bhash bucket per bound IPv6 socket | each bound socket appears in exactly one bucket determined by `inet_bhashfn(port, netns)` | `kani::proofs::net::ipv6::bhash_invariants` |

### Layer 4: Functional correctness (opt-in)

(deferred — same gate as `net/ipv6/00-overview.md`)

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-`ipv6_pinfo` + per-bind-bucket refcounts use `Refcount` (saturating) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: per-AF_INET6-socket slab caches (often type-shared with IPv4 via `tcp6_sock`/`udp6_sock` per-IPPROTO caches)
- **CONSTIFY**: `inet6_family_ops`, per-protocol `inet6_protosw` arrays, SOL_IPV6 option table `static const`
- **USERCOPY**: setsockopt/getsockopt buffer movement uses `copy_{from,to}_user` exclusively
- **SIZE_OVERFLOW**: cmsg-length arithmetic uses checked operators
- **KERNEXEC**: per-protocol dispatch via `static const fn-ptr` arrays only

### Row-2 / GR-RBAC integration

- LSM hook `security_socket_create` for AF_INET6 (already standard)
- LSM hook `security_socket_bind` (with sin6_addr + scope_id check)
- Default GR-RBAC policy: empty so behavior matches upstream

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — AF_INET6 ABI is exhaustively specified by upstream + RFCs)

## Out of Scope

- Per-IPPROTO Tier-3 docs (`net/ipv6/tcp-v6.md`, `udp-v6.md`, `raw-v6.md`, `icmpv6.md`)
- IPv6 routing (cross-ref `net/ipv6/route.md`)
- IPv6 RX/TX path (cross-ref `net/ipv6/ip6-input.md`, `ip6-output.md`)
- 32-bit-only paths
- Implementation code
