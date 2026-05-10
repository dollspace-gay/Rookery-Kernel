# Tier-3: net/rxrpc/call_object.c — RxRPC call object lifecycle

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/rxrpc/00-overview.md
upstream-paths:
  - net/rxrpc/call_object.c (~773 lines)
  - net/rxrpc/ar-internal.h (struct rxrpc_call, enum rxrpc_call_state, enum rxrpc_call_completion)
  - net/rxrpc/conn_client.c (rxrpc_look_up_bundle, rxrpc_get_bundle, rxrpc_deactivate_bundle)
  - net/rxrpc/conn_object.c (rxrpc_put_connection, rxrpc_get_peer)
  - net/rxrpc/local_object.c (rxrpc_get_local, rxrpc_put_local, rxrpc_wake_up_io_thread)
  - net/rxrpc/peer_object.c (rxrpc_get_peer, rxrpc_put_peer)
  - net/rxrpc/call_state.c (rxrpc_set_call_state, rxrpc_set_call_completion, rxrpc_prefail_call)
  - net/rxrpc/security.c (rxrpc_init_client_call_security)
  - net/rxrpc/txbuf.c (rxrpc_put_txbuf)
  - net/rxrpc/sendmsg.c (rxrpc_propose_abort)
  - include/net/af_rxrpc.h (rxrpc_kernel_* kAFS API)
-->

## Summary

**RxRPC** is the Rx-protocol RPC transport (originally Coda/AFS, now used by **kAFS** in-kernel AFS client) running over UDP. Every remote-procedure invocation is a `struct rxrpc_call` — an independently addressable, sequence-numbered conversation identified by `(local-endpoint, peer, cid, callNumber)`. `net/rxrpc/call_object.c` owns the per-call lifecycle: allocation from the `rxrpc_call_jar` kmem_cache, state-machine progression (`UNINITIALISED` → `CLIENT_AWAIT_CONN` → `CLIENT_SEND_REQUEST` → `CLIENT_AWAIT_REPLY` → `CLIENT_RECV_REPLY` → `COMPLETE`, or server-side `SERVER_PREALLOC` → `SERVER_RECV_REQUEST` → `SERVER_ACK_REQUEST` → `SERVER_SEND_REPLY` → `SERVER_AWAIT_ACK` → `COMPLETE`), socket attachment via `rb_tree` keyed by `user_call_ID`, refcount-driven destruction, and abort-on-error completion. Per-`rxrpc_call_limiter` (1000) / `rxrpc_kernel_call_limiter` (1000) semaphores bound concurrent outstanding calls separately for user vs kernel callers. Per-call retransmit-tx queue (`tx_queue` of `rxrpc_txqueue`), per-call rx-reassembly + out-of-order queues (`rx_queue`, `rx_oos_queue`, `recvmsg_queue`), per-call timer (delay-ack, RACK, ping, keepalive, expect-rx, expect-req, hard-timo, expect-term-by), per-call congestion (cong_cwnd, cong_ssthresh). Per-`RXRPC_CALL_EXCLUSIVE` flag: once-only connection (no channel sharing). Critical for: kAFS client/server reliability, in-kernel filesystem-over-network correctness, RPC slot fairness.

This Tier-3 covers `net/rxrpc/call_object.c` (~773 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rxrpc_call` | per-RPC-call object | `RxrpcCall` |
| `enum rxrpc_call_state` | per-call FSM | `RxrpcCallState` |
| `enum rxrpc_call_completion` | per-completion classification | `RxrpcCallCompletion` |
| `rxrpc_call_states[]` | per-state debug-string table | `RXRPC_CALL_STATES` |
| `rxrpc_call_completions[]` | per-completion debug-string table | `RXRPC_CALL_COMPLETIONS` |
| `rxrpc_call_jar` | per-rxrpc_call kmem_cache | `RxrpcCallState::jar` |
| `rxrpc_call_limiter` | per-user-call-slot semaphore (1000) | `RXRPC_CALL_LIMITER` |
| `rxrpc_kernel_call_limiter` | per-kernel-call-slot semaphore (1000) | `RXRPC_KERNEL_CALL_LIMITER` |
| `rxrpc_alloc_call()` | per-call zalloc + scalar init | `RxrpcCall::alloc` |
| `rxrpc_alloc_client_call()` | per-client-call setup + state=AWAIT_CONN | `RxrpcCall::alloc_client` |
| `rxrpc_new_client_call()` | per-sendmsg public entry | `RxrpcCall::new_client` |
| `rxrpc_incoming_call()` | per-server-side dispatch | `RxrpcCall::incoming` |
| `rxrpc_connect_call()` | per-bundle-lookup + IO-thread queue | `RxrpcCall::connect` |
| `rxrpc_find_call_by_user_ID()` | per-rb_tree lookup by user_call_ID | `RxrpcCall::find_by_user_id` |
| `rxrpc_release_call()` | per-detach-from-socket | `RxrpcCall::release` |
| `rxrpc_release_calls_on_socket()` | per-sock close drain | `RxrpcCall::release_on_socket` |
| `rxrpc_get_call()` / `rxrpc_try_get_call()` / `rxrpc_put_call()` | per-refcount get/try/put | `RxrpcCall::get` / `try_get` / `put` |
| `rxrpc_see_call()` | per-trace-only ref observation | `RxrpcCall::see` |
| `rxrpc_cleanup_call()` | per-final cleanup dispatch | `RxrpcCall::cleanup` |
| `rxrpc_destroy_call()` | per-work-queue final teardown | `RxrpcCall::destroy` |
| `rxrpc_rcu_free_call()` | per-RCU final free | `RxrpcCall::rcu_free` |
| `rxrpc_destroy_all_calls()` | per-netns rmmod drain | `RxrpcCall::destroy_all` |
| `rxrpc_cleanup_tx_buffers()` | per-tx_queue drain | `RxrpcCall::cleanup_tx` |
| `rxrpc_cleanup_rx_buffers()` | per-rx-queues purge | `RxrpcCall::cleanup_rx` |
| `rxrpc_start_call_timer()` | per-call timer arm | `RxrpcCall::start_timer` |
| `rxrpc_call_timer_expired()` | per-timer wakeup → poke | `RxrpcCall::timer_expired` |
| `rxrpc_poke_call()` | per-IO-thread enqueue | `RxrpcCall::poke` |
| `rxrpc_get_call_slot()` / `_put_call_slot()` | per-semaphore admission | `RxrpcCall::get_slot` / `put_slot` |
| `rxrpc_kernel_query_call_security()` | kAFS API: query security params | `RxrpcCall::kernel_query_security` |
| `RXRPC_CALL_KERNEL/_EXCLUSIVE/_UPGRADE/_RELEASED/_HAS_USERID/_EXPOSED/_DISCONNECTED/_CONN_CHALLENGING` | per-flag bits | `RxrpcCallFlag` |
| `RXRPC_MIN_CWND` (4) / `RXRPC_TX_MAX_WINDOW` (128) | per-cong constants | shared |
| `RXRPC_CALL_RTT_AVAIL_MASK` (0xf) | per-RTT-probe slot mask | shared |

## Compatibility contract

REQ-1: struct rxrpc_call (key fields):
- ref: refcount_t (init 1).
- debug_id: u32 — assigned at alloc, used for tracing.
- user_mutex: mutex serializing user (sendmsg/recvmsg) against background processing.
- timer: timer_list firing `rxrpc_call_timer_expired`.
- destroyer: work_struct → `rxrpc_destroy_call`.
- rcu: rcu_head.
- link / wait_link / accept_link / recvmsg_link / sock_link / attend_link / error_link: list_head linkages.
- sock_node: rb_node into `rx->calls` (key: user_call_ID).
- recvmsg_queue / rx_queue / rx_oos_queue: sk_buff_head Rx pipeline.
- waitq: wait queue for ack/term wait.
- notify_lock: spinlock guarding notify-on-recvmsg state.
- flags: bitset of RXRPC_CALL_* (RELEASED, HAS_USERID, EXPOSED, DISCONNECTED, KERNEL, UPGRADE, EXCLUSIVE, CONN_CHALLENGING).
- events: pending-event bitset (processed by IO thread).
- _state: enum rxrpc_call_state.
- completion: enum rxrpc_call_completion (SUCCEEDED / REMOTELY_ABORTED / LOCALLY_ABORTED / LOCAL_ERROR / NETWORK_ERROR).
- abort_code / error: per-completion abort details.
- socket: RCU-protected back-pointer to rxrpc_sock.
- conn / bundle / peer / local: refcount-held subsystem links.
- key: per-call security key.
- user_call_ID: opaque user-supplied ID (kAFS uses op*).
- call_id / cid: rx protocol IDs.
- dest_srx: sockaddr_rxrpc (peer + service_id).
- security_level / security_ix / security_enctype: security context.
- tx_queue / tx_pending / tx_total_len / tx_jumbo_max / tx_winsize: tx state.
- rx_winsize / ackr_window / ackr_wtop: rx state.
- cong_cwnd / cong_ssthresh / cong_tstamp / acks_latest_ts: congestion.
- delay_ack_at / rack_timo_at / ping_at / keepalive_at / expect_rx_by / expect_req_by / expect_term_by: ktime deadlines.
- next_rx_timo / next_req_timo / hard_timo: timeout config.
- rxnet: per-netns rxrpc_net.
- rtt_avail: RTT-probe slot mask (RXRPC_CALL_RTT_AVAIL_MASK).
- interruptibility: RXRPC_INTERRUPTIBLE / _PREINTERRUPTIBLE / _UNINTERRUPTIBLE.

REQ-2: rxrpc_alloc_call(rx, gfp, debug_id) → call ∨ NULL:
- call = `kmem_cache_zalloc(rxrpc_call_jar, gfp)`; on null: NULL.
- mutex_init(&user_mutex).
- if `rx->sk.sk_kern_sock`: `lockdep_set_class(&user_mutex, &rxrpc_call_user_mutex_lock_class_key)` (kAFS path: prevents false-positive with mm sem).
- `timer_setup(&timer, rxrpc_call_timer_expired, 0)`.
- INIT_WORK(&destroyer, rxrpc_destroy_call).
- INIT_LIST_HEAD on link, wait_link, accept_link, recvmsg_link, sock_link, attend_link.
- skb_queue_head_init on recvmsg_queue, rx_queue, rx_oos_queue.
- init_waitqueue_head(&waitq); spin_lock_init(&notify_lock).
- refcount_set(&ref, 1).
- debug_id = debug_id; tx_total_len = -1; tx_jumbo_max = 1.
- next_rx_timo = 20*HZ; next_req_timo = 1*HZ.
- ackr_window = 1; ackr_wtop = 1.
- delay_ack_at / rack_timo_at / ping_at / keepalive_at / expect_rx_by / expect_req_by / expect_term_by = KTIME_MAX.
- memset(&sock_node, 0xed, sizeof(sock_node)) — poison-fill for use-while-uninserted bug detection.
- rx_winsize = rxrpc_rx_window_size; tx_winsize = 16.
- cong_cwnd = RXRPC_MIN_CWND (4); cong_ssthresh = RXRPC_TX_MAX_WINDOW (128).
- rxrpc_call_init_rtt(call).
- rxnet = rxrpc_net(sock_net(&rx->sk)); rtt_avail = RXRPC_CALL_RTT_AVAIL_MASK.
- `atomic_inc(&rxnet->nr_calls)`.
- return call.

REQ-3: rxrpc_alloc_client_call(rx, cp, p, gfp, debug_id) → call ∨ ERR_PTR:
- call = rxrpc_alloc_call(rx, gfp, debug_id); on null: -ENOMEM.
- now = ktime_get_real(); acks_latest_ts = now; cong_tstamp = now.
- dest_srx = `cp->peer->srx`; dest_srx.srx_service = cp->service_id.
- interruptibility = p->interruptibility; tx_total_len = p->tx_total_len.
- key = key_get(cp->key).
- peer = rxrpc_get_peer(cp->peer, rxrpc_peer_get_call).
- local = rxrpc_get_local(cp->local, rxrpc_local_get_call).
- security_level = cp->security_level.
- if p->kernel: `__set_bit(RXRPC_CALL_KERNEL, &flags)`.
- if cp->upgrade: `__set_bit(RXRPC_CALL_UPGRADE, &flags)`.
- if cp->exclusive: `__set_bit(RXRPC_CALL_EXCLUSIVE, &flags)`.
- if p->timeouts.normal: next_rx_timo = `umin(p->timeouts.normal, 1)`.
- if p->timeouts.idle: next_req_timo = `umin(p->timeouts.idle, 1)`.
- if p->timeouts.hard: hard_timo = p->timeouts.hard.
- ret = `rxrpc_init_client_call_security(call)`.
- if ret < 0: `rxrpc_prefail_call(call, RXRPC_CALL_LOCAL_ERROR, ret)`; rxrpc_put_call; return ERR_PTR(ret).
- `rxrpc_set_call_state(call, RXRPC_CALL_CLIENT_AWAIT_CONN)`.
- return call.

REQ-4: rxrpc_get_call_slot(p, gfp) → semaphore ∨ NULL:
- limiter = `p->kernel ? &rxrpc_kernel_call_limiter : &rxrpc_call_limiter`.
- if `p->interruptibility == RXRPC_UNINTERRUPTIBLE`: `down(limiter)`; return limiter.
- return `down_interruptible(limiter) < 0 ? NULL : limiter`.

REQ-5: rxrpc_put_call_slot(call):
- limiter = `test_bit(RXRPC_CALL_KERNEL, &flags) ? &rxrpc_kernel_call_limiter : &rxrpc_call_limiter`.
- up(limiter).

REQ-6: rxrpc_new_client_call(rx, cp, p, gfp, debug_id) → call:
- /* __releases(&rx->sk.sk_lock), __acquires(&call->user_mutex) */
- if WARN_ON_ONCE(!cp->peer): release_sock; return ERR_PTR(-EIO).
- limiter = rxrpc_get_call_slot(p, gfp); if !limiter: release_sock; return ERR_PTR(-ERESTARTSYS).
- call = rxrpc_alloc_client_call(rx, cp, p, gfp, debug_id); if IS_ERR: release_sock; up(limiter); return.
- mutex_lock(&call->user_mutex).
- write_lock(&rx->call_lock).
- /* Insert into rb_tree keyed by user_call_ID */
- pp = &rx->calls.rb_node; parent = NULL.
- while *pp:
  - parent = *pp; xcall = rb_entry(parent, sock_node).
  - if user_call_ID < xcall->user_call_ID: pp = left.
  - else if user_call_ID > xcall->user_call_ID: pp = right.
  - else: goto error_dup_user_ID.
- rcu_assign_pointer(call->socket, rx).
- call->user_call_ID = p->user_call_ID.
- `__set_bit(RXRPC_CALL_HAS_USERID, &flags)`.
- rxrpc_get_call(call, rxrpc_call_get_userid).
- rb_link_node + rb_insert_color into rx->calls.
- list_add(&sock_link, &rx->sock_calls).
- write_unlock(&rx->call_lock).
- /* Attach to per-netns calls list under rxnet->call_lock */
- spin_lock(&rxnet->call_lock); list_add_tail_rcu(&call->link, &rxnet->calls); spin_unlock.
- release_sock(&rx->sk).
- ret = rxrpc_connect_call(call, gfp).
- if ret < 0: goto error_attached_to_socket.
- return call.
- /* error_dup_user_ID */
- write_unlock; release_sock; rxrpc_prefail_call(call, RXRPC_CALL_LOCAL_ERROR, -EEXIST); mutex_unlock; rxrpc_put_call; return ERR_PTR(-EEXIST).
- /* error_attached_to_socket — leave call attached; recvmsg will surface error */
- rxrpc_set_call_completion(call, RXRPC_CALL_LOCAL_ERROR, 0, ret).
- return call (with error stamped, caller proceeds; recvmsg observes failure).

REQ-7: rxrpc_connect_call(call, gfp) → 0 ∨ -err:
- ret = rxrpc_look_up_bundle(call, gfp); on err: __set_bit(RXRPC_CALL_DISCONNECTED, &flags); return ret.
- rxrpc_get_call(call, rxrpc_call_get_io_thread).
- spin_lock_irq(&local->client_call_lock); list_add_tail(&call->wait_link, &local->new_client_calls); spin_unlock_irq.
- rxrpc_wake_up_io_thread(local).
- return 0.

REQ-8: rxrpc_incoming_call(rx, call, skb):
- /* called with interrupts disabled; cannot fail */
- conn = call->conn; sp = rxrpc_skb(skb).
- rcu_assign_pointer(call->socket, rx).
- call_id = sp->hdr.callNumber; dest_srx.srx_service = sp->hdr.serviceId.
- cid = sp->hdr.cid; cong_tstamp = skb->tstamp.
- `__set_bit(RXRPC_CALL_EXPOSED, &flags)`.
- rxrpc_set_call_state(call, RXRPC_CALL_SERVER_RECV_REQUEST).
- spin_lock(&conn->state_lock).
- switch conn->state:
  - SERVICE_UNSECURED / SERVICE_CHALLENGING: __set_bit(RXRPC_CALL_CONN_CHALLENGING, &flags).
  - SERVICE: nop.
  - ABORTED: rxrpc_set_call_completion(call, conn->completion, conn->abort_code, conn->error).
  - default: BUG().
- rxrpc_get_call(call, rxrpc_call_get_io_thread).
- chan = sp->hdr.cid & RXRPC_CHANNELMASK.
- conn->channels[chan].call_counter = call_id; .call_id = call_id; .call = call.
- spin_unlock(&conn->state_lock).
- spin_lock(&conn->peer->lock); hlist_add_head(&call->error_link, &conn->peer->error_targets); spin_unlock.
- rxrpc_start_call_timer(call).

REQ-9: rxrpc_find_call_by_user_ID(rx, user_call_ID) → call ∨ NULL:
- read_lock(&rx->call_lock).
- Walk rx->calls.rb_node:
  - Compare user_call_ID to current rb_entry sock_node.
  - On match: rxrpc_get_call(call, rxrpc_call_get_sendmsg); read_unlock; return call.
- read_unlock; return NULL.

REQ-10: rxrpc_release_call(rx, call):
- if `test_and_set_bit(RXRPC_CALL_RELEASED, &flags)`: BUG (double release).
- rxrpc_put_call_slot(call).
- /* Memory-barrier via spin+lock-unlock against recvmsg */
- spin_lock_irq(&rx->recvmsg_lock); spin_unlock_irq.
- write_lock(&rx->call_lock).
- if `test_and_clear_bit(RXRPC_CALL_HAS_USERID, &flags)`:
  - rb_erase(&sock_node, &rx->calls).
  - memset(&sock_node, 0xdd, sizeof(sock_node)).
  - putu = true.
- list_del(&sock_link).
- write_unlock.
- if putu: rxrpc_put_call(call, rxrpc_call_put_userid).

REQ-11: rxrpc_release_calls_on_socket(rx) — sock-close drain:
- while !list_empty(&rx->to_be_accepted):
  - call = list_entry first; list_del(&accept_link).
  - rxrpc_propose_abort(call, RX_CALL_DEAD, -ECONNRESET, rxrpc_abort_call_sock_release_tba).
  - rxrpc_put_call(call, rxrpc_call_put_release_sock_tba).
- while !list_empty(&rx->sock_calls):
  - call = first; rxrpc_get_call(call, rxrpc_call_get_release_sock).
  - rxrpc_propose_abort(call, RX_CALL_DEAD, -ECONNRESET, rxrpc_abort_call_sock_release).
  - rxrpc_release_call(rx, call).
  - rxrpc_put_call(call, rxrpc_call_put_release_sock).
- while call = first_entry_or_null(&rx->recvmsg_q):
  - list_del_init(&recvmsg_link); rxrpc_put_call(call, rxrpc_call_put_release_recvmsg_q).

REQ-12: rxrpc_get_call / rxrpc_try_get_call / rxrpc_put_call:
- get: `__refcount_inc(&call->ref, &r)`; trace.
- try_get: `__refcount_inc_not_zero(&call->ref, &r)`; on false: return NULL; trace.
- put: `__refcount_dec_and_test(&call->ref, &r)`; trace.
- if put-zero (dead):
  - ASSERTCMP(__rxrpc_call_state(call), ==, RXRPC_CALL_COMPLETE).
  - spin_lock(&rxnet->call_lock); list_del_rcu(&call->link); spin_unlock.
  - rxrpc_cleanup_call(call).

REQ-13: rxrpc_cleanup_call(call):
- memset(&sock_node, 0xcd, sizeof(sock_node)) — poison; detect post-cleanup tree-touch.
- ASSERTCMP(__rxrpc_call_state(call), ==, RXRPC_CALL_COMPLETE).
- ASSERT(test_bit(RXRPC_CALL_RELEASED, &call->flags)).
- timer_delete(&timer).
- if rcu_read_lock_held(): schedule_work(&destroyer) — defer to wq context.
- else: rxrpc_destroy_call(&destroyer).

REQ-14: rxrpc_destroy_call(work):
- timer_delete_sync(&timer).
- rxrpc_cleanup_tx_buffers — for each tq in tx_queue: kfree each rxrpc_txbuf via rxrpc_put_txbuf; kfree(tq).
- rxrpc_cleanup_rx_buffers — purge recvmsg_queue, rx_queue, rx_oos_queue.
- rxrpc_put_txbuf(tx_pending, rxrpc_txbuf_put_cleaned).
- rxrpc_put_connection(call->conn, rxrpc_conn_put_call).
- rxrpc_deactivate_bundle(call->bundle).
- rxrpc_put_bundle(call->bundle, rxrpc_bundle_put_call).
- rxrpc_put_peer(call->peer, rxrpc_peer_put_call).
- rxrpc_put_local(call->local, rxrpc_local_put_call).
- key_put(call->key).
- call_rcu(&call->rcu, rxrpc_rcu_free_call).

REQ-15: rxrpc_rcu_free_call(rcu):
- call = container_of(rcu, struct rxrpc_call, rcu).
- rxnet = READ_ONCE(call->rxnet).
- kmem_cache_free(rxrpc_call_jar, call).
- if `atomic_dec_and_test(&rxnet->nr_calls)`: wake_up_var(&rxnet->nr_calls).

REQ-16: rxrpc_destroy_all_calls(rxnet) — module exit:
- if !list_empty(&rxnet->calls):
  - for call in &rxnet->calls (up to 10): rxrpc_see_call(call, rxrpc_call_see_still_live); pr_err "Call %p still in use".
- atomic_dec(&rxnet->nr_calls).
- wait_var_event(&rxnet->nr_calls, !atomic_read(&rxnet->nr_calls)).

REQ-17: rxrpc_start_call_timer(call):
- if hard_timo: delay = ms_to_ktime(hard_timo * 1000); expect_term_by = ktime_add(ktime_get_real(), delay); trace.
- timer.expires = jiffies — fire ASAP for first pass.

REQ-18: rxrpc_poke_call(call, what):
- if !test_bit(RXRPC_CALL_DISCONNECTED, &flags):
  - spin_lock_irq(&local->lock).
  - busy = !list_empty(&call->attend_link).
  - if !busy ∧ !rxrpc_try_get_call(call, rxrpc_call_get_poke): busy = true.
  - if !busy: list_add_tail(&call->attend_link, &local->call_attend_q).
  - spin_unlock_irq.
  - if !busy: rxrpc_wake_up_io_thread(local).

REQ-19: Per-state machine (legal transitions, client side):
- UNINITIALISED → CLIENT_AWAIT_CONN (alloc_client_call).
- CLIENT_AWAIT_CONN → CLIENT_SEND_REQUEST (connection acquired).
- CLIENT_SEND_REQUEST → CLIENT_AWAIT_REPLY (final tx fragment).
- CLIENT_AWAIT_REPLY → CLIENT_RECV_REPLY (first rx fragment).
- CLIENT_RECV_REPLY → COMPLETE (last rx fragment + ACK).
- ANY → COMPLETE (abort / error path).

REQ-20: Per-state machine (server side):
- UNINITIALISED → SERVER_PREALLOC (preallocated) → SERVER_RECV_REQUEST (incoming_call) → SERVER_ACK_REQUEST → SERVER_SEND_REPLY → SERVER_AWAIT_ACK → COMPLETE.

REQ-21: Per-completion classification:
- SUCCEEDED — normal end.
- REMOTELY_ABORTED — peer sent ABORT.
- LOCALLY_ABORTED — we sent ABORT.
- LOCAL_ERROR — local resource/setup failure.
- NETWORK_ERROR — ICMP / socket-error path.

## Acceptance Criteria

- [ ] AC-1: rxrpc_alloc_call: ref==1; tx_total_len=-1; cong_cwnd==RXRPC_MIN_CWND; rxnet->nr_calls++.
- [ ] AC-2: rxrpc_alloc_client_call: state==CLIENT_AWAIT_CONN; peer/local refs held; key_get applied.
- [ ] AC-3: rxrpc_alloc_client_call: cp->kernel=true → RXRPC_CALL_KERNEL bit set; slot taken from rxrpc_kernel_call_limiter.
- [ ] AC-4: rxrpc_alloc_client_call: cp->exclusive=true → RXRPC_CALL_EXCLUSIVE bit set (no channel sharing).
- [ ] AC-5: rxrpc_new_client_call: duplicate user_call_ID returns -EEXIST; call prefailed with RXRPC_CALL_LOCAL_ERROR.
- [ ] AC-6: rxrpc_new_client_call: rb_tree insertion correct under write_lock; sock_link added.
- [ ] AC-7: rxrpc_connect_call: bundle lookup failure sets RXRPC_CALL_DISCONNECTED.
- [ ] AC-8: rxrpc_incoming_call from conn ABORTED: sets call completion to conn->completion/abort_code/error.
- [ ] AC-9: rxrpc_release_call: double release BUG; HAS_USERID cleared and ref dropped on rb_erase.
- [ ] AC-10: rxrpc_put_call dropping last ref: requires state==COMPLETE (ASSERTCMP); list_del_rcu from rxnet->calls; schedule cleanup.
- [ ] AC-11: rxrpc_destroy_call: all subsystem refs (conn, bundle, peer, local, key) released; call_rcu for slab free.
- [ ] AC-12: rxrpc_call_limiter / _kernel_call_limiter bound at 1000 each; concurrent allocators block.
- [ ] AC-13: rxrpc_destroy_all_calls: waits for rxnet->nr_calls == 0 via wait_var_event.
- [ ] AC-14: rxrpc_poke_call: idempotent re-add prevented by attend_link non-empty check; gets ref before enqueue.
- [ ] AC-15: rxrpc_find_call_by_user_ID: rb_tree O(log n) lookup; ref taken on hit.

## Architecture

```
struct RxrpcCall {
  ref: AtomicRefcount,
  debug_id: u32,
  user_mutex: Mutex<()>,
  timer: TimerList,
  destroyer: WorkStruct,
  rcu: RcuHead,
  link: ListHead,              // rxnet->calls
  wait_link: ListHead,         // local->new_client_calls
  accept_link: ListHead,       // rx->to_be_accepted
  recvmsg_link: ListHead,      // rx->recvmsg_q
  sock_link: ListHead,         // rx->sock_calls
  attend_link: ListHead,       // local->call_attend_q
  error_link: HlistNode,       // peer->error_targets
  sock_node: RbNode,           // rx->calls (key: user_call_ID)
  recvmsg_queue: SkbQueueHead,
  rx_queue: SkbQueueHead,
  rx_oos_queue: SkbQueueHead,
  waitq: WaitQueue,
  notify_lock: SpinLock<()>,
  flags: AtomicUsize,          // RXRPC_CALL_*
  events: AtomicUsize,
  _state: AtomicI8,            // enum RxrpcCallState
  completion: RxrpcCallCompletion,
  abort_code: u32,
  error: i32,
  socket: RcuPtr<RxrpcSock>,
  conn: *RxrpcConnection,
  bundle: *RxrpcBundle,
  peer: *RxrpcPeer,
  local: *RxrpcLocal,
  key: *Key,
  user_call_ID: usize,
  call_id: u32,
  cid: u32,
  dest_srx: SockaddrRxrpc,
  security_level: u8,
  security_ix: u8,
  security_enctype: u32,
  tx_queue: *RxrpcTxqueue,
  tx_pending: *RxrpcTxbuf,
  tx_total_len: i64,
  tx_jumbo_max: u8,
  tx_winsize: u16,
  rx_winsize: u16,
  ackr_window: u16,
  ackr_wtop: u16,
  cong_cwnd: u16,              // init = RXRPC_MIN_CWND
  cong_ssthresh: u16,          // init = RXRPC_TX_MAX_WINDOW
  cong_tstamp: KTime,
  acks_latest_ts: KTime,
  delay_ack_at: KTime,         // init = KTIME_MAX
  rack_timo_at: KTime,         // init = KTIME_MAX
  ping_at: KTime,              // init = KTIME_MAX
  keepalive_at: KTime,         // init = KTIME_MAX
  expect_rx_by: KTime,         // init = KTIME_MAX
  expect_req_by: KTime,        // init = KTIME_MAX
  expect_term_by: KTime,       // init = KTIME_MAX
  next_rx_timo: u32,           // init = 20 * HZ
  next_req_timo: u32,          // init = 1 * HZ
  hard_timo: u32,
  rxnet: *RxrpcNet,
  rtt_avail: u8,               // RXRPC_CALL_RTT_AVAIL_MASK
  interruptibility: RxrpcInterruptibility,
}

enum RxrpcCallState {
  Uninitialised,
  ClientAwaitConn,
  ClientSendRequest,
  ClientAwaitReply,
  ClientRecvReply,
  ServerPrealloc,
  ServerRecvRequest,
  ServerAckRequest,
  ServerSendReply,
  ServerAwaitAck,
  Complete,
}

enum RxrpcCallCompletion {
  Succeeded,
  RemotelyAborted,
  LocallyAborted,
  LocalError,
  NetworkError,
}
```

`RxrpcCall::alloc(rx, gfp, debug_id) -> Option<*RxrpcCall>`:
1. call = kmem_cache_zalloc(RXRPC_CALL_JAR, gfp)?.
2. mutex_init(&call.user_mutex).
3. if rx.sk.sk_kern_sock: lockdep_set_class(&call.user_mutex, &RXRPC_CALL_USER_MUTEX_KEY).
4. timer_setup(&call.timer, RxrpcCall::timer_expired, 0).
5. INIT_WORK(&call.destroyer, RxrpcCall::destroy).
6. INIT_LIST_HEAD on (link, wait_link, accept_link, recvmsg_link, sock_link, attend_link).
7. skb_queue_head_init on (recvmsg_queue, rx_queue, rx_oos_queue).
8. init_waitqueue_head(&call.waitq); spin_lock_init(&call.notify_lock).
9. refcount_set(&call.ref, 1).
10. /* Scalars */
11. call.debug_id = debug_id; call.tx_total_len = -1; call.tx_jumbo_max = 1.
12. call.next_rx_timo = 20*HZ; call.next_req_timo = 1*HZ.
13. call.ackr_window = 1; call.ackr_wtop = 1.
14. call.delay_ack_at = call.rack_timo_at = call.ping_at = call.keepalive_at
   = call.expect_rx_by = call.expect_req_by = call.expect_term_by = KTIME_MAX.
15. memset(&call.sock_node, 0xed, sizeof(call.sock_node))  /* poison */
16. call.rx_winsize = rxrpc_rx_window_size; call.tx_winsize = 16.
17. call.cong_cwnd = RXRPC_MIN_CWND; call.cong_ssthresh = RXRPC_TX_MAX_WINDOW.
18. rxrpc_call_init_rtt(call).
19. call.rxnet = rxrpc_net(sock_net(&rx.sk)).
20. call.rtt_avail = RXRPC_CALL_RTT_AVAIL_MASK.
21. atomic_inc(&call.rxnet.nr_calls).
22. Some(call).

`RxrpcCall::new_client(rx, cp, p, gfp, debug_id) -> *RxrpcCall`:
1. if WARN_ON_ONCE(cp.peer.is_null()): release_sock(&rx.sk); return ERR_PTR(-EIO).
2. limiter = RxrpcCall::get_slot(p, gfp).
3. if limiter.is_none(): release_sock; return ERR_PTR(-ERESTARTSYS).
4. call = RxrpcCall::alloc_client(rx, cp, p, gfp, debug_id).
5. if IS_ERR(call): release_sock; up(limiter); return call.
6. mutex_lock(&call.user_mutex).
7. write_lock(&rx.call_lock).
8. /* rb_tree insert keyed by user_call_ID */
9. Walk rx.calls; if dup: error_dup_user_ID.
10. rcu_assign_pointer(call.socket, rx).
11. call.user_call_ID = p.user_call_ID.
12. __set_bit(RXRPC_CALL_HAS_USERID, &call.flags).
13. RxrpcCall::get(call, get_userid).
14. rb_link_node + rb_insert_color into rx.calls.
15. list_add(&call.sock_link, &rx.sock_calls).
16. write_unlock.
17. spin_lock(&call.rxnet.call_lock); list_add_tail_rcu(&call.link, &call.rxnet.calls); spin_unlock.
18. release_sock(&rx.sk).
19. ret = RxrpcCall::connect(call, gfp).
20. if ret < 0: goto error_attached_to_socket.
21. return call.
22. /* error_dup_user_ID */
23. write_unlock; release_sock; rxrpc_prefail_call(call, RxrpcCallCompletion::LocalError, -EEXIST).
24. mutex_unlock; RxrpcCall::put(call, put_userid_exists); return ERR_PTR(-EEXIST).
25. /* error_attached_to_socket */
26. rxrpc_set_call_completion(call, RxrpcCallCompletion::LocalError, 0, ret); return call.

`RxrpcCall::put(call, why)`:
1. dead = __refcount_dec_and_test(&call.ref, &r); trace.
2. if dead:
   - ASSERTCMP(__rxrpc_call_state(call), ==, RxrpcCallState::Complete).
   - spin_lock(&call.rxnet.call_lock); list_del_rcu(&call.link); spin_unlock.
   - RxrpcCall::cleanup(call).

`RxrpcCall::cleanup(call)`:
1. memset(&call.sock_node, 0xcd, sizeof(call.sock_node))  /* poison */.
2. ASSERTCMP(state, ==, Complete); ASSERT(RXRPC_CALL_RELEASED set).
3. timer_delete(&call.timer).
4. if rcu_read_lock_held(): schedule_work(&call.destroyer).
5. else: RxrpcCall::destroy(&call.destroyer).

`RxrpcCall::destroy(work)`:
1. timer_delete_sync(&call.timer).
2. RxrpcCall::cleanup_tx(call) — free tx_queue + per-buf rxrpc_put_txbuf.
3. RxrpcCall::cleanup_rx(call) — purge recvmsg_queue, rx_queue, rx_oos_queue.
4. rxrpc_put_txbuf(call.tx_pending, rxrpc_txbuf_put_cleaned).
5. rxrpc_put_connection(call.conn, rxrpc_conn_put_call).
6. rxrpc_deactivate_bundle(call.bundle).
7. rxrpc_put_bundle(call.bundle, rxrpc_bundle_put_call).
8. rxrpc_put_peer(call.peer, rxrpc_peer_put_call).
9. rxrpc_put_local(call.local, rxrpc_local_put_call).
10. key_put(call.key).
11. call_rcu(&call.rcu, RxrpcCall::rcu_free).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `refcount_init_one_post_alloc` | INVARIANT | per-alloc_call: refcount_set(&ref, 1). |
| `state_machine_legal_transitions` | INVARIANT | per-set_call_state: only enumerated transitions. |
| `release_one_shot` | INVARIANT | per-release_call: test_and_set RXRPC_CALL_RELEASED — second release BUG. |
| `put_last_requires_complete` | INVARIANT | per-put_call (dead): __rxrpc_call_state == COMPLETE. |
| `userid_ref_paired` | INVARIANT | per-new_client_call: get_call(userid) paired with rb_erase + put_call(userid). |
| `slot_admit_pair` | INVARIANT | per-get_call_slot / put_call_slot: matched up/down on right limiter (kernel vs user). |
| `rb_tree_locked` | INVARIANT | per-rb_insert/erase: under write_lock(&rx->call_lock). |
| `find_under_read_lock` | INVARIANT | per-find_call_by_user_ID: read_lock held during walk; ref taken before unlock. |
| `socket_publish_via_rcu_assign` | INVARIANT | per-new_client_call / incoming_call: socket pointer set via rcu_assign_pointer. |
| `destroy_call_releases_all_subsystem_refs` | INVARIANT | per-destroy_call: conn+bundle+peer+local+key all _put before call_rcu. |
| `poke_get_ref_before_enqueue` | INVARIANT | per-poke_call: rxrpc_try_get_call before list_add_tail. |
| `disconnected_skip_poke` | INVARIANT | per-poke_call: RXRPC_CALL_DISCONNECTED ⟹ no-op. |

### Layer 2: TLA+

`net/rxrpc/call-object.tla`:
- States × {Uninit, ClAwtConn, ClSndReq, ClAwtRpl, ClRcvRpl, SvPrealc, SvRcvReq, SvAckReq, SvSndRpl, SvAwtACK, Complete}.
- Refcount lifecycle: alloc(=1) ↔ get/put deltas ↔ dead(=0).
- Properties:
  - `safety_state_machine_legal` — per-call_state: only client-side chain or server-side chain or any→Complete.
  - `safety_refcount_nonneg` — per-call: ref >= 0 at all times.
  - `safety_dead_iff_complete` — per-put (dead): state == Complete.
  - `safety_userid_uniqueness` — per-(rx, user_call_ID): at most one call with RXRPC_CALL_HAS_USERID set in rx->calls.
  - `safety_slot_balance` — per-call: get_call_slot paired with exactly one put_call_slot.
  - `safety_released_one_shot` — per-call: RXRPC_CALL_RELEASED bit set at most once.
  - `liveness_destroy_all_drains` — per-rxnet exit: eventually nr_calls == 0.
  - `liveness_complete_eventually_freed` — per-call: state==Complete + ref==0 ⟹ eventually kmem_cache_free.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `RxrpcCall::alloc` post: ref==1, tx_total_len==-1, cong_cwnd==RXRPC_MIN_CWND, rxnet.nr_calls++ | `RxrpcCall::alloc` |
| `RxrpcCall::alloc_client` post: state==ClientAwaitConn ∧ peer/local refs held | `RxrpcCall::alloc_client` |
| `RxrpcCall::new_client` post (success): in rx.calls rb_tree ∧ in rxnet.calls list ∧ in rx.sock_calls | `RxrpcCall::new_client` |
| `RxrpcCall::new_client` post (dup): -EEXIST ∧ call prefailed with LocalError | `RxrpcCall::new_client` |
| `RxrpcCall::release` post: RELEASED bit set; HAS_USERID cleared if present; one ref dropped | `RxrpcCall::release` |
| `RxrpcCall::put` post (dead): list_del_rcu from rxnet.calls; cleanup scheduled | `RxrpcCall::put` |
| `RxrpcCall::destroy` post: all subsystem refs released; call_rcu queued | `RxrpcCall::destroy` |
| `RxrpcCall::find_by_user_id` post: ref incremented before read_unlock | `RxrpcCall::find_by_user_id` |
| `RxrpcCall::incoming` post: state==ServerRecvRequest; EXPOSED set; timer started | `RxrpcCall::incoming` |
| `RxrpcCall::poke` post: at most one attend_link enqueue per call | `RxrpcCall::poke` |
| `RxrpcCall::get_slot` post: limiter held ∨ -ERESTARTSYS (interruptible) | `RxrpcCall::get_slot` |

### Layer 4: Verus/Creusot functional

`Per-call lifecycle (ALLOC → AWAIT_CONN → SEND → AWAIT_REPLY → RECV → COMPLETE → RELEASE → put-last → destroy → rcu_free) ↔ struct rxrpc_call FSM` semantic equivalence: per-Rx-protocol-specification (Transarc/AFS-3) and per-kAFS in-tree consumer expectations. Refcount discipline: one ref per (allocation, userid attachment, IO-thread enqueue, sendmsg lookup, poke); each paired with a single put. Subsystem ownership: conn / bundle / peer / local / key strict get-on-alloc, put-on-destroy.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

RxRPC call-object reinforcement:

- **Per-RXRPC_CALL_RELEASED test_and_set BUG** — defense against per-double-release UAF.
- **Per-state-machine ASSERTCMP COMPLETE before put-last** — defense against per-early-free of live call.
- **Per-user_call_ID rb_tree under write_lock** — defense against per-rb_tree concurrent mutation.
- **Per-socket pointer via rcu_assign_pointer** — defense against per-publish-before-init race.
- **Per-call_limiter / kernel_call_limiter slot semaphore (1000 each)** — defense against per-fork-bomb call exhaustion.
- **Per-kernel-vs-user limiter separation** — defense against per-kAFS starvation under userspace flood.
- **Per-sock_node poison-fill (0xed pre-insert, 0xdd post-erase, 0xcd post-cleanup)** — defense against per-stale-rb-pointer use.
- **Per-mutex_init lockdep class for kAFS** — defense against per-AFS-mmap-sem false-positive deadlock report.
- **Per-call_rcu free** — defense against per-RCU-reader UAF on call pointer.
- **Per-rxrpc_destroy_all_calls wait_var_event** — defense against per-rmmod-with-live-call slab UAF.
- **Per-subsystem refs strict get/put on destroy** — defense against per-conn/bundle/peer/local/key leak.
- **Per-RXRPC_CALL_DISCONNECTED gate on poke** — defense against per-poke-after-disconnect IO-thread spam.
- **Per-EXCLUSIVE flag enforces once-only conn** — defense against per-channel-sharing for security-sensitive calls.
- **Per-error_attached_to_socket path preserves call** — defense against per-leaked-userid (recvmsg surfaces error cleanly).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/rxrpc/conn_client.c bundle / channel management (covered separately if expanded)
- net/rxrpc/conn_object.c connection lifecycle (covered separately)
- net/rxrpc/local_object.c local-endpoint + IO thread (covered separately)
- net/rxrpc/peer_object.c peer + RTT (covered separately)
- net/rxrpc/call_state.c set_call_state / set_call_completion machinery (covered separately)
- net/rxrpc/sendmsg.c / recvmsg.c data-path (covered separately)
- net/rxrpc/input.c packet receive (covered separately)
- net/rxrpc/output.c packet transmit (covered separately)
- net/rxrpc/security.c / rxkad.c / rxgk.c (covered separately)
- net/rxrpc/proc.c / sysctl.c diagnostics (covered separately)
- Implementation code
