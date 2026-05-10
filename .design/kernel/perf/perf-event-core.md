# Tier-3: kernel/events/core.c — perf_event_open(2) syscall + per-cpu/per-task contexts + scheduling onto PMUs + group/multiplex

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/perf/00-overview.md
upstream-paths:
  - kernel/events/core.c
  - kernel/events/internal.h
  - include/linux/perf_event.h
  - include/uapi/linux/perf_event.h
-->

## Summary

The brain of the perf subsystem — `perf_event_open(2)` syscall implementation + the entire `struct perf_event` lifecycle. Every perf-stat counter, every perf-record sample, every BPF tracing prog attached to a tracepoint, every NMI-watchdog hardware-cycles event, every cgroup-attached perf event, every sigtrap-on-sample userspace allocator, every Intel-PT/BTS AUX-area sample lives here. ~15,000 lines — the largest single file in `kernel/events/`.

This Tier-3 covers `kernel/events/core.c` for: the syscall + struct perf_event allocation/free + per-cpu and per-task contexts + scheduling events onto PMU hardware (counters/fixed-PMC slots/breakpoints) + grouping (group leaders + members all-or-nothing scheduled) + multiplexing (round-robin when more events than counter slots) + inheritance (per-task events propagate to forked children) + per-event-fd file_operations (read/poll/ioctl) + sample emission to ring-buffer (cross-ref `kernel/perf/ring-buffer.md`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct perf_event` | per-event control block | `kernel::perf::Event` |
| `struct perf_event_context` (`perf_event_context`) | per-cpu OR per-task event list | `kernel::perf::EventContext` |
| `struct perf_event_attr` | UAPI per-event configuration struct | `uapi::PerfEventAttr` |
| `struct pmu` | per-PMU vtable + state | `kernel::perf::Pmu` (trait) |
| `sys_perf_event_open(attr, pid, cpu, group_fd, flags)` | syscall entry | `Subsystem::syscall_perf_event_open` |
| `perf_event_alloc(attr, cpu, task, group_leader, ...)` | allocate Event | `Event::alloc` |
| `perf_event_release_kernel(event)` / `_free_event(event)` | release Event | `Event::release` (Drop) |
| `perf_install_in_context(ctx, event, cpu)` | install event into per-cpu / per-task ctx | `EventContext::install_event` |
| `perf_event_enable(event)` / `_disable(event)` | enable/disable event (kernel-side; exposed via PERF_EVENT_IOC_ENABLE/_DISABLE) | `Event::enable` / `_disable` |
| `perf_event_read(event, group)` | read event count | `Event::read` |
| `perf_event_read_value(event, &enabled, &running)` | read with time_enabled/_running | `Event::read_value` |
| `perf_event_period(event, value)` | set sample period (PERF_EVENT_IOC_PERIOD) | `Event::set_period` |
| `perf_event_update_userpage(event)` | update mmap'd user page | `Event::update_userpage` |
| `perf_event_mmap(vma)` | per-event mmap (data + AUX rings) | `Event::mmap` |
| `perf_pmu_disable(pmu)` / `_enable(pmu)` | global disable/enable around context-switch | `Pmu::disable` / `_enable` |
| `perf_event_task_sched_out(prev, next)` / `_in(task)` | sched hooks for context-switch | `Event::sched_out` / `_in` |
| `perf_event_init_task(child, parent_clone_flags)` / `_exit_task(child)` | per-fork/exit inheritance | `Event::init_task` / `_exit_task` |
| `perf_pmu_register(pmu, name, type)` | register a PMU | `Pmu::register` |
| `perf_pmu_unregister(pmu)` | unregister PMU | `Pmu::unregister` |
| `perf_event_group_sched_in(group_event, ctx)` / `_out` | per-group scheduling | `Event::group_sched_in` / `_out` |
| `event_sched_in(event)` / `event_sched_out(event)` | per-event PMU schedule | `Event::sched_pmu_in` / `_out` |
| `perf_event_for_each(event, func)` / `_for_each_child(event, func)` | iterate group / inherited children | `Event::for_each*` |
| `perf_event_overflow(event, data, regs)` | per-overflow-IRQ sample emission | `Event::on_overflow` |
| `perf_event_create_kernel_counter(attr, cpu, task, callback, ctx)` | kernel-internal create (e.g., NMI watchdog uses) | `Event::create_kernel` |
| `bpf_perf_event_set_attr(perf_event, prog)` (PERF_EVENT_IOC_SET_BPF) | attach BPF prog (cross-ref `kernel/bpf/bpf-core.md`) | `Event::set_bpf_prog` |

## Compatibility contract

REQ-1: `sys_perf_event_open` syscall #298 ABI byte-identical: `attr` (struct perf_event_attr) layout + `pid` (-1 / 0 / >0) + `cpu` (-1 / >=0) + `group_fd` + `flags` (FD_NO_GROUP / FD_OUTPUT / PID_CGROUP / FD_CLOEXEC).

REQ-2: `struct perf_event_attr` size-versioned via `attr->size`; current = `PERF_ATTR_SIZE_VER8`. Forward-compat: smaller `size` extended with zeros, larger size accepted only if extra bytes are zero.

REQ-3: Per-event types: `PERF_TYPE_HARDWARE` / `_SOFTWARE` / `_TRACEPOINT` / `_HW_CACHE` / `_RAW` / `_BREAKPOINT` / dynamic PMU type IDs registered via `/sys/bus/event_source/devices/<pmu>/type`; per-type config interpretation byte-identical.

REQ-4: Per-event-fd ioctls byte-identical: `PERF_EVENT_IOC_ENABLE`, `_DISABLE`, `_REFRESH`, `_RESET`, `_PERIOD`, `_SET_OUTPUT`, `_SET_FILTER`, `_ID`, `_SET_BPF`, `_PAUSE_OUTPUT`, `_QUERY_BPF`, `_MODIFY_ATTRIBUTES`, `_PAUSE_AUX`.

REQ-5: Per-cpu vs per-task context: per-cpu (`pid==-1, cpu>=0`) installs in per-cpu context; per-task (`pid>0, cpu==-1`) installs in per-task context; per-task-per-cpu (`pid>0, cpu>=0`) installs in per-task context but only counts on specified cpu.

REQ-6: Grouping: events with `group_fd == leader_fd` form a group; group all-or-nothing scheduled (either entire group on, or none). Group leader read returns sibling values too if `attr.read_format & PERF_FORMAT_GROUP`.

REQ-7: Multiplexing: when more events than counter slots, time-slice via per-context multiplex hrtimer; per-event `time_enabled` / `time_running` track wallclock vs on-CPU time.

REQ-8: Inheritance: `attr.inherit=1` events propagate to forked children; per-child copy installed at `perf_event_init_task` from `copy_process`.

REQ-9: Sample emission: per-overflow IRQ → `perf_event_overflow` → builds sample record + writes to ring buffer (cross-ref `kernel/perf/ring-buffer.md`). Optionally fires SIGTRAP if `attr.sigtrap=1`.

REQ-10: BPF prog attach via `PERF_EVENT_IOC_SET_BPF`: per-event `bpf_prog` ptr installed; on overflow, BPF prog runs before sample emission (filter / aggregate via map).

REQ-11: Per-event mmap (`PROT_READ | PROT_WRITE, MAP_SHARED`): page 0 = user-page (control header), pages 1..2^n = ring buffer; AUX area separate via second mmap (cross-ref `ring-buffer.md`).

REQ-12: Sysctl knobs: `/proc/sys/kernel/perf_event_paranoid` (-1/0/1/2/3 controlling non-root access), `_max_sample_rate`, `_cpu_time_max_percent`, `_event_mlock_kb`, `_event_max_stack`, `_event_max_contexts_per_stack` byte-identical.

REQ-13: cgroup-attached perf events (`flags & PERF_FLAG_PID_CGROUP`): event scoped to per-cgroup; counted only when current task in cgroup.

## Acceptance Criteria

- [ ] AC-1: `perf list` shows the same PMU events + descriptors as upstream baseline.
- [ ] AC-2: `perf stat -e cycles,instructions /bin/ls` produces results matching upstream within 1% noise.
- [ ] AC-3: `perf record -F 99 -ag sleep 5` captures sample-records correctly.
- [ ] AC-4: Group test: `perf stat -e '{cycles,instructions,branches}'` reads all 3 from group fd as a single read.
- [ ] AC-5: Inheritance test: `perf stat --pid <pid> -e cycles -- sleep 5` counts cycles in fork'd children too.
- [ ] AC-6: Multiplex test: 8 hardware events on 4-counter PMU; `time_enabled` and `time_running` reported correctly per-event.
- [ ] AC-7: BPF tracing test: `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm]=count() }'` works.
- [ ] AC-8: cgroup test: per-cgroup perf event with `PERF_FLAG_PID_CGROUP` only counts when target cgroup tasks are on-cpu.
- [ ] AC-9: sigtrap-on-sample: GWP-ASan-style test program receives SIGTRAP at correct sample IP.
- [ ] AC-10: kselftest `tools/testing/selftests/perf_events/` passes.

## Architecture

`Event` lives in `kernel::perf::Event`:

```
struct Event {
  refcount: Refcount,
  attr: UapiPerfEventAttr,
  pmu: &'static dyn Pmu,
  pmu_priv: KBox<dyn Any>,
  ctx: Arc<EventContext>,
  cpu: i32,
  state: AtomicI32,           // PERF_EVENT_STATE_DEAD / _ERROR / _OFF / _INACTIVE / _ACTIVE
  group_leader: Arc<Event>,    // points at self if leader
  group_caps: u32,
  group_entry: ListEntry,      // links into ctx.event_list
  sibling_list: Mutex<Vec<Arc<Event>>>,
  hw: HwPerfEvent,             // per-event PMU register state (per-counter idx, sample period, last value)
  count: AtomicU64,             // accumulated count (interpreted per-PMU)
  child_count: AtomicU64,       // sum of inherited-child counts
  oncpu: AtomicI32,             // current pmu CPU or -1
  parent: Option<Arc<Event>>,    // for inherited
  child_list: Mutex<Vec<Arc<Event>>>,
  total_time_enabled: AtomicU64,
  total_time_running: AtomicU64,
  rb: AtomicPtr<RingBuffer>,    // mmap'd ring (cross-ref ring-buffer.md)
  user_page: AtomicPtr<PerfEventMmapPage>,
  bpf_prog: AtomicPtr<BpfProg>,  // attached BPF prog
  cgroup: Option<Arc<PerfCgroup>>,
  task_event_handler: AtomicPtr<EventHandler>,
  sigtrap_perf_data: u64,
  ns_namespace_handle: Option<Arc<Namespace>>,
}
```

`EventContext` lives in `kernel::perf::EventContext`:

```
struct EventContext {
  refcount: Refcount,
  pmu: Option<&'static dyn Pmu>,
  type_: ContextType,           // per-cpu / per-task
  task: Option<Arc<Task>>,
  active: Mutex<()>,
  pinned_groups: Mutex<Vec<Arc<Event>>>,
  flexible_groups: Mutex<Vec<Arc<Event>>>,
  event_list: Mutex<Vec<Arc<Event>>>,
  nr_events: AtomicU32,
  nr_active: AtomicU32,
  nr_pinned: AtomicU32,
  nr_freq: AtomicU32,
  is_active: AtomicU32,
  rotate_disable: AtomicI32,
  nr_inherited: AtomicU32,
  generation: AtomicU64,
  pinned_active: Mutex<Vec<Arc<Event>>>,
  flexible_active: Mutex<Vec<Arc<Event>>>,
}
```

`sys_perf_event_open` flow:
1. Copy `attr` from userspace via `perf_copy_attr` (validates `attr->size` versioning + zero-extension).
2. Resolve PMU by `attr.type`:
   - `PERF_TYPE_HARDWARE` / `_HW_CACHE` → x86_pmu (cross-ref `kernel/perf/x86-pmu-*.md`).
   - `PERF_TYPE_SOFTWARE` → software events PMU.
   - `PERF_TYPE_TRACEPOINT` → tracepoint PMU (cross-ref `kernel/trace/00-overview.md`).
   - `PERF_TYPE_RAW` → vendor-encoded HW event.
   - `PERF_TYPE_BREAKPOINT` → hw_breakpoint PMU (cross-ref `kernel/perf/hw-breakpoint.md`).
   - dynamic type → look up in `/sys/bus/event_source/devices/<pmu>/type`.
3. Resolve target context: per-cpu (cpu>=0) or per-task (lookup task by pid + acquire context).
4. Resolve group leader (group_fd != -1 → fdget + verify same context, same PMU, etc.).
5. `Event::alloc(&attr, cpu, task, group_leader, ...)`:
   - Allocate Event struct + per-PMU priv via `pmu->event_init(event)`.
6. Allocate event-fd via `anon_inode_getfd("[perf_event]", &perf_fops, event, ...)`.
7. `EventContext::install_event(ctx, event, cpu)`:
   - Acquire `ctx.active` mutex.
   - Add event to `pinned_groups` or `flexible_groups`.
   - If `state == OFF`, leave inactive.
   - Else: `Event::sched_pmu_in(&event)` → `pmu->add(event, flags)` to schedule onto a counter slot if available; otherwise multiplex.
8. Return event_fd.

Per-context-switch hook (`Event::sched_out` from `__schedule`):
1. `Pmu::disable(pmu)` (PMU-global disable to avoid mid-switch sampling).
2. For prev task ctx: walk pinned_active + flexible_active, `pmu->del(event, ...)` to free counter slots.
3. For next task ctx: walk pinned_groups + flexible_groups, `pmu->add(event, ...)` to install on counters.
4. `Pmu::enable(pmu)`.

Multiplex hrtimer per-context: when nr_events > nr_counters, hrtimer fires at multiplex_interval (default 1ms × HZ adjustments); rotates flexible_groups membership in/out of counters.

Per-overflow IRQ → `perf_event_overflow(event, data, regs)`:
1. Increment `event.count` from PMU register.
2. If `attr.sample_period` > 0 AND `count >= sample_period`:
   - Build sample record per `attr.sample_type` bits (IP, TID, TIME, ADDR, READ, CALLCHAIN, ID, CPU, PERIOD, CALLCHAIN, RAW, BRANCH_STACK, REGS_USER, STACK_USER, WEIGHT, DATA_SRC, IDENTIFIER, TRANSACTION, REGS_INTR, PHYS_ADDR, AUX, CGROUP, DATA_PAGE_SIZE, CODE_PAGE_SIZE).
   - If `event.bpf_prog`: call `Event::run_bpf_prog(&event, ctx, regs)` to filter/transform.
   - Reserve ring-buffer space (cross-ref `ring-buffer.md`); copy sample bytes; commit.
   - If `attr.sigtrap`: `do_send_sig_info(SIGTRAP, ..., task)`.
3. Reset PMU counter for next overflow.

Per-event mmap: `Event::mmap(vma)` — page 0 mapped as `perf_event_mmap_page` (control header w/ data_head, data_tail, time_enabled, time_running, cap_*); pages 1..2^n mapped as ring buffer (`event.rb`).

Inheritance: `Event::init_task(child, clone_flags)` from `copy_process` — for each parent inheritable event, allocate child copy + install in child task context.

cgroup-attached: events with `PERF_FLAG_PID_CGROUP` use `EventContext::cgroup_ctx`; per-context-switch checks current task's cgroup membership before scheduling-in.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `event_no_uaf` | UAF | `Arc<Event>` outlives any in-flight sample emission; close-during-overflow safe (RCU-defer free). |
| `ctx_no_uaf` | UAF | `Arc<EventContext>` outlives all events in it; install_event holds ctx ref. |
| `attr_size_no_oob` | OOB | `perf_copy_attr` validates `attr->size` against PERF_ATTR_SIZE_VERN; zero-extends short; rejects oversized non-zero tail. |
| `pmu_idx_no_oob` | OOB | per-pmu counter-idx allocator bounded by `pmu->num_counters`; never indexes past hw counter array. |
| `bpf_prog_no_uaf` | UAF | per-event bpf_prog ptr installed via cmpxchg; replace + RCU-grace before old prog freed. |

### Layer 2: TLA+

`models/perf/event_scheduler.tla` (parent-declared): proves per-cpu PMU event-scheduling under counter-pressure: pinned events always run when on, exclusive events block others, group events all-or-nothing, inheritance into child task preserves attr.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Event::install_in_context` post: event in ctx.event_list AND state ∈ {OFF, INACTIVE, ACTIVE}; never DEAD/ERROR | `EventContext::install_event` |
| Per-group all-or-nothing: when `group.state == ACTIVE`, all siblings have `state == ACTIVE`; when `OFF/INACTIVE`, siblings match | group sched |
| `total_time_running <= total_time_enabled` always (running is a subset of enabled time) | `Event::update_event_times` |
| `attr.read_format & PERF_FORMAT_GROUP` read returns sibling array sized exactly `1 + nr_siblings` | `Event::read` |

### Layer 4: Verus/Creusot functional

`Event::sched_in(prev, next)` ↔ `Event::sched_out` round-trip equivalence: per-task event accumulates counts only during on-cpu intervals; sum of count deltas across all on-cpu intervals equals total observed count. Encoded as Verus invariant on event time-tracking.

## Hardening

(Inherits row-1 features from `kernel/perf/00-overview.md` § Hardening.)

perf-event-core specific reinforcement:

- **`perf_event_paranoid` default = 2** in Rookery (matches upstream restrictive default since 5.8 hardening series). Non-root cannot create kernel-level samples.
- **Per-event ringbuf size cap** via `perf_event_mlock_kb` sysctl + per-uid cap; defense against perf-record memory exhaustion.
- **Per-CPU max sample rate** via `perf_cpu_time_max_percent` sysctl (default 25%); throttle if sample-emission consumes too much CPU.
- **Callchain depth cap** via `perf_event_max_stack` sysctl (default 127); defense against stack-walk infinite-loop on corrupted unwinder data.
- **BPF prog attach via `PERF_EVENT_IOC_SET_BPF` requires CAP_BPF + CAP_PERFMON** — defense against unprivileged perf+bpf escalation.
- **Per-event refcount saturating** — overflow saturates at u32::MAX.
- **Per-fd close revokes pending samples** — close + RCU-grace before freeing event ensures in-flight sample emission completes safely.
- **sigtrap on perf_event_paranoid >= 2** requires CAP_SYS_ADMIN — defense against unprivileged process forging SIGTRAP via perf.
- **`PERF_TYPE_BREAKPOINT` requires CAP_SYS_PTRACE** — hw-breakpoint resources are scarce + privilege-leaking.
- **Per-task event inheritance bounded** — total inherited events per task capped at `inherit_stat ? 16384 : 8192`; defense against fork-bomb-amplified perf-event-flood.
- **Per-event mmap CAP_SYS_ADMIN-mediated** for kernel-only events.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Ring-buffer impl (covered in `kernel/perf/ring-buffer.md` future Tier-3)
- Per-PMU implementations (covered in `kernel/perf/x86-pmu-*.md` future Tier-3s)
- HW breakpoint allocator (covered in `kernel/perf/hw-breakpoint.md` future Tier-3)
- uprobes (covered in `kernel/perf/uprobes.md` future Tier-3)
- Tracepoint integration (covered in `kernel/trace/bpf-trace.md` future Tier-3)
- 32-bit-only paths
- Implementation code
