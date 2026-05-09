---
title: "Tier-3: net/netfilter/nft-sets — set storage backends (hash / rbtree / pipapo / bitmap)"
tags: ["design-doc", "tier-3", "net", "netfilter"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the four nftables set storage backends — pluggable per-set implementations selected by NEWSET's `NFTA_SET_POLICY` (performance vs. memory tradeoff) and the requested key/value types. Sets are queried by the `lookup` expression (cross-ref `nft-expressions.md` REQ-6) and mutated by `dynset` expression. Used everywhere: ip-blocklists, allow-lists, port-ranges, per-source-IP rate-limit tables, hash:ip,port concatenations.

Four backends:
- **`nft_set_hash`** — per-bucket hashtable; O(1) lookup; any-key-type; default for performance-prioritized sets
- **`nft_set_rbtree`** — interval red-black tree; O(log n); supports range matching (e.g., `1.2.3.0-1.2.3.255` matches `1.2.3.42`)
- **`nft_set_pipapo`** — patricia-trie + sparse bitmap (PIle PAcket POlicy); O(packet-fields) lookup for multi-dimension keys (concatenations); shines for per-(saddr, daddr, sport, dport) tuples
- **`nft_set_bitmap`** — bit-array indexed by integer key; only for key types ≤ 8-bit (e.g., DSCP, IP protocol, ICMP type)

Sub-tier-3 of `net/netfilter/00-overview.md`. Pairs with `net/netfilter/nft-api.md` (NEWSET / NEWSETELEM API), `net/netfilter/nft-expressions.md` (lookup + dynset consumers), `net/netfilter/nft-core.md` (VM eval).

### Requirements

- REQ-1: Four backend modules registered: `nft_set_hash` / `nft_set_rbtree` / `nft_set_pipapo` / `nft_set_bitmap`.
- REQ-2: `struct nft_set_ops` vtable layout-equivalent for first cache-line.
- REQ-3: Per-backend `features` flag declares which `NFT_SET_*` flags supported; `select_ops` returns matching backend per requested features.
- REQ-4: `nft_set_hash`: rhashtable; per-bucket lock; O(1) avg lookup; supports any key type ≤ NFT_SET_MAX_KEY.
- REQ-5: `nft_set_rbtree`: interval rbtree; O(log n) lookup; supports INTERVAL flag; interval queries identical to upstream's interval-tree algorithm.
- REQ-6: `nft_set_pipapo`: sparse bitmap; supports CONCAT flag; AVX2-accelerated on x86_64.
- REQ-7: `nft_set_bitmap`: bit-array; ≤256-element keys; INTERVAL flag rejected.
- REQ-8: Per-element extensions (`nft_set_ext`): key / key_end / data / timeout / expiration / userdata / expressions / objref; layout-byte-identical accessors.
- REQ-9: Per-TIMEOUT-set GC: periodic walk; expired elements removed.
- REQ-10: `NFTA_SET_POLICY` semantics: PERFORMANCE → hash by default; MEMORY → rbtree/bitmap by default.
- REQ-11: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct nft_set_ops` byte-identical first cache-line. (covers REQ-2)
- [ ] AC-2: hash test: `nft add set ... blocklist { type ipv4_addr; size 100000; }`; insert 100K IPs; lookup ≤1us avg per element. (covers REQ-1, REQ-4)
- [ ] AC-3: rbtree test: `nft add set ... ranges { type ipv4_addr; flags interval; }`; insert `1.2.3.0-1.2.3.255` interval; lookup of `1.2.3.42` matches; lookup of `5.6.7.8` doesn't. (covers REQ-1, REQ-5)
- [ ] AC-4: pipapo test: `nft add set ... services { type ipv4_addr . inet_service; }`; insert (`1.2.3.4`, 80); lookup matches. (covers REQ-1, REQ-6)
- [ ] AC-5: bitmap test: `nft add set ... protos { type inet_proto; }`; insert proto 6 (TCP); lookup matches. (covers REQ-1, REQ-7)
- [ ] AC-6: pipapo AVX2 test on x86_64: sets with concatenation lookups demonstrate AVX2 acceleration; non-AVX2 fallback works on legacy CPUs. (covers REQ-6)
- [ ] AC-7: TIMEOUT test: `nft add set ... shortlived { type ipv4_addr; timeout 5s; }`; insert 1.2.3.4; wait 6s; element auto-removed. (covers REQ-8, REQ-9)
- [ ] AC-8: MAP test: `nft add map ... port_to_chain { type inet_service : verdict; }; add element ... port_to_chain { 80 : jump web_chain }`; rule looks up port → jumps to chain. (covers REQ-8)
- [ ] AC-9: PERFORMANCE policy test: NEWSET with `NFTA_SET_POLICY=PERFORMANCE` selects hash backend by default. (covers REQ-10)
- [ ] AC-10: MEMORY policy test: NEWSET with `NFTA_SET_POLICY=MEMORY` and small key (≤8-bit) → bitmap; larger key + INTERVAL → rbtree. (covers REQ-10)
- [ ] AC-11: feature negotiation: `select_ops` rejects rbtree for non-INTERVAL+non-CONCAT sets when MEMORY policy + small key would prefer bitmap. (covers REQ-3)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-11)

### Architecture

### Rust module organization

- `kernel::net::netfilter::nft::sets::Registry` — per-system nft_set_type list
- `kernel::net::netfilter::nft::sets::SetOps` — `struct nft_set_ops` wrapper
- `kernel::net::netfilter::nft::sets::Ext` — `nft_set_ext` per-extension stacking
- `kernel::net::netfilter::nft::sets::hash::Hash` — rhashtable backend
- `kernel::net::netfilter::nft::sets::rbtree::Rbtree` — interval rbtree backend
- `kernel::net::netfilter::nft::sets::pipapo::Pipapo` — sparse bitmap backend
- `kernel::net::netfilter::nft::sets::pipapo::Avx2` — x86_64 AVX2 acceleration
- `kernel::net::netfilter::nft::sets::bitmap::Bitmap` — bit-array backend
- `kernel::net::netfilter::nft::sets::Gc` — per-TIMEOUT-set periodic GC
- `kernel::net::netfilter::nft::sets::Selector` — `select_ops` policy logic

### Locking and concurrency

- **rhashtable per-bucket spinlocks** (hash backend): bit_spin_lock; lockless reads via RCU
- **Per-set seqcount** (rbtree backend): seqcount-protected interval walk; mutators take seqcount + per-set lock
- **Per-set spinlock** (pipapo + bitmap backends): protects per-field bitmap mutation
- **RCU**: lookup hot path RCU-side across all backends; mutators use RCU-defer free

### Error handling

- `Err(EINVAL)` — bad NLA / unsupported feature combination
- `Err(EEXIST)` — duplicate element
- `Err(ENOENT)` — element not found (for delete)
- `Err(ENOMEM)` — alloc fail
- `Err(EOPNOTSUPP)` — feature not supported by selected backend

### Out of Scope

- VM eval engine (cross-ref `nft-core.md`)
- Per-expression types (cross-ref `nft-expressions.md`)
- API + transactions (cross-ref `nft-api.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Backend | Upstream path | Key features |
|---|---|---|
| `nft_set_hash` | `net/netfilter/nft_set_hash.c` | rhashtable + per-bucket lock; any key type ≤ NFT_SET_MAX_KEY |
| `nft_set_rbtree` | `net/netfilter/nft_set_rbtree.c` | rb_root + per-set seqcount; supports interval/range matching |
| `nft_set_pipapo` | `net/netfilter/nft_set_pipapo.c` | per-field bitmap; concatenation-friendly; AVX2-accelerated lookup on x86_64 |
| `nft_set_bitmap` | `net/netfilter/nft_set_bitmap.c` | bit-array; for ≤256-element key spaces (DSCP / IP proto / etc.) |
| Public API | `include/net/netfilter/nf_tables.h` | `nft_set_ops` vtable + `NFT_SET_*` flags |

### compatibility contract

### `struct nft_set_ops` vtable (per-backend)

```c
struct nft_set_ops {
    bool   (*lookup)(const struct net *, const struct nft_set *, const u32 *, const struct nft_set_ext **);
    bool   (*update)(struct nft_set *, const u32 *, void *(*new)(struct nft_set *, const struct nft_expr *, struct nft_regs *), const struct nft_expr *, struct nft_regs *, const struct nft_set_ext **);
    bool   (*delete)(const struct nft_set *, const u32 *);
    int    (*insert)(const struct net *, const struct nft_set *, const struct nft_set_elem *, struct nft_set_ext **);
    void   (*activate)(const struct net *, const struct nft_set *, const struct nft_set_elem *);
    void * (*deactivate)(const struct net *, const struct nft_set *, const struct nft_set_elem *);
    bool   (*flush)(const struct net *, const struct nft_set *, void *);
    void   (*remove)(const struct net *, const struct nft_set *, const struct nft_set_elem *);
    void   (*walk)(const struct nft_ctx *, struct nft_set *, struct nft_set_iter *);
    void * (*get)(const struct net *, const struct nft_set *, const struct nft_set_elem *, unsigned int);
    int    (*ksize)(u32);
    int    (*usize)(u32);
    u32    (*adjust_maxsize)(const struct nft_set *);
    int    (*init)(const struct nft_set *, const struct nft_set_desc *, const struct nlattr * const nla[]);
    void   (*destroy)(const struct nft_ctx *, const struct nft_set *);
    void   (*gc_init)(const struct nft_set *);
    unsigned int   elemsize;
    u32           features;
};
```

Layout-equivalent. Per-backend module exports a `nft_set_type` carrying `(features, ops_select)`.

### Per-backend `features` flag

Each backend declares which `NFT_SET_*` flags it supports:
- `NFT_SET_INTERVAL` — interval/range keys (only rbtree + pipapo)
- `NFT_SET_MAP` — element carries associated value (any backend)
- `NFT_SET_TIMEOUT` — per-element timeout (any backend except bitmap)
- `NFT_SET_EVAL` — element is a stateful object (counter / quota / dynset)
- `NFT_SET_OBJECT` — element references stateful object
- `NFT_SET_CONCAT` — multi-field concatenated keys (pipapo's strength)
- `NFT_SET_ANONYMOUS` — set is anonymously named (lifetime tied to using rule)

Per-backend `select_ops` callback returns either the matching `ops` or NULL based on requested features. Identical contract.

### `nft_set_hash` (rhashtable backend)

Per-bucket linked list with per-bucket lock; rhashtable-style. Lookup O(1) (avg); insert + remove also O(1) (amortized). Resize on size threshold via `rhashtable_walk_*`. Per-bucket lock fairness via `bit_spin_lock`.

Optimal for: large sets of unique-element keys (10K+ elements), exact-match queries.

### `nft_set_rbtree` (interval rbtree backend)

Per-set rb_root holding interval nodes (`[start, end]` pairs). Lookup walks tree with O(log n) comparisons; supports interval queries (find element whose range contains key). Identical to RFC interval-tree.

Optimal for: range matching (`192.0.2.0/24` ranges; port ranges).

### `nft_set_pipapo` (sparse-bitmap concatenation backend)

PIle PAcket POlicy: per-field bitmap indexed by per-byte fields of the concatenation key; intersection across fields produces the set of matching elements. AVX2-accelerated on x86_64 via `nft_pipapo_avx2.c` (cross-ref future Tier-3 if extracted).

Optimal for: large multi-field concatenations (`hash:ip,port,proto`).

### `nft_set_bitmap` (per-key-byte bit-array)

`elements[256]` bit-array indexed by key byte; for key sizes ≤ 8 bits. Lookup: `(elements[key / 8] >> (key % 8)) & 1`.

Optimal for: small finite key spaces (DSCP — 6 bits, IP proto — 8 bits, ICMP type — 8 bits).

### `nft_set_ext` (per-element extensions)

Each element carries per-extension extras stacked via `nft_set_ext`:
- key (always present)
- key-end (for INTERVAL sets)
- data (for MAP sets — associated value)
- timeout (for TIMEOUT sets — per-element expiry)
- expiration (timestamp of next expiry check)
- userdata (opaque user data)
- expressions (for EVAL — per-element nested expressions like counters)
- objref (for OBJECT — reference to stateful object)
- pad (alignment)

Per-extension layout matches upstream's `nft_set_ext_*` accessors.

### GC for TIMEOUT sets

Per-backend periodic GC walks elements; expired ones removed. Per-set `gc_interval` configurable (default 30s).

### `NFTA_SET_POLICY` (performance vs. memory)

NFT_SET_POL_PERFORMANCE (0): backend selection prefers fast lookup (hash by default).
NFT_SET_POL_MEMORY (1): backend prefers memory efficiency (rbtree or bitmap by default for small spaces).

Identical policy semantics.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| rhashtable insert/erase under per-bucket lock | `kani::proofs::net::netfilter::nft::sets::hash_safety` |
| rbtree interval insert/erase under seqcount | `kani::proofs::net::netfilter::nft::sets::rbtree_safety` |
| pipapo per-field bitmap mutation under set spinlock | `kani::proofs::net::netfilter::nft::sets::pipapo_safety` |
| bitmap per-bit set/clear (no out-of-bounds; key ≤ 256) | `kani::proofs::net::netfilter::nft::sets::bitmap_safety` |
| nft_set_ext stacking + accessor (per-extension offset arithmetic) | `kani::proofs::net::netfilter::nft::sets::ext_safety` |
| Per-TIMEOUT GC walk (RCU-side; no use-after-free of just-removed elements) | `kani::proofs::net::netfilter::nft::sets::gc_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; relies on `models/net/nft_vm_eval.tla` from `nft-core.md` for transaction-atomicity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| rhashtable | every element appears in exactly the bucket determined by `set->key_hashfn`; sets use rhashtable's standard invariants | `kani::proofs::net::netfilter::nft::sets::hash_invariants` |
| rbtree | sorted by key.start ascending; per-element [start, end] non-overlapping (when INTERVAL); rbtree balance properties hold | `kani::proofs::net::netfilter::nft::sets::rbtree_invariants` |
| pipapo per-field bitmaps | per-field count = element count for keys with non-NULL bit at that field's position | `kani::proofs::net::netfilter::nft::sets::pipapo_invariants` |
| bitmap | popcount(elements) = set's element count; key index < 256 always | `kani::proofs::net::netfilter::nft::sets::bitmap_invariants` |
| Per-element extensions | extension layout matches `nft_set_ext_*` offset accessors; offsets non-overlapping | `kani::proofs::net::netfilter::nft::sets::ext_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Backend equivalence theorem** via Verus — proves: ∀ key set `K` and queries `q`, all four backends return same lookup result for q ∈ K queries (modulo INTERVAL/CONCAT support).
- **Pipapo AVX2 equivalence theorem** via Verus — proves: AVX2-accelerated pipapo lookup result equals scalar-lookup result for any input.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-set + per-element refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-backend element slab caches | § Mandatory |
| **CONSTIFY** | per-backend `nft_set_ops` instances `static const` | § Mandatory |
| **MEMORY_SANITIZE** | freed elements cleared (carry match keys + map values + secctx) | § Default-on configurable off |
| **SIZE_OVERFLOW** | per-backend lookup arithmetic uses checked operators | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, CONSTIFY, MEMORY_SANITIZE, SIZE_OVERFLOW**: see above
- **USERCOPY**: NLA parsing in NEWSETELEM uses bound-checked accessors
- **KERNEXEC**: per-backend dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook: same as `nft-api.md` (CAP_NET_ADMIN gate; mutations only via API).
- Default GR-RBAC policy: empty.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

