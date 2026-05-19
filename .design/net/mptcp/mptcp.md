# Tier-3: net/mptcp/protocol.c — MPTCP (Multipath TCP) RFC 8684 protocol

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/mptcp/protocol.c (~4710 lines)
  - net/mptcp/protocol.h
  - net/mptcp/subflow.c
  - net/mptcp/pm_netlink.c (path manager)
  - net/mptcp/options.c (TCP options handling)
  - net/mptcp/{token, mib, sockopt, fastopen, diag}.c
  - include/uapi/linux/mptcp.h
-->

## Summary

MPTCP (Multipath TCP; RFC 8684) extends TCP to use multiple parallel paths simultaneously over a single logical connection. Per-MPTCP `msk` (master socket) is the user-facing socket; per-subflow `ssk` is per-path TCP-like connection. Per-MPTCP-INIT: SYN with MP_CAPABLE option negotiates token + keys. Per-additional-subflow: SYN with MP_JOIN option authenticates via HMAC. Per-data sent across subflows is re-assembled at receive via per-subflow DSS (Data Sequence Signal) mapping. Per-path-manager (kernel-netlink or BPF) dynamically adds/removes subflows. Critical for: mobile (Wi-Fi + cellular), datacenter ECMP-aware, high-throughput multi-NIC.

This Tier-3 covers `protocol.c` (~4710 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct mptcp_sock` | per-master socket | `MptcpSock` |
| `struct mptcp_subflow_context` | per-subflow state on TCP | `MptcpSubflowContext` |
| `struct mptcp_pm_data` | per-path-manager state | `MptcpPmData` |
| `mptcp_init_sock()` | per-msk init | `Mptcp::init_sock` |
| `mptcp_close()` | per-msk close | `Mptcp::close` |
| `mptcp_sendmsg()` | per-msk send dispatched across subflows | `Mptcp::sendmsg` |
| `mptcp_recvmsg()` | per-msk recv merged from subflows | `Mptcp::recvmsg` |
| `mptcp_subflow_create()` | per-subflow create | `Subflow::create` |
| `mptcp_subflow_init()` | per-subflow init | `Subflow::init` |
| `mptcp_subflow_handle_join()` | MP_JOIN handshake | `Subflow::handle_join` |
| `mptcp_data_ready()` | per-subflow data-ready notify | `Mptcp::data_ready` |
| `mptcp_push_pending()` | per-msk push send-queue across subflows | `Mptcp::push_pending` |
| `mptcp_sendmsg_frag()` | per-frag-send | `Mptcp::sendmsg_frag` |
| `mptcp_pm_add_addr()` / `_rm_addr()` | path-manager netlink | `Pm::add_addr` / `rm_addr` |
| `mptcp_pm_create_subflow_or_signal_addr()` | per-PM trigger | `Pm::create_subflow_or_signal_addr` |
| `MPTCP_MIB_*` | per-counter | UAPI |
| `MPTCPOPT_*` | per-TCP-option type | shared |

## Compatibility contract

REQ-1: socket(AF_INET, SOCK_STREAM, IPPROTO_MPTCP):
- Returns per-MPTCP fd backed by mptcp_sock.
- Per-msk has first_subflow (initial TCP connection).

REQ-2: MP_CAPABLE option (handshake):
- SYN: MP_CAPABLE with sender-key (64-bit).
- SYN-ACK: MP_CAPABLE with receiver-key.
- ACK: MP_CAPABLE confirm.
- Per-keys derive: token, idsn (Initial Data Sequence Number).
- Per-pair: msk.local_key + msk.remote_key.

REQ-3: MP_JOIN option (additional subflow):
- SYN: MP_JOIN with token + nonce.
- SYN-ACK: MP_JOIN with HMAC + nonce.
- ACK: MP_JOIN with HMAC.
- Per-HMAC authenticates: SHA-256-HMAC(key, nonces).
- Per-token resolves to existing msk.

REQ-4: DSS (Data Sequence Signal) option:
- Per-skb TCP-option carries per-msk DSN (Data Sequence Number).
- Per-ack: per-msk DACK (Data ACK).
- Per-subflow has its own TCP-seq + DSN-mapping.
- Per-receiver: re-orders per-DSN to per-msk receive-queue.

REQ-5: ADD_ADDR / REMOVE_ADDR options:
- Per-msk advertises additional local addresses.
- Per-PM: kernel decides ADD; userspace can override via netlink.
- HMAC-protected.

REQ-6: Per-PM (path-manager):
- MPTCP_PM_TYPE_KERNEL: in-kernel default.
- MPTCP_PM_TYPE_USERSPACE: netlink-driven by daemon.
- MPTCP_PM_TYPE_BPF: BPF program decides.

REQ-7: Per-PM per-userspace ABI:
- /sys/kernel/mptcp/* knobs.
- ip mptcp endpoint show: per-endpoint addresses.
- ip mptcp limits: per-net subflow + endpoint limits.

REQ-8: mptcp_sendmsg:
- Append to per-msk send-queue.
- mptcp_push_pending: distribute pending across per-subflow sockets.
- Per-subflow: tcp_sendmsg from per-frag.
- Per-skb has mpext (DSN + flags).

REQ-9: mptcp_recvmsg:
- Per-subflow data-ready triggers mptcp_data_ready.
- Per-subflow's skbs re-ordered into msk.receive_queue by DSN.
- recvmsg drains msk.receive_queue.

REQ-10: Per-subflow state:
- mptcp_subflow_context attached to per-subflow's tcp_sock.
- mp_capable / mp_join.
- token + idsn + map_seq + ssn_offset.

REQ-11: Per-fastopen:
- mptcp_sendmsg_fastopen: TFO + MP_CAPABLE init in one round-trip.

REQ-12: Per-namespace:
- Per-net MPTCP instance; per-net sysctl knobs.

## Acceptance Criteria

- [ ] AC-1: socket(AF_INET, SOCK_STREAM, IPPROTO_MPTCP): succeeds.
- [ ] AC-2: connect to MPTCP-capable peer: MP_CAPABLE handshake; msk established.
- [ ] AC-3: Per-Wi-Fi-up event: PM adds ADD_ADDR; peer initiates MP_JOIN.
- [ ] AC-4: Per-additional-subflow: SYN with MP_JOIN + HMAC; ACK + HMAC confirms.
- [ ] AC-5: sendmsg distributes across subflows: per-DSN segments assigned.
- [ ] AC-6: recvmsg: per-DSN order delivered.
- [ ] AC-7: Per-subflow loss: TCP retransmit on that subflow only.
- [ ] AC-8: REMOVE_ADDR: per-removed-subflow cleaned up.
- [ ] AC-9: PM userspace via netlink: per-policy controlled.
- [ ] AC-10: PM BPF: per-program controls.
- [ ] AC-11: mptcp_close: per-msk + per-subflow gracefully closed.

## Architecture

Per-msk:

```
struct MptcpSock {
  sk: InetConnectionSock,                         // base AF_INET
  ack_seq: u64,                                   // master-seq receive
  rcv_wnd_sent: u64,
  rcv_data_fin_seq: u64,
  snd_una: u64,                                   // master-seq sent
  snd_nxt: u64,
  write_seq: u64,
  local_key: u64,
  remote_key: u64,
  token: u32,
  idsn: u64,                                      // initial DSN
  conn_list: ListHead<&Sock>,                     // subflows
  rcv_pruned: u64,
  pm: MptcpPmData,
  /* DSS */
  recv_pruned: u64,
  ...
}

struct MptcpSubflowContext {
  conn_list: ListLink,
  tcp_sock: *TcpSock,
  master: *MptcpSock,
  request_mptcp: bool,
  fully_established: bool,
  mp_capable: bool,
  mp_join: bool,
  data_avail: bool,
  remote_key: u64,
  local_key: u64,
  token: u32,
  thmac: u64,                                     // HMAC for ACK
  remote_nonce: u32,
  local_nonce: u32,
  idsn: u64,
  remote_id: u8,                                  // ADD_ADDR id
  local_id: u8,
  map_subflow_seq: u32,
  map_data_len: u16,
  map_dsn: u64,
  ssn_offset: u32,
}

struct MptcpPmData {
  add_addr_signal: ListHead<MptcpPmAddrEntry>,
  add_addr_accepted: ListHead<MptcpPmAddrEntry>,
  local_addr_used: u8,
  ...
}
```

`Mptcp::init_sock(sk) -> Result<()>`:
1. msk = (MptcpSock*)sk.
2. msk.first_subflow = NULL.
3. Generate local_key (random 64-bit).
4. INIT_LIST_HEAD(&msk.conn_list).
5. Hash key → token.

`Mptcp::sendmsg(sk, msg, len) -> Result<usize>`:
1. msk = (MptcpSock*)sk.
2. /* Buffer skb into msk.send-queue */
3. for each iov: build msk-skb; embed DSN.
4. mptcp_push_pending(msk, ssk, flags) → walk subflows, send per-frag via tcp_sendmsg.

`Mptcp::push_pending(msk)`:
1. for subflow in msk.conn_list:
   - if subflow.backup ∨ !writeable: skip.
   - for skb in msk.send-queue:
     - info = build_sendmsg_info.
     - copy_to_subflow(skb, subflow, &info).

`Mptcp::sendmsg_frag(sk, ssk, dfrag, info)`:
1. /* Per-fragment: build TCP-skb with MP-ext DSN */
2. mpext = mptcp_skb_ext_add(skb).
3. mpext.dsn = info.dsn; mpext.data_len = ...; mpext.mp_capable = ...
4. tcp_sendmsg_locked(ssk, &iter, info.size).

`Subflow::create(msk, local, remote) -> Result<*Sock>`:
1. /* Create per-subflow tcp_sock */
2. ssk = sk_alloc_subflow_sk(net).
3. /* Per-MP_JOIN handshake initiator */
4. mptcp_subflow_init(ssk, msk, &join_info).
5. tcp_connect(ssk).

`Subflow::handle_join(req, ssk)`:
1. /* Authenticate HMAC */
2. compute_hmac(msk.local_key, msk.remote_key, &local_nonce, &remote_nonce, hmac_out).
3. if !memcmp_const_time(req.hmac, hmac_out): reject.
4. /* Add subflow to msk.conn_list */
5. list_add(&subflow.conn_list, &msk.conn_list).

`Mptcp::recvmsg(sk, msg, len, flags, addr_len) -> Result<usize>`:
1. /* msk.receive_queue contains DSN-ordered skbs */
2. /* Dequeue + copy to msg.iov */
3. for skb in msk.receive_queue:
   - copy_to_iter(skb.data, ..., msg.iov).
   - if MSG_PEEK: keep skb; else: drop.

`Mptcp::data_ready(sk)`:
1. /* Per-subflow data-ready callback */
2. subflow = mptcp_subflow_ctx(sk).
3. msk = subflow.master.
4. /* Try to extract DSN map; move into msk.receive_queue */
5. mptcp_can_accept_more_subflows: check limits.
6. wake msk.wait-queue.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mp_capable_or_mp_join` | INVARIANT | per-subflow.mp_capable XOR mp_join. |
| `subflow_in_master_conn_list` | INVARIANT | per-subflow in msk.conn_list. |
| `dsn_monotonic_within_msk` | INVARIANT | per-msk write-DSN strictly-increasing. |
| `hmac_validated_pre_accept` | INVARIANT | per-MP_JOIN: HMAC verified before list-add. |
| `token_unique_per_msk` | INVARIANT | per-net token-hashtable unique per-msk. |

### Layer 2: TLA+

`net/mptcp/protocol.tla`:
- Per-MPTCP handshake + per-subflow add/remove + per-DSN reorder.
- Properties:
  - `safety_no_unauthenticated_subflow` — per-MP_JOIN: HMAC pre-add.
  - `safety_DSN_order_preserved` — per-msk recv: bytes delivered in DSN order.
  - `safety_per_subflow_TCP_invariants` — per-subflow: TCP semantics preserved.
  - `liveness_per_path_failure_retry_on_other` — per-subflow-loss: data retransmits on other subflow.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mptcp::init_sock` post: msk allocated; token unique | `Mptcp::init_sock` |
| `Mptcp::sendmsg` post: per-frag distributed to subflows; DSN-tagged | `Mptcp::sendmsg` |
| `Mptcp::recvmsg` post: per-skb dequeue in DSN-order | `Mptcp::recvmsg` |
| `Subflow::create` post: per-subflow in msk.conn_list after MP_JOIN | `Subflow::create` |
| `Subflow::handle_join` post: HMAC validated; subflow added | `Subflow::handle_join` |

### Layer 4: Verus/Creusot functional

`Per-msk multi-subflow TCP-like reliable byte-stream with per-DSS reordering` semantic equivalence: per-RFC 8684 MPTCP.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

MPTCP-specific reinforcement:

- **Per-HMAC validation pre-add subflow** — defense against per-subflow-hijack.
- **Per-token unique-per-msk** — defense against per-token-collision cross-msk.
- **Per-local-key never logged** — defense against per-key-leak.
- **Per-MP_JOIN rate-limit per-msk** — defense against per-MPTCP-DDoS adding subflows.
- **Per-subflow count cap** — defense against per-msk subflow-bomb.
- **Per-ADD_ADDR HMAC-protected** — defense against per-ADD_ADDR-spoof.
- **Per-namespace MPTCP scoped** — defense against cross-ns leak.
- **Per-CAP_NET_ADMIN for PM userspace** — defense against unprivileged PM-config.
- **Per-subflow recv-buf bound per-msk recv-buf** — defense against per-subflow-buffer overflow.
- **Per-DSN window-tracking** — defense against per-DSN-wrap.
- **Per-MIB counters track misuse** — defense against per-attack-pattern undetected.

## Grsecurity/PaX-style Reinforcement

Rationale: MPTCP layers a multi-subflow connection-management protocol on top of TCP, introducing connection-tokens (per-msk), HMAC-SHA256-keyed MP_JOIN authentication, ADD_ADDR address-advertisement (also HMAC-protected), and a 64-bit data-sequence-number space. Token-table integrity, HMAC-key sanitization, and DSN-window arithmetic all carry the security weight — a token-collision or HMAC-key disclosure pivots to subflow-hijack on any active MPTCP connection.

Baseline (cross-ref `net/00-overview.md` § Hardening):
- **PAX_USERCOPY**: `mptcp_sendmsg` / `mptcp_recvmsg` iter copies; setsockopt MPTCP_INFO / MPTCP_FULL_INFO structs slab-USERCOPY-whitelisted.
- **PAX_KERNEXEC**: `mptcp_prot`, `mptcp_v6_prot`, `mptcp_stream_ops`, `mptcp_pm_ops` placed `__ro_after_init`.
- **PAX_RANDKSTACK**: re-randomise on every `mptcp_recvmsg` / `__mptcp_push_pending` entry.
- **PAX_REFCOUNT**: `mptcp_sock` + per-subflow back-refs saturating; defends mass-subflow-join wrap.
- **PAX_MEMORY_SANITIZE**: freed `mptcp_sock` (incl. embedded `mptcp_pm_data` + `local_key`/`remote_key`) zero-filled with memzero_explicit on key material.
- **PAX_UDEREF**: nl-PM netlink attribute parse via NLA_POLICY only.
- **PAX_RAP / kCFI**: `mptcp_pm_ops`, `mptcp_sched_ops` indirect calls kCFI-tagged.
- **GRKERNSEC_HIDESYM**: `mptcp_sock`, subflow `sock` pointers + token never rendered to `/proc/net/mptcp` (only %pK + token-hash).
- **GRKERNSEC_DMESG**: token-collision and HMAC-fail warns ratelimited.

mptcp-specific reinforcement:
- **MPTCP token CAP_NET_ADMIN gate for userspace PM** — only CAP_NET_ADMIN-holders may inject MPTCP_PM_CMD_* netlink ops (already row-2) — extended with GR-RBAC role gate.
- **HMAC-SHA256 key MEMORY_SANITIZE strict** — `local_key` + `remote_key` memzero_explicit on `mptcp_sock_destruct`.
- **Token-table per-net cap** — refuse > GRSEC_MPTCP_TOKENS_MAX (default 65536) live tokens per netns; defends token-exhaustion DoS.
- **ADD_ADDR HMAC strict** (already row-2) — extended with audit-log on per-msk repeated HMAC-fail beyond threshold.
- **Per-msk subflow count cap** — refuse > GRSEC_MPTCP_SUBFLOW_MAX (default 8) joins; defends subflow-flood DoS.
- **DSN window-tracking** (already row-2) — wrap-detect uses saturating arithmetic; audit-log on suspected wrap.
- **Cross-namespace token-lookup refused** — token-table strictly per-net.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/mptcp/{subflow, pm_netlink, options, token, mib, sockopt, fastopen, diag}.c (covered separately if expanded)
- BPF-PM (covered separately)
- ss / iproute2 mptcp tools (out-of-tree)
- Implementation code
