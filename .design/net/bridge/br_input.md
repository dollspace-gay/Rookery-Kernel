# Tier-3: net/bridge/br_input.c — Linux bridge ingress path (RX → forward / local-up / STP)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/bridge/br_input.c (~485 lines)
  - net/bridge/br_private.h
  - include/uapi/linux/if_bridge.h
-->

## Summary

Per-bridge port (slave) RX-handler `br_handle_frame` is invoked from `__netif_receive_skb` per-skb arriving on a bridge-port netdev. Per-skb classified: BPDU (forwarded to STP), local-MAC (passed up to bridge-master netdev), unicast/multicast (forwarded to other ports per FDB lookup). Per-frame: VLAN-aware filter via `br_handle_ingress_vlan_tunnel`, FDB src-MAC learning via `br_fdb_update`, multicast snooping (IGMP/MLD) via `br_multicast_rcv`. Per-handler integrates with netfilter pre-routing chain. Critical for: in-kernel L2 bridging, container networking, virt bridge, STP/RSTP, MLD/IGMP-snooping.

This Tier-3 covers `br_input.c` (~485 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `br_handle_frame()` | per-skb rx_handler entry | `BridgeIn::handle_frame` |
| `br_handle_frame_finish()` | post-NF-PREROUTING hook | `BridgeIn::handle_frame_finish` |
| `br_handle_local_finish()` | per-local-MAC up-pass finalize | `BridgeIn::handle_local_finish` |
| `br_pass_frame_up()` | per-skb local up-pass | `BridgeIn::pass_frame_up` |
| `br_handle_ingress_vlan_tunnel()` | per-VLAN-tunnel rx | `BridgeIn::handle_ingress_vlan_tunnel` |
| `br_fdb_update()` | per-src-MAC learning | `Br::fdb_update` |
| `br_multicast_rcv()` | per-multicast snoop | `Br::multicast_rcv` |
| `br_should_route_hook` | per-skb route-decision hook | `Br::should_route_hook` |
| `br_handle_frame_dummy` | per-MRP-only port handler | `BridgeIn::handle_frame_dummy` |
| `is_link_local_ether_addr()` | per-MAC link-local 01:80:c2:00:00:0X check | helper |
| `BR_INPUT_SKB_CB(skb)` | per-skb bridge metadata | `BridgeSkbCb` |
| `nbp` | net_bridge_port | `NetBridgePort` |
| `port->state` | STP state (DISABLED/BLOCKING/LISTENING/LEARNING/FORWARDING) | shared |

## Compatibility contract

REQ-1: Per-port RX handler:
- `netdev_rx_handler_register(slave_dev, br_handle_frame, port)` at port-add.
- Per-skb received on slave: `br_handle_frame(&skb)` invoked.
- Returns: RX_HANDLER_CONSUMED (taken by bridge) or RX_HANDLER_PASS (continue normal).

REQ-2: Per-skb classification flow:
- Strip eth-header peek; check for VLAN tag.
- Per-link-local MAC (01:80:c2:00:00:0X): per-protocol BPDU/PAUSE/etc.
  - 01:80:c2:00:00:00 (STP): forward to br_stp_rcv unless STP disabled.
  - 01:80:c2:00:00:0E (LLDP): per-port group_fwd_mask check.
- Per-non-link-local: enter forwarding logic.

REQ-3: Per-port-state gate:
- DISABLED / BLOCKING: drop.
- LISTENING / LEARNING: allow only BPDU; don't forward.
- FORWARDING: full processing.

REQ-4: br_handle_frame_finish (post-NF):
- VLAN ingress check; drop if VLAN filtering blocks.
- FDB src-MAC update (learning).
- DST: lookup FDB by dst-MAC.
  - If hit: forward to that port via `br_forward`.
  - If miss (unknown unicast):
    - flood (per-port flag + ageing).
  - If multicast: br_multicast_rcv + flood-to-multicast-group.
  - If broadcast: flood-all-ports + pass-up if PROMISC.

REQ-5: br_pass_frame_up:
- Per-bridge-master netdev (br0): re-inject skb up via `netif_receive_skb_core`.
- Sets skb.dev = bridge-master.

REQ-6: Per-NF integration:
- NF_HOOK_LIST(NFPROTO_BRIDGE, NF_BR_PRE_ROUTING, ..., br_handle_frame_finish).
- Per-iptables-bridge rules applied.

REQ-7: Per-FDB-update (learning):
- Per-rx_skb on FORWARDING port:
  - br_fdb_update(br, p, eth.src, vid, 0).
  - FDB[(src_mac, vid)] = port; updated/refreshed.

REQ-8: Per-multicast-snoop:
- IGMP/MLD snooping hooks per-skb.
- Per-multicast group → port-list (mdb).

REQ-9: Per-VLAN ingress:
- br_handle_ingress_vlan_tunnel: per-port VLAN-tunnel rx (e.g. VxLAN ingress).
- Per-VLAN-aware bridge: filters per-VID allowed.

REQ-10: Per-frame TC handling:
- Per-bridge tc filters applied if present.

## Acceptance Criteria

- [ ] AC-1: Add port to br0: br_handle_frame registered as rx_handler.
- [ ] AC-2: Per-bridge-master netdev br0 sees frames bridged from ports.
- [ ] AC-3: Per-STP BPDU on port: br_stp_rcv invoked.
- [ ] AC-4: Per-port BLOCKING state: non-BPDU dropped.
- [ ] AC-5: Per-FORWARDING + unicast known-MAC: forwarded to dst port only.
- [ ] AC-6: Per-FORWARDING + unicast unknown: flooded.
- [ ] AC-7: Per-multicast-snoop + IGMP join: subsequent multicast to that group only forwarded to joining port.
- [ ] AC-8: Per-VLAN-aware bridge: ingress non-allowed VID dropped.
- [ ] AC-9: Per-FDB learning: src-MAC + ingress-port + vid recorded.
- [ ] AC-10: Per-NF-PREROUTING bridge rule: applied.

## Architecture

Per-skb bridge metadata:

```
struct BridgeSkbCb {
  brdev: *NetDev,
  src: *NetBridgePort,
  vlan_filtered: bool,
  ...
}
```

Per-port state:

```
struct NetBridgePort {
  br: *NetBridge,
  dev: *NetDev,
  state: u8,                                     // BR_STATE_*
  flags: u32,                                    // BR_LEARNING / BR_FLOOD / ...
  port_no: u16,
  group_fwd_mask: u16,
  vlgrp: *NetPortVlans,                          // VLAN filter
  ...
}
```

Per-bridge state:

```
struct NetBridge {
  ports: ListHead<NetBridgePort>,
  fdb_hash: HashTable<NetBridgeFdb>,
  mdb: NetBridgeMcastDb,
  vlan_proto: u16,
  vlan_filtering: bool,
  multicast_router: u8,
  ...
}
```

`BridgeIn::handle_frame(pskb) -> RxHandlerResult`:
1. skb = *pskb.
2. p = br_port_get_rcu(skb.dev).
3. If !p: return RX_HANDLER_PASS.
4. dest = eth_hdr(skb).h_dest.
5. If is_link_local_ether_addr(dest):
   - per-protocol filtering (STP, PAUSE, LLDP, etc.).
   - per-fwd_mask check.
   - if BPDU: br_do_proxy_suppress / br_handle_local_finish / passthrough.
6. Per-port-state:
   - BR_STATE_FORWARDING: per-handle ingress vlan tunnel; src-MAC learning; NF_HOOK pre-routing → br_handle_frame_finish.
   - BR_STATE_LEARNING: per-FDB-update only; drop frame (or pass up).
   - BR_STATE_DISABLED / BLOCKING: drop.
7. Return RX_HANDLER_CONSUMED.

`BridgeIn::handle_frame_finish(net, sk, skb) -> i32`:
1. p = BR_INPUT_SKB_CB(skb).src.
2. br = p.br.
3. If !br_allowed_ingress(br, p, skb, &vid): kfree_skb; return.
4. unicast = !is_multicast_ether_addr(eth.h_dest).
5. If unicast:
   - dst = br_fdb_find_rcu(br, eth.h_dest, vid).
   - If dst:
     - if dst.is_local: br_pass_frame_up(skb, ...); return.
     - br_forward(dst.dst, skb, ...).
   - Else (unknown unicast):
     - br_flood(br, skb, ..., BR_PKT_UNICAST, ...).
6. Else (multicast):
   - br_multicast_rcv(...).
   - mdst = br_mdb_get(br, skb, vid).
   - If mdst: br_multicast_flood(mdst, skb).
   - Else: br_flood(br, skb, ..., BR_PKT_MULTICAST, ...).
7. If broadcast: br_pass_frame_up(skb, true) + br_flood.
8. br_fdb_update(br, p, eth.h_source, vid, 0).
9. Return 0.

`BridgeIn::pass_frame_up(skb, promisc)`:
1. brdev = BR_INPUT_SKB_CB(skb).brdev.
2. skb.dev = brdev.
3. dev_sw_netstats_rx_add(brdev, skb.len).
4. NF_HOOK(NFPROTO_BRIDGE, NF_BR_LOCAL_IN, ..., netif_receive_skb).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `port_state_in_range` | INVARIANT | port.state ∈ BR_STATE_*. |
| `local_finish_only_for_local` | INVARIANT | per-local-up-pass: skb.dev == brdev. |
| `forward_only_in_forwarding` | INVARIANT | per-forwarded skb: source-port state == FORWARDING. |
| `link_local_filter_strict` | INVARIANT | per-link-local MAC: gw_fwd_mask checked before forward. |
| `vlan_ingress_validated` | INVARIANT | per-skb-with-vlan: br_allowed_ingress called. |

### Layer 2: TLA+

`net/bridge/br_input.tla`:
- Per-port-state + per-skb classify + per-FDB hit/miss + per-NF hook.
- Properties:
  - `safety_no_forward_in_blocking` — port.state ∈ {DISABLED, BLOCKING} ⟹ no forward.
  - `safety_fdb_learning_when_forwarding` — per-FORWARDING port: FDB updated for src-MAC.
  - `safety_link_local_per_fwd_mask` — per-link-local MAC: forwarded iff in fwd_mask.
  - `liveness_per_fdb_eventually_aged` — per-fdb-entry expires after fdb_age.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `BridgeIn::handle_frame` post: returns CONSUMED iff per-bridge-port handled | `BridgeIn::handle_frame` |
| `BridgeIn::handle_frame_finish` post: per-classified skb forwarded/upped/dropped per-spec | `BridgeIn::handle_frame_finish` |
| `BridgeIn::pass_frame_up` post: skb.dev = brdev; netif_receive invoked | `BridgeIn::pass_frame_up` |

### Layer 4: Verus/Creusot functional

`Per-port skb classified per-(STP-state, dst-MAC, VID) → forward or up-pass or drop` semantic equivalence: per-IEEE 802.1D bridge spec.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

Bridge-input-specific reinforcement:

- **Per-state enforced before forward** — defense against BLOCKING-port leak.
- **Per-link-local fwd_mask gate** — defense against BPDU bypass.
- **Per-FDB rcu-rd-lock during lookup** — defense against per-port-mutate UAF.
- **Per-NF-PREROUTING applied** — defense against per-rule bypass.
- **Per-VLAN ingress vid validation** — defense against unauthorized VID.
- **Per-multicast snoop scoped per-mdb** — defense against per-multicast flood beyond joiners.
- **Per-source-MAC learning rate-limited** — defense against MAC-address-flood DoS.
- **Per-promisc up-pass only when set** — defense against unintended local-up-pass.
- **Per-bridge-master xmit serialized via per-CPU stats** — defense against per-stats race.
- **Per-skb-cb size bound** — defense against per-skb metadata overflow.
- **Per-handler RCU-protected port lookup** — defense against per-port-add/remove race.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/bridge/br_forward.c (covered separately)
- net/bridge/br_fdb.c (covered separately)
- net/bridge/br_stp.c (covered separately)
- net/bridge/br_multicast.c (covered separately)
- Implementation code
