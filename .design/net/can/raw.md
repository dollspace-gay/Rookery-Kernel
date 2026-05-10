# Tier-3: net/can/raw.c — CAN_RAW protocol (Controller Area Network raw socket)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/can/00-overview.md
upstream-paths:
  - net/can/raw.c (~1142 lines)
  - net/can/af_can.c (~932 lines, family core)
  - include/uapi/linux/can/raw.h
  - include/uapi/linux/can.h
-->

## Summary

CAN_RAW is the SocketCAN raw-protocol: per-socket binds to per-CAN-iface (vcan0/can0/etc.) + filter set; per-skb received iff matches at least one filter (id & mask). Per-skb send: kernel constructs CAN-frame; hands to per-iface tx-queue. Critical for: industrial automation (J1939), automotive ECU, embedded sensor I/O. Per-frame format: 11-bit (CAN-2.0A) or 29-bit (CAN-2.0B) ID + DLC + payload up to 8 (CAN) or 64 (CAN-FD) or 2048 (CAN-XL) bytes. Per-can_filter ABI lets userspace install up to N (kernel-default 32) filter pairs. Per-loopback policy controls own-tx echoback.

This Tier-3 covers `raw.c` (~1142 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct raw_sock` | per-socket state | `RawSock` |
| `raw_init()` | per-socket init | `Raw::init` |
| `raw_create()` | socket(AF_CAN, SOCK_RAW, CAN_RAW) | `Raw::create` |
| `raw_release()` | close() | `Raw::release` |
| `raw_bind()` | bind to (ifindex, can_proto) | `Raw::bind` |
| `raw_sendmsg()` | per-frame send | `Raw::sendmsg` |
| `raw_recvmsg()` | per-frame recv | `Raw::recvmsg` |
| `raw_rcv()` | per-skb input from net/can | `Raw::rcv` |
| `raw_setsockopt()` | per-CAN_RAW_* sockopt | `Raw::setsockopt` |
| `raw_getsockopt()` | per-CAN_RAW_* sockopt | `Raw::getsockopt` |
| `can_rx_register()` | per-filter register w/ AF_CAN | `Can::rx_register` |
| `can_rx_unregister()` | per-filter unregister | `Can::rx_unregister` |
| `struct can_filter` | per-filter (can_id + can_mask) | UAPI |
| `CAN_RAW_FILTER` | per-sockopt filter list | UAPI |
| `CAN_RAW_ERR_FILTER` | per-error-frame filter | UAPI |
| `CAN_RAW_LOOPBACK` | per-tx echoback | UAPI |
| `CAN_RAW_RECV_OWN_MSGS` | per-self-tx receive | UAPI |
| `CAN_RAW_FD_FRAMES` | per-CAN-FD enable | UAPI |
| `CAN_RAW_XL_FRAMES` | per-CAN-XL enable | UAPI |
| `CAN_RAW_JOIN_FILTERS` | per-AND filter mode | UAPI |

## Compatibility contract

REQ-1: socket(AF_CAN, SOCK_RAW, CAN_RAW):
- raw_create allocates RawSock; sets per-default filter (1 filter: id=0, mask=0 = match-all).
- Per-socket has CAP_NET_RAW = NOT required for CAN_RAW (no analog to AF_PACKET).

REQ-2: bind(sockaddr_can):
- struct sockaddr_can: { can_family=AF_CAN, can_ifindex, ... }.
- ifindex == 0 → match-any iface.
- Per-iface AF_CAN family registered at `dev_add_pack` for ETH_P_CAN / CANFD.
- raw_bind: removes old filter-registrations; install new for new iface.

REQ-3: Per-CAN_RAW_FILTER setsockopt:
- struct can_filter { can_id, can_mask }.
- Per-skb matches if (skb_can_id & filter.can_mask) == (filter.can_id & filter.can_mask).
- Per-OR semantics by default; CAN_RAW_JOIN_FILTERS = AND semantics.

REQ-4: Per-CAN_RAW_ERR_FILTER:
- Bitmap of error-class subscriptions (CAN_ERR_*).
- Per-skb-error: matches per-bit.

REQ-5: raw_sendmsg(sk, msg, size):
- skb = sock_alloc_send_skb(size + sizeof(can_skb_priv)).
- copy can_frame from userspace.
- skb.dev = bound iface; dev_queue_xmit(skb).
- Per-loopback if enabled: echoback to local sockets.

REQ-6: raw_rcv (per-skb input):
- Per-skb msg_flags: MSG_DONTROUTE_RECV / SCM_LOOPBACK / etc.
- skb_clone; skb_queue_tail(sk_receive_queue).
- sk_data_ready.

REQ-7: Per-loopback:
- CAN_RAW_LOOPBACK (default 1): per-tx own skb echobacks to local-recv.
- CAN_RAW_RECV_OWN_MSGS (default 0): per-self-tx visible to own recv.

REQ-8: CAN-FD frames:
- struct canfd_frame { can_id, len, flags, __res0, __res1, data[64] }.
- CAN_RAW_FD_FRAMES sockopt enables FD send/recv.
- Per-non-FD-enabled socket: drops FD frames.

REQ-9: CAN-XL frames:
- struct canxl_frame { ..., data[2048] }.
- CAN_RAW_XL_FRAMES sockopt.

REQ-10: Per-AF_CAN family core (af_can.c):
- Per-iface NETIF_F_CAN registration.
- Per-{(ifindex, proto)} hash bucket: rx-list of (can_id, can_mask, callback, data).
- Per-skb dispatch on receive walks per-iface bucket.

REQ-11: Per-error-frame:
- can_id has CAN_ERR_FLAG bit set.
- Per-driver populates per-error-class.

REQ-12: Per-namespace:
- AF_CAN per-net.

## Acceptance Criteria

- [ ] AC-1: socket(AF_CAN, SOCK_RAW, CAN_RAW): succeeds.
- [ ] AC-2: bind to vcan0: per-filter installed via can_rx_register.
- [ ] AC-3: Send can_frame: dev_queue_xmit on vcan0.
- [ ] AC-4: Recv can_frame: matched by default match-all filter; delivered.
- [ ] AC-5: setsockopt CAN_RAW_FILTER {0x123, 0x7FF}: only id 0x123 received.
- [ ] AC-6: CAN-FD socket: send 64-byte payload; recv on FD-enabled receiver.
- [ ] AC-7: Non-FD socket: drops FD frame from iface.
- [ ] AC-8: CAN_RAW_LOOPBACK=1 + RECV_OWN_MSGS=1: echo-receives own tx.
- [ ] AC-9: CAN_RAW_JOIN_FILTERS: all-filters-must-match (AND).
- [ ] AC-10: CAN_RAW_ERR_FILTER: error-frame delivered when subscribed bit set.
- [ ] AC-11: ifindex=0: bound to all CAN ifaces.
- [ ] AC-12: candump userspace tool: works via CAN_RAW.

## Architecture

Per-socket state:

```
struct RawSock {
  sk: Sock,                                       // base AF_*
  bound: bool,
  ifindex: i32,
  count: u32,                                     // # filters
  filter: Vec<CanFilter>,
  filter_kmem: Option<&[CanFilter]>,
  err_mask: u32,
  loopback: bool,
  recv_own_msgs: bool,
  fd_frames: bool,
  xl_frames: bool,
  join_filters: bool,
  uid_filter: bool,
}

struct CanFilter {
  can_id: __u32,
  can_mask: __u32,
}
```

Per-AF_CAN state (af_can.c):

```
struct CanRcv {
  list: HListLink,                                // (ifindex, proto) bucket
  can_id: u32,
  mask: u32,
  func: fn(skb, data),
  data: *void,
  matches: AtomicU64,
  ident: &'static str,
  rcu: RcuHead,
}

struct CanRcvList {
  rx: HListHead<CanRcv>,
  ...
}
```

`Raw::create(net, sock, proto, kern) -> Result<()>`:
1. Validate sock.type == SOCK_RAW; proto == CAN_RAW.
2. Allocate RawSock.
3. ro.bound = false; ro.ifindex = 0.
4. Default filter: ro.filter[0] = {can_id=0, can_mask=0} (match-all).
5. ro.count = 1.
6. ro.loopback = true; ro.recv_own_msgs = false.
7. ro.fd_frames = false; ro.xl_frames = false.

`Raw::bind(sock, addr, len) -> Result<()>`:
1. addr_can = (struct sockaddr_can*) addr.
2. ifindex = addr_can.can_ifindex.
3. lock(sk).
4. If ro.bound:
   - Per-filter Can::rx_unregister(old-iface, ro.filter[i]).
5. For each filter:
   - Can::rx_register(net, ifindex, filter.can_id, filter.can_mask, raw_rcv, sk, "raw", sk).
6. ro.ifindex = ifindex; ro.bound = true.
7. unlock.

`Raw::rcv(skb, data)`:
1. sk = data; ro = (RawSock*)sk.
2. /* Filter sanity (already done by AF_CAN level) */
3. /* CAN_RAW_FD_FRAMES gate */
4. if (skb.is_fd ∧ !ro.fd_frames): return.
5. nskb = skb_clone(skb, GFP_ATOMIC).
6. /* Build sockaddr_can msg metadata */
7. skb_queue_tail(&sk.sk_receive_queue, nskb).
8. sk.sk_data_ready(sk).

`Raw::sendmsg(sk, msg, size) -> Result<usize>`:
1. ro = (RawSock*)sk.
2. dev = dev_get_by_index(net, ro.ifindex)?
3. skb = sock_alloc_send_skb(sk, size + sizeof(can_skb_priv)).
4. memcpy_from_msg(skb_put(skb, size), msg, size).
5. skb.dev = dev; skb.protocol = htons(ETH_P_CAN) ∨ ETH_P_CANFD.
6. err = can_send(skb, ro.loopback)?
7. Return size.

`Raw::setsockopt(sock, level, optname, optval, optlen) -> Result<()>`:
1. switch optname:
   - CAN_RAW_FILTER:
     - Allocate new filter array; copy_from_user.
     - lock(sk); per-old-filter unregister; per-new-filter register.
     - ro.filter = new; unlock.
   - CAN_RAW_ERR_FILTER: ro.err_mask = data.
   - CAN_RAW_LOOPBACK: ro.loopback = data.
   - CAN_RAW_RECV_OWN_MSGS: ro.recv_own_msgs = data.
   - CAN_RAW_FD_FRAMES: ro.fd_frames = data.
   - CAN_RAW_XL_FRAMES: ro.xl_frames = data.
   - CAN_RAW_JOIN_FILTERS: ro.join_filters = data.

`Raw::release(sock) -> Result<()>`:
1. ro = (RawSock*)sk.
2. If ro.bound:
   - per-filter Can::rx_unregister.
3. sock.sk = NULL.
4. sock_orphan(sk).
5. sock_put(sk).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `proto_eq_can_raw` | INVARIANT | per-create: proto == CAN_RAW. |
| `filter_count_le_max` | INVARIANT | ro.count ≤ CAN_RAW_FILTER_MAX. |
| `bound_iface_valid` | INVARIANT | ro.ifindex valid OR 0 (any). |
| `fd_frames_required_for_fd_skb` | INVARIANT | per-rcv: skb.is_fd ⟹ ro.fd_frames. |
| `loopback_default_true` | INVARIANT | per-init: ro.loopback == true. |

### Layer 2: TLA+

`net/can/raw.tla`:
- Per-socket bind / filter / send / rcv lifecycle.
- Properties:
  - `safety_filter_match_pre_deliver` — per-rcv: filter matched.
  - `safety_no_fd_to_non_fd_socket` — per-FD-frame: only delivered to FD-enabled.
  - `liveness_per_bound_eventually_receives` — per-RX matching frame: socket eventually delivers.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Raw::bind` post: per-filter registered with AF_CAN | `Raw::bind` |
| `Raw::rcv` post: skb cloned + queued; data-ready signaled | `Raw::rcv` |
| `Raw::sendmsg` post: skb on iface tx-queue; loopback per-policy | `Raw::sendmsg` |
| `Raw::setsockopt CAN_RAW_FILTER` post: old filters unregistered; new registered | `Raw::setsockopt` |

### Layer 4: Verus/Creusot functional

`Per-CAN_RAW socket bind + filter set + send/recv ↔ per-iface CAN-frame egress/ingress` semantic equivalence: per-Documentation/networking/can.rst.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

CAN_RAW-specific reinforcement:

- **Per-filter count cap (CAN_RAW_FILTER_MAX)** — defense against per-filter-bomb.
- **Per-iface filter unregister on rebind** — defense against per-stale callback.
- **Per-FD-frame gated by fd_frames sockopt** — defense against per-non-FD socket dropping silently.
- **Per-loopback echoback policy** — defense against per-self-tx duplicate.
- **Per-skb_clone bounded** — defense against per-rcv OOM.
- **Per-namespace AF_CAN scoped** — defense against cross-ns leak.
- **Per-ifindex 0 = wildcard** — defense against per-unbind silent skb loss.
- **Per-rcu free of CanRcv** — defense against per-callback UAF.
- **Per-can_id mask check** — defense against per-malformed mask spurious-match.
- **Per-filter join (AND) opt-in** — defense against per-default OR semantics surprise.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/can/af_can.c family core (covered separately if expanded)
- net/can/bcm.c (broadcast manager; covered separately)
- net/can/isotp.c (ISO-TP; covered separately)
- net/can/j1939/ (covered separately)
- Per-CAN-driver (vcan, mcp251x, etc.; covered separately)
- Implementation code
