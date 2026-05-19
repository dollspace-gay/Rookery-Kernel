# Tier-3: kernel/bpf/cgroup.c — BPF cgroup attachments (skb / sk / sock_addr / sock_ops / device / sysctl / sockopt / LSM_CGROUP)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/bpf/00-overview.md
upstream-paths:
  - kernel/bpf/cgroup.c (~2757 lines)
  - kernel/bpf/local_storage.c (per-cgroup local storage map type)
  - include/linux/bpf-cgroup.h (struct cgroup_bpf, struct bpf_cgroup_link, struct bpf_prog_list, enum cgroup_bpf_attach_type)
  - include/linux/bpf-cgroup-defs.h (MAX_CGROUP_BPF_ATTACH_TYPE)
  - include/uapi/linux/bpf.h (BPF_CGROUP_INET_INGRESS .. BPF_LSM_CGROUP, BPF_F_ALLOW_OVERRIDE, BPF_F_ALLOW_MULTI, BPF_F_REPLACE, BPF_F_BEFORE, BPF_F_AFTER, BPF_F_PREORDER, BPF_F_ID, BPF_F_LINK, BPF_F_QUERY_EFFECTIVE)
  - kernel/cgroup/cgroup-internal.h (cgroup_lifetime_notifier)
-->

## Summary

`kernel/bpf/cgroup.c` is the **per-cgroup BPF attachment layer**: it implements attach / detach / replace / query of `BPF_PROG_TYPE_CGROUP_*` programs against a `struct cgroup`, computes the effective program chain by walking the cgroup hierarchy, and provides the per-hook fast-path runners that kernel call sites invoke. Per-cgroup state lives in `struct cgroup_bpf` (embedded in `struct cgroup`), comprising: `progs[atype]` (a `hlist_head` of `bpf_prog_list` entries, one per cgroup-local attachment), `effective[atype]` (an RCU-pointer to a `bpf_prog_array` of the actual hierarchical chain), `flags[atype]` (the active `BPF_F_ALLOW_OVERRIDE` / `BPF_F_ALLOW_MULTI` mode for this cgroup-level), `revisions[atype]` (monotonic counter used for `BPF_F_REPLACE` / expected_revision matching), `storages` (list of per-cgroup local storage maps owned by attached programs), `refcnt` (percpu_ref enabling deferred cleanup), and `release_work` (workqueue entry running on `cgroup_bpf_destroy_wq`). Per-program registration: `__cgroup_bpf_attach` allocates a `bpf_prog_list pl`, inserts under `cgrp->bpf.progs[atype]` according to BPF_F_BEFORE / BPF_F_AFTER / BPF_F_PREORDER + anchor (BPF_F_ID / fd / link target), then calls `update_effective_progs` which walks every descendant cgroup, recomputes `compute_effective_progs` (walks up from descendant collecting parent progs honoring BPF_F_ALLOW_OVERRIDE / BPF_F_ALLOW_MULTI and reverse-ordering pre-order pokes), and finally atomically swaps the RCU-protected `effective` arrays via `activate_effective_progs`. Per-hook execution is uniformly via `bpf_prog_run_array_cg(&cgrp->bpf, atype, ctx, run_prog, retval, *ret_flags)` running each `prog` from `cgrp->bpf.effective[atype]->items[]` under RCU + migrate-disable with a `bpf_cg_run_ctx` (`retval` carries through the chain; `func_ret == 0` from any prog sets `retval = -EPERM`; helpers `bpf_get_retval` / `bpf_set_retval` access `current->bpf_ctx`). Per-hook flavors: skb (`__cgroup_bpf_run_filter_skb` for INGRESS/EGRESS — manages `__skb_push`/`__skb_pull` around `bpf_compute_and_save_data_end`, converts BPF return into NET_XMIT_SUCCESS / NET_XMIT_DROP / NET_XMIT_CN); sk (`__cgroup_bpf_run_filter_sk` for INET_SOCK_CREATE / INET_SOCK_RELEASE / INET4_POST_BIND / INET6_POST_BIND); sock_addr (`__cgroup_bpf_run_filter_sock_addr` for CONNECT / SENDMSG / RECVMSG / GETPEERNAME / GETSOCKNAME across INET4/INET6/UNIX — supports modifying sockaddr length for AF_UNIX); sock_ops (`__cgroup_bpf_run_filter_sock_ops` for CGROUP_SOCK_OPS); device (`__cgroup_bpf_check_dev_permission` for CGROUP_DEVICE — passes `bpf_cgroup_dev_ctx` with major/minor/access); sysctl (`__cgroup_bpf_run_filter_sysctl` — copies cur_val + new_val into BPF context, allows program to override new value via `bpf_sysctl_set_new_value`); sockopt (`__cgroup_bpf_run_filter_setsockopt` / `__cgroup_bpf_run_filter_getsockopt` / `__cgroup_bpf_run_filter_getsockopt_kern` — copies optval into `bpf_sockopt_kern`, supports `ctx.optlen == -1` to bypass kernel handler); LSM_CGROUP (`__cgroup_bpf_run_lsm_sock` / `__cgroup_bpf_run_lsm_socket` / `__cgroup_bpf_run_lsm_current` — trampoline shims dispatch through `shim_prog->aux->cgroup_atype` into `bpf_prog_run_array_cg`). Per-multi-prog policy: `BPF_F_ALLOW_OVERRIDE` (single prog overridable by descendant); `BPF_F_ALLOW_MULTI` (multiple progs at this cgroup, descendants can stack via MULTI too); neither (single prog non-overridable). `hierarchy_allows_attach` walks parents to reject incompatible attach modes. Per-link: `bpf_cgroup_link` (refcounted `bpf_link` wrapping a cgroup + attach_type) supports `link_update` (`__cgroup_bpf_replace` swaps `link->prog` via xchg and rewrites every descendant's effective array via `replace_effective_prog`), `link_release` (calls `__cgroup_bpf_detach` with the link as detach key), and auto-detach via cgroup lifetime notifier when the cgroup dies. Per-storage: `bpf_cgroup_storages_alloc` / `_link` manages `bpf_cgroup_storage` map entries keyed by `(cgroup_inode_id, attach_type)` so that BPF programs can keep persistent state per (cgroup, hook). Per-recursion: each `bpf_prog_run_array_cg` uses `rcu_read_lock_dont_migrate` + `bpf_set_run_ctx` to nest a `bpf_cg_run_ctx` over `current->bpf_ctx`; this also gates `cgroup_bpf_enabled_key[atype]` (per-attach-type static-branch) so call sites pay zero cost when no program is attached. Critical for: Cilium / Calico kube-policy, systemd-resolved / NetworkManager sock-policy, container runtime device cgroup v2 enforcement, systemd-sysctl filtering, runc / podman cgroup-isolated networking, kube-apiserver SO_REUSEPORT routing.

This Tier-3 covers `kernel/bpf/cgroup.c` (~2757 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct cgroup_bpf` | per-cgroup state (progs, effective, flags, revisions, storages, refcnt) | `CgroupBpf` |
| `struct bpf_prog_list` | per-attachment entry (prog, link, flags, storage[], hlist node) | `BpfProgList` |
| `struct bpf_cgroup_link` | per-link state (link, cgroup) | `BpfCgroupLink` |
| `struct prog_poke_elem` | per-tracked-prog JIT (used elsewhere; cgroup paths share `bpf_prog_array_item`) | shared |
| `enum cgroup_bpf_attach_type` | per-attach-type index (CGROUP_INET_INGRESS .. CGROUP_LSM_END) | `CgroupBpfAttachType` |
| `cgroup_bpf_enabled_key` | per-attach-type static-branch (DEFINE_STATIC_KEY_ARRAY_FALSE) | `cgroup_bpf_enabled_key` |
| `cgroup_bpf_destroy_wq` | dedicated workqueue for `cgroup_bpf_release` | `cgroup_bpf_destroy_wq` |
| `cgroup_bpf_lifetime_notify()` / `cgroup_bpf_lifetime_notifier_init()` | per-cgroup-lifetime ONLINE / OFFLINE hook | `Cgroup::bpf_lifetime_notify` |
| `cgroup_bpf_inherit()` | per-online inherit (percpu_ref_init + percpu_ref_get parents + compute_effective_progs for each atype) | `Cgroup::bpf_inherit` |
| `cgroup_bpf_offline()` | per-offline (percpu_ref_kill triggers release) | `Cgroup::bpf_offline` |
| `cgroup_bpf_release()` | per-release worker (drains progs[]; auto-detaches links; frees effective[]; releases storages; cgroup_bpf_put parents) | `Cgroup::bpf_release` |
| `cgroup_bpf_release_fn()` | per-percpu_ref-zero callback (queues release_work) | `Cgroup::bpf_release_fn` |
| `bpf_prog_run_array_cg()` | per-hook chain runner | `Cgroup::run_array_cg` |
| `__cgroup_bpf_run_lsm_sock()` / `__cgroup_bpf_run_lsm_socket()` / `__cgroup_bpf_run_lsm_current()` | per-LSM trampoline shim | `Cgroup::run_lsm_sock` / `run_lsm_socket` / `run_lsm_current` |
| `bpf_cgroup_atype_find()` / `bpf_cgroup_atype_get()` / `bpf_cgroup_atype_put()` | per-LSM-cgroup btf_id → atype mapping (CGROUP_LSM_START..CGROUP_LSM_END) | `Cgroup::atype_find` / `atype_get` / `atype_put` |
| `bpf_cgroup_storages_alloc()` / `_free()` / `_assign()` / `_link()` | per-attach cgroup-storage lifecycle | `Cgroup::storages_alloc` / `free` / `assign` / `link` |
| `hierarchy_allows_attach()` | per-attach mode compatibility check (walks parents) | `Cgroup::hierarchy_allows_attach` |
| `compute_effective_progs()` | per-recompute hierarchical chain (front + reversed pre-order) | `Cgroup::compute_effective_progs` |
| `activate_effective_progs()` | per-RCU-swap effective array | `Cgroup::activate_effective_progs` |
| `update_effective_progs()` | per-descendant recompute + activate (with cleanup on OOM) | `Cgroup::update_effective_progs` |
| `purge_effective_progs()` | per-recover in-place if update_effective_progs OOMs on detach | `Cgroup::purge_effective_progs` |
| `find_attach_entry()` / `find_detach_entry()` | per-attach / detach lookup | `Cgroup::find_attach_entry` / `find_detach_entry` |
| `bpf_get_anchor_prog()` / `bpf_get_anchor_link()` | per-BPF_F_BEFORE/AFTER anchor resolve | `Cgroup::get_anchor_prog` / `get_anchor_link` |
| `get_prog_list()` / `insert_pl_to_hlist()` | per-insertion-position selection | `Cgroup::get_prog_list` / `insert_pl_to_hlist` |
| `prog_list_length()` / `prog_list_prog()` | per-list helpers | `Cgroup::prog_list_length` / `prog_list_prog` |
| `__cgroup_bpf_attach()` / `cgroup_bpf_attach()` | per-attach (under cgroup_mutex) | `Cgroup::bpf_attach_inner` / `bpf_attach` |
| `__cgroup_bpf_detach()` / `cgroup_bpf_detach()` | per-detach (under cgroup_mutex) | `Cgroup::bpf_detach_inner` / `bpf_detach` |
| `__cgroup_bpf_replace()` / `cgroup_bpf_replace()` / `replace_effective_prog()` | per-link-replace (xchg + per-descendant patch) | `Cgroup::bpf_replace_inner` / `bpf_replace` / `replace_effective_prog` |
| `__cgroup_bpf_query()` / `cgroup_bpf_query()` | per-BPF_PROG_QUERY copy_to_user (effective or cgroup-local; per-LSM range) | `Cgroup::bpf_query_inner` / `bpf_query` |
| `cgroup_bpf_prog_attach()` / `cgroup_bpf_prog_detach()` / `cgroup_bpf_prog_query()` / `cgroup_bpf_link_attach()` | syscall entries | `Cgroup::bpf_prog_attach` / `prog_detach` / `prog_query` / `link_attach` |
| `bpf_cgroup_link_release()` / `_dealloc()` / `_detach()` / `_show_fdinfo()` / `_fill_link_info()` / `bpf_cgroup_link_lops` | per-link bpf_link_ops | `Cgroup::link_release` / `dealloc` / `detach` / `show_fdinfo` / `fill_link_info` / `link_lops` |
| `bpf_cgroup_link_auto_detach()` | per-cgroup-death (auto-detaches link without freeing memory; release later via fd close) | `Cgroup::link_auto_detach` |
| `__cgroup_bpf_run_filter_skb()` | per-CGROUP_INET_INGRESS / _EGRESS runner | `Cgroup::run_filter_skb` |
| `__cgroup_bpf_run_filter_sk()` | per-CGROUP_INET_SOCK_CREATE / _RELEASE / _POST_BIND runner | `Cgroup::run_filter_sk` |
| `__cgroup_bpf_run_filter_sock_addr()` | per-CGROUP_INET_BIND / CONNECT / SENDMSG / RECVMSG / GETPEER/SOCKNAME / UNIX_* runner | `Cgroup::run_filter_sock_addr` |
| `__cgroup_bpf_run_filter_sock_ops()` | per-CGROUP_SOCK_OPS runner | `Cgroup::run_filter_sock_ops` |
| `__cgroup_bpf_check_dev_permission()` | per-CGROUP_DEVICE permission check | `Cgroup::check_dev_permission` |
| `__cgroup_bpf_run_filter_sysctl()` | per-CGROUP_SYSCTL runner (cur_val + new_val buffers) | `Cgroup::run_filter_sysctl` |
| `__cgroup_bpf_run_filter_setsockopt()` / `__cgroup_bpf_run_filter_getsockopt()` / `__cgroup_bpf_run_filter_getsockopt_kern()` | per-CGROUP_SETSOCKOPT / _GETSOCKOPT runners | `Cgroup::run_filter_setsockopt` / `getsockopt` / `getsockopt_kern` |
| `sockopt_alloc_buf()` / `sockopt_free_buf()` / `sockopt_buf_allocated()` | per-sockopt scratch buffer (inline `bpf_sockopt_buf::data` or kzalloc) | `Cgroup::sockopt_alloc_buf` / `free_buf` |
| `bpf_get_local_storage` / `_proto` | per-program access to per-cgroup storage | `bpf_get_local_storage` |
| `bpf_get_retval` / `bpf_set_retval` / `_proto` | per-program retval get/set (via bpf_cg_run_ctx) | `bpf_get_retval` / `bpf_set_retval` |
| `cgroup_common_func_proto()` | per-attach-type filter for retval helpers + storage helper | `Cgroup::common_func_proto` |
| `cg_dev_prog_ops` / `cg_dev_verifier_ops` / `cgroup_dev_func_proto()` / `cgroup_dev_is_valid_access()` | per-CGROUP_DEVICE verifier | `Cgroup::dev_*` |
| `cg_sysctl_prog_ops` / `cg_sysctl_verifier_ops` / `sysctl_func_proto()` / `sysctl_is_valid_access()` / `sysctl_convert_ctx_access()` | per-CGROUP_SYSCTL verifier | `Cgroup::sysctl_*` |
| `cg_sockopt_prog_ops` / `cg_sockopt_verifier_ops` / `cg_sockopt_func_proto()` / `cg_sockopt_is_valid_access()` / `cg_sockopt_convert_ctx_access()` / `cg_sockopt_get_prologue()` | per-CGROUP_SETSOCKOPT / _GETSOCKOPT verifier | `Cgroup::sockopt_*` |
| `bpf_sysctl_get_name()` / `_get_current_value()` / `_get_new_value()` / `_set_new_value()` / their `_proto`s | per-sysctl BPF helpers | `Cgroup::sysctl_get_name` etc. |
| `bpf_get_netns_cookie_sockopt()` / `_proto` | per-sockopt netns-cookie helper | `Cgroup::sockopt_get_netns_cookie` |

## Compatibility contract

REQ-1: struct cgroup_bpf (per-cgroup, embedded in `struct cgroup`):
- progs: `struct hlist_head[MAX_CGROUP_BPF_ATTACH_TYPE]` — per-atype attachment list at this cgroup level.
- effective: `struct bpf_prog_array __rcu *[MAX_CGROUP_BPF_ATTACH_TYPE]` — per-atype runtime chain (this cgroup's programs + hierarchical-allowed ancestors).
- inactive: `struct bpf_prog_array *[MAX_CGROUP_BPF_ATTACH_TYPE]` — scratch slot used during update_effective_progs allocate-then-activate.
- flags: `u32[MAX_CGROUP_BPF_ATTACH_TYPE]` — per-atype mode (BPF_F_ALLOW_OVERRIDE | BPF_F_ALLOW_MULTI; 0 = single non-overridable).
- revisions: `u64[MAX_CGROUP_BPF_ATTACH_TYPE]` — monotonic counter for expected_revision matching.
- storages: `struct list_head` — owned per-cgroup local storage entries.
- refcnt: `struct percpu_ref` — released by `cgroup_bpf_offline` (`percpu_ref_kill`) → `cgroup_bpf_release_fn` queues `release_work`.
- release_work: `struct work_struct` — runs `cgroup_bpf_release` on `cgroup_bpf_destroy_wq`.

REQ-2: struct bpf_prog_list (per-attachment):
- prog: `struct bpf_prog *` — direct attach (NULL if link).
- link: `struct bpf_cgroup_link *` — link attach (NULL if direct).
- flags: `u32` — original attach flags (carries BPF_F_PREORDER, BPF_F_ALLOW_*).
- storage: `struct bpf_cgroup_storage *[MAX_BPF_CGROUP_STORAGE_TYPE]` — per-attach storage handle (SHARED + PERCPU types).
- node: `struct hlist_node` — link in `cgrp->bpf.progs[atype]`.

REQ-3: struct bpf_cgroup_link (per-link):
- link: `struct bpf_link` (embeds prog, lops, attach_type, refcnt).
- cgroup: `struct cgroup *` — pinned cgroup; cleared to NULL on auto-detach.

REQ-4: cgroup_bpf_lifetime_notify(nb, action, data):
- if cgrp.root != &cgrp_dfl_root: return NOTIFY_OK /* only default cgroup root supports BPF */.
- switch action:
  - CGROUP_LIFETIME_ONLINE: ret = cgroup_bpf_inherit(cgrp).
  - CGROUP_LIFETIME_OFFLINE: cgroup_bpf_offline(cgrp).
- return notifier_from_errno(ret).

REQ-5: cgroup_bpf_inherit(cgrp):
- percpu_ref_init(&cgrp.bpf.refcnt, cgroup_bpf_release_fn, 0, GFP_KERNEL).
- for p = cgrp.parent..root: cgroup_bpf_get(p) /* take ref so descendants outlive their ancestors' progs */.
- for i in 0..NR: INIT_HLIST_HEAD(&cgrp.bpf.progs[i]).
- INIT_LIST_HEAD(&cgrp.bpf.storages).
- for i in 0..NR: compute_effective_progs(cgrp, i, &arrays[i]).
- for i in 0..NR: activate_effective_progs(cgrp, i, arrays[i]).
- on err: bpf_prog_array_free(arrays[i]); cgroup_bpf_put(p) for each; percpu_ref_exit; return -ENOMEM.

REQ-6: cgroup_bpf_offline(cgrp):
- cgroup_get(cgrp) /* extend lifetime past percpu_ref_kill */.
- percpu_ref_kill(&cgrp.bpf.refcnt) /* triggers cgroup_bpf_release_fn → schedule release_work */.

REQ-7: cgroup_bpf_release(work):
- cgroup_lock.
- for atype in 0..NR:
  - hlist_for_each_entry_safe(pl, pltmp, &cgrp.bpf.progs[atype], node):
    - hlist_del(&pl.node).
    - if pl.prog:
      - if pl.prog.expected_attach_type == BPF_LSM_CGROUP: bpf_trampoline_unlink_cgroup_shim(pl.prog).
      - bpf_prog_put(pl.prog).
    - if pl.link:
      - if pl.link.link.prog.expected_attach_type == BPF_LSM_CGROUP: bpf_trampoline_unlink_cgroup_shim(pl.link.link.prog).
      - bpf_cgroup_link_auto_detach(pl.link).
    - kfree(pl).
    - static_branch_dec(&cgroup_bpf_enabled_key[atype]).
  - old_array = rcu_dereference_protected(cgrp.bpf.effective[atype], lockdep_is_held(&cgroup_mutex)).
  - bpf_prog_array_free(old_array).
- /* Drop owned storages */
- list_for_each_entry_safe(storage, stmp, &cgrp.bpf.storages, list_cg): bpf_cgroup_storage_unlink; bpf_cgroup_storage_free.
- cgroup_unlock.
- /* Release parents' bpf-refs */
- for p = cgrp.parent..root: cgroup_bpf_put(p).
- percpu_ref_exit(&cgrp.bpf.refcnt); cgroup_put(cgrp).

REQ-8: hierarchy_allows_attach(cgrp, atype):
- /* Walk parents */
- p = cgroup_parent(cgrp); if !p: return true.
- do:
  - flags = p.bpf.flags[atype].
  - if (flags & BPF_F_ALLOW_MULTI): return true /* multi mode allows descendants */.
  - cnt = prog_list_length(&p.bpf.progs[atype], NULL).
  - WARN_ON_ONCE(cnt > 1) /* non-multi → at most 1 prog */.
  - if cnt == 1: return !!(flags & BPF_F_ALLOW_OVERRIDE) /* parent's single prog must be overridable */.
  - p = cgroup_parent(p).
- return true.

REQ-9: compute_effective_progs(cgrp, atype, *array):
- /* Pass 1: count + count preorder */
- cnt = 0; preorder_cnt = 0; p = cgrp.
- do:
  - if cnt == 0 ∨ (p.bpf.flags[atype] & BPF_F_ALLOW_MULTI): cnt += prog_list_length(&p.bpf.progs[atype], &preorder_cnt).
  - p = cgroup_parent(p).
- while p.
- progs = bpf_prog_array_alloc(cnt, GFP_KERNEL).
- /* Pass 2: populate */
- fstart = preorder_cnt /* normal entries appended after preorder block */.
- bstart = preorder_cnt - 1 /* preorder entries inserted in reverse */.
- p = cgrp.
- do:
  - if cnt > 0 ∧ !(p.bpf.flags[atype] & BPF_F_ALLOW_MULTI): continue /* non-multi stops chain */.
  - init_bstart = bstart.
  - hlist_for_each_entry(pl, &p.bpf.progs[atype], node):
    - if !prog_list_prog(pl): continue.
    - if (pl.flags & BPF_F_PREORDER): item = &progs.items[bstart--].
    - else: item = &progs.items[fstart++].
    - item.prog = prog_list_prog(pl); bpf_cgroup_storages_assign(item.cgroup_storage, pl.storage).
    - cnt++.
  - /* reverse the preorder run from this cgroup level so children-pre-order runs in declaration order */
  - for i = bstart + 1, j = init_bstart; i < j; i++, j--: swap(progs.items[i], progs.items[j]).
- while p = cgroup_parent(p).
- *array = progs.

REQ-10: activate_effective_progs(cgrp, atype, old_array):
- old_array = rcu_replace_pointer(cgrp.bpf.effective[atype], old_array, lockdep_is_held(&cgroup_mutex)).
- bpf_prog_array_free(old_array) /* free *previous* effective after grace period — readers may still walk it */.

REQ-11: update_effective_progs(cgrp, atype):
- /* Two-phase: allocate-all then activate-all so partial failure leaves no descendant with stale effective */.
- css_for_each_descendant_pre(css, &cgrp.self):
  - desc = container_of(css, cgroup, self).
  - if percpu_ref_is_zero(&desc.bpf.refcnt): continue.
  - err = compute_effective_progs(desc, atype, &desc.bpf.inactive); on err: goto cleanup.
- css_for_each_descendant_pre(css, &cgrp.self):
  - if percpu_ref_is_zero: drop inactive if any; continue.
  - activate_effective_progs(desc, atype, desc.bpf.inactive); desc.bpf.inactive = NULL.
- cleanup: free each desc.bpf.inactive on OOM.

REQ-12: __cgroup_bpf_attach(cgrp, prog, replace_prog, link, type, flags, id_or_fd, revision) — under cgroup_mutex:
- saved_flags = flags & (BPF_F_ALLOW_OVERRIDE | BPF_F_ALLOW_MULTI).
- new_prog = prog ? : link.link.prog.
- /* Flag combinator validation */
- if (BPF_F_ALLOW_OVERRIDE ∧ BPF_F_ALLOW_MULTI) ∨ (BPF_F_REPLACE ∧ !BPF_F_ALLOW_MULTI): return -EINVAL.
- if (BPF_F_REPLACE) ∧ (flags & (BPF_F_BEFORE | BPF_F_AFTER)): return -EINVAL.
- if link ∧ (prog ∨ replace_prog): return -EINVAL.
- if !!replace_prog != !!(flags & BPF_F_REPLACE): return -EINVAL.
- /* atype resolution (handles LSM_CGROUP btf_id slot mapping) */
- atype = bpf_cgroup_atype_find(type, new_prog.aux.attach_btf_id). if atype < 0: return -EINVAL.
- if revision ∧ revision != cgrp.bpf.revisions[atype]: return -ESTALE.
- progs = &cgrp.bpf.progs[atype].
- if !hierarchy_allows_attach(cgrp, atype): return -EPERM.
- if !hlist_empty(progs) ∧ cgrp.bpf.flags[atype] != saved_flags: return -EPERM /* mode mismatch */.
- if prog_list_length(progs, NULL) >= BPF_CGROUP_MAX_PROGS (64): return -E2BIG.
- pl = find_attach_entry(progs, prog, link, replace_prog, BPF_F_ALLOW_MULTI?); on err propagate.
- bpf_cgroup_storages_alloc(storage, new_storage, type, prog ? : link.link.prog, cgrp).
- if pl: old_prog = pl.prog /* REPLACE path */.
- else: pl = kzalloc; insert_pl_to_hlist(pl, progs, prog, link, flags, id_or_fd).
- pl.prog = prog; pl.link = link; pl.flags = flags; bpf_cgroup_storages_assign(pl.storage, storage); cgrp.bpf.flags[atype] = saved_flags.
- if type == BPF_LSM_CGROUP: bpf_trampoline_link_cgroup_shim(new_prog, atype, type).
- update_effective_progs(cgrp, atype) /* propagates to all descendants */.
- cgrp.bpf.revisions[atype]++.
- if old_prog: if LSM_CGROUP: bpf_trampoline_unlink_cgroup_shim(old_prog); bpf_prog_put(old_prog).
- else: static_branch_inc(&cgroup_bpf_enabled_key[atype]).
- bpf_cgroup_storages_link(new_storage, cgrp, type).
- on cleanup: revert pl, free new_storage, unlink trampoline shim.

REQ-13: insert_pl_to_hlist(pl, progs, prog, link, flags, id_or_fd):
- pltmp = get_prog_list(progs, prog, link, flags, id_or_fd) /* resolves anchor (BEFORE/AFTER) */.
- if !pltmp: hlist_add_head(&pl.node, progs).
- else if (flags & BPF_F_BEFORE): hlist_add_before(&pl.node, &pltmp.node).
- else: hlist_add_behind(&pl.node, &pltmp.node).

REQ-14: get_prog_list(progs, prog, link, flags, id_or_fd):
- is_link = flags & BPF_F_LINK; is_id = flags & BPF_F_ID; is_before = flags & BPF_F_BEFORE; is_after = flags & BPF_F_AFTER; preorder = flags & BPF_F_PREORDER.
- if (is_link ∨ is_id ∨ id_or_fd):
  - if is_before == is_after: return -EINVAL /* exactly one direction required */.
  - if (is_link ∧ !link) ∨ (!is_link ∧ !prog): return -EINVAL.
- else if !hlist_empty(progs) ∧ is_before ∧ is_after: return -EINVAL.
- /* Resolve anchor prog or link */
- if is_link: anchor_link = bpf_get_anchor_link(flags, id_or_fd).
- else if is_id ∨ id_or_fd: anchor_prog = bpf_get_anchor_prog(flags, id_or_fd).
- if no anchor: walk list — is_before → first; else → last; return.
- /* Find anchor in list; preorder mismatch → -EINVAL */
- hlist_for_each_entry(pltmp, progs):
  - if (anchor_prog == pltmp.prog) ∨ (anchor_link == &pltmp.link.link):
    - if !!(pltmp.flags & BPF_F_PREORDER) != preorder: return -EINVAL.
    - return pltmp.
- return -ENOENT.

REQ-15: find_attach_entry(progs, prog, link, replace_prog, allow_multi):
- if !allow_multi:
  - if hlist_empty: return NULL /* fresh attach */.
  - return first entry /* single-mode: replace existing */.
- hlist_for_each_entry(pl, progs):
  - if prog ∧ pl.prog == prog ∧ prog != replace_prog: return -EINVAL /* duplicate */.
  - if link ∧ pl.link == link: return -EINVAL.
- /* REPLACE: find replace_prog */
- if replace_prog:
  - find entry with pl.prog == replace_prog else return -ENOENT.
- return NULL /* new entry to add */.

REQ-16: __cgroup_bpf_detach(cgrp, prog, link, type, revision) — under cgroup_mutex:
- atype = bpf_cgroup_atype_find(type, attach_btf_id of prog/link).
- if revision ∧ revision != cgrp.bpf.revisions[atype]: return -ESTALE.
- if prog ∧ link: return -EINVAL.
- pl = find_detach_entry(progs, prog, link, flags & BPF_F_ALLOW_MULTI). if err: return.
- /* Tombstone for recompute */
- old_prog = pl.prog; pl.prog = NULL; pl.link = NULL.
- if update_effective_progs(cgrp, atype) fails: /* OOM */
  - pl.prog = old_prog; pl.link = link.
  - purge_effective_progs(cgrp, old_prog, link, atype) /* recover in-place: delete-safe-at index */.
- hlist_del(&pl.node); cgrp.bpf.revisions[atype]++; kfree(pl).
- if hlist_empty(progs): cgrp.bpf.flags[atype] = 0 /* reset mode */.
- if old_prog: if LSM_CGROUP: bpf_trampoline_unlink_cgroup_shim; bpf_prog_put(old_prog).
- static_branch_dec(&cgroup_bpf_enabled_key[atype]).

REQ-17: purge_effective_progs(cgrp, prog, link, atype) — recovery on OOM during detach:
- css_for_each_descendant_pre:
  - find position of (prog, link) in desc's effective array via hierarchical walk.
  - bpf_prog_array_delete_safe_at(progs, pos) — in-place delete + WARN_ONCE on failure.

REQ-18: __cgroup_bpf_replace(cgrp, link, new_prog) — under cgroup_mutex:
- atype = bpf_cgroup_atype_find(link.link.attach_type, new_prog.aux.attach_btf_id).
- if link.link.prog.type != new_prog.type: return -EINVAL.
- locate pl with pl.link == link in cgrp.bpf.progs[atype]; if not found: return -ENOENT.
- cgrp.bpf.revisions[atype]++.
- old_prog = xchg(&link.link.prog, new_prog).
- replace_effective_prog(cgrp, atype, link) /* descend + WRITE_ONCE item.prog */.
- bpf_prog_put(old_prog).

REQ-19: replace_effective_prog(cgrp, atype, link):
- css_for_each_descendant_pre:
  - find position of link in desc's effective by walking from desc up to a multi-allowing ancestor.
  - progs = rcu_dereference_protected(desc.bpf.effective[atype]).
  - item = &progs.items[pos].
  - WRITE_ONCE(item.prog, link.link.prog) /* atomic update; readers see old-or-new */.

REQ-20: __cgroup_bpf_query(cgrp, attr, uattr) — under cgroup_mutex:
- /* Two modes: effective (rcu_dereference effective array) vs cgroup-local (walk progs[]) */
- effective_query = attr.query.query_flags & BPF_F_QUERY_EFFECTIVE.
- if effective_query ∧ prog_attach_flags: return -EINVAL.
- /* LSM_CGROUP iterates CGROUP_LSM_START..CGROUP_LSM_END */
- if type == BPF_LSM_CGROUP: from_atype = CGROUP_LSM_START; to_atype = CGROUP_LSM_END.
- else: from_atype = to_atype = to_cgroup_bpf_attach_type(type); flags = cgrp.bpf.flags[atype].
- /* Pass 1: count */
- for atype: total_cnt += (effective_query ? bpf_prog_array_length(effective) : prog_list_length(progs)).
- copy_to_user(attach_flags, prog_cnt, revision).
- if prog_cnt == 0 ∨ !prog_ids ∨ !total_cnt: return 0.
- if attr.query.prog_cnt < total_cnt: total_cnt = attr.query.prog_cnt; ret = -ENOSPC.
- /* Pass 2: copy IDs (and per-prog flags if requested) */
- for atype:
  - if effective_query: bpf_prog_array_copy_to_user.
  - else: iterate progs[] writing prog.aux.id + per-prog flags.

REQ-21: cgroup_bpf_prog_attach(attr, ptype, prog) — syscall entry:
- cgrp = cgroup_get_from_fd(attr.target_fd).
- if (attach_flags & BPF_F_ALLOW_MULTI) ∧ (attach_flags & BPF_F_REPLACE): replace_prog = bpf_prog_get_type(attr.replace_bpf_fd, ptype).
- cgroup_bpf_attach(cgrp, prog, replace_prog, NULL, attr.attach_type, attr.attach_flags, attr.relative_fd, attr.expected_revision).
- cleanup refs.

REQ-22: cgroup_bpf_link_attach(attr, prog):
- if attr.link_create.flags & ~BPF_F_LINK_ATTACH_MASK: return -EINVAL.
  - BPF_F_LINK_ATTACH_MASK = BPF_F_ID | BPF_F_BEFORE | BPF_F_AFTER | BPF_F_PREORDER | BPF_F_LINK.
- cgrp = cgroup_get_from_fd(attr.link_create.target_fd).
- link = kzalloc(bpf_cgroup_link). bpf_link_init(&link.link, BPF_LINK_TYPE_CGROUP, &bpf_cgroup_link_lops, prog, attr.link_create.attach_type). link.cgroup = cgrp.
- bpf_link_prime(&link.link, &link_primer).
- cgroup_bpf_attach(cgrp, NULL, NULL, link, link.link.attach_type, BPF_F_ALLOW_MULTI | attr.link_create.flags, relative_fd, expected_revision).
- return bpf_link_settle(&link_primer).

REQ-23: bpf_cgroup_link_release(link):
- /* link may have been auto-detached by dying cgroup */
- if !cg_link.cgroup: return.
- cgroup_lock; re-check; if still attached: WARN_ON(__cgroup_bpf_detach(cg_link.cgroup, NULL, cg_link, link.attach_type, 0)).
- if LSM_CGROUP: bpf_trampoline_unlink_cgroup_shim.
- cg_link.cgroup = NULL; cgroup_unlock; cgroup_put(cg).

REQ-24: bpf_cgroup_link_auto_detach(link):
- cgroup_put(link.cgroup); link.cgroup = NULL /* link memory freed later by bpf_link release fop on fd close */.

REQ-25: bpf_prog_run_array_cg(cgrp_bpf, atype, ctx, run_prog, retval, *ret_flags):
- run_ctx.retval = retval.
- rcu_read_lock_dont_migrate().
- array = rcu_dereference(cgrp_bpf.effective[atype]).
- item = &array.items[0].
- old_run_ctx = bpf_set_run_ctx(&run_ctx.run_ctx) /* sets current.bpf_ctx */.
- while prog = READ_ONCE(item.prog):
  - run_ctx.prog_item = item.
  - func_ret = run_prog(prog, ctx).
  - if ret_flags: *ret_flags |= (func_ret >> 1); func_ret &= 1.
  - if !func_ret ∧ !IS_ERR_VALUE((long)run_ctx.retval): run_ctx.retval = -EPERM /* default deny on first 0-return */.
  - item++.
- bpf_reset_run_ctx(old_run_ctx).
- rcu_read_unlock_migrate().
- return run_ctx.retval.

REQ-26: __cgroup_bpf_run_filter_skb(sk, skb, atype) (BPF_CGROUP_INET_INGRESS / _EGRESS):
- if sk.sk_family != AF_INET ∧ AF_INET6: return 0.
- cgrp = sock_cgroup_ptr(&sk.sk_cgrp_data).
- save_sk = skb.sk; skb.sk = sk; __skb_push(skb, -skb_network_offset(skb)).
- bpf_compute_and_save_data_end(skb, &saved_data_end).
- if atype == CGROUP_INET_EGRESS:
  - flags=0; ret = bpf_prog_run_array_cg(&cgrp.bpf, atype, skb, __bpf_prog_run_save_cb, 0, &flags).
  - cn = (flags & BPF_RET_SET_CN).
  - /* return-mapping: 0=drop, 1=keep, 2=drop+cn, 3=keep+cn → NET_XMIT_* */
  - if ret ∧ !IS_ERR_VALUE: ret = -EFAULT.
  - if !ret: ret = cn ? NET_XMIT_CN : NET_XMIT_SUCCESS.
  - else: ret = cn ? NET_XMIT_DROP : ret.
- else: ret = bpf_prog_run_array_cg(&cgrp.bpf, atype, skb, __bpf_prog_run_save_cb, 0, NULL).
- bpf_restore_data_end(skb, saved_data_end); __skb_pull(skb, offset); skb.sk = save_sk.

REQ-27: __cgroup_bpf_run_filter_sk(sk, atype) (BPF_CGROUP_INET_SOCK_CREATE / _RELEASE / _POST_BIND):
- cgrp = sock_cgroup_ptr(&sk.sk_cgrp_data).
- return bpf_prog_run_array_cg(&cgrp.bpf, atype, sk, bpf_prog_run, 0, NULL).

REQ-28: __cgroup_bpf_run_filter_sock_addr(sk, uaddr, *uaddrlen, atype, t_ctx, *flags) (CONNECT / SENDMSG / RECVMSG / GETPEER/SOCKNAME for INET4 / INET6 / UNIX):
- if !sk_is_inet(sk) ∧ !sk_is_unix(sk): return 0.
- ctx = { sk, uaddr, t_ctx }; if !ctx.uaddr: ctx.uaddr = &storage; ctx.uaddrlen = 0.
- cgrp = sock_cgroup_ptr(&sk.sk_cgrp_data).
- ret = bpf_prog_run_array_cg(&cgrp.bpf, atype, &ctx, bpf_prog_run, 0, flags).
- if !ret ∧ uaddr: *uaddrlen = ctx.uaddrlen /* AF_UNIX: program may shorten addr */.

REQ-29: __cgroup_bpf_run_filter_sock_ops(sk, sock_ops, atype) (BPF_CGROUP_SOCK_OPS):
- cgrp = sock_cgroup_ptr(&sk.sk_cgrp_data).
- bpf_prog_run_array_cg(&cgrp.bpf, atype, sock_ops, bpf_prog_run, 0, NULL).

REQ-30: __cgroup_bpf_check_dev_permission(dev_type, major, minor, access, atype) (BPF_CGROUP_DEVICE):
- ctx = { access_type = (access<<16) | dev_type, major, minor }.
- rcu_read_lock; cgrp = task_dfl_cgroup(current); ret = bpf_prog_run_array_cg(&cgrp.bpf, atype, &ctx, bpf_prog_run, 0, NULL); rcu_read_unlock.

REQ-31: __cgroup_bpf_run_filter_sysctl(head, table, write, **buf, *pcount, *ppos, atype) (BPF_CGROUP_SYSCTL):
- ctx = { head, table, write, ppos, cur_val=NULL, cur_len=PAGE_SIZE, new_val=NULL, new_len=0, new_updated=0 }.
- ctx.cur_val = kmalloc(PAGE_SIZE). If !cur_val or read fails via table.proc_handler: cur_len = 0.
- if write ∧ *buf ∧ *pcount: ctx.new_val = kmalloc(PAGE_SIZE); new_len = min(PAGE_SIZE, *pcount); memcpy(new_val, *buf, new_len). if !new_val: new_len = 0.
- rcu_read_lock; cgrp = task_dfl_cgroup(current); ret = bpf_prog_run_array_cg(&cgrp.bpf, atype, &ctx, bpf_prog_run, 0, NULL); rcu_read_unlock.
- if ret == 1 ∧ ctx.new_updated: kfree(*buf); *buf = ctx.new_val; *pcount = ctx.new_len.
- else: kfree(ctx.new_val). kfree(ctx.cur_val).

REQ-32: __cgroup_bpf_run_filter_setsockopt(sk, *level, *optname, optval, *optlen, **kernel_optval) (BPF_CGROUP_SETSOCKOPT):
- ctx = { sk, level=*level, optname=*optname, optlen=*optlen }.
- max_optlen = max(16, *optlen); allocate scratch (inline buf or kzalloc up to PAGE_SIZE).
- copy_from_sockptr(ctx.optval, optval, min(*optlen, max_optlen)).
- lock_sock(sk); ret = bpf_prog_run_array_cg(&cgrp.bpf, CGROUP_SETSOCKOPT, &ctx, bpf_prog_run, 0, NULL); release_sock(sk).
- if ctx.optlen == -1: ret = 1 /* bypass kernel handler */.
- else if ctx.optlen ∉ [0, max_optlen]: -EFAULT (or rate-limited pr_info_once if user buffer > PAGE_SIZE).
- else: *level = ctx.level; *optname = ctx.optname; if ctx.optlen != 0: *optlen = ctx.optlen; if !inline-buf: *kernel_optval = ctx.optval (keep alloc); else: kmalloc + memcpy.

REQ-33: __cgroup_bpf_run_filter_getsockopt(sk, level, optname, optval, optlen, max_optlen, retval) (BPF_CGROUP_GETSOCKOPT):
- ctx = { sk, level, optname, current_task=current }.
- if !retval: copy_from_sockptr(&ctx.optlen, optlen); copy_from_sockptr(ctx.optval, optval, min(ctx.optlen, max_optlen)).
- lock_sock(sk); ret = bpf_prog_run_array_cg(&cgrp.bpf, CGROUP_GETSOCKOPT, &ctx, bpf_prog_run, retval, NULL); release_sock(sk).
- if ctx.optlen ∉ [0, max_optlen]: rate-limited info; -EFAULT.
- if ctx.optlen != 0: copy_to_sockptr(optval, ctx.optval, ctx.optlen); copy_to_sockptr(optlen, &ctx.optlen).

REQ-34: __cgroup_bpf_run_filter_getsockopt_kern(sk, level, optname, optval, *optlen, retval):
- ctx = { sk, level, optname, optlen=*optlen, optval, optval_end=optval+*optlen, current_task=current }.
- ret = bpf_prog_run_array_cg(&cgrp.bpf, CGROUP_GETSOCKOPT, &ctx, bpf_prog_run, retval, NULL).
- if ctx.optlen > *optlen: return -EFAULT /* program tried to extend */.
- if ctx.optlen != 0: *optlen = ctx.optlen /* shrink permitted */.

REQ-35: bpf_get_local_storage(map, flags) — helper:
- stype = cgroup_storage_type(map) /* SHARED or PERCPU */.
- ctx = container_of(current.bpf_ctx, bpf_cg_run_ctx, run_ctx).
- storage = ctx.prog_item.cgroup_storage[stype].
- if stype == SHARED: return &READ_ONCE(storage.buf).data[0]; else: return this_cpu_ptr(storage.percpu_buf).

REQ-36: bpf_get_retval / bpf_set_retval — helpers:
- ctx = container_of(current.bpf_ctx, bpf_cg_run_ctx, run_ctx).
- get: return ctx.retval. set: ctx.retval = retval; return 0.
- cgroup_common_func_proto gates availability: per-attach-type — INGRESS/EGRESS/SOCK_OPS/UDP4/6_RECVMSG/UNIX_RECVMSG/GETPEERNAME/GETSOCKNAME → return NULL (not allowed); others → proto.

REQ-37: bpf_cgroup_storages_alloc(storages, new_storages, type, prog, cgrp):
- key = { cgroup_inode_id = cgroup_id(cgrp), attach_type = type }.
- for stype in storage_types:
  - map = prog.aux.cgroup_storage[stype]; if !map: continue.
  - storages[stype] = cgroup_storage_lookup(map, &key, false).
  - if existing: continue /* reused */.
  - storages[stype] = bpf_cgroup_storage_alloc(prog, stype); if IS_ERR: free new_storages; return -ENOMEM.
  - new_storages[stype] = storages[stype].

REQ-38: bpf_cgroup_storages_link(storages, cgrp, attach_type):
- for stype: bpf_cgroup_storage_link(storages[stype], cgrp, attach_type) /* adds to cgrp.bpf.storages list */.

REQ-39: bpf_cgroup_atype_find(attach_type, attach_btf_id) (CONFIG_BPF_LSM):
- if attach_type != BPF_LSM_CGROUP: return to_cgroup_bpf_attach_type(attach_type).
- /* LSM_CGROUP: dynamically allocated slot keyed by btf_id */
- lockdep_assert_held(&cgroup_mutex).
- search cgroup_lsm_atype[i].attach_btf_id == attach_btf_id → return CGROUP_LSM_START + i.
- else find free slot (attach_btf_id == 0) → return slot.
- else return -E2BIG.

REQ-40: bpf_cgroup_atype_get(attach_btf_id, cgroup_atype):
- i = cgroup_atype - CGROUP_LSM_START. WARN_ON_ONCE if attach_btf_id mismatch.
- cgroup_lsm_atype[i].attach_btf_id = attach_btf_id; refcnt++.

REQ-41: bpf_cgroup_atype_put(cgroup_atype):
- cgroup_lock; if --refcnt ≤ 0: attach_btf_id = 0; WARN if refcnt < 0; cgroup_unlock.

REQ-42: __cgroup_bpf_run_lsm_sock / _socket / _current — trampoline shims:
- shim_prog = (insn - offsetof(bpf_prog, insnsi)) /* shim emitted into prog->insnsi */.
- sock/socket variant: cgrp = sock_cgroup_ptr(&sk.sk_cgrp_data).
- current variant: cgrp = task_dfl_cgroup(current) /* requires trampoline's __bpf_prog_enter_lsm_cgroup to hold RCU */.
- if cgrp: bpf_prog_run_array_cg(&cgrp.bpf, shim_prog.aux.cgroup_atype, ctx, bpf_prog_run, 0, NULL).

REQ-43: cgroup_bpf_enabled_key[atype] — static-branch:
- DEFINE_STATIC_KEY_ARRAY_FALSE(cgroup_bpf_enabled_key, MAX_CGROUP_BPF_ATTACH_TYPE).
- static_branch_inc on first attach of atype; dec on detach (or release).
- Call sites guard with `static_branch_unlikely(&cgroup_bpf_enabled_key[atype])` so no programs == zero overhead.

REQ-44: BPF_CGROUP_MAX_PROGS = 64 — per-cgroup-per-atype limit (`__cgroup_bpf_attach` returns -E2BIG above this).

REQ-45: ARRAY_CREATE_FLAG_MASK at attach (BPF_F_ALLOW_OVERRIDE / _ALLOW_MULTI / _REPLACE / _BEFORE / _AFTER / _ID / _LINK / _PREORDER):
- Validation matrix:
  - OVERRIDE ∧ MULTI → -EINVAL.
  - REPLACE ∧ !MULTI → -EINVAL.
  - REPLACE ∧ (BEFORE ∨ AFTER) → -EINVAL.
  - LINK requires link != NULL; non-LINK requires prog != NULL.
  - BEFORE ∧ AFTER on existing list → -EINVAL.

REQ-46: link-update lifecycle:
- cgroup_bpf_replace(link, new_prog, old_prog) → cgroup_lock → if !cg_link.cgroup: -ENOLINK → if old_prog ∧ link.prog != old_prog: -EPERM → __cgroup_bpf_replace.

## Acceptance Criteria

- [ ] AC-1: cgroup_bpf_inherit creates per-atype empty progs[] hlists + computes effective[] from parent chain; on OOM propagates -ENOMEM and releases percpu_ref + parent refs.
- [ ] AC-2: cgroup_bpf_offline → percpu_ref_kill → cgroup_bpf_release runs on cgroup_bpf_destroy_wq and drains progs[]/storages, decrements `cgroup_bpf_enabled_key` per attachment, frees effective[].
- [ ] AC-3: hierarchy_allows_attach returns false if any non-MULTI ancestor has a non-OVERRIDE prog attached.
- [ ] AC-4: __cgroup_bpf_attach rejects BPF_F_ALLOW_OVERRIDE | BPF_F_ALLOW_MULTI together → -EINVAL.
- [ ] AC-5: __cgroup_bpf_attach rejects BPF_F_REPLACE without BPF_F_ALLOW_MULTI → -EINVAL.
- [ ] AC-6: __cgroup_bpf_attach with mismatched flags vs existing cgroup.bpf.flags[atype] → -EPERM.
- [ ] AC-7: __cgroup_bpf_attach above 64 progs at a single (cgroup, atype) → -E2BIG.
- [ ] AC-8: __cgroup_bpf_attach with expected_revision mismatch → -ESTALE.
- [ ] AC-9: compute_effective_progs orders pre-order (BPF_F_PREORDER) entries before normal entries at each cgroup level; reverses pre-order block per level so child declarations execute in declaration order.
- [ ] AC-10: update_effective_progs is atomic per descendant: allocate-all then activate-all; OOM leaves no descendant with stale effective.
- [ ] AC-11: __cgroup_bpf_detach on OOM during update calls purge_effective_progs as in-place recovery.
- [ ] AC-12: cgroup_bpf_replace (link path) atomically swaps link.prog and rewrites every descendant's effective[atype].items[pos].prog via WRITE_ONCE; bpf_prog_put on old prog.
- [ ] AC-13: bpf_prog_run_array_cg sets retval = -EPERM the first time any prog returns 0 (default-deny semantics).
- [ ] AC-14: BPF_CGROUP_INET_EGRESS return-value mapping: 0 → NET_XMIT_DROP, 1 → NET_XMIT_SUCCESS, 2/3 → cn variants; errors carried through.
- [ ] AC-15: BPF_CGROUP_SOCK_ADDR with AF_UNIX allows program to modify uaddrlen; INET[6] keeps uaddrlen read-only.
- [ ] AC-16: BPF_CGROUP_SYSCTL with prog returning 1 and ctx.new_updated commits ctx.new_val/new_len back to *buf/*pcount.
- [ ] AC-17: BPF_CGROUP_SETSOCKOPT with ctx.optlen == -1 bypasses kernel handler (returns 1); with positive optlen ≤ PAGE_SIZE re-exports modified buffer; >PAGE_SIZE rejected.
- [ ] AC-18: BPF_CGROUP_GETSOCKOPT may shrink optlen but not extend (kernel variant returns -EFAULT if extended).
- [ ] AC-19: bpf_get_local_storage returns ctx.prog_item.cgroup_storage[stype] (SHARED: pointer to data buffer; PERCPU: this_cpu_ptr).
- [ ] AC-20: BPF_LSM_CGROUP attach finds free cgroup_lsm_atype slot or returns -E2BIG; trampoline shim installed via bpf_trampoline_link_cgroup_shim.
- [ ] AC-21: bpf_cgroup_link_release calls __cgroup_bpf_detach; cgroup death auto-detaches via bpf_cgroup_link_auto_detach without freeing link memory.
- [ ] AC-22: cgroup_bpf_enabled_key[atype] is incremented exactly when first prog attached at that atype anywhere in the system; decremented on last detach.
- [ ] AC-23: BPF_PROG_QUERY effective vs non-effective: returns either effective-array (post-hierarchy) or cgroup-local list; copy_to_user reports prog_cnt, attach_flags (0 for effective query), revision.
- [ ] AC-24: per-cgroup recursion: bpf_set_run_ctx nests bpf_cg_run_ctx so a hook invoking another hook (e.g. SOCK_OPS triggering SETSOCKOPT) preserves outer retval/storage context.

## Architecture

```
struct CgroupBpf {
  progs: [HlistHead<BpfProgList>; MAX_CGROUP_BPF_ATTACH_TYPE],   // local attachments
  effective: [RcuPtr<BpfProgArray>; MAX_CGROUP_BPF_ATTACH_TYPE], // runtime chain
  inactive: [Option<*BpfProgArray>; MAX_CGROUP_BPF_ATTACH_TYPE], // scratch
  flags: [u32; MAX_CGROUP_BPF_ATTACH_TYPE],                      // ALLOW_OVERRIDE | ALLOW_MULTI
  revisions: [u64; MAX_CGROUP_BPF_ATTACH_TYPE],                  // monotonic counter
  storages: list_head<BpfCgroupStorage>,                         // owned local storage
  refcnt: PercpuRef,                                             // killed by cgroup_bpf_offline
  release_work: WorkStruct,                                      // cgroup_bpf_destroy_wq
}

struct BpfProgList {
  prog: Option<*BpfProg>,                                        // direct attach
  link: Option<*BpfCgroupLink>,                                  // or link attach
  flags: u32,                                                    // BPF_F_PREORDER, ALLOW_*
  storage: [Option<*BpfCgroupStorage>; MAX_BPF_CGROUP_STORAGE_TYPE],
  node: HlistNode,
}

struct BpfCgroupLink {
  link: BpfLink,                                                 // embeds prog, lops, attach_type
  cgroup: Option<*Cgroup>,                                       // None after auto-detach
}

enum CgroupBpfAttachType {
  CGROUP_INET_INGRESS,
  CGROUP_INET_EGRESS,
  CGROUP_INET_SOCK_CREATE,
  CGROUP_INET_SOCK_RELEASE,
  CGROUP_SOCK_OPS,
  CGROUP_DEVICE,
  CGROUP_INET4_BIND, CGROUP_INET6_BIND,
  CGROUP_INET4_POST_BIND, CGROUP_INET6_POST_BIND,
  CGROUP_INET4_CONNECT, CGROUP_INET6_CONNECT,
  CGROUP_UNIX_CONNECT,
  CGROUP_UDP4_SENDMSG, CGROUP_UDP6_SENDMSG, CGROUP_UNIX_SENDMSG,
  CGROUP_UDP4_RECVMSG, CGROUP_UDP6_RECVMSG, CGROUP_UNIX_RECVMSG,
  CGROUP_INET4_GETPEERNAME, CGROUP_INET6_GETPEERNAME, CGROUP_UNIX_GETPEERNAME,
  CGROUP_INET4_GETSOCKNAME, CGROUP_INET6_GETSOCKNAME, CGROUP_UNIX_GETSOCKNAME,
  CGROUP_SYSCTL,
  CGROUP_SETSOCKOPT, CGROUP_GETSOCKOPT,
  CGROUP_LSM_START..CGROUP_LSM_END,
}
```

`Cgroup::bpf_inherit(cgrp) -> Result<()>`:
1. percpu_ref_init(&cgrp.bpf.refcnt, Cgroup::bpf_release_fn, 0, GFP_KERNEL)?.
2. for p = cgroup_parent(cgrp); p; p = cgroup_parent(p): cgroup_bpf_get(p) /* take parent refs */.
3. for i in 0..NR: INIT_HLIST_HEAD(&cgrp.bpf.progs[i]).
4. INIT_LIST_HEAD(&cgrp.bpf.storages).
5. for i in 0..NR: Cgroup::compute_effective_progs(cgrp, i, &arrays[i])?.
6. for i in 0..NR: Cgroup::activate_effective_progs(cgrp, i, arrays[i]).
7. on err: free arrays; cgroup_bpf_put parents; percpu_ref_exit; return -ENOMEM.

`Cgroup::compute_effective_progs(cgrp, atype, *array) -> Result<()>`:
1. /* Pass 1: count cnt + preorder_cnt walking up */
2. p = cgrp; cnt = 0; preorder_cnt = 0.
3. while p:
   - if cnt == 0 ∨ (p.bpf.flags[atype] & BPF_F_ALLOW_MULTI): cnt += prog_list_length(&p.bpf.progs[atype], &preorder_cnt).
   - p = cgroup_parent(p).
4. progs = bpf_prog_array_alloc(cnt, GFP_KERNEL)?.
5. /* Pass 2: populate fstart-ascending for normal, bstart-descending for preorder */
6. cnt = 0; p = cgrp; fstart = preorder_cnt; bstart = preorder_cnt - 1.
7. while p:
   - if cnt > 0 ∧ !(p.bpf.flags[atype] & BPF_F_ALLOW_MULTI): goto next.
   - init_bstart = bstart.
   - for pl in p.bpf.progs[atype]:
     - if !prog_list_prog(pl): continue.
     - item = (pl.flags & BPF_F_PREORDER) ? &progs.items[bstart--] : &progs.items[fstart++].
     - item.prog = prog_list_prog(pl); Cgroup::storages_assign(item.cgroup_storage, pl.storage).
     - cnt++.
   - /* Reverse preorder block at this level so child declarations run in order */
   - for i = bstart+1, j = init_bstart; i < j; i++, j--: swap.
   - next: p = cgroup_parent(p).
8. *array = progs; return Ok(()).

`Cgroup::bpf_attach_inner(cgrp, prog, replace_prog, link, type, flags, id_or_fd, revision) -> Result<()>` — under cgroup_mutex:
1. /* Flag combinator checks (REQ-12) */ → -EINVAL on incompatibilities.
2. atype = Cgroup::atype_find(type, new_prog.aux.attach_btf_id)?.
3. if revision ∧ revision != cgrp.bpf.revisions[atype]: return -ESTALE.
4. if !Cgroup::hierarchy_allows_attach(cgrp, atype): return -EPERM.
5. if !hlist_empty ∧ cgrp.bpf.flags[atype] != saved_flags: return -EPERM.
6. if prog_list_length ≥ 64: return -E2BIG.
7. pl = Cgroup::find_attach_entry(progs, prog, link, replace_prog, allow_multi)?.
8. Cgroup::storages_alloc(storage, new_storage, type, prog ? : link.prog, cgrp)?.
9. if pl: old_prog = pl.prog (REPLACE).
10. else: pl = kzalloc; Cgroup::insert_pl_to_hlist(pl, progs, prog, link, flags, id_or_fd)?.
11. pl.prog = prog; pl.link = link; pl.flags = flags; Cgroup::storages_assign(pl.storage, storage); cgrp.bpf.flags[atype] = saved_flags.
12. if type == BPF_LSM_CGROUP: bpf_trampoline_link_cgroup_shim(new_prog, atype, type)?.
13. Cgroup::update_effective_progs(cgrp, atype)?.
14. cgrp.bpf.revisions[atype]++.
15. if old_prog: (LSM_CGROUP: unlink shim); bpf_prog_put(old_prog).
16. else: static_branch_inc(&cgroup_bpf_enabled_key[atype]).
17. Cgroup::storages_link(new_storage, cgrp, type).
18. on err: revert pl, unlink shim if installed, free new_storage.

`Cgroup::bpf_detach_inner(cgrp, prog, link, type, revision) -> Result<()>`:
1. atype = Cgroup::atype_find(type, btf_id)?.
2. if revision mismatch: -ESTALE.
3. pl = Cgroup::find_detach_entry(progs, prog, link, allow_multi)?.
4. /* Tombstone */
5. old_prog = pl.prog; pl.prog = None; pl.link = None.
6. if Cgroup::update_effective_progs(cgrp, atype).is_err():
   - pl.prog = Some(old_prog); pl.link = link.
   - Cgroup::purge_effective_progs(cgrp, old_prog, link, atype) /* delete-safe-at recovery */.
7. hlist_del(&pl.node); cgrp.bpf.revisions[atype]++; kfree(pl).
8. if hlist_empty: cgrp.bpf.flags[atype] = 0.
9. (LSM_CGROUP: unlink shim); bpf_prog_put(old_prog).
10. static_branch_dec(&cgroup_bpf_enabled_key[atype]).

`Cgroup::bpf_replace_inner(cgrp, link, new_prog) -> Result<()>`:
1. atype = Cgroup::atype_find(link.attach_type, new_prog.aux.attach_btf_id)?.
2. if link.prog.type != new_prog.type: return -EINVAL.
3. find pl with pl.link == link in cgrp.bpf.progs[atype] or -ENOENT.
4. cgrp.bpf.revisions[atype]++.
5. old_prog = xchg(&link.link.prog, new_prog).
6. Cgroup::replace_effective_prog(cgrp, atype, link) /* descend + WRITE_ONCE */.
7. bpf_prog_put(old_prog).

`Cgroup::run_array_cg(cgrp_bpf, atype, ctx, run_prog, retval, *ret_flags) -> i32`:
1. run_ctx.retval = retval.
2. rcu_read_lock_dont_migrate.
3. array = rcu_dereference(cgrp_bpf.effective[atype]).
4. item = &array.items[0].
5. old_run_ctx = bpf_set_run_ctx(&run_ctx.run_ctx).
6. while prog = READ_ONCE(item.prog):
   - run_ctx.prog_item = item.
   - func_ret = run_prog(prog, ctx).
   - if ret_flags: *ret_flags |= func_ret >> 1; func_ret &= 1.
   - if !func_ret ∧ !IS_ERR_VALUE(run_ctx.retval): run_ctx.retval = -EPERM.
   - item++.
7. bpf_reset_run_ctx(old_run_ctx); rcu_read_unlock_migrate.
8. return run_ctx.retval.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cgroup_mutex_held_for_attach_detach` | INVARIANT | per-__cgroup_bpf_attach / _detach / _replace / _query: cgroup_mutex held. |
| `effective_rcu_protected` | INVARIANT | per-bpf_prog_run_array_cg: rcu_dereference(effective[atype]) within rcu_read_lock_dont_migrate. |
| `flag_combinators_rejected` | INVARIANT | per-attach: (ALLOW_OVERRIDE ∧ ALLOW_MULTI) ∨ (REPLACE ∧ !MULTI) ∨ (REPLACE ∧ (BEFORE ∨ AFTER)) → -EINVAL. |
| `mode_consistent_across_attaches` | INVARIANT | per-attach: cgrp.bpf.flags[atype] == saved_flags (or list empty). |
| `prog_count_capped_at_64` | INVARIANT | per-attach: prog_list_length(&cgrp.bpf.progs[atype]) ≤ BPF_CGROUP_MAX_PROGS. |
| `static_key_balanced` | INVARIANT | per-attach: static_branch_inc on first attach of atype; static_branch_dec on last detach. |
| `effective_swap_under_rcu` | INVARIANT | per-activate_effective_progs: rcu_replace_pointer + grace-period free of old array. |
| `revision_monotonic` | INVARIANT | per-attach / detach / replace: cgrp.bpf.revisions[atype] strictly increases. |
| `default_deny_on_zero_return` | INVARIANT | per-run_array_cg: any prog returning 0 → run_ctx.retval = -EPERM (unless already error). |
| `link_release_idempotent` | INVARIANT | per-bpf_cgroup_link_release: if cg_link.cgroup == NULL: no-op (auto-detached already). |
| `lsm_atype_refcnt_nonneg` | INVARIANT | per-bpf_cgroup_atype_put: WARN_ON refcnt < 0. |
| `purge_only_on_oom` | INVARIANT | per-detach: purge_effective_progs called only after update_effective_progs returns err. |
| `effective_array_walks_terminated_by_null_prog` | INVARIANT | per-run_array_cg: bpf_prog_array_alloc reserves a trailing NULL-prog sentinel. |
| `link_attach_only_via_multi` | INVARIANT | per-cgroup_bpf_link_attach: BPF_F_ALLOW_MULTI implicitly OR'd into flags. |
| `mode_zero_when_list_empty` | INVARIANT | per-detach: hlist_empty(progs) → cgrp.bpf.flags[atype] = 0. |

### Layer 2: TLA+

`kernel/bpf/cgroup-bpf.tla`:
- Per-cgroup-online → cgroup_bpf_inherit + initial effective[].
- Per-attach + per-detach + per-replace + per-query under cgroup_mutex.
- Per-descendant-update propagating compute_effective_progs.
- Per-hook fast-path bpf_prog_run_array_cg under RCU.
- Per-cgroup-offline → percpu_ref_kill → cgroup_bpf_release → drain progs + auto-detach links + free effective + free storages.
- Properties:
  - `safety_attach_mode_consistent` — per-(cgroup, atype): all attachments share the same flags.
  - `safety_hierarchy_compat` — per-attach: hierarchy_allows_attach(cgrp, atype) ⟹ ancestors are MULTI or have ≤ 1 OVERRIDE prog.
  - `safety_effective_is_local_plus_inheritable_ancestors` — per-effective: items = local progs ∪ MULTI-flagged ancestors' progs (pre-order block first per level, reversed).
  - `safety_replace_atomic` — per-replace: every descendant's effective[atype] items[pos].prog updated atomically; old prog dropped after grace period.
  - `safety_oom_on_detach_recovers` — per-detach: on update OOM, purge_effective_progs leaves descendants without the detached prog.
  - `safety_storage_owned_by_cgroup` — per-storage: link/unlink balanced; released at cgroup_bpf_release.
  - `safety_link_auto_detach_on_offline` — per-link: if its cgroup goes offline, cg_link.cgroup becomes NULL; bpf_cgroup_link_release becomes a no-op.
  - `safety_lsm_atype_slot_unique` — per-(attach_btf_id): cgroup_lsm_atype slot reused via refcnt; not reassigned while refcnt > 0.
  - `liveness_per_release_completes` — per-cgroup_bpf_release: queued on dedicated cgroup_bpf_destroy_wq → eventually drains progs + storages + effective; cgroup_put balances cgroup_get from cgroup_bpf_offline.
  - `liveness_per_run_array_cg_terminates` — per-hook invocation: walks finite items[] chain; returns within O(num_progs).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Cgroup::bpf_inherit` post: refcnt initialized; progs[] empty; effective[] computed from parents | `Cgroup::bpf_inherit` |
| `Cgroup::compute_effective_progs` post: array.items[..cnt].prog ∈ {parents' MULTI progs} ∪ {cgrp's progs}; pre-order block reversed per level | `Cgroup::compute_effective_progs` |
| `Cgroup::bpf_attach_inner` post: pl in cgrp.bpf.progs[atype]; effective[] for desc-tree updated; revisions++ | `Cgroup::bpf_attach_inner` |
| `Cgroup::bpf_detach_inner` post: pl removed; effective[] for desc-tree updated (or in-place purged); revisions++ | `Cgroup::bpf_detach_inner` |
| `Cgroup::bpf_replace_inner` post: every descendant's effective items[pos].prog == new_prog; old_prog put | `Cgroup::bpf_replace_inner` |
| `Cgroup::run_array_cg` post: ret == final run_ctx.retval; -EPERM if any prog returned 0 (default-deny) | `Cgroup::run_array_cg` |
| `Cgroup::run_filter_skb` post: skb data-end restored; skb.sk restored; return mapped to NET_XMIT_* (EGRESS) or 0/-EPERM (INGRESS) | `Cgroup::run_filter_skb` |
| `Cgroup::run_filter_sysctl` post: if prog returned 1 and new_updated: *buf == new_val, *pcount == new_len; cur_val freed | `Cgroup::run_filter_sysctl` |
| `Cgroup::run_filter_setsockopt` post: ctx.optlen ∈ {-1, [0, max_optlen]}; kernel_optval pointer valid if optlen != 0 | `Cgroup::run_filter_setsockopt` |
| `Cgroup::run_filter_getsockopt_kern` post: ctx.optlen ≤ *optlen (no extension); *optlen reflects shrink | `Cgroup::run_filter_getsockopt_kern` |
| `Cgroup::bpf_release` post: progs[] drained; effective[] freed; storages unlinked; parents' refs put; refcnt exited | `Cgroup::bpf_release` |
| `Cgroup::link_release` post: link's cgroup detached or already None; cgroup_put paired with cgroup_get on attach | `Cgroup::link_release` |

### Layer 4: Verus/Creusot functional

`Per-online: cgroup_bpf_inherit → percpu_ref_init + cgroup_bpf_get(parents) + compute_effective_progs(per-atype) + activate_effective_progs.` `Per-attach: __cgroup_bpf_attach → flag-combinator + hierarchy_allows_attach + find_attach_entry + storages_alloc + insert_pl_to_hlist + update_effective_progs (per-descendant compute_effective_progs + activate) + revisions++ + static_branch_inc.` `Per-detach: __cgroup_bpf_detach → find_detach_entry → tombstone → update_effective_progs (purge on OOM) → hlist_del + revisions++ + bpf_prog_put + static_branch_dec.` `Per-replace: __cgroup_bpf_replace → xchg(link.prog) + replace_effective_prog (descendants WRITE_ONCE items[pos].prog) + bpf_prog_put.` `Per-hook: bpf_prog_run_array_cg under rcu_read_lock_dont_migrate iterates effective[atype].items until READ_ONCE NULL.` `Per-skb (INGRESS / EGRESS): __cgroup_bpf_run_filter_skb manages __skb_push/_pull + bpf_compute_and_save_data_end + EGRESS return-mapping.` `Per-sock_addr: __cgroup_bpf_run_filter_sock_addr supports AF_UNIX uaddrlen modification.` `Per-sysctl: cur_val + new_val buffers; ret==1 + new_updated commits.` `Per-sockopt: setsockopt {ctx.optlen==-1 bypass; ≥0 ≤PAGE_SIZE re-export} / getsockopt {shrink-only} / getsockopt_kern {strict no-extend}.` `Per-LSM_CGROUP: __cgroup_bpf_run_lsm_* shim resolves cgroup via sock/socket/current and dispatches via shim_prog.aux.cgroup_atype.` `Per-offline: cgroup_bpf_offline → percpu_ref_kill → cgroup_bpf_release_fn schedules release_work on cgroup_bpf_destroy_wq → cgroup_bpf_release drains progs[] (per-LSM_CGROUP unlinks shim), auto-detaches links, frees effective[] post-grace, releases storages, cgroup_bpf_put(parents), percpu_ref_exit.` Semantic equivalence: per-`Documentation/bpf/prog_cgroup_v2.rst` + `Documentation/bpf/prog_lsm.rst` + `Documentation/bpf/cgroup-bpf.rst` + `tools/testing/selftests/bpf/test_cgroup_storage.c` + `tools/testing/selftests/bpf/progs/test_sock_addr*.c` + `tools/testing/selftests/bpf/progs/test_sockopt*.c` + `tools/testing/selftests/bpf/progs/test_sysctl*.c` + `tools/testing/selftests/bpf/progs/test_cgroup_link.c` + `tools/testing/selftests/bpf/progs/lsm_cgroup.c`.

## Hardening

(Inherits row-1 features from `kernel/bpf/00-overview.md` § Hardening.)

BPF-cgroup reinforcement:

- **Per-cgroup_mutex around attach / detach / replace / query** — defense against per-list-corruption + per-hierarchy-race; rcu_dereference_protected uses lockdep_is_held(&cgroup_mutex).
- **Per-attach mode validation** — defense against per-mode-confusion: ALLOW_OVERRIDE and ALLOW_MULTI are mutually exclusive; REPLACE requires MULTI; per-cgroup-per-atype mode immutable while list non-empty.
- **Per-BPF_CGROUP_MAX_PROGS=64 cap** — defense against per-attach DoS (effective array grows linearly with attach count).
- **Per-expected_revision check** — defense against per-stale-attach races (caller asserts revision; -ESTALE on mismatch).
- **Per-hierarchy_allows_attach walk** — defense against per-policy-bypass: non-MULTI, non-OVERRIDE ancestor blocks descendant attach.
- **Per-update_effective_progs two-phase (allocate-all then activate-all)** — defense against per-descendant-inconsistency on OOM.
- **Per-purge_effective_progs in-place recovery on detach OOM** — defense against per-detach-failure leaving stale progs in effective[].
- **Per-RCU-protected effective array** — defense against per-walk-during-update UAF: rcu_replace_pointer + bpf_prog_array_free deferred to grace period.
- **Per-static-branch cgroup_bpf_enabled_key** — defense against per-hook overhead when no programs attached (zero-cost call site via static_branch_unlikely).
- **Per-cgroup_bpf_destroy_wq dedicated workqueue** — defense against per-system_percpu_wq saturation under heavy cgroup destruction concurrency (max_active set to 1 separate from system).
- **Per-default-deny (-EPERM on zero-return)** — defense against per-misimplemented-prog: any prog returning 0 sets retval to -EPERM unless prior prog set an explicit errno.
- **Per-IS_ERR_VALUE retval preservation** — defense against per-error-clobber: once a prog sets retval to an error, subsequent 0-returns do not overwrite.
- **Per-rcu_read_lock_dont_migrate in run_array_cg** — defense against per-CPU-migration-during-walk + per-grace-period-elision: migration disabled for prog execution.
- **Per-bpf_set_run_ctx nesting** — defense against per-helper-context-leak: outer bpf_get_local_storage / bpf_get_retval restored after nested hook.
- **Per-LSM_CGROUP cgroup_atype slot refcounted** — defense against per-btf_id-slot collision: slot reused via refcnt; WARN_ON underflow.
- **Per-link cgroup auto-detach** — defense against per-link-outlives-cgroup: cgroup death sets link.cgroup = NULL; link release becomes idempotent no-op.
- **Per-trampoline shim link/unlink for BPF_LSM_CGROUP** — defense against per-stale-LSM-hook: shim removed on detach / release before bpf_prog_put.
- **Per-sock-cgroup-ptr deref via sock_cgroup_ptr / task_dfl_cgroup** — defense against per-stale-cgroup-pointer on sock close: sk.sk_cgrp_data carries cgroup pinned for sk lifetime.
- **Per-AF_UNIX uaddrlen modification gated** — defense against per-INET-address-truncation: only AF_UNIX allows uaddrlen rewrite.
- **Per-sysctl new_val PAGE_SIZE cap** — defense against per-unbounded-write: bpf_sysctl_set_new_value rejects buf_len > PAGE_SIZE - 1.
- **Per-setsockopt PAGE_SIZE optlen cap** — defense against per-large-buffer DoS: max_optlen clamped to PAGE_SIZE; oversized rejected with pr_info_once rate-limit.
- **Per-getsockopt_kern no-extend** — defense against per-buffer-overflow: kernel-internal getsockopt path rejects ctx.optlen > *optlen.
- **Per-cgroup_storage indexed by (cgroup_inode_id, attach_type)** — defense against per-cross-attach storage leak: each (cgroup, attach_type) gets its own storage entry; list-cg owns lifetime.
- **Per-percpu_ref refcnt** — defense against per-release-while-walking: cgroup_bpf_offline → percpu_ref_kill ensures all readers drain before cgroup_bpf_release.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-check `copy_from_user` on every cgroup-BPF context (sockopt buf, sysctl new_val, sockaddr) against the per-prog `ctx_size` and slab whitelist; reject if length crosses PAGE_SIZE.
- **PAX_KERNEXEC** — keep `cgroup_bpf_attach_ops`, the per-attach-type `bpf_cgroup_link_lops`, and the `cgroup_bpf` effective-prog dispatch arrays in `__ro_after_init`.
- **PAX_RANDKSTACK** — re-randomize stack on every cgroup-BPF run point (`__cgroup_bpf_run_filter_*`); the on-stack `bpf_sockopt_kern`/`bpf_sysctl_kern` are stable disclosure targets.
- **PAX_REFCOUNT** — saturating refcount on `struct bpf_cgroup_link`, `struct bpf_prog`, and the cgroup `percpu_ref`; wraparound MUST panic.
- **PAX_MEMORY_SANITIZE** — zero `bpf_cgroup_storage` and `bpf_sockopt_kern.optval_end - optval` scratch on release; never recycle prior cgroup's per-cpu storage.
- **PAX_UDEREF** — strict user/kernel separation when reading sockopt/sysctl user buffers from the cgroup-BPF context.
- **PAX_RAP / kCFI** — type-check every `cgroup_bpf_link_lops->update_prog`/`detach`, the `bpf_prog_run`/`bpf_func` dispatch, and the `BPF_LSM_CGROUP` shim trampolines.
- **GRKERNSEC_HIDESYM** — hide `cgroup_bpf_*`, `__cgroup_bpf_run_filter_*`, and per-attach-type symbols from non-root kallsyms.
- **GRKERNSEC_DMESG** — restrict dmesg so per-attach error spew (which echoes cgroup ids and prog tags) is not harvestable.
- **Per-cgroup BPF attach** — `CAP_NET_ADMIN`/`CAP_SYS_ADMIN`/`CAP_BPF` MUST be re-checked against the *current* creds at every `BPF_PROG_ATTACH`/`BPF_LINK_CREATE`, not the cgroupfs-open creds; a SCM_RIGHTS-passed cgroup fd MUST NOT escalate attach rights.
- **`BPF_F_ALLOW_MULTI` strict** — when `BPF_F_ALLOW_MULTI` is set, every per-cgroup prog list ordering MUST be stable across `cgroup_bpf_inherit`; ancestor progs run first, descendant progs last, and a child cgroup MUST NOT override an ancestor's `BPF_F_ALLOW_OVERRIDE=0` attachment. Under grsec, any inheritance anomaly triggers `audit_panic`.
- **Rationale** — cgroup-BPF runs attacker-influenced programs in the critical path of every socket/sockopt/sysctl in the cgroup; PAX_USERCOPY on the sockopt/sysctl buffers, PAX_RAP on the LSM-cgroup shim trampolines, and strict ALLOW_MULTI inheritance close the historical class of cgroup-BPF UAF (link/cgroup release race) and policy-inversion (descendant overriding ancestor) bugs.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-cgroup local-storage map implementation (`kernel/bpf/local_storage.c`) (covered separately if expanded)
- bpf_trampoline_link_cgroup_shim / unlink internals (covered in `bpf-trampoline.md` if expanded)
- bpf_prog_array allocation + per-element item layout (covered in `bpf-core.md` Tier-3)
- bpf_link lifecycle (`bpf_link_init`, `bpf_link_prime`, `bpf_link_settle`, `bpf_link_cleanup`) (covered in `bpf-link.md` if expanded)
- cgroup core (`kernel/cgroup/cgroup.c`) lifetime notifier + `cgroup_lifetime_notifier` (covered in `kernel/cgroup/cgroup.md` Tier-3)
- sock_cgroup_ptr / task_dfl_cgroup semantics (covered in `kernel/cgroup/cgroup.md` Tier-3)
- `__bpf_prog_run_save_cb` / `bpf_compute_and_save_data_end` / `bpf_restore_data_end` (covered in `net-filter.md` if expanded)
- BPF verifier ctx-access conversion + insn rewriting for cgroup hooks (covered in `verifier.md` Tier-3)
- BPF helper protos (`bpf_event_output_data_proto`, `bpf_sk_storage_*`, `bpf_tcp_sock_proto`) (covered in `bpf-helpers.md` if expanded)
- Sysctl table walking (`ctl_table_header`, `proc_handler`) (covered in `fs/proc/proc_sysctl.md` if expanded)
- LSM hook dispatch architecture (`security/security.c`) (covered separately if expanded)
- Implementation code
