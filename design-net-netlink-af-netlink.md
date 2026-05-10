---
title: "Tier-3: net/netlink/af_netlink.c — AF_NETLINK socket family core"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`AF_NETLINK` is Linux's primary in-kernel ↔ userspace control-plane socket family. Used for: `NETLINK_ROUTE` (rtnetlink — interfaces/routes/neighbours), `NETLINK_AUDIT`, `NETLINK_NETFILTER`, `NETLINK_GENERIC` (genetlink — ethtool/wireless/devlink), `NETLINK_SELINUX`, `NETLINK_KOBJECT_UEVENT`, `NETLINK_INET_DIAG` (socket-stats), `NETLINK_XFRM`, `NETLINK_USERSOCK` (user↔user), `NETLINK_CRYPTO`, `NETLINK_RDMA`, etc. Per-protocol kernel-side multicast group + per-PID unicast + per-protocol policy/dump callbacks.

This Tier-3 covers `af_netlink.c` (~2953 lines).

### Acceptance Criteria

- [ ] AC-1: `socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)` succeeds.
- [ ] AC-2: bind to nl_pid=0: kernel-assigned PID returned.
- [ ] AC-3: bind to nl_pid=2: subsequent socket bind to nl_pid=2 fails EADDRINUSE.
- [ ] AC-4: sendmsg(RTM_GETLINK) + recvmsg: NLM_F_MULTI dump received.
- [ ] AC-5: NETLINK_AUDIT broadcast from kernel: per-listener receives.
- [ ] AC-6: NLM_F_ACK flag: kernel returns NLMSG_ERROR with error=0.
- [ ] AC-7: per-protocol policy mismatch: extack populated; -EINVAL.
- [ ] AC-8: SO_RCVBUF full: ENOBUFS returned (default).
- [ ] AC-9: NETLINK_NO_ENOBUFS sockopt: silent drop instead of ENOBUFS.
- [ ] AC-10: per-namespace: NETLINK_ROUTE in nsA invisible from nsB.
- [ ] AC-11: Per-rxqueue: 16 in-flight unicast msgs to same socket: all delivered FIFO.
- [ ] AC-12: NETLINK_LISTEN_ALL_NSID: per-skb NETLINK_NSID attr present.

### Architecture

Per-protocol netlink table:

```
struct NetlinkTable {
  hash: RhashTable<NetlinkSock>,         // per-PID lookup
  mc_list: HListHead,                     // multicast subscribers
  bind: Option<fn(net, group) -> i32>,
  unbind: Option<fn(net, group)>,
  flags: NLF,
  registered: u32,
  cb_mutex: Mutex<()>,
}
const NETLINK_MAX_PROTOCOLS: usize = 32;
struct NetnsNetlink {
  table: [NetlinkTable; NETLINK_MAX_PROTOCOLS],
}
```

Per-socket state:

```
struct NetlinkSock {
  sk: Sock,                              // base AF_*
  portid: u32,                           // bind PID
  dst_portid: u32,                       // last connected PID
  dst_group: u32,
  flags: NetlinkSockFlags,
  subscriptions: u32,                    // # multicast groups
  ngroups: u32,
  groups: BitVec,                        // per-group subscribed
  state: NLS,
  dump_done_errno: i32,
  cb: Option<NetlinkCallback>,           // active dump
  cb_running: bool,
  cb_def: Option<...>,
  netlink_rcv: Option<fn(skb)>,          // kernel-side input
  netlink_bind: Option<fn(net, group)>,
}
```

Per-skb netlink CB:

```
struct NetlinkCb {
  portid: u32,
  dst_group: u32,
  flags: u32,
  nsid: u32,
  nsid_is_set: bool,
  creds: SkbCreds,
  sk: Option<Arc<Sock>>,
}
```

`Netlink::create` (socket()):
1. Validate protocol < NETLINK_MAX_PROTOCOLS.
2. Allocate NetlinkSock.
3. sk.sk_family = AF_NETLINK; sk.sk_protocol = protocol.
4. Per-protocol per-net netlink_table[protocol].registered: must be set or NL_CFG_F_NONROOT_RECV permits.
5. Insert into sock-list.

`Netlink::bind` (bind()):
1. Validate sockaddr_nl.
2. If nl_pid == 0: assign tgid; ensure unique.
3. Insert into netlink_table[protocol].hash by PID.
4. Per-group subscribe via netlink_table.bind(net, group).
5. Update mc_list per-group.

`Netlink::sendmsg` (sendmsg()):
1. Per-msg parse sockaddr_nl + iov.
2. Build skb with NETLINK_CB(skb).portid = sk.portid; .dst_group = nl_groups; .creds = current_creds().
3. If nl_pid != 0: netlink_unicast(sk, skb, nl_pid, MSG_DONTWAIT?).
4. If nl_groups != 0: netlink_broadcast(sk, skb, sk.portid, nl_groups, GFP_KERNEL).

`Netlink::unicast`:
1. Lookup target NetlinkSock by portid in table.
2. If kernel-side: invoke target.netlink_rcv(skb) directly (synchronous; locks cb_mutex).
3. If userspace: skb_queue_tail(target.sk_receive_queue) + sk_data_ready.
4. Per-rcvbuf: if full, ENOBUFS or silent-drop.

`Netlink::broadcast`:
1. Per-listener in mc_list with group bit set:
   - skb_clone (or skb_get for last).
   - skb_queue_tail(listener.sk_receive_queue).

`Netlink::kernel_create` (kernel-side):
1. Allocate kernel-side NetlinkSock.
2. Bind to PID 0; mark as kernel.
3. Set netlink_rcv = cfg.input.
4. Insert into table.

`Netlink::dump_start`:
1. Allocate cb; populate from control.
2. cb_running = true.
3. Call cb.dump(skb) repeatedly (queued); each per-iter fills until NLMSG_DONE.

### Out of Scope

- genetlink (covered separately)
- rtnetlink (covered separately)
- Netlink-INET-DIAG (covered with /proc/net/sock_diag separately)
- Netlink-XFRM (covered with xfrm/ separately)
- nfnetlink (covered with netfilter/ separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct netlink_sock` | per-socket state | `NetlinkSock` |
| `struct netlink_table[]` | per-protocol table | `NetlinkTable` |
| `struct netlink_kernel_cfg` | per-protocol kernel config | `NetlinkKernelCfg` |
| `netlink_create()` | AF_NETLINK socket() | `Netlink::create` |
| `netlink_bind()` | bind PID+groups | `Netlink::bind` |
| `netlink_sendmsg()` | per-msg send | `Netlink::sendmsg` |
| `netlink_recvmsg()` | per-msg recv | `Netlink::recvmsg` |
| `netlink_unicast()` | per-PID unicast | `Netlink::unicast` |
| `netlink_broadcast()` | per-group multicast | `Netlink::broadcast` |
| `netlink_kernel_create()` | per-protocol kernel sock | `Netlink::kernel_create` |
| `netlink_dump_start()` | dump iteration | `Netlink::dump_start` |
| `nlmsg_parse()` | per-attr parse | `Nlmsg::parse` |
| `nla_put()` / `nla_get_u32()` | per-attr put/get | `Nla::*` |
| `genlmsg_parse()` | genetlink parse | `Genlmsg::parse` |
| `NETLINK_CB(skb)` | per-skb netlink CB | `NetlinkCb` |
| `NETLINK_PORTID(skb)` | per-skb origin PID | `NetlinkCb::portid` |
| `NETLINK_GROUPS()` | per-skb dst groups | `NetlinkCb::dst_group` |

### compatibility contract

REQ-1: `AF_NETLINK` socket() with per-protocol type:
- `socket(AF_NETLINK, SOCK_RAW or SOCK_DGRAM, NETLINK_*)`.
- Per-protocol entry in netlink_table[] determines kernel-side handler.
- Per-socket `netlink_sock` allocated.

REQ-2: bind() with `sockaddr_nl`:
- `nl_pid` = bind PID (0 = kernel-assigned to thread-group-id).
- `nl_groups` = per-group bitmap (multicast subscription).
- Per-PID uniqueness within protocol.

REQ-3: sendmsg() with msg_iov + sockaddr_nl:
- If sockaddr_nl.nl_pid != 0: unicast to that PID.
- If sockaddr_nl.nl_groups != 0: multicast to those groups.
- Per-msg validated: NLMSG_HDR length consistency.

REQ-4: recvmsg(): per-skb dequeue from sk_receive_queue:
- Per-skb NETLINK_CB tracks origin-PID + dst-group + sk_credentials.
- Per-msg framed: `struct nlmsghdr` header + payload.
- Per-msg flags: NLM_F_REQUEST / NLM_F_MULTI / NLM_F_ACK / NLM_F_ECHO / NLM_F_DUMP_INTR.

REQ-5: per-protocol kernel-side handler:
- `netlink_kernel_create(net, protocol, cfg)`:
  - cfg.input → per-msg callback `(*input)(skb)`.
  - cfg.bind → per-bind callback (multicast group subscribe).
  - cfg.flags: NL_CFG_F_NONROOT_RECV / NL_CFG_F_NONROOT_SEND.
- Per-protocol kernel-sock holds reference; per-skb input invoked under per-sock mutex.

REQ-6: per-protocol policy:
- `nlmsg_parse(nlh, hdrlen, tb, maxtype, policy, extack)`:
  - Per-attribute (nla) walks `NLA_F_NESTED` tree.
  - Per-attribute type-checked against policy.
  - extack populates user-visible error string.

REQ-7: dump (NLM_F_DUMP):
- `netlink_dump_start(sk, skb, nlh, control)`:
  - control.start: pre-iter callback.
  - control.dump: per-call dump (each fills skb until NLM_F_MULTI cleared).
  - control.done: post-iter cleanup.
- Per-call resumes from per-cb cb_seq.

REQ-8: ACK (NLM_F_ACK):
- Per-request reply with NLMSG_ERROR (error=0) or NLMSG_ERROR (error=-errno).
- Per-error: optional NLMSGERR_ATTR_MSG / NLMSGERR_ATTR_OFFS extack attributes.

REQ-9: per-protocol multicast:
- `netlink_broadcast(ssk, skb, portid, group, allocation)`:
  - Per-listener-on-group with subscriber pid != portid: skb_clone + skb_queue_tail.
- Per-skb optionally CLONE'd or skb_get for shared.

REQ-10: per-skb credentials:
- NETLINK_CREDS macro: per-skb sk_credentials (uid/gid/pid).
- Per-protocol input handler may CAP_NET_ADMIN-check via nl_capable().

REQ-11: NETLINK_LISTEN_ALL_NSID:
- Per-socket sockopt: receive multicast from all net-namespaces.
- Per-skb attr NETLINK_NSID indicates origin-net.

REQ-12: NETLINK_NO_ENOBUFS / SO_RCVBUF:
- Per-socket: if rcvbuf full, default returns ENOBUFS to userspace.
- NETLINK_NO_ENOBUFS sockopt: drop+silent.

REQ-13: per-namespace netlink-table:
- Per-protocol `bind()` validated against current netns.
- Per-skb netlink_broadcast scoped to skb_net.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bind_pid_unique_in_table` | INVARIANT | per-(protocol, pid) at most one bound NetlinkSock. |
| `protocol_lt_max_protocols` | INVARIANT | per-create protocol < NETLINK_MAX_PROTOCOLS. |
| `cb_running_implies_cb_some` | INVARIANT | per-sock cb_running ⟹ cb.is_some(). |
| `mc_list_per_group_consistent` | INVARIANT | per-group: mc_list contains only subscribers of that group. |
| `skb_creds_present_per_sendmsg` | INVARIANT | per-NETLINK_CB(skb).creds populated from current. |

### Layer 2: TLA+

`net/netlink/af_netlink.tla`:
- Per-protocol bind/unbind + send/recv lifecycle.
- Per-skb traversal queue → unicast/multicast.
- Properties:
  - `safety_no_double_bind` — per-(protocol, pid) bound at most once.
  - `safety_unicast_to_bound_only` — per-unicast portid resolves to bound sock.
  - `safety_multicast_subset` — per-broadcast: receivers ⊆ group-subscribers.
  - `liveness_dump_eventually_done` — per-NLM_F_DUMP eventually receives NLMSG_DONE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Netlink::bind` post: per-pid unique-in-table; mc_list updated per group | `Netlink::bind` |
| `Netlink::unicast` post: target-sock exists or ECONNREFUSED | `Netlink::unicast` |
| `Netlink::broadcast` post: per-subscriber receives clone | `Netlink::broadcast` |
| `Netlink::dump_start` post: cb_running=true; cb.callback bound | `Netlink::dump_start` |
| `Netlink::sendmsg` post: NETLINK_CB.creds == current.creds | `Netlink::sendmsg` |

### Layer 4: Verus/Creusot functional

`Per-NLM_F_DUMP issue → multiple NLM_F_MULTI replies → NLMSG_DONE` semantic equivalence: per-protocol dump stream framing matches `man 7 netlink`.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

Netlink-specific reinforcement:

- **Per-bind capability check** — defense against unprivileged process binding to NETLINK_AUDIT.
- **Per-msg policy validation** — defense against malformed netlink-attr crashing kernel parser.
- **Per-skb credentials propagated** — defense against PID-spoofing.
- **Per-rcvbuf ENOBUFS** — defense against malicious kernel-side flooding userspace.
- **Per-protocol mc_list scoped to net-namespace** — defense against cross-ns leak.
- **Per-broadcast skb_clone/get** — defense against per-receiver state corruption.
- **Per-extack populated on parse fail** — defense against silent error-swallowing.
- **Per-NLM_F_REQUEST: per-input handler holds cb_mutex** — defense against concurrent kernel handler racing.
- **Per-rhashtable bind PID lookup** — defense against O(n) walking attack.
- **Per-nsid attr restricted to LISTEN_ALL_NSID-permitted** — defense against cross-ns observability leak.
- **Per-msg NLMSG_HDR length validated** — defense against malformed header overrunning.

