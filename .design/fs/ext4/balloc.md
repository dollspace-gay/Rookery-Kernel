# Tier-3: fs/ext4/balloc.c — ext4 block-bitmap allocator (legacy path)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/ext4/00-overview.md
upstream-paths:
  - fs/ext4/balloc.c (~1003 lines)
  - fs/ext4/bitmap.c (block-bitmap CRC helpers)
  - fs/ext4/ext4.h (struct ext4_group_desc, EXT4_BG_BLOCK_UNINIT)
  - fs/ext4/mballoc.c (ext4_free_blocks — bitmap-update path)
-->

## Summary

`balloc.c` implements the **block-bitmap allocator** layer used by ext4 alongside the multi-block-allocator (`mballoc.c`). Per-block-group has 1 block-bitmap (`bg_block_bitmap_lo|_hi`) of `EXT4_CLUSTERS_PER_GROUP(sb) / 8` bytes describing per-cluster allocation state (bit set = used). Per-`ext4_read_block_bitmap_nowait` reads the bitmap buffer-head, lazily-initializes uninit groups (`EXT4_BG_BLOCK_UNINIT`), and validates the on-disk CRC (`bg_block_bitmap_csum_lo|_hi` = crc32c(s_uuid + grp_num + bbitmap)). Per-`ext4_get_group_no_and_offset` maps an `ext4_fsblk_t` to a (group, intra-group-cluster-offset). Per-`ext4_count_free_clusters` aggregates `bg_free_blocks_count_lo|_hi` across all groups; debug builds re-count from bitmap. Per-`ext4_has_free_clusters` / `ext4_claim_free_clusters` enforces per-r_blocks reservation + per-resuid / -resgid root-pool. Per-`ext4_should_retry_alloc` waits on a journal-commit on ENOSPC. Per-`ext4_free_blocks` (lives in `mballoc.c`) consumes this layer to clear bitmap bits during free. Critical for: per-cluster accuracy, ENOSPC behaviour, fsck-clean bitmap state, online-resize (`s_reserved_gdt_blocks`).

This Tier-3 covers `fs/ext4/balloc.c` (~1003 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `ext4_get_group_number()` | per-blocknr → group | `Balloc::group_number` |
| `ext4_get_group_no_and_offset()` | per-blocknr → (group, offset) | `Balloc::group_no_and_offset` |
| `ext4_block_in_group()` | per-blocknr in-group? | `Balloc::block_in_group` |
| `ext4_num_overhead_clusters()` | per-group metadata-cluster count | `Balloc::num_overhead_clusters` |
| `ext4_init_block_bitmap()` | per-lazy-init bitmap (UNINIT group) | `Balloc::init_block_bitmap` |
| `ext4_free_clusters_after_init()` | per-UNINIT free count | `Balloc::free_clusters_after_init` |
| `ext4_get_group_desc()` | per-group descriptor lookup | `Balloc::get_group_desc` |
| `ext4_get_group_info()` | per-group in-memory info | `Balloc::get_group_info` |
| `ext4_validate_block_bitmap()` | per-bitmap CRC + sanity check | `Balloc::validate_block_bitmap` |
| `ext4_read_block_bitmap_nowait()` | per-group submit-read (no wait) | `Balloc::read_block_bitmap_nowait` |
| `ext4_wait_block_bitmap()` | per-group I/O completion | `Balloc::wait_block_bitmap` |
| `ext4_read_block_bitmap()` | per-group sync read+wait | `Balloc::read_block_bitmap` |
| `ext4_has_free_clusters()` | per-fs availability check | `Balloc::has_free_clusters` |
| `ext4_claim_free_clusters()` | per-fs reserve dirty | `Balloc::claim_free_clusters` |
| `ext4_should_retry_alloc()` | per-ENOSPC retry decision | `Balloc::should_retry_alloc` |
| `ext4_new_meta_blocks()` | per-meta alloc wrapper → mballoc | `Balloc::new_meta_blocks` |
| `ext4_count_free_clusters()` | per-fs free-cluster sum | `Balloc::count_free_clusters` |
| `ext4_count_free()` | per-byte-array bit-count helper (`bitmap.c`) | `Balloc::count_free` |
| `ext4_block_bitmap_csum_verify()` | per-bitmap CRC verify (`bitmap.c`) | `Balloc::bbitmap_csum_verify` |
| `ext4_block_bitmap_csum_set()` | per-bitmap CRC set (`bitmap.c`) | `Balloc::bbitmap_csum_set` |
| `ext4_bg_has_super()` | per-group has-SB? | `Balloc::bg_has_super` |
| `ext4_bg_num_gdb()` | per-group GDT-block count | `Balloc::bg_num_gdb` |
| `ext4_num_base_meta_blocks()` | per-group base meta block count | `Balloc::num_base_meta_blocks` |
| `ext4_inode_to_goal_block()` | per-inode allocation hint | `Balloc::inode_to_goal_block` |
| `struct ext4_group_desc` | per-group on-disk descriptor (32B / 64B) | `Ext4GroupDesc` |
| `EXT4_BG_BLOCK_UNINIT` | per-bg_flags lazy-init flag | shared |
| `EXT4_GROUP_INFO_BBITMAP_CORRUPT` | per-group corruption bit | shared |
| `EXT4_MAX_BLOCKS()` | per-(size, offset, blkbits) upper bound | shared |

## Compatibility contract

REQ-1: struct ext4_group_desc (on-disk; 32B legacy, 64B with 64BIT feature):
- bg_block_bitmap_lo / _hi (u32+u32): per-group block-bitmap block-no.
- bg_inode_bitmap_lo / _hi (u32+u32): per-group inode-bitmap block-no.
- bg_inode_table_lo / _hi (u32+u32): per-group inode-table first block.
- bg_free_blocks_count_lo / _hi (u16+u16): per-group cached free-cluster count.
- bg_free_inodes_count_lo / _hi (u16+u16): per-group cached free-inode count.
- bg_used_dirs_count_lo / _hi (u16+u16): per-group #directories (Orlov hint).
- bg_flags (u16): per-group EXT4_BG_INODE_UNINIT | _BLOCK_UNINIT | _INODE_ZEROED.
- bg_exclude_bitmap_lo / _hi (u32+u32): per-snapshots exclude bitmap.
- bg_block_bitmap_csum_lo / _hi (u16+u16): per-group crc32c(s_uuid + grp_num + bbitmap).
- bg_inode_bitmap_csum_lo / _hi (u16+u16): per-group ibitmap CRC.
- bg_itable_unused_lo / _hi (u16+u16): per-group #unused-inode-table-entries.
- bg_checksum (u16): per-group crc16(sb_uuid + group + desc) (descriptor self-CRC; superseded by per-bitmap CRCs).
- bg_reserved (u32): padding to 64 bytes.

REQ-2: ext4_get_group_number(sb, block):
- If STD_GROUP_SIZE option set, fast-path: shift-only — `group = (block − s_first_data_block) >> (BLOCK_SIZE_BITS + CLUSTER_BITS + 3)`.
- Else: delegate to ext4_get_group_no_and_offset.

REQ-3: ext4_get_group_no_and_offset(sb, blocknr, *blockgrpp, *offsetp):
- blocknr -= s_first_data_block.
- offset = (blocknr mod EXT4_BLOCKS_PER_GROUP(sb)) >> s_cluster_bits.
- blocknr /= EXT4_BLOCKS_PER_GROUP(sb).
- *blockgrpp = blocknr; *offsetp = offset (each NULL-tolerant).

REQ-4: ext4_block_in_group(sb, block, group) -> bool:
- ext4_get_group_number(sb, block) == group.

REQ-5: ext4_num_overhead_clusters(sb, group, gdp):
- Start with base_clusters = ext4_num_base_meta_clusters (SB + GDT + reserved_gdt_blocks if applicable).
- If inode-table cluster-range overlaps group [start, end]:
  - num_clusters += (itbl_cluster_end − itbl_cluster_start + 1).
  - Subtract 1 if itbl_cluster_start coincides with the border cluster of base_clusters (overlap dedup).
- For block-bitmap cluster within group (FLEX_BG may move it out): +1 unless already counted.
- For inode-bitmap cluster within group (FLEX_BG may move it out): +1 unless already counted (and != block_cluster).
- Returns total #metadata-clusters consumed in this group.

REQ-6: ext4_init_block_bitmap(sb, bh, group, gdp):
- ASSERT(buffer_locked(bh)).
- If !ext4_group_desc_csum_verify(sb, group, gdp): mark group BBITMAP|IBITMAP_CORRUPT; return -EFSBADCRC.
- memset(bh.b_data, 0, sb.s_blocksize).
- bit_max = ext4_num_base_meta_clusters(sb, group).
- If (bit_max >> 3) >= bh.b_size: return -EFSCORRUPTED.
- For bit in 0..bit_max: ext4_set_bit(bit, bh.b_data) — pre-mark SB + GDT + reserved-GDT clusters.
- Set bits for block-bitmap, inode-bitmap, and inode-table-clusters that fall inside this group (FLEX_BG may externalize them).
- ext4_mark_bitmap_end(num_clusters_in_group(sb, group), sb.s_blocksize * 8, bh.b_data) — set tail-padding to 1 when group is smaller than bitmap.

REQ-7: ext4_free_clusters_after_init(sb, group, gdp):
- Return num_clusters_in_group(sb, group) − ext4_num_overhead_clusters(sb, group, gdp).
- Used when group is `EXT4_BG_BLOCK_UNINIT` so the count cannot be derived from a bitmap that has not been written.

REQ-8: ext4_get_group_desc(sb, group, *bh):
- ngroups = ext4_get_groups_count(sb).
- If group >= ngroups: ext4_error; return NULL.
- group_desc = group >> EXT4_DESC_PER_BLOCK_BITS(sb).
- offset = group & (EXT4_DESC_PER_BLOCK(sb) − 1).
- bh_p = sbi_array_rcu_deref(sbi, s_group_desc, group_desc).
- desc = (ext4_group_desc *)(bh_p.b_data + offset * EXT4_DESC_SIZE(sb)).
- if (bh) *bh = bh_p; return desc.

REQ-9: ext4_valid_block_bitmap_padding(sb, group, bh):
- If FLEX_BG: skip — bitmap may live outside group.
- bitmap_size = sb.s_blocksize * 8.
- offset = num_clusters_in_group(sb, group).
- If bitmap_size > offset: next_zero_bit = ext4_find_next_zero_bit(bh.b_data, bitmap_size, offset); return non-zero if a stray zero was found in the tail-padding region.

REQ-10: ext4_valid_block_bitmap(sb, desc, group, bh):
- If FLEX_BG: return 0 (skip — bitmap may live outside the group).
- group_first_block = ext4_group_first_block_no(sb, group).
- Verify bit set at: block-bitmap cluster, inode-bitmap cluster, inode-table-cluster-range.
- Each offset is checked: 0 <= offset and EXT4_B2C(offset) < max_bit and ext4_test_bit(EXT4_B2C(offset), bh.b_data); fail returns the offending blocknr.
- Verify no zero bit in inode-table-cluster-range via ext4_find_next_zero_bit.

REQ-11: ext4_validate_block_bitmap(sb, desc, group, bh):
- Skip if EXT4_FC_REPLAY.
- grp = ext4_get_group_info(sb, group); if !grp or EXT4_MB_GRP_BBITMAP_CORRUPT(grp): return -EFSCORRUPTED.
- If buffer_verified(bh): return 0.
- Acquire ext4_lock_group; re-check buffer_verified.
- Call ext4_block_bitmap_csum_verify (and ext4_simulate_fail(EXT4_SIM_BBITMAP_CRC) for fault-injection). On mismatch: unlock, ext4_error, mark BBITMAP_CORRUPT, return -EFSBADCRC.
- Call ext4_valid_block_bitmap; on mismatch: mark BBITMAP_CORRUPT, return -EFSCORRUPTED.
- Call ext4_valid_block_bitmap_padding; on mismatch: same.
- set_buffer_verified(bh); release group-lock; return 0.

REQ-12: ext4_read_block_bitmap_nowait(sb, group, ignore_locked):
- desc = ext4_get_group_desc(sb, group, NULL); !desc → ERR_PTR(-EFSCORRUPTED).
- bitmap_blk = ext4_block_bitmap(sb, desc) (combines bg_block_bitmap_lo and _hi).
- Bound-check: bitmap_blk > s_first_data_block and < ext4_blocks_count(es).
- bh = sb_getblk(sb, bitmap_blk); !bh → ERR_PTR(-ENOMEM).
- If ignore_locked and buffer_locked(bh): put_bh; return NULL (prefetch path).
- If bitmap_uptodate(bh): goto verify.
- lock_buffer(bh). If bitmap_uptodate now: goto verify.
- Group-lock. If has-group-desc-csum and desc.bg_flags & EXT4_BG_BLOCK_UNINIT:
  - For group == 0: this is illegal — ext4_error; return -EFSCORRUPTED.
  - ext4_init_block_bitmap(sb, bh, group, desc); set_bitmap_uptodate; set_buffer_uptodate; set_buffer_verified; release locks; return bh.
- Else: if buffer_uptodate(bh) (raced): goto verify; else submit ext4_read_bh_nowait with REQ_META|REQ_PRIO (+REQ_RAHEAD if ignore_locked), completion ext4_end_bitmap_read, ext4_simulate_fail EXT4_SIM_BBITMAP_EIO.
- Return bh (caller waits separately).

REQ-13: ext4_wait_block_bitmap(sb, group, bh):
- If !buffer_new(bh): return 0 (uptodate from prior read).
- wait_on_buffer(bh); if !buffer_uptodate: ext4_error_err(EIO); mark BBITMAP_CORRUPT; return -EIO.
- clear_buffer_new(bh); return ext4_validate_block_bitmap(sb, desc, group, bh).

REQ-14: ext4_read_block_bitmap(sb, group) — sync wrapper:
- bh = ext4_read_block_bitmap_nowait(sb, group, false). IS_ERR → return.
- err = ext4_wait_block_bitmap. err → put_bh; return ERR_PTR(err).
- Return bh.

REQ-15: ext4_has_free_clusters(sbi, nclusters, flags):
- free_clusters = percpu_counter_read_positive(s_freeclusters_counter).
- dirty_clusters = percpu_counter_read_positive(s_dirtyclusters_counter).
- resv_clusters = atomic64_read(s_resv_clusters).
- rsv = (ext4_r_blocks_count(es) >> s_cluster_bits) + resv_clusters.
- If close to watermark (`EXT4_FREECLUSTERS_WATERMARK`): refresh with percpu_counter_sum_positive (slow precise).
- If free_clusters >= rsv + nclusters + dirty_clusters: return 1.
- Else if current matches resuid / resgid / EXT4_MB_USE_ROOT_BLOCKS / CAP_SYS_RESOURCE:
  - If free_clusters >= nclusters + dirty_clusters + resv_clusters: return 1.
- Else if flags & EXT4_MB_USE_RESERVED:
  - If free_clusters >= nclusters + dirty_clusters: return 1.
- Else: return 0.

REQ-16: ext4_claim_free_clusters(sbi, nclusters, flags):
- If ext4_has_free_clusters: percpu_counter_add(s_dirtyclusters_counter, nclusters); return 0.
- Else: return -ENOSPC.

REQ-17: ext4_should_retry_alloc(sb, *retries):
- !s_journal: return 0.
- ++*retries > 3: percpu_counter_inc(s_sra_exceeded_retry_limit); return 0.
- smp_mb. If s_mb_free_pending == 0:
  - if DISCARD: atomic_inc(s_retry_alloc_pending); flush_work(s_discard_work); atomic_dec.
  - return ext4_has_free_clusters(sbi, 1, 0).
- ext4_debug; jbd2_journal_force_commit_nested(s_journal); return 1.

REQ-18: ext4_free_blocks bitmap-update path (defined in mballoc.c, consumed by callers throughout ext4 — extents.c, indirect.c, migrate.c, xattr.c):
- Per-(handle, inode, bh-of-block-being-freed-if-known, block, count, flags).
- Discovers per-(group, offset) via ext4_get_group_no_and_offset.
- Acquires per-bitmap buffer via ext4_read_block_bitmap.
- ext4_lock_group; clears bits via ext4_mb_unload_buddy + mb_free_blocks (mballoc-side) — semantically equivalent to per-bit `mb_clear_bits` on the bitmap.
- Updates bg_free_blocks_count, recomputes bg_block_bitmap_csum via ext4_block_bitmap_csum_set.
- ext4_mb_clear_bb / ext4_group_desc_csum_set; mark group-desc-buffer dirty in jbd2.
- percpu_counter_add(s_freeclusters_counter, freed); percpu_counter_sub(s_dirtyclusters_counter) if previously reserved.
- ext4_unlock_group; if metadata: jbd2_journal_get_write_access + jbd2_journal_dirty_metadata on the bitmap-bh and group-desc-bh.

REQ-19: ext4_count_free_clusters(sb):
- desc_count = 0.
- For each group i in 0..ngroups:
  - gdp = ext4_get_group_desc(sb, i, NULL); !gdp → continue.
  - grp = ext4_get_group_info(sb, i) (if s_group_info initialized).
  - If !grp or !EXT4_MB_GRP_BBITMAP_CORRUPT(grp): desc_count += ext4_free_group_clusters(sb, gdp) (combines bg_free_blocks_count_lo + _hi).
- Under EXT4FS_DEBUG: additionally read each block-bitmap and ext4_count_free(bh.b_data, EXT4_CLUSTERS_PER_GROUP/8); compare with cached count and printk discrepancies. Return bitmap_count in debug; desc_count otherwise.

REQ-20: ext4_count_free(bitmap, numchars) [fs/ext4/bitmap.c]:
- return numchars * BITS_PER_BYTE − memweight(bitmap, numchars). Used by ialloc/balloc/super for debug accounting.

REQ-21: ext4_block_bitmap_csum_verify / _set [fs/ext4/bitmap.c]:
- If !ext4_has_feature_metadata_csum(sb): verify returns 1; set returns immediately.
- sz = EXT4_CLUSTERS_PER_GROUP(sb) / 8.
- csum = ext4_chksum(s_csum_seed, bh.b_data, sz).
- gdp.bg_block_bitmap_csum_lo = csum & 0xFFFF; if s_desc_size >= EXT4_BG_BLOCK_BITMAP_CSUM_HI_END: gdp.bg_block_bitmap_csum_hi = csum >> 16; else compare against low 16 bits only.

REQ-22: ext4_bg_has_super(sb, group) -> 0|1:
- group == 0: 1.
- ext4_has_feature_sparse_super2(sb): 1 iff group == s_backup_bgs[0|1].
- Else (sparse_super): 1 iff group<=1 or group is odd power of 3/5/7 (test_root).
- Else: 1 (every group has SB).

REQ-23: ext4_bg_num_gdb(sb, group):
- Without META_BG (or group is in non-meta range): ext4_bg_num_gdb_nometa — if bg has SB: s_first_meta_bg (under META_BG) or s_gdb_count.
- META_BG meta range: ext4_bg_num_gdb_meta — 1 GDT block per (first, first+1, last) within metagroup.

REQ-24: ext4_num_base_meta_blocks(sb, group):
- num = ext4_bg_has_super(sb, group).
- If !META_BG or group < first_meta_bg * desc_per_block:
  - if num: num += ext4_bg_num_gdb_nometa + s_reserved_gdt_blocks (preserves growable-fs slack).
- Else (META_BG range): num += ext4_bg_num_gdb_meta.
- ext4_num_base_meta_clusters = EXT4_NUM_B2C(sbi, ext4_num_base_meta_blocks).

REQ-25: s_reserved_gdt_blocks (super_block s_es field):
- u16 per-group reservation of GDT-slack blocks for online-resize (`resize2fs`); included into base-meta block-count for groups that hold superblock backups.
- Capped at sb.s_blocksize / 4 (super.c bound).
- Decremented one-by-one as resize consumes slots (resize.c).

REQ-26: ext4_inode_to_goal_block(inode):
- ei = EXT4_I(inode); block_group = ei.i_block_group.
- If flex_size >= EXT4_FLEX_SIZE_DIR_ALLOC_SCHEME: snap to flex-group base; regular files shift to bg+1, directories stay at flex base (improves fsck locality).
- bg_start = ext4_group_first_block_no(sb, block_group); last_block = ext4_blocks_count(es) − 1.
- If DELALLOC: return bg_start (no colour).
- Else: colour = (current.pid mod 16) * (per-group-blocks / 16) (or shrunk for short last group); return bg_start + colour.

## Acceptance Criteria

- [ ] AC-1: ext4_get_group_no_and_offset round-trips with ext4_group_first_block_no for arbitrary blocknr inside FS bounds.
- [ ] AC-2: ext4_read_block_bitmap on a freshly-mounted FS for an EXT4_BG_BLOCK_UNINIT group returns a buffer with SB+GDT+itable bits set and tail-padding set; sets buffer_verified.
- [ ] AC-3: ext4_read_block_bitmap on group 0 with EXT4_BG_BLOCK_UNINIT returns -EFSCORRUPTED (illegal state).
- [ ] AC-4: ext4_validate_block_bitmap returns -EFSBADCRC when bg_block_bitmap_csum mismatches; marks group EXT4_GROUP_INFO_BBITMAP_CORRUPT.
- [ ] AC-5: ext4_validate_block_bitmap returns -EFSCORRUPTED when SB/iBM/itable bits are not set in bitmap (non-FLEX_BG).
- [ ] AC-6: ext4_has_free_clusters honours resuid/resgid/CAP_SYS_RESOURCE/EXT4_MB_USE_RESERVED dipping rules.
- [ ] AC-7: ext4_claim_free_clusters on ENOSPC does NOT modify s_dirtyclusters_counter.
- [ ] AC-8: ext4_should_retry_alloc returns 0 after 3 retries; otherwise forces jbd2_journal_force_commit_nested.
- [ ] AC-9: ext4_count_free_clusters non-debug build sums bg_free_blocks_count over all groups, skipping BBITMAP_CORRUPT groups.
- [ ] AC-10: ext4_count_free(bitmap, n) == n * 8 − popcount(bitmap[0..n)).
- [ ] AC-11: ext4_block_bitmap_csum_set then _verify round-trips for the same (s_uuid+group+bh.b_data) sequence with and without 64-bit desc.
- [ ] AC-12: ext4_bg_has_super(group) matches `mke2fs -E sparse_super`/`sparse_super2`/`!sparse_super` expected backups.
- [ ] AC-13: ext4_num_base_meta_blocks(sb, 0) == 1 (SB) + bg_num_gdb_nometa(0) + s_reserved_gdt_blocks for legacy non-META_BG ext4.
- [ ] AC-14: ext4_free_blocks bitmap-update on free-of-allocated-cluster: bitmap-bit-cleared, bg_free_blocks_count incremented, bg_block_bitmap_csum updated, jbd2-journal records buffer dirty.
- [ ] AC-15: ext4_inode_to_goal_block returns bg_start when DELALLOC mounted regardless of pid.

## Architecture

```
struct Ext4GroupDesc {
  bg_block_bitmap_lo: __le32,
  bg_inode_bitmap_lo: __le32,
  bg_inode_table_lo: __le32,
  bg_free_blocks_count_lo: __le16,
  bg_free_inodes_count_lo: __le16,
  bg_used_dirs_count_lo: __le16,
  bg_flags: __le16,                    // EXT4_BG_INODE_UNINIT | _BLOCK_UNINIT | _INODE_ZEROED
  bg_exclude_bitmap_lo: __le32,
  bg_block_bitmap_csum_lo: __le16,
  bg_inode_bitmap_csum_lo: __le16,
  bg_itable_unused_lo: __le16,
  bg_checksum: __le16,
  // 64BIT feature extends:
  bg_block_bitmap_hi: __le32,
  bg_inode_bitmap_hi: __le32,
  bg_inode_table_hi: __le32,
  bg_free_blocks_count_hi: __le16,
  bg_free_inodes_count_hi: __le16,
  bg_used_dirs_count_hi: __le16,
  bg_itable_unused_hi: __le16,
  bg_exclude_bitmap_hi: __le32,
  bg_block_bitmap_csum_hi: __le16,
  bg_inode_bitmap_csum_hi: __le16,
  bg_reserved: u32,
}
```

`Balloc::group_no_and_offset(sb, blocknr) -> (group, offset)`:
1. blocknr -= le32(es.s_first_data_block).
2. q = blocknr / EXT4_BLOCKS_PER_GROUP(sb).
3. r = blocknr % EXT4_BLOCKS_PER_GROUP(sb).
4. offset = r >> sbi.s_cluster_bits.
5. return (q as ext4_group_t, offset as ext4_grpblk_t).

`Balloc::read_block_bitmap_nowait(sb, group, ignore_locked) -> Result<BufferHead>`:
1. desc = Balloc::get_group_desc(sb, group)?  // -EFSCORRUPTED on miss
2. bitmap_blk = ext4_block_bitmap(sb, desc).
3. /* Bound-check */
4. if bitmap_blk <= s_first_data_block ∨ bitmap_blk >= ext4_blocks_count(es):
   - mark BBITMAP_CORRUPT; return Err(EFSCORRUPTED).
5. bh = sb_getblk(sb, bitmap_blk)?  // -ENOMEM
6. if ignore_locked ∧ buffer_locked(bh): put_bh; return Ok(None).
7. if bitmap_uptodate(bh): goto verify.
8. lock_buffer(bh); if bitmap_uptodate after-lock: goto unlock_verify.
9. ext4_lock_group(sb, group).
10. if has_group_desc_csum ∧ (desc.bg_flags & EXT4_BG_BLOCK_UNINIT):
    - if group == 0: unlock+error; return Err(EFSCORRUPTED).
    - Balloc::init_block_bitmap(sb, bh, group, desc)? — pre-marks SB/GDT/iBM/iTBL bits and tail padding.
    - set_bitmap_uptodate; set_buffer_uptodate; set_buffer_verified.
    - unlock_group; unlock_buffer; return Ok(bh).
11. ext4_unlock_group(sb, group).
12. if buffer_uptodate(bh): set_bitmap_uptodate; unlock_buffer; goto verify.
13. set_buffer_new(bh); ext4_read_bh_nowait(bh, REQ_META|REQ_PRIO|RAHEAD?, end_bitmap_read, simulate-EIO?).
14. return Ok(bh) (asynchronous; caller waits).
verify:
15. Balloc::validate_block_bitmap(sb, desc, group, bh)?; return Ok(bh).

`Balloc::validate_block_bitmap(sb, desc, group, bh)`:
1. if s_mount_state & EXT4_FC_REPLAY: return Ok.
2. grp = Balloc::get_group_info(sb, group); if !grp ∨ BBITMAP_CORRUPT(grp): return Err(EFSCORRUPTED).
3. if buffer_verified(bh): return Ok.
4. ext4_lock_group(sb, group).
5. if buffer_verified now: goto verified.
6. if !Balloc::bbitmap_csum_verify(sb, desc, bh) ∨ simulate_fail(BBITMAP_CRC):
   - unlock; mark BBITMAP_CORRUPT; return Err(EFSBADCRC).
7. blk = Balloc::valid_block_bitmap(sb, desc, group, bh); blk != 0:
   - unlock; mark BBITMAP_CORRUPT; return Err(EFSCORRUPTED).
8. blk = Balloc::valid_block_bitmap_padding(sb, group, bh); blk != 0:
   - unlock; mark BBITMAP_CORRUPT; return Err(EFSCORRUPTED).
9. set_buffer_verified(bh).
verified:
10. ext4_unlock_group(sb, group); return Ok.

`Balloc::has_free_clusters(sbi, nclusters, flags) -> bool`:
1. free = percpu_counter_read_positive(s_freeclusters_counter).
2. dirty = percpu_counter_read_positive(s_dirtyclusters_counter).
3. resv = atomic64_read(s_resv_clusters).
4. rsv = (ext4_r_blocks_count(es) >> s_cluster_bits) + resv.
5. /* Watermark refresh: spend slow precise read if close */
6. if free − (nclusters + rsv + dirty) < EXT4_FREECLUSTERS_WATERMARK:
   - free = percpu_counter_sum_positive(s_freeclusters_counter).
   - dirty = percpu_counter_sum_positive(s_dirtyclusters_counter).
7. if free >= rsv + nclusters + dirty: return true.
8. /* Root-pool dip */
9. if uid_eq(s_resuid, current_fsuid) ∨ in_group_p(s_resgid) ∨ (flags & USE_ROOT_BLOCKS) ∨ capable(CAP_SYS_RESOURCE):
   - if free >= nclusters + dirty + resv: return true.
10. /* Reserved-pool dip */
11. if flags & USE_RESERVED:
    - if free >= nclusters + dirty: return true.
12. return false.

`Balloc::should_retry_alloc(sb, *retries) -> bool`:
1. if !s_journal: return false.
2. *retries += 1; if *retries > 3: percpu_counter_inc(s_sra_exceeded_retry_limit); return false.
3. smp_mb.
4. if atomic_read(s_mb_free_pending) == 0:
   - if DISCARD: atomic_inc(s_retry_alloc_pending); flush_work(s_discard_work); atomic_dec.
   - return Balloc::has_free_clusters(sbi, 1, 0).
5. jbd2_journal_force_commit_nested(s_journal).
6. return true.

`Balloc::free_blocks_bitmap_update(handle, inode, bh, block, count, flags)`:
1. /* Decompose */
2. (group, offset) = Balloc::group_no_and_offset(sb, block).
3. bbh = Balloc::read_block_bitmap(sb, group)?
4. /* Per-bg lock + journaling */
5. ext4_lock_group(sb, group).
6. jbd2_journal_get_write_access(handle, bbh).
7. for k in 0..count: mb_clear_bit(EXT4_B2C(sbi, offset + k), bbh.b_data).
8. /* Update cached free count */
9. ext4_group_desc_free_blocks_count_add(gdp, count).
10. /* Re-CRC */
11. Balloc::bbitmap_csum_set(sb, gdp, bbh).
12. /* Re-CRC group-desc */
13. ext4_group_desc_csum_set(sb, group, gdp).
14. ext4_unlock_group(sb, group).
15. jbd2_journal_dirty_metadata(handle, bbh).
16. /* gd-bh dirty for desc-update */
17. jbd2_journal_dirty_metadata(handle, sbi.s_group_desc[group_desc_idx]).
18. /* Counters */
19. percpu_counter_add(s_freeclusters_counter, count_clusters).
20. percpu_counter_sub(s_dirtyclusters_counter, count_clusters_was_reserved).
21. trace_ext4_mballoc_free(...).

`Balloc::count_free_clusters(sb) -> u64`:
1. ngroups = ext4_get_groups_count(sb).
2. acc = 0.
3. for i in 0..ngroups:
   - gdp = Balloc::get_group_desc(sb, i)?.
   - grp = Balloc::get_group_info(sb, i).
   - if !grp ∨ !EXT4_MB_GRP_BBITMAP_CORRUPT(grp):
     - acc += ext4_free_group_clusters(sb, gdp).
4. /* Debug variant additionally cross-checks via ext4_count_free(bitmap) */
5. return acc.

`Balloc::inode_to_goal_block(inode) -> ext4_fsblk_t`:
1. ei = EXT4_I(inode); block_group = ei.i_block_group.
2. flex_size = ext4_flex_bg_size(sbi).
3. if flex_size >= EXT4_FLEX_SIZE_DIR_ALLOC_SCHEME:
   - block_group &= ~(flex_size − 1).
   - if S_ISREG(inode.i_mode): block_group += 1.
4. bg_start = ext4_group_first_block_no(sb, block_group).
5. last_block = ext4_blocks_count(es) − 1.
6. if test_opt(sb, DELALLOC): return bg_start.
7. if bg_start + EXT4_BLOCKS_PER_GROUP(sb) <= last_block:
   - colour = (current.pid % 16) * (EXT4_BLOCKS_PER_GROUP(sb) / 16).
8. else:
   - colour = (current.pid % 16) * ((last_block − bg_start) / 16).
9. return bg_start + colour.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `group_no_and_offset_inverse_first_block_no` | INVARIANT | for blocknr ∈ [s_first_data_block, ext4_blocks_count(es)): ext4_group_first_block_no(sb, group) + offset * sbi.s_cluster_ratio ≤ blocknr < next-group. |
| `bitmap_bound_check` | INVARIANT | per-read_block_bitmap_nowait: bitmap_blk > s_first_data_block ∧ bitmap_blk < ext4_blocks_count(es) ⟹ proceed; else EFSCORRUPTED. |
| `group0_never_uninit` | INVARIANT | per-read_block_bitmap_nowait: group == 0 with BLOCK_UNINIT ⟹ EFSCORRUPTED. |
| `bbitmap_csum_round_trip` | INVARIANT | per-(s_uuid, group, bh.b_data): csum_set followed by csum_verify returns 1. |
| `count_free_popcount_complement` | INVARIANT | per-bitmap: ext4_count_free(b, n) + popcount(b[0..n)) == n * 8. |
| `claim_free_no_partial` | INVARIANT | per-claim_free_clusters: ENOSPC ⟹ s_dirtyclusters_counter unchanged. |
| `should_retry_bounded` | INVARIANT | per-should_retry_alloc: *retries monotonic; >3 ⟹ false. |
| `validate_under_group_lock` | INVARIANT | per-validate_block_bitmap: bbitmap_csum_verify and valid_block_bitmap called with ext4_lock_group held. |
| `init_block_bitmap_locked_buffer` | INVARIANT | per-init_block_bitmap: ASSERT(buffer_locked(bh)). |
| `free_blocks_csum_update` | INVARIANT | per-free_blocks bitmap-update: bg_block_bitmap_csum recomputed before unlock_group. |

### Layer 2: TLA+

`fs/ext4/balloc.tla`:
- States: per-group {UNINIT, INIT_INFLIGHT, READY, CORRUPT}.
- Transitions: read_block_bitmap_nowait, init_block_bitmap, validate_block_bitmap, free_blocks-bitmap-update.
- Properties:
  - `safety_uninit_group0_unreachable` — per-group-0 never enters UNINIT-init-path.
  - `safety_csum_monotone` — per-group: every transition that mutates bitmap-data also recomputes bg_block_bitmap_csum.
  - `safety_no_double_count` — per-claim/free: s_freeclusters_counter conservative (sum across groups ≤ total clusters).
  - `safety_root_pool_only_for_privileged` — per-has_free_clusters: dip only allowed under resuid/resgid/cap.
  - `liveness_should_retry_terminates` — per-should_retry_alloc: bounded by 3 retries.
  - `liveness_validate_terminates` — per-validate_block_bitmap: under lock, terminates in O(group-size).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Balloc::group_no_and_offset` post: offset < EXT4_CLUSTERS_PER_GROUP(sb) ∧ group < ext4_get_groups_count(sb) | `Balloc::group_no_and_offset` |
| `Balloc::read_block_bitmap_nowait` post: success ⟹ bitmap_uptodate(bh) ∧ (UNINIT-path ⟹ buffer_verified) | `Balloc::read_block_bitmap_nowait` |
| `Balloc::validate_block_bitmap` post: Ok ⟹ buffer_verified(bh); Err ⟹ EXT4_MB_GRP_BBITMAP_CORRUPT(grp) | `Balloc::validate_block_bitmap` |
| `Balloc::has_free_clusters` post: true ⟹ ∃ pool selection in {default, root, reserved} | `Balloc::has_free_clusters` |
| `Balloc::claim_free_clusters` post: 0 ⟹ s_dirtyclusters_counter += nclusters; ENOSPC ⟹ unchanged | `Balloc::claim_free_clusters` |
| `Balloc::should_retry_alloc` post: returned-false ⟹ *retries ∈ {>3, mb_free_pending==0 ∧ !has_free} | `Balloc::should_retry_alloc` |
| `Balloc::free_blocks_bitmap_update` post: bitmap-bits cleared; bg_free_blocks_count += count; bg_block_bitmap_csum re-set; bbh+gdbh journaled | `Balloc::free_blocks_bitmap_update` |
| `Balloc::count_free_clusters` post: returns Σ ext4_free_group_clusters over non-corrupt groups | `Balloc::count_free_clusters` |
| `Balloc::inode_to_goal_block` post: result ∈ [bg_start, bg_start + EXT4_BLOCKS_PER_GROUP] | `Balloc::inode_to_goal_block` |
| `Balloc::init_block_bitmap` post: bits 0..ext4_num_base_meta_clusters set; bits for in-group iBM/iTBL set; tail-padding set | `Balloc::init_block_bitmap` |

### Layer 4: Verus/Creusot functional

`Per-group bitmap lifecycle: get_group_desc → read_block_bitmap (uninit ⟹ init_block_bitmap else submit-IO + wait) → validate_block_bitmap (csum + sanity + padding) → consume in mballoc/free_blocks → free_blocks-bitmap-update (re-CRC + journal)` semantic equivalence: per-Documentation/filesystems/ext4/blockgroup.rst + per-Documentation/filesystems/ext4/group_descr.rst.

## Hardening

(Inherits row-1 features from `fs/ext4/00-overview.md` § Hardening.)

Balloc-specific reinforcement:

- **Per-bitmap CRC verify on every read** — defense against per-disk-bit-rot / per-untrusted-device.
- **Per-EXT4_BG_BLOCK_UNINIT for group 0 rejected** — defense against per-superblock-craft attempting to skip SB/GDT pre-marks.
- **Per-group-bitmap bound-check (s_first_data_block < bitmap_blk < ext4_blocks_count)** — defense against per-malformed-gdp pointing bitmap inside SB or beyond device.
- **Per-EXT4_MB_GRP_BBITMAP_CORRUPT sticky** — defense against per-retry-reuse of known-bad bitmap.
- **Per-padding-of-bitmap-must-be-1 invariant** — defense against per-crafted-bitmap implying bogus free clusters past num_clusters_in_group.
- **Per-validate_block_bitmap under ext4_lock_group** — defense against per-concurrent free/alloc racing validation.
- **Per-r_blocks reservation enforced (resuid/resgid/CAP_SYS_RESOURCE/USE_RESERVED gates)** — defense against per-unprivileged exhaustion of root-pool.
- **Per-EXT4_FREECLUSTERS_WATERMARK precise refresh** — defense against per-percpu-skew false-success near ENOSPC.
- **Per-should_retry_alloc bounded to 3** — defense against per-livelock on persistent ENOSPC.
- **Per-init_block_bitmap requires buffer_locked(bh)** — defense against per-concurrent-read seeing half-initialized bitmap.
- **Per-free_blocks-bitmap-update re-CRCs before unlock_group** — defense against per-crash-window with stale csum on disk.
- **Per-fault-injection EXT4_SIM_BBITMAP_CRC / _EIO honored** — defense against per-fragile fault-handling paths.
- **Per-FLEX_BG bitmap-validation skip preserves correctness** — defense against per-flex-bg false-positive corruption.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on `EXT4_IOC_*` ioctls that surface block-bitmap stats.
- **PAX_KERNEXEC** — W^X for any executable mapping reachable from balloc.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization across `ext4_mb_*` entry points called from `write` / `fallocate`.
- **PAX_REFCOUNT** — saturating refcount on `struct ext4_group_info`, on the `buffer_head` of every bitmap, and on `s_freeclusters_counter` so percpu drift cannot wrap.
- **PAX_MEMORY_SANITIZE** — zero-on-free for the freed group-info slab and bitmap scratch pages.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access on the ioctl surface.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on the journal callback (`ext4_mb_release_blocks_on_commit`).
- **GRKERNSEC_HIDESYM** — kernel pointer hiding in `/proc/fs/ext4/<dev>/mb_groups`.
- **GRKERNSEC_DMESG** — syslog restriction on bitmap-corruption diagnostics that otherwise leak block-group geometry.
- **Bitmap CRC verification** — `ext4_validate_block_bitmap` recomputes the CRC32C over the on-disk bitmap before any allocation decision; mismatch flags the group `EXT4_GROUP_INFO_BBITMAP_CORRUPT` and excludes it.
- **CAP_SYS_RESOURCE on root-pool dipping** — `ext4_has_free_clusters` allows allocations into `s_r_blocks_count` only for `CAP_SYS_RESOURCE` or for the explicit fs-owning uid (`s_resuid`/`s_resgid`), so unprivileged tasks cannot exhaust emergency reserve.
- **FLEX_BG bitmap-validation skip is bounded** — when a flex-group's metadata is colocated, the per-leader CRC is still verified; only the trailing per-group CRC is skipped.
- **`init_block_bitmap` under `buffer_locked(bh)`** — lazy-init path holds the buffer lock so a concurrent reader cannot observe a half-initialized bitmap.
- **`should_retry_alloc` bounded to 3** — persistent ENOSPC cannot livelock the writeback path or pin CPU.
- **`EXT4_FREECLUSTERS_WATERMARK` precise refresh** — near-ENOSPC the percpu cache is forcibly drained so false-success allocations cannot cross the reserve threshold.
- **Fault-injection `EXT4_SIM_BBITMAP_CRC` / `_EIO` honored** — corruption simulators are wired through the same refusal paths so verification covers the panic-free branch.

Per-doc rationale: the block allocator is the gatekeeper for every write in ext4; a single missed CRC verify, a single overflow of `s_freeclusters_counter`, or a single unprivileged dip into the root pool turns a quota leak into a remote-DoS or a corruption that persists across reboot. PaX/grsec reinforcement keeps allocation arithmetic saturating, keeps bitmap reads CRC-gated, and keeps the reserve pool capability-bound.

## Open Questions

- Should Rookery merge `balloc.c` and `bitmap.c` into a single `Balloc` module, or keep CRC helpers separate to mirror upstream file split?
- Should the lazy `init_block_bitmap` path defer to a background kthread (akin to lazy-itable-init) to avoid stalling first-touch allocations?

## Out of Scope

- fs/ext4/mballoc.c multi-block allocator (per-buddy + per-preallocation + per-discard) — covered in `mballoc.md` Tier-3.
- fs/ext4/ialloc.c inode-bitmap allocator (sibling path) — covered separately if expanded.
- fs/ext4/resize.c online resize (consumer of s_reserved_gdt_blocks) — covered separately if expanded.
- fs/ext4/extents.c extent-tree walk / split (consumer of ext4_free_blocks for shrink) — covered in `extents.md` Tier-3.
- jbd2 journaling internals — covered in `jbd2` Tier-3.
- Implementation code.
