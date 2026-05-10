# Tier-2: fs/proc — procfs (process and kernel-state pseudo-filesystem)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/proc/
  - include/linux/proc_fs.h
-->

## Summary
Tier-2 overview for procfs — the kernel's longest-standing pseudo-filesystem (predates sysfs/cgroupfs/configfs/debugfs/tracefs). Exposes:
1. **Per-task** `/proc/<pid>/*` — task state (`stat`, `status`, `cmdline`, `comm`, `cwd`, `exe`, `root`, `mem`, `maps`, `numa_maps`, `smaps`, `pagemap`, `fd/`, `task/`, `ns/`, `wchan`, `sched`, `oom_score`, `cgroup`, `mountinfo`, `mountstats`, `latency`, `personality`, `loginuid`, `sessionid`, `attr/`, `auxv`, `environ`, `gid_map`, `uid_map`, `setgroups`, `clear_refs`, `coredump_filter`, `make-it-fail`, …)
2. **System-wide kernel state** — `/proc/cpuinfo`, `/proc/meminfo`, `/proc/loadavg`, `/proc/uptime`, `/proc/stat`, `/proc/version`, `/proc/cmdline`, `/proc/devices`, `/proc/interrupts`, `/proc/iomem`, `/proc/ioports`, `/proc/swaps`, `/proc/diskstats`, `/proc/partitions`, `/proc/filesystems`, `/proc/mounts`, `/proc/modules`, `/proc/kallsyms`, `/proc/kcore`, `/proc/kmsg`, `/proc/buddyinfo`, `/proc/zoneinfo`, `/proc/vmstat`, `/proc/slabinfo`, `/proc/locks`, `/proc/timer_list`, `/proc/timer_stats`, `/proc/consoles`, `/proc/softirqs`, `/proc/schedstat`, `/proc/asound/`, `/proc/sys/`, `/proc/net/`, `/proc/scsi/`, …
3. **Sysctl tree** — `/proc/sys/...` (cross-ref `proc-sysctl.md`) routes reads/writes to per-subsys-registered `ctl_table`s (the sysctl-via-procfs path coexists with `syscall(__NR_sysctl)` deprecated alternative)
4. **Per-netns + per-network state** — `/proc/net/...` (cross-ref `proc-net.md`)
5. **Per-tty** — `/proc/tty/`
6. **Symlinks** — `/proc/self`, `/proc/thread-self`, `/proc/<pid>/exe`, etc.

procfs is the single most-used pseudo-filesystem on every Linux system: every `ps` / `htop` / `top` / `lsof` / `pgrep` / `iotop` / `pmap` / `strace -p` / `gdb attach` / kernel module enumeration / system-info command depends on it. Its compatibility surface is enormous and its format is "load-bearing" for distro tooling.

Sub-tier-2 of `fs/00-overview.md` (under VFS umbrella). Sibling of `fs/sysfs/00-overview.md` (deferred), `fs/cgroup/00-overview.md` (deferred for Phase D), `fs/debugfs/00-overview.md` (deferred).

## Scope

This Tier-2 governs **all** of `/home/doll/linux-src/fs/proc/` (~30 source files) plus the public API headers. Per-file Tier-3 docs live under `.design/fs/proc/`.

## Compatibility contract — outline

### Mount semantics

`mount -t proc proc /proc` (default mount per init scripts). Per-netns rendering for `/proc/net/*` (per-netns view of network state).

Per-mount-options:
- `hidepid=N` — control per-pid visibility (`0`=all, `1`=hide-other-uid pids, `2`=hide-pid-listing entirely)
- `gid=N` — group that bypasses hidepid restrictions
- `subset=pid` — show only `/proc/<pid>` (no system-wide entries) — used by container-runtime to constrain per-container /proc

Identical mount-options + semantics.

### Per-task `/proc/<pid>/*` (`fs/proc/base.c`, `array.c`, `task_mmu.c`)

Per-pid directory dynamically generated; entries match upstream byte-for-byte.

Key entries (subset):
- `stat` — single-line space-delimited per-task stats (52 fields, including pid/comm/state/ppid/pgrp/session/tty_nr/tpgid/flags/minflt/cminflt/majflt/cmajflt/utime/stime/cutime/cstime/priority/nice/num_threads/itrealvalue/starttime/vsize/rss/rsslim/startcode/endcode/startstack/kstkesp/kstkeip/signal/blocked/sigignore/sigcatch/wchan/nswap/cnswap/exit_signal/processor/rt_priority/policy/delayacct_blkio_ticks/guest_time/cguest_time/start_data/end_data/start_brk/arg_start/arg_end/env_start/env_end/exit_code) — format byte-identical, parsed by every `ps`/`top`/`htop`
- `status` — multi-line key:value (Name/Umask/State/Tgid/Ngid/Pid/PPid/TracerPid/Uid/Gid/FDSize/Groups/NStgid/NSpid/NSpgid/NSsid/VmPeak/VmSize/VmLck/VmPin/VmHWM/VmRSS/VmData/VmStk/VmExe/VmLib/VmPTE/VmSwap/THP_enabled/HugetlbPages/CoreDumping/THP_enabled/Threads/SigQ/SigPnd/ShdPnd/SigBlk/SigIgn/SigCgt/CapInh/CapPrm/CapEff/CapBnd/CapAmb/NoNewPrivs/Seccomp/Speculation_Store_Bypass/Cpus_allowed/Cpus_allowed_list/Mems_allowed/Mems_allowed_list/voluntary_ctxt_switches/nonvoluntary_ctxt_switches)
- `cmdline` — argv joined with NULs; format byte-identical
- `comm` — task-name string (16 bytes max + newline)
- `cwd`, `exe`, `root` — symlinks to actual paths
- `maps` — VMA list; format byte-identical
- `numa_maps`, `smaps`, `smaps_rollup` — VMA NUMA + per-VMA memory stats
- `pagemap` — binary per-VPN→PFN+flags mapping (used by `pagemap2 -e`/`vmtouch`/`page-types`)
- `fd/` — per-FD subdirectory with symlinks to underlying file
- `fdinfo/` — per-FD detailed info (pos/flags/mnt_id/eventfd/perf-event-attached/etc.)
- `task/<tid>/` — per-thread sub-directories with same entries as `<pid>`
- `ns/` — per-task namespace symlinks (mnt/uts/ipc/pid/net/user/cgroup/time/all)
- `mountinfo`, `mounts`, `mountstats` — per-task view of mounted filesystems
- `cgroup` — current cgroup memberships
- `oom_score`, `oom_score_adj`, `oom_adj` (deprecated) — per-task OOM tuning
- `loginuid`, `sessionid` — audit-related
- `gid_map`, `uid_map`, `setgroups` — user-namespace mappings
- `clear_refs` — per-VMA referenced-bit clear (for working-set probing)
- `coredump_filter` — per-task core-dump-segment filter
- `attr/current`, `attr/exec`, `attr/fscreate`, `attr/keycreate`, `attr/sockcreate` — LSM attributes (SELinux/Smack/AppArmor)

Format byte-identical for every entry (this is the highest-criticality compatibility surface in the kernel).

### System-wide entries

- `/proc/cpuinfo` — per-CPU info (architecture-specific format; cross-ref `arch/x86/kernel-platform.md`)
- `/proc/meminfo` — system-wide memory; format byte-identical (per `top`/`free` parser)
- `/proc/loadavg` — `1min 5min 15min running/total last_pid` byte-identical
- `/proc/uptime` — `uptime_sec idle_sec` byte-identical
- `/proc/stat` — per-CPU + system stats (cpu/user/nice/system/idle/iowait/irq/softirq/steal/guest/guest_nice + intr + ctxt + btime + processes + procs_running + procs_blocked + softirq) byte-identical
- `/proc/cmdline` — kernel boot cmdline byte-identical
- `/proc/version` — kernel-version-string byte-identical
- `/proc/devices` — registered char + block major numbers
- `/proc/filesystems` — filesystem types
- `/proc/modules` — loaded modules
- `/proc/kallsyms` — kernel symbol table (gated by `kptr_restrict` sysctl)
- `/proc/kcore` — kernel-core ELF (debug)
- `/proc/kmsg` — kernel printk output (consumed by syslogd / klogd)
- `/proc/buddyinfo`, `/proc/zoneinfo`, `/proc/vmstat` (lives in mm/) — memory subsystem stats

### `/proc/sys/...` (sysctl bridge)

Per-subsystem `register_sysctl()` registers `ctl_table` arrays; `proc_sysctl.c` exposes them as procfs files. Read/write syscalls translate to per-handler get/set (e.g., `proc_dointvec`/`proc_dostring`/`proc_dointvec_minmax`).

Cross-ref `fs/proc/proc-sysctl.md` Tier-3 (deferred).

### `/proc/net/...` (per-netns network state)

Per-netns rendering of network subsystem state (cross-ref `proc-net.md` Tier-3 deferred). Examples: `/proc/net/tcp`, `/proc/net/udp`, `/proc/net/unix`, `/proc/net/route`, `/proc/net/snmp`, `/proc/net/dev`, `/proc/net/netfilter/*`.

Cross-ref individual subsystem Tier-3s for content (e.g., `net/ipv4/tcp.md` defines `/proc/net/tcp` format; `net/netfilter/conntrack-core.md` defines `/proc/net/nf_conntrack` format).

## Tier-3 docs governed by this Tier-2

(Phase C will add these incrementally.)

| Tier-3 doc | Scope |
|---|---|
| `fs/proc/proc-task.md` | Per-task `/proc/<pid>/*` (`base.c`, `array.c`, `task_mmu.c`, `fd.c`) |
| `fs/proc/proc-system.md` | System-wide entries (`cpuinfo.c`, `meminfo.c`, `loadavg.c`, `stat.c`, `uptime.c`, etc.) |
| `fs/proc/proc-sysctl.md` | `/proc/sys/...` sysctl bridge (`proc_sysctl.c`) |
| `fs/proc/proc-net.md` | `/proc/net/...` per-netns network state (`proc_net.c`) |
| `fs/proc/proc-namespaces.md` | `/proc/<pid>/ns/*` (`namespaces.c`) |
| `fs/proc/proc-fd.md` | `/proc/<pid>/fd/` + `/proc/<pid>/fdinfo/` (`fd.c`) |
| `fs/proc/proc-generic.md` | Generic procfs framework (`generic.c`, `inode.c`, `root.c`, `internal.h`) |
| `fs/proc/proc-kcore.md` | `/proc/kcore` (`kcore.c`) |
| `fs/proc/proc-kmsg.md` | `/proc/kmsg` (`kmsg.c`) |

## Compatibility outline (top-level)

- REQ-O1: Per-task `/proc/<pid>/*` entries byte-identical to upstream for stat/status/cmdline/comm/maps/smaps/numa_maps/pagemap/fd*/task*/ns*/cgroup/mounts*/oom_score/loginuid/gid_map/uid_map/clear_refs/attr*/auxv/environ.
- REQ-O2: System-wide entries byte-identical: cpuinfo (per-arch via `arch/*/kernel-platform.md`), meminfo, loadavg, uptime, stat, cmdline, version, devices, filesystems, modules, kallsyms, kcore, kmsg, buddyinfo, zoneinfo.
- REQ-O3: `/proc/sys/...` sysctl bridge: `register_sysctl()` per-subsys; per-handler `proc_dointvec`/`proc_dostring`/`proc_dointvec_minmax` etc. byte-identical.
- REQ-O4: `/proc/net/...` per-netns rendering; per-subsystem-defined format.
- REQ-O5: Per-mount options: `hidepid=N`, `gid=N`, `subset=pid` byte-identical semantics.
- REQ-O6: Symlinks `/proc/self`, `/proc/thread-self`, `/proc/<pid>/exe`, `/proc/<pid>/cwd`, `/proc/<pid>/root`, `/proc/<pid>/fd/<n>` resolve identically.
- REQ-O7: Per-Tier-3 child documents define per-component requirements.
- REQ-O8: Hardening: `kptr_restrict`, `dmesg_restrict`, `hidepid` defaults secured per `00-security-principles.md`.

## Acceptance Criteria (top-level)

- [ ] AC-O1: `cat /proc/<pid>/stat` byte-identical to upstream's; `ps -ef` parsing succeeds. (covers REQ-O1)
- [ ] AC-O2: `cat /proc/meminfo` + `cat /proc/cpuinfo` + `cat /proc/loadavg` byte-identical. (covers REQ-O2)
- [ ] AC-O3: `sysctl net.ipv4.tcp_max_syn_backlog` reads/writes via `/proc/sys/...` byte-identical. (covers REQ-O3)
- [ ] AC-O4: `cat /proc/net/tcp` byte-identical (cross-ref `net/ipv4/tcp.md` format). (covers REQ-O4)
- [ ] AC-O5: `mount -t proc -o hidepid=2 proc /proc`; non-root user sees only own pids. (covers REQ-O5)
- [ ] AC-O6: `readlink /proc/self/exe` returns the calling process's executable. (covers REQ-O6)
- [ ] AC-O7: Per-Tier-3 ACs cumulatively cover the procfs ABI surface. (meta-AC)
- [ ] AC-O8: Hardening section per Tier-3 child docs is non-empty. (covers REQ-O8)

## Architecture (top-level)

Each Tier-3 child doc declares its own Rust module organization. Shared abstractions:

- `kernel::fs::proc::ProcFs` — top-level filesystem type
- `kernel::fs::proc::generic::ProcInode` — `struct proc_inode` wrapper
- `kernel::fs::proc::generic::ProcDirEntry` — `struct proc_dir_entry`
- `kernel::fs::proc::base::ProcPid` — per-pid directory generator
- `kernel::fs::proc::base::ProcSelf` / `ThreadSelf` — symlinks
- `kernel::fs::proc::array::TaskStat` / `TaskStatus` — per-task formatted output
- `kernel::fs::proc::task_mmu::TaskMaps` / `Smaps` / `Pagemap` — per-task memory state
- `kernel::fs::proc::fd::ProcFd` — `/proc/<pid>/fd/` + `/proc/<pid>/fdinfo/`
- `kernel::fs::proc::namespaces::ProcNs` — `/proc/<pid>/ns/*`
- `kernel::fs::proc::sysctl::ProcSysctl` — `/proc/sys/...` bridge
- `kernel::fs::proc::net::ProcNet` — `/proc/net/...` per-netns
- `kernel::fs::proc::system::*` — per-system-wide-file modules (cpuinfo / meminfo / etc.)

## Verification (top-level)

### Layer 1: Kani SAFETY proofs

Each Tier-3 child doc declares its own.

### Layer 2: TLA+ models — mandatory list

(none mandatory at this level — procfs is read-mostly with per-file-snapshot semantics; no concurrent-state-machine surfaces requiring TLA+)

### Layer 3: invariant harnesses

Per Tier-3.

### Layer 4: functional correctness (opt-in)

- **Per-task /proc/<pid>/stat snapshot consistency** via Verus — proves: per-task stat output is a consistent snapshot (no half-updated counter values), even under concurrent task-state mutation.

## Hardening (top-level)

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this Tier-2

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-`proc_dir_entry` + per-task ref refcounts use `Refcount` (saturating) | § Mandatory |
| **AUTOSLAB** | per-`proc_inode` slab cache | § Mandatory |
| **MEMORY_SANITIZE** | freed proc-inode + per-task formatted-output buffers cleared (carry sensitive data — auxv, environ, secctx) | § Default-on configurable off |
| **HIDEPID-DEFAULT** | per-mount default `hidepid=2` (preserves non-root /proc privacy from accidental info disclosure; configurable) | § Default-on configurable off |
| **KPTR_RESTRICT** | `/proc/kallsyms` redacts addresses (`%pK` printk) for non-CAP_SYSLOG processes; default `kptr_restrict=1` (or 2 for stricter, locking even root) | § Default-on configurable off |
| **DMESG_RESTRICT** | `/proc/kmsg` + `/proc/<pid>/syscall` etc. require CAP_SYSLOG; default on | § Default-on configurable off |

### Row-1 features consumed by this Tier-2

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: per-file `proc_ops` vtable + per-pid file-list `static const`
- **USERCOPY**: per-file read/write uses bound-checked `simple_*_to_user`/`copy_*_user`
- **SIZE_OVERFLOW**: per-pid file-offset arithmetic uses checked operators
- **KERNEXEC**: per-file dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hooks: `security_inode_permission` / `security_file_permission` for every procfs read/write — gives SELinux/AppArmor/Smack control over per-file access.
- LSM hook `security_task_to_inode` for per-task /proc inode labeling.
- Default useful GR-RBAC policy: deny `/proc/<pid>/mem` writes outside gradm-marked `debugger` role; deny `/proc/kcore` reads outside `kernel_debugger` role.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Open Questions

(none at Tier-2 — defer to per-Tier-3)

## Out of Scope

- Per-Tier-3 child docs (Phase C continuation)
- `vmstat.c` lives in `mm/vmstat.c` (cross-ref `mm/00-overview.md`); shown via procfs but logically owned by mm
- 32-bit-only paths
- Implementation code
