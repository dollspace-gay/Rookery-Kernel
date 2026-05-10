# Tier-3: drivers/ata/libata-core.c — libata core (ATA/SATA/PATA)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/ata/00-overview.md
upstream-paths:
  - drivers/ata/libata-core.c (~6915 lines)
  - include/linux/libata.h
  - include/linux/ata.h
-->

## Summary

`libata-core.c` is the heart of the Linux ATA/SATA/PATA stack. It owns the **per-host / per-port / per-link / per-device** object graph (`struct ata_host`, `struct ata_port`, `struct ata_link`, `struct ata_device`), drives the **probe / identify / configure** pipeline, builds **taskfile (TF) register images** from block-layer R/W requests, dispatches queued commands (`struct ata_queued_cmd` aka `qc`) through low-level driver (LLD) `qc_prep` + `qc_issue` ops, and provides the **completion + EH (error-handler) handoff** path that all SATA/PATA controllers (AHCI, sata_mv, sata_sil24, every pata_*) inherit. Per-device: DMA-mode negotiation (PIO0..6, MWDMA0..2, UDMA0..6), LBA28/LBA48 selection, NCQ + FUA + TRIM + write-cache + read-look-ahead + ZAC + CDL feature gating from IDENTIFY DEVICE. Per-port: power-management (suspend/resume/freeze/thaw/poweroff/restore for S3/S4 hibernation), hotplug + rescan, port-multiplier fanout, async port probe. Per-host: devres-bound lifetime, IRQ wiring via `ata_host_activate`, SCSI host registration via `ata_scsi_add_hosts`, polling-only mode fallback. Critical for: every `/dev/sd<X>` ATA disk visible through SCSI bridge, every AHCI controller boot path, hibernation correctness, hotplug response.

This Tier-3 covers `drivers/ata/libata-core.c` (~6915 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ata_host` | per-controller container | `AtaHost` |
| `struct ata_port` | per-port (one cable / channel) | `AtaPort` |
| `struct ata_link` | per-PHY (host or PMP-fanout) | `AtaLink` |
| `struct ata_device` | per-device (master/slave/PMP-port) | `AtaDevice` |
| `struct ata_queued_cmd` | per-command (`qc`) | `AtaQueuedCmd` |
| `struct ata_taskfile` | per-command TF register image | `AtaTaskfile` |
| `struct ata_port_operations` | per-LLD vtable | `AtaPortOps` |
| `struct ata_port_info` | per-port template | `AtaPortInfo` |
| `ata_host_alloc()` | per-host alloc | `AtaHost::alloc` |
| `ata_host_alloc_pinfo()` | per-host alloc + pinfo | `AtaHost::alloc_pinfo` |
| `ata_host_start()` | per-host start (port_start + freeze) | `AtaHost::start` |
| `ata_host_register()` | per-host SCSI + async probe | `AtaHost::register` |
| `ata_host_activate()` | per-host start + IRQ + register | `AtaHost::activate` |
| `ata_host_detach()` | per-host detach | `AtaHost::detach` |
| `ata_host_suspend()` / `ata_host_resume()` | per-host PM | `AtaHost::suspend` / `resume` |
| `ata_port_alloc()` / `ata_port_free()` | per-port alloc/free | `AtaPort::alloc` / `free` |
| `ata_port_probe()` | per-port probe trigger | `AtaPort::probe` |
| `ata_port_detach()` | per-port detach | `AtaPort::detach` |
| `ata_link_init()` | per-link init | `AtaLink::init` |
| `ata_link_next()` / `ata_dev_next()` | per-iteration | `AtaLink::next` / `AtaDevice::next` |
| `ata_dev_init()` | per-device init | `AtaDevice::init` |
| `ata_dev_configure()` | per-device configure pipeline | `AtaDevice::configure` |
| `ata_dev_read_id()` | per-IDENTIFY-DEVICE | `AtaDevice::read_id` |
| `ata_dev_classify()` | per-signature classify | `AtaDevice::classify` |
| `ata_id_xfermask()` | per-IDENTIFY xfer-mask | `AtaDevice::id_xfermask` |
| `ata_dev_set_xfermode()` | per-SET FEATURES xfer-mode | `AtaDevice::set_xfermode` |
| `ata_dev_set_feature()` | per-SET FEATURES generic | `AtaDevice::set_feature` |
| `ata_dev_config_ncq()` | per-NCQ feature config | `AtaDevice::config_ncq` |
| `ata_dev_config_lba()` / `_chs()` | per-LBA48/CHS sel | `AtaDevice::config_lba` / `config_chs` |
| `ata_dev_config_lpm()` | per-link power mgmt | `AtaDevice::config_lpm` |
| `ata_dev_config_fua()` | per-FUA | `AtaDevice::config_fua` |
| `ata_dev_config_devslp()` | per-DevSlp | `AtaDevice::config_devslp` |
| `ata_dev_config_zac()` | per-ZAC | `AtaDevice::config_zac` |
| `ata_dev_config_trusted()` | per-Trusted Computing | `AtaDevice::config_trusted` |
| `ata_dev_config_cdl()` | per-CDL | `AtaDevice::config_cdl` |
| `ata_dev_xfermask()` / `ata_down_xfermask_limit()` | per-mode neg | `AtaDevice::xfermask` / `down_xfermask_limit` |
| `ata_set_mode()` | per-link mode set | `AtaLink::set_mode` |
| `ata_build_rw_tf()` | per-R/W TF build | `AtaQueuedCmd::build_rw_tf` |
| `ata_qc_issue()` | per-qc issue path | `AtaQueuedCmd::issue` |
| `ata_qc_complete()` | per-qc completion | `AtaQueuedCmd::complete` |
| `__ata_qc_complete()` | per-qc raw completion | `AtaQueuedCmd::complete_inner` |
| `ata_qc_free()` | per-qc free | `AtaQueuedCmd::free` |
| `ata_qc_get_active()` | per-port active bitmap | `AtaPort::qc_get_active` |
| `ata_exec_internal()` | per-internal-cmd (probe/EH) | `AtaDevice::exec_internal` |
| `ata_sg_init()` / `ata_sg_setup()` / `ata_sg_clean()` | per-qc DMA scatter-gather | `AtaQueuedCmd::sg_*` |
| `ata_pack_xfermask()` / `ata_unpack_xfermask()` | per-mask helpers | `AtaXfer::pack` / `unpack` |
| `ata_xfer_mask2mode()` / `_mode2mask()` / `_mode2shift()` | per-mask/mode conv | `AtaXfer::*` |
| `ata_std_qc_defer()` | per-NCQ defer policy | `AtaPortOps::std_qc_defer` |
| `ata_wait_ready()` / `ata_wait_after_reset()` | per-BSY-drop wait | `AtaLink::wait_ready` / `wait_after_reset` |
| `ata_std_prereset()` / `ata_std_postreset()` | per-reset stubs | `AtaLink::std_prereset` / `std_postreset` |
| `sata_print_link_status()` | per-SATA SSTATUS print | `AtaLink::print_status` |
| `sata_link_init_spd()` | per-SCR_CONTROL spd init | `AtaLink::init_spd` |
| `ata_dev_power_set_active()` / `_set_standby()` | per-dev PM TF | `AtaDevice::power_*` |
| `ata_port_request_pm()` / `ata_port_suspend()` / `ata_port_resume()` | per-port PM via EH | `AtaPort::*_pm` |
| `ata_sas_port_suspend()` / `ata_sas_port_resume()` | per-SAS-shared PM | `AtaPort::sas_*` |
| `ata_pci_remove_one()` / `_shutdown_one()` | per-PCI exit | `AtaPci::remove_one` / `shutdown_one` |
| `ata_pci_device_suspend()` / `_resume()` | per-PCI PM | `AtaPci::dev_suspend` / `dev_resume` |
| `ata_platform_remove_one()` | per-platform exit | `AtaPlatform::remove_one` |
| `ata_dummy_port_ops` / `ata_dummy_port_info` | per-dummy port | shared singleton |
| `ata_base_port_ops` | per-base port-ops | `AtaPortOps::BASE` |
| `ata_msleep()` / `ata_ratelimit()` / `ata_wait_register()` | per-helpers | `Ata::msleep` / `ratelimit` / `wait_register` |
| `EXPORT_SYMBOL_GPL(ata_*)` | per-LLD ABI surface | `extern fn` |

## Compatibility contract

REQ-1: `struct ata_host` lifetime:
- Allocated via `ata_host_alloc(dev, n_ports)` — `kzalloc(sizeof(ata_host) + n_ports * sizeof(void *))`.
- Bound to parent `struct device` via `devres_open_group` + `devres_add(ata_devres_release)`.
- `spin_lock_init(&host->lock)`, `mutex_init(&host->eh_mutex)`, `kref_init(&host->kref)`.
- For each port slot: `ata_port_alloc(host)` and `host->ports[i] = ap`.
- `dev_set_drvdata(dev, host)`.
- On failure: `devres_release_group(dev, NULL)`.

REQ-2: `ata_host_alloc_pinfo(dev, ppi, n_ports)`:
- Calls `ata_host_alloc`.
- For each port `i`, picks `pi = ppi[j++]` until NULL-terminator, last-pinfo for remainder.
- Copies `pi->pio_mask`, `pi->mwdma_mask`, `pi->udma_mask`, `pi->flags`, `pi->link_flags`, `pi->port_ops` into `ap`.
- Picks `host->ops` from first non-`ata_dummy_port_ops`.

REQ-3: `ata_host_start(host)`:
- Early-return if `ATA_HOST_STARTED`.
- `ata_finalize_port_ops(host->ops)` — flatten `->inherits` chain into a single table; NULLs resolved against ancestor; `ATA_OP_NULL` forces NULL; per-table mutex `lock`.
- For each port: `ata_finalize_port_ops(ap->ops)`; `port_start(ap)` if non-NULL; `ata_eh_freeze_port(ap)`.
- Allocates `devres(ata_host_stop, ...)` to schedule `host_stop` + `port_stop` on devres release.
- Sets `ATA_HOST_STARTED`.

REQ-4: `ata_host_register(host, sht)`:
- Pre: `host->flags & ATA_HOST_STARTED`.
- `host->n_tags = clamp(sht->can_queue, 1, ATA_MAX_QUEUE)`.
- `ata_tport_add(host->dev, host->ports[i])` for each port (sysfs transport class).
- `ata_scsi_add_hosts(host, sht)` → allocates `Scsi_Host` per port.
- Per port: set `ap->cbl = ATA_CBL_SATA` if SATA + unset; `sata_link_init_spd(&ap->link)`; print "%cATA max %s %s".
- Per port: `ap->cookie = async_schedule(async_port_probe, ap)` — probe is asynchronous.
- On error: `ata_tport_delete` rollback.

REQ-5: `ata_host_activate(host, irq, irq_handler, irq_flags, sht)`:
- `ata_host_start(host)`.
- If `irq == 0`: polling — `ata_host_register(host, sht)` only.
- Else `devm_request_irq(host->dev, irq, irq_handler, irq_flags, irq_desc, host)`.
- Per port: `ata_port_desc_misc(ap, irq)` to record IRQ in port desc.
- `ata_host_register(host, sht)`. If fail: `devm_free_irq`.

REQ-6: `struct ata_port` alloc (`ata_port_alloc`):
- `kzalloc_obj(*ap)`.
- `ap->pflags |= ATA_PFLAG_INITIALIZING | ATA_PFLAG_FROZEN`.
- `ap->lock = &host->lock` (per-host lock shared by ports unless LLD overrides).
- `ap->print_id = ida_alloc_min(&ata_ida, 1, GFP_KERNEL)` → monotonic `ataN`.
- `ap->host = host`, `ap->dev = host->dev`.
- `mutex_init(&ap->scsi_scan_mutex)`.
- `INIT_DELAYED_WORK(&ap->hotplug_task, ata_scsi_hotplug)`.
- `INIT_DELAYED_WORK(&ap->scsi_rescan_task, ata_scsi_dev_rescan)`.
- `INIT_WORK(&ap->deferred_qc_work, ata_scsi_deferred_qc_work)`.
- `timer_setup(&ap->fastdrain_timer, ata_eh_fastdrain_timerfn, 0)`.
- `ata_link_init(ap, &ap->link, 0)` — primary link, PMP=0.

REQ-7: `struct ata_link` init (`ata_link_init(ap, link, pmp)`):
- `memset` from `ATA_LINK_CLEAR_BEGIN` to `ATA_LINK_CLEAR_END` (preserves device array).
- `link->ap = ap`; `link->pmp = pmp`.
- `link->active_tag = ATA_TAG_POISON`.
- `link->hw_sata_spd_limit = UINT_MAX`.
- For each `i in 0..ATA_MAX_DEVICES`: `link->device[i].link = link`; `link->device[i].devno = i`; `ata_dev_init(&link->device[i])`.

REQ-8: `struct ata_device` init (`ata_dev_init`):
- `link->sata_spd_limit = link->hw_sata_spd_limit`; `link->sata_spd = 0`.
- Under `ap->lock`: `dev->flags &= ~ATA_DFLAG_INIT_MASK`; `dev->quirks = 0`.
- `memset` from `ATA_DEVICE_CLEAR_BEGIN` to `ATA_DEVICE_CLEAR_END`.
- `dev->pio_mask = UINT_MAX`; `dev->mwdma_mask = UINT_MAX`; `dev->udma_mask = UINT_MAX`.

REQ-9: `ata_dev_read_id(dev, *p_class, flags, id)`:
- /* Issue IDENTIFY DEVICE (`ATA_CMD_ID_ATA`) or IDENTIFY PACKET DEVICE (`ATA_CMD_ID_ATAPI`) per class */
- `ap->ops->dev_select(ap, dev->devno)`.
- Build TF; `ata_exec_internal(dev, &tf, NULL, DMA_FROM_DEVICE, id, ATA_ID_WORDS * 2, deadline)`.
- Per-`swap_buf_le16(id, ATA_ID_WORDS)` on big-endian.
- Validate checksum (word 255 = 0xA5 + sum-low-byte).
- Reclassify if needed (CFA vs ATA, ATAPI vs ATA).

REQ-10: `ata_dev_classify(tf)`:
- Match `(tf->lbam, tf->lbah)` signature:
  - `(0x14, 0xeb)` → `ATA_DEV_ATAPI`.
  - `(0x69, 0x96)` → `ATA_DEV_PMP`.
  - `(0x3c, 0xc3)` → `ATA_DEV_SEMB`.
  - `(0xcd, 0xab)` → `ATA_DEV_ZAC`.
  - `(0, 0)` if `nsect == 1 ∧ lbal == 1 ∧ status nonzero` → `ATA_DEV_ATA`.
- Else `ATA_DEV_UNKNOWN` or `ATA_DEV_NONE`.

REQ-11: `ata_dev_configure(dev)`:
- Pre: `dev` enabled, `dev->id` populated by `read_id`.
- `ata_clear_log_directory(dev)`.
- `dev->quirks |= ata_dev_quirks(dev)`; `ata_force_quirks(dev)`.
- If `ATA_QUIRK_DISABLE`: `ata_dev_disable(dev)`; return 0.
- If ATAPI and disabled by `atapi_enabled` or `ATA_FLAG_NO_ATAPI`: disable.
- `ata_do_link_spd_quirk(dev)`; `ata_acpi_on_devcfg(dev)`; `ata_hpa_resize(dev)` — early since HPA changes IDENTIFY.
- Clear to-be-cfg: `dev->flags &= ~ATA_DFLAG_CFG_MASK`; `max_sectors=0, cdb_len=0, n_sectors=0, cylinders=0, heads=0, sectors=0, multi_count=0`.
- `xfer_mask = ata_id_xfermask(id)`.
- ATA / ZAC class:
  - Read major version → `revbuf "ATA-N"` or `"CFA"`.
  - `dev->n_sectors = ata_id_n_sectors(id)`; if `ata_id_is_locked(id)` set `n_sectors=0`.
  - R/W Multiple count: parse `id[47]/id[59]` (powers-of-2).
  - If `ata_id_has_lba(id)`: `ata_dev_config_lba(dev)` (sets LBA48 + n_sectors). Else `ata_dev_config_chs(dev)`.
  - `ata_dev_config_lpm`, `ata_dev_config_fua`, `ata_dev_config_devslp`, `ata_dev_config_sense_reporting`, `ata_dev_config_zac`, `ata_dev_config_trusted`, `ata_dev_config_cpr`, `ata_dev_config_cdl`.
  - `dev->cdb_len = 32`.
- ATAPI class:
  - `dev->cdb_len = atapi_cdb_len(id)` (must be 12 ≤ x ≤ 16).
  - Optional ATAPI AN: `ata_dev_set_feature(dev, SETFEATURES_SATA_ENABLE, SATA_AN)`.
  - CDB IRQ / DMADIR / DA flags from id.
  - `ata_dev_config_lpm(dev)`.
- `dev->max_sectors = ATA_MAX_SECTORS` (or `ATA_MAX_SECTORS_LBA48` for LBA48).
- Knobble (PATA-on-SATA bridge): `udma_mask &= ATA_UDMA5`; `max_sectors = ATA_MAX_SECTORS`.
- Apply `ATA_QUIRK_MAX_SEC` / `ATA_QUIRK_MAX_SEC_LBA48` / `ATA_QUIRK_STUCK_ERR`.
- If `ap->ops->dev_config`: call LLD hook.
- Warn on `ATA_QUIRK_DIAGNOSTIC` / `ATA_QUIRK_FIRMWARE_WARN`.

REQ-12: `ata_dev_config_ncq(dev, desc)`:
- Requires `ATA_FLAG_NCQ`, `ata_id_has_ncq(id)`, hardware queue depth ≥ 2.
- Compute `dev->n_native_xfer = min(ATA_MAX_QUEUE, hw_depth)`.
- If `ata_id_has_ncq_send_recv(id)`: `ata_dev_config_ncq_send_recv(dev)`.
- If `ata_id_has_ncq_non_data(id)`: `ata_dev_config_ncq_non_data(dev)`.
- NCQ-priority: `ata_dev_config_ncq_prio(dev)` when supported and host-`ATA_FLAG_NCQ_PRIO`.

REQ-13: `ata_dev_xfermask(dev)`:
- Start: `dev->pio_mask = pi_mask ∩ ap->pio_mask`; same for MWDMA/UDMA.
- Apply forced via `ata_force_xfermask(dev)`.
- Per-`cable_is_40wire(ap)`: clamp UDMA to UDMA2 (no UDMA3+).
- Per-quirk: `ATA_QUIRK_NODMA`, `ATA_QUIRK_NO_NCQ`, `ATA_QUIRK_NOLPM`, etc.

REQ-14: `ata_set_mode(link, r_failed_dev)`:
- For each enabled dev: `ata_dev_set_xfermode(dev)` — issues `ATA_CMD_SET_FEATURES` subcmd `SETFEATURES_XFER` value `dev->xfer_mode`.
- On success: `dev->xfer_shift` recorded; `ata_dev_set_mode(dev)` for follow-up (e.g., re-IDENTIFY).
- If LLD overrides `ap->ops->set_mode`: use it; else default.

REQ-15: `ata_build_rw_tf(qc, block, n_block, tf_flags, cdl, class)`:
- Sets `tf->flags |= ATA_TFLAG_ISADDR | ATA_TFLAG_DEVICE | tf_flags`.
- NCQ path (`ata_ncq_enabled(dev)`): require `lba_48_ok(block, n_block)` → -ERANGE if not.
  - `tf->protocol = ATA_PROT_NCQ`; `tf->flags |= ATA_TFLAG_LBA | ATA_TFLAG_LBA48`.
  - `tf->command = ATA_CMD_FPDMA_WRITE` or `ATA_CMD_FPDMA_READ`.
  - `tf->nsect = qc->hw_tag << 3` (tag in upper bits).
  - `hob_feature = (n_block >> 8) & 0xff`, `feature = n_block & 0xff` (sector count).
  - Pack LBA across (hob_)lba{h,m,l}.
  - `tf->device = ATA_LBA`; FUA → `|= 1 << 7`.
  - NCQ priority if `IOPRIO_CLASS_RT`; CDL set via `ata_set_tf_cdl(qc, cdl)`.
- LBA path (`ATA_DFLAG_LBA`):
  - LBA28 if no FUA, no CDL, and `lba_28_ok(block, n_block)`.
  - LBA48 if `lba_48_ok` and `ATA_DFLAG_LBA48` set; require LBA48 capable.
  - Else -ERANGE.
  - `ata_set_rwcmd_protocol(dev, tf)` picks PIO/DMA/READ/WRITE/multi-sector cmd from `ata_rw_cmds[]`.
- CHS path: only for `lba_28_ok`; translate via `dev->sectors, dev->heads`; bounds check.

REQ-16: `ata_qc_issue(qc)`:
- Pre: `ata_tag_valid(qc->tag)` ∧ `!ata_tag_valid(link->active_tag)` (non-NCQ exclusive).
- NCQ protocol: `link->sactive |= 1 << qc->hw_tag`; if first NCQ: `ap->nr_active_links++`.
- Non-NCQ: `link->active_tag = qc->tag`; `ap->nr_active_links++`.
- `qc->flags |= ATA_QCFLAG_ACTIVE`; `ap->qc_active |= 1ULL << qc->tag`.
- If `!ata_adapter_is_online(ap)`: `err_mask |= AC_ERR_HOST_BUS`; `goto sys_err`.
- Data cmd: require `qc->sg ∧ qc->n_elem ∧ qc->nbytes`.
- DMA cmd (or PIO+`ATA_FLAG_PIO_DMA`): `ata_sg_setup(qc)` to dma_map_sg.
- If `ATA_DFLAG_SLEEPING`: schedule reset via `ATA_EH_RESET`, `ata_link_abort(link)`; return.
- If `ap->ops->qc_prep`: `qc->err_mask |= ap->ops->qc_prep(qc)` (e.g., AHCI CFIS build).
- `qc->err_mask |= ap->ops->qc_issue(qc)` — LLD writes TF and arms DMA/PIO.
- On error path: `ata_qc_complete(qc)` with `AC_ERR_SYSTEM`.

REQ-17: `ata_qc_complete(qc)`:
- LED activity via `ledtrig_disk_activity(WRITE?)`.
- Internal-tag fast path: `fill_result_tf(qc)`; `__ata_qc_complete(qc)`; return.
- If `qc->err_mask`: set `ATA_QCFLAG_EH`; `fill_result_tf`; `ata_qc_schedule_eh(qc)`; return.
- WARN if `ata_port_is_frozen(ap)`.
- If `ATA_QCFLAG_RESULT_TF`: `fill_result_tf`.
- CDL success-with-SENSE: force EH via `ATA_EH_GET_SUCCESS_SENSE` + `ATA_PFLAG_EH_PENDING`.
- Post-success processing per `tf.command`:
  - `ATA_CMD_SET_FEATURES` (WC/RA) → `ATA_EH_REVALIDATE` + schedule EH.
  - `ATA_CMD_INIT_DEV_PARAMS`, `ATA_CMD_SET_MULTI` → revalidate.
  - `ATA_CMD_SLEEP` → `dev->flags |= ATA_DFLAG_SLEEPING`.
- `ata_verify_xfer(qc)` may clear `ATA_DFLAG_DUBIOUS_XFER`.
- `__ata_qc_complete(qc)` → clears `ATA_QCFLAG_ACTIVE`, calls `qc->complete_fn`.

REQ-18: `__ata_qc_complete(qc)`:
- Clears `sactive` / `active_tag`; decrements `ap->nr_active_links` if last.
- Clears `ATA_QCFLAG_CLEAR_EXCL` exclusive-link.
- `qc->flags &= ~ATA_QCFLAG_ACTIVE`; `ap->qc_active &= ~(1ULL << qc->tag)`.
- Invokes `qc->complete_fn(qc)` (typically `ata_scsi_qc_complete`).

REQ-19: `ata_exec_internal(dev, tf, cdb, dma_dir, buf, buflen, deadline)`:
- Used during probe and EH only (tag = `ATA_TAG_INTERNAL`).
- Allocates fixed slot; serializes via `ata_eh_release` / `ata_eh_acquire` around `ata_qc_issue`.
- Waits with `wait_for_completion_timeout`; on timeout aborts qc.
- Returns `err_mask`.

REQ-20: SATA OOB + SCR (System Control Register) access:
- `sata_scr_read(link, reg, *val)`: dispatch via `ap->ops->scr_read`; reg ∈ {SCR_STATUS, SCR_ERROR, SCR_CONTROL, SCR_ACTIVE, SCR_NOTIFICATION}.
- `sata_link_init_spd(link)`: reads `SCR_CONTROL`, extracts current SPD (bits 4-7), narrows `hw_sata_spd_limit`.
- `sata_print_link_status(link)`: dmesg `"SATA link up Y.Z Gbps (SStatus X SControl Y)"` or "link down".
- Link state via `SCR_STATUS`: `ata_sstatus_online()` per bits[3:0] == 0x3 (PHY ready).

REQ-21: Reset hooks (LLD-overridable):
- `ata_std_prereset(link, deadline)`: minimal — `sata_link_resume(link, ...)` if SATA.
- `ata_std_postreset(link, classes)`: reset SCR_ERROR; clear SError.
- `ata_wait_ready(link, deadline, check_ready)`: poll until BSY clears.
- `ata_wait_after_reset(link, deadline, check_ready)`: sleep 150ms then `ata_wait_ready`.

REQ-22: Port iteration:
- `ata_link_next(link, ap, mode)` per ATA_LITER_{HOST_FIRST, PMP_FIRST, EDGE} — iterates `&ap->link` then `&ap->pmp_link[0..nr_pmp_links]`.
- `ata_dev_next(dev, link, mode)` per ATA_DITER_{ENABLED, ALL, ENABLED_REVERSE, ALL_REVERSE}.
- `ata_dev_phys_link(dev)`: physical (host) link of a possibly-PMP'd device.

REQ-23: Port power management:
- `ata_port_pm_ops` per `dev_pm_ops`:
  - `.suspend = ata_port_pm_suspend` (PMSG_SUSPEND).
  - `.freeze = ata_port_pm_freeze` (PMSG_FREEZE).
  - `.poweroff = ata_port_pm_poweroff` (PMSG_HIBERNATE — S4).
  - `.resume = ata_port_pm_resume` (PMSG_RESUME).
  - `.thaw / .restore = ata_port_pm_resume`.
  - `.runtime_suspend / _resume / _idle` for autosuspend.
- `ata_port_request_pm(ap, mesg, action, ehi_flags, async)`:
  - Wait for prior `ATA_PFLAG_PM_PENDING` to clear.
  - Set `ap->pm_mesg = mesg`; `ap->pflags |= ATA_PFLAG_PM_PENDING`.
  - For each link: `eh_info.action |= action`; `eh_info.flags |= ehi_flags`.
  - `ata_port_schedule_eh(ap)` — actual work runs in EH thread.
  - If sync: `ata_port_wait_eh(ap)`.
- Suspend uses `ATA_EHI_QUIET | ATA_EHI_NO_AUTOPSY | ATA_EHI_NO_RECOVERY`.
- Resume uses `ATA_EH_RESET | ATA_EHI_NO_AUTOPSY | ATA_EHI_QUIET`.
- Runtime-idle returns `-EBUSY` if any ATAPI ODD without ZPODD attached (avoids constant reset).

REQ-24: Host suspend/resume:
- `ata_host_suspend(host, mesg)`: just records `host->dev->power.power_state = mesg`.
- `ata_host_resume(host)`: sets `power_state = PMSG_ON`. Real work in per-port `dev_pm_ops`.

REQ-25: SCSI host registration entry point:
- `ata_scsi_add_hosts(host, sht)` — called from `ata_host_register`.
- Per port: `scsi_host_alloc(sht, sizeof(struct ata_port *))`.
- `shost->eh_noresume = 1`; `hostdata[0] = ap`; `ap->scsi_host = shost`.
- `shost->transportt = &ata_scsi_transportt`; `max_id=16, max_lun=1, max_channel=1, max_cmd_len=32`.
- `shost->max_host_blocked = 1` (libata does its own deferral).
- `scsi_add_host_with_dma(shost, &ap->tdev, host->dev)`.

REQ-26: Async port probe:
- `async_port_probe(ap, cookie)` invokes EH-based probe via `ATA_EH_RESET` action.
- EH calls reset → IDENTIFY → `ata_dev_configure` → `ata_scsi_scan_host(ap, 1)` to instantiate `/dev/sd<X>`.

REQ-27: `ata_dev_set_feature(dev, subcmd, action)`:
- Issues `ATA_CMD_SET_FEATURES` via `ata_exec_internal` with `tf.feature = subcmd`, `tf.nsect = action`, `tf.protocol = ATA_PROT_NODATA`.

REQ-28: `ata_dev_power_set_active(dev)` / `_set_standby(dev)`:
- Builds VERIFY (LBA0, 1 sector) for active, or `ATA_CMD_STANDBYNOW1` for standby.
- Submitted via internal-cmd; clears/sets `ATA_DFLAG_PLL` etc.
- `ata_dev_power_init_tf` returns whether device is power-managed.

REQ-29: Port detach + host detach:
- `ata_port_detach(ap)`: sets `ATA_PFLAG_UNLOADING`, `ata_port_schedule_eh(ap)`, waits for EH; `cancel_delayed_work_sync` on hotplug/rescan/deferred-qc; `ata_tport_delete(ap)`.
- `ata_host_detach(host)`: per-port `ata_port_detach`; `kref_put(&host->kref, ata_host_release)`.
- `ata_dev_free_resources(dev)`: cleans CDL log buffer and any per-device alloc.

REQ-30: PCI shim:
- `ata_pci_remove_one(pdev)`: `ata_host_detach(dev_get_drvdata(&pdev->dev))`.
- `ata_pci_shutdown_one(pdev)`: per-port disable + freeze (shutdown path).
- `ata_pci_device_do_suspend / _do_resume`: low-level PCI save/restore.
- `ata_pci_device_suspend / _resume`: full suspend including `ata_host_suspend`.
- `pci_test_config_bits(pdev, bits)`: utility for legacy IDE probe.

REQ-31: Force parameters (cmdline `libata.force=...`):
- `ata_parse_force_param()` populates `ata_force_tbl[]` from `ata_force_param_buf` (e.g., `1.00:udma5,nodma`).
- `ata_force_pflags`, `ata_force_link_limits`, `ata_force_xfermask`, `ata_force_quirks`, `ata_force_cbl` consume entries.

REQ-32: Dummy port:
- `ata_dummy_port_ops` + `ata_dummy_port_info` for ports that are physically absent — `qc_issue` returns `AC_ERR_SYSTEM`, `error_handler` is no-op.

## Acceptance Criteria

- [ ] AC-1: `ata_host_alloc(dev, 2)` returns `ata_host*` with `n_ports==2`, `ports[0..1]` non-NULL, `dev_get_drvdata(dev) == host`.
- [ ] AC-2: `ata_host_alloc_pinfo` copies `pio_mask/mwdma_mask/udma_mask/flags/port_ops` from pinfo into ports, with NULL-terminator fallback.
- [ ] AC-3: `ata_host_start` is idempotent — second call early-returns.
- [ ] AC-4: `ata_host_activate` with `irq=0` does not call `request_irq`.
- [ ] AC-5: `ata_host_register` schedules `async_port_probe` per port.
- [ ] AC-6: `ata_dev_read_id` IDENTIFY DEVICE on ATA disk returns 256 words; checksum validated.
- [ ] AC-7: `ata_dev_classify((0x14, 0xeb))` → `ATA_DEV_ATAPI`.
- [ ] AC-8: `ata_dev_configure` on LBA48 disk sets `dev->flags & ATA_DFLAG_LBA48`, `max_sectors == ATA_MAX_SECTORS_LBA48`, `n_sectors > 0`.
- [ ] AC-9: `ata_dev_config_ncq` enables `ATA_DFLAG_NCQ_ON` when `ata_id_has_ncq(id) && ATA_FLAG_NCQ`.
- [ ] AC-10: `ata_build_rw_tf` with NCQ-enabled dev, LBA=10, n_block=8 produces `tf.command == ATA_CMD_FPDMA_READ`, `tf.protocol == ATA_PROT_NCQ`, `tf.flags & (ATA_TFLAG_LBA | ATA_TFLAG_LBA48)`.
- [ ] AC-11: `ata_qc_issue` on NCQ qc sets `link->sactive` bit for `qc->hw_tag` before invoking `qc_issue`.
- [ ] AC-12: `ata_qc_complete` with `qc->err_mask != 0` schedules EH (not direct completion).
- [ ] AC-13: `ata_port_pm_suspend` while `pm_runtime_suspended(dev)` is a no-op.
- [ ] AC-14: `ata_port_pm_poweroff` (hibernation/S4) issues `ata_port_suspend(ap, PMSG_HIBERNATE)`.
- [ ] AC-15: `ata_host_suspend(host, mesg)` updates `host->dev->power.power_state` without touching ports directly.

## Architecture

```
struct AtaHost {
  dev: *Device,
  ops: *AtaPortOps,
  lock: SpinLock,
  eh_mutex: Mutex,
  kref: Kref,
  n_ports: u32,
  n_tags: u32,
  flags: u32,                  // ATA_HOST_*
  ports: [*AtaPort; n_ports],
  private_data: *void,
}

struct AtaPort {
  host: *AtaHost,
  ops: *AtaPortOps,
  lock: *SpinLock,             // typically &host->lock
  flags: u32,                  // ATA_FLAG_*
  pflags: u32,                 // ATA_PFLAG_*
  print_id: u32,               // ata<N>
  port_no: u32,
  pio_mask: u32,
  mwdma_mask: u32,
  udma_mask: u32,
  cbl: u8,                     // ATA_CBL_*
  link: AtaLink,               // primary
  pmp_link: [AtaLink; SATA_PMP_MAX_PORTS],
  nr_pmp_links: u32,
  qc_active: u64,              // bitmap of active tags
  nr_active_links: u32,
  scsi_host: *ScsiHost,
  deferred_qc: *AtaQueuedCmd,
  fastdrain_timer: Timer,
  pm_mesg: PmMessage,
  cookie: AsyncCookie,
  dev: *Device,                // tdev / parent
  /* ... */
}

struct AtaLink {
  ap: *AtaPort,
  pmp: u32,
  flags: u32,
  active_tag: u32,
  sactive: u32,                // NCQ tag bitmap
  hw_sata_spd_limit: u32,
  sata_spd_limit: u32,
  sata_spd: u32,
  saved_scontrol: u32,
  eh_info: AtaEhInfo,
  eh_context: AtaEhContext,
  device: [AtaDevice; ATA_MAX_DEVICES],
}

struct AtaDevice {
  link: *AtaLink,
  devno: u32,
  class: u32,                  // ATA_DEV_{ATA,ATAPI,ZAC,PMP,SEMB,NONE}
  flags: u32,                  // ATA_DFLAG_*
  quirks: u64,
  id: [u16; ATA_ID_WORDS],     // IDENTIFY DEVICE
  n_sectors: u64,
  max_sectors: u32,
  cdb_len: u32,
  pio_mask: u32,
  mwdma_mask: u32,
  udma_mask: u32,
  xfer_mode: u8,
  xfer_shift: u8,
  multi_count: u32,
  /* ... */
}

struct AtaQueuedCmd {
  ap: *AtaPort,
  dev: *AtaDevice,
  scsicmd: *ScsiCmnd,
  scsidone: fn(*ScsiCmnd),
  flags: u32,                  // ATA_QCFLAG_*
  tag: u32,
  hw_tag: u8,
  tf: AtaTaskfile,             // command TF
  result_tf: AtaTaskfile,      // status / error
  err_mask: u32,
  protocol: u8,
  dma_dir: DmaDataDirection,
  sg: *Scatterlist,
  n_elem: u32,
  nbytes: u32,
  complete_fn: fn(*AtaQueuedCmd),
}

struct AtaTaskfile {
  flags: u32,                  // ATA_TFLAG_*
  protocol: u8,                // ATA_PROT_*
  ctl: u8,
  hob_feature: u8,
  hob_nsect: u8,
  hob_lbal: u8, hob_lbam: u8, hob_lbah: u8,
  feature: u8,
  nsect: u8,
  lbal: u8, lbam: u8, lbah: u8,
  device: u8,
  command: u8,
  error: u8,                   // result-only
  status: u8,                  // result-only
  auxiliary: u32,              // 32-byte CDB extension
}
```

`AtaHost::alloc(dev, n_ports) -> *AtaHost`:
1. `sz = sizeof(AtaHost) + n_ports * sizeof(*AtaPort)`.
2. `host = kzalloc(sz, GFP_KERNEL)`; if NULL → return NULL.
3. `devres_open_group(dev, NULL, GFP_KERNEL)`.
4. `dr = devres_alloc(ata_devres_release, 0, GFP_KERNEL)`; `devres_add(dev, dr)`.
5. `dev_set_drvdata(dev, host)`.
6. `spin_lock_init`, `mutex_init(&eh_mutex)`, `kref_init`.
7. `host->dev = dev`; `host->n_ports = n_ports`.
8. For i in 0..n_ports: `ap = AtaPort::alloc(host)`; if NULL → goto err_out; `ap->port_no = i`; `host->ports[i] = ap`.
9. `devres_remove_group(dev, NULL)`; return host.

`AtaHost::start(host) -> i32`:
1. If `flags & ATA_HOST_STARTED` → 0.
2. `ata_finalize_port_ops(host->ops)`.
3. For each port: `ata_finalize_port_ops(ap->ops)`; if `ops->port_start`: `rc = ops->port_start(ap)`; record `have_stop` if `ops->port_stop`.
4. If `have_stop`: `devres(ata_host_stop, 0)`.
5. `ata_eh_freeze_port(ap)` per port.
6. `host->flags |= ATA_HOST_STARTED`; return 0.

`AtaHost::register(host, sht) -> i32`:
1. Pre: `flags & ATA_HOST_STARTED` (BUG/-EINVAL otherwise).
2. `host->n_tags = clamp(sht->can_queue, 1, ATA_MAX_QUEUE)`.
3. For each port: `ata_tport_add(host->dev, ap)`.
4. `ata_scsi_add_hosts(host, sht)`.
5. Per port: set `ap->cbl = ATA_CBL_SATA` if SATA-unset; `sata_link_init_spd(&ap->link)`; `sata_link_init_spd(slave_link)` if present; compute and print xfer_mask string.
6. Per port: `ap->cookie = async_schedule(async_port_probe, ap)`.
7. Return 0.

`AtaDevice::configure(dev) -> i32`:
1. Pre: `dev` enabled, `dev->id` populated.
2. Clear log dir cache; gather quirks.
3. If `ATA_QUIRK_DISABLE`: disable + return 0.
4. ACPI hook + HPA resize.
5. Clear cfg state (`ATA_DFLAG_CFG_MASK`, max_sectors, etc.).
6. Per-class branch: ATA/ZAC → R/W-multi, LBA/CHS, LPM, FUA, DevSlp, sense-rep, ZAC, trusted, CDL; cdb_len=32. ATAPI → cdb_len, AN, CDB-intr, DMADIR, LPM.
7. Compute `max_sectors`; apply quirks; LLD `dev_config` hook.

`AtaQueuedCmd::issue(qc)`:
1. Validate tag and active state (NCQ vs non-NCQ).
2. Update `link->sactive`/`active_tag` + `ap->qc_active`.
3. Set `ATA_QCFLAG_ACTIVE`.
4. Adapter-online check; data-cmd sanity (`sg ∧ n_elem ∧ nbytes`).
5. `ata_sg_setup(qc)` for DMA.
6. SLEEPING-dev: queue ATA_EH_RESET; `ata_link_abort`; return.
7. `ops->qc_prep(qc)` if non-NULL; on err → `ata_qc_complete(AC_ERR_*)`.
8. `ops->qc_issue(qc)`.

`AtaQueuedCmd::complete(qc)`:
1. LED activity trigger.
2. Set `ATA_QCFLAG_EH` if `err_mask`.
3. Internal tag → `fill_result_tf` + `__ata_qc_complete`.
4. EH-failed → `fill_result_tf` + `ata_qc_schedule_eh`.
5. Result-TF or CDL-with-SENSE → fill + maybe schedule EH.
6. Post-success TF.command-specific handling (SET_FEATURES, SET_MULTI, SLEEP).
7. `ata_verify_xfer(qc)`.
8. `__ata_qc_complete(qc)` → `qc->complete_fn(qc)`.

`AtaPort::request_pm(ap, mesg, action, ehi_flags, async)`:
1. Lock `ap->lock`.
2. If `ATA_PFLAG_PM_PENDING`: unlock, `ata_port_wait_eh(ap)`, re-lock.
3. `ap->pm_mesg = mesg`; set `ATA_PFLAG_PM_PENDING`.
4. For each link: `eh_info.action |= action`; `eh_info.flags |= ehi_flags`.
5. `ata_port_schedule_eh(ap)`; unlock.
6. If !async: `ata_port_wait_eh(ap)`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `host_ports_array_inbounds` | INVARIANT | per-`AtaHost::alloc`: `ports[0..n_ports]` allocated and indexable; OOB indexing UB-free. |
| `port_alloc_id_unique` | INVARIANT | per-`AtaPort::alloc`: `ida_alloc_min` returns monotonic IDs; never duplicates across live ports. |
| `link_clear_preserves_devices` | INVARIANT | per-`AtaLink::init`: `memset(ATA_LINK_CLEAR_BEGIN..END)` does not touch `link->device[]`. |
| `dev_init_idempotent` | INVARIANT | per-`AtaDevice::init`: invoking twice leaves identical state. |
| `qc_issue_tag_disjoint` | INVARIANT | per-`AtaQueuedCmd::issue`: NCQ qc and non-NCQ qc never both active on same link. |
| `qc_complete_active_flag_cleared` | INVARIANT | per-`AtaQueuedCmd::complete`: `ATA_QCFLAG_ACTIVE` cleared before `complete_fn` returns. |
| `qc_active_bitmap_consistent` | INVARIANT | per-`AtaPort`: `qc_active` bit set iff some qc with that tag has `ATA_QCFLAG_ACTIVE`. |
| `build_rw_tf_range_checked` | INVARIANT | per-`ata_build_rw_tf`: returns `-ERANGE` for any block/n_block exceeding addressing capability of selected protocol. |
| `pm_pending_serialized` | INVARIANT | per-`AtaPort::request_pm`: only one PM op in flight per port (`ATA_PFLAG_PM_PENDING` is a monitor). |
| `host_started_idempotent` | INVARIANT | per-`AtaHost::start`: second invocation is a no-op. |

### Layer 2: TLA+

`drivers/ata/libata-core.tla`:
- Per-port state machine: `INITIALIZING → FROZEN → PROBED → READY ⇄ EH ⇄ SUSPENDED`.
- Per-qc lifecycle: `FREE → ALLOC → ACTIVE → (NORMAL_COMPLETE | EH_SCHEDULED) → FREE`.
- Per-PM model: suspend/resume request → EH-thread observes → completion clears `ATA_PFLAG_PM_PENDING`.
- Properties:
  - `safety_qc_no_double_active` — per-tag at most one outstanding qc.
  - `safety_ncq_excludes_non_ncq` — `link->sactive ≠ 0 ⟹ active_tag == ATA_TAG_POISON`.
  - `safety_host_started_monotone` — `ATA_HOST_STARTED` once set stays set until `host_release`.
  - `safety_pm_pending_unique` — at most one PM operation per port at a time.
  - `liveness_async_probe_terminates` — `async_port_probe` eventually completes (success or EH-disable).
  - `liveness_qc_eventually_completes` — every issued qc eventually reaches `__ata_qc_complete` (modulo permanently-frozen port → EH-reaped).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `AtaHost::alloc` post: `host.n_ports == n_ports ∧ host.ports[i] != NULL ∀ i` | `AtaHost::alloc` |
| `AtaHost::start` post: `ATA_HOST_STARTED ∧ each ap finalized + frozen` | `AtaHost::start` |
| `AtaHost::register` post: per port `ap->scsi_host != NULL ∧ ap->cookie scheduled` | `AtaHost::register` |
| `AtaDevice::configure` post: `dev.flags & ATA_DFLAG_CFG_MASK` correctly reflects IDENTIFY | `AtaDevice::configure` |
| `ata_build_rw_tf` post: `tf.command` matches `(protocol, write?, LBA28/48/CHS/NCQ)` from `ata_rw_cmds[]` | `ata_build_rw_tf` |
| `AtaQueuedCmd::issue` post: success ⟹ `ATA_QCFLAG_ACTIVE ∧ qc_active bit set ∧ ops->qc_issue called` | `AtaQueuedCmd::issue` |
| `AtaQueuedCmd::complete` post: `ATA_QCFLAG_ACTIVE` cleared; either EH scheduled or `complete_fn` invoked | `AtaQueuedCmd::complete` |
| `AtaPort::request_pm` post: `ATA_PFLAG_PM_PENDING ∧ ata_port_schedule_eh` invoked | `AtaPort::request_pm` |

### Layer 4: Verus/Creusot functional

`Per-host bringup → ata_host_alloc → ata_host_start (freeze) → ata_host_register (SCSI + async probe) → ata_dev_read_id → ata_dev_configure → ata_set_mode → ata_scsi_scan_host → /dev/sd<X>` semantic equivalence: per-`Documentation/driver-api/libata.rst` and per-IDENTIFY-DEVICE word layout per ATA/ATAPI-8 ACS-5.

## Hardening

(Inherits row-1 features from `drivers/ata/00-overview.md` § Hardening.)

libata-core reinforcement:

- **Per-`devres`-bound host lifetime** — defense against per-driver-unbind leak; `ata_devres_release` cleans up regardless of error path.
- **Per-`kref`-counted `ata_host`** — defense against per-detach UAF; `ata_host_get` / `ata_host_put` pair around async + EH usage.
- **Per-`ata_finalize_port_ops` flatten-then-NULL `inherits`** — defense against per-cyclic vtable inheritance; spinlock-protected for concurrent finalize.
- **Per-port `ATA_PFLAG_FROZEN` at alloc** — defense against per-spurious IRQ before LLD is ready.
- **Per-`async_port_probe` cookie tracked** — defense against per-detach-during-probe race; `async_synchronize_cookie` honored.
- **Per-port `spinlock_t lock` (typically `&host->lock`)** — defense against per-cross-port command-issue race in shared-IRQ designs.
- **Per-tag exclusive non-NCQ enforcement** — defense against per-tag-collision corruption when LLD mixes legacy and queued commands.
- **Per-`fill_result_tf` `ATA_QCFLAG_RTF_FILLED` latch** — defense against per-double-read of HW registers (which on AHCI can race FIS area).
- **Per-`ATA_QCFLAG_EH` schedule-not-complete** — defense against per-error-path-completing-twice; EH owns errored qc until released.
- **Per-`ATA_QUIRK_DISABLE` early-disable** — defense against per-broken-device-bringing-down-bus.
- **Per-cable type sanity (`ata_cable_*`, `cable_is_40wire`)** — defense against per-UDMA-overdrive-on-40-pin-cable corruption (clamps to UDMA2).
- **Per-`ATA_DFLAG_SLEEPING` → schedule reset on issue** — defense against per-issue-to-sleeping-device timeout / lockup.
- **Per-LBA48 vs LBA28 strict range** — defense against per-truncated-LBA silent-corruption beyond 137 GB.
- **Per-`ata_id_is_locked` → `n_sectors = 0`** — defense against per-SED-locked-disk partition-scan I/O errors.
- **Per-host PM via EH-thread only** — defense against per-PM-during-IRQ deadlock; suspend/resume serialized with command path.
- **Per-`ata_port_pm_poweroff` flushes write cache + spindown** — defense against per-S4-hibernation data-loss.
- **Per-`ATA_PFLAG_UNLOADING` on detach** — defense against per-EH-running-after-host-gone.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `libata-eh.c` error handler internals (covered by Tier-3 `libata-eh.md` when expanded; this doc covers only the schedule-EH handoff)
- `libata-scsi.c` SCSI translation (covered in `libata-scsi.md` Tier-3)
- `libata-pmp.c` Port Multiplier (covered in `libata-pmp.md` Tier-3)
- `libata-acpi.c` ACPI integration (covered in `libata-acpi.md` Tier-3)
- `libata-sff.c` / `libata-sata.c` BMDMA + SATA helpers (covered in `libata-sff-sata.md` Tier-3)
- AHCI controller LLD (covered in `ahci-core.md` Tier-3)
- Per-vendor SATA/PATA LLDs (covered in respective `sata_*.md` / `pata_*.md` Tier-3)
- Implementation code
