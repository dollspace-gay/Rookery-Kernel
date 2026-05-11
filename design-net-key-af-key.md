---
title: "Tier-3: net/key/af_key.c — PF_KEY (KAME) key-management socket"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-11
updated: 2026-05-11
---


## Design Specification

### Summary

**PF_KEY v2** is the KAME-derived **key-management socket** that bridges userspace IKE/IPsec daemons (racoon, iked, setkey) to the in-kernel **xfrm** SA / SP database. Per-`socket(PF_KEY, SOCK_RAW, PF_KEY_V2)` creates `struct pfkey_sock` (wraps `struct sock`, registers in per-netns `netns_pfkey.table`). Per-`sendmsg` carries a `struct sadb_msg` (base header + chained `sadb_ext_*` extensions) describing one of the SADB operations: `SADB_GETSPI`, `SADB_UPDATE`, `SADB_ADD`, `SADB_DELETE`, `SADB_GET`, `SADB_ACQUIRE`, `SADB_REGISTER`, `SADB_EXPIRE`, `SADB_FLUSH`, `SADB_DUMP`, `SADB_X_PROMISC`, `SADB_X_PCHANGE`, `SADB_X_SPDUPDATE`, `SADB_X_SPDADD`, `SADB_X_SPDDELETE`, `SADB_X_SPDGET`, `SADB_X_SPDDUMP`, `SADB_X_SPDFLUSH`, `SADB_X_SPDSETIDX`, `SADB_X_SPDDELETE2`, `SADB_X_SPDACQUIRE` (notify-only), `SADB_X_MIGRATE`, `SADB_X_NAT_T_NEW_MAPPING` (notify-only). Dispatch via `pfkey_funcs[SADB_MAX + 1]` table. Per-message translation between PF_KEY (`sadb_*`) and xfrm (`xfrm_state` / `xfrm_policy`) wire formats. Per-`xfrm_mgr` registration (`pfkeyv2_mgr`) routes asynchronous kernel-originated events (acquire, expire, migrate, new-mapping, policy-notify) back to subscribed PF_KEY sockets via `pfkey_broadcast()` with `BROADCAST_REGISTERED` / `BROADCAST_PROMISC_ONLY` / `BROADCAST_ALL` / `BROADCAST_ONE`. Per-`CAP_NET_ADMIN` gated. Per-`xfrm_cfg_mutex` serialises SADB-config writes. Critical for: IKEv1 / IKEv2 daemon compatibility, BSD-derived setkey tooling, transitional path during the deprecation window (removal scheduled 2027 per `pr_warn_once`).

This Tier-3 covers `net/key/af_key.c` (~3953 lines).

### Acceptance Criteria

- [ ] AC-1: socket(PF_KEY, SOCK_RAW, PF_KEY_V2) succeeds for CAP_NET_ADMIN.
- [ ] AC-2: socket(PF_KEY, SOCK_RAW, PF_KEY_V2) without CAP_NET_ADMIN ⟹ -EPERM.
- [ ] AC-3: socket(PF_KEY, SOCK_DGRAM, …) ⟹ -ESOCKTNOSUPPORT.
- [ ] AC-4: socket(PF_KEY, SOCK_RAW, /*not-V2*/) ⟹ -EPROTONOSUPPORT.
- [ ] AC-5: SADB_GETSPI installs LARVAL SA with SPI in [0x100, 0x0fffffff]; reply seq matches request.
- [ ] AC-6: SADB_ADD then SADB_GET round-trip preserves SPI, addresses, algo, lifetimes.
- [ ] AC-7: SADB_UPDATE rekeys existing SA (same SPI/daddr/proto), broadcasts to BROADCAST_REGISTERED.
- [ ] AC-8: SADB_DELETE removes SA; subsequent GET ⟹ ENOENT.
- [ ] AC-9: SADB_REGISTER subscribes socket; subsequent kernel SA event delivered to that socket.
- [ ] AC-10: SADB_FLUSH removes all SAs of given satype; broadcasts SADB_FLUSH.
- [ ] AC-11: SADB_DUMP iterates all SAs; resumes after recvmsg drains queue.
- [ ] AC-12: SADB_X_PROMISC enables receipt of all PF_KEY traffic on that socket.
- [ ] AC-13: SADB_X_SPDADD inserts xfrm_policy; SADB_X_SPDDUMP enumerates; SADB_X_SPDDELETE removes by selector.
- [ ] AC-14: SADB_X_MIGRATE (config-gated) migrates policy and broadcasts SADB_X_MIGRATE.
- [ ] AC-15: Kernel-side ACQUIRE on out-policy trigger ⟹ socket receives SADB_ACQUIRE with sadb_msg_seq.
- [ ] AC-16: Malformed sadb_msg (bad version / reserved != 0 / type out-of-range / bad len) ⟹ -EINVAL or -EMSGSIZE.
- [ ] AC-17: ext-hdr-chain with mis-stated sadb_*_len ⟹ -EINVAL via parse_exthdrs.

### Architecture

```
struct PfkeySock {
  sk: Sock,                           /* first member */
  registered: AtomicU32,              /* satype bitmap */
  promisc: AtomicBool,
  dump: PfkeyDumpState,
  dump_lock: Mutex<()>,
}

struct PfkeyDumpState {
  msg_version: u8,
  msg_portid: u32,
  dump: Option<fn(&PfkeySock) -> i32>,
  done: Option<fn(&PfkeySock)>,
  walker: PfkeyWalker,                /* XfrmPolicyWalk | XfrmStateWalk */
  skb: Option<SkBuff>,
}

struct NetnsPfkey {
  table: HlistHead<PfkeySock>,
  socks_nr: AtomicU32,
}
```

`AfKey::create(net, sock, protocol, kern) -> Result<()>`:
1. Require ns_capable(net.user_ns, CAP_NET_ADMIN) else Err(-EPERM).
2. Require sock.type == SOCK_RAW else Err(-ESOCKTNOSUPPORT).
3. Require protocol == PF_KEY_V2 else Err(-EPROTONOSUPPORT).
4. sk = sk_alloc(net, PF_KEY, GFP_KERNEL, &PROTO, kern).
5. pfk = pfkey_sk(sk); Mutex::init(&pfk.dump_lock).
6. sock.ops = &OPS; sock_init_data(sock, sk).
7. sk.sk_family = PF_KEY; sk.sk_destruct = AfKey::sock_destruct.
8. net_pfkey.socks_nr.fetch_add(1).
9. AfKey::insert(sk) — mutex(pfkey_mutex), sk_add_node_rcu.

`AfKey::sendmsg(sock, msg, len) -> Result<usize>`:
1. If msg.flags & MSG_OOB ⟹ Err(-EOPNOTSUPP).
2. If len > sk.sk_sndbuf - 32 ⟹ Err(-EMSGSIZE).
3. skb = alloc_skb(len, GFP_KERNEL).
4. memcpy_from_msg(skb_put(skb, len), msg, len).
5. hdr = AfKey::get_base_msg(skb)?; validates version / reserved / type / length-in-quads.
6. xfrm_cfg_mutex.lock().
7. err = AfKey::process(sk, skb, hdr).
8. xfrm_cfg_mutex.unlock().
9. On err ∧ hdr.is_some(): pfkey_error(hdr, err, sk) — synthesise reply with errno field.
10. kfree_skb(skb); Ok(len) or Err(err).

`AfKey::process(sk, skb, hdr)`:
1. AfKey::broadcast(skb_clone(skb), GFP_KERNEL, BROADCAST_PROMISC_ONLY, None, net) — promisc fan-out.
2. ext_hdrs: [Option<*const u8>; SADB_EXT_MAX].
3. parse_exthdrs(skb, hdr, &mut ext_hdrs) — chain walk + per-ext sanity (sadb_ext_min_len, verify_address_len, verify_key_len, verify_sec_ctx_len).
4. handler = FUNCS[hdr.sadb_msg_type].ok_or(-EOPNOTSUPP)?.
5. handler(sk, skb, hdr, &ext_hdrs).

`AfKey::broadcast(skb, gfp, flags, one_sk, net)`:
1. rcu_read_lock.
2. For sk in net_pfkey.table:
   - If sk == one_sk: continue (delivered separately).
   - If flags != BROADCAST_ALL:
     - flags & BROADCAST_PROMISC_ONLY ∧ !pfk.promisc: continue.
     - flags & BROADCAST_REGISTERED  ∧ !pfk.registered: continue.
     - flags & BROADCAST_ONE: continue (only one_sk).
   - err2 = AfKey::broadcast_one(skb, GFP_ATOMIC, sk).
   - If BROADCAST_REGISTERED ∧ err: err = err2 (clear after ≥1 success).
3. rcu_read_unlock.
4. If one_sk.is_some(): err = AfKey::broadcast_one(skb, gfp, one_sk).
5. kfree_skb(skb); return err.

`Sadb::getspi(sk, skb, hdr, ext_hdrs)`:
1. Validate src/dst present and same family.
2. proto = pfkey_satype2proto(hdr.sadb_msg_satype).
3. Decode SADB_X_EXT_SA2 → (mode, reqid) or (0, 0).
4. Extract xfrm_address_t for src/dst per family.
5. x = if hdr.sadb_msg_seq != 0: xfrm_find_acq_byseq(net, DUMMY_MARK, seq, UINT_MAX) (validate daddr) else xfrm_find_acq(net, &dummy_mark, mode, reqid, 0, UINT_MAX, proto, xdaddr, xsaddr, 1, family).
6. (min_spi, max_spi) = SADB_EXT_SPIRANGE or (0x100, 0x0fffffff).
7. verify_spi_info(x.id.proto, min_spi, max_spi, NULL).
8. xfrm_alloc_spi(x, min_spi, max_spi, NULL).
9. resp_skb = Xlate::xfrm_state_to_msg(x); fill out_hdr (SADB_GETSPI, satype, errno=0, seq, pid).
10. xfrm_state_put(x); AfKey::broadcast(resp_skb, GFP_KERNEL, BROADCAST_ONE, sk, net).

`Sadb::add(sk, skb, hdr, ext_hdrs)`:
1. x = Xlate::msg_to_xfrm_state(net, hdr, ext_hdrs).
2. xfrm_init_state(x).
3. event = if msg_type == SADB_ADD: xfrm_state_add(x) else xfrm_state_update(x).
4. key_notify_sa(x, &km_event{seq, portid, event, data}).

`Sadb::register_socket(sk, skb, hdr, ext_hdrs)`:
1. satype = hdr.sadb_msg_satype.
2. pfk.registered |= bit-for-satype (or full bitmap if SADB_SATYPE_UNSPEC).
3. resp = compose_sadb_supported(hdr, /*aalg-list*/, /*ealg-list*/).
4. AfKey::broadcast(resp, GFP_KERNEL, BROADCAST_REGISTERED, None, net).

`Sadb::send_acquire(x, t, xp)`:
1. seq = get_acqseq().
2. Build SADB_ACQUIRE skb (HDR | SA | SRC | DST | sec_ctx | identity_src | identity_dst | proposal | combs).
3. AfKey::broadcast(skb, GFP_ATOMIC, BROADCAST_REGISTERED, None, xs_net(x)).

`Sadb::send_new_mapping(x, ipaddr, sport)`:
1. Build SADB_X_NAT_T_NEW_MAPPING skb (HDR | SA | OLD_ADDR_SRC | OLD_PORT | NEW_ADDR_DST | NEW_PORT).
2. AfKey::broadcast(skb, GFP_ATOMIC, BROADCAST_REGISTERED, None, xs_net(x)).

### Out of Scope

- net/xfrm/xfrm_state.c SA database (covered separately if expanded)
- net/xfrm/xfrm_policy.c SP database (covered separately if expanded)
- net/xfrm/xfrm_user.c AF_NETLINK XFRM (covered separately if expanded)
- net/xfrm/xfrm_input.c / xfrm_output.c datapath (covered separately if expanded)
- ESP / AH / IPCOMP transforms (covered separately if expanded)
- IKE / racoon / iked userspace logic (out of kernel scope)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pfkey_sock` | per-socket state (registered, promisc, dump cursor) | `PfkeySock` |
| `struct netns_pfkey` | per-netns socket list + counter | `NetnsPfkey` |
| `pfkey_create()` | per-`socket(PF_KEY, SOCK_RAW, PF_KEY_V2)` | `AfKey::create` |
| `pfkey_release()` | per-close | `AfKey::release` |
| `pfkey_sendmsg()` | per-userspace SADB request | `AfKey::sendmsg` |
| `pfkey_recvmsg()` | per-userspace dequeue | `AfKey::recvmsg` |
| `pfkey_process()` | per-message dispatch | `AfKey::process` |
| `pfkey_funcs[SADB_MAX+1]` | per-opcode handler table | `AfKey::FUNCS` |
| `pfkey_broadcast()` | per-multicast to listeners | `AfKey::broadcast` |
| `pfkey_broadcast_one()` | per-recipient enqueue | `AfKey::broadcast_one` |
| `pfkey_insert()` / `_remove()` | per-table lifecycle | `AfKey::insert` / `remove` |
| `parse_exthdrs()` | per-`sadb_ext` chain walk | `AfKey::parse_ext_hdrs` |
| `pfkey_getspi()` | per-`SADB_GETSPI` | `Sadb::getspi` |
| `pfkey_add()` | per-`SADB_ADD` / `_UPDATE` | `Sadb::add` |
| `pfkey_delete()` | per-`SADB_DELETE` | `Sadb::delete` |
| `pfkey_get()` | per-`SADB_GET` | `Sadb::get` |
| `pfkey_acquire()` | per-`SADB_ACQUIRE` (userspace ACK) | `Sadb::acquire` |
| `pfkey_register()` | per-`SADB_REGISTER` (subscribe) | `Sadb::register_socket` |
| `pfkey_flush()` | per-`SADB_FLUSH` | `Sadb::flush` |
| `pfkey_dump()` | per-`SADB_DUMP` | `Sadb::dump` |
| `pfkey_promisc()` | per-`SADB_X_PROMISC` toggle | `Sadb::promisc` |
| `pfkey_spdadd()` | per-`SADB_X_SPDADD` / `_SPDUPDATE` / `_SPDSETIDX` | `Sadb::spdadd` |
| `pfkey_spddelete()` | per-`SADB_X_SPDDELETE` | `Sadb::spddelete` |
| `pfkey_spdget()` | per-`SADB_X_SPDGET` / `_SPDDELETE2` | `Sadb::spdget` |
| `pfkey_spddump()` | per-`SADB_X_SPDDUMP` | `Sadb::spddump` |
| `pfkey_spdflush()` | per-`SADB_X_SPDFLUSH` | `Sadb::spdflush` |
| `pfkey_migrate()` | per-`SADB_X_MIGRATE` | `Sadb::migrate` |
| `pfkey_msg2xfrm_state()` | per-`sadb_msg → xfrm_state` decode | `Xlate::msg_to_xfrm_state` |
| `pfkey_xfrm_state2msg()` | per-`xfrm_state → sadb_msg` encode | `Xlate::xfrm_state_to_msg` |
| `pfkey_xfrm_policy2msg()` | per-policy encode | `Xlate::xfrm_policy_to_msg` |
| `pfkey_send_acquire()` | per-kernel→user ACQUIRE | `Sadb::send_acquire` |
| `pfkey_send_notify()` | per-kernel→user SA notify | `Sadb::send_notify` |
| `pfkey_send_policy_notify()` | per-kernel→user SP notify | `Sadb::send_policy_notify` |
| `pfkey_send_new_mapping()` | per-NAT-T new-mapping | `Sadb::send_new_mapping` |
| `pfkey_send_migrate()` | per-migrate notify | `Sadb::send_migrate` |
| `pfkey_compile_policy()` | per-setsockopt(IPSEC_POLICY) compile | `Sadb::compile_policy` |
| `pfkey_is_alive()` | per-`xfrm_mgr.is_alive` | `Sadb::is_alive` |
| `pfkeyv2_mgr` (`struct xfrm_mgr`) | per-xfrm km bridge | `AfKey::MGR` |
| `key_proto` (`struct proto`) | per-proto definition | `AfKey::PROTO` |
| `pfkey_ops` (`struct proto_ops`) | per-socket vtable | `AfKey::OPS` |
| `pfkey_family_ops` (`struct net_proto_family`) | per-`sock_register(PF_KEY)` | `AfKey::FAMILY` |
| `pfkey_net_ops` | per-pernet init/exit | `AfKey::NET_OPS` |

### compatibility contract

REQ-1: struct pfkey_sock layout:
- sk: per-`struct sock` (first member — cast-compat).
- registered: per-bitmap of subscribed satypes (set by `SADB_REGISTER`).
- promisc: per-flag; if set, socket receives copy of every PF_KEY message.
- dump.msg_version / msg_portid: per-cursor metadata for ongoing dump.
- dump.dump / done: per-resumable-dump callbacks (`pfkey_dump_sa` / `pfkey_dump_sp`).
- dump.u.policy / u.state: per-xfrm walker cursor (`xfrm_policy_walk` / `xfrm_state_walk`).
- dump.skb: per-pending dump skb.
- dump_lock: per-dump-serialisation mutex.

REQ-2: pfkey_create(net, sock, protocol, kern):
- Require `ns_capable(net.user_ns, CAP_NET_ADMIN)` else -EPERM.
- Require sock.type == SOCK_RAW else -ESOCKTNOSUPPORT.
- Require protocol == PF_KEY_V2 else -EPROTONOSUPPORT.
- sk = sk_alloc(net, PF_KEY, GFP_KERNEL, &key_proto, kern); -ENOMEM on failure.
- pfk = pfkey_sk(sk); mutex_init(&pfk.dump_lock).
- sock.ops = &pfkey_ops; sock_init_data(sock, sk).
- sk.sk_family = PF_KEY; sk.sk_destruct = pfkey_sock_destruct.
- atomic_inc(&net_pfkey.socks_nr).
- pfkey_insert(sk) — mutex_lock(pfkey_mutex), sk_add_node_rcu(sk, &net_pfkey.table), unlock.

REQ-3: pfkey_release(sock):
- pfkey_remove(sk) — mutex_lock(pfkey_mutex), sk_del_node_init_rcu(sk), unlock.
- sock_orphan(sk); sock.sk = NULL; skb_queue_purge(&sk.sk_write_queue).
- synchronize_rcu().
- sock_put(sk).
- pfkey_sock_destruct: pfkey_terminate_dump(pfk); skb_queue_purge(receive_queue); decrement socks_nr.

REQ-4: pfkey_sendmsg(sock, msg, len):
- err = -EOPNOTSUPP if msg.msg_flags & MSG_OOB.
- err = -EMSGSIZE if len > sk.sk_sndbuf - 32.
- skb = alloc_skb(len, GFP_KERNEL); -ENOBUFS on failure.
- memcpy_from_msg(skb_put(skb,len), msg, len); -EFAULT on failure.
- hdr = pfkey_get_base_msg(skb, &err); validate sadb_msg_version == PF_KEY_V2, reserved == 0, 0 < sadb_msg_type ≤ SADB_MAX, sadb_msg_len == skb.len/8 ≥ sizeof(sadb_msg)/8.
- mutex_lock(&net.xfrm.xfrm_cfg_mutex).
- err = pfkey_process(sk, skb, hdr).
- mutex_unlock(&net.xfrm.xfrm_cfg_mutex).
- On err with valid hdr: pfkey_error(hdr, err, sk) — synthesise reply with sadb_msg_errno = err and BROADCAST_ALL.
- kfree_skb(skb); return err ?: len.

REQ-5: pfkey_process(sk, skb, hdr):
- /* Promisc fan-out for observers */
- pfkey_broadcast(skb_clone(skb, GFP_KERNEL), GFP_KERNEL, BROADCAST_PROMISC_ONLY, NULL, sock_net(sk)).
- memset(ext_hdrs, 0, sizeof(ext_hdrs[SADB_EXT_MAX])).
- err = parse_exthdrs(skb, hdr, ext_hdrs) — walks chained sadb_ext_*; validates length per sadb_ext_min_len[type] and per-ext sanity (verify_address_len / verify_key_len / verify_sec_ctx_len).
- If err == 0 ∧ pfkey_funcs[hdr.sadb_msg_type] != NULL:
  - err = pfkey_funcs[hdr.sadb_msg_type](sk, skb, hdr, ext_hdrs).
- Else err = -EOPNOTSUPP.
- Return err.

REQ-6: pfkey_funcs dispatch table:
- [SADB_RESERVED] = pfkey_reserved (always -EOPNOTSUPP).
- [SADB_GETSPI]   = pfkey_getspi.
- [SADB_UPDATE]   = pfkey_add (rekey path).
- [SADB_ADD]      = pfkey_add (new SA install).
- [SADB_DELETE]   = pfkey_delete.
- [SADB_GET]      = pfkey_get.
- [SADB_ACQUIRE]  = pfkey_acquire (userspace ACK of in-kernel ACQUIRE).
- [SADB_REGISTER] = pfkey_register (subscribe socket to satype notifications).
- [SADB_EXPIRE]   = NULL (kernel-originated only).
- [SADB_FLUSH]    = pfkey_flush.
- [SADB_DUMP]     = pfkey_dump.
- [SADB_X_PROMISC]  = pfkey_promisc.
- [SADB_X_PCHANGE]  = NULL.
- [SADB_X_SPDUPDATE] = pfkey_spdadd.
- [SADB_X_SPDADD]    = pfkey_spdadd.
- [SADB_X_SPDDELETE] = pfkey_spddelete.
- [SADB_X_SPDGET]    = pfkey_spdget.
- [SADB_X_SPDACQUIRE] = NULL (kernel-originated only).
- [SADB_X_SPDDUMP]   = pfkey_spddump.
- [SADB_X_SPDFLUSH]  = pfkey_spdflush.
- [SADB_X_SPDSETIDX] = pfkey_spdadd.
- [SADB_X_SPDDELETE2] = pfkey_spdget (lookup-then-delete by index).
- [SADB_X_MIGRATE]    = pfkey_migrate (requires CONFIG_NET_KEY_MIGRATE).

REQ-7: pfkey_getspi: per-SPI allocation:
- Require ext_hdrs[SADB_EXT_ADDRESS_SRC-1] and [_DST-1] present-and-same-family.
- proto = pfkey_satype2proto(hdr.sadb_msg_satype); 0 ⟹ -EINVAL.
- Decode optional SADB_X_EXT_SA2 → mode + reqid.
- Extract xsaddr / xdaddr per AF_INET / AF_INET6.
- If hdr.sadb_msg_seq != 0: x = xfrm_find_acq_byseq(net, DUMMY_MARK, seq, UINT_MAX); validate daddr-match else release.
- Else x = xfrm_find_acq(net, &dummy_mark, mode, reqid, 0, UINT_MAX, proto, xdaddr, xsaddr, /*create=*/1, family).
- min_spi = 0x100, max_spi = 0x0fffffff (or per SADB_EXT_SPIRANGE).
- verify_spi_info(x.id.proto, min_spi, max_spi, NULL).
- xfrm_alloc_spi(x, min_spi, max_spi, NULL).
- resp_skb = pfkey_xfrm_state2msg(x); fill out_hdr (type = SADB_GETSPI, satype, errno = 0, seq, pid).
- xfrm_state_put(x); pfkey_broadcast(resp_skb, GFP_KERNEL, BROADCAST_ONE, sk, net).

REQ-8: pfkey_add: per-SA install (ADD / UPDATE):
- x = pfkey_msg2xfrm_state(net, hdr, ext_hdrs) — decode sa, sa2, lifetime, address-src/dst, auth, encrypt, identity, sec-ctx, nat-t, encap, replay-window, encap-tmpl.
- xfrm_init_state(x) initialises algorithms / mode handlers.
- If hdr.sadb_msg_type == SADB_ADD: xfrm_state_add(x).
- If hdr.sadb_msg_type == SADB_UPDATE: xfrm_state_update(x) (rekey).
- On success: key_notify_sa(x, &km_event) broadcasts to BROADCAST_REGISTERED matching satype.

REQ-9: pfkey_delete / pfkey_get:
- x = pfkey_xfrm_state_lookup(net, hdr, ext_hdrs) by (spi, daddr, proto, family).
- delete: xfrm_state_delete(x); broadcast SADB_DELETE notify.
- get: encode x via pfkey_xfrm_state2msg; broadcast BROADCAST_ONE to caller.

REQ-10: pfkey_register: per-satype subscription:
- satype = hdr.sadb_msg_satype.
- For each requested satype, set bit in pfk.registered.
- Build SADB_REGISTER reply via compose_sadb_supported (per-aalg / per-ealg algo list with sadb_alg_id / minbits / maxbits / ivlen).
- BROADCAST_REGISTERED reply.

REQ-11: pfkey_flush: per-SA flush:
- xfrm_state_flush(net, proto-from-satype, audit-info, /*task_valid=*/true).
- key_notify_sa_flush broadcasts SADB_FLUSH to REGISTERED listeners; unicast_flush_resp to caller.

REQ-12: pfkey_dump: per-SA dump (resumable):
- mutex_lock(&pfk.dump_lock).
- Initialise pfk.dump.dump = pfkey_dump_sa, pfk.dump.done = pfkey_dump_sa_done.
- pfkey_do_dump(pfk) loops xfrm_state_walk_init → walk_step iterations subject to pfkey_can_dump (3*rmem_alloc ≤ 2*rcvbuf).
- Each row emitted via dump_sa → pfkey_xfrm_state2msg → BROADCAST_ONE.
- On suspend (-ENOBUFS): keep cursor; pfkey_recvmsg resumes via pfkey_do_dump after socket drain.

REQ-13: pfkey_promisc:
- Per-`SADB_X_PROMISC`: toggle pfk.promisc per hdr payload.
- BROADCAST_ALL the original message so observers see the toggle.

REQ-14: pfkey_spdadd / spddelete / spdget / spddump / spdflush:
- spdadd handles X_SPDADD, X_SPDUPDATE, X_SPDSETIDX; converts sadb_x_policy + sec-ctx + addresses into xfrm_policy via pfkey_msg2xfrm_policy and installs via xfrm_policy_insert.
- spddelete looks up by selector/dir/sec-ctx and xfrm_policy_delete.
- spdget retrieves by sadb_x_policy_id (also services X_SPDDELETE2 with delete-after-get).
- spddump resumable via pfk.dump.policy walker (analogous to SA dump).
- spdflush via xfrm_policy_flush; key_notify_policy_flush broadcast.

REQ-15: pfkey_migrate (CONFIG_NET_KEY_MIGRATE):
- Parse SADB_X_EXT_KMADDRESS, SADB_X_EXT_POLICY, ipsecrequest chain into xfrm_migrate[XFRM_MAX_DEPTH].
- xfrm_migrate(&sel, dir, XFRM_POLICY_TYPE_MAIN, m, i, kma ? &k : NULL, net, NULL, 0, NULL, NULL).
- Without CONFIG_NET_KEY_MIGRATE: return -ENOPROTOOPT.

REQ-16: pfkey_broadcast(skb, allocation, broadcast_flags, one_sk, net):
- BROADCAST_ALL = 0: fan out to all sockets in netns_pfkey.table.
- BROADCAST_ONE = 1: deliver to one_sk only (out-of-band reply path).
- BROADCAST_REGISTERED = 2: deliver to sockets with pfk.registered set.
- BROADCAST_PROMISC_ONLY = 4: deliver only to pfk.promisc sockets.
- rcu_read_lock; sk_for_each_rcu iterates the per-netns table; pfkey_broadcast_one clones + skb_set_owner_r + skb_queue_tail; sk.sk_data_ready notifies.
- If one_sk != NULL ∧ flag matches: deliver to one_sk last with caller allocation gfp.

REQ-17: xfrm_mgr bridge (pfkeyv2_mgr):
- .notify       = pfkey_send_notify          — SA add/update/delete/expire to BROADCAST_REGISTERED.
- .acquire      = pfkey_send_acquire         — synthesises SADB_ACQUIRE on kernel SP-trigger; allocates seq via get_acqseq().
- .compile_policy = pfkey_compile_policy     — IP_IPSEC_POLICY / IPV6_IPSEC_POLICY setsockopt.
- .new_mapping  = pfkey_send_new_mapping     — SADB_X_NAT_T_NEW_MAPPING on NAT keepalive mapping change.
- .notify_policy = pfkey_send_policy_notify  — SP add/del/update.
- .migrate      = pfkey_send_migrate         — SADB_X_MIGRATE on policy migrate event.
- .is_alive     = pfkey_is_alive             — liveness probe for ACQUIRE retry suppression (returns true iff ≥1 socket registered for proto).
- xfrm_register_km(&pfkeyv2_mgr) at module init; xfrm_unregister_km on exit.

REQ-18: Per-netns lifecycle:
- pfkey_net_init: INIT_HLIST_HEAD(&net_pfkey.table); atomic_set(socks_nr, 0); pfkey_init_proc (creates /proc/net/pfkey).
- pfkey_net_exit: pfkey_exit_proc; WARN_ON(!hlist_empty(&net_pfkey.table)).
- pernet_operations registered via register_pernet_subsys.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_net_admin_enforced` | INVARIANT | per-create: ns_capable(CAP_NET_ADMIN) required. |
| `sock_type_raw_only` | INVARIANT | per-create: sock.type == SOCK_RAW. |
| `protocol_v2_only` | INVARIANT | per-create: protocol == PF_KEY_V2. |
| `sadb_msg_version_v2` | INVARIANT | per-get_base_msg: sadb_msg_version == PF_KEY_V2 else -EINVAL. |
| `sadb_msg_type_range` | INVARIANT | per-get_base_msg: SADB_RESERVED < type ≤ SADB_MAX. |
| `sadb_msg_len_in_quads` | INVARIANT | per-get_base_msg: sadb_msg_len * 8 == skb.len ∧ ≥ sizeof(sadb_msg). |
| `ext_hdr_chain_bounded` | INVARIANT | per-parse_exthdrs: cumulative ext length ≤ skb.len. |
| `xfrm_cfg_mutex_held` | INVARIANT | per-process: net.xfrm.xfrm_cfg_mutex held during dispatch. |
| `broadcast_skb_owned` | INVARIANT | per-broadcast_one: skb_set_owner_r before queue. |
| `dump_lock_held` | INVARIANT | per-do_dump: pfk.dump_lock held. |
| `getspi_range_strict` | INVARIANT | per-getspi: 0x100 ≤ min_spi ≤ max_spi ≤ 0x0fffffff. |
| `registered_satype_bitmap` | INVARIANT | per-broadcast: BROADCAST_REGISTERED delivers iff pfk.registered & (1 << satype). |
| `error_synthesis_pad_bug_on` | INVARIANT | per-pfkey_error: 1 ≤ errno < 256. |

### Layer 2: TLA+

`net/key/af-key.tla`:
- States: SocketCreated, Registered(satype-set), Dumping, Closed.
- Events: SendMsg(opcode), KernelEvent(km_event), RecvDeq, Close.
- Properties:
  - `safety_only_capnetadmin_creates` — per-create: !CAP_NET_ADMIN ⟹ no socket.
  - `safety_register_then_event_delivered` — register(satype) ∧ kernel SA event of satype ⟹ socket receives event.
  - `safety_dispatch_under_xfrm_cfg_mutex` — every handler executes with xfrm_cfg_mutex.
  - `safety_no_handler_for_kernel_only_op` — type ∈ {EXPIRE, X_PCHANGE, X_SPDACQUIRE} ⟹ -EOPNOTSUPP from userspace.
  - `safety_promisc_observes_all` — promisc=true ⟹ every PF_KEY message is queued.
  - `liveness_dump_eventually_completes` — sufficient recvmsg drains ⟹ dump_done fires.
  - `liveness_acquire_retry_bounded` — per-`is_alive` false ⟹ kernel refrains from re-ACQUIRE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `AfKey::create` post: ret ⟹ sk inserted in net_pfkey.table ∧ ops/destruct wired | `AfKey::create` |
| `AfKey::sendmsg` post: len-bytes processed under xfrm_cfg_mutex; err synthesised via pfkey_error | `AfKey::sendmsg` |
| `AfKey::process` post: dispatch only to non-NULL FUNCS[type] | `AfKey::process` |
| `AfKey::broadcast` post: every target sk in net == sock_net(sk) | `AfKey::broadcast` |
| `Sadb::getspi` post: SPI ∈ [min, max]; x ref balanced | `Sadb::getspi` |
| `Sadb::add` post: SA installed; key_notify_sa fan-out to REGISTERED | `Sadb::add` |
| `Sadb::register_socket` post: bit(satype) set; supported-algos reply emitted | `Sadb::register_socket` |
| `Sadb::dump` post: walker resumable on -ENOBUFS | `Sadb::dump` |
| `Sadb::migrate` post: CONFIG_NET_KEY_MIGRATE off ⟹ -ENOPROTOOPT | `Sadb::migrate` |

### Layer 4: Verus/Creusot functional

`Per-sendmsg → parse → handler → xfrm-state/policy mutation → broadcast` semantic equivalence: per-RFC 2367 (PF_KEY Key Management API v2) + per-KAME extensions documented in `include/uapi/linux/pfkeyv2.h`. Round-trip: ADD/UPDATE then GET produces equivalent `sadb_msg`; DELETE then GET yields ENOENT.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

PF_KEY reinforcement:

- **Per-CAP_NET_ADMIN gating at create** — defense against unprivileged SADB tamper.
- **Per-PF_KEY_V2-only protocol** — defense against version-confusion.
- **Per-SOCK_RAW-only** — defense against per-stream/dgram-semantic surprise.
- **Per-`xfrm_cfg_mutex` serialisation of all writes** — defense against per-concurrent-SADB-mutation race.
- **Per-`sadb_msg_len` validation in 8-byte quads against skb.len** — defense against per-truncated/over-long header DoS.
- **Per-`parse_exthdrs` strict min-length + family/sec-ctx checks** — defense against per-malformed-extension OOB read.
- **Per-`pfkey_error` errno range 1..255 BUG_ON** — defense against per-platform-errno aliasing.
- **Per-`pfkey_can_dump` rmem watermark (3*rmem ≤ 2*rcvbuf)** — defense against per-dump-driven memory pressure.
- **Per-`dump_lock` mutex** — defense against per-concurrent-dump-walker corruption.
- **Per-`xfrm_state_put` / `xfrm_policy_put` balanced ref-counts** — defense against per-SA/SP UAF.
- **Per-`BROADCAST_REGISTERED` satype-bitmap gate** — defense against per-unrelated-listener info-leak.
- **Per-`SADB_X_PROMISC` ungated only for CAP_NET_ADMIN-holder (already filtered at create)** — defense against per-promisc-by-unprivileged.
- **Per-`pfkey_is_alive` suppression of unbounded ACQUIRE retries** — defense against per-no-daemon retry storm.
- **Per-deprecation pr_warn_once + 2027 removal schedule** — defense against per-legacy lock-in (forces migration to AF_NETLINK XFRM).

