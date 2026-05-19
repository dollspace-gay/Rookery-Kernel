# Tier-3: fs/xfs/xfs_super.c — XFS superblock + mount

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/xfs/00-overview.md
upstream-paths:
  - fs/xfs/xfs_super.c (~2721 lines)
  - fs/xfs/libxfs/xfs_sb.c
  - fs/xfs/libxfs/xfs_format.h
  - fs/xfs/xfs_mount.h
-->

## Summary

XFS is a high-performance journaling filesystem from SGI (IRIX origin, ported to Linux). Per-fs has multiple **Allocation Groups (AGs)** for parallelism; each AG has its own free-block, inode, and refcount/realtime trees. Per-on-disk `struct xfs_sb` is the primary superblock at sector 0; replicated at start of each AG. Per-`xfs_mount_t` tracks per-fs in-memory state. Per-CoW-DATA support for shared-blocks (reflink). Per-`xfs_fill_super` reads + validates + opens AGs + logs. Per-journal v5 metadata (CRC + UUID + LSN). Per-features: V5 = CRC + reflink + finobt + sparseinode + rmapbt + bigtime. Critical for: HPC, large-file workloads (databases), VMware-on-Linux, RHEL/CentOS default.

This Tier-3 covers `xfs_super.c` (~2721 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct xfs_mount` | per-fs in-memory state | `XfsMount` |
| `struct xfs_sb` | on-disk superblock | `XfsSb` |
| `xfs_fs_fill_super()` | per-mount entry | `Xfs::fs_fill_super` |
| `xfs_finish_super()` | per-final mount step | `Xfs::finish_super` |
| `xfs_remount_rw()` / `_ro()` | per-remount | `Xfs::remount_*` |
| `xfs_fs_put_super()` | per-umount | `Xfs::fs_put_super` |
| `xfs_fs_statfs()` | per-statfs | `Xfs::fs_statfs` |
| `xfs_validate_sb_common()` | per-sb validate | `Xfs::validate_sb_common` |
| `xfs_validate_sb_read()` | per-on-mount validate | `Xfs::validate_sb_read` |
| `xfs_buf_read_uncached()` | per-block read | `Xfs::buf_read_uncached` |
| `xfs_mount_validate_sb()` | per-pre-recovery validate | `Xfs::mount_validate_sb` |
| `xfs_fs_freeze()` / `_unfreeze()` | per-freeze for snapshots | `Xfs::fs_freeze` |
| `xfs_quiesce_attr()` | per-quiesce | `Xfs::quiesce_attr` |
| `XFS_SB_MAGIC` (0x58465342 "XFSB") | per-magic | UAPI |
| `XFS_SB_VERSION_5` | per-V5 features | shared |

## Compatibility contract

REQ-1: On-disk xfs_sb:
- sb_magicnum: 0x58465342 "XFSB".
- sb_blocksize: filesystem block size (512..65536).
- sb_dblocks: total disk blocks.
- sb_rblocks: realtime device blocks.
- sb_rextents: realtime extents.
- sb_uuid: per-fs UUID (16 bytes).
- sb_logstart: journal start (filesystem-internal log).
- sb_rootino: root inode #.
- sb_rbmino: realtime bitmap inode.
- sb_rsumino: realtime summary inode.
- sb_rextsize: realtime extent size.
- sb_agblocks: blocks per AG.
- sb_agcount: # of AGs.
- sb_rbmblocks: realtime bitmap blocks.
- sb_logblocks: journal log size in blocks.
- sb_versionnum: feature version bits.
- sb_sectsize: sector size.
- sb_inodesize: inode size (256, 512, 1024, 2048).
- sb_inopblock: inodes per block.
- sb_fname: volume name (12 chars).
- sb_blocklog: log2(blocksize).
- sb_sectlog: log2(sectsize).
- sb_inodelog: log2(inodesize).
- sb_inopblog: log2(inopblock).
- sb_agblklog: log2(agblocks).
- sb_rextslog: log2(rextents).
- sb_inprogress: mkfs-in-progress bit.
- sb_imax_pct: inode max % of fs.
- sb_icount / sb_ifree: total + free inodes.
- sb_fdblocks: free data blocks.
- sb_frextents: free realtime extents.
- sb_uquotino / sb_gquotino / sb_pquotino: quota inodes.
- sb_qflags: quota flags.
- sb_features2 / sb_bad_features2: v5 feature bits.
- sb_features_compat / _ro_compat / _incompat / _log_incompat: v5 feature-bitmaps.
- sb_crc: CRC32C of sb.
- sb_spino_align: sparse-inode alignment.
- sb_pquotino: project quota inode.
- sb_lsn: per-LSN of last mod.
- sb_meta_uuid: per-meta UUID (separate from fs UUID for re-UUID).

REQ-2: Per-AG layout:
- AG-header: AG-free-space-info + AG-inode-info + AG-rmap-info + AG-refcount-info.
- Per-AG B+trees: free-by-block (bnobt), free-by-size (cntbt), inode-allocation (ibt), free-inodes (finobt), reverse-map (rmapbt), refcount (refcountbt).

REQ-3: Per-V5 feature bits:
- COMPAT: lazysbcount.
- COMPAT_RO: project-quota, finobt, rmapbt, refcountbt, bigtime, inobtcnt.
- INCOMPAT: directory v2-extents, attr1+attr2, log-v2.
- LOG_INCOMPAT: per-format-version on log.

REQ-4: xfs_fs_fill_super(sb, fc):
- Allocate xfs_mount.
- mp.m_super = sb.
- xfs_open_devices: open data + log + realtime.
- xfs_readsb: read primary sb.
- xfs_validate_sb_read: validate.
- xfs_mount_validate_sb: per-version checks.
- xfs_initialize_perag: per-AG init.
- xfs_log_mount: open journal.
- xfs_log_mount_finish: complete log mount + replay.
- xfs_qm_mount_quotas: per-uquota/gquota/pquota.
- xfs_mountfs: per-AG read.
- iget(rootino).
- d_make_root.

REQ-5: xfs_remount_rw / _ro:
- rw → ro: quiesce; flush log; xfs_log_mount_cancel.
- ro → rw: xfs_log_mount; replay if needed; xfs_quotacheck.

REQ-6: Per-CRC validation:
- xfs_sb_verify_crc: compute CRC32C; compare against sb.sb_crc.
- v4 (legacy): no CRC.
- v5: CRC required.

REQ-7: xfs_validate_sb_common:
- magic check.
- blocksize 512..65536 + power-of-2.
- sectorsize 512..8192 + power-of-2.
- inodesize 256, 512, 1024, 2048.
- agcount within max.

REQ-8: Per-features-incompat enforced:
- Unknown INCOMPAT: -EFSCORRUPTED.
- Unknown COMPAT_RO + rw mount: -EFSCORRUPTED.

REQ-9: xfs_fs_statfs:
- f_type = XFS_SUPER_MAGIC.
- f_blocks = total data blocks.
- f_bfree = sb.sb_fdblocks.
- f_files = sb.sb_icount.
- f_ffree = sb.sb_ifree.

REQ-10: xfs_fs_freeze:
- Drain all I/O; flush log; mark fs frozen.

## Acceptance Criteria

- [ ] AC-1: mount -t xfs /dev/sda1 /mnt: primary sb read; AGs opened.
- [ ] AC-2: V5 fs with CRC: per-block read validates CRC.
- [ ] AC-3: Unknown INCOMPAT feature: -EFSCORRUPTED.
- [ ] AC-4: Unknown COMPAT_RO + rw mount: fails; ro succeeds.
- [ ] AC-5: Per-AG init: per-AG perag structs allocated.
- [ ] AC-6: xfs_log_mount: journal-replay if dirty.
- [ ] AC-7: xfs_remount rw→ro: quiesce + flush.
- [ ] AC-8: xfs_fs_statfs: per-fs stats.
- [ ] AC-9: xfs_fs_freeze: pauses I/O; LSN-stable.
- [ ] AC-10: realtime device + main device: both opened.

## Architecture

Per-mount:

```
struct XfsMount {
  m_super: *SuperBlock,
  m_sb: XfsSb,
  m_log: *XfsLog,
  m_perag_lock: SpinLock,
  m_perag_tree: XArray<*XfsPerAg>,
  m_ddev_targp: *XfsBuftarg,                      // data device
  m_logdev_targp: *XfsBuftarg,
  m_rtdev_targp: *XfsBuftarg,
  m_quotainfo: *XfsQuotaInfo,
  m_dirops: *XfsDirOps,
  m_features: u64,                                // XFS_FEAT_*
  m_quotapages: u32,
  m_flags: u64,
  ...
}
```

Per-on-disk sb:

```
#[repr(C)]
struct XfsSb {
  sb_magicnum: u32,
  sb_blocksize: u32,
  sb_dblocks: u64,
  sb_rblocks: u64,
  sb_rextents: u64,
  sb_uuid: [u8; 16],
  sb_logstart: u64,
  sb_rootino: u64,
  sb_rbmino: u64,
  sb_rsumino: u64,
  sb_rextsize: u32,
  sb_agblocks: u32,
  sb_agcount: u32,
  sb_rbmblocks: u32,
  sb_logblocks: u32,
  sb_versionnum: u16,
  sb_sectsize: u16,
  sb_inodesize: u16,
  sb_inopblock: u16,
  sb_fname: [u8; 12],
  sb_blocklog: u8,
  sb_sectlog: u8,
  sb_inodelog: u8,
  sb_inopblog: u8,
  sb_agblklog: u8,
  sb_rextslog: u8,
  sb_inprogress: u8,
  sb_imax_pct: u8,
  sb_icount: u64,
  sb_ifree: u64,
  sb_fdblocks: u64,
  sb_frextents: u64,
  sb_uquotino: u64,
  sb_gquotino: u64,
  sb_qflags: u16,
  sb_flags: u8,
  sb_shared_vn: u8,
  sb_inoalignmt: u32,
  sb_unit: u32,
  sb_width: u32,
  sb_dirblklog: u8,
  sb_logsectlog: u8,
  sb_logsectsize: u16,
  sb_logsunit: u32,
  sb_features2: u32,
  sb_bad_features2: u32,
  sb_features_compat: u32,
  sb_features_ro_compat: u32,
  sb_features_incompat: u32,
  sb_features_log_incompat: u32,
  sb_crc: u32,
  sb_spino_align: u32,
  sb_pquotino: u64,
  sb_lsn: u64,
  sb_meta_uuid: [u8; 16],
}
```

`Xfs::fs_fill_super(sb, fc) -> Result<()>`:
1. mp = xfs_mount_alloc(sb).
2. mp.m_super = sb; sb.s_fs_info = mp.
3. err = xfs_open_devices(mp, fc).
4. err = xfs_readsb(mp, 0).
5. err = Xfs::validate_sb_read(mp).
6. err = xfs_mount_validate_sb(mp).
7. err = xfs_initialize_perag(mp, mp.m_sb.sb_agcount).
8. err = xfs_log_mount(mp).
9. err = xfs_mountfs(mp).
10. /* Root inode */
11. err = xfs_iget(mp, NULL, mp.m_sb.sb_rootino, 0, 0, &rip).
12. sb.s_root = d_make_root(VFS_I(rip)).

`Xfs::validate_sb_read(mp) -> Result<()>`:
1. sb = &mp.m_sb.
2. if sb.sb_magicnum != XFS_SB_MAGIC: return -EWRONGFS.
3. if sb.sb_versionnum > XFS_SB_VERSION_NUMBITS: return -EFSCORRUPTED.
4. if XFS_SB_VERSION_NUM(sb) == XFS_SB_VERSION_5:
   - /* CRC required */
   - if !sb.sb_crc: return -EFSBADCRC.
   - computed = crc32c(sb, sizeof(XfsSb) - sizeof(sb.sb_crc)).
   - if computed != sb.sb_crc: return -EFSBADCRC.

`Xfs::mount_validate_sb(mp) -> Result<()>`:
1. /* V5-feature gates */
2. if features_incompat & ~XFS_SB_FEAT_INCOMPAT_SUPPORTED: return -EOPNOTSUPP.
3. if !ro ∧ features_ro_compat & ~XFS_SB_FEAT_RO_COMPAT_SUPPORTED: return -EOPNOTSUPP.
4. if blocksize > XFS_MAX_BLOCKSIZE: return -EINVAL.

`Xfs::fs_statfs(dentry, buf) -> Result<()>`:
1. mp = XFS_M(dentry.d_sb).
2. buf.f_type = XFS_SUPER_MAGIC.
3. buf.f_bsize = mp.m_sb.sb_blocksize.
4. buf.f_blocks = mp.m_sb.sb_dblocks.
5. buf.f_bfree = mp.m_sb.sb_fdblocks.
6. buf.f_files = mp.m_sb.sb_icount.
7. buf.f_ffree = mp.m_sb.sb_ifree.

`Xfs::fs_freeze(sb) -> Result<()>`:
1. mp = XFS_M(sb).
2. xfs_quiesce_attr(mp).
3. xfs_log_quiesce(mp).
4. Ok.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `magic_matches` | INVARIANT | post-readsb: sb_magicnum == 0x58465342. |
| `crc_validates_v5` | INVARIANT | V5 fs: sb_crc matches computed. |
| `blocksize_pow2` | INVARIANT | sb_blocksize power-of-2 in [512, 65536]. |
| `agcount_positive` | INVARIANT | sb_agcount > 0. |
| `inodesize_in_set` | INVARIANT | sb_inodesize ∈ {256, 512, 1024, 2048}. |

### Layer 2: TLA+

`fs/xfs/super.tla`:
- Per-mount sb-read + feature-check + AG-init + log-mount + root-iget.
- Properties:
  - `safety_no_mount_unsupported_incompat` — per-INCOMPAT not-supp ⟹ -EOPNOTSUPP.
  - `safety_crc_validates_per_v5_mount` — per-V5: sb_crc verified.
  - `liveness_per_aggregate_init` — per-mount-success: per-AG perag allocated.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Xfs::fs_fill_super` post: mp allocated; sb.s_root non-NULL | `Xfs::fs_fill_super` |
| `Xfs::validate_sb_read` post: returns Ok iff magic + crc + version-bits valid | `Xfs::validate_sb_read` |
| `Xfs::mount_validate_sb` post: features ⊆ supported | `Xfs::mount_validate_sb` |
| `Xfs::fs_statfs` post: per-stat fields populated | `Xfs::fs_statfs` |

### Layer 4: Verus/Creusot functional

`Per-mount on-disk sb → per-AG init → per-log mount → root inode resolved → vfs SB ready` semantic equivalence: per-XFS on-disk format spec.

## Hardening

(Inherits row-1 features from `fs/xfs/00-overview.md` § Hardening.)

xfs-super reinforcement:

- **Per-V5 CRC mandatory** — defense against per-corruption silent.
- **Per-INCOMPAT enforced** — defense against per-feature mismatch.
- **Per-COMPAT_RO rw-mount block** — defense against per-write-incompatible corruption.
- **Per-multi-device opens validated** — defense against per-data/log/rt mismatch.
- **Per-CAP_SYS_ADMIN for mount-options** — defense against unprivileged tweak.
- **Per-quiesce on freeze** — defense against per-snapshot inconsistent.
- **Per-AG isolation** — defense against per-AG-corrupt cascading.
- **Per-features-log-incompat enforced** — defense against per-log-recovery wrong-version.
- **Per-spino_align validated** — defense against per-sparse-inode mis-align.
- **Per-meta-uuid separate from fs-uuid** — defense against per-uuid-rewrite causing scrub-confusion.
- **Per-sb LSN tracked** — defense against per-revert older sb-copy used.

## Grsecurity/PaX-style Reinforcement

XFS superblock is the trust anchor for the entire filesystem; a single bad sb propagates through every allocator and journal replay. Rookery applies grsec/PaX floor:

- **PAX_USERCOPY** — sb-derived sizes used in `copy_to_user`/`copy_from_user` (xattr, ACL, mount-stat output) are bound-checked against the validated sb fields, not the on-disk raw values.
- **PAX_KERNEXEC** — `xfs_super_operations`, `xfs_fs_type`, and per-feature dispatch tables are `static const`; per-format-version dispatch `__ro_after_init`.
- **PAX_RANDKSTACK** — mount/unmount and journal-replay paths run under randomized kstack offset.
- **PAX_REFCOUNT** — `xfs_mount.m_active_trans`, per-inode + per-AG refcounts, and `xfs_buf.b_hold` are saturating refcount.
- **PAX_MEMORY_SANITIZE** — `xfs_mount` allocation and per-AG `xfs_perag` slabs zeroed on free; on-disk sb scratch buffers sanitized so stale superblock copies cannot bleed.
- **PAX_UDEREF** — mount-option parser uses `fs_parameter` boundary; `_IOC_*` ioctl paths use `copy_*_user`; raw `__user` deref forbidden.
- **PAX_RAP/kCFI** — `xfs_super_operations.{alloc_inode,destroy_inode,write_inode,evict_inode,put_super,sync_fs,freeze_fs,unfreeze_fs,statfs,remount_fs,show_options}` are CFI-typed and `__ro_after_init`.
- **GRKERNSEC_HIDESYM** — sb-corruption diagnostics (`xfs_warn`, `xfs_alert`) strip kernel pointers; per-buffer addresses scrubbed for non-CAP_SYSLOG dmesg consumers.
- **GRKERNSEC_DMESG** — `xfs_alert` rate-limited; sb-corruption flood from a hostile image cannot DoS dmesg.
- **XFS V5 superblock CRC32C verify** — every sb read (primary + per-AG secondary) validates CRC32C against `sb_crc` field; mismatch is a mount-fail (not warn) for V5; V4 mount denied entirely (read-only legacy support guarded by explicit mount option + CAP_SYS_ADMIN).
- **Log-recovery LSN integrity** — every recovered log record's LSN compared against `sb_lsn`; out-of-range or non-monotonic LSN aborts recovery; matches grsec "no silent corruption" floor and defeats crafted-image LSN-rewind attacks.
- **Per-feature-bit allow-list** — `xfs_sb_has_compat_feature`/`xfs_sb_has_ro_compat_feature`/`xfs_sb_has_incompat_feature` evaluated against a kernel-side `__ro_after_init` known-features mask; unknown INCOMPAT bits fail-closed.
- **Mount CAP_SYS_ADMIN floor** — all XFS-specific mount options (logbsize, sunit, swidth, allocsize, attr2, ikeep, ...) require CAP_SYS_ADMIN; matches grsec mount-option lockdown.

Rationale: XFS has been a sustained CVE source (CVE-2018-13095, CVE-2021-4155, and numerous fuzzer-found mount-time bugs); the common vector is a malformed on-disk superblock or log reaching kernel decode without CRC/LSN validation. Rookery's contract pins both gates at the sb-load boundary so corrupt images fail-closed before any allocator state is constructed.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- fs/xfs/{xfs_log, xfs_inode, xfs_buf, xfs_aops, xfs_iomap, xfs_rmap, ...}.c (covered separately if expanded)
- libxfs (covered separately)
- xfs_repair / xfsprogs userspace (out-of-tree)
- Implementation code
