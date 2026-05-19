---
title: "Tier-3: kernel/events/core.c — perf_event subsystem architecture (syscall, contexts, ring-buffer, attach)"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

The architectural backbone of `perf_event_open(2)` — the abstract structures, context taxonomy, and lifecycle that every concrete PMU (HW counters, software events, tracepoints, hw-breakpoints, kprobes, uprobes) plugs into. This Tier-3 is the high-level architectural cut of `kernel/events/core.c`; the per-helper exhaustive treatment lives in `kernel/perf/perf-event-core.md`. Here we cover: how `struct perf_event` + `struct perf_event_context` form the per-event / per-domain control planes; how `sys_perf_event_open` constructs an event-fd and decides per-task vs per-cpu vs per-cgroup attach; how `perf_event_alloc` + `perf_event_create_kernel_counter` + `perf_event_release_kernel` are the three legitimate constructors / destructor; what ioctls (PERF_EVENT_IOC_*) drive the fd; and how the mmap'd ring buffer pipes samples back to userspace. Critical for: perf-stat / perf-record, BPF tracing, NMI-watchdog, container-scoped sampling, kernel-internal counter consumers.

This Tier-3 covers `kernel/events/core.c` (~15371 lines) at the architectural level — `struct perf_event`, the context taxonomy, the syscall surface, the fd ABI (ioctl/mmap/poll/read), and the three constructor entry points + matching destructor.

### Acceptance Criteria

- [ ] AC-1: `perf_event_open(attr, -1, 0, -1, 0)` opens a per-cpu event on cpu 0; returns fd ≥ 0.
- [ ] AC-2: `perf_event_open(attr, getpid(), -1, -1, 0)` opens a per-task event on current; survives `sched_setaffinity`.
- [ ] AC-3: `perf_event_open(attr, cgroup_fd, 0, -1, PERF_FLAG_PID_CGROUP)`: only counts when current cgroup matches.
- [ ] AC-4: `ioctl(fd, PERF_EVENT_IOC_ENABLE, 0)` then `read(fd, &count, sizeof(count))`: count increases monotonically.
- [ ] AC-5: `mmap(fd, (1+pow2)*PAGE_SIZE, ...)`: page 0 has `perf_event_mmap_page` magic; data ring at offset PAGE_SIZE.
- [ ] AC-6: Sampling event with `sample_period`: `PERF_RECORD_SAMPLE` records appear in ring buffer at ~sample_period interval.
- [ ] AC-7: Group leader (group_fd=-1) + member (group_fd=leader_fd): both scheduled together or neither.
- [ ] AC-8: `attr.inherit=1` + fork(): child task's counter counts on its own (not parent's).
- [ ] AC-9: `ioctl(fd, PERF_EVENT_IOC_SET_BPF, prog_fd)`: BPF prog runs on each overflow.
- [ ] AC-10: `perf_event_create_kernel_counter`: kernel-counter returns valid event pointer; `overflow_handler` called on each overflow.
- [ ] AC-11: `perf_event_release_kernel`: idempotent; ctx unlinked; ring buffer freed; no UAF.
- [ ] AC-12: nr_active > num_counters: `time_running < time_enabled`; count scaled correctly.
- [ ] AC-13: `perf_event_paranoid=2`: non-root cannot open kernel-tracing events without `CAP_PERFMON`.
- [ ] AC-14: SET_OUTPUT cross-event ring sharing: samples from event A appear in event B's ring buffer.
- [ ] AC-15: poll(fd, POLLIN): returns when ring buffer has data; `data_head - data_tail > wakeup_watermark`.

### Architecture

```
struct Event {                               // = struct perf_event
    list_entry: ListHead,                    // ctx->event_list
    sibling_list: ListHead,                  // group sibling list
    group_leader: *Event,                    // self if leader
    children: ListHead,                      // inherited children
    child_mutex: Mutex,
    ctx: *EventContext,                      // owning context (per-cpu or per-task)
    pmu: *Pmu,                               // dispatch vtable
    pmu_ctx: *EventPmuContext,
    attr: PerfEventAttr,                     // UAPI snapshot
    hw: HwPerfEvent,                         // PMU-private hw state (counter idx, MSR, prev_count, period_left)
    count: AtomicI64,                        // accumulated count
    child_count: AtomicI64,                  // sum from released children
    total_time_enabled: u64, total_time_running: u64,
    state: EventState,                       // OFF / INACTIVE / ACTIVE / ERROR / EXIT / DEAD
    rb: *RingBuffer,                         // mmap'd output (NULL for kernel-counter)
    overflow_handler: Option<fn(*Event, *SampleData, *PtRegs)>,
    overflow_handler_context: *u8,
    bpf_prog: *BpfProg,                      // attached BPF (if any)
    cgrp: *PerfCgroup,                       // cgroup attach (if any)
    owner: *TaskStruct,                      // userspace owner (TOMBSTONE if kernel)
    fasync: *FasyncStruct,
    id: u64,
    ns: AtomicI64,                           // namespace count
    parent: *Event,                          // for inherited child
    ...
}

struct EventContext {                        // = struct perf_event_context
    refcount: AtomicI32,
    mutex: Mutex,
    lock: SpinLock,
    pin_count: AtomicI32,
    nr_events: i32, nr_user: i32, nr_active: i32, is_active: i32,
    nr_stat: i32, nr_freq: i32,
    rotate_disable: i32,
    event_list: ListHead,                    // all events in ctx
    pinned_groups: ListHead, flexible_groups: ListHead,
    task: *TaskStruct,                       // NULL for per-cpu ctx
    time: PerfTimeCtx,
    pmu_ctx_list: ListHead,                  // per-PMU sub-contexts
    cycles_t time_enabled, time_running, generation,
    rcu_head: RcuHead,
    ...
}

struct CpuContext {                          // = struct perf_cpu_context (per-CPU)
    ctx: EventContext,                       // embedded per-cpu ctx
    online: bool,
    task_ctx: *EventContext,                 // task ctx currently active (or NULL)
    sched_cb_usage: i32,
    heap_default: [*EventPmuContext; ...],
    heap: *(*EventPmuContext),
    ...
}

struct EventPmuContext {                     // = struct perf_event_pmu_context
    pmu_ctx_entry: ListHead,                 // in ctx->pmu_ctx_list
    pinned_active: ListHead, flexible_active: ListHead,
    refcount: AtomicI32,
    nr_events: i32, nr_cgroups: i32, nr_freq: i32,
    rotate_necessary: i32,
    pmu: *Pmu,
    ctx: *EventContext,
    pmu_ctx: *CpuPmuContext,                 // per-CPU side
    ...
}
```

`kernel::perf::Subsystem::syscall_perf_event_open(attr_uptr, pid, cpu, group_fd, flags) -> i32`:
1. /* Security check */
2. err = security_perf_event_open(PERF_SECURITY_OPEN).
3. if err: return err.
4. /* Validate flags */
5. if flags & ~PERF_FLAG_ALL: return -EINVAL.
6. /* Copy attr from user (size-versioned) */
7. err = perf_copy_attr(attr_uptr, &attr).
8. if err: return err.
9. /* Validate sample / read formats */
10. err = perf_event_validate_attr(&attr).
11. if err: return err.
12. /* Resolve target task */
13. task = if pid > 0 { find_lively_task_by_vpid(pid) } else if pid == 0 { current } else { None }.
14. if pid > 0 ∧ !task: return -ESRCH.
15. /* Resolve target cpu */
16. if cpu >= nr_cpu_ids ∨ !cpu_online(cpu): return -EINVAL.
17. if pid == -1 ∧ cpu == -1: return -EINVAL.
18. /* Resolve group leader (if any) */
19. group_leader = if group_fd >= 0 { fdget(group_fd).file.private_data } else { NULL }.
20. if group_leader ∧ !is_perf_fd(group_fd): return -EINVAL.
21. /* Cgroup attach? */
22. cgrp_fd = if flags & PERF_FLAG_PID_CGROUP { pid } else { -1 }.
23. /* Allocate event */
24. event = perf_event_alloc(&attr, cpu, task, group_leader, NULL, NULL, NULL, cgrp_fd).
25. if IS_ERR(event): goto err_free.
26. /* Get-or-create context */
27. ctx = find_get_context(task, event).
28. if IS_ERR(ctx): goto err_free_event.
29. /* Anonymously alloc fd */
30. event_file = anon_inode_getfile("[perf_event]", &perf_fops, event, O_RDWR | (flags & PERF_FLAG_FD_CLOEXEC ? O_CLOEXEC : 0)).
31. fd = get_unused_fd_flags(...).
32. /* Install into context (cross-cpu IPI as needed) */
33. perf_install_in_context(ctx, event, cpu).
34. /* Owner */
35. event->owner = current.
36. /* Wire fd */
37. fd_install(fd, event_file).
38. return fd.
39. err_free_event: perf_event_release_kernel(event).
40. err_free: return err.

`Event::alloc(attr, cpu, task, group_leader, parent_event, overflow_handler, context, cgroup_fd) -> Result<*Event, Err>`:
1. /* Refuse impossible combinations */
2. if !task ∧ cpu == -1: return ERR(-EINVAL).
3. if attr.freq ∧ attr.sample_freq == 0: return ERR(-EINVAL).
4. /* Allocate from kmem_cache */
5. event = kmem_cache_zalloc(perf_event_cache, GFP_KERNEL).
6. if !event: return ERR(-ENOMEM).
7. /* Init lists / mutex / refs */
8. INIT_LIST_HEAD(&event.event_entry).
9. INIT_LIST_HEAD(&event.sibling_list); INIT_LIST_HEAD(&event.children).
10. mutex_init(&event.child_mutex).
11. mutex_init(&event.mmap_mutex).
12. atomic_set(&event.refcount, 1).
13. /* Copy attr */
14. event.attr = *attr.
15. event.cpu = cpu.
16. event.parent = parent_event.
17. event.overflow_handler = overflow_handler.
18. event.overflow_handler_context = context.
19. event.state = if attr.disabled { OFF } else { INACTIVE }.
20. /* Group leader */
21. event.group_leader = if group_leader { group_leader } else { event }.
22. /* Resolve PMU by attr.type */
23. err = perf_init_event(event).
24. if err: goto err_pmu.
25. /* Cgroup connect (if cgroup_fd >= 0) */
26. if cgroup_fd >= 0:
    - err = perf_cgroup_connect(cgroup_fd, event, attr, group_leader, &cgrp).
    - if err: goto err_pmu.
    - event.cgrp = cgrp.
27. /* Security event_alloc hook */
28. err = security_perf_event_alloc(event).
29. if err: goto err_cgroup.
30. /* Account global tally */
31. account_event(event).
32. return event.
33. err_*: rollback partial state; kmem_cache_free; return ERR(err).

`Event::create_kernel(attr, cpu, task, overflow_handler, context) -> Result<*Event, Err>`:
1. /* Kernel-internal create: no fd, no userspace owner */
2. event = Event::alloc(attr, cpu, task, NULL, NULL, overflow_handler, context, -1).
3. if IS_ERR(event): return event.
4. /* Get-or-create ctx */
5. ctx = find_get_context(task, event).
6. if IS_ERR(ctx): goto err_free.
7. /* Install */
8. mutex_lock(&ctx.mutex).
9. err = perf_install_in_context(ctx, event, cpu).
10. mutex_unlock(&ctx.mutex).
11. if err: goto err_unlock.
12. /* Mark as kernel-event via sentinel owner */
13. event.owner = TASK_TOMBSTONE.
14. /* Hook up to global kernel-event list (cleanup at module exit, etc.) */
15. perf_event_install_in_owner(event).
16. return event.
17. err_*: perf_event_release_kernel(event); return ERR(err).

`Event::release(event)` (= perf_event_release_kernel):
1. if !event: return 0.
2. /* Walk children, releasing each (recursive via per-child) */
3. mutex_lock(&event.child_mutex).
4. list_for_each_entry_safe(child, n, &event.children, child_list):
   - list_del(&child.child_list).
   - perf_remove_from_context(child, DETACH_GROUP).
   - free_event(child).
5. mutex_unlock(&event.child_mutex).
6. /* Remove from ctx */
7. ctx = perf_event_ctx_lock(event).
8. perf_remove_from_context(event, DETACH_GROUP | DETACH_DEAD).
9. perf_event_ctx_unlock(event, ctx).
10. /* Detach BPF / cgroup / ring buffer */
11. if event.bpf_prog: perf_event_free_bpf_prog(event).
12. if event.cgrp: perf_detach_cgroup(event).
13. if event.rb: ring_buffer_put(event.rb).
14. /* Final teardown */
15. _free_event(event).
16. return 0.

`Event::ioctl(file, cmd, arg)` (= perf_ioctl):
1. event = file->private_data.
2. /* Refuse on dying event */
3. ctx = perf_event_ctx_lock(event).
4. ret = _perf_ioctl(event, cmd, arg).
5. perf_event_ctx_unlock(event, ctx).
6. return ret.

`Event::ioctl_inner(event, cmd, arg)` (= _perf_ioctl):
1. switch cmd:
2.   PERF_EVENT_IOC_ENABLE → perf_event_enable(event); return 0.
3.   PERF_EVENT_IOC_DISABLE → perf_event_disable(event); return 0.
4.   PERF_EVENT_IOC_RESET → perf_event_for_each(event, perf_event_reset); return 0.
5.   PERF_EVENT_IOC_REFRESH → perf_event_refresh(event, (u32)arg); return.
6.   PERF_EVENT_IOC_PERIOD → copy_from_user(&value, arg, sizeof(u64)); perf_event_period(event, value); return.
7.   PERF_EVENT_IOC_ID → put_user(primary_event_id(event), (u64 __user *)arg); return.
8.   PERF_EVENT_IOC_SET_OUTPUT → fd_target = fdget(arg); perf_event_set_output(event, fd_target.file); return.
9.   PERF_EVENT_IOC_SET_FILTER → ftrace_profile_set_filter(event, event.id, (char __user *)arg).
10.  PERF_EVENT_IOC_SET_BPF → perf_event_set_bpf_prog(event, (u32)arg, 0).
11.  PERF_EVENT_IOC_PAUSE_OUTPUT → perf_event_set_paused(event, (u32)arg).
12.  PERF_EVENT_IOC_QUERY_BPF → perf_event_query_prog_array(event, (void __user *)arg).
13.  PERF_EVENT_IOC_MODIFY_ATTRIBUTES → perf_event_modify_attr(event, attr_from_user).
14.  default → return -ENOTTY.

`Event::mmap(file, vma)` (= perf_mmap):
1. event = file->private_data.
2. /* Validate vma: PROT_READ|PROT_WRITE, MAP_SHARED, length is (1 + 2^n) * PAGE_SIZE */
3. if !(vma.vm_flags & VM_SHARED): return -EINVAL.
4. nr_pages = vma_pages(vma).
5. if nr_pages == 1 + 0: /* page 0 only; pseudo-mmap for control-only */
6. /* Or detect AUX-area mmap by offset */
7. if vma.vm_pgoff != 0:
   - return perf_aux_mmap(event, vma).
8. /* Allocate ring buffer */
9. user_locked = atomic_long_read(&user.locked_vm) + nr_pages.
10. if user_locked > user_lock_limit: return -EPERM.
11. mutex_lock(&event.mmap_mutex).
12. if event.rb ∧ event.rb.nr_pages != nr_pages - 1: return -EINVAL.
13. if !event.rb:
    - event.rb = rb_alloc(nr_pages - 1, watermark, event.cpu, flags). — cross-ref `kernel/perf/ring-buffer.md`.
    - ring_buffer_attach(event, event.rb).
14. vma.vm_ops = &perf_mmap_vmops. (open/close for mmap refcount tracking)
15. atomic_inc(&event.rb.mmap_count).
16. mutex_unlock(&event.mmap_mutex).
17. perf_event_update_userpage(event).
18. return 0.

`Event::poll(file, wait)` (= perf_poll):
1. event = file->private_data.
2. rb = ring_buffer_get(event).
3. if !rb: return 0.
4. poll_wait(file, &event.waitq, wait).
5. events = atomic_xchg(&rb.poll, 0). — rb sets POLLIN when crossing wakeup_watermark.
6. ring_buffer_put(rb).
7. return events.

`Event::read_fd(file, buf, count, ppos)` (= perf_read):
1. event = file->private_data.
2. /* Determine read format */
3. read_size = perf_event_read_size(event). — depends on read_format.
4. if count < read_size: return -ENOSPC.
5. /* Single event or group */
6. if event.attr.read_format & PERF_FORMAT_GROUP:
   - ret = perf_read_group(event, buf).
7. else:
   - ret = perf_read_one(event, buf).
8. return ret.

`kernel::perf::EventContext::find_get(task, event) -> Result<*EventContext, Err>`:
1. /* Per-cpu vs per-task */
2. if !task:
   - cpuctx = per_cpu_ptr(&perf_cpu_context, event.cpu).
   - ctx = &cpuctx.ctx.
   - get_ctx(ctx).
   - return ctx.
3. /* Per-task: look up or alloc */
4. mutex_lock(&task.perf_event_mutex).
5. ctx = task.perf_event_ctxp.
6. if ctx:
   - get_ctx(ctx).
   - mutex_unlock.
   - return ctx.
7. /* Alloc new task ctx */
8. ctx = alloc_perf_context().
9. if !ctx: return ERR(-ENOMEM).
10. /* Try cmpxchg-install (lockless wrt concurrent finder) */
11. if cmpxchg(&task.perf_event_ctxp, NULL, ctx) ≠ NULL:
    - kfree(ctx). — concurrent installer won; retry.
    - mutex_unlock; goto step 1.
12. mutex_unlock(&task.perf_event_mutex).
13. return ctx.

`kernel::perf::EventContext::install_event(ctx, event, cpu)` (= perf_install_in_context):
1. lockdep_assert_held(&ctx.mutex).
2. event.ctx = ctx.
3. /* If event-context belongs to a task, IPI the task's CPU */
4. if ctx.task:
    - if !task_function_call(ctx.task, __perf_install_in_context, event):
      - /* Task not running: install via direct list_add under ctx lock */
      - perf_event_attach_ctx(ctx, event).
5. else:
    - /* Per-cpu ctx: IPI target cpu */
    - cpu_function_call(cpu, __perf_install_in_context, event).
6. return.

`Event::on_overflow(event, data, regs)` (= perf_event_overflow):
1. ret = __perf_event_overflow(event, data, regs).
2. return ret.

`__perf_event_overflow(event, data, regs) -> i32`:
1. /* Count up overflow */
2. event.pending_kill = POLL_IN.
3. atomic_inc(&event.event_limit).
4. /* If kernel-counter: invoke overflow_handler */
5. if event.overflow_handler:
   - event.overflow_handler(event, data, regs).
6. else:
   - /* Build sample record + emit to ring buffer */
   - perf_event_output(event, data, regs). — cross-ref `kernel/perf/ring-buffer.md`.
7. /* Sigtrap delivery (if attr.sigtrap) */
8. if event.attr.sigtrap:
   - perf_pending_irq(event). — defer SIGTRAP via irq_work.
9. /* Frequency adjustment */
10. if event.attr.freq:
    - perf_adjust_period(event, data.period, ...).
11. /* BPF prog (if any) ran inside perf_event_output before record write */
12. return ret.

### Out of Scope

- Ring buffer implementation (`kernel/events/ring_buffer.c`) — covered by `kernel/perf/ring-buffer.md`.
- Hardware breakpoint API (`kernel/events/hw_breakpoint.c`) — covered by `kernel/perf/hw-breakpoint.md`.
- Uprobes infrastructure (`kernel/events/uprobes.c`) — covered by `kernel/perf/uprobes.md`.
- Tracepoint integration (`include/linux/tracepoint.h`, `kernel/tracepoint.c`) — covered by trace Tier-2.
- BPF prog dispatch detail (`kernel/bpf/`) — covered by BPF Tier-2.
- x86 PMU vendor specifics (`arch/x86/events/core.c`) — covered by `arch/x86/perf-pmu.md`.
- ARM / RISC-V / PowerPC PMU specifics — covered by per-arch Tier-3.
- KVM guest PMU virtualization — covered by KVM Tier-2.
- The helper-by-helper exhaustive treatment of `kernel/events/core.c` — that lives in `kernel/perf/perf-event-core.md`.
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct perf_event` (perf_event.h) | per-event control block | `kernel::perf::Event` |
| `struct perf_event_context` (perf_event.h) | per-cpu / per-task event-list anchor | `kernel::perf::EventContext` |
| `struct perf_cpu_context` (internal.h) | per-CPU side of context | `kernel::perf::CpuContext` |
| `struct perf_event_pmu_context` (perf_event.h) | per-(event-context × pmu) intersection | `kernel::perf::EventPmuContext` |
| `struct perf_cpu_pmu_context` (internal.h) | per-(CPU × pmu) state | `kernel::perf::CpuPmuContext` |
| `struct perf_event_attr` (uapi) | per-event UAPI config struct | `uapi::PerfEventAttr` |
| `struct perf_buffer` (internal.h) | per-event ring buffer object | `kernel::perf::RingBuffer` |
| `struct perf_event_mmap_page` (uapi) | per-event mmap'd control page (page 0) | `uapi::PerfEventMmapPage` |
| `SYSCALL_DEFINE5(perf_event_open, ...)` | syscall #298 | `Subsystem::syscall_perf_event_open` |
| `perf_event_alloc(attr, cpu, task, group_leader, parent_event, overflow_handler, context, cgroup_fd)` | per-event allocator | `Event::alloc` |
| `perf_event_create_kernel_counter(attr, cpu, task, overflow_handler, context)` | per-event kernel-internal create | `Event::create_kernel` |
| `perf_event_release_kernel(event)` | per-event destroy (kernel-side) | `Event::release` |
| `perf_release(inode, file)` (= perf_fops.release) | per-fd close | `Event::fd_release` |
| `_free_event(event)` | per-event teardown helper | `Event::free_inner` |
| `perf_install_in_context(ctx, event, cpu)` | per-event install into context | `EventContext::install_event` |
| `find_get_context(task, event)` | per-(task, event) context lookup-or-alloc | `EventContext::find_get` |
| `perf_event_ctx_lock_nested(event, mutex_class)` | per-event ctx mutex acquire | `Event::ctx_lock_nested` |
| `event_function_call(event, func, data)` | per-event IPI-or-local execute under ctx lock | `Event::event_function_call` |
| `perf_fops` (file_operations) | per-fd vtable: read/poll/mmap/ioctl/release | `kernel::perf::PERF_FOPS` |
| `_perf_ioctl(event, cmd, arg)` | per-ioctl dispatch | `Event::ioctl_inner` |
| `perf_ioctl(file, cmd, arg)` (= perf_fops.unlocked_ioctl) | per-ioctl top-level | `Event::ioctl` |
| `perf_compat_ioctl(file, cmd, arg)` | 32-bit-userspace ioctl shim | `Event::compat_ioctl` |
| `perf_mmap(file, vma)` (= perf_fops.mmap) | per-event ring/AUX mmap | `Event::mmap` |
| `perf_mmap_open(vma)` / `perf_mmap_close(vma)` | mmap refcounting | `Event::mmap_open` / `_close` |
| `perf_poll(file, wait)` | per-event poll/select | `Event::poll` |
| `perf_read(file, buf, count, ppos)` | per-event read-counter | `Event::read_fd` |
| `perf_event_overflow(event, data, regs)` | per-overflow sample-emit | `Event::on_overflow` |
| `perf_event_enable(event)` / `_disable(event)` | per-event enable/disable | `Event::enable` / `_disable` |
| `perf_event_period(event, value)` (= PERF_EVENT_IOC_PERIOD) | per-event change sample period | `Event::set_period` |
| `perf_event_refresh(event, refresh)` (= PERF_EVENT_IOC_REFRESH) | per-event re-arm sampling | `Event::refresh` |
| `perf_event_for_each(event, func)` / `_for_each_child(event, func)` | iterate group / inherited | `Event::for_each*` |
| `perf_event_read(event, group)` / `_read_value(event, ...)` | per-event read counter value | `Event::read_value` |
| `perf_event_update_userpage(event)` | per-event mmap control-page update | `Event::update_userpage` |
| `perf_event_task_sched_in(task)` / `_sched_out(prev, next)` | per-context-switch hooks | `Event::sched_in` / `_sched_out` |
| `perf_event_init_task(child, clone_flags)` / `_exit_task(child)` | per-fork/exit inheritance | `Event::init_task` / `_exit_task` |
| `perf_cgroup_switch(task)` | per-cgroup-switch hook | `kernel::perf::cgroup_switch` |
| `perf_cgroup_connect(fd, event, attr, group_leader, cgrp_out)` | per-cgroup attach | `kernel::perf::cgroup_connect` |
| `perf_mux_hrtimer_handler(hr)` | per-context multiplexing hrtimer | `kernel::perf::mux_hrtimer_handler` |
| `perf_rotate_context(cpc)` | per-CPU-PMU multiplex rotation | `kernel::perf::rotate_context` |
| `perf_pmu_register(pmu, name, type)` / `_unregister(pmu)` | per-PMU registration | `Pmu::register` / `_unregister` |
| `find_pmu_context(pmu, ctx)` | per-(pmu, ctx) lookup | `Pmu::find_context` |
| `is_kernel_event(event)` | predicate: created via `_create_kernel_counter` | `Event::is_kernel` |
| `is_cgroup_event(event)` | predicate: cgroup-attached | `Event::is_cgroup` |
| `is_software_event(event)` | predicate: software pmu | `Event::is_software` |
| `sysctl_perf_event_paranoid` / `_max_sample_rate` / `_cpu_time_max_percent` / `_event_mlock_kb` | per-policy sysctls | `kernel::perf::sysctl::*` |

### compatibility contract

REQ-1: `sys_perf_event_open(attr, pid, cpu, group_fd, flags) -> int`:
- Syscall #298 on x86_64 (ABI byte-identical).
- `attr`: pointer to `struct perf_event_attr` (size-versioned via `attr->size`; current `PERF_ATTR_SIZE_VER8`).
- `pid`: -1 (no task / per-cpu), 0 (current task), >0 (specified task).
- `cpu`: -1 (any cpu / per-task), ≥0 (specified cpu).
- `group_fd`: -1 (standalone) or fd of group leader.
- `flags`: bitmask of `PERF_FLAG_FD_NO_GROUP`, `_FD_OUTPUT`, `_PID_CGROUP`, `_FD_CLOEXEC`.
- Returns: file descriptor of new event, or negative errno.

REQ-2: pid/cpu domain matrix:
- `pid==-1, cpu>=0` → **per-cpu event** (counts everything on `cpu`).
- `pid>0, cpu==-1` → **per-task event** (counts task wherever it runs).
- `pid>0, cpu>=0` → **per-task-on-cpu** (counts task only while on `cpu`).
- `pid==-1, cpu==-1` → -EINVAL.
- `pid==0, cpu==X` → equivalent to `pid==current_pid, cpu==X`.

REQ-3: cgroup attach (`flags & PERF_FLAG_PID_CGROUP`):
- `pid` is reinterpreted as a cgroupfs fd.
- Event is **per-cpu** (`cpu>=0` required), but only counts when the running task belongs to the target cgroup.
- Implemented via `perf_cgroup_connect` + `perf_cgroup_switch` hook on context-switch.

REQ-4: Event grouping (`group_fd >= 0`):
- New event becomes a member of the group whose leader is `group_fd`.
- All members of a group are scheduled all-or-nothing onto PMU.
- Group leader's `read()` with `attr.read_format & PERF_FORMAT_GROUP` returns sibling counts in one call.
- `PERF_FLAG_FD_NO_GROUP`: prevents the new fd from inheriting any group context.

REQ-5: `perf_event_alloc(attr, cpu, task, group_leader, parent_event, overflow_handler, context, cgroup_fd)`:
- Allocates `struct perf_event` from `perf_event_cache` kmem_cache.
- Initializes lists (sibling_list, child_list, owner_entry, event_entry).
- Resolves `event->pmu` from `attr->type` via `perf_init_event` → `idr_find` on pmu type space.
- Calls `event->pmu->event_init(event)` (vendor / software hook).
- Returns `*perf_event` or ERR_PTR.

REQ-6: `perf_event_create_kernel_counter(attr, cpu, task, overflow_handler, context)`:
- Kernel-internal variant — no syscall, no fd, no userspace.
- Used by: NMI watchdog (`watchdog_hardlockup_perf.c`), perf-trace bpf-attach helpers, kprobes/uprobes infrastructure, drivers (intel_rapl, hw_breakpoint).
- Calls `perf_event_alloc` then `find_get_context` then `perf_install_in_context`.
- Sets `event->overflow_handler = overflow_handler` (custom kernel callback in lieu of ring-buffer emit).
- Marked `is_kernel_event(event) == true` via `event->owner == TASK_TOMBSTONE` sentinel.

REQ-7: `perf_event_release_kernel(event)`:
- Tear-down for kernel-counter (paired with `perf_event_create_kernel_counter`).
- Removes from context, frees ring buffer (if any), drops children, drops event.
- Userspace events use `perf_release` (= perf_fops.release) which wraps this.
- Idempotent on `event == NULL` (early-exit).

REQ-8: `perf_install_in_context(ctx, event, cpu)`:
- For per-task ctx: IPI-or-local-call `__perf_install_in_context` under ctx mutex.
- For per-cpu ctx: `cpu_function_call(cpu, __perf_install_in_context, event)`.
- Adds event to `ctx->event_list`, updates `nr_events` / `nr_active`, links into per-PMU subcontext.

REQ-9: Per-event-fd ioctl table (`PERF_EVENT_IOC_*`, all byte-identical):
- `_ENABLE` (start counting).
- `_DISABLE` (stop counting).
- `_REFRESH` (re-arm sampling for `refresh` overflows).
- `_RESET` (zero counter).
- `_PERIOD` (set new sample period; arg = u64 *value).
- `_SET_OUTPUT` (redirect samples to another event's ring buffer; arg = fd of target).
- `_SET_FILTER` (set tracepoint filter string).
- `_ID` (return event id; arg = u64 *id).
- `_SET_BPF` (attach BPF prog; arg = bpf prog fd).
- `_PAUSE_OUTPUT` (pause/resume sample emission; arg = u32).
- `_QUERY_BPF` (enumerate attached BPF progs).
- `_MODIFY_ATTRIBUTES` (online-modify a subset of attr fields).
- `_PAUSE_AUX` (pause AUX-area emission, e.g. Intel PT).

REQ-10: Per-event mmap (`mmap(fd, length, PROT_READ|PROT_WRITE, MAP_SHARED, ...)`):
- Length must be `(1 + 2^n) * PAGE_SIZE` for n ≥ 0 (page 0 = control, rest = ring buffer).
- AUX area: a separate `mmap` at offset `mmap_page->aux_offset` with length `mmap_page->aux_size`.
- Page 0 = `struct perf_event_mmap_page` (userspace control: header.version, data_head, data_tail, data_offset, data_size, aux_head, aux_tail, time_enabled, time_running, capabilities for user-rdpmc).
- Page 1..2^n = lockless single-producer (kernel) single-consumer (userspace) byte-ring.

REQ-11: `perf_event_overflow(event, data, regs)`:
- Called from PMU IRQ (PMI NMI on x86) when counter overflows.
- Builds `struct perf_sample_data` per `attr.sample_type`: PERIOD, IP, TID, TIME, ADDR, READ, CALLCHAIN, ID, CPU, BRANCH_STACK, REGS_INTR, REGS_USER, STACK_USER, WEIGHT, DATA_SRC, TRANSACTION, PHYS_ADDR, AUX, CGROUP, ….
- If `event->overflow_handler != NULL` (kernel-counter): calls handler instead of ring-buffer emit.
- Else writes `PERF_RECORD_SAMPLE` to ring buffer (cross-ref `kernel/perf/ring-buffer.md`).
- If `attr.sigtrap`: queue SIGTRAP to event's owning task.
- If `attr.freq`: adjust sample_period via `perf_adjust_period` toward target rate.

REQ-12: Per-event lifecycle:
- alloc (cache) → init (pmu hook) → install (ctx) → enable (start counting) → [overflow / read / ioctl loop] → disable → uninstall → free.
- Group leader release also releases all siblings.
- Inherited children (`attr.inherit=1`) released on parent release or child task exit, whichever first.

REQ-13: `perf_event_init_task(child, clone_flags)`:
- Called from `copy_process()` per fork.
- For each inheritable event in parent's ctx (`attr.inherit==1` ∧ `_INHERIT_STAT` flags): clone event for child via `perf_event_alloc(parent_event=parent_evt)`.
- Cloned event has `event->parent = parent_evt`; readonly snapshot until parent reads.
- Installed into child's new per-task ctx via `perf_install_in_context`.

REQ-14: `perf_event_exit_task(child)`:
- Called from `do_exit()`.
- Walks child's ctx event_list; for each child event with parent, accumulate count back into parent (`perf_event_release_kernel` chain).
- Frees child events.

REQ-15: Multiplexing:
- When `nr_active > num_counters` for a PMU, per-CPU-PMU `cpc->mux_hrtimer` rotates groups.
- `perf_mux_hrtimer_handler` calls `perf_rotate_context` to swap event groups in/out.
- Period: per-`cpc->mux_interval_ns` (default ~1ms, sysctl-adjustable via `perf_event_mux_interval_ms`).
- Per-event `time_enabled` tracks wallclock since enable; `time_running` tracks on-PMU time; userspace scales count by `time_enabled/time_running`.

REQ-16: Sysctl knobs (byte-identical):
- `/proc/sys/kernel/perf_event_paranoid`: -1 / 0 / 1 / 2 / 3 — unprivileged access gating.
- `/proc/sys/kernel/perf_event_max_sample_rate`: cap on PMI rate.
- `/proc/sys/kernel/perf_cpu_time_max_percent`: cap on perf cpu-time.
- `/proc/sys/kernel/perf_event_mlock_kb`: per-user mmap quota.
- `/proc/sys/kernel/perf_event_max_stack`: per-sample callchain depth.
- `/proc/sys/kernel/perf_event_max_contexts_per_stack`: per-callchain unwinder budget.

REQ-17: `perf_fops` vtable:
- `.llseek = no_llseek`.
- `.release = perf_release`.
- `.read = perf_read`.
- `.poll = perf_poll`.
- `.unlocked_ioctl = perf_ioctl`.
- `.compat_ioctl = perf_compat_ioctl`.
- `.mmap = perf_mmap`.
- `.fasync = perf_fasync`.

REQ-18: `event_function_call(event, func, data)`:
- Cross-CPU primitive to execute `func(event, ctx, data)` under ctx mutex on the cpu where the event is currently scheduled.
- Used for ENABLE/DISABLE/PERIOD/SET_OUTPUT/SET_BPF on running events.
- Bounces via IPI (`task_function_call` for per-task ctx, `cpu_function_call` for per-cpu ctx).
- If the event is currently inactive (off-PMU), falls back to direct call under mutex.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `event_refcount_balanced` | INVARIANT | per-event alloc/release: refcount strict get/put. |
| `ctx_install_under_mutex` | INVARIANT | per-install_in_context: ctx.mutex held. |
| `event_function_call_under_lock` | INVARIANT | per-event_function_call: target runs with ctx.mutex held. |
| `perf_fops_release_idempotent` | INVARIANT | per-perf_release: calling on partially-freed event safe (NULL checks). |
| `mmap_pages_pow2_plus_one` | INVARIANT | per-perf_mmap: nr_pages ∈ {1+2^n : n ≥ 0}. |
| `kernel_event_owner_sentinel` | INVARIANT | per-kernel-counter: event.owner == TASK_TOMBSTONE. |
| `cgroup_event_requires_cpu` | INVARIANT | per-cgroup attach: cpu ≥ 0 always. |
| `inherit_propagates_only_inheritable` | INVARIANT | per-perf_event_init_task: only `attr.inherit==1` events cloned. |
| `overflow_handler_xor_rb` | INVARIANT | per-event: (overflow_handler != NULL) XOR (rb != NULL). |
| `paranoid_gates_unprivileged` | INVARIANT | per-perf_event_open: paranoid=2 ∧ !CAP_PERFMON ⟹ reject kernel-tracing events. |

### Layer 2: TLA+

`kernel/perf-events.tla`:
- Per-event lifecycle: ALLOCATED → INSTALLED → (ENABLED ↔ DISABLED) → REMOVED → FREED.
- Per-overflow: ARMED → OVERFLOW → (SAMPLE_EMITTED | HANDLER_CALLED) → REARMED.
- Per-context: IDLE → INSTALLING → ACTIVE → REMOVING.
- Properties:
  - `safety_event_freed_only_after_remove` — never FREED while in any ctx.event_list.
  - `safety_no_overflow_after_release` — overflow_handler / ring buffer not invoked post-release.
  - `safety_per_task_ctx_per_task` — at most one perf_event_context per task per pmu.
  - `safety_install_observed` — after perf_install_in_context returns, target ctx contains event.
  - `liveness_per_open_eventually_returns_fd_or_err` — sys_perf_event_open terminates.
  - `liveness_per_overflow_eventually_emits_sample` — overflow eventually drains to userspace via ring buffer (or kernel handler).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Subsystem::syscall_perf_event_open` post: ret ≥ 0 ⟹ event installed in some ctx | `Subsystem::syscall_perf_event_open` |
| `Event::alloc` post: event.refcount = 1; event.pmu != NULL | `Event::alloc` |
| `Event::create_kernel` post: event.owner = TASK_TOMBSTONE; ctx contains event | `Event::create_kernel` |
| `Event::release` post: event detached from ctx; rb freed; bpf_prog detached | `Event::release` |
| `EventContext::install_event` post: event.ctx = ctx; event in ctx.event_list | `EventContext::install_event` |
| `EventContext::find_get` post: ret valid; ctx.refcount incremented | `EventContext::find_get` |
| `Event::on_overflow` post: exactly one of {overflow_handler called, sample emitted} | `Event::on_overflow` |
| `Event::mmap` post: event.rb installed; mmap_count incremented; userpage updated | `Event::mmap` |
| `Event::ioctl_inner` post: cmd dispatched per REQ-9; -ENOTTY for unknown | `Event::ioctl_inner` |
| `perf_event_init_task` post: each inheritable parent event has matching child | `perf_event_init_task` |

### Layer 4: Verus/Creusot functional

`Per-event lifecycle: sys_perf_event_open → perf_event_alloc → find_get_context → perf_install_in_context → ioctl/mmap/read loop → perf_release → perf_event_release_kernel → kmem_cache_free` semantic equivalence: per-Documentation/admin-guide/perf-security.rst, per-Documentation/userspace-api/perf_ring_buffer.rst, per-tools/perf/Documentation/perf-record.txt.

`Per-overflow: PMU IRQ → perf_event_overflow → (overflow_handler kernel | perf_event_output → ring buffer) → poll/read wake` semantic equivalence: per-Documentation/perf/intel-hybrid.rst + Linux kernel perf ABI definition.

### hardening

(Inherits row-1 features from `kernel/perf/00-overview.md` § Hardening.)

perf-event-subsystem reinforcement:

- **Per-event refcount strict get/put** — defense against per-event UAF on concurrent close + ioctl.
- **Per-ctx mutex around install / remove / ioctl** — defense against per-context list race.
- **Per-event_function_call IPI under mutex** — defense against per-CPU-migrate during mutation.
- **Per-perf_event_paranoid gating (with CAP_PERFMON)** — defense against per-unprivileged kernel info leak.
- **Per-perf_event_mlock_kb per-user mmap quota** — defense against per-user mlock-DOS.
- **Per-perf_event_max_sample_rate cap** — defense against per-PMI livelock.
- **Per-perf_cpu_time_max_percent cap** — defense against per-perf-cpu-runaway.
- **Per-cgroup attach requires CAP_PERFMON** — defense against per-cross-cgroup snoop.
- **Per-kernel-counter owner=TASK_TOMBSTONE** — defense against per-userspace-mistaken close of kernel event.
- **Per-overflow_handler XOR ring buffer** — defense against per-double-emit silent corruption.
- **Per-inherit only with attr.inherit=1** — defense against per-uncontrolled event propagation.
- **Per-mmap nr_pages must be 1+2^n** — defense against per-malformed-ring-size confusion.
- **Per-perf_event_release_kernel idempotent + walks children** — defense against per-orphan-child leak.
- **Per-security_perf_event_open / _alloc LSM hooks** — defense against per-policy bypass.
- **Per-set_output cycle detection** — defense against per-redirect-loop (event A → B → A).
- **Per-sigtrap deferred via irq_work** — defense against per-NMI-context signal-delivery deadlock.

