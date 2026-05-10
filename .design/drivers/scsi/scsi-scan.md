# Tier-3: drivers/scsi/scsi_scan.c — SCSI bus scan / device discovery

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/scsi/00-overview.md
upstream-paths:
  - drivers/scsi/scsi_scan.c (~2151 lines)
  - drivers/scsi/scsi_devinfo.c (bflags lookup, INQUIRY blacklist)
  - drivers/scsi/scsi_sysfs.c (sysfs scan/rescan/delete entries)
  - include/scsi/scsi_device.h (struct scsi_device, struct scsi_target)
  - include/scsi/scsi_host.h (struct Scsi_Host)
  - include/scsi/scsi.h (INQUIRY, REPORT_LUNS, MODE_SENSE constants)
-->

## Summary

`scsi_scan.c` is the SCSI **mid-layer device discovery** engine — walks `<channel, target, lun>` triples behind every `Scsi_Host` (HBA), issues `INQUIRY`, parses the SPC-standard 36+-byte response, optionally issues `REPORT LUNS` for SCSI-3+ targets, and allocates `struct scsi_device` per discovered LUN. Per-`scsi_scan_host` (called by HBA after `scsi_add_host`) chooses sync / async / manual / none mode; per-`__scsi_scan_target` issues LUN-0 INQUIRY then either REPORT-LUNS scan (modern) or sequential-LUN scan (legacy / `BLIST_REPORTLUN2`-fallback); per-`scsi_probe_and_add_lun` performs the 3-pass INQUIRY (36 → response-len → fallback) and routes the result to `scsi_add_lun` which initializes the sdev, attaches VPD pages (per-SCSI-3+), checks CDL, sets queue depth, calls `hostt->sdev_init` / `hostt->sdev_configure`, and finally `scsi_sysfs_add_sdev`. Per-`/sys/class/scsi_host/hostN/scan` ⟹ `scsi_scan_host_selected`. Per-`/sys/.../rescan` ⟹ `scsi_rescan_device` (per-existing sdev re-INQUIRY + VPD refresh). Per-`scsi_add_device(host, ch, id, lun)` ⟹ `__scsi_add_device` (HBA hotplug). Per-`scsi_forget_host` ⟹ remove on `scsi_remove_host`. Per-async-scan tracking: global `scanning_hosts` list of `async_scan_data` cookies with `prev_finished` completions to order sysfs publication.

This Tier-3 covers `drivers/scsi/scsi_scan.c` (~2151 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct async_scan_data` | per-host async-scan cookie | `AsyncScanData` |
| `scsi_scan_type` (module param `scan=sync\|async\|manual\|none`) | per-bus scan-mode | `Scan::mode` |
| `max_scsi_luns` (module param `max_luns`) | per-bus LUN ceiling | `Scan::max_luns` |
| `scsi_inq_timeout` (module param `inq_timeout`) | per-INQUIRY timeout (s) | `Scan::inq_timeout` |
| `scsi_scan_host()` | per-HBA initial scan entry | `Scan::scan_host` |
| `do_scsi_scan_host()` | per-HBA core (LLD scan_start / scan_finished or wild-card) | `Scan::do_scan_host` |
| `do_scan_async()` | per-HBA async cookie body | `Scan::do_scan_async` |
| `scsi_prep_async_scan()` / `scsi_finish_async_scan()` | per-async-scan lifecycle | `Scan::prep_async` / `Scan::finish_async` |
| `scsi_complete_async_scans()` | per-wait-for-prior-async-scans | `Scan::wait_for_prior_async` |
| `scsi_scan_host_selected()` | per-(channel,id,lun) entry from sysfs scan | `Scan::scan_host_selected` |
| `scsi_scan_channel()` | per-channel iteration | `Scan::scan_channel` |
| `scsi_scan_target()` / `__scsi_scan_target()` | per-target scan (LUN 0 + RL or seq) | `Scan::scan_target` / `scan_target_inner` |
| `scsi_sequential_lun_scan()` | per-target sequential LUN-1.. scan | `Scan::sequential_lun_scan` |
| `scsi_report_lun_scan()` | per-target REPORT-LUNS-based scan | `Scan::report_lun_scan` |
| `scsi_probe_and_add_lun()` | per-LUN probe + add | `Scan::probe_and_add_lun` |
| `scsi_probe_lun()` | per-LUN INQUIRY (3-pass) | `Scan::probe_lun` |
| `scsi_add_lun()` | per-LUN setup + sysfs publish | `Scan::add_lun` |
| `scsi_alloc_sdev()` | per-LUN scsi_device alloc | `Scan::alloc_sdev` |
| `scsi_alloc_target()` / `__scsi_find_target()` | per-target alloc / lookup | `Scan::alloc_target` / `find_target` |
| `scsi_target_reap()` / `scsi_target_reap_ref_release()` | per-target last-LUN-removed reap | `Scan::target_reap` / `target_reap_release` |
| `scsi_target_destroy()` / `scsi_target_dev_release()` | per-target finalize | `Scan::target_destroy` / `target_release` |
| `scsi_sanitize_inquiry_string()` | per-INQUIRY string fixup | `Scan::sanitize_inquiry_string` |
| `scsi_unlock_floptical()` | per-BLIST_KEY device unlock via MODE SENSE 0x2e | `Scan::unlock_floptical` |
| `scsi_realloc_sdev_budget_map()` | per-sdev sbitmap budget map alloc | `Scan::realloc_budget_map` |
| `__scsi_add_device()` / `scsi_add_device()` | per-LU hotplug entry | `Scan::add_device_inner` / `Scan::add_device` |
| `scsi_resume_device()` / `scsi_rescan_device()` | per-existing-sdev resume / rescan | `Scan::resume_device` / `Scan::rescan_device` |
| `scsi_forget_host()` | per-HBA-remove tear down all sdevs | `Scan::forget_host` |
| `scsi_get_pseudo_sdev()` | per-host pseudo-sdev for passthrough/tag-mgmt | `Scan::get_pseudo_sdev` |
| `scsi_enable_async_suspend()` | per-dev enable async PM-suspend if scan=async | `Scan::enable_async_suspend` |
| `scsi_attach_vpd()` | per-sdev VPD pages 0x00, 0x80, 0x83, 0xb0–0xb2 | `Scan::attach_vpd` |
| `scsi_cdl_check()` | per-sdev Command Duration Limits check | `Scan::cdl_check` |
| `scanning_hosts` / `async_scan_lock` | global async-scan list | `Scan::ASYNC_HOSTS` |
| `scsi_null_device_strs` ("nullnullnullnull") | placeholder vendor/model/rev | shared |

## Compatibility contract

REQ-1: Module params:
- `scan=sync|async|manual|none` (default per CONFIG_SCSI_SCAN_ASYNC).
  - `sync`: per-HBA bus scan blocks scsi_scan_host caller.
  - `async`: per-HBA bus scan returns immediately; async-publish ordered globally.
  - `manual`: per-HBA bus scan skipped; only sysfs `scan` writes scan.
  - `none`: per-HBA all scans + add_device disabled.
- `max_luns` (default 512): per-bus LUN ceiling.
- `inq_timeout` (default 20s): per-INQUIRY timeout.

REQ-2: Return codes:
- SCSI_SCAN_NO_RESPONSE (0): no valid response / alloc failure.
- SCSI_SCAN_TARGET_PRESENT (1): target responded; no device on LUN.
- SCSI_SCAN_LUN_PRESENT (2): target responded; sdev allocated.

REQ-3: enum scsi_scan_mode:
- SCSI_SCAN_INITIAL: first scan of HBA (skip rescan-only code).
- SCSI_SCAN_RESCAN: rescan existing target / LUN.
- SCSI_SCAN_MANUAL: force scan even if `scan=manual`.

REQ-4: scsi_scan_host(shost):
- if scsi_scan_type ∈ {none, manual}: return.
- if scsi_autopm_get_host(shost) < 0: return.
- data = scsi_prep_async_scan(shost).
- if !data: do_scsi_scan_host(shost); scsi_autopm_put_host(shost); return.
- async_schedule(do_scan_async, data) /* scsi_autopm_put_host deferred to finish_async */.

REQ-5: do_scsi_scan_host(shost):
- if hostt.scan_finished:
  - start = jiffies; if hostt.scan_start: hostt.scan_start(shost).
  - while !hostt.scan_finished(shost, jiffies-start): msleep(10).
- else:
  - scsi_scan_host_selected(shost, SCAN_WILD_CARD, SCAN_WILD_CARD, SCAN_WILD_CARD, SCSI_SCAN_INITIAL).

REQ-6: scsi_scan_host_selected(shost, channel, id, lun, rescan):
- /* range check */
- if (channel != WC ∧ channel > max_channel) ∨ (id != WC ∧ id >= max_id) ∨ (lun != WC ∧ lun >= max_lun): return -EINVAL.
- mutex_lock(&shost.scan_mutex).
- if !shost.async_scan: scsi_complete_async_scans().
- if scsi_host_scan_allowed(shost) ∧ scsi_autopm_get_host(shost) == 0:
  - if channel == WC: for c in 0..=max_channel: scsi_scan_channel(shost, c, id, lun, rescan).
  - else: scsi_scan_channel(shost, channel, id, lun, rescan).
  - scsi_autopm_put_host(shost).
- mutex_unlock(&shost.scan_mutex).
- return 0.

REQ-7: scsi_scan_channel(shost, channel, id, lun, rescan):
- if id == WC: for id in 0..max_id: order_id = (shost.reverse_ordering ? max_id-id-1 : id); __scsi_scan_target(&shost.shost_gendev, channel, order_id, lun, rescan).
- else: __scsi_scan_target(&shost.shost_gendev, channel, id, lun, rescan).

REQ-8: scsi_scan_target(parent, channel, id, lun, rescan):
- if scsi_scan_type == "none": return.
- if rescan != SCSI_SCAN_MANUAL ∧ scsi_scan_type == "manual": return.
- mutex_lock(&shost.scan_mutex).
- if !shost.async_scan: scsi_complete_async_scans().
- if scsi_host_scan_allowed(shost) ∧ scsi_autopm_get_host(shost) == 0:
  - __scsi_scan_target(parent, channel, id, lun, rescan).
  - scsi_autopm_put_host(shost).
- mutex_unlock(&shost.scan_mutex).

REQ-9: __scsi_scan_target(parent, channel, id, lun, rescan):
- if shost.this_id == id: return /* skip HBA's own ID */.
- starget = scsi_alloc_target(parent, channel, id); if !starget: return.
- scsi_autopm_get_target(starget).
- if lun != SCAN_WILD_CARD:
  - scsi_probe_and_add_lun(starget, lun, NULL, NULL, rescan, NULL).
  - goto out_reap.
- /* scan LUN 0 first */
- res = scsi_probe_and_add_lun(starget, 0, &bflags, NULL, rescan, NULL).
- if res == LUN_PRESENT ∨ TARGET_PRESENT:
  - if scsi_report_lun_scan(starget, bflags, rescan) != 0:
    - scsi_sequential_lun_scan(starget, bflags, starget.scsi_level, rescan).
- out_reap: scsi_autopm_put_target; scsi_target_reap(starget); put_device(&starget.dev).

REQ-10: scsi_alloc_target(parent, channel, id):
- size = sizeof(struct scsi_target) + shost.transportt.target_size.
- starget = kzalloc(size, GFP_KERNEL).
- device_initialize(&starget.dev); kref_init(&starget.reap_ref).
- starget.dev.parent = get_device(parent); dev_set_name(target%d:%d:%d, host_no, channel, id).
- starget.dev.bus = &scsi_bus_type; starget.dev.type = &scsi_target_type.
- scsi_enable_async_suspend(&starget.dev).
- starget.id = id; .channel = channel; .can_queue = 0.
- INIT_LIST_HEAD(&starget.siblings / .devices); state = STARGET_CREATED; scsi_level = SCSI_2.
- max_target_blocked = SCSI_DEFAULT_TARGET_BLOCKED.
- retry:
  - spin_lock_irqsave(shost.host_lock).
  - found = __scsi_find_target(parent, channel, id).
  - if found:
    - spin_unlock; ref_got = kref_get_unless_zero(&found.reap_ref); put_device(&starget.dev).
    - if ref_got: return found.
    - /* dying — wait for STARGET_DEL */ msleep(1); goto retry.
  - list_add_tail(&starget.siblings, &shost.__targets).
  - spin_unlock_irqrestore.
  - transport_setup_device(&starget.dev).
  - if hostt.target_alloc: error = hostt.target_alloc(starget); on err: scsi_target_destroy; return NULL.
  - get_device(&starget.dev).
  - return starget.

REQ-11: scsi_alloc_sdev(starget, lun, hostdata):
- sdev = kzalloc(sizeof(scsi_device) + transportt.device_size, GFP_KERNEL).
- sdev.vendor = sdev.model = sdev.rev = scsi_null_device_strs.
- sdev.host = shost; sdev.id = starget.id; sdev.lun = lun; sdev.channel = starget.channel.
- sdev.queue_ramp_up_period = SCSI_DEFAULT_RAMP_UP_PERIOD.
- mutex_init(&sdev.state_mutex / .inquiry_mutex); sdev.sdev_state = SDEV_CREATED.
- INIT_LIST_HEAD(&sdev.siblings / .same_target_siblings / .starved_entry / .event_list).
- spin_lock_init(&sdev.list_lock).
- INIT_WORK(&sdev.event_work, scsi_evt_thread); INIT_WORK(&sdev.requeue_work, scsi_requeue_run_queue).
- sdev.sdev_gendev.parent = get_device(&starget.dev); sdev.sdev_target = starget; sdev.hostdata = hostdata.
- sdev.max_device_blocked = SCSI_DEFAULT_DEVICE_BLOCKED; sdev.type = -1; sdev.borken = 1.
- sdev.sg_reserved_size = INT_MAX.
- scsi_init_limits(shost, &lim); q = blk_mq_alloc_queue(&shost.tag_set, &lim, sdev); if IS_ERR(q): put_device + kfree; out.
- kref_get(&shost.tagset_refcnt); sdev.request_queue = q.
- scsi_sysfs_device_initialize(sdev).
- if is_pseudo_dev(sdev): return sdev.
- depth = shost.cmd_per_lun ?: 1.
- scsi_realloc_sdev_budget_map(sdev, depth); scsi_change_queue_depth(sdev, depth).
- if hostt.sdev_init: ret = hostt.sdev_init(sdev); on -ENXIO no error message; on err: __scsi_remove_device; return NULL.
- return sdev.

REQ-12: scsi_probe_lun(sdev, inq_result, result_len, &bflags) — INQUIRY (3-pass):
- failure_defs allow 3 retries on UNIT_ATTENTION (asc 0x28 / 0x29) + DID_TIME_OUT.
- pass 1: try_inquiry_len = sdev.inquiry_len ?: 36.
- next_pass:
  - for count in 0..3:
    - memset(scsi_cmd, 0, 6); scsi_cmd[0] = INQUIRY; scsi_cmd[4] = try_inquiry_len.
    - memset(inq_result, 0, try_inquiry_len).
    - result = scsi_execute_cmd(sdev, scsi_cmd, REQ_OP_DRV_IN, inq_result, try_inquiry_len, HZ/2 + HZ*inq_timeout, 3, &exec_args).
    - if result == 0 ∧ resid == try_inquiry_len: continue /* USB workaround */.
    - break.
- if result == 0:
  - sanitize vendor (8..16), product (16..32), rev (32..36).
  - response_len = inq_result[4] + 5; if response_len > 255: response_len = first_inquiry_len.
  - *bflags = scsi_get_device_flags(sdev, &inq[8], &inq[16]).
  - if pass == 1:
    - if BLIST_INQUIRY_36: next_inquiry_len = 36.
    - else if sdev.inquiry_len ∧ response_len > sdev.inquiry_len ∧ (inq[2] & 0x7) < 6: next_inquiry_len = sdev.inquiry_len.
    - else: next_inquiry_len = response_len.
    - if next > try: pass = 2; try = next; goto next_pass.
- else if pass == 2: try = first_inquiry_len; pass = 3; goto next_pass.
- if result: return -EIO.
- sdev.inquiry_len = min(try, response_len).
- if sdev.inquiry_len < 36: sdev.host.short_inquiry = 1; sdev.inquiry_len = 36.
- sdev.scsi_level = (inq[2] & 0x0f).
- if scsi_level >= 2 ∨ (level == 1 ∧ (inq[3] & 0x0f) == 1): ++scsi_level.
- sdev.sdev_target.scsi_level = sdev.scsi_level.
- sdev.lun_in_cdb = (scsi_level <= SCSI_2 ∧ scsi_level != SCSI_UNKNOWN ∧ !host.no_scsi2_lun_in_cdb).
- return 0.

REQ-13: scsi_probe_and_add_lun(starget, lun, &bflagsp, &sdevp, rescan, hostdata):
- sdev = scsi_device_lookup_by_target(starget, lun).
- if sdev:
  - if rescan != SCSI_SCAN_INITIAL ∨ !scsi_device_created(sdev):
    - if sdevp: *sdevp = sdev else: scsi_device_put.
    - if bflagsp: *bflagsp = scsi_get_device_flags(sdev, vendor, model).
    - return LUN_PRESENT.
  - else: scsi_device_put(sdev).
- else: sdev = scsi_alloc_sdev(starget, lun, hostdata).
- if !sdev: goto out.
- if scsi_device_is_pseudo_dev(sdev): if bflagsp: *bflagsp = BLIST_NOLUN; return LUN_PRESENT.
- result = kmalloc(256); if !result: goto out_free_sdev.
- if scsi_probe_lun(sdev, result, 256, &bflags): goto out_free_result.
- if bflagsp: *bflagsp = bflags.
- /* PQ = 3: no LU possible */
- if (result[0] >> 5) == 3: res = TARGET_PRESENT; goto out_free_result.
- /* PQ=1 ∨ pdt_1f_for_no_lun ∧ PDT=0x1f ∧ !W-LUN: target-but-no-LU */
- if (((result[0] >> 5) == 1 ∨ starget.pdt_1f_for_no_lun) ∧ (result[0] & 0x1f) == 0x1f ∧ !scsi_is_wlun(lun)): res = TARGET_PRESENT; goto out_free_result.
- res = scsi_add_lun(sdev, result, &bflags, shost.async_scan).
- if res == LUN_PRESENT ∧ (bflags & BLIST_KEY): sdev.lockable = 0; scsi_unlock_floptical(sdev, result).
- out_free_result: kfree(result).
- out_free_sdev: if res == LUN_PRESENT: if sdevp: if scsi_device_get(sdev)==0: *sdevp = sdev else: remove + NO_RESPONSE; else: __scsi_remove_device(sdev).
- out: return res.

REQ-14: scsi_add_lun(sdev, inq_result, &bflags, async):
- sdev.inquiry = kmemdup(inq_result, max(inquiry_len, 36), GFP_KERNEL); if NULL: return NO_RESPONSE.
- sdev.vendor = inq+8; .model = inq+16; .rev = inq+32.
- sdev.is_ata = (strncmp(vendor, "ATA     ", 8) == 0); if is_ata: sdev.allow_restart = 1.
- if BLIST_ISROM: sdev.type = TYPE_ROM; .removable = 1.
- else: sdev.type = inq[0] & 0x1f; .removable = (inq[1] & 0x80) >> 7.
- if scsi_is_wlun(lun) ∧ type != TYPE_WLUN: force type = TYPE_WLUN.
- if type ∈ {TYPE_RBC, TYPE_ROM} ∧ !BLIST_REPORTLUN2: *bflags |= BLIST_NOREPORTLUN.
- sdev.inq_periph_qual = (inq[0] >> 5) & 7.
- sdev.lockable = sdev.removable.
- sdev.soft_reset = (inq[7] & 1) ∧ ((inq[3] & 7) == 2).
- if scsi_level >= SCSI_3 ∨ (inquiry_len > 56 ∧ inq[56] & 0x04): sdev.ppr = 1.
- if inq[7] & 0x60: wdtr = 1; if inq[7] & 0x10: sdtr = 1.
- log "TYPE VENDOR MODEL REV PQ ANSI CCS?".
- if scsi_level >= SCSI_2 ∧ (inq[7] & 2) ∧ !BLIST_NOTQ: sdev.tagged_supported = 1; .simple_tags = 1.
- if !BLIST_BORKEN: sdev.borken = 0.
- bflags ⟹ no_uld_attach / select_no_atn / no_start_on_add / single_lun / use_10_for_rw=1 / no_report_opcodes / not_lockable / retry_hwerror / no_dif / unmap_limit_for_ws / ignore_media_change / try_vpd_pages / skip_vpd_pages / no_vpd_size.
- mutex_lock(&sdev.state_mutex); ret = scsi_device_set_state(sdev, SDEV_RUNNING); if ret: scsi_device_set_state(sdev, SDEV_BLOCK); mutex_unlock; if ret: return NO_RESPONSE.
- sdev.eh_timeout = SCSI_DEFAULT_EH_TIMEOUT.
- transport_configure_device(&sdev.sdev_gendev); sdev.sdev_bflags = *bflags.
- if is_pseudo_dev: return LUN_PRESENT.
- lim = queue_limits_start_update(sdev.request_queue).
- BLIST_MAX_512 / _MAX_1024 ⟹ lim.max_hw_sectors = 512 / 1024.
- if hostt.sdev_configure: ret = hostt.sdev_configure(sdev, &lim); on err: queue_limits_cancel_update; return NO_RESPONSE.
- queue_limits_commit_update(sdev.request_queue, &lim).
- if hostt.sdev_configure: scsi_realloc_sdev_budget_map(sdev, sdev.queue_depth).
- if scsi_level >= SCSI_3: scsi_attach_vpd(sdev).
- scsi_cdl_check(sdev).
- sdev.max_queue_depth = sdev.queue_depth.
- if !async: if scsi_sysfs_add_sdev(sdev) != 0: return NO_RESPONSE /* else deferred to scsi_sysfs_add_devices */.
- return LUN_PRESENT.

REQ-15: scsi_sequential_lun_scan(starget, bflags, scsi_level, rescan):
- max_dev_lun = min(max_scsi_luns, shost.max_lun).
- if BLIST_SPARSELUN: max_dev_lun = shost.max_lun; sparse_lun = 1.
- if BLIST_FORCELUN: max_dev_lun = shost.max_lun.
- if BLIST_MAX5LUN: max_dev_lun = min(5U, max_dev_lun).
- if scsi_level < SCSI_3 ∧ !BLIST_LARGELUN: max_dev_lun = min(8U, max_dev_lun); else max_dev_lun = min(256U, max_dev_lun).
- for lun in 1..max_dev_lun:
  - if scsi_probe_and_add_lun(starget, lun, NULL, NULL, rescan, NULL) != LUN_PRESENT ∧ !sparse_lun: return.

REQ-16: scsi_report_lun_scan(starget, bflags, rescan):
- /* skip conditions */
- if BLIST_NOREPORTLUN: return 1.
- if scsi_level < SCSI_2 ∧ != SCSI_UNKNOWN: return 1.
- if scsi_level < SCSI_3 ∧ (!BLIST_REPORTLUN2 ∨ shost.max_lun <= 8): return 1.
- if BLIST_NOLUN: return 0.
- if starget.no_report_luns: return 1.
- /* get sdev on LUN 0 (alloc if missing) */
- sdev = scsi_device_lookup_by_target(starget, 0) ?: scsi_alloc_sdev(starget, 0, NULL).
- length = (511 + 1) * sizeof(struct scsi_lun).
- retry:
  - lun_data = kmalloc(length).
  - scsi_cmd[0] = REPORT_LUNS; zero [1..5]; put_unaligned_be32(length, &scsi_cmd[6]); [10]=[11]=0.
  - result = scsi_execute_cmd(sdev, scsi_cmd, REQ_OP_DRV_IN, lun_data, length, SCSI_REPORT_LUNS_TIMEOUT (30*HZ), 3, &exec_args).
  - if result: ret = 1; goto out_err.
  - reported = get_unaligned_be32(lun_data.scsi_lun) /* the header */; if reported + sizeof(scsi_lun) > length: length = reported + sizeof(scsi_lun); kfree(lun_data); goto retry.
  - length = reported; num_luns = length / sizeof(scsi_lun).
  - for lunp in &lun_data[1]..=&lun_data[num_luns]:
    - lun = scsilun_to_int(lunp).
    - if lun > shost.max_lun: warn.
    - else: res = scsi_probe_and_add_lun(starget, lun, NULL, NULL, rescan, NULL); if res == NO_RESPONSE: break.
- out_err: kfree(lun_data).
- out: if scsi_device_created(sdev): __scsi_remove_device(sdev) /* unused LUN-0 sdev */; scsi_device_put(sdev).
- return ret.

REQ-17: __scsi_add_device(shost, channel, id, lun, hostdata):
- if scsi_scan_type == "none": return ERR_PTR(-ENODEV).
- starget = scsi_alloc_target(parent=&shost.shost_gendev, channel, id); if !starget: return ERR_PTR(-ENOMEM).
- scsi_autopm_get_target(starget).
- mutex_lock(&shost.scan_mutex).
- if !shost.async_scan: scsi_complete_async_scans().
- if scsi_host_scan_allowed(shost) ∧ scsi_autopm_get_host(shost) == 0:
  - scsi_probe_and_add_lun(starget, lun, NULL, &sdev, SCSI_SCAN_RESCAN, hostdata).
  - scsi_autopm_put_host(shost).
- mutex_unlock; scsi_autopm_put_target; scsi_target_reap(starget); put_device(&starget.dev).
- return sdev.

REQ-18: scsi_add_device(host, channel, target, lun):
- sdev = __scsi_add_device(host, channel, target, lun, NULL).
- if IS_ERR(sdev): return PTR_ERR(sdev).
- scsi_device_put(sdev); return 0.

REQ-19: scsi_resume_device(sdev):
- device_lock(dev).
- if sdev.sdev_state != SDEV_RUNNING ∨ blk_queue_pm_only(sdev.request_queue): ret = -EWOULDBLOCK; goto unlock.
- if dev.driver ∧ try_module_get(driver.owner): if drv.resume: ret = drv.resume(dev); module_put.
- unlock: device_unlock; return ret.

REQ-20: scsi_rescan_device(sdev):
- device_lock(dev).
- if sdev.sdev_state != SDEV_RUNNING ∨ blk_queue_pm_only: ret = -EWOULDBLOCK; goto unlock.
- scsi_attach_vpd(sdev); scsi_cdl_check(sdev).
- if sdev.handler ∧ handler.rescan: handler.rescan(sdev).
- if dev.driver ∧ try_module_get(owner): if drv.rescan: drv.rescan(dev); module_put.
- unlock: device_unlock; return ret.

REQ-21: scsi_forget_host(shost):
- restart:
  - spin_lock_irqsave(shost.host_lock).
  - list_for_each_entry(sdev, &shost.__devices, siblings):
    - if is_pseudo_dev(sdev) ∨ sdev_state == SDEV_DEL: continue.
    - spin_unlock_irqrestore; __scsi_remove_device(sdev); goto restart.
  - spin_unlock_irqrestore.
- if shost.pseudo_sdev: __scsi_remove_device(shost.pseudo_sdev) /* remove last */.

REQ-22: Async scan ordering:
- global: async_scan_lock spinlock + scanning_hosts list of async_scan_data.
- scsi_prep_async_scan(shost): mutex_lock(scan_mutex); if shost.async_scan: error; data = kmalloc; data.shost = scsi_host_get(shost); init_completion(prev_finished); shost.async_scan = true; mutex_unlock; spin_lock(async_scan_lock); if list_empty(scanning_hosts): complete(prev_finished) /* first */; list_add_tail(&data.list, &scanning_hosts); spin_unlock.
- scsi_finish_async_scan(data): mutex_lock(scan_mutex); wait_for_completion(&data.prev_finished) /* serialize sysfs publish */; scsi_sysfs_add_devices(shost); shost.async_scan = false; mutex_unlock; spin_lock(async_scan_lock); list_del; if !list_empty: complete(next.prev_finished); spin_unlock; scsi_autopm_put_host; scsi_host_put; kfree(data).
- scsi_complete_async_scans(): poll-wait, sleeping 1 ms if kmalloc fails; appends sentinel data to scanning_hosts and waits on prev_finished.

REQ-23: scsi_target_reap_ref_release(kref) — last LUN gone:
- starget = container_of(kref, scsi_target, reap_ref).
- if state != STARGET_CREATED ∧ != STARGET_CREATED_REMOVE:
  - transport_remove_device(&starget.dev); device_del(&starget.dev).
- scsi_target_destroy(starget) /* unlinks, calls hostt.target_destroy, put_device */.

REQ-24: scsi_target_destroy(starget):
- BUG_ON(state == STARGET_DEL); state = STARGET_DEL.
- transport_destroy_device(&starget.dev).
- spin_lock_irqsave(shost.host_lock).
- if hostt.target_destroy: hostt.target_destroy(starget).
- list_del_init(&starget.siblings).
- spin_unlock_irqrestore.
- put_device(&starget.dev).

REQ-25: scsi_sanitize_inquiry_string(s, len):
- for each byte: if 0: terminated = 1.
- if terminated ∨ < 0x20 ∨ > 0x7e: byte = ' ' /* SPC: graphic-ASCII padded with spaces */.

REQ-26: scsi_unlock_floptical(sdev, result):
- /* BLIST_KEY: vendor MODE SENSE 0x2e to unlock */
- scsi_cmd = MODE_SENSE, 0, 0x2e, 0, 0x2a, 0.
- scsi_execute_cmd(sdev, scsi_cmd, REQ_OP_DRV_IN, result, 0x2a, SCSI_TIMEOUT, 3, NULL).

REQ-27: PQ / PDT semantics from SPC INQUIRY[0]:
- PQ = (inq[0] >> 5):
  - 0 (000b): physical device present.
  - 1 (001b): supported, not currently connected ⟹ may still create sdev (compat).
  - 3 (011b): not capable ⟹ TARGET_PRESENT only.
- PDT = inq[0] & 0x1f: TYPE_DISK / TYPE_TAPE / TYPE_ROM / TYPE_WLUN (0x1e) / 0x1f = unknown/none.

REQ-28: VPD attach (SCSI_3+, on success):
- scsi_attach_vpd reads page 0x00 (supported-pages), then 0x80 (USN), 0x83 (device-id), 0xb0 (block-limits), 0xb1 (block-dev-char), 0xb2 (LBPRZ/TPE).
- BLIST_TRY_VPD_PAGES ⟹ try_vpd_pages = 1 (force attempt on pre-SCSI-3).
- BLIST_SKIP_VPD_PAGES ⟹ skip_vpd_pages = 1.

REQ-29: sysfs scan/rescan/delete (drivers/scsi/scsi_sysfs.c):
- /sys/class/scsi_host/hostN/scan: write "channel id lun" ⟹ scsi_scan_host_selected(SCSI_SCAN_MANUAL).
- /sys/.../device/rescan: write "1" ⟹ scsi_rescan_device.
- /sys/.../device/delete: write "1" ⟹ scsi_remove_device.

REQ-30: scan_mutex / autopm ordering:
- Outer: shost.scan_mutex held across scan/probe/add_lun chain.
- Inner: starget.dev autopm get/put.
- scsi_complete_async_scans called before sync scan if async_scan == false to avoid sysfs race.

## Acceptance Criteria

- [ ] AC-1: scan=none ⟹ scsi_scan_host returns immediately; __scsi_add_device returns -ENODEV.
- [ ] AC-2: scan=manual ⟹ initial scan_host skipped; sysfs scan-write with SCSI_SCAN_MANUAL works.
- [ ] AC-3: scan=async ⟹ scsi_scan_host returns immediately; sysfs publish ordered by global async list.
- [ ] AC-4: __scsi_scan_target with lun != SCAN_WILD_CARD calls scsi_probe_and_add_lun on that LUN only.
- [ ] AC-5: __scsi_scan_target with wild-card LUN: LUN-0 INQUIRY; LUN_PRESENT ∨ TARGET_PRESENT ⟹ REPORT-LUNS-or-sequential scan.
- [ ] AC-6: SCSI-3+ target ⟹ scsi_report_lun_scan attempted; on failure ⟹ sequential.
- [ ] AC-7: scsi_probe_lun does up to 3 passes; sanitizes vendor/product/rev to printable ASCII.
- [ ] AC-8: PQ=3 ⟹ TARGET_PRESENT only; no sdev added.
- [ ] AC-9: PQ=1 + PDT=0x1f + !W-LUN ⟹ TARGET_PRESENT only.
- [ ] AC-10: scsi_add_lun on SCSI_3+ calls scsi_attach_vpd.
- [ ] AC-11: scsi_alloc_target deduplicates: existing target with matching channel:id returned.
- [ ] AC-12: STARGET_CREATED + last LUN gone ⟹ scsi_target_reap_ref_release skips device_del.
- [ ] AC-13: BLIST_KEY ⟹ scsi_unlock_floptical via MODE_SENSE 0x2e issued.
- [ ] AC-14: scsi_forget_host removes all sdevs; pseudo_sdev removed last.
- [ ] AC-15: sysfs rescan only proceeds if sdev_state == SDEV_RUNNING ∧ !blk_queue_pm_only.

## Architecture

```
struct AsyncScanData {
  list: ListNode,                              // in scanning_hosts
  shost: *ScsiHost,
  prev_finished: Completion,                   // signaled by predecessor
}

enum ScsiScanMode { Initial, Rescan, Manual }
enum ScsiScanResult { NoResponse = 0, TargetPresent = 1, LunPresent = 2 }

struct ScsiTarget {
  dev: Device,
  id: u32, channel: u32,
  reap_ref: Kref,
  state: enum { Created, Running, Remove, CreatedRemove, Del },
  scsi_level: u8,                              // initial SCSI_2; raised by INQUIRY
  siblings: ListNode,                          // in shost.__targets
  devices: ListNode,                           // sdevs on this target
  pdt_1f_for_no_lun: bool,
  no_report_luns: bool,
  ...
}

struct ScsiDevice {
  host: *ScsiHost,
  sdev_target: *ScsiTarget,
  id: u32, channel: u32, lun: u64,
  scsi_level: u8,
  sdev_state: enum SdevState,                  // CREATED → RUNNING / BLOCK / CANCEL / DEL
  inquiry: *u8,                                // kmemdup'd INQUIRY result
  inquiry_len: u16,
  vendor: *u8, model: *u8, rev: *u8,           // pointers into .inquiry
  type: i32,                                   // SPC PDT
  removable, lockable, is_ata, ppr, wdtr, sdtr, tagged_supported, simple_tags: bool,
  sdev_bflags: BlistFlags,
  request_queue: *RequestQueue,
  budget_map: Sbitmap,
  ...
}
```

`Scan::scan_host(shost)`:
1. if mode ∈ {None, Manual}: return.
2. if autopm_get_host(shost) < 0: return.
3. data = prep_async(shost).
4. if !data: do_scsi_scan_host(shost); autopm_put_host(shost); return.
5. async_schedule(do_scan_async, data).

`Scan::do_scsi_scan_host(shost)`:
1. if hostt.scan_finished:
   - start = jiffies; if hostt.scan_start: hostt.scan_start(shost).
   - while !hostt.scan_finished(shost, jiffies - start): msleep(10).
2. else: scsi_scan_host_selected(shost, WC, WC, WC, Initial).

`Scan::scan_target_inner(parent, channel, id, lun, rescan)`:
1. if shost.this_id == id: return.
2. starget = alloc_target(parent, channel, id); if !starget: return.
3. autopm_get_target(starget).
4. if lun != WC:
   - probe_and_add_lun(starget, lun, NULL, NULL, rescan, NULL).
   - goto out_reap.
5. res = probe_and_add_lun(starget, 0, &bflags, NULL, rescan, NULL).
6. if res ∈ {LunPresent, TargetPresent}:
   - if report_lun_scan(starget, bflags, rescan) != 0:
     - sequential_lun_scan(starget, bflags, starget.scsi_level, rescan).
7. out_reap: autopm_put_target; target_reap(starget); put_device(&starget.dev).

`Scan::probe_and_add_lun(starget, lun, bflagsp, sdevp, rescan, hostdata) -> i32`:
1. sdev = scsi_device_lookup_by_target(starget, lun).
2. if sdev:
   - if rescan != Initial ∨ !scsi_device_created(sdev):
     - return-existing path.
   - else: scsi_device_put.
3. else: sdev = alloc_sdev(starget, lun, hostdata); if !sdev: goto out.
4. if is_pseudo_dev: *bflagsp = BLIST_NOLUN; return LunPresent.
5. result = kmalloc(256); if !result: goto out_free_sdev.
6. if probe_lun(sdev, result, 256, &bflags): goto out_free_result.
7. /* PQ=3 */
8. if (result[0] >> 5) == 3: res = TargetPresent; goto out_free_result.
9. /* PQ=1 ∨ pdt_1f_for_no_lun ∧ PDT=0x1f ∧ !wlun */
10. if (((result[0] >> 5) == 1 ∨ starget.pdt_1f_for_no_lun) ∧ (result[0] & 0x1f) == 0x1f ∧ !is_wlun(lun)): res = TargetPresent; goto out_free_result.
11. res = add_lun(sdev, result, &bflags, shost.async_scan).
12. if res == LunPresent ∧ (bflags & BLIST_KEY): sdev.lockable = 0; unlock_floptical(sdev, result).
13. out_free_result: kfree(result).
14. out_free_sdev: handle sdevp / __scsi_remove_device on failure.
15. return res.

`Scan::probe_lun(sdev, inq_result, result_len, bflags) -> i32`:
1. pass = 1; try_len = sdev.inquiry_len ?: 36.
2. next_pass loop: build INQUIRY (op 0x12, len in CDB[4]); scsi_execute_cmd.
3. retry on UA / TIMEOUT up to 3 times.
4. parse on success: sanitize vendor/product/rev; response_len = inq[4]+5.
5. bflags = scsi_get_device_flags(sdev, vendor, product).
6. pass-2 if response_len > try_len; pass-3 fallback if pass-2 fails.
7. sdev.inquiry_len = min(try_len, response_len); clamp to ≥ 36.
8. sdev.scsi_level = (inq[2] & 0x0f) + (level >= 2 ∨ level == 1 && CCS) ? 1 : 0.
9. sdev.sdev_target.scsi_level = sdev.scsi_level.
10. sdev.lun_in_cdb = (level <= SCSI_2 ∧ !no_scsi2_lun_in_cdb).

`Scan::add_lun(sdev, inq, bflags, async) -> i32`:
1. sdev.inquiry = kmemdup(inq, max(inquiry_len, 36)); if NULL: return NoResponse.
2. vendor / model / rev pointers into inquiry.
3. is_ata = (vendor == "ATA     "); if is_ata: allow_restart = 1.
4. type / removable from inq[0] / inq[1]; W-LUN forcing; ROM/RBC ⟹ BLIST_NOREPORTLUN.
5. inq_periph_qual / lockable / soft_reset / ppr / wdtr / sdtr.
6. log INQUIRY line.
7. tagged_supported / simple_tags from SCSI_2+ + inq[7] & 2 + !BLIST_NOTQ.
8. bflag-driven sdev flags.
9. scsi_device_set_state(sdev, SDEV_RUNNING) ?? fallback SDEV_BLOCK.
10. lim = queue_limits_start_update; hostt.sdev_configure(sdev, &lim) ?? queue_limits_commit_update.
11. if SCSI_3+: scsi_attach_vpd(sdev); scsi_cdl_check(sdev).
12. max_queue_depth = queue_depth.
13. if !async: scsi_sysfs_add_sdev(sdev).
14. return LunPresent.

`Scan::report_lun_scan(starget, bflags, rescan) -> i32`:
1. skip-conditions (BLIST, scsi_level, no_report_luns).
2. acquire LUN-0 sdev (alloc if missing).
3. retry: build REPORT_LUNS CDB, length = (511+1)*sizeof(scsi_lun).
4. scsi_execute_cmd; if result: ret = 1; out_err.
5. reported = get_unaligned_be32(lun_data[0]); if too small: bump length; retry.
6. for lunp in lun_data[1..=num_luns]: probe_and_add_lun(starget, scsilun_to_int(lunp), NULL, NULL, rescan, NULL); break on NoResponse.
7. cleanup: kfree(lun_data); remove LUN-0 sdev if created here.

`Scan::sequential_lun_scan(starget, bflags, level, rescan)`:
1. max_dev_lun = min(max_scsi_luns, shost.max_lun).
2. BLIST_SPARSELUN / FORCELUN / MAX5LUN / LARGELUN adjustments.
3. for lun in 1..max_dev_lun: if !LunPresent ∧ !sparse_lun: return.

`Scan::alloc_target(parent, channel, id) -> *ScsiTarget`:
1. kzalloc(sizeof + transportt.target_size).
2. device_initialize; kref_init reap_ref.
3. dev.parent / .bus / .type; dev_set_name "target%d:%d:%d".
4. retry:
   - host_lock; __scsi_find_target.
   - if found ∧ kref_get_unless_zero(reap_ref): unlock; put_device(&starget.dev); return found.
   - if found ∧ dying: unlock; msleep(1); goto retry.
   - list_add_tail(&siblings, &shost.__targets); unlock.
   - transport_setup_device; if hostt.target_alloc: error ⟹ destroy + NULL.
5. get_device(&starget.dev); return starget.

`Scan::alloc_sdev(starget, lun, hostdata) -> *ScsiDevice`:
1. kzalloc + transportt.device_size.
2. Init pointers, state mutexes, work items.
3. blk_mq_alloc_queue + sysfs_device_initialize.
4. depth = shost.cmd_per_lun ?: 1.
5. realloc_budget_map; change_queue_depth.
6. if hostt.sdev_init: on err: __scsi_remove_device.

`Scan::forget_host(shost)`:
1. restart loop under host_lock; for each non-pseudo non-DEL sdev: drop lock; __scsi_remove_device; goto restart.
2. if pseudo_sdev: __scsi_remove_device(pseudo_sdev).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `probe_lun_three_passes_max` | INVARIANT | per-scsi_probe_lun: pass ≤ 3; goto next_pass executed at most twice. |
| `inquiry_sanitization_total` | INVARIANT | per-probe_lun success: vendor/product/rev contain only [0x20..0x7e] or ' '. |
| `inquiry_len_min_36` | INVARIANT | per-add_lun: sdev.inquiry_len ≥ 36. |
| `target_dedup_under_host_lock` | INVARIANT | per-alloc_target: __scsi_find_target called under host_lock. |
| `report_luns_buffer_grow_terminates` | INVARIANT | per-report_lun_scan retry: length monotonically increases bounded by device-reported. |
| `pq3_no_sdev_added` | INVARIANT | per-probe_and_add_lun: (result[0] >> 5) == 3 ⟹ no sdev published to sysfs. |
| `pseudo_sdev_removed_last` | INVARIANT | per-forget_host: pseudo_sdev not in main loop; processed after. |
| `async_scan_singleton_per_host` | INVARIANT | per-prep_async: shost.async_scan == false on entry; true on success. |
| `inquiry_kmemdup_freed_on_destroy` | INVARIANT | per-sdev destroy path: sdev.inquiry kfree'd. |
| `target_reap_ref_balanced` | INVARIANT | per-alloc_target / put_device: kref_get/put balance ⟹ reap_ref_release exactly once. |

### Layer 2: TLA+

`drivers/scsi/scsi-scan.tla`:
- Models: per-mode (sync / async / manual / none) × per-LUN scan-strategy (sequential / report-luns) × per-target lifecycle (CREATED → RUNNING → DEL).
- Properties:
  - `safety_no_double_add_lun` — per-target,lun pair: scsi_add_lun called at most once per `(channel, id, lun)` per scan.
  - `safety_target_dedup` — per-(channel,id): at most one starget on shost.__targets.
  - `safety_async_order_preserved` — per-async-host: sysfs publish in scan_host call order.
  - `safety_lun0_first_for_wildcard` — per-wildcard-scan-target: LUN 0 probed before any LUN > 0.
  - `safety_report_lun_then_seq` — per-target: REPORT-LUNS attempted before sequential if applicable.
  - `safety_scan_under_scan_mutex` — per-host: __scsi_scan_target / scsi_probe_and_add_lun called under shost.scan_mutex.
  - `liveness_scan_terminates` — per-scsi_scan_host: finite steps.
  - `liveness_async_completion_eventually` — per-prep_async: corresponding finish_async signals next.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Scan::probe_lun` post: ret == 0 ⟹ inquiry_len ∈ [36, response_len ∪ try_len] | `Scan::probe_lun` |
| `Scan::add_lun` post: ret == LunPresent ⟹ sdev.inquiry != NULL ∧ sdev.sdev_state ∈ {RUNNING, BLOCK} | `Scan::add_lun` |
| `Scan::probe_and_add_lun` post: PQ=3 ⟹ sdev not in shost.__devices | `Scan::probe_and_add_lun` |
| `Scan::report_lun_scan` post: ret==0 ⟹ all LUNs in REPORT-LUNS response visited | `Scan::report_lun_scan` |
| `Scan::sequential_lun_scan` post: stop on first absent LUN unless sparse | `Scan::sequential_lun_scan` |
| `Scan::alloc_target` post: ret != NULL ⟹ target in shost.__targets ∧ reap_ref >= 1 | `Scan::alloc_target` |
| `Scan::forget_host` post: shost.__devices contains only SDEV_DEL or empty | `Scan::forget_host` |
| `Scan::scan_host_selected` pre: channel/id/lun bounded ∨ SCAN_WILD_CARD | `Scan::scan_host_selected` |

### Layer 4: Verus/Creusot functional

`Per-scsi_scan_host → do_scsi_scan_host → scan_host_selected → scan_channel → __scsi_scan_target → (probe LUN 0) → (report_lun_scan ∨ sequential_lun_scan) → probe_and_add_lun → probe_lun (INQUIRY 3-pass) → add_lun (state ⟶ RUNNING, vpd, cdl, queue, sysfs)` semantic equivalence: per-SPC-5 INQUIRY (T10/BSR INCITS 502), per-SAM-5 LU model, per-Linux Documentation/scsi/scsi_mid_low_api.rst.

## Hardening

(Inherits row-1 features from `drivers/scsi/00-overview.md` § Hardening.)

SCSI-scan reinforcement:

- **Per-INQUIRY string sanitization** — defense against per-non-graphic vendor/model/rev injection (avoid syslog/sysfs garbage).
- **Per-inquiry_len ≥ 36** — defense against per-short-INQUIRY garbage Vendor/Model/Rev reads.
- **Per-probe_lun 3-pass with UA retries** — defense against per-power-on-reset spurious INQUIRY failure.
- **Per-INQUIRY_36 BLIST cap** — defense against per-broken-device that lies about additional-length.
- **Per-PQ=3 rejection** — defense against per-phantom-LU added by mistake.
- **Per-PDT=0x1f + PQ=1 / pdt_1f_for_no_lun rejection** — defense against per-NetApp / USB-UFI phantom-LU.
- **Per-target dedup under host_lock + kref_get_unless_zero** — defense against per-dying-target re-alloc race.
- **Per-REPORT-LUNS buffer grow loop bounded** — defense against per-device-claims-huge-list DoS.
- **Per-LUN range check vs shost.max_lun** — defense against per-out-of-range LUN reported by device.
- **Per-scan_mutex on all entry points** — defense against per-concurrent-scan corruption.
- **Per-async-scan global ordering** — defense against per-sysfs-publish reorder on multi-HBA boot.
- **Per-BLIST_KEY MODE_SENSE 0x2e unlock isolated** — defense against per-floptical-locked-IO indefinite retry.
- **Per-scsi_scan_type=none disables all add_device** — defense against per-runaway-LLD calling __scsi_add_device.
- **Per-pseudo_sdev removed last in forget_host** — defense against per-passthrough-cmd-in-flight UAF on host teardown.

## Open Questions

- (none at this Tier-3 level)

## Out of Scope

- `drivers/scsi/scsi.c` mid-layer dispatch / scsi_execute_cmd (covered in `scsi-mid-core.md` Tier-3)
- `drivers/scsi/scsi_error.c` error-handler thread (covered separately)
- `drivers/scsi/scsi_lib.c` blk-mq glue (covered in `scsi-mid-core.md`)
- `drivers/scsi/scsi_sysfs.c` sysfs surface (covered separately)
- `drivers/scsi/scsi_devinfo.c` BLIST tables (covered separately)
- `drivers/scsi/scsi_transport_*.c` transport classes (covered per-transport)
- `drivers/scsi/sd.c` / `sr.c` / `st.c` ULDs (covered per-ULD)
- VPD page decoding internals (covered in `scsi-vpd.md` if expanded)
- Command Duration Limits internals (covered in `scsi-cdl.md` if expanded)
- Implementation code
