# Subsystem: kernel/ — kernel core (sched, locking, time, ipc, bpf, …)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - kernel/
  - kernel/sched/
  - kernel/locking/
  - kernel/time/
  - kernel/rcu/
  - kernel/irq/
  - kernel/cgroup/
  - kernel/bpf/
  - kernel/futex/
  - kernel/events/
  - kernel/trace/
  - kernel/module/
  - kernel/printk/
  - kernel/power/
  - kernel/entry/
  - kernel/dma/
  - kernel/livepatch/
  - kernel/liveupdate/
  - kernel/debug/
  - kernel/unwind/
  - kernel/kcsan/
  - kernel/gcov/
  - include/linux/sched.h
  - include/linux/spinlock.h
  - include/linux/mutex.h
  - include/linux/rcupdate.h
  - include/linux/wait.h
  - include/linux/jiffies.h
  - include/linux/signal.h
  - include/linux/cgroup.h
  - include/linux/bpf.h
  - include/linux/perf_event.h
  - include/linux/futex.h
  - include/linux/module.h
  - include/uapi/linux/bpf.h
  - include/uapi/linux/perf_event.h
  - include/uapi/linux/futex.h
-->

## Summary
Tier-2 overview for `kernel/` — the kernel core. Owns task lifecycle (fork, exit, exec, signal, kthread), the scheduler family (CFS/EEVDF, RT, deadline, ext, idle), locking primitives (mutex, spinlock, rwsem, RCU, futex), timekeeping and timers (jiffies, hrtimer, posix-timer, alarmtimer), IRQ infrastructure, the cgroup framework, BPF (verifier + JIT-driven programs), perf events, ftrace + tracing, kernel modules, printk, power management orchestration, panic/oops/BUG, and the cross-architecture syscall-entry helpers consumed by `arch/x86/entry/`.

This is the largest single Tier-2 subsystem by surface area: 106 top-level C files plus 22 subdirectories. The Tier-3 docs spawned here cover the components individually; this overview is the structural map.

## Upstream references in scope

`kernel/` (106 top-level + 22 subdirs at baseline). Mapping to Tier-3 docs:

| Category | Upstream paths (selection) | Planned Tier-3 doc |
|---|---|---|
| Scheduler family | `kernel/sched/` (core, fair (CFS/EEVDF), rt, deadline, ext (sched_ext), idle, autogroup, isolation, loadavg, membarrier, pelt, topology, cpufreq, cpufreq_schedutil, completion, clock, debug) | `sched/00-overview.md` (Tier 3 spawning further docs: `cfs.md`, `rt.md`, `deadline.md`, `ext.md`, `idle.md`, `pelt.md`, `topology.md`, `clock.md`, `cpufreq.md`) |
| Locking primitives + lockdep | `kernel/locking/` (mutex, qspinlock, qrwlock, percpu-rwsem, osq_lock, lockdep, locktorture) | `locking/00-overview.md` (Tier 3 spawning: `mutex.md`, `spinlock.md`, `rwsem.md`, `seqlock.md`, `lockdep.md`) |
| RCU | `kernel/rcu/` (tree, tasks, srcu, sync, ref-scale, scale, torture) | `locking/rcu.md` (under locking subtree) |
| Time, timers, clocks | `kernel/time/` (timekeeping, hrtimer, alarmtimer, posix-{cpu,clock,timers}, jiffies, ntp, sched_clock, namespace, namespace_vdso, tick-*) | `time/00-overview.md` (Tier 3 spawning: `hrtimer.md`, `clocksource.md`, `tick.md`, `posix-timers.md`, `namespace.md`) |
| Task lifecycle (fork/exit/exec/signal) + kthread + cred | `kernel/fork.c`, `kernel/exit.c`, `kernel/signal.c`, `kernel/exec_domain.c`, `kernel/kthread.c`, `kernel/cred.c`, `kernel/groups.c`, `kernel/capability.c`, `kernel/user.c`, `kernel/user_namespace.c`, `kernel/pid.c`, `kernel/pid_namespace.c`, `kernel/nsproxy.c`, `kernel/ucount.c` | `task-lifecycle.md` |
| Futex | `kernel/futex/` | `futex.md` |
| IRQ infrastructure (cross-arch) | `kernel/irq/` (chip, descriptor, generic-chip, manage, msi, devres, proc, resend, settings, spurious, debugfs, ipi, irqdomain, autoprobe, matrix, migration, pm, timings) | `irq.md` |
| Workqueue, async work, irq_work | `kernel/workqueue.c`, `kernel/async.c`, `kernel/irq_work.c`, `kernel/stop_machine.c` | `workqueue.md` |
| Smp / cpu hotplug | `kernel/smp.c`, `kernel/smpboot.c`, `kernel/cpu.c` (the cross-arch portions; arch-specific bringup is in `arch/x86/`) | `smp-hotplug.md` |
| Cgroup framework + controllers | `kernel/cgroup/` (cgroup, cgroup-v1, cpuset, cpuset-v1, freezer, legacy_freezer, namespace, pids, rdma, dmem, debug, misc) | `cgroup/00-overview.md` (Tier 3 spawning: `core.md`, `cpuset.md`, `pids.md`, `freezer.md`) |
| BPF (verifier + JIT-driven program runtime) | `kernel/bpf/` (65+ files: core, verifier, syscall, hashtab, arraymap, lpm_trie, ringbuf, queue_stack_maps, btf, bloom_filter, struct_ops, lsm, …) | `bpf/00-overview.md` (Tier 3 spawning: `verifier.md`, `program-types.md`, `maps.md`, `btf.md`, `lsm.md`, `iter.md`, `helpers.md`, `arena.md`, `struct-ops.md`) — note: x86 BPF JIT lives in `arch/x86/00-overview.md` § bpf-jit.md |
| Perf events | `kernel/events/` (core, ring_buffer, hw_breakpoint, callchain, internal.h) | `perf-events.md` |
| Tracing (ftrace, blktrace, etc.) | `kernel/trace/` (~80 files; ftrace core, ring_buffer, trace, trace_events, kprobe, uprobe, blktrace, fgraph, …) | `trace/00-overview.md` (Tier 3 spawning: `ftrace.md`, `tracepoints.md`, `kprobes.md`, `uprobes.md`, `blktrace.md`, `eventfs.md`) |
| Module loading | `kernel/module/` (main, decompress, internal, kdb, kallsyms, livepatch, signing, sysfs, tracking, tree_lookup, version) | `module-loading.md` |
| Printk + kernel logging | `kernel/printk/` (printk, printk_safe, printk_ringbuffer, internal.h, console_cmdline, nbcon) | `printk.md` |
| Power management orchestration (cross-arch portions) | `kernel/power/` (main, suspend, hibernate, snapshot, swap, user, qos, autosleep, console, energy_model, sleeptime, wakelock) | `power-mgmt.md` (cross-references `arch/x86/00-overview.md` § power.md) |
| Cross-arch syscall entry helpers | `kernel/entry/` (common, kvm, syscall_user_dispatch) — generic helpers consumed by every `arch/<arch>/entry/` | `syscall-entry-helpers.md` |
| DMA mapping API | `kernel/dma/` (mapping, debug, contiguous, swiotlb, coherent, direct, ops_helpers, pool, remap) | `dma-mapping.md` |
| Live kernel patching | `kernel/livepatch/` | `livepatch.md` |
| Live kernel update (newer; v0 may defer) | `kernel/liveupdate/` | `liveupdate.md` (deferred candidate; see Q3) |
| Debugger / KGDB | `kernel/debug/` (debug_core, gdbstub, kdb) | `kgdb.md` |
| Stack unwinder | `kernel/unwind/` | `unwind.md` |
| KCSAN | `kernel/kcsan/` | folded into `mm/sanitizers.md` (cross-ref) — kernel/kcsan IS the substrate, mm-side hookup lives there |
| Code coverage (gcov, kcov) | `kernel/gcov/`, `kernel/kcov.c` | `coverage.md` |
| Audit | `kernel/audit*.c`, `kernel/auditsc.c`, `kernel/audit_*.c`, `kernel/audit_fsnotify.c`, `kernel/audit_tree.c`, `kernel/audit_watch.c`, `kernel/auditfilter.c` | `audit.md` (cross-references `security/00-overview.md`) |
| Crash / kexec / kdump | `kernel/kexec.c`, `kernel/kexec_core.c`, `kernel/kexec_elf.c`, `kernel/kexec_file.c`, `kernel/crash_core.c`, `kernel/crash_dump_dm_crypt.c`, `kernel/crash_reserve.c`, `kernel/elfcorehdr.c` | `crash-kexec.md` (x86-arch portion cross-referenced from `arch/x86/00-overview.md` § boot.md) |
| Panic + BUG | `kernel/panic.c`, `kernel/sys.c` (reboot/halt), `kernel/reboot.c`, `kernel/hung_task.c`, `kernel/watchdog.c` | `panic-reboot.md` |
| Configuration / sysctl | `kernel/sysctl.c`, `kernel/configs.c`, `kernel/configs/`, `kernel/ksysfs.c` | `sysctl-ksysfs.md` |
| Kallsyms + symbol resolution | `kernel/kallsyms.c`, `kernel/ksyms_common.c`, `kernel/kallsyms_selftest.c` | `kallsyms.md` |
| Kprobes (substrate; consumers in `kernel/trace/`) | `kernel/kprobes.c` | covered in `trace/kprobes.md` |
| Function tracer infra (CFI, jump_label, alternative dispatch) | `kernel/cfi.c`, `kernel/jump_label.c`, `kernel/extable.c`, `kernel/static_call.c` (if present), `kernel/static_call_inline.c` (if present) | `runtime-codepatching.md` |
| RSEQ | `kernel/rseq.c` | folded into `task-lifecycle.md` |
| Resource accounting | `kernel/acct.c`, `kernel/delayacct.c`, `kernel/cpu_pm.c` | `accounting.md` |
| Misc (groups, kcmp, kheaders, fail_function, freezer, iomem, dma.c) | `kernel/groups.c` (folded into task-lifecycle), `kernel/kcmp.c`, `kernel/kheaders.c`, `kernel/fail_function.c`, `kernel/freezer.c`, `kernel/iomem.c`, `kernel/dma.c` | various Tier-3 docs adopt these |
| Test fixtures | `kernel/backtracetest.c`, `kernel/crash_core_test.c`, `kernel/kallsyms_selftest.c`, `kernel/torture.c` | (not separately documented; tests track their components) |

## Compatibility contract

kernel/ owns a substantial slice of the userspace-visible compat surface:

### Syscall surface

Most of the syscall surface beyond mm/ touches kernel/. Categorized:

| Category | Syscalls | Tier-3 owner |
|---|---|---|
| Process lifecycle | `fork`, `vfork`, `clone`, `clone3`, `exec*`, `execve`, `execveat`, `exit`, `exit_group`, `wait4`, `waitid`, `setpgid`, `getpgid`, `setsid`, `getsid`, `setrlimit`, `getrlimit`, `prlimit64`, `prctl`, `arch_prctl`, `set_tid_address`, `gettid`, `getpid`, `getppid`, `getpgrp`, `setns` | `task-lifecycle.md` |
| Signals | `kill`, `tkill`, `tgkill`, `pidfd_send_signal`, `rt_sigaction`, `rt_sigprocmask`, `rt_sigpending`, `rt_sigqueueinfo`, `rt_sigsuspend`, `rt_sigreturn`, `rt_sigtimedwait`, `sigaltstack`, `signalfd`, `signalfd4`, `pause`, `pidfd_open` | `task-lifecycle.md` (signal subsection) |
| Credentials | `setuid`, `seteuid`, `setresuid`, `setreuid`, `getuid`, `geteuid`, `getresuid`, `setgid`, `setegid`, `setresgid`, `setregid`, `getgid`, `getegid`, `getresgid`, `setgroups`, `getgroups`, `setfsuid`, `setfsgid`, `capset`, `capget` | `task-lifecycle.md` (cred subsection) |
| Scheduler | `sched_setaffinity`, `sched_getaffinity`, `sched_setscheduler`, `sched_getscheduler`, `sched_setparam`, `sched_getparam`, `sched_setattr`, `sched_getattr`, `sched_yield`, `sched_get_priority_min/max`, `sched_rr_get_interval`, `sched_get_priority_min/max`, `nice`, `setpriority`, `getpriority`, `sched_get_period`, `sched_get_runtime` | `sched/00-overview.md` |
| Time | `gettimeofday`, `settimeofday`, `time`, `stime`, `nanosleep`, `clock_nanosleep`, `clock_gettime`, `clock_settime`, `clock_getres`, `clock_adjtime`, `adjtimex`, `timer_create`, `timer_delete`, `timer_settime`, `timer_gettime`, `timer_getoverrun`, `timerfd_create`, `timerfd_settime`, `timerfd_gettime`, `times`, `getitimer`, `setitimer`, `alarm` | `time/00-overview.md` |
| Futex | `futex`, `futex_wait`, `futex_wake`, `futex_waitv`, `futex_requeue`, `set_robust_list`, `get_robust_list` | `futex.md` |
| Cgroup (some via syscall, most via /sys/fs/cgroup) | `setns`, `unshare` (with NS flags) | `cgroup/00-overview.md` |
| BPF | `bpf` (and its `BPF_*` cmd subcommands) | `bpf/00-overview.md` |
| Perf | `perf_event_open` | `perf-events.md` |
| Tracing | `process_vm_readv`, `process_vm_writev`, `ptrace`, `kcmp` | `trace/00-overview.md` (ptrace-side); `kallsyms.md` (kcmp) |
| Module | `init_module`, `finit_module`, `delete_module` | `module-loading.md` |
| Printk / klog | `syslog` | `printk.md` |
| Power | `reboot`, `kexec_load`, `kexec_file_load` | `panic-reboot.md`, `crash-kexec.md` |
| Audit | (audit syscalls go through netlink, not direct syscall) | `audit.md` |
| Misc | `seccomp`, `prctl(PR_SET_SECCOMP)`, `landlock_*`, `pidfd_getfd`, `process_madvise`, `process_mrelease`, `userfaultfd` (covered in `mm/00-overview.md`) | various |

### `/proc` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/proc/<pid>/stat`, `/proc/<pid>/status`, `/proc/<pid>/cmdline`, `/proc/<pid>/environ`, `/proc/<pid>/exe`, `/proc/<pid>/cwd`, `/proc/<pid>/root`, `/proc/<pid>/fd/`, `/proc/<pid>/fdinfo/`, `/proc/<pid>/comm`, `/proc/<pid>/wchan`, `/proc/<pid>/syscall`, `/proc/<pid>/stack`, `/proc/<pid>/io`, `/proc/<pid>/limits`, `/proc/<pid>/cgroup`, `/proc/<pid>/sessionid`, `/proc/<pid>/personality`, `/proc/<pid>/ns/*`, `/proc/<pid>/sched`, `/proc/<pid>/schedstat`, `/proc/<pid>/timers`, `/proc/<pid>/timerslack_ns`, `/proc/<pid>/setgroups`, `/proc/<pid>/uid_map`, `/proc/<pid>/gid_map`, `/proc/<pid>/projid_map`, `/proc/<pid>/loginuid`, `/proc/<pid>/oom_*` (mm-owned) | `task-lifecycle.md`; per-area Tier-3 docs claim subfields | Format-identical |
| `/proc/<pid>/task/<tid>/...` | `task-lifecycle.md` | Mirrors per-pid layout |
| `/proc/sched_debug`, `/proc/schedstat`, `/proc/loadavg` | `sched/00-overview.md` | Format-identical |
| `/proc/timer_list`, `/proc/timer_stats` | `time/00-overview.md` | Format-identical |
| `/proc/locks`, `/proc/kpagecount`, `/proc/kpageflags`, `/proc/kpagecgroup` (mm-owned but referenced from kernel/-side cgroup tracking) | `task-lifecycle.md`, `cgroup/00-overview.md` | Format-identical |
| `/proc/interrupts`, `/proc/softirqs`, `/proc/irq/<n>/{affinity,smp_affinity,smp_affinity_list,...}` | `irq.md` | Format-identical |
| `/proc/cpuinfo` (per-arch fields owned by `arch/x86/cpuinfo.md`; common fields here) | `arch/x86/cpuinfo.md` (delegate) | Identical |
| `/proc/{stat,uptime,loadavg}` | `task-lifecycle.md`, `sched/00-overview.md`, `time/00-overview.md` | Identical |
| `/proc/sys/kernel/*` (sysctl knobs: `pid_max`, `threads-max`, `randomize_va_space`, `core_pattern`, `kptr_restrict`, `dmesg_restrict`, `panic_*`, `printk_ratelimit*`, `unprivileged_bpf_disabled`, `bpf_stats_enabled`, hundreds more) | `sysctl-ksysfs.md`; per-area Tier-3 docs claim subfields | Identical |
| `/proc/sys/debug/*` | `mm-debug.md` (mostly), `sysctl-ksysfs.md` | Identical |
| `/proc/sys/fs/*` | `fs/00-overview.md` (Phase B; this overview cross-references) | Identical |
| `/proc/{filesystems,partitions,mounts,modules,kallsyms,kcore,iomem,ioports,buddyinfo,cmdline,version,vmstat,zoneinfo,slabinfo,interrupts,meminfo,cpuinfo,uptime,stat,loadavg,locks,misc,devices,diskstats,scsi/...}` | various Tier-3 docs; mm-owned ones in `mm/00-overview.md` | Identical |
| `/proc/sys/kernel/random/*`, `/proc/sys/kernel/keys/*` | `random.md` (in `crypto/00-overview.md`); `keys.md` (in `security/00-overview.md`) | Identical |

### `/sys/kernel/*` surfaces

| Path | Owner doc | Compat level |
|---|---|---|
| `/sys/kernel/debug/tracing/*` (ftrace) | `trace/00-overview.md` | Identical |
| `/sys/kernel/tracing/*` (modern) | `trace/00-overview.md` | Identical |
| `/sys/kernel/debug/sched_debug` | `sched/00-overview.md` | Identical |
| `/sys/kernel/debug/cgroup/*` | `cgroup/00-overview.md` | Identical |
| `/sys/kernel/debug/dma-buf/*` | `dma-mapping.md` cross-ref | Identical |
| `/sys/kernel/debug/dynamic_debug/*` (lib-owned) | `lib/debug-tooling.md` (delegate) | Identical |
| `/sys/kernel/debug/lockdep/*` | `locking/00-overview.md` | Identical |
| `/sys/kernel/debug/printk/*` | `printk.md` | Identical |
| `/sys/kernel/livepatch/*` | `livepatch.md` | Identical |
| `/sys/kernel/cgroup/*` | `cgroup/00-overview.md` | Identical |
| `/sys/kernel/btf/vmlinux` | `bpf/00-overview.md` | Identical (BTF binary format) |
| `/sys/kernel/security/*` | `security/00-overview.md` (Phase B) | Identical |
| `/sys/devices/system/cpu/cpu<N>/{online,topology,...}` | `smp-hotplug.md`; arch-specific cpuinfo in `arch/x86/cpuinfo.md` | Identical |
| `/sys/devices/system/cpu/vulnerabilities/*` | `arch/x86/cpu-mitigations.md` (delegate) | Identical |
| `/sys/fs/cgroup/*` (the cgroupfs mount point) | `cgroup/00-overview.md` | Identical |
| `/sys/fs/bpf/*` (bpf pin filesystem) | `bpf/00-overview.md` | Identical |

### Other userspace-visible interfaces

- **Kernel command line parsing** (`Documentation/admin-guide/kernel-parameters.txt`) — every documented parameter must be parsed identically. Owned by per-Tier-3 doc plus `init/00-overview.md` (Phase B).
- **`/proc/kallsyms` symbol format and `/proc/kcore` ELF layout** — preserved.
- **BTF (BPF Type Format) `/sys/kernel/btf/vmlinux`** — binary-identical layout to upstream so existing CO-RE BPF programs work unchanged.
- **dmesg output prefixes** (loglevel, timestamps, subsystem tags) — preserved to keep journald and existing log-parsing tools working.
- **netlink protocol families managed by kernel/ (audit netlink, perf netlink-equivalent)** — wire-format identical.

## Requirements

- REQ-1: Every syscall under kernel/'s purview is implemented byte-identically with upstream — entry/exit conventions, register usage, errno returns, struct layouts.
- REQ-2: Every `/proc` and `/sys` path listed above produces format-identical content for equivalent kernel state.
- REQ-3: Scheduler behavior (CFS/EEVDF nice→weight mapping, runqueue ordering, preemption points, RT priorities, deadline policies) preserves upstream semantics; reference workloads exhibit identical scheduling outcomes.
- REQ-4: Locking primitive observable behavior (mutex contention path, spinlock fairness, RCU grace periods, lockdep ordering reports) matches upstream.
- REQ-5: Time-of-day, monotonic clock, and posix-timer semantics match upstream — including leap-second handling, NTP discipline, and `/dev/ptp*` interaction.
- REQ-6: futex semantics — `FUTEX_WAIT`, `FUTEX_WAKE`, `FUTEX_REQUEUE`, `FUTEX_CMP_REQUEUE`, `FUTEX_LOCK_PI`, `FUTEX_UNLOCK_PI`, `FUTEX_TRYLOCK_PI`, robust list, priority-inheritance — match upstream byte-for-byte.
- REQ-7: cgroup v1 and v2 controllers (cpu, cpuset, cpuacct, memory [mm-owned], pids, freezer, hugetlb [mm-owned], rdma, dmem, perf_event, devices, net_cls, net_prio, blkio [block-owned], misc) all preserve their userspace-visible knob set and hierarchical accounting semantics.
- REQ-8: BPF user-visible ABI: bpf(2) syscall + all BPF_* commands, BTF format, helper function set, program-type list, map-type list, verifier accept/reject semantics for the upstream test corpus, JIT'd-code semantics. The verifier is permitted to be re-implemented in Rust, but its accept/reject decisions on the upstream test corpus must match.
- REQ-9: perf_event_open(2) ABI: counter types, event configurations, sampling modes, ring-buffer format, mmap'd page layout — match upstream so existing `perf` userspace works unmodified.
- REQ-10: ftrace ABI: tracepoint set + format, kprobe / uprobe semantics, ring-buffer format under `/sys/kernel/tracing/`, eventfs hierarchy. Existing trace-cmd / perf trace consumers work unmodified.
- REQ-11: Kernel module loading: the `init_module(2)` / `finit_module(2)` / `delete_module(2)` syscalls accept identical .ko binaries (per the recompile-from-source compat from `00-overview.md` REQ-4 / D2). Module signing semantics (CONFIG_MODULE_SIG) preserved.
- REQ-12: printk format: kernel log line format (`<loglevel>[<seq>][<timestamp>] subsys: message`), `dmesg` parser compatibility, the printk ringbuffer's userspace mmap layout (`/dev/kmsg`, `/proc/kmsg`).
- REQ-13: panic / oops / BUG output format: stack trace format, register dump format, "Tainted:" line, "RIP:" / "Code:" lines (x86 cross-ref) — preserved so crash-analysis tooling (kdump-tools, makedumpfile, crash, drgn) continues to work.
- REQ-14: kexec / kdump: REQ-15 of `00-overview.md`. Kernel-side glue lives in `crash-kexec.md`.
- REQ-15: Live-kernel-patching ABI: `/sys/kernel/livepatch/*` knobs and the kpatch `.ko` module conventions match upstream so existing live patches load.
- REQ-16: KGDB protocol over serial / network — preserved.
- REQ-17: All Tier-3 docs spawned from this overview each declare their unsafe-block clusters, TLA+ models (where novel concurrency primitives are introduced), and Kani harnesses (mandatory Layer-3 for `sched/runqueue invariants`).
- REQ-18: The implementation reuses upstream rust-for-linux's existing kernel abstractions where they exist (`rust/kernel/sync/`, `rust/kernel/workqueue.rs`, `rust/kernel/task.rs`, `rust/kernel/cred.rs`, `rust/kernel/cpu.rs`, `rust/kernel/cpumask.rs`, etc.) and extends rather than parallel-implementing.
- REQ-19: `kernel/sched/` is one of the four MANDATORY Layer-3 subsystems per `00-overview.md` D4 — Kani invariant harnesses for the runqueue's ordering invariants are required.

## Acceptance Criteria

- [ ] AC-1: Every syscall in kernel/'s purview has a `strace` golden-trace test on Rookery and upstream producing byte-identical traces. (covers REQ-1)
- [ ] AC-2: A diff between Rookery's and upstream's `/proc/<pid>/stat`, `status`, `cgroup`, `sched`, `wchan`, `stack`, `syscall`, `limits`, `comm`, `personality`, `ns/*`, and the system-wide `/proc/sched_debug`, `/proc/timer_list`, `/proc/interrupts`, `/proc/softirqs`, `/proc/sys/kernel/*` reports zero non-ignorable differences. (covers REQ-2)
- [ ] AC-3: A reference scheduler workload (e.g., `schbench`, `hackbench`, kernel-build) shows latency / throughput within accepted noise (~5%) of upstream. (covers REQ-3)
- [ ] AC-4: `lockdep`-enabled kernels run kernel selftests under `tools/testing/selftests/locking/` with the same pass/fail set as upstream. (covers REQ-4)
- [ ] AC-5: Time-related selftests under `tools/testing/selftests/timers/` pass identically. (covers REQ-5)
- [ ] AC-6: Futex selftests (`tools/testing/selftests/futex/`) pass identically. (covers REQ-6)
- [ ] AC-7: cgroup v1 + v2 selftests (`tools/testing/selftests/cgroup/`) pass identically. (covers REQ-7)
- [ ] AC-8: The full BPF selftest suite (`tools/testing/selftests/bpf/`) passes identically. (covers REQ-8)
- [ ] AC-9: `perf` userspace tool's recording, reporting, and stat modes run successfully against Rookery and produce equivalent results. (covers REQ-9)
- [ ] AC-10: ftrace selftests (`tools/testing/selftests/ftrace/`) pass identically. (covers REQ-10)
- [ ] AC-11: A canonical kernel module (`drivers/misc/lkdtm` or similar) built from the same source against Rookery's `vmlinux` loads and exhibits the same behavior as one built against upstream's `vmlinux`. (covers REQ-11)
- [ ] AC-12: `dmesg | grep <subsys>` output for early boot reports the same kernel-msg lines (modulo timestamps) on Rookery and upstream. (covers REQ-12)
- [ ] AC-13: A panic-and-kdump test produces a `vmcore` file consumable by `crash` or `drgn` against Rookery's `vmlinux`, with stack-trace decoding succeeding. (covers REQ-13)
- [ ] AC-14: A live-patch module (`samples/livepatch/livepatch-sample.ko`) loads onto a running Rookery kernel and applies its replacement function. (covers REQ-15)
- [ ] AC-15: KGDB selftest pass: kernel boots with `kgdboc=ttyS0` and `gdb` connects to it. (covers REQ-16)
- [ ] AC-16: `make verify` runs all Kani harnesses under `kernel/<area>/proofs/`; `make tla` runs all `.tla` models under `kernel/<area>/models/`. Both pass. (covers REQ-17, REQ-19)
- [ ] AC-17: A grep over Rookery for usages of `kernel::sync::*` shows reuse of upstream rust-for-linux abstractions; no parallel `kernel::myself::sync::*` namespace exists. (covers REQ-18)
- [ ] AC-18: A `kexec`-tools-driven test loads a Rookery `bzImage` via `kexec_load(2)` and `kexec_file_load(2)` and successfully boots into the loaded kernel; a mirror test from upstream → Rookery passes. (covers REQ-14)

## Architecture

### Layout map (Tier-3 docs spawned from this overview)

```
.design/kernel/
  00-overview.md              ← this document
  task-lifecycle.md           ← fork, exit, exec, signal, kthread, cred, capability, pid, namespaces
  sched/
    00-overview.md            ← scheduler family hub
    cfs.md                    ← CFS / EEVDF (the kernel/sched/fair.c content)
    rt.md                     ← real-time scheduler
    deadline.md               ← deadline scheduler
    ext.md                    ← sched_ext (BPF-driven scheduler class)
    idle.md                   ← idle scheduler + idle policies
    pelt.md                   ← Per-Entity Load Tracking
    topology.md               ← NUMA / cache topology
    clock.md                  ← sched_clock substrate
    cpufreq.md                ← cpufreq governor (schedutil; cross-references arch power)
  locking/
    00-overview.md            ← locking primitive hub
    mutex.md
    spinlock.md
    rwsem.md
    seqlock.md
    rcu.md
    lockdep.md
  time/
    00-overview.md            ← time hub
    hrtimer.md
    clocksource.md
    tick.md
    posix-timers.md
    namespace.md              ← time namespace
  futex.md                    ← futex syscalls + robust list + PI
  irq.md                      ← cross-arch IRQ infrastructure (irqdesc, irqdomain, MSI, IPI, …)
  workqueue.md                ← workqueue + async + irq_work + stop_machine
  smp-hotplug.md              ← SMP coordination + CPU hotplug; cross-references arch/x86/00-overview.md § kernel-platform.md
  cgroup/
    00-overview.md            ← cgroup framework hub
    core.md                   ← cgroupfs + cgroup_fork
    cpuset.md
    pids.md
    freezer.md
  bpf/
    00-overview.md            ← BPF hub
    verifier.md
    program-types.md
    maps.md                   ← hashtab, arraymap, lpm_trie, ringbuf, queue_stack_maps, bloom_filter, …
    btf.md                    ← BPF Type Format
    lsm.md                    ← BPF LSM (cross-ref security/)
    iter.md                   ← BPF iter
    helpers.md                ← helper function set
    arena.md                  ← arena maps (newer)
    struct-ops.md             ← struct_ops (kernel-data-structure-impl-via-BPF)
  perf-events.md              ← perf core + ring buffer + hw_breakpoint
  trace/
    00-overview.md            ← tracing hub
    ftrace.md                 ← function tracer
    tracepoints.md            ← static tracepoints
    kprobes.md                ← kprobes substrate (consumers in trace + bpf)
    uprobes.md
    blktrace.md               ← block tracing
    eventfs.md                ← /sys/kernel/tracing/ filesystem
  module-loading.md           ← init_module/finit_module/delete_module + .ko format + module signing
  printk.md                   ← printk + ringbuffer + nbcon + /dev/kmsg + console drivers (cross-ref)
  power-mgmt.md               ← suspend / hibernate / runtime PM core (arch portions delegated)
  syscall-entry-helpers.md    ← kernel/entry/ generic helpers consumed by arch/<arch>/entry/
  dma-mapping.md              ← DMA mapping API
  livepatch.md
  liveupdate.md               ← (deferred candidate; see Q3)
  kgdb.md                     ← KGDB substrate
  unwind.md                   ← stack unwinder (frame pointer / orc / dwarf)
  coverage.md                 ← gcov + kcov
  audit.md                    ← audit subsystem (cross-ref security/)
  crash-kexec.md              ← kexec/kdump kernel-side glue (arch glue in arch/x86/00-overview.md § boot.md)
  panic-reboot.md             ← panic / hung_task / watchdog / reboot
  sysctl-ksysfs.md            ← sysctl + ksysfs + configs.c + .config exposure
  kallsyms.md                 ← kernel symbol table + /proc/kallsyms + kcmp
  runtime-codepatching.md     ← jump_label + cfi + extable + static_call
  accounting.md               ← acct + delayacct + cpu_pm
```

### Cross-references

- `arch/x86/00-overview.md` — IRQ entry hardware bring, IDT setup, FPU per-task, signal frame layout (`arch/x86/signal.md`), kexec hand-off ABI (`arch/x86/boot.md`), CPU-mitigations sysfs.
- `mm/00-overview.md` — page-fault handler interaction with task state; OOM-killer + memcg; cgroup memory controller.
- `lib/00-overview.md` — data structures (rbtree, list, kfifo, maple_tree, xarray) used pervasively; vsprintf for printk; KUnit framework; vdso-core for time.
- `fs/00-overview.md` (Phase B) — VFS interaction with task lifecycle (file table, fdtable); /proc/<pid>/ pseudo-FS implementation lives in fs/proc/.
- `block/00-overview.md` (Phase B) — blktrace tracing.
- `net/00-overview.md` (Phase B) — netlink for audit/perf-equivalents; BPF program types (XDP, SK_*).
- `crypto/00-overview.md` (Phase B) — random.c is in `drivers/char/random.c` actually but RNG seed feeding from kernel/ is here; module signing crypto is here.
- `security/00-overview.md` (Phase B) — LSM hooks, BPF LSM, audit, capability checks, credentials.

### Rust module organization (informative)

Many already exist in upstream rust-for-linux; Rookery extends:

- `kernel::sync` — exists (`rust/kernel/sync/`); extend with RwSemaphore (issue #4)
- `kernel::workqueue` — exists (`rust/kernel/workqueue.rs`)
- `kernel::task` — exists (`rust/kernel/task.rs`); extend with Kthread spawn (issue #4)
- `kernel::cred` — exists (`rust/kernel/cred.rs`)
- `kernel::cpu` — exists (`rust/kernel/cpu.rs`); extend with PerCpu<T> (issue #4)
- `kernel::cpumask` — exists (`rust/kernel/cpumask.rs`)
- `kernel::cpufreq` — exists (`rust/kernel/cpufreq.rs`)
- `kernel::time` — Rookery to author (Tier 3 in `time/00-overview.md`)
- `kernel::sched` — Rookery to author (Tier 3 in `sched/00-overview.md`)
- `kernel::futex` — Rookery to author
- `kernel::cgroup` — Rookery to author
- `kernel::bpf` — Rookery to author (large)
- `kernel::perf_event` — Rookery to author
- `kernel::trace` — Rookery to author
- `kernel::module` — Rookery to author
- `kernel::printk` — Rookery to author
- `kernel::panic` — Rookery to author (this is the panic_handler from `00-rust-conventions.md` REQ-5)

### Locking and concurrency

The kernel/ subsystem authors most concurrency primitives. Locking landscape:
- The locking primitives themselves are designed in `locking/00-overview.md` (mutex, spinlock, rwsem, seqlock, RCU). Each carries a TLA+ model (Layer 2).
- Scheduler runqueue: per-CPU `rq->lock` (raw_spinlock); cross-CPU coordination via IPI + RCU. CFS rbtree updates serialize via rq->lock.
- RCU itself: see `locking/rcu.md`. The most concurrency-dense single component in the kernel.
- Workqueue worker pools: per-CPU + unbound pools, each protected by `pool->lock`.
- Cgroup hierarchy: rwsem `cgroup_mutex` for hierarchy mutation; per-cgroup rwsem for membership.
- BPF: verifier holds bpf_verifier_lock during program load; running programs use RCU to traverse program arrays.
- perf events: per-CPU + per-task contexts; switching between them on schedule via context-switch hooks.
- futex: hashed wait queues; per-bucket spinlock.

### Error handling

Conformant per `00-rust-conventions.md` REQ-6: `Result<T, KernelError>` everywhere a fallible op exists. Specific notes:
- Scheduler `schedule()` has no fallible-return surface — it does not return until the next time the task runs. The Rust counterpart (`kernel::sched::schedule()`) returns `()`.
- `kernel::panic!` macro (per `00-rust-conventions.md` REQ-5) implements `!` (never returns) and routes to `kernel/panic.c::panic`.
- BPF verifier returns `Result<bpf_prog, KernelError>` with `EINVAL` for rejection; the program load syscall surfaces the verifier's verbose log via the `log_buf` argument.

## Verification

### Layer 1: Kani SAFETY proofs

Anticipated `unsafe` clusters (logical groupings):
- `kernel::sched::*` — runqueue manipulation, context-switch low-level handoff, percpu access. Harnesses: `kani::proofs::kernel::sched::*_safety`.
- `kernel::sync::*` — already partly exists upstream; Rookery's RwSemaphore + per-CPU additions need new harnesses.
- `kernel::futex::*` — userspace shared-memory access, atomic ops on user pages.
- `kernel::bpf::verifier::*` — heavy unsafe in bytecode walking and abstract interpretation.
- `kernel::printk::*` — ringbuffer write barriers.
- `kernel::module::*` — relocation + symbol resolution against loaded code.
- `kernel::trace::ftrace::*` — runtime code-patching (see also `runtime-codepatching.md`).
- `kernel::workqueue::*` — already partly exists upstream; extensions need harnesses.

### Layer 2: TLA+ models (mandatory for novel concurrency)

Models required at this tier:
- `models/kernel/sched/runqueue.tla` — proves CFS rbtree ordering invariants under concurrent enqueue / dequeue / wakeup / migration.
- `models/kernel/sched/load_balance.tla` — proves load-balancer's task migration preserves runqueue invariants.
- `models/kernel/sched/preempt.tla` — proves preemption-disable / preemption-enable counter is well-balanced and never goes negative.
- `models/kernel/locking/qspinlock.tla` — proves MCS-based queued spinlock fairness + safety. (Already exists in academic literature; Rookery ships the Rust-implementation-specific model.)
- `models/kernel/locking/qrwlock.tla` — queued rwlock fairness.
- `models/kernel/locking/rcu_grace_period.tla` — proves grace-period detection: a grace period ends only after every previously-running RCU read-side critical section has completed.
- `models/kernel/locking/percpu_rwsem.tla` — fast-path / slow-path correctness.
- `models/kernel/futex/futex_wait_wake.tla` — proves no fault is lost across `FUTEX_WAIT` + `FUTEX_WAKE` on multiple CPUs.
- `models/kernel/futex/futex_pi.tla` — proves PI-futex priority-inheritance correctness (no priority inversion).
- `models/kernel/cgroup/hierarchy.tla` — proves cgroup hierarchical accounting consistency under concurrent fork / migrate / destroy.
- `models/kernel/workqueue/pool.tla` — proves worker-pool work-item ordering and per-CPU vs. unbound semantics.
- `models/kernel/bpf/verifier.tla` — high-level model of verifier termination + soundness (a program declared safe by the verifier has no UB-equivalent in BPF semantics).
- `models/kernel/printk/ringbuf.tla` — proves printk ringbuffer is lock-free-write-able under NMI without torn states.
- `models/kernel/trace/ringbuf.tla` — analogous for ftrace ring buffer.

### Layer 3: Kani harnesses for data-structure invariants (MANDATORY for kernel/sched)

Per `00-overview.md` D4, scheduler runqueue is one of the four mandatory Layer-3 areas:

| Data structure | Invariant | Harness |
|---|---|---|
| CFS rbtree (per CPU) | "The leftmost node has the smallest vruntime among RUNNABLE tasks on this CPU" + "rbtree red-black invariants hold" | `kani::proofs::kernel::sched::cfs::rbtree_invariants` |
| RT runqueue priority arrays | "Highest priority bitmap matches highest-priority non-empty array" | `kani::proofs::kernel::sched::rt::priority_invariants` |
| Deadline rbtree | "Leftmost node has earliest absolute deadline" | `kani::proofs::kernel::sched::dl::deadline_invariants` |
| Priority-inversion graph (PI futex / RT mutex) | "PI graph contains no cycles" | `kani::proofs::kernel::futex::pi_acyclic` |
| Cgroup hierarchical accounting | "Sum of children's charges + own charge ≤ parent's charge" | `kani::proofs::kernel::cgroup::accounting_invariants` |

Plus invariants per Tier-3 doc as their components mature.

### Layer 4: Functional correctness via Creusot / Verus / Prusti (opt-in)

Strong opt-in candidates:
- `locking/rcu.md` — RCU grace-period detection algorithm (well-known, formally interesting; Verus is a candidate).
- `bpf/verifier.md` — verifier soundness theorem; this is a major undertaking but high-value. May slip past v0.
- `sched/cfs.md` — fairness theorems (every runnable task eventually runs, weights produce proportional CPU time on average); Verus or Creusot.
- `sched/deadline.md` — schedulability (admitted set is feasible per Earliest-Deadline-First).

## Hardening

Placeholder per `00-overview.md` D6. The kernel/ tier owns implementation of:

- **PAX_PRIVATE_KSTACKS-equivalent**: per-CPU IST stacks (arch tier owns x86 instance); per-task kernel stacks.
- **PAX_RANDKSTACK-equivalent**: kernel-stack offset randomization on syscall entry (cross-arch infrastructure here, x86 instantiation in `arch/x86/entry.md`).
- **CFI / kCFI / RAP-equivalent**: `runtime-codepatching.md` owns the substrate; `arch/x86/cpu-mitigations.md` owns x86-side instantiation.
- **CONFIG_STACKPROTECTOR_STRONG**: enabled by default per upstream.
- **GRKERNSEC_HIDESYM-equivalent**: `/proc/kallsyms` `kptr_restrict` semantics enforced; opt-in stricter modes via the deferred `00-security-principles.md`.
- **GRKERNSEC_DMESG-equivalent**: `dmesg_restrict` sysctl semantics enforced.
- **Speculative-execution mitigations**: `arch/x86/cpu-mitigations.md` owns the implementation; the kernel-tier glue (mds_user_clear, switch_mm hooks) is here.

Each Tier-3 doc whose content touches a hardening feature gains a "Hardening" section once `00-security-principles.md` is authored.

## Resolved Decisions

### D1 (2026-05-09): sched_ext IN v0
`kernel/sched/ext.c` (BPF-driven scheduler class) is in v0 scope. Distros ship scx tools (`scx_simple`, `scx_rusty`); excluding breaks them. The BPF surface is well-isolated. `sched/ext.md` Tier 3 covers it.

### D2 (2026-05-09): BPF verifier — RE-IMPLEMENT in Rust (highest-leverage Layer-4 target)
The BPF verifier (~25K LoC) is re-implemented in Rust. Per the formal-verification baseline (`00-overview.md` D4), the verifier is a strong candidate for Layer-4 functional-correctness proofs (Verus). This is the one place a Rust-from-scratch + formal-proof approach pays dividends most clearly: a verifier soundness theorem ("any program declared safe by the verifier has no UB-equivalent in BPF semantics") is high-value and tractable in Verus. `bpf/verifier.md` Tier-3 will be the largest single Tier-3 doc in the project.

### D3 (2026-05-09): liveupdate (`kernel/liveupdate/`) OUT OF SCOPE for v0
Depends on serializing arbitrary kernel state across an upgrade — interacts with everything. The `kernel/liveupdate/` directory left empty in Rookery; CONFIG_LIVEUPDATE off by default. Re-evaluate in v1+.

### D4 (2026-05-09): Audit subsystem IN v0
CONFIG_AUDIT=y is the RHEL/Fedora/SLES distro default. `audit.md` Tier 3 covers syscall-audit + filesystem-audit + LSM-audit-helpers split.

### D5 (2026-05-09): Full ftrace IN v0 (not minimal tracepoints-only)
ftrace (~30K LoC across `kernel/trace/`) is fully reimplemented. Existing trace-cmd / perf trace / bpftrace consumers depend on the full surface. Tier-3 docs (`trace/ftrace.md` etc.) acknowledge the substantial implementation cost; Layer-2 TLA+ verification focuses on the ringbuffer.

## Open Questions

(none — all open questions for this subsystem document are resolved above)

## Out of Scope

- 32-bit-only kernel paths (consistent with `arch/x86/00-overview.md` D1).
- Live update (`kernel/liveupdate/`) — pending Q3.
- Architecture-specific debugging support beyond x86 KGDB (no v0 ARM debug stub, etc.).
- Test fixtures — covered by the components they test, not separately documented.
- Implementation code — `.design/` contains specs only.
