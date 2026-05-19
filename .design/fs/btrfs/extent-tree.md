# Tier-3: fs/btrfs/extent-tree.c — Btrfs extent allocator + reference counting

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/btrfs/super.md
upstream-paths:
  - fs/btrfs/extent-tree.c (~6921 lines)
  - fs/btrfs/free-space-cache.c (per-block-group cache)
  - fs/btrfs/block-group.c
  - include/uapi/linux/btrfs_tree.h (EXTENT_ITEM, METADATA_ITEM, EXTENT_DATA_REF, etc.)
-->

## Summary

Btrfs extent-tree (`fs/btrfs/extent-tree.c`) implements per-block-group extent allocator + per-extent ref-counting. Per-extent represented by EXTENT_ITEM (data) or METADATA_ITEM (tree-block). Per-extent has back-references via TREE_BLOCK_REF / EXTENT_DATA_REF / SHARED_BLOCK_REF / SHARED_DATA_REF. Per-snapshot/clone shares extents with refcount++. Per-allocator uses per-block-group free-space-cache (in-memory rbtree) backed by FREE_SPACE_INFO + FREE_SPACE_BITMAP / FREE_SPACE_EXTENT items. Per-allocator picks profile-matched (DUP/RAID0/RAID1/RAID10/RAID5/RAID6/SINGLE) block-group and per-cluster contiguous range. Critical for: btrfs space management, snapshot CoW, balance/resize.

This Tier-3 covers `fs/btrfs/extent-tree.c` (~6921 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `btrfs_reserve_extent()` | per-allocate | `Btrfs::reserve_extent` |
| `find_free_extent()` | per-search | `Btrfs::find_free_extent` |
| `btrfs_alloc_tree_block()` | per-metadata-block | `Btrfs::alloc_tree_block` |
| `btrfs_alloc_data_chunk_ondemand()` | per-data-chunk | `Btrfs::alloc_data_chunk_ondemand` |
| `btrfs_inc_extent_ref()` | per-ref++ | `Btrfs::inc_extent_ref` |
| `btrfs_free_extent()` | per-ref-- | `Btrfs::free_extent` |
| `btrfs_pin_extent()` | per-pin (txn-bound) | `Btrfs::pin_extent` |
| `btrfs_finish_extent_commit()` | per-txn-commit free | `Btrfs::finish_extent_commit` |
| `btrfs_lookup_extent_info()` | per-extent meta lookup | `Btrfs::lookup_extent_info` |
| `btrfs_run_delayed_refs()` | per-delayed-ref drain | `Btrfs::run_delayed_refs` |
| `add_delayed_data_ref()` / `add_delayed_tree_ref()` | per-ref defer | `Btrfs::add_delayed_*_ref` |
| `select_delayed_ref()` | per-pick | shared |
| `__run_one_delayed_ref()` | per-ref apply | shared |
| `btrfs_block_group_cache_done()` | per-cache-load | shared |
| `add_to_free_space_tree()` / `remove_from_free_space_tree()` | per-FS-tree | shared |
| `btrfs_create_pending_block_groups()` | per-new-bg | `Btrfs::create_pending_block_groups` |
| `btrfs_make_block_group()` | per-bg create | `Btrfs::make_block_group` |
| `btrfs_remove_block_group()` | per-bg destroy | `Btrfs::remove_block_group` |
| `btrfs_force_chunk_alloc()` | per-force-alloc | `Btrfs::force_chunk_alloc` |
| `btrfs_relocate_block_group()` | per-balance/resize | `Btrfs::relocate_block_group` |

## Compatibility contract

REQ-1: EXTENT_ITEM (key.type = 0xA8):
- objectid = bytenr (start of extent in dev-byte-addr).
- offset = num_bytes.
- value: refs (u64), generation (u64), flags (u64: DATA / TREE_BLOCK), tree_block_info (if TREE_BLOCK).

REQ-2: METADATA_ITEM (key.type = 0xA9):
- Like EXTENT_ITEM but with key.offset = level (skinny metadata).
- Per-skinny-metadata feature.

REQ-3: Backrefs:
- TREE_BLOCK_REF (0xB0): per-tree-id ownership.
- SHARED_BLOCK_REF (0xB2): per-snapshot back-pointer.
- EXTENT_DATA_REF (0xB4): per-data-extent back-pointer (root, owner, offset).
- SHARED_DATA_REF (0xB6): per-snapshot data-back-pointer.

REQ-4: btrfs_reserve_extent(num_bytes, flags) -> (bytenr, alloc_bytes):
- Per-flush-policy: BTRFS_RESERVE_NO_FLUSH / FLUSH_LIMIT / FLUSH_ALL / FLUSH_ALL_STEAL.
- Pick block-group matching flags (data/metadata/system) + profile.
- find_free_extent within block-group(s).
- Update free-space-cache.

REQ-5: find_free_extent loop:
- Per-search-list: avg/empty-bitmap/empty-cluster/cached.
- Per-cluster: per-bg cluster cache.
- Per-bitmap: scan free-bitmap.
- Per-extent: scan rbtree.
- If none: btrfs_chunk_alloc to grow fs.

REQ-6: Per-delayed-ref:
- All ref++/-- deferred via `btrfs_delayed_ref_root`.
- Per-add_delayed_*_ref enqueues into per-head rb-tree.
- Per-run_delayed_refs drains under transaction.

REQ-7: btrfs_inc_extent_ref:
- Add delayed ref ACTION_ADD.
- On commit: refs++, write back-ref.

REQ-8: btrfs_free_extent:
- Add delayed ref ACTION_DROP.
- On commit: refs--; if refs == 0: remove EXTENT_ITEM + add to free-space + return chunk if empty.

REQ-9: btrfs_alloc_tree_block:
- For metadata: alloc 4K extent (sector-size on most fs).
- Set generation, owner_root, level.
- Call btrfs_inc_extent_ref.

REQ-10: btrfs_pin_extent:
- During commit: extents being freed pinned to prevent re-alloc same-txn.
- Per-EXTENT_DIRTY in pinned_extents io-tree.

REQ-11: btrfs_finish_extent_commit:
- Drain pinned_extents → unpin → return to free-space.

REQ-12: Per-block-group:
- objectid = bg start; offset = bg length.
- BLOCK_GROUP_ITEM (0xC0).
- flags = (DATA|METADATA|SYSTEM) | profile (RAID0/1/10/5/6/DUP/SINGLE).

REQ-13: Free-space tree (FST) (mount option `space_cache=v2`):
- FREE_SPACE_INFO per-bg.
- FREE_SPACE_EXTENT (per-large-region) or FREE_SPACE_BITMAP.

REQ-14: Per-fragmentation:
- bg may be fragmented; extent-cluster refines for higher-perf.

REQ-15: Per-RAID profile:
- RAID0: stripe across N devs.
- RAID1: 2-way mirror across 2 devs.
- RAID10: 2-way mirror striped.
- RAID5: striped + 1 parity.
- RAID6: striped + 2 parity.
- DUP: 2 copies same dev.
- SINGLE: no redundancy.

REQ-16: Per-balance:
- Move extents from one bg to another; may relocate.

REQ-17: Per-NOCOW path (preallocated/nodatacow):
- Skip CoW; reuse existing extent.

REQ-18: Per-quota (qgroup) hooks:
- Per-extent alloc/free informs qgroup accounting.

## Acceptance Criteria

- [ ] AC-1: btrfs_reserve_extent(4K, DATA): returns valid bytenr from data-bg.
- [ ] AC-2: btrfs_alloc_tree_block: metadata-bg, 4K, with TREE_BLOCK_REF.
- [ ] AC-3: btrfs_inc_extent_ref(bytenr): EXTENT_ITEM.refs++ on commit.
- [ ] AC-4: btrfs_free_extent on last-ref: EXTENT_ITEM removed + bytes returned to free-space.
- [ ] AC-5: snapshot creates: SHARED_BLOCK_REF / SHARED_DATA_REF for shared extents.
- [ ] AC-6: balance: moves extents to new bg; back-refs updated.
- [ ] AC-7: ENOSPC: -ENOSPC after exhausting all bgs + chunks.
- [ ] AC-8: btrfs_pin_extent: pinned during txn.
- [ ] AC-9: btrfs_finish_extent_commit: pinned → unpinned → free.
- [ ] AC-10: btrfs check: post-balance ref-counts consistent.
- [ ] AC-11: btrfs_run_delayed_refs: empties per-head queue.
- [ ] AC-12: skinny metadata: METADATA_ITEM.key.offset = level (not num_bytes).

## Architecture

```
struct Btrfs;
impl Btrfs {
  fn reserve_extent(fs_info: &FsInfo, num_bytes: u64, min_alloc: u64,
                    empty_size: u64, flags: u64) -> Result<KeyAndOffset>;
  fn find_free_extent(fs_info: &FsInfo, ...) -> Result<KeyAndOffset>;
  fn alloc_tree_block(trans: &Trans, root: &Root, parent: u64,
                      level: i32, hint: u64, root_objectid: u64,
                      key: &DiskKey, generation: u64) -> Result<*ExtentBuffer>;
  fn inc_extent_ref(trans: &Trans, root: &Root, ref_: ExtentRef) -> Result<()>;
  fn free_extent(trans: &Trans, root: &Root, ref_: ExtentRef) -> Result<()>;
  fn pin_extent(trans: &Trans, bytenr: u64, num_bytes: u64,
                reserved: bool) -> Result<()>;
  fn finish_extent_commit(trans: &Trans) -> Result<()>;
  fn run_delayed_refs(trans: &Trans, count: u64) -> Result<()>;
}
```

`Btrfs::reserve_extent(fs_info, num_bytes, ...)`:
1. /* Determine flags + profile */
2. data_flags = btrfs_get_alloc_profile(fs_info, BTRFS_BLOCK_GROUP_DATA).
3. /* Loop until alloc-or-grow */
4. for ffe_ctl in find_free_extent_ctl (cluster → empty_cluster → bitmap → extent):
   - bg = pick_block_group(data_flags, ffe_ctl).
   - if !bg: break.
   - found = ffe_ctl.search(bg, num_bytes).
   - if found: return (bytenr, alloc_bytes).
5. /* No space → grow via chunk_alloc */
6. ret = btrfs_chunk_alloc(trans, fs_info, data_flags, FLUSH_LIMIT).
7. if ret == -ENOSPC: -ENOSPC.
8. retry from step 4.

`Btrfs::find_free_extent(fs_info, num_bytes, hint, flags)`:
1. /* Per-bg loop ordered by hint, then space_info.bg_list */
2. for bg in space_info.bgs:
   - if !bg.cached: btrfs_cache_block_group(bg, true).
   - cluster = &bg.free_space_cluster.
   - if cluster.bytes >= num_bytes: alloc from cluster.
   - else: bg.free_space_ctl.find_free_extent(num_bytes).
3. if found: return.

`Btrfs::alloc_tree_block(trans, root, parent, level, hint, root_objectid, key, generation)`:
1. /* Reserve metadata */
2. block_size = fs_info.nodesize.
3. ins = Btrfs::reserve_extent(fs_info, block_size, ..., METADATA).
4. eb = btrfs_init_new_buffer(trans, root, ins.objectid, level).
5. eb.write_header(generation, owner = root_objectid, level).
6. /* Add ref */
7. ref = TreeBlockRef { root = root_objectid, level }.
8. Btrfs::inc_extent_ref(trans, root, ref).
9. return eb.

`Btrfs::inc_extent_ref(trans, root, ref) -> Result<()>`:
1. /* Add delayed ref ACTION_ADD */
2. delayed = btrfs_delayed_ref_create(ACTION_ADD, ref).
3. btrfs_add_delayed_data_ref / _tree_ref(trans, delayed).
4. /* Drain when transaction commits or threshold */

`Btrfs::free_extent(trans, root, ref)`:
1. /* Add delayed ref ACTION_DROP */
2. delayed = btrfs_delayed_ref_create(ACTION_DROP, ref).
3. btrfs_add_delayed_*_ref(trans, delayed).

`Btrfs::run_delayed_refs(trans, count)`:
1. for i = 0; i < count:
   - head = btrfs_select_ref_head(trans).
   - if !head: break.
   - locked = btrfs_delayed_ref_lock(head).
   - if !locked: continue.
   - while ref = btrfs_select_delayed_ref(head):
     - __run_one_delayed_ref(trans, ref) (per-action).
2. /* per-ACTION_ADD: insert/update EXTENT_ITEM */
3. /* per-ACTION_DROP: dec EXTENT_ITEM.refs; if 0 → remove + free-space + qgroup */

`Btrfs::pin_extent(trans, bytenr, num_bytes, reserved)`:
1. set_extent_dirty(&trans.fs_info.pinned_extents, bytenr, bytenr + num_bytes - 1).

`Btrfs::finish_extent_commit(trans)`:
1. /* Iter pinned_extents io_tree */
2. for (start, end) in trans.fs_info.pinned_extents:
   - bg = btrfs_lookup_block_group(start).
   - btrfs_add_free_space(bg, start, end - start + 1).
3. clear_extent_dirty(pinned_extents).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bytenr_aligned_to_sectorsize` | INVARIANT | per-extent: bytenr ∈ aligned-to-sectorsize. |
| `extent_within_chunk` | INVARIANT | per-extent ⊆ chunk-range. |
| `refs_decrement_to_zero_then_free` | INVARIANT | per-extent: refs == 0 ⟹ EXTENT_ITEM removed. |
| `delayed_ref_drained_in_commit` | INVARIANT | per-trans commit: delayed_refs.head_ref == 0. |
| `pinned_freed_in_finish_commit` | INVARIANT | per-finish_extent_commit: pinned cleared. |
| `bg_cached_before_alloc` | INVARIANT | per-find_free_extent: bg.cached at call. |

### Layer 2: TLA+

`fs/btrfs/extent-tree.tla`:
- Per-bg list + per-extent rbtree + per-delayed-ref queue + per-pinned-extents.
- Per-reserve_extent + per-free_extent + per-snapshot-share + per-balance.
- Properties:
  - `safety_no_double_alloc` — per-bytenr: at most 1 EXTENT_ITEM with refs > 0 + (refs+pending) consistent.
  - `safety_pinned_not_realloced` — per-pinned-extent during txn: not returned by find_free_extent.
  - `safety_refcount_consistent` — per-EXTENT_ITEM.refs equals number of back-refs.
  - `liveness_per_alloc_eventually_succeeds_or_enospc` — per-reserve_extent terminates with bytenr or -ENOSPC.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Btrfs::reserve_extent` post: returns (bytenr, alloc_bytes) ∨ -ENOSPC | `reserve_extent` |
| `Btrfs::alloc_tree_block` post: TreeBlockRef installed; refs == 1 | `alloc_tree_block` |
| `Btrfs::inc_extent_ref` post: delayed ACTION_ADD enqueued | `inc_extent_ref` |
| `Btrfs::free_extent` post: delayed ACTION_DROP enqueued | `free_extent` |
| `Btrfs::run_delayed_refs` post: refs persist EXTENT_ITEM consistent | `run_delayed_refs` |
| `Btrfs::pin_extent` post: pinned_extents marked dirty | `pin_extent` |

### Layer 4: Verus/Creusot functional

`Per-extent alloc → per-bg cluster/bitmap pick → reserve + delayed_ref ACTION_ADD; per-free → delayed_ref ACTION_DROP → pin → on-commit return to free-space` semantic equivalence: per-Documentation/filesystems/btrfs.rst + per-btrfs-progs verification.

## Hardening

(Inherits row-1 features from `fs/btrfs/super.md` § Hardening.)

Extent-tree reinforcement:

- **Per-bytenr alignment checked** — defense against per-mis-aligned alloc.
- **Per-extent-range within fs-bytenr bounds** — defense against per-OOR extent.
- **Per-refs u64 saturating** — defense against per-refcount-overflow.
- **Per-back-ref entry-count == EXTENT_ITEM.refs** — defense against per-refcount drift.
- **Per-delayed-ref draining commit-mandatory** — defense against per-stale ref-on-commit.
- **Per-pinned-extents not-realloced same-txn** — defense against per-extent-reuse-before-commit.
- **Per-bg cache-completed checked** — defense against per-stale-cache alloc.
- **Per-profile flags strict (no mixed RAID/SINGLE collisions)** — defense against per-profile-mismatch alloc.
- **Per-tree-mod-log captures rb-tree ops** — defense against per-snapshot-while-mod inconsistency.
- **Per-skinny-metadata flag-gated** — defense against per-mis-format extent-item.
- **Per-quota hook on alloc/free** — defense against per-qgroup undercount.
- **Per-balance back-ref-rewrite atomic** — defense against per-balance-crash inconsistency.
- **Per-enospc-flush escalation policy** — defense against per-DOS via early-ENOSPC.

## Grsecurity/PaX-style Reinforcement

Beyond the upstream hardening above, Rookery layers the following grsec/PaX-style controls onto `fs/btrfs/extent-tree.c`:

- **PAX_USERCOPY** — bounds-checks any extent-tree-derived data exposed via ioctls (e.g. `BTRFS_IOC_TREE_SEARCH*`) so a malformed search cannot drive a slab overrun.
- **PAX_KERNEXEC** — keeps the extent-tree callback tables in read-only memory so a memory bug cannot rewrite delayed-ref or back-ref processors.
- **PAX_RANDKSTACK** — randomizes kernel-stack offset on each transaction-start so an attacker cannot deterministically probe the long extent-alloc path.
- **PAX_REFCOUNT** — wraps `btrfs_delayed_ref_node.refs` and `btrfs_extent_item.refs` so a malicious image driving extreme ref churn cannot wrap to zero.
- **PAX_MEMORY_SANITIZE** — zero-on-free for delayed-ref nodes and extent-item scratch buffers so leftover back-ref keys cannot be recovered from the freelist.
- **PAX_UDEREF** — enforces user/kernel pointer separation across the extent-tree-search ioctl boundary.
- **PAX_RAP / kCFI** — forward-edge CFI on the delayed-ref `process` callback so a corrupted node cannot redirect ref processing to a userspace gadget.
- **GRKERNSEC_HIDESYM** — strips kernel pointers from extent-tree warning printks so a crafted image cannot leak kASLR offsets via dmesg.
- **GRKERNSEC_DMESG** — gates dmesg on `CAP_SYSLOG` so extent-tree corruption diagnostics do not leak pointer material to unprivileged users.
- **Delayed-ref PAX_REFCOUNT** — `btrfs_delayed_ref_node.refs` (incremented at every add-ref/drop-ref) is wrapped with the hardened-refcount layer so a malicious image driving extreme delayed-ref churn cannot wrap and produce an early-free UAF on a delayed-ref node.
- **EXTENT_ITEM signature** — `btrfs_extent_item` decode validates the type/refs/generation triple against the BG type at every read, treating a mismatched signature as a hard mount-time abort rather than a recoverable warning, so a crafted on-disk extent cannot smuggle a confused-deputy refcount drift.
- **GRKERNSEC_HIDESYM on back-ref-count drift warnings** — `__btrfs_run_delayed_refs` warning paths sanitize embedded pointers so a crafted back-ref attack cannot extract delayed-ref or rbnode addresses via dmesg.
- **CAP_SYS_ADMIN on extent-tree search ioctls** — `BTRFS_IOC_TREE_SEARCH*` is restricted to privileged callers so unprivileged users cannot enumerate the on-disk extent layout.

Rationale: extent-tree refcount drift is the dominant root cause of btrfs on-disk corruption; layered refcount hardening + EXTENT_ITEM signature validation closes the gap between upstream "best-effort" balance/quota checks and a hardened-by-construction allocator.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/btrfs/free-space-cache.c (covered separately if expanded)
- fs/btrfs/free-space-tree.c (covered separately if expanded)
- fs/btrfs/block-group.c (covered separately if expanded)
- fs/btrfs/relocation.c balance (covered separately if expanded)
- fs/btrfs/qgroup.c (covered separately if expanded)
- fs/btrfs/transaction.c (covered separately if expanded)
- Implementation code
