# Tier-3: net/mptcp/subflow.c — MPTCP subflow management

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/mptcp/00-overview.md
upstream-paths:
  - net/mptcp/subflow.c (~2215 lines)
  - net/mptcp/protocol.h (struct mptcp_subflow_context)
  - net/mptcp/options.c (DSS / MPC / MPJ / ADD_ADDR option pack-unpack)
  - net/mptcp/pm.c (path manager)
  - net/mptcp/pm_userspace.c (mptcp-pm-userspace netlink front-end)
  - net/mptcp/crypto.c (HMAC-SHA256 / key→token)
  - include/net/mptcp.h
  - include/uapi/linux/mptcp.h
-->

## Summary

MPTCP carries one logical connection across multiple TCP subflows; each subflow is a regular TCP socket overlaid with a ULP context. `struct mptcp_subflow_context` (~88-bytes) hangs off `icsk->icsk_ulp_data` and links the underlying TCP `struct sock` (`tcp_sock`) to its parent `struct mptcp_sock` (`msk`). Subflow creation has two flavours: the **initial** (MP_CAPABLE) subflow exchanges 64-bit local/remote keys in SYN/SYN-ACK/ACK; **additional** (MP_JOIN) subflows authenticate with a SHA-256 truncated-HMAC over local-nonce, remote-nonce, local-key, remote-key, addressed by a 32-bit token derived from `mptcp_crypto_key_sha`. Per-subflow data-plane: a Data Sequence Signal (DSS) option in TCP options maps the subflow byte-stream into a 64-bit MPTCP-level data sequence number (DSN); `get_mapping_status` validates each mapping (data_seq, subflow_seq, data_len, optional csum). Per-subflow add/remove is driven by the in-kernel path manager (`net/mptcp/pm.c`) or by mptcp-pm-userspace (`net/mptcp/pm_userspace.c`); ADD_ADDR / RM_ADDR options carry the address advertisement. Per-subflow priority is flipped by MP_PRIO (backup bit), and per-subflow failure mode is fastclose (RFC 8684 § 3.5) and infinite-mapping fallback. Critical for: multipath aggregation, hand-off across interfaces, mobility, and bonded throughput.

This Tier-3 covers `net/mptcp/subflow.c` (~2215 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mptcp_subflow_context` | per-subflow ULP context | `MptcpSubflowContext` |
| `struct mptcp_subflow_request_sock` | per-pending JOIN req | `MptcpSubflowReqSock` |
| `subflow_check_req()` | per-SYN parse MP_CAPABLE / MP_JOIN | `MptcpSubflow::check_req` |
| `subflow_init_req()` | per-request_sock init | `MptcpSubflow::init_req` |
| `subflow_token_join_request()` | per-MP_JOIN token→msk lookup | `MptcpSubflow::token_join_request` |
| `subflow_req_create_thmac()` | per-truncated-HMAC for SYN-ACK | `MptcpSubflow::req_create_thmac` |
| `subflow_generate_hmac()` | per-HMAC-SHA256 helper | `MptcpSubflow::generate_hmac` |
| `subflow_thmac_valid()` | per-validate SYN-ACK thmac | `MptcpSubflow::thmac_valid` |
| `subflow_hmac_valid()` | per-validate 3-ACK hmac | `MptcpSubflow::hmac_valid` |
| `mptcp_subflow_init_cookie_req()` | per-syncookie path | `MptcpSubflow::init_cookie_req` |
| `subflow_v4_route_req` / `_v6_route_req` | per-route_req hook | `MptcpSubflow::v4_route_req` / `v6_route_req` |
| `subflow_v4_send_synack` / `_v6_send_synack` | per-SYN-ACK send | `MptcpSubflow::v4_send_synack` / `v6_send_synack` |
| `subflow_v4_conn_request` / `_v6_conn_request` | per-LISTEN conn_request | `MptcpSubflow::v4_conn_request` / `v6_conn_request` |
| `subflow_syn_recv_sock()` | per-child-sock create on 3-ACK | `MptcpSubflow::syn_recv_sock` |
| `subflow_finish_connect()` | per-active SYN-ACK consumer | `MptcpSubflow::finish_connect` |
| `__mptcp_subflow_fully_established()` | per-subflow estd marker | `MptcpSubflow::fully_established` |
| `mptcp_subflow_reset()` | per-subflow RST + cleanup | `MptcpSubflow::reset` |
| `__mptcp_sync_state()` | per-msk state lift | `MptcpSubflow::sync_state` |
| `subflow_set_remote_key()` | per-msk remote key install | `MptcpSubflow::set_remote_key` |
| `mptcp_propagate_state()` | per-ssk → msk state propagation | `MptcpSubflow::propagate_state` |
| `subflow_set_local_id()` / `_chk_local_id` | per-local-id assignment | `MptcpSubflow::set_local_id` / `chk_local_id` |
| `subflow_rebuild_header()` / `_v6_rebuild_header` | per-header rebuild hook | `MptcpSubflow::rebuild_header` |
| `mptcp_subflow_reqsk_alloc()` | per-request_sock alloc | `MptcpSubflow::reqsk_alloc` |
| `mptcp_subflow_drop_ctx()` | per-fallback ctx-free | `MptcpSubflow::drop_ctx` |
| `get_mapping_status()` | per-DSS mapping parse | `MptcpSubflow::get_mapping_status` |
| `validate_mapping()` | per-mapping coverage check | `MptcpSubflow::validate_mapping` |
| `validate_data_csum()` | per-DSS optional csum | `MptcpSubflow::validate_data_csum` |
| `skb_is_fully_mapped()` | per-skb mapping coverage | `MptcpSubflow::skb_is_fully_mapped` |
| `mptcp_subflow_data_available()` | per-rx data-avail probe | `MptcpSubflow::data_available` |
| `subflow_check_data_avail()` | per-rx mapping loop | `MptcpSubflow::check_data_avail` |
| `mptcp_subflow_discard_data()` | per-rx duplicate-drop | `MptcpSubflow::discard_data` |
| `mptcp_subflow_fail()` | per-MP_FAIL graceful failure | `MptcpSubflow::fail` |
| `subflow_sched_work_if_closed()` | per-half-close handoff | `MptcpSubflow::sched_work_if_closed` |
| `subflow_data_ready()` / `_write_space` | per-tcp callback override | `MptcpSubflow::data_ready` / `write_space` |
| `subflow_state_change()` / `__subflow_state_change` | per-tcp state-change wrap | `MptcpSubflow::state_change` |
| `subflow_error_report()` | per-tcp error report wrap | `MptcpSubflow::error_report` |
| `mptcp_subflow_queue_clean()` | per-listener accept-queue purge | `MptcpSubflow::queue_clean` |
| `__mptcp_subflow_connect()` | per-PM-driven JOIN initiator | `MptcpSubflow::connect` |
| `mptcp_subflow_create_socket()` | per-subflow TCP-socket alloc | `MptcpSubflow::create_socket` |
| `subflow_create_ctx()` | per-ctx alloc on ULP init | `MptcpSubflow::create_ctx` |
| `subflow_ulp_init()` / `_release` / `_clone` | per-TCP-ULP ops | `MptcpSubflow::ulp_init` / `release` / `clone` |
| `subflow_ulp_fallback()` | per-fallback to plain TCP | `MptcpSubflow::ulp_fallback` |
| `tcp_release_cb_override()` | per-TCP release_cb override | `MptcpSubflow::release_cb_override` |
| `tcp_abort_override()` | per-TCP abort override | `MptcpSubflow::abort_override` |
| `mptcpv6_handle_mapped()` | per-v4-mapped-v6 switch | `MptcpSubflow::handle_v6_mapped` |
| `mptcp_info2sockaddr()` | per-mptcp_addr_info → sockaddr | `MptcpSubflow::info2sockaddr` |
| `__mptcp_inherit_memcg()` / `_cgrp_data` / `attach_cgroup` | per-cgroup inherit | `MptcpSubflow::inherit_*` |
| `subflow_ops_init()` / `mptcp_subflow_init()` / `mptcp_subflow_v6_init` | per-module init | `MptcpSubflow::init` / `init_v6` |
| `mptcp_space()` | per-rcv-window export | `MptcpSubflow::space` |

## Compatibility contract

REQ-1: struct mptcp_subflow_context (per `protocol.h`):
- node: list_head — per-msk `conn_list` linkage.
- local_key / remote_key: u64 — MPC keys (peer key valid only after MPC ACK).
- idsn: u64 — initial DSN derived `mptcp_crypto_key_sha(key) → token, idsn`.
- map_seq / map_subflow_seq / map_data_len / map_data_csum / map_csum_len: u64/u32 — current DSS mapping.
- rel_write_seq: u32 — relative write subflow seq inside DSS mapping.
- snd_isn / ssn_offset: u32 — TCP-seq base for the subflow seq number.
- token: u32 — MPTCP token (HASH(local_key) low-32).
- remote_token / local_nonce / remote_nonce: u32 — JOIN nonces.
- thmac: u64 — truncated-HMAC carried in MP_JOIN SYN-ACK.
- hmac[MPTCPOPT_HMAC_LEN]: 20-byte HMAC for 3rd ACK (MP_JOIN), unioned w/ iasn (MP_CAPABLE).
- request_mptcp / request_join / request_bkup: bitflags — outgoing intent.
- mp_capable / mp_join: bitflags — negotiated.
- backup / send_mp_prio / send_mp_fail / send_fastclose / send_infinite_map: bitflags — control-event tx.
- map_valid / map_data_fin / mpc_map / map_csum_reqd / valid_csum_seen: bitflags — DSS state.
- pm_notified / fully_established / pm_listener: bitflags — PM state.
- disposable / closing / stale / mpc_drop / close_event_done / is_mptfo / remote_key_valid: bitflags — lifecycle.
- data_avail / scheduled: bool — fast-path.
- local_id: s16 (negative ⇒ unassigned).
- remote_id: u8.
- reset_seen:1 / reset_transient:1 / reset_reason:4 / stale_count: u8 — reset bookkeeping.
- subflow_id: u32 — per-msk unique id.
- delegated_status / delegated_node / fail_tout: u64/list_head/ulong — delegated-actions, MP_FAIL timeout.
- setsockopt_seq: u32 — sockopt sync watermark.
- tcp_sock: *sock — TCP back-pointer.
- conn: *sock — parent msk back-pointer.
- icsk_af_ops: per-orig AF ops backup (override-restore).
- tcp_state_change / tcp_error_report: saved TCP callbacks.
- rcu: rcu_head — defer-free.

REQ-2: subflow_check_req(req, sk_listener, skb):
- /* Per-MD5SIG: incompatible with MPTCP */
- if rcu_access_pointer(tcp_sk(sk_listener).md5sig_info): reset(MPTCP_RST_EMPTCP); return -EINVAL.
- mptcp_get_options(skb, &mp_opt).
- opt_mp_capable = !!(suboptions & OPTION_MPTCP_MPC_SYN).
- opt_mp_join = !!(suboptions & OPTION_MPTCP_MPJ_SYN).
- if opt_mp_capable:
  - if listener.pm_listener: reset(MPTCP_RST_EPROHIBIT); return -EPERM.
- /* MP_CAPABLE active leg */
- if opt_mp_capable ∧ listener.request_mptcp:
  - up-to-MPTCP_TOKEN_MAX_RETRIES: get_random_bytes(&subflow_req.local_key, 8).
  - if syncookie: derive token/idsn via `mptcp_crypto_key_sha`.
  - else: mptcp_token_new_request(req). On collision: retry; on overflow: count MPTCP_MIB_TOKENFALLBACKINIT.
  - subflow_req.mp_capable = 1.
- /* MP_JOIN passive leg */
- else if opt_mp_join ∧ listener.request_mptcp:
  - subflow_req.mp_join = 1.
  - subflow_req.backup = mp_opt.backup.
  - subflow_req.remote_id = mp_opt.join_id.
  - subflow_req.token = mp_opt.token.
  - subflow_req.remote_nonce = mp_opt.nonce.
  - subflow_req.msk = subflow_token_join_request(req).
  - if !msk: reset(MPTCP_RST_EMPTCP); return -EPERM.
  - if subflow_use_different_sport ∧ !mptcp_pm_sport_in_anno_list(msk, sk): reset(MPTCP_RST_EPROHIBIT); return -EPERM.
  - subflow_req_create_thmac(subflow_req).
  - if syncookie ∧ !mptcp_can_accept_new_subflow(msk): reset(MPTCP_RST_EPROHIBIT); return -EPERM.

REQ-3: HMAC-SHA256:
- `mptcp_crypto_hmac_sha(key1, key2, msg, msg_len, out[32])`.
- `subflow_generate_hmac(k1, k2, n1, n2, out)`: msg = BE32(n1) ‖ BE32(n2); hmac key = k1 ‖ k2.
- truncated-HMAC (thmac) for SYN-ACK = `BE64(hmac[0..8])`.
- full 20-byte HMAC for 3rd ACK = `hmac[0..20]` (MPTCPOPT_HMAC_LEN).
- `subflow_thmac_valid(subflow)`: regenerate over (remote_key, local_key, remote_nonce, local_nonce); compare BE64(hmac[0..8]) == subflow.thmac.
- `subflow_hmac_valid(req, mp_opt)`: regenerate over (remote_key, local_key, remote_nonce, local_nonce); compare `crypto_memneq(hmac, mp_opt.hmac, MPTCPOPT_HMAC_LEN) == 0`.

REQ-4: Token / IDSN derivation:
- `mptcp_crypto_key_sha(key, &token, &idsn)` ⇒ SHA-256(key as 8-bytes BE); token = top 32 bits; idsn = next 64 bits.
- token uniquely identifies an msk via `mptcp_token_get_sock(net, token)`.
- token namespace is per-net; collisions cause MP_CAPABLE fallback.

REQ-5: subflow_syn_recv_sock(sk, skb, req, dst, req_unhash, own_req, opt_child_init):
- mp_opt.suboptions = 0.
- fallback_is_fatal = tcp_rsk(req).is_mptcp ∧ subflow_req.mp_join.
- fallback = !tcp_rsk(req).is_mptcp.
- if mp_capable: parse opts; if neither MPC_SYN | MPC_ACK present ⇒ fallback.
- if mp_join: parse opts; if no MPJ_ACK present ⇒ fallback.
- child = listener.icsk_af_ops.syn_recv_sock(sk, skb, req, dst, req_unhash, own_req, opt_child_init).
- if child ∧ own_req:
  - if !ctx ∨ fallback:
    - if fallback_is_fatal: reset(MPTCP_RST_EMPTCP); dispose_child.
    - else: drop ctx; return child as plain TCP.
  - if ctx.mp_capable:
    - ctx.conn = mptcp_sk_clone_init(listener.conn, mp_opt, child, req).
    - subflow_id = 1.
    - if deny_join_id0: msk.pm.remote_deny_join_id0 = true.
    - mptcp_pm_new_connection(owner, child, 1).
    - if MPC_ACK: mptcp_pm_fully_established; pm_notified = 1.
  - else if ctx.mp_join:
    - owner = subflow_req.msk; require !NULL else dispose.
    - if !subflow_hmac_valid: count MPTCP_MIB_JOINACKMAC; reset(MPTCP_RST_EMPTCP); dispose.
    - if !mptcp_can_accept_new_subflow: count MPTCP_MIB_JOINREJECTED; reset(MPTCP_RST_EPROHIBIT); dispose.
    - ctx.conn = msk.
    - if subflow_use_different_sport ∧ !mptcp_pm_sport_in_anno_list: reset; dispose.
    - mptcp_finish_join(child).
    - count MPTCP_MIB_JOINACKRX.
    - tcp_rsk(req).drop_req = true.

REQ-6: subflow_finish_connect(sk, skb) — active leg post-SYN-ACK:
- if subflow.conn_finished: skip.
- subflow.rel_write_seq = 1; subflow.conn_finished = 1; subflow.ssn_offset = TCP_SKB_CB(skb).seq.
- if subflow.request_mptcp:
  - if !(suboptions & OPTION_MPTCP_MPC_SYNACK):
    - mptcp_try_fallback(sk, MPTCP_MIB_MPCAPABLEACTIVEFALLBACK); else MIB_FALLBACKFAILED + reset.
  - else: csum-enable; deny_join_id0; subflow.mp_capable = 1; mptcp_finish_connect; mptcp_active_enable; mptcp_propagate_state.
- else if subflow.request_join:
  - if !(suboptions & OPTION_MPTCP_MPJ_SYNACK): reset_reason = MPTCP_RST_EMPTCP; do_reset.
  - subflow.backup / .thmac / .remote_nonce / .remote_id ← mp_opt.
  - if !subflow_thmac_valid: count MPTCP_MIB_JOINSYNACKMAC; reset.
  - if !mptcp_finish_join(sk): reset.
  - subflow_generate_hmac(local_key, remote_key, local_nonce, remote_nonce, hmac); memcpy(subflow.hmac, hmac, MPTCPOPT_HMAC_LEN).
  - subflow.mp_join = 1.
- else if mptcp_check_fallback(sk):
  - if subflow.mpc_drop: mptcp_active_disable(parent).
  - mptcp_propagate_state(parent, sk, subflow, NULL).

REQ-7: subflow_set_remote_key(msk, subflow, mp_opt):
- if subflow.remote_key_valid: skip.
- subflow.remote_key_valid = 1; subflow.remote_key = mp_opt.sndr_key.
- mptcp_crypto_key_sha(remote_key, NULL, &subflow.iasn); subflow.iasn++.
- subflow.map_seq = subflow.iasn.
- WRITE_ONCE(msk.remote_key = subflow.remote_key); msk.ack_seq = subflow.iasn; msk.can_ack = true.
- atomic64_set(&msk.rcv_wnd_sent, subflow.iasn).

REQ-8: __mptcp_subflow_fully_established(msk, subflow, mp_opt):
- subflow_set_remote_key.
- WRITE_ONCE(subflow.fully_established = true); WRITE_ONCE(msk.fully_established = true).

REQ-9: mptcp_propagate_state(sk, ssk, subflow, mp_opt):
- mptcp_data_lock(sk).
- if mp_opt:
  - WRITE_ONCE(msk.snd_una = subflow.idsn + 1).
  - WRITE_ONCE(msk.wnd_end = subflow.idsn + 1 + tcp_sk(ssk).snd_wnd).
  - subflow_set_remote_key.
- if !sock_owned_by_user(sk): __mptcp_sync_state(sk, ssk.sk_state).
- else: msk.pending_state = ssk.sk_state; __set_bit(MPTCP_SYNC_STATE, &msk.cb_flags).
- mptcp_data_unlock.

REQ-10: mptcp_subflow_reset(ssk):
- if ssk.sk_state == TCP_CLOSE: return.
- sock_hold(subflow.conn).
- mptcp_send_active_reset_reason(ssk).
- tcp_done(ssk).
- if !test_and_set_bit(MPTCP_WORK_CLOSE_SUBFLOW, &msk.flags): mptcp_schedule_work(sk).
- sock_put.

REQ-11: DSS mapping — get_mapping_status(ssk, msk):
- skb = skb_peek(ssk.sk_receive_queue); if !skb: MAPPING_EMPTY.
- if mptcp_check_fallback(ssk): MAPPING_DUMMY.
- mpext = mptcp_get_ext(skb).
- if !mpext ∨ !use_map:
  - if !subflow.map_valid ∧ !skb.len: drain 0-len FIN; MAPPING_EMPTY.
  - if !subflow.map_valid: MAPPING_NODSS.
  - else: goto validate_seq.
- data_len = mpext.data_len; if 0: count MPTCP_MIB_INFINITEMAPRX; MAPPING_INVALID.
- if mpext.data_fin == 1:
  - if data_len == 1: mptcp_update_rcv_data_fin(msk, mpext.data_seq, mpext.dsn64); MAPPING_DATA_FIN.
  - else: data_fin_seq = mpext.data_seq + data_len - 1; mask to 32-bit if !dsn64; data_len--.
- map_seq = mptcp_expand_seq(msk.ack_seq, mpext.data_seq, mpext.dsn64).
- WRITE_ONCE(msk.use_64bit_ack = mpext.dsn64).
- if subflow.map_valid:
  - if identical map: skb_ext_del; validate_csum.
  - if skb_is_fully_mapped: MAPPING_INVALID (no caching).
  - else: validate_csum (new map after current consumed).
- /* New map */
- subflow.map_seq / map_subflow_seq / map_data_len / map_valid / map_data_fin / mpc_map / map_csum_reqd / map_csum_len / map_data_csum ← mpext.
- if map_csum_reqd != csum_reqd: MAPPING_INVALID (RFC 8684 § 3.3.0).
- validate_mapping(ssk, skb): subflow_seq lies inside [map_subflow_seq, map_subflow_seq + map_data_len].
- validate_data_csum(ssk, skb, csum_reqd): csum_unfold(map_data_csum) folded into DSS-CSUM as bytes consumed.

REQ-12: subflow_check_data_avail(ssk):
- if !skb_peek(ssk.sk_receive_queue): subflow.data_avail = false.
- if subflow.data_avail: return true.
- loop:
  - status = get_mapping_status(ssk, msk).
  - if INVALID|DUMMY|BAD_CSUM|NODSS: fallback.
  - if OK: read msk.ack_seq, ack_seq = mptcp_subflow_get_mapped_dsn(subflow).
    - if before64(ack_seq, old_ack): mptcp_subflow_discard_data; continue.
    - else: subflow.data_avail = true; break.
  - else: no_data.
- return true.

REQ-13: mptcp_subflow_discard_data(ssk, skb, limit):
- offset = tp.copied_seq - TCP_SKB_CB(skb).seq.
- avail_len = skb.len - offset; incr = min(limit, avail_len) + (fin?1:0).
- count MPTCP_MIB_DUPDATA.
- tcp_sk(ssk).copied_seq += incr.
- if past end_seq: sk_eat_skb.
- if mapping consumed: subflow.map_valid = 0.

REQ-14: __mptcp_subflow_connect(sk, local, remote) — PM-driven JOIN:
- precondition: mptcp_is_fully_established(sk).
- mptcp_subflow_create_socket(sk, local.addr.family, &sf).
- subflow.local_nonce = get_random_u32() (nonzero).
- if local IPv4 ANY or IPv6 ANY: defer local_id (post-route).
- else: subflow_set_local_id.
- subflow.remote_key_valid = 1.
- subflow.remote_key = msk.remote_key; subflow.local_key = msk.local_key; subflow.token = msk.token.
- mptcp_info2sockaddr(local.addr, &addr, family).
- ssk.sk_bound_dev_if = local.ifindex.
- kernel_bind(sf, addr, addrlen).
- mptcp_crypto_key_sha(remote_key, &remote_token, NULL).
- subflow.remote_token = remote_token; remote_id; request_join = 1; request_bkup = !!(local.flags & MPTCP_PM_ADDR_FLAG_BACKUP).
- subflow.subflow_id = msk.subflow_id++.
- list_add_tail(&subflow.node, &msk.conn_list).
- kernel_connect(sf, remote_addr, O_NONBLOCK).
- mptcp_sock_graft; iput(SOCK_INODE(sf)); mptcp_stop_tout_timer.
- failure paths: mptcp_pm_close_subflow(msk).

REQ-15: mptcp_subflow_create_socket(sk, family, **new_sock):
- sock_create_kern(net, family, SOCK_STREAM, IPPROTO_TCP, &sf).
- lock_sock_nested(sf.sk, SINGLE_DEPTH_NESTING).
- security_mptcp_add_subflow(sk, sf.sk).
- mptcp_attach_cgroup(sk, sf.sk).
- sk_net_refcnt_upgrade(sf.sk).
- tcp_set_ulp(sf.sk, "mptcp"). [drives `subflow_ulp_init`]
- mptcp_sockopt_sync_locked(msk, sf.sk).

REQ-16: subflow_ulp_init(sk) / _release / _clone:
- _init: alloc ctx; rcu_assign icsk.icsk_ulp_data = ctx; save (icsk_af_ops, tcp_state_change, tcp_error_report); override icsk_af_ops to subflow_specific/v6_specific; override sk.sk_prot to tcp_prot_override/tcpv6_prot_override; ctx.tcp_sock = sk.
- _release: subflow_ulp_fallback (restore TCP ops, restore sk_prot); kfree_rcu(ctx).
- _clone(req, new_sk, old_ctx): allocate ctx for child; copy needed fields from listener / request_sock.

REQ-17: subflow_ulp_fallback(sk, old_ctx):
- mptcp_subflow_tcp_fallback(sk, old_ctx) (restore TCP callbacks).
- icsk.icsk_ulp_ops = NULL; rcu_assign_pointer(icsk.icsk_ulp_data, NULL).
- tcp_sk(sk).is_mptcp = 0.
- mptcp_subflow_ops_undo_override(sk).

REQ-18: subflow priority — MP_PRIO:
- subflow.send_mp_prio = 1 schedules emission of MP_PRIO option.
- subflow.backup = 1 marks subflow as standby (not for first-class scheduling).
- Receiver path: set via mp_opt.backup in MP_JOIN(SYN/SYN-ACK) or MP_PRIO control packet.

REQ-19: fastclose / fastopen:
- send_fastclose: tx of MP_FASTCLOSE option carrying remote_key; immediate teardown.
- is_mptfo: subflow is using TCP-Fast-Open; `mptcp_fastopen_subflow_synack_set_params` synthesises MPC handshake state.

REQ-20: subflow_sched_work_if_closed(msk, ssk):
- if ssk.sk_state ∉ {TCP_CLOSE, TCP_CLOSE_WAIT-while-msk-ESTABLISHED}: return.
- if !skb_queue_empty(ssk.sk_receive_queue): return (data still pending).
- if !test_and_set_bit(MPTCP_WORK_CLOSE_SUBFLOW, &msk.flags): mptcp_schedule_work(sk).
- if fallback ∧ subflow_is_done ∧ ssk == msk.first: synth data-fin via mptcp_update_rcv_data_fin.

REQ-21: mptcp_subflow_fail(msk, ssk):
- spin_lock_bh(&msk.fallback_lock).
- require msk.allow_infinite_fallback (else return false).
- msk.allow_subflows = false.
- spin_unlock_bh.
- require ssk == msk.first (MP_FAIL is allowed only on initial subflow).
- if SOCK_DEAD on msk: return true (already torn down).
- fail_tout = jiffies + TCP_RTO_MAX (zero ⇒ 1).
- WRITE_ONCE(subflow.fail_tout, fail_tout); tcp_send_ack(ssk).
- mptcp_reset_tout_timer(msk, subflow.fail_tout); return true.

REQ-22: mptcp_subflow_queue_clean(listener_sk, listener_ssk):
- Iterate listener.icsk_accept_queue.fastopenq and rskq_accept_head; drop subflow ctx; tcp_done child.
- Required to flush half-baked subflow children at listener teardown.

REQ-23: mptcp_info2sockaddr(info, sockaddr, family):
- AF_INET: sin_port = info.port; sin_addr = info.addr.
- AF_INET6: sin6_port = info.port; sin6_addr = info.addr6; sin6_scope_id = info.scope_id when LL.

REQ-24: subflow_data_ready(sk):
- subflow_check_data_avail(sk).
- mptcp_data_ready(parent, sk).
- subflow_sched_work_if_closed(msk, sk).

REQ-25: subflow_write_space(ssk):
- delegate to msk write-space wake (mptcp_write_space).

REQ-26: subflow_state_change(sk) / __subflow_state_change(sk):
- wrap-then-call-original tcp_state_change; on FIN-WAIT-1/CLOSE-WAIT/CLOSING/LAST-ACK/CLOSE: propagate to msk via mptcp_subflow_event_handler.

REQ-27: subflow_error_report(ssk):
- propagate ssk.sk_err / sk_err_soft to msk (sock_error_report on parent).

REQ-28: subflow_init_req(req, sk_listener):
- subflow_req.mp_capable = 0; .mp_join = 0.
- .csum_reqd = mptcp_is_checksum_enabled(net).
- .allow_join_id0 = mptcp_allow_join_id0(net).
- .msk = NULL.
- mptcp_token_init_request(req).

REQ-29: mptcp_subflow_init_cookie_req(req, sk_listener, skb):
- per-syncookie 3-ACK path; parse MPC_ACK or MPJ_ACK; assign local_key from mp_opt.rcvr_key for MPC; set ssn_offset = TCP_SKB_CB(skb).seq − 1.
- exclude opt_mp_capable ∧ opt_mp_join (EINVAL).

REQ-30: subflow_chk_local_id(sk):
- if subflow.local_id >= 0: return 0.
- err = mptcp_pm_get_local_id(msk, sk).
- subflow_set_local_id(subflow, err); request_bkup = mptcp_pm_is_backup(msk, sk).

REQ-31: Per-PM hook surface (consumed):
- `mptcp_pm_new_connection`, `mptcp_pm_fully_established`, `mptcp_pm_close_subflow`, `mptcp_pm_get_local_id`, `mptcp_pm_is_backup`, `mptcp_pm_sport_in_anno_list`, `mptcp_pm_is_userspace`, `mptcp_userspace_pm_active`.
- `mptcp_token_*` family: token registry per-net.

## Acceptance Criteria

- [ ] AC-1: MPC handshake: passive subflow_check_req sees MPC_SYN → allocates 64-bit local_key → mp_capable = 1 → SYN-ACK carries MPC_SYNACK with sndr_key/rcvr_key.
- [ ] AC-2: MPC handshake: active subflow_finish_connect on MPC_SYNACK sets remote_key/IDSN/mp_capable; mptcp_finish_connect runs.
- [ ] AC-3: MP_JOIN: passive on MPJ_SYN derives msk via token, generates thmac, sends SYN-ACK with thmac+local_nonce; rejects mismatched-port unless announced.
- [ ] AC-4: MP_JOIN: active on MPJ_SYNACK validates thmac via subflow_thmac_valid; on 3rd ACK builds full HMAC into subflow.hmac (20 bytes).
- [ ] AC-5: subflow_syn_recv_sock on MPJ_ACK runs subflow_hmac_valid; bad HMAC ⇒ MPTCP_RST_EMPTCP and dispose_child.
- [ ] AC-6: mptcp_can_accept_new_subflow gates JOIN-ACK accept; reject ⇒ MPTCP_RST_EPROHIBIT.
- [ ] AC-7: DSS parse: get_mapping_status accepts mapping w/ matching subflow_seq coverage; rejects non-identical replacement.
- [ ] AC-8: DSS infinite mapping (data_len == 0): triggers MIB_INFINITEMAPRX and MAPPING_INVALID.
- [ ] AC-9: DSS DATA_FIN (data_len == 1, data_fin == 1): updates msk rcv data-fin; MAPPING_DATA_FIN.
- [ ] AC-10: dsn64 == false ⇒ 32-bit DSN mask applied; mptcp_expand_seq promotes to 64-bit against msk.ack_seq.
- [ ] AC-11: DSS-CSUM mismatch ⇒ MAPPING_BAD_CSUM and MIB_DATACSUMERR.
- [ ] AC-12: Duplicate DSS ⇒ mptcp_subflow_discard_data advances copied_seq; MIB_DUPDATA.
- [ ] AC-13: __mptcp_subflow_connect requires fully-established msk; failure paths free socket and call mptcp_pm_close_subflow.
- [ ] AC-14: subflow create installs tcp_set_ulp("mptcp"); icsk_af_ops override; sk_prot override; cgroup-attached.
- [ ] AC-15: subflow_ulp_fallback restores plain TCP ops; tcp_sk(sk).is_mptcp = 0; ctx kfree_rcu'd.
- [ ] AC-16: MP_FAIL on non-first subflow ⇒ false; on first w/ allow_infinite_fallback ⇒ schedules fail_tout RTO_MAX.
- [ ] AC-17: subflow_sched_work_if_closed on TCP_CLOSE w/ empty recv queue sets MPTCP_WORK_CLOSE_SUBFLOW.
- [ ] AC-18: MP_PRIO bit drives subflow.backup flag; scheduler skips non-backup before backup.
- [ ] AC-19: mptcp_subflow_reset on already TCP_CLOSE subflow is no-op.
- [ ] AC-20: Cookie-init path: mp_capable ∧ mp_join simultaneously ⇒ EINVAL.
- [ ] AC-21: MD5SIG enabled on listener ⇒ MPTCP refused on inbound SYN (RST_EMPTCP).
- [ ] AC-22: pm_listener flag on listener ⇒ non-MPC, non-MPJ SYN ⇒ MPTCP_RST_EPROHIBIT.
- [ ] AC-23: Token collision in mptcp_token_new_request retried up to MPTCP_TOKEN_MAX_RETRIES; final fallback ⇒ MIB_TOKENFALLBACKINIT.
- [ ] AC-24: subflow_set_local_id rejects local_id ∉ [0, 255] (WARN_ON_ONCE).
- [ ] AC-25: mptcp_subflow_queue_clean drops half-baked subflow children at listener teardown without leaking ctx.

## Architecture

```
struct MptcpSubflowContext {
  node: ListHead,                       // msk.conn_list linkage
  avg_pacing_rate: u64,
  local_key: u64,
  remote_key: u64,
  idsn: u64,
  map_seq: u64,
  rcv_wnd_sent: u64,
  snd_isn: u32,
  token: u32,
  rel_write_seq: u32,
  map_subflow_seq: u32,
  ssn_offset: u32,
  map_data_len: u32,
  map_data_csum: Wsum,
  map_csum_len: u32,
  prev_rtt_seq: u32,
  flags: SubflowFlags,                  // request_mptcp / mp_capable / mp_join / backup / ...
  data_avail: bool,
  scheduled: bool,
  pm_listener: bool,
  fully_established: bool,
  lent_mem_frag: u32,
  remote_nonce: u32,
  thmac: u64,
  local_nonce: u32,
  remote_token: u32,
  auth: SubflowAuth,                    // hmac[20] for MPJ, iasn for MPC
  local_id: i16,
  remote_id: u8,
  reset_seen: u8,
  reset_transient: u8,
  reset_reason: u8,                     // 4 bits
  stale_count: u8,
  subflow_id: u32,
  delegated_status: u64,
  fail_tout: u64,
  delegated_node: ListHead,
  setsockopt_seq: u32,
  stale_rcv_tstamp: u32,
  cached_sndbuf: i32,
  tcp_sock: *Sock,
  conn: *Sock,                          // parent msk
  icsk_af_ops: *InetConnectionSockAfOps,
  tcp_state_change: fn(*Sock),
  tcp_error_report: fn(*Sock),
  rcu: RcuHead,
}

enum MappingStatus {
  Ok, Invalid, Empty, DataFin, Dummy, BadCsum, NoDss,
}
```

`MptcpSubflow::generate_hmac(k1, k2, n1, n2, out)`:
1. msg[0..4] = n1.to_be(); msg[4..8] = n2.to_be().
2. MptcpCrypto::hmac_sha(k1, k2, msg, out).

`MptcpSubflow::check_req(req, sk_listener, skb) -> i32`:
1. if tcp_sk(sk_listener).md5sig_info.is_some(): reset(MPTCP_RST_EMPTCP); return -EINVAL.
2. mp_opt = MptcpOptions::get(skb).
3. opt_mp_capable = mp_opt.suboptions.contains(MPC_SYN).
4. opt_mp_join = mp_opt.suboptions.contains(MPJ_SYN).
5. if opt_mp_capable: count MPC_PASSIVE; if listener.pm_listener: reset(MPTCP_RST_EPROHIBIT); return -EPERM.
6. else if opt_mp_join: count JOIN_SYN_RX (+JOIN_SYN_BACKUP_RX if backup).
7. else if listener.pm_listener: reset; return -EPERM.
8. if opt_mp_capable ∧ listener.request_mptcp:
   - retries = MPTCP_TOKEN_MAX_RETRIES.
   - loop: get_random_bytes(&subflow_req.local_key, 8) — must be nonzero.
   - if syncookie: mptcp_crypto_key_sha(local_key, &token, &idsn); if mptcp_token_exists: retry; else mp_capable = 1.
   - else: mptcp_token_new_request; on success mp_capable = 1; on collision retry; on exhaustion MIB_TOKENFALLBACKINIT.
9. else if opt_mp_join ∧ listener.request_mptcp:
   - subflow_req.ssn_offset = TCP_SKB_CB(skb).seq.
   - subflow_req.mp_join = 1; backup; remote_id; token; remote_nonce ← mp_opt.
   - subflow_req.msk = MptcpToken::join_request(req). On NULL: reset(MPTCP_RST_EMPTCP); return -EPERM.
   - if different sport ∧ !sport_in_anno_list: reset(MPTCP_RST_EPROHIBIT); return -EPERM.
   - subflow_req_create_thmac(subflow_req).
   - if syncookie ∧ !mptcp_can_accept_new_subflow: reset; return -EPERM. Save cookie-join state.

`MptcpSubflow::syn_recv_sock(sk, skb, req, dst, req_unhash, own_req, opt_child_init) -> *Sock`:
1. mp_opt.suboptions = 0.
2. fallback_is_fatal = tcp_rsk(req).is_mptcp ∧ subflow_req.mp_join.
3. fallback = !tcp_rsk(req).is_mptcp.
4. if subflow_req.mp_capable: parse opts; require MPC_SYN | MPC_ACK; else fallback = true.
5. else if subflow_req.mp_join: parse opts; require MPJ_ACK; else fallback = true.
6. child = listener.icsk_af_ops.syn_recv_sock(...).
7. if child ∧ own_req:
   - ctx = mptcp_subflow_ctx(child).
   - if !ctx ∨ fallback:
     - if fallback_is_fatal: reset; goto dispose_child.
     - goto fallback.
   - ctx.setsockopt_seq = listener.setsockopt_seq.
   - if ctx.mp_capable:
     - ctx.conn = mptcp_sk_clone_init(listener.conn, mp_opt, child, req); if !conn ⇒ fallback.
     - ctx.subflow_id = 1.
     - if mp_opt.deny_join_id0: msk.pm.remote_deny_join_id0 = true.
     - mptcp_pm_new_connection(msk, child, 1).
     - if mp_opt.suboptions.contains(MPC_ACK): mptcp_pm_fully_established(msk, child); ctx.pm_notified = 1.
   - else if ctx.mp_join:
     - owner = subflow_req.msk; require !NULL else dispose.
     - if !subflow_hmac_valid: count JOINACKMAC; reset(MPTCP_RST_EMPTCP); dispose.
     - if !mptcp_can_accept_new_subflow: count JOINREJECTED; reset(MPTCP_RST_EPROHIBIT); dispose.
     - subflow_req.msk = NULL; ctx.conn = msk.
     - if different sport ∧ !sport_in_anno_list: reset; dispose.
     - if !mptcp_finish_join: reset; dispose.
     - count MIB_JOINACKRX.
     - tcp_rsk(req).drop_req = true.
8. WARN_ON_ONCE(!ctx ∨ !ctx.conn) on mptcp child.
9. return child.

`MptcpSubflow::finish_connect(sk, skb)`:
1. subflow.icsk_af_ops.sk_rx_dst_set(sk, skb).
2. if subflow.conn_finished: return.
3. msk = mptcp_sk(parent); subflow.rel_write_seq = 1; conn_finished = 1; ssn_offset = TCP_SKB_CB(skb).seq.
4. mp_opt = MptcpOptions::get(skb).
5. if subflow.request_mptcp:
   - if !mp_opt.suboptions.contains(MPC_SYNACK):
     - if !mptcp_try_fallback(sk, MIB_MPC_ACTIVE_FALLBACK): count MIB_FALLBACKFAILED; do_reset.
     - else: goto fallback.
   - if mp_opt.suboptions.contains(CSUMREQD): msk.csum_enabled = true.
   - if mp_opt.deny_join_id0: msk.pm.remote_deny_join_id0 = true.
   - subflow.mp_capable = 1; count MIB_MPC_ACTIVE_ACK.
   - mptcp_finish_connect(sk); mptcp_active_enable(parent).
   - mptcp_propagate_state(parent, sk, subflow, &mp_opt).
6. else if subflow.request_join:
   - require mp_opt.suboptions.contains(MPJ_SYNACK); else reset_reason = MPTCP_RST_EMPTCP; do_reset.
   - subflow.backup = mp_opt.backup; thmac = mp_opt.thmac; remote_nonce = mp_opt.nonce; remote_id = mp_opt.join_id.
   - if !subflow_thmac_valid: count MIB_JOIN_SYNACK_MAC; reset.
   - if !mptcp_finish_join(sk): reset.
   - subflow_generate_hmac(local_key, remote_key, local_nonce, remote_nonce, &hmac).
   - subflow.hmac[..MPTCPOPT_HMAC_LEN] = hmac[..MPTCPOPT_HMAC_LEN].
   - subflow.mp_join = 1; count MIB_JOIN_SYNACK_RX (+BACKUP if backup, +DIFFPORT if different dport).
7. else if mptcp_check_fallback(sk):
   - if subflow.mpc_drop: mptcp_active_disable(parent).
   - mptcp_propagate_state(parent, sk, subflow, NULL).
8. do_reset: subflow.reset_transient = 0; mptcp_subflow_reset(sk).

`MptcpSubflow::get_mapping_status(ssk, msk) -> MappingStatus`:
1. skb = skb_peek(ssk.sk_receive_queue); if !skb: Empty.
2. if mptcp_check_fallback(ssk): Dummy.
3. mpext = mptcp_get_ext(skb).
4. if !mpext ∨ !mpext.use_map:
   - if !subflow.map_valid ∧ !skb.len: drop 0-len; Empty.
   - if !subflow.map_valid: NoDss.
   - else: goto validate_seq.
5. data_len = mpext.data_len; if 0: count INFINITEMAPRX; Invalid.
6. if mpext.data_fin == 1:
   - if data_len == 1: update_rcv_data_fin(msk, data_seq, dsn64); DataFin.
   - else: data_fin_seq = data_seq + data_len - 1; mask 32-bit if !dsn64; data_len -= 1.
7. map_seq = mptcp_expand_seq(msk.ack_seq, data_seq, dsn64); msk.use_64bit_ack = dsn64.
8. if subflow.map_valid:
   - if identical map: skb_ext_del; goto validate_csum.
   - if skb_is_fully_mapped: count DSSNOMATCH; Invalid.
   - else: goto validate_csum.
9. else: install new map (map_seq / map_subflow_seq / map_data_len / map_data_csum / ...).
10. if map_csum_reqd ≠ msk.csum_enabled: Invalid.
11. validate_seq: validate_mapping(ssk, skb); if false ⇒ count DSSTCPMISMATCH; Invalid.
12. validate_csum: validate_data_csum(ssk, skb, csum_reqd).

`MptcpSubflow::check_data_avail(ssk) -> bool`:
1. if skb_peek empty: subflow.data_avail = false.
2. if subflow.data_avail: return true.
3. loop:
   - status = get_mapping_status.
   - if Invalid|Dummy|BadCsum|NoDss: goto fallback.
   - if status ≠ Ok: goto no_data.
   - ack_seq = mptcp_subflow_get_mapped_dsn(subflow); old_ack = msk.ack_seq.
   - if !msk.can_ack: goto fallback.
   - if before64(ack_seq, old_ack): mptcp_subflow_discard_data(ssk, skb, old_ack - ack_seq); continue.
   - else: subflow.data_avail = true; break.
4. return true.

`MptcpSubflow::connect(sk, local, remote) -> i32`:
1. require mptcp_is_fully_established(sk) (else -ENOTCONN).
2. mptcp_subflow_create_socket(sk, local.addr.family, &sf).
3. ssk = sf.sk; subflow = mptcp_subflow_ctx(ssk).
4. loop: subflow.local_nonce = get_random_u32() until nonzero.
5. if local IPv4 ANY ∨ IPv6 ANY: local_id = -1 (defer).
6. else: subflow_set_local_id(subflow, local_id).
7. subflow.remote_key_valid = 1; remote_key = msk.remote_key; local_key = msk.local_key; token = msk.token.
8. mptcp_info2sockaddr(local.addr, &addr, family).
9. ssk.sk_bound_dev_if = local.ifindex; kernel_bind(sf, &addr, addrlen).
10. mptcp_crypto_key_sha(remote_key, &remote_token, NULL); subflow.remote_token = remote_token.
11. subflow.remote_id = remote.id; request_join = 1; request_bkup = (local.flags & BACKUP).
12. subflow.subflow_id = msk.subflow_id++.
13. mptcp_info2sockaddr(remote, &raddr, family).
14. sock_hold(ssk); list_add_tail(&subflow.node, &msk.conn_list).
15. kernel_connect(sf, &raddr, addrlen, O_NONBLOCK).
16. mptcp_sock_graft(ssk, sk.sk_socket); iput(SOCK_INODE(sf)); mptcp_stop_tout_timer(sk).
17. return 0; on failure list_del+sock_put+release+mptcp_pm_close_subflow.

`MptcpSubflow::reset(ssk)`:
1. if ssk.sk_state == TCP_CLOSE: return.
2. sock_hold(subflow.conn).
3. mptcp_send_active_reset_reason(ssk); tcp_done(ssk).
4. if !test_and_set_bit(MPTCP_WORK_CLOSE_SUBFLOW, &msk.flags): mptcp_schedule_work(parent).
5. sock_put(parent).

`MptcpSubflow::fail(msk, ssk) -> bool`:
1. spin_lock_bh(&msk.fallback_lock).
2. if !msk.allow_infinite_fallback: unlock; return false.
3. msk.allow_subflows = false; spin_unlock_bh.
4. if ssk ≠ msk.first: WARN_ON_ONCE; return false.
5. if SOCK_DEAD on msk: return true.
6. fail_tout = jiffies + TCP_RTO_MAX (≥ 1).
7. subflow.fail_tout = fail_tout; tcp_send_ack(ssk); mptcp_reset_tout_timer(msk, fail_tout).
8. return true.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mpc_handshake_keys_paired` | INVARIANT | per-MPC: subflow.local_key and remote_key both nonzero before mp_capable = 1. |
| `mpj_thmac_valid_before_mp_join_set` | INVARIANT | per-MPJ active: subflow.mp_join only set after subflow_thmac_valid. |
| `mpj_hmac_valid_before_accept` | INVARIANT | per-MPJ passive 3-ACK: hmac mismatch ⇒ dispose_child (no msk acquisition). |
| `token_unique_per_net` | INVARIANT | per-mptcp_token_new_request: collision detected; returns -EBUSY w/ retry. |
| `dss_no_partial_cover` | INVARIANT | per-get_mapping_status: new map after current fully consumed; never partial-covered. |
| `dsn64_consistency` | INVARIANT | per-mpext.dsn64: same flag across mappings of one subflow; else MIB_DSSNOMATCH. |
| `csum_required_match` | INVARIANT | per-DSS: map_csum_reqd == msk.csum_enabled or Invalid. |
| `subflow_ctx_attached_when_mptcp` | INVARIANT | per-syn_recv_sock: tcp_sk(child).is_mptcp ⇒ mptcp_subflow_ctx(child) != NULL ∧ ctx.conn != NULL. |
| `request_join_requires_msk` | INVARIANT | per-request_join active: msk pre-fetched via token; subflow.remote_key == msk.remote_key. |
| `local_id_byte_range` | INVARIANT | per-subflow_set_local_id: 0 ≤ local_id ≤ 255. |

### Layer 2: TLA+

`net/mptcp/subflow.tla`:
- Per-MPC handshake (SYN/SYN-ACK/ACK).
- Per-MPJ handshake (SYN/SYN-ACK/ACK + HMAC).
- Per-DSS mapping consume / new-map / data-fin / infinite-map fallback.
- Per-MP_FAIL / fastclose / fallback.
- Properties:
  - `safety_mpj_authenticates` — per-JOIN passive: server never moves to mp_join = 1 without valid 3-ACK HMAC.
  - `safety_mpj_authenticates_client` — per-JOIN active: client never moves to mp_join = 1 without valid SYN-ACK thmac.
  - `safety_token_uniqueness` — per-net: at most one msk per active token.
  - `safety_idsn_monotone` — per-subflow: idsn-driven DSN never regresses.
  - `safety_no_dual_capability` — per-request: ¬(mp_capable ∧ mp_join).
  - `safety_fail_only_on_first` — per-MP_FAIL: ssk == msk.first.
  - `liveness_full_estd_reached` — per-MPC: handshake terminates ⟹ msk.fully_established.
  - `liveness_join_completes_or_resets` — per-MPJ: terminates as accept-or-reject (no hang).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `check_req` post: mp_capable ⊕ mp_join (never both) | `MptcpSubflow::check_req` |
| `syn_recv_sock` post: ctx.mp_join ⇒ subflow_hmac_valid called | `MptcpSubflow::syn_recv_sock` |
| `finish_connect` post: request_join ⇒ subflow.hmac populated | `MptcpSubflow::finish_connect` |
| `get_mapping_status` post: result ∈ MappingStatus | `MptcpSubflow::get_mapping_status` |
| `connect` pre: msk.fully_established | `MptcpSubflow::connect` |
| `reset` post: ssk transitions to TCP_CLOSE | `MptcpSubflow::reset` |
| `fail` post: subflow.fail_tout ≠ 0 ⇒ tcp_send_ack issued | `MptcpSubflow::fail` |
| `discard_data` post: tp.copied_seq advanced; MIB_DUPDATA incremented | `MptcpSubflow::discard_data` |
| `set_remote_key` post: idempotent (remote_key_valid guards re-entry) | `MptcpSubflow::set_remote_key` |
| `ulp_init` post: icsk.icsk_ulp_data set; icsk_af_ops overridden | `MptcpSubflow::ulp_init` |
| `ulp_fallback` post: tcp_sk(sk).is_mptcp = 0; icsk_ulp_data NULL | `MptcpSubflow::ulp_fallback` |

### Layer 4: Verus/Creusot functional

`Per-MPC-handshake (SYN → SYN/ACK → ACK) → per-MPJ-handshake (SYN+token → SYN/ACK+thmac → ACK+HMAC) → per-DSS data plane → per-MP_PRIO / MP_FAIL / fastclose → per-fallback` semantic equivalence: per-RFC 8684 § 3.1 (MPC), § 3.2 (MPJ + HMAC), § 3.3 (DSS), § 3.4 (ADD_ADDR/RM_ADDR via PM hooks), § 3.5 (Close/Fastclose), and per-`Documentation/networking/mptcp.rst`.

## Hardening

(Inherits row-1 features from `net/mptcp/00-overview.md` § Hardening.)

Subflow-specific reinforcement:

- **Per-MPC key entropy ≥ 64-bit** — defense against per-MPC key-collision.
- **Per-MPJ HMAC-SHA256 over (k1, k2, n1, n2)** — defense against per-blind-JOIN injection.
- **Per-nonce nonzero / random** — defense against per-JOIN replay.
- **Per-token namespace per-net** — defense against per-cross-namespace hijack.
- **Per-MPTCP_TOKEN_MAX_RETRIES** — defense against per-token-exhaustion DoS.
- **Per-pm_listener flag refuses non-MPC SYN** — defense against per-leaking listener to plain TCP.
- **Per-MD5SIG-incompatibility detect** — defense against per-option-overflow.
- **Per-mismatch-sport requires PM announcement** — defense against per-rogue-port-hijack.
- **Per-MPJ-ACK HMAC `crypto_memneq`** — defense against per-timing-side-channel.
- **Per-fallback_is_fatal on mp_join** — defense against per-downgrade attack.
- **Per-csum_required match** — defense against per-csum-disable injection.
- **Per-DSS infinite mapping triggers MIB_INFINITEMAPRX and Invalid** — defense against per-middlebox-strip.
- **Per-mptcp_can_accept_new_subflow gate** — defense against per-subflow-flood.
- **Per-allow_infinite_fallback for MP_FAIL** — defense against per-rogue-MP_FAIL.
- **Per-list_add under msk lock** — defense against per-concurrent-conn_list-corruption.
- **Per-fail_out RTO_MAX bounded** — defense against per-stuck-fail.
- **Per-ULP fallback restores TCP ops atomically** — defense against per-half-restored sk_prot UAF.
- **Per-cgroup-attach inherits parent** — defense against per-cgroup-bypass.
- **Per-security_mptcp_add_subflow LSM hook** — defense against per-LSM-bypass.
- **Per-syncookie path rejects mp_capable ∧ mp_join** — defense against per-cookie-confusion.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/mptcp/protocol.c msk-level data plane (covered in `protocol.md` Tier-3 if expanded)
- net/mptcp/options.c MPTCP-option encode/decode (covered separately if expanded)
- net/mptcp/pm.c in-kernel path manager (covered in `pm.md` Tier-3 if expanded)
- net/mptcp/pm_userspace.c mptcp-pm-userspace netlink (covered in `pm-userspace.md` Tier-3 if expanded)
- net/mptcp/crypto.c HMAC-SHA256 / key-SHA primitives (covered in `crypto.md` Tier-3 if expanded)
- net/mptcp/sockopt.c MPTCP_INFO / MPTCP_TCPINFO (covered separately if expanded)
- net/mptcp/token.c token registry (covered separately if expanded)
- Implementation code
