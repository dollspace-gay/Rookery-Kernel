# Tier-3: net/core/dev.c — netdev core (per-net_device lifecycle + RX/TX path + softnet + napi + dev_queue_xmit)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/core/dev.c
  - net/core/dev.h
  - include/linux/netdevice.h
  - include/linux/netdev_features.h
-->

## Summary

`net/core/dev.c` is the kernel-network device-layer entry point — every per-NIC `struct net_device` flows through it for: per-NIC registration via `register_netdevice` (creates /sys/class/net/), per-skb TX dispatch via `dev_queue_xmit` (selects per-CPU softnet, applies QoS, calls driver xmit fn), per-skb RX via `__netif_receive_skb_core` (per-NIC NAPI poll → protocol demux → upper-layer L3), per-CPU `softnet_data` for backlog + RPS + RFS, NAPI bridge for poll-based RX. ~13314 lines — second-largest network file (after sock.c). Critical for: every packet in/out of every NIC.

This Tier-3 covers `net/core/dev.c` (~13314 lines) — selective subset focusing on TX/RX hot path.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct net_device` | per-NIC descriptor | `net::core::dev::NetDevice` |
| `struct softnet_data` | per-CPU networking state | `SoftnetData` |
| `register_netdevice(dev)` | per-NIC register | `NetDevice::register` |
| `unregister_netdevice(dev)` | per-NIC unregister | `NetDevice::unregister` |
| `__dev_get_by_name(net, name)` | per-name lookup | `NetDevice::get_by_name` |
| `__dev_get_by_index(net, ifindex)` | per-ifindex lookup | `NetDevice::get_by_index` |
| `dev_queue_xmit(skb)` / `__dev_queue_xmit(skb, sb_dev)` | per-skb TX entry | `NetDevice::queue_xmit` |
| `__dev_xmit_skb(skb, q, dev, txq)` | per-queue dispatch | `NetDevice::xmit_skb` |
| `dev_direct_xmit(skb, queue_id)` | per-queue direct xmit (XDP) | `NetDevice::direct_xmit` |
| `netif_receive_skb(skb)` / `__netif_receive_skb(skb)` | per-skb RX entry | `NetDevice::receive_skb` |
| `__netif_receive_skb_core(...)` | per-skb protocol demux | `NetDevice::receive_skb_core` |
| `napi_schedule(napi)` | schedule NAPI poll | `Napi::schedule` |
| `napi_poll(napi, budget)` | per-NAPI poll | `Napi::poll` |
| `net_rx_action(softirq)` | NET_RX_SOFTIRQ entry | `Softnet::net_rx_action` |
| `net_tx_action(softirq)` | NET_TX_SOFTIRQ entry | `Softnet::net_tx_action` |
| `process_backlog(napi, budget)` | per-CPU backlog poll | `Softnet::process_backlog` |
| `enqueue_to_backlog(skb, cpu, qtail)` | per-skb enqueue to RPS-target | `Softnet::enqueue_to_backlog` |
| `dev_pick_tx(dev, skb, sb_dev)` | per-skb queue selection | `NetDevice::pick_tx` |
| `dev_open(dev)` / `dev_close(dev)` | per-NIC bring-up/down | `NetDevice::open` / `_close` |
| `netif_carrier_on(dev)` / `_off(dev)` | per-NIC link-state | `NetDevice::carrier_on` / `_off` |
| `netif_tx_wake_queue(...)` / `_stop_queue(...)` | per-tx-queue flow control | `Netif::tx_wake_queue` / `_tx_stop_queue` |
| `dev_hold(dev)` / `dev_put(dev)` | per-NIC refcount | `NetDevice::hold` / `_put` |
| `register_netdevice_notifier(nb)` | per-NIC change notifier | `NetdeviceNotifier::register` |
| `__skb_pad(skb, pad)` | per-skb padding | `Skb::pad` |

## Compatibility contract

REQ-1: Per-NIC `net_device`:
- `name` (interface name; e.g., "eth0").
- `ifindex` (per-namespace unique).
- `state` (FLAGS bitmap).
- `mtu` / `min_mtu` / `max_mtu`.
- `priv_flags`.
- `features` (per-NIC offload bitmap; NETIF_F_*: TSO, GSO, RXCSUM, etc.).
- `hard_header_len` / `min_header_len`.
- `mc` / `uc` / `dev_addrs` (multicast/unicast/dev addresses).
- `flags` (per-IFF: UP, BROADCAST, LOOPBACK, etc.).
- `gflags`.
- `tx_queue_len`.
- `_tx[]` / `_rx[]` (per-queue netdev_queue / netdev_rx_queue).
- `num_tx_queues` / `num_rx_queues`.
- `xdp_prog` (per-NIC XDP program).
- `qdisc` (root queueing discipline).
- `netdev_ops` (vtable: ndo_open, ndo_close, ndo_start_xmit, ndo_get_stats, etc.).
- `ethtool_ops` (ethtool vtable).
- `priv` (driver-private data; e.g., per-NIC ring-state).

REQ-2: Per-CPU `softnet_data`:
- `input_pkt_queue` (per-CPU SKB queue from drivers).
- `process_queue` (per-CPU SKB queue under processing).
- `output_queue` (per-CPU output SKB queue).
- `output_queue_tailp`.
- `completion_queue`.
- `cpu` (this CPU index).
- `dropped` (per-CPU counter).
- `time_squeeze`.
- `received_rps`.
- `flow_limit_count`.
- `rps_ipi_list` (cross-CPU IPI for RPS).
- `xfrm_backlog` (XFRM transform backlog).
- `napi_struct` (per-CPU process_backlog NAPI).

REQ-3: Per-skb TX flow (`dev_queue_xmit`):
1. dev := skb->dev.
2. q := dev_pick_tx(dev, skb, sb_dev): select per-tx-queue (XPS / hash / etc.).
3. txq := netdev_pick_tx_queue(dev, skb, sb_dev).
4. spinlock(txq.lock).
5. If qdisc: qdisc.enqueue(skb, qdisc, &to_free).
6. qdisc.qdisc_run.
7. unlock.

REQ-4: Per-driver `ndo_start_xmit(skb, dev)`:
- Per-NIC driver implements: posts skb to HW TX ring; returns NETDEV_TX_OK / _BUSY.
- KVM virtio_net / NIC drivers register here.

REQ-5: Per-skb RX flow:
1. driver IRQ handler → napi_schedule(napi).
2. NET_RX_SOFTIRQ runs net_rx_action.
3. napi_poll(napi, budget) calls per-driver poll fn.
4. Per-poll: dequeue from HW RX ring; build skb; netif_receive_skb(skb).
5. __netif_receive_skb_core: protocol-demux via ptype_base + ptype_specific.

REQ-6: NAPI scheduling:
- Per-NAPI struct: weight, poll_list, gro_list.
- NAPI starts in NAPI_STATE_SCHED; busy-poll until budget exhausted; rescheduled if more work.
- Per-CPU softirq drains per-CPU poll_list.

REQ-7: Per-CPU RPS (Receive Packet Steering):
- Per-NIC RX hash → per-CPU target.
- IPI between CPUs to deliver to target's softnet_data.
- Used for spreading network load.

REQ-8: Per-CPU RFS (Receive Flow Steering):
- Per-flow hash → per-CPU target based on application affinity.
- Per-flow rps_dev_flow records target.

REQ-9: Per-NIC features bitmap:
- NETIF_F_TSO: TCP Segmentation Offload.
- NETIF_F_GSO: Generic Segmentation Offload.
- NETIF_F_RXCSUM / _NOCACHE_COPY / _UFO / _IPV6_CSUM / etc.
- Per-skb features ⊆ NIC features.

REQ-10: XDP integration:
- Per-NIC xdp_prog: BPF program at NIC-driver ingress.
- Returns XDP_PASS / _DROP / _TX / _REDIRECT.
- Bypasses softnet stack for low-latency.

REQ-11: Per-NIC carrier state:
- carrier_on/_off: link-up/down.
- Drivers signal via netif_carrier_on(dev) / _off(dev).

REQ-12: Multi-queue + qdisc:
- Per-NIC multiple TX queues for SMP scaling.
- Per-queue qdisc for QoS (mq, fq, prio, htb, etc.).

## Acceptance Criteria

- [ ] AC-1: Boot system; ip link list shows registered devices; per-NIC ifindex unique.
- [ ] AC-2: ping 8.8.8.8: skb constructed via tcp/icmp; dev_queue_xmit succeeds; reply received via __netif_receive_skb_core.
- [ ] AC-3: Multi-queue TX: 4-queue NIC; XPS distributes per-CPU; per-queue lock isolates.
- [ ] AC-4: NAPI: per-NIC interrupt; napi_schedule; per-CPU softirq drains; budget enforced.
- [ ] AC-5: RPS: enable RPS on rx queue; flow-hash distributes to target CPU; cross-CPU IPI fires.
- [ ] AC-6: RFS: enable rps_flow_cnt; per-flow target CPU follows app placement.
- [ ] AC-7: XDP: attach BPF program; XDP_DROP packet at NIC ingress; skb never built.
- [ ] AC-8: Carrier state: cable pull → carrier_off; subsequent TX returns NETDEV_TX_OK but skb dropped.
- [ ] AC-9: register_netdevice_notifier: per-add/remove/up/down events delivered.
- [ ] AC-10: lftp performance: TSO/GSO offload reduces per-skb count vs disabled baseline.

## Architecture

`NetDevice`:

```
struct NetDevice {
  name: KString,
  ifindex: i32,
  state: AtomicU64,
  flags: AtomicU32,                            // IFF_UP / IFF_BROADCAST / etc.
  priv_flags: u32,
  features: NetdevFeatures,
  hw_features: NetdevFeatures,
  vlan_features: NetdevFeatures,
  mtu: u32,
  min_mtu: u32,
  max_mtu: u32,
  hard_header_len: u32,
  
  netdev_ops: KArc<NetDeviceOps>,
  ethtool_ops: KArc<EthtoolOps>,
  
  num_tx_queues: u32,
  real_num_tx_queues: u32,
  _tx: KVec<NetdevQueue>,
  
  num_rx_queues: u32,
  real_num_rx_queues: u32,
  _rx: KVec<NetdevRxQueue>,
  
  qdisc: AtomicPtr<Qdisc>,
  
  xdp_prog: AtomicPtr<BpfProg>,
  
  ip_ptr: AtomicPtr<InDev>,                    // IPv4 device data
  ip6_ptr: AtomicPtr<Inet6Dev>,
  
  pcpu_refcnt: PerCpu<i32>,
  
  link_watch_list: ListNode,
  
  ...
}

struct SoftnetData {
  output_queue: KArc<Sk_buff>,
  output_queue_tailp: KAtomicPtr<KArc<Sk_buff>>,
  completion_queue: KArc<Sk_buff>,
  cpu: u32,
  process_queue: SkbQueue,
  input_pkt_queue: SkbQueue,
  napi_struct: KArc<Napi>,
  flow_limit_count: u32,
  ...
}
```

`NetDevice::queue_xmit(skb)`:
1. dev := skb->dev.
2. txq := dev_pick_tx(dev, skb, sb_dev).
3. spinlock(txq.lock).
4. q := rcu_dereference_bh(dev.qdisc).
5. ret := q.enqueue(skb, q, &to_free).
6. qdisc_run(q): drain qdisc → ndo_start_xmit per-skb.
7. unlock.

`NetDevice::receive_skb_core(skb, &ret, ppt_prev)`:
1. orig_dev := skb->dev.
2. Per-protocol packet_type list (rcu_protected):
   - For each ptype matching skb->protocol:
     - ptype.func(skb, dev, ptype, orig_dev) → ETH_P_IP → ip_rcv → IP layer.
3. Return outcome.

`Softnet::net_rx_action(softirq)`:
1. budget := netdev_budget (~64).
2. start := jiffies.
3. While !empty(softnet_data.poll_list) && budget > 0:
   - napi := first(poll_list).
   - work := napi.poll(napi, weight).
   - If work < weight: napi_complete; remove from poll_list.
   - Else: leave on poll_list for next round.
   - budget -= work.

`Softnet::process_backlog(napi, quota)` (per-CPU NAPI for backlog):
1. While quota > 0 && !empty(input_pkt_queue):
   - skb := dequeue(input_pkt_queue).
   - __netif_receive_skb(skb).
   - quota--.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `dev_pcpu_refcnt_no_underflow` | INVARIANT | per-CPU dev refcnt summed ≥ 0; defense against orphan dev_put. |
| `txq_lock_serialized` | INVARIANT | per-tx-queue dispatch under txq.lock. |
| `napi_state_machine` | INVARIANT | per-NAPI state ∈ {OFF, SCHED, RUNNING}; transitions valid. |
| `softnet_input_queue_len_bounded` | INVARIANT | per-CPU input_pkt_queue.qlen ≤ netdev_max_backlog. |
| `xdp_prog_atomic_swap` | UAF | per-NIC xdp_prog updated via rcu_assign + synchronize_rcu. |

### Layer 2: TLA+

`net/core/dev_lifecycle.tla`:
- Per-NIC state ∈ {Unregistered, Registered, Up, Down, Unregistering}.
- Properties:
  - `safety_no_xmit_when_down` — dev_queue_xmit only valid when state == Up.
  - `safety_no_register_with_existing_name` — per-name uniqueness in netns.
  - `liveness_unregister_eventually_completes` — Unregistering → Unregistered after RCU sync + refcount drain.

`net/core/napi_poll.tla`:
- Per-NAPI state machine.
- Properties:
  - `safety_one_napi_per_cpu` — at most one NAPI of a struct active per-CPU at a time.
  - `liveness_pending_eventually_polled` — assuming softirq runs, pending NAPI polled.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `NetDevice::register` post: ifindex assigned; sysfs entry created; notifier chain notified | `NetDevice::register` |
| `NetDevice::queue_xmit` post: skb queued to qdisc OR NETDEV_TX_BUSY OR dropped | `NetDevice::queue_xmit` |
| `NetDevice::receive_skb_core` post: skb dispatched to per-protocol handler OR dropped | `NetDevice::receive_skb_core` |
| `Softnet::net_rx_action` post: budget consumed; per-NAPI work tracked | `Softnet::net_rx_action` |

### Layer 4: Verus/Creusot functional

`Per-skb: dev_queue_xmit eventually delivers to peer (via wire) iff carrier on + qdisc accepts + driver xmit succeeds` semantic equivalence: per-skb the (data, eventually-on-wire) preserved across stack.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

netdev-core-specific reinforcement:

- **Per-CPU refcnt scaling** — defense against single-counter contention on hot path.
- **RCU-protected dev lookup** — defense against unregister-during-lookup UAF.
- **Per-tx-queue lock isolated** — defense against single-lock bottleneck on multi-queue NIC.
- **Per-CPU softnet input queue cap** — defense against driver-flood OOM.
- **NAPI weight cap** — defense against single-NIC starving other NICs in softirq.
- **XDP rcu-protected swap** — defense against tx-during-update UAF.
- **Per-NIC features ⊆ hw_features** — defense against software-offload claiming HW capability.
- **Per-namespace netdev list** — defense against cross-netns leak.
- **Per-notifier-chain rcu-protected** — defense against notifier-add-during-walk UAF.
- **Per-NIC dev_open under rtnl_lock** — defense against concurrent up/down causing inconsistent state.
- **Per-driver ndo_start_xmit return validated** — defense against driver returning unsupported value.
- **carrier_off drop subsequent xmit** — defense against transmit-while-down.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-protocol RX (covered in `net/ipv4/` / `net/ipv6/` Tier-3s)
- struct sock (covered in `net/struct-sock.md` Tier-2)
- skbuff (covered in `net/skbuff.md` Tier-2)
- qdisc (covered in `net/sched/` future Tier-3)
- XDP (covered in `kernel/bpf/bpf-core.md` Tier-3 + `net/core/xdp.md` future Tier-3)
- ethtool (covered in `net/core/ethtool.md` future Tier-3)
- net_namespace (covered in `net/core/net_namespace.md` future Tier-3)
- Implementation code
