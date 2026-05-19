# Tier-3: drivers/scsi/sd_zbc.c — SCSI ZBC zoned block command support

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/scsi/00-overview.md
upstream-paths:
  - drivers/scsi/sd_zbc.c (~641 lines)
  - drivers/scsi/sd.h (struct scsi_disk zoned-related fields, struct zoned_disk_info)
  - drivers/scsi/sd_trace.h
  - include/scsi/scsi_proto.h (ZBC_IN / ZBC_OUT / ZI_REPORT_ZONES / ZBC_ZONE_TYPE_GAP / ZBC_CONSTANT_ZONE_LENGTH / ZBC_CONSTANT_ZONE_START_OFFSET / ZBC_ZONE_COND_FULL / ZBC_REPORT_ZONE_PARTIAL)
  - include/linux/blkzoned.h (struct blk_zone, struct blk_report_zones_args)
  - block/blk-zoned.c (blk_revalidate_disk_zones, blkdev_report_zones, disk_report_zone, op_is_zone_mgmt)
-->

## Summary

The **SCSI ZBC sidecar** of the `sd` ULP — implements Zoned Block Commands (ZBC and ZAC) for host-managed (`TYPE_ZBC`) and host-aware (`TYPE_DISK` with VPD page B6) zoned storage. Owns: per-disk zone discovery (`sd_zbc_read_zones` → VPD page 0xB6 + REPORT_ZONES at LBA 0 → derive zone_blocks + nr_zones + zone alignment method), per-rq zone-management CDB build (`sd_zbc_setup_zone_mgmt_cmnd` for `REQ_OP_ZONE_RESET` / `_RESET_ALL` / `_OPEN` / `_CLOSE` / `_FINISH` emitted as `ZBC_OUT` 16-byte CDBs with op = `ZO_RESET_WRITE_POINTER` / `ZO_OPEN_ZONE` / `ZO_CLOSE_ZONE` / `ZO_FINISH_ZONE`), per-disk REPORT ZONES iterator (`sd_zbc_report_zones` callback for `blkdev_report_zones` — issues partial REPORT_ZONES via `ZBC_IN`/`ZI_REPORT_ZONES`, parses 64-byte zone descriptors into `struct blk_zone`, invokes `disk_report_zone` callback), per-cmd completion ILLEGAL_REQUEST/ASC=0x24 quieting (`sd_zbc_complete` — silences "invalid field in CDB" on conv-zone zone-mgmt attempts), per-disk capacity check via REPORT_ZONES max_lba reconciliation, per-disk zone-alignment-method selection (`ZBC_CONSTANT_ZONE_LENGTH` vs `ZBC_CONSTANT_ZONE_START_OFFSET` granularity), per-disk revalidate trigger via `blk_revalidate_disk_zones`. Critical for: SMR drives, ZNS-emulating SCSI translations, host-managed enterprise zoned storage.

This Tier-3 covers `drivers/scsi/sd_zbc.c` (~641 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `sd_zbc_is_gap_zone()` | per-descriptor gap-zone detect | `SdZbc::is_gap_zone` |
| `sd_zbc_parse_report()` | per-descriptor → blk_zone | `SdZbc::parse_report` |
| `sd_zbc_do_report_zones()` | per-REPORT_ZONES SCSI exec | `SdZbc::do_report_zones` |
| `sd_zbc_alloc_report_buffer()` | per-bufsize-fit alloc | `SdZbc::alloc_report_buffer` |
| `sd_zbc_zone_sectors()` | per-disk zone-size sectors | `SdZbc::zone_sectors` |
| `sd_zbc_report_zones()` | per-disk .report_zones cb | `SdZbc::report_zones` |
| `sd_zbc_cmnd_checks()` | per-rq pre-issue sanity | `SdZbc::cmnd_checks` |
| `sd_zbc_setup_zone_mgmt_cmnd()` | per-rq ZBC_OUT CDB build | `SdZbc::setup_zone_mgmt_cmnd` |
| `sd_zbc_complete()` | per-cmd quiet conv-zone errors | `SdZbc::complete` |
| `sd_zbc_check_zoned_characteristics()` | per-disk VPD 0xB6 parse | `SdZbc::check_zoned_characteristics` |
| `sd_zbc_check_capacity()` | per-disk capacity reconcile | `SdZbc::check_capacity` |
| `sd_zbc_print_zones()` | per-disk klog summary | `SdZbc::print_zones` |
| `sd_zbc_revalidate_zones()` | per-disk re-emit blk-zoned | `SdZbc::revalidate_zones` |
| `sd_zbc_read_zones()` | per-disk init-time discover | `SdZbc::read_zones` |
| `struct zoned_disk_info` | per-disk zone_blocks + nr_zones | `ZonedDiskInfo` |
| `ZBC_IN` / `ZBC_OUT` | SCSI opcodes (0x95 / 0x94) | const |
| `ZI_REPORT_ZONES` | sub-opcode (0x00) | const |
| `ZBC_REPORT_ZONE_PARTIAL` | partial flag | const |
| `ZBC_ZONE_TYPE_GAP` | descriptor zone-type nibble | const |
| `ZBC_ZONE_COND_FULL` | descriptor zone-cond | const |
| `ZBC_CONSTANT_ZONE_LENGTH` | VPD alignment method | const |
| `ZBC_CONSTANT_ZONE_START_OFFSET` | VPD alignment method | const |

## Compatibility contract

REQ-1: SCSI zone descriptor layout (64-byte block returned by REPORT_ZONES):
- buf[0] low nibble = zone type (CONV / SEQ_REQ / SEQ_PREF / GAP).
- buf[1] bits [7:4] = zone condition (NOT_WP / EMPTY / IMP_OPEN / EXP_OPEN / CLOSED / READONLY / FULL / OFFLINE).
- buf[1] bit 0 = RESET-recommended flag (host-aware only).
- buf[1] bit 1 = NON_SEQ flag (host-aware only).
- buf[8..16] big-endian u64 = zone capacity (logical blocks).
- buf[16..24] big-endian u64 = zone start LBA (logical blocks).
- buf[24..32] big-endian u64 = write-pointer LBA (logical blocks).

REQ-2: REPORT_ZONES CDB (ZBC_IN sub-cmd ZI_REPORT_ZONES, 16-byte CDB):
- cmd[0] = ZBC_IN (0x95).
- cmd[1] = ZI_REPORT_ZONES (0x00).
- cmd[2..10] = start LBA (big-endian u64).
- cmd[10..14] = allocation length (big-endian u32).
- cmd[14] = reporting-options byte; bit-7 (`ZBC_REPORT_ZONE_PARTIAL`) requests partial reporting.
- timeout = `sdp->request_queue->rq_timeout`, retries = `SD_MAX_RETRIES`.
- Reply header: 64-byte preamble with be32 list-length at offset 0; subsequent 64-byte descriptors.

REQ-3: Zone-management CDB (ZBC_OUT, 16-byte CDB):
- cmd[0] = ZBC_OUT (0x94).
- cmd[1] = op ∈ { RESET_WRITE_POINTER, OPEN_ZONE, CLOSE_ZONE, FINISH_ZONE }.
- cmd[2..10] = zone start LBA (big-endian u64); ignored when "all" is true.
- cmd[14] bit-0 = ALL flag (set for `REQ_OP_ZONE_RESET_ALL` / open-all / close-all / finish-all).
- cmd->cmd_len = 16; sc_data_direction = DMA_NONE; transfersize = 0; allowed = 0.
- timeout = `SD_TIMEOUT`.

REQ-4: VPD page 0xB6 ("Zoned block device characteristics") parse:
- buf[4] bit 0 = URSWRZ (unrestricted reads in sequential write-required zones).
- buf[8..12] be32 = optimal-open-zones (host-aware only).
- buf[12..16] be32 = optimal-non-seq zones (host-aware only).
- buf[16..20] be32 = max-open-zones (host-managed only).
- buf[23] low nibble = zone alignment method.
- buf[24..32] be64 = zone-starting-LBA granularity (only when alignment method == ZBC_CONSTANT_ZONE_START_OFFSET).

REQ-5: sd_zbc_is_gap_zone(buf) → (buf[0] & 0xf) == ZBC_ZONE_TYPE_GAP.

REQ-6: sd_zbc_parse_report(sdkp, buf, idx, args):
- If sd_zbc_is_gap_zone(buf): WARN_ON_ONCE + return -EINVAL.
- zone.type = buf[0] & 0x0f.
- zone.cond = (buf[1] >> 4) & 0xf.
- zone.reset = (buf[1] & 0x01) ? 1 : 0.
- zone.non_seq = (buf[1] & 0x02) ? 1 : 0.
- start_lba = get_unaligned_be64(&buf[16]).
- zone.start = logical_to_sectors(sdp, start_lba).
- zone.capacity = logical_to_sectors(sdp, get_unaligned_be64(&buf[8])).
- zone.len = zone.capacity (default).
- If sdkp->zone_starting_lba_gran != 0:
  - gran = logical_to_sectors(sdp, sdkp->zone_starting_lba_gran).
  - If zone.len > gran: KERN_ERR + -EINVAL (invalid zone).
  - zone.len = gran (override length with granularity).
- If zone.cond == ZBC_ZONE_COND_FULL: zone.wp = zone.start + zone.len.
- Else: zone.wp = logical_to_sectors(sdp, get_unaligned_be64(&buf[24])).
- Return disk_report_zone(sdkp->disk, &zone, idx, args).

REQ-7: sd_zbc_do_report_zones(sdkp, buf, buflen, lba, partial):
- /* Build 16-byte ZBC_IN/ZI_REPORT_ZONES CDB */ — see REQ-2.
- scsi_execute_cmd(sdp, cmd, REQ_OP_DRV_IN, buf, buflen, timeout, SD_MAX_RETRIES, &exec_args).
- On error: KERN_ERR + sd_print_result + sd_print_sense_hdr (if scsi_sense_valid); return -EIO.
- rep_len = get_unaligned_be32(&buf[0]).
- If rep_len < 64: KERN_ERR ("REPORT ZONES report invalid length") + return -EIO.
- Return 0.

REQ-8: sd_zbc_alloc_report_buffer(sdkp, nr_zones, *buflen):
- nr_zones = min(nr_zones, sdkp->zone_info.nr_zones).
- bufsize = roundup((nr_zones + 1) * 64, SECTOR_SIZE) /* +1 for reply header */.
- bufsize = min(bufsize, queue_max_hw_sectors(q) << SECTOR_SHIFT).
- max_segments = min(BIO_MAX_INLINE_VECS, queue_max_segments(q)).
- bufsize = min(bufsize, max_segments << PAGE_SHIFT).
- Loop: while bufsize >= SECTOR_SIZE:
  - buf = kvzalloc(bufsize, GFP_KERNEL | __GFP_NORETRY).
  - On success: *buflen = bufsize; return buf.
  - Else: bufsize = rounddown(bufsize >> 1, SECTOR_SIZE).
- Return NULL.

REQ-9: sd_zbc_zone_sectors(sdkp) = logical_to_sectors(sdkp->device, sdkp->zone_info.zone_blocks).

REQ-10: sd_zbc_report_zones(disk, sector, nr_zones, args) — block-layer `.report_zones` callback:
- sdkp = scsi_disk(disk); lba = sectors_to_logical(sdkp->device, sector).
- If sdkp->device->type != TYPE_ZBC: return -EOPNOTSUPP.
- If !sdkp->capacity: return -ENODEV.
- buf = sd_zbc_alloc_report_buffer(sdkp, nr_zones, &buflen); if !buf: return -ENOMEM.
- zone_idx = 0.
- While zone_idx < nr_zones ∧ lba < sdkp->capacity:
  - ret = sd_zbc_do_report_zones(sdkp, buf, buflen, lba, true /* partial */); if ret: goto out.
  - offset = 0.
  - nr = min(nr_zones, get_unaligned_be32(&buf[0]) / 64) /* descriptor count in this reply */.
  - If !nr: break.
  - For i in 0..nr where zone_idx < nr_zones:
    - offset += 64.
    - start_lba = get_unaligned_be64(&buf[offset + 16]).
    - zone_length = get_unaligned_be64(&buf[offset + 8]).
    - /* Sanity: first-iter LBA must lie within first zone; subsequent zones must abut prior */
    - If (zone_idx == 0 ∧ (lba < start_lba ∨ lba ≥ start_lba + zone_length))
       ∨ (zone_idx > 0 ∧ start_lba != lba)
       ∨ start_lba + zone_length < start_lba:
      - KERN_ERR("Zone %d at LBA %llu is invalid"); ret = -EINVAL; goto out.
    - lba = start_lba + zone_length.
    - If sd_zbc_is_gap_zone(&buf[offset]):
      - If sdkp->zone_starting_lba_gran: continue (gap zones legal under constant-offset alignment).
      - Else: KERN_ERR("Gap zone without constant LBA offsets"); ret = -EINVAL; goto out.
    - ret = sd_zbc_parse_report(sdkp, buf + offset, zone_idx, args); if ret: goto out.
    - zone_idx += 1.
- ret = zone_idx (number of zones successfully reported).
- out: kvfree(buf); return ret.

REQ-11: sd_zbc_cmnd_checks(cmd):
- rq = scsi_cmd_to_rq(cmd); sdkp = scsi_disk(rq->q->disk); sector = blk_rq_pos(rq).
- If sdkp->device->type != TYPE_ZBC: return BLK_STS_IOERR.
- If sdkp->device->changed: return BLK_STS_IOERR.
- If sector & (sd_zbc_zone_sectors(sdkp) - 1) /* unaligned to zone boundary */: return BLK_STS_IOERR.
- Return BLK_STS_OK.

REQ-12: sd_zbc_setup_zone_mgmt_cmnd(cmd, op, all):
- ret = sd_zbc_cmnd_checks(cmd); if != BLK_STS_OK: return ret.
- cmd->cmd_len = 16.
- memset(cmd->cmnd, 0, 16).
- cmd->cmnd[0] = ZBC_OUT.
- cmd->cmnd[1] = op.
- If all: cmd->cmnd[14] = 0x1.
- Else: put_unaligned_be64(block /* sectors_to_logical(sdkp->device, sector) */, &cmd->cmnd[2]).
- rq->timeout = SD_TIMEOUT.
- cmd->sc_data_direction = DMA_NONE.
- cmd->transfersize = 0.
- cmd->allowed = 0.
- Return BLK_STS_OK.

REQ-13: sd_zbc_complete(cmd, good_bytes, sshdr):
- rq = scsi_cmd_to_rq(cmd).
- If op_is_zone_mgmt(req_op(rq)) ∧ cmd->result ∧ sshdr->sense_key == ILLEGAL_REQUEST ∧ sshdr->asc == 0x24:
  - /* "INVALID FIELD IN CDB" against a conventional zone — expected; quiet it */
  - rq->rq_flags |= RQF_QUIET.
- Return good_bytes (unchanged).

REQ-14: sd_zbc_check_zoned_characteristics(sdkp, buf):
- If scsi_get_vpd_page(sdp, 0xb6, buf, 64): KERN_NOTICE + return -ENODEV.
- If sdkp->device->type != TYPE_ZBC /* host-aware path */:
  - sdkp->urswrz = 1.
  - sdkp->zones_optimal_open = get_unaligned_be32(&buf[8]).
  - sdkp->zones_optimal_nonseq = get_unaligned_be32(&buf[12]).
  - sdkp->zones_max_open = 0.
  - return 0.
- /* host-managed path */
- sdkp->urswrz = buf[4] & 1.
- sdkp->zones_optimal_open = 0; sdkp->zones_optimal_nonseq = 0.
- sdkp->zones_max_open = get_unaligned_be32(&buf[16]).
- /* Zone alignment method (buf[23] & 0xf): */
- case 0 | ZBC_CONSTANT_ZONE_LENGTH: use zone length from REPORT_ZONES.
- case ZBC_CONSTANT_ZONE_START_OFFSET:
  - zone_starting_lba_gran = get_unaligned_be64(&buf[24]).
  - If gran == 0 ∨ !is_power_of_2(gran) ∨ logical_to_sectors(sdp, gran) > UINT_MAX: KERN_ERR + -ENODEV.
  - sdkp->zone_starting_lba_gran = gran.
- default: KERN_ERR + -ENODEV.
- /* Constrained-read host-managed devices unsupported */
- If !sdkp->urswrz: KERN_NOTICE (on first_scan) + return -ENODEV.
- Return 0.

REQ-15: sd_zbc_check_capacity(sdkp, buf, *zblocks):
- ret = sd_zbc_do_report_zones(sdkp, buf, SD_BUF_SIZE, 0, false /* full */); if ret: return ret.
- If sdkp->rc_basis == 0 /* READ CAPACITY reports max LBA */:
  - max_lba = get_unaligned_be64(&buf[8]).
  - If sdkp->capacity != max_lba + 1: KERN_WARNING (first_scan) + sdkp->capacity = max_lba + 1.
- If sdkp->zone_starting_lba_gran == 0:
  - /* First reported zone descriptor at &buf[64]; capacity at &rec[8] */
  - zone_blocks = get_unaligned_be64(&buf[64 + 8]).
  - If logical_to_sectors(sdp, zone_blocks) > UINT_MAX: KERN_NOTICE (first_scan) + -EFBIG.
- Else: zone_blocks = sdkp->zone_starting_lba_gran.
- If !is_power_of_2(zone_blocks): KERN_ERR + -EINVAL.
- *zblocks = zone_blocks; return 0.

REQ-16: sd_zbc_revalidate_zones(sdkp):
- q = sdkp->disk->queue.
- zone_blocks = sdkp->early_zone_info.zone_blocks; nr_zones = sdkp->early_zone_info.nr_zones.
- If !blk_queue_is_zoned(q): return 0 /* regular disk including host-aware-with-partitions */.
- If sdkp->zone_info.zone_blocks == zone_blocks ∧ sdkp->zone_info.nr_zones == nr_zones ∧ disk->nr_zones == nr_zones: return 0 (no-op).
- sdkp->zone_info.zone_blocks = zone_blocks; sdkp->zone_info.nr_zones = nr_zones.
- flags = memalloc_noio_save(); ret = blk_revalidate_disk_zones(disk); memalloc_noio_restore(flags).
- If ret: sdkp->zone_info = {}; sdkp->capacity = 0; return ret.
- sd_zbc_print_zones(sdkp); return 0.

REQ-17: sd_zbc_read_zones(sdkp, lim, buf):
- If sdkp->device->type != TYPE_ZBC: return 0 /* host-aware drives fall through later */.
- lim->features |= BLK_FEAT_ZONED.
- lim->zone_write_granularity = sdkp->physical_block_size /* ZBC/ZAC mandate phys-block-size aligned writes in seq-req zones */.
- sdkp->device->use_16_for_rw = 1; sdkp->device->use_10_for_rw = 0; sdkp->device->use_16_for_sync = 1 /* READ16/WRITE16/SYNC16 mandatory */.
- ret = sd_zbc_check_zoned_characteristics(sdkp, buf); if ret: goto err.
- ret = sd_zbc_check_capacity(sdkp, buf, &zone_blocks); if ret: goto err.
- nr_zones = round_up(sdkp->capacity, zone_blocks) >> ilog2(zone_blocks).
- sdkp->early_zone_info.nr_zones = nr_zones; sdkp->early_zone_info.zone_blocks = zone_blocks.
- If sdkp->zones_max_open == U32_MAX: lim->max_open_zones = 0; else lim->max_open_zones = sdkp->zones_max_open.
- lim->max_active_zones = 0.
- lim->chunk_sectors = logical_to_sectors(sdp, zone_blocks).
- return 0.
- err: sdkp->capacity = 0; return ret.

REQ-18: sd_zbc_print_zones(sdkp) — logs `"%u zones of %u logical blocks"` or `"%u zones of %u logical blocks + 1 runt zone"` (when capacity is not a multiple of zone_blocks). Skips quietly if device is not TYPE_ZBC or capacity == 0.

## Acceptance Criteria

- [ ] AC-1: TYPE_ZBC device probe: `sd_zbc_read_zones` sets `BLK_FEAT_ZONED`, forces 16-byte RW/SYNC, and populates `early_zone_info.{zone_blocks,nr_zones}`.
- [ ] AC-2: Non-TYPE_ZBC device: `sd_zbc_read_zones` returns 0 without modifying limits (host-aware drives surface zone metadata through VPD only).
- [ ] AC-3: VPD page 0xB6 fetch failure (`scsi_get_vpd_page` returns non-zero): `sd_zbc_check_zoned_characteristics` returns -ENODEV; outer `sd_zbc_read_zones` zeroes capacity.
- [ ] AC-4: Host-managed device with `URSWRZ == 0`: `sd_zbc_check_zoned_characteristics` returns -ENODEV (constrained-read devices rejected).
- [ ] AC-5: VPD alignment method = `ZBC_CONSTANT_ZONE_START_OFFSET` with non-power-of-2 granularity or granularity overflowing UINT_MAX sectors: -ENODEV.
- [ ] AC-6: `sd_zbc_check_capacity` with `rc_basis == 0` and mismatched `READ CAPACITY` vs REPORT_ZONES max_lba: capacity is overridden to `max_lba + 1` (with `first_scan` warning).
- [ ] AC-7: REPORT_ZONES reply with `rep_len < 64`: `sd_zbc_do_report_zones` returns -EIO.
- [ ] AC-8: `sd_zbc_report_zones` invoked on non-TYPE_ZBC disk: returns -EOPNOTSUPP.
- [ ] AC-9: `sd_zbc_report_zones` invoked on disk with capacity == 0: returns -ENODEV.
- [ ] AC-10: Gap-zone descriptor (`buf[0] & 0xf == ZBC_ZONE_TYPE_GAP`) without `zone_starting_lba_gran` set: iteration aborts with -EINVAL.
- [ ] AC-11: Zone descriptor with `start_lba + zone_length` overflow or non-abutting start: iteration aborts with -EINVAL.
- [ ] AC-12: `sd_zbc_setup_zone_mgmt_cmnd(REQ_OP_ZONE_RESET_ALL)`: CDB has `cmnd[14] == 0x1` and `cmnd[2..10] == 0`.
- [ ] AC-13: `sd_zbc_setup_zone_mgmt_cmnd` with rq sector misaligned to zone-size: returns BLK_STS_IOERR (via `sd_zbc_cmnd_checks`).
- [ ] AC-14: `sd_zbc_complete` on zone-mgmt op with sense ILLEGAL_REQUEST/ASC=0x24: sets `RQF_QUIET` and returns `good_bytes` unchanged.
- [ ] AC-15: `sd_zbc_revalidate_zones` no-op when zone_info matches early_zone_info and disk->nr_zones agrees.

## Architecture

```
struct ZonedDiskInfo {
  nr_zones: u32,
  zone_blocks: u32,
}

struct ScsiDiskZoned {                 /* fields embedded in ScsiDisk */
  urswrz: bool,                        /* unrestricted reads in SWR zones */
  zones_optimal_open: u32,             /* host-aware only */
  zones_optimal_nonseq: u32,           /* host-aware only */
  zones_max_open: u32,                 /* host-managed only */
  zone_starting_lba_gran: u64,         /* 0 = use REPORT_ZONES length */
  zone_info: ZonedDiskInfo,            /* live state */
  early_zone_info: ZonedDiskInfo,      /* discovered at read_zones */
}

const ZBC_IN: u8 = 0x95;
const ZBC_OUT: u8 = 0x94;
const ZI_REPORT_ZONES: u8 = 0x00;
const ZBC_REPORT_ZONE_PARTIAL: u8 = 0x80;
const ZBC_ZONE_TYPE_GAP: u8 = 0x05;
const ZBC_ZONE_COND_FULL: u8 = 0xE;
const ZBC_CONSTANT_ZONE_LENGTH: u8 = 0x01;
const ZBC_CONSTANT_ZONE_START_OFFSET: u8 = 0x08;
```

`SdZbc::read_zones(sdkp, lim, buf) -> Result<()>`:
1. If sdkp.device.type != TYPE_ZBC: return Ok(()).
2. lim.features |= BLK_FEAT_ZONED.
3. lim.zone_write_granularity = sdkp.physical_block_size.
4. /* Force 16-byte RW/SYNC */
5. sdkp.device.use_16_for_rw = true; use_10_for_rw = false; use_16_for_sync = true.
6. SdZbc::check_zoned_characteristics(sdkp, buf)? /* VPD 0xB6 */
7. SdZbc::check_capacity(sdkp, buf, &mut zone_blocks)? /* REPORT_ZONES at LBA 0 */
8. nr_zones = round_up(sdkp.capacity, zone_blocks) >> ilog2(zone_blocks).
9. sdkp.early_zone_info = ZonedDiskInfo { nr_zones, zone_blocks }.
10. lim.max_open_zones = if sdkp.zones_max_open == U32_MAX { 0 } else { sdkp.zones_max_open }.
11. lim.max_active_zones = 0; lim.chunk_sectors = logical_to_sectors(sdp, zone_blocks).
12. On error path: sdkp.capacity = 0; return Err.

`SdZbc::report_zones(disk, sector, nr_zones, args) -> Result<i32>`:
1. sdkp = scsi_disk(disk); lba = sectors_to_logical(sdkp.device, sector).
2. If sdkp.device.type != TYPE_ZBC: return Err(EOPNOTSUPP).
3. If sdkp.capacity == 0: return Err(ENODEV).
4. buf = SdZbc::alloc_report_buffer(sdkp, nr_zones, &mut buflen).ok_or(ENOMEM)?
5. zone_idx = 0.
6. While zone_idx < nr_zones ∧ lba < sdkp.capacity:
   a. SdZbc::do_report_zones(sdkp, buf, buflen, lba, true)?
   b. nr = min(nr_zones, get_unaligned_be32(&buf[0]) / 64).
   c. If nr == 0: break.
   d. For each descriptor i in 0..nr while zone_idx < nr_zones:
      - offset += 64.
      - Validate abutting / non-overflowing zone arithmetic.
      - lba = start_lba + zone_length.
      - If gap zone: skip (when constant-offset) or fail (-EINVAL).
      - SdZbc::parse_report(sdkp, buf + offset, zone_idx, args)?
      - zone_idx += 1.
7. kvfree(buf); return Ok(zone_idx).

`SdZbc::parse_report(sdkp, buf, idx, args) -> Result<i32>`:
1. /* WARN+EINVAL on gap zone (caller should have filtered) */
2. zone.type = buf[0] & 0x0f; zone.cond = (buf[1] >> 4) & 0xf.
3. zone.reset = (buf[1] & 0x01) != 0; zone.non_seq = (buf[1] & 0x02) != 0.
4. start_lba = get_unaligned_be64(&buf[16]).
5. zone.start = logical_to_sectors(sdp, start_lba).
6. zone.capacity = logical_to_sectors(sdp, get_unaligned_be64(&buf[8])).
7. zone.len = zone.capacity.
8. If sdkp.zone_starting_lba_gran != 0:
   - gran = logical_to_sectors(sdp, sdkp.zone_starting_lba_gran).
   - If zone.len > gran: return Err(EINVAL).
   - zone.len = gran.
9. If zone.cond == ZBC_ZONE_COND_FULL: zone.wp = zone.start + zone.len.
10. Else: zone.wp = logical_to_sectors(sdp, get_unaligned_be64(&buf[24])).
11. Return disk_report_zone(sdkp.disk, &zone, idx, args).

`SdZbc::setup_zone_mgmt_cmnd(cmd, op, all) -> BlkStatus`:
1. SdZbc::cmnd_checks(cmd)? /* TYPE_ZBC, !changed, zone-aligned sector */
2. cmd.cmd_len = 16; memset(cmd.cmnd, 0, 16).
3. cmd.cmnd[0] = ZBC_OUT.
4. cmd.cmnd[1] = op /* RESET_WP / OPEN / CLOSE / FINISH */.
5. If all: cmd.cmnd[14] = 0x1.
6. Else: put_unaligned_be64(sectors_to_logical(sdp, blk_rq_pos(rq)), &cmd.cmnd[2]).
7. rq.timeout = SD_TIMEOUT.
8. cmd.sc_data_direction = DMA_NONE; cmd.transfersize = 0; cmd.allowed = 0.
9. Return BLK_STS_OK.

`SdZbc::do_report_zones(sdkp, buf, buflen, lba, partial) -> Result<()>`:
1. cmd[0] = ZBC_IN; cmd[1] = ZI_REPORT_ZONES.
2. put_unaligned_be64(lba, &cmd[2]); put_unaligned_be32(buflen, &cmd[10]).
3. If partial: cmd[14] = ZBC_REPORT_ZONE_PARTIAL.
4. scsi_execute_cmd(sdp, cmd, REQ_OP_DRV_IN, buf, buflen, timeout, SD_MAX_RETRIES, exec_args)? /* else -EIO with sense print */
5. rep_len = get_unaligned_be32(&buf[0]); if rep_len < 64: return Err(EIO).
6. Return Ok(()).

`SdZbc::complete(cmd, good_bytes, sshdr) -> u32`:
1. If op_is_zone_mgmt(req_op(rq)) ∧ cmd.result != 0 ∧ sshdr.sense_key == ILLEGAL_REQUEST ∧ sshdr.asc == 0x24:
   - rq.rq_flags |= RQF_QUIET /* conv-zone zone-mgmt attempt is harmless */.
2. Return good_bytes (unchanged).

`SdZbc::revalidate_zones(sdkp) -> Result<()>`:
1. q = sdkp.disk.queue.
2. zone_blocks = sdkp.early_zone_info.zone_blocks; nr_zones = sdkp.early_zone_info.nr_zones.
3. If !blk_queue_is_zoned(q): return Ok(()).
4. If sdkp.zone_info.zone_blocks == zone_blocks ∧ sdkp.zone_info.nr_zones == nr_zones ∧ disk.nr_zones == nr_zones: return Ok(()).
5. sdkp.zone_info.zone_blocks = zone_blocks; sdkp.zone_info.nr_zones = nr_zones.
6. flags = memalloc_noio_save().
7. ret = blk_revalidate_disk_zones(disk).
8. memalloc_noio_restore(flags).
9. On error: sdkp.zone_info = {}; sdkp.capacity = 0; propagate.
10. SdZbc::print_zones(sdkp).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `report_zones_alloc_bounded` | INVARIANT | per-alloc_report_buffer: bufsize is bounded by `queue_max_hw_sectors << SECTOR_SHIFT` and `BIO_MAX_INLINE_VECS << PAGE_SHIFT`. |
| `descriptor_offset_in_bounds` | INVARIANT | per-report_zones loop: `offset + 64 <= buflen` for every dereference of `buf[offset + N]`. |
| `gap_zone_filtered` | INVARIANT | per-parse_report: callable only when `is_gap_zone(buf) == false`; WARN+EINVAL otherwise. |
| `zone_arith_no_overflow` | INVARIANT | per-report_zones: `start_lba + zone_length` checked for overflow before use. |
| `zone_size_power_of_two` | INVARIANT | per-check_capacity: `is_power_of_2(zone_blocks)` enforced. |
| `mgmt_cdb_zone_aligned` | INVARIANT | per-setup_zone_mgmt_cmnd: sector is zone-aligned (mod sd_zbc_zone_sectors). |
| `read_zones_only_on_TYPE_ZBC` | INVARIANT | per-read_zones: BLK_FEAT_ZONED set only when `device.type == TYPE_ZBC`. |
| `urswrz_required_for_host_managed` | INVARIANT | per-check_zoned_characteristics: host-managed with !urswrz → -ENODEV. |

### Layer 2: TLA+

`drivers/scsi/sd-zbc.tla`:
- Per-probe (read_zones) → per-revalidate (revalidate_zones) → per-iterate (report_zones over all zones) → per-mgmt (setup_zone_mgmt_cmnd) → per-complete (complete).
- Properties:
  - `safety_zone_iteration_terminates` — per-report_zones loop terminates with `lba >= sdkp.capacity` or `zone_idx >= nr_zones`.
  - `safety_no_descriptor_oob` — per-report_zones: descriptor offsets stay within `buflen`.
  - `safety_zone_mgmt_cdb_well_formed` — per-setup_zone_mgmt_cmnd: ZBC_OUT CDB has exactly 16 bytes with op in {RESET_WP, OPEN, CLOSE, FINISH}.
  - `safety_host_managed_implies_urswrz` — per-host-managed disk that completes read_zones has `urswrz == 1`.
  - `liveness_per_report_zones_progresses_or_stops` — per-do_report_zones: returns descriptors or a definite error each call.
  - `safety_revalidate_idempotent` — per-revalidate_zones: re-invocation with unchanged early_zone_info is a no-op.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `SdZbc::read_zones` post: BLK_FEAT_ZONED ⟺ TYPE_ZBC | `SdZbc::read_zones` |
| `SdZbc::check_zoned_characteristics` post: host-managed ⟹ urswrz == 1 (else -ENODEV) | `SdZbc::check_zoned_characteristics` |
| `SdZbc::check_capacity` post: `is_power_of_2(*zblocks)` | `SdZbc::check_capacity` |
| `SdZbc::report_zones` post: returned count ≤ nr_zones | `SdZbc::report_zones` |
| `SdZbc::parse_report` post: `zone.wp ∈ [zone.start, zone.start + zone.len]` | `SdZbc::parse_report` |
| `SdZbc::do_report_zones` post: `get_unaligned_be32(&buf[0]) ≥ 64` ∨ Err(EIO) | `SdZbc::do_report_zones` |
| `SdZbc::setup_zone_mgmt_cmnd` post: `cmnd[0] == ZBC_OUT ∧ cmnd[1] == op` | `SdZbc::setup_zone_mgmt_cmnd` |
| `SdZbc::complete` post: `op_is_zone_mgmt ∧ ASC==0x24 ⟹ RQF_QUIET` set | `SdZbc::complete` |

### Layer 4: Verus/Creusot functional

`Per-disk discover → VPD 0xB6 parse → REPORT_ZONES at LBA 0 → capacity reconcile → zone_blocks = pow2 → emit chunk_sectors + max_open_zones → blk_revalidate_disk_zones` semantic equivalence: per ZBC/ZBC-2 (T10 r05c) and ZAC-2 (T13) specs and Documentation/block/zoned.rst.

## Hardening

(Inherits row-1 features from `drivers/scsi/00-overview.md` § Hardening.)

ZBC-sidecar reinforcement:

- **Per-REPORT_ZONES rep_len ≥ 64 check** — defense against per-truncated reply oob-read.
- **Per-descriptor offset stays ≤ buflen − 64** — defense against per-malicious-target oob-read across the 64-byte descriptor.
- **Per-zone start_lba + zone_length overflow check** — defense against per-arith-overflow leading to infinite report loop.
- **Per-zone abutment check (start_lba == lba on iter > 0)** — defense against per-target lying about layout, infinite loop avoidance.
- **Per-gap-zone refusal unless `zone_starting_lba_gran` set** — defense against per-malformed report under length-aligned layout.
- **Per-host-managed `urswrz == 1` requirement** — defense against per-constrained-read drive surprising the block layer with EIO after wp.
- **Per-`zone_blocks` power-of-two enforcement** — defense against per-non-power-of-two divisor in fast-path `>> ilog2(zone_blocks)`.
- **Per-`zone_starting_lba_gran` ≤ UINT_MAX sectors** — defense against per-sector-count overflow.
- **Per-zone-aligned rq sector check (sd_zbc_cmnd_checks)** — defense against per-mid-zone zone-mgmt command.
- **Per-DMA_NONE / transfersize=0 on zone-mgmt CDB** — defense against per-DMA-misuse with no data buffer.
- **Per-RQF_QUIET on ASC=0x24 conv-zone reject** — defense against per-dmesg flood when fs targets a conventional zone.
- **Per-`memalloc_noio_save` around blk_revalidate_disk_zones** — defense against per-revalidate-allocates-IO recursion on a stalling disk.
- **Per-`__GFP_NORETRY` + `kvzalloc` with halving fallback** — defense against per-OOM during report-buffer alloc.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- block/blk-zoned.c (`blk_revalidate_disk_zones`, `blkdev_report_zones`, `disk_report_zone`) — block-layer surface; covered in `block/blk-zoned.md` Tier-3 when expanded.
- drivers/scsi/sd.c — sd_init_command, sd_revalidate_disk dispatch logic that invokes ZBC entry points (covered in `drivers/scsi/sd.md` Tier-3).
- include/scsi/scsi_proto.h — opcode tables (covered in `drivers/scsi/00-overview.md`).
- ZBD emulation in null_blk / dm-zoned (separate driver Tier-3 docs).
- NVMe ZNS zone-management (covered in `drivers/nvme/zns.md` Tier-3 when expanded).
- Implementation code
