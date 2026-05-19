# Tier-3: net/smc/af_smc.c — AF_SMC protocol family (SMC-R + SMC-D over TCP-handover)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/smc/af_smc.c (~3607 lines)
  - net/smc/smc.h
  - net/smc/smc_clc.c / smc_clc.h (CLC handshake)
  - net/smc/smc_core.c / smc_core.h (link groups)
  - net/smc/smc_ib.c (RoCE / ib_verbs)
  - net/smc/smc_ism.c (ISM / s390 Internal Shared Memory)
  - net/smc/smc_tx.c / smc_rx.c (TX/RX data plane)
  - net/smc/smc_cdc.c (Connection Data Control)
  - net/smc/smc_llc.c (Link Layer Control over RoCE)
  - net/smc/smc_close.c (orderly close)
  - net/smc/smc_pnet.c (PNETID table)
  - net/smc/smc_netlink.c (netlink mgmt)
  - net/smc/smc_sysctl.c (per-netns sysctls)
  - include/net/smc.h
  - include/uapi/linux/smc.h
  - include/uapi/linux/smc_diag.h
-->

## Summary

**SMC (Shared Memory Communications)** is an IBM-originated transport that piggybacks on TCP for connection setup, then "hands over" the data plane to shared-memory transfers — either **SMC-R** (over RoCE, using RDMA verbs + per-link-group QPs) or **SMC-D** (over s390 ISM / `vSOCK`-style internal shared memory, optionally Emulated-ISM). `af_smc.c` is the **`PF_SMC` protocol family** front-end: it implements `socket(AF_SMC, SOCK_STREAM, IPPROTO_SMC|IPPROTO_SMC6)` as a TCP-mimicking AF, allocates a `struct smc_sock` (which wraps `struct sock`) + an internal `clcsock` (a kernel `IPPROTO_TCP` `kernel_socket`) used to carry the CLC (Connection Layer Control) handshake. After CLC negotiation (`smc_connect_clc` / `smc_listen_work`), data is steered through SMC's RX/TX rings into peer **RMB (Remote Memory Buffer)** regions; if any step fails (peer doesn't advertise SMC, no RoCE/ISM device, IPsec in use, MTU mismatch, etc.), the socket transparently **falls back** to its underlying TCP `clcsock` (`smc_switch_to_fallback`). State machine: `SMC_INIT → SMC_ACTIVE / SMC_LISTEN → SMC_PEERCLOSEWAIT1/2 → SMC_APPCLOSEWAIT1/2 → SMC_CLOSED`. Critical for: IBM Z workload acceleration (SMC-D Emulated-ISM in KVM/`vsock`-replacement), RoCE-accelerated TCP-compatible apps, mainframe-tier shared-memory throughput.

This Tier-3 covers `net/smc/af_smc.c` (~3607 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct smc_sock` | per-socket SMC state (embeds `struct sock` first; via `smc_sk()` container_of) | `SmcSock` |
| `struct smc_connection` | per-conn data-plane state (lgr, rmb, sndbuf, CDC cursors) | `SmcConnection` |
| `struct smc_link_group` | per-(peer-systemid, vlan, role) link-group: shared RMBs/QPs across conns | `SmcLinkGroup` |
| `struct smc_link` | per-RoCE link inside lgr (QP, CQs, GID, MAC) | `SmcLink` |
| `struct smc_init_info` | per-handshake scratch (version negotiation, devices, EID) | `SmcInitInfo` |
| `struct smc_clc_msg_proposal` | CLC proposal | `SmcClcProposal` |
| `struct smc_clc_msg_accept_confirm` | CLC accept/confirm | `SmcClcAcceptConfirm` |
| `smc_create()` | per-`socket(2)` create handler | `SmcAf::create` |
| `smc_create_clcsk()` | per-create: alloc kernel TCP `clcsock` | `SmcAf::create_clcsk` |
| `smc_sock_alloc()` | per-sock alloc (`sk_alloc(PF_SMC, ...)`) | `SmcAf::sock_alloc` |
| `smc_sk_init()` | per-sk-state init (SMC_INIT, workqueues, locks) | `SmcAf::sk_init` |
| `smc_destruct()` | per-sk destructor (delegates to inet/inet6) | `SmcAf::destruct` |
| `smc_bind()` | per-`bind(2)` → forwards to `clcsock` | `SmcAf::bind` |
| `smc_connect()` / `__smc_connect()` / `smc_connect_work()` | per-`connect(2)` blocking/non-blocking + CLC + RoCE/ISM connect | `SmcAf::connect` / `connect_inner` / `connect_work` |
| `smc_connect_clc()` | per-CLC proposal+accept exchange (client side) | `SmcAf::connect_clc` |
| `smc_connect_rdma()` | per-SMC-R client conn-create + link-select + buf-create + RTOK + `smcr_clnt_conf_first_link` | `SmcAf::connect_rdma` |
| `smc_connect_ism()` | per-SMC-D client conn-create + buf-attach + `smc_clc_send_confirm` | `SmcAf::connect_ism` |
| `smc_listen()` | per-`listen(2)`: hook `clcsock` callbacks; install `smc_tcp_syn_recv_sock` af_ops | `SmcAf::listen` |
| `smc_accept()` | per-`accept(2)`: wait on `smc_accept_dequeue` | `SmcAf::accept` |
| `smc_listen_work()` | per-accepted-conn worker: CLC proposal recv → device find → CLC accept send → confirm recv | `SmcAf::listen_work` |
| `smc_tcp_listen_work()` | per-listen-sock worker: drain `clcsock` accept queue | `SmcAf::tcp_listen_work` |
| `smc_clcsock_accept()` | per-listen: `kernel_accept(clcsock)` + alloc child smc_sock | `SmcAf::clcsock_accept` |
| `smc_clcsock_data_ready()` | per-clcsock readable: schedule `tcp_listen_work` | `SmcAf::clcsock_data_ready` |
| `smc_listen_find_device()` | per-listen device-pick (ISM v2 → ISM v1 → RoCE v2 → RoCE v1) | `SmcAf::listen_find_device` |
| `smc_listen_decline()` | per-decline + fallback path | `SmcAf::listen_decline` |
| `smc_release()` / `__smc_release()` | per-`close(2)`: active close (CDC) or fallback close | `SmcAf::release` / `release_inner` |
| `smc_getname()` | per-`getsockname`/`getpeername`: delegate to `clcsock` | `SmcAf::getname` |
| `smc_sendmsg()` / `smc_recvmsg()` | per-data-plane I/O (smc_tx/smc_rx or `clcsock` if fallback) | `SmcAf::sendmsg` / `recvmsg` |
| `smc_splice_read()` | per-`splice_read(2)` | `SmcAf::splice_read` |
| `smc_poll()` / `smc_accept_poll()` | per-poll merge of `clcsock` poll + SMC state | `SmcAf::poll` / `accept_poll` |
| `smc_shutdown()` | per-`shutdown(2)` | `SmcAf::shutdown` |
| `smc_setsockopt()` / `__smc_setsockopt()` | per-`setsockopt`: TCP-options proxied to clcsock; `SOL_SMC` handled locally | `SmcAf::setsockopt` |
| `smc_getsockopt()` / `__smc_getsockopt()` | per-`getsockopt` | `SmcAf::getsockopt` |
| `smc_ioctl()` | per-`ioctl(2)` (`SIOCOUTQ`/`SIOCINQ`/`SIOCATMARK`) | `SmcAf::ioctl` |
| `smc_hash_sk()` / `smc_unhash_sk()` | per-bind hash tables (`smc_v4_hashinfo`/`smc_v6_hashinfo`) | `SmcAf::hash_sk` / `unhash_sk` |
| `smc_switch_to_fallback()` | per-handover: install fallback callbacks, mark `use_fallback=true` | `SmcAf::switch_to_fallback` |
| `smc_fback_replace_callbacks()` | per-fallback: rewrite `sk_state_change`/`sk_data_ready`/`sk_write_space`/`sk_error_report` to forwarders | `SmcAf::fback_replace_callbacks` |
| `smc_fback_restore_callbacks()` | per-release: undo above | `SmcAf::fback_restore_callbacks` |
| `smc_connect_fallback()` / `smc_connect_decline_fallback()` | per-connect-time fallback | `SmcAf::connect_fallback` / `connect_decline_fallback` |
| `smc_release_cb()` | per-sock `__sk_flush_backlog` callback | `SmcAf::release_cb` |
| `smcr_clnt_conf_first_link()` | per-SMC-R client: CONFIRM_LINK + ADD_LINK over LLC | `SmcAf::smcr_clnt_conf_first_link` |
| `smcr_serv_conf_first_link()` | per-SMC-R server: respond CONFIRM_LINK / ADD_LINK | `SmcAf::smcr_serv_conf_first_link` |
| `smcr_lgr_reg_sndbufs()` / `smcr_lgr_reg_rmbs()` | per-RoCE: `ib_reg_mr` for snd/rcv buf vmalloc regions | `SmcAf::reg_sndbufs` / `reg_rmbs` |
| `smc_conn_save_peer_info()` / `_fce()` | per-CLC accept: capture peer cursors/role | `SmcAf::conn_save_peer_info` |
| `smc_link_save_peer_info()` | per-link: capture peer QPN/GID/MAC | `SmcAf::link_save_peer_info` |
| `smc_find_proposal_devices()` | per-connect: pick candidate RoCE/ISM v1/v2 devices | `SmcAf::find_proposal_devices` |
| `smc_find_ism_v2_device_clnt()` / `_serv()` | per-CLC v2 ISM device search | `SmcAf::find_ism_v2_device` |
| `smc_find_rdma_device()` | per-PNETID RoCE device lookup | `SmcAf::find_rdma_device` |
| `smc_find_ism_device()` | per-PNETID ISM device lookup | `SmcAf::find_ism_device` |
| `smc_listen_v2_check()` / `smc_listen_prfx_check()` | per-listen CLC validity checks | `SmcAf::listen_v2_check` / `listen_prfx_check` |
| `smc_listen_rdma_init()` / `smc_listen_rdma_reg()` / `smc_listen_rdma_finish()` | per-listen RoCE buf-create + reg + first-link confirm | `SmcAf::listen_rdma_*` |
| `smc_listen_ism_init()` | per-listen ISM buf-create | `SmcAf::listen_ism_init` |
| `smc_accept_enqueue()` / `_dequeue()` / `_unlink()` | per-listen accept-queue | `SmcAf::accept_enqueue` / `dequeue` / `unlink` |
| `smc_close_non_accepted()` | per-listen: discard untaken backlog | `SmcAf::close_non_accepted` |
| `smc_nl_dump_hs_limitation()` / `_enable()` / `_disable()` | per-netlink ctrl: handshake-limitation knob | `SmcAf::nl_hs_limit_*` |
| `smc_hs_congested()` | per-TCP congestion-callback: throttle SMC handshakes | `SmcAf::hs_congested` |
| `smc_tcp_syn_recv_sock()` | per-`icsk_af_ops.syn_recv_sock` override: tag child as SMC-capable | `SmcAf::tcp_syn_recv_sock` |
| `smc_init()` / `smc_exit()` | per-module register/unregister (`PF_SMC`, pernet, IB client, workqueues) | `SmcAf::module_init` / `module_exit` |

## Compatibility contract

REQ-1: `struct smc_sock`:
- First field: `struct sock sk` — `smc_sk(sk)` is `container_of_const(sk, struct smc_sock, sk)`.
- `clcsock: *socket` — internal `kernel_socket` of family `PF_INET`/`PF_INET6`, type `SOCK_STREAM`, proto `IPPROTO_TCP`; carries CLC handshake and (after fallback) raw payload.
- `conn: smc_connection` — data-plane state.
- `listen_smc: *smc_sock` — parent listen sock for accepted children.
- `tcp_listen_work, smc_listen_work, connect_work` — per-sock workers on `smc_tcp_ls_wq` / `smc_hs_wq`.
- `accept_q, accept_q_lock` — list of accepted-but-not-yet-`accept(2)`'d children.
- `clcsock_release_lock` — serializes `clcsock` close vs fallback callbacks.
- `clcsk_data_ready, clcsk_state_change, clcsk_write_space, clcsk_error_report` — saved original `clcsock->sk` callbacks (restored on release).
- `af_ops, ori_af_ops` — overridden ICSK af-ops on `clcsock->sk` to splice in `smc_tcp_syn_recv_sock`.
- `use_fallback: bool` — true once `smc_switch_to_fallback` ran.
- `fallback_rsn: int` — decline-reason code (`SMC_CLC_DECL_*`).
- `limit_smc_hs: u8` — copies `net->smc.limit_smc_hs` at init.
- `connect_nonblock: u8` — connect-in-progress flag for non-blocking connect.
- `sockopt_defer_accept: u32` — `TCP_DEFER_ACCEPT` proxy.
- `queued_smc_hs: atomic_t` — pending handshakes on the listener.

REQ-2: SMC sk state values (`net/smc/smc.h`):
- `SMC_ACTIVE = 1` — established, data-plane up.
- `SMC_INIT = 2` — created, not yet connected.
- `SMC_CLOSED = 7` — fully closed.
- `SMC_LISTEN = 10` — listening.
- Plus `SMC_PEERCLOSEWAIT1/2`, `SMC_APPCLOSEWAIT1/2`, `SMC_PROCESSABORT`, `SMC_PEERABORTWAIT`, etc. (close.c).

REQ-3: Protocol numbers (UAPI `linux/smc.h`):
- `SMCPROTO_SMC = 0` — IPv4 underlay.
- `SMCPROTO_SMC6 = 1` — IPv6 underlay.
- Family: `PF_SMC` (= `AF_SMC` = 43).

REQ-4: `smc_create(net, sock, protocol, kern)`:
- if `sock->type != SOCK_STREAM`: `-ESOCKTNOSUPPORT`.
- if `protocol != SMCPROTO_SMC && protocol != SMCPROTO_SMC6`: `-EPROTONOSUPPORT`.
- `sock->ops = &smc_sock_ops; sock->state = SS_UNCONNECTED`.
- `sk = smc_sock_alloc(net, sock, protocol)` (`sk_alloc(PF_SMC, …, smc_proto/smc_proto6, 0)` then `sock_init_data` then `smc_sk_init`).
- `smc_create_clcsk(net, sk, family)` — `sock_create_kern(net, PF_INET|PF_INET6, SOCK_STREAM, IPPROTO_TCP, &smc->clcsock)`; `sk_net_refcnt_upgrade(smc->clcsock->sk)`.
- on failure: `sk_common_release(sk)`.

REQ-5: `smc_sk_init(net, sk, protocol)`:
- `sk->sk_state = SMC_INIT`.
- `sk->sk_destruct = smc_destruct`.
- `sk->sk_protocol = protocol`.
- `sndbuf = 2 * net->smc.sysctl_wmem; rcvbuf = 2 * net->smc.sysctl_rmem`.
- Initialize workers: `tcp_listen_work = smc_tcp_listen_work; connect_work = smc_connect_work; conn.tx_work (delayed) = smc_tx_work`.
- `INIT_LIST_HEAD(&smc->accept_q); spin_lock_init(&smc->accept_q_lock); spin_lock_init(&smc->conn.send_lock); sock_lock_init_class_and_name; sk_prot->hash(sk); mutex_init(&smc->clcsock_release_lock); smc_init_saved_callbacks; use_fallback = false; fallback_rsn = 0; smc_close_init(smc)`.

REQ-6: `smc_bind(sock, uaddr, addr_len)`:
- Validate `sockaddr_in` (`AF_INET`/`AF_INET6`/`AF_UNSPEC` with `INADDR_ANY`); else `-EINVAL`/`-EAFNOSUPPORT`.
- `lock_sock(sk)`.
- if `sk->sk_state != SMC_INIT || smc->connect_nonblock`: `-EINVAL`.
- copy `sk_reuse`, `sk_reuseport` to `clcsock->sk`.
- `kernel_bind(smc->clcsock, uaddr, addr_len)` — bind delegates to internal TCP.

REQ-7: `smc_connect(sock, addr, alen, flags)`:
- validate addr family `AF_INET`/`AF_INET6`.
- `lock_sock(sk)`.
- per `sock->state`:
  - `SS_CONNECTED`: `-EISCONN` (`SMC_ACTIVE`) or `-EINVAL`.
  - `SS_CONNECTING`: if `SMC_ACTIVE` go to `connected`.
  - `SS_UNCONNECTED`: transition to `SS_CONNECTING`.
- per `sk->sk_state`:
  - `SMC_CLOSED`: `sock_error(sk) ?: -ECONNABORTED`.
  - `SMC_ACTIVE`: `-EISCONN`.
  - `SMC_INIT`: continue.
- `smc_copy_sock_settings_to_clc(smc)`.
- `tcp_sk(smc->clcsock->sk)->syn_smc = 1` — advertise SMC capability in TCP options.
- if `smc->connect_nonblock`: `-EALREADY`.
- `kernel_connect(smc->clcsock, addr, alen, flags)`; if `rc && rc != -EINPROGRESS`: bail.
- if `smc->use_fallback`: leave as TCP-connect.
- `sock_hold(&smc->sk)`.
- if `O_NONBLOCK`: `queue_work(smc_hs_wq, &smc->connect_work); smc->connect_nonblock = 1; return -EINPROGRESS`.
- else: `__smc_connect(smc)`.

REQ-8: `__smc_connect(smc)`:
1. `version = smc_ism_is_v2_capable() ? SMC_V2 : SMC_V1`.
2. if `smc->use_fallback`: `return smc_connect_fallback(smc, smc->fallback_rsn)`.
3. if `!tcp_sk(smc->clcsock->sk)->syn_smc`: peer did not advertise SMC → `smc_connect_fallback(smc, SMC_CLC_DECL_PEERNOSMC)`.
4. if `using_ipsec(smc)`: `smc_connect_decline_fallback(smc, SMC_CLC_DECL_IPSEC, version)`.
5. `ini = kzalloc(struct smc_init_info)`.
6. `ini->smcd_version = SMC_V1 | SMC_V2; ini->smcr_version = SMC_V1 | SMC_V2`; `smc_type_v1 = SMC_TYPE_B; smc_type_v2 = SMC_TYPE_B`.
7. `smc_vlan_by_tcpsk(clcsock, ini)` — vlan disambiguation.
8. `smc_find_proposal_devices(smc, ini)`.
9. `buf = kzalloc(SMC_CLC_MAX_ACCEPT_LEN)`; `aclc = (smc_clc_msg_accept_confirm *)buf`.
10. `smc_connect_clc(smc, aclc, ini)` — send Proposal, wait for Accept.
11. `smc_connect_check_aclc(ini, aclc)` — verify mode/version match.
12. dispatch:
    - `aclc->hdr.typev1 == SMC_TYPE_R`: `smc_connect_rdma(smc, aclc, ini)`.
    - `aclc->hdr.typev1 == SMC_TYPE_D`: `smc_connect_ism(smc, aclc, ini)`.
13. on success: `SMC_STAT_CLNT_SUCC_INC; smc_connect_ism_vlan_cleanup; kfree(buf); kfree(ini)`.
14. any failure post-CLC: `smc_connect_decline_fallback`.

REQ-9: `smc_connect_rdma(smc, aclc, ini)`:
- `ini->is_smcd = false; ini->ib_clcqpn = ntoh24(aclc->r0.qpn); ini->first_contact_peer = aclc->hdr.typev2 & SMC_FIRST_CONTACT_MASK; memcpy peer_systemid/peer_gid/peer_mac; max_conns/max_links from constants`.
- `smc_connect_rdma_v2_prepare(smc, aclc, ini)`.
- `mutex_lock(&smc_client_lgr_pending); smc_conn_create(smc, ini)` — finds-or-creates link-group; sets `smc->conn.lgr`, `smc->conn.lnk`.
- `smc_conn_save_peer_info(smc, aclc)`.
- if `!first_contact_local`: pick matching `link` from `lgr.lnk[]` by peer QPN/GID/MAC; if none → `SMC_CLC_DECL_NOSRVLINK`; `smc_switch_link_and_count`.
- `smc_buf_create(smc, false)` — allocate `sndbuf_desc` and `rmb_desc`.
- if `first_contact_local`: `smc_link_save_peer_info; smc_rmb_rtoken_handling` (verify peer's RToken).
- `smc_rx_init(smc)`.
- if first contact local: `smc_ib_ready_link(link)`.
- else: if `sndbuf is_vm`: `smcr_lgr_reg_sndbufs(link, sndbuf_desc)`; always `smcr_lgr_reg_rmbs(link, rmb_desc)` (`ib_reg_mr`).
- if `version > SMC_V1`: `eid = aclc->r1.eid; smc_fill_gid_list(link->lgr, &ini->smcrv2.gidlist, link->smcibdev, link->gid)`.
- `smc_clc_send_confirm(smc, first_contact_local, version, eid, ini)`.
- `smc_tx_init(smc)`.
- if first contact local: `smc_llc_flow_initiate(lgr, SMC_LLC_FLOW_ADD_LINK); smcr_clnt_conf_first_link(smc); smc_llc_flow_stop`.
- `mutex_unlock(&smc_client_lgr_pending)`.
- `smc->sk.sk_state = SMC_ACTIVE` (if was `SMC_INIT`).

REQ-10: `smc_connect_ism(smc, aclc, ini)`:
- `ini->is_smcd = true; ini->first_contact_peer = …`.
- if `version == SMC_V2`:
  - if `first_contact_peer`: validate v2x features via `smc_clc_clnt_v2x_features_validate(fce, ini)`.
  - `smc_v2_determine_accepted_chid(aclc, ini)` — match CHID from CLC ACCEPT.
  - if `__smc_ism_is_emulated`: `ism_peer_gid[selected].gid_ext = ntohll(aclc->d1.gid_ext)`.
- `ism_peer_gid[selected].gid = ntohll(aclc->d0.gid)`.
- `mutex_lock(&smc_server_lgr_pending)`; `smc_conn_create(smc, ini)`.
- `smc_buf_create(smc, true)` — SMC-D shared-DMB buffers.
- `smc_conn_save_peer_info`.
- if `smc_ism_support_dmb_nocopy(lgr.smcd)`: `smcd_buf_attach(smc)`.
- `smc_rx_init; smc_tx_init`.
- `smc_clc_send_confirm(smc, first_contact_local, version, eid, ini)`.
- `mutex_unlock(&smc_server_lgr_pending)`.
- `sk_state = SMC_ACTIVE`.

REQ-11: `smc_listen(sock, backlog)`:
- `lock_sock(sk)`.
- if `sk_state ∉ {SMC_INIT, SMC_LISTEN} || connect_nonblock || sock->state != SS_UNCONNECTED`: `-EINVAL`.
- if already `SMC_LISTEN`: just update `sk_max_ack_backlog`.
- `smc_copy_sock_settings_to_clc(smc)`.
- if `!use_fallback`: `tcp_sk(clcsock->sk)->syn_smc = 1`.
- `write_lock_bh(&clcsock->sk->sk_callback_lock); __rcu_assign_sk_user_data_with_flags(clcsock->sk, smc, SK_USER_DATA_NOCOPY); smc_clcsock_replace_cb(&clcsock->sk->sk_data_ready, smc_clcsock_data_ready, &smc->clcsk_data_ready); write_unlock_bh`.
- Save `ori_af_ops = inet_csk(clcsock->sk)->icsk_af_ops`; build `smc->af_ops = *ori_af_ops; af_ops.syn_recv_sock = smc_tcp_syn_recv_sock; inet_csk(clcsock->sk)->icsk_af_ops = &smc->af_ops`.
- if `limit_smc_hs`: `tcp_sk(clcsock->sk)->smc_hs_congested = smc_hs_congested`.
- `kernel_listen(clcsock, backlog)`.
- on err: restore data_ready, clear `sk_user_data`, return err.
- `sock_set_flag(sk, SOCK_RCU_FREE); sk->sk_max_ack_backlog = backlog; sk->sk_ack_backlog = 0; sk->sk_state = SMC_LISTEN`.

REQ-12: `smc_tcp_listen_work(work)`:
- `lock_sock(lsk)`.
- while `lsk->sk_state == SMC_LISTEN`:
  - `smc_clcsock_accept(lsmc, &new_smc)` — `kernel_accept(lsmc->clcsock, ...)` + alloc child `new_smc`.
  - if no new conn: break.
  - if `tcp_sk(new_smc->clcsock->sk)->syn_smc`: `atomic_inc(&lsmc->queued_smc_hs)`.
  - `new_smc->listen_smc = lsmc; use_fallback = lsmc->use_fallback; fallback_rsn = lsmc->fallback_rsn`.
  - `sock_hold(lsk)`; `INIT_WORK(&new_smc->smc_listen_work, smc_listen_work); smc_copy_sock_settings_to_smc(new_smc); sock_hold(&new_smc->sk); queue_work(smc_hs_wq, &new_smc->smc_listen_work)`.

REQ-13: `smc_listen_work(work)`:
1. `lock_sock(&new_smc->sk)` (released by `smc_listen_out_*`).
2. if listener no longer `SMC_LISTEN`: `smc_listen_out_err`.
3. if `new_smc->use_fallback`: `smc_listen_out_connected` (TCP-mode).
4. if `!tcp_sk(newclcsock->sk)->syn_smc`: `smc_switch_to_fallback(new_smc, SMC_CLC_DECL_PEERNOSMC)`; out_connected.
5. allocate proposal buf (`smc_clc_msg_proposal_area`).
6. `smc_clc_wait_msg(new_smc, pclc, ..., SMC_CLC_PROPOSAL, CLC_WAIT_TIME)`.
7. if peer hdr version > V1: `proposal_version = SMC_V2`.
8. if `using_ipsec`: `SMC_CLC_DECL_IPSEC`.
9. `smc_listen_v2_check(new_smc, pclc, ini); smc_clc_srv_v2x_features_validate(new_smc, pclc, ini)`.
10. `mutex_lock(&smc_server_lgr_pending); smc_rx_init; smc_tx_init`.
11. `smc_listen_find_device(new_smc, pclc, ini)` — picks ISM v2 / v1 / RoCE v2 / v1.
12. `smc_clc_send_accept(new_smc, first_contact_local, accept_version, negotiated_eid, ini)`.
13. for SMC-D: `mutex_unlock(&smc_server_lgr_pending)` early (no LLC).
14. `smc_clc_wait_msg(new_smc, cclc, ..., SMC_CLC_CONFIRM, CLC_WAIT_TIME)`.
15. `smc_clc_v2x_features_confirm_check(cclc, ini)`.
16. `smc_conn_save_peer_info_fce(new_smc, cclc)`.
17. for SMC-R: `smc_listen_rdma_finish(new_smc, cclc, first_contact_local, ini); mutex_unlock(&smc_server_lgr_pending)`.
18. `smc_conn_save_peer_info(new_smc, cclc)`.
19. if SMC-D + `smc_ism_support_dmb_nocopy`: `smcd_buf_attach(new_smc)`.
20. `smc_listen_out_connected(new_smc)`.

REQ-14: `smc_accept(sock, new_sock, arg)`:
- `lock_sock(sk)`; if not `SMC_LISTEN`: `-EINVAL`.
- `add_wait_queue_exclusive(sk_sleep(sk), &wait)`.
- loop until `smc_accept_dequeue(sk, new_sock)` returns child or timeout.
- on `signal_pending`: `sock_intr_errno`.
- after dequeue: if `sockopt_defer_accept && !O_NONBLOCK`: wait for first byte (`smc_rx_wait` or fallback-clcsock `sk_wait_data`).

REQ-15: `smc_switch_to_fallback(smc, reason_code)`:
- `mutex_lock(&smc->clcsock_release_lock)`.
- if `!smc->clcsock`: `-EBADF`.
- `smc->use_fallback = true; smc->fallback_rsn = reason_code; smc_stat_fallback(smc); trace_smc_switch_to_fallback`.
- if `smc->sk.sk_socket && smc->sk.sk_socket->file`:
  - `smc->clcsock->file = smc->sk.sk_socket->file; clcsock->file->private_data = smc->clcsock` — splice fd → clcsock.
  - merge `wq.fasync_list` from smc-sock into clcsock.
  - `smc_fback_replace_callbacks(smc)` — install forwarding callbacks so polls on the SMC sk see clcsock state.
- `mutex_unlock`.

REQ-16: `smc_fback_replace_callbacks(smc)`:
- save `clcsock->sk` callbacks (`sk_state_change`, `sk_data_ready`, `sk_write_space`, `sk_error_report`).
- install forwarders that call `smc_fback_state_change` / `_data_ready` / `_write_space` / `_error_report`, each of which wakes `smc->sk` waitqueues.

REQ-17: `smc_release(sock)`:
- `sock_hold(sk)`.
- if `connect_nonblock && old_state == SMC_INIT`: `tcp_abort(clcsock->sk, ECONNABORTED)`.
- `cancel_work_sync(&connect_work)` (with sock_put on success).
- `lock_sock` (nested for listen-children).
- if `old_state == SMC_INIT && sk_state == SMC_ACTIVE && !use_fallback`: `smc_close_active_abort` (race during connect_work).
- `__smc_release(smc)`:
  - if `!use_fallback`: `smc_close_active(smc); sock_dead + shutdown=ALL`.
  - else: handle `SMC_LISTEN` (`kernel_sock_shutdown(clcsock, SHUT_RDWR)`); `smc_restore_fallback_changes`.
  - `sk->sk_prot->unhash(sk)`.
  - if reached `SMC_CLOSED`: `smc_clcsock_release(smc)` (drop kernel-socket); `if !use_fallback: smc_conn_free(&smc->conn)`.
- `sock_orphan(sk); sock->sk = NULL; release_sock; sock_put; sock_put`.

REQ-18: `smc_sendmsg(sock, msg, len)`:
- `lock_sock(sk)`.
- if `MSG_FASTOPEN`: only allowed via fallback transition pre-`SMC_ACTIVE`; else `-EINVAL`.
- if `sk_state != SMC_ACTIVE` and not in `SMC_APPCLOSEWAIT1`: `-EPIPE`.
- if `use_fallback`: `smc->clcsock->ops->sendmsg(clcsock, msg, len)`.
- else: `smc_tx_sendmsg(smc, msg, len)`.

REQ-19: `smc_recvmsg(sock, msg, len, flags)`:
- if `use_fallback`: `smc->clcsock->ops->recvmsg(clcsock, …)`.
- else: `smc_rx_recvmsg(smc, msg, NULL, len, flags)`.

REQ-20: `smc_poll(file, sock, wait)`:
- merge `tcp_poll(clcsock)` with SMC-state-derived bits (`EPOLLIN` based on `bytes_to_rcv`, `EPOLLOUT` based on tx ring availability, `EPOLLHUP`/`EPOLLERR` from state).

REQ-21: `smc_shutdown(sock, how)`:
- if `use_fallback`: forward to clcsock.
- else: per-half-close transitions in `smc_close.c` (CDC `closing` flag), wake peer.

REQ-22: `smc_setsockopt(sock, level, optname, optval, optlen)`:
- if `level == SOL_TCP && optname == TCP_ULP`: `-EOPNOTSUPP`.
- if `level == SOL_SMC`: `__smc_setsockopt` (own opts: `SMCPROTO_SMC` and family options).
- else: forward to `clcsock->ops->setsockopt` under `clcsock_release_lock`.
- post-forward: per-`optname`:
  - `TCP_FASTOPEN*`: not supported; if pre-connect, switch to fallback `SMC_CLC_DECL_OPTUNSUPP`; else `-EINVAL`.
  - `TCP_NODELAY` / `TCP_CORK`: trigger tx-flush state change in `smc->conn`.
  - `TCP_DEFER_ACCEPT`: capture in `smc->sockopt_defer_accept`.

REQ-23: `smc_hash_sk(sk)` / `smc_unhash_sk(sk)`:
- Choose `smc_v4_hashinfo` or `smc_v6_hashinfo` per `sk_family`.
- `spin_lock_bh(hashinfo.lock); sk_add_node(sk, &hashinfo.ht); sock_set_flag(sk, SOCK_RCU_FREE); spin_unlock_bh`.

REQ-24: `smc_tcp_syn_recv_sock(sk, skb, …)`:
- Wraps `ori_af_ops->syn_recv_sock`; sets `tcp_sk(child)->syn_smc` based on whether SMC option was in SYN.
- Replaces `clcsock->sk->sk_data_ready` on the listening sock with `smc_clcsock_data_ready` to drive `smc_tcp_listen_work` from per-listener context.

REQ-25: `smc_init()`:
1. `register_pernet_subsys(&smc_net_ops)` (`smc_net_init` calls `smc_sysctl_net_init` + `smc_pnet_net_init`).
2. `register_pernet_subsys(&smc_net_stat_ops)`.
3. `smc_ism_init()`.
4. `smc_clc_init()`.
5. `smc_nl_init()` — netlink family.
6. `smc_pnet_init()` — pnet (PNETID ↔ device) table.
7. Allocate workqueues: `smc_tcp_ls_wq`, `smc_hs_wq`, `smc_close_wq` (all `WQ_PERCPU`).
8. `smc_core_init()` — link-group infrastructure.
9. `smc_llc_init()`; `smc_cdc_init()`.
10. `proto_register(&smc_proto, 1); proto_register(&smc_proto6, 1)`.
11. `sock_register(&smc_sock_family_ops)` — `{.family=PF_SMC, .owner=THIS_MODULE, .create=smc_create}`.
12. `INIT_HLIST_HEAD(&smc_v4_hashinfo.ht); &smc_v6_hashinfo.ht`.
13. `smc_ib_register_client()` — register as `ib_client` for RoCE.
14. `smc_inet_init()` — `IPPROTO_SMC` ULP / inet integration.
15. `bpf_smc_hs_ctrl_init()` — BPF hook for HS limitation.
16. `static_branch_enable(&tcp_have_smc)` — enable TCP fast-path to recognize SMC option.

REQ-26: `smc_exit()`:
- Reverse: disable static branch; `sock_unregister(PF_SMC)`; tear down workqueues, IB client, proto, pernet, clc/ism, etc.; `rcu_barrier()`.

REQ-27: Fallback determinism — any of:
- Peer SYN lacks SMC TCP option (`SMC_CLC_DECL_PEERNOSMC`).
- IPsec on path (`SMC_CLC_DECL_IPSEC`).
- No PNETID-matching RoCE/ISM device (`SMC_CLC_DECL_NOSMCRDEV` / `_NOSMCDDEV`).
- CLC malformed / version unsupported (`SMC_CLC_DECL_VERSMISMAT` / `_MODEUNSUPP`).
- Buffer/RMB allocation failure (`SMC_CLC_DECL_MEM` / `_MAX_DMB`).
- LLC ADD_LINK / RToken failure (`SMC_CLC_DECL_ERR_RDYLNK` / `_ERR_RTOK` / `_ERR_REGBUF`).
- `setsockopt(TCP_FASTOPEN*)` pre-connect (`SMC_CLC_DECL_OPTUNSUPP`).

REQ-28: CLC handshake message types (`smc_clc.h`): `SMC_CLC_PROPOSAL`, `SMC_CLC_ACCEPT`, `SMC_CLC_CONFIRM`, `SMC_CLC_DECLINE`. Version field: `SMC_V1` (1) or `SMC_V2` (2).

REQ-29: Per-netns sysctls (`smc_sysctl.c`):
- `wmem` / `rmem` (initial SO_SNDBUF/RCVBUF).
- `limit_handshakes` (`net->smc.limit_smc_hs` knob).
- Auto-corking, smcr/smcd-version, max-conns, max-links.

## Acceptance Criteria

- [ ] AC-1: `socket(AF_SMC, SOCK_STREAM, SMCPROTO_SMC)` returns an fd; `sk_state == SMC_INIT`; `smc->clcsock` is a `PF_INET`/`SOCK_STREAM`/`IPPROTO_TCP` kernel socket.
- [ ] AC-2: `bind` forwards address to `smc->clcsock`; `sk_state` must be `SMC_INIT` and `connect_nonblock == 0`.
- [ ] AC-3: `connect`: when peer does not advertise `tcp_sk->syn_smc`, fallback path enters; `smc->use_fallback == true`; `fallback_rsn == SMC_CLC_DECL_PEERNOSMC`.
- [ ] AC-4: `connect` with IPsec on path returns SMC-decline fallback (`SMC_CLC_DECL_IPSEC`).
- [ ] AC-5: `connect` with RoCE-matched PNETID device: full SMC-R path; final state `SMC_ACTIVE`; `conn.lgr != NULL`; `conn.lnk != NULL`.
- [ ] AC-6: `connect` with ISM device: SMC-D path; `conn.lgr.smcd != NULL`.
- [ ] AC-7: `listen` installs `smc_clcsock_data_ready` on `clcsock->sk`, overrides `icsk_af_ops` with `smc->af_ops` whose `syn_recv_sock = smc_tcp_syn_recv_sock`.
- [ ] AC-8: `smc_tcp_listen_work` drains `kernel_accept(clcsock)` and schedules `smc_listen_work` per child; bounded by `smc_hs_wq` throughput.
- [ ] AC-9: `smc_listen_work` for a non-SMC peer (no `syn_smc` on child) transitions to fallback (`SMC_CLC_DECL_PEERNOSMC`).
- [ ] AC-10: `smc_listen_work` rejects malformed CLC Proposal → `smc_listen_decline` → fallback.
- [ ] AC-11: `accept` waits on the SMC accept-queue (`smc_accept_dequeue`); honors `O_NONBLOCK` and `sockopt_defer_accept`.
- [ ] AC-12: `sendmsg`/`recvmsg` in fallback mode forwards to `clcsock->ops->{sendmsg, recvmsg}`; in SMC mode goes through `smc_tx_sendmsg`/`smc_rx_recvmsg`.
- [ ] AC-13: `release`: SMC mode closes via CDC (`smc_close_active`) then `smc_conn_free`; fallback closes clcsock; both reach `SMC_CLOSED` and unhash.
- [ ] AC-14: `setsockopt(SOL_TCP, TCP_ULP, …)` returns `-EOPNOTSUPP`.
- [ ] AC-15: `setsockopt(.., TCP_FASTOPEN, ...)` pre-`SMC_ACTIVE` switches to fallback with `SMC_CLC_DECL_OPTUNSUPP`; in `SMC_ACTIVE` returns `-EINVAL`.

## Architecture

```
struct SmcSock {                  // first field: struct Sock
  sk:                Sock,
  clcsock:           Option<*mut Socket>,
  conn:              SmcConnection,
  listen_smc:        Option<*mut SmcSock>,

  tcp_listen_work:   WorkStruct,  // smc_tcp_listen_work
  smc_listen_work:   WorkStruct,  // smc_listen_work
  connect_work:      WorkStruct,  // smc_connect_work

  accept_q:          ListHead,
  accept_q_lock:     SpinLock,

  clcsock_release_lock: Mutex,
  clcsk_data_ready:  Option<DataReadyFn>,
  clcsk_state_change: Option<StateChangeFn>,
  clcsk_write_space: Option<WriteSpaceFn>,
  clcsk_error_report: Option<ErrorReportFn>,

  af_ops:            InetConnSockAfOps,
  ori_af_ops:        Option<*const InetConnSockAfOps>,

  use_fallback:      bool,
  fallback_rsn:      i32,        // SMC_CLC_DECL_*
  limit_smc_hs:      u8,
  connect_nonblock:  u8,
  sockopt_defer_accept: u32,
  queued_smc_hs:     AtomicI32,
}

struct SmcConnection {
  lgr:               Option<*mut SmcLinkGroup>,
  lnk:               Option<*mut SmcLink>,        // SMC-R only
  sndbuf_desc:       Option<*mut SmcBufDesc>,
  rmb_desc:          Option<*mut SmcBufDesc>,
  tx_work:           DelayedWork,                  // smc_tx_work
  send_lock:         SpinLock,
  bytes_to_rcv:      AtomicI32,
  // CDC cursors (sndbuf_seq, peer_rmb_seq) ...
}
```

`SmcAf::create(net, sock, protocol)`:
1. Reject `sock.type != SOCK_STREAM` → `-ESOCKTNOSUPPORT`.
2. Reject `protocol ∉ {SMCPROTO_SMC, SMCPROTO_SMC6}` → `-EPROTONOSUPPORT`.
3. `sock.ops = smc_sock_ops; sock.state = SS_UNCONNECTED`.
4. `sk = sock_alloc(net, sock, protocol)`; init via `sk_init`.
5. `create_clcsk(net, sk, if SMCPROTO_SMC { PF_INET } else { PF_INET6 })`.
6. on failure: `sk_common_release(sk); sock.sk = None`.

`SmcAf::connect_inner(smc)`:
1. `version = if SmcIsm::is_v2_capable() { SMC_V2 } else { SMC_V1 }`.
2. `if smc.use_fallback { return connect_fallback(smc, smc.fallback_rsn) }`.
3. `if !tcp_sk(clcsock.sk).syn_smc { return connect_fallback(smc, DECL_PEERNOSMC) }`.
4. `if using_ipsec(smc) { return connect_decline_fallback(smc, DECL_IPSEC, version) }`.
5. `ini = SmcInitInfo::new_zeroed()`; set v1|v2 for both R and D.
6. `vlan_by_tcpsk(clcsock, &mut ini)`.
7. `find_proposal_devices(smc, &mut ini)?`.
8. allocate accept-buf, run `connect_clc(smc, &mut aclc, &mut ini)`.
9. `connect_check_aclc(&ini, &aclc)?`.
10. dispatch by `aclc.hdr.typev1`:
    - `SMC_TYPE_R` → `connect_rdma(smc, &aclc, &mut ini)`.
    - `SMC_TYPE_D` → `connect_ism(smc, &aclc, &mut ini)`.
11. on failure post-CLC: `connect_decline_fallback`.

`SmcAf::listen(sock, backlog)`:
1. Validate state.
2. `tcp_sk(clcsock.sk).syn_smc = 1` (unless already fallback).
3. Save `clcsock.sk.sk_data_ready` → `smc.clcsk_data_ready`; install `smc_clcsock_data_ready`.
4. Save `ori_af_ops = inet_csk(clcsock.sk).icsk_af_ops`; build `af_ops = *ori_af_ops; af_ops.syn_recv_sock = smc_tcp_syn_recv_sock; inet_csk(clcsock.sk).icsk_af_ops = &af_ops`.
5. If `limit_smc_hs`: `tcp_sk(clcsock.sk).smc_hs_congested = smc_hs_congested`.
6. `kernel_listen(clcsock, backlog)`; on error restore step-3 callbacks.
7. `sk.sk_state = SMC_LISTEN`.

`SmcAf::listen_work(new_smc)`:
1. Lock new_smc.
2. fallback-shortcut paths.
3. Receive `Proposal` CLC; version-check; v2x features.
4. Lock `smc_server_lgr_pending`; `rx_init/tx_init`.
5. `listen_find_device(new_smc, pclc, ini)` (ISM v2 → ISM v1 → RoCE v2 → RoCE v1).
6. Send `Accept` CLC.
7. For SMC-D early-unlock; receive `Confirm` CLC.
8. `conn_save_peer_info_fce`; for SMC-R `listen_rdma_finish` then unlock; `conn_save_peer_info`.
9. SMC-D dmb-nocopy: `smcd_buf_attach`.
10. `listen_out_connected(new_smc)` (move to listener's accept-queue, wake listener).

`SmcAf::switch_to_fallback(smc, reason_code)`:
1. Lock `clcsock_release_lock`; bail if `clcsock` gone.
2. `smc.use_fallback = true; smc.fallback_rsn = reason_code; stat_fallback(smc); trace_switch_to_fallback`.
3. If user-fd attached: splice `smc.sk.sk_socket.file → clcsock.file`, transfer `fasync_list`, `fback_replace_callbacks`.
4. Unlock.

`SmcAf::release(sock)`:
1. `sock_hold`.
2. cancel pending `connect_work`; tcp_abort if non-blocking connect in flight.
3. `lock_sock(sk)` (nested for `SMC_LISTEN`).
4. `__smc_release(smc)`:
   - `!use_fallback`: `smc_close_active(smc); sock_dead + shutdown=ALL`.
   - else: shutdown(SHUT_RDWR) listener clcsock; `restore_fallback_changes(smc)`.
   - unhash.
   - If `SMC_CLOSED`: `smc_clcsock_release(smc)` (drop kernel socket); `smc_conn_free(&smc.conn)`.
5. `sock_orphan; sock.sk = None; release_sock; sock_put; sock_put`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `create_only_stream_smc_proto` | INVARIANT | per-`smc_create`: `sock.type == SOCK_STREAM ∧ proto ∈ {SMC, SMC6}`. |
| `connect_state_init_only` | INVARIANT | per-`smc_connect`: enters via `sk_state == SMC_INIT`. |
| `bind_state_init_only` | INVARIANT | per-`smc_bind`: rejects unless `sk_state == SMC_INIT && !connect_nonblock`. |
| `listen_callback_set_under_lock` | INVARIANT | per-`smc_listen`: `clcsock.sk.sk_callback_lock` write-held when replacing `sk_data_ready`. |
| `fallback_serializes_with_release` | INVARIANT | per-`switch_to_fallback`: `clcsock_release_lock` held when reading `smc.clcsock`. |
| `accept_q_locked` | INVARIANT | per-accept-queue mutations: `accept_q_lock` held. |
| `clcsock_kernel_socket` | INVARIANT | per-`create_clcsk`: returns a `sock_create_kern` socket; `sk_net_refcnt_upgrade` called. |
| `sk_state_terminal_closed` | INVARIANT | per-`release`: post-`__smc_release` reaches `SMC_CLOSED`. |
| `fallback_use_implies_clcsock` | INVARIANT | per-`use_fallback == true`: `smc.clcsock != NULL` and ops delegate. |
| `connect_work_canceled_on_release` | INVARIANT | per-`smc_release`: `cancel_work_sync(&connect_work)` before tear-down. |

### Layer 2: TLA+

`net/smc-af-fsm.tla`:
- Per-SMC state machine: `{SMC_INIT, SMC_LISTEN, SMC_ACTIVE, SMC_APPCLOSEWAIT1, SMC_APPCLOSEWAIT2, SMC_PEERCLOSEWAIT1, SMC_PEERCLOSEWAIT2, SMC_PROCESSABORT, SMC_PEERABORTWAIT, SMC_CLOSED}` + `use_fallback` flag.
- Properties:
  - `safety_init_to_active_via_handshake` — per-conn: `SMC_INIT → SMC_ACTIVE` only after CLC accept+confirm.
  - `safety_fallback_or_smc_path` — exactly one of `(use_fallback) ↔ (conn.lgr != NULL)`.
  - `safety_no_smc_after_fallback` — once `use_fallback == true`, no further SMC-mode I/O occurs (sendmsg/recvmsg route to clcsock).
  - `safety_listen_drains_clcsock_q` — every `kernel_accept` on `clcsock` is consumed by `smc_tcp_listen_work`.
  - `safety_release_unhashes` — per-`release`: `sk_prot.unhash(sk)` called.
  - `liveness_connect_terminates` — per-`__smc_connect`: returns 0, decline-fallback, or error in finite steps.
  - `liveness_listen_work_progresses` — per-`smc_listen_work`: terminates with `out_connected` or `decline`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SmcAf::create` post: `sock.sk != NULL ∧ smc.clcsock != NULL ∧ sk_state == SMC_INIT` | `SmcAf::create` |
| `SmcAf::bind` post: `kernel_bind(clcsock, addr, len)` invoked; `sk_state` unchanged | `SmcAf::bind` |
| `SmcAf::connect_inner` post: `sk_state ∈ {SMC_ACTIVE, SMC_CLOSED}` ∨ `use_fallback == true` | `SmcAf::connect_inner` |
| `SmcAf::connect_rdma` post: success ⟹ `conn.lgr != NULL ∧ conn.lnk != NULL ∧ rmb_desc registered (ib_reg_mr)` | `SmcAf::connect_rdma` |
| `SmcAf::connect_ism` post: success ⟹ `conn.lgr.smcd != NULL ∧ rmb_desc bound to DMB` | `SmcAf::connect_ism` |
| `SmcAf::listen` post: `sk_state == SMC_LISTEN ∧ ori_af_ops captured ∧ smc_clcsock_data_ready installed` | `SmcAf::listen` |
| `SmcAf::listen_work` post: child reaches `SMC_ACTIVE` (in either SMC or fallback mode) ∨ is freed via `listen_out_err` | `SmcAf::listen_work` |
| `SmcAf::switch_to_fallback` post: `use_fallback == true ∧ clcsock callbacks rewritten to forwarders` | `SmcAf::switch_to_fallback` |
| `SmcAf::release` post: `sk_state == SMC_CLOSED ∧ sock.sk == NULL ∧ refcount → 0 reached` | `SmcAf::release` |
| `SmcAf::sendmsg` post: `use_fallback ⟹ data sent via clcsock; else via smc_tx` | `SmcAf::sendmsg` |

### Layer 4: Verus/Creusot functional

`Per-conn: socket → bind → connect (CLC propose ↔ accept) → device-find → buf-create → CLC confirm → SMC_ACTIVE → sendmsg/recvmsg (RDMA-write into peer RMB or ISM-dmb-copy/attach) → shutdown → close → SMC_CLOSED` semantic equivalence with upstream against `Documentation/networking/smc-sysctl.rst`, IBM SMC-R protocol specification, and `Documentation/networking/smc.rst`. Per-CLC version negotiation rules (V1 vs V2, fallback determinism on PEERNOSMC/IPSEC/etc.), per-LinkGroup role (`SMC_LGR_CLIENT` vs `SMC_LGR_SERVER`) and per-RoCE/ISM device selection ordering must match upstream behavior on a sample workload (`af_smc` self-test net+test).

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

AF_SMC reinforcement:

- **Per-`SOCK_STREAM` only** — defense against per-datagram/raw misuse of `PF_SMC`.
- **Per-CLC decline → TCP fallback** — defense against per-handshake-failure stuck conn (transparent degradation).
- **Per-IPsec opt-out** — defense against per-IPsec-on-path subverting SMC's plaintext shared-memory channel.
- **Per-PNETID device matching** — defense against per-wrong-RoCE/ISM device picked for tenant traffic.
- **Per-`tcp_sk->syn_smc` capability** — defense against per-peer-doesn't-support pretending; falls back instead of stalling.
- **Per-`use_fallback` once-set monotonic** — defense against per-toggling back to SMC mid-stream (would tear data plane).
- **Per-`clcsock_release_lock` mutex** — defense against per-close vs fallback-callback race.
- **Per-`SOCK_RCU_FREE` flag on listener** — defense against per-`sk_data_ready` running on freed sock.
- **Per-`smc_v4_hashinfo`/`smc_v6_hashinfo` lockdep + RCU** — defense against per-cross-AF lookup confusion.
- **Per-`smc_hs_congested` rate-limit** — defense against per-SMC-handshake DoS at listener.
- **Per-`limit_smc_hs` per-netns admin knob** — defense against per-tenant-handshake exhaustion.
- **Per-`smc_listen_decline` always emits CLC DECLINE before fallback** — defense against per-peer being left in `WAIT_ACCEPT` state.
- **Per-`smcr_lgr_reg_rmbs` reg-MR scoped to link group** — defense against per-MR-leak / cross-LGR access.
- **Per-CDC cursor monotonic checks** — defense against per-cursor-replay (in `smc_cdc.c`, referenced here).
- **Per-`smc_close_active_abort` on connect-race** — defense against per-`connect_work` finishing after `release` started.

## Grsecurity/PaX-style Reinforcement

Baseline PaX/grsecurity mitigations applicable to PF_SMC (SMC-R / SMC-D):

- **PAX_USERCOPY** — SMC sockopts (`SMC_*`) and CLC-handshake user buffers traverse whitelisted `copy_{to,from}_user`.
- **PAX_KERNEXEC** — `smc_rx` / `smc_tx` and CLC negotiation execute from W^X .text.
- **PAX_RANDKSTACK** — per-syscall stack randomization frustrates inference of RMB / DMB offset state.
- **PAX_REFCOUNT** — `smc_sock`, `smc_link_group`, `smc_link` refcounts saturating; LGR-create storms cannot wrap.
- **PAX_MEMORY_SANITIZE** — RMB / DMB-mapped buffers, freed `smc_connection`, and CLC handshake payload sanitized on release.
- **PAX_UDEREF** — `setsockopt(SOL_SMC, ...)` arg parsing traverses user pointers under UDEREF.
- **PAX_RAP / kCFI** — `smc_proto`, `smc_proto_ops` `static const`; CDC / LLC fn-ptr dispatch CFI-checked.
- **GRKERNSEC_HIDESYM** — `smc_listen_decline`, `smcr_lgr_reg_rmbs`, `smc_clc_send_decline` hidden from unprivileged kallsyms.
- **GRKERNSEC_DMESG** — CLC-DECLINE / handshake-failure warnings CAP_SYSLOG-gated.

AF_SMC-specific reinforcement:

- **PF_SMC CAP_NET_ADMIN for ULP install** — defense against unprivileged installation of SMC ULP on TCP sockets (parallels kTLS `TCP_ULP` gating).
- **RoCE/ISM hardware-attach CAP_SYS_ADMIN** — defense against unprivileged manipulation of RDMA/ISM device-link in the LGR; raw RoCE QP/MR programming requires elevated privilege.
- **RMB / DMB PAX_MEMORY_SANITIZE on detach** — defense against shared-memory residue between tenants on RMB release.
- **`smc_proto` PAX_RAP-typed** — protocol-ops dispatch CFI-checked across CLC fallback transitions.
- **GRKERNSEC_HIDESYM on `smcr_lgr_reg_rmbs`** — RDMA MR-registration symbol hidden from unprivileged probing.

Rationale: SMC's threat model is multi-tenant RDMA / shared-memory leakage and CLC-handshake DoS; MEMORY_SANITIZE on RMB/DMB on release, CAP_SYS_ADMIN gating on hardware-link operations, and CFI on the protocol-ops vtable bound the cross-tenant data-leak and unauthorized-fallback surfaces.

## Open Questions

- Whether SMC-D's Emulated-ISM (`__smc_ism_is_emulated`) on non-s390 hosts (KVM/virt) shares the same Tier-3 or merits a separate one — orthogonal to af_smc.c.
- Detailed BPF hook surface (`bpf_smc_hs_ctrl_init`, `smc_hs_bpf.h`) — covered separately if expanded.

## Out of Scope

- `net/smc/smc_clc.c` CLC handshake parsing/marshaling (separate Tier-3 if expanded)
- `net/smc/smc_core.c` link-group lifecycle / link-add / link-confirm (separate Tier-3 if expanded)
- `net/smc/smc_ib.c` RoCE / ib_verbs glue (separate Tier-3 if expanded)
- `net/smc/smc_ism.c` ISM / s390 driver glue (separate Tier-3 if expanded)
- `net/smc/smc_tx.c` / `smc_rx.c` data-plane RX/TX (separate Tier-3 if expanded)
- `net/smc/smc_cdc.c` Connection Data Control (cursors) (separate Tier-3 if expanded)
- `net/smc/smc_llc.c` Link Layer Control over RoCE (separate Tier-3 if expanded)
- `net/smc/smc_close.c` orderly close FSM (referenced; not enumerated)
- `net/smc/smc_pnet.c` PNETID table (separate Tier-3 if expanded)
- `net/smc/smc_diag.c` `SOCK_DIAG`/sock-stat introspection
- `net/smc/smc_netlink.c` netlink mgmt beyond the three hs-limitation handlers
- Implementation code
