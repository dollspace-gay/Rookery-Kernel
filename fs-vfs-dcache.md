---
title: "Tier-3: fs/vfs/dcache — directory entry cache"
tags: ["design-doc", "tier-3", "fs"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the directory entry cache (dcache). Owns `struct dentry` lifecycle, the dcache hash table, the per-superblock dentry LRU list, the RCU-walk fast path for path resolution (`__lookup_fast`, `walk_component`), the parent → child reference graph, and the dcache shrinker that releases dentries under memory pressure.

This is **the fourth and final MANDATORY Layer-3-Kani-harness subsystem** per `00-overview.md` D4. Dcache invariants — every dentry's parent is reachable from a fs root, every parent's child list contains the dentry, LRU list contains exactly the refcount-zero dentries — are mechanically verified.

The dcache is one of the most concurrency-sensitive structures in the kernel: every path resolution touches it; many access patterns are RCU-only (no lock held during traversal).

### Requirements

- REQ-1: Dentry struct (`struct dentry`) layout-equivalent to upstream for first-cache-line + commonly-accessed-via-macro fields. Fields: `d_lock`, `d_iname`, `d_name`, `d_parent`, `d_inode`, `d_subdirs`, `d_child`, `d_alias`, `d_lockref` (refcount), `d_op`, `d_sb`, `d_flags`, `d_seq`, `d_lru`, `d_hash`.
- REQ-2: Dcache hash table identical: per-bucket hlist; hash function `full_name_hash`; bucket count power-of-two per upstream.
- REQ-3: `dput(d)` semantics: refcount decrement; if 0, add to per-sb LRU; if also DCACHE_DENTRY_KILLED, free immediately.
- REQ-4: `d_alloc(parent, name)` semantics identical: reference grandparent for `d_parent`, copy name, link into parent's d_subdirs.
- REQ-5: `d_invalidate(d)` cuts the subtree out of the dcache.
- REQ-6: RCU-walk fast-path: `walk_component` accepts the dentry's `d_seq`; on each step, `read_seqcount_begin` + recheck. Mismatch → restart in ref-walk mode.
- REQ-7: `d_path(path, buf, len)` — formats a (vfsmount, dentry) pair as a / -separated path; respects chroot + bind mounts; output byte-identical to upstream.
- REQ-8: Negative dentries (lookup miss → cached negative) preserve upstream behavior; counted in `/proc/sys/fs/dentry-state` `nr_negative` column.
- REQ-9: Dcache LRU shrinker: registered via `register_shrinker`; under memory pressure, frees per-sb LRU entries; respects DCACHE_REFERENCED bit.
- REQ-10: idmapped-mount integration: `vfsuid_t` / `vfsgid_t` typed handles vs. raw uid/gid; cross-ref `fs/vfs/mount.md`.
- REQ-11: Dcache invariant Layer-3 Kani harnesses are MANDATORY (per `00-overview.md` D4 / `fs/00-overview.md` REQ-16).
- REQ-12: TLA+ model `models/fs/dcache_rcu_walk.tla` proves RCU-walk's "no torn read" invariant under racing rename.
- REQ-13: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `pahole struct dentry` first-cache-line layout byte-identical vs. upstream. (covers REQ-1)
- [ ] AC-2: `cat /proc/sys/fs/dentry-state` format-identical. (covers REQ-2 partly)
- [ ] AC-3: A `dput`-cycle test creates 1M dentries, drops references; final dcache state matches upstream's (size, free-list contents). (covers REQ-3, REQ-4)
- [ ] AC-4: `d_invalidate` test on a partially-populated dcache subtree leaves the dcache consistent (no orphans). (covers REQ-5)
- [ ] AC-5: A 32-CPU concurrent `path_lookup` benchmark produces zero RCU-walk torn reads (verified by tracepoints + the TLA+ model). (covers REQ-6, REQ-12)
- [ ] AC-6: `d_path` golden-output test for a curated set of paths (chroot, bind-mount, deleted-while-open, etc.) byte-identical vs. upstream. (covers REQ-7)
- [ ] AC-7: Negative-dentry stress test: 100K bogus lookups followed by `cat /proc/sys/fs/dentry-state` shows `nr_negative` = 100K. (covers REQ-8)
- [ ] AC-8: Memory-pressure test (`echo 2 > /proc/sys/vm/drop_caches`) reduces dentry LRU; `nr_unused` falls; per-sb LRU consistency invariant remains. (covers REQ-9)
- [ ] AC-9: An idmapped-mount test creates a mount with a non-trivial uid_map; lookups in the mount return paths with correctly-translated `i_uid` per upstream. (covers REQ-10)
- [ ] AC-10: `make verify` passes `kani::proofs::fs::dcache::*` Kani harnesses (mandatory L3). (covers REQ-11)
- [ ] AC-11: `make tla` passes `models/fs/dcache_rcu_walk.tla`. (covers REQ-12)
- [ ] AC-12: Hardening section present and follows template. (covers REQ-13)

### Architecture

### Rust module organization

- `kernel::fs::dcache::Dentry` — `struct dentry` wrapper
- `kernel::fs::dcache::DcacheHash` — per-superblock dcache hash table
- `kernel::fs::dcache::Lru` — per-sb LRU list
- `kernel::fs::dcache::rcu_walk` — RCU-walk fast path
- `kernel::fs::dcache::ref_walk` — ref-walk fallback
- `kernel::fs::dcache::shrinker` — dcache memory-pressure shrinker
- `kernel::fs::dcache::d_path` — path-formatting helper
- `kernel::fs::namei::Namei` — name-lookup driver (cross-ref `fs/vfs/path-resolution.md`)

### Locking and concurrency

- **`d_lock`** (per-dentry spinlock): held for `d_subdirs` + `d_alias` + flag updates
- **`d_seq`** (per-dentry sequence counter): incremented on rename + invalidate; read-side: RCU walkers
- **`hash_bucket->hlock`** (per-bucket spinlock — actually fine-grained): held during hash insert/remove
- **`sb->s_inode_list_lock`** (per-superblock spinlock): protects sb's inode list
- **`s_dentry_lru` lock**: per-LRU-list spinlock; protects LRU manipulation
- **dput lockref**: `lockref_*` family — combined refcount + spinlock (fast path: cmpxchg-only)

TLA+ model `models/fs/dcache_rcu_walk.tla` proves the seqcount-based read-side under racing writers.

### Error handling

- `Err(ENOMEM)` — dentry alloc failed
- `Err(ESTALE)` — dentry invalidated mid-walk; caller should retry from start
- `Err(EAGAIN)` — RCU walk failed; switch to ref-walk
- `Err(ENOENT)` — name not found (returns negative dentry)

### Out of Scope

- Path resolution itself (cross-ref `fs/vfs/path-resolution.md`)
- Inode cache (cross-ref `fs/vfs/inode.md`)
- File-table (cross-ref `fs/vfs/file-table.md`)
- Per-FS-specific dcache extensions (cross-ref individual fs/ Tier-3 docs)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Dcache core: dentry lifecycle + hash + lookup | `fs/dcache.c` |
| Path-formatting helpers (d_path family) | `fs/d_path.c` |
| Path resolution (RCU-walk / ref-walk) | `fs/namei.c` (cross-ref `fs/vfs/path-resolution.md`) |
| Public API | `include/linux/dcache.h` |
| LRU list helpers | `include/linux/list_lru.h`, `mm/list_lru.c` (cross-ref `lib/data-structures.md`) |

### compatibility contract

### `/proc` surfaces

- `/proc/sys/fs/dentry-state` — format: `<nr_dentry> <nr_unused> <age_limit> <dummy> <dummy> <nr_negative>`. Format-identical.

### `dput` / `dget` semantics

Reference-counted dentries; `dput(d)` decrements + adds to LRU when refcount=0; `dget(d)` increments. Userspace doesn't observe directly, but kernel-internal callers (every fs operation) depend on the semantics.

### dentry hash function

`include/linux/dcache.h` defines `full_name_hash` for d_hash + dcache hash table indexing. Bit-identical hash function so `/proc/<pid>/maps` filename hashes consistent with upstream's introspection tools (e.g., crash, drgn).

### RCU-walk path resolution

`namei.c::walk_component` performs path-component-by-component lookup under RCU read-side; on conflict (dentry being concurrently invalidated, mount-point traversal, symlink), falls back to ref-walk. Behavior must be byte-identical so applications observe POSIX-defined path semantics.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Dentry alloc + parent linking | `kani::proofs::fs::dcache::alloc_safety` |
| `dput` lockref cmpxchg fast path | `kani::proofs::fs::dcache::dput_safety` |
| RCU-walk seqcount read | `kani::proofs::fs::dcache::rcu_walk_seq_safety` |
| Hash insertion + collision-list link | `kani::proofs::fs::dcache::hash_link_safety` |
| LRU add + remove | `kani::proofs::fs::dcache::lru_link_safety` |

### Layer 2: TLA+ models

- `models/fs/dcache_rcu_walk.tla` (mandatory per `fs/00-overview.md` Layer 2) — proves RCU-walk's "no torn read" invariant: even when a parent dentry is renamed during the walk, the walker either succeeds with a coherent path or fails to ENOENT/EAGAIN — never reads garbage. Safety + (where meaningful) liveness (eventual progress under fairness assumption).

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY per `00-overview.md` D4)

| Data structure | Invariant | Harness |
|---|---|---|
| Dcache hash | "Every dentry's parent is reachable from a fs root, and the parent's d_subdirs list contains the dentry" | `kani::proofs::fs::dcache::tree_invariants` |
| Dcache LRU | "Every dentry on the LRU list has refcount==0 and the corresponding LRU flag set; conversely, every refcount-0 non-killed dentry IS on the LRU list" | `kani::proofs::fs::dcache::lru_invariants` |
| Hash buckets | "Each dentry appears in exactly one hash bucket, indexed by `full_name_hash(d_name)` modulo bucket count" | `kani::proofs::fs::dcache::hash_invariants` |
| Negative dentries | "A negative dentry has d_inode == NULL + DCACHE_FALLTHRU not set" | `kani::proofs::fs::dcache::negative_invariants` |
| Parent-child consistency | "For every dentry d with parent p: p->d_subdirs contains d, and d->d_parent == p" (the harness exhaustively checks both directions) | `kani::proofs::fs::dcache::parent_child_invariants` |

### Layer 4: Functional correctness (opt-in)

- **`d_path` correctness** via Creusot — proves: for any (vfsmount, dentry) pair, the formatted path matches the path that walking from the dentry to the fs root would produce.
- **`full_name_hash` distribution** via Creusot — proves: hash distribution is uniform-ish (not formal correctness but a soft guarantee that bucket fill stays bounded). Lower priority.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** (saturating) | `dentry->d_lockref` is a `Lockref` (combined spinlock + saturating Refcount) | § Mandatory |
| **AUTOSLAB** | Dentries allocated via `KmemCache::<Dentry>::new()` per-type cache; per-sb LRU keys per-type tagging | § Mandatory |
| **MEMORY_SANITIZE** | Freed dentries zeroed unless cache marked SLAB_NO_ZERO_ON_FREE | § Default-on configurable off |

### Row-1 features consumed by this component

- **UDEREF**: dcache never accepts user pointers directly; path-resolution callers convert user paths into kernel-side `qstr` structures via `getname` (cross-ref `lib/usercopy.md`)
- **SIZE_OVERFLOW**: hash-bucket index arithmetic uses checked operators
- **CONSTIFY**: dentry-operations (`d_op`) vtables are `static const`
- **KERNEXEC**: dcache code's text is RX/RO

### Row-2 / GR-RBAC integration

The dcache itself doesn't enforce LSM hooks (it's a cache). Path resolution invokes `security_inode_*` hooks at each component (`fs/vfs/path-resolution.md` owns the dispatch). GR-RBAC's TPE policy can deny path traversal at the LSM hook layer; dcache merely provides the hash + LRU substrate.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

