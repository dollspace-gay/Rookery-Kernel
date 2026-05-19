# Tier-3: kernel/trace/trace.c — Trace core (instances + ringbuf-glue + event-commit + current_tracer plug-in)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/trace/00-overview.md
upstream-paths:
  - kernel/trace/trace.c (~10017 lines)
  - kernel/trace/trace.h
  - kernel/trace/trace_output.c
  - kernel/trace/trace_seq.c
  - include/linux/trace_events.h
  - include/linux/trace.h
  - include/linux/tracefs.h
-->

## Summary

`kernel/trace/trace.c` is the **trace core**: the glue between the lockless ring buffer (`ring_buffer.c`), the tracefs filesystem hierarchy at `/sys/kernel/tracing/`, the per-event class registration machinery (`trace_events.c`), the swappable `current_tracer` plug-ins (function, function_graph, irqsoff, wakeup, osnoise, hwlat, nop, ...), and the user-space-visible read/write/splice path. Per-instance state lives in `struct trace_array` (one `&global_trace` plus N `instances/<name>/`); each `trace_array` owns one `struct array_buffer` whose `.buffer` field is a per-instance `struct trace_buffer` (lockless ring) plus a per-cpu `struct trace_array_cpu` data pointer. Per-CPU producer path: tracer → `trace_event_buffer_lock_reserve()` → `__trace_buffer_lock_reserve()` → `ring_buffer_lock_reserve()` → fill payload → `trace_event_buffer_commit()` → `__buffer_unlock_commit()` → optional `output_printk` + `ftrace_exports` + `__trace_stack` + user-stack. Per-`tracing_on/off` flips `ring_buffer_record_on/off` AND mirrors a `tr->buffer_disabled` shadow flag for tracers in the fast path that haven't allocated buffers yet. Per-`current_tracer` swap: `tracing_set_tracer(tr, name)` looks up `struct tracer` in `trace_types` list, calls `current_trace->reset(tr)`, briefly retargets to `nop_trace` under RCU, optionally allocates/frees the snapshot buffer, calls `trace->init(tr)`, installs. Per-snapshot (CONFIG_TRACER_SNAPSHOT): a second `array_buffer` (`snapshot_buffer`) swappable atomically with the live one. Per-event payload: `struct ring_buffer_event { type_len, time_delta, array[] }` carrying a `struct trace_entry` header + per-event class fields rendered from `events/<grp>/<evt>/format`. Per-`tracing_thresh`: a nsec threshold consumed by latency tracers (irqsoff, wakeup, hwlat) to drop sub-threshold traces. Per-`trace_options` (`/sys/kernel/tracing/trace_options`, `options/<name>`): bitmask `tr->trace_flags` controlling stack-dump, raw-timestamp, overwrite, pid-record, tgid-record, irq-info, latency-fmt, ... gated through `set_tracer_flag()` so the active tracer can veto changes.

This Tier-3 covers `kernel/trace/trace.c` (~10017 lines) — the *core*, not the function-tracer (covered by `ftrace.md`) and not the ring-buffer-internals (covered by `ring-buffer.md`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct trace_array` | per-instance trace state | `TraceArray` |
| `struct array_buffer` | per-instance ring + percpu data pair | `ArrayBuffer` |
| `struct trace_array_cpu` | per-cpu per-instance counters | `TraceArrayCpu` |
| `struct tracer` | swappable tracer plug-in vtable | `Tracer` (trait obj) |
| `struct tracers` | per-instance tracer-list node | `TracerNode` |
| `global_trace` | the top-level instance | `Trace::global` |
| `ftrace_trace_arrays` | list of all instances | `Trace::arrays` |
| `trace_types` | list of registered tracer plug-ins | `Trace::types` |
| `nop_trace` | the no-op tracer (default current_trace) | `Trace::nop` |
| `register_tracer()` | per-plug-in registration | `Trace::register_tracer` |
| `tracing_set_tracer()` | per-instance current_tracer swap | `TraceArray::set_tracer` |
| `tracer_init()` | per-tracer init dispatch | `Tracer::init` |
| `tracing_on()` / `tracing_off()` | global enable/disable | `Trace::on` / `off` |
| `tracer_tracing_on/off/disable/enable` | per-instance | `TraceArray::on`/`off`/`disable_record`/`enable_record` |
| `tracing_is_on()` | per-state-query | `Trace::is_on` |
| `tracing_snapshot()` | per-snapshot trigger | `Trace::snapshot` |
| `tracing_alloc_snapshot()` | per-snapshot alloc | `Trace::alloc_snapshot` |
| `tracing_snapshot_alloc()` | per-snapshot-alloc-then-take | `Trace::snapshot_alloc` |
| `tracing_snapshot_instance()` | per-instance swap live↔snapshot | `TraceArray::snapshot` |
| `trace_event_buffer_lock_reserve()` | per-event reserve (with filter-buffer fast path) | `TraceEventBuffer::lock_reserve` |
| `trace_event_buffer_commit()` | per-event commit (trigger-discard + export + stack) | `TraceEventBuffer::commit` |
| `__trace_buffer_lock_reserve()` | per-event lower-half reserve | `TraceArray::buffer_lock_reserve` |
| `trace_buffer_unlock_commit_regs()` | per-event commit + stack | `TraceArray::commit_regs` |
| `trace_buffer_unlock_commit_nostack()` | per-event commit-no-stack | `TraceArray::commit_nostack` |
| `__buffer_unlock_commit()` | per-event commit lower | `TraceArray::commit` |
| `trace_function()` | per-function-tracer entry record | `TraceArray::function` |
| `__ftrace_trace_stack()` | per-kstack-record | `TraceArray::trace_stack` |
| `ftrace_trace_userstack()` | per-ustack-record | `TraceArray::trace_userstack` |
| `trace_buffered_event_enable/disable` | per-cpu filter scratch buffers | `Trace::buffered_event_{enable,disable}` |
| `tracing_gen_ctx_irq_test()` | per-trace_ctx encode (preempt/irq/migration) | `Trace::gen_ctx` |
| `trace_handle_return()` | per-output return-code mapper | `TraceSeq::handle_return` |
| `trace_event_setup()` | per-event header fill (ts, ctx) | `TraceEvent::setup` |
| `trace_array_get()` / `trace_array_put()` | per-instance refcount | `TraceArray::get/put` |
| `trace_array_get_by_name()` | per-instance lookup-or-create | `Trace::array_get_by_name` |
| `trace_array_find()` | per-instance lookup | `Trace::array_find` |
| `trace_array_destroy()` | per-instance teardown | `TraceArray::destroy` |
| `trace_array_create()` | per-instance create | `Trace::array_create` |
| `trace_array_create_systems()` | per-instance create with event-filter | `Trace::array_create_systems` |
| `instance_mkdir()` / `instance_rmdir()` | tracefs `instances/<name>` hook | `Trace::instance_mkdir`/`rmdir` |
| `allocate_trace_buffer()` | per-instance ring alloc | `TraceArray::allocate_buffer` |
| `allocate_trace_buffers()` | per-instance ring + snapshot alloc | `TraceArray::allocate_buffers` |
| `init_tracer_tracefs()` | per-instance tracefs node creation | `TraceArray::init_tracefs` |
| `tracing_reset_cpu()` | per-cpu ringbuf clear | `TraceArray::reset_cpu` |
| `tracing_reset_online_cpus()` | per-online-cpus ringbuf clear | `TraceArray::reset_online_cpus` |
| `tracing_resize_ring_buffer()` | per-instance resize | `TraceArray::resize_buffer` |
| `__tracing_resize_ring_buffer()` | per-instance resize lower | `TraceArray::resize_buffer_inner` |
| `tracing_set_clock()` | per-instance clock selector | `TraceArray::set_clock` |
| `set_tracer_flag()` | per-trace_flags option setter | `TraceArray::set_flag` |
| `set_tracer_option()` / `__set_tracer_option()` | per-tracer-specific option | `TraceArray::set_tracer_option` |
| `trace_set_options()` | per-string-parse flag setter | `TraceArray::set_options` |
| `tracing_thresh` (var) | per-latency-tracer ns threshold | `Trace::thresh` |
| `tracing_thresh_read/write` | tracefs knob | `Trace::thresh_read/write` |
| `tracing_set_tracer` | tracefs `current_tracer` write | `TraceArray::set_tracer` |
| `__tracing_open()` | tracefs `trace` (and snapshot) read open | `TraceArray::trace_open` |
| `tracing_read_pipe()` / `tracing_splice_read_pipe()` | tracefs `trace_pipe` blocking read | `TraceArray::pipe_read`/`splice` |
| `tracing_buffers_read()` / `tracing_buffers_splice_read()` | tracefs `per_cpu/cpu<N>/trace_pipe_raw` | `TraceArray::buffers_read`/`splice` |
| `tracing_buffers_mmap()` | tracefs ring zero-copy mmap | `TraceArray::buffers_mmap` |
| `tracing_mark_write()` / `tracing_mark_raw_write()` | tracefs `trace_marker[_raw]` userspace inject | `TraceArray::mark_write`/`mark_raw_write` |
| `tracing_log_err()` | `error_log` writer | `TraceArray::log_err` |
| `trace_total_entries()` / `trace_total_entries_cpu()` | per-instance stats | `TraceArray::total_entries`/`_cpu` |
| `trace_array_printk_buf()` | per-instance ringbuf-printk | `TraceArray::printk_buf` |
| `ftrace_exports` | per-event/-marker external consumer hooks | `Trace::exports` |
| `register_ftrace_export()` / `unregister_ftrace_export()` | export-list ctl | `Trace::register_export`/`unregister_export` |
| `output_printk()` | tp_printk: dump events to printk | `Trace::output_printk` |
| `tracepoint_printk_sysctl` | `kernel.tracepoint_printk` sysctl | `Trace::tracepoint_printk_sysctl` |
| `disable_trace_on_warning()` | per-WARN auto-stop | `Trace::disable_on_warning` |

## Compatibility contract

REQ-1: `struct trace_array` shape (subset visible at this Tier-3 level):
- `name`: per-instance name (`global_trace.name == NULL` ⟹ top-level; everything else ⟹ `instances/<name>/`).
- `dir`: tracefs dentry for the instance.
- `array_buffer`: `struct array_buffer { tr, buffer (ring_buffer), data (per-cpu trace_array_cpu), time_start }`.
- `snapshot_buffer`: optional second `array_buffer` (CONFIG_TRACER_SNAPSHOT).
- `mapped`: per-zero-copy-mmap refcount.
- `current_trace`: pointer to active `struct tracer`.
- `current_trace_flags`: snapshot of `current_trace->flags` (or per-tracer-option overlay).
- `tracers`: per-instance list of `struct tracers` nodes wrapping `struct tracer`.
- `trace_flags`: 64-bit bitmask of `TRACE_ITER_*` flags.
- `trace_flags_index`: index table for per-flag tracefs files.
- `tracing_cpumask`: cpumask of CPUs being traced.
- `pipe_cpumask`: cpumask of CPUs whose `trace_pipe` is currently open.
- `max_lock`: arch_spinlock_t guarding snapshot swap.
- `start_lock`: raw spinlock for `tracing_start/stop` nest counter.
- `buffer_disabled`: shadow of `ring_buffer_record_is_on(...)` for tracers in the no-buffer fast path.
- `ring_buffer_expanded`: bool — has the ring been resized past the default 1KB-startup-stub.
- `ref`: per-instance refcount.
- `function_pids`, `function_no_pids`: `struct trace_pid_list` for `set_ftrace_pid` / `set_ftrace_notrace_pid` filters.
- `cond_snapshot`: optional `struct cond_snapshot` (snapshot trigger).
- `systems`, `events`, `hist_vars`, `err_log`, `marker_list`, `mod_events`: per-instance event/hist/marker/error/module sub-lists.
- `clock_id`: index into `trace_clocks[]` (`local`, `global`, `counter`, `uptime`, `perf`, `mono`, `mono_raw`, `boot`, `tai`).
- `range_addr_start`, `range_addr_size`, `module_delta`: persistent (boot-time / mapped) ring buffer state.
- `func_probes`, `mod_trace`, `mod_notrace`: per-instance function-probe + module-trace lists.

REQ-2: `struct tracer` (plug-in vtable):
- `name`: short identifier (`"function"`, `"function_graph"`, `"irqsoff"`, `"wakeup"`, `"wakeup_rt"`, `"wakeup_dl"`, `"hwlat"`, `"osnoise"`, `"timerlat"`, `"branch"`, `"mmiotrace"`, `"nop"`).
- `init(tr) -> int`: called by `tracing_set_tracer`; on success the tracer owns the buffer's interrupt path until `reset()`.
- `reset(tr)`: called before swap-out.
- `start(tr)` / `stop(tr)`: per-tracing_start/stop hooks.
- `update_thresh(tr)`: invoked when `/tracing_thresh` is written.
- `open(iter)` / `pipe_open(iter)` / `close(iter)` / `pipe_close(iter)`: per-`trace` and `trace_pipe` reader entries.
- `read(iter, filp, ubuf, cnt, ppos)`: legacy splice path.
- `splice_read(iter, ...)`: zero-copy splice.
- `wait_pipe(iter)`: blocking wait-for-data.
- `print_header(seq)` / `print_line(iter)`: per-event renderer.
- `set_flag(tr, mask, set)` / `flag_changed(tr, mask, set)`: per-tracer-specific option override.
- `flags`: per-tracer `struct tracer_flags` default option set.
- `print_max`: tracer renders max-latency record at output (irqsoff/wakeup family).
- `use_max_tr`: tracer needs the snapshot/max buffer.
- `allocated_snapshot`: bool — snapshot buffer alloc done.
- `noboot`: tracer is not allowed via the `ftrace=` cmdline.
- `enabled`: refcount across instances.
- `next`: list link into `trace_types`.

REQ-3: `register_tracer(type)`:
- Validate `type->name` non-NULL and < `MAX_TRACER_SIZE`.
- `if security_locked_down(LOCKDOWN_TRACEFS)` ⟹ `-EPERM`.
- Acquire `trace_types_lock`.
- Reject if a tracer with the same name is in `trace_types`.
- `do_run_tracer_selftest(type)` (when CONFIG_FTRACE_STARTUP_TEST and not deferred).
- For each `tr` in `ftrace_trace_arrays`: `add_tracer(tr, type)` to allocate the per-instance `struct tracers` node + per-tracer-options dir.
- Link `type` at head of `trace_types`.
- Release lock.
- If `default_bootup_tracer == type->name`: call `tracing_set_tracer(&global_trace, type->name)`, run `apply_trace_boot_options()`.

REQ-4: `tracing_set_tracer(tr, buf)`:
- `guard(mutex)(&trace_types_lock)`.
- `update_last_data(tr)` (per-module-reload bookkeeping).
- If `!tr->ring_buffer_expanded`: grow to `trace_buf_size` (default 1MB) across `RING_BUFFER_ALL_CPUS`.
- Walk `tr->tracers`: find `struct tracers* t` whose `t->tracer->name == buf`.
- `-EINVAL` if none.
- Return 0 if `trace == tr->current_trace` (no-op).
- If `tracer_uses_snapshot(trace)`: hold `tr->max_lock` IRQ-disabled, ensure no `cond_snapshot` is armed; else `-EBUSY`.
- If `system_state < SYSTEM_RUNNING && trace->noboot`: warn + `-EINVAL`.
- If `!trace_ok_for_array(trace, tr)` (some tracers only for top-level): `-EINVAL`.
- If `tr->trace_ref` (somebody has `trace_pipe` open): `-EBUSY`.
- `trace_branch_disable()` (branch tracer guard).
- `tr->current_trace->enabled--`.
- `if tr->current_trace->reset`: `tr->current_trace->reset(tr)`.
- Set `tr->current_trace = &nop_trace; tr->current_trace_flags = nop_trace.flags` (RCU transition).
- If old used snapshot AND new doesn't: `synchronize_rcu()` + `free_snapshot(tr)` + `tracing_disarm_snapshot(tr)`.
- If old didn't use snapshot AND new does: `tracing_arm_snapshot_locked(tr)` ⟹ allocate snapshot buffer.
- `tr->current_trace_flags = t->flags ?: trace->flags`.
- If `trace->init`: `tracer_init(trace, tr)`; on failure rewind + return error.
- `tr->current_trace = trace`; `tr->current_trace->enabled++`.
- `trace_branch_enable(tr)`.
- Return 0.

REQ-5: `tracing_on()` / `tracing_off()` (global):
- `tracing_on()` ⟹ `tracer_tracing_on(&global_trace)` ⟹ `ring_buffer_record_on(tr->array_buffer.buffer)` + `tr->buffer_disabled = 0`.
- `tracing_off()` ⟹ symmetric `ring_buffer_record_off(...)` + `tr->buffer_disabled = 1`.
- `tracing_is_on()` ⟹ `tracer_tracing_is_on(&global_trace)` ⟹ `ring_buffer_record_is_set_on(tr->array_buffer.buffer)` (or shadow if not allocated).
- `tracer_tracing_disable()` / `tracer_tracing_enable()` are *counted* (nestable) and route through `ring_buffer_record_disable/enable`.
- All four `EXPORT_SYMBOL_GPL`.

REQ-6: `tracing_snapshot()` (CONFIG_TRACER_SNAPSHOT):
- Per-`tracing_snapshot` ⟹ `tracing_snapshot_instance(&global_trace)`.
- Per-snapshot mechanic: under `tr->max_lock` (arch_spinlock, IRQs disabled), swap `tr->array_buffer.buffer` ↔ `tr->snapshot_buffer.buffer` and the per-cpu `data` percpu pointer.
- Per-`tracing_alloc_snapshot()`: allocate snapshot buffer iff not already allocated (calls into `tracing_alloc_snapshot_instance`).
- Per-`tracing_snapshot_alloc()`: alloc-then-take.
- If !CONFIG_TRACER_SNAPSHOT: stub `WARN_ONCE` and return.
- `EXPORT_SYMBOL_GPL` for `tracing_snapshot`, `tracing_snapshot_alloc`.

REQ-7: `trace_event_buffer_lock_reserve(current_rb, trace_file, type, len, trace_ctx)`:
- `*current_rb = trace_file->tr->array_buffer.buffer`.
- Per-filter-buffer fast path: if `!tr->no_filter_buffering_ref` AND `trace_file->flags & (EVENT_FILE_FL_SOFT_DISABLED | EVENT_FILE_FL_FILTERED)`:
  - `preempt_disable_notrace()`.
  - Try the per-cpu scratch `trace_buffered_event` (page-sized; allocated by `trace_buffered_event_enable()`).
  - Increment `trace_buffered_event_cnt`; if it was 0 (no nested write) AND len ≤ max: setup entry, fill `array[0] = len`, return entry (preempt still disabled). Caller must check filter then either commit (copy to ring) or drop.
  - Else `this_cpu_dec(trace_buffered_event_cnt)` and `preempt_enable_notrace()`, fall through.
- Slow path: `entry = __trace_buffer_lock_reserve(*current_rb, type, len, trace_ctx)`.
- Per-temp_buffer fallback: if `!entry && (trace_file->flags & EVENT_FILE_FL_TRIGGER_COND)`: route through global `temp_buffer` so triggers can still inspect the event.
- Return entry pointer (or NULL).
- `EXPORT_SYMBOL_GPL`.

REQ-8: `trace_event_buffer_commit(fbuffer)`:
- Per-trigger discard: `__event_trigger_test_discard(file, buffer, event, entry, &tt)` returns true ⟹ skip ring-commit, only run post-call triggers.
- Per-`tp_printk` static key: `output_printk(fbuffer)` dumps the event to printk via `trace_event->funcs->trace()`.
- Per-export static key (`trace_event_exports_enabled`): `ftrace_exports(event, TRACE_EXPORT_EVENT)`.
- Per-commit: `trace_buffer_unlock_commit_regs(tr, buffer, event, trace_ctx, regs)` ⟹ `__buffer_unlock_commit` + `ftrace_trace_stack` + `ftrace_trace_userstack`.
- Per-post-trigger: `event_triggers_post_call(file, tt)`.
- `EXPORT_SYMBOL_GPL`.

REQ-9: Per-`tracing_gen_ctx_irq_test(irqs_status)` encodes `trace_ctx` (16-bit packed):
- preempt-count (low bits).
- preempt-lazy.
- need-resched / need-resched-lazy.
- migration-disable (extracted via `migration_disable_value()` when CONFIG_PREEMPT_RT).
- hardirq / softirq / NMI flags from `irqs_status`.
- Used by every record commit so the reader can render the `<idle>-0     [001] d.h2.` prefix identically.

REQ-10: `trace_buffered_event_enable()` / `_disable()`:
- Per-enable: protected by `event_mutex`; if `trace_buffered_event_ref++ == 0`: allocate one node-local page per traced CPU into `per_cpu(trace_buffered_event, cpu)`.
- Per-disable: protected by `event_mutex`; when refcount drops to 0: `on_each_cpu_mask(tracing_buffer_mask, disable_trace_buffered_event)` (increments per-cpu cnt so reservers fall through to slow path), `synchronize_rcu()`, free pages, second `synchronize_rcu()`, then re-enable via `enable_trace_buffered_event` (decrement cnt).
- Two synchronize_rcu() are required so racing reservers can't observe the cleared pointer with a stale cnt.

REQ-11: Per-instance create:
- `trace_array_get_by_name(name, systems)` ⟹ if exists, `__trace_array_get` (refcount), else `trace_array_create_systems(name, systems, 0, 0)`.
- `trace_array_create_systems`:
  - `kzalloc` `struct trace_array`.
  - `tr->name = kstrdup(name)`.
  - Allocate `tracing_cpumask` (default all-CPUs) + `pipe_cpumask` (default empty).
  - Optionally `tr->system_names = kstrdup_const(systems)` for selective event registration.
  - `tr->trace_flags = global_trace.trace_flags & ~ZEROED_TRACE_FLAGS`.
  - `tr->current_trace = &nop_trace`.
  - Init lists `systems`, `events`, `hist_vars`, `err_log`, `tracers`, `marker_list`, optional `mod_events`.
  - `allocate_trace_buffers(tr, trace_buf_size)`:
    - `allocate_trace_buffer(tr, &tr->array_buffer, size)` ⟹ `ring_buffer_alloc(size, rb_flags)` (or `ring_buffer_alloc_range` for memmap'd persistent ring).
    - `tr->array_buffer.data = alloc_percpu(struct trace_array_cpu)`.
    - `trace_set_buffer_entries(...)`.
    - If CONFIG_TRACER_SNAPSHOT and not range-mapped: `trace_allocate_snapshot(tr, size)`.
  - `trace_set_ring_buffer_expanded(tr)`.
  - `ftrace_allocate_ftrace_ops(tr)` (per-instance function-tracer ops slot).
  - `trace_array_init_autoremove(tr)` (per-empty-instance auto-removal worker).
  - `ftrace_init_trace_array(tr)`.
  - `init_trace_flags_index(tr)`.
  - If `trace_instance_dir`: `trace_array_create_dir(tr)` ⟹ `tracefs_create_dir(name, trace_instance_dir)` + `event_trace_add_tracer(dir, tr)` + `init_tracer_tracefs(tr, dir)` + `__update_tracer(tr)`.
  - Add to `ftrace_trace_arrays`; `tr->ref++`; return.
- `instance_mkdir(name)` is the tracefs callback (`tracefs/instances` `i_op->mkdir`); takes `event_mutex` + `trace_types_lock`, calls `trace_array_create(name)`.
- `instance_rmdir(name)`: lookup, `__remove_instance(tr)`.

REQ-12: `trace_array_destroy(this_tr)`:
- Walk `ftrace_trace_arrays`; under `trace_types_lock` + `event_mutex`.
- `__remove_instance(tr)`: bail if `tr->ref > 1` or `tr->mapped` or open pipes.
- Reset tracer to `nop_trace`, tear down events, free probes, `event_trace_del_tracer`, `ftrace_destroy_function_files`, `tracefs_remove(tr->dir)`, `free_trace_buffers(tr)`, `free_cpumask_var(...)`, `kfree(tr->name)`, `kfree(tr)`.
- `EXPORT_SYMBOL_GPL`.

REQ-13: Per-`trace_flags` (`tr->trace_flags`, bit per option) — exposed at `options/<name>` and concatenated in `trace_options`. Settings via `set_tracer_flag(tr, mask, enable)`:
- Validate via `tr->current_trace->flag_changed` (tracer veto).
- Apply `tr->trace_flags |= mask` or `&= ~mask`.
- Side-effects per flag:
  - `TRACE_ITER_TRACE_PRINTK`: re-pin `printk_trace` to `tr`.
  - `TRACE_ITER_COPY_MARKER`: `update_marker_trace(tr, enabled)`.
  - `TRACE_ITER_RECORD_CMD`: `trace_event_enable_cmd_record(enabled)` (savedcmdlines).
  - `TRACE_ITER_RECORD_TGID`: alloc tgid_map; `trace_event_enable_tgid_record(enabled)`.
  - `TRACE_ITER_EVENT_FORK`: `trace_event_follow_fork(tr, enabled)`.
  - `TRACE_ITER_FUNC_FORK`: `ftrace_pid_follow_fork(tr, enabled)`.
  - `TRACE_ITER_OVERWRITE`: `ring_buffer_change_overwrite(tr->array_buffer.buffer, enabled)` + snapshot.
  - `TRACE_ITER_PRINTK`: `trace_printk_start_stop_comm` + `trace_printk_control`.
  - `TRACE_GRAPH_GRAPH_TIME`: `ftrace_graph_graph_time_control(enabled)`.
- Two flags `RECORD_TGID`, `RECORD_CMD`, `TRACE_PRINTK`, `COPY_MARKER` require `event_mutex` held — asserted via `lockdep_assert_held`.

REQ-14: Per-`tracing_thresh` (global ns threshold):
- Boot cmdline: `__setup("tracing_thresh=", set_tracing_thresh)`.
- tracefs: `tracing_thresh_read/write` reads/writes `tracing_thresh` as ns; on write, calls `tr->current_trace->update_thresh(tr)` if defined (irqsoff / wakeup / hwlat consumes it to skip sub-threshold records).
- Tracefs file mode 0644, root-owned.

REQ-15: Per-current_tracer file (`current_tracer`):
- `tracing_set_trace_read`: returns `tr->current_trace->name`.
- `tracing_set_trace_write`: copies a name (≤ MAX_TRACER_SIZE), `strim`s, dispatches to `tracing_set_tracer(tr, name)`.
- File mode 0644.

REQ-16: `trace_array_get()` / `trace_array_put()`:
- `__trace_array_get(tr)` ⟹ if `tr->ref == INT_MAX` ⟹ `-EBUSY`; else `tr->ref++`; return 0.
- `__trace_array_put(tr)`: `tr->ref--`; if zero ⟹ kick auto-remove worker (`trace_array_kick_autoremove`).
- `trace_array_put` is `EXPORT_SYMBOL_GPL`; `trace_array_get` exists internally + via `trace_array_get_by_name`.
- All ref ops under `trace_types_lock`.

REQ-17: Per-event-export hooks:
- `register_ftrace_export(export)`: register an external consumer (`flag` ∈ `TRACE_EXPORT_FUNCTION|EVENT|MARKER`) to be called from `ftrace_exports` after every commit.
- Linked into either `ftrace_function_list`, `ftrace_event_list`, or `ftrace_marker_list`.
- Static keys `trace_function_exports_enabled` / `_event_` / `_marker_` gate the fast path.
- `EXPORT_SYMBOL_GPL`.

REQ-18: Boot cmdline parsers:
- `ftrace=<tracer>` ⟹ `default_bootup_tracer`.
- `ftrace_dump_on_oops[=orig_cpu|=2|=all]` ⟹ `ftrace_dump_on_oops` global.
- `traceoff_on_warning` ⟹ `__disable_trace_on_warning`.
- `trace_instance=<name>` ⟹ `boot_instance_index/info` to mkdir during init.
- `trace_options=<csv>` ⟹ `trace_boot_options_buf`.
- `trace_clock=<name>` ⟹ `trace_boot_clock`.
- `tp_printk` ⟹ static_key.
- `tp_printk_stop_on_boot` ⟹ stop after init.
- `traceoff_after_boot` ⟹ `tracing_off()` after late_initcall.
- `trace_buf_size=<size>` ⟹ `trace_buf_size` (clamped ≥ 4096).
- `tracing_thresh=<ns>` ⟹ `tracing_thresh`.

## Acceptance Criteria

- [ ] AC-1: `trace_array_create("foo")` produces `instances/foo/` with `current_tracer`, `tracing_on`, `trace`, `trace_pipe`, `trace_marker`, `set_event`, `events/`, `per_cpu/`, `buffer_size_kb`, `options/`.
- [ ] AC-2: `echo function > /sys/kernel/tracing/current_tracer` swaps `current_trace`; `cat current_tracer` reports `function`; sched events still functional.
- [ ] AC-3: `echo 0 > tracing_on` ⟹ `ring_buffer_record_is_set_on` returns false; no events recorded; `echo 1 > tracing_on` re-enables.
- [ ] AC-4: `echo 1 > snapshot` swaps live↔snapshot atomically under `max_lock` IRQ-disabled; reading `snapshot` shows previously-live contents; reading `trace` shows fresh.
- [ ] AC-5: `trace_event_buffer_lock_reserve` with a filtered event uses per-cpu scratch `trace_buffered_event`; if filter accepts, copies into ring on commit; if filter rejects, dropped without ring traffic.
- [ ] AC-6: `tracing_thresh` write triggers `current_trace->update_thresh`; irqsoff drops records < threshold.
- [ ] AC-7: `tracing_snapshot()` is no-op + WARN_ONCE when CONFIG_TRACER_SNAPSHOT=n.
- [ ] AC-8: `tp_printk` boot opt routes every event through `output_printk` to console.
- [ ] AC-9: `traceoff_on_warning` + a `WARN()` calls `disable_trace_on_warning` ⟹ `tracing_off()` for both `global_trace` and `printk_trace`.
- [ ] AC-10: Set `trace_options` `overwrite` ⟹ `ring_buffer_change_overwrite(tr, true)`; live buffer overwrites oldest on wrap.
- [ ] AC-11: `register_tracer` rejects duplicate name and rejects under LOCKDOWN_TRACEFS with `-EPERM`.
- [ ] AC-12: `trace_array_destroy(tr)` rejects with `-EBUSY` if `tr->ref > 1` or `tr->mapped` or pipe open.
- [ ] AC-13: `set_tracer_flag(tr, RECORD_TGID, 1)` allocates `tgid_map`; subsequent events have `task_struct->tgid` recorded; `cat saved_tgids` non-empty.
- [ ] AC-14: `register_ftrace_export` enables static-key `trace_event_exports_enabled`; every event commit calls the registered export.

## Architecture

```
struct TraceArray {
  name:                Option<KString>,
  dir:                 *TracefsDentry,
  array_buffer:        ArrayBuffer,
  snapshot_buffer:     Option<ArrayBuffer>,
  current_trace:       *Tracer,
  current_trace_flags: u64,
  tracers:             List<TracerNode>,
  trace_flags:         u64,
  trace_flags_index:   [u8; TRACE_FLAGS_MAX_SIZE],
  tracing_cpumask:     CpuMask,
  pipe_cpumask:        CpuMask,
  buffer_disabled:     bool,
  ring_buffer_expanded:bool,
  ref:                 i32,
  mapped:              i32,
  trace_ref:           i32,                 // open pipe count
  max_lock:            ArchSpinLock,
  start_lock:          RawSpinLock,
  systems:             List<EventSubsystem>,
  events:              List<TraceEventFile>,
  hist_vars:           List<HistVar>,
  err_log:             List<TracingLogErr>,
  marker_list:         List<MarkerCopy>,
  function_pids:       Option<TracePidList>,
  function_no_pids:    Option<TracePidList>,
  current_trace_priv:  *mut(),              // tracer-private
  clock_id:            u8,
  flags:               u32,                 // TRACE_ARRAY_FL_*
  // ...
}

struct ArrayBuffer {
  tr:         *TraceArray,
  buffer:     *TraceBuffer,               // ring_buffer.c handle
  data:       PerCpu<TraceArrayCpu>,
  time_start: u64,
}

struct Tracer {
  name:           &'static str,
  init:           Option<fn(*TraceArray) -> i32>,
  reset:          Option<fn(*TraceArray)>,
  start:          Option<fn(*TraceArray)>,
  stop:           Option<fn(*TraceArray)>,
  update_thresh:  Option<fn(*TraceArray) -> i32>,
  open:           Option<fn(*TraceIterator)>,
  pipe_open:      Option<fn(*TraceIterator)>,
  close:          Option<fn(*TraceIterator)>,
  read:           Option<...>,
  splice_read:    Option<...>,
  wait_pipe:      Option<...>,
  print_header:   Option<fn(*SeqFile)>,
  print_line:     Option<fn(*TraceIterator) -> PrintLineT>,
  set_flag:       Option<fn(*TraceArray, u32, u64, i32) -> i32>,
  flag_changed:   Option<fn(*TraceArray, u64, i32) -> i32>,
  flags:          Option<*TracerFlags>,
  print_max:      bool,
  use_max_tr:     bool,
  allocated_snapshot: bool,
  noboot:         bool,
  enabled:        i32,
  next:           *Tracer,
}
```

`Trace::register_tracer(type) -> i32`:
1. Validate `type.name`.
2. If `security_locked_down(LOCKDOWN_TRACEFS)`: return `-EPERM`.
3. `guard(mutex)(&trace_types_lock)`.
4. For each `t in trace_types`: if `name eq` ⟹ return `-1`.
5. If `type.flags`: `type.flags.trace = type`.
6. `do_run_tracer_selftest(type)`; on `< 0` return.
7. For each `tr in ftrace_trace_arrays`: `add_tracer(tr, type)`.
8. `type.next = trace_types; trace_types = type`.
9. If `default_bootup_tracer` matches: `tracing_set_tracer(&global_trace, type.name)` + `apply_trace_boot_options()` + `disable_tracing_selftest`.
10. Return 0.

`TraceArray::set_tracer(buf) -> i32`:
1. `guard(mutex)(&trace_types_lock)`.
2. `update_last_data(self)`.
3. If `!ring_buffer_expanded`: `__tracing_resize_ring_buffer(self, trace_buf_size, ALL_CPUS)`.
4. Search `self.tracers` for `t.tracer.name == buf` ⟹ `trace`.
5. `-EINVAL` if not found; 0 if `trace == current_trace`.
6. If `tracer_uses_snapshot(trace)`: under `max_lock` IRQ-off, ensure `cond_snapshot == NULL` else `-EBUSY`.
7. If `system_state < SYSTEM_RUNNING && trace.noboot`: `-EINVAL`.
8. If `!trace_ok_for_array(trace, self)`: `-EINVAL`.
9. If `self.trace_ref`: `-EBUSY`.
10. `trace_branch_disable()`.
11. `current_trace.enabled--`.
12. If `current_trace.reset`: `current_trace.reset(self)`.
13. `had_max_tr = tracer_uses_snapshot(current_trace)`.
14. `current_trace = &nop_trace; current_trace_flags = nop_trace.flags`.
15. If `had_max_tr && !tracer_uses_snapshot(trace)`: `synchronize_rcu(); free_snapshot(self); tracing_disarm_snapshot(self)`.
16. If `!had_max_tr && tracer_uses_snapshot(trace)`: `tracing_arm_snapshot_locked(self)` else rewind.
17. `current_trace_flags = t.flags ?: trace.flags`.
18. If `trace.init`: `tracer_init(trace, self)`; on error rewind to `nop_trace`.
19. `current_trace = trace; current_trace.enabled++`.
20. `trace_branch_enable(self)`.
21. Return 0.

`Trace::on() / off()`:
1. `tracer_tracing_on(&global_trace)` / `_off(...)`.
2. Per-`tracer_tracing_on(tr)`: if `tr.array_buffer.buffer`: `ring_buffer_record_on(buf)`; `tr.buffer_disabled = 0`.
3. Per-`tracer_tracing_off(tr)`: symmetric; `tr.buffer_disabled = 1`.
4. Per-`tracer_tracing_disable(tr)` / `_enable(tr)`: counted; `ring_buffer_record_disable/enable`.

`TraceArray::snapshot()`:
1. `if !CONFIG_TRACER_SNAPSHOT { WARN_ONCE; return }`.
2. `tracing_snapshot_instance(self)`:
   - Per-`if !tr.allocated_snapshot && !tr.cond_snapshot`: alloc.
   - Under `tr.max_lock` IRQ-off: swap `array_buffer ↔ snapshot_buffer` (pointers, percpu data) + `update_max_tr_single` for latency tracers.

`TraceEventBuffer::lock_reserve(current_rb, trace_file, type, len, trace_ctx) -> *Event`:
1. `*current_rb = trace_file.tr.array_buffer.buffer`.
2. If `!tr.no_filter_buffering_ref && (trace_file.flags & (SOFT_DISABLED|FILTERED))`:
   - `preempt_disable_notrace()`.
   - `event = this_cpu_read(trace_buffered_event)`.
   - If `event`: `val = this_cpu_inc_return(trace_buffered_event_cnt)`.
     - If `val == 1 && len ≤ PAGE_SIZE - struct_size(event, array, 1)`:
       - `trace_event_setup(event, type, trace_ctx); event.array[0] = len`.
       - Return event (preempt still disabled).
     - Else `this_cpu_dec(trace_buffered_event_cnt)`.
   - `preempt_enable_notrace()`.
3. `entry = __trace_buffer_lock_reserve(*current_rb, type, len, trace_ctx)`.
4. If `!entry && (trace_file.flags & TRIGGER_COND)`: `*current_rb = temp_buffer; entry = __trace_buffer_lock_reserve(...)`.
5. Return entry.

`TraceEventBuffer::commit(fbuffer)`:
1. `tt = ETT_NONE`.
2. If `__event_trigger_test_discard(file, buffer, event, entry, &tt)`: jump to discard.
3. If `tracepoint_printk_key`: `output_printk(fbuffer)`.
4. If `trace_event_exports_enabled`: `ftrace_exports(event, TRACE_EXPORT_EVENT)`.
5. `trace_buffer_unlock_commit_regs(file.tr, buffer, event, trace_ctx, regs)`:
   - `__buffer_unlock_commit(buffer, event)`.
   - `ftrace_trace_stack(tr, buffer, ctx, regs?0:STACK_SKIP, regs)`.
   - `ftrace_trace_userstack(tr, buffer, ctx)`.
6. discard: if `tt != ETT_NONE`: `event_triggers_post_call(file, tt)`.

`Trace::buffered_event_enable()`:
1. `WARN_ON_ONCE(!mutex_is_locked(&event_mutex))`.
2. `if trace_buffered_event_ref++ != 0`: return.
3. For each tracing CPU:
   - `page = alloc_pages_node(cpu_to_node(cpu), GFP_KERNEL | __GFP_NORETRY, 0)`.
   - If !page: warn, break (best-effort).
   - `event = page_address(page); memset(event, 0, sizeof(*event))`.
   - `per_cpu(trace_buffered_event, cpu) = event`.

`Trace::buffered_event_disable()`:
1. Assert `event_mutex`.
2. `if --trace_buffered_event_ref != 0`: return.
3. `on_each_cpu_mask(tracing_buffer_mask, disable_trace_buffered_event)` (increment cnt to force slow path).
4. `synchronize_rcu()`.
5. For each tracing CPU: `free_page(per_cpu(trace_buffered_event, cpu))`; set NULL.
6. `synchronize_rcu()` (second barrier so racing reservers can't observe stale pointer + cleared cnt).
7. `on_each_cpu_mask(tracing_buffer_mask, enable_trace_buffered_event)` (decrement cnt).

`Trace::array_create_systems(name, systems, range_start, range_size) -> *TraceArray`:
1. `kzalloc(TraceArray)`.
2. `name = kstrdup(name)`.
3. Alloc `tracing_cpumask` (all-CPUs), `pipe_cpumask` (zero).
4. Optional `system_names = kstrdup_const(systems)`.
5. `range_addr_start = range_start; range_addr_size = range_size`.
6. `trace_flags = global_trace.trace_flags & ~ZEROED_TRACE_FLAGS`.
7. `current_trace = &nop_trace`.
8. Init all the lists.
9. `allocate_trace_buffers(self, trace_buf_size)`:
   - `allocate_trace_buffer(self, &array_buffer, size)` ⟹ `ring_buffer_alloc(size, rb_flags)` or `_alloc_range`.
   - `array_buffer.data = alloc_percpu(TraceArrayCpu)`.
   - `trace_set_buffer_entries(...)`.
   - `trace_allocate_snapshot(self, size)` if CONFIG_TRACER_SNAPSHOT.
10. `trace_set_ring_buffer_expanded(self)`.
11. `ftrace_allocate_ftrace_ops(self)`.
12. `trace_array_init_autoremove(self)`.
13. `ftrace_init_trace_array(self)`.
14. `init_trace_flags_index(self)`.
15. If `trace_instance_dir`: `trace_array_create_dir(self)`.
16. `list_add(&self.list, &ftrace_trace_arrays); self.ref = 1`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `tr_ref_balanced` | INVARIANT | per-instance: `trace_array_get/put` pairs across all paths; `ref ∈ [0, INT_MAX]`. |
| `set_tracer_no_reentrant_init` | INVARIANT | per-set_tracer: `current_trace = nop_trace` set before calling `trace.init`; no concurrent init for same tr. |
| `snapshot_swap_under_max_lock_irq_off` | INVARIANT | per-snapshot swap: `max_lock` held, IRQs disabled. |
| `buffered_event_double_synchronize_rcu` | INVARIANT | per-disable: two `synchronize_rcu()` separate the per-cpu pointer clear from the cnt re-enable. |
| `trace_buffered_event_cnt_balanced` | INVARIANT | per-cpu cnt inc/dec paired across reserve/commit/abandon. |
| `lock_reserve_returns_preempt_disabled_on_fast_path` | INVARIANT | per-reserve fast path returns with preempt disabled. |
| `instance_create_failure_unwinds` | INVARIANT | per-create_systems: every alloc failure jumps to `out_free_tr` and cleans up all earlier allocations. |
| `trace_array_destroy_rejects_pinned` | INVARIANT | per-destroy: `ref > 1` or `mapped > 0` or `trace_ref > 0` ⟹ refused. |
| `current_trace_flags_initialized_before_use` | INVARIANT | every read of `tr.current_trace_flags` follows a `set_tracer` or instance-create initialization. |

### Layer 2: TLA+

`kernel/trace/trace-core.tla`:
- Per-N-CPU producer + per-instance + per-tracer + per-snapshot subsystem.
- Properties:
  - `safety_record_lifecycle` — per-event: reserve ⟹ commit XOR discard (no torn).
  - `safety_set_tracer_atomic` — per-set_tracer: observers see either old `current_trace` or new, never a partially-initialized state.
  - `safety_snapshot_swap_atomic` — per-snapshot: every event is in exactly one of {live, snapshot} buffers, never both or neither.
  - `safety_filter_scratch_drain` — per-buffered_event_disable: no producer holds a pointer past the second `synchronize_rcu`.
  - `liveness_pipe_reader_progress` — per-trace_pipe reader: eventually receives data when producer commits.
  - `liveness_tracing_off_observed` — per-tracing_off: bounded latency for `ring_buffer_record_is_set_on` to flip.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `register_tracer` post: name unique in `trace_types` OR err | `Trace::register_tracer` |
| `set_tracer` post: `current_trace == trace`, ref accounting balanced | `TraceArray::set_tracer` |
| `set_tracer` exception path: on `init` failure, `current_trace == nop_trace` | `TraceArray::set_tracer` |
| `tracing_on` post: `ring_buffer_record_is_set_on(buffer) == true` | `Trace::on` |
| `tracing_off` post: `ring_buffer_record_is_set_on(buffer) == false` | `Trace::off` |
| `snapshot` post: live and snapshot pointers swapped exactly once | `TraceArray::snapshot` |
| `lock_reserve` post: returns valid event ptr ∨ NULL; if filter-fast-path returned event then preempt disabled | `TraceEventBuffer::lock_reserve` |
| `commit` post: ring committed XOR temp_buffer routed XOR discarded | `TraceEventBuffer::commit` |
| `trace_array_create_systems` post: instance fully initialized + in list OR err with no leaks | `Trace::array_create_systems` |
| `trace_array_destroy` post: instance freed iff refcount drained | `TraceArray::destroy` |
| `set_tracer_flag` post: `tr.trace_flags & mask == enabled ? mask : 0` ∧ side-effect dispatched | `TraceArray::set_flag` |
| `tracing_thresh_write` post: `tracing_thresh == ns(input)` ∧ `update_thresh` invoked if defined | `Trace::thresh_write` |
| `register_ftrace_export` post: static key incremented ∧ list contains export | `Trace::register_export` |

### Layer 4: Verus/Creusot functional

`Per-trace lifecycle` semantic equivalence (per `Documentation/trace/ftrace.rst` and `Documentation/trace/ring-buffer-design.rst`):

```
mkdir instances/foo
  → trace_array_create("foo") → instance fully visible at instances/foo/
echo function > current_tracer
  → tracing_set_tracer(tr, "function") → tr.current_trace = function_tracer; tr.current_trace.enabled = 1
echo 0 > tracing_on
  → tracing_off() → ring_buffer_record_off(buf)
echo 1 > snapshot
  → tracing_snapshot_alloc() (if needed) + tracing_snapshot()
event commit path:
  trace_event_buffer_lock_reserve(rb, file, type, len, ctx)
    → __trace_buffer_lock_reserve → ring_buffer_lock_reserve
  fill entry
  trace_event_buffer_commit(fbuffer)
    → trigger-discard? → discard
    → tp_printk? → output_printk
    → exports? → ftrace_exports
    → __buffer_unlock_commit → ring_buffer_unlock_commit
    → ftrace_trace_stack + ftrace_trace_userstack
    → event_triggers_post_call
rmdir instances/foo
  → instance_rmdir → __remove_instance → ringbuf+events+probes torn down
```

## Hardening

(Inherits row-1 features from `kernel/trace/00-overview.md` § Hardening.)

Trace-core reinforcement:

- **Per-`trace_types_lock` mutex strict** — defense against per-concurrent set_tracer + register_tracer race.
- **Per-`tr->max_lock` arch_spinlock IRQ-off for snapshot** — defense against per-NMI-during-snapshot torn buffer pointer.
- **Per-double-`synchronize_rcu()` on buffered_event teardown** — defense against per-stale-percpu-pointer use-after-free.
- **Per-`current_trace = nop_trace` transition before `init`** — defense against per-`init`-faulting-with-old-tracer-half-uninstalled.
- **Per-`trace_branch_disable` around set_tracer** — defense against per-branch-tracer re-entrancy.
- **Per-`tr.trace_ref` blocks set_tracer** — defense against per-pipe-reader observing inconsistent tracer mid-read.
- **Per-`tr.mapped` blocks destroy** — defense against per-userspace-zero-copy-mmap UAF.
- **Per-`LOCKDOWN_TRACEFS` security gate at register_tracer** — defense against per-secureboot-tracer-injection.
- **Per-`disable_trace_on_warning` flips tracing_off on first WARN** — defense against per-WARN-storm cascading log loss + capture preceding events.
- **Per-`trace_buffered_event` page is per-cpu, per-node, GFP_KERNEL|__GFP_NORETRY** — defense against per-OOM-during-tracer-enable.
- **Per-event-trigger-discard before commit** — defense against per-malicious-filter-bypass.
- **Per-tracefs default mode 0700 (root-only)** — defense against per-unprivileged-trace-read.
- **Per-`tr.ref` strict get/put balanced** — defense against per-instance-UAF on rmdir-race.
- **Per-`saved_func`/`ops->func` distinct copies** — defense against per-pid-filter mutating live func.

## Grsecurity/PaX-style Reinforcement

Baseline (apply to every Tier-3 surface):

- **PAX_USERCOPY** — tracefs `write` paths copy command lines (`set_ftrace_filter`, `trace_marker`) through size-bounded slab whitelists; `trace_marker_raw` enforces `TRACE_BUF_SIZE` upper bound on every copy.
- **PAX_KERNEXEC** — `trace_event_call->probe` callbacks live in `.text`; W^X enforced by static-call patching.
- **PAX_RANDKSTACK** — re-randomized per syscall entry to tracefs.
- **PAX_REFCOUNT** — `trace_array.ref`, `trace_event_file.flags` user-counter and `tracer->ref` saturate.
- **PAX_MEMORY_SANITIZE** — `trace_array` slab freed via call-RCU with zero-on-free.
- **PAX_UDEREF** — tracefs reads/writes only via `copy_from/to_user`.
- **PAX_RAP / kCFI** — tracer's `init/reset/start/stop/print_line` op-table type-tagged.
- **GRKERNSEC_HIDESYM** — tracefs file ops (`tracing_open_file_tr` etc.) hidden from non-CAP_SYSLOG kallsyms.
- **GRKERNSEC_DMESG** — `WARN_ONCE` inside `trace.c` gated behind CAP_SYSLOG.

Subsystem-specific reinforcement:

- **tracefs CAP_SYS_ADMIN** — `tracefs_create_file` permission mask is `0644` for root only; opens of `set_ftrace_filter`, `current_tracer`, `set_event` and per-instance equivalents require CAP_SYS_ADMIN in the initial userns. Userns CAP_SYS_ADMIN explicitly insufficient under GRKERNSEC_FTRACE strict.
- **trace_event filter validation** — `apply_subsystem_event_filter` / `create_filter` walk the parsed predicate AST validating field offsets against the event's `trace_event_fields[]` table; out-of-range or type-mismatched comparisons rejected with `-EINVAL` before commit, preventing OOB-read via crafted filter string.
- **Instance lifecycle (`tr->ref`)** — `instance_rmdir` waits for `tr->ref == 0`; PAX_REFCOUNT saturation prevents underflow that would race-free a live instance and produce UAF on concurrent `trace_pipe` reader.
- **`saved_func` distinct copy** — `set_ftrace_pid` updates a side `saved_func` rather than mutating the live `ops->func`; flip is via static-call swap atomically.
- **Rationale** — `trace.c` is the front door to ftrace/tracefs; capability gating + filter-AST validation + refcounted instance lifecycle collapse the historical "write a filter, get a kernel read primitive" and "rmdir-during-read UAF" classes.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Ring buffer internals (covered in `ring-buffer.md` Tier-3).
- Function-tracer mcount/fentry patching (covered in `ftrace.md` Tier-3).
- Event class registration + filter/trigger/hist (covered in `trace-events.md` Tier-3).
- Dynamic kprobe/uprobe/eprobe events (covered in `dynevent.md` Tier-3).
- Synthetic events (covered in `synth.md` Tier-3).
- Boot-time tracing (covered in `boot.md` Tier-3).
- BPF tracing prog dispatch (covered in `bpf-trace.md` Tier-3).
- Specialized tracers (irqsoff, wakeup, osnoise, hwlat, ...) covered in `specialized.md` Tier-3.
- Implementation code.
