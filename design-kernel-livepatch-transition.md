---
title: "Tier-3: kernel/livepatch/transition.c — Per-task consistency-model transition"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **livepatch consistency model** moves the system from "no patch" to "patch applied" (or vice-versa) one task at a time so that running threads finish executing the *old* function on their current stack before being flipped to the *new* function — preventing semantically-inconsistent mid-stack patching. Per-`klp_target_state` global (`KLP_TRANSITION_IDLE = -1`, `KLP_TRANSITION_UNPATCHED = 0`, `KLP_TRANSITION_PATCHED = 1`) plus per-task `task_struct.patch_state` field (same domain) drive `klp_ftrace_handler()`'s decision of "execute old vs new". Per-`TIF_PATCH_PENDING` thread-info flag is set on every task at `klp_start_transition()` and cleared by `klp_update_patch_state()` (or `__klp_sched_try_switch()`) once the task has been safely flipped. Per-`klp_check_stack()` walks each task's reliable stack trace (`stack_trace_save_tsk_reliable`) and rejects the switch if any in-flight return address falls inside a to-be-patched/-unpatched function range — yielding `-EADDRINUSE`. Per-fork: `klp_copy_process()` propagates parent's `TIF_PATCH_PENDING` and `patch_state` to the child so newborns join the transition. Per-kthread: a fake-signal wake (`klp_send_signals()` after `SIGNALS_TIMEOUT == 15` periodic retries) nudges sleeping kthreads to a quiescent point; non-kthread tasks get a `set_notify_signal()` poke. Per-idle: each CPU's idle task is flipped at the idle-loop switch point (and offline-CPU idle is flipped immediately). Per-sysfs `force`: administrator can drop all `TIF_PATCH_PENDING` and force the transition complete (marking the patch `forced` so `module_put` is deferred). Per-`klp_reverse_transition()` allows mid-flight cancellation by flipping target state, clearing pending flags, and re-starting transition. Per-`klp_sched_try_switch_key` static-key gates a fast-path stack check inside `__schedule()` to help CPU-bound kthreads. Critical for: safe in-place function replacement under concurrent execution, kthread + idle handling, fork-newborn propagation, forced-completion administrator escape hatch.

This Tier-3 covers `kernel/livepatch/transition.c` (~731 lines).

### Acceptance Criteria

- [ ] AC-1: klp_init_transition: every task (incl. all CPU idle tasks) ends with patch_state == !state ∧ klp_target_state == state.
- [ ] AC-2: klp_start_transition: every task whose patch_state != klp_target_state gains TIF_PATCH_PENDING.
- [ ] AC-3: klp_try_switch_task: task with empty/no-relevant stack and no concurrent execution flips patch_state and clears TIF_PATCH_PENDING.
- [ ] AC-4: klp_check_stack: stack entry inside [func.old_func, func.old_func + func.old_size) returns -EADDRINUSE.
- [ ] AC-5: klp_check_stack: stack entry inside [func.new_func, func.new_func + func.new_size) on UNPATCHED direction returns -EADDRINUSE.
- [ ] AC-6: klp_check_and_switch_task: task_curr(task) ∧ task != current ⟹ -EBUSY.
- [ ] AC-7: klp_check_and_switch_task: stack_trace_save_tsk_reliable failure ⟹ -EINVAL.
- [ ] AC-8: klp_update_patch_state: with TIF_PATCH_PENDING set, task.patch_state becomes klp_target_state and TIF is cleared.
- [ ] AC-9: klp_copy_process: child inherits parent's TIF_PATCH_PENDING and patch_state.
- [ ] AC-10: klp_force_transition: after invocation, all tasks have patch_state == klp_target_state and no TIF_PATCH_PENDING.
- [ ] AC-11: klp_force_transition: in UNPATCHED direction marks klp_transition_patch.forced = true; in PATCHED + replace marks all superseded patches forced.
- [ ] AC-12: klp_try_complete_transition: incomplete pass re-arms klp_transition_work delayed by ~HZ.
- [ ] AC-13: klp_complete_transition: every task ends with patch_state == KLP_TRANSITION_IDLE; klp_transition_patch == NULL; klp_target_state == IDLE.
- [ ] AC-14: klp_reverse_transition: all TIF_PATCH_PENDING cleared before flip; klp_target_state and patch.enabled both inverted; new TIF flags set by re-running start.
- [ ] AC-15: Offline idle task is switched immediately (no stack check needed).

### Architecture

```
enum KlpTransitionState {
  Idle = -1,
  Unpatched = 0,
  Patched = 1,
}

struct Transition {
  /* per-CPU scratch */
  stack_entries: PerCpu<[u64; MAX_STACK_ENTRIES]>,    // MAX_STACK_ENTRIES = 100
  /* global state (under klp_mutex) */
  target_state: i32,                                  // KlpTransitionState
  transition_patch: Option<*KlpPatch>,
  signals_cnt: u32,                                   // SIGNALS_TIMEOUT = 15
  /* static key gating __schedule() fast path */
  sched_try_switch_key: StaticKeyFalse,
  /* periodic retry */
  transition_work: DelayedWork,
}

/* SIGNALS_TIMEOUT iterations of klp_try_complete_transition before
   klp_send_signals nudges remaining tasks. */
const SIGNALS_TIMEOUT: u32 = 15;
const MAX_STACK_ENTRIES: usize = 100;
```

`Transition::init(patch, state)`:
1. WARN_ON(KLP_TARGET_STATE != IDLE).
2. KLP_TRANSITION_PATCH = Some(patch).
3. KLP_TARGET_STATE = state.
4. initial_state = !state.
5. tasklist_lock.read():
   - for_each_process_thread(task): WARN_ON(task.patch_state != IDLE); task.patch_state = initial_state.
6. for_each_possible_cpu(cpu): task = idle_task(cpu); WARN_ON(task.patch_state != IDLE); task.patch_state = initial_state.
7. smp_wmb.
8. klp_for_each_object(patch, obj): klp_for_each_func(obj, func): func.transition = true.

`Transition::start()`:
1. WARN_ON(KLP_TARGET_STATE == IDLE).
2. pr_notice.
3. tasklist_lock.read(): for_each_process_thread(task): if task.patch_state != KLP_TARGET_STATE: set_tsk_thread_flag(task, TIF_PATCH_PENDING).
4. for_each_possible_cpu(cpu): task = idle_task(cpu); if task.patch_state != KLP_TARGET_STATE: set TIF_PATCH_PENDING.
5. klp_resched_enable.
6. KLP_SIGNALS_CNT = 0.

`Transition::try_complete()`:
1. WARN_ON(KLP_TARGET_STATE == IDLE).
2. complete = true.
3. tasklist_lock.read(): for_each_process_thread(task): if !Transition::try_switch_task(task): complete = false.
4. cpus_read_lock:
   - for_each_possible_cpu(cpu): task = idle_task(cpu):
     - if cpu_online(cpu): if !Transition::try_switch_task(task): complete = false; wake_up_if_idle(cpu).
     - else if task.patch_state != KLP_TARGET_STATE: clear TIF_PATCH_PENDING; task.patch_state = KLP_TARGET_STATE.
5. if !complete:
   - if KLP_SIGNALS_CNT > 0 ∧ KLP_SIGNALS_CNT % SIGNALS_TIMEOUT == 0: Transition::send_signals.
   - KLP_SIGNALS_CNT += 1.
   - schedule_delayed_work(KLP_TRANSITION_WORK, round_jiffies_relative(HZ)).
   - return.
6. klp_resched_disable.
7. patch = KLP_TRANSITION_PATCH.unwrap().
8. Transition::complete().
9. if !patch.enabled: Klp::free_patch_async(patch).
10. else if patch.replace: Klp::free_replaced_patches_async(patch).

`Transition::try_switch_task(task) -> bool`:
1. if task.patch_state == KLP_TARGET_STATE: return true.
2. if !klp_have_reliable_stack(): return false.
3. if task == current: ret = Transition::check_and_switch_task(current, &old_name).
4. else: ret = task_call_func(task, Transition::check_and_switch_task, &old_name).
5. match ret:
   - 0 ⟹ success.
   - -EBUSY / -EINVAL / -EADDRINUSE ⟹ pr_debug describing reason.
   - other ⟹ pr_debug "Unknown error code".
6. return ret == 0.

`Transition::check_and_switch_task(task, arg) -> i32`:
1. if task_curr(task) ∧ task != current: return -EBUSY.
2. ret = Transition::check_stack(task, arg)?
3. clear_tsk_thread_flag(task, TIF_PATCH_PENDING).
4. task.patch_state = KLP_TARGET_STATE.
5. return 0.

`Transition::check_stack(task, *oldname) -> i32`:
1. lockdep_assert_preemption_disabled.
2. entries = this_cpu_ptr(KLP_STACK_ENTRIES).
3. nr = stack_trace_save_tsk_reliable(task, entries, MAX_STACK_ENTRIES); if nr < 0: return -EINVAL.
4. klp_for_each_object(KLP_TRANSITION_PATCH, obj):
   - if !obj.patched: continue.
   - klp_for_each_func(obj, func):
     - ret = Transition::check_stack_func(func, entries, nr).
     - if ret: *oldname = func.old_name; return -EADDRINUSE.
5. return 0.

`Transition::check_stack_func(func, entries, nr) -> i32`:
1. if KLP_TARGET_STATE == UNPATCHED:
   - func_addr = func.new_func; func_size = func.new_size.
2. else:
   - ops = klp_find_ops(func.old_func).
   - if ops.func_stack.is_singular(): func_addr = func.old_func; func_size = func.old_size.
   - else: prev = list_next_entry(func, stack_node); func_addr = prev.new_func; func_size = prev.new_size.
3. for entry in entries[0..nr]: if entry ∈ [func_addr, func_addr + func_size): return -EAGAIN.
4. return 0.

`Transition::update_patch_state(task)`:
1. preempt_disable_notrace.
2. if test_and_clear_tsk_thread_flag(task, TIF_PATCH_PENDING):
   - task.patch_state = READ_ONCE(KLP_TARGET_STATE).
3. preempt_enable_notrace.

`Transition::sched_try_switch()`:
1. lockdep_assert_preemption_disabled.
2. if !klp_patch_pending(current): return.
3. smp_rmb.
4. Transition::try_switch_task(current).

`Transition::complete()`:
1. pr_debug "completing %s transition".
2. patch = KLP_TRANSITION_PATCH.unwrap().
3. if patch.replace ∧ KLP_TARGET_STATE == PATCHED: Klp::unpatch_replaced_patches(patch); Klp::discard_nops(patch).
4. if KLP_TARGET_STATE == UNPATCHED:
   - Klp::unpatch_objects(patch).
   - Transition::synchronize.
5. klp_for_each_object(patch, obj): klp_for_each_func(obj, func): func.transition = false.
6. if KLP_TARGET_STATE == PATCHED: Transition::synchronize.
7. tasklist_lock.read(): for_each_process_thread(task): WARN_ON_ONCE(test TIF_PATCH_PENDING); task.patch_state = IDLE.
8. for_each_possible_cpu(cpu): task = idle_task(cpu); WARN_ON_ONCE; task.patch_state = IDLE.
9. klp_for_each_object(patch, obj):
   - if !obj.loaded: continue.
   - if KLP_TARGET_STATE == PATCHED: Klp::post_patch_callback(obj).
   - else if KLP_TARGET_STATE == UNPATCHED: Klp::post_unpatch_callback(obj).
10. pr_notice "complete".
11. KLP_TARGET_STATE = IDLE; KLP_TRANSITION_PATCH = None.

`Transition::cancel()`:
1. WARN_ON_ONCE(KLP_TARGET_STATE != PATCHED).
2. KLP_TARGET_STATE = UNPATCHED.
3. Transition::complete.

`Transition::reverse()`:
1. pr_debug.
2. tasklist_lock.read(): for_each_process_thread(task): clear TIF_PATCH_PENDING.
3. for_each_possible_cpu(cpu): clear TIF_PATCH_PENDING on idle_task(cpu).
4. Transition::synchronize.
5. patch.enabled = !patch.enabled.
6. KLP_TARGET_STATE = !KLP_TARGET_STATE.
7. smp_wmb.
8. Transition::start.

`Transition::copy_process(child)`:
1. if test_tsk_thread_flag(current, TIF_PATCH_PENDING): set_tsk_thread_flag(child, TIF_PATCH_PENDING).
2. else: clear_tsk_thread_flag(child, TIF_PATCH_PENDING).
3. child.patch_state = current.patch_state.

`Transition::force()`:
1. pr_warn "forcing remaining tasks".
2. tasklist_lock.read(): for_each_process_thread(task): Transition::update_patch_state(task).
3. for_each_possible_cpu(cpu): Transition::update_patch_state(idle_task(cpu)).
4. if KLP_TARGET_STATE == UNPATCHED: patch.forced = true.
5. else if patch.replace: for old in klp_patches: if old != patch: old.forced = true.

`Transition::send_signals()`:
1. if KLP_SIGNALS_CNT == SIGNALS_TIMEOUT: pr_notice.
2. tasklist_lock.read(): for_each_process_thread(task):
   - if !klp_patch_pending(task): continue.
   - if task.flags & PF_KTHREAD: wake_up_state(task, TASK_INTERRUPTIBLE).
   - else: set_notify_signal(task).

### Out of Scope

- Per-patch sysfs lifecycle (covered in `kernel/livepatch/core.md` Tier-3)
- ftrace handler internals (covered in `kernel/livepatch/patch.md` Tier-3)
- klp_state per-system-state cookie (covered in `kernel/livepatch/state.md` Tier-3)
- klp_shadow variables (covered in `kernel/livepatch/shadow.md` Tier-3)
- arch reliable-stack-unwinder (`stack_trace_save_tsk_reliable`) implementation (covered in arch Tier-3 docs)
- kernel-exit `exit_to_user_mode_loop` integration (covered in `kernel/entry/common.md` Tier-3)
- `__schedule()` static-key gate (covered in `kernel/sched/core.md` Tier-3)
- `copy_process()` fork integration (covered in `kernel/fork.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `KLP_TRANSITION_IDLE` (-1) | per-no-transition sentinel | const in `KlpTransitionState` |
| `KLP_TRANSITION_UNPATCHED` (0) | per-running-old code | const in `KlpTransitionState` |
| `KLP_TRANSITION_PATCHED` (1) | per-running-new code | const in `KlpTransitionState` |
| `klp_target_state` (static) | per-global transition direction | `KLP_TARGET_STATE` |
| `klp_transition_patch` | per-active-transition patch ptr | `KLP_TRANSITION_PATCH` |
| `klp_signals_cnt` (static) | per-retry counter for signal nudging | `KLP_SIGNALS_CNT` |
| `klp_stack_entries` (DEFINE_PER_CPU) | per-CPU stack-trace scratch | `KLP_STACK_ENTRIES` |
| `klp_sched_try_switch_key` (static-key) | per-fast-path enable in `__schedule()` | `KLP_SCHED_TRY_SWITCH_KEY` |
| `klp_transition_work` (delayed_work) | per-periodic retry | `KLP_TRANSITION_WORK` |
| `klp_transition_work_fn` | per-retry callback | `Transition::work_fn` |
| `klp_sync` | per-stub for `schedule_on_each_cpu` | `Transition::sync_stub` |
| `klp_synchronize_transition` | per-cross-CPU sync | `Transition::synchronize` |
| `klp_init_transition` | per-init-state + per-task initial state | `Transition::init` |
| `klp_start_transition` | per-set-TIF_PATCH_PENDING-on-all + arm sched key | `Transition::start` |
| `klp_try_complete_transition` | per-walk-all-tasks + commit-or-retry | `Transition::try_complete` |
| `klp_complete_transition` | per-cleanup post-success | `Transition::complete` |
| `klp_cancel_transition` | per-error-path cancellation | `Transition::cancel` |
| `klp_reverse_transition` | per-mid-flight reverse | `Transition::reverse` |
| `klp_force_transition` | per-admin force-complete | `Transition::force` |
| `klp_update_patch_state` | per-task flip (current/inactive) | `Transition::update_patch_state` |
| `__klp_sched_try_switch` | per-schedule()-fast-path | `Transition::sched_try_switch` |
| `klp_try_switch_task` | per-task switch attempt | `Transition::try_switch_task` |
| `klp_check_and_switch_task` | per-task check + clear flag | `Transition::check_and_switch_task` |
| `klp_check_stack` | per-task reliable-stack walk | `Transition::check_stack` |
| `klp_check_stack_func` | per-func range check | `Transition::check_stack_func` |
| `klp_send_signals` | per-retry fake-signal nudge | `Transition::send_signals` |
| `klp_copy_process` | per-fork inheritance | `Transition::copy_process` |

### compatibility contract

REQ-1: Global state machine:
- klp_target_state ∈ {KLP_TRANSITION_IDLE, KLP_TRANSITION_UNPATCHED, KLP_TRANSITION_PATCHED}.
- klp_transition_patch: NULL ⟺ klp_target_state == KLP_TRANSITION_IDLE.
- Per-task: task.patch_state ∈ {IDLE, UNPATCHED, PATCHED}.
- Per-task: TIF_PATCH_PENDING thread-info flag.

REQ-2: klp_init_transition(patch, state):
- WARN_ON(klp_target_state != KLP_TRANSITION_IDLE).
- klp_transition_patch = patch.
- klp_target_state = state.
- initial_state = !state (PATCHED ⟹ initial UNPATCHED; UNPATCHED ⟹ initial PATCHED).
- read_lock(tasklist_lock); for_each_process_thread(g, task): WARN_ON(task.patch_state != IDLE); task.patch_state = initial_state.
- for_each_possible_cpu(cpu): task = idle_task(cpu); WARN_ON(task.patch_state != IDLE); task.patch_state = initial_state.
- smp_wmb (order task.patch_state initialisations and func.transition updates, and klp_target_state write vs future TIF_PATCH_PENDING writes).
- klp_for_each_object(patch, obj): klp_for_each_func(obj, func): func.transition = true.

REQ-3: klp_start_transition():
- WARN_ON(klp_target_state == KLP_TRANSITION_IDLE).
- pr_notice("'%s': starting %s transition", patch.mod.name, patched ? "patching" : "unpatching").
- read_lock(tasklist_lock); for_each_process_thread(g, task): if task.patch_state != klp_target_state: set_tsk_thread_flag(task, TIF_PATCH_PENDING).
- for_each_possible_cpu(cpu): task = idle_task(cpu); if task.patch_state != klp_target_state: set TIF_PATCH_PENDING.
- klp_resched_enable (static_branch_enable klp_sched_try_switch_key).
- klp_signals_cnt = 0.

REQ-4: klp_try_complete_transition():
- WARN_ON(klp_target_state == KLP_TRANSITION_IDLE).
- complete = true.
- read_lock(tasklist_lock); for_each_process_thread(g, task): if !klp_try_switch_task(task): complete = false.
- cpus_read_lock.
- for_each_possible_cpu(cpu): task = idle_task(cpu):
  - if cpu_online(cpu): if !klp_try_switch_task(task): complete = false; wake_up_if_idle(cpu).
  - else if task.patch_state != klp_target_state: clear TIF_PATCH_PENDING; task.patch_state = klp_target_state.
- if !complete:
  - if klp_signals_cnt > 0 ∧ klp_signals_cnt % SIGNALS_TIMEOUT == 0: klp_send_signals.
  - klp_signals_cnt++.
  - schedule_delayed_work(klp_transition_work, round_jiffies_relative(HZ)).
  - return.
- /* Done */
- klp_resched_disable.
- patch = klp_transition_patch.
- klp_complete_transition.
- if !patch.enabled: klp_free_patch_async(patch).
- else if patch.replace: klp_free_replaced_patches_async(patch).

REQ-5: klp_try_switch_task(task):
- if task.patch_state == klp_target_state: return true (already switched).
- if !klp_have_reliable_stack: return false.
- if task == current: ret = klp_check_and_switch_task(current, &old_name).
- else: ret = task_call_func(task, klp_check_and_switch_task, &old_name).
- switch (ret):
  - 0: success.
  - -EBUSY: pr_debug "is running".
  - -EINVAL: pr_debug "has an unreliable stack".
  - -EADDRINUSE: pr_debug "is sleeping on function %s".
  - default: pr_debug "Unknown error code".
- return !ret.

REQ-6: klp_check_and_switch_task(task, arg):
- if task_curr(task) ∧ task != current: return -EBUSY (the task is currently running on another CPU).
- ret = klp_check_stack(task, arg); if ret: return ret.
- clear_tsk_thread_flag(task, TIF_PATCH_PENDING).
- task.patch_state = klp_target_state.
- return 0.

REQ-7: klp_check_stack(task, *oldname):
- lockdep_assert_preemption_disabled.
- entries = this_cpu_ptr(klp_stack_entries).
- nr_entries = stack_trace_save_tsk_reliable(task, entries, MAX_STACK_ENTRIES); if < 0: return -EINVAL.
- klp_for_each_object(klp_transition_patch, obj):
  - if !obj.patched: continue.
  - klp_for_each_func(obj, func):
    - ret = klp_check_stack_func(func, entries, nr_entries).
    - if ret: *oldname = func.old_name; return -EADDRINUSE.
- return 0.

REQ-8: klp_check_stack_func(func, entries, nr_entries):
- if klp_target_state == KLP_TRANSITION_UNPATCHED:
  - func_addr = func.new_func; func_size = func.new_size (check for to-be-unpatched function).
- else:
  - ops = klp_find_ops(func.old_func).
  - if list_is_singular(ops->func_stack): func_addr = func.old_func; func_size = func.old_size.
  - else: prev = list_next_entry(func, stack_node); func_addr = prev.new_func; func_size = prev.new_size.
- for i in 0..nr_entries: if entries[i] in [func_addr, func_addr + func_size): return -EAGAIN.
- return 0.

REQ-9: klp_update_patch_state(task):
- preempt_disable_notrace.
- if test_and_clear_tsk_thread_flag(task, TIF_PATCH_PENDING): task.patch_state = READ_ONCE(klp_target_state).
- preempt_enable_notrace.
- (The test_and_clear acts as smp_rmb for two orderings: TIF_PATCH_PENDING vs klp_target_state read, and TIF_PATCH_PENDING vs func.transition read in klp_ftrace_handler).

REQ-10: __klp_sched_try_switch():
- lockdep_assert_preemption_disabled.
- if !klp_patch_pending(current): return.
- smp_rmb (order TIF_PATCH_PENDING read vs klp_target_state read in klp_try_switch_task).
- klp_try_switch_task(current).
- (Gated by klp_sched_try_switch_key static-key inside __schedule()).

REQ-11: klp_complete_transition():
- pr_debug "'%s': completing %s transition".
- if patch.replace ∧ target == PATCHED: klp_unpatch_replaced_patches; klp_discard_nops.
- if target == UNPATCHED:
  - klp_unpatch_objects(patch) — remove new funcs from func_stack.
  - klp_synchronize_transition (so klp_ftrace_handler can no longer see them).
- klp_for_each_object: klp_for_each_func: func.transition = false.
- if target == PATCHED: klp_synchronize_transition (prevent handler seeing IDLE state mid-clear).
- read_lock(tasklist_lock); for_each_process_thread(g, task): WARN_ON_ONCE(test_tsk_thread_flag(task, TIF_PATCH_PENDING)); task.patch_state = KLP_TRANSITION_IDLE.
- for_each_possible_cpu(cpu): task = idle_task(cpu); WARN_ON_ONCE(TIF_PATCH_PENDING); task.patch_state = IDLE.
- klp_for_each_object: if obj.loaded: target == PATCHED ⟹ klp_post_patch_callback(obj); target == UNPATCHED ⟹ klp_post_unpatch_callback(obj).
- pr_notice "'%s': %s complete".
- klp_target_state = KLP_TRANSITION_IDLE; klp_transition_patch = NULL.

REQ-12: klp_cancel_transition():
- WARN_ON_ONCE(klp_target_state != KLP_TRANSITION_PATCHED) — only meaningful in error path of enable.
- klp_target_state = KLP_TRANSITION_UNPATCHED.
- klp_complete_transition.

REQ-13: klp_reverse_transition():
- pr_debug "reversing transition from %s".
- read_lock(tasklist_lock); for_each_process_thread(g, task): clear_tsk_thread_flag(task, TIF_PATCH_PENDING).
- for_each_possible_cpu(cpu): clear_tsk_thread_flag(idle_task(cpu), TIF_PATCH_PENDING).
- klp_synchronize_transition (ensure all in-flight klp_update_patch_state / __klp_sched_try_switch see cleared TIF).
- patch.enabled = !patch.enabled.
- klp_target_state = !klp_target_state.
- smp_wmb (order new target_state vs future TIF_PATCH_PENDING writes in klp_start_transition).
- klp_start_transition.

REQ-14: klp_copy_process(child) — called from copy_process during fork:
- if test_tsk_thread_flag(current, TIF_PATCH_PENDING): set_tsk_thread_flag(child, TIF_PATCH_PENDING).
- else: clear_tsk_thread_flag(child, TIF_PATCH_PENDING).
- child.patch_state = current.patch_state.
- (Serialised against klp_*_transition operations via tasklist_lock; only racers are klp_update_patch_state(current) and __klp_sched_try_switch, which cannot race because we are current.)

REQ-15: klp_force_transition():
- pr_warn "forcing remaining tasks to the patched state".
- read_lock(tasklist_lock); for_each_process_thread(g, task): klp_update_patch_state(task).
- for_each_possible_cpu(cpu): klp_update_patch_state(idle_task(cpu)).
- if klp_target_state == KLP_TRANSITION_UNPATCHED: klp_transition_patch.forced = true.
- else if klp_transition_patch.replace: klp_for_each_patch(patch): if patch != klp_transition_patch: patch.forced = true.

REQ-16: klp_send_signals():
- if klp_signals_cnt == SIGNALS_TIMEOUT: pr_notice "signaling remaining tasks".
- read_lock(tasklist_lock); for_each_process_thread(g, task):
  - if !klp_patch_pending(task): continue.
  - if task.flags & PF_KTHREAD: wake_up_state(task, TASK_INTERRUPTIBLE).
  - else: set_notify_signal(task) (fake signal — kernel-exit path will run klp_update_patch_state).

REQ-17: klp_synchronize_transition():
- schedule_on_each_cpu(klp_sync) — hard force scheduler sync even on CPUs where RCU is not watching (e.g. before user_exit), so func_stack manipulation is safe.

REQ-18: Memory-ordering contract:
- smp_wmb after task.patch_state initialisation in klp_init_transition (before func.transition writes).
- smp_wmb after klp_target_state write in klp_init_transition and klp_reverse_transition (before future TIF_PATCH_PENDING writes).
- smp_rmb in __klp_sched_try_switch (after TIF_PATCH_PENDING read, before klp_target_state read in klp_try_switch_task).
- test_and_clear_tsk_thread_flag in klp_update_patch_state acts as smp_rmb between TIF_PATCH_PENDING read and klp_target_state / func.transition reads.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `target_state_invariant` | INVARIANT | KLP_TARGET_STATE == IDLE ⟺ KLP_TRANSITION_PATCH == None. |
| `tif_cleared_implies_switched` | INVARIANT | per-update_patch_state: TIF_PATCH_PENDING cleared ⟹ task.patch_state == KLP_TARGET_STATE. |
| `running_task_busy` | INVARIANT | per-check_and_switch_task: task_curr ∧ task != current ⟹ -EBUSY. |
| `stack_range_check_inclusive` | INVARIANT | per-check_stack_func: entry ∈ [addr, addr + size) ⟹ -EAGAIN. |
| `complete_clears_all_pending` | INVARIANT | per-complete: post-state ∀ task: !TIF_PATCH_PENDING ∧ task.patch_state == IDLE. |
| `fork_inherits_pending` | INVARIANT | per-copy_process: child.TIF_PATCH_PENDING == current.TIF_PATCH_PENDING ∧ child.patch_state == current.patch_state. |
| `force_marks_forced_flag` | INVARIANT | per-force: UNPATCHED ⟹ patch.forced = true; PATCHED+replace ⟹ all old patches.forced = true. |

### Layer 2: TLA+

`kernel/livepatch/transition.tla`:
- States per task: {IDLE, INITIAL, PENDING_AT_TARGET, AT_TARGET}.
- Global: {IDLE_GLOBAL, INITIALIZING, RUNNING, REVERSING, COMPLETING}.
- Actions: init_transition, start_transition, try_switch_task, update_patch_state, sched_try_switch, copy_process, reverse_transition, force_transition, complete_transition, cancel_transition.
- Properties:
  - `safety_single_in_flight` — at-most-one transition patch globally.
  - `safety_idle_means_no_tif` — KLP_TARGET_STATE == IDLE ⟹ ∀ task: !TIF_PATCH_PENDING.
  - `safety_complete_drains_all` — Completing ⟹ ∀ task: patch_state == IDLE.
  - `safety_check_stack_rejects_in_flight_caller` — task with stack entry in target func range cannot switch.
  - `safety_force_quiesces_immediately` — force ⟹ next-state: all tasks at KLP_TARGET_STATE.
  - `safety_fork_does_not_split_state` — child.patch_state == parent.patch_state at fork instant.
  - `safety_offline_idle_immediate` — offline CPU idle task is switched without stack check.
  - `liveness_eventually_completes_or_force` — assuming reliable stacks and no permanent runners: ◇ KLP_TARGET_STATE == IDLE; otherwise force completes it.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Transition::init` post: KLP_TRANSITION_PATCH = Some(patch); KLP_TARGET_STATE = state; ∀ task: patch_state = !state | `Transition::init` |
| `Transition::start` post: ∀ task with patch_state != target: TIF_PATCH_PENDING set | `Transition::start` |
| `Transition::try_complete` post: complete ⟹ Transition::complete invoked; else delayed_work re-armed | `Transition::try_complete` |
| `Transition::try_switch_task` post: returns true ⟹ task.patch_state == KLP_TARGET_STATE | `Transition::try_switch_task` |
| `Transition::check_stack_func` post: -EAGAIN ⟺ ∃ entry ∈ [func_addr, func_addr+func_size) | `Transition::check_stack_func` |
| `Transition::update_patch_state` post: TIF cleared ⟹ task.patch_state = KLP_TARGET_STATE | `Transition::update_patch_state` |
| `Transition::complete` post: ∀ task: patch_state = IDLE; KLP_TARGET_STATE = IDLE; KLP_TRANSITION_PATCH = None | `Transition::complete` |
| `Transition::reverse` post: KLP_TARGET_STATE = !old; patch.enabled = !old.enabled; ∀ task with patch_state != new target: TIF set | `Transition::reverse` |
| `Transition::copy_process` post: child.patch_state == current.patch_state ∧ child.TIF == current.TIF | `Transition::copy_process` |
| `Transition::force` post: KLP_TARGET_STATE ∈ {UNPATCHED ⟹ patch.forced, PATCHED+replace ⟹ ∀ old patch: forced} | `Transition::force` |

### Layer 4: Verus/Creusot functional

`Per-klp_init_transition → klp_start_transition (sets TIF_PATCH_PENDING) → klp_try_complete_transition × N (each task: check_stack reliable-walk + flip patch_state) → klp_complete_transition (drain, clear TIF, set IDLE, post-callbacks) → KLP_TRANSITION_PATCH = NULL` semantic equivalence with upstream consistency model: per-Documentation/livepatch/livepatch.rst § "Consistency Model" and § "Livepatching at the function level". Per-fork `klp_copy_process` semantic equivalence with kernel/fork.c integration; per-`__klp_sched_try_switch` semantic equivalence with kernel/sched/core.c `__schedule()` integration; per-`klp_update_patch_state` semantic equivalence with kernel-exit `exit_to_user_mode_loop()` invocation in arch entry code.

### hardening

(Inherits row-1 features from `kernel/livepatch/00-overview.md` § Hardening.)

Transition reinforcement:

- **Per-reliable-stack required** — defense against per-unreliable-stack switching task mid-old-call.
- **Per-running-task on other CPU returns -EBUSY** — defense against per-flip-while-executing race.
- **Per-stack-range check inclusive on both old and new** — defense against per-mid-stack patching either direction.
- **Per-test_and_clear acts as smp_rmb** — defense against per-ordering-violation in klp_ftrace_handler reading stale klp_target_state / func.transition.
- **Per-smp_wmb after target_state writes** — defense against per-CPU racing TIF set/read against state visibility.
- **Per-tasklist_lock during init / start / complete / reverse / force / send_signals** — defense against per-fork/exit racing transition bookkeeping.
- **Per-cpus_read_lock during idle-task iteration** — defense against per-CPU-hotplug racing idle_task pointer.
- **Per-fork TIF_PATCH_PENDING propagation** — defense against per-newborn-task missing the transition.
- **Per-fake-signal nudge after SIGNALS_TIMEOUT** — defense against per-userspace-task sleeping indefinitely.
- **Per-PF_KTHREAD wake_up_state** — defense against per-kthread sleeping interruptedly missing the transition.
- **Per-offline-idle immediate flip** — defense against per-offline-CPU never running through idle loop.
- **Per-force marks `forced` flag (deferred module_put)** — defense against per-ftrace-handler still seeing patch module text post-unload.
- **Per-WARN_ON_ONCE on residual TIF in complete** — defense against per-task escaping the drain.
- **Per-klp_synchronize_transition (schedule_on_each_cpu)** — defense against per-RCU-not-watching CPU reading stale func_stack.
- **Per-stack_entries per-CPU buffer + preempt-disabled** — defense against per-concurrent reentrancy corrupting scratch.

