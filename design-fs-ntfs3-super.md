---
title: "Tier-3: fs/ntfs3/super.c — NTFS-3 superblock + mount"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

ntfs3 is the **Paragon-contributed kernel-mode NTFS read/write driver** that replaced the long-dormant `fs/ntfs/` read-only driver. It implements the on-disk format used by Windows NT/2000/XP/Vista/7/8/10/11 (NTFS versions 3.0 + 3.1) with **resident + non-resident attributes**, **runs (compressed VCN→LCN extent maps)**, **MFT $MFT lookups**, **$MFTMirr backup record mirror**, **$LogFile journal replay**, **$Bitmap cluster bitmap**, **$Secure SDS/SDH/SII indices**, **$Extend/$Reparse + $ObjId**, the **$UpCase case-folding table**, optional **read-only LZX/Xpress decompression** (`CONFIG_NTFS3_LZX_XPRESS`), and **64-bit-LCN** support for > 16 TB volumes (`CONFIG_NTFS3_64BIT_CLUSTER`). `super.c` owns the mount-API plumbing: the `struct fs_context_operations`, the `fs_parameter_spec` table parsing `uid=/gid=/umask=/dmask=/fmask=/sparse/showmeta/iocharset=/discard/force/prealloc/nocase/acl/...`, `ntfs_init_fs_context` (allocate `struct ntfs_sb_info` + `ntfs_mount_options`), `ntfs_fs_get_tree` → `get_tree_bdev(ntfs_fill_super)`, `ntfs_init_from_boot` (parse + cross-check the 512-byte `struct NTFS_BOOT` sector — and on failure fall through to the alternative boot at the last sector of the device), `ntfs_fill_super` (load $Volume → $MFTMirr → $LogFile → replay → $MFT → $Bitmap → $BadClus → $AttrDef → $UpCase → optionally $Secure/$Extend/$Reparse/$ObjId → root dentry), `ntfs_put_super` / `ntfs3_put_sbi` / `ntfs3_kill_sb` (tear-down ordering with `ntfs_set_state(NTFS_DIRTY_CLEAR)` + `ntfs_update_mftmirr`), `ntfs_sync_fs`, `ntfs_statfs`, `ntfs_shutdown`, and the NFS-export operations (`ntfs_fh_to_dentry` / `ntfs_fh_to_parent`). Critical for: dual-boot Windows/Linux setups, external NTFS drives, Steam-on-Linux installations on NTFS, automotive/embedded image-mounting.

This Tier-3 covers `fs/ntfs3/super.c` (~1997 lines).

### Acceptance Criteria

- [ ] AC-1: `mount -t ntfs3 /dev/sdX1 /mnt` on a known-good 4 KiB-cluster volume: succeeds; root dentry reachable; `statfs` returns correct cluster size + total/free.
- [ ] AC-2: Per-`NTFS_BOOT` validation: short / wrong-signature / corrupt `bytes_per_sector` / `mft_clst` ≥ sectors / non-pow2 cluster size → mount fails with EINVAL and clear `ntfs_err` message.
- [ ] AC-3: Per-alternative-boot fallback: corrupt primary boot sector but intact last-sector boot → mount succeeds, primary boot is rewritten from alt, log "primary boot is updated".
- [ ] AC-4: Per-2 MiB cluster volume (encoded `sectors_per_clusters = 0xF4..`): `true_sectors_per_clst` recovers correct power-of-2; mount succeeds.
- [ ] AC-5: Per-dirty volume w/o `force`: ro-mount succeeds; rw-mount returns EINVAL "volume is dirty and \"force\" flag is not set!".
- [ ] AC-6: Per-`NTFS_FLAGS_NEED_REPLAY` after `ntfs_loadlog_and_replay` failure: rw-mount returns EINVAL "failed to replay log file. Can't mount rw!". Remount rw→ro→rw with replay still pending → "Couldn't remount rw because journal is not replayed".
- [ ] AC-7: Per-`uid=` / `gid=` / `umask=` / `dmask=` / `fmask=` / `iocharset=` / `nocase` / `sparse` / `showmeta`: appear correctly in `/proc/mounts` (via `ntfs_show_options`).
- [ ] AC-8: Per-`acl` option without `CONFIG_NTFS3_FS_POSIX_ACL`: mount returns EINVAL "Support for ACL not compiled in!".
- [ ] AC-9: Per-shared $UpCase: mount two NTFS volumes concurrently; both share the same `upcase` pointer in memory (verified via debug counter).
- [ ] AC-10: Per-NFS export: `exportfs -o fsid=0 /mnt` + remote-NFS `stat` round-trip works via `ntfs_fh_to_dentry` + `ntfs3_get_parent`.
- [ ] AC-11: Per-`sync(2)` + `ntfs_sync_fs`: cluster-bitmap + $Secure/$ObjId/$Reparse written back; `NTFS_DIRTY_CLEAR` set if no error; `$MFTMirr` updated to mirror MFT[0..recs_mirr).
- [ ] AC-12: Per-umount: `ntfs_put_super` clears `VOLUME_FLAG_DIRTY`; bitmap windows closed; `volume.real_dirty` post-condition matches actual dirty state on disk.
- [ ] AC-13: Per-`shutdown` ioctl: `ntfs_shutdown` sets `NTFS_FLAGS_SHUTDOWN_BIT`; subsequent `sync_fs` returns -EIO; rw I/O fails.
- [ ] AC-14: Per-reparse-point: WSL `IO_REPARSE_TAG_LX_SYMLINK` is readable via readlink(2); `$Extend/$Reparse` index lookup hits.
- [ ] AC-15: Per-`CONFIG_NTFS3_64BIT_CLUSTER` build mounting a 32 TiB volume: succeeds; 32-bit-only build refuses with "is too big to use 32 bits per cluster".

### Architecture

```
struct NtfsBoot {                        // 512 bytes, on-disk
  jump_code:        [u8; 3],
  system_id:        [u8; 8],             // "NTFS    "
  bytes_per_sector: [u8; 2],             // unaligned u16
  sectors_per_clusters: u8,              // 1..0x80 literal; 0xF4..0xFF = 1u<<(-(i8)val)
  unused1:          [u8; 7],
  media_type:       u8,
  unused2:          [u8; 2],
  sct_per_track:    le16,
  heads:            le16,
  hidden_sectors:   le32,
  unused3:          [u8; 4],
  bios_drive_num:   u8,
  unused4:          u8,
  signature_ex:     u8,
  unused5:          u8,
  sectors_per_volume: le64,
  mft_clst:         le64,                // first cluster of $MFT
  mft2_clst:        le64,                // first cluster of $MFTMirr
  record_size:      i8,                  // ≥0 clusters; <0 = 1u<<(-val)
  unused6:          [u8; 3],
  index_size:       i8,
  unused7:          [u8; 3],
  serial_num:       le64,
  check_sum:        le32,
  boot_code:        [u8; 0x200 - 0x50 - 2 - 4],
  boot_magic:       [u8; 2],             // 0x55, 0xAA — non-mandatory
}

struct NtfsSbInfo {
  sb:                  *SuperBlock,
  discard_granularity: u32,
  discard_granularity_mask_inv: u64,
  bdev_blocksize:      u32,

  cluster_size:        u32,              // == sector_size * sct_per_clst
  cluster_mask:        u32,              // cluster_size - 1
  cluster_mask_inv:    u64,
  block_mask:          u32,              // s_blocksize - 1
  blocks_per_cluster:  u32,

  record_size:         u32,              // MFT-record bytes
  index_size:          u32,              // INDX-record bytes
  cluster_bits:        u8,               // log2(cluster_size)
  record_bits:         u8,

  maxbytes:            u64,
  maxbytes_sparse:     u64,
  flags:               AtomicUlong,      // NTFS_FLAGS_*

  zone_max:            Clst,
  bad_clusters:        Clst,

  max_bytes_per_attr:  u16,
  attr_size_tr:        u16,              // resident→non-resident threshold

  objid_no:            Clst,
  quota_no:            Clst,
  reparse_no:          Clst,
  usn_jrnl_no:         Clst,

  def_table:           *AttrDefEntry,    // $AttrDef in memory
  def_entries:         u32,
  ea_max_size:         u32,

  new_rec:             *MftRec,          // template MFT record

  upcase:              *u16,              // 65536 entries, shared

  mft: MftState {
    lbo:                u64,             // $MFT first byte
    lbo2:               u64,             // $MFTMirr first byte
    ni:                 *NtfsInode,
    bitmap:             WndBitmap,       // $MFT::Bitmap
    reserved_bitmap:    ulong,           // records [11..24)
    next_free:          usize,
    used:               usize,
    recs_mirr:          u32,
    next_reserved:      u8,
    reserved_bitmap_inited: u8,
  },

  used: ClusterState {
    bitmap:             WndBitmap,       // $Bitmap::Data
    next_free_lcn:      Clst,
    da:                 AtomicI64,       // delayed-alloc clusters
  },

  volume: VolState {
    size:               u64,
    blocks:             u64,
    ser_num:            u64,
    ni:                 *NtfsInode,
    flags:              le16,            // cached VOLUME_FLAG_DIRTY
    major_ver:          u8,
    minor_ver:          u8,
    label:              [u8; FSLABEL_MAX],
    real_dirty:         bool,
  },

  security:  SecurityState  { index_sii: NtfsIndex, index_sdh: NtfsIndex, ni: *NtfsInode, next_id: u32, next_off: u64, def_security_id: le32 },
  reparse:   ReparseState   { index_r:   NtfsIndex, ni: *NtfsInode, max_size: u64 },  // ≤ 16 KiB
  objid:     ObjidState     { index_o:   NtfsIndex, ni: *NtfsInode },

  compress: CompressState {
    mtx_lznt:  Mutex,  lznt:   *Lznt,
    mtx_xpress: Mutex, xpress: *XpressDecompressor,    // cfg-gated
    mtx_lzx:    Mutex, lzx:    *LzxDecompressor,       // cfg-gated
  },

  options:       *NtfsMountOptions,
  msg_ratelimit: RatelimitState,
  procdir:       Option<*ProcDirEntry>,
}

struct NtfsMountOptions {
  nls_name:        Option<String>,
  nls:             Option<*NlsTable>,
  fs_uid:          Kuid,
  fs_gid:          Kgid,
  fs_fmask_inv:    u16,
  fs_dmask_inv:    u16,
  fmask:           bool,
  dmask:           bool,
  sys_immutable:   bool,
  discard:         bool,
  sparse:          bool,
  showmeta:        bool,
  nohidden:        bool,
  hide_dot_files:  bool,
  windows_names:   bool,
  force:           bool,
  prealloc:        bool,
  nocase:          bool,
  delalloc:        bool,
}

const MFT_REC_MFT:      Clst = 0;
const MFT_REC_MIRR:     Clst = 1;
const MFT_REC_LOG:      Clst = 2;
const MFT_REC_VOL:      Clst = 3;
const MFT_REC_ATTR:     Clst = 4;
const MFT_REC_ROOT:     Clst = 5;
const MFT_REC_BITMAP:   Clst = 6;
const MFT_REC_BOOT:     Clst = 7;
const MFT_REC_BADCLUST: Clst = 8;
const MFT_REC_SECURE:   Clst = 9;
const MFT_REC_UPCASE:   Clst = 10;
const MFT_REC_EXTEND:   Clst = 11;
const MFT_REC_RESERVED: Clst = 12;
const MFT_REC_FREE:     Clst = 16;
const MFT_REC_USER:     Clst = 24;
```

`Ntfs3::init_fs_context(fc) -> Result<()>`:
1. Allocate `NtfsMountOptions` (zeroed).
2. Defaults: fs_uid = current_uid; fs_gid = current_gid; fs_fmask_inv = fs_dmask_inv = !current_umask; prealloc = true.
3. If `CONFIG_NTFS3_FS_POSIX_ACL`: `fc.sb_flags |= SB_POSIXACL`.
4. If `fc.purpose == FS_CONTEXT_FOR_RECONFIGURE`: skip sbi alloc; assign opts; return.
5. Allocate `NtfsSbInfo` (zeroed).
6. Allocate `sbi.upcase` = `kvmalloc(0x10000 * 2)`.
7. `ratelimit_state_init(&sbi.msg_ratelimit, DEFAULT_RATELIMIT_INTERVAL, DEFAULT_RATELIMIT_BURST)`.
8. Init compression mutexes.
9. `fc.s_fs_info = sbi`; `fc.fs_private = opts`; `fc.ops = &NTFS_CONTEXT_OPS`.

`Ntfs3::fill_super(sb, fc) -> Result<()>`:
1. Copy `fc.fs_private` opts into a fresh `options` on `sbi`.
2. Init `sb`: SB_NODIRATIME, magic 0x7366746e, sops, export-ops, time-gran 100 ns, xattr handlers, dentry-ops if `nocase`.
3. Load NLS table (`ntfs_load_nls`); EINVAL if name unknown.
4. Cache discard params from bdev.
5. `init_from_boot(sb, bdev_logical_block_size, bdev_nr_bytes, &boot2)`.
6. iget5 $Volume → cache version/flags/serial; warn-recommend chkdsk if dirty.
7. iget5 $MFTMirr → `mft.recs_mirr`.
8. iget5 $LogFile → `loadlog_and_replay`. Enforce no-rw-mount-with-pending-replay. Enforce no-rw-on-dirty-without-force.
9. iget5 $MFT → load_all_mi → merge bitmap extents → wnd_init(mft.bitmap).
10. iget5 $Bitmap → wnd_init(used.bitmap) → refresh MFT zone.
11. iget5 $BadClus → mark used in used.bitmap; log bad-block count.
12. iget5 $AttrDef → kvmalloc + read; cache reparse.max_size + ea_max_size.
13. iget5 $UpCase → read 128 KiB; byte-swap on BE; `ntfs_set_shared` (de-dup across mounts).
14. If `is_ntfs3` (version ≥ 3.0): security_init (fatal), extend_init (warn), reparse_init (warn), objid_init (warn).
15. iget5 $Root → `d_make_root` → `sb.s_root`.
16. If boot2: rewrite primary boot from alt.
17. `create_procdir`.

`Ntfs3::init_from_boot(sb, sector_size, dev_size, &boot2) -> Result<()>`:
1. `sb_min_blocksize(sb, PAGE_SIZE)`.
2. `boot_block = 0`; `hint = "Primary boot"`.
3. /* read_boot label */ Read 512 B from `boot_block`. Memcmp `"NTFS    "`.
4. Decode `bytes_per_sector` (unaligned u16). Validate ≥ SECTOR_SIZE + pow2.
5. `sct_per_clst = true_sectors_per_clst(boot)`. Validate pow2.
6. `cluster_size = boot_sector_size * sct_per_clst`. Validate sane.
7. `mlcn = boot.mft_clst; mlcn2 = boot.mft2_clst`. Validate `mlcn * sct_per_clst < sectors_per_volume`. Same for mlcn2.
8. Decode `record_size` (signed magic). Validate.
9. Decode `index_size` (signed magic). Validate.
10. Compute `volume.size`. Warn-rdonly on truncated dev. Compute clusters.
11. Reject overflow on 32-bit-LCN build.
12. Init `used.bitmap.nbits = clusters`. Allocate `new_rec` template.
13. `sb_set_blocksize(sb, min(cluster_size, PAGE_SIZE))`. Init blocks_per_cluster + maxbytes.
14. `zone_max = min(0x20000000 >> cluster_bits, clusters >> 3)`.
15. If reading alt-boot success ∧ !rdonly: `boot2 = kmemdup(boot, 0x200, GFP_NOFS | __GFP_NOWARN)`.
16. On EINVAL with `boot_block == 0` ∧ `dev_size > PAGE_SHIFT`: compute alt-boot offset (last sector) and goto read_boot.

`Ntfs3::put_super(sb)`:
1. `remove_procdir(sb)`.
2. `set_state(sbi, NTFS_DIRTY_CLEAR)`.
3. `put_mount_options(sbi.options)`; null it.
4. `put_sbi(sbi)`.

`Ntfs3::put_sbi(sbi)`:
1. `wnd_close(&mft.bitmap)`; `wnd_close(&used.bitmap)`.
2. iput each loaded system inode in order: mft, security, reparse, objid, volume.
3. `update_mftmirr(sbi)` (flush dirty MFT[0..recs_mirr) copies to $MFTMirr at `mft.lbo2`).
4. `indx_clear` security.index_sii / .index_sdh / reparse.index_r / objid.index_o.

`Ntfs3::sync_fs(sb, wait) -> Result<()>`:
1. If shutdown-bit set: -EIO.
2. write_inode each of security.ni / objid.ni / reparse.ni.
3. On all-ok: set_state(NTFS_DIRTY_CLEAR).
4. update_mftmirr.
5. If wait: sync_blockdev + blkdev_issue_flush.

`Ntfs3::reconfigure(fc) -> Result<()>`:
1. ro→rw transition: forbid if NTFS_FLAGS_NEED_REPLAY ("journal not replayed").
2. Swap in new options; free old.

### Out of Scope

- `fs/ntfs3/fslog.c` $LogFile replay state machine (covered separately if expanded to `ntfs3/fslog.md`)
- `fs/ntfs3/inode.c` ntfs_iget5 / read_folio / write paths (covered separately if expanded)
- `fs/ntfs3/attrib.c` resident/non-resident attribute manipulation (covered separately if expanded)
- `fs/ntfs3/index.c` INDX b+-tree directories (covered separately if expanded)
- `fs/ntfs3/bitmap.c` `wnd_bitmap` implementation (covered separately if expanded)
- `fs/ntfs3/{xattr.c,acl.c}` xattr + POSIX ACL plumbing (covered separately)
- `fs/ntfs3/{lznt.c,lib/lzx_decompress.c,lib/xpress_decompress.c}` decompression (covered separately)
- $UsnJrnl change-journal (mostly unused on Linux)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ntfs_sb_info` | per-mount in-core state | `NtfsSbInfo` |
| `struct ntfs_mount_options` | per-mount options | `NtfsMountOptions` |
| `struct NTFS_BOOT` | per-volume on-disk boot (512 B) | UAPI / `NtfsBoot` |
| `struct VOLUME_INFO` | per-$Volume ATTR_VOL_INFO | UAPI / `VolumeInfo` |
| `enum MFT_REC_*` | reserved MFT record indices (0..15) | UAPI / shared |
| `ntfs_init_fs_context()` | fs_context init | `Ntfs3::init_fs_context` |
| `ntfs_fs_parse_param()` | per-param fs_parser callback | `Ntfs3::parse_param` |
| `ntfs_fs_reconfigure()` | per-remount handler | `Ntfs3::reconfigure` |
| `ntfs_fs_get_tree()` | per-mount `get_tree_bdev` wrapper | `Ntfs3::get_tree` |
| `ntfs_fs_free()` | per-fs_context teardown | `Ntfs3::fs_free` |
| `ntfs_fill_super()` | per-mount super_block init | `Ntfs3::fill_super` |
| `ntfs_init_from_boot()` | per-boot-sector decode | `Ntfs3::init_from_boot` |
| `true_sectors_per_clst()` | per-cluster-size decode (>2^7 negated form) | `Ntfs3::true_sectors_per_clst` |
| `ntfs_loadlog_and_replay()` | $LogFile replay (fslog.c) | `Ntfs3::loadlog_and_replay` |
| `ntfs_load_nls()` | iocharset table loader | `Ntfs3::load_nls` |
| `ntfs_put_super()` | super_operations::put_super | `Ntfs3::put_super` |
| `ntfs3_put_sbi()` | per-sbi inode-iput + indx_clear | `Ntfs3::put_sbi` |
| `ntfs3_free_sbi()` | per-sbi free | `Ntfs3::free_sbi` |
| `ntfs3_kill_sb()` | file_system_type::kill_sb | `Ntfs3::kill_sb` |
| `ntfs_sync_fs()` | super_operations::sync_fs | `Ntfs3::sync_fs` |
| `ntfs_statfs()` | super_operations::statfs | `Ntfs3::statfs` |
| `ntfs_show_options()` | super_operations::show_options | `Ntfs3::show_options` |
| `ntfs_shutdown()` | super_operations::shutdown | `Ntfs3::shutdown` |
| `ntfs_alloc_inode()` / `ntfs_free_inode()` | per-inode slab alloc | `Ntfs3::alloc_inode` / `free_inode` |
| `ntfs_iget5()` | per-MFT-ref inode lookup | `Ntfs3::iget5` |
| `ntfs_set_state()` | $Volume VOLUME_FLAG_DIRTY toggle | `Ntfs3::set_state` |
| `ntfs_update_mftmirr()` | $MFTMirr write-back of MFT[0..recs_mirr) | `Ntfs3::update_mftmirr` |
| `ntfs_set_shared()` / `ntfs_put_shared()` | per-$UpCase global de-dup (one 128 KiB upcase table shared across all NTFS mounts) | `Ntfs3::set_shared` / `put_shared` |
| `ntfs_fh_to_dentry()` / `ntfs_fh_to_parent()` | NFS export-ops | `Ntfs3::fh_to_dentry` / `fh_to_parent` |
| `ntfs_nfs_commit_metadata()` | NFS commit | `Ntfs3::nfs_commit_metadata` |
| `ntfs_security_init()` | $Secure / SDS+SDH+SII | `Ntfs3::security_init` |
| `ntfs_extend_init()` / `ntfs_reparse_init()` / `ntfs_objid_init()` | $Extend / $Extend/$Reparse / $Extend/$ObjId | `Ntfs3::extend_init` / `reparse_init` / `objid_init` |
| `wnd_init()` / `wnd_close()` | per-bitmap window (used.bitmap + mft.bitmap) | `WndBitmap::init` / `close` |
| `ntfs_unmap_meta()` | per-(lcn,len) unmap_underlying_metadata | `Ntfs3::unmap_meta` |
| `ntfs_discard()` | per-(lcn,len) blkdev_issue_discard | `Ntfs3::discard` |

### compatibility contract

REQ-1: `struct NTFS_BOOT` (512 bytes — `static_assert(sizeof(struct NTFS_BOOT) == 0x200)`):
- `jump_code[3]` (0x00): x86 jump.
- `system_id[8]` (0x03): must equal `"NTFS    "` (8 chars incl. trailing spaces).
- `bytes_per_sector[2]` (0x0B): unaligned little-endian u16. Must be ≥ `SECTOR_SIZE` and power-of-2.
- `sectors_per_clusters` (0x0D): 1..0x80 = literal; 0xF4..0xFF = `1U << (-(s8)val)` (encodes >128 sectors/cluster up to 2 MiB).
- `media_type` (0x15): 0xF8 = hard disk.
- `sct_per_track`, `heads`, `hidden_sectors` (0x18..0x1F): geometry hints.
- `bios_drive_num` (0x24): 0x80.
- `signature_ex` (0x26): 0x80.
- `sectors_per_volume` (0x28): __le64 total sectors.
- `mft_clst` (0x30): __le64 first cluster of $MFT.
- `mft2_clst` (0x38): __le64 first cluster of $MFTMirr.
- `record_size` (0x40): signed; ≥ 0 = clusters; < 0 = `1u << -val` (encodes < 1 cluster MFT-record size). Must be ≥ SECTOR_SIZE, power-of-2, ≤ `MAXIMUM_BYTES_PER_MFT`.
- `index_size` (0x44): same encoding as record_size; ≤ `MAXIMUM_BYTES_PER_INDEX`.
- `serial_num` (0x48): __le64 volume serial (exposed via `statfs.f_fsid`).
- `check_sum` (0x50): additive u32 checksum.
- `boot_code[]` (0x54..0x1FD): x86 boot code.
- `boot_magic[2]` (0x1FE): 0x55 0xAA (per upstream comment, this is non-mandatory — chkdsk skips it).

REQ-2: `struct ntfs_sb_info` (in-core mount state — `ntfs_fs.h:211`):
- `sb`: back-ptr to `super_block`.
- `discard_granularity`, `discard_granularity_mask_inv`, `bdev_blocksize`.
- `cluster_size`, `cluster_mask`, `cluster_mask_inv`, `cluster_bits` (log2), `block_mask`, `blocks_per_cluster`.
- `record_size`, `record_bits` (MFT-record size in bytes / log2).
- `index_size` (index record size).
- `maxbytes` (max normal file size), `maxbytes_sparse` (max sparse file size).
- `flags`: `NTFS_FLAGS_NODISCARD | _SHUTDOWN_BIT | _LOG_REPLAYING | _MFTMIRR | _NEED_REPLAY`.
- `zone_max` (per-MFT-zone clusters cap, `min(0x20000000 >> cluster_bits, clusters >> 3)` = ≤ 512 MiB ∧ ≤ ⅛ volume).
- `bad_clusters`.
- `max_bytes_per_attr` (max resident attribute size in MFT record).
- `attr_size_tr` (resident→non-resident threshold ≈ `5 * record_size / 16`).
- `objid_no` / `quota_no` / `reparse_no` / `usn_jrnl_no` (per-$Extend record indices).
- `def_table` (per-$AttrDef table copy in-memory), `def_entries`, `ea_max_size`.
- `new_rec` (template MFT record buffer for new allocations).
- `upcase` (256 KiB / 65536 u16 — Unicode upcase table, shared across mounts).
- `mft`: { `lbo` (LBO of $MFT), `lbo2` (LBO of $MFTMirr), `ni` (`struct ntfs_inode *` for $MFT), `bitmap` (per-$MFT::Bitmap `wnd_bitmap`), `reserved_bitmap` (records [11..24) reserved for MFT expansion), `next_free`, `used` (valid MFT records), `recs_mirr` (#records covered by $MFTMirr), `next_reserved`, `reserved_bitmap_inited` }.
- `used`: { `bitmap` (per-$Bitmap::Data `wnd_bitmap`), `next_free_lcn`, `da` (atomic delayed-alloc counter) }.
- `volume`: { `size` (bytes), `blocks`, `ser_num`, `ni` (for $Volume), `flags` (cached `VOLUME_FLAG_DIRTY` bit), `major_ver`, `minor_ver`, `label[FSLABEL_MAX]`, `real_dirty` (true once we touched dirty state) }.
- `security`: { `index_sii`, `index_sdh`, `ni` ($Secure), `next_id`, `next_off`, `def_security_id` }.
- `reparse`: { `index_r`, `ni` ($Extend/$Reparse), `max_size` (16 KiB cap) }.
- `objid`: { `index_o`, `ni` ($Extend/$ObjId) }.
- `compress`: per-mutex protected `lznt` / `xpress` / `lzx` decompressors (LZX+Xpress only if `CONFIG_NTFS3_LZX_XPRESS`).
- `options`: `*ntfs_mount_options`.
- `msg_ratelimit` (`ntfs_printk` ratelimit).
- `procdir` (`/proc/fs/ntfs3/<dev>/`).

REQ-3: `struct ntfs_mount_options` (per-mount user-tunables):
- `nls_name` / `nls` (per-iocharset NLS table; NULL == utf8).
- `fs_uid` / `fs_gid` (kuid/kgid for file owner overrides).
- `fs_fmask_inv` / `fs_dmask_inv` (inverted file/dir permission masks).
- Bitfields: `fmask`/`dmask` (mask was set), `sys_immutable`, `discard`, `sparse`, `showmeta` (show $MFT/$Bitmap/etc.), `nohidden`, `hide_dot_files`, `windows_names` (reject names Windows would reject), `force` (mount-rw even if dirty), `prealloc`, `nocase`, `delalloc`.

REQ-4: `enum Opt` + `ntfs_fs_parameters[]` — `fs_parameter_spec` table:
- `uid` / `gid` (`fsparam_uid` / `_gid`).
- `umask` / `dmask` / `fmask` (`fsparam_u32oct`, validates value & ~07777 == 0).
- `sys_immutable` / `discard` / `force` / `sparse` / `nohidden` / `hide_dot_files` / `windows_names` / `showmeta` / `nocase` (`fsparam_flag`).
- `acl` (`fsparam_flag` + `fsparam_bool`: enables `SB_POSIXACL` iff `CONFIG_NTFS3_FS_POSIX_ACL`; otherwise returns -EINVAL "Support for ACL not compiled in!").
- `iocharset` (`fsparam_string`: takes ownership of param->string).
- `prealloc` (`fsparam_flag` + `fsparam_bool`).
- `delalloc` (`fsparam_flag` + `fsparam_bool`).
- Per-param: `fs_parse(fc, ntfs_fs_parameters, param, &result)`; switch on result; set opt-field; -EINVAL on unknown.

REQ-5: `ntfs_init_fs_context(fc)`:
- Allocate `struct ntfs_mount_options` via `kzalloc_obj`; ENOMEM if fail.
- Defaults: `fs_uid = current_uid()`, `fs_gid = current_gid()`, `fs_fmask_inv = ~current_umask()`, `fs_dmask_inv = ~current_umask()`, `prealloc = 1`.
- `CONFIG_NTFS3_FS_POSIX_ACL`: set `fc->sb_flags |= SB_POSIXACL` (default ACL on).
- If `fc->purpose == FS_CONTEXT_FOR_RECONFIGURE`: skip sbi alloc (just remount with new opts).
- Else: allocate `struct ntfs_sb_info` via `kzalloc_obj`. Allocate `sbi->upcase` as 0x10000 × u16 (128 KiB) via `kvmalloc`. `ratelimit_state_init(&sbi->msg_ratelimit, DEFAULT_RATELIMIT_INTERVAL, DEFAULT_RATELIMIT_BURST)`. `mutex_init(&sbi->compress.mtx_lznt)` (+ xpress/lzx if LZX_XPRESS).
- `fc->s_fs_info = sbi`; `fc->fs_private = opts`; `fc->ops = &ntfs_context_ops`.

REQ-6: `ntfs_fs_get_tree(fc) -> get_tree_bdev(fc, ntfs_fill_super)`.

REQ-7: `ntfs_fs_free(fc)`:
- If `sbi` present: `ntfs3_put_sbi(sbi)` + `ntfs3_free_sbi(sbi)`.
- If `opts` present: `put_mount_options(opts)`.
- Called after fill_super/reconfigure even on success (callees must steal pointers if keeping them).

REQ-8: `ntfs_init_from_boot(sb, sector_size, dev_size, &boot2)`:
- /* Save original dev_size for alt-boot fallback */ `dev_size0 = dev_size`.
- /* Initial blocksize for reading boot */ `sb_min_blocksize(sb, PAGE_SIZE)` — return -EINVAL on fail.
- /* read_boot label */ `bh = ntfs_bread(sb, boot_block)` (boot_block initially 0).
- Verify `memcmp(boot->system_id, "NTFS    ", 8) == 0`; else log "Primary boot signature is not NTFS" + EINVAL.
- `boot_sector_size` = unaligned `bytes_per_sector` u16 → must be ≥ SECTOR_SIZE + power-of-2.
- `sct_per_clst = true_sectors_per_clst(boot)`: ≤ 0x80 → literal; ≥ 0xF4 → `1U << -val` (up to 2 MiB); else -EINVAL. Validate power-of-2.
- `cluster_size = boot_sector_size * sct_per_clst`; `cluster_bits = blksize_bits(cluster_size)`.
- `mlcn = le64(boot->mft_clst)`, `mlcn2 = le64(boot->mft2_clst)`, `sectors = le64(boot->sectors_per_volume)`. Reject if `mlcn * sct_per_clst ≥ sectors` ∨ `mlcn2 * sct_per_clst ≥ sectors` (MFT past EOV).
- `record_size`: positive = `<<cluster_bits`; negative ≤ `MAXIMUM_SHIFT_BYTES_PER_MFT` = `1u << -val`; validate SECTOR_SIZE ≤ size, power-of-2, ≤ `MAXIMUM_BYTES_PER_MFT`.
- `index_size`: same encoding; ≤ `MAXIMUM_BYTES_PER_INDEX`.
- `volume.size = sectors * boot_sector_size`.
- If `boot_sector_size != sector_size` (volume formatted with different sector size than current media): warn + adjust dev_size up by sector_size-1.
- `bdev_blocksize = max(boot_sector_size, sector_size)`.
- `mft.lbo = mlcn << cluster_bits`; `mft.lbo2 = mlcn2 << cluster_bits`.
- Reject if `cluster_size < boot_sector_size`.
- Reject if `cluster_size < sector_size` ("no way to use ntfs_get_block").
- `max_bytes_per_attr = record_size - ALIGN(MFTRECORD_FIXUP_OFFSET, 8) - ALIGN((record_size >> SECTOR_SHIFT) * 2, 8) - ALIGN(sizeof(ATTR_TYPE), 8)`.
- `volume.ser_num = le64(boot->serial_num)`.
- If `dev_size < volume.size + boot_sector_size` (RAW: dev smaller than the volume's recorded size): warn + force `SB_RDONLY`.
- `clusters = volume.size >> cluster_bits`. Without `CONFIG_NTFS3_64BIT_CLUSTER`: reject if `clusters >> 32 != 0` (32-bit LCN overflow).
- `used.bitmap.nbits = clusters`.
- Allocate `new_rec` template MFT record: `record_size` zeroed; fill `rhdr.sign = NTFS_FILE_SIGNATURE` ("FILE"), `fix_off`, `fix_num`, `attr_off`, `used`, `total`, terminator ATTRIB at `attr_off`.
- `sb_set_blocksize(sb, min(cluster_size, PAGE_SIZE))`. `block_mask = blocksize - 1`. `blocks_per_cluster = cluster_size >> blocksize_bits`.
- `maxbytes`: 64-bit-LCN build → `MAX_LFS_FILESIZE`; 32-bit build → `(clusters << cluster_bits) - 1`; `maxbytes_sparse = (1ULL << (cluster_bits + 32)) - 1`. `sb->s_maxbytes = 0xFFFFFFFFull << cluster_bits` on 32-bit-LCN.
- `zone_max = min(0x20000000 >> cluster_bits, clusters >> 3)`.
- If we read the alt-boot (`bh->b_blocknr != 0`) and not read-only → `boot2 = kmemdup(boot, sizeof(*boot), GFP_NOFS | __GFP_NOWARN)` (primary will be re-written from alt later by `ntfs_fill_super`).
- /* out path */ `brelse(bh)`. If `err == -EINVAL ∧ !boot_block ∧ dev_size0 > PAGE_SHIFT`: compute `lbo = dev_size0 - sizeof(*boot)`; `boot_block = lbo >> blksize_bits(block_size)`; `boot_off = lbo & (block_size - 1)`; if valid → goto `read_boot` (try alternative boot at last sector).

REQ-9: `ntfs_fill_super(sb, fc)` — the canonical mount-flow:
- `sbi->sb = sb`. `fc_opts = fc->fs_private`; must be non-NULL.
- `options = kmemdup(fc_opts, sizeof(*fc_opts), GFP_KERNEL)`; also dup nls_name string.
- `sbi->options = options`. `sb->s_flags |= SB_NODIRATIME`. `sb->s_magic = 0x7366746e` ("ntfs"). `sb->s_op = &ntfs_sops`. `sb->s_export_op = &ntfs_export_ops`. `sb->s_time_gran = NTFS_TIME_GRAN` (100 ns). `sb->s_xattr = ntfs_xattr_handlers`. `set_default_d_op(sb, nocase ? &ntfs_dentry_ops : NULL)`.
- `options->nls = ntfs_load_nls(options->nls_name)`; if PTR_ERR → "Cannot load nls" + EINVAL.
- If `bdev_max_discard_sectors ∧ bdev_discard_granularity`: cache discard_granularity + discard_granularity_mask_inv.
- /* Parse boot */ `err = ntfs_init_from_boot(sb, bdev_logical_block_size(bdev), bdev_nr_bytes(bdev), &boot2)`.
- /* Load $Volume (MFT_REC_VOL = 3) — before $LogFile because `sbi->volume.ni` is needed by `ntfs_set_state` */: `ref.low = MFT_REC_VOL; ref.seq = MFT_REC_VOL; inode = ntfs_iget5(sb, &ref, &NAME_VOLUME)`. Find ATTR_LABEL → utf16-to-utf8 into `sbi->volume.label`. Find ATTR_VOL_INFO → cache `major_ver` / `minor_ver` / `flags` (`VOLUME_FLAG_DIRTY`). If `VOLUME_FLAG_DIRTY`: `real_dirty = true` + log "recommend chkdsk".
- /* Load $MFTMirr (MFT_REC_MIRR = 1) */: `inode = ntfs_iget5(sb, ..., &NAME_MIRROR)`. `mft.recs_mirr = ntfs_up_cluster(sbi, inode->i_size) >> record_bits`. iput.
- /* Load $LogFile (MFT_REC_LOG = 2) + replay */: `inode = ntfs_iget5(..., &NAME_LOGFILE)`. `ntfs_loadlog_and_replay(ni, sbi)` (fslog.c performs LSN scan + redo + undo). iput.
- If `(sbi->flags & NTFS_FLAGS_NEED_REPLAY) ∧ !ro`: "failed to replay log file. Can't mount rw!" + EINVAL.
- If `(volume.flags & VOLUME_FLAG_DIRTY) ∧ !ro ∧ !options->force`: "volume is dirty and \"force\" flag is not set!" + EINVAL.
- /* Load $MFT (MFT_REC_MFT = 0, seq = 1) */: `inode = ntfs_iget5(sb, ..., &NAME_MFT)`. `mft.used = ni->i_valid >> record_bits`. `tt = i_size >> record_bits`. `mft.next_free = MFT_REC_USER` (= 24, first user record index).
- `ni_load_all_mi(ni)` (load all $MFT extents from ATTR_LIST).
- Merge $MFT::Bitmap runs from extent records (ni_enum_attr_ex → for each non-resident ATTR_BITMAP with svcn != 0: `run_unpack_ex(&mft.bitmap.run, sbi, MFT_REC_MFT, svcn, evcn, ...)`).
- `wnd_init(&sbi->mft.bitmap, sb, tt)`.
- `sbi->mft.ni = ni`.
- /* Load $Bitmap (MFT_REC_BITMAP = 6) */: `inode = ntfs_iget5(..., &NAME_BITMAP)`. Without 64-bit-LCN: reject if `i_size >> 32 != 0`. Sanity: `i_size ≥ ntfs3_bitmap_size(clusters)`. `wnd_init(&sbi->used.bitmap, sb, clusters)`. iput.
- `ntfs_refresh_zone(sbi)` (compute MFT-zone window in the cluster bitmap).
- /* Load $BadClus (MFT_REC_BADCLUST = 8) */: `inode = ntfs_iget5(..., &NAME_BADCLUS)`. For each (vcn, lcn, len) run with `lcn != SPARSE_LCN`: accumulate bad_len + bad_frags; if !ro and `wnd_set_used_safe(&used.bitmap, lcn, len, &tt) || tt`: mark `NTFS_DIRTY_ERROR`. Log "%zu bad blocks in %zu fragments". iput.
- /* Load $AttrDef (MFT_REC_ATTR = 4) */: `inode = ntfs_iget5(..., &NAME_ATTRDEF)`. Sanity: `sizeof(ATTR_DEF_ENTRY) ≤ i_size ≤ 100 × sizeof(ATTR_DEF_ENTRY)`. `def_table = kvmalloc(bytes)`. `inode_read_data(inode, def_table, bytes)`. First entry must be `ATTR_STD`. Iterate entries; pick out `ATTR_REPARSE`'s max_sz → `reparse.max_size`; `ATTR_EA`'s max_sz → `ea_max_size`. `def_entries = N`. iput.
- /* Load $UpCase (MFT_REC_UPCASE = 10) */: must be exactly `0x10000 * sizeof(short)` = 128 KiB. `inode_read_data(inode, sbi->upcase, ...)`. On __BIG_ENDIAN: `__swab16s` each u16. `shared = ntfs_set_shared(sbi->upcase, 128 KiB)`: if another mount already had the identical upcase, drop ours via `kvfree` and adopt the shared pointer. iput.
- /* If `is_ntfs3(sbi)` (volume version ≥ 3.0): */
  - `ntfs_security_init(sbi)` — fatal on err (load $Secure/SDS+SDH+SII indices).
  - `ntfs_extend_init(sbi)` — warn-only ("Failed to initialize $Extend"); goto load_root.
  - `ntfs_reparse_init(sbi)` — warn-only; goto load_root.
  - `ntfs_objid_init(sbi)` — warn-only; goto load_root.
- /* load_root */: `inode = ntfs_iget5(sb, &(MFT_REC_ROOT=5, seq=5), &NAME_ROOT)`. `sb->s_root = d_make_root(inode)`. Fail with ENOMEM if d_make_root returns NULL.
- /* If primary boot was bad but alternative boot was good (boot2 set): rewrite primary boot now that the volume is recognized as NTFS */: `bh0 = sb_getblk(sb, 0); memcpy(bh0->b_data, boot2, 0x200); mark_buffer_dirty; sync_dirty_buffer`. kfree(boot2).
- `ntfs_create_procdir(sb)` (per-mount `/proc/fs/ntfs3/<bdev>/` with `volinfo` + `label` files, if `CONFIG_PROC_FS`).
- Return 0.
- /* error paths: put_inode_out / out */ — iput inode then `put_mount_options(options)` + `ntfs3_put_sbi(sbi)` + kfree(boot2).

REQ-10: `ntfs_fs_reconfigure(fc)`:
- If transitioning `ro → rw` and `sbi->flags & NTFS_FLAGS_NEED_REPLAY`: error "Couldn't remount rw because journal is not replayed. Please umount/remount instead."
- Replace `sbi->options` with new opts (kmemdup); free old.

REQ-11: `ntfs_put_super(sb)` (super_operations::put_super):
- `ntfs_remove_procdir(sb)`.
- `ntfs_set_state(sbi, NTFS_DIRTY_CLEAR)` — clear `VOLUME_FLAG_DIRTY` if we set it.
- `put_mount_options(sbi->options)`; `sbi->options = NULL`.
- `ntfs3_put_sbi(sbi)`.

REQ-12: `ntfs3_put_sbi(sbi)` (per-sbi resource release):
- `wnd_close(&mft.bitmap)`; `wnd_close(&used.bitmap)`.
- `iput(&mft.ni->vfs_inode)` (if loaded); same for security / reparse / objid / volume.
- `ntfs_update_mftmirr(sbi)` (write back any pending $MFTMirr copies of MFT[0..recs_mirr) records).
- `indx_clear(&security.index_sii / .index_sdh / reparse.index_r / objid.index_o)`.

REQ-13: `ntfs3_kill_sb(sb)` (file_system_type::kill_sb):
- `kill_block_super(sb)` (VFS standard tear-down).
- `put_mount_options(sbi->options)` (if surviving — sbi may have been left if mount failed late).
- `ntfs3_free_sbi(sbi)` (free `new_rec`, drop shared upcase via `ntfs_put_shared`, free `def_table`, free lznt/xpress/lzx decompressors, kfree sbi).

REQ-14: `ntfs_sync_fs(sb, wait)` (super_operations::sync_fs):
- If `ntfs3_forced_shutdown(sb)`: return -EIO.
- For each of `security.ni`, `objid.ni`, `reparse.ni`: `_ni_write_inode(inode, wait)`.
- On success: `ntfs_set_state(sbi, NTFS_DIRTY_CLEAR)`.
- `ntfs_update_mftmirr(sbi)` always.
- If `wait`: `sync_blockdev(sb->s_bdev)` + `blkdev_issue_flush(sb->s_bdev)`.

REQ-15: `ntfs_statfs(dentry, buf)`:
- `f_type = sb->s_magic` (= 0x7366746e).
- `f_bsize = f_frsize = cluster_size`.
- `f_blocks = used.bitmap.nbits` (total clusters).
- `f_bfree = wnd_zeroes(&used.bitmap) - ntfs_get_da(sbi)` (subtract delayed-alloc reservation; floor 0).
- `f_bavail = f_bfree`.
- `f_fsid.val[0/1] = volume.ser_num low/high u32`.
- `f_namelen = NTFS_NAME_LEN` (255).

REQ-16: `ntfs_show_options(m, root)`:
- Emit `,uid=%u,gid=%u` (user-ns translated).
- If `dmask`: `,dmask=%04o` (XOR 0xffff to recover original from inverted form). Same for `fmask`.
- Flags echoed when set: `sys_immutable`, `discard`, `force`, `sparse`, `nohidden`, `hide_dot_files`, `windows_names`, `showmeta`.
- `,acl` if `SB_POSIXACL`.
- `,iocharset=<name>` (or `=utf8` if no nls).
- `prealloc` / `nocase` / `delalloc` if set.

REQ-17: Reparse-point support (per-`ntfs_reparse_init`):
- $Extend/$Reparse is an INDEX with per-(reparse_tag, MFT-ref) entries.
- `reparse.max_size` capped at 16 KiB (per `$AttrDef[ATTR_REPARSE].max_sz`).
- Per-inode with `FILE_ATTRIBUTE_REPARSE_POINT`: `ATTR_REPARSE` holds REPARSE_POINT header + per-tag data.
- Tags handled by file/inode code (not super.c): `IO_REPARSE_TAG_SYMLINK`, `IO_REPARSE_TAG_MOUNT_POINT`, `IO_REPARSE_TAG_WOF` (Windows-overlay-filter), WSL tags (`IO_REPARSE_TAG_LX_SYMLINK`, `_LX_FIFO`, `_LX_CHR`, `_LX_BLK`), etc.

REQ-18: NFS-export operations (`ntfs_export_ops`):
- `.encode_fh = generic_encode_ino32_fh`.
- `.fh_to_dentry = ntfs_fh_to_dentry` → `generic_fh_to_dentry(sb, fid, fh_len, fh_type, ntfs_export_get_inode)`.
- `.fh_to_parent = ntfs_fh_to_parent` → `generic_fh_to_parent(...)`.
- `.get_parent = ntfs3_get_parent`.
- `.commit_metadata = ntfs_nfs_commit_metadata` → `_ni_write_inode(inode, 1)`.
- `ntfs_export_get_inode`: build `struct MFT_REF` { low = ino & 0xffffffff, high = (ino>>32) if 64-bit-LCN else 0, seq = generation }, then `ntfs_iget5`. Map bad-inode to ESTALE.

REQ-19: Module init/exit (`init_ntfs_fs` / `exit_ntfs_fs`):
- Print enabled-features banner: POSIX_ACL, 64BIT_CLUSTER, LZX_XPRESS.
- `ntfs_create_proc_root()` → `/proc/fs/ntfs3/`.
- `ntfs3_init_bitmap()` (bitmap.c slab/global state).
- `kmem_cache_create("ntfs_inode_cache", sizeof(struct ntfs_inode), 0, SLAB_RECLAIM_ACCOUNT | SLAB_ACCOUNT, init_once)`.
- `register_filesystem(&ntfs_fs_type)`.
- Exit: `rcu_barrier()` → kmem_cache_destroy → unregister → `ntfs3_exit_bitmap` → remove proc root.

REQ-20: `struct file_system_type ntfs_fs_type`:
- `.owner = THIS_MODULE`.
- `.name = "ntfs3"`.
- `.init_fs_context = ntfs_init_fs_context`.
- `.parameters = ntfs_fs_parameters`.
- `.kill_sb = ntfs3_kill_sb`.
- `.fs_flags = FS_REQUIRES_DEV | FS_ALLOW_IDMAP`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `boot_signature_checked` | INVARIANT | per-init_from_boot: NTFS signature memcmp executed before any field is trusted. |
| `mft_lcn_within_volume` | INVARIANT | per-init_from_boot: `mft_clst * sct_per_clst < sectors_per_volume` ∧ same for mft2_clst. |
| `cluster_size_pow2` | INVARIANT | per-init_from_boot: cluster_size ≥ sector_size ∧ is_power_of_2. |
| `record_size_pow2` | INVARIANT | per-init_from_boot: record_size ≥ SECTOR_SIZE ∧ ≤ MAXIMUM_BYTES_PER_MFT ∧ is_power_of_2. |
| `upcase_buffer_128k` | INVARIANT | per-fill_super: upcase read exactly `0x10000 * 2` bytes. |
| `iput_balanced_on_fail` | INVARIANT | per-fill_super error paths: every successfully ntfs_iget5'd inode is iput'd. |
| `boot2_freed` | INVARIANT | per-fill_super: boot2 kfree'd on every path including error. |
| `replay_required_for_rw` | INVARIANT | per-fill_super: NEED_REPLAY ∧ !ro ⟹ -EINVAL before mount completes. |
| `dirty_required_force_for_rw` | INVARIANT | per-fill_super: VOLUME_FLAG_DIRTY ∧ !ro ∧ !force ⟹ -EINVAL. |
| `shared_upcase_refcount_balanced` | INVARIANT | per-set_shared/put_shared: ref get/put across mount lifecycle. |
| `clusters_fit_32bit_unless_64bit_cfg` | INVARIANT | per-init_from_boot: `!CONFIG_NTFS3_64BIT_CLUSTER ⟹ clusters >> 32 == 0`. |

### Layer 2: TLA+

`fs/ntfs3/mount.tla`:
- Per-mount lifecycle: init_fs_context → parse_param* → get_tree → init_from_boot → (load $Volume → $MFTMirr → $LogFile → replay → $MFT → $Bitmap → $BadClus → $AttrDef → $UpCase → [$Secure / $Extend / $Reparse / $ObjId] → root) → ok | error.
- Per-error injection at each step: pre-allocated state unwound (iput / kfree / wnd_close) on every error path.
- Properties:
  - `safety_no_partial_mount_observable` — never returns 0 with sbi only half-populated.
  - `safety_no_rw_with_pending_replay` — sb.s_flags has SB_RDONLY whenever NTFS_FLAGS_NEED_REPLAY is set after replay attempt.
  - `safety_dirty_clear_only_on_clean_unmount` — put_super calls set_state(DIRTY_CLEAR) exactly once.
  - `safety_alt_boot_falls_back_only_on_primary_fail` — alt-boot read only attempted if primary returned EINVAL.
  - `liveness_per_mount_terminates` — fill_super either returns ok or rolls back fully.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ntfs3::init_fs_context` post: opts + sbi allocated ∨ -ENOMEM | `Ntfs3::init_fs_context` |
| `Ntfs3::init_from_boot` post: cluster_size/record_size/index_size all pow2 ∧ within bounds | `Ntfs3::init_from_boot` |
| `Ntfs3::fill_super` post: ok ⟹ sb.s_root != null ∧ mft.ni != null ∧ used.bitmap.inited | `Ntfs3::fill_super` |
| `Ntfs3::put_sbi` post: all bitmap windows closed + all system-inodes iput'd | `Ntfs3::put_sbi` |
| `Ntfs3::sync_fs` post: $MFTMirr written; if wait → bdev flushed | `Ntfs3::sync_fs` |
| `Ntfs3::reconfigure` post: ro→rw forbidden when NEED_REPLAY | `Ntfs3::reconfigure` |
| `Ntfs3::parse_param Opt_acl` post: SB_POSIXACL set iff CONFIG_NTFS3_FS_POSIX_ACL | `Ntfs3::parse_param` |

### Layer 4: Verus/Creusot functional

`Per-mount on-disk flow → boot decode → system-inode load (in mandated order: $Volume, $MFTMirr, $LogFile+replay, $MFT, $Bitmap, $BadClus, $AttrDef, $UpCase, $Secure?, $Extend?/$Reparse?/$ObjId?) → root dentry`: semantic equivalence with `Documentation/filesystems/ntfs3.rst` + Paragon spec.

### hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

ntfs3-superblock-specific reinforcement:

- **Per-`memcmp("NTFS    ", 8)` before any field is trusted** — defense against per-arbitrary-block-mounted-as-NTFS.
- **Per-`is_power_of_2` checks on cluster_size, record_size, index_size, bytes_per_sector** — defense against per-corrupt-boot integer-overflow.
- **Per-`mft_clst * sct_per_clst < sectors_per_volume` bounds-check** — defense against per-MFT-LCN-overflow OOB read.
- **Per-alt-boot fallback gated to `boot_block == 0` first** — defense against per-infinite-retry on persistent corruption.
- **Per-`MAXIMUM_BYTES_PER_MFT` / `MAXIMUM_BYTES_PER_INDEX` upper bounds** — defense against per-record-size DoS.
- **Per-`!CONFIG_NTFS3_64BIT_CLUSTER ⟹ clusters fit in 32 bits`** — defense against per-LCN-truncation silent corruption on > 16 TB volumes.
- **Per-`force` required to mount rw on `VOLUME_FLAG_DIRTY`** — defense against per-replay-bug data loss.
- **Per-NEED_REPLAY blocks rw mount** — defense against per-uncommitted-log-overwrite.
- **Per-`SB_RDONLY` forced when `dev_size < volume.size + boot_sector_size`** — defense against per-truncated-image data loss.
- **Per-`umask & ~07777` validation** — defense against per-mode-bits-poisoning.
- **Per-iocharset must resolve via `load_nls`** — defense against per-unknown-charset mount + later utf16 conversion crash.
- **Per-error rollback iputs every loaded system-inode and `wnd_close`s every initialized bitmap** — defense against per-half-mount UAF.
- **Per-`ntfs_set_state(NTFS_DIRTY_CLEAR)` only on clean put_super / successful sync_fs** — defense against per-spurious-clean state on crash.
- **Per-`ntfs_set_shared` + `ntfs_put_shared` refcount on $UpCase** — defense against per-double-free of the 128 KiB shared upcase table.
- **Per-`msg_ratelimit` on `ntfs_printk`** — defense against per-corrupt-volume log flood.

