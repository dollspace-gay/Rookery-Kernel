# Tier-3: net/netfilter/nf_conntrack_core.c — connection tracking engine

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/netfilter/00-overview.md
upstream-paths:
  - net/netfilter/nf_conntrack_core.c (~2819 lines)
  - net/netfilter/nf_conntrack_expect.c
  - net/netfilter/nf_conntrack_helper.c
  - net/netfilter/nf_conntrack_extend.c
  - include/net/netfilter/nf_conntrack.h
  - include/net/netfilter/nf_conntrack_tuple.h
  - include/net/netfilter/nf_conntrack_expect.h
  - include/net/netfilter/nf_conntrack_helper.h
  - include/uapi/linux/netfilter/nf_conntrack_common.h
-->

## Summary

Netfilter connection tracking ("conntrack") classifies every packet into a per-flow `struct nf_conn` identified by a pair of `struct nf_conntrack_tuple` (one per direction). The conntrack engine sits in `nf_conntrack_in` at hook PRE_ROUTING (+ LOCAL_OUT), populates `skb->_nfct` with `(conn, ctinfo)`, and lets later hooks (NAT, filtering, helpers) make stateful decisions. Per-netns flow set is stored in a global `nf_conntrack_hash` array of `hlist_nulls_head` buckets (allocated via `nf_ct_alloc_hashtable`), with both the ORIGINAL and REPLY tuples inserted (a packet matching either gets the same `nf_conn`). Flows start UNCONFIRMED (allocated by `init_conntrack`, attached to the skb), become CONFIRMED at POST_ROUTING by `__nf_conntrack_confirm` (which atomically inserts both tuples under bucket locks), and end DYING when their `timeout` fires (lazy GC by `gc_worker` workqueue, plus per-skb expiry checks). Expectations (`struct nf_conntrack_expect`) registered by L7 helpers (FTP, SIP, …) seed the table for predicted related child flows. Per-protocol state machines (TCP/UDP/ICMP/SCTP/GRE) live in sibling files. Critical for: stateful firewalling, masquerade NAT, container egress, ipvs persistence, ct-action in tc.

This Tier-3 covers `net/netfilter/nf_conntrack_core.c` (~2819 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nf_conn` | per-flow conntrack entry | `NfConn` |
| `struct nf_conntrack_tuple` | per-direction (src, dst, proto, port/icmp, l3num, dir) key | `NfConntrackTuple` |
| `struct nf_conntrack_tuple_hash` | per-tuple hlist-nulls node | `NfConntrackTupleHash` |
| `struct nf_conntrack_expect` | per-expected-related-flow | `NfConntrackExpect` |
| `struct nf_conntrack_helper` | per-L7 helper (FTP/SIP/PPTP/…) | `NfConntrackHelper` |
| `struct nf_conn_help` | per-flow helper attachment (ext slot) | `NfConnHelp` |
| `struct nf_ct_ext` | per-flow extension area (acct/labels/timeout/…) | `NfCtExt` |
| `struct conntrack_gc_work` | per-deferred GC work | `ConntrackGcWork` |
| `nf_conntrack_hash` (`hlist_nulls_head[]`) | global per-bucket chain | `Conntrack::HASH` |
| `nf_conntrack_locks` (`spinlock_t[CONNTRACK_LOCKS]`) | per-bucket-stripe lock | `Conntrack::LOCKS` |
| `nf_conntrack_htable_size` | per-bucket-count | `Conntrack::HTABLE_SIZE` |
| `nf_conntrack_max` | per-netns soft cap | `Conntrack::MAX` |
| `nf_conntrack_hash_rnd` (`siphash_key`) | per-boot hash secret | `Conntrack::HASH_RND` |
| `nf_conntrack_generation` (`seqcount`) | per-hash-resize generation | `Conntrack::GENERATION` |
| `hash_conntrack_raw()` / `hash_conntrack()` / `scale_hash()` | per-tuple → 32-bit hash | `Conntrack::hash_*` |
| `nf_ct_get_tuple()` / `nf_ct_get_tuplepr()` | per-skb → tuple extraction (per-proto offset) | `Conntrack::get_tuple` |
| `nf_ct_invert_tuple()` | per-original → reply tuple | `Conntrack::invert_tuple` |
| `nf_ct_alloc_hashtable()` | per-bucket-array allocator (kvzalloc + INIT_HLIST_NULLS) | `Conntrack::alloc_hashtable` |
| `nf_conntrack_hash_resize()` | per-online resize (rehash all) | `Conntrack::hash_resize` |
| `__nf_conntrack_alloc()` / `nf_conntrack_alloc()` | per-flow slab alloc (SLAB_TYPESAFE_BY_RCU) | `NfConn::alloc` |
| `nf_conntrack_free()` | per-flow slab free | `NfConn::free` |
| `init_conntrack()` | per-new-flow setup (alloc + expect lookup + helper assign) | `Conntrack::init_conn` |
| `resolve_normal_ct()` | per-skb tuple → existing or new `nf_conn` | `Conntrack::resolve_normal_ct` |
| `__nf_conntrack_find_get()` | per-tuple bucket lookup (RCU + refcnt) | `Conntrack::find_get` |
| `nf_conntrack_find_get()` | per-zone tuple lookup wrapper | `Conntrack::find_get_zone` |
| `__nf_conntrack_hash_insert()` | per-confirm bucket linkage (both directions) | `Conntrack::hash_insert` |
| `nf_conntrack_hash_check_insert()` | per-NAT-clash check + insert | `Conntrack::hash_check_insert` |
| `__nf_conntrack_confirm()` | per-skb POSTROUTING confirm path | `Conntrack::confirm` |
| `nf_conntrack_in()` | per-skb PREROUTING / LOCAL_OUT entry | `Conntrack::nf_in` |
| `nf_conntrack_handle_packet()` | per-proto packet dispatch (TCP/UDP/ICMP/SCTP/GRE/generic) | `Conntrack::handle_packet` |
| `nf_conntrack_handle_icmp()` | per-ICMP error → related-flow tagging | `Conntrack::handle_icmp` |
| `__nf_ct_refresh_acct()` | per-skb timeout refresh + acct increment | `NfConn::refresh_acct` |
| `nf_ct_acct_add()` | per-direction byte/packet counters | `NfConn::acct_add` |
| `nf_ct_delete()` | per-flow death (notify + unlink + put) | `NfConn::delete` |
| `nf_ct_destroy()` | per-flow final free (after refcnt reaches 0) | `NfConn::destroy` |
| `nf_ct_tmpl_alloc()` / `_free()` | per-template (CT target via iptables/nftables) | `NfConn::tmpl_alloc` / `tmpl_free` |
| `early_drop()` / `early_drop_list()` | per-table-full random-victim drop | `Conntrack::early_drop` |
| `gc_worker()` | per-deferred-WQ expiry scan | `Conntrack::gc_worker` |
| `conntrack_gc_work_init()` | per-WQ setup | `Conntrack::gc_work_init` |
| `nf_ct_find_expectation()` | per-tuple expectation match (in `init_conntrack`) | `Expect::find` |
| `__nf_ct_try_assign_helper()` | per-flow auto-helper from policy | `Helper::try_assign` |
| `nf_conntrack_tuple_taken()` | per-tuple uniqueness check (used by NAT alloc) | `Conntrack::tuple_taken` |
| `nf_ct_iterate_cleanup_net()` / `nf_ct_iterate_destroy()` | per-netns walk + per-conn callback | `Conntrack::iterate_cleanup_net` |
| `nf_ct_resolve_clash()` / `_harder()` | per-confirm clash recovery | `Conntrack::resolve_clash` |
| `nf_conntrack_attach()` | per-related skb (e.g. REJECT) tag | `Conntrack::attach` |
| `nf_conntrack_update()` | per-skb post-NAT tuple update | `Conntrack::update` |
| `nf_conntrack_cleanup_start()` / `_end()` / `_net()` / `_net_list()` | per-init / teardown | `Conntrack::cleanup_*` |
| `nf_conntrack_init_start()` | per-module init (slab + hashtable + GC) | `Conntrack::init_start` |
| `nf_conntrack_hook` (`struct nf_ct_hook`) | per-indirect hook vtable | `Conntrack::HOOK` |

## Compatibility contract

REQ-1: struct nf_conn:
- ct_general: `struct nf_conntrack { use: refcount_t }` — refcount (1 = hash; +1 per skb; +1 per master expectation).
- lock: spinlock_t — per-flow lock (TCP state mutation, NAT seq adjust, helper state).
- timeout: u32 jiffies32 deadline (absolute, vs `nfct_time_stamp`).
- zone: per-conntrack-zone (CONFIG_NF_CONNTRACK_ZONES).
- tuplehash[IP_CT_DIR_MAX]: per-direction `nf_conntrack_tuple_hash` (original + reply) — both indexed in `nf_conntrack_hash`.
- status: unsigned long bitset — IPS_* flags (EXPECTED, SEEN_REPLY, ASSURED, CONFIRMED, …, DYING).
- ct_net: `possible_net_t` per-netns back-pointer.
- nat_bysource: `hlist_node` (CONFIG_NF_NAT) per-source hashtable link.
- __nfct_init_offset: marker; all below members memset on alloc.
- master: per-expectation parent (nf_conn).
- mark / secmark: per-policy tags.
- ext: per-extension area (acct, ecache, timeout, labels, secmark, helper, …).
- proto: per-protocol union scratch.

REQ-2: struct nf_conntrack_tuple:
- src.l3num: u16 — NFPROTO_IPV4 / IPV6.
- src.u3: union nf_inet_addr — IPv4 / IPv6 src.
- src.u: union nf_conntrack_man_proto — per-proto src field (tcp.port / udp.port / icmp.id / sctp.port / gre.key).
- dst.u3: union nf_inet_addr — dst.
- dst.u: per-proto dst (port / icmp.{type,code} / sctp.port / gre.key).
- dst.protonum: u8 IP proto.
- dst.dir: u8 IP_CT_DIR_ORIGINAL / IP_CT_DIR_REPLY.
- `__nfct_hash_offsetend` marks bytes covered by hash (excludes dir).

REQ-3: struct nf_conntrack_tuple_hash:
- hnnode: `hlist_nulls_node` — bucket link.
- tuple: `nf_conntrack_tuple`.
- `nf_ct_tuplehash_to_ctrack(h) = container_of(h, nf_conn, tuplehash[h->tuple.dst.dir])`.

REQ-4: Global hashtable:
- `nf_conntrack_hash` is an array of `nf_conntrack_htable_size` `hlist_nulls_head` buckets.
- Buckets allocated via `nf_ct_alloc_hashtable(&sizep, /*nulls=*/1)`: rounds size up to PAGE_SIZE / sizeof(head), `kvzalloc_objs`, `INIT_HLIST_NULLS_HEAD(&hash[i], i)` per slot for SLAB_TYPESAFE_BY_RCU-safe walks.
- Per-bucket stripe lock: `nf_conntrack_locks[CONNTRACK_LOCKS]` indexed by `hash & (CONNTRACK_LOCKS - 1)`.
- Per-`nf_conntrack_locks_all_lock` + `_locks_all` boolean for global table writes (resize, cleanup).
- `nf_conntrack_double_lock(h1, h2, seq)` re-checks `read_seqcount(&nf_conntrack_generation) == seq` after locking; retries on resize race.

REQ-5: Hashing:
- `hash_conntrack_raw(tuple, zoneid, net)`: siphash over `(tuple.src.u3 || tuple.dst.u3 || tuple.src.u.all || tuple.dst.u.all || tuple.dst.protonum || tuple.src.l3num || zoneid || net.hash_mix)`, using boot-random `nf_conntrack_hash_rnd`.
- `scale_hash(hash) = reciprocal_scale(hash, READ_ONCE(nf_conntrack_htable_size))`.
- `hash_conntrack(net, tuple, zoneid) = scale_hash(hash_conntrack_raw(...))`.
- `__hash_conntrack(net, tuple, zoneid, size)` = explicit-size variant for resize.

REQ-6: `nf_conntrack_in(skb, state) -> u32`:
- tmpl = nf_ct_get(skb, &ctinfo).
- If tmpl set ∧ !template ∨ ctinfo == IP_CT_UNTRACKED: return NF_ACCEPT (already classified or skip).
- skb._nfct = 0.
- get_l4proto(skb, network_offset, pf, &protonum) → dataoff (≤ 0: bump invalid, NF_ACCEPT).
- If protonum ∈ {IPPROTO_ICMP, IPPROTO_ICMPV6}: nf_conntrack_handle_icmp(tmpl, skb, dataoff, protonum, state); if ICMP error tagged related flow → goto out.
- 'repeat: ret = resolve_normal_ct(tmpl, skb, dataoff, protonum, state). On error (-ENOMEM): NF_CT_STAT_INC(drop); ret = NF_DROP; goto out.
- ct = nf_ct_get(skb, &ctinfo). If NULL: NF_CT_STAT_INC(invalid); NF_ACCEPT.
- ret = nf_conntrack_handle_packet(ct, skb, dataoff, ctinfo, state). On ret ≤ 0: nf_ct_put(ct); skb._nfct = 0; if ret == -NF_REPEAT: goto 'repeat (TCP reopen); else NF_CT_STAT_INC(invalid); if NF_DROP: NF_CT_STAT_INC(drop); ret = -ret; goto out.
- If ctinfo == IP_CT_ESTABLISHED_REPLY ∧ test_and_set_bit(IPS_SEEN_REPLY_BIT, &ct.status) == 0: nf_conntrack_event_cache(IPCT_REPLY, ct).
- out: if tmpl: nf_ct_put(tmpl). return ret.

REQ-7: `resolve_normal_ct(tmpl, skb, dataoff, protonum, state) -> i32`:
- nf_ct_get_tuple(skb, net_offset, dataoff, pf, protonum, net, &tuple) → false: return 0 (skip).
- zone = nf_ct_zone_tmpl(tmpl, skb, &tmp).
- zone_id = nf_ct_zone_id(zone, IP_CT_DIR_ORIGINAL).
- hash = hash_conntrack_raw(&tuple, zone_id, net).
- h = __nf_conntrack_find_get(net, zone, &tuple, hash).
- If !h ∧ zone has separate REPLY zone-id: retry with rid hash.
- If !h: h = init_conntrack(net, tmpl, &tuple, skb, dataoff, hash); on ERR: return PTR_ERR; on NULL: return 0.
- ct = nf_ct_tuplehash_to_ctrack(h).
- Direction determines ctinfo: NF_CT_DIRECTION(h) == IP_CT_DIR_REPLY → IP_CT_ESTABLISHED_REPLY; else status & IPS_SEEN_REPLY → IP_CT_ESTABLISHED; status & IPS_EXPECTED → IP_CT_RELATED; else IP_CT_NEW.
- nf_ct_set(skb, ct, ctinfo). Return 0.

REQ-8: `__nf_conntrack_find_get(net, zone, tuple, hash) -> tuple_hash*`:
- 'begin: rcu_read_lock.
- sequence = read_seqcount_begin(&nf_conntrack_generation).
- Walk `hlist_nulls_for_each_entry_rcu(h, n, &nf_conntrack_hash[scale_hash(hash)], hnnode)`:
  - If nf_ct_key_equal(h, tuple, zone, net): refcount_inc_not_zero(&ct->ct_general.use); if success: rcu_read_unlock; return h.
- Per-nulls-marker termination: if (((unsigned long)n)&1) → ((((unsigned long)n)>>1) != scale_hash(hash)) → goto 'begin (resize race).
- If read_seqcount_retry: goto 'begin.
- rcu_read_unlock. return NULL.

REQ-9: `init_conntrack(net, tmpl, tuple, skb, dataoff, hash) -> tuple_hash*`:
- nf_ct_invert_tuple(&repl_tuple, tuple) → false: return NULL.
- zone = nf_ct_zone_tmpl(tmpl, skb, &tmp).
- ct = __nf_conntrack_alloc(net, zone, tuple, &repl_tuple, GFP_ATOMIC, hash). On ERR: ERR_CAST.
- nf_ct_add_synproxy(ct, tmpl) → false: nf_conntrack_free(ct); return ERR_PTR(-ENOMEM).
- timeout_ext = tmpl ? nf_ct_timeout_find(tmpl) : NULL; if set: nf_ct_timeout_ext_add(ct, rcu_dereference(timeout_ext.timeout), GFP_ATOMIC).
- nf_ct_acct_ext_add(ct, GFP_ATOMIC); nf_ct_tstamp_ext_add; nf_ct_labels_ext_add.
- Ecache ext if events enabled.
- cnet = nf_ct_pernet(net); if cnet.expect_count > 0: spin_lock_bh(&nf_conntrack_expect_lock); exp = nf_ct_find_expectation(net, zone, tuple, !tmpl || nf_ct_is_confirmed(tmpl)).
- If exp: set IPS_EXPECTED_BIT in ct.status; ct.master = exp.master; if exp.helper: nf_ct_helper_ext_add(ct); rcu_assign_pointer(help.helper, exp.helper); copy mark / secmark; NF_CT_STAT_INC(expect_new); spin_unlock_bh.
- If !exp ∧ tmpl: __nf_ct_try_assign_helper(ct, tmpl, GFP_ATOMIC).
- smp_wmb.
- refcount_set(&ct.ct_general.use, 1).
- If exp: exp.expectfn(ct, exp); nf_ct_expect_put(exp).
- Return &ct.tuplehash[IP_CT_DIR_ORIGINAL].

REQ-10: `__nf_conntrack_alloc(net, zone, orig, repl, gfp, hash)`:
- ct_count = atomic_inc_return(&cnet.count). If > nf_conntrack_max ∧ early_drop fails: dec; warn-rate-limited; return ERR_PTR(-ENOMEM).
- ct = kmem_cache_alloc(nf_conntrack_cachep, gfp) (SLAB_TYPESAFE_BY_RCU — must not use kmem_cache_zalloc; init only the per-instance state).
- spin_lock_init(&ct.lock).
- ct.tuplehash[ORIGINAL].tuple = *orig; .pprev = NULL.
- ct.tuplehash[REPLY].tuple = *repl; cache hash in (unsigned long*)&.pprev for reuse during confirm.
- ct.status = 0. WRITE_ONCE(ct.timeout, 0). write_pnet(&ct.ct_net, net).
- memset_after(ct, 0, __nfct_init_offset) — zero the second half.
- nf_ct_zone_add(ct, zone).
- refcount_set(&ct.ct_general.use, 0) — pre-list state; set to 1 only when attached to skb.
- Return ct.

REQ-11: `__nf_conntrack_confirm(skb) -> u32` (POST_ROUTING):
- ct = nf_ct_get(skb, &ctinfo). If CTINFO2DIR(ctinfo) != IP_CT_DIR_ORIGINAL: return NF_ACCEPT.
- zone = nf_ct_zone(ct). local_bh_disable.
- do { sequence = read_seqcount_begin(&nf_conntrack_generation); hash = scale_hash(*(unsigned long*)&ct.tuplehash[REPLY].hnnode.pprev) /* reuse cached */; reply_hash = hash_conntrack(net, &ct.tuplehash[REPLY].tuple, nf_ct_zone_id(REPLY)); } while (nf_conntrack_double_lock(hash, reply_hash, sequence)).
- If unlikely nf_ct_is_confirmed(ct): WARN; unlock; return NF_DROP.
- If !nf_ct_ext_valid_pre(ct.ext) ∨ nf_ct_is_dying(ct): NF_CT_STAT_INC(insert_failed); goto dying.
- max_chainlen = MIN_CHAINLEN + get_random_u32_below(MAX_CHAINLEN).
- Walk hash bucket: if key_equal with ORIGINAL tuple → goto out (lost race); chainlen overflow → chaintoolong.
- Walk reply_hash bucket: same with REPLY tuple.
- ct.timeout += nfct_time_stamp.
- __nf_conntrack_insert_prepare(ct).
- __nf_conntrack_hash_insert(ct, hash, reply_hash) — set IPS_CONFIRMED_BIT before inserting both nodes; pairs with RCU lookup that filters non-IPS_CONFIRMED.
- nf_conntrack_double_unlock; local_bh_enable; nf_ct_acct_merge / event notify.
- Return NF_ACCEPT.
- out (clash): if NAT clash → nf_ct_resolve_clash (merge bytes/pkts into existing conn) or NF_DROP.

REQ-12: `__nf_conntrack_hash_insert(ct, hash, reply_hash)`:
- set_bit(IPS_CONFIRMED_BIT, &ct.status) — lookup filter.
- hlist_nulls_add_head_rcu(&ct.tuplehash[ORIGINAL].hnnode, &nf_conntrack_hash[hash]).
- hlist_nulls_add_head_rcu(&ct.tuplehash[REPLY].hnnode, &nf_conntrack_hash[reply_hash]).
- atomic_inc(&cnet.count) is NOT done here (done in alloc).

REQ-13: Connection-tracking state machine (per skb):
- ctinfo (`enum ip_conntrack_info`):
  - IP_CT_NEW (0): first packet, no reply seen.
  - IP_CT_ESTABLISHED (1): two-way seen.
  - IP_CT_RELATED (2): expectation-matched (FTP data, ICMP error).
  - IP_CT_ESTABLISHED_REPLY (3): reply direction.
  - IP_CT_RELATED_REPLY (4): reply direction of related.
  - IP_CT_UNTRACKED (7): explicit notrack (template).
- Status (`IPS_*` bits in ct.status):
  - IPS_EXPECTED (bit 0): seeded by expectation.
  - IPS_SEEN_REPLY (bit 1): reply observed.
  - IPS_ASSURED (bit 2): TCP assured / UDP-stream.
  - IPS_CONFIRMED (bit 3): inserted into table.
  - IPS_SRC_NAT / DST_NAT / SRC_NAT_DONE / DST_NAT_DONE (bits 4-7).
  - IPS_DYING (bit 9): scheduled for delete.
  - IPS_FIXED_TIMEOUT (bit 10): refresh does not extend.
  - IPS_TEMPLATE (bit 11): not a real flow (skb tmpl).
  - IPS_UNTRACKED, IPS_HELPER, IPS_OFFLOAD, IPS_HW_OFFLOAD, …

REQ-14: Expectations (`nf_conntrack_expect`):
- Per-master conntrack expects a specific child tuple under a class.
- Storage: `nf_ct_expect_hash` (global) + per-master `nf_conn_help.expectations` list.
- `nf_ct_find_expectation(net, zone, tuple, confirmed_only)` matches a packet's tuple against expectations under `nf_conntrack_expect_lock`.
- Per-expect timeout (`expect.timeout`) is a `timer_list` deleting the expect when fired.
- `expectfn(new, expect)` called from `init_conntrack` after refcount_set(1).
- Per-`expect.master.use` ref pinned for the expect's lifetime.

REQ-15: Helpers (`nf_conntrack_helper`):
- Per-name + (l3num, protonum) lookup table.
- Per-`help(skb, protoff, ct, ctinfo)` invoked per packet; returns NF_ACCEPT or negative on parse failure.
- Per-`expect_policy[]` describes max expectations per class.
- Attached via `nf_ct_helper_ext_add(ct, GFP_ATOMIC)` (creates `nf_conn_help` extension); `__nf_ct_try_assign_helper(ct, tmpl, gfp)` auto-assigns from policy.

REQ-16: Timeout policy:
- Per-proto default timeout array (TCP: per-state; UDP: stream / unreplied; ICMP / GRE / SCTP / generic).
- `nf_ct_timeout_lookup(ct)` returns active per-conn timeout (may override default via `nf_ct_timeout_ext`).
- `__nf_ct_refresh_acct(ct, ctinfo, extra_jiffies, bytes)`: if !IPS_FIXED_TIMEOUT: WRITE_ONCE(ct.timeout, nfct_time_stamp + extra_jiffies); + acct merge.
- Per-flow expires when `nf_ct_is_expired(ct) = time_after32(nfct_time_stamp, ct.timeout)` is true.

REQ-17: GC worker (`gc_worker` workqueue):
- Scans buckets in chunks (`GC_SCAN_MAX_DURATION`, `GC_SCAN_INITIAL_COUNT`).
- For each entry: if expired or DYING → `nf_ct_gc_expired(ct)` (`nf_ct_kill` if refcount allows).
- Per-`early_drop` flag activates when table nears `nf_conntrack_max`: random-bucket victim drop.
- Adaptive interval `avg_timeout` reduced if many expired found.

REQ-18: Cleanup / lifecycle:
- `nf_ct_delete(ct, portid, report)`: under per-flow lock, mark IPS_DYING, drop from hash + lists, notify ecache, nf_ct_put.
- `nf_ct_destroy(nfct)`: invoked when use refcount reaches 0; runs `nf_conntrack_destroy()` callbacks then kmem_cache_free.
- Per-NAT: `destroy_gre_conntrack(ct)` for GRE PPTP master tear-down.

REQ-19: Templates (`nf_ct_tmpl_*`):
- Per-skb tag (via iptables `-j CT` / nftables `ct ...`) selecting zone, mark, helper, timeout — not a real flow.
- IPS_TEMPLATE bit; not inserted into hash; tmpl_alloc allocates via slab and adds extensions; tmpl_free frees ext + cache.

REQ-20: Per-netns init:
- `nf_conntrack_init_start()` (one-shot): create `nf_conntrack_cachep` slab (SLAB_HWCACHE_ALIGN | SLAB_TYPESAFE_BY_RCU), allocate `nf_conntrack_hash` via `nf_ct_alloc_hashtable`, init `nf_conntrack_hash_rnd` (siphash key), init `conntrack_gc_work`.
- Per-netns init via `nf_conntrack_proto_pernet_init` + `nf_conntrack_helper_pernet_init` + `nf_conntrack_ecache_pernet_init`.

## Acceptance Criteria

- [ ] AC-1: First packet of new flow: `nf_conntrack_in` allocates `nf_conn`, tags skb with `IP_CT_NEW`.
- [ ] AC-2: Reply packet of established flow: `nf_conntrack_in` resolves existing nf_conn; tags `IP_CT_ESTABLISHED_REPLY`; sets `IPS_SEEN_REPLY_BIT`.
- [ ] AC-3: POST_ROUTING confirm inserts both ORIGINAL and REPLY tuples atomically; `IPS_CONFIRMED_BIT` set before insertion.
- [ ] AC-4: Table-full + early_drop fails: `__nf_conntrack_alloc` returns -ENOMEM; NF_CT_STAT_INC(drop); packet drops with warn-rate-limited message.
- [ ] AC-5: TCP reopen (`-NF_REPEAT` from TCP tracker): `nf_conntrack_in` loops via 'repeat label; fresh conntrack created.
- [ ] AC-6: ICMP error: `nf_conntrack_handle_icmp` tags packet related to original flow (RELATED).
- [ ] AC-7: Expectation match: `init_conntrack` finds expect, sets IPS_EXPECTED, links master, inherits helper / mark / secmark; calls `expectfn`.
- [ ] AC-8: Per-helper `help()` invoked once per packet under per-flow `ct->lock`; returns NF_ACCEPT or invalidates.
- [ ] AC-9: Per-flow timeout expiry: `gc_worker` evicts via `nf_ct_gc_expired`; refcount eventually reaches 0; slab freed.
- [ ] AC-10: Concurrent hashtable resize: in-flight `__nf_conntrack_find_get` retries via seqcount + nulls marker.
- [ ] AC-11: Concurrent confirm + delete: `nf_conntrack_double_lock` covers both buckets; `nf_ct_is_confirmed(ct)` double-confirm rejected.
- [ ] AC-12: Per-netns teardown: `nf_conntrack_cleanup_net` walks netns flows; kill all; free per-netns hash slot if non-shared.
- [ ] AC-13: Template skb: `IP_CT_UNTRACKED` short-circuits `nf_conntrack_in` to NF_ACCEPT immediately.
- [ ] AC-14: Reach `nf_conntrack_max`: subsequent allocs trigger `early_drop`; if no victim → packet dropped.
- [ ] AC-15: `nf_conntrack_max` change at runtime via sysctl/`set_hashsize`: `nf_conntrack_hash_resize` rehashes all flows.

## Architecture

```
struct NfConntrackTuple {
  src: NfConntrackMan {
    u3: NfInetAddr,                      // IPv4 / IPv6
    u: NfConntrackManProto {             // tcp.port | udp.port | icmp.id | sctp.port | gre.key | all
      all: BeU16,
    },
    l3num: u16,                          // AF_INET / AF_INET6
  },
  dst: NfConntrackDst {
    u3: NfInetAddr,
    u: NfConntrackProto {                // tcp.port | udp.port | icmp.{type,code} | sctp.port | gre.key | all
      all: BeU16,
    },
    protonum: u8,                        // IPPROTO_*
    dir: u8,                             // IP_CT_DIR_ORIGINAL | IP_CT_DIR_REPLY
  },
}
```

```
struct NfConntrackTupleHash {
  hnnode: HlistNullsNode,
  tuple: NfConntrackTuple,
}
```

```
struct NfConn {
  ct_general: NfConntrack { use: RefCount },
  lock: SpinLock,
  timeout: u32,                          // jiffies32 absolute
  zone: NfConntrackZone,                 // CONFIG_NF_CONNTRACK_ZONES
  tuplehash: [NfConntrackTupleHash; IP_CT_DIR_MAX],
  status: AtomicULong,                   // IPS_* bitset
  ct_net: PossibleNet,
  nat_bysource: HlistNode,               // CONFIG_NF_NAT
  master: Option<*NfConn>,               // expectation parent
  mark: u32,                             // CONFIG_NF_CONNTRACK_MARK
  secmark: u32,                          // CONFIG_NF_CONNTRACK_SECMARK
  ext: Option<*NfCtExt>,                 // acct/labels/timeout/ecache/helper/...
  proto: NfConntrackProtoUnion,
}
```

```
struct NfConntrackExpect {
  lnode: HlistNode,                      // master.help.expectations
  hnode: HlistNode,                      // global expect hash
  net: PossibleNet,
  tuple: NfConntrackTuple,
  mask: NfConntrackTupleMask,
  zone: NfConntrackZone,
  use: RefCount,
  flags: u32,
  class: u32,
  expectfn: Option<fn(*NfConn, *NfConntrackExpect)>,
  helper: Rcu<*NfConntrackHelper>,
  master: *NfConn,
  timeout: TimerList,
  saved_addr: NfInetAddr,                // CONFIG_NF_NAT
  saved_proto: NfConntrackManProto,
  dir: u8,
}
```

```
struct NfConntrackHelper {
  hnode: HlistNode,
  name: CStr<NF_CT_HELPER_NAME_LEN>,
  refcnt: RefCount,
  me: *Module,
  expect_policy: *NfConntrackExpectPolicy,
  tuple: NfConntrackTuple,
  help: fn(*SkBuff, u32, *NfConn, IpConntrackInfo) -> i32,
  destroy: Option<fn(*NfConn)>,
  from_nlattr: Option<fn(*Nlattr, *NfConn) -> i32>,
  to_nlattr: Option<fn(*SkBuff, *NfConn) -> i32>,
  expect_class_max: u32,
  flags: u32,
  queue_num: u32,                        // userspace helper
  data_len: u16,
  nat_mod_name: CStr<NF_CT_HELPER_NAME_LEN>,
}
```

`Conntrack::nf_in(skb, state) -> u32`:
1. tmpl = nf_ct_get(skb, &ctinfo).
2. if (tmpl ∧ !nf_ct_is_template(tmpl)) ∨ ctinfo == IP_CT_UNTRACKED: return NF_ACCEPT.
3. skb._nfct = 0.
4. dataoff = Conntrack::get_l4proto(skb, skb_network_offset(skb), state.pf, &protonum).
5. if dataoff ≤ 0: NF_CT_STAT_INC(state.net, invalid); ret = NF_ACCEPT; goto out.
6. if protonum ∈ {IPPROTO_ICMP, IPPROTO_ICMPV6}:
   - ret = Conntrack::handle_icmp(tmpl, skb, dataoff, protonum, state).
   - if ret ≤ 0: ret = -ret; goto out.
   - if skb._nfct: goto out.
7. 'repeat:
8. ret = Conntrack::resolve_normal_ct(tmpl, skb, dataoff, protonum, state).
9. if ret < 0: NF_CT_STAT_INC(drop); ret = NF_DROP; goto out.
10. ct = nf_ct_get(skb, &ctinfo).
11. if !ct: NF_CT_STAT_INC(invalid); ret = NF_ACCEPT; goto out.
12. ret = Conntrack::handle_packet(ct, skb, dataoff, ctinfo, state).
13. if ret ≤ 0:
    - nf_ct_put(ct). skb._nfct = 0.
    - if ret == -NF_REPEAT: goto 'repeat.
    - NF_CT_STAT_INC(invalid). if ret == NF_DROP: NF_CT_STAT_INC(drop).
    - ret = -ret. goto out.
14. if ctinfo == IP_CT_ESTABLISHED_REPLY ∧ !test_and_set_bit(IPS_SEEN_REPLY_BIT, &ct.status):
    - nf_conntrack_event_cache(IPCT_REPLY, ct).
15. out: if tmpl: nf_ct_put(tmpl). return ret.

`Conntrack::resolve_normal_ct(tmpl, skb, dataoff, protonum, state) -> i32`:
1. if !nf_ct_get_tuple(skb, skb_network_offset(skb), dataoff, state.pf, protonum, state.net, &tuple): return 0.
2. zone = nf_ct_zone_tmpl(tmpl, skb, &tmp).
3. zone_id = nf_ct_zone_id(zone, IP_CT_DIR_ORIGINAL).
4. hash = Conntrack::hash_conntrack_raw(&tuple, zone_id, state.net).
5. h = Conntrack::find_get(state.net, zone, &tuple, hash).
6. if !h ∧ rid = nf_ct_zone_id(zone, IP_CT_DIR_REPLY) != zone_id:
   - h = Conntrack::find_get(state.net, zone, &tuple, Conntrack::hash_conntrack_raw(&tuple, rid, state.net)).
7. if !h:
   - h = Conntrack::init_conn(state.net, tmpl, &tuple, skb, dataoff, hash).
   - if h.is_err(): return PTR_ERR(h).
   - if !h: return 0.
8. ct = nf_ct_tuplehash_to_ctrack(h).
9. if NF_CT_DIRECTION(h) == IP_CT_DIR_REPLY: ctinfo = IP_CT_ESTABLISHED_REPLY.
10. else: status = READ_ONCE(ct.status); if status & IPS_SEEN_REPLY: ctinfo = IP_CT_ESTABLISHED; else if status & IPS_EXPECTED: ctinfo = IP_CT_RELATED; else: ctinfo = IP_CT_NEW.
11. nf_ct_set(skb, ct, ctinfo). return 0.

`Conntrack::confirm(skb) -> u32`:
1. ct = nf_ct_get(skb, &ctinfo). net = nf_ct_net(ct).
2. if CTINFO2DIR(ctinfo) != IP_CT_DIR_ORIGINAL: return NF_ACCEPT.
3. zone = nf_ct_zone(ct). local_bh_disable.
4. do {
   - sequence = read_seqcount_begin(&nf_conntrack_generation).
   - hash = scale_hash(*(u_long*)&ct.tuplehash[REPLY].hnnode.pprev).
   - reply_hash = Conntrack::hash_conntrack(net, &ct.tuplehash[REPLY].tuple, nf_ct_zone_id(REPLY)).
   } while (Conntrack::double_lock(hash, reply_hash, sequence)).
5. if unlikely nf_ct_is_confirmed(ct): WARN_ON_ONCE(1); Conntrack::double_unlock; local_bh_enable; return NF_DROP.
6. if !Conntrack::ext_valid_pre(ct.ext) ∨ nf_ct_is_dying(ct): NF_CT_STAT_INC(insert_failed); goto dying.
7. max_chainlen = MIN_CHAINLEN + get_random_u32_below(MAX_CHAINLEN).
8. for h in hlist_nulls @ nf_conntrack_hash[hash]:
   - if nf_ct_key_equal(h, ORIGINAL_TUPLE, zone, net): goto out.
   - if ++chainlen > max_chainlen: goto chaintoolong.
9. chainlen = 0.
10. for h in hlist_nulls @ nf_conntrack_hash[reply_hash]:
    - if nf_ct_key_equal(h, REPLY_TUPLE, zone, net): goto out.
    - if ++chainlen > max_chainlen: NF_CT_STAT_INC(chaintoolong, insert_failed); ret = NF_DROP; goto dying.
11. ct.timeout += nfct_time_stamp.
12. Conntrack::insert_prepare(ct).
13. Conntrack::hash_insert(ct, hash, reply_hash) — set IPS_CONFIRMED first, then add ORIGINAL + REPLY tuples to buckets.
14. Conntrack::double_unlock; local_bh_enable.
15. nf_conntrack_event_cache(IPCT_NEW, ct).
16. return NF_ACCEPT.
17. out: clash recovery via Conntrack::resolve_clash; possibly NF_DROP.
18. dying: Conntrack::double_unlock; local_bh_enable; return ret.

`Conntrack::init_conn(net, tmpl, tuple, skb, dataoff, hash) -> Result<*tuple_hash, i32>`:
1. if !Conntrack::invert_tuple(&repl, tuple): return Ok(NULL).
2. zone = nf_ct_zone_tmpl(tmpl, skb, &tmp).
3. ct = Conntrack::alloc(net, zone, tuple, &repl, GFP_ATOMIC, hash)?.
4. if !nf_ct_add_synproxy(ct, tmpl): NfConn::free(ct); return Err(-ENOMEM).
5. nf_ct_timeout_ext_add if tmpl has timeout-ext; acct + tstamp + labels exts; ecache ext if events.
6. cnet = nf_ct_pernet(net).
7. if cnet.expect_count > 0:
   - spin_lock_bh(&nf_conntrack_expect_lock).
   - exp = Expect::find(net, zone, tuple, !tmpl || nf_ct_is_confirmed(tmpl)).
   - if exp:
     - set_bit(IPS_EXPECTED_BIT, &ct.status). ct.master = exp.master.
     - if exp.helper: nf_ct_helper_ext_add(ct, GFP_ATOMIC); rcu_assign_pointer(help.helper, exp.helper).
     - ct.mark = READ_ONCE(exp.master.mark).
     - ct.secmark = exp.master.secmark.
     - NF_CT_STAT_INC(expect_new).
   - spin_unlock_bh.
8. if !exp ∧ tmpl: Helper::try_assign(ct, tmpl, GFP_ATOMIC).
9. smp_wmb.
10. refcount_set(&ct.ct_general.use, 1).
11. if exp:
    - if exp.expectfn: exp.expectfn(ct, exp).
    - nf_ct_expect_put(exp).
12. return Ok(&ct.tuplehash[IP_CT_DIR_ORIGINAL]).

`Conntrack::find_get(net, zone, tuple, hash) -> *tuple_hash`:
1. 'begin: rcu_read_lock.
2. sequence = read_seqcount_begin(&nf_conntrack_generation).
3. bucket = scale_hash(hash).
4. for h, n in hlist_nulls_for_each_entry_rcu(&nf_conntrack_hash[bucket]):
   - ct = nf_ct_tuplehash_to_ctrack(h).
   - if !nf_ct_key_equal(h, tuple, zone, net): continue.
   - if !refcount_inc_not_zero(&ct.ct_general.use): continue (concurrent free).
   - if !nf_ct_key_equal(h, tuple, zone, net): nf_ct_put(ct); continue (recycled under SLAB_TYPESAFE_BY_RCU).
   - rcu_read_unlock. return h.
5. if get_nulls_value(n) != bucket: goto 'begin (concurrent resize).
6. if read_seqcount_retry(&nf_conntrack_generation, sequence): goto 'begin.
7. rcu_read_unlock. return NULL.

`Conntrack::gc_worker(work)`:
1. gc_work = container_of(work, ConntrackGcWork, dwork.work).
2. i = gc_work.next_bucket.
3. if gc_work.early_drop: max95 = nf_conntrack_max * 0.95.
4. if i == 0: reset avg_timeout = GC_SCAN_INTERVAL_INIT; count = GC_SCAN_INITIAL_COUNT; start_time = nfct_time_stamp.
5. for each bucket until budget:
   - rcu_read_lock.
   - for h in bucket:
     - ct = nf_ct_tuplehash_to_ctrack(h).
     - if gc_worker_skip_ct(ct): continue.
     - if nf_ct_is_expired(ct): nf_ct_gc_expired(ct).
   - rcu_read_unlock.
6. if early_drop ∧ over max95: random_bucket victim drop.
7. schedule_delayed_work(&gc_work.dwork, next_run).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tuple_hash_both_directions` | INVARIANT | per-confirmed nf_conn: tuplehash[ORIG] and tuplehash[REPLY] both linked. |
| `refcount_zero_on_alloc` | INVARIANT | per-__nf_conntrack_alloc: refcount_set(0) before slab return; +1 only at attach. |
| `confirmed_after_seqlock` | INVARIANT | per-confirm: IPS_CONFIRMED set before hash_insert; hash_insert under double-bucket lock. |
| `nulls_marker_terminates_walk` | INVARIANT | per-find_get: hlist_nulls walk terminates on marker; mismatch ⟹ retry. |
| `slab_typesafe_recycle_safe` | INVARIANT | per-find_get: post-refcount-bump re-equal-check rejects recycled-slab races. |
| `gc_seq_paired_with_double_lock` | INVARIANT | per-confirm: nf_conntrack_double_lock retries on seqcount change. |
| `expect_lock_serializes_init_conn` | INVARIANT | per-init_conntrack: nf_conntrack_expect_lock held during exp lookup + bind. |
| `template_skb_short_circuits_in` | INVARIANT | per-nf_in: ctinfo==IP_CT_UNTRACKED ⟹ NF_ACCEPT before alloc. |

### Layer 2: TLA+

`net/netfilter/nf_conntrack.tla`:
- Per-skb FSM: receive → lookup → (hit | init_conntrack) → handle_packet → forward → confirm.
- Per-flow lifecycle: UNCONFIRMED → CONFIRMED → DYING.
- Properties:
  - `safety_no_double_insert` — per-flow: IPS_CONFIRMED transitions 0→1 exactly once.
  - `safety_lookup_finds_confirmed_only` — per-RCU lookup: returns only IPS_CONFIRMED tuples.
  - `safety_both_tuples_visible_together` — per-confirm: ORIGINAL + REPLY published before IPS_CONFIRMED observed.
  - `safety_dying_invisible_to_new_lookups` — per-DYING: refcount_inc_not_zero returns 0 → skipped.
  - `safety_expected_implies_master_alive` — per-flow with IPS_EXPECTED: master pinned via refcount.
  - `safety_helper_assigned_under_extlock` — per-flow: helper rcu_assign_pointer under expect_lock.
  - `liveness_gc_evicts_expired` — per-expired flow: gc_worker eventually evicts.
  - `liveness_confirm_terminates` — per-skb: confirm returns NF_ACCEPT ∨ NF_DROP.
  - `liveness_lookup_terminates_under_resize` — per-find_get: bounded retries.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Conntrack::nf_in` post: returns NF_ACCEPT ∨ NF_DROP; skb._nfct set iff NF_ACCEPT and flow valid | `Conntrack::nf_in` |
| `Conntrack::confirm` post: ret == NF_ACCEPT ⟹ IPS_CONFIRMED set ∧ both tuples in hash | `Conntrack::confirm` |
| `Conntrack::resolve_normal_ct` post: skb tagged with (ct, ctinfo) ∨ returns 0 | `Conntrack::resolve_normal_ct` |
| `Conntrack::init_conn` post: ct.ct_general.use == 1 ∧ ct.timeout == 0 (not yet refreshed) | `Conntrack::init_conn` |
| `NfConn::alloc` post: refcount == 0 ∧ status == 0 ∧ timeout == 0 | `NfConn::alloc` |
| `Conntrack::find_get` post: returned tuple_hash has refcount ≥ 1 ∧ key_equal | `Conntrack::find_get` |
| `Conntrack::hash_insert` post: IPS_CONFIRMED set ∧ both hnnode linked | `Conntrack::hash_insert` |
| `Expect::find` post: returned expect has refcount ≥ 1 ∧ master pinned | `Expect::find` |

### Layer 4: Verus/Creusot functional

`Per-skb PREROUTING → nf_conntrack_in → (resolve_normal_ct | handle_icmp) → init_conntrack (alloc + expect + helper) | find_get → handle_packet (per-proto state) → forward → POSTROUTING → __nf_conntrack_confirm (double-lock + hash insert + IPS_CONFIRMED)` semantic equivalence: per-Documentation/networking/nf_conntrack-sysctl.rst + Documentation/networking/conntrack/conntrack.txt.

`Per-helper expected flow: master.help.expectations registered → matching skb → nf_ct_find_expectation under expect_lock → init_conntrack inherits master/helper/mark → expectfn callback` semantic equivalence: per-RFC 959 (FTP), RFC 3261 (SIP) data-channel semantics implemented in helper modules.

## Hardening

(Inherits row-1 features from `net/netfilter/00-overview.md` § Hardening.)

Conntrack-core reinforcement:

- **Per-`nf_conntrack_max` table cap + early_drop** — defense against per-DoS connection-flood OOM.
- **Per-`MAX_CHAINLEN` chain-length cap + random offset** — defense against per-bucket hash-collision DoS.
- **Per-SLAB_TYPESAFE_BY_RCU slab + post-lookup re-equal-check** — defense against per-recycled-slab UAF in lockless lookup.
- **Per-`nf_conntrack_generation` seqcount paired with `nf_conntrack_double_lock`** — defense against per-resize torn insert.
- **Per-`hlist_nulls` walk with bucket-id marker** — defense against per-RCU walker landing in wrong bucket post-resize.
- **Per-`nf_conntrack_hash_rnd` siphash boot-key** — defense against per-userspace hash-flood (predictable tuples).
- **Per-IPS_CONFIRMED ordering: set before hash insert** — defense against per-lookup-observing-non-inserted flow.
- **Per-`refcount_inc_not_zero` lookup admission** — defense against per-DYING UAF.
- **Per-`nf_ct_ext_valid_pre / _post` extension checks** — defense against per-extension-realloc race.
- **Per-`nf_conntrack_expect_lock` serialization** — defense against per-expect lookup ↔ unlink race.
- **Per-`gc_worker` adaptive interval + bounded duration (`GC_SCAN_MAX_DURATION`)** — defense against per-GC-CPU-starvation.
- **Per-`__nf_ct_resolve_clash` NAT-clash recovery** — defense against per-double-allocation under SNAT collisions.
- **Per-helper `try_module_get` refcount on assign** — defense against per-helper-module unload UAF.
- **Per-template skb (IPS_TEMPLATE) not in hash** — defense against per-NOTRACK leaking into table.
- **Per-`nf_ct_iterate_cleanup` flush on netns exit** — defense against per-netns-conn leak.
- **Per-`nf_conntrack_hook` indirect vtable** — defense against per-symbol direct dependency outside CONFIG_NF_CONNTRACK.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — sysctl ingress (nf_conntrack_max, nf_conntrack_buckets, generic-timeout, helper) bounds-validated; ctnetlink batch attribute payloads validated before slab alloc.
- **PAX_KERNEXEC** — `nf_conntrack_hook`, `nf_ct_iterate_cleanup`, l4proto + helper vtables resident in R-X kernel text.
- **PAX_RANDKSTACK** — `nf_conntrack_in`, `__nf_conntrack_confirm`, `__nf_conntrack_find_get` stacks re-randomised so per-packet locals, hash-seed locals, and lookup keys are unpredictable.
- **PAX_REFCOUNT** — `nf_conn.ct_general.use` + `nf_conntrack_htable.hnnode` ref + per-net counter saturate-trap; conn-flood storms cannot wrap.
- **PAX_MEMORY_SANITIZE** — `nf_conn` slab + extensions (`nf_conn_help`, `nf_conn_nat`, `nf_conn_seqadj`, `nf_conn_labels`, `nf_conn_ecache`) freed via RCU + kmem_cache zeroed.
- **PAX_UDEREF** — every nlattr-driven mutation (CTA_TUPLE_*, CTA_PROTOINFO_*, CTA_NAT_*) validates `nla_len` before deref.
- **PAX_RAP/kCFI** — `nf_conntrack_hook` indirect-vtable, l4proto / l3proto / helper indirect calls all signed with conntrack CFI signature.
- **GRKERNSEC_HIDESYM** — ct + extension pointers, hash addresses never leak via /proc/net/nf_conntrack, ctnetlink dump, sysfs.
- **GRKERNSEC_DMESG** — table-full / invalid-state / helper-miss warnings rate-limited + dmesg-restricted.
- **Per-`nf_conn` `use` PAX_REFCOUNT** — defense against per-conn refcount wrap → UAF on confirm/destroy race.
- **Per-`__nf_conntrack_find_get` RCU read-lock + reference-pin** — defense against per-stale-tuple deref racing destroy.
- **Per-`nf_conntrack_max` clamp + early-drop** — defense against per-conn-table OOM under flood.
- **Per-`nf_ct_iterate_cleanup` netns-exit flush (already noted)** — defense against per-netns destroy leak.
- **Per-hash seed (`nf_conntrack_hash_rnd`) randomised at boot** — defense against per-precomputed-collision DoS.
- **Per-expectation (`nf_conntrack_expect`) ref + limit clamp** — defense against per-ALG-expectation table OOM.

Rationale: `nf_conn` lifecycle + the global conntrack hash are the most-attacked surface in netfilter (flood DoS, refcount UAF, helper-driven expectation OOM); PAX_REFCOUNT on `ct_general.use`, RCU on find_get, the netns-exit flush, and the boot-randomised hash seed are the load-bearing measures.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-protocol state machines (TCP/UDP/ICMP/SCTP/GRE) (covered in `conntrack-proto.md` Tier-3)
- Conntrack netlink CTA_* dump/notify (covered in `conntrack-netlink.md` Tier-3)
- Conntrack helpers L7 details (FTP/SIP/PPTP/IRC/…) (covered in `conntrack-helper.md` Tier-3)
- NAT integration (`nf_nat_setup_info`, `nf_nat_alloc_null_binding`, masquerade) (covered in `nat-core.md` Tier-3)
- Conntrack BPF programs (covered in `conntrack-bpf.md` Tier-3)
- nftables ct expression (`nft_ct.c`) (covered in `nft-core.md` Tier-3)
- tc act_ct flowtable offload (covered in `flowtable.md` and `act-ct.md` Tier-3)
- Implementation code
