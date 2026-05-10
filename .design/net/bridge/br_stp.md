# Tier-3: net/bridge/br_stp.c — Linux bridge IEEE 802.1D Spanning Tree Protocol (STP/RSTP)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/bridge/br_input.md
upstream-paths:
  - net/bridge/br_stp.c (~713 lines)
  - net/bridge/br_stp_bpdu.c (~247 lines)
  - net/bridge/br_stp_if.c
  - net/bridge/br_stp_timer.c
  - net/bridge/br_private_stp.h
-->

## Summary

Linux bridge implements IEEE 802.1D STP (default) and 802.1w RSTP. Per-bridge runs root-bridge election: lowest-priority root_id wins. Per-port: per-port-priority + per-port-cost compose path-cost to root. Per-port-state: DISABLED → BLOCKING → LISTENING → LEARNING → FORWARDING (or STP-off direct to FORWARDING). Per-BPDU received: process Configuration BPDU (or TCN). Per-timer: hello_timer, fdb_timer, tcn_timer. Per-topology-change: notifies via TCN BPDU. Critical for: loop-prevention in multi-bridge L2 networks; userspace alternative via mstpd.

This Tier-3 covers `br_stp.c` (~713 lines) + `br_stp_bpdu.c` (~247 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct net_bridge_port` | per-port STP state | `NetBridgePort` |
| `struct bridge_id` | per-bridge ID (priority + MAC) | `BridgeId` |
| `br_stp_change_bridge_id()` | per-bridge ID-change recompute | `Stp::change_bridge_id` |
| `br_stp_recalculate_bridge_id()` | per-bridge re-elect | `Stp::recalculate_bridge_id` |
| `br_stp_change_state()` | per-port state transition | `Stp::change_state` |
| `br_received_config_bpdu()` | per-port RX Config-BPDU | `Stp::received_config_bpdu` |
| `br_received_tcn_bpdu()` | per-port RX TCN-BPDU | `Stp::received_tcn_bpdu` |
| `br_topology_change_detection()` | per-bridge TC announce | `Stp::topology_change_detection` |
| `br_topology_change_acknowledge()` | per-bridge TC ack | `Stp::topology_change_acknowledge` |
| `br_become_root_bridge()` | per-bridge root self-promote | `Stp::become_root_bridge` |
| `br_root_selection()` | per-bridge re-elect | `Stp::root_selection` |
| `br_send_config_bpdu()` | per-port TX Config-BPDU | `Stp::send_config_bpdu` |
| `br_send_tcn_bpdu()` | per-port TX TCN-BPDU | `Stp::send_tcn_bpdu` |
| `br_stp_recalculate_path_cost()` | per-port path-cost re-compute | `Stp::recalculate_path_cost` |
| `br_supersedes_port_info()` | per-bpdu compare | `Stp::supersedes_port_info` |
| `br_stp_timer_init()` | per-bridge/port timers | `Stp::timer_init` |
| `BR_STATE_*` | port state IDs | UAPI |

## Compatibility contract

REQ-1: Per-bridge BridgeId:
- 8 bytes: 2-byte priority (default 32768) + 6-byte MAC.
- Lowest-priority + lowest-MAC wins root election.

REQ-2: Per-port states:
- BR_STATE_DISABLED (0): port admin-down or STP-disabled.
- BR_STATE_LISTENING (1): listen for BPDUs (15s).
- BR_STATE_LEARNING (2): listen + learn FDB (15s).
- BR_STATE_FORWARDING (3): full forwarding.
- BR_STATE_BLOCKING (4): block frames (initial state).

REQ-3: Per-bridge fields:
- bridge_id: BridgeId.
- root_id: BridgeId (current root).
- root_path_cost: u32.
- root_port: u16 (port-no of designated root-port).
- max_age: u32 (default 20s).
- hello_time: u32 (default 2s).
- forward_delay: u32 (default 15s).
- bridge_max_age, bridge_hello_time, bridge_forward_delay: per-bridge config.
- topology_change: bool.
- topology_change_detected: bool.

REQ-4: Per-port fields:
- state: u8 (BR_STATE_*).
- port_id: u16 (priority + port_no).
- path_cost: u32 (10Gbps=20, 100Gbps=2, etc.).
- designated_root, designated_bridge: BridgeId.
- designated_cost, designated_port: u32, u16.

REQ-5: br_received_config_bpdu(p, bpdu):
- supersedes = br_supersedes_port_info(p, bpdu).
- If supersedes:
  - record_config_information(p, bpdu).
  - Update designated_root, etc.
  - If !is_root: br_root_selection(br).
- If !supersedes ∧ p.designated_port == self: send_config_bpdu(p).
- topology_change handling.

REQ-6: br_received_tcn_bpdu(p):
- If is_root: br_topology_change_detection(br).
- Else: br_send_tcn_bpdu(parent_root_port).

REQ-7: br_topology_change_detection(br):
- Per-root: ageing-time = forward_delay; topology_change=true.

REQ-8: br_root_selection(br):
- Pick lowest cost path-to-root across non-disabled non-blocking ports.
- Update root_port + root_path_cost + designated_*.

REQ-9: br_become_root_bridge(br):
- self.bridge_id == root_id; root_path_cost = 0.

REQ-10: Per-port-state transition:
- DISABLED → BLOCKING (port up).
- BLOCKING → LISTENING → LEARNING → FORWARDING (per forward_delay).
- Per-link-down: → DISABLED.
- Per-RX BPDU + supersedes: re-eval → FORWARDING/BLOCKING.

REQ-11: BPDU format (Config-BPDU 35 bytes):
- protocol_id (2): 0.
- protocol_version (1): 0=STP / 2=RSTP.
- bpdu_type (1): 0x00=Config / 0x80=TCN.
- flags (1): TC + TC-ACK.
- root_id (8).
- root_path_cost (4).
- bridge_id (8).
- port_id (2).
- message_age, max_age, hello_time, forward_delay (2 each).

REQ-12: Per-LLC encapsulation:
- BPDU sent with dst-MAC = 01:80:c2:00:00:00 (STP).
- LLC header: DSAP=0x42, SSAP=0x42, control=0x03 (UI).

REQ-13: STP daemon coordination:
- bridge.stp_enabled: 0=off / 1=on / 2=user-space (mstpd).
- When user-space mode: kernel only forwards BPDU to userspace; doesn't run state-machine.

REQ-14: Per-userspace ABI:
- bridge sysfs: /sys/class/net/<br>/bridge/{stp_state, root_id, root_port, hello_time, ageing_time, forward_delay, max_age, ...}.
- ip link bridge / brctl tools.

## Acceptance Criteria

- [ ] AC-1: bridge add + STP enable: per-port → BLOCKING.
- [ ] AC-2: 4-port bridge: lowest-priority/MAC = root; others compute path-cost.
- [ ] AC-3: Per-port state machine progresses: BLOCKING → LISTENING → LEARNING → FORWARDING (15s × 2 = 30s).
- [ ] AC-4: BPDU rx with lower root_id: triggers root re-election; topology_change.
- [ ] AC-5: Topology change: TCN BPDU sent toward root; ageing-time temporarily reduced.
- [ ] AC-6: Hello timer fires every 2s on root: send Config-BPDU on each forwarding port.
- [ ] AC-7: max_age timer expires: per-port BPDU stale → recompute.
- [ ] AC-8: bridge.stp_enabled=2 (userspace): kernel forwards BPDUs to mstpd; doesn't run sm.
- [ ] AC-9: Per-port config priority via sysfs: re-compute root.
- [ ] AC-10: Loop prevented: ports redundant link → BLOCKING.
- [ ] AC-11: Link down on root-port: re-elect root.

## Architecture

Per-port STP state:

```
struct NetBridgePort {
  ...
  state: u8,                                     // BR_STATE_*
  port_id: u16,                                  // priority(top4) + portno(low12)
  path_cost: u32,
  designated_root: BridgeId,
  designated_bridge: BridgeId,
  designated_cost: u32,
  designated_port: u16,
  forward_delay_timer: TimerList,
  hold_timer: TimerList,
  message_age_timer: TimerList,
  topology_change_ack: bool,
  config_pending: bool,
  ...
}
```

Per-bridge STP state:

```
struct NetBridge {
  ...
  bridge_id: BridgeId,
  root_id: BridgeId,
  root_path_cost: u32,
  root_port: u16,
  max_age: u32,                                  // jiffies
  hello_time: u32,
  forward_delay: u32,
  bridge_max_age: u32,
  bridge_hello_time: u32,
  bridge_forward_delay: u32,
  ageing_time: u32,
  topology_change: u8,
  topology_change_detected: u8,
  hello_timer: TimerList,
  tcn_timer: TimerList,
  topology_change_timer: TimerList,
  gc_timer: TimerList,
  stp_enabled: u8,                               // BR_NO_STP / BR_KERNEL_STP / BR_USER_STP
  group_addr: [u8; 6],                           // STP-mcast 01:80:c2:00:00:00
  ...
}
```

`Stp::supersedes_port_info(p, bpdu) -> bool`:
1. /* Compare per-IEEE 802.1D §8.6.1 */
2. cmp = (bpdu.root_id, bpdu.root_path_cost, bpdu.bridge_id, bpdu.port_id) lex-less than (p.designated_*).
3. Return cmp.

`Stp::received_config_bpdu(p, bpdu)`:
1. If !p: return.
2. If Stp::supersedes_port_info(p, bpdu):
   - Stp::record_config_information(p, bpdu).
   - Stp::root_selection(p.br).
   - If p.br.root_port == p.port_no:
     - br_topology_change_acknowledge(p.br).
     - Stp::send_tcn_bpdu(p) if topology_change.
3. Else if p.designated_port == self:
   - Stp::send_config_bpdu(p).

`Stp::root_selection(br)`:
1. min_cost = ~0; root_port = 0; root_id = br.bridge_id; cost_to_root = 0.
2. for p in br.port_list:
   - if p.state == DISABLED: continue.
   - cand_root = p.designated_root.
   - cand_cost = p.designated_cost + p.path_cost.
   - if (cand_root, cand_cost) less-than (root_id, cost_to_root):
     - root_id = cand_root; cost_to_root = cand_cost; root_port = p.port_no.
3. br.root_id = root_id.
4. br.root_path_cost = cost_to_root.
5. br.root_port = root_port.
6. for p in br.port_list:
   - Stp::set_designated_port_for(p).
7. for p:
   - if p.designated_port == self ∧ should-forward:
     - Stp::change_state(p, BR_STATE_FORWARDING) (after forward_delay × 2).
   - else: Stp::change_state(p, BR_STATE_BLOCKING).

`Stp::topology_change_detection(br)`:
1. If br.is_root:
   - br.topology_change = true.
   - mod_timer(&br.topology_change_timer, jiffies + br.bridge_max_age + br.bridge_forward_delay).
2. Else:
   - br.topology_change_detected = true.
   - mod_timer(&br.tcn_timer, jiffies + br.bridge_hello_time).

`Stp::send_config_bpdu(p)`:
1. Build Config-BPDU bytes.
2. skb_alloc; eth_hdr.dst = STP_MCAST 01:80:c2:00:00:00.
3. dev_queue_xmit on p.dev.

`Stp::send_tcn_bpdu(p)`:
1. Build TCN-BPDU (4 bytes).
2. dev_queue_xmit.

`Stp::change_state(p, new_state)`:
1. p.state = new_state.
2. /* Per-state transition triggers timer (re)arm */

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `state_in_range` | INVARIANT | per-port.state ∈ {DISABLED, LISTENING, LEARNING, FORWARDING, BLOCKING}. |
| `root_id_le_bridge_ids` | INVARIANT | root_id ≤ all-bridge-ids in path. |
| `root_path_cost_consistent` | INVARIANT | root_path_cost = sum-of-path-costs from self to root. |
| `topology_change_implies_timer` | INVARIANT | topology_change=true ⟹ topology_change_timer armed. |
| `bpdu_protocol_id_zero` | INVARIANT | per-RX BPDU: protocol_id == 0. |

### Layer 2: TLA+

`net/bridge/br_stp.tla`:
- Per-bridge root election + per-port state machine + BPDU exchange + topology change.
- Properties:
  - `safety_unique_root_per_segment` — per-network-segment: one root.
  - `safety_no_loop` — per-bridge state stable ⟹ no forwarding loop.
  - `liveness_per_topology_change_propagated` — per-link-event: topology change reaches root within bounded time.
  - `liveness_state_progression` — per-port LISTENING → LEARNING → FORWARDING within 2 × forward_delay.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Stp::supersedes_port_info` post: returns true iff bpdu strictly-less per-IEEE 802.1D | `Stp::supersedes_port_info` |
| `Stp::root_selection` post: br.root_id is min over all reachable bridge_ids | `Stp::root_selection` |
| `Stp::change_state` post: per-state-transition timers armed | `Stp::change_state` |
| `Stp::topology_change_detection` post: per-root sets topology_change | `Stp::topology_change_detection` |

### Layer 4: Verus/Creusot functional

`Per-bridge STP state machine + BPDU exchange → loop-free spanning tree` semantic equivalence: per-IEEE 802.1D-2004 / 802.1w STP/RSTP spec.

## Hardening

(Inherits row-1 features from `net/bridge/br_input.md` § Hardening.)

STP-specific reinforcement:

- **Per-BPDU protocol_id validated** — defense against per-malformed BPDU crashing parser.
- **Per-port state transitions atomic** — defense against per-state torn-update.
- **Per-userspace mode kernel doesn't run sm** — defense against per-mstpd kernel-conflict.
- **Per-CAP_NET_ADMIN for sysfs writes** — defense against unprivileged STP-config.
- **Per-RX BPDU CAP-checked at port** — defense against per-spoofed BPDU triggering root-flip.
- **Per-BPDU-flood threshold** — defense against per-BPDU-flood DoS (BPDU-guard equivalent).
- **Per-topology-change-timer bound** — defense against per-TC tracking unbounded.
- **Per-hello-timer bounded by max_age** — defense against per-hello floods.
- **Per-port path_cost validated** — defense against per-config invalid u32.
- **Per-root-port-flap rate-limit** — defense against per-spurious-flap recompute storm.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Bridge ingress (covered in `br_input.md` Tier-3)
- Bridge forwarding (covered in `br_forward.md` Tier-3)
- Bridge FDB (covered in `br_fdb.md` Tier-3)
- mstpd userspace (out-of-tree)
- 802.1s MSTP (covered separately)
- Implementation code
