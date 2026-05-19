---
title: "Tier-3: fs/ext4/mballoc.c — ext4 multi-block allocator"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

mballoc is ext4's data-block allocator. It serves block-allocation requests at the granularity of *clusters* (bigalloc) or *blocks* (default), with two parallel mechanisms: per-inode and per-locality-group **preallocation** (PA) to reduce fragmentation and bitmap thrash, and a **buddy bitmap** kept in-memory per block group to satisfy non-PA-fittable requests. Per-`ext4_mb_new_blocks` is the entry point invoked from `ext4_ext_map_blocks` (and the legacy ind-block path); it returns a physical block number after either consuming preallocation space or running the regular allocator pipeline. Per-`ext4_allocation_request` carries the desired logical block, length, and goal pblk plus `EXT4_MB_HINT_*` flags. Per-`ext4_allocation_context` is the in-flight state: original/goal/best/found extents (`ac_o_ex`, `ac_g_ex`, `ac_b_ex`, `ac_f_ex`), criteria (`ac_criteria`), prefetch counters, and PA hooks (`ac_pa`, `ac_lg`). Per-`ext4_buddy` is the per-group buddy-bitmap handle (`bd_buddy`, `bd_bitmap`, `bd_group`, `bd_info`). Per-`ext4_prealloc_space` represents a reserved PA window (either per-inode rbtree-indexed by `pa_lstart` or per-locality-group on a `lg_prealloc_list[PREALLOC_TB_SIZE]`). Per-CPU `ext4_locality_group` provides stream-allocation locality. Per-`mb_optimize_scan` mount option toggles XArray-backed group searches by largest-free-order or avg-fragment-order, replacing the linear O(N) walk. Allocation proceeds across **5 criteria** (`CR_POWER2_ALIGNED → CR_GOAL_LEN_FAST → CR_BEST_AVAIL_LEN → CR_GOAL_LEN_SLOW → CR_ANY_FREE`), falling back when a phase finds nothing. Per-`MBALLOC_USE_RESERVED` / `MBALLOC_DELALLOC_RESERVED` / `MB_USE_ROOT_BLOCKS` modulate quota and reserved-block-pool behaviour. Per-`ext4_free_blocks` is the symmetric free path, queueing a `ext4_free_data` extent into a per-transaction rbtree for jbd2-commit-time release. Critical for: write/append throughput, defrag resistance, ENOSPC behaviour, bigalloc clustering.

This Tier-3 covers `fs/ext4/mballoc.c` (~7283 lines).

### Acceptance Criteria

- [ ] AC-1: ext4_mb_new_blocks(non-replay, ar.len=N) returns a physical block; ar.len ≤ N on partial allocation.
- [ ] AC-2: PA hit on second consecutive append-write within same window: use_inode_pa called; new bitmap I/O avoided.
- [ ] AC-3: Small (< MB_DEFAULT_STREAM_THRESHOLD blocks) request takes LG path (use_group_pa or new_group_pa).
- [ ] AC-4: Power-of-2 request (≥ MB_DEFAULT_ORDER2_REQS log2) starts at CR_POWER2_ALIGNED; non-PO2 starts at CR_GOAL_LEN_FAST.
- [ ] AC-5: ext4_mb_regular_allocator falls through to CR_ANY_FREE only after CR_GOAL_LEN_SLOW exhausts; s_mb_lost_chunks incremented on best-found loss.
- [ ] AC-6: mb_optimize_scan=1: scan_p2_aligned uses XArray walk by largest_free_order; mb_optimize_scan=0: linear group walk.
- [ ] AC-7: ext4_mb_mark_diskspace_used updates bitmap + bg_free_blocks + checksums; on jbd2 abort: change rolled back.
- [ ] AC-8: ext4_free_blocks during open txn enqueues ext4_free_data; only released after jbd2 commit callback.
- [ ] AC-9: ext4_discard_preallocations(inode) frees all unused leftover PA blocks at file close / unlink.
- [ ] AC-10: ENOSPC: when discard_preallocations_should_retry fires, allocator restarts up to 3 times.
- [ ] AC-11: EXT4_MB_USE_RESERVED bypasses ext4_claim_free_clusters quota check; succeeds when only reserved pool remains.
- [ ] AC-12: FITRIM ioctl with minlen=M: only contiguous runs ≥ M discarded; smaller runs skipped.
- [ ] AC-13: ext4_mb_init / _release leave no GroupInfo, no LG, no PA leaks at umount.
- [ ] AC-14: Stream allocator: 16 sequential writes from same inode pack into adjacent groups via STREAM_ALLOC + s_mb_last_groups[hash].
- [ ] AC-15: ext4_mb_normalize_request preserves pa_pstart non-overlap invariant across concurrent inode allocs (asserted by ext4_mb_pa_assert_overlap).

### Architecture

```
struct AllocReq {                  // ext4_allocation_request
  inode: *Inode,
  len: u32,
  logical: ext4_lblk_t,
  lleft: ext4_lblk_t,
  lright: ext4_lblk_t,
  goal: ext4_fsblk_t,
  pleft: ext4_fsblk_t,
  pright: ext4_fsblk_t,
  flags: u32,                      // EXT4_MB_HINT_* | EXT4_MB_*
}

struct FreeExtent {                // ext4_free_extent
  fe_logical: ext4_lblk_t,
  fe_start: ext4_grpblk_t,         // cluster units, within group
  fe_group: ext4_group_t,
  fe_len: ext4_grpblk_t,           // cluster units
}

struct AllocCtx {                  // ext4_allocation_context
  ac_inode: *Inode,
  ac_sb: *SuperBlock,
  ac_o_ex: FreeExtent,             // original
  ac_g_ex: FreeExtent,             // goal (post-normalize)
  ac_b_ex: FreeExtent,             // best so far
  ac_f_ex: FreeExtent,             // best at PA-accept
  ac_orig_goal_len: ext4_grpblk_t,
  ac_prefetch_grp: ext4_group_t,
  ac_prefetch_ios: u32,
  ac_prefetch_nr: u32,
  ac_first_err: i32,
  ac_flags: u32,
  ac_groups_scanned: u16,
  ac_found: u16,
  ac_cX_found: [u16; EXT4_MB_NUM_CRS],
  ac_tail: u16,
  ac_buddy: u16,
  ac_status: u8,                   // CONTINUE | FOUND | BREAK
  ac_criteria: u8,                 // Criterion
  ac_2order: u8,
  ac_op: u8,                       // HISTORY_*
  ac_e4b: *Buddy,
  ac_bitmap_folio: *Folio,
  ac_buddy_folio: *Folio,
  ac_pa: Option<*PreallocSpace>,
  ac_lg: Option<*LocalityGroup>,
}

struct Buddy {                     // ext4_buddy
  bd_buddy_folio: *Folio,
  bd_buddy: *u8,
  bd_bitmap_folio: *Folio,
  bd_bitmap: *u8,
  bd_info: *GroupInfo,
  bd_sb: *SuperBlock,
  bd_blkbits: u16,
  bd_group: ext4_group_t,
}

struct PreallocSpace {             // ext4_prealloc_space
  pa_node: { inode_node: RbNode | lg_list: ListHead },
  pa_group_list: ListHead,
  u: { pa_tmp_list: ListHead | pa_rcu: RcuHead },
  pa_lock: SpinLock,
  pa_count: AtomicI32,
  pa_deleted: u32,
  pa_pstart: ext4_fsblk_t,
  pa_lstart: ext4_lblk_t,
  pa_len: ext4_grpblk_t,
  pa_free: ext4_grpblk_t,
  pa_type: u16,                    // MB_INODE_PA | MB_GROUP_PA
  pa_node_lock: { inode_lock: *RwLock | lg_lock: *SpinLock },
  pa_inode: *Inode,
}

struct LocalityGroup {             // per-CPU
  lg_mutex: Mutex,
  lg_prealloc_list: [ListHead; PREALLOC_TB_SIZE],   // PREALLOC_TB_SIZE = 10
  lg_prealloc_lock: SpinLock,
}

struct FreedData {                 // ext4_free_data
  efd_list: ListHead,              // per-sbi
  efd_node: RbNode,                // per-group
  efd_group: ext4_group_t,
  efd_start_cluster: ext4_grpblk_t,
  efd_count: ext4_grpblk_t,
  efd_tid: tid_t,
}

enum Criterion {
  CR_POWER2_ALIGNED = 0,
  CR_GOAL_LEN_FAST = 1,
  CR_BEST_AVAIL_LEN = 2,
  CR_GOAL_LEN_SLOW = 3,
  CR_ANY_FREE = 4,
  EXT4_MB_NUM_CRS = 5,
}
```

`Mb::new_blocks(handle, ar) -> Result<ext4_fsblk_t>`:
1. /* Fast-replay fork */
2. if sbi.s_mount_state & EXT4_FC_REPLAY: return Mb::new_blocks_simple(ar).
3. /* Quota privilege for quota files */
4. if is_quota_file(ar.inode): ar.flags |= EXT4_MB_USE_ROOT_BLOCKS.
5. /* Claim cluster reservation + quota */
6. if !(ar.flags & EXT4_MB_DELALLOC_RESERVED):
   - while ar.len ∧ ext4_claim_free_clusters(sbi, ar.len, ar.flags): cond_resched; ar.len >>= 1.
   - if ar.len == 0: return Err(ENOSPC).
   - dquot_alloc_block_nofail (root) or dquot_alloc_block (user) shrinking ar.len on EDQUOT.
7. /* Build ac */
8. ac = kmem_cache_zalloc(ext4_ac_cachep, GFP_NOFS).
9. Mb::initialize_context(ac, ar).
10. /* PA fast-path */
11. ac.ac_op = HISTORY_PREALLOC; seq = this_cpu_read(discard_pa_seq).
12. if Mb::use_preallocated(ac): goto found.
13. /* Real alloc */
14. ac.ac_op = HISTORY_ALLOC.
15. Mb::normalize_request(ac, ar).
16. Mb::pa_alloc(ac)?.
17. loop {
    - Mb::regular_allocator(ac)?.
    - if ac.ac_status == FOUND: break.
    - if retries++ < 3 ∧ Mb::discard_preallocations_should_retry(sb, ac, &seq): continue.
    - return Err(ENOSPC).
   }
18. /* Commit to disk */
19. found: Mb::mark_diskspace_used(ac, handle)?.
20. block = ext4_grp_offs_to_block(sb, &ac.ac_b_ex); ar.len = ac.ac_b_ex.fe_len.
21. /* Cleanup */
22. Mb::release_context(ac); kmem_cache_free.
23. Quota restore for the partial-allocation gap.
24. return block.

`Mb::regular_allocator(ac) -> Result<()>`:
1. /* Goal shortcut */
2. Mb::find_by_goal(ac, &e4b)?.
3. if found ∨ (ac.ac_flags & GOAL_ONLY): return.
4. /* Pick starting criterion */
5. i = fls(ac.ac_g_ex.fe_len).
6. if i ∈ [sbi.s_mb_order2_reqs, MB_NUM_ORDERS(sb)] ∧ is_power_of_2(fe_len):
   - ac.ac_2order = i - 1.
7. ac.ac_criteria = if ac.ac_2order != 0 { CR_POWER2_ALIGNED } else { CR_GOAL_LEN_FAST }.
8. /* Stream */
9. if ac.ac_flags & STREAM_ALLOC:
   - ac.ac_g_ex.fe_group = sbi.s_mb_last_groups[inode.i_ino % nr_globals].
   - clear HINT_TRY_GOAL.
10. /* Loop */
11. loop {
    - while ac.ac_criteria < EXT4_MB_NUM_CRS:
      - Mb::scan_groups(ac)?.
      - if ac.ac_status != CONTINUE: break.
    - if ac.ac_b_ex.fe_len > 0 ∧ !found ∧ !(ac_flags & HINT_FIRST):
      - Mb::try_best_found(ac, &e4b).
      - if still !found:
        - atomic_inc(&sbi.s_mb_lost_chunks).
        - reset ac.ac_b_ex; ac.ac_criteria = CR_ANY_FREE; ac.ac_flags |= HINT_FIRST; continue.
    - break.
   }
12. /* Stats + cleanup */
13. if found: atomic64_inc(&sbi.s_bal_cX_hits[ac.ac_criteria]).
14. if ac.ac_prefetch_nr: Mb::prefetch_fini(sb, ac.ac_prefetch_grp, ac.ac_prefetch_nr).

`Mb::use_preallocated(ac) -> bool`:
1. /* Per-inode PA */
2. read_lock(&ei.i_prealloc_lock).
3. node = rb_lookup_le(&ei.i_prealloc_node, ac.ac_o_ex.fe_logical).
4. if pa = container_of(node) and pa.pa_lstart ≤ logical < pa_lstart+pa_len and pa_free ≥ o_ex.fe_len:
   - Mb::pa_goal_check(ac, pa).
   - Mb::use_inode_pa(ac, pa). read_unlock; return true.
5. read_unlock.
6. /* Per-LG PA */
7. if !(ac.ac_flags & HINT_GROUP_ALLOC): return false.
8. for hash in (order_of(o_ex.fe_len) .. PREALLOC_TB_SIZE):
   - rcu_read_lock(lg.lg_prealloc_list[hash]).
   - foreach pa in list_rcu:
     - if Mb::check_group_pa(goal_block, pa, &cpa):
       - Mb::use_group_pa(ac, cpa). return true.
   - rcu_read_unlock.
9. return false.

`Mb::mark_diskspace_used(ac, handle) -> Result<()>`:
1. group = ac.ac_b_ex.fe_group; start = ac.ac_b_ex.fe_start.
2. bitmap_bh = ext4_read_block_bitmap(sb, group)?.
3. gdp = ext4_get_group_desc(sb, group)?.
4. ext4_lock_group(sb, group).
5. Mb::mark_context(handle, sb, true /*state*/, group, start, len, MARK_FLAGS, &changed).
6. ext4_free_group_clusters_set(sb, gdp, ext4_free_group_clusters(sb, gdp) - len).
7. ext4_block_bitmap_csum_set(sb, gdp, bitmap_bh).
8. ext4_group_desc_csum_set(sb, group, gdp).
9. ext4_unlock_group(sb, group).
10. percpu_counter_sub(&sbi.s_freeclusters_counter, len).
11. ext4_handle_dirty_metadata for both bitmap_bh and gdp.

`Mb::free_blocks(handle, inode, bh, block, count, flags)`:
1. /* Bounds + system-zone validation */
2. ext4_check_blockref(inode, block, count)?.
3. /* Split across groups if needed */
4. while count > 0:
   - (group, blkoff) = ext4_get_group_no_and_offset(sb, block).
   - cluster_start = blkoff >> sbi.s_cluster_bits; cluster_len = ...
   - bitmap_bh = ext4_read_block_bitmap(sb, group)?.
   - ext4_lock_group(sb, group).
   - Mb::clear_bb(handle, inode, block, cluster_len, bitmap_bh, group, blkoff, flags).
   - if jbd2-defer-needed: Mb::free_metadata enqueues FreedData on grp.bb_free_root rbtree + sbi.s_freed_data_list.
   - else: mb_free_blocks immediate via buddy.
   - if !(flags & FREE_BLOCKS_NO_QUOT_UPDATE): dquot_free_block.
   - ext4_unlock_group.
   - advance by cluster_len.

### Out of Scope

- fs/ext4/extents.c extent-tree manipulation — covered in `fs/ext4/extents.md`
- fs/ext4/balloc.c block bitmap I/O wrappers — covered separately if expanded
- fs/ext4/super.c sbi tunables (sysfs `/sys/fs/ext4/<dev>/*`) — covered in `fs/ext4/super.md`
- fs/ext4/extents_status.c es-cache — covered separately if expanded
- fs/jbd2/* transaction layer — covered in `fs/jbd2/*.md`
- fs/ext4/fast_commit.c FC replay path — covered separately if expanded
- fs/ext4/ioctl.c FITRIM ioctl plumbing — covered in `fs/ext4/ioctl.md` if expanded
- fs/ext4/resize.c online resize (uses EXT4_MB_USE_RESERVED) — covered separately if expanded
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ext4_allocation_request` | per-request descriptor (inode, len, goal, flags) | `AllocReq` |
| `struct ext4_allocation_context` | per-in-flight state | `AllocCtx` |
| `struct ext4_buddy` | per-group buddy-bitmap handle | `Buddy` |
| `struct ext4_free_extent` | per-physical-extent record | `FreeExtent` |
| `struct ext4_prealloc_space` | per-PA window (inode or LG) | `PreallocSpace` |
| `struct ext4_locality_group` | per-CPU LG | `LocalityGroup` |
| `struct ext4_free_data` | per-freed-extent staged-to-commit | `FreedData` |
| `struct ext4_group_info` | per-group state (free, frag, prealloc list) | `GroupInfo` |
| `enum criteria` (CR_POWER2_ALIGNED..CR_ANY_FREE) | per-pass strategy | `Criterion` |
| `ext4_mb_new_blocks()` | per-allocate entry | `Mb::new_blocks` |
| `ext4_mb_new_blocks_simple()` | per-fc-replay slow path | `Mb::new_blocks_simple` |
| `ext4_mb_initialize_context()` | per-ac setup | `Mb::initialize_context` |
| `ext4_mb_normalize_request()` | per-request → per-PA-window normalize | `Mb::normalize_request` |
| `ext4_mb_normalize_group_request()` | per-LG-request normalize | `Mb::normalize_group_request` |
| `ext4_mb_use_preallocated()` | per-PA-cache lookup | `Mb::use_preallocated` |
| `ext4_mb_use_inode_pa()` | per-inode-PA consume | `Mb::use_inode_pa` |
| `ext4_mb_use_group_pa()` | per-LG-PA consume | `Mb::use_group_pa` |
| `ext4_mb_regular_allocator()` | per-criteria-loop allocator | `Mb::regular_allocator` |
| `ext4_mb_find_by_goal()` | per-CR_GOAL_LEN_FAST goal hit | `Mb::find_by_goal` |
| `ext4_mb_scan_groups()` | per-criteria group iteration | `Mb::scan_groups` |
| `ext4_mb_scan_groups_p2_aligned()` | per-CR_POWER2_ALIGNED | `Mb::scan_p2_aligned` |
| `ext4_mb_scan_groups_goal_fast()` | per-CR_GOAL_LEN_FAST | `Mb::scan_goal_fast` |
| `ext4_mb_scan_groups_best_avail()` | per-CR_BEST_AVAIL_LEN | `Mb::scan_best_avail` |
| `ext4_mb_scan_groups_linear()` | per-CR_GOAL_LEN_SLOW / CR_ANY_FREE | `Mb::scan_linear` |
| `ext4_mb_simple_scan_group()` | per-power-of-2 buddy scan | `Mb::simple_scan_group` |
| `ext4_mb_complex_scan_group()` | per-arbitrary-len buddy scan | `Mb::complex_scan_group` |
| `ext4_mb_good_group()` / `_nolock()` | per-group eligibility | `Mb::good_group` |
| `ext4_mb_measure_extent()` | per-found-extent score | `Mb::measure_extent` |
| `ext4_mb_use_best_found()` | per-commit-best | `Mb::use_best_found` |
| `ext4_mb_try_best_found()` | per-fallback-claim | `Mb::try_best_found` |
| `ext4_mb_mark_diskspace_used()` | per-bitmap+groupdesc update | `Mb::mark_diskspace_used` |
| `ext4_mb_mark_context()` | per-jbd2 state-transition helper | `Mb::mark_context` |
| `ext4_mb_load_buddy_gfp()` / `_load_buddy()` | per-load buddy folio + lock | `Mb::load_buddy` |
| `ext4_mb_unload_buddy()` | per-release buddy refs | `Mb::unload_buddy` |
| `ext4_mb_init_group()` | per-init group buddy on-demand | `Mb::init_group` |
| `ext4_mb_generate_buddy()` | per-build buddy from bitmap | `Mb::generate_buddy` |
| `mb_find_extent()` | per-buddy-bitmap extent locate | `Mb::find_extent_buddy` |
| `mb_mark_used()` | per-buddy-bitmap mark | `Mb::mark_used_buddy` |
| `mb_free_blocks()` | per-buddy-bitmap free | `Mb::free_blocks_buddy` |
| `ext4_mb_prefetch()` / `_fini()` | per-bg-bitmap I/O prefetch | `Mb::prefetch`, `Mb::prefetch_fini` |
| `ext4_mb_new_inode_pa()` | per-inode-PA creation | `Mb::new_inode_pa` |
| `ext4_mb_new_group_pa()` | per-LG-PA creation | `Mb::new_group_pa` |
| `ext4_mb_pa_alloc()` | per-PA-slab alloc | `Mb::pa_alloc` |
| `ext4_mb_release_inode_pa()` | per-discard-PA leftover | `Mb::release_inode_pa` |
| `ext4_mb_release_group_pa()` | per-discard-LG-PA leftover | `Mb::release_group_pa` |
| `ext4_discard_preallocations()` | per-inode flush-all-PA | `Mb::discard_preallocations` |
| `ext4_mb_discard_group_preallocations()` | per-group reclaim PA | `Mb::discard_group_preallocations` |
| `ext4_mb_discard_lg_preallocations()` | per-LG reclaim | `Mb::discard_lg_preallocations` |
| `ext4_mb_release_context()` | per-end-of-ac cleanup | `Mb::release_context` |
| `ext4_free_blocks()` | per-free entry | `Mb::free_blocks` |
| `ext4_mb_free_metadata()` | per-stage-free metadata | `Mb::free_metadata` |
| `ext4_process_freed_data()` | per-jbd2-commit-callback release | `Mb::process_freed_data` |
| `ext4_mb_init()` / `_release()` | per-mount/umount init | `Mb::init`, `Mb::release` |
| `ext4_mb_alloc_groupinfo()` / `_add_groupinfo()` | per-add group | `Mb::alloc_groupinfo` |
| `ext4_trim_fs()` | per-FITRIM ioctl | `Mb::trim_fs` |
| `ext4_mballoc_query_range()` | per-query for FSMAP | `Mb::query_range` |
| `MB_DEFAULT_MAX_TO_SCAN` (200) | per-MAX_GROUPS_SCANNED | const |
| `MB_DEFAULT_MIN_TO_SCAN` (10) | per-MIN_GROUPS_SCANNED | const |
| `MB_DEFAULT_STREAM_THRESHOLD` (16) | per-stream-vs-LG threshold | const |
| `MB_DEFAULT_ORDER2_REQS` (2) | per-buddy-order-2 threshold | const |
| `MB_DEFAULT_GROUP_PREALLOC` (512) | per-LG-PA size | const |
| `MB_DEFAULT_LINEAR_LIMIT` (4) | per-linear-vs-optimized switch | const |
| `MB_DEFAULT_BEST_AVAIL_TRIM_ORDER` (3) | per-CR_BEST_AVAIL trim | const |
| `PREALLOC_TB_SIZE` (10) | per-LG hash size | const |

### compatibility contract

REQ-1: struct ext4_allocation_request:
- inode: target inode.
- len: requested blocks (clamped during quota / claim).
- logical: target logical block within inode.
- lleft / lright: closest left/right allocated logical neighbours.
- goal: physical-block hint (target).
- pleft / pright: physical neighbours of lleft / lright.
- flags: EXT4_MB_HINT_* bitset (DATA, MERGE, FIRST, NOPREALLOC, GROUP_ALLOC, GOAL_ONLY, TRY_GOAL, DELALLOC_RESERVED, STREAM_ALLOC, USE_ROOT_BLOCKS, USE_RESERVED, STRICT_CHECK).

REQ-2: struct ext4_allocation_context:
- ac_inode, ac_sb: target inode + super.
- ac_o_ex: original request (fe_logical, fe_len).
- ac_g_ex: goal request (post-normalize).
- ac_b_ex: best-so-far (across all groups scanned).
- ac_f_ex: best-found at time of PA-attempt-acceptance.
- ac_orig_goal_len: original goal length (preserved across CR_BEST_AVAIL_LEN trims).
- ac_prefetch_grp / _ios / _nr: per-prefetch state.
- ac_first_err: first non-fatal error.
- ac_flags: copy of ar.flags + dynamic state.
- ac_groups_scanned, ac_found, ac_cX_found[EXT4_MB_NUM_CRS]: stats.
- ac_status ∈ {AC_STATUS_CONTINUE, AC_STATUS_FOUND, AC_STATUS_BREAK}.
- ac_criteria: current criterion (CR_*).
- ac_2order: log2(len) if request is power-of-2 with N ≥ MB_DEFAULT_ORDER2_REQS, else 0.
- ac_op: history op (EXT4_MB_HISTORY_ALLOC | _PREALLOC).
- ac_e4b: per-group buddy handle.
- ac_bitmap_folio, ac_buddy_folio: pinned bitmap + buddy folios.
- ac_pa: in-flight PA reservation.
- ac_lg: per-CPU locality group (set when STREAM_ALLOC).

REQ-3: struct ext4_buddy:
- bd_buddy_folio, bd_buddy: per-group buddy bitmap pointer.
- bd_bitmap_folio, bd_bitmap: per-group block bitmap pointer.
- bd_info: pointer to GroupInfo.
- bd_sb: super.
- bd_blkbits: log2(blocksize).
- bd_group: group index.

REQ-4: struct ext4_free_extent:
- fe_logical: logical-block start (in inode).
- fe_start: cluster-offset within group.
- fe_group: group index.
- fe_len: length in clusters.

REQ-5: struct ext4_prealloc_space:
- pa_node.{inode_node|lg_list}: rbtree node for inode PA, list-head for LG PA.
- pa_group_list: per-group bb_prealloc_list link.
- pa_lock: pa-internal lock.
- pa_count: refcount.
- pa_deleted: tombstone for RCU teardown.
- pa_pstart, pa_lstart: physical and logical start.
- pa_len: total reserved length (clusters).
- pa_free: remaining free clusters within PA.
- pa_type ∈ {MB_INODE_PA, MB_GROUP_PA}.
- pa_node_lock.{inode_lock|lg_lock}: external lock for the container.
- pa_inode: owning inode (for group-discard reverse-lookup).

REQ-6: struct ext4_locality_group (per-CPU):
- lg_mutex: serializes within LG.
- lg_prealloc_list[PREALLOC_TB_SIZE]: PA hash by fls(pa_free)-1.
- lg_prealloc_lock: spinlock.

REQ-7: struct ext4_free_data:
- efd_list: per-sbi staged-extents linkage.
- efd_node: per-group rbtree node.
- efd_group: group containing the freed range.
- efd_start_cluster, efd_count: cluster range.
- efd_tid: jbd2 transaction id that freed it (release deferred until commit).

REQ-8: enum criteria (ordered, fall-through):
- CR_POWER2_ALIGNED: power-of-2 length using buddy bitmap; no disk I/O.
- CR_GOAL_LEN_FAST: in-memory data-structure search for goal-meeting group; no disk I/O.
- CR_BEST_AVAIL_LEN: trim goal length up to MB_DEFAULT_BEST_AVAIL_TRIM_ORDER and re-search.
- CR_GOAL_LEN_SLOW: sequential per-group scan with possible disk I/O.
- CR_ANY_FREE: any free chunk; last resort.
- EXT4_MB_NUM_CRS == 5.

REQ-9: ext4_mb_new_blocks(handle, ar, errp) — top entry:
- /* Trace + replay shim */
- trace_ext4_request_blocks(ar).
- if sbi.s_mount_state & EXT4_FC_REPLAY: return ext4_mb_new_blocks_simple(ar, errp).
- /* Quota-file privilege */
- if ext4_is_quota_file(ar.inode): ar.flags |= EXT4_MB_USE_ROOT_BLOCKS.
- /* Claim free clusters + quota (skipped when DELALLOC_RESERVED) */
- while ar.len ∧ ext4_claim_free_clusters(sbi, ar.len, ar.flags):
  - cond_resched(); ar.len >>= 1.
- if !ar.len: *errp = -ENOSPC; show_pa; return 0.
- reserv_clstrs = ar.len.
- if USE_ROOT_BLOCKS: dquot_alloc_block_nofail. else: dquot_alloc_block with ar.len shrink-on-fail.
- /* Allocate context */
- ac = kmem_cache_zalloc(ext4_ac_cachep, GFP_NOFS).
- ext4_mb_initialize_context(ac, ar).
- /* PA cache hit */
- ac.ac_op = EXT4_MB_HISTORY_PREALLOC; seq = this_cpu_read(discard_pa_seq).
- if ext4_mb_use_preallocated(ac): goto found.
- /* Real allocation */
- ac.ac_op = EXT4_MB_HISTORY_ALLOC.
- ext4_mb_normalize_request(ac, ar).
- ext4_mb_pa_alloc(ac).
- repeat: ext4_mb_regular_allocator(ac).
- if !found and retries < 3 and discard_preallocations_should_retry: goto repeat.
- if found: ext4_mb_mark_diskspace_used(ac, handle); block = ext4_grp_offs_to_block(sb, &ac.ac_b_ex); ar.len = ac.ac_b_ex.fe_len.
- else: *errp = -ENOSPC.
- ext4_mb_release_context; kmem_cache_free; restore quota on partial. Return block.

REQ-10: ext4_mb_initialize_context(ac, ar):
- ac.ac_inode = ar.inode; ac.ac_sb = ar.inode.i_sb.
- Maps (ar.goal, ar.logical, ar.len) → (ac.ac_o_ex.fe_group, fe_start, fe_logical, fe_len) in cluster units.
- ac.ac_g_ex = ac.ac_o_ex (pre-normalize).
- ac.ac_flags = ar.flags.
- Decide LG: ext4_mb_group_or_file picks per-CPU LG for small/streaming requests (size < MB_DEFAULT_STREAM_THRESHOLD); sets EXT4_MB_STREAM_ALLOC | EXT4_MB_HINT_GROUP_ALLOC.

REQ-11: ext4_mb_normalize_request(ac, ar) — extend the goal upward to a per-inode-PA-window:
- Computes desired PA window around (ar.logical) — round to power-of-2 sized window between sbi.s_mb_min_to_scan and sbi.s_mb_max_to_scan.
- ext4_mb_pa_adjust_overlap shrinks/aligns window against existing inode PAs (overlap-free invariant).
- Asserts no overlap via ext4_mb_pa_assert_overlap.
- Final ac.ac_g_ex.fe_len ≥ ac.ac_o_ex.fe_len; logical aligned.

REQ-12: ext4_mb_use_preallocated(ac) — PA cache lookup:
- Per-inode PA rbtree lookup by pa_lstart: find PA whose [pa_lstart, pa_lstart + pa_len) covers ac.ac_o_ex.fe_logical.
- if hit ∧ pa.pa_free ≥ ac.ac_o_ex.fe_len ∧ ext4_mb_pa_goal_check passes: ext4_mb_use_inode_pa.
- else iterate LG list at hash(fls(pa_free)-1); ext4_mb_check_group_pa for fit; if accepted: ext4_mb_use_group_pa.
- Returns true on hit.

REQ-13: ext4_mb_use_inode_pa(ac, pa):
- ac.ac_b_ex.fe_logical = ac.ac_o_ex.fe_logical.
- offset = ac.ac_o_ex.fe_logical - pa.pa_lstart.
- ac.ac_b_ex.fe_group, fe_start = ext4_get_group_no_and_offset(sb, pa.pa_pstart + offset).
- ac.ac_b_ex.fe_len = min(pa.pa_free, ac.ac_o_ex.fe_len).
- pa.pa_free -= fe_len.
- pa.pa_count++ (held until release_context).
- ac.ac_status = AC_STATUS_FOUND; ac.ac_pa = pa.

REQ-14: ext4_mb_regular_allocator(ac) — criteria loop:
- BUG_ON(ac.ac_status == AC_STATUS_FOUND).
- /* Goal hit shortcut */
- ext4_mb_find_by_goal(ac, &e4b). If found: return.
- if ac.ac_flags & EXT4_MB_HINT_GOAL_ONLY: return.
- /* Pick starting criterion */
- i = fls(ac.ac_g_ex.fe_len).
- if i ∈ [sbi.s_mb_order2_reqs, MB_NUM_ORDERS(sb)] ∧ is_power_of_2(ac.ac_g_ex.fe_len):
  - ac.ac_2order = i - 1.
- ac.ac_criteria = (ac.ac_2order != 0) ? CR_POWER2_ALIGNED : CR_GOAL_LEN_FAST.
- /* Stream allocator override */
- if ac.ac_flags & EXT4_MB_STREAM_ALLOC:
  - hash = inode.i_ino % sbi.s_mb_nr_global_goals.
  - ac.ac_g_ex.fe_group = sbi.s_mb_last_groups[hash].
- repeat:
  - while ac.ac_criteria < EXT4_MB_NUM_CRS:
    - err = ext4_mb_scan_groups(ac).
    - if ac.ac_status != AC_STATUS_CONTINUE: break.
- if ac.ac_b_ex.fe_len > 0 ∧ !found ∧ !(ac_flags & HINT_FIRST):
  - ext4_mb_try_best_found(ac, &e4b). If still not found: atomic_inc s_mb_lost_chunks; ac.ac_criteria = CR_ANY_FREE; ac.ac_flags |= HINT_FIRST; goto repeat.
- Stats: s_bal_cX_hits[ac.ac_criteria]++; stream-goal counter on stream-hit.
- ext4_mb_prefetch_fini if any pending prefetch.

REQ-15: ext4_mb_scan_groups(ac):
- Dispatch by ac.ac_criteria:
  - CR_POWER2_ALIGNED ⟹ ext4_mb_scan_groups_p2_aligned (largest_free_order XArray).
  - CR_GOAL_LEN_FAST ⟹ ext4_mb_scan_groups_goal_fast (avg_frag_order XArray + p2_aligned XArray).
  - CR_BEST_AVAIL_LEN ⟹ ext4_mb_scan_groups_best_avail (avg_frag_order XArray with trimmed goal).
  - CR_GOAL_LEN_SLOW | CR_ANY_FREE ⟹ ext4_mb_scan_groups_linear.
- For each group, call ext4_mb_scan_group → ext4_mb_load_buddy → simple_scan_group or complex_scan_group; on suitable extent: ext4_mb_use_best_found.
- Group filter: ext4_mb_good_group_nolock + ext4_mb_good_group.

REQ-16: ext4_mb_simple_scan_group(ac, e4b):
- Used when ac.ac_2order != 0.
- mb_find_buddy(e4b, ac.ac_2order, &max).
- find_next_bit_le on buddy bitmap for first set bit; mark via mb_mark_used.
- ac.ac_b_ex = exact 2^N extent.

REQ-17: ext4_mb_complex_scan_group(ac, e4b):
- Iterates free extents via mb_find_extent.
- ext4_mb_measure_extent(ex, ac) computes score (closeness to goal, length).
- ext4_mb_check_limits aborts after ac.ac_found ≥ s_mb_max_to_scan unless best-found ≥ goal.
- After loop: ext4_mb_use_best_found.

REQ-18: ext4_mb_scan_aligned(ac, e4b):
- Specifically aligns to stripe-size (sbi.s_stripe) when set (RAID).

REQ-19: ext4_mb_use_best_found(ac, e4b):
- Sanity: ac.ac_b_ex.fe_len ≥ ac.ac_o_ex.fe_len ∨ accept-trimmed.
- ac.ac_status = AC_STATUS_FOUND.
- Computes ac.ac_b_ex.fe_logical from offset within PA window.
- if !(ac.ac_flags & HINT_NOPREALLOC): ext4_mb_new_preallocation creates new inode-PA or LG-PA from leftover ac.ac_g_ex.

REQ-20: ext4_mb_new_inode_pa(ac):
- pa = ac.ac_pa (pre-allocated by ext4_mb_pa_alloc).
- pa.pa_pstart = ext4_grp_offs_to_block(sb, &ac.ac_b_ex).
- pa.pa_lstart, pa.pa_len = window from ac.ac_g_ex.
- pa.pa_free = pa.pa_len - ac.ac_b_ex.fe_len.
- pa.pa_type = MB_INODE_PA.
- pa.pa_node_lock.inode_lock = &ei.i_prealloc_lock.
- write_lock; ext4_mb_pa_rb_insert into ei.i_prealloc_node.
- Adds pa to grp.bb_prealloc_list.

REQ-21: ext4_mb_new_group_pa(ac):
- pa = ac.ac_pa.
- pa.pa_pstart = ext4_grp_offs_to_block(sb, &ac.ac_b_ex).
- pa.pa_lstart = 0 (LG PA is logical-agnostic).
- pa.pa_len = MB_DEFAULT_GROUP_PREALLOC (or ac.ac_b_ex.fe_len).
- pa.pa_type = MB_GROUP_PA.
- pa.pa_node_lock.lg_lock = &lg.lg_prealloc_lock.
- list_add_rcu into lg.lg_prealloc_list[order].

REQ-22: ext4_mb_mark_diskspace_used(ac, handle):
- Loads bitmap_bh = ext4_read_block_bitmap(sb, group).
- Loads gdp = ext4_get_group_desc(sb, group).
- BUG-checks bitmap range not already set (ext4_mb_check_ondisk_bitmap optionally).
- ext4_mb_mark_context(handle, sb, true, group, start, len, EXT4_MB_BITMAP_MARKED_CHECK, &changed).
- Adjusts gdp.bg_free_blocks_count -= len.
- Recomputes bg_checksum + bg_block_bitmap_csum.
- ext4_handle_dirty_metadata for bitmap + gdp.
- percpu_counter_sub(&sbi.s_freeclusters_counter, len).

REQ-23: ext4_mb_load_buddy(sb, group, e4b) → loads bitmap + buddy folios:
- If group's buddy not initialized: ext4_mb_init_group(sb, group, GFP_NOFS) — reads bitmap, calls ext4_mb_generate_buddy.
- Pins both folios; folio_lock for write-on-modify is via ext4_lock_group.

REQ-24: ext4_mb_generate_buddy(sb, buddy, bitmap, group, grp):
- For each order 0..MB_NUM_ORDERS-1: derive parent free-bits from child pair.
- Updates grp.bb_counters[order] free-count per order.
- mb_set_largest_free_order, mb_update_avg_fragment_size update XArray indices when mb_optimize_scan.

REQ-25: mb_find_extent(e4b, block, needed, ex):
- Locates contiguous run of `needed` free clusters starting at `block` in buddy bitmap.
- Walks buddy at successively lower orders.

REQ-26: mb_mark_used(e4b, ex):
- Updates buddy bitmap to mark ex as used.
- mb_buddy_mark_used decrements grp.bb_counters; rebalances orders.
- Returns leftover length when ex partially fits.

REQ-27: mb_free_blocks(inode, e4b, first, count):
- Clears bits via mb_buddy_mark_free; merges buddies upward order-by-order.
- Updates grp.bb_counters, largest_free_order, avg_fragment_size.

REQ-28: ext4_free_blocks(handle, inode, bh, block, count, flags):
- Validates [block, block+count] in fs-bounds, not system zone.
- For each spanned group:
  - bitmap_bh = ext4_read_block_bitmap.
  - ext4_lock_group; ext4_mb_clear_bb clears bitmap bits + double-checks already-clear.
  - if !(flags & EXT4_FREE_BLOCKS_NO_QUOT_UPDATE): dquot_free_block.
  - if jbd2 needs commit-deferral: ext4_mb_free_metadata enqueues ext4_free_data on per-group rbtree.
  - else: mb_free_blocks immediately.

REQ-29: ext4_process_freed_data(sb, commit_tid):
- Called from jbd2 commit-callback.
- Walks per-sbi efd_list for entries with efd_tid ≤ commit_tid.
- For each: ext4_free_data_in_buddy releases the buddy bits (now safe — txn committed).
- ext4_issue_discard if mount has discard flag and entry is large enough.

REQ-30: ext4_discard_preallocations(inode):
- Iterates ei.i_prealloc_node rbtree; under write-lock, mark each pa.pa_deleted = 1.
- Drops references; RCU-defers ext4_mb_pa_callback.
- Releases bitmap blocks via ext4_mb_release_inode_pa (buddy mb_free_blocks for leftover pa_free).

REQ-31: ext4_mb_init(sb) — at mount:
- ext4_mb_alloc_groupinfo + ext4_mb_init_backend (per-group GroupInfo allocation).
- s_locality_groups = alloc_percpu(struct ext4_locality_group); init each.
- s_mb_offsets, s_mb_maxs constructed for buddy.
- Tunables read from sysfs defaults: s_mb_max_to_scan, s_mb_min_to_scan, s_mb_stats, s_mb_order2_reqs, s_mb_group_prealloc, s_mb_stream_request, s_mb_max_inode_prealloc.
- Initializes XArrays sbi.s_mb_largest_free_orders, s_mb_avg_fragment_size when mb_optimize_scan mount option (toggled by EXT4_MOUNT2_MB_OPTIMIZE_SCAN).

REQ-32: ext4_mb_release(sb) — at umount:
- Flush ext4_discard_work delayed work.
- For each group: ext4_mb_cleanup_pa drains residual prealloc.
- Free buddy bitmap pages, GroupInfo arrays, per-cpu LGs.
- ext4_mb_avg_fragment_size_destroy + ext4_mb_largest_free_orders_destroy.

REQ-33: ext4_trim_fs(sb, range) — FITRIM ioctl:
- Iterates groups in [range.start, range.start+range.len].
- For each: ext4_trim_all_free → repeated ext4_try_to_trim_range → ext4_trim_extent → ext4_issue_discard.
- Honors range.minlen as floor on each contiguous run.
- Returns total trimmed bytes.

REQ-34: mb_optimize_scan toggle:
- When set: scan_groups_p2_aligned / scan_groups_goal_fast / scan_groups_avg_frag_order_range use XArray range iteration over groups indexed by largest_free_order / avg_fragment_size_order.
- When unset: scan_groups_linear (O(N) walk), with linear-limit MB_DEFAULT_LINEAR_LIMIT skipping early-fail-fast scans.
- Sysfs toggle: `/sys/fs/ext4/<dev>/mb_optimize_scan`.

REQ-35: EXT4_MB_HINT_* and EXT4_MB_* flags:
- EXT4_MB_HINT_DATA: data block (vs metadata).
- EXT4_MB_HINT_FIRST: prefer "first" block (used during re-allocate-best fallback).
- EXT4_MB_HINT_MERGE: prefer goal-adjacent (post-extent-merge).
- EXT4_MB_HINT_NOPREALLOC: skip PA creation (small tails).
- EXT4_MB_HINT_GROUP_ALLOC: use LG PA path.
- EXT4_MB_HINT_GOAL_ONLY: only return if goal is satisfiable (used by some converters).
- EXT4_MB_HINT_TRY_GOAL: try goal first but allow fallback.
- EXT4_MB_DELALLOC_RESERVED: blocks already counted via delalloc reservation.
- EXT4_MB_STREAM_ALLOC: stream/file-small request (LG-locality).
- EXT4_MB_USE_ROOT_BLOCKS: dip into root-reserved pool (CAP_SYS_RESOURCE or quota-file).
- EXT4_MB_USE_RESERVED: use the "reserved" cluster pool (atomic-write / resize emergencies).
- EXT4_MB_STRICT_CHECK: strict free-block recount when retrying.

REQ-36: MBALLOC_F_ATOMIC (via EXT4_GET_BLOCKS_USE_RESERVED + EXT4_MB_USE_RESERVED):
- Used by `ext4_convert_unwritten_extents_atomic` and resize paths to draw from a reserved pool that doesn't honor quotas.
- ext4_claim_free_clusters bypasses normal reservation check.

REQ-37: Per-CPU LG hot-path (ext4_mb_group_or_file):
- if request length ≤ MB_DEFAULT_STREAM_THRESHOLD ∧ EXT4_FEATURE_INCOMPAT_FLEX_BG not set as obstruction:
  - get_cpu_ptr(sbi.s_locality_groups).
  - mutex_lock(&lg.lg_mutex).
  - ac.ac_lg = lg; ac.ac_flags |= EXT4_MB_HINT_GROUP_ALLOC | EXT4_MB_STREAM_ALLOC.

REQ-38: Retry on PA exhaustion:
- ext4_mb_discard_preallocations_should_retry(sb, ac, &seq) attempts to invoke ext4_mb_discard_group_preallocations on groups with stale PAs and rerun the allocator (max 3 times).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ar_len_monotonically_decreases_in_claim_loop` | INVARIANT | per-new_blocks: ar.len only decreases in claim-free-clusters loop. |
| `ac_status_state_machine` | INVARIANT | per-ac: ac_status transitions: CONTINUE → (CONTINUE | FOUND | BREAK); FOUND terminal. |
| `criteria_monotonic_within_scan` | INVARIANT | per-regular_allocator: ac_criteria only increases (except CR_ANY_FREE reset). |
| `pa_free_nonneg` | INVARIANT | per-PA: pa_free ≥ 0 throughout. |
| `pa_overlap_free_per_inode` | INVARIANT | per-inode-PA-rbtree: any two PAs have disjoint [lstart, lstart+len). |
| `lg_mutex_held_during_lg_path` | INVARIANT | per-LG-alloc: lg.lg_mutex held from group_or_file to release_context. |
| `bitmap_bit_cleared_on_alloc_taken` | INVARIANT | per-mark_diskspace_used: bitmap_bh's selected bits were 0 pre and 1 post. |
| `buddy_counters_sum_invariant` | INVARIANT | per-group: sum_{order}(bb_counters[order] * 2^order) == free clusters in group. |
| `discard_pa_seq_monotonic` | INVARIANT | per-CPU discard_pa_seq monotonically increases. |
| `freed_data_efd_tid_ge_handle_tid` | INVARIANT | per-FreedData: efd_tid ≥ handle.h_transaction.t_tid at enqueue. |
| `cluster_alignment` | INVARIANT | per-bigalloc: ac.ac_b_ex.fe_start aligned on sbi.s_cluster_ratio. |

### Layer 2: TLA+

`fs/ext4/mballoc.tla`:
- Per-group buddy bitmap + PA pool + criteria walk.
- Operations: NewBlocks, NormalizeRequest, UsePreallocated, RegularAllocator, MarkDiskUsed, FreeBlocks, DiscardPreallocations, ProcessFreedData.
- Properties:
  - `safety_no_double_alloc` — per-cluster: never marked used twice without intervening free.
  - `safety_no_double_free` — per-cluster: never marked free twice without intervening alloc.
  - `safety_pa_disjoint` — per-inode: any two live PAs have disjoint logical ranges.
  - `safety_quota_accounted` — per-alloc: dquot reservations + claim_free_clusters balance.
  - `safety_freed_data_after_commit` — per-free: deferred FreedData released only after jbd2 commit.
  - `safety_buddy_consistent_with_bitmap` — per-group: buddy bitmap accurately mirrors block bitmap.
  - `liveness_alloc_terminates_or_ENOSPC` — per-new_blocks: ≤ 3 PA-discard retries then ENOSPC.
  - `liveness_criteria_walk_terminates` — per-regular_allocator: traverses ≤ EXT4_MB_NUM_CRS phases.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mb::new_blocks` post: ar.len ≤ original; block ∈ [s_first_data_block, s_blocks_count) on success | `Mb::new_blocks` |
| `Mb::initialize_context` post: ac.ac_o_ex == ac.ac_g_ex prior to normalize | `Mb::initialize_context` |
| `Mb::normalize_request` post: ac.ac_g_ex.fe_len ≥ ac.ac_o_ex.fe_len; window non-overlapping with extant inode PAs | `Mb::normalize_request` |
| `Mb::use_inode_pa` post: pa.pa_free -= ac.ac_b_ex.fe_len; pa.pa_count incremented | `Mb::use_inode_pa` |
| `Mb::regular_allocator` post: ac.ac_status == FOUND ⟹ ac.ac_b_ex.fe_len ≥ ac.ac_o_ex.fe_len (or HINT_FIRST trim) | `Mb::regular_allocator` |
| `Mb::find_by_goal` post: hit ⟹ ac.ac_b_ex covers goal pblk | `Mb::find_by_goal` |
| `Mb::simple_scan_group` post: returned extent is power-of-2 length 2^ac.ac_2order | `Mb::simple_scan_group` |
| `Mb::mark_diskspace_used` post: bitmap, gdp.bg_free_blocks, sbi.s_freeclusters all decremented atomically | `Mb::mark_diskspace_used` |
| `Mb::free_blocks` post: bitmap cleared ∨ FreedData enqueued; quota refunded unless NO_QUOT_UPDATE | `Mb::free_blocks` |
| `Mb::process_freed_data` post: all FreedData with efd_tid ≤ commit_tid removed from list, bits cleared in buddy | `Mb::process_freed_data` |
| `Mb::discard_preallocations` post: inode rbtree empty; all unused PA blocks returned to buddy | `Mb::discard_preallocations` |
| `Mb::init` post: per-group GroupInfo, per-CPU LGs, XArrays allocated; tunables defaulted | `Mb::init` |
| `Mb::release` post: no PA, no FreedData, no GroupInfo outstanding | `Mb::release` |

### Layer 4: Verus/Creusot functional

`Per-request → quota-claim → context-build → PA-cache-lookup → normalize → criteria-loop (goal → CR_POWER2_ALIGNED → CR_GOAL_LEN_FAST → CR_BEST_AVAIL_LEN → CR_GOAL_LEN_SLOW → CR_ANY_FREE) → best-found-fallback → diskspace-mark (bitmap + gdp + sbi counter) → release-context` and `Per-free → cluster-split → bitmap-clear → buddy-or-jbd2-defer → quota-refund` semantic equivalence: per `Documentation/filesystems/ext4/blockmap.rst` and `Documentation/admin-guide/ext4.rst` mb_optimize_scan / stripe / bigalloc.

### hardening

(Inherits row-1 features from `fs/ext4/00-overview.md` § Hardening.)

mballoc reinforcement:

- **Per-claim_free_clusters strict check** — defense against per-overcommit and per-ENOSPC-deadlock under concurrent allocators.
- **Per-PA overlap assertion (ext4_mb_pa_assert_overlap)** — defense against per-double-use of the same physical range.
- **Per-RCU PA teardown (pa_rcu, pa_deleted)** — defense against per-use-after-free in concurrent discard.
- **Per-lg_mutex serializes per-CPU LG** — defense against per-CPU LG corruption under preempt.
- **Per-bitmap-vs-buddy consistency check (__mb_check_buddy)** — defense against per-buddy-drift after a corrupted bitmap read.
- **Per-bitmap checksum verified on read** — defense against per-stale-bitmap allocation.
- **Per-group lock (ext4_lock_group) around mark/clear** — defense against per-concurrent allocator races within a group.
- **Per-FreedData jbd2-commit-deferred release** — defense against per-realloc-of-freed-block before transaction commit.
- **Per-dquot_alloc_block / _free_block correctness** — defense against per-quota-drift on partial alloc + ENOSPC paths.
- **Per-EXT4_MB_USE_RESERVED CAP_SYS_RESOURCE-gated** — defense against unprivileged exhaustion of reserved pool.
- **Per-EXT4_MB_USE_ROOT_BLOCKS only for quota-file + root** — defense against unprivileged dip into root-reserved.
- **Per-EXT4_MB_STRICT_CHECK on retry** — defense against per-stale-free-count succeeding on truly-full fs.
- **Per-mb_optimize_scan XArray bounds** — defense against per-OOB index from corrupted group_info.
- **Per-bb_counters audit at unload** — defense against per-leak of buddy state across umount.

