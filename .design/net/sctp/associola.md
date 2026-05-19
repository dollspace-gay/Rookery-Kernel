# Tier-3: net/sctp/associola.c — SCTP association management

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sctp/00-overview.md
upstream-paths:
  - net/sctp/associola.c (~1711 lines)
  - include/net/sctp/structs.h (struct sctp_association, struct sctp_transport)
  - include/net/sctp/constants.h (SCTP_STATE_*, SCTP_TRANSPORT_*, SCTP_EVENT_TIMEOUT_*)
  - net/sctp/sm_statefuns.c (state-function table consumed via sctp_do_sm)
  - net/sctp/sm_sideeffect.c (sctp_do_sm dispatcher)
  - net/sctp/endpointola.c (sctp_endpoint_add_asoc / hold / put)
  - net/sctp/transport.c (struct sctp_transport lifecycle)
  - net/sctp/outqueue.c (per-asoc outqueue / retransmit)
  - net/sctp/input.c (sctp_inq dispatch into asoc base inqueue)
  - net/sctp/auth.c (SCTP-AUTH shared-keys, AUTH_RANDOM)
  - net/sctp/stream.c (per-stream state)
-->

## Summary

An **SCTP association** is the long-lived per-peer state encapsulating one logical connection between two SCTP endpoints. It is a separate concept from the **endpoint** (one per `struct sock`, owning the bind address list) and the **transport** (one per remote IP address, owning RTO/cwnd/PMTU). Per-association: `struct sctp_association` (>2 KB on a typical kernel) is allocated via `sctp_association_new(ep, sk, scope, gfp)` which composes `sctp_association_init` (state = `SCTP_STATE_CLOSED`, my_vtag/initial_tsn generated, per-timeout init, output/input queues set up, stream init, AUTH random sampled). Per-state lifecycle: CLOSED → COOKIE_WAIT → COOKIE_ECHOED → ESTABLISHED → (SHUTDOWN_{PENDING,SENT,RECEIVED,ACK_SENT}) → CLOSED, all transitions driven by `sctp_do_sm(net, event_type, subtype, state, ep, asoc, arg, gfp)` against the state-function table in `sm_statefuns.c`. Per-peer multipath: `asoc->peer.transport_addr_list` is a list of `sctp_transport` objects, one per remote IP, each with its own RTO/cwnd/ssthresh/state; the **active_path** carries new transmissions and the **retran_path** carries retransmissions; failed transports are evicted from the active pool via `sctp_assoc_control_transport` and heartbeats track liveness per transport. Per-retransmission: outgoing data lives in `asoc->outqueue` (per-`sctp_outq`), and each transport keeps a `transmitted` list scanned by `sctp_assoc_lookup_tsn` to locate the transport a given TSN was sent on. Critical for: SCTP multihoming, mobility, lossy-path resilience, and signaling-protocol stacks (Diameter, M3UA, SIGTRAN).

This Tier-3 covers `net/sctp/associola.c` (~1711 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sctp_association` | per-association state | `SctpAssociation` |
| `struct sctp_transport` | per-peer-IP path | `SctpTransport` |
| `sctp_association_new()` | per-alloc + init | `SctpAssoc::new` |
| `sctp_association_init()` | per-zero-init | `SctpAssoc::init` |
| `sctp_association_free()` | per-graceful-teardown | `SctpAssoc::free` |
| `sctp_association_destroy()` | per-final-free | `SctpAssoc::destroy` |
| `sctp_association_hold()` | per-refcount inc | `SctpAssoc::hold` |
| `sctp_association_put()` | per-refcount dec + free | `SctpAssoc::put` |
| `sctp_association_get_next_tsn()` | per-TSN allocator | `SctpAssoc::get_next_tsn` |
| `sctp_assoc_set_primary()` | per-set primary path | `SctpAssoc::set_primary` |
| `sctp_assoc_rm_peer()` | per-remove transport | `SctpAssoc::rm_peer` |
| `sctp_assoc_add_peer()` | per-add transport | `SctpAssoc::add_peer` |
| `sctp_assoc_lookup_paddr()` | per-find transport by addr | `SctpAssoc::lookup_paddr` |
| `sctp_assoc_del_nonprimary_peers()` | per-shrink to primary | `SctpAssoc::del_nonprimary_peers` |
| `sctp_assoc_control_transport()` | per-state cmd UP/DOWN/HBSUCC/HBFAIL/PF/CONFIRM | `SctpAssoc::control_transport` |
| `sctp_assoc_lookup_tsn()` | per-find transport carrying a TSN | `SctpAssoc::lookup_tsn` |
| `sctp_assoc_update_retran_path()` | per-rotate retran transport | `SctpAssoc::update_retran_path` |
| `sctp_assoc_choose_alter_transport()` | per-choose alternate | `SctpAssoc::choose_alter_transport` |
| `sctp_assoc_update_frag_point()` | per-pmtu/userfrag recompute | `SctpAssoc::update_frag_point` |
| `sctp_assoc_set_pmtu()` | per-set asoc PMTU | `SctpAssoc::set_pmtu` |
| `sctp_assoc_sync_pmtu()` | per-aggregate transport PMTUs | `SctpAssoc::sync_pmtu` |
| `sctp_assoc_rwnd_increase()` | per-rcv-window grow + SACK | `SctpAssoc::rwnd_increase` |
| `sctp_assoc_rwnd_decrease()` | per-rcv-window shrink | `SctpAssoc::rwnd_decrease` |
| `sctp_assoc_set_bind_addr_from_ep()` | per-bind-addr from endpoint | `SctpAssoc::bind_addr_from_ep` |
| `sctp_assoc_set_bind_addr_from_cookie()` | per-bind-addr from cookie | `SctpAssoc::bind_addr_from_cookie` |
| `sctp_assoc_lookup_laddr()` | per-find local addr | `SctpAssoc::lookup_laddr` |
| `sctp_assoc_set_id()` | per-allocate assoc id | `SctpAssoc::set_id` |
| `sctp_assoc_bh_rcv()` | per-input dispatch loop (workqueue) | `SctpAssoc::bh_rcv` |
| `sctp_assoc_migrate()` | per-accept move to new sock | `SctpAssoc::migrate` |
| `sctp_assoc_update()` | per-restart / cookie-collision update | `SctpAssoc::update` |
| `sctp_assoc_clean_asconf_ack_cache()` | per-trim ASCONF-ACK cache | `SctpAssoc::clean_asconf_ack_cache` |
| `sctp_assoc_lookup_asconf_ack()` | per-find cached ASCONF-ACK | `SctpAssoc::lookup_asconf_ack` |
| `sctp_asconf_queue_teardown()` | per-flush ASCONF queues | `SctpAssoc::asconf_queue_teardown` |
| `sctp_get_ecne_prepend()` | per-prepend ECNE chunk | `SctpAssoc::get_ecne_prepend` |
| `sctp_cmp_addr_exact()` | per-address-equality helper | `SctpAssoc::cmp_addr_exact` |
| `sctp_select_active_and_retran_path()` | per-elect active/retran | `SctpAssoc::select_active_and_retran` |
| `sctp_trans_score()` | per-transport rank | `SctpAssoc::trans_score` |
| `sctp_trans_elect_best()` / `_tie()` | per-tiebreaker election | `SctpAssoc::trans_elect_best` / `tie` |
| `sctp_peer_needs_update()` | per-rwnd update gate | `SctpAssoc::peer_needs_update` |
| `sctp_endpoint_add_asoc()` (consumed) | per-attach to endpoint | shared from `endpointola` |
| `sctp_do_sm()` (consumed) | per-state-machine dispatch | shared from `sm_sideeffect` |
| `sctp_inq_init()` / `_set_th_handler` (consumed) | per-asoc input queue | shared from `input` |
| `sctp_outq_init()` / `_tail` / `_free` (consumed) | per-asoc outqueue | shared from `outqueue` |
| `sctp_transport_new()` / `_route` / `_pl_reset` (consumed) | per-transport lifecycle | shared from `transport` |
| `sctp_ulpq_init()` / `_flush` (consumed) | per-ULP reassembly | shared from `ulpqueue` |
| `sctp_stream_init()` / `_free` / `_clear` / `_update` (consumed) | per-stream state | shared from `stream` |
| `sctp_auth_asoc_copy_shkeys()` / `_init_active_key` (consumed) | per-auth shared keys | shared from `auth` |
| `sctp_tsnmap_init()` (consumed) | per-peer-TSN map | shared from `tsnmap` |
| `sctp_ulpevent_notify_peer_addr_change()` (consumed) | per-ULP notify | shared from `ulpevent` |

## Compatibility contract

REQ-1: struct sctp_association (per `include/net/sctp/structs.h`, only fields touched here):
- base: `sctp_ep_common` — type = SCTP_EP_TYPE_ASSOCIATION, refcnt, dead, bind_addr, inqueue.
- ep: `*sctp_endpoint` — backref + refcount.
- state: `sctp_state_t` — { CLOSED, COOKIE_WAIT, COOKIE_ECHOED, ESTABLISHED, SHUTDOWN_PENDING, SHUTDOWN_SENT, SHUTDOWN_RECEIVED, SHUTDOWN_ACK_SENT }.
- c: `sctp_cookie` — my_vtag, my_port, initial_tsn, peer_vtag, sinit_*, auth_random, auth_hmacs, auth_chunks.
- next_tsn: u32 — next TSN to assign.
- ctsn_ack_point: u32 — highest TSN cumulatively ACKed.
- adv_peer_ack_point: u32 — advanced peer-ack point (PR-SCTP).
- highest_sacked: u32 — highest TSN seen in a SACK.
- last_cwr_tsn: u32 — last CWR sent.
- rwnd, a_rwnd, rwnd_over, rwnd_press: u32 — local rcv-window state.
- peer.rwnd: u32 — peer-advertised rcv-window.
- peer.transport_addr_list: list_head — RCU-protected list of `sctp_transport`.
- peer.transport_count: u32.
- peer.primary_path / peer.active_path / peer.retran_path / peer.last_data_from: `*sctp_transport`.
- peer.peer_random / peer.peer_chunks / peer.peer_hmacs: SCTP-AUTH peer-side.
- peer.tsn_map: `sctp_tsnmap` — cumulative-TSN tracking.
- peer.sack_needed: bool; peer.sack_generation: u8.
- peer.auth_capable: bool.
- peer.ipv4_address / peer.ipv6_address: bool — peer-supported AFs.
- peer.i: `sctp_inithdr` saved from INIT/INIT-ACK.
- peer.port: u16.
- timers[SCTP_NUM_TIMEOUT_TYPES] / timeouts[…]: per-timer-type.
- max_retrans / pf_retrans / ps_retrans / pf_expose / pathmaxrxt: u16 — retransmit thresholds.
- rto_initial / rto_max / rto_min: u32 (jiffies) — RTO bounds.
- hbinterval / probe_interval: u32 (jiffies) — heartbeat / probing cadence.
- pathmtu / frag_point / user_frag: u32 — path MTU + fragmentation.
- max_burst: u32 — DATA burst cap (RFC 4960 § 6.1).
- max_init_attempts / max_init_timeo: u32 — INIT-retry policy.
- subscribe: sctp_event_subscribe — notification mask.
- assocparams.sasoc_*: cookie-life + thresholds.
- outqueue: `sctp_outq` — pending DATA + retransmit queue.
- ulpq: `sctp_ulpq` — reassembly + ordered delivery.
- stream: `sctp_stream` — per-stream state.
- asocs: list_head — ep.asocs linkage.
- addip_chunk_list / asconf_ack_list / addip_serial / strreset_outseq: list_head/u32 — ASCONF.
- active_key_id / strreset_enable / endpoint_shared_keys: u16/u16/list_head — SCTP-AUTH.
- assoc_id: sctp_assoc_t — IDR-assigned per-net.
- encap_port / flowlabel / dscp / sackdelay / sackfreq / param_flags / pathmtu / max_burst: per-asoc cfg.
- overall_error_count: u8 — association-wide error counter.
- unack_data: u32 — DATA chunks awaiting ACK.
- rmem_alloc: atomic_t — rmem charged to this asoc (when ep.rcvbuf_policy).
- wait: wait_queue_head.

REQ-2: sctp_association_init(asoc, ep, sk, scope, gfp):
- /* Base linkage */
- asoc.ep = ep (drop const); asoc.base.sk = sk; asoc.base.net = sock_net(sk).
- sctp_endpoint_hold(ep); sock_hold(sk).
- asoc.base.type = SCTP_EP_TYPE_ASSOCIATION; refcount_set(&base.refcnt, 1).
- sctp_bind_addr_init(&asoc.base.bind_addr, ep.base.bind_addr.port).
- /* Initial state + cookie life */
- asoc.state = SCTP_STATE_CLOSED.
- asoc.cookie_life = ms_to_ktime(sp.assocparams.sasoc_cookie_life).
- asoc.user_frag = sp.user_frag.
- asoc.max_retrans = sp.assocparams.sasoc_asocmaxrxt.
- asoc.pf_retrans = sp.pf_retrans; ps_retrans = sp.ps_retrans; pf_expose = sp.pf_expose.
- asoc.rto_initial / rto_max / rto_min ← msecs_to_jiffies(sp.rtoinfo.*).
- asoc.hbinterval ← msecs_to_jiffies(sp.hbinterval); probe_interval ← sp.probe_interval.
- asoc.encap_port = sp.encap_port; pathmaxrxt = sp.pathmaxrxt; flowlabel; dscp.
- asoc.sackdelay ← msecs_to_jiffies(sp.sackdelay); sackfreq.
- asoc.param_flags = sp.param_flags.
- asoc.max_burst = sp.max_burst.
- asoc.subscribe = sp.subscribe.
- /* Initial timeouts */
- timeouts[T1_COOKIE] = timeouts[T1_INIT] = timeouts[T2_SHUTDOWN] = rto_initial.
- timeouts[T5_SHUTDOWN_GUARD] = 5 * rto_max (sctpimpguide § 2.12.2).
- timeouts[SACK] = sackdelay; timeouts[AUTOCLOSE] = sp.autoclose * HZ.
- for i in [NONE, NUM_TIMEOUT_TYPES): timer_setup(&timers[i], sctp_timer_events[i], 0).
- /* Cookie + TSN seed */
- asoc.c.sinit_max_instreams = sp.initmsg.sinit_max_instreams.
- asoc.c.sinit_num_ostreams = sp.initmsg.sinit_num_ostreams.
- asoc.max_init_attempts = sp.initmsg.sinit_max_attempts.
- asoc.max_init_timeo = msecs_to_jiffies(sp.initmsg.sinit_max_init_timeo).
- /* Local rcv window */
- asoc.rwnd = max(SCTP_DEFAULT_MINWINDOW, sk.sk_rcvbuf / 2).
- asoc.a_rwnd = asoc.rwnd.
- asoc.peer.rwnd = SCTP_DEFAULT_MAXWINDOW.
- atomic_set(&asoc.rmem_alloc, 0); init_waitqueue_head(&asoc.wait).
- /* Tags */
- asoc.c.my_vtag = sctp_generate_tag(ep); my_port = ep.base.bind_addr.port.
- asoc.c.initial_tsn = sctp_generate_tsn(ep).
- asoc.next_tsn = initial_tsn.
- asoc.ctsn_ack_point = next_tsn - 1; adv_peer_ack_point = highest_sacked = last_cwr_tsn = ctsn_ack_point.
- asoc.addip_serial = strreset_outseq = c.initial_tsn.
- INIT_LIST_HEAD(addip_chunk_list, asconf_ack_list, peer.transport_addr_list, asocs).
- asoc.peer.sack_needed = 1; sack_generation = 1.
- /* I/O */
- sctp_inq_init(&base.inqueue); sctp_inq_set_th_handler(&base.inqueue, sctp_assoc_bh_rcv).
- sctp_outq_init(asoc, &outqueue).
- sctp_ulpq_init(&ulpq, asoc).
- sctp_stream_init(&stream, c.sinit_num_ostreams, 0, gfp); on failure ⇒ stream_free.
- asoc.pathmtu = sp.pathmtu; sctp_assoc_update_frag_point(asoc).
- asoc.peer.ipv4_address = 1; if sk.sk_family == PF_INET6: peer.ipv6_address = 1.
- /* Defaults */
- asoc.default_stream / default_ppid / default_flags / default_context / default_timetolive / default_rcv_context ← sp.
- /* AUTH */
- INIT_LIST_HEAD(endpoint_shared_keys).
- sctp_auth_asoc_copy_shkeys(ep, asoc, gfp); on failure ⇒ stream_free.
- active_key_id = ep.active_key_id; strreset_enable = ep.strreset_enable.
- copy ep.auth_hmacs_list / auth_chunk_list into c.auth_hmacs / c.auth_chunks.
- AUTH_RANDOM param: c.auth_random sampled via get_random_bytes(SCTP_AUTH_RANDOM_LENGTH).

REQ-3: sctp_association_new(ep, sk, scope, gfp):
- kzalloc_obj(*asoc, gfp).
- sctp_association_init(asoc, ep, sk, scope, gfp); on NULL ⇒ kfree + return NULL.
- SCTP_DBG_OBJCNT_INC(assoc).
- return asoc.

REQ-4: sctp_association_free(asoc):
- if !list_empty(&asoc.asocs): list_del; if TCP-style+LISTENING ⇒ sk_acceptq_removed(sk).
- asoc.base.dead = true.
- sctp_outq_free(&asoc.outqueue).
- for_each in peer.transport_addr_list: sctp_transport_free.
- timers: for each i ∈ NUM_TIMEOUT_TYPES: timer_delete(&asoc.timers[i]).
- sctp_asconf_queue_teardown(asoc).
- /* Drop endpoint reference and association reference */
- sctp_association_put(asoc).

REQ-5: sctp_association_destroy(asoc):
- if WARN_ON(!asoc.base.dead): return.
- WARN_ON(atomic_read(&asoc.rmem_alloc)).
- sctp_stream_free(&stream).
- sctp_endpoint_put(asoc.ep); sock_put(asoc.base.sk).
- if asoc.assoc_id ≠ 0: sctp_assoc_clear_id(asoc).
- /* Free AUTH peer state */
- kfree(peer.peer_random); kfree(peer.peer_chunks); kfree(peer.peer_hmacs).
- sctp_auth_destroy_keys(&endpoint_shared_keys).
- SCTP_DBG_OBJCNT_DEC(assoc).
- kfree(asoc).

REQ-6: sctp_association_hold(asoc):
- refcount_inc(&asoc.base.refcnt).

REQ-7: sctp_association_put(asoc):
- if refcount_dec_and_test(&asoc.base.refcnt): sctp_association_destroy(asoc).

REQ-8: sctp_association_get_next_tsn(asoc):
- retval = asoc.next_tsn.
- asoc.next_tsn++; asoc.unack_data++.
- return retval.   /* RFC 4960 § 1.6: wraps at 2^32 - 1 */

REQ-9: sctp_assoc_set_primary(asoc, transport):
- asoc.peer.primary_path = transport.
- memcpy(&asoc.peer.primary_addr, &transport.ipaddr, sizeof(union sctp_addr)).
- if !asoc.peer.active_path: asoc.peer.active_path = transport.
- if transport.state != SCTP_UNCONFIRMED ∧ asoc.peer.retran_path == NULL: retran_path = transport.

REQ-10: sctp_assoc_add_peer(asoc, addr, gfp, peer_state):
- port = ntohs(addr.v4.sin_port).
- if asoc.peer.port == 0: asoc.peer.port = port.
- peer = sctp_assoc_lookup_paddr(asoc, addr); if peer ∧ state == SCTP_UNKNOWN ⇒ state = SCTP_ACTIVE; return peer (dedup).
- peer = sctp_transport_new(net, addr, gfp); if !peer return NULL.
- sctp_transport_set_owner(peer, asoc).
- peer.hbinterval = asoc.hbinterval; probe_interval; encap_port; pathmaxrxt; pf_retrans; ps_retrans; sackdelay; sackfreq; param_flags; dscp.
- if v6 ∧ flowinfo nonzero: peer.flowlabel = ntohl(flowinfo & IPV6_FLOWLABEL_MASK) | SCTP_FLOWLABEL_SET_MASK.
- else: peer.flowlabel = asoc.flowlabel.
- sctp_transport_route(peer, NULL, sp).
- sctp_assoc_set_pmtu(asoc, asoc.pathmtu ? min(peer.pathmtu, asoc.pathmtu) : peer.pathmtu).
- sctp_packet_init(&peer.packet, peer, asoc.base.bind_addr.port, asoc.peer.port).
- /* RFC 4960 § 7.2.1 slow-start */
- peer.cwnd = min(4 * asoc.pathmtu, max(2 * asoc.pathmtu, 4380)).
- peer.ssthresh = SCTP_DEFAULT_MAXWINDOW.
- peer.rto = asoc.rto_initial; sctp_max_rto(asoc, peer).
- peer.state = peer_state.
- if sctp_hash_transport(peer): sctp_transport_free; return NULL.
- sctp_transport_pl_reset(peer).
- list_add_tail_rcu(&peer.transports, &asoc.peer.transport_addr_list); asoc.peer.transport_count++.
- sctp_ulpevent_notify_peer_addr_change(peer, SCTP_ADDR_ADDED, 0).
- if !asoc.peer.primary_path: sctp_assoc_set_primary(asoc, peer); asoc.peer.retran_path = peer.
- if asoc.peer.active_path == asoc.peer.retran_path ∧ peer.state ≠ SCTP_UNCONFIRMED: asoc.peer.retran_path = peer.
- return peer.

REQ-11: sctp_assoc_rm_peer(asoc, peer):
- emit SCTP_ADDR_REMOVED notification.
- if peer == asoc.peer.primary_path: pick replacement primary via sctp_assoc_set_primary or NULL.
- if peer in (active_path, retran_path, last_data_from): clear / re-elect via sctp_select_active_and_retran_path.
- sctp_unhash_transport(peer).
- list_del_rcu(&peer.transports); asoc.peer.transport_count--.
- sctp_transport_free(peer).

REQ-12: sctp_assoc_lookup_paddr(asoc, addr):
- list_for_each_entry(t, &asoc.peer.transport_addr_list, transports):
  - if sctp_cmp_addr_exact(addr, &t.ipaddr): return t.
- return NULL.

REQ-13: sctp_assoc_del_nonprimary_peers(asoc, primary):
- list_for_each_entry_safe in transport_addr_list:
  - if t ≠ primary: sctp_assoc_rm_peer(asoc, t).

REQ-14: sctp_assoc_control_transport(asoc, transport, command, error):
- per-command ∈ { SCTP_TRANSPORT_UP, _DOWN, _PF, _CONFIRMED, … }:
  - UP: transport.state = SCTP_ACTIVE; spc_state = SCTP_ADDR_AVAILABLE.
  - DOWN: transport.state = SCTP_INACTIVE; spc_state = SCTP_ADDR_UNREACHABLE.
  - PF: transport.state = SCTP_PF; spc_state = SCTP_ADDR_POTENTIALLY_FAILED.
  - CONFIRMED: transport.state = SCTP_ACTIVE (newly confirmed); spc_state = SCTP_ADDR_CONFIRMED.
  - default: return (no ULP-notify).
- if ulp_notify: sctp_ulpevent_notify_peer_addr_change(transport, spc_state, error).
- sctp_select_active_and_retran_path(asoc).

REQ-15: sctp_trans_score(trans):
- SCTP_ACTIVE → 3; SCTP_UNKNOWN → 2; SCTP_PF → 1; default (INACTIVE) → 0.

REQ-16: sctp_trans_elect_best(curr, best):
- if best == NULL ∨ curr == best: return curr.
- if score(curr) > score(best): return curr.
- if score(curr) == score(best): return sctp_trans_elect_tie(best, curr).
- else: return best.

REQ-17: sctp_trans_elect_tie(trans1, trans2):
- if trans1.error_count > trans2.error_count: return trans2.
- else if equal ∧ trans2.last_time_heard > trans1.last_time_heard: return trans2.
- else: return trans1.

REQ-18: sctp_assoc_update_retran_path(asoc):
- start = asoc.peer.retran_path.
- iterate transport_addr_list in round-robin order from start:
  - next = list_next_or_first(transport_addr_list, t).
  - if score(next) ≥ 2 (ACTIVE/UNKNOWN) ∧ next ≠ primary_path: asoc.peer.retran_path = next; return.
- /* Fallback: any PF */ pick best-PF.
- /* Last resort */: keep current.

REQ-19: sctp_assoc_choose_alter_transport(asoc, last_sent_to):
- if last_sent_to == NULL: return asoc.peer.active_path.
- if last_sent_to == asoc.peer.retran_path: sctp_assoc_update_retran_path(asoc).
- return asoc.peer.retran_path.

REQ-20: sctp_assoc_update_frag_point(asoc):
- frag = sctp_mtu_payload(sp, asoc.pathmtu, sctp_datachk_len(&asoc.stream)).
- if asoc.user_frag: frag = min(frag, user_frag).
- frag = min(frag, SCTP_MAX_CHUNK_LEN - sctp_datachk_len(&asoc.stream)).
- asoc.frag_point = SCTP_TRUNC4(frag).

REQ-21: sctp_assoc_set_pmtu(asoc, pmtu):
- if asoc.pathmtu ≠ pmtu: asoc.pathmtu = pmtu; sctp_assoc_update_frag_point(asoc).

REQ-22: sctp_assoc_sync_pmtu(asoc):
- iterate transport_addr_list:
  - if t.pmtu_pending ∧ t.dst: sctp_transport_update_pmtu(t, atomic_read(&t.mtu_info)); t.pmtu_pending = 0.
  - pmtu = (pmtu == 0 ∨ t.pathmtu < pmtu) ? t.pathmtu : pmtu.
- sctp_assoc_set_pmtu(asoc, pmtu).

REQ-23: sctp_peer_needs_update(asoc):
- state ∈ { ESTABLISHED, SHUTDOWN_PENDING, SHUTDOWN_RECEIVED, SHUTDOWN_SENT } ∧ (rwnd > a_rwnd) ∧ (rwnd - a_rwnd ≥ max(sk_rcvbuf >> rwnd_upd_shift, pathmtu)) ⇒ true.

REQ-24: sctp_assoc_rwnd_increase(asoc, len):
- account against rwnd_over first; remainder grows asoc.rwnd.
- recover rwnd_press: change = min(pathmtu, rwnd_press); rwnd += change; rwnd_press -= change.
- if sctp_peer_needs_update(asoc):
  - asoc.a_rwnd = asoc.rwnd.
  - sack = sctp_make_sack(asoc); asoc.peer.sack_needed = 0.
  - sctp_outq_tail(&asoc.outqueue, sack, GFP_ATOMIC).
  - timer_delete(&timers[SACK]) ⇒ sctp_association_put(asoc).

REQ-25: sctp_assoc_rwnd_decrease(asoc, len):
- if rx_count ≥ sk_rcvbuf: over = 1.
- if asoc.rwnd ≥ len:
  - asoc.rwnd -= len.
  - if over: asoc.rwnd_press += rwnd; rwnd = 0.
- else: asoc.rwnd_over += len - rwnd; rwnd = 0.

REQ-26: sctp_assoc_bh_rcv(work) — workqueue trampoline:
- asoc = container_of(work, struct sctp_association, base.inqueue.immediate).
- sctp_association_hold(asoc).
- while ((chunk = sctp_inq_pop(&asoc.base.inqueue))):
  - state = asoc.state; subtype = SCTP_ST_CHUNK(chunk.chunk_hdr.type).
  - /* SCTP-AUTH section 6.3 */
  - if first_time ∧ subtype.chunk == SCTP_CID_AUTH:
    - peek next chunk; if COOKIE_ECHO: clone skb to chunk.auth_chunk; chunk.auth = 1; continue (defer AUTH processing).
  - normal: if sctp_auth_recv_cid(subtype.chunk, asoc) ∧ !chunk.auth: continue (silent drop).
  - if sctp_chunk_is_data(chunk): asoc.peer.last_data_from = chunk.transport.
  - else: SCTP_INC_STATS(net, MIB_INCTRLCHUNKS); ictrlchunks++; SACK ⇒ isacks++.
  - if chunk.transport: chunk.transport.last_time_heard = ktime_get().
  - error = sctp_do_sm(net, SCTP_EVENT_T_CHUNK, subtype, state, ep, asoc, chunk, GFP_ATOMIC).
  - if asoc.base.dead: break.
  - if error: chunk.pdiscard = 1.
  - first_time = 0.
- sctp_association_put(asoc).

REQ-27: sctp_assoc_migrate(asoc, newsk):
- list_del_init(&asoc.asocs).
- if TCP-style oldsk: sk_acceptq_removed(oldsk).
- sctp_endpoint_put(asoc.ep); sock_put(asoc.base.sk).
- asoc.ep = newsp.ep; sctp_endpoint_hold(asoc.ep).
- asoc.base.sk = newsk; sock_hold(newsk).
- sctp_endpoint_add_asoc(newsp.ep, asoc).

REQ-28: sctp_assoc_update(asoc, new) — restart / cookie-collision:
- copy peer parameters: asoc.c = new.c; peer.rwnd / sack_needed / auth_capable / i.
- sctp_tsnmap_init(&peer.tsn_map, SCTP_TSN_MAP_INITIAL, peer.i.initial_tsn, GFP_ATOMIC); -ENOMEM on failure.
- iterate transport_addr_list:
  - if !sctp_assoc_lookup_paddr(new, &t.ipaddr): sctp_assoc_rm_peer(asoc, t); continue.
  - if state ≥ SCTP_STATE_ESTABLISHED: sctp_transport_reset(t).
- if asoc.state ≥ SCTP_STATE_ESTABLISHED (Case A — restart):
  - asoc.next_tsn = new.next_tsn.
  - asoc.ctsn_ack_point = new.ctsn_ack_point.
  - asoc.adv_peer_ack_point = new.adv_peer_ack_point.
  - sctp_stream_clear(&asoc.stream).
  - sctp_ulpq_flush(&asoc.ulpq).
  - asoc.overall_error_count = 0.
- else (Case B):
  - iterate new.peer.transport_addr_list: sctp_assoc_add_peer(asoc, addr, GFP_ATOMIC, state); -ENOMEM on failure.
  - asoc.ctsn_ack_point = asoc.next_tsn - 1; adv_peer_ack_point = ctsn_ack_point.
  - if state COOKIE_WAIT: sctp_stream_update(&asoc.stream, &new.stream).
  - sctp_assoc_set_id(asoc, GFP_ATOMIC); -ENOMEM on failure.
- transfer AUTH peer state (peer_random / peer_chunks / peer_hmacs).
- sctp_auth_asoc_init_active_key(asoc, GFP_ATOMIC).

REQ-29: sctp_assoc_set_id(asoc, gfp):
- IDR-allocate a non-zero `sctp_assoc_t` within net.sctp.assocs_idr.
- on success: asoc.assoc_id = id; return 0.
- on -ENOMEM: return -ENOMEM.

REQ-30: sctp_assoc_lookup_tsn(asoc, tsn):
- key = htonl(tsn); match = NULL.
- /* Optimistic check on active path first */
- iterate transmitted list of active_path:
  - if chunk.subh.data_hdr.tsn == key: return active.
- /* Else scan all other transports */
- for transport in transport_addr_list (skip active):
  - iterate transmitted list:
    - if match: return transport.

REQ-31: sctp_get_ecne_prepend(asoc):
- if !asoc.need_ecne: return NULL.
- return sctp_make_ecne(asoc, asoc.last_ecne_tsn).

REQ-32: sctp_cmp_addr_exact(ss1, ss2):
- af = sctp_get_af_specific(ss1.sa.sa_family); if !af: return 0.
- return af.cmp_addr(ss1, ss2).

REQ-33: sctp_assoc_set_bind_addr_from_ep(asoc, scope, gfp):
- flags = (sk.sk_family == PF_INET6) ? SCTP_ADDR6_ALLOWED : 0.
- if !inet_v6_ipv6only(sk): flags |= SCTP_ADDR4_ALLOWED.
- if peer.ipv4_address: flags |= SCTP_ADDR4_PEERSUPP.
- if peer.ipv6_address: flags |= SCTP_ADDR6_PEERSUPP.
- return sctp_bind_addr_copy(net, &asoc.base.bind_addr, &ep.base.bind_addr, scope, gfp, flags).

REQ-34: sctp_assoc_set_bind_addr_from_cookie(asoc, cookie, gfp):
- parse cookie.peer_addr_param vector into asoc.base.bind_addr (additional addresses learned from COOKIE).

REQ-35: sctp_assoc_lookup_laddr(asoc, laddr):
- scan asoc.base.bind_addr.address_list for laddr match; respect scope/type.

REQ-36: sctp_assoc_lookup_asconf_ack(asoc, serial):
- iterate asoc.asconf_ack_list; return cached ASCONF-ACK chunk whose serial == requested (ADDIP reordering).

REQ-37: sctp_assoc_clean_asconf_ack_cache(asoc):
- prune ASCONF-ACK cache to bounded depth (RFC 5061 § 5.4).

REQ-38: sctp_asconf_queue_teardown(asoc):
- sctp_assoc_free_asconf_acks(asoc).
- sctp_assoc_free_asconf_queue(asoc).

REQ-39: State machine integration:
- All asynchronous state transitions enter via `sctp_do_sm(net, event_type, subtype, state, ep, asoc, arg, gfp)` (defined in `sm_sideeffect.c`); event_type ∈ { CHUNK, TIMEOUT, OTHER, PRIMITIVE }.
- Transition table indexed by (state × subtype) returns a state-function (`sm_statefuns.c`) that emits side-effects via `sctp_cmd_seq`.
- `sctp_assoc_bh_rcv` is the chunk-driven entry; per-timer callbacks (T1_INIT, T1_COOKIE, T2_SHUTDOWN, T3_RTX, T4_RTO, T5_SHUTDOWN_GUARD, AUTOCLOSE, SACK, HEARTBEAT) are timeout-driven entries.

REQ-40: Heartbeat / RTO / multipath:
- per-transport `T3_RTX` timer: governed by transport.rto (initialized to asoc.rto_initial, updated via sctp_transport_update_rto on each ACK using RFC 4960 § 6.3.1).
- per-transport HEARTBEAT timer: hbinterval; on hb-failure increments transport.error_count; ≥ pf_retrans → SCTP_PF; ≥ pathmaxrxt → SCTP_INACTIVE; on asoc.overall_error_count > max_retrans → assoc-abort.
- on ACK of HEARTBEAT (HEARTBEAT-ACK) ⇒ sctp_assoc_control_transport(_, _, SCTP_TRANSPORT_UP, 0) when previously DOWN.

REQ-41: Multistream:
- sctp_stream_init(&asoc.stream, num_ostreams, in_streams = 0, gfp): per-direction SSN.
- sctp_stream_update / clear: handled in association restart.

REQ-42: Multipath (load-distribution choice):
- Per-DATA: asoc.peer.active_path is the next-hop.
- Per-retransmit: asoc.peer.retran_path; rotated via sctp_assoc_update_retran_path.
- sctp_select_active_and_retran_path: re-elects after sctp_assoc_control_transport state changes.

## Acceptance Criteria

- [ ] AC-1: sctp_association_new returns asoc with state == CLOSED, my_vtag non-zero, initial_tsn non-zero, base.refcnt == 1.
- [ ] AC-2: sctp_association_free marks base.dead = true; defers final free to sctp_association_destroy via refcount.
- [ ] AC-3: sctp_assoc_add_peer adds new transport to transport_addr_list, sets primary on first add, initializes cwnd = min(4*pmtu, max(2*pmtu, 4380)).
- [ ] AC-4: sctp_assoc_add_peer of duplicate addr in SCTP_UNKNOWN flips to SCTP_ACTIVE and returns existing.
- [ ] AC-5: sctp_assoc_lookup_paddr returns NULL when addr not in list; returns matching transport otherwise.
- [ ] AC-6: sctp_assoc_control_transport UP/DOWN/PF/CONFIRMED issues SCTP_PEER_ADDR_CHANGE notification with correct spc_state.
- [ ] AC-7: sctp_assoc_update_retran_path round-robins through transports with score ≥ 2; falls back to PF; never picks INACTIVE.
- [ ] AC-8: sctp_assoc_lookup_tsn returns the transport whose transmitted list contains the chunk with given TSN; NULL otherwise.
- [ ] AC-9: sctp_assoc_get_next_tsn returns next_tsn and post-increments; unack_data also incremented.
- [ ] AC-10: sctp_assoc_set_pmtu updates pathmtu and recomputes frag_point when pmtu changes.
- [ ] AC-11: sctp_assoc_sync_pmtu picks min(t.pathmtu) across transports and applies to asoc.
- [ ] AC-12: sctp_assoc_rwnd_increase issues SACK when sctp_peer_needs_update returns true; cancels SACK timer.
- [ ] AC-13: sctp_assoc_rwnd_decrease tracks rwnd_over / rwnd_press correctly when rx_count ≥ sk_rcvbuf.
- [ ] AC-14: sctp_assoc_bh_rcv handles AUTH-before-COOKIE_ECHO by carrying chunk.auth_chunk forward.
- [ ] AC-15: sctp_assoc_bh_rcv silently drops AUTH-required chunks that arrived without prior AUTH (SCTP-AUTH § 6.3).
- [ ] AC-16: sctp_assoc_bh_rcv breaks the loop when asoc.base.dead becomes true mid-dispatch.
- [ ] AC-17: sctp_assoc_migrate moves asoc to new sock, re-links to new endpoint, increments new and drops old refcounts.
- [ ] AC-18: sctp_assoc_update Case A (restart from established): clears stream / flushes ulpq / resets overall_error_count.
- [ ] AC-19: sctp_assoc_update Case B (cookie-collision pre-established): adds new peers and calls sctp_assoc_set_id.
- [ ] AC-20: sctp_assoc_set_id returns 0 + sets asoc.assoc_id to a non-zero IDR-allocated value on success.
- [ ] AC-21: sctp_assoc_del_nonprimary_peers leaves only the primary in transport_addr_list.
- [ ] AC-22: T5_SHUTDOWN_GUARD = 5 * rto_max (per sctpimpguide § 2.12.2).
- [ ] AC-23: sctp_get_ecne_prepend returns NULL when !need_ecne; returns ECNE chunk otherwise.
- [ ] AC-24: sctp_cmp_addr_exact returns 0 when AF unsupported (no sctp_af); returns af.cmp_addr otherwise.
- [ ] AC-25: sctp_assoc_set_bind_addr_from_ep computes scope flags per (sk.family, peer.ipv4_address, peer.ipv6_address, ipv6only).
- [ ] AC-26: sctp_assoc_clean_asconf_ack_cache bounds asconf_ack_list size; old entries evicted.
- [ ] AC-27: Free path: sctp_association_destroy frees AUTH peer state, drops endpoint, drops sock, releases assoc_id, kfrees asoc.

## Architecture

```
struct SctpAssociation {
  base: SctpEpCommon,                       // type = ASSOCIATION
  ep: *SctpEndpoint,
  state: SctpStateT,                        // CLOSED..SHUTDOWN_ACK_SENT
  c: SctpCookie,
  next_tsn: u32,
  ctsn_ack_point: u32,
  adv_peer_ack_point: u32,
  highest_sacked: u32,
  last_cwr_tsn: u32,
  unack_data: u32,
  rwnd: u32,
  a_rwnd: u32,
  rwnd_over: u32,
  rwnd_press: u32,
  rmem_alloc: AtomicI32,
  wait: WaitQueueHead,
  peer: SctpPeer {
    rwnd: u32,
    transport_addr_list: ListHead,          // RCU
    transport_count: u32,
    primary_path: *SctpTransport,
    active_path: *SctpTransport,
    retran_path: *SctpTransport,
    last_data_from: *SctpTransport,
    tsn_map: SctpTsnmap,
    sack_needed: bool,
    sack_generation: u8,
    auth_capable: bool,
    ipv4_address: bool,
    ipv6_address: bool,
    port: u16,
    i: SctpInithdr,
    peer_random: *u8,
    peer_chunks: *u8,
    peer_hmacs: *u8,
  },
  timers: [TimerList; SCTP_NUM_TIMEOUT_TYPES],
  timeouts: [u64; SCTP_NUM_TIMEOUT_TYPES],   // jiffies
  rto_initial: u32, rto_max: u32, rto_min: u32,
  hbinterval: u32, probe_interval: u32,
  pathmtu: u32, frag_point: u32, user_frag: u32,
  max_retrans: u16, pf_retrans: u16, ps_retrans: u16, pf_expose: u16,
  pathmaxrxt: u16, max_burst: u32,
  max_init_attempts: u32, max_init_timeo: u32,
  cookie_life: Ktime,
  subscribe: SctpEventSubscribe,
  encap_port: u16, flowlabel: u32, dscp: u8,
  sackdelay: u32, sackfreq: u16, param_flags: u32,
  outqueue: SctpOutq,
  ulpq: SctpUlpq,
  stream: SctpStream,
  asocs: ListHead,                          // ep.asocs linkage
  addip_chunk_list: ListHead,
  asconf_ack_list: ListHead,
  addip_serial: u32,
  strreset_outseq: u32,
  endpoint_shared_keys: ListHead,
  active_key_id: u16,
  strreset_enable: u16,
  assoc_id: SctpAssocT,
  overall_error_count: u8,
  need_ecne: bool,
  last_ecne_tsn: u32,
  pending_state: SctpStateT,
}
```

`SctpAssoc::new(ep, sk, scope, gfp) -> Option<Box<SctpAssoc>>`:
1. asoc = kzalloc(*asoc, gfp).
2. SctpAssoc::init(asoc, ep, sk, scope, gfp).
3. On init failure: kfree(asoc); return None.
4. SCTP_DBG_OBJCNT_INC(assoc).
5. return Some(asoc).

`SctpAssoc::init(asoc, ep, sk, scope, gfp) -> Result<()>`:
1. /* Base linkage */ asoc.ep = ep; base.sk = sk; base.net = sock_net(sk); type = ASSOCIATION; refcnt = 1.
2. sctp_endpoint_hold(ep); sock_hold(sk).
3. sctp_bind_addr_init(&base.bind_addr, ep.base.bind_addr.port).
4. /* Default state */ state = CLOSED.
5. Pull sock-level params: cookie_life, max_retrans, pf_retrans, ps_retrans, pf_expose, rto_*, hbinterval, probe_interval, encap_port, pathmaxrxt, sackdelay, sackfreq, param_flags, max_burst, subscribe.
6. /* Initial timeouts (jiffies) */ T1_COOKIE = T1_INIT = T2_SHUTDOWN = rto_initial; T5_SHUTDOWN_GUARD = 5 * rto_max; SACK = sackdelay; AUTOCLOSE = autoclose * HZ.
7. for i ∈ [NONE, NUM_TIMEOUT_TYPES): timer_setup(&timers[i], sctp_timer_events[i], 0).
8. Pull init params: sinit_max_instreams, sinit_num_ostreams, max_init_attempts, max_init_timeo.
9. rwnd = max(SCTP_DEFAULT_MINWINDOW, sk.sk_rcvbuf / 2); a_rwnd = rwnd; peer.rwnd = SCTP_DEFAULT_MAXWINDOW.
10. atomic_set(&rmem_alloc, 0); init_waitqueue_head(&wait).
11. /* Cookie + TSN seeds */ c.my_vtag = sctp_generate_tag(ep); c.my_port = ep.base.bind_addr.port; c.initial_tsn = sctp_generate_tsn(ep).
12. next_tsn = c.initial_tsn; ctsn_ack_point = next_tsn - 1; adv_peer_ack_point = highest_sacked = last_cwr_tsn = ctsn_ack_point.
13. addip_serial = strreset_outseq = c.initial_tsn.
14. INIT_LIST_HEAD on addip_chunk_list, asconf_ack_list, peer.transport_addr_list, asocs.
15. peer.sack_needed = 1; sack_generation = 1.
16. sctp_inq_init(&base.inqueue); sctp_inq_set_th_handler(&base.inqueue, SctpAssoc::bh_rcv).
17. sctp_outq_init(asoc, &outqueue); sctp_ulpq_init(&ulpq, asoc).
18. sctp_stream_init(&stream, c.sinit_num_ostreams, 0, gfp). On failure: stream_free path; release endpoint/sock; return Err.
19. pathmtu = sp.pathmtu; SctpAssoc::update_frag_point(asoc).
20. peer.ipv4_address = 1; if sk.family == PF_INET6: peer.ipv6_address = 1.
21. Defaults: default_stream, default_ppid, default_flags, default_context, default_timetolive, default_rcv_context ← sp.
22. INIT_LIST_HEAD(endpoint_shared_keys).
23. sctp_auth_asoc_copy_shkeys(ep, asoc, gfp). On failure: stream_free path.
24. active_key_id = ep.active_key_id; strreset_enable = ep.strreset_enable.
25. memcpy c.auth_hmacs / c.auth_chunks from ep.
26. p = (struct sctp_paramhdr *)c.auth_random; p.type = SCTP_PARAM_RANDOM; p.length = htons(sizeof(*p) + SCTP_AUTH_RANDOM_LENGTH); get_random_bytes(p+1, SCTP_AUTH_RANDOM_LENGTH).
27. return Ok.

`SctpAssoc::free(asoc)`:
1. if !list_empty(&asoc.asocs):
   - list_del(&asoc.asocs).
   - if TCP-style ∧ LISTENING(sk): sk_acceptq_removed(sk).
2. asoc.base.dead = true.
3. sctp_outq_free(&outqueue).
4. iter transport_addr_list: sctp_transport_free.
5. for i ∈ NUM_TIMEOUT_TYPES: timer_delete(&timers[i]).
6. SctpAssoc::asconf_queue_teardown(asoc).
7. SctpAssoc::put(asoc).

`SctpAssoc::add_peer(asoc, addr, gfp, peer_state) -> *SctpTransport`:
1. port = ntohs(addr.v4.sin_port).
2. if peer.port == 0: peer.port = port.
3. peer = SctpAssoc::lookup_paddr(asoc, addr). If !NULL:
   - if peer.state == SCTP_UNKNOWN: peer.state = SCTP_ACTIVE.
   - return peer.
4. peer = sctp_transport_new(net, addr, gfp); if !peer: return NULL.
5. sctp_transport_set_owner(peer, asoc).
6. Mirror asoc fields onto peer: hbinterval, probe_interval, encap_port, pathmaxrxt, pf_retrans, ps_retrans, sackdelay, sackfreq, param_flags, dscp.
7. v6 flowlabel inheritance: if addr is v6 ∧ flowinfo nonzero: peer.flowlabel = ntohl(flowinfo & FLOWLABEL_MASK) | SCTP_FLOWLABEL_SET_MASK. Else peer.flowlabel = asoc.flowlabel.
8. sctp_transport_route(peer, NULL, sp). pmtu_pending = 0.
9. SctpAssoc::set_pmtu(asoc, asoc.pathmtu ? min(peer.pathmtu, asoc.pathmtu) : peer.pathmtu).
10. sctp_packet_init(&peer.packet, peer, asoc.base.bind_addr.port, asoc.peer.port).
11. /* Slow-start (RFC 4960 § 7.2.1) */
    peer.cwnd = min(4 * asoc.pathmtu, max(2 * asoc.pathmtu, 4380)).
    peer.ssthresh = SCTP_DEFAULT_MAXWINDOW.
12. peer.partial_bytes_acked = 0; flight_size = 0; burst_limited = 0.
13. peer.rto = asoc.rto_initial; sctp_max_rto(asoc, peer).
14. peer.state = peer_state.
15. if sctp_hash_transport(peer): sctp_transport_free(peer); return NULL.
16. sctp_transport_pl_reset(peer).
17. list_add_tail_rcu(&peer.transports, &asoc.peer.transport_addr_list); transport_count++.
18. sctp_ulpevent_notify_peer_addr_change(peer, SCTP_ADDR_ADDED, 0).
19. if !asoc.peer.primary_path: SctpAssoc::set_primary(asoc, peer); asoc.peer.retran_path = peer.
20. if asoc.peer.active_path == asoc.peer.retran_path ∧ peer.state ≠ SCTP_UNCONFIRMED: asoc.peer.retran_path = peer.
21. return peer.

`SctpAssoc::control_transport(asoc, transport, cmd, error)`:
1. spc_state = SCTP_ADDR_AVAILABLE; ulp_notify = true.
2. match cmd:
   - SCTP_TRANSPORT_UP: transport.state = SCTP_ACTIVE; spc_state = SCTP_ADDR_AVAILABLE.
   - SCTP_TRANSPORT_DOWN: transport.state = SCTP_INACTIVE; spc_state = SCTP_ADDR_UNREACHABLE.
   - SCTP_TRANSPORT_PF: transport.state = SCTP_PF; spc_state = SCTP_ADDR_POTENTIALLY_FAILED.
   - SCTP_TRANSPORT_CONFIRMED: transport.state = SCTP_ACTIVE; spc_state = SCTP_ADDR_CONFIRMED.
   - default: return (no notify).
3. if ulp_notify: sctp_ulpevent_notify_peer_addr_change(transport, spc_state, error).
4. SctpAssoc::select_active_and_retran(asoc).

`SctpAssoc::bh_rcv(work)`:
1. asoc = container_of(work, SctpAssociation, base.inqueue.immediate).
2. SctpAssoc::hold(asoc).
3. while (chunk = sctp_inq_pop(&asoc.base.inqueue)):
   - state = asoc.state; subtype = SCTP_ST_CHUNK(chunk.chunk_hdr.type).
   - if first_time ∧ subtype.chunk == SCTP_CID_AUTH:
     - next_hdr = sctp_inq_peek; if next_hdr.type == SCTP_CID_COOKIE_ECHO: chunk.auth_chunk = skb_clone(chunk.skb, GFP_ATOMIC); chunk.auth = 1; continue.
   - if sctp_auth_recv_cid(subtype.chunk, asoc) ∧ !chunk.auth: continue (silent drop).
   - if sctp_chunk_is_data(chunk): peer.last_data_from = chunk.transport.
   - else: SCTP_INC_STATS(net, MIB_INCTRLCHUNKS); ictrlchunks++; if SACK ⇒ isacks++.
   - if chunk.transport: chunk.transport.last_time_heard = ktime_get().
   - error = sctp_do_sm(net, T_CHUNK, subtype, state, ep, asoc, chunk, GFP_ATOMIC).
   - if asoc.base.dead: break.
   - if error: chunk.pdiscard = 1.
   - first_time = 0.
4. SctpAssoc::put(asoc).

`SctpAssoc::update(asoc, new) -> i32`:
1. Copy peer params: asoc.c = new.c; peer.rwnd / sack_needed / auth_capable / i ← new.peer.*.
2. sctp_tsnmap_init(&peer.tsn_map, SCTP_TSN_MAP_INITIAL, peer.i.initial_tsn, GFP_ATOMIC). On ENOMEM ⇒ return -ENOMEM.
3. Iterate transport_addr_list:
   - if !SctpAssoc::lookup_paddr(new, &t.ipaddr): SctpAssoc::rm_peer(asoc, t); continue.
   - if state ≥ ESTABLISHED: sctp_transport_reset(t).
4. if asoc.state ≥ ESTABLISHED (Case A — restart):
   - next_tsn = new.next_tsn; ctsn_ack_point = new.ctsn_ack_point; adv_peer_ack_point = new.adv_peer_ack_point.
   - sctp_stream_clear(&asoc.stream).
   - sctp_ulpq_flush(&asoc.ulpq).
   - overall_error_count = 0.
5. else (Case B — pre-established):
   - Iterate new.peer.transport_addr_list: SctpAssoc::add_peer(asoc, &t.ipaddr, GFP_ATOMIC, t.state). On NULL ⇒ -ENOMEM.
   - ctsn_ack_point = next_tsn - 1; adv_peer_ack_point = ctsn_ack_point.
   - if asoc.state == COOKIE_WAIT: sctp_stream_update(&asoc.stream, &new.stream).
   - SctpAssoc::set_id(asoc, GFP_ATOMIC). On error ⇒ -ENOMEM.
6. Transfer AUTH peer state (peer_random / peer_chunks / peer_hmacs); kfree old; null out new.
7. return sctp_auth_asoc_init_active_key(asoc, GFP_ATOMIC).

`SctpAssoc::lookup_tsn(asoc, tsn) -> *SctpTransport`:
1. key = htonl(tsn); match = NULL.
2. active = asoc.peer.active_path.
3. iter active.transmitted:
   - if chunk.subh.data_hdr.tsn == key: match = active; break.
4. if match: return match.
5. for transport in transport_addr_list (skip active):
   - iter transport.transmitted:
     - if chunk.subh.data_hdr.tsn == key: match = transport; return.
6. return NULL.

`SctpAssoc::rwnd_increase(asoc, len)`:
1. if rwnd_over:
   - if rwnd_over ≥ len: rwnd_over -= len.
   - else: rwnd += len - rwnd_over; rwnd_over = 0.
2. else: rwnd += len.
3. /* Recover window pressure */
   if rwnd_press: change = min(pathmtu, rwnd_press); rwnd += change; rwnd_press -= change.
4. if SctpAssoc::peer_needs_update(asoc):
   - a_rwnd = rwnd.
   - sack = sctp_make_sack(asoc); if !sack: return.
   - peer.sack_needed = 0.
   - sctp_outq_tail(&outqueue, sack, GFP_ATOMIC).
   - if timer_delete(&timers[SACK]): SctpAssoc::put(asoc).

`SctpAssoc::migrate(asoc, newsk)`:
1. list_del_init(&asoc.asocs).
2. if TCP-style oldsk: sk_acceptq_removed(oldsk).
3. sctp_endpoint_put(asoc.ep); sock_put(asoc.base.sk).
4. asoc.ep = newsp.ep; sctp_endpoint_hold(asoc.ep).
5. asoc.base.sk = newsk; sock_hold(newsk).
6. sctp_endpoint_add_asoc(newsp.ep, asoc).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `state_init_closed` | INVARIANT | per-new asoc: state == SCTP_STATE_CLOSED. |
| `refcnt_balanced` | INVARIANT | per-association_hold/put pair: refcnt monotonic; destroy iff dec_and_test. |
| `transport_owner_set` | INVARIANT | per-add_peer: peer.asoc == asoc; peer in asoc.peer.transport_addr_list. |
| `primary_implies_active` | INVARIANT | per-set_primary: !active_path ⇒ active_path = primary; !retran_path ⇒ retran_path = primary (when CONFIRMED). |
| `path_state_no_double_count` | INVARIANT | per-control_transport: state transition single-shot per command; notification matched 1:1. |
| `tsn_strict_monotone` | INVARIANT | per-get_next_tsn: next_tsn strictly increases (mod 2^32). |
| `path_mtu_min_aggregated` | INVARIANT | per-sync_pmtu: asoc.pathmtu == min over transport.pathmtu. |
| `rwnd_over_press_consistent` | INVARIANT | per-rwnd_decrease/increase: rwnd_over and rwnd_press are non-negative; rwnd_over=0 when no over-commit. |
| `bh_rcv_dispatches_under_hold` | INVARIANT | per-bh_rcv: sctp_association_hold/put bracket the loop; loop exits if dead. |
| `update_caseA_clears_stream` | INVARIANT | per-update: state ≥ ESTABLISHED ⇒ sctp_stream_clear called. |
| `update_caseB_assigns_id` | INVARIANT | per-update: state < ESTABLISHED ⇒ sctp_assoc_set_id called. |
| `auth_random_filled` | INVARIANT | per-init: c.auth_random initialized with SCTP_AUTH_RANDOM_LENGTH random bytes. |
| `t5_shutdown_guard_5x_rto_max` | INVARIANT | per-init: timeouts[T5_SHUTDOWN_GUARD] == 5 * rto_max. |
| `lookup_tsn_active_first` | INVARIANT | per-lookup_tsn: probes active_path before remainder. |

### Layer 2: TLA+

`net/sctp/associola.tla`:
- States: CLOSED, COOKIE_WAIT, COOKIE_ECHOED, ESTABLISHED, SHUTDOWN_PENDING, SHUTDOWN_SENT, SHUTDOWN_RECEIVED, SHUTDOWN_ACK_SENT.
- Events: INIT, INIT_ACK, COOKIE_ECHO, COOKIE_ACK, DATA, SACK, SHUTDOWN, SHUTDOWN_ACK, SHUTDOWN_COMPLETE, ABORT, HEARTBEAT, HEARTBEAT_ACK, T1_INIT, T1_COOKIE, T2_SHUTDOWN, T3_RTX, T5_SHUTDOWN_GUARD.
- Properties:
  - `safety_state_only_via_sm` — per-asoc: state transitions only from sctp_do_sm dispatch.
  - `safety_dead_no_dispatch` — per-asoc: base.dead = true ⇒ no further state transitions.
  - `safety_refcnt_nonneg` — per-asoc: refcnt ≥ 0 at all times.
  - `safety_primary_always_in_list` — per-asoc: primary_path ∈ transport_addr_list ∨ NULL.
  - `safety_active_and_retran_in_list` — per-asoc: active_path / retran_path ∈ transport_addr_list ∨ NULL.
  - `safety_overall_error_bounded` — per-asoc: overall_error_count ≤ max_retrans before abort.
  - `safety_tsn_monotone` — per-asoc: next_tsn never regresses.
  - `liveness_init_eventually_terminates` — per-asoc: COOKIE_WAIT terminates as ESTABLISHED or aborts after max_init_attempts.
  - `liveness_shutdown_eventually_closes` — per-asoc: SHUTDOWN_PENDING reaches CLOSED via T5_SHUTDOWN_GUARD or normal flow.
  - `liveness_heartbeat_evicts_or_recovers` — per-transport: PF or INACTIVE eventually transitions via heartbeat or chunk activity.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SctpAssoc::init` post: state = CLOSED ∧ refcnt = 1 ∧ my_vtag ≠ 0 ∧ initial_tsn seeded | `SctpAssoc::init` |
| `SctpAssoc::free` post: base.dead = true; outqueue freed; timers cancelled | `SctpAssoc::free` |
| `SctpAssoc::add_peer` post: peer ∈ transport_addr_list ⊕ NULL (alloc fail) | `SctpAssoc::add_peer` |
| `SctpAssoc::rm_peer` post: peer ∉ transport_addr_list ∧ transport_count decremented | `SctpAssoc::rm_peer` |
| `SctpAssoc::control_transport` post: spc_state consistent with cmd; select_active_and_retran called | `SctpAssoc::control_transport` |
| `SctpAssoc::update_retran_path` post: retran_path.state ≥ SCTP_PF if any non-INACTIVE exists | `SctpAssoc::update_retran_path` |
| `SctpAssoc::set_pmtu` post: pathmtu = pmtu ∧ frag_point recomputed | `SctpAssoc::set_pmtu` |
| `SctpAssoc::sync_pmtu` post: pathmtu = min(transport.pathmtu) | `SctpAssoc::sync_pmtu` |
| `SctpAssoc::rwnd_increase` post: rwnd ≥ a_rwnd post-update; SACK queued if peer_needs_update | `SctpAssoc::rwnd_increase` |
| `SctpAssoc::bh_rcv` post: chunks consumed in FIFO; AUTH-COOKIE_ECHO carried as chunk.auth_chunk | `SctpAssoc::bh_rcv` |
| `SctpAssoc::update` post: peer state migrated; tsn_map reinitialized; assoc_id set if Case B | `SctpAssoc::update` |
| `SctpAssoc::migrate` post: asoc on new ep.asocs; old ep refcount decremented | `SctpAssoc::migrate` |
| `SctpAssoc::lookup_tsn` post: result.transmitted contains chunk w/ TSN ∨ NULL | `SctpAssoc::lookup_tsn` |
| `SctpAssoc::get_next_tsn` post: returns old next_tsn; next_tsn = old + 1; unack_data = old + 1 | `SctpAssoc::get_next_tsn` |

### Layer 4: Verus/Creusot functional

`Per-INIT → COOKIE_WAIT (T1_INIT, max_init_attempts) → COOKIE_ECHO → COOKIE_ECHOED (T1_COOKIE) → ESTABLISHED → SHUTDOWN-{PENDING,SENT,RECEIVED,ACK_SENT} (T2_SHUTDOWN, T5_SHUTDOWN_GUARD) → CLOSED` with per-transport `(SCTP_UNCONFIRMED → SCTP_ACTIVE) ∨ (→ SCTP_PF → SCTP_INACTIVE)` lifecycle and per-RTO/CC `T3_RTX → cwnd halving / ssthresh adjustment → retran via update_retran_path` semantic equivalence: per-RFC 4960 (SCTP), RFC 5061 (ASCONF), RFC 4895 (SCTP-AUTH), RFC 7053 (I-DATA / SCTP_INTERLEAVE), RFC 6458 (sockets API), and per-`Documentation/networking/sctp.rst`.

## Hardening

(Inherits row-1 features from `net/sctp/00-overview.md` § Hardening.)

Association-specific reinforcement:

- **Per-my_vtag random 32-bit** — defense against per-blind-INIT injection (RFC 4960 § 5.3).
- **Per-initial_tsn random 32-bit** — defense against per-TSN-prediction.
- **Per-AUTH_RANDOM 32-byte** — defense against per-rogue-AUTH replay (RFC 4895 § 6.1).
- **Per-T5_SHUTDOWN_GUARD = 5 × rto_max** — defense against per-shutdown-hang (sctpimpguide § 2.12.2).
- **Per-max_init_attempts bounded** — defense against per-INIT-flood.
- **Per-rwnd-press / rwnd_over accounting** — defense against per-rcv-buffer-over-commit.
- **Per-overall_error_count ≤ max_retrans** — defense against per-stuck assoc (assoc-abort threshold).
- **Per-transport.error_count ≥ pathmaxrxt** — defense against per-broken path (eviction).
- **Per-transport.state ≥ SCTP_PF threshold** — defense against per-grey-path (degraded selection).
- **Per-bh_rcv breaks on base.dead** — defense against per-UAF after free during dispatch.
- **Per-SCTP-AUTH chunk-list silent-drop** — defense against per-unauthenticated chunk injection.
- **Per-AUTH-before-COOKIE_ECHO carry** — defense against per-out-of-order AUTH bypass.
- **Per-list_add_tail_rcu transport_addr_list** — defense against per-RCU-reader race.
- **Per-IDR-allocated assoc_id non-zero** — defense against per-id-0 collision.
- **Per-sctp_assoc_lookup_paddr dedup** — defense against per-double-add transport.
- **Per-cwnd init min(4×pmtu, max(2×pmtu, 4380))** — defense against per-too-aggressive slow-start (RFC 4960 § 7.2.1).
- **Per-ssthresh init SCTP_DEFAULT_MAXWINDOW** — defense against per-uninitialized-ssthresh.
- **Per-pf_expose policy** — defense against per-PF-information-leak to ULP.
- **Per-asconf_ack_list trim** — defense against per-ASCONF-cache exhaustion.
- **Per-allow_infinite_fallback not applicable to SCTP** (distinct from MPTCP, but `peer.auth_capable` similarly gates AUTH-required chunks).
- **Per-flowlabel inheritance from v6 sin6_flowinfo** — defense against per-flow-tracking-bypass on multihomed v6.

## Grsecurity/PaX-style Reinforcement

Baseline PaX/grsecurity mitigations applicable to `sctp_association`:

- **PAX_USERCOPY** — `getsockopt(SCTP_GET_PEER_ADDRS)` / `setsockopt(SCTP_AUTH_KEY)` payloads under whitelist copy.
- **PAX_KERNEXEC** — assoc state-machine and transport-selection paths execute from W^X .text.
- **PAX_RANDKSTACK** — per-syscall stack randomization frustrates inference of TSN / verification-tag entropy.
- **PAX_REFCOUNT** — `sctp_association.base.refcnt`, `sctp_transport.refcnt`, and `sctp_chunk.refcnt` use saturating refcounts; INIT/COOKIE-ECHO storms cannot wrap.
- **PAX_MEMORY_SANITIZE** — `sctp_association`, `sctp_transport`, `sctp_chunk`, and the AUTH_RANDOM / COOKIE secret-key buffers sanitized on free.
- **PAX_UDEREF** — cmsg / sndinfo / rcvinfo parsing traverses user pointers under UDEREF.
- **PAX_RAP / kCFI** — `sctp_sm_table[][]` state-function pointers `static const`; dispatch CFI-checked.
- **GRKERNSEC_HIDESYM** — `sctp_endpoint_lookup_assoc`, `sctp_unpack_cookie` hidden from unprivileged kallsyms.
- **GRKERNSEC_DMESG** — INIT/COOKIE/AUTH failure warnings CAP_SYSLOG-gated.

Association-specific reinforcement:

- **`sctp_association.base.refcnt` PAX_REFCOUNT saturating** — defense against rapid create/abort wrap UAF.
- **COOKIE-ECHO HMAC key MEMORY_SANITIZE on rekey** — defense against stale-key residue leak after cookie-secret rotation.
- **GRKERNSEC_RANDNET TSN seeding** — `sctp_association.next_tsn` and `my_vtag` seeded from the hardened RNG pool, defending against TSN-prediction blind injection (RFC 4960 § 5.3).
- **AUTH_RANDOM 32-byte buffer MEMORY_SANITIZE on free** — defense against residual AUTH key disclosure after assoc release.
- **`sctp_endpoint_lookup_assoc` PAX_RAP-typed** — assoc-lookup dispatch CFI-verified to prevent fn-ptr confusion across SCTP state machine.
- **GRKERNSEC_HIDESYM on `sctp_unpack_cookie`** — cookie-validation symbol hidden from probing.

Rationale: SCTP's association is the canonical attacker target (verification tag, AUTH key, COOKIE secret); GRKERNSEC_RANDNET seeding plus MEMORY_SANITIZE on the secret-bearing fields and saturating refcounts on rapid assoc churn directly answer the historical SCTP CVE classes (UAF on shutdown, TSN-prediction, cookie-key reuse).

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/sctp/sm_statefuns.c per-state functions (covered separately if expanded)
- net/sctp/sm_sideeffect.c command dispatcher (covered separately if expanded)
- net/sctp/endpointola.c endpoint lifecycle (covered separately if expanded)
- net/sctp/transport.c per-transport details (covered separately if expanded)
- net/sctp/outqueue.c retransmit + congestion control (covered separately if expanded)
- net/sctp/auth.c SCTP-AUTH key derivation (covered separately if expanded)
- net/sctp/stream.c stream-reset / I-DATA (covered separately if expanded)
- net/sctp/ulpqueue.c reassembly (covered separately if expanded)
- net/sctp/sm_make_chunk.c chunk encode/decode (covered separately if expanded)
- net/sctp/input.c packet input demux (covered separately if expanded)
- net/sctp/socket.c sockets API (covered separately if expanded)
- Implementation code
