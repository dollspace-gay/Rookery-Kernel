# Tier-3: net/struct-sock — protocol-agnostic socket state (struct sock)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/core/sock.c
  - net/core/scm.c
  - include/net/sock.h
  - include/net/inet_sock.h
  - include/net/inet_connection_sock.h
-->

## Summary
Tier-3 design for `struct sock` — the protocol-agnostic kernel-side socket state. Owns sock lifecycle (`sk_alloc` → `sk_free`), per-sock receive/send queues, sockopts dispatch (SOL_SOCKET layer), credentials + namespace + cgroup tagging, the wait-queue + wake mechanism, sk_buff handling (cross-ref `net/skbuff.md`), and the inet_sock + inet_connection_sock specializations consumed by IPv4/IPv6 protocols.

Sub-tier-3 of `net/00-overview.md`. Every protocol (TCP, UDP, SCTP, RDS, UNIX, …) extends `struct sock` with its own per-sock state struct.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| sock lifecycle + lock | `net/core/sock.c` |
| SCM helpers | `net/core/scm.c` |
| sock public types | `include/net/sock.h` |
| inet_sock (per-IP-protocol extension) | `include/net/inet_sock.h` |
| inet_connection_sock (per-connection-protocol extension; TCP, DCCP) | `include/net/inet_connection_sock.h` |

## Compatibility contract

### `struct sock` layout

`include/net/sock.h` defines `struct sock` with the `__sk_common` member as the first field (so subclassing struct sock pointers can be cast). First-cache-line + commonly-accessed fields layout-equivalent:

- `__sk_common` (sk_family, sk_state, sk_reuse, sk_reuseport, sk_kern_sock, sk_userlocks, sk_protocol, sk_type, sk_bound_dev_if, sk_bind_node, sk_bind2_node, sk_node, skc_hash, skc_u16hashes, skc_dport, skc_num, skc_addrpair, skc_portpair, skc_ipv6only, skc_net_refcnt, skc_bound_dev_if, skc_state, skc_v6_daddr, skc_v6_rcv_saddr, skc_cookie, skc_listener, skc_reuseport_cb)
- `sk_lock` (composite spin-lock + lock_count)
- `sk_drops` (atomic packet-drop count)
- `sk_rcvlowat`, `sk_receive_queue`, `sk_backlog`, `sk_sndbuf`, `sk_rcvbuf`
- `sk_flags`, `sk_no_check_tx`, `sk_no_check_rx`, `sk_userlocks`
- `sk_data_ready`, `sk_state_change`, `sk_write_space`, `sk_error_report`, `sk_destruct` (callback set)
- `sk_prot` (proto vtable: per-protocol; close, connect, disconnect, accept, ioctl, init, destroy, shutdown, setsockopt, getsockopt, sendmsg, recvmsg, hash, unhash, get_port, recv_skb, send_skb, ...)
- `sk_socket` (back-pointer to userspace-facing struct socket)
- `sk_uid`, `sk_priority`, `sk_mark`, `sk_napi_id`, `sk_max_pacing_rate`, `sk_pacing_rate`
- `sk_route_caps`, `sk_gso_type`, `sk_gso_max_size`, `sk_gso_max_segs`
- `sk_filter` (BPF filter)
- `sk_rxhash`, `sk_classid`
- `sk_bpf_storage`, `sk_security`
- `sk_cgrp_data`, `sk_memcg`, `sk_peer_cred`, `sk_peer_pid`, `sk_peer_lock`
- `sk_route_caps`, `sk_pacing_status`
- `sk_send_head`, `sk_pending_setattr`, `sk_zckey`, `sk_dontcopy_*`

Layout-equivalent.

### Sock subclasses

- `struct inet_sock` — adds IPv4/IPv6 fields: inet_daddr, inet_saddr, inet_dport, inet_sport, inet_num, mc_loop, rcv_addr_any, rcv_tos, recverr, hdrincl, ...
- `struct inet_connection_sock` — adds connection-tracking fields: icsk_accept_queue, icsk_timeout, icsk_retransmits, icsk_pending, icsk_backoff, icsk_syn_retries, icsk_probes_out, icsk_ext_hdr_len, icsk_ack, icsk_mtup, icsk_user_timeout, icsk_ca_state, icsk_ca_priv, ...
- `struct tcp_sock` — extends inet_connection_sock; covered in `net/ipv4/tcp.md`

Each subclass starts with the parent struct so up/down-casting works via pointer casting. Layout-equivalent.

### sock_destruct lifecycle

When `sk->sk_refcount` drops to zero: `sk_destruct` callback runs (per-protocol); then `__sk_destruct` schedules RCU-free; finally `sk_prot_free`. Identical.

## Requirements

- REQ-1: `struct sock` first-cache-line + commonly-accessed fields layout-equivalent so existing protocol modules work unchanged.
- REQ-2: `struct sock_common` (`__sk_common`) layout-equivalent — used by hash/lookup tables across protocols.
- REQ-3: `struct inet_sock` + `struct inet_connection_sock` layouts equivalent.
- REQ-4: `struct proto` vtable layout byte-identical (per-protocol operations).
- REQ-5: Sock lifecycle: `sk_alloc` (per-protocol slab) → `sk_init` → use → `sk_release` / `sk_destruct` → RCU-free.
- REQ-6: Sock lock: `sk_lock` is a composite spin-lock + lock_count + waitqueue (the upstream "sock_lock_t"). `lock_sock` / `release_sock` semantics identical: allows nested calls; defers BH-context skb-receive to `sk->sk_backlog_rcv`.
- REQ-7: SOL_SOCKET sockopt dispatch in `sock.c::sock_setsockopt` / `sock_getsockopt` matches upstream byte-for-byte (cross-ref `net/socket-api.md` for the SO_* set).
- REQ-8: Wait-queue + sk_data_ready / sk_write_space wakes match upstream timing (visible via tracepoints).
- REQ-9: Per-sock cgroup integration: `sk_cgrp_data` carries cgroup-v2 socket data; `sock_update_classid` updates classid. cgroup v2 socket controllers (cls, prio, memory) work unchanged.
- REQ-10: SCM helpers (sock_scm_get / sock_scm_send): identical SCM_RIGHTS + SCM_CREDENTIALS marshaling (cross-ref `net/socket-api.md` for the SCM ABI).
- REQ-11: `sk_filter` (BPF socket filter) attachment via SO_ATTACH_FILTER + SO_ATTACH_BPF; per-sock filter program runs on every recv. Identical (cross-ref `kernel/00-overview.md` § bpf).
- REQ-12: Per-sock memory accounting: `sk_rmem_alloc`, `sk_wmem_alloc`, `sk_omem_alloc`, `sk_forward_alloc` track per-sock memory; `sk->sk_memcg` for cgroup memcg.
- REQ-13: `sk_clone_lock` for accept-time clone (TCP listening): clones the parent sock, retains lock, allocates new connection-state. Identical.
- REQ-14: TLA+ co-ownership: `models/net/skb_clone.tla` (cross-ref `net/skbuff.md`) covers per-sock skb-receive-queue concurrency.
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct sock`, `struct sock_common`, `struct inet_sock`, `struct inet_connection_sock` byte-identical first-cache-line layouts vs. upstream. (covers REQ-1, REQ-2, REQ-3)
- [ ] AC-2: `pahole struct proto` byte-identical. (covers REQ-4)
- [ ] AC-3: A test creating + destroying 1M sockets (each protocol) shows correct refcount transitions; sk_destruct fires on final put. (covers REQ-5)
- [ ] AC-4: lock_sock + release_sock under contention (16 CPUs) doesn't deadlock; backlog dispatch correct. (covers REQ-6)
- [ ] AC-5: Every SOL_SOCKET sockopt round-trips (cross-ref `net/socket-api.md` AC-5). (covers REQ-7)
- [ ] AC-6: An epoll-monitored TCP socket fires sk_data_ready on each received frame; visible via bpftrace. (covers REQ-8)
- [ ] AC-7: cgroup v2 socket-mem-controller test: process exceeds cgroup memory.max via socket buffers; in-cgroup pressure triggers reclaim. (covers REQ-9)
- [ ] AC-8: SCM_RIGHTS + SCM_CREDENTIALS round-trip across an AF_UNIX socketpair. (covers REQ-10)
- [ ] AC-9: SO_ATTACH_BPF test: attach a BPF socket filter; subsequent recv returns only filter-passed frames. (covers REQ-11)
- [ ] AC-10: Per-sock memory-accounting test: write large data; sk_wmem_alloc tracks expected; cgroup charging matches. (covers REQ-12)
- [ ] AC-11: A TCP listening socket's accept produces a child sock cloned via `sk_clone_lock`; verifiable via `pahole`-tracing. (covers REQ-13)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::net::sock::Sock` — `struct sock` wrapper
- `kernel::net::sock::common::SockCommon` — `__sk_common` for hash/lookup
- `kernel::net::sock::lifecycle` — alloc / init / release / RCU-free
- `kernel::net::sock::lock::SockLock` — composite spin-lock + waitqueue
- `kernel::net::sock::sockopts` — SOL_SOCKET dispatch
- `kernel::net::sock::wait` — wait-queue + sk_*_callback wakes
- `kernel::net::sock::cgroup` — cgroup integration
- `kernel::net::sock::filter` — BPF socket filter
- `kernel::net::sock::memory` — per-sock memory accounting
- `kernel::net::sock::clone::Cloner` — sk_clone_lock for accept
- `kernel::net::sock::inet::InetSock` — IPv4/IPv6 specialization
- `kernel::net::sock::inet::ConnectionSock` — connection-tracking specialization

### Locking and concurrency

- **`sk_lock`** (composite): spin-lock + lock_count + waitqueue. Acquired via `lock_sock` (sleepable from process context). BH-context recv path defers via `sk->sk_backlog`.
- **`sk_receive_queue.lock`** (spinlock): protects per-sock receive queue
- **`sk_callback_lock`** (rwlock): protects callback set
- **`sk_peer_lock`** (rwlock): protects peer credentials
- **`sk_filter` RCU**: BPF filter is RCU-managed; reads under rcu_read_lock

### Error handling

- `Err(ENOMEM)` — alloc failed
- `Err(EINVAL)` — bad sockopt / bad arg
- `Err(EAFNOSUPPORT)` — proto/family mismatch
- `Err(EBADF)` — sock fd already closed
- `Err(ENOTCONN)` — operation requires connection

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| sock alloc + sk_init | `kani::proofs::net::sock::alloc_safety` |
| lock_sock / release_sock state machine | `kani::proofs::net::sock::lock_safety` |
| Backlog deferred-dispatch | `kani::proofs::net::sock::backlog_safety` |
| sk_clone_lock for accept | `kani::proofs::net::sock::clone_safety` |
| SCM SCM_RIGHTS fd-pass | `kani::proofs::net::scm::rights_safety` |

### Layer 2: TLA+ models

(none mandatory at this sub-tier — concurrency models live with skbuff and per-protocol Tier-3s)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-sock receive queue | qlen counter matches list length | `kani::proofs::net::sock::recv_queue_invariants` |
| Per-sock memory accounting | sk_rmem_alloc + sk_wmem_alloc + sk_omem_alloc bound by sk_rcvbuf + sk_sndbuf | `kani::proofs::net::sock::mem_alloc_invariants` |
| Sock state | sk_state transitions per per-protocol state machine; never goes "backwards" | `kani::proofs::net::sock::state_invariants` |

### Layer 4: Functional correctness (opt-in)

- **lock_sock/release_sock** correctness via Verus — proves: backlog skbs dispatched exactly once; no concurrent `sk_backlog_rcv` invocations on the same sock.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | sk_refcount + sk_wmem_alloc + sk_rmem_alloc use `Refcount` | § Mandatory |
| **AUTOSLAB** | Per-protocol sock subclasses each get their own `KmemCache::<MyProtoSock>::new()`; type-tagged in /proc/slabinfo | § Mandatory |
| **MEMORY_SANITIZE** | Sock cache freed objects zeroed (especially relevant for TLS keys, sk_filter prog refs, peer creds) | § Default-on configurable off |

### Row-1 features consumed by this component

- **UDEREF**: SOL_SOCKET sockopt values from userspace via `UserPtr<...>`
- **SIZE_OVERFLOW**: per-sock memory accounting uses checked operators
- **CONSTIFY**: per-protocol `proto` vtables provided by protocol modules are `static const`

### Row-2 / GR-RBAC integration

LSM hooks (cross-ref `net/socket-api.md` for the dispatch list):
- `security_sk_alloc` / `security_sk_free` / `security_sk_clone_security`
- `security_sk_classify_flow` (used by netfilter + tcp connection tracking)
- `security_socket_*` operations propagate via sock_create / sock_destroy

GR-RBAC's policy can deny connection establishment, classify flows, etc. via these hooks.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded copy on every SOL_SOCKET sockopt user-buffer + every SCM helper marshaling; per-protocol slabs annotated with usercopy regions.
- **PAX_KERNEXEC** — W^X on per-protocol `struct proto` vtables (TCP/UDP/SCTP/UNIX) and on JIT'd `sk_filter` BPF programs.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization at every `lock_sock`/`release_sock` entry from process context.
- **PAX_REFCOUNT** — saturating refcount on `sk_refcount`, `sk_wmem_alloc`, `sk_rmem_alloc`, `sk_omem_alloc`; per-cgroup memcg charging uses checked arithmetic.
- **PAX_MEMORY_SANITIZE** — zero-on-free for sock cache (especially `sk_security`, `sk_filter` prog refs, `sk_peer_cred`, `sk_peer_pid`, kTLS keys in `sk_ulp_data`).
- **PAX_UDEREF** — SOL_SOCKET sockopt values from userspace via typed `UserPtr<...>`; no naked `copy_from_user` on optval.
- **GRKERNSEC_HIDESYM** — hide `struct sock` pointers in `/proc/net/sockstat`, `ss` netlink output, drop-reason traces, and `inet_diag` dumps.
- **GRKERNSEC_NO_SIMULT_CONNECT** — `sk_clone_lock` (accept path) audited per-uid to throttle parallel-accept-amplification.
- **GRKERNSEC_BLACKHOLE** — `sk_drops` increments without audit when peer is probing; configurable rate-limit before audit fires.
- **GRKERNSEC_RANDNET** — `sk->sk_hash`, secret cookies for TCP-MD5/AO, and per-sock 4-tuple hash seed all from gr-random pool.
- **GRKERNSEC_NETFILTER** — netfilter integration restricts `sk->sk_mark`/`sk->sk_priority` mutation to CAP_NET_ADMIN-in-init-userns.
- **GRKERNSEC_SOCK_PRIV** — `sk_alloc` audit for privileged protocols (RAW, PACKET, NETLINK NFNL).
- **PAX_SIZE_OVERFLOW** — per-sock memory-accounting arithmetic (`sk_rmem_alloc + skb->truesize`) uses checked operators; overflow halts allocation.
- **CAP_NET_RAW / CAP_NET_ADMIN** strict — enforced in `current->nsproxy->net_ns->user_ns`.

Per-doc rationale: `struct sock` is the protocol-agnostic kernel-side socket state shared by every L4 protocol — TCP, UDP, SCTP, UNIX, NETLINK all inherit. PaX/Grsecurity reinforcement here propagates into every protocol-specific subclass automatically. Critical because (a) `sk_filter` carries attacker-controllable BPF programs that read sk fields, (b) `sk_security` is the LSM anchor for the entire flow, (c) `sk_clone_lock` is invoked from softirq under heavy SYN load, and (d) per-sock cgroup memcg charging is a known overflow CVE-class.

## Open Questions

(none — sock semantics are exhaustively specified by upstream conventions)

## Out of Scope

- Per-protocol sock subclasses (cross-ref individual protocol Tier-3 docs)
- 32-bit-only paths
- Implementation code
