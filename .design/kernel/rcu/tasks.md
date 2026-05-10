# Tier-3: kernel/rcu/tasks.h — Task-based RCU flavors (Tasks / Tasks-Rude / Tasks-Trace)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/rcu/00-overview.md
upstream-paths:
  - kernel/rcu/tasks.h (~1608 lines)
  - kernel/rcu/update.c (rcu_tasks export glue)
  - include/linux/rcupdate.h (call_rcu_tasks / synchronize_rcu_tasks declarations)
  - include/linux/rcupdate_trace.h (rcu_tasks_trace_srcu_struct + rcu_read_lock_trace)
  - include/linux/sched.h (task_struct::rcu_tasks_* fields)
-->

## Summary

The classic `rcu_read_lock()` flavor of RCU treats every preempt-disable region as a read-side critical section, which is too restrictive for two important callers: (a) **trampoline removal** (ftrace, kprobes, livepatch) where the read-side is a sequence of instructions that may sleep, and (b) **mid-kernel preemptible execution** that calls into BPF programs that may explicitly take read-side locks. RCU-Tasks fills this gap with three flavors driven by a single generic engine in `kernel/rcu/tasks.h`:

- **RCU Tasks** (`CONFIG_TASKS_RCU`): quiescent state = voluntary context switch / `cond_resched_tasks_rcu_qs()` / userspace return / idle. Grace period waits until every non-idle task has passed through at least one such quiescent state. Used by ftrace/kprobes/livepatch trampoline removal.
- **RCU Tasks Rude** (`CONFIG_TASKS_RUDE_RCU`): induces a context switch on every online CPU via `schedule_on_each_cpu(rcu_tasks_be_rude)`. Used where even short non-preemptible regions must drain (e.g., ftrace with `CONFIG_ARCH_WANTS_NO_INSTR`).
- **RCU Tasks Trace** (`CONFIG_TASKS_TRACE_RCU`): read-side delimited by `rcu_read_lock_trace()` / `rcu_read_unlock_trace()`, safe across BPF preemption. In 7.1.0-rc2 this is implemented as a thin mapping onto **SRCU-fast** via `DEFINE_SRCU_FAST(rcu_tasks_trace_srcu_struct)`.

A single state machine `struct rcu_tasks` (per-flavor singleton) drives all variants. It holds per-CPU callback lists (`rcu_tasks_percpu` with embedded `rcu_segcblist`), a grace-period kthread (`rcu_tasks_kthread`), per-flavor function pointers (`pregp_func` / `pertask_func` / `postscan_func` / `holdouts_func` / `postgp_func`), a barrier completion (`rcu_barrier_tasks*`), and stall-detector timers. The grace-period sequence is recorded in `tasks_gp_seq` (using the standard `rcu_seq_*` API). Twelve `RTGS_*` states drive a visible state machine for stall debugging.

Critical for: ftrace/livepatch/kprobes trampoline freeing without UAF, BPF program-array release after `synchronize_rcu_tasks_trace()`, dependable read-side semantics for tracing hooks.

This Tier-3 covers `kernel/rcu/tasks.h` (~1608 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct rcu_tasks` | per-flavor singleton state | `RcuTasks` |
| `struct rcu_tasks_percpu` | per-CPU cblist + lazy timer + irq_work | `RcuTasksPercpu` |
| `DEFINE_RCU_TASKS` | per-flavor declarator macro | `define_rcu_tasks!` |
| `call_rcu_tasks_generic()` | per-enqueue any-flavor | `RcuTasks::call_generic` |
| `synchronize_rcu_tasks_generic()` | per-sync any-flavor | `RcuTasks::synchronize_generic` |
| `rcu_tasks_kthread()` | per-flavor GP kthread loop | `RcuTasks::kthread_main` |
| `rcu_tasks_one_gp()` | per-GP one pass | `RcuTasks::one_gp` |
| `rcu_tasks_need_gpcb()` | per-CPU advance + need decision | `RcuTasks::need_gpcb` |
| `rcu_tasks_invoke_cbs()` | per-CPU ready-callback invoke | `RcuTasks::invoke_cbs` |
| `rcu_tasks_invoke_cbs_wq()` | per-CPU workqueue trampoline | `RcuTasks::invoke_cbs_wq` |
| `call_rcu_tasks_iw_wakeup()` | per-CPU deferred wakeup irq_work | `RcuTasks::iw_wakeup` |
| `call_rcu_tasks_generic_timer()` | per-CPU lazy unlazify timer | `RcuTasks::lazy_timer_fn` |
| `cblist_init_generic()` | per-flavor cblist setup | `RcuTasks::cblist_init` |
| `set_tasks_gp_state()` | per-state RTGS_* transition | `RcuTasks::set_state` |
| `rcu_tasks_wait_gp()` | per-GP scan-list flavor body | `RcuTasks::wait_gp_scan` |
| `rcu_tasks_pregp_step()` | per-Tasks-classic pregp (sync_rcu) | `RcuTasks::tasks_pregp` |
| `rcu_tasks_pertask()` | per-task holdout enqueue | `RcuTasks::tasks_pertask` |
| `rcu_tasks_is_holdout()` | per-task quiescent classifier | `RcuTasks::is_holdout` |
| `rcu_tasks_postscan()` | per-flavor mid-exit drain | `RcuTasks::tasks_postscan` |
| `check_holdout_task()` | per-holdout poll | `RcuTasks::check_holdout` |
| `check_all_holdout_tasks()` | per-pass holdout-list scan | `RcuTasks::check_all_holdouts` |
| `rcu_tasks_postgp()` | per-flavor post-GP sync_rcu | `RcuTasks::tasks_postgp` |
| `rcu_tasks_be_rude()` | per-CPU empty workqueue fn | `RcuTasks::be_rude` |
| `rcu_tasks_rude_wait_gp()` | per-Rude GP (schedule_on_each_cpu) | `RcuTasks::rude_wait_gp` |
| `call_rcu_tasks` / `synchronize_rcu_tasks` / `rcu_barrier_tasks` | classic public API | `RcuTasks::{call,sync,barrier}` |
| `call_rcu_tasks_rude` / `synchronize_rcu_tasks_rude` | rude public API | `RcuTasks::{call,sync}_rude` |
| `rcu_tasks_trace_srcu_struct` | Trace = SRCU-fast | `RcuTasks::trace_srcu` |
| `exit_tasks_rcu_start` / `exit_tasks_rcu_finish` | per-exit blind-spot guard | `RcuTasks::exit_start` / `_finish` |
| `tasks_rcu_exit_srcu_stall` | per-postscan stall timer fn | `RcuTasks::exit_stall_fn` |
| `rcu_barrier_tasks_generic` | per-flavor barrier | `RcuTasks::barrier_generic` |
| `rcu_barrier_tasks_generic_cb` | per-CPU barrier completion cb | `RcuTasks::barrier_cb` |
| `show_rcu_tasks_generic_gp_kthread` | per-flavor /proc/sched_debug dump | `RcuTasks::show_kthread` |
| `rcu_tasks_torture_stats_print_generic` | per-flavor rcutorture stats | `RcuTasks::torture_stats` |
| `rcu_tasks_get_gp_data` | per-flavor sequence snapshot | `RcuTasks::get_gp_data` |
| `RTGS_INIT..RTGS_WAIT_CBS` | 12 grace-period states | `RtGpState` enum |
| `task_struct::rcu_tasks_holdout` | per-task holdout flag | `Task::rcu_tasks_holdout` |
| `task_struct::rcu_tasks_holdout_list` | per-task linkage on rtp holdout list | `Task::rcu_tasks_holdout_list` |
| `task_struct::rcu_tasks_nvcsw` | per-task snapshot of nvcsw | `Task::rcu_tasks_nvcsw` |
| `task_struct::rcu_tasks_idle_cpu` | per-task nohz_full idle CPU | `Task::rcu_tasks_idle_cpu` |
| `task_struct::rcu_tasks_exit_list` | per-CPU exit-blind-spot linkage | `Task::rcu_tasks_exit_list` |

## Compatibility contract

REQ-1: struct rcu_tasks (per-flavor singleton):
- cbs_wait: rcuwait the kthread sleeps on.
- cbs_gbl_lock: per-flavor global cblist lock (adjustment / barrier path).
- tasks_gp_mutex: per-flavor GP serialization (covers mid-boot dead zone).
- gp_state: current RTGS_* phase, written via set_tasks_gp_state.
- gp_sleep: per-flavor inter-GP idle sleep (HZ/10 classic & rude).
- init_fract: per-flavor initial holdout backoff fract (HZ/10 classic; doubles up to HZ).
- gp_jiffies / gp_start: timestamps for stall detection.
- tasks_gp_seq: rcu_seq_* style GP sequence (atomic-snapshot reader API).
- n_ipis / n_ipis_fails: per-flavor IPI counters (Rude variant primarily).
- kthread_ptr: per-flavor GP/cb-invocation kthread (smp_store_release on spawn).
- lazy_jiffies: per-flavor lazy callback timeout (DIV_ROUND_UP(HZ, 4)).
- gp_func / pregp_func / pertask_func / postscan_func / holdouts_func / postgp_func: per-flavor hooks.
- call_func: per-flavor public call_rcu_tasks_*() pointer.
- wait_state: TASK_UNINTERRUPTIBLE classic / TASK_IDLE for synchronous wait.
- rtpcpu: per-CPU rcu_tasks_percpu pointer.
- rtpcp_array: cpu_possible_mask-indexed pointer table (used by invoke_cbs flood).
- percpu_enqueue_shift / percpu_enqueue_lim / percpu_dequeue_lim / percpu_dequeue_gpseq: per-flavor callback shard policy (collapses to CPU0 under low contention).
- barrier_q_mutex / barrier_q_count / barrier_q_completion / barrier_q_seq / barrier_q_start: per-flavor rcu_barrier_tasks_* machinery.
- name / kname: per-flavor strings.

REQ-2: struct rcu_tasks_percpu:
- cblist: per-CPU rcu_segcblist (DONE, WAIT, READY-TO-INVOKE segments).
- lock: per-CPU raw_spinlock_t (rcu_node lockclass).
- rtp_jiffies / rtp_n_lock_retries: contention statistics for per-CPU-shard adaptation.
- lazy_timer: per-CPU jiffy-wheel timer that unlazifies callbacks.
- urgent_gp: per-CPU non-lazy count (>0 ⟹ wake kthread).
- rtp_work: per-CPU system_percpu_wq work_struct invoking ready callbacks.
- rtp_irq_work: per-CPU irq_work for deferred wakeup from call_rcu_tasks_generic.
- barrier_q_head: per-CPU rcu_head entrained into segcblist by rcu_barrier_tasks_generic.
- rtp_blkd_tasks: per-CPU list of tasks blocked as readers (reserved for future use).
- rtp_exit_list: per-CPU list of tasks in do_exit() between exit_tasks_rcu_start/_finish.
- cpu / index / rtpp: identity + back-pointer.

REQ-3: DEFINE_RCU_TASKS macro:
- Declares per-CPU rcu_tasks_percpu instance + struct rcu_tasks instance.
- Initializes percpu lock as RAW_SPIN_LOCK_UNLOCKED.
- Initializes rtp_irq_work with IRQ_WORK_INIT_HARD(call_rcu_tasks_iw_wakeup).
- Initializes cbs_wait, cbs_gbl_lock, tasks_gp_mutex statically.
- Sets gp_func / call_func / name from arguments.
- Sets wait_state = TASK_UNINTERRUPTIBLE (overridden to TASK_IDLE by classic-Tasks bootup).
- Sets percpu_enqueue_shift = order_base_2(CONFIG_NR_CPUS).
- Sets percpu_{enqueue,dequeue}_lim = 1 (single-CPU shard initially).
- Sets barrier_q_seq = (0UL - 50UL) << RCU_SEQ_CTR_SHIFT (so first rcu_seq_done returns true).
- Sets lazy_jiffies = DIV_ROUND_UP(HZ, 4).

REQ-4: RTGS_* state set:
- 0 RTGS_INIT — initial.
- 1 RTGS_WAIT_WAIT_CBS — unused legacy.
- 2 RTGS_WAIT_GP — waiting for gp_func to return (the scan / IPI step).
- 3 RTGS_PRE_WAIT_GP — entered pregp_func.
- 4 RTGS_SCAN_TASKLIST — for_each_process_thread(pertask_func).
- 5 RTGS_POST_SCAN_TASKLIST — entered postscan_func (drain exit blind-spot).
- 6 RTGS_WAIT_SCAN_HOLDOUTS — backoff sleep before scan_holdouts.
- 7 RTGS_SCAN_HOLDOUTS — holdouts_func pass.
- 8 RTGS_POST_GP — postgp_func (closing synchronize_rcu()).
- 9 RTGS_WAIT_READERS — (Tasks-Trace legacy path, reserved).
- 10 RTGS_INVOKE_CBS — rcu_tasks_invoke_cbs flood.
- 11 RTGS_WAIT_CBS — sleeping in rcuwait_wait_event for new callbacks.

REQ-5: set_tasks_gp_state(rtp, newstate):
- rtp.gp_state = newstate.
- rtp.gp_jiffies = jiffies.

REQ-6: cblist_init_generic(rtp):
- /* Adjust enqueue_lim */
- If rcu_task_enqueue_lim < 0: lim = 1; rcu_task_cb_adjust = true (auto-grow).
- Else if == 0: lim = 1.
- /* Per-CPU array */
- rtpcp_array = kzalloc(num_possible_cpus()).
- For each possible CPU c:
  - rtpcp = per_cpu_ptr(rtp.rtpcpu, c).
  - if c: raw_spin_lock_init(rtpcp.lock).
  - if cblist empty: rcu_segcblist_init.
  - INIT_WORK(rtp_work, rcu_tasks_invoke_cbs_wq).
  - rtpcp.cpu = c; rtpcp.rtpp = rtp; rtpcp.index = index++.
  - INIT_LIST_HEAD(rtp_blkd_tasks) if !next.
  - INIT_LIST_HEAD(rtp_exit_list) if !next.
  - barrier_q_head.next = &barrier_q_head (self-pointer = "not entrained").
- rcu_task_cpu_ids = maxcpu + 1.
- Compute percpu_enqueue_shift = ilog2(rcu_task_cpu_ids / lim) (rounded up).
- rtp.percpu_{enqueue,dequeue}_lim = lim.

REQ-7: call_rcu_tasks_generic(rhp, func, rtp):
- /* Pick a per-CPU shard */
- local_irq_save.
- rcu_read_lock (for percpu_enqueue_shift snapshot).
- ideal_cpu = smp_processor_id() >> READ_ONCE(percpu_enqueue_shift).
- chosen_cpu = cpumask_next(ideal_cpu - 1, cpu_possible_mask).
- rtpcp = per_cpu_ptr(rtp.rtpcpu, chosen_cpu).
- /* Try lock; on contention, count retries → may trigger per-shard expansion */
- If raw_spin_trylock_rcu_node fails: take lock; if (rcu_task_cb_adjust ∧ retries>contend_lim ∧ enqueue_lim != cpu_ids): needadjust = true.
- /* Enqueue */
- WARN if cblist not enabled (re-init); needwake = (func == wakeme_after_rcu) ∨ (cb count == lazy_lim).
- If kthread spawned ∧ !needwake ∧ !lazy_timer_pending: mod_timer(lazy_timer, jiffies + lazy_jiffies).
- If needwake: rtpcp.urgent_gp = 3.
- rcu_segcblist_enqueue(cblist, rhp).
- raw_spin_unlock_irqrestore_rcu_node.
- /* Adjust shard policy */
- If needadjust: under cbs_gbl_lock: percpu_enqueue_shift = 0; percpu_dequeue_lim = cpu_ids; smp_store_release(percpu_enqueue_lim, cpu_ids).
- rcu_read_unlock.
- /* Deferred wakeup (irq_work) */
- If needwake ∧ rtp.kthread_ptr: irq_work_queue(rtp_irq_work).

REQ-8: call_rcu_tasks_iw_wakeup(iwp):
- rtpcp = container_of(iwp, rcu_tasks_percpu, rtp_irq_work).
- rcuwait_wake_up(&rtpcp.rtpp.cbs_wait).

REQ-9: call_rcu_tasks_generic_timer(tlp):
- rtpcp = timer_container_of(rtpcp, tlp, lazy_timer).
- raw_spin_lock_irqsave_rcu_node.
- If !cblist_empty ∧ lazy_jiffies:
  - If !urgent_gp: urgent_gp = 1.
  - needwake = true; re-arm lazy_timer.
- raw_spin_unlock_irqrestore.
- If needwake: rcuwait_wake_up(cbs_wait).

REQ-10: rcu_tasks_need_gpcb(rtp) -> int:
- gpdone = poll_state_synchronize_rcu(percpu_dequeue_gpseq).
- For cpu in [0, dequeue_lim):
  - rcu_segcblist_advance(cblist, rcu_seq_current(tasks_gp_seq)).
  - rcu_segcblist_accelerate(cblist, rcu_seq_snap(tasks_gp_seq)).
  - If urgent_gp > 0 ∧ pend_cbs: decrement (if lazy_jiffies); needgpcb |= 0x3.
  - Else if cblist_empty: urgent_gp = 0.
  - If ready_cbs: needgpcb |= 0x1.
- /* Per-shard collapse policy */
- If rcu_task_cb_adjust ∧ ncbs ≤ rcu_task_collapse_lim ∧ enqueue_lim > 1:
  - percpu_enqueue_shift = order_base_2(rcu_task_cpu_ids); percpu_enqueue_lim = 1.
  - percpu_dequeue_gpseq = get_state_synchronize_rcu (wait one full RCU GP before dropping dequeue_lim).
- If rcu_task_cb_adjust ∧ !ncbsnz ∧ gpdone ∧ enqueue_lim < dequeue_lim:
  - percpu_dequeue_lim = 1.
- Return needgpcb (0x1 = invoke, 0x2 = need-GP, 0x3 = both).

REQ-11: rcu_tasks_invoke_cbs(rtp, rtpcp):
- /* Fan out to two children of binary index */
- index = rtpcp.index*2 + 1.
- If index < nr_possible_cpus: queue_work_on(cpu, system_percpu_wq, rtpcp_array[index].rtp_work).
- index++ and again.
- /* Self */
- If cblist_empty: return.
- raw_spin_lock_irqsave_rcu_node.
- rcu_segcblist_advance + rcu_segcblist_extract_done_cbs → rcl.
- raw_spin_unlock.
- For rhp in rcl: local_bh_disable; rhp.func(rhp); local_bh_enable; cond_resched.
- raw_spin_lock again; rcu_segcblist_add_len(-len); rcu_segcblist_accelerate.
- raw_spin_unlock.

REQ-12: rcu_tasks_one_gp(rtp, midboot):
- mutex_lock(tasks_gp_mutex).
- If !midboot:
  - mutex_unlock; set_state(RTGS_WAIT_CBS).
  - rcuwait_wait_event(cbs_wait, (needgpcb = need_gpcb(rtp)), TASK_IDLE).
  - mutex_lock(tasks_gp_mutex).
- Else needgpcb = 0x2.
- If needgpcb & 0x2:
  - set_state(RTGS_WAIT_GP); gp_start = jiffies.
  - rcu_seq_start(tasks_gp_seq); rtp.gp_func(rtp); rcu_seq_end(tasks_gp_seq).
- set_state(RTGS_INVOKE_CBS).
- rcu_tasks_invoke_cbs(rtp, per_cpu_ptr(rtpcpu, 0)).
- mutex_unlock.

REQ-13: rcu_tasks_kthread(arg):
- For each possible CPU: timer_setup(lazy_timer, call_rcu_tasks_generic_timer); urgent_gp = 1.
- housekeeping_affine(current, HK_TYPE_RCU).
- smp_store_release(&rtp.kthread_ptr, current).
- Loop forever: rcu_tasks_one_gp(rtp, false); schedule_timeout_idle(gp_sleep).

REQ-14: synchronize_rcu_tasks_generic(rtp):
- If rcu_scheduler_active == INACTIVE: WARN_ONCE; return.
- If rtp.kthread_ptr: wait_rcu_gp_state(wait_state, call_func); return.
- Else (mid-boot pre-kthread): rcu_tasks_one_gp(rtp, true).

REQ-15: rcu_tasks_wait_gp(rtp) — scan-list flavor body:
- set_state(RTGS_PRE_WAIT_GP); rtp.pregp_func(holdouts).
- set_state(RTGS_SCAN_TASKLIST).
- If rtp.pertask_func: rcu_read_lock; for_each_process_thread(g, t) pertask_func(t, holdouts); rcu_read_unlock.
- set_state(RTGS_POST_SCAN_TASKLIST); rtp.postscan_func(holdouts).
- fract = rtp.init_fract.
- Loop while holdouts non-empty:
  - set_state(RTGS_WAIT_SCAN_HOLDOUTS); schedule_timeout_idle(fract) (or hrtimeout under PREEMPT_RT).
  - If fract < HZ: fract++.
  - rtst = READ_ONCE(rcu_task_stall_timeout); needreport = rtst>0 ∧ jiffies past lastreport+rtst.
  - set_state(RTGS_SCAN_HOLDOUTS); rtp.holdouts_func(holdouts, needreport, &firstreport).
  - If rtsi-elapsed: print pre-stall pr_info.
- set_state(RTGS_POST_GP); rtp.postgp_func(rtp).

REQ-16: RCU-Tasks classic per-flavor hooks:
- pregp_func = rcu_tasks_pregp_step:
  - synchronize_rcu() — forces in-flight t->on_rq / t->nvcsw transitions (done IRQs-off) to complete.
- pertask_func = rcu_tasks_pertask:
  - If t != current ∧ rcu_tasks_is_holdout(t): get_task_struct; t.rcu_tasks_nvcsw = READ_ONCE(t.nvcsw); WRITE_ONCE(t.rcu_tasks_holdout, true); list_add to holdouts.
- rcu_tasks_is_holdout(t):
  - If !READ_ONCE(t.on_rq): return false (voluntary sleep).
  - If is_idle_task(t): return false.
  - If t == idle_task(cpu) ∧ !rcu_cpu_online(cpu): return false.
  - Else return true.
- postscan_func = rcu_tasks_postscan:
  - Start exit-stall timer (rtsi jiffies).
  - For each possible CPU: under rtpcp.lock, iterate rtp_exit_list; if t not on holdout list: rcu_tasks_pertask(t, holdouts).
  - Periodically drop lock + cond_resched.
  - Delete exit-stall timer.
- holdouts_func = check_all_holdout_tasks:
  - list_for_each_entry_safe(holdouts) → check_holdout_task(t, needreport, &firstreport); cond_resched.
- check_holdout_task(t, needreport, fr):
  - If !rcu_tasks_holdout ∨ nvcsw changed ∨ !is_holdout ∨ (NO_HZ_FULL ∧ idle_cpu>=0):
    - WRITE_ONCE(holdout, false); list_del_init; put_task_struct; return.
  - rcu_request_urgent_qs_task(t).
  - If needreport: pr_err if first; sched_show_task(t).
- postgp_func = rcu_tasks_postgp:
  - synchronize_rcu() — closes ordering on outgoing on_rq/nvcsw stores and waits for exiting-task preempt-disable region.

REQ-17: exit-blind-spot guard (CONFIG_TASKS_RCU only):
- exit_tasks_rcu_start(): preempt_disable; this_cpu rtpcp; t.rcu_tasks_exit_cpu = smp_processor_id; under rtpcp.lock list_add(t.rcu_tasks_exit_list, rtpcp.rtp_exit_list); preempt_enable.
- exit_tasks_rcu_finish(): rtpcp = per_cpu_ptr(rcu_tasks.rtpcpu, t.rcu_tasks_exit_cpu); under rtpcp.lock list_del_init(t.rcu_tasks_exit_list).
- tasks_rcu_exit_srcu_stall(): pr_info GP-old + suggest checking exit-blind-spot tasks; re-arm timer.

REQ-18: RCU-Tasks classic public API:
- DEFINE_RCU_TASKS(rcu_tasks, rcu_tasks_wait_gp, call_rcu_tasks, "RCU Tasks").
- void call_rcu_tasks(rhp, func) { call_rcu_tasks_generic(rhp, func, &rcu_tasks); } EXPORT_SYMBOL_GPL.
- void synchronize_rcu_tasks(void) { synchronize_rcu_tasks_generic(&rcu_tasks); } EXPORT_SYMBOL_GPL.
- void rcu_barrier_tasks(void) { rcu_barrier_tasks_generic(&rcu_tasks); } EXPORT_SYMBOL_GPL.
- rcu_spawn_tasks_kthread initcall: gp_sleep = HZ/10; init_fract = HZ/10; pregp/pertask/postscan/holdouts/postgp set; wait_state = TASK_IDLE.

REQ-19: RCU-Tasks-Rude flavor:
- DEFINE_RCU_TASKS(rcu_tasks_rude, rcu_tasks_rude_wait_gp, call_rcu_tasks_rude, "RCU Tasks Rude").
- rcu_tasks_be_rude(work): empty function (forces context switch on the worker CPU).
- rcu_tasks_rude_wait_gp(rtp): n_ipis += cpumask_weight(cpu_online_mask); schedule_on_each_cpu(rcu_tasks_be_rude).
- call_rcu_tasks_rude is static (no longer exported); only synchronize_rcu_tasks_rude is public.
- synchronize_rcu_tasks_rude: gated on !CONFIG_ARCH_WANTS_NO_INSTR || CONFIG_FORCE_TASKS_RUDE_RCU, then synchronize_rcu_tasks_generic(&rcu_tasks_rude).
- rcu_spawn_tasks_rude_kthread initcall: gp_sleep = HZ/10.

REQ-20: RCU-Tasks-Trace flavor (CONFIG_TASKS_TRACE_RCU):
- Implemented as SRCU-fast: DEFINE_SRCU_FAST(rcu_tasks_trace_srcu_struct).
- rcu_read_lock_trace / unlock_trace map to srcu_read_lock_fast / _unlock_fast.
- synchronize_rcu_tasks_trace / call_rcu_tasks_trace map to synchronize_srcu / call_srcu on rcu_tasks_trace_srcu_struct (defined in include/linux/rcupdate_trace.h and kernel/rcu/update.c, not in tasks.h).
- The full DEFINE_RCU_TASKS infrastructure is not instantiated for Trace in 7.1.0-rc2.

REQ-21: rcu_barrier_tasks_generic(rtp):
- s = rcu_seq_snap(barrier_q_seq).
- mutex_lock(barrier_q_mutex).
- If rcu_seq_done(barrier_q_seq, s): smp_mb; unlock; return.
- barrier_q_start = jiffies; rcu_seq_start(barrier_q_seq).
- init_completion(barrier_q_completion); atomic_set(barrier_q_count, 2).
- For cpu in [0, dequeue_lim): set barrier_q_head.func; under lock if rcu_segcblist_entrain succeeded: atomic_inc(barrier_q_count).
- If atomic_sub_and_test(2, count): complete.
- wait_for_completion; rcu_seq_end; mutex_unlock.

REQ-22: Boot-time / runtime self tests (CONFIG_PROVE_RCU):
- rcu_tasks_initiate_self_tests: synchronize_rcu_tasks + call_rcu_tasks (Tasks); synchronize_rcu_tasks_rude (Rude); synchronize_rcu_tasks_trace + call_rcu_tasks_trace (Trace).
- late_initcall(rcu_tasks_verify_schedule_work) reschedules verification every HZ until pass or RCU_TASK_BOOT_STALL_TIMEOUT (HZ*30).

REQ-23: Stall / info reporting:
- rcu_task_stall_timeout default RCU_TASK_STALL_TIMEOUT = HZ*60*10 (10 min).
- rcu_task_stall_info default = HZ*10; multiplier 3 (geometric backoff of info messages).
- show_rcu_tasks_generic_gp_kthread: dumps name, GP age, state, n_ipis, etc.

REQ-24: Module parameters:
- rcu_task_stall_timeout (RW 0644); rcu_task_stall_info (RW 0644); rcu_task_stall_info_mult (RO 0444).
- rcu_task_enqueue_lim / contend_lim / collapse_lim / lazy_lim (all RO 0444).
- rcu_tasks_lazy_ms (RO 0444).

## Acceptance Criteria

- [ ] AC-1: call_rcu_tasks(&rhp, f) returns; f invoked exactly once on rtp_work after the next grace period.
- [ ] AC-2: synchronize_rcu_tasks() blocks the caller until every non-idle task has passed a voluntary context switch / cond_resched_tasks_rcu_qs / userspace return after the call.
- [ ] AC-3: synchronize_rcu_tasks_rude() returns only after every online CPU has performed at least one context switch since entry.
- [ ] AC-4: call_rcu_tasks_trace / synchronize_rcu_tasks_trace satisfy SRCU-fast semantics with respect to rcu_read_lock_trace.
- [ ] AC-5: Tasks-classic: a task spinning in a tight loop with preempt disabled but never voluntarily scheduling is reported by check_holdout_task after rcu_task_stall_timeout.
- [ ] AC-6: rcu_tasks_kthread runs on housekeeping CPUs (HK_TYPE_RCU).
- [ ] AC-7: rcu_barrier_tasks() blocks until all callbacks enqueued before the call have been invoked.
- [ ] AC-8: rcu_barrier_tasks_rude() blocks until all rude callbacks enqueued before the call have been invoked.
- [ ] AC-9: exit_tasks_rcu_start / _finish track the exiting task on rtp_exit_list of its CPU.
- [ ] AC-10: rcu_tasks_postscan walks rtp_exit_list and adds any not-already-tracked exiting task to the holdout list.
- [ ] AC-11: rcu_tasks_pertask skips current and idle tasks; classifies on_rq && !is_idle as holdout.
- [ ] AC-12: rcu_tasks_is_holdout returns false for an idle task on an offline CPU.
- [ ] AC-13: Per-CPU shard auto-grow: contention > rcu_task_contend_lim within one jiffy triggers percpu_enqueue_lim = rcu_task_cpu_ids.
- [ ] AC-14: Per-CPU shard auto-shrink: callbacks ≤ rcu_task_collapse_lim across one RCU GP triggers collapse to CPU0.
- [ ] AC-15: Lazy callbacks (≤ rcu_task_lazy_lim) delay wake by lazy_jiffies (HZ/4 default) unless wakeme_after_rcu or boundary hit.
- [ ] AC-16: Stall timer (tasks_rcu_exit_srcu_stall_timer) re-arms every rcu_task_stall_info jiffies while postscan walks rtp_exit_list.
- [ ] AC-17: GP sequence (rtp->tasks_gp_seq) advances by 2 per call to rcu_tasks_one_gp (rcu_seq_start / rcu_seq_end).
- [ ] AC-18: synchronize_rcu_tasks_rude is compiled-out (returns without waiting) when CONFIG_ARCH_WANTS_NO_INSTR ∧ !CONFIG_FORCE_TASKS_RUDE_RCU.

## Architecture

```
struct RcuTasks {
  cbs_wait: RcuWait,
  cbs_gbl_lock: RawSpinLock,
  tasks_gp_mutex: Mutex,
  gp_state: AtomicI32,                 // RTGS_*
  gp_sleep: i32,
  init_fract: i32,
  gp_jiffies: AtomicU64,
  gp_start: u64,
  tasks_gp_seq: RcuSeq,
  n_ipis: u64,
  n_ipis_fails: u64,
  kthread_ptr: AtomicPtr<TaskStruct>,
  lazy_jiffies: u64,
  gp_func: fn(&RcuTasks),
  pregp_func: Option<fn(&mut List<TaskStruct>)>,
  pertask_func: Option<fn(&TaskStruct, &mut List<TaskStruct>)>,
  postscan_func: Option<fn(&mut List<TaskStruct>)>,
  holdouts_func: Option<fn(&mut List<TaskStruct>, bool, &mut bool)>,
  postgp_func: Option<fn(&RcuTasks)>,
  call_func: fn(*RcuHead, RcuCallback),
  wait_state: u32,                     // TASK_UNINTERRUPTIBLE / TASK_IDLE
  rtpcpu: PerCpu<RcuTasksPercpu>,
  rtpcp_array: Box<[*mut RcuTasksPercpu]>,
  percpu_enqueue_shift: AtomicI32,
  percpu_enqueue_lim: AtomicI32,
  percpu_dequeue_lim: AtomicI32,
  percpu_dequeue_gpseq: u64,
  barrier_q_mutex: Mutex,
  barrier_q_count: AtomicI32,
  barrier_q_completion: Completion,
  barrier_q_seq: RcuSeq,
  barrier_q_start: u64,
  name: &'static str,
  kname: &'static str,
}

struct RcuTasksPercpu {
  cblist: RcuSegCblist,
  lock: RawSpinLock,
  rtp_jiffies: u64,
  rtp_n_lock_retries: u64,
  lazy_timer: TimerList,
  urgent_gp: u32,
  rtp_work: WorkStruct,
  rtp_irq_work: IrqWork,
  barrier_q_head: RcuHead,
  rtp_blkd_tasks: ListHead,
  rtp_exit_list: ListHead,
  cpu: i32,
  index: i32,
  rtpp: *mut RcuTasks,
}
```

`RcuTasks::call_generic(rhp, func)`:
1. local_irq_save.
2. rcu_read_lock.
3. ideal_cpu = smp_processor_id() >> READ_ONCE(self.percpu_enqueue_shift).
4. chosen_cpu = cpumask_next(ideal_cpu - 1, cpu_possible_mask).
5. rtpcp = self.rtpcpu.get(chosen_cpu).
6. /* Try-lock; on contention bump retries and maybe set needadjust */
7. If !rtpcp.lock.try_lock_rcu_node():
   - rtpcp.lock.lock_rcu_node().
   - j = jiffies; if rtpcp.rtp_jiffies != j: rtpcp.rtp_jiffies = j; rtpcp.rtp_n_lock_retries = 0.
   - rtpcp.rtp_n_lock_retries += 1.
   - If self.cb_adjust ∧ retries > contend_lim ∧ percpu_enqueue_lim != cpu_ids: needadjust = true.
8. needwake = (func is wakeme_after_rcu) ∨ (segcblist.n_cbs() == lazy_lim).
9. If self.kthread_ptr.load() ∧ !needwake ∧ !rtpcp.lazy_timer.pending():
   - If self.lazy_jiffies: mod_timer(rtpcp.lazy_timer, jiffies + self.lazy_jiffies).
   - Else needwake = segcblist.is_empty().
10. If needwake: rtpcp.urgent_gp = 3.
11. rcu_segcblist_enqueue(&mut rtpcp.cblist, rhp).
12. rtpcp.lock.unlock_irqrestore(flags).
13. If needadjust: self.cbs_gbl_lock.with_irqsave(|| { percpu_enqueue_shift = 0; percpu_dequeue_lim = cpu_ids; smp_store_release(percpu_enqueue_lim, cpu_ids); }).
14. rcu_read_unlock.
15. If needwake ∧ self.kthread_ptr.load(): rtpcp.rtp_irq_work.queue().

`RcuTasks::one_gp(midboot)`:
1. self.tasks_gp_mutex.lock().
2. If !midboot:
   - drop mutex; set_state(RTGS_WAIT_CBS).
   - rcuwait_wait_event(self.cbs_wait, (needgpcb = self.need_gpcb()), TASK_IDLE).
   - reacquire mutex.
3. Else needgpcb = 0x2.
4. If needgpcb & 0x2:
   - set_state(RTGS_WAIT_GP); self.gp_start = jiffies.
   - rcu_seq_start(&self.tasks_gp_seq).
   - (self.gp_func)(self).
   - rcu_seq_end(&self.tasks_gp_seq).
5. set_state(RTGS_INVOKE_CBS).
6. self.invoke_cbs(self.rtpcpu.get(0)).
7. drop mutex.

`RcuTasks::kthread_main(rtp)`:
1. For each possible CPU: timer_setup(lazy_timer, lazy_timer_fn); urgent_gp = 1.
2. housekeeping_affine(current, HK_TYPE_RCU).
3. smp_store_release(&rtp.kthread_ptr, current).
4. Loop:
   - rtp.one_gp(false).
   - schedule_timeout_idle(rtp.gp_sleep).

`RcuTasks::wait_gp_scan(rtp)` — body for Tasks-classic flavor:
1. set_state(RTGS_PRE_WAIT_GP); rtp.pregp_func(&holdouts).
2. set_state(RTGS_SCAN_TASKLIST).
3. If rtp.pertask_func.is_some(): rcu_read_lock; for_each_process_thread(g, t) rtp.pertask_func(t, &holdouts); rcu_read_unlock.
4. set_state(RTGS_POST_SCAN_TASKLIST); rtp.postscan_func(&holdouts).
5. fract = rtp.init_fract.
6. While !holdouts.empty():
   - set_state(RTGS_WAIT_SCAN_HOLDOUTS); schedule_timeout_idle(fract) (or hrtimer under PREEMPT_RT).
   - fract = min(fract+1, HZ).
   - rtst = READ_ONCE(rcu_task_stall_timeout). needreport = rtst>0 ∧ time_after(jiffies, lastreport+rtst).
   - WARN_ON(signal_pending(current)).
   - set_state(RTGS_SCAN_HOLDOUTS); rtp.holdouts_func(&holdouts, needreport, &mut firstreport).
   - If pre-stall info window expired: pr_info "%s grace period number %lu is %lu jiffies old".
7. set_state(RTGS_POST_GP); rtp.postgp_func(rtp).

`RcuTasks::tasks_pregp(holdouts)`:
1. synchronize_rcu().

`RcuTasks::tasks_pertask(t, holdouts)`:
1. If t == current: return.
2. If !RcuTasks::is_holdout(t): return.
3. get_task_struct(t).
4. t.rcu_tasks_nvcsw = READ_ONCE(t.nvcsw).
5. WRITE_ONCE(t.rcu_tasks_holdout, true).
6. list_add(&t.rcu_tasks_holdout_list, holdouts).

`RcuTasks::is_holdout(t) -> bool`:
1. If !READ_ONCE(t.on_rq): return false.
2. If is_idle_task(t): return false.
3. cpu = task_cpu(t).
4. If t == idle_task(cpu) ∧ !rcu_cpu_online(cpu): return false.
5. Else return true.

`RcuTasks::tasks_postscan(holdouts)`:
1. Start tasks_rcu_exit_srcu_stall_timer (jiffies + rcu_task_stall_info).
2. For cpu in cpu_possible_mask:
   - j = jiffies + 1; rtpcp = per_cpu_ptr(rcu_tasks.rtpcpu, cpu).
   - rtpcp.lock.lock_irq_rcu_node().
   - list_for_each_entry_safe(t, t1, rtpcp.rtp_exit_list):
     - If t.rcu_tasks_holdout_list.is_empty(): tasks_pertask(t, holdouts).
     - If !PREEMPT_RT ∧ time_before(jiffies, j): continue.
     - list_add(&tmp, &t.rcu_tasks_exit_list); rtpcp.lock.unlock_irq.
     - cond_resched.
     - rtpcp.lock.lock_irq.
     - t1 = list_entry(tmp.next, ...); list_del(&tmp); j = jiffies + 1.
   - rtpcp.lock.unlock_irq.
3. timer_delete_sync(tasks_rcu_exit_srcu_stall_timer).

`RcuTasks::check_holdout(t, needreport, firstreport)`:
1. If !READ_ONCE(t.rcu_tasks_holdout) ∨ t.rcu_tasks_nvcsw != READ_ONCE(t.nvcsw) ∨ !is_holdout(t) ∨ (NO_HZ_FULL ∧ !is_idle(t) ∧ READ_ONCE(t.rcu_tasks_idle_cpu) ≥ 0):
   - WRITE_ONCE(t.rcu_tasks_holdout, false).
   - list_del_init(&t.rcu_tasks_holdout_list).
   - put_task_struct(t).
   - return.
2. rcu_request_urgent_qs_task(t).
3. If !needreport: return.
4. If *firstreport: pr_err "INFO: rcu_tasks detected stalls on tasks:"; *firstreport = false.
5. pr_alert with cpu / nvcsw / holdout / idle_cpu; sched_show_task(t).

`RcuTasks::tasks_postgp(rtp)`:
1. synchronize_rcu().

`RcuTasks::rude_wait_gp(rtp)`:
1. rtp.n_ipis += cpumask_weight(cpu_online_mask).
2. schedule_on_each_cpu(rcu_tasks_be_rude).

`RcuTasks::be_rude(work)`:
1. /* intentionally empty */

`RcuTasks::exit_start(current)`:
1. preempt_disable.
2. rtpcp = this_cpu_ptr(rcu_tasks.rtpcpu).
3. current.rcu_tasks_exit_cpu = smp_processor_id().
4. rtpcp.lock.with_irqsave(|| list_add(&current.rcu_tasks_exit_list, &rtpcp.rtp_exit_list)).
5. preempt_enable.

`RcuTasks::exit_finish(current)`:
1. rtpcp = per_cpu_ptr(rcu_tasks.rtpcpu, current.rcu_tasks_exit_cpu).
2. rtpcp.lock.with_irqsave(|| list_del_init(&current.rcu_tasks_exit_list)).

`RcuTasks::barrier_generic(rtp)`:
1. s = rcu_seq_snap(&rtp.barrier_q_seq).
2. rtp.barrier_q_mutex.lock().
3. If rcu_seq_done(&rtp.barrier_q_seq, s): smp_mb; drop mutex; return.
4. rtp.barrier_q_start = jiffies; rcu_seq_start(&rtp.barrier_q_seq).
5. init_completion(&rtp.barrier_q_completion); atomic_set(&rtp.barrier_q_count, 2).
6. For cpu in [0, smp_load_acquire(rtp.percpu_dequeue_lim)):
   - rtpcp = per_cpu_ptr(rtp.rtpcpu, cpu).
   - rtpcp.barrier_q_head.func = barrier_cb.
   - rtpcp.lock.with_irqsave(|| if rcu_segcblist_entrain(cblist, &rtpcp.barrier_q_head): atomic_inc(&rtp.barrier_q_count)).
7. If atomic_sub_and_test(2, &rtp.barrier_q_count): complete(&rtp.barrier_q_completion).
8. wait_for_completion(&rtp.barrier_q_completion).
9. rcu_seq_end(&rtp.barrier_q_seq); drop mutex.

`RcuTasks::trace_*` (CONFIG_TASKS_TRACE_RCU):
- rcu_tasks_trace_srcu_struct is a DEFINE_SRCU_FAST.
- read-side: srcu_read_lock_fast / srcu_read_unlock_fast.
- update-side: synchronize_srcu(rcu_tasks_trace_srcu_struct); call_srcu(rcu_tasks_trace_srcu_struct, …).
- No pertask scan, no holdout list, no kthread of its own (uses SRCU's kthreads).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `kthread_ptr_release_visible` | INVARIANT | per-kthread_main: writers see smp_store_release ordering for rtp.kthread_ptr before any caller of synchronize_rcu_tasks_generic observes it. |
| `enqueue_under_percpu_lock` | INVARIANT | per-call_generic: rcu_segcblist_enqueue runs with rtpcp.lock held. |
| `iw_wakeup_only_after_kthread` | INVARIANT | per-call_generic: irq_work_queue gated on smp_load_acquire(kthread_ptr) != NULL. |
| `barrier_count_balanced` | INVARIANT | per-barrier_generic: every increment of barrier_q_count is matched by a barrier_cb-driven decrement before wait_for_completion returns. |
| `holdout_task_refcount_balanced` | INVARIANT | per-pertask/check_holdout: get_task_struct in pertask matched by put_task_struct in check_holdout's clear path. |
| `exit_list_per_cpu_balanced` | INVARIANT | per-exit_start/_finish: every list_add matched by list_del_init on the same CPU index. |
| `gp_seq_monotone` | INVARIANT | per-one_gp: rcu_seq_start / rcu_seq_end strictly monotone (each call advances by 2). |
| `pertask_skips_current_and_idle` | INVARIANT | per-pertask: t == current ∨ is_idle_task(t) ⟹ no get_task_struct. |
| `rude_wait_drains_online_cpus` | INVARIANT | per-rude_wait_gp: schedule_on_each_cpu waits on all online CPUs. |
| `trace_srcu_fast_read_lock_balanced` | INVARIANT | per-Trace: srcu_read_lock_fast / _unlock_fast are paired. |

### Layer 2: TLA+

`kernel/rcu/rcu-tasks.tla`:
- Per-flavor state machine over RTGS_* states + per-flavor sequence rtp.tasks_gp_seq.
- Properties:
  - `safety_state_transitions_only_via_set_state` — per-set_tasks_gp_state.
  - `safety_postgp_sync_rcu_required` — per-Tasks: postgp_func always synchronize_rcu before return.
  - `safety_pregp_sync_rcu_required` — per-Tasks: pregp_func always synchronize_rcu before scan.
  - `safety_holdout_drain_progress` — per-wait_gp: every loop pass either deletes ≥1 holdout or reschedules with backoff.
  - `safety_rude_drains_all_online_cpus` — per-Rude: schedule_on_each_cpu waits on cpumask_weight(cpu_online_mask) workers.
  - `safety_callback_invoked_after_gp_end` — per-invoke_cbs: rcu_seq_done before func(rhp).
  - `safety_barrier_blocks_until_drained` — per-barrier_generic.
  - `liveness_kthread_progress` — per-kthread_main: eventually wakes when urgent_gp > 0.
  - `liveness_stall_eventually_reported` — per-check_holdout_task: stall older than rcu_task_stall_timeout ⟹ pr_err.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `call_generic` post: rhp on rtpcp.cblist of chosen_cpu | `RcuTasks::call_generic` |
| `need_gpcb` post: returns ∈ {0, 0x1, 0x2, 0x3}; bits reflect ready/pend on at least one shard | `RcuTasks::need_gpcb` |
| `one_gp` post: gp_state == RTGS_INVOKE_CBS at exit; gp_seq advanced by 2 if 0x2 was set | `RcuTasks::one_gp` |
| `wait_gp_scan` post: holdouts list empty | `RcuTasks::wait_gp_scan` |
| `tasks_pertask` post: current not on holdout list; idle tasks not on holdout list | `RcuTasks::tasks_pertask` |
| `check_holdout` post: if cleared, list_del_init && put_task_struct called once | `RcuTasks::check_holdout` |
| `tasks_postgp` post: synchronize_rcu has returned | `RcuTasks::tasks_postgp` |
| `barrier_generic` post: every entrained cb has run before completion | `RcuTasks::barrier_generic` |
| `rude_wait_gp` post: each online CPU has run rcu_tasks_be_rude | `RcuTasks::rude_wait_gp` |

### Layer 4: Verus/Creusot functional

`Per-flavor pipeline: call_rcu_tasks(rhp,f) → enqueue → kthread sees urgent → one_gp(pregp → scan → postscan → holdouts-loop → postgp) → invoke_cbs → f(rhp)` semantic equivalence: per-Documentation/RCU/{rcu_tasks,Design/Requirements}.rst and per-include/linux/rcupdate.h.

`Per-rude pipeline: synchronize_rcu_tasks_rude() → schedule_on_each_cpu(empty) → every online CPU context-switched → return` semantic equivalence: per-Documentation/RCU/rcu_tasks.rst Rude section.

`Per-trace pipeline: rcu_read_lock_trace / unlock_trace = srcu_read_lock_fast / _unlock_fast(rcu_tasks_trace_srcu_struct); synchronize_rcu_tasks_trace = synchronize_srcu(...)` semantic equivalence: per-include/linux/rcupdate_trace.h.

## Hardening

(Inherits row-1 features from `kernel/rcu/00-overview.md` § Hardening.)

RCU-Tasks reinforcement:

- **Per-RCU-Tasks pregp_func and postgp_func bracket every Tasks GP with synchronize_rcu()** — defense against per-IRQ-disabled on_rq / nvcsw store reordering.
- **Per-exit-blind-spot two-section guard (exit_tasks_rcu_start / _finish)** — defense against per-exiting-task escaping the global tasklist scan.
- **Per-task refcount get/put strict in pertask / check_holdout** — defense against per-holdout UAF when task exits mid-scan.
- **Per-housekeeping_affine(HK_TYPE_RCU) kthread placement** — defense against per-isolated-CPU disturbance for nohz_full workloads.
- **Per-percpu_enqueue_shift gate read inside rcu_read_lock** — defense against per-shard-policy races during call_generic.
- **Per-rcu_segcblist_is_enabled WARN_ON_ONCE before enqueue** — defense against per-uninitialized-cblist enqueue.
- **Per-rude variant excluded under CONFIG_ARCH_WANTS_NO_INSTR** — defense against per-noinstr IPI hazard.
- **Per-tasks_rcu_exit_srcu_stall_timer in postscan** — defense against per-rtp_exit_list scan hang.
- **Per-rcu_task_stall_timeout / stall_info_mult** — defense against per-silent GP stall.
- **Per-shard collapse held off one full RCU GP via percpu_dequeue_gpseq** — defense against per-dequeue-after-collapse miss.
- **Per-call_rcu_tasks_iw_wakeup deferred wakeup via irq_work** — defense against per-call-site irqs-off rcuwait wakeup recursion.
- **Per-rcu_request_urgent_qs_task in check_holdout** — defense against per-CPU-bound holdout never reaching scheduler.
- **Per-rcu_barrier_tasks_generic_cb container_of via barrier_q_head** — defense against per-stale barrier callback misrouting.

## Open Questions

(none at this Tier-3 level — RCU-Tasks-Trace's pertask-scan implementation history is captured by the comment "implemented via a straightforward mapping onto SRCU-fast" in the upstream source; Rookery follows the 7.1.0-rc2 baseline.)

## Out of Scope

- kernel/rcu/tree.c classic preemptible RCU (covered in `tree.md` Tier-3)
- kernel/rcu/srcutree.c SRCU + SRCU-fast (covered in `srcu.md` Tier-3)
- kernel/rcu/rcuscale.c / rcutorture.c testing harnesses (covered separately if expanded)
- kernel/rcu/update.c symbol-export glue (covered in `00-overview.md` Tier-2)
- include/linux/rcupdate_trace.h headers that define rcu_read_lock_trace inline wrappers
- kernel/livepatch/ + ftrace trampoline lifecycle (covered separately)
- BPF program-array reclamation through call_rcu_tasks_trace (covered in BPF Tier-3 if expanded)
- Implementation code
