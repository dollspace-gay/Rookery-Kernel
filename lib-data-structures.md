---
title: "Tier-3: lib/data-structures — kernel data structure library"
tags: ["design-doc", "tier-3", "lib"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 component design for the generic data structures in `lib/`. Owns the design of every kernel-internal collection type: red-black trees, doubly-linked lists, the maple tree (range-keyed), xarray (radix tree of pointers), associative arrays, btrees, interval trees, kfifo (lock-free SPSC ring), heaps, sort, bit search, hash tables, klist (locked list with notification), and stackdepot (the kernel's stack-trace deduplication store).

Per `lib/00-overview.md` this Tier-3 is the largest single collection of foundational primitives in the project. It is depended on by `mm/` (VMA tree → maple_tree, page cache → xarray), `fs/` (dcache → custom hashtable, pagecache → xarray, mount tree → custom), `kernel/` (scheduler runqueues → custom rbtree, futex → hashtable), `net/` (FIB → custom trie, conntrack → hashtable), `block/` (request queues → custom), and effectively every other subsystem.

### Requirements

- REQ-1: All upstream rust-for-linux abstractions for `lib/data-structures` concepts (`kernel::list::List<T>`, `kernel::rbtree::RBTree<K,V>`, `kernel::xarray::XArray<T>`) are the canonical Rust types. Rookery extends these — does NOT introduce parallel `kernel::lib::*` namespaces.
- REQ-2: Every data structure in scope has a Rust API with at least the following methods where applicable: `new()`, `insert(key, value)`, `remove(key)`, `lookup(key)`, `iter()`, `len()`, `is_empty()`. Method names match Rust convention; the upstream C function set IS the implementation reference, not the API surface.
- REQ-3: Generic-typed data structures (rbtree, btree, hashtable, xarray, maple_tree, kfifo, min_heap, bitmap iter helpers) accept Rust generic type parameters (`<T>`, `<K, V>`) — not the upstream pattern of macro-typed-in-place.
- REQ-4: Intrusive list (`include/linux/list.h`) is exposed as `kernel::list::List<T>` per upstream rust-for-linux convention; intrusive nodes use `pin_init` + `ListLinks` for safety.
- REQ-5: Lock-free / RCU-friendly variants (xarray, maple_tree, hlist_nulls, klist) preserve their concurrency contracts. TLA+ models for xarray and maple_tree are mandatory (per `lib/00-overview.md` Layer 2). RCU-read-side helpers expose `kernel::sync::rcu::Guard` borrowing semantics.
- REQ-6: Bitmap operations preserve the upstream bit-numbering convention (LSB of word 0 = bit 0). Userspace-exposed bitmaps (cpu masks, fdsets) are byte-identical.
- REQ-7: Sort APIs preserve stability behavior identical to upstream (sort is NOT guaranteed stable; if a caller needs stability, it must use `list_sort` which IS stable per `lib/list_sort.c`).
- REQ-8: kfifo preserves its power-of-two capacity invariant + lock-free SPSC concurrency contract.
- REQ-9: Stackdepot's hash function is bit-identical so a stack trace produces the same `depot_handle_t` on Rookery as upstream.
- REQ-10: Each data structure declares its `unsafe`-block clusters (Layer 1 SAFETY proofs), invariant harnesses (Layer 3 — mandatory per `lib/00-overview.md` REQ-9 inheritance), and any TLA+ models (Layer 2 for concurrency primitives).
- REQ-11: A Rookery-specific clippy lint (issue #3) `bare_arithmetic_on_size` ensures bitmap and ring-buffer arithmetic uses `checked_*` / `wrapping_*` explicitly.
- REQ-12: Every data structure that holds RCU-protected pointers exposes a `RcuList<T>` / `RcuTree<T>`-style typed wrapper rather than raw `core::ptr::*` operations.

### Acceptance Criteria

- [ ] AC-1: `kernel::list::List<T>`, `kernel::rbtree::RBTree<K,V>`, `kernel::xarray::XArray<T>` are imported from upstream rust-for-linux without re-declaration in Rookery's tree. (covers REQ-1)
- [ ] AC-2: For every collection in scope, a unit test exercises `insert / lookup / remove / iter` and asserts equivalence to a `BTreeMap`-from-`alloc` reference (or to upstream's behavior for byte-level invariants). (covers REQ-2, REQ-3, REQ-4)
- [ ] AC-3: A Kani harness for each of: rbtree red-black invariants (root black; no two consecutive red; equal black-height across paths); doubly-linked list link integrity (`next->prev == self`); maple tree leaf disjointness + in-order; xarray radix node validity; kfifo head/tail-modulo-capacity; min_heap parent ≤ children; bitmap bit-bounds; stackdepot chain integrity; iov_iter cursor (cross-ref `lib/usercopy.md`). All harnesses pass under `make verify`. (covers REQ-10, plus the Layer-3 mandate inherited from `lib/00-overview.md`)
- [ ] AC-4: TLA+ models in `models/lib/{xarray,maple_tree,stackdepot}.tla` pass under `make tla`. (covers REQ-5)
- [ ] AC-5: A stackdepot-hash golden test asserts bit-identical `depot_handle_t` for curated stack traces between Rookery and upstream. (covers REQ-9)
- [ ] AC-6: A bitmap operations test exercises every documented bit op (`set_bit`, `clear_bit`, `test_bit`, `find_first_bit`, `find_next_bit`, `find_last_bit`, `bitmap_and`, `bitmap_or`, `bitmap_xor`, `bitmap_andnot`, `bitmap_complement`, `bitmap_equal`, `bitmap_intersects`, `bitmap_subset`, `bitmap_empty`, `bitmap_full`, `bitmap_weight`, `bitmap_shift_right`, `bitmap_shift_left`, `bitmap_replace`, `bitmap_alloc`, `bitmap_free`, `bitmap_zalloc`, …) and produces byte-identical output for matching inputs. (covers REQ-6)
- [ ] AC-7: A `list_sort` stability test asserts stable ordering on inputs with equal keys (matches upstream's stable-sort guarantee). A `sort` (`lib/sort.c`) test does NOT assert stability (matches upstream's unstable guarantee). (covers REQ-7)
- [ ] AC-8: A kfifo concurrency test (one producer + one consumer thread) over 1M operations produces no torn reads + no lost entries. (covers REQ-8)
- [ ] AC-9: A grep over Rookery for `unsafe { *p =` or raw `core::ptr::write_unaligned` outside `bindings/` returns matches only inside Layer-1-proofed unsafe blocks (covered by `00-rust-conventions.md` REQ-8). (covers REQ-12)

### Architecture

### Rust API shape

#### Already-present in upstream rust-for-linux (extend, don't fork)

- **`kernel::list::List<T>`** — `rust/kernel/list.rs`. Intrusive doubly-linked list with `ListLinks` field embedded in `T`. API: `push_back`, `push_front`, `pop_back`, `pop_front`, `iter`, `len`, `is_empty`, `into_iter`. Pin-required; uses `pin_init` machinery.
- **`kernel::rbtree::RBTree<K, V>`** — `rust/kernel/rbtree.rs`. Generic-typed rbtree with `K: Ord`. API: `insert`, `remove`, `get`, `iter`, `iter_mut`, `entry`, `len`, `is_empty`. Wraps upstream `lib/rbtree.c`'s C primitives via bindings.
- **`kernel::xarray::XArray<T>`** — `rust/kernel/xarray.rs`. Wraps upstream `lib/xarray.c`. Partial coverage — extend with: range operations, marks, RCU iteration helpers.

#### Rookery to author

- **`kernel::maple_tree::MapleTree<T>`** (NEW) — wraps `lib/maple_tree.c`. Range-keyed (`Range<usize>` keys). API: `insert(range, value)`, `remove(range)`, `find(addr) -> Option<(Range, &T)>`, `iter`, `iter_range(range)`. Extensively used by `mm/` (the VMA tree); the doc home for the cross-arch maple_tree contract.
- **`kernel::radix_tree::RadixTree<T>`** (legacy, lower priority) — only for places where xarray hasn't displaced it. Mark `#[deprecated]` in module docs to nudge consumers to xarray.
- **`kernel::generic_radix_tree::GenericRadixTree<T>`** — wrapper around `lib/generic-radix-tree.c`. Used by bcachefs.
- **`kernel::btree::Btree<K, V>`** — wraps `lib/btree.c`. Generic-typed.
- **`kernel::interval_tree::IntervalTree<T>`** — wraps `lib/interval_tree.c`. Augmented rbtree.
- **`kernel::kfifo::Kfifo<T, const N: usize>`** — wraps `lib/kfifo.c`. Power-of-two const-N capacity. Provides `Sender<T>` / `Receiver<T>` split for lock-free SPSC use.
- **`kernel::sort::sort_slice<T, F>(slice: &mut [T], cmp: F)`** — wraps `lib/sort.c`. Unstable. Plus `sort_slice_stable` that delegates to `list_sort` for stability.
- **`kernel::min_heap::MinHeap<T>`** — wraps `lib/min_heap.c`. Generic-typed. API: `push`, `pop`, `peek`.
- **`kernel::bsearch::binary_search<T, F>`** — wraps `lib/bsearch.c`. Generic.
- **`kernel::assoc_array::AssocArray<K, V>`** — wraps `lib/assoc_array.c`. Used by keyrings (`security/keys`).
- **`kernel::bitmap::Bitmap<const N: usize>`** — wraps `lib/bitmap.c`. Const-N for static-size bitmaps; `kernel::bitmap::DynBitmap` for runtime-sized.
- **`kernel::find_bit::*`** — bitmap iteration helpers. Slice-of-`u64`-typed API.
- **`kernel::hexdump::*`** — formatting helpers.
- **`kernel::stackdepot::StackDepot`** — wraps `lib/stackdepot.c`. API: `save_stack(trace) -> Handle`, `fetch_stack(handle) -> &[StackFrame]`. Used by KASAN, KMSAN, page_owner, slub_debug.
- **`kernel::klist::Klist<T>`** — wraps `lib/klist.c`. Locked list with deferred-free + notifier. Used by driver model (`drivers/base/`).
- **`kernel::hashtable::HashTable<const BITS: usize, T>`** — generic-typed wrapper around `include/linux/hashtable.h`'s C macros. Power-of-two bucket count (BITS bits → 2^BITS buckets).
- **`kernel::list_sort::list_sort<T, F>(list: &mut List<T>, cmp: F)`** — wraps `lib/list_sort.c`. Stable; mergesort.

### Locking and concurrency

Most data structures expose lock-free or caller-locked APIs. The collection's documentation states the contract explicitly:

- **Caller-locked** (caller provides external lock): rbtree, list, btree, interval_tree, sort, min_heap, bsearch, bitmap (bitops have atomic variants `set_bit_atomic` etc. but bulk ops are caller-locked), assoc_array, hashtable, klist (provides its own internal lock).
- **Internal RCU + spinlock**: xarray (xa_lock spinlock + RCU readers), maple_tree (per-node spinlock + RCU readers), stackdepot (per-pool RCU).
- **Lock-free SPSC**: kfifo (one producer, one consumer with memory ordering).

TLA+ models live with the lock-free/RCU collections:
- `models/lib/xarray.tla` (Layer 2; per `lib/00-overview.md`)
- `models/lib/maple_tree.tla` (Layer 2)
- `models/lib/stackdepot.tla` (Layer 2)
- `models/lib/kfifo_spsc.tla` (Layer 2 — added in this Tier-3 spec)

### Error handling

Standard `Result<T, kernel::error::Error>` per `00-rust-conventions.md` REQ-6:
- `Err(ENOMEM)` — out of memory in any growing collection
- `Err(EBUSY)` — already-exists in single-occupancy collections (keyring assoc_array)
- `Err(EINVAL)` — bad input (bitmap range out of bounds; sort comparator violates total order — debug-only check)

For never-failing operations (lookup-miss, remove of non-existent key) the API returns `Option<T>` rather than `Result`.

### Reuse vs. authoring summary

| Type | Already in upstream rust/kernel/ | Rookery to author |
|---|---|---|
| `List<T>`, `ListLinks` | ✓ | extend (no major work) |
| `RBTree<K, V>` | ✓ | extend (augmented rbtree variant in `rbtree_augmented.h` not yet wrapped) |
| `XArray<T>` | ✓ partial | extend with marks + RCU iter helpers |
| MapleTree, Btree, IntervalTree, Kfifo, MinHeap, AssocArray, Bitmap, FindBit, Hexdump, StackDepot, Klist, HashTable, list_sort, sort, bsearch, generic_radix_tree, radix_tree | — | author from scratch |

### Out of Scope

- Domain-specific data structures (FIB tries, conntrack hash, dcache hash) — those live in their owning subsystem's Tier-3 doc.
- Generic algorithms beyond what `lib/` ships (e.g., suffix arrays, Bloom filters not in upstream) — out of v0.
- Concurrent / lock-free data structures beyond what upstream provides (e.g., crossbeam-style atomics-based queues) — Rust uses the upstream model: kernel locks + RCU.
- 32-bit-only paths.
- Implementation code.

### upstream references in scope

| Component | Upstream path | Approx LoC |
|---|---|---|
| Red-black tree | `lib/rbtree.c` + `include/linux/rbtree.h` + `include/linux/rbtree_augmented.h` | 618 + headers |
| Doubly-linked list (intrusive) | `include/linux/list.h` (header-only generic; debug helpers in `lib/list_debug.c`) | header + 72 |
| List sort (mergesort over `list_head`) | `lib/list_sort.c` + `include/linux/list_sort.h` | 247 |
| Maple tree (range-keyed B-tree variant) | `lib/maple_tree.c` + `include/linux/maple_tree.h` | 7042 + 936 |
| xarray (radix tree of pointers) | `lib/xarray.c` + `include/linux/xarray.h` | 2481 + 1915 |
| Radix tree (legacy; phasing out) | `lib/radix-tree.c` + `include/linux/radix-tree.h` | 1608 |
| Generic radix tree (bcachefs use) | `lib/generic-radix-tree.c` + `include/linux/generic-radix-tree.h` | 230 |
| Btree (in-memory) | `lib/btree.c` + `include/linux/btree.h` | 795 |
| Interval tree (rbtree augmented for ranges) | `lib/interval_tree.c` + `include/linux/interval_tree.h` | 156 |
| kfifo (lock-free SPSC ring) | `lib/kfifo.c` + `include/linux/kfifo.h` | 595 |
| Sort (qsort/heapsort) | `lib/sort.c` + `include/linux/sort.h` | 357 |
| Min-heap | `lib/min_heap.c` + `include/linux/min_heap.h` | 70 |
| Binary search | `lib/bsearch.c` + `include/linux/bsearch.h` | 36 |
| Associative array (used by keyrings) | `lib/assoc_array.c` + `include/linux/assoc_array.h` | 1723 |
| Bitmap | `lib/bitmap.c` + `lib/bitmap-str.c` + `include/linux/bitmap.h` | 897 + 510 |
| Find bit (efficient bitmap iteration) | `lib/find_bit.c` + `include/linux/find.h` | 310 |
| Hexdump | `lib/hexdump.c` (header-only API in `include/linux/printk.h`) | 296 |
| Stackdepot (stack-trace dedup) | `lib/stackdepot.c` + `include/linux/stackdepot.h` | 878 |
| klist (locked + notifier list) | `lib/klist.c` + `include/linux/klist.h` | (small) |
| Hashtable (header-only macros) | `include/linux/hashtable.h` | header |

Total: ~21K LoC of upstream C plus headers. Rust counterparts in upstream rust-for-linux: `list.rs` (covers `include/linux/list.h`), `rbtree.rs` (covers `lib/rbtree.c`), `xarray.rs` (covers `lib/xarray.c` partially) — Rookery extends these and authors the rest.

### compatibility contract

`lib/data-structures` is mostly internal kernel code; the userspace-visible compat surface is narrow but real:

| Interface | Compat level |
|---|---|
| `/proc/<pid>/maps` ordering (consumes the maple tree's range-iter order) | Format-identical; the maple tree's iteration order MUST produce the same address-sorted listing as upstream |
| Stackdepot stable hash for a given stack trace | Bit-identical hash function (so `/sys/kernel/debug/...` stackdepot output matches upstream when a tool reads it) |
| `struct hlist_head` / `struct hlist_node` field offsets (referenced by macro-accessing inline code in many subsystems) | Field-offset-identical |
| Bitmap representation in any UAPI struct that exposes one (e.g., `cpu_set_t`-equivalent via `sched_*affinity` syscalls) | Bit-identical layout |

The deeper data-structure invariants (tree balance, hash collision resolution, sort stability) are kernel-internal and Rust-implementation can reorganize freely so long as the user-visible compat surface is preserved.

### verification

### Layer 1: Kani SAFETY proofs (mandatory per `00-rust-conventions.md` REQ-8)

Anticipated `unsafe` clusters and proof harnesses:

| Component | Unsafe surface | Harness |
|---|---|---|
| List | Pin projection, link manipulation under pin invariants | `kani::proofs::lib::list::link_safety` |
| RBTree | Tree-rotation pointer fixups | `kani::proofs::lib::rbtree::rotate_safety` |
| Maple tree | Slot-array index arithmetic, RCU-protected child reads | `kani::proofs::lib::maple_tree::walk_safety` |
| xarray | Radix-bit shifts, RCU pointer dereference | `kani::proofs::lib::xarray::walk_safety` |
| Btree | Slot-shift on insert/remove | `kani::proofs::lib::btree::shift_safety` |
| Kfifo | Wraparound arithmetic on power-of-two capacity | `kani::proofs::lib::kfifo::wrap_safety` |
| Bitmap | Word-index + bit-index arithmetic | `kani::proofs::lib::bitmap::index_safety` |
| Stackdepot | Hash-bucket pointer chase under RCU | `kani::proofs::lib::stackdepot::lookup_safety` |
| Klist | Lock + RCU coordination on add/remove | `kani::proofs::lib::klist::membership_safety` |
| Sort/list_sort | Slice swap operations | `kani::proofs::lib::sort::swap_safety` |

### Layer 2: TLA+ models (mandatory for novel concurrency)

Inherited from `lib/00-overview.md`:
- `models/lib/xarray.tla` — RCU readers + spinlock writers; readers always observe coherent tree state
- `models/lib/maple_tree.tla` — analogous
- `models/lib/stackdepot.tla` — RCU hash invariants

Added in this Tier-3:
- `models/lib/kfifo_spsc.tla` — proves SPSC ring's lock-free producer/consumer ordering: producer sees no torn reads on stale-tail; consumer sees no lost entries.
- `models/lib/klist.tla` — proves klist's add/remove + notifier-chain semantics under racing iteration.

### Layer 3: Kani harnesses for data-structure invariants (mandatory per `lib/00-overview.md` REQ-9)

Per the parent doc, every data structure in this scope has a mandatory invariant harness. Enumerated:

| Data structure | Invariant | Harness |
|---|---|---|
| RBTree | Root is black; no two consecutive red nodes; all root-to-leaf paths have equal black-height | `kani::proofs::lib::rbtree::invariants` |
| List (klist + ll-list + list_head) | Doubly-linked correctness: `next->prev == self`, `prev->next == self`; no cycles in linear lists | `kani::proofs::lib::list::link_invariants` |
| Maple tree | Range-keyed disjointness; all leaves in-order; parent-pivot bounds correct | `kani::proofs::lib::maple_tree::invariants` |
| xarray | Radix-tree invariants: every internal node has ≥ 1 valid child; no orphan slots | `kani::proofs::lib::xarray::invariants` |
| Btree | All nodes within key-bounds for their position | `kani::proofs::lib::btree::invariants` |
| Interval tree | rbtree invariants + augmented `__subtree_last` field consistent | `kani::proofs::lib::interval_tree::invariants` |
| Kfifo | Power-of-two ring; `head - tail` always within capacity; producer/consumer order | `kani::proofs::lib::kfifo::ring_invariants` |
| Min-heap | Parent ≤ children for every node | `kani::proofs::lib::min_heap::heap_invariant` |
| Bitmap | Boundary-bit handling; `count_set_bits` correctness | `kani::proofs::lib::bitmap::set_bit_invariants` |
| Stackdepot | Hash-chain integrity; bucket head reachable from at least one entry | `kani::proofs::lib::stackdepot::chain_invariants` |
| Iov_iter | Cursor: `count + remaining == initial`; segment cursor never beyond segment length (cross-ref `lib/usercopy.md`) | `kani::proofs::lib::iov_iter::cursor_invariants` |
| Klist | Lock-protected list invariants + refcount consistency | `kani::proofs::lib::klist::invariants` |

### Layer 4: Functional correctness (opt-in)

Strong opt-in candidates (not v0-mandatory, but high-leverage):
- **Sort** (`lib/sort.c`) — qsort/heapsort correctness + termination via Creusot.
- **List_sort** — stable mergesort correctness.
- **Min-heap** push/pop preserves heap invariant — Creusot.
- **Maple tree** range-key invariants under arbitrary insert/remove sequences — Verus (very ambitious; tracked but not v0).

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | This component's responsibility | Default inherited from |
|---|---|---|
| **AUTOSLAB** | Per-collection-type slab caches via `KmemCache::<NodeOf<T>>::new()` per upstream rust-for-linux; `core::any::type_name` provides the per-type tag | `00-security-principles.md` § Mandatory |
| **MEMORY_SANITIZE** (zero on free) | Each Tier-3 data-structure unit declares its node type as `SLAB_NO_ZERO_ON_FREE` only when explicitly safe (default leaves `vm.zero_on_free=1` on); sensitive types like keyring assoc_array nodes always force-zero | `00-security-principles.md` § Default-on configurable off |
| **CONSTIFY** | All vtable-style structs (e.g., comparator function pointers in sort) are `static FOO: SortVtable = …` — Rust enforces const | `00-security-principles.md` § Mandatory |

### Row-1 features consumed by this component

- **UDEREF**: this component never accepts userspace pointers directly. Bitmap operations on user-supplied bitmasks (e.g., for sched_setaffinity) take `UserPtr<u64; N>` from the syscall layer; bitmap operations themselves work on kernel-side `&[u64]`.
- **SIZE_OVERFLOW**: bitmap word/bit arithmetic, kfifo head/tail wraparound, list_sort recursion-depth all use `checked_*` / `wrapping_*` explicitly per the `bare_arithmetic_on_size` clippy lint (issue #3).

### Row-2 / GR-RBAC integration

This component does NOT directly call LSM hooks; collections are infrastructure. Higher-tier consumers (e.g., `security/keys` via assoc_array) invoke LSM hooks at their own boundaries.

### Userspace-visible behavior changes

None beyond what `lib/00-overview.md` already documents (vsprintf %p extensions, dynamic_debug control, KUnit TAP — all in other Tier-3 docs).

### Verification

(See § Verification above; consolidated table covers Layers 1–4.)

