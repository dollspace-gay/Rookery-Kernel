# Tier-2: kernel/perf — perf_event subsystem (perf_event_open + ring-buffer + hw-breakpoint + uprobes/kprobes)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/events/
  - include/linux/perf_event.h
  - include/uapi/linux/perf_event.h
  - arch/x86/events/
  - kernel/trace/trace_kprobe.c
  - kernel/trace/trace_uprobe.c
-->

## Summary

Tier-2 overview for `perf_event_open(2)` — the unified counter / sampling / tracing fabric. Every `perf` userspace tool, every BPF tracing program, every cgroup CPU usage report, every `top -h` percentage, every PMU-based driver (`intel_pstate` heuristics, `intel_rdt` MBM, `amd_pstate`) ultimately reads here. The subsystem braids together:

- **Hardware PMU events** (per-CPU + per-task, fixed + general-purpose counters; cycles, instructions, branch-misses, cache-misses, …)
- **Software events** (cpu-clock, task-clock, page-faults, context-switches, cpu-migrations, alignment-faults, emulation-faults, …)
- **Tracepoint events** (every `TRACE_EVENT(...)` in the kernel; cross-ref future `kernel/trace/00-overview.md`)
- **Kernel-probe events** (kprobes via `tracefs`, dynamic entry/exit instrumentation)
- **User-probe events** (uprobes — userspace breakpoints fired from kernel)
- **Hardware breakpoints** (`hw_breakpoint_*` API; debug-register-backed; consumed by ptrace + perf + bpf)
- **Raw events** (vendor-specific PMU encodings)
- **Bridge events** (cgroup-attached, container-scoped sampling)
- **Per-event ring-buffer** (mmap'd to userspace; AUX area for high-bandwidth sources like Intel-PT / SPE)
- **Callchain capture** (kernel + userspace stack unwind via FP / ORC / DWARF / LBR)

Heavily cross-referenced from `arch/x86/cpu-init.md` (PMU CPUID + LBR + Intel-PT MSRs), `arch/x86/idt.md` (NMI watchdog perf event), `kernel/sched/00-overview.md` (sched-switch tracepoints + task-clock + cpu-migrations), `kernel/cgroup/00-overview.md` (cgroup-attached perf events), `virt/kvm/00-overview.md` (guest PMU passthrough/emulation), `kernel/bpf/00-overview.md` (BPF prog-types `BPF_PROG_TYPE_PERF_EVENT` + `BPF_PROG_TYPE_KPROBE` + `BPF_PROG_TYPE_TRACEPOINT` + `BPF_PROG_TYPE_RAW_TRACEPOINT`), `mm/00-overview.md` (page-fault sampling), every architecture's `perf_event_open` arch-glue.

## Components

- **events core** (`kernel/events/core.c`): `perf_event_open(2)` syscall + `struct perf_event` lifecycle + per-cpu / per-task contexts + scheduling of events onto PMUs + grouping + multiplexing + inheritance into child tasks
- **ring-buffer** (`kernel/events/ring_buffer.c`): mmap'd circular ring + AUX area for snapshot-style PMUs (Intel-PT, ARM-CoreSight); per-buffer head/tail/lost ABI
- **callchain** (`kernel/events/callchain.c`): kernel + user stack unwinder dispatch (per-arch); ORC / FP / DWARF backends
- **hw-breakpoint** (`kernel/events/hw_breakpoint.c`): generic hw-breakpoint allocator (debug registers); shared between ptrace + perf + kgdb + bpf
- **uprobes** (`kernel/events/uprobes.c`): generic uprobe machinery — single-step / xol-area allocation / inode-based probe points; consumed by trace_uprobe + bpf
- **trace_kprobe** (`kernel/trace/trace_kprobe.c`): tracefs-frontend for kprobes; `/sys/kernel/tracing/kprobe_events`
- **trace_uprobe** (`kernel/trace/trace_uprobe.c`): tracefs-frontend for uprobes; `/sys/kernel/tracing/uprobe_events`
- **arch-glue**: `arch/x86/events/core.c` (Intel + AMD common dispatch), `arch/x86/events/intel/{core,bts,cstate,ds,knc,lbr,p4,p6,pt,uncore*,...}.c` (Intel PMUs), `arch/x86/events/amd/{core,ibs,iommu,lbr,power,uncore}.c` (AMD PMUs), `arch/x86/events/msr.c` (MSR-based pseudo-PMU), `arch/x86/events/probe.c`
- **PMU drivers** (`drivers/perf/` — separate Tier-2 future): per-SoC PMU drivers (out-of-scope here, in-scope drivers/perf wrapper)

## Scope

This Tier-2 governs:
- `/home/doll/linux-src/kernel/events/` (5 source files)
- `/home/doll/linux-src/arch/x86/events/` (~25 source files across intel/ and amd/ subdirs)
- The two tracefs frontends `kernel/trace/trace_{kprobe,uprobe}.c` (the rest of `kernel/trace/` belongs to a separate `kernel/trace/00-overview.md` Tier-2)
- `include/linux/perf_event.h` (in-kernel API)
- `include/uapi/linux/perf_event.h` (UAPI struct + ioctl numbers + `enum perf_type_id` + `enum perf_hw_id` + `enum perf_sw_ids` + `enum perf_hw_cache_id` + `struct perf_event_attr` — frozen wire format)

## Compatibility contract — outline

### `perf_event_open(2)` syscall

Direct syscall (no glibc wrapper); takes:
- `struct perf_event_attr *attr` — the canonical 128-byte UAPI struct (size-versioned via `attr->size`); MUST be byte-identical layout
- `pid_t pid` — `-1` (all tasks) / `0` (current) / `>0` (target task) / `-1 with cpu>=0` (per-cpu, all tasks)
- `int cpu` — `-1` (any) / `>=0` (specific cpu)
- `int group_fd` — group leader fd or `-1`
- `unsigned long flags` — `PERF_FLAG_FD_NO_GROUP` / `PERF_FLAG_FD_OUTPUT` / `PERF_FLAG_PID_CGROUP` / `PERF_FLAG_FD_CLOEXEC`

Return: file descriptor (mappable + ioctl-controllable + readable + epoll-able).

### `struct perf_event_attr` layout

Every field offset frozen — `perf-tools-perf-trace`-style consumers parse this struct directly.
- `type`: `PERF_TYPE_HARDWARE` / `_SOFTWARE` / `_TRACEPOINT` / `_HW_CACHE` / `_RAW` / `_BREAKPOINT` / dynamic PMU type ids registered under `/sys/bus/event_source/devices/<pmu>/type`
- `size`: ABI versioning; current is `PERF_ATTR_SIZE_VER8`
- `config` / `config1` / `config2` / `config3`: vendor encoding
- `sample_period` / `sample_freq` / `freq` bit
- `sample_type` bitmask: `PERF_SAMPLE_IP / TID / TIME / ADDR / READ / CALLCHAIN / ID / CPU / PERIOD / STREAM_ID / RAW / BRANCH_STACK / REGS_USER / STACK_USER / WEIGHT / DATA_SRC / IDENTIFIER / TRANSACTION / REGS_INTR / PHYS_ADDR / AUX / CGROUP / DATA_PAGE_SIZE / CODE_PAGE_SIZE / WEIGHT_STRUCT`
- `read_format` bitmask
- Boolean bits: `disabled / inherit / pinned / exclusive / exclude_user / exclude_kernel / exclude_hv / exclude_idle / mmap / comm / freq / inherit_stat / enable_on_exec / task / watermark / precise_ip:2 / mmap_data / sample_id_all / exclude_host / exclude_guest / exclude_callchain_kernel / exclude_callchain_user / mmap2 / comm_exec / use_clockid / context_switch / write_backward / namespaces / ksymbol / bpf_event / aux_output / cgroup / text_poke / build_id / inherit_thread / remove_on_exec / sigtrap / aux_action / aux_start_paused / aux_pause / aux_resume`
- `wakeup_events` / `wakeup_watermark`
- `bp_type` / `bp_addr` / `bp_len` (for breakpoints)
- `branch_sample_type` (LBR / branch-stack filtering)
- `sample_regs_user` / `sample_stack_user` / `clockid` / `sample_regs_intr`
- `aux_watermark` / `sample_max_stack` / `aux_sample_size` / `aux_action`
- `sig_data` (when sigtrap=1)
- `config3`

Wire format byte-identical including padding bytes.

### perf-event ioctls

`ioctl(perf_fd, PERF_EVENT_IOC_*, arg)`:
- `_ENABLE` / `_DISABLE` / `_REFRESH(n)` / `_RESET`
- `_PERIOD` (set sample_period) / `_SET_OUTPUT(fd)` / `_SET_FILTER(str)` / `_ID(*id)` / `_SET_BPF(bpf_fd)` / `_PAUSE_OUTPUT` / `_QUERY_BPF` / `_MODIFY_ATTRIBUTES` / `_PAUSE_AUX(fd)`

Numbers + arg semantics byte-identical.

### Read format

`read(perf_fd, buf, sz)` returns one of three layouts based on `attr.read_format`:
- Single-event: `value [, time_enabled] [, time_running] [, id]`
- Group: `nr [, time_enabled] [, time_running] {value [, id]} * nr` (PERF_FORMAT_GROUP)
- Group with PERF_FORMAT_LOST: includes `lost` field

Layouts byte-identical so `perf-stat` etc. parse correctly.

### mmap'd ring buffer layout

Userspace `mmap(NULL, (1+2^n) * page_size, PROT_READ|PROT_WRITE, MAP_SHARED, perf_fd, 0)`:
- Page 0: `struct perf_event_mmap_page` (control header — `data_head`, `data_tail`, `data_offset`, `data_size`, `aux_head`, `aux_tail`, `aux_offset`, `aux_size`, `time_enabled`, `time_running`, `cap_*` self-monitoring caps for vDSO-style fast-read of counters)
- Pages 1..2^n: ring of `struct perf_event_header` records (`PERF_RECORD_MMAP / LOST / COMM / EXIT / THROTTLE / UNTHROTTLE / FORK / READ / SAMPLE / MMAP2 / AUX / ITRACE_START / LOST_SAMPLES / SWITCH / SWITCH_CPU_WIDE / NAMESPACES / KSYMBOL / BPF_EVENT / CGROUP / TEXT_POKE / AUX_OUTPUT_HW_ID`)
- AUX area (separately mmap'd at `aux_offset`): high-bandwidth payload (Intel PT raw branch trace, ARM CoreSight ETM stream)

Layout + record format byte-identical.

### tracefs probe-point frontends

`/sys/kernel/tracing/kprobe_events` writable: write `p[:GRP/EVENT] [MOD:]SYM[+offs] [FETCHARGS]` to add a kprobe; readable: enumerates added probes. Same for `uprobe_events`. Format byte-identical so `perf probe` + `bpftrace` + `bcc-tools` all work.

### `/sys/bus/event_source/devices/<pmu>/`

Per-PMU sysfs directory exposes:
- `type` (numeric `PERF_TYPE_*` id assigned to this PMU)
- `format/<field>` (per-attr-config bit field encoding — used by `perf stat -e <pmu>/event=0x...,umask=0x.../`)
- `events/<name>` (named pre-canned event encodings — used by `perf stat -e <pmu>/<name>/`)
- `cpus` / `cpumask` (for uncore PMUs that bind to specific CPUs)
- `caps/` (per-PMU capability bits)
- `nr_addr_filters` (for AUX-area PMUs that support address filters)
- Per-PMU specific files (e.g., `branches` for LBR, `pt_filter` for Intel-PT)

Layout + content byte-identical.

### `/proc/sys/kernel/perf_*` sysctls

- `perf_event_paranoid` (-1/0/1/2/3 — controls non-root access)
- `perf_event_max_sample_rate`
- `perf_cpu_time_max_percent`
- `perf_event_mlock_kb`
- `perf_event_max_stack`
- `perf_event_max_contexts_per_stack`

Wire-format identical (default values = upstream defaults).

### `/proc/<pid>/perf_event_paranoid` (per-task override) — N/A

(Upstream has only the system-wide sysctl; Rookery does not add a per-task override.)

### NMI watchdog (perf-event-backed)

`/proc/sys/kernel/{nmi_watchdog,soft_watchdog,watchdog_thresh,watchdog_cpumask}` etc. — the hard-lockup detector is a perf-event PMU consumer (cycles event with sample_period = watchdog_thresh × cpu_freq, sets PMU NMI handler). Wire format identical so `nmi_watchdog=0` and friends behave the same.

### sigtrap-on-sample (CONFIG_PERF_EVENTS, attr.sigtrap=1)

Per-task perf event with `sigtrap=1` + `remove_on_exec=1` delivers SIGTRAP synchronously on sample with `si_perf_data = sig_data`. Used by GWP-ASan-style userspace allocators + AddressSanitizer. Behavior identical.

### bpf-event linkage

`PERF_EVENT_IOC_SET_BPF(bpf_prog_fd)` attaches a BPF program to fire on each sample / probe-trigger. Consumes from kernel/bpf side (cross-ref `kernel/bpf/00-overview.md`). Wire-format identical (`bpf_prog_fd` is just an `int`; the binding uses bpf-link infrastructure).

### Tracepoint-id mapping

`/sys/kernel/tracing/events/<group>/<event>/id` returns the numeric tracepoint id consumed as `attr.config` when `attr.type == PERF_TYPE_TRACEPOINT`. Mapping persistence + numeric stability documented in `kernel/trace/00-overview.md`; perf depends on it.

## Tier-3 docs governed by this Tier-2

(Phase D will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `kernel/perf/perf-event-core.md` | `kernel/events/core.c`: `perf_event_open(2)` + per-cpu/per-task ctx + scheduling onto PMUs + group/multiplex |
| `kernel/perf/ring-buffer.md` | `kernel/events/ring_buffer.c`: mmap'd ring + AUX area + record emission + lost-record accounting |
| `kernel/perf/callchain.md` | `kernel/events/callchain.c`: kernel/user stack unwinder dispatch |
| `kernel/perf/hw-breakpoint.md` | `kernel/events/hw_breakpoint.c`: generic debug-register allocator |
| `kernel/perf/uprobes.md` | `kernel/events/uprobes.c`: inode-based user probe + xol-area + single-step |
| `kernel/perf/trace-kprobe.md` | `kernel/trace/trace_kprobe.c`: tracefs kprobe frontend |
| `kernel/perf/trace-uprobe.md` | `kernel/trace/trace_uprobe.c`: tracefs uprobe frontend |
| `kernel/perf/x86-pmu-core.md` | `arch/x86/events/core.c`: x86 PMU dispatch (Intel + AMD common) |
| `kernel/perf/x86-pmu-intel.md` | `arch/x86/events/intel/core.c` + helpers (cstate, lbr, ds, p4, p6, knc) |
| `kernel/perf/x86-pmu-intel-pt.md` | `arch/x86/events/intel/pt.c`: Intel Processor Trace (AUX-area sink) |
| `kernel/perf/x86-pmu-intel-bts.md` | `arch/x86/events/intel/bts.c`: Intel Branch Trace Store |
| `kernel/perf/x86-pmu-intel-uncore.md` | `arch/x86/events/intel/uncore*.c`: Intel uncore PMUs |
| `kernel/perf/x86-pmu-amd.md` | `arch/x86/events/amd/{core,ibs,lbr,power,uncore,iommu}.c`: AMD PMUs |
| `kernel/perf/x86-pmu-msr.md` | `arch/x86/events/msr.c`: MSR-based pseudo-PMU |

## Compatibility outline (top-level)

- REQ-O1: `perf_event_open(2)` syscall number + ABI byte-identical (struct + ioctls + return-fd semantics).
- REQ-O2: `struct perf_event_attr` layout + `attr->size` versioning + every documented field offset byte-identical (PERF_ATTR_SIZE_VER8 is the current high-water-mark).
- REQ-O3: All `PERF_RECORD_*` mmap-ring record formats byte-identical (perf-tools / bpftrace / bcc-tools / pyperf consume unchanged).
- REQ-O4: All `PERF_TYPE_*`, `PERF_COUNT_HW_*`, `PERF_COUNT_SW_*`, `PERF_COUNT_HW_CACHE_*` enum values byte-identical.
- REQ-O5: `/sys/bus/event_source/devices/<pmu>/` per-PMU sysfs surface byte-identical (per-PMU `type`, `format/`, `events/`, `cpus`, `caps/`, `nr_addr_filters`).
- REQ-O6: `tracefs` `/sys/kernel/tracing/{kprobe,uprobe}_events` wire format byte-identical.
- REQ-O7: `/proc/sys/kernel/perf_*` sysctls + default values byte-identical.
- REQ-O8: NMI hard-lockup-detector backed by perf-event identically (`watchdog_thresh` knob behaves the same).
- REQ-O9: sigtrap-on-sample semantics identical (GWP-ASan + ASan-style allocators work unchanged).
- REQ-O10: BPF program attach via `PERF_EVENT_IOC_SET_BPF` identical.
- REQ-O11: PMU group / multiplex / inherit semantics identical (perf stat -B group counters round-trip-equivalent).
- REQ-O12: TLA+ models declared at this Tier-2 (ring-buffer producer-consumer, event scheduler PMU assignment, hw-breakpoint allocator).
- REQ-O13: Verus/Creusot Layer-4 functional contracts on perf_event refcount + ring-buffer head/tail mmap-shared invariant.
- REQ-O14: Hardening: row-1 features applied per `00-security-principles.md`, with perf-specific reinforcement (perf_event_paranoid default-2, fortified callchain-unwinder, bounds-checked ring-buffer reservations).

## Acceptance Criteria (top-level)

- [ ] AC-O1: `perf list` shows the same PMU events + descriptors as upstream baseline. (covers REQ-O4, REQ-O5)
- [ ] AC-O2: `perf stat -e cycles,instructions,branches,branch-misses /bin/ls` produces results matching upstream within 1% noise. (covers REQ-O1, REQ-O2, REQ-O11)
- [ ] AC-O3: `perf record -F 99 -ag sleep 5 && perf report` decodes records correctly. (covers REQ-O3)
- [ ] AC-O4: `perf trace -e syscalls` correctly traces syscall enter/exit. (covers REQ-O3, REQ-O6)
- [ ] AC-O5: `perf probe --add 'do_sys_open filename:string'` adds a kprobe; `perf record -e probe:do_sys_open` captures samples. (covers REQ-O6)
- [ ] AC-O6: `bpftrace -e 'tracepoint:syscalls:sys_enter_openat { @[comm] = count(); }'` works. (covers REQ-O6, REQ-O10)
- [ ] AC-O7: `perf record -e intel_pt//u -- /bin/ls` captures Intel-PT trace; `perf script` decodes. (covers REQ-O3 AUX area)
- [ ] AC-O8: `cat /proc/sys/kernel/perf_event_paranoid` default value matches upstream. (covers REQ-O7)
- [ ] AC-O9: Hard-lockup test (NMI watchdog) fires within `watchdog_thresh` seconds when CPU is wedged. (covers REQ-O8)
- [ ] AC-O10: GWP-ASan-style sigtrap test program receives SIGTRAP at the correct sample IP. (covers REQ-O9)
- [ ] AC-O11: kselftest `tools/testing/selftests/perf_events/` passes. (covers REQ-O11, REQ-O12)
- [ ] AC-O12: kselftest `tools/testing/selftests/breakpoints/` passes. (covers REQ-O12)

## Verification (top-level)

### Layer 2: TLA+ models — mandatory list

| Model | Owned by |
|---|---|
| `models/perf/ring_buffer.tla` | `kernel/perf/ring-buffer.md` (proves: mmap'd `data_head` / `data_tail` producer-consumer with kernel writer + userspace reader observes acquire-release ordering; lost-record accounting correctly bounded; AUX head/tail independent of data head/tail) |
| `models/perf/event_scheduler.tla` | `kernel/perf/perf-event-core.md` (proves: per-cpu PMU event-scheduling under counter-pressure: pinned events always run when on, exclusive events block others, group events all-or-nothing, inheritance into child task preserves attr) |
| `models/perf/hw_breakpoint_alloc.tla` | `kernel/perf/hw-breakpoint.md` (proves: per-CPU debug-register slot allocator + reservation never double-allocates a slot under concurrent ptrace + perf + kgdb consumers; per-task slot count + per-cpu pinned slot count correctly maintained) |

### Layer 4: Functional contracts (Verus / Creusot)

| Component | Contract topic |
|---|---|
| `kernel/perf/perf-event-core.md` | `perf_event` refcount + close-during-mmap safety: ring-buffer mmap holds a reference; close while mapped doesn't free the event |
| `kernel/perf/ring-buffer.md` | `data_head` advance pre/postcondition: reservation always returns a contiguous slice not crossing the wrap boundary; lost counter incremented exactly once when reservation fails |
| `kernel/perf/uprobes.md` | xol-area allocator invariant: per-task xol slot never double-issued; single-step always returns to the original instruction at original IP+len |

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Default inherited from |
|---|---|
| **REFCOUNT** | per-event + per-event-context + per-pmu refcounts use `Refcount` (saturating) | § Mandatory |
| **CONSTIFY** | per-PMU `struct pmu` ops vtable + per-arch `x86_pmu` desc `static const` | § Mandatory |
| **SIZE_OVERFLOW** | ring-buffer reservation (size + alignment), sample-stack-user copy size, callchain depth use checked operators | § Mandatory |
| **RAP / FORTIFY_RW** | per-PMU vtables read-only-after-init | § Mandatory |
| **MEMORY_SANITIZE** | freed perf_event state cleared (carries per-cgroup attach + bpf prog ptr) | § Default-on configurable off |
| **STRICT_KERNEL_RWX** | uprobe trampoline + xol-area W^X enforced (xol pages remain RX after instruction install; mw via dedicated remap) | § Mandatory |

### Row-2 (LSM-stackable) features for perf

LSM hooks called: `security_perf_event_open` (on syscall), `security_perf_event_alloc`, `security_perf_event_read`, `security_perf_event_write`. SELinux + AppArmor + future GR-RBAC stackable LSM each get a chance to mediate.

GR-RBAC adds:
- Per-role disallow of `perf_event_open` entirely (most service accounts).
- Per-role disallow of kernel-mode sampling (`exclude_kernel=0` requires explicit cap).
- Per-role disallow of `PERF_TYPE_TRACEPOINT` (denies bpf-via-perf trace prog attach).
- Per-role disallow of `PERF_TYPE_BREAKPOINT` (denies hw-breakpoint allocation).
- Per-role audit of every `perf_event_open` with the resolved pmu + attr summary.

### perf-specific reinforcement

- **`perf_event_paranoid` default = 2** in Rookery (upstream default = 4 since 5.8 / hardening series; Rookery matches the more-restrictive default).
- **Callchain unwinder bounds**: `perf_event_max_stack` capped + verified (kernel + user stack walk MUST NOT recurse past limit; ORC unwinder validated against return-address sled).
- **Ring-buffer reservation**: every `perf_aux_output_begin` / `perf_output_begin` validates `aux_head` / `data_head` arithmetic doesn't wrap past `aux_size` / `data_size` (defense against userspace writing bogus tail).
- **mmap-page `cap_*` fast-read**: when self-monitoring fast-read is enabled (`cap_user_rdpmc`), the offset+multiplier+shift fields published to userspace are validated; userspace cannot mis-decode and trick the kernel into reading wrong MSR (it's a one-way publish).
- **uprobes inode-based probe-point**: cross-namespace probe attach denied unless CAP_SYS_ADMIN in init userns.

## Open Questions

(none at Tier-2 — defer to per-Tier-3 once Phase D begins; expected hot questions: Intel-PT cpuid-min-version pin, perf_event_paranoid distro-config interaction, whether to keep deprecated `PERF_SAMPLE_STACK_USER` size cap default, sched_clock vs perf_clock for `clockid=CLOCK_MONOTONIC` event timestamps)

## Out of Scope

- Per-Tier-3 (Phase D)
- ARM / RISC-V / s390 / PPC / MIPS / LoongArch arch-glue (`arch/arm64/events/` etc. — out of v0)
- `drivers/perf/` SoC PMU drivers (separate Tier-2 wrapper future)
- `kernel/trace/` proper (separate Tier-2 wrapper future — only the `trace_{k,u}probe.c` frontends are in-scope here as perf consumers)
- 32-bit-only paths
- Implementation code
