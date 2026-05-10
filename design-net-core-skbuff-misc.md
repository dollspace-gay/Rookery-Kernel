---
title: "Tier-3: net/core/skbuff.c (helpers subset) — sk_buff manipulation helpers (clone/copy/expand/trim/pull/push/split/segment)"
tags: ["tier-3", "net", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Per-skb manipulation helpers cover the wide path between alloc and free: per-skb clone (shallow copy w/ shared frags), per-skb copy (deep), per-skb expand-head (grow headroom/tailroom), per-skb pull/push/trim (linear-region pointer moves), per-skb split (split linear-data in two), per-skb segment (GSO split into per-MSS segments), per-skb seq-read (iterate fragmented data), per-skb copy-bits (gather scattered fragments). Critical for: TCP segmentation, packet tunneling/encap, network filters, packet capture (PACKET socket), GSO offload.

This Tier-3 covers `skbuff.c` helpers subset (~3000 lines of helpers within ~7476 line file).

### Acceptance Criteria

- [ ] AC-1: skb_clone: returns skb with shared shinfo; original.dataref == 2.
- [ ] AC-2: skb_copy: returns skb with private shinfo; per-frag bytes equal.
- [ ] AC-3: pskb_expand_head(skb, 64, 0): skb.headroom += 64; data ptr preserved.
- [ ] AC-4: skb_push(skb, 14): can write 14 bytes of L2 header before data.
- [ ] AC-5: skb_pull(skb, 20): consume 20 bytes of L3 header.
- [ ] AC-6: skb_trim(skb, len): skb.len == len.
- [ ] AC-7: skb_split(skb, skb1, 1500): skb has 1500 bytes; skb1 has remainder.
- [ ] AC-8: skb_segment(skb, features) on 10000-byte TCP-skb: 7 × 1500-byte segments.
- [ ] AC-9: skb_copy_bits(skb, 100, buf, 50): copies 50 bytes at offset 100.
- [ ] AC-10: skb_unshare on shared skb: returns deep-copy.
- [ ] AC-11: pskb_pull_tail(skb, 200): pulls 200 bytes from frag into linear.
- [ ] AC-12: kfree_skb on cloned skb: original still readable; refcnt decremented.

### Architecture

(Builds atop `Skb` struct from `skbuff-alloc.md`.)

`Skb::clone(skb, gfp) -> Option<Skb>`:
1. n = kmem_cache_alloc(skbuff_head_cache, gfp)?.
2. __skb_clone(n, skb).
3. Return Some(n).

`Skb::__clone(n, skb)`:
1. n.* = skb.* (memcpy fields up to truesize boundary).
2. n.users = 1.
3. n.cloned = 1; skb.cloned = 1.
4. shinfo(skb).dataref += 1 (atomic).

`Skb::copy(skb, gfp) -> Option<Skb>`:
1. n = alloc_skb(skb.head_len + skb.data_len + headroom).
2. skb_copy_header(n, skb).
3. skb_copy_bits(skb, -headroom, n.head, ...).
4. Return Some(n).

`Skb::pskb_expand_head(skb, nhead, ntail, gfp) -> Result<()>`:
1. New size = ksize(skb.head + nhead + ntail).
2. data = kmalloc(size).
3. memcpy(data + nhead, skb.head, skb.tail - skb.head).
4. Per-shared: bump dataref of shinfo.
5. skb.head = data; skb.data += nhead; skb.tail += nhead; skb.end = data + size.
6. skb.cloned = 0.

`Skb::push(skb, len) -> *mut u8`:
1. skb.data -= len.
2. skb.len += len.
3. Assert: skb.data ≥ skb.head.
4. Return skb.data.

`Skb::pull(skb, len) -> *mut u8`:
1. Assert: skb.len ≥ len.
2. If len > skb.head_len: __pskb_pull_tail(skb, len - skb.head_len)?
3. skb.data += len; skb.len -= len.
4. Return skb.data.

`Skb::trim(skb, len)`:
1. if len ≥ skb.len: return.
2. skb.tail = skb.data + len.
3. skb.len = len.
4. shinfo(skb).gso_size = 0.

`Skb::copy_bits(skb, offset, to, len) -> Result<()>`:
1. Copy from linear: min(len, head_len - offset).
2. Copy from frags: walk per-frag with offset adjustment.
3. Recurse into frag_list.
4. Return Err if offset/len OOB.

`Skb::split(skb, skb1, len)`:
1. Per-linear: if len ≤ skb.head_len:
   - skb_split_inside_header(skb, skb1, ...).
   - Per-frag stays on skb1.
2. Else:
   - skb_split_no_header(skb, skb1, ..., pos = len - skb.head_len).
   - Per-frag split at pos.
3. skb.tail = skb.data + len; skb.len = len.

`Skb::segment(head_skb, features) -> Skb`:
1. mss = head_skb.gso_size.
2. Per-segment alloc skb of size mss + headers.
3. Copy per-skb header.
4. Copy per-segment data slice.
5. Link via skb.next.
6. Return linked-list head.

### Out of Scope

- skbuff alloc/free (covered in `skbuff-alloc.md` Tier-3)
- net/core/dev (covered in `dev.md` Tier-3)
- TCP/UDP segmentation specifics (covered in `tcp-output.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `skb_clone()` | shallow clone (shared frags) | `Skb::clone` |
| `__skb_clone()` | per-skb shallow clone backend | `Skb::__clone` |
| `skb_copy()` | deep copy linear+frags | `Skb::copy` |
| `__pskb_copy_fclone()` | per-skb partial copy | `Skb::pskb_copy_fclone` |
| `pskb_expand_head()` | expand headroom + tailroom | `Skb::pskb_expand_head` |
| `skb_copy_expand()` | copy + expand combined | `Skb::copy_expand` |
| `skb_push()` / `skb_pull()` | move data pointer | `Skb::push` / `pull` |
| `skb_pull_data()` | pull + return removed data | `Skb::pull_data` |
| `skb_trim()` / `___pskb_trim()` | shrink length | `Skb::trim` / `__pskb_trim` |
| `__pskb_pull_tail()` | pull non-linear into linear | `Skb::__pskb_pull_tail` |
| `skb_copy_bits()` | gather bytes from fragmented skb | `Skb::copy_bits` |
| `skb_split()` | split linear at offset | `Skb::split` |
| `skb_segment()` | GSO segment | `Skb::segment` |
| `skb_segment_list()` | per-fraglist segment | `Skb::segment_list` |
| `skb_seq_read()` | iterate per-frag bytes | `Skb::seq_read` |
| `skb_clone_sk()` | per-sk clone | `Skb::clone_sk` |
| `skb_unshare()` | force-private (write-after-clone) | `Skb::unshare` |
| `skb_share_check()` | clone-if-shared | `Skb::share_check` |
| `skb_copy_header()` | per-skb metadata copy | `Skb::copy_header` |

### compatibility contract

REQ-1: `skb_clone(skb, gfp)`:
- Allocates new sk_buff struct.
- Copies fields verbatim from skb (linear ptrs + shinfo ptr).
- Per-skb dataref++ (shared frags + shinfo).
- Returns new skb sharing data.

REQ-2: `skb_copy(skb, gfp)`:
- Deep copy; new linear data buffer.
- Per-frag copied (no sharing).
- Returns fully-private skb.

REQ-3: `pskb_expand_head(skb, nhead, ntail, gfp)`:
- Expand head room by nhead, tail room by ntail.
- Reallocate data buffer; preserve linear-data; preserve frags+shinfo.
- Per-skb cloned: forces uncoupling (unshare).
- Updates skb.head, skb.data, skb.tail, skb.end.

REQ-4: `skb_push(skb, len)`:
- skb.data -= len; skb.len += len.
- Asserts skb.data ≥ skb.head (room available).
- Returns new skb.data.

REQ-5: `skb_pull(skb, len)`:
- skb.data += len; skb.len -= len.
- Asserts skb.data + len ≤ skb.tail.
- Returns new skb.data.

REQ-6: `skb_trim(skb, len)`:
- If len ≥ skb.len: no-op.
- Else: skb.tail = skb.data + len; skb.len = len; flush frags beyond.

REQ-7: `___pskb_trim(skb, len)`:
- Per-non-linear-trim: walk frags; reduce/free as needed.
- Per-fraglist: walk skb_shinfo(skb)->frag_list.
- Returns 0 on success or -ENOMEM.

REQ-8: `__pskb_pull_tail(skb, delta)`:
- Pull `delta` bytes from non-linear into linear.
- May allocate; may copy from frags into linear.
- Returns ptr to new linear-end-of-data.

REQ-9: `skb_copy_bits(skb, offset, to, len)`:
- Copy `len` bytes starting at `offset` into `to`.
- Walks linear + frags + frag_list.
- Returns 0 on success or -EFAULT.

REQ-10: `skb_split(skb, skb1, len)`:
- Split skb at `len`: skb keeps [0..len), skb1 keeps [len..end).
- Per-linear: skb_split_inside_header / skb_split_no_header.
- Updates per-skb len + tail; per-skb1 head + tail + frags.

REQ-11: `skb_segment(head_skb, features)`:
- GSO segment: split into per-MSS skb's.
- Per-segment: copy per-skb header + per-segment data slice.
- Returns linked list of skb's.
- Per-features: NETIF_F_TSO / NETIF_F_GSO_TCPv6 / etc. select MSS rules.

REQ-12: `skb_unshare(skb, gfp)`:
- If skb_shared(skb) (refcnt > 1): skb_copy(skb).
- Else: return skb.

REQ-13: `skb_share_check(skb, gfp)`:
- If shared (refcnt > 1) ∧ writeable needed: skb_clone.
- Caller frees old via kfree_skb.

REQ-14: Per-skb refcount semantics:
- skb_get(skb): refcnt++.
- kfree_skb(skb): refcnt--; free if reaches 0.
- skb_dataref(shinfo): refcnt for shared frag-data.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `clone_dataref_bumped` | INVARIANT | post-clone shinfo(skb).dataref ≥ 2. |
| `expand_head_preserves_data` | INVARIANT | post-pskb_expand_head: skb.data bytes [0..len) unchanged. |
| `push_within_headroom` | INVARIANT | per-push: skb.data ≥ skb.head. |
| `pull_within_data` | INVARIANT | per-pull: skb.data + len ≤ skb.tail. |
| `trim_le_original_len` | INVARIANT | post-trim skb.len ≤ original-len. |
| `segment_total_len_eq_head_len` | INVARIANT | sum-of-segment-lens == head_skb.len (modulo headers). |

### Layer 2: TLA+

`net/core/skbuff_helpers.tla`:
- Per-skb operations sequenced + frag-data shared.
- Properties:
  - `safety_no_free_while_shared` — per-shared skb (refcnt > 1) ⟹ data not freed.
  - `safety_clone_isolation_post_unshare` — per-unshare-ed skb mutated independently.
  - `safety_segment_byte_total_preserved` — per-segment: total bytes == input total.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Skb::clone` post: returned-skb shares shinfo; both cloned=1 | `Skb::clone` |
| `Skb::pskb_expand_head` post: skb.data bytes preserved; head/tail expanded | `Skb::pskb_expand_head` |
| `Skb::push` post: skb.data -= len; skb.len += len | `Skb::push` |
| `Skb::trim` post: skb.len == len; tail moved | `Skb::trim` |
| `Skb::segment` post: linked-list of skbs covering input | `Skb::segment` |

### Layer 4: Verus/Creusot functional

`Per-TCP GSO skb segment → per-MSS-sized skbs that reassemble into original` semantic equivalence: per-Linux skb_segment matches GSO spec.

### hardening

(Inherits row-1 features from `net/core/skbuff-alloc.md` § Hardening.)

Skb-helpers-specific reinforcement:

- **Per-clone shinfo-dataref bumped atomically** — defense against double-free or premature-free.
- **Per-push/pull bounds-checked** — defense against OOB pointer arithmetic.
- **Per-trim shrinks gso_size** — defense against per-trimmed skb claiming larger MSS.
- **Per-segment per-segment-skb sized to mss** — defense against oversized GSO segments.
- **Per-pskb_expand_head: dataref==1 fast-path** — defense against per-shared-data eager-copy.
- **Per-unshare allocates new linear** — defense against post-clone aliasing causing data races.
- **Per-copy_bits offset/len bounds** — defense against per-frag OOB read leaking memory.
- **Per-split keeps per-frag refcount balanced** — defense against per-shared-frag double-free.
- **Per-segment per-skb uses gso_size only from origin** — defense against per-feature-derived MSS divergence.
- **Per-skb_share_check allocates only when needed** — defense against unnecessary alloc DoS.

