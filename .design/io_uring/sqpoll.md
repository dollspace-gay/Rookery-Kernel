# Tier-3: io_uring/sqpoll.c — io_uring SQPOLL kernel thread

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/sqpoll.c (~569 lines)
  - io_uring/sqpoll.h
  - include/uapi/linux/io_uring.h (IORING_SETUP_SQPOLL, IORING_SETUP_SQ_AFF, IORING_SETUP_ATTACH_WQ, IORING_SQ_NEED_WAKEUP)
-->

## Summary

When userspace passes `IORING_SETUP_SQPOLL` to `io_uring_setup(2)`, the kernel spawns a dedicated kernel thread (`iou-sqp-<pid>`) that polls the SQ ring on the application's behalf, so syscall-free submission becomes possible. Per-thread state: `struct io_sq_data` (lock, wait, ctx_list, sq_thread_idle, sq_cpu, refs, park_pending, state). One sqd may serve many ctxs (`IORING_SETUP_ATTACH_WQ` + matching tgid); per-iter fair share via `IORING_SQPOLL_CAP_ENTRIES_VALUE = 8`. Per-idle timeout: `sq_thread_idle` jiffies after which the thread parks itself with `IORING_SQ_NEED_WAKEUP` set so userspace wakes it on next submit via `io_uring_enter(IORING_ENTER_SQ_WAKEUP)`. Per-NUMA placement: `IORING_SETUP_SQ_AFF` + `p.sq_thread_cpu` pins thread; `io_uring_register(IORING_REGISTER_IOWQ_AFF)` sets affinity at runtime. Park/unpark protocol via `IO_SQ_THREAD_SHOULD_PARK` and `park_pending` counter lets registration / cancellation acquire `sqd->lock` against the sq thread. Critical for: low-latency submission, multi-ring sharing, kernel-thread audit/sec_uring policy, NUMA fanout.

This Tier-3 covers `io_uring/sqpoll.c` (~569 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_sq_data` | per-SQPOLL-thread shared state | `IoSqData` |
| `struct io_sq_time` | per-iter cputime accounting | `IoSqTime` |
| `IO_SQ_THREAD_SHOULD_STOP` / `_SHOULD_PARK` | per-state bits | `SqThreadStateBit` |
| `IORING_SQPOLL_CAP_ENTRIES_VALUE` | per-fairness cap (8) | constant |
| `IORING_TW_CAP_ENTRIES_VALUE` | per-task-work cap (32) | constant |
| `io_sq_thread_unpark()` | per-release-of-lock | `Sqpoll::thread_unpark` |
| `io_sq_thread_park()` | per-acquire-with-thread-yield | `Sqpoll::thread_park` |
| `io_sq_thread_stop()` | per-termination | `Sqpoll::thread_stop` |
| `io_put_sq_data()` | per-refcount drop | `Sqpoll::put` |
| `io_sqd_update_thread_idle()` | per-ctx-add idle-recompute | `Sqpoll::update_idle` |
| `io_sq_thread_finish()` | per-ctx detach | `Sqpoll::finish` |
| `io_attach_sq_data()` | per-IORING_SETUP_ATTACH_WQ lookup | `Sqpoll::attach` |
| `io_get_sq_data()` | per-create-or-attach | `Sqpoll::get_or_create` |
| `io_sqd_events_pending()` | per-event-bit read | `Sqpoll::events_pending` |
| `io_sq_cpu_usec()` | per-task cputime | `Sqpoll::cpu_usec` |
| `io_sq_update_worktime()` / `_start_worktime` | per-iter accounting | `Sqpoll::worktime_*` |
| `__io_sq_thread()` | per-ctx single-pass | `Sqpoll::single_pass` |
| `io_sqd_handle_event()` | per-park-or-signal | `Sqpoll::handle_event` |
| `io_sq_tw()` | per-task-work batched run | `Sqpoll::tw_run` |
| `io_sq_tw_pending()` | per-task-work readiness | `Sqpoll::tw_pending` |
| `io_sq_thread()` | per-thread main loop | `Sqpoll::main_loop` |
| `io_sqpoll_wait_sq()` | per-userspace SQ-full wait | `Sqpoll::wait_sq` |
| `io_sq_offload_create()` | per-setup creation | `Sqpoll::offload_create` |
| `io_sqpoll_wq_cpu_affinity()` | per-IORING_REGISTER_IOWQ_AFF | `Sqpoll::wq_cpu_affinity` |
| `sqpoll_task_locked()` | per-sqd locked-thread accessor | `Sqpoll::task_locked` |

## Compatibility contract

REQ-1: struct io_sq_data (per-thread shared):
- lock: mutex serializing ctx_list / thread / sq_cpu mutation.
- wait: waitqueue the thread sleeps on when idle.
- ctx_list: list of io_ring_ctx serviced by this thread.
- sq_thread_idle: per-iter idle timeout (jiffies); max across ctx_list.
- sq_cpu: bound CPU id or -1 (no SQ_AFF).
- refs: refcount across ctxs sharing the sqd.
- park_pending: count of pending park requests.
- state: bitmap IO_SQ_THREAD_SHOULD_STOP | _SHOULD_PARK.
- thread: RCU-published task_struct of the kernel thread.
- task_pid / task_tgid: owner tgid + thread pid (after rename).
- sq_creds: cred snapshot of submitter (used during submit override).
- exited: completion signalled on thread exit.
- work_time: cumulative cputime stat.

REQ-2: State bits:
- IO_SQ_THREAD_SHOULD_STOP (0): terminate request.
- IO_SQ_THREAD_SHOULD_PARK (1): yield-and-wait request.

REQ-3: io_sq_thread_park(sqd):
- __acquires(&sqd.lock).
- atomic_inc(&sqd.park_pending).
- set_bit(IO_SQ_THREAD_SHOULD_PARK, &sqd.state).
- mutex_lock(&sqd.lock).
- tsk = sqpoll_task_locked(sqd).
- if tsk: WARN_ON_ONCE(tsk == current); wake_up_process(tsk).

REQ-4: io_sq_thread_unpark(sqd):
- __releases(&sqd.lock).
- WARN_ON_ONCE(sqpoll_task_locked(sqd) == current).
- clear_bit(IO_SQ_THREAD_SHOULD_PARK, &sqd.state).
- if atomic_dec_return(&sqd.park_pending): set_bit(IO_SQ_THREAD_SHOULD_PARK, ...).  /* race-with-other-parker */
- mutex_unlock(&sqd.lock).
- wake_up(&sqd.wait).

REQ-5: io_sq_thread_stop(sqd):
- WARN_ON_ONCE(test_bit(IO_SQ_THREAD_SHOULD_STOP, ...)).
- set_bit(IO_SQ_THREAD_SHOULD_STOP, &sqd.state).
- mutex_lock(&sqd.lock).
- tsk = sqpoll_task_locked(sqd). if tsk: WARN_ON_ONCE(tsk == current); wake_up_process(tsk).
- mutex_unlock(&sqd.lock).
- wait_for_completion(&sqd.exited).

REQ-6: io_put_sq_data(sqd):
- if refcount_dec_and_test(&sqd.refs):
  - WARN_ON_ONCE(atomic_read(&sqd.park_pending)).
  - io_sq_thread_stop(sqd).
  - kfree(sqd).

REQ-7: io_sqd_update_thread_idle(sqd):
- /* Recompute per-ctx max */
- max = 0.
- for ctx in sqd.ctx_list: max = max(max, ctx.sq_thread_idle).
- sqd.sq_thread_idle = max.

REQ-8: io_sq_thread_finish(ctx):
- sqd = ctx.sq_data. if !sqd: return.
- io_sq_thread_park(sqd).
- list_del_init(&ctx.sqd_list).
- io_sqd_update_thread_idle(sqd).
- io_sq_thread_unpark(sqd).
- io_put_sq_data(sqd).
- ctx.sq_data = NULL.

REQ-9: io_attach_sq_data(p):
- CLASS(fd, f)(p.wq_fd). if fd_empty(f): return -ENXIO.
- if !io_is_uring_fops(fd_file(f)): return -EINVAL.
- ctx_attach = fd_file(f).private_data.
- sqd = ctx_attach.sq_data. if !sqd: return -EINVAL.
- /* Cross-tgid prohibited */
- if sqd.task_tgid != current.tgid: return -EPERM.
- refcount_inc(&sqd.refs).
- return sqd.

REQ-10: io_get_sq_data(p, *attached):
- *attached = false.
- if p.flags & IORING_SETUP_ATTACH_WQ:
  - sqd = io_attach_sq_data(p).
  - if !IS_ERR(sqd): *attached = true; return sqd.
  - if PTR_ERR(sqd) != -EPERM: return sqd.
  - /* EPERM ⟹ fall through to create new sqd */
- sqd = kzalloc_obj(*sqd). if !sqd: return -ENOMEM.
- atomic_set(&sqd.park_pending, 0). refcount_set(&sqd.refs, 1).
- INIT_LIST_HEAD(&sqd.ctx_list). mutex_init(&sqd.lock).
- init_waitqueue_head(&sqd.wait). init_completion(&sqd.exited).
- return sqd.

REQ-11: __io_sq_thread(ctx, sqd, cap_entries, ist) — per-ctx single-pass:
- to_submit = io_sqring_entries(ctx).
- if cap_entries ∧ to_submit > IORING_SQPOLL_CAP_ENTRIES_VALUE: to_submit = 8.
- if to_submit ∨ !list_empty(&ctx.iopoll_list):
  - io_sq_start_worktime(ist).
  - if ctx.sq_creds != current_cred(): creds = override_creds(ctx.sq_creds).
  - mutex_lock(&ctx.uring_lock).
  - if !list_empty(&ctx.iopoll_list): io_do_iopoll(ctx, true).
  - if to_submit ∧ !percpu_ref_is_dying(&ctx.refs) ∧ !(ctx.flags & IORING_SETUP_R_DISABLED):
    - ret = io_submit_sqes(ctx, to_submit).
  - mutex_unlock(&ctx.uring_lock).
  - if to_submit ∧ wq_has_sleeper(&ctx.sqo_sq_wait): wake_up(&ctx.sqo_sq_wait).
  - if creds: revert_creds(creds).
- return ret.

REQ-12: io_sqd_handle_event(sqd) — park-or-signal:
- if test_bit(IO_SQ_THREAD_SHOULD_PARK, &sqd.state) ∨ signal_pending(current):
  - mutex_unlock(&sqd.lock).
  - if signal_pending(current): did_sig = get_signal(&ksig).
  - wait_event(sqd.wait, !atomic_read(&sqd.park_pending)).
  - mutex_lock(&sqd.lock).
  - sqd.sq_cpu = raw_smp_processor_id().
- return did_sig ∨ test_bit(IO_SQ_THREAD_SHOULD_STOP, &sqd.state).

REQ-13: io_sq_tw(retry_list, max_entries) — task-work runner:
- tctx = current.io_uring. count = 0.
- if *retry_list: *retry_list = io_handle_tw_list(*retry_list, &count, max_entries). if count >= max_entries: goto out. max_entries -= count.
- *retry_list = tctx_task_work_run(tctx, max_entries, &count).
- out: if task_work_pending(current): task_work_run().
- return count.

REQ-14: io_sq_thread(data) — main loop:
- /* Bootstrap */
- if !current.io_uring: rcu_assign_pointer(sqd.thread, NULL); put_task_struct(current); goto err_out.
- snprintf(buf, "iou-sqp-%d", sqd.task_pid). set_task_comm(current, buf).
- sqd.task_pid = current.pid.
- if sqd.sq_cpu != -1: set_cpus_allowed_ptr(current, cpumask_of(sqd.sq_cpu)).
- else: set_cpus_allowed_ptr(current, cpu_online_mask). sqd.sq_cpu = raw_smp_processor_id().
- /* Force audit setup so prep async-ops don't fault audit */
- audit_uring_entry(IORING_OP_NOP). audit_uring_exit(true, 0).
- mutex_lock(&sqd.lock).
- /* Main loop */
- while 1:
  - sqt_spin = false. ist = {}.
  - /* 1. Event check */
  - if io_sqd_events_pending(sqd) ∨ signal_pending(current):
    - if io_sqd_handle_event(sqd): break.
    - timeout = jiffies + sqd.sq_thread_idle.
  - /* 2. Per-ctx submit */
  - cap_entries = !list_is_singular(&sqd.ctx_list).
  - for ctx in sqd.ctx_list:
    - ret = __io_sq_thread(ctx, sqd, cap_entries, &ist).
    - if !sqt_spin ∧ (ret > 0 ∨ !list_empty(&ctx.iopoll_list)): sqt_spin = true.
  - /* 3. Task-work */
  - if io_sq_tw(&retry_list, IORING_TW_CAP_ENTRIES_VALUE): sqt_spin = true.
  - /* 4. NAPI busy poll per-ctx */
  - for ctx in sqd.ctx_list:
    - if io_napi(ctx): io_sq_start_worktime(&ist). io_napi_sqpoll_busy_poll(ctx).
  - io_sq_update_worktime(sqd, &ist).
  - /* 5. Spin or sleep decision */
  - if sqt_spin ∨ !time_after(jiffies, timeout):
    - if sqt_spin: timeout = jiffies + sqd.sq_thread_idle.
    - if need_resched(): mutex_unlock(&sqd.lock). cond_resched(). mutex_lock(&sqd.lock). sqd.sq_cpu = raw_smp_processor_id().
    - continue.
  - /* 6. Idle: prepare wait, advertise NEED_WAKEUP */
  - prepare_to_wait(&sqd.wait, &wait, TASK_INTERRUPTIBLE).
  - if !io_sqd_events_pending(sqd) ∧ !io_sq_tw_pending(retry_list):
    - needs_sched = true.
    - for ctx in sqd.ctx_list:
      - atomic_or(IORING_SQ_NEED_WAKEUP, &ctx.rings.sq_flags).
      - if (ctx.flags & IORING_SETUP_IOPOLL) ∧ !list_empty(&ctx.iopoll_list): needs_sched = false; break.
      - smp_mb__after_atomic().  /* pair with userspace SQ-tail load */
      - if io_sqring_entries(ctx): needs_sched = false; break.
    - if needs_sched: mutex_unlock(&sqd.lock). schedule(). mutex_lock(&sqd.lock). sqd.sq_cpu = raw_smp_processor_id().
    - for ctx in sqd.ctx_list: atomic_andnot(IORING_SQ_NEED_WAKEUP, &ctx.rings.sq_flags).
  - finish_wait(&sqd.wait, &wait).
  - timeout = jiffies + sqd.sq_thread_idle.
- /* Exit */
- if retry_list: io_sq_tw(&retry_list, UINT_MAX).
- io_uring_cancel_generic(true, sqd).
- rcu_assign_pointer(sqd.thread, NULL).
- put_task_struct(current).
- for ctx in sqd.ctx_list: atomic_or(IORING_SQ_NEED_WAKEUP, &ctx.rings.sq_flags).
- io_run_task_work(). mutex_unlock(&sqd.lock).
- err_out: complete(&sqd.exited). do_exit(0).

REQ-15: io_sqpoll_wait_sq(ctx) — userspace SQ-full wait:
- do:
  - if !io_sqring_full(ctx): break.
  - prepare_to_wait(&ctx.sqo_sq_wait, &wait, TASK_INTERRUPTIBLE).
  - if !io_sqring_full(ctx): break.
  - schedule().
- while (!signal_pending(current)).
- finish_wait(&ctx.sqo_sq_wait, &wait).

REQ-16: io_sq_offload_create(ctx, p) — IORING_SETUP_SQPOLL handler:
- /* ATTACH_WQ without SQPOLL: validate fd is io_uring */
- if (ctx.flags & (IORING_SETUP_ATTACH_WQ | IORING_SETUP_SQPOLL)) == IORING_SETUP_ATTACH_WQ:
  - CLASS(fd, f)(p.wq_fd). if fd_empty(f): return -ENXIO. if !io_is_uring_fops(fd_file(f)): return -EINVAL.
- if ctx.flags & IORING_SETUP_SQPOLL:
  - ret = security_uring_sqpoll(). if ret: return ret.
  - sqd = io_get_sq_data(p, &attached). on err: goto err.
  - ctx.sq_creds = get_current_cred().
  - ctx.sq_data = sqd.
  - ctx.sq_thread_idle = msecs_to_jiffies(p.sq_thread_idle). if 0: ctx.sq_thread_idle = HZ.
  - io_sq_thread_park(sqd).
  - list_add(&ctx.sqd_list, &sqd.ctx_list).
  - io_sqd_update_thread_idle(sqd).
  - /* Forbid attach to dying thread */
  - ret = (attached ∧ !sqd.thread) ? -ENXIO : 0.
  - io_sq_thread_unpark(sqd).
  - if ret < 0: goto err.
  - if attached: return 0.
  - /* New thread: parse SQ_AFF */
  - if p.flags & IORING_SETUP_SQ_AFF:
    - cpu = p.sq_thread_cpu. if cpu >= nr_cpu_ids ∨ !cpu_online(cpu): return -EINVAL.
    - allowed_mask = alloc_cpumask_var(GFP_KERNEL). if NULL: return -ENOMEM.
    - cpuset_cpus_allowed(current, allowed_mask).
    - if !cpumask_test_cpu(cpu, allowed_mask): free; return -EINVAL.
    - free_cpumask_var(allowed_mask). sqd.sq_cpu = cpu.
  - else: sqd.sq_cpu = -1.
  - sqd.task_pid = current.pid. sqd.task_tgid = current.tgid.
  - tsk = create_io_thread(io_sq_thread, sqd, NUMA_NO_NODE). on err: return PTR_ERR(tsk).
  - mutex_lock(&sqd.lock). rcu_assign_pointer(sqd.thread, tsk). mutex_unlock(&sqd.lock).
  - get_task_struct(tsk). tctx = io_uring_alloc_task_context(tsk, ctx).
  - if !IS_ERR(tctx): tsk.io_uring = tctx. else: ret = PTR_ERR(tctx).
  - wake_up_new_task(tsk).
  - if ret: goto err.
- else if p.flags & IORING_SETUP_SQ_AFF: return -EINVAL.  /* SQ_AFF without SQPOLL illegal */
- return 0.
- err_sqpoll: complete(&ctx.sq_data.exited).
- err: io_sq_thread_finish(ctx). return ret.

REQ-17: io_sqpoll_wq_cpu_affinity(ctx, mask) — IORING_REGISTER_IOWQ_AFF (sqpoll case):
- sqd = ctx.sq_data. if !sqd: return -EINVAL.
- io_sq_thread_park(sqd).
- tsk = sqpoll_task_locked(sqd).
- if tsk: ret = io_wq_cpu_affinity(tsk.io_uring, mask).
- io_sq_thread_unpark(sqd).
- return ret.

REQ-18: Per-IORING_SQ_NEED_WAKEUP flag:
- Set on every ctx.rings.sq_flags before schedule().
- Cleared on wake.
- Userspace must check sq_flags & NEED_WAKEUP and call io_uring_enter(... IORING_ENTER_SQ_WAKEUP) if set.

## Acceptance Criteria

- [ ] AC-1: IORING_SETUP_SQPOLL: kernel thread iou-sqp-<pid> created with appropriate task_comm.
- [ ] AC-2: IORING_SETUP_SQ_AFF without IORING_SETUP_SQPOLL ⟹ -EINVAL.
- [ ] AC-3: IORING_SETUP_SQ_AFF + cpu offline or out-of-range ⟹ -EINVAL.
- [ ] AC-4: IORING_SETUP_SQ_AFF + cpu not in cpuset.cpus_allowed ⟹ -EINVAL.
- [ ] AC-5: IORING_SETUP_ATTACH_WQ across tgid boundary ⟹ -EPERM (falls back to new sqd).
- [ ] AC-6: IORING_SETUP_ATTACH_WQ to non-SQPOLL ctx ⟹ -EINVAL.
- [ ] AC-7: Attach to dying SQPOLL thread (sqd.thread NULL) ⟹ -ENXIO.
- [ ] AC-8: sq_thread_idle = 0 ⟹ defaults to HZ (1 second).
- [ ] AC-9: Multi-ctx sqd: sq_thread_idle is max across ctx_list.
- [ ] AC-10: Multi-ctx sqd: cap_entries = true ⟹ per-iter submit capped at 8.
- [ ] AC-11: Idle expired: IORING_SQ_NEED_WAKEUP set, thread schedules; cleared on wake.
- [ ] AC-12: Park: park_pending atomic counter respects concurrent parkers (double-park serialized).
- [ ] AC-13: Stop: IO_SQ_THREAD_SHOULD_STOP set, thread completes &sqd.exited.
- [ ] AC-14: io_put_sq_data drops final ref ⟹ thread stopped and sqd freed.
- [ ] AC-15: IORING_REGISTER_IOWQ_AFF on SQPOLL ctx: parks thread, sets affinity, unparks.

## Architecture

```
struct IoSqData {
  lock: Mutex,
  wait: WaitQueueHead,
  ctx_list: ListHead<IoRingCtx>,
  sq_thread_idle: u32,                  // jiffies
  sq_cpu: i32,                          // -1 = unbound
  refs: RefCount,
  park_pending: AtomicI32,
  state: AtomicULong,                   // STOP | PARK
  thread: RcuPointer<TaskStruct>,
  task_pid: i32,
  task_tgid: i32,
  sq_creds: *Cred,
  exited: Completion,
  work_time: u64,
}

struct IoSqTime {
  started: bool,
  usec: u64,
}

const IORING_SQPOLL_CAP_ENTRIES_VALUE: u32 = 8;
const IORING_TW_CAP_ENTRIES_VALUE: i32 = 32;

#[repr(u8)]
enum SqThreadStateBit {
  ShouldStop = 0,
  ShouldPark = 1,
}
```

`Sqpoll::thread_park(sqd)`:
1. atomic_inc(&sqd.park_pending).
2. set_bit(SHOULD_PARK, &sqd.state).
3. mutex_lock(&sqd.lock).
4. if let Some(tsk) = Sqpoll::task_locked(sqd):
   - debug_assert!(tsk != current).
   - wake_up_process(tsk).

`Sqpoll::thread_unpark(sqd)`:
1. debug_assert!(Sqpoll::task_locked(sqd) != current).
2. clear_bit(SHOULD_PARK, &sqd.state).
3. if atomic_dec_return(&sqd.park_pending) != 0: set_bit(SHOULD_PARK, &sqd.state).
4. mutex_unlock(&sqd.lock).
5. wake_up(&sqd.wait).

`Sqpoll::single_pass(ctx, sqd, cap_entries, ist) -> i32`:
1. let mut to_submit = io_sqring_entries(ctx).
2. if cap_entries ∧ to_submit > 8: to_submit = 8.
3. if to_submit > 0 ∨ !ctx.iopoll_list.is_empty():
   - Sqpoll::start_worktime(ist).
   - let creds = if ctx.sq_creds != current_cred() { Some(override_creds(ctx.sq_creds)) } else { None }.
   - mutex_lock(&ctx.uring_lock).
   - if !ctx.iopoll_list.is_empty(): io_do_iopoll(ctx, true).
   - if to_submit > 0 ∧ !percpu_ref_is_dying(&ctx.refs) ∧ !(ctx.flags & IORING_SETUP_R_DISABLED):
     - ret = io_submit_sqes(ctx, to_submit).
   - mutex_unlock(&ctx.uring_lock).
   - if to_submit > 0 ∧ wq_has_sleeper(&ctx.sqo_sq_wait): wake_up(&ctx.sqo_sq_wait).
   - if let Some(c) = creds: revert_creds(c).
4. return ret.

`Sqpoll::main_loop(data)`:
1. Bootstrap:
   - if current.io_uring.is_none(): rcu_assign_pointer(sqd.thread, None). put_task_struct(current). goto err_out.
   - snprintf(buf, "iou-sqp-{}", sqd.task_pid). set_task_comm(current, buf).
   - sqd.task_pid = current.pid.
   - if sqd.sq_cpu != -1: set_cpus_allowed_ptr(current, cpumask_of(sqd.sq_cpu)).
   - else: set_cpus_allowed_ptr(current, cpu_online_mask). sqd.sq_cpu = raw_smp_processor_id().
   - audit_uring_entry(IORING_OP_NOP). audit_uring_exit(true, 0).
2. mutex_lock(&sqd.lock).
3. loop:
   - let mut sqt_spin = false. let mut ist = IoSqTime::default().
   - /* Event check */
   - if Sqpoll::events_pending(sqd) ∨ signal_pending(current):
     - if Sqpoll::handle_event(sqd): break.
     - timeout = jiffies + sqd.sq_thread_idle.
   - /* Per-ctx submit */
   - let cap_entries = !list_is_singular(&sqd.ctx_list).
   - for ctx in &sqd.ctx_list:
     - let ret = Sqpoll::single_pass(ctx, sqd, cap_entries, &mut ist).
     - if !sqt_spin ∧ (ret > 0 ∨ !ctx.iopoll_list.is_empty()): sqt_spin = true.
   - /* Task-work */
   - if Sqpoll::tw_run(&mut retry_list, IORING_TW_CAP_ENTRIES_VALUE) > 0: sqt_spin = true.
   - /* NAPI */
   - for ctx in &sqd.ctx_list: if io_napi(ctx): Sqpoll::start_worktime(&mut ist). io_napi_sqpoll_busy_poll(ctx).
   - Sqpoll::update_worktime(sqd, &ist).
   - /* Spin / sleep */
   - if sqt_spin ∨ !time_after(jiffies, timeout):
     - if sqt_spin: timeout = jiffies + sqd.sq_thread_idle.
     - if need_resched(): mutex_unlock(&sqd.lock). cond_resched(). mutex_lock(&sqd.lock). sqd.sq_cpu = raw_smp_processor_id().
     - continue.
   - prepare_to_wait(&sqd.wait, &wait, TASK_INTERRUPTIBLE).
   - if !Sqpoll::events_pending(sqd) ∧ !Sqpoll::tw_pending(&retry_list):
     - let mut needs_sched = true.
     - for ctx in &sqd.ctx_list:
       - atomic_or(IORING_SQ_NEED_WAKEUP, &ctx.rings.sq_flags).
       - if (ctx.flags & IORING_SETUP_IOPOLL) ∧ !ctx.iopoll_list.is_empty(): needs_sched = false; break.
       - smp_mb__after_atomic().
       - if io_sqring_entries(ctx) > 0: needs_sched = false; break.
     - if needs_sched: mutex_unlock(&sqd.lock). schedule(). mutex_lock(&sqd.lock). sqd.sq_cpu = raw_smp_processor_id().
     - for ctx in &sqd.ctx_list: atomic_andnot(IORING_SQ_NEED_WAKEUP, &ctx.rings.sq_flags).
   - finish_wait(&sqd.wait, &wait).
   - timeout = jiffies + sqd.sq_thread_idle.
4. /* Exit */
5. if !retry_list.is_empty(): Sqpoll::tw_run(&mut retry_list, u32::MAX).
6. io_uring_cancel_generic(true, sqd).
7. rcu_assign_pointer(sqd.thread, None). put_task_struct(current).
8. for ctx in &sqd.ctx_list: atomic_or(IORING_SQ_NEED_WAKEUP, &ctx.rings.sq_flags).
9. io_run_task_work(). mutex_unlock(&sqd.lock).
10. err_out: complete(&sqd.exited). do_exit(0).

`Sqpoll::offload_create(ctx, p) -> i32`:
1. /* ATTACH_WQ alone: validate */
2. if (ctx.flags & (ATTACH_WQ | SQPOLL)) == ATTACH_WQ:
   - validate p.wq_fd is open + io_uring_fops; else -ENXIO / -EINVAL.
3. if ctx.flags & SQPOLL:
   - security_uring_sqpoll()?.
   - let (sqd, attached) = Sqpoll::get_or_create(p)?.
   - ctx.sq_creds = get_current_cred().
   - ctx.sq_data = Some(sqd).
   - ctx.sq_thread_idle = msecs_to_jiffies(p.sq_thread_idle).
   - if ctx.sq_thread_idle == 0: ctx.sq_thread_idle = HZ.
   - Sqpoll::thread_park(sqd).
   - list_add(&ctx.sqd_list, &sqd.ctx_list).
   - Sqpoll::update_idle(sqd).
   - let ret = if attached ∧ sqd.thread.is_none() { -ENXIO } else { 0 }.
   - Sqpoll::thread_unpark(sqd).
   - if ret < 0: goto err.
   - if attached: return 0.
   - /* New thread */
   - if p.flags & SQ_AFF:
     - let cpu = p.sq_thread_cpu. if cpu >= nr_cpu_ids ∨ !cpu_online(cpu): -EINVAL.
     - let mask = alloc_cpumask_var(GFP_KERNEL).ok_or(-ENOMEM)?.
     - cpuset_cpus_allowed(current, &mask).
     - if !mask.test(cpu): -EINVAL.
     - sqd.sq_cpu = cpu.
   - else: sqd.sq_cpu = -1.
   - sqd.task_pid = current.pid. sqd.task_tgid = current.tgid.
   - let tsk = create_io_thread(Sqpoll::main_loop, sqd, NUMA_NO_NODE)?.
   - mutex_lock(&sqd.lock). rcu_assign_pointer(sqd.thread, tsk). mutex_unlock(&sqd.lock).
   - get_task_struct(tsk). let tctx = io_uring_alloc_task_context(tsk, ctx).
   - if Ok(t): tsk.io_uring = t. wake_up_new_task(tsk).
4. else if p.flags & SQ_AFF: return -EINVAL.
5. return 0.

`Sqpoll::wq_cpu_affinity(ctx, mask) -> i32`:
1. let sqd = ctx.sq_data?. else return -EINVAL.
2. Sqpoll::thread_park(sqd).
3. let ret = match Sqpoll::task_locked(sqd):
   - Some(tsk) ⟹ io_wq_cpu_affinity(tsk.io_uring, mask),
   - None ⟹ -EINVAL,
4. Sqpoll::thread_unpark(sqd).
5. return ret.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `park_pending_balanced` | INVARIANT | per-park/unpark: park_pending counter never < 0. |
| `park_only_holds_lock_on_return` | INVARIANT | per-thread_park: mutex_lock acquired on return. |
| `unpark_only_releases_lock` | INVARIANT | per-thread_unpark: mutex_unlock matches preceding lock. |
| `self_wake_forbidden` | INVARIANT | per-park/unpark/stop: WARN_ON_ONCE(tsk == current). |
| `sq_cpu_in_bounds_or_neg1` | INVARIANT | per-offload_create: sq_cpu ∈ [-1, nr_cpu_ids). |
| `sq_cpu_online_when_aff` | INVARIANT | per-offload_create: SQ_AFF ⟹ cpu_online(sq_cpu). |
| `sq_thread_idle_nonzero` | INVARIANT | per-offload_create: ctx.sq_thread_idle ≥ 1 (zero ⟹ HZ). |
| `attach_same_tgid` | INVARIANT | per-attach: sqd.task_tgid == current.tgid. |
| `sq_aff_requires_sqpoll` | INVARIANT | per-offload_create: SQ_AFF without SQPOLL ⟹ -EINVAL. |
| `need_wakeup_paired` | INVARIANT | per-main_loop: NEED_WAKEUP set before schedule() ∧ cleared after. |
| `submit_under_uring_lock` | INVARIANT | per-single_pass: io_submit_sqes called with ctx.uring_lock held. |
| `creds_override_balanced` | INVARIANT | per-single_pass: override_creds paired with revert_creds. |
| `refcount_no_underflow` | INVARIANT | per-put: refcount_dec_and_test on positive count. |

### Layer 2: TLA+

`io_uring/sqpoll.tla`:
- Per-thread state machine × per-park/unpark × per-stop × per-ctx-list mutation.
- Properties:
  - `safety_no_double_stop` — per-stop: SHOULD_STOP transitions 0→1 only.
  - `safety_park_serializes_lock` — per-park: lock held until unpark or thread sleeps in handle_event.
  - `safety_attach_tgid_match` — per-attach: cross-tgid rejected.
  - `safety_idle_max_across_ctx` — per-update_idle: sq_thread_idle = max(ctx.sq_thread_idle).
  - `safety_cap_entries_fairness` — per-multi-ctx: to_submit ≤ 8.
  - `liveness_thread_exits` — per-stop ⟹ exited completed in finite time.
  - `liveness_userspace_wake_eventually_runs` — per-NEED_WAKEUP + io_uring_enter wake ⟹ thread runs.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sqpoll::thread_park` post: park_pending incremented; SHOULD_PARK set; lock held | `Sqpoll::thread_park` |
| `Sqpoll::thread_unpark` post: lock released; park_pending decremented; wait queue woken | `Sqpoll::thread_unpark` |
| `Sqpoll::thread_stop` post: SHOULD_STOP set; exited completed | `Sqpoll::thread_stop` |
| `Sqpoll::single_pass` post: returns submitted count ≥ 0; creds restored | `Sqpoll::single_pass` |
| `Sqpoll::offload_create` post (ok): sq_data attached; thread alive (or attached path); sq_cpu validated | `Sqpoll::offload_create` |
| `Sqpoll::get_or_create` post: returns sqd with refs ≥ 1 ∨ -ERROR | `Sqpoll::get_or_create` |
| `Sqpoll::attach` post (ok): refcount_inc(&sqd.refs); same tgid | `Sqpoll::attach` |
| `Sqpoll::finish` post: ctx.sq_data == None; ctx removed from sqd.ctx_list | `Sqpoll::finish` |
| `Sqpoll::update_idle` post: sqd.sq_thread_idle == max over ctx_list | `Sqpoll::update_idle` |

### Layer 4: Verus/Creusot functional

`Per-IORING_SETUP_SQPOLL → io_sq_offload_create → create_io_thread(io_sq_thread) → main loop submits SQEs and runs task_work and NAPI poll → idle ⟹ IORING_SQ_NEED_WAKEUP + schedule → userspace io_uring_enter(IORING_ENTER_SQ_WAKEUP) wakes thread; Per-IORING_SETUP_ATTACH_WQ → share existing sqd within same tgid; Per-IORING_REGISTER_IOWQ_AFF → park-set-affinity-unpark` semantic equivalence: per-io_uring_setup(2), io_uring_enter(2), io_uring_register(2).

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

SQPOLL reinforcement:

- **Per-security_uring_sqpoll LSM gate** — defense against per-policy-violating SQPOLL spawn.
- **Per-cross-tgid attach -EPERM** — defense against per-process-isolation breach via shared SQPOLL thread.
- **Per-attach-to-dying-thread -ENXIO** — defense against per-dangling-attach UAF.
- **Per-SQ_AFF cpu_online + cpuset.cpus_allowed check** — defense against per-illegal-CPU pin.
- **Per-SQ_AFF requires SQPOLL** — defense against per-orphan-affinity flag.
- **Per-sq_thread_idle default HZ** — defense against per-busy-loop-by-default if zero passed.
- **Per-IORING_SQ_NEED_WAKEUP + smp_mb before SQ-tail load** — defense against per-userspace-wake race / lost submit.
- **Per-park-pending atomic counter** — defense against per-concurrent-parker lost-unpark.
- **Per-WARN_ON_ONCE(tsk == current)** — defense against per-self-park / -stop deadlock.
- **Per-refcount sqd.refs strict** — defense against per-sqd UAF / leak.
- **Per-creds override + revert balanced** — defense against per-cred-leak across submit.
- **Per-uring_lock held during io_submit_sqes** — defense against per-concurrent-submit-and-register.
- **Per-cap_entries 8 for multi-ctx** — defense against per-ctx starvation.
- **Per-task_work cap 32 (TW_CAP)** — defense against per-task-work-flood blocking submission.
- **Per-IORING_REGISTER_IOWQ_AFF park-set-unpark** — defense against per-affinity-update mid-flight race.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `io_uring_params.sq_thread_cpu`, `sq_thread_idle`, and `wq_fd` are validated on entry; `cpu` is bounds-checked against the submitter's `cpus_allowed`, and the copy is exact-width (no slack).
- **PAX_KERNEXEC** — the SQ poll thread's function table is `__ro_after_init`; `io_sq_thread` cannot be patched at runtime.
- **PAX_RANDKSTACK** — the SQ poll kthread inherits the standard kernel-stack randomization at fork; per-iteration `cond_resched` does not stabilize the stack pointer between submissions.
- **PAX_REFCOUNT** — `sqd->refs`, the per-ctx attach counter, and `task_struct` references held by the poll thread route through the saturating wrapper.
- **PAX_MEMORY_SANITIZE** — on `IORING_SETUP_SQPOLL` teardown, the `io_sq_data` slab is zero-poisoned before `kfree`; task pointer fields cleared so a late wakeup traps.
- **PAX_UDEREF** — params copy from userspace through SMAP-enforced accessors.
- **PAX_RAP / kCFI** — `io_sq_thread` is the kthread entry; kCFI tag matches the kthread signature so an attacker-controlled function pointer cannot reroute it.
- **GRKERNSEC_HIDESYM** — sqd/task pointer never appears in dmesg/audit; only PID is logged.
- **GRKERNSEC_DMESG** — affinity-update failures and CPU-bind rejections are ratelimited per ring.
- **SQ_THREAD CAP_SYS_NICE** — `IORING_SETUP_SQ_AFF` (pinning to a specific CPU) requires CAP_SYS_NICE under grsec, matching the kernel default; further, `sq_thread_idle == 0` (busy-spin) requires CAP_SYS_NICE to avoid a CPU-monopolizing kthread for unprivileged tasks.
- **ATTACH_WQ thread-group strict** — `IORING_SETUP_ATTACH_WQ` is refused if the target ring's `sqd->task_pid` is not in the submitter's thread group AND submitter lacks CAP_SYS_ADMIN; defense against cross-thread-group SQE injection via a shared worker thread.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- io_uring/io_uring.c io_submit_sqes / io_do_iopoll (covered in `io_uring-core.md` Tier-3)
- io_uring/tctx.c io_uring_alloc_task_context / tctx_task_work_run (covered separately if expanded)
- io_uring/napi.c io_napi_sqpoll_busy_poll (covered separately if expanded)
- io_uring/cancel.c io_uring_cancel_generic (covered in `cancel.md` Tier-3)
- io_uring/io-wq.c io_wq_cpu_affinity (covered separately if expanded)
- kernel/cred.c override_creds / revert_creds (covered separately if expanded)
- security/security.c security_uring_sqpoll LSM hook (covered separately if expanded)
- create_io_thread / kernel-thread infrastructure (covered separately if expanded)
- Implementation code
