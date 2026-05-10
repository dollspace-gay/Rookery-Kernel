---
title: "Tier-3: kernel/exit.c — Task termination and reaping"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **exit subsystem** orchestrates task termination from the moment a thread invokes `exit(2)` / `exit_group(2)` / receives a fatal signal until its `task_struct` is freed. Per-thread teardown sequence (`do_exit`): synchronize_group_exit → ptrace EVENT_EXIT → io_uring_files_cancel → sched_mm_cid_exit → exit_signals (sets PF_EXITING) → seccomp_filter_release → acct_update_integrals → group-dead bookkeeping → perf_event_exit_task → unwind_deferred_task_exit → exit_mm → exit_sem/exit_shm/exit_files/exit_fs → disassociate_ctty (group dead) → exit_nsproxy_namespaces → exit_task_work → exit_thread → sched_autogroup_exit_task → cgroup_task_exit → flush_ptrace_hw_breakpoint → exit_tasks_rcu_start → exit_notify (becomes EXIT_ZOMBIE; SIGCHLD to parent; autoreap decision) → proc_exit_connector → mpol_put_task_policy → futex pi_state cleanup → debug_check_no_locks_held → exit_io_context → put pipe/page → exit_task_stack_account → check_stack_usage → exit_rcu → exit_tasks_rcu_finish → lockdep_free_task → do_task_dead (TASK_DEAD, scheduler removes). Per-`do_group_exit` (whole tgid): sets `SIGNAL_GROUP_EXIT`, `zap_other_threads` SIGKILL to peers, then `do_exit`. Per-`release_task` runs from parent's `wait*()` or kernel autoreap: __exit_signal (signal_struct accumulate + sighand release), proc_flush_pid, free_pids, release_thread, flush_sigqueue, put_task_struct_rcu_user. Per-parent reparenting: `find_new_reaper` walks thread group → child_subreaper ancestor → pid_ns->child_reaper (init); orphan pgrps may receive SIGHUP+SIGCONT. Per-`wait4`/`waitpid`/`waitid` use `wait_opts` and `wait_consider_task` to pick between zombie/stopped/continued events. Per-`autoreap` (SA_NOCLDWAIT or SIG_IGN of SIGCHLD): zombie immediately transitions to EXIT_DEAD and is reaped. Critical for: PID lifecycle, memory reclaim, parent notification, container teardown, deterministic exit_code propagation.

This Tier-3 covers `kernel/exit.c` (~2003 lines).

### Acceptance Criteria

- [ ] AC-1: exit(2): single thread teardown; mm/files/fs/sighand dropped; EXIT_ZOMBIE set; SIGCHLD to parent.
- [ ] AC-2: exit_group(2): zap_other_threads SIGKILL peers; group_exit_code stored; all threads do_exit.
- [ ] AC-3: Parent ignores SIGCHLD (SIG_IGN or SA_NOCLDWAIT): zombie auto-reaped; wait4 returns -ECHILD.
- [ ] AC-4: Sub-thread (non-leader) exit: autoreap = true; never visible to wait(); leader survives as zombie until last thread.
- [ ] AC-5: Parent exits with children alive: children reparented to subreaper or pid_ns->init via find_new_reaper.
- [ ] AC-6: init (pid 1) thread exits while children exist: panic("Attempted to kill init!").
- [ ] AC-7: wait4(WNOHANG) with no zombie: returns 0; with zombie: returns pid + status.
- [ ] AC-8: waitid(P_PID, pid, infop, WEXITED): infop->si_code in {CLD_EXITED, CLD_KILLED, CLD_DUMPED}; si_status = exit code or signal.
- [ ] AC-9: waitid(P_PIDFD, pidfd, ...): resolves pidfd → pid; same semantics as P_PID.
- [ ] AC-10: WSTOPPED on stopped child (SIGSTOP): returns pid; si_code = CLD_STOPPED; status preserved until reaped.
- [ ] AC-11: WCONTINUED on resumed child: returns once per SIGCONT; signal->flags cleared.
- [ ] AC-12: Orphan process group with stopped jobs: SIGHUP + SIGCONT broadcast on parent exit.
- [ ] AC-13: release_task: detach_pid for PID/TGID/PGID/SID; sighand kfree'd; task freed via RCU.
- [ ] AC-14: Child cputime accumulated into parent's signal->cutime/cstime on reap.
- [ ] AC-15: ptraced child exit: collected via wait() by tracer; EXIT_TRACE if real parent has yet to see.

### Architecture

```
struct WaitOpts {
  wo_type: PidType,                  // PID / TGID / PGID / MAX
  wo_pid: Option<*Pid>,
  wo_flags: u32,                     // W*
  wo_info: Option<*WaitidInfo>,
  wo_stat: Option<UserPtr<i32>>,
  wo_rusage: Option<UserPtr<RUsage>>,
  notask_error: i32,                 // starts -ECHILD
  child_wait: WaitQueueEntry,
}

enum ExitState { Running, Zombie, Dead, Trace }

struct ReleaseTaskPost {
  pids: [Option<*Pid>; PIDTYPE_MAX],
}
```

`Exit::do_exit(code) -> !`:
1. WARN_ON(irqs_disabled()); WARN_ON(plug).
2. if let Some(kth) = tsk_is_kthread(current): kthread_do_exit(kth, code).
3. kcov_task_exit; kmsan_task_exit.
4. Exit::synchronize_group_exit(current, code).
5. ptrace_event(PTRACE_EVENT_EXIT, code).
6. user_events_exit(current).
7. io_uring_files_cancel(); sched_mm_cid_exit(current).
8. Signal::exit_signals(current) /* PF_EXITING + retarget */.
9. seccomp_filter_release(current).
10. acct_update_integrals(current).
11. group_dead = atomic_dec_and_test(&current.signal.live).
12. if group_dead:
    - if is_global_init(current): panic("Attempted to kill init!").
    - hrtimer_cancel(&signal.real_timer); exit_itimers(current).
    - if current.mm: setmax_mm_hiwater_rss(&signal.maxrss, current.mm).
13. acct_collect(code, group_dead).
14. if group_dead: tty_audit_exit().
15. audit_free(current).
16. current.exit_code = code.
17. taskstats_exit(current, group_dead); trace_sched_process_exit.
18. perf_event_exit_task(current).
19. unwind_deferred_task_exit(current).
20. Exit::exit_mm().
21. if group_dead: acct_process().
22. exit_sem(current); exit_shm(current); exit_files(current); exit_fs(current).
23. if group_dead: disassociate_ctty(1).
24. exit_nsproxy_namespaces(current); exit_task_work(current); exit_thread(current).
25. sched_autogroup_exit_task(current); cgroup_task_exit(current); flush_ptrace_hw_breakpoint(current).
26. exit_tasks_rcu_start().
27. Exit::exit_notify(current, group_dead).
28. proc_exit_connector(current); mpol_put_task_policy(current).
29. if let Some(pi) = current.pi_state_cache: kfree(pi).
30. debug_check_no_locks_held().
31. if let Some(io) = current.io_context: exit_io_context(current).
32. if let Some(pipe) = current.splice_pipe: free_pipe_info(pipe).
33. if let Some(p) = current.task_frag.page: put_page(p).
34. exit_task_stack_account(current); check_stack_usage().
35. preempt_disable().
36. if current.nr_dirtied > 0: per_cpu_add(dirty_throttle_leaks, nr_dirtied).
37. exit_rcu(); exit_tasks_rcu_finish().
38. lockdep_free_task(current).
39. do_task_dead() /* TASK_DEAD; __schedule(false); BUG; noreturn */.

`Exit::do_group_exit(code) -> !`:
1. sig = current.signal.
2. if sig.flags & SIGNAL_GROUP_EXIT: code = sig.group_exit_code.
3. else if sig.group_exec_task: code = 0.
4. else:
   - sighand = current.sighand.
   - sighand.siglock.lock_irq().
   - re-check under lock.
   - if neither: sig.group_exit_code = code; sig.flags = SIGNAL_GROUP_EXIT; zap_other_threads(current).
   - unlock.
5. Exit::do_exit(code).

`Exit::exit_notify(tsk, group_dead)`:
1. let mut dead = LinkedList::new().
2. tasklist_lock.write_lock_irq().
3. Exit::forget_original_parent(tsk, &mut dead).
4. if group_dead: Exit::kill_orphaned_pgrp(tsk.group_leader, None).
5. tsk.exit_state = EXIT_ZOMBIE.
6. let autoreap = if tsk.ptrace != 0 {
     let sig = if thread_group_empty(tsk) && !ptrace_reparented(tsk) { tsk.exit_signal } else { SIGCHLD };
     do_notify_parent(tsk, sig)
   } else if thread_group_leader(tsk) {
     thread_group_empty(tsk) && do_notify_parent(tsk, tsk.exit_signal)
   } else {
     do_notify_pidfd(tsk); true
   }.
7. if autoreap: tsk.exit_state = EXIT_DEAD; dead.push_back(tsk).
8. if tsk.signal.notify_count < 0: wake_up_process(tsk.signal.group_exec_task).
9. tasklist_lock.write_unlock_irq().
10. for p in dead: list_del_init(&p.ptrace_entry); Exit::release_task(p).

`Exit::release_task(p)`:
1. loop {
2. let mut post = ReleaseTaskPost::zeroed().
3. dec_rlimit_ucounts(task_ucounts(p), UCOUNT_RLIMIT_NPROC, 1).
4. pidfs_exit(p); cgroup_task_release(p).
5. let thread_pid = task_pid(p).
6. tasklist_lock.write_lock_irq().
7. ptrace_release_task(p).
8. Exit::exit_signal_inner(&mut post, p).
9. let mut zap_leader = false; let leader = p.group_leader.
10. if leader != p && thread_group_empty(leader) && leader.exit_state == EXIT_ZOMBIE:
    - if leader.signal.flags & SIGNAL_GROUP_EXIT: leader.exit_code = leader.signal.group_exit_code.
    - zap_leader = do_notify_parent(leader, leader.exit_signal).
    - if zap_leader: leader.exit_state = EXIT_DEAD.
11. tasklist_lock.write_unlock_irq().
12. proc_flush_pid(thread_pid).
13. exit_cred_namespaces(p).
14. add_device_randomness(&p.se.sum_exec_runtime as bytes).
15. free_pids(&post.pids).
16. release_thread(p).
17. flush_sigqueue(&p.pending).
18. if thread_group_leader(p): flush_sigqueue(&p.signal.shared_pending).
19. put_task_struct_rcu_user(p).
20. if !zap_leader: return.
21. p = leader.
22. }.

`Exit::exit_signal_inner(post, tsk)`:
1. let sig = tsk.signal.
2. let group_dead = thread_group_leader(tsk).
3. let sighand = rcu_dereference_check(tsk.sighand, lockdep_tasklist_lock_is_held()).
4. sighand.siglock.lock().
5. posix_cpu_timers_exit(tsk); if group_dead: posix_cpu_timers_exit_group(tsk).
6. if group_dead: let tty = sig.tty; sig.tty = None.
   else:
   - if sig.notify_count > 0 { sig.notify_count -= 1; if sig.notify_count == 0 { wake_up_process(sig.group_exec_task); } }.
   - if tsk == sig.curr_target: sig.curr_target = next_thread(tsk).
7. let (utime, stime) = task_cputime(tsk).
8. sig.stats_lock.write_seqlock().
9. sig.utime += utime; sig.stime += stime; sig.gtime += task_gtime(tsk).
10. sig.min_flt += tsk.min_flt; sig.maj_flt += tsk.maj_flt.
11. sig.nvcsw += tsk.nvcsw; sig.nivcsw += tsk.nivcsw.
12. sig.inblock += io_inblock(tsk); sig.oublock += io_oublock(tsk).
13. task_io_accounting_add(&sig.ioac, &tsk.ioac).
14. sig.sum_sched_runtime += tsk.se.sum_exec_runtime.
15. sig.nr_threads -= 1.
16. Exit::unhash_process(post, tsk, group_dead).
17. sig.stats_lock.write_sequnlock().
18. tsk.sighand = None.
19. sighand.siglock.unlock().
20. Exit::cleanup_sighand(sighand) /* kfree if last ref */.
21. if group_dead { tty_kref_put(tty); }.

`Exit::find_new_reaper(father, child_reaper) -> *TaskStruct`:
1. if let Some(t) = Exit::find_alive_thread(father): return t.
2. let mut reaper = father.real_parent.
3. while reaper != child_reaper:
   - if reaper.signal.is_child_subreaper:
     - if let Some(t) = Exit::find_alive_thread(reaper): return t.
   - reaper = reaper.real_parent.
4. return child_reaper.

`Exit::do_wait(wo) -> i64`:
1. init_waitqueue_func_entry(&wo.child_wait, child_wait_callback).
2. wo.child_wait.private = current.
3. add_wait_queue(&current.signal.wait_chldexit, &wo.child_wait).
4. loop:
   - set_current_state(TASK_INTERRUPTIBLE).
   - retval = Exit::do_wait_inner(wo).
   - if retval != -ERESTARTSYS: break.
   - if signal_pending(current): break.
   - schedule().
5. set_current_state(TASK_RUNNING).
6. remove_wait_queue(&current.signal.wait_chldexit, &wo.child_wait).
7. return retval.

`Exit::do_wait_inner(wo) -> i64`:
1. wo.notask_error = -ECHILD.
2. if wo.wo_type < PIDTYPE_MAX && (wo.wo_pid.is_none() || !pid_has_task(wo.wo_pid, wo.wo_type)): goto notask.
3. tasklist_lock.read_lock().
4. if wo.wo_type == PIDTYPE_PID:
   - retval = Exit::do_wait_pid(wo); if retval: return retval.
5. else:
   - tsk = current.
   - loop:
     - retval = Exit::do_wait_thread(wo, tsk); if retval: return retval.
     - retval = Exit::ptrace_do_wait(wo, tsk); if retval: return retval.
     - if wo.wo_flags & __WNOTHREAD: break.
     - tsk = while_each_thread(current, tsk).
6. tasklist_lock.read_unlock().
7. notask: retval = wo.notask_error.
8. if retval == 0 && !(wo.wo_flags & WNOHANG): return -ERESTARTSYS.
9. return retval.

`Exit::wait_consider_task(wo, ptrace, p) -> i64`:
1. let exit_state = READ_ONCE(p.exit_state).
2. if exit_state == EXIT_DEAD: return 0.
3. if !Exit::eligible_child(wo, ptrace, p): return 0.
4. if exit_state == EXIT_TRACE:
   - if !ptrace: wo.notask_error = 0.
   - return 0.
5. if !ptrace && p.ptrace != 0 && !ptrace_reparented(p): ptrace = 1.
6. if exit_state == EXIT_ZOMBIE && !delay_group_leader(p):
   - if ptrace != 0 || p.ptrace == 0: return Exit::wait_task_zombie(wo, p).
7. if !ptrace || (wo.wo_flags & (WCONTINUED | WEXITED)): wo.notask_error = 0.
8. let ret = Exit::wait_task_stopped(wo, ptrace, p); if ret: return ret.
9. return Exit::wait_task_continued(wo, p).

`Exit::wait_task_zombie(wo, p) -> i64`:
1. if !(wo.wo_flags & WEXITED): return 0.
2. if wo.wo_flags & WNOWAIT:
   - status = if p.signal.flags & SIGNAL_GROUP_EXIT { p.signal.group_exit_code } else { p.exit_code }.
   - get_task_struct(p); tasklist_lock.read_unlock().
   - if wo.wo_rusage: getrusage(p, RUSAGE_BOTH, wo.wo_rusage).
   - put_task_struct(p); goto out_info.
3. state = if ptrace_reparented(p) && thread_group_leader(p) { EXIT_TRACE } else { EXIT_DEAD }.
4. if cmpxchg(&p.exit_state, EXIT_ZOMBIE, state) != EXIT_ZOMBIE: return 0.
5. tasklist_lock.read_unlock().
6. if state == EXIT_DEAD && thread_group_leader(p):
   - (tgutime, tgstime) = thread_group_cputime_adjusted(p).
   - psig = current.signal.
   - psig.stats_lock.write_seqlock_irq().
   - psig.cutime += tgutime + p.signal.cutime.
   - psig.cstime += tgstime + p.signal.cstime.
   - psig.cgtime += task_gtime(p) + p.signal.gtime + p.signal.cgtime.
   - psig.cmin_flt += p.min_flt + p.signal.min_flt + p.signal.cmin_flt.
   - psig.cmaj_flt += p.maj_flt + p.signal.maj_flt + p.signal.cmaj_flt.
   - psig.cnvcsw / cnivcsw / cinblock / coublock += ...
   - psig.cmaxrss = max(psig.cmaxrss, p.signal.maxrss).
   - task_io_accounting_add(&psig.ioac, &p.signal.ioac).
   - psig.stats_lock.write_sequnlock_irq().
7. compose siginfo (SIGCHLD; CLD_EXITED/_KILLED/_DUMPED; si_status; si_pid; si_uid; si_utime; si_stime).
8. copy to wo.wo_info / wo.wo_stat / wo.wo_rusage.
9. if state == EXIT_DEAD: Exit::release_task(p).
10. return pid.

### Out of Scope

- arch-specific exit_thread (FPU/TLS/segment teardown) — covered in arch/x86 tier.
- kernel/signal.c get_signal / exit_signals body — covered in `signal.md` Tier-3.
- kernel/fork.c copy_process / dup_task_struct allocation — covered in `fork.md`.
- kernel/cgroup/cgroup.c cgroup_task_exit / cgroup_task_release — covered in cgroup tier.
- kernel/pid_namespace.c zap_pid_ns_processes — covered in pid_namespace tier.
- kernel/futex/futex_exit.c mm_release futex robust-list cleanup — covered in futex tier.
- io_uring_files_cancel / perf_event_exit_task internals — covered in their tier docs.
- kernel/audit.c audit_free — covered in audit tier.
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `do_exit(code)` | per-thread teardown | `Exit::do_exit` |
| `do_group_exit(code)` | per-tgid teardown | `Exit::do_group_exit` |
| `make_task_dead(signr)` | per-fatal-signal exit wrapper | `Exit::make_task_dead` |
| `synchronize_group_exit()` | per-group barrier on group_exit_task | `Exit::synchronize_group_exit` |
| `exit_signals(tsk)` | per-PF_EXITING signal retarget | `Signal::exit_signals` (kernel/signal.c) |
| `exit_mm()` | per-mm detach | `Exit::exit_mm` |
| `exit_files(tsk)` | per-fdtable drop | `Exit::exit_files` (fs/file.c) |
| `exit_fs(tsk)` | per-fs_struct drop | `Exit::exit_fs` (fs/fs_struct.c) |
| `exit_sem(tsk)` / `exit_shm(tsk)` | per-SysV IPC drop | `Exit::exit_sem` / `Exit::exit_shm` |
| `exit_thread(tsk)` | per-arch thread state free | `Exit::exit_thread` (arch) |
| `exit_task_namespaces(tsk)` / `exit_nsproxy_namespaces` | per-nsproxy drop | `Exit::exit_task_namespaces` |
| `exit_task_work(tsk)` | per-task_work_run final | `Exit::exit_task_work` |
| `exit_notify(tsk, group_dead)` | per-EXIT_ZOMBIE + SIGCHLD + autoreap | `Exit::exit_notify` |
| `forget_original_parent(father, dead)` | per-reparent children | `Exit::forget_original_parent` |
| `reparent_leader(father, p, dead)` | per-child reparent | `Exit::reparent_leader` |
| `find_new_reaper(father, reaper)` | per-reparent target picker | `Exit::find_new_reaper` |
| `find_child_reaper(father, dead)` | per-pid_ns init picker | `Exit::find_child_reaper` |
| `find_alive_thread(p)` | per-thread-group survivor | `Exit::find_alive_thread` |
| `kill_orphaned_pgrp(tsk, parent)` | per-orphan-pgrp SIGHUP+SIGCONT | `Exit::kill_orphaned_pgrp` |
| `release_task(p)` | per-task final unhash + free | `Exit::release_task` |
| `__exit_signal(post, tsk)` | per-task signal_struct accumulate + unhash | `Exit::exit_signal_inner` |
| `__cleanup_sighand(sighand)` | per-sighand kfree on last ref | `Exit::cleanup_sighand` |
| `__unhash_process(post, p, group_dead)` | per-task detach from lists | `Exit::unhash_process` |
| `delayed_put_task_struct(rhp)` | per-RCU put | `Exit::delayed_put_task_struct` |
| `put_task_struct_rcu_user(task)` | per-rcu-grace put | `Exit::put_task_struct_rcu_user` |
| `coredump_task_exit(tsk)` | per-coredump barrier | `Exit::coredump_task_exit` |
| `mm_update_next_owner(mm)` | per-mm.owner transfer | `Exit::mm_update_next_owner` |
| `__do_wait(wo)` | per-wait inner | `Exit::do_wait_inner` |
| `do_wait(wo)` | per-wait loop (sleep + retry) | `Exit::do_wait` |
| `do_wait_thread(wo, tsk)` | per-children iter | `Exit::do_wait_thread` |
| `ptrace_do_wait(wo, tsk)` | per-ptraced iter | `Exit::ptrace_do_wait` |
| `do_wait_pid(wo)` | per-PIDTYPE_PID fast path | `Exit::do_wait_pid` |
| `wait_consider_task(wo, ptrace, p)` | per-candidate eval | `Exit::wait_consider_task` |
| `wait_task_zombie(wo, p)` | per-zombie reap | `Exit::wait_task_zombie` |
| `wait_task_stopped(wo, ptrace, p)` | per-stopped report | `Exit::wait_task_stopped` |
| `wait_task_continued(wo, p)` | per-continued report | `Exit::wait_task_continued` |
| `eligible_child(wo, ptrace, p)` | per-WCLONE/WALL filter | `Exit::eligible_child` |
| `eligible_pid(wo, p)` | per-pid match | `Exit::eligible_pid` |
| `pid_child_should_wake(wo, p)` | per-wake filter | `Exit::pid_child_should_wake` |
| `child_wait_callback(wait, ...)` | per-wakeup cb | `Exit::child_wait_callback` |
| `__wake_up_parent(p, parent)` | per-wait_chldexit wake | `Exit::wake_up_parent` |
| `kernel_waitid_prepare(wo, which, ...)` | per-waitid arg parse | `Exit::waitid_prepare` |
| `kernel_waitid(which, ...)` | per-kernel waitid | `Exit::kernel_waitid` |
| `kernel_wait4(upid, ...)` | per-kernel wait4 | `Exit::kernel_wait4` |
| `kernel_wait(pid, stat)` | per-kernel wrapper | `Exit::kernel_wait` |
| SYSCALL_DEFINE1(exit) | per-exit(2) | `Exit::sys_exit` |
| SYSCALL_DEFINE1(exit_group) | per-exit_group(2) | `Exit::sys_exit_group` |
| SYSCALL_DEFINE5(waitid) | per-waitid(2) | `Exit::sys_waitid` |
| SYSCALL_DEFINE4(wait4) | per-wait4(2) | `Exit::sys_wait4` |
| SYSCALL_DEFINE3(waitpid) | per-waitpid(2) | `Exit::sys_waitpid` |
| `rcuwait_wake_up(w)` | per-rcuwait signaler | `Exit::rcuwait_wake_up` |
| `is_current_pgrp_orphaned()` | per-pgrp orphan probe | `Exit::is_current_pgrp_orphaned` |
| `has_stopped_jobs(pgrp)` | per-pgrp stopped probe | `Exit::has_stopped_jobs` |
| oops_limit / oops_count | per-make_task_dead panic-bomb | shared sysctl/sysfs |

### compatibility contract

REQ-1: Exit states (task_struct.exit_state) and task_state:
- 0 (running): normal.
- EXIT_ZOMBIE (0x0020): do_exit complete, awaiting wait()-reap by parent or autoreap.
- EXIT_DEAD (0x0010): selected for reap; release_task in flight.
- EXIT_TRACE (EXIT_ZOMBIE | EXIT_DEAD): ptracer reaped but real parent has not seen; transitional.
- TASK_DEAD (0x0080) in task->__state: set by do_task_dead; scheduler drops from runqueue.

REQ-2: do_exit(code) — single-thread teardown ordering:
1. WARN_ON(irqs_disabled()); WARN_ON(plug).
2. if kthread: kthread_do_exit(kthread, code) (noreturn).
3. kcov_task_exit; kmsan_task_exit.
4. synchronize_group_exit(tsk, code).
5. ptrace_event(PTRACE_EVENT_EXIT, code).
6. user_events_exit.
7. io_uring_files_cancel.
8. sched_mm_cid_exit.
9. exit_signals(tsk) /* sets PF_EXITING; retargets shared pending */.
10. seccomp_filter_release.
11. acct_update_integrals.
12. group_dead = atomic_dec_and_test(&tsk->signal->live).
13. if group_dead:
    - if is_global_init(tsk): panic("Attempted to kill init! exitcode=...").
    - CONFIG_POSIX_TIMERS: hrtimer_cancel(real_timer); exit_itimers.
    - if tsk->mm: setmax_mm_hiwater_rss(&signal->maxrss, mm).
14. acct_collect(code, group_dead).
15. if group_dead: tty_audit_exit.
16. audit_free(tsk).
17. tsk->exit_code = code.
18. taskstats_exit(tsk, group_dead).
19. trace_sched_process_exit(tsk, group_dead).
20. perf_event_exit_task(tsk).
21. unwind_deferred_task_exit(tsk).
22. exit_mm() /* drops mm; clears tsk->mm; mmput; if TIF_MEMDIE: exit_oom_victim */.
23. if group_dead: acct_process().
24. exit_sem(tsk); exit_shm(tsk).
25. exit_files(tsk) /* fdtable put */.
26. exit_fs(tsk) /* fs_struct put */.
27. if group_dead: disassociate_ctty(1).
28. exit_nsproxy_namespaces(tsk).
29. exit_task_work(tsk).
30. exit_thread(tsk) /* arch */.
31. sched_autogroup_exit_task.
32. cgroup_task_exit(tsk).
33. flush_ptrace_hw_breakpoint(tsk).
34. exit_tasks_rcu_start().
35. exit_notify(tsk, group_dead).
36. proc_exit_connector(tsk).
37. mpol_put_task_policy(tsk).
38. CONFIG_FUTEX: kfree(pi_state_cache).
39. debug_check_no_locks_held().
40. if io_context: exit_io_context.
41. if splice_pipe: free_pipe_info.
42. if task_frag.page: put_page.
43. exit_task_stack_account(tsk).
44. check_stack_usage().
45. preempt_disable.
46. if nr_dirtied: __this_cpu_add(dirty_throttle_leaks, nr_dirtied).
47. exit_rcu(); exit_tasks_rcu_finish().
48. lockdep_free_task(tsk).
49. do_task_dead() /* sets TASK_DEAD; __schedule(false); BUG; noreturn */.

REQ-3: do_group_exit(exit_code):
- sig = current->signal.
- if SIGNAL_GROUP_EXIT: exit_code = sig->group_exit_code.
- else if group_exec_task: exit_code = 0.
- else: siglock; recheck under lock; if neither: sig->group_exit_code = exit_code; sig->flags = SIGNAL_GROUP_EXIT; zap_other_threads(current); unlock.
- do_exit(exit_code) (noreturn).

REQ-4: SYSCALL exit(2) and exit_group(2):
- sys_exit(error_code): do_exit((error_code & 0xff) << 8).
- sys_exit_group(error_code): do_group_exit((error_code & 0xff) << 8).
- POSIX low-byte status encoded into bits 8..15 of wait status.

REQ-5: make_task_dead(signr):
- Called from fatal-fault handlers when current cannot survive (e.g. die in irq context, force_sig_seccomp force_coredump).
- WARN if in_interrupt + current task.
- if oops_count exceeds oops_limit: panic_on_oops behaviour.
- preempt_disable; sched + tracing barrier; do_exit(SIGKILL or signr).
- noreturn.

REQ-6: synchronize_group_exit(tsk, code):
- Wait for any other thread that initiated do_group_exit so we observe SIGNAL_GROUP_EXIT before reading group_exit_code.
- coredump_task_exit also synchronizes core_state participants.

REQ-7: exit_mm():
- mm = current->mm.
- exit_mm_release(current, mm) /* vfork wakeup, mm_release callbacks */.
- if !mm: return (kthread / pre-exec).
- mmap_read_lock(mm); mmgrab_lazy_tlb(mm); BUG_ON(mm != active_mm).
- task_lock(current).
- smp_mb__after_spinlock; local_irq_disable.
- current->mm = NULL.
- membarrier_update_current_mm(NULL).
- enter_lazy_tlb(mm, current).
- local_irq_enable.
- task_unlock(current); mmap_read_unlock(mm).
- mm_update_next_owner(mm).
- mmput(mm) /* drops mm_users; may free mm */.
- if TIF_MEMDIE: exit_oom_victim() /* release OOM budget */.

REQ-8: exit_notify(tsk, group_dead):
- LIST_HEAD(dead).
- write_lock_irq(tasklist_lock).
- forget_original_parent(tsk, &dead) /* reparent children; collect autoreap candidates */.
- if group_dead: kill_orphaned_pgrp(group_leader, NULL).
- tsk->exit_state = EXIT_ZOMBIE.
- if tsk->ptrace: sig = (thread_group_empty(tsk) && !ptrace_reparented(tsk)) ? exit_signal : SIGCHLD; autoreap = do_notify_parent(tsk, sig).
- else if thread_group_leader(tsk): autoreap = thread_group_empty(tsk) && do_notify_parent(tsk, exit_signal).
- else: autoreap = true; do_notify_pidfd(tsk).
- if autoreap: exit_state = EXIT_DEAD; list_add(&tsk->ptrace_entry, &dead).
- if signal->notify_count < 0: wake_up_process(signal->group_exec_task).
- write_unlock_irq.
- for p in dead: list_del_init(&p->ptrace_entry); release_task(p).

REQ-9: Autoreap conditions:
- Sub-thread (non-leader) exiting: always autoreap = true.
- Leader with parent ignoring SIGCHLD (SIG_IGN) or with SA_NOCLDWAIT: do_notify_parent returns true; autoreap.
- ptraced: tracer must wait() unless tracer detaches; otherwise EXIT_TRACE.
- Result: parent never sees zombie in wait4 — release_task runs from exit_notify itself.

REQ-10: forget_original_parent(father, dead):
- write_lock_irq(tasklist_lock) held by caller.
- reaper = find_child_reaper(father, dead) /* may release/re-acquire lock if pid_ns dies */.
- if !same_thread_group(reaper, father): reaper = find_new_reaper(father, reaper).
- for p in father->children: reparent_leader(father, p, dead) /* moves p->real_parent / parent / sibling lists; may autoreap on transition */.
- list_splice_tail(&father->ptraced, &reaper->ptraced) /* hand over tracees */.

REQ-11: find_new_reaper(father, child_reaper):
- /* Step 1: alive sibling in same thread group */
- thread = find_alive_thread(father); if thread: return thread.
- /* Step 2: walk ancestors for child_subreaper */
- for reaper = father->real_parent; reaper != child_reaper; reaper = reaper->real_parent:
  - if reaper->signal->is_child_subreaper:
    - thread = find_alive_thread(reaper); if thread: return thread.
- /* Step 3: pid_ns init */
- return child_reaper.

REQ-12: reparent_leader(father, p, dead):
- if !ptrace_reparented(p): p->exit_signal = SIGCHLD.
- p->real_parent = reaper; if !p->ptrace: p->parent = reaper.
- if p->exit_state == EXIT_ZOMBIE && thread_group_empty(p) && !ptrace_reparented(p):
  - if do_notify_parent(p, p->exit_signal): list_add(&p->ptrace_entry, dead) /* autoreap */.
- kill_orphaned_pgrp(p, father).

REQ-13: kill_orphaned_pgrp(tsk, parent):
- pgrp = task_pgrp(tsk).
- /* If pgrp becomes orphaned per POSIX (no outside member with same session and ppid in different pgrp) AND has stopped jobs */:
- if will_become_orphaned_pgrp(pgrp, tsk_subset) && has_stopped_jobs(pgrp):
  - __kill_pgrp_info(SIGHUP, SEND_SIG_PRIV, pgrp).
  - __kill_pgrp_info(SIGCONT, SEND_SIG_PRIV, pgrp).

REQ-14: release_task(p):
- repeat:
- memset(&post, 0, ...).
- dec_rlimit_ucounts(task_ucounts, UCOUNT_RLIMIT_NPROC, 1).
- pidfs_exit(p); cgroup_task_release(p).
- thread_pid = task_pid(p) /* before __unhash zeroes pids */.
- write_lock_irq(tasklist_lock).
- ptrace_release_task(p).
- __exit_signal(&post, p).
- zap_leader = 0; leader = p->group_leader.
- if leader != p && thread_group_empty(leader) && leader->exit_state == EXIT_ZOMBIE:
  - if leader->signal->flags & SIGNAL_GROUP_EXIT: leader->exit_code = signal->group_exit_code.
  - zap_leader = do_notify_parent(leader, leader->exit_signal).
  - if zap_leader: leader->exit_state = EXIT_DEAD.
- write_unlock_irq.
- proc_flush_pid(thread_pid).
- exit_cred_namespaces(p).
- add_device_randomness(&se.sum_exec_runtime, ...).
- free_pids(post.pids).
- release_thread(p) /* arch */.
- /* No more sighand; lock_task_sighand will fail */
- flush_sigqueue(&p->pending).
- if thread_group_leader(p): flush_sigqueue(&p->signal->shared_pending).
- put_task_struct_rcu_user(p).
- if zap_leader: p = leader; goto repeat.

REQ-15: __exit_signal(post, tsk):
- sig = tsk->signal; group_dead = thread_group_leader(tsk).
- sighand = rcu_dereference_check(tsk->sighand, lockdep_tasklist_lock_is_held).
- spin_lock(&sighand->siglock).
- CONFIG_POSIX_TIMERS: posix_cpu_timers_exit(tsk); if group_dead: posix_cpu_timers_exit_group(tsk).
- if group_dead: tty = sig->tty; sig->tty = NULL.
- else:
  - if sig->notify_count > 0 && !--sig->notify_count: wake_up_process(group_exec_task).
  - if tsk == sig->curr_target: sig->curr_target = next_thread(tsk).
- /* Accumulate cputime into signal_struct */
- task_cputime(tsk, &utime, &stime).
- write_seqlock(&sig->stats_lock).
- sig->utime += utime; stime += stime; gtime += task_gtime; min_flt + maj_flt + nvcsw + nivcsw + inblock + oublock + io accounting + sum_sched_runtime.
- sig->nr_threads--.
- __unhash_process(post, tsk, group_dead).
- write_sequnlock(&sig->stats_lock).
- tsk->sighand = NULL.
- spin_unlock(&sighand->siglock).
- __cleanup_sighand(sighand) /* refcount put; kfree on 0 */.
- if group_dead: tty_kref_put(tty).

REQ-16: __unhash_process(post, p, group_dead):
- detach_pid(post.pids, p, PIDTYPE_PID).
- if group_dead:
  - detach_pid(pids, p, PIDTYPE_TGID).
  - detach_pid(pids, p, PIDTYPE_PGID).
  - detach_pid(pids, p, PIDTYPE_SID).
  - list_del_rcu(&p->tasks).
  - list_del_init(&p->sibling).
  - __this_cpu_dec(process_counts).
- list_del_rcu(&p->thread_node).

REQ-17: put_task_struct_rcu_user(task):
- if refcount_dec_and_test(&task->rcu_users): call_rcu(&task->rcu, delayed_put_task_struct).
- delayed_put_task_struct(rhp): put_task_struct(task) /* drops final usage_count; frees task_struct via slab */.

REQ-18: struct wait_opts:
- wo_type: PIDTYPE_PID / PIDTYPE_TGID / PIDTYPE_PGID / PIDTYPE_MAX.
- wo_pid: struct pid pointer.
- wo_flags: WNOHANG | WUNTRACED/WSTOPPED | WCONTINUED | WEXITED | WNOWAIT | __WCLONE | __WALL | __WNOTHREAD.
- wo_info: struct waitid_info * (waitid).
- wo_stat: int * (wait4 stat_addr).
- wo_rusage: struct rusage * (optional).
- notask_error: starts -ECHILD; cleared when any candidate matched.
- child_wait: wait_queue_entry.

REQ-19: wait4(2) / waitpid(2) / waitid(2) syscall path:
- wait4(upid, stat, options, rusage): kernel_wait4 → wait_opts; do_wait; copy stat+rusage to user.
- waitpid(pid, stat, options): kernel_wait4(pid, stat, options, NULL).
- waitid(which, upid, infop, options, rusage): kernel_waitid_prepare → do_wait → copy_siginfo_to_user (siginfo_t with si_signo=SIGCHLD, si_code in {CLD_*}, si_status, si_utime, si_stime, si_pid, si_uid).

REQ-20: do_wait(wo):
- init child_wait waitqueue entry; add to current->signal->wait_chldexit.
- loop:
  - set_current_state(TASK_INTERRUPTIBLE).
  - retval = __do_wait(wo).
  - if retval != -ERESTARTSYS: break.
  - if signal_pending(current): break.
  - schedule().
- __set_current_state(TASK_RUNNING).
- remove_wait_queue.
- return retval.

REQ-21: __do_wait(wo):
- wo->notask_error = -ECHILD.
- if wo_type < PIDTYPE_MAX and (!wo_pid or !pid_has_task(wo_pid, wo_type)): goto notask.
- read_lock(tasklist_lock).
- if PIDTYPE_PID: retval = do_wait_pid(wo).
- else:
  - tsk = current; do { retval = do_wait_thread(wo, tsk); ... ; retval = ptrace_do_wait(wo, tsk); if __WNOTHREAD break; } while_each_thread.
- read_unlock(tasklist_lock).
- notask: retval = wo->notask_error; if !retval && !(WNOHANG): return -ERESTARTSYS.
- return retval.

REQ-22: wait_consider_task(wo, ptrace, p):
- exit_state = READ_ONCE(p->exit_state).
- if EXIT_DEAD: return 0 (already reaped).
- if !eligible_child(wo, ptrace, p): return 0.
- if EXIT_TRACE: if !ptrace: notask_error = 0; return 0.
- if !ptrace && p->ptrace && !ptrace_reparented(p): ptrace = 1 (real-parent pretends).
- if EXIT_ZOMBIE && !delay_group_leader(p):
  - if ptrace || !p->ptrace: return wait_task_zombie(wo, p).
- if (!ptrace) || (wo_flags & (WCONTINUED|WEXITED)): notask_error = 0.
- ret = wait_task_stopped(wo, ptrace, p); if ret: return ret.
- return wait_task_continued(wo, p).

REQ-23: wait_task_zombie(wo, p) — zombie reap:
- if !WEXITED: return 0.
- if WNOWAIT: read-only snapshot; copy status; return.
- /* claim the zombie */
- state = (ptrace_reparented(p) && thread_group_leader(p)) ? EXIT_TRACE : EXIT_DEAD.
- if cmpxchg(&p->exit_state, EXIT_ZOMBIE, state) != EXIT_ZOMBIE: return 0 (raced).
- read_unlock(tasklist_lock).
- /* Accumulate child cputime into parent's signal_struct (group leader) */
- if state == EXIT_DEAD && thread_group_leader(p):
  - thread_group_cputime_adjusted(p, &tgutime, &tgstime).
  - write_seqlock_irq(&psig->stats_lock).
  - psig->cutime/cstime/cgtime/cmin_flt/cmaj_flt/cnvcsw/cnivcsw/cinblock/coublock += ...
  - write_sequnlock_irq.
- copy exit status, rusage, siginfo to wo->wo_info / wo->wo_stat / wo->wo_rusage.
- if state == EXIT_DEAD: release_task(p).
- /* EXIT_TRACE remains: real parent will release on detach */
- return pid (via wo->wo_info->pid or stat written).

REQ-24: wait_task_stopped(wo, ptrace, p):
- if !(wo_flags & (WSTOPPED | WUNTRACED)): return 0.
- exit_code = task_stopped_code(p, ptrace) /* p->jobctl & JOBCTL_TRAPPING/STOP | __SIGRTMIN-shifted */.
- if !exit_code: return 0.
- if !WNOWAIT && !ptrace: p->exit_code = 0 (consume).
- compose CLD_TRAPPED / CLD_STOPPED siginfo; copy to user.
- return pid.

REQ-25: wait_task_continued(wo, p):
- if !(wo_flags & WCONTINUED): return 0.
- if !(signal->flags & SIGNAL_STOP_CONTINUED): return 0.
- if !WNOWAIT: signal->flags &= ~SIGNAL_STOP_CONTINUED.
- siginfo CLD_CONTINUED; copy to user.
- return pid.

REQ-26: eligible_child(wo, ptrace, p):
- if !eligible_pid(wo, p): return 0.
- if ptrace || (wo_flags & __WALL): return 1.
- /* __WCLONE inverts SIGCHLD test */
- if ((p->exit_signal != SIGCHLD) ^ !!(wo_flags & __WCLONE)): return 0.
- return 1.

REQ-27: child_wait_callback(wait, mode, sync, key):
- p = key; wo = container_of(wait, wait_opts, child_wait).
- if !pid_child_should_wake(wo, p): return 0.
- return default_wake_function(wait, mode, sync, key).

REQ-28: __wake_up_parent(p, parent):
- __wake_up_sync_key(&parent->signal->wait_chldexit, TASK_INTERRUPTIBLE, p).

REQ-29: rcuwait_wake_up(w):
- rcu_read_lock; task = rcu_dereference(w->task); if task: wake_up_process(task); rcu_read_unlock.

REQ-30: oops bomb (kernel.oops_limit sysctl):
- Each make_task_dead increments oops_count.
- If oops_count > oops_limit: invoke panic("Oopsing too often (~) - panic_on_oops").
- Exposed as sysctl kernel.oops_limit (default 10000) and sysfs kernel/oops_count.

REQ-31: PIDTYPE encoding:
- PIDTYPE_PID: single thread.
- PIDTYPE_TGID: thread group (process).
- PIDTYPE_PGID: process group.
- PIDTYPE_SID: session.
- PIDTYPE_MAX: any (waitid P_ALL).

REQ-32: Reaper protection:
- init (pid_ns->child_reaper) cannot be reaped while children exist.
- find_child_reaper: if reaper exiting and no alive sibling, calls zap_pid_ns_processes (recursive teardown).
- is_global_init: panic on init's last-thread exit (init must never die in root pid_ns).

REQ-33: ptrace + exit interaction:
- If a tracee dies while tracer attached: exit_signal forced to SIGCHLD via ptrace_reparented; tracer collects via wait. After tracer detaches/dies: real parent observes via EXIT_TRACE transition.

REQ-34: thread_group_leader semantics:
- Leader survives in EXIT_ZOMBIE until all sub-threads exit (delay_group_leader).
- Once last sub-thread runs release_task, it may set leader to EXIT_DEAD if leader is zombie and notify-parent permits.

REQ-35: WNOWAIT semantics:
- Read exit status without consuming the zombie: exit_state stays EXIT_ZOMBIE; subsequent wait() can reap.

REQ-36: -ERESTARTSYS handling in wait:
- If no candidate matched and no WNOHANG: -ERESTARTSYS returned.
- do_wait outer loop: signal_pending(current) breaks out (returns -ERESTARTSYS); arch decides restart vs EINTR per sa_flags.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `do_exit_pf_exiting_first` | INVARIANT | per-do_exit: PF_EXITING set via exit_signals before any subsystem teardown. |
| `do_exit_noreturn` | INVARIANT | per-do_exit: do_task_dead never returns; no path falls through. |
| `do_group_exit_zaps_under_siglock` | INVARIANT | per-do_group_exit: zap_other_threads under sighand.siglock. |
| `init_pid1_panic_on_last_thread` | INVARIANT | per-do_exit: is_global_init + last thread ⟹ panic. |
| `exit_state_transitions_monotone` | INVARIANT | per-task: Running → Zombie → (Dead ∨ Trace) → Dead; never reverse. |
| `release_task_holds_tasklist_write` | INVARIANT | per-release_task: __exit_signal under tasklist_lock write. |
| `exit_signal_clears_sighand_ptr` | INVARIANT | per-__exit_signal: tsk.sighand = NULL before __cleanup_sighand. |
| `cmpxchg_exit_state_zombie_to_dead` | INVARIANT | per-wait_task_zombie: only one reaper claims via cmpxchg. |
| `autoreap_implies_release` | INVARIANT | per-exit_notify: autoreap=true ⟹ release_task in dead list. |
| `reaper_never_self` | INVARIANT | per-find_new_reaper: result != exiting father. |
| `wait_consumes_zombie_unless_wnowait` | INVARIANT | per-wait_task_zombie: !WNOWAIT ⟹ exit_state transitions to EXIT_DEAD/_TRACE. |
| `notask_error_cleared_on_match` | INVARIANT | per-wait_consider_task: any eligible child ⟹ wo.notask_error = 0. |

### Layer 2: TLA+

`kernel/exit-lifecycle.tla`:
- States: Running → ExitInProgress → Zombie → {Dead, Trace} → Dead → Freed.
- Per-do_exit phases (signals, mm, files, fs, namespaces, notify, task_dead) modeled with order constraints.
- Per-wait/release race modeled (multiple waiters + autoreap).
- Per-reparent modeled (subreaper chain + init).
- Properties:
  - `safety_pid_unique_until_freed` — per-task: pid bound to task until release_task completes.
  - `safety_exit_state_monotone` — per-task: no backward transition.
  - `safety_init_never_dead` — per-pid_ns: child_reaper alive while any task in ns.
  - `safety_zombie_eventually_reaped` — per-zombie: ∃ parent or autoreap ⟹ release_task fires.
  - `safety_group_exit_kills_all_peers` — per-do_group_exit: every peer eventually enters do_exit.
  - `safety_release_task_double_reap_impossible` — per-cmpxchg: at most one reaper.
  - `liveness_wait_returns_on_child_event` — per-wait4: WEXITED + zombie ⟹ wait returns pid.
  - `liveness_orphan_subreap` — per-parent-exit: every child gets new real_parent.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Exit::do_exit` post: TASK_DEAD; no return | `Exit::do_exit` |
| `Exit::do_group_exit` post: SIGNAL_GROUP_EXIT set; all peers SIGKILL pending | `Exit::do_group_exit` |
| `Exit::exit_mm` post: current.mm = NULL; mmput called once | `Exit::exit_mm` |
| `Exit::exit_notify` post: exit_state ∈ {EXIT_ZOMBIE, EXIT_DEAD} | `Exit::exit_notify` |
| `Exit::release_task` post: pid unhashed; sighand freed; task RCU-scheduled for free | `Exit::release_task` |
| `Exit::find_new_reaper` post: returns alive thread in current ns | `Exit::find_new_reaper` |
| `Exit::wait_task_zombie` post: WNOWAIT preserves EXIT_ZOMBIE; else EXIT_DEAD/_TRACE | `Exit::wait_task_zombie` |
| `Exit::do_wait_inner` post: returns pid ∨ -ECHILD ∨ -ERESTARTSYS ∨ 0(WNOHANG) | `Exit::do_wait_inner` |

### Layer 4: Verus/Creusot functional

`Per-exit(2)/exit_group(2)/waitid(2)/wait4(2)/waitpid(2) semantic equivalence per Documentation/admin-guide/sysctl/kernel.rst (oops_limit), wait(2), waitid(2), POSIX.1-2017 §3.270 wait functions, prctl(2) PR_SET_CHILD_SUBREAPER, SA_NOCLDWAIT, ptrace(2) PTRACE_O_TRACEEXIT. Per-status word encoding: low 8 bits = signal-or-flag, next 8 bits = exit code (POSIX W*() macros).`

### hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

Exit/wait reinforcement:

- **Per-init pid 1 last-thread panic** — defense against per-userspace-init death silent hang.
- **Per-oops_limit / oops_count bomb** — defense against per-runaway-oops infinite-spam.
- **Per-PF_EXITING set early via exit_signals** — defense against per-mid-exit signal targeting.
- **Per-cmpxchg exit_state EXIT_ZOMBIE → EXIT_DEAD** — defense against per-double-reap race.
- **Per-tasklist_lock write around release_task** — defense against per-list-traversal UAF.
- **Per-RCU-deferred task_struct free (put_task_struct_rcu_user)** — defense against per-rcu-reader UAF.
- **Per-sighand refcount + __cleanup_sighand** — defense against per-shared-sighand premature free.
- **Per-find_new_reaper walks ancestors then init** — defense against per-orphan-with-no-reaper.
- **Per-zap_pid_ns_processes on pid_ns init death** — defense against per-pid-ns-leak.
- **Per-autoreap on SA_NOCLDWAIT/SIG_IGN** — defense against per-zombie-accumulation by careless daemons.
- **Per-WNOWAIT preserves zombie** — defense against per-status-peek consuming the zombie.
- **Per-thread_group_cputime accumulated under stats_lock seqlock** — defense against per-getrusage torn-read.
- **Per-do_notify_parent dropped-on-tracer-detach via EXIT_TRACE** — defense against per-tracer-vs-parent reap race.

