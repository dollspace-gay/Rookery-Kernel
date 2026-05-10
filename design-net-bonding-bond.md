---
title: "Tier-3: drivers/net/bonding/bond_main.c — Linux bonding driver (NIC link aggregation)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Linux bonding driver aggregates multiple physical NIC slaves under a single virtual `bondN` netdev. Per-slave: standard netdev with rx_handler hijacked. Per-bond modes: 0=round-robin (RR), 1=active-backup (failover), 2=XOR (per-skb-hash slave-pick), 3=broadcast (replicate to all), 4=802.3ad LACP (IEEE link-aggregation), 5=ALB (adaptive-load-balance), 6=TLB (transmit-load-balance). Per-MII / ARP-monitor periodically probes slave-link; failover on link-down. Per-LACP runs IEEE 802.3ad state machine. Critical for: server fault-tolerance + bandwidth aggregation; replaced largely by team driver but still widely deployed.

This Tier-3 covers `bond_main.c` (~6661 lines).

### Acceptance Criteria

- [ ] AC-1: `ip link add bond0 type bond mode 1 miimon 100`: bond0 created.
- [ ] AC-2: `ip link set eth0 master bond0`: enslave; rx_handler registered.
- [ ] AC-3: ACTIVEBACKUP + slave[0] link-up: TX via slave[0].
- [ ] AC-4: slave[0] link-down: bond_select_active_slave switches to slave[1].
- [ ] AC-5: ROUNDROBIN: 1000 packets distributed evenly across slaves.
- [ ] AC-6: 802.3AD: LACPDU exchange; aggregator formed.
- [ ] AC-7: bond_arp_rcv: ARP reply within timeout → link-up.
- [ ] AC-8: BROADCAST: skb cloned to all slaves.
- [ ] AC-9: ALB: per-flow rebalance via ARP-rewrite.
- [ ] AC-10: bond_release: rx_handler unregistered; IFF_SLAVE cleared.

### Architecture

Per-bond state:

```
struct Bonding {
  dev: *NetDev,
  params: BondParams,
  slave_list: ListHead<Slave>,
  curr_active_slave: AtomicPtr<Slave>,
  primary_slave: AtomicPtr<Slave>,
  current_arp_slave: *Slave,
  rr_tx_counter: AtomicU32,
  send_peer_notif: u8,
  ad_info: Ad8023Info,                            // LACP state
  alb_info: Option<AlbBondInfo>,
  tlb_info: Option<TlbBondInfo>,
  recv_probe: fn(skb, dev),
  ...
}

struct BondParams {
  mode: u8,                                       // BOND_MODE_*
  miimon: u32,                                    // ms
  updelay: u32,
  downdelay: u32,
  arp_interval: u32,
  arp_ip_target: [u32; BOND_MAX_ARP_TARGETS],
  primary: String,
  fail_over_mac: u8,
  ad_select: u8,                                  // 802.3AD aggregator policy
  lacp_fast: u8,
  ...
}

struct Slave {
  link: ListLink,
  dev: *NetDev,
  bond: *Bonding,
  delay: u32,
  link_state: u32,
  link_failure_count: u32,
  perm_hwaddr: [u8; ETH_ALEN],
  speed: u32,
  duplex: u8,
  ad_aggregator_info: Ad8023AggregatorInfo,
  ...
}
```

`Bond::create(net, ifname) -> Result<*NetDev>`:
1. dev = alloc_netdev(priv_size = sizeof(Bonding), ifname, NET_NAME_UNKNOWN, bond_setup).
2. err = register_netdev(dev).
3. Return dev.

`Bond::init(dev) -> Result<()>`:
1. bond = bond_priv(dev).
2. bond.dev = dev.
3. INIT_LIST_HEAD(&bond.slave_list).
4. bond.params = default-params.

`Bond::enslave(bond_dev, slave_dev, extack) -> Result<()>`:
1. bond = bond_priv(bond_dev).
2. /* Validate slave_dev not already slave; not bridge etc. */
3. Allocate Slave; init.
4. Per-mode init: ALB: bond_alb_init_slave; 802.3AD: bond_3ad_bind_slave.
5. err = netdev_rx_handler_register(slave_dev, bond_handle_frame, slave).
6. slave_dev.flags |= IFF_SLAVE.
7. dev_master_upper_dev_link(slave_dev, bond_dev, slave, NULL).
8. list_add(&slave.link, &bond.slave_list).
9. /* MAC assignment per fail_over_mac policy */
10. Bond::select_active_slave(bond) // active-backup mode picks a primary.

`Bond::release(bond_dev, slave_dev) -> Result<()>`:
1. bond = bond_priv(bond_dev); slave = bond_slave_get_rcu(slave_dev).
2. netdev_rx_handler_unregister(slave_dev).
3. dev_master_upper_dev_unlink(slave_dev, bond_dev).
4. slave_dev.flags &= ~IFF_SLAVE.
5. list_del(&slave.link).
6. /* Per-mode release: ALB / 802.3AD */
7. kfree(slave).

`Bond::handle_frame(pskb) -> RxHandlerResult`:
1. skb = *pskb; slave = bond_slave_get_rcu(skb.dev); bond = slave.bond.
2. mode = bond.params.mode.
3. /* Per-mode rx-filter */
4. match mode:
   - ACTIVEBACKUP: if slave != bond.curr_active_slave: drop unless multicast/promisc.
   - 802.3AD: if LACP-frame: bond_3ad_lacpdu_recv; CONSUMED.
   - ALB: bond_alb_arp_xmit / bond_alb_xmit (mac-rewrite for inbound balancing).
5. skb.dev = bond.dev.
6. Return RX_HANDLER_ANOTHER.

`Bond::xmit(skb, dev) -> NetdevTxT`:
1. bond = bond_priv(dev).
2. mode = bond.params.mode.
3. switch mode:
   - 0 RR: slave = slaves[(bond.rr_tx_counter.fetch_add(1)) % nr_active]; bond_dev_queue_xmit(slave, skb).
   - 1 AB: slave = bond.curr_active_slave; if !slave: drop; bond_dev_queue_xmit.
   - 2 XOR: hash = skb_xmit_hash(skb); slave = slaves[hash % nr_active].
   - 3 BCAST: for each slave: skb_clone + bond_dev_queue_xmit.
   - 4 8023AD: bond_3ad_xmit_xor.
   - 5/6: bond_alb_xmit / bond_tlb_xmit.

`Bond::mii_monitor(work)`:
1. bond = container_of(work, Bonding, mii_work).
2. for_each_slave(bond, slave):
   - speed_duplex = bond_check_dev_link(bond, slave.dev).
   - per-state transitions (UP/DOWN with debounce: updelay, downdelay).
3. If link change: Bond::select_active_slave.
4. Re-arm at miimon-interval.

`Bond::select_active_slave(bond)`:
1. best = bond_find_best_slave(bond)  // priority-ordered link-up.
2. if best != bond.curr_active_slave: Bond::change_active_slave(bond, best).

### Out of Scope

- net/core/dev (covered in `dev.md` Tier-3)
- 802.3AD LACP detail (covered in `bond_3ad.md` if added)
- ALB/TLB detail (covered in `bond_alb.md` if added)
- Team driver (separate modern alternative)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct bonding` | per-bond state | `Bonding` |
| `struct slave` | per-slave state | `Slave` |
| `bond_init()` | per-bond netdev init | `Bond::init` |
| `bond_create()` | per-bond netdev alloc | `Bond::create` |
| `bond_enslave()` | per-slave attach | `Bond::enslave` |
| `bond_release()` | per-slave detach | `Bond::release` |
| `bond_handle_frame()` | per-slave rx_handler | `Bond::handle_frame` |
| `bond_xmit()` | per-bond TX dispatcher | `Bond::xmit` |
| `bond_xmit_roundrobin()` / `_activebackup()` / `_xor()` / `_broadcast()` | per-mode TX | `Bond::xmit_*` |
| `bond_select_active_slave()` | per-bond active-slave selection | `Bond::select_active_slave` |
| `bond_change_active_slave()` | per-bond switch active | `Bond::change_active_slave` |
| `bond_mii_monitor()` | per-bond MII-link periodic check | `Bond::mii_monitor` |
| `bond_arp_rcv()` | per-bond ARP-monitor RX | `Bond::arp_rcv` |
| `bond_3ad_state_machine_handler()` | LACP state machine | `Bond::3ad_state_machine` |
| `bond_alb_init_slave()` | ALB per-slave init | `Bond::alb_init_slave` |
| `BOND_MODE_*` | mode IDs | UAPI |

### compatibility contract

REQ-1: Per-bond netdev (bondN):
- alloc_netdev with priv_size = sizeof(Bonding).
- bond.params: mode, miimon, arp_interval, downdelay, updelay, etc.

REQ-2: Modes (BOND_MODE_*):
- 0 ROUNDROBIN: per-skb cycles slaves.
- 1 ACTIVEBACKUP: only active slave TXes.
- 2 XOR: per-skb hash → slave.
- 3 BROADCAST: TX replicates to all slaves.
- 4 802.3AD: LACP-aggregated (IEEE 802.3ad/802.1AX).
- 5 BALANCE_TLB: per-slave TX-load-based.
- 6 BALANCE_ALB: per-slave RX-load-balanced via ARP-rewrite.

REQ-3: Per-enslave:
- netdev_rx_handler_register(slave.dev, bond_handle_frame, slave).
- slave.dev.flags |= IFF_SLAVE.
- bond.slave_list: add.
- Per-mode-specific init (ALB: bond_alb_init_slave; 802.3AD: bond_3ad_bind_slave).

REQ-4: Per-release:
- netdev_rx_handler_unregister(slave.dev).
- slave.dev.flags &= ~IFF_SLAVE.
- bond.slave_list: remove.

REQ-5: bond_handle_frame (rx-handler):
- Per-mode dispatch:
  - ACTIVEBACKUP: drop if slave != active (unless promisc allmulti rules).
  - 802.3AD LACP: per-LACP-frame: forward to LACP state-machine.
  - ALB: ARP-snoop, learn-MAC.
- Set skb.dev = bond.dev (for upper-stack).
- Return RX_HANDLER_ANOTHER (continue with bond as netdev) or RX_HANDLER_CONSUMED.

REQ-6: bond_xmit (per-bond TX):
- mode = bond.params.mode.
- match mode:
  - 0: bond_xmit_roundrobin: pick slave[(bond.rr_tx++) % nr_active].
  - 1: bond_xmit_activebackup: pick bond.curr_active_slave.
  - 2: bond_xmit_xor: hash = skb_xmit_hash; slave = slaves[hash % nr_active].
  - 3: bond_xmit_broadcast: skb_clone for each slave; tx all.
  - 4: bond_3ad_xmit_xor (LACP-aware XOR).
  - 5/6: bond_alb_xmit / bond_tlb_xmit.
- skb.dev = chosen-slave.dev; dev_queue_xmit.

REQ-7: bond_select_active_slave (active-backup):
- Per-link-down: bond_change_active_slave(bond, find_best_slave(bond)).
- Per-priority: highest-prio link-up slave.

REQ-8: bond_mii_monitor (periodic):
- Per-miimon-interval (default 100ms) timer.
- For each slave: read MII status (per-driver ndo_eth_ioctl).
- Per-link-state-change: bond_select_active_slave.

REQ-9: bond_arp_rcv (ARP-monitor):
- Per-arp_interval timer sends ARP probe to peer.
- Per-arp_target: per-slave probe.
- Link-up if ARP reply within deadline; else link-down.

REQ-10: 802.3AD LACP:
- Per-LACPDU exchange (every 30s slow / 1s fast).
- Per-aggregator: groups compatible slaves.
- bond_3ad_state_machine_handler: rx-machine, mux-machine, tx-machine.
- Per-aggregator selects active aggregation.

REQ-11: ALB / TLB:
- bond_alb_init_slave: per-slave hash-table for outgoing flows.
- TLB: per-skb hashes to slave by load.
- ALB: + ARP-rewrite to balance incoming flows.

REQ-12: Per-bond MAC:
- Default: inherit slave[0].dev_addr.
- Per-slave gets bond MAC (active-backup); or slave's own (round-robin if fail_over_mac=active).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mode_in_range` | INVARIANT | bond.params.mode ∈ [0, 6]. |
| `slave_unique_per_bond` | INVARIANT | per-slave on at most one bond.slave_list. |
| `active_slave_in_list` | INVARIANT | bond.curr_active_slave ∈ bond.slave_list ∨ NULL. |
| `iff_slave_set_iff_enslaved` | INVARIANT | slave.dev.flags & IFF_SLAVE ⟺ slave on bond.slave_list. |
| `mii_monitor_periodic` | INVARIANT | per-armed mii_work fires at miimon-interval. |

### Layer 2: TLA+

`drivers/net/bonding/bond.tla`:
- Per-mode TX dispatch + per-link-monitor + per-LACP state machine.
- Properties:
  - `safety_no_xmit_after_release` — per-released slave: no TX dispatched.
  - `safety_active_unique_in_ab` — ACTIVEBACKUP: at most one curr_active_slave.
  - `safety_lacp_partner_negotiated` — 802.3AD aggregator: per-partner-DEFAULT eventually OPERATIONAL.
  - `liveness_link_down_eventual_failover` — slave[0] down + slave[1] up ⟹ active=slave[1].

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Bond::enslave` post: slave on list; rx_handler registered; IFF_SLAVE set | `Bond::enslave` |
| `Bond::release` post: slave removed; rx_handler unregistered | `Bond::release` |
| `Bond::handle_frame` post: skb.dev = bond.dev (or CONSUMED) | `Bond::handle_frame` |
| `Bond::xmit` post: skb sent on per-mode-selected slave OR cloned | `Bond::xmit` |

### Layer 4: Verus/Creusot functional

`Per-bond {mode, slave-list, monitor-policy} → per-skb routed to correct slave; per-link-event triggers active-slave update` semantic equivalence: per-Linux bonding(7) man page.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

Bonding-specific reinforcement:

- **Per-mode validated** — defense against per-config invalid mode.
- **Per-rx-handler unregistered atomically with list-remove** — defense against per-rx UAF on slave-detach.
- **Per-IFF_SLAVE flag enforced** — defense against per-slave doubly-enslaved.
- **Per-mii-monitor debounce (updelay/downdelay)** — defense against per-link-flap thrashing.
- **Per-LACP timeout: slow=30s/fast=1s** — defense against per-LACP partner stuck.
- **Per-ALB ARP-rewrite serialized** — defense against per-flow inconsistent MAC.
- **Per-bond_release wait-for-RCU** — defense against per-rx in-flight using freed slave.
- **Per-fail_over_mac configurable** — defense against per-mac-spoof when bonded slave becomes inactive.
- **Per-broadcast-mode skb_clone bounded** — defense against per-bond unbounded clones.
- **Per-roundrobin-counter atomic** — defense against per-counter race causing duplicate slave-pick.

