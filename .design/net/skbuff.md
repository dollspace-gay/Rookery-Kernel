# Tier-3: net/skbuff — sk_buff packet container + GRO + GSO

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - net/core/skbuff.c
  - net/core/datagram.c
  - net/core/gro.c
  - net/core/gso.c
  - net/core/gro_cells.c
  - include/linux/skbuff.h
-->

## Summary
Tier-3 design for `struct sk_buff` (skb) — the canonical packet container used by every protocol layer. Owns skb allocation + clone + free, frag-list machinery for jumbo packets, GRO (Generic Receive Offload — RX-side aggregation of small packets), GSO (Generic Segmentation Offload — TX-side splitting of jumbo packets), datagram-socket helpers, and shared-info refcounting for cloned skbs.

Sub-tier-3 of `net/00-overview.md`. The skb is the most-touched data structure in the network stack: every received packet lives in an skb; every transmitted packet starts as one.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| skb core (alloc, clone, free, copy) | `net/core/skbuff.c` |
| Datagram-socket skb helpers | `net/core/datagram.c` |
| GRO (Generic Receive Offload) | `net/core/gro.c`, `net/core/gro_cells.c` |
| GSO (Generic Segmentation Offload) | `net/core/gso.c` |
| Public API | `include/linux/skbuff.h` |

## Compatibility contract

### `struct sk_buff` layout

`include/linux/skbuff.h` defines `struct sk_buff` (~250 bytes; ~80 fields). First-cache-line + commonly-macro-accessed fields layout-equivalent to upstream so out-of-tree drivers + BPF programs accessing skb fields via inline macros work unchanged.

Key fields:
- `next`, `prev` (sk_buff_head list links)
- `tstamp` (timestamp)
- `sk` (socket back-pointer)
- `dev` (net_device)
- `cb[48]` (per-protocol-layer scratch space — protocols cast to their own `struct *_skb_cb`)
- `len`, `data_len`, `truesize`
- `mac_header`, `network_header`, `transport_header`, `inner_header_*` offsets (16-bit)
- `headers` (bitfield section; protocol/encapsulation/csum hints)
- `priority`, `mark`, `napi_id`
- `data`, `head`, `tail`, `end` (data pointers within the head buffer)
- `users` (atomic; saturating refcount)
- `cloned`, `nohdr`, `peeked`, `xmit_more`, `pfmemalloc`, `gso_*`
- `secmark` (LSM blob)
- `hash`, `vlan_*`, `inner_*`

`struct skb_shared_info` (at end of head buffer):
- `nr_frags` (page-vec count)
- `tx_flags`, `frag_list`, `gso_*`, `ip6_frag_id`, `dataref`
- `frags[MAX_SKB_FRAGS]` (each: page + page_offset + size)

Layout-equivalent.

### `sk_buff_head`

Locked queue of skbs:
- `next`, `prev` (list head)
- `qlen` (count)
- `lock` (spinlock)

Layout-equivalent.

### GRO / GSO observable behavior

GRO + GSO are TX/RX optimizations userspace observes only via:
- `ethtool -k <dev>` (lists `gro` / `gso` / `tso` / `ufo` / `lro` capability)
- Throughput characteristics (correct GRO/GSO produce upstream-equivalent throughput)
- pcap captures (post-GRO RX shows aggregated packet; pre-GSO TX captures pre-segmentation)

Identical to upstream so existing tcpdump + Wireshark output formats work.

### Userspace-visible offload knobs

`SOL_SOCKET` opts: `SO_GRO_BYTES`, `SO_BUSY_POLL` interaction with GRO, `SO_INCOMING_NAPI_ID`. Identical numeric + semantic.

ethtool: `ETHTOOL_GRXFH`, `ETHTOOL_SRXFH`, `ETHTOOL_GFLAGS`, `ETHTOOL_SFLAGS` for GRO/GSO/TSO/UFO/LRO knobs. Identical.

## Requirements

- REQ-1: `struct sk_buff` first-cache-line + commonly-accessed fields layout-equivalent to upstream so out-of-tree drivers + BPF programs accessing fields via inline macros work unchanged.
- REQ-2: `struct skb_shared_info` layout-equivalent.
- REQ-3: skb alloc family: `__alloc_skb`, `alloc_skb`, `alloc_skb_with_frags`, `build_skb`, `build_skb_around`, `napi_alloc_skb`, `dev_alloc_skb`. Each preserves upstream's GFP-flag handling + size + frag-allocator integration.
- REQ-4: skb clone family: `skb_clone` (shallow clone; shares head buffer + frags), `skb_copy` (deep copy), `pskb_copy` (head copy + frag share), `skb_copy_expand`. Reference counts (skb->users + shinfo->dataref) preserved per upstream.
- REQ-5: skb free family: `kfree_skb`, `kfree_skb_reason` (with drop-reason annotation for tracepoints), `consume_skb` (free without drop tracking), `napi_consume_skb`. Identical.
- REQ-6: Frag-list (jumbo packets > 1 page): `skb_segment` (split jumbo into MTU-sized skbs for TX), `skb_aggregate` (RX-side via GRO).
- REQ-7: GRO: `napi_gro_receive` accumulates small packets at NAPI poll time; `gro_normal_list` flushes on poll-budget expiration. Decision rules + flow-table sizing match upstream.
- REQ-8: GSO: `__skb_gso_segment` produces an MTU-sized skb chain from a jumbo skb at TX time. Per-protocol gso_segment callbacks (TCP, UDP, GRE, VxLAN, ...) in their respective Tier-3 docs.
- REQ-9: Datagram-socket helpers: `__skb_recv_datagram`, `skb_copy_datagram_iter`, `skb_copy_and_csum_datagram_iter`. Identical semantics.
- REQ-10: skb fclone (fast-clone optimization): `alloc_skb_fclone` returns an skb pre-paired with a clone-ready slot; clones consume the slot without slab roundtrip. Identical performance characteristic.
- REQ-11: zerocopy + devmem-tcp: `skb_zerocopy_iter_dgram` / `skb_zerocopy_iter_stream` for MSG_ZEROCOPY; SCM_DEVMEM_DMABUF support for hardware-tagged frames. Identical.
- REQ-12: Tracepoint: `kfree_skb` tracepoint with reason code (per `enum skb_drop_reason`); identical reason set.
- REQ-13: Layer-3 invariant harness: skb frag list invariant — `shinfo->dataref` consistent with `nr_frags` + frag-list length. (Cross-ref `lib/data-structures.md` § iov_iter cursor invariants.)
- REQ-14: TLA+ model `models/net/skb_clone.tla` (per `net/00-overview.md` Layer 2 list) proves shared-skb refcounting under concurrent clone/free.
- REQ-15: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `pahole struct sk_buff` first-cache-line layout byte-identical vs. upstream. (covers REQ-1)
- [ ] AC-2: `pahole struct skb_shared_info` byte-identical. (covers REQ-2)
- [ ] AC-3: A test allocating 1M skbs via each documented alloc fn produces upstream-equivalent slab-cache utilization. (covers REQ-3)
- [ ] AC-4: A clone test: 1000 clones of one skb; shared-info dataref reaches 1001; freeing all clones drops to 1; final free completes. (covers REQ-4)
- [ ] AC-5: A `kfree_skb_reason` test verifies the dropped reason is observable via the kfree_skb tracepoint. (covers REQ-5, REQ-12)
- [ ] AC-6: A jumbo-packet TX (64K skb) is segmented to MTU-sized chain by skb_segment; reassembly via GRO yields original byte content. (covers REQ-6, REQ-7, REQ-8)
- [ ] AC-7: An iperf3 TCP test on Rookery shows GRO + GSO optimization (visible via `ethtool -k <dev>`); throughput within ±5% of upstream. (covers REQ-7, REQ-8)
- [ ] AC-8: A UDP `recvfrom` benchmark exercises datagram-helper alloc + iov_iter copy correctly. (covers REQ-9)
- [ ] AC-9: An fclone-using TCP test allocates skbs + clones for retransmit queue without per-clone slab roundtrip. (covers REQ-10)
- [ ] AC-10: An MSG_ZEROCOPY test on TCP send produces SCM_TIMESTAMPING completion notifications identically. (covers REQ-11)
- [ ] AC-11: `bpftrace -e 'tracepoint:skb:kfree_skb { printf("reason=%d\\n", args->reason); }'` emits identical reason codes vs. upstream. (covers REQ-12)
- [ ] AC-12: `make verify` passes `kani::proofs::net::skb::frag_invariants`. (covers REQ-13)
- [ ] AC-13: `make tla` passes `models/net/skb_clone.tla`. (covers REQ-14)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-15)

## Architecture

### Rust module organization

- `kernel::net::skb::SkBuff` — `struct sk_buff` wrapper
- `kernel::net::skb::shinfo::SharedInfo` — `skb_shared_info`
- `kernel::net::skb::alloc` — alloc / build / napi_alloc family
- `kernel::net::skb::clone` — clone / copy / pskb_copy
- `kernel::net::skb::frag` — frag_list manipulation
- `kernel::net::skb::seg::Segmenter` — skb_segment GSO
- `kernel::net::skb::gro::Gro` — GRO accumulator
- `kernel::net::skb::datagram` — datagram-socket helpers
- `kernel::net::skb::queue::SkbQueue` — `sk_buff_head` wrapper
- `kernel::net::skb::cb::Cb` — typed access to `cb[48]` per-layer scratch space

### Locking and concurrency

- **Per-`sk_buff_head` `lock`** (spinlock): protects the queue
- **Per-skb `users`** (atomic): saturating refcount; clone increments, free decrements
- **`shinfo->dataref`** (atomic): refcount on the head data buffer
- **GRO per-NAPI flow table**: per-CPU; protected by NAPI-poll preempt-disable

TLA+ model `models/net/skb_clone.tla` covers the shared-info refcount under concurrent clone/free.

### Error handling

- `Err(ENOMEM)` — alloc failed
- `Err(EFAULT)` — bad userspace pointer in zerocopy path
- `Err(EMSGSIZE)` — packet too large for socket
- `Err(EINVAL)` — bad GSO/GRO config

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| skb head buffer pointer arithmetic | `kani::proofs::net::skb::head_arith_safety` |
| Frag page reference take/drop | `kani::proofs::net::skb::frag_safety` |
| skb_segment split arithmetic | `kani::proofs::net::skb::segment_safety` |
| GRO accumulator merge | `kani::proofs::net::skb::gro_merge_safety` |
| sk_buff_head queue manipulation | `kani::proofs::net::skb::queue_safety` |

### Layer 2: TLA+ models

- `models/net/skb_clone.tla` (mandatory per `net/00-overview.md` Layer 2) — proves shared-skb refcounting under concurrent clone + free across multiple CPUs. Owned here.

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| skb frag list | shared-info dataref consistent with frag_list length + per-frag refcounts | `kani::proofs::net::skb::frag_invariants` |
| sk_buff_head | qlen counter matches actual list length under lock | `kani::proofs::net::skb::queue_invariants` |
| GRO flow table | Each entry's flow_id appears at most once in the table | `kani::proofs::net::skb::gro_table_invariants` |

### Layer 4: Functional correctness (opt-in)

- **skb_segment correctness** via Verus — proves: segmenting a jumbo skb produces a chain whose reassembly yields the original byte content. High-leverage; segmentation/reassembly bugs cause silent data corruption.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | skb->users + shinfo->dataref + per-frag page refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | skb allocated via per-class slab cache (skbuff_head_cache, skbuff_fclone_cache); per-class type-tagged | § Mandatory |
| **MEMORY_SANITIZE** | freed skbs route through page-allocator/slab; zeroed if `vm.zero_on_free=1` | § Default-on configurable off |

### Row-1 features consumed by this component

- **UDEREF**: never accepts user pointers directly; iov_iter is the consumer-side abstraction (cross-ref `lib/usercopy.md`)
- **SIZE_OVERFLOW**: skb len + truesize + frag-size arithmetic uses checked operators
- **CONSTIFY**: per-protocol skb-cb access patterns are typed via per-protocol `cb` accessor structs (each `static const` shape)

### Row-2 / GR-RBAC integration

skb is below the LSM-hook layer; LSM hooks fire at higher levels (socket-layer + protocol-layer). However, `secmark` field on skb carries LSM tags propagated through the network stack — used by SELinux + AppArmor to label packets.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none — skb semantics are exhaustively specified by upstream conventions)

## Out of Scope

- Per-protocol cb usage (cross-ref individual protocol Tier-3 docs)
- Hardware offload feature negotiation (cross-ref `net/ethtool.md`)
- 32-bit-only paths
- Implementation code
