# Tier-2: kernel/trace — tracing subsystem (ftrace + tracefs + ringbuf + kprobes/uprobes events + bpf-trace + blktrace + RV monitor + boot-time tracing)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/trace/
  - include/linux/trace_events.h
  - include/linux/ftrace.h
  - include/linux/tracepoint.h
-->

## Summary

Tier-2 wrapper for the kernel tracing subsystem — every `TRACE_EVENT(...)` definition, every `function` ftrace probe, every kprobe/uprobe-via-tracefs event, every BPF tracing program's kernel-side dispatch, every `blktrace` / `blktrace -d /dev/sda`, every Runtime Verification monitor. Components: **ftrace core** (`ftrace.c` + `fgraph.c` + `fprobe.c` + `rethook.c` — function-tracer + function-graph + multi-fprobe + return-hooks), **trace core** (`trace.c` + `trace.h` + `trace_*.c` ~50 files: per-event registration, per-instance ringbuf, tracefs hierarchy at `/sys/kernel/tracing/`, output formatters, latency-tracer, function-tracer-with-args, `osnoise`/`hwlat`/`mmiotrace`/`branch`/`stack`), **ring_buffer** (`ring_buffer.c` + `simple_ring_buffer.c` — per-cpu lockless ring used by all tracers), **synthetic + dynamic events** (`trace_kprobe.c` + `trace_uprobe.c` + `trace_eprobe.c` + `synth_event_gen_test.c` + `kprobe_event_gen_test.c` + `trace_dynevent.c` + `trace_synth.c` + `trace_probe.c` — dynamic kprobe/uprobe + synthetic events from userspace via tracefs writable knobs), **boot-time tracing** (`trace_boot.c` — `ftrace=` / `trace_event=` cmdline-controlled tracing-during-boot), **bpf-trace** (`bpf_trace.c` + `bpf_trace.h` — BPF program attach to tracepoints / kprobes / uprobes / perf events; consumer of `kernel/bpf/`), **blktrace** (`blktrace.c` — block IO tracer), **RV** (`rv/` — Runtime Verification monitor framework, attaches DA model monitors to tracepoints to detect runtime spec violations), **specialized tests** (`preemptirq_delay_test.c`, `trace_benchmark.c`, `ring_buffer_benchmark.c`, `remote_test.c`), **error-report** (`error_report-traces.c`), **power** (`power-traces.c`, `rpm-traces.c`), **pid_list** (`pid_list.c`).

Heavy cross-ref: `kernel/perf/00-overview.md` (perf-event-via-tracepoint shares trace_kprobe/uprobe frontends), `kernel/bpf/00-overview.md` (BPF tracing prog types), `block/00-overview.md` (blktrace), `kernel/cgroup/00-overview.md` (per-cgroup tracing), `arch/x86/idt.md` (function-tracer's mcount/fentry mechanism + INT3-based kprobes).

## Compatibility contract — outline

- `/sys/kernel/tracing/` (or legacy `/sys/kernel/debug/tracing/`) full hierarchy byte-identical: `current_tracer`, `available_tracers`, `tracing_on`, `trace`, `trace_pipe`, `trace_marker`, `trace_marker_raw`, `set_event`, `events/<group>/<event>/{enable,filter,trigger,format,id,hist,inject}`, `set_ftrace_filter`, `set_ftrace_notrace`, `set_graph_function`, `set_graph_notrace`, `set_event_pid`, `set_ftrace_pid`, `kprobe_events`, `uprobe_events`, `dynamic_events`, `synthetic_events`, `osnoise/`, `hwlat_detector/`, `instances/<name>/...` (per-instance trace), `per_cpu/cpu<N>/...`, `printk_formats`, `available_filter_functions`, `available_events`, `error_log`, `function_profile_enabled`, `enabled_functions`, `tracing_max_latency`, `tracing_thresh`, `buffer_size_kb`, `buffer_total_size_kb`, `buffer_subbuf_size_kb`, `free_buffer`, `options/{...}`, `saved_cmdlines`, `saved_cmdlines_size`, `saved_tgids`, `set_event_pid`, `stack_max_size`, `stack_trace`, `stack_trace_filter`, `tracing_cpumask`, `tracing_cpu_filter`, `trace_clock`, `trace_options`, `trace_stat/`, `trampoline_size`, `uprobe_profile`, `kprobe_profile`, `events/header_event`, `events/header_page`, `events/enable`, `events/filter`, `set_kprobes_blacklist`.

- Per-event format file (`events/<grp>/<evt>/format`) byte-identical so libtraceevent / trace-cmd / perf-trace / bpftrace / bcc-tools consume unchanged.

- `/sys/kernel/tracing/` mount permissions + admin gating identical (TRACEFS_MAGIC).

- `events/<group>/<event>/id` numeric IDs stable per-build (consumed as `attr.config` for `perf_event_open(PERF_TYPE_TRACEPOINT, ...)`).

- Boot-time `ftrace=`, `trace_event=`, `trace_options=`, `trace_buf_size=`, `tp_printk`, `nokaslr` (for stable kprobe addresses) cmdline parsed identically.

- blktrace `BLKTRACESETUP`, `_START`, `_STOP`, `_TEARDOWN` IOCTLs byte-identical (blktrace + blkparse + btt consume unchanged).

- RV `/sys/kernel/tracing/rv/` (CONFIG_RV) per-monitor enable/disable byte-identical.

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `kernel/trace/ftrace.md` | `ftrace.c` + `fgraph.c` + `fprobe.c` + `rethook.c`: function-tracer + graph + fprobe + rethook |
| `kernel/trace/ring-buffer.md` | `ring_buffer.c` + `simple_ring_buffer.c`: per-cpu lockless ring |
| `kernel/trace/trace-core.md` | `trace.c` + `trace.h` + `trace_output.c` + `trace_seq.c`: per-instance trace + tracefs hierarchy |
| `kernel/trace/trace-events.md` | `trace_events.c` + `trace_events_*.c` + `trace_event_perf.c`: event registration + filter + trigger + hist |
| `kernel/trace/dynevent.md` | `trace_dynevent.c` + `trace_kprobe.c` + `trace_uprobe.c` + `trace_eprobe.c`: dynamic events (kprobes/uprobes/eprobes via tracefs) |
| `kernel/trace/synth.md` | `trace_synth.c` + `synth_event_gen_test.c`: synthetic events |
| `kernel/trace/boot.md` | `trace_boot.c`: boot-time tracing via cmdline |
| `kernel/trace/bpf-trace.md` | `bpf_trace.c` + `bpf_trace.h`: BPF prog attach to tracepoints / kprobes / perf-events |
| `kernel/trace/blktrace.md` | `blktrace.c`: block IO tracer |
| `kernel/trace/specialized.md` | `trace_branch.c`, `trace_osnoise.c`, `trace_hwlat.c`, `mmiotrace.c`, `trace_stack.c`, `trace_functions.c`, `trace_functions_graph.c`, `trace_irqsoff.c`, `trace_sched_wakeup.c`, `trace_nop.c`, `trace_mmiotrace.c` |
| `kernel/trace/rv.md` | `rv/`: Runtime Verification monitor framework |
| `kernel/trace/pid-list.md` | `pid_list.c` + `pid_list.h`: per-instance PID filter |

## Compatibility outline

- REQ-O1: Full `/sys/kernel/tracing/` hierarchy byte-identical (trace-cmd / perf / bpftrace / bcc-tools / libtraceevent consume unchanged).
- REQ-O2: Per-event `format` file content byte-identical.
- REQ-O3: ftrace `current_tracer` set/unset for every available tracer behaves identically.
- REQ-O4: kprobe + uprobe + eprobe + synthetic event creation via tracefs writable knobs identical.
- REQ-O5: BPF prog attach via `BPF_RAW_TRACEPOINT_OPEN` + `BPF_LINK_CREATE` identical.
- REQ-O6: blktrace IOCTLs + procfs identical.
- REQ-O7: Boot-time tracing cmdline parsed identically.
- REQ-O8: RV monitor enable/disable identical.
- REQ-O9: TLA+ models (ring-buffer per-cpu lockless producer-consumer; ftrace mcount-call atomic-patch; kprobe INT3 install/uninstall race; BPF prog attach/detach during fire).
- REQ-O10: Hardening: tracefs root-only by default; kprobe creation CAP_SYS_ADMIN + LSM mediation; BPF prog attach gated.

## Acceptance Criteria

- [ ] AC-O1: `trace-cmd record -e 'sched:*'` for 1s captures sched events; `trace-cmd report` decodes correctly.
- [ ] AC-O2: `bpftrace -e 'kprobe:do_sys_openat2 { @[comm]=count() }'` runs.
- [ ] AC-O3: `blktrace -d /dev/sda` + `blkparse` round-trip works.
- [ ] AC-O4: `perf record -e 'syscalls:sys_enter_*'` captures syscall enter events.
- [ ] AC-O5: `osnoise` tracer reports thread-osnoise samples.
- [ ] AC-O6: kselftest `tools/testing/selftests/ftrace/` passes.

## Verification

| TLA+ Model | Owner |
|---|---|
| `models/trace/ring_buffer.tla` | `kernel/trace/ring-buffer.md` (proves: per-cpu lockless producer + concurrent reader-snapshot; sub-buffer commit-counter + reader-page swap; no torn event, no lost commit, no double-read) |
| `models/trace/ftrace_patch.tla` | `kernel/trace/ftrace.md` (proves: mcount/fentry call-site atomic patch via text-poke + IPI sync; concurrent execution + patch never observes torn instruction; rollback on partial-failure complete) |
| `models/trace/kprobe_int3.tla` | `kernel/trace/dynevent.md` (proves: kprobe install via INT3 byte-write + concurrent execution at probe-point; INT3-handler dispatches correctly + single-step via per-cpu xol-page; uninstall waits for in-flight INT3-handler completion) |
| `models/trace/bpf_attach.tla` | `kernel/trace/bpf-trace.md` (proves: BPF prog attach to tracepoint + concurrent fire from N CPUs + detach; refcount + RCU-grace ensures no fire after detach completes) |

## Hardening

| Feature | Default |
|---|---|
| **REFCOUNT** | per-instance trace_array + per-event_call + per-tracepoint refcounts use `Refcount` | § Mandatory |
| **CONSTIFY** | per-tracer `tracer_ops`, per-event `trace_event_class` `static const` | § Mandatory |
| **SIZE_OVERFLOW** | per-event payload-size + ring-buffer page-count arithmetic checked | § Mandatory |
| **STRICT_KERNEL_RWX** | ftrace mcount-patch + kprobe INT3-patch use text-poke with ROX-then-RW transition | § Mandatory |

Trace-specific reinforcement: tracefs root-only-by-default (mode 0700 unless explicit `tracefs.gid=` cmdline + group-grant); kprobe creation requires CAP_SYS_ADMIN + LSM hook (`security_kprobe_event_create`); BPF tracing prog attach requires CAP_BPF + CAP_PERFMON; per-instance trace + per-pid trace filter respected; `trace_marker` write rate-limited per-uid.

## Open Questions
(none at Tier-2)

## Out of Scope
- Implementation code
- 32-bit-only paths
