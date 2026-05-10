# Tier-3: net/sched/cls_matchall.c — `matchall` TC classifier (unconditional match)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/sched/cls-api.md
upstream-paths:
  - net/sched/cls_matchall.c (~420 lines)
-->

## Summary

`matchall` is the simplest tc classifier: per-skb unconditionally matches, applying its per-filter actions. Per-(prio, handle) classifier instance has a single optional action-list (per cls-api). Common use: per-interface single-policy (mirror-all-egress, rate-limit-all, BPF-prog-all, etc.). Per-userspace `tc filter add dev eth0 ingress matchall action ...`. Per-HW-offload via `cls_matchall_offload` for switchdev / TC-flower-fast-path. Critical for: simple per-iface always-on policy.

This Tier-3 covers `cls_matchall.c` (~420 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cls_mall_head` | per-filter state | `ClsMallHead` |
| `mall_init()` | per-classifier init | `Mall::init` |
| `mall_destroy()` | per-classifier destroy | `Mall::destroy` |
| `mall_classify()` | per-skb classify | `Mall::classify` |
| `mall_change()` | per-filter create/update | `Mall::change` |
| `mall_delete()` | per-filter delete | `Mall::delete` |
| `mall_walk()` | per-classifier walk | `Mall::walk` |
| `mall_dump()` | per-filter dump | `Mall::dump` |
| `mall_replace_hw_filter()` | per-HW-offload swap | `Mall::replace_hw_filter` |
| `mall_destroy_hw_filter()` | per-HW-offload remove | `Mall::destroy_hw_filter` |
| `TCA_MATCHALL_*` | per-attr UAPI | UAPI |

## Compatibility contract

REQ-1: tc UAPI tcf_proto_ops:
- proto.ops = &cls_mall_ops with kind="matchall".

REQ-2: Per-(tcf_proto, prio): at most one matchall filter.
- mall_change checks tp.root != NULL → -EEXIST unless TCA_MATCHALL_FLAGS_KSWAP.

REQ-3: mall_classify(skb, tp, res):
- /* No header inspection */
- head = rcu_dereference_bh(tp.root).
- if !head: return -1.
- res.classid = head.handle.
- res.class = 0.
- tcf_exts_exec(skb, &head.exts, res).  // execute action-list
- return result-action.

REQ-4: Per-action-list (cls-api):
- TCA_MATCHALL_CLASSID: per-classid for child qdisc.
- TCA_MATCHALL_ACT: per-tcf_action-list.
- TCA_MATCHALL_FLAGS: TCA_CLS_FLAGS_*.

REQ-5: Per-HW-offload:
- if flags & TCA_CLS_FLAGS_SKIP_SW: only-HW.
- if flags & TCA_CLS_FLAGS_SKIP_HW: only-SW.
- Else: try HW first, fallback SW.
- TC_SETUP_CLSMATCHALL ndo command to driver.

REQ-6: Per-mall_destroy:
- Cancel any HW-offload via mall_destroy_hw_filter.
- kfree per-filter rcu.

## Acceptance Criteria

- [ ] AC-1: `tc filter add dev eth0 ingress matchall action mirror egress redirect dev eth1`: matchall installed.
- [ ] AC-2: Per-skb ingress: matches; tcf_exts_exec runs mirror action.
- [ ] AC-3: tc filter del: classifier destroyed.
- [ ] AC-4: TCA_CLS_FLAGS_SKIP_SW + supported driver: HW-only offload.
- [ ] AC-5: Per-second matchall: -EEXIST.
- [ ] AC-6: tc filter show: per-filter attributes dumped.
- [ ] AC-7: action BPF (TCA_BPF): per-skb BPF program runs.
- [ ] AC-8: Per-egress matchall: applied to TX.
- [ ] AC-9: Per-HW-offload fall-back: SW path works.

## Architecture

Per-filter:

```
struct ClsMallHead {
  exts: TcfExts,                                 // action list
  handle: u32,
  flags: u32,                                    // TCA_CLS_FLAGS_*
  in_hw_count: u32,                              // # HW-offloads using this
  rcu: RcuHead,
}
```

`Mall::classify(skb, tp, res) -> i32`:
1. head = rcu_dereference_bh(tp.root).
2. if !head: return -1.
3. res.classid = head.handle.
4. res.class = 0.
5. return tcf_exts_exec(skb, &head.exts, res).

`Mall::change(net, tp, base, handle, tca, arg, flags, extack) -> Result<()>`:
1. if tp.root: return -EEXIST.
2. head = kzalloc.
3. err = tcf_exts_init(&head.exts, ...).
4. err = tcf_exts_validate(net, tp, tca, &head.exts, ...).
5. head.handle = handle; head.flags = nla_get_u32(TCA_MATCHALL_FLAGS).
6. /* HW offload */
7. if head.flags & SKIP_SW: must-succeed HW.
8. err = tc_setup_cb_call(...TC_SETUP_CLSMATCHALL, ...).
9. rcu_assign_pointer(tp.root, head).

`Mall::delete(tp, arg, last, ...)`:
1. /* For matchall: only single filter; delete = destroy */
2. Mall::destroy(tp).

`Mall::destroy(tp, force, extack)`:
1. head = rtnl_dereference(tp.root).
2. if head: Mall::destroy_hw_filter(tp, head, ...).
3. tcf_exts_destroy(&head.exts).
4. kfree_rcu(head, rcu).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `single_filter_per_proto` | INVARIANT | per-tp at most one matchall head. |
| `classify_returns_classid` | INVARIANT | per-classify-Some: res.classid == head.handle. |
| `exts_initialized_pre_classify` | INVARIANT | per-classify: head.exts initialized. |
| `hw_in_count_balanced` | INVARIANT | per-replace: in_hw_count balanced. |

### Layer 2: TLA+

`net/sched/cls_matchall.tla`:
- Per-classifier install + per-skb match + per-destroy.
- Properties:
  - `safety_every_skb_matches` — per-skb on tcf_proto with mall: always match.
  - `safety_action_executed_per_match` — per-match: tcf_exts_exec called.
  - `liveness_destroy_eventually_frees` — per-destroy: HW + SW released within RCU.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mall::classify` post: returns tcf_action; res.classid populated | `Mall::classify` |
| `Mall::change` post: tp.root = new head; HW-offload attempted per-flags | `Mall::change` |
| `Mall::destroy` post: HW-offload removed; head freed via RCU | `Mall::destroy` |

### Layer 4: Verus/Creusot functional

`Per-filter matchall: per-skb unconditional match → action-list executed` semantic equivalence: per-Linux TC matchall classifier.

## Hardening

(Inherits row-1 features from `net/sched/cls-api.md` § Hardening.)

Matchall-specific reinforcement:

- **Per-classifier single-filter enforced** — defense against per-filter-explosion.
- **Per-RCU-protected classify** — defense against per-replace UAF.
- **Per-HW-offload paired add/del** — defense against per-stale HW-filter.
- **Per-CAP_NET_ADMIN for change/delete** — defense against unprivileged filter-config.
- **Per-TCA_CLS_FLAGS validated** — defense against per-config invalid combo.
- **Per-tcf_exts validated** — defense against per-malformed action-list.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TC cls-api (covered in `cls-api.md` Tier-3)
- Other classifiers (covered in respective Tier-3)
- Per-action subsystem (covered in `act-api.md` Tier-3)
- Implementation code
