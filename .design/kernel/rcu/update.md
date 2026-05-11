# Tier-3: kernel/rcu/update.c — RCU shared utilities + read-side flavor-agnostic helpers

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/rcu/update.c (~700 lines)
  - kernel/rcu/rcu.h
  - include/linux/rcupdate.h
  - include/linux/rcupdate_wait.h
  - include/linux/rcupdate_trace.h
-->

## Summary

`kernel/rcu/update.c` is the **flavor-agnostic** RCU shared-utility module. It does not implement a grace-period engine itself; instead it provides: (a) the lockdep `lockdep_map` instances that all RCU flavors register their read-side critical sections against (`rcu_lock_map`, `rcu_bh_lock_map`, `rcu_sched_lock_map`, `rcu_callback_map`), (b) the read-side held-checkers `rcu_read_lock_held()` / `_bh_held()` / `_sched_held()` / `_any_held()`, (c) the expedite/normal/hurry/lazy mode-toggle state machine consulted by tree-RCU and SRCU, (d) the `wakeme_after_rcu` callback + `__wait_rcu_gp` synchronous-wait helper used by `synchronize_rcu()` / `synchronize_srcu()` etc., (e) the rcu_head debugobjects descriptor + `init_rcu_head` / `init_rcu_head_on_stack` / `destroy_rcu_head[_on_stack]` for UAF-detection of free-not-by-callback bugs, (f) `do_trace_rcu_torture_read` tracepoint shim, (g) the cookie-style polling sentinel `get_completed_synchronize_rcu()` (returns `RCU_GET_STATE_COMPLETED` — a pre-completed grace-period token consumed by `poll_state_synchronize_rcu()` which lives in `tree.c`), (h) RCU CPU-stall sysfs module parameters (`rcu_cpu_stall_suppress`, `rcu_cpu_stall_timeout`, `rcu_exp_cpu_stall_timeout`, `rcu_cpu_stall_cputime`, `rcu_exp_stall_task_details`, `rcu_cpu_stall_ftrace_dump`, `rcu_cpu_stall_notifiers`), (i) the boot self-test `rcu_test_sync_prims()` / `rcu_early_boot_tests()` + late-initcall verifier, and (j) the `rcupdate_announce_bootup_oddness` boot-banner. The public `rcu_barrier()` and `synchronize_rcu_expedited()` are flavor-implementations (in `tree.c`); update.c is the shared glue layer they all share. Critical for: lockdep RCU-correctness auditing across the whole kernel, debugobject UAF detection of callback misuse, the boot-time normal/expedited transition that lets the kernel boot under expedited grace periods and then relax once userspace starts.

This Tier-3 covers `kernel/rcu/update.c` (~700 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `rcu_lock_map` | per-RCU lockdep map | `RcuUpdate::LOCK_MAP` |
| `rcu_bh_lock_map` | per-RCU-bh lockdep map | `RcuUpdate::BH_LOCK_MAP` |
| `rcu_sched_lock_map` | per-RCU-sched lockdep map | `RcuUpdate::SCHED_LOCK_MAP` |
| `rcu_callback_map` | per-callback-invocation lockdep map | `RcuUpdate::CALLBACK_MAP` |
| `debug_lockdep_rcu_enabled()` | per-lockdep gate | `RcuUpdate::debug_lockdep_enabled` |
| `rcu_read_lock_held_common()` | per-extended-QS shortcut | `RcuUpdate::read_lock_held_common` |
| `rcu_read_lock_held()` | per-read-side check | `RcuUpdate::read_lock_held` |
| `rcu_read_lock_bh_held()` | per-bh read-side check | `RcuUpdate::read_lock_bh_held` |
| `rcu_read_lock_sched_held()` | per-sched read-side check | `RcuUpdate::read_lock_sched_held` |
| `rcu_read_lock_any_held()` | per-any-flavor check | `RcuUpdate::read_lock_any_held` |
| `rcu_gp_is_normal()` | per-expedited-suppress test | `RcuUpdate::gp_is_normal` |
| `rcu_gp_is_expedited()` | per-expedite-active test | `RcuUpdate::gp_is_expedited` |
| `rcu_expedite_gp()` / `rcu_unexpedite_gp()` | per-nesting expedite toggle | `RcuUpdate::expedite_gp` / `unexpedite_gp` |
| `rcu_async_should_hurry()` | per-lazy-callback gate | `RcuUpdate::async_should_hurry` |
| `rcu_async_hurry()` / `rcu_async_relax()` | per-lazy toggle | `RcuUpdate::async_hurry` / `async_relax` |
| `rcu_end_inkernel_boot()` | per-boot transition | `RcuUpdate::end_inkernel_boot` |
| `rcu_inkernel_boot_has_ended()` | per-boot-state query | `RcuUpdate::inkernel_boot_has_ended` |
| `rcu_test_sync_prims()` | per-mode-change self-test | `RcuUpdate::test_sync_prims` |
| `rcu_set_runtime_mode()` (core_initcall) | per-init transition | `RcuUpdate::set_runtime_mode` |
| `rcu_early_boot_tests()` | per-early-boot self-test | `RcuUpdate::early_boot_tests` |
| `rcu_verify_early_boot_tests()` (late_initcall) | per-late-boot verifier | `RcuUpdate::verify_early_boot_tests` |
| `wakeme_after_rcu()` | per-callback completion-wake | `RcuUpdate::wakeme_after_rcu` |
| `__wait_rcu_gp()` | per-multi-flavor sync wait | `RcuUpdate::wait_rcu_gp` |
| `finish_rcuwait()` | per-rcuwait teardown | `RcuUpdate::finish_rcuwait` |
| `init_rcu_head()` / `destroy_rcu_head()` | per-heap-alloc head debugobject | `RcuUpdate::init_head` / `destroy_head` |
| `init_rcu_head_on_stack()` / `destroy_rcu_head_on_stack()` | per-stack head debugobject | `RcuUpdate::init_head_on_stack` / `destroy_head_on_stack` |
| `rcuhead_debug_descr` | debugobject descriptor | `RcuUpdate::HEAD_DEBUG_DESCR` |
| `rcuhead_is_static_object()` | per-debugobject static-test | `RcuUpdate::head_is_static_object` |
| `do_trace_rcu_torture_read()` | per-rcutorture tracepoint shim | `RcuUpdate::trace_torture_read` |
| `torture_sched_setaffinity()` | per-rcutorture/locktorture | `RcuUpdate::torture_sched_setaffinity` |
| `synchronize_rcu_trivial_preempt()` | per-CONFIG_TRIVIAL_PREEMPT_RCU sync | `RcuUpdate::trivial_preempt_sync` |
| `get_completed_synchronize_rcu()` | per-polling pre-completed cookie | `RcuUpdate::get_completed` |
| `rcu_cpu_stall_suppress` | per-stall-warning suppress | shared sysctl |
| `rcu_cpu_stall_timeout` | per-stall-warning timeout (sec) | shared sysctl |
| `rcu_exp_cpu_stall_timeout` | per-expedited-stall timeout (ms) | shared sysctl |
| `rcu_cpu_stall_cputime` | per-stall CPU-time dump | shared sysctl |
| `rcu_exp_stall_task_details` | per-expedited-stall task-detail dump | shared sysctl |
| `rcu_cpu_stall_ftrace_dump` | per-stall ftrace-dump trigger | shared sysctl |
| `rcu_cpu_stall_notifiers` | per-stall notifier chain enable | shared sysctl |
| `rcu_cpu_stall_suppress_at_boot` | per-boot-stall suppress | shared sysctl |
| `rcu_normal`, `rcu_expedited` | per-mode boot/sysfs param | shared sysctl |
| `rcupdate_announce_bootup_oddness()` | per-boot banner | `RcuUpdate::announce_bootup_oddness` |

## Compatibility contract

REQ-1: `rcu_lock_map` / `_bh_lock_map` / `_sched_lock_map` / `_callback_map` are static `struct lockdep_map` instances with `STATIC_LOCKDEP_MAP_INIT`-shape initialization (name string + `lock_class_key` + wait-type pair):
- `rcu_lock_map`: name `"rcu_read_lock"`, wait-type outer `LD_WAIT_FREE`, inner `LD_WAIT_CONFIG` (PREEMPT_RT implies PREEMPT_RCU).
- `rcu_bh_lock_map`: name `"rcu_read_lock_bh"`, outer `LD_WAIT_FREE`, inner `LD_WAIT_CONFIG`.
- `rcu_sched_lock_map`: name `"rcu_read_lock_sched"`, outer `LD_WAIT_FREE`, inner `LD_WAIT_SPIN`.
- `rcu_callback_map`: name `"rcu_callback"`.
- All four EXPORT_SYMBOL_GPL; consumed by `rcu_read_lock()`/`_unlock()`, `__call_rcu_common()`, `rcu_do_batch()`, and lockdep RCU-assertion macros.

REQ-2: `debug_lockdep_rcu_enabled()` is noinstr+notrace:
- Returns true iff: `rcu_scheduler_active != RCU_SCHEDULER_INACTIVE` ∧ `READ_ONCE(debug_locks)` ∧ `current->lockdep_recursion == 0`.
- Used to gate every RCU-lockdep assertion to avoid false-positives during boot or while lockdep itself is disabled.

REQ-3: `rcu_read_lock_held_common(bool *ret) -> bool` shortcut:
- if !debug_lockdep_rcu_enabled(): *ret = true; return true (caller treats as held).
- if !rcu_is_watching(): *ret = false; return true (CPU in extended QS — idle).
- if !rcu_lockdep_current_cpu_online(): *ret = false; return true (CPU offline).
- else: return false (caller must check lock_is_held).

REQ-4: read-side-held checkers — each is notrace, each first calls `read_lock_held_common`:
- `rcu_read_lock_held()` → lock_is_held(&rcu_lock_map).
- `rcu_read_lock_bh_held()` → in_softirq() ∨ irqs_disabled() (BH disable counts).
- `rcu_read_lock_sched_held()` → lock_is_held(&rcu_sched_lock_map) ∨ !preemptible().
- `rcu_read_lock_any_held()` → any of the three lock_maps held ∨ !preemptible().

REQ-5: expedited/normal mode state:
- `rcu_normal` (module-param, 0444): boot/sysfs forces normal grace periods.
- `rcu_expedited` (module-param, 0444): boot/sysfs forces expedited grace periods.
- `rcu_normal_after_boot`: after `rcu_end_inkernel_boot`, force normal (defaults to `IS_ENABLED(CONFIG_PREEMPT_RT)`).
- `rcu_expedited_nesting` (atomic_t, init = 1 — expedited until first unexpedite at boot end): per-section nest counter.
- `rcu_gp_is_normal()` = `READ_ONCE(rcu_normal) && rcu_scheduler_active != RCU_SCHEDULER_INIT`.
- `rcu_gp_is_expedited()` = `rcu_expedited || atomic_read(&rcu_expedited_nesting)`.
- `rcu_expedite_gp()` = atomic_inc(&rcu_expedited_nesting).
- `rcu_unexpedite_gp()` = atomic_dec(&rcu_expedited_nesting).

REQ-6: lazy-callback async hurry-vs-relax (CONFIG_RCU_LAZY):
- `rcu_async_hurry_nesting` (atomic_t, init = 1).
- `rcu_async_should_hurry()` = `!IS_ENABLED(CONFIG_RCU_LAZY) || atomic_read(&rcu_async_hurry_nesting)`.
- `rcu_async_hurry()` = if CONFIG_RCU_LAZY: atomic_inc.
- `rcu_async_relax()` = if CONFIG_RCU_LAZY: atomic_dec.

REQ-7: `rcu_end_inkernel_boot()` end-of-boot transition (called by `rest_init` / late kernel-init):
- rcu_unexpedite_gp() — drop the boot-time expedited bias.
- rcu_async_relax() — drop the boot-time hurry bias.
- if rcu_normal_after_boot: WRITE_ONCE(rcu_normal, 1).
- rcu_boot_ended = true.
- `rcu_inkernel_boot_has_ended()` reads rcu_boot_ended (no smp_mb).

REQ-8: `rcu_test_sync_prims()` (only under CONFIG_PROVE_RCU):
- pr_info("Running RCU synchronous self tests").
- synchronize_rcu().
- synchronize_rcu_expedited().

REQ-9: `rcu_set_runtime_mode()` (core_initcall, !CONFIG_TINY_RCU):
- rcu_test_sync_prims().
- rcu_scheduler_active = RCU_SCHEDULER_RUNNING.
- kfree_rcu_scheduler_running() — let kfree_rcu start its kworker.
- rcu_test_sync_prims() (post-transition).

REQ-10: `wakeme_after_rcu(struct rcu_head *head)`:
- rcu = container_of(head, struct rcu_synchronize, head).
- complete(&rcu->completion).
- This is the callback installed by all flavor-agnostic synchronous-wait wrappers.

REQ-11: `__wait_rcu_gp(checktiny, state, n, crcu_array, rs_array)`:
- For each i in 0..n: skip if (checktiny && crcu_array[i] == call_rcu) — TINY_RCU need not register since synchronize_rcu reduces to barrier; also dedupe identical crcu_array entries.
- For each unique entry: init_rcu_head_on_stack(&rs_array[i].head), init_completion, invoke crcu_array[i](&rs_array[i].head, wakeme_after_rcu).
- Second pass: wait_for_completion_state(&rs_array[i].completion, state); destroy_rcu_head_on_stack.

REQ-12: `finish_rcuwait(rcuwait *w)`:
- rcu_assign_pointer(w->task, NULL).
- __set_current_state(TASK_RUNNING).
- Symmetric to `rcuwait_wait_event` in rcuwait.h.

REQ-13: rcu_head debugobject API (CONFIG_DEBUG_OBJECTS_RCU_HEAD):
- `rcuhead_debug_descr` = { .name = "rcu_head", .is_static_object = rcuhead_is_static_object }.
- `rcuhead_is_static_object(void *)` returns true unconditionally — every rcu_head is treated as a static object (no fixup on uninit detected).
- `init_rcu_head(head)` → debug_object_init(head, &rcuhead_debug_descr).
- `destroy_rcu_head(head)` → debug_object_free(head, &rcuhead_debug_descr).
- `init_rcu_head_on_stack(head)` → debug_object_init_on_stack(head, &rcuhead_debug_descr).
- `destroy_rcu_head_on_stack(head)` → debug_object_free(head, &rcuhead_debug_descr).
- All four EXPORT_SYMBOL_GPL.
- Detection: call_rcu on a freed rcu_head, kfree of an rcu_head queued for callback, double-init, init-without-free.

REQ-14: `do_trace_rcu_torture_read(rcutorturename, rhp, secs, c_old, c)`:
- if CONFIG_TREE_RCU || CONFIG_RCU_TRACE: trace_rcu_torture_read tracepoint.
- else: empty stub macro.
- EXPORT_SYMBOL_GPL.

REQ-15: rcutorture / locktorture helpers (CONFIG_RCU_TORTURE_TEST || CONFIG_LOCK_TORTURE_TEST, built-in or module):
- `torture_sched_setaffinity(pid, in_mask, dowarn)` → sched_setaffinity(pid, in_mask).
- WARN_ONCE on failure if dowarn.
- Exists so module-loaded rcutorture can rebind kthreads without exporting sched_setaffinity globally.

REQ-16: `synchronize_rcu_trivial_preempt()` (CONFIG_TRIVIAL_PREEMPT_RCU):
- smp_mb() — order prior accesses before GP start.
- rcu_read_lock() — protect task list.
- for_each_process_thread(g, t):
  - if t == current: continue.
  - while smp_load_acquire(&t->rcu_trivial_preempt_nesting): cpu_relax / continue.
- rcu_read_unlock().
- A trivially-correct preemptible-RCU GP implementation for fault-injection / education builds.

REQ-17: RCU CPU-stall sysfs / boot parameters (CONFIG_RCU_STALL_COMMON):
- `rcu_cpu_stall_ftrace_dump` (module-param, 0644): non-zero → ftrace_dump_one on stall.
- `rcu_cpu_stall_notifiers` (module-param, 0444, only CONFIG_RCU_CPU_STALL_NOTIFIER): enable per-stall notifier chain.
- `rcu_cpu_stall_suppress` (module-param, 0644): suppress stall warnings.
- `rcu_cpu_stall_timeout` (module-param, 0644, default CONFIG_RCU_CPU_STALL_TIMEOUT): normal GP stall timeout (sec).
- `rcu_exp_cpu_stall_timeout` (module-param, 0644, default CONFIG_RCU_EXP_CPU_STALL_TIMEOUT): expedited GP stall timeout (ms).
- `rcu_cpu_stall_cputime` (module-param, 0644, default `IS_ENABLED(CONFIG_RCU_CPU_STALL_CPUTIME)`): dump CPU time on stall.
- `rcu_exp_stall_task_details` (module-param, 0644): expedited-stall task-detail dump.
- `rcu_cpu_stall_suppress_at_boot` (module-param, 0444): suppress boot-time stalls (also used by rcutorture even when stall warnings excluded).

REQ-18: `get_completed_synchronize_rcu()`:
- Returns `RCU_GET_STATE_COMPLETED`.
- Treated by `poll_state_synchronize_rcu()` (in tree.c) as a cookie whose grace period has already elapsed → poll returns true immediately.
- Enables call sites to obtain a pre-completed cookie without invoking a real grace period (initialization paths).

REQ-19: early-boot self-tests (CONFIG_PROVE_RCU):
- `rcu_self_test` (module-param, 0444): opt-in.
- `rcu_early_boot_tests()`:
  - pr_info("Running RCU self tests").
  - if rcu_self_test: early_boot_test_call_rcu (issues call_rcu + start_poll_synchronize_srcu + call_srcu + kfree_rcu).
  - rcu_test_sync_prims().
- `rcu_verify_early_boot_tests()` (late_initcall):
  - if rcu_self_test: rcu_barrier(); srcu_barrier(&early_srcu); WARN_ON_ONCE(!poll_state_synchronize_srcu(&early_srcu, early_srcu_cookie)); cleanup_srcu_struct.
  - WARN_ON if rcu_self_test_counter mismatch.
- Verifies call_rcu + kfree_rcu + SRCU all delivered callbacks before late_initcall returns.

REQ-20: `rcupdate_announce_bootup_oddness()`:
- if rcu_normal: pr_info "No expedited grace period (rcu_normal)".
- else if rcu_normal_after_boot: pr_info "No expedited grace period (rcu_normal_after_boot)".
- else if rcu_expedited: pr_info "All grace periods are expedited (rcu_expedited)".
- if rcu_cpu_stall_suppress: pr_info "RCU CPU stall warnings suppressed".
- if rcu_cpu_stall_timeout != default: pr_info timeout.
- rcu_tasks_bootup_oddness() (delegates to tasks.h).

REQ-21: file includes `tasks.h` (rcu-tasks implementation pulled in from `kernel/rcu/tasks.h`) at end of file, conditional code path.

## Acceptance Criteria

- [ ] AC-1: `rcu_read_lock_held()` returns 1 from inside `rcu_read_lock()`/`_unlock()` and 0 outside (CONFIG_DEBUG_LOCK_ALLOC).
- [ ] AC-2: `rcu_read_lock_bh_held()` returns 1 from inside `local_bh_disable()` regardless of explicit `rcu_read_lock_bh` (BH-disable suffices).
- [ ] AC-3: `rcu_read_lock_sched_held()` returns 1 from `preempt_disable()` section even without `rcu_read_lock_sched`.
- [ ] AC-4: `rcu_expedite_gp()` followed by `rcu_unexpedite_gp()` leaves `rcu_expedited_nesting` unchanged.
- [ ] AC-5: `rcu_end_inkernel_boot()` decrements expedited_nesting to baseline (1 → 0) and asserts rcu_boot_ended.
- [ ] AC-6: `init_rcu_head_on_stack()` followed by call_rcu followed by `destroy_rcu_head_on_stack()` raises no debugobject warning.
- [ ] AC-7: kfree of an rcu_head still queued for callback raises `ODEBUG_BUG: object` warning.
- [ ] AC-8: `wakeme_after_rcu()` invoked from callback completes the associated completion (synchronize_rcu wakes its waiter).
- [ ] AC-9: `__wait_rcu_gp` with n=3 and 2 identical entries deduplicates (only 2 callbacks registered).
- [ ] AC-10: `get_completed_synchronize_rcu()` returns `RCU_GET_STATE_COMPLETED`; `poll_state_synchronize_rcu` of that cookie returns true without waiting.
- [ ] AC-11: `rcu_self_test=1` boot → CONFIG_PROVE_RCU late_initcall completes with `rcu_self_test_counter == early_boot_test_counter`.
- [ ] AC-12: `rcu_cpu_stall_suppress=1` boot parameter accepted; `/sys/module/rcupdate/parameters/rcu_cpu_stall_suppress` shows "1".

## Architecture

```
pub struct RcuUpdate;

// Static lockdep maps
pub static LOCK_MAP: LockdepMap = LockdepMap::new("rcu_read_lock",
    WaitType::Free, WaitType::Config);
pub static BH_LOCK_MAP: LockdepMap = LockdepMap::new("rcu_read_lock_bh",
    WaitType::Free, WaitType::Config);
pub static SCHED_LOCK_MAP: LockdepMap = LockdepMap::new("rcu_read_lock_sched",
    WaitType::Free, WaitType::Spin);
pub static CALLBACK_MAP: LockdepMap = LockdepMap::new("rcu_callback",
    WaitType::Inv, WaitType::Inv);

// Mode toggles
static RCU_EXPEDITED_NESTING: AtomicI32 = AtomicI32::new(1);   // boot=expedited
static RCU_ASYNC_HURRY_NESTING: AtomicI32 = AtomicI32::new(1); // boot=hurry
static mut RCU_NORMAL: bool = false;
static mut RCU_EXPEDITED: bool = false;
static mut RCU_NORMAL_AFTER_BOOT: bool = cfg!(CONFIG_PREEMPT_RT);
static mut RCU_BOOT_ENDED: bool = false;
```

`RcuUpdate::debug_lockdep_enabled() -> bool`:
1. /* notrace, noinstr */
2. if rcu_scheduler_active == RCU_SCHEDULER_INACTIVE: return false.
3. if !READ_ONCE(debug_locks): return false.
4. if current.lockdep_recursion != 0: return false.
5. return true.

`RcuUpdate::read_lock_held_common(ret: &mut bool) -> bool`:
1. if !Self::debug_lockdep_enabled(): *ret = true; return true.
2. if !rcu_is_watching(): *ret = false; return true.
3. if !rcu_lockdep_current_cpu_online(): *ret = false; return true.
4. return false.

`RcuUpdate::read_lock_held() -> i32` (notrace):
1. let mut ret = false.
2. if Self::read_lock_held_common(&mut ret): return ret as i32.
3. return lock_is_held(&LOCK_MAP).

`RcuUpdate::read_lock_bh_held() -> i32`:
1. let mut ret = false.
2. if Self::read_lock_held_common(&mut ret): return ret as i32.
3. return in_softirq() || irqs_disabled().

`RcuUpdate::read_lock_sched_held() -> i32`:
1. let mut ret = false.
2. if Self::read_lock_held_common(&mut ret): return ret as i32.
3. return lock_is_held(&SCHED_LOCK_MAP) || !preemptible().

`RcuUpdate::read_lock_any_held() -> i32`:
1. let mut ret = false.
2. if Self::read_lock_held_common(&mut ret): return ret as i32.
3. if lock_is_held(&LOCK_MAP) || lock_is_held(&BH_LOCK_MAP) || lock_is_held(&SCHED_LOCK_MAP): return 1.
4. return !preemptible().

`RcuUpdate::gp_is_normal() -> bool`:
1. READ_ONCE(RCU_NORMAL) && rcu_scheduler_active != RCU_SCHEDULER_INIT.

`RcuUpdate::gp_is_expedited() -> bool`:
1. RCU_EXPEDITED || RCU_EXPEDITED_NESTING.load(Relaxed) != 0.

`RcuUpdate::expedite_gp()`:
1. RCU_EXPEDITED_NESTING.fetch_add(1, Relaxed).

`RcuUpdate::unexpedite_gp()`:
1. RCU_EXPEDITED_NESTING.fetch_sub(1, Relaxed).

`RcuUpdate::async_should_hurry() -> bool`:
1. !cfg!(CONFIG_RCU_LAZY) || RCU_ASYNC_HURRY_NESTING.load(Relaxed) != 0.

`RcuUpdate::async_hurry()` / `async_relax()`:
1. if cfg!(CONFIG_RCU_LAZY): nesting.fetch_add(1) / fetch_sub(1).

`RcuUpdate::end_inkernel_boot()`:
1. Self::unexpedite_gp().
2. Self::async_relax().
3. if RCU_NORMAL_AFTER_BOOT: WRITE_ONCE(&RCU_NORMAL, true).
4. RCU_BOOT_ENDED = true.

`RcuUpdate::wakeme_after_rcu(head: *mut RcuHead)`:
1. let rcu = container_of!(head, RcuSynchronize, head).
2. complete(&rcu.completion).

`RcuUpdate::wait_rcu_gp(checktiny: bool, state: u32, n: i32, crcu_array: &[CallRcuFn], rs_array: &mut [RcuSynchronize])`:
1. /* Register callbacks */
2. for i in 0..n:
   - if checktiny && crcu_array[i] == call_rcu:
     - might_sleep().
     - continue.
   - /* dedupe */
   - for j in 0..i:
     - if crcu_array[j] == crcu_array[i]: break.
   - if j == i:
     - Self::init_head_on_stack(&mut rs_array[i].head).
     - init_completion(&mut rs_array[i].completion).
     - (crcu_array[i])(&mut rs_array[i].head, Self::wakeme_after_rcu).
3. /* Wait */
4. for i in 0..n:
   - if checktiny && crcu_array[i] == call_rcu: continue.
   - for j in 0..i:
     - if crcu_array[j] == crcu_array[i]: break.
   - if j == i:
     - wait_for_completion_state(&rs_array[i].completion, state).
     - Self::destroy_head_on_stack(&mut rs_array[i].head).

`RcuUpdate::finish_rcuwait(w: *mut RcuWait)`:
1. rcu_assign_pointer(&w.task, ptr::null_mut()).
2. __set_current_state(TASK_RUNNING).

`RcuUpdate::init_head(head: *mut RcuHead)`:
1. /* CONFIG_DEBUG_OBJECTS_RCU_HEAD only */
2. debug_object_init(head, &HEAD_DEBUG_DESCR).

`RcuUpdate::destroy_head(head: *mut RcuHead)`:
1. debug_object_free(head, &HEAD_DEBUG_DESCR).

`RcuUpdate::init_head_on_stack(head: *mut RcuHead)`:
1. debug_object_init_on_stack(head, &HEAD_DEBUG_DESCR).

`RcuUpdate::destroy_head_on_stack(head: *mut RcuHead)`:
1. debug_object_free(head, &HEAD_DEBUG_DESCR).

`RcuUpdate::trace_torture_read(name: &CStr, rhp: *mut RcuHead, secs: u64, c_old: u64, c: u64)`:
1. /* CONFIG_TREE_RCU || CONFIG_RCU_TRACE */
2. trace_rcu_torture_read(name, rhp, secs, c_old, c).
3. /* else: empty */

`RcuUpdate::torture_sched_setaffinity(pid: PidT, in_mask: &CpuMask, dowarn: bool) -> i64`:
1. ret = sched_setaffinity(pid, in_mask).
2. WARN_ONCE(dowarn && ret != 0, ...).
3. return ret.

`RcuUpdate::trivial_preempt_sync()`:
1. /* CONFIG_TRIVIAL_PREEMPT_RCU only */
2. smp_mb().
3. rcu_read_lock().
4. for_each_process_thread(g, t):
   - if t == current: continue.
   - while smp_load_acquire(&t.rcu_trivial_preempt_nesting) != 0: cpu_relax / continue.
5. rcu_read_unlock().

`RcuUpdate::get_completed() -> u64`:
1. return RCU_GET_STATE_COMPLETED.

`RcuUpdate::test_sync_prims()`:
1. /* CONFIG_PROVE_RCU only */
2. pr_info("Running RCU synchronous self tests\n").
3. synchronize_rcu().
4. synchronize_rcu_expedited().

`RcuUpdate::set_runtime_mode()` (core_initcall):
1. Self::test_sync_prims().
2. rcu_scheduler_active = RCU_SCHEDULER_RUNNING.
3. kfree_rcu_scheduler_running().
4. Self::test_sync_prims().

`RcuUpdate::early_boot_tests()`:
1. pr_info("Running RCU self tests\n").
2. if RCU_SELF_TEST: Self::early_boot_test_call_rcu().
3. Self::test_sync_prims().

`RcuUpdate::verify_early_boot_tests()` (late_initcall):
1. if RCU_SELF_TEST:
   - early_boot_test_counter += 1.
   - rcu_barrier().
   - early_boot_test_counter += 1.
   - srcu_barrier(&EARLY_SRCU).
   - WARN_ON_ONCE(!poll_state_synchronize_srcu(&EARLY_SRCU, EARLY_SRCU_COOKIE)).
   - cleanup_srcu_struct(&EARLY_SRCU).
2. if RCU_SELF_TEST_COUNTER != early_boot_test_counter:
   - WARN_ON(1); return -1.
3. return 0.

`RcuUpdate::announce_bootup_oddness()`:
1. if RCU_NORMAL: pr_info("\tNo expedited grace period (rcu_normal).\n").
2. else if RCU_NORMAL_AFTER_BOOT: pr_info("\tNo expedited grace period (rcu_normal_after_boot).\n").
3. else if RCU_EXPEDITED: pr_info("\tAll grace periods are expedited (rcu_expedited).\n").
4. if RCU_CPU_STALL_SUPPRESS: pr_info stall-suppress.
5. if RCU_CPU_STALL_TIMEOUT != CONFIG_RCU_CPU_STALL_TIMEOUT: pr_info timeout-override.
6. rcu_tasks_bootup_oddness().

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lock_map_static_init` | INVARIANT | per-LOCK_MAP/BH/SCHED/CALLBACK: name + key + wait-type initialized at startup. |
| `expedited_nesting_balanced` | INVARIANT | per-expedite_gp + unexpedite_gp paired ⟹ nesting unchanged. |
| `async_hurry_nesting_balanced` | INVARIANT | per-async_hurry + async_relax paired ⟹ nesting unchanged. |
| `read_lock_held_common_idle_returns_false` | INVARIANT | per-rcu_is_watching == false ⟹ *ret = false. |
| `wait_rcu_gp_dedup` | INVARIANT | per-identical crcu_array entries ⟹ only one callback registered. |
| `wait_rcu_gp_completion_per_unique` | INVARIANT | per-unique entry ⟹ one wait_for_completion_state matched. |
| `debug_obj_init_destroy_paired` | INVARIANT | per-init_head_on_stack ⟹ destroy_head_on_stack before scope exit. |
| `get_completed_returns_sentinel` | INVARIANT | per-get_completed: returns RCU_GET_STATE_COMPLETED. |
| `end_inkernel_boot_sets_normal` | INVARIANT | per-end_inkernel_boot ∧ RCU_NORMAL_AFTER_BOOT ⟹ RCU_NORMAL = true. |
| `boot_ended_monotonic` | INVARIANT | RCU_BOOT_ENDED is write-once true. |

### Layer 2: TLA+

`kernel/rcu/update.tla`:
- Variables: rcu_expedited_nesting, rcu_async_hurry_nesting, rcu_normal, rcu_boot_ended, scheduler_active.
- Actions: ExpediteGp, UnexpediteGp, AsyncHurry, AsyncRelax, EndInkernelBoot, SetRuntimeMode.
- Properties:
  - `safety_expedited_nesting_nonneg` — per-step: nesting ≥ 0 (caller must pair).
  - `safety_boot_ended_monotonic` — per-step: rcu_boot_ended only transitions false → true.
  - `safety_scheduler_active_monotonic` — INACTIVE → INIT → RUNNING (no rollback).
  - `liveness_end_inkernel_boot_terminates` — per-EndInkernelBoot: writes complete in finite steps.
  - `liveness_wait_rcu_gp_completes` — given a sane call_rcu impl, every wait_rcu_gp completes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `RcuUpdate::debug_lockdep_enabled` post: noinstr/notrace; correct gating | `RcuUpdate::debug_lockdep_enabled` |
| `RcuUpdate::read_lock_held*` post: returns 1 inside crit-sect, 0 outside | `RcuUpdate::read_lock_held` / `_bh_held` / `_sched_held` / `_any_held` |
| `RcuUpdate::expedite_gp` + `unexpedite_gp` post: atomic counter symmetric | `RcuUpdate::expedite_gp` / `unexpedite_gp` |
| `RcuUpdate::end_inkernel_boot` post: nesting decremented + boot_ended set | `RcuUpdate::end_inkernel_boot` |
| `RcuUpdate::wakeme_after_rcu` post: completion completed | `RcuUpdate::wakeme_after_rcu` |
| `RcuUpdate::wait_rcu_gp` post: every unique callback waited | `RcuUpdate::wait_rcu_gp` |
| `RcuUpdate::init_head` + `destroy_head` post: debugobject lifecycle paired | `RcuUpdate::init_head` / `destroy_head` |
| `RcuUpdate::get_completed` post: returns RCU_GET_STATE_COMPLETED | `RcuUpdate::get_completed` |

### Layer 4: Verus/Creusot functional

`Per-mode-toggle (rcu_expedite_gp / rcu_unexpedite_gp / rcu_async_hurry / rcu_async_relax) → per-end_inkernel_boot transition → per-rcu_self_test verify` semantic equivalence: per Documentation/RCU/RTFP.txt + Documentation/admin-guide/kernel-parameters.txt (rcupdate.rcu_*).

`Per-debugobject lifecycle (init_rcu_head_on_stack → call_rcu → callback → wakeme_after_rcu → destroy_rcu_head_on_stack)` semantic equivalence: per Documentation/RCU/checklist.rst + Documentation/RCU/lockdep.rst.

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

RCU-update reinforcement:

- **Per-rcu_lock_map static-key separation** — defense against per-cross-flavor lockdep aliasing.
- **Per-debug_lockdep_rcu_enabled gating** — defense against per-boot false-positive RCU-lockdep WARN.
- **Per-rcu_is_watching/CPU-online shortcut in read_lock_held_common** — defense against per-idle-CPU false-positive.
- **Per-rcu_expedited_nesting atomic with init=1** — defense against per-boot non-expedited GP (boot expedites until rcu_end_inkernel_boot).
- **Per-rcu_async_hurry_nesting atomic with init=1** — defense against per-boot lazy-callback stalls.
- **Per-wakeme_after_rcu container_of-correctness** — defense against per-callback type-confusion.
- **Per-__wait_rcu_gp dedupe of identical crcu_array entries** — defense against per-double-init of completion.
- **Per-init_rcu_head_on_stack/destroy_rcu_head_on_stack pairing** — defense against per-stack-rcu_head UAF after function return.
- **Per-rcuhead_debug_descr.is_static_object = true** — defense against debugobject false-positive on legitimate static rcu_head.
- **Per-rcu_self_test gated by CONFIG_PROVE_RCU + module-param opt-in** — defense against per-untrusted-boot self-test overhead.
- **Per-rcu_cpu_stall_suppress_at_boot** — defense against per-boot stall-flood (cold-cache GPs).
- **Per-WARN_ON_ONCE in torture_sched_setaffinity** — defense against per-rcutorture silent affinity-failure.
- **Per-RCU_GET_STATE_COMPLETED sentinel** — defense against per-poll-of-uninitialized-cookie false-pending.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `kernel/rcu/tree.c` Tree-RCU GP engine + `rcu_barrier()` impl + `synchronize_rcu_expedited()` impl (covered in `kernel/rcu/tree.md` Tier-3)
- `kernel/rcu/srcu.c` SRCU (covered in `kernel/rcu/srcu.md` Tier-3)
- `kernel/rcu/tasks.h` RCU-Tasks family (covered in `kernel/rcu/tasks.md` Tier-3; included at end of update.c)
- `kernel/rcu/tree_stall.h` CPU-stall warning impl
- `kernel/rcu/refscale.c` / `rcuscale.c` / `rcutorture.c` test modules
- Implementation code
