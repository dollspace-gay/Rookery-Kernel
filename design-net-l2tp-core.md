---
title: "Tier-3: net/l2tp/l2tp_core.c — L2TPv2/v3 core (tunnel + session, encap, recv, xmit)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`net/l2tp/l2tp_core.c` implements the kernel-side core of **Layer 2 Tunneling Protocol** versions 2 (RFC 2661) and 3 (RFC 3931). It owns two life-cycled objects: `struct l2tp_tunnel` (one per L2TP control-plane session, riding either a UDP socket or a raw IPPROTO_L2TP socket) and `struct l2tp_session` (one per pseudowire — PPP, Ethernet, etc. — multiplexed onto a tunnel). Per-net storage is `struct l2tp_net` (per-net-namespace IDR for tunnels keyed by `tunnel_id`, two IDRs for v2/v3 sessions, plus a 16-bucket hash table for v3 session-ID collisions). Lookup paths (`l2tp_tunnel_get`, `l2tp_v2_session_get`, `l2tp_v3_session_get`) are RCU-read-locked with `refcount_inc_not_zero`. Receive path: `l2tp_udp_encap_recv` (registered via `udp_sk(sk)->encap_rcv`) parses the L2TP header, dispatches data frames into `l2tp_recv_common`, which handles optional cookies, optional Ns/Nr sequence numbers, optional offset (v2), and the L2TPv3 L2-specific sublayer, then feeds the skb through `l2tp_recv_data_seq` (reorder queue or in-order discard) and `l2tp_recv_dequeue` (per-session reorder pull + private `recv_skb` callback). Transmit path: `l2tp_xmit_skb` → `l2tp_xmit_core` builds the v2/v3 L2TP header, optionally wraps in UDP (`udp_set_csum`), then `ip_queue_xmit` / `inet6_csk_xmit`. Control plane (HELLO/SCCRQ/SCCRP/SCCCN/StopCCN/...) is **userspace's job** — control frames (`T=1`) are passed up to the controlling daemon via the same UDP socket; the kernel only handles data frames here. Tunnel/session deletion is deferred to a global workqueue `l2tp_wq`. Critical for: PPPoL2TP VPN gateways, L2TPv3 pseudowire (Ethernet-over-IP), carrier-grade access concentrators.

This Tier-3 covers `net/l2tp/l2tp_core.c` (~1953 lines).

### Acceptance Criteria

- [ ] AC-1: Tunnel registration: `l2tp_tunnel_register` reserves `tunnel_id` (ENOSPC → -EEXIST), validates socket, installs `udp_sk(sk)->encap_rcv = l2tp_udp_encap_recv` for UDP encap, publishes via `idr_replace`.
- [ ] AC-2: Tunnel lookup: `l2tp_tunnel_get(net, tunnel_id)` returns refcounted tunnel; concurrent delete: `refcount_inc_not_zero` fails ⟹ NULL.
- [ ] AC-3: v2 session lookup: `l2tp_v2_session_get` keys by `(tunnel_id<<16)|session_id`.
- [ ] AC-4: v3 session lookup: `l2tp_v3_session_get` falls back to `l2tp_v3_session_htable` walk on collision.
- [ ] AC-5: IP encap session registration: collision returns -EEXIST (no collision list permitted).
- [ ] AC-6: UDP encap session registration: collision adds session to `coll_list` + hlist.
- [ ] AC-7: UDP rx: control frame (T=1) → return 1 (delivered to userspace).
- [ ] AC-8: UDP rx: data frame, valid session, version match → `l2tp_recv_common`; return 0.
- [ ] AC-9: UDP rx: data frame, no matching session → return 1 (pass to userspace) with UDP header re-pushed.
- [ ] AC-10: `l2tp_recv_common`: cookie mismatch → `rx_cookie_discards++` + drop.
- [ ] AC-11: `l2tp_recv_common`: required-but-absent sequence numbers → `rx_seq_discards++` + drop.
- [ ] AC-12: Reorder enabled (`reorder_timeout != 0`): out-of-order skbs queued and drained when in-order; expired skbs dropped.
- [ ] AC-13: Reorder disabled: OOS skbs dropped; `nr_oos_count > nr_oos_count_max` triggers `reorder_skip` resync.
- [ ] AC-14: `l2tp_xmit_skb`: v2 — `flags|tunnel_id|session_id` BE16, optional Ns/Nr; `send_seq` increments `ns` mod 0x10000.
- [ ] AC-15: `l2tp_xmit_skb`: v3 UDP — `flags|0` BE16/BE16 + `session_id` BE32 + cookie + sublayer; `ns` mod 0x1000000.
- [ ] AC-16: `l2tp_xmit_skb`: UDP encap — UDP header pushed with correct length and checksum (v4 `udp_set_csum`, v6 `udp6_set_csum`); `udp_len > U16_MAX` → drop.
- [ ] AC-17: `l2tp_xmit_skb`: tunnel.fd≥0 and `sk_state != TCP_ESTABLISHED` → drop.
- [ ] AC-18: Tunnel delete: `test_and_set_bit(0, &dead)` idempotent; second call is a no-op.
- [ ] AC-19: `l2tp_tunnel_del_work`: closes all sessions, releases kernel-created socket, removes from IDR, drops two refs.
- [ ] AC-20: Per-net exit: `l2tp_pre_exit_net` deletes all tunnels and flushes wq twice; `l2tp_exit_net` asserts IDRs empty.
- [ ] AC-21: Sequence window: `nr_window_size = nr_max/2`; `l2tp_seq_check_rx_window` accepts within window, rejects outside.

### Architecture

```
const L2TP_HDRFLAG_T: u16 = 0x8000;
const L2TP_HDRFLAG_L: u16 = 0x4000;
const L2TP_HDRFLAG_S: u16 = 0x0800;
const L2TP_HDRFLAG_O: u16 = 0x0200;
const L2TP_HDRFLAG_P: u16 = 0x0100;
const L2TP_HDR_VER_MASK: u16 = 0x000F;
const L2TP_HDR_VER_2: u16 = 0x0002;
const L2TP_HDR_VER_3: u16 = 0x0003;
const L2TP_SLFLAG_S: u32 = 0x40000000;
const L2TP_SL_SEQ_MASK: u32 = 0x00ffffff;
const L2TP_HDR_SIZE_MAX: usize = 14;
const L2TP_DEPTH_NESTING: u32 = 2;

#[repr(u8)]
enum L2tpEncapType { Udp = 0, Ip = 1 }

struct L2tpSkbCb {
  ns: u32,
  has_seq: u16,
  length: u16,
  expires: u64,                  // jiffies
}

struct L2tpNet {
  tunnel_idr_lock: SpinLock<()>,
  tunnel_idr: Idr<u32, *L2tpTunnel>,
  session_idr_lock: SpinLock<()>,
  v2_session_idr: Idr<u32, *L2tpSession>,      // key = (tid << 16) | sid
  v3_session_idr: Idr<u32, *L2tpSession>,      // key = session_id
  v3_session_htable: [HlistHead; 16],          // collision chain
}

struct L2tpSessionCollList {
  lock: SpinLock<()>,
  list: ListHead,                              // -> L2tpSession::clist
  ref_count: AtomicRefcount,
}

struct L2tpTunnel {
  ref_count: AtomicRefcount,
  rcu: RcuHead,
  sock: Option<*Sock>,
  l2tp_net: *Net,
  tunnel_id: u32,
  peer_tunnel_id: u32,
  version: u16,                                // 2 | 3
  encap: L2tpEncapType,
  fd: i32,                                     // -1 = kernel-created socket
  acpt_newsess: bool,
  dead: AtomicBit,
  list_lock: SpinLock<()>,
  session_list: ListHead,
  del_work: WorkStruct,                        // -> L2tpTunnel::del_work_fn
  name: [u8; 20],                              // "tunl <id>"
  stats: L2tpStats,                            // rx_packets/bytes/invalid/...
}

struct L2tpSession {
  magic: u32,                                  // L2TP_SESSION_MAGIC
  ref_count: AtomicRefcount,
  rcu: RcuHead,
  tunnel: *L2tpTunnel,
  session_id: u32,
  peer_session_id: u32,
  pwtype: u16,                                 // pseudowire type
  hdr_len: u16,
  send_seq: bool, recv_seq: bool, lns_mode: bool,
  ns: u32, nr: u32, nr_max: u32, nr_window_size: u32,
  nr_oos: u32, nr_oos_count: u32, nr_oos_count_max: u32,
  reorder_skip: u8,
  reorder_q: SkBuffHead,
  reorder_timeout: u64,
  cookie_len: u8, peer_cookie_len: u8,
  cookie: [u8; 8], peer_cookie: [u8; 8],
  l2specific_type: u8,
  hlist: HlistNode,                            // v3 collision htable
  hlist_key: u64,
  clist: ListHead,                             // coll_list link
  coll_list: Option<*L2tpSessionCollList>,
  list: ListHead,                              // tunnel.session_list link
  dead: AtomicBit,
  del_work: WorkStruct,
  ifname: [u8; IFNAMSIZ],
  name: [u8; 32],                              // "sess <tid>/<sid>"
  recv_skb: Option<fn(session: *L2tpSession, skb: *SkBuff, length: i32)>,
  session_close: Option<fn(session: *L2tpSession)>,
  stats: L2tpStats,
}

static L2TP_WQ: Once<*WorkqueueStruct>;
static L2TP_NET_ID: PerNetId;
```

`L2tpNet::v2_session_key(tid u16, sid u16) -> u32`: `((tid as u32) << 16) | sid as u32`.

`L2tpNet::tunnel_get(net, tid) -> Option<RefCounted<*L2tpTunnel>>`:
1. rcu_read_lock_bh.
2. tunnel = idr_find(net.tunnel_idr, tid).
3. if tunnel.is_some() && refcount_inc_not_zero(&tunnel.ref_count):
   - rcu_read_unlock_bh; return Some(tunnel).
4. rcu_read_unlock_bh; return None.

`L2tpNet::v2_session_get(net, tid u16, sid u16) -> Option<RefCounted<*L2tpSession>>`:
1. key = L2tpNet::v2_session_key(tid, sid).
2. rcu_read_lock_bh.
3. s = idr_find(net.v2_session_idr, key).
4. if s && refcount_inc_not_zero(&s.ref_count): unlock; return Some(s).
5. unlock; return None.

`L2tpNet::v3_session_get(net, sk, sid u32) -> Option<RefCounted<*L2tpSession>>`:
1. rcu_read_lock_bh.
2. s = idr_find(net.v3_session_idr, sid).
3. if s.is_some() && !hash_hashed(&s.hlist) && refcount_inc_not_zero(&s.ref_count): unlock; return Some(s).
4. if s.is_some() && sk.is_some():
   - key = L2tpNet::v3_session_hashkey(sk, sid).
   - for s' in net.v3_session_htable[key & 0xF].iter():
     - t = READ_ONCE(s'.tunnel).
     - if s'.session_id == sid && t.is_some() && t.sock == sk && refcount_inc_not_zero(&s'.ref_count):
       - unlock; return Some(s').
5. unlock; return None.

`L2tpSession::recv_common(session, skb, ptr, optr, hdrflags, length)`:
1. tunnel = session.tunnel.
2. /* Cookie */
3. if session.peer_cookie_len > 0:
   - if memcmp(ptr, &session.peer_cookie, session.peer_cookie_len) != 0:
     - session.stats.rx_cookie_discards += 1; goto discard.
   - ptr += session.peer_cookie_len.
4. /* Seq */
5. L2TP_SKB_CB(skb).has_seq = 0.
6. if tunnel.version == 2 && (hdrflags & L2TP_HDRFLAG_S):
   - cb.ns = ntohs(*(ptr as *BE16)); cb.has_seq = 1; ptr += 4 (skip Nr too).
7. else if session.l2specific_type == L2TP_L2SPECTYPE_DEFAULT:
   - l2h = ntohl(*(ptr as *BE32)).
   - if l2h & 0x40000000: cb.ns = l2h & 0x00ffffff; cb.has_seq = 1.
   - ptr += 4.
8. if cb.has_seq:
   - if !session.lns_mode && !session.send_seq: session.send_seq = true; set_header_len.
9. else:
   - if session.recv_seq: rx_seq_discards++; goto discard.
   - if !session.lns_mode && session.send_seq: session.send_seq = false; set_header_len.
   - else if session.send_seq: rx_seq_discards++; goto discard.
10. /* v2 offset */
11. if tunnel.version == 2 && (hdrflags & L2TP_HDRFLAG_O):
    - offset = ntohs(*(ptr as *BE16)); ptr += 2 + offset.
12. offset_total = ptr - optr.
13. if !pskb_may_pull(skb, offset_total): goto discard.
14. __skb_pull(skb, offset_total).
15. cb.length = length; cb.expires = jiffies + (session.reorder_timeout if session.reorder_timeout != 0 else HZ).
16. if cb.has_seq:
    - if L2tpSession::recv_data_seq(session, skb) != 0: goto discard.
17. else: skb_queue_tail(&session.reorder_q, skb).
18. L2tpSession::recv_dequeue(session).
19. return.
20. discard: stats.rx_errors++; kfree_skb(skb).

`L2tpSession::xmit_core(session, skb, *len) -> NetXmit`:
1. tunnel = session.tunnel; sk = tunnel.sock; data_len = skb.len.
2. uhlen = if tunnel.encap == UDP { sizeof(udphdr) } else { 0 }.
3. headroom = NET_SKB_PAD + sizeof(iphdr) + uhlen + session.hdr_len.
4. if skb_cow_head(skb, headroom): kfree_skb(skb); return DROP.
5. /* L2TP header */
6. if tunnel.version == 2: L2tpSession::build_v2_header(session, __skb_push(skb, session.hdr_len)).
7. else: L2tpSession::build_v3_header(session, __skb_push(skb, session.hdr_len)).
8. memset(skb.cb, 0, sizeof(skb.cb)); nf_reset_ct(skb).
9. spin_lock_nested(&sk.sk_lock.slock, L2TP_DEPTH_NESTING).
10. if sock_owned_by_user(sk): kfree; ret = DROP; goto unlock.
11. if tunnel.fd >= 0 && sk.sk_state != TCP_ESTABLISHED: kfree; ret = DROP; goto unlock.
12. *len = skb.len.                                    // pre-UDP-push
13. match tunnel.encap:
    - UDP:
      - __skb_push(skb, sizeof(udphdr)); skb_reset_transport_header(skb).
      - uh = udp_hdr(skb); uh.source = inet.sport; uh.dest = inet.dport.
      - udp_len = uhlen + session.hdr_len + data_len.
      - if udp_len > U16_MAX: kfree; ret = DROP; goto unlock.
      - uh.len = htons(udp_len).
      - if l2tp_sk_is_v6(sk): udp6_set_csum(...).
      - else: udp_set_csum(sk.sk_no_check_tx, skb, inet.saddr, inet.daddr, udp_len).
    - IP: nothing.
14. ret = L2tpTunnel::xmit_queue(tunnel, skb, &inet.cork.fl).
15. unlock; return ret.

`L2tpTunnel::xmit_queue(tunnel, skb, fl) -> NetXmit`:
1. skb.ignore_df = 1; skb_dst_drop(skb).
2. if l2tp_sk_is_v6(tunnel.sock): err = inet6_csk_xmit(tunnel.sock, skb, None).
3. else: err = ip_queue_xmit(tunnel.sock, skb, fl).
4. return if err >= 0 { SUCCESS } else { DROP }.

`L2tpSession::build_v2_header(session, buf) -> usize`:
1. tunnel = session.tunnel.
2. flags = L2TP_HDR_VER_2; if session.send_seq: flags |= L2TP_HDRFLAG_S.
3. write_be16(buf+0, flags).
4. write_be16(buf+2, tunnel.peer_tunnel_id as u16).
5. write_be16(buf+4, session.peer_session_id as u16).
6. if session.send_seq:
   - write_be16(buf+6, session.ns as u16); write_be16(buf+8, 0).
   - session.ns = (session.ns + 1) & 0xffff.
   - return 10.
7. return 6.

`L2tpSession::build_v3_header(session, buf) -> usize`:
1. tunnel = session.tunnel. n = 0.
2. if tunnel.encap == UDP: write_be16(buf+0, L2TP_HDR_VER_3); write_be16(buf+2, 0); n = 4.
3. write_be32(buf+n, session.peer_session_id); n += 4.
4. if session.cookie_len > 0: memcpy(buf+n, &session.cookie, session.cookie_len); n += session.cookie_len.
5. if session.l2specific_type == L2TP_L2SPECTYPE_DEFAULT:
   - l2h = if session.send_seq { 0x40000000 | session.ns } else { 0 }.
   - write_be32(buf+n, l2h); n += 4.
   - if session.send_seq: session.ns = (session.ns + 1) & 0xffffff.
6. return n.

`L2tpSession::register_with(session, tunnel) -> Result<(), Errno>`:
1. pn = L2tpNet::for_net(tunnel.l2tp_net).
2. spin_lock_bh(&tunnel.list_lock); spin_lock_bh(&pn.session_idr_lock).
3. if !tunnel.acpt_newsess: err = -ENODEV; goto out.
4. if tunnel.version == 3:
   - key = session.session_id.
   - err = idr_alloc_u32(&pn.v3_session_idr, NULL, &key, key, GFP_ATOMIC).
   - if err == -ENOSPC && tunnel.encap == UDP:
     - other = idr_find(&pn.v3_session_idr, key).
     - err = L2tpNet::collision_add(pn, session, other).
5. else:
   - key = L2tpNet::v2_session_key(tunnel.tunnel_id, session.session_id).
   - err = idr_alloc_u32(&pn.v2_session_idr, NULL, &key, key, GFP_ATOMIC).
6. if err: if err == -ENOSPC { err = -EEXIST }; goto out.
7. refcount_inc(&tunnel.ref_count); WRITE_ONCE(session.tunnel, tunnel); list_add_rcu(&session.list, &tunnel.session_list).
8. if tunnel.version == 3 && other.is_none(): idr_replace(&pn.v3_session_idr, session, key).
9. else if tunnel.version == 2: idr_replace(&pn.v2_session_idr, session, key).
10. out: unlock both; if !err: trace_register_session(session); return err.

`L2tpTunnel::udp_encap_recv(sk, skb) -> i32`:
1. net = sock_net(sk).
2. __skb_pull(skb, sizeof(udphdr)).
3. if !pskb_may_pull(skb, L2TP_HDR_SIZE_MAX): goto pass.
4. optr = skb.data; ptr = skb.data.
5. hdrflags = ntohs(*(ptr as *BE16)).
6. version = hdrflags & L2TP_HDR_VER_MASK.
7. length = skb.len.
8. if hdrflags & L2TP_HDRFLAG_T: goto pass.       // control → userspace
9. ptr += 2.
10. session = match version:
    - 2:
      - if hdrflags & L2TP_HDRFLAG_L: ptr += 2.
      - tid = ntohs(*(ptr as *BE16)); ptr += 2.
      - sid = ntohs(*(ptr as *BE16)); ptr += 2.
      - L2tpNet::v2_session_get(net, tid, sid).
    - 3:
      - ptr += 2.                                  // reserved
      - sid = ntohl(*(ptr as *BE32)); ptr += 4.
      - L2tpNet::v3_session_get(net, sk, sid).
11. if session.is_none() || session.recv_skb.is_none():
    - if session.is_some(): l2tp_session_put(session.unwrap()).
    - goto pass.
12. tunnel = session.tunnel.
13. if version != tunnel.version: goto invalid (after put).
14. if version == 3 && l2tp_v3_ensure_opt_in_linear(session, skb, &ptr, &optr) != 0: invalid.
15. L2tpSession::recv_common(session, skb, ptr, optr, hdrflags, length).
16. l2tp_session_put(session); return 0.
17. invalid: tunnel.stats.rx_invalid += 1.
18. pass: __skb_push(skb, sizeof(udphdr)); return 1.

### Out of Scope

- L2TP control plane (HELLO, SCCRQ, SCCRP, SCCCN, StopCCN, ICRQ, ICRP, ICCN, CDN, ZLB, WEN, SLI, AVP encode/decode, retransmission state machine) — userspace daemon territory (`xl2tpd`, `openl2tp`)
- `net/l2tp/l2tp_ppp.c` PPPoL2TP socket family (covered separately if expanded)
- `net/l2tp/l2tp_eth.c` L2TPv3 Ethernet pseudowire netdev (covered separately if expanded)
- `net/l2tp/l2tp_ip.c` / `l2tp_ip6.c` IPPROTO_L2TP raw-socket transport (covered separately if expanded)
- `net/l2tp/l2tp_netlink.c` AF_NETLINK management interface (covered separately if expanded)
- `net/l2tp/l2tp_debugfs.c` debugfs introspection (covered separately if expanded)
- `net/l2tp/trace.h` tracepoint definitions (covered separately if expanded)
- `include/uapi/linux/l2tp.h` userspace ABI (referenced as immutable contract)
- `l2tp_v3_ensure_opt_in_linear` helper (declared in `l2tp_core.h`, covered with `l2tp_core.h`)
- UDP socket plumbing (`udp_sock_create`, `setup_udp_tunnel_sock`) — covered by `net/ipv4/udp_tunnel.c` Tier-3 if expanded
- PPPoL2TP / Ethernet / IP / netlink subsystem details
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct l2tp_net` | per-net IDRs + locks | `L2tpNet` |
| `struct l2tp_tunnel` (declared in l2tp_core.h) | per-tunnel context | `L2tpTunnel` |
| `struct l2tp_session` | per-session context | `L2tpSession` |
| `struct l2tp_skb_cb` | per-skb L2TP control buf | `L2tpSkbCb` |
| `struct l2tp_session_coll_list` | per-v3 session-id collision list | `L2tpSessionCollList` |
| `enum l2tp_encap_type {UDP, IP}` | per-tunnel transport | `L2tpEncapType` |
| `L2TP_HDRFLAG_T / _L / _S / _O / _P` | per-header bit flags | `L2tpHdrFlag::*` |
| `L2TP_HDR_VER_2 / _VER_3` | per-protocol version | `L2tpHdrVer::*` |
| `L2TP_SLFLAG_S` / `L2TP_SL_SEQ_MASK` | v3 L2-specific sublayer | `L2tpSlSeq` |
| `l2tp_v2_session_key()` | per-(tunnel_id,session_id) key | `L2tpNet::v2_session_key` |
| `l2tp_v3_session_hashkey()` | per-(sk,session_id) hash | `L2tpNet::v3_session_hashkey` |
| `l2tp_pernet()` | per-net accessor | `L2tpNet::for_net` |
| `l2tp_tunnel_free()` / `l2tp_session_free()` | per-RCU-free | `L2tpTunnel::free` / `L2tpSession::free` |
| `l2tp_tunnel_put()` / `l2tp_session_put()` | per-refcount drop | `L2tpTunnel::put` / `L2tpSession::put` |
| `l2tp_sk_to_tunnel()` | per-sock reverse lookup | `L2tpTunnel::from_sk` |
| `l2tp_tunnel_get()` / `_get_next()` | per-tunnel_id lookup / iter | `L2tpNet::tunnel_get` / `tunnel_get_next` |
| `l2tp_v2_session_get()` / `_get_next()` | per-v2 lookup / iter | `L2tpNet::v2_session_get` / `v2_session_get_next` |
| `l2tp_v3_session_get()` / `_get_next()` | per-v3 lookup / iter | `L2tpNet::v3_session_get` / `v3_session_get_next` |
| `l2tp_session_get()` / `_get_next()` | per-version dispatch | `L2tpNet::session_get` / `session_get_next` |
| `l2tp_session_get_by_ifname()` | per-ifname linear scan | `L2tpNet::session_get_by_ifname` |
| `l2tp_session_collision_add()` / `_del()` | per-v3-id collision hlist | `L2tpNet::collision_add` / `_del` |
| `l2tp_session_coll_list_add()` | per-collision-list insert | `L2tpNet::coll_list_add` |
| `l2tp_session_register()` | per-session-publish | `L2tpSession::register_with` |
| `l2tp_recv_queue_skb()` / `_dequeue_skb()` / `_dequeue()` | per-reorder-queue | `L2tpSession::recv_queue` / `recv_dequeue_skb` / `recv_dequeue` |
| `l2tp_seq_check_rx_window()` | per-NR-window check | `L2tpSession::seq_check_rx_window` |
| `l2tp_recv_data_seq()` | per-seq accept | `L2tpSession::recv_data_seq` |
| `l2tp_recv_common()` | per-decap entry | `L2tpSession::recv_common` |
| `l2tp_udp_encap_recv()` | per-UDP-encap rx | `L2tpTunnel::udp_encap_recv` |
| `l2tp_udp_encap_err_recv()` | per-UDP-encap err rx | `L2tpTunnel::udp_encap_err_recv` |
| `l2tp_build_l2tpv2_header()` / `_l2tpv3_header()` | per-tx hdr build | `L2tpSession::build_v2_header` / `build_v3_header` |
| `l2tp_xmit_queue()` | per-tx ip_queue_xmit / inet6_csk_xmit | `L2tpTunnel::xmit_queue` |
| `l2tp_xmit_core()` | per-tx encap + queue | `L2tpSession::xmit_core` |
| `l2tp_xmit_skb()` | per-tx public entry | `L2tpSession::xmit_skb` |
| `l2tp_session_queue_purge()` | per-session reorder_q drain | `L2tpSession::queue_purge` |
| `l2tp_session_unhash()` / `_delete()` | per-session shutdown | `L2tpSession::unhash` / `delete` |
| `l2tp_tunnel_closeall()` / `_remove()` / `_delete()` | per-tunnel shutdown | `L2tpTunnel::close_all` / `remove` / `delete` |
| `l2tp_udp_encap_destroy()` | per-sock-close hook | `L2tpTunnel::udp_encap_destroy` |
| `l2tp_tunnel_del_work()` / `session_del_work()` | per-workqueue teardown | `L2tpTunnel::del_work` / `L2tpSession::del_work` |
| `l2tp_tunnel_sock_create()` | per-kernel-socket creation | `L2tpTunnel::sock_create` |
| `l2tp_tunnel_create()` / `_register()` | per-tunnel init/publish | `L2tpTunnel::create` / `register_with` |
| `l2tp_validate_socket()` | per-sock validation | `L2tpTunnel::validate_socket` |
| `l2tp_session_create()` | per-session alloc | `L2tpSession::create` |
| `l2tp_session_set_header_len()` | per-recalc hdr len | `L2tpSession::set_header_len` |
| `l2tp_init_net()` / `_pre_exit_net()` / `_exit_net()` | per-net lifecycle | `L2tpNet::init` / `pre_exit` / `exit` |
| `l2tp_wq` (workqueue) | per-module deferred-teardown | `L2tpWq` (module-static) |
| `l2tp_net_id` (per-net ID) | per-net generic key | `L2TP_NET_ID` |

### compatibility contract

REQ-1: L2TP header constants (network byte order on the wire):
- T (Type) = 0x8000: 1 = control frame, 0 = data frame.
- L (Length) = 0x4000: optional Length field present.
- S (Sequence) = 0x0800: optional Ns + Nr present.
- O (Offset) = 0x0200: optional offset present (v2 only).
- P (Priority) = 0x0100.
- VER mask = 0x000F: 2 = L2TPv2, 3 = L2TPv3.
- L2TPv3 L2-specific sublayer "default": S-flag = 0x40000000; sequence in low 24 bits (mask 0x00ffffff).
- `L2TP_HDR_SIZE_MAX = 14`.

REQ-2: Per-net storage `struct l2tp_net`:
- `l2tp_tunnel_idr_lock`: spinlock for write access to tunnel IDR.
- `l2tp_tunnel_idr`: IDR keyed by tunnel_id.
- `l2tp_session_idr_lock`: spinlock for write access to both v2/v3 session IDRs and v3 htable.
- `l2tp_v2_session_idr`: IDR keyed by `(tunnel_id << 16) | session_id`.
- `l2tp_v3_session_idr`: IDR keyed by session_id alone.
- `l2tp_v3_session_htable[16]`: hash table for v3 session-ID collisions across tunnels.

REQ-3: v2 session key:
- `l2tp_v2_session_key(tunnel_id u16, session_id u16) = ((u32)tunnel_id << 16) | session_id`.

REQ-4: v3 session hash key:
- `l2tp_v3_session_hashkey(sk, session_id) = ((unsigned long)sk) + session_id`.

REQ-5: Tunnel allocation and lifecycle:
- `l2tp_tunnel_create(fd, version, tunnel_id, peer_tunnel_id, cfg, tunnelp)`: zalloc, set version, ids, encap (UDP default), name `"tunl <id>"`, init `list_lock`, init empty `session_list`, set `acpt_newsess=true`, set `refcount=1`, set `fd`, init `del_work` to `l2tp_tunnel_del_work`. Returns 0.
- `l2tp_tunnel_register(tunnel, net, cfg)`:
  - `idr_alloc_u32(&pn->l2tp_tunnel_idr, NULL, &tunnel_id, tunnel_id, GFP_ATOMIC)` — reserve slot; ENOSPC → -EEXIST.
  - If `tunnel->fd < 0`: `l2tp_tunnel_sock_create(net, ...)`.
  - Else: `sockfd_lookup(tunnel->fd, ...)`.
  - `lock_sock(sk); write_lock_bh(&sk->sk_callback_lock); l2tp_validate_socket(sk, net, tunnel->encap)`.
  - For UDP: `setup_udp_tunnel_sock` with `encap_type=UDP_ENCAP_L2TPINUDP`, `encap_rcv=l2tp_udp_encap_recv`, `encap_err_rcv=l2tp_udp_encap_err_recv`, `encap_destroy=l2tp_udp_encap_destroy`.
  - `sk->sk_allocation = GFP_ATOMIC; release_sock(sk); sock_hold(sk); tunnel->sock = sk; tunnel->l2tp_net = net`.
  - `idr_replace(&pn->l2tp_tunnel_idr, tunnel, tunnel->tunnel_id)`.
  - `trace_register_tunnel(tunnel)`.
- `l2tp_tunnel_delete(tunnel)`: `test_and_set_bit(0, &dead)` then `refcount_inc; queue_work(l2tp_wq, &del_work)`.
- `l2tp_tunnel_del_work`: `close_all → if fd<0 sock_release → tunnel_remove → put (initial) → put (workqueue)`.
- `l2tp_tunnel_free`: clears `udp_sk(sk)->encap_*`, `sock_put(sk)`, `kfree_rcu`.

REQ-6: Tunnel lookup:
- `l2tp_tunnel_get(net, tunnel_id) -> *l2tp_tunnel`:
  - `rcu_read_lock_bh`.
  - `tunnel = idr_find(&pn->l2tp_tunnel_idr, tunnel_id)`.
  - if `tunnel && refcount_inc_not_zero(&tunnel->ref_count)`: unlock; return tunnel.
  - unlock; return NULL.
- `l2tp_tunnel_get_next(net, key)`: `idr_get_next_ul` loop, skip not-yet-published (refcount fails) entries, advance `*key`.
- `l2tp_sk_to_tunnel(sk)`: linear scan of tunnel IDR; match `tunnel->sock == sk`.

REQ-7: Session allocation and lifecycle:
- `l2tp_session_create(priv_size, tunnel, session_id, peer_session_id, cfg)`:
  - `kzalloc(sizeof(*session)+priv_size, GFP_KERNEL)`.
  - `magic = L2TP_SESSION_MAGIC`; set ids; `nr=0`; `nr_max = 0xffff` (v2) or `0xffffff` (v3); `nr_window_size = nr_max/2`; `nr_oos_count_max = 4`; `reorder_skip = 1`.
  - Name `"sess <tid>/<sid>"`.
  - `skb_queue_head_init(&reorder_q)`.
  - `hlist_key = l2tp_v3_session_hashkey(tunnel->sock, session_id)`.
  - Init `hlist`, `clist`, `list`, `del_work = l2tp_session_del_work`.
  - If `cfg`: copy pwtype, send_seq, recv_seq, lns_mode, reorder_timeout, l2specific_type, cookie_len + cookie[], peer_cookie_len + peer_cookie[].
  - `l2tp_session_set_header_len(session, version, encap)`.
  - `refcount_set(&ref_count, 1)`.
- `l2tp_session_register(session, tunnel)`:
  - Lock `tunnel->list_lock`, then `pn->l2tp_session_idr_lock`.
  - if `!tunnel->acpt_newsess`: -ENODEV.
  - v3: `idr_alloc_u32(&pn->l2tp_v3_session_idr, NULL, &session_key, session_id, GFP_ATOMIC)`. On `-ENOSPC` with UDP encap: collision via `l2tp_session_collision_add`. (IP encap requires globally unique session IDs; -EEXIST on collision.)
  - v2: `idr_alloc_u32(&pn->l2tp_v2_session_idr, NULL, &session_key, l2tp_v2_session_key(tunnel_id, session_id), GFP_ATOMIC)`.
  - On success: `refcount_inc(&tunnel->ref_count); WRITE_ONCE(session->tunnel, tunnel); list_add_rcu(&session->list, &tunnel->session_list)`.
  - `idr_replace` (after publish) iff no collision: makes session visible to lockless getters.
  - Unlock; `trace_register_session(session)`; return 0.
- `l2tp_session_delete(session)`: `test_and_set_bit(0, &dead)` then `queue_work(l2tp_wq, &del_work)`.
- `l2tp_session_del_work`: `unhash → queue_purge → session_close? → put (initial) → put (workqueue)`.

REQ-8: Session lookup:
- `l2tp_v2_session_get(net, tunnel_id, session_id)`: rcu_read_lock_bh; `idr_find(&pn->l2tp_v2_session_idr, l2tp_v2_session_key(tunnel_id, session_id))`; `refcount_inc_not_zero`; unlock.
- `l2tp_v3_session_get(net, sk, session_id)`:
  - `idr_find(&pn->l2tp_v3_session_idr, session_id)`.
  - If found and `!hash_hashed(&session->hlist)` and `refcount_inc_not_zero`: return.
  - If found and `sk`: walk `l2tp_v3_session_htable` for `(sk, session_id)` collision match.
- `l2tp_session_get(net, sk, pver, tunnel_id, session_id)`: dispatches by `pver`.
- `l2tp_session_get_by_ifname(net, ifname)`: linear scan tunnel IDR × tunnel session_list.

REQ-9: v3 session-id collision hlist (UDP encap only):
- `l2tp_session_coll_list_add(clist, session)`:
  - `refcount_inc(&session->ref_count)`; `session->coll_list = clist`; spin_lock(clist->lock); `list_add(&session->clist, &clist->list)`; unlock.
- `l2tp_session_collision_add(pn, session1, session2)`:
  - Requires `pn->l2tp_session_idr_lock` held.
  - If `session2 == NULL`: return -EEXIST.
  - If `session2->tunnel->encap == L2TP_ENCAPTYPE_IP`: return -EEXIST (IP encap forbids collision).
  - If `session2->coll_list == NULL`: alloc `clist`, init lock/list, `refcount_set(1)`, add session2.
  - If `!hash_hashed(&session2->hlist)`: `hash_add_rcu(htable, &session2->hlist, session2->hlist_key)`.
  - `hash_add_rcu(htable, &session1->hlist, session1->hlist_key)`.
  - `refcount_inc(&clist->ref_count)`; add session1.
- `l2tp_session_collision_del(pn, session)`:
  - `hash_del_rcu(&session->hlist)`.
  - If `clist`: spin_lock; `list_del_init(&session->clist)`; pick replacement session2; `idr_replace` or `idr_remove`; `coll_list = NULL`; unlock; `refcount_dec_and_test(clist)` → kfree; `l2tp_session_put(session)`.

REQ-10: Per-skb `struct l2tp_skb_cb`:
- `ns: u32` — sequence number from header.
- `has_seq: u16` — sequence-numbers-present flag.
- `length: u16` — payload length.
- `expires: unsigned long` — reorder deadline (jiffies).
- Stored in skb->cb at offset `sizeof(struct inet_skb_parm)`.

REQ-11: `l2tp_udp_encap_recv(sk, skb)`:
- `__skb_pull(skb, sizeof(struct udphdr))`.
- if `!pskb_may_pull(skb, L2TP_HDR_SIZE_MAX)`: goto pass.
- `hdrflags = ntohs(*(__be16 *)ptr)`; `version = hdrflags & L2TP_HDR_VER_MASK`; `length = skb->len`.
- if `hdrflags & L2TP_HDRFLAG_T`: control frame; goto pass (userspace).
- Advance `ptr` past flags.
- v2: optional Length skipped (`L`); read `tunnel_id (u16 BE)`, `session_id (u16 BE)`; `session = l2tp_v2_session_get(net, tunnel_id, session_id)`.
- v3: skip 2 bytes reserved; read `session_id (u32 BE)`; `session = l2tp_v3_session_get(net, sk, session_id)`.
- if `!session || !session->recv_skb`: put if set, goto pass.
- `tunnel = session->tunnel`; if `version != tunnel->version`: invalid.
- v3: if `l2tp_v3_ensure_opt_in_linear(session, skb, &ptr, &optr)` fails: invalid.
- `l2tp_recv_common(session, skb, ptr, optr, hdrflags, length); l2tp_session_put(session); return 0`.
- invalid: `atomic_long_inc(&tunnel->stats.rx_invalid)`; fallthrough.
- pass: `__skb_push(skb, sizeof(struct udphdr))`; return 1.

REQ-12: `l2tp_recv_common(session, skb, ptr, optr, hdrflags, length)`:
- Cookie: if `session->peer_cookie_len > 0`: `memcmp(ptr, peer_cookie, peer_cookie_len)`; mismatch → rx_cookie_discards++; discard. Advance ptr.
- Sequence handling:
  - v2: if `hdrflags & L2TP_HDRFLAG_S`: read Ns (BE16) into cb; `has_seq=1`; skip Nr (BE16).
  - v3 with `l2specific_type == L2TP_L2SPECTYPE_DEFAULT`: read 32-bit sublayer; if S-flag (0x40000000) set: `cb->ns = l2h & 0x00ffffff; has_seq=1`. Advance 4 bytes.
- If `has_seq` and we're LAC (`!lns_mode`) and not yet sending seq: enable `send_seq`; `set_header_len`.
- If `!has_seq`:
  - If `recv_seq`: rx_seq_discards++; discard.
  - If we're LAC and were sending seq: disable `send_seq` (LNS told us to stop); set_header_len.
  - Else if still sending seq: discard.
- v2 only: if `L2TP_HDRFLAG_O`: read offset (BE16); advance `ptr += 2 + offset`.
- `offset = ptr - optr`; if `!pskb_may_pull(skb, offset)`: discard.
- `__skb_pull(skb, offset)`.
- `cb->length = length`; `cb->expires = jiffies + (reorder_timeout ? reorder_timeout : HZ)`.
- If `has_seq`: `l2tp_recv_data_seq(session, skb)`; on discard return: free.
- Else: `skb_queue_tail(&reorder_q, skb)`.
- `l2tp_recv_dequeue(session)`.
- discard path: `stats.rx_errors++; kfree_skb`.

REQ-13: `l2tp_recv_data_seq(session, skb)`:
- `cb = L2TP_SKB_CB(skb)`.
- if `!l2tp_seq_check_rx_window(session, cb->ns)`: trace OOR; discard (return 1).
- if `session->reorder_timeout != 0`: `l2tp_recv_queue_skb(session, skb)`; return 0.
- Else (reorder disabled):
  - if `cb->ns == session->nr`: `skb_queue_tail(&reorder_q, skb)`.
  - else:
    - `nr_oos = cb->ns`; `nr_next = (session->nr_oos + 1) & session->nr_max`.
    - if `nr_oos == nr_next`: `nr_oos_count++`; else `nr_oos_count = 0`.
    - `nr_oos = cb->ns`.
    - if `nr_oos_count > nr_oos_count_max`: `reorder_skip = 1`.
    - if `!reorder_skip`: rx_seq_discards++; discard.
    - else `skb_queue_tail(&reorder_q, skb)`.

REQ-14: `l2tp_seq_check_rx_window(session, nr)`:
- `nws = (nr >= session->nr) ? (nr - session->nr) : ((session->nr_max + 1) - (session->nr - nr))`.
- return `nws < session->nr_window_size`.

REQ-15: `l2tp_recv_queue_skb(session, skb)`:
- spin_lock_bh(&reorder_q.lock); insert before first entry with greater `ns` (rx_oos_packets++ on insertion), else queue tail; unlock.

REQ-16: `l2tp_recv_dequeue(session)`:
- start: spin_lock_bh.
- For each skb in reorder_q:
  - if `time_after(jiffies, cb->expires)`: rx_seq_discards++; rx_errors++; reorder_skip=1; unlink; kfree.
  - if `cb->has_seq`:
    - if `reorder_skip`: `reorder_skip=0; session->nr = cb->ns`.
    - if `cb->ns != session->nr`: goto out (wait).
  - unlink; unlock; `l2tp_recv_dequeue_skb(session, skb)`; goto start.
- out: unlock.

REQ-17: `l2tp_recv_dequeue_skb(session, skb)`:
- `tunnel = session->tunnel`; `length = cb->length`.
- `skb_orphan(skb)`.
- Stats: `tunnel.stats.rx_packets++/rx_bytes+=length`; same on session.
- if `cb->has_seq`: `nr = (nr+1) & nr_max`.
- if `session->recv_skb`: `(*recv_skb)(session, skb, length)`; else `kfree_skb`.

REQ-18: `l2tp_xmit_skb(session, skb)`:
- `ret = l2tp_xmit_core(session, skb, &len)`.
- if success: tunnel/session `tx_packets++ / tx_bytes+=len`.
- else: `tx_errors++` on both.

REQ-19: `l2tp_xmit_core(session, skb, *len)`:
- `uhlen = (encap == UDP) ? sizeof(udphdr) : 0`.
- `headroom = NET_SKB_PAD + sizeof(iphdr) + uhlen + session->hdr_len`.
- if `skb_cow_head(skb, headroom)`: kfree; return NET_XMIT_DROP.
- v2: `l2tp_build_l2tpv2_header(session, __skb_push(skb, session->hdr_len))`.
- v3: `l2tp_build_l2tpv3_header(...)`.
- `memset(skb->cb, 0, sizeof(skb->cb)); nf_reset_ct(skb)`.
- `spin_lock_nested(&sk->sk_lock.slock, L2TP_DEPTH_NESTING)`.
- if `sock_owned_by_user(sk)`: drop.
- if `tunnel->fd >= 0 && sk->sk_state != TCP_ESTABLISHED`: drop.
- `*len = skb->len` (before UDP push, for stats).
- UDP encap: `__skb_push(skb, sizeof(udphdr))`; reset transport header; set `uh->source/dest/len`; if `udp_len > U16_MAX` drop; `udp_set_csum` (v4) or `udp6_set_csum` (v6).
- `ret = l2tp_xmit_queue(tunnel, skb, &inet->cork.fl)`; unlock; return ret.

REQ-20: `l2tp_build_l2tpv2_header(session, buf)`:
- `flags = L2TP_HDR_VER_2`; if `send_seq`: `flags |= L2TP_HDRFLAG_S`.
- BE16: flags, peer_tunnel_id, peer_session_id.
- if `send_seq`: BE16 Ns; BE16 0 (Nr); `ns = (ns+1) & 0xffff`.
- Return bytes written.

REQ-21: `l2tp_build_l2tpv3_header(session, buf)`:
- UDP encap: BE16 `L2TP_HDR_VER_3`; BE16 0.
- BE32 `peer_session_id`.
- If `cookie_len`: memcpy `cookie[0..cookie_len]`.
- If `l2specific_type == L2TP_L2SPECTYPE_DEFAULT`: BE32 `(send_seq ? 0x40000000 | ns : 0)`; `ns = (ns+1) & 0xffffff`.

REQ-22: `l2tp_xmit_queue(tunnel, skb, fl)`:
- `skb->ignore_df = 1; skb_dst_drop(skb)`.
- v6: `inet6_csk_xmit(tunnel->sock, skb, NULL)`.
- v4: `ip_queue_xmit(tunnel->sock, skb, fl)`.
- Return `NET_XMIT_SUCCESS` if `err >= 0` else `NET_XMIT_DROP`.

REQ-23: `l2tp_session_set_header_len(session, version, encap)`:
- v2: `hdr_len = 6`; if `send_seq`: `+= 4`.
- v3: `hdr_len = 4 + cookie_len + l2tp_get_l2specific_len(session)`; if UDP encap: `+= 4`.

REQ-24: Lockdep nesting:
- `L2TP_DEPTH_NESTING = 2`. Tx uses `spin_lock_nested(&sk->sk_lock.slock, L2TP_DEPTH_NESTING)` to avoid lockdep splats when an L2TP socket is used to carry traffic that also takes user-socket locks.

REQ-25: `l2tp_validate_socket(sk, net, encap)`:
- Reject: net mismatch, non-SOCK_DGRAM, family != PF_INET/PF_INET6, protocol mismatch (UDP vs IPPROTO_L2TP), UDP socket already has `sk_user_data`, existing `l2tp_sk_to_tunnel` hit.

REQ-26: `l2tp_tunnel_sock_create(net, tunnel_id, peer_tunnel_id, cfg, sockp)`:
- UDP encap: build `udp_port_cfg` (IPv4 or IPv6 family, local/peer addr/port, checksum flags); `udp_sock_create`.
- IP encap: `sock_create_kern(AF_INET[6], SOCK_DGRAM, IPPROTO_L2TP)`; `kernel_bind(&sockaddr_l2tpip)`; `kernel_connect(&sockaddr_l2tpip)` to peer.
- On error: shutdown + release sock; null out `*sockp`.

REQ-27: `l2tp_udp_encap_destroy(sk)`: `l2tp_sk_to_tunnel(sk) → l2tp_tunnel_delete → l2tp_tunnel_put`.

REQ-28: `l2tp_udp_encap_err_recv(sk, skb, err, port, info, payload)`: store err in `sk->sk_err`; `sk_error_report`; if v4 + RECVERR: `ip_icmp_error`; if v6 + RECVERR6: `ipv6_icmp_error`.

REQ-29: `l2tp_tunnel_closeall(tunnel)`:
- spin_lock_bh(&tunnel->list_lock); `acpt_newsess = false`; for each session in `session_list`: `l2tp_session_delete(session)`; unlock.

REQ-30: `l2tp_tunnel_remove(net, tunnel)`:
- spin_lock_bh(&pn->l2tp_tunnel_idr_lock); `idr_remove(&pn->l2tp_tunnel_idr, tunnel->tunnel_id)`; unlock.

REQ-31: Per-net lifecycle:
- `l2tp_init_net`: `idr_init` on tunnel_idr, v2_session_idr, v3_session_idr; spin_lock_init on both locks.
- `l2tp_pre_exit_net`: rcu walk tunnel IDR; `l2tp_tunnel_delete` each; `__flush_workqueue(l2tp_wq)` twice (tunnel-del work queues session-del work).
- `l2tp_exit_net`: assert per-net IDRs empty (`idr_for_each` with `l2tp_idr_item_unexpected`); `idr_destroy` all three.
- `l2tp_net_ops`: `init/exit/pre_exit`; `id = &l2tp_net_id`; `size = sizeof(struct l2tp_net)`.

REQ-32: Module init/exit:
- `l2tp_init`: `register_pernet_device(&l2tp_net_ops)`; `l2tp_wq = alloc_workqueue("l2tp", WQ_UNBOUND, 0)`; pr_info version.
- `l2tp_exit`: `unregister_pernet_device`; `destroy_workqueue(l2tp_wq); l2tp_wq = NULL`.
- `MODULE_LICENSE("GPL")`.

REQ-33: Control plane (HELLO, SCCRQ, SCCRP, SCCCN, StopCCN, ICRQ, ICRP, ICCN, CDN, ZLB, ...):
- Handled by **userspace daemon** (e.g., `xl2tpd`, `openl2tp`) over the same UDP socket the kernel-side tunnel rides.
- Kernel-side: data frames (T=0) are decapped here; control frames (T=1) are passed through to userspace via the unmodified UDP socket (`goto pass` in `l2tp_udp_encap_recv` returns 1, which tells `udp.c` to deliver to userspace).
- Tier-3 does not implement the AVP/state-machine layer.

REQ-34: Reorder semantics:
- If `reorder_timeout != 0`: reorder enabled; in-order skbs queued; reorder_q drained in NR-order; expired skbs dropped with `reorder_skip` resync.
- If `reorder_timeout == 0`: in-sequence only; out-of-sequence dropped unless a run of `nr_oos_count_max` adjacent-OOS skbs trigger `reorder_skip` to resync.

REQ-35: `acpt_newsess` gate:
- Set false on `l2tp_tunnel_closeall`. Subsequent `l2tp_session_register` returns -ENODEV.

REQ-36: Statistics atomics:
- `atomic_long_t` on tunnel and session: `tx_packets`, `tx_bytes`, `tx_errors`, `rx_packets`, `rx_bytes`, `rx_seq_discards`, `rx_oos_packets`, `rx_errors`, `rx_cookie_discards`, `rx_invalid` (tunnel only).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tunnel_idr_lookup_refcounted` | INVARIANT | per-tunnel_get: returned ptr has `refcount_inc_not_zero` applied; concurrent free does not return freed ptr. |
| `session_idr_lookup_refcounted` | INVARIANT | per-{v2,v3}_session_get: same. |
| `v3_collision_ip_encap_rejected` | INVARIANT | per-session_register: v3 + IP encap + colliding key ⟹ -EEXIST. |
| `v3_collision_udp_encap_accepted` | INVARIANT | per-session_register: v3 + UDP encap + colliding key ⟹ added to hlist + coll_list. |
| `recv_common_cookie_match_required` | INVARIANT | per-recv_common: peer_cookie_len > 0 ∧ memcmp != 0 ⟹ drop. |
| `recv_common_seq_required_drops` | INVARIANT | per-recv_common: recv_seq ∧ !has_seq ⟹ drop. |
| `recv_window_rejects_out_of_range` | INVARIANT | per-recv_data_seq: !seq_check_rx_window ⟹ drop. |
| `reorder_disabled_oos_drops_unless_skip` | INVARIANT | per-recv_data_seq: reorder_timeout==0 ∧ ns!=nr ∧ !reorder_skip ⟹ drop. |
| `xmit_udp_len_overflow_drops` | INVARIANT | per-xmit_core: udp_len > U16_MAX ⟹ DROP. |
| `xmit_skb_cow_head_failure_drops` | INVARIANT | per-xmit_core: skb_cow_head ⟹ kfree + DROP. |
| `xmit_fd_state_check` | INVARIANT | per-xmit_core: tunnel.fd >= 0 ∧ sk_state != TCP_ESTABLISHED ⟹ DROP. |
| `xmit_v2_seq_increment_modulo` | INVARIANT | per-build_v2_header: send_seq ⟹ ns_after == (ns_before+1) & 0xffff. |
| `xmit_v3_seq_increment_modulo` | INVARIANT | per-build_v3_header: send_seq ⟹ ns_after == (ns_before+1) & 0xffffff. |
| `tunnel_delete_idempotent` | INVARIANT | per-tunnel_delete: second call (test_and_set_bit returns 1) is no-op. |
| `session_delete_idempotent` | INVARIANT | per-session_delete: same. |
| `pre_exit_drains_wq_twice` | INVARIANT | per-pre_exit_net: __flush_workqueue called twice (tunnel-del then session-del). |
| `exit_asserts_idrs_empty` | INVARIANT | per-exit_net: idr_for_each warns on any non-NULL entry. |
| `control_frame_passes_to_userspace` | INVARIANT | per-udp_encap_recv: T-bit set ⟹ skb unchanged, return 1. |
| `nested_socket_lock_class` | INVARIANT | per-xmit_core: spin_lock_nested with L2TP_DEPTH_NESTING. |

### Layer 2: TLA+

`net/l2tp/l2tp-core.tla`:
- Constants: `Versions = {2, 3}`, `Encaps = {UDP, IP}`, finite `TunnelIds`, `SessionIds`.
- Variables: per-net `tunnel_idr ∈ [TunnelIds -> TunnelState]`, per-net `v2/v3_session_idr`, `v3_htable`, per-tunnel `session_list`, per-session `state ∈ {New, Registered, Dead}`, per-skb `phase ∈ {OnWire, InReorderQ, Delivered, Dropped}`.
- Actions: `TunnelCreate`, `TunnelRegister`, `SessionCreate`, `SessionRegister` (v2 / v3-non-collide / v3-collide-UDP / v3-collide-IP), `Rx(udp_pkt)`, `RecvCommon`, `RecvDataSeq`, `RecvDequeue`, `Xmit`, `SessionDelete`, `TunnelDelete`, `PreExitNet`.
- Properties:
  - `safety_tunnel_id_unique_per_net` — per-net: idr_alloc_u32 with reserved key returns -ENOSPC if used.
  - `safety_v3_ip_encap_globally_unique_session_id` — per-IP-encap tunnel: collision rejected.
  - `safety_only_data_frames_decapped_in_kernel` — per-rx: T=1 ⟹ pass to userspace.
  - `safety_cookie_mismatch_dropped` — per-recv_common: cookie mismatch never reaches recv_skb.
  - `safety_seq_window_enforced` — per-recv_data_seq: ns outside window never queued.
  - `safety_reorder_in_order_delivery` — per-recv_dequeue: skbs delivered to recv_skb in monotonically-increasing ns (with wraparound).
  - `safety_xmit_udp_len_bounded` — per-xmit_core: udp_len ≤ U16_MAX or DROP.
  - `safety_delete_serialized_by_dead_bit` — per-tunnel/session: concurrent delete callers see exactly one wins.
  - `safety_pre_exit_then_exit_empty` — per-pre_exit_net then exit_net: all per-net IDRs empty.
  - `liveness_registered_eventually_visible` — per-session_register success: subsequent v2/v3_session_get returns it.
  - `liveness_delete_eventually_freed` — per-tunnel_delete: del_work eventually frees tunnel after refcount reaches 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `L2tpTunnel::register_with` post: Ok ⟹ tunnel ∈ pn.tunnel_idr ∧ udp_sk.encap_rcv == udp_encap_recv (for UDP) | `L2tpTunnel::register_with` |
| `L2tpNet::tunnel_get` post: Some(t) ⟹ refcount_inc_not_zero succeeded | `L2tpNet::tunnel_get` |
| `L2tpNet::v2_session_get` post: Some(s) ⟹ key matches `(tid<<16)|sid` | `L2tpNet::v2_session_get` |
| `L2tpNet::v3_session_get` post: Some(s) ⟹ session.session_id == sid ∧ (no-collision ∨ session.tunnel.sock == sk) | `L2tpNet::v3_session_get` |
| `L2tpSession::register_with` post: v3+IP+collision ⟹ Err(-EEXIST); v3+UDP+collision ⟹ session ∈ htable | `L2tpSession::register_with` |
| `L2tpSession::recv_common` post: cookie mismatch ⟹ skb freed; else ⟹ recv_data_seq or tail-queue + dequeue | `L2tpSession::recv_common` |
| `L2tpSession::recv_data_seq` post: returns 1 ⟹ skb not in reorder_q | `L2tpSession::recv_data_seq` |
| `L2tpSession::xmit_core` post: SUCCESS ⟹ skb passed to ip_queue_xmit / inet6_csk_xmit | `L2tpSession::xmit_core` |
| `L2tpSession::build_v2_header` post: bytes_written ∈ {6, 10} | `L2tpSession::build_v2_header` |
| `L2tpSession::build_v3_header` post: bytes_written matches `set_header_len` formula | `L2tpSession::build_v3_header` |
| `L2tpTunnel::xmit_queue` post: err < 0 ⟹ NET_XMIT_DROP; else NET_XMIT_SUCCESS | `L2tpTunnel::xmit_queue` |
| `L2tpTunnel::delete` post: dead bit set ⟹ del_work queued; bit already set ⟹ no-op | `L2tpTunnel::delete` |
| `L2tpNet::pre_exit` post: tunnel_idr_iter triggers delete on every tunnel; wq flushed twice | `L2tpNet::pre_exit` |
| `L2tpNet::exit` post: all three IDRs `idr_destroy`'d; non-empty entries WARN | `L2tpNet::exit` |

### Layer 4: Verus/Creusot functional

`Per-rx: udp_encap_recv → header parse → l2tp_v[23]_session_get → recv_common → recv_data_seq → recv_dequeue → session.recv_skb` ≡ Linux upstream call chain per `net/l2tp/l2tp_core.c:1014-1113` and `:864-1000`.

`Per-tx: l2tp_xmit_skb → xmit_core → build_l2tpv[23]_header → udp_set_csum / udp6_set_csum → xmit_queue → ip_queue_xmit / inet6_csk_xmit` ≡ upstream `:1327-1344`, `:1224-1322`, `:1140-1205`, `:1208-1222`.

`Per-session-id collision (v3): on idr_alloc -ENOSPC ∧ UDP encap → coll_list_add → hash_add_rcu → idr stays pointing at original; on session removal → idr_replace with next or idr_remove` ≡ upstream `:459-550`.

`Per-net lifecycle: init → pernet ops registered ; pre_exit → tunnel_delete all + flush_wq×2 ; exit → idr asserts empty + idr_destroy×3` ≡ upstream `:1841-1906`, gives RFC-2661 / RFC-3931 data-plane equivalence.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening, including socket-skb cgroup and netfilter hooks.)

L2TP-core reinforcement:

- **Per-RCU + refcount_inc_not_zero on all lookups** — defense against per-UAF when tunnel/session is concurrently deleted.
- **Per-tunnel `dead` test_and_set_bit gates del_work enqueue** — defense against per-double-free via concurrent `l2tp_tunnel_delete`.
- **Per-session `dead` test_and_set_bit, same** — defense against per-double-free of session.
- **Per-tunnel `acpt_newsess=false` on closeall** — defense against per-register-after-shutdown.
- **Per-`l2tp_validate_socket` checks family/type/protocol/user-data/existing-tunnel** — defense against per-confused-deputy reuse of arbitrary UDP sockets.
- **Per-v3 IP-encap globally-unique session_id (collision -EEXIST)** — defense against per-RFC-3931 ambiguity in IP encap (no UDP port to disambiguate).
- **Per-cookie memcmp constant-time-irrelevant but mandatory** — defense against per-spoofed-session-id traffic injection.
- **Per-sequence window `nr_window_size = nr_max/2`** — defense against per-replay and per-wraparound confusion.
- **Per-reorder expiry (`jiffies + reorder_timeout|HZ`)** — defense against per-reorder-queue exhaustion DoS.
- **Per-skb cow_head before push** — defense against per-shared-skb modification breaking other holders.
- **Per-`udp_len > U16_MAX` drop** — defense against per-truncated-UDP-length corrupting peer's header parse.
- **Per-`sock_owned_by_user` check during xmit** — defense against per-recursive-socket-lock.
- **Per-`L2TP_DEPTH_NESTING` lockdep class** — defense against per-spurious-lockdep-splat when L2TP rides another userspace socket.
- **Per-`set_bit/atomic_long_inc` stats** — defense against per-bitfield-tearing concurrent stats.
- **Per-net `pre_exit` flushes wq twice (tunnel del → session del cascade)** — defense against per-net-leak on namespace teardown.
- **Per-net `exit` asserts IDRs empty (`l2tp_idr_item_unexpected` WARN)** — defense against per-leak-silent-pass.
- **Per-control-frame pass-through (T=1)** — defense against per-kernel-side AVP parsing complexity (left to userspace, smaller attack surface).
- **Per-`v3_ensure_opt_in_linear` before parse** — defense against per-non-linear-skb OOB read.

