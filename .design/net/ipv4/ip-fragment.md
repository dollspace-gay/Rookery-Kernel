# Tier-3: net/ipv4/ip_fragment.c — IPv4 reassembly (ingress fragment queue + ip_defrag)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/ipv4/00-overview.md
upstream-paths:
  - net/ipv4/ip_fragment.c (~753 lines)
  - include/net/inet_frag.h (struct inet_frag_queue, struct inet_frags, struct fqdir)
  - include/net/ip.h (IP_FRAG_TIME, struct frag_v4_compare_key, ip_defrag_user_in_between)
  - net/ipv4/inet_fragment.c (shared LRU + rhashtable framework)
-->

## Summary

`net/ipv4/ip_fragment.c` is the IPv4 **reassembly** side of fragmentation. Per-fragment ingress (called from `ip_local_deliver` once iph→frag_off has IP_MF or non-zero offset, plus from netfilter `conntrack_defrag` and from raw socket / af_packet via `ip_check_defrag`): `ip_defrag(net, skb, user)` → `ip_find(net, iph, user, vif)` looks up `struct ipq` in per-netns `net->ipv4.fqdir` rhashtable keyed by `frag_v4_compare_key {saddr, daddr, user, vif, id, protocol}` → `ip_frag_queue(qp, skb, &refs)` inserts the fragment into the queue's rbtree, tracks `INET_FRAG_FIRST_IN`/`INET_FRAG_LAST_IN`, RFC791 8-byte alignment (end &= ~7), ECN merge per RFC3168, max_df_size tracking → when `qp->q.meat == qp->q.len` (all fragments arrived) `ip_frag_reasm(qp, skb, prev_tail, dev, refs)` runs: `inet_frag_reasm_prepare` + `inet_frag_reasm_finish` coalesce fragments, total len ≤ 65535 check, reset iph→tot_len/tos/frag_off (set IP_DF if max_df_size == max_size), `ip_send_check`, `IPSTATS_MIB_REASMOKS`. Per-timeout (`IP_FRAG_TIME = 30*HZ`): `ip_expire` timer fires → `inet_frag_kill` evicts queue → emits ICMP `TIME_EXCEEDED/FRAG_REASM_TIMEDOUT` for non-conntrack users. Per-pressure: `inet_frag_evictor` LRU evicts oldest queues when fqdir memory > `high_thresh` (default 4MB), prunes down to `low_thresh` (3MB). Per-netns `ipv4_frags_init_net` provisions `fqdir` with `high_thresh`/`low_thresh`/`timeout`/`max_dist=64` sysctls. Critical for: NFS-over-UDP, DNS-over-UDP > 1500B, OpenVPN large datagrams, IPsec encapsulated fragments, conntrack-NAT for fragmented flows.

This Tier-3 covers `net/ipv4/ip_fragment.c` (~753 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ipq` | per-queue IPv4 reassembly state | `Ipq` |
| `struct inet_frag_queue` | shared base (key, rbtree, timer, meat, len) | shared `InetFragQueue` |
| `struct frag_v4_compare_key` | per-queue lookup key (saddr,daddr,user,vif,id,protocol) | `FragV4CompareKey` |
| `struct fqdir` | per-netns frag dir (high/low_thresh, timeout, max_dist, mem) | shared `Fqdir` |
| `struct inet_frags ip4_frags` | per-family ops (constructor, destructor, frag_expire, rhash_params) | `IpFragment::IP4_FRAGS` |
| `ip4_frag_init(q, key)` | per-queue constructor (set key, alloc inet_peer if max_dist) | `IpFragment::frag_init` |
| `ip4_frag_free(q)` | per-queue destructor (release inet_peer) | `IpFragment::frag_free` |
| `ip4_frag_ecn(tos)` | per-frag ECN bitmask (RFC3168) | `IpFragment::frag_ecn` |
| `ip_expire(timer)` | per-queue timeout handler + ICMP TIME_EXCEEDED | `IpFragment::expire` |
| `frag_expire_skip_icmp(user)` | per-user skip ICMP (conntrack/af_packet) | `IpFragment::expire_skip_icmp` |
| `ip_find(net, iph, user, vif)` | per-netns hashtable lookup/create ipq | `IpFragment::find` |
| `ip_frag_too_far(qp)` | per-peer reorder distance check (max_dist) | `IpFragment::frag_too_far` |
| `ip_frag_reinit(qp)` | per-queue reset after too_far / timeout | `IpFragment::frag_reinit` |
| `ip_frag_queue(qp, skb, &refs)` | per-skb insert into queue (offset, MF, dup check) | `IpFragment::frag_queue` |
| `ip_frag_coalesce_ok(qp)` | per-user coalesce-eligibility (LOCAL_DELIVER only) | `IpFragment::coalesce_ok` |
| `ip_frag_reasm(qp, skb, prev_tail, dev, refs)` | per-queue final coalesce → datagram | `IpFragment::frag_reasm` |
| `ip_defrag(net, skb, user)` | per-skb entry (lookup + queue) | `IpFragment::defrag` |
| `ip_check_defrag(net, skb, user)` | per-skb conditional defrag (raw/af_packet) | `IpFragment::check_defrag` |
| `ipv4_frags_init_net(net)` | per-netns fqdir init + sysctls | `IpFragment::init_net` |
| `ipv4_frags_pre_exit_net(net)` | per-netns fqdir_pre_exit (mark dead) | `IpFragment::pre_exit_net` |
| `ipv4_frags_exit_net(net)` | per-netns fqdir_exit (free) | `IpFragment::exit_net` |
| `ip4_frags_ns_ctl_register(net)` | per-netns sysctl table | `IpFragment::ns_ctl_register` |
| `ip4_key_hashfn / ip4_obj_hashfn / ip4_obj_cmpfn` | rhashtable jhash2 ops | `IpFragment::HASH_OPS` |
| `ip_frag_ecn_table[]` | per-ECN-mask combine table (RFC3168) | shared `Inet::FRAG_ECN_TABLE` |
| `IPSTATS_MIB_REASMREQDS / REASMOKS / REASMFAILS / REASMTIMEOUT / REASM_OVERLAPS` | per-event counters | shared `IpStats` |
| `IPCB(skb)->frag_max_size` | per-skb reassembly mark | shared `IpCb` |
| `IPSKB_FRAG_PMTU` | per-skb DF-on-refragment hint | shared `IpCb` |
| `IPCB(skb)->flags IPSKB_FRAG_COMPLETE` | per-skb skip-too-far during xmit-side coalesce | shared `IpCb` |
| `ip_defrag_user_in_between(user, lo, hi)` | per-user range check (conntrack helpers) | shared `Inet::user_in_between` |

## Compatibility contract

REQ-1: `struct ipq` (per-queue, embeds shared base):
- q: InetFragQueue — shared (key, rbtree of skbs, fragments_tail, timer, flags, meat, len, max_size, lock, stamp, ...).
- ecn: u8 — RFC3168 ECN bitmask accumulator (`ip_frag_ecn_table` later resolves).
- max_df_size: u16 — largest fragment seen with IP_DF set (used at reasm to force IP_DF on output).
- iif: i32 — input ifindex (for skb->dev recovery in ICMP TIME_EXCEEDED).
- rid: u32 — peer's run id at queue creation (for max_dist tracking).
- peer: *InetPeer — peer entry (NULL unless fqdir->max_dist > 0).

REQ-2: `struct frag_v4_compare_key` (rhashtable key, 32 bytes / aligned-u32):
- saddr: __be32.
- daddr: __be32.
- user: u32 — `enum ip_defrag_users` (LOCAL_DELIVER, CALL_RA_CHAIN, CONNTRACK_IN, CONNTRACK_OUT, CONNTRACK_BRIDGE_IN, VS_IN, VS_OUT, VS_FWD, AF_PACKET, MACVLAN, ...).
- vif: i32 — L3 master device index (l3mdev VRF separation).
- id: __be16 — IPv4 ID field.
- protocol: u8 — IP protocol.

REQ-3: `struct fqdir` per-netns init (`ipv4_frags_init_net`):
- fqdir_init(&net->ipv4.fqdir, &ip4_frags, net).
- high_thresh = 4 * 1024 * 1024 (4 MB).
- low_thresh = 3 * 1024 * 1024 (3 MB).
- timeout = IP_FRAG_TIME (30 * HZ).
- max_dist = 64 (RFC1858 anti-overlap mitigation distance).
- ip4_frags_ns_ctl_register(net) — exposes `ipfrag_high_thresh`, `ipfrag_low_thresh`, `ipfrag_time`, `ipfrag_max_dist` sysctls.

REQ-4: `ip_defrag(net, skb, user)`:
- __IP_INC_STATS(net, IPSTATS_MIB_REASMREQDS).
- rcu_read_lock.
- dev = skb->dev ?: skb_dst_dev_rcu(skb).
- vif = l3mdev_master_ifindex_rcu(dev).
- qp = ip_find(net, ip_hdr(skb), user, vif).
- if qp:
  - refs = 0.
  - spin_lock(&qp->q.lock).
  - ret = ip_frag_queue(qp, skb, &refs).
  - spin_unlock(&qp->q.lock).
  - rcu_read_unlock.
  - inet_frag_putn(&qp->q, refs).
  - return ret.
- rcu_read_unlock.
- __IP_INC_STATS(net, IPSTATS_MIB_REASMFAILS).
- kfree_skb(skb).
- return -ENOMEM.

REQ-5: `ip_find(net, iph, user, vif)`:
- key = { iph->saddr, iph->daddr, user, vif, iph->id, iph->protocol }.
- q = inet_frag_find(net->ipv4.fqdir, &key) — rhashtable lookup; if not present, allocates via inet_frags->constructor = ip4_frag_init (which sets q->key.v4 = key and conditionally takes inet_peer ref for max_dist tracking); LRU bookkeeping via `inet_frag_lru_move` (shared inet_fragment.c).
- if !q: return NULL.
- return container_of(q, struct ipq, q).

REQ-6: `ip4_frag_init(q, key)`:
- qp = container_of(q, struct ipq, q).
- net = q->fqdir->net.
- q->key.v4 = *key.
- qp->ecn = 0.
- if q->fqdir->max_dist:
  - rcu_read_lock.
  - p = inet_getpeer_v4(net->ipv4.peers, key->saddr, key->vif).
  - if p ∧ !refcount_inc_not_zero(&p->refcnt): p = NULL.
  - rcu_read_unlock.
- qp->peer = p.

REQ-7: `ip4_frag_free(q)`:
- qp = container_of(q, struct ipq, q).
- if qp->peer: inet_putpeer(qp->peer).

REQ-8: `ip_frag_queue(qp, skb, &refs)` (RFC791 8-byte alignment + dup/overlap detection):
- if qp->q.flags & INET_FRAG_COMPLETE: SKB_DR_SET(DUP_FRAG); goto err.
- if !(IPCB(skb)->flags & IPSKB_FRAG_COMPLETE) ∧ ip_frag_too_far(qp) ∧ ip_frag_reinit(qp):
  - inet_frag_kill(&qp->q, refs); goto err.
- ecn = ip4_frag_ecn(ip_hdr(skb)->tos).
- offset = ntohs(ip_hdr(skb)->frag_off); flags = offset & ~IP_OFFSET; offset &= IP_OFFSET; offset <<= 3.  /* 8-byte units → bytes */
- ihl = ip_hdrlen(skb).
- end = offset + skb->len - skb_network_offset(skb) - ihl.
- /* Last fragment? */
- if (flags & IP_MF) == 0:
  - if end < qp->q.len ∨ (LAST_IN ∧ end != qp->q.len): goto discard_qp.  /* Corrupt */
  - qp->q.flags |= INET_FRAG_LAST_IN; qp->q.len = end.
- else:
  - if end & 7: end &= ~7; if skb->ip_summed != CHECKSUM_UNNECESSARY: skb->ip_summed = CHECKSUM_NONE.  /* RFC791 8-byte alignment for non-last */
  - if end > qp->q.len: if LAST_IN: goto discard_qp; qp->q.len = end.
- if end == offset: goto discard_qp.
- if !pskb_pull(skb, skb_network_offset(skb) + ihl): goto discard_qp.
- err = pskb_trim_rcsum(skb, end - offset); if err: goto discard_qp.
- prev_tail = qp->q.fragments_tail.
- err = inet_frag_queue_insert(&qp->q, skb, offset, end).  /* rbtree insert, dup/overlap detection */
- if err: goto insert_error.
- if dev: qp->iif = dev->ifindex.
- qp->q.stamp = skb->tstamp; qp->q.tstamp_type = skb->tstamp_type; qp->q.meat += skb->len.
- qp->ecn |= ecn.
- add_frag_mem_limit(qp->q.fqdir, skb->truesize).  /* Memory accounting */
- if offset == 0: qp->q.flags |= INET_FRAG_FIRST_IN.
- fragsize = skb->len + ihl.
- if fragsize > qp->q.max_size: qp->q.max_size = fragsize.
- if iph->frag_off & IP_DF ∧ fragsize > qp->max_df_size: qp->max_df_size = fragsize.
- /* All fragments arrived? */
- if qp->q.flags == (FIRST_IN | LAST_IN) ∧ qp->q.meat == qp->q.len:
  - orefdst = skb->_skb_refdst; skb->_skb_refdst = 0.
  - err = ip_frag_reasm(qp, skb, prev_tail, dev, refs).
  - skb->_skb_refdst = orefdst.
  - if err: inet_frag_kill(&qp->q, refs).
  - return err.
- skb_dst_drop(skb); skb_orphan(skb).
- return -EINPROGRESS.
- /* insert_error / discard_qp / err */
- if err == IPFRAG_DUP: SKB_DR_SET(DUP_FRAG); err = -EINVAL.
- else: __IP_INC_STATS(IPSTATS_MIB_REASM_OVERLAPS).
- inet_frag_kill(&qp->q, refs); __IP_INC_STATS(REASMFAILS).
- kfree_skb_reason(skb, reason); return err.

REQ-9: `ip_frag_too_far(qp)` (RFC1858 / max_dist):
- peer = qp->peer; max = qp->q.fqdir->max_dist.
- if !peer ∨ !max: return 0.
- start = qp->rid; end = atomic_inc_return(&peer->rid); qp->rid = end.
- rc = qp->q.fragments_tail ∧ (end - start) > max.
- if rc: __IP_INC_STATS(IPSTATS_MIB_REASMFAILS).
- return rc.

REQ-10: `ip_frag_reinit(qp)`:
- if !mod_timer_pending(&qp->q.timer, jiffies + fqdir->timeout): return -ETIMEDOUT.
- inet_frag_queue_flush(&qp->q, SKB_DROP_REASON_FRAG_TOO_FAR).
- Zero q.flags/len/meat/rb_fragments/fragments_tail/last_run_head; qp->iif=0; qp->ecn=0.
- return 0.

REQ-11: `ip_frag_reasm(qp, skb, prev_tail, dev, refs)`:
- inet_frag_kill(&qp->q, refs).  /* Take off rhashtable + LRU */
- ecn = ip_frag_ecn_table[qp->ecn].
- if ecn == 0xff: err = -EINVAL; goto out_fail.
- reasm_data = inet_frag_reasm_prepare(&qp->q, skb, prev_tail); if !reasm_data: goto out_nomem.
- len = ip_hdrlen(skb) + qp->q.len.
- if len > 65535: err = -E2BIG; goto out_oversize.  /* RFC791 max */
- inet_frag_reasm_finish(&qp->q, skb, reasm_data, ip_frag_coalesce_ok(qp)).  /* True only for LOCAL_DELIVER user */
- skb->dev = dev.
- IPCB(skb)->frag_max_size = max(qp->max_df_size, qp->q.max_size).
- iph = ip_hdr(skb); iph->tot_len = htons(len); iph->tos |= ecn.
- /* IP_DF / IPSKB_FRAG_PMTU on refragment */
- if qp->max_df_size == qp->q.max_size: IPCB(skb)->flags |= IPSKB_FRAG_PMTU; iph->frag_off = htons(IP_DF).
- else: iph->frag_off = 0.
- ip_send_check(iph).
- __IP_INC_STATS(REASMOKS).
- Zero qp->q.rb_fragments / fragments_tail / last_run_head.
- return 0.
- /* out_nomem / out_oversize / out_fail */
- __IP_INC_STATS(REASMFAILS); return err.

REQ-12: `ip_frag_coalesce_ok(qp)`:
- return qp->q.key.v4.user == IP_DEFRAG_LOCAL_DELIVER.  /* Other users (conntrack, af_packet) keep fragments separable */

REQ-13: `ip_expire(timer)`:
- frag = container_of(timer); qp = container_of(frag, ipq, q).
- spin_lock(&qp->q.lock).
- if INET_FRAG_COMPLETE: out.
- qp->q.flags |= INET_FRAG_DROP; inet_frag_kill(&qp->q, &refs).
- if fqdir dead: inet_frag_queue_flush(0); out.
- __IP_INC_STATS(REASMFAILS); __IP_INC_STATS(REASMTIMEOUT).
- if !(FIRST_IN): out.
- head = inet_frag_pull_head(&qp->q).
- if !head: out.
- head->dev = dev_get_by_index_rcu(net, qp->iif).
- if !head->dev: out.
- reason = ip_route_input_noref(head, ...); if reason: out.
- if frag_expire_skip_icmp(qp->q.key.v4.user) ∧ rt_type != RTN_LOCAL: out (no ICMP for forwarded path).
- spin_unlock(&qp->q.lock).
- icmp_send(head, ICMP_TIME_EXCEEDED, ICMP_EXC_FRAGTIME, 0).
- /* out paths free head + inet_frag_putn(&qp->q, refs). */

REQ-14: `frag_expire_skip_icmp(user)`:
- return user == IP_DEFRAG_AF_PACKET ∨ ip_defrag_user_in_between(user, CONNTRACK_IN, __CONNTRACK_IN_END) ∨ ip_defrag_user_in_between(user, CONNTRACK_BRIDGE_IN, __CONNTRACK_BRIDGE_IN).

REQ-15: rhashtable params (`ip4_rhash_params`):
- head_offset = offsetof(InetFragQueue, node).
- key_offset = offsetof(InetFragQueue, key).
- key_len = sizeof(FragV4CompareKey) (24 bytes).
- hashfn = ip4_key_hashfn → jhash2(data, key_len/4, seed).
- obj_hashfn = ip4_obj_hashfn → jhash2 of fq->key.v4.
- obj_cmpfn = ip4_obj_cmpfn → memcmp(&fq->key, key, sizeof key).
- automatic_shrinking = true.

REQ-16: LRU eviction (shared `inet_frag_evictor` in inet_fragment.c, triggered by `add_frag_mem_limit`):
- When fqdir->mem.count > fqdir->high_thresh: evict oldest queues from LRU list (within netns), free skbs, drop ref → constructor's destructor (`ip4_frag_free`) releases inet_peer.
- Stop when mem.count ≤ fqdir->low_thresh.
- Per-IPSTATS_MIB_REASMFAILS bump on eviction.

REQ-17: `ip_check_defrag(net, skb, user)` (af_packet / raw):
- If skb is fragment (iph->frag_off & (IP_MF | IP_OFFSET) != 0):
  - skb_share_check + clone-on-cow.
  - return ip_defrag(net, skb, user) ? NULL : reassembled-skb.
- Else: return skb (pass through).

REQ-18: Per-netns lifecycle:
- pernet_operations: { init = ipv4_frags_init_net, pre_exit = ipv4_frags_pre_exit_net, exit = ipv4_frags_exit_net }.
- pre_exit calls fqdir_pre_exit (WRITE_ONCE dead = true; subsequent ip_expire flushes silently).
- exit calls ip4_frags_ns_ctl_unregister + fqdir_exit (RCU-deferred free; ensures no in-flight ip_defrag references).

## Acceptance Criteria

- [ ] AC-1: Two fragments arrive in order (offset 0 + IP_MF, then offset N): ip_defrag → ip_find creates ipq → ip_frag_queue twice → ip_frag_reasm coalesces; returns 0; reassembled skb has tot_len = sum-data + ihl.
- [ ] AC-2: Fragments arrive out of order: rbtree insert orders correctly; reasm still succeeds when meat == len ∧ FIRST_IN+LAST_IN.
- [ ] AC-3: Duplicate fragment (same offset): inet_frag_queue_insert returns IPFRAG_DUP → REASMFAILS bumped, skb dropped with DUP_FRAG reason; queue retained.
- [ ] AC-4: Overlapping fragment: REASM_OVERLAPS bumped + REASMFAILS bumped; queue killed; skb freed.
- [ ] AC-5: Non-last fragment with end not 8-byte aligned: end &= ~7 forced; ip_summed downgraded to CHECKSUM_NONE.
- [ ] AC-6: Reassembled length > 65535: -E2BIG; REASMFAILS bumped; queue killed.
- [ ] AC-7: Timeout (30s default): ip_expire fires → REASMTIMEOUT + REASMFAILS bumped; if FIRST_IN ∧ user is LOCAL_DELIVER ∧ rt_type RTN_LOCAL: ICMP TIME_EXCEEDED/FRAGTIME emitted.
- [ ] AC-8: Conntrack user (CONNTRACK_IN range): frag_expire_skip_icmp returns true → no ICMP on timeout.
- [ ] AC-9: max_dist exceeded (peer rid jump > 64): ip_frag_too_far → ip_frag_reinit; if timer not already pending → -ETIMEDOUT, drop.
- [ ] AC-10: Memory pressure: fqdir->mem.count > high_thresh → LRU eviction shrinks to low_thresh; REASMFAILS bumped per eviction.
- [ ] AC-11: Two flows differing only in vif (l3mdev VRF): two distinct ipq queues; no cross-VRF reassembly.
- [ ] AC-12: Two flows differing only in user (LOCAL_DELIVER vs CONNTRACK_IN): two distinct ipq queues.
- [ ] AC-13: ECN merge: per RFC3168 ip_frag_ecn_table; if any frag has ECT(1) ∧ another has ECT(0) → merged correctly; if 0xff (illegal combo) → -EINVAL.
- [ ] AC-14: Reassembled skb with all fragments DF-set: IPSKB_FRAG_PMTU set; iph→frag_off = IP_DF (so output side will refragment with DF).
- [ ] AC-15: Per-netns teardown: pre_exit marks fqdir dead; pending ip_expire silently flushes; exit frees rhashtable; no use-after-free.

## Architecture

```
struct FragV4CompareKey {
  saddr: __be32,
  daddr: __be32,
  user: u32,                // ip_defrag_users
  vif: i32,                 // l3mdev master ifindex
  id: __be16,
  protocol: u8,
}

struct Ipq {
  q: InetFragQueue,         // shared base: { key, lock, timer, rb_fragments, fragments_tail,
                            //                meat, len, max_size, flags, stamp, fqdir, refcount }
  ecn: u8,                  // RFC3168 accumulator
  max_df_size: u16,
  iif: i32,
  rid: u32,
  peer: Option<*InetPeer>,
}

struct Fqdir {              // per-netns
  high_thresh: u32,         // 4 MB default
  low_thresh: u32,          // 3 MB default
  timeout: u32,             // IP_FRAG_TIME (30 * HZ)
  max_dist: u32,            // 64
  mem: AtomicLongCounter,
  rhashtable: RhashTable<InetFragQueue>,
  net: *Net,
  dead: bool,
  ops: *InetFrags,          // family ops
}
```

`IpFragment::defrag(net, skb, user) -> i32`:
1. __IP_INC_STATS(net, IPSTATS_MIB_REASMREQDS).
2. rcu_read_lock().
3. dev = skb->dev ?: skb_dst_dev_rcu(skb).
4. vif = l3mdev_master_ifindex_rcu(dev).
5. qp = IpFragment::find(net, ip_hdr(skb), user, vif).
6. if let Some(qp) = qp:
   - let mut refs = 0.
   - spin_lock(&qp.q.lock).
   - ret = IpFragment::frag_queue(qp, skb, &mut refs).
   - spin_unlock(&qp.q.lock).
   - rcu_read_unlock().
   - inet_frag_putn(&qp.q, refs).
   - return ret.
7. rcu_read_unlock().
8. __IP_INC_STATS(net, IPSTATS_MIB_REASMFAILS).
9. kfree_skb(skb).
10. return -ENOMEM.

`IpFragment::find(net, iph, user, vif) -> Option<*Ipq>`:
1. key = FragV4CompareKey { saddr: iph->saddr, daddr: iph->daddr, user, vif, id: iph->id, protocol: iph->protocol }.
2. q = inet_frag_find(net.ipv4.fqdir, &key).
3. if q.is_null(): return None.
4. return Some(container_of(q, Ipq, q)).

`IpFragment::frag_queue(qp, skb, refs) -> i32`:
1. if qp.q.flags & INET_FRAG_COMPLETE: SKB_DR_SET(DUP_FRAG); goto err.
2. if !(IPCB(skb).flags & IPSKB_FRAG_COMPLETE) ∧ IpFragment::frag_too_far(qp) ∧ IpFragment::frag_reinit(qp): inet_frag_kill(&qp.q, refs); goto err.
3. ecn = IpFragment::frag_ecn(ip_hdr(skb)->tos).
4. raw = ntohs(ip_hdr(skb)->frag_off); flags = raw & !IP_OFFSET; offset = (raw & IP_OFFSET) << 3.
5. ihl = ip_hdrlen(skb); end = offset + skb->len - skb_network_offset(skb) - ihl.
6. if !(flags & IP_MF):
   - if end < qp.q.len ∨ ((qp.q.flags & LAST_IN) ∧ end != qp.q.len): goto discard_qp.
   - qp.q.flags |= LAST_IN; qp.q.len = end.
7. else:
   - if end & 7: end &= !7; if skb->ip_summed != CHECKSUM_UNNECESSARY: skb->ip_summed = CHECKSUM_NONE.
   - if end > qp.q.len: if (qp.q.flags & LAST_IN): goto discard_qp; qp.q.len = end.
8. if end == offset: goto discard_qp.
9. if !pskb_pull(skb, skb_network_offset(skb) + ihl): goto discard_qp.
10. if pskb_trim_rcsum(skb, end - offset): goto discard_qp.
11. prev_tail = qp.q.fragments_tail.
12. err = inet_frag_queue_insert(&qp.q, skb, offset, end).
13. if err: goto insert_error.
14. if dev: qp.iif = dev.ifindex.
15. qp.q.stamp = skb->tstamp; qp.q.tstamp_type = skb->tstamp_type; qp.q.meat += skb->len.
16. qp.ecn |= ecn.
17. add_frag_mem_limit(qp.q.fqdir, skb->truesize).
18. if offset == 0: qp.q.flags |= FIRST_IN.
19. fragsize = skb->len + ihl; if fragsize > qp.q.max_size: qp.q.max_size = fragsize.
20. if (iph->frag_off & IP_DF) ∧ fragsize > qp.max_df_size: qp.max_df_size = fragsize.
21. if qp.q.flags == (FIRST_IN | LAST_IN) ∧ qp.q.meat == qp.q.len:
    - save+null skb->_skb_refdst.
    - err = IpFragment::frag_reasm(qp, skb, prev_tail, dev, refs).
    - restore skb->_skb_refdst.
    - if err: inet_frag_kill(&qp.q, refs).
    - return err.
22. skb_dst_drop(skb); skb_orphan(skb).
23. return -EINPROGRESS.
24. /* insert_error / discard_qp / err */:
    - if err == IPFRAG_DUP: reason = DUP_FRAG; err = -EINVAL.
    - else: __IP_INC_STATS(REASM_OVERLAPS); inet_frag_kill(&qp.q, refs); __IP_INC_STATS(REASMFAILS).
    - kfree_skb_reason(skb, reason); return err.

`IpFragment::frag_reasm(qp, skb, prev_tail, dev, refs) -> i32`:
1. inet_frag_kill(&qp.q, refs).
2. ecn = ip_frag_ecn_table[qp.ecn]; if ecn == 0xff: err = -EINVAL; goto out_fail.
3. reasm_data = inet_frag_reasm_prepare(&qp.q, skb, prev_tail); if !reasm_data: goto out_nomem.
4. len = ip_hdrlen(skb) + qp.q.len.
5. if len > 65535: err = -E2BIG; goto out_oversize.
6. inet_frag_reasm_finish(&qp.q, skb, reasm_data, IpFragment::coalesce_ok(qp)).
7. skb->dev = dev.
8. IPCB(skb)->frag_max_size = max(qp.max_df_size, qp.q.max_size).
9. iph = ip_hdr(skb); iph->tot_len = htons(len); iph->tos |= ecn.
10. if qp.max_df_size == qp.q.max_size: IPCB(skb)->flags |= IPSKB_FRAG_PMTU; iph->frag_off = htons(IP_DF).
11. else: iph->frag_off = 0.
12. ip_send_check(iph).
13. __IP_INC_STATS(REASMOKS).
14. Zero qp.q.rb_fragments / fragments_tail / last_run_head.
15. return 0.

`IpFragment::expire(timer)`:
1. frag = container_of(timer); qp = container_of(frag, Ipq, q); net = qp.q.fqdir->net.
2. rcu_read_lock(); spin_lock(&qp.q.lock); refs = 1.
3. if qp.q.flags & INET_FRAG_COMPLETE: goto out.
4. qp.q.flags |= INET_FRAG_DROP; inet_frag_kill(&qp.q, &refs).
5. if READ_ONCE(qp.q.fqdir->dead): inet_frag_queue_flush(&qp.q, 0); goto out.
6. __IP_INC_STATS(net, REASMFAILS); __IP_INC_STATS(net, REASMTIMEOUT).
7. if !(qp.q.flags & FIRST_IN): goto out.
8. head = inet_frag_pull_head(&qp.q); if !head: goto out.
9. head->dev = dev_get_by_index_rcu(net, qp.iif); if !head->dev: goto out.
10. iph = ip_hdr(head); reason = ip_route_input_noref(head, iph->daddr, iph->saddr, ip4h_dscp(iph), head->dev); if reason: goto out.
11. reason = FRAG_REASM_TIMEOUT.
12. if IpFragment::expire_skip_icmp(qp.q.key.v4.user) ∧ skb_rtable(head)->rt_type != RTN_LOCAL: goto out.
13. spin_unlock(&qp.q.lock).
14. icmp_send(head, ICMP_TIME_EXCEEDED, ICMP_EXC_FRAGTIME, 0).
15. /* out / out_rcu_unlock: kfree_skb_reason(head, reason); inet_frag_putn(&qp.q, refs); rcu_read_unlock. */

`IpFragment::init_net(net) -> i32`:
1. res = fqdir_init(&net.ipv4.fqdir, &ip4_frags, net); if res < 0: return res.
2. net.ipv4.fqdir->high_thresh = 4 * 1024 * 1024.
3. net.ipv4.fqdir->low_thresh = 3 * 1024 * 1024.
4. net.ipv4.fqdir->timeout = IP_FRAG_TIME.
5. net.ipv4.fqdir->max_dist = 64.
6. res = ip4_frags_ns_ctl_register(net).
7. if res < 0: fqdir_exit(net.ipv4.fqdir).
8. return res.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `frag_offset_8byte_aligned` | INVARIANT | per-ip_frag_queue: non-last frag end &= ~7. |
| `reasm_len_le_65535` | INVARIANT | per-ip_frag_reasm: len > 65535 ⟹ -E2BIG. |
| `ipq_key_immutable_after_insert` | INVARIANT | per-frag_v4_compare_key: not modified once rhashtable insert succeeds. |
| `INET_FRAG_COMPLETE_blocks_more_frags` | INVARIANT | per-ip_frag_queue: COMPLETE ⟹ DUP_FRAG drop. |
| `peer_refcount_balanced` | INVARIANT | per-ip4_frag_init / _free: inet_peer get/put symmetric. |
| `qp_timer_canceled_on_kill` | INVARIANT | per-inet_frag_kill: timer canceled before free. |
| `frag_mem_limit_accounted` | INVARIANT | per-ip_frag_queue success: add_frag_mem_limit(skb->truesize). |
| `evict_under_high_thresh` | INVARIANT | per-fqdir mem: mem > high_thresh ⟹ evictor runs until ≤ low_thresh. |
| `ICMP_skip_for_conntrack_user` | INVARIANT | per-ip_expire: conntrack user ranges suppress ICMP. |
| `vif_partitions_queues` | INVARIANT | per-find: different vif ⟹ different rhashtable bucket. |

### Layer 2: TLA+

`net/ipv4/ip-fragment.tla`:
- Per-fragment ingress: defrag(skb) → find(qp) → frag_queue(qp,skb) → (insert | reasm | discard).
- Per-timer: ip_expire on timeout → kill + maybe-ICMP.
- Per-pressure: evictor on mem > high_thresh → kill oldest LRU until ≤ low_thresh.
- Properties:
  - `safety_dup_dropped` — per-queue: duplicate-offset fragment dropped, queue intact.
  - `safety_overlap_kills_queue` — per-queue: overlap ⟹ REASM_OVERLAPS + kill.
  - `safety_alignment_RFC791` — per-frag: end %= 0 mod 8 for non-last.
  - `safety_reasm_size_bounded` — per-queue: reasm ⟹ len ≤ 65535.
  - `safety_timer_bounded` — per-queue: lifetime ≤ fqdir->timeout (30s) absent reasm.
  - `safety_vif_user_separation` — per-key: (saddr,daddr,id,protocol) same but different (user,vif) ⟹ separate queues.
  - `liveness_per_queue_eventually_reasm_or_kill` — per-queue: terminates (no leak).
  - `safety_mem_bounded` — per-fqdir: mem.count ≤ high_thresh after evictor stabilizes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IpFragment::defrag` post: refs balanced; rcu_unlock; counter bumped | `IpFragment::defrag` |
| `IpFragment::frag_queue` post: -EINPROGRESS ∨ 0 (reasm) ∨ -EINVAL ∨ -ENOMEM | `IpFragment::frag_queue` |
| `IpFragment::frag_queue` precond: spin_lock(qp.q.lock) held | `IpFragment::frag_queue` |
| `IpFragment::frag_reasm` post: qp killed; iph->tot_len = len; ip_send_check; REASMOKS bumped | `IpFragment::frag_reasm` |
| `IpFragment::frag_too_far` post: rid monotonic; rc only when (end-start) > max_dist | `IpFragment::frag_too_far` |
| `IpFragment::expire` post: refs balanced; rcu_unlock; if not skipped ⟹ ICMP TIME_EXCEEDED | `IpFragment::expire` |
| `IpFragment::init_net` post: fqdir.high_thresh > low_thresh; timeout = IP_FRAG_TIME; max_dist = 64 | `IpFragment::init_net` |
| `IpFragment::frag_init` post: peer.is_some ⟺ fqdir.max_dist > 0 ∧ inet_getpeer_v4 ok | `IpFragment::frag_init` |

### Layer 4: Verus/Creusot functional

`Per-fragment-queue` semantic equivalence to RFC791 § 3.2 reassembly:
- Each fragment carries (id, src, dst, proto, offset*8, MF).
- Hold-time bounded by timeout.
- Discard partial if final not received in time.
- Discard if total > 65535.
- 8-byte offset granularity.
`Per-rhashtable + LRU` semantic equivalence to per-netns memory-bounded queue framework (Documentation/networking/ip-sysctl.rst § ipfrag_*).
`Per-user partition` semantic equivalence to conntrack vs local-delivery vs af_packet isolation (no cross-user fragment leakage).
`Per-RFC1858` defense: max_dist 64 limits anti-overlap reassembly attack window.

## Hardening

(Inherits row-1 features from `net/ipv4/00-overview.md` § Hardening.)

IPv4 reassembly reinforcement:

- **Per-RFC791 8-byte alignment enforced (end &= ~7)** — defense against per-non-aligned fragment overlap exploit.
- **Per-65535 max reassembled length** — defense against per-jumbo-frag amplification (CVE-2017-7889 class).
- **Per-IPFRAG_DUP drop without queue-kill** — defense against per-DoS via dup-flood.
- **Per-IPFRAG_OVERLAP kill queue + REASM_OVERLAPS counter** — defense against per-teardrop attack (CVE-1997-* legacy).
- **Per-max_dist 64 (RFC1858)** — defense against per-anti-overlap reassembly attack.
- **Per-fqdir->high_thresh/low_thresh LRU evictor** — defense against per-memory-exhaustion via dangling frag queues.
- **Per-fqdir->timeout 30s** — defense against per-stale-queue accumulation.
- **Per-netns fqdir isolation** — defense against per-container cross-netns reassembly.
- **Per-l3mdev vif key partition** — defense against per-VRF reassembly collision.
- **Per-user partition (LOCAL_DELIVER vs CONNTRACK_* vs AF_PACKET)** — defense against per-cross-context queue confusion.
- **Per-ICMP TIME_EXCEEDED suppressed for conntrack/af_packet users** — defense against per-amplification reflection.
- **Per-fqdir dead flag at pre_exit + RCU-deferred fqdir_exit** — defense against per-netns-teardown UAF.
- **Per-inet_peer refcount get/put with refcount_inc_not_zero** — defense against per-peer UAF.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/ipv4/inet_fragment.c shared rhashtable / LRU framework (covered separately if expanded)
- net/ipv6/reassembly.c IPv6 reassembly (covered separately)
- net/netfilter/nf_conntrack_proto_*.c conntrack defrag integration (covered separately)
- net/ipv4/ip_output.c transmit-side fragmentation (covered in `ip-output.md` Tier-3)
- net/ipv4/ip_input.c local-deliver pre-defrag hook (covered in `ip_input.md` Tier-3)
- net/ipv4/inetpeer.c inet_peer entry management (covered separately)
- Implementation code
