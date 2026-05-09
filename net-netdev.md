---
title: "Tier-3: net/netdev — net_device, NAPI, RX/TX paths"
tags: ["design-doc", "tier-3", "net"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `struct net_device` (netdev) — the abstraction over a network interface. Owns netdev lifecycle (`alloc_netdev` → `register_netdev` → ops → `unregister_netdev` → `free_netdev`), the per-netdev RX/TX queue array, NAPI (New API — interrupt-mitigation polling framework), link-state watching (carrier up/down), per-CPU device statistics, hardware-multicast / unicast / VLAN address lists, and the master `dev_*` API consumed by every protocol layer.

Sub-tier-3 of `net/00-overview.md`. Every NIC driver implements `struct net_device_ops`; every received packet enters via netdev's RX path; every transmitted packet exits via TX. Userspace observes via `/sys/class/net/<dev>/`, ethtool, and ioctl(SIOCGIFNAME/SIFFLAGS/...).

### Requirements

- REQ-1: `struct net_device` first-cache-line + commonly-accessed fields layout-equivalent.
- REQ-2: `struct net_device_ops` function-pointer-table layout byte-identical so out-of-tree drivers work unchanged.
- REQ-3: `IFF_*` flag values byte-identical; `ARPHRD_*` device-type values byte-identical.
- REQ-4: netdev lifecycle: alloc → register (rtnl-locked; assigns ifindex) → use → unregister (rtnl-locked; tear down) → free. Identical state transitions per `enum reg_state`.
- REQ-5: Per-queue RX + TX arrays + per-queue NAPI semantics match upstream. NAPI weight + budget + complete behavior identical.
- REQ-6: Link-state watcher: carrier-on / carrier-off transitions emit `RTM_NEWLINK` notifications; `IFLA_OPERSTATE` carries the state. udev rules trigger on link state changes.
- REQ-7: Address lists: per-netdev `mc_list` (multicast) + `uc_list` (unicast) + VLAN list maintained per upstream; `dev_mc_add` / `dev_uc_add` / `__hw_addr_*` API byte-identical.
- REQ-8: Per-CPU statistics: `dev->tstats` + `dev->core_stats` aggregated identically; visible via `/sys/class/net/<dev>/statistics/*`.
- REQ-9: Devmem (DMA-buf-backed RX zerocopy): `SO_DEVMEM_LINEAR` / `SO_DEVMEM_DMABUF` socket-side; `ndo_xdp_*` driver-side. Identical.
- REQ-10: ethtool integration: `dev->ethtool_ops` vtable + ethtool ioctl + ethtool netlink. Cross-ref `net/ethtool.md` Tier-3.
- REQ-11: `/sys/class/net/<dev>/*` content format-identical for every documented attribute.
- REQ-12: rtnetlink wire format byte-identical: `iproute2`, `ip`, `ifconfig`, `bridge`, `tc` (cross-ref `net/traffic-control.md`) all work unmodified.
- REQ-13: TLA+ models: `models/net/napi_state.tla` (mandatory per `net/00-overview.md` Layer 2) proves NAPI's STATE_SCHED / STATE_NPSVC / STATE_DISABLE state machine never deadlocks. `models/net/rtnl_lock.tla` (mandatory) proves rtnl_lock's big-mutex semantics serialize all netdev mutations.
- REQ-14: XDP attachment ABI per upstream (cross-ref `net/bpf-net.md`).
- REQ-15: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct net_device` + `pahole struct net_device_ops` byte-identical first-cache-line layouts vs. upstream. (covers REQ-1, REQ-2)
- [ ] AC-2: `enum IFF_*` + `enum ARPHRD_*` values compile-time const-asserted to match upstream. (covers REQ-3)
- [ ] AC-3: A test allocates 1000 dummy netdevs via `register_netdev`; each gets a unique ifindex; `unregister_netdev` cleans up. (covers REQ-4)
- [ ] AC-4: An iperf3 RX test exercises NAPI poll budget; per-NAPI weight + completion behavior visible via tracepoints. (covers REQ-5)
- [ ] AC-5: A `ip link set <dev> up/down` test produces RTM_NEWLINK notifications with correct OPERSTATE; udev sees the events. (covers REQ-6)
- [ ] AC-6: A multicast-listener test (`IP_ADD_MEMBERSHIP`) populates dev->mc_list correctly. (covers REQ-7)
- [ ] AC-7: An iperf3 TCP load run reports correct RX/TX byte counts via `/sys/class/net/<dev>/statistics/{rx_bytes,tx_bytes}`. (covers REQ-8)
- [ ] AC-8: A devmem-tcp test (with hardware support) receives zero-copy frames; SCM_DEVMEM_DMABUF carries dmabuf handle. (covers REQ-9)
- [ ] AC-9: `ethtool -i <dev>`, `ethtool -k <dev>`, `ethtool -S <dev>` produce identical output vs. upstream. (covers REQ-10)
- [ ] AC-10: A diff of `cat /sys/class/net/lo/*` between Rookery and upstream is empty. (covers REQ-11)
- [ ] AC-11: `iproute2`'s `ip link show`, `ip addr show`, `ip -s link` produce identical output. (covers REQ-12)
- [ ] AC-12: `make tla` passes `models/net/napi_state.tla` and `models/net/rtnl_lock.tla`. (covers REQ-13)
- [ ] AC-13: An XDP-attached BPF program (e.g., a simple drop-program) attaches successfully via `bpf(BPF_LINK_CREATE, ...)`. (covers REQ-14)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-15)

### Architecture

### Rust module organization

- `kernel::net::netdev::NetDevice` — `struct net_device` wrapper
- `kernel::net::netdev::ops::NetDeviceOps` — net_device_ops vtable trait
- `kernel::net::netdev::lifecycle` — alloc / register / unregister / free
- `kernel::net::netdev::queue::{TxQueue, RxQueue}` — per-queue
- `kernel::net::netdev::napi::Napi` — NAPI poll framework
- `kernel::net::netdev::link::LinkWatch` — link-state watcher
- `kernel::net::netdev::addr_list::AddrList` — mc/uc/VLAN list
- `kernel::net::netdev::stats::DeviceStats` — per-CPU statistics
- `kernel::net::netdev::ioctl::DevIoctl` — SIOC*IFREQ ioctl handlers
- `kernel::net::netdev::devmem::DevMem` — DMA-buf-backed RX zerocopy
- `kernel::net::netdev::sysfs::SysfsNet` — /sys/class/net/ tree

### Locking and concurrency

- **`rtnl_lock`** (mutex): protects all netdev mutations + most rtnetlink ops. Big-mutex; held during register/unregister/setlink.
- **Per-netdev `addr_list_lock`** (spinlock): per-netdev address list manipulation.
- **Per-NAPI `state`** (atomic): NAPI state machine.
- **Per-queue `_xmit_lock`** (spinlock): TX path serialization (when not lockless).
- **`dev_base_lock`** (rwlock): per-netns netdev list (read fast path; write rare).
- **RCU read-side**: rtnetlink dump + most read paths.

TLA+ models cover napi_state + rtnl_lock per `net/00-overview.md` Layer 2.

### Error handling

- `Err(ENODEV)` — netdev not found
- `Err(EINVAL)` — bad arg / unsupported feature
- `Err(EBUSY)` — netdev already up / already registered
- `Err(EOPNOTSUPP)` — operation not supported by driver
- `Err(EAFNOSUPPORT)` — address family not supported
- `Err(ENOMEM)` — alloc failed
- `Err(ETXTBSY)` — XDP attach with conflicting attachment

### Out of Scope

- Per-driver NIC implementations (cross-ref `drivers/net/00-overview.md` Tier-3)
- ethtool detail (cross-ref `net/ethtool.md`)
- XDP detail (cross-ref `net/bpf-net.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| netdev core (alloc, register, lifecycle, RX/TX dispatch) | `net/core/dev.c` |
| netdev ioctl (SIOC*IFREQ family) | `net/core/dev_ioctl.c` |
| netdev API helpers | `net/core/dev_api.c` |
| Devmem (DMA-buf-backed RX zerocopy) | `net/core/devmem.c` |
| Link-state watcher | `net/core/link_watch.c` |
| Hot-data extraction | `net/core/hotdata.c` |
| Device address lists (mc + uc + VLAN) | `net/core/dev_addr_lists.c` |
| Public types | `include/linux/netdevice.h` |
| UAPI | `include/uapi/linux/if.h`, `include/uapi/linux/if_link.h`, `include/uapi/linux/sockios.h` |

### compatibility contract

### `struct net_device` layout

`include/linux/netdevice.h` defines `struct net_device` (~1500 bytes; ~150 fields). First-cache-line + commonly-accessed fields layout-equivalent to upstream so out-of-tree drivers + BPF programs accessing fields via inline macros work unchanged.

Key fields:
- `name[IFNAMSIZ]` (interface name; e.g., "eth0")
- `ifindex` (kernel-internal index)
- `flags` (IFF_UP, IFF_BROADCAST, IFF_DEBUG, IFF_LOOPBACK, IFF_POINTOPOINT, IFF_NOTRAILERS, IFF_RUNNING, IFF_NOARP, IFF_PROMISC, IFF_ALLMULTI, IFF_MASTER, IFF_SLAVE, IFF_MULTICAST, IFF_PORTSEL, IFF_AUTOMEDIA, IFF_DYNAMIC, IFF_LOWER_UP, IFF_DORMANT, IFF_ECHO)
- `mtu`, `min_mtu`, `max_mtu`
- `type` (ARPHRD_*: ETHER, INFINIBAND, LOOPBACK, ...)
- `hard_header_len`, `min_header_len`, `needed_headroom`, `needed_tailroom`
- `dev_addr` (HW address)
- `addr_len`, `dev_addr_shadow`
- `mc_list`, `uc_list` (multicast/unicast lists)
- `tx_queue_len`, `weight`
- `_tx`, `_rx` (per-queue arrays)
- `num_tx_queues`, `num_rx_queues`, `real_num_tx_queues`, `real_num_rx_queues`
- `state` (atomic: __LINK_STATE_*)
- `nd_net` (netns back-pointer)
- `netdev_ops` (vtable)
- `ethtool_ops`, `xfrmdev_ops`, `tlsdev_ops`, `dcbnl_ops`, `header_ops`, `l3mdev_ops`, `ndisc_ops`, `xdp_metadata_ops`, `xdp_state` (XDP RX path)
- `priv_flags`, `gflags`
- `dev_port`, `mc_count`, `uc_count`
- `napi_list` (per-NAPI struct linked list)
- `tstats`, `core_stats` (statistics)

Layout-equivalent.

### Network device ops vtable

`include/linux/netdevice.h` defines `struct net_device_ops`. Function-pointer-table layout byte-identical so existing NIC drivers work unchanged.

Key functions: `ndo_init`, `ndo_uninit`, `ndo_open`, `ndo_stop`, `ndo_start_xmit`, `ndo_select_queue`, `ndo_change_rx_flags`, `ndo_set_rx_mode`, `ndo_set_mac_address`, `ndo_validate_addr`, `ndo_do_ioctl`, `ndo_eth_ioctl`, `ndo_siocbond`, `ndo_siocdevprivate`, `ndo_set_config`, `ndo_change_mtu`, `ndo_neigh_setup`, `ndo_tx_timeout`, `ndo_get_stats64`, `ndo_get_stats`, `ndo_has_offload_stats`, `ndo_get_offload_stats`, `ndo_vlan_rx_add_vid`, `ndo_vlan_rx_kill_vid`, `ndo_set_vf_mac`, `ndo_set_vf_vlan`, `ndo_set_vf_rate`, `ndo_set_vf_spoofchk`, `ndo_set_vf_link_state`, `ndo_get_vf_config`, `ndo_set_vf_port`, `ndo_get_vf_port`, `ndo_set_vf_guid`, `ndo_set_vf_rss_query_en`, `ndo_setup_tc`, `ndo_rx_flow_steer`, `ndo_add_slave`, `ndo_del_slave`, `ndo_get_xmit_slave`, `ndo_sk_get_lower_dev`, `ndo_fix_features`, `ndo_set_features`, `ndo_neigh_construct`, `ndo_neigh_destroy`, `ndo_fdb_add`, `ndo_fdb_del`, `ndo_fdb_dump`, `ndo_fdb_get`, `ndo_mdb_add`, `ndo_mdb_del`, `ndo_mdb_dump`, `ndo_bridge_setlink`, `ndo_bridge_getlink`, `ndo_bridge_dellink`, `ndo_change_carrier`, `ndo_get_phys_port_id`, `ndo_get_port_parent_id`, `ndo_get_phys_port_name`, `ndo_dfwd_*`, `ndo_get_lock_subclass`, `ndo_select_queue` (more arg variant), `ndo_features_check`, `ndo_xsk_wakeup`, `ndo_get_devlink_port`, `ndo_tunnel_*`, `ndo_set_tx_maxrate`, `ndo_xdp_xmit`, `ndo_xdp`, `ndo_xdp_get_xmit_slave`, `ndo_xsk_wakeup`, `ndo_get_iflink`, `ndo_fill_metadata_dst`, `ndo_get_peer_dev`.

(150+ ops at baseline.)

### Userspace ABIs

- **`/sys/class/net/<dev>/*`**: per-netdev sysfs tree. `addr_assign_type`, `address`, `addr_len`, `broadcast`, `carrier`, `carrier_changes`, `carrier_down_count`, `carrier_up_count`, `dev_id`, `dev_port`, `dormant`, `duplex`, `flags`, `gro_flush_timeout`, `ifalias`, `ifindex`, `iflink`, `link_mode`, `mtu`, `name_assign_type`, `napi_defer_hard_irqs`, `netdev_group`, `operstate`, `phys_port_id`, `phys_port_name`, `phys_switch_id`, `proto_down`, `speed`, `tx_queue_len`, `type`, `uevent`, `queues/{rx,tx}-N/*`, `statistics/*`, `wireless/*` (for wireless devices). All format-identical.

- **`SIOCGIFNAME` / `SIOCSIFFLAGS` / `SIOCGIFADDR` / `SIOCETHTOOL` / etc.** ioctls — numeric values + struct ifreq layout byte-identical.

- **rtnetlink RTM_NEWLINK / RTM_DELLINK / RTM_GETLINK / RTM_SETLINK** wire format byte-identical; existing iproute2 / ip / ifconfig tools work unmodified.

- **udev events**: `add` / `remove` on `/sys/class/net/<dev>` carry NETWORK_INTERFACE-type events; format-identical.

### NAPI

`struct napi_struct` per-driver per-queue:
- `poll_list` (per-CPU NAPI poll list)
- `state` (atomic: NAPI_STATE_SCHED, NAPI_STATE_NPSVC, NAPI_STATE_DISABLE, NAPI_STATE_HASHED, NAPI_STATE_PREFER_BUSY_POLL)
- `weight` (poll budget)
- `defer_hard_irqs_count`
- `gro_hash[]`, `rx_count`, `rx_budget`, `rx_packets`, `rx_bytes`
- `napi_id`
- `poll` callback (driver-provided)

NAPI lifecycle: driver calls `__napi_schedule` from interrupt → kernel schedules poll on next softirq → poll callback drains RX queue + calls `napi_complete_done` when budget exhausted or no more packets.

### XDP (eXpress Data Path)

XDP attachment per-netdev: `XDP_ATTACHED_NONE`, `XDP_ATTACHED_DRV`, `XDP_ATTACHED_SKB`, `XDP_ATTACHED_HW`. Attached BPF program runs at RX before allocating skb. Cross-ref `net/bpf-net.md` for the full XDP design.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| netdev alloc + initialization | `kani::proofs::net::netdev::alloc_safety` |
| Per-queue ring manipulation | `kani::proofs::net::netdev::queue_safety` |
| NAPI state-machine atomic transitions | `kani::proofs::net::netdev::napi_state_safety` |
| Address-list link manipulation | `kani::proofs::net::netdev::addr_list_safety` |
| Devmem DMA-buf binding | `kani::proofs::net::netdev::devmem_safety` |

### Layer 2: TLA+ models

- `models/net/napi_state.tla` (mandatory per `net/00-overview.md` Layer 2) — proves NAPI state machine never deadlocks. Owned here.
- `models/net/rtnl_lock.tla` (mandatory) — proves rtnl_lock serializes netdev mutations correctly. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-netns netdev list | Each netdev appears exactly once; ifindex unique within a netns | `kani::proofs::net::netdev::list_invariants` |
| Address list (mc/uc/VLAN) | Each address appears at most once per type; refcounts consistent | `kani::proofs::net::netdev::addr_invariants` |
| NAPI poll list (per-CPU) | Each NAPI on the list has STATE_SCHED set | `kani::proofs::net::netdev::napi_list_invariants` |
| Per-queue tx_queue + rx_queue arrays | num_*_queues bound; each queue index < num_*_queues | `kani::proofs::net::netdev::queue_index_invariants` |

### Layer 4: Functional correctness (opt-in)

- **NAPI state machine** via Verus — proves: every `__napi_schedule` is paired with exactly one poll-completion; no NAPI starvation under fairness.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | netdev refcount, NAPI refcount, address-list refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | netdev allocated via `KmemCache::<NetDevice>::new()`; per-driver typing | § Mandatory |

### Row-1 features consumed by this component

- **UDEREF**: ioctl args from userspace go through `UserPtr<...>`
- **SIZE_OVERFLOW**: queue-index + buffer-size + frag-count arithmetic uses checked operators
- **CONSTIFY**: per-driver net_device_ops vtables provided by NIC drivers are `static const`
- **MEMORY_SANITIZE**: freed netdev objects zeroed (especially relevant for HW-key state in tlsdev_ops / xfrmdev_ops)

### Row-2 / GR-RBAC integration

LSM hooks: `security_netif_*` (limited; older API). Modern hook surface is via socket-layer (`net/socket-api.md`) + LSM-tagged secmark in skbs.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

