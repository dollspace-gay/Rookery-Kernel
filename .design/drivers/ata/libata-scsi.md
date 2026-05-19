# Tier-3: drivers/ata/libata-scsi.c — libata SCSI emulation

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/ata/00-overview.md
upstream-paths:
  - drivers/ata/libata-scsi.c (~5053 lines)
  - include/linux/libata.h
  - include/scsi/scsi_cmnd.h
  - include/scsi/scsi_host.h
-->

## Summary

`libata-scsi.c` is the bridge that lets ATA / SATA / ATAPI devices appear as **SCSI devices** to the rest of the Linux storage stack (sd / sr / block layer / multipath / smartctl / hdparm / udisks). Each `struct ata_port` owns a `struct Scsi_Host` (allocated from a libata-provided `scsi_host_template`); SCSI commands arrive via the per-host `queuecommand` callback (`ata_scsi_queuecmd`) which translates the SCSI CDB into either (a) an **ATA taskfile (TF)** issued to the device via `ata_qc_issue`, or (b) an **emulated response** (INQUIRY, READ_CAPACITY, REPORT_LUNS, MODE_SENSE, …) generated wholly in kernel via the `ata_scsiop_*` actor pattern, or (c) an **ATA PASS-THROUGH** (ATA_12 / ATA_16 / VARIABLE_LENGTH_CMD) that hands the raw 7-byte TF over to the device. Translator (`ata_scsi_*_xlat`) functions populate `qc->tf` from CDB fields (`scsi_10_lba_len`, `scsi_16_lba_len`, `scsi_dld`); `ata_scsi_qc_complete` translates ATA status / error registers back into SCSI sense (via `ata_to_sense_error` table-lookup). Critical for: every `/dev/sd<X>` block-device probe on AHCI/SATA, every `smartctl --scan` ATA passthru, every `hdparm` IOCTL, every TRIM (UNMAP / WRITE SAME 16 + UNMAP) on SSD, every CDROM (`/dev/sr<X>`) ATAPI command routing.

This Tier-3 covers `drivers/ata/libata-scsi.c` (~5053 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `ata_scsi_queuecmd()` | per-host queuecommand entry | `AtaScsi::queuecmd` |
| `__ata_scsi_queuecmd()` | per-dev dispatch | `AtaScsi::queuecmd_inner` |
| `ata_scsi_translate()` | per-translate-and-issue | `AtaScsi::translate` |
| `ata_scsi_simulate()` | per-simulate-and-respond | `AtaScsi::simulate` |
| `ata_get_xlat_func()` | per-CDB → xlat-func lookup | `AtaScsi::get_xlat_func` |
| `ata_scsi_rw_xlat()` | per-READ/WRITE 6/10/16 | `AtaScsi::rw_xlat` |
| `ata_scsi_flush_xlat()` | per-SYNCHRONIZE CACHE | `AtaScsi::flush_xlat` |
| `ata_scsi_verify_xlat()` | per-VERIFY 10/16 | `AtaScsi::verify_xlat` |
| `ata_scsi_start_stop_xlat()` | per-START STOP UNIT | `AtaScsi::start_stop_xlat` |
| `ata_scsi_pass_thru()` | per-ATA_12 / ATA_16 / 32-byte | `AtaScsi::pass_thru` |
| `ata_scsi_var_len_cdb_xlat()` | per-VARIABLE LENGTH CDB | `AtaScsi::var_len_cdb_xlat` |
| `ata_scsi_write_same_xlat()` | per-WRITE SAME 16 (UNMAP) | `AtaScsi::write_same_xlat` |
| `ata_scsi_security_inout_xlat()` | per-SECURITY PROTOCOL IN/OUT | `AtaScsi::security_xlat` |
| `ata_scsi_zbc_in_xlat()` / `_out_xlat()` | per-ZBC | `AtaScsi::zbc_in_xlat` / `zbc_out_xlat` |
| `ata_scsi_mode_select_xlat()` | per-MODE SELECT | `AtaScsi::mode_select_xlat` |
| `ata_scsi_qc_new()` | per-qc allocate | `AtaScsi::qc_new` |
| `ata_scsi_qc_complete()` | per-qc completion → SCSI done | `AtaScsi::qc_complete` |
| `ata_scsi_qc_done()` | per-qc final free + done | `AtaScsi::qc_done` |
| `ata_scsi_qc_issue()` | per-defer-aware issue | `AtaScsi::qc_issue` |
| `ata_scsiop_inquiry()` | per-INQUIRY dispatcher | `AtaScsi::inq` |
| `ata_scsiop_inq_std/_00/_80/_83/_89/_b0/_b1/_b2/_b6/_b9` | per-INQUIRY page | `AtaScsi::inq_*` |
| `ata_scsiop_read_cap()` | per-READ CAPACITY 10/16 | `AtaScsi::read_cap` |
| `ata_scsiop_mode_sense()` | per-MODE SENSE 6/10 | `AtaScsi::mode_sense` |
| `ata_scsiop_report_luns()` | per-REPORT LUNS | `AtaScsi::report_luns` |
| `ata_scsiop_maint_in()` | per-MAINTENANCE IN | `AtaScsi::maint_in` |
| `atapi_xlat()` | per-ATAPI relay | `AtaScsi::atapi_xlat` |
| `atapi_qc_complete()` | per-ATAPI completion | `AtaScsi::atapi_qc_complete` |
| `ata_to_sense_error()` | per-ATA status → SCSI sense table | `AtaScsi::to_sense_error` |
| `ata_gen_ata_sense()` | per-ATA-failure sense | `AtaScsi::gen_ata_sense` |
| `ata_gen_passthru_sense()` | per-PASS-THROUGH sense | `AtaScsi::gen_passthru_sense` |
| `ata_scsi_set_sense()` | per-build sense buf | `AtaScsi::set_sense` |
| `ata_scsi_set_sense_information()` | per-set sense LBA descriptor | `AtaScsi::set_sense_info` |
| `ata_scsi_set_passthru_sense_fields()` | per-passthru sense ATA-regs | `AtaScsi::set_passthru_sense_fields` |
| `ata_scsi_set_invalid_field()` | per-INVALID FIELD IN CDB | `AtaScsi::set_invalid_field` |
| `ata_scsi_set_invalid_parameter()` | per-INVALID FIELD IN PARAMETER LIST | `AtaScsi::set_invalid_parameter` |
| `ata_scsi_rbuf_fill()` | per-emulated-response wrapper | `AtaScsi::rbuf_fill` |
| `ata_scsi_sdev_init()` | per-`slave_alloc` | `AtaScsi::sdev_init` |
| `ata_scsi_sdev_configure()` | per-`slave_configure` | `AtaScsi::sdev_configure` |
| `ata_scsi_sdev_destroy()` | per-`slave_destroy` | `AtaScsi::sdev_destroy` |
| `ata_scsi_sdev_config()` | per-sdev libata flags | `AtaScsi::sdev_config` |
| `ata_scsi_dev_config()` | per-queue_limits config | `AtaScsi::dev_config` |
| `ata_scsi_find_dev()` | per-scsi_device → ata_device | `AtaScsi::find_dev` |
| `__ata_scsi_find_dev()` | per-low-level find | `AtaScsi::find_dev_inner` |
| `ata_scsi_scan_host()` | per-port async SCSI scan | `AtaScsi::scan_host` |
| `ata_scsi_user_scan()` | per-`/sys/.../scan` write | `AtaScsi::user_scan` |
| `ata_scsi_hotplug()` | per-hotplug worker | `AtaScsi::hotplug` |
| `ata_scsi_dev_rescan()` | per-rescan worker | `AtaScsi::dev_rescan` |
| `ata_scsi_offline_dev()` | per-dev SCSI offline | `AtaScsi::offline_dev` |
| `ata_scsi_remove_dev()` | per-dev SCSI remove | `AtaScsi::remove_dev` |
| `ata_scsi_handle_link_detach()` | per-link-detach SCSI cleanup | `AtaScsi::handle_link_detach` |
| `ata_scsi_media_change_notify()` | per-AN media-change | `AtaScsi::media_change_notify` |
| `ata_scsi_dma_need_drain()` | per-passthru drain hook | `AtaScsi::dma_need_drain` |
| `ata_scsi_ioctl()` / `ata_sas_scsi_ioctl()` | per-IOCTL entry | `AtaScsi::ioctl` / `sas_ioctl` |
| `ata_cmd_ioctl()` / `ata_task_ioctl()` | per-HDIO_DRIVE_{CMD,TASK} | `AtaScsi::cmd_ioctl` / `task_ioctl` |
| `ata_get_identity()` | per-HDIO_GET_IDENTITY | `AtaScsi::get_identity` |
| `ata_std_bios_param()` | per-BIOS CHS geom | `AtaScsi::std_bios_param` |
| `ata_scsi_unlock_native_capacity()` | per-HPA unlock | `AtaScsi::unlock_native_capacity` |
| `ata_common_sdev_groups[]` | per-sysfs attr group | shared |
| `ata_scsi_park_show()` / `_store()` | per-`unload_heads` sysfs | `AtaScsi::park_*` |
| `ata_scsi_deferred_qc_work` | per-deferred-qc worker | `AtaScsi::deferred_qc_work` |
| `ata_scsi_requeue_deferred_qc()` | per-reset deferred-qc requeue | `AtaScsi::requeue_deferred_qc` |
| `ata_scsi_add_hosts()` | per-host SCSI registration | `AtaScsi::add_hosts` |
| `EXPORT_SYMBOL_GPL(ata_scsi_*)` | per-LLD ABI surface | `extern fn` |

## Compatibility contract

REQ-1: `ata_scsi_queuecmd(shost, cmd)` (SCSI host `queuecommand` callback):
- `ap = ata_shost_to_port(shost)` — recover `ata_port` from hostdata.
- `spin_lock_irqsave(ap->lock, irq_flags)`.
- `dev = ata_scsi_find_dev(ap, cmd->device)`; if NULL: `cmd->result = DID_BAD_TARGET << 16`; `scsi_done(cmd)`.
- Else: `rc = __ata_scsi_queuecmd(cmd, dev)`.
- `spin_unlock_irqrestore`.
- Returns `enum scsi_qc_status` (0 / `SCSI_MLQUEUE_DEVICE_BUSY` / `SCSI_MLQUEUE_HOST_BUSY`).

REQ-2: `__ata_scsi_queuecmd(scmd, dev)`:
- If `ata_port_eh_scheduled(ap)`: return `SCSI_MLQUEUE_DEVICE_BUSY` (re-issue after EH).
- Validate `scmd->cmd_len > 0`.
- ATA / ZAC device class:
  - Validate `scmd->cmd_len ≤ dev->cdb_len`.
  - `xlat_func = ata_get_xlat_func(dev, scmd->cmnd[0])`.
- ATAPI device class (and `scsi_op != ATA_16 ∨ !atapi_passthru16`):
  - Validate `COMMAND_SIZE(scsi_op) ≤ min(scmd->cmd_len, dev->cdb_len, ATAPI_CDB_LEN)`.
  - `xlat_func = atapi_xlat`.
- ATAPI device with ATA_16 passthru: validate len ≤ 16, `xlat_func = ata_get_xlat_func(dev, scsi_op)`.
- If `xlat_func`: return `ata_scsi_translate(dev, scmd, xlat_func)`.
- Else: `ata_scsi_simulate(dev, scmd)`; return 0.

REQ-3: `ata_get_xlat_func(dev, cmd)` switch table:
- `READ_6 | READ_10 | READ_16 | WRITE_6 | WRITE_10 | WRITE_16` → `ata_scsi_rw_xlat`.
- `WRITE_SAME_16` → `ata_scsi_write_same_xlat`.
- `SYNCHRONIZE_CACHE | SYNCHRONIZE_CACHE_16` → `ata_scsi_flush_xlat` (only if `ata_try_flush_cache(dev)`).
- `VERIFY | VERIFY_16` → `ata_scsi_verify_xlat`.
- `ATA_12 | ATA_16` → `ata_scsi_pass_thru`.
- `VARIABLE_LENGTH_CMD` → `ata_scsi_var_len_cdb_xlat`.
- `MODE_SELECT | MODE_SELECT_10` → `ata_scsi_mode_select_xlat`.
- `ZBC_IN` → `ata_scsi_zbc_in_xlat`; `ZBC_OUT` → `ata_scsi_zbc_out_xlat`.
- `SECURITY_PROTOCOL_IN | _OUT` → `ata_scsi_security_inout_xlat` (only if `ATA_DFLAG_TRUSTED`).
- `START_STOP` → `ata_scsi_start_stop_xlat`.
- Else: NULL → `ata_scsi_simulate`.

REQ-4: `ata_scsi_translate(dev, cmd, xlat_func)`:
- Pre: `lockdep_assert_held(ap->lock)`.
- `qc = ata_scsi_qc_new(dev, cmd)`; if NULL → return 0 (qc_new already called `scsi_done`).
- If data direction non-NONE:
  - Validate `scsi_bufflen(cmd) ≥ 1`.
  - `ata_sg_init(qc, scsi_sglist(cmd), scsi_sg_count(cmd))`.
  - `qc->dma_dir = cmd->sc_data_direction`.
- `qc->complete_fn = ata_scsi_qc_complete`.
- `xlat_func(qc)` → if non-zero (failure) → `ata_qc_free(qc)`; `scsi_done(cmd)`; return 0.
- Else `ata_scsi_qc_issue(ap, qc)` — returns `enum scsi_qc_status`.

REQ-5: `ata_scsi_simulate(dev, cmd)`:
- Switch `cmnd[0]`:
  - `INQUIRY` → `ata_scsi_rbuf_fill(dev, cmd, ata_scsiop_inquiry)`.
  - `MODE_SENSE | MODE_SENSE_10` → `ata_scsiop_mode_sense`.
  - `READ_CAPACITY | SERVICE_ACTION_IN_16` → `ata_scsiop_read_cap`.
  - `REPORT_LUNS` → `ata_scsiop_report_luns`.
  - `REQUEST_SENSE` → `ata_scsi_set_sense(dev, cmd, 0, 0, 0)`.
  - `SYNCHRONIZE_CACHE | _16` (when WB cache disabled) → no-op success.
  - `REZERO_UNIT | SEEK_6 | SEEK_10 | TEST_UNIT_READY` → no-op success.
  - `SEND_DIAGNOSTIC` → validate SelfTest only (bit3=0, code=0x4, param=0).
  - `MAINTENANCE_IN` → `ata_scsiop_maint_in`.
  - Default → `ata_scsi_set_sense(ILLEGAL_REQUEST, 0x20, 0x0)` "Invalid command operation code".
- Always `scsi_done(cmd)`.

REQ-6: `ata_scsi_rbuf_fill(dev, cmd, actor)`:
- Global `ata_scsi_rbuf[ATA_SCSI_RBUF_SIZE]` protected by `ata_scsi_rbuf_lock`.
- `memset(ata_scsi_rbuf, 0, ATA_SCSI_RBUF_SIZE)`.
- `len = actor(dev, cmd, ata_scsi_rbuf)` — returns 0 on error (with sense already set).
- If `len`: `sg_copy_from_buffer(scsi_sglist(cmd), scsi_sg_count(cmd), ata_scsi_rbuf, ATA_SCSI_RBUF_SIZE)`; `cmd->result = SAM_STAT_GOOD`; set residue.

REQ-7: `ata_scsiop_inquiry(dev, cmd, rbuf)`:
- `cdb[1] & 1` (EVPD): dispatch by `cdb[2]` (page code):
  - 0x00 → `ata_scsiop_inq_00` (supported VPD list).
  - 0x80 → `ata_scsiop_inq_80` (serial number).
  - 0x83 → `ata_scsiop_inq_83` (device identification — WWN, model).
  - 0x89 → `ata_scsiop_inq_89` (ATA Information page — embeds IDENTIFY).
  - 0xB0 → `ata_scsiop_inq_b0` (Block Limits — OPTIMAL_XFER, MAX_UNMAP).
  - 0xB1 → `ata_scsiop_inq_b1` (Block Device Characteristics — RPM, FORM_FACTOR).
  - 0xB2 → `ata_scsiop_inq_b2` (Logical Block Provisioning).
  - 0xB6 → `ata_scsiop_inq_b6` (Zoned Block Device Characteristics — ZAC only).
  - 0xB9 → `ata_scsiop_inq_b9` (Concurrent Positioning Ranges — CPR).
- Else (standard INQUIRY): `ata_scsiop_inq_std` — emit hdr + versions + vendor "ATA" / model / fw-rev.

REQ-8: `ata_scsiop_read_cap(dev, cmd, rbuf)`:
- `last_lba = dev->n_sectors - 1`.
- READ_CAPACITY (10): truncate to 0xFFFFFFFF, 8-byte response: u32 last_lba + u32 sector_size.
- READ_CAPACITY (16) (`SERVICE_ACTION_IN_16` w/ SA `SAI_READ_CAPACITY_16`): 16-byte response: u64 last_lba + u32 sector_size + RC_BASIS (if ZAC) + log2_per_phys + (lowest_aligned | LBPME | LBPRZ).
- TRIM supported (`ata_id_has_trim(dev->id)` ∧ !`ATA_QUIRK_NOTRIM`) → set `LBPME = 1 << 7` of byte 14.
- `ata_id_has_zero_after_trim(dev->id) ∧ ATA_QUIRK_ZERO_AFTER_TRIM` → set `LBPRZ = 1 << 6`.

REQ-9: `ata_scsi_rw_xlat(qc)` (READ/WRITE 6/10/16 → ATA TF):
- Pre: `qc` allocated, `tf` zeroed.
- Parse `cdb[0]`:
  - WRITE_6/10/16 → `tf_flags |= ATA_TFLAG_WRITE`.
  - READ_6 / WRITE_6 (6-byte): `scsi_6_lba_len`; n_block==0 ⟹ 256.
  - READ_10 / WRITE_10: `scsi_10_lba_len`; `cdb[1] & 0x8` ⟹ FUA.
  - READ_16 / WRITE_16: `scsi_16_lba_len`; `scsi_dld(cdb)` for CDL; FUA.
- Validate `ata_check_nblocks(scmd, n_block)`; if exceeds queue limits → INVALID FIELD IN CDB.
- If `n_block == 0` (10/16-byte): `cmd->result = SAM_STAT_GOOD`; return 1 (nothing-to-do — but 6-byte 0 is 256).
- `qc->flags |= ATA_QCFLAG_IO`; `qc->nbytes = n_block * sector_size`.
- `rc = ata_build_rw_tf(qc, block, n_block, tf_flags, dld, class)`:
  - 0 → success.
  - -ERANGE → "LBA out of range" sense.
  - other → "INVALID FIELD IN CDB".

REQ-10: `ata_scsi_pass_thru(qc)` (ATA_12 / ATA_16 / 32-byte VARIABLE_LENGTH):
- `cdb_offset = 9` for VARIABLE_LENGTH_CMD, else 0.
- `tf->protocol = ata_scsi_map_proto(cdb[1 + cdb_offset])` — extract bits[1:4]; if ATA_PROT_UNKNOWN → INVALID FIELD.
- If `T_LENGTH == 0` (cdb[2+off] & 3 == 0): require DMA_NONE direction; NCQ→`ATA_PROT_NCQ_NODATA`.
- Set `ATA_TFLAG_LBA`.
- ATA_16 (CDB byte 1 bit 0 = EXTEND): copy hob_* + LBA48 flag from cdb[3,5,7,9,11]; copy feature/nsect/lbal/lbam/lbah/device/command from cdb[4,6,8,10,12,13,14].
- ATA_12: copy feature/nsect/lbal/lbam/lbah/device/command from cdb[3..9]; no LBA48.
- 32-byte: extract auxiliary from cdb[28..31] (big-endian u32); same TF layout offset-shifted.
- NCQ: `tf->nsect = qc->hw_tag << 3` (overwrite count with tag for FPDMA).
- Enforce master/slave: `tf->device = dev->devno ? (tf->device | ATA_DEV1) : (tf->device & ~ATA_DEV1)`.
- Special TF.commands (READ/WRITE LONG, READ/WRITE MULTIPLE, PIO single-sector): adjust `qc->nbytes` per command idiosyncrasies.

REQ-11: `ata_scsi_write_same_xlat(qc)` (WRITE SAME 16 with UNMAP — TRIM):
- Requires `ata_dma_enabled(dev)` ∧ block-layer request (not SG_IO passthrough).
- `scsi_16_lba_len(cdb, &block, &n_block)`.
- `unmap = cdb[1] & 0x8`; require unmap=1 ∧ `ata_id_has_trim(dev->id)` ∧ `!ATA_QUIRK_NOTRIM`.
- `trmax = sector_size >> 3` (DSM TRIM range entries per sector).
- Validate `n_block ≤ 0xffff * trmax`.
- `ata_format_dsm_trim_descr(scmd, trmax, block, n_block)` — fills payload with 8-byte LBA range descriptors.
- NCQ TRIM (`ata_ncq_enabled ∧ ata_fpdma_dsm_supported(dev)`):
  - `tf->command = ATA_CMD_FPDMA_SEND`; `tf->hob_nsect = ATA_SUBCMD_FPDMA_SEND_DSM`; `tf->nsect = qc->hw_tag << 3`; `auxiliary = 1`.
- Non-NCQ:
  - `tf->command = ATA_CMD_DSM`; `tf->feature = ATA_DSM_TRIM`.
- `tf->flags |= ATA_TFLAG_ISADDR | ATA_TFLAG_DEVICE | ATA_TFLAG_LBA48 | ATA_TFLAG_WRITE`.

REQ-12: `ata_scsi_flush_xlat(qc)` (SYNCHRONIZE CACHE 10/16):
- `tf->protocol = ATA_PROT_NODATA`.
- `tf->command = ATA_CMD_FLUSH_EXT` (LBA48-capable) or `ATA_CMD_FLUSH`.
- `tf->flags |= ATA_TFLAG_DEVICE`.

REQ-13: `ata_scsi_verify_xlat(qc)` (VERIFY 10/16):
- Build TF using `ata_build_rw_tf`-like logic but with `ATA_CMD_VERIFY` / `ATA_CMD_VERIFY_EXT`.
- No data transfer; sector-count + LBA only.

REQ-14: `ata_scsi_start_stop_xlat(qc)` (START STOP UNIT):
- Decode `LOEJ` (eject), `START`, `IMMED`, `POWER CONDITION`.
- Map to `ATA_CMD_STANDBYNOW1`, `ATA_CMD_IDLEIMMEDIATE`, `ATA_CMD_SLEEP`, or VERIFY (active-on-restart).
- ATA_PROT_NODATA.

REQ-15: `atapi_xlat(qc)`:
- Build CDB-relay TF: `tf->command = ATA_CMD_PACKET`, `tf->protocol = ATA_PROT_ATAPI` (or `ATAPI_DMA` for DMA-capable + write cmd subset).
- `qc->cdb` = `scmd->cmnd` (16 bytes).
- DMA or PIO direction per `ATA_QCFLAG_DMAMAP`.

REQ-16: `ata_scsi_qc_complete(qc)` (qc completion callback installed by `translate`):
- `cdb[0]`: detect ATA_12 / ATA_16 passthru → `is_ata_passthru`.
- `is_ck_cond_request = cdb[2] & 0x20` (CK_COND in passthru forces sense even on success).
- `have_sense = qc->flags & ATA_QCFLAG_SENSE_VALID`.
- `is_error = qc->err_mask != 0`.
- Passthru + (CK_COND ∨ error ∨ have_sense):
  - `ata_gen_passthru_sense(qc)` if !have_sense.
  - `ata_scsi_set_passthru_sense_fields(qc)` — copies result_tf registers into sense descriptor 0x09.
  - CK_COND: `set_status_byte(scmd, SAM_STAT_CHECK_CONDITION)`.
- Else if error:
  - `ata_gen_ata_sense(qc)` if !have_sense.
  - `ata_scsi_set_sense_information(qc)` — descriptor 0x00 for LBA on medium error.
- `ata_scsi_qc_done(qc, false, 0)` → `ata_qc_free(qc)`; `done(scmd)`.
- `ata_scsi_schedule_deferred_qc(ap)` to dispatch any deferred qc.

REQ-17: `ata_to_sense_error(drv_stat, drv_err, *sk, *asc, *ascq)` (ATA → SCSI sense map):
- `sense_table[]` matches `drv_err` byte (ICRC|ABRT, ECC, BBD, ID, MAR, MC, MCR, …) → (sense_key, asc, ascq).
- `stat_table[]` fallback by `drv_stat` (BUSY, DF, CORR).
- Special cases: `ATA_BUSY` ⟹ ignore `drv_err` (invalid); ABRT-alone uses status decode.
- Default: `ABORTED_COMMAND, 0x00, 0x00`.

REQ-18: `ata_gen_passthru_sense(qc)`:
- Require `ATA_QCFLAG_RTF_FILLED`; else `ABORTED_COMMAND` if `err_mask`.
- If status has BUSY|DF|ERR|DRQ ∨ err_mask: `ata_to_sense_error(tf->status, tf->error, ...)` → set_sense.
- Else (CK_COND with no error): `scsi_build_sense(cmd, 1, RECOVERED_ERROR, 0, 0x1D)` "ATA PASS-THROUGH INFORMATION AVAILABLE" (descriptor format, hardcoded for hdparm/hddtemp/udisks bug-compat).

REQ-19: `ata_gen_ata_sense(qc)`:
- `ata_dev_disabled(dev)` → `NOT_READY, 0x04, 0x21` "LU not ready, hard reset required".
- `ata_id_is_locked(dev->id)` → `DATA_PROTECT, 0x74, 0x71` "LU access not authorized".
- `!ATA_QCFLAG_RTF_FILLED` → `ABORTED_COMMAND, 0, 0`.
- Else `ata_to_sense_error` from result TF.

REQ-20: `ata_scsi_set_sense(dev, cmd, sk, asc, ascq)`:
- Builds descriptor or fixed sense per `ATA_DFLAG_D_SENSE` (D_SENSE bit from CONTROL mode page).
- Stores in `cmd->sense_buffer`.

REQ-21: `ata_scsi_qc_new(dev, cmd) -> qc`:
- `ata_qc_new_init(dev, scmd->request->tag)` — assigns a tag from block layer's pre-allocated pool (libata uses scsi_cmnd->request as the tag source).
- Stamps `qc->scsicmd = cmd`, `qc->scsidone = scsi_done`.
- On failure (no free tag): set DID_ERROR, scsi_done.

REQ-22: `ata_scsi_qc_issue(ap, qc)`:
- If `ap->ops->qc_defer == NULL`: `ata_qc_issue(qc)`; return 0.
- If `ap->deferred_qc`: free new qc; return `SCSI_MLQUEUE_DEVICE_BUSY`.
- Call `ap->ops->qc_defer(qc)`:
  - 0 → issue.
  - `ATA_DEFER_LINK` → return `SCSI_MLQUEUE_DEVICE_BUSY`.
  - `ATA_DEFER_PORT` → return `SCSI_MLQUEUE_HOST_BUSY`.
- If deferred + non-NCQ: stash as `ap->deferred_qc`; return 0 (mid-layer thinks issued); `ata_scsi_deferred_qc_work` will retry.

REQ-23: `ata_scsi_deferred_qc_work(work)`:
- Locked re-issue path; calls `WARN_ON_ONCE(ap->ops->qc_defer(qc))` then `ata_qc_issue(qc)`.

REQ-24: `ata_scsi_requeue_deferred_qc(ap)`:
- On reset / NCQ failure: cancel work; `ata_scsi_qc_done(qc, true, DID_REQUEUE << 16)`.

REQ-25: SCSI host template (libata-supplied per controller; `ATA_BASE_SHT` macro):
- `.module`, `.name`, `.queuecommand = ata_scsi_queuecmd`.
- `.this_id = ATA_SHT_THIS_ID` (−1).
- `.emulated = ATA_SHT_EMULATED`.
- `.proc_name = "libata"`.
- `.slave_alloc = ata_scsi_sdev_init`.
- `.slave_configure = ata_scsi_sdev_configure`.
- `.slave_destroy = ata_scsi_sdev_destroy`.
- `.bios_param = ata_std_bios_param`.
- `.unlock_native_capacity = ata_scsi_unlock_native_capacity`.
- `.eh_strategy_handler = ata_scsi_error` (defined in libata-eh.c).
- `.ioctl = ata_scsi_ioctl`, `.compat_ioctl = ata_scsi_ioctl`.
- `.cmd_size = sizeof(struct ata_queued_cmd)` (block-layer reservation).
- `.sdev_groups = ata_common_sdev_groups`.

REQ-26: `ata_scsi_sdev_init(sdev)`:
- `ata_scsi_sdev_config(sdev)` — set `use_10_for_rw=1`, `use_10_for_ms=1`, `no_write_same=1` (unless TRIM via WRITE_SAME_16 path used).
- `device_link_add(&sdev->sdev_gendev, &ap->tdev, DL_FLAG_STATELESS | DL_FLAG_PM_RUNTIME | DL_FLAG_RPM_ACTIVE)` — ensures sd suspends before port.

REQ-27: `ata_scsi_sdev_configure(sdev, lim)`:
- `dev = __ata_scsi_find_dev(ap, sdev)`; if found `ata_scsi_dev_config(sdev, lim, dev)`:
  - Set `queue_limits` from ATA: `max_hw_sectors`, `physical_block_size`, `io_min/opt`, `discard_granularity`, `max_discard_sectors`, `max_write_zeroes_sectors`, `chunk_sectors` (zoned), `dma_alignment`.
  - Set sdev properties: `manage_runtime_pm`, `removable`, `allow_restart`, `simple_tags` (NCQ), `max_queue_depth`.

REQ-28: `ata_scsi_sdev_destroy(sdev)`:
- `device_link_remove`.
- Under `ap->lock`: if `dev->sdev == sdev` and warm-unplug-initiated: `dev->sdev = NULL`; `dev->flags |= ATA_DFLAG_DETACH`; `ata_port_schedule_eh(ap)`.

REQ-29: `ata_scsi_find_dev(ap, scsidev)`:
- `dev = __ata_scsi_find_dev(ap, scsidev)`:
  - Non-PMP: skip `channel != 0 ∨ lun != 0`; `devno = scsidev->id`.
  - PMP: skip `id != 0 ∨ lun != 0`; `devno = scsidev->channel`.
- Then `ata_find_dev(ap, devno)`.
- Return NULL if `!ata_adapter_is_online(ap) ∨ !ata_dev_enabled(dev)`.

REQ-30: `ata_scsi_scan_host(ap, sync)`:
- Under `ap->scsi_scan_mutex`.
- For each enabled dev with `!dev->sdev`: build target template + `scsi_scan_target(&ap->scsi_host->shost_gendev, channel, id, lun, SCSI_SCAN_INITIAL)`.
- Async on first probe; sync on user-rescan.

REQ-31: `ata_scsi_hotplug(work)` (delayed_work):
- For each link: handle detached (`ATA_DFLAG_DETACH`) devices via `ata_scsi_remove_dev`.
- Then `ata_scsi_scan_host(ap, 0)` for new devices.
- Re-schedule if work remains.

REQ-32: `ata_scsi_dev_rescan(work)`:
- For each dev: if `ATA_DFLAG_RESCAN`: `scsi_rescan_device(dev->sdev)`.

REQ-33: `ata_scsi_offline_dev(dev)`:
- Sets sdev offline via `scsi_device_set_state(SDEV_OFFLINE)`.

REQ-34: `ata_scsi_remove_dev(dev)`:
- `scsi_remove_device(dev->sdev)`; clear `dev->sdev`; `ata_dev_disable(dev)`.

REQ-35: `ata_scsi_handle_link_detach(link)`:
- Per dev on link: schedule remove via `ata_scsi_remove_dev`.

REQ-36: `ata_scsi_media_change_notify(dev)`:
- Triggered by ATAPI AN (Asynchronous Notification).
- `scsi_device_set_media_present` + sysfs uevent.

REQ-37: IOCTL surface:
- `ata_scsi_ioctl(scsidev, cmd, arg)` → `ata_sas_scsi_ioctl(ap, ...)` for SAS.
- `HDIO_GET_IDENTITY` → `ata_get_identity` (returns IDENTIFY words via ioctl).
- `HDIO_DRIVE_CMD` → `ata_cmd_ioctl` (issue ATA cmd, return status).
- `HDIO_DRIVE_TASK` → `ata_task_ioctl` (issue with full TF).
- `HDIO_DRIVE_TASKFILE` falls through to SG_IO/PASS_THRU path via scsi.

REQ-38: `ata_std_bios_param(sdev, bdev, capacity, geom[3])`:
- Classic IDE-on-BIOS geom: `geom[2] = capacity / (255 * 63)`; clamp heads/sectors.

REQ-39: `ata_scsi_unlock_native_capacity(sdev)`:
- Sets `dev->flags |= ATA_DFLAG_UNLOCK_HPA`; `ata_port_schedule_eh(ap)` to re-IDENTIFY and unlock HPA.

REQ-40: `ata_scsi_dma_need_drain(rq)`:
- Returns true if request is `ATA_16` passthru — block layer must drain to avoid DMA over-read.

REQ-41: `ata_common_sdev_groups` (sysfs):
- `dev_attr_unload_heads` (ATA8 IDLE_IMMEDIATE UNLOAD): `unload_heads_show/store` (named `ata_scsi_park_show/store` internally — parks head for shock-protection).

## Acceptance Criteria

- [ ] AC-1: `ata_scsi_queuecmd(shost, cmd)` with `DID_BAD_TARGET` device returns 0 and calls `scsi_done` with `DID_BAD_TARGET << 16`.
- [ ] AC-2: `ata_scsi_queuecmd` while `ata_port_eh_scheduled(ap)` returns `SCSI_MLQUEUE_DEVICE_BUSY`.
- [ ] AC-3: `ata_get_xlat_func(dev, READ_10)` returns `ata_scsi_rw_xlat`; `WRITE_SAME_16` → `ata_scsi_write_same_xlat`; `ATA_16` → `ata_scsi_pass_thru`.
- [ ] AC-4: `ata_scsi_simulate(dev, INQUIRY)` produces 8-byte SCSI vendor "ATA" + model + fw-rev when EVPD=0.
- [ ] AC-5: `ata_scsiop_read_cap` with `dev->n_sectors = 1<<33` and READ_CAPACITY (10) reports `last_lba = 0xFFFFFFFF`; with READ_CAPACITY (16) reports actual `last_lba`.
- [ ] AC-6: `ata_scsi_rw_xlat` with WRITE_10 LBA=0, n_block=8 produces `tf.flags & ATA_TFLAG_WRITE` and (for NCQ-enabled dev) `tf.command == ATA_CMD_FPDMA_WRITE`.
- [ ] AC-7: `ata_scsi_write_same_xlat` with UNMAP=0 returns INVALID FIELD IN CDB at offset 1, bit 3.
- [ ] AC-8: `ata_scsi_pass_thru(ATA_16, EXTEND=1)` copies hob_* registers from cdb[3,5,7,9,11] and sets `ATA_TFLAG_LBA48`.
- [ ] AC-9: `ata_to_sense_error(ATA_ERR, ATA_ICRC|ATA_ABORTED, ...)` returns `(ABORTED_COMMAND, 0x47, 0x00)` (SCSI parity error).
- [ ] AC-10: `ata_to_sense_error(ATA_ERR, ATA_UNC, ...)` returns `(MEDIUM_ERROR, 0x11, 0x04)` (unrecovered read error).
- [ ] AC-11: `ata_gen_passthru_sense` with CK_COND=1 and err_mask=0 produces `RECOVERED_ERROR, 0x00, 0x1D` (ATA PASS-THROUGH INFO AVAILABLE).
- [ ] AC-12: `ata_gen_ata_sense` on `ata_dev_disabled` device produces `NOT_READY, 0x04, 0x21`.
- [ ] AC-13: `ata_scsi_qc_complete` on failed non-passthru qc invokes `ata_gen_ata_sense` and `ata_scsi_set_sense_information`.
- [ ] AC-14: `ata_scsi_sdev_init` sets `use_10_for_rw=1`, `use_10_for_ms=1` on sdev and creates device link to `ap->tdev`.
- [ ] AC-15: `ata_scsi_find_dev(ap, sdev)` on offline adapter returns NULL.

## Architecture

```
struct AtaScsiOps {
  queuecmd: fn(*ScsiHost, *ScsiCmnd) -> ScsiQcStatus,
  /* dispatch table for xlat */
}

type AtaXlatFunc = fn(*AtaQueuedCmd) -> u32;  // 0 = success, !0 = abort

const SCSI_HOST_TEMPLATE: ScsiHostTemplate = ScsiHostTemplate {
  module: this_module(),
  name: "libata",
  proc_name: "libata",
  queuecommand: AtaScsi::queuecmd,
  this_id: -1,
  emulated: 1,
  slave_alloc: AtaScsi::sdev_init,
  slave_configure: AtaScsi::sdev_configure,
  slave_destroy: AtaScsi::sdev_destroy,
  bios_param: AtaScsi::std_bios_param,
  unlock_native_capacity: AtaScsi::unlock_native_capacity,
  eh_strategy_handler: AtaEh::scsi_error,
  ioctl: AtaScsi::ioctl,
  compat_ioctl: AtaScsi::ioctl,
  cmd_size: size_of::<AtaQueuedCmd>(),
  sdev_groups: &AtaScsi::COMMON_SDEV_GROUPS,
  /* per-LLD overrides: can_queue, sg_tablesize, dma_boundary, max_segment_size */
};
```

`AtaScsi::queuecmd(shost, cmd) -> ScsiQcStatus`:
1. `ap = ata_shost_to_port(shost)`.
2. Lock `ap->lock` (IRQ-save).
3. `dev = AtaScsi::find_dev(ap, cmd.device)`.
4. If `dev`: `rc = AtaScsi::queuecmd_inner(cmd, dev)`.
5. Else: `cmd.result = DID_BAD_TARGET << 16`; `scsi_done(cmd)`; `rc = 0`.
6. Unlock; return `rc`.

`AtaScsi::queuecmd_inner(cmd, dev) -> ScsiQcStatus`:
1. If `ata_port_eh_scheduled(ap)`: return `SCSI_MLQUEUE_DEVICE_BUSY`.
2. `op = cmd.cmnd[0]`.
3. Per dev.class:
   - ATA/ZAC: `xlat = AtaScsi::get_xlat_func(dev, op)`.
   - ATAPI ∧ (op ≠ ATA_16 ∨ !atapi_passthru16): `xlat = AtaScsi::atapi_xlat`.
   - ATAPI ∧ ATA_16 passthru: `xlat = AtaScsi::get_xlat_func(dev, op)`.
4. If `xlat`: return `AtaScsi::translate(dev, cmd, xlat)`.
5. Else: `AtaScsi::simulate(dev, cmd)`; return 0.

`AtaScsi::translate(dev, cmd, xlat) -> ScsiQcStatus`:
1. `qc = AtaScsi::qc_new(dev, cmd)`; if NULL → 0 (already errored).
2. If data-dir: validate `scsi_bufflen ≥ 1`; `AtaQueuedCmd::sg_init(qc, scsi_sglist, scsi_sg_count)`; `qc.dma_dir = cmd.sc_data_direction`.
3. `qc.complete_fn = AtaScsi::qc_complete`.
4. `rc = xlat(qc)`; if non-zero → free + done.
5. Return `AtaScsi::qc_issue(ap, qc)`.

`AtaScsi::qc_complete(qc)`:
1. `cdb = qc.scsicmd.cmnd`.
2. `is_passthru = cdb[0] in {ATA_12, ATA_16}`.
3. `is_ck_cond = cdb[2] & 0x20`.
4. `have_sense = qc.flags & ATA_QCFLAG_SENSE_VALID`.
5. `is_error = qc.err_mask != 0`.
6. If passthru ∧ (ck_cond ∨ error ∨ have_sense):
   - !have_sense → `gen_passthru_sense(qc)`.
   - `set_passthru_sense_fields(qc)`.
   - ck_cond → `set_status_byte(scsicmd, SAM_STAT_CHECK_CONDITION)`.
7. Else if error:
   - !have_sense → `gen_ata_sense(qc)`.
   - `set_sense_information(qc)`.
8. `qc_done(qc, false, 0)` → `ata_qc_free(qc)`; `done(scsicmd)`.
9. `schedule_deferred_qc(ap)`.

`AtaScsi::rw_xlat(qc) -> u32`:
1. Parse cdb opcode; set `tf_flags |= ATA_TFLAG_WRITE` for writes.
2. Decode (block, n_block) via 6/10/16 helper.
3. FUA bit (cdb[1] bit 3) → `ATA_TFLAG_FUA`.
4. `ata_check_nblocks(scmd, n_block)`; bad → INVALID FIELD.
5. If n_block == 0 (and not 6-byte): `result = SAM_STAT_GOOD`; return 1.
6. `qc.flags |= ATA_QCFLAG_IO`; `qc.nbytes = n_block * sector_size`.
7. `ata_build_rw_tf(qc, block, n_block, tf_flags, dld, class)`:
   - 0 → return 0.
   - -ERANGE → ILLEGAL_REQUEST 0x21 0x0 → return 1.
   - other → INVALID FIELD → return 1.

`AtaScsi::pass_thru(qc) -> u32`:
1. `cdb_offset = (cdb[0] == VARIABLE_LENGTH_CMD) ? 9 : 0`.
2. `protocol = ata_scsi_map_proto(cdb[1 + cdb_offset])`; unknown → INVALID FIELD.
3. T_LENGTH==0 path: require DMA_NONE; NCQ→NCQ_NODATA.
4. `tf.flags |= ATA_TFLAG_LBA`.
5. Branch by cdb[0]: ATA_16 (EXTEND→LBA48 + hob_*), ATA_12, 32-byte (auxiliary).
6. Copy feature/nsect/lbal/lbam/lbah/device/command from CDB.
7. NCQ: `tf.nsect = qc.hw_tag << 3`.
8. Master/slave: enforce ATA_DEV1 bit per `dev.devno`.
9. Per-cmd nbytes adjustment (READ/WRITE LONG, MULTIPLE).

`AtaScsi::to_sense_error(stat, err, *sk, *asc, *ascq)`:
1. ATA_BUSY ⟹ err = 0 (invalid).
2. If err: walk sense_table (err byte match); set (sk, asc, ascq); return.
3. Else walk stat_table (stat byte match); set; return.
4. Default: `(ABORTED_COMMAND, 0, 0)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rbuf_lock_serializes_actor` | INVARIANT | per-`rbuf_fill`: `ata_scsi_rbuf` mutated only with `ata_scsi_rbuf_lock` held. |
| `queuecmd_returns_only_valid_status` | INVARIANT | per-`queuecmd`: return ∈ {0, `SCSI_MLQUEUE_DEVICE_BUSY`, `SCSI_MLQUEUE_HOST_BUSY`}. |
| `find_dev_offline_returns_null` | INVARIANT | per-`find_dev`: `!ata_adapter_is_online ⟹ NULL`. |
| `rw_xlat_nblock_zero_handled` | INVARIANT | per-`rw_xlat`: n_block==0 → success-zero (10/16) or 256 (6). |
| `pass_thru_proto_validated` | INVARIANT | per-`pass_thru`: ATA_PROT_UNKNOWN → INVALID FIELD; never issues unknown protocol. |
| `write_same_requires_unmap_and_trim` | INVARIANT | per-`write_same_xlat`: !UNMAP ∨ !`ata_id_has_trim` → INVALID FIELD. |
| `simulate_always_calls_scsi_done` | INVARIANT | per-`simulate`: every path ends with `scsi_done(cmd)`. |
| `qc_complete_no_active_after_done` | INVARIANT | per-`qc_complete`: on exit qc freed, scmd done; no active reference. |
| `sense_translation_total` | INVARIANT | per-`to_sense_error`: always yields some (sk, asc, ascq) — no UB read. |
| `passthru_sense_requires_rtf` | INVARIANT | per-`gen_passthru_sense`: `!ATA_QCFLAG_RTF_FILLED` → ABORTED_COMMAND, no UB read of result_tf. |
| `find_dev_pmp_devno_inbounds` | INVARIANT | per-`find_dev`: PMP path indexes `pmp_link[devno]` with `devno < nr_pmp_links`. |

### Layer 2: TLA+

`drivers/ata/libata-scsi.tla`:
- Per-cmd state machine: `SCSI_ARRIVE → DEV_LOOKUP → {SIMULATE | TRANSLATE → QC_ISSUE → ATA_COMPLETE → SCSI_DONE} | EH_BUSY → REQUEUE`.
- Per-deferred-qc model: `qc_defer == ATA_DEFER_*` → optional stash → `deferred_qc_work` retry.
- Properties:
  - `safety_scsi_done_exactly_once` — every accepted scsi_cmnd has exactly one `scsi_done` call.
  - `safety_no_qc_leak_on_xlat_fail` — xlat failure path calls `ata_qc_free` before `scsi_done`.
  - `safety_passthru_ck_cond_implies_check_condition` — CK_COND=1 ⟹ `SAM_STAT_CHECK_CONDITION` byte set.
  - `safety_eh_busy_does_not_consume_qc` — busy-return path does not allocate qc.
  - `liveness_every_scsi_cmd_terminates` — given finite EH and finite defer chain, every cmd reaches `scsi_done`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `queuecmd` post: returns 0 ⟹ `scsi_done` will be called by some path (translate completion or simulate) | `AtaScsi::queuecmd` |
| `translate` post: qc allocated ⟹ either issued or freed+done | `AtaScsi::translate` |
| `rw_xlat` post: success ⟹ `tf.command` ∈ ATA R/W cmds and `qc.nbytes == n_block * sector_size` | `AtaScsi::rw_xlat` |
| `pass_thru` post: success ⟹ `tf` filled from CDB; protocol ≠ UNKNOWN | `AtaScsi::pass_thru` |
| `write_same_xlat` post: success ⟹ `tf.command ∈ {ATA_CMD_DSM, ATA_CMD_FPDMA_SEND}` with `feature == ATA_DSM_TRIM` (non-NCQ) | `AtaScsi::write_same_xlat` |
| `read_cap` post: 10-byte buf ≤ 0xFFFFFFFF; 16-byte buf reflects `dev.n_sectors - 1` | `AtaScsi::read_cap` |
| `qc_complete` post: scsicmd has sense set if passthru CK_COND ∨ error | `AtaScsi::qc_complete` |
| `to_sense_error` post: (sk, asc, ascq) all assigned for any input | `AtaScsi::to_sense_error` |
| `find_dev` post: result device (if non-NULL) has `enabled ∧ link.ap == ap ∧ adapter_online` | `AtaScsi::find_dev` |

### Layer 4: Verus/Creusot functional

`Per-SCSI command arrival → host_lock acquired → dev resolved → CDB dispatch (xlat | simulate | atapi_xlat) → on translate: TF built (ata_build_rw_tf / pass_thru / write_same) → ata_qc_issue → LLD qc_issue → completion → ata_scsi_qc_complete → sense translation via ata_to_sense_error → scsi_done` semantic equivalence: per-SAT (SCSI/ATA Translation) SAT-5 spec for INQUIRY/READ_CAPACITY/MODE_SENSE/READ/WRITE/SYNCHRONIZE_CACHE/WRITE_SAME/UNMAP and per-`Documentation/scsi/scsi.rst` for sense data formats.

## Hardening

(Inherits row-1 features from `drivers/ata/00-overview.md` § Hardening.)

libata-scsi reinforcement:

- **Per-`ap->lock` IRQ-save around `queuecmd`** — defense against per-IRQ-completion racing CDB dispatch.
- **Per-`ata_port_eh_scheduled` → SCSI_MLQUEUE_DEVICE_BUSY** — defense against per-issue-during-EH that would race with EH-owned qcs.
- **Per-`__ata_scsi_find_dev` channel/lun reject** — defense against per-out-of-range LUN/channel referencing unallocated `pmp_link` (UB / OOB read).
- **Per-`ata_scsi_rbuf_lock` global rbuf serialization** — defense against per-concurrent-INQUIRY tearing the shared response buffer.
- **Per-`cmd_len ≤ dev->cdb_len`** — defense against per-oversize-CDB reading beyond CDB bounds.
- **Per-`COMMAND_SIZE(scsi_op) ≤ scmd->cmd_len` for ATAPI** — defense against per-undersized-CDB SG_IO injecting truncated cmds.
- **Per-`ata_check_nblocks` queue-limits cross-check** — defense against per-block-layer-bypass (SG_IO) requesting too-large transfers.
- **Per-`SECURITY_PROTOCOL_*` gated on `ATA_DFLAG_TRUSTED`** — defense against per-issuing-TCG-cmd to non-trusted device.
- **Per-`WRITE_SAME_16` requires UNMAP + TRIM + non-passthrough** — defense against per-SG_IO crafting malicious DATA-OUT that becomes TRIM (data loss).
- **Per-`pass_thru` protocol map (`ata_scsi_map_proto`) rejects unknown** — defense against per-arbitrary-protocol injection (e.g., wrong DMA dir).
- **Per-`tf.device` master/slave enforcement** — defense against per-cross-device passthru cmd.
- **Per-`ata_to_sense_error` total mapping** — defense against per-zero-init garbage sense (always overwrites sk/asc/ascq).
- **Per-`gen_passthru_sense` RTF-required gate** — defense against per-reading-uninitialized-result_tf on missing-RTF.
- **Per-`ata_dev_disabled` and `ata_id_is_locked` sense paths** — defense against per-stuck-locked-drive partition-scan loops (returns clear sense).
- **Per-`ata_scsi_dma_need_drain` for ATA_16 passthru** — defense against per-DMA-overshoot when device returns fewer bytes than CDB requested.
- **Per-`device_link_add` PM ordering (sdev consumer of ap supplier)** — defense against per-suspend-out-of-order resulting in sdev I/O to a suspended port.
- **Per-`deferred_qc` single-slot policy** — defense against per-defer-queue-runaway when LLD keeps deferring.
- **Per-`ata_scsi_hotplug` + `dev_rescan` delayed_work cancel on detach** — defense against per-detach-with-pending-rescan UAF.
- **Per-`ata_scsi_user_scan` validated channel/id/lun** — defense against per-/sys/.../scan invalid input crashing find_dev.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on every SG_IO CDB ingress + DATA-IN/OUT staging.
- **PAX_KERNEXEC** — W^X enforcement on the SCSI mid-layer dispatch + libata-scsi `queuecmd` callback.
- **PAX_RANDKSTACK** — kernel-stack randomization on SCSI command-completion entry.
- **PAX_REFCOUNT** — saturating refcount on `scsi_device` ↔ `ata_port` link and `deferred_qc` slot.
- **PAX_MEMORY_SANITIZE** — zero-on-free for INQUIRY/VPD/MODE-SENSE response buffers (may carry serial numbers, encryption status, SED state) and sense buffer.
- **PAX_UDEREF** — SMAP/SMEP enforcement on every SG_IO `iov` and bsg ingress.
- **PAX_RAP / kCFI** — `scsi_host_template` + `ata_port_operations` indirect calls hardened; `static const` for every libata-scsi template.
- **GRKERNSEC_HIDESYM** — kernel-pointer hiding in sysfs `/sys/class/scsi_*` and dmesg.
- **GRKERNSEC_DMESG** — syslog restriction on libata-scsi error sense logs (which expose LBA + opcode).
- **GRKERNSEC_KMOD** — denies opportunistic LLD module load.
- **CAP_SYS_RAWIO strict + LSM `security_file_ioctl`** — every SG_IO / ATA_PASSTHROUGH / `WRITE_SAME_16` / `SECURITY_PROTOCOL_*` gated; GR-RBAC denies grant.
- **PAX_CONSTIFY_PLUGIN** — `ata_scsi_pass_thru_*` vtables + `scsi_host_template` `static const`.
- **GRKERNSEC_SYSFS_RESTRICT** — `/sys/.../scan` write restricted; `ata_scsi_user_scan` LSM-checked.
- **PAX_SIZE_OVERFLOW** — `cmd_len`, `nblocks`, `scmd->cmd_len` arithmetic checked.
- **LSM `security_capable(CAP_SYS_RAWIO)`** verified on every passthrough entry.
- **LSM `security_inode_permission`** on `/dev/sgN` and `/dev/bsg/*`.

Per-doc rationale: libata-scsi is the SG_IO + ATA_PASSTHROUGH bridge — the same set of commands but funneled through SCSI semantics, which broadens the attack surface (TRIM, SECURITY_PROTOCOL, WRITE_SAME). PAX_USERCOPY + UDEREF gate CDB+payload ingress, CAP_SYS_RAWIO + LSM gates the passthrough entry, PAX_RAP locks the `scsi_host_template` vtable that every SG_IO indirects through, and PAX_MEMORY_SANITIZE wipes INQUIRY/VPD pages that carry sensitive device identity.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `libata-core.c` core dispatch + lifetime (covered in `libata-core.md` Tier-3)
- `libata-eh.c` error handler internals (covered in dedicated Tier-3 when expanded; this doc only invokes `ata_port_schedule_eh` / `ata_qc_schedule_eh`)
- `libata-pmp.c` Port Multiplier (covered in `libata-pmp.md` Tier-3)
- `libata-zpodd.c` Zero-Power Optical (covered in `libata-zpodd.md` Tier-3)
- `libata-transport.c` sysfs transport class (covered in `libata-transport.md` Tier-3)
- SCSI midlayer (`drivers/scsi/scsi.c`, `drivers/scsi/sd.c`, `drivers/scsi/sr.c`) — covered separately under `drivers/scsi/`
- Block layer (`block/blk-mq.c`, etc.) — covered under `block/`
- ZBC zone management semantics — covered in `libata-zac.md` when expanded
- Implementation code
