# Tier-3: net/core/dev_addr_lists.c — per-netdev hardware address lists (UC/MC/dev_addr)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/core/dev.md
upstream-paths:
  - net/core/dev_addr_lists.c (~1434 lines)
  - include/linux/netdevice.h (relevant ranges; HW addr lists)
-->

## Summary

Per-netdev maintains four hardware-address lists: **dev_addr** (primary MAC), **dev_addrs** (multi-MAC for net-bonding/multi-queue NIC), **uc** (unicast secondary addrs from per-VLAN/-IPv6-Stateless), **mc** (multicast addrs from IGMP/MLD/IPv6-Solicited-Node). Per-list entries reference-counted (multiple bonded slaves / VLANs share via per-master ↔ per-slave sync). Per-netdev `dev->ndo_set_rx_mode` invoked when MC/UC list changes to push to HW MAC-filter (or set ALLMULTI for non-filterable). Critical for: VLAN/bonding/macvlan/IPv6 multicast/Wake-on-LAN setup.

This Tier-3 covers `dev_addr_lists.c` (~1434 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct netdev_hw_addr` | per-entry HW addr | `NetdevHwAddr` |
| `struct netdev_hw_addr_list` | per-list head + count + sync_cnt | `NetdevHwAddrList` |
| `dev->dev_addr` | primary MAC | `NetDev::dev_addr` |
| `dev->dev_addrs` | multi-MAC list | `NetDev::dev_addrs` |
| `dev->uc` | secondary unicast list | `NetDev::uc` |
| `dev->mc` | multicast list | `NetDev::mc` |
| `dev_addr_add()` / `dev_addr_del()` | per-secondary-MAC add/del | `Dev::addr_add` / `addr_del` |
| `dev_uc_add()` / `dev_uc_del()` | per-UC add/del | `Dev::uc_add` / `uc_del` |
| `dev_mc_add()` / `dev_mc_del()` | per-MC add/del | `Dev::mc_add` / `mc_del` |
| `__dev_set_rx_mode()` | per-netdev push-to-HW | `Dev::__set_rx_mode` |
| `dev_uc_sync()` / `__dev_uc_sync()` | per-master ↔ slave sync | `Dev::uc_sync` |
| `dev_mc_sync()` / `__dev_mc_sync()` | per-master ↔ slave sync | `Dev::mc_sync` |
| `dev_addr_init()` / `dev_addr_flush()` | per-netdev init/flush | `Dev::addr_init` / `addr_flush` |
| `netdev_for_each_uc_addr` | per-list iter | `NetDev::for_each_uc` |
| `netdev_for_each_mc_addr` | per-list iter | `NetDev::for_each_mc` |

## Compatibility contract

REQ-1: `struct netdev_hw_addr`:
- `addr[MAX_ADDR_LEN]`: per-HW addr (typically 6 bytes for Ethernet).
- `type`: NETDEV_HW_ADDR_T_LAN / WLAN / SLAVE / UNICAST / MULTICAST.
- `refcount`: per-entry reference count.
- `global_use`: set iff added via dev_uc_add_global (always-keep).
- `synced`: set iff propagated to HW.
- `sync_cnt`: per-master ↔ slave sync reference count.
- list_head: doubly-linked list within hw_addr_list.

REQ-2: `struct netdev_hw_addr_list`:
- `list`: list head.
- `count`: number of entries.

REQ-3: dev_addr_add(dev, addr, addr_type):
- If addr already in dev->dev_addrs: refcount++.
- Else: allocate netdev_hw_addr; insert.
- Trigger __dev_set_rx_mode.

REQ-4: dev_uc_add(dev, addr):
- Synchronizes per-master / per-slave UC list.
- If addr in dev->uc: refcount++.
- Else: allocate; insert; sync to slaves.
- Trigger __dev_set_rx_mode.

REQ-5: dev_mc_add(dev, addr):
- Symmetric for MC list.
- Same refcount semantics.

REQ-6: __dev_set_rx_mode():
- Computes desired per-netdev flags: IFF_PROMISC / IFF_ALLMULTI based on UC/MC list size vs HW capacity (dev->uc.count > dev->priv_flags.max_uc) → IFF_PROMISC.
- Calls dev->netdev_ops->ndo_set_rx_mode(dev) under per-netdev tx-queue-lock.

REQ-7: Per-master ↔ slave sync:
- dev_uc_sync(to, from): iterate from->uc; for each !synced: __dev_set_uc(to, addr); set entry.synced=true.
- dev_uc_unsync(to, from): iterate to->uc with sync_cnt > 0 ↔ from: del.

REQ-8: Per-bonding driver:
- Per-bond_slave: dev_uc_sync(slave, bond_dev); dev_mc_sync(slave, bond_dev).
- Per-bond_unslave: dev_uc_unsync.

REQ-9: Per-VLAN driver:
- Per-vlan_dev_uc_add → also adds to underlying real_dev.
- Per-vlan_set_rx_mode → propagates to real_dev.

REQ-10: dev_addr_flush(dev):
- Free all dev->dev_addrs entries.
- Used in netdev unregister.

REQ-11: Per-netdev addr-resolution:
- dev_get_by_index(net, ifindex) ↔ dev->dev_addr lookup.
- Per-netdev iflink: virt-dev → master ifindex.

REQ-12: Per-netdev IFF_PROMISC / IFF_ALLMULTI handling:
- Per-netdev promiscuity-counter tracks all callers (incremented on every dev_set_promiscuity(+1)).
- Per-netdev allmulticast-counter same.

## Acceptance Criteria

- [ ] AC-1: `ip link set eth0 address aa:bb:cc:dd:ee:ff`: dev->dev_addr updated.
- [ ] AC-2: `ip link add link eth0 macvlan0 type macvlan`: macvlan dev_uc_add(eth0, macvlan_addr).
- [ ] AC-3: Per-IPv4 IGMP join 224.0.0.1: dev_mc_add invoked.
- [ ] AC-4: Per-MC list size > HW capacity: IFF_ALLMULTI set.
- [ ] AC-5: dev_uc_add same addr twice: refcount=2; del once: still in list with refcount=1.
- [ ] AC-6: dev_uc_sync from bond → slave: slave receives all bond's UC addrs.
- [ ] AC-7: bond-unslave: slave UC list de-synced.
- [ ] AC-8: VLAN UC add: real_dev sees it.
- [ ] AC-9: dev->ndo_set_rx_mode invoked on UC/MC list change.
- [ ] AC-10: netdev_unregister: dev_addr_flush clears all entries.

## Architecture

Per-entry:

```
struct NetdevHwAddr {
  list: ListLink,
  addr: [u8; MAX_ADDR_LEN],
  type_: NetdevHwAddrType,                       // LAN / WLAN / SLAVE / UNICAST / MULTICAST
  refcount: AtomicI32,
  global_use: bool,
  synced: bool,
  sync_cnt: AtomicI32,
  rcu: RcuHead,
}

const MAX_ADDR_LEN: usize = 32;
```

Per-list head:

```
struct NetdevHwAddrList {
  list: ListHead<NetdevHwAddr>,
  count: AtomicI32,
}
```

Per-netdev (subset):

```
struct NetDev {
  ...
  dev_addr: [u8; MAX_ADDR_LEN],
  dev_addrs: NetdevHwAddrList,                   // multi-MAC
  uc: NetdevHwAddrList,                          // secondary UC
  mc: NetdevHwAddrList,                          // multicast
  uc_promisc: bool,                               // overrun promisc
  flags: NetDevFlags,                             // IFF_PROMISC, IFF_ALLMULTI
  promiscuity: AtomicI32,
  allmulti: AtomicI32,
}
```

`Dev::__hw_addr_add_ex(list, addr, addr_len, type, global)`:
1. Iterate list: if entry.addr == addr ∧ entry.type == type:
   - entry.refcount++; entry.global_use |= global; return Ok.
2. Else: allocate new entry; insert; list.count++.

`Dev::__hw_addr_del_ex(list, addr, addr_len, type, global)`:
1. Iterate list: if entry.addr == addr ∧ entry.type == type:
   - entry.refcount--.
   - If entry.refcount == 0 ∧ !entry.global_use: list_del; kfree_rcu; list.count--.
   - Return Ok.
2. Return -ENOENT.

`Dev::uc_add(dev, addr) -> Result<()>`:
1. netif_addr_lock(dev).
2. err = __hw_addr_add_ex(&dev->uc, addr, ETH_ALEN, NETDEV_HW_ADDR_T_UNICAST, false).
3. __dev_set_rx_mode(dev).
4. netif_addr_unlock(dev).
5. err.

`Dev::__set_rx_mode(dev)`:
1. flags = dev.flags.
2. If dev.uc.count > dev.priv_flags.uc_max_count: flags |= IFF_PROMISC.
3. If dev.mc.count > dev.priv_flags.mc_max_count: flags |= IFF_ALLMULTI.
4. If dev.netdev_ops.ndo_set_rx_mode: call under tx-queue-lock.

`Dev::__hw_addr_sync(to, from, addr_len)`:
1. Iterate from.list: per-entry !synced → __hw_addr_add_ex(to, entry); entry.synced = true.

`Dev::__hw_addr_unsync(to, from, addr_len)`:
1. Iterate from.list: per-entry synced → __hw_addr_del_ex(to, entry); entry.synced = false.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `refcount_ge_zero` | INVARIANT | per-NetdevHwAddr refcount ≥ 0. |
| `count_eq_list_len` | INVARIANT | list.count == #entries in list.list. |
| `synced_implies_in_master` | INVARIANT | per-slave-entry.synced ⟹ corresponding entry in master. |
| `addr_unique_per_type` | INVARIANT | per-list at most one entry per (addr, type). |
| `global_use_implies_no_free` | INVARIANT | per-entry global_use ∧ refcount == 0 ⟹ entry not freed. |

### Layer 2: TLA+

`net/core/dev_addr_lists.tla`:
- Per-list add/del/sync flow.
- Properties:
  - `safety_refcount_balance` — per-add #refcounts == per-del #refcounts at quiescence ⟹ entry freed.
  - `safety_set_rx_mode_invoked_on_change` — per-list mutation ⟹ ndo_set_rx_mode called.
  - `safety_master_slave_sync_eventual` — per-master.uc add ⟹ each slave eventually has entry.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Dev::__hw_addr_add_ex` post: entry in list with refcount ≥ 1 | `Dev::__hw_addr_add_ex` |
| `Dev::__hw_addr_del_ex` post: refcount-- ; freed iff refcount == 0 ∧ !global_use | `Dev::__hw_addr_del_ex` |
| `Dev::__set_rx_mode` post: flags reflect overrun-promisc / overrun-allmulti | `Dev::__set_rx_mode` |
| `Dev::__hw_addr_sync` post: per-from-entry !synced ⟹ master updated | `Dev::__hw_addr_sync` |

### Layer 4: Verus/Creusot functional

`Per-VLAN/bond/macvlan UC/MC add → underlying eth0 sees union of all addrs → ndo_set_rx_mode programs HW filter` semantic equivalence: per-IGMP/IPv6/SOL_SOCKET semantics match Linux netdev addr-list spec.

## Hardening

(Inherits row-1 features from `net/core/dev.md` § Hardening.)

Addr-list-specific reinforcement:

- **Per-entry refcount-based lifetime** — defense against UAF on shared HW addr.
- **Per-list netif_addr_lock** — defense against concurrent add/del corruption.
- **Per-master ↔ slave sync_cnt prevents double-sync** — defense against per-add propagating multiple times.
- **Per-global-use anchor** — defense against per-system-mac forgotten on slave-detach.
- **Per-overrun IFF_PROMISC/ALLMULTI fall-back** — defense against silent drop when HW filter full.
- **Per-MAX_ADDR_LEN bounded** — defense against per-driver claiming larger-than-MAX HW addr.
- **Per-flush on netdev unregister** — defense against UAF post-unregister.
- **Per-RCU-free** — defense against concurrent reader seeing freed entry.
- **Per-MC group bound to per-netdev** — defense against cross-netdev MC leakage.
- **Per-set_rx_mode invoked under tx-queue-lock** — defense against concurrent xmit + filter-update.

## Grsecurity/PaX-style Reinforcement

Baseline grsec/PaX posture inherited workspace-wide:

- **PAX_USERCOPY** — strict bounds on every `SIOCADDMULTI` / `SIOCDELMULTI` / `SIOCSIFHWADDR` ioctl and the netlink `IFLA_ADDRESS` / `IFLA_BROADCAST` attribute copies.
- **PAX_KERNEXEC** — `.rodata` `dev_addr_*` static-call targets; W^X across the per-list manipulation API.
- **PAX_RANDKSTACK** — per-syscall randomisation on rtnetlink/ioctl entries into the HW-addr list path.
- **PAX_REFCOUNT** — saturating `refcount_t` on `struct netdev_hw_addr` (`refcount`, `synced` counter).
- **PAX_MEMORY_SANITIZE** — zero-on-free for `netdev_hw_addr` slabs so dropped HW addrs do not linger.
- **PAX_UDEREF** — enforced isolation between user `ifreq` HW-addr buffers and the in-kernel uc/mc list.
- **PAX_RAP / kCFI** — forward-edge CFI on `ndo_set_rx_mode`, `ndo_set_mac_address`, and `dev_addr_*` indirect dispatch.
- **GRKERNSEC_HIDESYM** — dev-addr internal symbols withheld from non-CAP_SYSLOG kallsym readers.
- **GRKERNSEC_DMESG** — addr-list WARN gated behind CAP_SYSLOG.

dev_addr_lists-specific reinforcement:

- **HW-addr mutation gated by CAP_NET_ADMIN** — defense against unprivileged userspace altering `dev->dev_addr`, unicast filter, or multicast list.
- **PAX_REFCOUNT saturation on `netdev_hw_addr->refcount`** — defense against synced-counter wrap via repeated `dev_uc_sync` / `dev_mc_sync` from a hostile upper device (bond/team/macvlan).
- **`MAX_ADDR_LEN` strictly bounded** — defense against per-driver claiming larger-than-MAX HW addr triggering OOB write.
- **Flush-on-unregister enforced** — defense against UAF post-netdev-unregister.
- **RCU-free for entries** — defense against concurrent reader seeing freed entry during `ndo_set_rx_mode`.
- **MC group bound to its owning netdev** — defense against cross-netdev MC leakage via syncdev abuse.

Rationale: dev_addr_lists is the lookup-and-sync engine that feeds NIC HW filters. Without privilege gating, refcount saturation, and RCU-free, a hostile upper device or unprivileged ioctl can desync HW/SW filters, smuggle promisc semantics, or trigger UAF during teardown. The grsec stack closes each of those primitives.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/core/dev (covered in `dev.md` Tier-3)
- VLAN driver (covered separately)
- Bonding driver (covered separately)
- Macvlan driver (covered separately)
- IGMP / MLD (covered separately)
- Per-driver ndo_set_rx_mode impl (driver-specific)
- Implementation code
