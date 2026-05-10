---
title: "Tier-3: net/core/sock.c — struct sock allocation + per-protocol ops + sk_buff queue management + memory accounting"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`net/core/sock.c` is the per-socket lifecycle backbone — every kernel-managed socket (TCP, UDP, RAW, UNIX, NETLINK, etc.) ultimately backs to a `struct sock`. Per-sock has: send/receive queues, per-sock memory accounting (sk_wmem_alloc / sk_rmem_alloc), wakeup-info, per-protocol ops (proto_ops + sk_prot), TCP/UDP-specific state in inet_sock + tcp_sock subclass. ~4570 lines providing alloc/release/sendmsg/recvmsg dispatch. Critical for: every userspace networked process, BSD-socket API, kernel networking from drivers/iSCSI/NFS.

This Tier-3 covers `net/core/sock.c` (~4570 lines).

### Acceptance Criteria

- [ ] AC-1: socket(AF_INET, SOCK_STREAM, 0): allocates struct sock + struct socket; refcount = 1.
- [ ] AC-2: connect(): triggers sk_prot->connect; socket transitions to TCP_ESTABLISHED.
- [ ] AC-3: send(): sock_sendmsg → sk_prot->sendmsg; data queued + buffered.
- [ ] AC-4: recv(): sock_recvmsg → sk_prot->recvmsg; data delivered to userspace.
- [ ] AC-5: setsockopt(SO_SNDBUF, X): sk->sk_sndbuf = X*2 (kernel doubles).
- [ ] AC-6: Per-sock memory accounting: 100MB write-buffer; sk_wmem_alloc tracks.
- [ ] AC-7: SO_BPF_FILTER: install BPF filter; per-skb evaluated before delivery.
- [ ] AC-8: close(): sock_release → sk_destruct → sk_free; memory uncharged.
- [ ] AC-9: Memory pressure: sock_under_memory_pressure → throttle.
- [ ] AC-10: Per-protocol slab: AF_INET TCP socks share slab; per-cpu freelists used.

### Architecture

`Sock`:

```
struct Sock {
  __sk_common: SockCommon,                     // family, state, etc.
  sk_lock: SockLock,
  sk_send_head: AtomicPtr<SkBuff>,
  sk_receive_queue: SkbQueue,
  sk_write_queue: SkbQueue,
  sk_wmem_queued: AtomicI32,
  sk_wmem_alloc: AtomicI32,
  sk_rmem_alloc: AtomicI32,
  sk_sndbuf: i32,
  sk_rcvbuf: i32,
  sk_pacing_rate: u32,
  sk_protocol: u8,
  sk_state: u8,
  sk_type: u16,
  sk_prot: KArc<Proto>,
  sk_socket: KWeak<Socket>,
  sk_callback_lock: RwLock<()>,
  sk_data_ready: SockCallback,
  sk_write_space: SockCallback,
  sk_error_report: SockCallback,
  sk_state_change: SockCallback,
  sk_filter: AtomicPtr<SockFilter>,
  sk_forward_alloc: i32,
  sk_drops: AtomicI32,
  ...
}

struct Proto {
  name: KStr,
  owner: KModule,
  obj_size: u32,
  slab: KArc<KmemCache>,
  init: ProtoInit,
  destroy: ProtoDestroy,
  close: ProtoClose,
  accept: ProtoAccept,
  connect: ProtoConnect,
  ioctl: ProtoIoctl,
  setsockopt: ProtoSetSockOpt,
  getsockopt: ProtoGetSockOpt,
  sendmsg: ProtoSendMsg,
  recvmsg: ProtoRecvMsg,
  bind: ProtoBind,
  backlog_rcv: ProtoBacklogRcv,
  release_cb: ProtoReleaseCb,
  hash: ProtoHash,
  unhash: ProtoUnhash,
  ...
  memory_allocated: AtomicLong,
  sockets_allocated: PerCpu<i32>,
  memory_pressure: AtomicBool,
  sysctl_mem: [i32; 3],
  sysctl_wmem: [i32; 3],
  sysctl_rmem: [i32; 3],
}
```

`Sock::create(net, family, type, proto, &sock, kern)`:
1. Look up family-handler (per-AF registered).
2. handler->create(net, sock, proto, kern).
3. *sock_out = sock.

`Sock::sk_alloc(net, family, gfp, prot, kern)`:
1. sk := kmem_cache_alloc(prot->slab, gfp).
2. memset(sk, 0, prot->obj_size).
3. sock_init_data(sock, sk):
   - Default callbacks.
   - sk->sk_net = net.
   - sk->sk_protocol = -1 (TBD).
4. module_get(prot->owner).
5. Return sk.

`Sock::sendmsg(sock, msg)`:
1. security_socket_sendmsg(sock, msg, msg->msg_iter.count).
2. ret := sock->sk->sk_prot->sendmsg(sock->sk, msg, msg->msg_iter.count).
3. Return ret.

`Sock::wmem_schedule(sk, size)`:
1. If size <= sk->sk_forward_alloc: charge and return.
2. amt := DIV_ROUND_UP(size - sk->sk_forward_alloc, SK_MEM_QUANTUM).
3. allocated := atomic_long_add_return(amt, &prot->memory_allocated).
4. If allocated > sysctl_mem[2]: revert + return false (memory-pressure-deny).
5. sk->sk_forward_alloc += amt * SK_MEM_QUANTUM.

### Out of Scope

- struct sock Tier-2 (`net/struct-sock.md`)
- Per-protocol details (TCP/UDP — covered separately)
- struct socket Tier-2 (covered)
- BPF filter (covered in `kernel/bpf/bpf-core.md` Tier-3)
- BSD socket API surface (covered in `net/socket-api.md` Tier-2)
- net_namespace (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sock` | per-socket common base | `net::core::sock::Sock` |
| `struct proto` | per-protocol vtable (TCP/UDP/etc.) | `Proto` |
| `struct proto_ops` | per-AF socket-API vtable | `ProtoOps` |
| `sock_alloc(net, family, gfp, prot, kern)` | per-sock alloc | `Sock::alloc` |
| `sock_release(sock)` | per-sock release | `Sock::release` |
| `sk_alloc(net, family, gfp, prot, kern)` | low-level sk alloc | `Sock::sk_alloc` |
| `sk_free(sk)` | low-level sk free | `Sock::sk_free` |
| `sock_init_data(sock, sk)` | per-sock init defaults | `Sock::init_data` |
| `sk_send_sigurg(sk)` | per-sock SIGURG | `Sock::send_sigurg` |
| `__sock_create(net, family, type, proto, &sock, kern)` | per-AF/type create | `Sock::create` |
| `sock_create_lite(...)` | minimal create (no inode) | `Sock::create_lite` |
| `sock_recvmsg(sock, msg, flags)` | per-sock recv-dispatcher | `Sock::recvmsg` |
| `sock_sendmsg(sock, msg)` | per-sock send-dispatcher | `Sock::sendmsg` |
| `__sock_sendmsg(sock, msg)` | low-level send (no security check) | `Sock::__sendmsg` |
| `sk_dst_check(sk, cookie)` | per-sock dst-cache validate | `Sock::dst_check` |
| `__sk_dst_set(sk, dst)` | per-sock dst-set | `Sock::__dst_set` |
| `sk_setup_caps(sk, dst)` | per-sock dst-derived caps | `Sock::setup_caps` |
| `sock_wmalloc(sk, size, force, gfp)` | per-sock wmem-tracked alloc | `Sock::wmalloc` |
| `sock_kmalloc(sk, size, gfp)` | per-sock km alloc + accounting | `Sock::kmalloc` |
| `sock_kfree_s(sk, mem, size)` | per-sock km free + accounting | `Sock::kfree_s` |
| `sk_wmem_schedule(sk, size)` | per-sock send-buffer reservation | `Sock::wmem_schedule` |
| `sk_rmem_schedule(sk, skb, size)` | per-sock recv-buffer reservation | `Sock::rmem_schedule` |
| `sk_mem_charge(sk, size)` | charge to memcg | `Sock::mem_charge` |
| `sk_mem_uncharge(sk, size)` | uncharge | `Sock::mem_uncharge` |
| `sock_no_*` (placeholder ops) | fallback for unsupported ops | `Sock::no_*` |
| `sock_setsockopt(sock, level, optname, optval, optlen)` | SO_* options | `Sock::setsockopt` |
| `sock_getsockopt(...)` | SO_* read | `Sock::getsockopt` |
| `sk_filter(sk, skb)` | BPF socket-filter dispatch | `Sock::filter` |

### compatibility contract

REQ-1: Per-sock `sock` (~512 bytes, varies by config):
- `__sk_common` (struct sock_common): family, state, bound_dev_if, etc.
- `sk_send_head` / `sk_receive_queue` (per-sock skb queues).
- `sk_wmem_queued` (write-memory-queued).
- `sk_wmem_alloc` (atomic: write-memory-allocated).
- `sk_rmem_alloc` (atomic: read-memory-allocated).
- `sk_sndbuf` / `sk_rcvbuf` (limits).
- `sk_lock` (per-sock spinlock + bh-disable; struct sock_lock).
- `sk_pacing_rate`.
- `sk_protocol` (IPPROTO_*).
- `sk_state` (TCP_LISTEN / _CLOSE / etc.).
- `sk_prot` (KArc<Proto>).
- `sk_socket` (back-ref to struct socket).
- `sk_callback_lock` (rwlock for per-sock callbacks).
- `sk_data_ready` / `sk_write_space` / `sk_error_report` / `sk_state_change` (callbacks).
- `sk_filter` (BPF filter).

REQ-2: `proto` per-protocol vtable:
- `name` (e.g., "TCP").
- `obj_size` (per-sock memory footprint).
- `slab` (per-protocol slab cache).
- `init` / `destroy` / `close` / `accept` / `connect` / `disconnect` / `shutdown`.
- `setsockopt` / `getsockopt`.
- `sendmsg` / `recvmsg`.
- `bind` / `bind_add` (for SO_BINDTODEVICE etc.).
- `backlog_rcv` (per-sock backlog dispatch).
- `release_cb` (per-sock release).
- `hash` / `unhash` (per-protocol hash table mgmt).
- `get_port` (per-protocol bind-port resolution).
- `forward_alloc` (per-sock memory forward-allocation).
- `memory_allocated` (per-protocol global mem counter).
- `sysctl_*` (per-protocol sysctl).

REQ-3: `proto_ops` per-AF (socket-API):
- `family` (AF_INET / AF_UNIX / etc.).
- `release` / `bind` / `connect` / `socketpair` / `accept` / `getname` / `poll`.
- `ioctl` / `gettstamp`.
- `listen` / `shutdown` / `setsockopt` / `getsockopt`.
- `sendmsg` / `recvmsg`.
- `mmap` / `sendpage`.
- `read_skb` (per-AF skb-pull-helper).

REQ-4: Per-sock alloc flow:
1. `sk_alloc(net, family, gfp, prot, kern)`:
   - sk := kmem_cache_alloc(prot.slab, gfp).
   - sock_init_data(sock, sk).
   - sk->sk_net = net.
   - sk->sk_prot = prot.
   - module_get(prot.owner).

REQ-5: Per-sock free flow:
1. `sk_free(sk)`:
   - If sk->sk_destruct: sk_destruct(sk).
   - sk_filter_uncharge.
   - sock_diag_broadcast_destroy.
   - kmem_cache_free(prot.slab, sk).
   - module_put(prot.owner).

REQ-6: Per-sock send memory accounting:
1. `sock_wmalloc(sk, size, force, gfp)`:
   - If !force && atomic_read(&sk->sk_wmem_alloc) + size > sk->sk_sndbuf: -ENOBUFS.
   - skb := alloc_skb(size, gfp).
   - skb_set_owner_w(skb, sk): atomic_add(skb->truesize, &sk->sk_wmem_alloc).
   - Return skb.

REQ-7: Per-sock recv memory accounting:
1. `sock_recvmsg(sock, msg, flags)`:
   - sk := sock->sk.
   - msg := use lock + sk->sk_prot->recvmsg(sk, msg, flags).

REQ-8: Per-sock memory charge / forward-allocation:
- `sk_mem_charge(sk, size)`:
  - sk->sk_forward_alloc -= size.
  - If sk->sk_forward_alloc <= 0: sk_mem_schedule (refill from per-protocol pool).
- `sk_mem_uncharge(sk, size)`:
  - sk->sk_forward_alloc += size.
  - If sk->sk_forward_alloc > SK_MEM_QUANTUM: sk_mem_reclaim.

REQ-9: Per-protocol global memory:
- `prot.memory_allocated` (long; in pages).
- Compared against prot.sysctl_mem[3] thresholds (low / pressure / high).
- Triggers sock_pressure / sock_no_pressure transitions.

REQ-10: Per-sock callbacks:
- `sk->sk_data_ready(sk)`: called when data arrives (wakes recv waiters).
- `sk->sk_write_space(sk)`: called when send buffer has space.
- `sk->sk_error_report(sk)`: per-error-event.
- `sk->sk_state_change(sk)`: per-state transition.
- Default: sock_def_*.

REQ-11: Per-sock filter (BPF):
- `sk_filter(sk, skb)`: invoke per-sock BPF filter.
- Defense against unauthorized data-receive.

REQ-12: Per-sock release_cb:
- Called from per-protocol release-context.
- Used by TCP for ACK delay etc.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sk_refcount_no_underflow` | INVARIANT | per-sock refcount ≥ 0; defense against double-release. |
| `sk_state_valid` | INVARIANT | per-sock state ∈ valid TCP_*-state range. |
| `wmem_alloc_no_underflow` | INVARIANT | sk_wmem_alloc ≥ 0; defense against double-uncharge. |
| `forward_alloc_balance` | INVARIANT | sk->sk_forward_alloc ≥ 0 within per-quantum tolerance. |
| `prot_slab_per_protocol` | INVARIANT | per-proto slab distinct; defense against cross-proto allocation. |

### Layer 2: TLA+

`net/core/sock_lifecycle.tla`:
- Per-sock state ∈ {Free, Allocated, Bound, Connected, Listening, Closing, Released}.
- Properties:
  - `safety_no_use_after_release` — Released sock not accessed.
  - `safety_state_transitions_per_TCP_RFC` — only valid TCP transitions.
  - `liveness_pending_eventually_released` — every Allocated eventually Released on close.

`net/core/sock_memory.tla`:
- Per-sock memory accounting state.
- Properties:
  - `safety_alloc_balanced` — every charge paired with eventual uncharge.
  - `safety_memory_pressure_throttle` — when under-pressure, allocations rejected.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sock::sk_alloc` post: sk allocated; init_data called; module_get | `Sock::sk_alloc` |
| `Sock::sendmsg` post: per-protocol sendmsg called; security check passed | `Sock::sendmsg` |
| `Sock::wmem_schedule` post: per-sock forward_alloc updated; per-prot memory_allocated tracks | `Sock::wmem_schedule` |
| Per-sock callbacks always non-NULL (default vs custom) | invariants on init_data |

### Layer 4: Verus/Creusot functional

`Per-sock send/recv: data flows through per-protocol ops + memory accounting + filter` semantic equivalence: per-message data preserved + memory accounted + filter applied.

### hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

sock-core-specific reinforcement:

- **Per-sock refcount tracked** — defense against UAF on close-during-active-IO.
- **Per-protocol obj_size validated** — defense against per-proto sk-struct corruption from mis-cast.
- **Per-protocol slab + per-cpu freelist** — defense against allocator hot-path contention.
- **Per-sock memory accounting** — defense against memory exhaustion via unbounded per-sock allocation.
- **Per-protocol memory pressure thresholds** — defense against sock-flood OOM.
- **Per-sock filter ref-counted** — defense against UAF on filter-update-during-evaluation.
- **Per-sock callback rwlock** — defense against callback-replace-during-call.
- **Per-cgroup memcg accounting** — defense against per-cgroup unbounded socket-memory.
- **sk_state validated against TCP-RFC** — defense against state-machine bypass.
- **module_get/put paired** — defense against module-unload-with-active-socks.
- **Per-AF security_socket_* hooks** — defense against unauthorized AF use.
- **Per-protocol hash table per-namespace** — defense against cross-netns leak.

