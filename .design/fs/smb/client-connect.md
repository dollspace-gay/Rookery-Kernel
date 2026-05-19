# Tier-3: fs/smb/client/connect.c — SMB client TCP/SMB-session/tree-connect + reconnect + DFS + multichannel

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/smb/client/connect.c (~4532 lines) [moved from fs/cifs/connect.c]
  - fs/smb/client/cifsglob.h (struct TCP_Server_Info, struct cifs_ses, struct cifs_tcon, securityEnum, smb_version_operations)
  - fs/smb/client/smb2pdu.c (SMB2 negotiate + session_setup + tree_connect XDR)
  - fs/smb/client/smb1ops.c, smb2ops.c, smb20pdu.c, smb21ops.c, smb30ops.c (per-dialect smb_version_operations)
  - fs/smb/client/fs_context.c (struct smb3_fs_context parsing)
  - fs/smb/client/sess.c (channel binding, smb3_update_ses_channels)
  - fs/smb/client/dfs_cache.c, dfs.c (DFS retargeting under CONFIG_CIFS_DFS_UPCALL)
  - fs/smb/client/transport.c (mid_q_entry lifecycle, cifs_send / cifs_read_from_socket)
  - fs/smb/client/smbdirect.c (RDMA transport)
-->

## Summary

The SMB client (formerly `fs/cifs/`, moved to `fs/smb/client/` upstream) manages per-server TCP/RDMA transports, per-user SMB sessions, and per-share tree-connections. Per-`cifs_mount()`: build `struct smb3_fs_context` -> `cifs_get_tcp_session()` (transport) -> `cifs_get_smb_ses()` (auth) -> `cifs_get_tcon()` (share) -> `mount_setup_tlink()`. Per-transport: `struct TCP_Server_Info` carries the socket, the per-server `smb_version_operations` (dialect dispatcher), credit accounting, in-flight MIDs, sequence number, server capabilities, and a dedicated `cifsd` kernel thread running `cifs_demultiplex_thread()`. Per-`generic_ip_connect()`: dual-stack `AF_INET`/`AF_INET6` kernel socket created in the mount's net-ns, bound, optionally `TCP_NODELAY`'d, then `kernel_connect()`'d; `ip_connect()` wraps with port-445-first-then-139 fallback. Per-dialect negotiation: `cifs_negotiate_protocol()` calls `ops->negotiate` (SMB1/SMB2.x/SMB3.x/SMB3.1.1 selectable by `vers=` mount option), driving the `CifsNew -> CifsNeedNegotiate -> CifsInNegotiate -> CifsGood` state machine. Per-session setup: `cifs_setup_session()` runs `ops->sess_setup` (NTLMv2/Kerberos/RawNTLMSSP) producing per-channel signing keys; SMB3.0+ encrypts when negotiated. Per-tree-connect: `cifs_tree_connect()` runs `ops->tree_connect`, transitioning `TID_NEW -> TID_IN_TCON -> TID_GOOD`. Per-`cifs_demultiplex_thread()`: per-connection RX kthread reading RFC1002 framing, parsing MID, dispatching to per-request `mid_q_entry` callback; on transport failure issues `cifs_reconnect()` which marks `CifsNeedReconnect`, aborts in-flight MIDs (`cifs_abort_connection`), and either re-resolves the hostname or walks the DFS target list (`reconnect_dfs_server`). Per-multichannel: SMB3.0+ interface-list queried by `smb2_query_server_interfaces`; secondary channels created via `smb3_update_ses_channels` -> `cifs_get_tcp_session(ctx, primary_server)`. Per-SMB3.11 POSIX extensions: negotiated capability promoting POSIX path handling, mode bits, and reparse semantics on the per-tcon. Critical for: connection multiplexing, transparent reconnect, DFS namespace failover, multichannel performance, dialect agility.

This Tier-3 covers `fs/smb/client/connect.c` (~4532 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct TCP_Server_Info` | per-transport (socket + state + cifsd thread + credits) | `TcpServerInfo` |
| `struct cifs_ses` | per-user SMB session (signing keys, channels, security) | `CifsSes` |
| `struct cifs_tcon` | per-share tree-connect | `CifsTcon` |
| `struct smb3_fs_context` | per-mount parsed options | `Smb3FsContext` |
| `struct cifs_mount_ctx` | per-mount transient | `CifsMountCtx` |
| `struct mchan_mount` | per-multichannel setup work | `MchanMount` |
| `cifs_get_tcp_session()` | per-allocate-or-reuse-transport | `Connect::get_tcp_session` |
| `cifs_find_tcp_session()` | per-lookup-existing | `Connect::find_tcp_session` |
| `cifs_put_tcp_session()` | per-release-transport | `Connect::put_tcp_session` |
| `cifs_get_smb_ses()` | per-allocate-or-reuse-ses | `Connect::get_smb_ses` |
| `__cifs_put_smb_ses()` | per-release-ses | `Connect::put_smb_ses` |
| `cifs_setup_ipc()` | per-IPC$ tcon (admin pipe) | `Connect::setup_ipc` |
| `cifs_get_tcon()` | per-allocate-or-reuse-tcon | `Connect::get_tcon` |
| `cifs_put_tcon()` | per-release-tcon | `Connect::put_tcon` |
| `cifs_tree_connect()` | per-(re)tree-connect | `Connect::tree_connect` |
| `cifs_negotiate_protocol()` | per-dialect-negotiation | `Connect::negotiate_protocol` |
| `cifs_setup_session()` | per-session-setup (auth) | `Connect::setup_session` |
| `ip_connect()` / `generic_ip_connect()` / `ip_rfc1001_connect()` | per-TCP-connect + port fallback + RFC1001 NetBIOS | `Connect::ip_connect` / `_generic` / `_rfc1001` |
| `bind_socket()` | per-srcaddr bind | `Connect::bind_socket` |
| `cifs_demultiplex_thread()` | per-connection RX kthread | `Connect::demultiplex_thread` |
| `cifs_handle_standard()` | per-MID-resolve + signature-verify | `Connect::handle_standard` |
| `standard_receive3()` | per-PDU body read | `Connect::standard_receive3` |
| `cifs_reconnect()` / `_cifs_reconnect()` / `__cifs_reconnect()` | per-transport-reconnect | `Connect::reconnect_*` |
| `reconnect_dfs_server()` | per-DFS-target-failover | `Connect::reconnect_dfs_server` |
| `reconnect_target_locked()` / `__reconnect_target_locked()` | per-DFS-tgt-iter | `Connect::reconnect_target_*` |
| `cifs_abort_connection()` | per-shutdown + MID-retry-queue | `Connect::abort_connection` |
| `cifs_echo_request()` | per-keepalive (delayed_work) | `Connect::echo_request` |
| `server_unresponsive()` | per-echo-timeout detection | `Connect::server_unresponsive` |
| `reconn_set_ipaddr_from_hostname()` | per-DNS-re-resolve | `Connect::reconn_set_ipaddr_from_hostname` |
| `smb2_query_server_interfaces()` | per-multichannel-iface-poll | `Connect::query_server_interfaces` |
| `mchan_mount_alloc()` / `_work_fn` | per-mount multichannel deferred setup | `Connect::mchan_mount_*` |
| `cifs_mount_get_session()` / `_get_tcon()` | per-mount stage assembly | `Connect::mount_get_session` / `_tcon` |
| `cifs_mount()` | per-mount top-level | `Connect::mount` |
| `cifs_umount()` | per-unmount top-level | `Connect::umount` |
| `cifs_match_super()` / `match_server()` / `match_session()` / `match_tcon()` | per-sb-reuse decision | `Connect::match_*` |
| `is_smb_response()` / `allocate_buffers()` / `dequeue_mid()` | per-RX framing helpers | `Connect::is_smb_response` / `allocate_buffers` / `dequeue_mid` |
| `cifs_enable_signing()` | per-signing-policy decision | `Connect::enable_signing` |

## Compatibility contract

REQ-1: struct TCP_Server_Info (subset):
- hostname (kstrdup of ctx->server_hostname).
- leaf_fullpath (DFS).
- ops (smb_version_operations dialect dispatcher).
- vals (smb_version_values: protocol_id, dialect, version_string).
- ssocket (struct socket *).
- srcaddr, dstaddr (sockaddr_storage).
- tcpStatus ∈ { CifsNew, CifsInNegotiate, CifsNeedNegotiate, CifsGood, CifsNeedReconnect, CifsExiting }.
- conn_id (atomic_inc of tcpSesNextId).
- srv_count, smb_ses_list, tcp_ses_list (∈ cifs_tcp_ses_list global).
- in_flight, max_in_flight, credits, max_credits.
- sequence_number, channel_sequence_num, reconnect_instance.
- client_guid (16 bytes, generate_random_uuid unless ctx->use_client_guid).
- response_q, request_q (waitqueues), pending_mid_q.
- echo (delayed_work, cifs_echo_request), reconnect (delayed_work, smb2_reconnect_server).
- srv_mutex (cifs_server_lock), srv_lock, req_lock, mid_queue_lock.
- tsk (cifsd kthread_run, "cifsd").
- noblockcnt, noblocksnd, noautotune, tcp_nodelay, rdma, ignore_signature.
- workstation_RFC1001_name, server_RFC1001_name, rfc1001_sessinit, with_rfc1001.
- dialect (negotiated, e.g., SMB311_PROT_ID).
- capabilities (server-side bitmask).
- sign (enforced signing), sec_mode (server-side security mode).
- compression.requested.
- echo_interval (ctx->echo_interval * HZ).
- min_offload, retrans.
- smbd_conn (struct smbd_connection *, when rdma).
- primary_server (NULL for primary, non-NULL for secondary channels: SERVER_IS_CHAN()).
- dns_dom (DNS domain, strscpy from ctx->dns_dom).
- dfs_conn (DFS-originated).
- nr_targets (DFS target count, drives reconnect timeout).

REQ-2: cifs_get_tcp_session(ctx, primary_server):
- /* Reuse path */
- tcp_ses = cifs_find_tcp_session(ctx) — match by srcaddr, dstaddr/port, hostname, srv_count++, returns existing.
- if tcp_ses: return.
- /* New */
- tcp_ses = kzalloc; hostname = kstrdup(ctx->server_hostname); leaf_fullpath = kstrdup(ctx->leaf_fullpath).
- ops = ctx->ops; vals = ctx->vals (dialect dispatcher per `vers=`).
- cifs_set_net_ns(tcp_ses, get_net(current->nsproxy->net_ns)) — bind transport to current netns.
- sign = ctx->sign; conn_id = atomic_inc_return(&tcpSesNextId).
- noblockcnt = ctx->rootfs; noblocksnd = ctx->noblocksnd || ctx->rootfs; tcp_nodelay = ctx->sockopt_tcp_nodelay; rdma = ctx->rdma.
- credits = 1 (initial).
- if primary_server: ++primary_server->srv_count; tcp_ses->primary_server = primary_server (chan).
- Init waitqueues, mutexes, lists, delayed_work (echo + reconnect).
- workstation_RFC1001_name / server_RFC1001_name = ctx->source_/target_rfc1001_name.
- rfc1001_sessinit = ctx->rfc1001_sessinit (auto/on/off).
- echo_interval = ctx->echo_interval * HZ.
- generate_random_uuid(client_guid) unless ctx->use_client_guid.
- tcpStatus = CifsNew; ++srv_count.
- if rdma: tcp_ses->smbd_conn = smbd_get_connection(tcp_ses, dstaddr); else: rc = ip_connect(tcp_ses).
- tcp_ses->tsk = kthread_run(cifs_demultiplex_thread, tcp_ses, "cifsd").
- tcp_ses->tcpStatus = CifsNeedNegotiate.
- max_credits = clamp(ctx->max_credits, 20, 60000) else SMB2_MAX_CREDITS_AVAILABLE.
- nr_targets = 1.
- list_add(&tcp_ses->tcp_ses_list, &cifs_tcp_ses_list) under cifs_tcp_ses_lock.
- queue_delayed_work(cifsiod_wq, &echo, echo_interval).
- return tcp_ses; or unwind on error (sock_release, kfree).

REQ-3: cifs_get_smb_ses(server, ctx):
- /* Reuse */
- ses = cifs_find_smb_ses(server, ctx) — match (server, sectype, user, domain, multichannel, dfs_root_ses).
- if ses ∧ cifs_chan_needs_reconnect(ses, server):
  - mutex_lock(&ses->session_mutex).
  - cifs_negotiate_protocol(xid, ses, server) — REQ-9.
  - cifs_setup_session(xid, ses, server, ctx->local_nls) — REQ-10.
  - On EACCES/EKEYEXPIRED/EKEYREVOKED ∧ password2: swap(password, password2); retry once.
  - mutex_unlock; cifs_put_tcp_session(server, 0) — extra ref returned.
- /* New */
- ses = sesInfoAlloc; ses->server = server.
- ses->ip_addr = formatted dstaddr (AF_INET6 "%pI6" or AF_INET "%pI4").
- ses->user_name / password / password2 / domainName = kstrdup(ctx->*) (sensitive).
- workstation_name = strscpy(ctx->workstation_name).
- ses->cred_uid = ctx->cred_uid; linux_uid = ctx->linux_uid.
- unicode = ctx->unicode; sectype = ctx->sectype; sign = ctx->sign.
- upcall_target ∈ { UPTARGET_APP (default), UPTARGET_MOUNT }.
- local_nls = load_nls(ctx->local_nls->charset).
- chans[0].server = server; chan_count = 1; chan_max = ctx->multichannel ? ctx->max_channels : 1; chans_need_reconnect = 1.
- cifs_negotiate_protocol -> cifs_setup_session (loop with password2 retry).
- memcpy chans[0].signkey = ses->smb3signingkey.
- list_add(&ses->smb_ses_list, &server->smb_ses_list).
- ipc = cifs_setup_ipc(ses, ctx->seal) — IPC$ tcon for admin RPC (DFS, query-info).
- ses->tcon_ipc = !IS_ERR(ipc) ? ipc : NULL.

REQ-4: cifs_get_tcon(ses, ctx):
- /* Reuse */
- tcon = cifs_find_tcon(ses, ctx); if tcon: cifs_put_smb_ses(ses) (drop caller's extra ref); return.
- /* New */
- nohandlecache = (server->dialect >= SMB20_PROT_ID ∧ caps & DIRECTORY_LEASING) ? ctx->nohandlecache || !dir_cache_timeout : true.
- tcon = tcon_info_alloc(!nohandlecache).
- snapshot_time / handle_timeout — SMB2.0+ required.
- ses linked; password kstrdup.
- seal: SMB3+ required; tcon->seal = true iff server caps & SMB2_GLOBAL_CAP_ENCRYPTION.
- linux_ext: SMB3.11 POSIX extensions:
  - if server->posix_ext_supported: tcon->posix_extensions = true (pr_warn_once experimental).
  - else if SMB311 / SMB3ANY / SMBDEFAULT: -EOPNOTSUPP.
  - else if SMB10: cap_unix(ses) -> SMB1-Unix-Ext path; else -EOPNOTSUPP.
- cifs_tree_connect(xid, tcon) — REQ-7.

REQ-5: cifs_setup_ipc(ses, seal):
- ipc = tcon_info_alloc.
- tree_name = "\\<server>\IPC$".
- tcon->ses = ses; ipc-specific flags.
- ops->tree_connect (admin pipe).
- Used for DFS referrals + Get-DFS-Referral SRV_COPYCHUNK + named-pipe RPC.

REQ-6: generic_ip_connect(server) (dual-stack):
- saddr = &server->dstaddr.
- AF_INET6: sport = ipv6->sin6_port; slen = sizeof(sockaddr_in6); sfamily = AF_INET6.
- AF_INET: sport = ipv4->sin_port; slen = sizeof(sockaddr_in); sfamily = AF_INET.
- if server->ssocket already exists: reuse; else sock_create_kern(cifs_net_ns(server), sfamily, SOCK_STREAM, IPPROTO_TCP, &server->ssocket).
- sk_net_refcnt_upgrade(sk).
- sk->sk_allocation = GFP_NOFS (reclaim-safe).
- sk->sk_use_task_frag = false.
- cifs_reclassify_socket{4,6} (lockdep class).
- bind_socket(server) — if ctx->srcaddr set, kernel_bind to srcaddr.
- sk->sk_rcvtimeo = 7 * HZ; sk_sndtimeo = 5 * HZ.
- if server->noautotune: ensure sndbuf >= 200 KiB, rcvbuf >= 140 KiB.
- if server->tcp_nodelay: tcp_sock_set_nodelay.
- rc = kernel_connect(socket, saddr, slen, server->noblockcnt ? O_NONBLOCK : 0).
- noblockcnt + -EINPROGRESS: treated as success (rootfs fast-path).
- on error: sock_release; ssocket = NULL; return rc.
- if with_rfc1001 ∨ rfc1001_sessinit == 1 ∨ (== -1 ∧ sport == htons(RFC1001_PORT)): ip_rfc1001_connect (NetBIOS session init "RFC1002 SESSION REQUEST").

REQ-7: ip_connect(server) (port-fallback):
- sport = &dst sin/sin6 port.
- if *sport == 0: try 445 (CIFS_PORT) first; on failure try 139 (RFC1001_PORT).
- Else: pass through to generic_ip_connect.

REQ-8: bind_socket(server):
- if srcaddr.ss_family != AF_UNSPEC: kernel_bind(ssocket, srcaddr, srcaddr_len).

REQ-9: cifs_negotiate_protocol(xid, ses, server):
- if !ops->need_neg ∨ !ops->negotiate: -ENOSYS.
- retry: srv_lock; if tcpStatus ∉ { CifsGood, CifsNew, CifsNeedNegotiate }: -EHOSTDOWN.
- if !ops->need_neg(server) ∧ tcpStatus == CifsGood: 0 (idempotent).
- tcpStatus = CifsInNegotiate; neg_start = jiffies; srv_unlock.
- rc = ops->negotiate(xid, ses, server) — sends NEGOTIATE PDU, parses NEGOTIATE_RESPONSE; per-dialect:
  - SMB1: SMB_COM_NEGOTIATE listing dialect strings up to "NT LM 0.12".
  - SMB2.x/3.x: SMB2_NEGOTIATE with DialectCount + Dialects[]; server picks one.
  - SMB3.1.1: NEGOTIATE_CONTEXT (PreauthIntegrity, Encryption, Compression, NetName, RDMA).
- -EAGAIN allowed one retry (in_retry guard).
- On success: tcpStatus = CifsGood.
- On error: tcpStatus = CifsNeedNegotiate.

REQ-10: cifs_setup_session(xid, ses, server, nls_info):
- pserver = SERVER_IS_CHAN(server) ? server->primary_server : server.
- if ses_status ∉ { SES_GOOD, SES_NEW, SES_NEED_RECON }: -EHOSTDOWN.
- if CIFS_ALL_CHANS_GOOD(ses): SES_NEED_RECON -> SES_GOOD; return 0.
- cifs_chan_set_in_reconnect; is_binding = !CIFS_ALL_CHANS_NEED_RECONNECT.
- if !is_binding: ses_status = SES_IN_SETUP; iface_last_update = 0 (force iface refresh).
- if server == pserver: format ses->ip_addr from primary dstaddr.
- if !is_binding:
  - ses->capabilities = server->capabilities; mask off cap_unix if !linuxExtEnabled.
  - Check unicode capability per ses->unicode option.
  - Free previous auth_key.response.
- rc = ops->sess_setup(xid, ses, server, nls_info) — SMB1/SMB2 SESSION_SETUP, possibly multiple roundtrips for NTLMSSP / SPNEGO.
- On success: ses_status = SES_GOOD; cifs_chan_clear_in_reconnect; clear_need_reconnect.
- On failure: SES_NEED_RECON.

REQ-11: cifs_tree_connect(xid, tcon):
- /* TID state machine */
- if tcon->need_reconnect: tcon->status = TID_NEED_TCON.
- if tcon->status == TID_GOOD: 0.
- if tcon->status ∉ { TID_NEW, TID_NEED_TCON }: -EHOSTDOWN.
- tcon->status = TID_IN_TCON.
- rc = ops->tree_connect(xid, tcon->ses, tcon->tree_name, tcon, ses->local_nls).
- On success: TID_GOOD; need_reconnect = false.
- On failure: TID_NEED_TCON.

REQ-12: cifs_demultiplex_thread(p):
- noreclaim_flag = memalloc_noreclaim_save (RX must succeed under memory pressure).
- allow_kernel_signal(SIGKILL) (clean shutdown path).
- while (server->tcpStatus != CifsExiting):
  - try_to_freeze.
  - allocate_buffers(server) — allocate smallbuf + (lazy) bigbuf from cifs_req_poolp.
  - buf = server->smallbuf; pdu_length = 4.
  - length = cifs_read_from_socket(server, buf, pdu_length) — RFC1002 header.
  - pdu_length = be32_to_cpup(buf) & 0xffffff.
  - if !is_smb_response(server, buf[0]): continue (skip session-keepalive / negative-session-response).
  - next_pdu: server->pdu_size = pdu_length.
  - if pdu_size < MID_HEADER_SIZE(server): cifs_reconnect(server, true).
  - cifs_read_from_socket(server, buf, MID_HEADER_SIZE) — header.
  - server->ops->next_header (chained compound responses).
  - if ops->is_transform_hdr ∧ ops->receive_transform ∧ ops->is_transform_hdr(buf): receive_transform (decrypt SMB3-encrypted PDU into mids/bufs/num_mids).
  - else: mids[0] = ops->find_mid(server, buf); mids[0]->response_pdu_len = pdu_length; standard_receive3 ∨ mids[0]->receive.
  - if ops->is_status_io_timeout(buf): num_io_timeout++; if > MAX_STATUS_IO_TIMEOUT: pending_reconnect = true.
  - server->lstrp = jiffies (last received timestamp).
  - For each mid: mid->resp_buf_size = pdu_size; if ops->is_network_name_deleted: server-dbg "Share deleted"; if !multiRsp ∨ multiEnd: mid_execute_callback (wake sender); release_mid.
  - Else (no mid match): if ops->is_oplock_break: smb2_add_credits_from_hdr; log oplock; else log "No task to wake" + dump.
  - if pdu_length > pdu_size: reallocate; goto next_pdu (chained PDU stream).
  - if pending_reconnect: cifs_reconnect(server, true).
- Cleanup: cifs_buf_release(bigbuf); cifs_small_buf_release(smallbuf).
- task_to_wake = xchg(&server->tsk, NULL).
- clean_demultiplex_info(server) — drain MIDs, release-mid via callbacks.
- if !task_to_wake: wait for signal then exit (parent gone).
- memalloc_noreclaim_restore; module_put_and_kthread_exit(0).

REQ-13: cifs_reconnect / _cifs_reconnect / __cifs_reconnect (no-DFS path):
- cifs_tcp_ses_needs_reconnect(server, nr_targets=1) — sets tcpStatus = CifsNeedReconnect.
- if mark_smb_session: cifs_signal_cifsd_for_reconnect(server, true).
- cifs_mark_tcp_ses_conns_for_reconnect(server, mark_smb_session) — mark all ses/tcon for re-setup.
- cifs_abort_connection — shutdown socket, free session_key, walk pending_mid_q moving to retry_list with MID_RETRY_NEEDED, issue mid callbacks.
- do {
  - try_to_freeze; cifs_server_lock.
  - reconn_set_ipaddr_from_hostname (DNS re-resolve) unless SERVER_IS_CHAN.
  - rc = rdma ? smbd_reconnect : generic_ip_connect.
  - on rc: msleep(3000); if once: break.
  - else: atomic_inc(&tcpSesReconnectCount); set_credits(server, 1); tcpStatus = CifsNeedNegotiate; cifs_swn_reset_server_dstaddr; cifs_queue_server_reconn.
- } while tcpStatus == CifsNeedReconnect.
- mod_delayed_work(cifsiod_wq, &echo, 0) — kick echo.
- wake_up(&server->response_q).

REQ-14: reconnect_dfs_server (CONFIG_CIFS_DFS_UPCALL):
- ref_path = server->leaf_fullpath + 1.
- dfs_cache_noreq_find(ref_path, NULL, &tl) -> num_targets.
- cifs_tcp_ses_needs_reconnect(server, num_targets).
- cifs_mark_tcp_ses_conns_for_reconnect(server, true) — unconditional (may failover to different server/share).
- cifs_abort_connection.
- do {
  - cifs_server_lock; rc = reconnect_target_locked(server, &tl, &target_hint) — walks dfs_cache_get_tgt_iterator; for each: extract_hostname, kfree old, set server->hostname, reconn_set_ipaddr_from_hostname, generic_ip_connect.
  - if rc: unlock; msleep(3000); continue.
  - on success: tcpStatus = CifsNeedNegotiate; cifs_queue_server_reconn.
- } while tcpStatus == CifsNeedReconnect.
- dfs_cache_noreq_update_tgthint(ref_path, target_hint).

REQ-15: cifs_abort_connection(server):
- cifs_server_lock.
- if ssocket: kernel_sock_shutdown(SHUT_WR); sock_release; ssocket = NULL.
- elif cifs_rdma_enabled: smbd_destroy(server).
- sequence_number = 0; session_estab = false; kfree_sensitive(session_key.response); session_key reset.
- Walk pending_mid_q: smb_get_mid; if MID_REQUEST_SUBMITTED -> MID_RETRY_NEEDED; list_move to retry_list; deleted_from_q = true.
- For each retry: mid_execute_callback (wake sender with retry semantics); release_mid.

REQ-16: cifs_echo_request (delayed_work):
- If server->ops->need_echo(server): ops->echo(server) — SMB1 ECHO ∨ SMB2 ECHO PDU.
- Re-queue at echo_interval (default 60s, ctx->echo_interval).
- On no-reply via server_unresponsive: cifs_reconnect.

REQ-17: server_unresponsive(server):
- if (echoes_in_flight ≥ SMB_ECHO_INTERVAL_MAX) ∧ (jiffies - lstrp > 2 * server->echo_interval + 5 * HZ): true (force-reconnect).

REQ-18: reconn_set_ipaddr_from_hostname(server):
- if !server->hostname[0]: return -ENOENT.
- dns_resolve_server_name_to_ip(hostname, &dst_addr, NULL) — kernel DNS upcall (request_key user_helper).
- spin_lock(srv_lock); memcpy server->dstaddr; spin_unlock.

REQ-19: cifs_setup_cifs_sb(cifs_sb):
- INIT_DELAYED_WORK(&cifs_sb->prune_tlinks, cifs_prune_tlinks).
- INIT_LIST_HEAD(&cifs_sb->tcon_sb_link); tlink_tree = RB_ROOT.
- local_nls = ctx->iocharset ? load_nls : load_nls_default.
- ctx->local_nls = cifs_sb->local_nls.
- sbflags = smb3_update_mnt_flags(cifs_sb).
- direct_io / cache_ro / cache_rw flags.
- prepath kstrdup -> CIFS_MOUNT_USE_PREFIX_PATH.
- atomic_set(&cifs_sb->mnt_cifs_flags, sbflags).

REQ-20: cifs_mount_get_session(mnt_ctx):
- server = cifs_get_tcp_session(ctx, NULL).
- ses = cifs_get_smb_ses(server, ctx).
- if ctx->persistent ∧ !(ses->server->capabilities & SMB2_GLOBAL_CAP_PERSISTENT_HANDLES): -EOPNOTSUPP.
- mnt_ctx->{xid, server, ses, tcon=NULL}.

REQ-21: cifs_mount_get_tcon(mnt_ctx):
- tcon = cifs_get_tcon(ses, ctx).
- if tcon->posix_extensions: atomic_or(CIFS_MOUNT_POSIX_PATHS); atomic_andnot(MAP_SFM_CHR | MAP_SPECIAL_CHR).
- cifs_negotiate_iosize(server, ctx, tcon) — pick wsize/rsize from server limits.
- if sbflags & CIFS_MOUNT_FSCACHE: cifs_fscache_get_super_cookie(tcon).

REQ-22: cifs_mount (CONFIG_CIFS_DFS_UPCALL):
- dfs_mount_share(&mnt_ctx).
- if ctx->multichannel: mchan_mount = mchan_mount_alloc(mnt_ctx.ses).
- if ctx->dfs_conn: cifs_autodisable_serverino; force CIFS_MOUNT_USE_PREFIX_PATH; cifs_sb->prepath = ctx->prepath.
- mount_setup_tlink(cifs_sb, ses, tcon).
- if ctx->multichannel: queue_work(cifsiod_wq, &mchan_mount->work) — mchan_mount_work_fn calls smb3_update_ses_channels.

REQ-23: cifs_mount (!CONFIG_CIFS_DFS_UPCALL):
- cifs_mount_get_session.
- cifs_mount_get_tcon.
- WARN_ON server/ses/tcon NULL.
- cifs_is_path_remote — detect DFS root mount when DFS disabled; -EOPNOTSUPP if -EREMOTE.
- ctx->multichannel: mchan_mount_alloc + queue_work.
- mount_setup_tlink.

REQ-24: mount_setup_tlink:
- tlink = kzalloc; tl_uid = ses->linux_uid; tl_tcon = tcon; tl_time = jiffies.
- TCON_LINK_MASTER, TCON_LINK_IN_TREE flags.
- master_tlink = tlink.
- tlink_rb_insert(&cifs_sb->tlink_tree, tlink) under tlink_tree_lock.
- list_add(&cifs_sb->tcon_sb_link, &tcon->cifs_sb_list).
- queue_delayed_work(cifsiod_wq, &cifs_sb->prune_tlinks, TLINK_IDLE_EXPIRE).

REQ-25: smb2_query_server_interfaces (work):
- container_of(work, struct TCP_Server_Info, interface_lease_work.work) ∨ ses-keyed variant.
- ops->query_server_interfaces — SMB2 IOCTL FSCTL_QUERY_NETWORK_INTERFACE_INFO.
- Parses NETWORK_INTERFACE_INFO entries -> cifs_server_iface[]: IfIndex, Capability (RSS_CAPABLE / RDMA_CAPABLE), LinkSpeed, sockaddr.
- iface_last_update = jiffies; iface_count.
- Drives multichannel placement decisions in smb3_update_ses_channels.

REQ-26: Multichannel:
- ctx->multichannel + ctx->max_channels.
- ses->chan_max = max_channels (per REQ-3).
- mchan_mount_alloc holds a ref on ses; mchan_mount_work_fn invokes smb3_update_ses_channels(ses, ses->server, false, false) deferring channel-add to cifsiod_wq.
- smb3_update_ses_channels creates secondary TCP_Server_Info via cifs_get_tcp_session(ctx, primary_server=ses->server), bound to a chosen network interface; each channel gets its own signing key (chans[i].signkey).
- SES_NEED_RECON triggers cifs_chan_set_in_reconnect / clear_need_reconnect per channel.
- iface_last_update = 0 forces re-query at next setup_session.

REQ-27: posix_extensions:
- Negotiated as NEGOTIATE_CONTEXT in SMB3.1.1 NEGOTIATE; server sets posix_ext_supported.
- On per-tcon, ctx->linux_ext + server->posix_ext_supported -> tcon->posix_extensions = true.
- Effect (REQ-21): atomic_or(CIFS_MOUNT_POSIX_PATHS); clear SFM_CHR / SPECIAL_CHR remapping (no '/' / ':' substitution).
- Per-tcon path handling: preserve forward slashes and special characters end-to-end; mode bits exchanged in SMB2 CREATE.

REQ-28: Cleanup:
- cifs_put_tcp_session: --srv_count; if 0: spin_lock(&cifs_tcp_ses_lock); list_del; spin_unlock; cancel echo + reconnect; kthread_stop(tsk); sock_release/smbd_destroy; kfree.
- __cifs_put_smb_ses: --ses_count; if 0: list_del; cifs_setup_session/cifs_negotiate -> nope, just unlink + sesInfoFree (kfree_sensitive of password / auth keys).
- cifs_put_tcon: tear down handle-cache; ops->tree_disconnect; tconInfoFree.
- cifs_umount: cancel prune_tlinks; walk tlink_tree, drop refs; kfree prepath; call_rcu(&cifs_sb->rcu, delayed_free).

REQ-29: Locking discipline:
- cifs_tcp_ses_lock — global, guards cifs_tcp_ses_list.
- server->srv_lock — guards tcpStatus, dstaddr changes.
- server->_srv_mutex (cifs_server_lock) — guards socket + reconnect serialization.
- server->mid_queue_lock — guards pending_mid_q.
- ses->ses_lock — guards ses_status.
- ses->chan_lock — guards chans[] / channel bitmaps.
- ses->session_mutex — serializes negotiate + sess_setup.
- ses->iface_lock — guards iface_list.
- tcon->tc_lock — guards tcon->status.

## Acceptance Criteria

- [ ] AC-1: cifs_mount with `vers=3.1.1` negotiates SMB 3.1.1 dialect with NEGOTIATE_CONTEXT (PreauthIntegrity / Encryption / Compression).
- [ ] AC-2: cifs_mount with `vers=3` negotiates SMB 3.0 (or 3.0.2); `vers=2.1` SMB 2.1; `vers=1.0` SMB 1 if CONFIG_CIFS_ALLOW_INSECURE_LEGACY.
- [ ] AC-3: ip_connect tries port 445 first then port 139 when ctx->dstaddr.port == 0.
- [ ] AC-4: generic_ip_connect creates the socket in the mount's net-ns and binds srcaddr if specified.
- [ ] AC-5: cifs_demultiplex_thread reads RFC1002 4-byte length header then full PDU and calls ops->find_mid + mid->callback.
- [ ] AC-6: On socket failure cifs_demultiplex_thread invokes cifs_reconnect(server, true) and resumes after CifsNeedNegotiate.
- [ ] AC-7: cifs_reconnect drains pending_mid_q to retry list and re-resolves hostname via DNS upcall before reconnect.
- [ ] AC-8: reconnect_dfs_server walks dfs_cache_get_tgt_iterator and retargets server->hostname per target.
- [ ] AC-9: cifs_setup_session retries once with ses->password2 on -EACCES/-EKEYEXPIRED/-EKEYREVOKED when password2 present.
- [ ] AC-10: TID state machine: TID_NEW -> TID_IN_TCON -> TID_GOOD on cifs_tree_connect success.
- [ ] AC-11: ctx->multichannel queues mchan_mount_work_fn which invokes smb3_update_ses_channels at cifsiod_wq.
- [ ] AC-12: ctx->linux_ext on SMB3.11 with server->posix_ext_supported sets tcon->posix_extensions and CIFS_MOUNT_POSIX_PATHS.
- [ ] AC-13: cifs_setup_ipc creates an IPC$ tcon usable for DFS referral RPC.
- [ ] AC-14: SERVER_IS_CHAN(tcp_ses) properly holds a srv_count reference on primary_server.
- [ ] AC-15: cifs_echo_request runs at echo_interval; server_unresponsive triggers reconnect after 2x interval + 5s.

## Architecture

```
struct TcpServerInfo {
  tcp_ses_list: ListNode,                // ∈ cifs_tcp_ses_list (global)
  smb_ses_list: ListHead,                // child cifs_ses list
  hostname: OwnedStr,
  leaf_fullpath: Option<OwnedStr>,       // DFS leaf
  dns_dom: [u8; CIFS_MAX_DOMAINNAME_LEN],
  ops: &'static SmbVersionOperations,    // dialect dispatcher
  vals: &'static SmbVersionValues,
  ssocket: Option<*Socket>,
  smbd_conn: Option<*SmbdConnection>,    // RDMA
  srcaddr: SockaddrStorage,
  dstaddr: SockaddrStorage,
  tcpStatus: TcpStatus,                  // CifsNew/InNegotiate/NeedNegotiate/Good/NeedReconnect/Exiting
  conn_id: u64,
  srv_count: AtomicI32,
  in_flight: AtomicI32,
  max_in_flight: u32,
  credits: AtomicI32,
  max_credits: u32,
  sequence_number: u64,
  channel_sequence_num: u64,
  reconnect_instance: u32,
  client_guid: [u8; SMB2_CLIENT_GUID_SIZE],
  capabilities: u32,                     // server-side
  dialect: u16,                          // SMB10/20/21/30/302/311
  sec_mode: u8,
  sign: bool,
  ignore_signature: bool,
  workstation_RFC1001_name: [u8; RFC1001_NAME_LEN_WITH_NULL],
  server_RFC1001_name: [u8; RFC1001_NAME_LEN_WITH_NULL],
  rfc1001_sessinit: i8,
  with_rfc1001: bool,
  echo_interval: Jiffies,
  echo: DelayedWork,
  reconnect: DelayedWork,
  pending_mid_q: ListHead,
  response_q: WaitQueueHead,
  request_q: WaitQueueHead,
  _srv_mutex: Mutex<()>,                 // cifs_server_lock
  srv_lock: SpinLock,
  req_lock: SpinLock,
  mid_queue_lock: SpinLock,
  mid_counter_lock: SpinLock,
  tsk: Option<*TaskStruct>,              // cifsd
  primary_server: Option<*TcpServerInfo>,// non-NULL for channel
  noblockcnt: bool,                      // rootfs
  noblocksnd: bool,
  noautotune: bool,
  tcp_nodelay: bool,
  rdma: bool,
  dfs_conn: bool,
  nosharesock: bool,
  nr_targets: u32,                       // DFS target count
  lstrp: Jiffies,                        // last received timestamp
  compression: CompressionState,
  min_offload: u32,
  retrans: u32,
  net_ns: *Net,
}

struct CifsSes {
  smb_ses_list: ListNode,                // ∈ TcpServerInfo.smb_ses_list
  server: *TcpServerInfo,                // primary channel
  chans: [Channel; CIFS_MAX_CHANNELS],
  chan_count: u32,
  chan_max: u32,
  chans_need_reconnect: u32,             // bitmap
  ses_status: SesStatus,                 // SES_NEW/IN_SETUP/GOOD/NEED_RECON
  ses_lock: SpinLock,
  chan_lock: SpinLock,
  session_mutex: Mutex<()>,
  iface_lock: SpinLock,
  iface_list: ListHead,
  iface_last_update: Jiffies,
  user_name: Option<OwnedStr>,
  password: Option<SecretStr>,
  password2: Option<SecretStr>,
  domainName: Option<OwnedStr>,
  workstation_name: [u8; SMB2_WORKSTATION_NAME_MAX],
  cred_uid: Kuid,
  linux_uid: Kuid,
  ip_addr: [u8; INET6_ADDRSTRLEN],
  sectype: SecurityEnum,                 // Unspecified/NTLMv2/Kerberos/RawNTLMSSP/...
  sign: bool,
  unicode: i8,
  upcall_target: UpcallTarget,           // UPTARGET_APP / UPTARGET_MOUNT
  local_nls: *NlsTable,
  capabilities: u32,
  smb3signingkey: [u8; SMB3_SIGN_KEY_SIZE],
  auth_key: AuthKey,
  tcon_ipc: Option<*CifsTcon>,           // IPC$
  dfs_root_ses: Option<*CifsSes>,
}

struct CifsTcon {
  tcon_list: ListNode,                   // ∈ CifsSes.tcon_list
  ses: *CifsSes,
  status: TidStatus,                     // TID_NEW/IN_TCON/GOOD/NEED_TCON/EXITING
  tc_lock: SpinLock,
  tree_name: OwnedStr,
  password: Option<SecretStr>,
  posix_extensions: bool,                // SMB3.11 POSIX
  unix_ext: bool,                        // SMB1 Unix Ext (legacy)
  seal: bool,
  nohandlecache: bool,
  snapshot_time: u64,
  handle_timeout: u32,
  need_reconnect: bool,
  capabilities: u32,
  share_flags: u32,
  ...
}
```

`Connect::get_tcp_session(ctx, primary_server) -> Result<*TcpServerInfo>`:
1. tcp_ses = Connect::find_tcp_session(ctx); if Some(s): return Ok(s).
2. tcp_ses = kzalloc(TcpServerInfo).
3. tcp_ses.hostname = kstrdup(ctx.server_hostname).
4. if ctx.leaf_fullpath: tcp_ses.leaf_fullpath = kstrdup(ctx.leaf_fullpath).
5. if ctx.dns_dom: strscpy(tcp_ses.dns_dom, ctx.dns_dom).
6. tcp_ses.nosharesock = ctx.nosharesock; dfs_conn = ctx.dfs_conn.
7. tcp_ses.ops = ctx.ops; vals = ctx.vals.
8. cifs_set_net_ns(tcp_ses, get_net(current.nsproxy.net_ns)).
9. tcp_ses.sign = ctx.sign; conn_id = atomic_inc_return(&tcpSesNextId).
10. tcp_ses.noblockcnt = ctx.rootfs; noblocksnd = ctx.noblocksnd || ctx.rootfs; noautotune = ctx.noautotune; tcp_nodelay = ctx.sockopt_tcp_nodelay; rdma = ctx.rdma.
11. tcp_ses.credits = 1.
12. if primary_server: ++primary_server.srv_count; tcp_ses.primary_server = primary_server.
13. init_waitqueue_head(&tcp_ses.response_q / request_q); INIT_LIST_HEAD(&pending_mid_q); mutex_init(&_srv_mutex); spin_lock_init each.
14. memcpy workstation_RFC1001_name = ctx.source_rfc1001_name; server_RFC1001_name = ctx.target_rfc1001_name.
15. INIT_DELAYED_WORK(&echo, cifs_echo_request); INIT_DELAYED_WORK(&reconnect, smb2_reconnect_server); mutex_init(&reconnect_mutex).
16. memcpy srcaddr / dstaddr from ctx.
17. if ctx.use_client_guid: memcpy(client_guid, ctx.client_guid, SMB2_CLIENT_GUID_SIZE); else generate_random_uuid(client_guid).
18. tcp_ses.tcpStatus = CifsNew; ++srv_count; echo_interval = ctx.echo_interval * HZ.
19. if rdma: tcp_ses.smbd_conn = smbd_get_connection(tcp_ses, &ctx.dstaddr).
20. else: rc = Connect::ip_connect(tcp_ses).
21. __module_get(THIS_MODULE).
22. tcp_ses.tsk = kthread_run(cifs_demultiplex_thread, tcp_ses, "cifsd").
23. tcp_ses.min_offload = ctx.min_offload; retrans = ctx.retrans.
24. tcp_ses.tcpStatus = CifsNeedNegotiate.
25. max_credits = if 20 ≤ ctx.max_credits ≤ 60000 { ctx.max_credits } else { SMB2_MAX_CREDITS_AVAILABLE }.
26. nr_targets = 1; ignore_signature = ctx.ignore_signature.
27. spin_lock(&cifs_tcp_ses_lock); list_add(&tcp_ses.tcp_ses_list, &cifs_tcp_ses_list); spin_unlock.
28. queue_delayed_work(cifsiod_wq, &echo, echo_interval).
29. return Ok(tcp_ses).

`Connect::ip_connect(server) -> i32`:
1. sport = if AF_INET6: &dst6.sin6_port else &dst4.sin_port.
2. if *sport == 0:
   - *sport = htons(CIFS_PORT) /* 445 */; rc = generic_ip_connect; if rc ≥ 0: return rc.
   - *sport = htons(RFC1001_PORT) /* 139 */.
3. return generic_ip_connect(server).

`Connect::generic_ip_connect(server) -> i32`:
1. saddr = &server.dstaddr; (sport, slen, sfamily) = if AF_INET6 then (sin6_port, 28, AF_INET6) else (sin_port, 16, AF_INET).
2. if server.ssocket.is_some: reuse; else sock_create_kern(cifs_net_ns(server), sfamily, SOCK_STREAM, IPPROTO_TCP, &server.ssocket).
3. sk_net_refcnt_upgrade(sk); sk.sk_allocation = GFP_NOFS; sk.sk_use_task_frag = false.
4. cifs_reclassify_socket{4,6}(socket) /* lockdep class */.
5. Connect::bind_socket(server) /* if srcaddr present */.
6. sk.sk_rcvtimeo = 7 * HZ; sk_sndtimeo = 5 * HZ.
7. if noautotune: clamp sndbuf ≥ 200 KiB, rcvbuf ≥ 140 KiB.
8. if tcp_nodelay: tcp_sock_set_nodelay.
9. rc = kernel_connect(socket, saddr, slen, if noblockcnt { O_NONBLOCK } else 0).
10. if noblockcnt ∧ rc == -EINPROGRESS: rc = 0 /* rootfs fast-path */.
11. if rc < 0: trace_smb3_connect_err; sock_release; ssocket = NULL; return rc.
12. trace_smb3_connect_done.
13. if with_rfc1001 ∨ rfc1001_sessinit == 1 ∨ (rfc1001_sessinit == -1 ∧ sport == htons(RFC1001_PORT)): rc = ip_rfc1001_connect(server).
14. return rc.

`Connect::demultiplex_thread(p: *TcpServerInfo)`:
1. noreclaim_flag = memalloc_noreclaim_save.
2. allow_kernel_signal(SIGKILL); set_freezable.
3. mempool_resize(cifs_req_poolp, ...).
4. while server.tcpStatus != CifsExiting {
   - try_to_freeze; allocate_buffers(server).
   - buf = server.smallbuf; pdu_length = 4.
   - length = cifs_read_from_socket(server, buf, pdu_length); if < 0: continue.
   - pdu_length = be32_to_cpup(buf) & 0xffffff.
   - if !is_smb_response(server, buf[0]): continue.
   - next_pdu: server.pdu_size = pdu_length.
   - if pdu_size < MID_HEADER_SIZE(server): cifs_reconnect(server, true); continue.
   - length = cifs_read_from_socket(server, buf, MID_HEADER_SIZE(server)).
   - if ops.next_header(server, buf, &next_offset): cifs_reconnect(server, true); continue.
   - if next_offset: server.pdu_size = next_offset.
   - if ops.is_transform_hdr(buf): length = ops.receive_transform(server, mids, bufs, &num_mids) /* decrypt SMB3 */.
   - else: mids[0] = ops.find_mid(server, buf); bufs[0] = buf; num_mids = 1; standard_receive3(server, mids[0]) or mids[0].receive.
   - if length < 0: release_mids; continue.
   - if ops.is_status_io_timeout(buf): num_io_timeout++; if > MAX_STATUS_IO_TIMEOUT: pending_reconnect = true; num_io_timeout = 0.
   - server.lstrp = jiffies.
   - For each mid in mids:
     - resp_buf_size = pdu_size.
     - if ops.is_network_name_deleted(bufs[i], server): log "Share deleted".
     - if !multiRsp ∨ multiEnd: mid_execute_callback(server, mids[i]).
     - release_mid(server, mids[i]).
   - if mids[i] is null ∧ ops.is_oplock_break(bufs[i], server): smb2_add_credits_from_hdr; log "oplock break".
   - if pdu_length > pdu_size: allocate_buffers; goto next_pdu.
   - if pending_reconnect: cifs_reconnect(server, true).
}
5. cifs_buf_release(server.bigbuf); cifs_small_buf_release(server.smallbuf).
6. task_to_wake = xchg(&server.tsk, NULL).
7. clean_demultiplex_info(server).
8. if !task_to_wake: wait for signal; exit.
9. memalloc_noreclaim_restore; module_put_and_kthread_exit(0).

`Connect::reconnect(server, mark_smb_session) -> i32`:
1. if !server.leaf_fullpath: __cifs_reconnect(server, mark_smb_session, false).
2. else: reconnect_dfs_server(server).

`Connect::__cifs_reconnect(server, mark_smb_session, once) -> i32`:
1. if !cifs_tcp_ses_needs_reconnect(server, 1): return 0.
2. if mark_smb_session: cifs_signal_cifsd_for_reconnect(server, true).
3. cifs_mark_tcp_ses_conns_for_reconnect(server, mark_smb_session).
4. cifs_abort_connection(server).
5. loop {
   - try_to_freeze; cifs_server_lock.
   - if !cifs_swn_set_server_dstaddr(server) ∧ !SERVER_IS_CHAN(server): reconn_set_ipaddr_from_hostname(server).
   - rc = if cifs_rdma_enabled(server) { smbd_reconnect } else { generic_ip_connect }.
   - if rc: cifs_server_unlock; if once: break; msleep(3000).
   - else: atomic_inc(&tcpSesReconnectCount); set_credits(server, 1); tcpStatus = CifsNeedNegotiate; cifs_swn_reset_server_dstaddr; cifs_server_unlock; cifs_queue_server_reconn(server).
   - if tcpStatus != CifsNeedReconnect: break.
}
6. if tcpStatus == CifsNeedNegotiate: mod_delayed_work(cifsiod_wq, &echo, 0).
7. wake_up(&server.response_q); return rc.

`Connect::get_smb_ses(server, ctx) -> Result<*CifsSes>`:
1. xid = get_xid.
2. ses = Connect::find_smb_ses(server, ctx); if Some:
   - if cifs_chan_needs_reconnect(ses, server):
     - mutex_lock(&ses.session_mutex).
     - rc = cifs_negotiate_protocol(xid, ses, server).
     - if !rc: rc = cifs_setup_session(xid, ses, server, ctx.local_nls).
     - if rc ∈ { -EACCES, -EKEYEXPIRED, -EKEYREVOKED } ∧ ses.password2 ∧ !retries: swap(ses.password, ses.password2); retries++; goto retry.
     - mutex_unlock.
   - cifs_put_tcp_session(server, 0); free_xid; return ses.
3. ses = sesInfoAlloc; ses.server = server.
4. Format ses.ip_addr from dstaddr.
5. ses.user_name = kstrdup(ctx.username); password = kstrdup(ctx.password); password2 = kstrdup(ctx.password2); domainName = kstrdup(ctx.domainname).
6. strscpy(ses.workstation_name, ctx.workstation_name).
7. ses.cred_uid = ctx.cred_uid; linux_uid = ctx.linux_uid; unicode = ctx.unicode; sectype = ctx.sectype; sign = ctx.sign.
8. ses.upcall_target = ctx.upcall_target.
9. ses.local_nls = load_nls(ctx.local_nls.charset).
10. chans[0].server = server; chan_count = 1; chan_max = ctx.multichannel ? ctx.max_channels : 1; chans_need_reconnect = 1.
11. retry_new_session: mutex_lock(&session_mutex); cifs_negotiate_protocol -> cifs_setup_session; mutex_unlock.
12. memcpy chans[0].signkey = ses.smb3signingkey.
13. If rc ∈ password-error-set ∧ password2 ∧ !retries: swap; retries++; retry.
14. list_add(&ses.smb_ses_list, &server.smb_ses_list).
15. ipc = cifs_setup_ipc(ses, ctx.seal); ses.tcon_ipc = !IS_ERR(ipc) ? ipc : NULL.
16. free_xid; return ses.

`Connect::get_tcon(ses, ctx) -> Result<*CifsTcon>`:
1. tcon = Connect::find_tcon(ses, ctx); if Some: cifs_put_smb_ses(ses); return tcon.
2. if !ses.server.ops.tree_connect: -ENOSYS.
3. nohandlecache decision (DIRECTORY_LEASING gated).
4. tcon = tcon_info_alloc(!nohandlecache).
5. snapshot_time, handle_timeout: require SMB2.0+.
6. tcon.ses = ses; password kstrdup.
7. seal: require SMB3+ ∧ SMB2_GLOBAL_CAP_ENCRYPTION.
8. linux_ext: SMB3.11 + posix_ext_supported -> tcon.posix_extensions = true (pr_warn_once experimental).
9. cifs_tree_connect(xid, tcon).

`Connect::tree_connect(xid, tcon) -> i32`:
1. spin_lock(&tcon.tc_lock).
2. if tcon.need_reconnect: tcon.status = TID_NEED_TCON.
3. if tcon.status == TID_GOOD: spin_unlock; return 0.
4. if tcon.status ∉ { TID_NEW, TID_NEED_TCON }: spin_unlock; return -EHOSTDOWN.
5. tcon.status = TID_IN_TCON; spin_unlock.
6. rc = ops.tree_connect(xid, tcon.ses, tcon.tree_name, tcon, ses.local_nls).
7. spin_lock; if status == TID_IN_TCON: if rc -> TID_NEED_TCON else { TID_GOOD; need_reconnect = false }; spin_unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `srv_count_balanced` | INVARIANT | per-get_tcp_session ↔ put_tcp_session: srv_count get'd/put'd 1:1; primary_server ref balanced for channels. |
| `ssocket_lifecycle` | INVARIANT | per-generic_ip_connect: ssocket created or reused; on error sock_release + ssocket = NULL. |
| `tcpStatus_state_machine` | INVARIANT | tcpStatus transitions ∈ { CifsNew->CifsNeedNegotiate, CifsNeedNegotiate->CifsInNegotiate->CifsGood, CifsGood->CifsNeedReconnect->CifsNeedNegotiate, *->CifsExiting }. |
| `tid_state_machine` | INVARIANT | TID_NEW/NEED_TCON -> TID_IN_TCON -> TID_GOOD ∨ TID_NEED_TCON. |
| `ses_status_state_machine` | INVARIANT | SES_NEW -> SES_IN_SETUP -> SES_GOOD ∨ SES_NEED_RECON. |
| `cifsd_kthread_paired` | INVARIANT | per-tcp_ses: kthread_run cifsd ↔ kthread_stop on put; module_get/put balanced. |
| `mid_q_drained_on_abort` | INVARIANT | per-cifs_abort_connection: pending_mid_q drained to retry_list before sock_release. |
| `noreclaim_balanced` | INVARIANT | per-demultiplex_thread: memalloc_noreclaim_save / restore paired. |
| `net_ns_refcounted` | INVARIANT | per-cifs_set_net_ns: get_net at create, put_net at free. |
| `signing_key_cleared_on_abort` | INVARIANT | per-cifs_abort_connection: kfree_sensitive(session_key.response); len = 0. |
| `dfs_target_iter_bounded` | INVARIANT | per-reconnect_dfs_server: target walk bounded by dfs_cache_get_nr_tgts. |

### Layer 2: TLA+

`fs/smb/client/connect.tla`:
- Per-transport state machine: { CifsNew, CifsInNegotiate, CifsNeedNegotiate, CifsGood, CifsNeedReconnect, CifsExiting }.
- Per-ses state machine: { SES_NEW, SES_IN_SETUP, SES_GOOD, SES_NEED_RECON }.
- Per-tcon state machine: { TID_NEW, TID_IN_TCON, TID_GOOD, TID_NEED_TCON, TID_EXITING }.
- Properties:
  - `safety_no_tcon_without_ses` — TID_GOOD ⟹ ses.ses_status ∈ { SES_GOOD, SES_NEED_RECON }.
  - `safety_no_ses_without_negotiated_transport` — SES_GOOD ⟹ tcpStatus == CifsGood.
  - `safety_no_tx_during_reconnect` — tcpStatus == CifsNeedReconnect ⟹ no new MIDs submitted; existing -> MID_RETRY_NEEDED.
  - `safety_dfs_path_required_for_dfs_reconnect` — reconnect_dfs_server entry ⟹ server.leaf_fullpath.is_some.
  - `safety_port_fallback_terminates` — ip_connect tries at most {445, 139}.
  - `safety_password2_used_at_most_once` — cifs_get_smb_ses retries == 1 on password-error-set.
  - `liveness_reconnect_eventually_terminates_or_loops_forever` — once != true ⟹ loop while CifsNeedReconnect; once == true ⟹ break after one attempt.
  - `liveness_demultiplex_drains` — CifsExiting eventually observed ⟹ thread exits via module_put_and_kthread_exit.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Connect::get_tcp_session` post: ret.tcpStatus == CifsNeedNegotiate ∨ ret matches existing | `Connect::get_tcp_session` |
| `Connect::ip_connect` post: ssocket != NULL ⟹ tcpStatus moves toward CifsGood | `Connect::ip_connect` |
| `Connect::generic_ip_connect` post: rc < 0 ⟹ ssocket == NULL | `Connect::generic_ip_connect` |
| `Connect::demultiplex_thread` post: exit ⟹ tcpStatus == CifsExiting ∨ signal | `Connect::demultiplex_thread` |
| `Connect::abort_connection` post: pending_mid_q empty ∨ all moved to retry_list | `Connect::abort_connection` |
| `Connect::reconnect` post: tcpStatus ∈ { CifsNeedNegotiate, CifsExiting } | `Connect::reconnect` |
| `Connect::get_smb_ses` post: ret.ses_status == SES_GOOD ∨ Err | `Connect::get_smb_ses` |
| `Connect::tree_connect` post: tcon.status ∈ { TID_GOOD, TID_NEED_TCON } | `Connect::tree_connect` |
| `Connect::negotiate_protocol` post: tcpStatus == CifsGood ∨ retry-once exhausted | `Connect::negotiate_protocol` |
| `Connect::setup_session` post: ses.ses_status ∈ { SES_GOOD, SES_NEED_RECON } | `Connect::setup_session` |

### Layer 4: Verus/Creusot functional

Per-SMB protocol semantics:
- SMB2 NEGOTIATE per `[MS-SMB2] 3.2.4.2.2.1`: client publishes Dialects[] including 0x0202 (2.0.2), 0x0210 (2.1), 0x0300 (3.0), 0x0302 (3.0.2), 0x0311 (3.1.1); server selects.
- SMB 3.1.1 NEGOTIATE_CONTEXT chain (PreauthIntegrity SHA-512, Encryption AES-128-GCM / AES-128-CCM / AES-256-*, Compression LZNT1/LZ77/LZ77+Huffman, NetName, RDMA, Signing AES-CMAC / AES-GMAC) per `[MS-SMB2] 3.2.4.2.2.2`.
- SMB2 SESSION_SETUP per `[MS-SMB2] 3.2.4.2.3`: SPNEGO/NTLMSSP/Kerberos blobs; possibly two-roundtrip NTLMSSP negotiate/auth.
- SMB2 TREE_CONNECT per `[MS-SMB2] 3.2.4.2.4`: returns TreeId; tcon-status promotion.
- RFC1002 NetBIOS session-init only when explicitly enabled or port 139 default.
- Multichannel per `[MS-SMB2] 3.2.4.1.7`: secondary channels created over additional interfaces returned by FSCTL_QUERY_NETWORK_INTERFACE_INFO; signing-key per channel derived from same Session.SigningKey except for SMB 3.1.1 where each channel has its own signing key (chans[i].signkey).
- DFS retargeting per `[MS-DFSC]`: GET_DFS_REFERRAL returns target list; reconnect walks targets until one succeeds.

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

SMB client connect reinforcement:

- **Per-transport in caller's net-ns** — defense against per-cross-netns mount leak (`cifs_set_net_ns(tcp_ses, get_net(current->nsproxy->net_ns))`).
- **Per-`sk_use_task_frag = false`** — defense against per-task-frag-leak across kthread / userspace context.
- **Per-`GFP_NOFS` socket allocation** — defense against per-reclaim-recursion deadlock on socket allocation under writeback.
- **Per-`memalloc_noreclaim_save` in cifsd** — defense against per-RX-thread-blocked-on-reclaim.
- **Per-`kfree_sensitive` for session_key + password** — defense against per-secret leak via slab debugging.
- **Per-password2 + retries ≤ 1** — defense against per-password-DoS (account lockout).
- **Per-credit-clamp `[20, 60000]`** — defense against per-server-malicious-credit-grant exhausting RAM.
- **Per-`is_status_io_timeout` reconnect threshold (`MAX_STATUS_IO_TIMEOUT`)** — defense against per-server-stuck silently.
- **Per-`MAX_COMPOUND` chained-PDU bound** — defense against per-server-compound-response-flood.
- **Per-`sk_rcvtimeo`/`sk_sndtimeo` (7s / 5s)** — defense against per-stuck-socket starvation.
- **Per-`MID_HEADER_SIZE` lower-bound check** — defense against per-truncated-PDU header-confusion.
- **Per-DNS-re-resolve before reconnect** — defense against per-IP-staleness on long-lived mounts.
- **Per-DFS target list bounded** — defense against per-DFS-namespace-loop.
- **Per-`pr_warn_once` for SMB3.11 POSIX experimental** — defense against per-silent-experimental-feature-use.
- **Per-channel-signing-key isolation** — defense against per-channel-cross-replay (SMB 3.1.1).
- **Per-`allow_kernel_signal(SIGKILL)`** — defense against per-cifsd unkillable-on-shutdown.
- **Per-`SERVER_IS_CHAN` primary-ref accounting** — defense against per-primary-leak when last channel goes away.

## Grsecurity/PaX-style Reinforcement

SMB client connect is a high-value remote-attacker-controlled surface: the server can supply hostile responses to session-setup, tree-connect, and DFS referrals. Rookery applies the grsec/PaX floor:

- **PAX_USERCOPY** — `cifs_mount_data` and `smb3_fs_context` parse paths bound every `copy_from_user` against the per-field allocation; mount-options scratch is sized at allocation.
- **PAX_KERNEXEC** — `smb_version_operations`, `smb_version_values`, and per-dialect vtables are `static const`; dialect dispatch is `__ro_after_init`.
- **PAX_RANDKSTACK** — connect-thread stack offset randomized; per-mount kstack-shape leaks denied.
- **PAX_REFCOUNT** — `TCP_Server_Info.srv_count`, `cifs_ses.ses_count`, `cifs_tcon.tc_count`, `cifs_sb.sb_active` all saturating-refcount.
- **PAX_MEMORY_SANITIZE** — `kfree_sensitive` on session_key, password, and per-channel-signing-key; mount-options scratch zeroed; SMB session-setup credential buffers cannot bleed via slab reuse.
- **PAX_UDEREF** — mount-option parser treats userspace mount-data as `void __user *`; every field consumed via `copy_*_user` or `strncpy_from_user`.
- **PAX_RAP/kCFI** — `smb_version_operations.*` are CFI-typed; per-dialect handlers (`smb2_ops`, `smb3_ops`, `smb311_ops`) must match the canonical signature.
- **GRKERNSEC_HIDESYM** — connect failures and reconnect diagnostics never emit kernel pointers; `cifs_dbg` macros strip `%p`/`%px` to hashed pointer for non-CAP_SYSLOG dmesg consumers.
- **GRKERNSEC_DMESG** — server name + IP logged at most rate-limited via `pr_warn_once`; reconnect storm cannot DoS dmesg.
- **SMB session-setup credential MEMORY_SANITIZE** — every secret material slab (NTLMv2 hash, GSS context, kerberos ticket, AES-CMAC subkey) zeroed on free via `kfree_sensitive`; per-channel-signing-key isolated and zeroed on channel teardown.
- **SMB 3.1.1 pre-auth integrity verify** — the pre-auth integrity hash (computed over Negotiate/SessionSetup PDUs) is verified server-side response against client-tracked transcript before tree-connect; mismatched hash terminates the connection (defends against SMB3 downgrade and PDU-injection attacks).
- **DFS target list PAX_USERCOPY bound** — DFS referral response parse caps target count and per-target name length; malformed referrals cannot OOM the mount path.
- **Per-credit-clamp `[20, 60000]`** — hostile server cannot grant unbounded credits to balloon kernel allocation; layered with `MAX_COMPOUND` for chained PDUs.

Rationale: SMB has been a recurring kernel CVE source (CVE-2022-47939, CVE-2023-32250 etc.) precisely because server-controlled buffers reach kernel slab allocators; the grsec-equivalent contract here is to combine `PAX_MEMORY_SANITIZE` (no key bleed) with `PAX_USERCOPY` (every server-controlled length bound at allocation) at the connect/session-setup chokepoint.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `fs/smb/client/smb2pdu.c` SMB2 PDU XDR / response parsing (covered separately if expanded)
- `fs/smb/client/sess.c` SMB3 channel-binding logic / smb3_update_ses_channels (covered separately if expanded)
- `fs/smb/client/transport.c` cifs_send / cifs_read_from_socket / mid_q_entry lifecycle (covered separately if expanded)
- `fs/smb/client/smbdirect.c` SMB Direct (RDMA) transport (covered separately if expanded)
- `fs/smb/client/dfs_cache.c` DFS referral cache (covered separately if expanded)
- `fs/smb/client/cifsacl.c` ACL mapping (covered separately if expanded)
- `fs/smb/server/` ksmbd (covered separately, distinct subsystem)
- Kerberos / SPNEGO upcall path (cifs.upcall, cifs_spnego.c)
- Implementation code
