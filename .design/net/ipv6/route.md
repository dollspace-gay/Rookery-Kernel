# Tier-3: net/ipv6/route — IPv6 routing (FIB6 trie + route lookup + RTM_*ROUTE)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/route.c
  - net/ipv6/ip6_fib.c
  - net/ipv6/fib6_rules.c
  - net/ipv6/fib6_notifier.c
  - include/net/ip6_route.h
  - include/net/ip6_fib.h
-->

## Summary
Tier-3 design for IPv6 routing: the FIB6 trie data structure (radix-trie of IPv6 prefixes), per-route nexthop + flags + metric storage, source-specific routing (RFC 6724) per-source-prefix path selection, the per-CPU `rt6_info` route-cache (PMTU + redirect), policy routing via fib6_rules (RFC 5340-style multi-table), the RTM_NEWROUTE/DELROUTE/GETROUTE/NEWNEXTHOP NETLINK_ROUTE wire interface, and `fib6_notifier` for in-kernel route-change subscribers (BGP daemons via NLA bus, BPF, etc.).

Sub-tier-3 of `net/ipv6/00-overview.md`. Anchors `RTM_*ROUTE` ABI for `ip -6 route`, `ip -6 rule`, `bird`, `frr`, and netd-style userspace.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Route lookup, redirect, PMTU, allocation, GC | `net/ipv6/route.c` |
| FIB6 trie + fib6_info insert/erase | `net/ipv6/ip6_fib.c` |
| Policy routing (multi-table) | `net/ipv6/fib6_rules.c` |
| In-kernel notifier | `net/ipv6/fib6_notifier.c` |
| Public API | `include/net/ip6_route.h`, `include/net/ip6_fib.h` |

## Compatibility contract

### `ip -6 route` (RTM_*ROUTE wire format)

| RTM | Operation |
|---|---|
| `RTM_NEWROUTE` | Add a route |
| `RTM_DELROUTE` | Delete a route |
| `RTM_GETROUTE` | Lookup or dump |
| `RTM_NEWNEXTHOP` / `DELNEXTHOP` / `GETNEXTHOP` | Standalone nexthop objects |

NETLINK_ROUTE messages with `rtm_family=AF_INET6` carry route descriptors via NLA attributes:
- `RTA_DST`, `RTA_SRC`, `RTA_IIF`, `RTA_OIF`, `RTA_GATEWAY`, `RTA_PRIORITY` (metric), `RTA_PREFSRC`, `RTA_METRICS` (nested PMTU/RTT/etc.), `RTA_MULTIPATH`, `RTA_FLOW`, `RTA_CACHEINFO`, `RTA_TABLE`, `RTA_MARK`, `RTA_MFC_STATS`, `RTA_VIA`, `RTA_NEWDST`, `RTA_PREF`, `RTA_ENCAP_TYPE`, `RTA_ENCAP`, `RTA_EXPIRES`, `RTA_PAD`, `RTA_UID`, `RTA_TTL_PROPAGATE`, `RTA_IP_PROTO`, `RTA_SPORT`, `RTA_DPORT`, `RTA_NH_ID`.

Wire format byte-identical so iproute2 + bird + frr work unchanged.

### `/proc/net/ipv6_route`

Format: per-route line with `dst prefixlen src srclen iif oif metric flags ref use rt6_table dst+srcdev`. Format-byte-identical so userspace tools work.

### `struct fib6_info` layout

`include/net/ip6_fib.h`: per-route descriptor. Fields `fib6_node`, `fib6_table`, `fib6_dst` (struct fib6_config dst+plen), `fib6_src`, `fib6_nh` (struct fib6_nh embedded or external for nexthop-objects), `fib6_flags`, `fib6_type`, `fib6_metrics`, `fib6_ref`, `fib6_destroying`, `rt6i_pmtu`, `rt6i_idev`. First-cache-line layout-equivalent.

### `struct rt6_info` layout (route cache)

`include/net/ip6_fib.h`: lookup-result wrapper for the rest of the kernel. Contains `dst` (struct dst_entry — shared with IPv4), `rt6i_idev`, `rt6i_gateway`, `rt6i_dst`, `rt6i_src`, `rt6i_flags`, `rt6i_pmtu`. Layout-equivalent for hot path.

### Policy routing (fib6_rules)

Up to `0xFFFFFFFF` route tables (default: `local=255`, `main=254`, `default=253`). `ip -6 rule add` adds rules with selector (saddr/daddr/iif/oif/fwmark/uid range/sport/dport range) → table. Identical algorithm + wire format.

### `fib6_notifier`

In-kernel atomic notifier chain receiving `FIB6_EVENT_ENTRY_REPLACE/ADD/DEL`. Subscribers: BPF route-control programs, BGP-listener LSM-aware modules, `fib_dump` rebuilders. Identical API.

### sysctls — `/proc/sys/net/ipv6/route/*`

`gc_thresh`, `gc_min_interval_ms`, `max_size`, `gc_timeout`, `gc_interval`, `gc_elasticity`, `mtu_expires`, `min_adv_mss`, `skip_notify_on_dev_down`. Format-byte-identical.

## Requirements

- REQ-1: FIB6 trie: per-table radix-trie keyed on (dst-prefix, src-prefix); insert/lookup/delete identical algorithm.
- REQ-2: Per-route fib6_info with full nexthop (or reference to standalone nexthop-object); first-cache-line layout-equivalent.
- REQ-3: Source-specific routing: when `RTA_SRC` is non-zero, lookup walks (dst-trie at dst-prefix → secondary src-trie at src-prefix) — match returns most-specific. Identical.
- REQ-4: rt6_info route-cache: per-flow PMTU + redirect cache; per-NUMA pcpu hash; insertion via `rt6_get_cookie_safe`; identical eviction policy.
- REQ-5: PMTU: ICMPv6 PTB → update per-flow rt6_info.rt6i_pmtu; expires after `mtu_expires` seconds. Identical.
- REQ-6: ICMPv6 redirect: redirect message updates per-flow rt6_info.rt6i_gateway; expires after redirect-lifetime. Identical.
- REQ-7: Policy routing fib6_rules: multi-table dispatch; rule selector (`{saddr, daddr, iif, oif, fwmark, uid, sport, dport, ipproto}`) matched in `rule->priority` order; first match wins. Identical.
- REQ-8: RTM_NEWROUTE/DELROUTE/GETROUTE/NEWNEXTHOP wire format byte-identical.
- REQ-9: ECMP / multipath: `RTA_MULTIPATH` with array of nexthops; per-flow hash selects nexthop. Identical algorithm (hash-based stable per-flow).
- REQ-10: `fib6_notifier` chain delivers FIB6_EVENT_ENTRY_REPLACE/ADD/DEL synchronously to subscribers under fib6_lock; identical.
- REQ-11: per-netns `gc_thresh` / `max_size` enforced; route-cache GC at insertion when `rt6_stats.fib_rt_cache > max_size`.
- REQ-12: `/proc/net/ipv6_route` format byte-identical.
- REQ-13: TLA+ model `models/net/ipv6_route_cache.tla` (mandatory per `net/ipv6/00-overview.md` Layer 2) — proves: route-cache producer/consumer race-free; lookup never returns a partially-constructed rt6_info; rt6_info refcount transitions are sound across PMTU updates.
- REQ-14: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `ip -6 route add 2001:db8::/64 dev eth0` then `ip -6 route show table all` byte-identical to upstream after equivalent operations. (covers REQ-1, REQ-2, REQ-8)
- [ ] AC-2: Source-specific route test: add `2001:db8::/64 from 2001:db8:1::/64 dev eth0`; lookup with src=`2001:db8:1::1` → matches; src=`2001:db8:2::1` → falls through to default. (covers REQ-3)
- [ ] AC-3: PMTU test: send a packet that triggers ICMPv6 PTB MTU=1280; subsequent lookups for the same flow return rt6i_pmtu=1280. (covers REQ-5)
- [ ] AC-4: Redirect test: send ICMPv6 redirect for `dst=2001:db8::1 gateway=fe80::2`; subsequent lookups return rt6i_gateway=fe80::2; after expiry, fall back. (covers REQ-6)
- [ ] AC-5: Policy routing: `ip -6 rule add fwmark 0x1 lookup 100`; with fwmark=0x1, lookup goes to table 100. (covers REQ-7)
- [ ] AC-6: ECMP: 2-nexthop multipath route; tcpdump shows 2-flow distribution across nexthops; same flow hashes consistently to same nexthop. (covers REQ-9)
- [ ] AC-7: BPF `fib6_notifier` listener gets FIB6_EVENT_ENTRY_ADD on `ip -6 route add`. (covers REQ-10)
- [ ] AC-8: Filling route cache to `max_size + 1` triggers GC; oldest entries evicted. (covers REQ-11)
- [ ] AC-9: `cat /proc/net/ipv6_route` byte-identical content. (covers REQ-12)
- [ ] AC-10: TLA+ `models/net/ipv6_route_cache.tla` proves the stated invariants. (covers REQ-13)
- [ ] AC-11: Hardening section present and follows template. (covers REQ-14)

## Architecture

### Rust module organization

- `kernel::net::ipv6::route::Fib6Trie` — radix-trie of IPv6 prefixes
- `kernel::net::ipv6::route::Fib6Info` — `struct fib6_info`
- `kernel::net::ipv6::route::Rt6Info` — lookup result + dst_entry holder
- `kernel::net::ipv6::route::lookup::Ip6Route` — main lookup entry-point
- `kernel::net::ipv6::route::cache::PmtuCache` — per-flow PMTU
- `kernel::net::ipv6::route::cache::RedirectCache` — per-flow redirect
- `kernel::net::ipv6::route::ecmp::Ecmp` — multipath nexthop selection
- `kernel::net::ipv6::route::rules::Fib6Rules` — policy routing
- `kernel::net::ipv6::route::notifier::Fib6Notifier` — atomic notifier chain
- `kernel::net::ipv6::route::netlink::RtmRouteHandler` — RTM_*ROUTE wire

### Locking and concurrency

- **Per-table `tb6_lock`** (rwlock): protects FIB6 trie mutations
- **Per-`rt6_info` refcount**: `Refcount` (saturating)
- **`fib6_walker_lock`** (per-table): walker iterator coordination
- **`rt6_exception_bucket_lock`** (per-bucket spinlock): PMTU/redirect cache
- **RCU**: lookup is RCU-read-side; write-side via `tb6_lock`

### Error handling

- `Err(EEXIST)` — RTM_NEWROUTE for duplicate
- `Err(ENOENT)` — RTM_DELROUTE for non-existent
- `Err(EINVAL)` — bad attributes
- `Err(ENETUNREACH)` — lookup miss returning -ENETUNREACH unicast route
- `Err(EHOSTUNREACH)` — neighbour fail

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| FIB6 trie insert/erase under tb6_lock | `kani::proofs::net::ipv6::route::trie_safety` |
| Source-specific lookup secondary trie walk | `kani::proofs::net::ipv6::route::ssrouting_safety` |
| PMTU/redirect bucket insert/lookup under bucket lock | `kani::proofs::net::ipv6::route::cache_safety` |
| ECMP hash-then-select arithmetic | `kani::proofs::net::ipv6::route::ecmp_safety` |
| RTM_NEWROUTE NLA parser | `kani::proofs::net::ipv6::route::netlink_safety` |

### Layer 2: TLA+ models

- `models/net/ipv6_route_cache.tla` (mandatory per `net/ipv6/00-overview.md`) — proves PMTU/redirect cache producer/consumer + rt6_info refcount soundness. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| FIB6 trie | for every node, all leaves under it have prefix matching node-prefix | `kani::proofs::net::ipv6::route::trie_invariants` |
| Per-bucket PMTU cache | bucket's flow-hash matches bucket-id | `kani::proofs::net::ipv6::route::pmtu_invariants` |
| ECMP nexthop array | per-flow hash deterministically maps to a single nexthop index in `[0, n)` | `kani::proofs::net::ipv6::route::ecmp_invariants` |
| Rule list | rules ordered strictly by `rule->priority` ascending | `kani::proofs::net::ipv6::route::rules_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Source-specific routing longest-match correctness theorem** via Verus — proves: for any (src, dst) pair, lookup returns the entry with the longest combined (src-prefix, dst-prefix) tuple ranked per RFC 6724 source-address selection.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | fib6_info + rt6_info refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-fib6_info + per-rt6_info slab caches | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: `fib6_*_ops` vtables `static const`
- **USERCOPY**: NLA parsing uses `nla_*` accessors that bound-check
- **SIZE_OVERFLOW**: prefix-length + nexthop-count arithmetic uses checked operators

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for RTM_*ROUTE writes; GR-RBAC policy can deny per-subject (default empty).
- Useful default policy: deny RTM_NEWROUTE outside gradm-marked `routing_admin` role (route table mutation is high-privilege).

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on every RTM_*ROUTE / RTM_*NEXTHOP netlink message; per-NLA accessor bound-checks before any deref.
- **PAX_KERNEXEC** — W^X on `fib6_*_ops`, route-encap dispatch vtables, and BPF programs attached via `fib6_notifier`.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every rtnetlink + sysctl-write entry to the IPv6 routing subsystem.
- **PAX_REFCOUNT** — saturating refcount on every `fib6_info`, `rt6_info`, per-bucket PMTU/redirect cache entry, ECMP nexthop array, and standalone nexthop-object.
- **PAX_MEMORY_SANITIZE** — zero-on-free for fib6_info + rt6_info slab objects (carry gateway addresses + metrics that may leak topology).
- **PAX_UDEREF / PAX_MEMORY_UDEREF** — netlink message parse via `nla_*` accessors; never directly deref user pointers from the rtnetlink path.
- **GRKERNSEC_HIDESYM** — hide kernel pointers in `/proc/net/ipv6_route`, `ip -6 route show`, RTM_GETROUTE dumps, and `fib6_notifier` BPF event records.
- **GRKERNSEC_BLACKHOLE** — IPv6 unreachable/prohibit route targets audited at rate-limit; silent black-hole mode suppresses ICMPv6 dest-unreachable on policy-routed drops.
- **GRKERNSEC_RANDNET** — ECMP per-flow hash seed seeded from gr-random (prevents nexthop-prediction attacks); per-flow PMTU cache-key seed randomized.
- **GRKERNSEC_NETFILTER** — `fib6_rules` selector mutation restricted to CAP_NET_ADMIN-in-init-userns; per-namespace rule mutation gated by GR-RBAC role.
- **GRKERNSEC_SOCK_PRIV** — RTM_NEWROUTE / RTM_NEWNEXTHOP audited (high-privilege; default-deny outside `routing_admin` role).
- **PAX_SIZE_OVERFLOW** — prefix-length (0..128) + nexthop-count + multipath-weight arithmetic uses checked operators; trie node-depth saturating.
- **CAP_NET_ADMIN** strict — every RTM_NEWROUTE / RTM_DELROUTE / RTM_NEWNEXTHOP / sysctl write to `/proc/sys/net/ipv6/route/*` requires CAP_NET_ADMIN in init_user_ns (not the calling userns — prevents userns-faked routing-table mutation).
- **CAP_NET_RAW** — `RTA_ENCAP`/`RTA_ENCAP_TYPE` (lightweight tunnels, SR-IPv6 encap) gated by CAP_NET_RAW + CAP_NET_ADMIN composite.

Per-doc rationale: IPv6 routing is the configuration anchor for the entire IPv6 packet path. PaX/Grsecurity reinforcement focuses on (a) saturating refs on the FIB6 trie + rt6_info route cache (PMTU/redirect entries can fan out per-flow under attack), (b) GRKERNSEC_RANDNET for ECMP per-flow hash to defeat nexthop-prediction tracking, (c) GRKERNSEC_HIDESYM on `/proc/net/ipv6_route` and rtnetlink dumps to prevent network-topology disclosure, and (d) enforcing CAP_NET_ADMIN in init_user_ns specifically for RTM_*ROUTE writes — route-table mutation from a child userns is a documented privilege-escalation surface that must be sealed.

## Open Questions

(none — IPv6 routing semantics are exhaustively specified by upstream + RFC 6724 + RFC 4191)

## Out of Scope

- IPv6 address autoconfiguration (cross-ref `net/ipv6/addrconf.md`)
- Neighbor Discovery (cross-ref `net/ipv6/ndisc.md`)
- IPv6 RX/TX path (cross-ref `net/ipv6/ip6-input.md`, `ip6-output.md`)
- 32-bit-only paths
- Implementation code
