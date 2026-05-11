# Tier-3: net/atm/svc.c — ATM PF_ATMSVC Switched-Virtual-Circuit socket

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/atm/svc.c (~696 lines)
  - net/atm/common.c (vcc_create / vcc_release / vcc_sendmsg / vcc_recvmsg)
  - net/atm/signaling.c + signaling.h (sigd, sigd_enq, sigd_enq2, as_* messages)
  - net/atm/addr.c + addr.h (atm_reset_addr / atm_add_addr / atm_get_addr — ESI cache)
  - include/uapi/linux/atm.h + atmsvc.h (struct sockaddr_atmsvc, atmsvc_msg, ATM_VF_*)
-->

## Summary

**PF_ATMSVC** is the **ATM Switched Virtual Circuit** socket family — connection-oriented ATM with **dynamic call setup** via the in-userspace **`atmsigd`** signalling daemon implementing **ITU-T Q.2931 / Q.2971**. Per-`socket(PF_ATMSVC, SOCK_*, proto)` allocates `struct atm_vcc` via `vcc_create()`; per-`bind()` registers an `struct sockaddr_atmsvc` (`sas_family = AF_ATMSVC`, 20-byte AESA: 13-byte network prefix + 6-byte **ESI** End-System-Identifier + 1-byte SEL) with the in-kernel ATM address cache and forwards `as_bind` to `sigd`; per-`connect()` issues `as_connect` and blocks until sigd answers; per-`listen()` / `accept()` cycle uses `as_listen` to install at the switch and dequeues incoming `atmsvc_msg` setup notifications from `sk->sk_receive_queue`; per-`as_accept` / `as_reject` userspace daemon arbitrates Q.2931 connect-confirm. Kernel↔daemon plumbing uses `sigd_enq()` / `sigd_enq2()` queueing through the special `sigd` VCC. All blocking waits use **prepare_to_wait + schedule + ATM_VF_WAITING bit** patterns; **bail out if `sigd == NULL`** (daemon died) with `-EUNATCH`. Per-`SO_ATMSAP` setsockopt installs **B-HLI / B-LLI service-access-point** for higher-layer matching at setup; per-`SO_MULTIPOINT` enables point-to-multipoint **leaf-initiated joins** with `ATM_ADDPARTY` / `ATM_DROPPARTY` ioctls; per-`svc_change_qos` re-negotiates `atm_qos` mid-call (`as_modify`). Critical for: classical-IP-over-ATM RFC 1577, LAN-Emulation, MPOA, native ATM applications.

This Tier-3 covers `net/atm/svc.c` (~696 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct atm_vcc` | per-VCC kernel state (flags, qos, sap, local, remote, itf/vpi/vci) | `AtmVcc` |
| `sigd` (`struct atm_vcc *`) | per-singleton link to atmsigd daemon | `Atm::sigd` |
| `sigd_enq(vcc, type, listen_vcc, pvc, svc)` | per-message enqueue (zero qos/ep_ref) | `Atm::sigd_enq` |
| `sigd_enq2(vcc, type, listen_vcc, pvc, svc, qos, ep_ref)` | per-message enqueue with qos / endpoint-ref | `Atm::sigd_enq2` |
| `svc_create()` | per-`socket(PF_ATMSVC, …)` | `Svc::create` |
| `svc_release()` | per-close | `Svc::release` |
| `svc_disconnect()` | per-tear-down (as_close handshake) | `Svc::disconnect` |
| `svc_bind()` | per-`bind(sockaddr_atmsvc)` | `Svc::bind` |
| `svc_connect()` | per-`connect(sockaddr_atmsvc)` | `Svc::connect` |
| `svc_listen()` | per-`listen()` | `Svc::listen` |
| `svc_accept()` | per-`accept()` | `Svc::accept` |
| `svc_getname()` | per-`getsockname` / `getpeername` | `Svc::getname` |
| `svc_shutdown()` | per-`shutdown` (no-op; kept for compat) | `Svc::shutdown` |
| `svc_setsockopt()` | per-`setsockopt(SO_ATMSAP \| SO_MULTIPOINT)` | `Svc::setsockopt` |
| `svc_getsockopt()` | per-`getsockopt(SO_ATMSAP)` | `Svc::getsockopt` |
| `svc_change_qos()` | per-mid-call QoS modify | `Svc::change_qos` |
| `svc_addparty()` | per-`ATM_ADDPARTY` (P2MP join) | `Svc::addparty` |
| `svc_dropparty()` | per-`ATM_DROPPARTY` (P2MP leave) | `Svc::dropparty` |
| `svc_ioctl()` / `svc_compat_ioctl()` | per-ioctl dispatch | `Svc::ioctl` / `compat_ioctl` |
| `svc_proto_ops` | per-`struct proto_ops` vtable | `Svc::OPS` |
| `svc_family_ops` | per-`struct net_proto_family` | `Svc::FAMILY` |
| `atmsvc_init()` | per-module init (sock_register PF_ATMSVC) | `Svc::init` |
| `atmsvc_exit()` | per-module exit | `Svc::exit` |
| `vcc_create()` | per-VCC alloc (common with PVC) | shared |
| `vcc_release()` | per-VCC free | shared |
| `vcc_connect()` | per-itf/vpi/vci attach | shared |
| `vcc_insert_socket()` | per-listen-socket registration | shared |
| `vcc_sendmsg` / `vcc_recvmsg` / `vcc_poll` | per-data-path | shared |
| `as_bind` / `as_connect` / `as_listen` / `as_accept` / `as_reject` / `as_close` / `as_modify` / `as_addparty` / `as_dropparty` / `as_indicate` | per-`enum atmsvc_msg_type` | `AsMsg::*` |
| `struct atmsvc_msg` | per-userspace daemon wire format | `AtmsvcMsg` |
| `struct sockaddr_atmsvc` | per-AESA address (AF_ATMSVC) | `SockaddrAtmsvc` |
| `ATM_VF_BOUND` / `_HASQOS` / `_HASSAP` / `_LISTEN` / `_SESSION` / `_REGIS` / `_RELEASED` / `_CLOSE` / `_WAITING` / `_READY` | per-VCC flag bits | `AtmVf` |

## Compatibility contract

REQ-1: PF_ATMSVC initialisation:
- atmsvc_init: sock_register(&svc_family_ops) with .family = PF_ATMSVC, .create = svc_create.
- atmsvc_exit: sock_unregister(PF_ATMSVC).
- Per-AF_ATMSVC only init_net is supported (svc_create rejects with -EAFNOSUPPORT for non-init_net).

REQ-2: svc_create(net, sock, protocol, kern):
- Require net_eq(net, &init_net) else -EAFNOSUPPORT.
- sock.ops = &svc_proto_ops.
- err = vcc_create(net, sock, protocol, AF_ATMSVC, kern); propagate on error.
- ATM_SD(sock).local.sas_family = AF_ATMSVC; ATM_SD(sock).remote.sas_family = AF_ATMSVC.
- Per-vcc_create: allocs struct atm_vcc, attaches to sk, initialises atm_qos defaults, sets sk_family = PF_ATMSVC.

REQ-3: svc_proto_ops vtable:
- .family = PF_ATMSVC, .owner = THIS_MODULE.
- .release = svc_release, .bind = svc_bind, .connect = svc_connect.
- .socketpair = sock_no_socketpair.
- .accept = svc_accept, .getname = svc_getname.
- .poll = vcc_poll, .ioctl = svc_ioctl, .compat_ioctl = svc_compat_ioctl (CONFIG_COMPAT).
- .gettstamp = sock_gettstamp.
- .listen = svc_listen, .shutdown = svc_shutdown.
- .setsockopt = svc_setsockopt, .getsockopt = svc_getsockopt.
- .sendmsg = vcc_sendmsg, .recvmsg = vcc_recvmsg (shared with PVC).
- .mmap = sock_no_mmap.

REQ-4: svc_shutdown(sock, how):
- Returns 0 unconditionally (Q.2931 has its own teardown via svc_release / as_close).

REQ-5: svc_release(sock):
- If sk != NULL:
  - vcc = ATM_SD(sock).
  - clear_bit(ATM_VF_READY, &vcc.flags).
  - svc_disconnect(vcc).
  - vcc_release(sock) — shared free path.

REQ-6: svc_disconnect(vcc):
- If test_bit(ATM_VF_REGIS, &vcc.flags):
  - sigd_enq(vcc, as_close, NULL, NULL, NULL).
  - Loop: prepare_to_wait(sk_sleep(sk), TASK_UNINTERRUPTIBLE); break if ATM_VF_RELEASED set or sigd == NULL; schedule().
  - finish_wait.
- Drain sk.sk_receive_queue of pending listen-indications: atm_return(vcc, skb.truesize); sigd_enq2(NULL, as_reject, vcc, NULL, NULL, &vcc.qos, 0); dev_kfree_skb(skb).
- clear_bit(ATM_VF_REGIS).

REQ-7: svc_bind(sock, sockaddr, sockaddr_len):
- Require sockaddr_len == sizeof(struct sockaddr_atmsvc) else -EINVAL.
- lock_sock(sk).
- Require sock.state ∈ {SS_UNCONNECTED} (SS_CONNECTED ⟹ -EISCONN, other ⟹ -EINVAL).
- Require addr.sas_family == AF_ATMSVC else -EAFNOSUPPORT.
- Require test_bit(ATM_VF_HASQOS, &vcc.flags) else -EBADFD (must SO_ATMQOS first).
- clear_bit(ATM_VF_BOUND) (rebind kills old binding).
- vcc.local = *addr (full 20-byte AESA: 13-byte prefix + 6-byte ESI + 1-byte SEL).
- set_bit(ATM_VF_WAITING).
- sigd_enq(vcc, as_bind, NULL, NULL, &vcc.local).
- Loop prepare_to_wait(TASK_UNINTERRUPTIBLE) until !ATM_VF_WAITING ∨ !sigd; schedule().
- finish_wait; clear_bit(ATM_VF_REGIS).
- If !sigd ⟹ -EUNATCH.
- If !sk.sk_err: set_bit(ATM_VF_BOUND).
- release_sock; return -sk.sk_err.

REQ-8: svc_connect(sock, sockaddr, sockaddr_len, flags):
- Require sockaddr_len == sizeof(struct sockaddr_atmsvc) else -EINVAL.
- Per-state machine:
  - SS_CONNECTED ⟹ -EISCONN.
  - SS_CONNECTING ∧ ATM_VF_WAITING ⟹ -EALREADY (async in-flight).
  - SS_CONNECTING ∧ !ATM_VF_WAITING: read sk.sk_err; reset to SS_UNCONNECTED.
  - SS_UNCONNECTED: continue to setup.
  - default ⟹ -EINVAL.
- Require addr.sas_family == AF_ATMSVC.
- Require test_bit(ATM_VF_HASQOS) (must SO_ATMQOS first).
- Reject vcc.qos.{tx,rx}tp.traffic_class == ATM_ANYCLASS.
- Reject both tx and rx traffic_class == 0.
- vcc.remote = *addr.
- set_bit(ATM_VF_WAITING).
- sigd_enq(vcc, as_connect, NULL, NULL, &vcc.remote).
- If O_NONBLOCK: sock.state = SS_CONNECTING; return -EINPROGRESS.
- Else: prepare_to_wait(TASK_INTERRUPTIBLE) loop while ATM_VF_WAITING ∧ sigd:
  - On signal_pending: enq(as_close), wait for !ATM_VF_WAITING, then for ATM_VF_RELEASED; clear REGIS/RELEASED/CLOSE; return -EINTR.
- finish_wait; if !sigd ⟹ -EUNATCH; if sk.sk_err ⟹ -sk.sk_err.
- vcc.qos.txtp.max_pcr = SELECT_TOP_PCR(vcc.qos.txtp); zero out pcr and min_pcr.
- err = vcc_connect(sock, vcc.itf, vcc.vpi, vcc.vci); on success sock.state = SS_CONNECTED.
- On error: svc_disconnect(vcc).

REQ-9: svc_listen(sock, backlog):
- lock_sock(sk).
- Reject if ATM_VF_SESSION (P2MP source can't listen) — -EINVAL.
- Reject if already ATM_VF_LISTEN — -EADDRINUSE.
- set_bit(ATM_VF_WAITING).
- sigd_enq(vcc, as_listen, NULL, NULL, &vcc.local).
- Wait loop TASK_UNINTERRUPTIBLE until !ATM_VF_WAITING ∨ !sigd; schedule().
- If !sigd ⟹ -EUNATCH.
- set_bit(ATM_VF_LISTEN).
- vcc_insert_socket(sk) — register on the listen hash.
- sk.sk_max_ack_backlog = backlog > 0 ? backlog : ATM_BACKLOG_DEFAULT.
- return -sk.sk_err.

REQ-10: svc_accept(sock, newsock, arg):
- lock_sock(sk).
- err = svc_create(sock_net(sk), newsock, 0, arg.kern) — fresh child socket inherits PF_ATMSVC.
- new_vcc = ATM_SD(newsock).
- Loop:
  - prepare_to_wait(TASK_INTERRUPTIBLE).
  - Inner loop dequeues sk_receive_queue (sigd posts an `atmsvc_msg` setup-indication via as_indicate); meanwhile check ATM_VF_RELEASED / ATM_VF_CLOSE / O_NONBLOCK / signal_pending.
  - msg = (struct atmsvc_msg *) skb.data.
  - new_vcc.qos = msg.qos; set_bit(ATM_VF_HASQOS).
  - new_vcc.remote = msg.svc; new_vcc.local = msg.local; new_vcc.sap = msg.sap.
  - err = vcc_connect(newsock, msg.pvc.sap_addr.itf, .vpi, .vci).
  - On error: sigd_enq2(NULL, as_reject, old_vcc, NULL, NULL, &old_vcc.qos, error); dev_kfree_skb; error == -EAGAIN ⟹ -EBUSY.
  - On success: dev_kfree_skb; sk_acceptq_removed(sk).
  - set_bit(ATM_VF_WAITING, &new_vcc.flags).
  - sigd_enq(new_vcc, as_accept, old_vcc, NULL, NULL).
  - Wait for !ATM_VF_WAITING on new_vcc (TASK_UNINTERRUPTIBLE; release_sock around schedule()).
  - If !sigd ⟹ -EUNATCH.
  - If sk_atm(new_vcc).sk_err == 0: break out of outer loop (success).
  - Else if sk_err != ERESTARTSYS: -sk_err; goto out.
  - Else: retry — another incoming setup waits.
- newsock.state = SS_CONNECTED.

REQ-11: svc_getname(sock, sockaddr, peer):
- memcpy(sockaddr, peer ? &ATM_SD(sock).remote : &ATM_SD(sock).local, sizeof(struct sockaddr_atmsvc)).
- Return sizeof(struct sockaddr_atmsvc).

REQ-12: svc_change_qos(vcc, qos):
- set_bit(ATM_VF_WAITING).
- sigd_enq2(vcc, as_modify, NULL, NULL, &vcc.local, qos, 0).
- Wait TASK_UNINTERRUPTIBLE until !ATM_VF_WAITING ∨ ATM_VF_RELEASED ∨ !sigd.
- !sigd ⟹ -EUNATCH; else return -sk.sk_err.

REQ-13: svc_setsockopt(sock, level, optname, optval, optlen):
- SO_ATMSAP (level == SOL_ATM, optlen == sizeof(struct atm_sap)):
  - copy_from_sockptr(&vcc.sap, optval, optlen); -EFAULT on fault.
  - set_bit(ATM_VF_HASSAP).
- SO_MULTIPOINT (level == SOL_ATM, optlen == sizeof(int)):
  - copy_from_sockptr(&value, optval, sizeof(int)).
  - value == 1 ⟹ set_bit(ATM_VF_SESSION).
  - value == 0 ⟹ clear_bit(ATM_VF_SESSION).
  - else -EINVAL.
- default: fall through to vcc_setsockopt (shared).

REQ-14: svc_getsockopt(sock, level, optname, optval, optlen):
- If !__SO_LEVEL_MATCH(optname, level) ∨ optname != SO_ATMSAP: vcc_getsockopt.
- get_user(len, optlen); require len == sizeof(struct atm_sap).
- copy_to_user(optval, &ATM_SD(sock).sap, sizeof(struct atm_sap)).

REQ-15: svc_addparty(sock, sockaddr, sockaddr_len, flags) — P2MP join:
- lock_sock; set_bit(ATM_VF_WAITING).
- sigd_enq(vcc, as_addparty, NULL, NULL, (struct sockaddr_atmsvc *) sockaddr).
- If O_NONBLOCK ⟹ -EINPROGRESS.
- Wait TASK_INTERRUPTIBLE until !ATM_VF_WAITING ∨ !sigd.
- error = -xchg(&sk.sk_err_soft, 0).
- release_sock.

REQ-16: svc_dropparty(sock, ep_ref) — P2MP leave:
- lock_sock; set_bit(ATM_VF_WAITING).
- sigd_enq2(vcc, as_dropparty, NULL, NULL, NULL, NULL, ep_ref).
- Wait TASK_INTERRUPTIBLE until !ATM_VF_WAITING ∨ !sigd.
- !sigd ⟹ -EUNATCH.
- error = -xchg(&sk.sk_err_soft, 0).

REQ-17: svc_ioctl(sock, cmd, arg):
- ATM_ADDPARTY: require ATM_VF_SESSION; copy_from_user(sockaddr_atmsvc); svc_addparty.
- ATM_DROPPARTY: require ATM_VF_SESSION; copy_from_user(int ep_ref); svc_dropparty.
- default: vcc_ioctl(sock, cmd, arg).
- Compat-ioctl rewrites COMPAT_ATM_ADDPARTY → ATM_ADDPARTY (sockaddr_atmsvc has no 32/64 layout split) then dispatches; other cmds → vcc_compat_ioctl.

REQ-18: Daemon-death contract (sigd == NULL):
- Every wait loop short-circuits on `!sigd`.
- Bind / listen / connect / change_qos / dropparty return -EUNATCH.
- disconnect drains receive_queue and rejects pending indications.
- accept returns -EUNATCH after queue drains.

## Acceptance Criteria

- [ ] AC-1: socket(PF_ATMSVC, SOCK_DGRAM, 0) on non-init netns ⟹ -EAFNOSUPPORT.
- [ ] AC-2: bind without prior atmlib SO_ATMQOS ⟹ -EBADFD.
- [ ] AC-3: bind to addr.sas_family != AF_ATMSVC ⟹ -EAFNOSUPPORT.
- [ ] AC-4: bind with sockaddr_len != sizeof(sockaddr_atmsvc) ⟹ -EINVAL.
- [ ] AC-5: bind on already-connected socket ⟹ -EISCONN.
- [ ] AC-6: connect from SS_CONNECTING with ATM_VF_WAITING ⟹ -EALREADY.
- [ ] AC-7: connect O_NONBLOCK ⟹ -EINPROGRESS; later getsockopt(SO_ERROR) yields result.
- [ ] AC-8: connect interrupted by signal triggers as_close handshake and ⟹ -EINTR.
- [ ] AC-9: listen twice ⟹ -EADDRINUSE.
- [ ] AC-10: listen on ATM_VF_SESSION socket ⟹ -EINVAL.
- [ ] AC-11: accept successfully creates child socket with msg.qos / msg.svc / msg.local / msg.sap inherited.
- [ ] AC-12: accept failure (vcc_connect error) sends as_reject and returns -EBUSY when underlying err == -EAGAIN.
- [ ] AC-13: setsockopt(SOL_ATM, SO_ATMSAP, …) sets ATM_VF_HASSAP and stores 8-byte B-HLI / B-LLI.
- [ ] AC-14: setsockopt(SOL_ATM, SO_MULTIPOINT, 1) sets ATM_VF_SESSION.
- [ ] AC-15: ATM_ADDPARTY without ATM_VF_SESSION ⟹ -EINVAL.
- [ ] AC-16: ATM_DROPPARTY without ATM_VF_SESSION ⟹ -EINVAL.
- [ ] AC-17: All blocking wait loops abort with -EUNATCH when sigd becomes NULL.
- [ ] AC-18: getname returns local on peer==0, remote on peer==1, len = sizeof(sockaddr_atmsvc).
- [ ] AC-19: svc_change_qos issues as_modify and propagates daemon errno via sk.sk_err.

## Architecture

```
struct AtmVcc {
  sk: Sock,                           /* shared first member */
  flags: AtomicUsize,                 /* ATM_VF_* bits */
  qos: AtmQos,                        /* tx/rx traffic parameters */
  sap: AtmSap,                        /* B-HLI / B-LLI */
  local: SockaddrAtmsvc,              /* AESA: 13-prefix + 6-ESI + 1-SEL */
  remote: SockaddrAtmsvc,
  itf: i16,
  vpi: i16,
  vci: i32,
  push: fn(&AtmVcc, &SkBuff),
  pop: fn(&AtmVcc, &SkBuff),
  ...
}

bitflags AtmVf {
  ADDR     = 1 << 0,
  READY    = 1 << 1,
  PARTIAL  = 1 << 2,
  REGIS    = 1 << 3,
  RELEASED = 1 << 4,
  HASQOS   = 1 << 5,
  LISTEN   = 1 << 6,
  META     = 1 << 7,
  BOUND    = 1 << 8,
  CLOSE    = 1 << 9,
  HASSAP   = 1 << 10,
  WAITING  = 1 << 11,
  SESSION  = 1 << 12,
  ...
}

enum AsMsg { Bind, Connect, Listen, Accept, Reject, Close, Modify, Indicate, AddParty, DropParty }

struct AtmsvcMsg {
  hdr: AtmsvcMsgHeader { msg_type, vcc_id, listen_vcc_id, reply, sap, qos, ... },
  svc: SockaddrAtmsvc,
  local: SockaddrAtmsvc,
  pvc: AtmPvcAddr { sap_addr: { itf, vpi, vci }, ... },
  session_num: i32,
}
```

`Svc::create(net, sock, protocol, kern) -> Result<()>`:
1. If net != init_net: Err(-EAFNOSUPPORT).
2. sock.ops = &OPS.
3. vcc_create(net, sock, protocol, AF_ATMSVC, kern)?.
4. ATM_SD(sock).local.sas_family = AF_ATMSVC.
5. ATM_SD(sock).remote.sas_family = AF_ATMSVC.

`Svc::bind(sock, sockaddr, sockaddr_len) -> Result<()>`:
1. If sockaddr_len != sizeof(SockaddrAtmsvc): Err(-EINVAL).
2. lock_sock(sk).
3. Match sock.state: SS_CONNECTED ⟹ Err(-EISCONN); !SS_UNCONNECTED ⟹ Err(-EINVAL).
4. addr = sockaddr as SockaddrAtmsvc.
5. If addr.sas_family != AF_ATMSVC: Err(-EAFNOSUPPORT).
6. clear_bit(ATM_VF_BOUND).
7. If !test_bit(ATM_VF_HASQOS): Err(-EBADFD).
8. vcc.local = *addr.
9. set_bit(ATM_VF_WAITING).
10. Atm::sigd_enq(vcc, AsMsg::Bind, None, None, Some(&vcc.local)).
11. Wait TASK_UNINTERRUPTIBLE until !test_bit(ATM_VF_WAITING) ∨ Atm::sigd.is_none().
12. clear_bit(ATM_VF_REGIS).
13. If sigd.is_none(): Err(-EUNATCH).
14. If sk.sk_err == 0: set_bit(ATM_VF_BOUND).
15. release_sock; Result::from_err(sk.sk_err).

`Svc::connect(sock, sockaddr, sockaddr_len, flags) -> Result<()>`:
1. Validate sockaddr_len, sock.state, addr family.
2. Require ATM_VF_HASQOS.
3. Reject ATM_ANYCLASS or both-zero traffic_class.
4. vcc.remote = *addr; set_bit(ATM_VF_WAITING).
5. Atm::sigd_enq(vcc, AsMsg::Connect, None, None, Some(&vcc.remote)).
6. If O_NONBLOCK: sock.state = SS_CONNECTING; return Err(-EINPROGRESS).
7. Wait TASK_INTERRUPTIBLE handling signal abort (as_close handshake → -EINTR).
8. If sigd.is_none(): Err(-EUNATCH); if sk.sk_err != 0: Err(-sk.sk_err).
9. vcc.qos.txtp.max_pcr = SELECT_TOP_PCR(vcc.qos.txtp); zero pcr, min_pcr.
10. vcc_connect(sock, vcc.itf, vcc.vpi, vcc.vci)?; sock.state = SS_CONNECTED.
11. On any error past step 5: Svc::disconnect(vcc).

`Svc::listen(sock, backlog) -> Result<()>`:
1. lock_sock; reject if ATM_VF_SESSION (Err(-EINVAL)) or already ATM_VF_LISTEN (Err(-EADDRINUSE)).
2. set_bit(ATM_VF_WAITING).
3. Atm::sigd_enq(vcc, AsMsg::Listen, None, None, Some(&vcc.local)).
4. Wait TASK_UNINTERRUPTIBLE.
5. If sigd.is_none(): Err(-EUNATCH).
6. set_bit(ATM_VF_LISTEN); vcc_insert_socket(sk).
7. sk.sk_max_ack_backlog = if backlog > 0 { backlog } else { ATM_BACKLOG_DEFAULT }.

`Svc::accept(sock, newsock, arg) -> Result<()>`:
1. lock_sock(sk); Svc::create(sock_net(sk), newsock, 0, arg.kern)?.
2. new_vcc = ATM_SD(newsock).
3. Loop:
   - Wait + dequeue from sk.sk_receive_queue (as_indicate msgs).
   - msg = (AtmsvcMsg *) skb.data; populate new_vcc.{qos, remote, local, sap}; set_bit(ATM_VF_HASQOS).
   - vcc_connect(newsock, msg.pvc.sap_addr.itf, .vpi, .vci).
   - On error: sigd_enq2(None, AsMsg::Reject, old_vcc, None, None, &old_vcc.qos, error); -EAGAIN → -EBUSY.
   - On success: dev_kfree_skb; sk_acceptq_removed(sk).
   - set_bit(ATM_VF_WAITING, &new_vcc.flags); sigd_enq(new_vcc, AsMsg::Accept, old_vcc, None, None).
   - Wait TASK_UNINTERRUPTIBLE on new_vcc; release_sock around schedule().
   - Validate sigd; break if sk_err == 0; if sk_err != ERESTARTSYS: Err(-sk_err); else retry.
4. newsock.state = SS_CONNECTED.

`Svc::disconnect(vcc)`:
1. If test_bit(ATM_VF_REGIS):
   - sigd_enq(vcc, AsMsg::Close, None, None, None).
   - Wait TASK_UNINTERRUPTIBLE until ATM_VF_RELEASED ∨ !sigd.
2. While let Some(skb) = sk.sk_receive_queue.dequeue():
   - atm_return(vcc, skb.truesize); sigd_enq2(None, AsMsg::Reject, vcc, None, None, &vcc.qos, 0); dev_kfree_skb(skb).
3. clear_bit(ATM_VF_REGIS).

`Svc::change_qos(vcc, qos) -> Result<()>`:
1. set_bit(ATM_VF_WAITING).
2. sigd_enq2(vcc, AsMsg::Modify, None, None, &vcc.local, qos, 0).
3. Wait TASK_UNINTERRUPTIBLE until !ATM_VF_WAITING ∨ ATM_VF_RELEASED ∨ !sigd.
4. !sigd ⟹ Err(-EUNATCH); else Result::from_err(sk.sk_err).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `init_net_only` | INVARIANT | per-create: net == init_net else -EAFNOSUPPORT. |
| `sockaddr_atmsvc_len` | INVARIANT | per-bind/connect: sockaddr_len == sizeof(SockaddrAtmsvc). |
| `family_af_atmsvc` | INVARIANT | per-bind/connect: addr.sas_family == AF_ATMSVC. |
| `hasqos_before_bind_or_connect` | INVARIANT | per-bind/connect: ATM_VF_HASQOS required. |
| `wait_aborts_on_sigd_null` | INVARIANT | every wait loop exits when Atm::sigd == None. |
| `listen_not_session` | INVARIANT | per-listen: !ATM_VF_SESSION. |
| `listen_idempotent_failure` | INVARIANT | per-listen: ATM_VF_LISTEN ⟹ -EADDRINUSE. |
| `accept_reject_on_vcc_connect_err` | INVARIANT | per-accept: vcc_connect err ⟹ as_reject + skb freed. |
| `addparty_requires_session` | INVARIANT | per-ioctl ATM_ADDPARTY: ATM_VF_SESSION required. |
| `dropparty_requires_session` | INVARIANT | per-ioctl ATM_DROPPARTY: ATM_VF_SESSION required. |
| `connect_signal_abort_consistent` | INVARIANT | per-connect signal: as_close issued and ATM_VF_RELEASED awaited. |
| `setsockopt_optlen_strict` | INVARIANT | SO_ATMSAP optlen == sizeof(atm_sap); SO_MULTIPOINT optlen == sizeof(int). |
| `vcc_local_remote_sas_family_set` | INVARIANT | per-create: local.sas_family = remote.sas_family = AF_ATMSVC. |

### Layer 2: TLA+

`net/atm/svc.tla`:
- States: Created, Bound, Listening, Connecting, Connected, Closed.
- Daemon: SigdAlive(queue) | SigdDead.
- Events: Bind, Connect(nonblock), Listen, AsIndicate(msg), Accept, AsAccept, AsReject, AsClose, AsModify, SignalInt, SigdDie.
- Properties:
  - `safety_bind_before_connect_allowed` — bind ⟹ Bound; connect ⟹ Connecting → Connected.
  - `safety_no_listen_in_session` — ATM_VF_SESSION ⟹ no Listening transition.
  - `safety_accept_invariant` — Listening + AsIndicate ⟹ Accept yields child Connected ∨ Reject.
  - `safety_sigd_die_aborts_all` — SigdDie ⟹ every blocked op transitions to err=-EUNATCH.
  - `safety_signal_abort_clean` — connect+SignalInt ⟹ AsClose issued and ATM_VF_RELEASED awaited; result -EINTR.
  - `liveness_connect_terminates` — SigdAlive ⟹ Connecting eventually Connected ∨ err.
  - `liveness_accept_terminates` — Listening with backlog ⟹ Accept eventually Connected ∨ err.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Svc::create` post: ret ⟹ local.sas_family == remote.sas_family == AF_ATMSVC | `Svc::create` |
| `Svc::bind` post: ret ⟹ ATM_VF_BOUND or err mapped to sk.sk_err | `Svc::bind` |
| `Svc::connect` post: ret_ok ⟹ sock.state == SS_CONNECTED ∧ vcc.{itf,vpi,vci} attached | `Svc::connect` |
| `Svc::listen` post: ret ⟹ ATM_VF_LISTEN ∧ vcc_insert_socket succeeded | `Svc::listen` |
| `Svc::accept` post: ret ⟹ newsock.state == SS_CONNECTED ∧ msg.skb freed | `Svc::accept` |
| `Svc::disconnect` post: !ATM_VF_REGIS ∧ sk_receive_queue empty | `Svc::disconnect` |
| `Svc::change_qos` post: as_modify enqueued; result reflects sk.sk_err | `Svc::change_qos` |
| `Svc::setsockopt` post: SO_ATMSAP ⟹ ATM_VF_HASSAP; SO_MULTIPOINT(1) ⟹ ATM_VF_SESSION | `Svc::setsockopt` |
| `Svc::ioctl` post: ADDPARTY/DROPPARTY guarded by ATM_VF_SESSION | `Svc::ioctl` |

### Layer 4: Verus/Creusot functional

`Per-bind → as_bind → sigd-reply → BOUND` / `Per-listen → as_listen → LISTEN + insert` / `Per-accept → as_indicate dequeue → vcc_connect → as_accept → SS_CONNECTED` semantic equivalence: per-ITU-T Q.2931 message-flow + per-`atmsigd` daemon protocol (linux-atm `signaling/` package). Mid-call as_modify renegotiates `atm_qos` without VCC teardown.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

ATM SVC reinforcement:

- **Per-`init_net`-only family** — defense against per-netns isolation bypass for legacy ATM stack.
- **Per-`CAP_NET_ADMIN`-equivalent gating** through atmsigd attach prerequisite (`sigd_attach`) — defense against per-unprivileged daemon-spoof.
- **Per-`ATM_VF_HASQOS` precondition for bind/connect** — defense against per-unconfigured-traffic-class call setup.
- **Per-strict `sockaddr_atmsvc` length check** — defense against per-truncated-AESA buffer-confusion.
- **Per-`sas_family == AF_ATMSVC` check** — defense against per-cross-family address-confusion.
- **Per-sigd-death wait-loop short-circuit (-EUNATCH)** — defense against per-indefinite-hang on daemon crash.
- **Per-signal abort triggers as_close handshake** — defense against per-orphaned-VCC at switch.
- **Per-`ATM_VF_RELEASED` await before flag-reset** — defense against per-reuse-while-tearing-down race.
- **Per-`sk_acceptq_removed` on each accept** — defense against per-backlog-counter drift.
- **Per-as_reject on accept failure path** — defense against per-switch-state-leak (VC stuck SETUP_PENDING).
- **Per-`vcc_insert_socket` only after sigd OK for listen** — defense against per-half-installed-listener.
- **Per-`SELECT_TOP_PCR` clamp on connect-success** — defense against per-uninitialised-PCR enqueue.
- **Per-`ATM_VF_SESSION` gating for ADDPARTY / DROPPARTY** — defense against per-non-P2MP party-modify.
- **Per-`receive_queue` drain + reject in svc_disconnect** — defense against per-stale-indication leak.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/atm/common.c shared VCC create/release/data-path (covered separately if expanded)
- net/atm/pvc.c PF_ATMPVC permanent-VC sockets (covered separately if expanded)
- net/atm/signaling.c sigd VCC plumbing internals (covered separately if expanded)
- net/atm/addr.c ESI / AESA cache (covered separately if expanded)
- net/atm/resources.c per-itf / vpi / vci allocation (covered separately if expanded)
- net/atm/br2684.c / clip.c / lec.c / mpc.c higher-layer ATM services (covered separately if expanded)
- atmsigd userspace daemon (out of kernel scope)
- ITU-T Q.2931 / Q.2971 signalling state machine internals (lives in userspace daemon)
- Implementation code
