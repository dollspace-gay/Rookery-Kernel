# Tier-3: net/ipv4/fib_trie.c — IPv4 FIB lookup (LC-trie route-table)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/ipv4/route.md
upstream-paths:
  - net/ipv4/fib_trie.c (~3041 lines)
  - net/ipv4/fib_frontend.c (~1713 lines)
  - net/ipv4/fib_semantics.c
  - include/net/fib_rules.h
  - include/net/ip_fib.h
-->

## Summary

Linux IPv4 FIB (Forwarding Info Base) is implemented as an **LC-trie** (Level-Compressed trie) per-table. Per-net has primary (RT_TABLE_MAIN) + local (RT_TABLE_LOCAL) tables; per-net-namespace can have additional tables via `ip rule`. Per-route insertion `ip route add 10.0.0.0/8 via 192.168.1.1 dev eth0` calls fib_frontend → fib_trie_table.create_fib_alias → tnode insert. Per-lookup `fib_lookup(net, fl4, &res, flags)` walks trie via prefix-match + per-fib_alias scan + per-fib_info egress-pick. Per-result returns nh_dev + nh_gw + scope + type + tos. Critical for: IP routing, multipath, policy routing.

This Tier-3 covers `net/ipv4/fib_trie.c` (~3041 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct fib_table` | per-table | `FibTable` |
| `struct trie` | LC-trie root | `Trie` |
| `struct tnode` / `struct leaf_info` (replaced by tnode + fib_alias) | LC-trie node | `Tnode` |
| `struct key_vector` | per-trie-node | `KeyVector` |
| `fib_table_lookup()` | per-lookup | `Fib::table_lookup` |
| `fib_table_insert()` | per-route-add | `Fib::table_insert` |
| `fib_table_delete()` | per-route-del | `Fib::table_delete` |
| `fib_table_flush()` | per-link-down | `Fib::table_flush` |
| `fib_trie_table()` | per-table-alloc | `Fib::trie_table` |
| `fib_trie_unmerge()` | per-table-detach | `Fib::trie_unmerge` |
| `fib_trie_get_first()` / `_next()` | per-table-iter | `Fib::trie_get_*` |
| `fib_alias_find()` | per-prefix-find | `Fib::alias_find` |
| `fib_create_info()` | per-fib_info create | `Fib::create_info` |
| `fib_lookup()` | top-level wrapper | `Fib::lookup` |
| `fib_select_default()` | per-default-route pick | `Fib::select_default` |
| `fib_select_multipath()` | per-ECMP pick | `Fib::select_multipath` |
| `fib_table_dump()` | per-NETLINK route-dump | `Fib::table_dump` |

## Compatibility contract

REQ-1: Per-net IPv4 FIB tables:
- RT_TABLE_LOCAL (255): local-deliver routes (per-IP and broadcast on this host).
- RT_TABLE_MAIN (254): default routes for forwarding/transmit.
- RT_TABLE_DEFAULT (253): "default" rule fallback.
- Per-CONFIG_IP_MULTIPLE_TABLES: arbitrary id 1..2^31-1.

REQ-2: struct fib_table:
- tb_id: table id.
- tb_num_default: count of default routes.
- tb_data[]: pointer to struct trie.

REQ-3: struct trie:
- root: per-key-vector (RCU).
- per-stat counters (insertions, deletions, lookups).

REQ-4: LC-trie node (key_vector):
- key: prefix.
- pos: bit-position.
- bits: child-array-size in bits (2^bits children).
- slen: longest-prefix-stored-below + 1 (route summary for skip).
- tnode[]: array of children, indexed by per-key-bits-at-pos.
- For leaves: list of fib_alias.

REQ-5: struct fib_alias:
- fa_list: list-of-aliases at same prefix (sorted by tos + prio).
- fa_info: → struct fib_info (egress + nexthop).
- fa_tos: TOS.
- fa_type: RTN_UNICAST / LOCAL / BROADCAST / ANYCAST / MULTICAST / BLACKHOLE / UNREACHABLE / PROHIBIT / THROW / NAT / XRESOLVE.
- fa_state: FA_S_ACCESSED.
- fa_slen: prefix-length.
- tb_id: redundant table-id.

REQ-6: struct fib_info:
- fib_clntref: refcount.
- fib_protocol: BOOT / KERNEL / STATIC / RA / DHCP / ZEBRA / BIRD / OSPF / BGP / etc.
- fib_scope: HOST / LINK / UNIVERSE.
- fib_type: same as fa_type.
- fib_priority: route metric.
- fib_metrics: per-metric (RTAX_*).
- fib_nhs: count of nexthops.
- fib_nh[fib_nhs]: per-nexthop.
- fib_nh_table: → struct nexthop (per-net managed nexthop object) optional.

REQ-7: struct fib_nh / fib_nh_common:
- nh_oif: outgoing if-index.
- nh_dev: net_device.
- nh_gw: gateway IP.
- nh_scope: HOST / LINK / UNIVERSE.
- nh_weight: ECMP weight.
- nh_flags: ONLINK / DEAD / etc.

REQ-8: fib_table_lookup(tb, fl4, res, fib_flags):
- /* Walk trie */
- pref = fl4.daddr.
- node = trie.root.
- while node:
  - if node.bits == 0 (leaf): scan fib_alias list.
  - else: child = node.tnode[get_index(pref, node.pos, node.bits)].
- /* Per-found alias */
- res.fi = fa_info.
- res.fa_type = fa.fa_type.
- res.prefixlen = fa.fa_slen - 1 (or appropriate).
- res.nh_sel = (multipath: hash-pick).

REQ-9: fib_table_insert(tb, cfg, extack):
- /* Build fib_info */
- fi = fib_create_info(cfg).
- /* Find/create leaf */
- leaf = trie.find_or_create_leaf(prefix, len).
- /* Insert fib_alias sorted */
- fa_new = alloc(fib_alias).
- fa_new.fa_info = fi; fa_new.fa_tos = cfg.tos; ...
- list_add_rcu(&fa_new, leaf.fib_alias_list).

REQ-10: fib_table_delete(tb, cfg, extack):
- Find matching fib_alias.
- list_del_rcu.
- fib_alias_kfree_rcu.
- if alias-list empty: tnode_purge.

REQ-11: fib_table_flush(net, tb, flush_all):
- Iterate trie; for each fib_alias:
  - if fi.fib_dev->down: delete.

REQ-12: Multipath selection:
- fib_select_multipath(res, hash):
  - For each nh: if hash < cumulative_weight: pick this nh.

REQ-13: ECMP weight scaling:
- fib_rebalance: scale weights to sum-of-active.

REQ-14: nexthop object (RFC compliant):
- Per-net `struct nexthop` registry separate from fib_nh.
- Per-route can reference nh-id instead of inline fib_nh.

REQ-15: Per-trie compaction:
- LC-trie auto-resizes per-load (split / collapse).

REQ-16: RCU synchronization:
- Lookup is RCU-walk.
- Insert/delete via rtnl-lock + RCU-publish/free.

REQ-17: NETLINK dump:
- fib_table_dump iterates trie + emits RTM_NEWROUTE per-fib_alias.

REQ-18: TOS routing:
- Per-fib_alias.fa_tos: only matches if fl4.flowi4_tos == fa_tos.

## Acceptance Criteria

- [ ] AC-1: ip route add 10.0.0.0/8 dev eth0: trie has prefix; lookup matches.
- [ ] AC-2: ip route del 10.0.0.0/8: trie no longer matches.
- [ ] AC-3: ip route add 10.0.0.0/8 nexthop dev eth0 weight 1 nexthop dev eth1 weight 1: ECMP; multipath-select per-hash.
- [ ] AC-4: ip route show table local: includes local IPs + broadcasts.
- [ ] AC-5: ip rule add fwmark 0x1 lookup 100; ip route add ... table 100: per-rule selects table.
- [ ] AC-6: lookup 1.2.3.4 with no matching: -ENETUNREACH.
- [ ] AC-7: lookup 0.0.0.0/0 default: matches default route.
- [ ] AC-8: ip rule + ip route NETLINK dump: matches kernel.
- [ ] AC-9: ifdown eth0: routes via eth0 marked DEAD; flushed if NOTIFY_DOWN.
- [ ] AC-10: TOS-routing: per-tos field selects matching alias.
- [ ] AC-11: Per-RCU-lookup non-blocking; insert/delete under rtnl-lock.

## Architecture

```
struct FibTable {
  tb_id: u32,
  tb_num_default: u32,
  tb_data: Box<Trie>,
}

struct Trie {
  root: RcuPtr<KeyVector>,
  /* stats */
}

struct KeyVector {
  key: u32,
  pos: u8,
  bits: u8,                          // 2^bits children; 0 = leaf
  slen: u8,
  // For internal: child[2^bits] tnode pointers
  // For leaf: list of fib_alias
  tnode: Vec<RcuPtr<KeyVector>>,
  fib_alias: Vec<FibAlias>,          // for leaves
}

struct FibAlias {
  fa_info: RcuPtr<FibInfo>,
  fa_tos: u8,
  fa_type: u8,
  fa_state: u8,
  fa_slen: u8,
  tb_id: u32,
}

struct FibInfo {
  fib_protocol: u8,
  fib_scope: u8,
  fib_type: u8,
  fib_priority: u32,
  fib_metrics: [u32; RTAX_MAX],
  fib_nhs: u8,
  fib_nh: [FibNh; MAX_NH],
  fib_nh_table: Option<RcuPtr<Nexthop>>,
  fib_clntref: AtomicU32,
}
```

`Fib::table_lookup(tb, fl4, res, flags) -> Result<()>`:
1. trie = container_of(tb.tb_data, Trie).
2. n = rcu_dereference(trie.root).
3. while n.bits != 0:
   - cidx = (fl4.daddr >> (32 - n.pos - n.bits)) & ((1 << n.bits) - 1).
   - c = rcu_dereference(n.tnode[cidx]).
   - if !c: return -ENETUNREACH.
   - n = c.
4. /* leaf */
5. for fa in n.fib_alias:
   - if (fl4.daddr ^ n.key) & (BIT_MASK(fa.fa_slen)) == 0 ∧ tos-match:
     - fi = rcu_dereference(fa.fa_info).
     - if fi.fib_dead: continue.
     - res.fi = fi.
     - res.type = fa.fa_type.
     - res.prefixlen = fa.fa_slen - 1.
     - if fi.fib_nhs > 1: res.nh_sel = Fib::select_multipath(res, hash).
     - else: res.nh_sel = 0.
     - return Ok(()).
6. return -ENETUNREACH.

`Fib::table_insert(tb, cfg, extack) -> Result<()>`:
1. fi = Fib::create_info(net, cfg, extack).
2. if !fi: return PTR_ERR(fi).
3. /* Find/create leaf for prefix */
4. l = trie_lookup_or_create(tb, prefix, prefix_len).
5. /* Insert fib_alias sorted by (priority, tos) */
6. fa = alloc(FibAlias).
7. fa.fa_info = fi; fa.fa_tos = cfg.tos; fa.fa_type = cfg.type; fa.fa_slen = prefix_len + 1.
8. list_add_rcu_sorted(&fa, &l.fib_alias).
9. /* Notify */
10. call_fib4_notifier(FIB_EVENT_ENTRY_ADD).

`Fib::select_multipath(res, hash) -> i32`:
1. fi = res.fi.
2. total_weight = fib_weight_total(fi).
3. /* Scaled hash mod total */
4. h = (hash * total_weight) >> 32.
5. cumul = 0.
6. for nh_idx in 0..fi.fib_nhs:
   - cumul += fi.fib_nh[nh_idx].nh_weight.
   - if cumul > h: return nh_idx.
7. return fi.fib_nhs - 1.

`Fib::table_flush(net, tb, flush_all) -> i32`:
1. /* Walk trie */
2. n = trie_first_leaf.
3. count = 0.
4. while n:
   - for fa in n.fib_alias:
     - fi = fa.fa_info.
     - if fi.fib_dead ∨ flush_all ∨ fib_dev_dead(fi):
       - list_del_rcu(&fa).
       - fib_alias_kfree_rcu(fa).
       - count++.
   - if !n.fib_alias: trie_purge_node(n).
   - n = trie_next_leaf.
5. return count.

`Fib::create_info(net, cfg, extack) -> *FibInfo`:
1. /* Validate type */
2. if cfg.type not in valid_types: -EINVAL.
3. /* Allocate */
4. fi = kzalloc(sizeof(FibInfo) + nhs * sizeof(FibNh)).
5. /* Per-nh */
6. for i in 0..nhs:
   - nh = &fi.fib_nh[i].
   - nh.nh_dev = dev_get_by_index(net, cfg.nh_oif[i]).
   - nh.nh_gw = cfg.nh_gw[i].
   - nh.nh_scope = compute_scope(nh.nh_gw).
7. fib_create_info_finish(fi).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `trie_root_rcu_dereference` | INVARIANT | per-lookup: read under rcu_read_lock. |
| `tnode_index_within_2bits` | INVARIANT | per-tnode: cidx < 2^bits. |
| `fib_alias_sorted_by_tos_prio` | INVARIANT | per-leaf list: sorted asc tos, then prio. |
| `fib_info_refcounted` | INVARIANT | per-fib_alias.fa_info: refcount tracked. |
| `fib_dev_alive_for_lookup_match` | INVARIANT | per-match: !fi.fib_dead. |
| `multipath_idx_lt_nhs` | INVARIANT | per-select_multipath: returned idx < fi.fib_nhs. |

### Layer 2: TLA+

`net/ipv4/fib-trie.tla`:
- Per-trie insert + per-lookup + per-delete + per-flush.
- Per-RCU-publish ordering.
- Properties:
  - `safety_no_alias_after_flush_dead` — per-flush: no leaf with dead-fi alias.
  - `safety_lookup_terminates` — per-lookup: at most 32 trie-steps.
  - `safety_concurrent_lookup_progresses_during_insert` — per-RCU: lookups don't block.
  - `liveness_per_route_add_observable_after_grace` — per-add: subsequent lookup finds it.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Fib::table_lookup` post: returns res.fi or -ENETUNREACH | `Fib::table_lookup` |
| `Fib::table_insert` post: alias visible in lookup | `Fib::table_insert` |
| `Fib::table_delete` post: alias unlinked + freed-after-grace | `Fib::table_delete` |
| `Fib::select_multipath` post: per-weight proportional pick | `Fib::select_multipath` |
| `Fib::create_info` post: nhs validated, refs taken | `Fib::create_info` |

### Layer 4: Verus/Creusot functional

`Per-route → trie-leaf insert → fib_alias add; per-lookup → trie-walk → fib_alias scan → fib_info egress-pick → multipath-select` semantic equivalence: per-RFC 1812 + per-Linux-routing-reference.

## Hardening

(Inherits row-1 features from `net/ipv4/00-overview.md` § Hardening.)

FIB-trie reinforcement:

- **Per-RCU-walk lookup non-blocking** — defense against per-lookup-blocked-by-update.
- **Per-tnode cidx bounds-checked** — defense against per-OOR child access.
- **Per-fib_alias prefix-mask check** — defense against per-false-positive route-match.
- **Per-fib_info refcount-tracked** — defense against per-fib_info UAF.
- **Per-fib_dead flagged on dev_down** — defense against per-stale-route forwarding.
- **Per-CAP_NET_ADMIN for route-add/del** — defense against unprivileged routing-control.
- **Per-table-id range checked** — defense against per-OOR table ref.
- **Per-multipath weight-overflow guarded** — defense against per-DOS heavy-ECMP.
- **Per-RTM netlink size-validated** — defense against per-malformed-netlink.
- **Per-rtnl_lock for insert/delete** — defense against per-concurrent-update inconsistency.
- **Per-flush iter snapshot RCU-safe** — defense against per-iter-during-update UAF.
- **Per-nexthop_object lifecycle managed via netlink** — defense against per-stale-nexthop.

## Grsecurity/PaX-style Reinforcement

Rationale: the FIB-trie is the IPv4 routing-table lookup structure — every outgoing packet on the box transits its LC-trie lookup, and every RTM_NEWROUTE / RTM_DELROUTE mutates it. Corrupting trie internal nodes (`tnode`) or leaf-info (`fib_alias`) yields arbitrary route-redirect (traffic stealing) or RCU-walk UAF.

Baseline (cross-ref `net/00-overview.md` § Hardening):
- **PAX_USERCOPY**: netlink `RTA_*` attribute parsing for RTM_NEWROUTE uses bounded `nla_get_*`/`nla_memcpy`; `/proc/net/route` and `/proc/net/fib_trie` outputs use `seq_printf` with no slab-block dump.
- **PAX_KERNEXEC**: `fib_rules_ops` per-family, `fib_table.tb_ops` (if used), and per-`nexthop` callbacks live in `__ro_after_init`; refuse re-registration post-init.
- **PAX_RANDKSTACK**: every RTM_NEWROUTE / `fib_table_lookup` re-randomises kernel-stack offset before the deeply-recursive trie walk.
- **PAX_REFCOUNT**: `fib_info.fib_refcnt`, `fib_nh.nh_refcnt`, `nexthop.refcnt`, `fib_table.tb_refcnt` use saturating `Refcount`; defends RTM_NEWROUTE storm.
- **PAX_MEMORY_SANITIZE**: trie node free (`tnode_free`, `fib_alias_free`) zero-fills the slab block before slab-return, including the `fa_info` pointer slot.
- **PAX_UDEREF**: netlink attribute walks use `nla_data`/`nla_get_*` accessors only.
- **PAX_RAP / kCFI**: indirect calls through `fib_rules_ops->action`/`match`/`configure`/`compare`/`fill`, `nh_notifier_info` callback, and `fib_notifier_ops->fib_seq_read`/`->fib_dump` are kCFI-tagged.
- **GRKERNSEC_HIDESYM**: `fib_info*`, `fib_alias*`, `tnode*` kernel pointers never rendered to `/proc/net/route` or RTM_GETROUTE responses; only opaque ifindex + prefix + nexthop-IP.
- **GRKERNSEC_DMESG**: FIB-table-full warnings ratelimited; CAP_SYSLOG to read.

FIB-trie-specific reinforcement:
- **FIB-trie PAX_REFCOUNT** — `fib_info.fib_refcnt` saturates at INT_MAX with BUG-and-grsec-audit; defends against multipath-route storm attempting refcount wrap.
- **RCU-walk lookup safety** — `fib_table_lookup` runs entirely under `rcu_read_lock()`; mutators publish via `rcu_assign_pointer` + `synchronize_rcu` before slab-free; mismatch → BUG (no UAF window).
- **RTM_NEWROUTE strict CAP_NET_ADMIN in init_user_ns** — refuse from non-init userns even with CAP_NET_ADMIN (grsec policy).
- **NLA_POLICY_STRICT on RTM_NEWROUTE attribute table** — reject unknown attribute types (defends against TLV-fuzzing leaking uninitialised fields).
- **Multipath weight-overflow guard** — `fib_nh_match` cumulative-weight overflow detected via `check_add_overflow`; saturates and refuses route (defends ECMP-DOS via 65535-nexthop route).
- **Per-rtnl_lock invariant** — every mutator asserts `ASSERT_RTNL()`; lockless mutation → BUG().

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/ipv4/route.c (covered in `route.md` Tier-3)
- net/ipv4/fib_frontend.c (covered separately if expanded)
- net/ipv4/fib_semantics.c (covered separately if expanded)
- net/ipv4/fib_rules.c (covered separately if expanded)
- net/ipv4/nexthop.c (covered separately if expanded)
- Implementation code
