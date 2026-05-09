---
title: "Tier-3: block/bio — block I/O request"
tags: ["design-doc", "tier-3", "block"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for `struct bio` — the block I/O request data structure. Owns bio lifecycle (alloc → split → merge → endio → free), bio iteration via `bvec_iter` cursor, page-vec management, bio chaining (cloning + linking for resubmit), DMA-buffer integration, REQ_OP_* operation enumeration, and the higher-level helpers (`blk_lib`: discard, zeroout, sec-discard, write-zeroes).

Sub-tier-3 of `block/00-overview.md`. The bio is the second-most-touched data structure in the block layer (after request); every disk-FS read/write produces bios that the block layer dispatches to drivers.

### Requirements

- REQ-1: `struct bio` first-cache-line layout-equivalent.
- REQ-2: `struct bio_vec` byte-identical layout.
- REQ-3: REQ_OP_* + REQ_* numeric values byte-identical.
- REQ-4: bio alloc family: `bio_alloc`, `bio_kmalloc`, `bio_alloc_kiocb`, `bio_alloc_clone`, `bio_init` semantics identical. Per-pool mempool fallback for small alloc.
- REQ-5: bio chaining: `bio_chain` links a child bio to parent's chain; refcount management correct.
- REQ-6: bio split: `bio_split` produces two bios (one with first N sectors, one with remainder); identical to upstream's bvec-iter advance + clone.
- REQ-7: bio merge: `blk_bio_segment_split` decides if two bios coalesce per the request_queue's `max_segment_size` + `max_segments`. Identical heuristics.
- REQ-8: User-buffer mapping (`bio_map_user_iov`, `blk_rq_map_user_iov`): user pages pinned via GUP (cross-ref `mm/virtual-memory.md`); IOV → bio_vec conversion identical.
- REQ-9: Higher-level helpers: `blkdev_issue_discard`, `blkdev_issue_zeroout`, `blkdev_issue_secure_erase`, `blkdev_issue_flush`, `blkdev_issue_zone_*`. Identical semantics.
- REQ-10: bvec_iter cursor manipulation: `bvec_iter_advance` + `bvec_iter_skip` + `__bio_for_each_segment` macros preserve iteration semantics.
- REQ-11: bio integrity (T10-PI; cross-ref `block/bio-integrity.md`): bio carries optional integrity payload; checked at endio.
- REQ-12: cgroup integration: bi_blkg back-pointer for cgroup-tracked bios; cross-ref `block/cgroup-throttle.md`.
- REQ-13: Layer-3 invariant harness: bio chain by sequence number; chain length matches bi_iter consumption (per `block/00-overview.md` mandatory L3).
- REQ-14: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct bio`, `struct bio_vec` byte-identical first-cache-line layout. (covers REQ-1, REQ-2)
- [ ] AC-2: `enum req_opf` values compile-time const-asserted to match upstream. (covers REQ-3)
- [ ] AC-3: A `bio_alloc` test (1M allocations) verifies per-pool fallback works under pressure. (covers REQ-4)
- [ ] AC-4: A `bio_chain` test: parent bio with 5 children completes only after all children endio. (covers REQ-5)
- [ ] AC-5: A `bio_split` test: split a 64-page bio at 16 pages; two resulting bios cover the original range without overlap. (covers REQ-6)
- [ ] AC-6: A coalesce test: two consecutive bios merge into one when device limits permit. (covers REQ-7)
- [ ] AC-7: A direct-I/O `pwrite` test correctly maps user pages to bio_vec; releases on endio. (covers REQ-8)
- [ ] AC-8: `blkdev_issue_discard` on a TRIM-capable device produces correct REQ_OP_DISCARD bio. (covers REQ-9)
- [ ] AC-9: `bvec_iter_advance` test on a multi-page bio iterates correctly. (covers REQ-10)
- [ ] AC-10: A T10-PI bio integrity test (cross-ref `block/bio-integrity.md`) checks integrity at endio. (covers REQ-11)
- [ ] AC-11: A cgroup-confined process's bios carry bi_blkg back-pointer; charging works. (covers REQ-12)
- [ ] AC-12: `make verify` passes mandatory L3 harness for bio chain invariants. (covers REQ-13)
- [ ] AC-13: Hardening section present and follows template. (covers REQ-14)

### Architecture

### Rust module organization

- `kernel::block::bio::Bio` — `struct bio` wrapper
- `kernel::block::bio::vec::BioVec` — bio_vec wrapper
- `kernel::block::bio::iter::BvecIter` — cursor type
- `kernel::block::bio::alloc` — alloc / chain / split / clone
- `kernel::block::bio::merge` — segment merge
- `kernel::block::bio::map` — user-buffer mapping
- `kernel::block::bio::lib` — high-level helpers (discard, zeroout, secure_erase, flush, zone_*)
- `kernel::block::bio::endio` — endio callback dispatch + chain refcount
- `kernel::block::bio::pool::BioSet` — per-driver bio mempool
- `kernel::block::bio::integrity` — T10-PI payload (cross-ref `block/bio-integrity.md`)

### Locking and concurrency

- **Per-bioset `bs->rescue_lock`**: per-mempool fallback alloc lock
- **`bi_remaining` (atomic refcount)**: bio chain completion
- **No global bio lock**: bios are per-context; chained via bi_next pointers
- **Per-bio bvec_iter**: each consumer has own cursor; immutable bio data shared

### Error handling

- `Err(ENOMEM)` — bio alloc failed
- `Err(EFAULT)` — bad userspace pointer in map
- `Err(EOPNOTSUPP)` — operation not supported by device (TRIM, secure-erase)
- `Err(EILSEQ)` — bad data integrity (T10-PI mismatch on rx)
- `Err(EREMOTEIO)` — driver-reported error

### Out of Scope

- Per-driver bio handling (cross-ref `drivers/00-overview.md`)
- T10-PI integrity payload detail (cross-ref `block/bio-integrity.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| bio core (alloc, split, chain, endio, free) | `block/bio.c` |
| Bio merging (consecutive bio coalesce) | `block/blk-merge.c` |
| User-data → bio mapping (e.g., direct I/O) | `block/blk-map.c` |
| Higher-level helpers (discard, zeroout, secure-discard) | `block/blk-lib.c` |
| Public types | `include/linux/bio.h`, `include/linux/blk_types.h` |

### compatibility contract

### `struct bio` layout

`include/linux/blk_types.h` defines `struct bio`. First-cache-line + commonly-accessed fields layout-equivalent:

- `bi_next` (chain pointer)
- `bi_bdev` (block device)
- `bi_opf` (operation + flags: REQ_OP_* + REQ_*)
- `bi_status` (completion status; blk_status_t)
- `bi_iter` (`struct bvec_iter`: bi_sector + bi_size + bi_idx + bi_bvec_done)
- `bi_inline_vecs[]` (small inline bio_vec array)
- `bi_max_vecs` (max bio_vec count)
- `bi_io_vec` (pointer to bio_vec array; either inline or external)
- `bi_vcnt` (current bio_vec count)
- `bi_size` (current data size)
- `bi_end_io` (completion callback)
- `bi_private` (callback context)
- `bi_pool` (mempool for free)
- `bi_integrity` (T10-PI integrity payload; cross-ref `block/bio-integrity.md` Tier-3 to be authored)
- `__bi_remaining` (refcount on bio chain; bi_endio called when reaches 0)
- `bi_disk`, `bi_partno` (legacy; now via bi_bdev)
- `bi_seg_front_size`, `bi_seg_back_size`
- `bi_dio_inflight` (direct-I/O tracking)
- `bi_bdev`, `bi_blkg`, `bi_issue` (cgroup integration)
- `bi_cookie` (debug)

Layout-equivalent so out-of-tree drivers + BPF programs accessing fields via macro work unchanged.

### `struct bio_vec` layout

`include/linux/bvec.h`:
- `bv_page` (struct page pointer)
- `bv_offset` (offset within page)
- `bv_len` (length within page)

Layout byte-identical.

### REQ_OP_* enumeration

`include/linux/blk_types.h`:
- `REQ_OP_READ=0`, `REQ_OP_WRITE=1`, `REQ_OP_FLUSH=2`, `REQ_OP_DISCARD=3`, `REQ_OP_SECURE_ERASE=5`, `REQ_OP_ZONE_APPEND=7`, `REQ_OP_WRITE_ZEROES=9`, `REQ_OP_ZONE_OPEN=10`, `REQ_OP_ZONE_CLOSE=11`, `REQ_OP_ZONE_FINISH=12`, `REQ_OP_ZONE_RESET=13`, `REQ_OP_ZONE_RESET_ALL=15`, `REQ_OP_DRV_IN=34`, `REQ_OP_DRV_OUT=35`, `REQ_OP_LAST=36`. Numeric byte-identical.

### REQ_* flags

`REQ_FAILFAST_DEV`, `REQ_FAILFAST_TRANSPORT`, `REQ_FAILFAST_DRIVER`, `REQ_SYNC`, `REQ_META`, `REQ_PRIO`, `REQ_NOMERGE`, `REQ_IDLE`, `REQ_INTEGRITY`, `REQ_FUA`, `REQ_PREFLUSH`, `REQ_RAHEAD`, `REQ_BACKGROUND`, `REQ_NOWAIT`, `REQ_CGROUP_PUNT`, `REQ_POLLED`, `REQ_ALLOC_CACHE`, `REQ_DRV`, `REQ_SWAP`, `REQ_NOUNMAP`. Numeric byte-identical.

### Bio chain endio semantics

When a bio chain completes: each child's bi_endio fires; final endio when chain refcount reaches 0. Identical.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| bio scatter-list construction | `kani::proofs::block::bio::sg_safety` |
| bvec_iter cursor advance | `kani::proofs::block::bio::iter_safety` |
| bio chain refcount + endio | `kani::proofs::block::bio::chain_endio_safety` |
| bio split (sector + bvec advance) | `kani::proofs::block::bio::split_safety` |
| bio merge (segment-size + max-segments check) | `kani::proofs::block::bio::merge_safety` |

### Layer 2: TLA+ models

- `models/block/bio_integrity_chain.tla` (mandatory per `block/00-overview.md` Layer 2; co-owned with `block/bio-integrity.md` Tier-3) — proves bio-integrity chain (data + integrity in lock-step) preserves ordering across split/merge.

### Layer 3: Kani harnesses for data-structure invariants (mandatory per `block/00-overview.md`)

| Data structure | Invariant | Harness |
|---|---|---|
| bio chain | bi_next chain is acyclic; chain length matches bi_iter consumption | `kani::proofs::block::bio::chain_invariants` |
| bvec_iter cursor | bi_size + bi_iter_done == initial bi_size; cursor never beyond bv_len | `kani::proofs::block::bio::cursor_invariants` |
| bio integrity payload | If bi_integrity != NULL, integrity payload size + format match upstream T10-PI | `kani::proofs::block::bio::integrity_invariants` |

### Layer 4: Functional correctness (opt-in)

- **bio_split correctness** via Verus — proves: splitting a bio at offset N produces two bios whose union (in correct order) equals the original.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | bi_remaining (chain refcount) uses `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | bio + bio_vec allocated via per-bioset mempool + slab; per-driver type-tagged | § Mandatory |
| **MEMORY_SANITIZE** | freed bio objects zeroed (especially relevant for bi_private callback context that may carry sensitive data) | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **UDEREF**: user pages pinned via GUP; bio_vec carries kernel-side struct page references (cross-ref `mm/virtual-memory.md`)
- **SIZE_OVERFLOW**: bi_size + bi_sector + bvec offset arithmetic uses checked operators
- **CONSTIFY**: per-driver bio operations table provided by drivers is `static const`

### Row-2 / GR-RBAC integration

bio is below the LSM-hook layer; LSM hooks fire at fs/syscall level.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

