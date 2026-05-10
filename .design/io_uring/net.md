# Tier-3: io_uring/net.c — io_uring network operations

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/net.c (~1872 lines)
  - io_uring/net.h
  - io_uring/notif.{c,h} (zerocopy notifications)
  - io_uring/zcrx.{c,h} (zero-copy receive)
  - include/uapi/linux/io_uring.h (IORING_OP_SEND/RECV/SENDMSG/RECVMSG/CONNECT/ACCEPT/SHUTDOWN/SOCKET/BIND/LISTEN/SEND_ZC/SENDMSG_ZC/RECV_ZC, IORING_RECVSEND_*, IORING_SEND_*, IORING_ACCEPT_*)
-->

## Summary

`io_uring/net.c` implements the network-socket family of io_uring opcodes: per-IORING_OP_SEND / _RECV / _SENDMSG / _RECVMSG (stream + datagram payload exchange), per-IORING_OP_SEND_ZC / _SENDMSG_ZC (zero-copy egress with notification CQE), per-IORING_OP_RECV_ZC (page-pool zero-copy ingress via `io_zcrx_ifq`), per-IORING_OP_CONNECT / _ACCEPT / _BIND / _LISTEN / _SOCKET / _SHUTDOWN (socket lifecycle). Per-op flag-set: `IORING_RECVSEND_POLL_FIRST` (skip first issue, arm poll), `IORING_RECVSEND_FIXED_BUF` (use registered-buffer-table fixed buffer for I/O), `IORING_RECVSEND_BUNDLE` (consume multiple provided-buffer entries in one send/recv), `IORING_RECV_MULTISHOT` (one SQE → many CQEs), `IORING_SEND_VECTORIZED` (registered iovec), `IORING_ACCEPT_MULTISHOT`, `IORING_ACCEPT_DONTWAIT`, `IORING_ACCEPT_POLL_FIRST`. Multishot fairness via `MULTISHOT_MAX_RETRY = 32` retry budget. Per-bundle send: drives `io_buffers_select` to map a vector of provided-buffer entries into a single `sock_sendmsg` call; bundle-recv: `io_buffers_peek` allocates from buffer-ring then drains until socket queue empty. Per-zero-copy notification: each `IORING_OP_SEND_ZC` allocates a notification kiocb via `io_alloc_notif`, posts `IORING_CQE_F_NOTIF` once the kernel-network-stack releases the pinned user pages. Critical for: liburing async-net workloads, accept-storm scaling, low-overhead send/recv, RDMA-bypass-style zero-copy egress.

This Tier-3 covers `io_uring/net.c` (~1872 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_sr_msg` | per-send/recv state | `IoSrMsg` |
| `struct io_accept` | per-accept state | `IoAccept` |
| `struct io_socket` | per-socket-op state | `IoSocket` |
| `struct io_connect` | per-connect state | `IoConnect` |
| `struct io_bind` / `io_listen` / `io_shutdown` | per-op state | `IoBind` / `IoListen` / `IoShutdown` |
| `struct io_recvzc` | per-zerocopy-recv state | `IoRecvzc` |
| `struct io_async_msghdr` | per-async msghdr+iov state | `IoAsyncMsghdr` |
| `io_send_setup()` / `io_sendmsg_setup()` | per-prep import | `Net::send_setup` / `Net::sendmsg_setup` |
| `io_sendmsg_prep()` / `io_recvmsg_prep()` | per-SQE parse | `Net::sendmsg_prep` / `Net::recvmsg_prep` |
| `io_send()` / `io_sendmsg()` | per-issue (send / sendmsg) | `Net::send` / `Net::sendmsg` |
| `io_recv()` / `io_recvmsg()` | per-issue (recv / recvmsg) | `Net::recv` / `Net::recvmsg` |
| `io_recv_finish()` / `io_send_finish()` | per-multishot/bundle finalize | `Net::recv_finish` / `Net::send_finish` |
| `io_recvmsg_multishot()` | per-mshot recvmsg | `Net::recvmsg_multishot` |
| `io_send_zc_prep()` / `io_sendmsg_zc()` | per-zerocopy-send prep+issue | `Net::send_zc_prep` / `Net::sendmsg_zc` |
| `io_send_zc_import()` | per-fixed-buf zc import | `Net::send_zc_import` |
| `io_sg_from_iter()` / `io_sg_from_iter_iovec()` | per-skb-frag fill (zc / non-zc) | `Net::sg_from_iter` / `Net::sg_from_iter_iovec` |
| `io_recvzc_prep()` / `io_recvzc()` | per-zcrx ingress | `Net::recvzc_prep` / `Net::recvzc` |
| `io_accept_prep()` / `io_accept()` | per-accept | `Net::accept_prep` / `Net::accept` |
| `io_socket_prep()` / `io_socket()` | per-socket-create | `Net::socket_prep` / `Net::socket` |
| `io_connect_prep()` / `io_connect()` | per-connect | `Net::connect_prep` / `Net::connect` |
| `io_bind_prep()` / `io_bind()` | per-bind | `Net::bind_prep` / `Net::bind` |
| `io_listen_prep()` / `io_listen()` | per-listen | `Net::listen_prep` / `Net::listen` |
| `io_shutdown_prep()` / `io_shutdown()` | per-shutdown | `Net::shutdown_prep` / `Net::shutdown` |
| `io_net_retry()` | per-MSG_WAITALL retry policy | `Net::net_retry` |
| `io_msg_alloc_async()` / `io_netmsg_recycle()` | per-msghdr alloc/free cache | `Net::msg_alloc_async` / `Net::netmsg_recycle` |
| `io_msg_copy_hdr()` / `io_compat_msg_copy_hdr()` | per-msghdr copy-in | `Net::msg_copy_hdr` / `Net::compat_msg_copy_hdr` |
| `io_recvmsg_mshot_prep()` / `io_recvmsg_prep_multishot()` | per-mshot header layout | `Net::recvmsg_mshot_prep` / `Net::recvmsg_prep_multishot` |
| `io_bundle_nbufs()` | per-bundle consumed-segment count | `Net::bundle_nbufs` |
| `io_send_select_buffer()` / `io_recv_buf_select()` | per-bundle buffer pick | `Net::send_select_buffer` / `Net::recv_buf_select` |
| `io_net_kbuf_recyle()` | per-partial-send kbuf recycle | `Net::net_kbuf_recycle` |
| `io_netmsg_cache_free()` | per-msghdr cache slab free | `Net::netmsg_cache_free` |
| `io_sendrecv_fail()` | per-fail CQE flag carry | `Net::sendrecv_fail` |
| `io_send_zc_cleanup()` / `io_sendmsg_recvmsg_cleanup()` | per-cleanup | `Net::send_zc_cleanup` / `Net::sendmsg_recvmsg_cleanup` |
| `io_socket_bpf_populate()` | per-BPF op-ctx fill | `Net::socket_bpf_populate` |
| `IORING_RECVSEND_POLL_FIRST` | per-flag: poll-then-issue | shared |
| `IORING_RECVSEND_FIXED_BUF` | per-flag: registered-buffer I/O | shared |
| `IORING_RECVSEND_BUNDLE` | per-flag: multi-buffer batch | shared |
| `IORING_RECV_MULTISHOT` | per-flag: one SQE → many CQE | shared |
| `IORING_SEND_VECTORIZED` | per-flag: send accepts iovec array | shared |
| `IORING_SEND_ZC_REPORT_USAGE` | per-flag: report copied vs zero-copied | shared |
| `IORING_ACCEPT_MULTISHOT` / `_DONTWAIT` / `_POLL_FIRST` | per-flag accept variants | shared |
| `IORING_CQE_F_MORE` | per-CQE: more CQEs to come | shared |
| `IORING_CQE_F_NOTIF` | per-CQE: zero-copy notification | shared |
| `IORING_CQE_F_SOCK_NONEMPTY` | per-CQE: socket queue still has data | shared |
| `IORING_CQE_F_BUF_MORE` | per-CQE: more buffer-ring entries follow | shared |
| `MULTISHOT_MAX_RETRY` (=32) | per-fairness bound | const |

## Compatibility contract

REQ-1: struct io_sr_msg (per-send/recv state):
- file: per-socket file*.
- umsg / umsg_compat / buf: per-user msghdr-or-buf pointer (union).
- len: per-buffer byte count.
- done_io: per-partial-transfer carry.
- msg_flags: per-msghdr MSG_* (MSG_NOSIGNAL forced on send).
- flags: u16 — lower 8 = UAPI IORING_RECVSEND_*; upper 8 = internal (`IORING_RECV_RETRY`, `_PARTIAL_MAP`, `_MSHOT_CAP`, `_MSHOT_LIM`, `_MSHOT_DONE`).
- nr_multishot_loops: per-mshot retry counter (vs `MULTISHOT_MAX_RETRY`).
- buf_group: per-provided-buffer group id (when `REQ_F_BUFFER_SELECT`).
- mshot_len: per-mshot per-iter limit.
- mshot_total_len: per-mshot lifetime byte limit (sqe->optlen).
- msg_control: per-cmsg user pointer (saved across sendmsg).
- notif: per-zerocopy notification kiocb (NULL outside zc).

REQ-2: io_sendmsg_prep(req, sqe):
- sr.done_io = 0.
- sr.len = sqe.len; if sr.len < 0: return -EINVAL.
- sr.flags = sqe.ioprio.
- if sr.flags & ~SENDMSG_FLAGS (=POLL_FIRST|BUNDLE|VECTORIZED): return -EINVAL.
- sr.msg_flags = sqe.msg_flags | MSG_NOSIGNAL.
- if sr.msg_flags & MSG_DONTWAIT: req.flags |= REQ_F_NOWAIT.
- if REQ_F_BUFFER_SELECT: sr.buf_group = req.buf_index.
- if sr.flags & BUNDLE:
  - if opcode == SENDMSG: return -EINVAL (bundle only for SEND).
  - sr.msg_flags |= MSG_WAITALL.
  - req.flags |= REQ_F_MULTISHOT.
- if io_is_compat: sr.msg_flags |= MSG_CMSG_COMPAT.
- alloc async msghdr via io_msg_alloc_async; return -ENOMEM on fail.
- if opcode != SENDMSG: return io_send_setup(req, sqe).
- if sqe.addr2 || sqe.file_index: return -EINVAL.
- return io_sendmsg_setup(req, sqe).

REQ-3: io_send_setup(req, sqe):
- sr.buf = sqe.addr (user ptr).
- if sqe.__pad3[0]: return -EINVAL.
- clear kmsg.msg.{msg_name,msg_namelen,msg_control,msg_controllen,msg_ubuf}.
- if sqe.addr2 (dest sockaddr): move_addr_to_kernel; kmsg.msg.msg_name = &kmsg.addr.
- if sr.flags & FIXED_BUF:
  - if !VECTORIZED: req.flags |= REQ_F_IMPORT_BUFFER (deferred import); return 0.
  - else: io_prep_reg_iovec(req, &kmsg.vec, sr.buf, sr.len).
- if REQ_F_BUFFER_SELECT: return 0 (buffer picked at issue).
- if sr.flags & VECTORIZED: io_net_import_vec(... ITER_SOURCE).
- else: import_ubuf(ITER_SOURCE, sr.buf, sr.len, ...).

REQ-4: io_sendmsg_setup(req, sqe):
- sr.flags |= VECTORIZED.
- sr.umsg = sqe.addr (user msghdr ptr).
- io_msg_copy_hdr(req, kmsg, &msg, ITER_SOURCE, NULL).
- sr.msg_control = kmsg.msg.msg_control_user (save — __sys_sendmsg overwrites).
- if FIXED_BUF: io_prep_reg_iovec(req, &kmsg.vec, msg.msg_iov, msg.msg_iovlen).
- if REQ_F_BUFFER_SELECT: return 0.
- io_net_import_vec(req, kmsg, msg.msg_iov, msg.msg_iovlen, ITER_SOURCE).

REQ-5: io_sendmsg(req, issue_flags):
- sock = sock_from_file(req.file); if !sock: -ENOTSOCK.
- if !REQ_F_POLLED ∧ POLL_FIRST: return -EAGAIN (arm async-poll).
- flags = sr.msg_flags; if NONBLOCK: flags |= MSG_DONTWAIT.
- if MSG_WAITALL: min_ret = iov_iter_count.
- kmsg.msg.msg_control_user = sr.msg_control.
- ret = __sys_sendmsg_sock(sock, &kmsg.msg, flags).
- if ret < min_ret:
  - if -EAGAIN ∧ NONBLOCK: return -EAGAIN.
  - if ret > 0 ∧ io_net_retry (MSG_WAITALL ∧ STREAM-or-SEQPACKET):
    - clear msg_control; sr.done_io += ret; return -EAGAIN.
  - if -ERESTARTSYS: ret = -EINTR.
  - req_set_fail(req).
- io_req_msg_cleanup (recycle msghdr cache).
- if ret >= 0: ret += sr.done_io.
- else if sr.done_io: ret = sr.done_io.
- io_req_set_res(req, ret, 0); return IOU_COMPLETE.

REQ-6: io_send(req, issue_flags) (no-msghdr path, supports BUNDLE):
- sock_from_file; -ENOTSOCK on fail.
- POLL_FIRST gate: same as sendmsg.
- retry_bundle: sel.buf_list = NULL.
- if io_do_buffer_select: io_send_select_buffer(req, issue_flags, &sel, kmsg) (pick provided-buffer / bundle vec).
- if MSG_WAITALL ∨ BUNDLE: min_ret = iov_iter_count.
- flags &= ~MSG_INTERNAL_SENDMSG_FLAGS.
- ret = sock_sendmsg(sock, &kmsg.msg).
- if ret < min_ret:
  - -EAGAIN+NONBLOCK: return -EAGAIN.
  - ret > 0 ∧ io_net_retry: sr.len -= ret; sr.buf += ret; sr.done_io += ret; io_net_kbuf_recyle.
  - -ERESTARTSYS: -EINTR.
  - req_set_fail.
- sel.val = aggregated ret; if !io_send_finish(req, kmsg, &sel): goto retry_bundle.
- io_req_msg_cleanup; return sel.val.

REQ-7: io_send_finish(req, kmsg, sel) → bool:
- !BUNDLE: cflags = io_put_kbuf(req, sel.val, sel.buf_list); goto finish.
- cflags = io_put_kbufs(req, sel.val, sel.buf_list, io_bundle_nbufs(kmsg, sel.val)).
- if bundle_finished (sel.val ≤ 0) ∨ REQ_F_BL_EMPTY ∨ REQ_F_POLLED: goto finish.
- if io_req_post_cqe(req, sel.val, cflags | IORING_CQE_F_MORE):
  - io_mshot_prep_retry(req, kmsg); return false (continue bundle).
- finish: io_req_set_res(req, sel.val, cflags); sel.val = IOU_COMPLETE; return true.

REQ-8: io_recvmsg_prep(req, sqe):
- sr.done_io = 0.
- if sqe.addr2: -EINVAL.
- sr.umsg = sqe.addr; sr.len = sqe.len.
- sr.flags = sqe.ioprio; if ~RECVMSG_FLAGS (=POLL_FIRST|MULTISHOT|BUNDLE): -EINVAL.
- sr.msg_flags = sqe.msg_flags.
- if MSG_DONTWAIT: REQ_F_NOWAIT.
- if MSG_ERRQUEUE: REQ_F_CLEAR_POLLIN.
- if REQ_F_BUFFER_SELECT: sr.buf_group = req.buf_index.
- mshot_total_len = mshot_len = 0.
- if MULTISHOT:
  - require REQ_F_BUFFER_SELECT (else -EINVAL).
  - reject MSG_WAITALL.
  - if opcode == RECV: mshot_len = sr.len; mshot_total_len = sqe.optlen; if optlen: flags |= MSHOT_LIM.
  - else (RECVMSG): reject sqe.optlen.
  - req.flags |= REQ_F_APOLL_MULTISHOT.
- else if sqe.optlen: -EINVAL.
- if BUNDLE: reject opcode == RECVMSG.
- if compat: msg_flags |= MSG_CMSG_COMPAT.
- sr.nr_multishot_loops = 0.
- io_recvmsg_prep_setup(req).

REQ-9: io_recvmsg(req, issue_flags):
- sock_from_file; -ENOTSOCK.
- POLL_FIRST gate.
- retry_multishot: sel.buf_list = NULL.
- if io_do_buffer_select: io_buffer_select(req, &len, sr.buf_group, issue_flags); if !sel.addr: -ENOBUFS.
- if REQ_F_APOLL_MULTISHOT: io_recvmsg_prep_multishot to layout `struct io_uring_recvmsg_out` + sockaddr + cmsg in user buf.
- kmsg.msg.msg_inq = -1 (unknown).
- if APOLL_MULTISHOT: io_recvmsg_multishot(sock, sr, kmsg, flags, &mshot_finished).
- else: if MSG_WAITALL ∧ !msg_controllen: min_ret = iov_iter_count; ret = __sys_recvmsg_sock(sock, &kmsg.msg, sr.umsg, kmsg.uaddr, flags).
- if ret < min_ret:
  - -EAGAIN ∧ force_nonblock: io_kbuf_recycle; return IOU_RETRY.
  - ret > 0 ∧ io_net_retry: sr.done_io += ret; io_net_kbuf_recyle.
  - -ERESTARTSYS → -EINTR.
  - req_set_fail.
- else if MSG_WAITALL ∧ (MSG_TRUNC|MSG_CTRUNC): req_set_fail.
- if ret > 0: ret += sr.done_io; else if sr.done_io: ret = sr.done_io; else: io_kbuf_recycle.
- sel.val = ret; if !io_recv_finish: goto retry_multishot.
- return sel.val.

REQ-10: io_recv_finish(req, kmsg, sel, mshot_finished, issue_flags) → bool:
- cflags = 0.
- if msg.msg_inq > 0: cflags |= IORING_CQE_F_SOCK_NONEMPTY.
- if sel.val > 0 ∧ MSHOT_LIM: mshot_total_len -= min(sel.val, mshot_total_len); if 0: flags |= MSHOT_DONE; mshot_finished = true.
- if BUNDLE: cflags |= io_put_kbufs(...); if REQ_F_BL_EMPTY: finish.
  - if !NO_RETRY ∧ msg_inq > 1 ∧ this_ret > 0 ∧ !iov_iter_count: flags |= RETRY; return false.
- else: cflags |= io_put_kbuf(req, sel.val, sel.buf_list).
- if REQ_F_APOLL_MULTISHOT ∧ !mshot_finished ∧ io_req_post_cqe(req, sel.val, cflags | F_MORE):
  - sel.val = IOU_RETRY; io_mshot_prep_retry.
  - if F_SOCK_NONEMPTY ∨ msg_inq < 0:
    - if nr_multishot_loops++ < MULTISHOT_MAX_RETRY ∧ !MSHOT_CAP: return false.
    - else: nr_multishot_loops = 0; clear MSHOT_CAP; if F_MULTISHOT: sel.val = IOU_REQUEUE.
  - return true.
- finish: io_req_set_res(req, sel.val, cflags); sel.val = IOU_COMPLETE; cleanup; return true.

REQ-11: io_recvmsg_multishot(sock, io, kmsg, flags, &finished):
- pre-layout: `struct io_uring_recvmsg_out` header followed by sockaddr_storage (BUILD_BUG_ON offset gap), followed by cmsg, followed by payload.
- if O_NONBLOCK: flags |= MSG_DONTWAIT.
- err = sock_recvmsg; finished = (err ≤ 0); if err < 0: return err.
- hdr.controllen = original_controllen - remaining_controllen.
- hdr.flags = msg_flags & ~MSG_CMSG_COMPAT.
- hdr.payloadlen = err; cap err to payloadlen.
- hdr.namelen = kmsg.msg.msg_namelen (pre-truncation, per POSIX-1003.1g).
- copy_to_user(io.buf, &hdr, copy_len); on fail: finished = true; return -EFAULT.
- return sizeof(out) + namelen + controllen + err (advanced CQE.res).

REQ-12: io_send_zc_prep(req, sqe) (IORING_OP_SEND_ZC / _SENDMSG_ZC):
- zc.done_io = 0.
- reject sqe.__pad2[0].
- reject REQ_F_CQE_SKIP (zc needs both data + notif CQE).
- notif = io_alloc_notif(ctx); on fail: -ENOMEM.
- notif.cqe.user_data = sqe.addr3 ?: req.cqe.user_data.
- notif.cqe.res = 0; notif.cqe.flags = IORING_CQE_F_NOTIF.
- req.flags |= REQ_F_NEED_CLEANUP | REQ_F_POLL_NO_LAZY.
- zc.flags = sqe.ioprio; allowed = IO_ZC_FLAGS_VALID = COMMON | SEND_ZC_REPORT_USAGE | SEND_VECTORIZED; reject else.
- if SEND_ZC_REPORT_USAGE: nd.zc_report = true; nd.zc_used = false; nd.zc_copied = false.
- zc.len = sqe.len; zc.msg_flags = sqe.msg_flags | MSG_NOSIGNAL | MSG_ZEROCOPY.
- req.buf_index = sqe.buf_index.
- if MSG_DONTWAIT: REQ_F_NOWAIT.
- io_msg_alloc_async.
- if opcode == SEND_ZC: io_send_setup; else: io_sendmsg_setup (reject sqe.addr2 / file_index).
- if !FIXED_BUF: iomsg.msg.sg_from_iter = io_sg_from_iter_iovec; io_notif_account_mem(zc.notif, count).
- else: iomsg.msg.sg_from_iter = io_sg_from_iter.

REQ-13: io_sendmsg_zc(req, issue_flags):
- sock_from_file; -ENOTSOCK.
- if !SOCK_SUPPORT_ZC: -EOPNOTSUPP.
- POLL_FIRST gate.
- if REQ_F_IMPORT_BUFFER: io_send_zc_import (import via reg buf or reg vec).
- msg_flags = sr.msg_flags; if NONBLOCK: |= MSG_DONTWAIT.
- if MSG_WAITALL: min_ret = iov_iter_count.
- kmsg.msg.msg_ubuf = &io_notif_to_data(sr.notif).uarg (associates notif with skb).
- if opcode == SEND_ZC: msg_flags &= ~MSG_INTERNAL_SENDMSG_FLAGS; sock_sendmsg.
- else: kmsg.msg.msg_control_user = sr.msg_control; __sys_sendmsg_sock.
- error/retry semantics mirror io_sendmsg.
- if !UNLOCKED: io_notif_flush(sr.notif); sr.notif = NULL; io_req_msg_cleanup.
- io_req_set_res(req, ret, IORING_CQE_F_MORE).
- return IOU_COMPLETE.

REQ-14: io_sg_from_iter / io_sg_from_iter_iovec (skb-frag fill):
- io_sg_from_iter (managed): set SKBFL_MANAGED_FRAG_REFS on empty skb; iterate bvec_iter; fill `MAX_SKB_FRAGS` slots via `__skb_fill_page_desc_noacc`; update skb.{data_len,len,truesize}; -EMSGSIZE on overflow.
- io_sg_from_iter_iovec: downgrade-managed; delegate to zerocopy_fill_skb_from_iter.

REQ-15: io_recvzc_prep / io_recvzc (IORING_OP_RECV_ZC):
- prep: reject sqe.addr2/addr/addr3; ifq_idx = sqe.zcrx_ifq_idx; zc.ifq = xa_load(&ctx.zcrx_ctxs, ifq_idx); if !ifq: -EINVAL.
- zc.len = sqe.len; zc.flags = sqe.ioprio; reject sqe.msg_flags.
- allowed flags = POLL_FIRST | MULTISHOT; require MULTISHOT (else -EINVAL).
- req.flags |= REQ_F_APOLL_MULTISHOT (all data CQEs are aux).
- issue: POLL_FIRST gate; sock_from_file.
- io_zcrx_recv(req, ifq, sock, 0, issue_flags, &zc.len).
- if len && zc.len == 0: io_req_set_res(req, 0, 0); IOU_COMPLETE (no more bytes).
- if ret ≤ 0 ∧ ret != -EAGAIN:
  - -ERESTARTSYS → -EINTR; IOU_REQUEUE pass-through.
  - req_set_fail; IOU_COMPLETE.
- IOU_RETRY (mshot continuation).

REQ-16: io_accept_prep / io_accept:
- prep: reject sqe.len / sqe.buf_index.
- accept.addr / addr_len from sqe.addr / addr2.
- accept.flags = sqe.accept_flags.
- accept.iou_flags = sqe.ioprio; reject ~ACCEPT_FLAGS (=MULTISHOT|DONTWAIT|POLL_FIRST).
- accept.file_slot = sqe.file_index; if set:
  - reject SOCK_CLOEXEC.
  - if MULTISHOT: require file_slot == IORING_FILE_INDEX_ALLOC.
- accept.flags must be subset of SOCK_CLOEXEC|SOCK_NONBLOCK.
- if SOCK_NONBLOCK aliased to O_NONBLOCK: translate.
- if MULTISHOT: REQ_F_APOLL_MULTISHOT.
- if DONTWAIT: REQ_F_NOWAIT.
- issue (retry: label):
  - if !fixed: fd = __get_unused_fd_flags.
  - arg.err = 0; arg.is_empty = -1.
  - file = do_accept(req.file, &arg, accept.addr, accept.addr_len, accept.flags).
  - on error: -EAGAIN+nonblock+!DONTWAIT: IOU_RETRY; -ERESTARTSYS→-EINTR.
  - else fixed: io_fixed_fd_install; non-fixed: fd_install; ret = fd.
  - cflags = 0; if !arg.is_empty: |= IORING_CQE_F_SOCK_NONEMPTY.
  - if ret ≥ 0 ∧ APOLL_MULTISHOT ∧ io_req_post_cqe(req, ret, cflags|F_MORE):
    - if F_SOCK_NONEMPTY ∨ arg.is_empty == -1: goto retry.
    - return IOU_RETRY.
  - io_req_set_res(req, ret, cflags); if ret < 0: req_set_fail; IOU_COMPLETE.

REQ-17: io_socket_prep / io_socket / io_socket_bpf_populate:
- prep: reject sqe.addr / rw_flags / buf_index; sock.domain = sqe.fd; type = sqe.off; protocol = sqe.len.
- file_slot = sqe.file_index; nofile = RLIMIT_NOFILE.
- sock.flags = type & ~SOCK_TYPE_MASK; reject (file_slot ∧ SOCK_CLOEXEC); restrict flags to SOCK_CLOEXEC|SOCK_NONBLOCK.
- issue: __sys_socket_file; fd_install (or io_fixed_fd_install).
- bpf_populate: family/type/protocol exposed to BPF op-ctx.

REQ-18: io_connect_prep / io_connect:
- prep: reject sqe.len/buf_index/rw_flags/splice_fd_in.
- conn.addr = sqe.addr; addr_len = sqe.addr2.
- in_progress = seen_econnaborted = false.
- io_msg_alloc_async; move_addr_to_kernel(conn.addr, addr_len, &io.addr).
- issue: if in_progress: vfs_poll(EPOLLERR) → if err: goto get_sock_err.
- ret = __sys_connect_file(req.file, &io.addr, conn.addr_len, file_flags).
- if -EAGAIN/-EINPROGRESS/-ECONNABORTED ∧ force_nonblock:
  - -EINPROGRESS: in_progress = true.
  - -ECONNABORTED: if seen_econnaborted: goto out; else: seen_econnaborted = true.
  - return -EAGAIN.
- if in_progress ∧ (-EBADFD ∨ -EISCONN): get_sock_err: ret = sock_error(sock_from_file.sk).
- -ERESTARTSYS → -EINTR.
- out: if ret < 0: req_set_fail; io_req_msg_cleanup; io_req_set_res; IOU_COMPLETE.

REQ-19: io_bind_prep / io_bind / io_listen_prep / io_listen / io_shutdown_prep / io_shutdown:
- bind: reject sqe.len/buf_index/rw_flags/splice_fd_in; move_addr_to_kernel; __sys_bind_socket(sock, &io.addr, bind.addr_len).
- listen: reject sqe.addr/buf_index/rw_flags/splice_fd_in/addr2; listen.backlog = sqe.len; __sys_listen_socket(sock, backlog).
- shutdown: prep rejects sqe.off/addr/rw_flags/buf_index/splice_fd_in; shutdown.how = sqe.len; req.flags |= REQ_F_FORCE_ASYNC.
- issue: WARN if NONBLOCK (shutdown is force-async); __sys_shutdown_sock(sock, how).

REQ-20: io_net_retry(sock, flags):
- return false if !(flags & MSG_WAITALL).
- return true iff sock.type == SOCK_STREAM ∨ SOCK_SEQPACKET.

REQ-21: io_msg_alloc_async / io_netmsg_recycle (per-async msghdr cache):
- alloc: io_uring_alloc_async_data(&ctx.netmsg_cache, req); if hdr.vec.iovec: REQ_F_NEED_CLEANUP.
- recycle: if IO_URING_F_UNLOCKED: io_netmsg_iovec_free(hdr); return.
- io_alloc_cache_vec_kasan(&hdr.vec); if hdr.vec.nr > IO_VEC_CACHE_SOFT_CAP: io_vec_free.
- if io_alloc_cache_put(&ctx.netmsg_cache, hdr): io_req_async_data_clear(req, REQ_F_NEED_CLEANUP).

REQ-22: Internal sr_msg flag set (upper bits of sr.flags):
- IORING_RECV_RETRY (1<<15) — bundle retry pending.
- IORING_RECV_PARTIAL_MAP (1<<14) — partial buffer-ring map.
- IORING_RECV_MSHOT_CAP (1<<13) — mshot retry budget exhausted.
- IORING_RECV_MSHOT_LIM (1<<12) — mshot has total-byte limit.
- IORING_RECV_MSHOT_DONE (1<<11) — mshot finished cleanly.
- IORING_RECV_RETRY_CLEAR = RETRY|PARTIAL_MAP.
- IORING_RECV_NO_RETRY = RETRY|PARTIAL_MAP|MSHOT_CAP|MSHOT_DONE.

REQ-23: io_sendrecv_fail(req): on per-fail-completion,
- if sr.done_io: req.cqe.res = sr.done_io.
- if REQ_F_NEED_CLEANUP ∧ (SEND_ZC ∨ SENDMSG_ZC): req.cqe.flags |= IORING_CQE_F_MORE (notif still pending).

## Acceptance Criteria

- [ ] AC-1: IORING_OP_SEND with byte buffer transmits len bytes (or short-send + io_net_retry); CQE.res = bytes sent.
- [ ] AC-2: IORING_OP_RECV with REQ_F_BUFFER_SELECT picks a buffer from buf_group, fills it, and reports BID via IORING_CQE_F_BUFFER.
- [ ] AC-3: IORING_OP_SENDMSG copies user msghdr + iovec, transmits via __sys_sendmsg_sock; cmsg passed through.
- [ ] AC-4: IORING_OP_RECVMSG with IORING_RECV_MULTISHOT delivers many CQEs (each with IORING_CQE_F_MORE) until socket queue drained or buffer-ring empty.
- [ ] AC-5: IORING_OP_RECVMSG multishot layouts `io_uring_recvmsg_out` + sockaddr + cmsg + payload in the selected user buffer.
- [ ] AC-6: IORING_RECVSEND_POLL_FIRST: first issue returns -EAGAIN immediately, arming async-poll; second issue runs the op.
- [ ] AC-7: IORING_RECVSEND_FIXED_BUF: SEND-ZC against registered-buffer index N transmits with MSG_ZEROCOPY; notif CQE posted on page-pin-release.
- [ ] AC-8: IORING_RECVSEND_BUNDLE on SEND: consumes multiple provided-buffer entries in a single sock_sendmsg.
- [ ] AC-9: IORING_OP_ACCEPT with IORING_ACCEPT_MULTISHOT: each new client posts a CQE (with F_MORE) and re-arms.
- [ ] AC-10: IORING_OP_SEND_ZC posts two CQEs: data CQE (F_MORE) then notif CQE (F_NOTIF) with the same user_data (or addr3 override).
- [ ] AC-11: IORING_OP_SENDMSG_ZC with SEND_ZC_REPORT_USAGE flags zc_copied vs zc_used in the notif.
- [ ] AC-12: MULTISHOT_MAX_RETRY (=32): exceeded → mshot requeued onto io-wq (IOU_REQUEUE) to enforce fairness.
- [ ] AC-13: IORING_OP_CONNECT with -EINPROGRESS on non-blocking socket: in_progress=true, EAGAIN; later EPOLLERR resolved via sock_error.
- [ ] AC-14: IORING_OP_SHUTDOWN: how=SHUT_RDWR closes both directions; CQE.res = 0.
- [ ] AC-15: IORING_OP_SOCKET with file_slot: socket installed into fixed-file table; non-fixed: into fd table.

## Architecture

```
struct IoSrMsg {
  file: *File,
  // Union:
  umsg: *user UserMsghdr,           // SENDMSG / RECVMSG path
  umsg_compat: *user CompatMsghdr,
  buf: *user u8,                    // SEND / RECV path
  len: i32,
  done_io: u32,                     // partial-transfer carry
  msg_flags: u32,                   // MSG_*
  nr_multishot_loops: u32,          // vs MULTISHOT_MAX_RETRY
  flags: u16,                       // UAPI lower-8 + internal upper-8
  buf_group: u16,                   // provided-buffer group id
  mshot_len: u32,                   // per-iter mshot cap
  mshot_total_len: u32,             // lifetime mshot cap (sqe.optlen)
  msg_control: *user u8,            // saved cmsg ptr
  notif: Option<*IoKiocb>,          // zc notification kiocb
}

const MULTISHOT_MAX_RETRY: u32 = 32;

const SENDMSG_FLAGS: u16 = IORING_RECVSEND_POLL_FIRST
                         | IORING_RECVSEND_BUNDLE
                         | IORING_SEND_VECTORIZED;
const RECVMSG_FLAGS: u16 = IORING_RECVSEND_POLL_FIRST
                         | IORING_RECV_MULTISHOT
                         | IORING_RECVSEND_BUNDLE;
const ACCEPT_FLAGS: u32  = IORING_ACCEPT_MULTISHOT
                         | IORING_ACCEPT_DONTWAIT
                         | IORING_ACCEPT_POLL_FIRST;

const IO_ZC_FLAGS_COMMON: u32 = IORING_RECVSEND_POLL_FIRST
                              | IORING_RECVSEND_FIXED_BUF;
const IO_ZC_FLAGS_VALID:  u32 = IO_ZC_FLAGS_COMMON
                              | IORING_SEND_ZC_REPORT_USAGE
                              | IORING_SEND_VECTORIZED;
```

`Net::sendmsg_prep(req, sqe) -> Result<()>`:
1. sr.done_io = 0; sr.len = sqe.len.
2. if sr.len < 0: return -EINVAL.
3. sr.flags = sqe.ioprio; check vs SENDMSG_FLAGS.
4. sr.msg_flags = sqe.msg_flags | MSG_NOSIGNAL.
5. apply REQ_F_NOWAIT / REQ_F_BUFFER_SELECT / BUNDLE→MULTISHOT.
6. io_msg_alloc_async ?: -ENOMEM.
7. dispatch: SEND → send_setup; SENDMSG → sendmsg_setup.

`Net::send(req, issue_flags) -> IoComplete`:
1. sock = sock_from_file(req.file)?.
2. poll-first gate.
3. retry_bundle: select buffer if needed.
4. sock_sendmsg(sock, &kmsg.msg).
5. handle short-send via io_net_retry / io_net_kbuf_recyle.
6. send_finish: if !done && BUNDLE, loop.
7. msg_cleanup → IOU_COMPLETE.

`Net::recvmsg(req, issue_flags) -> IoComplete | IoRetry`:
1. sock_from_file?.
2. poll-first gate.
3. retry_multishot: buffer_select if needed.
4. APOLL_MULTISHOT? → recvmsg_multishot (layouts header + sockaddr + cmsg + payload in selected buffer).
5. else: __sys_recvmsg_sock.
6. recv_finish: post CQE (F_MORE), enforce MULTISHOT_MAX_RETRY fairness.

`Net::send_zc_prep(req, sqe) -> Result<()>`:
1. allocate notif kiocb (io_alloc_notif); -ENOMEM on fail.
2. set notif.cqe.flags = IORING_CQE_F_NOTIF; req → REQ_F_NEED_CLEANUP | REQ_F_POLL_NO_LAZY.
3. validate zc.flags vs IO_ZC_FLAGS_VALID.
4. zc.msg_flags |= MSG_NOSIGNAL | MSG_ZEROCOPY.
5. delegate setup (send_setup or sendmsg_setup).
6. if !FIXED_BUF: sg_from_iter = sg_from_iter_iovec + io_notif_account_mem.
7. else: sg_from_iter = io_sg_from_iter (managed).

`Net::sendmsg_zc(req, issue_flags) -> IoComplete`:
1. sock_from_file; require SOCK_SUPPORT_ZC.
2. poll-first gate; if REQ_F_IMPORT_BUFFER: send_zc_import.
3. kmsg.msg.msg_ubuf = &notif.uarg (associate notif with skbs).
4. sock_sendmsg or __sys_sendmsg_sock.
5. on completion: if !UNLOCKED: io_notif_flush(notif); cleanup.
6. io_req_set_res(req, ret, IORING_CQE_F_MORE) (notif CQE follows).

`Net::accept(req, issue_flags) -> IoComplete | IoRetry`:
1. poll-first gate.
2. retry: get_unused_fd (or fixed slot).
3. do_accept(req.file, &arg, addr, addr_len, flags).
4. cflags |= F_SOCK_NONEMPTY if !arg.is_empty.
5. APOLL_MULTISHOT + post_cqe(F_MORE): retry if F_SOCK_NONEMPTY or empty-unknown.

`Net::recvzc(req, issue_flags) -> IoComplete | IoRetry`:
1. poll-first gate.
2. io_zcrx_recv(req, ifq, sock, 0, issue_flags, &zc.len) — drives page-pool zero-copy ingress.
3. len consumed exhausted → IOU_COMPLETE.
4. errors (non-EAGAIN): IOU_COMPLETE; otherwise IOU_RETRY.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sendmsg_poll_first_no_first_issue` | INVARIANT | per-send/recv: POLL_FIRST ∧ !POLLED ⟹ first issue returns -EAGAIN. |
| `multishot_retry_budget_bounded` | INVARIANT | per-recv: nr_multishot_loops ≤ MULTISHOT_MAX_RETRY before requeue. |
| `zc_notif_paired_with_data` | INVARIANT | per-SEND_ZC: every data CQE has F_MORE; one matching F_NOTIF CQE eventually posted. |
| `recvmsg_multishot_layout_no_overlap` | INVARIANT | per-recvmsg_prep_multishot: hdr|sockaddr|cmsg|payload do not overlap; total ≤ buffer len. |
| `bundle_only_send_not_sendmsg` | INVARIANT | per-prep: BUNDLE ∧ SENDMSG → -EINVAL. |
| `mshot_requires_buffer_select` | INVARIANT | per-prep: MULTISHOT requires REQ_F_BUFFER_SELECT. |
| `accept_multishot_requires_alloc_slot` | INVARIANT | per-accept_prep: MULTISHOT ∧ file_slot != IORING_FILE_INDEX_ALLOC → -EINVAL. |
| `fixed_buf_uses_registered_table` | INVARIANT | per-zc-import: FIXED_BUF resolves req.buf_index in ctx.buf_table. |
| `sock_support_zc_required` | INVARIANT | per-sendmsg_zc: SOCK_SUPPORT_ZC unset → -EOPNOTSUPP. |
| `connect_in_progress_one_shot` | INVARIANT | per-connect: -ECONNABORTED retried at most once (seen_econnaborted). |

### Layer 2: TLA+

`models/io_uring/net-multishot.tla`:
- Per-multishot recv: states {Idle, Poll, Issue, PostCQE, Retry, Requeue, Done}.
- Per-bundle send: states {Idle, Select, Issue, PartialDone, Finish}.
- Per-zc send: states {Idle, Issue, DataCQE, NotifPending, NotifPosted}.
- Properties:
  - `safety_no_orphan_notif` — every SEND_ZC eventually posts F_NOTIF CQE.
  - `safety_mshot_retry_bound` — multishot retry never exceeds MULTISHOT_MAX_RETRY without requeue.
  - `safety_poll_first_one_eagain` — POLL_FIRST emits exactly one -EAGAIN then runs.
  - `liveness_recv_terminates` — per-recv eventually completes or requeues.
  - `safety_data_then_notif_order` — F_NOTIF CQE never observed before its data CQE.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Net::sendmsg_prep` post: flags ⊆ SENDMSG_FLAGS ∨ -EINVAL | `Net::sendmsg_prep` |
| `Net::recvmsg_prep` post: MULTISHOT ⟹ REQ_F_BUFFER_SELECT | `Net::recvmsg_prep` |
| `Net::accept_prep` post: MULTISHOT ⟹ file_slot ∈ {0, IORING_FILE_INDEX_ALLOC} | `Net::accept_prep` |
| `Net::send_zc_prep` post: !REQ_F_CQE_SKIP ∧ notif != NULL | `Net::send_zc_prep` |
| `Net::sendmsg_zc` post: notif flushed before req cleanup ∨ deferred-cleanup path | `Net::sendmsg_zc` |
| `Net::recvmsg_multishot` post: copy_to_user(hdr) ⟹ finished | `Net::recvmsg_multishot` |
| `Net::recv_finish` post: F_MORE ⟹ post_cqe succeeded; else mshot terminates | `Net::recv_finish` |
| `Net::recvzc` post: ret > 0 ⟹ ifq valid ∧ zc-mshot active | `Net::recvzc` |

### Layer 4: Verus/Creusot functional

`Per-IORING_OP_RECV multishot → io_buffer_select → __sys_recvmsg_sock → io_req_post_cqe(F_MORE) → fairness-bounded requeue` semantic equivalence; `Per-IORING_OP_SEND_ZC → MSG_ZEROCOPY → skb_zerocopy_fill → io_notif_flush → F_NOTIF CQE` semantic equivalence; per Documentation/networking/msg_zerocopy.rst and the liburing zero-copy test suite.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

io_uring/net reinforcement:

- **Per-MSG_NOSIGNAL forced on send** — defense against per-SIGPIPE-on-broken-pipe surprising async submitter.
- **Per-IORING_RECV_MULTISHOT requires REQ_F_BUFFER_SELECT** — defense against per-mshot-without-buffer infinite loop.
- **Per-MULTISHOT_MAX_RETRY (=32) fairness bound** — defense against per-flooding-client starving other ops on the io-wq.
- **Per-IORING_RECVSEND_POLL_FIRST gate** — defense against per-syscall-thrash on non-ready sockets.
- **Per-IORING_RECVSEND_FIXED_BUF iovec-bounds check via reg buffer table** — defense against per-OOB-fixed-buf-access.
- **Per-SOCK_SUPPORT_ZC required for SEND_ZC** — defense against per-zerocopy-on-unsupported-proto silent data corruption.
- **Per-REQ_F_CQE_SKIP rejected on SEND_ZC** — defense against per-orphan-notif (notif must pair with data CQE).
- **Per-IORING_ACCEPT_MULTISHOT alloc-slot only** — defense against per-fixed-slot collision in mshot accept.
- **Per-IO_ZC_FLAGS_VALID strict mask** — defense against per-unknown-flag silent acceptance.
- **Per-io_uring fd reject in fixed-file register (io_is_uring_fops)** — defense against per-recursive-uring-registration.
- **Per-msghdr / iovec cache slab discipline** — defense against per-stale-iovec re-use across requests (io_netmsg_recycle).
- **Per-MSG_WAITALL + STREAM/SEQPACKET net_retry** — defense against per-spurious-short-send abort.
- **Per-bundle finish drops F_BUFFERS_COMMIT-pending state on retry** — defense against per-double-commit of provided buffer.
- **Per-zc-notif accounted memory via io_notif_account_mem** — defense against per-unaccounted-locked-pages.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- io_uring/notif.c notification kiocb lifecycle (covered separately in `notif.md` Tier-3 / folded into `net-ops`).
- io_uring/zcrx.c page-pool / interface-queue (`io_zcrx_ifq`) setup (covered separately in `zcrx.md` Tier-3 / folded into `net-ops`).
- io_uring/cmd_net.c URING_CMD network passthrough (covered in `uring-cmd.md` Tier-3).
- io_uring/napi.c NAPI-busy-poll integration (covered separately).
- net/socket.c socket-layer entry points (covered in `net/` Tier-3 docs).
- net/ipv4 / net/ipv6 protocol families (covered in `net/` Tier-3 docs).
- Implementation code
