# Tier-3: net/sched/act_api.c — TC actions API

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/00-overview.md
upstream-paths:
  - net/sched/act_api.c (~2304 lines)
  - include/net/act_api.h
  - include/net/pkt_cls.h (tcf_exts, tcf_exts_exec)
  - include/uapi/linux/pkt_cls.h (TC_ACT_*, TCA_ACT_*)
  - include/uapi/linux/rtnetlink.h (RTM_NEWACTION / DELACTION / GETACTION)
-->

## Summary

The **TC actions API** is the framework that owns the post-classify per-skb side-effect list executed when a `tcf_proto` returns its `tcf_result`. Per-`struct tc_action`: one action instance keyed by `tcfa_index` inside an action-kind-specific idr (`act_gact`, `act_mirred`, `act_pedit`, `act_csum`, `act_nat`, `act_skbedit`, `act_vlan`, `act_tunnel_key`, `act_bpf`, `act_police`, `act_sample`, `act_ct`, `act_mpls`, `act_gate`, ...). Per-`struct tc_action_ops`: per-kind vtable carrying `act`, `init`, `dump`, `cleanup`, `walk`, `lookup`, `offload_act_setup`, `get_dev`, `get_psample_group`, `stats_update`. Per-`struct tc_action_net`: per-netns + per-kind `tcf_idrinfo` storing the action idr and pernet ops id (used during `tcf_action_reoffload_cb` to walk all installed actions per kind). Per-`tcf_action_exec(skb, actions[], nr, res)` hot path: walks `actions` array, dispatches per-`a->ops->act` with TC_ACT_REPEAT (≤32-deep), TC_ACT_JUMP (jmp_prgcnt; TCA_ACT_MAX_PRIO ttl), TC_ACT_GOTO_CHAIN (cross-chain jump via `a->goto_chain`), TC_ACT_PIPE (continue pipeline), TC_ACT_SHOT/STOLEN/QUEUED (break). Per-action registration via `tcf_register_action` / `tcf_unregister_action` against `act_base` + `act_pernet_id_list`. Per-netlink: `RTM_NEWACTION` / `DELACTION` / `GETACTION` operate on standalone action instances; classifiers populate via `tcf_action_init` from a TCA_ACT_TAB nested attribute. Per-bind: actions can be bound (created-by-classifier with `tcfa_bindcnt`) or standalone (created via RTM_NEWACTION with `tcfa_refcnt` only). Per-offload: standalone actions hot-offloaded via `tcf_action_offload_add` / `_del` against per-driver `flow_indr_dev_setup_offload`; reoffload-callback `tcf_action_reoffload_cb` walks all `tc_action_net` idrs on cb registration. Per-cookie: actions carry an optional opaque `user_cookie` (RCU-protected). Critical for: action pipeline correctness, idr-based id allocation, ref + bind count balance, offload reoffload-callback iteration.

This Tier-3 covers `net/sched/act_api.c` (~2304 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tc_action` | per-instance | `TcAction` |
| `struct tc_action_ops` | per-kind vtable | `TcActionOps` |
| `struct tc_action_net` | per-net + per-kind idr-holder | `TcActionNet` |
| `struct tcf_idrinfo` | per-kind idr container | `TcfIdrInfo` |
| `tcf_register_action()` | per-kind registration | `ActApi::register_action` |
| `tcf_unregister_action()` | per-kind unregistration | `ActApi::unregister_action` |
| `tc_lookup_action_n()` | per-kind lookup by string | `ActApi::lookup_action_n` |
| `tc_lookup_action()` | per-kind lookup by nla | `ActApi::lookup_action` |
| `tcf_action_init()` | per-classifier-binding init batch | `ActApi::action_init` |
| `tcf_action_init_1()` | per-instance init | `ActApi::action_init_1` |
| `tc_action_load_ops()` | per-kind module load + ops lookup | `ActApi::load_ops` |
| `tcf_action_exec()` | per-skb pipeline | `ActApi::action_exec` |
| `tcf_action_destroy()` | per-batch destroy | `ActApi::action_destroy` |
| `tcf_action_cleanup()` | per-instance teardown | `ActApi::action_cleanup` |
| `__tcf_action_put()` | per-instance refcount dec | `ActApi::action_put` |
| `__tcf_idr_release()` | per-instance idr-release | `ActApi::idr_release` |
| `tcf_idr_release()` | per-instance idr-release public | `ActApi::idr_release_pub` |
| `tcf_idr_create()` | per-instance allocation | `ActApi::idr_create` |
| `tcf_idr_create_from_flags()` | per-instance allocation with flags | `ActApi::idr_create_from_flags` |
| `tcf_idr_check_alloc()` | per-instance pre-alloc / lookup | `ActApi::idr_check_alloc` |
| `tcf_idr_cleanup()` | per-instance error rollback | `ActApi::idr_cleanup` |
| `tcf_idr_insert_many()` | per-batch commit | `ActApi::idr_insert_many` |
| `tcf_idr_search()` | per-index lookup | `ActApi::idr_search` |
| `tcf_idr_delete_index()` | per-instance delete | `ActApi::idr_delete_index` |
| `tcf_idrinfo_destroy()` | per-kind net-exit teardown | `ActApi::idrinfo_destroy` |
| `tcf_generic_walker()` | per-kind walk dispatcher | `ActApi::generic_walker` |
| `tcf_dump_walker()` / `tcf_del_walker()` | per-kind dump / del walker | `ActApi::dump_walker` / `del_walker` |
| `tc_ctl_action()` | per-RTM_*ACTION dispatch | `ActApi::ctl_action` |
| `tcf_action_add()` | per-RTM_NEWACTION | `ActApi::action_add` |
| `tcf_del_notify()` | per-RTM_DELACTION dispatch | `ActApi::del_notify` |
| `tcf_add_notify()` | per-RTM_NEWACTION notify | `ActApi::add_notify` |
| `tca_action_gd()` | per-get/del shared body | `ActApi::action_gd` |
| `tca_action_flush()` | per-flush-by-kind | `ActApi::action_flush` |
| `tc_dump_action()` | per-RTM_GETACTION dumpit | `ActApi::dump_action` |
| `tcf_action_dump()` / `_dump_1()` / `_dump_terse()` | per-instance dump | `ActApi::action_dump` / `_dump_1` / `_dump_terse` |
| `tcf_action_offload_add()` / `_del()` | per-instance offload bind | `ActApi::action_offload_add` / `_del` |
| `tcf_action_reoffload_cb()` | per-cb replay over all-instances | `ActApi::action_reoffload_cb` |
| `tcf_action_update_hw_stats()` | per-instance hw-stat refresh | `ActApi::action_update_hw_stats` |
| `tcf_action_update_stats()` | per-instance sw/hw stat increment | `ActApi::action_update_stats` |
| `tcf_action_copy_stats()` | per-instance stats dump | `ActApi::action_copy_stats` |
| `tcf_action_check_ctrlact()` | per-act ctrl validation (GOTO_CHAIN) | `ActApi::action_check_ctrlact` |
| `tcf_action_set_ctrlact()` | per-act ctrl install | `ActApi::action_set_ctrlact` |

## Compatibility contract

REQ-1: struct tc_action_ops (per-action-kind vtable):
- `head`: per-registry linked-list node on `act_base`.
- `kind[IFNAMSIZ]`: unique action name (`"gact"`, `"mirred"`, ...); duplicate-register fails -EEXIST.
- `id`: enum tca_id; must match kind.
- `net_id`: per-pernet ops id.
- `size`: sizeof per-kind tc_action subtype.
- `owner`: module ptr.
- `act(skb, a, res) -> int`: per-skb hot path; called under RCU-BH; returns TC_ACT_OK / SHOT / PIPE / STOLEN / QUEUED / REPEAT / JUMP / GOTO_CHAIN / TRAP / UNSPEC.
- `dump(skb, a, bind, ref) -> int`: per-instance netlink dump.
- `cleanup(a)`: per-instance teardown (called from `tcf_action_cleanup`).
- `lookup(net, a, index) -> int`: per-index lookup (returns 1 on hit).
- `init(net, nla, est, act, tp, flags, extack) -> int`: per-instance init from netlink.
- `walk(net, skb, cb, type, ops, extack) -> int`: per-kind dump/del walker dispatch.
- `stats_update(a, bytes, packets, drops, lastuse, hw)`: per-instance hw-stat refresh.
- `get_fill_size(a) -> size_t`: per-instance dump-size estimate.
- `get_dev(a, *destructor) -> *net_device`: per-instance redirect-target (mirred).
- `get_psample_group(a, *destructor) -> *psample_group`: per-instance psample target (sample).
- `offload_act_setup(act, entry_data, *index_inc, bind, extack) -> int`: per-instance hw-offload-entry filler (called from flow_action build).

REQ-2: tcf_register_action(act, ops):
- validate: act.act ∧ act.dump ∧ act.init must be ≠ NULL; else -EINVAL.
- register_pernet_subsys(ops); if fail: return err.
- if ops.id: tcf_pernet_add_id_list(*ops.id); if -EEXIST: unwind unregister_pernet_subsys.
- write_lock(act_mod_lock).
- for a in act_base: if act.id == a.id ∨ strcmp(act.kind, a.kind) == 0: -EEXIST.
- list_add_tail(act.head, &act_base).
- write_unlock.

REQ-3: tcf_unregister_action(act, ops):
- write_lock(act_mod_lock).
- for a in act_base: if a == act: list_del; rc = 0; break.
- write_unlock.
- if !rc: if ops.id: tcf_pernet_del_id_list(*ops.id); unregister_pernet_subsys(ops).
- return rc.

REQ-4: struct tc_action:
- `ops`: *const tc_action_ops.
- `type`: per-TCA_OLD_COMPAT backward compat marker.
- `idrinfo`: *tcf_idrinfo (parent per-kind idr container).
- `tcfa_index`: u32 (per-kind unique idr key).
- `tcfa_refcnt`: refcount_t (total references).
- `tcfa_bindcnt`: atomic_t (per-classifier bind references).
- `tcfa_action`: int (per-instance control: TC_ACT_OK / SHOT / GOTO_CHAIN(...) base).
- `tcfa_tm`: install / lastuse / firstuse / expires jiffies.
- `tcfa_bstats` / `tcfa_bstats_hw`: per-instance sw / hw basic stats.
- `tcfa_drops` / `tcfa_overlimits`: per-instance qstat counters.
- `tcfa_rate_est`: per-net_rate_estimator (RCU).
- `tcfa_lock`: per-instance spinlock.
- `cpu_bstats` / `cpu_bstats_hw` / `cpu_qstats`: per-cpu fast-path stats (alloc-toggle via TCA_ACT_FLAGS_NO_PERCPU_STATS).
- `user_cookie`: *rcu tc_cookie (opaque blob ≤ TC_COOKIE_MAX_SIZE).
- `goto_chain`: *rcu tcf_chain (for TC_ACT_GOTO_CHAIN).
- `tcfa_flags`: TCA_ACT_FLAGS_* (POLICE, BIND, REPLACE, NO_RTNL, AT_INGRESS, AT_INGRESS_OR_CLSACT, SKIP_HW, SKIP_SW, NO_PERCPU_STATS).
- `hw_stats` / `used_hw_stats` / `used_hw_stats_valid`: per-instance hw-stat reporting.
- `in_hw_count`: per-instance offload-cb count.

REQ-5: struct tc_action_net + tcf_idrinfo (per-net + per-kind):
- tcf_idrinfo:
  - `lock`: mutex (protects action_idr).
  - `action_idr`: idr keyed by tcfa_index, value = *tc_action.
  - `net`: *net.
- tc_action_net:
  - `idrinfo`: *tcf_idrinfo (allocated per-net at tc_action_net_init).
  - `ops`: *const tc_action_ops.
- act_pernet_id_list: global list of per-kind pernet ops ids (mutex act_id_mutex) used by tcf_action_reoffload_cb.

REQ-6: tcf_action_exec(skb, actions[], nr_actions, res) -> int (per-skb hot path, RCU-BH):
- jmp_prgcnt = 0; jmp_ttl = TCA_ACT_MAX_PRIO; ret = TC_ACT_OK.
- if skb_skip_tc_classify(skb): return TC_ACT_OK.
- restart_act_graph:
- for i in 0..nr_actions:
  - a = actions[i].
  - if jmp_prgcnt > 0: jmp_prgcnt -= 1; continue.
  - if tc_act_skip_sw(a.tcfa_flags): continue.
  - repeat_ttl = 32.
  - repeat: ret = tc_act(skb, a, res).
  - if ret == TC_ACT_REPEAT: if --repeat_ttl ≠ 0: goto repeat; net_warn_ratelimited; return TC_ACT_OK.
  - if TC_ACT_EXT_CMP(ret, TC_ACT_JUMP):
    - jmp_prgcnt = ret & TCA_ACT_MAX_PRIO_MASK.
    - if !jmp_prgcnt ∨ jmp_prgcnt > nr_actions: return TC_ACT_OK (faulty opcode).
    - jmp_ttl -= 1; if jmp_ttl > 0: goto restart_act_graph; else: return TC_ACT_OK (faulty graph).
  - elif TC_ACT_EXT_CMP(ret, TC_ACT_GOTO_CHAIN):
    - if !rcu_access_pointer(a.goto_chain): drop SKB_DROP_REASON_TC_CHAIN_NOTFOUND; return TC_ACT_SHOT.
    - tcf_action_goto_chain_exec(a, res); /* sets res.goto_tp = chain.filter_chain */.
  - if ret ≠ TC_ACT_PIPE: break.
- return ret.

REQ-7: tcf_action_check_ctrlact(action, tp, *newchain, extack):
- opcode = TC_ACT_EXT_OPCODE(action).
- if !opcode: if action > TC_ACT_VALUE_MAX: -EINVAL; else: 0.
- elif opcode ≤ TC_ACT_EXT_OPCODE_MAX ∨ action == TC_ACT_UNSPEC: 0.
- else: -EINVAL.
- if TC_ACT_EXT_CMP(action, TC_ACT_GOTO_CHAIN):
  - chain_index = action & TC_ACT_EXT_VAL_MASK.
  - if !tp ∨ !newchain: -EINVAL.
  - *newchain = tcf_chain_get_by_act(tp.chain.block, chain_index).
  - if !*newchain: -ENOMEM.

REQ-8: tcf_action_set_ctrlact(a, action, goto_chain):
- a.tcfa_action = action.
- goto_chain = rcu_replace_pointer(a.goto_chain, goto_chain, 1) (returns old goto_chain).

REQ-9: tcf_idr_check_alloc(tn, *index, **a, bind) — slot reservation:
- if *index ≠ 0:
  - rcu_read_lock; p = idr_find(action_idr, *index).
  - if IS_ERR(p): -EAGAIN (slot reserved but action not yet installed).
  - if !p: /* empty slot */ max = *index; rcu_read_unlock; goto new.
  - if !refcount_inc_not_zero(&p.tcfa_refcnt): -EAGAIN.
  - if bind: atomic_inc(&p.tcfa_bindcnt).
  - *a = p; rcu_read_unlock; return 1 (existing).
- else: *index = 1; max = UINT_MAX.
- new: *a = NULL; mutex_lock(&idrinfo.lock); ret = idr_alloc_u32(action_idr, ERR_PTR(-EBUSY), index, max, GFP_KERNEL); mutex_unlock.
- if ret == -ENOSPC ∧ *index == max: -EAGAIN.
- return ret.

REQ-10: tcf_idr_create(tn, index, est, **a, ops, bind, cpustats, flags):
- p = kzalloc(ops.size, GFP_KERNEL).
- refcount_set(&p.tcfa_refcnt, 1); if bind: atomic_set(&p.tcfa_bindcnt, 1).
- if cpustats: alloc cpu_bstats / cpu_bstats_hw / cpu_qstats; -ENOMEM on any failure.
- gnet_stats_basic_sync_init; spin_lock_init; p.tcfa_index = index; p.tcfa_tm.{install,lastuse} = jiffies; p.tcfa_flags = flags.
- if est: gen_new_estimator(&p.tcfa_bstats, p.cpu_bstats, &p.tcfa_rate_est, &p.tcfa_lock, false, est).
- p.idrinfo = idrinfo; __module_get(ops.owner); p.ops = ops.
- *a = p; return 0.

REQ-11: tcf_idr_insert_many(actions[], init_res[]) (batch commit):
- for a in actions: if init_res[i] == ACT_P_BOUND: continue.
- mutex_lock(&a.idrinfo.lock).
- idr_replace(&idrinfo.action_idr, a, a.tcfa_index) — replaces ERR_PTR(-EBUSY) sentinel.
- mutex_unlock.

REQ-12: __tcf_action_put(p, bind) — refcount decrement:
- if bind: atomic_dec(&p.tcfa_bindcnt).
- if refcount_dec_and_test(&p.tcfa_refcnt):
  - idr_remove(&p.idrinfo.action_idr, p.tcfa_index).
  - tcf_action_cleanup(p) (per-ops cleanup → free_tcf).
  - return 1.
- return 0.

REQ-13: __tcf_idr_release(p, bind, strict):
- if !bind ∧ strict ∧ atomic_read(&p.tcfa_bindcnt) > 0: -EPERM (cannot remove bound action via act API).
- if __tcf_action_put(p, bind): return ACT_P_DELETED.
- return 0.

REQ-14: tcf_action_init_1(net, tp, nla, est, a_o, *init_res, flags, extack) (per-instance init):
- parse nla via tcf_action_policy (TCA_ACT_KIND, _INDEX, _COOKIE, _OPTIONS, _FLAGS, _HW_STATS).
- if TCA_ACT_COOKIE: user_cookie = nla_memdup_cookie(tb).
- hw_stats = tcf_action_hw_stats_get(tb[TCA_ACT_HW_STATS]).
- if tb[TCA_ACT_FLAGS]: parse nla_bitfield32; tc_act_flags_valid(flags).
- err = a_o.init(net, tb[TCA_ACT_OPTIONS], est, &a, tp, flags, extack).
- on success: a.user_cookie = user_cookie; a.hw_stats = hw_stats.

REQ-15: tcf_action_init(net, tp, nla, est, actions[], init_res[], *attr_size, flags, fl_flags, extack) (batch init for one TCA_ACT_TAB):
- nla_parse_nested_deprecated tb up to TCA_ACT_MAX_PRIO + 1; if tb[TCA_ACT_MAX_PRIO + 1]: -EINVAL (too many).
- /* Two-pass: load ops, then init each */
- for i in 1..=TCA_ACT_MAX_PRIO ∧ tb[i]: ops[i-1] = tc_action_load_ops(tb[i], flags, extack).
- for i in 1..=TCA_ACT_MAX_PRIO ∧ tb[i]:
  - act = tcf_action_init_1(net, tp, tb[i], est, ops[i-1], &init_res[i-1], flags, extack).
  - sz += tcf_action_fill_size(act).
  - actions[i-1] = act.
  - per-bind: enforce skip_sw/skip_hw consistency with filter flags.
  - per-standalone: tcf_action_offload_add(act, extack).
- tcf_idr_insert_many(actions, init_res).
- *attr_size = tcf_action_full_attrs_size(sz).
- on err: tcf_action_destroy(actions, flags & TCA_ACT_FLAGS_BIND).
- for each loaded ops: module_put(ops.owner).

REQ-16: tc_ctl_action(skb, n, extack) (per-RTM_*ACTION single dispatch):
- if n.nlmsg_type ≠ RTM_GETACTION ∧ !netlink_capable(skb, CAP_NET_ADMIN): -EPERM.
- nlmsg_parse_deprecated → tca[TCA_ROOT_*].
- require tca[TCA_ACT_TAB] ≠ NULL.
- match n.nlmsg_type:
  - RTM_NEWACTION: flags = NLM_F_REPLACE ? TCA_ACT_FLAGS_REPLACE : 0; tcf_action_add(net, tca[TCA_ACT_TAB], n, portid, flags, extack).
  - RTM_DELACTION: tca_action_gd(net, tca[TCA_ACT_TAB], n, portid, RTM_DELACTION, extack).
  - RTM_GETACTION: tca_action_gd(net, tca[TCA_ACT_TAB], n, portid, RTM_GETACTION, extack).
  - default: BUG().

REQ-17: tcf_action_add(net, nla, n, portid, flags, extack):
- for loop in 0..10:
  - ret = tcf_action_init(net, NULL, nla, NULL, actions, init_res, &attr_size, flags, 0, extack).
  - if ret ≠ -EAGAIN: break.
- if ret < 0: return ret.
- tcf_add_notify(net, n, actions, portid, attr_size, extack) → rtnetlink_maybe_send(RTM_NEWACTION, RTNLGRP_TC).
- tca_put_bound_many(actions, init_res) — only releases bound actions (init_res==ACT_P_BOUND).

REQ-18: tca_action_gd(net, nla, n, portid, event, extack):
- nla_parse_nested_deprecated → tb[TCA_ACT_MAX_PRIO + 1].
- if event == RTM_DELACTION ∧ NLM_F_ROOT: if tb[1]: tca_action_flush(net, tb[1], n, portid, extack); else: -EINVAL.
- for i in 1..=TCA_ACT_MAX_PRIO ∧ tb[i]:
  - act = tcf_action_get_1(net, tb[i], n, portid, extack).
  - actions[i-1] = act; attr_size += tcf_action_fill_size(act).
- if event == RTM_GETACTION: tcf_get_notify(net, portid, n, actions, event, extack).
- else: tcf_del_notify(net, n, actions, portid, attr_size, extack) → tcf_action_delete + rtnetlink_maybe_send(RTM_DELACTION).

REQ-19: tca_action_flush(net, nla, n, portid, extack) (RTM_DELACTION | NLM_F_ROOT):
- nla_parse_nested → tb; ops = tc_lookup_action(tb[TCA_ACT_KIND]); if !ops: -EINVAL.
- nlmsg_put(RTM_DELACTION); nla_nest_start(TCA_ACT_TAB).
- __tcf_generic_walker(net, skb, &dcb, RTM_DELACTION, ops, extack) → tcf_del_walker iterates the action_idr.
- nla_nest_end; nlh.nlmsg_flags |= NLM_F_ROOT; module_put(ops.owner).
- rtnetlink_send(skb, net, portid, RTNLGRP_TC, NLM_F_ECHO).

REQ-20: tc_dump_action(skb, cb) (RTM_GETACTION dumpit):
- nlmsg_parse_deprecated → tb[TCA_ROOT_*].
- kind = find_dump_kind(tb); if !kind: return 0.
- a_o = tc_lookup_action(kind); if !a_o: return 0.
- parse TCA_ROOT_FLAGS (TCA_ACT_FLAG_LARGE_DUMP_ON | TCA_ACT_FLAG_TERSE_DUMP) into cb.args[2].
- parse TCA_ROOT_TIME_DELTA → msecs_since → jiffy_since.
- nlmsg_put(RTM_GETACTION); reserve TCA_ROOT_COUNT; nla_nest_start(TCA_ACT_TAB).
- __tcf_generic_walker(net, skb, cb, RTM_GETACTION, a_o, NULL) → tcf_dump_walker.
- on success: nla_nest_end; memcpy TCA_ROOT_COUNT.

REQ-21: tcf_action_offload_add(action, extack) (per-instance hw offload):
- fl_action = kzalloc(flow_offload_action); offload_action_init(fl_action, action, ...).
- tcf_action_offload_cmd_ex(fl_action, &in_hw_count) → flow_indr_dev_setup_offload(NULL, NULL, TC_SETUP_ACT, fl_action, ...).
- on success: offload_action_hw_count_set(action, in_hw_count).
- on EOPNOTSUPP: if tc_act_skip_sw(action.tcfa_flags): error.

REQ-22: tcf_action_reoffload_cb(cb, cb_priv, add) (per-cb replay over all installed actions, all nets):
- down_read(&net_rwsem); mutex_lock(&act_id_mutex).
- for each net for_each_net(net):
  - for id_ptr in act_pernet_id_list:
    - tn = net_generic(net, id_ptr.id); idrinfo = tn.idrinfo.
    - mutex_lock(&idrinfo.lock).
    - idr_for_each_entry_ul(action_idr, p, tmp, id):
      - if IS_ERR(p) ∨ tc_act_bind(p.tcfa_flags): continue.
      - if add: tcf_action_offload_add_ex(p, NULL, cb, cb_priv).
      - else: tcf_action_offload_del_ex(p, cb, cb_priv); if skip_sw ∧ !in_hw: tcf_reoffload_del_notify(net, p).
    - mutex_unlock(&idrinfo.lock).
- mutex_unlock(&act_id_mutex); up_read(&net_rwsem).

REQ-23: tcf_action_dump_1 (per-instance dump):
- tcf_action_dump_terse(skb, a, false) → TCA_ACT_KIND, stats, cookie.
- nla_put_bitfield32(TCA_ACT_HW_STATS), TCA_ACT_USED_HW_STATS.
- flags = a.tcfa_flags & TCA_ACT_FLAGS_USER_MASK; nla_put_bitfield32(TCA_ACT_FLAGS).
- nla_put_u32(TCA_ACT_IN_HW_COUNT, a.in_hw_count).
- nla_nest_start(TCA_ACT_OPTIONS); err = a.ops.dump(skb, a, bind, ref); nla_nest_end.

REQ-24: pernet:
- struct tc_action_net { tcf_idrinfo *idrinfo; const tc_action_ops *ops; }, one per-net per-kind via tc_action_net_init.
- act_pernet_id_list: global mutex-protected list of pernet ops ids; used to walk all kinds during reoffload.
- subsys_initcall(tc_action_init) → rtnl_register_many({RTM_NEWACTION, _DELACTION, _GETACTION}).

## Acceptance Criteria

- [ ] AC-1: tcf_register_action rejects duplicate kind / id with -EEXIST.
- [ ] AC-2: tcf_register_action requires act.act ∧ act.dump ∧ act.init non-NULL (-EINVAL).
- [ ] AC-3: tcf_unregister_action removes from act_base, deletes pernet-id from act_pernet_id_list, unregister_pernet_subsys.
- [ ] AC-4: tcf_action_exec TC_ACT_REPEAT loop ≤ 32; on overflow: net_warn_ratelimited; return TC_ACT_OK.
- [ ] AC-5: tcf_action_exec TC_ACT_JUMP with jmp_prgcnt == 0 ∨ > nr_actions: return TC_ACT_OK.
- [ ] AC-6: tcf_action_exec TC_ACT_JUMP TTL = TCA_ACT_MAX_PRIO; on overflow: TC_ACT_OK.
- [ ] AC-7: tcf_action_exec TC_ACT_GOTO_CHAIN with NULL goto_chain: drop SKB_DROP_REASON_TC_CHAIN_NOTFOUND; TC_ACT_SHOT.
- [ ] AC-8: tcf_idr_check_alloc with index==0: allocs index 1..UINT_MAX from idr.
- [ ] AC-9: tcf_idr_check_alloc reservation IS_ERR(-EBUSY) ⟹ -EAGAIN (concurrent insert).
- [ ] AC-10: __tcf_idr_release with strict ∧ bindcnt > 0: -EPERM.
- [ ] AC-11: tc_ctl_action RTM_NEWACTION without CAP_NET_ADMIN: -EPERM.
- [ ] AC-12: tc_ctl_action with NULL TCA_ACT_TAB: -EINVAL.
- [ ] AC-13: tcf_action_init batch: more than TCA_ACT_MAX_PRIO actions in TCA_ACT_TAB ⟹ -EINVAL.
- [ ] AC-14: tcf_action_add retries up to 10 times on -EAGAIN.
- [ ] AC-15: tcf_action_reoffload_cb walks act_pernet_id_list under act_id_mutex and net_rwsem read.

## Architecture

```
struct TcActionOps {
  head: ListHead,
  kind: [u8; IFNAMSIZ],
  id: TcaId,
  net_id: u32,
  size: usize,
  owner: *Module,
  act: fn(*SkBuff, *const TcAction, *TcfResult) -> i32,
  dump: fn(*SkBuff, *TcAction, i32, i32) -> i32,
  cleanup: fn(*TcAction),
  lookup: fn(*Net, **TcAction, u32) -> i32,
  init: fn(*Net, *Nlattr, *Nlattr, **TcAction, *TcfProto, u32, *NetlinkExtAck) -> i32,
  walk: fn(*Net, *SkBuff, *NetlinkCallback, i32, *const TcActionOps, *NetlinkExtAck) -> i32,
  stats_update: fn(*TcAction, u64, u64, u64, u64, bool),
  get_fill_size: fn(*const TcAction) -> usize,
  get_dev: fn(*const TcAction, **TcActionPrivDestructor) -> *NetDevice,
  get_psample_group: fn(*const TcAction, **TcActionPrivDestructor) -> *PsampleGroup,
  offload_act_setup: fn(*TcAction, *void, *u32, bool, *NetlinkExtAck) -> i32,
}

struct TcAction {
  ops: *const TcActionOps,
  type_: u32,                          // TCA_OLD_COMPAT
  idrinfo: *TcfIdrInfo,
  tcfa_index: u32,
  tcfa_refcnt: Refcount,
  tcfa_bindcnt: Atomic,
  tcfa_action: i32,
  tcfa_tm: TcfT,
  tcfa_bstats: GnetStatsBasicSync,
  tcfa_bstats_hw: GnetStatsBasicSync,
  tcfa_drops: Atomic,
  tcfa_overlimits: Atomic,
  tcfa_rate_est: *rcu NetRateEstimator,
  tcfa_lock: SpinLock,
  cpu_bstats: *per_cpu GnetStatsBasicSync,
  cpu_bstats_hw: *per_cpu GnetStatsBasicSync,
  cpu_qstats: *per_cpu GnetStatsQueue,
  user_cookie: *rcu TcCookie,
  goto_chain: *rcu TcfChain,
  tcfa_flags: u32,                     // TCA_ACT_FLAGS_*
  hw_stats: u8,
  used_hw_stats: u8,
  used_hw_stats_valid: bool,
  in_hw_count: u32,
}

struct TcfIdrInfo {
  lock: Mutex,
  action_idr: Idr,
  net: *Net,
}

struct TcActionNet {
  idrinfo: *TcfIdrInfo,
  ops: *const TcActionOps,
}
```

`ActApi::register_action(act, ops) -> i32`:
1. if !act.act ∨ !act.dump ∨ !act.init: return -EINVAL.
2. register_pernet_subsys(ops); if fail: return err.
3. if ops.id: err = tcf_pernet_add_id_list(*ops.id); on err: unregister_pernet_subsys; return.
4. write_lock(act_mod_lock).
5. for a in act_base: if act.id == a.id ∨ strcmp(act.kind, a.kind) == 0: ret = -EEXIST; goto err_out.
6. list_add_tail(&act.head, &act_base).
7. write_unlock; return 0.
8. err_out: write_unlock; if ops.id: tcf_pernet_del_id_list(*ops.id); unregister_pernet_subsys(ops); return ret.

`ActApi::unregister_action(act, ops) -> i32`:
1. write_lock(act_mod_lock).
2. for a in act_base: if a == act: list_del; err = 0; break.
3. write_unlock.
4. if err == 0: if ops.id: tcf_pernet_del_id_list(*ops.id); unregister_pernet_subsys(ops).
5. return err.

`ActApi::action_exec(skb, actions[], nr_actions, res) -> i32` (per-skb hot path):
1. jmp_prgcnt = 0; jmp_ttl = TCA_ACT_MAX_PRIO; ret = TC_ACT_OK.
2. if skb_skip_tc_classify(skb): return TC_ACT_OK.
3. restart_act_graph: for i in 0..nr_actions:
4.   a = actions[i].
5.   if jmp_prgcnt > 0: jmp_prgcnt -= 1; continue.
6.   if tc_act_skip_sw(a.tcfa_flags): continue.
7.   repeat_ttl = 32.
8.   repeat: ret = tc_act(skb, a, res).
9.   if ret == TC_ACT_REPEAT: if --repeat_ttl ≠ 0: goto repeat; net_warn_ratelimited; return TC_ACT_OK.
10.  if TC_ACT_EXT_CMP(ret, TC_ACT_JUMP): jmp_prgcnt = ret & TCA_ACT_MAX_PRIO_MASK; if !jmp_prgcnt ∨ > nr_actions: return TC_ACT_OK; jmp_ttl -= 1; if jmp_ttl > 0: goto restart_act_graph; else: return TC_ACT_OK.
11.  elif TC_ACT_EXT_CMP(ret, TC_ACT_GOTO_CHAIN): if !rcu_access_pointer(a.goto_chain): drop TC_CHAIN_NOTFOUND; return TC_ACT_SHOT. tcf_action_goto_chain_exec(a, res).
12.  if ret ≠ TC_ACT_PIPE: break.
13. return ret.

`ActApi::action_init_1(net, tp, nla, est, a_o, *init_res, flags, extack) -> Result<*TcAction>`:
1. parse nla via tcf_action_policy → tb.
2. if tb[TCA_ACT_COOKIE]: user_cookie = nla_memdup_cookie(tb); if !user_cookie: -ENOMEM.
3. hw_stats = tcf_action_hw_stats_get(tb[TCA_ACT_HW_STATS]).
4. if tb[TCA_ACT_FLAGS]: parse bitfield32; validate via tc_act_flags_valid.
5. err = a_o.init(net, tb[TCA_ACT_OPTIONS], est, &a, tp, flags, extack).
6. on err: kfree user_cookie; return ERR.
7. a.user_cookie = user_cookie; a.hw_stats = hw_stats; return a.

`ActApi::action_init(net, tp, nla, est, actions[], init_res[], *attr_size, flags, fl_flags, extack) -> i32`:
1. err = nla_parse_nested_deprecated(tb, TCA_ACT_MAX_PRIO + 1, nla, NULL, extack).
2. if tb[TCA_ACT_MAX_PRIO + 1]: -EINVAL (only TCA_ACT_MAX_PRIO supported).
3. for i in 1..=TCA_ACT_MAX_PRIO ∧ tb[i]: ops[i-1] = tc_action_load_ops(tb[i], flags, extack).
4. for i in 1..=TCA_ACT_MAX_PRIO ∧ tb[i]:
   - act = ActApi::action_init_1(net, tp, tb[i], est, ops[i-1], &init_res[i-1], flags, extack).
   - sz += tcf_action_fill_size(act); actions[i-1] = act.
   - if tc_act_bind(flags): enforce skip_sw/skip_hw vs filter flags; inherit if both compat case.
   - else: tcf_action_offload_add(act, extack); if skip_sw ∧ err: rollback.
5. tcf_idr_insert_many(actions, init_res).
6. *attr_size = tcf_action_full_attrs_size(sz).
7. on err: tcf_action_destroy(actions, flags & TCA_ACT_FLAGS_BIND).
8. for ops: module_put(ops.owner).

`ActApi::idr_check_alloc(tn, *index, **a, bind) -> i32`:
1. if *index ≠ 0:
   - rcu_read_lock; p = idr_find(action_idr, *index).
   - if IS_ERR(p): rcu_read_unlock; return -EAGAIN.
   - if !p: max = *index; rcu_read_unlock; goto new.
   - if !refcount_inc_not_zero(&p.tcfa_refcnt): rcu_read_unlock; return -EAGAIN.
   - if bind: atomic_inc(&p.tcfa_bindcnt).
   - *a = p; rcu_read_unlock; return 1.
2. else: *index = 1; max = UINT_MAX.
3. new: *a = NULL; mutex_lock(&idrinfo.lock); ret = idr_alloc_u32(action_idr, ERR_PTR(-EBUSY), index, max, GFP_KERNEL); mutex_unlock.
4. if ret == -ENOSPC ∧ *index == max: -EAGAIN.
5. return ret.

`ActApi::idr_create(tn, index, est, **a, ops, bind, cpustats, flags) -> i32`:
1. p = kzalloc(ops.size, GFP_KERNEL); if !p: -ENOMEM.
2. refcount_set(&p.tcfa_refcnt, 1); if bind: atomic_set(&p.tcfa_bindcnt, 1).
3. if cpustats: alloc cpu_bstats / cpu_bstats_hw / cpu_qstats (unwind via err4 → err1).
4. gnet_stats_basic_sync_init; spin_lock_init(&p.tcfa_lock).
5. p.tcfa_index = index; p.tcfa_tm.{install,lastuse} = jiffies; p.tcfa_flags = flags.
6. if est: gen_new_estimator(&p.tcfa_bstats, p.cpu_bstats, &p.tcfa_rate_est, &p.tcfa_lock, false, est).
7. p.idrinfo = idrinfo; __module_get(ops.owner); p.ops = ops; *a = p; return 0.

`ActApi::ctl_action(skb, n, extack) -> i32`:
1. if n.nlmsg_type ≠ RTM_GETACTION ∧ !netlink_capable(skb, CAP_NET_ADMIN): -EPERM.
2. nlmsg_parse_deprecated → tca.
3. if tca[TCA_ACT_TAB] == NULL: -EINVAL.
4. match n.nlmsg_type:
   - RTM_NEWACTION: flags = (NLM_F_REPLACE) ? TCA_ACT_FLAGS_REPLACE : 0; ActApi::action_add(net, tca[TCA_ACT_TAB], n, portid, flags, extack).
   - RTM_DELACTION | RTM_GETACTION: ActApi::action_gd(net, tca[TCA_ACT_TAB], n, portid, n.nlmsg_type, extack).
   - default: BUG().

`ActApi::action_add(net, nla, n, portid, flags, extack) -> i32`:
1. for loop in 0..10:
   - ret = ActApi::action_init(net, NULL, nla, NULL, actions, init_res, &attr_size, flags, 0, extack).
   - if ret ≠ -EAGAIN: break.
2. if ret < 0: return ret.
3. ret = ActApi::add_notify(net, n, actions, portid, attr_size, extack).
4. tca_put_bound_many(actions, init_res).
5. return ret.

`ActApi::action_reoffload_cb(cb, cb_priv, add) -> i32`:
1. if !cb: -EINVAL.
2. down_read(&net_rwsem); mutex_lock(&act_id_mutex).
3. for_each_net(net):
4.   for id_ptr in act_pernet_id_list:
5.     tn = net_generic(net, id_ptr.id); if !tn: continue.
6.     idrinfo = tn.idrinfo; if !idrinfo: continue.
7.     mutex_lock(&idrinfo.lock).
8.     idr_for_each_entry_ul(action_idr, p, tmp, id):
9.       if IS_ERR(p) ∨ tc_act_bind(p.tcfa_flags): continue.
10.      if add: tcf_action_offload_add_ex(p, NULL, cb, cb_priv).
11.      else: ret = tcf_action_offload_del_ex(p, cb, cb_priv); if tc_act_skip_sw(p.tcfa_flags) ∧ !tc_act_in_hw(p): tcf_reoffload_del_notify(net, p).
12.    mutex_unlock(&idrinfo.lock).
13. mutex_unlock(&act_id_mutex); up_read(&net_rwsem); return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `action_ops_unique_kind_and_id` | INVARIANT | per-register_action: duplicate kind or id rejected -EEXIST. |
| `register_action_required_hooks` | INVARIANT | per-register_action: act, dump, init must all be ≠ NULL. |
| `action_exec_repeat_bounded` | INVARIANT | per-action_exec: TC_ACT_REPEAT loop ≤ 32. |
| `action_exec_jump_bounded` | INVARIANT | per-action_exec: TC_ACT_JUMP TTL ≤ TCA_ACT_MAX_PRIO. |
| `action_exec_goto_chain_safe` | INVARIANT | per-action_exec: NULL goto_chain ⟹ SHOT(TC_CHAIN_NOTFOUND). |
| `idr_check_alloc_busy_marker` | INVARIANT | per-idr_check_alloc: ERR_PTR(-EBUSY) reservation observed ⟹ -EAGAIN. |
| `idr_release_strict_bindcnt` | INVARIANT | per-__tcf_idr_release: strict ∧ bindcnt > 0 ⟹ -EPERM. |
| `action_put_idr_remove_on_zero` | INVARIANT | per-__tcf_action_put: refcnt == 0 ⟹ idr_remove + cleanup. |
| `action_init_max_prio` | INVARIANT | per-action_init: > TCA_ACT_MAX_PRIO actions in TCA_ACT_TAB ⟹ -EINVAL. |
| `reoffload_cb_locks_held` | INVARIANT | per-action_reoffload_cb: net_rwsem read + act_id_mutex + idrinfo.lock held during idr walk. |

### Layer 2: TLA+

`net/sched/act-api.tla`:
- Per-RTM_NEWACTION + per-RTM_DELACTION + per-action_exec + per-bind/unbind + per-reoffload.
- Properties:
  - `safety_action_kind_unique` — per-registry: at most one ops with given kind ∨ id.
  - `safety_action_refcount_balanced` — per-instance: refcnt + bindcnt ≥ 0; cleanup at refcnt==0 only.
  - `safety_action_exec_terminates` — per-action_exec: returns within 32×REPEAT × TCA_ACT_MAX_PRIO×JUMP steps.
  - `safety_idr_alloc_atomic` — per-idr_check_alloc: index reservation ERR_PTR(-EBUSY) → idr_replace (insert) ∨ idr_remove (cleanup).
  - `safety_standalone_offload_consistent` — per-reoffload_cb: bound actions excluded.
  - `liveness_action_add_retries` — per-action_add: -EAGAIN retries up to 10 then resolves.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ActApi::register_action` post: list-add only on unique kind/id and validated hooks | `ActApi::register_action` |
| `ActApi::action_exec` post: result ∈ {TC_ACT_OK, _SHOT, _PIPE, _STOLEN, _QUEUED, _REPEAT-resolved, _JUMP-resolved, _GOTO_CHAIN<n>} | `ActApi::action_exec` |
| `ActApi::idr_check_alloc` post: 1 ⟹ existing p with refcnt++; 0 ⟹ ERR_PTR(-EBUSY) reserved at *index; <0 ⟹ -EAGAIN/-ENOSPC | `ActApi::idr_check_alloc` |
| `ActApi::idr_create` post: tcfa_refcnt == 1 ∧ tcfa_bindcnt == (bind?1:0) ∧ ops module reffed | `ActApi::idr_create` |
| `ActApi::action_init_1` post: a.user_cookie installed iff TCA_ACT_COOKIE present; ops.init invoked | `ActApi::action_init_1` |
| `ActApi::action_add` post: tcf_add_notify RTM_NEWACTION emitted; bound-only released | `ActApi::action_add` |
| `ActApi::action_gd RTM_DELACTION` post: tcf_action_delete invoked; RTM_DELACTION notify emitted | `ActApi::action_gd` |
| `ActApi::action_reoffload_cb` post: each non-bind action visited once per cb (add) ∨ once per cb (del) | `ActApi::action_reoffload_cb` |

### Layer 4: Verus/Creusot functional

`Per-userspace-tc-action-add → tc_ctl_action(RTM_NEWACTION) → tcf_action_add → tcf_action_init (loop ≤ 10 on -EAGAIN) → tcf_action_init_1 → a_o.init → tcf_idr_insert_many → tcf_add_notify(RTM_NEWACTION)` semantic equivalence: per-iproute2 tc actions add interoperability.

`Per-classify-result → tcf_action_exec → per-a.ops.act → TC_ACT_{REPEAT,JUMP,GOTO_CHAIN,PIPE,...} resolution` semantic equivalence: per-include/uapi/linux/pkt_cls.h TC_ACT_* + Documentation/networking/tc-actions-env-rules.rst.

## Hardening

(Inherits row-1 features from `net/sched/00-overview.md` § Hardening.)

act-api reinforcement:

- **Per-CAP_NET_ADMIN on RTM_NEWACTION / DELACTION** — defense against per-unprivileged action installation.
- **Per-required-hooks act/dump/init enforced at registration** — defense against per-ops-NULL-deref in datapath.
- **Per-action kind ∧ id unique** — defense against per-alias module load.
- **Per-action_exec REPEAT loop ≤ 32** — defense against per-attacker REPEAT livelock.
- **Per-action_exec JUMP TTL ≤ TCA_ACT_MAX_PRIO** — defense against per-graph-cycle infinite pipeline.
- **Per-GOTO_CHAIN NULL-chain check** — defense against per-rcu-published-but-NULL chain UAF.
- **Per-tcfa_refcnt strict get/put** — defense against per-action UAF between classifier and act API.
- **Per-tcfa_bindcnt strict guard on act-API release** — defense against per-classifier holding action UAF.
- **Per-idr ERR_PTR(-EBUSY) reservation** — defense against per-concurrent idr-slot collision.
- **Per-action user_cookie ≤ TC_COOKIE_MAX_SIZE** — defense against per-cookie memory exhaustion.
- **Per-action standalone-vs-bind skip_hw/skip_sw consistency** — defense against per-offload-flag mismatch.
- **Per-reoffload_cb under net_rwsem read + act_id_mutex + idrinfo.lock** — defense against per-driver-cb-replay racing with action delete.
- **Per-TCA_ACT_TAB ≤ TCA_ACT_MAX_PRIO entries** — defense against per-large-attr DoS.
- **Per-action_add retry ≤ 10 on -EAGAIN** — defense against per-replay livelock during module load.
- **Per-tcf_action_destroy unwind on partial init** — defense against per-init-error leak.
- **Per-flush requires NLM_F_ROOT + kind lookup + module-get** — defense against per-flush-of-wrong-kind.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- net/sched/cls_api.c TC classifiers (covered in `cls-api.md` Tier-3)
- net/sched/sch_api.c qdiscs (covered in `sch-api.md` Tier-3)
- Per-action kinds (act_gact, act_mirred, act_pedit, act_csum, act_nat, act_skbedit, act_vlan, act_tunnel_key, act_bpf, act_police, act_sample, act_ct, act_mpls, act_gate): covered separately in `act-mirred.md`, `act-police.md`, `act-ct.md` Tier-3s
- net/sched/sch_frag.c sch_frag_xmit_hook (referenced only via tcf_frag_xmit_count static-key)
- net/core/flow_offload.c flow_indr_dev_setup_offload infrastructure (covered separately)
- include/net/tc_act/* per-action userspace types
- Implementation code
