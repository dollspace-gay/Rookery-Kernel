# Tier-3: net/packet/af_packet.c — AF_PACKET socket family (raw L2 socket / PACKET_MMAP / TPACKET_v3)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/packet/00-overview.md
upstream-paths:
  - net/packet/af_packet.c (~4819 lines)
  - net/packet/internal.h
  - include/uapi/linux/if_packet.h (TPACKET API)
-->

## Summary

`AF_PACKET` is Linux's raw L2 socket family for packet capture (tcpdump / libpcap), packet injection, and zero-copy bulk-receive (DPDK afp / XDP socket precursor). Per-socket binds to `(ifindex, protocol)`; per-skb received via per-protocol ptype-list dispatch in net/core/dev.c. Per-PACKET_MMAP allocates ring of TPACKET_v1/v2/v3 frames in shared mmap'd region for zero-copy receive. Per-TPACKET_v3 supports per-block delivery (jumbo-aggregation). Per-PACKET_FANOUT fans-in across multiple sockets via hash/CPU/RR. Per-PACKET_TX_RING zero-copy transmit. Critical for: tcpdump / wireshark / libpcap / userspace-bypass packet processing.

This Tier-3 covers `af_packet.c` (~4819 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct packet_sock` | per-socket state | `PacketSock` |
| `struct packet_mclist` | per-mcast subscription | `PacketMclist` |
| `struct packet_ring_buffer` | per-direction ring (RX/TX) | `PacketRingBuffer` |
| `struct packet_fanout` | per-fanout group | `PacketFanout` |
| `packet_create()` | socket(AF_PACKET) | `Packet::create` |
| `packet_bind()` / `packet_bind_spkt()` | per-(ifindex, proto) bind | `Packet::bind` |
| `packet_sendmsg()` / `packet_sendmsg_spkt()` | per-skb send | `Packet::sendmsg` |
| `packet_recvmsg()` | per-skb recv | `Packet::recvmsg` |
| `packet_rcv()` / `packet_rcv_fanout()` | per-skb input from net/core/dev | `Packet::rcv` |
| `packet_setsockopt()` | per-PACKET_* sockopt | `Packet::setsockopt` |
| `packet_set_ring()` | per-PACKET_RX_RING / TX_RING setup | `Packet::set_ring` |
| `tpacket_rcv()` | per-skb TPACKET_v1/v2 fill | `Packet::tpacket_rcv` |
| `tpacket_v3_dispatch()` | per-skb TPACKET_v3 fill | `Packet::tpacket_v3_dispatch` |
| `packet_mmap()` | per-VMA mmap of ring | `Packet::mmap` |
| `PACKET_FANOUT` | per-sockopt fanout | UAPI |
| `TPACKET_V1/V2/V3` | per-version frame format | UAPI |
| `packet_diag` | per-INET_DIAG dump (ABI) | UAPI |

## Compatibility contract

REQ-1: socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL)):
- Per-socket bound to all protocols by default.
- Per-process must have CAP_NET_RAW.

REQ-2: bind(sockaddr_ll):
- struct sockaddr_ll: { sll_family, sll_protocol, sll_ifindex, sll_hatype, sll_pkttype, sll_halen, sll_addr[8] }.
- Per-socket: sets sk_ifindex + sk_protocol filter.
- Per-skb received iff (skb.dev.ifindex == sk_ifindex ∨ sk_ifindex == 0) ∧ (skb.protocol == sk_protocol ∨ sk_protocol == ETH_P_ALL).

REQ-3: Per-protocol dispatch:
- Per-net.dev.packet_type list: per-protocol entries (ETH_P_IP / ETH_P_ARP / ETH_P_ALL).
- net/core/dev_queue_xmit + netif_receive_skb traverses ptype-list.
- Per-PacketSock has ptype struct registered.

REQ-4: setsockopt PACKET_RX_RING:
- struct tpacket_req: { tp_block_size, tp_block_nr, tp_frame_size, tp_frame_nr }.
- Allocates per-block array of pages (kmalloc-or-vmalloc).
- Per-block contains tp_block_size / tp_frame_size frames.
- Per-frame: TPACKET_HDR (struct tpacket_hdr or tpacket2_hdr or tpacket3_hdr).

REQ-5: TPACKET_V1:
- struct tpacket_hdr: per-frame metadata + offsets.
- Per-frame: hdr + skb-data + padding.

REQ-6: TPACKET_V2:
- struct tpacket2_hdr: + nanosecond timestamp + vlan-tag + tp_padding.

REQ-7: TPACKET_V3:
- struct tpacket3_hdr: per-frame within block.
- Per-block: tpacket_block_desc header + per-frame array.
- Per-block-timeout: tp_retire_blk_tov in ms.
- Per-private-data: tp_sizeof_priv per frame.

REQ-8: mmap(socket-fd, length, PROT_READ|PROT_WRITE, MAP_SHARED, 0):
- Userspace mmaps the ring at offset 0.
- Length = block_size × block_nr (RX) [+ tx-ring].
- Pages remain pinned via per-socket reference.

REQ-9: setsockopt PACKET_FANOUT:
- struct fanout: { id, type_flags }.
- type: PACKET_FANOUT_HASH / CPU / ROLLOVER / RND / QM / CBPF / EBPF.
- Multiple AF_PACKET sockets share a fanout-id; per-skb dispatched to one based on type-hash.

REQ-10: setsockopt PACKET_TX_RING:
- Symmetric for transmit.
- Userspace fills frames + wakes via send.

REQ-11: Per-socket filter:
- BPF/eBPF SO_ATTACH_FILTER works.
- Per-skb filter applied before delivery.

REQ-12: Per-VLAN handling:
- TPACKET2/3 expose vlan_tci + vlan_tpid in hdr.
- Per-PACKET_AUXDATA sockopt: cmsg with vlan-tag.

REQ-13: Per-RX-block close/release:
- TPACKET3 block-status: TP_STATUS_USER (kernel done) → userspace processes; userspace marks TP_STATUS_KERNEL.
- Per-block-retire-timeout fires if block partially-filled.

REQ-14: Per-CAP_NET_RAW gate:
- All raw L2 socket creation requires CAP_NET_RAW.

## Acceptance Criteria

- [ ] AC-1: socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL)): succeeds with CAP_NET_RAW.
- [ ] AC-2: bind to lo+ETH_P_IP: per-skb on lo with IP delivered.
- [ ] AC-3: setsockopt PACKET_RX_RING + mmap: ring mmap'd; user reads frames.
- [ ] AC-4: TPACKET_V3 + block-retire-timeout: per-partial-block retired at deadline.
- [ ] AC-5: PACKET_FANOUT + 4 sockets: per-skb hashed across sockets.
- [ ] AC-6: PACKET_TX_RING + sendto: userspace TX-ring transmits.
- [ ] AC-7: SO_ATTACH_FILTER: BPF filter applied; non-matching dropped.
- [ ] AC-8: Non-CAP_NET_RAW: socket() returns -EPERM.
- [ ] AC-9: PACKET_AUXDATA: cmsg includes vlan-tag.
- [ ] AC-10: tcpdump captures via AF_PACKET ring-mmap.

## Architecture

Per-socket state:

```
struct PacketSock {
  sk: Sock,                                       // base AF_*
  prot_hook: PacketType,                          // ptype-list registration
  tp_version: u8,                                  // TPACKET_V1/V2/V3
  rx_ring: PacketRingBuffer,
  tx_ring: PacketRingBuffer,
  fanout: Option<&PacketFanout>,
  filter: Option<&BpfProg>,
  ifindex: i32,
  hatype: u16,
  pkttype: u16,
  origdev: bool,
  auxdata: bool,
  has_vnet_hdr: bool,
  copy_thresh: u32,
  tp_loss: u32,
  num: __be16,
  mclist: ListHead<PacketMclist>,
}

struct PacketRingBuffer {
  buffer: Vec<&PageRange>,
  prb_bdqc: Option<TpacketKbdqCore>,              // TPACKET_v3
  pg_vec: Vec<*mut u8>,
  head: u32,
  cur_frame: u32,
  pending_refcnt: AtomicU32,
  ...
}

struct PacketFanout {
  id: u16,
  type_: u16,
  flags: u16,
  num_members: u16,
  arr: Vec<&Sock>,
  rover: AtomicU32,
  prog: Option<&BpfProg>,
  defrag: bool,
}
```

`Packet::create(net, sock, proto, kern) -> Result<()>`:
1. Verify CAP_NET_RAW.
2. Allocate PacketSock.
3. po.num = proto; po.tp_version = TPACKET_V1; po.copy_thresh = 0.
4. Init prot_hook = { type: proto, dev: NULL, func: packet_rcv, af_packet_priv: po }.
5. dev_add_pack(&po.prot_hook).

`Packet::bind(sk, ifindex, proto)`:
1. dev_remove_pack(&po.prot_hook).
2. po.ifindex = ifindex; po.num = proto.
3. dev_add_pack(&po.prot_hook).

`Packet::rcv(skb, dev, pt, orig_dev) -> i32`:
1. po = container_of(pt, PacketSock, prot_hook).
2. If po.fanout: return packet_rcv_fanout(skb, dev, pt, orig_dev).
3. If po.filter: bpf_run(po.filter, skb); if skipped: drop.
4. If po.rx_ring.pg_vec: tpacket_rcv(skb, ...) (zero-copy).
5. Else: skb_clone; sk_receive_skb_list_reason.

`Packet::set_ring(sk, req, tx_ring) -> Result<()>`:
1. Validate req sizes (block-size aligned to PAGE_SIZE).
2. Allocate per-block pages (alloc_pages_node order = log2(block_size/PAGE_SIZE)).
3. ring.pg_vec[i] = page_addr.
4. ring.frames_per_block = block_size / frame_size.

`Packet::mmap(file, vma) -> Result<()>`:
1. Map ring.pg_vec pages into vma.
2. expected size = block_size × block_nr (×2 for tx).

`Packet::tpacket_rcv(skb, sk, status, snaplen)`:
1. h.raw = ring.pg_vec[ring.head] + ring.head_offset.
2. Build h.tpacket_hdr (or v2/v3): tp_status, tp_len, tp_snaplen, tp_mac, tp_net, tp_sec, tp_nsec.
3. memcpy(&h.raw + tp_mac, skb.data, snaplen).
4. ring.head++.
5. h.tp_status = TP_STATUS_USER; smp_mb.
6. Wake up sk.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_net_raw_for_create` | INVARIANT | per-socket-create requires CAP_NET_RAW. |
| `ring_size_aligned` | INVARIANT | ring.block_size aligned to PAGE_SIZE. |
| `frame_offset_in_ring` | INVARIANT | per-frame: offset + frame_size ≤ ring.total_size. |
| `head_lt_frame_count` | INVARIANT | per-tpacket_rcv: ring.head < ring.frames_per_block × block_nr. |
| `fanout_consistent` | INVARIANT | per-fanout: arr[].po.fanout == self. |

### Layer 2: TLA+

`net/packet/af_packet.tla`:
- Per-skb arrival + per-ring fill + per-userspace consume + per-fanout dispatch.
- Properties:
  - `safety_no_userspace_overrun` — per-frame TP_STATUS_USER set only after kernel done.
  - `safety_fanout_dispatch_unique` — per-skb to fanout reaches at most one member.
  - `liveness_block_eventually_retired` — per-tpacket_v3 partial-block retired at timeout.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Packet::create` post: po.prot_hook on ptype-list; CAP-checked | `Packet::create` |
| `Packet::bind` post: po.ifindex/num updated; ptype-list reflects | `Packet::bind` |
| `Packet::rcv` post: skb cloned + queued OR zero-copy filled in ring | `Packet::rcv` |
| `Packet::tpacket_rcv` post: per-frame TP_STATUS_USER set after copy | `Packet::tpacket_rcv` |

### Layer 4: Verus/Creusot functional

`Per-skb arrives → ptype-list dispatch → per-AF_PACKET-socket: filter → ring/queue → userspace mmap-read` semantic equivalence: per-AF_PACKET matches Linux ABI for tcpdump / libpcap.

## Hardening

(Inherits row-1 features from `net/00-overview.md` § Hardening.)

AF_PACKET-specific reinforcement:

- **Per-CAP_NET_RAW required** — defense against unprivileged raw-L2 access.
- **Per-ring-size validated** — defense against per-config OOM.
- **Per-frame TP_STATUS race-free** — defense against per-frame torn read.
- **Per-fanout-arr bounded** — defense against per-fanout-add unbounded.
- **Per-mcast subscription scoped to bound dev** — defense against cross-dev mcast leak.
- **Per-tpacket_v3 block-retire-timeout** — defense against per-partial-block stall.
- **Per-eBPF/BPF filter run pre-deliver** — defense against per-malicious-skb traversal.
- **Per-TX_RING TP_STATUS_AVAILABLE → SEND_REQUEST → SENT/WRONG_FORMAT state machine** — defense against per-TX-userspace race.
- **Per-VMA mmap pinned via socket ref** — defense against per-mmap UAF on socket close.
- **Per-RX zero-copy bounded by snaplen** — defense against per-frame OOB write.
- **Per-fanout member add validates BPF prog** — defense against per-bpf-injection.

## Grsecurity/PaX-style Reinforcement

Baseline kernel-wide hardening leveraged by AF_PACKET:

- **PAX_USERCOPY** — `tpacket_hdr`/`tpacket2_hdr`/`tpacket3_hdr` ring frames live in a dedicated slab/`vmalloc` region with PAX_USERCOPY-whitelisted copy windows; oversized snaplen cannot bleed adjacent kernel pages into mmap.
- **PAX_KERNEXEC** — `packet_ops`, `packet_ops_spkt`, and `packet_mmap_ops` function tables live in `__ro_after_init`; a corrupted `packet_sock` cannot redirect `recvmsg`/`sendmsg` through attacker-controlled code.
- **PAX_RANDKSTACK** — `packet_rcv`, `tpacket_rcv`, and `tpacket_snd` stack frames are randomized on each entry, defeating layout-based ROP through the deep RX path.
- **PAX_REFCOUNT** — `po->mapped`, `po->tp_status`, `pkt_sk(sk)->run_filter`, and fanout-member refs use saturating `refcount_t`; rapid bind/unbind cannot wrap.
- **PAX_MEMORY_SANITIZE** — ring buffer pages zeroed on `packet_set_ring(0)` teardown and on socket close, erasing prior-capture residue across mmap reuse.
- **PAX_UDEREF** — `tpacket_get_status` and friends never deref userspace; mmap pages are kernel-resident, status flags read via `READ_ONCE` on the kernel mapping.
- **PAX_RAP/kCFI** — fanout dispatch (`fanout_demux_hash`, `_lb`, `_cpu`, `_rollover`, `_bpf`) forward-edge-checked; `dispatch_hook[]` is `const`.
- **GRKERNSEC_HIDESYM** — `packet_sock` and fanout-list pointers stripped from `/proc/net/packet`.
- **GRKERNSEC_DMESG** — malformed-bind / oversized-frame `printk_ratelimited` paths capped; cannot be flooded.

AF_PACKET-specific reinforcement:

- **PF_PACKET socket-create requires CAP_NET_RAW in the user-ns** — enforced strictly; no fallback path to unprivileged raw capture.
- **TX_RING frame copy uses PAX_USERCOPY-whitelisted `tp_net`/`tp_mac` offsets** — defense against TX-side OOB read into adjacent slab.
- **mmap'd ring length bounded by `tp_block_nr * tp_block_size`** with PAX_USERCOPY enforcement; integer overflow rejected at `packet_set_ring`.
- **VMA pin coupled to socket ref via PAX_REFCOUNT** — mmap cannot survive `close(2)` and pivot to UAF.
- **fanout-group BPF prog goes through verifier with W^X (PAX_KERNEXEC) on the JIT page** — denies code-injection via fanout.

Rationale: PF_PACKET is the canonical raw-network sandbox-escape primitive (CVE-2017-7308, CVE-2020-14386); strict CAP_NET_RAW gating combined with USERCOPY-whitelisted ring access neutralizes the historical exploit class.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- AF_PACKET diag (covered in `packet_diag.c`; separate Tier-3)
- libpcap (out-of-tree)
- AF_XDP (covered separately; net/xdp/)
- BPF cBPF/eBPF (covered separately)
- Implementation code
