# Tier-3: kernel/kthread.c — Kernel thread infrastructure

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/kthread.c (~1739 lines)
  - include/linux/kthread.h
  - include/linux/sched.h (PF_KTHREAD)
-->

## Summary

`kernel/kthread.c` is the canonical factory and lifecycle controller for **kernel threads**. A kthread is a `task_struct` flagged with `PF_KTHREAD` and with `task->worker_private` pointing at a `struct kthread` (the per-thread bookkeeping: flags, start completion, exit completion, threadfn pointer, optional `preferred_affinity`, optional `blkcg_css`). Kthreads are spawned exclusively by the **`kthreadd` parent thread** (`pid 2`): callers post a `struct kthread_create_info` to `kthread_create_list`, wake `kthreadd`, which then calls `kernel_thread(kthread, create, ...)`. The trampoline `kthread(_create)` records the resulting `task_struct` in `create->result` and parks itself in `TASK_UNINTERRUPTIBLE` until the creator wakes it. Per-thread lifecycle is driven by three bits in `kthread->flags`: `KTHREAD_IS_PER_CPU`, `KTHREAD_SHOULD_STOP`, `KTHREAD_SHOULD_PARK`. Threads cooperate by polling `kthread_should_stop()` / `kthread_should_park()` and may call `kthread_parkme()`. `kthread_stop(k)` sets the STOP bit, unparks, wakes, then waits on `kthread->exited`. The file also exports the **kthread-worker** subsystem: `kthread_worker_fn` is a generic dispatcher loop that pulls `kthread_work` items from `worker->work_list` and executes their `func`; users build queues with `kthread_create_worker*()`, `kthread_queue_work()`, `kthread_(mod_)delayed_work`, `kthread_flush_work()`, `kthread_cancel_work_sync()`, `kthread_destroy_worker()`. Finally `kthread_use_mm()` / `kthread_unuse_mm()` let a kthread temporarily adopt a userspace address space (the `io_uring` worker pattern, vhost, etc.). Critical for: every async kernel subsystem (workqueues, RCU, ksoftirqd, migration, kswapd, oom_reaper, watchdog ping, io-uring SQ poll, kvm-vcpu thread).

This Tier-3 covers `kernel/kthread.c` (~1739 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kthread` | per-thread bookkeeping (flags, completions, threadfn, affinity) | `Kthread` |
| `struct kthread_create_info` | per-creation request handoff to kthreadd | `KthreadCreateInfo` |
| `struct kthread_worker` | per-worker queue + lock + task | `KthreadWorker` |
| `struct kthread_work` | per-item function + node + worker back-ref | `KthreadWork` |
| `struct kthread_delayed_work` | `kthread_work` + `timer_list` | `KthreadDelayedWork` |
| `struct kthread_flush_work` | one-shot flush sentinel | `KthreadFlushWork` |
| `kthreadd` | per-kthreadd_task forever-loop | `Kthread::kthreadd_main` |
| `kthread_create_on_node()` | per-create entry (NUMA-aware) | `Kthread::create_on_node` |
| `__kthread_create_on_node()` | per-create internal (build create_info, wait completion) | `Kthread::create_on_node_inner` |
| `kthread_create()` macro | per-NUMA_NO_NODE wrapper | macro |
| `kthread_create_on_cpu()` | per-create + bind | `Kthread::create_on_cpu` |
| `kthread_run` macro | per-create + wake_up_process | macro |
| `kthread_bind()` / `kthread_bind_mask()` | per-cpu / per-mask affinity (before first wake) | `Kthread::bind` / `bind_mask` |
| `kthread_set_per_cpu()` | per-mark percpu post-bind | `Kthread::set_per_cpu` |
| `kthread_is_per_cpu()` | per-query | `Kthread::is_per_cpu` |
| `kthread_affine_preferred()` | per-preferred (housekeeping-aware) | `Kthread::affine_preferred` |
| `kthread_park()` / `kthread_unpark()` / `kthread_parkme()` | per-park lifecycle | `Kthread::park` / `unpark` / `parkme` |
| `kthread_should_park()` / `kthread_should_stop()` / `kthread_should_stop_or_park()` / `kthread_freezable_should_stop()` | per-cooperative checks | `Kthread::should_*` |
| `kthread_stop()` / `kthread_stop_put()` | per-stop + wait_for_completion(exited) | `Kthread::stop` / `stop_put` |
| `kthread_complete_and_exit()` | per-exit-with-completion | `Kthread::complete_and_exit` |
| `kthread_data()` | per-task `kthread->data` accessor | `Kthread::data` |
| `kthread_func()` | per-task `kthread->threadfn` accessor | `Kthread::func` |
| `kthread_probe_data()` | per-debugger probe (copy_from_kernel_nofault) | `Kthread::probe_data` |
| `set_kthread_struct()` / `free_kthread_struct()` | per-task install/drop | `Kthread::install` / `release` |
| `to_kthread()` / `tsk_is_kthread()` | per-`PF_KTHREAD` cast | inline helpers |
| `get_kthread_comm()` | per-name (incl. full_name) | `Kthread::comm` |
| `__kthread_init_worker()` | per-worker init | `KthreadWorker::init` |
| `kthread_worker_fn()` | per-worker dispatcher loop | `KthreadWorker::run` |
| `kthread_create_worker_on_node()` / `_on_cpu()` | per-worker spawn | `KthreadWorker::create_on_node` / `_on_cpu` |
| `kthread_queue_work()` | per-work enqueue + wake | `KthreadWorker::queue_work` |
| `kthread_queue_delayed_work()` / `kthread_mod_delayed_work()` | per-delayed enqueue | `KthreadWorker::queue_delayed` / `mod_delayed` |
| `kthread_delayed_work_timer_fn()` | per-timer expiry → enqueue | `KthreadWorker::delayed_timer_fn` |
| `kthread_flush_work()` / `kthread_flush_worker()` | per-flush barrier | `KthreadWorker::flush_work` / `flush_worker` |
| `kthread_cancel_work_sync()` / `kthread_cancel_delayed_work_sync()` | per-cancel + drain | `KthreadWorker::cancel_work_sync` / `cancel_delayed_work_sync` |
| `kthread_destroy_worker()` | per-flush + stop + free | `KthreadWorker::destroy` |
| `kthread_use_mm()` / `kthread_unuse_mm()` | per-adopt-userspace-mm (io_uring/vhost) | `Kthread::use_mm` / `unuse_mm` |
| `kthread_associate_blkcg()` / `kthread_blkcg()` | per-blkcg-css attach (CONFIG_BLK_CGROUP) | `Kthread::associate_blkcg` / `blkcg` |
| `kthread_affine_node()` / `kthread_fetch_affinity()` | per-NUMA-default affinity | internal |
| `kthreads_update_affinity()` / `kthreads_update_housekeeping()` / `kthreads_online_cpu()` | per-cpuset/hotplug re-affine | internal |
| `kthread_do_exit()` / `kthread_exit()` | per-exit path | `Kthread::do_exit` |
| `PF_KTHREAD` | per-task flag | shared task_struct flag |
| `TASK_PARKED` (special state) | per-park sleep | shared scheduler state |
| `KTHREAD_IS_PER_CPU` / `KTHREAD_SHOULD_STOP` / `KTHREAD_SHOULD_PARK` | per-kthread flag bits | `KthreadFlag` |
| `KTW_FREEZABLE` | per-worker flag | `KthreadWorkerFlag` |
| `kthread_create_lock` / `kthread_create_list` | per-handoff queue + lock | internal |
| `kthread_affinity_lock` / `kthread_affinity_list` | per-preferred-affinity registry | internal |

## Compatibility contract

REQ-1: `struct kthread` bookkeeping:
- flags: bitset over `KTHREAD_IS_PER_CPU=0`, `KTHREAD_SHOULD_STOP=1`, `KTHREAD_SHOULD_PARK=2`.
- cpu: u32 — bound CPU when `KTHREAD_IS_PER_CPU` is set.
- node: i32 — NUMA node (NUMA_NO_NODE = unbound).
- started: 0/1 — set to 1 after the first wake.
- result: i32 — return value passed back to `kthread_stop`.
- threadfn: `fn(*mut ()) -> i32` — the user-supplied entry.
- data: `*mut ()` — opaque user payload.
- parked: `Completion` — kthreadd-side handshake for `kthread_park`.
- exited: `Completion` — set on kthread exit; waited by `kthread_stop`.
- blkcg_css: `Option<*CgroupSubsysState>` (CONFIG_BLK_CGROUP).
- full_name: `Option<*c_char>` — full (untruncated) thread name.
- task: `*TaskStruct` — back-pointer.
- affinity_node: list node in `kthread_affinity_list`.
- preferred_affinity: `Option<*Cpumask>` (allocated by `kthread_affine_preferred`).

REQ-2: `struct kthread_create_info`:
- full_name: `*c_char` — kvasprintf'd name.
- threadfn / data / node: from caller.
- result: `*TaskStruct` — written by kthread trampoline before completion.
- done: `*Completion` — caller waits on this (killable).
- list: list node in `kthread_create_list`.

REQ-3: `Kthread::install(p)` (`set_kthread_struct`):
- if to_kthread(p): WARN; return false.
- kthread = kzalloc(struct kthread).
- if !kthread: return false.
- init_completion(&kthread.exited).
- init_completion(&kthread.parked).
- INIT_LIST_HEAD(&kthread.affinity_node).
- p.vfork_done = &kthread.exited.
- kthread.task = p.
- kthread.node = tsk_fork_get_node(current).
- p.worker_private = kthread.
- return true.

REQ-4: `Kthread::release(k)` (`free_kthread_struct`):
- kthread = to_kthread(k); if !kthread: return.
- WARN if kthread.blkcg_css (CONFIG_BLK_CGROUP).
- k.worker_private = NULL.
- kfree(kthread.full_name).
- kfree(kthread).

REQ-5: `Kthread::should_stop()`:
- return test_bit(KTHREAD_SHOULD_STOP, &to_kthread(current).flags).

REQ-6: `Kthread::should_park()`:
- return test_bit(KTHREAD_SHOULD_PARK, &to_kthread(current).flags).

REQ-7: `Kthread::should_stop_or_park()`:
- kthread = tsk_is_kthread(current); if !kthread: return false.
- return kthread.flags & (BIT(SHOULD_STOP) | BIT(SHOULD_PARK)).

REQ-8: `Kthread::freezable_should_stop(was_frozen)`:
- might_sleep.
- if try_to_freeze: frozen = true.
- if was_frozen: *was_frozen = frozen.
- return kthread_should_stop().

REQ-9: `Kthread::data(task)`: return to_kthread(task).data.
- /* Stable; no lock; read-only from the task's perspective once installed. */

REQ-10: `Kthread::probe_data(task)`:
- copy_from_kernel_nofault(&data, &to_kthread(task).data, sizeof(data)).
- return data (or NULL on fault).

REQ-11: `Kthread::parkme()` / `__kthread_parkme(self)`:
- loop:
  - set_special_state(TASK_PARKED).
  - if !test_bit(KTHREAD_SHOULD_PARK, &self.flags): break.
  - preempt_disable.
  - complete(&self.parked).
  - schedule_preempt_disabled.
  - preempt_enable.
- __set_current_state(TASK_RUNNING).

REQ-12: `Kthread::create_on_node(threadfn, data, node, namefmt, ...) -> *TaskStruct`:
- va_start; task = `Kthread::create_on_node_inner(threadfn, data, node, namefmt, args)`; va_end.
- return task.

REQ-13: `Kthread::create_on_node_inner(threadfn, data, node, namefmt, args) -> *TaskStruct`:
- DECLARE_COMPLETION_ONSTACK(done).
- create = kmalloc(struct kthread_create_info, GFP_KERNEL).
- if !create: return ERR_PTR(-ENOMEM).
- create.threadfn = threadfn; create.data = data; create.node = node; create.done = &done.
- create.full_name = kvasprintf(GFP_KERNEL, namefmt, args).
- if !create.full_name: task = ERR_PTR(-ENOMEM); goto free_create.
- spin_lock(&kthread_create_lock).
- list_add_tail(&create.list, &kthread_create_list).
- spin_unlock.
- wake_up_process(kthreadd_task).
- if wait_for_completion_killable(&done) != 0 [killed]:
  - if xchg(&create.done, NULL): return ERR_PTR(-EINTR).
  - wait_for_completion(&done).   /* kthreadd will complete shortly */.
- task = create.result.
- free_create: kfree(create); return task.

REQ-14: `kthreadd(unused)` (`pid 2`):
- set_task_comm("kthreadd").
- ignore_signals(tsk).
- set_mems_allowed(node_states[N_MEMORY]).
- current.flags |= PF_NOFREEZE.
- cgroup_init_kthreadd.
- kthread_affine_node.
- forever:
  - set_current_state(TASK_INTERRUPTIBLE).
  - if list_empty(&kthread_create_list): schedule.
  - __set_current_state(TASK_RUNNING).
  - spin_lock(&kthread_create_lock).
  - while !list_empty(&kthread_create_list):
    - create = list_entry(kthread_create_list.next, struct kthread_create_info, list).
    - list_del_init(&create.list).
    - spin_unlock; `create_kthread(create)`; spin_lock.
  - spin_unlock.

REQ-15: `create_kthread(create)`:
- current.pref_node_fork = create.node (CONFIG_NUMA).
- pid = kernel_thread(kthread, create, create.full_name, CLONE_FS | CLONE_FILES | SIGCHLD).
- if pid < 0:
  - done = xchg(&create.done, NULL).
  - kfree(create.full_name).
  - if !done: kfree(create); return.
  - create.result = ERR_PTR(pid); complete(done).

REQ-16: Trampoline `kthread(_create)` (called by `kernel_thread`):
- create = _create; threadfn = create.threadfn; data = create.data.
- self = to_kthread(current).
- done = xchg(&create.done, NULL).
- if !done: kfree(create.full_name); kfree(create); kthread_exit(-EINTR).
- self.full_name = create.full_name; self.threadfn = threadfn; self.data = data.
- sched_setscheduler_nocheck(current, SCHED_NORMAL, &param=0).
- __set_current_state(TASK_UNINTERRUPTIBLE).
- create.result = current.
- preempt_disable; complete(done); schedule_preempt_disabled; preempt_enable.
- self.started = 1.
- if !(current.flags & PF_NO_SETAFFINITY) ∧ !self.preferred_affinity: kthread_affine_node.
- ret = -EINTR.
- if !test_bit(SHOULD_STOP, &self.flags):
  - cgroup_kthread_ready.
  - __kthread_parkme(self).
  - ret = threadfn(data).
- kthread_exit(ret).

REQ-17: `Kthread::bind(p, cpu)`:
- kthread = to_kthread(p).
- `__kthread_bind(p, cpu, TASK_UNINTERRUPTIBLE)`.
- WARN_ON_ONCE(kthread.started).

REQ-18: `__kthread_bind_mask(p, mask, state)`:
- if !wait_task_inactive(p, state): WARN; return.
- scoped_guard(raw_spinlock_irqsave, &p.pi_lock): set_cpus_allowed_force(p, mask).
- p.flags |= PF_NO_SETAFFINITY.

REQ-19: `Kthread::create_on_cpu(threadfn, data, cpu, namefmt)`:
- p = kthread_create_on_node(threadfn, data, cpu_to_node(cpu), namefmt, cpu).
- if IS_ERR(p): return p.
- kthread_bind(p, cpu).
- to_kthread(p).cpu = cpu.    /* for re-bind on unpark */
- return p.

REQ-20: `Kthread::set_per_cpu(k, cpu)`:
- kthread = to_kthread(k); if !kthread: return.
- WARN_ON_ONCE(!(k.flags & PF_NO_SETAFFINITY)).
- if cpu < 0: clear_bit(IS_PER_CPU, &kthread.flags); return.
- kthread.cpu = cpu; set_bit(IS_PER_CPU, &kthread.flags).

REQ-21: `Kthread::unpark(k)`:
- kthread = to_kthread(k).
- if !test_bit(SHOULD_PARK, &kthread.flags): return.
- if test_bit(IS_PER_CPU, &kthread.flags): `__kthread_bind(k, kthread.cpu, TASK_PARKED)`.
- clear_bit(SHOULD_PARK, &kthread.flags).
- wake_up_state(k, TASK_PARKED).

REQ-22: `Kthread::park(k) -> i32`:
- if WARN_ON(k.flags & PF_EXITING): return -ENOSYS.
- if WARN_ON_ONCE(test_bit(SHOULD_PARK, &kthread.flags)): return -EBUSY.
- set_bit(SHOULD_PARK, &kthread.flags).
- if k != current:
  - wake_up_process(k).
  - wait_for_completion(&kthread.parked).
  - WARN_ON_ONCE(!wait_task_inactive(k, TASK_PARKED)).
- return 0.

REQ-23: `Kthread::stop(k) -> i32`:
- get_task_struct(k).
- kthread = to_kthread(k).
- set_bit(SHOULD_STOP, &kthread.flags).
- kthread_unpark(k).
- set_tsk_thread_flag(k, TIF_NOTIFY_SIGNAL).
- wake_up_process(k).
- wait_for_completion(&kthread.exited).
- ret = kthread.result.
- put_task_struct(k).
- return ret.

REQ-24: `Kthread::stop_put(k) -> i32`:
- ret = kthread_stop(k); put_task_struct(k); return ret.

REQ-25: `Kthread::complete_and_exit(comp, code)` (noreturn):
- if comp: complete(comp).
- kthread_exit(code).

REQ-26: `Kthread::do_exit(kthread, result)` (called from exit path):
- kthread.result = result.
- if !list_empty(&kthread.affinity_node):
  - mutex_lock(&kthread_affinity_lock).
  - list_del(&kthread.affinity_node).
  - mutex_unlock.
  - if kthread.preferred_affinity: kfree(kthread.preferred_affinity); = NULL.

REQ-27: `Kthread::affine_preferred(p, mask) -> i32`:
- if !wait_task_inactive(p, TASK_UNINTERRUPTIBLE) ∨ kthread.started: WARN; return -EINVAL.
- affinity = zalloc_cpumask_var. if fail: return -ENOMEM.
- kthread.preferred_affinity = kzalloc(struct cpumask). if fail: ret = -ENOMEM; goto out.
- mutex_lock(&kthread_affinity_lock).
- cpumask_copy(kthread.preferred_affinity, mask).
- list_add_tail(&kthread.affinity_node, &kthread_affinity_list).
- kthread_fetch_affinity(kthread, affinity).
- scoped_guard(raw_spinlock_irqsave, &p.pi_lock): set_cpus_allowed_force(p, affinity).
- mutex_unlock.
- out: free_cpumask_var(affinity); return ret.

REQ-28: `kthread_fetch_affinity(kthread, cpumask)`:
- guard(rcu).
- pref = kthread.preferred_affinity OR cpumask_of_node(kthread.node) OR housekeeping_cpumask(HK_TYPE_DOMAIN).
- cpumask_and(cpumask, pref, housekeeping_cpumask(HK_TYPE_DOMAIN)).
- if cpumask_empty(cpumask): cpumask_copy(cpumask, housekeeping_cpumask(HK_TYPE_DOMAIN)).

REQ-29: `kthreads_update_affinity(force)`:
- guard(mutex)(&kthread_affinity_lock).
- if list_empty(&kthread_affinity_list): return 0.
- affinity = zalloc_cpumask_var. if fail: return -ENOMEM.
- for k in kthread_affinity_list:
  - if (k.task.flags & PF_NO_SETAFFINITY) ∨ kthread_is_per_cpu(k.task): WARN; ret = -EINVAL; continue.
  - if force ∨ k.preferred_affinity ∨ k.node != NUMA_NO_NODE:
    - kthread_fetch_affinity(k, affinity).
    - set_cpus_allowed_ptr(k.task, affinity).
- free_cpumask_var.
- return ret.

REQ-30: `Kthread::init_worker(worker, name, key)` (`__kthread_init_worker`):
- memset(worker, 0).
- raw_spin_lock_init(&worker.lock).
- INIT_LIST_HEAD(&worker.work_list).
- INIT_LIST_HEAD(&worker.delayed_work_list).

REQ-31: `KthreadWorker::run(worker_ptr) -> i32` (`kthread_worker_fn`):
- worker = worker_ptr.
- WARN_ON(worker.task ∧ worker.task != current).
- worker.task = current.
- if worker.flags & KTW_FREEZABLE: set_freezable.
- repeat:
  - set_current_state(TASK_INTERRUPTIBLE).   /* mb-paired w/ kthread_stop */
  - if kthread_should_stop:
    - __set_current_state(TASK_RUNNING).
    - raw_spin_lock_irq(&worker.lock); worker.task = NULL; raw_spin_unlock_irq.
    - return 0.
  - work = NULL.
  - raw_spin_lock_irq(&worker.lock).
  - if !list_empty(&worker.work_list):
    - work = list_first_entry(&worker.work_list, struct kthread_work, node).
    - list_del_init(&work.node).
  - worker.current_work = work.
  - raw_spin_unlock_irq.
  - if work:
    - __set_current_state(TASK_RUNNING).
    - work.func(work).
  - else if !freezing(current): schedule.
  - else: __set_current_state(TASK_RUNNING).
  - try_to_freeze; cond_resched; goto repeat.

REQ-32: `KthreadWorker::create_on_node(flags, node, namefmt, ...)`:
- worker = kzalloc(struct kthread_worker). if !worker: return ERR_PTR(-ENOMEM).
- kthread_init_worker(worker).
- task = __kthread_create_on_node(kthread_worker_fn, worker, node, namefmt, args).
- if IS_ERR(task): kfree(worker); return ERR_CAST(task).
- worker.flags = flags; worker.task = task; return worker.

REQ-33: `KthreadWorker::create_on_cpu(cpu, flags, namefmt)`:
- worker = kthread_create_worker_on_node(flags, cpu_to_node(cpu), namefmt, cpu).
- if !IS_ERR(worker): kthread_bind(worker.task, cpu).
- return worker.

REQ-34: `KthreadWorker::queue_work(worker, work) -> bool`:
- raw_spin_lock_irqsave(&worker.lock).
- if !queuing_blocked(worker, work):  /* not already pending AND not canceling */
  - kthread_insert_work(worker, work, &worker.work_list).
  - ret = true.
- raw_spin_unlock_irqrestore; return ret.

REQ-35: `kthread_insert_work(worker, work, pos)`:
- WARN_ON_ONCE(!list_empty(&work.node)).
- WARN_ON_ONCE(work.worker ∧ work.worker != worker).
- list_add_tail(&work.node, pos).
- work.worker = worker.
- if !worker.current_work ∧ likely(worker.task): wake_up_process(worker.task).

REQ-36: `KthreadWorker::delayed_timer_fn(t)`:
- dwork = timer_container_of(dwork, t, timer).
- work = &dwork.work.
- worker = work.worker.
- WARN_ON_ONCE(!worker) ⟹ return.
- raw_spin_lock_irqsave(&worker.lock).
- WARN_ON_ONCE(list_empty(&work.node)).
- list_del_init(&work.node).
- if !work.canceling: kthread_insert_work(worker, work, &worker.work_list).
- raw_spin_unlock_irqrestore.

REQ-37: `KthreadWorker::queue_delayed(worker, dwork, delay) -> bool` / `__kthread_queue_delayed_work`:
- enqueues `dwork.work` on `worker.delayed_work_list` and arms `dwork.timer` to fire `kthread_delayed_work_timer_fn` at `jiffies + delay`. delay == 0 ⟹ immediate `kthread_insert_work`.

REQ-38: `KthreadWorker::mod_delayed(worker, dwork, delay) -> bool`:
- Lock-acquire; if !work.worker: fast_queue. Else cancel timer + cancel from list, then `__kthread_queue_delayed_work`.

REQ-39: `KthreadWorker::flush_work(work)`:
- worker = work.worker; if !worker: return.
- fwork = { KTHREAD_WORK_INIT(fwork.work, kthread_flush_work_fn), COMPLETION_INITIALIZER_ONSTACK(fwork.done) }.
- raw_spin_lock_irq(&worker.lock).
- if !list_empty(&work.node): kthread_insert_work(worker, &fwork.work, work.node.next).
- else if worker.current_work == work: kthread_insert_work(worker, &fwork.work, worker.work_list.next).
- else: noop = true.
- raw_spin_unlock_irq.
- if !noop: wait_for_completion(&fwork.done).

REQ-40: `KthreadWorker::flush_worker(worker)`:
- fwork = ... as above.
- kthread_queue_work(worker, &fwork.work).
- wait_for_completion(&fwork.done).

REQ-41: `__kthread_cancel_work_sync(work, is_dwork) -> bool`:
- worker = work.worker; if !worker: return false.
- raw_spin_lock_irqsave(&worker.lock).
- if is_dwork: kthread_cancel_delayed_work_timer(work, &flags).
- ret = __kthread_cancel_work(work).
- if worker.current_work == work:
  - work.canceling++; unlock; kthread_flush_work(work); lock; work.canceling--.
- unlock; return ret.

REQ-42: `KthreadWorker::destroy(worker)`:
- task = worker.task; if !task: WARN; return.
- kthread_flush_worker(worker).
- kthread_stop(task).
- WARN if delayed_work_list or work_list non-empty.
- kfree(worker).

REQ-43: `Kthread::use_mm(mm)`:
- WARN if !(tsk.flags & PF_KTHREAD).
- WARN if tsk.mm.
- WARN if !mm.user_ns.
- mmgrab(mm).
- task_lock(tsk); local_irq_disable.
- active_mm = tsk.active_mm.
- tsk.active_mm = mm; tsk.mm = mm.
- membarrier_update_current_mm(mm).
- switch_mm_irqs_off(active_mm, mm, tsk).
- local_irq_enable; task_unlock.
- finish_arch_post_lock_switch (arch-conditional).
- mmdrop_lazy_tlb(active_mm).

REQ-44: `Kthread::unuse_mm(mm)`:
- WARN if !(tsk.flags & PF_KTHREAD).
- WARN if !tsk.mm.
- task_lock(tsk).
- smp_mb__after_spinlock.
- local_irq_disable.
- tsk.mm = NULL.
- membarrier_update_current_mm(NULL).
- mmgrab_lazy_tlb(mm).
- enter_lazy_tlb(mm, tsk).
- local_irq_enable; task_unlock.
- mmdrop(mm).

REQ-45: `Kthread::associate_blkcg(css)` (CONFIG_BLK_CGROUP):
- if !(current.flags & PF_KTHREAD): return.
- if !kthread: return.
- if kthread.blkcg_css: css_put; kthread.blkcg_css = NULL.
- if css: css_get; kthread.blkcg_css = css.

REQ-46: `Kthread::blkcg()`:
- if !(current.flags & PF_KTHREAD): return NULL.
- return kthread.blkcg_css.

REQ-47: NUMA on fork:
- tsk_fork_get_node(tsk): for kthreadd_task, return pref_node_fork; else NUMA_NO_NODE.

REQ-48: `KTHREAD_WORK_INIT(work, fn)`: `{ .node = LIST_HEAD_INIT(work.node), .func = fn }`.

REQ-49: `KTW_FREEZABLE` (= 1): worker honors `set_freezable` and may be frozen at safepoints in `kthread_worker_fn`.

REQ-50: `kthread_run(threadfn, data, namefmt, …)` macro:
- task = kthread_create(...). if !IS_ERR(task): wake_up_process(task). return task.

## Acceptance Criteria

- [ ] AC-1: `kthread_create_on_node` returns a task with `PF_KTHREAD` set and `task->mm == NULL`.
- [ ] AC-2: `kthread_run` wakes the new task; the threadfn runs after `wake_up_process`.
- [ ] AC-3: `kthread_stop` causes `kthread_should_stop()` to return true and returns `threadfn`'s value.
- [ ] AC-4: `kthread_park` (k != current) blocks until `kthread_parkme()` completes, then returns 0.
- [ ] AC-5: `kthread_unpark` clears SHOULD_PARK and wakes the thread from TASK_PARKED.
- [ ] AC-6: `kthread_park` on `current` only sets SHOULD_PARK and returns 0.
- [ ] AC-7: `kthread_park` on a PF_EXITING task returns -ENOSYS.
- [ ] AC-8: `kthread_bind` before first wake binds and sets PF_NO_SETAFFINITY.
- [ ] AC-9: `kthread_create_on_cpu` binds to `cpu` and records `to_kthread(p)->cpu` for re-bind on unpark.
- [ ] AC-10: `kthread_data(task)` is stable for the life of the kthread.
- [ ] AC-11: `kthread_queue_work` returns false if work is already on a list or canceling, true otherwise.
- [ ] AC-12: `kthread_flush_work` returns only after the work has executed (or never queued).
- [ ] AC-13: `kthread_cancel_work_sync` blocks while `worker->current_work == work` and prevents re-queue (work.canceling > 0).
- [ ] AC-14: `kthread_destroy_worker` flushes, stops, then frees the worker.
- [ ] AC-15: `kthread_use_mm` requires `PF_KTHREAD` and `tsk->mm == NULL`; switches `active_mm` and `mm`.
- [ ] AC-16: `kthread_unuse_mm` restores lazy-TLB state (`mmgrab_lazy_tlb` / `enter_lazy_tlb`); `tsk->mm = NULL`.
- [ ] AC-17: `kthreadd` (pid 2) loops servicing `kthread_create_list`; never freezes (PF_NOFREEZE).
- [ ] AC-18: `kthread_freezable_should_stop` enters refrigerator and returns `should_stop`.

## Architecture

```
struct Kthread {
  flags: AtomicU64,                       // bits: IS_PER_CPU, SHOULD_STOP, SHOULD_PARK
  cpu: u32,
  node: i32,
  started: u8,
  result: i32,
  threadfn: fn(*mut ()) -> i32,
  data: *mut (),
  parked: Completion,
  exited: Completion,
  blkcg_css: Option<*CgroupSubsysState>,  // CONFIG_BLK_CGROUP
  full_name: Option<*c_char>,
  task: *TaskStruct,
  affinity_node: ListHead,
  preferred_affinity: Option<*Cpumask>,
}

struct KthreadCreateInfo {
  full_name: *c_char,
  threadfn: fn(*mut ()) -> i32,
  data: *mut (),
  node: i32,
  result: *TaskStruct,
  done: *Completion,
  list: ListHead,
}

struct KthreadWorker {
  lock: RawSpinLock,
  flags: u32,                              // KTW_FREEZABLE
  work_list: ListHead,
  delayed_work_list: ListHead,
  task: *TaskStruct,
  current_work: *KthreadWork,
}

struct KthreadWork {
  node: ListHead,
  func: fn(&KthreadWork),
  worker: *KthreadWorker,
  canceling: AtomicI32,
}

struct KthreadDelayedWork {
  work: KthreadWork,
  timer: TimerList,
}
```

`Kthread::create_on_node(threadfn, data, node, namefmt, args) -> Result<*TaskStruct, Errno>`:
1. let mut create = Box::new(KthreadCreateInfo { threadfn, data, node, done: &done, full_name: kvasprintf(namefmt, args)?, list: ListHead::new(), result: null_mut() }).
2. /* Hand off to kthreadd */
3. {
   - let _g = kthread_create_lock.lock().
   - kthread_create_list.push_back(&create.list).
4. }
5. wake_up_process(kthreadd_task).
6. /* Wait killable in case OOM picks us */
7. match wait_for_completion_killable(&done) {
   - Killed ⟹ if create.done.xchg(NULL).is_some() { return Err(-EINTR) } else { wait_for_completion(&done) }.
   - Ok ⟹ {} .
8. }
9. let task = create.result.
10. drop(create).
11. Ok(task).

`Kthread::kthreadd_main() -> i32`:
1. set_task_comm(current, "kthreadd").
2. ignore_signals(current).
3. set_mems_allowed(node_states[N_MEMORY]).
4. current.flags |= PF_NOFREEZE.
5. cgroup_init_kthreadd; kthread_affine_node.
6. loop:
   - set_current_state(TASK_INTERRUPTIBLE).
   - if kthread_create_list.is_empty(): schedule.
   - __set_current_state(TASK_RUNNING).
   - while let Some(create) = kthread_create_list.pop_front_locked(&kthread_create_lock) { create_kthread(create) }.

`Kthread::stop(k) -> i32`:
1. get_task_struct(k).
2. let kthread = to_kthread(k).
3. kthread.flags.bit(SHOULD_STOP).set().
4. Kthread::unpark(k).
5. set_tsk_thread_flag(k, TIF_NOTIFY_SIGNAL).
6. wake_up_process(k).
7. wait_for_completion(&kthread.exited).
8. let ret = kthread.result.
9. put_task_struct(k).
10. ret.

`Kthread::park(k) -> Result<(), Errno>`:
1. if k.flags & PF_EXITING: return Err(-ENOSYS).
2. let kthread = to_kthread(k).
3. if kthread.flags.bit(SHOULD_PARK).is_set(): return Err(-EBUSY).
4. kthread.flags.bit(SHOULD_PARK).set().
5. if k != current:
   - wake_up_process(k).
   - wait_for_completion(&kthread.parked).
   - debug_assert!(wait_task_inactive(k, TASK_PARKED)).
6. Ok(()).

`KthreadWorker::run(worker_ptr) -> i32`:
1. let worker = worker_ptr.
2. debug_assert!(worker.task == null ∨ worker.task == current).
3. worker.task = current.
4. if worker.flags & KTW_FREEZABLE: set_freezable().
5. loop:
   - set_current_state(TASK_INTERRUPTIBLE).
   - if kthread_should_stop():
     - __set_current_state(TASK_RUNNING).
     - { let _g = worker.lock.lock_irq(); worker.task = null; }.
     - return 0.
   - let mut work: *KthreadWork = null.
   - { let _g = worker.lock.lock_irq();
       if let Some(w) = worker.work_list.pop_front() { list_del_init(&w.node); work = w }.
       worker.current_work = work;
   }.
   - if !work.is_null():
     - __set_current_state(TASK_RUNNING).
     - (work.func)(work).
   - else if !freezing(current): schedule.
   - else: __set_current_state(TASK_RUNNING).
   - try_to_freeze(); cond_resched().

`KthreadWorker::queue_work(worker, work) -> bool`:
1. let _g = worker.lock.lock_irqsave().
2. if !queuing_blocked(worker, work):
   - kthread_insert_work(worker, work, &worker.work_list).
   - return true.
3. false.

`KthreadWorker::flush_work(work)`:
1. let worker = work.worker; if worker.is_null(): return.
2. let fwork = KthreadFlushWork { work: KTHREAD_WORK_INIT(_, flush_work_fn), done: Completion::on_stack() }.
3. { let _g = worker.lock.lock_irq();
     if !work.node.is_empty(): kthread_insert_work(worker, &fwork.work, work.node.next).
     else if worker.current_work == work: kthread_insert_work(worker, &fwork.work, worker.work_list.next).
     else: noop = true.
4. }
5. if !noop: wait_for_completion(&fwork.done).

`Kthread::use_mm(mm)`:
1. debug_assert!(current.flags & PF_KTHREAD).
2. debug_assert!(current.mm.is_null()).
3. debug_assert!(mm.user_ns.is_some()).
4. mmgrab(mm).
5. task_lock(current).
6. local_irq_disable().
7. let active_mm = current.active_mm.
8. current.active_mm = mm; current.mm = mm.
9. membarrier_update_current_mm(mm).
10. switch_mm_irqs_off(active_mm, mm, current).
11. local_irq_enable().
12. task_unlock(current).
13. mmdrop_lazy_tlb(active_mm).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `kthread_struct_installed_before_threadfn` | INVARIANT | per-trampoline: to_kthread(current) != NULL when threadfn runs. |
| `create_done_consumed_exactly_once` | INVARIANT | per-create: xchg(&create.done, NULL) succeeds exactly once. |
| `pf_kthread_set_for_kthreads` | INVARIANT | per-spawn: task.flags & PF_KTHREAD. |
| `task_mm_null_until_use_mm` | INVARIANT | per-kthread: task.mm is NULL unless kthread_use_mm was called. |
| `use_mm_only_for_pf_kthread` | INVARIANT | per-kthread_use_mm: PF_KTHREAD ∧ !tsk.mm. |
| `unuse_mm_balanced` | INVARIANT | per-use_mm: matching unuse_mm before exit. |
| `kthread_bind_only_pre_start` | INVARIANT | per-bind: kthread.started == 0. |
| `park_completion_required` | INVARIANT | per-kthread_park(k != current): parked completion waited. |
| `stop_waits_exited` | INVARIANT | per-kthread_stop: exited completion waited before put. |
| `worker_lock_held_for_queue` | INVARIANT | per-queue_work: worker.lock held during insertion. |
| `worker_current_work_clears_after_func` | INVARIANT | per-worker loop: current_work cleared on next iteration. |
| `delayed_timer_irqsafe` | INVARIANT | per-delayed_work_timer_fn: called with IRQs off; lock irqsave. |
| `cancel_sync_blocks_requeue` | INVARIANT | per-cancel_work_sync: work.canceling > 0 during flush. |
| `destroy_worker_after_flush_and_stop` | INVARIANT | per-destroy: flush_worker → stop → kfree (order). |

### Layer 2: TLA+

`kernel/kthread.tla`:
- Per-create_on_node (post + wait) / kthreadd dispatch / trampoline / park / unpark / stop / worker_fn / queue_work / flush_work / cancel_work_sync.
- Properties:
  - `safety_kthread_unique_owner` — per-task.worker_private: at most one kthread struct.
  - `safety_park_requires_handshake` — per-park (k != current): completes only after kthread_parkme.
  - `safety_stop_eventually_exits` — per-stop: kthread.exited signalled within bounded steps.
  - `safety_worker_processes_at_most_one_at_a_time` — per-worker.current_work serializes work.func.
  - `safety_queue_work_idempotent_while_pending` — per-queue_work on pending: returns false; no double-enqueue.
  - `safety_use_unuse_mm_balanced` — per-kthread: stack of use_mm == stack of unuse_mm before exit.
  - `liveness_create_eventually_runs` — per-create: kthreadd eventually spawns and threadfn eventually runs (unless killed/EINTR).
  - `liveness_flush_eventually_completes` — per-flush_work: work executes ⟹ flush returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `create_on_node_inner` post: Ok(task) ⟹ task.flags & PF_KTHREAD ∧ to_kthread(task) != NULL | `Kthread::create_on_node_inner` |
| `kthreadd_main` invariant: kthread_create_list drained whenever woken; PF_NOFREEZE held | `Kthread::kthreadd_main` |
| `bind` pre: !kthread.started; post: PF_NO_SETAFFINITY set | `Kthread::bind` |
| `park` post (k != current): TASK_PARKED ∧ SHOULD_PARK | `Kthread::park` |
| `unpark` post: !SHOULD_PARK ∧ wake_up_state(k, TASK_PARKED) | `Kthread::unpark` |
| `stop` post: SHOULD_STOP set ∧ exited waited ∧ refcount balanced | `Kthread::stop` |
| `queue_work` post: ret ⟺ inserted ∧ work.worker == worker | `KthreadWorker::queue_work` |
| `flush_work` post: returns ⟹ either never queued or work.func observed completion | `KthreadWorker::flush_work` |
| `cancel_work_sync` post: ret ⟺ pending-then-removed; on return work is neither pending nor running | `KthreadWorker::cancel_work_sync` |
| `destroy_worker` post: worker freed ∧ task stopped ∧ both lists empty | `KthreadWorker::destroy` |
| `use_mm` post: current.mm == mm ∧ current.active_mm == mm ∧ mmgrab balanced by mmdrop_lazy_tlb | `Kthread::use_mm` |
| `unuse_mm` post: current.mm == NULL ∧ lazy_tlb restored ∧ mmdrop balanced | `Kthread::unuse_mm` |

### Layer 4: Verus/Creusot functional

`Per-create posted → kthreadd kernel_thread → trampoline records result & parks UNINTERRUPTIBLE → caller wake_up_process → threadfn runs → cooperative SHOULD_STOP/SHOULD_PARK polling → kthread_stop wakes & waits exited → kthread_do_exit cleans affinity` and `Per-kthread_worker_fn: queue/flush/cancel/delayed semantics equivalent to workqueue.c contract minus per-CPU MQ semantics` and `Per-use_mm/unuse_mm: io_uring SQPOLL / vhost / drm-scheduler invariant preserved` — semantic equivalence: per-Documentation/scheduler/sched-design-CFS.rst, Documentation/core-api/workqueue.rst, and Documentation/io_uring/io_uring.rst.

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

Kthread reinforcement:

- **Per-PF_KTHREAD strict invariant on use_mm/unuse_mm/data/probe_data** — defense against per-non-kthread misuse causing UAF or mm corruption.
- **Per-`set_kthread_struct` once-only (WARN if already installed)** — defense against per-double-install corrupting `worker_private`.
- **Per-`xchg(create.done, NULL)` race-safe handoff** — defense against per-creator-killed-mid-wait double-free of create_info.
- **Per-`wait_for_completion_killable` on create** — defense against per-OOM-deadlock with kthreadd allocating for new thread.
- **Per-`PF_NO_SETAFFINITY` set in `__kthread_bind_mask`** — defense against per-cpuset clobbering kernel-internal binding.
- **Per-`kthread_bind` WARN if `kthread->started`** — defense against per-late-bind racing with already-running threadfn.
- **Per-`kthread_park` WARN if PF_EXITING / already parked** — defense against per-stop/park interleaving.
- **Per-`TASK_PARKED` special-state with `set_special_state`** — defense against per-wakeup losing the park request (store-store collision).
- **Per-`TIF_NOTIFY_SIGNAL` set in `kthread_stop`** — defense against per-`wait_event_*` failing to observe `SHOULD_STOP` because it's parked inside a kernel sleep.
- **Per-`PF_NOFREEZE` on kthreadd** — defense against per-suspend deadlocking thread creation.
- **Per-`worker->lock` raw spinlock IRQ-saving** — defense against per-IRQ-context delayed-work timer-fn vs userspace queue race.
- **Per-`work->canceling` counter blocks re-queue during cancel_work_sync** — defense against per-cancel/queue infinite races.
- **Per-`mmgrab` paired with `mmdrop_lazy_tlb` in use_mm/unuse_mm** — defense against per-mm refcount imbalance and post-free TLB IPIs.
- **Per-`switch_mm_irqs_off` under `task_lock + local_irq_disable`** — defense against per-membarrier IPI observing inconsistent `tsk->mm`.
- **Per-`membarrier_update_current_mm` on both transitions** — defense against per-membarrier {private,global}-expedited missing a kthread mm switch.
- **Per-kthread `blkcg_css` strict get/put in `associate_blkcg`** — defense against per-blkcg-css refcount leak.
- **Per-`kthread_destroy_worker` flush-then-stop-then-free ordering** — defense against per-pending-work-after-free.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `kernel/fork.c` `kernel_thread` / `copy_process` (covered alongside `fork.md` Tier-3).
- Scheduler internals (`schedule`, `wake_up_process`, `wait_task_inactive`, `set_cpus_allowed_force`) — covered in `kernel/sched/` Tier-3s.
- Workqueue (`kernel/workqueue.c`) — distinct subsystem, covered in `workqueue.md` Tier-3.
- Freezer (`kernel/power/process.c`) — covered separately.
- CPU hotplug callback wiring (`cpuhp_setup_state(CPUHP_AP_KTHREADS_ONLINE, ...)`) — covered in `kernel/cpu.c` Tier-3.
- Housekeeping / isolation cpumask (`housekeeping_cpumask`) — covered in `kernel/sched/isolation.c` Tier-3.
- io_uring SQPOLL / vhost specific use of `kthread_use_mm` — covered in their respective subsystem Tier-3s.
- Implementation code.
