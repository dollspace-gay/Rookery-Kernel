# Tier-3: net/tipc/socket.c — TIPC (Transparent Inter-Process Communication) socket family

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/tipc/socket.c (~4010 lines)
  - net/tipc/core.c (~229 lines)
  - net/tipc/{node, link, name_table, msg, group}.c
  - include/uapi/linux/tipc.h
-->

## Summary

TIPC (Transparent Inter-Process Communication) is a per-cluster IPC protocol designed for telecom-grade clustering: per-message named services (`tipc_service` + `tipc_socket_addr`) instead of per-IP addresses. Per-socket types: SOCK_RDM (reliable datagram), SOCK_SEQPACKET (sequenced datagram), SOCK_STREAM (reliable byte stream). Per-cluster nodes auto-discover via `tipc_disc`; per-link L2 ethernet bearer (or UDP). Per-publication: name binding to (name_type, name_instance) for service registration. Per-multicast group support. Per-address-format compact (4 bytes per node-id). Critical for: cluster-aware applications (Ericsson AXE, etc.); high-availability service-name failover.

This Tier-3 covers `socket.c` (~4010 lines) + `core.c` (~229 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tipc_sock` | per-socket state | `TipcSock` |
| `tipc_sk_create()` | socket(AF_TIPC, ...) | `Tipc::sk_create` |
| `tipc_release()` | close() | `Tipc::release` |
| `tipc_bind()` | per-(service, instance) bind | `Tipc::bind` |
| `tipc_connect()` | per-socket connect | `Tipc::connect` |
| `tipc_listen()` | per-stream listen | `Tipc::listen` |
| `tipc_accept()` | per-stream accept | `Tipc::accept` |
| `tipc_sendmsg()` | per-msg send | `Tipc::sendmsg` |
| `tipc_recvmsg()` | per-msg recv | `Tipc::recvmsg` |
| `tipc_setsockopt()` / `tipc_getsockopt()` | per-TIPC_* sockopt | `Tipc::setsockopt` / `getsockopt` |
| `tipc_sendmcast()` | per-multicast send | `Tipc::sendmcast` |
| `tipc_init()` | per-net init | `Tipc::init` |
| `tipc_node_create()` | per-cluster-node | `TipcNode::create` |
| `tipc_link_create()` | per-bearer link | `TipcLink::create` |
| `tipc_nametbl_insert_publ()` | per-service publish | `TipcNametbl::insert_publ` |
| `tipc_topsrv_*` | per-topology subscription server | `Topsrv::*` |
| `TIPC_PUBLISHED` / `TIPC_WITHDRAWN` | per-event | UAPI |
| `TIPC_SUBSCR_TIMEOUT` | per-subscription | UAPI |
| `TIPC_DESTNAME` / `TIPC_DESTID` | per-msg-cmsg | UAPI |

## Compatibility contract

REQ-1: socket(AF_TIPC, type, 0):
- type ∈ {SOCK_RDM, SOCK_SEQPACKET, SOCK_STREAM, SOCK_DGRAM}.
- Returns fd; per-socket allocated.

REQ-2: bind(sockaddr_tipc):
- struct sockaddr_tipc: { family=AF_TIPC, addrtype, scope, addr: { id | name | nameseq } }.
- TIPC_SERVICE_RANGE: bind to (type, lower, upper) range.
- TIPC_SERVICE_ADDR: bind to (type, instance).
- TIPC_SOCKET_ADDR: bind to (node, ref) (reverse-lookup).

REQ-3: connect():
- Per-stream: connect to remote (service, instance).
- Per-rdm/seqpacket: assigns peer; subsequent send default-routes there.

REQ-4: tipc_sendmsg:
- Build TIPC_MSG_BUILD chain.
- Resolve dest via name-table lookup or socket-addr.
- Per-link enqueue.

REQ-5: TIPC_MULTICAST:
- Per-(type, lower, upper) range broadcast.
- Per-cluster-node subscribers receive.

REQ-6: tipc_listen / tipc_accept:
- Per-SOCK_STREAM: connection-state machine.
- Per-INCOMING_CONN: queue accept-fd.

REQ-7: Per-name table (name_table.c):
- per-service-type: rb-tree of publications by (lower, upper) range.
- Per-publish: insert per-publication.
- Per-withdraw: remove.
- Per-lookup-by-name: returns matching socket-addr.

REQ-8: Per-publish/withdraw events:
- TIPC_PUBLISHED / TIPC_WITHDRAWN.
- Topology server (topsrv) subscribers receive.

REQ-9: Per-link (link.c):
- Per-bearer per-cluster-node 1+ links.
- Per-link sequence-numbered for reliability.
- Per-link congestion control.

REQ-10: Per-socket TIPC_GROUP:
- Per-(type, instance) group join via setsockopt TIPC_GROUP_JOIN.
- Per-msg dispatch to all group members.

REQ-11: Per-event:
- Per-publication change: subscribed listeners get TIPC_PUBLICATION event.
- Per-conn establishment: TIPC_CONN_SHUTDOWN_EVT on shutdown.

REQ-12: Per-namespace:
- Per-net-namespace TIPC instance.

REQ-13: Per-bearer:
- L2 (Ethernet): TIPC over raw eth-frames.
- UDP: TIPC over UDP/IP.

## Acceptance Criteria

- [ ] AC-1: socket(AF_TIPC, SOCK_RDM, 0): succeeds.
- [ ] AC-2: bind to TIPC_SERVICE_ADDR (type=42, instance=1): name table updated.
- [ ] AC-3: Per-cluster-node-2 sendmsg to (type=42, instance=1): delivered.
- [ ] AC-4: Per-multicast (type=42, lower=1, upper=10): per-subscriber 1..10 receives.
- [ ] AC-5: connect() SOCK_STREAM to (type=42, instance=1): connection established.
- [ ] AC-6: TIPC_GROUP_JOIN: per-msg fanned out to all members.
- [ ] AC-7: Subscribe via topsrv: per-publish/withdraw events received.
- [ ] AC-8: Bearer down: per-link reroute via alternate bearer.
- [ ] AC-9: Cluster grows: tipc_node_create new-node added; auto-discovery.
- [ ] AC-10: tipc_release: name-table publications withdrawn.

## Architecture

Per-socket state:

```
struct TipcSock {
  sk: Sock,                                       // base AF_*
  tsock_local_id: u32,
  conn_type: u32,
  conn_instance: u32,
  published: bool,
  publish_count: u32,
  publications: ListHead<TipcPublication>,        // per-bind
  group: Option<&TipcGroup>,
  rcv_unacked: u32,
  snd_win: u32,
  ...
}
```

Per-publication:

```
struct TipcPublication {
  list: ListLink,
  service: u32,                                   // type
  upper: u32,
  lower: u32,
  port: u32,                                      // socket port
  node: u32,                                      // node-id
  scope: u8,                                      // TIPC_NODE_SCOPE / CLUSTER_SCOPE / ZONE_SCOPE
}
```

`Tipc::sk_create(net, sock, proto, kern) -> Result<()>`:
1. Validate proto == 0; type ∈ {SOCK_RDM, SOCK_SEQPACKET, SOCK_STREAM, SOCK_DGRAM}.
2. Allocate TipcSock; init.
3. tsock_local_id = atomic_add_return(1, &net.tipc.last_local_id).
4. Install ops based on type.

`Tipc::bind(sock, skaddr, alen) -> Result<()>`:
1. addr = skaddr as sockaddr_tipc.
2. switch addr.addrtype:
   - TIPC_SERVICE_ADDR / TIPC_SERVICE_RANGE:
     - publish via tipc_nametbl_insert_publ.
     - tsk.published = true.
   - TIPC_SOCKET_ADDR: special (1-time bind).

`Tipc::sendmsg(sock, msg, len) -> Result<usize>`:
1. addr = parse_destination(msg).
2. switch type:
   - SOCK_STREAM: per-conn send.
   - SOCK_RDM / SOCK_DGRAM: per-msg-build + dispatch via name_lookup.
3. Per-link enqueue.

`Tipc::release(sock) -> Result<()>`:
1. tsk = (TipcSock*)sk.
2. Per-publication in tsk.publications: tipc_nametbl_withdraw_publ; per-listener TIPC_WITHDRAWN.
3. Per-conn close.
4. sock_orphan; sock_put.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `addrtype_in_set` | INVARIANT | per-bind: addrtype ∈ {TIPC_SERVICE_ADDR, TIPC_SERVICE_RANGE, TIPC_SOCKET_ADDR}. |
| `publication_count_eq_list_len` | INVARIANT | tsk.publish_count == len(tsk.publications). |
| `released_no_send` | INVARIANT | post-release: tipc_sendmsg returns -EBADF. |
| `name_tbl_unique_per_publ` | INVARIANT | per-publication unique in name-table per-(type, upper, lower, port, node). |

### Layer 2: TLA+

`net/tipc/socket.tla`:
- Per-socket lifecycle + per-name-table publish/withdraw + per-link reliable delivery.
- Properties:
  - `safety_per_publish_subscriber_notified` — per-publish: subscribers eventually receive event.
  - `safety_seq_packet_order_preserved` — per-SEQPACKET: receive order matches send order.
  - `liveness_per_msg_eventually_delivered` — per-link active + dest reachable: msg delivered.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tipc::sk_create` post: tipc_sock allocated; tsock_local_id unique | `Tipc::sk_create` |
| `Tipc::bind` post: per-(type, lower, upper) publication in name-table | `Tipc::bind` |
| `Tipc::sendmsg` post: per-msg dispatched via link or name-lookup | `Tipc::sendmsg` |
| `Tipc::release` post: publications withdrawn; conn closed | `Tipc::release` |

### Layer 4: Verus/Creusot functional

`Per-cluster TIPC service publish + name-table lookup + per-link reliable delivery` semantic equivalence: per-tipc(7) + Documentation/networking/tipc.rst.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

TIPC-specific reinforcement:

- **Per-bind CAP-check at TIPC_PUBLISH** — defense against per-name-spoof.
- **Per-name-table rb-tree balanced** — defense against per-publish O(N).
- **Per-link sequence-number reliability** — defense against per-msg-loss.
- **Per-RCU-protected name-table reads** — defense against per-walk UAF.
- **Per-multicast bounded by group size** — defense against per-fanout DoS.
- **Per-link bearer-failover** — defense against per-bearer-down service-loss.
- **Per-namespace TIPC scoped** — defense against cross-ns leakage.
- **Per-socket cleanup withdraws publications** — defense against per-stale name-table entry.
- **Per-cmsg validated** — defense against per-malformed-cmsg crash.

## Grsecurity/PaX-style Reinforcement

Baseline PaX/grsecurity mitigations applicable to TIPC (PF_TIPC):

- **PAX_USERCOPY** — TIPC sockopts and name-publish/withdraw payloads traverse whitelisted `copy_{to,from}_user`.
- **PAX_KERNEXEC** — TIPC RX (`tipc_rcv`), name-table lookup, and link state-machine execute from W^X .text.
- **PAX_RANDKSTACK** — per-syscall stack randomization frustrates name-table inference via timing.
- **PAX_REFCOUNT** — `tipc_sock`, `tipc_node`, `tipc_link` saturating refcounts on adversarial publish/withdraw churn.
- **PAX_MEMORY_SANITIZE** — name-table publication entries, cluster-auth secret buffers, and freed `tipc_node` sanitized on release.
- **PAX_UDEREF** — TIPC cmsg / TIPC_PUBLISH attribute parsing under UDEREF.
- **PAX_RAP / kCFI** — `tipc_proto`, `tipc_proto_ops` `static const`; bearer / link callbacks CFI-checked.
- **GRKERNSEC_HIDESYM** — `tipc_publ_lookup`, `tipc_node_find` hidden from unprivileged kallsyms.
- **GRKERNSEC_DMESG** — bearer-up/down / link-failover warnings CAP_SYSLOG-gated.

TIPC-specific reinforcement:

- **PF_TIPC bearer/node admin CAP_NET_ADMIN** — defense against unprivileged cluster reconfiguration (bearer enable/disable, node creation).
- **Cluster auth secret PAX_MEMORY_SANITIZE on rekey** — defense against stale cluster-auth-key residue after `TIPC_NL_NET_SET` rekey.
- **`tipc_proto` PAX_RAP-typed** — protocol-ops dispatch CFI-checked.
- **Name-table RB-tree write CAP_NET_ADMIN-gated** — defense against unprivileged name-spoof at `TIPC_PUBLISH`.
- **GRKERNSEC_HIDESYM on `tipc_node_find`** — node-lookup symbol hidden from probing.

Rationale: TIPC is cluster-wide naming + transparent failover; cluster-auth-key sanitize-on-rekey, CAP_NET_ADMIN on bearer/node admin, and CFI on the protocol-ops vtable bound the cross-cluster impersonation and stale-key residue surfaces.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/tipc/{node, link, name_table, msg, group}.c (covered separately if expanded)
- Topology server (covered separately)
- Implementation code
