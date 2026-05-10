# Tier-2: net/ipv6 — IPv6 stack

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/
  - include/net/ipv6.h
  - include/uapi/linux/ipv6.h
  - include/uapi/linux/in6.h
-->

## Summary
Tier-2 overview for the IPv6 stack: AF_INET6 socket family, IPv6 RX/TX path (`ip6_input.c` / `ip6_output.c`), IPv6 routing (FIB6), IPv6 address autoconfiguration (RFC 4862 SLAAC) + DHCPv6 client cooperation, Neighbor Discovery Protocol (RFC 4861), ICMPv6 (RFC 4443), multicast (MLDv1/v2 — RFC 3810), extension headers (Hop-by-Hop, Routing, Fragment, Destination Options), TCP-over-IPv6, UDP-over-IPv6, RAW-over-IPv6, IPSec/AH/ESP for IPv6, GRE/VTI/IPv6-tunnel encapsulators, IPv6 multicast routing (PIM), and the IPv6 sysctl surface.

Mirrors `net/ipv4/00-overview.md` structure; differences focus on IPv6-specific machinery (extension headers, ND, SLAAC, flow labels, anycast, scoped addresses).

Sub-tier-2 of `net/00-overview.md`.

## Scope

This Tier-2 governs **all** of `/home/doll/linux-src/net/ipv6/` (~50 source files) plus the public API + UAPI headers. Per-file Tier-3 docs live under `.design/net/ipv6/`.

## Compatibility contract — outline

### AF_INET6 socket family

`socket(AF_INET6, SOCK_{STREAM,DGRAM,RAW,SEQPACKET}, proto)` — same set of socket types as IPv4 plus `IPPROTO_ICMPV6`. Identical to upstream.

### `struct sockaddr_in6` layout

```c
struct sockaddr_in6 {
    sa_family_t     sin6_family;       /* AF_INET6 */
    __be16          sin6_port;
    __be32          sin6_flowinfo;     /* flow label + traffic class */
    struct in6_addr sin6_addr;         /* 16 bytes */
    __u32           sin6_scope_id;     /* link-local scope */
};
```

Layout-identical.

### Dual-stack (IPv4-mapped IPv6 addresses)

`::ffff:0:0/96` IPv4-mapped addresses on AF_INET6 sockets dispatch to the IPv4 path transparently unless `IPV6_V6ONLY` is set. Identical semantics so /etc/gai.conf-driven sysd applications work.

### sysctls — `/proc/sys/net/ipv6/`

Per-netns + per-device knobs: `accept_ra`, `autoconf`, `forwarding`, `disable_ipv6`, `temp_prefered_lft`, `temp_valid_lft`, `mtu`, `optimistic_dad`, `accept_redirects`, `flush`, `route/*`, `neigh/*`. Format byte-identical so distro `sysctl.d` rules work unchanged.

### `/proc/net/ipv6_route`, `/proc/net/if_inet6`, `/proc/net/snmp6`

Format-identical so `ip -6 route`, `ifconfig`, snmpd work unchanged.

### NETLINK_ROUTE for IPv6: `RTM_NEWADDR/DELADDR/NEWROUTE/DELROUTE/GETNEIGH/...`

Wire format byte-identical so iproute2 `ip -6` works unchanged.

## Tier-3 docs governed by this Tier-2

(Phase C will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `net/ipv6/af-inet6.md` | AF_INET6 socket family + dual-stack |
| `net/ipv6/ip6-input.md` | IPv6 RX path |
| `net/ipv6/ip6-output.md` | IPv6 TX path + frag |
| `net/ipv6/route.md` | IPv6 route lookup + FIB6 trie |
| `net/ipv6/addrconf.md` | SLAAC + DHCPv6 cooperation |
| `net/ipv6/ndisc.md` | Neighbor Discovery (NS/NA/RS/RA/Redirect) |
| `net/ipv6/icmpv6.md` | ICMPv6 |
| `net/ipv6/mcast.md` | MLDv1/v2 |
| `net/ipv6/exthdrs.md` | Extension headers (HBH, Routing, Frag, DestOpt, AH, ESP) |
| `net/ipv6/tcp-v6.md` | TCP-over-IPv6 (mostly inherits TCP from `net/ipv4/tcp.md`; covers v6-specific socket-level differences) |
| `net/ipv6/udp-v6.md` | UDP-over-IPv6 (similar inheritance pattern) |
| `net/ipv6/raw-v6.md` | RAW IPv6 sockets |
| `net/ipv6/flowlabel.md` | IPv6 flow label management |
| `net/ipv6/ip6mr.md` | IPv6 multicast routing |
| `net/ipv6/ipv6-sockglue.md` | per-socket setsockopt SOL_IPV6 + IPV6_RECVPKTINFO etc. |

## Compatibility outline (top-level)

- REQ-O1: AF_INET6 socket family + struct sockaddr_in6 layout-identical.
- REQ-O2: Dual-stack via IPv4-mapped addresses (`::ffff:0:0/96`); `IPV6_V6ONLY` setsockopt disables.
- REQ-O3: Per-Tier-3 child documents define per-component requirements — total IPv6 ABI fidelity covered cumulatively.
- REQ-O4: Per-netns `/proc/sys/net/ipv6/*` sysctls + per-netns `/proc/net/if_inet6`/`ipv6_route`/`snmp6` format-identical.
- REQ-O5: NETLINK_ROUTE wire-format identical for IPv6 RTM messages.
- REQ-O6: IPv6 extension-header parser bounded + DoS-hardened (defense vs. CVE-2017-7542-class extension-header-explosion attacks); cross-ref `net/ipv6/exthdrs.md`.
- REQ-O7: ND cache (`neigh_table` for IPv6) and FIB6 trie share core abstractions with IPv4 `neighbour` + `fib_trie`; cross-ref `net/core/00-overview.md`.
- REQ-O8: IPv6 RX/TX path Tier-3 docs declare TLA+ models for any concurrency-sensitive surface (route-cache, ND backlog, fragment reassembly).

## Acceptance Criteria (top-level)

- [ ] AC-O1: `pahole struct sockaddr_in6` byte-identical. (covers REQ-O1)
- [ ] AC-O2: `nc -6 -l ::1 8080` + `nc -6 ::1 8080` exchange works; same with IPv4-mapped (`nc -6 ::ffff:127.0.0.1 8080` in non-v6only mode). (covers REQ-O2)
- [ ] AC-O3: Per-Tier-3 ACs cumulatively cover the IPv6 ABI surface. (meta-AC)
- [ ] AC-O4: `sysctl -a -p` regex `^net.ipv6\.` byte-identical content list. (covers REQ-O4)
- [ ] AC-O5: `ip -6 route show table all` byte-identical. (covers REQ-O5)
- [ ] AC-O6: An IPv6 packet with 64 nested HBH options + 32 nested Routing options is rejected with documented error code; doesn't trigger CPU exhaustion. (covers REQ-O6)
- [ ] AC-O7: ND probe + reachability transition matches RFC 4861 state machine; cross-ref Tier-3 doc tests. (covers REQ-O7)

## Architecture (top-level)

Each Tier-3 child doc declares its own Rust module organization. The shared abstractions:

- `kernel::net::ipv6::AfInet6` — socket family root
- `kernel::net::ipv6::Inet6Sock` — `struct ipv6_pinfo` (per-AF_INET6-socket state)
- `kernel::net::ipv6::route::Fib6` — FIB6 trie
- `kernel::net::ipv6::addrconf::Addrconf` — SLAAC state machine
- `kernel::net::ipv6::ndisc::Ndisc` — ND state machine
- `kernel::net::ipv6::exthdrs::ExthdrParser` — bounded extension-header walker
- `kernel::net::ipv6::flowlabel::FlowLabelMgr`

## Verification (top-level)

### Layer 1: Kani SAFETY proofs

Each Tier-3 child doc declares its own.

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/net/ipv6_route_cache.tla` | `net/ipv6/route.md` (parallels IPv4 route-cache) |
| `models/net/ipv6_ndisc.tla` | `net/ipv6/ndisc.md` (RFC 4861 ND state machine) |
| `models/net/ipv6_addrconf.tla` | `net/ipv6/addrconf.md` (RFC 4862 SLAAC; DAD; tentative→preferred→deprecated lifetimes) |
| `models/net/ipv6_frag_reasm.tla` | `net/ipv6/exthdrs.md` (fragment reassembly safety) |

### Layer 3: invariant harnesses

Per Tier-3.

### Layer 4: functional correctness (opt-in)

- **Extension-header bounded-walk theorem** via Verus — proves: extension-header parser terminates within ≤ MAX_EXTHDRS_PARSED iterations even for adversarial chains; rejects loops.
- **DAD non-collision theorem** via TLA+ refinement — proves: no two interfaces in the same link adopt the same address (modulo DAD-disabled override).

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | `inet6_dev`, `fib6_info`, `rt6_info`, `inet6_ifaddr`, neigh entries use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-type slab caches | § Mandatory |
| **MEMORY_SANITIZE** | freed sockets + neighbour cache entries cleared (sensitive: address state) | § Default-on configurable off |

### Row-1 features consumed by this Tier-2

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: per-protocol ops vtables (`ipv6_protocol`, `inet6_protosw`, `proto_ops`) `static const`
- **USERCOPY**: send/recvmsg payload movement uses `copy_to_iter`/`copy_from_iter`
- **SIZE_OVERFLOW**: extension-header length arithmetic uses checked operators (defense vs. CVE-2017-7542-class)
- **KERNEXEC**: parser-table dispatch via `static const fn-ptr` arrays only

### Row-2 / GR-RBAC integration

- LSM hooks: `security_socket_create` for AF_INET6 (already standard), `security_path_*` for /proc/net/ipv6 visibility
- Default useful policy: empty so behavior matches upstream

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none at Tier-2 — defer to per-Tier-3)

## Out of Scope

- IPv4-only paths (cross-ref `net/ipv4/00-overview.md`)
- 32-bit-only paths
- Implementation code
