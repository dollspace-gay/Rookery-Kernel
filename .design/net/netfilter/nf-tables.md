# Tier-3: net/netfilter/nf_tables_api.c — nftables core

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: net/netfilter/00-overview.md
upstream-paths:
  - net/netfilter/nf_tables_api.c (~12266 lines)
  - net/netfilter/nf_tables_core.c (per-skb nft_do_chain)
  - include/net/netfilter/nf_tables.h
  - include/uapi/linux/netfilter/nf_tables.h (NFT_MSG_*, NFTA_*)
  - include/uapi/linux/netfilter/nfnetlink.h
-->

## Summary

nftables is the modern packet-classification engine that replaces iptables / ip6tables / ebtables / arptables. The control plane is a netlink subsystem (`NFNL_SUBSYS_NFTABLES`) carrying batched `NFT_MSG_*` transactions wrapped in `NFNL_MSG_BATCH_BEGIN`/`NFNL_MSG_BATCH_END`. Each batch parses into a `struct nft_trans` per object change, is appended to `nftables_pernet.commit_list`, and either commits atomically by bumping `net->nft.gencursor` (publishing the prepared `rules_gen_X` blob via RCU) or is rolled back via `__nf_tables_abort`. Per-table contains chains (lists of rules + a per-generation rule blob), sets (`rhash` / `rbtree` / `bitmap` / `pipapo` backends, selected by `nft_select_set_ops` using a `nft_set_estimate` cost model), flow tables, and stateful objects. Each rule is a tightly packed array of `struct nft_expr` (register-VM instructions). `nft_do_chain()` walks the current-generation rule blob, dispatches `expr->ops->eval` per instruction, and resolves verdicts (`NFT_CONTINUE / BREAK / JUMP / GOTO / RETURN` plus terminal `NF_ACCEPT / DROP / QUEUE / STOLEN`). Critical for: surviving every modern container, k8s, firewalld and systemd-networkd workload that stopped using iptables.

This Tier-3 covers `net/netfilter/nf_tables_api.c` (~12266 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct nft_table` | per-table root (chains/sets/objects/flowtables) | `NftTable` |
| `struct nft_chain` | per-chain (rules + per-gen blob) | `NftChain` |
| `struct nft_base_chain` | per-base-chain (hook ops, policy, stats) | `NftBaseChain` |
| `struct nft_rule` | per-rule (handle + expression bytestream) | `NftRule` |
| `struct nft_expr` | per-expression (ops + private data) | `NftExpr` |
| `struct nft_expr_type` | per-expression-type (registered via `nft_register_expr`) | `NftExprType` |
| `struct nft_set` | per-set (storage + ops dispatch table) | `NftSet` |
| `struct nft_set_ops` | per-set-backend vtable (`lookup`/`insert`/`activate`/`walk`/…) | `NftSetOps` |
| `struct nft_set_type` | per-set-type registration | `NftSetType` |
| `struct nft_object` | per-stateful-object (counter, quota, ct-helper, …) | `NftObject` |
| `struct nft_flowtable` | per-flowtable (offload candidate flows) | `NftFlowtable` |
| `struct nft_ctx` | per-transaction parse context | `NftCtx` |
| `struct nft_trans` | per-object transaction entry | `NftTrans` |
| `nft_request_module()` | per-autoload of expr/set/object type | `Nft::request_module` |
| `nft_register_expr()` / `nft_unregister_expr()` | per-expression-type registration | `Nft::register_expr` / `unregister_expr` |
| `nft_register_obj()` / `nft_unregister_obj()` | per-object-type registration | `Nft::register_obj` / `unregister_obj` |
| `nft_register_chain_type()` / `_unregister_` | per-chain-type registration (filter/nat/route) | `Nft::register_chain_type` |
| `nft_register_flowtable_type()` / `_unregister_` | per-flowtable-type registration | `Nft::register_flowtable_type` |
| `nf_tables_newtable()` / `nf_tables_deltable()` / `nf_tables_gettable()` | per-`NFT_MSG_{NEW,DEL,GET}TABLE` handler | `Nft::newtable` / `deltable` / `gettable` |
| `nf_tables_newchain()` / `nf_tables_delchain()` / `nf_tables_getchain()` | per-`NFT_MSG_*CHAIN` handler | `Nft::newchain` / `delchain` / `getchain` |
| `nf_tables_newrule()` / `nf_tables_delrule()` / `nf_tables_getrule()` | per-`NFT_MSG_*RULE` handler | `Nft::newrule` / `delrule` / `getrule` |
| `nf_tables_newset()` / `nf_tables_delset()` / `nf_tables_getset()` | per-`NFT_MSG_*SET` handler | `Nft::newset` / `delset` / `getset` |
| `nf_tables_newsetelem()` / `nf_tables_delsetelem()` / `nf_tables_getsetelem()` | per-`NFT_MSG_*SETELEM` handler | `Nft::newsetelem` / `delsetelem` / `getsetelem` |
| `nf_tables_newobj()` / `nf_tables_delobj()` / `nf_tables_getobj()` | per-`NFT_MSG_*OBJ` handler | `Nft::newobj` / `delobj` / `getobj` |
| `nf_tables_newflowtable()` / `nf_tables_delflowtable()` | per-`NFT_MSG_*FLOWTABLE` handler | `Nft::newflowtable` / `delflowtable` |
| `nf_tables_getgen()` | per-`NFT_MSG_GETGEN` (read `gencursor`) | `Nft::getgen` |
| `nf_tables_commit()` | per-batch commit (publish `rules_gen_X`, bump `base_seq`) | `Nft::commit` |
| `__nf_tables_abort()` / `nf_tables_abort()` | per-batch rollback | `Nft::abort` |
| `nf_tables_commit_chain_prepare()` | per-chain blob-alloc for next gen | `Nft::commit_chain_prepare` |
| `nf_tables_commit_chain()` | per-chain RCU swap to next gen blob | `Nft::commit_chain` |
| `nft_chain_validate()` / `nft_table_validate()` | per-validation graph walk (jumps + sets) | `Nft::chain_validate` |
| `nft_set_lookup_global()` / `nft_set_lookup()` / `nft_set_lookup_byid()` | per-set name/handle/id resolve | `NftSet::lookup_*` |
| `nft_select_set_ops()` | per-backend selection (size/lookup cost) | `Nft::select_set_ops` |
| `nft_set_elem_init()` / `_destroy()` / `_expr_alloc()` | per-element setup | `NftSet::elem_init` / `destroy` |
| `nf_tables_bind_set()` / `nf_tables_unbind_set()` | per-binding to chain (anonymous set lifecycle) | `Nft::bind_set` / `unbind_set` |
| `nf_tables_bind_chain()` / `nf_tables_unbind_chain()` | per-jumpvmap binding | `Nft::bind_chain` / `unbind_chain` |
| `nft_trans_alloc()` / `nft_trans_destroy()` | per-trans alloc | `NftTrans::alloc` / `destroy` |
| `nft_trans_commit_list_add_tail()` / `_add_elem()` | per-trans enqueue on commit list | `NftTrans::commit_list_add` |
| `nft_trans_gc_alloc()` / `_queue_async()` / `_queue_sync()` | per-set async GC of expired/dead elements | `Nft::trans_gc_*` |
| `nft_do_chain()` | per-skb VM driver (chain walk + expr dispatch) | `Nft::do_chain` |
| `nft_data_init()` / `_release()` / `_dump()` | per-data-blob marshalling (value/verdict) | `Nft::data_*` |
| `nft_parse_register_load()` / `_store()` | per-register vreg → physreg translation | `Nft::parse_register_load` / `store` |
| `nf_tables_register_hook()` / `_unregister_hook()` | per-base-chain hook attach to nf_hook chains | `Nft::register_hook` |
| `nf_tables_table_destroy()` / `_chain_destroy()` / `_rule_release()` | per-object tear-down | `Nft::table_destroy` / `chain_destroy` / `rule_release` |
| `__nft_release_tables()` | per-netns teardown | `Nft::release_tables` |
| `nf_tables_subsys` (`nfnetlink_subsystem`) | per-NFNL subsys registration | `Nft::SUBSYS` |
| `nf_tables_cb[NFT_MSG_MAX]` | per-message dispatch table | `Nft::CB` |
| `nf_tables_valid_genid()` | per-`NFT_MSG_BEGIN` `NFNL_BATCH_GENID` check | `Nft::valid_genid` |

## Compatibility contract

REQ-1: struct nft_table:
- list: per-netns table list link.
- chains_ht: per-name rhashtable.
- chains: per-stable-walk list.
- sets: per-set list.
- objects: per-stateful-object list.
- flowtables: per-flowtable list.
- hgenerator: per-handle generator.
- handle: per-table handle (kernel-assigned u64).
- use: per-chain-ref count.
- family: per-AF (NFPROTO_IPV4 / IPV6 / INET / ARP / BRIDGE / NETDEV / UNSPEC).
- flags: per-NFT_TABLE_F_* (DORMANT / OWNER / PERSIST).
- genmask: per-pending-{old,new} bit pair.
- nlpid: per-NFT_TABLE_F_OWNER owner portid.
- name: NUL-term (≤ NFT_TABLE_MAXNAMELEN).
- validate_state: NFT_VALIDATE_{SKIP,NEED,DO}.

REQ-2: struct nft_chain:
- blob_gen_0 / blob_gen_1: per-rule-blob double-buffer (RCU); active selected by `net->nft.gencursor`.
- rules: per-chain rule list (control-plane).
- list: per-table chain-list link.
- rhlhead: per-name rhlist node in chains_ht.
- table: per-back-pointer.
- handle: per-chain handle.
- use: per-jump-ref count.
- flags: per-NFT_CHAIN_F_* (BASE / BINDING / HW_OFFLOAD).
- bound: per-binding marker.
- name: NUL-term.

REQ-3: struct nft_base_chain (NFT_CHAIN_BASE only):
- ops: per-`nf_hook_ops` (hooks netfilter at hook + priority).
- hook_list: per-NETDEV family per-device hook list.
- type: per-`nft_chain_type` (filter / nat / route).
- policy: per-default-verdict (NF_ACCEPT / NF_DROP).
- flags: per-base flags.
- stats: per-CPU `nft_stats` (bytes/packets).
- chain: embedded base chain.
- flow_block: per-hardware-offload block.

REQ-4: struct nft_rule:
- list: per-chain control-plane list.
- handle: per-rule 42-bit handle.
- genmask: per-{old,new} 2-bit.
- dlen: per-expression-bytestream length (12-bit).
- udata: per-user-data appended flag.
- data: ALIGN(struct nft_expr) bytestream — sequence of `struct nft_expr { ops, data[] }`.

REQ-5: struct nft_expr:
- ops: per-`nft_expr_ops` (eval / init / destroy / dump / validate / clone / size / type).
- data: per-expression private storage (alignment u64).
- Iteration helpers: `nft_expr_first(rule)`, `nft_expr_next(expr) = (u8*)expr + expr->ops->size`, `nft_expr_last(rule) = &rule->data[dlen]`.

REQ-6: struct nft_set:
- table: per-back-pointer.
- name: NUL-term.
- handle: per-set handle.
- ktype: per-key data-type id (`NFT_DATA_VALUE` / `NFT_DATA_VERDICT`).
- dtype: per-value data-type id (maps).
- objtype: per-object-data type id (object-maps).
- size: per-max-element budget.
- field_len[NFT_REG32_COUNT]: per-concat-field byte widths.
- field_count: per-concat-field count (≥ 1; > 1 = concat key).
- bindings: per-binding list (rules referencing this set).
- nelems: per-current-size atomic.
- timeout: per-element default timeout (jiffies).
- gc_int: per-GC interval (msecs).
- policy: NFT_SET_POL_{PERFORMANCE,MEMORY}.
- flags: per-NFT_SET_F_* (ANONYMOUS / CONSTANT / INTERVAL / MAP / TIMEOUT / EVAL / OBJECT / CONCAT).
- klen: per-key bytes.
- dlen: per-data bytes (maps).
- exprs[NFT_SET_EXPR_MAX]: per-stateful-per-elem expressions.
- catchall_list: per-catch-all element list.
- ops: per-backend vtable.
- data: per-backend private storage.

REQ-7: struct nft_set_ops:
- Hot path (RCU-read): `lookup(net, set, key) -> nft_set_ext*`; `update(set, key, expr, regs)`; `delete(set, key) -> bool`.
- Control plane: `insert`, `activate`, `deactivate`, `flush`, `remove`, `walk`, `get`, `init`, `destroy`, `gc_init`, `commit`, `abort`.
- Cost model: `estimate(desc, features, est) -> bool` populates `nft_set_estimate { size, lookup, space }`.
- Capacity: `ksize(usize)`, `usize(ksize)`, `adjust_maxsize(set)`, `privsize(nla, desc)`.

REQ-8: Set backends (`nft_set_types[]` order = preference within tier):
- `nft_set_hash_fast_type` — fixed-size hashtable, small keys.
- `nft_set_hash_type` — general hashtable.
- `nft_set_rhash_type` — `rhashtable`-backed (resizes).
- `nft_set_bitmap_type` — 1-byte key, dense ranges.
- `nft_set_rbtree_type` — interval key (`NFT_SET_INTERVAL`).
- `nft_set_pipapo_avx2_type` (x86_64) / `nft_set_pipapo_type` — concat key (`NFT_SET_CONCAT`).
- `nft_select_set_ops(ctx, flags, desc)`: walk types matching `(flags & type->features) == (flags & NFT_SET_FEATURES)` (NFT_SET_FEATURES = INTERVAL | MAP | TIMEOUT | OBJECT | EVAL), call `estimate`, pick by policy (PERFORMANCE = min lookup→space; MEMORY = min space→lookup).

REQ-9: Chain types (`chain_type[NFPROTO_NUMPROTO][NFT_CHAIN_T_MAX]`):
- NFT_CHAIN_T_DEFAULT (filter): registered for every family; `hook_mask` ⊇ all NF_INET_* hooks; `hooks[]` invoke `nft_do_chain`.
- NFT_CHAIN_T_NAT (nat): registered when NF_NAT enabled; `hook_mask` ⊆ PREROUTING | POSTROUTING | INPUT | OUTPUT; `ops_register/unregister` integrate with NAT subsystem.
- NFT_CHAIN_T_ROUTE (route): per-`nf_hook_state` reroute-on-change; `hook_mask` ⊆ OUTPUT (IPv4/IPv6).
- `nft_register_chain_type(ctype)` slots into `chain_type[ctype->family][ctype->type]`.
- `nft_chain_validate_dependency(chain, type)` rejects mixing.
- `nft_chain_validate_hooks(chain, hook_flags)` rejects out-of-mask hook num.

REQ-10: Expression registration:
- `nft_register_expr(type)` appends to `nf_tables_expressions` under `nf_tables_destroy_list_lock`.
- `nft_expr_type_get(net, family, name)` resolves by name + family (with `nft_expr_type_request_module` autoload via `nfnl-expr-<family>-<name>`).
- Per-expr `nft_expr_ops { eval, init, destroy, dump, validate, clone, size, type }`.

REQ-11: Object registration:
- `nft_register_obj(obj_type)` appends to `nf_tables_objects`.
- Per-`nft_object_type { ops, type (NFT_OBJECT_*), maxattr, family, policy, init, destroy, dump }`.
- Lookup: `nft_obj_lookup(net, table, nla, objtype, genmask)` → rhltable `nft_objname_ht` keyed by (name, table).

REQ-12: Flowtable type registration:
- `nft_register_flowtable_type(type)` appends to `nf_tables_flowtables`.
- Per-`nf_flowtable_type { family, init, setup, action, free, type, owner }`.

REQ-13: Transaction container (`struct nft_trans`):
- msg_type: NFT_MSG_*.
- table: per-table back-pointer.
- list: per-`commit_list` link.
- binding_obj: optional binding flag.
- One subtype per object: `nft_trans_table`, `nft_trans_chain`, `nft_trans_rule`, `nft_trans_set`, `nft_trans_elem`, `nft_trans_obj`, `nft_trans_flowtable`, `nft_trans_hook`.
- Each carries `new_*` + `old_*` snapshots sufficient for commit + abort.

REQ-14: Per-batch flow (NFNL semantics):
- Userspace sends `NFNL_MSG_BATCH_BEGIN` (with `nfgenmsg.res_id = NFNL_SUBSYS_NFTABLES`, `nfgenmsg.version`), then a series of `NFT_MSG_*`, then `NFNL_MSG_BATCH_END`.
- Begin: `nf_tables_valid_genid(net, genid)` rejects if userspace's stale-genid (≠ `net->nft.base_seq`).
- Each `NFT_MSG_*` handler runs under `nft_net->commit_mutex`, parses NLA into per-trans, links onto `commit_list`.
- End: `nf_tables_commit(net, skb)` (success) or `nf_tables_abort(net, skb, action)` (failure).

REQ-15: `nf_tables_commit(net, skb)`:
- 0. Reject unbound anonymous sets / chains in `binding_list`.
- 0a. `nf_tables_validate(net)` walks all tables with `validate_state == NFT_VALIDATE_DO` and runs `nft_chain_validate` on every chain (jumps + register data-type compat + set element validation).
- 0b. `nft_flow_rule_offload_commit(net)` — push to NIC drivers.
- 1. For every trans: `nf_tables_commit_audit_alloc(&adl, table)`; for NEWRULE/DELRULE: `nf_tables_commit_chain_prepare(net, chain)` allocates `chain->blob_next` from the per-chain control-plane rule list.
- 2. For every chain in every table: `nf_tables_commit_chain(net, chain)` atomically swaps `blob_gen_(gencursor^1)` to `blob_next` and queues old via call_rcu.
- 3. Bump `net->nft.base_seq` (skipping 0); `smp_store_release` pairs with `nft_lookup_eval`'s `smp_load_acquire`.
- 4. `gc_seq = nft_gc_seq_begin(nft_net)`; `net->nft.gencursor = nft_gencursor_next(net)` (flip).
- 5. Walk `commit_list` per msg_type and apply the change to control-plane structures (e.g. for NEWRULE: drop the old rule from `chain->rules`; for DELTABLE: detach from `nft_net->tables`).
- 6. `nft_set_commit_update(set_update_list)` calls per-set `ops->commit`.
- 7. `nf_tables_commit_audit_log` per-table audit message.
- 8. `nf_tables_commit_release(net)` schedules destroy work (`trans_destroy_work`), `nft_gc_seq_end`.

REQ-16: `__nf_tables_abort(net, action)`:
- Walk `commit_list` in reverse, undo per-msg_type changes.
- Per-NEW*: unlink from table / chain / set; release.
- Per-DEL*: re-link.
- Per-set element: `nft_setelem_remove` / `_data_activate` to restore.
- Per-`abort_skip_removal` set-ops: skip remove during abort (e.g. pipapo).
- Per-chain: cancel `blob_next` allocations via `nf_tables_commit_chain_prepare_cancel`.

REQ-17: Generations + RCU publication:
- `net->nft.gencursor` ∈ {0, 1} selects active `blob_gen_X`.
- `net->nft.base_seq` increments per-commit; dumps in flight observe it via `cb->seq` and abort if changed (NFNL_F_DUMP_INTR semantics).
- Per-rule `genmask` 2-bit lifecycle: bit 0 active in gen 0, bit 1 active in gen 1; `nft_gencursor_active(net, genmask)` checks the bit for the current cursor.

REQ-18: `nft_do_chain(pkt, priv) -> verdict`:
- Per-skb VM driver in `net/netfilter/nf_tables_core.c`.
- Load `blob = rcu_dereference(chain->blob_gen_X)` based on `READ_ONCE(net->nft.gencursor)`.
- For each `rule` until `rule->is_last`:
  - For each `expr` in rule: fast paths (`nft_cmp_fast_ops`, `nft_cmp16_fast_ops`, `nft_bitwise_fast_ops`, `nft_payload_fast_ops`); else `expr_call_ops_eval(expr, &regs, pkt)`.
  - On `regs.verdict.code != NFT_CONTINUE`: break expr loop.
  - Per-rule outcome: NFT_BREAK (rule didn't match) → continue rule loop; NFT_CONTINUE → log trace + continue; else exit rule loop.
- Per-jumpstack (NFT_JUMP_STACK_SIZE entries): NFT_JUMP pushes return-rule and switches to target chain; NFT_GOTO replaces without pushing; NFT_RETURN pops.
- Terminal verdicts: NF_ACCEPT / NF_QUEUE / NF_STOLEN / NF_DROP returned directly.
- Empty stack + final NFT_CONTINUE: return base-chain policy.

## Acceptance Criteria

- [ ] AC-1: `NFNL_MSG_BATCH_BEGIN` with stale genid: `-ERESTART` (`nf_tables_valid_genid`).
- [ ] AC-2: `NFT_MSG_NEWTABLE` with same name + family idempotent (replace) or `-EEXIST` if `NLM_F_EXCL`.
- [ ] AC-3: `NFT_MSG_NEWCHAIN` for base chain registers `nf_hook_ops` at requested hook + priority + family.
- [ ] AC-4: `NFT_MSG_NEWRULE` enqueues `nft_trans_rule`; rule visible after commit; not visible after abort.
- [ ] AC-5: `NFT_MSG_NEWSET` selects backend via `nft_select_set_ops` honoring policy and flags.
- [ ] AC-6: `NFT_MSG_NEWSETELEM` element with `NFT_SET_F_TIMEOUT` honors per-set/per-elem timeout; GC removes after expiry.
- [ ] AC-7: `commit` failure (validation) → `commit_list` undone; ruleset unchanged.
- [ ] AC-8: Concurrent dump observes either old or new gen consistently (genmask + base_seq).
- [ ] AC-9: `nft_do_chain` evaluates expressions in order; verdict precedence honored (terminal > GOTO > JUMP > BREAK > CONTINUE).
- [ ] AC-10: Anonymous set unbound at commit: `-EINVAL`.
- [ ] AC-11: Base chain `policy=NF_DROP` and rules all NFT_CONTINUE → return NF_DROP.
- [ ] AC-12: NFT_TABLE_F_OWNER table denies non-owner mutations (`nlpid` check).
- [ ] AC-13: `nft_set_lookup` hot path is lockless (RCU); concurrent insert via `ops->insert` does not block lookup.
- [ ] AC-14: `__nft_release_tables` per-netns exit releases tables → chains → rules → sets → objects → flowtables in dependency order.

## Architecture

```
struct NftTable {
  list: ListNode,
  chains_ht: RhlTable,
  chains: List<NftChain>,
  sets: List<NftSet>,
  objects: List<NftObject>,
  flowtables: List<NftFlowtable>,
  hgenerator: u64,
  handle: u64,
  use: u32,
  family: u8,             // NFPROTO_*
  flags: u8,              // NFT_TABLE_F_*
  genmask: u8,
  nlpid: u32,
  name: CStr<NFT_TABLE_MAXNAMELEN>,
  udlen: u16,
  udata: Option<Box<[u8]>>,
  validate_state: u8,     // NFT_VALIDATE_*
}
```

```
struct NftChain {
  blob_gen_0: Rcu<NftRuleBlob>,
  blob_gen_1: Rcu<NftRuleBlob>,
  rules: List<NftRule>,
  list: ListNode,
  rhlhead: RhlistNode,
  table: *NftTable,
  handle: u64,
  use: u32,
  flags: u8,              // 5-bit NFT_CHAIN_F_*
  bound: bool,            // 1-bit
  genmask: u8,            // 2-bit
  name: CStr,
  blob_next: Option<*NftRuleBlob>,
  vstate: NftChainValidateState,
}
```

```
struct NftBaseChain {
  ops: NfHookOps,
  hook_list: List<NftHook>,
  type: *NftChainType,
  policy: u8,             // NF_ACCEPT | NF_DROP
  flags: u8,
  stats: PerCpu<NftStats>,
  chain: NftChain,
  flow_block: FlowBlock,
}
```

```
struct NftRule {
  list: ListNode,
  handle: u64,            // 42-bit
  genmask: u8,            // 2-bit
  dlen: u16,              // 12-bit
  udata: bool,            // 1-bit
  data: [u8; dlen],       // packed nft_expr stream
}
```

```
struct NftExpr {
  ops: *NftExprOps,
  data: [u8; ops.size - sizeof(NftExpr)],
}
```

```
struct NftSet {
  list: ListNode,
  bindings: List<NftSetBinding>,
  refs: RefCount,
  table: *NftTable,
  net: PossibleNet,
  name: CStr,
  handle: u64,
  ktype: u32,             // NFT_DATA_*
  dtype: u32,
  objtype: u32,
  size: u32,
  field_len: [u8; NFT_REG32_COUNT],
  field_count: u8,
  use: u32,
  nelems: Atomic<u32>,
  ndeact: u32,
  timeout: u64,
  gc_int: u32,
  policy: u16,            // NFT_SET_POL_*
  udlen: u16,
  udata: Option<Box<[u8]>>,
  ops: *NftSetOps,
  flags: u16,             // 13-bit
  dead: bool,             // 1-bit
  genmask: u8,            // 2-bit
  klen: u8,
  dlen: u8,
  num_exprs: u8,
  exprs: [*NftExpr; NFT_SET_EXPR_MAX],
  catchall_list: List<NftSetElemCatchall>,
  data: [u8],             // backend private
}
```

`Nft::commit(net, skb) -> Result<(), i32>`:
1. /* Phase 0a: refuse unbound anonymous sets / chains */
2. for trans_binding in nft_net.binding_list: if NEWSET && anonymous && !bound: return Err(-EINVAL); same for NEWCHAIN binding.
3. /* Phase 0b: chain-graph validation if any trans demanded it */
4. if Nft::tables_validate(net) < 0: nft_net.validate_state = NFT_VALIDATE_DO; return Err(-EAGAIN).
5. /* Phase 0c: HW offload commit */
6. Nft::flow_rule_offload_commit(net)?.
7. /* Phase 1: prepare per-chain next-gen rule blobs + audit slots */
8. for trans in nft_net.commit_list:
   - Nft::commit_audit_alloc(&adl, trans.table)?.
   - if trans.msg_type ∈ {NFT_MSG_NEWRULE, NFT_MSG_DELRULE}: Nft::commit_chain_prepare(net, trans.rule_chain())?.
9. /* Phase 2: publish per-chain blob_gen_X via RCU */
10. for table in nft_net.tables: for chain in table.chains: Nft::commit_chain(net, chain).
11. /* Phase 3: bump base_seq + flip cursor */
12. base_seq = nft_base_seq(net); while ++base_seq == 0 {}.
13. smp_store_release(&net.nft.base_seq, base_seq).
14. gc_seq = nft_gc_seq_begin(nft_net).
15. net.nft.gencursor = nft_gencursor_next(net).
16. /* Phase 4: apply per-trans control-plane mutations */
17. for trans in nft_net.commit_list (FIFO): match trans.msg_type:
    - NFT_MSG_NEWTABLE: clear genmask new-bit; table visible.
    - NFT_MSG_DELTABLE: unlink + flush.
    - NFT_MSG_NEWCHAIN: clear new-bit.
    - NFT_MSG_DELCHAIN: unlink + drop hook.
    - NFT_MSG_NEWRULE: unlink old rule (if replace) from chain.rules.
    - NFT_MSG_DELRULE: unlink.
    - NFT_MSG_NEWSET / DELSET: clear genmask; nf_tables_set_notify.
    - NFT_MSG_NEWSETELEM / DELSETELEM: per-elem activate / deactivate.
    - NFT_MSG_NEWOBJ / DELOBJ: clear genmask; per-update propagate.
    - NFT_MSG_NEWFLOWTABLE / DELFLOWTABLE: per-hook attach/detach.
18. Nft::set_commit_update(&set_update_list).
19. nft_commit_notify(net, portid).
20. Nft::commit_audit_log(&adl, base_seq).
21. nft_gc_seq_end(nft_net, gc_seq).
22. Nft::commit_release(net) — sched trans-destroy-work.

`Nft::abort(net, skb, action)`:
1. Walk nft_net.commit_list LIFO.
2. for trans: match msg_type:
   - NEW* → unlink fresh object; release.
   - DEL* → restore (clear deactivate flag).
   - NEWRULE → release rule blob_next allocations via nf_tables_commit_chain_prepare_cancel.
   - NEWSETELEM → revert nft_trans_elems_new_abort (per-set abort_skip_removal honored).
3. INIT_LIST_HEAD(&nft_net.commit_list).
4. nf_tables_module_autoload(net) if pending.

`Nft::do_chain(pkt, chain) -> u32`:
1. basechain = chain; stackptr = 0; jumpstack[NFT_JUMP_STACK_SIZE].
2. regs.verdict.code = NFT_CONTINUE.
3. genbit = READ_ONCE(net.nft.gencursor).
4. /* Trace if enabled */
5. if static_branch_unlikely(nft_trace_enabled): nft_trace_init(&info, pkt, basechain).
6. 'do_chain: blob = rcu_dereference(chain.blob_gen_(genbit)).
7. rule = blob.data as *NftRuleDp.
8. 'next_rule: regs.verdict.code = NFT_CONTINUE.
9. for !rule.is_last; rule = nft_rule_next(rule):
   - for expr in nft_rule_dp_for_each_expr(rule):
     - dispatch fast-paths (cmp / cmp16 / bitwise / payload), else expr_call_ops_eval.
     - if regs.verdict.code != NFT_CONTINUE: break.
   - match regs.verdict.code: NFT_BREAK → continue; NFT_CONTINUE → trace + continue; else break.
10. /* Final verdict */
11. match regs.verdict.code & NF_VERDICT_MASK:
    - NF_ACCEPT | NF_QUEUE | NF_STOLEN → return code.
    - NF_DROP → return NF_DROP_REASON.
12. match regs.verdict.code:
    - NFT_JUMP: jumpstack[stackptr++].rule = nft_rule_next(rule); fall through.
    - NFT_GOTO: chain = regs.verdict.chain; goto 'do_chain.
    - NFT_CONTINUE | NFT_RETURN: break.
13. if stackptr > 0: stackptr--; rule = jumpstack[stackptr].rule; goto 'next_rule.
14. /* Trace policy */
15. if nft_counters_enabled: nft_update_chain_stats(basechain, pkt).
16. policy = nft_base_chain(basechain).policy.
17. return (policy == NF_DROP) ? NF_DROP_REASON(...) : policy.

`Nft::select_set_ops(ctx, flags, desc) -> Result<*NftSetOps, i32>`:
1. lockdep_assert(commit_mutex).
2. bops = NULL; best = NftSetEstimate { size: u64::MAX, lookup: u64::MAX, space: u64::MAX }.
3. for type in nft_set_types:
   - if (flags & type.features) != (flags & NFT_SET_FEATURES): continue.
   - if !type.ops.estimate(desc, flags, &est): continue.
   - match desc.policy:
     - NFT_SET_POL_PERFORMANCE: prefer (lower lookup, then lower space).
     - NFT_SET_POL_MEMORY: if !desc.size prefer (lower space, then lower lookup); else prefer lower size.
   - if better: bops = &type.ops; best = est.
4. if bops.is_some(): return Ok(bops); else return Err(-EOPNOTSUPP).

`Nft::chain_validate(ctx, chain) -> Result<(), i32>`:
1. depth = chain.vstate.depth; if depth >= NFT_JUMP_STACK_SIZE: return Err(-EMLINK).
2. for rule in chain.rules:
   - for expr in rule:
     - if expr.ops.validate: expr.ops.validate(ctx, expr, &chain.vstate)?.
3. return Ok(()).

`Nft::newtable(skb, info, attrs) -> Result<(), i32>`:
1. genmask = nft_genmask_next(net).
2. family = info.nfmsg.family; name = NLA_STRING(NFTA_TABLE_NAME).
3. table = nft_table_lookup(net, name, family, genmask, NLM_F_*).
4. if !IS_ERR(table):
   - if NLM_F_EXCL: return -EEXIST.
   - if NLM_F_REPLACE: return -EOPNOTSUPP.
   - return nf_tables_updtable(&ctx).
5. else if PTR_ERR != -ENOENT: return PTR_ERR(table).
6. /* Create */
7. table = kzalloc(sizeof(NftTable)).
8. set name, family, flags, udata, hgenerator, handle = nf_tables_alloc_handle(table).
9. INIT_LIST_HEAD on chains, sets, objects, flowtables.
10. rhltable_init(&table.chains_ht).
11. nft_trans_table_add(&ctx, NFT_MSG_NEWTABLE).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `commit_publish_under_smp_release` | INVARIANT | per-commit: `net.nft.base_seq` published via `smp_store_release`. |
| `do_chain_load_acquire` | INVARIANT | per-do_chain: `gencursor` read via READ_ONCE; blob via rcu_dereference. |
| `jumpstack_bound` | INVARIANT | per-do_chain: stackptr < NFT_JUMP_STACK_SIZE. |
| `rule_genmask_one_bit_set` | INVARIANT | per-rule: at most one of {old, new} genmask bits set during transition. |
| `commit_mutex_held_select_set_ops` | INVARIANT | per-select_set_ops: `nft_net.commit_mutex` held. |
| `binding_list_drained_on_commit` | INVARIANT | per-commit success: nft_net.binding_list emptied. |
| `set_ops_lookup_no_block` | INVARIANT | per-set: ops.lookup never takes set->bindings or commit_mutex. |
| `abort_releases_pending_only` | INVARIANT | per-abort: only commit_list pending objects freed; committed unchanged. |

### Layer 2: TLA+

`net/netfilter/nf_tables.tla`:
- Per-batch + per-trans + per-commit + per-abort + per-do_chain + per-gencursor flip.
- Properties:
  - `safety_atomic_commit` — per-commit: all-or-nothing apply.
  - `safety_abort_restores_state` — per-abort: ruleset == pre-batch state.
  - `safety_gencursor_flip_after_blob_publish` — per-commit: `gencursor` flip strictly follows `commit_chain` blob swaps (else readers see torn rules).
  - `safety_base_seq_monotonic` — per-commit success: `base_seq` strictly increases (skipping 0).
  - `safety_dump_consistency` — per-dump: dump observes single base_seq; on change emits NFNL_F_DUMP_INTR.
  - `liveness_commit_terminates` — per-commit: terminates ∨ returns error.
  - `liveness_genid_check_serializes` — per-`nf_tables_valid_genid`: stale → -ERESTART.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Nft::commit` post: ret == Ok ⟹ commit_list empty ∧ base_seq bumped ∧ gencursor flipped | `Nft::commit` |
| `Nft::abort` post: commit_list empty ∧ no committed state mutated | `Nft::abort` |
| `Nft::do_chain` post: returns terminal NF_* or base policy | `Nft::do_chain` |
| `Nft::do_chain` post: regs.verdict.code = NFT_CONTINUE upon return | `Nft::do_chain` |
| `Nft::select_set_ops` post: returned ops minimal cost for policy | `Nft::select_set_ops` |
| `Nft::chain_validate` post: depth ≤ NFT_JUMP_STACK_SIZE | `Nft::chain_validate` |
| `NftSet::lookup` post: returned ext active in current gen | `NftSet::lookup` |
| `NftTrans::commit_list_add` post: trans linked at tail | `NftTrans::commit_list_add` |

### Layer 4: Verus/Creusot functional

`Per-NFNL_MSG_BATCH_BEGIN → genid check → per-NFT_MSG_* parse → commit (validate / offload / chain-prepare / publish / cursor-flip / apply / notify / release) ∨ abort (LIFO undo)` semantic equivalence: per-Documentation/networking/nf_tables/.

`Per-skb nft_do_chain → rule walk → expression eval → register VM → jumpstack → verdict resolution → base policy` semantic equivalence: per-Documentation/networking/netfilter-sysctl.rst + nftables wiki rule-evaluation order.

## Hardening

(Inherits row-1 features from `net/netfilter/00-overview.md` § Hardening.)

nftables core reinforcement:

- **Per-batch genid check (`nf_tables_valid_genid`)** — defense against per-stale-userspace concurrent edit.
- **Per-commit_mutex serializes control plane** — defense against per-torn-trans.
- **Per-gencursor + RCU rule-blob publication** — defense against per-data-plane reading half-installed ruleset.
- **Per-`smp_store_release` of base_seq + `smp_load_acquire` in nft_lookup_eval** — defense against per-ARM/POWER reorder visibility hole.
- **Per-`nft_table_has_owner` portid check** — defense against per-cross-namespace table hijack.
- **Per-anonymous set unbound rejection at commit** — defense against per-orphan-set-leak.
- **Per-`NFT_JUMP_STACK_SIZE`-bounded chain recursion** — defense against per-chain-call-cycle stack overflow.
- **Per-`nft_chain_validate` cycle-free check (per `vstate.depth`)** — defense against per-jump-cycle infinite loop in `nft_do_chain`.
- **Per-rule `genmask` 2-bit lifecycle** — defense against per-double-publication.
- **Per-set `abort_skip_removal` flag for non-revertible backends (pipapo)** — defense against per-set-corruption on abort.
- **Per-`nft_use_inc`/`nft_use_dec` saturating refcount with WARN-on-zero-dec** — defense against per-use-after-free of referenced objects.
- **Per-`rhltable` chain/object name hashtable** — defense against per-linear-scan DoS on large rulesets.
- **Per-`nft_request_module` autoloaded subset** — defense against per-`request_module`-spam (only registered names allowed).
- **Per-`NFNL_F_DUMP_INTR` dump abort on commit** — defense against per-iteration-over-mutated dump returning torn state.
- **Per-trans destroy via `trans_destroy_work` (schedule_work)** — defense against per-commit-path RCU stall.
- **Per-`nft_trans_gc_alloc` async GC** — defense against per-commit_mutex held during free of expired set elements.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `net/netfilter/nf_tables_core.c` per-expression fast-paths (covered in `nft-core.md` Tier-3)
- `net/netfilter/nf_tables_offload.c` HW-offload commit (covered in `flowtable.md` / `nft-api.md` Tier-3)
- `net/netfilter/nf_tables_trace.c` per-trace netlink notify (covered in `nft-api.md` Tier-3)
- Per-expression types (`nft_cmp.c`, `nft_payload.c`, `nft_meta.c`, `nft_lookup.c`, …) (covered in their own Tier-3 docs)
- Per-set backends (`nft_set_rhash.c`, `nft_set_hash.c`, `nft_set_bitmap.c`, `nft_set_rbtree.c`, `nft_set_pipapo.c`) (covered separately)
- `net/netfilter/nf_conntrack_core.c` conntrack (covered in `nf-conntrack-core.md` Tier-3)
- NAT integration (covered in `nat-core.md` Tier-3)
- Implementation code
