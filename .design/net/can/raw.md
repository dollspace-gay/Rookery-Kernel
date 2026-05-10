# Tier-3: net/can/raw.c — CAN_RAW protocol (Controller Area Network raw socket)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/can/00-overview.md
upstream-paths:
  - net/can/raw.c (~1142 lines)
  - net/can/af_can.c (AF_CAN family core: can_rx_register / can_rx_unregister / can_send)
  - include/uapi/linux/can.h (struct can_frame / canfd_frame / canxl_frame / sockaddr_can)
  - include/uapi/linux/can/raw.h (CAN_RAW_* sockopt names + struct can_raw_vcid_options)
  - include/linux/can/skb.h (struct can_skb_ext + can_skb_ext_add + can_is_can{,fd,xl}_skb)
  - include/linux/can/core.h (can_send + can_rx_register/unregister)
  - include/linux/can/can-ml.h (per-iface multi-listener bookkeeping)
-->

## Summary

CAN_RAW is the lowest-level SocketCAN protocol — a per-socket binding to one or all CAN network interfaces, with **direct CAN-frame send/receive** and a userspace-installed **filter list** of `(can_id, can_mask)` pairs. Per-`raw_sock` carries: the bound `ifindex` + tracked `struct net_device *dev`, the per-socket filter array (single-filter is stored inline in `dfilter` to avoid heap alloc on the common case; multi-filter uses `kmalloc`'d `filter[]`), `err_mask` (subscribed bits of the synthesized error-frame), one-bit options (`loopback`, `recv_own_msgs`, `fd_frames`, `xl_frames`, `join_filters`), CAN-XL VCID options (`raw_vcid_opts` + shifted hot-path copies), a per-CPU `uniqframe` cache for OR/AND-mode filter de-duplication, a `notifier` list link for NETDEV_DOWN/UNREGISTER, and a `dev_tracker` for the bound device. The send path (`raw_sendmsg`) supports classical CAN (`CAN_MTU` = 16), CAN-FD (`CANFD_MTU` = 72), and CAN-XL (`CANXL_MTU` = up to 2048 + header) — gated by the destination iface's `can_cap_enabled(dev, CAN_CAP_{CC,FD,XL})` plus the socket's `fd_frames`/`xl_frames`. The receive path is a callback (`raw_rcv`) registered with the AF_CAN core via `can_rx_register(net, dev, can_id, can_mask, raw_rcv, sk, "raw", sk)` for each filter pair; AF_CAN dispatches each ingress skb against its per-(net, ifindex, proto) rx-list. Critical for: automotive ECU diagnostic stacks (UDS / OBD-II via ISO-TP-on-RAW), `candump` / `cansend` (can-utils), industrial PLC (J1939 stacks layered on raw), and `python-can` userspace.

This Tier-3 covers `raw.c` (~1142 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct raw_sock` | per-socket state (embeds `struct sock` at offset 0) | `RawSock` |
| `struct uniqframe` (per-CPU) | per-CPU de-dup cache (skb pointer + hash + join_rx_count) | `UniqFrame` |
| `raw_module_init()` / `raw_module_exit()` | per-module init/exit | `Raw::module_init` / `module_exit` |
| `raw_init(sk)` | per-socket post-create init (per `can_proto.init`) | `Raw::init` |
| `raw_release(sock)` | per-close: detach filters, drop dev ref, sock_put | `Raw::release` |
| `raw_bind(sock, uaddr, len)` | bind to `(can_family=AF_CAN, can_ifindex)` | `Raw::bind` |
| `raw_getname(sock, uaddr, peer)` | getsockname → bound ifindex | `Raw::getname` |
| `raw_setsockopt(sock, level, optname, optval, optlen)` | per-CAN_RAW_* sockopt | `Raw::setsockopt` |
| `raw_getsockopt(sock, level, optname, opt)` | per-CAN_RAW_* sockopt readback | `Raw::getsockopt` |
| `raw_sendmsg(sock, msg, size)` | per-frame send | `Raw::sendmsg` |
| `raw_recvmsg(sock, msg, size, flags)` | per-frame recv + per-skb cmsg + flags | `Raw::recvmsg` |
| `raw_rcv(oskb, data)` | per-skb input hook (registered with AF_CAN) | `Raw::rcv` |
| `raw_enable_filters` / `raw_disable_filters` | per-filter array register/unregister | `Raw::enable_filters` / `disable_filters` |
| `raw_enable_errfilter` / `raw_disable_errfilter` | per-error-mask register/unregister | `Raw::enable_errfilter` / `disable_errfilter` |
| `raw_enable_allfilters` / `raw_disable_allfilters` | combined per-bind/-rebind | `Raw::enable_allfilters` / `disable_allfilters` |
| `raw_check_txframe(ro, skb, dev)` | per-tx MTU classifier (CAN / FD / XL) | `Raw::check_txframe` |
| `raw_put_canxl_vcid(ro, skb)` | per-tx CAN-XL VCID pass-through / set / clear | `Raw::put_canxl_vcid` |
| `raw_notify(ro, msg, dev)` | per-socket NETDEV_DOWN / NETDEV_UNREGISTER handler | `Raw::notify` |
| `raw_notifier(nb, msg, ptr)` | notifier-chain registrar | `Raw::notifier` |
| `raw_sock_destruct(sk)` | per-sock destructor (free percpu uniq) | `Raw::sock_destruct` |
| `raw_sock_no_ioctlcmd(sock, cmd, arg)` | per-ioctl shim returning -ENOIOCTLCMD | `Raw::no_ioctlcmd` |
| `raw_ops` | `struct proto_ops` for CAN_RAW | static |
| `raw_proto` | `struct proto` (slab + size) | static |
| `raw_can_proto` | `struct can_proto` (registered with AF_CAN) | static |
| `canraw_notifier` | `struct notifier_block` on netdev chain | static |
| `can_rx_register()` / `can_rx_unregister()` | AF_CAN per-filter hookup | `Can::rx_register` / `rx_unregister` |
| `can_send(skb, loop)` | AF_CAN tx (per-iface + optional loopback) | `Can::send` |
| `can_skb_ext_add(skb)` | per-skb side-channel for `can_iif` etc. | `Can::skb_ext_add` |
| `can_is_can_skb` / `can_is_canfd_skb` / `can_is_canxl_skb` | per-skb classifier | UAPI |
| `can_cap_enabled(dev, CAN_CAP_{CC,FD,XL,RO})` | per-iface capability gate | shared |
| `struct can_filter { can_id, can_mask }` | per-filter (with `CAN_INV_FILTER` flip-bit) | UAPI |
| `CAN_RAW_FILTER` (1) | per-sockopt filter list | UAPI |
| `CAN_RAW_ERR_FILTER` (2) | per-error-frame bitmap | UAPI |
| `CAN_RAW_LOOPBACK` (3) | per-tx echoback to local sockets | UAPI |
| `CAN_RAW_RECV_OWN_MSGS` (4) | per-self-tx visible to own recv | UAPI |
| `CAN_RAW_FD_FRAMES` (5) | per-CAN-FD frame enable | UAPI |
| `CAN_RAW_JOIN_FILTERS` (6) | per-AND filter mode | UAPI |
| `CAN_RAW_XL_FRAMES` (7) | per-CAN-XL frame enable (implies FD) | UAPI |
| `CAN_RAW_XL_VCID_OPTS` (8) | per-CAN-XL VCID set/pass/filter | UAPI |
| `CAN_RAW_FILTER_MAX` (512) | per-filter-count upper bound | UAPI |

## Compatibility contract

REQ-1: `struct raw_sock` layout (`raw.c:84-105`):
- `sk: struct sock` (MUST be first field — `raw_sk(sk) := (struct raw_sock *)sk`).
- `dev: *net_device` (bound iface, NULL if ifindex==0 wildcard).
- `dev_tracker: netdevice_tracker` (for `netdev_hold` / `netdev_put`).
- `notifier: list_head` (link on global `raw_notifier_list`).
- `ifindex: i32` (0 = wildcard).
- Bitfields: `bound:1`, `loopback:1`, `recv_own_msgs:1`, `fd_frames:1`, `xl_frames:1`, `join_filters:1`.
- `raw_vcid_opts: struct can_raw_vcid_options` (UAPI struct: `flags`, `tx_vcid`, `rx_vcid`, `rx_vcid_mask`).
- `tx_vcid_shifted` / `rx_vcid_shifted` / `rx_vcid_mask_shifted: canid_t` (precomputed `<< CANXL_VCID_OFFSET` for hot path).
- `err_mask: can_err_mask_t` (subscribed CAN_ERR_* bits).
- `count: i32` (active filter count).
- `dfilter: struct can_filter` (inline single-filter to skip kmalloc).
- `filter: *can_filter` (either `&dfilter` when count==1 or kmalloc'd when count>1).
- `uniq: *struct uniqframe __percpu` (per-CPU de-dup cache).

REQ-2: `raw_init(sk)` (per `can_proto.init`):
- `ro->bound = 0; ro->ifindex = 0; ro->dev = NULL`.
- /* Default filter = match-all */ `ro->dfilter.can_id = 0; ro->dfilter.can_mask = MASK_ALL (=0)`. `ro->filter = &ro->dfilter; ro->count = 1`.
- /* Default flags */ `ro->loopback = 1` (per-Documentation/networking/can.rst default), `ro->recv_own_msgs = 0`, `ro->fd_frames = 0`, `ro->xl_frames = 0`, `ro->join_filters = 0`.
- `ro->uniq = alloc_percpu(struct uniqframe)`; -ENOMEM if fail.
- `sk->sk_destruct = raw_sock_destruct`.
- spin_lock(`raw_notifier_lock`); list_add_tail(`&ro->notifier`, `&raw_notifier_list`); spin_unlock.

REQ-3: `raw_release(sock)` (per `proto_ops.release`):
- /* Remove self from notifier list; wait if a notifier-walk currently has a pointer to us (`raw_busy_notifier == ro`) — `schedule_timeout_uninterruptible(1)` poll. */
- rtnl_lock; lock_sock.
- If bound: `raw_disable_allfilters(dev_net(ro->dev), ro->dev, sk)` + `netdev_put(ro->dev, &ro->dev_tracker)` (or `..(net, NULL, sk)` if wildcard).
- If `count > 1`: `kfree(ro->filter)`.
- Zero out ifindex/bound/dev/count.
- `sock_orphan(sk); sock->sk = NULL`.
- release_sock; rtnl_unlock; `sock_prot_inuse_add(net, sk_prot, -1)`; `sock_put(sk)`.

REQ-4: `raw_bind(sock, uaddr, len)`:
- `len ≥ RAW_MIN_NAMELEN` (= `CAN_REQUIRED_SIZE(struct sockaddr_can, can_ifindex)`); else -EINVAL.
- `addr->can_family == AF_CAN`; else -EINVAL.
- rtnl_lock; lock_sock.
- If already bound to same ifindex: idempotent ok.
- If `addr->can_ifindex != 0`:
  - `dev = dev_get_by_index(sock_net(sk), addr->can_ifindex)`; -ENODEV if NULL.
  - `dev->type == ARPHRD_CAN`; else -ENODEV.
  - If `!(dev->flags & IFF_UP)`: `notify_enetdown = 1` (deferred sk_err=ENETDOWN after unlocks).
  - `ifindex = dev->ifindex`.
  - `raw_enable_allfilters(sock_net(sk), dev, sk)` — registers per-filter + err_mask with AF_CAN; rolls back on partial fail.
- Else (`addr->can_ifindex == 0`, wildcard): `ifindex = 0`; `raw_enable_allfilters(sock_net(sk), NULL, sk)` (registered on every CAN iface).
- On success: if already bound, disable old filters first (avoid period with no filters); then `ro->ifindex = ifindex; ro->bound = 1; ro->dev = dev`; `if (dev) netdev_hold(dev, &ro->dev_tracker, GFP_KERNEL)`.
- `dev_put(dev)` (drop ref from `dev_get_by_index`, regardless of bind outcome — the `netdev_hold` above is the kept reference).
- release_sock; rtnl_unlock.
- If notify_enetdown: `sk->sk_err = ENETDOWN`; `sk_error_report(sk)` (deliver via signal/poll).

REQ-5: `raw_rcv(oskb, data)` (per-skb input callback, called by AF_CAN per matched filter):
- `sk = data; ro = raw_sk(sk)`.
- /* CAN_RAW_RECV_OWN_MSGS gate */ `if (!ro->recv_own_msgs && oskb->sk == sk) return` (echo of own tx).
- /* CAN_RAW_FD_FRAMES gate */ `if (!ro->fd_frames && can_is_canfd_skb(oskb)) return`.
- /* CAN-XL gate + VCID filter */ If `can_is_canxl_skb(oskb)`:
  - If `!ro->xl_frames`: return.
  - If `raw_vcid_opts.flags & CAN_RAW_XL_VCID_RX_FILTER`: require `(cxl->prio & rx_vcid_mask_shifted) == (rx_vcid_shifted & rx_vcid_mask_shifted)`; else drop.
  - Else (no VCID filter): drop any frame with `cxl->prio & CANXL_VCID_MASK` (no VCID-tagged frame forwarded without explicit opt-in).
- /* OR / AND filter de-dup via per-CPU uniqframe */
  - If `this_cpu_ptr(ro->uniq)->skb == oskb && ->hash == oskb->hash`:
    - If `!ro->join_filters`: drop (already delivered this skb to this socket via an earlier-matched filter).
    - Else (`join_filters`): `this_cpu_inc(ro->uniq->join_rx_count)`; drop until `join_rx_count >= ro->count` (i.e. **all** filters matched).
  - Else (new skb): record skb + hash; `join_rx_count = 1`; if `join_filters && ro->count > 1`: drop first encounter (still need ≥ ro->count matches).
- `skb = skb_clone(oskb, GFP_ATOMIC)`; drop if NULL.
- Stash `sockaddr_can` (can_family=AF_CAN, can_ifindex=`skb->dev->ifindex`) in `skb->cb[]` for `raw_recvmsg`.
- Stash per-skb raw_flags: `MSG_DONTROUTE` if `oskb->sk` (some socket sent it); `MSG_CONFIRM` if `oskb->sk == sk` (we sent it ourselves and `recv_own_msgs` is on).
- `sock_queue_rcv_skb_reason(sk, skb)`; on failure → `sk_skb_reason_drop(sk, skb, reason)`.

REQ-6: `raw_sendmsg(sock, msg, size)`:
- `CANXL_HDR_SIZE + CANXL_MIN_DLEN ≤ size ≤ CANXL_MTU` (= 12 + 1 = 13 ≤ size ≤ 2048+12); else -EINVAL.
- If `msg->msg_name`: parse `sockaddr_can`, require `msg_namelen ≥ RAW_MIN_NAMELEN` + `can_family == AF_CAN`; `ifindex = addr->can_ifindex`. Else `ifindex = ro->ifindex`.
- `dev = dev_get_by_index(sock_net(sk), ifindex)`; -ENXIO if NULL.
- `can_cap_enabled(dev, CAN_CAP_RO) ⟹ -EACCES` (listen-only iface).
- `skb = sock_alloc_send_skb(sk, size, msg->msg_flags & MSG_DONTWAIT, &err)`.
- `csx = can_skb_ext_add(skb)` (side-channel for `can_iif`); -ENOMEM if fail.
- `csx->can_iif = dev->ifindex`.
- `memcpy_from_msg(skb_put(skb, size), msg, size)` — fills user payload.
- /* Classify + validate against device capability */
  - `txmtu = raw_check_txframe(ro, skb, dev)`:
    - classical: `can_is_can_skb(skb) && can_cap_enabled(dev, CAN_CAP_CC) ⟹ CAN_MTU` (16).
    - FD: `ro->fd_frames && can_is_canfd_skb(skb) && can_cap_enabled(dev, CAN_CAP_FD) ⟹ CANFD_MTU` (72).
    - XL: `ro->xl_frames && can_is_canxl_skb(skb) && can_cap_enabled(dev, CAN_CAP_XL) ⟹ CANXL_MTU` (≤ 2060).
    - else 0 → -EINVAL.
- If `txmtu == CANXL_MTU`: `raw_put_canxl_vcid(ro, skb)` (sanitize non-XL bits; clear VCID unless `CAN_RAW_XL_VCID_TX_PASS`; set VCID if `CAN_RAW_XL_VCID_TX_SET`).
- `sockcm_init(&sockc, sk)`; if `msg->msg_controllen`: `sock_cmsg_send(sk, msg, &sockc)` (parses SCM_TXTIME, SO_MARK, SO_PRIORITY).
- Stamp `skb->dev = dev`, `skb->priority = sockc.priority`, `skb->mark = sockc.mark`, `skb->tstamp = sockc.transmit_time`. `skb_setup_tx_timestamp(skb, &sockc)`.
- `err = can_send(skb, ro->loopback)` — AF_CAN core hands it to the iface's tx queue and, if loopback set, echobacks to local-bound sockets.
- `dev_put(dev)`. Return `size` on success.

REQ-7: `raw_recvmsg(sock, msg, size, flags)`:
- If `flags & MSG_ERRQUEUE`: `sock_recv_errqueue(sk, msg, size, SOL_CAN_RAW, SCM_CAN_RAW_ERRQUEUE)` (delivers TX-timestamp + ENOBUFS error queue).
- `skb = skb_recv_datagram(sk, flags, &err)`; return err if NULL.
- If `size < skb->len`: `msg->msg_flags |= MSG_TRUNC`; else `size = skb->len`.
- `memcpy_to_msg(msg, skb->data, size)`.
- `sock_recv_cmsgs(msg, sk, skb)` (timestamps, dropmonitor).
- If `msg->msg_name`: copy back the stashed `sockaddr_can` (ifindex + family) from `skb->cb`; `msg_namelen = RAW_MIN_NAMELEN`.
- `msg->msg_flags |= *raw_flags(skb)` (MSG_DONTROUTE / MSG_CONFIRM stashed by raw_rcv).
- `skb_free_datagram(sk, skb)`. Return size.

REQ-8: `raw_setsockopt(sock, level=SOL_CAN_RAW, optname, optval, optlen)`:
- `level != SOL_CAN_RAW ⟹ -EINVAL`.
- `CAN_RAW_FILTER`:
  - `optlen % sizeof(can_filter) == 0`; `optlen ≤ CAN_RAW_FILTER_MAX * sizeof(can_filter)` (512 filters); else -EINVAL.
  - `count = optlen / sizeof(can_filter)`.
  - If `count > 1`: `filter = memdup_sockptr(optval, optlen)` (kmalloc dup).
  - If `count == 1`: `copy_from_sockptr(&sfilter, optval, sizeof(sfilter))`.
  - rtnl_lock; lock_sock.
  - If bound + dev: validate `dev->reg_state == NETREG_REGISTERED`; else -ENODEV.
  - If bound: `raw_enable_filters(net, dev, sk, new, count)`; on success `raw_disable_filters(net, dev, sk, ro->filter, ro->count)`.
  - If `ro->count > 1`: kfree old `ro->filter`.
  - If `count == 1`: `ro->dfilter = sfilter; filter = &ro->dfilter`.
  - `ro->filter = filter; ro->count = count`.
- `CAN_RAW_ERR_FILTER`:
  - `optlen == sizeof(can_err_mask_t)`.
  - `err_mask &= CAN_ERR_MASK`.
  - If bound + dev: `dev->reg_state == NETREG_REGISTERED`.
  - If bound: `raw_enable_errfilter(net, dev, sk, err_mask)` (registers `(can_id=0, mask=err_mask|CAN_ERR_FLAG)`); on success disable old.
  - `ro->err_mask = err_mask`.
- `CAN_RAW_LOOPBACK`: `optlen == sizeof(int)`; `ro->loopback = !!flag`.
- `CAN_RAW_RECV_OWN_MSGS`: `optlen == sizeof(int)`; `ro->recv_own_msgs = !!flag`.
- `CAN_RAW_FD_FRAMES`: `optlen == sizeof(int)`. If `ro->xl_frames && !flag`: -EINVAL (XL requires FD). `ro->fd_frames = !!flag`.
- `CAN_RAW_XL_FRAMES`: `optlen == sizeof(int)`; `ro->xl_frames = !!flag`. If enabling XL: `ro->fd_frames = ro->xl_frames` (XL implies FD).
- `CAN_RAW_XL_VCID_OPTS`: `optlen == sizeof(raw_vcid_opts)`; copy whole struct; precompute `tx_vcid_shifted` = `raw_vcid_opts.tx_vcid << CANXL_VCID_OFFSET`, same for rx.
- `CAN_RAW_JOIN_FILTERS`: `optlen == sizeof(int)`; `ro->join_filters = !!flag`.
- default: -ENOPROTOOPT.

REQ-9: `raw_getsockopt(sock, level, optname, opt)`:
- `level != SOL_CAN_RAW ⟹ -EINVAL`.
- `CAN_RAW_FILTER`: `lock_sock`; if `ro->count > 0`: if user buffer smaller than `count * sizeof(can_filter)` → -ERANGE + write needed `opt->optlen`; else `copy_to_iter(ro->filter, len, &opt->iter_out)`. Release sock.
- `CAN_RAW_ERR_FILTER` / `CAN_RAW_LOOPBACK` / `CAN_RAW_RECV_OWN_MSGS` / `CAN_RAW_FD_FRAMES` / `CAN_RAW_XL_FRAMES` / `CAN_RAW_JOIN_FILTERS`: copy single value (with length clipping to the field size).
- `CAN_RAW_XL_VCID_OPTS`: same length-handshake as FILTER (ERANGE on short user buffer).

REQ-10: `raw_notify(ro, msg, dev)` (per-socket NETDEV notifier handler):
- Skip if `!net_eq(dev_net(dev), sock_net(sk))` or `ro->dev != dev`.
- `NETDEV_UNREGISTER`: lock_sock; if bound: `raw_disable_allfilters` + `netdev_put`; if `count > 1` kfree filter; zero ifindex/bound/dev/count; release_sock; `sk->sk_err = ENODEV` + sk_error_report.
- `NETDEV_DOWN`: `sk->sk_err = ENETDOWN` + sk_error_report.

REQ-11: `raw_notifier(nb, msg, ptr)` (notifier chain callback):
- `dev->type == ARPHRD_CAN`; else NOTIFY_DONE.
- `msg ∈ {NETDEV_UNREGISTER, NETDEV_DOWN}`; else NOTIFY_DONE.
- `raw_busy_notifier == NULL` (re-entrancy guard).
- spin_lock(`raw_notifier_lock`); for each ro on `raw_notifier_list`: store ro in `raw_busy_notifier` (so `raw_release` can wait for us), unlock, call `raw_notify(ro, msg, dev)`, relock. `raw_busy_notifier = NULL`; unlock.

REQ-12: Filter registration with AF_CAN (`raw_enable_filters`):
- For i in 0..count: `can_rx_register(net, dev, filter[i].can_id, filter[i].can_mask, raw_rcv, sk, "raw", sk)`. On any failure: `can_rx_unregister(..)` everything successfully registered so far, return err.

REQ-13: Error-filter registration (`raw_enable_errfilter`):
- If `err_mask != 0`: `can_rx_register(net, dev, /*can_id=*/0, /*mask=*/err_mask | CAN_ERR_FLAG, raw_rcv, sk, "raw", sk)`.

REQ-14: `raw_sock_destruct(sk)`:
- `free_percpu(ro->uniq)`.
- `can_sock_destruct(sk)`.

REQ-15: MTU rules (per `raw_check_txframe`):
- classical CAN: `CAN_MTU` = 16 (8-byte payload); requires `CAN_CAP_CC`.
- CAN-FD: `CANFD_MTU` = 72 (64-byte payload); requires socket `fd_frames` ∧ `CAN_CAP_FD`.
- CAN-XL: `CANXL_MTU` = 2060 (12 B header + up to 2048-byte payload); requires socket `xl_frames` ∧ `CAN_CAP_XL`. XL also implies FD.
- ifindex must be a CAN device (`dev->type == ARPHRD_CAN`).
- Read-only iface (`CAN_CAP_RO`): tx returns -EACCES.

REQ-16: `proto_ops raw_ops`:
- `.family = PF_CAN`, `.release = raw_release`, `.bind = raw_bind`, `.connect = sock_no_connect`, `.socketpair = sock_no_socketpair`, `.accept = sock_no_accept`, `.getname = raw_getname`, `.poll = datagram_poll`, `.ioctl = raw_sock_no_ioctlcmd`, `.gettstamp = sock_gettstamp`, `.listen = sock_no_listen`, `.shutdown = sock_no_shutdown`, `.setsockopt = raw_setsockopt`, `.getsockopt = raw_getsockopt`, `.sendmsg = raw_sendmsg`, `.recvmsg = raw_recvmsg`, `.mmap = sock_no_mmap`.

REQ-17: `struct proto raw_proto`:
- `.name = "CAN_RAW"`, `.owner = THIS_MODULE`, `.obj_size = sizeof(struct raw_sock)`, `.init = raw_init`.

REQ-18: `struct can_proto raw_can_proto`:
- `.type = SOCK_RAW`, `.protocol = CAN_RAW`, `.ops = &raw_ops`, `.prot = &raw_proto`.
- Registered via `can_proto_register(&raw_can_proto)` in `raw_module_init`.

REQ-19: Module init/exit:
- `raw_module_init`: `pr_info("can: raw protocol")`. `can_proto_register(&raw_can_proto)`. `register_netdevice_notifier(&canraw_notifier)`.
- `raw_module_exit`: reverse — unregister notifier; `can_proto_unregister`.

REQ-20: Per-namespace:
- All `dev_get_by_index` / `dev_net` / `sock_net` calls go through the socket's network namespace; CAN_RAW is per-net.

## Acceptance Criteria

- [ ] AC-1: `socket(AF_CAN, SOCK_RAW, CAN_RAW)` succeeds in any user-ns; `raw_init` runs; `ro.loopback = 1`, single default match-all filter.
- [ ] AC-2: `bind(sk, { AF_CAN, ifindex=vcan0 }, sizeof(sockaddr_can))` succeeds; default match-all filter registered with AF_CAN for vcan0.
- [ ] AC-3: bind with `ifindex=0`: wildcard; `raw_enable_allfilters(net, NULL, sk)` registers on every existing AF_CAN iface.
- [ ] AC-4: bind to non-CAN iface (`dev->type != ARPHRD_CAN`) returns -ENODEV.
- [ ] AC-5: `cansend vcan0 123#DEADBEEF`: classical CAN frame transmitted; recv-loopback echos to other subscribers on vcan0.
- [ ] AC-6: `setsockopt(CAN_RAW_FILTER, {can_id=0x123, can_mask=0x7FF})`: only frames with id 0x123 are delivered; 0x124 is dropped (per-(id & mask)==(filter.can_id & mask)).
- [ ] AC-7: `CAN_RAW_JOIN_FILTERS = 1` with two filters {0x123, 0x7FF} + {0x124, 0x7FF}: no frame ever delivered (AND-mode requires both to match; impossible for a single id) — per-spec.
- [ ] AC-8: `CAN_RAW_FD_FRAMES = 1` + CAN-FD-capable iface: 64-byte payload tx + rx works; non-FD socket on same iface drops FD frame silently in `raw_rcv`.
- [ ] AC-9: `CAN_RAW_XL_FRAMES = 1`: 2048-byte payload tx + rx works on `CAN_CAP_XL` iface; auto-enables `fd_frames`. Trying `CAN_RAW_FD_FRAMES = 0` while `xl_frames` set → -EINVAL.
- [ ] AC-10: `CAN_RAW_LOOPBACK = 0`: tx not echobacked to other local sockets; remote receivers still get it via the wire.
- [ ] AC-11: `CAN_RAW_RECV_OWN_MSGS = 1` + loopback=1: own tx delivered back to same socket with MSG_CONFIRM in `msg_flags`.
- [ ] AC-12: `CAN_RAW_ERR_FILTER = CAN_ERR_BUSOFF | CAN_ERR_CRTL`: synthesized error frames delivered for bus-off + controller events.
- [ ] AC-13: `CAN_RAW_FILTER` count > `CAN_RAW_FILTER_MAX` (512): -EINVAL.
- [ ] AC-14: NETDEV_DOWN on bound iface: `sk_err = ENETDOWN`; poll/read wakes.
- [ ] AC-15: NETDEV_UNREGISTER: filters detached; `sk_err = ENODEV`; subsequent send returns -ENODEV (no dev).
- [ ] AC-16: `close(sk)`: filters detached; `dev_put` releases iface ref; race with concurrent notifier-walk handled by `raw_busy_notifier` poll loop.
- [ ] AC-17: `getsockopt(CAN_RAW_FILTER, buf, &len)` with `len < ro->count * sizeof(can_filter)`: returns -ERANGE and writes needed length to `optlen` (userspace re-sizes + retries).
- [ ] AC-18: `sendmsg` on `CAN_CAP_RO` (read-only/listen-only) iface: -EACCES.
- [ ] AC-19: per-CPU `uniqframe`: socket with 4 filters, all matching one skb in OR-mode: receives skb exactly once (not 4×); AND-mode: requires all 4 to match.
- [ ] AC-20: `candump vcan0` (can-utils) + `cansend vcan0 123#DEAD` end-to-end works with vcan + this driver compiled.

## Architecture

```
struct UniqFrame {                       // per-CPU
  skb:            *const SkBuff,
  hash:           u32,
  join_rx_count:  u32,
}

struct RawSock {
  sk:            Sock,                   // MUST be first
  dev:           Option<*NetDevice>,
  dev_tracker:   NetdeviceTracker,
  notifier:      ListHead,               // on global raw_notifier_list
  ifindex:       i32,
  bound:           bool,
  loopback:        bool,                 // default true
  recv_own_msgs:   bool,                 // default false
  fd_frames:       bool,                 // default false
  xl_frames:       bool,                 // default false; implies fd_frames
  join_filters:    bool,                 // default false (OR mode)
  raw_vcid_opts: CanRawVcidOptions,      // UAPI struct
  tx_vcid_shifted:      CanId,
  rx_vcid_shifted:      CanId,
  rx_vcid_mask_shifted: CanId,
  err_mask:      CanErrMask,
  count:         i32,
  dfilter:       CanFilter,              // inline single-filter
  filter:        *mut CanFilter,         // either &dfilter or kmalloc'd
  uniq:          PerCpu<UniqFrame>,
}

struct CanFilter {                       // UAPI
  can_id:   __u32,                       // top bits: CAN_INV_FILTER / CAN_EFF_FLAG / CAN_RTR_FLAG / CAN_ERR_FLAG
  can_mask: __u32,
}

const SOL_CAN_RAW:       i32 = 101;
const CAN_RAW_FILTER:    i32 = 1;
const CAN_RAW_ERR_FILTER:i32 = 2;
const CAN_RAW_LOOPBACK:  i32 = 3;
const CAN_RAW_RECV_OWN_MSGS: i32 = 4;
const CAN_RAW_FD_FRAMES: i32 = 5;
const CAN_RAW_JOIN_FILTERS: i32 = 6;
const CAN_RAW_XL_FRAMES: i32 = 7;
const CAN_RAW_XL_VCID_OPTS: i32 = 8;
const CAN_RAW_FILTER_MAX: usize = 512;
const CAN_MTU:    usize = 16;
const CANFD_MTU:  usize = 72;
const CANXL_MTU:  usize = 2060;
```

`Raw::init(sk) -> Result<()>`:
1. ro = raw_sk(sk).
2. ro.bound = false; ro.ifindex = 0; ro.dev = None.
3. ro.dfilter = CanFilter { can_id: 0, can_mask: 0 } (= MASK_ALL).
4. ro.filter = &ro.dfilter; ro.count = 1.
5. ro.loopback = true; ro.recv_own_msgs = false; ro.fd_frames = false; ro.xl_frames = false; ro.join_filters = false.
6. ro.uniq = alloc_percpu(UniqFrame)?
7. sk.sk_destruct = Raw::sock_destruct.
8. spin_lock(RAW_NOTIFIER_LOCK); list_add_tail(&ro.notifier, &RAW_NOTIFIER_LIST); spin_unlock.

`Raw::bind(sock, addr: SockaddrCan, len) -> Result<()>`:
1. len ≥ RAW_MIN_NAMELEN; addr.can_family == AF_CAN.
2. rtnl_lock(); lock_sock(sk).
3. If ro.bound ∧ addr.can_ifindex == ro.ifindex: ok (idempotent).
4. If addr.can_ifindex != 0:
   - dev = dev_get_by_index(sock_net(sk), addr.can_ifindex)?
   - dev.type == ARPHRD_CAN, else -ENODEV.
   - notify_enetdown = !(dev.flags & IFF_UP).
   - ifindex = dev.ifindex.
   - Raw::enable_allfilters(sock_net(sk), Some(dev), sk)?
5. Else: ifindex = 0; Raw::enable_allfilters(sock_net(sk), None, sk)?
6. If ro.bound: detach old (per-old-dev).
7. ro.ifindex = ifindex; ro.bound = true; ro.dev = dev.
8. If ro.dev.is_some(): netdev_hold(ro.dev, &ro.dev_tracker, GFP_KERNEL).
9. dev_put(dev) (always drop the dev_get_by_index ref; netdev_hold above is the kept one).
10. release_sock; rtnl_unlock.
11. If notify_enetdown: sk.sk_err = ENETDOWN; sk_error_report(sk).

`Raw::rcv(oskb, data)`:
1. sk = data; ro = raw_sk(sk).
2. /* RECV_OWN_MSGS */ if !ro.recv_own_msgs && oskb.sk == sk: return.
3. /* FD gate */ if !ro.fd_frames && can_is_canfd_skb(oskb): return.
4. /* XL gate + VCID */ if can_is_canxl_skb(oskb):
   - if !ro.xl_frames: return.
   - cxl = oskb.data as *CanXlFrame.
   - if ro.raw_vcid_opts.flags & CAN_RAW_XL_VCID_RX_FILTER:
     - if (cxl.prio & ro.rx_vcid_mask_shifted) != (ro.rx_vcid_shifted & ro.rx_vcid_mask_shifted): return.
   - else:
     - if cxl.prio & CANXL_VCID_MASK: return.
5. /* per-CPU dedup */ uniq = this_cpu_ptr(ro.uniq).
   - if uniq.skb == oskb && uniq.hash == oskb.hash:
     - if !ro.join_filters: return.
     - this_cpu_inc(uniq.join_rx_count).
     - if uniq.join_rx_count < ro.count: return.
   - else:
     - uniq.skb = oskb; uniq.hash = oskb.hash; uniq.join_rx_count = 1.
     - if ro.join_filters && ro.count > 1: return.
6. skb = skb_clone(oskb, GFP_ATOMIC)?
7. Stash sockaddr_can in skb.cb: { can_family: AF_CAN, can_ifindex: skb.dev.ifindex }.
8. Stash raw_flags in skb.cb (after sockaddr_can):
   - MSG_DONTROUTE if oskb.sk.is_some.
   - MSG_CONFIRM if oskb.sk == sk.
9. sock_queue_rcv_skb_reason(sk, skb) → if reason: sk_skb_reason_drop(sk, skb, reason).

`Raw::sendmsg(sock, msg, size) -> Result<usize>`:
1. CANXL_HDR_SIZE + CANXL_MIN_DLEN ≤ size ≤ CANXL_MTU, else -EINVAL.
2. If msg.msg_name: parse SockaddrCan; ifindex = addr.can_ifindex.
3. Else: ifindex = ro.ifindex.
4. dev = dev_get_by_index(sock_net(sk), ifindex)?
5. can_cap_enabled(dev, CAN_CAP_RO) ⟹ -EACCES (listen-only).
6. skb = sock_alloc_send_skb(sk, size, msg.msg_flags & MSG_DONTWAIT, &err)?
7. csx = can_skb_ext_add(skb)?; csx.can_iif = dev.ifindex.
8. memcpy_from_msg(skb_put(skb, size), msg, size)?
9. txmtu = Raw::check_txframe(ro, skb, dev); if 0 → -EINVAL.
10. If txmtu == CANXL_MTU: Raw::put_canxl_vcid(ro, skb).
11. sockcm_init(&sockc, sk); if msg.msg_controllen: sock_cmsg_send(sk, msg, &sockc)?
12. skb.dev = dev; skb.priority = sockc.priority; skb.mark = sockc.mark; skb.tstamp = sockc.transmit_time; skb_setup_tx_timestamp(skb, &sockc).
13. err = can_send(skb, ro.loopback); dev_put(dev). Return size on success.

`Raw::setsockopt(sock, level=SOL_CAN_RAW, optname, optval, optlen) -> Result<()>`:
1. level == SOL_CAN_RAW; else -EINVAL.
2. switch optname:
   - CAN_RAW_FILTER:
     - optlen % sizeof(CanFilter) == 0; optlen ≤ CAN_RAW_FILTER_MAX * sizeof(CanFilter).
     - count = optlen / sizeof(CanFilter).
     - If count > 1: filter = memdup_sockptr(optval, optlen).
     - If count == 1: copy_from_sockptr(&sfilter, optval, sizeof(sfilter)).
     - rtnl_lock; lock_sock.
     - If ro.bound + ro.dev: ro.dev.reg_state == NETREG_REGISTERED, else -ENODEV.
     - If ro.bound: enable_filters new; on ok disable_filters old.
     - If ro.count > 1: kfree(ro.filter).
     - If count == 1: ro.dfilter = sfilter; filter = &ro.dfilter.
     - ro.filter = filter; ro.count = count.
   - CAN_RAW_ERR_FILTER: err_mask &= CAN_ERR_MASK; if ro.bound: enable_errfilter new + disable old. ro.err_mask = err_mask.
   - CAN_RAW_LOOPBACK: ro.loopback = !!flag.
   - CAN_RAW_RECV_OWN_MSGS: ro.recv_own_msgs = !!flag.
   - CAN_RAW_FD_FRAMES: if ro.xl_frames && !flag → -EINVAL. ro.fd_frames = !!flag.
   - CAN_RAW_XL_FRAMES: ro.xl_frames = !!flag. If xl_frames: ro.fd_frames = xl_frames.
   - CAN_RAW_XL_VCID_OPTS: copy struct; precompute shifted hot-path values.
   - CAN_RAW_JOIN_FILTERS: ro.join_filters = !!flag.
   - else -ENOPROTOOPT.

`Raw::notify(ro, msg, dev)`:
1. If !net_eq(dev_net(dev), sock_net(sk)) || ro.dev != dev: return.
2. NETDEV_UNREGISTER: lock_sock; if bound: disable_allfilters + netdev_put. If count > 1: kfree filter. Zero ifindex/bound/dev/count. release_sock. sk.sk_err = ENODEV; sk_error_report.
3. NETDEV_DOWN: sk.sk_err = ENETDOWN; sk_error_report.

`Raw::release(sock) -> Result<()>`:
1. ro = raw_sk(sk); net = sock_net(sk).
2. /* Wait if notifier currently has us */ while raw_busy_notifier == ro: spin_unlock; schedule_timeout_uninterruptible(1); spin_lock.
3. list_del(&ro.notifier).
4. rtnl_lock; lock_sock.
5. If ro.bound: disable_allfilters + netdev_put (if dev).
6. If ro.count > 1: kfree(ro.filter).
7. Zero ifindex/bound/dev/count.
8. sock_orphan(sk); sock.sk = None.
9. release_sock; rtnl_unlock.
10. sock_prot_inuse_add(net, sk_prot, -1); sock_put(sk).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sk_first_field` | INVARIANT | per-raw_sock: `struct sock` is at offset 0 (raw_sk cast). |
| `default_filter_match_all` | INVARIANT | per-init: dfilter.can_id == 0 ∧ dfilter.can_mask == 0; ro.filter == &dfilter; ro.count == 1. |
| `filter_count_le_max` | INVARIANT | per-setsockopt CAN_RAW_FILTER: optlen ≤ CAN_RAW_FILTER_MAX*sizeof(CanFilter). |
| `xl_implies_fd` | INVARIANT | per-setsockopt: ro.xl_frames ⟹ ro.fd_frames. |
| `fd_off_with_xl_on_rejected` | INVARIANT | per-setsockopt CAN_RAW_FD_FRAMES = 0 while xl_frames: -EINVAL. |
| `bind_requires_arphrd_can` | INVARIANT | per-bind: dev.type == ARPHRD_CAN. |
| `unbind_releases_dev` | INVARIANT | per-release/notify_UNREGISTER: every netdev_hold balanced by netdev_put. |
| `notifier_reentry_guard` | INVARIANT | per-raw_notifier: raw_busy_notifier == NULL at entry. |
| `release_waits_for_notifier_walk` | LIVENESS | per-release: schedules out while raw_busy_notifier == ro. |
| `percpu_uniq_freed_on_destruct` | INVARIANT | per-sock_destruct: free_percpu(ro.uniq) called exactly once. |
| `fd_skb_dropped_on_non_fd_sock` | INVARIANT | per-rcv: can_is_canfd_skb(oskb) ∧ !ro.fd_frames ⟹ drop. |
| `xl_vcid_filter_obeyed` | INVARIANT | per-rcv: vcid filter applied when CAN_RAW_XL_VCID_RX_FILTER set. |
| `join_filters_requires_all_match` | INVARIANT | per-rcv: join_filters ⟹ delivery only when uniq.join_rx_count ≥ ro.count. |
| `bound_iface_alive_for_tx` | INVARIANT | per-sendmsg: dev_get_by_index returned non-NULL before can_send. |
| `ro_iface_cannot_send` | INVARIANT | per-sendmsg: can_cap_enabled(dev, CAN_CAP_RO) ⟹ -EACCES. |

### Layer 2: TLA+

`net/can/raw.tla`:
- States: socket lifecycle (CREATED, BOUND, UNBOUND), per-filter set, per-iface registration set.
- Actions: create / bind / rebind / setsockopt-filter / setsockopt-mode / sendmsg / rcv / netdev-down / netdev-unregister / release.
- Properties:
  - `safety_filter_set_matches_registrations` — at every state, the set of (can_id, can_mask) pairs in ro.filter equals the set registered with AF_CAN for ro.dev (or for "any" if ifindex==0).
  - `safety_dev_ref_balanced` — number of `netdev_hold` calls equals number of `netdev_put` calls.
  - `safety_no_fd_to_non_fd_socket` — non-FD socket never receives a CAN-FD skb.
  - `safety_no_xl_without_vcid_optin` — VCID-tagged XL frame never delivered without `CAN_RAW_XL_VCID_RX_FILTER` ∨ explicit pass-through.
  - `safety_join_filters_all_match` — join_filters mode delivers iff `join_rx_count ≥ count`.
  - `liveness_per_bound_eventually_receives_matching` — per-RX matching frame on bound iface: socket eventually delivers (modulo OOM-drop).
  - `liveness_release_terminates` — close completes even if notifier walk currently holds raw_busy_notifier.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Raw::init` post: percpu uniq allocated; default match-all filter installed; on notifier list | `Raw::init` |
| `Raw::bind` post: per-filter + err_mask registered with AF_CAN; netdev held; bound==true | `Raw::bind` |
| `Raw::rebind` post: old filters fully unregistered before new committed | `Raw::bind` |
| `Raw::rcv` post: skb cloned + sockaddr_can stashed in cb[] + raw_flags stashed; queued or dropped with reason | `Raw::rcv` |
| `Raw::sendmsg` post: skb on iface tx queue with txmtu ∈ {CAN_MTU, CANFD_MTU, CANXL_MTU}; loopback per ro.loopback | `Raw::sendmsg` |
| `Raw::setsockopt CAN_RAW_FILTER` post: old filters unregistered; new registered; rollback on failure | `Raw::setsockopt` |
| `Raw::setsockopt CAN_RAW_XL_FRAMES` post: xl_frames ⟹ fd_frames | `Raw::setsockopt` |
| `Raw::notify NETDEV_UNREGISTER` post: filters torn down; dev released; sk.sk_err = ENODEV | `Raw::notify` |
| `Raw::release` post: filters detached; dev released; sock orphaned; ref count -1 | `Raw::release` |

### Layer 4: Verus/Creusot functional

`Per-CAN_RAW socket bind + filter set + per-mode (CC/FD/XL) send/recv ↔ per-iface CAN-frame egress/ingress` semantic equivalence: per-`Documentation/networking/can.rst` + per-can-utils `cangen`/`candump` reference behavior.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

CAN_RAW-specific reinforcement:

- **Per-filter count cap (`CAN_RAW_FILTER_MAX = 512`)** — defense against per-userspace-installed filter-bomb DoS.
- **Per-`memdup_sockptr` bounded by `optlen`** — defense against per-kmalloc OOM via huge sockopt.
- **Per-iface filter unregister on rebind / release / NETDEV_UNREGISTER** — defense against per-stale callback delivering after sk freed.
- **Per-`netdev_hold` + `dev_tracker`** — defense against per-bind-then-iface-destroy UAF.
- **Per-`ARPHRD_CAN` type check at bind** — defense against per-bind-to-eth0 + spam-CAN-frames confusion.
- **Per-FD/XL frame gated by `fd_frames`/`xl_frames` sockopt** — defense against per-non-FD-aware userspace receiving truncated 64-byte payloads.
- **Per-`CAN_CAP_RO` listen-only enforcement at tx** — defense against per-write to read-only sniffer iface.
- **Per-`raw_busy_notifier` re-entrancy + release-side wait loop** — defense against per-release-vs-NETDEV_UNREGISTER race UAF.
- **Per-`CAN_RAW_XL_FRAMES ⟹ CAN_RAW_FD_FRAMES` invariant** — defense against per-FD-disabled-XL-enabled inconsistency.
- **Per-`CAN_RAW_XL_VCID_RX_FILTER` opt-in for VCID-tagged frames** — defense against per-VCID-multiplexed cross-domain leak (CAN-XL networks segment by VCID).
- **Per-CPU `uniqframe` de-dup** — defense against per-multi-filter same-skb 4× delivery duplication.
- **Per-`getsockopt CAN_RAW_FILTER` ERANGE re-sizing handshake** — defense against per-userspace-short-buffer truncation silent loss.
- **Per-`copy_from_sockptr` bounded by `optlen` per option's expected size** — defense against per-OOB-stack-write via crafted setsockopt.
- **Per-`can_send` loopback path bounded to local-net sockets only** — defense against per-cross-namespace loop leak.
- **Per-`raw_notifier` `dev->type == ARPHRD_CAN` early-return** — defense against per-eth-flap walking the CAN-raw socket list.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `net/can/af_can.c` family core (covered in `net/can/af-can.md` if expanded — handles `can_rx_register`/`unregister`/`can_send` + the per-(net, ifindex, proto) rx-list hashing)
- `net/can/bcm.c` (Broadcast Manager; covered in `net/can/bcm.md`)
- `net/can/isotp.c` (ISO-TP segmented transport; covered separately)
- `net/can/j1939/` (SAE J1939; covered separately)
- `net/can/gw.c` (CAN gateway; covered separately)
- per-CAN-controller drivers (`drivers/net/can/{vcan,mcp251xfd,m_can,flexcan,...}`)
- `include/linux/can/skb.h` skb side-channel (`can_skb_ext` + tx timestamps) helpers
- Implementation code
