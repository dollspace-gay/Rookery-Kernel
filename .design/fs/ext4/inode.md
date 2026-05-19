# Tier-3: fs/ext4/inode.c — ext4 inode operations (read/write/get/destroy/extent-mapping)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/ext4/00-overview.md
upstream-paths:
  - fs/ext4/inode.c (~6826 lines)
  - fs/ext4/ext4.h
  - fs/ext4/extents.c
  - fs/ext4/extents_status.c
-->

## Summary

ext4 inode-ops implement per-inode I/O: per-`ext4_iget` reads on-disk `struct ext4_inode` (160 bytes); per-`ext4_get_block` / `ext4_map_blocks` resolves logical-block to disk-block via extent-tree or indirect-block; per-write triggers writeback + jbd2-journaling. Per-`ext4_evict_inode` cleans inode on umount/orphan. Per-extent-status-tree caches per-(inode, lblk)→pblk mappings for fast lookup. Per-direct-IO + buffered-IO + per-DAX (Direct-Access for NVDIMM). Critical for: most file-I/O code-paths in ext4-backed systems.

This Tier-3 covers `inode.c` (~6826 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ext4_inode_info` | per-inode in-memory state | `Ext4InodeInfo` |
| `struct ext4_inode` | on-disk format (160 bytes) | `Ext4Inode` |
| `ext4_iget()` | per-(sb, inode_no) lookup | `Ext4::iget` |
| `ext4_inode_operations` | per-inode VFS ops | `Ext4::INODE_OPS` |
| `ext4_file_operations` | per-file VFS ops | `Ext4::FILE_OPS` |
| `ext4_write_inode()` | per-inode write-to-disk | `Ext4::write_inode` |
| `ext4_evict_inode()` | per-inode destroy | `Ext4::evict_inode` |
| `ext4_get_block()` | per-block-mapping (legacy bh-based) | `Ext4::get_block` |
| `ext4_map_blocks()` | per-extent-aware block-mapping | `Ext4::map_blocks` |
| `ext4_read_folio()` | per-page read | `Ext4::read_folio` |
| `ext4_writepages()` | per-pageblob writeback | `Ext4::writepages` |
| `ext4_dax_get_blocks()` | DAX-mode | `Ext4::dax_get_blocks` |
| `ext4_direct_IO_*()` | per-O_DIRECT | `Ext4::direct_io` |
| `ext4_alloc_da_blocks()` | per-delayed-alloc | `Ext4::alloc_da_blocks` |
| `ext4_truncate()` | per-shrink | `Ext4::truncate` |
| `ext4_setattr()` | per-chattr | `Ext4::setattr` |
| `EXT4_INODE_*` flags | per-inode-flag bits | UAPI |

## Compatibility contract

REQ-1: On-disk ext4_inode (160 bytes; extra space if i_extra_isize):
- i_mode (u16): file-type + permissions.
- i_uid_lo / _hi (u16+u16): UID.
- i_size_lo / _hi (u32+u32): file size.
- i_atime / _ctime / _mtime (u32 each): unix-epoch + nsec extension.
- i_dtime (u32): deletion-time.
- i_gid_lo / _hi (u16+u16): GID.
- i_links_count (u16): hardlink count.
- i_blocks_lo / _hi (u32+u16): #disk-blocks.
- i_flags (u32): EXT4_*_FL.
- i_block[15] (u32 each): per-block-pointer (extent-tree-header OR indirect-block-pointers).
- i_generation (u32): per-inode generation.
- i_file_acl_lo / _hi (u32+u16): extended-attr block.
- i_dir_acl (u32): legacy dir-ACL.
- i_faddr (u32): fragment-addr.
- i_extra_isize (u16): bytes of extra metadata after standard 160.
- i_csum (u16): per-inode CRC32C.
- i_projid (u32): project-id (if PROJECT enabled).

REQ-2: Per-EXT4_*_FL flags:
- EXT4_SECRM_FL: secure delete.
- EXT4_UNRM_FL: undelete.
- EXT4_COMPR_FL: compress.
- EXT4_SYNC_FL: sync writes.
- EXT4_IMMUTABLE_FL: immutable.
- EXT4_APPEND_FL: append-only.
- EXT4_NODUMP_FL: skip dump.
- EXT4_NOATIME_FL: no-atime.
- EXT4_EXTENTS_FL: uses extent-tree (vs indirect-blocks).
- EXT4_VERITY_FL: fs-verity.
- EXT4_ENCRYPT_FL: per-file encrypted.
- EXT4_INLINE_DATA_FL: per-small-file data inline in inode.
- EXT4_CASEFOLD_FL: per-dir case-folded.

REQ-3: ext4_iget(sb, ino, flags):
- Read inode-table block containing this inode.
- raw_inode = bh.b_data + offset.
- Validate via ext4_inode_csum_verify.
- Populate ext4_inode_info from raw_inode.
- Per-flags (EXT4_IGET_NORMAL / SPECIAL / HANDLE).

REQ-4: ext4_map_blocks(handle, inode, map, flags):
- map.m_lblk: logical-block.
- map.m_len: number of blocks to map.
- map.m_flags: result (NEW, MAPPED, BOUNDARY, UNWRITTEN, FROM_CLUSTER).
- map.m_pblk: physical-block result.
- if EXT4_EXTENTS_FL: ext4_ext_map_blocks (extent-tree walk).
- else: ext4_ind_map_blocks (indirect-blocks).

REQ-5: ext4_get_block (legacy):
- Wrapper around ext4_map_blocks for bh-based I/O.

REQ-6: Per-extent-tree:
- Inode i_block[0..4] = ext4_extent_header.
- Header: eh_magic + eh_entries + eh_max + eh_depth + eh_generation.
- Per-leaf: ext4_extent { ee_block, ee_len, ee_start_hi, ee_start_lo }.
- Per-index (depth > 0): ext4_extent_idx { ei_block, ei_leaf_lo, ei_leaf_hi }.

REQ-7: Per-extent-status tree:
- Per-inode RBTree of (lblk, len, pblk, status).
- Cache for fast lookup; invalidated on extent-mutate.

REQ-8: ext4_writepages:
- Per-dirty-pages collect into mpage_da_data.
- Per-extent-alloc per-block-cluster.
- Submit BIOs.

REQ-9: ext4_evict_inode:
- if !i_nlink (orphan): ext4_truncate to 0; ext4_free_inode.
- ext4_clear_inode (free in-memory).

REQ-10: Per-O_DIRECT:
- ext4_direct_IO_read / _write.
- Bypass page-cache.

REQ-11: Per-DAX:
- ext4_dax_get_blocks: direct-access from NVDIMM.
- No page-cache.

REQ-12: Per-encryption (ENCRYPT_FL):
- Per-folio crypto callback at read/write.

REQ-13: Per-verity (VERITY_FL):
- fs-verity Merkle-tree per-file.
- Per-read validates page hash.

## Acceptance Criteria

- [ ] AC-1: ext4_iget(sb, root_ino): reads inode-table; ext4_inode_info populated.
- [ ] AC-2: ext4_map_blocks(lblk=10, len=5): per-extent-tree or indirect-blocks resolved.
- [ ] AC-3: open + write 4KB: per-extent allocated; per-page dirty.
- [ ] AC-4: ext4_writepages: dirty-pages submitted via BIO.
- [ ] AC-5: unlink + iput: ext4_evict_inode frees blocks.
- [ ] AC-6: O_DIRECT write: bypasses page-cache.
- [ ] AC-7: DAX-mounted ext4: ext4_dax_get_blocks for mmap.
- [ ] AC-8: chattr +i: EXT4_IMMUTABLE_FL set; writes -EPERM.
- [ ] AC-9: chattr +e: EXT4_EXTENTS_FL set; new allocs via extent-tree.
- [ ] AC-10: fs-verity enabled file: per-read validates hash.
- [ ] AC-11: Per-inline-data: small file's data in i_block.

## Architecture

Per-inode_info:

```
struct Ext4InodeInfo {
  vfs_inode: Inode,                              // base VFS inode
  i_disksize: u64,
  i_es_tree: RbRoot,                              // extent-status tree
  i_es_lock: SpinLock,
  i_data: [u32; 15],                              // i_block (extent-header or indirect)
  i_flags: u32,
  i_dirty_data_max: u64,
  i_extra_isize: u16,
  i_csum_seed: u32,
  i_inline_data: Option<&[u8]>,
  jinode: *JBd2InodeInfo,
  fname: Option<&FileName>,                       // for casefold
  i_block_group: u32,
  vfs_inode: Inode,
  i_reserved_data_blocks: u64,
  ...
}
```

Per-on-disk inode (160 bytes minimum):

```
#[repr(C)]
struct Ext4Inode {
  i_mode: __le16,
  i_uid_lo: __le16,
  i_size_lo: __le32,
  i_atime: __le32,
  i_ctime: __le32,
  i_mtime: __le32,
  i_dtime: __le32,
  i_gid_lo: __le16,
  i_links_count: __le16,
  i_blocks_lo: __le32,
  i_flags: __le32,
  i_block: [__le32; 15],                          // EXT4_N_BLOCKS
  i_generation: __le32,
  i_file_acl_lo: __le32,
  i_size_hi: __le32,
  i_obso_faddr: __le32,
  i_blocks_high: __le16,
  i_file_acl_hi: __le16,
  i_uid_hi: __le16,
  i_gid_hi: __le16,
  i_csum_lo: __le16,
  i_extra_isize: __le16,
  i_csum_hi: __le16,
  i_ctime_extra: __le32,
  i_mtime_extra: __le32,
  i_atime_extra: __le32,
  i_crtime: __le32,
  i_crtime_extra: __le32,
  i_version_hi: __le32,
  i_projid: __le32,
}
```

`Ext4::iget(sb, ino, flags) -> Result<&Inode>`:
1. inode = iget_locked(sb, ino).
2. if !(inode.i_state & I_NEW): return inode (cached).
3. /* Read inode-table block */
4. block_group = (ino - 1) / inodes_per_group.
5. offset = ((ino - 1) % inodes_per_group) * inode_size.
6. block = group_inode_table_block + offset / blocksize.
7. bh = sb_bread(sb, block).
8. raw_inode = bh.b_data + offset % blocksize.
9. Ext4::inode_csum_verify(inode, raw_inode).
10. /* Populate from raw_inode */
11. inode.i_mode = le16_to_cpu(raw_inode.i_mode).
12. ext4_inode_info from raw_inode.i_block, flags, etc.
13. /* Per-i_flags */
14. /* Set i_op / i_fop */
15. if S_ISDIR(inode.i_mode): inode.i_op = &ext4_dir_inode_operations.
16. else if S_ISREG: inode.i_op = &ext4_file_inode_operations.
17. unlock_new_inode(inode).
18. Return inode.

`Ext4::map_blocks(handle, inode, map, flags) -> Result<()>`:
1. if EXT4_I(inode).i_flags & EXT4_EXTENTS_FL:
   - ret = ext4_ext_map_blocks(handle, inode, map, flags).
2. else:
   - ret = ext4_ind_map_blocks(handle, inode, map, flags).
3. if (flags & EXT4_GET_BLOCKS_CREATE) ∧ map.m_flags & EXT4_MAP_NEW:
   - mark new block-group bitmap.
   - extent-status-tree update.

`Ext4::write_inode(inode, wbc) -> Result<()>`:
1. ext4_iloc_get(inode, &iloc).
2. raw_inode = (Ext4Inode*)(iloc.bh.b_data + iloc.offset).
3. /* Copy in-memory fields back to raw_inode */
4. raw_inode.i_size_lo = inode.i_size & 0xFFFFFFFF.
5. raw_inode.i_atime = inode.i_atime.tv_sec.
6. ...
7. Compute checksum; raw_inode.i_csum_lo/hi = checksum.
8. mark_buffer_dirty(iloc.bh).

`Ext4::evict_inode(inode)`:
1. if !inode.i_nlink ∧ !is_bad_inode(inode):
   - ext4_orphan_add(handle, inode).
   - ext4_truncate(inode, 0).
   - ext4_free_inode(handle, inode).
2. ext4_clear_inode(inode).
3. clear_inode(inode).

`Ext4::writepages(mapping, wbc) -> Result<()>`:
1. /* Per-mpage walking */
2. mpd = mpage_da_data { inode, wbc, ... }.
3. /* Walk dirty folios; build extents */
4. mpage_prepare_extent_to_map(mpd).
5. /* Allocate per-extent blocks */
6. mpage_map_and_submit_buffers(mpd).
7. submit_bio per-extent.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `csum_validates_per_iget` | INVARIANT | post-iget: i_csum_lo/hi match computed. |
| `i_size_lo_hi_consistent` | INVARIANT | inode.i_size == (i_size_hi << 32) | i_size_lo. |
| `i_links_count_ge_zero` | INVARIANT | inode.i_links_count ≥ 0. |
| `extents_xor_indirect` | INVARIANT | EXTENTS_FL set xor !EXTENTS_FL (one mode). |
| `inline_data_le_inode_size` | INVARIANT | inline_data length ≤ inode_size - sizeof(metadata). |

### Layer 2: TLA+

`fs/ext4/inode.tla`:
- Per-iget + per-map_blocks + per-write_inode + per-evict.
- Properties:
  - `safety_no_double_evict` — per-inode evict_inode called once.
  - `safety_csum_persistent` — per-write_inode: post-disk csum matches.
  - `liveness_orphan_eventually_truncated` — per-i_nlink==0 evict: i_blocks → 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ext4::iget` post: inode populated; csum validated | `Ext4::iget` |
| `Ext4::map_blocks` post: per-extent or per-indirect lookup; flags set | `Ext4::map_blocks` |
| `Ext4::write_inode` post: raw_inode updated; csum recalculated | `Ext4::write_inode` |
| `Ext4::evict_inode` post: per-orphan freed; clear_inode called | `Ext4::evict_inode` |
| `Ext4::writepages` post: dirty-folios submitted; mapping clean | `Ext4::writepages` |

### Layer 4: Verus/Creusot functional

`Per-inode read-from-disk → in-memory state → write-back → on-disk; per-extent-or-indirect mapping for lblk→pblk` semantic equivalence: per-ext4 on-disk inode format.

## Hardening

(Inherits row-1 features from `fs/ext4/00-overview.md` § Hardening.)

ext4-inode reinforcement:

- **Per-inode csum CRC32C verified** — defense against per-disk-corruption.
- **Per-i_flags validated against EXT4_FL_USER_MODIFIABLE** — defense against per-malicious-flag.
- **Per-extents-or-indirect mode-check** — defense against per-tree confusion.
- **Per-immutable / append-only enforced** — defense against per-mutate.
- **Per-truncate journal-protected** — defense against per-crash-recovery inconsistent.
- **Per-orphan list managed** — defense against per-leak on unlink.
- **Per-inline-data size-cap** — defense against per-overflow.
- **Per-DAX bypasses page-cache** — defense against per-stale-pagecache.
- **Per-direct-IO bypasses pagecache** — defense against per-pagecache aliasing.
- **Per-encrypted-file per-folio decrypt** — defense against per-cache plaintext-leak.
- **Per-verity Merkle-tree per-read validate** — defense against per-content tampering.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on read/write paths and on every `getxattr`/`listxattr` egress.
- **PAX_KERNEXEC** — W^X for any executable mapping over an ext4-backed file.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization across `ext4_file_read_iter` / `ext4_buffered_write_iter`.
- **PAX_REFCOUNT** — saturating refcount on every `struct ext4_inode_info` and on the per-inode `i_es_tree` cache.
- **PAX_MEMORY_SANITIZE** — zero-on-free for the per-folio decrypt scratch buffer and for inline-data tail buffer on truncate.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access; iter copies use `iov_iter` helpers exclusively.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `address_space_operations` / `iomap_ops` ext4 vtables.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding in stat/inode-info ioctls.
- **GRKERNSEC_DMESG** — syslog restriction on inode-corruption diagnostics.
- **Extent vs indirect dispatch** — `ext4_map_blocks` consults `EXT4_INODE_EXTENTS` flag exactly once per call and pins the chosen back-end under `i_data_sem`; the two block-mapping back-ends are never multiplexed on a single inode within one syscall.
- **fs-verity integration** — Merkle-tree page reads are mandatory on every read of an `EXT4_VERITY_FL` inode; a mismatched root or interior hash forces `-EIO` to userspace.
- **fs-crypto integration** — per-folio decrypt happens before `mark_buffer_uptodate`; encrypted plaintext never enters the page cache as uptodate without a passing decrypt.
- **Inline-data size-cap** — `EXT4_MAX_INLINE_DATA` (60 bytes default, extended via xattr) is enforced on write so an inline tail cannot scribble into the next on-disk inode.
- **Orphan-list managed under journal** — `ext4_orphan_add`/`del` are journal-atomic so unlink-while-open can never leak blocks across crash.
- **DAX/direct-IO/page-cache mutual exclusion** — `S_DAX` is set/cleared only at lookup time and `O_DIRECT` writes bypass page cache atomically under `i_rwsem`.
- **CAP_SYS_RESOURCE on root-pool dipping** — reserved-block consumption gated through `ext4_has_free_clusters`.

Per-doc rationale: every regular-file read/write/truncate/fallocate in ext4 lands here; the inode dispatch picks between extent and indirect block mapping, between page-cache and DAX, between encrypted and plaintext, and between verity-checked and raw. PaX/grsec reinforcement holds the dispatch invariants under refcount/USERCOPY/RAP so a single mis-flag (e.g., `EXT4_INODE_EXTENTS` toggled mid-call) cannot let one back-end read state owned by the other.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/ext4/{extents, extents_status, file, namei, dir, balloc, ialloc, mballoc, ...}.c (covered separately if expanded)
- fs-crypto / fs-verity (covered separately)
- jbd2 (covered in `fs/jbd2.md` separately)
- Implementation code
