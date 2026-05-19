# Tier-3: fs/zonefs/super.c — Zonefs superblock and zone-as-file mount

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/zonefs/00-overview.md
upstream-paths:
  - fs/zonefs/super.c (~1472 lines)
  - fs/zonefs/zonefs.h
-->

## Summary

Zonefs is a **zone-as-file** filesystem for zoned block devices (HM-SMR HDDs, ZBC, ZAC, NVMe ZNS). Each device zone is exposed as exactly one regular file with a fixed-size, fixed-on-disk-location identity that never moves; the FS has no namespace operations beyond lookup (no `create`/`unlink`/`rename`/`mkdir`). Per-device layout: zone 0 holds the **zonefs_super** (4096-byte super block: s_magic 'Z'/'F'/'S'/0x53, s_version, s_features bitmap, s_uuid, optional s_uid/s_gid/s_perm, s_label, s_crc32). Per-mount layout: a 2-level read-only directory tree with root `/`, `cnv/` (conventional zones, may be aggregated under `ZONEFS_F_AGGRCNV`), and `seq/` (sequential-write zones — SWR/SWP). Per-zone-file: `<zone-group>/<N>` where N is the in-group ordinal as base-10 with no leading zeros; file size == zone write-pointer offset (sequential zones) or zone capacity (conventional / aggregated). Per-zone-state-machine: cooperates with the block layer's per-zone `BLK_ZONE_COND_*` (NOT_WP, EMPTY, IMP_OPEN, EXP_OPEN, CLOSED, FULL, READONLY, OFFLINE) via `blkdev_zone_mgmt(REQ_OP_ZONE_OPEN/CLOSE/RESET/FINISH)`. Per-`errors=` policy chooses how IO errors degrade the inode: `remount-ro` (default — remount fs RO), `zone-ro`, `zone-offline`, `repair`. Per-`explicit-open`: when the device has `max_active_zones` / `max_open_zones`, zonefs explicitly opens a zone on file open and closes it on close to share resources fairly. Critical for: zoned-block-device first-class support, sequential-write-only enforcement, deterministic per-zone IO behaviour.

This Tier-3 covers `fs/zonefs/super.c` (~1472 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct zonefs_super` | per-on-disk super block | `ZonefsOnDiskSuper` |
| `struct zonefs_sb_info` | per-mount in-core sb | `ZonefsSbInfo` |
| `struct zonefs_inode_info` | per-inode in-core | `ZonefsInodeInfo` |
| `struct zonefs_zone` | per-zone in-core descriptor | `ZonefsZone` |
| `struct zonefs_zone_group` | per-ztype zone array + dir-inode | `ZonefsZoneGroup` |
| `struct zonefs_context` | per-fs_context options | `ZonefsContext` |
| `struct zonefs_zone_data` | per-mount-time zone scratch | `ZonefsZoneData` |
| `enum zonefs_ztype` | CNV / SEQ | `ZonefsZType` |
| `zonefs_fill_super()` | per-mount | `ZonefsSb::fill_super` |
| `zonefs_read_super()` | per-on-disk-super parse + CRC | `ZonefsSb::read_super` |
| `zonefs_init_zgroups()` | per-zone-report + group init | `ZonefsSb::init_zgroups` |
| `zonefs_init_zgroup()` | per-ztype group init | `ZonefsSb::init_zgroup` |
| `zonefs_get_zone_info()` | per-blkdev_report_zones | `ZonefsSb::get_zone_info` |
| `zonefs_get_zone_info_cb()` | per-zone callback | `ZonefsSb::get_zone_info_cb` |
| `zonefs_check_zone_condition()` | per-zone-state classify | `ZonefsSb::check_zone_condition` |
| `zonefs_account_active()` | per-zone active-count | `ZonefsSb::account_active` |
| `zonefs_inode_account_active()` | per-inode active-count | `ZonefsSb::inode_account_active` |
| `zonefs_zone_mgmt()` | per-OPEN/CLOSE/RESET/FINISH | `ZonefsSb::zone_mgmt` |
| `zonefs_inode_zone_mgmt()` | per-inode zone op | `ZonefsSb::inode_zone_mgmt` |
| `zonefs_i_size_write()` | per-size-update + auto-close | `ZonefsSb::i_size_write` |
| `zonefs_update_stats()` | per-s_used_blocks delta | `ZonefsSb::update_stats` |
| `__zonefs_io_error()` | per-IO-error zone refresh | `ZonefsSb::io_error` |
| `zonefs_handle_io_error()` | per-error degrade-mode | `ZonefsSb::handle_io_error` |
| `zonefs_inode_update_mode()` | per-zone-cond mode adjust | `ZonefsSb::inode_update_mode` |
| `zonefs_get_file_inode()` | per-zone-file iget | `ZonefsSb::get_file_inode` |
| `zonefs_get_zgroup_inode()` | per-group-dir iget | `ZonefsSb::get_zgroup_inode` |
| `zonefs_get_dir_inode()` | per-cnv/seq dispatch | `ZonefsSb::get_dir_inode` |
| `zonefs_lookup()` | per-dentry resolve | `ZonefsSb::lookup` |
| `zonefs_readdir_root()` / `_zgroup()` | per-readdir | `ZonefsSb::readdir_*` |
| `zonefs_fname_to_fno()` | per-base-10 name → zone-no | `ZonefsSb::fname_to_fno` |
| `zonefs_statfs()` | per-statfs(2) | `ZonefsSb::statfs` |
| `zonefs_show_options()` | per-/proc/mounts | `ZonefsSb::show_options` |
| `zonefs_parse_param()` | per-mount-option | `ZonefsSb::parse_param` |
| `zonefs_reconfigure()` | per-remount | `ZonefsSb::reconfigure` |
| `zonefs_init_fs_context()` | per-fs_context init | `ZonefsSb::init_fs_context` |
| `zonefs_get_tree()` | per-get_tree_bdev | `ZonefsSb::get_tree` |
| `zonefs_kill_super()` | per-umount | `ZonefsSb::kill_super` |
| `zonefs_inode_setattr()` | per-chmod/chown/truncate | `ZonefsSb::inode_setattr` |
| `zonefs_alloc_inode()` / `_free_inode()` | per-inode slab | `ZonefsSb::alloc_inode` / `free_inode` |
| `ZONEFS_MAGIC` ('Z' 'F' 'S' 0x53) / `ZONEFS_SUPER_SIZE` (4096) | per-spec | shared |
| `ZONEFS_F_AGGRCNV` / `_UID` / `_GID` / `_PERM` | per-feature | shared |
| `ZONEFS_MNTOPT_ERRORS_RO` / `_ZRO` / `_ZOL` / `_REPAIR` / `_EXPLICIT_OPEN` | per-mount-opt | shared |
| `ZONEFS_ZONE_INIT_MODE/OPEN/ACTIVE/OFFLINE/READONLY/CNV` | per-z_flags | shared |
| `ZONEFS_ZTYPE_CNV` / `_SEQ` / `_MAX` | per-ztype | shared |
| `BLK_ZONE_TYPE_CONVENTIONAL` / `SEQWRITE_REQ` / `SEQWRITE_PREF` | per-block-layer-ztype | shared |
| `BLK_ZONE_COND_NOT_WP` / `EMPTY` / `IMP_OPEN` / `EXP_OPEN` / `CLOSED` / `FULL` / `READONLY` / `OFFLINE` | per-block-layer-cond | shared |

## Compatibility contract

REQ-1: struct zonefs_super (on disk, 4096 B at LBA 0 of zone 0):
- s_magic (u32 LE) = ZONEFS_MAGIC ('Z' | 'F' << 8 | 'S' << 16 | 0x53 << 24).
- s_version (u32 LE).
- s_features (u64 LE) — ZONEFS_F_AGGRCNV | _UID | _GID | _PERM.
- s_uid (u32 LE, present iff ZONEFS_F_UID).
- s_gid (u32 LE, present iff ZONEFS_F_GID).
- s_perm (u32 LE, present iff ZONEFS_F_PERM).
- s_crc (u32 LE) — CRC32 over the 4096-byte super with s_crc field zeroed.
- s_label[ZONEFS_LABEL_LEN].
- s_uuid[UUID_LEN].
- s_reserved[...] — must be entirely zero.

REQ-2: struct zonefs_sb_info fields used:
- s_mount_opts (unsigned long — ZONEFS_MNTOPT_*).
- s_features (u64).
- s_uid / s_gid / s_perm (defaults: 0/0/0640).
- s_uuid.
- s_zone_sectors_shift (ilog2(bdev_zone_sectors(bdev))).
- s_blocks / s_used_blocks (statfs accounting; spinlock-protected).
- s_lock (spinlock — used_blocks accounting).
- s_zgroup[ZONEFS_ZTYPE_MAX] (per-type group).
- s_wro_seq_files / s_max_wro_seq_files (atomic — open-zone refcount + cap).
- s_active_seq_files / s_max_active_seq_files (atomic — active-zone refcount + cap).
- sysfs registration handles.

REQ-3: struct zonefs_zone_group fields:
- g_zones: *Vec<ZonefsZone> (kvzalloc_objs).
- g_nr_zones: u32.
- g_inode: *Inode (zone-group dir inode, pinned).

REQ-4: struct zonefs_zone fields:
- z_flags (u32): ZONEFS_ZONE_INIT_MODE | OPEN | ACTIVE | OFFLINE | READONLY | CNV.
- z_sector (sector_t — zone start LBA).
- z_size (u64 — bytes).
- z_capacity (u64 — usable bytes, ≤ z_size; differs from z_size for some ZNS drives).
- z_wpoffset (u64 — bytes written so far / file size).
- z_mode, z_uid, z_gid (per-zone access caches).

REQ-5: zonefs_fill_super(sb, fc):
- if !bdev_is_zoned(sb.s_bdev): -EINVAL.
- sbi = kzalloc; sb.s_fs_info = sbi.
- spin_lock_init(sbi.s_lock).
- sb.s_magic = ZONEFS_MAGIC; s_maxbytes = 0; s_op = &zonefs_sops; s_time_gran = 1.
- sb_set_blocksize(sb, bdev_zone_write_granularity(sb.s_bdev)) — required for SEQ-write alignment.
- sbi.s_zone_sectors_shift = ilog2(bdev_zone_sectors(sb.s_bdev)).
- sbi.s_uid = GLOBAL_ROOT_UID; s_gid = GLOBAL_ROOT_GID; s_perm = 0640.
- sbi.s_mount_opts = ctx.s_mount_opts.
- atomic_set(s_wro_seq_files, 0); s_max_wro_seq_files = bdev_max_open_zones(bdev).
- atomic_set(s_active_seq_files, 0); s_max_active_seq_files = bdev_max_active_zones(bdev).
- ret = zonefs_read_super(sb).
- zonefs_info "Mounting %u zones".
- If max_open == 0 ∧ max_active == 0 ∧ EXPLICIT_OPEN: warn + clear EXPLICIT_OPEN.
- ret = zonefs_init_zgroups(sb) → blkdev_report_zones + init each group.
- inode = new_inode(sb); i_ino = bdev_nr_zones(bdev); i_mode = S_IFDIR | 0o555.
- i_op = &zonefs_dir_inode_operations; i_fop = &zonefs_dir_operations.
- inode.i_size = 2; nlink = 2.
- for ztype in 0..ZONEFS_ZTYPE_MAX: if group has zones: inc_nlink + ++i_size.
- sb.s_root = d_make_root(inode).
- ret = zonefs_get_zgroup_inodes(sb) (pin the cnv/seq dir inodes in icache).
- ret = zonefs_sysfs_register(sb).
- On error: zonefs_release_zgroup_inodes + zonefs_free_zgroups.

REQ-6: zonefs_read_super(sb):
- super = kmalloc(ZONEFS_SUPER_SIZE).
- ret = bdev_rw_virt(bdev, sector 0, super, ZONEFS_SUPER_SIZE, REQ_OP_READ).
- if le32(super.s_magic) != ZONEFS_MAGIC: -EINVAL.
- stored_crc = le32(super.s_crc).
- super.s_crc = 0.
- crc = crc32(~0U, super, sizeof(zonefs_super)).
- if crc != stored_crc: -EINVAL with log.
- sbi.s_features = le64(super.s_features).
- if s_features & ~ZONEFS_F_DEFINED_FEATURES (AGGRCNV|UID|GID|PERM): -EINVAL.
- if ZONEFS_F_UID: sbi.s_uid = make_kuid(super.s_uid); reject !uid_valid.
- if ZONEFS_F_GID: sbi.s_gid = make_kgid(super.s_gid); reject !gid_valid.
- if ZONEFS_F_PERM: sbi.s_perm = le32(super.s_perm).
- if memchr_inv(super.s_reserved, 0, sizeof(s_reserved)): -EINVAL.
- import_uuid(&sbi.s_uuid, super.s_uuid).
- kfree(super).

REQ-7: zonefs_get_zone_info(zd):
- zd.zones = kvzalloc_objs(blk_zone, bdev_nr_zones(bdev)).
- ret = blkdev_report_zones(bdev, 0, BLK_ALL_ZONES, zonefs_get_zone_info_cb, zd).
- if ret < 0 ∨ ret != bdev_nr_zones(bdev): -EIO.

REQ-8: zonefs_get_zone_info_cb(zone, idx, data):
- if !idx: return 0 (zone 0 holds super block; not a file).
- switch zone.type:
  - BLK_ZONE_TYPE_CONVENTIONAL:
    - if AGGRCNV: aggregate contiguous CNV → 1 file per run.
      - if !(g_nr_zones) ∨ zone.start != zd.cnv_zone_start: g_nr_zones++.
      - zd.cnv_zone_start = zone.start + zone.len.
    - else: g_nr_zones++.
  - BLK_ZONE_TYPE_SEQWRITE_REQ / SEQWRITE_PREF: ZONEFS_ZTYPE_SEQ.g_nr_zones++.
  - default: -EIO.
- memcpy(&zd.zones[idx], zone).

REQ-9: zonefs_init_zgroup(sb, zd, ztype):
- if !g_nr_zones: return 0.
- zgroup.g_zones = kvzalloc_objs(ZonefsZone, g_nr_zones).
- end = zd.zones + bdev_nr_zones(bdev).
- for zone in &zd.zones[1..end], next = zone+1:
  - if zonefs_zone_type(zone) != ztype: continue.
  - if AGGRCNV ∧ ztype == CNV:
    - while next.type == ztype:
      - zone.len += next.len.
      - zone.capacity += next.capacity.
      - if next.cond == READONLY ∧ zone.cond != OFFLINE: zone.cond = READONLY.
      - else if next.cond == OFFLINE: zone.cond = OFFLINE.
      - next++.
  - z = &zgroup.g_zones[n].
  - if ztype == CNV: z.z_flags |= ZONEFS_ZONE_CNV.
  - z.z_sector = zone.start.
  - z.z_size = zone.len << SECTOR_SHIFT.
  - if z.z_size > bdev_zone_sectors << SECTOR_SHIFT ∧ !AGGRCNV: -EINVAL.
  - z.z_capacity = min(MAX_LFS_FILESIZE, zone.capacity << SECTOR_SHIFT).
  - z.z_wpoffset = zonefs_check_zone_condition(sb, z, zone).
  - z.z_mode = S_IFREG | sbi.s_perm; z.z_uid = sbi.s_uid; z.z_gid = sbi.s_gid.
  - z.z_flags |= ZONEFS_ZONE_INIT_MODE.
  - sb.s_maxbytes = max(z.z_capacity, sb.s_maxbytes).
  - sbi.s_blocks += z.z_capacity >> blocksize_bits.
  - sbi.s_used_blocks += z.z_wpoffset >> blocksize_bits.
  - if ztype == SEQ ∧ zone.cond ∈ {IMP_OPEN, EXP_OPEN}:
    - ret = zonefs_zone_mgmt(sb, z, REQ_OP_ZONE_CLOSE) (force-close on mount to start at 0).
  - zonefs_account_active(sb, z).
  - n++.

REQ-10: zonefs_check_zone_condition(sb, z, zone):
- switch zone.cond:
  - OFFLINE: z.z_flags |= OFFLINE; return 0.
  - READONLY: z.z_flags |= READONLY; return z.z_capacity if CNV else z.z_wpoffset.
  - FULL: return z.z_capacity.
  - default: return z.z_capacity if CNV else (zone.wp - zone.start) << SECTOR_SHIFT.

REQ-11: zonefs_zone_mgmt(sb, z, op):
- if op == REQ_OP_ZONE_CLOSE ∧ !z.z_wpoffset: op = REQ_OP_ZONE_RESET (avoid "closed" state for unwritten zones — keeps active-zone count down on ZNS).
- trace_zonefs_zone_mgmt.
- ret = blkdev_zone_mgmt(bdev, op, z.z_sector, z.z_size >> SECTOR_SHIFT).

REQ-12: zonefs_account_active(sb, z):
- if z is CNV: return (CNV zones are not subject to active accounting).
- if z.z_flags & (OFFLINE | READONLY): go to clearing path.
- if (z.z_flags & OPEN) ∨ (0 < z.z_wpoffset < z.z_capacity):
  - if !(z.z_flags & ACTIVE): z.z_flags |= ACTIVE; atomic_inc(s_active_seq_files).
- else:
  - if z.z_flags & ACTIVE: z.z_flags &= ~ACTIVE; atomic_dec(s_active_seq_files).

REQ-13: zonefs_i_size_write(inode, isize):
- i_size_write(inode, isize).
- if isize >= z.z_capacity:
  - if z.z_flags & ACTIVE: atomic_dec(s_active_seq_files).
  - z.z_flags &= ~(OPEN | ACTIVE) (full zone needs no explicit close).

REQ-14: __zonefs_io_error(inode, write):
- if !is_seq(z):
  - synthesize fake blk_zone (cond=NOT_WP, wp=start+len): pass to handle_io_error.
- else:
  - memalloc_noio_save → blkdev_report_zones(bdev, z.z_sector, 1, cb, &zone) → restore.
  - if ret != 1: SB_RDONLY + zonefs_warn; return.
- zonefs_handle_io_error(inode, &zone, write).

REQ-15: zonefs_handle_io_error(inode, zone, write):
- data_size = check_zone_condition(sb, z, zone).
- isize = i_size_read(inode).
- if zone OK ∧ !write ∧ isize == data_size: return (transient read error).
- if isize != data_size: warn.
- if OFFLINE ∨ ZONEFS_MNTOPT_ERRORS_ZOL: z.z_flags |= OFFLINE; update_mode; data_size = 0.
- else if READONLY ∨ ZONEFS_MNTOPT_ERRORS_ZRO: z.z_flags |= READONLY; update_mode; data_size = isize.
- else if ZONEFS_MNTOPT_ERRORS_RO ∧ data_size > isize: data_size = isize (hide garbage).
- if EXPLICIT_OPEN ∧ (z is READONLY|OFFLINE): z.z_flags &= ~OPEN.
- if ZONEFS_MNTOPT_ERRORS_RO ∧ !sb_rdonly: sb.s_flags |= SB_RDONLY.
- zonefs_update_stats + zonefs_i_size_write + z.z_wpoffset = data_size.
- zonefs_inode_account_active.

REQ-16: zonefs_inode_update_mode(inode):
- if OFFLINE: inode.i_flags |= S_IMMUTABLE; inode.i_mode &= ~0o777.
- else if READONLY: inode.i_flags |= S_IMMUTABLE; if INIT_MODE { i_mode &= ~0o777 } else { i_mode &= ~0o222 }.
- z.z_flags &= ~INIT_MODE.
- z.z_mode = inode.i_mode.

REQ-17: zonefs_get_file_inode(dir, dentry):
- zgroup = dir.i_private.
- fno = zonefs_fname_to_fno(dentry.d_name) (base-10, no leading zeros).
- if fno < 0 ∨ fno >= zgroup.g_nr_zones: -ENOENT.
- z = &zgroup.g_zones[fno]; ino = z.z_sector >> sbi.s_zone_sectors_shift.
- inode = iget_locked(sb, ino).
- if !I_NEW: WARN_ON i_private != z; return inode.
- inode.i_mode/uid/gid/size/blocks from z.z_*.
- i_op = &zonefs_file_inode_operations; i_fop = &zonefs_file_operations.
- i_mapping->a_ops = &zonefs_file_aops; mapping_set_large_folios.
- zonefs_inode_update_mode.
- unlock_new_inode.

REQ-18: zonefs_lookup(dir, dentry, flags):
- if dentry.d_name.len > ZONEFS_NAME_MAX (16): -ENAMETOOLONG.
- if dir == sb.s_root: zonefs_get_dir_inode (cnv / seq).
- else: zonefs_get_file_inode.
- return d_splice_alias(inode, dentry).

REQ-19: zonefs_readdir_root(file, ctx):
- if ctx.pos >= inode.i_size: 0.
- dir_emit_dots.
- ctx.pos == 2: emit "cnv" (if CNV present) else "seq".
- ctx.pos == 3 ∧ have-seq: emit "seq".

REQ-20: zonefs_readdir_zgroup(file, ctx):
- if ctx.pos >= inode.i_size + 2: 0.
- dir_emit_dots.
- for f in [ctx.pos - 2 .. g_nr_zones): emit snprintf("%u", f), DT_REG.

REQ-21: Mount-options parameters:
- errors=remount-ro | zone-ro | zone-offline | repair (enum).
- explicit-open (flag).
- Default: ZONEFS_MNTOPT_ERRORS_RO.

REQ-22: zonefs_reconfigure(fc):
- sync_filesystem.
- sbi.s_mount_opts = ctx.s_mount_opts (full overwrite — no per-option whitelist).

REQ-23: zonefs_statfs(dentry, buf):
- f_type = ZONEFS_MAGIC; f_bsize = sb.s_blocksize; f_namelen = ZONEFS_NAME_MAX.
- f_blocks = sbi.s_blocks.
- f_bfree = f_blocks - sbi.s_used_blocks.
- f_bavail = f_bfree.
- f_files = sum over ztypes of (g_nr_zones + 1) (zone files + 1 dir per group).
- f_ffree = 0 (no namespace mutation possible).
- f_fsid = uuid_to_fsid(sbi.s_uuid).

REQ-24: zonefs_inode_setattr(idmap, dentry, iattr):
- if IS_IMMUTABLE(inode): -EPERM.
- setattr_prepare(&nop_mnt_idmap).
- if ATTR_MODE ∧ S_ISDIR ∧ (mode & 0o222): -EPERM (no write on group dirs).
- if (ATTR_UID ∧ uid != current) ∨ (ATTR_GID ∧ gid != current): dquot_transfer.
- if ATTR_SIZE: zonefs_file_truncate (calls REQ_OP_ZONE_RESET on seq).
- setattr_copy.
- If S_ISREG: persist mode/uid/gid into z.z_*.

REQ-25: zonefs_kill_super(sb):
- zonefs_release_zgroup_inodes (iput pinned group dir-inodes).
- kill_block_super.
- zonefs_sysfs_unregister + zonefs_free_zgroups.
- kfree(sbi).

REQ-26: super_operations (zonefs_sops):
- alloc_inode = zonefs_alloc_inode (alloc_inode_sb + inode_init_once + mutex_init i_truncate_mutex + zi.i_wr_refcnt = 0).
- free_inode = zonefs_free_inode.
- statfs = zonefs_statfs.
- show_options = zonefs_show_options.

## Acceptance Criteria

- [ ] AC-1: mount("zonefs", dev) on a non-zoned bdev: -EINVAL.
- [ ] AC-2: super-block magic mismatch: mount fails -EINVAL.
- [ ] AC-3: super-block CRC32 mismatch (s_crc field zeroed-during-compute): -EINVAL.
- [ ] AC-4: super-block s_features has an undefined bit: -EINVAL.
- [ ] AC-5: super-block s_reserved non-zero: -EINVAL.
- [ ] AC-6: device with only sequential zones: cnv/ directory absent; only seq/ enumerated.
- [ ] AC-7: device with mixed zones + ZONEFS_F_AGGRCNV: cnv/0 covers all contiguous CNV zones.
- [ ] AC-8: zone-0 is not exposed as a file (holds super block).
- [ ] AC-9: lookup "seq/3" returns inode whose i_size == zone[3].z_wpoffset.
- [ ] AC-10: write to a full sequential zone returns -ENOSPC (file size == z_capacity).
- [ ] AC-11: read of an OFFLINE zone returns -EIO; inode has S_IMMUTABLE.
- [ ] AC-12: READONLY zone: opening for write returns -EPERM (mode 0444 effective).
- [ ] AC-13: errors=zone-offline + IO error: zone transitions to OFFLINE; file becomes immutable + i_size=0.
- [ ] AC-14: errors=zone-ro + IO error: zone transitions to READONLY; i_size frozen.
- [ ] AC-15: errors=remount-ro + IO error: sb.s_flags |= SB_RDONLY.
- [ ] AC-16: explicit-open + zone close: blkdev_zone_mgmt(REQ_OP_ZONE_CLOSE) issued; ACTIVE/OPEN flags cleared.
- [ ] AC-17: zonefs_zone_mgmt(REQ_OP_ZONE_CLOSE) on z_wpoffset==0: rewritten to REQ_OP_ZONE_RESET.
- [ ] AC-18: full zone (isize >= z_capacity): OPEN/ACTIVE flags cleared automatically.
- [ ] AC-19: statfs reports f_blocks across all zones in blocks; f_files == sum (g_nr_zones + 1).
- [ ] AC-20: setattr on cnv/ or seq/ dir with mode & 0o222: -EPERM.
- [ ] AC-21: create/unlink/mkdir in any zonefs directory: -EPERM (no namespace mutation).
- [ ] AC-22: max_open_zones == 0 ∧ max_active_zones == 0 + explicit-open mount-option: ignored with warn.

## Architecture

```
struct ZonefsOnDiskSuper {           // 4096 bytes, LBA 0 of zone 0
  s_magic: u32,                      // LE — ZONEFS_MAGIC ('Z'|'F'<<8|'S'<<16|0x53<<24)
  s_version: u32,                    // LE
  s_features: u64,                   // LE — F_AGGRCNV | F_UID | F_GID | F_PERM
  s_uid: u32, s_gid: u32, s_perm: u32, // LE — present iff respective F_* feature
  s_crc: u32,                        // LE — CRC32 over the whole 4096-byte struct (s_crc==0)
  s_label: [u8; ZONEFS_LABEL_LEN],
  s_uuid: [u8; UUID_LEN],
  s_reserved: [u8; ...],             // must be zero
}

struct ZonefsZone {
  z_flags: u32,                      // INIT_MODE | OPEN | ACTIVE | OFFLINE | READONLY | CNV
  z_sector: u64,                     // device LBA of zone start
  z_size: u64,                       // bytes
  z_capacity: u64,                   // usable bytes
  z_wpoffset: u64,                   // bytes written
  z_mode: u16, z_uid: u32, z_gid: u32,
}

struct ZonefsZoneGroup {
  g_zones: *Vec<ZonefsZone>,
  g_nr_zones: u32,
  g_inode: *Inode,                   // pinned cnv/ or seq/ dir inode
}

struct ZonefsSbInfo {
  s_mount_opts: u64,                 // ZONEFS_MNTOPT_*
  s_features: u64,
  s_uid: Kuid, s_gid: Kgid, s_perm: u16,
  s_uuid: Uuid,
  s_zone_sectors_shift: u32,
  s_blocks: u64, s_used_blocks: u64, // statfs accounting
  s_lock: SpinLock,                  // protects s_used_blocks
  s_zgroup: [ZonefsZoneGroup; 2],    // CNV, SEQ
  s_wro_seq_files: AtomicU32, s_max_wro_seq_files: u32,
  s_active_seq_files: AtomicU32, s_max_active_seq_files: u32,
  /* sysfs handles */
}

struct ZonefsInodeInfo {
  i_truncate_mutex: Mutex<()>,       // serializes truncate vs zone-mgmt
  i_wr_refcnt: u32,                  // explicit-open refcount
  i_vnode: Inode,
}

struct ZonefsContext {
  s_mount_opts: u64,
}

struct ZonefsZoneData {               // mount-time scratch
  sb: *SuperBlock,
  nr_zones: [u32; ZONEFS_ZTYPE_MAX],
  cnv_zone_start: u64,               // LBA of next-contig CNV (for AGGRCNV)
  zones: *Vec<BlkZone>,              // raw report
}
```

`ZonefsSb::fill_super(sb, fc) -> Result<()>`:
1. if !bdev_is_zoned(sb.s_bdev): -EINVAL.
2. sbi = kzalloc; sb.s_fs_info = sbi.
3. sb.s_magic = ZONEFS_MAGIC; s_op = &ZONEFS_SOPS; s_time_gran = 1; s_maxbytes = 0.
4. sb_set_blocksize(sb, bdev_zone_write_granularity(bdev)).
5. sbi.s_zone_sectors_shift = ilog2(bdev_zone_sectors(bdev)).
6. sbi.s_uid = root_uid; s_gid = root_gid; s_perm = 0o640.
7. sbi.s_mount_opts = ctx.s_mount_opts.
8. atomic_set(s_wro_seq_files, 0); s_max_wro_seq_files = bdev_max_open_zones(bdev).
9. atomic_set(s_active_seq_files, 0); s_max_active_seq_files = bdev_max_active_zones(bdev).
10. ZonefsSb::read_super(sb)?
11. info "Mounting %u zones".
12. if max_open == 0 ∧ max_active == 0 ∧ EXPLICIT_OPEN: warn; clear EXPLICIT_OPEN.
13. ZonefsSb::init_zgroups(sb)?
14. inode = new_inode(sb); i_ino = bdev_nr_zones(bdev); i_mode = S_IFDIR | 0o555.
15. i_op = &ZONEFS_DIR_INODE_OPS; i_fop = &ZONEFS_DIR_OPS; i_size = 2; nlink = 2.
16. for ztype: if g_nr_zones > 0 { inc_nlink; ++i_size }.
17. sb.s_root = d_make_root(inode).
18. ZonefsSb::get_zgroup_inodes(sb)?  /* pin cnv/seq dir inodes */
19. ZonefsSb::sysfs_register(sb)?
20. cleanup on error: release_zgroup_inodes + free_zgroups.

`ZonefsSb::read_super(sb) -> Result<()>`:
1. super = kmalloc(ZONEFS_SUPER_SIZE).
2. bdev_rw_virt(bdev, 0, super, ZONEFS_SUPER_SIZE, REQ_OP_READ)?
3. if le32(super.s_magic) != ZONEFS_MAGIC: -EINVAL.
4. stored_crc = le32(super.s_crc).
5. super.s_crc = 0.
6. crc = crc32(~0, super, sizeof(zonefs_super)).
7. if crc != stored_crc: -EINVAL.
8. sbi.s_features = le64(super.s_features).
9. if s_features & ~ZONEFS_F_DEFINED_FEATURES: -EINVAL.
10. if ZONEFS_F_UID: sbi.s_uid = make_kuid(super.s_uid); reject !valid.
11. if ZONEFS_F_GID: sbi.s_gid = make_kgid(super.s_gid); reject !valid.
12. if ZONEFS_F_PERM: sbi.s_perm = le32(super.s_perm).
13. if any non-zero byte in s_reserved: -EINVAL.
14. import_uuid(&sbi.s_uuid, super.s_uuid).

`ZonefsSb::init_zgroups(sb) -> Result<()>`:
1. zd = ZonefsZoneData::new(sb).
2. ZonefsSb::get_zone_info(&zd)?  /* blkdev_report_zones + cb */
3. for ztype in 0..ZONEFS_ZTYPE_MAX:
   - ZonefsSb::init_zgroup(sb, &zd, ztype)?
4. free zd.zones (always).
5. on error: free_zgroups(sb).

`ZonefsSb::zone_mgmt(sb, z, op) -> Result<()>`:
1. if op == REQ_OP_ZONE_CLOSE ∧ z.z_wpoffset == 0: op = REQ_OP_ZONE_RESET.
2. trace_zonefs_zone_mgmt(sb, z, op).
3. blkdev_zone_mgmt(sb.s_bdev, op, z.z_sector, z.z_size >> SECTOR_SHIFT).

`ZonefsSb::account_active(sb, z)`:
1. if zonefs_zone_is_cnv(z): return.
2. if z.z_flags & (OFFLINE | READONLY): goto clear.
3. if (z.z_flags & OPEN) ∨ (0 < z.z_wpoffset < z.z_capacity):
   - if !(z.z_flags & ACTIVE): z.z_flags |= ACTIVE; atomic_inc(s_active_seq_files).
   - return.
4. clear: if z.z_flags & ACTIVE: z.z_flags &= ~ACTIVE; atomic_dec(s_active_seq_files).

`ZonefsSb::handle_io_error(inode, zone, write)`:
1. data_size = check_zone_condition(sb, z, zone).
2. isize = i_size_read(inode).
3. if zone OK ∧ !write ∧ isize == data_size: return.
4. if isize != data_size: warn.
5. branch by zone-state vs MNTOPT_ERRORS_*:
   - OFFLINE | ERRORS_ZOL: z.z_flags |= OFFLINE; update_mode; data_size = 0.
   - else READONLY | ERRORS_ZRO: z.z_flags |= READONLY; update_mode; data_size = isize.
   - else ERRORS_RO ∧ data_size > isize: data_size = isize.
6. if EXPLICIT_OPEN ∧ (READONLY|OFFLINE): z.z_flags &= ~OPEN.
7. if ERRORS_RO ∧ !sb_rdonly: sb.s_flags |= SB_RDONLY.
8. update_stats + i_size_write + z.z_wpoffset = data_size.
9. inode_account_active.

`ZonefsSb::lookup(dir, dentry, flags) -> Result<*Dentry>`:
1. if name.len > ZONEFS_NAME_MAX (16): -ENAMETOOLONG.
2. if dir is root: ZonefsSb::get_dir_inode (cnv/seq dispatch by 3-byte memcmp).
3. else: ZonefsSb::get_file_inode (base-10 fno → zone array).
4. d_splice_alias(inode, dentry).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bdev_is_zoned_required` | INVARIANT | per-fill_super: !bdev_is_zoned ⟹ -EINVAL. |
| `super_magic_required` | INVARIANT | per-read_super: s_magic == ZONEFS_MAGIC. |
| `super_crc_validated` | INVARIANT | per-read_super: crc32 (with s_crc==0) == stored_crc. |
| `super_features_subset` | INVARIANT | per-read_super: s_features ⊆ ZONEFS_F_DEFINED_FEATURES. |
| `super_reserved_zero` | INVARIANT | per-read_super: s_reserved is all-zero. |
| `zone0_not_a_file` | INVARIANT | per-get_zone_info_cb: idx == 0 returns 0 with no file created. |
| `seq_zone_close_at_zero_wp_resets` | INVARIANT | per-zone_mgmt: CLOSE with z_wpoffset==0 rewritten to RESET. |
| `cnv_zone_never_active` | INVARIANT | per-account_active: CNV zones do not increment s_active_seq_files. |
| `full_zone_clears_open_active` | INVARIANT | per-i_size_write: isize >= z_capacity ⟹ OPEN|ACTIVE cleared. |
| `offline_immutable` | INVARIANT | per-inode_update_mode: OFFLINE ⟹ S_IMMUTABLE ∧ mode & 0o777 == 0. |
| `readonly_clears_write_perms` | INVARIANT | per-inode_update_mode: READONLY ⟹ S_IMMUTABLE ∧ mode & 0o222 == 0. |
| `setattr_dir_rejects_writeable` | INVARIANT | per-setattr: S_ISDIR ∧ mode & 0o222 ⟹ -EPERM. |
| `report_zones_count_matches` | INVARIANT | per-get_zone_info: ret == bdev_nr_zones. |

### Layer 2: TLA+

`fs/zonefs/super.tla`:
- Per-zone-state machine: NOT_WP / EMPTY / IMP_OPEN / EXP_OPEN / CLOSED / FULL / READONLY / OFFLINE.
- Per-MNTOPT_ERRORS state: RO | ZRO | ZOL | REPAIR.
- Per-IO-error transition table: (zone_cond, write?, mount_opt) → (new_z_flags, new_i_size, sb_rdonly?).
- Properties:
  - `safety_seq_zone_writes_at_wp_only` — per-write: offset == z.z_wpoffset.
  - `safety_explicit_open_paired` — per-open/close: blkdev_zone_mgmt(OPEN) matched with mgmt(CLOSE) at refcount==0.
  - `safety_offline_no_io` — per-OFFLINE: read/write returns -EIO.
  - `safety_readonly_no_write` — per-READONLY: write returns -EPERM.
  - `safety_full_zone_no_open` — per-FULL: OPEN flag never set.
  - `safety_cnv_not_active_counted` — per-CNV: s_active_seq_files unaffected.
  - `safety_active_count_bounded` — per-mount: s_active_seq_files ≤ s_max_active_seq_files.
  - `safety_open_count_bounded` — per-mount: s_wro_seq_files ≤ s_max_wro_seq_files.
  - `liveness_io_error_eventually_consistent` — per-__zonefs_io_error: i_size converges to drive-reported wp.
  - `liveness_mount_terminates` — per-fill_super: succeeds or returns error in finite time.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ZonefsSb::read_super` post: features ⊆ defined; uid/gid valid if feature-set | `ZonefsSb::read_super` |
| `ZonefsSb::init_zgroups` post: ∀ z: z_size == zone.len << 9; z_capacity ≤ z_size | `ZonefsSb::init_zgroups` |
| `ZonefsSb::init_zgroup` post: CNV without AGGRCNV ⟹ z_size ≤ bdev_zone_sectors << 9 | `ZonefsSb::init_zgroup` |
| `ZonefsSb::zone_mgmt` post: CLOSE & wp==0 → RESET issued | `ZonefsSb::zone_mgmt` |
| `ZonefsSb::account_active` post: ACTIVE flag transitions paired with atomic_inc/dec | `ZonefsSb::account_active` |
| `ZonefsSb::i_size_write` post: isize ≥ z_capacity ⟹ !(z_flags & (OPEN|ACTIVE)) | `ZonefsSb::i_size_write` |
| `ZonefsSb::handle_io_error` post: zone-state and i_size consistent w.r.t. mount-opt | `ZonefsSb::handle_io_error` |
| `ZonefsSb::lookup` post: name.len ≤ 16 ∧ result inode.i_ino matches z.z_sector >> shift | `ZonefsSb::lookup` |
| `ZonefsSb::statfs` post: f_files == sum (g_nr_zones + 1 per group); f_ffree == 0 | `ZonefsSb::statfs` |
| `ZonefsSb::inode_setattr` post: S_ISDIR write-mode ⟹ -EPERM | `ZonefsSb::inode_setattr` |

### Layer 4: Verus/Creusot functional

`Per-mount: validate bdev_is_zoned → bdev_rw_virt sector 0 → ZONEFS_MAGIC + CRC32 → feature-bit check → import_uuid → blkdev_report_zones (BLK_ALL_ZONES) → classify each zone (CNV vs SEQ; AGGRCNV aggregation) → allocate ZonefsZone per file → set i_size = wp_offset (SEQ) or capacity (CNV) → REQ_OP_ZONE_CLOSE on any pre-existing IMP_OPEN/EXP_OPEN SEQ zones → build root → cnv/ + seq/ → pin group inodes → sysfs register` semantic equivalence: per-`Documentation/filesystems/zonefs.rst` and the Western Digital zonefs userland tools (mkzonefs).

## Hardening

(Inherits row-1 features from `fs/zonefs/00-overview.md` § Hardening.)

zonefs-superblock reinforcement:

- **Per-bdev_is_zoned mandatory** — defense against per-mount-on-regular-disk silent-data-loss.
- **Per-CRC32 on super (s_crc field zeroed-during-compute)** — defense against per-silent-bitrot on super.
- **Per-feature-bit subset check** — defense against per-forward-incompatible mount (unknown features rejected).
- **Per-s_reserved all-zero check** — defense against per-formatter using reserved field for vendor data.
- **Per-blocksize == bdev_zone_write_granularity** — defense against per-misaligned SEQ-write rejection by device.
- **Per-zone-0-not-a-file** — defense against per-application-overwriting-superblock.
- **Per-CLOSE-of-empty-zone → RESET rewrite** — defense against per-ZNS-resource-exhaustion (empty-CLOSED still counts as active).
- **Per-pre-existing-open SEQ zones force-CLOSED on mount** — defense against per-active-zone-count-drift-from-prior-crash.
- **Per-active-zone count bounded by bdev_max_active_zones** — defense against per-device-reject-explicit-open.
- **Per-OFFLINE → S_IMMUTABLE + mode 0o000** — defense against per-app-reading-stale-garbage from a dead zone.
- **Per-READONLY → S_IMMUTABLE + mode &= ~0o222** — defense against per-write-to-bad-zone.
- **Per-errors=zone-ro / zone-offline policy explicit** — defense against per-silent-error-papering.
- **Per-no-namespace-mutation (lookup-only)** — defense against per-create/unlink-on-zoned-device semantic-mismatch.
- **Per-setattr-on-dir mode-write rejected** — defense against per-userland-confusing-group-dirs-with-regular-dirs.
- **Per-iget_locked dedup by ino = z_sector >> shift** — defense against per-double-inode for the same zone.
- **Per-blkdev_report_zones GFP_NOIO context** — defense against per-reclaim-deadlock on truncate-mutex.
- **Per-truncate-mutex held for inode_zone_mgmt** — defense against per-concurrent-truncate-vs-zone-op race.
- **Per-write-pointer drift refresh on IO error** — defense against per-host-vs-device-wp divergence.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-checked copy on `blk_zone` reports crossing user/kernel and on `struct zonefs_zone_info` traversal during mount/parse.
- **PAX_KERNEXEC** — write-protects the read-only `zonefs_dir_inode_operations` and `zonefs_file_operations` tables after init; rejects late patching of dir/file op vectors.
- **PAX_RANDKSTACK** — per-syscall stack-offset randomization across `zonefs_iomap_begin`, `zonefs_file_write_iter`, and the report-zones callbacks.
- **PAX_REFCOUNT** — saturating `atomic_t` wraparound trap on `zi->i_wr_refcnt` (open-for-write counters per sequential zone) and on the shared `sb->s_active`.
- **PAX_MEMORY_SANITIZE** — auto-zeroes `struct zonefs_zone_group` and `struct zonefs_zone` allocations on free, scrubs the report-zones reply buffer after parse.
- **PAX_UDEREF** — explicit user-pointer dereference faulting for `ZONEFS_IOC_*` ioctl payloads (zone-info, zone-action).
- **PAX_RAP / kCFI** — forward-edge CFI on the `blk_zone_report_cb_t` callback dispatch (`zonefs_get_zone_info_cb`) and on iomap op vectors.
- **GRKERNSEC_HIDESYM** — strips zonefs symbol leakage from `/proc/kallsyms` and from `dmesg` zone-state traces.
- **GRKERNSEC_DMESG** — restricts the verbose `zonefs: zone %u state %s -> %s` transition logs to CAP_SYSLOG; defends against per-information-disclosure of ZBC zone layout.
- **ZBC zone-state validation** — every `BLK_ZONE_COND_*` transition cross-checked against the device report cache; rejects host-software-attempted state coercion (e.g. EMPTY⇄FULL forge) before write-pointer use.
- **Sequential-write enforcement** — `zonefs_write_checks` mandates `iocb->ki_pos == zi->i_wpoffset` for SEQ_WRITE_REQUIRED zones; `iov_iter_count` must align to `bdev_zone_write_granularity`; rejects out-of-order writes with `-EINVAL` before block-layer descent.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/zonefs/file.c — file_operations + iomap + direct-IO + dio_zone_append (separate Tier-3)
- fs/zonefs/sysfs.c — /sys/fs/zonefs attributes (separate Tier-3 if expanded)
- block/blk-zoned.c — blkdev_report_zones, blkdev_zone_mgmt (covered in block-layer Tier-3)
- block/blk-mq.c — REQ_OP_ZONE_APPEND request submission (covered in block-mq Tier-3)
- userland mkzonefs / zonefs-tools (out of kernel scope)
- Implementation code
