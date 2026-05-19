# Tier-3: net/sched/cls_api.c — TC classifier API

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/00-overview.md
upstream-paths:
  - net/sched/cls_api.c (~4125 lines)
  - include/net/pkt_cls.h
  - include/net/sch_generic.h (tcf_proto, tcf_chain, tcf_block, tcf_proto_ops)
  - include/uapi/linux/pkt_cls.h (TCA_*, TC_ACT_*)
  - include/uapi/linux/rtnetlink.h (RTM_NEWTFILTER / DELTFILTER / GETTFILTER / NEWCHAIN / DELCHAIN / GETCHAIN)
-->

## Summary

The **TC classifier API** is the framework that links userspace `tc filter add ...` to per-qdisc classifier instances. Per-`struct tcf_proto`: one filter entity at a given `(prio, protocol)` slot on a `tcf_chain`, with private state owned by an installed classifier kind (`cls_u32`, `cls_flower`, `cls_bpf`, `cls_basic`, `cls_matchall`, `cls_route`, `cls_fw`, ...). Per-`struct tcf_chain`: ordered list of `tcf_proto`, identified by chain index inside a block. Per-`struct tcf_block`: collection of chains, shared-or-private bound to one-or-more qdiscs / classids, owns hardware-offload state (`flow_block`) and per-block control mutex. Per-`tcf_classify()` hot path: walks `chain->filter_chain` in prio order, dispatches per-`tcf_proto.classify` until a non-`TC_ACT_UNSPEC` is returned, handles `TC_ACT_GOTO_CHAIN` / `TC_ACT_RECLASSIFY` with reclassify-limit and `TC_SKB_EXT` miss-cookie continuation. Per-classifier registration via `register_tcf_proto_ops` / `unregister_tcf_proto_ops` against `tcf_proto_base`. Per-netlink: `RTM_NEWTFILTER` / `DELTFILTER` / `GETTFILTER` operate on `tcf_proto` instances; `RTM_NEWCHAIN` / `DELCHAIN` / `GETCHAIN` operate on chains and templates. Per-hardware-offload: each block accumulates `flow_block_cb` entries via `ndo_setup_tc(TC_SETUP_BLOCK)`, replayed on filter add/replace/destroy through `tc_setup_cb_*`. Per-chain-template: optional pre-declaration via `tmplt_create` / `tmplt_destroy` / `tmplt_dump` / `tmplt_reoffload` enforces one classifier kind on a chain. Critical for: TC fast-path correctness, share-block lifecycle, offload bind/unbind, replay-safe filter add under concurrent unlocked classifiers.

This Tier-3 covers `net/sched/cls_api.c` (~4125 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tcf_proto` | per-filter entity | `TcfProto` |
| `struct tcf_proto_ops` | per-classifier-kind vtable | `TcfProtoOps` |
| `struct tcf_chain` | per-chain filter list | `TcfChain` |
| `struct tcf_block` | per-block chain + offload owner | `TcfBlock` |
| `register_tcf_proto_ops()` | per-kind registration | `ClsApi::register_proto_ops` |
| `unregister_tcf_proto_ops()` | per-kind unregistration | `ClsApi::unregister_proto_ops` |
| `tcf_proto_create()` | per-tp allocate + init | `ClsApi::proto_create` |
| `tcf_proto_destroy()` | per-tp teardown | `ClsApi::proto_destroy` |
| `tcf_proto_lookup_ops()` | per-kind lookup + module-get | `ClsApi::lookup_ops` |
| `tcf_chain_create()` | per-chain alloc | `ClsApi::chain_create` |
| `tcf_chain_get()` / `_put()` | per-chain refcnt | `ClsApi::chain_get` / `_put` |
| `tcf_chain_get_by_act()` / `_put_by_act()` | per-action chain ref | `ClsApi::chain_get_by_act` / `_put_by_act` |
| `tcf_chain_flush()` | per-chain unwind | `ClsApi::chain_flush` |
| `tcf_block_create()` | per-block alloc | `ClsApi::block_create` |
| `tcf_block_lookup()` | per-net block idr lookup | `ClsApi::block_lookup` |
| `tcf_block_get()` / `_put()` | default-binding per-qdisc | `ClsApi::block_get` / `_put` |
| `tcf_block_get_ext()` / `_put_ext()` | shared-binding per-qdisc | `ClsApi::block_get_ext` / `_put_ext` |
| `tcf_block_refcnt_get()` / `_put()` | per-block refcount | `ClsApi::block_refcnt_get` / `_put` |
| `tcf_block_offload_bind()` / `_unbind()` | per-block offload ctl | `ClsApi::block_offload_bind` / `_unbind` |
| `tcf_block_setup()` / `_bind()` / `_unbind()` | per-flow_block_offload dispatch | `ClsApi::block_setup` |
| `tcf_classify()` | per-skb hot-path | `ClsApi::classify` |
| `__tcf_classify()` | per-skb chain walk | `ClsApi::classify_inner` |
| `tc_new_tfilter()` | per-RTM_NEWTFILTER | `ClsApi::new_tfilter` |
| `tc_del_tfilter()` | per-RTM_DELTFILTER | `ClsApi::del_tfilter` |
| `tc_get_tfilter()` | per-RTM_GETTFILTER | `ClsApi::get_tfilter` |
| `tc_dump_tfilter()` | per-RTM_GETTFILTER-dump | `ClsApi::dump_tfilter` |
| `tc_ctl_chain()` | per-RTM_*CHAIN | `ClsApi::ctl_chain` |
| `tc_dump_chain()` | per-RTM_GETCHAIN-dump | `ClsApi::dump_chain` |
| `tc_chain_tmplt_add()` / `_del()` | per-chain template | `ClsApi::chain_tmplt_add` / `_del` |
| `tc_setup_cb_call()` | per-offload single-callback dispatch | `ClsApi::setup_cb_call` |
| `tc_setup_cb_add()` / `_replace()` / `_destroy()` / `_reoffload()` | per-offload filter ops | `ClsApi::setup_cb_add` / `_replace` / `_destroy` / `_reoffload` |
| `tcf_exts_init_ex()` / `_destroy()` / `_validate()` / `_dump()` | per-filter action-list | `ClsApi::exts_init` / `_destroy` / `_validate` / `_dump` |
| `tcf_block_playback_offloads()` | per-cb replay over existing filters | `ClsApi::block_playback_offloads` |
| `tcf_queue_work()` | per-rcu deferred destroy wq | `ClsApi::queue_work` |
| `tcf_chain_tp_find()` / `_insert()` / `_remove()` / `_insert_unique()` | per-chain tp-list ops | `ClsApi::chain_tp_find` / `_insert` / `_remove` / `_insert_unique` |
| `tcf_get_next_chain()` / `_next_proto()` | per-iter helpers (locked) | `ClsApi::get_next_chain` / `_next_proto` |

## Compatibility contract

REQ-1: struct tcf_proto_ops (per-classifier-kind vtable):
- `head`: per-registry linked-list node on `tcf_proto_base`.
- `kind[IFNAMSIZ]`: unique kind name; duplicate-register fails -EEXIST.
- `classify(skb, tp, res) -> int`: per-skb fast-path, called under RCU-BH; returns TC_ACT_OK / SHOT / UNSPEC / RECLASSIFY / PIPE / TRAP / GOTO_CHAIN(...).
- `init(tp) -> int`: per-proto allocation hook.
- `destroy(tp, rtnl_held, extack)`: per-proto teardown.
- `get(tp, handle) -> *void`: per-filter lookup.
- `put(tp, fh)`: per-filter release.
- `change(net, skb, tp, base, handle, tca[], fh, flags, extack)`: per-add/replace filter.
- `delete(tp, arg, last, rtnl_held, extack)`: per-filter delete.
- `delete_empty(tp) -> bool`: true ⟹ tp now empty (DOIT_UNLOCKED kinds must implement).
- `walk(tp, walker, rtnl_held)`: per-filter iter.
- `reoffload(tp, add, cb, cb_priv, extack)`: per-cb replay-over-this-tp.
- `hw_add` / `hw_del`: per-existing-filter offload callbacks.
- `bind_class`: per-class-bind callback (cls_basic / cls_route).
- `tmplt_create` / `tmplt_destroy` / `tmplt_dump` / `tmplt_reoffload`: per-chain-template hooks (required iff template advertised).
- `get_exts(tp, handle) -> *tcf_exts`: per-filter action-list lookup (cookie-miss path).
- `dump` / `terse_dump`: per-filter netlink dump.
- `owner`: module owner.
- `flags`: TCF_PROTO_OPS_DOIT_UNLOCKED (drops rtnl during change/delete).

REQ-2: register_tcf_proto_ops(ops):
- write_lock(cls_mod_lock).
- for tp_ops in tcf_proto_base: if strcmp(kind, tp_ops.kind) == 0: -EEXIST.
- list_add_tail(ops.head, &tcf_proto_base).
- write_unlock.

REQ-3: unregister_tcf_proto_ops(ops):
- /* Drain RCU + workqueue from in-flight tp destroys */
- rcu_barrier().
- flush_workqueue(tc_filter_wq).
- write_lock(cls_mod_lock).
- for tp_ops in tcf_proto_base: if tp_ops == ops: list_del, rc = 0, break.
- write_unlock.
- WARN(rc != 0).

REQ-4: struct tcf_proto:
- `next`: per-chain rcu list next.
- `root`: per-classifier-kind private root.
- `classify`: cached `ops.classify` for fast path.
- `protocol`: __be16 (ETH_P_*, 0=ETH_P_ALL).
- `prio`: chain priority (TC_H_MAJ).
- `data`: per-classifier private.
- `ops`: tcf_proto_ops*.
- `chain`: parent tcf_chain.
- `lock`: per-tp spinlock for unlocked classifiers.
- `deleting`: per-cancel marker.
- `counted`: per-useswcnt accounting.
- `usesw`: !ops.reoffload (SW-only classifier).
- `refcnt`: per-tp refcount.
- `rcu`: per-rcu free.
- `destroy_ht_node`: per-block proto_destroy_ht for in-flight teardown collision detection.

REQ-5: struct tcf_chain:
- `filter_chain_lock`: per-chain mutex.
- `filter_chain`: rcu head of tcf_proto list.
- `list`: per-block chain_list node.
- `block`: parent tcf_block.
- `index`: per-block chain id (0 = chain0, head-change-callbacks).
- `refcnt`: per-chain total refs.
- `action_refcnt`: per-action-only refs (GOTO_CHAIN).
- `explicitly_created`: per-user RTM_NEWCHAIN bit.
- `flushing`: per-flush in-progress bit.
- `tmplt_ops` / `tmplt_priv`: per-template (NULL = none).

REQ-6: struct tcf_block:
- `ports`: xarray of net_device pointers (ingress/egress block tracker).
- `lock`: per-block control mutex.
- `chain_list`: per-block chain list.
- `index`: per-net block id (idx 0 = private).
- `classid`: parent class id.
- `refcnt`: per-block refcount.
- `net`: per-net_namespace owner.
- `q`: parent Qdisc (NULL for shared blocks).
- `cb_lock`: per-block rwsem protecting `flow_block` cb_list and offload counters.
- `flow_block`: HW offload bind/unbind state.
- `owner_list`: bound Qdiscs.
- `useswcnt`: per-block SW-classifier count (drives `tcf_sw_enabled_key`).
- `offloadcnt`: per-block HW-offloaded filter count.
- `nooffloaddevcnt` / `lockeddevcnt`: per-driver counters.
- `chain0.chain` / `.filter_chain_list`: per-chain0 head-change subscriber list.
- `proto_destroy_ht` / `proto_destroy_lock`: per-block hash of tcf_protos being destroyed (used by `tcf_proto_exists_destroying` to short-circuit re-add).

REQ-7: tcf_classify(skb, block, tp, res, compat_mode) -> int (per-skb hot-path, called under RCU-BH):
- /* TC_SKB_EXT continuation (CONFIG_NET_TC_SKB_EXT) */
- if block ∧ ext = skb_ext_find(skb, TC_SKB_EXT) has (chain ≠ 0 ∨ act_miss):
  - if ext.act_miss: n = tcf_exts_miss_cookie_lookup(ext.act_miss_cookie, &act_index); if !n: drop SKB_DROP_REASON_TC_COOKIE_ERROR; TC_ACT_SHOT.
  - else: chain = ext.chain.
  - fchain = tcf_chain_lookup_rcu(block, chain).
  - if !fchain: drop SKB_DROP_REASON_TC_CHAIN_NOTFOUND; TC_ACT_SHOT.
  - skb_ext_del(skb, TC_SKB_EXT).
  - tp = rcu_dereference_bh(fchain.filter_chain).
- ret = __tcf_classify(skb, tp, orig_tp, res, compat_mode, n, act_index, &last_executed_chain).
- /* Plant tc_skb_ext on miss */
- if tc_skb_ext_tc_enabled ∧ ret == TC_ACT_UNSPEC ∧ last_executed_chain ≠ 0:
  - tc_skb_ext_alloc(skb); ext.chain = last_executed_chain; copy mru/post_ct/zone.
- return ret.

REQ-8: __tcf_classify(skb, tp, orig_tp, res, compat_mode, n, act_index, *last_executed_chain) -> int:
- /* Per-NET_CLS_ACT reclassify limit */
- max_reclassify_loop = 16; limit = 0.
- for tp; tp; tp = rcu_dereference_bh(tp.next):
  - protocol = skb_protocol(skb, false).
  - if n: per-miss-cookie path:
    - if n.tp_prio ≠ tp.prio: continue.
    - if n.tp ≠ tp ∨ n.tp.chain ≠ n.chain ∨ !tp.ops.get_exts: SHOT (COOKIE_ERROR).
    - exts = tp.ops.get_exts(tp, n.handle).
    - if !exts ∨ n.exts ≠ exts: SHOT (COOKIE_ERROR).
    - n = NULL; err = tcf_exts_exec_ex(skb, exts, act_index, res).
  - else:
    - if tp.protocol ≠ protocol ∧ tp.protocol ≠ ETH_P_ALL: continue.
    - err = tc_classify(skb, tp, res).
  - if err == TC_ACT_RECLASSIFY ∧ !compat_mode: first_tp = orig_tp; *last_executed_chain = first_tp.chain.index; goto reset.
  - elif TC_ACT_EXT_CMP(err, TC_ACT_GOTO_CHAIN): first_tp = res.goto_tp; *last_executed_chain = err & TC_ACT_EXT_VAL_MASK; goto reset.
  - if err ≥ 0: return err.
- if n: SHOT (COOKIE_ERROR).
- return TC_ACT_UNSPEC.
- reset: if ++limit ≥ 16: net_notice_ratelimited; drop SKB_DROP_REASON_TC_RECLASSIFY_LOOP; SHOT. tp = first_tp; goto reclassify.

REQ-9: tcf_block_get_ext(p_block, q, ei, extack) -> int (shared-or-private block binding):
- /* Shared block: look up by index */
- if ei.block_index ≠ 0: block = tcf_block_refcnt_get(net, ei.block_index).
- if !block:
  - block = tcf_block_create(net, q, ei.block_index, extack).
  - if tcf_block_shared(block): tcf_block_insert(block, net, extack).
- tcf_block_owner_add(block, q, ei.binder_type).
- tcf_block_owner_netif_keep_dst(...).
- tcf_chain0_head_change_cb_add(block, ei, extack).
- tcf_block_offload_bind(block, q, ei, extack).
- if tcf_block_tracks_dev(block, ei): xa_insert(block.ports, dev.ifindex, dev, GFP_KERNEL).
- *p_block = block.

REQ-10: tcf_block_get(p_block, p_filter_chain, q, extack) -> int (default-binding):
- ei = { chain_head_change = tcf_chain_head_change_dflt, chain_head_change_priv = p_filter_chain }.
- tcf_block_get_ext(p_block, q, &ei, extack).

REQ-11: tcf_block_put_ext(block, q, ei) (per-shared block release):
- if !block: return.
- if tcf_block_tracks_dev(block, ei): xa_erase(block.ports, dev.ifindex).
- tcf_chain0_head_change_cb_del(block, ei).
- tcf_block_owner_del(block, q, ei.binder_type).
- __tcf_block_put(block, q, ei, true).

REQ-12: tcf_block_offload_bind / _unbind / _cmd:
- bo = { command, binder_type, cb_list, ... }.
- if dev.netdev_ops.ndo_setup_tc: err = dev.netdev_ops.ndo_setup_tc(dev, TC_SETUP_BLOCK, &bo); tcf_block_setup(block, &bo).
- else: flow_indr_dev_setup_offload(dev, sch, TC_SETUP_BLOCK, block, &bo, tc_block_indr_cleanup); tcf_block_setup(block, &bo).
- on bind: per-cb tcf_block_playback_offloads(block, cb, cb_priv, add=true, ...).
- on unbind: per-cb tcf_block_playback_offloads(block, cb, cb_priv, false, ...).

REQ-13: tc_new_tfilter(skb, n, extack) (per-RTM_NEWTFILTER netlink doit):
- replay: tp_created = 0.
- nlmsg_parse_deprecated(n, ..., rtm_tca_policy, extack).
- prio = TC_H_MAJ(t.tcm_info); protocol = TC_H_MIN(t.tcm_info); parent = t.tcm_parent.
- if prio == 0 ∧ NLM_F_CREATE: prio = TC_H_MAKE(0x80000000U, 0U); prio_allocate = true; else: -ENOENT.
- __tcf_qdisc_find(net, &q, &parent, t.tcm_ifindex, false, extack).
- tcf_proto_check_kind(tca[TCA_KIND], name).
- /* Decide rtnl-lock */
- if rtnl_held ∨ (q ∧ !DOIT_UNLOCKED) ∨ !tcf_proto_is_unlocked(name): rtnl_lock; rtnl_held=true.
- __tcf_qdisc_cl_find; block = __tcf_block_find(...).
- block.classid = parent.
- chain_index = nla_get_u32_default(tca[TCA_CHAIN], 0); if > TC_ACT_EXT_VAL_MASK: -EINVAL.
- chain = tcf_chain_get(block, chain_index, true).
- mutex_lock(&chain.filter_chain_lock).
- tp = tcf_chain_tp_find(chain, &chain_info, protocol, prio, prio_allocate, extack).
- if tp == NULL:
  - if chain.flushing: -EAGAIN.
  - if !tca[TCA_KIND] ∨ !protocol: -EINVAL.
  - if !NLM_F_CREATE: -ENOENT.
  - if prio_allocate: prio = tcf_auto_prio(tcf_chain_tp_prev(chain, &chain_info)).
  - mutex_unlock(filter_chain_lock).
  - tp_new = tcf_proto_create(name, protocol, prio, chain, rtnl_held, extack).
  - tp_created = 1; tp = tcf_chain_tp_insert_unique(chain, tp_new, protocol, prio, rtnl_held).
- else: mutex_unlock(filter_chain_lock).
- if tca[TCA_KIND] ∧ nla_strcmp(tca[TCA_KIND], tp.ops.kind): -EINVAL.
- fh = tp.ops.get(tp, t.tcm_handle).
- if fh ∧ NLM_F_EXCL: -EEXIST.
- if chain.tmplt_ops ∧ chain.tmplt_ops ≠ tp.ops: -EINVAL.
- flags |= NLM_F_CREATE? 0 : TCA_ACT_FLAGS_REPLACE.
- err = tp.ops.change(net, skb, tp, cl, t.tcm_handle, tca, &fh, flags, extack).
- if err == 0: tfilter_notify(..., RTM_NEWTFILTER, ...); tcf_proto_count_usesw(tp, true); if q: q.flags &= ~TCQ_F_CAN_BYPASS.
- on -EAGAIN: rtnl_held=true; goto replay.

REQ-14: tc_del_tfilter(skb, n, extack) (per-RTM_DELTFILTER netlink doit):
- parse n; prio = TC_H_MAJ(t.tcm_info); protocol = TC_H_MIN(t.tcm_info); parent = t.tcm_parent.
- if prio == 0 ∧ (protocol ∨ t.tcm_handle ∨ tca[TCA_KIND]): -ENOENT (cannot flush with these set).
- __tcf_qdisc_find; rtnl-decide as in REQ-13.
- block = __tcf_block_find(...); chain = tcf_chain_get(block, chain_index, false).
- per-prio == 0: flush entire chain via tcf_chain_flush(chain, rtnl_held); send RTM_DELTFILTER notify.
- per-fh-targeted: tp.ops.delete(tp, fh, &last, rtnl_held, extack); if last: tcf_chain_tp_delete_empty.

REQ-15: tc_get_tfilter(skb, n, extack) (per-RTM_GETTFILTER netlink doit, single):
- parse n; require prio ≠ 0.
- __tcf_qdisc_find; rtnl-decide; block = __tcf_block_find; chain = tcf_chain_get(block, chain_index, false).
- mutex_lock(filter_chain_lock); tp = tcf_chain_tp_find(chain, ..., protocol, prio, false, extack); mutex_unlock.
- if !tp: -ENOENT.
- if tca[TCA_KIND] ∧ nla_strcmp(tca[TCA_KIND], tp.ops.kind): -EINVAL.
- fh = tp.ops.get(tp, t.tcm_handle).
- if !fh: -ENOENT.
- tfilter_notify(net, skb, n, tp, block, q, parent, fh, RTM_NEWTFILTER, true, rtnl_held, NULL).

REQ-16: tc_dump_tfilter(skb, cb) (per-RTM_GETTFILTER netlink dumpit):
- nlmsg_parse_deprecated.
- locate block (shared via tcm_block_index or per-Qdisc cops->tcf_block).
- per-chain: tcf_chain_dump(chain, q, parent, skb, cb, index_start, &p_index, terse).
- per-tp in chain: cb->args bookkeeping for resume; tp.ops.walk(tp, &arg.w, true) calls tcf_node_dump → tcf_fill_node(RTM_NEWTFILTER, NLM_F_MULTI).

REQ-17: tc_ctl_chain(skb, n, extack) (per-RTM_NEWCHAIN / DELCHAIN / GETCHAIN, single dispatch):
- nlmsg_parse_deprecated; block = tcf_block_find(...); chain_index = nla_get_u32_default(tca[TCA_CHAIN], 0).
- mutex_lock(block.lock).
- chain = tcf_chain_lookup(block, chain_index).
- per-RTM_NEWCHAIN:
  - if chain ∧ tcf_chain_held_by_acts_only(chain): tcf_chain_hold (re-publish to user).
  - elif chain: -EEXIST.
  - elif !NLM_F_CREATE: -ENOENT.
  - else: chain = tcf_chain_create(block, chain_index); -ENOMEM on fail.
  - tcf_chain_hold(chain); chain.explicitly_created = true.
- per-RTM_DEL/GETCHAIN: if !chain ∨ held-by-acts-only: -EINVAL.
- mutex_unlock.
- per-RTM_NEWCHAIN: tc_chain_tmplt_add(chain, net, tca, extack); tc_chain_notify(..., RTM_NEWCHAIN, ...).
- per-RTM_DELCHAIN: tfilter_notify_chain(..., RTM_DELTFILTER); tcf_chain_flush(chain, true); tcf_chain_put_explicitly_created(chain).
- per-RTM_GETCHAIN: tc_chain_notify(chain, skb, n.nlmsg_seq, n.nlmsg_flags, RTM_NEWCHAIN, true, extack).
- on -EAGAIN: goto replay.

REQ-18: tc_setup_cb_call / _add / _replace / _destroy / _reoffload:
- /* Per-tp offload setup */
- tc_setup_cb_call(block, type, type_data, err_stop, rtnl_held): single broadcast over block.flow_block.cb_list; per-cb cb(type, type_data, cb_priv); accumulates ok_count.
- tc_setup_cb_add(block, tp, type, type_data, err_stop, in_hw_count, ...) -> int: install one filter; on success increments offloadcnt and tp.counted.
- tc_setup_cb_replace(...): atomic replace under cb_lock.
- tc_setup_cb_destroy(...): remove + decrement offloadcnt.
- tc_setup_cb_reoffload(block, tp, add, cb, cb_priv, type, type_data, in_hw_count, extack): replay one cb against one tp (used during bind/unbind playback).

REQ-19: tc_chain_tmplt_add / _del (per-chain template):
- tmplt_add: ops = tcf_proto_lookup_ops(name, true, extack); require ops.tmplt_create ∧ _destroy ∧ _dump ∧ _reoffload; tmplt_priv = ops.tmplt_create(net, chain, tca, extack); chain.tmplt_ops = ops; chain.tmplt_priv = tmplt_priv.
- tmplt_del: if !tmplt_ops: return; tmplt_ops.tmplt_destroy(tmplt_priv); module_put(tmplt_ops.owner).
- per-RTM_NEWTFILTER: enforce chain.tmplt_ops == tp.ops.

REQ-20: tcf_chain_get / _put refcount-balance:
- __tcf_chain_get(block, chain_index, create, by_act):
  - mutex_lock(block.lock); chain = tcf_chain_lookup(block, chain_index).
  - if chain: ++chain.refcnt; else if create: tcf_chain_create.
  - if by_act: ++chain.action_refcnt.
  - is_first_reference = (refcnt - action_refcnt == 1).
  - mutex_unlock.
  - if is_first_reference ∧ !by_act: tc_chain_notify(chain, ..., RTM_NEWCHAIN).
- __tcf_chain_put(chain, by_act, explicitly_created):
  - mutex_lock(block.lock).
  - if explicitly_created ∧ chain.explicitly_created: clear it.
  - if by_act: --action_refcnt.
  - refcnt = --chain.refcnt; non_act_refcnt = refcnt - action_refcnt.
  - if non_act_refcnt == explicitly_created ∧ !by_act ∧ non_act_refcnt == 0: tc_chain_notify_delete(...); chain.flushing = false.
  - if refcnt == 0: free_block = tcf_chain_detach(chain).
  - mutex_unlock.
  - if refcnt == 0: tc_chain_tmplt_del(...); tcf_chain_destroy(chain, free_block).

REQ-21: tcf_proto_destroy ordering:
- ops.destroy(tp, rtnl_held, extack).
- tcf_proto_count_usesw(tp, false) (drops block.useswcnt; static_branch_dec on last).
- if sig_destroy: tcf_proto_signal_destroyed(chain, tp) (removes from proto_destroy_ht).
- tcf_chain_put(chain).
- module_put(tp.ops.owner).
- kfree_rcu(tp).

REQ-22: pernet registration:
- struct tcf_net { spinlock_t idr_lock; struct idr idr; } stored per-net via tcf_net_id (pernet_subsys).
- tcf_block_insert / _remove use net.tn.idr for shared-block indices.
- tc_filter_init: alloc_ordered_workqueue("tc_filter_workqueue"); register_pernet_subsys(&tcf_net_ops); xa_init_flags(&tcf_exts_miss_cookies_xa, XA_FLAGS_ALLOC1); rtnl_register_many.
- subsys_initcall.

## Acceptance Criteria

- [ ] AC-1: register_tcf_proto_ops rejects duplicate `kind` with -EEXIST.
- [ ] AC-2: unregister_tcf_proto_ops drains via rcu_barrier + flush_workqueue before list_del.
- [ ] AC-3: tc_new_tfilter with NLM_F_CREATE on existing tp + NLM_F_EXCL returns -EEXIST.
- [ ] AC-4: tc_new_tfilter without NLM_F_CREATE on missing tp returns -ENOENT.
- [ ] AC-5: tc_del_tfilter with prio == 0 and (protocol ∨ handle ∨ kind) set returns -ENOENT.
- [ ] AC-6: tcf_classify reclassify loop limit = 16; on overflow returns TC_ACT_SHOT with SKB_DROP_REASON_TC_RECLASSIFY_LOOP.
- [ ] AC-7: TC_SKB_EXT cookie miss with no matching tp returns TC_ACT_SHOT with SKB_DROP_REASON_TC_COOKIE_ERROR.
- [ ] AC-8: tcf_chain_get auto-creates chain when `create=true`, returns existing on lookup-hit with refcnt++.
- [ ] AC-9: tcf_chain_put drops refcount; tcf_chain_destroy fires at refcnt==0; tc_chain_notify_delete fires on last non-act ref.
- [ ] AC-10: tcf_block_get_ext on `ei.block_index != 0` returns existing shared block; bumps refcnt.
- [ ] AC-11: tcf_block_offload_bind to dev with `tc_can_offload(dev) == false` and existing offloaded filters returns -EOPNOTSUPP.
- [ ] AC-12: tc_ctl_chain RTM_NEWCHAIN on existing user-chain returns -EEXIST; on acts-only chain re-publishes it.
- [ ] AC-13: tc_chain_tmplt_add fails -EOPNOTSUPP when classifier lacks tmplt_* hooks.
- [ ] AC-14: tc_chain_tmplt_add sets chain.tmplt_ops; tc_new_tfilter rejects mismatched kind with -EINVAL.
- [ ] AC-15: tfilter_notify on success emits RTM_NEWTFILTER over RTNLGRP_TC.
- [ ] AC-16: tcf_block_put drops last ref ⟹ tcf_block_destroy fires (mutex_destroy + xa_destroy + kfree_rcu).

## Architecture

```
struct TcfProtoOps {
  head: ListHead,
  kind: [u8; IFNAMSIZ],
  classify: fn(*SkBuff, *const TcfProto, *TcfResult) -> i32,
  init: fn(*TcfProto) -> i32,
  destroy: fn(*TcfProto, bool, *NetlinkExtAck),
  get: fn(*TcfProto, u32) -> *void,
  put: fn(*TcfProto, *void),
  change: fn(*Net, *SkBuff, *TcfProto, u64, u32, *[*Nlattr], **void, u32, *NetlinkExtAck) -> i32,
  delete: fn(*TcfProto, *void, *bool, bool, *NetlinkExtAck) -> i32,
  delete_empty: fn(*TcfProto) -> bool,
  walk: fn(*TcfProto, *TcfWalker, bool),
  reoffload: fn(*TcfProto, bool, FlowSetupCb, *void, *NetlinkExtAck) -> i32,
  hw_add: fn(*TcfProto, *void),
  hw_del: fn(*TcfProto, *void),
  bind_class: fn(*void, u32, u64, *void, u64),
  tmplt_create: fn(*Net, *TcfChain, *[*Nlattr], *NetlinkExtAck) -> *void,
  tmplt_destroy: fn(*void),
  tmplt_reoffload: fn(*TcfChain, bool, FlowSetupCb, *void),
  tmplt_dump: fn(*SkBuff, *Net, *void) -> i32,
  get_exts: fn(*const TcfProto, u32) -> *TcfExts,
  dump: fn(*Net, *TcfProto, *void, *SkBuff, *Tcmsg, bool) -> i32,
  terse_dump: fn(*Net, *TcfProto, *void, *SkBuff, *Tcmsg, bool) -> i32,
  owner: *Module,
  flags: i32,                       // TCF_PROTO_OPS_DOIT_UNLOCKED
}

struct TcfProto {
  next: *rcu TcfProto,
  root: *rcu void,
  classify: fn(*SkBuff, *const TcfProto, *TcfResult) -> i32,
  protocol: __be16,
  prio: u32,
  data: *void,
  ops: *const TcfProtoOps,
  chain: *TcfChain,
  lock: SpinLock,
  deleting: bool,
  counted: bool,
  usesw: bool,
  refcnt: Refcount,
  rcu: RcuHead,
  destroy_ht_node: HlistNode,
}

struct TcfChain {
  filter_chain_lock: Mutex,
  filter_chain: *rcu TcfProto,
  list: ListHead,
  block: *TcfBlock,
  index: u32,
  refcnt: u32,
  action_refcnt: u32,
  explicitly_created: bool,
  flushing: bool,
  tmplt_ops: *const TcfProtoOps,
  tmplt_priv: *void,
  rcu: RcuHead,
}

struct TcfBlock {
  ports: XArray,
  lock: Mutex,
  chain_list: ListHead,
  index: u32,
  classid: u32,
  refcnt: Refcount,
  net: *Net,
  q: *Qdisc,
  cb_lock: RwSemaphore,
  flow_block: FlowBlock,
  owner_list: ListHead,
  keep_dst: bool,
  useswcnt: Atomic,
  offloadcnt: Atomic,
  nooffloaddevcnt: u32,
  lockeddevcnt: u32,
  chain0_chain: *TcfChain,
  chain0_filter_chain_list: ListHead,
  rcu: RcuHead,
  proto_destroy_ht: Hashtable,
  proto_destroy_lock: Mutex,
}
```

`ClsApi::register_proto_ops(ops) -> i32`:
1. write_lock(cls_mod_lock).
2. for t in tcf_proto_base: if t.kind == ops.kind: rc = -EEXIST; goto out.
3. list_add_tail(&ops.head, &tcf_proto_base); rc = 0.
4. out: write_unlock; return rc.

`ClsApi::unregister_proto_ops(ops)`:
1. rcu_barrier().
2. flush_workqueue(tc_filter_wq).
3. write_lock(cls_mod_lock).
4. for t in tcf_proto_base: if t == ops: list_del; rc = 0; break.
5. write_unlock.
6. WARN(rc != 0).

`ClsApi::proto_create(kind, protocol, prio, chain, rtnl_held, extack) -> Result<*TcfProto>`:
1. tp = kzalloc(TcfProto); if !tp: return ERR(-ENOBUFS).
2. tp.ops = ClsApi::lookup_ops(kind, rtnl_held, extack); if IS_ERR(tp.ops): kfree; return.
3. tp.classify = tp.ops.classify.
4. tp.protocol = protocol; tp.prio = prio; tp.chain = chain.
5. tp.usesw = !tp.ops.reoffload.
6. spin_lock_init(&tp.lock); refcount_set(&tp.refcnt, 1).
7. err = tp.ops.init(tp); if err: module_put(ops.owner); kfree; return.
8. return tp.

`ClsApi::classify(skb, block, tp, res, compat_mode) -> i32` (per-skb hot-path):
1. /* TC_SKB_EXT continuation */
2. if block: ext = skb_ext_find(skb, TC_SKB_EXT).
3. if ext ∧ (ext.chain ≠ 0 ∨ ext.act_miss):
   - if ext.act_miss: n = tcf_exts_miss_cookie_lookup(ext.act_miss_cookie, &act_index); if !n: drop TC_COOKIE_ERROR; TC_ACT_SHOT.
   - else: chain = ext.chain.
   - fchain = tcf_chain_lookup_rcu(block, chain); if !fchain: drop TC_CHAIN_NOTFOUND; TC_ACT_SHOT.
   - skb_ext_del(skb, TC_SKB_EXT).
   - tp = rcu_dereference_bh(fchain.filter_chain); last_executed_chain = fchain.index.
4. ret = __tcf_classify(skb, tp, orig_tp, res, compat_mode, n, act_index, &last_executed_chain).
5. if tc_skb_ext_tc_enabled() ∧ ret == TC_ACT_UNSPEC ∧ last_executed_chain ≠ 0:
   - ext = tc_skb_ext_alloc(skb).
   - ext.chain = last_executed_chain; ext.mru = cb.mru; ext.post_ct = ...; ext.zone = cb.zone.
6. return ret.

`ClsApi::classify_inner(skb, tp, orig_tp, res, compat_mode, n, act_index, *last_executed_chain) -> i32`:
1. limit = 0; const MAX_RECLASSIFY_LOOP = 16.
2. reclassify: for tp; tp; tp = rcu_dereference_bh(tp.next):
3. protocol = skb_protocol(skb, false).
4. if n: per-miss-cookie validate; err = tcf_exts_exec_ex(skb, exts, act_index, res); n = NULL.
5. else: if tp.protocol ≠ protocol ∧ tp.protocol ≠ ETH_P_ALL: continue. err = tc_classify(skb, tp, res).
6. if err == TC_ACT_RECLASSIFY ∧ !compat_mode: first_tp = orig_tp; goto reset.
7. elif TC_ACT_EXT_CMP(err, TC_ACT_GOTO_CHAIN): first_tp = res.goto_tp; goto reset.
8. if err ≥ 0: return err.
9. if n: drop TC_COOKIE_ERROR; TC_ACT_SHOT.
10. return TC_ACT_UNSPEC.
11. reset: if ++limit ≥ 16: net_notice_ratelimited; drop TC_RECLASSIFY_LOOP; TC_ACT_SHOT. tp = first_tp; goto reclassify.

`ClsApi::block_get_ext(p_block, q, ei, extack) -> i32`:
1. dev = qdisc_dev(q); net = qdisc_net(q).
2. if ei.block_index ≠ 0: block = tcf_block_refcnt_get(net, ei.block_index).
3. if !block: block = tcf_block_create(net, q, ei.block_index, extack); if tcf_block_shared(block): tcf_block_insert(block, net, extack).
4. tcf_block_owner_add(block, q, ei.binder_type).
5. tcf_block_owner_netif_keep_dst(block, q, ei.binder_type).
6. tcf_chain0_head_change_cb_add(block, ei, extack).
7. tcf_block_offload_bind(block, q, ei, extack).
8. if tcf_block_tracks_dev(block, ei): xa_insert(&block.ports, dev.ifindex, dev, GFP_KERNEL).
9. *p_block = block; return 0.

`ClsApi::chain_get(block, chain_index, create) -> *TcfChain`:
1. mutex_lock(&block.lock).
2. chain = tcf_chain_lookup(block, chain_index).
3. if chain: ++chain.refcnt.
4. elif create: chain = tcf_chain_create(block, chain_index).
5. is_first_reference = (chain.refcnt - chain.action_refcnt == 1).
6. mutex_unlock.
7. if is_first_reference ∧ !by_act: tc_chain_notify(chain, NULL, 0, NLM_F_CREATE | NLM_F_EXCL, RTM_NEWCHAIN, false, NULL).
8. return chain.

`ClsApi::chain_flush(chain, rtnl_held)`:
1. mutex_lock(&chain.filter_chain_lock).
2. for tp in chain.filter_chain: tcf_proto_signal_destroying(chain, tp).
3. tp = chain.filter_chain; RCU_INIT_POINTER(chain.filter_chain, NULL); tcf_chain0_head_change(chain, NULL); chain.flushing = true.
4. mutex_unlock.
5. for tp in (snapshot): tcf_proto_put(tp, rtnl_held, NULL).

`ClsApi::new_tfilter(skb, n, extack) -> i32` (replay-safe):
1. replay: parse n; resolve qdisc, class, block, chain.
2. mutex_lock(&chain.filter_chain_lock); tp = tcf_chain_tp_find(chain, ..., protocol, prio, prio_allocate, extack).
3. if !tp ∧ NLM_F_CREATE: tp_new = tcf_proto_create(...); tp = tcf_chain_tp_insert_unique(chain, tp_new, ...).
4. fh = tp.ops.get(tp, t.tcm_handle); enforce NLM_F_EXCL, kind-match, tmplt-match.
5. err = tp.ops.change(net, skb, tp, cl, t.tcm_handle, tca, &fh, flags, extack).
6. on success: tfilter_notify(RTM_NEWTFILTER); tcf_proto_count_usesw(tp, true); q.flags &= ~TCQ_F_CAN_BYPASS.
7. on -EAGAIN: rtnl_held = true; goto replay.

`ClsApi::ctl_chain(skb, n, extack) -> i32` (RTM_*CHAIN dispatcher):
1. replay: parse n; block = tcf_block_find(...).
2. chain_index = nla_get_u32_default(tca[TCA_CHAIN], 0).
3. mutex_lock(&block.lock); chain = tcf_chain_lookup(block, chain_index).
4. match n.nlmsg_type:
   - RTM_NEWCHAIN: if chain ∧ acts_only: tcf_chain_hold; elif chain: -EEXIST; elif !NLM_F_CREATE: -ENOENT; else: tcf_chain_create.
   - RTM_DELCHAIN | RTM_GETCHAIN: if !chain ∨ acts_only: -EINVAL; else: tcf_chain_hold.
5. if RTM_NEWCHAIN: tcf_chain_hold(chain); chain.explicitly_created = true.
6. mutex_unlock(&block.lock).
7. match:
   - RTM_NEWCHAIN: tc_chain_tmplt_add; tc_chain_notify(RTM_NEWCHAIN).
   - RTM_DELCHAIN: tfilter_notify_chain(RTM_DELTFILTER); tcf_chain_flush(chain, true); tcf_chain_put_explicitly_created(chain).
   - RTM_GETCHAIN: tc_chain_notify(chain, skb, n.nlmsg_seq, n.nlmsg_flags, RTM_NEWCHAIN, true, extack).
8. on -EAGAIN: goto replay.

`ClsApi::setup_cb_add(block, tp, type, type_data, err_stop, in_hw_count, ...) -> i32`:
1. lockdep_assert_held(&block.cb_lock).
2. ok_count = 0.
3. for cb in block.flow_block.cb_list:
   - err = cb.cb(type, type_data, cb.cb_priv).
   - if err == -EOPNOTSUPP: continue (per-driver opt-out).
   - if err: if err_stop: rewind via tc_cls_offload_cnt_reset; return err.
   - ++ok_count; tc_cls_offload_cnt_update(block, ok_count, &tp.usesw_count, &flags).
4. if ok_count ∧ !tp.counted: tp.counted = true; ++block.offloadcnt; if first: static_branch_inc(&tcf_sw_enabled_key).
5. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `proto_ops_unique_kind` | INVARIANT | per-register_proto_ops: duplicate kind rejected -EEXIST. |
| `unregister_drains_rcu` | INVARIANT | per-unregister_proto_ops: rcu_barrier+flush_workqueue precede list_del. |
| `classify_reclassify_bounded` | INVARIANT | per-__tcf_classify: limit ≤ 16; overflow ⟹ SHOT(TC_RECLASSIFY_LOOP). |
| `classify_cookie_path_safe` | INVARIANT | per-__tcf_classify: cookie miss ∨ stale tp ⟹ SHOT(TC_COOKIE_ERROR). |
| `chain_refcount_balanced` | INVARIANT | per-tcf_chain_get/put: refcnt + action_refcnt accounting symmetric. |
| `block_refcount_balanced` | INVARIANT | per-tcf_block_get/put: shared blocks released only when refcnt==0. |
| `tmplt_required_hooks` | INVARIANT | per-tc_chain_tmplt_add: requires tmplt_{create,destroy,dump,reoffload} all ≠ NULL. |
| `tp_create_path_balanced` | INVARIANT | per-tcf_proto_create: ops.init failure ⟹ module_put + kfree. |
| `block_cb_lock_held` | INVARIANT | per-tcf_block_playback_offloads / setup_cb_*: block.cb_lock down_write held. |
| `chain_filter_chain_lock_held` | INVARIANT | per-tcf_chain_tp_insert/_remove: chain.filter_chain_lock held. |

### Layer 2: TLA+

`net/sched/cls-api.tla`:
- Per-NEWTFILTER + per-DELTFILTER + per-classify + per-RTM_*CHAIN + per-block-bind/unbind.
- Properties:
  - `safety_proto_kind_unique` — per-registry: at most one ops with given kind.
  - `safety_chain_refcount_nonneg` — per-chain: refcnt and action_refcnt ≥ 0 across get/put.
  - `safety_reclassify_terminates` — per-__tcf_classify: returns within 16 reset-loops.
  - `safety_tp_insert_under_lock` — per-tcf_chain_tp_insert: chain.filter_chain_lock held.
  - `safety_block_offload_consistent` — per-bind/unbind: cb_list mirrors driver state; offloadcnt aligns.
  - `liveness_replay_eventually_succeeds` — per-tc_new_tfilter on -EAGAIN with rtnl_held=true: replay terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ClsApi::register_proto_ops` post: list-add only on unique kind | `ClsApi::register_proto_ops` |
| `ClsApi::proto_create` post: tp.ops.init succeeded ∨ ERR_PTR returned | `ClsApi::proto_create` |
| `ClsApi::classify_inner` post: result ∈ {TC_ACT_OK, _SHOT, _UNSPEC, _PIPE, _TRAP, _STOLEN, _QUEUED, _GOTO_CHAIN<n>} | `ClsApi::classify_inner` |
| `ClsApi::chain_get` post: chain.refcnt ≥ 1; tc_chain_notify(RTM_NEWCHAIN) iff first non-act ref | `ClsApi::chain_get` |
| `ClsApi::block_get_ext` post: *p_block ≠ NULL ∨ Err; block bound to q via owner_list | `ClsApi::block_get_ext` |
| `ClsApi::block_offload_bind` post: offload bound ∨ block.nooffloaddevcnt incremented | `ClsApi::block_offload_bind` |
| `ClsApi::new_tfilter` post: success ⟹ RTM_NEWTFILTER notify sent ∧ TCQ_F_CAN_BYPASS cleared | `ClsApi::new_tfilter` |
| `ClsApi::ctl_chain RTM_DELCHAIN` post: chain.flushing toggled then released | `ClsApi::ctl_chain` |
| `ClsApi::setup_cb_add` post: success ⟹ offloadcnt incremented exactly once | `ClsApi::setup_cb_add` |

### Layer 4: Verus/Creusot functional

`Per-userspace-tc-filter-add → tc_new_tfilter → tcf_block_find → tcf_chain_get → tcf_chain_tp_find → tcf_proto_create → tp.ops.change → tfilter_notify(RTM_NEWTFILTER)` semantic equivalence: per-Documentation/networking/tc-actions-env-rules.rst + iproute2 tc filter add interoperability.

`Per-skb-classify → tcf_classify → __tcf_classify(chain walk + reclassify + GOTO_CHAIN) → tcf_exts_exec_ex` semantic equivalence: per-include/net/pkt_cls.h tcf_classify contract.

## Hardening

(Inherits row-1 features from `net/sched/00-overview.md` § Hardening.)

cls-api reinforcement:

- **Per-classifier registry unique-kind** — defense against per-kind-aliasing module load.
- **Per-unregister rcu_barrier + flush_workqueue** — defense against per-classify-during-unregister UAF.
- **Per-tcf_proto refcount strict** — defense against per-tp UAF in concurrent change/delete.
- **Per-block cb_lock rwsem** — defense against per-offload-list racing with classify and ndo_setup_tc.
- **Per-chain filter_chain_lock mutex** — defense against per-chain tp-list torn writes.
- **Per-reclassify limit 16** — defense against per-attacker-induced reclassify livelock.
- **Per-TC_SKB_EXT cookie validation (tp + chain + exts triple-check)** — defense against per-HW-cookie-replay after tp-replace.
- **Per-tcf_block_offload_in_use guard on bind** — defense against per-offload-disabled-dev mismatch.
- **Per-tmplt_ops kind-locking on chain** — defense against per-chain-kind-confusion.
- **Per-replay loop on -EAGAIN with rtnl_held escalation** — defense against per-DOIT_UNLOCKED races during flush.
- **Per-block proto_destroy_ht collision check** — defense against per-tp re-add during in-flight destroy.
- **Per-NL_F_EXCL enforced on existing fh** — defense against per-RTM_NEWTFILTER silent overwrite.
- **Per-CAP_NET_ADMIN on RTM_NEWTFILTER / DELTFILTER / NEWCHAIN / DELCHAIN** — defense against per-unprivileged datapath manipulation (enforced upstream by rtnetlink core for non-RTM_GET*).
- **Per-shared-block xarray-keyed lookup** — defense against per-block-index-collision across nets.
- **Per-tcf_chain_held_by_acts_only gate on RTM_GETCHAIN** — defense against per-action-only-chain leak to user.
- **Per-flow_block_cb registration confined to block.cb_lock writer** — defense against per-callback-list torn-write.

## Grsecurity/PaX-style Reinforcement

Baseline kernel-wide hardening leveraged by `cls-api`:

- **PAX_USERCOPY** — `tcf_proto`, `tcf_chain`, `tcf_block`, and `tcf_exts` allocations live in PAX_USERCOPY-whitelisted slab caches; `RTM_GETTFILTER`/`RTM_GETCHAIN` dump cannot bleed adjacent kernel state.
- **PAX_KERNEXEC** — `tcf_proto_ops`, `tcf_chain_ops`, and the `flow_block_cb` ops vtables live in `__ro_after_init`; corrupted classifier cannot pivot via forged ops.
- **PAX_RANDKSTACK** — `tc_new_tfilter`, `tcf_classify`, `tc_get_chain` stack frames randomized on each netlink/softirq entry.
- **PAX_REFCOUNT** — `chain->refcnt`/`action_refcnt`, `block->refcnt`, `tp->refcnt`, and `tcf_exts->actions[]` refs use saturating `refcount_t`; replay/add-del churn cannot wrap.
- **PAX_MEMORY_SANITIZE** — `tcf_block_release`, `tcf_chain_destroy`, and `tcf_proto_destroy` zero per-block/chain/tp state on RCU free.
- **PAX_UDEREF** — netlink attr parsing operates on kernel-side copies; no user-pointer deref in fast path.
- **PAX_RAP/kCFI** — `tcf_proto_ops.classify`/`init`/`destroy`/`change`/`get`/`walk` and `flow_block_cb` dispatch forward-edge-checked.
- **GRKERNSEC_HIDESYM** — `RTM_GETTFILTER`/`RTM_GETCHAIN` strip kernel pointers (`tp`, `block`, `chain`).
- **GRKERNSEC_DMESG** — proto_destroy_ht collision and chain-template parse-fail `printk_ratelimited` capped.

cls-api-specific reinforcement:

- **CAP_NET_ADMIN required** for `RTM_NEWTFILTER`/`DELTFILTER`/`NEWCHAIN`/`DELCHAIN`; enforced by rtnetlink core for non-RTM_GET*.
- **PAX_REFCOUNT on `chain->refcnt`/`action_refcnt`** — rapid chain-template add/del cannot wrap.
- **Shared-block xarray-keyed lookup** under PAX_USERCOPY whitelist — cross-net block-index collision blocked.
- **`flow_block_cb` registration confined to `block.cb_lock` writer** under PAX_KERNEXEC `__ro_after_init` for ops — callback-list torn-write blocked.
- **`tcf_chain_held_by_acts_only` gate on `RTM_GETCHAIN`** — action-only chain not leaked to userspace.
- **PAX_RAP on `tcf_proto_ops.classify`** — softirq classify dispatch type-checked, blocks type-confusion across cls kinds.

Rationale: cls-api is the netlink front door to every TC classifier; CAP_NET_ADMIN + PAX_REFCOUNT on chain/block/tp refs + PAX_RAP on the classify vtable closes the cross-classifier UAF and type-confusion classes.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/sched/act_api.c TC actions (covered in `act-api.md` Tier-3)
- net/sched/sch_api.c qdiscs (covered in `sch-api.md` Tier-3)
- Per-classifier kinds (cls_u32, cls_flower, cls_bpf, cls_basic, cls_matchall, cls_route, cls_fw): covered in `cls-u32.md`, `cls-flower.md`, `cls-bpf.md`, `cls-matchall.md` Tier-3s
- net/core/flow_offload.c flow_block_cb infrastructure (covered separately)
- include/net/tc_act/* per-action userspace types
- Implementation code
