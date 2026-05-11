---
title: "Tier-3: drivers/scsi/scsi_lib.c — SCSI library (request handling + blk-mq integration + I/O completion + sense-decode)"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-11
updated: 2026-05-11
---


## Design Specification

### Summary

The **blk-mq adapter for SCSI**. Every block-layer I/O for a SCSI device passes through `scsi_queue_rq` — the `blk_mq_ops.queue_rq` callback wired into the per-host tag-set. Owns: per-`request` → per-`scsi_cmnd` materialization via `scsi_prepare_cmd`/`scsi_init_command`, per-cmd blk-mq budget gating (`scsi_mq_get_budget`/`scsi_mq_put_budget`) so the per-LLD `can_queue` budget is enforced before the request is started, per-cmd dispatch to the LLD via `scsi_dispatch_cmd` → `host->hostt->queuecommand`, per-cmd completion fan-in via `scsi_done` → `blk_mq_complete_request` → `scsi_complete` → `scsi_decide_disposition` (SUCCESS / NEEDS_RETRY / ADD_TO_MLQUEUE / scsi_eh), per-cmd I/O completion via `scsi_io_completion` (sense-decode, retry, requeue, REPREP fallback), per-cmd DMA-sg-table allocation (`scsi_alloc_sgtables` + DMA-drain), per-host `blk_mq_tag_set` setup (`scsi_mq_setup_tags`), per-`scsi_alloc_request`/`scsi_execute_cmd` synchronous-passthrough API used by ULPs to issue TEST UNIT READY / READ CAPACITY / REPORT LUNS / mode-sense / MODE SELECT. Critical for: SCSI throughput (this is the hot path), correct retry policy, per-cmd UAF freedom, blk-mq budget correctness.

This Tier-3 covers `drivers/scsi/scsi_lib.c` (~3578 lines).

### Acceptance Criteria

- [ ] AC-1: scsi_alloc_request returns a fully-initialized rq with cmd_len = MAX_COMMAND_SIZE and jiffies_at_alloc set.
- [ ] AC-2: scsi_execute_cmd issues a synchronous passthrough command, copies sense / sshdr / resid as requested, retries on scsi_failures match.
- [ ] AC-3: scsi_queue_rq gates dispatch on sdev/target/host queue-ready and budget; on backpressure returns BLK_STS_RESOURCE (or BLK_STS_DEV_RESOURCE when sdev blocked) and unwinds budget cleanly.
- [ ] AC-4: scsi_dispatch_cmd → host->hostt->queuecommand(host, cmd); on rtn != 0 the cmd is requeued with reason ∈ {DEVICE_BUSY, TARGET_BUSY, HOST_BUSY, EH_RETRY}; on rtn == 0 the LLD eventually calls scsi_done.
- [ ] AC-5: scsi_done with submitter == BLOCK_LAYER schedules scsi_complete on the issuing CPU exactly once (SCMD_STATE_COMPLETE bit gate).
- [ ] AC-6: scsi_complete dispatches by scsi_decide_disposition: SUCCESS → ULD ->done → scsi_io_completion; NEEDS_RETRY/ADD_TO_MLQUEUE → scsi_queue_insert; FAILED → scsi_eh_scmd_add.
- [ ] AC-7: scsi_io_completion on result == 0 and partial good_bytes < blk_rq_bytes(req) reissues remainder via scsi_mq_requeue_cmd.
- [ ] AC-8: scsi_io_completion_action: UNIT ATTENTION on removable → FAIL + changed = 1; on non-removable → RETRY.
- [ ] AC-9: scsi_io_completion_action: ILLEGAL_REQUEST + asc==0x20,ascq==0 on use_10_for_rw → flip use_10_for_rw = 0 and REPREP.
- [ ] AC-10: scsi_alloc_sgtables: maps blk_rq physical segments into cmd->sdb.table; appends DMA-drain if hostt->dma_need_drain; on integrity rq, also populates cmd->prot_sdb.
- [ ] AC-11: scsi_mq_setup_tags: tag_set->queue_depth = can_queue + nr_reserved_cmds; tag_set->cmd_size includes inline sg-list and (if T10-PI) prot scsi_data_buffer + inline prot sg-list.
- [ ] AC-12: scsi_mq_init_request: cmd->sense_buffer allocated from scsi_sense_cache slab; freed in scsi_mq_exit_request.
- [ ] AC-13: scsi_command_normalize_sense correctly parses fixed-format (0x70/0x71) and descriptor-format (0x72/0x73) sense buffers; scsi_sense_is_deferred returns true iff response_code ∈ {0x71, 0x73}.
- [ ] AC-14: scsi_result_to_blk_status maps SCSIML_STAT_RESV_CONFLICT → BLK_STS_RESV_CONFLICT, _NOSPC → BLK_STS_NOSPC, _MED_ERROR → BLK_STS_MEDIUM, _TGT_FAILURE → BLK_STS_TARGET, _DL_TIMEOUT → BLK_STS_DURATION_LIMIT; DID_TRANSPORT_FAILFAST/_MARGINAL → BLK_STS_TRANSPORT.
- [ ] AC-15: scsi_init_hctx wires hctx->driver_data = shost; scsi_map_queues delegates to hostt->map_queues if non-NULL else blk_mq_map_queues on HCTX_TYPE_DEFAULT.

### Architecture

```
struct ScsiCmnd {
  device: *ScsiDevice,
  submitter: u8,             // SUBMITTED_BY_*
  cmd_len: u16,
  cmnd: [u8; MAX_COMMAND_SIZE],
  sc_data_direction: u8,     // DMA_*
  sdb: ScsiDataBuffer,       // data sg-table + length
  prot_sdb: Option<*ScsiDataBuffer>,  // T10-PI sg-table
  sense_buffer: *u8,         // SCSI_SENSE_BUFFERSIZE, from scsi_sense_cache slab
  sense_len: u8,
  allowed: u8,
  retries: u8,
  result: u32,               // host_byte | msg_byte | status_byte | scsiml_byte
  resid_len: u32,
  transfersize: u32,
  underflow: u32,
  eh_eflags: u32,
  state: AtomicU32,          // SCMD_STATE_INFLIGHT | _COMPLETE
  flags: u32,                // SCMD_INITIALIZED | _TAGGED | _LAST | ...
  jiffies_at_alloc: u64,
  budget_token: i32,
  abort_work: DelayedWork,
  eh_entry: ListHead,
  rcu: RcuHead,
}

struct ScsiSenseHdr {
  response_code: u8,         // 0x70/0x71 fixed | 0x72/0x73 descriptor
  sense_key: u8,
  asc: u8,
  ascq: u8,
  byte4: u8,
  byte5: u8,
  byte6: u8,
  additional_length: u8,
}

enum QcStatus {
  Ok = 0,
  DeviceBusy = SCSI_MLQUEUE_DEVICE_BUSY,
  TargetBusy = SCSI_MLQUEUE_TARGET_BUSY,
  HostBusy = SCSI_MLQUEUE_HOST_BUSY,
  EhRetry = SCSI_MLQUEUE_EH_RETRY,
}

enum Disposition { Success, NeedsRetry, AddToMlqueue, Failed, FastIoFail, TimeoutError }
```

`Cmnd::queue_rq(hctx, bd) -> BlkStatus`:
1. cmd = blk_mq_rq_to_pdu(bd.rq); WARN(cmd.budget_token < 0).
2. /* Reserved-cmd bypass */
3. if !is_reserved(bd.rq):
   - if sdev.state != SDEV_RUNNING: ret = device_state_check(sdev, bd.rq); on err → out_put_budget.
   - if !Target::queue_ready(shost, sdev): { ret = RESOURCE; goto out_put_budget; }.
   - if Host::in_recovery(shost):
     - if cmd.flags & FAIL_IF_RECOVERING: ret = OFFLINE; else ret = RESOURCE; goto out_dec_target_busy.
   - if !Host::queue_ready(q, shost, sdev, cmd): ret = RESOURCE; goto out_dec_target_busy.
4. /* Per-LLD cmd-priv clear */
5. if hostt.cmd_size ∧ !init_cmd_priv: memset(scsi_cmd_priv(cmd), 0).
6. /* Prepare once */
7. if !(rq.rq_flags & RQF_DONTPREP):
   - ret = Cmnd::prepare(bd.rq); on err → out_dec_host_busy.
   - rq.rq_flags |= RQF_DONTPREP.
8. else: clear_bit(SCMD_STATE_COMPLETE).
9. cmd.flags &= SCMD_PRESERVED_FLAGS.
10. if sdev.simple_tags: cmd.flags |= SCMD_TAGGED.
11. if bd.last: cmd.flags |= SCMD_LAST.
12. cmd.set_resid(0); cmd.sense_buffer.zeroize(); cmd.submitter = BLOCK_LAYER.
13. blk_mq_start_request(bd.rq).
14. if is_reserved(bd.rq):
    - reason = hostt.queue_reserved_command(shost, cmd).
    - if reason: ret = RESOURCE; goto out_put_budget.
    - return Ok.
15. reason = Cmnd::dispatch(cmd).
16. if reason: scsi_set_blocked(cmd, reason); ret = RESOURCE; goto out_dec_host_busy.
17. return Ok.
18. /* Unwind */
19. out_dec_host_busy: Host::dec_busy(shost, cmd).
20. out_dec_target_busy: if target.can_queue > 0: atomic_dec(target.busy).
21. out_put_budget: Host::put_budget(q, cmd.budget_token); cmd.budget_token = -1.
22. /* Translate */
23. match ret { RESOURCE if device_blocked(sdev) → DEV_RESOURCE, AGAIN → cmd.result = DID_BUS_BUSY << 16; if DONTPREP: uninit, default → ... }.

`Cmnd::prepare(rq) -> BlkStatus`:
1. cmd = blk_mq_rq_to_pdu(rq); sdev = q.queuedata; shost = sdev.host.
2. in_flight = test_bit(SCMD_STATE_INFLIGHT, &cmd.state).
3. Cmnd::init(sdev, cmd):
   - if !is_passthrough(rq) ∧ !(cmd.flags & SCMD_INITIALIZED): cmd.flags |= SCMD_INITIALIZED; Cmnd::initialize_rq(rq).
   - cmd.device = sdev; cmd.eh_entry.init(); cmd.abort_work.init(scmd_eh_abort_handler).
4. cmd.eh_eflags = 0; cmd.prot_type = 0; cmd.prot_flags = 0; cmd.submitter = 0.
5. cmd.sdb = zeroed.
6. cmd.underflow = 0; cmd.transfersize = 0; cmd.host_scribble = NULL; cmd.result = 0; cmd.extra_len = 0; cmd.state = 0.
7. if in_flight: __set_bit(SCMD_STATE_INFLIGHT).
8. cmd.prot_op = SCSI_PROT_NORMAL.
9. cmd.sc_data_direction = if blk_rq_bytes(rq) > 0 { rq_dma_dir(rq) } else { DMA_NONE }.
10. cmd.sdb.table.sgl = (cmd + sizeof(ScsiCmnd) + hostt.cmd_size) as *Scatterlist.
11. if scsi_host_get_prot(shost):
    - cmd.prot_sdb.zeroize().
    - cmd.prot_sdb.table.sgl = (cmd.prot_sdb + 1) as *Scatterlist.
12. if is_passthrough(rq): return Cmnd::setup_passthrough(sdev, rq).
13. if sdev.handler ∧ handler.prep_fn:
    - ret = handler.prep_fn(sdev, rq); on err → return ret.
14. cmd.allowed = 0; cmd.cmnd.zeroize().
15. return ScsiDriver::for_cmnd(cmd).init_command(cmd).

`Cmnd::dispatch(cmd) -> QcStatus`:
1. host = cmd.device.host.
2. atomic_inc(&cmd.device.iorequest_cnt).
3. if cmd.device.sdev_state == SDEV_DEL: cmd.result = DID_NO_CONNECT << 16; goto done.
4. if Device::is_blocked(cmd.device): atomic_dec; return QcStatus::DeviceBusy.
5. if cmd.device.lun_in_cdb: cmd.cmnd[1] = (cmnd[1] & 0x1f) | ((lun << 5) & 0xe0).
6. scsi_log_send(cmd).
7. if cmd.cmd_len > host.max_cmd_len: cmd.result = DID_ABORT << 16; goto done.
8. if host.shost_state == SHOST_DEL: cmd.result = DID_NO_CONNECT << 16; goto done.
9. trace_scsi_dispatch_cmd_start(cmd).
10. rtn = host.hostt.queuecommand(host, cmd).
11. if rtn:
    - atomic_dec(&iorequest_cnt).
    - if rtn ∉ {DeviceBusy, TargetBusy}: rtn = HostBusy.
12. return rtn.
13. done: Cmnd::done(cmd); return Ok.

`Cmnd::io_completion(cmd, good_bytes)`:
1. result = cmd.result; req = cmd.to_rq(); blk_stat = Ok.
2. if result != 0: result = Cmnd::io_completion_nz(cmd, result, &mut blk_stat).
3. if blk_rq_bytes(req) > 0 ∨ blk_stat == Ok:
   - if !Cmnd::end_request(req, blk_stat, good_bytes): return.
4. if blk_stat ∧ Cmnd::noretry(cmd):
   - if Cmnd::end_request(req, blk_stat, blk_rq_bytes(req)) WARN_ONCE.
   - return.
5. if result == 0: Cmnd::requeue(cmd, 0).
6. else: Cmnd::io_completion_action(cmd, result).

`Cmnd::io_completion_action(cmd, result)`:
1. sense_valid = parse_sense(cmd, &mut sshdr); sense_current = sense_valid ∧ !sshdr.is_deferred().
2. blk_stat = result_to_blk_status(result).
3. action = match (host_byte(result), sense_valid, sshdr.sense_key) {
   (DID_RESET, _, _) → Retry,
   (_, true, UNIT_ATTENTION) → if removable { changed = 1; Fail } else { Retry },
   (_, true, ILLEGAL_REQUEST) → /* 10→6 fallback, DIX, opcode-invalid */,
   (_, true, ABORTED_COMMAND) → Fail (DIF → BLK_STS_PROTECTION),
   (_, true, NOT_READY) → /* per-ascq: DelayedRetry, DelayedReprep (ALUA), Fail (depopulation) */,
   (_, true, VOLUME_OVERFLOW | COMPLETED | _) → Fail,
   (_, true, DATA_PROTECT) → Fail (insufficient-zone → BLK_STS_ZONE_OPEN_RESOURCE),
   _ → Fail,
 }.
4. if action != Fail ∧ Cmnd::runtime_exceeced(cmd): action = Fail.
5. match action {
   Fail → log if !QUIET; if !end_request(req, blk_stat, rq_err_bytes(req)) return; fallthrough Reprep,
   Reprep → Cmnd::requeue(cmd, 0),
   DelayedReprep → Cmnd::requeue(cmd, ALUA_TRANSITION_REPREP_DELAY),
   Retry → Cmnd::queue_insert_inner(cmd, EH_RETRY, false),
   DelayedRetry → Cmnd::queue_insert_inner(cmd, DEVICE_BUSY, false),
 }.

`Cmnd::alloc_sgtables(cmd) -> BlkStatus`:
1. nr_segs = blk_rq_nr_phys_segments(rq).
2. if nr_segs == 0: return IoErr.
3. need_drain = sdev.dma_drain_len > 0 ∧ is_passthrough(rq) ∧ !op_is_write(req_op(rq)) ∧ hostt.dma_need_drain(rq).
4. if need_drain: nr_segs += 1.
5. if sg_alloc_table_chained(&cmd.sdb.table, nr_segs, cmd.sdb.table.sgl, SCSI_INLINE_SG_CNT) != 0: return Resource.
6. count = __blk_rq_map_sg(rq, cmd.sdb.table.sgl, &mut last_sg).
7. if blk_rq_bytes(rq) & q.limits.dma_pad_mask:
   - pad_len = (dma_pad_mask & ~blk_rq_bytes(rq)) + 1.
   - last_sg.length += pad_len; cmd.extra_len += pad_len.
8. if need_drain:
   - sg_unmark_end(last_sg); last_sg = sg_next(last_sg).
   - sg_set_buf(last_sg, sdev.dma_drain_buf, sdev.dma_drain_len); sg_mark_end.
   - cmd.extra_len += sdev.dma_drain_len; count += 1.
9. BUG_ON(count > sdb.table.nents); sdb.table.nents = count; sdb.length = blk_rq_payload_bytes(rq).
10. if blk_integrity_rq(rq):
    - prot_sdb = cmd.prot_sdb /* WARN if NULL */.
    - if sg_alloc_table_chained(&prot_sdb.table, rq.nr_integrity_segments, prot_sdb.table.sgl, SCSI_INLINE_PROT_SG_CNT): ret = Resource; goto out.
    - count = blk_rq_map_integrity_sg(rq, prot_sdb.table.sgl).
    - prot_sdb.table.nents = count.
11. return Ok.
12. out: Cmnd::free_sgtables(cmd); return ret.

### Out of Scope

- drivers/scsi/scsi_error.c full EH state machine (covered in `scsi-error.md` future Tier-3; scsi_eh_inc_host_failed touch-point only referenced here)
- drivers/scsi/scsi_scan.c LUN scanning (covered in `scsi-scan.md`)
- drivers/scsi/scsi.c subsystem init and Scsi_Host registration (covered in `scsi-mid-core.md`)
- drivers/scsi/hosts.c host alloc/add/remove (covered in `scsi-mid-core.md`)
- drivers/scsi/sd.c the disk ULD (covered in `sd.md`)
- block/blk-mq.c (covered in `block/blk-mq.md` future Tier-3)
- T10-PI / DIX integrity payload generation (covered in `block/integrity.md` future Tier-3; consumer-side only here)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct scsi_cmnd` | per-request command (allocated as blk-mq pdu) | `kernel::scsi::Cmnd` |
| `struct scsi_data_buffer` | per-cmd data-segment sgtable + length | `kernel::scsi::DataBuffer` |
| `struct scsi_sense_hdr` | per-cmd parsed sense bytes | `kernel::scsi::SenseHdr` |
| `enum scsi_qc_status` | per-LLD queue-state (HOST_BUSY / DEVICE_BUSY / TARGET_BUSY / EH_RETRY) | `kernel::scsi::QcStatus` |
| `enum scsi_disposition` | per-cmd post-completion verdict (SUCCESS / NEEDS_RETRY / ADD_TO_MLQUEUE / FAILED) | `kernel::scsi::Disposition` |
| `scsi_alloc_request(q, opf, flags)` | per-request allocation + init | `Cmnd::alloc_request` |
| `scsi_initialize_rq(rq)` | per-rq init (cdb cleared, cmd_len, jiffies_at_alloc) | `Cmnd::initialize_rq` |
| `scsi_init_command(dev, cmd)` | per-cmd init at prep time | `Cmnd::init` |
| `scsi_prepare_cmd(req)` | per-rq → per-cmd materialization, dispatch to ULD `init_command` | `Cmnd::prepare` |
| `scsi_setup_scsi_cmnd(sdev, req)` | per-passthrough setup (copy cdb, alloc sg) | `Cmnd::setup_passthrough` |
| `scsi_execute_cmd(sdev, cdb, opf, buf, len, timeout, retries, args)` | per-synchronous-passthrough wrapper | `Cmnd::execute` |
| `scsi_dispatch_cmd(cmd)` | per-cmd dispatch into LLD `queuecommand` | `Cmnd::dispatch` |
| `scsi_queue_rq(hctx, bd)` | per-rq blk-mq entry-point | `Cmnd::queue_rq` |
| `scsi_done(cmd)` / `scsi_done_direct(cmd)` | per-cmd completion entry from LLD ISR | `Cmnd::done` / `_direct` |
| `scsi_complete(rq)` | per-cmd post-IRQ completion bottom-half | `Cmnd::complete` |
| `scsi_io_completion(cmd, good_bytes)` | per-fs-I/O completion path | `Cmnd::io_completion` |
| `scsi_io_completion_action(cmd, result)` | per-error-action selector (FAIL / REPREP / RETRY / DELAYED_RETRY) | `Cmnd::io_completion_action` |
| `scsi_io_completion_nz_result(cmd, result, *blk_stat)` | per-non-zero-result classifier | `Cmnd::io_completion_nz` |
| `scsi_end_request(req, blk_stat, bytes)` | per-cmd bytes-done finalization | `Cmnd::end_request` |
| `scsi_mq_requeue_cmd(cmd, msecs)` | per-cmd unprepare + requeue | `Cmnd::requeue` |
| `scsi_queue_insert(cmd, reason)` | per-cmd reinsertion (busy / EH retry) | `Cmnd::queue_insert` |
| `__scsi_queue_insert(cmd, reason, unbusy)` | per-cmd internal reinsertion | `Cmnd::queue_insert_inner` |
| `scsi_result_to_blk_status(result)` | per-result-byte → blk_status_t | `Cmnd::result_to_blk_status` |
| `scsi_alloc_sgtables(cmd)` | per-cmd sg-table alloc (data + integrity + DMA drain) | `Cmnd::alloc_sgtables` |
| `scsi_free_sgtables(cmd)` | per-cmd sg-table free | `Cmnd::free_sgtables` |
| `scsi_mq_get_budget(q)` / `_put_budget` | per-LLD-budget gate (sbitmap) | `Host::get_budget` / `_put` |
| `scsi_dev_queue_ready(q, sdev)` | per-sdev `device_busy` + `queue_depth` check | `Device::queue_ready` |
| `scsi_target_queue_ready(shost, sdev)` | per-target `target_busy` + `can_queue` check | `Target::queue_ready` |
| `scsi_host_queue_ready(q, shost, sdev, cmd)` | per-host `host_busy` + `can_queue` check | `Host::queue_ready` |
| `scsi_device_unbusy(sdev, cmd)` | per-completion budget release | `Device::unbusy` |
| `scsi_dec_host_busy(shost, cmd)` | per-completion host counter dec | `Host::dec_busy` |
| `scsi_run_queue(q)` | per-queue restart (starved-list + kick-mq) | `Queue::run` |
| `scsi_run_host_queues(shost)` | per-host restart of every sdev queue | `Host::run_queues` |
| `scsi_starved_list_run(shost)` | per-host starved-list dispatch | `Host::starved_list_run` |
| `scsi_init_hctx(hctx, data, hctx_idx)` | per-hctx init (link `shost` into hctx) | `Host::init_hctx` |
| `scsi_map_queues(set)` | per-tag-set queue-mapping | `Host::map_queues` |
| `scsi_mq_init_request(set, rq, hctx_idx, numa_node)` | per-rq pdu init (alloc sense + cmd-priv) | `Cmnd::mq_init_request` |
| `scsi_mq_exit_request(set, rq, hctx_idx)` | per-rq pdu deinit (free sense + cmd-priv) | `Cmnd::mq_exit_request` |
| `scsi_mq_poll(hctx, iob)` | per-hctx polling completion | `Host::mq_poll` |
| `scsi_mq_setup_tags(shost)` | per-host blk-mq tag-set construction | `Host::setup_tags` |
| `scsi_mq_free_tags(kref)` | per-host tag-set teardown | `Host::free_tags` |
| `scsi_init_limits(shost, lim)` | per-host queue-limits init (sg, max_hw_sectors, dma) | `Host::init_limits` |
| `scsi_mq_lld_busy(q)` | per-q busy-test (target_busy or host_self_blocked) | `Host::lld_busy` |
| `scsi_decide_disposition(cmd)` | per-cmd disposition (SUCCESS / NEEDS_RETRY / EH) (defined in scsi_error.c) | `Cmnd::decide_disposition` |
| `scsi_eh_scmd_add(cmd)` | per-cmd escalation into EH list (defined in scsi_error.c) | `Eh::scmd_add` |
| `scsi_command_normalize_sense(cmd, &shdr)` | per-cmd sense-buffer → struct decode | `SenseHdr::from_cmnd` |
| `scsi_build_sense(scmd, desc, key, asc, ascq)` | per-cmd manual sense-buffer build | `SenseHdr::build` |
| `scsi_get_internal_cmd(sdev, dir, flags)` / `_put` | per-LLD-internal cmd alloc | `Cmnd::internal_alloc` / `_put` |
| `scsi_block_requests(shost)` / `_unblock_requests` | per-host self-block | `Host::block_requests` / `_unblock` |
| `scsi_device_quiesce(sdev)` / `_resume` | per-device freeze + drain | `Device::quiesce` / `_resume` |
| `scsi_internal_device_block_nowait(sdev)` / `_unblock_nowait` | per-device transient block | `Device::block_nowait` / `_unblock_nowait` |
| `scsi_host_block(shost)` / `_host_unblock` | per-host iter-block all sdev | `Host::block_all` / `_unblock_all` |
| `scsi_vpd_lun_id(sdev, id, id_len)` | per-LUN designator (NAA / EUI64 / serial) | `Device::vpd_lun_id` |
| `scsi_vpd_lun_serial(sdev, sn, sn_size)` | per-LUN serial-number from VPD page 0x80 | `Device::vpd_lun_serial` |
| `scsi_vpd_tpg_id(sdev, *rel_id)` | per-LUN target-port-group from VPD page 0x83 | `Device::vpd_tpg_id` |
| `scsi_kmap_atomic_sg(sgl, n, *off, *len)` / `_kunmap` | per-sg kmap helper | `Sg::kmap_atomic` / `_kunmap` |

### compatibility contract

REQ-1: struct scsi_cmnd (the per-blk-mq-request pdu):
- device: per-sdev pointer.
- submitter: SUBMITTED_BY_BLOCK_LAYER / _SCSI_ERROR_HANDLER / _SCSI_RESET_IOCTL.
- cmd_len: per-CDB length (6/10/12/16/32).
- cmnd[MAX_COMMAND_SIZE]: per-CDB bytes.
- sc_data_direction: DMA_TO_DEVICE / DMA_FROM_DEVICE / DMA_NONE / DMA_BIDIRECTIONAL.
- sdb: per-cmd data scsi_data_buffer (sg-table + length).
- prot_sdb: per-cmd protection-info scsi_data_buffer (T10 PI; NULL if DIX off).
- sense_buffer: SCSI_SENSE_BUFFERSIZE bytes allocated from `scsi_sense_cache` slab at `scsi_mq_init_request`.
- sense_len: per-cmd valid-sense-bytes count.
- allowed: per-cmd ml-retries budget.
- retries: per-cmd retries-so-far count.
- result: per-cmd packed result (host_byte | msg_byte | status_byte | scsiml_byte).
- resid_len: per-cmd residual bytes.
- transfersize: per-cmd transfer-unit (= sector_size for r/w).
- underflow: per-cmd minimum transfer (must move ≥ underflow bytes).
- eh_eflags: per-cmd EH-iteration flags.
- state: SCMD_STATE_INFLIGHT / _COMPLETE bitmap.
- flags: SCMD_INITIALIZED / _TAGGED / _LAST / _FAIL_IF_RECOVERING / _PRESERVED_FLAGS.
- jiffies_at_alloc: per-cmd allocation timestamp (used by `scsi_cmd_runtime_exceeced`).
- budget_token: per-cmd sbitmap-allocated slot (= scsi_dev_queue_ready return).
- abort_work: per-cmd delayed-EH-abort work_struct.
- eh_entry: per-EH-list linkage.
- rcu: per-cmd RCU head (for call_rcu_hurry in EH).

REQ-2: scsi_alloc_request(q, opf, flags):
- /* Pre-condition: q is a SCSI mq queue (q->mq_ops == &scsi_mq_ops*) */
- rq = blk_mq_alloc_request(q, opf, flags).
- if !IS_ERR(rq):
  - scsi_initialize_rq(rq) /* memset cmnd[0..MAX_COMMAND_SIZE], cmd_len = MAX_COMMAND_SIZE, sense_len = 0, init_rcu_head, jiffies_at_alloc = jiffies, retries = 0 */.
- return rq.

REQ-3: scsi_execute_cmd(sdev, cdb, opf, buf, bufflen, timeout, ml_retries, args):
- /* args may be NULL (defaults) or carry sense / sshdr / resid / scmd_flags / req_flags / failures */
- retry:
  - req = scsi_alloc_request(sdev->request_queue, opf, args->req_flags).
  - if IS_ERR(req): return PTR_ERR(req).
  - if bufflen:
    - ret = blk_rq_map_kern(req, buffer, bufflen, GFP_NOIO).
    - if ret: goto out.
  - scmd = blk_mq_rq_to_pdu(req).
  - scmd->cmd_len = COMMAND_SIZE(cmd[0]).
  - memcpy(scmd->cmnd, cdb, scmd->cmd_len).
  - scmd->allowed = ml_retries.
  - scmd->flags |= args->scmd_flags.
  - req->timeout = timeout.
  - req->rq_flags |= RQF_QUIET.
  - blk_execute_rq(req, true) /* head-injection; bypasses quiesce */.
  - if scsi_check_passthrough(scmd, args->failures) == -EAGAIN:
    - blk_mq_free_request(req); goto retry.
  - if scmd->resid_len > 0 ∧ scmd->resid_len <= bufflen:
    - memset(buffer + bufflen - resid_len, 0, resid_len) /* zero garbage residue */.
  - if args->resid: *args->resid = scmd->resid_len.
  - if args->sense: memcpy(args->sense, scmd->sense_buffer, SCSI_SENSE_BUFFERSIZE).
  - if args->sshdr: scsi_normalize_sense(scmd->sense_buffer, scmd->sense_len, args->sshdr).
  - ret = scmd->result.
- out: blk_mq_free_request(req).
- return ret.

REQ-4: scsi_queue_rq(hctx, bd) — blk_mq_ops.queue_rq:
- req = bd->rq; sdev = q->queuedata; shost = sdev->host; cmd = blk_mq_rq_to_pdu(req).
- WARN_ON_ONCE(cmd->budget_token < 0) /* must have come through scsi_mq_get_budget */.
- /* Reserved-cmd bypass */
- if !blk_mq_is_reserved_rq(req):
  - if sdev->sdev_state != SDEV_RUNNING:
    - ret = scsi_device_state_check(sdev, req).
    - if ret != BLK_STS_OK: goto out_put_budget.
  - if !scsi_target_queue_ready(shost, sdev): { ret = BLK_STS_RESOURCE; goto out_put_budget; }.
  - if scsi_host_in_recovery(shost):
    - if cmd->flags & SCMD_FAIL_IF_RECOVERING: ret = BLK_STS_OFFLINE.
    - else: ret = BLK_STS_RESOURCE.
    - goto out_dec_target_busy.
  - if !scsi_host_queue_ready(q, shost, sdev, cmd): { ret = BLK_STS_RESOURCE; goto out_dec_target_busy; }.
- /* Clear LLD-private */
- if shost->hostt->cmd_size ∧ !init_cmd_priv: memset(scsi_cmd_priv(cmd), 0, cmd_size).
- /* Prepare cmd once (rq_flags & RQF_DONTPREP gates re-entry) */
- if !(req->rq_flags & RQF_DONTPREP):
  - ret = scsi_prepare_cmd(req).
  - if ret != BLK_STS_OK: goto out_dec_host_busy.
  - req->rq_flags |= RQF_DONTPREP.
- else:
  - clear_bit(SCMD_STATE_COMPLETE, &cmd->state) /* re-armed for retry */.
- cmd->flags &= SCMD_PRESERVED_FLAGS.
- if sdev->simple_tags: cmd->flags |= SCMD_TAGGED.
- if bd->last: cmd->flags |= SCMD_LAST.
- scsi_set_resid(cmd, 0).
- memset(cmd->sense_buffer, 0, SCSI_SENSE_BUFFERSIZE).
- cmd->submitter = SUBMITTED_BY_BLOCK_LAYER.
- blk_mq_start_request(req) /* arms timeout, sets MQ_RQ_IN_FLIGHT */.
- if blk_mq_is_reserved_rq(req):
  - reason = shost->hostt->queue_reserved_command(shost, cmd).
  - if reason: { ret = BLK_STS_RESOURCE; goto out_put_budget; }.
  - return BLK_STS_OK.
- reason = scsi_dispatch_cmd(cmd).
- if reason:
  - scsi_set_blocked(cmd, reason) /* mark host/target/dev blocked */.
  - ret = BLK_STS_RESOURCE; goto out_dec_host_busy.
- return BLK_STS_OK.
- /* Unwind paths */
- out_dec_host_busy: scsi_dec_host_busy(shost, cmd).
- out_dec_target_busy: if scsi_target(sdev)->can_queue > 0: atomic_dec(&target_busy).
- out_put_budget: scsi_mq_put_budget(q, cmd->budget_token); cmd->budget_token = -1.
- /* Translate ret for blk-mq */
- if ret == BLK_STS_RESOURCE ∧ scsi_device_blocked(sdev): ret = BLK_STS_DEV_RESOURCE.
- if ret == BLK_STS_AGAIN: cmd->result = DID_BUS_BUSY << 16; if DONTPREP: scsi_mq_uninit_cmd(cmd).
- if default-error: cmd->result = DID_NO_CONNECT << 16 (offline) or DID_ERROR << 16; if DONTPREP: uninit; scsi_run_queue_async(sdev).
- return ret.

REQ-5: scsi_prepare_cmd(req):
- cmd = blk_mq_rq_to_pdu(req); sdev = q->queuedata; shost = sdev->host.
- in_flight = test_bit(SCMD_STATE_INFLIGHT, &cmd->state) /* preserved across requeue */.
- scsi_init_command(sdev, cmd):
  - if !passthrough ∧ !(cmd->flags & SCMD_INITIALIZED): cmd->flags |= SCMD_INITIALIZED; scsi_initialize_rq(rq).
  - cmd->device = sdev; INIT_LIST_HEAD(&cmd->eh_entry); INIT_DELAYED_WORK(&cmd->abort_work, scmd_eh_abort_handler).
- cmd->eh_eflags = 0; prot_type = 0; prot_flags = 0; submitter = 0; underflow = 0; transfersize = 0; host_scribble = NULL; result = 0; extra_len = 0; state = 0.
- if in_flight: __set_bit(SCMD_STATE_INFLIGHT).
- cmd->prot_op = SCSI_PROT_NORMAL.
- cmd->sc_data_direction = blk_rq_bytes(req) ? rq_dma_dir(req) : DMA_NONE.
- /* Inline sg-list immediately after pdu (cmnd struct + LLD cmd_size) */
- cmd->sdb.table.sgl = (void*)cmd + sizeof(scsi_cmnd) + hostt->cmd_size.
- if scsi_host_get_prot(shost): cmd->prot_sdb->table.sgl = (sg*)(prot_sdb + 1).
- if blk_rq_is_passthrough(req): return scsi_setup_scsi_cmnd(sdev, req) /* copies cdb from req->cmd into scmd->cmnd; scsi_alloc_sgtables */.
- if sdev->handler ∧ ->prep_fn: prep_fn(sdev, req) /* ALUA / multipath hook */.
- cmd->allowed = 0; memset(cmd->cmnd, 0, MAX_COMMAND_SIZE).
- return scsi_cmd_to_driver(cmd)->init_command(cmd) /* ULD-specific: sd_init_command / sr_init_command / ... */.

REQ-6: scsi_dispatch_cmd(cmd):
- host = cmd->device->host.
- atomic_inc(&cmd->device->iorequest_cnt).
- if sdev_state == SDEV_DEL: cmd->result = DID_NO_CONNECT << 16; goto done.
- if scsi_device_blocked(cmd->device): atomic_dec(&iorequest_cnt); return SCSI_MLQUEUE_DEVICE_BUSY.
- if cmd->device->lun_in_cdb: cmd->cmnd[1] = (cmnd[1] & 0x1f) | ((lun << 5) & 0xe0).
- scsi_log_send(cmd).
- if cmd->cmd_len > host->max_cmd_len: cmd->result = DID_ABORT << 16; goto done.
- if host->shost_state == SHOST_DEL: cmd->result = DID_NO_CONNECT << 16; goto done.
- trace_scsi_dispatch_cmd_start(cmd).
- rtn = host->hostt->queuecommand(host, cmd) /* LLD vtable callback; returns 0 on accept, non-zero = SCSI_MLQUEUE_*_BUSY */.
- if rtn:
  - atomic_dec(&iorequest_cnt).
  - trace_scsi_dispatch_cmd_error(cmd, rtn).
  - if rtn ∉ {DEVICE_BUSY, TARGET_BUSY}: rtn = SCSI_MLQUEUE_HOST_BUSY.
- return rtn.
- done: scsi_done(cmd); return 0.

REQ-7: queuecommand callback (LLD vtable):
- Signature: int (*queuecommand)(struct Scsi_Host *host, struct scsi_cmnd *cmd).
- Contract: take ownership of cmd, eventually call scsi_done(cmd) when LLD-completion arrives.
- Returns: 0 on accept; SCSI_MLQUEUE_DEVICE_BUSY / _TARGET_BUSY / _HOST_BUSY on transient backpressure → mid-layer reinserts; SCSI_MLQUEUE_EH_RETRY on EH-mediated retry.

REQ-8: scsi_done(cmd) / scsi_done_direct(cmd):
- /* LLD-completion entry; LLD calls from its ISR or bottom-half */
- internal: scsi_done_internal(cmd, complete_directly):
  - switch (cmd->submitter):
    - SUBMITTED_BY_BLOCK_LAYER: break.
    - SUBMITTED_BY_SCSI_ERROR_HANDLER: return scsi_eh_done(cmd) /* wakes EH */.
    - SUBMITTED_BY_SCSI_RESET_IOCTL: return.
  - if blk_should_fake_timeout(q): return /* fault-injection */.
  - if test_and_set_bit(SCMD_STATE_COMPLETE): return /* idempotent */.
  - trace_scsi_dispatch_cmd_done(cmd).
  - if complete_directly: blk_mq_complete_request_direct(req, scsi_complete).
  - else: blk_mq_complete_request(req) /* IPI to issuing CPU */.

REQ-9: scsi_complete(rq) — blk_mq_ops.complete:
- cmd = blk_mq_rq_to_pdu(rq).
- if blk_mq_is_reserved_rq(rq): scsi_mq_uninit_cmd(cmd); __blk_mq_end_request(rq, scsi_result_to_blk_status(result)); return.
- INIT_LIST_HEAD(&cmd->eh_entry).
- atomic_inc(&cmd->device->iodone_cnt).
- if cmd->result: atomic_inc(&cmd->device->ioerr_cnt).
- disposition = scsi_decide_disposition(cmd).
- if disposition != SUCCESS ∧ scsi_cmd_runtime_exceeced(cmd): disposition = SUCCESS /* give up; cmd outlived budget */.
- scsi_log_completion(cmd, disposition).
- switch (disposition):
  - SUCCESS: scsi_finish_command(cmd) /* → ULD ->done callback → scsi_io_completion */.
  - NEEDS_RETRY: scsi_queue_insert(cmd, SCSI_MLQUEUE_EH_RETRY).
  - ADD_TO_MLQUEUE: scsi_queue_insert(cmd, SCSI_MLQUEUE_DEVICE_BUSY).
  - default (FAILED): scsi_eh_scmd_add(cmd) /* escalate into per-host EH */.

REQ-10: scsi_io_completion(cmd, good_bytes):
- /* Called from ULD ->done (sd_done / sr_done / ...) after sense + good_bytes computed */
- result = cmd->result; blk_stat = BLK_STS_OK.
- if result: result = scsi_io_completion_nz_result(cmd, result, &blk_stat) /* normalize, recover, deferred-sense */.
- if blk_rq_bytes(req) > 0 ∨ blk_stat == BLK_STS_OK:
  - if !scsi_end_request(req, blk_stat, good_bytes): return /* fully done */.
- if blk_stat ∧ scsi_noretry_cmd(cmd):
  - if scsi_end_request(req, blk_stat, blk_rq_bytes(req)) WARN_ONCE.
  - return.
- if result == 0: scsi_mq_requeue_cmd(cmd, 0) /* leftover, no error: just requeue */.
- else: scsi_io_completion_action(cmd, result) /* sense-based action */.

REQ-11: scsi_io_completion_action(cmd, result):
- sense_valid = scsi_command_normalize_sense(cmd, &sshdr).
- sense_current = sense_valid ∧ !scsi_sense_is_deferred(&sshdr).
- blk_stat = scsi_result_to_blk_status(result).
- /* action ∈ {ACTION_FAIL, ACTION_REPREP, ACTION_DELAYED_REPREP, ACTION_RETRY, ACTION_DELAYED_RETRY} */
- if host_byte(result) == DID_RESET: action = RETRY.
- elif sense_valid ∧ sense_current:
  - switch (sshdr.sense_key):
    - UNIT_ATTENTION:
      - if removable: changed = 1; action = FAIL.
      - else: action = RETRY /* power-glitch / bus-reset */.
    - ILLEGAL_REQUEST:
      - if use_10_for_rw ∧ asc==0x20,ascq==0 ∧ (cmd[0]==READ_10∨WRITE_10): use_10_for_rw = 0; action = REPREP /* fall back to READ_6/WRITE_6 */.
      - elif asc==0x10: action = FAIL; blk_stat = BLK_STS_PROTECTION /* DIX */.
      - elif asc==0x20∨0x24: action = FAIL; blk_stat = BLK_STS_TARGET /* opcode/field invalid */.
      - else: action = FAIL.
    - ABORTED_COMMAND: action = FAIL; if asc==0x10: blk_stat = BLK_STS_PROTECTION /* DIF */.
    - NOT_READY:
      - if asc==0x04: per-ascq dispatch → DELAYED_RETRY (becoming ready / format / rebuild / spinup / sanitize / ...), DELAYED_REPREP (ALUA transition 0x0a), FAIL (depopulation 0x24/0x25).
      - else: action = FAIL.
    - VOLUME_OVERFLOW / COMPLETED / default: action = FAIL.
    - DATA_PROTECT: action = FAIL; insufficient-zone-resources (asc/ascq 0x0c/0x12 or 0x55 with 0x0e/0x0f) → BLK_STS_ZONE_OPEN_RESOURCE.
- else: action = FAIL.
- if action != FAIL ∧ scsi_cmd_runtime_exceeced(cmd): action = FAIL.
- switch (action):
  - FAIL: rate-limited log of sense + cmd; if !scsi_end_request(req, blk_stat, scsi_rq_err_bytes(req)) return; fallthrough to REPREP.
  - REPREP: scsi_mq_requeue_cmd(cmd, 0).
  - DELAYED_REPREP: scsi_mq_requeue_cmd(cmd, ALUA_TRANSITION_REPREP_DELAY).
  - RETRY: __scsi_queue_insert(cmd, SCSI_MLQUEUE_EH_RETRY, false).
  - DELAYED_RETRY: __scsi_queue_insert(cmd, SCSI_MLQUEUE_DEVICE_BUSY, false).

REQ-12: scsi_io_completion_nz_result(cmd, result, *blk_statp):
- sense_valid = scsi_command_normalize_sense(cmd, &sshdr).
- sense_current = sense_valid ∧ !scsi_sense_is_deferred(&sshdr).
- if blk_rq_is_passthrough(req):
  - if sense_valid: cmd->sense_len = min(8 + sense_buffer[7], SCSI_SENSE_BUFFERSIZE).
  - if sense_current: *blk_statp = scsi_result_to_blk_status(result).
- elif blk_rq_bytes(req) == 0 ∧ sense_current: *blk_statp = scsi_result_to_blk_status(result) /* FLUSH-class */.
- if sense_valid ∧ sshdr.sense_key == RECOVERED_ERROR:
  - if !(asc==0x0 ∧ ascq==0x1d /* ATA pass-through info */) ∧ !RQF_QUIET: scsi_print_sense(cmd).
  - result = 0; *blk_statp = BLK_STS_OK /* treat as success */.
- if (result & 0xff) ∧ scsi_status_is_good(result): result = 0; *blk_statp = BLK_STS_OK /* CONDITION_MET etc. */.
- return result.

REQ-13: scsi_end_request(req, error, bytes):
- cmd = blk_mq_rq_to_pdu(req); sdev = cmd->device; q = sdev->request_queue.
- if blk_update_request(req, error, bytes): return true /* still has bytes */.
- if q->limits.features & BLK_FEAT_ADD_RANDOM: add_disk_randomness(q->disk).
- cmd->flags = 0.
- destroy_rcu_head(&cmd->rcu).
- scsi_mq_uninit_cmd(cmd) /* scsi_free_sgtables + uninit_command vtable */.
- percpu_ref_get(&q->q_usage_counter).
- __blk_mq_end_request(req, error) /* frees rq + cmd pdu */.
- scsi_run_queue_async(sdev).
- percpu_ref_put(&q->q_usage_counter).
- return false.

REQ-14: scsi_result_to_blk_status(result):
- /* SCSI ML byte first */
- switch (scsi_ml_byte(result)):
  - SCSIML_STAT_OK: break.
  - _RESV_CONFLICT: return BLK_STS_RESV_CONFLICT.
  - _NOSPC: return BLK_STS_NOSPC.
  - _MED_ERROR: return BLK_STS_MEDIUM.
  - _TGT_FAILURE: return BLK_STS_TARGET.
  - _DL_TIMEOUT: return BLK_STS_DURATION_LIMIT.
- /* Host byte */
- switch (host_byte(result)):
  - DID_OK: return scsi_status_is_good(result) ? BLK_STS_OK : BLK_STS_IOERR.
  - DID_TRANSPORT_FAILFAST ∨ DID_TRANSPORT_MARGINAL: return BLK_STS_TRANSPORT.
  - default: return BLK_STS_IOERR.

REQ-15: scsi_alloc_sgtables(cmd):
- nr_segs = blk_rq_nr_phys_segments(rq).
- if WARN(!nr_segs): return BLK_STS_IOERR.
- need_drain = sdev->dma_drain_len ∧ passthrough ∧ !is_write ∧ hostt->dma_need_drain(rq).
- if need_drain: nr_segs++.
- if sg_alloc_table_chained(&cmd->sdb.table, nr_segs, cmd->sdb.table.sgl, SCSI_INLINE_SG_CNT): return BLK_STS_RESOURCE.
- count = __blk_rq_map_sg(rq, sdb.table.sgl, &last_sg).
- if blk_rq_bytes(rq) & q->limits.dma_pad_mask:
  - pad_len = (dma_pad_mask & ~blk_rq_bytes(rq)) + 1.
  - last_sg->length += pad_len; cmd->extra_len += pad_len.
- if need_drain:
  - sg_unmark_end(last_sg); last_sg = sg_next(last_sg).
  - sg_set_buf(last_sg, sdev->dma_drain_buf, sdev->dma_drain_len); sg_mark_end.
  - cmd->extra_len += sdev->dma_drain_len; count++.
- BUG_ON(count > sdb.table.nents); sdb.table.nents = count.
- sdb.length = blk_rq_payload_bytes(rq).
- if blk_integrity_rq(rq):
  - prot_sdb = cmd->prot_sdb /* WARN if NULL — host doesn't support DIX */.
  - sg_alloc_table_chained(&prot_sdb->table, rq->nr_integrity_segments, prot_sdb->table.sgl, SCSI_INLINE_PROT_SG_CNT) → on err: BLK_STS_RESOURCE.
  - count = blk_rq_map_integrity_sg(rq, prot_sdb->table.sgl).
  - prot_sdb->table.nents = count.
- return BLK_STS_OK.
- out_free_sgtables on err: scsi_free_sgtables(cmd); return ret.

REQ-16: scsi_mq_get_budget(q) / scsi_mq_put_budget(q, token):
- get: token = scsi_dev_queue_ready(q, sdev) /* returns sbitmap-allocated slot or -1 */.
- put: if sdev->budget_map.map: sbitmap_put(&sdev->budget_map, token).
- scsi_dev_queue_ready: enforces sdev->device_busy < queue_depth; sets cmd->budget_token; manages sdev->starved_list when starved.
- scsi_target_queue_ready: enforces atomic target_busy < target->can_queue (if > 0); per-single_lun blocking.
- scsi_host_queue_ready: enforces atomic host_busy < shost->can_queue; manages shost->starved_list.

REQ-17: scsi_init_hctx(hctx, data, hctx_idx) / scsi_map_queues(set):
- init_hctx: hctx->driver_data = shost /* per-hctx back-pointer to host */.
- map_queues: shost = container_of(set, Scsi_Host, tag_set); if hostt->map_queues: delegate; else blk_mq_map_queues(&set->map[HCTX_TYPE_DEFAULT]).

REQ-18: scsi_mq_setup_tags(shost):
- sgl_size = max(sizeof(scatterlist), inline_sgl_size(shost) /* = min(sg_tablesize, SCSI_INLINE_SG_CNT) * sizeof(sg) */).
- cmd_size = sizeof(scsi_cmnd) + hostt->cmd_size + sgl_size.
- if scsi_host_get_prot(shost): cmd_size += sizeof(scsi_data_buffer) + sizeof(sg) * SCSI_INLINE_PROT_SG_CNT.
- tag_set->ops = hostt->commit_rqs ? &scsi_mq_ops : &scsi_mq_ops_no_commit.
- tag_set->nr_hw_queues = shost->nr_hw_queues ?: 1.
- tag_set->nr_maps = shost->nr_maps ?: 1.
- tag_set->queue_depth = shost->can_queue + shost->nr_reserved_cmds.
- tag_set->reserved_tags = shost->nr_reserved_cmds.
- tag_set->cmd_size = cmd_size.
- tag_set->numa_node = dev_to_node(shost->dma_dev).
- if hostt->tag_alloc_policy_rr: BLK_MQ_F_TAG_RR.
- if shost->queuecommand_may_block: BLK_MQ_F_BLOCKING.
- if shost->host_tagset: BLK_MQ_F_TAG_HCTX_SHARED.
- return blk_mq_alloc_tag_set(tag_set).

REQ-19: scsi_mq_init_request / _exit_request (per-rq pdu life):
- init: cmd->sense_buffer = kmem_cache_alloc_node(scsi_sense_cache, GFP_KERNEL, numa_node) — fails → -ENOMEM.
- init: if scsi_host_get_prot: cmd->prot_sdb = (sg-after-cmd-inline) + inline_sgl_size.
- init: if hostt->init_cmd_priv: hostt->init_cmd_priv(shost, cmd) — on err: free sense_buffer.
- exit: if hostt->exit_cmd_priv: exit_cmd_priv(shost, cmd).
- exit: kmem_cache_free(scsi_sense_cache, cmd->sense_buffer).

REQ-20: scsi_init_limits(shost, lim):
- lim->max_segments = min(shost->sg_tablesize, SG_MAX_SEGMENTS).
- if scsi_host_prot_dma(shost): sg_prot_tablesize = min_not_zero(sg_prot_tablesize, SCSI_MAX_PROT_SG_SEGMENTS); BUG_ON(< sg_tablesize); lim->max_integrity_segments = sg_prot_tablesize.
- lim->max_hw_sectors = shost->max_sectors.
- lim->seg_boundary_mask = shost->dma_boundary.
- lim->max_segment_size = shost->max_segment_size.
- lim->virt_boundary_mask = shost->virt_boundary_mask.
- lim->dma_alignment = max(shost->dma_alignment, dma_get_cache_alignment() - 1).
- if dev->dma_parms: dma_set_seg_boundary(dev, dma_boundary); dma_set_max_seg_size(dev, max_segment_size).

REQ-21: scsi_queue_insert(cmd, reason) / __scsi_queue_insert(cmd, reason, unbusy):
- scsi_set_blocked(cmd, reason) /* set sdev/target/host blocked-flag matching reason */.
- if unbusy: scsi_device_unbusy(sdev, cmd) /* dec budget */.
- cmd->result = 0.
- blk_mq_requeue_request(req, kick = !scsi_host_in_recovery(host)) /* head-insert; auto-kick if not in EH */.

REQ-22: scsi_mq_requeue_cmd(cmd, msecs):
- /* Used for REPREP path: must drop sgtables before re-prep */
- if rq_flags & RQF_DONTPREP: clear RQF_DONTPREP; scsi_mq_uninit_cmd(cmd).
- else WARN_ON_ONCE (must be prepared).
- blk_mq_requeue_request(rq, false).
- if !scsi_host_in_recovery(host): blk_mq_delay_kick_requeue_list(q, msecs).

REQ-23: Sense-data parsing (scsi_command_normalize_sense / scsi_build_sense):
- scsi_command_normalize_sense(cmd, &shdr) /* defined in scsi_common.c */: parse cmd->sense_buffer[0..sense_len] into struct scsi_sense_hdr { response_code, sense_key, asc, ascq, byte4, byte5, byte6, additional_length }; supports fixed (0x70/0x71) and descriptor (0x72/0x73) formats.
- scsi_sense_is_deferred(&shdr): response_code ∈ {0x71, 0x73}.
- scsi_build_sense(scmd, desc, key, asc, ascq): construct a synthetic sense buffer in scmd->sense_buffer; set scmd->result with SAM_STAT_CHECK_CONDITION via scsi_build_sense_buffer; used by simulated EH paths.

REQ-24: scsi_run_queue / scsi_run_host_queues (queue restart):
- scsi_run_queue(q):
  - if !list_empty(&sdev->host->starved_list): scsi_starved_list_run(shost) /* re-eval host starved sdevs */.
  - if !shost->host_self_blocked: blk_mq_run_hw_queues(q, false).
- scsi_starved_list_run(shost):
  - for each sdev on shost->starved_list with available budget: list_del; blk_mq_run_hw_queues(sdev->request_queue, false).
- scsi_run_host_queues(shost): iter each sdev on shost; scsi_run_queue(sdev->request_queue).

REQ-25: scsi_device_quiesce / _resume:
- quiesce: set sdev_state = SDEV_QUIESCE; blk_mq_freeze_queue(q); drain pending; blk_mq_unfreeze_queue.
- resume: blk_mq_freeze_queue; set sdev_state = previous; blk_mq_unfreeze_queue.
- Used during runtime PM, target-state changes, and rescan.

REQ-26: scsi_eh integration (scsi_eh_inc_host_failed lives in drivers/scsi/scsi_error.c):
- After timeout (scsi_timeout) or EH-escalation (scsi_eh_scmd_add), the per-host EH thread is woken via call_rcu_hurry(&scmd->rcu, scsi_eh_inc_host_failed) — RCU-defer so the request is no longer in-flight when the EH counter is incremented; once host->host_failed reaches host->host_busy, the EH thread is woken to walk shost->eh_cmd_q.
- scsi_eh_done(cmd) is invoked when submitter == SUBMITTED_BY_SCSI_ERROR_HANDLER and the LLD finally completes an EH-injected command — wakes the EH thread waiting on a per-shost completion.

REQ-27: DMA mapping path (the per-bio → sg pipeline):
- blk-mq builds bio_vec list → __blk_rq_map_sg walks bios building scatterlist entries → scsi_alloc_sgtables stitches them into scmd->sdb.table.sgl and (optionally) appends a DMA-drain page → LLD calls scsi_dma_map(scmd) → dma_map_sg(shost->dma_dev, sgl, nents, dir) → DMA descriptors (LLD-private) → on completion, scsi_dma_unmap(scmd) → dma_unmap_sg(dma_dev, sgl, nents, dir).
- DMA-drain only on passthrough reads when hostt->dma_need_drain(rq) returns true and sdev->dma_drain_len > 0; drains tail bytes the device may have over-DMA-ed.

REQ-28: VPD helpers (scsi_vpd_lun_id / _serial / _tpg_id):
- scsi_vpd_lun_id(sdev, id, id_len): walk Device Identification VPD page 0x83 descriptors, pick highest-priority by designator_prio (NAA-IEEE-Registered-Extended > NAA-IEEE-Registered > EUI-64 > T10-Vendor-ID), format into id.
- scsi_vpd_lun_serial(sdev, sn, sn_size): read Unit Serial Number VPD page 0x80.
- scsi_vpd_tpg_id(sdev, *rel_id): read Target Port Group descriptor from page 0x83; output target-port-group + relative-target-port.

REQ-29: Internal command alloc (scsi_get_internal_cmd / scsi_put_internal_cmd):
- get: op = (dir == DMA_TO_DEVICE) ? REQ_OP_DRV_OUT : REQ_OP_DRV_IN; rq = scsi_alloc_request(sdev->request_queue, op, flags); scmd = blk_mq_rq_to_pdu(rq); scmd->device = sdev.
- put: blk_mq_free_request(blk_mq_rq_from_pdu(scmd)).
- Used by LLDs that need a back-channel cmd not driven by a fs/userspace request.

REQ-30: scsi_check_passthrough(scmd, failures):
- Matches scmd->result against scsi_failures array (per-execute-cmd caller-supplied retry policy).
- For each scsi_failure entry: compare host_byte, status_byte, sense_key, asc, ascq with wildcards SCMD_FAILURE_RESULT_ANY / _STAT_ANY / _SENSE_ANY / _ASC_ANY / _ASCQ_ANY.
- On match: if per-entry retries-budget OK or total-budget OK: return -EAGAIN (caller retries); else return 0 (give up).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `budget_token_balanced` | INVARIANT | per-queue_rq: every get_budget is paired with put_budget on every error path. |
| `dontprep_no_double_prepare` | INVARIANT | per-queue_rq: RQF_DONTPREP gates scsi_prepare_cmd to once-per-rq lifetime. |
| `state_complete_idempotent` | INVARIANT | per-done: test_and_set_bit(SCMD_STATE_COMPLETE) — second call short-circuits before blk_mq_complete_request. |
| `sense_buffer_lifetime_bound_to_pdu` | INVARIANT | per-mq_init/exit_request: sense_buffer allocated in init, freed in exit; never freed mid-flight. |
| `sgtable_alloc_paired_with_free` | INVARIANT | per-alloc_sgtables: every success → matching scsi_free_sgtables on path; error paths free partial allocation. |
| `dispatch_iorequest_cnt_balanced` | INVARIANT | per-dispatch_cmd: atomic_inc on entry; atomic_dec on every rejection / DID_NO_CONNECT path. |
| `submitter_disjoint_dispositions` | INVARIANT | per-scsi_done_internal: submitter ∈ {BLOCK_LAYER, SCSI_ERROR_HANDLER, SCSI_RESET_IOCTL} are mutually exclusive completion paths. |
| `runtime_exceeced_forces_fail` | INVARIANT | per-io_completion_action: scsi_cmd_runtime_exceeced(cmd) ⇒ action = FAIL. |
| `removable_unit_attention_fails` | INVARIANT | per-io_completion_action: UA on removable ⇒ FAIL ∧ changed = 1. |
| `recovered_error_zeroes_blk_stat` | INVARIANT | per-io_completion_nz: sense_key == RECOVERED_ERROR ⇒ result = 0 ∧ *blk_statp = OK. |
| `cmd_size_includes_inline_sg` | INVARIANT | per-mq_setup_tags: tag_set.cmd_size ≥ sizeof(scsi_cmnd) + hostt.cmd_size + inline_sgl_size. |

### Layer 2: TLA+

`drivers/scsi/scsi-lib.tla`:
- Per-rq lifecycle: prepare → dispatch → LLD-queuecommand → LLD-done → scsi_done → complete → disposition → ULD-done → io_completion → end_request | requeue | queue_insert | eh_scmd_add.
- Properties:
  - `safety_one_done_per_rq` — per-cmd: SCMD_STATE_COMPLETE bit set exactly once before __blk_mq_end_request.
  - `safety_budget_token_lifetime` — per-rq: get_budget → start_request → put_budget on completion-or-error path; never leaked.
  - `safety_dontprep_idempotent` — per-rq: RQF_DONTPREP prevents re-running scsi_prepare_cmd after a requeue (unless scsi_mq_requeue_cmd clears it).
  - `safety_no_dispatch_in_recovery` — per-host scsi_host_in_recovery() ∧ !FAIL_IF_RECOVERING ⇒ queue_rq returns RESOURCE without dispatch.
  - `safety_eh_path_excludes_block_layer` — per-cmd: submitter == ERROR_HANDLER ⇒ scsi_done routes to scsi_eh_done, not blk_mq_complete_request.
  - `liveness_per_rq_eventually_completes` — per-rq: any submitted rq either reaches end_request or escalates to scsi_eh_scmd_add (which the EH layer must then complete or abort).
  - `liveness_starved_list_progress` — per-shost: starved sdev is eventually dispatched once budget is released.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cmnd::queue_rq` post: returns Ok ⇒ blk_mq_start_request called ∧ either LLD owns cmd or requeue scheduled | `Cmnd::queue_rq` |
| `Cmnd::prepare` post: cmd.sdb.table.sgl points to inline-sg area; sc_data_direction matches rq | `Cmnd::prepare` |
| `Cmnd::dispatch` post: rtn == 0 ⇒ queuecommand returned 0; rtn != 0 ⇒ iorequest_cnt restored | `Cmnd::dispatch` |
| `Cmnd::done` post: state.bit(COMPLETE) set; blk_mq_complete_request scheduled once | `Cmnd::done` |
| `Cmnd::io_completion` post: ∑(good_bytes across calls) + failed-bytes ≤ blk_rq_bytes(req) | `Cmnd::io_completion` |
| `Cmnd::alloc_sgtables` post: sdb.table.nents == number-of-mapped-physical-segs (+1 if drain) | `Cmnd::alloc_sgtables` |
| `Host::setup_tags` post: tag_set.queue_depth == can_queue + nr_reserved_cmds | `Host::setup_tags` |
| `Cmnd::result_to_blk_status` total: every (scsi_ml_byte, host_byte) pair mapped | `Cmnd::result_to_blk_status` |
| `SenseHdr::from_cmnd` post: response_code-format determines fixed vs descriptor parse | `SenseHdr::from_cmnd` |

### Layer 4: Verus/Creusot functional

`Per-rq submit → blk-mq queue_rq → budget gate → prepare (ULD ->init_command) → start_request → dispatch (LLD ->queuecommand) → LLD-done (scsi_done) → complete → disposition (success | retry | EH) → ULD ->done → io_completion (sense-decode + retry-action) → end_request | requeue` semantic equivalence: per-Documentation/scsi/scsi_mid_low_api.rst + Documentation/block/blk-mq.rst.

### hardening

(Inherits row-1 features from `drivers/scsi/00-overview.md` § Hardening and `drivers/scsi/scsi-mid-core.md` § Hardening.)

scsi_lib reinforcement:

- **Per-budget-token strict get/put on every error path** — defense against per-budget leak that would starve the sdev queue.
- **Per-RQF_DONTPREP gate on scsi_prepare_cmd** — defense against per-double-init that would corrupt sgtable or LLD-private state.
- **Per-SCMD_STATE_COMPLETE test_and_set in scsi_done** — defense against per-double-completion racing LLD-ISR with timeout.
- **Per-sense-buffer allocation tied to mq pdu lifetime** — defense against per-UAF where sense decoded after blk_mq_free_request.
- **Per-sg-table free on every alloc_sgtables error path** — defense against per-sgtable leak draining the chained-sg pool.
- **Per-iorequest_cnt symmetric inc/dec in dispatch_cmd** — defense against per-counter drift that would cause LRU mis-throttling.
- **Per-SCMD_FAIL_IF_RECOVERING surfacing BLK_STS_OFFLINE** — defense against per-stuck-cmd during EH that would dead-lock multipath fail-fast.
- **Per-tag_set.cmd_size including inline sg + prot sg** — defense against per-OOB-write of sgtable into next pdu.
- **Per-host_self_blocked gate in scsi_run_queue** — defense against per-LLD-flushing-while-blocked re-entry.
- **Per-scsi_check_passthrough total-retries budget** — defense against per-infinite-retry on chronic ASC/ASCQ.
- **Per-DMA-drain extra sg-entry accounted in extra_len** — defense against per-DMA-overrun on hostt->dma_need_drain devices.
- **Per-result-byte mapping via SCSIML / host_byte precedence** — defense against per-mis-classification of NOSPC / RESV_CONFLICT into generic IOERR.
- **Per-RCU-defer on scsi_eh_inc_host_failed** — defense against per-EH-counter increment racing with blk_mq_complete_request finishing the rq.
- **Per-runtime-exceeced override to FAIL** — defense against per-cmd outliving SD_TIMEOUT * retry budget.

