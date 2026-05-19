# Tier-3: mm/vmpressure.c — Memory-pressure notification to userspace

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/vmpressure.c (~481 lines)
  - include/linux/vmpressure.h
  - include/linux/memcontrol.h (struct vmpressure embedded in struct mem_cgroup)
-->

## Summary

`vmpressure` is the kernel's userspace-facing memory-pressure notifier, used by oomd-style daemons to react before the in-kernel OOM killer fires. Per-memcg, `struct vmpressure` is embedded in `struct mem_cgroup` (one root + one per non-root memcg). Per-reclaim-callsite, `vmpressure(gfp, memcg, tree, scanned, reclaimed)` is called from vmscan accumulating `scanned` / `reclaimed` page counts; once the per-window accumulator (`vmpressure_win = SWAP_CLUSTER_MAX * 16` = 512 pages) overflows, `vmpressure_work_fn()` is scheduled. The work computes a pressure ratio = `100 - (reclaimed * 100 / scanned)` and bins to `VMPRESSURE_LOW` (< 60), `VMPRESSURE_MEDIUM` (≥ 60), or `VMPRESSURE_CRITICAL` (≥ 95). Per-priority, `vmpressure_prio(gfp, memcg, prio)` short-circuits to critical when reclaimer's `prio ≤ ilog2(10) = 3` ("close-to-OOM" scanning depth). Per-eventfd, `vmpressure_register_event(memcg, eventfd, "level[,mode]")` parses level + optional mode (`default` / `hierarchy` / `local`) and links it to the memcg. On event, the work walks the memcg hierarchy delivering eventfd signals respecting per-event mode flags (`VMPRESSURE_LOCAL` skips ancestor delivery, `VMPRESSURE_NO_PASSTHROUGH` stops once an ancestor has signalled). Critical for: pre-OOM userspace warning, container memory-pressure feedback loop, network socket-buffer back-pressure (`mem_cgroup_set_socket_pressure`).

This Tier-3 covers `mm/vmpressure.c` (~481 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct vmpressure` | per-memcg pressure state | `Vmpressure` |
| `struct vmpressure_event` | per-eventfd binding | `VmpressureEvent` |
| `enum vmpressure_levels` | LOW / MEDIUM / CRITICAL | `VmpressureLevel` |
| `enum vmpressure_modes` | default / hierarchy / local | `VmpressureMode` |
| `vmpressure()` | per-vmscan accumulator | `Vmpressure::account` |
| `vmpressure_prio()` | per-priority critical-trip | `Vmpressure::account_prio` |
| `vmpressure_work_fn()` | per-deferred-evaluate | `Vmpressure::work_fn` |
| `vmpressure_calc_level()` | per-ratio bin | `Vmpressure::calc_level` |
| `vmpressure_level()` | per-pressure → level | `Vmpressure::level_for` |
| `vmpressure_event()` | per-eventfd delivery | `Vmpressure::deliver` |
| `vmpressure_parent()` | per-hierarchy walk | `Vmpressure::parent` |
| `vmpressure_register_event()` | per-bind eventfd | `Vmpressure::register_event` |
| `vmpressure_unregister_event()` | per-detach eventfd | `Vmpressure::unregister_event` |
| `vmpressure_init()` | per-memcg-create init | `Vmpressure::init` |
| `vmpressure_cleanup()` | per-memcg-destroy flush | `Vmpressure::cleanup` |
| `vmpressure_win` (= SWAP_CLUSTER_MAX * 16 = 512) | per-window pages | `VMPRESSURE_WIN` |
| `vmpressure_level_med` (= 60) | per-medium threshold | `VMPRESSURE_LEVEL_MED` |
| `vmpressure_level_critical` (= 95) | per-critical threshold | `VMPRESSURE_LEVEL_CRITICAL` |
| `vmpressure_level_critical_prio` (= ilog2(10) = 3) | per-priority critical-trip | `VMPRESSURE_LEVEL_CRITICAL_PRIO` |
| `vmpressure_str_levels` / `vmpressure_str_modes` | per-name table | `Vmpressure::LEVEL_NAMES` / `MODE_NAMES` |
| `mem_cgroup_set_socket_pressure()` | per-socket buffer back-pressure | external (memcontrol) |

## Compatibility contract

REQ-1: struct vmpressure (embedded in struct mem_cgroup):
- scanned: ulong, per-memcg own-accumulator scanned-pages.
- reclaimed: ulong, per-memcg own-accumulator reclaimed-pages.
- tree_scanned: ulong, per-memcg subtree-mode scanned-pages.
- tree_reclaimed: ulong, per-memcg subtree-mode reclaimed-pages.
- sr_lock: spinlock_t, per-accumulator (s/r) lock.
- events: list_head, per-vmpressure_event chain.
- events_lock: mutex, per-events-list lock.
- work: work_struct, per-deferred evaluator (vmpressure_work_fn).

REQ-2: struct vmpressure_event (per-eventfd binding):
- efd: *eventfd_ctx, per-userspace fd.
- level: enum vmpressure_levels (LOW / MEDIUM / CRITICAL).
- mode: enum vmpressure_modes (NO_PASSTHROUGH / HIERARCHY / LOCAL).
- node: list_head, per-vmpressure.events chain.

REQ-3: enum vmpressure_levels:
- VMPRESSURE_LOW = 0.
- VMPRESSURE_MEDIUM = 1.
- VMPRESSURE_CRITICAL = 2.
- VMPRESSURE_NUM_LEVELS = 3.

REQ-4: enum vmpressure_modes:
- VMPRESSURE_NO_PASSTHROUGH = 0 ("default").
- VMPRESSURE_HIERARCHY = 1 ("hierarchy").
- VMPRESSURE_LOCAL = 2 ("local").
- VMPRESSURE_NUM_MODES = 3.

REQ-5: Tunables (compile-time consts):
- vmpressure_win = SWAP_CLUSTER_MAX * 16 (= 512 pages on 4 K, ≈ 2 MiB).
- vmpressure_level_med = 60 (%).
- vmpressure_level_critical = 95 (%).
- vmpressure_level_critical_prio = ilog2(100 / 10) = ilog2(10) = 3.

REQ-6: vmpressure_level(pressure) -> level:
- if pressure >= vmpressure_level_critical: return VMPRESSURE_CRITICAL.
- else if pressure >= vmpressure_level_med: return VMPRESSURE_MEDIUM.
- return VMPRESSURE_LOW.

REQ-7: vmpressure_calc_level(scanned, reclaimed) -> level:
- scale = scanned + reclaimed.
- pressure = 0.
- /* slab path: shrink_node bumps reclaimed without scanned; guard */
- if reclaimed >= scanned: goto out (pressure stays 0).
- pressure = scale - (reclaimed * scale / scanned).
- pressure = pressure * 100 / scale.
- pr_debug("pressure %3lu (s: %lu r: %lu)").
- return vmpressure_level(pressure).

REQ-8: vmpressure(gfp, memcg, tree, scanned, reclaimed):
- /* Disabled-cgroup fast-path */
- if mem_cgroup_disabled(): return.
- /* Legacy v1: in-kernel users only see subtree (tree==true) */
- if !cgroup_subsys_on_dfl(memory_cgrp_subsys) ∧ !tree: return.
- vmpr = memcg_to_vmpressure(memcg).
- /* User-actionable gfp only: HIGHMEM / MOVABLE / IO / FS */
- if !(gfp & (__GFP_HIGHMEM | __GFP_MOVABLE | __GFP_IO | __GFP_FS)): return.
- /* No scanned ⟹ no LRU; let vmpressure_prio handle critical */
- if !scanned: return.
- if tree:
  - spin_lock(&vmpr.sr_lock).
  - scanned = vmpr.tree_scanned += scanned.
  - vmpr.tree_reclaimed += reclaimed.
  - spin_unlock.
  - if scanned < vmpressure_win: return.
  - schedule_work(&vmpr.work).
- else:
  - if !memcg ∨ mem_cgroup_is_root(memcg): return /* root-level efficiency unused */.
  - spin_lock(&vmpr.sr_lock).
  - scanned = vmpr.scanned += scanned.
  - reclaimed = vmpr.reclaimed += reclaimed.
  - if scanned < vmpressure_win: spin_unlock; return.
  - vmpr.scanned = vmpr.reclaimed = 0.
  - spin_unlock.
  - level = vmpressure_calc_level(scanned, reclaimed).
  - if level > VMPRESSURE_LOW:
    - mem_cgroup_set_socket_pressure(memcg) /* 1-second hysteresis */.

REQ-9: vmpressure_prio(gfp, memcg, prio):
- if prio > vmpressure_level_critical_prio: return.
- /* Force critical by feeding scanned=vmpressure_win, reclaimed=0 */
- vmpressure(gfp, memcg, true, vmpressure_win, 0).

REQ-10: vmpressure_work_fn(work):
- vmpr = container_of(work, struct vmpressure, work).
- spin_lock(&vmpr.sr_lock).
- scanned = vmpr.tree_scanned.
- if !scanned: spin_unlock; return /* rescheduled twice */.
- reclaimed = vmpr.tree_reclaimed.
- vmpr.tree_scanned = 0; vmpr.tree_reclaimed = 0.
- spin_unlock.
- level = vmpressure_calc_level(scanned, reclaimed).
- ancestor = false; signalled = false.
- loop:
  - if vmpressure_event(vmpr, level, ancestor, signalled): signalled = true.
  - ancestor = true.
  - vmpr = vmpressure_parent(vmpr).
  - if !vmpr: break.

REQ-11: vmpressure_event(vmpr, level, ancestor, signalled) -> bool:
- ret = false.
- mutex_lock(&vmpr.events_lock).
- for ev in vmpr.events:
  - /* mode LOCAL: only fires on own memcg, not ancestors */
  - if ancestor ∧ ev.mode == VMPRESSURE_LOCAL: continue.
  - /* mode NO_PASSTHROUGH: stops once any ancestor signalled */
  - if signalled ∧ ev.mode == VMPRESSURE_NO_PASSTHROUGH: continue.
  - if level < ev.level: continue.
  - eventfd_signal(ev.efd).
  - ret = true.
- mutex_unlock.
- return ret.

REQ-12: vmpressure_parent(vmpr) -> *vmpressure:
- memcg = vmpressure_to_memcg(vmpr).
- memcg = parent_mem_cgroup(memcg).
- if !memcg: return NULL.
- return memcg_to_vmpressure(memcg).

REQ-13: vmpressure_register_event(memcg, eventfd, args) -> int:
- vmpr = memcg_to_vmpressure(memcg).
- mode = VMPRESSURE_NO_PASSTHROUGH.
- spec = kstrndup(args, MAX_VMPRESSURE_ARGS_LEN, GFP_KERNEL) /* "critical,hierarchy" + NUL */.
- if !spec: return -ENOMEM.
- token = strsep(&spec, ",").
- ret = match_string(vmpressure_str_levels, VMPRESSURE_NUM_LEVELS, token).
- if ret < 0: goto out (-EINVAL).
- level = ret.
- token = strsep(&spec, ",").
- if token:
  - ret = match_string(vmpressure_str_modes, VMPRESSURE_NUM_MODES, token).
  - if ret < 0: goto out (-EINVAL).
  - mode = ret.
- ev = kzalloc(sizeof(*ev), GFP_KERNEL).
- if !ev: ret = -ENOMEM; goto out.
- ev.efd = eventfd; ev.level = level; ev.mode = mode.
- mutex_lock(&vmpr.events_lock).
- list_add(&ev.node, &vmpr.events).
- mutex_unlock.
- ret = 0.
- out: kfree(spec_orig); return ret.

REQ-14: vmpressure_unregister_event(memcg, eventfd):
- vmpr = memcg_to_vmpressure(memcg).
- mutex_lock(&vmpr.events_lock).
- for ev in vmpr.events:
  - if ev.efd != eventfd: continue.
  - list_del(&ev.node).
  - kfree(ev).
  - break.
- mutex_unlock.

REQ-15: vmpressure_init(vmpr):
- spin_lock_init(&vmpr.sr_lock).
- mutex_init(&vmpr.events_lock).
- INIT_LIST_HEAD(&vmpr.events).
- INIT_WORK(&vmpr.work, vmpressure_work_fn).

REQ-16: vmpressure_cleanup(vmpr):
- flush_work(&vmpr.work) /* drain any in-flight evaluate before memcg dtor */.

REQ-17: MAX_VMPRESSURE_ARGS_LEN = strlen("critical") + strlen("hierarchy") + 2 (= 8 + 9 + 2 = 19).

REQ-18: vmpressure_str_levels = ["low", "medium", "critical"].

REQ-19: vmpressure_str_modes = ["default", "hierarchy", "local"].

## Acceptance Criteria

- [ ] AC-1: vmpressure(gfp without HIGHMEM|MOVABLE|IO|FS, …): no-op (not user-actionable).
- [ ] AC-2: vmpressure(scanned=0): no-op (no LRU touched).
- [ ] AC-3: vmpressure(tree=false) on root memcg: no-op.
- [ ] AC-4: vmpressure(tree=true): accumulator overflow at vmpressure_win triggers schedule_work.
- [ ] AC-5: vmpressure_calc_level(scanned=1024, reclaimed=0): returns VMPRESSURE_CRITICAL (pressure ≈ 100).
- [ ] AC-6: vmpressure_calc_level(scanned=1024, reclaimed=800): returns VMPRESSURE_LOW (pressure ≈ 22).
- [ ] AC-7: vmpressure_calc_level(reclaimed >= scanned): returns VMPRESSURE_LOW (pressure = 0).
- [ ] AC-8: vmpressure_prio(prio=3): trips critical level via vmpressure(scanned=win, reclaimed=0).
- [ ] AC-9: vmpressure_prio(prio=4): no-op (above threshold).
- [ ] AC-10: vmpressure_register_event("critical"): mode = VMPRESSURE_NO_PASSTHROUGH, level = CRITICAL.
- [ ] AC-11: vmpressure_register_event("medium,hierarchy"): mode = HIERARCHY, level = MEDIUM.
- [ ] AC-12: vmpressure_register_event("low,local"): mode = LOCAL, level = LOW.
- [ ] AC-13: vmpressure_register_event(bad-level): -EINVAL.
- [ ] AC-14: vmpressure_register_event with vmpressure_str_modes mismatch: -EINVAL.
- [ ] AC-15: vmpressure_work_fn: hierarchical walk delivers to each ancestor.
- [ ] AC-16: vmpressure_event with mode=LOCAL on ancestor walk: skipped.
- [ ] AC-17: vmpressure_event with mode=NO_PASSTHROUGH after a signalled ancestor: skipped.
- [ ] AC-18: vmpressure_cleanup: flush_work returns only after pending work completes.
- [ ] AC-19: non-tree path (tree=false) with level > LOW: mem_cgroup_set_socket_pressure invoked.

## Architecture

```
struct Vmpressure {                       // embedded in MemCgroup
  sr_lock: Spinlock,
  scanned: u64,
  reclaimed: u64,
  tree_scanned: u64,
  tree_reclaimed: u64,
  events_lock: Mutex,
  events: ListHead<VmpressureEvent>,
  work: WorkStruct,                       // dispatches Vmpressure::work_fn
}

struct VmpressureEvent {
  efd: *EventfdCtx,
  level: VmpressureLevel,                 // LOW / MEDIUM / CRITICAL
  mode: VmpressureMode,                   // NO_PASSTHROUGH / HIERARCHY / LOCAL
  node: ListHead,
}

enum VmpressureLevel { Low = 0, Medium = 1, Critical = 2 }
enum VmpressureMode  { NoPassthrough = 0, Hierarchy = 1, Local = 2 }

const VMPRESSURE_WIN: u64 = SWAP_CLUSTER_MAX * 16;       // 512 pages
const VMPRESSURE_LEVEL_MED: u32 = 60;
const VMPRESSURE_LEVEL_CRITICAL: u32 = 95;
const VMPRESSURE_LEVEL_CRITICAL_PRIO: u32 = 3;           // ilog2(10)
```

`Vmpressure::account(gfp, memcg, tree, scanned, reclaimed)`:
1. if mem_cgroup_disabled(): return.
2. if !cgroup_subsys_on_dfl(memory) ∧ !tree: return.
3. vmpr = memcg_to_vmpressure(memcg).
4. if !(gfp & (__GFP_HIGHMEM | __GFP_MOVABLE | __GFP_IO | __GFP_FS)): return.
5. if !scanned: return.
6. if tree:
   - spin_lock(&vmpr.sr_lock).
   - scanned = vmpr.tree_scanned += scanned.
   - vmpr.tree_reclaimed += reclaimed.
   - spin_unlock.
   - if scanned < VMPRESSURE_WIN: return.
   - schedule_work(&vmpr.work).
7. else:
   - if !memcg ∨ mem_cgroup_is_root(memcg): return.
   - spin_lock(&vmpr.sr_lock).
   - scanned = vmpr.scanned += scanned.
   - reclaimed = vmpr.reclaimed += reclaimed.
   - if scanned < VMPRESSURE_WIN: spin_unlock; return.
   - vmpr.scanned = 0; vmpr.reclaimed = 0.
   - spin_unlock.
   - level = Vmpressure::calc_level(scanned, reclaimed).
   - if level > Low: mem_cgroup_set_socket_pressure(memcg).

`Vmpressure::account_prio(gfp, memcg, prio)`:
1. if prio > VMPRESSURE_LEVEL_CRITICAL_PRIO: return.
2. Vmpressure::account(gfp, memcg, /*tree=*/true, VMPRESSURE_WIN, 0).

`Vmpressure::calc_level(scanned, reclaimed) -> VmpressureLevel`:
1. scale = scanned + reclaimed.
2. pressure = 0.
3. if reclaimed >= scanned: goto out.
4. pressure = scale - reclaimed * scale / scanned.
5. pressure = pressure * 100 / scale.
6. out: return Vmpressure::level_for(pressure).

`Vmpressure::level_for(pressure) -> VmpressureLevel`:
1. if pressure >= VMPRESSURE_LEVEL_CRITICAL: return Critical.
2. if pressure >= VMPRESSURE_LEVEL_MED: return Medium.
3. return Low.

`Vmpressure::work_fn(work)`:
1. vmpr = container_of(work, Vmpressure, work).
2. spin_lock(&vmpr.sr_lock).
3. scanned = vmpr.tree_scanned.
4. if !scanned: spin_unlock; return.
5. reclaimed = vmpr.tree_reclaimed.
6. vmpr.tree_scanned = 0; vmpr.tree_reclaimed = 0.
7. spin_unlock.
8. level = Vmpressure::calc_level(scanned, reclaimed).
9. ancestor = false; signalled = false.
10. loop:
    - if Vmpressure::deliver(vmpr, level, ancestor, signalled): signalled = true.
    - ancestor = true.
    - vmpr = Vmpressure::parent(vmpr).
    - if !vmpr: break.

`Vmpressure::deliver(vmpr, level, ancestor, signalled) -> bool`:
1. ret = false.
2. mutex_lock(&vmpr.events_lock).
3. for ev in vmpr.events:
   - if ancestor ∧ ev.mode == Local: continue.
   - if signalled ∧ ev.mode == NoPassthrough: continue.
   - if level < ev.level: continue.
   - eventfd_signal(ev.efd).
   - ret = true.
4. mutex_unlock.
5. return ret.

`Vmpressure::parent(vmpr) -> Option<*Vmpressure>`:
1. memcg = vmpressure_to_memcg(vmpr).
2. memcg = parent_mem_cgroup(memcg).
3. if !memcg: return None.
4. return Some(memcg_to_vmpressure(memcg)).

`Vmpressure::register_event(memcg, eventfd, args) -> Result<(), i32>`:
1. vmpr = memcg_to_vmpressure(memcg).
2. mode = NoPassthrough.
3. spec = kstrndup(args, MAX_VMPRESSURE_ARGS_LEN, GFP_KERNEL).
4. if !spec: return Err(-ENOMEM).
5. token = strsep(&spec, ",").
6. r = match_string(LEVEL_NAMES, VMPRESSURE_NUM_LEVELS, token).
7. if r < 0: kfree(spec_orig); return Err(-EINVAL).
8. level = r.
9. token = strsep(&spec, ",").
10. if token:
    - r = match_string(MODE_NAMES, VMPRESSURE_NUM_MODES, token).
    - if r < 0: kfree(spec_orig); return Err(-EINVAL).
    - mode = r.
11. ev = kzalloc(VmpressureEvent, GFP_KERNEL).
12. if !ev: kfree(spec_orig); return Err(-ENOMEM).
13. ev.efd = eventfd; ev.level = level; ev.mode = mode.
14. mutex_lock(&vmpr.events_lock).
15. list_add(&ev.node, &vmpr.events).
16. mutex_unlock.
17. kfree(spec_orig).
18. return Ok(()).

`Vmpressure::unregister_event(memcg, eventfd)`:
1. vmpr = memcg_to_vmpressure(memcg).
2. mutex_lock(&vmpr.events_lock).
3. for ev in vmpr.events:
   - if ev.efd != eventfd: continue.
   - list_del(&ev.node).
   - kfree(ev).
   - break.
4. mutex_unlock.

`Vmpressure::init(vmpr)`:
1. spin_lock_init(&vmpr.sr_lock).
2. mutex_init(&vmpr.events_lock).
3. INIT_LIST_HEAD(&vmpr.events).
4. INIT_WORK(&vmpr.work, Vmpressure::work_fn).

`Vmpressure::cleanup(vmpr)`:
1. flush_work(&vmpr.work).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `calc_level_ratio_bounded` | INVARIANT | per-calc_level: pressure ∈ [0, 100]; level ∈ {Low, Medium, Critical}. |
| `reclaimed_ge_scanned_yields_low` | INVARIANT | per-calc_level: reclaimed ≥ scanned ⟹ pressure = 0 ⟹ Low. |
| `accumulator_window_atomic` | INVARIANT | per-account: tree_scanned/tree_reclaimed reset only after work scheduled. |
| `sr_lock_held_for_accumulator` | INVARIANT | per-account ∧ work_fn: every read/write of (tree_)scanned/reclaimed occurs under sr_lock. |
| `events_list_under_mutex` | INVARIANT | per-deliver/register/unregister: events_lock held while traversing/mutating list. |
| `register_event_mode_default_no_passthrough` | INVARIANT | per-register_event: missing mode ⟹ mode = NoPassthrough. |
| `local_mode_suppresses_ancestor_delivery` | INVARIANT | per-deliver: ancestor=true ∧ ev.mode=Local ⟹ no eventfd_signal. |
| `nopassthrough_mode_stops_after_signal` | INVARIANT | per-deliver: signalled=true ∧ ev.mode=NoPassthrough ⟹ no eventfd_signal. |
| `prio_threshold_critical_only` | INVARIANT | per-account_prio: prio > VMPRESSURE_LEVEL_CRITICAL_PRIO ⟹ no-op. |
| `cleanup_flushes_pending_work` | INVARIANT | per-cleanup: returns only after vmpr.work has completed. |
| `gfp_filter_user_actionable` | INVARIANT | per-account: gfp without HIGHMEM/MOVABLE/IO/FS ⟹ no-op. |

### Layer 2: TLA+

`mm/vmpressure.tla`:
- Per-account + per-window-overflow + per-work + per-deliver + per-hierarchy-walk + per-register/unregister.
- Properties:
  - `safety_window_overflow_only_schedules_once` — per-tree-account: only the call that crosses VMPRESSURE_WIN boundary schedules work.
  - `safety_calc_level_monotone` — per-pressure: higher pressure ⟹ ≥ level.
  - `safety_hierarchy_walk_terminates` — per-work_fn: parent chain finite (root memcg has no parent).
  - `safety_local_event_isolated_to_own_memcg` — per-Local-mode event: only fires when ancestor=false.
  - `safety_no_passthrough_one_eventfd_per_walk` — per-NoPassthrough-mode event: at most one ancestor signals.
  - `liveness_per_window_eventually_delivers` — per-window-overflow: work_fn eventually runs, eventfd_signal called for matching events.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vmpressure::calc_level` post: returns Low ∣ Medium ∣ Critical | `Vmpressure::calc_level` |
| `Vmpressure::account` post: tree=true ∧ accum ≥ WIN ⟹ work scheduled | `Vmpressure::account` |
| `Vmpressure::account_prio` post: prio ≤ 3 ⟹ critical-equivalent account call | `Vmpressure::account_prio` |
| `Vmpressure::work_fn` post: scanned/reclaimed accumulators reset to 0 | `Vmpressure::work_fn` |
| `Vmpressure::deliver` post: ret ⟺ ≥ 1 eventfd_signal emitted | `Vmpressure::deliver` |
| `Vmpressure::register_event` post: ev appended under events_lock with parsed (level, mode) | `Vmpressure::register_event` |
| `Vmpressure::unregister_event` post: matching ev removed and freed | `Vmpressure::unregister_event` |
| `Vmpressure::cleanup` post: flush_work returned ⟺ work idle | `Vmpressure::cleanup` |

### Layer 4: Verus/Creusot functional

`Per-vmscan reclaim → vmpressure(scanned, reclaimed) → accumulator overflow → schedule_work → vmpressure_work_fn → calc_level → hierarchy walk → eventfd_signal` semantic equivalence: per-Documentation/admin-guide/cgroup-v1/memory.rst § "memory.pressure_level".

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

vmpressure reinforcement:

- **Per-sr_lock spinlock around accumulators** — defense against per-concurrent-vmscan torn-counters across CPUs.
- **Per-events_lock mutex around event list** — defense against per-register-vs-deliver iterator corruption.
- **Per-gfp filter (HIGHMEM | MOVABLE | IO | FS)** — defense against per-DMA-zone noise reaching userspace daemons.
- **Per-`!scanned` short-circuit** — defense against per-divide-by-zero in calc_level.
- **Per-`reclaimed >= scanned` guard** — defense against per-slab-shrinker negative-pressure underflow.
- **Per-VMPRESSURE_WIN rate-limit** — defense against per-eventfd flood (eventfd-storm DoS).
- **Per-root-memcg skip in non-tree path** — defense against per-root-attribution accounting noise.
- **Per-mode-LOCAL ancestor-suppression** — defense against per-leaf-pressure ancestor-cgroup mis-attribution.
- **Per-mode-NO_PASSTHROUGH single-handler-per-walk** — defense against per-duplicate-eventfd legacy semantic.
- **Per-flush_work in cleanup** — defense against per-memcg-destroy UAF of vmpr.work.
- **Per-MAX_VMPRESSURE_ARGS_LEN bound on args copy** — defense against per-unbounded-userspace-string kmalloc.
- **Per-match_string strict allowlist for level/mode** — defense against per-typo silent-default; -EINVAL fast.
- **Per-cgroup_subsys_on_dfl gate for legacy non-tree** — defense against per-v1/v2 mode confusion.
- **Per-mem_cgroup_disabled fast-path** — defense against per-CONFIG_MEMCG=n hot-path overhead.
- **Per-mem_cgroup_set_socket_pressure hysteresis (1 s)** — defense against per-socket-buffer thrash.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — vmpressure eventfd notify path copies only fixed-size u64 counters out; the args buffer copied in from userspace is bounded by MAX_VMPRESSURE_ARGS_LEN and validated before any per-CPU access.
- **PAX_KERNEXEC** — vmpressure work_struct handlers and eventfd notify callbacks execute from RX text; no JIT or writable text is registered into the event list.
- **PAX_RANDKSTACK** — cgroup write handler for vmpressure entry honors randomized kernel-stack offset; the args parse uses a heap copy rather than a VLA on stack.
- **PAX_REFCOUNT** — vmpressure_event and memcg references use hardened refcounts; eventfd_ctx_fileget/put pairing is enforced by saturating counters.
- **PAX_MEMORY_SANITIZE** — freed vmpressure_event structs are sanitized before kfree so leftover eventfd_ctx pointers cannot be re-observed via slab recycling.
- **PAX_UDEREF** — args copy uses copy_from_user with explicit length cap; no user pointer is dereferenced from softirq or workqueue context.
- **PAX_RAP / kCFI** — vmpressure_work_fn and eventfd_signal callbacks are CFI-typed; the event list cannot be hijacked into pivoting through an attacker-controlled function pointer.
- **GRKERNSEC_HIDESYM** — cgroup vmpressure interface files do not leak kernel pointers; per-memcg state is shown only to writers in the owning cgroup namespace.
- **GRKERNSEC_DMESG** — vmpressure-related WARN_ONs and event-register failures gate behind dmesg_restrict.
- **Per-cgroup CAP_SYS_ADMIN gate for legacy non-tree mode** — defense against per-v1-legacy-mode unprivileged event-source manipulation.
- **Per-eventfd notify PAX_USERCOPY whitelist** — defense against per-arbitrary-kernel-state being copied to the eventfd reader.
- **Per-events_lock mutex held during register/unregister** — defense against per-iterator-vs-deliver UAF.
- **Per-MAX_VMPRESSURE_ARGS_LEN strict cap on args** — defense against per-unbounded kmalloc from a userspace write.
- **Per-mem_cgroup_disabled fast-exit** — defense against per-CONFIG_MEMCG=n hot-path cost and grsec-audit noise.

Rationale: vmpressure is an unprivileged notification surface that a userspace daemon subscribes to; under grsec discipline the per-cgroup args path needs USERCOPY/UDEREF safety, the event list needs RAP-typed dispatch, the eventfd notify lifetime is governed by hardened refcounts, and HIDESYM/DMESG ensure that pressure-event timing cannot be amplified into a layout-disclosure side-channel.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/memcontrol.c memcg lifecycle (covered in `memcg.md` Tier-3)
- mm/vmscan.c reclaim callers of vmpressure / vmpressure_prio (covered in `reclaim.md` Tier-3)
- fs/eventfd.c eventfd plumbing (covered separately if expanded)
- mm/oom_kill.c (covered in `oom-kill.md` Tier-3)
- net/core/sock.c socket-pressure consumer (covered separately if expanded)
- Implementation code
