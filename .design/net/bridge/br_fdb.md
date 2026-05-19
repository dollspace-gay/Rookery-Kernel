# Tier-3: net/bridge/br_fdb.c — Linux bridge forwarding-database (FDB)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/bridge/br_input.md
upstream-paths:
  - net/bridge/br_fdb.c (~1639 lines)
  - net/bridge/br_private.h
  - include/uapi/linux/neighbour.h (NDA_*)
-->

## Summary

`br_fdb.c` implements the bridge forwarding-database: per-(MAC, VID, ingress-port) entries ageing-resolved per `ageing_time` (default 300s). Per-RX-skb on FORWARDING port: `br_fdb_update` learns src-MAC. Per-FDB lookup via `br_fdb_find_rcu` for forwarding decision. Per-static entry: persists across ageing. Per-local entry: bridge-master MAC delivery (vs forwarded). Per-rhashtable hashes by (MAC, VID). Per-NDA_* netlink ABI for `bridge fdb` userspace. Per-switchdev offload via ndo_fdb_add/del. Critical for: bridge L2 forwarding-decision; per-MAC mobility tracking.

This Tier-3 covers `br_fdb.c` (~1639 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct net_bridge_fdb_entry` | per-FDB entry | `BridgeFdbEntry` |
| `br_fdb_init()` | per-module init (kmem_cache) | `BridgeFdb::init` |
| `br_fdb_update()` | per-rx-MAC learning | `BridgeFdb::update` |
| `br_fdb_find()` | per-(MAC, vid) lookup (under lock) | `BridgeFdb::find` |
| `br_fdb_find_rcu()` | per-(MAC, vid) RCU lookup | `BridgeFdb::find_rcu` |
| `br_fdb_find_port()` | per-(br_dev, MAC, vid) → ingress-port | `BridgeFdb::find_port` |
| `fdb_create()` | per-entry alloc | `BridgeFdb::fdb_create` |
| `fdb_delete()` | per-entry free | `BridgeFdb::fdb_delete` |
| `fdb_delete_local()` | per-local-entry free | `BridgeFdb::fdb_delete_local` |
| `br_fdb_find_delete_local()` | per-local lookup + delete | `BridgeFdb::find_delete_local` |
| `br_fdb_changeaddr()` | per-port MAC change → fdb update | `BridgeFdb::changeaddr` |
| `br_fdb_age_out()` | periodic ageing-walker | `BridgeFdb::age_out` |
| `br_fdb_dump()` | per-RTM_GETNEIGH dump | `BridgeFdb::dump` |
| `br_fdb_add()` | per-RTM_NEWNEIGH netlink | `BridgeFdb::add` |
| `br_fdb_delete()` | per-RTM_DELNEIGH | `BridgeFdb::delete` |
| `br_fdb_dump_dest()` | per-rtnetlink emit | `BridgeFdb::dump_dest` |
| `br_fdb_external_learn_add()` | per-switchdev-learn | `BridgeFdb::external_learn_add` |
| `br_fdb_replay()` | per-switchdev resync | `BridgeFdb::replay` |
| `br_fdb_test_addr()` | per-pkt is-local check | `BridgeFdb::test_addr` |

## Compatibility contract

REQ-1: Per-FDB-entry layout:
- `addr`: u8[6] MAC.
- `vlan_id`: u16.
- `dst`: *NetBridgePort (ingress-port; NULL = local).
- `is_local`: bool (delivered up vs forwarded).
- `is_static`: bool (persists across ageing).
- `is_sticky`: bool (locked to current port).
- `offloaded`: bool (HW switchdev offloaded).
- `notify`: bool (need RTM_NEWNEIGH event).
- `used`: AtomicLong (last-seen jiffies).
- `updated`: AtomicLong (creation jiffies).
- `flags`: u16 (NDM_FLAG_*).
- `link`: rcu hash node.
- `gc_list`: list-link for gc-walker.

REQ-2: Per-bridge fdb_hash:
- rhashtable keyed by (addr, vlan_id).
- 256-bucket initial; auto-grow.

REQ-3: br_fdb_update(br, p, addr, vid, fastpath):
- f = br_fdb_find_rcu(br, addr, vid).
- If f:
  - if !f.is_static ∧ !f.is_local ∧ f.dst != p:
    - f.dst = p; f.notify = true; mark for RTM_NEWNEIGH.
  - f.used = jiffies.
- Else:
  - f = fdb_create(br, p, addr, vid, false, false).
  - rhashtable_insert(br.fdb_hash, &f.link).
  - f.notify = true.

REQ-4: br_fdb_find_rcu(br, addr, vid):
- key = (addr, vid).
- rhashtable_lookup_fast(br.fdb_hash, key, ...).

REQ-5: Per-ageing:
- ageing_time default 300s.
- br_fdb_cleanup periodic timer (delayed_work).
- For each non-static entry: if (jiffies - f.used) > ageing: fdb_delete.

REQ-6: Per-local entry:
- f.is_local = true; f.dst = NULL.
- Per-port MAC change: fdb_delete_local + fdb_create new.
- Per-bridge MAC change: same.

REQ-7: Per-static entry (NLM_F_CREATE | NLM_F_REPLACE):
- userspace via `bridge fdb add` netlink.
- f.is_static = true; not aged.

REQ-8: Per-NDM_STATE_* values:
- NUD_NOARP: static.
- NUD_PERMANENT: not aged out.
- NUD_REACHABLE: dynamic (default).

REQ-9: Per-RTM_NEWNEIGH / RTM_DELNEIGH:
- userspace via netlink: bridge fdb add/del.
- ndo_fdb_add forwarded to switchdev for HW offload.

REQ-10: Per-EVENT NEIGH_UPDATE:
- per-fdb-state-change: rtnl_notify NEIGH event.
- userspace listeners (e.g. systemd-networkd) react.

REQ-11: Per-switchdev offload:
- bond/team/vlan-aware: per-fdb-add → ndo_fdb_add(real_dev) → HW filter.
- HW-offload: f.offloaded = true.

REQ-12: Per-RCU-protected lookup:
- br_fdb_find_rcu used in fast path (br_input).
- br_fdb_find used under spin_lock (br_fdb mutators).

## Acceptance Criteria

- [ ] AC-1: Bridge add: br_fdb_init alloc'd kmem_cache; br.fdb_hash empty.
- [ ] AC-2: RX MAC=A on port[0]: br_fdb_update creates entry (A, vid=1, dst=port[0]).
- [ ] AC-3: Subsequent RX same MAC=A on port[1]: dst updated to port[1]; notify=true.
- [ ] AC-4: br_fdb_find_rcu returns entry; lookup O(1) avg.
- [ ] AC-5: Ageing timer fires every 1s: non-static entry > 300s old: fdb_delete.
- [ ] AC-6: `bridge fdb add ...static`: f.is_static=true; not aged.
- [ ] AC-7: `bridge fdb show`: lists all entries via br_fdb_dump.
- [ ] AC-8: Per-port MAC change: old local fdb deleted; new local created.
- [ ] AC-9: switchdev-aware NIC: ndo_fdb_add invoked; offloaded=true.
- [ ] AC-10: br_fdb_test_addr: returns true for is_local entries.
- [ ] AC-11: bridge with VLAN-aware: per-(MAC, VID) entries distinct.

## Architecture

Per-FDB entry:

```
struct BridgeFdbEntry {
  rhnode: RhashNode,                              // br.fdb_hash
  addr: [u8; 6],
  vlan_id: u16,
  dst: Option<*NetBridgePort>,                    // None = local
  used: AtomicI64,                                // jiffies
  updated: AtomicI64,
  is_local: bool,
  is_static: bool,
  is_sticky: bool,
  offloaded: bool,
  notify: bool,
  flags: u16,                                     // NDM_FLAG_*
  link: HListLink,
  gc_list: ListLink,
  rcu: RcuHead,
}
```

Per-bridge fdb-state:

```
struct NetBridge {
  ...
  fdb_hash: RhashTable<BridgeFdbEntry>,
  fdb_count: AtomicU32,
  ageing_time: u32,                              // jiffies
  fdb_max_learned: u32,
  gc_work: DelayedWork,
  ...
}
```

`BridgeFdb::init()`:
1. br_fdb_cache = kmem_cache_create("bridge_fdb_cache", sizeof(BridgeFdbEntry), 0, ...).
2. Ok.

`BridgeFdb::find_rcu(br, addr, vid) -> Option<&BridgeFdbEntry>`:
1. key = FdbKey { addr, vid }.
2. f = rhashtable_lookup_fast(&br.fdb_hash, &key, br_fdb_rht_params).
3. Return f.

`BridgeFdb::find(br, addr, vid) -> Option<&mut BridgeFdbEntry>`:
1. /* Caller holds br.fdb_hash.lock or RTNL */
2. Return rhashtable_lookup(&br.fdb_hash, ..., br_fdb_rht_params).

`BridgeFdb::fdb_create(br, source, addr, vid, is_local, is_static) -> Result<&BridgeFdbEntry>`:
1. f = kmem_cache_alloc(br_fdb_cache, GFP_ATOMIC).
2. f.addr = addr; f.vlan_id = vid; f.dst = source.
3. f.is_local = is_local; f.is_static = is_static.
4. f.used = f.updated = jiffies.
5. err = rhashtable_lookup_insert_fast(&br.fdb_hash, &f.rhnode, br_fdb_rht_params).
6. atomic_inc(&br.fdb_count).
7. Return Ok(f).

`BridgeFdb::fdb_delete(br, f, swdev_notify)`:
1. trace_fdb_delete(br, f).
2. rhashtable_remove_fast(&br.fdb_hash, &f.rhnode, br_fdb_rht_params).
3. If f.is_local: fdb_delete_local(br, p, f) (per-port-mac restore).
4. Else: kfree_rcu(f, rcu).
5. atomic_dec(&br.fdb_count).

`BridgeFdb::update(br, source, addr, vid, fastpath)`:
1. /* MAC learning: source-MAC → port */
2. f = BridgeFdb::find_rcu(br, addr, vid).
3. If f:
   - if !f.is_static ∧ !f.is_local ∧ f.dst != source:
     - WRITE_ONCE(f.dst, source).
     - f.notify = true.
   - WRITE_ONCE(f.used, jiffies).
4. Else:
   - if br.fdb_count >= br.fdb_max_learned: return.
   - f = BridgeFdb::fdb_create(br, source, addr, vid, false, false).
   - f.notify = true.

`BridgeFdb::age_out(work)`:
1. br = container_of(work, NetBridge, gc_work).
2. now = jiffies.
3. for f in &br.fdb_hash:
   - if f.is_static ∨ f.is_local: continue.
   - if (now - f.used) > br.ageing_time: BridgeFdb::fdb_delete(br, f, true).
4. queue_delayed_work(&br.gc_work, jiffies(1s)).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `entry_unique_per_addr_vid` | INVARIANT | per-(addr, vid): at most one entry. |
| `local_implies_no_dst` | INVARIANT | f.is_local ⟹ f.dst == None. |
| `static_persists_age` | INVARIANT | per-age-out: f.is_static ⟹ not deleted. |
| `count_matches_table` | INVARIANT | br.fdb_count == #entries in br.fdb_hash. |
| `find_rcu_under_rcu_read_lock` | INVARIANT | per-find_rcu: caller in RCU read-lock section. |

### Layer 2: TLA+

`net/bridge/br_fdb.tla`:
- Per-rx MAC learning + per-age cleanup + per-static persist.
- Properties:
  - `safety_no_concurrent_corrupt` — per-find_rcu sees consistent entry.
  - `safety_aged_out_only_dynamic` — per-aged-out entry: !is_static ∧ !is_local.
  - `safety_local_after_port_mac` — per-port-MAC + per-bridge-MAC: corresponding local entries.
  - `liveness_unused_eventually_aged` — per-non-static entry > ageing_time ⟹ deleted.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `BridgeFdb::update` post: entry exists with (addr, vid, port); used=jiffies | `BridgeFdb::update` |
| `BridgeFdb::find_rcu` post: returns Some ⟺ (addr, vid) entry exists | `BridgeFdb::find_rcu` |
| `BridgeFdb::fdb_create` post: entry in br.fdb_hash; count++ | `BridgeFdb::fdb_create` |
| `BridgeFdb::fdb_delete` post: entry removed; kfree_rcu queued | `BridgeFdb::fdb_delete` |
| `BridgeFdb::age_out` post: per-non-static-non-local-aged entry deleted | `BridgeFdb::age_out` |

### Layer 4: Verus/Creusot functional

`Per-(addr, vid, port) MAC learning + ageing → per-bridge forwarding-decision via FDB lookup` semantic equivalence: per-IEEE 802.1D bridge learning model.

## Hardening

(Inherits row-1 features from `net/bridge/br_input.md` § Hardening.)

FDB-specific reinforcement:

- **Per-(addr, vid) uniqueness via rhashtable** — defense against per-MAC duplicate corruption.
- **Per-fdb_count cap (fdb_max_learned)** — defense against per-bridge MAC-flood DoS.
- **Per-RCU-free** — defense against per-fast-path UAF.
- **Per-static persist across age** — defense against per-static accidentally aged.
- **Per-local on port MAC change handled** — defense against per-mac-change leaving stale local.
- **Per-ageing periodic timer** — defense against per-age O(n) walking starvation.
- **Per-rhashtable lookup O(1) avg** — defense against per-fdb-bomb.
- **Per-RTM_NEWNEIGH netlink CAP_NET_ADMIN** — defense against per-userspace fdb-spoof.
- **Per-switchdev offload synced** — defense against per-HW + SW divergence.
- **Per-VLAN-aware bridge: vid in lookup-key** — defense against per-VID confusion.
- **Per-update WRITE_ONCE on shared fields** — defense against per-RCU torn-read.

## Grsecurity/PaX-style Reinforcement

Baseline grsec/PaX posture inherited workspace-wide:

- **PAX_USERCOPY** — strict bounds on RTM_NEWNEIGH/RTM_GETNEIGH netlink attribute copies into `struct net_bridge_fdb_entry` shadow.
- **PAX_KERNEXEC** — `.rodata` `br_fdb_*` static-call targets and the rhashtable `params.obj_hashfn` / `params.obj_cmpfn`.
- **PAX_RANDKSTACK** — per-syscall randomisation on rtnetlink `NEIGHTBL` handler entries.
- **PAX_REFCOUNT** — saturating `refcount_t` on `struct net_bridge_fdb_entry`.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `net_bridge_fdb_entry` slab so dropped MACs don't linger.
- **PAX_UDEREF** — enforced isolation between netlink user-attr buffers and the in-kernel rhashtable insert path.
- **PAX_RAP / kCFI** — forward-edge CFI on rhashtable callbacks and the switchdev `br_switchdev_fdb_notify` indirect dispatch.
- **GRKERNSEC_HIDESYM** — FDB internal symbols withheld from non-CAP_SYSLOG kallsym readers.
- **GRKERNSEC_DMESG** — FDB warnings ("received packet on ... with own address ...") gated behind CAP_SYSLOG.

br_fdb-specific reinforcement:

- **`struct net_bridge_fdb_entry` PAX_REFCOUNT** — saturating against per-entry pin abuse across rcu-grace windows.
- **`/sys/class/net/<br>/brif/<port>` CAP_NET_ADMIN write-gate** — defense against unprivileged static-FDB injection.
- **RTM_NEWNEIGH from netlink restricted to CAP_NET_ADMIN** — defense against userspace MAC-table spoofing.
- **rhashtable insert capped per-port** — defense against per-port FDB-flood DoS (MAC-bomb).
- **VLAN-aware: `(addr, vid)` composite key strictly enforced** — defense against cross-VID FDB confusion + double-tag spoof.
- **Switchdev offload synced under rtnl + rcu** — defense against HW/SW table divergence racing concurrent unbridge.

Rationale: the bridge FDB is the address-of-record for L2 forwarding; an unprivileged or misbehaving fdb-injection vector can redirect traffic, exhaust hash buckets, or leave a dangling entry across rcu-grace. The grsec stack ensures privilege-gated mutation, slab-zeroing on free, and PAX_REFCOUNT-bounded per-entry holders.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Bridge ingress (covered in `br_input.md` Tier-3)
- Bridge forward (covered in `br_forward.md` Tier-3)
- Bridge multicast snoop (covered separately)
- Bridge STP (covered separately)
- rtnetlink (covered separately)
- Implementation code
