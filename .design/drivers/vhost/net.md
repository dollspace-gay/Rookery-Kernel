# Tier-3: drivers/vhost/net.c — vhost-net: kernel-side virtio-net data path

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: drivers/vhost/00-overview.md
upstream-paths:
  - drivers/vhost/net.c (~1894 lines)
  - drivers/vhost/vhost.h
  - include/uapi/linux/vhost.h
  - include/uapi/linux/virtio_net.h
  - include/linux/if_tun.h
  - include/linux/if_tap.h
-->

## Summary

**vhost-net** is the kernel-side server for the virtio-net device model. A KVM/QEMU guest exposes a virtio-net device with two virtqueues (RX = index 0, TX = index 1); the userspace VMM offloads the data-path to vhost-net by ioctl'ing `VHOST_NET_SET_BACKEND` on `/dev/vhost-net`, passing a tap/tun/raw-AF_PACKET socket fd as the network backend. The kernel then services the virtqueue indices from a per-device vhost worker (kthread), bypassing userspace on every packet. Per-VQ workers: `handle_tx` drains the guest's TX ring into the backend socket via `sock->ops->sendmsg`; `handle_rx` pulls incoming sk_buffs out of the tap's ptr_ring via `sock->ops->recvmsg` and writes them into guest RX-ring descriptor chains. Per-feature: `VHOST_NET_F_VIRTIO_NET_HDR` toggles whether vhost or the socket layer supplies the `virtio_net_hdr` per-packet; `VIRTIO_NET_F_MRG_RXBUF` enables multi-buffer RX (multiple descriptor heads per packet, with `num_buffers` patched into the header); `VIRTIO_F_IN_ORDER` collapses the used-ring to a single id+len cell per batch; `VIRTIO_F_ACCESS_PLATFORM` enables IOTLB. Per-batch: TX builds `xdp_buff` arrays (up to `VHOST_NET_BATCH = 64`) and submits via `TUN_MSG_PTR` control to `tun_sendmsg`, bypassing the sk_buff path for the GoodCopy region. Per-zerocopy (`experimental_zcopytx`): TX packets ≥ `VHOST_GOODCOPY_LEN = 256` may be sent with a `ubuf_info` so the lower driver DMAs directly from guest memory; completion fires `vhost_zerocopy_complete` which marks the used-ring entry. Per-busyloop: configurable spin on the paired VQ before going back to eventfd notification. Critical for: KVM guest networking throughput (line-rate on 10/25/40 GbE), zero-copy TX, virtio packed/split ring support.

This Tier-3 covers `drivers/vhost/net.c` (~1894 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vhost_net` | per-fd device | `VhostNet` |
| `struct vhost_net_virtqueue` | per-VQ (RX/TX) | `VhostNetVirtqueue` |
| `struct vhost_net_ubuf_ref` | per-VQ zerocopy refcount | `VhostNetUbufRef` |
| `struct vhost_net_buf` | per-VQ RX ptr_ring batch | `VhostNetBuf` |
| `vhost_net_open()` | per-/dev/vhost-net open | `VhostNet::open` |
| `vhost_net_release()` | per-fd close | `VhostNet::release` |
| `vhost_net_ioctl()` | per-ioctl dispatch | `VhostNet::ioctl` |
| `vhost_net_set_backend()` | per-VHOST_NET_SET_BACKEND | `VhostNet::set_backend` |
| `vhost_net_set_features()` | per-VHOST_SET_FEATURES | `VhostNet::set_features` |
| `vhost_net_set_owner()` | per-VHOST_SET_OWNER | `VhostNet::set_owner` |
| `vhost_net_reset_owner()` | per-VHOST_RESET_OWNER | `VhostNet::reset_owner` |
| `handle_tx()` | per-TX worker | `VhostNet::handle_tx` |
| `handle_tx_copy()` | per-TX copy/XDP path | `VhostNet::handle_tx_copy` |
| `handle_tx_zerocopy()` | per-TX zerocopy path | `VhostNet::handle_tx_zerocopy` |
| `handle_rx()` | per-RX worker | `VhostNet::handle_rx` |
| `handle_tx_kick()` / `handle_rx_kick()` | per-vhost_work entry | `VhostNet::tx_kick` / `rx_kick` |
| `handle_tx_net()` / `handle_rx_net()` | per-EPOLL{OUT,IN} backend wake | `VhostNet::tx_net` / `rx_net` |
| `get_tx_bufs()` | per-TX desc fetch | `VhostNet::get_tx_bufs` |
| `get_rx_bufs()` | per-RX desc-chain fetch | `VhostNet::get_rx_bufs` |
| `vhost_net_build_xdp()` | per-TX xdp_buff build | `VhostNet::build_xdp` |
| `vhost_tx_batch()` | per-TUN_MSG_PTR submit | `VhostNet::tx_batch` |
| `vhost_net_signal_used()` | per-batch used-ring signal | `VhostNet::signal_used` |
| `vhost_zerocopy_signal_used()` | per-zcopy used-ring | `VhostNet::zcopy_signal_used` |
| `vhost_zerocopy_complete()` | per-ubuf_info completion | `VhostNet::zcopy_complete` |
| `vhost_net_busy_poll()` | per-spin paired VQ | `VhostNet::busy_poll` |
| `peek_head_len()` / `vhost_net_rx_peek_head_len()` | per-RX peek | `VhostNet::peek_head_len` |
| `vhost_net_buf_produce()` | per-RX ptr_ring batch consume | `VhostNet::buf_produce` |
| `vhost_net_buf_consume()` | per-RX dequeue | `VhostNet::buf_consume` |
| `vhost_net_enable_vq()` / `_disable_vq()` | per-VQ poll start/stop | `VhostNet::enable_vq` / `disable_vq` |
| `vhost_net_ubuf_alloc()` / `_put()` / `_put_and_wait()` | per-zcopy ref lifecycle | `VhostNet::ubuf_alloc` / `put` / `put_and_wait` |
| `get_socket()` / `get_raw_socket()` / `get_tap_socket()` | per-backend fd→socket | `VhostNet::get_socket` |
| `vhost_net_chr_read_iter` / `_write_iter` / `_poll` | per-IOTLB msg I/O | `VhostNet::chr_read_iter` / `chr_write_iter` / `chr_poll` |
| `vhost_ubuf_ops` | per-ubuf_info callback table | shared |

## Compatibility contract

REQ-1: `struct vhost_net_virtqueue`:
- vq: embedded `struct vhost_virtqueue` (the generic per-VQ object).
- vhost_hlen: per-vhost-supplied `virtio_net_hdr` length (set when `VHOST_NET_F_VIRTIO_NET_HDR` negotiated).
- sock_hlen: per-socket-supplied vnet_hdr length (mutually exclusive with vhost_hlen).
- upend_idx: per-TX zerocopy producer index into `vq->heads[]` (DMA-in-progress slot).
- done_idx: per-TX zerocopy consumer index (DMA-done slot); also reused as per-RX batched-head count.
- batched_xdp: per-TX count of xdp_buffs queued in `nvq->xdp[]`, ≤ `VHOST_NET_BATCH`.
- ubuf_info: per-TX `ubuf_info_msgzc[UIO_MAXIOV]` (allocated on zerocopy enable).
- ubufs: per-TX `vhost_net_ubuf_ref` (refcount of outstanding ubufs).
- rx_ring: per-RX `ptr_ring*` borrowed from tap/tun for `peek_len` / batch consume.
- rxq: per-RX `vhost_net_buf` (head/tail/queue of `VHOST_NET_BATCH = 64` ptrs).
- xdp: per-TX `xdp_buff[VHOST_NET_BATCH]` slab.

REQ-2: `struct vhost_net`:
- dev: embedded `struct vhost_dev`.
- vqs[VHOST_NET_VQ_MAX]: `[RX=0, TX=1]` array of `vhost_net_virtqueue`.
- poll[VHOST_NET_VQ_MAX]: per-backend-socket `vhost_poll` (drives wake on EPOLLIN/OUT).
- tx_packets: per-tx-zcopy heuristic counter (mod 64).
- tx_zcopy_err: per-tx-zcopy error counter; ratio gates `vhost_net_tx_select_zcopy`.
- tx_flush: per-flush-in-progress flag (suppresses new DMAs).
- pf_cache: per-device `page_frag_cache` for XDP buffer allocation.

REQ-3: `vhost_net_open(inode, file)`:
- /* Per-fd allocation */
- n = kvmalloc(sizeof(*n), GFP_KERNEL | __GFP_RETRY_MAYFAIL).
- vqs = kmalloc_array(VHOST_NET_VQ_MAX, sizeof(*vqs), GFP_KERNEL).
- queue = kmalloc_array(VHOST_NET_BATCH, sizeof(void *), GFP_KERNEL).
- n.vqs[RX].rxq.queue = queue.
- xdp = kmalloc_array(VHOST_NET_BATCH, sizeof(*xdp), GFP_KERNEL).
- n.vqs[TX].xdp = xdp.
- /* Per-VQ init */
- n.vqs[TX].vq.handle_kick = handle_tx_kick.
- n.vqs[RX].vq.handle_kick = handle_rx_kick.
- for each i: ubufs=NULL; ubuf_info=NULL; upend_idx=0; done_idx=0; batched_xdp=0; vhost_hlen=0; sock_hlen=0; rx_ring=NULL; vhost_net_buf_init(&rxq).
- /* Per-vhost-dev init: total_iov = UIO_MAXIOV + VHOST_NET_BATCH; weight = VHOST_NET_WEIGHT (0x80000 bytes); pkt_weight = VHOST_NET_PKT_WEIGHT (256) */
- vhost_dev_init(&dev, vqs, VHOST_NET_VQ_MAX, UIO_MAXIOV + VHOST_NET_BATCH, VHOST_NET_PKT_WEIGHT, VHOST_NET_WEIGHT, true, NULL).
- /* Per-poll: tap socket → vhost_poll → schedule worker on EPOLL{OUT,IN} */
- vhost_poll_init(&poll[TX], handle_tx_net, EPOLLOUT, &dev, vqs[TX]).
- vhost_poll_init(&poll[RX], handle_rx_net, EPOLLIN, &dev, vqs[RX]).
- file.private_data = n.
- page_frag_cache_init(&n.pf_cache).
- return 0.

REQ-4: `vhost_net_release(inode, file)`:
- vhost_net_stop(n, &tx_sock, &rx_sock).
- vhost_net_flush(n).
- vhost_dev_stop(&n.dev).
- vhost_dev_cleanup(&n.dev).
- vhost_net_vq_reset(n).
- sockfd_put(tx_sock); sockfd_put(rx_sock) if set.
- synchronize_rcu (drain ubuf completion path).
- vhost_net_flush(n) (re-queued work).
- kfree(rxq.queue); kfree(xdp); kfree(dev.vqs).
- page_frag_cache_drain(&n.pf_cache).
- kvfree(n).

REQ-5: `vhost_net_set_backend(n, index, fd)`:
- mutex_lock(&n.dev.mutex).
- vhost_dev_check_owner(&n.dev) — must be owning task.
- if index ≥ VHOST_NET_VQ_MAX: return -ENOBUFS.
- mutex_lock(&vq.mutex).
- if fd == -1: vhost_clear_msg (drop pending IOTLB messages).
- if !vhost_vq_access_ok(vq): return -EFAULT.
- sock = get_socket(fd). // tries raw_socket then tap_socket (tun_get_socket / tap_get_socket).
- if PTR_ERR(sock): return.
- if sock != oldsock:
  - ubufs = vhost_net_ubuf_alloc(vq, sock && vhost_sock_zcopy(sock)).
  - vhost_net_disable_vq.
  - vhost_vq_set_backend(vq, sock).
  - vhost_net_buf_unproduce.
  - vhost_vq_init_access (snapshot avail/used pointers).
  - vhost_net_enable_vq (vhost_poll_start on sock.file).
  - if index == RX: nvq.rx_ring = get_tap_ptr_ring(sock.file).
  - oldubufs = nvq.ubufs; nvq.ubufs = ubufs.
  - tx_packets=0; tx_zcopy_err=0; tx_flush=false.
- mutex_unlock(&vq.mutex).
- if oldubufs: vhost_net_ubuf_put_wait_and_free; vhost_zerocopy_signal_used (drain residual heads).
- if oldsock: vhost_dev_flush; sockfd_put.
- mutex_unlock(&n.dev.mutex).

REQ-6: `get_socket(fd) -> Result<*socket, ERR>`:
- if fd == -1: return NULL (disable backend).
- sock = get_raw_socket(fd):
  - sockfd_lookup; check SOCK_RAW && AF_PACKET; else ESOCKTNOSUPPORT / EPFNOSUPPORT.
- else sock = get_tap_socket(fd):
  - fget(fd); try tun_get_socket (TUN) then tap_get_socket (TAP); fput on failure.
- else return ERR_PTR(-ENOTSOCK).

REQ-7: `vhost_net_set_features(n, features[])`:
- /* Determine hdr_len */
- hdr_len = (test(features, VIRTIO_NET_F_MRG_RXBUF) || test(VIRTIO_F_VERSION_1)) ? sizeof(virtio_net_hdr_mrg_rxbuf) : sizeof(virtio_net_hdr).
- if test(VIRTIO_NET_F_HOST_UDP_TUNNEL_GSO) || test(VIRTIO_NET_F_GUEST_UDP_TUNNEL_GSO): hdr_len = sizeof(virtio_net_hdr_v1_hash_tunnel).
- /* VHOST_NET_F_VIRTIO_NET_HDR: vhost owns the header */
- if test(VHOST_NET_F_VIRTIO_NET_HDR): vhost_hlen = hdr_len; sock_hlen = 0.
- else: vhost_hlen = 0; sock_hlen = hdr_len.
- if test(VHOST_F_LOG_ALL) && !vhost_log_access_ok(&dev): return -EFAULT.
- if test(VIRTIO_F_ACCESS_PLATFORM): vhost_init_device_iotlb.
- for each VQ: lock; virtio_features_copy(acked_features_array, features); nvq.vhost_hlen=vhost_hlen; nvq.sock_hlen=sock_hlen; unlock.

REQ-8: `handle_tx_kick(work)`:
- vq = container_of(work, vhost_virtqueue, poll.work).
- handle_tx(net_of_vq(vq)).

REQ-9: `handle_tx_net(work)` — backend EPOLLOUT path:
- handle_tx(net_of_work(work)).

REQ-10: `handle_tx(net)`:
- mutex_lock_nested(&vq.mutex, VHOST_NET_VQ_TX).
- sock = vhost_vq_get_backend(vq); if !sock: out.
- if !vq_meta_prefetch(vq): out.
- vhost_disable_notify; vhost_net_disable_vq.
- if vhost_sock_zcopy(sock): handle_tx_zerocopy(net, sock).
- else: handle_tx_copy(net, sock).
- mutex_unlock(&vq.mutex).

REQ-11: `handle_tx_copy(net, sock)` per-iteration:
- /* Per-batch flush */
- if done_idx == VHOST_NET_BATCH: vhost_tx_batch(net, nvq, sock, &msg).
- /* Fetch */
- head = get_tx_bufs(net, nvq, &msg, &out, &in, &len, &busyloop_intr, &ndesc).
- if head < 0: break.
- if head == vq.num: vhost_tx_batch; if busyloop_intr: vhost_poll_queue; else if vhost_enable_notify: disable+continue (race fix); break.
- /* XDP path if sndbuf is INT_MAX */
- if sock_can_batch:
  - err = vhost_net_build_xdp(nvq, &msg.msg_iter).
  - if !err: goto done (xdp batched).
  - else if err != -ENOSPC: vhost_tx_batch; vhost_discard_vq_desc; vhost_net_enable_vq; break.
  - if batched_xdp: vhost_tx_batch (flush XDP-batched).
  - msg.msg_control = NULL.
- else:
  - msg.msg_flags |= MSG_MORE if tx_can_batch (total_len < VHOST_NET_WEIGHT && !avail_empty) else &= ~MSG_MORE.
- err = sock.ops.sendmsg(sock, &msg, len):
  - if -EAGAIN / -ENOMEM / -ENOBUFS: vhost_discard_vq_desc; vhost_net_enable_vq; break.
- /* Record used */ if in_order: heads[0].id = head; else heads[done_idx].id = head; heads[done_idx].len = 0; ++done_idx.
- while (!vhost_exceeds_weight(vq, ++sent_pkts, total_len)): repeat.
- vhost_tx_batch (final flush).

REQ-12: `vhost_net_build_xdp(nvq, from)`:
- sock = vhost_vq_get_backend(vq).
- xdp = &nvq.xdp[batched_xdp].
- headroom = vhost_sock_xdp(sock) ? XDP_PACKET_HEADROOM : 0.
- pad = SKB_DATA_ALIGN(VHOST_NET_RX_PAD + headroom + sock_hlen). // VHOST_NET_RX_PAD = NET_IP_ALIGN + NET_SKB_PAD.
- if SKB_DATA_ALIGN(len+pad) + SKB_DATA_ALIGN(sizeof(skb_shared_info)) > PAGE_SIZE: return -ENOSPC.
- buflen = SKB_DATA_ALIGN(sizeof(skb_shared_info)) + SKB_DATA_ALIGN(len + pad).
- buf = page_frag_alloc_align(&net.pf_cache, buflen, GFP_KERNEL, SMP_CACHE_BYTES).
- copy_from_iter(buf + pad − sock_hlen, len, from) — must equal len.
- gso = (virtio_net_hdr*)(buf + pad − sock_hlen).
- /* Per-NEEDS_CSUM: validate csum_start+csum_offset+2 ≤ hdr_len, fix up hdr_len if needed; bound by len */
- memcpy(buf, buf + pad − sock_hlen, sock_hlen).
- xdp_init_buff(xdp, buflen, NULL).
- xdp_prepare_buff(xdp, buf, pad, len − sock_hlen, true).
- ++batched_xdp.

REQ-13: `vhost_tx_batch(net, nvq, sock, msghdr)`:
- in_order = vhost_has_feature(vq, VIRTIO_F_IN_ORDER).
- ctl = { .type = TUN_MSG_PTR, .num = batched_xdp, .ptr = nvq.xdp }.
- if in_order: heads[0].len = 0; nheads[0] = done_idx.
- if batched_xdp == 0: goto signal_used.
- msghdr.msg_control = &ctl; msghdr.msg_controllen = sizeof(ctl).
- err = sock.ops.sendmsg(sock, msghdr, 0).
- if err < 0: for each i in batched_xdp: put_page(virt_to_head_page(xdp[i].data)); batched_xdp=0; done_idx=0; return.
- signal_used: vhost_net_signal_used(nvq, in_order ? 1 : done_idx); batched_xdp = 0.

REQ-14: `handle_tx_zerocopy(net, sock)`:
- Per-iter: vhost_zerocopy_signal_used(net, vq) (drain DMA-done).
- head = get_tx_bufs(...).
- zcopy_used = len ≥ VHOST_GOODCOPY_LEN (256) && !vhost_exceeds_maxpend (UIO_MAXIOV/(VHOST_MAX_PEND=128) cap) && vhost_net_tx_select_zcopy (tx_packets/64 ≥ tx_zcopy_err && !tx_flush).
- if zcopy_used: ubuf = ubuf_info[upend_idx]; heads[upend_idx] = {.id=head, .len=VHOST_DMA_IN_PROGRESS=1}; ubuf.ctx=ubufs; ubuf.desc=upend_idx; ubuf.ops=&vhost_ubuf_ops; ubuf.flags=SKBFL_ZEROCOPY_FRAG; refcount_set(ubuf.refcnt,1); ctl = {.type=TUN_MSG_UBUF, .ptr=&ubuf.ubuf}; msg.msg_control=&ctl; atomic_inc(&ubufs.refcount); upend_idx = (upend_idx + 1) % UIO_MAXIOV.
- else: msg.msg_control = NULL; ubufs = NULL.
- err = sock.ops.sendmsg(sock, &msg, len).
- if err < 0: if zcopy_used: vhost_net_ubuf_put / set heads[ubuf.desc].len = VHOST_DMA_DONE_LEN; if retry: vhost_discard_vq_desc; vhost_net_enable_vq; break.
- if !zcopy_used: vhost_add_used_and_signal(&dev, vq, head, 0).
- else: vhost_zerocopy_signal_used.
- vhost_net_tx_packet (counter).

REQ-15: `vhost_zerocopy_signal_used(net, vq)`:
- /* Scan heads[done_idx .. upend_idx) */
- for i in done_idx .. upend_idx (mod UIO_MAXIOV):
  - if heads[i].len == VHOST_DMA_FAILED_LEN (3): vhost_net_tx_err.
  - if VHOST_DMA_IS_DONE(heads[i].len) (≥ VHOST_DMA_DONE_LEN = 2): heads[i].len = VHOST_DMA_CLEAR_LEN (0); ++j.
  - else break.
- while j: add = min(UIO_MAXIOV - done_idx, j); vhost_add_used_and_signal_n(&dev, vq, &heads[done_idx], NULL, add); done_idx = (done_idx + add) % UIO_MAXIOV; j -= add.

REQ-16: `vhost_zerocopy_complete(skb, ubuf_base, success)`:
- ubuf = uarg_to_msgzc(ubuf_base); ubufs = ubuf.ctx; vq = ubufs.vq.
- rcu_read_lock_bh.
- heads[ubuf.desc].len = success ? VHOST_DMA_DONE_LEN : VHOST_DMA_FAILED_LEN.
- cnt = vhost_net_ubuf_put(ubufs).
- /* Wake poll: refcount drained, or every 16 packets */
- if cnt ≤ 1 || !(cnt % 16): vhost_poll_queue(&vq.poll).
- rcu_read_unlock_bh.

REQ-17: `handle_rx(net)`:
- mutex_lock_nested(&vq.mutex, VHOST_NET_VQ_RX); sock = vhost_vq_get_backend(vq); !sock → out.
- if !vq_meta_prefetch(vq): out.
- vhost_disable_notify; vhost_net_disable_vq.
- vq_log = test(VHOST_F_LOG_ALL) ? vq.log : NULL.
- mergeable = test(VIRTIO_NET_F_MRG_RXBUF); set_num_buffers = mergeable || test(VIRTIO_F_VERSION_1).
- Per-iter:
  - sock_len = vhost_net_rx_peek_head_len(net, sock.sk, &busyloop_intr, &count).
  - if !sock_len: break.
  - sock_len += sock_hlen; vhost_len = sock_len + vhost_hlen.
  - headcount = get_rx_bufs(nvq, heads+count, nheads+count, vhost_len, &in, vq_log, &log, mergeable ? UIO_MAXIOV : 1, &ndesc).
  - if headcount < 0: out.
  - if !headcount: if busyloop_intr: vhost_poll_queue; else if vhost_enable_notify: disable+continue (race fix); out.
  - if rx_ring: msg.msg_control = vhost_net_buf_consume(&rxq) (skb from ptr_ring).
  - if headcount > UIO_MAXIOV: iov_iter_init for 1; recvmsg with MSG_TRUNC; pr_debug discard; continue.
  - iov_iter_init(&msg.msg_iter, ITER_DEST, vq.iov, in, vhost_len); fixup = msg.msg_iter.
  - if vhost_hlen: iov_iter_advance(&msg.msg_iter, vhost_hlen). // vhost supplies hdr.
  - err = sock.ops.recvmsg(sock, &msg, sock_len, MSG_DONTWAIT | MSG_TRUNC).
  - if err != sock_len: vhost_discard_vq_desc(vq, headcount, ndesc); continue.
  - /* Header: vhost-supplied vs socket-supplied */
  - if vhost_hlen: copy_to_iter(&hdr, sizeof(hdr), &fixup) — vhost writes the vnet_hdr.
  - else: iov_iter_advance(&fixup, sizeof(hdr)) — socket already wrote it.
  - /* num_buffers patch for MRG_RXBUF */
  - num_buffers = cpu_to_vhost16(vq, headcount).
  - if set_num_buffers: copy_to_iter(&num_buffers, sizeof(num_buffers), &fixup); fail → discard.
  - done_idx += headcount; count += in_order ? 1 : headcount.
  - if done_idx > VHOST_NET_BATCH: vhost_net_signal_used(nvq, count); count = 0.
  - if vq_log: vhost_log_write.
  - total_len += vhost_len.
- while (!vhost_exceeds_weight(vq, ++recv_pkts, total_len)).
- if busyloop_intr: vhost_poll_queue; else if !sock_len: vhost_net_enable_vq.
- vhost_net_signal_used(nvq, count); mutex_unlock.

REQ-18: `get_rx_bufs(nvq, heads, nheads, datalen, iovcount, log, log_num, quota, ndesc)`:
- in_order = vhost_has_feature(vq, VIRTIO_F_IN_ORDER).
- While datalen > 0 && headcount < quota:
  - if seg ≥ UIO_MAXIOV: r=-ENOBUFS; goto err.
  - r = vhost_get_vq_desc_n(vq, iov+seg, ARRAY_SIZE(iov)-seg, &out, &in, log, log_num, &desc_num) — supports indirect & chained descriptors.
  - d = r; if d == vq.num: r=0; goto err.
  - if out || in ≤ 0: vq_err; r=-EINVAL; goto err.
  - len = iov_length(iov+seg, in).
  - if !in_order: heads[headcount] = {.id = d, .len = len}.
  - ++headcount; datalen -= len; seg += in; n += desc_num.
- *iovcount = seg; if datalen > 0: r = UIO_MAXIOV + 1; goto err (overrun).
- if !in_order: heads[headcount-1].len = len + datalen.
- else: heads[0] = {.len = len + datalen, .id = d}; nheads[0] = headcount.
- *ndesc = n; return headcount.
- err: vhost_discard_vq_desc(vq, headcount, n); return r.

REQ-19: `peek_head_len(rvq, sk) / vhost_net_rx_peek_head_len(net, sk, busyloop_intr, count)`:
- if rx_ring: return vhost_net_buf_peek(rvq) (ptr_ring batch path).
- spin_lock_irqsave(&sk_receive_queue.lock); head = skb_peek; len = head.len + (vlan_present ? VLAN_HLEN : 0); spin_unlock_irqrestore.
- vhost_net_rx_peek_head_len additionally: if !len && tvq.busyloop_timeout: vhost_net_signal_used(nvq, *count); *count=0; vhost_net_busy_poll(net, rvq, tvq, busyloop_intr, true); re-peek.

REQ-20: `vhost_net_busy_poll(net, rvq, tvq, busyloop_intr, poll_rx)`:
- vq = poll_rx ? tvq : rvq (the *paired* VQ — we hold the polled-VQ mutex already).
- if !mutex_trylock(&vq.mutex): return (lock-order safety).
- vhost_disable_notify on paired.
- migrate_disable; endtime = busy_clock() + busyloop_timeout.
- Spin while vhost_can_busy_poll (no need_resched, no signal, !time_after):
  - if vhost_vq_has_work(vq): busyloop_intr=true; break.
  - if (sock_has_rx_data && !avail_empty(rvq)) || !avail_empty(tvq): break.
  - cpu_relax.
- migrate_enable.
- if poll_rx || sock_has_rx_data: vhost_net_busy_poll_try_queue(net, vq).
- else if !poll_rx: vhost_enable_notify on rvq.
- mutex_unlock.

REQ-21: `vhost_net_buf_*` (RX ptr_ring batch cache):
- vhost_net_buf_init: head=0; tail=0.
- vhost_net_buf_produce: tail = ptr_ring_consume_batched(rx_ring, queue, VHOST_NET_BATCH); head = 0; return tail.
- vhost_net_buf_consume: ptr = queue[head++]; return ptr (an sk_buff* or xdp_frame*).
- vhost_net_buf_peek: if empty produce; len = vhost_net_buf_peek_len(queue[head]); return len.
- vhost_net_buf_peek_len: if !((unsigned long)ptr & 1): skb path → skb.len + (vlan_present ? VLAN_HLEN : 0); else xdp_frame path: __ptr_to_xdp_frame.len.
- vhost_net_buf_unproduce: drain any cached entries back / drop.

REQ-22: `vhost_net_signal_used(nvq, count)`:
- vq = &nvq.vq; dev = vq.dev.
- if !done_idx: return.
- vhost_add_used_and_signal_n(dev, vq, vq.heads, vq.nheads, count).
- done_idx = 0.

REQ-23: `vhost_net_ubuf_*`:
- vhost_net_ubuf_alloc(vq, zcopy): if !zcopy: return NULL; kmalloc struct vhost_net_ubuf_ref; refcount=1; init_waitqueue_head(wait); ubufs.vq = vq.
- vhost_net_ubuf_put(ubufs): atomic_dec(&refcount); if 1: wake_up(wait); return new value.
- vhost_net_ubuf_put_and_wait: put; wait_event(wait, atomic_read(refcount) == 1).
- vhost_net_ubuf_put_wait_and_free: put_and_wait; kfree_rcu(ubufs, rcu).
- vhost_net_set_ubuf_info(n): for each tx-zcopy VQ allocate ubuf_info_msgzc[UIO_MAXIOV].
- vhost_net_clear_ubuf_info(n): kfree on each.

REQ-24: `vhost_net_enable_vq(n, vq) / disable_vq(n, vq)`:
- enable: sock = vhost_vq_get_backend(vq); if !sock: return 0; return vhost_poll_start(poll, sock.file).
- disable: if !vhost_vq_get_backend(vq): return; vhost_poll_stop(poll).

REQ-25: `vhost_net_stop(n, *tx_sock, *rx_sock)` / `vhost_net_stop_vq`:
- For each VQ: mutex_lock; sock = vhost_vq_get_backend; vhost_net_disable_vq; vhost_vq_set_backend(vq, NULL); vhost_net_buf_unproduce; rx_ring = NULL; mutex_unlock.
- Returns the held sock to caller for sockfd_put.

REQ-26: `vhost_net_flush(n)`:
- vhost_dev_flush(&n.dev) (drain workqueue).
- if tx ubufs: lock; tx_flush=true; unlock; vhost_net_ubuf_put_and_wait; lock; tx_flush=false; refcount = 1; unlock.

REQ-27: `vhost_net_ioctl(file, ioctl, arg)`:
- VHOST_NET_SET_BACKEND: copy_from_user(backend); vhost_net_set_backend(n, backend.index, backend.fd).
- VHOST_GET_FEATURES: copy_to_user(vhost_net_features[0]).
- VHOST_SET_FEATURES: copy_from_user; reject any bits outside `vhost_net_bits` mask; virtio_features_from_u64; vhost_net_set_features.
- VHOST_GET_FEATURES_ARRAY / VHOST_SET_FEATURES_ARRAY: extended (multi-u64) variants for features above bit 63.
- VHOST_GET_BACKEND_FEATURES: returns VHOST_NET_BACKEND_FEATURES = (1 << VHOST_BACKEND_F_IOTLB_MSG_V2).
- VHOST_SET_BACKEND_FEATURES: vhost_set_backend_features.
- VHOST_RESET_OWNER: vhost_net_reset_owner.
- VHOST_SET_OWNER: vhost_net_set_owner → vhost_net_set_ubuf_info → vhost_dev_set_owner → vhost_net_flush.
- default: vhost_dev_ioctl → vhost_vring_ioctl (per-vring set num / addr / base / kick / call / err / state / endian / busyloop_timeout / packed-ring).

REQ-28: `vhost_net_chr_{read,write}_iter / poll`:
- /dev/vhost-net read/write/poll = IOTLB control channel (when VIRTIO_F_ACCESS_PLATFORM negotiated).
- vhost_chr_read_iter / vhost_chr_write_iter / vhost_chr_poll.

REQ-29: Per-VHOST_NET_F_VIRTIO_NET_HDR:
- 1 (negotiated): vhost owns vnet_hdr (vhost_hlen ≠ 0, sock_hlen = 0). RX: vhost writes hdr into descriptor. TX: vhost skips hdr from socket-side iter advance.
- 0 (not negotiated): socket owns vnet_hdr (vhost_hlen = 0, sock_hlen = hdr_len).

REQ-30: Per-VIRTIO_NET_F_MRG_RXBUF / VIRTIO_F_VERSION_1:
- hdr_len = sizeof(virtio_net_hdr_mrg_rxbuf) (extra `num_buffers` field).
- set_num_buffers ⇒ patch `num_buffers = headcount` after recvmsg.

REQ-31: Per-VIRTIO_NET_F_{HOST,GUEST}_UDP_TUNNEL_GSO:
- hdr_len = sizeof(virtio_net_hdr_v1_hash_tunnel).

REQ-32: Per-VIRTIO_F_IN_ORDER:
- TX: heads[0].id = head (only the *latest* head is stamped; in-order ack).
- RX: heads[0] = {.id=d, .len=len+datalen}; nheads[0] = headcount (one used cell covers `headcount` heads).
- Batch flush: signal_used emits 1 cell rather than done_idx cells.

REQ-33: Per-VIRTIO_F_ACCESS_PLATFORM (IOTLB):
- vhost_init_device_iotlb on set_features.
- /dev/vhost-net read/write becomes the IOTLB message channel.

REQ-34: Per-packed-ring (`VIRTIO_F_RING_PACKED`):
- vhost_get_vq_desc_n dispatches to split or packed walker; same external `out/in/iov/heads` shape.
- vhost_add_used_and_signal_n abstracts split vs packed used-cell write.

REQ-35: Per-indirect descriptors:
- handled inside vhost_get_vq_desc_n: when a descriptor has VRING_DESC_F_INDIRECT, the walker follows the indirect table and counts each entry against UIO_MAXIOV; `desc_num` returns total count for `vhost_discard_vq_desc` accounting.

REQ-36: Per-busyloop:
- tvq.busyloop_timeout / rvq.busyloop_timeout configured via VHOST_SET_VRING_BUSYLOOP_TIMEOUT.
- TX: if get_tx_bufs returns vq.num and tvq.busyloop_timeout: flush XDP if !zcopy; vhost_net_busy_poll(RX→paired); re-fetch.
- RX: similar via vhost_net_rx_peek_head_len.

REQ-37: Per-VHOST_NET_WEIGHT / VHOST_NET_PKT_WEIGHT:
- 0x80000 bytes and 256 packets respectively; vhost_exceeds_weight short-circuits the per-worker loop to allow other VQs / cgroup-scheduling.

REQ-38: Per-XDP path requirement:
- sock_can_batch = (sock.sk.sk_sndbuf == INT_MAX).
- Only TX-copy path tries XDP build.
- TUN_MSG_PTR ctl batches xdp_buffs to tun_sendmsg; tun side runs XDP_TX / XDP_REDIRECT / XDP_PASS.

REQ-39: Per-rx_ring (ptr_ring batch):
- get_tap_ptr_ring(file): tun_get_tx_ring then tap_get_ptr_ring.
- vhost_net_buf_peek/produce/consume uses ptr_ring_consume_batched for VHOST_NET_BATCH skbs at a time.
- msg.msg_control = vhost_net_buf_consume(&rxq) — passes the dequeued skb directly to recvmsg.

## Acceptance Criteria

- [ ] AC-1: `open(/dev/vhost-net)` returns an fd; `VHOST_SET_OWNER` succeeds once and binds owner-task.
- [ ] AC-2: `VHOST_NET_SET_BACKEND` with a tun fd attaches the socket; subsequent guest avail-ring kicks invoke handle_tx_kick.
- [ ] AC-3: handle_tx_copy: per-packet sendmsg called on backend sock with iov skipping vhost_hlen.
- [ ] AC-4: handle_tx_zerocopy: `experimental_zcopytx=1` and SOCK_ZEROCOPY socket: len ≥ 256 packets pass `TUN_MSG_UBUF` ctl; zerocopy_complete updates heads[].len.
- [ ] AC-5: handle_rx: tap-to-guest copy: recvmsg fills descriptor chain, then vhost writes virtio_net_hdr + num_buffers (if MRG_RXBUF).
- [ ] AC-6: Indirect descriptor chain: get_rx_bufs traverses VRING_DESC_F_INDIRECT correctly; counts against UIO_MAXIOV.
- [ ] AC-7: VIRTIO_F_IN_ORDER: signal_used emits 1 used cell covering `headcount` heads.
- [ ] AC-8: Packed ring negotiated: handle_tx / handle_rx work; per-VQ access via packed walker.
- [ ] AC-9: VHOST_NET_F_VIRTIO_NET_HDR=1: vhost_hlen = hdr_len, sock_hlen = 0; vhost writes hdr into RX descriptor.
- [ ] AC-10: VHOST_NET_F_VIRTIO_NET_HDR=0: sock_hlen = hdr_len; iter advances over hdr supplied by tap socket.
- [ ] AC-11: VIRTIO_NET_F_MRG_RXBUF: num_buffers patched into hdr after multi-head RX.
- [ ] AC-12: vhost_exceeds_weight: per-worker yields after 0x80000 bytes or 256 packets.
- [ ] AC-13: Busy-poll: VHOST_SET_VRING_BUSYLOOP_TIMEOUT > 0: paired-VQ spin observed; busyloop_intr re-queues poll.
- [ ] AC-14: VHOST_NET_SET_BACKEND(fd = −1): old socket dropped; ubufs put-and-wait drained; vhost_clear_msg purges pending IOTLB.
- [ ] AC-15: release(): all sockfds put; ubufs drained; page_frag_cache drained; no memory leak.

## Architecture

```
struct VhostNetVirtqueue {
  vq: VhostVirtqueue,           // generic per-VQ
  vhost_hlen: usize,            // vhost-supplied vnet_hdr
  sock_hlen: usize,             // socket-supplied vnet_hdr
  upend_idx: i32,               // TX: zcopy producer
  done_idx: i32,                // TX: zcopy consumer / RX: batched-heads count
  batched_xdp: i32,             // TX: xdp_buff count
  ubuf_info: Option<*mut UbufInfoMsgzc>,
  ubufs: Option<*mut VhostNetUbufRef>,
  rx_ring: Option<*mut PtrRing>,
  rxq: VhostNetBuf,             // RX: VHOST_NET_BATCH ptr cache
  xdp: *mut XdpBuff,            // TX: batched xdp_buffs
}

struct VhostNet {
  dev: VhostDev,
  vqs: [VhostNetVirtqueue; VHOST_NET_VQ_MAX],   // [RX=0, TX=1]
  poll: [VhostPoll; VHOST_NET_VQ_MAX],
  tx_packets: u32,
  tx_zcopy_err: u32,
  tx_flush: bool,
  pf_cache: PageFragCache,
}

const VHOST_NET_WEIGHT: usize = 0x80000;
const VHOST_NET_PKT_WEIGHT: usize = 256;
const VHOST_NET_BATCH: usize = 64;
const VHOST_MAX_PEND: usize = 128;
const VHOST_GOODCOPY_LEN: usize = 256;
const VHOST_NET_RX_PAD: usize = NET_IP_ALIGN + NET_SKB_PAD;
```

`VhostNet::open(file)`:
1. n = kvmalloc(GFP_KERNEL | __GFP_RETRY_MAYFAIL).
2. vqs = kmalloc_array(VHOST_NET_VQ_MAX); queue = kmalloc_array(VHOST_NET_BATCH); xdp = kmalloc_array(VHOST_NET_BATCH).
3. RX.rxq.queue = queue; TX.xdp = xdp.
4. TX.vq.handle_kick = handle_tx_kick; RX.vq.handle_kick = handle_rx_kick.
5. For each VQ: zero-init ubufs / ubuf_info / upend_idx / done_idx / batched_xdp / hlens / rx_ring; vhost_net_buf_init(&rxq).
6. vhost_dev_init(&dev, vqs, 2, UIO_MAXIOV + VHOST_NET_BATCH, VHOST_NET_PKT_WEIGHT, VHOST_NET_WEIGHT, true, NULL).
7. vhost_poll_init(&poll[TX], handle_tx_net, EPOLLOUT, &dev, vqs[TX]).
8. vhost_poll_init(&poll[RX], handle_rx_net, EPOLLIN, &dev, vqs[RX]).
9. file.private_data = n; page_frag_cache_init(&pf_cache).

`VhostNet::set_backend(index, fd)`:
1. mutex_lock(dev); vhost_dev_check_owner.
2. mutex_lock(vq); if fd == -1: vhost_clear_msg.
3. if !vhost_vq_access_ok: -EFAULT.
4. sock = get_socket(fd) (raw → tap → tun).
5. if sock != oldsock:
   - ubufs = ubuf_alloc(vq, vhost_sock_zcopy(sock)).
   - disable_vq; set_backend; buf_unproduce; vhost_vq_init_access; enable_vq.
   - if index == RX: rx_ring = get_tap_ptr_ring(sock.file).
   - swap nvq.ubufs.
6. mutex_unlock(vq).
7. if oldubufs: ubuf_put_wait_and_free + zcopy_signal_used.
8. if oldsock: vhost_dev_flush; sockfd_put.

`VhostNet::handle_tx(net)`:
1. mutex_lock_nested(vq, VHOST_NET_VQ_TX); sock = backend(vq); if !sock: out.
2. if !vq_meta_prefetch: out.
3. disable_notify; disable_vq.
4. if vhost_sock_zcopy(sock): handle_tx_zerocopy. else handle_tx_copy.

`VhostNet::handle_tx_copy(net, sock)` (per-iter):
1. if done_idx == VHOST_NET_BATCH: tx_batch.
2. head = get_tx_bufs.
3. if head < 0: break; if head == vq.num: tx_batch; race-check via enable_notify; break.
4. if sock_can_batch (sndbuf == INT_MAX):
   - err = build_xdp(nvq, &msg.msg_iter).
   - !err → goto done (xdp batched).
   - err == -ENOSPC && batched_xdp → tx_batch then fall through.
   - err < 0 ≠ -ENOSPC → tx_batch; vhost_discard_vq_desc; enable_vq; break.
5. else: msg.msg_flags MSG_MORE on tx_can_batch else not.
6. err = sock.ops.sendmsg(sock, &msg, len). On EAGAIN/ENOMEM/ENOBUFS: discard; enable_vq; break.
7. done: heads[done_idx].id = head; heads[done_idx].len = 0; ++done_idx (or in_order one cell).
8. while !vhost_exceeds_weight; final tx_batch.

`VhostNet::handle_tx_zerocopy(net, sock)` (per-iter):
1. zcopy_signal_used (drain DMA-done).
2. head = get_tx_bufs.
3. zcopy_used = len ≥ VHOST_GOODCOPY_LEN && !exceeds_maxpend && tx_select_zcopy.
4. if zcopy_used: ubuf = ubuf_info[upend_idx]; heads[upend_idx] = {head, VHOST_DMA_IN_PROGRESS}; ubuf.ops = &vhost_ubuf_ops; ctl = {TUN_MSG_UBUF, &ubuf.ubuf}; msg.msg_control = &ctl; atomic_inc(ubufs.refcount); upend_idx = (upend_idx + 1) % UIO_MAXIOV.
5. msg.msg_flags MSG_MORE on tx_can_batch && !exceeds_maxpend.
6. err = sock.ops.sendmsg.
7. if !zcopy_used: add_used_and_signal(head, 0). else: zcopy_signal_used; tx_packet().

`VhostNet::build_xdp(nvq, from)`:
1. headroom = vhost_sock_xdp(sock) ? XDP_PACKET_HEADROOM : 0.
2. pad = SKB_DATA_ALIGN(VHOST_NET_RX_PAD + headroom + sock_hlen).
3. SKB_DATA_ALIGN(len+pad) + SKB_DATA_ALIGN(skb_shared_info) > PAGE_SIZE → -ENOSPC.
4. buflen = SKB_DATA_ALIGN(skb_shared_info) + SKB_DATA_ALIGN(len+pad).
5. buf = page_frag_alloc_align(&pf_cache, buflen, GFP_KERNEL, SMP_CACHE_BYTES); !buf → -ENOMEM.
6. copy_from_iter(buf + pad − sock_hlen, len, from); !=len → -EFAULT.
7. /* NEEDS_CSUM: csum_start+csum_offset+2 ≤ hdr_len else fixup; bound by len */.
8. memcpy(buf, buf + pad − sock_hlen, sock_hlen).
9. xdp_init_buff(xdp, buflen, NULL); xdp_prepare_buff(xdp, buf, pad, len − sock_hlen, true).
10. ++batched_xdp.

`VhostNet::tx_batch(nvq, sock, msghdr)`:
1. ctl = {TUN_MSG_PTR, .num = batched_xdp, .ptr = nvq.xdp}.
2. in_order: heads[0].len = 0; nheads[0] = done_idx.
3. if batched_xdp == 0: signal_used; return.
4. msghdr.msg_control = &ctl; err = sock.ops.sendmsg(sock, msghdr, 0).
5. if err < 0: for i in 0..batched_xdp: put_page(virt_to_head_page(xdp[i].data)); reset; return.
6. signal_used(nvq, in_order ? 1 : done_idx).

`VhostNet::handle_rx(net)` (per-iter):
1. sock_len = rx_peek_head_len(net, sock.sk, &busyloop_intr, &count).
2. !sock_len → break.
3. vhost_len = sock_len + sock_hlen + vhost_hlen.
4. headcount = get_rx_bufs(nvq, heads+count, nheads+count, vhost_len, &in, vq_log, &log, mergeable ? UIO_MAXIOV : 1, &ndesc).
5. !headcount → busyloop or enable_notify race or out.
6. rx_ring → msg.msg_control = buf_consume(&rxq).
7. headcount > UIO_MAXIOV → recvmsg-truncate; continue.
8. iov_iter_init(ITER_DEST, vq.iov, in, vhost_len); fixup = msg.msg_iter; if vhost_hlen: iov_iter_advance(&msg.msg_iter, vhost_hlen).
9. err = sock.ops.recvmsg(sock, &msg, sock_len, MSG_DONTWAIT|MSG_TRUNC).
10. if vhost_hlen: copy_to_iter(&hdr, sizeof(hdr), &fixup). else: iov_iter_advance(&fixup, sizeof(hdr)).
11. if set_num_buffers: copy_to_iter(&num_buffers=headcount, sizeof(num_buffers), &fixup).
12. done_idx += headcount; count += in_order ? 1 : headcount.
13. if done_idx > VHOST_NET_BATCH: signal_used(nvq, count); count=0.

`VhostNet::get_rx_bufs(...)`:
1. Loop datalen > 0 && headcount < quota:
   - seg ≥ UIO_MAXIOV → -ENOBUFS.
   - r = vhost_get_vq_desc_n(vq, iov+seg, n-seg, &out, &in, log, log_num, &desc_num).
   - d == vq.num → return 0.
   - out || in ≤ 0 → -EINVAL.
   - len = iov_length(iov+seg, in).
   - !in_order: heads[headcount] = {d, len}.
   - ++headcount; datalen -= len; seg += in; n += desc_num.
2. datalen > 0 → UIO_MAXIOV+1 overrun.
3. in_order: heads[0] = {d, len+datalen}; nheads[0] = headcount.
4. *ndesc = n; return headcount.

`VhostNet::zcopy_complete(skb, ubuf_base, success)`:
1. ubuf = uarg_to_msgzc(ubuf_base); vq = ubuf.ctx.vq.
2. rcu_read_lock_bh.
3. heads[ubuf.desc].len = success ? VHOST_DMA_DONE_LEN : VHOST_DMA_FAILED_LEN.
4. cnt = ubuf_put(ubufs).
5. if cnt ≤ 1 || !(cnt % 16): vhost_poll_queue(&vq.poll).
6. rcu_read_unlock_bh.

`VhostNet::set_features(features)`:
1. hdr_len = MRG_RXBUF || VERSION_1 ? sizeof(virtio_net_hdr_mrg_rxbuf) : sizeof(virtio_net_hdr).
2. UDP_TUNNEL_GSO → sizeof(virtio_net_hdr_v1_hash_tunnel).
3. VHOST_NET_F_VIRTIO_NET_HDR ? (vhost_hlen=hdr_len, sock_hlen=0) : (vhost_hlen=0, sock_hlen=hdr_len).
4. VHOST_F_LOG_ALL → require vhost_log_access_ok.
5. VIRTIO_F_ACCESS_PLATFORM → vhost_init_device_iotlb.
6. For each VQ: lock; copy acked_features_array; assign hlens; unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tx_done_idx_in_range` | INVARIANT | per-handle_tx_copy: done_idx ∈ [0, VHOST_NET_BATCH]. |
| `tx_upend_idx_wraps` | INVARIANT | per-handle_tx_zerocopy: upend_idx ∈ [0, UIO_MAXIOV); always mod. |
| `tx_batch_zero_after_flush` | INVARIANT | per-tx_batch: batched_xdp == 0 ∧ done_idx == 0 on return. |
| `rx_headcount_bounded` | INVARIANT | per-get_rx_bufs: headcount ≤ quota; quota ≤ UIO_MAXIOV. |
| `rx_iov_seg_bounded` | INVARIANT | per-get_rx_bufs: seg ≤ UIO_MAXIOV (else -ENOBUFS). |
| `ubuf_refcount_balanced` | INVARIANT | per-handle_tx_zerocopy: every atomic_inc paired with put on success/error. |
| `pf_cache_drained_on_release` | INVARIANT | per-vhost_net_release: page_frag_cache_drain called. |
| `backend_mutex_held_during_io` | INVARIANT | per-handle_tx / handle_rx: vq.mutex held for entire duration. |
| `paired_vq_trylock_only` | INVARIANT | per-vhost_net_busy_poll: paired vq accessed via mutex_trylock (lock-order safety). |
| `xdp_buflen_under_page` | INVARIANT | per-build_xdp: SKB_DATA_ALIGN(len+pad)+SKB_DATA_ALIGN(skb_shared_info) ≤ PAGE_SIZE. |

### Layer 2: TLA+

`drivers/vhost/net.tla`:
- Per-VQ state machine: Idle → Polling → Processing(N batches) → Signaling → Idle.
- Properties:
  - `safety_tx_used_eventually_signaled` — per-handle_tx_copy: every successful sendmsg ⟹ heads[done_idx] eventually published via vhost_add_used_and_signal_n.
  - `safety_zcopy_completion_ordered` — per-vhost_zerocopy_complete: heads[ubuf.desc].len transitions IN_PROGRESS → {DONE, FAILED} exactly once.
  - `safety_rx_num_buffers_matches_headcount` — per-handle_rx + MRG_RXBUF: published `num_buffers` == `headcount`.
  - `safety_features_immutable_during_io` — per-handle_tx/rx: acked_features_array stable while vq.mutex held.
  - `liveness_per_TX_kick_eventually_drained` — per-handle_tx: bounded iteration via vhost_exceeds_weight.
  - `liveness_per_RX_pending_eventually_consumed` — per-handle_rx: if backend has data and avail-ring non-empty, recvmsg eventually runs.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `VhostNet::open` post: vqs[].handle_kick set; poll[] init; pf_cache init | `VhostNet::open` |
| `VhostNet::set_backend` post: nvq.rx_ring set iff index == RX; ubufs lifecycle balanced | `VhostNet::set_backend` |
| `VhostNet::get_rx_bufs` post: headcount > 0 ⟹ Σ heads[].len ≥ datalen | `VhostNet::get_rx_bufs` |
| `VhostNet::handle_tx_copy` post: every consumed avail-ring entry has a corresponding used-ring entry or vhost_discard_vq_desc call | `VhostNet::handle_tx_copy` |
| `VhostNet::handle_rx` post: vhost_hlen ⟹ hdr written into descriptor; MRG_RXBUF ⟹ num_buffers patched | `VhostNet::handle_rx` |
| `VhostNet::tx_batch` post: on err < 0: all xdp pages put_page'd | `VhostNet::tx_batch` |
| `VhostNet::zcopy_complete` post: heads[ubuf.desc].len set; ubufs.refcount decremented | `VhostNet::zcopy_complete` |
| `VhostNet::busy_poll` post: paired vq mutex released; migrate_enable on every exit | `VhostNet::busy_poll` |
| `VhostNet::set_features` post: vhost_hlen XOR sock_hlen == 0 (mutually exclusive when feature off) | `VhostNet::set_features` |

### Layer 4: Verus/Creusot functional

`Per-VHOST_NET_SET_BACKEND → vhost_poll_start(sock.file) → guest kick → handle_tx_kick → handle_tx → sendmsg(sock, iov, len) → vhost_add_used_and_signal_n → guest interrupt`: semantic equivalence vs per-Documentation/virt/kvm/devices/vfio.rst and Documentation/networking/tuntap.rst data-path contract.

`Per-VHOST_NET_F_VIRTIO_NET_HDR(0) → sock owns hdr → iov includes hdr` vs `Per-VHOST_NET_F_VIRTIO_NET_HDR(1) → vhost owns hdr → iov skips hdr; vhost writes hdr on RX`: bit-exact wire equivalence per virtio-1.3 § 5.1.6.

`Per-VIRTIO_F_IN_ORDER negotiated → single used-cell per batch`: semantic equivalence per virtio-1.3 § 2.7.7.

## Hardening

(Inherits row-1 features from `drivers/vhost/00-overview.md` § Hardening.)

vhost-net reinforcement:

- **Per-vq.mutex covers the whole worker iteration** — defense against per-feature-flip race / per-backend-swap during sendmsg.
- **Per-vhost_dev_check_owner on all ioctls** — defense against per-cross-task ioctl that would corrupt VM state.
- **Per-mutex_trylock for paired-VQ busy poll** — defense against per-lock-order deadlock (TX↔RX).
- **Per-ubufs reference-count get/put strict** — defense against per-zerocopy UAF on flush / backend swap.
- **Per-vhost_net_ubuf_put_and_wait on backend swap / flush** — defense against per-DMA-outstanding when freeing ring/socket.
- **Per-vhost_exceeds_weight bounds worker run** — defense against per-VQ starvation of others under load.
- **Per-VHOST_GOODCOPY_LEN gate** — defense against per-zerocopy on tiny packets where setup cost exceeds savings.
- **Per-vhost_exceeds_maxpend cap (UIO_MAXIOV / VHOST_MAX_PEND)** — defense against per-unbounded outstanding DMAs.
- **Per-build_xdp PAGE_SIZE check** — defense against per-XDP-oversize that would spill across pages.
- **Per-get_rx_bufs UIO_MAXIOV seg guard + overrun detect** — defense against per-malicious-descriptor iov overflow.
- **Per-headcount > UIO_MAXIOV: recvmsg-truncate-discard** — defense against per-guest forcing OOB writes.
- **Per-rcu_read_lock_bh in zcopy_complete** — defense against per-completion observing freed vq.
- **Per-page_frag_cache_drain on release** — defense against per-fd-leak of pages.
- **Per-VHOST_F_LOG_ALL requires vhost_log_access_ok** — defense against per-dirty-log writing outside owner mm.
- **Per-VIRTIO_F_ACCESS_PLATFORM IOTLB validation** — defense against per-guest-DMA outside negotiated GPA→HVA range.
- **Per-backend swap drains old ubufs before kfree** — defense against per-DMA-into-freed-ubuf.
- **Per-vhost_clear_msg on fd=−1** — defense against per-stale IOTLB message after backend disable.
- **Per-experimental_zcopytx default off** — defense against per-untested zerocopy in production.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `VHOST_NET_SET_BACKEND`, `VHOST_SET_VRING_*`, and feature-mask ioctls bounce through bounded `copy_*_user`; iov-iter walks of guest memory go through `copy_from_iter_full` / `copy_to_iter` with length capped at `vq->iov_limit`.
- **PAX_KERNEXEC** — `vhost_net_ops`, per-vq `handle_kick` table, and `vhost_dev_ops` placed in `__ro_after_init`.
- **PAX_RANDKSTACK** — entropy added on every `vhost_net_open` / `vhost_net_ioctl` / `handle_tx` / `handle_rx` entry; neutralises stack-shape probing under vhost-poll races.
- **PAX_REFCOUNT** — `vhost_net.dev.refcnt`, `vq->kref`, and `ubufs->refcount` for zerocopy use saturating counters.
- **PAX_MEMORY_SANITIZE** — virtio-net header scratch (`vhost_hdr`), zerocopy `ubuf_info`, and IOTLB message buffers zero-on-free.
- **PAX_UDEREF** — every vhost-net ioctl dereferences user pointers only via user-AS-annotated copy helpers; mem-table user-VA ranges validated against `access_ok` before iov-iter installation.
- **PAX_RAP / kCFI** — `vhost_poll.fn`, `vhost_work.fn`, `handle_tx`, and `handle_rx` work-callbacks type-tagged.
- **GRKERNSEC_HIDESYM** — `/dev/vhost-net` and per-vq trace prints strip kernel pointers.
- **GRKERNSEC_DMESG** — IOTLB-fault / mem-table validation-failure prints gated behind `CAP_SYSLOG`.
- **/dev/vhost-net CAP_NET_ADMIN** — `open("/dev/vhost-net")` and `VHOST_NET_SET_BACKEND` require `CAP_NET_ADMIN`; defends against unprivileged installation of attacker-controlled tap/macvtap fds.
- **virtio-net DMA buffer PAX_USERCOPY** — guest-physical buffer translations performed via `vhost_iotlb_itree_iter_first` are bounded against `vq->iov[i].iov_len`; defends against per-IOTLB-translation overflow producing kernel-overlap reads.
- **Rationale** — vhost-net pipes guest network frames into the host's tap layer at near-line-rate; CAP_NET_ADMIN gating, IOTLB-bound USERCOPY, and RAP on the work-callbacks close the historic vhost-net "iov-iter overrun" and "ubuf double-free" families.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- drivers/vhost/vhost.c (vhost_dev / vhost_virtqueue / vhost_get_vq_desc_n / packed-ring walker / vhost_poll / vhost_work) — covered in `drivers/vhost/vhost.md` Tier-3.
- drivers/net/tun.c (tun_recvmsg / tun_sendmsg backend, XDP_TX / XDP_REDIRECT) — covered in `drivers/net/tun.md` Tier-3.
- drivers/net/tap.c — covered separately if expanded.
- include/uapi/linux/virtio_net.h ABI (per-bit semantics for VIRTIO_NET_F_*) — covered in virtio Tier-3.
- KVM guest-side virtio-net driver — covered in `drivers/net/virtio_net.md` Tier-3.
- VFIO / mdev pass-through — covered separately.
- Implementation code.
