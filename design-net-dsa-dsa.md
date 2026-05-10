---
title: "Tier-3: net/dsa/dsa.c — DSA (Distributed Switch Architecture) framework"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

DSA (Distributed Switch Architecture) is Linux's per-embedded-Ethernet-switch framework. Per-switch chip (e.g. Marvell MV88E6xxx, Microchip KSZ-series, Broadcom BCM-58xxx) exposes:
- 1+ **CPU ports** (uplink to host SoC's NIC; "conduit"),
- N **user ports** (front-panel; per-port netdev `lan0..lanN`).
Per-skb to-user-port: kernel attaches a per-switch **DSA tag** (Marvell EDSA, BRCM B53, KSZ-NN, ocelot-8021q, etc.) and queues to conduit-NIC; switch HW routes per-tag to user-port. Per-skb-from user-port: switch HW prepends tag; conduit driver receives; DSA strips and forwards to per-user-port netdev. Per-switch may join a per-cluster "tree" of cascaded switches. Critical for: embedded routers (OpenWrt), industrial switches, datacenter aggregation.

This Tier-3 covers `dsa.c` (~1900 lines).

### Acceptance Criteria

- [ ] AC-1: DT parsed, switch detected: per-port netdevs created (lan0, lan1, ...).
- [ ] AC-2: TX on lan0: tag prepended; skb on eth0 (conduit).
- [ ] AC-3: RX on eth0 with switch-tag: stripped; delivered to matching lan-N.
- [ ] AC-4: Add lan0 to br0: ndo_bridge_setlink informs switch HW.
- [ ] AC-5: VLAN filter: ndo_vlan_rx_add_vid propagates to HW.
- [ ] AC-6: STP state-change: .port_stp_state_set called on switch.
- [ ] AC-7: devlink-port: per-switch + per-port exposed.
- [ ] AC-8: Cascaded switches: per-DSA inter-switch port routing.
- [ ] AC-9: Per-namespace: nsA lanX invisible from nsB.
- [ ] AC-10: dsa_unregister_switch: per-port netdevs removed; tree cleaned.

### Architecture

Per-switch state:

```
struct DsaSwitch {
  ds: *Device,
  tree: *DsaSwitchTree,
  ops: *DsaSwitchOps,
  num_ports: u32,
  ports: Vec<DsaPort>,
  priv: *void,
  index: u32,                                    // index in tree
  setup: bool,
  ...
}

struct DsaSwitchTree {
  list: ListLink,
  index: u8,
  ports: ListHead<DsaPort>,
  refcount: AtomicI32,
  setup: bool,
  conduit_state_change: bool,
  rtable: [[s8; DSA_MAX_PORTS]; DSA_MAX_SWITCHES],  // per-(self, other) → port-no
  ...
}

struct DsaPort {
  list: ListLink,
  ds: *DsaSwitch,
  index: u32,
  type_: DsaPortType,                            // CPU / DSA / USER / UNUSED
  user_dev: Option<*NetDev>,                     // for USER
  conduit: Option<*NetDev>,                      // for CPU (linked NIC)
  bridge: Option<*BrideObject>,
  vlan_filtering: bool,
  ...
}
```

`Dsa::register_switch(ds) -> Result<()>`:
1. Validate ds.ops.
2. /* Find or create tree */
3. tree = dsa_tree_alloc(ds.index ∨ default).
4. ds.tree = tree.
5. tree.ports = ports-list-init.
6. /* Per-port walk */
7. for port in 0..ds.num_ports:
   - dsa_port_setup(&ds.ports[port]).
   - List_add(&port.list, &tree.ports).
8. err = Dsa::tree_setup(tree).

`Dsa::tree_setup(tree) -> Result<()>`:
1. dsa_tree_setup_cpu_ports.
2. dsa_tree_setup_switches.
3. dsa_tree_setup_ports.
4. dsa_tree_setup_conduit.
5. dsa_tree_setup_lags.
6. tree.setup = true.

`Dsa::tree_setup_default_cpu(tree) -> Result<()>`:
1. /* Pick first CPU port */
2. for port in tree.ports:
   - if port.type_ == CPU: tree.default_cpu_port = port; break.

`Dsa::tree_setup_conduit(tree) -> Result<()>`:
1. for cpu_port in tree.cpu_ports:
   - conduit = parse_DT_or_lookup-conduit-NIC.
   - cpu_port.conduit = conduit.
   - netdev_rx_handler_register(conduit, Dsa::switch_rcv, cpu_port).
   - conduit.dsa_ptr = cpu_port.

`Dsa::user_xmit(skb, dev) -> NetdevTxT`:
1. dp = dsa_user_to_port(dev).
2. /* Add tag bytes via per-protocol ops */
3. skb = dp.ds.tag_ops.xmit(skb, dev).
4. skb.dev = dp.cpu_dp.conduit.
5. dev_queue_xmit(skb).

`Dsa::switch_rcv(pskb) -> RxHandlerResult`:
1. skb = *pskb; conduit = skb.dev; cpu_port = conduit.dsa_ptr.
2. ds = cpu_port.ds.
3. /* Decode tag via per-protocol */
4. skb = ds.tag_ops.rcv(skb, conduit).
5. /* Per-protocol typically sets skb.protocol + skb.dev = user-port-netdev */
6. if !skb: return RX_HANDLER_CONSUMED  (drop).
7. nbp = skb.dev as user-port-netdev.
8. /* Per-user-port stats++ */
9. netif_rx(skb).
10. Return RX_HANDLER_CONSUMED.

### Out of Scope

- net/dsa/{port, slave, conduit, devlink}.c (covered separately if expanded)
- net/dsa/tag_*.c (per-protocol; covered separately)
- Per-switch driver (drivers/net/dsa/; covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct dsa_switch_tree` | per-tree of cascaded switches | `DsaSwitchTree` |
| `struct dsa_switch` | per-switch chip | `DsaSwitch` |
| `struct dsa_port` | per-(switch, port) | `DsaPort` |
| `struct dsa_switch_ops` | per-driver ops | `DsaSwitchOps` |
| `struct dsa_device_ops` | per-tag-protocol ops | `DsaDeviceOps` |
| `dsa_register_switch()` | per-driver register | `Dsa::register_switch` |
| `dsa_unregister_switch()` | per-driver unregister | `Dsa::unregister_switch` |
| `dsa_tree_setup()` | per-tree setup | `Dsa::tree_setup` |
| `dsa_tree_teardown()` | per-tree teardown | `Dsa::tree_teardown` |
| `dsa_tree_setup_default_cpu()` | per-tree CPU-port select | `Dsa::tree_setup_default_cpu` |
| `dsa_tree_setup_cpu_ports()` | per-tree CPU-port init | `Dsa::tree_setup_cpu_ports` |
| `dsa_tree_setup_switches()` | per-tree per-switch init | `Dsa::tree_setup_switches` |
| `dsa_tree_setup_conduit()` | per-tree conduit-NIC bind | `Dsa::tree_setup_conduit` |
| `dsa_user_xmit()` | per-skb TX from user-port | `Dsa::user_xmit` |
| `dsa_switch_rcv()` | per-skb RX on conduit | `Dsa::switch_rcv` |
| `dsa_tag_protocol_*` | per-tag dispatch | `DsaTagProtocol::*` |
| `dsa_switch_register_notifier()` | per-switch notifier | `Dsa::switch_register_notifier` |
| `DSA_TAG_PROTO_*` enum | per-tag protocol IDs | UAPI |

### compatibility contract

REQ-1: Per-driver dsa_switch:
- ops: DsaSwitchOps with per-port-setup, port-bridge, port-vlan, port-mdb, port-stp, etc.
- num_ports: total switch ports.
- ports: per-port DsaPort.

REQ-2: Per-port type:
- DSA_PORT_TYPE_UNUSED.
- DSA_PORT_TYPE_CPU: conduit (host-NIC).
- DSA_PORT_TYPE_DSA: inter-switch (cascade).
- DSA_PORT_TYPE_USER: front-panel; per-port netdev.

REQ-3: dsa_register_switch:
- Parse per-switch DT node (e.g. /soc/mdio/switch@1).
- Build DsaSwitchTree if tree-root; or join existing.
- Per-port walk: create dsa_port for each.
- Per-USER port: alloc netdev lanX.
- Per-CPU port: bind to conduit-NIC (eth0).

REQ-4: dsa_tree_setup_default_cpu:
- Per-tree selects one CPU port for upstream traffic.

REQ-5: Per-tag protocol:
- Each switch family supplies per-tag-protocol ID (DSA_TAG_PROTO_*).
- Per-TX: kernel prepends tag bytes; per-RX: kernel strips.
- Protocols: BRCM, EDSA (Marvell extended), KSZ8795, KSZ9477, OCELOT, OCELOT_8021Q, MTK, QCA, RTL4_A, RTL8_4, SJA1105, SJA1110.

REQ-6: Per-user-port netdev:
- ndo_start_xmit = dsa_user_xmit.
- ndo_set_rx_mode / set_mac_address propagates to switch HW.
- Per-port stats: rx_packets / tx_packets from switch counters via dsa_get_stats64.

REQ-7: dsa_user_xmit:
- 1. Add tag via DsaDeviceOps.xmit.
- 2. skb.dev = conduit-NIC.
- 3. dev_queue_xmit.

REQ-8: dsa_switch_rcv (per-conduit-RX hook):
- bound as rx_handler on conduit-NIC.
- 1. Decode tag via DsaDeviceOps.rcv.
- 2. Strip tag bytes.
- 3. skb.dev = lan<port> netdev.
- 4. netif_rx.

REQ-9: Per-bridge support:
- DSA user-ports can join Linux bridge.
- ndo_bridge_setlink / getlink invoked on switch HW.
- Per-FDB / per-MDB synced.

REQ-10: Per-VLAN:
- ndo_vlan_rx_add_vid / kill_vid propagates VID to switch HW filter.

REQ-11: Per-STP:
- Per-port-state propagated to switch via .port_stp_state_set.

REQ-12: Per-devlink:
- /sys/class/net/lanX/phys_switch_id: per-switch ID.
- devlink-port: per-(switch, port).

REQ-13: Per-namespace:
- Per-net DSA instance.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `port_type_in_set` | INVARIANT | per-port.type_ ∈ {UNUSED, CPU, DSA, USER}. |
| `cpu_port_has_conduit` | INVARIANT | per-port.type_ == CPU ⟹ port.conduit != NULL. |
| `user_port_has_netdev` | INVARIANT | per-port.type_ == USER ⟹ port.user_dev != NULL. |
| `tree_setup_only_once` | INVARIANT | per-tree.setup transition false→true once per dsa_tree_setup. |
| `tag_ops_non_null_at_xmit` | INVARIANT | per-user_xmit: ds.tag_ops != NULL. |

### Layer 2: TLA+

`net/dsa/dsa.tla`:
- Per-switch register + per-tree setup + per-skb tag insert/strip.
- Properties:
  - `safety_tag_round_trip` — per-TX skb tag-prepended + per-RX correct user-port-delivery.
  - `safety_no_user_port_to_user_port_direct` — per-skb routes via switch HW or conduit.
  - `liveness_per_register_completes` — per-driver register: dsa_tree_setup eventually succeeds.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Dsa::register_switch` post: ds in tree.ports; per-port type set | `Dsa::register_switch` |
| `Dsa::user_xmit` post: skb has tag; skb.dev == conduit | `Dsa::user_xmit` |
| `Dsa::switch_rcv` post: tag stripped; skb.dev = user-port | `Dsa::switch_rcv` |
| `Dsa::tree_setup_conduit` post: per-CPU-port has conduit; rx_handler registered | `Dsa::tree_setup_conduit` |

### Layer 4: Verus/Creusot functional

`Per-DSA switch: user-ports as virtual netdevs + conduit-NIC for upstream + per-switch tag protocol for steering` semantic equivalence: per-Documentation/networking/dsa/dsa.rst.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

DSA-specific reinforcement:

- **Per-port.type_ validated** — defense against per-misconfigured DT type.
- **Per-tag protocol fn-ptr null-check** — defense against per-driver tag-ops missing crash.
- **Per-rx_handler unregister at unregister-switch** — defense against per-rx UAF.
- **Per-bridge/VLAN/STP propagated only via ds.ops** — defense against per-config inconsistency between SW and HW.
- **Per-CPU-port refcount via dev_get** — defense against per-conduit UAF on detach.
- **Per-CAP_NET_ADMIN for bridge-add** — defense against per-unprivileged switch-config.
- **Per-DSA tree refcount** — defense against per-tree UAF.
- **Per-DSA inter-switch tag-routing checked** — defense against per-misrouted skb.
- **Per-namespace DSA scoped** — defense against cross-ns leak.
- **Per-skb_pull respects tag size** — defense against per-malformed-tag OOB.

