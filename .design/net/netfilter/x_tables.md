# Tier-3: net/netfilter/x_tables.c — Xtables shared framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/netfilter/00-overview.md
upstream-paths:
  - net/netfilter/x_tables.c (~2113 lines)
  - include/linux/netfilter/x_tables.h
  - include/uapi/linux/netfilter/x_tables.h
-->

## Summary

Xtables is the **shared backend** for the legacy classifier-family `iptables`, `ip6tables`, `arptables`, and `ebtables`. Per-AF (NFPROTO_IPV4, NFPROTO_IPV6, NFPROTO_ARP, NFPROTO_BRIDGE, NFPROTO_UNSPEC) it owns the per-system registry of **matches** (`struct xt_match`) and **targets** (`struct xt_target`), the per-netns registry of **tables** (`struct xt_table` + `struct xt_table_info`), the per-rule-blob installation primitive (`xt_replace_table`), entry-blob validation (`xt_check_entry_offsets`, `xt_check_match`, `xt_check_target`, `xt_check_table_hooks`), per-CPU jumpstacks for nested target invocation, per-CPU counter allocation (`xt_percpu_counter_alloc`), and `CONFIG_COMPAT` 32-on-64 user/kernel translation (`xt_compat_*`). The per-match/-target evaluator signature is `struct xt_action_param` (datapath) / `struct xt_mtchk_param` + `struct xt_tgchk_param` (control plane). Per-`/proc/net/{ip,ip6,arp,eb}_{tables,matches,targets}` enumeration is provided as a seq_file. Critical for: surviving iptables-legacy traffic, surviving in-place ruleset swap under live traffic, and surviving 32-bit-userspace-on-64-bit-kernel.

This Tier-3 covers `net/netfilter/x_tables.c` (~2113 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct xt_table` | per-table descriptor | `XtTable` |
| `struct xt_table_info` | per-rule-blob | `XtTableInfo` |
| `struct xt_match` | per-match-extension | `XtMatch` |
| `struct xt_target` | per-target-extension | `XtTarget` |
| `struct xt_entry_match` | per-rule match header | `XtEntryMatch` |
| `struct xt_entry_target` | per-rule target header | `XtEntryTarget` |
| `struct xt_action_param` | per-skb match/target ctx | `XtActionParam` |
| `struct xt_mtchk_param` | per-match checkentry ctx | `XtMtchkParam` |
| `struct xt_tgchk_param` | per-target checkentry ctx | `XtTgchkParam` |
| `struct xt_counters` | per-rule pkt/byte counter | `XtCounters` |
| `xt_register_target()` / `_targets()` | per-target register | `Xtables::register_target` |
| `xt_unregister_target()` / `_targets()` | per-target unregister | `Xtables::unregister_target` |
| `xt_register_match()` / `_matches()` | per-match register | `Xtables::register_match` |
| `xt_unregister_match()` / `_matches()` | per-match unregister | `Xtables::unregister_match` |
| `xt_find_match()` | per-name lookup + try_module_get | `Xtables::find_match` |
| `xt_request_find_match()` | per-name lookup + autoload | `Xtables::request_find_match` |
| `xt_request_find_target()` | per-name lookup + autoload | `Xtables::request_find_target` |
| `xt_find_revision()` | per-best-revision search | `Xtables::find_revision` |
| `xt_data_to_user()` / `xt_match_to_user()` / `xt_target_to_user()` | per-getsockopt copy-out | `Xtables::data_to_user` |
| `xt_check_proc_name()` | per-name validation | `Xtables::check_proc_name` |
| `xt_check_match()` | per-match validate | `Xtables::check_match` |
| `xt_check_target()` | per-target validate | `Xtables::check_target` |
| `xt_check_hooks_match()` / `_target()` | per-hook-mask validate | `Xtables::check_hooks` |
| `xt_check_entry_match()` (file-static) | per-match-list contiguity | `Xtables::check_entry_match` |
| `xt_check_entry_offsets()` | per-rule offset audit | `Xtables::check_entry_offsets` |
| `xt_check_table_hooks()` | per-hook entry/underflow audit | `Xtables::check_table_hooks` |
| `xt_alloc_entry_offsets()` | per-jump-target index | `Xtables::alloc_entry_offsets` |
| `xt_find_jump_offset()` | per-jump-target lookup | `Xtables::find_jump_offset` |
| `xt_alloc_table_info()` | per-info kvmalloc | `XtTableInfo::alloc` |
| `xt_free_table_info()` | per-info free | `XtTableInfo::free` |
| `xt_find_table()` | per-net per-af lookup (no lock) | `Xtables::find_table` |
| `xt_find_table_lock()` | per-net per-af + xt[af].mutex | `Xtables::find_table_lock` |
| `xt_request_find_table_lock()` | + per-tablename autoload | `Xtables::request_find_table_lock` |
| `xt_table_unlock()` | per-table mutex release | `Xtables::table_unlock` |
| `xt_replace_table()` | per-blob atomic swap | `Xtables::replace_table` |
| `xt_register_table()` | per-table register + initial blob | `Xtables::register_table` |
| `xt_unregister_table()` | per-table unregister | `Xtables::unregister_table` |
| `xt_register_template()` / `_unregister_template()` | per-larval table | `Xtables::register_template` |
| `xt_hook_ops_alloc()` | per-table nf_hook_ops fan-out | `Xtables::hook_ops_alloc` |
| `xt_proto_init()` / `xt_proto_fini()` | per-net per-af `/proc/net/*_*` | `Xtables::proto_init` |
| `xt_counters_alloc()` | per-rule counters | `XtCounters::alloc` |
| `xt_copy_counters()` | per-getsockopt counters | `XtCounters::copy_user` |
| `xt_percpu_counter_alloc()` / `_free()` | per-CPU rule counter slab | `XtCounters::pcpu_alloc` |
| `xt_compat_add_offset()` / `_init_offsets()` / `_flush_offsets()` / `_calc_jump()` | per-AF 32/64 offset table | `XtCompat::*` |
| `xt_compat_match_offset()` / `_target_offset()` | per-extension size delta | `XtCompat::match_offset` |
| `xt_compat_match_from_user()` / `_target_from_user()` | per-rule 32→64 expand | `XtCompat::match_from_user` |
| `xt_compat_match_to_user()` / `_target_to_user()` | per-rule 64→32 shrink | `XtCompat::match_to_user` |
| `xt_compat_check_entry_offsets()` | per-rule 32-bit offset audit | `XtCompat::check_entry_offsets` |
| `xt_compat_lock()` / `_unlock()` | per-af compat_mutex | `XtCompat::lock` |
| `xt_tee_enabled` static_key | per-TEE target patched | `Xtables::tee_enabled` |
| `xt_recseq` per-CPU `seqcount_t` | per-CPU eval seqcount | `Xtables::recseq` |
| `xt_net_init()` / `xt_net_exit()` | per-netns lifecycle | `Xtables::net_init` |
| `xt_init()` / `xt_fini()` | module-level lifecycle | `Xtables::init` |

## Compatibility contract

REQ-1: struct xt_table (per-table descriptor):
- list: per-netns per-af list_head.
- valid_hooks: per-hook-mask bitfield (NF_INET_PRE_ROUTING etc.).
- private: pointer to current `xt_table_info` (RCU-style, swapped under local_bh_disable + smp_wmb).
- ops: per-table allocated `nf_hook_ops` array.
- table_init: per-larval init callback.
- name: per-table name (XT_TABLE_MAXNAMELEN).
- af: NFPROTO_*.
- priority: per-hook priority.
- me: owning module.

REQ-2: struct xt_table_info (per-rule-blob):
- size: size of `entries[]` in bytes.
- number: per-rule count.
- initial_entries: per-bootstrap baseline (for counter-zeroing decisions).
- hook_entry[NF_INET_NUMHOOKS]: per-hook entry offset within entries[].
- underflow[NF_INET_NUMHOOKS]: per-hook policy/underflow offset.
- stacksize: per-rule max jump depth.
- jumpstack: per-CPU pointer-array of size `stacksize * 2` (the `* 2` reserves an inner frame for `-j TEE` re-entry).
- entries[]: variable-length rule blob.

REQ-3: xt_register_target / _unregister_target:
- af = target.family.
- mutex_lock(xt[af].mutex).
- list_add(&target.list, &xt[af].target). / list_del.
- mutex_unlock.

REQ-4: xt_register_match / _unregister_match:
- af = match.family.
- mutex_lock(xt[af].mutex).
- list_add(&match.list, &xt[af].match). / list_del.
- mutex_unlock.

REQ-5: xt_find_match(af, name, revision):
- if strnlen(name) == XT_EXTENSION_MAXNAMELEN: return ERR_PTR(-EINVAL).
- mutex_lock(xt[af].mutex).
- foreach m in xt[af].match:
  - if strcmp(m.name, name) == 0:
    - if m.revision == revision ∧ try_module_get(m.me): return m.
    - else: err = -EPROTOTYPE.
- mutex_unlock.
- if af != NFPROTO_UNSPEC: recurse with NFPROTO_UNSPEC.
- return ERR_PTR(err = -ENOENT).

REQ-6: xt_request_find_match / xt_request_find_target:
- match = xt_find_match(nfproto, name, revision).
- if IS_ERR(match): request_module("%st_%s", xt_prefix[nfproto], name); match = xt_find_match.
- return match.

REQ-7: xt_find_revision(af, name, revision, target, *err):
- Walk xt[af].{match,target} (per `target` flag) → highest revision ≤ given that is registered.

REQ-8: xt_check_proc_name(name, size):
- name[0] != '\0'.
- name[size-1] == '\0'.
- No '/' in name.
- No "..", no "..".

REQ-9: xt_check_match(par, size, proto, inv_proto):
- xt_check_match_common(par, size, proto, inv_proto):
  - XT_ALIGN(par.match.matchsize) == size OR matchsize == -1.
  - par.match.table NULL OR matches par.table.
  - NFPROTO_ARP only matched by NFPROTO_ARP match family.
  - par.hook_mask ⊆ par.match.hooks.
  - par.match.proto compatible (proto, inv_proto).
- xt_check_hooks_match: optional per-match check_hooks callback.
- xt_checkentry_match: optional per-match checkentry callback; return >0 ⟹ -EIO.

REQ-10: xt_check_target(par, size, proto, inv_proto):
- xt_check_target_common: matching size + table + family + hooks + proto checks.
- xt_check_hooks_target: optional callback.
- xt_checkentry_target: optional callback.

REQ-11: xt_check_entry_match(match, target, alignment):
- length = target − match.
- For each `xt_entry_match` pos in [match, target):
  - pos must be `alignment`-aligned.
  - length ≥ sizeof(struct xt_entry_match).
  - pos.u.match_size ≥ sizeof(struct xt_entry_match).
  - pos.u.match_size ≤ length.
  - length -= pos.u.match_size; advance pos.
- return 0 on success, -EINVAL on any failure.

REQ-12: xt_check_table_hooks(info, valid_hooks):
- Build-bug: ARRAY_SIZE(hook_entry) == ARRAY_SIZE(underflow).
- For each i where valid_hooks & (1<<i):
  - hook_entry[i] != 0xFFFFFFFF ∧ underflow[i] != 0xFFFFFFFF.
  - hook_entry strictly increasing in valid-hook iteration order.
  - underflow strictly increasing.
- return 0; else -EINVAL "unsorted underflow" / "duplicate underflow" / "unsorted entry" / "duplicate entry".

REQ-13: xt_check_entry_offsets(base, elems, target_offset, next_offset):
- target_offset within [sizeof(entry), next_offset − sizeof(xt_entry_target)].
- next_offset > target_offset.
- target_offset XT_ALIGNed.
- Matches in [elems, base + target_offset) pass xt_check_entry_match.

REQ-14: xt_replace_table(table, num_counters, newinfo, *error):
- xt_jumpstack_alloc(newinfo) → -ENOMEM ⟹ return NULL.
- local_bh_disable.
- private = table.private.
- if num_counters != private.number: local_bh_enable; *error = -EAGAIN; return NULL.
- newinfo.initial_entries = private.initial_entries.
- smp_wmb (publish newinfo contents before pointer).
- table.private = newinfo.
- smp_mb (all CPUs see new private).
- local_bh_enable.
- For each CPU: wait until per-CPU `xt_recseq` is no longer odd (drain in-progress walkers).
- audit_log_nfcfg(REGISTER/REPLACE).
- return private (caller frees old).

REQ-15: xt_register_table(net, input_table, bootstrap, newinfo):
- table = kmemdup(input_table).
- mutex_lock(xt[af].mutex).
- Refuse if same-name table already present in this netns.
- table.private = bootstrap; xt_replace_table(table, 0, newinfo, &ret).
- private.initial_entries = private.number.
- list_add(table, xt_net.tables[af]).
- mutex_unlock.

REQ-16: xt_unregister_table(table):
- mutex_lock(xt[af].mutex).
- private = table.private.
- list_del(table).
- mutex_unlock.
- audit_log_nfcfg(UNREGISTER).
- kfree(table.ops); kfree(table).
- return private (for caller to free).

REQ-17: xt_find_table_lock(net, af, name):
- mutex_lock(xt[af].mutex).
- Walk xt_net.tables[af]: found ∧ try_module_get(t.me) ⟹ return t (mutex held).
- Walk xt_templates[af]: matching name ⟹ try_module_get + drop mutex + tmpl.table_init(net) + reacquire mutex + rewalk to find created table.
- Not found: mutex_unlock; return ERR_PTR(-ENOENT).

REQ-18: xt_request_find_table_lock(net, af, name):
- t = xt_find_table_lock.
- IS_ERR(t) ⟹ request_module("%stable_%s", xt_prefix[af], name); retry.

REQ-19: xt_table_unlock(table):
- mutex_unlock(xt[table.af].mutex).

REQ-20: xt_alloc_table_info(size):
- sz = sizeof(struct xt_table_info) + size.
- sz overflow OR sz ≥ XT_MAX_TABLE_SIZE (512 MiB) ⟹ return NULL.
- info = kvmalloc(sz, GFP_KERNEL_ACCOUNT).
- memset(info, 0, sizeof(*info)); info.size = size.

REQ-21: xt_free_table_info(info):
- For each possible CPU: kvfree(info.jumpstack[cpu]).
- kvfree(info.jumpstack).
- kvfree(info).

REQ-22: xt_jumpstack_alloc(info):
- Per-CPU jumpstack pointer array (kvzalloc / kzalloc by size).
- For each CPU: kvmalloc_node(2 * stacksize * sizeof(void*), GFP_KERNEL, cpu_to_node(cpu)).
- Two stacks per CPU: one for first traversal, one for `-j TEE` re-entry.

REQ-23: xt_register_template(table, table_init):
- Per-AF list head xt_templates[af] of `struct xt_template { list, table_init, me, name }`.
- mutex_lock(xt[af].mutex); refuse duplicate-name; allocate + insert.

REQ-24: xt_hook_ops_alloc(table, fn):
- num_hooks = hweight32(table.valid_hooks).
- Allocate `nf_hook_ops[num_hooks]`.
- For each bit set in valid_hooks: ops[i].hook = fn; pf = af; hooknum; priority.

REQ-25: xt_proto_init(net, af) — per-netns:
- Create `/proc/net/{ip,ip6,arp,eb,x}_tables` seq_file (xt_table_seq_ops).
- Create `/proc/net/*_matches` (xt_match_seq_ops).
- Create `/proc/net/*_targets` (xt_target_seq_ops).
- chown root:root if root uid/gid valid.

REQ-26: xt_action_param (per-skb evaluator ctx, in `<linux/netfilter/x_tables.h>`):
- match / target (kernel-owned struct xt_match / xt_target).
- matchinfo / targinfo (per-rule data).
- thoff: transport header offset.
- fragoff: fragment offset.
- hotdrop: bool out-param (set ⟹ drop skb, even if no NF_DROP returned).
- family: NFPROTO_*.
- state: `struct nf_hook_state *` (in, out, hook, net, ...).

REQ-27: xt_mtchk_param (per-match checkentry ctx):
- net, table (name), entryinfo (AF-specific entry: ipt_ip / ip6t_ip6 / ebt_entry / arpt_arp), match, matchinfo, nft_compat, hook_mask, family.

REQ-28: xt_tgchk_param (per-target checkentry ctx):
- Same shape as xt_mtchk_param but for `struct xt_target` + `targinfo`.

REQ-29: xt_compat_add_offset(af, offset, delta) — CONFIG_NETFILTER_XTABLES_COMPAT:
- Append (offset, delta) to xt[af].compat_tab[xt[af].cur++].
- compat_tab pre-sized via xt_compat_init_offsets.

REQ-30: xt_compat_calc_jump(af, offset):
- Binary-search compat_tab for offset; return cumulative delta to translate user-rule jump → kernel-rule jump.

REQ-31: xt_compat_match_from_user / xt_compat_target_from_user:
- Per-extension `compat_from_user` callback expands 32-bit struct → 64-bit struct at dstptr.
- Pads to XT_ALIGN.

REQ-32: xt_compat_match_to_user / xt_compat_target_to_user:
- Inverse: 64-bit → 32-bit for getsockopt.

REQ-33: xt_compat_lock(af) / xt_compat_unlock(af):
- Per-AF mutex `xt[af].compat_mutex` independent of `xt[af].mutex`.

REQ-34: xt_counters_alloc(counters):
- counters == 0 OR counters > INT_MAX / sizeof(*mem) ⟹ NULL.
- counters * sizeof(*mem) > XT_MAX_TABLE_SIZE ⟹ NULL.
- return vzalloc(counters * sizeof(struct xt_counters)).

REQ-35: xt_percpu_counter_alloc(state, counter):
- nr_cpu_ids ≤ 1 ⟹ return true (no percpu).
- If !state.mem: __alloc_percpu(XT_PCPU_BLOCK_SIZE=4096, XT_PCPU_BLOCK_SIZE).
- counter.pcnt = (state.mem + state.off); state.off += sizeof(*counter).
- If state.off near end of block: state.mem = NULL; off = 0.

REQ-36: xt_percpu_counter_free(counters):
- pcnt = counters.pcnt.
- If nr_cpu_ids > 1 ∧ pcnt is XT_PCPU_BLOCK_SIZE-aligned (block base): free_percpu.

REQ-37: xt_recseq (DEFINE_PER_CPU(seqcount_t)):
- Per-CPU seqcount taken by rule walkers (ipt_do_table etc.).
- xt_replace_table waits per-CPU until seqcount even ⟹ no walker in old blob.

REQ-38: xt_net_init / _exit:
- For each NFPROTO: INIT_LIST_HEAD(xt_net.tables[i]).
- exit: WARN_ON_ONCE if any table still present.

REQ-39: xt_init / _fini:
- Per-CPU seqcount_init(xt_recseq) (CONFIG_NETFILTER_XTABLES_LEGACY).
- xt = kzalloc_objs(struct xt_af, NFPROTO_NUMPROTO).
- For each af: mutex_init, compat_mutex_init (CONFIG_NETFILTER_XTABLES_COMPAT), INIT_LIST_HEAD(target/match/templates).
- register_pernet_subsys(xt_net_ops).

REQ-40: XT_PCPU_BLOCK_SIZE = 4096.

REQ-41: XT_MAX_TABLE_SIZE = 512 MiB.

REQ-42: xt_prefix[NFPROTO_*] = "x" / "ip" / "arp" / "eb" / "ip6" (used for `request_module` and `/proc` name).

## Acceptance Criteria

- [ ] AC-1: `xt_register_target` / `_match` adds to per-AF list under `xt[af].mutex`; lookups see the new entry.
- [ ] AC-2: `xt_request_find_match` falls back to `request_module("xt_<name>")` (or `ipt_<name>`, `ip6t_<name>` etc.) on first miss and retries.
- [ ] AC-3: `xt_check_match` rejects mismatched matchsize (≠ XT_ALIGN(matchsize) and matchsize != -1).
- [ ] AC-4: `xt_check_match` rejects per-NFPROTO_ARP match used by non-ARP table.
- [ ] AC-5: `xt_check_match` rejects hook_mask ⊄ match.hooks.
- [ ] AC-6: `xt_check_table_hooks` rejects unsorted / duplicate entry offsets or underflow offsets.
- [ ] AC-7: `xt_check_entry_match` rejects misaligned or oversized `xt_entry_match` chain.
- [ ] AC-8: `xt_alloc_table_info` rejects size ≥ 512 MiB or overflow.
- [ ] AC-9: `xt_replace_table` retries `-EAGAIN` if `num_counters` no longer matches.
- [ ] AC-10: `xt_replace_table` waits for per-CPU `xt_recseq` to drain (no walker in old blob) before returning old.
- [ ] AC-11: `xt_register_table` refuses duplicate name within the same netns.
- [ ] AC-12: `xt_find_table_lock` autoload via per-template `table_init(net)` succeeds, then re-finds the created table.
- [ ] AC-13: `xt_request_find_table_lock` invokes `request_module("iptable_<name>")` etc. on first miss.
- [ ] AC-14: Per-CPU jumpstacks sized `stacksize * 2` (TEE-re-entry safe).
- [ ] AC-15: `xt_compat_calc_jump(af, offset)` binary-search returns correct cumulative delta.
- [ ] AC-16: `xt_percpu_counter_alloc` allocates from 4 KiB blocks; matching `_free` releases the whole block on its base.
- [ ] AC-17: `xt_proto_init` creates per-af `/proc/net/{ip,ip6,arp,eb,x}_{tables,matches,targets}` chmod 0440 root:root.
- [ ] AC-18: `xt_unregister_table` issues audit_log_nfcfg(UNREGISTER) and frees `ops`.

## Architecture

```
struct XtTable {
  list: ListLink,
  valid_hooks: u32,
  private: *XtTableInfo,            // RCU-style swap
  ops: *NfHookOps,
  table_init: Option<fn(*Net) -> Result>,
  name: [u8; XT_TABLE_MAXNAMELEN],
  af: u8,                           // NFPROTO_*
  priority: i32,
  me: *Module,
}

struct XtTableInfo {
  size: u32,
  number: u32,
  initial_entries: u32,
  hook_entry: [u32; NF_INET_NUMHOOKS],
  underflow: [u32; NF_INET_NUMHOOKS],
  stacksize: u32,
  jumpstack: *[*void; nr_cpu_ids],  // per-cpu, 2*stacksize each
  entries: [u8; size],
}

struct XtAf {
  mutex: Mutex,
  match: ListHead,
  target: ListHead,
  /* CONFIG_NETFILTER_XTABLES_COMPAT */
  compat_mutex: Mutex,
  compat_tab: *CompatDelta,
  number: u32,
  cur: u32,
}

struct XtPernet {
  tables: [ListHead; NFPROTO_NUMPROTO],
}

struct XtTemplate {
  list: ListLink,
  table_init: fn(*Net) -> Result,
  me: *Module,
  name: [u8; XT_TABLE_MAXNAMELEN],
}
```

`Xtables::register_target(target) -> Result`:
1. af = target.family.
2. mutex_lock(xt[af].mutex).
3. list_add(&target.list, &xt[af].target).
4. mutex_unlock.
5. return Ok.

`Xtables::find_match(af, name, revision) -> Result<*XtMatch, Errno>`:
1. if strnlen(name) == XT_EXTENSION_MAXNAMELEN: return Err(EINVAL).
2. err = ENOENT.
3. mutex_lock(xt[af].mutex).
4. foreach m in xt[af].match:
   - if strcmp(m.name, name) != 0: continue.
   - if m.revision == revision:
     - if try_module_get(m.me): mutex_unlock; return Ok(m).
   - else: err = EPROTOTYPE.
5. mutex_unlock.
6. if af != NFPROTO_UNSPEC: return Xtables::find_match(NFPROTO_UNSPEC, name, revision).
7. return Err(err).

`Xtables::request_find_match(nfproto, name, revision) -> Result<*XtMatch, Errno>`:
1. m = find_match(nfproto, name, revision).
2. if Err(_): request_module("%st_%s", xt_prefix[nfproto], name); m = find_match(...).
3. return m.

`Xtables::check_match(par, size, proto, inv_proto) -> Result`:
1. xt_check_match_common(par, size, proto, inv_proto)? — verify matchsize alignment, table restriction, NFPROTO_ARP-only rule, hook_mask ⊆ match.hooks, proto compatibility.
2. xt_check_hooks_match(par)? — invoke optional callback.
3. xt_checkentry_match(par) — invoke optional callback; > 0 ⟹ EIO.

`Xtables::check_entry_match(match, target, alignment) -> Result`:
1. length = target − match.
2. if length == 0: return Ok.
3. pos = match as *XtEntryMatch.
4. loop:
   - if (pos as usize) % alignment != 0: return Err(EINVAL).
   - if length < sizeof(XtEntryMatch): return Err(EINVAL).
   - if pos.match_size < sizeof(XtEntryMatch): return Err(EINVAL).
   - if pos.match_size > length: return Err(EINVAL).
   - length -= pos.match_size; pos = (pos + pos.match_size).
   - if length == 0: return Ok.

`Xtables::check_table_hooks(info, valid_hooks) -> Result`:
1. max_entry = 0; max_uflow = 0; check_hooks = false.
2. for i in 0..NF_INET_NUMHOOKS:
   - if !(valid_hooks & (1 << i)): continue.
   - if hook_entry[i] == 0xFFFFFFFF OR underflow[i] == 0xFFFFFFFF: return Err(EINVAL).
   - if check_hooks:
     - if max_uflow > underflow[i]: return Err("unsorted underflow").
     - if max_uflow == underflow[i]: return Err("duplicate underflow").
     - if max_entry > hook_entry[i]: return Err("unsorted entry").
     - if max_entry == hook_entry[i]: return Err("duplicate entry").
   - max_entry = hook_entry[i]; max_uflow = underflow[i]; check_hooks = true.
3. return Ok.

`Xtables::replace_table(table, num_counters, newinfo) -> Result<*XtTableInfo, Errno>`:
1. xt_jumpstack_alloc(newinfo)?.
2. local_bh_disable.
3. private = table.private.
4. if num_counters != private.number: local_bh_enable; return Err(EAGAIN).
5. newinfo.initial_entries = private.initial_entries.
6. smp_wmb.
7. table.private = newinfo.
8. smp_mb.
9. local_bh_enable.
10. for each possible CPU:
    - s = &per_cpu(xt_recseq, cpu).
    - seq = raw_read_seqcount(s).
    - if seq & 1: spin { cond_resched(); cpu_relax(); } while (raw_read_seqcount(s) == seq).
11. audit_log_nfcfg(table.name, table.af, private.number, REGISTER-or-REPLACE, GFP_KERNEL).
12. return Ok(private).

`Xtables::register_table(net, input_table, bootstrap, newinfo) -> Result<*XtTable, Errno>`:
1. table = kmemdup(input_table).
2. mutex_lock(xt[af].mutex).
3. foreach t in xt_net.tables[af]:
   - if strcmp(t.name, table.name) == 0: mutex_unlock; kfree(table); return Err(EEXIST).
4. table.private = bootstrap.
5. xt_replace_table(table, 0, newinfo, &ret)? else mutex_unlock + kfree(table) + return Err.
6. private = table.private.
7. private.initial_entries = private.number.
8. list_add(table.list, xt_net.tables[af]).
9. mutex_unlock.
10. return Ok(table).

`Xtables::find_table_lock(net, af, name) -> Result<*XtTable, Errno>`:
1. mutex_lock(xt[af].mutex).
2. foreach t in xt_net.tables[af]:
   - if strcmp(t.name, name) == 0 ∧ try_module_get(t.me): return Ok(t) (mutex still held).
3. foreach tmpl in xt_templates[af]:
   - if strcmp(tmpl.name, name) == 0:
     - if !try_module_get(tmpl.me): mutex_unlock; return Err(ENOENT).
     - owner = tmpl.me.
     - mutex_unlock.
     - err = tmpl.table_init(net).
     - if err: module_put(owner); return Err(err).
     - mutex_lock(xt[af].mutex).
     - break.
4. foreach t in xt_net.tables[af]:
   - if strcmp(t.name, name) == 0 ∧ owner == t.me: return Ok(t).
5. module_put(owner); mutex_unlock; return Err(ENOENT).

`Xtables::request_find_table_lock(net, af, name) -> Result<*XtTable, Errno>`:
1. t = find_table_lock(net, af, name).
2. if Err(_) ∧ CONFIG_MODULES:
   - request_module("%stable_%s", xt_prefix[af], name)?.
   - t = find_table_lock(net, af, name).
3. return t.

`XtTableInfo::alloc(size) -> Option<*XtTableInfo>`:
1. sz = sizeof(*info) + size.
2. if sz < sizeof(*info) OR sz ≥ XT_MAX_TABLE_SIZE: return None.
3. info = kvmalloc(sz, GFP_KERNEL_ACCOUNT)?.
4. memset(info, 0, sizeof(*info)).
5. info.size = size.
6. return Some(info).

`XtCounters::pcpu_alloc(state, counter) -> bool`:
1. if nr_cpu_ids ≤ 1: return true.
2. if state.mem.is_null():
   - state.mem = __alloc_percpu(XT_PCPU_BLOCK_SIZE=4096, XT_PCPU_BLOCK_SIZE)?.
3. counter.pcnt = (state.mem + state.off) as ulong.
4. state.off += sizeof(*counter).
5. if state.off > XT_PCPU_BLOCK_SIZE − sizeof(*counter):
   - state.mem = NULL; state.off = 0.
6. return true.

`XtCompat::calc_jump(af, offset) -> i32`:
1. Binary-search xt[af].compat_tab[..xt[af].cur] for the greatest offset ≤ given.
2. Return cumulative delta from match.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `register_under_xt_af_mutex` | INVARIANT | per-register_match/target: list_add only with xt[af].mutex held. |
| `find_match_module_get_or_err` | INVARIANT | per-find_match: returns either valid *XtMatch with refcount or ERR_PTR. |
| `replace_table_seqcount_drain` | INVARIANT | per-replace_table: returns only after every per-CPU xt_recseq is even or has advanced. |
| `replace_table_size_bound` | INVARIANT | per-alloc_table_info: size < XT_MAX_TABLE_SIZE = 512 MiB. |
| `entry_match_contiguous` | INVARIANT | per-check_entry_match: sum of match_size == target − match. |
| `table_hooks_sorted_unique` | INVARIANT | per-check_table_hooks: hook_entry and underflow both strictly increasing across valid hooks. |
| `jumpstack_pair_per_cpu` | INVARIANT | per-jumpstack_alloc: 2*stacksize entries per CPU (TEE-safe). |
| `pcpu_counter_block_size_4096` | INVARIANT | per-percpu_counter_alloc: BUILD_BUG_ON(XT_PCPU_BLOCK_SIZE < 2*sizeof(counter)). |
| `register_table_unique_name_per_netns` | INVARIANT | per-register_table: -EEXIST on duplicate name in same xt_net.tables[af]. |
| `compat_offset_sorted` | INVARIANT | per-compat_add_offset: appended offsets non-decreasing for binary search. |

### Layer 2: TLA+

`net/netfilter/x-tables.tla`:
- States: per-AF match/target lists, per-net per-AF table list, per-CPU xt_recseq even/odd, per-table private pointer, per-template list.
- Operations: register/unregister match/target, register/unregister table, replace_table, find_table_lock with template fallback, compat_add_offset.
- Properties:
  - `safety_register_match_under_mutex` — per-list_add: xt[af].mutex held.
  - `safety_table_unique_name_per_netns` — per-net: at most one xt_table with a given name in xt_net.tables[af].
  - `safety_replace_atomic` — per-replace_table: rule walkers observe either old or new private, never partial.
  - `safety_old_blob_drained_before_free` — per-replace_table: per-CPU xt_recseq drained before caller frees old.
  - `safety_compat_table_sorted` — per-compat_tab: monotone offsets ⟹ binary search correct.
  - `liveness_find_table_lock_or_enoent` — per-call: terminates with table or ENOENT.
  - `liveness_request_find_table_autoload_eventually` — per-call: after request_module, retry sees the table.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `register_target/match` post: target/match in xt[af].list | `Xtables::register_target` / `_match` |
| `find_match` post: Ok ⟹ try_module_get succeeded | `Xtables::find_match` |
| `check_match` post: all sub-checks Ok ⟹ par.match safe to dispatch | `Xtables::check_match` |
| `check_entry_match` post: matches contiguous + aligned | `Xtables::check_entry_match` |
| `check_table_hooks` post: hook_entry / underflow strictly sorted | `Xtables::check_table_hooks` |
| `alloc_table_info` post: size < XT_MAX_TABLE_SIZE ∧ info zero-initialized | `XtTableInfo::alloc` |
| `replace_table` post: table.private == newinfo ∧ recseq drained | `Xtables::replace_table` |
| `register_table` post: unique name per netns | `Xtables::register_table` |
| `find_table_lock` post: Ok ⟹ try_module_get on owning module | `Xtables::find_table_lock` |
| `percpu_counter_alloc` post: counter.pcnt within current 4 KiB block | `XtCounters::pcpu_alloc` |

### Layer 4: Verus/Creusot functional

`Per-rule-blob lifecycle → register_table(bootstrap) → replace_table(new) waits xt_recseq → private swapped under smp_wmb + smp_mb → audit_log_nfcfg → eventual unregister_table` semantic equivalence: per-`Documentation/networking/netfilter-sysctl.rst` and `iptables-restore(8)` atomic-load expectations. Per-CONFIG_NETFILTER_XTABLES_COMPAT: 32↔64 expansion via xt_compat_calc_jump and per-extension compat_from_user / compat_to_user.

## Hardening

(Inherits row-1 features from `net/netfilter/00-overview.md` § Hardening.)

x_tables reinforcement:

- **Per-name strnlen against XT_EXTENSION_MAXNAMELEN** — defense against per-unterminated-string copy_to_user / module-name OOB.
- **Per-xt[af].mutex held for every list_add/list_del** — defense against per-concurrent-modification list corruption.
- **Per-try_module_get during find_match/find_target** — defense against per-module-unload UAF of m->me / t->me.
- **Per-NFPROTO_ARP family restriction** — defense against per-ARP-table-with-non-ARP-match panic.
- **Per-hook_mask ⊆ match.hooks** — defense against per-extension-used-from-wrong-hook crash.
- **Per-XT_MAX_TABLE_SIZE (512 MiB) cap** — defense against per-allocation DoS via oversized info blob.
- **Per-check_entry_match contiguous alignment** — defense against per-malformed-blob OOB read in evaluator.
- **Per-check_table_hooks strictly sorted hook_entry/underflow** — defense against per-out-of-order rule blob.
- **Per-replace_table per-CPU seqcount drain** — defense against per-concurrent-walker UAF of old info.
- **Per-replace_table EAGAIN on stale num_counters** — defense against per-lost-update of counter set.
- **Per-jumpstack double-sized for TEE** — defense against per-jumpstack-overflow on `-j TEE` re-entry.
- **Per-percpu counter block aligned + free-only-on-base** — defense against per-double-free of the 4 KiB slab.
- **Per-audit_log_nfcfg on REGISTER/REPLACE/UNREGISTER** — defense against per-silent-ruleset-tamper.
- **Per-netns unique table name** — defense against per-collision-overwrite.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Per-AF `ipt_do_table` / `ip6t_do_table` / `arpt_do_table` / `ebt_do_table` evaluators (covered by `iptables.md`, `arptables.md`, `ebtables.md` Tier-3 siblings)
- `xt_NFQUEUE`, `xt_TEE`, `xt_CT`, `xt_conntrack`, `xt_state`, `xt_recent`, `xt_hashlimit`, `xt_set`, `xt_TPROXY` and other per-extension modules (covered separately under `net/netfilter/xt-extensions/` if expanded)
- nf_tables (nft) backend (covered in `nf-tables.md`, `nft-core.md`, `nft-sets.md`)
- nf_conntrack core (covered in `nf-conntrack-core.md`)
- iptables-nft translation (userspace)
- Implementation code
