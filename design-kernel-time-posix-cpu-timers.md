---
title: "Tier-3: kernel/time/posix-cpu-timers.c — POSIX CPU-time clocks and timers"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

POSIX defines three families of CPU-time clocks: per-thread (`CLOCK_THREAD_CPUTIME_ID` and any `MAKE_THREAD_CPUCLOCK(tid, *)`), per-process / thread-group (`CLOCK_PROCESS_CPUTIME_ID` and any `MAKE_PROCESS_CPUCLOCK(pid, *)`), and three sub-clocks each: `CPUCLOCK_PROF` (utime + stime), `CPUCLOCK_VIRT` (utime only), `CPUCLOCK_SCHED` (`task_sched_runtime` ≈ `sum_exec_runtime`). `kernel/time/posix-cpu-timers.c` plugs these into the generic posix-timers framework via three `struct k_clock` instances (`clock_posix_cpu` / `clock_process` / `clock_thread`) and provides the seven operations the posix-timers core dispatches to: `clock_getres`, `clock_set`, `clock_get_timespec`, `timer_create`, `nsleep`, `timer_set`, `timer_del`, `timer_get`, `timer_rearm`, `timer_wait_running`.

Per-task / per-thread-group expiry state lives in `struct posix_cputimers` (three `posix_cputimer_base` slots, one per CPUCLOCK_*; each has a `timerqueue_head tqhead` and a `u64 nextevt` cache). For per-process timers, the thread-group's running totals are aggregated atomically into `signal_struct::cputimer.cputime_atomic` (`task_cputime_atomic`: utime / stime / sum_exec_runtime) so that any thread of the group can sample them lock-free under the `timers_active` enable flag. Itimers (`setitimer(2)`) for `ITIMER_PROF` / `ITIMER_VIRTUAL` are stored in `signal_struct::it[CPUCLOCK_*]` and share the same expiry-cache machinery; `RLIMIT_CPU` and `RLIMIT_RTTIME` are also checked from this path.

Expiry is driven by `run_posix_cpu_timers()`, called from the timer-interrupt path (`update_process_times`) on each tick: it does a fast-path `expiry_cache_is_inactive` check, then on a hit either runs `handle_posix_cpu_timers` directly (`!CONFIG_POSIX_CPU_TIMERS_TASK_WORK`) or defers expiry to `task_work` (RT-safe). `handle_posix_cpu_timers` walks per-thread + per-process timer queues, moves expired timers to a private `firing` list under `sighand->lock`, then fires each via `cpu_timer_fire` → `posix_timer_queue_signal` which uses `send_signal_locked` to deliver the timer's `sigev_signo`. Critical for: `timer_create(2)`+`timer_settime(2)` with CPU clocks, `setitimer(ITIMER_PROF|_VIRTUAL)`, `RLIMIT_CPU` enforcement, `clock_nanosleep(CLOCK_PROCESS_CPUTIME_ID, ...)`.

This Tier-3 covers `kernel/time/posix-cpu-timers.c` (~1670 lines).

### Acceptance Criteria

- [ ] AC-1: clock_gettime(CLOCK_PROCESS_CPUTIME_ID, &ts) returns sum of utime + stime + sum_exec_runtime over the calling thread group (within sched_clock resolution).
- [ ] AC-2: clock_gettime(CLOCK_THREAD_CPUTIME_ID, &ts) returns the calling thread's sum_exec_runtime (CPUCLOCK_SCHED).
- [ ] AC-3: clock_getres(CLOCK_PROCESS_CPUTIME_ID, &ts) returns tv_sec=0 tv_nsec=1.
- [ ] AC-4: clock_settime on any CPU clock returns -EPERM.
- [ ] AC-5: timer_create with clockid embedding pid=0 + CPUCLOCK_PROF creates a per-process PROF timer.
- [ ] AC-6: timer_settime(.., TIMER_ABSTIME, ..) treats it_value as absolute against the underlying clock; relative timers are converted by += now.
- [ ] AC-7: timer_settime with sigev_notify == SIGEV_NONE does not arm but updates trigger_base_recalc_expires.
- [ ] AC-8: A timer whose new_expires ≤ now fires once immediately from posix_cpu_timer_set (before returning).
- [ ] AC-9: timer_gettime on an expired interval timer reports a remaining time ≥ 1ns until the signal is delivered (non-SIGEV_NONE).
- [ ] AC-10: setitimer(ITIMER_PROF, &it, NULL) updates signal.it[CPUCLOCK_PROF].{expires,incr} and delivers SIGPROF on expiry from check_cpu_itimer.
- [ ] AC-11: setitimer(ITIMER_VIRTUAL, ...) delivers SIGVTALRM.
- [ ] AC-12: RLIMIT_CPU soft-limit hit ⟹ SIGXCPU sent every second (rlim_cur bumped by 1 per send).
- [ ] AC-13: RLIMIT_CPU hard-limit hit ⟹ SIGKILL sent.
- [ ] AC-14: RLIMIT_RTTIME (real-time tasks) soft / hard delivers SIGXCPU / SIGKILL.
- [ ] AC-15: DL task with dl_overrun set ⟹ SIGXCPU delivered from check_dl_overrun.
- [ ] AC-16: fastpath_timer_check returns false when expiry_cache_is_inactive on both per-task and per-group + !dl_overrun.
- [ ] AC-17: run_posix_cpu_timers is a no-op while current.exit_state != 0.
- [ ] AC-18: With CONFIG_POSIX_CPU_TIMERS_TASK_WORK, __run_posix_cpu_timers defers via task_work_add(.., TWA_RESUME) and re-runs on return-to-userspace.
- [ ] AC-19: clock_nanosleep(CLOCK_THREAD_CPUTIME_ID, ..) (PERTHREAD with self) returns -EINVAL.
- [ ] AC-20: clock_nanosleep(CLOCK_PROCESS_CPUTIME_ID, TIMER_ABSTIME, ..) on a signal returns -ERESTARTNOHAND (no restart for absolute).
- [ ] AC-21: clock_nanosleep on a signal with relative timer returns -ERESTART_RESTARTBLOCK and the kernel re-enters via posix_cpu_nsleep_restart.
- [ ] AC-22: posix_cpu_timer_del on a firing timer returns TIMER_RETRY; the posix-timers core retries after wait_running.
- [ ] AC-23: bump_cpu_timer increments it_overrun by exactly floor((now+incr-expires)/incr) and updates expires.
- [ ] AC-24: posix_cpu_timer_exit drains tsk.posix_cputimers timer queues without re-arming any.

### Architecture

```
struct PosixCputimerBase {
  tqhead: TimerQueueHead,
  nextevt: AtomicU64,            // U64_MAX = inactive
}

struct PosixCputimers {
  bases: [PosixCputimerBase; CPUCLOCK_MAX],   // PROF, VIRT, SCHED
  timers_active: bool,
  expiry_active: bool,
}

struct ThreadGroupCputimer {
  cputime_atomic: TaskCputimeAtomic,   // {utime, stime, sum_exec_runtime}: 3x AtomicU64
}

struct CpuTimer {
  node: TimerQueueNode,
  head: *mut TimerQueueHead,
  pid: *mut Pid,
  elist: ListHead,                     // firing list linkage
  firing: bool,
  nanosleep: bool,
  handling: *mut TaskStruct,           // rcu pointer set by collect_timerqueue
}

struct PosixCputimersWork {        // per task_struct, only with CONFIG_POSIX_CPU_TIMERS_TASK_WORK
  work: CallbackHead,
  mutex: Mutex,
  scheduled: bool,
}
```

`PosixCpu::clock_get(clk, tp)`:
1. Rcu::read_lock.
2. tsk = pid_task(pid_for_clock(clk, gettime=true), clock_pid_type(clk)).
3. if tsk.is_null(): Rcu::read_unlock; return -EINVAL.
4. clkid = CPUCLOCK_WHICH(clk).
5. t = if CPUCLOCK_PERTHREAD(clk) { sample_task(clkid, tsk) } else { sample_group(clkid, tsk, start=false) }.
6. Rcu::read_unlock.
7. *tp = ns_to_timespec64(t); return 0.

`PosixCpu::sample_task(clkid, p)`:
1. if clkid == CPUCLOCK_SCHED: return task_sched_runtime(p).
2. task_cputime(p, &utime, &stime).
3. match clkid:
   - CPUCLOCK_PROF: utime + stime.
   - CPUCLOCK_VIRT: utime.
   - _: WARN_ON_ONCE; 0.

`PosixCpu::sample_group(clkid, p, start)`:
1. pct = &p.signal.posix_cputimers.
2. if !READ_ONCE(pct.timers_active):
   - if start: group_start_cputime(p, &samples) — flips timers_active to true.
   - else: group_read(p, &samples) — non-activating read via thread_group_cputime.
3. Else: store_samples(&samples, atomics read from signal.cputimer.cputime_atomic).
4. Return samples[clkid].

`PosixCpu::timer_create(new_timer)`:
1. Rcu::read_lock.
2. pid = pid_for_clock(new_timer.it_clock, gettime=false).
3. if pid.is_null(): Rcu::read_unlock; return -EINVAL.
4. if cfg(CONFIG_POSIX_CPU_TIMERS_TASK_WORK): lockdep_set_class(&new_timer.it_lock, &posix_cpu_timers_key).
5. new_timer.kclock = &CLOCK_POSIX_CPU.
6. timerqueue_init(&new_timer.it.cpu.node).
7. new_timer.it.cpu.pid = get_pid(pid).
8. Rcu::read_unlock; return 0.

`PosixCpu::timer_set(timer, flags, new, old) -> i32`:
1. Rcu::read_lock; p = task_rcu(timer); if p.is_null(): Rcu::read_unlock; return -ESRCH.
2. new_expires = ktime_to_ns(timespec64_to_ktime(new.it_value)).
3. let _g = lock_task_sighand(p)?; if let None = _g: Rcu::read_unlock; return -ESRCH.
4. old_expires = cpu_timer_getexpires(&timer.it.cpu).
5. if timer.it.cpu.firing: timer.it.cpu.firing = false; ret = TIMER_RETRY.
6. else: cpu_timer_dequeue(&timer.it.cpu); timer.it_status = POSIX_TIMER_DISARMED.
7. now = if CPUCLOCK_PERTHREAD(timer.it_clock) { sample_task(clkid, p) } else { sample_group(clkid, p, !sigev_none) }.
8. if let Some(old) = old:
   - old.it_value = TimespecZero.
   - if old_expires != 0: __timer_get(timer, old, now).
9. if ret == TIMER_RETRY: drop(_g); goto out.
10. if new_expires != 0 && !(flags & TIMER_ABSTIME): new_expires += now.
11. cpu_timer_setexpires(&timer.it.cpu, new_expires).
12. if !sigev_none:
    - if new_expires != 0 && now < new_expires: arm_timer(timer, p).
    - else: trigger_recalc(timer, p).
13. drop(_g).
14. posix_timer_set_common(timer, new).
15. if !sigev_none && new_expires != 0 && now ≥ new_expires: fire(timer).
16. out: Rcu::read_unlock; return ret.

`PosixCpu::handle(tsk)`:
1. let _g = lock_task_sighand(tsk)?.
2. loop:
   - start = READ_ONCE(jiffies); barrier().
   - check_thread(tsk, &mut firing).
   - check_process(tsk, &mut firing).
   - if enable_work(tsk, start): break.
3. drop(_g).
4. for timer in firing.iter_safe():
   - let _tl = timer.it_lock.lock().
   - list_del_init(&timer.it.cpu.elist).
   - cpu_firing = timer.it.cpu.firing; timer.it.cpu.firing = false.
   - if cpu_firing: fire(timer).
   - rcu_assign_pointer(&timer.it.cpu.handling, ptr::null_mut()).

`PosixCpu::fire(timer)`:
1. timer.it_status = POSIX_TIMER_DISARMED.
2. if timer.it.cpu.nanosleep:
   - wake_up_process(timer.it_process).
   - cpu_timer_setexpires(&timer.it.cpu, 0).
3. else:
   - posix_timer_queue_signal(timer).
   - if timer.it_interval == 0: cpu_timer_setexpires(&timer.it.cpu, 0).

`PosixCpu::run_per_tick()`:
1. tsk = current.
2. lockdep_assert_irqs_disabled.
3. if tsk.exit_state != 0: return.
4. if work_scheduled(tsk): return.
5. if !fastpath(tsk): return.
6. dispatch(tsk).

`PosixCpu::dispatch(tsk)` (CONFIG_POSIX_CPU_TIMERS_TASK_WORK):
1. WARN_ON_ONCE(tsk.posix_cputimers_work.scheduled) and return on hit.
2. tsk.posix_cputimers_work.scheduled = true.
3. task_work_add(tsk, &tsk.posix_cputimers_work.work, TWA_RESUME).

`PosixCpu::dispatch(tsk)` (!CONFIG_POSIX_CPU_TIMERS_TASK_WORK):
1. lockdep_posixtimer_enter; handle(tsk); lockdep_posixtimer_exit.

`PosixCpu::set_process_timer(tsk, clkid, newval, oldval)`:
1. WARN_ON_ONCE(clkid ≥ CPUCLOCK_SCHED).
2. nextevt = &mut tsk.signal.posix_cputimers.bases[clkid].nextevt.
3. now = sample_group(clkid, tsk, start=true).
4. if let Some(oldv) = oldval:
   - if *oldv != 0: if *oldv ≤ now { *oldv = TICK_NSEC } else { *oldv -= now }.
   - if *newval != 0: *newval += now.
5. if *newval < *nextevt: *nextevt = *newval.
6. tick_dep_set_signal(tsk, TICK_DEP_BIT_POSIX_TIMER).

`PosixCpu::do_nsleep(clkid, flags, rqtp) -> i32`:
1. let mut timer: KItimer = zeroed; spin_lock_init(&timer.it_lock); timer.it_clock = clkid; timer.it_overrun = -1.
2. timer_create(&mut timer)?.
3. timer.it_process = current; timer.it.cpu.nanosleep = true.
4. let mut it = ItimerSpec64::zero; it.it_value = *rqtp.
5. let mut g = timer.it_lock.lock_irq().
6. timer_set(&mut timer, flags, &it, None)?.
7. while !signal_pending(current):
   - if cpu_timer_getexpires(&timer.it.cpu) == 0:
     - timer_del(&mut timer); drop(g); return 0.
   - set_current_state(TASK_INTERRUPTIBLE); drop(g); schedule(); g = timer.it_lock.lock_irq().
8. /* signal */
9. expires = cpu_timer_getexpires(&timer.it.cpu).
10. err = timer_set(&mut timer, 0, &zero_it, Some(&mut it)).
11. if err == 0: timer_del(&mut timer).
12. else while err == TIMER_RETRY: wait_running_nsleep(&mut timer); err = timer_del(&mut timer).
13. drop(g).
14. if it.it_value == 0: return 0.
15. error = -ERESTART_RESTARTBLOCK; restart.nanosleep.expires = ns_to_ktime(expires); copy remaining.

### Out of Scope

- kernel/time/posix-timers.c generic posix-timer core (covered separately if expanded)
- kernel/time/hrtimer.c hrtimer infrastructure (covered in `hrtimer.md` Tier-3)
- kernel/time/timer.c legacy jiffy timer wheel (covered in `timer.md` Tier-3)
- kernel/time/alarmtimer.c RTC-wakeup timers (covered separately if expanded)
- kernel/time/timekeeping.c wall-clock / CLOCK_MONOTONIC maintenance (covered separately if expanded)
- kernel/signal.c send_signal_locked / posix_timer_queue_signal internals (covered in `kernel/signal.md` Tier-3 if expanded)
- kernel/sched/cputime.c task_cputime / task_sched_runtime accounting hooks (covered in `kernel/sched/cputime.md` Tier-3 if expanded)
- include/linux/posix-timers.h struct definitions and helper inlines (data-only)
- glibc / musl userspace wrappers
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct k_clock clock_posix_cpu` | per-arbitrary-pid k_clock vtable | `PosixCpu::CLOCK_POSIX_CPU` |
| `struct k_clock clock_process` | CLOCK_PROCESS_CPUTIME_ID vtable | `PosixCpu::CLOCK_PROCESS` |
| `struct k_clock clock_thread` | CLOCK_THREAD_CPUTIME_ID vtable | `PosixCpu::CLOCK_THREAD` |
| `posix_cpu_clock_getres()` | per-clock getres | `PosixCpu::clock_getres` |
| `posix_cpu_clock_set()` | per-clock set (returns EPERM) | `PosixCpu::clock_set` |
| `posix_cpu_clock_get()` | per-clock current time | `PosixCpu::clock_get` |
| `posix_cpu_timer_create()` | per-timer create | `PosixCpu::timer_create` |
| `posix_cpu_timer_set()` | per-timer settime guts | `PosixCpu::timer_set` |
| `posix_cpu_timer_get()` / `__posix_cpu_timer_get()` | per-timer gettime | `PosixCpu::timer_get` / `_inner` |
| `posix_cpu_timer_del()` | per-timer delete | `PosixCpu::timer_del` |
| `posix_cpu_timer_rearm()` | per-timer interval reload | `PosixCpu::timer_rearm` |
| `posix_cpu_timer_wait_running()` | per-cancel wait-on-firing | `PosixCpu::wait_running` |
| `posix_cpu_timer_wait_running_nsleep()` | per-nsleep firing-wait | `PosixCpu::wait_running_nsleep` |
| `posix_cpu_nsleep()` / `do_cpu_nanosleep()` / `posix_cpu_nsleep_restart()` | per-nsleep entry / body / restart | `PosixCpu::nsleep` / `do_nsleep` / `nsleep_restart` |
| `cpu_clock_sample()` | per-task sample for clkid | `PosixCpu::sample_task` |
| `cpu_clock_sample_group()` | per-thread-group sample for clkid | `PosixCpu::sample_group` |
| `store_samples()` / `task_sample_cputime()` / `proc_sample_cputime_atomic()` | per-pack utime/stime/runtime | `PosixCpu::{store,sample_task_cputime,sample_proc_atomic}` |
| `thread_group_start_cputime()` / `__thread_group_cputime()` | per-group cputime activate / read | `PosixCpu::group_start_cputime` / `_read` |
| `update_gt_cputime()` / `__update_gt_cputime()` | per-cmpxchg monotone bump | `PosixCpu::update_gt_cputime` |
| `pid_for_clock()` / `validate_clock_permissions()` / `clock_pid_type()` / `cpu_timer_task_rcu()` | per-clock pid resolution | `PosixCpu::{pid_for_clock,validate_perms,pid_type,task_rcu}` |
| `bump_cpu_timer()` | per-interval overrun accounting | `PosixCpu::bump_timer` |
| `expiry_cache_is_inactive()` | per-pct U64_MAX check | `PosixCpu::cache_is_inactive` |
| `timer_base()` | per-timer pct base picker | `PosixCpu::timer_base` |
| `trigger_base_recalc_expires()` | per-base recalc trigger | `PosixCpu::trigger_recalc` |
| `disarm_timer()` | per-timer dequeue + maybe-recalc | `PosixCpu::disarm` |
| `cleanup_timers()` / `cleanup_timerqueue()` | per-exit drain | `PosixCpu::cleanup` / `_queue` |
| `posix_cpu_timers_exit()` / `_exit_group()` | per-exit hook | `PosixCpu::exit_task` / `_exit_group` |
| `arm_timer()` | per-timer enqueue + cache update | `PosixCpu::arm_timer` |
| `cpu_timer_fire()` | per-expiry signal/wake | `PosixCpu::fire` |
| `collect_timerqueue()` / `collect_posix_cputimers()` | per-tick expiry sweep | `PosixCpu::collect_queue` / `_pct` |
| `check_thread_timers()` / `check_process_timers()` | per-tick per-task / per-group check | `PosixCpu::check_thread` / `_process` |
| `check_dl_overrun()` | per-DL-task SIGXCPU | `PosixCpu::check_dl` |
| `check_rlimit()` | per-RLIMIT_CPU / _RTTIME check | `PosixCpu::check_rlimit` |
| `check_cpu_itimer()` | per-itimer (ITIMER_PROF/_VIRT) check | `PosixCpu::check_itimer` |
| `stop_process_timers()` | per-tick deactivation | `PosixCpu::stop_process_timers` |
| `task_cputimers_expired()` / `fastpath_timer_check()` | per-tick fastpath | `PosixCpu::{expired,fastpath}` |
| `run_posix_cpu_timers()` | per-tick entry | `PosixCpu::run_per_tick` |
| `handle_posix_cpu_timers()` | per-task expiry main | `PosixCpu::handle` |
| `posix_cpu_timers_work()` | per-task_work expiry trampoline | `PosixCpu::work_fn` |
| `__run_posix_cpu_timers()` | per-context dispatch (irq vs task_work) | `PosixCpu::dispatch` |
| `posix_cpu_timers_work_scheduled()` | per-task_work pending flag | `PosixCpu::work_scheduled` |
| `posix_cpu_timers_enable_work()` | per-RT reenable + recheck | `PosixCpu::enable_work` |
| `clear_posix_cputimers_work()` / `posix_cputimers_init_work()` | per-fork init | `PosixCpu::clear_work` / `init_work` |
| `set_process_cpu_timer()` | per-itimer / RLIMIT_CPU primary setter | `PosixCpu::set_process_timer` |
| `update_rlimit_cpu()` | per-setrlimit driver | `PosixCpu::update_rlimit_cpu` |
| `posix_cputimers_group_init()` | per-fork group init | `PosixCpu::group_init` |
| `thread_group_sample_cputime()` | per-getrusage / proc fastpath | `PosixCpu::group_sample` |
| `task_struct::posix_cputimers` | per-task expiry cache | `Task::posix_cputimers` |
| `task_struct::posix_cputimers_work` | per-task task_work expiry handle | `Task::posix_cputimers_work` |
| `signal_struct::cputimer` (thread_group_cputimer) | per-group atomic totals | `Signal::cputimer` |
| `signal_struct::it[CPUCLOCK_*]` (cpu_itimer) | per-group ITIMER_PROF/_VIRT | `Signal::itimers` |
| `signal_struct::posix_cputimers` | per-group expiry cache | `Signal::posix_cputimers` |
| `struct k_itimer::it.cpu` | per-timer per-cpu-clock state | `Itimer::cpu` |
| `struct cpu_timer` | per-timer node + handling-task pointer | `CpuTimer` |

### compatibility contract

REQ-1: Clock ID encoding (per-include/uapi/linux/time.h):
- A `clockid_t` for CPU clocks packs three things via the CPUCLOCK_* macros:
  - CPUCLOCK_WHICH(clk) ∈ {CPUCLOCK_PROF=0, CPUCLOCK_VIRT=1, CPUCLOCK_SCHED=2}.
  - CPUCLOCK_PERTHREAD(clk): bit set for per-thread, cleared for per-thread-group.
  - CPUCLOCK_PID(clk): the pid (process or thread) the clock targets; 0 means "current" / "current's tgid".
- CPUCLOCK_MAX is the upper sentinel for CPUCLOCK_WHICH.
- The fixed names CLOCK_PROCESS_CPUTIME_ID / CLOCK_THREAD_CPUTIME_ID are short forms with pid 0 and CPUCLOCK_SCHED.

REQ-2: posix_cputimers (per-task and per-signal):
- bases[CPUCLOCK_MAX]: per-clock posix_cputimer_base:
  - tqhead: timerqueue_head of armed cpu_timer nodes (sorted by expires).
  - nextevt: u64 earliest expiry across the queue + relevant itimers / rlimit (U64_MAX = inactive).
- timers_active: bool, true while the group's cputime aggregation is running.
- expiry_active: bool, set true by check_process_timers while running to suppress concurrent passes from sibling threads.

REQ-3: thread_group_cputimer (signal_struct::cputimer):
- cputime_atomic: task_cputime_atomic {atomic64_t utime, stime, sum_exec_runtime}.
- These atomics are the running totals across all threads of the group. They are updated by accounting paths (account_user_time / account_system_time / scheduler tick) when timers_active is true.

REQ-4: k_clock vtables:
- clock_posix_cpu (used when a pid is embedded in the clockid):
  - clock_getres = posix_cpu_clock_getres.
  - clock_set = posix_cpu_clock_set.
  - clock_get_timespec = posix_cpu_clock_get.
  - timer_create = posix_cpu_timer_create.
  - nsleep = posix_cpu_nsleep.
  - timer_set = posix_cpu_timer_set.
  - timer_del = posix_cpu_timer_del.
  - timer_get = posix_cpu_timer_get.
  - timer_rearm = posix_cpu_timer_rearm.
  - timer_wait_running = posix_cpu_timer_wait_running.
- clock_process (CLOCK_PROCESS_CPUTIME_ID):
  - clock_getres / clock_get_timespec / timer_create / nsleep only; uses PROCESS_CLOCK = make_process_cpuclock(0, CPUCLOCK_SCHED).
- clock_thread (CLOCK_THREAD_CPUTIME_ID):
  - clock_getres / clock_get_timespec / timer_create only; uses THREAD_CLOCK = make_thread_cpuclock(0, CPUCLOCK_SCHED).

REQ-5: posix_cpu_clock_getres(clk, tp):
- validate_clock_permissions(clk) first.
- tp.tv_sec = 0; tp.tv_nsec = ((NSEC_PER_SEC + HZ - 1) / HZ).
- If CPUCLOCK_WHICH(clk) == CPUCLOCK_SCHED: tp.tv_nsec = 1 (sched_clock is much finer than 1/HZ but exact resolution is unknown).
- Returns 0 on success, -EINVAL on bad clock.

REQ-6: posix_cpu_clock_set(clk, tp):
- validate_clock_permissions first.
- Always returns -EPERM (CPU clocks are read-only).

REQ-7: posix_cpu_clock_get(clk, tp):
- rcu_read_lock.
- tsk = pid_task(pid_for_clock(clk, gettime=true), clock_pid_type(clk)).
- if !tsk: return -EINVAL.
- If CPUCLOCK_PERTHREAD: t = cpu_clock_sample(CPUCLOCK_WHICH(clk), tsk).
- Else: t = cpu_clock_sample_group(CPUCLOCK_WHICH(clk), tsk, start=false).
- rcu_read_unlock; *tp = ns_to_timespec64(t); return 0.

REQ-8: pid_for_clock(clk, gettime):
- thread = !!CPUCLOCK_PERTHREAD(clk); upid = CPUCLOCK_PID(clk).
- If CPUCLOCK_WHICH(clk) ≥ CPUCLOCK_MAX: return NULL.
- If upid == 0:
  - if thread: return task_pid(current).
  - else: return task_tgid(current).
- pid = find_vpid(upid); if !pid: return NULL.
- If thread:
  - tsk = pid_task(pid, PIDTYPE_PID).
  - return (tsk && same_thread_group(tsk, current)) ? pid : NULL.
- Else (process clock):
  - if gettime && pid == task_pid(current): return task_tgid(current).
  - return pid_has_task(pid, PIDTYPE_TGID) ? pid : NULL.

REQ-9: cpu_clock_sample(clkid, p):
- If clkid == CPUCLOCK_SCHED: return task_sched_runtime(p).
- task_cputime(p, &utime, &stime).
- case CPUCLOCK_PROF: return utime + stime.
- case CPUCLOCK_VIRT: return utime.
- default: WARN_ON_ONCE; return 0.

REQ-10: cpu_clock_sample_group(clkid, p, start):
- cputimer = &p.signal.cputimer; pct = &p.signal.posix_cputimers.
- If !READ_ONCE(pct.timers_active):
  - if start: thread_group_start_cputime(p, samples).
  - else: __thread_group_cputime(p, samples).
- Else: proc_sample_cputime_atomic(&cputimer.cputime_atomic, samples).
- Return samples[clkid].

REQ-11: thread_group_start_cputime(tsk, samples):
- lockdep_assert_task_sighand_held(tsk).
- If !READ_ONCE(pct.timers_active):
  - thread_group_cputime(tsk, &sum).
  - update_gt_cputime(&cputimer.cputime_atomic, &sum) — cmpxchg utime/stime/sum_exec_runtime to max(current, sum).
  - WRITE_ONCE(pct.timers_active, true).
- proc_sample_cputime_atomic(&cputimer.cputime_atomic, samples).

REQ-12: posix_cpu_timer_create(new_timer):
- rcu_read_lock.
- pid = pid_for_clock(new_timer.it_clock, false).
- if !pid: rcu_read_unlock; return -EINVAL.
- If CONFIG_POSIX_CPU_TIMERS_TASK_WORK: lockdep_set_class(&new_timer.it_lock, &posix_cpu_timers_key) (separate lock-class to avoid false interrupt-context warnings).
- new_timer.kclock = &clock_posix_cpu.
- timerqueue_init(&new_timer.it.cpu.node).
- new_timer.it.cpu.pid = get_pid(pid).
- rcu_read_unlock; return 0.

REQ-13: posix_cpu_timer_set(timer, flags, new, old):
- rcu_read_lock. p = cpu_timer_task_rcu(timer). if !p: rcu_read_unlock; return -ESRCH.
- new_expires = ktime_to_ns(timespec64_to_ktime(new.it_value)).
- sighand = lock_task_sighand(p, &flags). if !sighand: rcu_read_unlock; return -ESRCH.
- old_expires = cpu_timer_getexpires(&timer.it.cpu).
- If timer.it.cpu.firing: clear firing; ret = TIMER_RETRY.
- Else: cpu_timer_dequeue(&timer.it.cpu); timer.it_status = POSIX_TIMER_DISARMED.
- /* Sample */
- if CPUCLOCK_PERTHREAD: now = cpu_clock_sample(clkid, p).
- else: now = cpu_clock_sample_group(clkid, p, !sigev_none).
- /* Old value */
- if old != NULL: old.it_value = {0,0}; if old_expires: __posix_cpu_timer_get(timer, old, now).
- if ret == TIMER_RETRY: unlock_task_sighand; goto out.
- /* Relative → absolute */
- if new_expires && !(flags & TIMER_ABSTIME): new_expires += now.
- cpu_timer_setexpires(&timer.it.cpu, new_expires).
- /* Arm */
- if !sigev_none:
  - if new_expires && now < new_expires: arm_timer(timer, p).
  - else: trigger_base_recalc_expires(timer, p).
- unlock_task_sighand.
- posix_timer_set_common(timer, new) — store it_interval and signal config.
- /* Already-past: fire immediately */
- if !sigev_none && new_expires && now >= new_expires: cpu_timer_fire(timer).
- out: rcu_read_unlock; return ret.

REQ-14: __posix_cpu_timer_get(timer, itp, now):
- sigev_none = (timer.it_sigev_notify == SIGEV_NONE); iv = timer.it_interval.
- if iv && timer.it_status != POSIX_TIMER_ARMED: expires = bump_cpu_timer(timer, now) — advance through missed intervals, count overruns.
- else: expires = cpu_timer_getexpires(&timer.it.cpu).
- if now < expires: itp.it_value = ns_to_timespec64(expires - now).
- else:
  - if sigev_none: itp.it_value = 0 (single-shot SIGEV_NONE returns 0 when expired).
  - else: itp.it_value.tv_nsec = 1 (signal not yet delivered — must show ≥ 1ns remaining).

REQ-15: posix_cpu_timer_get(timer, itp):
- rcu_read_lock.
- p = cpu_timer_task_rcu(timer).
- if p && cpu_timer_getexpires(&timer.it.cpu):
  - itp.it_interval = ktime_to_timespec64(timer.it_interval).
  - if CPUCLOCK_PERTHREAD: now = cpu_clock_sample(clkid, p). else now = cpu_clock_sample_group(clkid, p, false).
  - __posix_cpu_timer_get(timer, itp, now).
- rcu_read_unlock.

REQ-16: posix_cpu_timer_del(timer):
- rcu_read_lock. p = cpu_timer_task_rcu(timer). if !p: goto out.
- sighand = lock_task_sighand(p, &flags).
- if !sighand: WARN_ON_ONCE(timer queued).
- else:
  - if timer.it.cpu.firing: clear firing; ret = TIMER_RETRY.
  - else: disarm_timer(timer, p).
  - unlock_task_sighand.
- out: rcu_read_unlock.
- if !ret: put_pid(timer.it.cpu.pid); timer.it_status = POSIX_TIMER_DISARMED.
- return ret.

REQ-17: arm_timer(timer, p):
- base = timer_base(timer, p).
- timer.it_status = POSIX_TIMER_ARMED.
- if !cpu_timer_enqueue(&base.tqhead, &timer.it.cpu): return (was not earliest).
- /* New earliest */
- if newexp < base.nextevt: base.nextevt = newexp.
- if CPUCLOCK_PERTHREAD: tick_dep_set_task(p, TICK_DEP_BIT_POSIX_TIMER).
- else: tick_dep_set_signal(p, TICK_DEP_BIT_POSIX_TIMER).

REQ-18: disarm_timer(timer, p):
- if !cpu_timer_dequeue(&timer.it.cpu): return (not queued).
- base = timer_base(timer, p).
- if cpu_timer_getexpires(&timer.it.cpu) == base.nextevt: trigger_base_recalc_expires(timer, p) — sets base.nextevt = 0 to force recompute on next tick.

REQ-19: cpu_timer_fire(timer):
- timer.it_status = POSIX_TIMER_DISARMED.
- if unlikely(timer.it.cpu.nanosleep):
  - wake_up_process(timer.it_process).
  - cpu_timer_setexpires(&timer.it.cpu, 0).
- else:
  - posix_timer_queue_signal(timer) — enqueues sigev_signo via send_signal_locked.
  - if !timer.it_interval: cpu_timer_setexpires(&timer.it.cpu, 0).

REQ-20: posix_cpu_timer_rearm(timer):
- rcu_read_lock. p = cpu_timer_task_rcu(timer). if !p: goto out.
- sighand = lock_task_sighand(p, &flags). if !sighand: goto out.
- if CPUCLOCK_PERTHREAD: now = cpu_clock_sample(clkid, p). else now = cpu_clock_sample_group(clkid, p, true).
- bump_cpu_timer(timer, now).
- arm_timer(timer, p).
- unlock_task_sighand.
- out: rcu_read_unlock.

REQ-21: bump_cpu_timer(timer, now):
- if !it_interval: return expires.
- if now < expires: return expires.
- incr = it_interval; delta = now + incr - expires.
- Iterate i = 0..log2(delta/incr): incr <<= 1.
- For i downward: if delta ≥ incr: expires += incr; it_overrun += 1 << i; delta -= incr.
- Return updated expires.

REQ-22: collect_timerqueue(head, firing, now) → next-evt:
- For up to MAX_COLLECTED (20) timers whose expires ≤ now:
  - ctmr.firing = true.
  - rcu_assign_pointer(ctmr.handling, current) — so wait_running sees the handler.
  - cpu_timer_dequeue.
  - list_add_tail(&ctmr.elist, firing).
- Return earliest still-armed expires (or U64_MAX if none).

REQ-23: collect_posix_cputimers(pct, samples, firing):
- For each clkid (PROF / VIRT / SCHED):
  - pct.bases[clkid].nextevt = collect_timerqueue(&pct.bases[clkid].tqhead, firing, samples[clkid]).

REQ-24: check_thread_timers(tsk, firing):
- pct = &tsk.posix_cputimers.
- if dl_task(tsk): check_dl_overrun(tsk).
- if expiry_cache_is_inactive(pct): return.
- task_sample_cputime(tsk, samples).
- collect_posix_cputimers(pct, samples, firing).
- /* RLIMIT_RTTIME */
- soft = task_rlimit(tsk, RLIMIT_RTTIME). if soft != INFINITY:
  - rttime = tsk.rt.timeout * (USEC_PER_SEC/HZ).
  - hard = task_rlimit_max(tsk, RLIMIT_RTTIME).
  - if hard != INFINITY && check_rlimit(rttime, hard, SIGKILL, rt=true, hard=true): return.
  - if check_rlimit(rttime, soft, SIGXCPU, rt=true, hard=false): soft += USEC_PER_SEC; signal.rlim[RLIMIT_RTTIME].rlim_cur = soft.
- if expiry_cache_is_inactive(pct): tick_dep_clear_task(tsk, TICK_DEP_BIT_POSIX_TIMER).

REQ-25: check_process_timers(tsk, firing):
- sig = tsk.signal; pct = &sig.posix_cputimers.
- if !READ_ONCE(pct.timers_active) || pct.expiry_active: return.
- pct.expiry_active = true.
- proc_sample_cputime_atomic(&sig.cputimer.cputime_atomic, samples).
- collect_posix_cputimers(pct, samples, firing).
- check_cpu_itimer(tsk, &sig.it[CPUCLOCK_PROF], &pct.bases[PROF].nextevt, samples[PROF], SIGPROF).
- check_cpu_itimer(tsk, &sig.it[CPUCLOCK_VIRT], &pct.bases[VIRT].nextevt, samples[VIRT], SIGVTALRM).
- /* RLIMIT_CPU */
- soft = task_rlimit(tsk, RLIMIT_CPU). if soft != INFINITY:
  - softns = soft * NSEC_PER_SEC; hardns = hard * NSEC_PER_SEC.
  - if hard != INFINITY && check_rlimit(ptime, hardns, SIGKILL, rt=false, hard=true): return.
  - if check_rlimit(ptime, softns, SIGXCPU, rt=false, hard=false): sig.rlim[RLIMIT_CPU].rlim_cur = soft + 1; softns += NSEC_PER_SEC.
  - if softns < pct.bases[PROF].nextevt: pct.bases[PROF].nextevt = softns.
- if expiry_cache_is_inactive(pct): stop_process_timers(sig).
- pct.expiry_active = false.

REQ-26: check_cpu_itimer(tsk, it, expires, cur_time, signo):
- if !it.expires: return.
- if cur_time ≥ it.expires:
  - it.expires = (it.incr) ? (it.expires + it.incr) : 0.
  - trace_itimer_expire(ITIMER_PROF/_VIRT, task_tgid(tsk), cur_time).
  - send_signal_locked(signo, SEND_SIG_PRIV, tsk, PIDTYPE_TGID).
- if it.expires && it.expires < *expires: *expires = it.expires.

REQ-27: check_rlimit(time, limit, signo, rt, hard):
- if time < limit: return false.
- if print_fatal_signals: pr_info "%s Watchdog Timeout (%s): %s[%d]".
- send_signal_locked(signo, SEND_SIG_PRIV, current, PIDTYPE_TGID).
- return true.

REQ-28: stop_process_timers(sig):
- pct = &sig.posix_cputimers.
- WRITE_ONCE(pct.timers_active, false).
- tick_dep_clear_signal(sig, TICK_DEP_BIT_POSIX_TIMER).

REQ-29: fastpath_timer_check(tsk):
- /* Per-thread */
- if !expiry_cache_is_inactive(&tsk.posix_cputimers):
  - task_sample_cputime(tsk, samples).
  - if task_cputimers_expired(samples, &tsk.posix_cputimers): return true.
- /* Per-group */
- sig = tsk.signal; pct = &sig.posix_cputimers.
- if READ_ONCE(pct.timers_active) && !READ_ONCE(pct.expiry_active):
  - proc_sample_cputime_atomic(&sig.cputimer.cputime_atomic, samples).
  - if task_cputimers_expired(samples, pct): return true.
- if dl_task(tsk) && tsk.dl.dl_overrun: return true.
- return false.

REQ-30: run_posix_cpu_timers():
- lockdep_assert_irqs_disabled.
- if tsk.exit_state: return.
- if posix_cpu_timers_work_scheduled(tsk): return.
- if !fastpath_timer_check(tsk): return.
- __run_posix_cpu_timers(tsk).

REQ-31: __run_posix_cpu_timers dispatch:
- CONFIG_POSIX_CPU_TIMERS_TASK_WORK:
  - if WARN_ON_ONCE(tsk.posix_cputimers_work.scheduled): return.
  - tsk.posix_cputimers_work.scheduled = true.
  - task_work_add(tsk, &tsk.posix_cputimers_work.work, TWA_RESUME).
- Else:
  - lockdep_posixtimer_enter; handle_posix_cpu_timers(tsk); lockdep_posixtimer_exit.

REQ-32: posix_cpu_timers_work (task_work callback):
- cw = container_of(work, posix_cputimers_work, work).
- mutex_lock(&cw.mutex); handle_posix_cpu_timers(current); mutex_unlock.

REQ-33: posix_cpu_timer_wait_running(timr):
- tsk = rcu_dereference(timr.it.cpu.handling).
- if !tsk: return.
- get_task_struct(tsk); rcu_read_unlock.
- mutex_lock(&tsk.posix_cputimers_work.mutex); mutex_unlock(&tsk.posix_cputimers_work.mutex).
- put_task_struct(tsk); rcu_read_lock (to balance caller).

REQ-34: posix_cpu_timer_wait_running_nsleep(timr):
- rcu_read_lock. spin_unlock_irq(&timr.it_lock). posix_cpu_timer_wait_running(timr). rcu_read_unlock.
- spin_lock_irq(&timr.it_lock).

REQ-35: clear_posix_cputimers_work(p) (per-fork):
- memset(&p.posix_cputimers_work.work, 0, sizeof work).
- init_task_work(&p.posix_cputimers_work.work, posix_cpu_timers_work).
- mutex_init(&p.posix_cputimers_work.mutex).
- p.posix_cputimers_work.scheduled = false.

REQ-36: posix_cputimers_init_work(): clear_posix_cputimers_work(current) (init task).

REQ-37: posix_cpu_timers_enable_work(tsk, start):
- if !PREEMPT_RT: tsk.posix_cputimers_work.scheduled = false; return true.
- /* RT */
- local_irq_disable.
- if start != jiffies && fastpath_timer_check(tsk): ret = false.
- else: tsk.posix_cputimers_work.scheduled = false.
- local_irq_enable.
- return ret.

REQ-38: handle_posix_cpu_timers(tsk) — main expiry path:
- if !lock_task_sighand(tsk, &flags): return.
- do:
  - start = READ_ONCE(jiffies); barrier.
  - check_thread_timers(tsk, &firing).
  - check_process_timers(tsk, &firing).
- while (!posix_cpu_timers_enable_work(tsk, start)).
- unlock_task_sighand.
- list_for_each_entry_safe(timer, next, &firing, it.cpu.elist):
  - spin_lock(&timer.it_lock).
  - list_del_init(&timer.it.cpu.elist).
  - cpu_firing = timer.it.cpu.firing; timer.it.cpu.firing = false.
  - if cpu_firing: cpu_timer_fire(timer).
  - rcu_assign_pointer(timer.it.cpu.handling, NULL).
  - spin_unlock(&timer.it_lock).

REQ-39: set_process_cpu_timer(tsk, clkid, newval, oldval):
- WARN_ON_ONCE(clkid >= CPUCLOCK_SCHED) — CPUCLOCK_SCHED never used as itimer/RLIMIT_CPU setter.
- nextevt = &tsk.signal.posix_cputimers.bases[clkid].nextevt.
- now = cpu_clock_sample_group(clkid, tsk, true).
- if oldval (setitimer): if *oldval ≤ now: *oldval = TICK_NSEC (just about to fire); else *oldval -= now (return remaining).
- if *newval: *newval += now (relative → absolute).
- if *newval < *nextevt: *nextevt = *newval.
- tick_dep_set_signal(tsk, TICK_DEP_BIT_POSIX_TIMER).

REQ-40: update_rlimit_cpu(task, rlim_new):
- nsecs = rlim_new * NSEC_PER_SEC.
- lock_task_sighand(task, &flags); on failure return -ESRCH.
- set_process_cpu_timer(task, CPUCLOCK_PROF, &nsecs, NULL).
- unlock_task_sighand; return 0.

REQ-41: posix_cputimers_group_init(pct, cpu_limit):
- posix_cputimers_init(pct).
- if cpu_limit != RLIM_INFINITY: pct.bases[CPUCLOCK_PROF].nextevt = cpu_limit * NSEC_PER_SEC; pct.timers_active = true.

REQ-42: thread_group_sample_cputime(tsk, samples):
- (called from getrusage / proc fast paths)
- proc_sample_cputime_atomic(&tsk.signal.cputimer.cputime_atomic, samples).

REQ-43: posix_cpu_timers_exit(tsk) / _exit_group(tsk):
- cleanup_timers(&tsk.posix_cputimers).
- cleanup_timers(&tsk.signal.posix_cputimers) for _exit_group.
- cleanup_timerqueue walks each base.tqhead and clears head pointers — the k_itimer objects remain but cannot be re-armed.

REQ-44: do_cpu_nanosleep(clkid, flags, rqtp):
- Build temporary on-stack k_itimer timer with nanosleep = true.
- posix_cpu_timer_create + posix_cpu_timer_set with rqtp as it_value.
- Loop:
  - if !cpu_timer_getexpires: timer_del; return 0 (already fired).
  - __set_current_state(TASK_INTERRUPTIBLE); spin_unlock_irq; schedule(); spin_lock_irq.
  - if signal_pending: break.
- On signal: posix_cpu_timer_set(zero_it, &it). If TIMER_RETRY: loop wait_running_nsleep + del.
- If timer expired: return 0.
- Else: error = -ERESTART_RESTARTBLOCK; restart.nanosleep.expires = ns_to_ktime(expires); copy remaining.

REQ-45: posix_cpu_nsleep(clk, flags, rqtp):
- Reject CPUCLOCK_PERTHREAD with pid == 0 or pid == self (cannot sleep on own CPU clock — would deadlock).
- error = do_cpu_nanosleep(...).
- On -ERESTART_RESTARTBLOCK:
  - if TIMER_ABSTIME: return -ERESTARTNOHAND (no restart for absolute).
  - else set restart.nanosleep.clockid = clk; set_restart_fn(restart, posix_cpu_nsleep_restart).

REQ-46: posix_cpu_nsleep_restart(rb):
- t = ktime_to_timespec64(rb.nanosleep.expires).
- return do_cpu_nanosleep(rb.nanosleep.clockid, TIMER_ABSTIME, &t).

REQ-47: clock_process / clock_thread fixed-id forwarders:
- process_cpu_clock_getres / _get / process_cpu_timer_create / process_cpu_nsleep forward to posix_cpu_* with PROCESS_CLOCK = make_process_cpuclock(0, CPUCLOCK_SCHED).
- thread_cpu_clock_getres / _get / thread_cpu_timer_create forward with THREAD_CLOCK = make_thread_cpuclock(0, CPUCLOCK_SCHED).
- clock_thread has no nsleep (no clock_nanosleep(CLOCK_THREAD_CPUTIME_ID)).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cpu_timer_pid_refcount_balanced` | INVARIANT | per-timer_create / _del: get_pid in create matched by put_pid in del. |
| `sighand_lock_held_for_timer_queue` | INVARIANT | per-timer_set / _del / arm_timer / check_process_timers: ops on signal.posix_cputimers.bases or signal.it[] happen under lock_task_sighand. |
| `it_status_transitions_legal` | INVARIANT | per-timer: DISARMED → ARMED only via arm_timer; ARMED → DISARMED only via dequeue / fire. |
| `firing_flag_balanced` | INVARIANT | per-collect_timerqueue / handle: every ctmr.firing = true is cleared either by handle_posix_cpu_timers' firing-loop or by timer_set/del TIMER_RETRY. |
| `handling_pointer_lifetime` | INVARIANT | per-collect / handle: rcu_assign_pointer(handling, current) before list_add; rcu_assign_pointer(handling, NULL) after fire. |
| `expiry_active_bool_locked` | INVARIANT | per-check_process_timers: pct.expiry_active toggled under sighand. |
| `task_work_scheduled_once_per_tick` | INVARIANT | per-dispatch: posix_cputimers_work.scheduled WARN_ON_ONCE if already set. |
| `update_gt_cputime_monotone` | INVARIANT | per-update_gt_cputime: atomic64 utime/stime/sum_exec_runtime never decrease (cmpxchg loop). |
| `clk_pid_perthread_same_group` | INVARIANT | per-pid_for_clock: thread clock with non-zero pid requires same_thread_group(tsk, current). |
| `nsleep_perthread_self_rejected` | INVARIANT | per-posix_cpu_nsleep: CPUCLOCK_PERTHREAD with pid==0 or pid==self returns -EINVAL. |
| `task_work_lock_class_isolated` | INVARIANT | per-timer_create with TASK_WORK: lockdep_set_class on it_lock to avoid false IRQ-context positives. |

### Layer 2: TLA+

`kernel/time/posix-cpu-timers.tla`:
- Per-timer state machine over {DISARMED, ARMED, FIRING}.
- Per-task and per-group expiry-cache state machine over {INACTIVE, ACTIVE, EXPIRING}.
- Properties:
  - `safety_arm_only_after_dequeue` — per-timer_set: cpu_timer_dequeue before re-arm.
  - `safety_fire_sees_handling_task` — per-collect_timerqueue: rcu_assign_pointer(handling, current) before list_add into firing.
  - `safety_send_signal_locked_under_sighand` — per-check_cpu_itimer / check_rlimit: send_signal_locked happens with sighand held.
  - `safety_rlimit_cpu_hard_implies_SIGKILL` — per-RLIMIT_CPU: ptime ≥ hardns ⟹ SIGKILL.
  - `safety_rlimit_rttime_hard_implies_SIGKILL` — per-RLIMIT_RTTIME.
  - `safety_dl_overrun_implies_SIGXCPU` — per-check_dl_overrun.
  - `safety_setitimer_signo_correct` — per-CPUCLOCK_PROF/_VIRT: SIGPROF / SIGVTALRM respectively.
  - `safety_clock_set_eperm` — per-posix_cpu_clock_set always returns -EPERM.
  - `safety_thread_clock_no_nsleep` — per-clock_thread: no .nsleep operation.
  - `liveness_armed_timer_eventually_fires` — per-tick: enough time passes ⟹ collect_timerqueue picks it up.
  - `liveness_task_work_runs_before_return_to_user` — per-dispatch: TWA_RESUME guarantees execution.
  - `liveness_rt_enable_work_retries_on_jiffies_advance` — per-PREEMPT_RT enable_work.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `timer_set` post: if !sigev_none ∧ new_expires > now ⟹ timer on base.tqhead, base.nextevt ≤ new_expires | `PosixCpu::timer_set` |
| `timer_set` post: if new_expires ≤ now ⟹ cpu_timer_fire invoked exactly once | `PosixCpu::timer_set` |
| `timer_del` post: ctmr.head == NULL; if !TIMER_RETRY ⟹ put_pid called once, status = DISARMED | `PosixCpu::timer_del` |
| `timer_rearm` post: bump_cpu_timer + arm_timer; new expires > now | `PosixCpu::timer_rearm` |
| `arm_timer` post: tick_dep_set_task / _signal called | `PosixCpu::arm_timer` |
| `disarm_timer` post: if was earliest, base.nextevt reset to 0 (recalc-pending) | `PosixCpu::disarm_timer` |
| `check_thread_timers` post: expired thread timers moved to firing list; RLIMIT_RTTIME hard ⟹ SIGKILL | `PosixCpu::check_thread` |
| `check_process_timers` post: expired itimers fired; RLIMIT_CPU hard ⟹ SIGKILL; pct.expiry_active = false on exit | `PosixCpu::check_process` |
| `handle` post: firing list empty; every timer's handling cleared | `PosixCpu::handle` |
| `set_process_cpu_timer` post: nextevt = min(nextevt, newval_absolute); tick_dep_set_signal | `PosixCpu::set_process_timer` |
| `cleanup_timers` post: all base.tqhead drained; head pointers cleared | `PosixCpu::cleanup` |
| `do_nsleep` post: returns 0 on natural expiry, -ERESTART_RESTARTBLOCK on signal with relative request, -EINTR/-ERESTART path on absolute | `PosixCpu::do_nsleep` |

### Layer 4: Verus/Creusot functional

`Per-timer pipeline: timer_create → timer_set → tick → fastpath → handle (check_thread + check_process → collect_timerqueue → firing list) → cpu_timer_fire → posix_timer_queue_signal → send_signal_locked` semantic equivalence: per-POSIX.1-2024 §27 (Timers) and Documentation/timers/.

`Per-clock_nanosleep pipeline: posix_cpu_nsleep → do_cpu_nanosleep (build temp timer with nanosleep=true) → schedule loop → cpu_timer_fire ⟹ wake_up_process → return 0` semantic equivalence: per-POSIX.1-2024 clock_nanosleep description.

`Per-RLIMIT_CPU pipeline: setrlimit(RLIMIT_CPU) → update_rlimit_cpu → set_process_cpu_timer(CPUCLOCK_PROF, &nsecs, NULL) → nextevt update → tick → check_process_timers → check_rlimit (soft=SIGXCPU/hard=SIGKILL)` semantic equivalence: per-getrlimit(2) man page.

`Per-itimer pipeline: setitimer(ITIMER_PROF) → set_process_cpu_timer(CPUCLOCK_PROF, &incr, &old) + sig.it[PROF].{expires,incr} = ... → tick → check_cpu_itimer → SIGPROF` semantic equivalence: per-setitimer(2) man page; ditto ITIMER_VIRTUAL → SIGVTALRM.

### hardening

(Inherits row-1 features from `kernel/time/00-overview.md` § Hardening.)

POSIX CPU timers reinforcement:

- **Per-pid get_pid / put_pid strict in timer_create / _del** — defense against per-timer pid UAF after target task reap.
- **Per-rcu_read_lock around cpu_timer_task_rcu** — defense against per-task UAF in concurrent timer-set/get/del.
- **Per-sighand lock_task_sighand for every queue / itimer / rlim mutation** — defense against per-thread-group concurrent armer.
- **Per-CPUCLOCK_SCHED rejected in set_process_cpu_timer (WARN_ON_ONCE)** — defense against per-setitimer/RLIMIT_CPU misuse on sched clock.
- **Per-MAX_COLLECTED = 20 cap on collect_timerqueue** — defense against per-tick unbounded firing-list growth (long-tail starvation).
- **Per-TASK_WORK deferral on PREEMPT_RT** — defense against per-RT lock-inversion in signal delivery from IRQ.
- **Per-jiffies-recheck in posix_cpu_timers_enable_work (RT)** — defense against per-RT missed-tick race after sighand drop.
- **Per-self-perthread-nsleep rejected (-EINVAL)** — defense against per-process deadlock (sleeping on own CPU clock).
- **Per-TIMER_ABSTIME no restart (-ERESTARTNOHAND)** — defense against per-signal-restart drift on absolute timers.
- **Per-handling pointer rcu_assign_pointer paired NULL-after-fire** — defense against per-cancel-vs-fire UAF detection in wait_running.
- **Per-firing flag preserved across timer_set TIMER_RETRY** — defense against per-cancel-in-flight signal loss.
- **Per-tick_dep_set / _clear paired** — defense against per-stuck-tick on idle CPUs after timer disarm (nohz_full).
- **Per-update_gt_cputime cmpxchg-monotone** — defense against per-thread-group accounting regression on concurrent thread reads.

