---
title: "Tier-3: kernel/tracepoint.c — Tracepoint subsystem (static call + SRCU dispatch)"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

**Tracepoints** are zero-cost-when-disabled per-named-instrumentation hooks emitted at compile time (`TRACE_EVENT(...)` / `DEFINE_TRACE(...)`) into kernel source. When no probe is attached, a single `static_branch` (jump-label) keeps the call site as a NOP. When one probe is attached, a `static_call` site dispatches directly to that probe (one indirect-call elided). When ≥ 2 probes are attached, a generic iterator walks the per-tp `funcs[]` array. The probe array is RCU-managed; per-tp updates run under `tracepoints_mutex` and free old arrays via `call_srcu(&tracepoint_srcu, ...)` for non-faultable tps or `call_rcu_tasks_trace(...)` for faultable (sleepable) tps. Module tracepoints are tracked on a separate `tracepoint_module_list` walked by per-coming/going notifiers (`tracepoint_module_notify`). Subsystems (ftrace, perf, BPF, kprobes) consume tracepoints via `for_each_kernel_tracepoint` / `for_each_module_tracepoint` and register probes via `tracepoint_probe_register{,_prio}` / `_unregister`. Per-`CONFIG_HAVE_SYSCALL_TRACEPOINTS`, an extra `syscall_regfunc` / `_unregfunc` toggles `SYSCALL_TRACEPOINT` work-flag on every task to enable syscall enter/exit dispatch. Critical for: ftrace / perf / BPF observability, zero overhead when off, deterministic transition between 0/1/N probe states.

This Tier-3 covers `kernel/tracepoint.c` (~793 lines).

### Acceptance Criteria

- [ ] AC-1: Register first probe: static_branch flipped enabled; static_call patched to probe; static_key_enabled(&tp.key) == true.
- [ ] AC-2: Register second probe: static_call patched to iterator; tp.funcs has 2 entries sorted by descending prio.
- [ ] AC-3: Register identical {probe, data}: returns -EEXIST.
- [ ] AC-4: ENOMEM in func_add: returns -ENOMEM; tp.funcs unchanged; first-probe regfunc undone via unregfunc.
- [ ] AC-5: Unregister last probe: static_branch disabled; tp.funcs == NULL; tp.ext.unregfunc invoked once.
- [ ] AC-6: Unregister with ENOMEM in func_remove: probe slot replaced with tp_stub_func via WRITE_ONCE; tp.funcs unchanged.
- [ ] AC-7: Probe priority: higher-prio probe placed before lower-prio in funcs[]; iterator calls in order.
- [ ] AC-8: Faultable tp: release_probes uses call_rcu_tasks_trace; non-faultable: call_srcu(&tracepoint_srcu, ...).
- [ ] AC-9: 1→0→1 transition: tp_rcu_get_state at 1→0 + tp_rcu_cond_sync at 0→1 guarantee no reader holds stale single-probe data.
- [ ] AC-10: N→2→1 transition with data change: tp_rcu_get_state + tp_rcu_cond_sync drain iterator readers before single-probe dispatch.
- [ ] AC-11: Module COMING with bad taint: skip; not added to tracepoint_module_list.
- [ ] AC-12: Module GOING with probes still attached: WARN_ON via tp_module_going_check_quiescent.
- [ ] AC-13: register_tracepoint_module_notifier: replays MODULE_STATE_COMING for all already-loaded modules.
- [ ] AC-14: First syscall-tp probe: every task has SYSCALL_TRACEPOINT work-flag set.
- [ ] AC-15: Last syscall-tp probe removed: every task has SYSCALL_TRACEPOINT cleared.
- [ ] AC-16: for_each_kernel_tracepoint visits exactly __stop___tracepoints_ptrs - __start___tracepoints_ptrs entries.

### Architecture

```
struct Tracepoint {
  name: &'static str,
  key: StaticKey,                              // jump-label NOP / call gate
  static_call_key: *StaticCallKey,             // 1-probe fast path
  static_call_tramp: *StaticCallTramp,
  iterator: *const (),                         // N-probe fallback dispatcher
  funcs: AtomicPtr<TracepointFunc>,            // RCU-managed array
  ext: Option<&'static TracepointExt>,         // {regfunc, unregfunc}
}

struct TracepointFunc {
  func: *const (),
  data: *mut (),
  prio: i32,
}

struct TpProbes {
  rcu: RcuHead,
  probes: [TracepointFunc; _],                 // flexible array
}

enum TpFuncState { F0, F1, F2, FN }
enum TpTransitionSync { S1_0_1, SN_2_1, _NR }

struct TpTransitionSnapshot {
  rcu: u64,                                    // get_state_synchronize_rcu cookie
  srcu_gp: u64,                                // start_poll_synchronize_srcu cookie
  ongoing: bool,
}

// Globals
const TRACEPOINTS_MUTEX: Mutex<()>;
const TRACEPOINT_SRCU: SrcuFast;
const TP_TRANSITION_SNAPSHOT: [TpTransitionSnapshot; _NR]; // under TRACEPOINTS_MUTEX

#[cfg(modules)]
const TRACEPOINT_MODULE_LIST_MUTEX: Mutex<()>;
#[cfg(modules)]
const TRACEPOINT_MODULE_LIST: ListHead<TpModule>;
#[cfg(modules)]
const TRACEPOINT_NOTIFY_LIST: BlockingNotifierHead;

#[cfg(have_syscall_tracepoints)]
static SYS_TRACEPOINT_REFCOUNT: i32;             // under TRACEPOINTS_MUTEX
```

`Tracepoint::probe_register_prio(tp, probe, data, prio) -> Result<(), Errno>`:
1. mutex_lock(TRACEPOINTS_MUTEX).
2. tp_func = TracepointFunc { func: probe, data, prio }.
3. ret = Tracepoint::add_func(tp, &tp_func, prio, warn=true).
4. mutex_unlock(TRACEPOINTS_MUTEX).
5. return ret.

`Tracepoint::add_func(tp, func, prio, warn) -> Result<(), Errno>`:
1. /* First-probe regfunc */
2. if tp.ext.regfunc.is_some() && !static_key_enabled(&tp.key):
   - tp.ext.regfunc()?; /* propagate error */
3. tp_funcs = rcu_dereference_protected(tp.funcs, lockdep_is_held(TRACEPOINTS_MUTEX)).
4. old = Tracepoint::func_add(&mut tp_funcs, func, prio).
5. if let Err(e) = old:
   - if first-probe { tp.ext.unregfunc(); }
   - if warn && e != ENOMEM { WARN_ON_ONCE(); }
   - return Err(e).
6. /* State-transition dispatch */
7. match nr_func_state(tp_funcs):
   - F1 /* 0→1 */:
     - Tracepoint::rcu_cond_sync(S1_0_1).
     - Tracepoint::update_call(tp, tp_funcs).
     - rcu_assign_pointer(tp.funcs, tp_funcs).
     - static_branch_enable(&tp.key).
   - F2 /* 1→2 */:
     - Tracepoint::update_call(tp, tp_funcs). /* Switch to iterator */
     - /* fallthrough */ rcu_assign_pointer(tp.funcs, tp_funcs).
     - if tp_funcs[0].data != old[0].data: Tracepoint::rcu_get_state(SN_2_1).
   - FN /* N→N+1 */:
     - rcu_assign_pointer(tp.funcs, tp_funcs).
     - if tp_funcs[0].data != old[0].data: Tracepoint::rcu_get_state(SN_2_1).
   - F0: WARN_ON_ONCE.
8. Tracepoint::release_probes(tp, old).
9. return Ok(()).

`Tracepoint::remove_func(tp, func) -> Result<(), Errno>`:
1. tp_funcs = rcu_dereference_protected(tp.funcs, ...).
2. old = Tracepoint::func_remove(&mut tp_funcs, func).
3. if let Err(e) = old { WARN_ON_ONCE; return Err(e); }
4. if tp_funcs == old { return Ok(()); /* stub-replaced */ }
5. match nr_func_state(tp_funcs):
   - F0 /* 1→0 */:
     - if tp.ext.unregfunc.is_some() && static_key_enabled(&tp.key): tp.ext.unregfunc().
     - static_branch_disable(&tp.key).
     - Tracepoint::update_call(tp, tp_funcs).
     - rcu_assign_pointer(tp.funcs, ptr::null_mut()).
     - Tracepoint::rcu_get_state(S1_0_1).
   - F1 /* 2→1 */:
     - rcu_assign_pointer(tp.funcs, tp_funcs).
     - if tp_funcs[0].data != old[0].data: Tracepoint::rcu_get_state(SN_2_1).
     - Tracepoint::rcu_cond_sync(SN_2_1).
     - Tracepoint::update_call(tp, tp_funcs).
   - F2 / FN /* N→N-1 */:
     - rcu_assign_pointer(tp.funcs, tp_funcs).
     - if tp_funcs[0].data != old[0].data: Tracepoint::rcu_get_state(SN_2_1).
6. Tracepoint::release_probes(tp, old).
7. return Ok(()).

`Tracepoint::func_add(funcs: &mut *mut TracepointFunc, tp_func, prio) -> Result<*mut TracepointFunc, Errno>`:
1. if tp_func.func.is_null() { WARN; return Err(EINVAL); }
2. old = *funcs.
3. if !old.is_null():
   - for iter = 0; old[iter].func != null; iter++:
     - if old[iter].func == tp_stub_func: continue.
     - if old[iter].func == tp_func.func && old[iter].data == tp_func.data: return Err(EEXIST).
     - nr_probes++.
4. new = Tracepoint::allocate_probes(nr_probes + 2). /* +1 new, +1 NULL */
5. if new.is_null(): return Err(ENOMEM).
6. /* Copy preserving descending-prio order with insertion */
7. ... (see func_add(): inserts tp_func at first position where old[i].prio < prio) ...
8. new[nr_probes].func = null.
9. *funcs = new.
10. return Ok(old).

`Tracepoint::func_remove(funcs, tp_func) -> Result<*mut TracepointFunc, Errno>`:
1. old = *funcs. if old.is_null(): return Err(ENOENT).
2. Count nr_probes / nr_del where matches {.func, .data} OR stub.
3. if nr_probes - nr_del == 0: *funcs = null; return Ok(old).
4. new = Tracepoint::allocate_probes(nr_probes - nr_del + 1).
5. if !new.is_null():
   - Copy non-matching non-stub entries; new[end].func = null; *funcs = new.
6. else /* ENOMEM */:
   - for i = 0; old[i].func != null; i++:
     - if matches: WRITE_ONCE(&old[i].func, tp_stub_func).
   - *funcs = old.
7. return Ok(old).

`Tracepoint::update_call(tp, tp_funcs)`:
1. if tp.static_call_key.is_null(): return. /* Synthetic events skip */
2. func = if nr_func_state(tp_funcs) == F1 { tp_funcs[0].func } else { tp.iterator }.
3. __static_call_update(tp.static_call_key, tp.static_call_tramp, func).

`Tracepoint::release_probes(tp, old)`:
1. if old.is_null(): return.
2. tp_probes = container_of(old, TpProbes, probes[0]).
3. if tracepoint_is_faultable(tp):
   - call_rcu_tasks_trace(&tp_probes.rcu, rcu_free_old_probes).
4. else:
   - call_srcu(TRACEPOINT_SRCU, &tp_probes.rcu, rcu_free_old_probes).

`Tracepoint::rcu_get_state(sync)`:
1. snap = &mut TP_TRANSITION_SNAPSHOT[sync].
2. snap.rcu = get_state_synchronize_rcu().
3. snap.srcu_gp = start_poll_synchronize_srcu(TRACEPOINT_SRCU).
4. snap.ongoing = true.

`Tracepoint::rcu_cond_sync(sync)`:
1. snap = &mut TP_TRANSITION_SNAPSHOT[sync].
2. if !snap.ongoing: return.
3. cond_synchronize_rcu(snap.rcu).
4. if !poll_state_synchronize_srcu(TRACEPOINT_SRCU, snap.srcu_gp): synchronize_srcu(TRACEPOINT_SRCU).
5. snap.ongoing = false.

`Tracepoint::module_coming(mod) -> Result<(), Errno>`:
1. if mod.num_tracepoints == 0: return Ok(()).
2. if Tracepoint::module_has_bad_taint(mod): return Ok(()). /* Silent skip */
3. tp_mod = kmalloc_obj::<TpModule>().ok_or(ENOMEM)?.
4. tp_mod.mod = mod.
5. mutex_lock(TRACEPOINT_MODULE_LIST_MUTEX).
6. list_add_tail(&tp_mod.list, TRACEPOINT_MODULE_LIST).
7. blocking_notifier_call_chain(TRACEPOINT_NOTIFY_LIST, MODULE_STATE_COMING, tp_mod).
8. mutex_unlock.
9. return Ok(()).

`Tracepoint::module_going(mod)`:
1. if mod.num_tracepoints == 0: return.
2. mutex_lock(TRACEPOINT_MODULE_LIST_MUTEX).
3. list_for_each_entry(tp_mod, TRACEPOINT_MODULE_LIST):
   - if tp_mod.mod == mod:
     - blocking_notifier_call_chain(MODULE_STATE_GOING, tp_mod).
     - list_del(&tp_mod.list); kfree(tp_mod).
     - Tracepoint::for_each_range(mod.tracepoints_ptrs, mod.tracepoints_ptrs + mod.num_tracepoints, tp_module_going_check_quiescent, NULL).
     - break.
4. mutex_unlock.

`Tracepoint::syscall_regfunc() -> Result<(), Errno>`:
1. if SYS_TRACEPOINT_REFCOUNT == 0:
   - read_lock(&tasklist_lock).
   - for_each_process_thread(p, t): set_task_syscall_work(t, SYSCALL_TRACEPOINT).
   - read_unlock(&tasklist_lock).
2. SYS_TRACEPOINT_REFCOUNT += 1.
3. return Ok(()).

`Tracepoint::syscall_unregfunc()`:
1. SYS_TRACEPOINT_REFCOUNT -= 1.
2. if SYS_TRACEPOINT_REFCOUNT == 0:
   - read_lock(&tasklist_lock).
   - for_each_process_thread(p, t): clear_task_syscall_work(t, SYSCALL_TRACEPOINT).
   - read_unlock(&tasklist_lock).

### Out of Scope

- TRACE_EVENT macro expansion / __DO_TRACE inlining (covered by `include/trace/...` codegen in Tier-2)
- ftrace consumer (`kernel/trace/trace_events.c` covered in `events/` Tier-3)
- perf consumer (`kernel/events/core.c` covered in `events/` Tier-3)
- BPF raw-tracepoint consumer (`kernel/trace/bpf_trace.c` covered in `bpf/` Tier-3)
- Static-key / jump-label internals (covered in `kernel/jump_label.c` Tier-3)
- Static-call internals (covered in `kernel/static_call.c` / arch backend Tier-3)
- SRCU and tasks-trace RCU primitives (covered in `kernel/rcu/...` Tier-3)
- Module loader infrastructure (`kernel/module/...` covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct tracepoint` | per-tp descriptor (key, funcs, static_call_key/tramp, ext) | `Tracepoint` |
| `struct tracepoint_func` | per-probe (func, data, prio) | `TracepointFunc` |
| `struct tp_probes` | per-RCU-freed probe-array container | `TpProbes` |
| `enum tp_func_state` | per-array-size state (0/1/2/N) | `TpFuncState` |
| `enum tp_transition_sync` | per-array-shape-change sync token | `TpTransitionSync` |
| `tracepoint_srcu` | per-SRCU-fast domain for tp probes | `TRACEPOINT_SRCU` |
| `tracepoints_mutex` | per-global tp-funcs mutex | `TRACEPOINTS_MUTEX` |
| `tracepoint_module_list_mutex` | per-module-list mutex | `TRACEPOINT_MODULE_LIST_MUTEX` |
| `tracepoint_module_list` | per-list of registered module tps | `TRACEPOINT_MODULE_LIST` |
| `tracepoint_notify_list` | per-module-coming/going notifier chain | `TRACEPOINT_NOTIFY_LIST` |
| `__start___tracepoints_ptrs` / `__stop___tracepoints_ptrs` | per-vmlinux tp ptr array | `__START_TRACEPOINTS_PTRS` / `__STOP_*` |
| `tp_stub_func` | per-allocation-failure stub | `Tracepoint::stub_func` |
| `allocate_probes()` | per-alloc kmalloc_flex of tp_probes | `Tracepoint::allocate_probes` |
| `release_probes()` | per-free via SRCU or rcu_tasks_trace | `Tracepoint::release_probes` |
| `rcu_free_old_probes()` | per-RCU-cb kfree | `Tracepoint::rcu_free_old_probes` |
| `func_add()` | per-probe-array grow (N → N+1) | `Tracepoint::func_add` |
| `func_remove()` | per-probe-array shrink (N → M) | `Tracepoint::func_remove` |
| `nr_func_state()` | per-array-length classifier | `Tracepoint::nr_func_state` |
| `tracepoint_update_call()` | per-static-call patch | `Tracepoint::update_call` |
| `tracepoint_add_func()` | per-probe insertion + state transition | `Tracepoint::add_func` |
| `tracepoint_remove_func()` | per-probe removal + state transition | `Tracepoint::remove_func` |
| `tp_rcu_get_state()` / `tp_rcu_cond_sync()` | per-transition deferred-sync snapshot | `Tracepoint::rcu_get_state` / `cond_sync` |
| `tracepoint_probe_register_prio_may_exist()` | per-register-noisy-suppress | `Tracepoint::probe_register_prio_may_exist` |
| `tracepoint_probe_register_prio()` | per-register w/ prio | `Tracepoint::probe_register_prio` |
| `tracepoint_probe_register()` | per-register default prio | `Tracepoint::probe_register` |
| `tracepoint_probe_unregister()` | per-unregister | `Tracepoint::probe_unregister` |
| `for_each_tracepoint_range()` | per-section iterator | `Tracepoint::for_each_range` |
| `for_each_kernel_tracepoint()` | per-vmlinux iterator | `Tracepoint::for_each_kernel` |
| `for_each_tracepoint_in_module()` | per-module iterator | `Tracepoint::for_each_in_module` |
| `for_each_module_tracepoint()` | per-all-modules iterator | `Tracepoint::for_each_module` |
| `register_tracepoint_module_notifier()` | per-register module coming/going cb | `Tracepoint::register_module_notifier` |
| `unregister_tracepoint_module_notifier()` | per-unregister | `Tracepoint::unregister_module_notifier` |
| `tracepoint_module_coming()` | per-module-MODULE_STATE_COMING handler | `Tracepoint::module_coming` |
| `tracepoint_module_going()` | per-module-MODULE_STATE_GOING handler | `Tracepoint::module_going` |
| `tp_module_going_check_quiescent()` | per-quiescent-check on unload | `Tracepoint::module_going_check_quiescent` |
| `trace_module_has_bad_taint()` | per-taint-vet | `Tracepoint::module_has_bad_taint` |
| `tracepoint_module_notify()` | per-module-notifier-trampoline | `Tracepoint::module_notify` |
| `syscall_regfunc()` | per-first-syscall-probe enable per-task work flag | `Tracepoint::syscall_regfunc` |
| `syscall_unregfunc()` | per-last-syscall-probe disable per-task work flag | `Tracepoint::syscall_unregfunc` |
| `sys_tracepoint_refcount` | per-global syscall-tp probe refcount | `SYS_TRACEPOINT_REFCOUNT` |

### compatibility contract

REQ-1: struct tracepoint (per-tp descriptor):
- name: const &'static str — fully-qualified event name.
- key: static_key_false — gates whether the tp fires.
- static_call_key, static_call_tramp: per-static_call dispatch (1-probe optimization).
- iterator: per-N-probe iterator entry (auto-generated by TRACE_EVENT macro).
- funcs: RCU-protected pointer to `tracepoint_func[]` (NUL-terminated).
- ext: optional pointer to {regfunc, unregfunc} hooks (first/last probe).
- Per-TRACE_EVENT macro emits this descriptor into `__tracepoints_ptrs` linker section.

REQ-2: struct tracepoint_func (per-probe):
- func: probe callback (NULL terminates array; tp_stub_func placeholder).
- data: opaque per-probe cookie.
- prio: i32 — higher prio sorts earlier in funcs[] (TRACEPOINT_DEFAULT_PRIO = 10).

REQ-3: Per-tp_func_state classifier (`nr_func_state(funcs)`):
- TP_FUNC_0: funcs == NULL.
- TP_FUNC_1: funcs[0].func != NULL ∧ funcs[1].func == NULL.
- TP_FUNC_2: funcs[2].func == NULL (exactly two).
- TP_FUNC_N: ≥ 3 probes.

REQ-4: tracepoint_add_func(tp, func, prio, warn):
- Hold tracepoints_mutex.
- /* First probe: invoke regfunc */
- if tp.ext && tp.ext.regfunc && !static_key_enabled(&tp.key):
  - ret = tp.ext.regfunc(); if ret < 0: return ret.
- tp_funcs = rcu_dereference_protected(tp.funcs, lockdep_is_held(&tracepoints_mutex)).
- old = func_add(&tp_funcs, func, prio).
- if IS_ERR(old):
  - if first-probe: tp.ext.unregfunc().
  - WARN_ON_ONCE on -ENOMEM-suppress.
  - return PTR_ERR(old).
- /* State-transition dispatch */
- switch nr_func_state(tp_funcs):
  - TP_FUNC_1 (0→1):
    - tp_rcu_cond_sync(TP_TRANSITION_SYNC_1_0_1). /* Drain pending 1→0 readers */
    - tracepoint_update_call(tp, tp_funcs). /* Patch static_call to single probe */
    - rcu_assign_pointer(tp.funcs, tp_funcs).
    - static_branch_enable(&tp.key). /* Flip jump-label NOP → call */
  - TP_FUNC_2 (1→2):
    - tracepoint_update_call(tp, tp_funcs). /* Patch static_call to iterator */
    - fallthrough.
  - TP_FUNC_N (N→N+1):
    - rcu_assign_pointer(tp.funcs, tp_funcs).
    - if tp_funcs[0].data != old[0].data: tp_rcu_get_state(TP_TRANSITION_SYNC_N_2_1).
- release_probes(tp, old).
- return 0.

REQ-5: tracepoint_remove_func(tp, func):
- Hold tracepoints_mutex.
- tp_funcs = rcu_dereference_protected(tp.funcs, ...).
- old = func_remove(&tp_funcs, func).
- if IS_ERR(old): return PTR_ERR(old).
- /* Allocation-failure stub: func replaced with tp_stub_func; bail */
- if tp_funcs == old: return 0.
- switch nr_func_state(tp_funcs):
  - TP_FUNC_0 (1→0):
    - if tp.ext && tp.ext.unregfunc && static_key_enabled(&tp.key): tp.ext.unregfunc().
    - static_branch_disable(&tp.key). /* Flip back to NOP */
    - tracepoint_update_call(tp, tp_funcs). /* Reset static_call to iterator */
    - rcu_assign_pointer(tp.funcs, NULL).
    - tp_rcu_get_state(TP_TRANSITION_SYNC_1_0_1). /* Snapshot to gate next 0→1 */
  - TP_FUNC_1 (2→1):
    - rcu_assign_pointer(tp.funcs, tp_funcs).
    - if tp_funcs[0].data != old[0].data: tp_rcu_get_state(TP_TRANSITION_SYNC_N_2_1).
    - tp_rcu_cond_sync(TP_TRANSITION_SYNC_N_2_1). /* Drain N→2→1 readers */
    - tracepoint_update_call(tp, tp_funcs). /* Patch back to single-probe */
  - TP_FUNC_2 (N→N-1) / TP_FUNC_N:
    - rcu_assign_pointer(tp.funcs, tp_funcs).
    - if tp_funcs[0].data != old[0].data: tp_rcu_get_state(TP_TRANSITION_SYNC_N_2_1).
- release_probes(tp, old).
- return 0.

REQ-6: func_add(funcs, tp_func, prio):
- WARN if tp_func.func == NULL: return -EINVAL.
- old = *funcs; iterate old to count probes, reject EEXIST on identical {func, data}, skip stubs.
- Allocate new array of size (nr_probes + 2) /* +1 new, +1 NULL */ via allocate_probes.
- ENOMEM → return -ENOMEM.
- Copy old entries into new, inserting tp_func before first lower-prio probe (descending prio order).
- new[nr_probes].func = NULL.
- *funcs = new; return old.

REQ-7: func_remove(funcs, tp_func):
- old = *funcs; if NULL: return -ENOENT.
- Count matches {func == tp_func.func ∧ data == tp_func.data} OR stubs.
- if nr_probes - nr_del == 0: *funcs = NULL; return old.
- Allocate new (nr_probes - nr_del + 1); on success copy non-matching; on ENOMEM replace target's func with tp_stub_func via WRITE_ONCE (graceful degradation; tp_stub_func is no-op).

REQ-8: tracepoint_update_call(tp, tp_funcs):
- /* Synthetic events have no static_call site */
- if !tp.static_call_key: return.
- /* TP_FUNC_1: dispatch directly to probe; else iterator */
- func = (nr_func_state(tp_funcs) == TP_FUNC_1) ? tp_funcs[0].func : tp.iterator.
- __static_call_update(tp.static_call_key, tp.static_call_tramp, func). /* Patches inline jump */

REQ-9: SRCU + tasks-trace RCU release:
- struct tp_probes { rcu_head rcu; tracepoint_func probes[]; }
- release_probes(tp, old):
  - if !old: return.
  - tp_probes = container_of(old, struct tp_probes, probes[0]).
  - if tracepoint_is_faultable(tp): call_rcu_tasks_trace(&tp_probes.rcu, rcu_free_old_probes). /* Sleepable */
  - else: call_srcu(&tracepoint_srcu, &tp_probes.rcu, rcu_free_old_probes). /* Non-sleep */

REQ-10: Per-tp-transition deferred-sync (tp_transition_snapshot[_NR_TP_TRANSITION_SYNC]):
- tp_rcu_get_state(sync): snapshot.rcu = get_state_synchronize_rcu(); snapshot.srcu_gp = start_poll_synchronize_srcu(&tracepoint_srcu); snapshot.ongoing = true.
- tp_rcu_cond_sync(sync): if ongoing: cond_synchronize_rcu(snapshot.rcu) + maybe synchronize_srcu(&tracepoint_srcu) if !poll_state_synchronize_srcu(snapshot.srcu_gp); clear ongoing.
- Two sync slots:
  - TP_TRANSITION_SYNC_1_0_1: per-readers observing the single-probe static_call must drain before the new probe overwrites.
  - TP_TRANSITION_SYNC_N_2_1: per-iterator-based reader's data must drain before single-probe static_call uses different data.

REQ-11: tracepoint_probe_register{,_prio,_may_exist}:
- Take tracepoints_mutex; populate {func, data, prio}; call tracepoint_add_func with warn flag; release mutex.
- _register default prio = TRACEPOINT_DEFAULT_PRIO (10), warn = true.
- _register_prio_may_exist: warn = false (idempotent register).
- All three are EXPORT_SYMBOL_GPL.

REQ-12: tracepoint_probe_unregister:
- Take tracepoints_mutex; tracepoint_remove_func; release.
- EXPORT_SYMBOL_GPL.

REQ-13: for_each_tracepoint_range(begin, end, fct, priv):
- if !begin: return.
- for iter = begin; iter < end; iter++: fct(tracepoint_ptr_deref(iter), priv).
- /* tracepoint_ptr_deref handles CONFIG_HAVE_ARCH_PREL32_RELOCATIONS encoding */

REQ-14: for_each_kernel_tracepoint(fct, priv):
- for_each_tracepoint_range(__start___tracepoints_ptrs, __stop___tracepoints_ptrs, fct, priv).
- EXPORT_SYMBOL_GPL.

REQ-15: Per-CONFIG_MODULES module-tracepoint tracking:
- struct tp_module { list_head list; struct module *mod; }
- tracepoint_module_list_mutex nests OUTSIDE tracepoints_mutex.
- tracepoint_module_coming(mod):
  - if !mod.num_tracepoints: return 0.
  - if trace_module_has_bad_taint(mod): return 0. /* Skip modules with disallowed taints */
  - allocate tp_module; tp_mod.mod = mod; list_add_tail to tracepoint_module_list.
  - blocking_notifier_call_chain(&tracepoint_notify_list, MODULE_STATE_COMING, tp_mod).
- tracepoint_module_going(mod):
  - List-walk; on match: call notifier(MODULE_STATE_GOING); list_del; kfree; for_each_tracepoint_range(mod.tracepoints_ptrs, ..., tp_module_going_check_quiescent, NULL) /* WARN if any tp still has probes */.

REQ-16: trace_module_has_bad_taint(mod):
- return mod.taints & ~((1 << TAINT_OOT_MODULE) | (1 << TAINT_CRAP) | (1 << TAINT_UNSIGNED_MODULE) | (1 << TAINT_TEST) | (1 << TAINT_LIVEPATCH)).
- /* Allowed: out-of-tree, crap, unsigned, test, livepatch. Forbidden: forced load, proprietary, etc. */

REQ-17: register_tracepoint_module_notifier(nb) / unregister:
- Hold tracepoint_module_list_mutex.
- blocking_notifier_chain_register(&tracepoint_notify_list, nb).
- On success: replay existing modules (call nb with MODULE_STATE_COMING for each tp_mod in list).
- Unregister: replay MODULE_STATE_GOING for each tp_mod.
- EXPORT_SYMBOL_GPL both.

REQ-18: Per-CONFIG_HAVE_SYSCALL_TRACEPOINTS:
- sys_tracepoint_refcount: per-global counter under tracepoints_mutex.
- syscall_regfunc():
  - if !sys_tracepoint_refcount /* first probe */:
    - read_lock(&tasklist_lock).
    - for_each_process_thread(p, t): set_task_syscall_work(t, SYSCALL_TRACEPOINT). /* Sets TIF flag */
    - read_unlock(&tasklist_lock).
  - sys_tracepoint_refcount++.
  - return 0.
- syscall_unregfunc():
  - sys_tracepoint_refcount--.
  - if !sys_tracepoint_refcount /* last */: clear SYSCALL_TRACEPOINT work on every task.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `add_func_holds_tracepoints_mutex` | INVARIANT | per-add_func: caller holds TRACEPOINTS_MUTEX (lockdep_is_held). |
| `func_add_appends_null_terminator` | INVARIANT | per-func_add: new[nr_probes].func == NULL on success. |
| `func_add_rejects_duplicate` | INVARIANT | per-func_add: identical {.func, .data} returns EEXIST. |
| `func_remove_enomem_uses_stub` | INVARIANT | per-func_remove ENOMEM: WRITE_ONCE to tp_stub_func only on matching entries. |
| `update_call_skips_synthetic` | INVARIANT | per-update_call: static_call_key.is_null ⟹ no-op. |
| `update_call_dispatches_single` | INVARIANT | per-update_call: F1 ⟹ static_call points to tp_funcs[0].func. |
| `update_call_dispatches_iter` | INVARIANT | per-update_call: F2/FN ⟹ static_call points to tp.iterator. |
| `release_probes_faultable_uses_tasks_trace` | INVARIANT | per-release_probes: tracepoint_is_faultable ⟹ call_rcu_tasks_trace. |
| `release_probes_non_faultable_uses_srcu` | INVARIANT | per-release_probes: !faultable ⟹ call_srcu(tracepoint_srcu, ...). |
| `transition_snapshot_under_tracepoints_mutex` | INVARIANT | per-rcu_get_state / rcu_cond_sync: TP_TRANSITION_SNAPSHOT mutation gated by TRACEPOINTS_MUTEX. |
| `module_list_under_module_list_mutex` | INVARIANT | per-module-list mutation: TRACEPOINT_MODULE_LIST_MUTEX held. |
| `mutex_nesting_order` | INVARIANT | per-locking: TRACEPOINT_MODULE_LIST_MUTEX nests OUTSIDE TRACEPOINTS_MUTEX. |
| `syscall_refcount_non_negative` | INVARIANT | per-syscall_unregfunc: SYS_TRACEPOINT_REFCOUNT >= 0. |
| `prio_descending_order` | INVARIANT | per-func_add: result has descending prio (forall i: new[i].prio >= new[i+1].prio). |

### Layer 2: TLA+

`kernel/tracepoint.tla`:
- Per-state-transition between {F0, F1, F2, FN} via add_func/remove_func with concurrent readers.
- Properties:
  - `safety_no_reader_sees_uninitialized_funcs` — per-rcu_assign_pointer ordering: readers see consistent funcs[] (NULL-terminated, no torn writes).
  - `safety_static_call_matches_state` — per-transition: F1 ⟹ static_call == funcs[0].func; F≥2 ⟹ static_call == iterator.
  - `safety_static_branch_matches_state` — per-transition: F0 ⟹ branch disabled; F≥1 ⟹ branch enabled.
  - `safety_1_0_1_drain` — per-1→0→1 transition: no reader observes funcs[0] from epoch-0 after epoch-1 begins (enforced by S1_0_1 snapshot + cond_sync).
  - `safety_N_2_1_drain_with_data_change` — per-N→2→1 transition with data change in funcs[0]: S_N_2_1 snapshot + cond_sync prevents iterator-resident reader from invoking single-probe dispatch with stale data.
  - `liveness_register_eventually_visible` — per-tracepoint_probe_register: eventually all CPUs see the new probe (jump-label patched everywhere).
  - `liveness_unregister_eventually_freed` — per-tracepoint_probe_unregister: old tp_probes eventually kfree'd via call_srcu/call_rcu_tasks_trace.
  - `safety_module_going_quiescent` — per-tp_module_going_check_quiescent: tp.funcs == NULL for every tp in module before kfree.
  - `safety_syscall_refcount_monotonic_after_zero` — per-syscall_regfunc/unregfunc: ref count integrity; all-task work-flag set exactly when refcount > 0.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Tracepoint::probe_register_prio` post: success ⟹ tp.funcs ≠ NULL ∧ probe in funcs[] | `Tracepoint::probe_register_prio` |
| `Tracepoint::probe_register_prio` post: ret == EEXIST ⟹ tp.funcs unchanged | `Tracepoint::probe_register_prio` |
| `Tracepoint::probe_register_prio` post: ret == ENOMEM ⟹ tp.funcs unchanged ∧ regfunc undone if first | `Tracepoint::probe_register_prio` |
| `Tracepoint::probe_unregister` post: success ⟹ probe removed from funcs[] OR replaced with stub | `Tracepoint::probe_unregister` |
| `Tracepoint::add_func` post: 0→1 transition ⟹ static_branch_enabled ∧ static_call to probe | `Tracepoint::add_func` |
| `Tracepoint::remove_func` post: 1→0 transition ⟹ !static_branch_enabled ∧ tp.funcs == NULL | `Tracepoint::remove_func` |
| `Tracepoint::release_probes` post: tp_probes scheduled for free via srcu or rcu_tasks_trace | `Tracepoint::release_probes` |
| `Tracepoint::for_each_kernel` post: visits all entries in [__start, __stop) range | `Tracepoint::for_each_kernel` |
| `Tracepoint::module_coming` post: bad-taint ⟹ tp_mod not in list | `Tracepoint::module_coming` |
| `Tracepoint::syscall_regfunc` post: refcount == 1 ⟹ every task has SYSCALL_TRACEPOINT set | `Tracepoint::syscall_regfunc` |

### Layer 4: Verus/Creusot functional

`Per-register prio_may_exist → add_func → func_add (insert at descending-prio) → update_call (F1 static_call, F≥2 iterator) → rcu_assign_pointer → static_branch_enable → release old (srcu/tasks_trace)` semantic equivalence: per-Documentation/trace/tracepoints.rst + include/linux/tracepoint.h DECLARE_TRACE macros.

### hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

Tracepoint reinforcement:

- **Per-mutex nesting strict (module_list_mutex outside tracepoints_mutex)** — defense against per-deadlock on module unload during probe register.
- **Per-rcu_assign_pointer for funcs[] update** — defense against per-reader-sees-uninitialized-array via release barrier.
- **Per-NULL-terminated probes[] array** — defense against per-reader-runs-off-end iteration.
- **Per-static_branch gate** — defense against per-disabled-tracepoint cost when no probes (zero-cost-when-off).
- **Per-static_call patch for F1** — defense against per-indirect-call cost / spectre v2 mitigation for single-probe path.
- **Per-1_0_1 deferred-sync** — defense against per-reader-runs-old-static_call-with-new-data.
- **Per-N_2_1 deferred-sync on data-change** — defense against per-iterator-reader-feeds-single-probe-stale-data.
- **Per-SRCU domain (tracepoint_srcu)** — defense against per-non-sleepable-probe-grace-period-stall.
- **Per-call_rcu_tasks_trace for faultable** — defense against per-sleepable-probe being freed mid-execution.
- **Per-tp_stub_func as ENOMEM-fallback** — defense against per-leak when removal allocation fails (graceful degradation).
- **Per-module bad-taint vet** — defense against per-forced-load-module corrupting tp descriptors.
- **Per-tp_module_going_check_quiescent WARN** — defense against per-module-unload-while-probes-still-attached UAF.
- **Per-syscall_regfunc tasklist_lock read-side** — defense against per-task-list-mutation during work-flag set/clear.
- **Per-EXPORT_SYMBOL_GPL discipline** — defense against per-non-GPL-module abusing tracepoint infrastructure.

