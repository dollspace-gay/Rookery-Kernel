---
title: "Tier-3: net/core/skbuff.c — sk_buff allocation, cloning, and fragmentation lifecycle"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The `sk_buff` is the kernel's network packet representation — every packet in/out flows through one. skbuff.c provides per-skb alloc/free, cloning (shared head, separate metadata), copy-on-write linearize, fragmentation (jumbo-frame split, TCP-segmentation-offload), and shared-info (skb_shared_info: nr_frags, frag-list, GSO-info). Per-skb data layout: head/tail/end pointers + per-skb sk_buff struct (~232 bytes per Linux ABI). skbuff.c is ~7476 lines — third-largest networking file (after dev.c + sock.c).

This Tier-3 covers `net/core/skbuff.c` (~7476 lines) — focused on alloc/clone/fragment/share semantics.

### Acceptance Criteria

- [ ] AC-1: __alloc_skb(1500): returns skb with head, data, tail at start; end at +1500.
- [ ] AC-2: skb_put(skb, 100): tail advances by 100; len = 100.
- [ ] AC-3: skb_clone: new skb with cloned=1; shared head; metadata independent.
- [ ] AC-4: skb_copy: new skb with cloned=0; head deep-copied.
- [ ] AC-5: skb_linearize on multi-frag skb: collapses to linear.
- [ ] AC-6: skb_segment for TSO: splits 64KB skb into 1500-byte segments.
- [ ] AC-7: napi_alloc_skb: per-CPU cache used; reduced slab-allocator pressure.
- [ ] AC-8: kfree_skb_reason: tracks per-drop reason; visible in /proc/net/skb_drops.
- [ ] AC-9: page_pool integration: drivers use; DMA mappings recycled.
- [ ] AC-10: skb-stress: 10M alloc/free cycles; no slab-leak; throughput within 10% of baseline.

### Architecture

`SkBuff`:

```
struct SkBuff {
  next: AtomicPtr<SkBuff>,
  prev: AtomicPtr<SkBuff>,
  dev: KArc<NetDevice>,
  cb: [u8; 48],
  _skb_refdst: KAtomicPtr<DstEntry>,
  sp: Option<KArc<SecPath>>,
  len: u32,
  data_len: u32,
  mac_len: u16,
  hdr_len: u16,
  queue_mapping: u16,
  truesize: u32,
  users: AtomicI32,
  head: NonNull<u8>,
  data: NonNull<u8>,
  tail: u32,                                   // offset from head
  end: u32,                                    // offset from head
  pkt_type: u8,
  ip_summed: u8,
  csum_start: u16,
  csum_offset: u16,
  cloned: bool,
  head_frag: bool,
  fclone: u8,
  nohdr: bool,
  ...
}

#[repr(C)]
struct SkbSharedInfo {
  nr_frags: u8,
  tx_flags: u8,
  gso_size: u16,
  gso_segs: u16,
  gso_type: u16,
  frag_list: AtomicPtr<SkBuff>,
  hwtstamps: SkbHwTimestamps,
  meta_len: u32,
  destructor_arg: KAtomicPtr<()>,
  dataref: AtomicI32,
  frags: [SkbFrag; MAX_SKB_FRAGS],
}

struct SkbFrag {
  bv_page: KArc<Page>,
  bv_offset: u32,
  bv_len: u32,
}
```

`SkBuff::alloc(size, gfp, flags, node)`:
1. skb := kmem_cache_alloc(skbuff_head_cache, gfp).
2. data := kmalloc_reserve(SKB_DATA_ALIGN(size + sizeof(SkbSharedInfo)), gfp).
3. skb.head = data.
4. skb.data = data.
5. skb.tail = 0; skb.end = size.
6. skb.users = 1.
7. shared_info := skb_shinfo(skb); shared_info.dataref = 1; shared_info.nr_frags = 0.
8. Return skb.

`SkBuff::clone(skb, gfp)`:
1. n := kmem_cache_alloc(skbuff_head_cache, gfp).
2. memcpy(n, skb, sizeof(*skb)) — shallow copy.
3. atomic_inc(&skb_shinfo(skb).dataref).
4. n.cloned = true; skb.cloned = true.
5. Return n.

`SkBuff::copy(skb, gfp)`:
1. n := __alloc_skb(skb.len, gfp, 0, NUMA_NO_NODE).
2. skb_put(n, skb.len).
3. skb_copy_bits(skb, 0, n.data, skb.len) — deep copy linear + frags.
4. Per-frag: refcount per-page (skb-shared dataref unchanged).
5. Return n.

`SkBuff::segment(skb, features)`:
1. mss := skb_shinfo(skb).gso_size.
2. nr_segs := skb_shinfo(skb).gso_segs.
3. For i in 0..nr_segs:
   - seg := alloc_skb(mss + headroom).
   - Copy header + per-segment data.
4. Chain segments.
5. Return first.

### Out of Scope

- skbuff Tier-2 overview (already exists in `net/skbuff.md`)
- net_device (covered in `net/core/dev.md` Tier-3)
- struct sock (covered in `net/struct-sock.md` Tier-2)
- page_pool (covered in `net/core/page_pool.md` Tier-3)
- napi (covered in `net/core/dev.md` Tier-3)
- dst_entry (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sk_buff` | per-packet metadata | `net::core::skbuff::SkBuff` |
| `struct skb_shared_info` | per-skb shared frag/GSO info | `SkbSharedInfo` |
| `__alloc_skb(size, gfp, flags, node)` | per-skb alloc | `SkBuff::alloc` |
| `__build_skb(data, frag_size)` | per-skb wrap pre-allocated buffer | `SkBuff::build` |
| `napi_alloc_skb(napi, length)` | per-NAPI fast-path alloc | `SkBuff::napi_alloc` |
| `netdev_alloc_skb(dev, length)` | per-netdev alloc | `SkBuff::netdev_alloc` |
| `kfree_skb(skb)` / `kfree_skb_reason(skb, reason)` | per-skb free | `SkBuff::kfree` / `_kfree_reason` |
| `consume_skb(skb)` | per-skb release (success path) | `SkBuff::consume` |
| `skb_clone(skb, gfp)` | per-skb cheap clone (shared head) | `SkBuff::clone` |
| `skb_copy(skb, gfp)` | per-skb deep copy | `SkBuff::copy` |
| `skb_copy_expand(skb, headroom, tailroom, gfp)` | per-skb copy with extra space | `SkBuff::copy_expand` |
| `skb_share_check(skb, gfp)` / `_unshare(skb, gfp)` | per-skb COW if shared | `SkBuff::share_check` / `_unshare` |
| `skb_linearize(skb)` | per-skb collapse non-linear into linear | `SkBuff::linearize` |
| `skb_pull(skb, len)` / `_push(skb, len)` | per-skb data-area-grow | `SkBuff::pull` / `_push` |
| `skb_put(skb, len)` / `_trim(skb, len)` | per-skb tail-extend / shrink | `SkBuff::put` / `_trim` |
| `skb_reserve(skb, len)` | per-skb reserve headroom | `SkBuff::reserve` |
| `skb_pad(skb, pad)` | per-skb pad-to-min | `SkBuff::pad` |
| `__skb_frag_set_page(...)` | per-frag set | `SkbFrag::set_page` |
| `skb_add_rx_frag(skb, idx, page, off, size, truesize)` | per-skb add frag-page | `SkBuff::add_rx_frag` |
| `skb_segment(skb, features)` | per-skb GSO segmentation | `SkBuff::segment` |
| `skb_tx_error(skb)` | per-skb mark TX-error | `SkBuff::tx_error` |
| `skb_morph(dst, src)` | per-skb meta-copy without data | `SkBuff::morph` |

### compatibility contract

REQ-1: Per-skb `sk_buff` (~232 bytes):
- `next` / `prev` (list head).
- `dev` (per-NIC ref).
- `cb[48]` (control buffer; per-protocol scratch).
- `_skb_refdst` (dst-cache encoded; lower 1 bit = noref-flag).
- `sp` (xfrm SecPath).
- `len` (current data length).
- `data_len` (non-linear length; in frags).
- `mac_len` / `hdr_len`.
- `truesize` (allocation footprint; head + frags + skb-itself).
- `users` (refcount; for shared-skb).
- `head` / `data` / `tail` / `end` (pointers into linear buffer).
- `peeked` (peek-buffer handling).
- `head_frag` (per-skb head data fragment).
- `cloned` (bool: is this a clone).
- `fclone` (fast-clone state).
- `nohdr` (no per-skb data on free).
- `pkt_type` / `ip_summed` / `csum_*` (checksum offload state).
- `priority` / `mark` / `queue_mapping`.
- `tstamp` (timestamp).
- `napi_id` (per-NAPI).
- `sender_cpu`.

REQ-2: Per-skb `skb_shared_info`:
- `nr_frags` (count of paged fragments).
- `frags[MAX_SKB_FRAGS]` (per-frag (page, offset, size, truesize)).
- `frag_list` (chained skb-frags).
- `tx_flags` (per-skb tx-timestamp flags).
- `gso_size` / `gso_segs` / `gso_type` (per-GSO config).
- `hwtstamps`.
- `dataref` (per-data-buffer refcount).
- `destructor_arg`.

REQ-3: `__alloc_skb` flow:
1. Validate size + gfp.
2. Allocate skb-struct via `kmem_cache_alloc(skbuff_head_cache)`.
3. data := kmalloc_reserve(SKB_DATA_ALIGN(size + sizeof(struct skb_shared_info)), gfp).
4. Init pointers: head = tail = data; end = data + size.
5. Init refcount: users = 1, dataref = 1.
6. Per-skb fields zero'd.

REQ-4: `__build_skb`:
- Wrap pre-allocated kmalloc-buffer (e.g., from page_pool) in skb head.
- No additional alloc; cheaper than __alloc_skb.

REQ-5: `napi_alloc_skb` fast-path:
- Per-CPU per-NAPI cache of pre-allocated skb-fragments.
- Avoids slab-allocator overhead per-RX skb.

REQ-6: `skb_clone`:
- Allocate new skb-struct (shallow); inherit head data via shared dataref++.
- cloned = 1; new skb has separate metadata.
- Used when packet flows through multiple paths (e.g., bridging).

REQ-7: `skb_copy`:
- Allocate new skb + new head buffer; deep-copy data.
- Used when caller needs independent modification of data.

REQ-8: `skb_share_check`:
- If skb.users > 1: deep-copy via skb_copy.
- Else: passthrough.

REQ-9: `skb_linearize`:
- Per-frag pages collapsed into single linear data area.
- Used when caller needs contiguous data (e.g., crypto operations).

REQ-10: GSO (Generic Segmentation Offload):
- Per-skb skb_shared_info.gso_size + .gso_type encodes segmentation.
- skb_segment splits per-MTU-sized segments before transmit.
- Eliminates per-segment skb-build cost.

REQ-11: Per-skb COW semantics:
- skb.cloned = 1 means head shared with another skb.
- Modifications to head require unshare → deep-copy.

REQ-12: page_pool integration:
- Per-NAPI-driver may use page_pool for per-frag allocation.
- DMA-mapped pre-allocated pages; recycled per-RX cycle.
- Frees skbs without unmapping DMA.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `skb_users_no_underflow` | INVARIANT | per-skb users ≥ 0; defense against double-free. |
| `dataref_no_underflow` | INVARIANT | per-shared_info dataref ≥ 0; defense against double-release. |
| `tail_le_end` | INVARIANT | per-skb tail ≤ end; defense against skb_put past end. |
| `data_le_tail` | INVARIANT | per-skb data ≤ tail; defense against skb_pull past data. |
| `nr_frags_bounded` | INVARIANT | per-shared_info nr_frags ≤ MAX_SKB_FRAGS. |

### Layer 2: TLA+

`net/core/skbuff_lifecycle.tla`:
- Per-skb state ∈ {Alloc, Cloned, Owned, Freed}.
- Per-shared_info state: dataref count.
- Properties:
  - `safety_no_use_after_free` — Freed skb not accessed.
  - `safety_dataref_balance` — dataref decreased on each free; head freed only at last decref.
  - `liveness_pending_eventually_freed` — unowned skb eventually freed.

`net/core/skb_segment.tla`:
- GSO segmentation produces N skbs each with per-segment header.
- Properties:
  - `safety_segments_cover_data` — Σ(segment.len) == orig_skb.len + N*hdr_len.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SkBuff::alloc` post: skb allocated; users == 1; dataref == 1 | `SkBuff::alloc` |
| `SkBuff::clone` post: per-shared dataref incremented; new skb shares head | `SkBuff::clone` |
| `SkBuff::copy` post: new skb has independent head; per-frag refcounts incremented | `SkBuff::copy` |
| `SkBuff::segment` post: chain of segments; total data == orig - hdr | `SkBuff::segment` |

### Layer 4: Verus/Creusot functional

`Per-skb operations preserve packet-data integrity` semantic equivalence: per-skb after clone+modify, original skb data unchanged; copy-expand preserves data semantics.

### hardening

(Inherits row-1 features from `net/skbuff.md` Tier-2 § Hardening.)

skbuff-alloc-specific reinforcement:

- **Per-skb users refcount** — defense against UAF on shared skb.
- **Per-shared_info dataref refcount** — defense against premature head-free.
- **kmalloc_reserve fallback to vmalloc** — defense against large-skb slab-failure.
- **napi_alloc_skb per-CPU cache** — defense against slab-allocator hot-path contention.
- **GSO size validated against MTU** — defense against pathological segment-count.
- **MAX_SKB_FRAGS cap** — defense against unbounded frag-count.
- **Per-segment header copy correct** — defense against corrupted segment-header.
- **page_pool DMA mapping cached** — defense against per-skb DMA-map overhead.
- **kfree_skb_reason tracking** — defense against silent drop hiding bug.
- **Per-skb truesize tracking** — defense against under-accounting in mem-cgroup.
- **Per-skb cloned flag honored** — defense against modifying shared head causing data corruption.
- **fclone fast-clone slot** — defense against double-allocation.

