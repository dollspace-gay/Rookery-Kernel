# Tier-3: drivers/vhost/scsi.c — vhost-scsi: virtio-scsi guest backend in host kernel

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/vhost/00-overview.md
upstream-paths:
  - drivers/vhost/scsi.c (~2993 lines)
  - drivers/vhost/vhost.h
  - include/uapi/linux/vhost.h (VHOST_SCSI_SET_ENDPOINT, _CLEAR_ENDPOINT, _GET_ABI_VERSION, _SET_EVENTS_MISSED, _GET_EVENTS_MISSED)
  - include/uapi/linux/virtio_scsi.h (struct virtio_scsi_cmd_req, virtio_scsi_cmd_req_pi, virtio_scsi_cmd_resp, virtio_scsi_ctrl_tmf_req, virtio_scsi_ctrl_an_req, virtio_scsi_event)
  - include/target/target_core_base.h (struct se_cmd, se_session, se_portal_group, se_lun, se_wwn)
  - include/target/target_core_fabric.h (struct target_core_fabric_ops, target_init_cmd, target_submit_prep, target_submit, target_submit_tmr, target_setup_session, target_remove_session, target_register_template)
-->

## Summary

**vhost-scsi** is the kernel-side server for the virtio-scsi device model. A KVM/QEMU guest exposes a virtio-scsi controller with three categories of virtqueues — **control** (index `VHOST_SCSI_VQ_CTL = 0`: TMF + AN), **event** (`VHOST_SCSI_VQ_EVT = 1`: hotplug/hotunplug AEN injection), and **I/O** (`VHOST_SCSI_VQ_IO = 2..N`: per-CPU command queues, up to `VHOST_SCSI_MAX_IO_VQ = 1024`). The userspace VMM opens `/dev/vhost-scsi`, programs the rings, then ioctl's `VHOST_SCSI_SET_ENDPOINT` with a `struct vhost_scsi_target { vhost_wwpn, vhost_tpgt }` — vhost-scsi looks up a matching `vhost_scsi_tpg` on the global `vhost_scsi_list` (populated by configfs at `/sys/kernel/config/target/vhost/<wwpn>/tpgt_N/`) and installs it as the per-VQ backend. Each guest virtio-scsi request lands at `vhost_scsi_handle_kick` → `vhost_scsi_handle_vq` (I/O VQ) or `vhost_scsi_ctl_handle_kick` → `vhost_scsi_ctl_handle_vq` (control), which dequeue with `vhost_get_vq_desc`, parse `virtio_scsi_cmd_req[_pi]`, allocate a preallocated `vhost_scsi_cmd` from `sbitmap scsi_tags`, map the guest payload iovecs into a host scatterlist via `vhost_scsi_mapal` (`get_user_pages` + `sg_init_table`), and submit to the LIO target stack with `target_init_cmd` + `target_submit_prep` + `target_submit`. The LIO backstore (file/iblock/pscsi/rbd) executes the CDB and calls back via `target_core_fabric_ops` — `queue_data_in` / `queue_status` / `queue_tm_rsp` / `release_cmd` — which enqueue on a per-VQ `completion_list` llist; `vhost_scsi_complete_cmd_work` (vhost worker context) drains it, builds `virtio_scsi_cmd_resp` with status/sense/residual, copies guest read buffers from host SGL via `vhost_scsi_copy_sgl_to_iov`, and calls `vhost_add_used_and_signal`. Per-T10-PI (`VIRTIO_SCSI_F_T10_PI`): the `_pi` request variant carries `pi_bytesout`/`pi_bytesin`, mapped to a separate `prot_sgl` and passed through `target_init_cmd`'s `TARGET_PROT_*` flags. Per-RESERVATION: PERSISTENT RESERVE OUT/IN CDBs flow through unchanged — LIO's `target_core_pr.c` handles SPC-3/4 reservations transparently. Per-TMF (`VIRTIO_SCSI_T_TMF_LOGICAL_UNIT_RESET`): `vhost_scsi_handle_tmf` allocates a `vhost_scsi_tmf` and dispatches via `target_submit_tmr`. Per-hotplug (`VIRTIO_SCSI_F_HOTPLUG`): `vhost_scsi_port_link` / `_port_unlink` (LIO `fabric_post_link`/`fabric_pre_unlink` callbacks) push `VIRTIO_SCSI_T_TRANSPORT_RESET` events to the event VQ via `vhost_scsi_send_evt`. Per-LIO-integration: vhost-scsi registers as a TCM fabric module with `target_register_template(&vhost_scsi_ops)` so that mkdir-ing `/sys/kernel/config/target/vhost/<wwpn>` calls `vhost_scsi_make_tport` → `vhost_scsi_make_tpg` → `vhost_scsi_make_nexus` (creates the I_T nexus session via `target_setup_session`). Critical for: zero-copy virt SCSI throughput, T10 protection-information passthrough to NVMe-backed storage, persistent reservations for clustered guests (Windows Server Failover Clustering on virtio-scsi LUNs).

This Tier-3 covers `drivers/vhost/scsi.c` (~2993 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vhost_scsi` | per-/dev/vhost-scsi fd device | `VhostScsi` |
| `struct vhost_scsi_virtqueue` | per-VQ (CTL/EVT/IO) wrapper + sbitmap tags + completion llist | `VhostScsiVirtqueue` |
| `struct vhost_scsi_cmd` | per-cmd descriptor (SGL + prot_sgl + se_cmd) | `VhostScsiCmd` |
| `struct vhost_scsi_tmf` | per-TMR descriptor | `VhostScsiTmf` |
| `struct vhost_scsi_evt` | per-AEN event | `VhostScsiEvt` |
| `struct vhost_scsi_tpg` | per-TCM Target Portal Group (configfs) | `VhostScsiTpg` |
| `struct vhost_scsi_tport` | per-WWPN port (configfs) | `VhostScsiTport` |
| `struct vhost_scsi_nexus` | per-I_T nexus (LIO se_session wrapper) | `VhostScsiNexus` |
| `struct vhost_scsi_inflight` | per-VQ inflight kref for flush | `VhostScsiInflight` |
| `struct vhost_scsi_ctx` | per-iteration parse context | `VhostScsiCtx` |
| `struct vhost_scsi_target` | per-`VHOST_SCSI_SET_ENDPOINT` argp (UAPI) | shared UAPI |
| `vhost_scsi_open()` | per-`/dev/vhost-scsi` open | `VhostScsi::open` |
| `vhost_scsi_release()` | per-fd close | `VhostScsi::release` |
| `vhost_scsi_ioctl()` | per-ioctl dispatch | `VhostScsi::ioctl` |
| `vhost_scsi_set_endpoint()` | per-`VHOST_SCSI_SET_ENDPOINT` | `VhostScsi::set_endpoint` |
| `vhost_scsi_clear_endpoint()` | per-`VHOST_SCSI_CLEAR_ENDPOINT` | `VhostScsi::clear_endpoint` |
| `vhost_scsi_set_features()` | per-`VHOST_SET_FEATURES` | `VhostScsi::set_features` |
| `vhost_scsi_handle_kick()` | per-IO VQ worker entry | `VhostScsi::handle_kick` |
| `vhost_scsi_handle_vq()` | per-IO VQ drain loop | `VhostScsi::handle_vq` |
| `vhost_scsi_ctl_handle_kick()` | per-CTL VQ worker entry | `VhostScsi::ctl_handle_kick` |
| `vhost_scsi_ctl_handle_vq()` | per-CTL VQ drain loop | `VhostScsi::ctl_handle_vq` |
| `vhost_scsi_evt_handle_kick()` | per-EVT VQ worker entry | `VhostScsi::evt_handle_kick` |
| `vhost_scsi_handle_tmf()` | per-TMR submission | `VhostScsi::handle_tmf` |
| `vhost_scsi_send_an_resp()` | per-AN response | `VhostScsi::send_an_resp` |
| `vhost_scsi_get_desc()` | per-vhost_get_vq_desc wrapper | `VhostScsi::get_desc` |
| `vhost_scsi_chk_size()` | per-iovec size validation | `VhostScsi::chk_size` |
| `vhost_scsi_get_req()` | per-req header copy + tpg lookup | `VhostScsi::get_req` |
| `vhost_scsi_get_cmd()` | per-tag allocation from sbitmap | `VhostScsi::get_cmd` |
| `vhost_scsi_mapal()` | per-guest iovec → host SGL mapping | `VhostScsi::mapal` |
| `vhost_scsi_iov_to_sgl()` | per-iovec → SG entries via GUP | `VhostScsi::iov_to_sgl` |
| `vhost_scsi_copy_iov_to_sgl()` | per-copy fallback for unaligned iovecs | `VhostScsi::copy_iov_to_sgl` |
| `vhost_scsi_copy_sgl_to_iov()` | per-host SGL → guest read iovec | `VhostScsi::copy_sgl_to_iov` |
| `vhost_scsi_target_queue_cmd()` | per-LIO submission entry | `VhostScsi::target_queue_cmd` |
| `vhost_scsi_to_tcm_attr()` | per-virtio task_attr → TCM_*_TAG | `VhostScsi::to_tcm_attr` |
| `vhost_scsi_complete_cmd_work()` | per-VQ completion drain worker | `VhostScsi::complete_cmd_work` |
| `vhost_scsi_release_cmd()` | per-LIO release_cmd fabric op | `VhostScsi::release_cmd` |
| `vhost_scsi_release_cmd_res()` | per-cmd resource teardown (SG + tags) | `VhostScsi::release_cmd_res` |
| `vhost_scsi_drop_cmds()` | per-flush draining of completion_list | `VhostScsi::drop_cmds` |
| `vhost_scsi_release_tmf_res()` | per-TMF teardown | `VhostScsi::release_tmf_res` |
| `vhost_scsi_write_pending()` | per-DMA_TO_DEVICE LIO fabric op | `VhostScsi::write_pending` |
| `vhost_scsi_queue_data_in()` | per-DMA_FROM_DEVICE LIO fabric op | `VhostScsi::queue_data_in` |
| `vhost_scsi_queue_status()` | per-status LIO fabric op | `VhostScsi::queue_status` |
| `vhost_scsi_queue_tm_rsp()` | per-TMR response fabric op | `VhostScsi::queue_tm_rsp` |
| `vhost_scsi_aborted_task()` | per-aborted LIO fabric op (no-op) | `VhostScsi::aborted_task` |
| `vhost_scsi_check_stop_free()` | per-LIO stop+free check | `VhostScsi::check_stop_free` |
| `vhost_scsi_send_evt()` | per-event queue (hotplug AEN) | `VhostScsi::send_evt` |
| `vhost_scsi_allocate_evt()` | per-event allocation w/ MAX_EVENT cap | `VhostScsi::allocate_evt` |
| `vhost_scsi_do_evt_work()` | per-event delivery to guest EVT VQ | `VhostScsi::do_evt_work` |
| `vhost_scsi_complete_events()` | per-EVT drain | `VhostScsi::complete_events` |
| `vhost_scsi_evt_work()` | per-EVT worker entry | `VhostScsi::evt_work` |
| `vhost_scsi_hotplug()` / `_hotunplug()` | per-LUN add/remove plug | `VhostScsi::hotplug` / `hotunplug` |
| `vhost_scsi_send_status()` | per-error status response | `VhostScsi::send_status` |
| `vhost_scsi_send_bad_target()` | per-BAD_TARGET response | `VhostScsi::send_bad_target` |
| `vhost_scsi_send_tmf_resp()` | per-TMR response writer | `VhostScsi::send_tmf_resp` |
| `vhost_scsi_tmf_resp_work()` | per-TMR completion worker | `VhostScsi::tmf_resp_work` |
| `vhost_scsi_tmf_flush_work()` | per-TMR flush worker | `VhostScsi::tmf_flush_work` |
| `vhost_scsi_init_inflight()` / `_get_inflight()` / `_put_inflight()` / `_done_inflight()` | per-VQ flush refcount | `VhostScsi::*_inflight` |
| `vhost_scsi_flush()` | per-dev flush (wait inflight ⟶ 0) | `VhostScsi::flush` |
| `vhost_scsi_setup_vq_cmds()` | per-VQ pre-alloc `scsi_cmds[]` + `sbitmap_init` | `VhostScsi::setup_vq_cmds` |
| `vhost_scsi_destroy_vq_cmds()` | per-VQ cmd-array teardown | `VhostScsi::destroy_vq_cmds` |
| `vhost_scsi_make_tport()` / `_drop_tport()` | per-`/sys/kernel/config/target/vhost/<wwpn>/` configfs | `VhostScsi::make_tport` / `drop_tport` |
| `vhost_scsi_make_tpg()` / `_drop_tpg()` | per-`tpgt_N` configfs subdir | `VhostScsi::make_tpg` / `drop_tpg` |
| `vhost_scsi_make_nexus()` / `_drop_nexus()` | per-I_T nexus init (`target_setup_session`) | `VhostScsi::make_nexus` / `drop_nexus` |
| `vhost_scsi_port_link()` / `_port_unlink()` | per-LUN-symlink LIO callback | `VhostScsi::port_link` / `port_unlink` |
| `vhost_scsi_get_fabric_wwn()` / `_get_tpgt()` | per-LIO fabric introspection | `VhostScsi::get_fabric_wwn` / `get_tpgt` |
| `vhost_scsi_check_prot_fabric_only()` | per-T10-PI gating | `VhostScsi::check_prot_fabric_only` |
| `vhost_scsi_log_write()` / `_copy_cmd_log()` | per-`VHOST_F_LOG_ALL` write tracking | `VhostScsi::log_write` / `copy_cmd_log` |
| `vhost_buf_to_lun()` | per-virtio LUN bytes → u16 | `VhostScsi::buf_to_lun` |
| `vhost_scsi_ops` (`target_core_fabric_ops`) | per-LIO registration vtable | `VHOST_SCSI_OPS` |

## Compatibility contract

REQ-1: struct vhost_scsi:
- `dev`: per-`vhost_dev` (vhost framework device).
- `vqs`: per-array of `nvqs` `vhost_scsi_virtqueue` (CTL + EVT + 1..N IO).
- `vs_tpg`: per-`VHOST_SCSI_MAX_TARGET = 256` array of `vhost_scsi_tpg *` indexed by target byte.
- `vs_vhost_wwpn`: per-active endpoint WWPN (set by `SET_ENDPOINT`).
- `old_inflight`: per-flush snapshot of per-VQ inflight pointers.
- `vs_event_work`: per-evt injection work.
- `vs_event_list`: per-llist of pending `vhost_scsi_evt`.
- `vs_events_missed`: per-overflow flag.
- `vs_events_nr`: per-pending event count (`≤ VHOST_SCSI_MAX_EVENT = 128`).
- `inline_sg_cnt`: per-tunable inline SG count (`VHOST_SCSI_PREALLOC_SGLS` default, settable via module param `inline_sg_cnt`).

REQ-2: struct vhost_scsi_virtqueue:
- `vq`: per-base `vhost_virtqueue`.
- `vs`: per-back-pointer.
- `inflights[2]`: per-double-buffered inflight refcount (kref + completion).
- `inflight_idx`: per-active idx ∈ {0,1}.
- `scsi_cmds`: per-preallocated `vhost_scsi_cmd[max_cmds]`.
- `scsi_tags`: per-`sbitmap` over `scsi_cmds[]` indices.
- `max_cmds`: per-VQ depth (= `vq->num`).
- `upages`: per-VQ scratch page-array for GUP (`UIO_MAXIOV` entries).
- `completion_work`: per-VQ completion worker.
- `completion_list`: per-VQ llist of completed `vhost_scsi_cmd`.

REQ-3: struct vhost_scsi_cmd:
- `tvc_vq_desc`: per-`vhost_get_vq_desc` head index (for `vhost_add_used`).
- `tvc_sgl_count`, `tvc_prot_sgl_count`: per-SG entry counts.
- `copied_iov`: per-flag — set if pages were allocated (`alloc_page` fallback for misaligned iovec); cleared if GUP-pinned.
- `read_iov` / `read_iter`: per-saved DMA_FROM_DEVICE iovec for late copy.
- `sgl` / `table`: per-data SGL (chained via `sg_alloc_table_chained` with `inline_sg_cnt` inline + spill).
- `prot_sgl` / `prot_table`: per-T10-PI SGL.
- `tvc_resp_iov` + `tvc_resp_iovs` + `tvc_resp_iovs_cnt`: per-saved response iovecs.
- `tvc_vq`: per-back-pointer to VQ.
- `tvc_se_cmd`: per-embedded LIO `se_cmd` (container_of-able).
- `tvc_sense_buf[TRANSPORT_SENSE_BUFFER]`: per-sense buffer.
- `tvc_log` + `tvc_log_num`: per-`VHOST_F_LOG_ALL` log entries (NULL unless logging enabled).
- `tvc_completion_list`: per-llist node for completion drain.
- `inflight`: per-back-ref to active `vhost_scsi_inflight`.

REQ-4: struct vhost_scsi_tpg:
- `tport_tpgt`: per-TPG tag (u16, ≤ `VHOST_SCSI_MAX_TARGET`).
- `tv_tpg_port_count`: per-LUN-link count.
- `tv_tpg_vhost_count`: per-active SET_ENDPOINT count (gates removal).
- `tv_fabric_prot_type`: per-T10-PI fabric type (0 / 1 / 3).
- `tv_tpg_list`: per-`vhost_scsi_list` node.
- `tv_tpg_mutex`: per-TPG lock (covers `tpg_nexus` + `vhost_scsi`).
- `tpg_nexus`: per-active I_T nexus (NULL = no initiator port set).
- `tport`: per-back-ref to WWPN port.
- `se_tpg`: per-LIO `se_portal_group` embedded.
- `vhost_scsi`: per-back-ref to bound `vhost_scsi` (NULL = no endpoint).

REQ-5: struct vhost_scsi_tport:
- `tport_proto_id`: per-SCSI_PROTOCOL_SAS / FCP / ISCSI.
- `tport_wwpn`: per-numeric WWPN.
- `tport_name[VHOST_SCSI_NAMELEN]`: per-string WWN.
- `tport_wwn`: per-LIO `se_wwn` embedded.

REQ-6: VhostScsi::open(file):
- /* Allocate vs + per-VQ array */
- `vs = kvzalloc`.
- `vs.inline_sg_cnt = vhost_scsi_inline_sg_cnt`.
- /* Clamp nvqs */
- `nvqs = vhost_scsi_max_io_vqs` clamped to (1, `VHOST_SCSI_MAX_IO_VQ`), then `nvqs += VHOST_SCSI_VQ_IO`.
- `vs.old_inflight = kmalloc_array(nvqs)`.
- `vs.vqs = kmalloc_array(nvqs)`.
- `vqs[VHOST_SCSI_VQ_CTL].handle_kick = vhost_scsi_ctl_handle_kick`.
- `vqs[VHOST_SCSI_VQ_EVT].handle_kick = vhost_scsi_evt_handle_kick`.
- For each IO VQ: `vqs[i].handle_kick = vhost_scsi_handle_kick`, init `completion_work` + `completion_list`.
- `vhost_work_init(vs_event_work, vhost_scsi_evt_work)`.
- `vhost_dev_init(vs.dev, vqs, nvqs, UIO_MAXIOV, VHOST_SCSI_WEIGHT=256, 0, true, NULL)`.
- `vhost_scsi_init_inflight(vs, NULL)`.
- `f.private_data = vs`.
- return 0.

REQ-7: VhostScsi::set_endpoint(t):
- Lock `vs.dev.mutex`.
- For each VQ: `vhost_vq_access_ok` must succeed.
- Reject if `vs.vs_tpg` already set (-EEXIST).
- Alloc `vs_tpg` array of `VHOST_SCSI_MAX_TARGET` `vhost_scsi_tpg *`.
- Lock `vhost_scsi_mutex`; iterate `vhost_scsi_list`:
  - For each tpg with `tpg.tpg_nexus != NULL` ∧ `tpg.tv_tpg_vhost_count == 0` ∧ `strcmp(tport.tport_name, t.vhost_wwpn) == 0`:
    - `target_depend_item(&se_tpg.tpg_group.cg_item)` — pin configfs.
    - `tpg.tv_tpg_vhost_count++`.
    - `tpg.vhost_scsi = vs`.
    - `vs_tpg[tpg.tport_tpgt] = tpg`.
- If no match: free `vs_tpg`; return -ENODEV.
- For each IO VQ that is setup: `vhost_scsi_setup_vq_cmds(vq, vq.num)` — allocates `scsi_cmds[]` + SG-tables + `sbitmap_init`.
- For each VQ: under `vq.mutex`, `vhost_vq_set_backend(vq, vs_tpg)` + `vhost_vq_init_access(vq)`.
- `vhost_scsi_flush(vs)` — synchronize-rcu equivalent.
- `vs.vs_tpg = vs_tpg`.
- return 0.

REQ-8: VhostScsi::clear_endpoint(t):
- Lock `vs.dev.mutex`.
- For each VQ: `vhost_vq_access_ok`.
- If `!vs.vs_tpg`: return 0.
- Validate every populated `vs.vs_tpg[i].tport.tport_name == t.vhost_wwpn` (-EINVAL else).
- For each VQ: under `vq.mutex`, `vhost_vq_set_backend(vq, NULL)` — stop new cmds.
- `vhost_scsi_flush(vs)` — drain inflight.
- For each VQ: `vhost_scsi_destroy_vq_cmds(vq)`.
- For each `vs.vs_tpg[i]` non-NULL: `tpg.tv_tpg_vhost_count--`; `tpg.vhost_scsi = NULL`; `target_undepend_item(&se_tpg.tpg_group.cg_item)`.
- `vhost_scsi_flush(vs)` — final.
- Free `vs.vs_tpg`; clear `vs_vhost_wwpn`; `WARN_ON(vs_events_nr)`.

REQ-9: VhostScsi::handle_vq(vs, vq) — IO command pipeline:
- Lock `vq.mutex`.
- `vs_tpg = vhost_vq_get_backend(vq)`; if NULL: bail.
- `vhost_disable_notify(vs.dev, vq)`.
- `vq_log = vhost_has_feature(VHOST_F_LOG_ALL) ? vq.log : NULL`.
- Loop until weight or -ENXIO:
  - `vhost_scsi_get_desc(vs, vq, &vc, vq_log, &log_num)`.
  - If `vhost_has_feature(vq, VIRTIO_SCSI_F_T10_PI)`: parse `virtio_scsi_cmd_req_pi`; else `virtio_scsi_cmd_req`.
  - `vhost_scsi_chk_size(vq, &vc)` — request/response sizes vs out/in iovec totals.
  - `vhost_scsi_get_req(vq, &vc, &tpg)` — `copy_from_iter_full(req)`, validate `vc.lunp == 1` (virtio-scsi requires `lun[0] == 1`), READ_ONCE `tpg = vs_tpg[vc.target]`.
  - Compute `data_direction`: `out_size > req_size ⟹ DMA_TO_DEVICE`; `in_size > rsp_size ⟹ DMA_FROM_DEVICE`; else `DMA_NONE`.
  - If T10-PI ∧ `pi_bytesout` or `pi_bytesin`: truncate `prot_iter` to prot_bytes; advance `data_iter` past prot.
  - Validate `scsi_command_size(cdb) ≤ VHOST_SCSI_MAX_CDB_SIZE = 32`.
  - `nexus = tpg.tpg_nexus`; if NULL: -EIO.
  - `cmd = vhost_scsi_get_cmd(vq, tag)` — `sbitmap_get(scsi_tags)` then return `&scsi_cmds[tag]`; -ENOMEM ⟹ TASK_SET_FULL.
  - `vhost_scsi_setup_resp_iovs(cmd, &vq.iov[vc.out], vc.in)` — copy response iovecs (single-iovec fast path uses embedded `tvc_resp_iov`).
  - If `vq_log && log_num`: `vhost_scsi_copy_cmd_log(vq, cmd, vq_log, log_num)`.
  - If `data_direction != DMA_NONE`: `vhost_scsi_mapal(vs, cmd, prot_bytes, &prot_iter, exp_data_len, &data_iter, data_direction)`.
  - `cmd.tvc_vq_desc = vc.head`.
  - `vhost_scsi_target_queue_cmd(nexus, cmd, cdb, lun, task_attr, data_direction, exp_data_len + prot_bytes)`.
- Error cases:
  - -ENXIO ⟹ break (no more requests).
  - -EIO ⟹ `vhost_scsi_send_bad_target(vs, vq, &vc, TYPE_IO_CMD)`.
  - -ENOMEM ⟹ `vhost_scsi_send_status(vs, vq, &vc, SAM_STAT_TASK_SET_FULL)`.
- Unlock `vq.mutex`.

REQ-10: VhostScsi::target_queue_cmd(nexus, cmd, cdb, lun, task_attr, data_dir, exp_data_len):
- `se_cmd = &cmd.tvc_se_cmd`.
- If `cmd.tvc_sgl_count`: `sg_ptr = cmd.table.sgl`; if `tvc_prot_sgl_count`: `sg_prot_ptr = cmd.prot_table.sgl`; else `se_cmd.prot_pto = true`.
- Else: `sg_ptr = NULL`.
- `se_cmd.tag = 0`.
- `target_init_cmd(se_cmd, nexus.tvn_se_sess, cmd.tvc_sense_buf, lun, exp_data_len, vhost_scsi_to_tcm_attr(task_attr), data_dir, TARGET_SCF_ACK_KREF)`.
- `target_submit_prep(se_cmd, cdb, sg_ptr, tvc_sgl_count, NULL, 0, sg_prot_ptr, tvc_prot_sgl_count, GFP_KERNEL)` — failure path returns silently (LIO already completed cmd).
- `target_submit(se_cmd)`.

REQ-11: VhostScsi::complete_cmd_work(work) — completion drain:
- `llnode = llist_del_all(svq.completion_list)`.
- Lock `svq.vq.mutex`.
- For each cmd in llnode:
  - Build `virtio_scsi_cmd_resp v_rsp` from `se_cmd.scsi_status`, `residual_count`, `scsi_sense_length`, `tvc_sense_buf`.
  - If `cmd.read_iter` (misaligned read fallback): `vhost_scsi_copy_sgl_to_iov(cmd)` — `copy_page_to_iter` per SG entry; if fail, set response `VIRTIO_SCSI_S_BAD_TARGET`.
  - `copy_to_iter(&v_rsp, sizeof v_rsp, &iov_iter)` to `cmd.tvc_resp_iovs`.
  - `vhost_add_used(cmd.tvc_vq, cmd.tvc_vq_desc, 0)`.
  - `vhost_scsi_log_write(cmd.tvc_vq, cmd.tvc_log, cmd.tvc_log_num)`.
  - `vhost_scsi_release_cmd_res(se_cmd)`.
- Unlock; if any succeeded: `vhost_signal(svq.vs.dev, svq.vq)`.

REQ-12: VhostScsi::release_cmd(se_cmd) — LIO fabric op:
- If `se_cmd.se_cmd_flags & SCF_SCSI_TMR_CDB`: `schedule_work(tmf.flush_work)`.
- Else: `llist_add(cmd.tvc_completion_list, svq.completion_list)`; if `!vhost_vq_work_queue(svq.vq, svq.completion_work)` (dev gone): `vhost_scsi_drop_cmds(svq)` — synchronous local drain.

REQ-13: VhostScsi::release_cmd_res(se_cmd):
- Walk `cmd.table.sgl`: if `copied_iov` ⟹ `__free_page(page)`; else `put_page(page)` (GUP).
- `sg_free_table_chained(&cmd.table, vs.inline_sg_cnt)`.
- Walk `cmd.prot_table.sgl` similarly.
- If `tvc_resp_iovs != &tvc_resp_iov`: `kfree(tvc_resp_iovs)`.
- `sbitmap_clear_bit(&svq.scsi_tags, se_cmd.map_tag)`.
- `vhost_scsi_put_inflight(cmd.inflight)`.

REQ-14: VhostScsi::ctl_handle_vq(vs, vq) — control pipeline:
- Lock `vq.mutex`.
- Backend NULL ⟹ bail.
- `vhost_disable_notify`.
- Loop:
  - `vhost_scsi_get_desc`.
  - Peek `v_req.type` (`__virtio32`) via `copy_from_iter_full`.
  - Switch `vhost32_to_cpu(vq, v_req.type)`:
    - `VIRTIO_SCSI_T_TMF` ⟹ `vc.req_size = sizeof virtio_scsi_ctrl_tmf_req`; `vc.rsp_size = sizeof virtio_scsi_ctrl_tmf_resp`.
    - `VIRTIO_SCSI_T_AN_QUERY` / `_AN_SUBSCRIBE` ⟹ `virtio_scsi_ctrl_an_req` / `_resp`.
    - else ⟹ continue (drop).
  - `vhost_scsi_chk_size(vq, &vc)`.
  - Advance `vc.req` past type header.
  - `vhost_scsi_get_req(vq, &vc, &tpg)`.
  - If TMF ⟹ `vhost_scsi_handle_tmf(vs, tpg, vq, &v_req.tmf, &vc, vq_log, log_num)`.
  - Else ⟹ `vhost_scsi_send_an_resp(vs, vq, &vc)`.
- Errors same shape as REQ-9 with `TYPE_CTRL_TMF` / `TYPE_CTRL_AN`.

REQ-15: VhostScsi::handle_tmf(vs, tpg, vq, vtmf, vc, log, log_num):
- Reject unless `vtmf.subtype == VIRTIO_SCSI_T_TMF_LOGICAL_UNIT_RESET`.
- Reject if `tpg.tpg_nexus == NULL`.
- Alloc `tmf = kzalloc`.
- `INIT_WORK(tmf.flush_work, vhost_scsi_tmf_flush_work)`.
- `vhost_work_init(tmf.vwork, vhost_scsi_tmf_resp_work)`.
- `tmf.vhost = vs`; `tmf.svq = svq`; `tmf.resp_iov = vq.iov[vc.out]`; `tmf.vq_desc = vc.head`; `tmf.in_iovs = vc.in`.
- `tmf.inflight = vhost_scsi_get_inflight(vq)`.
- If logging: `tmf.tmf_log = kmalloc_array(log_num)` + `memcpy(log)`.
- `target_submit_tmr(&tmf.se_cmd, tpg.tpg_nexus.tvn_se_sess, NULL, vhost_buf_to_lun(vtmf.lun), NULL, TMR_LUN_RESET, GFP_KERNEL, 0, TARGET_SCF_ACK_KREF)`.
- On reject path: `vhost_scsi_send_tmf_resp(VIRTIO_SCSI_S_FUNCTION_REJECTED)`.

REQ-16: VhostScsi::send_evt(vs, vq, tpg, lun, event, reason) — hotplug AEN:
- `evt = vhost_scsi_allocate_evt(vs, event, reason)` — rejects if `vs_events_nr > VHOST_SCSI_MAX_EVENT = 128` (sets `vs_events_missed = true`).
- If `tpg && lun`: encode virtio-scsi LUN bytes: `evt.event.lun[0] = 0x01`; `[1] = tpg.tport_tpgt`; `[2] = lun >> 8 | 0x40` (if ≥256); `[3] = lun & 0xFF`.
- `llist_add(evt.list, vs.vs_event_list)`.
- `vhost_vq_work_queue(vq, vs.vs_event_work)`; if device gone ⟹ `vhost_scsi_complete_events(vs, drop=true)` immediate.

REQ-17: VhostScsi::set_features(vs, features):
- Reject `features & ~VHOST_SCSI_FEATURES` (`VHOST_FEATURES | VIRTIO_SCSI_F_HOTPLUG | VIRTIO_SCSI_F_T10_PI`).
- Lock `vs.dev.mutex`.
- If `VHOST_F_LOG_ALL` ∧ `!vhost_log_access_ok(vs.dev)`: -EFAULT.
- For each VQ: `vq.acked_features = features` under `vq.mutex`.
- If `was_log ∧ !is_log`: walk IO VQs and `vhost_scsi_destroy_vq_log(vq)`.

REQ-18: VhostScsi::ioctl(ioctl, argp):
- `VHOST_SCSI_SET_ENDPOINT` ⟹ copy_from_user `vhost_scsi_target`; validate `reserved == 0`; call `set_endpoint`.
- `VHOST_SCSI_CLEAR_ENDPOINT` ⟹ same shape; call `clear_endpoint`.
- `VHOST_SCSI_GET_ABI_VERSION` ⟹ copy_to_user `VHOST_SCSI_ABI_VERSION`.
- `VHOST_SCSI_SET_EVENTS_MISSED` / `_GET_EVENTS_MISSED` ⟹ get/put under EVT VQ mutex.
- `VHOST_GET_FEATURES` ⟹ `VHOST_SCSI_FEATURES`.
- `VHOST_SET_FEATURES` ⟹ `set_features`.
- `VHOST_{NEW,FREE,ATTACH_VRING_,GET_VRING_}WORKER` ⟹ `vhost_worker_ioctl(vs.dev, ...)` under dev mutex.
- default ⟹ `vhost_dev_ioctl` then `vhost_vring_ioctl` if -ENOIOCTLCMD.

REQ-19: LIO configfs binding (`vhost_scsi_ops`):
- Registered via `target_register_template(&vhost_scsi_ops)` in `vhost_scsi_init`.
- `.fabric_name = "vhost"` ⟹ configfs path `/sys/kernel/config/target/vhost/`.
- `.max_data_sg_nents = VHOST_SCSI_PREALLOC_SGLS`.
- `.tpg_get_wwn = vhost_scsi_get_fabric_wwn`.
- `.tpg_get_tag = vhost_scsi_get_tpgt`.
- `.tpg_check_demo_mode = vhost_scsi_check_true` (always demo-mode — auto-create node ACL).
- `.tpg_check_demo_mode_cache = vhost_scsi_check_true`.
- `.tpg_check_prot_fabric_only = vhost_scsi_check_prot_fabric_only`.
- `.release_cmd = vhost_scsi_release_cmd`.
- `.check_stop_free = vhost_scsi_check_stop_free`.
- `.write_pending = vhost_scsi_write_pending`.
- `.queue_data_in = vhost_scsi_queue_data_in`.
- `.queue_status = vhost_scsi_queue_status`.
- `.queue_tm_rsp = vhost_scsi_queue_tm_rsp`.
- `.aborted_task = vhost_scsi_aborted_task`.
- `.fabric_make_wwn = vhost_scsi_make_tport`; `.fabric_drop_wwn = vhost_scsi_drop_tport`.
- `.fabric_make_tpg = vhost_scsi_make_tpg`; `.fabric_drop_tpg = vhost_scsi_drop_tpg`.
- `.fabric_post_link = vhost_scsi_port_link`; `.fabric_pre_unlink = vhost_scsi_port_unlink`.
- `.default_compl_type = TARGET_QUEUE_COMPL`; `.direct_compl_supp = 1`.
- `.default_submit_type = TARGET_QUEUE_SUBMIT`; `.direct_submit_supp = 1`.

REQ-20: Make-nexus (`vhost_scsi_make_nexus`):
- Lock `tpg.tv_tpg_mutex`.
- Reject if `tpg.tpg_nexus` already set.
- Alloc `tv_nexus`.
- `tv_nexus.tvn_se_sess = target_setup_session(&tpg.se_tpg, 0, 0, TARGET_PROT_DIN_PASS | TARGET_PROT_DOUT_PASS, name, tv_nexus, NULL)`.
- Install `tpg.tpg_nexus = tv_nexus`.

REQ-21: Persistent reservations:
- PERSISTENT RESERVE OUT/IN CDBs (0x5e/0x5f) flow unchanged through `target_submit_prep` + `target_submit`.
- LIO's `target_core_pr.c` resolves the reservation against the I_T-nexus identifier (the demo-mode `vhost_scsi_nexus.tvn_se_sess.se_node_acl.initiatorname` derived from configfs WWN).
- Reservation conflicts return `SAM_STAT_RESERVATION_CONFLICT` through the standard status path (REQ-11).

REQ-22: Per-`VHOST_SCSI_PREALLOC_SGLS` + inline-SG tunable:
- Module param `inline_sg_cnt` (range checked by `vhost_scsi_set_inline_sg_cnt`).
- Per-cmd inline SG entries stored inside `vhost_scsi_cmd`; spill via `sg_alloc_table_chained`.

REQ-23: Per-`vhost_scsi_flush`:
- `vhost_scsi_init_inflight(vs, vs.old_inflight)` — bump generation.
- For each VQ: `kref_put(old_inflight[i].kref)` — initial 1 ref dropped.
- `vhost_dev_flush(vs.dev)`.
- For each VQ: `wait_for_completion(old_inflight[i].comp)`.

REQ-24: Per-hotplug callback chain:
- `mkdir /sys/kernel/config/target/vhost/<wwpn>/tpgt_N/lun/lun_0/backstore_link` ⟹ LIO calls `vhost_scsi_port_link` ⟹ `tpg.tv_tpg_port_count++` + `vhost_scsi_hotplug(tpg, lun)` ⟹ `vhost_scsi_do_plug(tpg, lun, plug=true)` ⟹ `vhost_scsi_send_evt(vs, evt_vq, tpg, lun, VIRTIO_SCSI_T_TRANSPORT_RESET, VIRTIO_SCSI_EVT_RESET_RESCAN)`.
- Reverse for `port_unlink` ⟹ `VIRTIO_SCSI_EVT_RESET_REMOVED`.

REQ-25: Per-`/dev/vhost-scsi` chardev:
- `static struct miscdevice vhost_scsi_misc = { MISC_DYNAMIC_MINOR, "vhost-scsi", &vhost_scsi_fops }`.
- Registered via `misc_register` in `vhost_scsi_register` (called from `vhost_scsi_init`).
- fops: `.release = vhost_scsi_release`, `.unlocked_ioctl = vhost_scsi_ioctl`, `.compat_ioctl = compat_ptr_ioctl`, `.open = vhost_scsi_open`, `.llseek = noop_llseek`.

## Acceptance Criteria

- [ ] AC-1: `/dev/vhost-scsi` opens; `VHOST_SCSI_GET_ABI_VERSION` returns `VHOST_SCSI_ABI_VERSION`.
- [ ] AC-2: `VHOST_SCSI_SET_ENDPOINT` with WWN matching a configfs TPG installs `vs_tpg`; -ENODEV if no match; -EEXIST if endpoint already set.
- [ ] AC-3: virtio-scsi guest sends READ-10 / WRITE-10 ⟹ LIO backstore I/O completes ⟹ guest sees `SAM_STAT_GOOD` + correct residual.
- [ ] AC-4: T10-PI (`VIRTIO_SCSI_F_T10_PI`): negotiated ⟹ `pi_bytes*` parsed into `prot_sgl` ⟹ LIO passes prot through to NVMe-backed backstore.
- [ ] AC-5: LUN RESET TMF: guest sends `VIRTIO_SCSI_T_TMF_LOGICAL_UNIT_RESET` ⟹ `target_submit_tmr(TMR_LUN_RESET)` ⟹ response code `VIRTIO_SCSI_S_FUNCTION_SUCCEEDED` if LIO completes.
- [ ] AC-6: PERSISTENT RESERVE OUT/IN: reservation held by guest A; guest B sees `SAM_STAT_RESERVATION_CONFLICT` on conflicting access.
- [ ] AC-7: Hotplug: `mkdir lun_N`/`ln -s backstore` while endpoint active ⟹ guest receives `VIRTIO_SCSI_T_TRANSPORT_RESET / RESET_RESCAN`.
- [ ] AC-8: Hotunplug: `rm lun_N` ⟹ guest receives `RESET_REMOVED`; pending cmds complete cleanly.
- [ ] AC-9: BAD_TARGET: virtio-scsi req with unmapped target byte ⟹ `VIRTIO_SCSI_S_BAD_TARGET` response.
- [ ] AC-10: CDB > 32 bytes ⟹ -EIO ⟹ `VIRTIO_SCSI_S_BAD_TARGET` response.
- [ ] AC-11: `VHOST_SCSI_CLEAR_ENDPOINT` after I/O quiesces all VQs, flushes inflight via `vhost_scsi_flush`, frees cmds.
- [ ] AC-12: `VHOST_SCSI_MAX_EVENT = 128` cap: excess events ⟹ `vs_events_missed = true`; `EVENTS_MISSED` ioctl reflects.
- [ ] AC-13: `inline_sg_cnt` module param settable in (`VHOST_SCSI_PREALLOC_SGLS_RANGE`); affects per-cmd inline SG count.
- [ ] AC-14: `VHOST_F_LOG_ALL`: dirty writes recorded via `vhost_scsi_log_write` covering each completed cmd.

## Architecture

```
struct VhostScsi {
  dev: VhostDev,
  vqs: Vec<VhostScsiVirtqueue>,           // [CTL, EVT, IO_0..IO_N]
  vs_tpg: Option<[Option<*VhostScsiTpg>; VHOST_SCSI_MAX_TARGET]>,
  vs_vhost_wwpn: [u8; TRANSPORT_IQN_LEN],
  old_inflight: Vec<*VhostScsiInflight>,
  vs_event_work: VhostWork,
  vs_event_list: LlistHead,
  vs_events_missed: bool,
  vs_events_nr: u32,
  inline_sg_cnt: u32,
}

struct VhostScsiVirtqueue {
  vq: VhostVirtqueue,
  vs: *VhostScsi,
  inflights: [VhostScsiInflight; 2],
  inflight_idx: u8,
  scsi_cmds: Vec<VhostScsiCmd>,
  scsi_tags: Sbitmap,
  max_cmds: u32,
  upages: Vec<*Page>,
  completion_work: VhostWork,
  completion_list: LlistHead,
}

struct VhostScsiCmd {
  tvc_vq_desc: i32,
  tvc_sgl_count: u32, tvc_prot_sgl_count: u32,
  copied_iov: bool,
  read_iov: Option<*const Iovec>,
  read_iter: Option<*IovIter>,
  sgl: *Scatterlist, table: SgTable,
  prot_sgl: *Scatterlist, prot_table: SgTable,
  tvc_resp_iov: Iovec, tvc_resp_iovs: *Iovec, tvc_resp_iovs_cnt: u32,
  tvc_vq: *VhostVirtqueue,
  tvc_se_cmd: SeCmd,                     // LIO embedded
  tvc_sense_buf: [u8; TRANSPORT_SENSE_BUFFER],
  tvc_log: Option<*VhostLog>, tvc_log_num: u32,
  tvc_completion_list: LlistNode,
  inflight: *VhostScsiInflight,
}

struct VhostScsiTpg {
  tport_tpgt: u16,
  tv_tpg_port_count: i32,
  tv_tpg_vhost_count: i32,
  tv_fabric_prot_type: i32,
  tv_tpg_list: ListHead,
  tv_tpg_mutex: Mutex<()>,
  tpg_nexus: Option<*VhostScsiNexus>,
  tport: *VhostScsiTport,
  se_tpg: SePortalGroup,
  vhost_scsi: Option<*VhostScsi>,
}

struct VhostScsiTport {
  tport_proto_id: u8,                    // SAS / FCP / ISCSI
  tport_wwpn: u64,
  tport_name: [u8; VHOST_SCSI_NAMELEN],
  tport_wwn: SeWwn,
}

struct VhostScsiNexus {
  tvn_se_sess: *SeSession,                // LIO session
}

struct VhostScsiInflight {
  comp: Completion,
  kref: Kref,
}

struct VhostScsiCtx {                     // per-iteration scratch
  head: i32,
  out: u32, in_: u32,
  req_size: usize, rsp_size: usize,
  out_size: usize, in_size: usize,
  target: *const u8, lunp: *const u8,
  req: *void,
  out_iter: IovIter,
}
```

`VhostScsi::handle_vq(vs, vq)` — IO command pipeline:
1. lock `vq.mutex`.
2. `vs_tpg = vhost_vq_get_backend(vq)`; if NULL ⟹ unlock + return.
3. `memset(&vc, 0)`; `vc.rsp_size = sizeof virtio_scsi_cmd_resp`.
4. `vhost_disable_notify(&vs.dev, vq)`.
5. `vq_log = vhost_has_feature(VHOST_F_LOG_ALL) ? vq.log : NULL`.
6. loop while `!vhost_exceeds_weight(vq, ++c, 0)`:
   - `ret = vhost_scsi_get_desc(vs, vq, &vc, vq_log, &log_num)` — fills `vc.head, out, in, out_size, in_size, out_iter`.
   - if `VIRTIO_SCSI_F_T10_PI` acked: `vc.req = &v_req_pi; vc.req_size = sizeof v_req_pi; vc.lunp = &v_req_pi.lun[0]; vc.target = &v_req_pi.lun[1]`.
   - else: `vc.req = &v_req; vc.req_size = sizeof v_req; vc.lunp = &v_req.lun[0]; vc.target = &v_req.lun[1]`.
   - `vhost_scsi_chk_size(vq, &vc)` — request/response size validation.
   - `vhost_scsi_get_req(vq, &vc, &tpg)` — copy req header from `vc.out_iter`; require `*vc.lunp == 1`; lookup `tpg = vs_tpg[*vc.target]`.
   - compute `prot_bytes = 0; data_direction`:
     - `vc.out_size > vc.req_size` ⟹ `DMA_TO_DEVICE`, `exp_data_len = vc.out_size - vc.req_size`, `data_iter = vc.out_iter`.
     - else `vc.in_size > vc.rsp_size` ⟹ `DMA_FROM_DEVICE`, `exp_data_len = vc.in_size - vc.rsp_size`, init `in_iter = ITER_DEST(vq.iov[vc.out], vc.in)`, advance by `rsp_size`, `data_iter = in_iter`.
     - else ⟹ `DMA_NONE, exp_data_len = 0`.
   - if T10-PI:
     - if `v_req_pi.pi_bytesout && DMA_TO_DEVICE`: `prot_bytes = vhost32_to_cpu(pi_bytesout)`.
     - else if `v_req_pi.pi_bytesin && DMA_FROM_DEVICE`: `prot_bytes = vhost32_to_cpu(pi_bytesin)`.
     - else if either non-zero with wrong direction ⟹ -EIO.
     - if `prot_bytes`: `exp_data_len -= prot_bytes`; `prot_iter = data_iter`; truncate to `prot_bytes`; advance `data_iter` past `prot_bytes`.
     - `tag = vhost64_to_cpu(v_req_pi.tag)`; `task_attr = v_req_pi.task_attr`; `cdb = v_req_pi.cdb`; `lun = vhost_buf_to_lun(v_req_pi.lun)`.
   - else: `tag / task_attr / cdb / lun` from `v_req`.
   - validate `scsi_command_size(cdb) ≤ VHOST_SCSI_MAX_CDB_SIZE` (-EIO else).
   - `nexus = tpg.tpg_nexus`; NULL ⟹ -EIO.
   - `cmd = vhost_scsi_get_cmd(vq, tag)` — `tag = sbitmap_get(scsi_tags)`; on -ENOMEM caller will respond TASK_SET_FULL.
   - `cmd.tvc_vq = vq`.
   - `vhost_scsi_setup_resp_iovs(cmd, &vq.iov[vc.out], vc.in)` — save response iovecs.
   - if `vq_log && log_num`: `vhost_scsi_copy_cmd_log(vq, cmd, vq_log, log_num)`.
   - if `data_direction != DMA_NONE`: `vhost_scsi_mapal(vs, cmd, prot_bytes, &prot_iter, exp_data_len, &data_iter, data_direction)` — GUP-pin guest pages into `cmd.table.sgl` (and `prot_table.sgl`), fall back to alloc-page+copy for misaligned iovecs (sets `copied_iov`).
   - `cmd.tvc_vq_desc = vc.head`.
   - `vhost_scsi_target_queue_cmd(nexus, cmd, cdb, lun, task_attr, data_direction, exp_data_len + prot_bytes)`.
   - on `ret = -EIO`: `vhost_scsi_send_bad_target(vs, vq, &vc, TYPE_IO_CMD)` + log_write.
   - on `ret = -ENOMEM`: `vhost_scsi_send_status(vs, vq, &vc, SAM_STAT_TASK_SET_FULL)` + log_write.
   - on `ret = -ENXIO`: break.
7. unlock `vq.mutex`.

`VhostScsi::target_queue_cmd(nexus, cmd, cdb, lun, task_attr, data_dir, exp_data_len)`:
1. `se_cmd = &cmd.tvc_se_cmd`.
2. if `cmd.tvc_sgl_count > 0`:
   - `sg_ptr = cmd.table.sgl`.
   - if `cmd.tvc_prot_sgl_count > 0`: `sg_prot_ptr = cmd.prot_table.sgl`; else `se_cmd.prot_pto = true` (passthrough).
3. else: `sg_ptr = NULL`.
4. `se_cmd.tag = 0`.
5. `target_init_cmd(se_cmd, nexus.tvn_se_sess, cmd.tvc_sense_buf, lun, exp_data_len, vhost_scsi_to_tcm_attr(task_attr), data_dir, TARGET_SCF_ACK_KREF)`.
6. if `target_submit_prep(se_cmd, cdb, sg_ptr, tvc_sgl_count, NULL, 0, sg_prot_ptr, tvc_prot_sgl_count, GFP_KERNEL) != 0`: return (LIO has consumed cmd via release_cmd already).
7. `target_submit(se_cmd)`.

`VhostScsi::complete_cmd_work(work)`:
1. `llnode = llist_del_all(&svq.completion_list)`.
2. lock `svq.vq.mutex`.
3. for each `cmd` in `llnode`:
   - `se_cmd = &cmd.tvc_se_cmd`.
   - zero `v_rsp: virtio_scsi_cmd_resp`.
   - if `cmd.read_iter != NULL`:
     - if `vhost_scsi_copy_sgl_to_iov(cmd) != 0` ⟹ `v_rsp.response = VIRTIO_SCSI_S_BAD_TARGET`.
   - else:
     - `v_rsp.resid = cpu_to_vhost32(se_cmd.residual_count)`.
     - `v_rsp.status = se_cmd.scsi_status`.
     - `v_rsp.sense_len = cpu_to_vhost32(se_cmd.scsi_sense_length)`.
     - `memcpy(v_rsp.sense, cmd.tvc_sense_buf, scsi_sense_length)`.
   - `iov_iter_init(ITER_DEST, cmd.tvc_resp_iovs, cmd.tvc_resp_iovs_cnt, sizeof v_rsp)`.
   - `copy_to_iter(&v_rsp, sizeof v_rsp)`.
   - `vhost_add_used(cmd.tvc_vq, cmd.tvc_vq_desc, 0)` + set `signal = true`.
   - `vhost_scsi_log_write(cmd.tvc_vq, cmd.tvc_log, cmd.tvc_log_num)`.
   - `vhost_scsi_release_cmd_res(se_cmd)` — frees pages, sg-tables, clears tag, drops inflight.
4. unlock; if `signal`: `vhost_signal(&svq.vs.dev, &svq.vq)`.

`VhostScsi::ctl_handle_vq(vs, vq)` — TMF + AN pipeline:
1. lock `vq.mutex`.
2. backend NULL ⟹ unlock + return.
3. `vhost_disable_notify`.
4. loop:
   - `vhost_scsi_get_desc`.
   - peek `v_req.type` (`__virtio32`) via `copy_from_iter_full(&vc.out_iter, &v_req.type, sizeof type)` — if fault, continue (cannot send response w/o known type).
   - switch `vhost32_to_cpu(v_req.type)`:
     - `VIRTIO_SCSI_T_TMF` ⟹ `vc.req = &v_req.tmf; vc.req_size = sizeof virtio_scsi_ctrl_tmf_req; vc.rsp_size = sizeof virtio_scsi_ctrl_tmf_resp; vc.lunp = &v_req.tmf.lun[0]; vc.target = &v_req.tmf.lun[1]`.
     - `VIRTIO_SCSI_T_AN_QUERY` / `_AN_SUBSCRIBE` ⟹ `vc.req = &v_req.an; vc.req_size = sizeof virtio_scsi_ctrl_an_req; vc.rsp_size = sizeof virtio_scsi_ctrl_an_resp; vc.lunp = &v_req.an.lun[0]; vc.target = NULL`.
     - default ⟹ continue (drop).
   - `vhost_scsi_chk_size`; advance `vc.req` past `type`; decrement `vc.req_size` by `sizeof type`.
   - `vhost_scsi_get_req(vq, &vc, &tpg)`.
   - if TMF ⟹ `vhost_scsi_handle_tmf(vs, tpg, vq, &v_req.tmf, &vc, vq_log, log_num)`.
   - else ⟹ `vhost_scsi_send_an_resp(vs, vq, &vc)` + `vhost_scsi_log_write`.
   - on errors mirror REQ-9 but with `TYPE_CTRL_TMF` / `TYPE_CTRL_AN` for `send_bad_target`.
5. unlock.

`VhostScsi::handle_tmf` — see REQ-15.

`VhostScsi::send_evt` — see REQ-16.

`VhostScsi::set_endpoint` / `clear_endpoint` — see REQ-7, REQ-8.

`VhostScsi::ioctl` — see REQ-18.

`VhostScsi::flush` — see REQ-23.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `endpoint_set_unique` | INVARIANT | per-SET_ENDPOINT: rejects if `vs_tpg != NULL` (-EEXIST). |
| `target_index_bounded` | INVARIANT | per-handle_vq: `*vc.target < VHOST_SCSI_MAX_TARGET`. |
| `lun_byte0_one` | INVARIANT | per-get_req: `*vc.lunp == 1` enforced (virtio-scsi spec). |
| `cdb_size_bounded` | INVARIANT | per-handle_vq: `scsi_command_size(cdb) ≤ VHOST_SCSI_MAX_CDB_SIZE = 32`. |
| `tag_sbitmap_unique` | INVARIANT | per-get_cmd: `sbitmap_get` returns unique tag; clear on release_cmd_res. |
| `prot_dir_match` | INVARIANT | per-T10-PI: `pi_bytesout` ⟺ DMA_TO_DEVICE; `pi_bytesin` ⟺ DMA_FROM_DEVICE; mismatch ⟹ -EIO. |
| `inflight_kref_balanced` | INVARIANT | per-cmd: `get_inflight` and `put_inflight` paired across submit→release. |
| `tpg_vhost_count_balanced` | INVARIANT | per-set_endpoint / clear_endpoint: `tv_tpg_vhost_count` increment and decrement paired. |
| `target_depend_item_paired` | INVARIANT | per-set/clear: each `target_depend_item` matched by `target_undepend_item`. |
| `evt_cap` | INVARIANT | per-allocate_evt: `vs_events_nr ≤ VHOST_SCSI_MAX_EVENT = 128`; overflow ⟹ `vs_events_missed = true`. |
| `nexus_under_tv_tpg_mutex` | INVARIANT | per-make/drop_nexus: `tv_tpg_mutex` held while reading/writing `tpg_nexus`. |
| `cmd_log_only_when_log_feature` | INVARIANT | per-handle_vq: `vq_log != NULL ⟺ VHOST_F_LOG_ALL acked`. |
| `release_cmd_res_pages_freed` | INVARIANT | per-release: every page in `cmd.table.sgl`/`cmd.prot_table.sgl` freed (`__free_page` or `put_page`). |

### Layer 2: TLA+

`drivers/vhost/scsi.tla`:
- Per-`SET_ENDPOINT` ⟶ per-handle_vq ⟶ per-target_queue_cmd ⟶ per-LIO_complete ⟶ per-release_cmd ⟶ per-complete_cmd_work ⟶ per-add_used+signal.
- Per-`CLEAR_ENDPOINT` ⟶ per-vhost_vq_set_backend(NULL) ⟶ per-flush(wait_inflight=0) ⟶ per-destroy_vq_cmds ⟶ per-undepend.
- Per-TMF: ctl_handle_vq ⟶ handle_tmf ⟶ target_submit_tmr ⟶ queue_tm_rsp ⟶ release_cmd(schedule_work) ⟶ tmf_flush_work ⟶ tmf_resp_work ⟶ send_tmf_resp.
- Per-AEN: port_link/unlink ⟶ send_evt ⟶ evt_handle_kick ⟶ do_evt_work ⟶ guest receives `VIRTIO_SCSI_T_TRANSPORT_RESET`.
- Properties:
  - `safety_no_two_endpoints` — `SET_ENDPOINT` while bound ⟹ -EEXIST.
  - `safety_clear_drains_inflight` — `CLEAR_ENDPOINT` does not free `scsi_cmds` until all inflight kref → 0.
  - `safety_tag_unique_per_vq` — every inflight `vhost_scsi_cmd` has unique `map_tag` within VQ.
  - `safety_evt_cap` — pending events never exceed `VHOST_SCSI_MAX_EVENT`.
  - `safety_tpg_active_implies_vhost_count_ge_1` — an endpoint-bound TPG has `tv_tpg_vhost_count ≥ 1` until clear.
  - `liveness_request_completes` — every received virtio-scsi req either completes via `add_used` or is rejected via `send_bad_target`/`send_status`.
  - `liveness_flush_terminates` — `vhost_scsi_flush` returns after all in-flight kref drop.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VhostScsi::set_endpoint` post: ret == 0 ⟹ `vs_tpg != NULL` ∧ every matched TPG has `tv_tpg_vhost_count++` + `target_depend_item` held | `VhostScsi::set_endpoint` |
| `VhostScsi::clear_endpoint` post: ret == 0 ⟹ `vs_tpg == NULL` ∧ every matched TPG had `tv_tpg_vhost_count--` + `target_undepend_item` | `VhostScsi::clear_endpoint` |
| `VhostScsi::handle_vq` post: each successful iter ⟹ `target_submit` called exactly once with matching `prot_pto` semantics | `VhostScsi::handle_vq` |
| `VhostScsi::target_queue_cmd` post: `target_submit` issued ⟺ `target_submit_prep` returned 0 | `VhostScsi::target_queue_cmd` |
| `VhostScsi::complete_cmd_work` post: every drained cmd ⟹ `vhost_add_used` once + `release_cmd_res` once | `VhostScsi::complete_cmd_work` |
| `VhostScsi::release_cmd` post: TMR ⟹ `schedule_work(tmf.flush_work)`; non-TMR ⟹ `llist_add(svq.completion_list)` + `vhost_vq_work_queue` (or `drop_cmds` fallback) | `VhostScsi::release_cmd` |
| `VhostScsi::handle_tmf` post: only `TMF_LOGICAL_UNIT_RESET` admitted; others ⟹ `VIRTIO_SCSI_S_FUNCTION_REJECTED` | `VhostScsi::handle_tmf` |
| `VhostScsi::send_evt` post: `vs_events_nr ≤ VHOST_SCSI_MAX_EVENT` | `VhostScsi::send_evt` |
| `VhostScsi::flush` post: `wait_for_completion(old_inflight[i].comp)` per VQ | `VhostScsi::flush` |
| `VhostScsi::make_nexus` post: `tpg.tpg_nexus != NULL` ⟹ `target_setup_session` returned non-ERR_PTR ∧ holds `tv_tpg_mutex` during install | `VhostScsi::make_nexus` |
| `VhostScsi::drop_nexus` post: rejects if `tv_tpg_port_count != 0` or `tv_tpg_vhost_count != 0` (-EBUSY) | `VhostScsi::drop_nexus` |
| `VhostScsi::release_cmd_res` post: `sbitmap_clear_bit(map_tag)` + `put_inflight` | `VhostScsi::release_cmd_res` |

### Layer 4: Verus/Creusot functional

`Per-virtio-scsi I/O request → vhost_scsi_handle_vq parse → mapal(GUP pages) → target_queue_cmd → LIO backstore → release_cmd → complete_cmd_work → virtio_scsi_cmd_resp written ∧ vhost_add_used_and_signal` semantic equivalence: per-Documentation/virt/vhost/vhost.rst + virtio-scsi spec (PCI 2.0 + 1.x) + Documentation/target/tcm_mod_builder.rst.

`Per-LIO target_core_fabric_ops binding` semantic equivalence: per-target_core_fabric.h vtable; vhost-scsi is registered under fabric name `"vhost"`; configfs entry `/sys/kernel/config/target/vhost/<wwpn>/tpgt_N/` mirrors mainline byte-identically.

## Hardening

(Inherits row-1 features from `drivers/vhost/00-overview.md` § Hardening.)

vhost-scsi reinforcement:

- **Per-`CAP_SYS_ADMIN` on `/dev/vhost-scsi`** — defense against per-unprivileged-virtio-backend-spoof (inherited from miscdev access mode + vhost framework owner check).
- **Per-LUN byte[0] == 1 enforced** — defense against per-malformed-virtio-scsi-frame (virtio-scsi spec mandate).
- **Per-CDB size ≤ 32 (`VHOST_SCSI_MAX_CDB_SIZE`)** — defense against per-oversized-CDB heap-overflow into `tvc_cdb`.
- **Per-target byte indexed into bounded `vs_tpg[VHOST_SCSI_MAX_TARGET=256]`** — defense against per-OOB-target-index.
- **Per-T10-PI direction-match validated** — defense against per-prot/data direction mismatch ⟹ -EIO.
- **Per-sbitmap tag pool bounded by `vq.num`** — defense against per-tag-pool-exhaustion (responds TASK_SET_FULL, never overflows).
- **Per-`tv_tpg_vhost_count` ≠ 0 gates configfs `drop_nexus`** — defense against per-rmdir-while-endpoint-active UAF.
- **Per-`target_depend_item` on `tpg_group.cg_item`** — defense against per-configfs-removal during live endpoint.
- **Per-`vhost_scsi_flush` waits inflight kref ⟹ 0 before backend free** — defense against per-cmd-after-clear UAF.
- **Per-AEN cap `VHOST_SCSI_MAX_EVENT = 128`** — defense against per-event-flood OOM.
- **Per-`vs_events_missed` flag set on overflow** — defense against per-silent-event-drop (guest informed via `VIRTIO_SCSI_T_EVENTS_MISSED` flag on next event).
- **Per-`scsi_command_size` validated against CDB header** — defense against per-truncated-CDB read.
- **Per-`get_user_pages` returns checked + `put_page`/`__free_page` on release** — defense against per-pinned-page leak on guest-buggy iovec.
- **Per-`response.response = VIRTIO_SCSI_S_BAD_TARGET` for unmapped target / oversized CDB** — defense against per-silent-cmd-drop hanging guest IO.
- **Per-VQ `acked_features` validated against `VHOST_SCSI_FEATURES` mask** — defense against per-bogus-feature-bit (e.g. enabling T10-PI without backend support).
- **Per-`SCF_SCSI_TMR_CDB` routed through dedicated `vhost_scsi_tmf` lifecycle** — defense against per-TMR/IO state confusion.

## Open Questions

- Per-multi-queue tag-pool sizing: `vq.num` per IO-VQ versus shared sbitmap — Rookery should preserve per-VQ partition unless we can demonstrate equivalent fairness with a shared pool. Defer to Phase D.
- Per-`direct_compl_supp = 1` / `direct_submit_supp = 1`: the LIO direct-completion fast-path bypasses the `completion_list` for some backstores — verify our Rust binding exercises both paths.

## Out of Scope

- `drivers/target/` (LIO core, persistent reservations, backstores) — covered under `drivers/target/00-overview.md` Tier-2.
- `drivers/vhost/vhost.c` (vhost framework, `vhost_dev`, `vhost_virtqueue`, worker kthread) — covered under `drivers/vhost/core.md` Tier-3.
- `drivers/vhost/iotlb.c` — covered under `drivers/vhost/iotlb.md` Tier-3.
- virtio-scsi *guest* driver (`drivers/scsi/virtio_scsi.c`) — covered under `drivers/scsi/virtio-scsi.md` Tier-3 if expanded.
- Implementation code.
