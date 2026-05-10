# Tier-3: net/ipv4/ip_output.c — IPv4 transmit path (local_out, finish_output, fragmentation, corking)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/ipv4/00-overview.md
upstream-paths:
  - net/ipv4/ip_output.c (~1694 lines)
  - include/net/ip.h
  - include/net/inet_sock.h (struct inet_cork)
  - include/linux/skbuff.h (IPCB(skb), struct inet_skb_parm)
-->

## Summary

`net/ipv4/ip_output.c` is the IPv4 **transmit (egress)** half of the IP layer. Per-skb flow: transport layer → `ip_queue_xmit` / `ip_build_and_send_pkt` → builds IPv4 header → `ip_local_out` → NF `NF_INET_LOCAL_OUT` hook → `dst_output` → `ip_output` / `ip_mc_output` → NF `NF_INET_POST_ROUTING` hook → `ip_finish_output` → (GSO segmenting | fragmentation | direct) → `ip_finish_output2` → neighbour cache lookup (`ip_neigh_for_gw`) → `neigh_output` → device queue. Per-corking path (UDP, RAW, ICMP send buffers): `ip_setup_cork` → `ip_append_data` / `ip_append_page` build a chained sk_buff queue → `ip_push_pending_frames` → `__ip_make_skb` glues into a single datagram → `ip_send_skb` → `ip_local_out`. Per-fragmentation: `ip_fragment` validates DF / max_size, then `ip_do_fragment` walks `skb_shinfo->frag_list` fast-path or copies via `ip_frag_init` + `ip_frag_next` (RFC791 8-byte alignment, `IP_MF`/`IP_DF` bits, IPCB flag propagation). Per-GSO egress: `ip_finish_output_gso` segments oversize TSO/UFO skbs via `skb_gso_segment` then re-fragments per segment. Critical for: TCP/UDP/RAW datagram emission, tunneling integration via IPCB IPSKB_* flags, PMTU discovery via ICMP fragmentation-needed, neighbor cache integration, BPF cgroup egress hook, per-skb netfilter visibility.

This Tier-3 covers `net/ipv4/ip_output.c` (~1694 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `ip_send_check(iph)` | per-iph compute IPv4 header checksum | `IpOutput::send_check` |
| `__ip_local_out(net, sk, skb)` | per-skb totlen+csum, L3-master, NF LOCAL_OUT | `IpOutput::local_out_inner` |
| `ip_local_out(net, sk, skb)` | per-skb local-out + dst_output dispatch | `IpOutput::local_out` |
| `ip_build_and_send_pkt(skb, sk, saddr, daddr, opt, tos)` | per-SYNACK / minimal builder + emit | `IpOutput::build_and_send` |
| `ip_queue_xmit(sk, skb, fl)` | per-TCP egress entry | `IpOutput::queue_xmit` |
| `__ip_queue_xmit(sk, skb, fl, tos)` | per-skb route+build+emit | `IpOutput::queue_xmit_inner` |
| `ip_output(net, sk, skb)` | per-skb POST_ROUTING NF + ip_finish_output | `IpOutput::output` |
| `ip_mc_output(net, sk, skb)` | per-skb multicast egress (loopback clone) | `IpOutput::mc_output` |
| `ip_mc_finish_output(net, sk, skb)` | per-skb mc POST_ROUTING completion | `IpOutput::mc_finish_output` |
| `ip_finish_output(net, sk, skb)` | per-skb BPF egress + __ip_finish_output | `IpOutput::finish_output` |
| `__ip_finish_output(net, sk, skb)` | per-skb GSO vs fragment vs direct dispatch | `IpOutput::finish_output_inner` |
| `ip_finish_output_gso(net, sk, skb, mtu)` | per-skb GSO segment + per-seg fragment | `IpOutput::finish_output_gso` |
| `ip_finish_output2(net, sk, skb)` | per-skb neigh lookup + `neigh_output` | `IpOutput::finish_output2` |
| `ip_fragment(net, sk, skb, mtu, output)` | per-skb DF gate + ICMP FRAG_NEEDED | `IpOutput::fragment` |
| `ip_do_fragment(net, sk, skb, output)` | per-skb actual fragmentation | `IpOutput::do_fragment` |
| `ip_fraglist_init / _prepare / _next` | per-skb fast-path fraglist iterator | `IpOutput::fraglist_*` |
| `ip_frag_init(skb, hlen, ll_rs, mtu, DF, state)` | per-skb slow-path frag state init | `IpOutput::frag_init` |
| `ip_frag_next(skb, state)` | per-fragment slow-path alloc + copy | `IpOutput::frag_next` |
| `ip_setup_cork(sk, cork, ipc, rtp)` | per-cork init (fragsize, opt, dst) | `IpOutput::setup_cork` |
| `ip_append_data(sk, fl4, getfrag, ...)` | per-cork append user-data into queue | `IpOutput::append_data` |
| `__ip_append_data(...)` | per-cork core appender | `IpOutput::append_data_inner` |
| `ip_append_page(sk, fl4, page, ...)` | per-cork append page (splice) | `IpOutput::append_page` |
| `ip_push_pending_frames(sk, fl4)` | per-cork flush → ip_send_skb | `IpOutput::push_pending` |
| `ip_make_skb(sk, fl4, ...)` | per-shot build (UDP no-cork single) | `IpOutput::make_skb` |
| `__ip_make_skb(sk, fl4, queue, cork)` | per-queue concat + iph build | `IpOutput::make_skb_inner` |
| `ip_send_skb(net, skb)` | per-skb final ip_local_out wrap | `IpOutput::send_skb` |
| `ip_flush_pending_frames(sk)` | per-sock drop cork queue | `IpOutput::flush_pending` |
| `ip_cork_release(cork)` | per-cork free opt + dst_release | `IpOutput::cork_release` |
| `ip_send_unicast_reply(...)` | per-skb TCP-RST/ACK reply path | `IpOutput::send_unicast_reply` |
| `ip_neigh_for_gw(rt, skb, &is_v6gw)` | per-skb neighbor lookup for gw | shared `Neigh::for_gw` |
| `struct inet_cork` | per-socket cork state | `InetCork` |
| `IPCB(skb) / struct inet_skb_parm` | per-skb IP parm + IPSKB_* flags | `IpCb` |

## Compatibility contract

REQ-1: `struct inet_cork` (corking state, per-socket inet_sk(sk)->cork.base):
- flags: u8 — `IPCORK_OPT` (cork->opt valid) | `IPCORK_TS_OPT_ID` (ts_opt_id valid) | `IPCORK_ALLFRAG`.
- gso_size: u16 — per-cork GSO segment size; 0 = no GSO.
- fragsize: u32 — per-cork path-MTU (sk pmtu) or dev MTU.
- length: u32 — bytes appended so far (cumulative).
- dst: *Dst — stolen route entry.
- addr: __be32 — for IP_RECVOPTS or strict-source-route.
- opt: *IpOptions — IP options blob (kmalloc'd, freed on release).
- ttl, tos: per-cork override.
- mark, priority: skb metadata.
- transmit_time: u64 — earliest TX (EDT model).
- tx_flags: skb shinfo tx_flags (timestamping).
- ts_opt_id: per-cork user-supplied timestamp opt id.

REQ-2: `struct inet_skb_parm` (IPCB(skb)) flags (`IPSKB_*`):
- `IPSKB_FORWARDED` — packet was forwarded (not local-out).
- `IPSKB_XFRM_TUNNEL_SIZE` — XFRM may grow skb.
- `IPSKB_XFRM_TRANSFORMED` — XFRM transform done; do not redo policy.
- `IPSKB_FRAG_COMPLETE` — fragmentation already done; do not re-fragment.
- `IPSKB_REROUTED` — netfilter rerouted; skip POST_ROUTING.
- `IPSKB_DOREDIRECT` — emit ICMP-redirect.
- `IPSKB_FRAG_PMTU` — original frag came with DF; force DF on refragment.
- `IPSKB_L3SLAVE` — packet entered via L3 slave dev.
- `IPSKB_NOPOLICY` — bypass IPsec policy.
- `IPSKB_MULTIPATH` — multipath info present.
- frag_max_size: u16 — largest pre-existing fragment seen (limits refragment MTU).
- opt: per-skb IP options shadow (ip_options).

REQ-3: `__ip_local_out(net, sk, skb)`:
- iph = ip_hdr(skb).
- IP_INC_STATS(net, IPSTATS_MIB_OUTREQUESTS).
- iph_set_totlen(iph, skb->len).
- ip_send_check(iph) — RFC1071 ones-complement over ihl*4 bytes.
- skb = l3mdev_ip_out(sk, skb) — L3 master device redirect; if NULL return 0.
- skb->protocol = ETH_P_IP.
- return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, net, sk, skb, NULL, dst_dev, dst_output).

REQ-4: `ip_local_out(net, sk, skb)`:
- err = __ip_local_out(net, sk, skb).
- if err == 1: err = dst_output(net, sk, skb).  /* nf_hook returned ACCEPT */
- return err.

REQ-5: `ip_queue_xmit / __ip_queue_xmit(sk, skb, fl, tos)`:
- /* TCP egress hot path */
- rcu_read_lock.
- inet_opt = inet->inet_opt (RCU).
- if !skb_rtable(skb):
  - rt = __sk_dst_check(sk) — per-sock cached route.
  - if !rt: inet_sk_init_flowi4 + rt = ip_route_output_flow(net, fl4, sk); if IS_ERR(rt) → no_route.
  - sk_setup_caps(sk, &rt->dst).
- skb_dst_set_noref(skb, &rt->dst).
- if inet_opt->is_strictroute ∧ rt_uses_gateway: no_route (-EHOSTUNREACH).
- skb_push(skb, sizeof(iphdr) + opt->optlen); skb_reset_network_header.
- Fill iph: version=4, ihl=5, tos, frag_off (IP_DF if ip_dont_fragment), ttl, protocol, saddr/daddr via ip_copy_addrs.
- if opt->optlen: iph->ihl += opt->optlen >> 2; ip_options_build.
- ip_select_ident_segs(net, skb, sk, gso_segs ?: 1).
- skb->priority/mark from sk.
- res = ip_local_out(net, sk, skb).
- rcu_read_unlock; return res.

REQ-6: `ip_build_and_send_pkt(skb, sk, saddr, daddr, opt, tos)` (TCP SYNACK / minimal):
- skb_push for IP header + opt; reset network header.
- Fill iph (version, ihl, tos, ttl via ip_select_ttl, daddr = opt->faddr if SRR else daddr, saddr, protocol).
- Per-skb < IPV4_MIN_MTU ∨ ip_dont_fragment(sk, dst): IP_DF, id=0.
- else: IPPROTO_TCP → random IPID; else __ip_select_ident(net, iph, 1).
- if opt: iph->ihl += opt->optlen>>2; ip_options_build.
- skb->priority/mark.
- return ip_local_out(net, skb->sk, skb).

REQ-7: `ip_output(net, sk, skb)`:
- rcu_read_lock.
- dev = skb_dst_dev_rcu(skb); skb->dev = dev; skb->protocol = ETH_P_IP.
- NF_HOOK_COND(NFPROTO_IPV4, NF_INET_POST_ROUTING, net, sk, skb, indev, dev, ip_finish_output, !(IPCB(skb)->flags & IPSKB_REROUTED)).
- rcu_read_unlock.

REQ-8: `ip_mc_output(net, sk, skb)`:
- /* Multicast: loopback clone for local delivery */
- skb->dev = rt->dst.dev.
- if rt_flags & RTCF_MULTICAST ∧ sk_mc_loop(sk) ∧ (RTCF_LOCAL ∨ !IPSKB_FORWARDED):
  - newskb = skb_clone(skb, GFP_ATOMIC).
  - NF_HOOK POST_ROUTING → ip_mc_finish_output(newskb).
- if ip_hdr(skb)->ttl == 0: drop (multicast TTL guard).
- if RTCF_BROADCAST: also clone+loopback.
- return NF_HOOK_COND POST_ROUTING ip_finish_output.

REQ-9: `ip_finish_output(net, sk, skb)`:
- ret = BPF_CGROUP_RUN_PROG_INET_EGRESS(sk, skb).
- match ret:
  - NET_XMIT_SUCCESS → __ip_finish_output.
  - NET_XMIT_CN → __ip_finish_output ?: ret.
  - default → kfree_skb_reason(BPF_CGROUP_EGRESS).

REQ-10: `__ip_finish_output(net, sk, skb)`:
- #ifdef CONFIG_NETFILTER+CONFIG_XFRM: if skb_dst(skb)->xfrm: IPCB->flags |= IPSKB_REROUTED; return dst_output (XFRM re-entry).
- mtu = ip_skb_dst_mtu(sk, skb).
- if skb_is_gso(skb): return ip_finish_output_gso(net, sk, skb, mtu).
- if skb->len > mtu ∨ IPCB(skb)->frag_max_size: return ip_fragment(net, sk, skb, mtu, ip_finish_output2).
- return ip_finish_output2(net, sk, skb).

REQ-11: `ip_finish_output_gso(net, sk, skb, mtu)`:
- if skb_gso_validate_network_len(skb, mtu): /* seglen ≤ mtu */ return ip_finish_output2.
- /* Slowpath: GSO seglen > mtu */
- features = netif_skb_features(skb).
- BUILD_BUG_ON(sizeof(*IPCB(skb)) > SKB_GSO_CB_OFFSET).
- segs = skb_gso_segment(skb, features & ~NETIF_F_GSO_MASK).
- if IS_ERR_OR_NULL(segs): kfree + -ENOMEM.
- consume_skb(skb).
- skb_list_walk_safe(segs, segs, nskb):
  - skb_mark_not_on_list(segs).
  - err = ip_fragment(net, sk, segs, mtu, ip_finish_output2).
  - if err ∧ ret==0: ret = err.
- return ret.

REQ-12: `ip_finish_output2(net, sk, skb)`:
- dst = skb_dst(skb); rt = dst_rtable(dst); dev = dst_dev(dst); hh_len = LL_RESERVED_SPACE(dev).
- /* SNMP counters */
- if rt_type == RTN_MULTICAST: IP_UPD_PO_STATS(OUTMCAST).
- elif rt_type == RTN_BROADCAST: IP_UPD_PO_STATS(OUTBCAST).
- IP_UPD_PO_STATS(OUT, skb->len).
- /* Reserve link-layer headroom */
- if skb_headroom(skb) < hh_len ∧ dev->header_ops:
  - skb = skb_expand_head(skb, hh_len); if !skb: -ENOMEM.
- /* LWT redirect */
- if lwtunnel_xmit_redirect(dst->lwtstate):
  - res = lwtunnel_xmit(skb); if res != CONTINUE: return res.
- /* Neighbour lookup + emit */
- rcu_read_lock.
- neigh = ip_neigh_for_gw(rt, skb, &is_v6gw).
- if !IS_ERR(neigh):
  - sock_confirm_neigh(skb, neigh).
  - res = neigh_output(neigh, skb, is_v6gw).
  - rcu_read_unlock; return res.
- rcu_read_unlock.
- net_dbg_ratelimited("No header cache and no neighbour!\n").
- kfree_skb_reason(skb, NEIGH_CREATEFAIL); return PTR_ERR(neigh).

REQ-13: `ip_fragment(net, sk, skb, mtu, output)`:
- iph = ip_hdr(skb).
- if !(iph->frag_off & IP_DF): return ip_do_fragment(net, sk, skb, output).
- /* DF set */
- if !skb->ignore_df ∨ (IPCB->frag_max_size ∧ frag_max_size > mtu):
  - IP_INC_STATS(FRAGFAILS).
  - icmp_send(skb, DEST_UNREACH, FRAG_NEEDED, htonl(mtu)).
  - kfree_skb; return -EMSGSIZE.
- return ip_do_fragment(net, sk, skb, output).

REQ-14: `ip_do_fragment(net, sk, skb, output)` (RFC791 8-byte alignment):
- if skb->ip_summed == CHECKSUM_PARTIAL: skb_checksum_help(skb) (fold csum first).
- iph = ip_hdr(skb); rt = skb_rtable(skb).
- mtu = ip_skb_dst_mtu(sk, skb).
- if IPCB->frag_max_size ∧ < mtu: mtu = frag_max_size.
- hlen = iph->ihl * 4; mtu -= hlen.  /* Data space per frag */
- IPCB(skb)->flags |= IPSKB_FRAG_COMPLETE.
- ll_rs = LL_RESERVED_SPACE(dev).
- /* Fast path: skb_shinfo->frag_list already correctly sized */
- if skb_has_frag_list(skb) ∧ geometry-valid:
  - ip_fraglist_init(skb, iph, hlen, &iter) — first frag IP_MF.
  - Loop: for each iter.frag: IPCB(frag)->flags = IPCB(skb)->flags; ip_fraglist_prepare(skb, &iter); first_frag ∧ opt → ip_options_fragment + ip_send_check; output(net, sk, skb); skb_set_delivery_time; if err: break; skb = ip_fraglist_next(&iter).
  - if !err: IPSTATS_MIB_FRAGOKS.
  - else: kfree_skb_list(iter.frag); IPSTATS_MIB_FRAGFAILS.
- /* Slow path: copy */
- ip_frag_init(skb, hlen, ll_rs, mtu, IPSKB_FRAG_PMTU, &state) — state.left, state.ptr, state.offset, state.DF.
- while state.left > 0:
  - first_frag = (state.offset == 0).
  - skb2 = ip_frag_next(skb, &state) — alloc, copy hdr, copy mtu bytes, write iph->frag_off (offset>>3 | IP_MF/IP_DF), tot_len, ip_send_check.
  - ip_frag_ipcb(skb, skb2, first_frag) — copy IPCB flags; if first_frag: ip_options_fragment(from).
  - output(net, sk, skb2); IPSTATS_MIB_FRAGCREATES.
- consume_skb(skb); IPSTATS_MIB_FRAGOKS.

REQ-15: `ip_setup_cork(sk, cork, ipc, rtp)`:
- rt = *rtp; if !rt: -EFAULT.
- cork->fragsize = ip_sk_use_pmtu(sk) ? dst4_mtu(&rt->dst) : READ_ONCE(rt->dst.dev->mtu).
- if !inetdev_valid_mtu(cork->fragsize): -ENETUNREACH.
- if ipc->opt:
  - if !cork->opt: cork->opt = kmalloc(sizeof(ip_options) + 40, sk->sk_allocation); if !cork->opt: -ENOBUFS.
  - memcpy(cork->opt, &opt->opt, sizeof(ip_options) + opt->opt.optlen).
  - cork->flags |= IPCORK_OPT; cork->addr = ipc->addr.
- cork->gso_size = ipc->gso_size.
- cork->dst = &rt->dst; *rtp = NULL.  /* Steal route */
- cork->length = 0; cork->ttl/tos/mark/priority/transmit_time from ipc.
- sock_tx_timestamp(sk, &ipc->sockc, &cork->tx_flags).
- if ipc->sockc.tsflags & SOCKCM_FLAG_TS_OPT_ID: cork->flags |= IPCORK_TS_OPT_ID; cork->ts_opt_id = ipc->sockc.ts_opt_id.
- return 0.

REQ-16: `ip_append_data / __ip_append_data` (per-cork chain build):
- /* Reuses cork on existing sk_write_queue */
- if queue empty: ip_setup_cork(sk, &inet->cork.base, ipc, rtp).
- else: transhdrlen = 0.
- __ip_append_data(sk, fl4, &sk->sk_write_queue, &inet->cork.base, sk_page_frag(sk), getfrag, from, length, transhdrlen, flags).
- Per-__ip_append_data:
  - mtu = cork->gso_size ? IP_MAX_MTU : cork->fragsize; paged = !!cork->gso_size.
  - Reject if cork->length + length > maxnonfragsize - fragheaderlen (EMSGSIZE).
  - Append/extend skb chain; copy user data via getfrag callback into skb linear or page frag.
  - Per-MSG_MORE keep building; otherwise mark queue ready.
  - cork->length += length on success.

REQ-17: `__ip_make_skb(sk, fl4, queue, cork)` (concat + iph build):
- skb = __skb_dequeue(queue); if !skb: return NULL.
- tail_skb = &(skb_shinfo(skb)->frag_list).
- Move skb->data to network header.
- For each tmp_skb in queue: pull network_header_len; chain into frag_list; skb->len/data_len/truesize += tmp_skb->*.
- skb->ignore_df = ip_sk_ignore_df(sk).
- pmtudisc = READ_ONCE(inet->pmtudisc).
- df = (pmtudisc == IP_PMTUDISC_DO ∨ PROBE ∨ (skb->len ≤ dst_mtu ∧ ip_dont_fragment)) ? IP_DF : 0.
- opt = (cork->flags & IPCORK_OPT) ? cork->opt : NULL.
- ttl = cork->ttl ?: (RTN_MULTICAST ? inet->mc_ttl : ip_select_ttl(inet, dst)).
- Build iph: version=4, ihl=5, tos = cork->tos ?: inet->tos, frag_off=df, ttl, protocol = sk->sk_protocol, ip_copy_addrs(iph, fl4), ip_select_ident.
- if opt: iph->ihl += opt->optlen >> 2; ip_options_build(skb, opt, cork->addr, rt).
- skb->priority = cork->priority; skb->mark = cork->mark.
- skb_set_delivery_time / type by_clockid per sk_is_tcp.
- /* Steal route */ cork->dst = NULL; skb_dst_set(skb, &rt->dst).
- if iph->protocol == IPPROTO_ICMP: icmp_out_count(net, icmp_type).
- ip_cork_release(cork).
- return skb.

REQ-18: `ip_push_pending_frames(sk, fl4)`:
- skb = ip_finish_skb(sk, fl4)  /* __ip_make_skb on sk_write_queue */.
- if !skb: return 0.
- return ip_send_skb(sock_net(sk), skb).

REQ-19: `ip_send_skb(net, skb)`:
- err = ip_local_out(net, skb->sk, skb).
- if err: if err > 0: err = net_xmit_errno(err); IPSTATS_MIB_OUTDISCARDS.
- return err.

REQ-20: `ip_flush_pending_frames(sk)` / `__ip_flush_pending_frames(sk, queue, cork)`:
- Drain queue (kfree_skb each).
- ip_cork_release(cork).

REQ-21: `ip_make_skb(sk, fl4, getfrag, from, length, ..., cork, flags)` (no-cork one-shot, UDP fast path):
- if flags & MSG_PROBE: return NULL.
- __skb_queue_head_init(&queue) — local queue, no sk_write_queue.
- cork->flags = 0; cork->addr = 0; cork->opt = NULL.
- ip_setup_cork(sk, cork, ipc, rtp).
- __ip_append_data(sk, fl4, &queue, cork, &current->task_frag, getfrag, from, length, transhdrlen, flags).
- On error: __ip_flush_pending_frames(sk, &queue, cork); return ERR_PTR.
- return __ip_make_skb(sk, fl4, &queue, cork).

REQ-22: `ip_cork_release(cork)`:
- cork->flags &= ~IPCORK_OPT.
- kfree(cork->opt); cork->opt = NULL.
- dst_release(cork->dst); cork->dst = NULL.

REQ-23: `ip_send_check(iph)`:
- iph->check = 0; iph->check = ip_fast_csum((unsigned char *)iph, iph->ihl).

REQ-24: `ip_neigh_for_gw(rt, skb, &is_v6gw)` (RFC1122 § 2.3.2.2 next-hop):
- if rt->rt_uses_gateway: lookup neighbor for gateway address.
- else: lookup neighbor for skb destination address.
- is_v6gw set true if RFC5549 IPv4-over-IPv6-nexthop.

REQ-25: Tunneling integration: GRE/IPIP/VXLAN call ip_local_out / ip_send_skb after setting up inner iph; IPCB(skb)->flags carry IPSKB_FORWARDED / IPSKB_REROUTED / IPSKB_XFRM_TRANSFORMED across encapsulation boundaries.

REQ-26: Per-statistics:
- IPSTATS_MIB_OUTREQUESTS (every __ip_local_out).
- IPSTATS_MIB_OUT, OUTMCAST, OUTBCAST (ip_finish_output2 by rt_type).
- IPSTATS_MIB_FRAGOKS / FRAGFAILS / FRAGCREATES (ip_do_fragment).
- IPSTATS_MIB_OUTNOROUTES (ip_queue_xmit no_route).
- IPSTATS_MIB_OUTDISCARDS (ip_send_skb err).

## Acceptance Criteria

- [ ] AC-1: TCP packet via ip_queue_xmit: route lookup → iph build → ip_local_out → NF LOCAL_OUT → ip_output → POST_ROUTING → ip_finish_output → ip_finish_output2 → neigh_output emits to dev.
- [ ] AC-2: UDP send: ip_make_skb (no-cork) → __ip_append_data → __ip_make_skb → ip_send_skb path; skb has correct iph (ihl=5, ttl, daddr, ip_send_check valid).
- [ ] AC-3: Corked UDP (UDP_CORK enabled): ip_append_data extends sk_write_queue; ip_push_pending_frames glues to single datagram via __ip_make_skb.
- [ ] AC-4: skb->len > mtu ∧ DF clear: ip_do_fragment slow-path emits N fragments, each ≤ mtu, RFC791 8-byte aligned, IP_MF set on non-last, frag_off increments by data_len/8.
- [ ] AC-5: skb->len > mtu ∧ DF set ∧ !ignore_df: ip_fragment returns -EMSGSIZE; icmp_send DEST_UNREACH/FRAG_NEEDED with mtu in info.
- [ ] AC-6: GSO TSO skb with seglen ≤ mtu: ip_finish_output_gso → ip_finish_output2 (fast path, no segmentation).
- [ ] AC-7: GSO skb with seglen > mtu: skb_gso_segment → list of small skbs → ip_fragment each → ip_finish_output2.
- [ ] AC-8: Multicast skb: ip_mc_output clones for loopback + POST_ROUTING; ttl=0 dropped.
- [ ] AC-9: BPF cgroup egress: BPF_CGROUP_RUN_PROG_INET_EGRESS == NET_XMIT_DROP → drop+kfree_skb_reason.
- [ ] AC-10: ip_finish_output2: hh_len headroom expand; neigh lookup via ip_neigh_for_gw; neigh_output emits or returns PTR_ERR(neigh) on lookup failure.
- [ ] AC-11: IPCB IPSKB_REROUTED skip POST_ROUTING NF on re-entry.
- [ ] AC-12: XFRM transform on dst: __ip_finish_output → IPSKB_REROUTED + dst_output (XFRM re-entry).
- [ ] AC-13: ip_setup_cork: stolen rt; *rtp set NULL; cork->dst owns ref; ip_cork_release dst_release on flush.
- [ ] AC-14: ip_options_build per cork: ihl bumped (optlen >> 2); options bytes inserted.
- [ ] AC-15: ip_select_ident: per-flow IPID jenkins; ip_dont_fragment ∨ skb→len ≤ MIN_MTU forces id=0+IP_DF in ip_build_and_send_pkt.

## Architecture

```
struct InetCork {
  flags: u8,                 // IPCORK_OPT | IPCORK_TS_OPT_ID | IPCORK_ALLFRAG
  gso_size: u16,
  fragsize: u32,
  length: u32,
  dst: Option<*Dst>,
  addr: __be32,
  opt: Option<Box<IpOptions>>,
  ttl: u8,
  tos: i8,
  mark: u32,
  priority: u32,
  transmit_time: u64,
  tx_flags: u16,
  ts_opt_id: u32,
}

struct InetSkbParm {           // IPCB(skb)
  opt: IpOptionsShadow,
  flags: u16,                  // IPSKB_*
  frag_max_size: u16,
}
```

`IpOutput::local_out_inner(net, sk, skb) -> i32`:
1. iph = ip_hdr(skb).
2. IP_INC_STATS(net, IPSTATS_MIB_OUTREQUESTS).
3. iph_set_totlen(iph, skb->len).
4. IpOutput::send_check(iph) — 16-bit ones-complement over ihl*4.
5. skb = l3mdev_ip_out(sk, skb); if skb.is_none(): return 0.
6. skb->protocol = ETH_P_IP.
7. return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT, net, sk, skb, NULL, dst_dev, dst_output).

`IpOutput::local_out(net, sk, skb) -> i32`:
1. err = IpOutput::local_out_inner(net, sk, skb).
2. if err == 1: err = dst_output(net, sk, skb).
3. return err.

`IpOutput::queue_xmit_inner(sk, skb, fl, tos) -> i32`:
1. rcu_read_lock.
2. inet_opt = rcu_dereference(inet->inet_opt).
3. fl4 = &fl.u.ip4.
4. rt = skb_rtable(skb); if rt != null: goto packet_routed.
5. rt = dst_rtable(__sk_dst_check(sk, 0)).
6. if !rt:
   - inet_sk_init_flowi4(inet, fl4).
   - fl4->flowi4_dscp = inet_dsfield_to_dscp(tos).
   - rt = ip_route_output_flow(net, fl4, sk); if IS_ERR(rt): goto no_route.
   - sk_setup_caps(sk, &rt->dst).
7. skb_dst_set_noref(skb, &rt->dst).
8. /* packet_routed */
9. if inet_opt && inet_opt->opt.is_strictroute && rt->rt_uses_gateway: goto no_route.
10. skb_push(skb, sizeof(iphdr) + inet_opt.optlen).
11. skb_reset_network_header(skb); iph = ip_hdr(skb).
12. write iph: tos+version+ihl=5+ttl+protocol+ip_copy_addrs(iph, fl4).
13. iph->frag_off = (ip_dont_fragment(sk, &rt->dst) && !skb->ignore_df) ? IP_DF : 0.
14. if inet_opt && inet_opt.optlen: iph->ihl += optlen >> 2; ip_options_build.
15. ip_select_ident_segs(net, skb, sk, gso_segs ?: 1).
16. skb->priority/mark from sk.
17. res = IpOutput::local_out(net, sk, skb).
18. rcu_read_unlock; return res.

`IpOutput::finish_output2(net, sk, skb) -> i32`:
1. dst = skb_dst(skb); rt = dst_rtable(dst); dev = dst_dev(dst); hh_len = LL_RESERVED_SPACE(dev).
2. /* SNMP */
3. match rt->rt_type { RTN_MULTICAST => IP_UPD_PO_STATS(OUTMCAST), RTN_BROADCAST => IP_UPD_PO_STATS(OUTBCAST), _ => {} }.
4. IP_UPD_PO_STATS(net, IPSTATS_MIB_OUT, skb->len).
5. if skb_headroom(skb) < hh_len ∧ dev->header_ops:
   - skb = skb_expand_head(skb, hh_len); if skb.is_none(): return -ENOMEM.
6. if lwtunnel_xmit_redirect(dst->lwtstate):
   - res = lwtunnel_xmit(skb); if res != CONTINUE: return res.
7. rcu_read_lock.
8. neigh = Neigh::for_gw(rt, skb, &is_v6gw).
9. if neigh.is_ok():
   - sock_confirm_neigh(skb, neigh).
   - res = neigh_output(neigh, skb, is_v6gw).
   - rcu_read_unlock; return res.
10. rcu_read_unlock.
11. net_dbg_ratelimited("No header cache and no neighbour!").
12. kfree_skb_reason(skb, NEIGH_CREATEFAIL); return PTR_ERR(neigh).

`IpOutput::do_fragment(net, sk, skb, output) -> i32`:
1. if skb->ip_summed == CHECKSUM_PARTIAL: skb_checksum_help(skb).
2. iph = ip_hdr(skb); rt = skb_rtable(skb).
3. mtu = ip_skb_dst_mtu(sk, skb).
4. if IPCB(skb)->frag_max_size ∧ < mtu: mtu = IPCB(skb)->frag_max_size.
5. hlen = iph->ihl * 4; mtu -= hlen.
6. IPCB(skb)->flags |= IPSKB_FRAG_COMPLETE.
7. ll_rs = LL_RESERVED_SPACE(dev).
8. if skb_has_frag_list(skb) ∧ geometry-valid:
   - ip_fraglist_init(skb, iph, hlen, &iter).
   - loop: if iter.frag: IPCB(iter.frag) = IPCB(skb); ip_fraglist_prepare(skb, &iter); first_frag ∧ opt → ip_options_fragment + ip_send_check; skb_set_delivery_time(skb); err = output(net, sk, skb); if err ∨ !iter.frag: break; skb = ip_fraglist_next(&iter).
   - if !err: IPSTATS_MIB_FRAGOKS else kfree_skb_list(iter.frag); IPSTATS_MIB_FRAGFAILS.
9. /* slow_path */
10. ip_frag_init(skb, hlen, ll_rs, mtu, IPCB(skb)->flags & IPSKB_FRAG_PMTU, &state).
11. while state.left > 0:
    - first_frag = state.offset == 0.
    - skb2 = ip_frag_next(skb, &state); if IS_ERR: fail.
    - ip_frag_ipcb(skb, skb2, first_frag).
    - skb_set_delivery_time(skb2, tstamp, tstamp_type).
    - err = output(net, sk, skb2); if err: fail.
    - IPSTATS_MIB_FRAGCREATES.
12. consume_skb(skb); IPSTATS_MIB_FRAGOKS.

`IpOutput::setup_cork(sk, cork, ipc, &mut rtp) -> i32`:
1. rt = *rtp; if rt.is_none(): return -EFAULT.
2. cork->fragsize = ip_sk_use_pmtu(sk) ? dst4_mtu(&rt->dst) : rt->dst.dev->mtu.
3. if !inetdev_valid_mtu(cork->fragsize): return -ENETUNREACH.
4. if ipc->opt:
   - if !cork->opt: cork->opt = kmalloc(ip_options + 40); if !cork->opt: -ENOBUFS.
   - memcpy(cork->opt, &opt->opt, ip_options + opt->optlen).
   - cork->flags |= IPCORK_OPT; cork->addr = ipc->addr.
5. cork->gso_size = ipc->gso_size.
6. cork->dst = &rt->dst; *rtp = NULL.
7. cork->length = 0; cork->{ttl,tos,mark,priority,transmit_time} = ipc.*.
8. cork->tx_flags = 0; sock_tx_timestamp(sk, &ipc->sockc, &cork->tx_flags).
9. if ipc->sockc.tsflags & SOCKCM_FLAG_TS_OPT_ID: cork->flags |= IPCORK_TS_OPT_ID; cork->ts_opt_id = ipc->sockc.ts_opt_id.
10. return 0.

`IpOutput::make_skb_inner(sk, fl4, queue, cork) -> Option<*sk_buff>`:
1. skb = __skb_dequeue(queue); if !skb: return None.
2. tail_skb = &skb_shinfo(skb)->frag_list.
3. if skb->data < skb_network_header(skb): __skb_pull(skb, skb_network_offset(skb)).
4. while tmp = __skb_dequeue(queue):
   - __skb_pull(tmp, skb_network_header_len(skb)).
   - *tail_skb = tmp; tail_skb = &tmp->next.
   - skb->len/data_len/truesize += tmp->*.
   - tmp->destructor = None; tmp->sk = None.
5. skb->ignore_df = ip_sk_ignore_df(sk).
6. pmtudisc = inet->pmtudisc; df = (DO/PROBE ∨ (len ≤ mtu ∧ ip_dont_fragment)) ? IP_DF : 0.
7. opt = (cork->flags & IPCORK_OPT) ? cork->opt : None.
8. ttl = cork->ttl ?: (RTN_MULTICAST ? inet->mc_ttl : ip_select_ttl).
9. Build iph: version=4, ihl=5, tos = cork->tos ?: inet->tos, frag_off=df, ttl, protocol = sk->sk_protocol, ip_copy_addrs(iph, fl4), ip_select_ident.
10. if opt: iph->ihl += optlen >> 2; ip_options_build(skb, opt, cork->addr, rt).
11. skb->priority = cork->priority; skb->mark = cork->mark.
12. skb_set_delivery_time / type_by_clockid per sk_is_tcp.
13. cork->dst = None; skb_dst_set(skb, &rt->dst).  /* Steal rt */
14. if iph->protocol == IPPROTO_ICMP: icmp_out_count(net, icmp_type).
15. ip_cork_release(cork).
16. return Some(skb).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cork_dst_refcount_balanced` | INVARIANT | per-setup_cork → ip_cork_release: dst held + dst_release; no double-free. |
| `cork_opt_freed_on_release` | INVARIANT | per-ip_cork_release: kfree(cork->opt); cork->opt set NULL. |
| `frag_offset_8byte_aligned` | INVARIANT | per-ip_frag_next: state.offset % 8 == 0 on each iteration. |
| `iph_totlen_consistent` | INVARIANT | per-__ip_local_out: iph->tot_len == skb->len after iph_set_totlen. |
| `iph_csum_valid` | INVARIANT | per-ip_send_check post: ones-complement over ihl*4 == 0. |
| `IPSKB_FRAG_COMPLETE_set` | INVARIANT | per-ip_do_fragment: IPCB->flags |= IPSKB_FRAG_COMPLETE before frag emission. |
| `neigh_release_on_path` | INVARIANT | per-ip_finish_output2: rcu_read_lock around neigh deref; no UAF. |
| `df_set_min_mtu` | INVARIANT | per-ip_build_and_send_pkt: skb→len ≤ IPV4_MIN_MTU ⟹ IP_DF set + id=0. |
| `cork_length_monotonic` | INVARIANT | per-__ip_append_data success: cork->length += length. |
| `gso_segs_capacity_check` | INVARIANT | per-ip_finish_output_gso: BUILD_BUG_ON(sizeof IPCB > SKB_GSO_CB_OFFSET). |

### Layer 2: TLA+

`net/ipv4/ip-output.tla`:
- Per-skb egress: queue_xmit → local_out → NF_LOCAL_OUT → dst_output → ip_output → NF_POST_ROUTING → finish_output → (gso|fragment|direct) → finish_output2 → neigh_output.
- Per-cork: setup_cork → append_data (queue grows) → push_pending_frames → make_skb (concat) → send_skb → local_out.
- Properties:
  - `safety_no_packet_emitted_without_NF_post_routing` — per-skb: arrives at dev only after NF POST_ROUTING (unless IPSKB_REROUTED already consumed).
  - `safety_DF_fragment_emits_icmp` — per-skb DF + oversize + !ignore_df ⟹ icmp_send FRAG_NEEDED + -EMSGSIZE.
  - `safety_cork_length_invariant` — per-sock: queue empty ⟺ cork released after push/flush.
  - `safety_frag_alignment_RFC791` — per-fragment: offset % 8 == 0 ∧ each non-last has IP_MF.
  - `liveness_per_skb_eventually_emitted_or_dropped` — per-skb: terminates (no leak).
  - `safety_GSO_seglen_le_mtu_fastpath` — per-skb skb_gso_validate_network_len ⟹ direct finish_output2.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IpOutput::local_out` post: dst_output called ⟺ NF returned ACCEPT (err==1) | `IpOutput::local_out` |
| `IpOutput::queue_xmit_inner` post: rt set (skb_dst_set_noref) ∨ -EHOSTUNREACH | `IpOutput::queue_xmit_inner` |
| `IpOutput::finish_output_inner` post: routed to gso ⟺ skb_is_gso; routed to fragment ⟺ len > mtu ∨ frag_max_size | `IpOutput::finish_output_inner` |
| `IpOutput::do_fragment` post: emitted_bytes == skb_len - hlen + N*hlen; IPSTATS counters bumped | `IpOutput::do_fragment` |
| `IpOutput::fragment` post: DF ∧ oversize ∧ !ignore_df ⟹ ICMP emit + -EMSGSIZE | `IpOutput::fragment` |
| `IpOutput::make_skb_inner` post: cork->dst stolen (NULL after); skb_dst_set with rt | `IpOutput::make_skb_inner` |
| `IpOutput::setup_cork` post: cork->fragsize = pmtu ∨ dev_mtu; cork->length = 0 | `IpOutput::setup_cork` |
| `IpOutput::send_skb` post: IPSTATS_MIB_OUTDISCARDS bumped on err | `IpOutput::send_skb` |

### Layer 4: Verus/Creusot functional

`Per-skb egress` semantic equivalence to RFC791 § 2.3 + RFC1122 § 3.3:
- header build (version=4, ihl, tot_len, id, frag_off, ttl, protocol, src, dst, options, checksum).
- fragmentation (offset in 8-byte units, MF/DF bits, options on first frag only via ip_options_fragment).
- next-hop resolution (neighbor cache, gateway override per rt_uses_gateway).
`Per-cork` semantic equivalence to UDP_CORK / IP_PMTUDISC interaction model (Documentation/networking/ip-sysctl.rst).
`Per-GSO/TSO` semantic equivalence to NETIF_F_GSO + skb_gso_segment per Documentation/networking/segmentation-offloads.rst.

## Hardening

(Inherits row-1 features from `net/ipv4/00-overview.md` § Hardening.)

IPv4 output reinforcement:

- **Per-cork dst_release on flush / release** — defense against per-route-dst leak.
- **Per-cork opt kfree on release** — defense against per-options memory leak.
- **Per-cork length monotonic + maxnonfragsize check (EMSGSIZE)** — defense against per-cork oversize datagram.
- **Per-DF bit + !ignore_df ⟹ ICMP FRAG_NEEDED + drop** — defense against per-PMTU-violation forwarding.
- **Per-IPSKB_FRAG_COMPLETE re-entry guard** — defense against per-double-fragmentation.
- **Per-skb_expand_head hh_len reserved** — defense against per-link-layer header-room underflow.
- **Per-skb_has_frag_list geometry validation in ip_do_fragment** — defense against per-malformed fraglist crash.
- **Per-RFC791 8-byte alignment in ip_frag_next (len &= ~7)** — defense against per-non-aligned fragment.
- **Per-IPSKB_REROUTED skips POST_ROUTING** — defense against per-NF-loop after rerouting.
- **Per-rcu_read_lock around inet_opt + neigh deref** — defense against per-options/neigh UAF.
- **Per-ip_select_ident_segs anti-prediction (jenkins+SipHash)** — defense against per-IPID prediction (CVE class).
- **Per-IP_PMTUDISC_PROBE forces IP_DF** — defense against per-PMTU-discovery spoofing.
- **Per-skb->ignore_df honored** — defense against per-conntrack-fragment reassembly mismatch.
- **Per-BPF_CGROUP_EGRESS drop path zeroes skb** — defense against per-egress-bypass.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/ipv4/ip_input.c receive path (covered in `ip_input.md` Tier-3)
- net/ipv4/ip_fragment.c reassembly (covered in `ip-fragment.md` Tier-3)
- net/ipv4/route.c FIB / dst (covered in `route.md` Tier-3)
- net/ipv4/icmp.c ICMP emit (covered in `icmp.md` Tier-3)
- net/core/dev.c qdisc + dev_queue_xmit (covered separately)
- net/core/neighbour.c ARP / neighbour state machine (covered separately)
- net/xfrm/* IPsec transform (covered separately)
- Implementation code
