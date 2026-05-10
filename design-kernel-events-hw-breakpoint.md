---
title: "Tier-3: kernel/events/hw_breakpoint.c — Hardware-breakpoint subsystem"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The kernel arbitrates a small fixed number of CPU debug-register slots (x86: 4 = DR0..DR3 controlled by DR7; arm64: 2..16 BPs+WPs) between perf events (`PERF_TYPE_BREAKPOINT`), ptrace (`PTRACE_POKEUSR` on `user_hwdebug_state`), kgdb, and in-kernel users (`register_wide_hw_breakpoint`). The arch-independent core in `kernel/events/hw_breakpoint.c` decides whether a `struct perf_event` with `attr.type == PERF_TYPE_BREAKPOINT` can be admitted, given (a) the per-CPU pinned counter slots already occupied for the same `bp_type_idx` and (b) the maximum-over-tasks of per-task pinned counters via a `struct bp_slots_histogram` (histogram of slot-occupancy counts). Per-`enum bp_type_idx` = `TYPE_INST` (instruction = exec) and `TYPE_DATA` (read/write). Per-`HW_BREAKPOINT_R/W/RW/X/EMPTY` selects which arch debug-register mode is requested. Per-task constraints use `task_bps_ht` (rhltable hashed by `hw_perf_event.target`); per-CPU constraints use `bp_cpuinfo` (cpu_pinned + tsk_pinned histogram). Counting is done under a percpu rwsem `bp_cpuinfo_sem` (read-side for atomic histogram updates, write-side for stable snapshots and CPU-pinned writes). Critical for: gdbserver / lldb hardware watchpoints via ptrace, kgdb HW breakpoints (lockless `dbg_reserve/release_bp_slot`), in-kernel watchpoints (KASAN-HW-tags / KGDB / pmcraid memcheck), perf samples on data access.

This Tier-3 covers `kernel/events/hw_breakpoint.c` (~1025 lines).

### Acceptance Criteria

- [ ] AC-1: register_perf_hw_breakpoint on attr.bp_type = HW_BREAKPOINT_W with bp_addr in user space succeeds when slots are free; returns 0; bp.hw.info populated.
- [ ] AC-2: Over-subscription: reserving more than hw_breakpoint_slots_cached(TYPE_DATA) data watchpoints on one CPU returns -ENOSPC.
- [ ] AC-3: HW_BREAKPOINT_EMPTY or HW_BREAKPOINT_INVALID rejected with -EINVAL at __reserve_bp_slot.
- [ ] AC-4: Kernel-space bp_addr without CAP_SYS_ADMIN returns -EPERM via hw_breakpoint_parse.
- [ ] AC-5: attr.exclude_kernel inconsistent with kernel-space bp_addr returns -EINVAL.
- [ ] AC-6: register_wide_hw_breakpoint installs the BP on every online CPU; failure on one CPU unregisters all.
- [ ] AC-7: modify_user_hw_breakpoint changing bp_type updates per-type bookkeeping (release old + reserve new); rolls back on failure.
- [ ] AC-8: Per-task slot counting: two events on same task pinned to all CPUs (cpu=-1) bump the global tsk_pinned_all histogram from bucket k → k+1.
- [ ] AC-9: Per-CPU pinning of a task BP transitions counts from tsk_pinned_all → per-CPU tsk_pinned (Case 2.a fan-out).
- [ ] AC-10: dbg_reserve_bp_slot succeeds when constraints unlocked and is a no-lock fast path; returns -1 when bp_constraints_is_locked.
- [ ] AC-11: hw_breakpoint_is_used returns true iff any cpu_pinned or any tsk_pinned bucket is non-zero.
- [ ] AC-12: PMU.add path calls arch_install_hw_breakpoint; PMU.del calls arch_uninstall_hw_breakpoint.
- [ ] AC-13: Die-notifier hw_breakpoint_exceptions_nb registered at priority 0x7fffffff and called before generic kprobe/kgdb handlers.
- [ ] AC-14: ptrace inferior holds at most hw_breakpoint_slots_cached(TYPE_DATA) data watchpoints + hw_breakpoint_slots_cached(TYPE_INST) instruction breakpoints; quota enforced via reserve_bp_slot.

### Architecture

```
enum BpTypeIdx {
  TypeInst = 0,
  TypeData = if cfg(ARCH_SEPARATE_INST_DATA) { 1 } else { 0 },
}

struct BpSlotsHistogram {
  count: [AtomicI32; hw_breakpoint_slots(0)],  // or kzalloc'd if !static
}

struct BpCpuInfo {
  cpu_pinned: u32,
  tsk_pinned: BpSlotsHistogram,
}

struct HwBreakpoint {
  // per-CPU
  cpu_info: PerCpu<[BpCpuInfo; TYPE_MAX]>,
  // global
  cpu_pinned_global: [BpSlotsHistogram; TYPE_MAX],
  tsk_pinned_all:    [BpSlotsHistogram; TYPE_MAX],
  // task hashtable
  task_bps_ht: RhlTable,
  // locks
  cpuinfo_sem: PercpuRwSemaphore,
  // boot
  constraints_initialized: bool,            // __ro_after_init
  // PMU
  pmu: Pmu {
    task_ctx_nr: perf_sw_context,
    event_init: HwBreakpoint::event_init,
    add: HwBreakpoint::add,
    del: HwBreakpoint::del,
    start: HwBreakpoint::start,
    stop:  HwBreakpoint::stop,
    read:  HwBreakpoint::pmu_read,
  },
  exceptions_nb: NotifierBlock {
    notifier_call: hw_breakpoint_exceptions_notify, // arch
    priority: 0x7fffffff,
  },
}
```

`HwBreakpoint::reserve_slot(bp) -> Result<()>`:
1. mtx = bp_constraints_lock(bp).
2. ret = __reserve_bp_slot(bp, bp.attr.bp_type).
3. bp_constraints_unlock(mtx).
4. ret.

`HwBreakpoint::__reserve_bp_slot(bp, bp_type) -> Result<()>`:
1. if !constraints_initialized: return Err(-ENOMEM).
2. if bp_type == HW_BREAKPOINT_EMPTY ∨ bp_type == HW_BREAKPOINT_INVALID: return Err(-EINVAL).
3. type = find_slot_idx(bp_type).
4. weight = hw_breakpoint_weight(bp).
5. max_pinned = max_bp_pinned_slots(bp, type) + weight.
6. if max_pinned > hw_breakpoint_slots_cached(type): return Err(-ENOSPC).
7. toggle_bp_slot(bp, true, type, weight).

`HwBreakpoint::toggle_slot(bp, enable, type, weight) -> Result<()>`:
1. if !enable: weight = -weight.
2. /* CPU-pinned path */
3. if bp.hw.target.is_none():
   - info = cpu_info[bp.cpu][type].
   - lockdep_assert_held_write(cpuinfo_sem).
   - cpu_pinned_global[type].add(info.cpu_pinned, weight).
   - info.cpu_pinned += weight.
   - return Ok.
4. /* Task-pinned path */
5. lockdep_assert_held_read(cpuinfo_sem).
6. if !enable: rhltable_remove(task_bps_ht, &bp.hw.bp_list).
7. next_tsk_pinned = task_bp_pinned(-1, bp, type).
8. /* Per-Case routing per REQ-13 */
9. update tsk_pinned histograms.
10. if enable: rhltable_insert(task_bps_ht, &bp.hw.bp_list).

`HwBreakpoint::register_perf(bp) -> Result<()>`:
1. hw = ArchHwBreakpoint::zero.
2. reserve_slot(bp)?
3. hw_breakpoint_parse(bp, &bp.attr, &hw) (else release_slot + return err).
4. bp.hw.info = hw.
5. Ok.

`HwBreakpoint::event_init(bp) -> Result<()>` (PMU.event_init):
1. if bp.attr.type != PERF_TYPE_BREAKPOINT: return Err(-ENOENT).
2. if !slots_cached(find_slot_idx(bp.attr.bp_type)) ∨ has_branch_stack(bp): return Err(-EOPNOTSUPP).
3. register_perf(bp)?
4. bp.destroy = HwBreakpoint::destroy (release_slot).
5. Ok.

`HwBreakpoint::register_wide(attr, triggered, ctx) -> Result<*PerCpu<*PerfEvent>>`:
1. cpu_events = alloc_percpu.
2. cpus_read_lock.
3. for cpu in for_each_online_cpu():
   - bp = perf_event_create_kernel_counter(attr, cpu, None, triggered, ctx).
   - if Err: rollback.
   - cpu_events[cpu] = bp.
4. cpus_read_unlock.
5. Ok(cpu_events).

`HwBreakpoint::modify_user(bp, attr)`:
1. if irqs_disabled ∧ bp.ctx.task == current: perf_event_disable_local(bp).
2. else: perf_event_disable(bp).
3. err = modify_user_check(bp, attr, false).
4. if !bp.attr.disabled: perf_event_enable(bp).

`HwBreakpoint::init() -> Result<()>` (boot):
1. rhltable_init(task_bps_ht, params)?
2. init_breakpoint_slots()? /* allocate dynamic histograms if !static */
3. constraints_initialized = true.
4. perf_pmu_register(pmu, "breakpoint", PERF_TYPE_BREAKPOINT).
5. register_die_notifier(exceptions_nb).

### Out of Scope

- `arch/x86/kernel/hw_breakpoint.c` arch glue (DR0..DR3, DR6 status decode, encode_dr7) (covered separately if expanded)
- `arch/arm64/kernel/hw_breakpoint.c` arch glue (DBGBVRn_EL1 / DBGBCRn_EL1) (covered separately if expanded)
- `kernel/debug/gdbstub.c` kgdb HW-BP integration (covered in kgdb Tier-3 if expanded)
- `arch/x86/kernel/ptrace.c` ptrace HW-BP register exposure (covered in ptrace Tier-3 if expanded)
- `kernel/events/core.c` perf-event lifecycle (covered in `perf-event-core.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `enum bp_type_idx` (TYPE_INST, TYPE_DATA, TYPE_MAX) | per-type bucket | `BpTypeIdx` |
| `struct bp_cpuinfo` | per-CPU pinned + tsk-pinned histogram | `BpCpuInfo` |
| `struct bp_slots_histogram` | count[N] = number of slots with exactly N+1 pinned task BPs | `BpSlotsHistogram` |
| `struct arch_hw_breakpoint` | per-arch encoded DR contents | `ArchHwBreakpoint` (arch crate) |
| `register_perf_hw_breakpoint(bp)` | per-PMU event_init helper | `HwBreakpoint::register_perf` |
| `register_user_hw_breakpoint(attr, triggered, ctx, tsk)` | per-task in-kernel API | `HwBreakpoint::register_user` |
| `modify_user_hw_breakpoint(bp, attr)` | per-event modify | `HwBreakpoint::modify_user` |
| `modify_user_hw_breakpoint_check(bp, attr, check)` | per-event modify (no perf disable) | `HwBreakpoint::modify_user_check` |
| `unregister_hw_breakpoint(bp)` | per-event teardown | `HwBreakpoint::unregister` |
| `register_wide_hw_breakpoint(attr, triggered, ctx)` | per-CPU broadcast install | `HwBreakpoint::register_wide` |
| `unregister_wide_hw_breakpoint(cpu_events)` | per-CPU broadcast teardown | `HwBreakpoint::unregister_wide` |
| `reserve_bp_slot(bp)` / `__reserve_bp_slot(bp, bp_type)` | per-event admission | `HwBreakpoint::reserve_slot` |
| `release_bp_slot(bp)` / `__release_bp_slot(bp, bp_type)` | per-event release | `HwBreakpoint::release_slot` |
| `modify_bp_slot(bp, old_type, new_type)` / `__modify_bp_slot` | per-event re-type | `HwBreakpoint::modify_slot` |
| `dbg_reserve_bp_slot(bp)` / `dbg_release_bp_slot(bp)` | per-kgdb lockless | `HwBreakpoint::dbg_reserve` / `dbg_release` |
| `toggle_bp_slot(bp, enable, type, weight)` | per-toggle accounting | `HwBreakpoint::toggle_slot` |
| `max_bp_pinned_slots(bp, type)` | per-event admission cost | `HwBreakpoint::max_pinned` |
| `task_bp_pinned(cpu, bp, type)` | per-task slot count | `HwBreakpoint::task_pinned` |
| `max_task_bp_pinned(cpu, type)` | per-CPU max-over-tasks (writer-locked) | `HwBreakpoint::max_task_pinned` |
| `bp_slots_histogram_add(hist, old, val)` | per-histogram update | `BpSlotsHistogram::add` |
| `bp_slots_histogram_max(hist, type)` | per-histogram max-occupied-N | `BpSlotsHistogram::max` |
| `bp_slots_histogram_max_merge(h1, h2, type)` | per-histogram merge-max | `BpSlotsHistogram::max_merge` |
| `find_slot_idx(bp_type)` | per-bp_type → bp_type_idx | `BpTypeIdx::from_bp_type` |
| `hw_breakpoint_weight(bp)` | per-arch slot weight (default 1) | `ArchHwBreakpoint::weight` |
| `hw_breakpoint_slots_cached(type)` | per-arch slot count (static or boot-cached) | `ArchHwBreakpoint::slots` |
| `hw_breakpoint_parse(bp, attr, hw)` | per-event arch parse + kernel-space gate | `HwBreakpoint::parse` |
| `hw_breakpoint_arch_parse(bp, attr, hw)` | per-arch attr → arch_hw_breakpoint | arch crate |
| `arch_check_bp_in_kernelspace(hw)` | per-arch kernel-vaddr check | arch crate |
| `arch_install_hw_breakpoint(bp)` / `arch_uninstall_hw_breakpoint(bp)` | per-arch DR write | arch crate |
| `hw_breakpoint_event_init(bp)` | per-PMU init callback | `HwBreakpoint::event_init` |
| `hw_breakpoint_add/del/start/stop` | per-PMU schedule callbacks | `HwBreakpoint::add`/`del`/`start`/`stop` |
| `hw_breakpoint_exceptions_notify` | per-die-notifier hook | `HwBreakpoint::exceptions_notify` |
| `hw_breakpoint_is_used()` | per-system in-use query | `HwBreakpoint::is_used` |
| `bp_perf_event_destroy(event)` | per-event destroy callback | `HwBreakpoint::destroy` |
| `init_hw_breakpoint()` | per-boot init | `HwBreakpoint::init` |
| `task_bps_ht` (rhltable) | per-task BP list hashtable | `HwBreakpoint::task_bps` |
| `bp_cpuinfo[TYPE_MAX]` (DEFINE_PER_CPU) | per-CPU info | per-cpu `BpCpuInfo` |
| `cpu_pinned[TYPE_MAX]` | global pinned-CPU histogram | global `BpSlotsHistogram` |
| `tsk_pinned_all[TYPE_MAX]` | global pinned-CPU-independent task histogram | global `BpSlotsHistogram` |
| `bp_cpuinfo_sem` (percpu rwsem) | per-constraint reader/writer lock | `HwBreakpoint::cpuinfo_sem` |
| `task_struct::perf_event_mutex` | per-task list serialization | reused |
| `constraints_initialized` (__ro_after_init bool) | per-boot gate | `HwBreakpoint::initialized` |
| `HW_BREAKPOINT_R/W/RW/X/EMPTY/INVALID` | per-uapi bp_type bitmask | shared |
| `HW_BREAKPOINT_LEN_1..8` | per-uapi access-length | shared |
| `perf_breakpoint` (struct pmu) | per-perf PMU registration | `HwBreakpoint::pmu` |

### compatibility contract

REQ-1: enum bp_type_idx:
- TYPE_INST = 0 (instruction-fetch BP / exec).
- TYPE_DATA = 0 if !__ARCH_WANT_HW_BREAKPOINT_INST_DATA_SEP else 1 (read/write watchpoint).
- TYPE_MAX = enum max.

REQ-2: find_slot_idx(bp_type) -> bp_type_idx:
- if bp_type & HW_BREAKPOINT_RW: return TYPE_DATA.
- else: return TYPE_INST.

REQ-3: struct bp_cpuinfo (per-CPU, per-type):
- cpu_pinned: unsigned int = number of pinned-CPU breakpoints on this CPU for this type.
- tsk_pinned: bp_slots_histogram = histogram of how many tasks pin N+1 slots on this CPU for this type.

REQ-4: struct bp_slots_histogram:
- count[hw_breakpoint_slots(type)] = atomic_t array; count[k] = number of "task buckets" currently occupying (k+1) slots.
- Static when `hw_breakpoint_slots` is a compile-time constant; dynamic kzalloc-on-init otherwise (init_breakpoint_slots).

REQ-5: bp_slots_histogram_add(hist, old, val):
- /* Move one task-bucket from `old` slots to `old + val` slots */
- if old != 0: atomic_dec(hist.count[old - 1]).
- new = old + val.
- if new != 0: atomic_inc(hist.count[new - 1]).

REQ-6: bp_slots_histogram_max(hist, type):
- For k from hw_breakpoint_slots_cached(type)-1 down to 0:
  - if atomic_read(hist.count[k]) > 0: return k + 1.
- return 0.

REQ-7: bp_slots_histogram_max_merge(h1, h2, type):
- /* max over k of (k+1) such that count_combined[k] > 0 across both histograms */
- For k descending: if atomic_read(h1.count[k]) + atomic_read(h2.count[k]) > 0: return k + 1.
- return 0.

REQ-8: bp_constraints_lock(bp):
- tsk_mtx = bp.hw.target ? &target.perf_event_mutex : NULL.
- if tsk_mtx:
  - mutex_lock_nested(tsk_mtx, SINGLE_DEPTH_NESTING). /* child cannot deadlock vs parent */
  - percpu_down_read(bp_cpuinfo_sem).
- else: percpu_down_write(bp_cpuinfo_sem).
- return tsk_mtx.

REQ-9: bp_constraints_unlock(tsk_mtx):
- if tsk_mtx: percpu_up_read(bp_cpuinfo_sem); mutex_unlock(tsk_mtx).
- else: percpu_up_write(bp_cpuinfo_sem).

REQ-10: task_bp_pinned(cpu, bp, type) -> int:
- assert_bp_constraints_lock_held.
- rcu_read_lock.
- head = rhltable_lookup(task_bps_ht, &bp.hw.target, task_bps_ht_params).
- count = 0.
- for iter in rhl_for_each_entry_rcu(head, hw.bp_list):
  - if find_slot_idx(iter.attr.bp_type) != type: continue.
  - if iter.cpu >= 0:
    - if cpu == -1: count = -1; break. /* not CPU-independent */
    - if cpu != iter.cpu: continue.
  - count += hw_breakpoint_weight(iter).
- rcu_read_unlock.
- return count.

REQ-11: max_task_bp_pinned(cpu, type) -> unsigned:
- lockdep_assert_held_write(bp_cpuinfo_sem). /* writer required for stable snapshot */
- return bp_slots_histogram_max_merge(get_bp_info(cpu, type).tsk_pinned, tsk_pinned_all[type], type).

REQ-12: max_bp_pinned_slots(bp, type) -> int:
- cpumask = bp.cpu < 0 ? cpu_possible_mask : cpumask_of(bp.cpu).
- /* Fast path: task BP, CPU-independent */
- if bp.hw.target ∧ bp.cpu < 0:
  - max_pinned = task_bp_pinned(-1, bp, type).
  - if max_pinned >= 0:
    - return max_pinned + bp_slots_histogram_max(cpu_pinned[type], type).
- /* Slow path: scan cpumask */
- pinned_slots = 0.
- for cpu in cpumask:
  - nr = get_bp_info(cpu, type).cpu_pinned.
  - if !bp.hw.target: nr += max_task_bp_pinned(cpu, type).
  - else: nr += task_bp_pinned(cpu, bp, type).
  - pinned_slots = max(nr, pinned_slots).
- return pinned_slots.

REQ-13: toggle_bp_slot(bp, enable, type, weight):
- if !enable: weight = -weight.
- /* Case A: CPU-pinned (no target) */
- if !bp.hw.target:
  - info = get_bp_info(bp.cpu, type).
  - lockdep_assert_held_write(bp_cpuinfo_sem).
  - bp_slots_histogram_add(cpu_pinned[type], info.cpu_pinned, weight).
  - info.cpu_pinned += weight.
  - return 0.
- /* Case B: Task-pinned */
- lockdep_assert_held_read(bp_cpuinfo_sem).
- if !enable: rhltable_remove(task_bps_ht, &bp.hw.bp_list, params). /* remove before recount */
- next_tsk_pinned = task_bp_pinned(-1, bp, type).
- if next_tsk_pinned >= 0:
  - if bp.cpu < 0: /* Case 1: CPU-independent */
    - if !enable: next_tsk_pinned += hw_breakpoint_weight(bp).
    - bp_slots_histogram_add(tsk_pinned_all[type], next_tsk_pinned, weight).
  - else if enable: /* Case 2.a: first CPU-pinned task BP */
    - for_each_possible_cpu(cpu): bp_slots_histogram_add(get_bp_info(cpu, type).tsk_pinned, 0, next_tsk_pinned). /* fan-out existing tasks */
    - bp_slots_histogram_add(get_bp_info(bp.cpu, type).tsk_pinned, next_tsk_pinned, weight).
    - bp_slots_histogram_add(tsk_pinned_all[type], next_tsk_pinned, -next_tsk_pinned). /* rebalance */
  - else: /* Case 2.b: last CPU-pinned task BP */
    - bp_slots_histogram_add(get_bp_info(bp.cpu, type).tsk_pinned, next_tsk_pinned + hw_breakpoint_weight(bp), weight).
    - for_each_possible_cpu(cpu): bp_slots_histogram_add(get_bp_info(cpu, type).tsk_pinned, next_tsk_pinned, -next_tsk_pinned).
    - bp_slots_histogram_add(tsk_pinned_all[type], 0, next_tsk_pinned).
- else: /* Case 3: per-CPU scan */
  - for cpu in cpumask_of_bp(bp):
    - next_tsk_pinned = task_bp_pinned(cpu, bp, type).
    - if !enable: next_tsk_pinned += hw_breakpoint_weight(bp).
    - bp_slots_histogram_add(get_bp_info(cpu, type).tsk_pinned, next_tsk_pinned, weight).
- if enable: return rhltable_insert(task_bps_ht, &bp.hw.bp_list, params).
- return 0.

REQ-14: __reserve_bp_slot(bp, bp_type):
- if !constraints_initialized: return -ENOMEM.
- if bp_type == HW_BREAKPOINT_EMPTY ∨ bp_type == HW_BREAKPOINT_INVALID: return -EINVAL.
- type = find_slot_idx(bp_type). weight = hw_breakpoint_weight(bp).
- max_pinned_slots = max_bp_pinned_slots(bp, type) + weight.
- if max_pinned_slots > hw_breakpoint_slots_cached(type): return -ENOSPC.
- return toggle_bp_slot(bp, true, type, weight).

REQ-15: reserve_bp_slot(bp) / release_bp_slot(bp):
- Take bp_constraints_lock, call __reserve / __release, drop.

REQ-16: modify_bp_slot(bp, old_type, new_type):
- bp_constraints_lock.
- __release_bp_slot(bp, old_type).
- err = __reserve_bp_slot(bp, new_type).
- if err: WARN(__reserve_bp_slot(bp, old_type)). /* must succeed: just released */
- bp_constraints_unlock.

REQ-17: dbg_reserve_bp_slot / dbg_release_bp_slot (kgdb path):
- if bp_constraints_is_locked: return -1.
- lockdep_off. ret = __reserve_bp_slot / __release_bp_slot. lockdep_on.
- /* No lock acquired: kgdb assumes stop-the-world */

REQ-18: hw_breakpoint_parse(bp, attr, hw):
- err = hw_breakpoint_arch_parse(bp, attr, hw). /* fills arch_hw_breakpoint */
- if err: return err.
- if arch_check_bp_in_kernelspace(hw):
  - if attr.exclude_kernel: return -EINVAL.
  - if !capable(CAP_SYS_ADMIN): return -EPERM. /* unprivileged users can't put BP in trap path */
- return 0.

REQ-19: register_perf_hw_breakpoint(bp):
- arch_hw_breakpoint hw = {}.
- err = reserve_bp_slot(bp).
- if err: return err.
- err = hw_breakpoint_parse(bp, &bp.attr, &hw).
- if err: release_bp_slot(bp); return err.
- bp.hw.info = hw.
- return 0.

REQ-20: register_user_hw_breakpoint(attr, triggered, ctx, tsk):
- return perf_event_create_kernel_counter(attr, -1, tsk, triggered, ctx).
- /* perf event_init dispatches to hw_breakpoint_event_init → register_perf_hw_breakpoint */

REQ-21: modify_user_hw_breakpoint_check(bp, attr, check):
- err = hw_breakpoint_parse(bp, attr, &hw).
- if err: return err.
- if check:
  - old_attr = bp.attr. hw_breakpoint_copy_attr(&old_attr, attr).
  - if memcmp(&old_attr, attr, sizeof(*attr)): return -EINVAL. /* only bp_addr / bp_type / bp_len / disabled modifiable */
- if bp.attr.bp_type != attr.bp_type:
  - err = modify_bp_slot(bp, bp.attr.bp_type, attr.bp_type).
  - if err: return err.
- hw_breakpoint_copy_attr(&bp.attr, attr).
- bp.hw.info = hw.

REQ-22: modify_user_hw_breakpoint(bp, attr):
- /* Disable around the modify so PMU sees consistent state */
- if irqs_disabled ∧ bp.ctx.task == current: perf_event_disable_local(bp).
- else: perf_event_disable(bp).
- err = modify_user_hw_breakpoint_check(bp, attr, false).
- if !bp.attr.disabled: perf_event_enable(bp).

REQ-23: unregister_hw_breakpoint(bp):
- if !bp: return.
- perf_event_release_kernel(bp). /* destroy → release_bp_slot */

REQ-24: register_wide_hw_breakpoint(attr, triggered, ctx):
- cpu_events = alloc_percpu(perf_event *).
- cpus_read_lock.
- for_each_online_cpu(cpu):
  - bp = perf_event_create_kernel_counter(attr, cpu, NULL, triggered, ctx).
  - if IS_ERR(bp): err = PTR_ERR(bp); break.
  - per_cpu(*cpu_events, cpu) = bp.
- cpus_read_unlock.
- if err: unregister_wide_hw_breakpoint(cpu_events); return ERR_PTR.
- return cpu_events.

REQ-25: unregister_wide_hw_breakpoint(cpu_events):
- for_each_possible_cpu(cpu): unregister_hw_breakpoint(per_cpu(*cpu_events, cpu)).
- free_percpu(cpu_events).

REQ-26: hw_breakpoint_event_init (PMU.event_init):
- if attr.type != PERF_TYPE_BREAKPOINT: return -ENOENT.
- if !hw_breakpoint_slots_cached(find_slot_idx(attr.bp_type)) ∨ has_branch_stack(bp): return -EOPNOTSUPP.
- err = register_perf_hw_breakpoint(bp).
- if err: return err.
- bp.destroy = bp_perf_event_destroy. /* release_bp_slot */
- return 0.

REQ-27: PMU schedule callbacks:
- hw_breakpoint_add(bp, flags):
  - if !(flags & PERF_EF_START): bp.hw.state = PERF_HES_STOPPED.
  - if is_sampling_event: perf_swevent_set_period.
  - return arch_install_hw_breakpoint(bp). /* arch: write DR0..DR3, DR7 enable bits */
- hw_breakpoint_del(bp, flags): arch_uninstall_hw_breakpoint(bp).
- hw_breakpoint_start(bp, flags): bp.hw.state = 0.
- hw_breakpoint_stop(bp, flags): bp.hw.state = PERF_HES_STOPPED.

REQ-28: perf_breakpoint PMU:
- .task_ctx_nr = perf_sw_context.
- .event_init = hw_breakpoint_event_init.
- .add / .del / .start / .stop / .read = hw_breakpoint_pmu_read.

REQ-29: hw_breakpoint_is_used():
- if !constraints_initialized: return false.
- for cpu in possible: for type in TYPE_MAX:
  - if get_bp_info(cpu, type).cpu_pinned: return true.
  - for slot in hw_breakpoint_slots_cached(type): if tsk_pinned.count[slot] > 0: return true.
- for type in TYPE_MAX: for slot: if cpu_pinned[type].count[slot] (WARN if so) ∨ tsk_pinned_all[type].count[slot] > 0: return true.
- return false.

REQ-30: init_hw_breakpoint():
- rhltable_init(task_bps_ht, task_bps_ht_params).
- init_breakpoint_slots() (kzalloc dynamic histograms if needed).
- constraints_initialized = true. /* __ro_after_init store */
- perf_pmu_register(perf_breakpoint, "breakpoint", PERF_TYPE_BREAKPOINT).
- register_die_notifier(hw_breakpoint_exceptions_nb). /* priority 0x7fffffff: notified first */

REQ-31: hw_breakpoint_exceptions_nb:
- .notifier_call = hw_breakpoint_exceptions_notify (arch-provided; dispatches DR6 hits to perf_bp_event).
- .priority = 0x7fffffff. /* first in chain */

REQ-32: uapi HW_BREAKPOINT_* (include/uapi/linux/hw_breakpoint.h):
- HW_BREAKPOINT_EMPTY = 0 (no breakpoint, used to express ptrace "disabled" slot).
- HW_BREAKPOINT_R = 1, HW_BREAKPOINT_W = 2, HW_BREAKPOINT_RW = R|W = 3.
- HW_BREAKPOINT_X = 4 (execute).
- HW_BREAKPOINT_INVALID = RW | X = 7 (used as sentinel; reserve_bp_slot rejects).
- HW_BREAKPOINT_LEN_1..LEN_8 length encoding.

REQ-33: ptrace HW-breakpoint exposure:
- arch ptrace requests (e.g. x86: PTRACE_GETHBPREGS / PTRACE_SETHBPREGS, arm64: NT_ARM_HW_BREAK / NT_ARM_HW_WATCH) translate user register state → perf_event_attr → register_user_hw_breakpoint(attr, ptrace_hbptriggered, NULL, tsk).
- task.thread.ptrace_bps[i] holds per-slot perf_event for the inferior; modify path uses modify_user_hw_breakpoint.
- Slot-quota negotiated through reserve/release_bp_slot the same as perf.

REQ-34: bp_constraints_is_locked(bp):
- tsk_mtx = get_task_bps_mutex(bp).
- return percpu_is_write_locked(bp_cpuinfo_sem) ∨ (tsk_mtx ? mutex_is_locked(tsk_mtx) : percpu_is_read_locked(bp_cpuinfo_sem)).
- Used by dbg_* path to bail out when kgdb is invoked from a context that already holds constraints lock.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cpuinfo_sem_writer_for_cpu_pinned_update` | INVARIANT | toggle_bp_slot with !target asserts write-locked. |
| `cpuinfo_sem_reader_for_task_pinned_update` | INVARIANT | toggle_bp_slot with target asserts read-locked. |
| `task_bps_mutex_held_for_task_pinned` | INVARIANT | task_bp_pinned + toggle assert tsk.perf_event_mutex held. |
| `histogram_count_nonneg` | INVARIANT | bp_slots_histogram_add never leaves count[k] negative. |
| `slots_quota_enforced` | INVARIANT | post-reserve: max_bp_pinned_slots ≤ slots_cached(type). |
| `kernelspace_bp_requires_cap_sys_admin` | INVARIANT | arch_check_bp_in_kernelspace ⟹ capable(CAP_SYS_ADMIN) ∨ -EPERM. |
| `exclude_kernel_consistent` | INVARIANT | arch_check_bp_in_kernelspace ∧ attr.exclude_kernel ⟹ -EINVAL. |
| `empty_invalid_rejected` | INVARIANT | __reserve_bp_slot(HW_BREAKPOINT_EMPTY \| INVALID) returns -EINVAL. |
| `dbg_reserve_lockless` | INVARIANT | dbg_reserve_bp_slot returns -1 iff bp_constraints_is_locked. |
| `modify_bp_slot_rollback` | INVARIANT | __modify_bp_slot on err re-reserves old_type (must succeed). |
| `pmu_breakpoint_only_for_PERF_TYPE_BREAKPOINT` | INVARIANT | hw_breakpoint_event_init returns -ENOENT for other type. |
| `release_bp_slot_balances_reserve` | INVARIANT | every reserve_bp_slot has matching release_bp_slot via bp.destroy. |
| `constraints_initialized_required` | INVARIANT | reserve before init_hw_breakpoint returns -ENOMEM. |

### Layer 2: TLA+

`kernel/events/hw-breakpoint.tla`:
- Models: reserve/release/modify_bp_slot under bp_cpuinfo_sem with concurrent CPU-pinned and task-pinned events.
- Models: case 1/2.a/2.b/3 of toggle_bp_slot.
- Properties:
  - `safety_slots_quota` — pinned count never exceeds hw_breakpoint_slots_cached(type).
  - `safety_histogram_consistency` — sum over k of count[k] equals number of populated task-buckets.
  - `safety_modify_atomicity` — if __reserve(new) fails, __reserve(old) re-succeeds; bp ends in well-formed state.
  - `safety_wide_rollback` — register_wide failure causes unregister_wide to release every previously-installed CPU's BP.
  - `liveness_reserve_terminates` — under quota, reserve_bp_slot returns 0 in bounded steps.
  - `safety_kgdb_path_no_locks` — dbg_reserve never acquires bp_cpuinfo_sem nor task mutex.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `HwBreakpoint::reserve_slot` post: rhltable contains bp ∨ cpu_pinned incremented | `HwBreakpoint::reserve_slot` |
| `HwBreakpoint::release_slot` post: rhltable lacks bp ∧ cpu_pinned decremented | `HwBreakpoint::release_slot` |
| `HwBreakpoint::__reserve_bp_slot` pre: bp_type ∉ {EMPTY, INVALID} | `HwBreakpoint::__reserve_bp_slot` |
| `HwBreakpoint::modify_user_check` post: bp.attr.bp_type == attr.bp_type ∧ bp.hw.info == parsed | `HwBreakpoint::modify_user_check` |
| `HwBreakpoint::register_perf` post: bp.hw.info != zero ∨ slot released | `HwBreakpoint::register_perf` |
| `HwBreakpoint::register_wide` post: all CPUs hold bp or none (on error) | `HwBreakpoint::register_wide` |
| `BpSlotsHistogram::add` post: count[old-1]'' = count[old-1] - δ ∧ count[new-1]'' = count[new-1] + δ | `BpSlotsHistogram::add` |
| `HwBreakpoint::event_init` post: bp.destroy == HwBreakpoint::destroy | `HwBreakpoint::event_init` |
| `HwBreakpoint::is_used` post: false ⟺ all counters zero | `HwBreakpoint::is_used` |

### Layer 4: Verus/Creusot functional

`Per-event create → reserve_bp_slot → hw_breakpoint_parse (arch) → bp.hw.info = parsed → PMU.add (arch_install) → ... → PMU.del (arch_uninstall) → release_bp_slot` semantic equivalence: per-Documentation/trace/perf-hw-breakpoints.rst + arch-specific HBP_NUM constraints (x86: 4 slots, arm64: per-MDSCR_EL1.MDE / DBGBVRn_EL1 count).

### hardening

(Inherits row-1 features from `kernel/perf/00-overview.md` § Hardening.)

HW-breakpoint reinforcement:

- **Per-CAP_SYS_ADMIN gate on kernel-space bp_addr** — defense against per-trap-recursion (BP set on int3 / page-fault handler).
- **Per-attr.exclude_kernel cross-check** — defense against unprivileged user requesting kernel-space BP via lax PMU.
- **Per-HW_BREAKPOINT_EMPTY / INVALID reject in __reserve** — defense against per-encoder-error consuming a slot.
- **Per-hw_breakpoint_slots quota enforced** — defense against per-DR over-subscription causing arch_install_hw_breakpoint to silently drop probes.
- **Per-bp_cpuinfo_sem percpu rwsem writer for cpu_pinned, reader for tsk_pinned** — defense against per-tearing-counts race producing -ENOSPC false-positives or false-negatives.
- **Per-task perf_event_mutex nested SINGLE_DEPTH** — defense against per-parent/child ctx deadlock during inherited events.
- **Per-modify rollback (must-succeed __reserve(old_type))** — defense against per-modify-fail leaking a slot count.
- **Per-register_wide rollback** — defense against partially-installed broadcast leaving some CPUs without the BP and one CPU with it.
- **Per-rhltable with target key, automatic_shrinking** — defense against unbounded table growth when many tasks have BPs.
- **Per-priority 0x7fffffff die-notifier** — ensures HW-BP handling precedes generic kprobe / kgdb so the DR6 status bit is consumed before being clobbered.
- **Per-dbg_reserve_bp_slot lockless path** — defense against kgdb-context attempting to acquire constraint locks (would deadlock under stop-the-world).
- **Per-constraints_initialized __ro_after_init gate** — defense against use-before-init returning bogus 0 success.
- **Per-bp.destroy = bp_perf_event_destroy** — defense against per-event-leak missing release_bp_slot on perf-event-close.

