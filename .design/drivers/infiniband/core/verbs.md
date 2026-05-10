# Tier-3: drivers/infiniband/core/verbs.c — RDMA verbs API

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/infiniband/00-overview.md
upstream-paths:
  - drivers/infiniband/core/verbs.c (~3203 lines)
  - include/rdma/ib_verbs.h (struct ib_pd, ib_cq, ib_qp, ib_mr, ib_srq, ib_wc, ib_send_wr, ib_recv_wr, enums ib_qp_state / ib_wc_status / ib_wr_opcode)
  - include/uapi/rdma/ib_user_verbs.h (uverbs ABI struct + opcode constants)
-->

## Summary

The RDMA verbs layer is the kernel-side API by which both in-kernel consumers (ipoib, rdma_cm, srp, iser, nvmet-rdma, smb-direct, rds_rdma) and the uverbs ioctl/write fast-path translate verb-level operations into per-driver ops calls. Per-Protection-Domain (`struct ib_pd`) groups MR / QP / SRQ / AH allocations under a single trust boundary. Per-Completion-Queue (`struct ib_cq`) collects work completions (`struct ib_wc`) for poll/event delivery. Per-Queue-Pair (`struct ib_qp`) holds a send queue and recv queue, drives the RC/UC/UD/XRC/RAW state machine (RESET → INIT → RTR → RTS → SQD → SQE → ERR), and is modified via `ib_modify_qp` whose legality is validated by the upstream `qp_state_table[cur][next].{valid,req_param,opt_param}` matrix indexed by `enum ib_qp_type`. Per-Memory-Region (`struct ib_mr`) registers a virtual range with the HCA for RDMA READ/WRITE/atomic access. Per-Shared-Receive-Queue (`struct ib_srq`) pools receives across many UD/XRC QPs. Posting is via the inline shims `ib_post_send(qp, wr, &bad_wr)` and `ib_post_recv(qp, wr, &bad_wr)` that funnel through `device->ops.post_send / post_recv`; completion polling via `ib_poll_cq(cq, num, wc) → ops.poll_cq` and arming via `ib_req_notify_cq → ops.req_notify_cq`. Critical for: kernel ULP throughput, uverbs userspace RDMA, deterministic QP state transitions, IBA-compliant completion semantics.

This Tier-3 covers `drivers/infiniband/core/verbs.c` (~3203 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ib_pd` | per-protection-domain | `IbPd` |
| `struct ib_cq` | per-completion-queue | `IbCq` |
| `struct ib_qp` | per-queue-pair | `IbQp` |
| `struct ib_mr` | per-memory-region | `IbMr` |
| `struct ib_srq` | per-shared-recv-queue | `IbSrq` |
| `struct ib_ah` | per-address-handle | `IbAh` |
| `struct ib_xrcd` | per-XRC-domain | `IbXrcd` |
| `struct ib_wq` | per-RAW-packet work-queue | `IbWq` |
| `struct ib_wc` | per-work-completion | `IbWc` |
| `struct ib_send_wr` | per-send work-request | `IbSendWr` |
| `struct ib_recv_wr` | per-recv work-request | `IbRecvWr` |
| `struct ib_sge` | per-scatter-gather-element | `IbSge` |
| `struct ib_cqe` | per-CQ-entry callback | `IbCqe` |
| `struct ib_qp_init_attr` | per-QP-create attrs | `IbQpInitAttr` |
| `struct ib_qp_attr` | per-QP modify/query attrs | `IbQpAttr` |
| `struct ib_cq_init_attr` | per-CQ-create attrs | `IbCqInitAttr` |
| `struct ib_srq_init_attr` | per-SRQ-create attrs | `IbSrqInitAttr` |
| `struct rdma_ah_attr` | per-AH attrs (IB/RoCE) | `RdmaAhAttr` |
| `__ib_alloc_pd()` (macro `ib_alloc_pd`) | per-PD allocate | `IbPd::alloc` |
| `ib_dealloc_pd_user()` (macro `ib_dealloc_pd`) | per-PD free | `IbPd::dealloc` |
| `__ib_create_cq()` (macro `ib_create_cq`) | per-CQ create | `IbCq::create` |
| `ib_destroy_cq_user()` (macro `ib_destroy_cq`) | per-CQ destroy | `IbCq::destroy` |
| `rdma_set_cq_moderation()` | per-CQ moderation | `IbCq::set_moderation` |
| `create_qp()` (static) | per-QP construct + driver-alloc | `IbQp::construct` |
| `ib_create_qp_user()` | per-QP user create | `IbQp::create_user` |
| `ib_create_qp_kernel()` (macro `ib_create_qp`) | per-QP kernel create | `IbQp::create_kernel` |
| `ib_open_qp()` / `__ib_open_qp()` | per-XRC open shared QP | `IbQp::open_xrc` |
| `ib_modify_qp()` / `ib_modify_qp_with_udata()` / `_ib_modify_qp()` | per-QP attr modify | `IbQp::modify` |
| `ib_modify_qp_is_ok()` (uses qp_state_table) | per-transition validity check | `IbQp::is_modify_ok` |
| `ib_query_qp()` | per-QP attr query | `IbQp::query` |
| `ib_close_qp()` / `__ib_destroy_shared_qp()` | per-XRC close | `IbQp::close_xrc` |
| `ib_destroy_qp_user()` (macro `ib_destroy_qp`) | per-QP destroy | `IbQp::destroy` |
| `ib_qp_usecnt_inc()` / `ib_qp_usecnt_dec()` | per-QP refcount on PD/CQ/SRQ | `IbQp::usecnt` |
| `ib_post_send()` (inline) | per-send-WR post | `IbQp::post_send` |
| `ib_post_recv()` (inline) | per-recv-WR post | `IbQp::post_recv` |
| `ib_post_srq_recv()` (inline) | per-SRQ recv-WR post | `IbSrq::post_recv` |
| `ib_poll_cq()` (inline) | per-CQ completion poll | `IbCq::poll` |
| `ib_req_notify_cq()` (inline) | per-CQ arm notification | `IbCq::req_notify` |
| `ib_create_srq_user()` (macro `ib_create_srq`) | per-SRQ create | `IbSrq::create` |
| `ib_modify_srq()` / `ib_query_srq()` / `ib_destroy_srq_user()` | per-SRQ ops | `IbSrq::ops` |
| `ib_reg_user_mr()` / `ib_dereg_mr_user()` / `ib_advise_mr()` | per-MR user-mode | `IbMr::user` |
| `ib_alloc_mr()` / `ib_alloc_mr_integrity()` | per-MR alloc | `IbMr::alloc` |
| `ib_map_mr_sg()` / `ib_map_mr_sg_pi()` / `ib_sg_to_pages()` | per-MR sg-map | `IbMr::map_sg` |
| `ib_check_mr_status()` | per-MR PI-status | `IbMr::check_status` |
| `rdma_create_ah()` / `rdma_create_user_ah()` / `rdma_modify_ah()` / `rdma_query_ah()` / `rdma_destroy_ah_user()` | per-AH ops | `IbAh::ops` |
| `ib_create_ah_from_wc()` | per-AH from received WC | `IbAh::from_wc` |
| `ib_init_ah_attr_from_wc()` | per-AH-attr from WC | `IbAh::attr_from_wc` |
| `ib_get_gids_from_rdma_hdr()` / `ib_get_rdma_header_version()` | per-GRH inspection | `IbAh::grh_inspect` |
| `rdma_copy_ah_attr()` / `rdma_replace_ah_attr()` / `rdma_move_ah_attr()` / `rdma_destroy_ah_attr()` | per-ah-attr lifecycle | `RdmaAhAttr::ops` |
| `rdma_fill_sgid_attr()` / `rdma_unfill_sgid_attr()` / `rdma_update_sgid_attr()` | per-AH sgid resolution | `RdmaAhAttr::sgid` |
| `ib_alloc_xrcd_user()` / `ib_dealloc_xrcd_user()` | per-XRC domain | `IbXrcd::ops` |
| `ib_create_wq()` / `ib_destroy_wq_user()` | per-RAW-packet WQ | `IbWq::ops` |
| `ib_attach_mcast()` / `ib_detach_mcast()` | per-multicast | `IbQp::mcast` |
| `ib_drain_sq()` / `ib_drain_rq()` / `ib_drain_qp()` | per-QP graceful drain | `IbQp::drain` |
| `__ib_drain_sq()` / `__ib_drain_rq()` / `__ib_drain_srq()` | per-drain inner | `IbQp::drain_inner` |
| `ib_event_msg()` / `ib_wc_status_msg()` | per-event/status string | `IbCq::status_msg` |
| `ib_rate_to_mult()` / `mult_to_ib_rate()` / `ib_rate_to_mbps()` | per-rate conversion | `IbAh::rate` |
| `ib_port_attr_to_speed_info()` | per-port speed conversion | `IbCq::port_speed` |
| `rdma_node_get_transport()` / `rdma_port_get_link_layer()` | per-port type | `IbCq::port_type` |
| `ib_get_eth_speed()` | per-port netdev-derived speed | `IbCq::eth_speed` |
| `ib_set_vf_link_state()` / `ib_get_vf_config()` / `ib_get_vf_stats()` / `ib_set_vf_guid()` / `ib_get_vf_guid()` | per-SR-IOV VF | `IbDevice::vf_ops` |
| `rdma_alloc_netdev()` / `rdma_init_netdev()` | per-IPoIB netdev | `IbDevice::rdma_netdev` |
| `rdma_alloc_hw_stats_struct()` / `rdma_free_hw_stats_struct()` | per-HW-stats | `IbDevice::hw_stats` |
| `qp_state_table[IB_QPS_ERR + 1][IB_QPS_ERR + 1]` (static) | per-state-transition validity matrix | `IbQp::STATE_TABLE` |

## Compatibility contract

REQ-1: enum ib_qp_state (exact upstream order):
- IB_QPS_RESET = 0.
- IB_QPS_INIT  = 1.
- IB_QPS_RTR   = 2 (Ready-To-Receive).
- IB_QPS_RTS   = 3 (Ready-To-Send).
- IB_QPS_SQD   = 4 (Send-Queue-Drained).
- IB_QPS_SQE   = 5 (Send-Queue-Error; only UD/SMI/GSI).
- IB_QPS_ERR   = 6 (Error).

REQ-2: enum ib_qp_type:
- IB_QPT_SMI / IB_QPT_GSI / IB_QPT_RC / IB_QPT_UC / IB_QPT_UD / IB_QPT_RAW_IPV6 / IB_QPT_RAW_ETHERTYPE / IB_QPT_RAW_PACKET / IB_QPT_XRC_INI / IB_QPT_XRC_TGT / IB_QPT_MAX / IB_QPT_DRIVER (vendor-defined start).

REQ-3: enum ib_qp_attr_mask (bitmask; subset; see ib_verbs.h):
- IB_QP_STATE / _CUR_STATE / _EN_SQD_ASYNC_NOTIFY / _ACCESS_FLAGS / _PKEY_INDEX / _PORT / _QKEY / _AV / _PATH_MTU / _TIMEOUT / _RETRY_CNT / _RNR_RETRY / _RQ_PSN / _MAX_QP_RD_ATOMIC / _ALT_PATH / _MIN_RNR_TIMER / _SQ_PSN / _MAX_DEST_RD_ATOMIC / _PATH_MIG_STATE / _CAP / _DEST_QPN / _RATE_LIMIT.

REQ-4: enum ib_wr_opcode (kernel verbs; mirrors IB_UVERBS_WR_* on UAPI):
- IB_WR_RDMA_WRITE / _RDMA_WRITE_WITH_IMM / _SEND / _SEND_WITH_IMM / _RDMA_READ / _ATOMIC_CMP_AND_SWP / _ATOMIC_FETCH_AND_ADD / _BIND_MW / _LSO / _SEND_WITH_INV / _RDMA_READ_WITH_INV / _LOCAL_INV / _MASKED_ATOMIC_CMP_AND_SWP / _MASKED_ATOMIC_FETCH_AND_ADD / _FLUSH / _ATOMIC_WRITE.
- Kernel-only (not UAPI): IB_WR_REG_MR = 0x20, IB_WR_REG_MR_INTEGRITY.

REQ-5: enum ib_send_flags:
- IB_SEND_FENCE = 1, IB_SEND_SIGNALED = (1<<1), IB_SEND_SOLICITED = (1<<2), IB_SEND_INLINE = (1<<3), IB_SEND_IP_CSUM = (1<<4); driver-reserved bits 26..31.

REQ-6: enum ib_wc_status (exact upstream order):
- IB_WC_SUCCESS (0), LOC_LEN_ERR, LOC_QP_OP_ERR, LOC_EEC_OP_ERR, LOC_PROT_ERR, WR_FLUSH_ERR, MW_BIND_ERR, BAD_RESP_ERR, LOC_ACCESS_ERR, REM_INV_REQ_ERR, REM_ACCESS_ERR, REM_OP_ERR, RETRY_EXC_ERR, RNR_RETRY_EXC_ERR, LOC_RDD_VIOL_ERR, REM_INV_RD_REQ_ERR, REM_ABORT_ERR, INV_EECN_ERR, INV_EEC_STATE_ERR, FATAL_ERR, RESP_TIMEOUT_ERR, GENERAL_ERR.

REQ-7: enum ib_wc_opcode:
- IB_WC_SEND / _RDMA_WRITE / _RDMA_READ / _COMP_SWAP / _FETCH_ADD / _BIND_MW / _LOCAL_INV / _LSO / _ATOMIC_WRITE / _REG_MR / _MASKED_COMP_SWAP / _MASKED_FETCH_ADD / _FLUSH.
- IB_WC_RECV = (1 << 7); IB_WC_RECV_RDMA_WITH_IMM — consumers test (opcode & IB_WC_RECV) for recv-vs-send.

REQ-8: struct ib_sge:
- addr: u64 (virtual or DMA address depending on lkey).
- length: u32 (bytes).
- lkey: u32 (Local Key from MR registration).

REQ-9: struct ib_send_wr:
- next: *struct ib_send_wr (NULL = end-of-list).
- wr_id: u64 OR wr_cqe: *ib_cqe (union; CQE-mode preferred for ib_poll_cq with done callbacks).
- sg_list: *ib_sge.
- num_sge: i32.
- opcode: enum ib_wr_opcode.
- send_flags: i32 (ib_send_flags).
- ex.imm_data: __be32 OR ex.invalidate_rkey: u32.
- Subtype-extended structs (cast via container_of, helpers rdma_wr/atomic_wr/ud_wr/reg_wr):
  - struct ib_rdma_wr: { wr, u64 remote_addr, u32 rkey }.
  - struct ib_atomic_wr: { wr, u64 remote_addr, u64 compare_add, u64 swap, u64 compare_add_mask, u64 swap_mask, u32 rkey }.
  - struct ib_ud_wr: { wr, *ib_ah ah, u32 remote_qpn, u32 remote_qkey, u16 pkey_index, u32 port_num, ... }.
  - struct ib_reg_wr: { wr, *ib_mr mr, u32 key, int access }.

REQ-10: struct ib_recv_wr:
- next: *struct ib_recv_wr.
- wr_id or wr_cqe (union).
- sg_list: *ib_sge.
- num_sge: i32.

REQ-11: struct ib_wc:
- wr_id: u64 OR wr_cqe: *ib_cqe (union).
- status: enum ib_wc_status.
- opcode: enum ib_wc_opcode (test & IB_WC_RECV for recv-side).
- vendor_err: u32.
- byte_len: u32.
- qp: *ib_qp.
- ex.imm_data: __be32 OR ex.invalidate_rkey: u32.
- src_qp: u32 (UD only).
- slid: u32.
- wc_flags: i32 (ib_wc_flags: GRH / WITH_IMM / WITH_INVALIDATE / IP_CSUM_OK / WITH_SMAC / WITH_VLAN / WITH_NETWORK_HDR_TYPE).
- pkey_index: u16.
- sl: u8 (Service Level).
- dlid_path_bits: u8.
- port_num: u32 (DR SMPs on switches).
- smac: [u8; ETH_ALEN] (RoCE).
- vlan_id: u16 (RoCE).
- network_hdr_type: u8 (RoCEv2: IPv4 / IPv6).

REQ-12: struct ib_qp_attr (full set):
- qp_state / cur_qp_state: enum ib_qp_state.
- path_mtu: enum ib_mtu.
- path_mig_state: enum ib_mig_state.
- qkey / rq_psn / sq_psn / dest_qp_num: u32 (PSNs truncated to 24 bits when rdma_ib_or_roce(port) — see _ib_modify_qp).
- qp_access_flags: int.
- cap: struct ib_qp_cap { max_send_wr, max_recv_wr, max_send_sge, max_recv_sge, max_inline_data, max_rdma_ctxs }.
- ah_attr / alt_ah_attr: struct rdma_ah_attr.
- pkey_index / alt_pkey_index: u16.
- en_sqd_async_notify / sq_draining: u8.
- max_rd_atomic / max_dest_rd_atomic: u8.
- min_rnr_timer / timeout / retry_cnt / rnr_retry / alt_timeout: u8.
- port_num / alt_port_num: u32.
- rate_limit: u32.
- xmit_slave: *net_device (RoCE LAG).

REQ-13: __ib_alloc_pd(device, flags, caller) -> *ib_pd | ERR_PTR:
- pd = rdma_zalloc_drv_obj(device, ib_pd) — driver-size aware zalloc (uses size_ib_pd from device ops).
- pd.device = device; pd.flags = flags.
- rdma_restrack_new(&pd.res, RDMA_RESTRACK_PD); rdma_restrack_set_name(&pd.res, caller).
- ret = device.ops.alloc_pd(pd, NULL); on err: rdma_restrack_put; kfree; return ERR_PTR.
- rdma_restrack_add(&pd.res).
- If device.attrs.kernel_cap_flags & IBK_LOCAL_DMA_LKEY: pd.local_dma_lkey = device.local_dma_lkey; else mr_access_flags |= IB_ACCESS_LOCAL_WRITE.
- If flags & IB_PD_UNSAFE_GLOBAL_RKEY: pr_warn; mr_access_flags |= IB_ACCESS_REMOTE_READ | IB_ACCESS_REMOTE_WRITE.
- If mr_access_flags != 0: mr = pd.device.ops.get_dma_mr(pd, mr_access_flags); on ERR: ib_dealloc_pd; return ERR_CAST. Wire mr fields (device, pd, type=IB_MR_TYPE_DMA, uobject=NULL, need_inval=false). pd.__internal_mr = mr. If !IBK_LOCAL_DMA_LKEY: pd.local_dma_lkey = mr.lkey. If IB_PD_UNSAFE_GLOBAL_RKEY: pd.unsafe_global_rkey = mr.rkey.
- return pd.

REQ-14: ib_dealloc_pd_user(pd, udata) -> int:
- If pd.__internal_mr: WARN_ON(pd.device.ops.dereg_mr(pd.__internal_mr, NULL)); pd.__internal_mr = NULL.
- ret = pd.device.ops.dealloc_pd(pd, udata); on err return ret.
- rdma_restrack_del(&pd.res); kfree(pd); return 0.

REQ-15: __ib_create_cq(device, comp_handler, event_handler, cq_context, cq_attr, caller) -> *ib_cq | ERR_PTR:
- cq = rdma_zalloc_drv_obj(device, ib_cq).
- WARN_ON_ONCE(!cq_attr.cqe); return EINVAL if so.
- cq.device = device; cq.comp_handler = comp_handler; cq.event_handler = event_handler; cq.cq_context = cq_context; atomic_set(&cq.usecnt, 0).
- rdma_restrack_new(&cq.res, RDMA_RESTRACK_CQ); rdma_restrack_set_name(&cq.res, caller).
- ret = device.ops.create_cq(cq, cq_attr, NULL); on err: rdma_restrack_put; kfree; ERR_PTR.
- WARN_ON_ONCE(cq.umem) — driver must not set umem in kernel path.
- rdma_restrack_add(&cq.res); return cq.

REQ-16: ib_destroy_cq_user(cq, udata) -> int:
- WARN_ON_ONCE(cq.shared) — return EOPNOTSUPP if so (shared CQs use refcounted destroy).
- if atomic_read(&cq.usecnt) != 0: return -EBUSY.
- ret = cq.device.ops.destroy_cq(cq, udata); on err return.
- ib_umem_release(cq.umem); rdma_restrack_del(&cq.res); kfree(cq); return 0.

REQ-17: create_qp(dev, pd, attr, udata, uobj, caller) -> *ib_qp:
- if !dev.ops.create_qp: return ERR_PTR(-EOPNOTSUPP).
- qp = rdma_zalloc_drv_obj_numa(dev, ib_qp).
- qp.device = dev; qp.pd = pd; qp.uobject = uobj; qp.real_qp = qp.
- qp.qp_type = attr.qp_type; qp.rwq_ind_tbl = attr.rwq_ind_tbl; qp.srq = attr.srq.
- qp.event_handler = __ib_qp_event_handler; qp.registered_event_handler = attr.event_handler.
- qp.port = attr.port_num; qp.qp_context = attr.qp_context.
- spin_lock_init(&qp.mr_lock); INIT_LIST_HEAD(&qp.rdma_mrs); INIT_LIST_HEAD(&qp.sig_mrs); init_completion(&qp.srq_completion).
- qp.send_cq = attr.send_cq; qp.recv_cq = attr.recv_cq.
- rdma_restrack_new(&qp.res, RDMA_RESTRACK_QP); WARN_ONCE(!udata && !caller); rdma_restrack_set_name(&qp.res, if udata { NULL } else { caller }).
- ret = dev.ops.create_qp(qp, attr, udata); on err goto err_create.
- qp.send_cq = attr.send_cq; qp.recv_cq = attr.recv_cq (re-pin; mlx4 quirk).
- ret = ib_create_qp_security(qp, dev); on err goto err_security.
- rdma_restrack_add(&qp.res); return qp.
- err_security: qp.device.ops.destroy_qp(qp, udata ? &dummy : NULL). err_create: rdma_restrack_put; kfree; ERR_PTR.

REQ-18: ib_create_qp_kernel(pd, qp_init_attr, caller) -> *ib_qp:
- device = pd.device.
- If qp_init_attr.cap.max_rdma_ctxs: rdma_rw_init_qp(device, qp_init_attr) — pre-compute resources for the RDMA-RW helper.
- qp = create_qp(device, pd, qp_init_attr, NULL, NULL, caller).
- if IS_ERR(qp) return.
- ib_qp_usecnt_inc(qp) — atomic_inc on pd.usecnt, send_cq.usecnt, recv_cq.usecnt, srq.usecnt (if set), rwq_ind_tbl.usecnt (if set).
- If qp_init_attr.cap.max_rdma_ctxs: ret = rdma_rw_init_mrs(qp, qp_init_attr); on err: ib_destroy_qp; return ERR_PTR.
- qp.max_write_sge = qp_init_attr.cap.max_send_sge.
- qp.max_read_sge = min(qp_init_attr.cap.max_send_sge, device.attrs.max_sge_rd).
- If qp_init_attr.create_flags & IB_QP_CREATE_INTEGRITY_EN: qp.integrity_en = true.
- return qp.

REQ-19: ib_qp_usecnt_inc / ib_qp_usecnt_dec:
- inc: atomic_inc on pd, send_cq, recv_cq, srq, rwq_ind_tbl (each if non-NULL).
- dec: reverse order (rwq_ind_tbl, srq, recv_cq, send_cq, pd).

REQ-20: ib_modify_qp_is_ok(cur_state, next_state, type, mask) -> bool:
- If mask & IB_QP_CUR_STATE: cur_state ∈ {RTR, RTS, SQD, SQE} else return false.
- If !qp_state_table[cur_state][next_state].valid: return false.
- req_param = qp_state_table[cur][next].req_param[type]; opt_param = qp_state_table[cur][next].opt_param[type].
- If (mask & req_param) != req_param: return false.
- If mask & ~(req_param | opt_param | IB_QP_STATE): return false.
- return true.

REQ-21: qp_state_table (exact valid transitions; from upstream verbs.c):
- RESET → RESET (no params).
- RESET → INIT (req: PKEY_INDEX | PORT | ACCESS_FLAGS for UC/RC/XRC; PKEY_INDEX | PORT | QKEY for UD; PORT for RAW_PACKET; PKEY_INDEX | QKEY for SMI/GSI).
- INIT → RESET (no params).
- INIT → ERR.
- INIT → INIT (opt: same as RESET→INIT minus required PORT).
- INIT → RTR (req: AV | PATH_MTU | DEST_QPN | RQ_PSN for UC; plus MAX_DEST_RD_ATOMIC | MIN_RNR_TIMER for RC/XRC_TGT; opt: ALT_PATH | ACCESS_FLAGS | PKEY_INDEX | RATE_LIMIT).
- RTR → RESET / ERR / RTS.
- RTR → RTS (req: SQ_PSN for UD/UC/SMI/GSI; TIMEOUT | RETRY_CNT | RNR_RETRY | SQ_PSN | MAX_QP_RD_ATOMIC for RC/XRC_INI; TIMEOUT | SQ_PSN for XRC_TGT).
- RTS → RESET / ERR / RTS (opt: CUR_STATE | ACCESS_FLAGS | ALT_PATH | PATH_MIG_STATE | MIN_RNR_TIMER | RATE_LIMIT).
- RTS → SQD (opt: EN_SQD_ASYNC_NOTIFY).
- SQD → RESET / ERR / RTS / SQD (opt: per-type re-cfg).
- SQE → RESET / ERR / RTS (opt-limited; SQE valid only for UD/SMI/GSI in upstream).
- ERR → RESET / ERR.
- All non-listed transitions invalid.

REQ-22: _ib_modify_qp(qp, attr, attr_mask, udata) -> int (core):
- port = (attr_mask & IB_QP_PORT) ? attr.port_num : qp.port.
- attr.xmit_slave = NULL.
- If attr_mask & IB_QP_AV: rdma_fill_sgid_attr(qp.device, &attr.ah_attr, &old_sgid_attr_av).
  - If attr.ah_attr.type == RDMA_AH_ATTR_TYPE_ROCE ∧ is_qp_type_connected(qp):
    - If udata: ib_resolve_eth_dmac(qp.device, &attr.ah_attr).
    - slave = rdma_lag_get_ah_roce_slave(qp.device, &attr.ah_attr, GFP_KERNEL); attr.xmit_slave = slave.
- If attr_mask & IB_QP_ALT_PATH: rdma_fill_sgid_attr(qp.device, &attr.alt_ah_attr, &old_sgid_attr_alt_av); only allowed if rdma_protocol_ib for both ports.
- If rdma_ib_or_roce(qp.device, port):
  - If RQ_PSN & ~0xffffff: warn; attr.rq_psn &= 0xffffff.
  - If SQ_PSN & ~0xffffff: warn; attr.sq_psn &= 0xffffff.
- Auto-bind counter at RST2INIT: if !qp.counter ∧ (attr_mask & IB_QP_PORT) ∧ (attr_mask & IB_QP_STATE) ∧ attr.qp_state == IB_QPS_INIT: rdma_counter_bind_qp_auto(qp, attr.port_num).
- ret = ib_security_modify_qp(qp, attr, attr_mask, udata); on err goto out.
- If attr_mask & IB_QP_PORT: qp.port = attr.port_num.
- If attr_mask & IB_QP_AV: qp.av_sgid_attr = rdma_update_sgid_attr(&attr.ah_attr, qp.av_sgid_attr).
- If attr_mask & IB_QP_ALT_PATH: qp.alt_path_sgid_attr = rdma_update_sgid_attr(&attr.alt_ah_attr, qp.alt_path_sgid_attr).
- out: rdma_unfill_sgid_attr cleanup; return ret.

REQ-23: ib_modify_qp / ib_modify_qp_with_udata:
- ib_modify_qp_with_udata(qp, attr, mask, udata): _ib_modify_qp(qp.real_qp, attr, mask, udata).
- ib_modify_qp(qp, attr, mask): _ib_modify_qp(qp.real_qp, attr, mask, NULL) — kernel path.
- Both ultimately invoke qp.device.ops.modify_qp.

REQ-24: ib_destroy_qp_user(qp, udata) -> int:
- if qp.real_qp != qp: __ib_destroy_shared_qp(qp) — XRC shared QP path.
- ib_qp_usecnt_dec(qp).
- If qp.integrity_en: rdma_rw_cleanup_mrs(qp).
- ib_qp_security_destroy(qp).
- ret = qp.device.ops.destroy_qp(qp, udata); on err: re-inc usecnts; return.
- rdma_restrack_del(&qp.res).
- if qp.rdma_mrs not-empty: rdma_rw_cleanup_mrs(qp).
- kfree(qp); return 0.

REQ-25: ib_post_send(qp, send_wr, bad_send_wr) (inline in ib_verbs.h):
- dummy: *const ib_send_wr.
- return qp.device.ops.post_send(qp, send_wr, bad_send_wr ? bad_send_wr : &dummy).
- Contract: returns 0 on success or first-error error code; *bad_send_wr points to first WR that failed.
- Per-IBA Vol. 1 § 11.4.1.1: any earlier WRs in the list are still queued; only the failing-and-later WRs are not.

REQ-26: ib_post_recv(qp, recv_wr, bad_recv_wr) (inline):
- Symmetric to post_send; ops.post_recv.

REQ-27: ib_post_srq_recv(srq, recv_wr, bad_recv_wr) (inline):
- ops.post_srq_recv(srq, recv_wr, bad_recv_wr ? : &dummy).

REQ-28: ib_poll_cq(cq, num_entries, wc) (inline):
- return cq.device.ops.poll_cq(cq, num_entries, wc).
- Returns N >= 0 (entries written into wc[]) or negative error.

REQ-29: ib_req_notify_cq(cq, flags) (inline):
- return cq.device.ops.req_notify_cq(cq, flags).
- flags: IB_CQ_SOLICITED | IB_CQ_NEXT_COMP | IB_CQ_REPORT_MISSED_EVENTS.
- Returns 0 (armed; comp_handler will fire) or >0 (one or more completions were missed; consumer must immediately ib_poll_cq again).

REQ-30: ib_drain_qp / ib_drain_sq / ib_drain_rq:
- Caller-supplied graceful drain.
- ib_drain_sq: if ops.drain_sq: ops.drain_sq(qp); else __ib_drain_sq (modify to ERR, post WRITE WR with private wr_cqe, wait completion).
- ib_drain_rq: symmetric for recv side; if qp.srq use __ib_drain_srq (wait for LAST_WQE_REACHED event).
- ib_drain_qp: ib_drain_sq + ib_drain_rq.

REQ-31: ib_attach_mcast / ib_detach_mcast:
- attach: validate gid.raw[0]==0xff (multicast); validate lid in [0xC000, 0xFFFE]; return ops.attach_mcast(qp, gid, lid).
- detach: ops.detach_mcast.

REQ-32: ib_reg_user_mr(pd, start, length, virt_addr, access_flags) -> *ib_mr:
- If access_flags & IB_ACCESS_ON_DEMAND: require pd.device.attrs.kernel_cap_flags & IBK_ON_DEMAND_PAGING.
- mr = pd.device.ops.reg_user_mr(pd, start, length, virt_addr, access_flags, NULL, NULL); on ERR return.
- Wire mr.device, mr.type = IB_MR_TYPE_USER, mr.pd, mr.dm = NULL, atomic_inc(pd.usecnt), mr.iova = virt_addr, mr.length = length.
- rdma_restrack_new(&mr.res, RDMA_RESTRACK_MR); rdma_restrack_parent_name(&mr.res, &pd.res); rdma_restrack_add. return mr.

REQ-33: ib_alloc_mr(pd, mr_type, max_num_sg) -> *ib_mr:
- if !pd.device.ops.alloc_mr: return ERR_PTR(-EOPNOTSUPP).
- if mr_type == IB_MR_TYPE_INTEGRITY: return ERR_PTR(-EINVAL) (must use ib_alloc_mr_integrity).
- mr = pd.device.ops.alloc_mr(pd, mr_type, max_num_sg); on ERR return.
- Wire mr fields; rdma_restrack_new/parent/add. return mr.

REQ-34: ib_dereg_mr_user(mr, udata) -> int:
- ret = mr.device.ops.dereg_mr(mr, udata); on err return.
- atomic_dec(&pd.usecnt); if dm: atomic_dec(&dm.usecnt); rdma_restrack_del(&mr.res). return 0.

REQ-35: ib_map_mr_sg(mr, sg, sg_nents, sg_offset, page_size):
- mr.page_size = page_size; return mr.device.ops.map_mr_sg(mr, sg, sg_nents, sg_offset).
- Variant ib_map_mr_sg_pi for protection-information (T10-PI / DIF).
- Variant ib_sg_to_pages: iterate sgl populating a kernel-managed page list, invoke set_page callback per element.

REQ-36: ib_create_srq_user(pd, srq_init_attr, uobject, udata):
- srq = rdma_zalloc_drv_obj(pd.device, ib_srq).
- Wire srq.device/pd/event_handler/srq_context/srq_type/uobject.
- If ib_srq_has_cq(srq.srq_type) (XRC or TM): srq.ext.cq = srq_init_attr.ext.cq; atomic_inc(&ext.cq.usecnt).
- If srq.srq_type == IB_SRQT_XRC: srq.ext.xrc.xrcd = srq_init_attr.ext.xrc.xrcd; atomic_inc(&xrcd.usecnt) (if set).
- atomic_inc(&pd.usecnt). rdma_restrack_new + parent_name.
- ret = pd.device.ops.create_srq(srq, srq_init_attr, udata); on err: undo refcounts + kfree.
- rdma_restrack_add. return srq.

REQ-37: ib_destroy_srq_user(srq, udata) -> int:
- if atomic_read(&srq.usecnt) != 0: return -EBUSY.
- ret = srq.device.ops.destroy_srq(srq, udata); on err return.
- atomic_dec(&pd.usecnt); if XRC: atomic_dec(&xrcd.usecnt); if has_cq: atomic_dec(&ext.cq.usecnt).
- rdma_restrack_del. kfree. return 0.

REQ-38: AH lifecycle (rdma_create_ah / rdma_modify_ah / rdma_query_ah / rdma_destroy_ah_user):
- create: validate ah_attr (rdma_check_ah_attr); fill SGID (rdma_fill_sgid_attr); _rdma_create_ah → ops.create_ah; atomic_inc(&pd.usecnt); restrack.
- modify: ops.modify_ah(ah, ah_attr).
- query: ops.query_ah(ah, ah_attr).
- destroy: ops.destroy_ah(ah, flags); atomic_dec(&pd.usecnt); rdma_destroy_ah_attr; rdma_restrack_del.

REQ-39: Per-QP type "connected" predicate (is_qp_type_connected):
- true iff qp.qp_type ∈ {UC, RC, XRC_INI, XRC_TGT}.
- Used by _ib_modify_qp to decide whether to do RoCE eth-dmac resolution + LAG slave selection.

REQ-40: Event/status string tables:
- ib_event_msg(event) — returns string for IB_EVENT_* (CQ_ERR / QP_FATAL / QP_REQ_ERR / QP_ACCESS_ERR / COMM_EST / SQ_DRAINED / PATH_MIG / PATH_MIG_ERR / DEVICE_FATAL / PORT_ACTIVE / PORT_ERR / LID_CHANGE / PKEY_CHANGE / SM_CHANGE / SRQ_ERR / SRQ_LIMIT_REACHED / QP_LAST_WQE_REACHED / CLIENT_REREGISTER / GID_CHANGE).
- ib_wc_status_msg(status) — returns string for IB_WC_* (REQ-6).

## Acceptance Criteria

- [ ] AC-1: __ib_alloc_pd returns a valid pd with restrack-registered res and usecnt==0; ERR_PTR(-EOPNOTSUPP) if ops.alloc_pd missing.
- [ ] AC-2: ib_create_qp_kernel calls ib_qp_usecnt_inc; ib_destroy_qp calls ib_qp_usecnt_dec; atomic refcounts on pd/send_cq/recv_cq/srq balanced across create/destroy.
- [ ] AC-3: ib_modify_qp_is_ok rejects transitions not present in qp_state_table.
- [ ] AC-4: ib_modify_qp RESET→INIT for IB_QPT_RC with mask missing IB_QP_PKEY_INDEX returns false from is_modify_ok.
- [ ] AC-5: ib_modify_qp legal cur→next with all req_param bits set: ops.modify_qp invoked.
- [ ] AC-6: _ib_modify_qp truncates rq_psn/sq_psn to 24 bits when port is IB or RoCE.
- [ ] AC-7: ib_post_send with NULL bad_send_wr uses an internal dummy; returns ops.post_send return value verbatim.
- [ ] AC-8: ib_post_recv passes bad_recv_wr through (or dummy when caller passed NULL).
- [ ] AC-9: ib_poll_cq returns ops.poll_cq verdict; on success writes 0..num_entries struct ib_wc.
- [ ] AC-10: ib_req_notify_cq with IB_CQ_SOLICITED arms solicited-only; IB_CQ_NEXT_COMP arms next; positive return ⟹ at least one missed event ⟹ consumer must re-poll.
- [ ] AC-11: ib_post_srq_recv routes through ops.post_srq_recv on srq.device.
- [ ] AC-12: ib_destroy_cq returns -EBUSY when cq.usecnt > 0.
- [ ] AC-13: ib_destroy_srq returns -EBUSY when srq.usecnt > 0.
- [ ] AC-14: ib_attach_mcast rejects non-multicast gid (gid.raw[0] != 0xff) and out-of-range lid.
- [ ] AC-15: ib_drain_qp moves QP through IB_QPS_ERR, posts internal drain WRs to both queues, returns only after completions reaped (or LAST_WQE_REACHED for SRQ-bound recv).

## Architecture

```
struct IbPd {
  device: *mut IbDevice,
  uobject: Option<*mut IbUobject>,
  usecnt: AtomicI32,
  flags: u32,
  local_dma_lkey: u32,
  unsafe_global_rkey: u32,
  __internal_mr: Option<*mut IbMr>,
  res: RdmaRestrackEntry,
}

struct IbCq {
  device: *mut IbDevice,
  uobject: Option<*mut IbUobject>,
  comp_handler: Option<fn(&mut IbCq, *mut c_void)>,
  event_handler: Option<fn(&IbEvent, *mut c_void)>,
  cq_context: *mut c_void,
  cqe: i32,
  usecnt: AtomicI32,
  poll_ctx: u8,                          // IB_POLL_*
  shared: bool,
  res: RdmaRestrackEntry,
  // additional driver-private memory follows (rdma_zalloc_drv_obj)
}

struct IbQp {
  device: *mut IbDevice,
  pd: *mut IbPd,
  send_cq: *mut IbCq,
  recv_cq: *mut IbCq,
  srq: Option<*mut IbSrq>,
  rwq_ind_tbl: Option<*mut IbRwqIndTable>,
  real_qp: *mut IbQp,                    // self for non-XRC; parent for shared XRC
  uobject: Option<*mut IbUqpObject>,
  event_handler: fn(&IbEvent, *mut c_void),
  registered_event_handler: Option<fn(&IbEvent, *mut c_void)>,
  qp_context: *mut c_void,
  qp_num: u32,
  qp_type: u8,                           // enum ib_qp_type
  port: u32,
  state: u8,                             // tracked by driver via modify_qp
  max_write_sge: u32,
  max_read_sge: u32,
  integrity_en: bool,
  counter: Option<*mut RdmaCounter>,
  av_sgid_attr: Option<*const IbGidAttr>,
  alt_path_sgid_attr: Option<*const IbGidAttr>,
  mr_lock: SpinLock,
  rdma_mrs: ListHead,
  sig_mrs: ListHead,
  srq_completion: Completion,
  res: RdmaRestrackEntry,
}

struct IbMr {
  device: *mut IbDevice,
  pd: *mut IbPd,
  dm: Option<*mut IbDm>,
  uobject: Option<*mut IbUobject>,
  type_: u8,                             // IB_MR_TYPE_USER / _DMA / _MEM_REG / _INTEGRITY
  lkey: u32,
  rkey: u32,
  iova: u64,
  length: u64,
  page_size: u32,
  need_inval: bool,
  res: RdmaRestrackEntry,
}

struct IbSrq {
  device: *mut IbDevice,
  pd: *mut IbPd,
  event_handler: Option<fn(&IbEvent, *mut c_void)>,
  srq_context: *mut c_void,
  uobject: Option<*mut IbUsrqObject>,
  srq_type: u8,                          // BASIC / XRC / TM
  usecnt: AtomicI32,
  ext: IbSrqExt,                         // union: cq (XRC/TM) + xrc { xrcd, srq_num } (XRC)
  res: RdmaRestrackEntry,
}

struct IbWc {
  wr_id_or_cqe: WrIdUnion,               // u64 wr_id or *IbCqe
  status: u8,                            // enum ib_wc_status
  opcode: u8,                            // enum ib_wc_opcode (bit 7 = recv)
  vendor_err: u32,
  byte_len: u32,
  qp: *mut IbQp,
  ex: WcExUnion,                         // imm_data __be32 | invalidate_rkey u32
  src_qp: u32,
  slid: u32,
  wc_flags: i32,
  pkey_index: u16,
  sl: u8,
  dlid_path_bits: u8,
  port_num: u32,
  smac: [u8; 6],
  vlan_id: u16,
  network_hdr_type: u8,
}

struct IbSendWr {
  next: Option<*const IbSendWr>,
  wr_id_or_cqe: WrIdUnion,
  sg_list: *const IbSge,
  num_sge: i32,
  opcode: u8,                            // enum ib_wr_opcode
  send_flags: i32,
  ex: WrExUnion,                         // imm_data __be32 | invalidate_rkey u32
}

struct IbRecvWr {
  next: Option<*const IbRecvWr>,
  wr_id_or_cqe: WrIdUnion,
  sg_list: *const IbSge,
  num_sge: i32,
}
```

`IbQp::STATE_TABLE: [[Transition; 7]; 7]`:

Constant Rust translation of upstream `qp_state_table[IB_QPS_ERR + 1][IB_QPS_ERR + 1]`. Each `Transition { valid: bool, req_param: [IbQpAttrMask; IB_QPT_MAX], opt_param: [IbQpAttrMask; IB_QPT_MAX] }`. Population per REQ-21. Indexed as STATE_TABLE[cur][next].

`IbQp::is_modify_ok(cur, next, ty, mask) -> bool`:
1. If mask & IB_QP_CUR_STATE ∧ cur ∉ {RTR, RTS, SQD, SQE}: return false.
2. let t = STATE_TABLE[cur as usize][next as usize].
3. If !t.valid: return false.
4. req = t.req_param[ty]; opt = t.opt_param[ty].
5. If (mask & req) != req: return false.
6. If mask & !(req | opt | IB_QP_STATE) != 0: return false.
7. return true.

`IbPd::alloc(device, flags, caller) -> Result<*mut IbPd>`:
1. pd = rdma_zalloc_drv_obj(device, ib_pd); on ENOMEM return.
2. pd.device = device; pd.flags = flags.
3. rdma_restrack_new(&pd.res, RDMA_RESTRACK_PD); rdma_restrack_set_name(&pd.res, caller).
4. ret = device.ops.alloc_pd(pd, NULL); on err: restrack_put; kfree; return Err.
5. rdma_restrack_add(&pd.res).
6. If device.attrs.kernel_cap_flags & IBK_LOCAL_DMA_LKEY: pd.local_dma_lkey = device.local_dma_lkey; else mr_flags |= IB_ACCESS_LOCAL_WRITE.
7. If flags & IB_PD_UNSAFE_GLOBAL_RKEY: warn; mr_flags |= IB_ACCESS_REMOTE_READ | IB_ACCESS_REMOTE_WRITE.
8. If mr_flags != 0: mr = device.ops.get_dma_mr(pd, mr_flags); on ERR: dealloc_pd; return. Wire mr; pd.__internal_mr = Some(mr). Derive pd.local_dma_lkey / pd.unsafe_global_rkey.
9. return Ok(pd).

`IbCq::create(device, comp_handler, event_handler, cq_context, cq_attr, caller) -> Result<*mut IbCq>`:
1. cq = rdma_zalloc_drv_obj(device, ib_cq).
2. If cq_attr.cqe == 0: WARN_ON_ONCE; return Err(EINVAL).
3. Wire cq.device/comp_handler/event_handler/cq_context; atomic_set(&cq.usecnt, 0).
4. rdma_restrack_new / set_name.
5. ret = device.ops.create_cq(cq, cq_attr, NULL); on err: restrack_put; kfree; Err.
6. WARN_ON_ONCE(cq.umem).
7. rdma_restrack_add. return Ok.

`IbQp::create_kernel(pd, attr, caller) -> Result<*mut IbQp>`:
1. device = pd.device.
2. If attr.cap.max_rdma_ctxs: rdma_rw_init_qp(device, attr).
3. qp = IbQp::construct(device, pd, attr, NULL, NULL, caller).
4. ib_qp_usecnt_inc(qp).
5. If attr.cap.max_rdma_ctxs: rdma_rw_init_mrs(qp, attr); on err: ib_destroy_qp; return.
6. qp.max_write_sge = attr.cap.max_send_sge.
7. qp.max_read_sge = min(attr.cap.max_send_sge, device.attrs.max_sge_rd).
8. If attr.create_flags & IB_QP_CREATE_INTEGRITY_EN: qp.integrity_en = true.
9. return Ok(qp).

`IbQp::modify(qp, attr, mask) -> Result<()>` (kernel path):
1. real = qp.real_qp.
2. port = if mask & IB_QP_PORT { attr.port_num } else { real.port }.
3. If mask & IB_QP_AV: rdma_fill_sgid_attr; if RoCE ∧ is_qp_type_connected(real): NULL udata path leaves dmac to caller; rdma_lag_get_ah_roce_slave; attr.xmit_slave = slave.
4. If mask & IB_QP_ALT_PATH: rdma_fill_sgid_attr for alt; require both ports rdma_protocol_ib.
5. If rdma_ib_or_roce(real.device, port): rq_psn &= 0xffffff; sq_psn &= 0xffffff (with dev_warn).
6. Auto-bind counter on RST2INIT (if RST→INIT ∧ no counter ∧ port mask set).
7. ib_security_modify_qp(real, attr, mask, NULL).
8. If mask & IB_QP_PORT: real.port = attr.port_num.
9. If mask & IB_QP_AV: real.av_sgid_attr = rdma_update_sgid_attr(&attr.ah_attr, real.av_sgid_attr).
10. If mask & IB_QP_ALT_PATH: real.alt_path_sgid_attr = rdma_update_sgid_attr(&attr.alt_ah_attr, real.alt_path_sgid_attr).
11. cleanup (rdma_unfill_sgid_attr, rdma_lag_put). return Ok / Err.

`IbQp::post_send(qp, send_wr, bad_send_wr) -> Result<()>`:
1. dummy: *const IbSendWr.
2. let bad = bad_send_wr.unwrap_or(&dummy).
3. return qp.device.ops.post_send(qp, send_wr, bad).
4. Per-IBA: on first-error, earlier WRs in the chain remain queued.

`IbQp::post_recv(qp, recv_wr, bad_recv_wr) -> Result<()>`:
1. Symmetric to post_send via ops.post_recv.

`IbCq::poll(cq, num_entries, wc) -> Result<usize>`:
1. n = cq.device.ops.poll_cq(cq, num_entries, wc).
2. n >= 0 ⟹ Ok(n as usize); n < 0 ⟹ Err.
3. wc[0..n] populated; consumers test wc[i].opcode & IB_WC_RECV for send-vs-recv.

`IbCq::req_notify(cq, flags) -> Result<bool>`:
1. r = cq.device.ops.req_notify_cq(cq, flags).
2. r == 0 ⟹ Ok(false) (armed cleanly).
3. r > 0 ⟹ Ok(true) (events were missed during arm; consumer MUST immediately ib_poll_cq).
4. r < 0 ⟹ Err.

`IbSrq::post_recv(srq, recv_wr, bad_recv_wr) -> Result<()>`:
1. dummy: *const IbRecvWr.
2. return srq.device.ops.post_srq_recv(srq, recv_wr, bad_recv_wr.unwrap_or(&dummy)).

`IbQp::drain(qp)`:
1. IbQp::drain_sq(qp).
2. IbQp::drain_rq(qp).
3. Both: if ops.drain_sq/drain_rq present: dispatch; else fallback __ib_drain_sq/__ib_drain_rq (modify-to-ERR, post private CQE-bearing WR, wait for completion via wait_for_completion).
4. If qp.srq: drain_rq uses __ib_drain_srq (wait for IB_EVENT_QP_LAST_WQE_REACHED on srq_completion).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pd_usecnt_balanced` | INVARIANT | per-PD: every alloc_pd has matching dealloc_pd; usecnt==0 at dealloc. |
| `cq_busy_blocks_destroy` | INVARIANT | per-CQ destroy: usecnt>0 ⟹ EBUSY. |
| `srq_busy_blocks_destroy` | INVARIANT | per-SRQ destroy: usecnt>0 ⟹ EBUSY. |
| `qp_usecnt_balanced` | INVARIANT | per-QP: ib_qp_usecnt_inc paired with ib_qp_usecnt_dec across create/destroy. |
| `state_table_transitions` | INVARIANT | per-modify_qp: ib_modify_qp_is_ok ⟺ (cur,next) listed in qp_state_table with req_param subset of mask. |
| `psn_24bit_truncation` | INVARIANT | per-_ib_modify_qp on ib_or_roce port: rq_psn/sq_psn masked to 0xffffff. |
| `bad_wr_dummy_on_null` | INVARIANT | per-ib_post_send/recv: NULL bad_wr ⟹ uses internal dummy, no NULL deref. |
| `poll_cq_writes_le_num_entries` | INVARIANT | per-ib_poll_cq: returned N <= num_entries; wc[0..N] populated. |
| `req_notify_positive_then_poll` | INVARIANT | per-ib_req_notify_cq returns >0: consumer poll loop must re-invoke ib_poll_cq. |
| `mcast_gid_first_byte_ff` | INVARIANT | per-ib_attach_mcast: gid.raw[0] == 0xff. |
| `mcast_lid_in_range` | INVARIANT | per-ib_attach_mcast: lid ∈ [0xC000, 0xFFFE]. |
| `ah_pd_usecnt_balanced` | INVARIANT | per-AH create/destroy: pd.usecnt inc/dec balanced. |
| `mr_pd_usecnt_balanced` | INVARIANT | per-MR reg/dereg: pd.usecnt inc/dec balanced. |

### Layer 2: TLA+

`drivers/infiniband/core/verbs.tla`:
- QP state machine: RESET / INIT / RTR / RTS / SQD / SQE / ERR with transitions per qp_state_table.
- Per-CQ: posted-WR → driver processing → wc-enqueue → ib_poll_cq drains → ib_req_notify_cq arms.
- Per-SRQ: shared recv-WR pool across many UD/XRC QPs.
- Properties:
  - `safety_no_invalid_transition` — per-modify_qp: ib_modify_qp_is_ok=false ⟹ ops.modify_qp not invoked.
  - `safety_required_attrs_present` — per-valid-transition: (mask ⊇ req_param) for the (cur,next,type) triple.
  - `safety_post_send_only_in_RTS_RTR_SQD_SQE` (per-IBA): driver-level discipline; core never gates but ULPs must.
  - `safety_post_recv_only_after_INIT` — per-IBA: ops.post_recv from RESET returns -EINVAL.
  - `safety_cq_usecnt_zero_at_destroy` — per-destroy_cq: cq.usecnt == 0 invariant.
  - `safety_srq_usecnt_zero_at_destroy` — per-destroy_srq: srq.usecnt == 0 invariant.
  - `safety_pd_internal_mr_destroyed_before_pd` — per-dealloc_pd: __internal_mr dereg'd first.
  - `liveness_drain_qp_terminates` — per-drain: completion observed in bounded time after modify→ERR.
  - `liveness_req_notify_completes` — per-arm: comp_handler eventually fires after next completion (or returns >0 indicating already-missed).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IbPd::alloc` post: on Ok pd registered in restrack ∧ usecnt==0 ∧ device.ops.alloc_pd called exactly once | `IbPd::alloc` |
| `IbCq::create` post: on Ok cq registered ∧ usecnt==0 ∧ umem==NULL ∧ ops.create_cq called | `IbCq::create` |
| `IbQp::create_kernel` post: on Ok qp_usecnt_inc applied ∧ max_write_sge / max_read_sge set ∧ restrack entry added | `IbQp::create_kernel` |
| `IbQp::is_modify_ok` post: returns true ⟺ STATE_TABLE[cur][next].valid ∧ required params present ∧ no foreign mask bits | `IbQp::is_modify_ok` |
| `IbQp::modify` post: ops.modify_qp called only if is_modify_ok holds; psn-masked when ib_or_roce | `IbQp::modify` |
| `IbQp::post_send` post: result == ops.post_send(qp, send_wr, bad_or_dummy) | `IbQp::post_send` |
| `IbCq::poll` post: returned N <= num_entries ∧ ∀ i<N: wc[i].qp ∈ {qps on this cq} | `IbCq::poll` |
| `IbCq::req_notify` post: returned 0 ⟹ comp_handler armed; returned >0 ⟹ caller must poll immediately | `IbCq::req_notify` |
| `IbSrq::post_recv` post: result == ops.post_srq_recv(srq, recv_wr, bad_or_dummy) | `IbSrq::post_recv` |
| `IbQp::drain` post: send + recv queues drained; qp.state == ERR | `IbQp::drain` |

### Layer 4: Verus/Creusot functional

`Per-PD alloc → per-CQ create → per-QP create (RESET) → modify(RESET→INIT) → modify(INIT→RTR) → modify(RTR→RTS) → post_send/post_recv → poll_cq → req_notify_cq → drain → destroy_qp → destroy_cq → dealloc_pd` semantic equivalence: per-IBA Architecture Specification Vol. 1 § 10 (QP), § 11.4 (verbs), § 11.6.2 (work-request format), § 11.6.4 (work-completion format), § 11.4.2 (QP state machine); per-Documentation/infiniband/user_verbs.rst; per-`qp_state_table` byte-identical to upstream verbs.c. EXPORT_SYMBOL set (~70 symbols) must match upstream for module-binary compat with all in-tree and out-of-tree consumers (ipoib, rdma_cm, srp, iser, rds_rdma, nvmet-rdma, smb-direct, mlx5_ib, hfi1, irdma, bnxt_re, rxe, siw, efa, mana, erdma).

## Hardening

(Inherits row-1 features from `drivers/infiniband/00-overview.md` § Hardening.)

Verbs-API reinforcement:

- **Per-qp_state_table strict validation** — defense against per-out-of-spec QP transition trapping driver to undefined behavior.
- **Per-req_param subset check** — defense against per-incomplete-modify (e.g., RESET→INIT without PKEY_INDEX) silently corrupting QP context.
- **Per-foreign-mask-bit rejection** — defense against per-mask-typo accidentally clearing wrong attributes.
- **Per-24-bit PSN truncation with dev_warn** — defense against per-IBA-overflow silent wraparound on InfiniBand/RoCE ports.
- **Per-usecnt EBUSY on destroy** — defense against per-destroy-while-bound (CQ has live QPs; SRQ has live QPs; PD has live MRs).
- **Per-bad_wr dummy on NULL** — defense against per-driver-NULL-deref when caller declines to inspect failed WR.
- **Per-WARN_ON cq.umem in kernel path** — defense against per-driver-bug treating kernel CQ as userspace CQ.
- **Per-WARN_ON cq.shared in destroy** — defense against per-shared-CQ improper teardown (must use ib_cq_pool_put).
- **Per-multicast LID range + GID prefix check** — defense against per-spec-violation attach_mcast.
- **Per-rdma_restrack lifecycle** — defense against per-resource-leak: every allocated PD/CQ/QP/MR/SRQ/AH has restrack entry from alloc to free; orphans show up in /sys/kernel/debug/rdma.
- **Per-rdma_zalloc_drv_obj size from ops.size_*** — defense against per-driver-private-area underrun.
- **Per-IB_PD_UNSAFE_GLOBAL_RKEY pr_warn** — defense against per-silent-enabling-of-unsafe-mode (used by sw drivers only).
- **Per-ib_qp_usecnt_inc/dec on PD+CQ+SRQ+rwq_ind_tbl** — defense against per-destroy-out-of-order (PD freed while QP still references it).
- **Per-ib_security_modify_qp gate** — defense against per-LSM-bypass on QP attr change.
- **Per-counter auto-bind on RST→INIT only** — defense against per-mid-life rebinding causing counter delta jumps.
- **Per-rdma_lag_get_ah_roce_slave bound to attr lifetime** — defense against per-slave-netdev-disappearing race.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/infiniband/core/device.c` device registration (covered in `device.md` Tier-3)
- `drivers/infiniband/core/cq.c` CQ pool + IB_POLL_WORKQUEUE/SOFTIRQ/DIRECT polling-context dispatch (covered separately if expanded)
- `drivers/infiniband/core/uverbs_cmd.c` + `uverbs_main.c` + `uverbs_std_types_*.c` uverbs ABI (covered separately if expanded)
- `drivers/infiniband/core/cache.c` GID + PKey cache (covered separately if expanded)
- `drivers/infiniband/core/cma.c` rdma_cm connection management (covered separately if expanded)
- `drivers/infiniband/core/rw.c` RDMA-RW helper (rdma_rw_init_qp / rdma_rw_init_mrs / rdma_rw_ctx_init) (covered separately if expanded)
- `drivers/infiniband/core/security.c` ib_security_modify_qp + uverbs context security (covered separately if expanded)
- `drivers/infiniband/core/restrack.c` resource tracker (covered separately if expanded)
- `drivers/infiniband/core/counters.c` per-QP/per-port rdma_counter (covered separately if expanded)
- `drivers/infiniband/core/lag.c` RoCE LAG slave selection (covered separately if expanded)
- Per-driver HCA ops implementations (covered in per-HCA Tier-3 if expanded)
- Implementation code
