# Tier-3: net/xdp/xsk.c — AF_XDP socket (XDP user-space zero-copy)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/00-overview.md
upstream-paths:
  - net/xdp/xsk.c (~2085 lines)
  - net/xdp/xdp_umem.c
  - net/xdp/xsk_buff_pool.c
  - net/xdp/xsk_queue.c
  - net/xdp/xskmap.c
  - include/net/xdp.h
  - include/uapi/linux/if_xdp.h
-->

## Summary

AF_XDP provides per-userspace zero-copy raw L2 socket via XDP. Per-socket binds to (iface, queue-index) + UMEM (Userspace Memory; mmap'd ring buffer of frames). Per-skb at XDP_REDIRECT to AF_XDP socket: kernel passes frame to per-socket's RX ring; userspace reads ring entries (no copy if zero-copy). Per-driver supports zero-copy XDP: per-NIC rx descriptor points directly to UMEM frame. Per-non-zero-copy fallback: skb_copy via SKB-mode XDP. Per-XSKMAP BPF-program-target map. Critical for: DPDK-class userspace fast-path on stock Linux NICs, packet capture at line-rate, eBPF + AF_XDP networking apps (Cilium, Katran).

This Tier-3 covers `xsk.c` (~2085 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct xdp_sock` | per-socket state | `XdpSock` |
| `struct xdp_umem` | per-UMEM userspace mem-region | `XdpUmem` |
| `struct xsk_buff_pool` | per-(UMEM, queue) frame pool | `XskBuffPool` |
| `struct xsk_queue` | per-(producer / consumer) ring | `XskQueue` |
| `struct xsk_map` | BPF XSKMAP for redirect | `XskMap` |
| `xsk_create()` | socket(AF_XDP) | `Xsk::create` |
| `xsk_bind()` | bind to (iface, queue, UMEM) | `Xsk::bind` |
| `xsk_setsockopt()` | per-XDP_* | `Xsk::setsockopt` |
| `xsk_mmap()` | per-UMEM/ring mmap | `Xsk::mmap` |
| `xsk_sendmsg()` | per-TX kick | `Xsk::sendmsg` |
| `xsk_recvmsg()` | per-RX poll | `Xsk::recvmsg` |
| `__xsk_map_redirect()` | BPF-redirect target | `Xsk::map_redirect` |
| `xdp_do_redirect()` | per-XDP-prog redirect | `Xsk::do_redirect` |
| `xsk_umem_create()` | UMEM register | `XdpUmem::create` |
| `xsk_umem_consume()` / `_produce()` | per-ring move | `XskQueue::consume` / `produce` |
| `xsk_buff_alloc()` | per-pool alloc | `XskBuffPool::alloc` |
| `XDP_RX_RING` / `XDP_TX_RING` / `XDP_UMEM_FILL_RING` / `XDP_UMEM_COMPLETION_RING` | per-sockopt | UAPI |
| `XDP_MMAP_OFFSETS` | per-mmap offset | UAPI |

## Compatibility contract

REQ-1: socket(AF_XDP, SOCK_RAW, 0):
- Per-socket allocates xdp_sock; no UMEM yet.

REQ-2: setsockopt XDP_UMEM_REG:
- Userspace registers struct xdp_umem_reg { addr, len, chunk_size, headroom, flags }.
- UMEM = userspace shared-mem region.
- Per-(chunk_size) per-frame container.

REQ-3: setsockopt XDP_UMEM_FILL_RING / XDP_UMEM_COMPLETION_RING:
- Userspace allocates ring of per-frame-descriptor (addr in UMEM).
- FILL: producer ring; userspace produces empty frames for RX.
- COMPLETION: consumer ring; userspace gets back used frames after TX.

REQ-4: setsockopt XDP_RX_RING / XDP_TX_RING:
- RX: producer ring (kernel fills); userspace consumes.
- TX: producer ring (userspace fills); kernel consumes.

REQ-5: bind(sockaddr_xdp):
- struct sockaddr_xdp: { family=AF_XDP, sxdp_flags, sxdp_ifindex, sxdp_queue_id, sxdp_shared_umem_fd }.
- Per-flags: XDP_SHARED_UMEM / XDP_COPY / XDP_ZEROCOPY / XDP_USE_NEED_WAKEUP.

REQ-6: mmap (offset XDP_PGOFF_RX_RING / TX_RING / UMEM_PGOFF_FILL_RING / COMPLETION_RING):
- Userspace mmaps per-ring + UMEM into its addr-space.

REQ-7: Per-RX path (XDP_REDIRECT to XSKMAP):
- BPF XDP program returns XDP_REDIRECT.
- Driver invokes __xsk_map_redirect (per-XSKMAP indexed by (key, ifindex)).
- Frame produced into RX-ring with descriptor (addr-offset, len).
- xs.sk.sk_data_ready notified.

REQ-8: Per-TX path:
- Userspace produces TX-ring descriptor.
- sendmsg(socket, NULL, MSG_DONTWAIT) wakes up kernel.
- Kernel consumes TX descriptors; sends via driver.
- On completion: produces to COMPLETION-ring.

REQ-9: Zero-copy mode (XDP_ZEROCOPY):
- Per-driver supports zero-copy: NIC RX-descriptor points to UMEM frame.
- Per-driver supports zero-copy TX: NIC TX from UMEM frame.
- Per-flags XDP_USE_NEED_WAKEUP: kernel signals userspace when to call sendmsg.

REQ-10: Per-shared UMEM:
- Multiple sockets share single UMEM via XDP_SHARED_UMEM.
- Per-queue-id one xdp_sock owns FILL/COMPLETION rings.

REQ-11: Per-bpf XSKMAP:
- BPF program: bpf_map_update_elem(XSKMAP, key, &fd).
- bpf_redirect_map(XSKMAP, key, 0).

REQ-12: Per-non-XDP-prog mode:
- AF_XDP requires per-iface attached XDP-prog with bpf_redirect_map.
- libxdp / xdpdump provide simple programs.

## Acceptance Criteria

- [ ] AC-1: socket(AF_XDP) + XDP_UMEM_REG + bind: succeeds.
- [ ] AC-2: mmap UMEM + RX_RING + FILL_RING: userspace accessible.
- [ ] AC-3: Per-XDP-prog returns XDP_REDIRECT to XSKMAP: skb in RX_RING.
- [ ] AC-4: TX_RING populated + sendmsg: kernel TXs frames.
- [ ] AC-5: COMPLETION_RING receives sent descriptors.
- [ ] AC-6: XDP_ZEROCOPY + supported driver: no copy on RX/TX.
- [ ] AC-7: XDP_COPY fallback on non-ZC driver: skb_copy_to_umem.
- [ ] AC-8: Shared UMEM: 2 sockets share UMEM; each on diff queue.
- [ ] AC-9: XDP_USE_NEED_WAKEUP: per-need-wakeup userspace knows to syscall.
- [ ] AC-10: xdp-pcap or similar tool: captures via AF_XDP.

## Architecture

Per-xdp_sock:

```
struct XdpSock {
  sk: Sock,                                       // base AF_*
  state: XdpSockState,
  pool: *XskBuffPool,
  umem: *XdpUmem,
  rx: *XskQueue,                                  // kernel→user
  tx: *XskQueue,                                  // user→kernel
  fq_tmp: *XskQueue,                              // shared-UMEM tmp
  cq_tmp: *XskQueue,
  dev: *NetDev,
  queue_id: u32,
  flags: u16,                                    // XDP_*
  zc: bool,
  xs_node: ListLink,                              // xsk_map.list
}

struct XdpUmem {
  region: UmemRegion,
  fq: *XskQueue,                                  // fill-ring
  cq: *XskQueue,                                  // completion-ring
  npgs: u32,
  chunk_size: u32,
  headroom: u32,
  pgs: Vec<&Page>,
  refcount: AtomicI64,
  zc: bool,
}

struct XskBuffPool {
  fq: *XskQueue,
  cq: *XskQueue,
  umem: *XdpUmem,
  heads: Vec<XdpBuffXsk>,
  free_heads: Vec<&XdpBuffXsk>,
  free_heads_cnt: u32,
  /* per-chunk metadata */
}

struct XskQueue {
  cached_prod: u32,
  cached_cons: u32,
  mask: u32,
  size: u32,
  producer: *AtomicU32,
  consumer: *AtomicU32,
  ring: *void,                                    // mmap'd shared
}
```

`Xsk::create(net, sock, proto, kern) -> Result<()>`:
1. xs = allocate XdpSock.
2. xs.state = INACTIVE.
3. sock.ops = &xsk_proto_ops.

`Xsk::bind(sock, addr, len) -> Result<()>`:
1. sxdp = (sockaddr_xdp*)addr.
2. dev = dev_get_by_index(net, sxdp.sxdp_ifindex).
3. xs.dev = dev; xs.queue_id = sxdp.sxdp_queue_id.
4. if sxdp.sxdp_flags & XDP_SHARED_UMEM:
   - share-UMEM from xs_attached via sxdp.sxdp_shared_umem_fd.
5. /* Try ZC mode */
6. if sxdp.sxdp_flags & XDP_ZEROCOPY:
   - err = dev.netdev_ops.ndo_bpf(dev, &xdp_zc_disable).
   - if err: return -ENOTSUPP.
   - xs.zc = true.
7. xs.pool = xsk_buff_pool_create(xs.umem, xs.dev, xs.queue_id).
8. xs.state = BOUND.

`Xsk::sendmsg(sock, msg, len) -> Result<usize>`:
1. /* Wake kernel to process TX_RING */
2. if !xs.zc: xsk_generic_xmit(xs).
3. else: driver-wake; ndo_xsk_wakeup.

`Xsk::recvmsg(sock, msg, len, flags, addr_len) -> Result<usize>`:
1. /* Userspace expected to read RX_RING directly via mmap */
2. /* recvmsg may signal availability */
3. Return -EOPNOTSUPP (mmap-based, not recvmsg).

`Xsk::map_redirect(map, key, xdp)`:
1. xs = lookup XSKMAP[key].
2. if xs.dev != xdp.rxq.dev: return -EINVAL.
3. /* Enqueue xdp_buff descriptor into xs.rx ring */
4. desc = { addr: xsk_buff_xdp_get_addr(xdp), len: xdp_get_buff_len(xdp) }.
5. err = xskq_prod_reserve_desc(xs.rx, addr, len).
6. wake xs.sk.

`XskQueue::produce(q, desc) -> Result<()>`:
1. if !room: return -ENOBUFS.
2. q.ring[q.cached_prod & q.mask] = desc.
3. smp_wmb().
4. q.producer.store(q.cached_prod + 1).

`XskQueue::consume(q) -> Option<desc>`:
1. if q.cached_cons == q.producer.load_acquire(): return None.
2. desc = q.ring[q.cached_cons & q.mask].
3. q.cached_cons++.
4. q.consumer.store(q.cached_cons).
5. return Some(desc).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `umem_chunk_size_pow2` | INVARIANT | umem.chunk_size is power-of-2. |
| `queue_size_pow2` | INVARIANT | queue.size power-of-2. |
| `cached_cons_le_producer` | INVARIANT | q.cached_cons ≤ q.producer. |
| `xs_socket_state_machine` | INVARIANT | xs.state ∈ {INACTIVE, READY, BOUND}. |
| `zc_only_if_driver_supports` | INVARIANT | xs.zc ⟹ driver supports zc. |

### Layer 2: TLA+

`net/xdp/xsk.tla`:
- Per-socket bind + per-RX redirect + per-TX kick + per-UMEM share.
- Properties:
  - `safety_no_data_outside_umem` — per-RX/TX descriptor addr within UMEM.
  - `safety_no_double_use_chunk` — per-frame-chunk in FILL ⟹ NOT in RX.
  - `liveness_per_TX_eventually_completes` — per-TX descriptor: COMPLETION-ring eventually has it.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Xsk::bind` post: xs.pool created; xs.state == BOUND | `Xsk::bind` |
| `Xsk::map_redirect` post: descriptor in xs.rx ring | `Xsk::map_redirect` |
| `XskQueue::produce` post: ring[cached_prod & mask] = desc; producer++ | `XskQueue::produce` |
| `XskQueue::consume` post: returned descriptor; cached_cons++ | `XskQueue::consume` |

### Layer 4: Verus/Creusot functional

`Per-XDP_REDIRECT to AF_XDP socket → per-userspace-mmap'd ring receives frame at line-rate; per-zero-copy frees skb-alloc cost` semantic equivalence: per-Documentation/networking/af_xdp.rst.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

AF_XDP reinforcement:

- **Per-UMEM addr validation** — defense against per-userspace OOB UMEM-addr DMA.
- **Per-chunk_size power-of-2** — defense against per-mask-alignment UB.
- **Per-queue producer/consumer atomic** — defense against per-ring torn.
- **Per-zc only if driver advertises NETIF_F_XDP_TX_ZC etc.** — defense against per-driver-unsupported zc.
- **Per-XSKMAP lookup RCU-protected** — defense against per-map mutate during redirect.
- **Per-CAP_NET_RAW required** — defense against per-unprivileged AF_XDP-create.
- **Per-shared UMEM ref-balanced** — defense against per-shared UMEM UAF.
- **Per-bind validates queue-id < dev->real_num_rx_queues** — defense against per-OOR queue.
- **Per-mmap UMEM offsets sanity-check** — defense against per-mmap OOB.
- **Per-XDP_USE_NEED_WAKEUP opt-in** — defense against per-userspace polling-spin.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/xdp/xdp_umem.c / xsk_buff_pool.c / xsk_queue.c / xskmap.c (covered separately if expanded)
- BPF XDP framework (covered in `bpf/00-overview.md` separately)
- libxdp / libbpf userspace (out-of-tree)
- Implementation code
