---
title: "Tier-3: drivers/clk/clk.c — Common Clock Framework (CCF) core"
tags: ["tier-3", "drivers", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **Common Clock Framework (CCF)** is the kernel-internal abstraction that lets clock-provider drivers (PLLs, dividers, muxes, gates, fixed-rate, fixed-factor, composite) expose a tree of clocks to clock-consumer drivers via a uniform API. Per-`struct clk_core`: refcounted node owning the topology (parent, children, parents[], rate, enable_count, prepare_count, protect_count, flags, accuracy, phase, duty, kref). Per-`struct clk`: per-consumer handle (`clk_get` returns one; `clk_put` releases) pointing at a shared `clk_core`. Per-`struct clk_hw`: provider's view binding `clk_core` to the hardware-specific subclass. Per-`struct clk_ops`: vtable (`prepare`, `unprepare`, `enable`, `disable`, `recalc_rate`, `round_rate`, `determine_rate`, `set_rate`, `get_parent`, `set_parent`, `set_rate_and_parent`, `is_enabled`, `init`, `terminate`, `restore_context`, `save_context`, `debug_init`, `set_phase`, `get_phase`, `set_duty_cycle`, `get_duty_cycle`). Per-prepare/enable refcount split: `clk_prepare` (sleepable, e.g. PLL lock-wait) vs `clk_enable` (atomic, fast gate). Per-`clk_set_rate` walk: round → propagate `PRE_RATE_CHANGE` SRCU notifier → `clk_change_rate` recursively → `POST_RATE_CHANGE`; on failure → `ABORT_RATE_CHANGE`. Per-OF: `of_clk_add_hw_provider` registers, `of_clk_get` / `of_clk_get_from_provider` look up by phandle. Per-debugfs: `/sys/kernel/debug/clk/clk_summary` enumerates the entire tree. Critical for: every clocked subsystem (timers, UART, I2C, SPI, USB, network MACs, GPU), DVFS, power management, suspend/resume context save/restore.

This Tier-3 covers `drivers/clk/clk.c` (~5586 lines).

### Acceptance Criteria

- [ ] AC-1: clk_register: orphan parents resolved when their parent registers later.
- [ ] AC-2: clk_unregister: ops swapped to clk_nodrv_ops; consumers see -ENXIO.
- [ ] AC-3: clk_prepare/unprepare nest with reference counting; calls ops->prepare only on 0→1.
- [ ] AC-4: clk_enable/disable atomic; spinlock-protected; calls ops->enable only on 0→1.
- [ ] AC-5: clk_enable requires prepare_count > 0 (WARN_ON_ONCE otherwise).
- [ ] AC-6: clk_set_rate: PRE_RATE_CHANGE veto causes ABORT_RATE_CHANGE and -EBUSY.
- [ ] AC-7: clk_set_rate: POST_RATE_CHANGE fires only on successful change with rate != old.
- [ ] AC-8: clk_set_parent: returns -EBUSY when CLK_SET_PARENT_GATE && prepare_count > 0.
- [ ] AC-9: clk_set_rate: returns -EBUSY when protect_count > 0.
- [ ] AC-10: of_clk_get: locates clk via clkspec phandle; returns -EPROBE_DEFER if provider not yet registered.
- [ ] AC-11: clk_disable_unused: skipped under clk_ignore_unused; otherwise disables clocks with refcount == 0.
- [ ] AC-12: clk_summary debugfs walks the entire tree including orphans.
- [ ] AC-13: clk_save_context/restore_context preserve refcounts across suspend/resume.
- [ ] AC-14: CLK_IS_CRITICAL clocks remain enabled after boot.
- [ ] AC-15: clk_notifier callbacks never re-enter the clk API (enforced via lockdep assertion).

### Architecture

```
struct ClkCore {
  name: Arc<str>,
  ops: &'static ClkOps,
  hw: NonNull<ClkHw>,
  owner: Option<*Module>,
  dev: Option<*Device>,
  of_node: Option<*DeviceNode>,
  parent: Option<Arc<ClkCore>>,
  parents: Box<[ClkParentMap]>,
  num_parents: u8,
  new_parent_index: u8,
  rate: u64,
  req_rate: u64,
  new_rate: u64,
  new_parent: Option<*ClkCore>,
  new_child: Option<*ClkCore>,
  flags: u32,                          // CLK_*
  orphan: bool,
  rpm_enabled: bool,
  enable_count: u32,
  prepare_count: u32,
  protect_count: u32,
  min_rate: u64,
  max_rate: u64,
  accuracy: u64,
  phase: i32,
  duty: ClkDuty,
  children: HList<ClkCore>,
  hashtable_node: HlistNode,
  clks: HList<Clk>,                    // per-consumer handles
  notifier_count: u32,
  debug: Option<DebugfsState>,
  refcount: Kref,
}

struct Clk {
  core: Arc<ClkCore>,
  dev: Option<*Device>,
  dev_id: Option<Arc<str>>,
  con_id: Option<Arc<str>>,
  min_rate: u64,
  max_rate: u64,
  exclusive_count: u32,
  clks_node: HlistNode,                // link into core.clks
}

struct ClkHw {
  core: *mut ClkCore,
  init: Option<*const ClkInitData>,    // valid only during register
  clk: *mut Clk,                       // per-provider's own consumer
}
```

`ClkCore::register_internal(dev, of_node, hw) -> Result<*Clk, errno>`:
1. Allocate `clk_core`, init kref, init lists.
2. Copy `init->name`, `init->ops`, `init->flags`, `init->num_parents`.
3. Populate `parents[]` from `init->parent_names`, `init->parent_hws`, `init->parent_data`.
4. clk_pm_runtime_init.
5. clk_prepare_lock.
6. `__clk_core_init`:
   - Re-resolve any orphan parent for which `hw` matches name.
   - For each parent slot: resolve to clk_core or mark orphan.
   - core->parent = clk_core_get_parent_by_index(core, ops->get_parent(hw)) (or NULL if no get_parent).
   - core->rate = clk_recalc(core, parent_rate).
   - core->accuracy = ops->recalc_accuracy(hw, parent_acc) if ops->recalc_accuracy.
   - core->phase = ops->get_phase(hw) if ops->get_phase.
   - if (flags & CLK_IS_CRITICAL): clk_core_prepare; clk_core_enable.
   - if (ops->init): ops->init(hw).
7. hash_add(clk_hashtable, &core->hashtable_node, name_hash).
8. hlist_add_head(&core->child_node, parent ? &parent->children : (orphan ? &clk_orphan_list : &clk_root_list)).
9. clk_debug_register(core).
10. clk_prepare_unlock.
11. Return clk_hw_create_clk(dev, hw, NULL, NULL).

`ClkCore::unregister(clk)`:
1. clk_debug_unregister.
2. clk_prepare_lock.
3. core->ops = &clk_nodrv_ops (under enable_lock).
4. if ops->terminate: ops->terminate(hw).
5. For each child: clk_core_set_parent_nolock(child, None) — moves to clk_orphan_list.
6. clk_core_evict_parent_cache — null out any cached pointers to this core.
7. hash_del + hlist_del_init.
8. WARN if prepare_count || protect_count.
9. clk_prepare_unlock.
10. kref_put(&core->refcount, ClkCore::release); free_clk(clk).

`Clk::prepare(clk) -> Result<(), errno>`:
1. if !clk: return Ok(()).
2. clk_prepare_lock (mutex).
3. `ClkCore::prepare(core)`:
   - if prepare_count == 0:
     - ClkCore::prepare(parent) (recursive).
     - clk_pm_runtime_get(core).
     - if (ops->prepare): ret = ops->prepare(hw); on err: clk_pm_runtime_put + return.
     - trace_clk_prepare(core).
   - core->prepare_count++.
4. clk_prepare_unlock.

`Clk::enable(clk) -> Result<(), errno>`:
1. if !clk: return Ok(()).
2. flags = clk_enable_lock (spin_lock_irqsave).
3. `ClkCore::enable(core)`:
   - WARN_ON_ONCE(core->prepare_count == 0).
   - if enable_count == 0:
     - ClkCore::enable(parent) (recursive).
     - if (ops->enable): ret = ops->enable(hw); on err: return.
     - trace_clk_enable(core).
   - core->enable_count++.
4. clk_enable_unlock(flags).

`Clk::set_rate(clk, rate) -> Result<(), errno>`:
1. if !clk: return Ok(()).
2. clk_prepare_lock.
3. rounded = ClkCore::req_round_rate_nolock(core, rate).
4. if rounded == cached_rate: unlock; return Ok(()).
5. if core->protect_count > 0: unlock; return Err(EBUSY).
6. top = ClkCore::calc_new_rates(core, rate). // populates new_rate, new_parent, new_parent_index on subtree
7. clk_pm_runtime_get(core).
8. fail = ClkCore::propagate_rate_change(top, PRE_RATE_CHANGE).
9. if fail:
   - ClkCore::propagate_rate_change(top, ABORT_RATE_CHANGE).
   - clk_pm_runtime_put; unlock; return Err(EBUSY).
10. ClkCore::change_rate(top).
11. core->req_rate = rate.
12. clk_pm_runtime_put.
13. clk_prepare_unlock.

`ClkCore::change_rate(core)`:
1. old_rate = core->rate.
2. clk_pm_runtime_get.
3. if (CLK_SET_RATE_UNGATE): prepare + enable.
4. if (new_parent && new_parent != parent):
   - old_parent = __clk_set_parent_before(core, new_parent).
   - if ops->set_rate_and_parent: ops->set_rate_and_parent(hw, new_rate, parent_rate, idx); skip_set_rate = true.
   - else if ops->set_parent: ops->set_parent(hw, idx).
   - __clk_set_parent_after(core, new_parent, old_parent).
5. if (CLK_OPS_PARENT_ENABLE): clk_core_prepare_enable(parent).
6. if (!skip_set_rate && ops->set_rate): ops->set_rate(hw, new_rate, parent_rate).
7. core->rate = clk_recalc(core, parent_rate).
8. if (CLK_SET_RATE_UNGATE): disable + unprepare.
9. if (CLK_OPS_PARENT_ENABLE): clk_core_disable_unprepare(parent).
10. if (notifier_count && old_rate != core->rate): __clk_notify(core, POST_RATE_CHANGE, old_rate, core->rate).
11. Recurse over children (safe iteration).

`Clk::set_parent(clk, parent) -> Result<(), errno>`:
1. if !clk: return Ok(()).
2. clk_prepare_lock.
3. if core->num_parents < 2: -EINVAL.
4. if !clk_has_parent(clk, parent): -EINVAL.
5. if (flags & CLK_SET_PARENT_GATE) && prepare_count > 0: -EBUSY.
6. p_index = clk_fetch_parent_index(core, parent->core).
7. ret = __clk_set_parent(core, parent->core, p_index).
8. clk_prepare_unlock.

`OfClk::add_hw_provider(np, get_hw, data) -> Result<(), errno>`:
1. cp = alloc(of_clk_provider).
2. cp.node = of_node_get(np); cp.get_hw = get_hw; cp.data = data.
3. mutex_lock(of_clk_mutex); list_add(&cp->link, &of_clk_providers); mutex_unlock.
4. clk_core_reparent_orphans — sweeps `clk_orphan_list` resolving newly-available fwnames.
5. of_clk_set_defaults(np, true).
6. fwnode_dev_initialized(&np->fwnode, true).

`OfClk::get_hw_from_clkspec(clkspec) -> Result<*ClkHw, errno>`:
1. mutex_lock(of_clk_mutex).
2. For each `cp` in of_clk_providers:
   - if cp->node != clkspec->np: continue.
   - hw = cp->get_hw ? cp->get_hw(clkspec, data) : __clk_get_hw(cp->get(clkspec, data)).
3. mutex_unlock.
4. Return hw or ERR_PTR(-EPROBE_DEFER) if no match.

`Clk::notifier_register(clk, nb) -> Result<(), errno>`:
1. Validate clk + nb.
2. clk_prepare_lock.
3. Look up `clk_notifier { clk, notifier_head, node }` on `clk_notifier_list`.
4. If absent: alloc + srcu_init_notifier_head + list_add.
5. srcu_notifier_chain_register(&cn->notifier_head, nb).
6. core->notifier_count++.
7. clk_prepare_unlock.

### Out of Scope

- drivers/clk/clk-divider.c / clk-mux.c / clk-gate.c / clk-fixed-rate.c / clk-fixed-factor.c / clk-composite.c primitive implementations (covered separately if expanded)
- drivers/clk/<vendor>/* SoC-specific providers (covered separately if expanded)
- drivers/clk/clkdev.c clkdev lookup table (covered separately if expanded)
- drivers/clk/clk-conf.c assigned-clocks DT parsing (covered separately if expanded)
- include/linux/clk-provider.h primitive helpers (covered in 00-overview)
- VDSO clock_gettime (not the same "clock"; covered in `kernel/time/`)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct clk_core` | per-clock node (topology + state) | `ClkCore` |
| `struct clk` | per-consumer handle | `Clk` |
| `struct clk_hw` | provider's hw binding | `ClkHw` |
| `struct clk_ops` | provider vtable | `ClkOps` |
| `struct clk_parent_map` | per-parent fwname/index/cache | `ClkParentMap` |
| `__clk_register()` | per-register internal | `ClkCore::register_internal` |
| `clk_register()` / `clk_hw_register()` | per-register (deprecated / preferred) | `ClkCore::register` / `ClkHw::register` |
| `clk_unregister()` / `clk_hw_unregister()` | per-unregister | `ClkCore::unregister` |
| `__clk_release()` | per-kref destructor | `ClkCore::release` |
| `clk_prepare()` / `clk_unprepare()` | sleepable enable/disable | `Clk::prepare` / `unprepare` |
| `clk_enable()` / `clk_disable()` | atomic enable/disable | `Clk::enable` / `disable` |
| `clk_prepare_enable()` / `clk_disable_unprepare()` | per-combined | `Clk::prepare_enable` / `disable_unprepare` |
| `clk_set_rate()` | per-rate set | `Clk::set_rate` |
| `clk_set_rate_range()` / `clk_set_min_rate` / `clk_set_max_rate` | per-bounds | `Clk::set_rate_range` |
| `clk_set_rate_exclusive()` / `clk_rate_exclusive_get/put` | per-exclusive | `Clk::set_rate_exclusive` |
| `clk_get_rate()` / `clk_round_rate()` | per-rate query | `Clk::get_rate` / `round_rate` |
| `clk_set_parent()` / `clk_get_parent()` | per-mux | `Clk::set_parent` / `get_parent` |
| `clk_hw_set_parent()` / `clk_hw_reparent()` | per-provider reparent | `ClkHw::set_parent` |
| `clk_set_phase()` / `clk_get_phase()` | per-phase | `Clk::set_phase` / `get_phase` |
| `clk_set_duty_cycle()` / `clk_get_scaled_duty_cycle` | per-duty | `Clk::set_duty_cycle` |
| `clk_get_accuracy()` | per-ppb accuracy | `Clk::get_accuracy` |
| `clk_save_context()` / `clk_restore_context()` | per-suspend/resume | `Clk::save_context` / `restore_context` |
| `__clk_notify()` | per-SRCU notify | `ClkCore::notify` |
| `clk_notifier_register()` / `_unregister` | per-consumer notifier | `Clk::notifier_register` |
| `clk_propagate_rate_change()` | per-subtree notify | `ClkCore::propagate_rate_change` |
| `clk_change_rate()` | per-subtree set | `ClkCore::change_rate` |
| `clk_calc_new_rates()` | per-determine | `ClkCore::calc_new_rates` |
| `clk_core_prepare()` / `_enable` / `_disable` / `_unprepare` (+ `_lock` variants) | internal refcount paths | `ClkCore::*` |
| `clk_disable_unused()` / `_subtree` | per-late_initcall reaper | `ClkCore::disable_unused` |
| `__clk_lookup()` | per-name lookup (hashtable) | `ClkCore::lookup` |
| `clk_core_lookup()` | per-name lookup | `ClkCore::core_lookup` |
| `of_clk_add_provider()` / `of_clk_add_hw_provider()` | per-provider register | `OfClk::add_provider` / `add_hw_provider` |
| `of_clk_del_provider()` | per-provider unregister | `OfClk::del_provider` |
| `of_clk_get()` / `of_clk_get_by_name()` / `of_clk_get_from_provider()` | per-consumer lookup | `OfClk::get` |
| `of_clk_get_hw()` / `of_clk_get_hw_from_clkspec()` | per-hw lookup | `OfClk::get_hw` |
| `of_clk_get_parent_count()` / `of_clk_get_parent_name()` | per-tree introspection | `OfClk::get_parent_*` |
| `of_clk_set_defaults()` | per-DT defaults | `OfClk::set_defaults` |
| `__clk_get()` / `__clk_put()` | per-module refcount | `Clk::get` / `put` |
| `clk_hw_create_clk()` / `clk_hw_get_clk()` | per-consumer alloc | `ClkHw::create_clk` |
| `clk_summary_show()` / `clk_dump_show()` | debugfs | `ClkDebug::summary` / `dump` |
| `clk_debug_register()` / `_unregister` / `_init` | debugfs lifecycle | `ClkDebug::*` |
| `clk_nodrv_ops` | per-stub-after-unregister | `clk_nodrv_ops` |

### compatibility contract

REQ-1: struct clk_core:
- name: provider-supplied identifier.
- ops: `&clk_ops`.
- hw: back-pointer to `struct clk_hw`.
- owner: per-module reference.
- dev: per-device.
- rpm_node: per-runtime-PM list link.
- of_node: per-DT node.
- parent: current parent (`clk_core *`).
- parents: `clk_parent_map[num_parents]` cache of fwname/index/core.
- num_parents: u8.
- new_parent_index / new_parent / new_child / new_rate / req_rate: per-rate-change scratch.
- rate / req_rate: cached recalc'd rate and last requested rate.
- flags: CLK_SET_RATE_GATE / _SET_PARENT_GATE / _SET_RATE_PARENT / _IGNORE_UNUSED / _IS_BASIC / _GET_RATE_NOCACHE / _SET_RATE_NO_REPARENT / _GET_ACCURACY_NOCACHE / _RECALC_NEW_RATES / _SET_RATE_UNGATE / _IS_CRITICAL / _OPS_PARENT_ENABLE / _DUTY_CYCLE_PARENT.
- orphan: bool — parent not yet registered.
- rpm_enabled: bool.
- enable_count / prepare_count / protect_count: refcounts.
- min_rate / max_rate / accuracy / phase / duty: state.
- children / child_node / hashtable_node / clks: list links.
- notifier_count: u32.
- dentry / debug_node: per-debugfs.
- ref: `struct kref`.

REQ-2: struct clk (per-consumer):
- core: shared `clk_core *`.
- dev / dev_id / con_id: provenance.
- min_rate / max_rate: per-consumer bounds (intersected with core's).
- exclusive_count: per-`clk_rate_exclusive_get`.
- clks_node: link in `core->clks`.

REQ-3: struct clk_ops (vtable):
- prepare / unprepare: sleepable.
- is_prepared.
- unprepare_unused (boot).
- enable / disable / is_enabled / disable_unused: atomic context.
- recalc_rate(hw, parent_rate) → rate.
- round_rate(hw, rate, &parent_rate) (legacy) or determine_rate(hw, req).
- set_rate(hw, rate, parent_rate).
- set_parent(hw, index) / get_parent(hw) → index.
- set_rate_and_parent(hw, rate, parent_rate, index).
- recalc_accuracy.
- get_phase / set_phase.
- get_duty_cycle / set_duty_cycle.
- save_context / restore_context.
- init / terminate.
- debug_init.

REQ-4: clk_register / clk_hw_register / of_clk_hw_register:
- `__clk_register(dev, np, hw)`:
  - `clk_core = kzalloc(sizeof(*clk_core))`.
  - Copy `name`, `ops`, `flags`, `hw`, `dev`, `of_node` (per-`dev_or_parent_of_node`), `owner`.
  - `kref_init(&clk_core->ref)`.
  - INIT_HLIST_HEAD(&core->children); INIT_HLIST_HEAD(&core->clks).
  - clk_core_populate_parent_map(core, init) — copy parent fwnames.
  - clk_pm_runtime_init(core).
  - `clk_prepare_lock()`.
  - `__clk_core_init(core)` — find parents, recalc_rate, recalc_accuracy, recalc_phase, mark `CLK_IS_CRITICAL` prepared+enabled.
  - hash_add(clk_hashtable, &core->hashtable_node).
  - hlist_add_head(&core->child_node, &parent->children) or &clk_root_list / &clk_orphan_list.
  - clk_debug_register(core).
  - `clk_prepare_unlock()`.
  - Return per-consumer `struct clk *` via `clk_hw_create_clk(dev, hw, NULL, NULL)`.

REQ-5: clk_unregister:
- WARN if !clk ∨ IS_ERR(clk).
- clk_debug_unregister(clk->core).
- clk_prepare_lock.
- If ops == &clk_nodrv_ops: pr_err + return.
- clk_enable_lock; core->ops = &clk_nodrv_ops; clk_enable_unlock.
- If ops->terminate: ops->terminate(hw).
- For all children: clk_core_set_parent_nolock(child, NULL) — reparent to orphan list.
- clk_core_evict_parent_cache(core).
- hash_del + hlist_del_init.
- Warn-if prepare_count or protect_count non-zero.
- clk_prepare_unlock.
- kref_put(&core->ref, __clk_release); free_clk(clk).

REQ-6: clk_prepare / clk_unprepare (sleepable):
- `clk_prepare(clk)`:
  - if !clk: return 0.
  - clk_core_prepare_lock(core):
    - clk_prepare_lock (mutex).
    - clk_core_prepare(core):
      - if prepare_count == 0: clk_core_prepare(parent) recursively; clk_pm_runtime_get; ops->prepare; trace.
      - prepare_count++.
    - clk_prepare_unlock.
- `clk_unprepare(clk)`:
  - mirror: --prepare_count; if 0: ops->unprepare; clk_pm_runtime_put; clk_core_unprepare(parent).

REQ-7: clk_enable / clk_disable (atomic):
- Uses `enable_lock` spinlock (irqsave).
- `clk_enable(clk)`:
  - clk_core_enable_lock(core):
    - flags = clk_enable_lock (spin_lock_irqsave).
    - clk_core_enable(core):
      - if enable_count == 0: clk_core_enable(parent); ops->enable; trace.
      - enable_count++.
    - clk_enable_unlock(flags).
- `clk_disable(clk)`:
  - mirror; --enable_count; if 0: ops->disable.
- Constraint: prepare_count > 0 required before enable (lockdep + WARN_ON_ONCE if violated).

REQ-8: clk_set_rate:
- `clk_set_rate(clk, rate)`:
  - if !clk: return 0.
  - clk_prepare_lock.
  - ret = clk_core_set_rate_nolock(clk->core, rate).
  - clk_prepare_unlock.
- `clk_core_set_rate_nolock(core, req_rate)`:
  - rate = clk_core_req_round_rate_nolock(core, req_rate).
  - if rate == cached: return 0.
  - if protect_count > 0: return -EBUSY.
  - top = clk_calc_new_rates(core, req_rate) — recursive: chooses parent and new_rate.
  - clk_pm_runtime_get(core).
  - fail = clk_propagate_rate_change(top, PRE_RATE_CHANGE) — SRCU notify subtree.
  - if fail: clk_propagate_rate_change(top, ABORT_RATE_CHANGE); return -EBUSY.
  - clk_change_rate(top) — recursive: set_parent + set_rate + POST_RATE_CHANGE per node.
  - core->req_rate = req_rate.
  - clk_pm_runtime_put.

REQ-9: clk_change_rate (per-subtree):
- old_rate = core->rate.
- clk_pm_runtime_get.
- if (flags & CLK_SET_RATE_UNGATE): clk_core_prepare + clk_core_enable_lock.
- if (new_parent && new_parent != parent):
  - old_parent = __clk_set_parent_before(core, new_parent).
  - if ops->set_rate_and_parent: ops->set_rate_and_parent(hw, new_rate, parent_rate, idx); skip_set_rate = true.
  - else if ops->set_parent: ops->set_parent(hw, idx).
  - __clk_set_parent_after(core, new_parent, old_parent).
- if (flags & CLK_OPS_PARENT_ENABLE): clk_core_prepare_enable(parent).
- if (!skip_set_rate && ops->set_rate): ops->set_rate(hw, new_rate, parent_rate).
- core->rate = clk_recalc(core, parent_rate).
- if (flags & CLK_SET_RATE_UNGATE): clk_core_disable_lock + clk_core_unprepare.
- if (flags & CLK_OPS_PARENT_ENABLE): clk_core_disable_unprepare(parent).
- if (notifier_count && old_rate != core->rate): __clk_notify(core, POST_RATE_CHANGE, old_rate, core->rate).
- Recurse into children (safe iteration).

REQ-10: clk_set_parent:
- if !clk: return 0; if !parent ∧ !(flags & CLK_SET_PARENT_GATE): allow NULL.
- clk_prepare_lock.
- if num_parents <= 1: return -EINVAL.
- if (CLK_SET_PARENT_GATE) && prepare_count > 0: return -EBUSY.
- p_index = clk_fetch_parent_index(core, parent_core).
- ret = __clk_set_parent(core, parent_core, p_index).
- clk_prepare_unlock.

REQ-11: clk_notifier_register:
- Allocates a `struct clk_notifier` wrapping an SRCU notifier head if not already on `clk_notifier_list`.
- srcu_notifier_chain_register(&cn->notifier_head, nb).
- core->notifier_count++.
- Events: PRE_RATE_CHANGE / POST_RATE_CHANGE / ABORT_RATE_CHANGE — `struct clk_notifier_data { clk, old_rate, new_rate }`.
- Callbacks MUST NOT call back into clk API (would re-take prepare_lock; deadlock).

REQ-12: clk_save_context / clk_restore_context:
- Per-suspend: walk all roots, call ops->save_context on each.
- Per-resume: walk all roots, call ops->restore_context; if enable_count > 0: re-enable; if prepare_count > 0: re-prepare. Used by `clk_gate_restore_context` etc.

REQ-13: clk_disable_unused (late_initcall):
- Walk `clk_root_list` + `clk_orphan_list`:
  - clk_unprepare_unused_subtree: if !prepare_count && ops->unprepare_unused: invoke.
  - clk_disable_unused_subtree: if !enable_count && ops->disable_unused: invoke.
- Skipped if `clk_ignore_unused` cmdline param set.

REQ-14: of_clk_add_hw_provider / of_clk_get:
- `of_clk_add_hw_provider(np, get, data)`:
  - Allocates `of_clk_provider { node = of_node_get(np), get_hw = get, data }`.
  - mutex_lock(&of_clk_mutex); list_add(&cp->link, &of_clk_providers); mutex_unlock.
  - clk_core_reparent_orphans — pick up orphans whose fwname matches.
  - of_clk_set_defaults(np, true) — apply assigned-clocks / assigned-clock-rates / assigned-clock-parents from DT.
- `of_clk_get(np, index)` / `of_clk_get_by_name(np, name)`:
  - of_parse_phandle_with_args(np, "clocks", "#clock-cells", index, &clkspec).
  - hw = of_clk_get_hw_from_clkspec(&clkspec).
  - Walks `of_clk_providers`; calls provider->get_hw(&clkspec, data) or get(&clkspec, data).
  - Returns `clk_hw_create_clk(...)` per-consumer.

REQ-15: clk_summary (debugfs):
- /sys/kernel/debug/clk/clk_summary — table with columns: `enable_cnt prepare_cnt protect_cnt rate accuracy phase duty hardware_enable`.
- Walk all_lists[] (root + orphan); for each: clk_summary_show_subtree(s, c, 0) recursive (indented by level).
- Per-clock: also creates `/sys/kernel/debug/clk/<name>/{clk_rate, clk_min_rate, clk_max_rate, clk_accuracy, clk_phase, clk_flags, clk_prepare_count, clk_enable_count, clk_protect_count, clk_notifier_count, clk_duty_cycle, current_parent, possible_parents}`.

REQ-16: SRCU notifier event semantics:
- PRE_RATE_CHANGE: pre-change; callback may veto by returning NOTIFY_STOP / NOTIFY_BAD ∣ NOTIFY_STOP_MASK.
- POST_RATE_CHANGE: post-change; advisory.
- ABORT_RATE_CHANGE: pre-changed rate reverted because a sibling/child vetoed.

REQ-17: Critical clocks (CLK_IS_CRITICAL):
- During __clk_core_init: clk_core_prepare + clk_core_enable so refcount always > 0 — never disabled.

REQ-18: Rate-protect (clk_rate_exclusive_get/put):
- `clk_rate_exclusive_get(clk)`: clk_core_rate_protect(core); core->protect_count++; clk->exclusive_count++.
- While protect_count > 0: clk_set_rate returns -EBUSY for this subtree.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prepare_count_monotone_within_critical` | INVARIANT | per-clk_core_prepare: refcount only changes under prepare_lock. |
| `enable_count_monotone_within_critical` | INVARIANT | per-clk_core_enable: refcount only changes under enable_lock spinlock. |
| `prepare_before_enable` | INVARIANT | per-clk_core_enable: prepare_count > 0 (WARN_ON_ONCE). |
| `parent_chain_acyclic` | INVARIANT | per-clk_register: __clk_core_init refuses cyclic parent. |
| `nodrv_ops_after_unregister` | INVARIANT | per-clk_unregister: ops pointer atomically swapped to clk_nodrv_ops. |
| `protect_count_blocks_set_rate` | INVARIANT | per-clk_set_rate: protect_count > 0 ⟹ -EBUSY. |
| `notifier_callback_no_reentry` | INVARIANT | per-__clk_notify: lockdep_assert_held(prepare_lock) → callback must not call top-level clk APIs. |
| `kref_balance_register_unregister` | INVARIANT | per-clk_register / clk_unregister: kref pairs. |
| `orphan_list_membership` | INVARIANT | per-__clk_core_init: clk on exactly one of root / orphan list / child list. |

### Layer 2: TLA+

`drivers/clk/clk-core.tla`:
- Per-register, per-unregister, per-prepare/unprepare, per-enable/disable, per-set_rate (with PRE/ABORT/POST), per-set_parent, per-notifier-register.
- Properties:
  - `safety_prepare_count_nonneg` — never decrements below 0.
  - `safety_enable_implies_prepared` — enable_count > 0 ⟹ prepare_count > 0.
  - `safety_set_rate_atomic_under_protect` — protect_count > 0 forbids successful set_rate.
  - `safety_pre_rate_change_veto_aborts` — PRE_RATE_CHANGE NOTIFY_STOP ⟹ rate unchanged.
  - `safety_post_rate_change_only_on_change` — POST_RATE_CHANGE only fires when old_rate != new_rate.
  - `safety_parent_chain_acyclic` — at all times, parent chain has no cycle.
  - `safety_orphan_resolved_on_provider_register` — once parent registers, orphan moves to its children list before next set_rate.
  - `liveness_per_set_rate_returns` — set_rate eventually completes (succeed or fail).
  - `liveness_per_unregister_completes_with_consumers` — unregister + clk_nodrv_ops keeps consumers alive until last clk_put.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `__clk_register` post: orphan ⟺ no fwname-resolvable parent | `ClkCore::register_internal` |
| `clk_unregister` post: ops == &clk_nodrv_ops; children reparented to orphan list | `ClkCore::unregister` |
| `clk_prepare` post: prepare_count incremented; on err prepare_count unchanged | `Clk::prepare` |
| `clk_enable` post: enable_count incremented; on err enable_count unchanged | `Clk::enable` |
| `clk_set_rate` post: protect_count > 0 ⟹ returns -EBUSY; rate unchanged | `Clk::set_rate` |
| `clk_set_rate` post: PRE veto ⟹ ABORT broadcast; rate unchanged | `Clk::set_rate` |
| `clk_set_parent` post: CLK_SET_PARENT_GATE && prepare_count > 0 ⟹ -EBUSY | `Clk::set_parent` |
| `of_clk_get` post: missing provider ⟹ -EPROBE_DEFER | `OfClk::get` |
| `clk_disable_unused` post: enable_count == 0 clocks call ops->disable_unused | `ClkCore::disable_unused` |
| `clk_notifier_register` post: notifier_count incremented; SRCU chain has nb | `Clk::notifier_register` |
| `clk_save_context` / `clk_restore_context` post: refcounts preserved | `Clk::save_context` / `restore_context` |

### Layer 4: Verus/Creusot functional

`Per-consumer model: clk_get → clk_prepare → clk_enable → use → clk_disable → clk_unprepare → clk_put` semantic equivalence: per-Documentation/driver-api/clk.rst + Documentation/devicetree/bindings/clock/clock-bindings.yaml. Per-provider model: `__clk_register` populates tree, `clk_set_rate` walks PRE/ABORT/POST notifications, `clk_unregister` reparents children to orphan list and swaps ops to nodrv stub.

### hardening

(Inherits row-1 features from `drivers/clk/00-overview.md` § Hardening.)

CCF reinforcement:

- **Per-prepare_lock mutex** — defense against per-concurrent topology mutation.
- **Per-enable_lock spinlock** — defense against per-IRQ context atomic enable race.
- **Per-WARN_ON_ONCE enable-without-prepare** — defense against per-consumer ordering bug.
- **Per-CLK_SET_PARENT_GATE refused while prepared** — defense against per-glitching the consumer.
- **Per-CLK_SET_RATE_GATE / _UNGATE flags honored** — defense against per-illegal-while-active rate change.
- **Per-PRE_RATE_CHANGE veto / ABORT_RATE_CHANGE rollback** — defense against per-consumer-out-of-spec rate.
- **Per-protect_count blocks set_rate** — defense against per-DVFS conflict with exclusive consumer.
- **Per-clk_nodrv_ops after unregister** — defense against per-UAF against late consumers.
- **Per-clk_core_evict_parent_cache** — defense against per-stale parent pointer after unregister.
- **Per-orphan_list resolution on provider register** — defense against per-DT-probe-order dependency.
- **Per-CLK_IS_CRITICAL prepared+enabled at register** — defense against per-disabling-foundation-clock.
- **Per-clk_disable_unused skipped under clk_ignore_unused** — defense against per-early-boot disable.
- **Per-SRCU notifier (sleepable, no callback-reentry)** — defense against per-notifier deadlock.
- **Per-kref refcount on clk_core** — defense against per-UAF across module unload.

