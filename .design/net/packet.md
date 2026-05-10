# Tier-3: net/packet — AF_PACKET socket family (raw L2 access + TPACKET ringbuffer)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/packet/af_packet.c
  - net/packet/diag.c
  - net/packet/internal.h
  - include/uapi/linux/if_packet.h
-->

## Summary
Tier-3 design for the AF_PACKET socket family — raw layer-2 (Ethernet frame) access, used by tcpdump, dhclient, networkd, dhcpcd, libpcap, eBPF-XDP fallback, container CNI plugins, DPDK's pcap PMD, and any tool that needs to see/inject Ethernet frames bypassing the protocol stack.

Implements `SOCK_RAW` (full Ethernet frame including the L2 header) and `SOCK_DGRAM` (L2 header stripped, L3 payload only) modes; supports the high-throughput TPACKET ringbuffer interface (TPACKET_V1/V2/V3 frame layouts) for zero-copy packet capture; supports PACKET_FANOUT for multi-socket load balancing across CPUs.

Sub-tier-3 of `net/00-overview.md`. Pairs with `net/skbuff.md` (skb construction at RX), `net/netdev.md` (netdev bind via SO_BINDTODEVICE / sockaddr_ll), and `kernel/bpf/00-overview.md` (PACKET_MR_PROMISC + SO_ATTACH_BPF for filter).

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| AF_PACKET socket family core | `net/packet/af_packet.c` |
| sock_diag NETLINK responder | `net/packet/diag.c` |
| Internal types | `net/packet/internal.h` |
| UAPI | `include/uapi/linux/if_packet.h` |

## Compatibility contract

### `socket(AF_PACKET, type, htons(proto))`

`type` ∈ `{SOCK_RAW, SOCK_DGRAM}`.
`proto` is the EtherType (e.g., `ETH_P_ALL`, `ETH_P_IP`, `ETH_P_ARP`); `ETH_P_ALL` captures all protocols.

CAP_NET_RAW required (process or netns root).

### `struct sockaddr_ll`

```c
struct sockaddr_ll {
    unsigned short sll_family;      /* AF_PACKET */
    __be16        sll_protocol;     /* EtherType */
    int           sll_ifindex;
    unsigned short sll_hatype;       /* ARPHRD_ETHER etc. */
    unsigned char  sll_pkttype;      /* PACKET_HOST/BROADCAST/MULTICAST/OTHERHOST/OUTGOING/LOOPBACK/USER/KERNEL/FASTROUTE */
    unsigned char  sll_halen;
    unsigned char  sll_addr[8];
};
```

Layout-identical.

### TPACKET ringbuffer

Userspace creates a memory-mapped ringbuffer via `setsockopt(PACKET_RX_RING, ...)` / `PACKET_TX_RING`; kernel writes/reads frames at fixed offsets. Three frame versions:

| Version | Layout | Frame status field | Notable |
|---|---|---|---|
| TPACKET_V1 | `tpacket_hdr` | `tp_status` (volatile read) | original |
| TPACKET_V2 | `tpacket2_hdr` | adds vlan_tci, hv_vlan_proto | + nanosecond timestamps |
| TPACKET_V3 | `tpacket3_hdr` + per-block `tpacket_block_desc` | block-based; per-block timer | high-throughput |

Layout-byte-identical so libpcap + Suricata + tcpdump work unchanged.

### PACKET_FANOUT

`setsockopt(PACKET_FANOUT, ...)` joins a fanout group; multiple sockets share an RX-load distributor:
- `PACKET_FANOUT_HASH` — flow-hash distribution
- `PACKET_FANOUT_LB` — round-robin
- `PACKET_FANOUT_CPU` — per-CPU
- `PACKET_FANOUT_ROLLOVER` — rollover-on-fail
- `PACKET_FANOUT_RND` — random
- `PACKET_FANOUT_QM` — queue mapping
- `PACKET_FANOUT_CBPF` / `PACKET_FANOUT_EBPF` — BPF-program-driven distribution

Identical algorithm so DPDK + Suricata multi-worker setups work unchanged.

### Filter attach: SO_ATTACH_FILTER (cBPF) and SO_ATTACH_BPF (eBPF)

Filter applied at RX path before delivery to userspace; classic BPF (sock_fprog) or eBPF (program fd). Identical UAPI.

### `/proc/net/packet`

Per-socket diag table; format-identical so `lsof -i` / `ss -p` / netstat work.

### NETLINK_SOCK_DIAG family for AF_PACKET

`PACKET_DIAG_*` request/response identical wire format.

## Requirements

- REQ-1: `socket(AF_PACKET, type, htons(proto))` with CAP_NET_RAW gate; type ∈ SOCK_RAW/SOCK_DGRAM; proto matched per upstream's hash-table dispatch.
- REQ-2: `struct sockaddr_ll` layout-identical (REQ for /proc + diag readers).
- REQ-3: SOCK_RAW: deliver complete L2 frame (with Ethernet header) to receiver; sendmsg from this type expects a full L2 frame and pushes to driver `xmit`.
- REQ-4: SOCK_DGRAM: deliver L3 payload (L2 header stripped); sendmsg constructs an L2 header from sockaddr_ll's `sll_addr` (dest MAC) + sll_protocol (EtherType).
- REQ-5: TPACKET v1/v2/v3 ringbuffer: layout-identical; mmap'd region accessible via `mmap(MAP_SHARED, ...)` on the socket fd.
- REQ-6: TPACKET_V3 block timeout (per-block timer) + retire-block semantics identical.
- REQ-7: PACKET_FANOUT: all six fanout modes implemented; group-membership semantics identical.
- REQ-8: Filter attach (`SO_ATTACH_FILTER` cBPF + `SO_ATTACH_BPF` eBPF) applied pre-delivery; identical opcode + verifier rules (cross-ref `kernel/bpf/00-overview.md`).
- REQ-9: PACKET_AUXDATA setsockopt: per-recvmsg cmsg with VLAN tag + skb meta; format identical.
- REQ-10: PACKET_VNET_HDR for VirtIO-net pass-through; struct virtio_net_hdr layout-identical.
- REQ-11: sock_diag NETLINK_SOCK_DIAG for AF_PACKET wire-format identical.
- REQ-12: `/proc/net/packet` format identical.
- REQ-13: TLA+ model `models/net/packet_ring.tla` (mandatory per `net/00-overview.md` Layer 2 — TPACKET producer/consumer ordering safety) — proves: ring producer/consumer never read uninitialized frames; tp_status transitions monotonic per frame.
- REQ-14: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `tcpdump -i lo -w /tmp/x.pcap` captures frames; pcap byte-identical to upstream's. (covers REQ-1, REQ-3)
- [ ] AC-2: `pahole struct sockaddr_ll` byte-identical. (covers REQ-2)
- [ ] AC-3: A SOCK_DGRAM AF_PACKET socket sending an IP frame: kernel constructs an L2 header from `sll_addr`+`sll_protocol`; receiver sees the constructed Ethernet header. (covers REQ-4)
- [ ] AC-4: TPACKET_V3 ringbuffer setup: `setsockopt(PACKET_RX_RING, &v3_req)`, mmap, receive 100k frames in a tight loop, no dropped frames; `tp_status` transitions are monotonic. (covers REQ-5, REQ-6, REQ-13)
- [ ] AC-5: PACKET_FANOUT_HASH with 4 sockets in a group: identical 5-tuple flows hash to the same socket; different flows distribute across all 4. (covers REQ-7)
- [ ] AC-6: SO_ATTACH_BPF: an eBPF program returning `0` for non-TCP frames filters them out at RX. (covers REQ-8)
- [ ] AC-7: PACKET_AUXDATA: a VLAN-tagged frame is delivered with VLAN tag in cmsg; tag identical to skb's vlan_tci. (covers REQ-9)
- [ ] AC-8: A QEMU virtio-net VM using PACKET_VNET_HDR sends/receives traffic identically (verify via QEMU regression test). (covers REQ-10)
- [ ] AC-9: `ss -0` (sock_diag for AF_PACKET) byte-identical to upstream's. (covers REQ-11)
- [ ] AC-10: `cat /proc/net/packet` byte-identical. (covers REQ-12)
- [ ] AC-11: TLA+ `models/net/packet_ring.tla` proves: ∀ frame `f`, `f.tp_status` transitions are USER→KERNEL→USER (RX) or USER→KERNEL→USER (TX) — never out-of-order; producer never overwrites a frame in USER state. (covers REQ-13)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-14)

## Architecture

### Rust module organization

- `kernel::net::packet::AfPacket` — socket family root
- `kernel::net::packet::sock::PacketSock`
- `kernel::net::packet::sock_raw::SockRaw`
- `kernel::net::packet::sock_dgram::SockDgram`
- `kernel::net::packet::ring::TpacketRing` — TPACKET shared abstraction
- `kernel::net::packet::ring::v1::TpacketV1`
- `kernel::net::packet::ring::v2::TpacketV2`
- `kernel::net::packet::ring::v3::TpacketV3`
- `kernel::net::packet::fanout::Fanout`
- `kernel::net::packet::fanout::HashMode`, `LbMode`, `CpuMode`, `RolloverMode`, `RndMode`, `QmMode`, `BpfMode`
- `kernel::net::packet::filter::FilterAttach`
- `kernel::net::packet::diag::PacketDiag`

### Locking and concurrency

- **`packet_sock_list_lock`** (per-netns rwlock): protects per-netns AF_PACKET list
- **Per-`packet_sock` `pg_vec_lock`** (mutex): serializes ring resize / version change
- **Per-fanout-group `lock`** (spinlock): membership mutator
- **Per-ring frame-state**: lockless via `tp_status` `READ_ONCE`/`WRITE_ONCE` (with rmb()/wmb() barriers); validated by Layer-2 TLA+

### Error handling

- `Err(EPERM)` — non-CAP_NET_RAW
- `Err(EINVAL)` — bad fanout type, bad ring params
- `Err(ENOMEM)` — ring alloc fail
- `Err(EBUSY)` — ring resize while in use
- `Err(EFAULT)` — userspace ring-control buffer fault

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| TPACKET v1/v2/v3 frame producer/consumer barriers (READ_ONCE/WRITE_ONCE pairing) | `kani::proofs::net::packet::ring_barrier_safety` |
| Per-fanout-group membership add/remove | `kani::proofs::net::packet::fanout_safety` |
| sockaddr_ll cmsg construction | `kani::proofs::net::packet::sockaddr_ll_safety` |
| Filter attach refcount + replace path | `kani::proofs::net::packet::filter_safety` |

### Layer 2: TLA+ models

- `models/net/packet_ring.tla` (mandatory per `net/00-overview.md` Layer 2) — TPACKET producer/consumer ordering. Owned here (covers all v1/v2/v3 versions through abstraction).

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| TPACKET ring `pg_vec` | each frame is at offset `i * frame_size` for `i ∈ [0, frames-1]`; `pg_vec` array length matches block count | `kani::proofs::net::packet::pg_vec_invariants` |
| Per-fanout-group sockets list | every member's `fanout_group` pointer points back to this group | `kani::proofs::net::packet::fanout_invariants` |
| Per-netns packet sock list | every listed sock's netns matches the holding list | `kani::proofs::net::packet::list_invariants` |

### Layer 4: Functional correctness (opt-in)

- **TPACKET_V3 block-timer correctness theorem** via Verus — proves: a block becomes available to userspace iff (block is full) OR (block timer expired AND block has ≥1 frame); never both flags set simultaneously without the conjunction.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-packet_sock + per-fanout-group refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`packet_sock` slab cache | § Mandatory |
| **MEMORY_SANITIZE** | freed packet_sock + ring `pg_vec` zeroed | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: per-mode fanout dispatch ops vtables `static const`
- **USERCOPY**: sendmsg/recvmsg payload movement uses `copy_to_iter`/`copy_from_iter`
- **SIZE_OVERFLOW**: ring frame-size + frame-count multiplication uses checked operators (defense vs. CVE-2017-7308-class TPACKET_V3 integer-overflow attacks)
- **KERNEXEC**: TPACKET ring is mapped W^X — userspace mapping is RW-from-userspace, kernel writes via separate VA

### Row-2 / GR-RBAC integration

- LSM hook `security_socket_create` for AF_PACKET; GR-RBAC policy can deny per-subject (default empty). Useful default policy: deny AF_PACKET outside gradm-marked sniffer roles, since CAP_NET_RAW is a frequent capability-escape target.
- LSM hook `security_socket_setsockopt` for `PACKET_FANOUT` + ring setup; GR-RBAC policy can fence ring sizes to prevent kernel-memory-exhaustion via giant ring requests.

### Userspace-visible behavior changes

None beyond upstream defaults. CAP_NET_RAW gate identical.

### Verification

(See § Verification above.)

## Open Questions

(none — AF_PACKET ABI is exhaustively specified by upstream + libpcap + tcpdump test corpus)

## Out of Scope

- AF_PACKET on non-Ethernet link types beyond what upstream supports (e.g., DECnet — already deprecated upstream)
- XDP (cross-ref `net/xdp.md` Tier-3 — separate from AF_PACKET despite being the modern fast-RX alternative)
- 32-bit-only paths
- Implementation code
