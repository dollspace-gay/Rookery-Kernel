# Tier-3: drivers/nvme/host/tcp.c — NVMe-over-TCP transport (host)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/nvme/00-overview.md
upstream-paths:
  - drivers/nvme/host/tcp.c (~3088 lines)
  - drivers/nvme/host/fabrics.c (covered in host-fabrics.md)
  - include/linux/nvme-tcp.h (struct nvme_tcp_hdr / cmd_pdu / data_pdu / rsp_pdu / r2t_pdu / icreq_pdu / icresp_pdu / term_pdu, NVME_TCP_F_* flags)
  - net/tls/* (kTLS, tls_handshake)
-->

## Summary

`drivers/nvme/host/tcp.c` implements the **host side of the NVMe-over-TCP transport** (NVMe-oF/TCP, NVMe-TCP). It speaks a streaming PDU protocol over a kernel TCP socket: PDU types `ICReq`/`ICResp` (Initialize Connection), `CapsuleCmd` (= `nvme_tcp_cmd_pdu`, SQE-bearing), `CapsuleResp` (= `nvme_tcp_rsp_pdu`, CQE-bearing), `H2CData` (= `nvme_tcp_data_pdu` Host-to-Controller), `C2HData` (Controller-to-Host), `R2T` (Ready-to-Transfer, controller asks host for write data), and `C2HTermReq` (fatal terminator). Each TCP connection is exactly one **NVMe queue** (qid 0 = admin, qid > 0 = IO); per-controller queue count = `1 + nr_io_queues + nr_write_queues + nr_poll_queues`. Per-queue uses two state machines: a **send state** (`NVME_TCP_SEND_CMD_PDU → SEND_H2C_PDU → SEND_DATA → SEND_DDGST`) driven by `nvme_tcp_io_work` via `sock_sendmsg`/`kernel_sendmsg` with `MSG_SPLICE_PAGES | MSG_MORE | MSG_EOR`, and a **recv state** (`NVME_TCP_RECV_PDU → RECV_DATA → RECV_DDGST`) driven by the socket's `sk_data_ready` callback funneling into `nvme_tcp_recv_skb` (a `read_descriptor_t` callback installed via `sock->ops->read_sock`). Optional features: **kTLS** (`nvme_tcp_start_tls` via user-space `tlshd` handshake daemon + key-API PSK lookup), **header digest** (CRC32C over the PDU header), **data digest** (CRC32C over data payload). Critical for: cross-fabric block storage at scale on commodity Ethernet, TLS-encrypted NVMe sessions, dynamic queue mapping (DEFAULT / READ / POLL).

This Tier-3 covers `drivers/nvme/host/tcp.c` (~3088 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nvme_tcp_ctrl` | per-controller wrapper (1 list, queues array, async_req) | `NvmeTcpCtrl` |
| `struct nvme_tcp_queue` | per-connection state (sock, send/recv state) | `NvmeTcpQueue` |
| `struct nvme_tcp_request` | per-rq PDU + iter state | `NvmeTcpRequest` |
| `enum nvme_tcp_send_state` | per-req TX state machine | `NvmeTcpSendState` |
| `enum nvme_tcp_recv_state` | per-queue RX state machine | `NvmeTcpRecvState` |
| `enum nvme_tcp_queue_flags` | per-queue Q_ALLOCATED / Q_LIVE / Q_POLLING / Q_IO_CPU_SET | `NvmeTcpQueueFlag` |
| `nvme_tcp_init_connection()` | per-queue ICReq/ICResp handshake | `NvmeTcp::init_connection` |
| `nvme_tcp_alloc_queue()` | per-qid socket create / connect / start_tls / init | `NvmeTcp::alloc_queue` |
| `nvme_tcp_start_queue()` | per-qid set sock_ops, change LIVE | `NvmeTcp::start_queue` |
| `nvme_tcp_stop_queue()` / `_nowait()` / `wait_queue()` | per-qid shutdown | `NvmeTcp::stop_queue` |
| `nvme_tcp_free_queue()` | per-qid sock close, free pdu | `NvmeTcp::free_queue` |
| `nvme_tcp_alloc_admin_queue()` | qid=0 + TLS lookup | `NvmeTcp::alloc_admin_queue` |
| `nvme_tcp_alloc_io_queues()` / `__nvme_tcp_alloc_io_queues()` | qid>0 batch | `NvmeTcp::alloc_io_queues` |
| `nvme_tcp_io_work()` | per-queue send+recv work | `NvmeTcp::io_work` |
| `nvme_tcp_try_send()` | per-queue TX state machine | `NvmeTcp::try_send` |
| `nvme_tcp_try_send_cmd_pdu()` | per-req CapsuleCmd send | `NvmeTcp::try_send_cmd_pdu` |
| `nvme_tcp_try_send_data_pdu()` | per-req H2C PDU header | `NvmeTcp::try_send_data_pdu` |
| `nvme_tcp_try_send_data()` | per-req data payload | `NvmeTcp::try_send_data` |
| `nvme_tcp_try_send_ddgst()` | per-req data digest tail | `NvmeTcp::try_send_ddgst` |
| `nvme_tcp_try_recv()` | per-queue RX driver | `NvmeTcp::try_recv` |
| `nvme_tcp_recv_skb()` | per-skb dispatch (PDU/DATA/DDGST) | `NvmeTcp::recv_skb` |
| `nvme_tcp_recv_pdu()` | per-PDU header parse | `NvmeTcp::recv_pdu` |
| `nvme_tcp_recv_data()` | per-payload copy + CRC | `NvmeTcp::recv_data` |
| `nvme_tcp_recv_ddgst()` | per-DDGST verify | `NvmeTcp::recv_ddgst` |
| `nvme_tcp_handle_comp()` / `_process_nvme_cqe()` | per-CQE dispatch | `NvmeTcp::handle_comp` |
| `nvme_tcp_handle_c2h_data()` | per-C2HData PDU | `NvmeTcp::handle_c2h_data` |
| `nvme_tcp_handle_r2t()` | per-R2T PDU → setup H2C | `NvmeTcp::handle_r2t` |
| `nvme_tcp_handle_c2h_term()` | per-fatal terminator | `NvmeTcp::handle_c2h_term` |
| `nvme_tcp_setup_h2c_data_pdu()` | per-R2T build H2C header | `NvmeTcp::setup_h2c_data_pdu` |
| `nvme_tcp_data_ready()` / `_write_space()` / `_state_change()` | per-socket callbacks | `NvmeTcp::sock_cb_*` |
| `nvme_tcp_setup_sock_ops()` / `_restore_sock_ops()` | per-queue install/restore sk callbacks | `NvmeTcp::sock_ops_*` |
| `nvme_tcp_reclassify_socket()` | per-lockdep class fix | `NvmeTcp::reclassify_socket` |
| `nvme_tcp_init_recv_ctx()` | per-PDU RX reset | `NvmeTcp::init_recv_ctx` |
| `nvme_tcp_recv_state()` | per-queue derive state | `NvmeTcp::recv_state` |
| `nvme_tcp_verify_hdgst()` / `_check_ddgst()` / `_set_hdgst()` / `_ddgst_update()` / `_ddgst_final()` / `_hdgst()` | per-CRC32C | `NvmeTcp::digest_*` |
| `nvme_tcp_alloc_async_req()` / `_free_async_req()` | per-ctrl async event SQE | `NvmeTcp::*_async_req` |
| `nvme_tcp_async_req()` | per-req predicate | `NvmeTcp::is_async_req` |
| `nvme_tcp_submit_async_event()` | per-ctrl AEN submit | `NvmeTcp::submit_async_event` |
| `nvme_tcp_setup_cmd_pdu()` / `_map_data()` | per-rq build CapsuleCmd | `NvmeTcp::setup_cmd_pdu` |
| `nvme_tcp_set_sg_*` | per-cmd SGL descriptor | `NvmeTcp::set_sg_*` |
| `nvme_tcp_queue_rq()` / `_commit_rqs()` | blk-mq dispatch hooks | `NvmeTcp::queue_rq` |
| `nvme_tcp_poll()` | per-hctx irq-poll | `NvmeTcp::poll` |
| `nvme_tcp_timeout()` / `_complete_timed_out()` | per-rq timeout handler | `NvmeTcp::timeout` |
| `nvme_tcp_map_queues()` | tag-set queue map | `NvmeTcp::map_queues` |
| `nvme_tcp_set_queue_io_cpu()` | per-queue cpu pin | `NvmeTcp::set_queue_io_cpu` |
| `nvme_tcp_admin_queue()` / `_default_queue()` / `_read_queue()` / `_poll_queue()` | per-qid classifier | `NvmeTcp::queue_class` |
| `nvme_tcp_start_tls()` / `_tls_done()` | per-queue TLS handshake | `NvmeTcp::start_tls` |
| `nvme_tcp_setup_ctrl()` | per-ctrl admin+IO config | `NvmeTcp::setup_ctrl` |
| `nvme_tcp_configure_admin_queue()` / `_configure_io_queues()` | per-ctrl admin/io bring-up | `NvmeTcp::configure_*` |
| `nvme_tcp_teardown_admin_queue()` / `_teardown_io_queues()` / `_teardown_ctrl()` | per-ctrl shutdown | `NvmeTcp::teardown_*` |
| `nvme_tcp_error_recovery()` / `_error_recovery_work()` | per-ctrl error path → CONNECTING | `NvmeTcp::error_recovery` |
| `nvme_tcp_reconnect_or_remove()` / `_reconnect_ctrl_work()` | per-ctrl reconnect schedule | `NvmeTcp::reconnect_or_remove` |
| `nvme_reset_ctrl_work()` | per-ctrl reset work | `NvmeTcp::reset_ctrl_work` |
| `nvme_tcp_delete_ctrl()` / `_stop_ctrl()` / `_free_ctrl()` | per-ctrl life | `NvmeTcp::delete_ctrl` / `stop_ctrl` / `free_ctrl` |
| `nvme_tcp_alloc_ctrl()` / `_create_ctrl()` | per-controller alloc + add | `NvmeTcp::alloc_ctrl` / `create_ctrl` |
| `nvme_tcp_key_revoke_needed()` | per-secure-concat key revoke | `NvmeTcp::key_revoke_needed` |
| `nvme_tcp_transport` | `nvmf_transport_ops` entry | `NvmeTcp::TRANSPORT` |
| `nvme_tcp_ctrl_ops` | `nvme_ctrl_ops` vtable | `NvmeTcp::CTRL_OPS` |
| `nvme_tcp_init_module()` / `_cleanup_module()` | per-module life | `NvmeTcp::init` / `cleanup` |
| `nvme_tcp_wq` / `nvme_tcp_ctrl_list` / `nvme_tcp_ctrl_mutex` | per-module globals | `NvmeTcp::WQ` / `CTRL_LIST` |

## Compatibility contract

REQ-1: `struct nvme_tcp_request` fields (verbatim):
- `req: nvme_request` — generic NVMe per-rq header.
- `pdu: *void` — kalloc'd buffer holding the **CmdPDU** at offset 0 and the **DataPDU** at `sizeof(cmd_pdu) - sizeof(data_pdu)` (overlapping back-end).
- `queue: *NvmeTcpQueue`.
- `data_len: u32`, `pdu_len: u32`, `pdu_sent: u32` — current PDU progress.
- `h2cdata_left: u32`, `h2cdata_offset: u32` — R2T-controlled write progress.
- `ttag: u16` — transfer tag echoed from R2T.
- `status: __le16` — per-request error stash (e.g., DDGST mismatch).
- `entry: ListNode`, `lentry: LlistNode` — pending lists.
- `ddgst: __le32` — data digest tail buffer.
- `curr_bio: *bio`, `iter: iov_iter` — current data source/sink.
- `offset: usize`, `data_sent: usize`, `state: NvmeTcpSendState` — TX progress.

REQ-2: `struct nvme_tcp_queue` fields (verbatim):
- `sock: *socket` — kernel TCP socket.
- `io_work: work_struct`, `io_cpu: i32` — bound work.
- `queue_lock: mutex`, `send_mutex: mutex` — TX serialization.
- `req_list: llist_head`, `send_list: list_head` — pending requests (lock-free + locked).
- `pdu: *void`, `pdu_remaining: i32`, `pdu_offset: i32`, `data_remaining: usize`, `ddgst_remaining: usize`, `nr_cqe: u32` — RX state.
- `request: *NvmeTcpRequest` — TX cursor.
- `maxh2cdata: u32` — controller's max H2C payload per PDU (from ICResp).
- `cmnd_capsule_len: usize` — `ioccsz * 16` (IO) or `sizeof(nvme_command) + NVME_TCP_ADMIN_CCSZ` (admin).
- `ctrl: *NvmeTcpCtrl`.
- `flags: AtomicBitmap<NvmeTcpQueueFlag>`.
- `rd_enabled: bool` — recv gate.
- `hdr_digest: bool`, `data_digest: bool`, `tls_enabled: bool`.
- `rcv_crc: u32`, `snd_crc: u32`, `exp_ddgst: __le32`, `recv_ddgst: __le32` — CRC32C state.
- `tls_complete: completion`, `tls_err: i32`.
- `pf_cache: page_frag_cache`.
- `state_change`, `data_ready`, `write_space: fn(*sock)` — saved-original sk callbacks (chain).

REQ-3: `struct nvme_tcp_ctrl` fields (verbatim):
- `queues: *NvmeTcpQueue` (array), `tag_set`, `admin_tag_set`.
- `list: ListNode` (linked into `nvme_tcp_ctrl_list`).
- `addr: sockaddr_storage`, `src_addr: sockaddr_storage`.
- `ctrl: nvme_ctrl`.
- `err_work: work_struct` (error_recovery_work).
- `connect_work: delayed_work` (reconnect_ctrl_work).
- `async_req: NvmeTcpRequest` — per-ctrl AEN SQE buffer.
- `io_queues[HCTX_MAX_TYPES]: u32`.

REQ-4: Send state machine (per-req `state`): `NVME_TCP_SEND_CMD_PDU = 0 → SEND_H2C_PDU → SEND_DATA → SEND_DDGST`. Driven sequentially by `nvme_tcp_try_send`:
- `SEND_CMD_PDU`: `try_send_cmd_pdu` ⟹ `kernel_sendmsg`(cmd_pdu, hdgst). On full send: if has_inline_data ⟹ state = SEND_DATA + snd_crc = NVME_TCP_CRC_SEED; else `done_send_req`.
- `SEND_H2C_PDU`: `try_send_data_pdu` ⟹ kernel_sendmsg(data_pdu header, hdgst). On full: state = SEND_DATA + snd_crc = NVME_TCP_CRC_SEED.
- `SEND_DATA`: `try_send_data` ⟹ loop `sock_sendmsg` with `MSG_SPLICE_PAGES | MSG_MORE`/`MSG_EOR` per `nvme_tcp_pdu_last_send`. If data_digest ⟹ `ddgst_update`. On last+full: if data_digest ⟹ state = SEND_DDGST + req.ddgst = `ddgst_final(snd_crc)`; elif h2cdata_left ⟹ `setup_h2c_data_pdu` (chain another H2C); else `done_send_req`.
- `SEND_DDGST`: `try_send_ddgst` ⟹ `kernel_sendmsg`(&req.ddgst[offset], NVME_TCP_DIGEST_LENGTH-offset). On full: if h2cdata_left ⟹ `setup_h2c_data_pdu`; else `done_send_req`.

REQ-5: Recv state machine (per-queue): `NVME_TCP_RECV_PDU → RECV_DATA → RECV_DDGST`. Derived by `nvme_tcp_recv_state(queue)`:
- pdu_remaining > 0 ⟹ RECV_PDU.
- ddgst_remaining > 0 ⟹ RECV_DDGST.
- else RECV_DATA.

REQ-6: `nvme_tcp_recv_skb(desc, skb, offset, len)`:
- if !queue.rd_enabled ⟹ -EFAULT.
- while len > 0:
  - dispatch on `nvme_tcp_recv_state(queue)`:
    - RECV_PDU ⟹ `nvme_tcp_recv_pdu(queue, skb, &offset, &len)`.
    - RECV_DATA ⟹ `nvme_tcp_recv_data(queue, skb, &offset, &len)`.
    - RECV_DDGST ⟹ `nvme_tcp_recv_ddgst(queue, skb, &offset, &len)`.
  - on result < 0: queue.rd_enabled = false; `nvme_tcp_error_recovery(ctrl)`; return result.
- return consumed (== original len).

REQ-7: `nvme_tcp_recv_pdu`:
- skb_copy_bits(rcv_len = min(*len, pdu_remaining)) into queue.pdu[pdu_offset].
- pdu_remaining -= rcv_len; pdu_offset += rcv_len; *offset += rcv_len; *len -= rcv_len.
- if pdu_remaining > 0 ⟹ return 0 (need more).
- Validate `hdr->hlen == sizeof(nvme_tcp_rsp_pdu)`; if not, allow only the known PDU types (rsp / c2h_data / r2t / c2h_term) via `nvme_tcp_recv_pdu_supported`.
- if hdr->type == `nvme_tcp_c2h_term` ⟹ `handle_c2h_term(queue, pdu)` (logs FES, NO digest validation, returns -EINVAL to trigger recovery).
- if hdr_digest ⟹ `verify_hdgst(queue, pdu, hlen)`; if data_digest ⟹ `check_ddgst(queue, pdu)`.
- Dispatch by `hdr->type`:
  - `nvme_tcp_c2h_data` ⟹ `handle_c2h_data(queue, pdu)` (sets queue.data_remaining; validates SUCCESS-with-not-LAST mismatch).
  - `nvme_tcp_rsp` ⟹ `init_recv_ctx(queue)` + `handle_comp(queue, pdu)` (which calls `complete_async_event` for AEN command_ids, else `process_nvme_cqe`).
  - `nvme_tcp_r2t` ⟹ `init_recv_ctx(queue)` + `handle_r2t(queue, pdu)`.
  - default ⟹ "unsupported pdu".

REQ-8: `nvme_tcp_handle_r2t(queue, pdu)`:
- rq = `nvme_find_rq(tagset, pdu.command_id)`; if !rq ⟹ -ENOENT.
- Validate r2t_length > 0; (data_sent + r2t_length) ≤ data_len; r2t_offset ≥ data_sent; req not already on a list.
- req.pdu_len = 0; req.h2cdata_left = r2t_length; req.h2cdata_offset = r2t_offset; req.ttag = pdu.ttag.
- `setup_h2c_data_pdu(req)`.
- llist_add(req.lentry, queue.req_list).
- queue_work_on(queue.io_cpu, nvme_tcp_wq, queue.io_work).

REQ-9: `nvme_tcp_setup_h2c_data_pdu(req)`:
- data = `req_data_pdu(req)` (= req.pdu + sizeof(cmd_pdu) - sizeof(data_pdu)).
- h2cdata_sent = req.pdu_len. req.state = SEND_H2C_PDU. req.offset = 0.
- req.pdu_len = min(req.h2cdata_left, queue.maxh2cdata).
- req.pdu_sent = 0. req.h2cdata_left -= req.pdu_len. req.h2cdata_offset += h2cdata_sent.
- data.hdr.type = `nvme_tcp_h2c_data`; flags |= F_DATA_LAST if !h2cdata_left; |= F_HDGST if hdr_digest; |= F_DDGST if data_digest.
- data.hdr.hlen = sizeof(data). data.hdr.pdo = hlen + hdgst. data.hdr.plen = hlen + hdgst + pdu_len + ddgst.
- data.ttag = req.ttag; data.command_id = nvme_cid(rq); data.data_offset = h2cdata_offset; data.data_length = pdu_len.

REQ-10: `nvme_tcp_handle_c2h_data(queue, pdu)`:
- rq = `nvme_find_rq(tagset, pdu.command_id)`; if !rq ⟹ -ENOENT.
- if blk_rq_payload_bytes(rq) == 0 ⟹ -EIO.
- queue.data_remaining = le32(pdu.data_length).
- if (flags & F_DATA_SUCCESS) ∧ !(flags & F_DATA_LAST) ⟹ -EPROTO + error_recovery.

REQ-11: `nvme_tcp_recv_data(queue, skb, *offset, *len)`:
- pdu = queue.pdu; rq = `nvme_cid_to_rq(tagset, pdu.command_id)`; req = blk_mq_rq_to_pdu(rq).
- loop while data_remaining > 0 ∧ *len > 0:
  - if iov_iter_count(req.iter) == 0 ⟹ advance to next bio (req.curr_bio = req.curr_bio->bi_next; init_iter(ITER_DEST)); on no-more-bio ⟹ -EIO.
  - if data_digest ⟹ `skb_copy_and_crc32c_datagram_iter(skb, *offset, &req.iter, recv_len, &queue.rcv_crc)`.
  - else ⟹ `skb_copy_datagram_iter(skb, *offset, &req.iter, recv_len)`.
  - update *len, *offset, data_remaining.
- if data_remaining == 0:
  - if data_digest ⟹ queue.exp_ddgst = `ddgst_final(rcv_crc)`; queue.ddgst_remaining = NVME_TCP_DIGEST_LENGTH (transition to RECV_DDGST).
  - else if (flags & F_DATA_SUCCESS) ⟹ `nvme_tcp_end_request(rq, le16(req.status))` (CQE-elided fast path); init_recv_ctx.
  - else init_recv_ctx.

REQ-12: `nvme_tcp_recv_ddgst`:
- skb_copy_bits into ddgst buffer.
- if ddgst_remaining > 0 ⟹ return 0.
- if recv_ddgst ≠ exp_ddgst ⟹ req.status = NVME_SC_DATA_XFER_ERROR.
- if (flags & F_DATA_SUCCESS) ⟹ `nvme_tcp_end_request(rq, req.status)`.
- init_recv_ctx.

REQ-13: `nvme_tcp_init_recv_ctx(queue)`:
- queue.pdu_remaining = sizeof(nvme_tcp_rsp_pdu) + hdgst_len.
- queue.pdu_offset = 0. queue.data_remaining = -1 (cast as size_t). queue.ddgst_remaining = 0.

REQ-14: `nvme_tcp_init_connection(queue)` — ICReq/ICResp handshake (per-queue):
- icreq = kzalloc.
  - icreq.hdr.type = `nvme_tcp_icreq`.
  - icreq.hdr.hlen = sizeof(icreq). icreq.hdr.pdo = 0. icreq.hdr.plen = hlen.
  - icreq.pfv = `NVME_TCP_PFV_1_0`.
  - icreq.maxr2t = 0 (single inflight R2T).
  - icreq.hpda = 0 (no alignment constraint).
  - icreq.digest |= `NVME_TCP_HDR_DIGEST_ENABLE` if hdr_digest; |= `NVME_TCP_DATA_DIGEST_ENABLE` if data_digest.
- `kernel_sendmsg(queue.sock, &msg, &iov(icreq), 1, sizeof(icreq))`.
- icresp = kzalloc.
- If TLS: msg.msg_control = cbuf, msg.msg_controllen = sizeof(cbuf).
- msg.msg_flags = MSG_WAITALL. `kernel_recvmsg(queue.sock, ..., sizeof(icresp), MSG_WAITALL)`.
- if ret < sizeof(icresp) ⟹ -ECONNRESET. if ret < 0 ⟹ propagate.
- If TLS: `tls_get_record_type(sk, cmsghdr)` == `TLS_RECORD_TYPE_DATA` else -ENOTCONN.
- Validate icresp.hdr.type == `nvme_tcp_icresp`; plen == sizeof(icresp); pfv == NVME_TCP_PFV_1_0.
- ctrl_ddgst / ctrl_hdgst must equal queue.data_digest / queue.hdr_digest (mismatch ⟹ -EINVAL).
- icresp.cpda == 0 (no alignment).
- maxh2cdata = le32(icresp.maxdata); must be % 4 ∧ ≥ NVME_TCP_MIN_MAXH2CDATA; queue.maxh2cdata = maxh2cdata.
- kfree(icresp); kfree(icreq).

REQ-15: `nvme_tcp_alloc_queue(nctrl, qid, pskid)`:
- queue = &ctrl.queues[qid]. mutex_init(queue_lock, send_mutex). init_llist_head(req_list). INIT_LIST_HEAD(send_list). INIT_WORK(io_work, nvme_tcp_io_work).
- queue.cmnd_capsule_len = (qid > 0) ? `nctrl.ioccsz * 16` : `sizeof(nvme_command) + NVME_TCP_ADMIN_CCSZ`.
- sock_create_kern(net_ns, ctrl.addr.ss_family, SOCK_STREAM, IPPROTO_TCP, &queue.sock).
- sock_alloc_file(queue.sock, O_CLOEXEC, NULL).
- sk_net_refcnt_upgrade. reclassify_socket (lockdep class fix).
- tcp_sock_set_syncnt(sk, 1). tcp_sock_set_nodelay(sk). sock_no_linger(sk).
- if so_priority > 0 ⟹ sock_set_priority(sk, so_priority).
- if opts.tos ≥ 0 ⟹ ip_sock_set_tos(sk, opts.tos).
- sk.sk_rcvtimeo = 10 * HZ; sk.sk_allocation = GFP_ATOMIC; sk.sk_use_task_frag = false.
- queue.io_cpu = WORK_CPU_UNBOUND. queue.request = NULL. data/ddgst/pdu_remaining = 0.
- sk_set_memalloc(sk).
- if opts.host_traddr set ⟹ kernel_bind(sock, src_addr).
- if opts.host_iface set ⟹ sock_setsockopt(SOL_SOCKET, SO_BINDTODEVICE).
- queue.hdr_digest = opts.hdr_digest; queue.data_digest = opts.data_digest.
- queue.pdu = kmalloc(sizeof(nvme_tcp_rsp_pdu) + hdgst_len).
- kernel_connect(sock, &ctrl.addr).
- if `tls_configured(nctrl)` ∧ pskid ⟹ `nvme_tcp_start_tls(nctrl, queue, pskid)`.
- `nvme_tcp_init_connection(queue)`.
- set_bit(`NVME_TCP_Q_ALLOCATED`, &queue.flags).

REQ-16: `nvme_tcp_start_tls(nctrl, queue, pskid)`:
- Build `tls_handshake_args` (sock, role=client, peerid=pskid, done=`nvme_tcp_tls_done`, data=queue).
- init_completion(queue.tls_complete); queue.tls_err = -EOPNOTSUPP.
- `tls_client_hello_psk(&args, GFP_KERNEL)`.
- wait_for_completion_interruptible_timeout(queue.tls_complete, `tls_handshake_timeout` * HZ).
- on success: queue.tls_enabled = true.

REQ-17: `nvme_tcp_tls_done(data=queue, status, pskid)`:
- queue.tls_err = status. complete(&queue.tls_complete).

REQ-18: `nvme_tcp_start_queue(nctrl, idx)`:
- Set up sock callbacks via `nvme_tcp_setup_sock_ops` (sk_data_ready / sk_state_change / sk_write_space).
- `set_queue_io_cpu(queue)` ⟹ pick least-loaded CPU from mq_map per queue type, atomic_inc nvme_tcp_cpu_queues[io_cpu].
- queue.rd_enabled = true.
- For qid 0: `nvmf_connect_admin_queue`. For qid > 0: `nvmf_connect_io_queue`.
- set_bit(`NVME_TCP_Q_LIVE`, queue.flags).

REQ-19: `nvme_tcp_io_work(work)`:
- container_of(work, NvmeTcpQueue, io_work). deadline = jiffies + msecs_to_jiffies(1).
- Loop:
  - if mutex_trylock(send_mutex): try_send ⟹ pending = (result > 0); abort on error.
  - try_recv ⟹ pending = (result > 0); abort on error.
  - if has_pending ∧ sk_stream_is_writeable ⟹ pending = true.
  - if !pending ∨ !rd_enabled ⟹ return.
- On quota exhaust (time_after deadline): queue_work_on(io_cpu, nvme_tcp_wq, &io_work) — self-requeue.

REQ-20: `nvme_tcp_data_ready(sk)` (sk_data_ready override):
- read_lock_bh(sk.sk_callback_lock). queue = sk.sk_user_data.
- if queue ∧ queue.rd_enabled ∧ !test_bit(Q_POLLING, queue.flags):
  - queue_work_on(queue.io_cpu, nvme_tcp_wq, &queue.io_work).
- trace_sk_data_ready(sk).

REQ-21: `nvme_tcp_write_space(sk)` (sk_write_space override):
- queue = sk.sk_user_data. if sk_stream_is_writeable(sk):
  - clear_bit(SOCK_NOSPACE, sk.sk_socket.flags).
  - if `queue_tls(queue)` ⟹ queue.write_space(sk) (chain original for TLS partial retry).
  - queue_work_on(queue.io_cpu, nvme_tcp_wq, &queue.io_work).

REQ-22: `nvme_tcp_state_change(sk)` (sk_state_change override):
- queue = sk.sk_user_data.
- switch sk.sk_state: TCP_CLOSE / TCP_CLOSE_WAIT / TCP_LAST_ACK / TCP_FIN_WAIT1 / TCP_FIN_WAIT2 ⟹ `nvme_tcp_error_recovery(&queue.ctrl.ctrl)`.
- chain queue.state_change(sk).

REQ-23: `nvme_tcp_queue_tls` / `_tls_configured`:
- `queue_tls(queue)` returns false if !CONFIG_NVME_TCP_TLS, else queue.tls_enabled.
- `tls_configured(ctrl)` returns ctrl.opts.tls ∨ ctrl.opts.concat (gated on CONFIG_NVME_TCP_TLS).

REQ-24: Queue classifier (qid 0 = admin; qid > 0 split into DEFAULT/READ/POLL by `io_queues[HCTX_*]`):
- `admin_queue`: qid == 0.
- `default_queue`: !admin ∧ qid < 1 + io_queues[DEFAULT].
- `read_queue`: !admin ∧ !default ∧ qid < 1 + io_queues[DEFAULT] + io_queues[READ].
- `poll_queue`: !admin ∧ !default ∧ !read ∧ qid < 1 + sum(DEFAULT, READ, POLL).

REQ-25: `nvme_tcp_set_queue_io_cpu(queue)`:
- Choose mq_map by queue class (DEFAULT/READ/POLL).
- Iterate online cpus; pick cpu with smallest `nvme_tcp_cpu_queues[cpu]` whose mq_map[cpu] == (qid - 1).
- atomic_inc(nvme_tcp_cpu_queues[chosen]). set_bit(Q_IO_CPU_SET).
- if wq_unbound ⟹ skip pinning.

REQ-26: `nvme_tcp_setup_cmd_pdu(ns, rq)`:
- req = blk_mq_rq_to_pdu(rq). req.state = SEND_CMD_PDU. req.offset = 0.
- Setup nvme_command via `nvme_setup_cmd(ns, rq)`.
- pdu = req_cmd_pdu(req). memset; pdu.hdr.type = `nvme_tcp_cmd`; flags |= F_HDGST if hdr_digest; |= F_DDGST if data_digest ∧ inline_data.
- pdu.hdr.hlen = sizeof(cmd_pdu). pdu.hdr.pdo = hlen + hdgst.
- pdu.hdr.plen = hlen + hdgst + (inline_data ? data_len + ddgst : 0).
- pdu.cmd = nvme_command.
- `nvme_tcp_map_data(queue, rq, &pdu.cmd)` — sets SG descriptor: `set_sg_inline` if inline; `set_sg_host_data` for R2T-driven write; `set_sg_null` for no data.

REQ-27: `nvme_tcp_queue_rq(hctx, bd)`:
- Validates queue.flags has Q_LIVE (else BLK_STS_RESOURCE).
- Setup cmd PDU. `nvme_tcp_queue_request(req, last)`: llist_add to req_list; if can immediately try_send (mutex_trylock send_mutex) and queue empty ⟹ inline send; else queue_work_on(io_cpu).
- Return BLK_STS_OK / BLK_STS_*.

REQ-28: `nvme_tcp_commit_rqs(hctx)` and async req:
- `commit_rqs`: schedule io_work if pending (kick after batch).
- `alloc_async_req(ctrl)`: pre-allocate per-controller `async_req` SQE buffer (size sizeof(nvme_tcp_cmd_pdu) + hdgst).
- `submit_async_event(ctrl)`: build AEN command, `nvme_tcp_queue_request(async_req, true)`.

REQ-29: `nvme_tcp_timeout(rq) -> blk_eh_timer_return`:
- If queue not LIVE or ctrl in CONNECTING/RESETTING/DELETING ⟹ `complete_timed_out(rq)` (force-complete via nvmf_complete_timed_out_request) + return BLK_EH_DONE.
- Else trigger `nvme_tcp_error_recovery(&ctrl)`; return BLK_EH_DONE.

REQ-30: `nvme_tcp_setup_ctrl(ctrl, new)`:
- `configure_admin_queue(ctrl, new)`.
- if opts.concat ∧ !ctrl.tls_pskid: dev_dbg "restart admin queue for secure concatenation"; nvme_stop_keep_alive; `teardown_admin_queue(ctrl, false)`; `configure_admin_queue(ctrl, false)` (re-run with key now in keyring).
- if ctrl.icdoff ≠ 0 ⟹ -EOPNOTSUPP.
- if !`nvme_ctrl_sgl_supported(ctrl)` ⟹ -EOPNOTSUPP.
- Clamp opts.queue_size to ctrl.sqsize + 1; clamp ctrl.sqsize to ctrl.maxcmd - 1.
- if ctrl.queue_count > 1: `configure_io_queues(ctrl, new)`.
- `nvme_change_ctrl_state(ctrl, NVME_CTRL_LIVE)`.
- `nvme_start_ctrl(ctrl)`.

REQ-31: `nvme_tcp_reconnect_or_remove(ctrl, status)`:
- state = `nvme_ctrl_state(ctrl)`; if state ≠ CONNECTING ⟹ return (WARN if NEW/LIVE).
- if `nvmf_should_reconnect(ctrl, status)`: queue_delayed_work(nvme_wq, &connect_work, opts.reconnect_delay * HZ).
- else: nvme_delete_ctrl(ctrl).

REQ-32: `nvme_tcp_error_recovery_work(work)`:
- if `key_revoke_needed(ctrl)` ⟹ `nvme_auth_revoke_tls_key(ctrl)`.
- nvme_stop_keep_alive. flush_work(ctrl.async_event_work).
- `teardown_io_queues(ctrl, false)` + nvme_unquiesce_io_queues (fail-fast pending).
- `teardown_admin_queue(ctrl, false)` + nvme_unquiesce_admin_queue.
- nvme_auth_stop.
- nvme_change_ctrl_state(ctrl, CONNECTING).
- `reconnect_or_remove(ctrl, 0)`.

REQ-33: `nvme_tcp_create_ctrl(dev, opts)`:
- ctrl = `alloc_ctrl(dev, opts)` — kzalloc; opts → ctrl.ctrl.opts; queue_count = nr_io + nr_write + nr_poll + 1; sqsize = opts.queue_size - 1; kato = opts.kato; INIT_DELAYED_WORK(connect_work, reconnect_ctrl_work); INIT_WORK(err_work, error_recovery_work); reset_work = nvme_reset_ctrl_work; default trsvcid = NVME_TCP_DISC_PORT (4420) if not set; inet_pton_with_scope(addr); validate host_traddr/iface; check existing-controller via `nvmf_ip_options_match`; alloc ctrl.queues; nvme_init_ctrl.
- `nvme_add_ctrl(&ctrl.ctrl)`.
- `nvme_change_ctrl_state(ctrl, CONNECTING)` (else -EINTR).
- `setup_ctrl(ctrl, true)`.
- mutex_lock(nvme_tcp_ctrl_mutex). list_add_tail(ctrl.list, nvme_tcp_ctrl_list). mutex_unlock.

REQ-34: `nvme_tcp_transport` registration:
- name = "tcp"; module = THIS_MODULE.
- required_opts = `NVMF_OPT_TRADDR`.
- allowed_opts = `NVMF_OPT_TRSVCID | NVMF_OPT_RECONNECT_DELAY | NVMF_OPT_HOST_TRADDR | NVMF_OPT_CTRL_LOSS_TMO | NVMF_OPT_HDR_DIGEST | NVMF_OPT_DATA_DIGEST | NVMF_OPT_NR_WRITE_QUEUES | NVMF_OPT_NR_POLL_QUEUES | NVMF_OPT_TOS | NVMF_OPT_HOST_IFACE | NVMF_OPT_TLS | NVMF_OPT_KEYRING | NVMF_OPT_TLS_KEY | NVMF_OPT_CONCAT`.
- create_ctrl = `nvme_tcp_create_ctrl`.

REQ-35: `nvme_tcp_ctrl_ops` (`nvme_ctrl_ops` vtable):
- name = "tcp". flags = NVME_F_FABRICS | NVME_F_BLOCKING.
- reg_read32 / reg_read64 / reg_write32 / subsystem_reset = `nvmf_*` (delegated to fabrics core).
- free_ctrl = `nvme_tcp_free_ctrl`.
- submit_async_event = `nvme_tcp_submit_async_event`.
- delete_ctrl = `nvme_tcp_delete_ctrl`.
- get_address = `nvme_tcp_get_address`.
- stop_ctrl = `nvme_tcp_stop_ctrl`.
- get_virt_boundary = `nvmf_get_virt_boundary`.

REQ-36: Module init (`nvme_tcp_init_module`):
- BUILD_BUG_ON: sizeof(nvme_tcp_hdr) == 8; sizeof(cmd_pdu) == 72; sizeof(data_pdu) == 24; sizeof(rsp_pdu) == 24; sizeof(r2t_pdu) == 24; sizeof(icreq_pdu) == 128; sizeof(icresp_pdu) == 128; sizeof(term_pdu) == 24.
- wq_flags = WQ_MEM_RECLAIM | WQ_HIGHPRI | WQ_SYSFS (+ WQ_UNBOUND if wq_unbound module param).
- nvme_tcp_wq = alloc_workqueue("nvme_tcp_wq", wq_flags, 0).
- For each possible cpu: atomic_set(nvme_tcp_cpu_queues[cpu], 0).
- `nvmf_register_transport(&nvme_tcp_transport)`.

REQ-37: Module cleanup:
- nvmf_unregister_transport. For each ctrl in nvme_tcp_ctrl_list ⟹ nvme_delete_ctrl. flush_workqueue(nvme_delete_wq). destroy_workqueue(nvme_tcp_wq).

## Acceptance Criteria

- [ ] AC-1: `nvme_tcp_create_ctrl` with traddr=`10.0.0.1`, trsvcid="4420", transport="tcp" ⟹ allocates ctrl + 1 admin queue + N io queues, transitions to NVME_CTRL_LIVE.
- [ ] AC-2: `nvme_tcp_init_connection` sends ICReq (type=icreq, pfv=PFV_1_0, maxr2t=0, hpda=0), receives ICResp, sets queue.maxh2cdata.
- [ ] AC-3: ICResp pfv ≠ PFV_1_0 ⟹ -EINVAL.
- [ ] AC-4: ICResp maxdata not multiple of 4 ⟹ -EINVAL.
- [ ] AC-5: hdr_digest mismatch between host opts and ICResp.digest ⟹ -EINVAL.
- [ ] AC-6: nvme_tcp_queue_rq for write with R2T flow: send_cmd_pdu (no inline) → wait R2T → setup_h2c_data_pdu → send_data_pdu → send_data → (optional send_ddgst).
- [ ] AC-7: nvme_tcp_handle_r2t with r2t_offset < req.data_sent ⟹ -EPROTO.
- [ ] AC-8: nvme_tcp_handle_c2h_data: F_DATA_SUCCESS set but not F_DATA_LAST ⟹ -EPROTO + error_recovery.
- [ ] AC-9: nvme_tcp_recv_data with data_digest enabled accumulates queue.rcv_crc via skb_copy_and_crc32c_datagram_iter; on data_remaining == 0 transitions queue to RECV_DDGST.
- [ ] AC-10: recv_ddgst.recv_ddgst ≠ exp_ddgst ⟹ req.status = NVME_SC_DATA_XFER_ERROR before nvme_tcp_end_request.
- [ ] AC-11: nvme_tcp_state_change with TCP_CLOSE ⟹ nvme_tcp_error_recovery.
- [ ] AC-12: nvme_tcp_data_ready when Q_POLLING bit set ⟹ does NOT schedule io_work.
- [ ] AC-13: nvme_tcp_io_work yields after 1ms quota; self-requeues if rd_enabled and still has work.
- [ ] AC-14: nvme_tcp_alloc_queue with opts.tls ∧ pskid ⟹ nvme_tcp_start_tls called before nvme_tcp_init_connection; tls_enabled set on success.
- [ ] AC-15: nvme_tcp_set_queue_io_cpu picks least-loaded online cpu in the corresponding mq_map class (DEFAULT/READ/POLL).
- [ ] AC-16: nvme_tcp_stop_queue clears Q_LIVE, kernel_sock_shutdown(SHUT_RDWR), cancel_work_sync(io_work), restores sock callbacks.
- [ ] AC-17: nvme_tcp_reconnect_or_remove with should_reconnect ⟹ queue_delayed_work(connect_work, opts.reconnect_delay * HZ); else nvme_delete_ctrl.
- [ ] AC-18: BUILD_BUG_ON wire-PDU sizes match the spec (hdr=8, cmd_pdu=72, data_pdu=24, rsp_pdu=24, r2t_pdu=24, icreq=128, icresp=128, term_pdu=24).

## Architecture

```
enum NvmeTcpSendState {
  CmdPdu = 0,
  H2cPdu,
  Data,
  Ddgst,
}

enum NvmeTcpRecvState {
  Pdu = 0,
  Data,
  Ddgst,
}

enum NvmeTcpQueueFlag {
  Allocated = 0,
  Live      = 1,
  Polling   = 2,
  IoCpuSet  = 3,
}

struct NvmeTcpRequest {
  req: NvmeRequest,
  pdu: KBox<u8>,             // sizeof(cmd_pdu) + hdgst
  queue: *NvmeTcpQueue,
  data_len: u32,
  pdu_len: u32, pdu_sent: u32,
  h2cdata_left: u32, h2cdata_offset: u32,
  ttag: u16,
  status: Le16,
  entry: ListNode, lentry: LlistNode,
  ddgst: Le32,
  curr_bio: *Bio,
  iter: IovIter,
  offset: usize, data_sent: usize,
  state: NvmeTcpSendState,
}

struct NvmeTcpQueue {
  sock: *Socket,
  io_work: WorkStruct, io_cpu: i32,
  queue_lock: Mutex, send_mutex: Mutex,
  req_list: LlistHead, send_list: ListHead,
  pdu: KBox<u8>, pdu_remaining: i32, pdu_offset: i32,
  data_remaining: usize, ddgst_remaining: usize, nr_cqe: u32,
  request: Option<*NvmeTcpRequest>,
  maxh2cdata: u32, cmnd_capsule_len: usize,
  ctrl: *NvmeTcpCtrl,
  flags: AtomicBitmap<NvmeTcpQueueFlag>,
  rd_enabled: bool,
  hdr_digest: bool, data_digest: bool, tls_enabled: bool,
  rcv_crc: u32, snd_crc: u32, exp_ddgst: Le32, recv_ddgst: Le32,
  tls_complete: Completion, tls_err: i32,
  pf_cache: PageFragCache,
  state_change: SkCallback, data_ready: SkCallback, write_space: SkCallback,
}

struct NvmeTcpCtrl {
  queues: KVec<NvmeTcpQueue>,
  tag_set: BlkMqTagSet, admin_tag_set: BlkMqTagSet,
  list: ListNode,
  addr: SockaddrStorage, src_addr: SockaddrStorage,
  ctrl: NvmeCtrl,
  err_work: WorkStruct,
  connect_work: DelayedWork,
  async_req: NvmeTcpRequest,
  io_queues: [u32; HCTX_MAX_TYPES],
}
```

`NvmeTcp::try_send(queue) -> KResult<i32>`:
1. if queue.request.is_none() { queue.request = fetch_request(queue)? ; if none ⟹ return Ok(0) }.
2. req = queue.request.as_mut().unwrap().
3. let _g = memalloc_noreclaim_save().
4. dispatch by req.state:
   - CmdPdu ⟹ try_send_cmd_pdu(req)?; if !has_inline_data ⟹ return.
   - H2cPdu ⟹ try_send_data_pdu(req)?.
   - Data ⟹ try_send_data(req)?.
   - Ddgst ⟹ try_send_ddgst(req)?.
5. on -EAGAIN ⟹ return Ok(0). on other err ⟹ fail_request(req) + done_send_req.

`NvmeTcp::recv_skb(desc, skb, offset, len) -> i32`:
1. queue = desc.arg.data.
2. if !queue.rd_enabled ⟹ return -EFAULT.
3. while len > 0:
   - dispatch by recv_state(queue):
     - Pdu ⟹ recv_pdu(queue, skb, &offset, &len).
     - Data ⟹ recv_data(queue, skb, &offset, &len).
     - Ddgst ⟹ recv_ddgst(queue, skb, &offset, &len).
   - on err: queue.rd_enabled = false; error_recovery(ctrl); return err.
4. return consumed.

`NvmeTcp::init_connection(queue) -> KResult<()>`:
1. icreq = ICReqPdu { type: ICReq, hlen: sizeof, pdo: 0, plen: sizeof, pfv: PFV_1_0, maxr2t: 0, hpda: 0, digest: (HDR if hdr_digest) | (DATA if data_digest) }.
2. kernel_sendmsg(queue.sock, &iov(icreq))?.
3. msg = { msg_flags: MSG_WAITALL, msg_control: if tls { cbuf } else { 0 } }.
4. kernel_recvmsg(queue.sock, &iov(icresp), MSG_WAITALL).
5. if tls: tls_get_record_type ≡ DATA else -ENOTCONN.
6. Validate icresp { type == ICResp, plen == sizeof, pfv == PFV_1_0, cpda == 0 }.
7. Validate ctrl_hdgst == queue.hdr_digest; ctrl_ddgst == queue.data_digest.
8. maxh2cdata = icresp.maxdata; require (maxh2cdata % 4 == 0) ∧ (≥ NVME_TCP_MIN_MAXH2CDATA).
9. queue.maxh2cdata = maxh2cdata.

`NvmeTcp::alloc_queue(nctrl, qid, pskid) -> KResult<()>`:
1. Initialize queue locks/work/lists.
2. cmnd_capsule_len = if qid > 0 { nctrl.ioccsz * 16 } else { sizeof(NvmeCommand) + NVME_TCP_ADMIN_CCSZ }.
3. sock_create_kern + sock_alloc_file + sk_net_refcnt_upgrade + reclassify_socket.
4. tcp_sock_set_syncnt(1); tcp_sock_set_nodelay; sock_no_linger; sk_rcvtimeo = 10*HZ; sk_allocation = GFP_ATOMIC; sk_use_task_frag = false; sk_set_memalloc.
5. Bind host_traddr if set; SO_BINDTODEVICE for host_iface.
6. Allocate queue.pdu (sizeof(rsp_pdu) + hdgst).
7. kernel_connect(sock, ctrl.addr).
8. if tls_configured ∧ pskid ⟹ start_tls(nctrl, queue, pskid).
9. init_connection(queue).
10. set Q_ALLOCATED.

`NvmeTcp::error_recovery_work(work)`:
1. ctrl = container_of.
2. if key_revoke_needed(ctrl) ⟹ nvme_auth_revoke_tls_key.
3. nvme_stop_keep_alive; flush async_event_work.
4. teardown_io_queues(ctrl, false); nvme_unquiesce_io_queues.
5. teardown_admin_queue(ctrl, false); nvme_unquiesce_admin_queue.
6. nvme_auth_stop.
7. nvme_change_ctrl_state(ctrl, CONNECTING).
8. reconnect_or_remove(ctrl, 0).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `send_state_monotonic_unless_chain` | INVARIANT | per-try_send: state moves CmdPdu→[H2cPdu]→Data→[Ddgst]; back to H2cPdu only via setup_h2c_data_pdu after R2T. |
| `recv_state_well_formed` | INVARIANT | per-recv_state: pdu_remaining > 0 ⟺ RECV_PDU; ddgst_remaining > 0 ⟺ RECV_DDGST; else RECV_DATA. |
| `pdu_buffer_overlap_safe` | INVARIANT | per-req_data_pdu offset (= sizeof(cmd_pdu) - sizeof(data_pdu)): no aliasing with active cmd_pdu read. |
| `r2t_bounds_checked` | INVARIANT | per-handle_r2t: r2t_offset ≥ data_sent ∧ data_sent + r2t_length ≤ data_len ∧ r2t_length > 0. |
| `ddgst_seed_initialized` | INVARIANT | per-state=Data entry: snd_crc = NVME_TCP_CRC_SEED; recv: rcv_crc = NVME_TCP_CRC_SEED at init_iter. |
| `sock_callback_chain_preserved` | INVARIANT | per-setup_sock_ops/restore_sock_ops: queue saves original sk callbacks, restore on stop. |
| `tls_completion_init_before_use` | INVARIANT | per-start_tls: init_completion before tls_client_hello_psk; wait bounded by tls_handshake_timeout. |
| `io_cpu_atomic_balanced` | INVARIANT | per-set_queue_io_cpu / stop_queue_nowait: atomic_inc/atomic_dec on nvme_tcp_cpu_queues[io_cpu] balanced via Q_IO_CPU_SET bit. |
| `queue_pdu_alloc_balanced` | INVARIANT | per-alloc_queue / free_queue: kmalloc(queue.pdu) ⟹ kfree. |
| `module_get_balanced_on_create_ctrl` | INVARIANT | per-create_ctrl: try_module_get (from fabrics) ⟹ module_put on all paths. |

### Layer 2: TLA+

`drivers/nvme/host/tcp.tla`:
- Per-queue states: Allocated, Connecting (icreq), TlsHandshake, Connected, Live, Stopping, Stopped.
- Per-req states: idle, sent_cmd, awaiting_r2t, sending_h2c, sending_data, sending_ddgst, awaiting_cqe, completed.
- Per-ctrl states: New, Connecting, Live, Resetting, Deleting, Dead.
- Properties:
  - `safety_no_send_in_non_live` — per-queue: queue_rq with !Q_LIVE returns BLK_STS_RESOURCE without sending.
  - `safety_recv_pdu_dispatch_exhaustive` — per-recv_pdu: every type in {c2h_data, rsp, r2t, c2h_term} handled; unknown ⟹ error.
  - `safety_r2t_offset_monotonic` — per-rq sequence: r2t_offset[i+1] ≥ r2t_offset[i] + r2t_length[i].
  - `safety_data_digest_validated` — per-recv_data with data_digest: transition to RECV_DDGST before end_request.
  - `safety_socket_callbacks_restored` — per-stop_queue: data_ready / state_change / write_space restored to saved originals before sock close.
  - `safety_tls_handshake_first` — per-alloc_queue: start_tls (if applicable) happens before init_connection.
  - `safety_reconnect_bounded` — per-error_recovery: reconnect attempts bounded by max_reconnects (via should_reconnect).
  - `liveness_io_work_eventually_drains` — per-io_work: if pending+writeable, work re-runs (self-requeue on quota exhaust).
  - `liveness_recv_eventually_completes_cqe` — per-rq with valid CQE: nvme_tcp_end_request called.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `NvmeTcp::init_connection` post: queue.maxh2cdata set, (maxh2cdata % 4 == 0) ∧ (≥ NVME_TCP_MIN_MAXH2CDATA) | `NvmeTcp::init_connection` |
| `NvmeTcp::recv_pdu` post: hdgst_verified ⟹ no silent corruption of cmd_id dispatch | `NvmeTcp::recv_pdu` |
| `NvmeTcp::handle_r2t` post: req.h2cdata_left == r2t_length, req.h2cdata_offset == r2t_offset, req.ttag == pdu.ttag | `NvmeTcp::handle_r2t` |
| `NvmeTcp::setup_h2c_data_pdu` post: plen == hlen + hdgst + pdu_len + ddgst; F_DATA_LAST iff !h2cdata_left | `NvmeTcp::setup_h2c_data_pdu` |
| `NvmeTcp::try_send_cmd_pdu` post: success ⟹ done OR state=Data (if inline_data) | `NvmeTcp::try_send_cmd_pdu` |
| `NvmeTcp::try_send_data` post: state=Ddgst (if data_digest) ∨ state=H2cPdu (if h2cdata_left) ∨ done | `NvmeTcp::try_send_data` |
| `NvmeTcp::recv_data` post: data_remaining == 0 ⟹ either init_recv_ctx ∨ ddgst_remaining = DIGEST_LENGTH | `NvmeTcp::recv_data` |
| `NvmeTcp::state_change` post: TCP fin states ⟹ error_recovery scheduled | `NvmeTcp::state_change` |
| `NvmeTcp::create_ctrl` post: ctrl in CTRL_LIVE ∨ proper teardown | `NvmeTcp::create_ctrl` |

### Layer 4: Verus/Creusot functional

`Per-host write to /dev/nvme-fabrics → nvmf_create_ctrl → nvme_tcp_create_ctrl → alloc_admin_queue (sock_create + connect + (start_tls?) + ICReq/ICResp) → nvmf_connect_admin_queue → alloc_io_queues × N → CTRL_LIVE → blk_mq dispatch → setup_cmd_pdu → try_send_cmd_pdu → (R2T from controller) → setup_h2c_data_pdu → try_send_data → (data_digest? try_send_ddgst) → recv_pdu(rsp) → end_request` semantic equivalence: per-NVMe-over-Fabrics spec rev 1.1a §3.4 (NVMe-TCP) + RFC 9293 (TCP) framing.

`Per-recv {c2h_data ∨ rsp ∨ r2t ∨ c2h_term} → recv_pdu dispatch → handle_*` semantic equivalence: per-NVMe-TCP PDU encoding (`include/linux/nvme-tcp.h`).

`Per-error recovery: TCP fin / digest mismatch / timeout / CQE error → nvme_tcp_error_recovery_work → teardown → reconnect_or_remove (per nvmf_should_reconnect)` semantic equivalence: per-`Documentation/nvme/feature-and-quirk-policy.rst` and `drivers/nvme/host/fabrics.c::nvmf_should_reconnect`.

## Hardening

(Inherits row-1 features from `drivers/nvme/00-overview.md` § Hardening.)

NVMe-TCP host reinforcement:

- **Per-BUILD_BUG_ON wire-PDU sizes** — defense against per-struct-padding ABI drift (hdr=8, cmd=72, data=24, rsp=24, r2t=24, icreq=128, icresp=128, term=24).
- **Per-NVME_TCP_PFV_1_0 strict at ICResp** — defense against per-future-version silent misinterpretation.
- **Per-maxh2cdata % 4 == 0 ∧ ≥ NVME_TCP_MIN_MAXH2CDATA** — defense against per-malicious controller forcing illegal sizes.
- **Per-icresp.cpda == 0 strict** — defense against per-alignment-attack on data placement.
- **Per-hdr_digest/data_digest mismatch ⟹ -EINVAL at ICResp** — defense against per-mid-negotiation tamper.
- **Per-recv_pdu hdgst CRC32C verify before dispatch** — defense against per-bit-flip header injection.
- **Per-recv_data ddgst CRC32C verify before completion** — defense against per-bit-flip payload injection.
- **Per-r2t_offset / r2t_length / data_sent strict bounds** — defense against per-malicious R2T (buffer overflow via overlapping H2C).
- **Per-r2t-while-on-list rejection (`-EPROTO`)** — defense against per-double-R2T race.
- **Per-c2h_data SUCCESS-only-with-LAST flag check** — defense against per-protocol-violation early-completion.
- **Per-tls_get_record_type == DATA strict in init_connection** — defense against per-non-DATA TLS record fed into NVMe parser.
- **Per-sk_callback chain (save original, restore on stop)** — defense against per-leaked-callback after sock close.
- **Per-rd_enabled gate on data_ready** — defense against per-recv-after-stop race.
- **Per-Q_POLLING bypasses io_work** — defense against per-double-recv race between polling and softirq.
- **Per-send_mutex serializes try_send** — defense against per-concurrent-send corruption of state machine.
- **Per-nvme_tcp_cpu_queues atomic_inc/dec balanced via Q_IO_CPU_SET** — defense against per-cpu-counter drift.
- **Per-deadline (1 ms) in io_work with self-requeue** — defense against per-monopolize-cpu.
- **Per-sock_no_linger + SHUT_RDWR at stop** — defense against per-stale-data-after-reconnect.
- **Per-NVME_F_BLOCKING flag honored** — defense against per-non-blocking-context-violation in TLS / sock ops.
- **Per-concat (secure concatenation) revoke_tls_key on error-recovery** — defense against per-stale-derived-key reuse after error.
- **Per-handle_c2h_term logged FES and triggers recovery** — defense against per-silent-fatal-transport-error.
- **Per-existing-controller `nvmf_ip_options_match` check** — defense against per-duplicate-controller (unless `duplicate_connect`).

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — nvme_tcp PDU header slab (nvme_tcp_hdr / nvme_tcp_cmd_pdu / nvme_tcp_r2t_pdu) and per-request data_iter pages whitelisted; defense against per-oversized-PDU-leaking-adjacent-slab.
- **PAX_KERNEXEC** — nvme_tcp_io_work, nvme_tcp_recv_skb, nvme_tcp_try_send, kTLS sw-encrypt fast-path all W^X.
- **PAX_RANDKSTACK** — per-recv-state-machine entry (nvme_tcp_recv_pdu / nvme_tcp_recv_data / nvme_tcp_recv_ddgst) randomizes stack offset; defense against per-fabric-side-channel via stack-prefetch.
- **PAX_REFCOUNT** — nvme_tcp_queue, nvme_tcp_ctrl, sock refs saturating refcount_t; defense against per-reconnect-storm refcount overflow.
- **PAX_MEMORY_SANITIZE** — nvme_tcp_pdu pool, per-request data buffers, and TLS session keys poison-on-free; defense against per-prior-PDU leak in re-used slab.
- **PAX_UDEREF** — TCP path is kernel-internal (no user copy on the data path); kTLS ULP control via setsockopt enforces split user/kernel on key material.
- **PAX_RAP/kCFI** — nvme_tcp transport_ops (create_ctrl, free_ctrl, submit_async_event, get_address) CFI-protected; defense against per-forged-ops via crafted module-load race.
- **GRKERNSEC_HIDESYM** — nvme_tcp_* helpers and per-queue sock pointers hidden from kallsyms to unprivileged readers.
- **GRKERNSEC_DMESG** — "queue %d: tls handshake failed", "header digest error", "data digest error" prints restricted; defense against per-fabric-topology-disclosure.
- **kTLS hardened** — nvme-tcp tls=1.3 only (NVMe-TCP TLS PSK per TP-8019); kTLS ULP runs in kernel, AEAD ciphersuites only (AES-128-GCM / AES-256-GCM / CHACHA20-POLY1305); no NULL / RC4 / 3DES.
- **ALPN strict** — TLS handshake requires ALPN id "nvme" (NVMe-TCP TLS profile); defense against per-cross-protocol-attack (e.g. HTTPS proxy fronting an NVMe target).
- **Fabrics reconnect rate-limit** — nvme_tcp_reconnect_ctrl_work backs off (NVMF_DEF_RECONNECT_DELAY .. NVMF_DEF_CTRL_LOSS_TMO), max_reconnects honored; defense against per-flapping-target wedging CPU on reconnect work.
- **HDR/DATA digest CRC32C verified** — per-PDU header and data digests checked before queueing to block layer; defense against per-on-wire-tamper of NVMe command/data when TLS not negotiated.
- **Per-c2h_term log + recovery** — Controller-to-Host TerminationReq logs FES and triggers ctrl reset; defense against silent fatal transport error.
- **Per-handshake CAP_NET_ADMIN on /dev/nvme-fabrics** — TLS PSK lookup performed in kernel keyring under the namespace of the caller; defense against per-unprivileged-PSK-reuse.
- Rationale: nvme-tcp carries block-layer data over an untrusted IP fabric; grsec posture combines kTLS-only AEAD with strict ALPN, CRC32C digest verification, reconnect rate-limit, CFI on transport_ops, sanitize-on-free of PDU/key slabs, and usercopy-whitelisting on PDU headers.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `drivers/nvme/host/fabrics.c` common fabrics core (covered in `host-fabrics.md` Tier-3).
- `drivers/nvme/host/core.c` controller lifecycle / blk-mq submission (covered in `host-core.md` Tier-3).
- `drivers/nvme/host/auth.c` DH-HMAC-CHAP negotiation (covered separately if expanded).
- `net/tls/*` kTLS implementation (covered separately if expanded).
- `tls_handshake` user-space `tlshd` daemon protocol (covered separately if expanded).
- `drivers/nvme/target/tcp.c` NVMe-TCP target side (covered separately if expanded).
- `drivers/nvme/host/rdma.c` RDMA transport (covered separately if expanded).
- Implementation code.
