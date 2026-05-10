# Tier-3: net/core/neighbour.c — neighbour-discovery (ARP / NDP table + per-interface state machine + GC)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/core/neighbour.c
  - include/net/neighbour.h
  - include/uapi/linux/neighbour.h
  - net/ipv4/arp.c (ARP-specific)
  - net/ipv6/ndisc.c (NDP-specific)
-->

## Summary

`neighbour` is the kernel's L3-to-L2 address-resolution table — IPv4 ARP + IPv6 NDP both share this single subsystem. Per-(interface, L3-addr) entry tracks: L2-address (MAC), state (incomplete/reachable/stale/probe/failed/permanent), use-counter, refresh timestamps. State machine drives ARP/NDP packet emission: on missed-LOOKUP → solicit (broadcast/multicast); on reply → update entry. Per-(table, interface) GC reclaims unused entries via two-tier (gc_thresh1/2/3 + gc_interval). Dispatched per-bcast interface change via netlink. Critical for: every routed L3 packet must resolve neighbour before transmit.

This Tier-3 covers `net/core/neighbour.c` (~3968 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct neigh_table` | per-protocol neighbour-table | `net::core::neighbour::NeighTable` |
| `struct neighbour` | per-(table, dev, addr) entry | `Neighbour` |
| `struct neigh_parms` | per-interface tunable | `NeighParms` |
| `struct neigh_ops` | per-protocol vtable (output, queue_xmit, etc.) | `NeighOps` |
| `struct pneigh_entry` | per-proxy-entry | `PneighEntry` |
| `neigh_create(tbl, pkey, dev, want_ref)` | create or find entry | `NeighTable::create_neigh` |
| `neigh_lookup(tbl, pkey, dev)` | lookup | `NeighTable::lookup` |
| `__neigh_lookup_noref(tbl, pkey, dev)` | lookup no-ref (for read-only) | `NeighTable::lookup_noref` |
| `neigh_release(neigh)` | drop ref | `Neighbour::release` |
| `neigh_destroy(neigh)` | destroy entry | `Neighbour::destroy` |
| `neigh_event_send(neigh, skb)` | output skb if ready | `Neighbour::event_send` |
| `neigh_resolve_output(neigh, skb)` | per-resolve output | `Neighbour::resolve_output` |
| `neigh_connected_output(neigh, skb)` | per-connected fast-path output | `Neighbour::connected_output` |
| `neigh_update(neigh, lladdr, new, flags, nlmsg_pid)` | state-update + lladdr-write | `Neighbour::update` |
| `neigh_update_links(neigh, ...)` | per-link adjacency update | `Neighbour::update_links` |
| `neigh_ifdown(tbl, dev)` | per-interface flush | `NeighTable::ifdown` |
| `neigh_table_clear(tbl)` | full flush | `NeighTable::clear` |
| `neigh_periodic_work(work)` | GC worker | `NeighTable::periodic_work` |
| `neigh_proxy_redo(...)` | per-proxy-entry retry | `NeighTable::proxy_redo` |
| `pneigh_lookup(tbl, ...)` | proxy lookup | `NeighTable::pneigh_lookup` |
| `pneigh_create(tbl, ...)` | proxy create | `NeighTable::pneigh_create` |
| `pneigh_delete(tbl, ...)` | proxy delete | `NeighTable::pneigh_delete` |
| `neigh_xmit(family, dev, &addr, skb)` | per-family transmit helper | `Neighbour::xmit` |
| `neigh_state` (enum) | NUD_*: NONE / INCOMPLETE / REACHABLE / STALE / DELAY / PROBE / FAILED / PERMANENT | `NudState` |

## Compatibility contract

REQ-1: Per-protocol `neigh_table`:
- `family` (AF_INET / AF_INET6 / AF_DECnet).
- `entry_size`.
- `key_len`.
- `protocol` (per-arp/ndisc).
- `hash` (KArc<NeighHashTable>; rcu-protected).
- `parms` (per-table baseline NeighParms).
- `gc_interval` / `gc_thresh1` / `gc_thresh2` / `gc_thresh3`.
- `last_flush` / `last_rand`.
- `phash_buckets` (proxy entries).
- `nht` (NeighHashTable wrapping hash).

REQ-2: Per-entry `neighbour`:
- `next` (chain to next entry in hash bucket).
- `tbl` (back-ref to neigh_table).
- `parms` (per-interface NeighParms).
- `confirmed` / `updated` / `used` (jiffies timestamps).
- `flags` (NTF_*: USE / SELF / MASTER / PROXY / EXT_LEARNED / OFFLOADED / etc.).
- `nud_state` (NUD_*).
- `type` (RTN_UNICAST / _MULTICAST / _BROADCAST / etc.).
- `dead` (under-destroy flag).
- `refcnt` (refcount_t).
- `arp_queue` (per-entry queued skbs awaiting resolution).
- `arp_queue_len_bytes`.
- `timer` (per-entry timer for state transitions).
- `ha` (hardware address, MAC).
- `ops` (NeighOps).
- `output` (cached output func: connected_output / resolve_output).
- `primary_key` (variable-length L3 address).

REQ-3: Per-interface `neigh_parms`:
- `dev` (back-ref to net_device).
- `data[NEIGH_VAR_DATA_MAX]` (per-tunable values: mcast_solicit, ucast_solicit, app_probe, retrans_time, base_reachable_time, delay_probe_time, gc_staletime, etc.).
- `tbl` (back-ref to neigh_table).
- `proxy_qlen` / `queue_len_bytes` (per-interface limits).

REQ-4: NUD states:
- NONE (0): Initial.
- INCOMPLETE: Solicited; waiting for reply.
- REACHABLE: Recently confirmed; usable.
- STALE: Possibly stale; usable but should re-probe on next use.
- DELAY: Transition before PROBE.
- PROBE: Sending unicast probe.
- FAILED: Resolution failed; report unreachable.
- PERMANENT (0x80): Static entry; no expiry.
- NOARP: No-ARP; e.g., loopback.

REQ-5: State transitions:
- NONE → INCOMPLETE (first packet to neighbour; mcast solicit).
- INCOMPLETE → REACHABLE (reply received).
- INCOMPLETE → FAILED (max_solicit exceeded).
- REACHABLE → STALE (base_reachable_time elapsed without traffic).
- STALE → DELAY (next packet sent).
- DELAY → PROBE (delay_probe_time elapsed).
- PROBE → REACHABLE (unicast reply).
- PROBE → FAILED (max_unicast_probes exceeded).

REQ-6: Per-skb output flow:
1. Caller invokes neigh.output(neigh, skb) (cached pointer):
   - REACHABLE: connected_output → dev_queue_xmit (fast).
   - INCOMPLETE/PROBE: resolve_output → queue skb on arp_queue + emit solicit.
   - FAILED: drop skb; -EHOSTUNREACH.

REQ-7: Per-entry GC:
- Periodic neigh_periodic_work walks tables.
- Entries past gc_staletime in REACHABLE/STALE removed.
- gc_thresh1: lower threshold; under = no GC.
- gc_thresh2: garbage-collect if over.
- gc_thresh3: hard cap; reject new entries.

REQ-8: Per-table hash:
- RCU-protected; per-bucket rcu-list.
- Bucket-count grows dynamically (4 → 1024).
- Hash function: jhash(pkey, key_len, base).

REQ-9: Per-entry timer:
- Per-state transitions driven by per-entry timer (hrtimer).
- Per-state retrans_time / base_reachable_time tunable.

REQ-10: Proxy entries (`pneigh_entry`):
- Per-entry: respond ARP/NDP solicit on behalf of another host.
- Used for: routers handling DHCP relay, proxy_arp config.

REQ-11: Per-rt_genid invalidation:
- Per-namespace ID; on routing-table change: bump.
- Stale neighbour cache entries detected via genid mismatch.

REQ-12: Live update via netlink:
- RTM_NEWNEIGH / RTM_DELNEIGH / RTM_GETNEIGH.
- userspace `ip neigh` tool.

## Acceptance Criteria

- [ ] AC-1: Boot with eth0; ping 192.168.1.1; entry created in INCOMPLETE; reply transitions to REACHABLE.
- [ ] AC-2: Stale-detection: REACHABLE entry idle for base_reachable_time; transitions to STALE.
- [ ] AC-3: Probe: STALE entry next-use → DELAY → PROBE → REACHABLE on reply.
- [ ] AC-4: Resolution failure: solicit no-reply; transitions to FAILED after max retries.
- [ ] AC-5: Permanent entry: ip neigh add 192.168.1.1 lladdr 00:11:22:33:44:55 nud permanent; survives GC.
- [ ] AC-6: GC: 100K entries; gc_thresh3 limit reached; new entries rejected with -ENOBUFS.
- [ ] AC-7: Proxy ARP: ip neigh add proxy 10.0.0.1 dev eth0; ARP request for 10.0.0.1 replied.
- [ ] AC-8: IPv6 NDP: per-IPv6-addr resolution; ICMPv6 neighbor solicitation/advertisement.
- [ ] AC-9: Live update: ip neigh change 192.168.1.1 lladdr X; entry MAC updated.
- [ ] AC-10: Multi-interface: per-interface entries distinct.

## Architecture

`NeighTable`:

```
struct NeighTable {
  family: u32,
  entry_size: u32,
  key_len: u32,
  protocol: u16,
  hash_func: NeighHashFn,
  hash_buckets: AtomicPtr<NeighHashTable>,
  parms: KArc<NeighParms>,
  parms_list: ListHead,
  gc_interval: u32,
  gc_thresh1: u32,
  gc_thresh2: u32,
  gc_thresh3: u32,
  last_flush: u64,
  last_rand: u64,
  stats: PerCpu<NeighStats>,
  phash_buckets: KBox<[Hlist; PNEIGH_HASHMASK + 1]>,
  proxy_redo: ProxyRedoFn,
  proxy_queue: SkbQueue,
  proxy_timer: TimerList,
  ...
}

struct Neighbour {
  next: AtomicPtr<Neighbour>,                  // RCU chain
  tbl: KArc<NeighTable>,
  parms: KArc<NeighParms>,
  confirmed: AtomicU64,                         // jiffies
  updated: u64,
  used: AtomicU64,
  flags: u32,
  nud_state: AtomicU8,
  type_: u8,
  dead: AtomicBool,
  refcnt: AtomicI32,
  arp_queue: SkbQueue,
  arp_queue_len_bytes: AtomicU32,
  timer: TimerList,
  ha: KBox<[u8; MAX_ADDR_LEN]>,
  ops: KArc<NeighOps>,
  output: NeighOutputFn,
  primary_key: KBox<[u8]>,                     // variable
}

struct NeighOps {
  family: u32,
  solicit: NeighSolicitFn,
  error_report: NeighErrorReportFn,
  output: NeighOutputFn,
  connected_output: NeighOutputFn,
}
```

`NeighTable::create(tbl, pkey, dev, want_ref)`:
1. Allocate neigh.
2. neigh.tbl = tbl.
3. memcpy(neigh.primary_key, pkey, tbl.key_len).
4. neigh.dev = dev.
5. neigh.parms = neigh_parms_clone(dev, tbl).
6. Init refcnt to want_ref ? 2 : 1 (lookup ref + caller ref if want_ref).
7. Initial state := NUD_NONE.
8. Init per-entry timer.
9. Insert into hash.

`NeighTable::lookup(tbl, pkey, dev)`:
1. hash := tbl.hash_func(pkey, tbl.key_len).
2. rcu_read_lock.
3. For neigh in tbl.hash_buckets[hash]:
   - If neigh.dev == dev && memcmp(neigh.primary_key, pkey, tbl.key_len) == 0 && !neigh.dead:
     - refcount_inc; rcu_read_unlock; return neigh.
4. rcu_read_unlock.
5. Return NULL.

`Neighbour::event_send(neigh, skb)`:
1. now := jiffies.
2. Switch neigh.nud_state:
   - NUD_REACHABLE: neigh.used = now; return 0; caller proceeds with output.
   - NUD_INCOMPLETE / _PROBE: queue skb on arp_queue; return 0 (deferred).
   - NUD_FAILED: drop skb; return -EHOSTUNREACH.
   - NUD_NONE: transition to INCOMPLETE; emit solicit; queue skb.

`Neighbour::resolve_output(neigh, skb)`:
1. ret := event_send(neigh, skb).
2. If ret == 0 && neigh.nud_state == REACHABLE:
   - dev_hard_header(skb, neigh.dev, ETH_P_*, neigh.ha, NULL, skb.len).
   - dev_queue_xmit(skb).
3. Else: skb queued or dropped.

`NeighTable::periodic_work` (GC):
1. now := jiffies.
2. nht := rcu_dereference(tbl.hash_buckets).
3. For each bucket:
   - For each neigh:
     - If !(neigh.flags & NTF_PERMANENT) && (now - neigh.used) > gc_staletime:
       - neigh.dead = 1; neigh_destroy.
4. Re-queue periodic_work after gc_interval.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `neigh_refcount_no_underflow` | INVARIANT | per-neigh refcnt ≥ 0; defense against double-release. |
| `nud_state_valid` | INVARIANT | state ∈ {NONE, INCOMPLETE, REACHABLE, STALE, DELAY, PROBE, FAILED, PERMANENT}. |
| `arp_queue_len_bounded` | INVARIANT | per-neigh.arp_queue_len_bytes ≤ tbl.queue_len_bytes; defense against unbounded queue. |
| `hash_bucket_no_uaf` | UAF | RCU protects per-bucket walk; defense against destroy-during-lookup. |
| `dead_excludes_lookup` | INVARIANT | neigh.dead implies subsequent lookup skips entry. |

### Layer 2: TLA+

`net/core/neigh_state.tla`:
- Per-entry state ∈ NUD_*.
- Transitions per timer / packet-receive / packet-send.
- Properties:
  - `safety_solicit_count_capped` — INCOMPLETE → FAILED after max_unicast_probes.
  - `safety_reachable_resolves_quickly` — REACHABLE state direct-output.
  - `liveness_solicit_eventually_resolves` — assuming peer responds, INCOMPLETE eventually REACHABLE.

`net/core/neigh_gc.tla`:
- Per-table GC walks.
- Properties:
  - `safety_permanent_not_evicted` — NTF_PERMANENT entries excluded from GC.
  - `safety_active_not_evicted` — recently-used entries skipped in GC.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `NeighTable::create` post: entry inserted; refcnt initialized; state == NUD_NONE | `NeighTable::create` |
| `Neighbour::event_send` post: skb queued OR proceeds with output OR dropped | `Neighbour::event_send` |
| `Neighbour::update` post: lladdr written; state transitioned per spec | `Neighbour::update` |
| Per-table hash + bucket count consistent across reorganization | invariants on rehash |

### Layer 4: Verus/Creusot functional

`Per-skb: dev_queue_xmit(skb) preceded by either neigh.connected_output (REACHABLE) OR neigh.resolve_output → queue+solicit` semantic equivalence: per-skb the L2-encap eventually correct iff peer responds.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

neighbour-specific reinforcement:

- **Per-table gc_thresh3 cap** — defense against attacker flooding ARP/NDP to exhaust kernel mem.
- **Per-entry arp_queue_len_bytes cap** — defense against per-host queue overflow.
- **NUD_PERMANENT excluded from GC** — defense against admin-config loss.
- **Per-table hash RCU + dynamic-resize** — defense against bucket-collision DoS.
- **Per-entry refcount under RCU** — defense against UAF on concurrent lookup + destroy.
- **Per-state max-solicit cap** — defense against perpetual ARP/NDP flooding.
- **Solicit rate-limit per-interface** — defense against amplification attack.
- **NTF_OFFLOADED for HW-offload entries** — defense against double-dispatch.
- **Per-rt_genid invalidation on routing-change** — defense against stale-cache bypass.
- **Per-netlink RTM_NEWNEIGH privileged** — defense against unauthorized neighbour-table modification.
- **Per-namespace tables isolated** — defense against cross-netns leak.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- ARP-specific protocol (covered in `net/ipv4/arp.md` future Tier-3)
- NDP-specific protocol (covered in `net/ipv6/ndisc.md` future Tier-3)
- Per-interface net_device (covered in `net/00-overview.md`)
- struct sock (covered in `net/struct-sock.md` Tier-2)
- Netlink (covered in `net/netlink/` future Tier-3)
- Implementation code
