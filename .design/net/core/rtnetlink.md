# Tier-3: net/core/rtnetlink.c — RTNETLINK message dispatch (link/addr/route/neighbour management)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/core/rtnetlink.c (~7141 lines)
  - include/net/rtnetlink.h (~262 lines)
  - include/uapi/linux/rtnetlink.h
  - include/uapi/linux/if_link.h (RTM_*LINK attrs)
-->

## Summary

`rtnetlink` is the NETLINK_ROUTE-protocol message dispatcher controlling per-net interfaces (RTM_*LINK), per-net addresses (RTM_*ADDR), per-net routes (RTM_*ROUTE), per-net neighbours (RTM_*NEIGH), per-net rules (RTM_*RULE), per-net qdiscs (RTM_*QDISC) + per-net traffic-class. Per-msg-type handler registered via `rtnl_register_many` per (protocol, msg-type) tuple. Single global `rtnl_lock` mutex serializes all-of-net-namespace netdev configuration. Per-link rtnl_link_ops registers per-link-kind (vlan/bridge/veth/dummy/macvlan/wireguard/etc.) for newlink/dellink/changelink. iproute2 ↔ rtnetlink is the dominant control-plane.

This Tier-3 covers `rtnetlink.c` (~7141 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `rtnl_lock()` / `rtnl_unlock()` | global rtnetlink mutex | `Rtnl::lock` / `unlock` |
| `rtnl_register_many()` | per-msg-type handler register | `Rtnl::register_many` |
| `struct rtnl_msg_handler` | per (protocol, msg-type, doit, dumpit) | `RtnlMsgHandler` |
| `struct rtnl_link_ops` | per-link-kind operations | `RtnlLinkOps` |
| `__rtnl_link_register()` | per-link-kind register | `Rtnl::link_register` |
| `__rtnl_link_unregister()` | per-link-kind unregister | `Rtnl::link_unregister` |
| `rtnetlink_rcv()` | per-skb input handler | `Rtnl::rcv` |
| `rtnetlink_rcv_msg()` | per-msg dispatch | `Rtnl::rcv_msg` |
| `rtnl_unicast()` | per-reply unicast | `Rtnl::unicast` |
| `rtnetlink_send()` | per-mcast multicast | `Rtnl::send` |
| `rtnl_set_sk_err()` | per-sock notify error | `Rtnl::set_sk_err` |
| `rtnl_link_get_net()` | per-msg target netns | `Rtnl::link_get_net` |
| `rtnl_dump_ifinfo()` | per-RTM_GETLINK dump | `Rtnl::dump_ifinfo` |
| `rtnl_fill_ifinfo()` | per-link build NLA payload | `Rtnl::fill_ifinfo` |
| `RTM_NEWLINK` (16) ... `RTM_GETSTATS` (94) | per-msg-type | UAPI |
| `RTNLGRP_LINK` (1) ... | per-mcast-group | UAPI |
| `IFLA_*` | per-link attributes | UAPI |

## Compatibility contract

REQ-1: NETLINK_ROUTE family registration:
- Per-namespace `netlink_kernel_create(net, NETLINK_ROUTE, &cfg)`.
- cfg.input = `rtnetlink_rcv`.
- cfg.bind = `rtnetlink_bind`.
- cfg.flags = NL_CFG_F_NONROOT_RECV (allow non-root listen).

REQ-2: Global `rtnl_lock`:
- All RTM_NEW*/DEL*/SET* mutate per-netdev state under global mutex.
- Per-handler invoked under rtnl_lock.
- Per-RTM_GET* may use RCU-only path or rtnl_lock + dump.

REQ-3: Per-msg-type handler:
- `rtnl_register_many(handlers, n)`:
  - Per-handler: protocol (PF_*), msg-type (RTM_*), doit, dumpit, flags.
  - flags: RTNL_FLAG_DOIT_UNLOCKED (skip rtnl_lock), RTNL_FLAG_DUMP_UNLOCKED, RTNL_FLAG_DOIT_PERNET, RTNL_FLAG_BULK_DEL_SUPPORTED.
- Per-(protocol, msg-type) at most one handler.

REQ-4: rtnetlink_rcv flow:
- Per-skb netlink_rcv_skb(skb, &rtnetlink_rcv_msg).
- Per-msg: validate nlmsg_type ∈ [RTM_BASE, RTM_MAX].
- Lookup handler by (nlh.nlmsg_type, sockaddr.protocol).
- If NLM_F_DUMP: netlink_dump_start with handler.dumpit.
- Else: validate via handler.policy + invoke handler.doit.

REQ-5: Per-link-kind ops:
- `struct rtnl_link_ops`: kind (string), priv_size, setup, validate, newlink, changelink, dellink, get_size, fill_info, get_xstats_size, fill_xstats, get_link_net, slave_*.
- `__rtnl_link_register(ops)` per-kind.
- Per-RTM_NEWLINK with IFLA_LINKINFO.IFLA_INFO_KIND="vlan": dispatch ops.newlink.

REQ-6: Per-RTM_NEWLINK doit:
- Per-msg parse IFLA_* attrs.
- Per-msg validate.
- If IFLA_LINKINFO: dispatch link_ops by kind.
- Else: rtnl_change_link / rtnl_link_alloc.
- Per-newlink: register_netdevice + send NLM_F_ECHO reply if requested.

REQ-7: Per-RTM_GETLINK dumpit:
- Per-net netdev list iteration.
- Per-netdev rtnl_fill_ifinfohdr + IFLA_* attrs.
- Per-cb cb_seq tracks resume.
- NLM_F_MULTI per-skb until NLMSG_DONE.

REQ-8: Per-multicast notify:
- Per-add/del/change: `rtnl_notify(skb, net, portid, group, nlh, flags)`.
- Per-group: RTNLGRP_LINK / RTNLGRP_IPV4_IFADDR / RTNLGRP_IPV6_IFADDR / RTNLGRP_IPV4_ROUTE / etc.
- Per-listener via netlink_broadcast.

REQ-9: Per-RTM_NEWLINK CAP-check:
- CAP_NET_ADMIN required for state-mutating msgs.
- netlink_capable / nl_capable_for_user_ns gates.

REQ-10: Per-extack:
- Per-handler.policy validates IFLA_* per-attr.
- Per-fail: NL_SET_ERR_MSG_ATTR populates extack.

REQ-11: Per-net-namespace:
- Per-namespace per-rtnetlink-listener.
- Per-link IFLA_TARGET_NETNSID redirect target.

REQ-12: Per-bulk-del:
- RTNL_FLAG_BULK_DEL_SUPPORTED handlers accept multi-link RTM_DELLINK with NLM_F_BULK.

REQ-13: Per-stats:
- RTM_NEWSTATS / RTM_GETSTATS dump per-link statistics.
- IFLA_STATS_LINK_64 / IFLA_STATS_LINK_XSTATS / IFLA_STATS_LINK_OFFLOAD_XSTATS.

## Acceptance Criteria

- [ ] AC-1: `ip link show`: rtnl_dump_ifinfo dumps all interfaces; per-skb NLM_F_MULTI.
- [ ] AC-2: `ip link set lo up`: RTM_NEWLINK doit invoked; netdev brought up.
- [ ] AC-3: `ip link add type vlan ...`: rtnl_link_ops("vlan").newlink invoked.
- [ ] AC-4: `ip link del eth0.10`: RTM_DELLINK doit; IFLA_LINKINFO dispatch ops.dellink.
- [ ] AC-5: Per-non-CAP_NET_ADMIN process: RTM_NEWLINK -EPERM.
- [ ] AC-6: Per-RTNL link-add: RTNLGRP_LINK multicast received by listener.
- [ ] AC-7: Per-IFLA_LINKINFO unknown kind: -EOPNOTSUPP.
- [ ] AC-8: Per-namespace: nsA `ip link` doesn't see nsB interfaces.
- [ ] AC-9: Per-RTM_GETSTATS: NLA_NESTED IFLA_STATS_LINK_64 returned.
- [ ] AC-10: Per-bulk-del: NLM_F_BULK + RTM_DELLINK affects all matching.
- [ ] AC-11: Per-NLM_F_ECHO: RTM_NEWLINK reply contains created link.
- [ ] AC-12: Per-extack on policy fail: bad_attr-offset populated.

## Architecture

Per-handler registry:

```
struct RtnlMsgHandler {
  protocol: u8,                                   // PF_UNSPEC, PF_INET, etc.
  msgtype: u16,                                   // RTM_*
  doit: Option<fn(&mut Skb, &Nlmsghdr, &mut NlExtAck) -> Result<()>>,
  dumpit: Option<fn(&mut Skb, &mut NetlinkCallback) -> Result<usize>>,
  policy: Option<&'static [NlaPolicy]>,
  maxattr: u32,
  flags: RtnlFlags,
}

const RTM_MAX_PROTOCOLS: usize = 16;             // PF_*
const RTM_MAX_MSGTYPE: usize = (RTM_GETSTATS - RTM_BASE + 1);

struct RtnlRegistry {
  handlers: [[Option<RtnlMsgHandler>; RTM_MAX_MSGTYPE]; RTM_MAX_PROTOCOLS],
}
```

Per-link-kind ops:

```
struct RtnlLinkOps {
  kind: &'static str,
  priv_size: usize,
  setup: fn(&mut NetDev),
  validate: Option<fn(&[Option<&Nla>], &mut NlExtAck) -> Result<()>>,
  newlink: Option<fn(&mut NetDev, &[Option<&Nla>], &mut NlExtAck) -> Result<()>>,
  changelink: Option<fn(&mut NetDev, &[Option<&Nla>], &mut NlExtAck) -> Result<()>>,
  dellink: Option<fn(&mut NetDev, &mut Vec<&NetDev>) -> Result<()>>,
  fill_info: Option<fn(&mut Skb, &NetDev) -> Result<()>>,
  get_size: Option<fn(&NetDev) -> usize>,
}
```

Global rtnl mutex:

```
static RTNL_MUTEX: Mutex<()> = Mutex::new(());
```

`Rtnl::rcv(skb)`:
1. netlink_rcv_skb(skb, &Rtnl::rcv_msg).

`Rtnl::rcv_msg(skb, nlh, extack)`:
1. Validate nlh.nlmsg_type ∈ [RTM_BASE, RTM_MAX].
2. proto = sockaddr.nl_pid.protocol_from_msg(nlh.nlmsg_type) (typically PF_INET / PF_INET6 / PF_UNSPEC).
3. handler = registry.get(proto, nlh.nlmsg_type - RTM_BASE).
4. If !handler: -EOPNOTSUPP.
5. If nlh.nlmsg_flags & NLM_F_DUMP:
   - netlink_dump_start(sk, skb, nlh, &control { dump: handler.dumpit, ... }).
6. Else:
   - CAP-check (CAP_NET_ADMIN unless RTM_GET*).
   - nlmsg_parse_strict(nlh, hdrlen, attrs, handler.maxattr, handler.policy, extack).
   - Acquire rtnl_lock unless RTNL_FLAG_DOIT_UNLOCKED.
   - handler.doit(skb, nlh, extack)?
   - Release rtnl_lock.

`Rtnl::register_many(handlers)`:
1. Acquire rtnl_lock.
2. For each handler: registry[handler.protocol][handler.msgtype - RTM_BASE] = Some(handler).
3. Release rtnl_lock.

`Rtnl::link_register(ops)`:
1. Acquire rtnl_lock.
2. Insert ops into per-net rtnl_link_kind list.
3. For each existing netdev with kind == ops.kind: ops.setup(netdev).
4. Release rtnl_lock.

`Rtnl::dump_ifinfo(skb, cb)`:
1. start_idx = cb.args[0]; idx = 0.
2. For_each_netdev(net, dev):
   - If idx < start_idx: idx++; continue.
   - rtnl_fill_ifinfo(skb, dev, RTM_NEWLINK, NLM_F_MULTI).
   - If skb full: cb.args[0] = idx; return.
   - idx++.
3. Return idx.

`Rtnl::notify(skb, net, portid, group, nlh, flags)`:
1. nlmsg_multicast(net.rtnl, skb, portid, group, flags).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `handler_protocol_lt_max` | INVARIANT | per-register handler.protocol < RTM_MAX_PROTOCOLS. |
| `handler_msgtype_in_range` | INVARIANT | per-register handler.msgtype ∈ [RTM_BASE, RTM_MAX]. |
| `rtnl_lock_held_during_doit` | INVARIANT | per-doit invocation: !RTNL_FLAG_DOIT_UNLOCKED ⟹ rtnl_lock held. |
| `link_kind_unique` | INVARIANT | per-rtnl_link_kind list: at most one ops per kind. |

### Layer 2: TLA+

`net/core/rtnetlink.tla`:
- Per-msg dispatch + per-link-ops + per-mcast notify.
- Properties:
  - `safety_dispatch_only_to_registered` — per-msg dispatch ⟹ handler registered.
  - `safety_capability_required_for_state_change` — per-RTM_NEW/DEL/SET ⟹ CAP_NET_ADMIN.
  - `safety_rtnl_lock_serializes_state_change` — concurrent RTM_NEW serializes.
  - `liveness_dump_completes` — per-RTM_GET dump terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Rtnl::rcv_msg` post: per-msg dispatched to handler iff registered + CAP-pass | `Rtnl::rcv_msg` |
| `Rtnl::register_many` post: registry[p][m-RTM_BASE].is_some() | `Rtnl::register_many` |
| `Rtnl::link_register` post: ops in per-net link_kind list | `Rtnl::link_register` |
| `Rtnl::dump_ifinfo` post: cb.args[0] tracks resume | `Rtnl::dump_ifinfo` |

### Layer 4: Verus/Creusot functional

`Per-iproute2 RTM_NEWLINK ↔ rtnetlink_rcv → handler dispatch → ops.newlink invoked → netdev created` semantic equivalence: per-iproute2 wire-format matches Linux rtnetlink ABI.

## Hardening

(Inherits row-1 features from `net/netlink/af_netlink.md` § Hardening.)

Rtnetlink-specific reinforcement:

- **CAP_NET_ADMIN required for state-mutating msgs** — defense against unprivileged RTM_NEWLINK.
- **rtnl_lock serializes per-netdev state mutation** — defense against concurrent newlink corruption.
- **Per-link-ops kind validated against registry** — defense against non-existent kind crashing dispatch.
- **Per-msg policy enforced via NlaPolicy** — defense against malformed-attr.
- **Per-namespace dispatcher isolated** — defense against cross-ns leakage.
- **Per-extack populated on parse fail** — defense against silent error.
- **Per-handler maxattr capped** — defense against attr-bomb.
- **Per-bulk-del NLM_F_BULK supported only on flag-marked handlers** — defense against unintended bulk-mutation.
- **Per-RTM_DELLINK validates target ∈ current netns** — defense against cross-ns delete.
- **Per-RTM_GETSTATS NLA_NESTED depth-bounded** — defense against deep-nested stats parsing DoS.
- **Per-mcast notify scoped to per-net** — defense against listener seeing other-ns events.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-link-kind impl (vlan/bridge/veth/etc.; covered separately)
- iproute2 userspace (out-of-tree)
- AF_NETLINK core (covered in `net/netlink/af_netlink.md` Tier-3)
- NLA policy (covered in `net/netlink/policy.md` Tier-3)
- Implementation code
