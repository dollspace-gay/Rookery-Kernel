# Tier-3: net/ipv6/ndisc — Neighbor Discovery Protocol (RFC 4861)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/ipv6/ndisc.c
  - net/core/neighbour.c
  - include/net/ndisc.h
  - include/net/neighbour.h
  - include/uapi/linux/neighbour.h
  - include/uapi/linux/icmpv6.h
-->

## Summary
Tier-3 design for IPv6 Neighbor Discovery (RFC 4861) — the layer between IPv6 and the link-layer that discovers next-hop link-layer addresses, detects unreachability, and absorbs Router Advertisements. Implements all five ND message types: Neighbor Solicitation (NS), Neighbor Advertisement (NA), Router Solicitation (RS), Router Advertisement (RA), and ICMPv6 Redirect.

The ND state machine: per-neighbour `INCOMPLETE` → `REACHABLE` → `STALE` → `DELAY` → `PROBE` → (`REACHABLE` | `FAILED`). Implemented atop the generic `neighbour` framework in `net/core/neighbour.c` (shared with IPv4 ARP).

Sub-tier-3 of `net/ipv6/00-overview.md`. Cooperates with `net/ipv6/addrconf.md` (delivers RA messages → SLAAC), `net/ipv6/route.md` (RA-derived routes), `net/ipv6/icmpv6.md` (the carrier protocol). Anchors any IPv6 next-hop lookup.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| ND core: NS/NA/RS/RA/Redirect send + recv, RA RDNSS extraction | `net/ipv6/ndisc.c` |
| Generic neighbour framework | `net/core/neighbour.c` |
| Public API | `include/net/ndisc.h`, `include/net/neighbour.h` |
| UAPI | `include/uapi/linux/neighbour.h`, `include/uapi/linux/icmpv6.h` |

## Compatibility contract

### `ip -6 neigh` (NETLINK_ROUTE RTM_NEWNEIGH/DELNEIGH/GETNEIGH)

NLA attributes: `NDA_DST`, `NDA_LLADDR`, `NDA_CACHEINFO`, `NDA_PROBES`, `NDA_VLAN`, `NDA_PORT`, `NDA_VNI`, `NDA_IFINDEX`, `NDA_MASTER`, `NDA_LINK_NETNSID`, `NDA_SRC_VNI`, `NDA_PROTOCOL`, `NDA_NH_ID`, `NDA_FDB_EXT_ATTRS`, `NDA_FLAGS_EXT`, `NDA_NDM_STATE_MASK`, `NDA_NDM_FLAGS_MASK`. Wire format byte-identical so iproute2 + neighd-style userspace work unchanged.

### `/proc/net/ipv6_neigh` content (when CONFIG_PROC_FS)

Format: `addr ifindex prefixlen state ifname hwaddr`. Format-byte-identical.

### sysctls — `/proc/sys/net/ipv6/neigh/{default,<iface>}/`

Per-iface knobs: `gc_thresh1/2/3`, `gc_interval`, `gc_stale_time`, `mcast_solicit`, `ucast_solicit`, `app_solicit`, `mcast_resolicit`, `retrans_time_ms`, `base_reachable_time_ms`, `delay_first_probe_time`, `unres_qlen`, `unres_qlen_bytes`, `proxy_qlen`, `anycast_delay`, `proxy_delay`. Default values + format byte-identical.

### ND message wire format

ICMPv6 type/code per RFC 4861:

| Message | ICMPv6 Type |
|---|---|
| Router Solicitation (RS) | 133 |
| Router Advertisement (RA) | 134 |
| Neighbor Solicitation (NS) | 135 |
| Neighbor Advertisement (NA) | 136 |
| Redirect | 137 |

Each message carries TLV options: Source Link-Layer Address (1), Target Link-Layer Address (2), Prefix Information (3), Redirected Header (4), MTU (5), Route Information (24), RDNSS (25), DNSSL (31). All option types parsed identically.

### `struct neighbour` layout

`include/net/neighbour.h`: per-neighbour cache entry (shared IPv4/IPv6). Fields `next`, `tbl`, `parms`, `confirmed`, `updated`, `lock`, `refcnt`, `arp_queue` (skb backlog), `arp_queue_len_bytes`, `timer`, `used`, `probes`, `flags`, `nud_state`, `type`, `dead`, `ha_lock`, `ha`, `hh`, `output`, `ops`, `dev`, `primary_key`. Layout-equivalent for first cache-line.

### `nud_state` values

`NUD_INCOMPLETE | NUD_REACHABLE | NUD_STALE | NUD_DELAY | NUD_PROBE | NUD_FAILED | NUD_NOARP | NUD_PERMANENT`. Identical bit values.

### Default ND parameters

| Parameter | Default | Per RFC 4861 |
|---|---|---|
| `mcast_solicit` (multicast NS retries) | 3 | MAX_MULTICAST_SOLICIT |
| `ucast_solicit` (unicast NS retries) | 3 | MAX_UNICAST_SOLICIT |
| `retrans_time_ms` | 1000 | RetransTimer |
| `base_reachable_time_ms` | 30000 | ReachableTime |
| `gc_stale_time` | 60s | GC_STALETIME |

Defaults byte-identical so existing user knobs work.

## Requirements

- REQ-1: All five ND message types (RS/RA/NS/NA/Redirect) parsed + emitted per RFC 4861 wire format.
- REQ-2: ND state machine: per-neighbour transitions `INCOMPLETE` → `REACHABLE` → `STALE` → `DELAY` → `PROBE` → (`REACHABLE` | `FAILED`); identical transitions + timers.
- REQ-3: Multicast NS retries: `mcast_solicit` count; on no NA → state = FAILED.
- REQ-4: Unicast NS retries (PROBE state): `ucast_solicit` count; on no NA → state = FAILED.
- REQ-5: ND backlog (`arp_queue`): per-neighbour skb queue while INCOMPLETE; bounded by `unres_qlen_bytes`; on resolution → drained.
- REQ-6: ND DAD support (NS sent from `::` with target = candidate-address); cooperates with addrconf via `ndisc_send_ns` + `ndisc_send_na`.
- REQ-7: RA consumer: on RA receipt, parse PIO + RIO + RDNSS + DNSSL options; deliver PIO to addrconf (`addrconf_prefix_rcv`), RIO to route (`rt6_route_rcv`), RDNSS to user (via netlink RTM_NEWNDUSEROPT or rdisc6).
- REQ-8: Redirect: on Redirect receipt, validate (per RFC 4861 § 8.1: source must be link-local from default router; target is on-link); update route cache via `rt6_redirect`.
- REQ-9: Proxy ND (`proxy_arp` in v6): per-neighbour PROXY entry; respond to NS for proxied addresses via `pndisc_redo`.
- REQ-10: Per-iface sysctls work identically (gc_thresh1/2/3, retrans_time_ms, base_reachable_time_ms, etc.).
- REQ-11: Generic neighbour framework: `neigh_alloc`, `neigh_lookup`, `neigh_create`, `neigh_event_send`, `neigh_destroy` semantics shared with IPv4 ARP.
- REQ-12: NETLINK_ROUTE RTM_NEWNEIGH/DELNEIGH/GETNEIGH wire format byte-identical.
- REQ-13: TLA+ model `models/net/ipv6_ndisc.tla` (mandatory per `net/ipv6/00-overview.md` Layer 2) — proves: ND state machine has no stuck states; arrival of NA in any state correctly updates lladdr + state per RFC 4861 § 7.2.5.
- REQ-14: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct neighbour` byte-identical first cache-line. (covers REQ-2)
- [ ] AC-2: ND wire format test: capture an NS/NA/RS/RA/Redirect message via `ndisc-test-tool`; bytes byte-identical to upstream's. (covers REQ-1)
- [ ] AC-3: A first-time ping6 to a neighbour: state INCOMPLETE → multicast NS sent → NA received → state REACHABLE; backlog drained. (covers REQ-2, REQ-3, REQ-5)
- [ ] AC-4: After `base_reachable_time_ms`, neighbour transitions to STALE without traffic; on next packet, transitions to DELAY → PROBE → (REACHABLE on NA | FAILED on no NA). (covers REQ-2, REQ-4)
- [ ] AC-5: DAD test: NS from `::` with target = candidate-address; receives NA from another node → addrconf marks IFA_F_DADFAILED. (covers REQ-6)
- [ ] AC-6: RA test: synthetic RA with PIO `2001:db8::/64 a=1 prefered_lt=86400` → addrconf SLAAC fires, address assigned. (covers REQ-7)
- [ ] AC-7: ICMPv6 Redirect test: Redirect from default-router with target=fe80::2 for dst=2001:db8::1 → route cache updated; subsequent traffic uses fe80::2. (covers REQ-8)
- [ ] AC-8: Proxy ND test: `ip -6 neigh add proxy 2001:db8::1 dev eth0` → NS for `2001:db8::1` from any source on eth0 receives NA from kernel. (covers REQ-9)
- [ ] AC-9: For each per-iface sysctl in the table, a r/w round-trip returns the written value; behavior changes accordingly. (covers REQ-10)
- [ ] AC-10: `ip -6 neigh show` byte-identical content. (covers REQ-12)
- [ ] AC-11: TLA+ `models/net/ipv6_ndisc.tla` proves: state machine has no stuck states; NA → lladdr update is sound for all source-state combinations per RFC 4861 § 7.2.5. (covers REQ-13)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-14)

## Architecture

### Rust module organization

- `kernel::net::ipv6::ndisc::Ndisc` — top-level entrypoint
- `kernel::net::ipv6::ndisc::ns::Ns` — Neighbor Solicitation send/recv
- `kernel::net::ipv6::ndisc::na::Na` — Neighbor Advertisement send/recv
- `kernel::net::ipv6::ndisc::rs::Rs` — Router Solicitation send
- `kernel::net::ipv6::ndisc::ra::Ra` — Router Advertisement consumer
- `kernel::net::ipv6::ndisc::redirect::Redirect` — ICMPv6 Redirect handler
- `kernel::net::ipv6::ndisc::options::Tlv` — RFC 4861 TLV parser/builder
- `kernel::net::ipv6::ndisc::proxy::Proxy` — proxy ND
- `kernel::net::core::neighbour::Neighbour` — `struct neighbour` (shared with IPv4)
- `kernel::net::core::neighbour::NeighTable` — per-family neigh table
- `kernel::net::core::neighbour::Statemachine` — INCOMPLETE→REACHABLE→… transitions
- `kernel::net::core::neighbour::Backlog` — per-neighbour arp_queue

### Locking and concurrency

- **`tbl->lock`** (rwlock per `neigh_table`): protects per-table mutations
- **Per-`neighbour` `lock`** (spinlock): per-entry state transitions + lladdr
- **`tbl->gc_list_lock`** (spinlock): GC list mutator
- **`pndisc->lock`** (per-iface): proxy ND list

### Error handling

- `Err(ENOENT)` — RTM_DELNEIGH for non-existent
- `Err(EEXIST)` — RTM_NEWNEIGH for existing without REPLACE
- `Err(ENOMEM)` — neigh_alloc failed
- `Err(EINVAL)` — bad NLA / bad address
- Async failures: queued skbs dropped on FAILED state with `EHOSTUNREACH` to socket layer

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| ND TLV option parser (bounded walk; no infinite loop on malformed length) | `kani::proofs::net::ipv6::ndisc::tlv_parse_safety` |
| `neigh_lookup` hash bucket walk under tbl->lock | `kani::proofs::net::ipv6::ndisc::lookup_safety` |
| Per-neighbour state-transition (`neigh_event_send`) | `kani::proofs::net::ipv6::ndisc::transition_safety` |
| arp_queue drain on resolution | `kani::proofs::net::ipv6::ndisc::backlog_safety` |
| Proxy-ND list | `kani::proofs::net::ipv6::ndisc::proxy_safety` |

### Layer 2: TLA+ models

- `models/net/ipv6_ndisc.tla` (mandatory per `net/ipv6/00-overview.md` Layer 2) — proves RFC 4861 ND state machine soundness. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-table neighbour hash | every entry's hash bucket matches `arp_hashfn(key)` | `kani::proofs::net::ipv6::ndisc::hash_invariants` |
| Per-neighbour state | `nud_state` value is in the allowed set; transitions are RFC-4861-allowed | `kani::proofs::net::ipv6::ndisc::state_invariants` |
| arp_queue size accounting | `arp_queue_len_bytes` matches Σ `skb->truesize` over queued skbs | `kani::proofs::net::ipv6::ndisc::queue_invariants` |

### Layer 4: Functional correctness (opt-in)

- **ND state-machine refinement theorem** via TLA+ refinement — proves: implementation refines the abstract RFC 4861 state machine.
- **TLV option-walk termination theorem** via Verus — proves: TLV parser terminates within ≤ MAX_OPTIONS_WALK iterations even on adversarial chains; rejects 0-length options that would loop.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-neighbour `refcnt` uses `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-neighbour slab cache + per-table slab cache | § Mandatory |
| **MEMORY_SANITIZE** | freed neighbour cleared (lladdr is sensitive on private LANs) | § Default-on configurable off |
| **SIZE_OVERFLOW** | TLV-option-length arithmetic uses checked operators (defense vs. CVE-2018-class TLV-length-overflow) | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: `nd_msg_ops`, per-table ops vtables `static const`
- **USERCOPY**: NLA parsing uses `nla_*` accessors (bound-checked)
- **KERNEXEC**: ND option-handler dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_capable(CAP_NET_ADMIN)` already required for RTM_NEWNEIGH/DELNEIGH writes; GR-RBAC policy can deny per-subject (default empty).
- Useful default policy: deny RTM_NEWNEIGH outside gradm-marked `network_admin` role.
- DoS-prevention: per-table `gc_thresh3` cap prevents neighbour-table exhaustion via flood of NS to nonexistent targets.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — ND semantics are exhaustively specified by RFC 4861 + upstream)

## Out of Scope

- IPv6 routing (cross-ref `net/ipv6/route.md`)
- IPv6 address autoconfiguration (cross-ref `net/ipv6/addrconf.md`)
- ICMPv6 generic processing (cross-ref `net/ipv6/icmpv6.md` — only the ND-bound subset is covered here)
- IPv4 ARP (cross-ref `net/ipv4/arp.md` — uses the same `neighbour` framework but different protocol surface)
- 32-bit-only paths
- Implementation code
