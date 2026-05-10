# Tier-3: fs/erofs/super.c — EROFS superblock + mount

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/erofs/00-overview.md
upstream-paths:
  - fs/erofs/super.c (~1138 lines)
  - fs/erofs/erofs_fs.h (on-disk struct erofs_super_block)
  - fs/erofs/internal.h (struct erofs_sb_info)
-->

## Summary

EROFS (Enhanced Read-Only File System) is a read-only filesystem optimised for compressed Android system partitions, container image layers, and immutable read-only system installs. On-disk: 1024-byte `EROFS_SUPER_OFFSET` skip followed by `struct erofs_super_block` (magic `EROFS_SUPER_MAGIC_V1` = 0xE0F5E1E2) with a fixed power-of-two block size (`blkszbits ∈ [9, PAGE_SHIFT]`, typically 4 KiB). Per-mount `struct erofs_sb_info` (sbi) caches `feature_compat`, `feature_incompat`, `meta_blkaddr`, `xattr_blkaddr`, `root_nid`, `packed_nid`, `metabox_nid`, `dif0` (primary device-info), plus the device-info IDR for multi-device images. Per-`erofs_fc_fill_super` flow: allocate sbi via fs-context, choose backing (block-device / file-backed loop / fscache on-demand), read superblock, validate features against `EROFS_ALL_FEATURE_INCOMPAT`, scan extra devices, parse compressor configs (z_erofs), iget root/packed/metabox inodes. Per-fscache mode (`CONFIG_EROFS_FS_ONDEMAND`): superblock is fetched via fscache cookies, allowing chunk-based files to source data from a userspace daemon. Per-file-backed mode (`CONFIG_EROFS_FS_BACKED_BY_FILE`): allows mounting an EROFS image stored inside another filesystem. Critical for: Android boot, container layer dedup, read-only signed system trees.

This Tier-3 covers `fs/erofs/super.c` (~1138 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct erofs_super_block` | per on-disk SB | `ErofsSuperBlockOnDisk` |
| `struct erofs_sb_info` | per-mount info | `ErofsSbInfo` |
| `struct erofs_device_info` | per-extra-device | `ErofsDeviceInfo` |
| `erofs_fc_fill_super()` | per fill_super | `Erofs::fc_fill_super` |
| `erofs_read_superblock()` | per parse SB | `Erofs::read_superblock` |
| `erofs_superblock_csum_verify()` | per SB-CRC | `Erofs::sb_csum_verify` |
| `erofs_scan_devices()` | per multi-device | `Erofs::scan_devices` |
| `erofs_init_device()` | per-device slot | `Erofs::init_device` |
| `erofs_default_options()` | per opt-defaults | `Erofs::default_options` |
| `erofs_fc_parse_param()` | per parse-option | `Erofs::fc_parse_param` |
| `erofs_fc_set_dax_mode()` | per-DAX mode | `Erofs::fc_set_dax_mode` |
| `erofs_fc_get_tree()` | per get_tree | `Erofs::fc_get_tree` |
| `erofs_fc_reconfigure()` | per remount | `Erofs::fc_reconfigure` |
| `erofs_init_fs_context()` | per init-fc | `Erofs::init_fs_context` |
| `erofs_fc_free()` | per fc-free | `Erofs::fc_free` |
| `erofs_kill_sb()` | per kill_sb | `Erofs::kill_sb` |
| `erofs_put_super()` | per umount | `Erofs::put_super` |
| `erofs_alloc_inode()` / `erofs_free_inode()` | per-inode slab | `Erofs::alloc_inode` / `free_inode` |
| `erofs_drop_internal_inodes()` | per drop packed/metabox/cache | `Erofs::drop_internal_inodes` |
| `erofs_set_sysfs_name()` / `erofs_register_sysfs()` | per sysfs | `Erofs::sysfs_name` / `register_sysfs` |
| `erofs_statfs()` | per statfs | `Erofs::statfs` |
| `erofs_show_options()` | per /proc/mounts | `Erofs::show_options` |
| `erofs_evict_inode()` | per inode-evict | `Erofs::evict_inode` |
| `z_erofs_parse_cfgs()` | per-compr-config | `ZErofs::parse_cfgs` |
| `z_erofs_init_super()` | per-compr-init | `ZErofs::init_super` |
| `erofs_fscache_register_fs()` | per fscache mount | `ErofsFscache::register_fs` |
| `erofs_fscache_register_cookie()` | per-device cookie | `ErofsFscache::register_cookie` |
| `erofs_anon_fs_type` | per pseudo SB | `Erofs::anon_fs_type` |
| `erofs_module_init()` / `_exit()` | per module load | `Erofs::module_init` / `exit` |

## Compatibility contract

REQ-1: struct erofs_super_block on-disk (must match `erofs_fs.h`):
- magic (le32): `EROFS_SUPER_MAGIC_V1` = 0xE0F5E1E2.
- checksum (le32): CRC32C of SB area when `EROFS_FEATURE_COMPAT_SB_CHKSUM` set.
- feature_compat (le32): compat flag bitmask.
- blkszbits (u8): log2(block-size); `9 ≤ blkszbits ≤ PAGE_SHIFT`.
- sb_extslots (u8): SB extension slots count; `sb_size = 128 + sb_extslots * 16`, `sb_size ≤ PAGE_SIZE - EROFS_SUPER_OFFSET`.
- rb.rootnid_2b / rootnid_8b: 16-bit (legacy) / 64-bit (48BIT feature) root NID.
- inos (le64): total valid ino count.
- epoch (le64) / fixed_nsec (le32): timestamp base for compact inodes.
- blocks_lo / rb.blocks_hi (when 48BIT): total block count (32 or 48 bits).
- meta_blkaddr (le32) / xattr_blkaddr (le32): start block addrs.
- uuid[16] / volume_name[16]: identity.
- feature_incompat (le32): incompat flag bitmask.
- u1.available_compr_algs / u1.lz4_max_distance (overlay): compression hints.
- extra_devices (le16) / devt_slotoff (le16): multi-device table.
- dirblkbits (u8): directory blkbits; must be 0 in 7.1.0-rc2.
- xattr_prefix_count (u8) / xattr_prefix_start (le32): long xattr prefix table.
- packed_nid (le64): NID of packed-inode for fragmented tail data.
- xattr_filter_reserved (u8) / ishare_xattr_prefix_id (u8).
- build_time (le32): mkfs time = epoch + build_time.
- metabox_nid (le64): NID of metadata-compression box (when METABOX set).

REQ-2: EROFS_FEATURE_COMPAT_*:
- SB_CHKSUM (0x1): SB CRC32C valid.
- MTIME (0x2): per-inode mtime extension.
- XATTR_FILTER (0x4): bloom filter for xattr probe.
- SHARED_EA_IN_METABOX (0x8): shared EA inside metabox.
- PLAIN_XATTR_PFX (0x10): plain xattr-prefix encoding.
- ISHARE_XATTRS (0x20): inode-share xattrs (page-cache share).

REQ-3: EROFS_FEATURE_INCOMPAT_*:
- LZ4_0PADDING (0x1): LZ4 zero-padded.
- COMPR_CFGS (0x2): per-algorithm compr-configs table present.
- BIG_PCLUSTER (0x2): big pcluster (alias bit; mutually exclusive with COMPR_CFGS in older format).
- CHUNKED_FILE (0x4): chunk-based file layout.
- DEVICE_TABLE (0x8): multi-device table present.
- COMPR_HEAD2 (0x8): second compressor head (alias).
- ZTAILPACKING (0x10): inline tail-pack into inode.
- FRAGMENTS (0x20): fragments inode (packed_nid).
- DEDUPE (0x20): block-dedup (alias).
- XATTR_PREFIXES (0x40): long xattr-prefix table.
- 48BIT (0x80): 48-bit blocks/nid.
- METABOX (0x100): metadata-compression box (compresses on-disk inodes).
- EROFS_ALL_FEATURE_INCOMPAT = ((METABOX << 1) - 1).

REQ-4: erofs_read_superblock(sb):
- /* Read folio @ disk-offset 0; data + EROFS_SUPER_OFFSET = dsb */.
- if le32(dsb.magic) != EROFS_SUPER_MAGIC_V1: -EINVAL.
- sbi.blkszbits = dsb.blkszbits.
- if blkszbits < 9 ∨ blkszbits > PAGE_SHIFT: -EINVAL.
- if dsb.dirblkbits != 0: -EINVAL.
- sbi.feature_compat = le32(dsb.feature_compat).
- if SB_CHKSUM ∧ erofs_superblock_csum_verify fails: -EFSCORRUPTED.
- sbi.feature_incompat = le32(dsb.feature_incompat).
- if feature_incompat & ~EROFS_ALL_FEATURE_INCOMPAT: -EINVAL "unidentified incompatible feature".
- sbi.sb_size = 128 + dsb.sb_extslots * EROFS_SB_EXTSLOT_SIZE; bound check ≤ PAGE_SIZE - EROFS_SUPER_OFFSET.
- sbi.dif0.blocks = le32(dsb.blocks_lo).
- sbi.meta_blkaddr = le32(dsb.meta_blkaddr).
- if CONFIG_EROFS_FS_XATTR: copy xattr_blkaddr / xattr_prefix_start / xattr_prefix_count; validate ishare_xattr_prefix_id < xattr_prefix_count.
- sbi.islotbits = ilog2(sizeof(erofs_inode_compact)).
- /* 48-bit aware root-nid */
- if has_48bit ∧ dsb.rootnid_8b: sbi.root_nid = le64(dsb.rootnid_8b); blocks |= le16(dsb.rb.blocks_hi) << 32.
- else: sbi.root_nid = le16(dsb.rb.rootnid_2b).
- sbi.packed_nid = le64(dsb.packed_nid).
- if has_metabox: sb_size > offsetof(metabox_nid); sbi.metabox_nid = le64; reject self-loop (METABOX_BIT inside metabox_nid).
- sbi.inos = le64(dsb.inos); sbi.epoch = le64(dsb.epoch); sbi.fixed_nsec = le32(dsb.fixed_nsec).
- super_set_uuid(sb, dsb.uuid).
- if dsb.volume_name[0]: sbi.volume_name = kstrndup.
- if CONFIG_EROFS_FS_ZIP: z_erofs_parse_cfgs(sb, dsb).
- else if dsb.u1.available_compr_algs ∨ lz4_0padding-feature: -EOPNOTSUPP "compression disabled".
- erofs_scan_devices(sb, dsb).
- info-log EXPERIMENTAL warnings (48BIT, METABOX, fscache).

REQ-5: erofs_scan_devices(sb, dsb):
- sbi.total_blocks = sbi.dif0.blocks.
- if !has_DEVICE_TABLE: ondisk_extradevs = 0; else ondisk_extradevs = le16(dsb.extra_devices).
- if sbi.devs.extra_devices (preconfigured from mount-opt) ≠ ondisk_extradevs: -EINVAL "extra devices don't match".
- if DAX_ALWAYS ∧ !dif0.dax_dev: clear DAX_ALWAYS.
- if !ondisk_extradevs: return 0.
- if !extra_devices ∧ !fscache_mode: sbi.devs.flatdev = true.
- sbi.device_id_mask = roundup_pow_of_two(ondisk_extradevs + 1) - 1.
- pos = le16(dsb.devt_slotoff) * EROFS_DEVT_SLOT_SIZE.
- down_read(sbi.devs.rwsem); foreach slot: erofs_init_device(buf, sb, dif, &pos); up_read.

REQ-6: erofs_init_device(buf, sb, dif, *pos):
- dis = erofs_read_metabuf(buf, sb, *pos).
- if !flatdev ∧ !dif.path: copy dis.tag → dif.path (kmemdup_nul).
- if fscache-mode: dif.fscache = erofs_fscache_register_cookie(sb, dif.path, 0).
- else if !flatdev:
  - file = (fileio-mode ? filp_open(O_RDONLY|O_LARGEFILE) : bdev_file_open_by_path(BLK_OPEN_READ)).
  - if !fileio: dif.dax_dev = fs_dax_get_by_bdev(file_bdev(file), &dif.dax_part_off).
  - else if !S_ISREG(inode): fput; -EINVAL.
  - dif.file = file.
- dif.blocks = le32(dis.blocks_lo) | (48BIT ? le16(dis.blocks_hi)<<32 : 0).
- dif.uniaddr = le32(dis.uniaddr_lo) | (48BIT ? le16(dis.uniaddr_hi)<<32 : 0).
- sbi.total_blocks += dif.blocks; *pos += EROFS_DEVT_SLOT_SIZE.

REQ-7: erofs_fc_fill_super(sb, fc):
- sbi = EROFS_SB(sb); sb.s_magic = EROFS_SUPER_MAGIC; sb.s_flags |= SB_RDONLY | SB_NOATIME; s_maxbytes = MAX_LFS_FILESIZE; s_op = &erofs_sops.
- validate domain_id needed when INODE_SHARE; DAX_ALWAYS ⊕ INODE_SHARE.
- sbi.blkszbits = PAGE_SHIFT (initial).
- if !sb.s_bdev:
  - if fileio-mode: forbid stacking on another erofs file-backed mount (s_stack_depth == 0 only).
  - sb.s_blocksize = PAGE_SIZE; s_blocksize_bits = PAGE_SHIFT.
  - if fscache-mode: erofs_fscache_register_fs(sb).
  - super_setup_bdi(sb).
- else: sb_set_blocksize(sb, PAGE_SIZE); dif0.dax_dev = fs_dax_get_by_bdev.
- erofs_read_superblock(sb).
- if sb.s_blocksize_bits ≠ sbi.blkszbits: adjust (fscache-mode rejects mismatch; fileio sets directly; bdev calls sb_set_blocksize).
- if dif0.fsoff: must be block-aligned ∧ not in fscache-mode.
- if DAX_ALWAYS ∧ blkszbits ≠ PAGE_SHIFT: clear DAX_ALWAYS.
- if INODE_SHARE ∧ !ishare_xattrs-feature: clear INODE_SHARE.
- sb.s_time_gran = 1; s_xattr = erofs_xattr_handlers; s_export_op = &erofs_export_ops.
- if POSIX_ACL: sb.s_flags |= SB_POSIXACL.
- z_erofs_init_super(sb).
- if has_FRAGMENTS ∧ sbi.packed_nid: sbi.packed_inode = erofs_iget(sb, packed_nid).
- if has_METABOX: sbi.metabox_inode = erofs_iget(sb, metabox_nid).
- root = erofs_iget(sb, sbi.root_nid); validate S_ISDIR.
- sb.s_root = d_make_root(root).
- erofs_shrinker_register(sb); erofs_xattr_prefixes_init(sb); erofs_set_sysfs_name(sb); erofs_register_sysfs(sb).
- sbi.dir_ra_bytes = EROFS_DIR_RA_BYTES.
- info-log "mounted with root inode @ nid …".

REQ-8: erofs_fc_get_tree(fc):
- /* fscache-mode → nodev */
- if CONFIG_EROFS_FS_ONDEMAND ∧ sbi.fsid: return get_tree_nodev(fc, erofs_fc_fill_super).
- /* normal: block-device tree (allow quiet lookup when file-backed config) */
- ret = get_tree_bdev_flags(fc, erofs_fc_fill_super, fileio ? GET_TREE_BDEV_QUIET_LOOKUP : 0).
- /* fallback: file-backed mount */
- if fileio-config ∧ ret == -ENOTBLK:
  - file = filp_open(fc.source, O_RDONLY|O_LARGEFILE, 0).
  - sbi.dif0.file = file.
  - if S_ISREG ∧ a_ops.read_folio: get_tree_nodev(fc, erofs_fc_fill_super).
- return ret.

REQ-9: erofs_fc_reconfigure(fc):
- /* RO filesystem: remount-RW disallowed; only opt-tuning */
- DBG_BUGON(!sb_rdonly(sb)).
- ignore-with-info fsid|domain_id (cannot change backing).
- if POSIX_ACL: fc.sb_flags |= SB_POSIXACL.
- sb.s_flags retains SB_RDONLY.

REQ-10: erofs_kill_sb(sb):
- if (CONFIG_EROFS_FS_ONDEMAND ∧ fsid) ∨ dif0.file: kill_anon_super(sb).
- else: kill_block_super(sb).
- erofs_drop_internal_inodes(sbi).
- fs_put_dax(dif0.dax_dev).
- erofs_fscache_unregister_fs(sb).
- erofs_sb_free(sbi); sb.s_fs_info = NULL.

REQ-11: erofs_put_super(sb):
- erofs_unregister_sysfs; erofs_shrinker_unregister; erofs_xattr_prefixes_cleanup; erofs_drop_internal_inodes; erofs_free_dev_context(sbi.devs); erofs_fscache_unregister_fs.

REQ-12: erofs_drop_internal_inodes(sbi):
- iput(packed_inode); = NULL.
- iput(metabox_inode); = NULL.
- if CONFIG_EROFS_FS_ZIP: iput(managed_cache); = NULL.

REQ-13: erofs_fc_parse_param(fc, param):
- Parse user_xattr, posix_acl, cache_strategy={disabled,readahead,readaround}, dax={never,always,inode}, fsid, domain_id, fsoffset, inode_share, etc.
- All options accepted only via fs-context (mount(2) new-API).

REQ-14: erofs_anon_fs_type (pseudo_erofs):
- Pseudo-superblock for fscache anon-inodes or page-cache-share group inodes (CONFIG_EROFS_FS_ONDEMAND ∨ CONFIG_EROFS_FS_PAGE_CACHE_SHARE).
- init_fs_context = erofs_anon_init_fs_context; init_pseudo with EROFS_SUPER_MAGIC.
- super_operations.alloc_inode = erofs_alloc_inode; drop_inode = inode_just_drop.

REQ-15: erofs_superblock_csum_verify(sb, sbdata):
- crc = crc32c(~0, sbdata + EROFS_SUPER_OFFSET, sbi.sb_size - sizeof(le32_for_csum)).
- compare against le32(dsb.checksum); mismatch ⇒ -EFSCORRUPTED.

REQ-16: erofs_statfs(dentry, kstatfs):
- bsize = blocksize; blocks = sbi.total_blocks; bfree = 0 (RO); files = sbi.inos; ffree = 0; namelen = EROFS_NAME_LEN; fsid from sb.s_uuid; type = EROFS_SUPER_MAGIC.

REQ-17: erofs_evict_inode(inode):
- truncate_inode_pages_final(mapping).
- clear_inode(inode).
- if CONFIG_EROFS_FS_ZIP and inode has z_erofs metadata: z_erofs_free_inode_metadata.
- free per-inode chunk_idxes / metabox slot when applicable.

## Acceptance Criteria

- [ ] AC-1: Mount on EROFS image: erofs_fc_fill_super succeeds; root nid resolved.
- [ ] AC-2: Bad magic in dsb: erofs_read_superblock returns -EINVAL.
- [ ] AC-3: blkszbits = 8 or > PAGE_SHIFT: erofs_read_superblock returns -EINVAL.
- [ ] AC-4: feature_incompat & ~EROFS_ALL_FEATURE_INCOMPAT: -EINVAL "unidentified incompatible".
- [ ] AC-5: SB_CHKSUM bad: -EFSCORRUPTED.
- [ ] AC-6: 48BIT image: 48-bit root_nid + blocks parsed.
- [ ] AC-7: METABOX image: metabox_inode iget'd; self-loop rejected (-EFSCORRUPTED).
- [ ] AC-8: Multi-device image: erofs_scan_devices opens all extradevs; total_blocks sums.
- [ ] AC-9: fscache-mode mount (fsid set): get_tree_nodev path; erofs_fscache_register_fs.
- [ ] AC-10: File-backed mount: ENOTBLK fallback opens file via filp_open; rejects nested file-backed.
- [ ] AC-11: Compression disabled at build ∧ image has compressed data: -EOPNOTSUPP.
- [ ] AC-12: erofs_fc_reconfigure: SB_RDONLY preserved; remount-RW rejected.
- [ ] AC-13: umount: erofs_kill_sb drops packed/metabox/managed-cache inodes; sb.s_fs_info = NULL.
- [ ] AC-14: erofs_statfs returns bfree=0, ffree=0 (RO).
- [ ] AC-15: Mount with INODE_SHARE but no ishare-xattrs feature: silently clears INODE_SHARE.

## Architecture

```
struct ErofsSuperBlockOnDisk {     // packed LE; matches erofs_super_block byte-for-byte
  magic: u32,                       // EROFS_SUPER_MAGIC_V1
  checksum: u32,                    // crc32c over sb area
  feature_compat: u32,
  blkszbits: u8,
  sb_extslots: u8,
  rb_or_root: U16Union,             // rootnid_2b or (48BIT) blocks_hi
  inos: u64,
  epoch: u64,
  fixed_nsec: u32,
  blocks_lo: u32,
  meta_blkaddr: u32,
  xattr_blkaddr: u32,
  uuid: [u8; 16],
  volume_name: [u8; 16],
  feature_incompat: u32,
  u1: U32Union,                     // available_compr_algs / lz4_max_distance
  extra_devices: u16,
  devt_slotoff: u16,
  dirblkbits: u8,                   // == 0 in this baseline
  xattr_prefix_count: u8,
  xattr_prefix_start: u32,
  packed_nid: u64,
  xattr_filter_reserved: u8,
  ishare_xattr_prefix_id: u8,
  reserved: [u8; 2],
  build_time: u32,
  rootnid_8b: u64,                  // (48BIT)
  reserved2: u64,
  metabox_nid: u64,                 // (METABOX)
  reserved3: u64,
}

struct ErofsSbInfo {
  dif0: ErofsDeviceInfo,            // primary device
  devs: ErofsDevContext,            // IDR of extra devices
  blkszbits: u8,
  islotbits: u8,
  sb_size: u32,                     // 128 + sb_extslots * 16
  feature_compat: u32,
  feature_incompat: u32,
  meta_blkaddr: u32,
  xattr_blkaddr: u32,
  xattr_prefix_start: u32,
  xattr_prefix_count: u8,
  ishare_xattr_prefix_id: u8,
  xattr_filter_reserved: u8,
  device_id_mask: u32,
  total_blocks: u64,
  inos: u64,
  root_nid: u64,
  packed_nid: u64,
  metabox_nid: u64,
  packed_inode: Option<InodeRef>,
  metabox_inode: Option<InodeRef>,
  managed_cache: Option<InodeRef>,  // CONFIG_EROFS_FS_ZIP
  epoch: i64,
  fixed_nsec: u32,
  volume_name: Option<String>,
  fsid: Option<String>,             // fscache
  domain_id: Option<String>,
  opt: ErofsMountOpts,              // cache_strategy, dax, posix_acl, inode_share, ...
  dir_ra_bytes: u32,
}

struct ErofsDeviceInfo {
  file: Option<FileHandle>,
  fscache: Option<FscacheCookie>,
  dax_dev: Option<DaxDeviceRef>,
  dax_part_off: u64,
  blocks: u64,                       // 32 or 48-bit blocks
  uniaddr: u64,
  fsoff: u64,
  path: Option<String>,
}
```

`Erofs::fc_fill_super(sb, fc) -> Result<(), Errno>`:
1. sbi = ErofsSb::from_fc(fc).
2. sb.s_magic = EROFS_SUPER_MAGIC; sb.s_flags |= SB_RDONLY | SB_NOATIME; sb.s_maxbytes = MAX_LFS_FILESIZE; sb.s_op = &EROFS_SOPS.
3. /* INODE_SHARE / DAX constraints */
4. if sbi.domain_id.is_none() ∧ sbi.opt.inode_share: errorfc; return -EINVAL.
5. if sbi.opt.dax_always ∧ sbi.opt.inode_share: errorfc; return -EINVAL.
6. sbi.blkszbits = PAGE_SHIFT.
7. /* Backing setup */
8. if sb.s_bdev.is_none():
   - if fileio-mode: enforce s_stack_depth == 0.
   - sb.s_blocksize = PAGE_SIZE; sb.s_blocksize_bits = PAGE_SHIFT.
   - if fscache-mode: Erofs::fscache::register_fs(sb)?.
   - super_setup_bdi(sb)?.
9. else: sb_set_blocksize(sb, PAGE_SIZE)?; sbi.dif0.dax_dev = fs_dax_get_by_bdev(...).
10. Erofs::read_superblock(sb)?.
11. /* Re-align block-size if ondisk blkszbits ≠ PAGE_SHIFT */
12. if sb.s_blocksize_bits ≠ sbi.blkszbits: per backing-mode adjust (reject in fscache-mode).
13. /* dif0.fsoff alignment & exclusion */
14. /* DAX_ALWAYS / INODE_SHARE post-validate */
15. sb.s_time_gran = 1; sb.s_xattr = &EROFS_XATTR_HANDLERS; sb.s_export_op = &EROFS_EXPORT_OPS.
16. if sbi.opt.posix_acl: sb.s_flags |= SB_POSIXACL.
17. ZErofs::init_super(sb)?.
18. /* Internal inodes */
19. if has_fragments ∧ sbi.packed_nid ≠ 0: sbi.packed_inode = Some(erofs_iget(sb, sbi.packed_nid)?).
20. if has_metabox: sbi.metabox_inode = Some(erofs_iget(sb, sbi.metabox_nid)?).
21. root = erofs_iget(sb, sbi.root_nid)?; require S_ISDIR.
22. sb.s_root = d_make_root(root).
23. erofs_shrinker_register(sb); erofs_xattr_prefixes_init(sb)?; erofs_set_sysfs_name(sb); erofs_register_sysfs(sb)?.
24. sbi.dir_ra_bytes = EROFS_DIR_RA_BYTES.
25. info!("mounted with root inode @ nid {}", sbi.root_nid).
26. return Ok(()).

`Erofs::read_superblock(sb) -> Result<(), Errno>`:
1. data = erofs_read_metabuf(&buf, sb, 0, false)?.
2. dsb = (data + EROFS_SUPER_OFFSET) as *const ErofsSuperBlockOnDisk.
3. if le32(dsb.magic) ≠ EROFS_SUPER_MAGIC_V1: err!("invalid magic"); -EINVAL.
4. sbi.blkszbits = dsb.blkszbits.
5. if blkszbits < 9 ∨ blkszbits > PAGE_SHIFT: -EINVAL.
6. if dsb.dirblkbits ≠ 0: -EINVAL.
7. sbi.feature_compat = le32(dsb.feature_compat).
8. if has_sb_chksum: Erofs::sb_csum_verify(sb, data)?.
9. sbi.feature_incompat = le32(dsb.feature_incompat).
10. if feature_incompat & ~EROFS_ALL_FEATURE_INCOMPAT ≠ 0: -EINVAL.
11. sbi.sb_size = 128 + dsb.sb_extslots * EROFS_SB_EXTSLOT_SIZE.
12. if sb_size > PAGE_SIZE - EROFS_SUPER_OFFSET: -EINVAL.
13. sbi.dif0.blocks = le32(dsb.blocks_lo); sbi.meta_blkaddr = le32(dsb.meta_blkaddr).
14. if CONFIG_EROFS_FS_XATTR: copy xattr_blkaddr / prefix table; validate ishare_xattr_prefix_id < count.
15. sbi.islotbits = ilog2(sizeof(erofs_inode_compact)).
16. if has_48bit ∧ dsb.rootnid_8b ≠ 0:
    - sbi.root_nid = le64(dsb.rootnid_8b).
    - sbi.dif0.blocks |= le16(dsb.rb.blocks_hi) << 32.
17. else: sbi.root_nid = le16(dsb.rb.rootnid_2b).
18. sbi.packed_nid = le64(dsb.packed_nid).
19. if has_metabox:
    - require sb_size > offsetof(ErofsSuperBlockOnDisk, metabox_nid).
    - sbi.metabox_nid = le64(dsb.metabox_nid).
    - reject self-loop: metabox_nid & BIT_ULL(EROFS_DIRENT_NID_METABOX_BIT) ⇒ -EFSCORRUPTED.
20. sbi.inos = le64(dsb.inos); sbi.epoch = le64(dsb.epoch); sbi.fixed_nsec = le32(dsb.fixed_nsec).
21. super_set_uuid(sb, &dsb.uuid).
22. if dsb.volume_name[0] ≠ 0: sbi.volume_name = Some(kstrndup(dsb.volume_name)?).
23. if CONFIG_EROFS_FS_ZIP: ZErofs::parse_cfgs(sb, dsb)?.
    else if dsb.u1.available_compr_algs ≠ 0 ∨ has_lz4_0padding: -EOPNOTSUPP.
24. Erofs::scan_devices(sb, dsb)?.
25. info-log EXPERIMENTAL warnings.

`Erofs::scan_devices(sb, dsb) -> Result<(), Errno>`:
1. sbi.total_blocks = sbi.dif0.blocks.
2. if !has_device_table: ondisk_extradevs = 0; else ondisk_extradevs = le16(dsb.extra_devices).
3. if sbi.devs.extra_devices ≠ 0 ∧ ondisk_extradevs ≠ sbi.devs.extra_devices: -EINVAL "extra devices don't match".
4. if dax_always ∧ !dif0.dax_dev: clear dax_always (info-log).
5. if ondisk_extradevs == 0: return Ok(()).
6. if sbi.devs.extra_devices == 0 ∧ !fscache_mode: sbi.devs.flatdev = true.
7. sbi.device_id_mask = roundup_pow_of_two(ondisk_extradevs + 1) - 1.
8. pos = le16(dsb.devt_slotoff) * EROFS_DEVT_SLOT_SIZE.
9. down_read(sbi.devs.rwsem).
10. if sbi.devs.extra_devices > 0:
    - idr_for_each(sbi.devs.tree, |dif| Erofs::init_device(&buf, sb, dif, &pos)).
11. else:
    - for id in 0..ondisk_extradevs:
      - dif = kzalloc_obj(ErofsDeviceInfo)?; idr_alloc; sbi.devs.extra_devices += 1.
      - Erofs::init_device(&buf, sb, dif, &pos)?.
12. up_read(sbi.devs.rwsem).

`Erofs::init_device(buf, sb, dif, pos) -> Result<(), Errno>`:
1. dis = erofs_read_metabuf(buf, sb, *pos, false)?.
2. if !flatdev ∧ dif.path.is_none():
   - if dis.tag[0] == 0: -EINVAL "empty device tag".
   - dif.path = Some(kmemdup_nul(dis.tag)?).
3. if fscache_mode:
   - dif.fscache = Some(Erofs::fscache::register_cookie(sb, &dif.path?, 0)?).
4. else if !flatdev:
   - file = if fileio_mode { filp_open(&dif.path?, O_RDONLY|O_LARGEFILE, 0)? }
            else { bdev_file_open_by_path(&dif.path?, BLK_OPEN_READ, sb.s_type)? }.
   - if !fileio_mode: dif.dax_dev = fs_dax_get_by_bdev(file_bdev(&file), &mut dif.dax_part_off).
   - else if !S_ISREG(file_inode(&file).i_mode): fput(file); -EINVAL.
   - dif.file = Some(file).
5. let _48bit = has_48bit.
6. dif.blocks = le32(dis.blocks_lo) | (_48bit ? le16(dis.blocks_hi)<<32 : 0).
7. dif.uniaddr = le32(dis.uniaddr_lo) | (_48bit ? le16(dis.uniaddr_hi)<<32 : 0).
8. sbi.total_blocks += dif.blocks; *pos += EROFS_DEVT_SLOT_SIZE.

`Erofs::fc_get_tree(fc) -> Result<(), Errno>`:
1. /* fscache on-demand path */
2. if CONFIG_EROFS_FS_ONDEMAND ∧ sbi.fsid.is_some():
   - return get_tree_nodev(fc, Erofs::fc_fill_super).
3. /* normal: block-device */
4. ret = get_tree_bdev_flags(fc, Erofs::fc_fill_super,
       if CONFIG_EROFS_FS_BACKED_BY_FILE { GET_TREE_BDEV_QUIET_LOOKUP } else { 0 }).
5. /* file-backed fallback */
6. if CONFIG_EROFS_FS_BACKED_BY_FILE ∧ ret == -ENOTBLK:
   - require fc.source.is_some(); else invalf "No source specified".
   - file = filp_open(fc.source, O_RDONLY|O_LARGEFILE, 0)?.
   - sbi.dif0.file = Some(file).
   - if S_ISREG ∧ file.f_mapping.a_ops.read_folio:
     - return get_tree_nodev(fc, Erofs::fc_fill_super).
7. return ret.

`Erofs::kill_sb(sb)`:
1. sbi = EROFS_SB(sb).
2. if (CONFIG_EROFS_FS_ONDEMAND ∧ sbi.fsid.is_some()) ∨ sbi.dif0.file.is_some():
   - kill_anon_super(sb).
3. else: kill_block_super(sb).
4. Erofs::drop_internal_inodes(sbi).
5. fs_put_dax(sbi.dif0.dax_dev, None).
6. Erofs::fscache::unregister_fs(sb).
7. Erofs::sb_free(sbi); sb.s_fs_info = None.

`Erofs::drop_internal_inodes(sbi)`:
1. iput(sbi.packed_inode.take()).
2. iput(sbi.metabox_inode.take()).
3. if CONFIG_EROFS_FS_ZIP: iput(sbi.managed_cache.take()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `magic_validated_before_use` | INVARIANT | per-read_superblock: le32(dsb.magic) checked == EROFS_SUPER_MAGIC_V1 before any further dsb deref. |
| `blkszbits_bounded` | INVARIANT | per-read_superblock: blkszbits ∈ [9, PAGE_SHIFT]. |
| `sb_size_bounded` | INVARIANT | per-read_superblock: sb_size ≤ PAGE_SIZE - EROFS_SUPER_OFFSET. |
| `incompat_mask_strict` | INVARIANT | per-read_superblock: (feature_incompat & ~EROFS_ALL_FEATURE_INCOMPAT) == 0. |
| `metabox_self_loop_excluded` | INVARIANT | per-read_superblock: metabox_nid lacks DIRENT_NID_METABOX_BIT. |
| `extradevs_count_match` | INVARIANT | per-scan_devices: ondisk_extradevs == sbi.devs.extra_devices (when preconfigured). |
| `dif0_refcount_balanced` | INVARIANT | per-init_device error path: filp_open success ⟹ either dif.file owned or fput on err. |

### Layer 2: TLA+

`fs/erofs/super.tla`:
- Variables: sb_state ∈ {Unmounted, ReadingSB, ScanningDevs, Mounted, KillingSb}, feature_bits, devices, root_nid.
- Properties:
  - `safety_RO_invariant` — per-Mount: sb.s_flags & SB_RDONLY persists across reconfigure.
  - `safety_no_unknown_incompat` — per-mount: feature_incompat ⊆ ALL_FEATURE_INCOMPAT.
  - `safety_metabox_no_self_loop` — per-mount: metabox_nid ≠ any dirent self-ref.
  - `safety_extradevs_match` — per-scan_devices: pre-configured device count matches on-disk.
  - `liveness_mount_terminates` — per-fc_fill_super: terminates (Ok | Err).
  - `liveness_kill_sb_releases` — per-umount: all internal inodes iput'd; managed_cache, packed_inode, metabox_inode all NULL.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Erofs::read_superblock` post: ret == Ok ⟹ sbi.root_nid set ∧ feature bits parsed | `Erofs::read_superblock` |
| `Erofs::scan_devices` post: total_blocks == Σ dif.blocks (dif0 + extras) | `Erofs::scan_devices` |
| `Erofs::init_device` post: dif owns exactly one of (file, fscache cookie) | `Erofs::init_device` |
| `Erofs::fc_fill_super` post: ret Ok ⟹ sb.s_root ≠ NULL ∧ S_ISDIR(root.i_mode) ∧ SB_RDONLY set | `Erofs::fc_fill_super` |
| `Erofs::fc_reconfigure` post: sb.s_flags & SB_RDONLY unchanged | `Erofs::fc_reconfigure` |
| `Erofs::kill_sb` post: all internal inodes released; sb.s_fs_info == NULL | `Erofs::kill_sb` |
| `Erofs::drop_internal_inodes` post: packed_inode == NULL ∧ metabox_inode == NULL ∧ managed_cache == NULL | `Erofs::drop_internal_inodes` |

### Layer 4: Verus/Creusot functional

`Per-mount: bdev → read folio 0 → validate magic/blkszbits/features → scan devt slots → z_erofs parse_cfgs → iget(root) → d_make_root → register sysfs` semantic equivalence: per Documentation/filesystems/erofs.rst + erofs-utils on-disk reference; mkfs.erofs-produced images byte-identical to Linux-mounted view (xfstests RO subset and `erofsfuse` cross-validation).

## Hardening

(Inherits row-1 features from `fs/erofs/00-overview.md` § Hardening.)

EROFS-super reinforcement:

- **Per-magic-then-deref ordering** — defense against per-uninitialised-SB-field-read on EFSCORRUPTED images.
- **Per-blkszbits ∈ [9, PAGE_SHIFT] enforced** — defense against per-OOB metabuf-read with absurd shift.
- **Per-sb_size ≤ PAGE_SIZE - EROFS_SUPER_OFFSET** — defense against per-sb-extslot OOB.
- **Per-feature_incompat strict mask** — defense against per-future-feature mounted on old kernel reading garbage.
- **Per-SB_CHKSUM verify before scan_devices** — defense against per-corrupt-SB poisoning device table.
- **Per-metabox self-loop rejected** — defense against per-attacker-crafted iget recursion.
- **Per-DEVICE_TABLE extra-devs count match** — defense against per-mount-time device-set mismatch.
- **Per-init_device fput on error** — defense against per-file/bdev leak on partial scan failure.
- **Per-fscache-mode + DAX mutually exclusive checks** — defense against per-DAX-on-fscache UB.
- **Per-file-backed nesting forbidden (s_stack_depth == 0)** — defense against per-stack-overflow on recursive erofs-on-erofs file mount.
- **Per-fsoff block-aligned** — defense against per-misaligned-IO panic.
- **Per-INODE_SHARE only when ishare-xattrs feature present** — defense against per-share-without-key data leak.
- **Per-kill_anon vs kill_block_super dispatch** — defense against per-wrong-superblock-teardown crash.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- z_erofs compressor implementations (covered in `decompressor.md` Tier-3)
- z_erofs cluster decompress + readahead (covered in `zdata-zmap.md` Tier-3)
- erofs xattr handlers and prefix table (covered in `xattr.md` Tier-3)
- fscache cookie I/O paths (covered in `fscache.md` Tier-3)
- erofs inode / dir / namei (covered in `inode-dir-namei.md` Tier-3)
- Implementation code
