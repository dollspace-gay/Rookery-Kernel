---
title: "Tier-3: net/ipv4/inet_connection_sock.c — Connection-oriented INET sockets (TCP, DCCP-removed, SCTP-shared bits)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`inet_connection_sock` is the shared base struct for connection-oriented INET protocols (TCP primary; DCCP removed in mainline; MPTCP layered). Per-icsk wraps `inet_sock` with: accept-queue (`request_sock_queue`), retransmission timer (`retransmit_timer`), delayed-ACK timer (`delack_timer`), keepalive timer wrapper, congestion-control ops (`icsk_ca_ops`), pending-state machine (`icsk_pending`), per-syn-recv table integration. Per-listen socket has accept-queue holding completed (full handshake done) child socks; per-accept dequeues one. Critical for: TCP listen/accept, retransmission, congestion control plug-in, BPF socket lookup.

This Tier-3 covers `net/ipv4/inet_connection_sock.c` (~1563 lines).

### Acceptance Criteria

- [ ] AC-1: bind(sk, ...): inet_csk_get_port picks port; per-bind-conflict checks pass.
- [ ] AC-2: listen(sk, backlog): TCP_LISTEN + per-accept-queue.
- [ ] AC-3: incoming SYN: SYN-RECV req hashed via inet_csk_reqsk_queue_hash_add.
- [ ] AC-4: handshake completes: inet_csk_complete_hashdance moves to ESTAB; req on accept-q.
- [ ] AC-5: accept(): inet_csk_accept dequeues; returns child sock.
- [ ] AC-6: O_NONBLOCK accept on empty q: -EAGAIN.
- [ ] AC-7: RTO fires: retransmit_timer fn called; icsk_pending == TIME_RETRANS.
- [ ] AC-8: delayed ACK: delack_timer; sends ACK + clears DACK.
- [ ] AC-9: PROBE0: ICSK_TIME_PROBE0; sends zero-window probe.
- [ ] AC-10: close listen sk: per-accept-q drained; per-syn-recv reqs aborted.
- [ ] AC-11: setsockopt TCP_CONGESTION "bbr": icsk_ca_ops switched.
- [ ] AC-12: setsockopt TCP_ULP "tls": icsk_ulp_ops set.

### Architecture

```
struct InetConnSock {
  base: InetSock,                          // includes Sock
  bind_hash: *BindBucket,
  bind2_hash: *Bind2Bucket,
  timeout: u32,                            // jiffies
  retransmit_timer: TimerList,
  delack_timer: TimerList,
  rto: u32,
  rto_min: u32,
  pmtu_cookie: u32,
  ca_ops: &dyn CongestionOps,
  ca_priv: [u8; CONG_OPS_BUF_SIZE],
  ca_state: u8,
  af_ops: &dyn AfOps,
  ulp_ops: Option<&dyn UlpOps>,
  ulp_data: *mut (),
  pending: u8,                             // ICSK_TIME_*
  retransmits: u8,
  probes_out: u8,
  probes_tstamp: u32,
  syn_retries: u8,
  ack: AckState,
  mtup: MtuProbeState,
  accept_queue: RequestSockQueue,
  listen_portaddr_node: HListNode,
}
```

`InetCsk::get_port(sk, snum) -> Result<()>`:
1. /* Enter bind-bucket lock */
2. net = sock_net(sk).
3. if snum == 0:
   - inet_get_local_port_range(net, &low, &high).
   - rover = prandom % (high - low) + low.
   - for tries in 0..(high - low):
     - if !inet_csk_bind_conflict(sk, head, hint = rover): snum = rover; break.
     - rover++.
4. else:
   - if inet_csk_bind_conflict: -EADDRINUSE.
5. /* Install bind */
6. tb = bind_bucket_create(snum).
7. inet_bind_hash(sk, tb, snum).
8. return.

`InetCsk::listen_start(sk) -> Result<()>`:
1. icsk = inet_csk(sk).
2. err = reqsk_queue_alloc(&icsk.accept_queue).
3. if err: return err.
4. sk.sk_max_ack_backlog = backlog.
5. inet_sk_state_store(sk, TCP_LISTEN).
6. err = sk.sk_prot.hash(sk).
7. if err: inet_sk_state_store(sk, TCP_CLOSE); return err.
8. return.

`InetCsk::accept(sk, flags) -> Result<*Sock>`:
1. lock_sock(sk).
2. if sk.sk_state != TCP_LISTEN: ret = -EINVAL; goto out.
3. error = sock_error(sk).
4. if error: goto out.
5. if reqsk_queue_empty(&icsk.accept_queue):
   - if flags & O_NONBLOCK: ret = -EAGAIN; goto out.
   - timeo = sock_rcvtimeo(sk).
   - inet_csk_wait_for_connect(sk, timeo).
6. req = reqsk_queue_remove(&icsk.accept_queue).
7. newsk = req.sk.
8. ret = newsk.
9. release_sock(sk).
10. return ret.

`InetCsk::clone_lock(sk, req, gfp) -> *Sock`:
1. newsk = sk_clone_lock(sk, gfp).
2. if !newsk: return NULL.
3. inet_sk_set_state(newsk, TCP_SYN_RECV).
4. newicsk = inet_csk(newsk).
5. newicsk.bind_hash = NULL.
6. newicsk.retransmits = 0.
7. newicsk.backoff = 0.
8. newicsk.probes_out = 0.
9. newicsk.rto = TCP_TIMEOUT_INIT.
10. inet_clone_ulp(req, newsk, GFP_ATOMIC).
11. return newsk.

`InetCsk::reset_xmit_timer(sk, what, when, max_when)`:
1. when = min(when, max_when).
2. icsk = inet_csk(sk).
3. if what == ICSK_TIME_RETRANS ∨ what == ICSK_TIME_PROBE0 ∨ what == ICSK_TIME_LOSS_PROBE ∨ what == ICSK_TIME_REO_TIMEOUT:
   - icsk.pending = what.
   - icsk.timeout = jiffies + when.
   - sk_reset_timer(sk, &icsk.retransmit_timer, icsk.timeout).
4. else if what == ICSK_TIME_DACK:
   - icsk.ack.pending |= ICSK_ACK_TIMER.
   - icsk.ack.timeout = jiffies + when.
   - sk_reset_timer(sk, &icsk.delack_timer, icsk.ack.timeout).

`InetCsk::complete_hashdance(sk, newsk, req, own_req) -> *Sock`:
1. if own_req:
   - reqsk_queue_unlink(req).
   - reqsk_queue_removed(&inet_csk(sk).accept_queue, req).
   - reqsk_queue_add(&inet_csk(sk).accept_queue, req, newsk).
   - return newsk.
2. else:
   - inet_ehash_nolisten(newsk, NULL, NULL).
   - return newsk.

### Out of Scope

- net/ipv4/tcp_ipv4.c (covered separately if expanded)
- net/ipv4/tcp.c core (covered in `tcp.md` Tier-3)
- net/ipv4/tcp_input.c (covered in `tcp-input.md` Tier-3)
- net/ipv4/tcp_output.c (covered in `tcp-output.md` Tier-3)
- net/ipv4/inet_hashtables.c (covered in `inet-hashtables.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct inet_connection_sock` | per-conn-sock | `InetConnSock` |
| `inet_csk_get_port()` | per-bind port-pick | `InetCsk::get_port` |
| `inet_csk_bind_conflict()` | per-bind conflict | `InetCsk::bind_conflict` |
| `inet_csk_accept()` | per-accept-q dequeue | `InetCsk::accept` |
| `inet_csk_listen_start()` | per-listen | `InetCsk::listen_start` |
| `inet_csk_listen_stop()` | per-close-listen | `InetCsk::listen_stop` |
| `inet_csk_route_req()` | per-syn-recv route | `InetCsk::route_req` |
| `inet_csk_clone_lock()` | per-syn-recv child | `InetCsk::clone_lock` |
| `inet_csk_destroy_sock()` | per-destroy | `InetCsk::destroy_sock` |
| `inet_csk_reqsk_queue_drop()` | per-syn-recv drop | `InetCsk::reqsk_queue_drop` |
| `inet_csk_reqsk_queue_add()` | per-syn-recv enqueue | `InetCsk::reqsk_queue_add` |
| `inet_csk_reqsk_queue_hash_add()` | per-syn-recv hash | `InetCsk::reqsk_queue_hash_add` |
| `inet_csk_reset_keepalive_timer()` | per-keepalive | `InetCsk::reset_keepalive_timer` |
| `inet_csk_reset_xmit_timer()` | per-RTO/probe/delack | `InetCsk::reset_xmit_timer` |
| `inet_csk_clear_xmit_timer()` | per-cancel | `InetCsk::clear_xmit_timer` |
| `inet_csk_delete_keepalive_timer()` | per-delete | `InetCsk::delete_keepalive_timer` |
| `inet_csk_init_xmit_timers()` | per-init | `InetCsk::init_xmit_timers` |
| `inet_csk_complete_hashdance()` | per-syn-recv → established | `InetCsk::complete_hashdance` |
| `icsk_accept_queue` | per-listen accept-q | shared |

### compatibility contract

REQ-1: struct inet_connection_sock embeds inet_sock + adds:
- icsk_accept_queue: request_sock_queue (per-listen completed-conn-q).
- icsk_bind_hash: bind-bucket entry.
- icsk_bind2_hash: bind2 (per-IP+port) entry.
- icsk_timeout: jiffies of next RTO/probe.
- icsk_retransmit_timer: timer for RTO/probe/delack.
- icsk_delack_timer: timer for delayed ACK.
- icsk_rto: current RTO (jiffies).
- icsk_rto_min: per-tcp_rto_min.
- icsk_pmtu_cookie: PMTU cache cookie.
- icsk_ca_ops: per-congestion-control vtable.
- icsk_af_ops: per-AF (ipv4_specific / ipv6_specific) vtable.
- icsk_ulp_ops: per-ULP (TLS, MPTCP) vtable.
- icsk_ulp_data: per-ULP private.
- icsk_clean_acked: per-clean-acked CB.
- icsk_listen_portaddr_node: per-listen-port hash node.
- icsk_pending: ICSK_TIME_RETRANS / _DACK / _PROBE0 / _LOSS_PROBE / _REO_TIMEOUT.
- icsk_ack: ack-related state (pingpong, dack-rcvd, dack-quick, ato).
- icsk_mtup: PMTU probing state.

REQ-2: icsk_pending values:
- ICSK_TIME_RETRANS = 1: RTO retransmit pending.
- ICSK_TIME_DACK = 2: delayed ACK pending.
- ICSK_TIME_PROBE0 = 3: zero-window probe.
- ICSK_TIME_EARLY_RETRANS = 4 (legacy).
- ICSK_TIME_LOSS_PROBE = 5: TLP.
- ICSK_TIME_REO_TIMEOUT = 6: per-RACK reordering.

REQ-3: inet_csk_get_port(sk, snum):
- if snum == 0: pick ephemeral port (per-net inet_get_local_port_range).
- else: try snum.
- bind-conflict check via inet_csk_bind_conflict.
- per-bind-bucket installation.

REQ-4: inet_csk_listen_start(sk):
- /* Allocate accept queue */
- reqsk_queue_alloc(&icsk.icsk_accept_queue).
- /* Hash for syn-recv */
- inet_listen_hashbucket install.
- sk.sk_state = TCP_LISTEN.
- sk.sk_max_ack_backlog = backlog.

REQ-5: inet_csk_accept(sk, flags):
- if !icsk.icsk_accept_queue: -EINVAL.
- if reqsk_queue_empty:
  - if O_NONBLOCK: -EAGAIN.
  - wait_for_data(sk, ...).
- newsk = reqsk_queue_get_child(icsk_accept_queue).
- return newsk.

REQ-6: inet_csk_clone_lock(sk, req, gfp):
- newsk = sk_clone_lock(sk, gfp).
- newsk.sk_state = TCP_SYN_RECV.
- /* Inherit ULP, congestion-control */
- icsk_new = inet_csk(newsk).
- icsk_new.icsk_ca_ops = icsk.icsk_ca_ops.
- newsk.sk_destruct = inet_sock_destruct.

REQ-7: inet_csk_complete_hashdance(sk, newsk, req, own_req):
- /* SYN-RECV → ESTABLISHED in single hash op */
- if own_req: __inet_inherit_port + reqsk_queue_add → accept_q.
- inet_ehash_nolisten(newsk, NULL, NULL).
- return newsk.

REQ-8: inet_csk_reset_xmit_timer(sk, what, when, max):
- /* what ∈ {ICSK_TIME_RETRANS, ICSK_TIME_DACK, ICSK_TIME_PROBE0, ...} */
- icsk.icsk_pending = what.
- icsk.icsk_timeout = jiffies + min(when, max).
- mod_timer(&icsk.icsk_retransmit_timer, icsk_timeout).

REQ-9: inet_csk_route_req(sk, fl4, req):
- /* Per-SYN-RECV: route lookup */
- ip_route_output_flow(net, fl4, ...).
- return rt.

REQ-10: inet_csk_listen_stop(sk):
- /* Drain accept-q */
- while req = reqsk_queue_remove:
  - inet_csk_destroy_sock(req.sk).
- reqsk_queue_destroy.

REQ-11: Per-bind2 (port + IP combo):
- inet_bhash2_addr_match: same-port-different-IP allowed.
- inet_bind2_bucket_create.

REQ-12: Per-ULP (TLS / MPTCP / SMC):
- icsk_ulp_ops + icsk_ulp_data set via setsockopt(SOL_TCP, TCP_ULP, name).

REQ-13: Per-congestion-control (CC):
- icsk_ca_ops: per-CC plugin (cubic, bbr, reno, vegas, ...).
- icsk_ca_priv: per-CC private state (CONG_OPS_BUF_SIZE).
- ca_state: CA_Open / Disorder / CWR / Recovery / Loss.

REQ-14: Per-keepalive timer:
- inet_csk_reset_keepalive_timer.
- per-tcp_keepalive_time / _intvl / _probes.

REQ-15: Per-PMTU probing (icsk_mtup):
- enabled: TCP_MTU_PROBING.
- search_high / search_low.
- probe_size / probe_seq.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `accept_queue_head_invariant` | INVARIANT | per-listen sk: accept-q FIFO ordering. |
| `bind_bucket_unique_per_port` | INVARIANT | per-port: ≥ 1 bind-bucket; sk's not double-installed. |
| `pending_at_most_one` | INVARIANT | icsk.pending: at most one xmit-timer kind active. |
| `state_listen_implies_accept_q` | INVARIANT | sk.state == TCP_LISTEN ⟹ accept_queue allocated. |
| `clone_lock_holds_ref` | INVARIANT | per-sk_clone_lock newsk: refcount > 0; lock held. |
| `keepalive_timer_only_in_estab` | INVARIANT | per-keepalive: TCP_ESTABLISHED. |

### Layer 2: TLA+

`net/ipv4/inet_connection_sock.tla`:
- Per-state-machine: TCP_CLOSE → TCP_LISTEN → (TCP_SYN_RECV) → TCP_ESTABLISHED.
- Per-accept-q + per-syn-recv-hash + per-xmit-timer.
- Properties:
  - `safety_at_most_one_pending_timer` — per-icsk: pending ∈ {0, 1 of TIME_*}.
  - `safety_listen_accept_q_drained_on_close` — listen_stop ⟹ accept-q empty + reqs destroyed.
  - `liveness_per_syn_recv_eventually_estab_or_drop` — req: established or expired-drop.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `InetCsk::get_port` post: bind-bucket installed ∨ -EADDRINUSE | `InetCsk::get_port` |
| `InetCsk::listen_start` post: accept-q allocated; state=LISTEN | `InetCsk::listen_start` |
| `InetCsk::accept` post: returns child or -EAGAIN/-EINTR | `InetCsk::accept` |
| `InetCsk::clone_lock` post: child sk refcount==1, held lock | `InetCsk::clone_lock` |
| `InetCsk::reset_xmit_timer` post: icsk.pending=what; timer scheduled | `InetCsk::reset_xmit_timer` |
| `InetCsk::complete_hashdance` post: child on accept-q if own_req | `InetCsk::complete_hashdance` |

### Layer 4: Verus/Creusot functional

`Per-listen sk → per-SYN → per-syn-recv hash → per-handshake → per-clone_lock child → per-accept-q add → per-accept dequeue` semantic equivalence: per-RFC 793 + per-Linux-TCP-implementation.

### hardening

(Inherits row-1 features from `net/ipv4/00-overview.md` § Hardening.)

INET-csk reinforcement:

- **Per-ephemeral-port range pin and randomized pick** — defense against per-port-prediction.
- **Per-bind-conflict strict check (per-IP + port + reuseport)** — defense against per-stale-listener hijack.
- **Per-accept-q backlog cap** — defense against per-SYN-flood-of-completed-conns.
- **Per-syn-recv-hash with cookies** — defense against per-SYN-flood (TCP-SYN-cookies).
- **Per-icsk_ca_priv buffer fixed-size + ca-init bounded** — defense against per-CC-overflow.
- **Per-icsk_pending exclusive (one timer kind)** — defense against per-stale-timer fire.
- **Per-clone_lock newsk cred inherited carefully** — defense against per-cred-confusion.
- **Per-listen_stop drains accept-q + aborts in-flight reqs** — defense against per-leaked-conn.
- **Per-keepalive only in ESTAB** — defense against per-stray-keepalive-on-non-estab.
- **Per-PMTU probing capped** — defense against per-mtu-probe-DOS.
- **Per-ULP setsockopt validates name** — defense against per-typo-ULP-load.
- **Per-bind2 same-port-IP separation** — defense against per-IP-cross-bind escape.

