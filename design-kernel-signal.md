---
title: "Tier-3: kernel/signal.c — POSIX signal subsystem"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The **signal subsystem** delivers asynchronous notifications (POSIX signals SIGHUP..SIGSYS plus real-time SIGRTMIN..SIGRTMAX) between tasks, from kernel to task (faults, timers, child-exit), and from external sources (terminal driver, OOM, ptrace). Per-task signal state spans `task_struct.pending` (private sigqueue + bitmap) and `task_struct.signal->shared_pending` (thread-group-wide). Per-disposition tables live in `task_struct.sighand->action[64]` (shared by CLONE_SIGHAND threads). Per-blocked mask is `task_struct.blocked`. Per-`TIF_SIGPENDING` thread-flag drives kernel-exit signal scan; `recalc_sigpending` recomputes it. Per-`get_signal` (called from arch return-to-user) dequeues, checks disposition, invokes handler or default action. Per-real-time signals queue independently (one sigqueue per send, FIFO, lowest-numbered-first); per-standard signals coalesce (one bit, optional one sigqueue with siginfo). Per-handler restart via `-ERESTARTSYS` / `-ERESTARTNOHAND` / `-ERESTART_RESTARTBLOCK`. Per-sigaltstack lets handlers run on dedicated stack. Per-signalfd integrates pending signals as a file descriptor. Critical for: process control, fault delivery, job control, ptrace-stop, container/pidfd signaling.

This Tier-3 covers `kernel/signal.c` (~5053 lines).

### Acceptance Criteria

- [ ] AC-1: kill(2) to valid pid: signal queued; target wakes via TIF_SIGPENDING; default action applied if no handler.
- [ ] AC-2: SIGRTMIN sent N times: N sigqueues queued; dequeue in FIFO order; lowest RT number delivered before higher.
- [ ] AC-3: Standard signal sent N times while pending: bit set once; second sigqueue dropped (legacy_queue).
- [ ] AC-4: sigaction(SIGINT, SIG_IGN) then kill(pid, SIGINT): pending SIGINT discarded per POSIX 3.3.1.3.
- [ ] AC-5: sigprocmask(SIG_BLOCK, {SIGKILL,SIGSTOP}): kernel silently strips; mask cannot block them.
- [ ] AC-6: do_sigaction(SIGKILL, &act): -EINVAL.
- [ ] AC-7: rt_sigtimedwait(set, info, ts=100ms) with no pending: returns -EAGAIN after timeout.
- [ ] AC-8: sigsuspend(set): returns -ERESTARTNOHAND; saved mask restored on sigreturn.
- [ ] AC-9: Thread group exit (exit_group(2)): zap_other_threads sends SIGKILL to peers; all enter do_exit.
- [ ] AC-10: exec(): sighand action[] non-SIG_IGN entries reset to SIG_DFL; pending queues survive.
- [ ] AC-11: pidfd_send_signal(pidfd, SIGTERM, NULL, 0): sig delivered to whole tgid for non-thread pidfd.
- [ ] AC-12: SIGKILL bypasses SIG_IGN, blocked-mask, sigwaitinfo blocked set: still delivered.
- [ ] AC-13: SA_RESETHAND on handler entry: action reset to SIG_DFL atomically before user runs.
- [ ] AC-14: SA_NODEFER: signal not auto-added to handler-active blocked mask.
- [ ] AC-15: signalfd: read(2) returns siginfo for next pending signal in mask; clears bit.

### Architecture

```
struct SigHand {
  count: AtomicI32,
  action: [KSigAction; 64],
  siglock: SpinLockIrq,
  signalfd_wqh: WaitQueueHead,
}

struct SigPending {
  list: List<SigQueue>,
  signal: SigSet,                    // 64-bit bitmap
}

struct SigQueue {
  list: ListLink,
  flags: u32,                        // SIGQUEUE_PREALLOC | _TIMER
  info: KernelSigInfo,
  ucounts: *UCounts,
}

struct KSignal {
  ka: KSigAction,
  info: KernelSigInfo,
  sig: i32,
}

struct KSigAction {
  sa_handler: SigHandler,            // SIG_DFL / SIG_IGN / fn ptr
  sa_flags: u64,                     // SA_RESTART | SA_NODEFER | SA_RESETHAND | SA_SIGINFO | SA_ONSTACK | SA_IMMUTABLE | ...
  sa_restorer: Option<UserPtr>,
  sa_mask: SigSet,
}
```

`Signal::send_signal_locked_inner(sig, info, t, type, force) -> Result<(), Errno>`:
1. lockdep_assert held(t.sighand.siglock).
2. /* Disposition + group-exit + ignore */
3. if !Signal::prepare_signal(sig, t, force): trace(IGNORED); return Ok(()).
4. pending = if type != PIDTYPE_PID { &t.signal.shared_pending } else { &t.pending }.
5. /* Standard coalesce */
6. if Signal::legacy_queue(pending, sig): trace(ALREADY_PENDING); return Ok(()).
7. /* SIGKILL + PF_KTHREAD shortcut */
8. if sig == SIGKILL || t.flags & PF_KTHREAD: goto out_set.
9. /* RT override */
10. override_rlimit = if sig < SIGRTMIN { is_si_special(info) || info.si_code >= 0 } else { false }.
11. q = Signal::sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit).
12. match q:
    - Some(q): list_add_tail(&q.list, &pending.list); fill info per SEND_SIG_NOINFO / _PRIV / copy_siginfo.
    - None if sig >= SIGRTMIN && !is_si_special(info) && info.si_code != SI_USER: trace(OVERFLOW_FAIL); return Err(EAGAIN).
    - None: trace(LOSE_INFO).
13. out_set: Signal::signalfd_notify(t, sig); sigaddset(&pending.signal, sig).
14. /* Multiprocess (fork-in-flight) propagate */
15. if type > PIDTYPE_TGID: iterate t.signal.multiprocess and merge.
16. Signal::complete_signal(sig, t, type).
17. return Ok(()).

`Signal::dequeue_signal(mask, info, type_out) -> i32`:
1. lockdep_assert held(current.sighand.siglock).
2. loop:
   - *type_out = PIDTYPE_PID; timer_sigq = None.
   - signr = Signal::dequeue_signal_inner(&current.pending, mask, info, &mut timer_sigq).
   - if !signr:
     - *type_out = PIDTYPE_TGID.
     - signr = Signal::dequeue_signal_inner(&current.signal.shared_pending, mask, info, &mut timer_sigq).
     - if signr == SIGALRM: posixtimer_rearm_itimer(current).
   - Signal::recalc_sigpending().
   - if !signr: return 0.
   - if sig_kernel_stop(signr): current.jobctl |= JOBCTL_STOP_DEQUEUED.
   - if timer_sigq:
     - if !posixtimer_deliver_signal(info, timer_sigq): continue.
   - return signr.

`Signal::get_signal(ksig) -> bool`:
1. clear_notify_signal(); if task_work_pending: task_work_run().
2. if !task_sigpending(current): return false.
3. if uprobe_deny_signal(): return false.
4. try_to_freeze().
5. relock: siglock.lock_irq().
6. if signal.flags & SIGNAL_CLD_MASK:
   - why = CLD_CONTINUED or CLD_STOPPED.
   - signal.flags &= !SIGNAL_CLD_MASK.
   - unlock; read_lock(tasklist); do_notify_parent_cldstop(current, false, why); if ptrace_reparented(leader): do_notify_parent_cldstop(leader, true, why); read_unlock; goto relock.
7. loop:
   - if SIGNAL_GROUP_EXIT || group_exec_task: signr = SIGKILL; sigdelset(SIGKILL); recalc_sigpending(); goto fatal.
   - if JOBCTL_STOP_PENDING && do_signal_stop(0): goto relock.
   - if JOBCTL_TRAP_MASK | JOBCTL_TRAP_FREEZE: handle; goto relock.
   - if frozen-leaving: update cgroup; recalc_sigpending.
   - signr = Signal::dequeue_signal(&current.blocked, &ksig.info, &type).
   - if !signr: break.
   - if current.ptrace: signr = ptrace_signal(signr, &ksig.info, type); if !signr: continue.
   - ka = &sighand.action[signr-1].
   - if ka.sa_handler == SIG_IGN: continue.
   - if ka.sa_handler != SIG_DFL:
     - ksig.ka = *ka; if SA_RESETHAND: ka.sa_handler = SIG_DFL; hide_si_addr_tag_bits(ksig).
     - unlock; ksig.sig = signr; return true.
   - /* SIG_DFL */
   - if sig_kernel_ignore(signr): continue.
   - if sig_kernel_stop(signr) && !is_global_init(current.group_leader): unlock; do_signal_stop(signr); goto relock.
   - if sig_kernel_coredump(signr): unlock; do_coredump(ksig); relock; record exit_code; goto fatal.
   - /* terminate */
   - goto fatal.
8. fatal: unlock; do_group_exit(signr) (noreturn).
9. unlock; return false.

`Signal::do_sigaction(sig, act, oact) -> Result<(), Errno>`:
1. if !valid_signal(sig) || (act.is_some() && sig_kernel_only(sig)): return Err(EINVAL).
2. k = &current.sighand.action[sig-1].
3. siglock.
4. if k.sa_flags & SA_IMMUTABLE: unlock; return Err(EINVAL).
5. if oact: *oact = *k.
6. BUILD_BUG(UAPI_SA_FLAGS & SA_UNSUPPORTED == 0).
7. if act: act.sa_flags &= UAPI_SA_FLAGS.
8. if oact: oact.sa_flags &= UAPI_SA_FLAGS.
9. sigaction_compat_abi(act, oact).
10. if act:
    - was_ignored = (k.sa_handler == SIG_IGN).
    - sigdelsetmask(&act.sa_mask, sigmask(SIGKILL) | sigmask(SIGSTOP)).
    - *k = *act.
    - if sig_handler_ignored(sig_handler(p, sig), sig):
      - mask = singleton(sig).
      - flush_sigqueue_mask(p, &mask, &p.signal.shared_pending).
      - for each t in thread group: flush_sigqueue_mask(p, &mask, &t.pending).
    - else if was_ignored: posixtimer_sig_unignore(p, sig).
11. unlock; return Ok(()).

`Signal::sigprocmask(how, newset) -> sigset_t /* old */`:
1. old = current.blocked.
2. match how:
   - SIG_BLOCK: new = old | newset.
   - SIG_UNBLOCK: new = old & !newset.
   - SIG_SETMASK: new = newset.
3. sigdelsetmask(&new, sigmask(SIGKILL) | sigmask(SIGSTOP)).
4. Signal::set_current_blocked(&new).
5. return old.

`Signal::set_current_blocked(newset)`:
1. sigdelsetmask(&newset, sigmask(SIGKILL) | sigmask(SIGSTOP)).
2. siglock.
3. Signal::set_task_blocked(current, &newset).
4. unlock.

`Signal::set_task_blocked(tsk, newset)`:
1. /* If unblocking shared signals, retarget back to self */
2. retarget = shared_pending.signal & (tsk.blocked & ~newset).
3. tsk.blocked = *newset.
4. Signal::recalc_sigpending_tsk(tsk).
5. if !sigisemptyset(&retarget): retarget_shared_pending(tsk, &retarget).

`Signal::recalc_sigpending_tsk(t) -> bool`:
1. if (t.jobctl & JOBCTL_PENDING_MASK) || PENDING(&t.pending, &t.blocked) || PENDING(&t.signal.shared_pending, &t.blocked):
   - set_tsk_thread_flag(t, TIF_SIGPENDING); return true.
2. return false.

`Signal::recalc_sigpending()`:
1. if !Signal::recalc_sigpending_tsk(current) && !freezing(current):
   - if test_thread_flag(TIF_SIGPENDING): clear_thread_flag(TIF_SIGPENDING).

`Signal::do_sigaltstack(ss, oss, sp, min_ss_size) -> Result<(), Errno>`:
1. if oss: oss.ss_sp = current.sas_ss_sp; oss.ss_size = current.sas_ss_size; oss.ss_flags = sas_ss_flags(sp) | (current.sas_ss_flags & SS_FLAG_BITS).
2. if !ss: return Ok(()).
3. if ss.ss_flags & ~(SS_DISABLE | SS_AUTODISARM | SS_ONSTACK): return Err(EINVAL).
4. if on_sig_stack(sp): return Err(EPERM).
5. if ss.ss_flags & SS_DISABLE:
   - current.sas_ss_sp = 0; sas_ss_size = 0; sas_ss_flags = SS_DISABLE.
6. else:
   - if ss.ss_size < min_ss_size: return Err(ENOMEM).
   - install ss; sas_ss_flags = ss.ss_flags & SS_FLAG_BITS.
7. Ok(()).

`Signal::do_sigtimedwait(which, info, ts) -> i32`:
1. sigdelsetmask(&which, sigmask(SIGKILL) | sigmask(SIGSTOP)).
2. signotset(&mask) /* invert */.
3. siglock.
4. signr = Signal::dequeue_signal(&mask, info, &type).
5. if signr: unlock; return signr.
6. if ts && ts.is_zero: unlock; return -EAGAIN.
7. timeout = if ts: ts; else: forever.
8. current.real_blocked = which (allow these to wake us).
9. signotset(&current.blocked) etc.; recalc_sigpending.
10. unlock; schedule_hrtimeout(timeout, HRTIMER_MODE_REL); /* sleeps TASK_INTERRUPTIBLE */.
11. siglock; sigemptyset(&current.real_blocked); recalc_sigpending; retry dequeue.
12. signr = Signal::dequeue_signal(&mask, info, &type).
13. unlock; return signr or -EAGAIN.

`Signal::exit_signals(tsk)`:
1. cgroup_threadgroup_change_begin(tsk).
2. if thread_group_empty(tsk) || (tsk.signal.flags & SIGNAL_GROUP_EXIT):
   - tsk.flags |= PF_EXITING; cgroup_threadgroup_change_end; return.
3. siglock.
4. tsk.flags |= PF_EXITING.
5. cgroup_threadgroup_change_end.
6. if !task_sigpending(tsk): unlock; return.
7. unblocked = !tsk.blocked.
8. retarget_shared_pending(tsk, &unblocked).
9. if (tsk.jobctl & JOBCTL_STOP_PENDING) && task_participate_group_stop(tsk): group_stop = CLD_STOPPED.
10. unlock.
11. if group_stop: read_lock(tasklist); do_notify_parent_cldstop(tsk, false, group_stop); read_unlock.

`Signal::retarget_shared_pending(tsk, which)`:
1. for sig in which:
   - if !sigismember(&shared_pending.signal, sig): continue.
   - /* Find thread that doesn't block sig */
   - for t in thread_group, t != tsk:
     - if !sigismember(&t.blocked, sig):
       - signal_wake_up(t, false); break.

### Out of Scope

- arch/x86/kernel/signal_*.c sigframe setup / restore_sigcontext / rt_sigreturn (covered in arch tier).
- fs/signalfd.c file_operations (covered separately).
- kernel/sys.c reboot / kexec signal-disable (covered separately).
- kernel/ptrace.c ptrace_signal upstream of get_signal (covered in ptrace.md).
- kernel/time/posix-timers.c POSIX timer fire path (covered in posix-timers.md).
- kernel/fork.c copy_signal/copy_sighand (covered in fork.md).
- kernel/exec.c flush_signal_handlers (covered in exec.md / exit.md).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sighand_struct` | per-thread-group disposition table | `SigHand` |
| `struct sigpending` | per-queue (signal bitmap + sigqueue list) | `SigPending` |
| `struct sigqueue` | per-queued siginfo | `SigQueue` |
| `struct ksignal` | per-delivery handoff to arch | `KSignal` |
| `struct k_sigaction` | per-disposition slot | `KSigAction` |
| `recalc_sigpending()` | per-mask TIF_SIGPENDING recompute | `Signal::recalc_sigpending` |
| `recalc_sigpending_tsk()` | per-task variant | `Signal::recalc_sigpending_tsk` |
| `calculate_sigpending()` | per-fork tail | `Signal::calculate_sigpending` |
| `next_signal()` | per-pending lowest-prio pick | `Signal::next_signal` |
| `dequeue_signal()` | per-syscall return dequeue | `Signal::dequeue_signal` |
| `__dequeue_signal()` | per-queue inner | `Signal::dequeue_signal_inner` |
| `collect_signal()` | per-queue siginfo collect | `Signal::collect_signal` |
| `flush_sigqueue()` | per-queue free | `Signal::flush_sigqueue` |
| `flush_signals()` | per-task purge | `Signal::flush_signals` |
| `flush_sigqueue_mask()` | per-mask purge | `Signal::flush_sigqueue_mask` |
| `ignore_signals()` | per-exec disposition reset | `Signal::ignore_signals` |
| `prepare_signal()` | per-send disposition check | `Signal::prepare_signal` |
| `wants_signal()` | per-thread eligibility | `Signal::wants_signal` |
| `complete_signal()` | per-send target+wake | `Signal::complete_signal` |
| `__send_signal_locked()` | per-send queue+post | `Signal::send_signal_locked_inner` |
| `send_signal_locked()` | per-send entry (siglock held) | `Signal::send_signal_locked` |
| `do_send_sig_info()` | per-send lock+route | `Signal::do_send_sig_info` |
| `send_sig_info()` / `send_sig()` | per-send public | `Signal::send_sig_info` |
| `force_sig()` / `force_sig_info()` | per-fault unblockable | `Signal::force_sig` |
| `force_sig_fault()` | per-fault siginfo (SIGSEGV/SIGBUS) | `Signal::force_sig_fault` |
| `force_sig_mceerr()` | per-MCE memory-failure siginfo | `Signal::force_sig_mceerr` |
| `force_sig_seccomp()` | per-seccomp-violation | `Signal::force_sig_seccomp` |
| `zap_other_threads()` | per-exec/exit-group kill peers | `Signal::zap_other_threads` |
| `group_send_sig_info()` | per-tgid send | `Signal::group_send_sig_info` |
| `kill_pid_info()` / `kill_pgrp()` / `kill_pid()` | per-pid/pgrp send | `Signal::kill_pid_info` etc. |
| `kill_something_info()` | per-kill(2) dispatch | `Signal::kill_something_info` |
| `do_pidfd_send_signal()` | per-pidfd_send_signal(2) | `Signal::pidfd_send_signal` |
| `do_tkill()` | per-tkill/tgkill(2) | `Signal::do_tkill` |
| `lock_task_sighand()` | per-rcu-safe siglock | `Signal::lock_task_sighand` |
| `do_sigaction()` | per-sigaction(2) | `Signal::do_sigaction` |
| `do_sigaltstack()` | per-sigaltstack(2) | `Signal::do_sigaltstack` |
| `do_sigtimedwait()` | per-rt_sigtimedwait(2) | `Signal::do_sigtimedwait` |
| `do_sigpending()` | per-rt_sigpending(2) | `Signal::do_sigpending` |
| `sigprocmask()` | per-mask change | `Signal::sigprocmask` |
| `set_current_blocked()` | per-mask install + recalc | `Signal::set_current_blocked` |
| `set_user_sigmask()` | per-pselect/ppoll mask | `Signal::set_user_sigmask` |
| `sigsuspend()` | per-rt_sigsuspend(2) | `Signal::sigsuspend` |
| `get_signal()` | per-return-to-user dequeue+act | `Signal::get_signal` |
| `signal_delivered()` | per-handler-done mask install | `Signal::signal_delivered` |
| `signal_setup_done()` | per-arch setup_frame finish | `Signal::signal_setup_done` |
| `exit_signals()` | per-task PF_EXITING signal retarget | `Signal::exit_signals` |
| `retarget_shared_pending()` | per-exit shared-queue retarget | `Signal::retarget_shared_pending` |
| `signalfd_notify()` | per-send signalfd wakeup | `Signal::signalfd_notify` |
| `signal_wake_up_state()` | per-send wake-target | `Signal::signal_wake_up_state` |
| `check_kill_permission()` | per-send cred check | `Signal::check_kill_permission` |
| `kill_ok_by_cred()` | per-cred-match | `Signal::kill_ok_by_cred` |
| `do_notify_parent()` | per-exit SIGCHLD to parent | `Signal::do_notify_parent` |
| `do_notify_parent_cldstop()` | per-stop/cont SIGCHLD | `Signal::do_notify_parent_cldstop` |
| `ptrace_stop()` / `ptrace_notify()` | per-ptrace event | `Signal::ptrace_stop` |
| `do_signal_stop()` | per-SIGSTOP group-stop | `Signal::do_signal_stop` |
| `posixtimer_send_sigqueue()` | per-POSIX-timer fire | `Signal::posixtimer_send_sigqueue` |
| `unhandled_signal()` | per-default-fatal classify | `Signal::unhandled_signal` |
| `restore_altstack()` / `__save_altstack()` | per-sigreturn stack | `Signal::restore_altstack` |
| sigqueue_cachep slab | per-allocator | `Signal::sigqueue_slab` |

### compatibility contract

REQ-1: Signal numbers and ranges:
- 1..31: standard signals (SIGHUP..SIGSYS), non-queued (coalesce on pending bitmap).
- 32..33: kernel-reserved (SIGCANCEL/SIGSETXID, glibc-internal).
- SIGRTMIN..SIGRTMAX (typically 34..64): real-time, queued, FIFO ordered, ANSI-priority delivery (lowest number first).
- valid_signal(sig): 1 <= sig <= _NSIG (64).
- sig_kernel_only(sig): SIGKILL (9), SIGSTOP (19) — cannot be caught/blocked/ignored.

REQ-2: struct sighand_struct:
- count: refcount (CLONE_SIGHAND share).
- action[_NSIG]: per-signal k_sigaction (handler + sa_mask + sa_flags + sa_restorer).
- siglock: spinlock guarding action[], task->pending, signal->shared_pending, signal->flags.
- signalfd_wqh: per-signalfd waitqueue.

REQ-3: struct sigpending:
- list: linked list of struct sigqueue (siginfo carriers).
- signal: sigset_t bitmap (one bit per pending signal).

REQ-4: struct sigqueue:
- list: link in sigpending.list.
- flags: SIGQUEUE_PREALLOC (POSIX-timer) / SIGQUEUE_PREALLOC | SIGQUEUE_TIMER.
- info: kernel_siginfo_t.
- ucounts: rlimit accounting (RLIMIT_SIGPENDING).
- Allocated from sigqueue_cachep slab via sigqueue_alloc(); freed via __sigqueue_free.

REQ-5: TIF_SIGPENDING and recalc:
- Set when a signal is queued and !blocked && !ignored && wants_signal(t, sig).
- recalc_sigpending_tsk(t): if has_pending_signals(&t->pending.signal, &t->blocked) || has_pending(&t->signal->shared_pending.signal, &t->blocked) || JOBCTL_PENDING_MASK: set TIF_SIGPENDING else clear.
- recalc_sigpending(): same for current; if freezing() also set.
- calculate_sigpending(): post-fork, set TIF_SIGPENDING then recompute under siglock.

REQ-6: send_signal_locked(sig, info, t, type):
- type: PIDTYPE_PID (per-thread), PIDTYPE_TGID (per-process), PIDTYPE_PGID (per-group), PIDTYPE_MAX (per-multiprocess).
- Calls __send_signal_locked(sig, info, t, type, force=false).
- siglock of t->sighand must be held.

REQ-7: __send_signal_locked(sig, info, t, type, force):
- result = TRACE_SIGNAL_IGNORED; if !prepare_signal(sig, t, force): goto ret (silent drop).
- pending = (type != PIDTYPE_PID) ? &t->signal->shared_pending : &t->pending.
- if legacy_queue(pending, sig): goto ret (standard sig already pending, no extra sigqueue).
- if sig == SIGKILL || PF_KTHREAD: set bit only, no sigqueue (out_set path).
- if sig < SIGRTMIN: override_rlimit = is_si_special(info) || info->si_code >= 0.
- else: override_rlimit = 0 (RT signals must respect RLIMIT_SIGPENDING).
- q = sigqueue_alloc(sig, t, GFP_ATOMIC, override_rlimit).
- if q: list_add_tail(&q->list, &pending->list); fill q->info per SEND_SIG_NOINFO / SEND_SIG_PRIV / copy_siginfo.
- else if sig >= SIGRTMIN && si_code != SI_USER && !is_si_special: return -EAGAIN (RT overflow).
- else: TRACE_SIGNAL_LOSE_INFO (silent info loss for standard signals).
- out_set: signalfd_notify(t, sig); sigaddset(&pending->signal, sig).
- if type > PIDTYPE_TGID: propagate to delayed multiprocess_signals (CLONE in-flight).
- complete_signal(sig, t, type).

REQ-8: prepare_signal(sig, t, force):
- if t->signal->flags & SIGNAL_GROUP_EXIT && sig != SIGKILL: return false.
- if sig_kernel_stop(sig): clear pending SIGCONT.
- if sig == SIGCONT: clear pending stop signals; SIGNAL_CLD_CONTINUED.
- if !force && sig_task_ignored(t, sig, force): return false (SIG_IGN or default-ignore + standard).
- return true.

REQ-9: complete_signal(sig, t, type):
- /* Pick which thread should wake to handle */
- if wants_signal(sig, t): pass.
- else if type == PIDTYPE_PID: return (only this thread).
- else: iterate threads to find wants_signal; if none: return.
- /* Fatal? */
- if sig_fatal(t, sig) && !(t->signal->flags & SIGNAL_GROUP_EXIT) && !sig_kernel_coredump(sig) && !is_global_init(t->group_leader): mark SIGNAL_GROUP_EXIT, zap_other_threads.
- signal_wake_up(t, sig == SIGKILL).

REQ-10: wants_signal(sig, p):
- !sigismember(&p->blocked, sig) && !(p->flags & PF_EXITING) && (sig == SIGKILL || !task_is_stopped_or_traced(p)) && task_curr(p) || !signal_pending(p).

REQ-11: dequeue_signal(mask, info, type):
- siglock held.
- signr = __dequeue_signal(&current->pending, mask, info, &timer_sigq); type = PIDTYPE_PID.
- if !signr: signr = __dequeue_signal(&current->signal->shared_pending, ...); type = PIDTYPE_TGID.
- recalc_sigpending().
- if sig_kernel_stop(signr): jobctl |= JOBCTL_STOP_DEQUEUED.
- if posixtimer-queued: posixtimer_deliver_signal() (may loop again).
- return signr.

REQ-12: __dequeue_signal(pending, mask, info, timer_sigq):
- sig = next_signal(pending, mask).
- if sig: collect_signal(sig, pending, info, timer_sigq).
- return sig.

REQ-13: next_signal(pending, mask):
- /* Per-POSIX: synchronous signals first, then real-time lowest, then standard lowest */
- s = pending->signal; m = *mask.
- sigandnsets(&x, &s, &m).
- if first 32 bits has SIGSYNCHRONOUS_MASK: pick lowest.
- else if RT range nonzero (33..64): pick lowest RT.
- else: pick lowest standard.

REQ-14: collect_signal(sig, list, info, timer_sigq):
- iterate list->list, first matching sig: q.
- if q: copy_siginfo(info, &q->info); list_del(&q->list); __sigqueue_free(q) or save timer.
- if no q (bit-only): info filled with SI_USER / SI_KERNEL synthesized.
- sigdelset(&list->signal, sig) only if no other queued instance remains (RT may have more).

REQ-15: get_signal(ksig) — return-to-user main loop:
1. clear_notify_signal(); if task_work_pending: task_work_run().
2. if !task_sigpending(current): return false.
3. uprobe_deny_signal check.
4. try_to_freeze().
5. relock: spin_lock_irq(&sighand->siglock).
6. SIGNAL_CLD_MASK: notify parent of stop/cont, goto relock.
7. for (;;):
   - if SIGNAL_GROUP_EXIT || group_exec_task: signr = SIGKILL; goto fatal.
   - if JOBCTL_STOP_PENDING: do_signal_stop(0); goto relock.
   - if JOBCTL_TRAP_MASK | TRAP_FREEZE: do_jobctl_trap / do_freezer_trap; goto relock.
   - if frozen: clear bit and recalc.
   - signr = dequeue_signal(&blocked, &ksig->info, &type).
   - if !signr: break (no actionable).
   - if PTRACE: signr = ptrace_signal(signr, &ksig->info, type); if !signr: continue.
   - ka = &sighand->action[signr-1].
   - if ka->sa.sa_handler == SIG_IGN: continue.
   - if ka->sa.sa_handler != SIG_DFL: ksig->ka = *ka; hide_si_addr_tag_bits; if !(SA_NODEFER): nothing; if (SA_RESETHAND): ka->sa.sa_handler = SIG_DFL; goto user (return true).
   - /* SIG_DFL */
   - if sig_kernel_ignore(signr): continue.
   - if sig_kernel_stop(signr) && !is_global_init: do_signal_stop(signr); goto relock.
   - /* Default-terminate / default-core */
   - if sig_kernel_coredump(signr): unlock; do_coredump(); relock; record exit_code.
   - goto fatal: do_group_exit(signr).
8. spin_unlock_irq(&sighand->siglock); return false.
9. user-path: spin_unlock_irq; return true (ksig populated).

REQ-16: signal_delivered(ksig, stepping):
- clear_restore_sigmask().
- blocked = current->blocked | ksig->ka.sa.sa_mask.
- if !(SA_NODEFER): sigaddset(&blocked, ksig->sig).
- set_current_blocked(&blocked).
- if sas_ss_flags & SS_AUTODISARM: sas_ss_reset.
- if stepping: ptrace_notify(SIGTRAP, 0).

REQ-17: do_sigaction(sig, act, oact):
- if !valid_signal(sig) || sig_kernel_only(sig)+act: return -EINVAL.
- siglock; if SA_IMMUTABLE: -EINVAL.
- if oact: *oact = *k.
- act->sa_flags &= UAPI_SA_FLAGS (clear unknown bits).
- if act: was_ignored = (handler == SIG_IGN); sigdelsetmask(&sa_mask, sigmask(SIGKILL)|sigmask(SIGSTOP)); *k = *act.
- if new handler is ignore-effective: flush_sigqueue_mask of pending (shared + per-thread) (POSIX 3.3.1.3).
- else if was_ignored: posixtimer_sig_unignore(p, sig).

REQ-18: do_sigaltstack(ss, oss, sp, min_ss_size):
- Output oss: ss_sp, ss_size, ss_flags from current.sas_ss_*.
- Input ss: validate flags (SS_DISABLE | SS_AUTODISARM | SS_ONSTACK).
- If !SS_DISABLE: validate ss_size >= MINSIGSTKSZ; install.
- If on_sig_stack(sp): cannot change/disable: -EPERM.

REQ-19: sigprocmask(how, set, oldset) and rt_sigprocmask(2):
- how: SIG_BLOCK / SIG_UNBLOCK / SIG_SETMASK.
- oldset = current->blocked.
- newset computed.
- sigdelsetmask(&newset, sigmask(SIGKILL)|sigmask(SIGSTOP)).
- __set_current_blocked(&newset).
- __set_task_blocked: blocked = *new; recalc_sigpending; if pending unblocked: retarget shared queue back to current.

REQ-20: sigsuspend(set):
- current->saved_sigmask = current->blocked.
- set_current_blocked(set).
- TASK_INTERRUPTIBLE schedule loop until signal arrives.
- on signal: set_restore_sigmask() so sigreturn / signal_delivered restores saved.
- return -ERESTARTNOHAND.

REQ-21: do_sigtimedwait(which, info, ts):
- siglock; signr = dequeue_signal(&which, info); if signr: return signr.
- else: TASK_INTERRUPTIBLE; schedule_hrtimeout(ts).
- on wake: re-acquire siglock; retry dequeue; if timeout: -EAGAIN.

REQ-22: kill(2) — kill_something_info:
- pid > 0: kill_pid_info(sig, info, find_vpid(pid)).
- pid == 0: kill_pgrp(task_pgrp(current), sig, 0).
- pid == -1: iterate all tasks (broadcast) except init/current.
- pid < -1: kill_pgrp(find_vpid(-pid), sig, 0).

REQ-23: pidfd_send_signal(2) — do_pidfd_send_signal:
- f = fdget(pidfd).
- pid = pidfd_to_pid(file).
- type = PIDTYPE_TGID for pidfd; PIDTYPE_PID with PIDFD_THREAD; PIDTYPE_PGID with PIDFD_PGID.
- copy_siginfo_from_user_any(&kinfo, info); validate si_pid == 0 || matches.
- check_kill_permission via kill_pid_info_type(sig, &kinfo, pid, type).

REQ-24: tkill/tgkill — do_tkill(tgid, pid, sig):
- info: SI_TKILL with current uid/pid.
- p = find_get_task_by_vpid(pid).
- if tgid > 0 && task_tgid_vnr(p) != tgid: -ESRCH.
- do_send_sig_info(sig, &info, p, PIDTYPE_PID).

REQ-25: rt_sigqueueinfo(2) and rt_tgsigqueueinfo(2):
- copy_siginfo_from_user(&info, uinfo).
- if SI_TKILL/SI_USER and not from kernel cred: -EPERM.
- info.si_signo = sig.
- kill_proc_info / do_send_specific(tgid, pid, sig, &info).

REQ-26: check_kill_permission(sig, info, t):
- if !valid_signal(sig): -EINVAL.
- if si_fromuser(info):
  - if !kill_ok_by_cred(t): -EPERM (UID/EUID check unless CAP_KILL).
  - if same session/ancestor pgrp + SIGCONT: bypass cred check.
- audit_signal_info.

REQ-27: force_sig family (synchronous fault):
- force_sig_info(info): siglock; if blocked or ignored: clear SIG_IGN -> SIG_DFL, unblock, raise SA_NODEFER bit; deliver guaranteed.
- force_sig_fault(sig, code, addr): si_signo=sig, si_code=code, si_addr=addr.
- force_sig_mceerr(code, addr, lsb): SIGBUS BUS_MCEERR_*; lsb = least-significant-bit of corrupted page span.
- force_sig_seccomp(syscall, reason, force_coredump): SIGSYS si_code=SYS_SECCOMP; if force: also SIGNAL_UNKILLABLE bypass.
- force_sigsegv(sig): used by signal_setup_done on frame setup failure.

REQ-28: zap_other_threads(p):
- /* Called from do_group_exit / de_thread */
- for each t in thread group, t != p: sigaddset(&t->pending.signal, SIGKILL); signal_wake_up(t, 1).
- return count of zapped.

REQ-29: exit_signals(tsk) — invoked early in do_exit:
- cgroup_threadgroup_change_begin.
- if thread_group_empty || SIGNAL_GROUP_EXIT: PF_EXITING; end; return.
- siglock; PF_EXITING; end.
- if !task_sigpending: out.
- unblocked = ~tsk->blocked (signotset).
- retarget_shared_pending(tsk, &unblocked): for each unblocked-pending in shared, find another thread that doesn't block it, transfer.
- if JOBCTL_STOP_PENDING && participate: group_stop = CLD_STOPPED.
- out: unlock; if group_stop: do_notify_parent_cldstop.

REQ-30: signalfd_notify(t, sig):
- wake_up_poll(&t->sighand->signalfd_wqh, EPOLLIN) so signalfd readers see new pending bit.

REQ-31: -ERESTART* handling (returned from interrupted syscalls):
- ERESTARTSYS: restart if no handler or SA_RESTART set; else EINTR.
- ERESTARTNOINTR: always restart (kernel-internal).
- ERESTARTNOHAND: restart only if no handler installed; else EINTR.
- ERESTART_RESTARTBLOCK: restart via SYS_restart_syscall + current->restart_block (used by clock_nanosleep, epoll_wait, etc.).
- get_signal / arch signal_setup_done propagates back to syscall return; arch sets %rax / r0 / ... before iret.

REQ-32: do_notify_parent(tsk, sig):
- /* Send SIGCHLD to parent on exit */
- info: SIGCHLD; si_code = CLD_EXITED/_KILLED/_DUMPED; si_pid=pid; si_uid; si_status; si_utime; si_stime.
- if parent ignores SIGCHLD (SIG_IGN) or SA_NOCLDWAIT: return true (autoreap).
- send to parent + wake; return false.

REQ-33: do_signal_stop(signr):
- siglock held.
- /* Per-job-control */
- if first thread to enter SIGSTOP: SIGNAL_STOP_STOPPED; group_stop_count = #threads-not-stopped; signal SIGNAL_CLD_STOPPED.
- TASK_STOPPED schedule; on resume return 1 (retry get_signal).

REQ-34: ptrace_signal(signr, info, type):
- /* PT-tracer intercept */
- ptrace_stop with signr; tracer may zero / change signr / inject info.
- if tracer cleared signr: skip.
- return new signr.

REQ-35: SIGKILL / SIGSTOP cannot be:
- blocked (sigprocmask strips them).
- ignored (do_sigaction rejects).
- caught (sig_kernel_only excludes).

REQ-36: Default actions table (sig_kernel_*):
- term: SIGHUP SIGINT SIGPIPE SIGALRM SIGTERM SIGUSR1 SIGUSR2 SIGPROF SIGVTALRM SIGIO SIGPOLL.
- core: SIGQUIT SIGILL SIGABRT SIGFPE SIGSEGV SIGBUS SIGSYS SIGTRAP SIGXCPU SIGXFSZ.
- ignore: SIGCHLD SIGURG SIGWINCH SIGPWR.
- stop: SIGSTOP SIGTSTP SIGTTIN SIGTTOU.
- cont: SIGCONT.

REQ-37: Per-fork/exec inheritance:
- fork (copy_signal/copy_sighand): CLONE_SIGHAND -> share; else copy action[] from parent; pending queues fresh per-thread, shared empty.
- exec (flush_signal_handlers): each non-SIG_IGN handler reset to SIG_DFL; SIG_IGN preserved; sa_flags cleared except SA_ONSTACK; sigaltstack preserved per arch.

REQ-38: signalfd integration:
- fs/signalfd.c reads &current->signal->shared_pending OR &current->pending and dequeues via dequeue_signal under siglock with overridden mask.
- signalfd_notify on every send keeps poll() current.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `send_signal_holds_siglock` | INVARIANT | per-__send_signal_locked: t.sighand.siglock held. |
| `dequeue_signal_holds_siglock` | INVARIANT | per-dequeue_signal: siglock held on entry. |
| `sigkill_never_blocked` | INVARIANT | per-set_current_blocked: SIGKILL/SIGSTOP stripped from mask. |
| `sigkill_never_ignored` | INVARIANT | per-do_sigaction: sig_kernel_only rejected. |
| `rt_signal_queue_overflow_eagain` | INVARIANT | per-__send_signal_locked: RT sig + user sender + alloc fail ⟹ -EAGAIN. |
| `legacy_signal_coalesce` | INVARIANT | per-standard-sig second send: no extra sigqueue allocated. |
| `tif_sigpending_recomputed` | INVARIANT | per-mask change / send / dequeue: TIF_SIGPENDING reflects has_pending(pending, blocked). |
| `sigqueue_balance` | INVARIANT | per-sigqueue: alloc ↔ __sigqueue_free balanced (no leak). |
| `signalfd_notify_per_send` | INVARIANT | per-out_set: signalfd_notify(t, sig) called before complete_signal. |
| `force_sig_bypasses_block` | INVARIANT | per-force_sig_info: blocked/SIG_IGN cleared before deliver. |
| `kill_perm_checked` | INVARIANT | per-kill_pid_info: check_kill_permission invoked. |
| `exit_signals_pf_exiting_set` | INVARIANT | per-exit_signals: PF_EXITING set under siglock if !thread_group_empty. |

### Layer 2: TLA+

`kernel/signal-routing.tla`:
- Per-send (kill/tkill/pidfd/sigqueueinfo) → __send_signal_locked → complete_signal → wakeup.
- Per-dequeue (get_signal) → handler dispatch / default action / stop / coredump / fatal.
- Per-fork copy → per-exec flush.
- Properties:
  - `safety_sigkill_always_eventually_delivered` — per-SIGKILL: target enters get_signal → fatal path.
  - `safety_rt_queue_fifo` — per-SIGRTMIN+i sequential sends: dequeue in same order.
  - `safety_standard_coalesce` — per-N sends of same standard sig: at most 1 sigqueue + 1 bit.
  - `safety_kill_stop_uncatchable` — per-SIGKILL/SIGSTOP: do_sigaction rejects; mask strips.
  - `safety_handler_mask_install` — per-handler return (sigreturn): blocked mask = pre + sa_mask + (sig unless SA_NODEFER).
  - `liveness_dequeue_advances` — per-pending: TIF_SIGPENDING ⟹ eventually get_signal dequeues or exits.
  - `liveness_sigsuspend_returns_on_signal` — per-sigsuspend: signal arrival ⟹ returns -ERESTARTNOHAND.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Signal::send_signal_locked_inner` post: returns Ok ∨ EAGAIN(RT-overflow); pending bit set on success | `Signal::send_signal_locked_inner` |
| `Signal::dequeue_signal` post: signr in [1,_NSIG] ∨ 0; *type ∈ {PIDTYPE_PID, PIDTYPE_TGID} | `Signal::dequeue_signal` |
| `Signal::recalc_sigpending` post: TIF_SIGPENDING ⟺ has_pending_signals ∨ JOBCTL_PENDING_MASK ∨ freezing | `Signal::recalc_sigpending` |
| `Signal::do_sigaction` post: action[sig-1] = *act; pending sig flushed if new disposition ignores | `Signal::do_sigaction` |
| `Signal::sigprocmask` post: current.blocked = computed; SIGKILL/SIGSTOP cleared | `Signal::sigprocmask` |
| `Signal::get_signal` post: returns true ⟹ ksig.ka.sa_handler is user fn ∧ sig delivered; returns false ⟹ no pending or do_group_exit invoked | `Signal::get_signal` |
| `Signal::exit_signals` post: PF_EXITING set ∧ shared pending retargeted | `Signal::exit_signals` |
| `Signal::zap_other_threads` post: every other thread in group has SIGKILL pending ∧ TIF_SIGPENDING | `Signal::zap_other_threads` |

### Layer 4: Verus/Creusot functional

`Per-kill(2)/sigaction(2)/sigprocmask(2)/sigsuspend(2)/sigtimedwait(2)/rt_sigreturn semantic equivalence per Documentation/admin-guide/signal.rst, POSIX.1-2017 §2.4 Signal Concepts, signal(7), sigaction(2), rt_sigtimedwait(2). Per-RT queue ordering invariant: dequeue order = send order for same signo; lowest signo first across signos.`

### hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

Signal reinforcement:

- **Per-SIGKILL/SIGSTOP uncatchable** — defense against per-runaway-task immortality.
- **Per-do_sigaction SA_IMMUTABLE** — defense against per-libc-internal-handler override.
- **Per-check_kill_permission strict cred match** — defense against per-cross-user signal injection.
- **Per-RLIMIT_SIGPENDING enforced for RT signals** — defense against per-sigqueue slab exhaustion.
- **Per-RT overflow -EAGAIN for user senders** — defense against per-DoS via rt_sigqueueinfo flood.
- **Per-sigqueue ucounts accounting** — defense against per-cred-quota bypass.
- **Per-sig_kernel_only blocks block/ignore/catch of SIGKILL,SIGSTOP** — defense against per-init-immortal lock-in.
- **Per-force_sig clears SIG_IGN/blocked-mask before deliver** — defense against per-fault unhandled-by-design.
- **Per-flush_sigqueue_mask on SIG_IGN install** — defense against per-stale-pending leak (POSIX 3.3.1.3).
- **Per-siglock IRQ-safe + lockdep_assert** — defense against per-interrupt re-entry corruption.
- **Per-signalfd_notify before pending-bit set order** — defense against per-poll-missed-wake.
- **Per-multiprocess delayed signal propagation on fork-in-flight** — defense against per-fork-race signal loss.
- **Per-pidfd cred capture at open** — defense against per-pid-reuse signaling wrong task.

