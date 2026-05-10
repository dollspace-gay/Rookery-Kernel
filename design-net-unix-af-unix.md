---
title: "Tier-3: net/unix/af_unix.c — AF_UNIX (Unix-domain socket) family"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

AF_UNIX is the Linux Unix-domain socket family used for IPC. Three socket types: `SOCK_STREAM` (reliable in-order byte stream — like pipe but bidirectional), `SOCK_DGRAM` (datagram, message-boundary preserved), `SOCK_SEQPACKET` (reliable sequenced datagram). Per-socket bind to per-filesystem-path or per-abstract-name (NUL-prefixed). Per-msg send fd's via SCM_RIGHTS cmsg + credentials via SO_PEERCRED + abstract namespace. Per-stream zero-copy via `splice` + `sendfile`. Critical for: docker socket, systemd D-Bus, X11, Wayland, postgres, redis, /dev/log, every Linux IPC fabric.

This Tier-3 covers `af_unix.c` (~3981 lines).

### Acceptance Criteria

- [ ] AC-1: socket(AF_UNIX, SOCK_STREAM, 0) + bind("/tmp/sock") + listen + accept: server-side sock created.
- [ ] AC-2: connect("/tmp/sock") + sendmsg + recvmsg: bytes transferred.
- [ ] AC-3: sendmsg with SCM_RIGHTS: fd duplicated at receiver.
- [ ] AC-4: SO_PEERCRED returns peer pid/uid/gid.
- [ ] AC-5: bind to abstract @"foo": only nameservices in same netns see it.
- [ ] AC-6: SOCK_DGRAM 100-byte send: recvmsg returns 100-byte msg.
- [ ] AC-7: SOCK_DGRAM oversized + MSG_TRUNC: returns truncated + MSG_TRUNC flag.
- [ ] AC-8: Cyclic SCM_RIGHTS pass: unix_gc collects after sockets close.
- [ ] AC-9: SOCK_SEQPACKET: sequence preserved across send.
- [ ] AC-10: splice from pipe to AF_UNIX stream: zero-copy success.
- [ ] AC-11: Per-netns: nsA abstract sockets invisible from nsB.

### Architecture

Per-socket state:

```
struct UnixSock {
  sk: Sock,                                       // base AF_*
  addr: Option<&UnixAddress>,                     // bound name
  path: Path,                                     // filesystem path (if not abstract)
  iolock: Mutex<()>,                              // serializes recv
  bindlock: Mutex<()>,
  peer: Option<&Sock>,                            // connected peer (SOCK_DGRAM)
  link: HListLink,                                // unix_socket_table[hash]
  inflight: AtomicI64,                            // GC scan
  scm_stat: ScmStat,
  oob_skb: Option<&Skb>,                          // SOCK_STREAM out-of-band
  ...
}

struct UnixAddress {
  refcnt: AtomicI32,
  len: u32,
  hash: u32,
  name: SockaddrUn,                               // sun_path bytes
}
```

Per-namespace:

```
struct Net {
  ...
  unix: NetUnix,
}

struct NetUnix {
  socket_table: [HListHead<UnixSock>; UNIX_HASH_SIZE],   // 256
  table_lock: SpinLock,
  abstract_socket_count: AtomicI32,
  scm_inflight: ListHead,
  scm_inflight_lock: SpinLock,
}
```

`Unix::create(net, sock, proto, kern) -> Result<()>`:
1. Validate proto == 0.
2. Match sock.type:
   - SOCK_STREAM: ops = unix_stream_ops.
   - SOCK_DGRAM: ops = unix_dgram_ops.
   - SOCK_SEQPACKET: ops = unix_seqpacket_ops.
3. sk = Unix::create1(net, sock, kern).
4. Initialize: sk.sk_destruct = unix_sock_destructor.

`Unix::bind(sk, sockaddr_un, len) -> Result<()>`:
1. Resolve abstract vs path.
2. If abstract:
   - Compute hash; check uniqueness in netns.unix.socket_table.
3. Else (path):
   - kern_path_create(sun_path).
   - vfs_mknod with S_IFSOCK; per-inode i_mode.
   - sk.path = result.path.
4. Allocate UnixAddress; populate.
5. Insert sk into socket_table[hash] under net.unix.table_lock.

`Unix::stream_sendmsg(sk, msg, len) -> Result<usize>`:
1. peer = sk.peer.
2. Allocate skb; copy msg.iov data.
3. Process scm-cmsg (SCM_RIGHTS / SCM_CREDENTIALS).
4. skb_queue_tail(&peer.sk_receive_queue, skb).
5. peer.sk_data_ready(peer).
6. Return bytes copied.

`Unix::stream_recvmsg(sk, msg, len) -> Result<usize>`:
1. mutex_lock(&sk.iolock).
2. skb = skb_recv_datagram(sk, MSG_DONTWAIT, ...).
3. Copy skb.data to msg.iov.
4. process scm-cmsg from skb.
5. consume_skb.
6. mutex_unlock.

`Unix::scm_attach_fds(scm, skb)`:
1. For each fd in scm.fp.fp[]:
   - file = fget(fd).
   - skb.scm.fp.fp[i] = file.
   - inflight++ for AF_UNIX-typed file.

`Unix::scm_detach_fds(skb, scm)`:
1. For each file in skb.scm.fp.fp[]:
   - get_unused_fd; fd_install.
   - inflight-- for AF_UNIX-typed file.

`Unix::gc()`:
1. Walks net.unix.scm_inflight list.
2. Per-cycle detect: marks reachable from "live" sockets via SCM_RIGHTS edges.
3. Per-unreachable cycle: kfree skbs with embedded fds.

### Out of Scope

- Unix diag (covered in `unix_diag.c` separately)
- BPF inspecting AF_UNIX (covered separately)
- /proc/net/unix (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct unix_sock` | per-socket state | `UnixSock` |
| `struct unix_address` | per-bound name | `UnixAddress` |
| `unix_create()` | socket(AF_UNIX) | `Unix::create` |
| `unix_create1()` | per-socket alloc | `Unix::create1` |
| `unix_bind()` / `unix_bind_bsd()` / `unix_bind_abstract()` | per-(path or abstract) bind | `Unix::bind` |
| `unix_connect()` / `unix_dgram_connect()` | per-socket connect | `Unix::connect` |
| `unix_listen()` | per-stream listen | `Unix::listen` |
| `unix_accept()` | per-stream accept | `Unix::accept` |
| `unix_stream_sendmsg()` / `unix_dgram_sendmsg()` | per-msg send | `Unix::sendmsg` |
| `unix_stream_recvmsg()` / `unix_dgram_recvmsg()` | per-msg recv | `Unix::recvmsg` |
| `unix_release()` | per-socket close | `Unix::release` |
| `scm_send()` / `scm_recv()` | SCM_RIGHTS / SCM_CREDENTIALS | `Scm::send` / `recv` |
| `unix_attach_fds()` / `unix_detach_fds()` | per-FD-passing | `Unix::attach_fds` / `detach_fds` |
| `unix_gc()` | per-FD-garbage-collection (cycle-break) | `Unix::gc` |
| `unix_inq_len()` | per-recvmsg ioctl(SIOCINQ) | `Unix::inq_len` |
| `unix_state_lock()` | per-socket-state lock | `Unix::state_lock` |
| `UNIX_HASH_SIZE` (256) | per-socket-hash bucket | shared |

### compatibility contract

REQ-1: socket(AF_UNIX, type, 0):
- type ∈ {SOCK_STREAM, SOCK_DGRAM, SOCK_SEQPACKET}.
- Returns fd; per-socket allocated.

REQ-2: bind(sockaddr_un):
- struct sockaddr_un: { sun_family, sun_path[108] }.
- Per-path: filesystem inode (per-mount visibility).
- Per-abstract: sun_path[0] == '\0' followed by name; per-network-namespace.
- Per-path: must not exist.
- Per-abstract: must be unique within netns.

REQ-3: listen(sock, backlog):
- Per-SOCK_STREAM/SEQPACKET only.
- sk.sk_max_ack_backlog = backlog.

REQ-4: connect():
- Per-SOCK_STREAM: per-server unix_accept produces new sk; per-server sk fd attached.
- Per-SOCK_DGRAM: assigns peer; subsequent send goes to peer.

REQ-5: SCM_RIGHTS:
- struct cmsghdr { ..., type=SCM_RIGHTS, len = N+CMSG_LEN }.
- Userspace passes int fd[N]; kernel duplicates into recipient's fd-table.
- Per-pkt SCM_RIGHTS attached at send; consumed at recv.

REQ-6: SCM_CREDENTIALS:
- struct ucred { pid_t pid, uid_t uid, gid_t gid }.
- SO_PEERCRED sockopt: read peer's ucred.
- SO_PASSCRED enable: per-recv cmsg includes peer ucred.

REQ-7: Per-stream zero-copy:
- splice(in, out, ...) over AF_UNIX.
- sendfile(in_fd_file, out_fd_unix_socket, ...).
- skb fragment passing.

REQ-8: Abstract-namespace per-netns:
- sun_path[0] == 0 + name: per-netns abstract socket.
- Per-clone CLONE_NEWNET: fresh abstract namespace.

REQ-9: SOCK_DGRAM message boundary:
- Per-msg datagram preserved.
- Per-recvmsg returns one datagram (truncated if buf too small + MSG_TRUNC flag).

REQ-10: SOCK_SEQPACKET:
- Reliable + sequenced + boundary preserved.
- Combines stream-reliability with dgram-boundary.

REQ-11: Per-FD-passing GC:
- Per-cyclic SCM_RIGHTS reference cycle: e.g. socketpair where each end passes the other.
- unix_gc walks per-AF_UNIX socket-graph; collects unreferenced cycles.

REQ-12: Per-net.unix.socket-table:
- Per-netns hash table: 256 buckets indexed by full_name(sun_path).

REQ-13: Per-stream send:
- skb chained via sk.sk_receive_queue.
- Per-WAKE_OUTPUT triggers EPOLLOUT.

REQ-14: Per-pipe-buffer flow-control:
- sk.sk_max_msg = unix_max_dgram_qlen × MAX skb size.
- per-blockable: sleep until queue drains.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bind_unique_per_path` | INVARIANT | per-path: at most one bound socket. |
| `bind_unique_per_abstract_per_netns` | INVARIANT | per-(netns, abstract-name): at most one bound socket. |
| `peer_consistent` | INVARIANT | sk.peer.peer == sk (for connected pair). |
| `scm_fd_inflight_balanced` | INVARIANT | per-attached fd: inflight++; per-detached: inflight--. |
| `state_lock_held_during_mut` | INVARIANT | per-state mutation: state_lock held. |

### Layer 2: TLA+

`net/unix/af_unix.tla`:
- Per-socket lifecycle + per-msg send/recv + per-FD-passing.
- Properties:
  - `safety_no_orphan_fd` — per-attached fd eventually has matching detach OR socket destroyed.
  - `safety_dgram_boundary_preserved` — per-DGRAM send: recvmsg returns same msg.
  - `safety_stream_byte_order` — per-STREAM bytes: receiver sees same order.
  - `liveness_dgram_eventually_received` — per-send to connected peer ⟹ receiver eventually delivers.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Unix::bind` post: sk in netns.unix.socket_table[hash] | `Unix::bind` |
| `Unix::stream_sendmsg` post: skb queued at peer; peer woken | `Unix::stream_sendmsg` |
| `Unix::scm_attach_fds` post: per-fd file held; inflight++ | `Unix::scm_attach_fds` |
| `Unix::gc` post: per-unreachable-cycle skbs freed | `Unix::gc` |

### Layer 4: Verus/Creusot functional

`Per-AF_UNIX socketpair / connection ↔ exchange of (bytes, fds, creds)` semantic equivalence: per-unix(7) + scm(7) + cred(7) Linux man pages.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

AF_UNIX-specific reinforcement:

- **Per-bind path filesystem-permissions enforced** — defense against per-path bypass.
- **Per-abstract-name per-netns** — defense against cross-namespace name collision.
- **Per-SCM_RIGHTS fd refcount via fget/fput** — defense against UAF on passed fd.
- **Per-GC walks reachability** — defense against per-cycle leak.
- **Per-ucred filtered through CAP_SYS_ADMIN** — defense against per-spoofed creds.
- **Per-stream backlog cap** — defense against per-server unbounded accept queue.
- **Per-dgram qlen cap** — defense against per-peer flooding.
- **Per-iolock serializes recv** — defense against per-skb torn read.
- **Per-state_lock for connect/listen/accept** — defense against per-state torn-update.
- **Per-SCM_RIGHTS bounds checked (max 253 fds per msg)** — defense against per-cmsg-bomb.
- **Per-OOB skb single-pending** — defense against per-OOB queue overflow.

