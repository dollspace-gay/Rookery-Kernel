---
title: "Tier-3: drivers/scsi/sd.c — SCSI disk (sd) upper-level driver"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-11
updated: 2026-05-11
---


## Design Specification

### Summary

The **SCSI disk ULP** — every block-device that surfaces as `/dev/sd*` is owned by this driver. Routes `REQ_OP_READ` / `REQ_OP_WRITE` / `REQ_OP_DISCARD` / `REQ_OP_WRITE_ZEROES` / `REQ_OP_FLUSH` / `REQ_OP_ZONE_*` from the block layer into SCSI CDBs (`READ_6`/`READ_10`/`READ_16`/`READ_32`/`WRITE_6`/`WRITE_10`/`WRITE_16`/`WRITE_32`/`UNMAP`/`WRITE_SAME`/`WRITE_SAME_16`/`SYNCHRONIZE_CACHE`/`SYNCHRONIZE_CACHE_16`/`ZBC_IN`/`ZBC_OUT`/`WRITE_ATOMIC_16`). Owns: per-disk `struct scsi_disk` control block (capacity, sector_size, physical_block_size, WCE/RCD/FUA/DPOFUA bits, protection_type, provisioning_mode, zoned-flag, opal_dev, openers refcount), per-disk probe (sd_probe → blk_mq_alloc_disk_for_queue → sd_format_disk_name → sd_revalidate_disk → device_add_disk), per-disk revalidate (read-capacity + VPD + cache + write-protect + zones + atomic-write + protection + queue-limits commit), per-rq command setup (sd_init_command dispatch by req_op + sd_setup_read_write_cmnd CDB selection + DMA-prot setup), per-cmd completion bottom-half (sd_done → sense-decode → sd_completed_bytes + sd_zbc_complete + invalid-opcode fallback for UNMAP/WRITE_SAME), per-host EH callback (sd_eh_action — medium-access timeout offline policy), per-disk sysfs class (`/sys/class/scsi_disk/<host>:<channel>:<id>:<lun>/` — cache_type, FUA, manage_*_start_stop, protection_type, protection_mode, app_tag_own, thin_provisioning, provisioning_mode, zeroing_mode, max_write_same_blocks, max_medium_access_timeouts, max_retries, zoned_cap), persistent reservations via `pr_ops` (sd_pr_register/sd_pr_reserve/sd_pr_release/sd_pr_preempt/sd_pr_clear). Critical for: every Linux block-storage workload that lands on a SCSI/SAS/iSCSI/USB-MS/UAS/virtio-scsi disk.

This Tier-3 covers `drivers/scsi/sd.c` (~4524 lines). The ZBC sidecar `drivers/scsi/sd_zbc.c` is referenced because `sd_init_command` and `sd_revalidate_disk` invoke it inline.

### Acceptance Criteria

- [ ] AC-1: sd_probe attaches a gendisk to TYPE_DISK / TYPE_ZBC / TYPE_MOD / TYPE_RBC sdev; assigns index via sd_index_ida; calls sd_revalidate_disk before device_add_disk.
- [ ] AC-2: sd_format_disk_name(index = 0) = "sda"; (25) = "sdz"; (26) = "sdaa"; (701) = "sdzz"; (702) = "sdaaa"; on buflen-exceed returns -EINVAL.
- [ ] AC-3: sd_setup_read_write_cmnd picks READ_6/WRITE_6 only when nr_blocks ≤ 0xff ∧ lba ≤ 0x1fffff ∧ no protect ∧ no FUA ∧ no write-hint.
- [ ] AC-4: sd_setup_read_write_cmnd picks READ_16/WRITE_16 when use_16_for_rw ∨ nr_blocks > 0xffff.
- [ ] AC-5: sd_setup_read_write_cmnd picks 32-byte VARIABLE_LENGTH CDB when sdkp->protection_type == T10_PI_TYPE2_PROTECTION ∧ protect != 0.
- [ ] AC-6: sd_setup_unmap_cmnd issues UNMAP (0x42) with 24-byte param-list carrying BE64 lba + BE32 nr_blocks.
- [ ] AC-7: sd_setup_write_same16_cmnd issues WRITE_SAME_16 (0x93) with unmap-bit = 0x8 when caller asks for unmap.
- [ ] AC-8: sd_done with sense_key == MEDIUM_ERROR returns good_bytes = sd_completed_bytes() = INFORMATION-field-derived partial-bytes-good.
- [ ] AC-9: sd_done with ILLEGAL_REQUEST asc ∈ {0x20, 0x24} on UNMAP disables provisioning (provisioning_mode = SD_LBP_DISABLE).
- [ ] AC-10: sd_eh_action: medium-access cmd timeout ∧ TUR success ∧ medium_access_timed_out ≥ max_medium_access_timeouts ⇒ set SDEV_OFFLINE and return SUCCESS.
- [ ] AC-11: sd_revalidate_disk: invokes READ CAPACITY 16 first when (max_cmd_len ≥ 16) ∧ (scsi_level > SPC-2 ∨ protection-capable ∨ !try_rc_10_first); on >32-bit capacity also sets use_16_for_rw.
- [ ] AC-12: sd_revalidate_disk: on ZBC drive, calls sd_zbc_revalidate_zones after set_capacity_and_notify; on zone-revalidate failure, clears capacity to 0.
- [ ] AC-13: sd_zbc_report_zones: on TYPE_ZBC iterates ZBC_IN/REPORT_ZONES with partial = true; on non-ZBC returns -EOPNOTSUPP.
- [ ] AC-14: sd_probe + device_add_disk triggers partition-table scan via block layer; sectors 0–N are read through sd_setup_read_write_cmnd.
- [ ] AC-15: init_sd registers 16 majors (8, 65..71, 128..135), creates sd_disk_class, creates sd_page_pool, registers sd_template.
- [ ] AC-16: sd_open returns -EROFS on write-prot disk opened BLK_OPEN_WRITE; -ENOMEDIUM on removable+no-media+!NDELAY.
- [ ] AC-17: sd_shutdown issues SYNCHRONIZE_CACHE only if WCE ∧ media_present; START_STOP UNIT only per manage_*_start_stop policy.

### Architecture

```
struct Disk {
  device: *ScsiDevice,
  disk_dev: Device,              // /sys/class/scsi_disk/<h:c:t:l>/
  disk: *GenDisk,
  opal_dev: Option<*OpalDev>,
  early_zone_info: ZonedDiskInfo,  // CONFIG_BLK_DEV_ZONED
  zone_info: ZonedDiskInfo,
  zone_starting_lba_gran: u32,
  openers: AtomicI32,
  capacity: u64,                 // sectors-in-logical-blocks
  max_retries: i32,
  min_xfer_blocks: u32,
  max_xfer_blocks: u32,
  opt_xfer_blocks: u32,
  max_ws_blocks: u32,
  max_unmap_blocks: u32,
  unmap_granularity: u32,
  unmap_alignment: u32,
  max_atomic: u32,
  atomic_alignment: u32,
  atomic_granularity: u32,
  max_atomic_with_boundary: u32,
  max_atomic_boundary: u32,
  index: u32,
  physical_block_size: u32,
  max_medium_access_timeouts: u32,
  medium_access_timed_out: u32,
  permanent_stream_count: u16,
  media_present: u8,
  write_prot: u8,
  protection_type: u8,
  provisioning_mode: u8,         // SD_LBP_*
  zeroing_mode: u8,
  nr_actuators: u8,
  suspended: bool,
  flags: DiskFlags,              // ATO, WCE, RCD, DPOFUA, lbpme, lbprz, lbpu, lbpws, lbpws10, ws10, ws16, rc_basis, zoned, security, ...
}

bitflags! struct DiskFlags : u32 {
  ATO, CACHE_OVERRIDE, WCE, RCD, DPOFUA, FIRST_SCAN,
  LBPME, LBPRZ, LBPU, LBPWS, LBPWS10, LBPVPD, WS10, WS16,
  URSWRZ, SECURITY, IGNORE_MEDIUM_ACCESS_ERRORS,
  RSCS, USE_ATOMIC_WRITE_BOUNDARY,
  RC_BASIS(u2), ZONED(u2),
}
```

`Sd::probe(sdp) -> Result<()>`:
1. scsi_autopm_get_device(sdp).
2. /* Type filter */
3. if sdp.type ∉ {TYPE_DISK, TYPE_ZBC, TYPE_MOD, TYPE_RBC}: return Err(-ENODEV).
4. if sdp.type == TYPE_ZBC ∧ !cfg!(CONFIG_BLK_DEV_ZONED): warn; return Err(-ENODEV).
5. sdkp = Disk::alloc_zeroed().
6. gd = blk_mq_alloc_disk_for_queue(sdp.request_queue, &sd_bio_compl_lkclass).
7. index = sd_index_ida.alloc().
8. Sd::format_disk_name("sd", index, &mut gd.disk_name).
9. sdkp.device = sdp; sdkp.disk = gd; sdkp.index = index; sdkp.max_retries = SD_MAX_RETRIES.
10. if !sdp.request_queue.rq_timeout: blk_queue_rq_timeout(q, if type==TYPE_MOD { SD_MOD_TIMEOUT } else { SD_TIMEOUT }).
11. sdkp.disk_dev.init(parent=sdev_dev, class=&sd_disk_class, name=sdev_name); device_add(&sdkp.disk_dev).
12. dev_set_drvdata(sdev_dev, sdkp).
13. gd.major = Sd::major((index & 0xf0) >> 4).
14. gd.first_minor = ((index & 0xf) << 4) | (index & 0xfff00); gd.minors = SD_MINORS = 16.
15. gd.fops = &sd_fops; gd.private_data = sdkp.
16. sdp.sector_size = 512; sdkp.media_present = 1; sdkp.first_scan = 1.
17. Disk::revalidate(gd).
18. if sdp.sector_size > PAGE_SIZE: Sd::large_pool_create().
19. if sdp.removable: gd.flags |= GENHD_FL_REMOVABLE; gd.events |= DISK_EVENT_MEDIA_CHANGE.
20. blk_pm_runtime_init(q, sdev_dev).
21. device_add_disk(sdev_dev, gd, NULL) /* scans partition table inline */.
22. if sdkp.security: sdkp.opal_dev = init_opal_dev(sdkp, &sd_sec_submit).
23. scsi_autopm_put_device(sdp).
24. Ok(()).

`Disk::format_disk_name(prefix, index, buf) -> Result<()>`:
1. base = 26.
2. begin = buf + prefix.len; end = buf + buflen.
3. p = end - 1; *p = '\0'; unit = base.
4. loop:
   - if p == begin: return Err(-EINVAL).
   - p -= 1; *p = b'a' + (index % unit) as u8.
   - index = (index / unit) - 1.
   - if index < 0: break.
5. memmove(begin, p, end - p); memcpy(buf, prefix, prefix.len).
6. Ok(()).

`Disk::init_command(cmd) -> BlkStatus`:
1. match req_op(cmd.rq) {
   REQ_OP_DISCARD => match sdkp.provisioning_mode {
     SD_LBP_UNMAP => Disk::setup_unmap(cmd),
     SD_LBP_WS16 => Disk::setup_write_same16(cmd, true),
     SD_LBP_WS10 => Disk::setup_write_same10(cmd, true),
     SD_LBP_ZERO => Disk::setup_write_same10(cmd, false),
     _ => BlkStatus::Target,
   },
   REQ_OP_WRITE_ZEROES => Disk::setup_write_zeroes(cmd),
   REQ_OP_FLUSH => Disk::setup_flush(cmd),
   REQ_OP_READ | REQ_OP_WRITE => Disk::setup_rw_cmnd(cmd),
   REQ_OP_ZONE_RESET => Disk::zbc_setup_zone_mgmt(cmd, ZO_RESET_WRITE_POINTER, false),
   REQ_OP_ZONE_RESET_ALL => Disk::zbc_setup_zone_mgmt(cmd, ZO_RESET_WRITE_POINTER, true),
   REQ_OP_ZONE_OPEN => Disk::zbc_setup_zone_mgmt(cmd, ZO_OPEN_ZONE, false),
   REQ_OP_ZONE_CLOSE => Disk::zbc_setup_zone_mgmt(cmd, ZO_CLOSE_ZONE, false),
   REQ_OP_ZONE_FINISH => Disk::zbc_setup_zone_mgmt(cmd, ZO_FINISH_ZONE, false),
   _ => { WARN_ON_ONCE; BlkStatus::NotSupp },
 }.

`Disk::setup_rw_cmnd(cmd) -> BlkStatus`:
1. lba = sectors_to_logical(sdp, blk_rq_pos(rq)).
2. nr_blocks = sectors_to_logical(sdp, blk_rq_sectors(rq)).
3. mask = logical_to_sectors(sdp, 1) - 1.
4. write = rq_data_dir(rq) == WRITE.
5. ret = Cmnd::alloc_sgtables(cmd); on err return ret.
6. if !online ∨ sdp.changed: goto fail Iorr.
7. if blk_rq_pos(rq) + blk_rq_sectors(rq) > get_capacity(disk): goto fail.
8. if (blk_rq_pos & mask) ∨ (blk_rq_sectors & mask): goto fail.
9. threshold = sdkp.capacity - SD_LAST_BUGGY_SECTORS.
10. if sdp.last_sector_bug ∧ lba + nr_blocks > threshold:
    - nr_blocks = if lba < threshold { threshold - lba } else { 1 }.
11. fua = if rq.cmd_flags & REQ_FUA { 0x8 } else { 0 }.
12. dix = scsi_prot_sg_count(cmd); dif = scsi_host_dif_capable(host, protection_type).
13. dld = Disk::cdl_dld(sdkp, cmd).
14. protect = if dif ∨ dix { Disk::setup_protect(cmd, dix, dif) } else { 0 }.
15. ret = if protect ∧ protection_type == TYPE2 {
        Disk::setup_rw32(cmd, write, lba, nr_blocks, protect | fua, dld)
      } else if rq.cmd_flags & REQ_ATOMIC {
        Disk::setup_atomic(cmd, lba, nr_blocks, use_atomic_write_boundary, protect | fua)
      } else if sdp.use_16_for_rw ∨ nr_blocks > 0xffff {
        Disk::setup_rw16(cmd, write, lba, nr_blocks, protect | fua, dld)
      } else if nr_blocks > 0xff ∨ lba > 0x1fffff ∨ sdp.use_10_for_rw ∨ protect != 0 ∨ rq.bio.bi_write_hint != 0 {
        Disk::setup_rw10(cmd, write, lba, nr_blocks, protect | fua)
      } else {
        Disk::setup_rw6(cmd, write, lba, nr_blocks, protect | fua)
      };
16. if ret != Ok: goto fail.
17. cmd.transfersize = sdp.sector_size.
18. cmd.underflow = nr_blocks << 9.
19. cmd.allowed = sdkp.max_retries.
20. cmd.sdb.length = nr_blocks * sdp.sector_size.
21. return Ok.
22. fail: Cmnd::free_sgtables(cmd); return ret.

`Disk::revalidate(gd)`:
1. old_capacity = sdkp.capacity.
2. if !online(sdp): return.
3. lim = QueueLimits::alloc(); buffer = kmalloc(SD_BUF_SIZE = 512, GFP_KERNEL).
4. Disk::spinup(sdkp) /* TUR-loop */.
5. *lim = queue_limits_start_update(q).
6. if sdkp.media_present:
   - Disk::read_capacity(sdkp, lim, buffer) /* RC16-first heuristic, fall through to RC10 */.
   - if sdp.read_before_ms: Disk::read_block_zero(sdkp).
   - lim.features |= BLK_FEAT_ROTATIONAL | BLK_FEAT_ADD_RANDOM.
   - if scsi_device_supports_vpd(sdp):
     - Disk::read_block_provisioning(sdkp).
     - Disk::read_block_limits(sdkp, lim).
     - Disk::read_block_limits_ext(sdkp).
     - Disk::read_block_characteristics(sdkp, lim) /* may clear BLK_FEAT_ROTATIONAL */.
     - Disk::zbc_read_zones(sdkp, lim, buffer).
   - Disk::config_discard(sdkp, lim, Disk::discard_mode(sdkp)).
   - Disk::print_capacity(sdkp, old_capacity).
   - Disk::read_wp_flag(sdkp, buffer).
   - Disk::read_cache_type(sdkp, buffer).
   - Disk::read_io_hints(sdkp, buffer).
   - Disk::read_app_tag_own(sdkp, buffer).
   - Disk::read_write_same(sdkp, buffer).
   - Disk::read_security(sdkp, buffer).
   - Disk::config_protection(sdkp, lim).
7. Disk::set_flush_flag(sdkp, lim) /* WCE → BLK_FEAT_WRITE_CACHE; FUA → BLK_FEAT_FUA */.
8. dev_max = if sdp.use_16_for_rw { SD_MAX_XFER_BLOCKS } else { SD_DEF_XFER_BLOCKS }; min_not_zero(sdkp.max_xfer_blocks).
9. lim.max_dev_sectors = logical_to_sectors(sdp, dev_max).
10. lim.io_min = if validate_min_xfer_size { logical_to_bytes(sdp, min_xfer_blocks) } else { 0 }.
11. lim.io_opt = sdp.host.opt_sectors << SECTOR_SHIFT; if validate_opt_xfer_size: min_not_zero(logical_to_bytes(sdp, opt_xfer_blocks)).
12. sdkp.first_scan = 0.
13. set_capacity_and_notify(gd, logical_to_sectors(sdp, sdkp.capacity)).
14. Disk::config_write_same(sdkp, lim).
15. queue_limits_commit_update_frozen(q, lim) — on err: jump to out (zone-revalidate skipped).
16. if media_present ∧ supports_vpd: Disk::read_cpr(sdkp).
17. if Disk::zbc_revalidate_zones(sdkp): set_capacity_and_notify(gd, 0).
18. out: kfree(buffer); kfree(lim).

`Disk::done(scmd) -> u32 /* good_bytes */`:
1. result = scmd.result; good_bytes = if result == 0 { scsi_bufflen(scmd) } else { 0 }.
2. sector_size = scmd.device.sector_size.
3. match req_op(req) {
   DISCARD | WRITE_ZEROES | ZONE_RESET | ZONE_RESET_ALL | ZONE_OPEN | ZONE_CLOSE | ZONE_FINISH => {
     good_bytes = if result == 0 { blk_rq_bytes(req) } else { 0 };
     scsi_set_resid(scmd, if result != 0 { blk_rq_bytes(req) } else { 0 });
   },
   _ /* r/w */ => {
     resid = scsi_get_resid(scmd);
     if resid & (sector_size - 1) {
       log "Unaligned partial completion";
       scsi_print_command(scmd);
       resid = min(scsi_bufflen(scmd), round_up(resid, sector_size));
       scsi_set_resid(scmd, resid);
     }
   },
 }.
4. if result != 0:
   - sense_valid = parse_sense(scmd, &mut sshdr).
   - sense_deferred = sense_valid ∧ sshdr.is_deferred().
5. sdkp.medium_access_timed_out = 0.
6. if !is_check_condition(result) ∨ !sense_valid ∨ sense_deferred: goto out.
7. match sshdr.sense_key {
   HARDWARE_ERROR | MEDIUM_ERROR => good_bytes = Disk::completed_bytes(scmd),
   RECOVERED_ERROR => good_bytes = scsi_bufflen(scmd),
   NO_SENSE => { scmd.result = 0; scmd.sense_buffer.zeroize(); },
   ABORTED_COMMAND if sshdr.asc == 0x10 => good_bytes = Disk::completed_bytes(scmd),
   ILLEGAL_REQUEST => match sshdr.asc {
     0x10 => good_bytes = Disk::completed_bytes(scmd),
     0x20 | 0x24 => match scmd.cmnd[0] {
       UNMAP => Disk::disable_discard(sdkp),
       WRITE_SAME_16 | WRITE_SAME => {
         if scmd.cmnd[1] & 0x8 { Disk::disable_discard(sdkp); }
         else { Disk::disable_write_same(sdkp); req.rq_flags |= RQF_QUIET; }
       },
       _ => (),
     },
     _ => (),
   },
   _ => (),
 }.
8. out: if scmd.device.type == TYPE_ZBC: good_bytes = Disk::zbc_complete(scmd, good_bytes, &sshdr).
9. return good_bytes.

### Out of Scope

- drivers/scsi/sd_dif.c full T10-PI host-side integration (covered in `block/integrity.md` future Tier-3; sd-side hook only documented here)
- drivers/scsi/sd_zbc.c full zone-state machine (only the surface referenced by sd_init_command / sd_revalidate_disk / sd_done is covered here; full state machine to be `sd-zbc.md` future Tier-3)
- drivers/scsi/sr.c (SCSI CD-ROM ULP — covered in `sr.md` future Tier-3)
- drivers/scsi/st.c (SCSI tape ULP — covered in `st.md` future Tier-3)
- drivers/scsi/sg.c (SCSI generic char-dev — covered in `sg.md` future Tier-3)
- drivers/scsi/ch.c (SCSI media-changer ULP)
- drivers/scsi/scsi_lib.c (covered in `scsi-lib.md`)
- drivers/scsi/scsi_error.c full EH state machine (covered in `scsi-error.md` future Tier-3)
- block/partitions/ MBR / GPT / Sun / Apple parse (covered in `block/partitions.md` future Tier-3; sd-side device_add_disk hook only mentioned)
- TCG-Opal full state machine (covered in `block/opal.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct scsi_disk` | per-disk control block | `kernel::scsi::sd::Disk` |
| `to_scsi_disk(obj)` / `scsi_disk(disk)` | container_of helpers | `Disk::from_class_dev` / `Disk::from_gendisk` |
| `struct scsi_driver sd_template` | per-ULP registration record | `kernel::scsi::sd::Template` |
| `init_sd()` / `exit_sd()` | module entry/exit (16 major numbers + class + page-pool + driver register) | `Sd::init` / `Sd::exit` |
| `sd_default_probe(devt)` | per-blkdev request-module trigger | `Sd::default_probe` |
| `sd_major(idx)` | per-index major-number table (8, 65..71, 128..135) | `Sd::major` |
| `sd_probe(sdp)` | per-scsi_device probe → bind gendisk | `Sd::probe` |
| `sd_remove(sdp)` | per-scsi_device remove → del_gendisk + class device_del | `Sd::remove` |
| `sd_shutdown(sdp)` | per-shutdown SYNC CACHE + START/STOP UNIT | `Sd::shutdown` |
| `sd_format_disk_name(prefix, idx, buf, buflen)` | per-index → "sda".."sdz","sdaa".."sdzz","sdaaa",... | `Sd::format_disk_name` |
| `sd_init_command(cmd)` | per-rq → CDB-builder dispatch by req_op | `Disk::init_command` |
| `sd_uninit_command(scmd)` | per-rq cleanup (free special-bvec for WRITE_SAME/UNMAP) | `Disk::uninit_command` |
| `sd_setup_read_write_cmnd(cmd)` | per-rw-cmd CDB selection (6/10/16/32 + atomic + DIF/DIX) | `Disk::setup_rw_cmnd` |
| `sd_setup_rw32_cmnd(cmd, write, lba, n, flags, dld)` | per-T10-PI-TYPE2 VARIABLE_LENGTH 32B CDB | `Disk::setup_rw32` |
| `sd_setup_rw16_cmnd(cmd, write, lba, n, flags, dld)` | per-READ_16 / WRITE_16 builder | `Disk::setup_rw16` |
| `sd_setup_rw10_cmnd(cmd, write, lba, n, flags)` | per-READ_10 / WRITE_10 builder | `Disk::setup_rw10` |
| `sd_setup_rw6_cmnd(cmd, write, lba, n, flags)` | per-READ_6 / WRITE_6 builder (legacy small-LBA) | `Disk::setup_rw6` |
| `sd_setup_atomic_cmnd(cmd, lba, n, boundary, flags)` | per-WRITE_ATOMIC_16 builder | `Disk::setup_atomic` |
| `sd_setup_unmap_cmnd(cmd)` | per-UNMAP (10B CDB + 24B parameter list) | `Disk::setup_unmap` |
| `sd_setup_write_same16_cmnd(cmd, unmap)` | per-WRITE_SAME_16 builder | `Disk::setup_write_same16` |
| `sd_setup_write_same10_cmnd(cmd, unmap)` | per-WRITE_SAME (10B) builder | `Disk::setup_write_same10` |
| `sd_setup_write_zeroes_cmnd(cmd)` | per-REQ_OP_WRITE_ZEROES dispatch (WS16/WS10 unmap | WS10 zero) | `Disk::setup_write_zeroes` |
| `sd_setup_flush_cmnd(cmd)` | per-REQ_OP_FLUSH → SYNCHRONIZE_CACHE / _16 | `Disk::setup_flush` |
| `sd_setup_protect_cmnd(scmd, dix, dif)` | per-cmd T10-PI prot_op + RDPROTECT/WRPROTECT byte | `Disk::setup_protect` |
| `sd_prot_op(write, dix, dif)` | per-(write, dix, dif) → SCSI_PROT_* enum | `Disk::prot_op` |
| `sd_prot_flag_mask(prot_op)` | per-prot_op → valid PROT_GUARD/REF/IP/TRANSFER mask | `Disk::prot_flag_mask` |
| `sd_cdl_dld(sdkp, scmd)` | per-cmd CDL duration-limit descriptor index | `Disk::cdl_dld` |
| `sd_group_number(cmd)` | per-cmd group-number from write-hint | `Disk::group_number` |
| `sd_done(SCpnt)` | per-cmd ULP completion bottom-half | `Disk::done` |
| `sd_completed_bytes(scmd)` | per-cmd partial-completion via INFORMATION-sense field | `Disk::completed_bytes` |
| `sd_revalidate_disk(disk)` | per-disk online refresh (capacity + VPD + caching + zones) | `Disk::revalidate` |
| `sd_need_revalidate(disk, sdkp)` | per-disk gate (media-change ∨ removable ∨ BLKRRPART) | `Disk::need_revalidate` |
| `sd_read_capacity(sdkp, lim, buffer)` | per-disk dispatch READ CAPACITY 10/16 | `Disk::read_capacity` |
| `read_capacity_10(sdkp, sdp, buffer)` | per-READ_CAPACITY(10) (32-bit max LBA, 32-bit block size) | `Disk::read_capacity_10` |
| `read_capacity_16(sdkp, sdp, lim, buffer)` | per-READ_CAPACITY(16) (64-bit max LBA + phys-block + LBPME + LBPRZ + protection) | `Disk::read_capacity_16` |
| `sd_try_rc16_first(sdp)` | per-disk policy (max_cmd_len ≥ 16 ∧ SPC-2+ ∨ protection-capable) | `Disk::try_rc16_first` |
| `sd_read_block_provisioning(sdkp)` | per-VPD-0xb2 (LBPME, LBPRZ, LBPU, LBPWS, LBPWS10) | `Disk::read_block_provisioning` |
| `sd_read_block_limits(sdkp, lim)` | per-VPD-0xb0 (max-unmap, opt-xfer, max-xfer, write-same, atomic-write) | `Disk::read_block_limits` |
| `sd_read_block_limits_ext(sdkp)` | per-VPD-0xb7 (atomic-write extended) | `Disk::read_block_limits_ext` |
| `sd_read_block_characteristics(sdkp, lim)` | per-VPD-0xb1 (RPM, form-factor, zoned, WAB exceeded) | `Disk::read_block_characteristics` |
| `sd_read_write_protect_flag(sdkp, buffer)` | per-MODE-SENSE WP-bit | `Disk::read_wp_flag` |
| `sd_read_cache_type(sdkp, buffer)` | per-MODE-SENSE-Page-0x08 (WCE, RCD, FUA, DPOFUA) | `Disk::read_cache_type` |
| `sd_read_io_hints(sdkp, buffer)` | per-VPD-0xb5 (concurrent positioning + streams) | `Disk::read_io_hints` |
| `sd_read_app_tag_own(sdkp, buffer)` | per-mode-page T10 app-tag owner | `Disk::read_app_tag_own` |
| `sd_read_write_same(sdkp, buffer)` | per-mode-page max-write-same-blocks | `Disk::read_write_same` |
| `sd_read_security(sdkp, buffer)` | per-VPD-0x86 / TCG-Opal advertisement | `Disk::read_security` |
| `sd_read_cpr(sdkp)` | per-VPD-0xb9 concurrent positioning ranges | `Disk::read_cpr` |
| `sd_zbc_read_zones(sdkp, lim, buffer)` | per-ZBC zone-info inventory (called via revalidate) | `Disk::zbc_read_zones` |
| `sd_zbc_revalidate_zones(sdkp)` | per-ZBC zone-map refresh after capacity set | `Disk::zbc_revalidate_zones` |
| `sd_zbc_setup_zone_mgmt_cmnd(cmd, op, all)` | per-REQ_OP_ZONE_* → ZBC_OUT CDB (RESET_WP/OPEN/CLOSE/FINISH) | `Disk::zbc_setup_zone_mgmt` |
| `sd_zbc_report_zones(disk, sector, nr, args)` | per-blockdev .report_zones callback (ZBC_IN/REPORT_ZONES) | `Disk::zbc_report_zones` |
| `sd_zbc_complete(scmd, good_bytes, sshdr)` | per-ZBC sense-translation | `Disk::zbc_complete` |
| `sd_eh_action(scmd, eh_disp)` | per-EH callback (medium-access timeout → offline) | `Disk::eh_action` |
| `sd_eh_reset(scmd)` | per-EH-run reset of medium-access-error gate | `Disk::eh_reset` |
| `sd_open(disk, mode)` | per-open (block error-recovery + revalidate + write-prot + medium-removal lock) | `Disk::open` |
| `sd_release(disk)` | per-close (medium-removal unlock + put) | `Disk::release` |
| `sd_check_events(disk, clearing)` | per-media-change poll | `Disk::check_events` |
| `sd_ioctl(bdev, mode, cmd, arg)` | per-ioctl pass-through | `Disk::ioctl` |
| `sd_getgeo(disk, geo)` | per-HDIO_GETGEO (heads/sectors/cylinders heuristic) | `Disk::getgeo` |
| `sd_get_unique_id(disk, id[16], type)` | per-disk unique-id (NAA/EUI64/T10-vendor) for blk-crypto | `Disk::get_unique_id` |
| `sd_sec_submit(data, spsp, secp, buf, len, send)` | per-Opal SECURITY_PROTOCOL_IN/OUT submit | `Disk::sec_submit` |
| `sd_pr_register/_reserve/_release/_preempt/_clear/_read_keys/_read_reservation` | per-PR-ops (PERSISTENT RESERVE IN/OUT) | `Disk::pr_*` |
| `sd_sync_cache(sdkp)` | per-disk SYNCHRONIZE_CACHE / _16 | `Disk::sync_cache` |
| `sd_start_stop_device(sdkp, start)` | per-disk START_STOP UNIT | `Disk::start_stop` |
| `sd_suspend_system/_runtime/_resume_system/_resume_runtime` | per-PM hooks | `Disk::suspend_*` / `_resume_*` |
| `sd_rescan(dev)` | per-rescan trigger (sd_revalidate_disk + blk_revalidate_disk) | `Disk::rescan` |
| `sd_config_discard(sdkp, lim, mode)` | per-disk discard-mode → SD_LBP_UNMAP / _WS16 / _WS10 / _ZERO / _DISABLE | `Disk::config_discard` |
| `sd_config_write_same(sdkp, lim)` | per-disk write-same-blocks limit | `Disk::config_write_same` |
| `sd_config_atomic(sdkp, lim)` | per-disk atomic-write limits (unit min/max + boundary) | `Disk::config_atomic` |
| `sd_config_protection(sdkp, lim)` | per-disk T10-PI / DIX configuration | `Disk::config_protection` |
| `sd_set_flush_flag(sdkp, lim)` | per-disk BLK_FEAT_WRITE_CACHE / _FUA derivation from WCE/FUA | `Disk::set_flush_flag` |
| `sd_disable_discard(sdkp)` / `sd_disable_write_same(sdkp)` | per-disk quirk-disable on INVALID-OPCODE sense | `Disk::disable_discard` / `_write_same` |
| `scsi_disk_release(dev)` | per-class device release | `Disk::release_class_dev` |
| `scsi_disk_free_disk(disk)` | per-gendisk free hook | `Disk::free_disk` |

### compatibility contract

REQ-1: struct scsi_disk:
- device: per-sdev back-pointer.
- disk_dev: per-`/sys/class/scsi_disk/<h:c:t:l>/` device.
- disk: per-gendisk.
- opal_dev: per-Opal-NVMe-equivalent dev (NULL if not security-capable).
- early_zone_info / zone_info (CONFIG_BLK_DEV_ZONED): per-ZBC zone metadata.
- zone_starting_lba_gran: per-ZBC zone-LBA stride (0 if non-uniform).
- openers: per-disk atomic open-count.
- capacity: per-disk size in logical blocks.
- max_retries: per-disk ML-retry budget (default SD_MAX_RETRIES = 5).
- min_xfer_blocks / max_xfer_blocks / opt_xfer_blocks: per-VPD-0xb0.
- max_ws_blocks / max_unmap_blocks: per-VPD-0xb0.
- unmap_granularity / unmap_alignment: per-VPD-0xb0.
- max_atomic / atomic_alignment / atomic_granularity / max_atomic_with_boundary / max_atomic_boundary: per-VPD-0xb7.
- index: per-disk index → /dev/sd<letter(s)>.
- physical_block_size: per-VPD-0xb0.
- max_medium_access_timeouts / medium_access_timed_out: per-disk EH-offline thresholds.
- permanent_stream_count: per-VPD-0xb5.
- media_present / write_prot / protection_type / provisioning_mode / zeroing_mode: per-disk state.
- ATO / cache_override / WCE / RCD / DPOFUA: per-MODE-SENSE-Page-0x08 bits.
- first_scan: 1 on first revalidate; affects log verbosity.
- lbpme / lbprz / lbpu / lbpws / lbpws10 / lbpvpd / ws10 / ws16: per-VPD-0xb2 thin-prov flags.
- rc_basis / zoned / urswrz / security / rscs / use_atomic_write_boundary: per-VPD-0xb1 / 0xb6 / 0xb9 flags.

REQ-2: sd_format_disk_name(prefix, index, buf, buflen):
- /* Maps index ∈ [0, ∞) → "sda".."sdz","sdaa".."sdzz","sdaaa",... */
- base = 'z' - 'a' + 1 = 26.
- begin = buf + strlen(prefix); end = buf + buflen.
- p = end - 1; *p = '\0'; unit = base.
- do:
  - if p == begin: return -EINVAL /* exceeded buflen */.
  - *--p = 'a' + (index % unit).
  - index = (index / unit) - 1 /* the -1 makes the first digit "a-z" and second-and-later "a-z" with an implied leading nil */.
- while (index >= 0).
- memmove(begin, p, end - p); memcpy(buf, prefix, strlen(prefix)).
- return 0.

REQ-3: sd_probe(sdp):
- scsi_autopm_get_device(sdp) /* keep awake during probe */.
- /* Filter device-type: TYPE_DISK / TYPE_ZBC (if CONFIG_BLK_DEV_ZONED) / TYPE_MOD / TYPE_RBC */
- if !{TYPE_DISK, TYPE_ZBC, TYPE_MOD, TYPE_RBC}.contains(sdp->type): -ENODEV; out.
- if sdp->type == TYPE_ZBC ∧ !CONFIG_BLK_DEV_ZONED: warn; -ENODEV; out.
- sdkp = kzalloc_obj(*sdkp) /* GFP_KERNEL zeroed scsi_disk */.
- gd = blk_mq_alloc_disk_for_queue(sdp->request_queue, &sd_bio_compl_lkclass) /* attach gendisk to existing per-sdev request_queue */.
- index = ida_alloc(&sd_index_ida, GFP_KERNEL).
- sd_format_disk_name("sd", index, gd->disk_name, DISK_NAME_LEN).
- sdkp->device = sdp; sdkp->disk = gd; sdkp->index = index; sdkp->max_retries = SD_MAX_RETRIES.
- if !sdp->request_queue->rq_timeout: blk_queue_rq_timeout(q, type == MOD ? SD_MOD_TIMEOUT : SD_TIMEOUT).
- device_initialize(&sdkp->disk_dev); .parent = sdev_dev; .class = &sd_disk_class; dev_set_name; device_add.
- dev_set_drvdata(dev, sdkp).
- gd->major = sd_major((index & 0xf0) >> 4); gd->first_minor = ((index & 0xf) << 4) | (index & 0xfff00); gd->minors = SD_MINORS = 16.
- gd->fops = &sd_fops; gd->private_data = sdkp.
- /* defaults until device-told */
- sdp->sector_size = 512; sdkp->capacity = 0; sdkp->media_present = 1; sdkp->first_scan = 1; sdkp->max_medium_access_timeouts = SD_MAX_MEDIUM_TIMEOUTS.
- sd_revalidate_disk(gd) /* may issue READ CAPACITY, MODE SENSE, INQUIRY-VPD */.
- if sdp->sector_size > PAGE_SIZE: sd_large_pool_create() /* GFP_KERNEL mempool for special-bvec on > PAGE_SIZE LBA */.
- if sdp->removable: gd->flags |= GENHD_FL_REMOVABLE; gd->events |= DISK_EVENT_MEDIA_CHANGE.
- blk_pm_runtime_init(q, dev); if rpm_autosuspend: set delay.
- device_add_disk(dev, gd, NULL) /* exposes /dev/sd<N>, scans partitions */.
- if sdkp->security: sdkp->opal_dev = init_opal_dev(sdkp, &sd_sec_submit).
- scsi_autopm_put_device(sdp); return 0.

REQ-4: sd_remove(sdp):
- sdkp = dev_get_drvdata(&sdp->sdev_gendev).
- scsi_autopm_get_device(sdkp->device).
- device_del(&sdkp->disk_dev) /* sysfs class device gone */.
- del_gendisk(sdkp->disk) /* drains in-flight, removes /dev/sd<N>, exits partitions */.
- if !sdkp->suspended: sd_shutdown(sdp) /* SYNC CACHE + START_STOP UNIT */.
- put_disk(sdkp->disk).
- if sdp->sector_size > PAGE_SIZE: sd_large_pool_destroy().

REQ-5: sd_shutdown(sdp):
- if !sdkp ∨ pm_runtime_suspended(dev): return.
- if sdkp->WCE ∧ sdkp->media_present: sd_sync_cache(sdkp) /* SYNCHRONIZE_CACHE / _16 */.
- per-system_state policy (manage_system_start_stop / manage_shutdown / manage_runtime_start_stop / manage_restart) → sd_start_stop_device(sdkp, 0) /* spin-down */.

REQ-6: sd_init_command(cmd):
- /* Per-rq dispatch — invoked by scsi_prepare_cmd via scsi_cmd_to_driver(cmd)->init_command */
- switch (req_op(rq)):
  - REQ_OP_DISCARD: per-sdkp->provisioning_mode dispatch:
    - SD_LBP_UNMAP → sd_setup_unmap_cmnd.
    - SD_LBP_WS16 → sd_setup_write_same16_cmnd(cmd, unmap=true).
    - SD_LBP_WS10 → sd_setup_write_same10_cmnd(cmd, unmap=true).
    - SD_LBP_ZERO → sd_setup_write_same10_cmnd(cmd, unmap=false).
    - default → BLK_STS_TARGET /* SD_LBP_DISABLE / unsupported */.
  - REQ_OP_WRITE_ZEROES → sd_setup_write_zeroes_cmnd.
  - REQ_OP_FLUSH → sd_setup_flush_cmnd.
  - REQ_OP_READ | REQ_OP_WRITE → sd_setup_read_write_cmnd.
  - REQ_OP_ZONE_RESET → sd_zbc_setup_zone_mgmt_cmnd(cmd, ZO_RESET_WRITE_POINTER, all=false).
  - REQ_OP_ZONE_RESET_ALL → ZO_RESET_WRITE_POINTER, all=true.
  - REQ_OP_ZONE_OPEN → ZO_OPEN_ZONE.
  - REQ_OP_ZONE_CLOSE → ZO_CLOSE_ZONE.
  - REQ_OP_ZONE_FINISH → ZO_FINISH_ZONE.
  - default → WARN_ON_ONCE; BLK_STS_NOTSUPP.

REQ-7: sd_setup_read_write_cmnd(cmd):
- /* The hot path */
- lba = sectors_to_logical(sdp, blk_rq_pos(rq)).
- nr_blocks = sectors_to_logical(sdp, blk_rq_sectors(rq)).
- mask = logical_to_sectors(sdp, 1) - 1 /* sector-size alignment mask */.
- write = (rq_data_dir(rq) == WRITE).
- ret = scsi_alloc_sgtables(cmd) /* sgs first */; on err return ret.
- if !online ∨ sdp->changed: log; goto fail BLK_STS_IOERR.
- if blk_rq_pos(rq) + blk_rq_sectors(rq) > get_capacity(disk): log "beyond end"; fail.
- if (blk_rq_pos(rq) & mask) ∨ (blk_rq_sectors(rq) & mask): log "unaligned"; fail.
- /* SD-card-reader last-sector quirk */
- threshold = sdkp->capacity - SD_LAST_BUGGY_SECTORS.
- if sdp->last_sector_bug ∧ lba + nr_blocks > threshold:
  - nr_blocks = max(threshold - lba, 1) /* possibly split; block layer will requeue remainder */.
- fua = (rq->cmd_flags & REQ_FUA) ? 0x8 : 0.
- dix = scsi_prot_sg_count(cmd); dif = scsi_host_dif_capable(host, sdkp->protection_type).
- dld = sd_cdl_dld(sdkp, cmd) /* CDL duration-limit descriptor index ∈ {0..7} */.
- if dif ∨ dix: protect = sd_setup_protect_cmnd(cmd, dix, dif) /* sets cmd->prot_op, prot_flags */.
- else: protect = 0.
- /* CDB-selection ladder */
- if protect ∧ sdkp->protection_type == T10_PI_TYPE2_PROTECTION: sd_setup_rw32_cmnd(cmd, write, lba, nr_blocks, protect | fua, dld) /* 32B VARIABLE_LENGTH_CMD */.
- elif rq->cmd_flags & REQ_ATOMIC: sd_setup_atomic_cmnd(cmd, lba, nr_blocks, sdkp->use_atomic_write_boundary, protect | fua) /* WRITE_ATOMIC_16 */.
- elif sdp->use_16_for_rw ∨ nr_blocks > 0xffff: sd_setup_rw16_cmnd(cmd, write, lba, nr_blocks, protect | fua, dld).
- elif nr_blocks > 0xff ∨ lba > 0x1fffff ∨ sdp->use_10_for_rw ∨ protect ∨ rq->bio->bi_write_hint: sd_setup_rw10_cmnd(cmd, write, lba, nr_blocks, protect | fua).
- else: sd_setup_rw6_cmnd(cmd, write, lba, nr_blocks, protect | fua).
- if ret != BLK_STS_OK: goto fail.
- cmd->transfersize = sdp->sector_size; cmd->underflow = nr_blocks << 9.
- cmd->allowed = sdkp->max_retries.
- cmd->sdb.length = nr_blocks * sdp->sector_size.
- return BLK_STS_OK.
- fail: scsi_free_sgtables(cmd); return ret.

REQ-8: CDB byte layouts (the per-op encoding):
- READ_6 / WRITE_6 (6-byte, 21-bit LBA, 8-bit count, no FUA):
  - cmd[0] = 0x08 (READ_6) | 0x0a (WRITE_6).
  - cmd[1] = (lba >> 16) & 0x1f.
  - cmd[2..3] = (lba >> 8) & 0xff, lba & 0xff.
  - cmd[4] = nr_blocks /* WARN if 0 (would map to 256) */.
  - cmd[5] = 0.
- READ_10 / WRITE_10 (10-byte, 32-bit LBA, 16-bit count, FUA/DPO supported):
  - cmd[0] = 0x28 (READ_10) | 0x2a (WRITE_10).
  - cmd[1] = flags /* RDPROTECT/WRPROTECT in [7:5], DPO bit 4, FUA bit 3 */.
  - cmd[2..5] = BE32(lba).
  - cmd[6] = sd_group_number(cmd).
  - cmd[7..8] = BE16(nr_blocks).
  - cmd[9] = 0.
- READ_16 / WRITE_16 (16-byte, 64-bit LBA, 32-bit count, DLD for CDL):
  - cmd[0] = 0x88 (READ_16) | 0x8a (WRITE_16).
  - cmd[1] = flags | ((dld >> 2) & 0x01).
  - cmd[2..9] = BE64(lba).
  - cmd[10..13] = BE32(nr_blocks).
  - cmd[14] = ((dld & 0x03) << 6) | sd_group_number(cmd).
  - cmd[15] = 0.
- READ_32 / WRITE_32 (VARIABLE_LENGTH 32-byte; T10 PI TYPE 2 mandates the expected-block-app-tag fields):
  - cmd[0] = VARIABLE_LENGTH_CMD = 0x7f.
  - cmd[6] = group_number; cmd[7] = 0x18 (additional CDB len).
  - cmd[9] = WRITE_32 / READ_32 service action.
  - cmd[10] = flags; cmd[11] = dld & 0x07.
  - cmd[12..19] = BE64(lba).
  - cmd[20..23] = BE32(lba) /* Expected Indirect LBA */.
  - cmd[28..31] = BE32(nr_blocks).
- UNMAP (10B, payload-carries-LBA-list):
  - cmd[0] = UNMAP = 0x42.
  - cmd[8] = 24 (allocation length).
  - Param-list (24 bytes): header[0..1] = BE16(22); header[2..3] = BE16(16); BE64(lba) at offset 8; BE32(nr_blocks) at offset 16.
  - timeout = SD_TIMEOUT.
- WRITE_SAME (10B):
  - cmd[0] = WRITE_SAME = 0x41.
  - cmd[1] = unmap ? 0x8 : 0.
  - cmd[2..5] = BE32(lba); cmd[7..8] = BE16(nr_blocks).
  - timeout = unmap ? SD_TIMEOUT : SD_WRITE_SAME_TIMEOUT.
- WRITE_SAME_16 (16B):
  - cmd[0] = WRITE_SAME_16 = 0x93.
  - cmd[1] = unmap ? 0x8 : 0.
  - cmd[2..9] = BE64(lba); cmd[10..13] = BE32(nr_blocks).
- WRITE_ATOMIC_16 (16B):
  - cmd[0] = WRITE_ATOMIC_16 = 0x9c.
  - cmd[1] = flags.
  - cmd[2..9] = BE64(lba); cmd[12..13] = BE16(nr_blocks).
  - cmd[10..11] = boundary ? BE16(nr_blocks) : 0.
- SYNCHRONIZE_CACHE (10B) / _16 (16B):
  - cmd[0] = SYNCHRONIZE_CACHE = 0x35 | SYNCHRONIZE_CACHE_16 = 0x91 (use_16_for_sync).
- ZBC_IN/REPORT_ZONES (16B): cmd[0] = ZBC_IN = 0x95; cmd[1] = ZI_REPORT_ZONES = 0; BE64(lba) at [2]; BE32(buflen) at [10]; cmd[14] = ZBC_REPORT_ZONE_PARTIAL when partial.

REQ-9: T10-PI protection setup (sd_setup_protect_cmnd):
- prot_op = sd_prot_op(write, dix, dif) /* 3-bit lookup */:
  - (0,0,0) → NORMAL; (0,0,1) → READ_STRIP; (0,1,0) → READ_INSERT; (0,1,1) → READ_PASS.
  - (1,0,0) → NORMAL; (1,0,1) → WRITE_INSERT; (1,1,0) → WRITE_STRIP; (1,1,1) → WRITE_PASS.
- scmd->prot_op = prot_op; scmd->prot_flags = bio-derived flags ∩ sd_prot_flag_mask(prot_op).
- If bio_integrity_flagged(bio, BIP_IP_CHECKSUM): scmd->prot_flags |= SCSI_PROT_IP_CHECKSUM.
- Returns: protect byte for CDB[1]'s top 3 bits (RDPROTECT/WRPROTECT) based on protection_type.

REQ-10: sd_done(SCpnt):
- result = SCpnt->result; good_bytes = result ? 0 : scsi_bufflen(SCpnt); sector_size = sdp->sector_size.
- switch (req_op(req)):
  - REQ_OP_DISCARD | _WRITE_ZEROES | _ZONE_RESET | _ZONE_RESET_ALL | _ZONE_OPEN | _ZONE_CLOSE | _ZONE_FINISH:
    - good_bytes = result ? 0 : blk_rq_bytes(req).
    - scsi_set_resid(SCpnt, result ? blk_rq_bytes(req) : 0).
  - default (r/w):
    - resid = scsi_get_resid(SCpnt).
    - if resid & (sector_size - 1): log "unaligned"; scsi_print_command; round up to sector_size; scsi_set_resid.
- if result: sense_valid = scsi_command_normalize_sense(SCpnt, &sshdr); sense_deferred = sense_valid ∧ scsi_sense_is_deferred.
- sdkp->medium_access_timed_out = 0 /* normal completion clears the EH-counter */.
- if !is_check_condition(result) ∨ (!sense_valid ∨ sense_deferred): goto out.
- switch (sshdr.sense_key):
  - HARDWARE_ERROR | MEDIUM_ERROR: good_bytes = sd_completed_bytes(SCpnt) /* INFORMATION-field bad-LBA → partial-bytes-good */.
  - RECOVERED_ERROR: good_bytes = scsi_bufflen(SCpnt) /* full success */.
  - NO_SENSE: SCpnt->result = 0; clear sense /* false check-condition */.
  - ABORTED_COMMAND: if asc == 0x10 (DIF target corruption): good_bytes = sd_completed_bytes.
  - ILLEGAL_REQUEST:
    - asc == 0x10: good_bytes = sd_completed_bytes (DIX host corruption).
    - asc ∈ {0x20, 0x24} ∧ cmd[0] ∈ {UNMAP}: sd_disable_discard(sdkp).
    - asc ∈ {0x20, 0x24} ∧ cmd[0] ∈ {WRITE_SAME_16, WRITE_SAME}:
      - if cmd[1] & 0x8 /* UNMAP */: sd_disable_discard(sdkp).
      - else: sd_disable_write_same(sdkp); rq->rq_flags |= RQF_QUIET.
- out: if TYPE_ZBC: good_bytes = sd_zbc_complete(SCpnt, good_bytes, &sshdr).
- return good_bytes /* fed into scsi_io_completion */.

REQ-11: sd_eh_action(scmd, eh_disp):
- sdkp = scsi_disk(req->q->disk); sdev = scmd->device.
- if !online ∨ !scsi_medium_access_command(scmd) ∨ host_byte(result) != DID_TIME_OUT ∨ eh_disp != SUCCESS: return eh_disp.
- /* Medium-access cmd timed out but EH TEST UNIT READY succeeded */
- if !sdkp->ignore_medium_access_errors: sdkp->medium_access_timed_out++; sdkp->ignore_medium_access_errors = true.
- if sdkp->medium_access_timed_out >= sdkp->max_medium_access_timeouts:
  - log "Offlining disk".
  - scsi_device_set_state(sdev, SDEV_OFFLINE) /* under sdev->state_mutex */.
  - return SUCCESS /* EH succeeds; rq is failed */.
- return eh_disp.

REQ-12: sd_revalidate_disk(disk):
- old_capacity = sdkp->capacity.
- if !scsi_device_online(sdp): return.
- lim = kmalloc_obj(*lim); buffer = kmalloc(SD_BUF_SIZE = 512, GFP_KERNEL).
- sd_spinup_disk(sdkp) /* TEST UNIT READY loop until media-ready or no-medium */.
- *lim = queue_limits_start_update(disk->queue).
- if media_present:
  - sd_read_capacity(sdkp, lim, buffer) /* RC16 first if max_cmd_len ≥ 16, else RC10; on overflow retry RC16 */.
  - if read_before_ms: sd_read_block_zero(sdkp) /* USB/UAS workaround */.
  - lim->features |= BLK_FEAT_ROTATIONAL | BLK_FEAT_ADD_RANDOM.
  - if scsi_device_supports_vpd:
    - sd_read_block_provisioning (LBPME/LBPRZ/LBPU/LBPWS/LBPWS10).
    - sd_read_block_limits (max_unmap, opt_xfer, max_xfer, atomic-write hw caps).
    - sd_read_block_limits_ext (atomic-write extended).
    - sd_read_block_characteristics (RPM, form-factor, zoned, BLK_FEAT_ROTATIONAL clearing).
    - sd_zbc_read_zones (if TYPE_ZBC).
  - sd_config_discard(sdkp, lim, sd_discard_mode(sdkp)) /* maps LBPU/LBPWS/LBPWS10 → SD_LBP_* */.
  - sd_print_capacity(sdkp, old_capacity).
  - sd_read_write_protect_flag (MODE SENSE).
  - sd_read_cache_type (MODE SENSE Page 0x08 — WCE/RCD/FUA/DPOFUA).
  - sd_read_io_hints (VPD 0xb5 streams).
  - sd_read_app_tag_own.
  - sd_read_write_same.
  - sd_read_security (VPD 0x86 / TCG Opal advertisement).
  - sd_config_protection(sdkp, lim).
- sd_set_flush_flag(sdkp, lim) /* WCE → BLK_FEAT_WRITE_CACHE; FUA → BLK_FEAT_FUA */.
- dev_max = sdp->use_16_for_rw ? SD_MAX_XFER_BLOCKS : SD_DEF_XFER_BLOCKS; min_not_zero(max_xfer_blocks).
- lim->max_dev_sectors = logical_to_sectors(sdp, dev_max).
- lim->io_min / io_opt: derived from min_xfer_blocks / opt_xfer_blocks (validated).
- sdkp->first_scan = 0.
- set_capacity_and_notify(disk, logical_to_sectors(sdp, sdkp->capacity)).
- sd_config_write_same(sdkp, lim).
- queue_limits_commit_update_frozen(disk->queue, lim) /* atomic commit; on err leave queue stopped */.
- if media_present ∧ supports_vpd: sd_read_cpr(sdkp) /* after limits_commit unlocked */.
- if sd_zbc_revalidate_zones(sdkp): set_capacity_and_notify(disk, 0) /* zone-revalidate failed; force fail-closed */.

REQ-13: Partition-table read:
- sd_probe → device_add_disk(dev, gd, NULL).
- device_add_disk calls disk_scan_partitions which reads sector 0 + 1 via the gendisk's request_queue → blk_mq submits READ requests → sd_init_command → sd_setup_read_write_cmnd → SCSI READ_10/READ_16 → LLD → completion → sd_done → scsi_io_completion → block layer parses MBR / GPT / Sun / Apple labels (block/partitions/) → adds per-partition block_device.
- On revalidate after media change: blk_revalidate_disk_zones / disk_scan_partitions re-triggered through sd_revalidate_disk + set_capacity_and_notify.

REQ-14: REPORT ZONES (sd_zbc.c, invoked from sd_fops.report_zones):
- sd_zbc_report_zones(disk, sector, nr_zones, args):
  - if sdp->type != TYPE_ZBC: return -EOPNOTSUPP.
  - if !capacity: return -ENODEV.
  - buf = sd_zbc_alloc_report_buffer(sdkp, nr_zones, &buflen) /* bounded by queue_max_hw_sectors + queue_max_segments + BIO_MAX_INLINE_VECS */.
  - while zone_idx < nr_zones ∧ lba < capacity:
    - sd_zbc_do_report_zones(sdkp, buf, buflen, lba, partial=true):
      - cmd = { ZBC_IN, ZI_REPORT_ZONES, BE64(lba) at [2], BE32(buflen) at [10], cmd[14] = ZBC_REPORT_ZONE_PARTIAL }.
      - scsi_execute_cmd(REQ_OP_DRV_IN, ...).
      - parse 64-byte header + 64-byte descriptors; call args->report_zone_cb per descriptor.
    - advance lba; bump zone_idx.
  - kvfree(buf); return zone_idx.

REQ-15: MQ tag-set integration:
- sd uses the per-host blk_mq_tag_set allocated by scsi_mq_setup_tags (covered in scsi-lib.md REQ-18).
- sd does not own the tag-set; it shares the per-shost tag-set among every sdev under that host.
- gendisk → request_queue allocated via blk_mq_alloc_disk_for_queue uses the same tag-set, so scsi_disk(gd)->device → sdev → sdev->host->tag_set is the single source of MQ-rq pdus.
- Per-rq cmd_size in tag_set includes hostt->cmd_size + inline sg-list, so sd never allocates pdu memory itself.

REQ-16: Discard mode policy (sd_config_discard / sd_discard_mode):
- sd_discard_mode(sdkp):
  - if !sdkp->lbpme: return SD_LBP_FULL /* no thin-prov advertisement */.
  - if sdkp->lbpws: return SD_LBP_WS16 /* WRITE_SAME(16) + UNMAP */.
  - if sdkp->lbpws10: return SD_LBP_WS10 /* WRITE_SAME(10) + UNMAP */.
  - if sdkp->lbpu ∧ sdkp->max_unmap_blocks: return SD_LBP_UNMAP /* UNMAP */.
  - return SD_LBP_DISABLE.
- sd_config_discard: configures lim->max_discard_sectors / max_hw_discard_sectors / discard_granularity / discard_alignment from max_unmap_blocks + unmap_granularity + unmap_alignment.
- Quirks: invalid-opcode sense on UNMAP / WRITE_SAME unmap → sd_disable_discard(sdkp) sets provisioning_mode = SD_LBP_DISABLE.

REQ-17: WRITE_SAME / WRITE_ZEROES policy:
- sd_setup_write_zeroes_cmnd(cmd):
  - if !(rq->cmd_flags & REQ_NOUNMAP):
    - SD_ZERO_WS16_UNMAP → sd_setup_write_same16_cmnd(cmd, unmap=true).
    - SD_ZERO_WS10_UNMAP → sd_setup_write_same10_cmnd(cmd, unmap=true).
  - if sdp->no_write_same: rq_flags |= RQF_QUIET; return BLK_STS_TARGET.
  - if sdkp->ws16 ∨ lba > 0xffffffff ∨ nr_blocks > 0xffff: sd_setup_write_same16_cmnd(cmd, false).
  - else: sd_setup_write_same10_cmnd(cmd, false).

REQ-18: sd_open / sd_release:
- sd_open(disk, mode):
  - scsi_device_get(sdev) /* refcount +1 */.
  - scsi_block_when_processing_errors(sdev) /* wait until !EH-in-flight or fail */; on fail: -ENXIO.
  - if sd_need_revalidate(disk, sdkp): sd_revalidate_disk(disk).
  - if removable ∧ !media_present ∧ !(mode & BLK_OPEN_NDELAY): -ENOMEDIUM.
  - if write_prot ∧ (mode & BLK_OPEN_WRITE): -EROFS.
  - if !online: -ENXIO.
  - if atomic_inc_return(&sdkp->openers) == 1 ∧ removable: scsi_set_medium_removal(sdev, SCSI_REMOVAL_PREVENT) /* PREVENT_ALLOW_MEDIUM_REMOVAL CDB */.
- sd_release(disk):
  - if atomic_dec_return(&sdkp->openers) == 0 ∧ removable: scsi_set_medium_removal(sdev, SCSI_REMOVAL_ALLOW).
  - scsi_device_put(sdev).

REQ-19: Persistent Reservations (sd_pr_ops):
- sd_pr_register / _reserve / _release / _preempt / _clear / _read_keys / _read_reservation issue PERSISTENT RESERVE IN (0x5e) / OUT (0x5f) via scsi_execute_cmd.
- sd_scsi_to_pr_err(sshdr, result) maps SCSI sense → PR-API error codes.

REQ-20: Module init/exit (init_sd / exit_sd):
- init_sd:
  - for i ∈ 0..SD_MAJORS (= 16): __register_blkdev(sd_major(i), "sd", sd_default_probe) /* registers majors 8, 65..71, 128..135 */.
  - class_register(&sd_disk_class) /* /sys/class/scsi_disk/ */.
  - sd_page_pool = mempool_create_page_pool(SD_MEMPOOL_SIZE, 0) /* per-discard / write-same special-bvec backing */.
  - scsi_register_driver(&sd_template) /* registers sd_template as ULP */.
- exit_sd: scsi_unregister_driver; mempool_destroy; class_unregister; per-i unregister_blkdev.

REQ-21: sd_check_events (media-change polling):
- sd_check_events(disk, clearing):
  - TEST UNIT READY via scsi_execute_cmd.
  - if NOT_READY ∧ MEDIUM-NOT-PRESENT (asc 0x3a): set_media_not_present.
  - if UNIT_ATTENTION ∧ MEDIUM-MAY-HAVE-CHANGED (asc 0x28): return DISK_EVENT_MEDIA_CHANGE.

REQ-22: sd_sync_cache (issued at shutdown / suspend):
- timeout = rq_timeout * SD_FLUSH_TIMEOUT_MULTIPLIER.
- cmd[0] = sdp->use_16_for_sync ? SYNCHRONIZE_CACHE_16 : SYNCHRONIZE_CACHE.
- scsi_execute_cmd with failure_defs allowing 3 retries on SCMD_FAILURE_RESULT_ANY.
- On sense-key NOT_READY + asc 0x04 ascq 0x04 (format in progress) / ILLEGAL_REQUEST: return 0 (best-effort during shutdown).
- On asc 0x3a (no medium) / asc 0x20 (invalid command) / asc 0x74 ascq 0x71 (password-locked): return 0.
- On host_byte ∈ {DID_BAD_TARGET, DID_NO_CONNECT}: return 0; on {BUS_BUSY, IMM_RETRY, REQUEUE, SOFT_ERROR}: return -EBUSY; else -EIO.

REQ-23: sd_get_unique_id (blk-crypto + holder-checks):
- Walks VPD page 0x83 designators using scsi_vpd_lun_id; outputs 16-byte NAA/EUI64/T10-vendor id based on requested type ∈ {NAA, EUI64, UUID}.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `format_disk_name_total_for_buflen_DISK_NAME_LEN` | TOTAL | per-sd_format_disk_name(index ∈ [0, 26^4 + 26^3 + 26^2 + 26]): returns 0 with valid C-string ∈ ["sda".."sdzzzz"]. |
| `setup_rw_cdb_selection_disjoint` | INVARIANT | per-setup_read_write_cmnd: exactly one of rw32 / atomic / rw16 / rw10 / rw6 is called. |
| `setup_rw6_warns_zero_blocks` | INVARIANT | per-setup_rw6_cmnd: nr_blocks == 0 ⇒ WARN_ON_ONCE + return Iorr (avoids 6-byte 0 → 256 misinterpretation). |
| `unmap_param_list_24_bytes` | INVARIANT | per-setup_unmap_cmnd: payload length = 24; header.list_length = 22; descriptor.length = 16. |
| `done_sd_disable_discard_on_invalid_opcode` | INVARIANT | per-sd_done: ILLEGAL_REQUEST + asc ∈ {0x20, 0x24} + cmd[0] == UNMAP ⇒ provisioning_mode = SD_LBP_DISABLE. |
| `eh_action_offline_after_max_timeouts` | INVARIANT | per-sd_eh_action: medium_access_timed_out ≥ max_medium_access_timeouts ⇒ scsi_device_set_state(SDEV_OFFLINE). |
| `revalidate_capacity_set_after_read_capacity` | INVARIANT | per-sd_revalidate_disk: set_capacity_and_notify only called after sd_read_capacity returned. |
| `probe_index_freed_on_error` | INVARIANT | per-sd_probe: ida_alloc paired with ida_free on every error path. |
| `open_blocks_for_eh` | INVARIANT | per-sd_open: scsi_block_when_processing_errors gate before any revalidate / medium-prevent. |
| `release_decrements_openers` | INVARIANT | per-sd_release: atomic_dec_return == 0 ⇒ medium-removal allowed. |

### Layer 2: TLA+

`drivers/scsi/sd.tla`:
- Per-sdev probe → revalidate → device_add_disk → I/O traffic → suspend / shutdown → remove.
- Properties:
  - `safety_index_unique_per_disk` — per-sd_index_ida: every live sdkp.index is unique across all sd-attached sdevs.
  - `safety_capacity_monotone_within_lifecycle` — per-disk: after first sd_revalidate_disk, capacity changes only via media-change-induced revalidate, never by I/O completion path.
  - `safety_rw_cdb_selection_disjoint` — per-rq: exactly one CDB-family is chosen by setup_rw_cmnd.
  - `safety_open_blocks_during_eh` — per-sd_open: open-attempt while sdev in EH waits or fails ENXIO.
  - `safety_medium_access_timeout_offlines_disk` — per-disk: ≥ max_medium_access_timeouts consecutive medium-access-cmd timeouts ⇒ SDEV_OFFLINE.
  - `safety_discard_disable_on_invalid_opcode` — per-disk: INVALID-OPCODE on UNMAP/WRITE_SAME-unmap ⇒ provisioning_mode = SD_LBP_DISABLE; subsequent REQ_OP_DISCARD returns BLK_STS_TARGET.
  - `safety_no_io_below_capacity_check` — per-rq: blk_rq_pos + blk_rq_sectors > get_capacity ⇒ no CDB emitted, return IOERR.
  - `liveness_per_probe_eventually_completes` — per-sd_probe: either device_add_disk succeeds and gd is exposed, or every allocated resource (index, gd, disk_dev, sdkp) is rolled back.
  - `liveness_per_shutdown_drains_cache` — per-sd_shutdown: WCE ⇒ SYNCHRONIZE_CACHE issued (best-effort) before START_STOP UNIT.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sd::probe` post: success ⇒ gd exposed under /dev/sd<N> ∧ sdkp.index ∈ sd_index_ida ∧ /sys/class/scsi_disk/<sdev> populated | `Sd::probe` |
| `Sd::format_disk_name` total: prefix.len + len-of-base26-of-index + 1 ≤ buflen ⇒ Ok | `Sd::format_disk_name` |
| `Disk::setup_rw_cmnd` post: cmd.cmd_len ∈ {6, 10, 16, 32} ∧ cmd.cmnd encodes (lba, nr_blocks, flags, dld) per CDB-family layout | `Disk::setup_rw_cmnd` |
| `Disk::setup_unmap` post: cmd.cmd_len == 10 ∧ cmd.cmnd[0] == UNMAP ∧ buf.len == 24 ∧ buf encodes BE16(22)/BE16(16)/BE64(lba)/BE32(nr_blocks) | `Disk::setup_unmap` |
| `Disk::setup_write_same16` post: cmd.cmd_len == 16 ∧ cmd.cmnd[0] == WRITE_SAME_16 ∧ unmap-bit = (unmap ? 0x8 : 0) ∧ BE64(lba) ∧ BE32(nr_blocks) | `Disk::setup_write_same16` |
| `Disk::done` post: good_bytes ≤ scsi_bufflen(scmd) ∧ unaligned-resid is sector-rounded | `Disk::done` |
| `Disk::eh_action` post: returns SUCCESS only after SDEV_OFFLINE was set under state_mutex | `Disk::eh_action` |
| `Disk::revalidate` post: queue_limits_commit_update_frozen called exactly once per invocation | `Disk::revalidate` |
| `Disk::zbc_report_zones` post: on TYPE_ZBC returns ≤ nr_zones; on other types -EOPNOTSUPP | `Disk::zbc_report_zones` |
| `Disk::sync_cache` post: returns -EBUSY on transient host-byte; returns 0 on "no medium" / "format in progress" sense (shutdown path) | `Disk::sync_cache` |

### Layer 4: Verus/Creusot functional

`Per-rq REQ_OP_{READ,WRITE,DISCARD,WRITE_ZEROES,FLUSH,ZONE_*} → sd_init_command → CDB build (READ_6/10/16/32, WRITE_6/10/16/32, UNMAP, WRITE_SAME/_16, SYNCHRONIZE_CACHE/_16, WRITE_ATOMIC_16, ZBC_IN/OUT) → blk-mq dispatch → LLD → completion → sd_done → scsi_io_completion(good_bytes)` semantic equivalence: per-SCSI Block Commands SBC-4 (READ/WRITE/UNMAP/WRITE SAME), Zoned Block Commands ZBC-2 (REPORT ZONES, RESET WRITE POINTER, OPEN/CLOSE/FINISH ZONE), SCSI Primary Commands SPC-5 (PERSISTENT RESERVE IN/OUT, MODE SENSE, INQUIRY-VPD).

### hardening

(Inherits row-1 features from `drivers/scsi/00-overview.md` § Hardening and `drivers/scsi/scsi-lib.md` § Hardening.)

sd reinforcement:

- **Per-format_disk_name buflen-check** — defense against per-OOB-write on > 26^4-index pathological scenario.
- **Per-rq capacity-bound check before CDB emit** — defense against per-read-past-end giving garbage data (security-leak) or wrap-around behaviour.
- **Per-rq alignment check (blk_rq_pos & mask) (blk_rq_sectors & mask)** — defense against per-sub-sector CDB that some firmware mis-interprets.
- **Per-disk INVALID-OPCODE on UNMAP/WRITE_SAME disables feature** — defense against per-retry-storm on hardware that rejects optional commands.
- **Per-disk medium-access-timeout offline threshold (max_medium_access_timeouts)** — defense against per-zombie-disk that succeeds TUR but never completes I/O.
- **Per-EH-reset clears ignore_medium_access_errors gate** — defense against per-stale-EH-state across multiple EH runs.
- **Per-revalidate queue_limits_commit_update_frozen atomic update** — defense against per-partial-limit-update racing with in-flight I/O.
- **Per-revalidate zone-revalidate failure clears capacity to 0** — defense against per-stale-zone-map causing zone-pointer-violations on ZBC drives.
- **Per-sd_index_ida ida_free on every probe error path** — defense against per-sd-letter exhaustion under repeated probe/remove cycles.
- **Per-disk_dev.parent = get_device(sdev_dev)** — defense against per-sysfs UAF when sdev disappears while /sys/class/scsi_disk/ entry still open.
- **Per-removable scsi_set_medium_removal PREVENT on first open / ALLOW on last close** — defense against per-eject during open file descriptors.
- **Per-shutdown SYNCHRONIZE_CACHE only if WCE ∧ media_present** — defense against per-dirty-cache loss on power-off / suspend.
- **Per-sd_done unaligned-resid rounded up** — defense against per-bogus-firmware reporting sub-sector residual.
- **Per-sd_zbc_report_zones bounded by queue_max_hw_sectors / queue_max_segments / BIO_MAX_INLINE_VECS** — defense against per-OOM on extremely large nr_zones request.
- **Per-PERSISTENT-RESERVE-IN/OUT routed through scsi_execute_cmd with sshdr** — defense against per-PR-error-misinterpretation.
- **Per-Opal init_opal_dev only on sdkp->security from VPD-0x86** — defense against per-spurious-TCG-init on non-security devices.

