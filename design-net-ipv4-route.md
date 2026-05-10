---
title: "Tier-3: net/ipv4/route.c — IPv4 routing core (FIB lookup + per-route metric cache + per-cpu rt_cache)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`net/ipv4/route.c` is the IPv4 route-lookup hot-path entry — every outbound IPv4 skb dispatches via `ip_route_output_flow` to find a route + per-route dst_entry; every inbound skb dispatches via `ip_route_input_*` for forwarding decision. Backed by FIB (Forwarding Information Base) trie at `net/ipv4/fib_trie.c` (LC-Trie); per-route fib_info + fib_alias track route-attributes, gateway, prefsrc. Per-CPU `rt_cache` exists per-route for Path-MTU + ICMP redirects. Critical for: every IPv4 packet that requires routing decision (router, host, transparent-proxy, NAT).

This Tier-3 covers `route.c` (~3799 lines) + `fib_trie.c` (~3041 lines) + `fib_frontend.c` (~1713 lines).

### Acceptance Criteria

- [ ] AC-1: Boot system; ip route show: shows MAIN table routes.
- [ ] AC-2: Outbound TCP: per-skb ip_route_output_flow → rtable + nexthop.
- [ ] AC-3: Default route: 0.0.0.0/0 → gateway; per-flow uses default.
- [ ] AC-4: Multipath: 2-nexthop default; per-flow hash distributes.
- [ ] AC-5: ICMP redirect: receive ICMP redirect; per-route gw updated.
- [ ] AC-6: PMTU: receive ICMP frag-needed; per-route pmtu reduced.
- [ ] AC-7: Per-namespace: 2 netns with distinct routes.
- [ ] AC-8: ip route add via netlink: route inserted; ip route show reflects.
- [ ] AC-9: 100K-route stress: FIB lookups within 100ns.
- [ ] AC-10: Live update: route-table change; rt_cache invalidated; new lookups use new state.

### Architecture

`Rtable`:

```
struct Rtable {
  dst: DstEntry,
  rt_genid: u32,
  rt_flags: u32,
  rt_type: u16,
  rt_is_input: bool,
  rt_uses_gateway: bool,
  rt_iif: i32,
  rt_gw_family: u16,
  rt_gw4: u32,
  rt_pmtu: u32,
  rt_mtu_locked: bool,
  ...
}

struct FibTable {
  tb_id: u32,
  tb_lock: RwLock<()>,
  tb_data: KBox<[u8]>,                          // trie root
  tb_num_default: u32,
  ...
}

struct FibInfo {
  fib_net: KArc<Net>,
  fib_protocol: u8,
  fib_scope: u8,
  fib_type: u8,
  fib_priority: u32,
  fib_metrics: KArc<DstMetrics>,
  fib_nhs: u32,
  fib_nh: KVec<FibNh>,
  fib_prefsrc: u32,
  ...
}

struct FibAlias {
  fa_list: ListNode,
  fa_info: KArc<FibInfo>,
  fa_tos: u8,
  fa_type: u8,
  fa_state: u8,
  fa_default: bool,
}

struct FibResult {
  prefix: u32,
  prefixlen: u8,
  nh_sel: u8,
  type_: u8,
  scope: u8,
  fi: KArc<FibInfo>,
  table: KArc<FibTable>,
  fa_head: KArc<FibAlias>,
}
```

`IpRoute::output_flow(net, &fl4, sk)`:
1. ret := fib_lookup(net, &fl4, &res, 0).
2. If ret: return ERR_PTR(ret).
3. dev := res.fi.fib_nh[res.nh_sel].nh_dev.
4. saddr := fl4.saddr or inet_select_addr(dev, fl4.daddr, scope).
5. rt := __mkroute_output(&res, &fl4, ...).
6. Return rt.

`Fib::lookup(net, &fl4, &res, flags)`:
1. tbid := determine table-id from fl4 (per-policy-routing rules).
2. tb := fib_get_table(net, tbid).
3. err := fib_table_lookup(tb, fl4, &res, flags).
4. Return err.

`FibTable::lookup(tb, &flp, &res, flags)` (LC-Trie):
1. Walk trie from root using flp->daddr.
2. At each node: longest-prefix-match.
3. Per-leaf: walk fa_list; match per-TOS / per-flag.
4. Result: fi + alias.
5. Return.

`IpRoute::__mkroute_output(&res, &fl4, ...)`:
1. rt := rt_dst_alloc(dev, flags, type, noxfrm).
2. rt.rt_iif = -1 (output).
3. rt.rt_gw_family = res.fi.gw_family.
4. rt.rt_gw4 = res.fi.fib_nh[res.nh_sel].gw.
5. dst_set_metrics(&rt.dst, res.fi.fib_metrics).
6. Return rt.

### Out of Scope

- IPv6 routing (covered in `net/ipv6/route.md` future Tier-3)
- Policy-routing rules (covered in `net/ipv4/fib_rules.md` future Tier-3)
- Netlink RTM_NEWROUTE / RTM_DELROUTE (covered separately)
- TCP / UDP (covered separately)
- VRF (Virtual Routing & Forwarding) (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rtable` | per-route IPv4 dst | `net::ipv4::route::Rtable` |
| `struct fib_table` | per-tableid FIB | `FibTable` |
| `struct fib_info` | per-route info (gw, dev, prefsrc) | `FibInfo` |
| `struct fib_alias` | per-(prefix, length) alias | `FibAlias` |
| `struct fib_result` | per-lookup result | `FibResult` |
| `ip_route_output_flow(net, &fl4, sk)` | per-flow output route | `IpRoute::output_flow` |
| `ip_route_output_key_hash(net, &fl4, skb)` | per-flow lookup | `IpRoute::output_key_hash` |
| `ip_route_input_noref(skb, daddr, saddr, tos, dev)` | per-skb input route | `IpRoute::input_noref` |
| `ip_route_input_slow(skb, daddr, saddr, tos, dev, &res)` | per-skb slow-path | `IpRoute::input_slow` |
| `__mkroute_output(...)` | per-flow rtable construct | `IpRoute::__mkroute_output` |
| `__mkroute_input(...)` | per-input rtable construct | `IpRoute::__mkroute_input` |
| `fib_lookup(net, &fl4, &res, flags)` | per-flow FIB lookup | `Fib::lookup` |
| `fib_table_lookup(tb, &flp, &res, flags)` (fib_trie.c) | per-table lookup | `FibTable::lookup` |
| `fib_table_insert(net, tb, &cfg)` | per-route insert | `FibTable::insert` |
| `fib_table_delete(net, tb, &cfg)` | per-route delete | `FibTable::delete` |
| `fib_table_flush(net, tb)` | per-table flush | `FibTable::flush` |
| `fib_create_info(&cfg)` | per-route info alloc | `Fib::create_info` |
| `fib_release_info(fi)` | per-route info release | `Fib::release_info` |
| `__ip_dev_find(net, addr, devref)` | per-dev IPv4-addr lookup | `IpDev::find` |
| `inet_select_addr(dev, dst, scope)` | per-dev source-addr select | `Inet::select_addr` |
| `rt_cache_route(...)` | per-cpu rt cache | `IpRoute::cache_route` |
| `rt_dst_alloc(dev, flags, type, noxfrm)` | per-flow dst alloc | `IpRoute::dst_alloc` |
| `rt_set_nexthop(rth, daddr, &res, fl4, prefix, type)` | per-route nexthop set | `IpRoute::set_nexthop` |

### compatibility contract

REQ-1: Per-route `rtable`:
- `dst` (struct dst_entry; cached output func, refcount).
- `rt_genid` (per-namespace generation; bumped on route-table change).
- `rt_flags` (RTCF_*).
- `rt_type` (RTN_UNICAST / _BROADCAST / _MULTICAST / _LOCAL).
- `rt_is_input` (input route vs output).
- `rt_uses_gateway` (bool: has gateway).
- `rt_iif` (input ifindex).
- `rt_gw_family` / `rt_gw4` / `rt_gw6` (gateway address).
- `rt_pmtu` (per-route Path-MTU).

REQ-2: Per-FIB `fib_table`:
- `tb_id` (table id; LOCAL=255, MAIN=254, DEFAULT=253, custom).
- `tb_data[]` (variable: per-trie root or per-hash buckets).
- `tb_num_default` (default route count).
- `tb_lock` (rcu + writer-lock).

REQ-3: Per-route `fib_info`:
- `fib_net` (per-namespace).
- `fib_protocol` (RTPROT_*).
- `fib_scope` (RT_SCOPE_*).
- `fib_type` (RTN_*).
- `fib_priority` (metric).
- `fib_metrics` (KArc<DstMetrics>; MTU, hop-limit, etc.).
- `fib_nh` (KVec<FibNh>; nexthops; multipath).
- `fib_nhs` (count of nexthops).
- `fib_dev` (per-output device).
- `fib_prefsrc` (preferred source addr).

REQ-4: Per-route `fib_alias`:
- `fa_list` (chain).
- `fa_info` (FibInfo back-ref).
- `fa_tos` (per-TOS qualifier).
- `fa_type` (RTN_*).
- `fa_state` (FA_S_ACCESSED).
- `fa_default` (default-route flag).

REQ-5: Per-flow `flowi4`:
- `flowi4_oif` (output ifindex).
- `flowi4_iif` (input ifindex).
- `flowi4_mark` (skb mark).
- `flowi4_tos` (Type-of-Service).
- `flowi4_scope` (RT_SCOPE_*).
- `flowi4_proto` (IPPROTO_*).
- `flowi4_flags`.
- `daddr` / `saddr` (per-flow addresses).
- `fl4_sport` / `fl4_dport` (per-flow ports).

REQ-6: Per-output route flow (`ip_route_output_flow`):
1. fib_lookup(net, &fl4, &res, 0).
2. If err: return ERR_PTR(err).
3. __mkroute_output(&res, &fl4, ...): construct rtable.
4. Return rtable.

REQ-7: Per-input route flow (`ip_route_input_noref`):
1. cache := per-CPU rt_cache lookup by (daddr, saddr, iif).
2. If hit: return.
3. ip_route_input_slow:
   - fib_lookup with input-direction.
   - __mkroute_input: construct rtable; set rt_iif.
   - Cache in per-CPU rt_cache.

REQ-8: FIB lookup (LC-Trie):
- Per-trie node: prefix + len + child nodes.
- Lookup: longest-match-prefix walk.
- O(log N) for N routes.

REQ-9: Per-route ICMP redirect handling:
- ICMP-redirect updates per-route gateway.
- Per-flow rt_cache invalidated.

REQ-10: Per-route Path-MTU update:
- ICMPv4 fragmentation-needed → update rtable.rt_pmtu.
- Per-route metric.mtu used by IP-layer for TX.

REQ-11: Multipath routing:
- Per-route multiple FibNh.
- Per-flow hash → select nexthop.
- ECMP (Equal-Cost-Multi-Path) supported.

REQ-12: Per-namespace routing:
- Per-net struct with per-table routes.
- net_namespaces fully isolated.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rt_genid_consistent` | INVARIANT | per-namespace rt_genid bumped on route-table change. |
| `fib_alias_chain_no_uaf` | UAF | RCU-protected chain; defense against del-during-walk. |
| `nh_sel_bounded` | OOB | per-result nh_sel < res.fi.fib_nhs. |
| `rt_pmtu_min_bounded` | INVARIANT | rt_pmtu ≥ IPV4_MIN_MTU (68). |
| `prefix_len_bounded` | OOB | per-route prefixlen ≤ 32. |

### Layer 2: TLA+

`net/ipv4/fib_lookup.tla`:
- Per-FIB longest-match resolution.
- Properties:
  - `safety_longest_match` — lookup returns longest-prefix-match per packet daddr.
  - `safety_per_namespace_isolated` — per-namespace lookups don't cross.
  - `liveness_lookup_terminates` — assuming finite trie depth, lookup terminates.

`net/ipv4/route_cache.tla`:
- Per-CPU rt_cache.
- Properties:
  - `safety_cache_invalidation_on_route_change` — per-rt_genid mismatch invalidates cache entry.
  - `safety_cache_consistent_with_fib` — cache hit reflects current FIB state.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IpRoute::output_flow` post: rtable returned with valid dst + gw + dev | `IpRoute::output_flow` |
| `Fib::lookup` post: res.fi populated; res.prefix matches longest-match | `Fib::lookup` |
| `FibTable::insert` post: route inserted; tb_num_default updated if default | `FibTable::insert` |
| `IpRoute::cache_route` post: rt_cache populated; subsequent same-flow hits | `IpRoute::cache_route` |

### Layer 4: Verus/Creusot functional

`Per-flow IP route lookup: returns nexthop matching longest-prefix in FIB` semantic equivalence: per-flow the resulting rtable.gw + dev correspond to the routing-table's longest-prefix-match for daddr.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

route-specific reinforcement:

- **Per-rt_genid invalidation** — defense against stale-cache after route-table change.
- **RCU-protected FIB walk** — defense against insert/delete-during-lookup UAF.
- **Per-route nh_sel bounded** — defense against multipath select OOB.
- **Per-PMTU min-MTU floor** — defense against pathological PMTU attack.
- **ICMP-redirect-rate-limit** — defense against ICMP-redirect storm.
- **Per-route metric ref-counted** — defense against metric-free during use.
- **Per-namespace tables isolated** — defense against cross-netns leak.
- **Per-FIB tb_id namespace-scoped** — defense against tb_id collision across netns.
- **Per-flow flowi4 sanitized** — defense against attacker-controlled fl4 fields.
- **prefix_len validated ≤ 32** — defense against trie-walk OOB.
- **Multipath hash domain validated** — defense against attacker biasing nexthop selection.
- **Per-route gw addr validated** — defense against gw == self causing loop.

