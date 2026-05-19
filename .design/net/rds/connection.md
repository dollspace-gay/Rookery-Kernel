# Tier-3: net/rds/connection.c — RDS connection management

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/rds/00-overview.md
upstream-paths:
  - net/rds/connection.c (~990 lines)
  - net/rds/rds.h (struct rds_connection, struct rds_conn_path, RDS_CONN_* states)
  - net/rds/loop.c (loopback transport hookup, rds_loop_net_init / _exit)
  - net/rds/send.c (rds_send_path_reset, rds_send_worker, rds_send_ping)
  - net/rds/recv.c (rds_recv_worker)
  - net/rds/threads.c (rds_connect_worker, rds_shutdown_worker, rds_queue_reconnect)
  - net/rds/cong.c (rds_cong_get_maps, rds_cong_add_conn, rds_cong_remove_conn)
  - net/rds/transport.c (rds_trans_get_preferred, rds_trans_put)
  - include/uapi/linux/rds.h (rds_info_connection, rds6_info_connection, RDS_INFO_*)
-->

## Summary

**RDS** (Reliable Datagram Sockets) provides a connection-oriented, reliable-datagram transport over an underlying carrier (TCP for general-purpose, InfiniBand RC for HPC). `net/rds/connection.c` owns the per-(laddr, faddr, transport, tos, dev_if) `struct rds_connection`, which spans the lifetime of all retransmittable messages between the two endpoints — independent of the underlying transport-level connection lifetime. Per-connection multipath (MPRDS): when `trans->t_mp_capable`, the connection contains `RDS_MPATH_WORKERS` (8) parallel `rds_conn_path`s, each with its own state-atomic, send/retrans queues, workers, and CM mutex. Per-path state machine: `RDS_CONN_DOWN` → `_CONNECTING` → `_UP` → `_DISCONNECTING` → back to `_DOWN`, with `_ERROR` and `_RESETTING` as off-path transitions. State transitions are cmpxchg-atomic via `rds_conn_path_transition`. Per-hash-table `rds_conn_hash[4096]` (12-bit) is keyed by `__inet_ehashfn(lhash, 0, fhash, 0, rds_hash_secret)` of (laddr, faddr) with a randomized boot-time secret. Per-loopback shortcut: if local socket can bind to the destination, transport is rewritten to `rds_loop_transport`. Critical for: distributed RDBMS (Oracle RAC), HPC over IB, message-retransmit guarantees that survive a TCP reset.

This Tier-3 covers `net/rds/connection.c` (~990 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rds_connection` | per-(laddr, faddr, trans, tos, dev_if) connection | `RdsConnection` |
| `struct rds_conn_path` | per-path (MPRDS) state + tx/rx queues | `RdsConnPath` |
| `rds_conn_hash[]` | per-(laddr, faddr) hash bucket array (4096) | `RDS_CONN_HASH` |
| `rds_conn_lock` | per-bucket-modification spinlock | `RdsConnState::lock` |
| `rds_conn_slab` | per-`rds_connection` kmem_cache | `RdsConnState::slab` |
| `rds_conn_bucket()` | per-key bucket lookup | `RdsConn::bucket` |
| `rds_conn_lookup()` | per-bucket linear search under RCU | `RdsConn::lookup` |
| `rds_conn_create()` | per-incoming-listener factory | `RdsConn::create` |
| `rds_conn_create_outgoing()` | per-active-connect factory | `RdsConn::create_outgoing` |
| `__rds_conn_create()` | per-shared create body (in/out, passive) | `RdsConn::create_inner` |
| `__rds_conn_path_init()` | per-path init (workers, queues, state) | `RdsConn::path_init` |
| `rds_conn_path_reset()` | per-path tx/recv state reset | `RdsConn::path_reset` |
| `rds_conn_shutdown()` | per-path quiesce + transport shutdown | `RdsConn::shutdown` |
| `rds_conn_destroy()` | per-connection rmmod free | `RdsConn::destroy` |
| `rds_conn_path_destroy()` | per-path free | `RdsConn::path_destroy` |
| `rds_conn_path_transition()` | per-state atomic cmpxchg | `RdsConn::path_transition` |
| `rds_conn_path_drop()` | per-path force-error → drop | `RdsConn::path_drop` |
| `rds_conn_drop()` | per-conn force-drop (non-mpath) | `RdsConn::drop` |
| `rds_conn_path_connect_if_down()` | per-path reconnect kick | `RdsConn::path_connect_if_down` |
| `rds_conn_connect_if_down()` | per-conn reconnect kick (non-mpath) | `RdsConn::connect_if_down` |
| `rds_check_all_paths()` | per-conn iterate paths reconnect | `RdsConn::check_all_paths` |
| `__rds_conn_path_error()` | per-path printk + drop | `RdsConn::path_error` |
| `rds_for_each_conn_info()` | per-netlink-info iter | `RdsConn::for_each_info` |
| `rds_walk_conn_path_info()` | per-path netlink-info iter | `RdsConn::walk_path_info` |
| `rds_conn_info()` / `rds6_conn_info()` | per-RDS_INFO_CONNECTIONS visitor | `RdsConn::info_v4` / `info_v6` |
| `rds_conn_message_info_send/_retrans` | per-RDS_INFO_*_MESSAGES visitor | `RdsConn::msg_info_*` |
| `rds_conn_init()` / `rds_conn_exit()` | per-module init/exit | `RdsConn::init` / `exit` |
| `RDS_CONN_DOWN/_CONNECTING/_DISCONNECTING/_UP/_RESETTING/_ERROR` | per-state enum | `RdsConnState` |
| `RDS_MPATH_WORKERS` (8) | per-multipath path count | `RDS_MPATH_WORKERS` |
| `RDS_CONNECTION_HASH_BITS` (12) | per-hash size | `RDS_CONNECTION_HASH_BITS` |

## Compatibility contract

REQ-1: struct rds_connection:
- c_hash_node: hlist_node into `rds_conn_hash[bucket]`.
- c_laddr / c_faddr: in6_addr (v4-mapped for IPv4).
- c_isv6: bool — laddr is non-v4-mapped IPv6.
- c_dev_if: int — bound netdev index (used for IPv6 link-local).
- c_bound_if: int — derived bind interface (0 if not link-local).
- c_tos: u8 — RDS QoS class.
- c_trans: `*rds_transport` — current transport (TCP / IB / loop).
- c_loopback: bool — is loop-resolved.
- c_passive: `*rds_connection` — passive companion for IB loopback (active+passive pair).
- c_path: `*rds_conn_path` — array of `npaths` paths (1 or RDS_MPATH_WORKERS).
- c_npaths: u8 — count (1 or RDS_MPATH_WORKERS).
- c_hs_waitq: handshake wait queue.
- c_proposed_version: protocol version negotiation seed (RDS_PROTOCOL_VERSION).
- c_my_gen_num: per-boot generation number (peer-restart detection).
- c_peer_gen_num: peer's reported gen.
- c_net: net namespace.

REQ-2: struct rds_conn_path:
- cp_lock: spinlock guarding cp_send_queue / cp_retrans.
- cp_state: atomic_t state (one of RDS_CONN_*).
- cp_flags: bitset (RDS_IN_XMIT, RDS_RECONNECT_PENDING, RDS_RECV_REFILL).
- cp_next_tx_seq / cp_next_rx_seq: u64 sequence numbers.
- cp_send_queue / cp_retrans: per-path queued / in-flight messages.
- cp_xmit_rm: currently-transmitting rds_message.
- cp_send_w / cp_recv_w / cp_conn_w: delayed_work for send/recv/connect.
- cp_down_w: work for shutdown.
- cp_wq: per-path ordered workqueue (`krds_cp_wq#<count>/<index>`); falls back to `rds_wq`.
- cp_cm_lock: mutex serializing CM (connection-manager) events vs shutdown.
- cp_waitq: wait-for-RDS_IN_XMIT/_RECV_REFILL completion.
- cp_index: u8 path index.
- cp_conn: back-pointer to rds_connection.
- cp_transport_data: transport's per-path private (rds_ib_connection / rds_tcp_connection).
- cp_send_gen: send-generation counter.
- cp_reconnect_jiffies: backoff timestamp.

REQ-3: rds_conn_bucket(laddr, faddr) → bucket-head:
- Per-net_get_random_once init `rds_hash_secret` and `rds6_hash_secret` lazily.
- /* IPv4-mapped */ lhash = laddr.s6_addr32[3].
- /* IPv6 */ fhash = `__ipv6_addr_jhash(faddr, rds6_hash_secret)`; /* v4 */ fhash = faddr.s6_addr32[3].
- hash = `__inet_ehashfn(lhash, 0, fhash, 0, rds_hash_secret)`.
- return `&rds_conn_hash[hash & RDS_CONNECTION_HASH_MASK]`.

REQ-4: rds_conn_lookup(net, head, laddr, faddr, trans, tos, dev_if):
- /* Caller holds rcu_read_lock OR rds_conn_lock */
- For each `conn` in `head` via `hlist_for_each_entry_rcu`:
  - if `ipv6_addr_equal(c_faddr, faddr)` ∧ `ipv6_addr_equal(c_laddr, laddr)` ∧ `c_trans == trans` ∧ `c_tos == tos` ∧ `rds_conn_net(conn) == net` ∧ `c_dev_if == dev_if`: return conn.
- return NULL.

REQ-5: __rds_conn_create(net, laddr, faddr, trans, gfp, tos, is_outgoing, dev_if) → conn:
- /* Pre-lookup: race-tolerant pre-allocation */
- npaths = `trans->t_mp_capable ? RDS_MPATH_WORKERS : 1`.
- rcu_read_lock; conn = rds_conn_lookup(...).
- /* IB-loopback active-conn case: caller is incoming, find peer's outgoing as parent */
- if `conn` ∧ `c_loopback` ∧ `c_trans != &rds_loop_transport` ∧ `ipv6_addr_equal(laddr, faddr)` ∧ `!is_outgoing`: parent = conn; conn = parent->c_passive.
- rcu_read_unlock; if conn: return existing.
- /* Allocate */
- conn = `kmem_cache_zalloc(rds_conn_slab, gfp)`; on OOM: -ENOMEM.
- conn->c_path = kzalloc_objs(struct rds_conn_path, npaths, gfp); on OOM: free + -ENOMEM.
- Init c_laddr, c_faddr, c_isv6, c_dev_if, c_tos, c_bound_if.
- rds_conn_net_set(conn, net).
- /* Congestion maps */ rds_cong_get_maps(conn); on err: free + return err.
- /* Loopback detection */
- loop_trans = `rds_trans_get_preferred(net, faddr, c_dev_if)`.
- if loop_trans:
  - rds_trans_put(loop_trans); c_loopback = 1.
  - if trans->t_prefer_loopback ∧ is_outgoing: trans = `&rds_loop_transport`.
  - if trans->t_prefer_loopback ∧ !is_outgoing: free + -EOPNOTSUPP.
- conn->c_trans = trans.
- /* Init each path */
- for i in 0..npaths: `__rds_conn_path_init(conn, &c_path[i], is_outgoing)`; cp_index = i.
  - cp_wq = `alloc_ordered_workqueue("krds_cp_wq#%lu/%d", 0, rds_conn_count, i)`; on fail: cp_wq = rds_wq.
- /* Transport alloc */
- rcu_read_lock; if rds_destroy_pending(conn): ret = -ENETDOWN; else ret = trans->conn_alloc(conn, GFP_ATOMIC).
- if ret: rcu_read_unlock; free; return err.
- /* Publish under spinlock; recheck after */
- spin_lock_irqsave(&rds_conn_lock, flags).
- if parent (passive):
  - if parent->c_passive: rollback conn (race lost); conn = parent->c_passive.
  - else: parent->c_passive = conn; rds_cong_add_conn(conn); rds_conn_count++.
- else (normal):
  - found = rds_conn_lookup(...).
  - if found: rollback; conn = found.
  - else: c_my_gen_num = rds_gen_num; c_peer_gen_num = 0; `hlist_add_head_rcu(&c_hash_node, head)`; rds_cong_add_conn; rds_conn_count++.
- spin_unlock_irqrestore; rcu_read_unlock.

REQ-6: __rds_conn_path_init(conn, cp, is_outgoing):
- spin_lock_init(&cp_lock).
- cp_next_tx_seq = 1.
- init_waitqueue_head(&cp_waitq).
- INIT_LIST_HEAD(&cp_send_queue, &cp_retrans).
- cp_conn = conn; atomic_set(&cp_state, RDS_CONN_DOWN); cp_send_gen = 0; cp_reconnect_jiffies = 0.
- conn->c_proposed_version = RDS_PROTOCOL_VERSION.
- INIT_DELAYED_WORK(&cp_send_w, rds_send_worker).
- INIT_DELAYED_WORK(&cp_recv_w, rds_recv_worker).
- INIT_DELAYED_WORK(&cp_conn_w, rds_connect_worker).
- INIT_WORK(&cp_down_w, rds_shutdown_worker).
- mutex_init(&cp_cm_lock); cp_flags = 0.

REQ-7: rds_conn_shutdown(cp):
- if `!rds_conn_path_transition(cp, RDS_CONN_DOWN, RDS_CONN_DOWN)` (i.e., not already DOWN):
  - mutex_lock(&cp_cm_lock).
  - /* Transition any of {UP, ERROR, RESETTING} → DISCONNECTING */
  - if none of `path_transition(cp, RDS_CONN_UP, _DISCONNECTING)`, `(RDS_CONN_ERROR, _DISCONNECTING)`, `(RDS_CONN_RESETTING, _DISCONNECTING)` succeeded:
    - rds_conn_path_error("shutdown called in state %d"); mutex_unlock; return.
  - mutex_unlock(&cp_cm_lock).
  - wait_event(cp_waitq, !test_bit(RDS_IN_XMIT, &cp_flags)).
  - wait_event(cp_waitq, !test_bit(RDS_RECV_REFILL, &cp_flags)).
  - c_trans->conn_path_shutdown(cp).
  - rds_conn_path_reset(cp).
  - /* DISCONNECTING / ERROR → DOWN */
  - if neither `path_transition(cp, RDS_CONN_DISCONNECTING, RDS_CONN_DOWN)` nor `(RDS_CONN_ERROR, RDS_CONN_DOWN)` succeeded:
    - rds_conn_path_error("failed to transition to state DOWN"); return.
- cancel_delayed_work_sync(&cp_conn_w).
- clear_bit(RDS_RECONNECT_PENDING, &cp_flags).
- rcu_read_lock; if `!hlist_unhashed(&conn->c_hash_node)`:
  - rcu_read_unlock.
  - /* mpath ping on path 0 */
  - if conn->c_trans->t_mp_capable ∧ cp_index == 0: rds_send_ping(conn, 0).
  - rds_queue_reconnect(cp).
- else: rcu_read_unlock.
- if c_trans->conn_slots_available: c_trans->conn_slots_available(conn, false).

REQ-8: rds_conn_path_reset(cp):
- rds_stats_inc(s_conn_reset).
- rds_send_path_reset(cp) — clears partial send / retrans state.
- cp_flags = 0.
- /* Note: do NOT clear cp_next_rx_seq — preserves dedup across reconnects */.

REQ-9: rds_conn_destroy(conn):
- /* Module-unload only — caller must guarantee no further refs */
- spin_lock_irq(&rds_conn_lock); `hlist_del_init_rcu(&c_hash_node)`; spin_unlock_irq.
- synchronize_rcu().
- npaths = c_trans->t_mp_capable ? RDS_MPATH_WORKERS : 1.
- for i in 0..npaths: rds_conn_path_destroy(&c_path[i]); BUG_ON(!list_empty(&c_path[i].cp_retrans)).
- rds_cong_remove_conn(conn).
- kfree(c_path); kmem_cache_free(rds_conn_slab, conn).
- spin_lock_irqsave(&rds_conn_lock, flags); rds_conn_count--; spin_unlock_irqrestore.

REQ-10: rds_conn_path_destroy(cp):
- if `!cp_transport_data`: return.
- cancel_delayed_work_sync(&cp_send_w, &cp_recv_w).
- rds_conn_path_drop(cp, /*destroy=*/true).
- flush_work(&cp_down_w).
- /* Drain cp_send_queue */
- list_for_each_entry_safe(rm in cp_send_queue): list_del_init(&rm->m_conn_item); BUG_ON(!list_empty(&rm->m_sock_item)); rds_message_put(rm).
- if cp_xmit_rm: rds_message_put(cp_xmit_rm).
- WARN_ON pending delayed/work.
- if cp_wq != rds_wq: destroy_workqueue(cp_wq); cp_wq = NULL.
- c_trans->conn_free(cp_transport_data).

REQ-11: rds_conn_path_transition(cp, old, new) → bool:
- `atomic_cmpxchg(&cp_state, old, new) == old`.

REQ-12: Per-state transitions (legal):
- DOWN → CONNECTING (rds_connect_worker on path_connect_if_down).
- CONNECTING → UP (transport connect complete).
- CONNECTING → DOWN (handshake failure path).
- UP → DISCONNECTING (shutdown).
- UP → ERROR (path_drop).
- ERROR → DISCONNECTING (shutdown after drop).
- ERROR → DOWN (recovery without intermediate).
- RESETTING → DISCONNECTING (reset path).
- DISCONNECTING → DOWN (final).
- DOWN → DOWN (idempotent — rds_conn_shutdown nop path).

REQ-13: rds_conn_path_drop(cp, destroy):
- atomic_set(&cp_state, RDS_CONN_ERROR).
- rcu_read_lock; if `!destroy ∧ rds_destroy_pending(cp_conn)`: rcu_read_unlock; return.
- queue_work(cp_wq, &cp_down_w); rcu_read_unlock.

REQ-14: rds_conn_path_connect_if_down(cp):
- rcu_read_lock; if rds_destroy_pending(cp_conn): rcu_read_unlock; return.
- if `rds_conn_path_state(cp) == RDS_CONN_DOWN` ∧ `!test_and_set_bit(RDS_RECONNECT_PENDING, &cp_flags)`:
  - queue_delayed_work(cp_wq, &cp_conn_w, 0).
- rcu_read_unlock.

REQ-15: rds_for_each_conn_info(...) — netlink visitor:
- rcu_read_lock.
- For each bucket in `rds_conn_hash`, for each `conn` in bucket (hlist_for_each_entry_rcu):
  - memset(buffer, 0, item_len) — zero so unused fields / padding cannot leak stack to user.
  - if visitor(conn, buffer): if len >= item_len: rds_info_copy(iter, buffer, item_len); len -= item_len.
  - lens->nr++.
- rcu_read_unlock.

REQ-16: rds_conn_init():
- rds_loop_net_init() — register per-net loopback callback; on err: return.
- rds_conn_slab = KMEM_CACHE(rds_connection, 0); on null: rds_loop_net_exit; -ENOMEM.
- rds_info_register_func(RDS_INFO_CONNECTIONS, rds_conn_info).
- rds_info_register_func(RDS_INFO_SEND_MESSAGES, ...send).
- rds_info_register_func(RDS_INFO_RETRANS_MESSAGES, ...retrans).
- IPv6: register RDS6_INFO_{CONNECTIONS, SEND_MESSAGES, RETRANS_MESSAGES}.

REQ-17: rds_conn_exit():
- rds_loop_net_exit(); rds_loop_exit().
- `WARN_ON(!hlist_empty(rds_conn_hash))` — leak guard.
- kmem_cache_destroy(rds_conn_slab).
- rds_info_deregister_func for all keys above.

## Acceptance Criteria

- [ ] AC-1: rds_conn_create with (laddr, faddr, trans, tos, dev_if): hash bucket = `__inet_ehashfn(...)` mod 4096; bucket list contains the new conn after publish.
- [ ] AC-2: Concurrent rds_conn_create for same key from two CPUs: exactly one entry in bucket; the loser's allocation freed via conn_free and slab.
- [ ] AC-3: t_mp_capable transport: conn->c_path has RDS_MPATH_WORKERS (8) paths; each has its own cp_wq.
- [ ] AC-4: rds_conn_path_transition: cmpxchg atomic; concurrent CONNECTING→UP and CONNECTING→DOWN: exactly one succeeds.
- [ ] AC-5: rds_conn_shutdown: from RDS_CONN_UP: transitions UP → DISCONNECTING → DOWN; waits for RDS_IN_XMIT and RDS_RECV_REFILL clear.
- [ ] AC-6: rds_conn_shutdown from RDS_CONN_DOWN: nop (initial cmpxchg succeeds; fast-path).
- [ ] AC-7: rds_conn_path_drop: sets state to RDS_CONN_ERROR and queues cp_down_w.
- [ ] AC-8: rds_conn_path_connect_if_down: only from DOWN ∧ !RDS_RECONNECT_PENDING: sets bit and queues cp_conn_w; idempotent otherwise.
- [ ] AC-9: Loopback detection: rds_trans_get_preferred returns non-NULL: c_loopback = 1; if trans->t_prefer_loopback ∧ is_outgoing: trans rewritten to `rds_loop_transport`.
- [ ] AC-10: rds_conn_destroy: hlist_del_init_rcu under rds_conn_lock; synchronize_rcu before path teardown; cp_retrans empty BUG_ON.
- [ ] AC-11: rds_for_each_conn_info: buffer zeroed before visitor invocation; no padding leak to userspace.
- [ ] AC-12: rds_conn_path_reset: cp_next_rx_seq preserved (dedup invariant); cp_flags cleared; s_conn_reset incremented.
- [ ] AC-13: rds_conn_exit: empty hash WARN_ON; slab destroyed; info callbacks deregistered.

## Architecture

```
struct RdsConnection {
  c_hash_node: HlistNode,
  c_laddr: In6Addr,
  c_faddr: In6Addr,
  c_isv6: bool,
  c_dev_if: i32,
  c_bound_if: i32,
  c_tos: u8,
  c_trans: *RdsTransport,
  c_loopback: bool,
  c_passive: Option<*RdsConnection>,
  c_path: *[RdsConnPath],
  c_npaths: u8,
  c_hs_waitq: WaitQueue,
  c_proposed_version: u32,
  c_my_gen_num: u32,
  c_peer_gen_num: u32,
  c_net: *Net,
}

struct RdsConnPath {
  cp_lock: SpinLock<()>,
  cp_state: AtomicI32,                // RDS_CONN_*
  cp_flags: AtomicUsize,              // RDS_IN_XMIT | _RECONNECT_PENDING | _RECV_REFILL
  cp_next_tx_seq: u64,
  cp_next_rx_seq: u64,
  cp_send_queue: ListHead,            // struct rds_message, m_conn_item
  cp_retrans: ListHead,
  cp_xmit_rm: Option<*RdsMessage>,
  cp_send_w: DelayedWork,
  cp_recv_w: DelayedWork,
  cp_conn_w: DelayedWork,
  cp_down_w: Work,
  cp_wq: *Workqueue,                  // ordered, "krds_cp_wq#<n>/<idx>"
  cp_cm_lock: Mutex<()>,
  cp_waitq: WaitQueue,
  cp_index: u8,
  cp_conn: *RdsConnection,
  cp_transport_data: *mut (),
  cp_send_gen: u64,
  cp_reconnect_jiffies: u64,
}
```

`RdsConn::create_inner(net, laddr, faddr, trans, gfp, tos, is_outgoing, dev_if) -> Result<*RdsConnection>`:
1. npaths = if trans.t_mp_capable { RDS_MPATH_WORKERS } else { 1 }.
2. head = RdsConn::bucket(laddr, faddr).
3. /* Optimistic lookup */
4. rcu_read_lock().
5. conn = RdsConn::lookup(net, head, laddr, faddr, trans, tos, dev_if).
6. /* IB-loopback passive */
7. if conn ∧ conn.c_loopback ∧ conn.c_trans != &RDS_LOOP_TRANSPORT ∧ laddr == faddr ∧ !is_outgoing:
   - parent = conn; conn = parent.c_passive.
8. rcu_read_unlock().
9. if conn.is_some(): return Ok(conn).
10. /* Allocate */
11. conn = kmem_cache_zalloc(RDS_CONN_SLAB, gfp).ok_or(-ENOMEM)?.
12. conn.c_path = kzalloc_objs::<RdsConnPath>(npaths, gfp).map_err(|_| { kmem_cache_free(...); -ENOMEM })?.
13. /* Init scalar fields */
14. conn.c_laddr = *laddr; conn.c_isv6 = !ipv6_addr_v4mapped(laddr); conn.c_faddr = *faddr.
15. conn.c_dev_if = dev_if; conn.c_tos = tos.
16. conn.c_bound_if = if ipv6_addr_type(laddr) & IPV6_ADDR_LINKLOCAL { dev_if } else { 0 }.
17. rds_conn_net_set(conn, net).
18. rds_cong_get_maps(conn)?  // on err: free path + slab.
19. /* Loopback resolution */
20. if let Some(lt) = rds_trans_get_preferred(net, faddr, conn.c_dev_if):
   - rds_trans_put(lt). conn.c_loopback = true.
   - if trans.t_prefer_loopback:
     - if is_outgoing: trans = &RDS_LOOP_TRANSPORT.
     - else: free + return Err(-EOPNOTSUPP).
21. conn.c_trans = trans.
22. /* Path init */
23. for i in 0..npaths:
   - RdsConn::path_init(conn, &conn.c_path[i], is_outgoing).
   - conn.c_path[i].cp_index = i.
   - conn.c_path[i].cp_wq = alloc_ordered_workqueue("krds_cp_wq#{}/{}", 0, rds_conn_count, i)
     .unwrap_or(RDS_WQ).
24. /* Transport alloc under rcu */
25. rcu_read_lock().
26. let ret = if rds_destroy_pending(conn) { Err(-ENETDOWN) } else { trans.conn_alloc(conn, GFP_ATOMIC) }.
27. if ret.is_err(): rcu_read_unlock(); free path + slab; return ret.
28. /* Publish + race recheck */
29. spin_lock_irqsave(&RDS_CONN_LOCK).
30. if let Some(parent) = parent:
   - if parent.c_passive.is_some(): /* race lost */ rollback conn; conn = parent.c_passive.
   - else: parent.c_passive = conn; rds_cong_add_conn(conn); rds_conn_count++.
31. else:
   - found = RdsConn::lookup(net, head, ...).
   - if found.is_some(): rollback; conn = found.
   - else: conn.c_my_gen_num = rds_gen_num; conn.c_peer_gen_num = 0; hlist_add_head_rcu(&conn.c_hash_node, head); rds_cong_add_conn(conn); rds_conn_count++.
32. spin_unlock_irqrestore.
33. rcu_read_unlock.
34. Ok(conn).

`RdsConn::path_transition(cp, old, new) -> bool`:
1. `atomic_cmpxchg(&cp.cp_state, old, new) == old`.

`RdsConn::shutdown(cp)`:
1. /* Fast path: already DOWN → just kick reconnect */
2. if !RdsConn::path_transition(cp, RDS_CONN_DOWN, RDS_CONN_DOWN):
   - /* Was not DOWN — start shutdown */
   - mutex_lock(&cp.cp_cm_lock).
   - let any_to_dis = path_transition(cp, RDS_CONN_UP, RDS_CONN_DISCONNECTING)
                     || path_transition(cp, RDS_CONN_ERROR, RDS_CONN_DISCONNECTING)
                     || path_transition(cp, RDS_CONN_RESETTING, RDS_CONN_DISCONNECTING).
   - if !any_to_dis: RdsConn::path_error(cp, "shutdown called in state %d"); mutex_unlock; return.
   - mutex_unlock(&cp.cp_cm_lock).
   - wait_event(cp.cp_waitq, !cp.cp_flags.contains(RDS_IN_XMIT)).
   - wait_event(cp.cp_waitq, !cp.cp_flags.contains(RDS_RECV_REFILL)).
   - cp.cp_conn.c_trans.conn_path_shutdown(cp).
   - RdsConn::path_reset(cp).
   - if !path_transition(cp, RDS_CONN_DISCONNECTING, RDS_CONN_DOWN)
        ∧ !path_transition(cp, RDS_CONN_ERROR, RDS_CONN_DOWN):
       - RdsConn::path_error(cp, "failed to transition to state DOWN"); return.
3. cancel_delayed_work_sync(&cp.cp_conn_w).
4. clear_bit(RDS_RECONNECT_PENDING, &cp.cp_flags).
5. rcu_read_lock.
6. if !hlist_unhashed(&cp.cp_conn.c_hash_node):
   - rcu_read_unlock.
   - if cp.cp_conn.c_trans.t_mp_capable ∧ cp.cp_index == 0: rds_send_ping(cp.cp_conn, 0).
   - rds_queue_reconnect(cp).
7. else: rcu_read_unlock.
8. if let Some(slots) = cp.cp_conn.c_trans.conn_slots_available: slots(cp.cp_conn, false).

`RdsConn::path_drop(cp, destroy)`:
1. atomic_set(&cp.cp_state, RDS_CONN_ERROR).
2. rcu_read_lock.
3. if !destroy ∧ rds_destroy_pending(cp.cp_conn): rcu_read_unlock; return.
4. queue_work(cp.cp_wq, &cp.cp_down_w).
5. rcu_read_unlock.

`RdsConn::destroy(conn)`:
1. spin_lock_irq(&RDS_CONN_LOCK).
2. hlist_del_init_rcu(&conn.c_hash_node).
3. spin_unlock_irq.
4. synchronize_rcu().
5. let npaths = if conn.c_trans.t_mp_capable { RDS_MPATH_WORKERS } else { 1 }.
6. for i in 0..npaths:
   - RdsConn::path_destroy(&conn.c_path[i]).
   - BUG_ON(!list_empty(&conn.c_path[i].cp_retrans)).
7. rds_cong_remove_conn(conn).
8. kfree(conn.c_path); kmem_cache_free(RDS_CONN_SLAB, conn).
9. spin_lock_irqsave(&RDS_CONN_LOCK, flags); rds_conn_count -= 1; spin_unlock_irqrestore.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bucket_lookup_under_rcu_or_lock` | INVARIANT | per-rds_conn_lookup: caller holds rcu_read_lock OR rds_conn_lock. |
| `publish_under_rds_conn_lock` | INVARIANT | per-__rds_conn_create: hlist_add_head_rcu only under rds_conn_lock. |
| `state_transition_cmpxchg_atomic` | INVARIANT | per-path_transition: cmpxchg only mechanism for cp_state changes. |
| `shutdown_under_cp_cm_lock` | INVARIANT | per-shutdown: UP/_ERROR/_RESETTING → _DISCONNECTING transitions under cp_cm_lock. |
| `path_destroy_cancels_workers` | INVARIANT | per-path_destroy: cp_send_w, cp_recv_w cancelled before conn_free. |
| `destroy_synchronize_rcu_before_free` | INVARIANT | per-destroy: synchronize_rcu after hlist_del_init_rcu, before path teardown. |
| `info_buffer_zeroed` | INVARIANT | per-walk_conn_path_info / for_each_conn_info: buffer memset to 0 before visitor. |
| `passive_paired_under_rds_conn_lock` | INVARIANT | per-create: parent->c_passive set only under rds_conn_lock. |

### Layer 2: TLA+

`net/rds/connection.tla`:
- States × {DOWN, CONNECTING, DISCONNECTING, UP, RESETTING, ERROR} × {hash-bucket-set, refcount}.
- Actions: create, connect_if_down, transition, drop, shutdown, destroy.
- Properties:
  - `safety_state_machine_legal` — per-cp_state: only legal transitions enumerated in REQ-12.
  - `safety_one_conn_per_key` — per-(net, laddr, faddr, trans, tos, dev_if): at most one in bucket (passive excluded).
  - `safety_destroy_after_synchronize_rcu` — per-destroy: hash_node unlinked ∧ rcu-quiesce ∧ then path_destroy.
  - `safety_path_reset_preserves_rx_seq` — per-path_reset: cp_next_rx_seq invariant.
  - `liveness_shutdown_reaches_DOWN` — per-shutdown from UP/_ERROR/_RESETTING: eventually cp_state = DOWN ∨ error logged.
  - `liveness_connect_if_down_eventually_CONNECTING` — per-DOWN ∧ !destroy_pending: cp_conn_w eventually scheduled.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `RdsConn::create_inner` post: returns existing or publishes new in `rds_conn_hash[bucket]` | `RdsConn::create_inner` |
| `RdsConn::path_init` post: cp_state = RDS_CONN_DOWN ∧ cp_next_tx_seq = 1 ∧ cp_flags = 0 | `RdsConn::path_init` |
| `RdsConn::path_transition` post: returns true iff prior state == old (cmpxchg semantics) | `RdsConn::path_transition` |
| `RdsConn::shutdown` post: cp_state ∈ {DOWN} ∨ error logged; cp_conn_w cancelled | `RdsConn::shutdown` |
| `RdsConn::path_drop` post: cp_state = RDS_CONN_ERROR; cp_down_w queued if not pending-destroy | `RdsConn::path_drop` |
| `RdsConn::destroy` post: hlist_del_init_rcu + synchronize_rcu + all paths destroyed + slab freed | `RdsConn::destroy` |
| `RdsConn::path_reset` post: s_conn_reset incremented; cp_flags = 0; cp_next_rx_seq unchanged | `RdsConn::path_reset` |
| `RdsConn::for_each_info` post: per-visitor-buffer memset(0) before visitor | `RdsConn::for_each_info` |

### Layer 4: Verus/Creusot functional

`Per-connection lifecycle (CREATE → CONNECT → UP → SHUTDOWN → RECONNECT-or-DESTROY) ↔ struct rds_connection state machine` semantic equivalence: per-`Documentation/networking/rds.rst` and per-RFC-less RDS protocol (Oracle-published spec). Multipath fan-out + per-path independent state machines: equivalent to RDS_MPATH_WORKERS independent atomic cmpxchg-driven FSMs sharing one rds_connection envelope.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

RDS-connection reinforcement:

- **Per-(net, laddr, faddr, trans, tos, dev_if) uniqueness under rds_conn_lock** — defense against per-duplicate-conn UAF on shutdown race.
- **Per-cp_state cmpxchg-only mutation** — defense against per-torn-state-write across CPUs.
- **Per-cp_cm_lock around state→DISCONNECTING** — defense against per-CM-event vs shutdown race.
- **Per-synchronize_rcu in destroy after hash_del** — defense against per-RCU-reader UAF on conn pointer.
- **Per-cp_retrans BUG_ON empty at destroy** — defense against per-leaked-message refcount.
- **Per-info-buffer memset(0) before visitor** — defense against per-stack-padding-leak to userspace (CVE class).
- **Per-bucket hash randomized at boot via net_get_random_once** — defense against per-DoS hash-flood.
- **Per-conn_alloc under rcu + rds_destroy_pending guard** — defense against per-conn-alloc-during-netns-teardown.
- **Per-cp_wq ordered, fallback to rds_wq** — defense against per-wq-OOM total path failure.
- **Per-cp_next_rx_seq preserved across reset** — defense against per-duplicate-delivery on reconnect (RDS reliability contract).
- **Per-IB-loopback active+passive paired under rds_conn_lock** — defense against per-double-passive race.
- **Per-RDS_MPATH_WORKERS bounded** — defense against per-fan-out-amplification DoS.
- **Per-hlist_unhashed check before queue_reconnect** — defense against per-reconnect-after-destroy.

## Grsecurity/PaX-style Reinforcement

Baseline kernel-wide hardening leveraged by `rds_connection`:

- **PAX_USERCOPY** — `rds_connection`, `rds_conn_path`, and `rds_message` allocations live in slab caches with PAX_USERCOPY-whitelisted copy windows; `getsockopt(RDS_INFO_CONNECTIONS)` cannot bleed adjacent kernel state.
- **PAX_KERNEXEC** — `rds_transport`, `rds_conn_ops`, and per-transport callbacks (`conn_alloc`, `conn_path_connect`, `xmit`) live in `__ro_after_init`; a corrupted conn cannot pivot through forged transport ops.
- **PAX_RANDKSTACK** — `rds_conn_create`, `rds_conn_path_connect_if_down`, and `rds_send_xmit` stack frames randomized on each entry.
- **PAX_REFCOUNT** — `conn->c_refcount`, `cp->cp_refcount`, and per-message refs use saturating `refcount_t`; multipath fan-out cannot wrap.
- **PAX_MEMORY_SANITIZE** — `rds_conn_destroy` zeroes per-path state and message slots, erasing residual cp_xmit_rm / cp_next_rx_seq state.
- **PAX_UDEREF** — sockopt parsing (`rds_setsockopt`) and `rds_info` getters operate only on kernel-side copies; no user-pointer deref in transport.
- **PAX_RAP/kCFI** — `t_conn_path_connect`/`xmit`/`stats_info_copy` dispatch via forward-edge-checked transport vtables.
- **GRKERNSEC_HIDESYM** — `/proc/net/rds/connections`, `/sys/kernel/debug/rds/*` strip `rds_connection` and per-cp kernel pointers.
- **GRKERNSEC_DMESG** — RDS-IB / RDS-TCP connect-fail `printk` paths rate-limited; cannot flood dmesg to fingerprint hash seed.

RDS-connection-specific reinforcement:

- **RDS control-plane requires CAP_NET_ADMIN in user-ns** for socket-options that alter transport (bind to non-zero saddr, MPATH workers); strict.
- **`rds_conn_create` runs under RCU + `rds_destroy_pending` guard** — netns teardown cannot race with conn alloc.
- **MPATH path-array bounded by `RDS_MPATH_WORKERS`** — fan-out amplification capped, PAX_REFCOUNT saturates per-path ref.
- **hash buckets seeded by `net_get_random_once`** — hash-flood DoS denied; seed not exposed via HIDESYM.

Rationale: RDS has been a recurring CVE source (CVE-2010-3904, CVE-2018-5333, CVE-2019-16714); strict CAP_NET_ADMIN gating, saturating multipath refs, and bounded per-conn fan-out close the historical exploit primitives.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/rds/send.c message-tx pipeline (covered separately if expanded)
- net/rds/recv.c rds_recv path + RDS_INFO_RECV_MESSAGES (covered separately)
- net/rds/threads.c rds_connect_worker / rds_shutdown_worker (covered separately)
- net/rds/cong.c congestion-map machinery (covered separately)
- net/rds/tcp*.c TCP transport (covered separately)
- net/rds/ib*.c InfiniBand transport (covered separately)
- net/rds/loop.c loopback transport (covered separately)
- net/rds/info.c RDS_INFO netlink-info infrastructure (covered separately)
- Implementation code
