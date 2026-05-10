---
title: "Tier-3: net/bridge/br_forward.c — Linux bridge forwarding (per-port deliver + flood)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`br_forward.c` implements the bridge forwarding plane invoked from `br_input.c` after FDB lookup or multicast classification. Per-skb forward to single port (`br_forward`), flood to all forwarding ports (`br_flood`), or multicast-flood to MDB-resolved subscriber ports (`br_multicast_flood`). Per-port `should_deliver` gates per-port-state (FORWARDING) + VLAN-egress check + per-flag mask. Per-skb cloned for multi-port delivery; per-NF_BR_FORWARD chain integrated. Critical for: bridge per-port packet delivery + per-multicast subscriber routing.

This Tier-3 covers `br_forward.c` (~363 lines).

### Acceptance Criteria

- [ ] AC-1: br_forward to FORWARDING port: skb.dev = port.dev; NF_BR_FORWARD invoked.
- [ ] AC-2: should_deliver returns false for BLOCKING port: skb dropped.
- [ ] AC-3: should_deliver returns false for skb.dev == p.dev: no delivery (no echo).
- [ ] AC-4: br_flood to N forwarding ports: skb cloned (N-1) times; last consumed.
- [ ] AC-5: BR_PKT_UNICAST + port.flags & ~BR_FLOOD: not delivered.
- [ ] AC-6: BR_PKT_MULTICAST + BR_MCAST_FLOOD off: not flooded to that port.
- [ ] AC-7: VLAN-aware bridge: egress VLAN tag stripped/inserted per-port config.
- [ ] AC-8: NF_BR_FORWARD iptables drop: skb dropped.
- [ ] AC-9: deliver_clone OOM: original skb still consumed; no double-free.
- [ ] AC-10: switchdev offload: skb.offload_fwd_mark set; SW skips.

### Architecture

`BridgeFwd::should_deliver(p, skb) -> bool`:
1. If skb.dev == p.dev: return false.
2. If p.state != BR_STATE_FORWARDING: return false.
3. vid = br_vid_get(skb).
4. If !br_allowed_egress(p.br, p, skb, vid): return false.
5. Return true.

`BridgeFwd::__forward(to, skb, local_orig)`:
1. skb.dev = to.dev.
2. Vlan::dev_egress_set_tag(skb, to).  // VLAN tag mgmt.
3. nbp_switchdev_frame_mark_tx_fwd_offload(to, skb).
4. NF_HOOK(NFPROTO_BRIDGE, NF_BR_FORWARD, ..., BridgeFwd::forward_finish).

`BridgeFwd::forward(to, skb, local_rcv, local_orig)`:
1. If !BridgeFwd::should_deliver(to, skb): kfree_skb; return.
2. If local_rcv: BridgeFwd::deliver_clone(to, skb, local_orig).
3. Else: BridgeFwd::__forward(to, skb, local_orig).

`BridgeFwd::deliver_clone(prev, skb, local_orig) -> Result<()>`:
1. skb = skb_clone(skb, GFP_ATOMIC).
2. If !skb: per-stat tx_dropped; return Err(NoMem).
3. BridgeFwd::__forward(prev, skb, local_orig).

`BridgeFwd::flood(br, skb, pkt_type, local_rcv, local_orig)`:
1. prev = None.
2. for port in br.port_list:
   - if !BridgeFwd::should_deliver(port, skb): continue.
   - flag = match pkt_type: UNICAST→BR_FLOOD; MULTICAST→BR_MCAST_FLOOD; BROADCAST→BR_BCAST_FLOOD.
   - if !(port.flags & flag): continue.
   - if let Some(prev_p) = prev: BridgeFwd::deliver_clone(prev_p, skb, local_orig).
   - prev = Some(port).
3. if let Some(prev_p) = prev:
   - if local_rcv: BridgeFwd::deliver_clone(prev_p, skb, local_orig).
   - else: BridgeFwd::__forward(prev_p, skb, local_orig).
4. else: kfree_skb.

`BridgeFwd::forward_finish(net, sk, skb) -> i32`:
1. skb_push(skb, ETH_HLEN).
2. dev_queue_xmit(skb).
3. Return 0.

### Out of Scope

- Bridge ingress (covered in `br_input.md` Tier-3)
- Bridge FDB (covered separately)
- Bridge multicast snoop (covered separately)
- Bridge STP (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `should_deliver()` | per-(port, skb) deliver-decision | `BridgeFwd::should_deliver` |
| `br_forward()` | per-port forward dispatcher | `BridgeFwd::forward` |
| `__br_forward()` | per-port low-level forward | `BridgeFwd::__forward` |
| `br_forward_finish()` | post-NF_BR_FORWARD finalize | `BridgeFwd::forward_finish` |
| `deliver_clone()` | per-skb clone-and-forward | `BridgeFwd::deliver_clone` |
| `br_flood()` | per-bridge flood to all forwarding ports | `BridgeFwd::flood` |
| `maybe_deliver_addr()` | per-skb deliver-with-mac-replace | `BridgeFwd::maybe_deliver_addr` |
| `nbp_switchdev_frame_mark_tx_fwd_offload()` | per-port HW-offload mark | `BridgeFwd::switchdev_mark` |

### compatibility contract

REQ-1: should_deliver(p, skb):
- Return false if !port-state == FORWARDING.
- Return false if br_allowed_egress(vlangrp, skb, vid) fails.
- Return false if skb.dev == p.dev (don't deliver back to source).
- Return true otherwise.

REQ-2: br_forward(to, skb, local_rcv, local_orig):
- If !should_deliver(to, skb): kfree_skb; return.
- If local_rcv (also pass-up): deliver_clone (skb cloned).
- Else: __br_forward (consume skb).

REQ-3: __br_forward(to, skb, local_orig):
- skb.dev = to.dev.
- nbp_switchdev_frame_mark_tx_fwd_offload(to, skb).
- NF_HOOK(NFPROTO_BRIDGE, NF_BR_FORWARD, ..., br_forward_finish).

REQ-4: br_forward_finish(net, sk, skb):
- skb_push(skb, ETH_HLEN) (re-add eth header).
- dev_queue_xmit(skb).

REQ-5: deliver_clone(prev, skb, local_orig):
- skb = skb_clone(skb, GFP_ATOMIC).
- If !skb: return -ENOMEM.
- __br_forward(prev, skb, local_orig).

REQ-6: br_flood(br, skb, pkt_type, local_rcv, local_orig):
- prev = NULL.
- For each port in br.port_list:
  - If !should_deliver(port, skb): continue.
  - If pkt_type == BR_PKT_UNICAST ∧ !(port.flags & BR_FLOOD): continue.
  - If pkt_type == BR_PKT_MULTICAST ∧ !(port.flags & BR_MCAST_FLOOD): continue.
  - If pkt_type == BR_PKT_BROADCAST ∧ !(port.flags & BR_BCAST_FLOOD): continue.
  - If prev != NULL: deliver_clone(prev, skb, local_orig).
  - prev = port.
- If prev != NULL:
  - If local_rcv: deliver_clone(prev, skb, local_orig).
  - Else: __br_forward(prev, skb, local_orig)  // consume.
- Else: kfree_skb.

REQ-7: br_multicast_flood (in br_multicast.c, calls br_forward):
- Per-mdb-entry: walk subscriber-port-list.
- For each subscriber port: __br_forward.

REQ-8: Per-VLAN-egress:
- br_handle_egress_vlan_tag(skb, p, vlangrp, vid) — strip/insert VLAN tag per egress port config.

REQ-9: Per-NF integration:
- NF_HOOK(NF_BR_FORWARD): per-iptables-bridge FORWARD chain.

REQ-10: Per-port flag BR_FLOOD / BR_MCAST_FLOOD / BR_BCAST_FLOOD:
- Per-port can be set to no-flood for unknown unicast / multicast / broadcast.
- E.g., `bridge link set dev port no_unicast_flood`.

REQ-11: Per-switchdev offload:
- nbp_switchdev_frame_mark_tx_fwd_offload sets skb.offload_fwd_mark if HW already forwarded.
- Avoids double-forward by HW + SW.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `no_self_delivery` | INVARIANT | should_deliver(p, skb): skb.dev != p.dev. |
| `state_forwarding_required` | INVARIANT | should_deliver: p.state == FORWARDING. |
| `last_port_consumes_or_clones` | INVARIANT | per-flood: last-port = consume; preceding = clone. |
| `flag_check_per_pkt_type` | INVARIANT | per-pkt-type flood: per-port flag enforced. |
| `nf_hook_invoked` | INVARIANT | per-forward: NF_HOOK(NF_BR_FORWARD) invoked before xmit. |

### Layer 2: TLA+

`net/bridge/br_forward.tla`:
- Per-port state + per-skb forward + per-flood walk.
- Properties:
  - `safety_no_back_to_source` — never deliver skb back to source-port.
  - `safety_per_flood_visits_each_eligible_port` — per-flood: every eligible port visited once.
  - `safety_skb_consumed_or_cloned` — per-skb: last-port consumes; preceding clones.
  - `liveness_eligible_port_eventually_delivered` — per-eligible port + non-NF-drop: skb delivered.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `BridgeFwd::should_deliver` post: returns true ⟺ all gates pass | `BridgeFwd::should_deliver` |
| `BridgeFwd::forward` post: skb to-port-delivered or dropped | `BridgeFwd::forward` |
| `BridgeFwd::flood` post: per-eligible-port skb delivered (clone or consume) | `BridgeFwd::flood` |
| `BridgeFwd::__forward` post: NF_BR_FORWARD called; skb.dev = to.dev | `BridgeFwd::__forward` |

### Layer 4: Verus/Creusot functional

`Per-bridge skb forward to per-port-or-flood respecting per-port state + flags + VLAN egress + NF chain` semantic equivalence: per-IEEE 802.1D forwarding model.

### hardening

(Inherits row-1 features from `net/bridge/br_input.md` § Hardening.)

Bridge-forward-specific reinforcement:

- **Per-port state checked before delivery** — defense against per-BLOCKING leak.
- **Per-self-port skip** — defense against per-skb-echo loop.
- **Per-VLAN egress tag mgmt** — defense against per-VLAN tag mismatch.
- **Per-NF-BR-FORWARD chain applied** — defense against per-rule bypass.
- **Per-flood per-port flag** — defense against per-config flood-disable bypass.
- **Per-skb_clone OOM handled** — defense against per-flood OOM partial-delivery.
- **Per-switchdev mark prevents double-forward** — defense against per-HW + SW double-deliver.
- **Per-multicast snoop scoped to MDB subscribers** — defense against per-multicast unbounded flood.
- **Per-port-add/remove RCU-safe** — defense against per-flood walking deleted port.
- **Per-skb_push(ETH_HLEN) bounds-checked** — defense against per-skb headroom OOB.

