# Tier-3: fs/proc/proc-misc — top-level read-only /proc files (cmdline, version, uptime, loadavg, meminfo, stat, cpuinfo, interrupts, diskstats, partitions, swaps, buddyinfo, zoneinfo, vmstat, vmallocinfo, softirqs, consoles, devices)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/proc/00-overview.md
upstream-paths:
  - fs/proc/cmdline.c (~24 lines)
  - fs/proc/version.c (~27 lines)
  - fs/proc/uptime.c (~49 lines)
  - fs/proc/loadavg.c (~37 lines)
  - fs/proc/meminfo.c (~188 lines)
  - fs/proc/stat.c (~216 lines)
  - fs/proc/cpuinfo.c (~28 lines)
  - fs/proc/interrupts.c (~42 lines)
  - fs/proc/softirqs.c (~37 lines)
  - fs/proc/consoles.c (~116 lines)
  - fs/proc/devices.c (~64 lines)
  - mm/page_alloc.c (proc_create_seq buddyinfo, pagetypeinfo)
  - mm/vmstat.c (proc_create_seq vmstat, zoneinfo)
  - mm/vmalloc.c (proc_create_single vmallocinfo)
  - mm/swapfile.c (proc_create swaps)
  - block/genhd.c (proc_create_seq diskstats, partitions)
  - include/linux/proc_fs.h
  - include/linux/seq_file.h
-->

## Summary

The `/proc` root directory hosts a family of small read-only files, each owned by its subsystem and registered via the procfs single-show / seq-file helpers. Per-file pattern: a tiny `*_proc_show(seq_file, void *)` (or `seq_operations`) prints one snapshot of subsystem state on read; registration runs at `fs_initcall` and produces a permanent `struct proc_dir_entry`. No write side. Per-`/proc/cmdline` per-`/proc/version` per-`/proc/uptime` per-`/proc/loadavg` are one-line dumps of `saved_command_line` / `linux_proc_banner` + `utsname()` / `ktime_get_boottime_ts64` + idle / `get_avenrun`. Per-`/proc/meminfo` per-`/proc/stat` per-`/proc/cpuinfo` per-`/proc/interrupts` per-`/proc/softirqs` are larger dumps: meminfo prints per-counter `kB` rows from `global_node_page_state` + `si_meminfo`; stat prints per-CPU cpustat + irq counters + ctxt + procs_running; cpuinfo + interrupts are seq-iterators over per-arch / per-IRQ data. Per-`/proc/diskstats` per-`/proc/partitions` live in `block/genhd.c`. Per-`/proc/swaps` lives in `mm/swapfile.c`. Per-`/proc/buddyinfo` per-`/proc/pagetypeinfo` live in `mm/page_alloc.c`. Per-`/proc/zoneinfo` per-`/proc/vmstat` live in `mm/vmstat.c`. Per-`/proc/vmallocinfo` lives in `mm/vmalloc.c`. Critical for: `ps` / `top` / `free` / `vmstat` / `lscpu` / `iostat` / `swapon -s` and dozens of other userland-observability tools whose entire data-path is these files.

This Tier-3 covers approximately 18 endpoints across ~828 lines of registration / show glue in `fs/proc/` plus the per-subsystem owners listed above. The procfs scaffolding (`proc_create`, `proc_create_single`, `proc_create_seq`, `pde_make_permanent`, `single_open_size`) is covered in `proc-generic.md`; this Tier-3 covers only the per-file show functions, their data sources, and the user-visible textual format.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `cmdline_proc_show()` (fs/proc/cmdline.c) | per-read /proc/cmdline | `ProcMisc::cmdline_show` |
| `proc_cmdline_init()` | per-fs_initcall register | `ProcMisc::cmdline_init` |
| `version_proc_show()` (fs/proc/version.c) | per-read /proc/version | `ProcMisc::version_show` |
| `proc_version_init()` | per-fs_initcall register | `ProcMisc::version_init` |
| `uptime_proc_show()` (fs/proc/uptime.c) | per-read /proc/uptime | `ProcMisc::uptime_show` |
| `proc_uptime_init()` | per-fs_initcall register | `ProcMisc::uptime_init` |
| `loadavg_proc_show()` (fs/proc/loadavg.c) | per-read /proc/loadavg | `ProcMisc::loadavg_show` |
| `proc_loadavg_init()` | per-fs_initcall register | `ProcMisc::loadavg_init` |
| `meminfo_proc_show()` (fs/proc/meminfo.c) | per-read /proc/meminfo | `ProcMisc::meminfo_show` |
| `show_val_kb()` | per-counter `kB` formatter | `ProcMisc::show_val_kb` |
| `arch_report_meminfo()` (weak) | per-arch hook | `ProcMisc::arch_report_meminfo` |
| `proc_meminfo_init()` | per-fs_initcall register | `ProcMisc::meminfo_init` |
| `show_stat()` (fs/proc/stat.c) | per-read /proc/stat | `ProcMisc::stat_show` |
| `get_idle_time()` | per-CPU idle nsec | `ProcMisc::get_idle_time` |
| `get_iowait_time()` (static) | per-CPU iowait nsec | `ProcMisc::get_iowait_time` |
| `show_all_irqs()` / `show_irq_gap()` (static) | per-IRQ counters | `ProcMisc::show_all_irqs` |
| `stat_open()` + `stat_proc_ops` | per-file size hint open | `ProcMisc::stat_open` |
| `proc_stat_init()` | per-fs_initcall register | `ProcMisc::stat_init` |
| `cpuinfo_op` (arch-specific seq_ops; e.g. arch/x86/kernel/cpu/proc.c) | per-CPU cpuinfo iterator | shared arch-side |
| `cpuinfo_open()` + `cpuinfo_proc_ops` | per-file open | `ProcMisc::cpuinfo_open` |
| `proc_cpuinfo_init()` | per-fs_initcall register | `ProcMisc::cpuinfo_init` |
| `int_seq_ops` / `int_seq_start` / `_next` / `_stop` (fs/proc/interrupts.c) | per-IRQ iterator | `ProcMisc::interrupts_seq_ops` |
| `show_interrupts()` (kernel/irq/proc.c) | per-IRQ row print | shared irq-side |
| `proc_interrupts_init()` | per-fs_initcall register | `ProcMisc::interrupts_init` |
| `show_softirqs()` (fs/proc/softirqs.c) | per-softirq counts | `ProcMisc::softirqs_show` |
| `proc_softirqs_init()` | per-fs_initcall register | `ProcMisc::softirqs_init` |
| `consoles_op` (fs/proc/consoles.c) | per-console iterator | `ProcMisc::consoles_seq_ops` |
| `proc_consoles_init()` | per-fs_initcall register | `ProcMisc::consoles_init` |
| `devinfo_ops` (fs/proc/devices.c) | per-major iterator | `ProcMisc::devices_seq_ops` |
| `proc_devices_init()` | per-fs_initcall register | `ProcMisc::devices_init` |
| `fragmentation_op` (mm/page_alloc.c) | per-zone buddy-free histogram | shared mm-side |
| `pagetypeinfo_op` (mm/page_alloc.c) | per-migratetype histogram | shared mm-side |
| `vmstat_op` (mm/vmstat.c) | per-counter iterator | shared mm-side |
| `zoneinfo_op` (mm/vmstat.c) | per-zone watermarks + counters | shared mm-side |
| `vmalloc_info_show()` (mm/vmalloc.c) | per-vmap_area dump | shared mm-side |
| `swaps_proc_ops` (mm/swapfile.c) | per-swap-area iterator | shared mm-side |
| `diskstats_op` (block/genhd.c) | per-disk I/O counters | shared blk-side |
| `partitions_op` (block/genhd.c) | per-partition / major-minor | shared blk-side |
| `saved_command_line` / `saved_command_line_len` | per-boot cmdline buffer | shared |
| `linux_proc_banner` | per-boot version format | shared |
| `utsname()` (uts_namespace) | per-namespaced uname | shared |
| `get_avenrun()` (kernel/sched/loadavg.c) | per-load-average snapshot | shared |
| `kcpustat_cpu_fetch()` | per-CPU cpustat snapshot | shared |
| `global_node_page_state()` / `global_zone_page_state()` | per-counter atomic read | shared mm-side |
| `si_meminfo()` / `si_swapinfo()` | per-sysinfo aggregate | shared mm-side |
| `pde_make_permanent()` | per-PDE pin (never-freed) | shared proc-side |

## Compatibility contract

REQ-1: /proc/cmdline (fs/proc/cmdline.c):
- `cmdline_proc_show(m, v)`:
  - `seq_puts(m, saved_command_line)`.
  - `seq_putc(m, '\n')`.
  - return 0.
- `proc_cmdline_init` (fs_initcall):
  - pde = `proc_create_single("cmdline", 0, NULL, cmdline_proc_show)`.
  - `pde_make_permanent(pde)`.
  - `pde->size = saved_command_line_len + 1`.

REQ-2: /proc/version (fs/proc/version.c):
- `version_proc_show(m, v)`:
  - `seq_printf(m, linux_proc_banner, utsname()->sysname, utsname()->release, utsname()->version)`.
- `proc_version_init` (fs_initcall):
  - pde = `proc_create_single("version", 0, NULL, version_proc_show)`.
  - `pde_make_permanent(pde)`.
- Per-uts_namespace: each container sees its own `utsname()->release` per `CLONE_NEWUTS`.

REQ-3: /proc/uptime (fs/proc/uptime.c):
- `uptime_proc_show(m, v)`:
  - idle_nsec = 0.
  - for_each_possible_cpu(i):
    - `kcpustat_cpu_fetch(&kcs, i)`.
    - idle_nsec += `get_idle_time(&kcs, i)`.
  - `ktime_get_boottime_ts64(&uptime)`.
  - `timens_add_boottime(&uptime)` (per-time_namespace offset).
  - idle.tv_sec = idle_nsec / NSEC_PER_SEC; idle.tv_nsec = idle_nsec % NSEC_PER_SEC.
  - `seq_printf(m, "%lu.%02lu %lu.%02lu\n", uptime.tv_sec, uptime.tv_nsec/(NSEC_PER_SEC/100), idle.tv_sec, idle.tv_nsec/(NSEC_PER_SEC/100))`.
- Per-time_namespace: each container sees boottime offset by `timens` boottime offset (CLOCK_BOOTTIME virtualization).

REQ-4: /proc/loadavg (fs/proc/loadavg.c):
- `loadavg_proc_show(m, v)`:
  - `get_avenrun(avnrun, FIXED_1/200, 0)` (snapshot, +0.005 rounding).
  - `seq_printf(m, "%lu.%02lu %lu.%02lu %lu.%02lu %u/%d %d\n", LOAD_INT(avnrun[0]), LOAD_FRAC(avnrun[0]), LOAD_INT(avnrun[1]), LOAD_FRAC(avnrun[1]), LOAD_INT(avnrun[2]), LOAD_FRAC(avnrun[2]), nr_running(), nr_threads, idr_get_cursor(&task_active_pid_ns(current)->idr) - 1)`.
- Per-pid_namespace: 5th column (last-pid cursor) is per-namespace via `task_active_pid_ns(current)->idr`.

REQ-5: /proc/meminfo (fs/proc/meminfo.c):
- `meminfo_proc_show(m, v)`:
  - `si_meminfo(&i)` — fills totalram, freeram, sharedram, bufferram, totalhigh, freehigh.
  - `si_swapinfo(&i)` — fills totalswap, freeswap.
  - committed = `vm_memory_committed()`.
  - cached = `global_node_page_state(NR_FILE_PAGES) - total_swapcache_pages() - i.bufferram`; clamp to 0.
  - for lru in [LRU_BASE..NR_LRU_LISTS): pages[lru] = `global_node_page_state(NR_LRU_BASE + lru)`.
  - available = `si_mem_available()`.
  - sreclaimable = `global_node_page_state_pages(NR_SLAB_RECLAIMABLE_B)`.
  - sunreclaim = `global_node_page_state_pages(NR_SLAB_UNRECLAIMABLE_B)`.
  - `show_val_kb` rows in fixed order: `MemTotal:` / `MemFree:` / `MemAvailable:` / `Buffers:` / `Cached:` / `SwapCached:` / `Active:` / `Inactive:` / `Active(anon):` / `Inactive(anon):` / `Active(file):` / `Inactive(file):` / `Unevictable:` / `Mlocked:` / (CONFIG_HIGHMEM: `HighTotal:` / `HighFree:` / `LowTotal:` / `LowFree:`) / `SwapTotal:` / `SwapFree:` / (CONFIG_ZSWAP: `Zswap:` / `Zswapped:`) / `Dirty:` / `Writeback:` / `AnonPages:` / `Mapped:` / `Shmem:` / `KReclaimable:` / `Slab:` / `SReclaimable:` / `SUnreclaim:` / `KernelStack:` / (CONFIG_SHADOW_CALL_STACK: `ShadowCallStack:`) / `PageTables:` / `SecPageTables:` / `NFS_Unstable:` (always 0) / `Bounce:` (always 0) / `WritebackTmp:` (always 0) / `CommitLimit:` / `Committed_AS:` / `VmallocTotal:` / `VmallocUsed:` / `VmallocChunk:` (always 0) / `Percpu:`.
  - `memtest_report_meminfo(m)`.
  - (CONFIG_MEMORY_FAILURE: `HardwareCorrupted:`).
  - (CONFIG_TRANSPARENT_HUGEPAGE: `AnonHugePages:` / `ShmemHugePages:` / `ShmemPmdMapped:` / `FileHugePages:` / `FilePmdMapped:`).
  - (CONFIG_CMA: `CmaTotal:` / `CmaFree:`).
  - (CONFIG_UNACCEPTED_MEMORY: `Unaccepted:`).
  - `Balloon:` / `GPUActive:` / `GPUReclaim:`.
  - `hugetlb_report_meminfo(m)` — appends `HugePages_Total:` / `HugePages_Free:` / `HugePages_Rsvd:` / `HugePages_Surp:` / `Hugepagesize:` / `Hugetlb:`.
  - `arch_report_meminfo(m)` — per-arch (e.g. x86 may append `DirectMap4k:` / `DirectMap2M:` / `DirectMap1G:`).
- `show_val_kb(m, label, num)`:
  - `seq_put_decimal_ull_width(m, label, num << (PAGE_SHIFT - 10), 8)`.
  - `seq_write(m, " kB\n", 4)`.

REQ-6: /proc/stat (fs/proc/stat.c):
- `show_stat(p, v)`:
  - Initialize accumulators to 0; `getboottime64(&boottime)`; `timens_sub_boottime(&boottime)`.
  - for_each_possible_cpu(i):
    - `kcpustat_cpu_fetch(&kcpustat, i)`.
    - Accumulate user/nice/system/idle/iowait/irq/softirq/steal/guest/guest_nice (idle via `get_idle_time`; iowait via `get_iowait_time`).
    - sum += `kstat_cpu_irqs_sum(i)` + `arch_irq_stat_cpu(i)`.
    - for j in [0..NR_SOFTIRQS): per_softirq_sums[j] += `kstat_softirqs_cpu(j, i)`; sum_softirq += same.
  - sum += `arch_irq_stat()`.
  - Print `cpu  user nice system idle iowait irq softirq steal guest guest_nice\n` (note double-space) in `nsec_to_clock_t` units.
  - for_each_online_cpu(i): print `cpuN ...` row (single-space).
  - `intr <sum> ...` line: total then per-IRQ counters via `show_all_irqs` (uses `kstat_irqs_usr(i)` and `show_irq_gap` for unused IRQs).
  - `ctxt <nr_context_switches()>\n`.
  - `btime <boottime.tv_sec>\n` (timens-adjusted).
  - `processes <total_forks>\n`.
  - `procs_running <nr_running()>\n`.
  - `procs_blocked <nr_iowait()>\n`.
  - `softirq <sum_softirq> ...` per-NR_SOFTIRQS counts.
- `stat_open(inode, file)`:
  - size = 1024 + 128 * `num_online_cpus()` + 2 * `irq_get_nr_irqs()`.
  - return `single_open_size(file, show_stat, NULL, size)`.
- `stat_proc_ops`: `.proc_flags = PROC_ENTRY_PERMANENT`, `.proc_open = stat_open`, `.proc_read_iter = seq_read_iter`, `.proc_lseek = seq_lseek`, `.proc_release = single_release`.

REQ-7: /proc/cpuinfo (fs/proc/cpuinfo.c):
- `cpuinfo_open(inode, file)`:
  - return `seq_open(file, &cpuinfo_op)`.
- `cpuinfo_op` is defined per-architecture (e.g. `arch/x86/kernel/cpu/proc.c`, `arch/arm64/kernel/cpuinfo.c`).
- `cpuinfo_proc_ops`: `.proc_flags = PROC_ENTRY_PERMANENT`, `.proc_open = cpuinfo_open`, `.proc_read_iter = seq_read_iter`, `.proc_lseek = seq_lseek`, `.proc_release = seq_release`.

REQ-8: /proc/interrupts (fs/proc/interrupts.c):
- `int_seq_start(f, *pos)`: return `*pos <= irq_get_nr_irqs() ? pos : NULL`.
- `int_seq_next(f, v, *pos)`: ++*pos; return `*pos > irq_get_nr_irqs() ? NULL : pos`.
- `int_seq_stop(f, v)`: nothing.
- `int_seq_ops.show = show_interrupts` (defined in `kernel/irq/proc.c`).
- `proc_interrupts_init` registers via `proc_create_seq`.

REQ-9: /proc/softirqs (fs/proc/softirqs.c):
- `show_softirqs(m, v)`:
  - Header row "                CPU0       CPU1       ...".
  - For each softirq in 0..NR_SOFTIRQS: `seq_printf(m, "%12s:", softirq_to_name[i])` then per-CPU `kstat_softirqs_cpu(i, j)`.

REQ-10: /proc/consoles (fs/proc/consoles.c):
- Iterates `console_list`; per-row format: name + index + flags ("EpNbac" mask: enabled/preferred/console/braille/anyc/cmdline) + read/write pointer hex + driver name.

REQ-11: /proc/devices (fs/proc/devices.c):
- Iterates `chrdevs[]` + `bdev_map[]` per-major; emits `Character devices:` section then `Block devices:` section.

REQ-12: /proc/buddyinfo + /proc/pagetypeinfo (mm/page_alloc.c):
- `proc_create_seq("buddyinfo", 0444, NULL, &fragmentation_op)`.
- `proc_create_seq("pagetypeinfo", 0400, NULL, &pagetypeinfo_op)`.
- buddyinfo: per-`Node N, zone Z`: free-list count at each `order` (0..MAX_ORDER).
- pagetypeinfo: per-migratetype (Unmovable / Movable / Reclaimable / HighAtomic / CMA / Isolate) per-`order` histogram.

REQ-13: /proc/zoneinfo + /proc/vmstat (mm/vmstat.c):
- `proc_create_seq("vmstat", 0444, NULL, &vmstat_op)`.
- `proc_create_seq("zoneinfo", 0444, NULL, &zoneinfo_op)`.
- vmstat: flat `name value` rows, one per NR_VM_ZONE_STAT_ITEMS + NR_VM_NODE_STAT_ITEMS + NR_VM_EVENT_ITEMS.
- zoneinfo: per-Node-per-Zone: `pages free / min / low / high / spanned / present / managed`, watermarks, per-LRU counters, vm_stat counters.

REQ-14: /proc/vmallocinfo (mm/vmalloc.c):
- `proc_create_single("vmallocinfo", 0400, NULL, vmalloc_info_show)`.
- Iterates `vmap_area_list`; per-vmap_area row: `addr-end   size   caller [pages=N] [phys=...] [ioremap|vmalloc|vmap|user|vpages]`.
- 0400 (root-only): leaks kernel addresses (used by `kptr_restrict` consumers).

REQ-15: /proc/swaps (mm/swapfile.c):
- `proc_create("swaps", 0, NULL, &swaps_proc_ops)`.
- Iterates `swap_info[]`; per-row: `Filename Type Size Used Priority`.

REQ-16: /proc/diskstats + /proc/partitions (block/genhd.c):
- `proc_create_seq("diskstats", 0, NULL, &diskstats_op)`.
- `proc_create_seq("partitions", 0, NULL, &partitions_op)`.
- diskstats: per-block-device 17-column counters (reads-completed / reads-merged / sectors-read / read-time-ms / writes-completed / writes-merged / sectors-written / write-time-ms / inflight / io_ticks / time-in-queue / discards-completed / discards-merged / sectors-discarded / discard-time-ms / flush-completed / flush-time-ms).
- partitions: `major minor #blocks name`.

REQ-17: Per-`fs_initcall` registration ordering:
- All 18 endpoints register at `fs_initcall` level, after `vfs_caches_init` and `proc_root_init`.
- Each calls `proc_create_single` / `proc_create_seq` / `proc_create` against the root `procfs` namespace (`NULL` parent).
- `pde_make_permanent(pde)` marks the entry as never-freeable (cannot be removed via `remove_proc_entry`).

REQ-18: Per-mode bits:
- Most files: mode = 0 ⟹ defaults to 0444 (world-readable).
- `pagetypeinfo`: 0400 (root-only — page-type leak).
- `vmallocinfo`: 0400 (root-only — kptr leak).
- Other files explicitly set 0444 via `proc_create_seq`'s mode arg.

## Acceptance Criteria

- [ ] AC-1: `read /proc/cmdline` returns `saved_command_line` + '\n', size matches `saved_command_line_len + 1`.
- [ ] AC-2: `read /proc/version` returns `linux_proc_banner` formatted with current `utsname()`, per-uts_namespace.
- [ ] AC-3: `read /proc/uptime` returns "%.2f %.2f" with system uptime + cumulative idle time across all CPUs; per-time_namespace boottime offset applied.
- [ ] AC-4: `read /proc/loadavg` returns five fields: load1 / load5 / load15 / running/threads / last-pid; last-pid per-pid_namespace.
- [ ] AC-5: `read /proc/meminfo` returns the canonical ordered key list; values in `kB`; CONFIG-gated rows present iff CONFIG is enabled.
- [ ] AC-6: `read /proc/stat` returns `cpu` aggregate row + per-CPU rows + `intr` / `ctxt` / `btime` / `processes` / `procs_running` / `procs_blocked` / `softirq`; jiffies in `clock_t` units.
- [ ] AC-7: `read /proc/cpuinfo` invokes per-arch `cpuinfo_op`; iterator returns one logical-CPU block per processor.
- [ ] AC-8: `read /proc/interrupts` invokes `show_interrupts` per-IRQ-nr in range; missing IRQs skipped via `show_irq_gap` zeros.
- [ ] AC-9: `read /proc/softirqs` invokes `show_softirqs`; per-softirq per-CPU counters from `kstat_softirqs_cpu`.
- [ ] AC-10: `read /proc/buddyinfo` invokes `fragmentation_op`; per-zone per-order free-count row.
- [ ] AC-11: `read /proc/vmstat` invokes `vmstat_op`; flat name/value rows.
- [ ] AC-12: `read /proc/vmallocinfo` requires CAP_SYS_ADMIN (mode 0400 + kptr restriction).
- [ ] AC-13: `read /proc/swaps` per-swap-area row.
- [ ] AC-14: `read /proc/diskstats` per-block-device 17-column row.
- [ ] AC-15: `remove_proc_entry("cmdline", NULL)` no-ops (PROC_ENTRY_PERMANENT).

## Architecture

```
struct ProcMisc;                       // ZST collecting per-file glue

// Generic helper used by meminfo
fn show_val_kb(m: *SeqFile, label: &str, num: u64);
                                       // prints "<label> %8lu kB\n" with num<<(PAGE_SHIFT-10)
```

`ProcMisc::cmdline_show(m, _v)`:
1. `seq_puts(m, saved_command_line())`.
2. `seq_putc(m, '\n')`.
3. return Ok(()).

`ProcMisc::cmdline_init()` (fs_initcall):
1. pde = `proc_create_single("cmdline", 0, NULL, ProcMisc::cmdline_show)`.
2. `pde_make_permanent(pde)`.
3. `pde.size = saved_command_line_len() + 1`.

`ProcMisc::version_show(m, _v)`:
1. uts = `utsname()` (per-uts_ns of current).
2. `seq_printf(m, linux_proc_banner, uts.sysname, uts.release, uts.version)`.

`ProcMisc::uptime_show(m, _v)`:
1. idle_nsec = 0_u64.
2. for i in for_each_possible_cpu():
   - kcs = `kcpustat_cpu_fetch(i)`.
   - idle_nsec += `ProcMisc::get_idle_time(&kcs, i)`.
3. uptime = `ktime_get_boottime_ts64()`.
4. `timens_add_boottime(&mut uptime)` — apply per-time_ns offset.
5. idle.tv_sec = idle_nsec / NSEC_PER_SEC; idle.tv_nsec = idle_nsec % NSEC_PER_SEC.
6. `seq_printf(m, "%lu.%02lu %lu.%02lu\n", uptime.tv_sec, uptime.tv_nsec/(NSEC_PER_SEC/100), idle.tv_sec, idle.tv_nsec/(NSEC_PER_SEC/100))`.

`ProcMisc::loadavg_show(m, _v)`:
1. avnrun: [u64; 3].
2. `get_avenrun(&mut avnrun, FIXED_1/200, 0)`.
3. last_pid = `idr_get_cursor(&task_active_pid_ns(current).idr) - 1`.
4. `seq_printf(m, "%lu.%02lu %lu.%02lu %lu.%02lu %u/%d %d\n", LOAD_INT(avnrun[0]), LOAD_FRAC(avnrun[0]), ..., nr_running(), nr_threads, last_pid)`.

`ProcMisc::meminfo_show(m, _v)`:
1. i = sysinfo{}; `si_meminfo(&mut i)`; `si_swapinfo(&mut i)`.
2. committed = `vm_memory_committed()`.
3. cached = max(0, `global_node_page_state(NR_FILE_PAGES) - total_swapcache_pages() - i.bufferram`).
4. pages[0..NR_LRU_LISTS] from `global_node_page_state(NR_LRU_BASE + lru)`.
5. available = `si_mem_available()`.
6. sreclaimable = `global_node_page_state_pages(NR_SLAB_RECLAIMABLE_B)`.
7. sunreclaim = `global_node_page_state_pages(NR_SLAB_UNRECLAIMABLE_B)`.
8. `show_val_kb` rows in canonical order (REQ-5).
9. CONFIG-gated rows (HIGHMEM / NOMMU / ZSWAP / SHADOW_CALL_STACK / MEMORY_FAILURE / TRANSPARENT_HUGEPAGE / CMA / UNACCEPTED_MEMORY).
10. `hugetlb_report_meminfo(m)`.
11. `arch_report_meminfo(m)` (per-arch weak override).

`ProcMisc::stat_show(p, _v)`:
1. Accumulate u64 totals: user / nice / system / idle / iowait / irq / softirq / steal / guest / guest_nice.
2. boottime = `getboottime64()`; `timens_sub_boottime(&mut boottime)`.
3. for i in for_each_possible_cpu():
   - kcpu = `kcpustat_cpu_fetch(i)`.
   - Sum each CPUTIME_* counter; idle via `get_idle_time(&kcpu, i)`; iowait via `get_iowait_time(&kcpu, i)`.
   - sum += `kstat_cpu_irqs_sum(i)` + `arch_irq_stat_cpu(i)`.
   - per_softirq_sums[j] += `kstat_softirqs_cpu(j, i)` for j in 0..NR_SOFTIRQS.
4. sum += `arch_irq_stat()`.
5. Print aggregate `cpu  ` row (double-space).
6. for i in for_each_online_cpu(): print `cpuN` row.
7. `intr <sum>` + `show_all_irqs(p)`.
8. `ctxt <nr_context_switches()>` / `btime <boottime.tv_sec>` / `processes <total_forks>` / `procs_running <nr_running()>` / `procs_blocked <nr_iowait()>`.
9. `softirq <sum_softirq>` + per_softirq_sums[..].

`ProcMisc::stat_open(inode, file)`:
1. size = 1024 + 128 * `num_online_cpus()` + 2 * `irq_get_nr_irqs()`.
2. return `single_open_size(file, stat_show, NULL, size)`.

`ProcMisc::cpuinfo_open(inode, file)`:
1. return `seq_open(file, &cpuinfo_op)` (per-arch).

`ProcMisc::interrupts_seq_ops`:
- `.start = int_seq_start`, `.next = int_seq_next`, `.stop = int_seq_stop`, `.show = show_interrupts`.

`ProcMisc::get_idle_time(kcs, cpu) -> u64`:
1. idle_usecs = if `cpu_online(cpu)` then `get_cpu_idle_time_us(cpu, None)` else `-1_u64`.
2. if idle_usecs == -1: return kcs.cpustat[CPUTIME_IDLE] (NO_HZ-or-offline fallback).
3. else: return idle_usecs * NSEC_PER_USEC.

`ProcMisc::get_iowait_time(kcs, cpu) -> u64`:
1. Mirror of get_idle_time using `get_cpu_iowait_time_us` / `cpustat[CPUTIME_IOWAIT]`.

`ProcMisc::show_all_irqs(p)`:
1. next = 0.
2. for i in for_each_active_irq():
   - `show_irq_gap(p, i - next)`.
   - `seq_put_decimal_ull(p, " ", kstat_irqs_usr(i))`.
   - next = i + 1.
3. `show_irq_gap(p, irq_get_nr_irqs() - next)`.

`ProcMisc::show_irq_gap(p, gap)`:
1. while gap > 0:
   - inc = min(gap, ARRAY_SIZE(zeros)/2).
   - `seq_write(p, zeros, 2*inc)`.
   - gap -= inc.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cmdline_size_matches_pde` | INVARIANT | per-cmdline_init: pde.size == saved_command_line_len + 1 (covers '\n'). |
| `version_uses_current_uts_ns` | INVARIANT | per-version_show: utsname() respects task.nsproxy.uts_ns. |
| `uptime_per_timens_offset_applied` | INVARIANT | per-uptime_show: timens_add_boottime called before printing. |
| `loadavg_last_pid_per_pid_ns` | INVARIANT | per-loadavg_show: idr_get_cursor uses task_active_pid_ns(current). |
| `meminfo_show_val_kb_no_overflow` | INVARIANT | per-show_val_kb: num << (PAGE_SHIFT-10) does not overflow u64 within 2^54 pages. |
| `stat_size_hint_bounds_buffer` | INVARIANT | per-stat_open: single_open_size receives a hint big enough for cpu_count*128 + irq_count*2 + 1024. |
| `interrupts_iter_bounds` | INVARIANT | per-int_seq_next: pos <= irq_get_nr_irqs(). |
| `irq_gap_zeros_fit` | INVARIANT | per-show_irq_gap: inc <= ARRAY_SIZE(zeros)/2 ⟹ seq_write never overshoots zeros[]. |
| `pde_make_permanent_unremovable` | INVARIANT | per-init: remove_proc_entry on permanent PDE no-ops. |
| `vmallocinfo_mode_0400` | INVARIANT | per-vmalloc_init: mode bit pattern == 0400 (CAP_SYS_ADMIN gate). |
| `pagetypeinfo_mode_0400` | INVARIANT | per-page_alloc_init: pagetypeinfo created with 0400. |

### Layer 2: TLA+

`fs/proc/proc-misc.tla`:
- Per-/proc/cmdline + per-/proc/version + per-/proc/uptime + per-/proc/loadavg + per-/proc/meminfo + per-/proc/stat lifecycle.
- Properties:
  - `safety_pde_permanent_never_removed` — per-pde_make_permanent: any remove_proc_entry leaves entry intact.
  - `safety_per_namespace_isolation` — per-version: uts_ns; per-uptime: time_ns; per-loadavg: pid_ns visibility.
  - `safety_meminfo_kb_units` — per-meminfo: all `*_kb` rows are in `kB` (pages << PAGE_SHIFT-10).
  - `safety_stat_clock_t_units` — per-stat: all CPU-time fields are `nsec_to_clock_t()`-converted.
  - `liveness_fs_initcall_registration` — per-fs_initcall ordering: all 18 endpoints registered before `device_initcall`.
  - `liveness_seq_show_terminates` — per-open: seq_read drains in finite steps (bounded by num_online_cpus + irq_get_nr_irqs).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ProcMisc::cmdline_show` post: emitted bytes == saved_command_line + "\n" | `ProcMisc::cmdline_show` |
| `ProcMisc::version_show` post: emitted == linux_proc_banner % (sysname, release, version) | `ProcMisc::version_show` |
| `ProcMisc::uptime_show` post: idle_nsec == Σ get_idle_time(kcpustat[i], i); timens-adjusted boottime | `ProcMisc::uptime_show` |
| `ProcMisc::loadavg_show` post: 5 fields; last_pid >= 0 | `ProcMisc::loadavg_show` |
| `ProcMisc::meminfo_show` post: row-set is a prefix-superset of canonical labels per CONFIG | `ProcMisc::meminfo_show` |
| `ProcMisc::stat_show` post: cpu-aggregate == Σ percpu rows; per_softirq_sums[j] == Σ kstat_softirqs_cpu(j, i) | `ProcMisc::stat_show` |
| `ProcMisc::stat_open` post: single_open_size buffer hint >= 1024 + 128*num_online + 2*nr_irqs | `ProcMisc::stat_open` |
| `ProcMisc::get_idle_time` post: returns CPUTIME_IDLE on NO_HZ-off / offline; else microseconds*1000 | `ProcMisc::get_idle_time` |

### Layer 4: Verus/Creusot functional

`Per-/proc/<file> open → seq_read_iter → show_fn drain → close` semantic equivalence: per-Documentation/filesystems/proc.rst tables; field-by-field text equality with upstream output on identical kernel-state snapshot.

## Hardening

(Inherits row-1 features from `fs/proc/00-overview.md` § Hardening.)

proc-misc reinforcement:

- **Per-`pde_make_permanent` for all 18 entries** — defense against per-rmmod / per-`remove_proc_entry` accidental removal of system-critical files.
- **Per-`0400` mode for vmallocinfo / pagetypeinfo** — defense against per-unprivileged kptr-leak (vmallocinfo prints raw `addr-end` + caller PC; pagetypeinfo reveals migratetype layout for heap-grooming).
- **Per-uts_namespace for /proc/version** — defense against per-container host-kernel-version leak (`utsname()` per-`current->nsproxy->uts_ns`).
- **Per-time_namespace for /proc/uptime** — defense against per-container host-uptime-leak (CLOCK_BOOTTIME virtualization via `timens_add_boottime`).
- **Per-pid_namespace for /proc/loadavg last-pid** — defense against per-container host-pid-cursor leak.
- **Per-`single_open_size` precomputed buffer hint for /proc/stat** — defense against per-realloc-storm under large `num_online_cpus()` × `irq_get_nr_irqs()`.
- **Per-`kcpustat_cpu_fetch` snapshot semantics** — defense against per-torn-read of u64 cpustat counters on 32-bit.
- **Per-`kptr_restrict` honored by `%pK` in vmallocinfo** — defense against per-printk-leak of kernel addresses.
- **Per-`seq_file` lseek capped** — defense against per-seek-overflow on /proc/meminfo, /proc/stat.
- **Per-`get_idle_time` NO_HZ-aware** — defense against per-stale-idle on tickless CPUs.
- **Per-`global_node_page_state` atomic read** — defense against per-torn meminfo row.
- **Per-`PROC_ENTRY_PERMANENT` proc_ops flag for /proc/stat, /proc/cpuinfo** — defense against per-late-unregister race.
- **Per-`show_irq_gap` zeros-array bounded write** — defense against per-OOB-write in IRQ-gap fill.

## Grsecurity/PaX-style Reinforcement

- PAX_USERCOPY: every per-misc-file `seq_printf`/`seq_write` flushes through `seq_buf` slab-whitelisted into user space; /proc/loadavg, /proc/uptime, /proc/version, /proc/cpuinfo and /proc/meminfo cannot exfiltrate adjacent allocator metadata.
- PAX_KERNEXEC: misc-show callbacks (`loadavg_proc_show`, `uptime_proc_show`, `version_proc_show`, `meminfo_proc_show`) execute with WP asserted; a stray `seq_printf` argument-list mismatch cannot patch kernel text.
- PAX_RANDKSTACK: per-open kernel stack for misc seq-iterators is re-randomized; cpuinfo timing oracles cannot infer fixed stack offsets across reads.
- PAX_REFCOUNT: `pde_users` and `proc_inode::ent` refcounts on each misc entry use `refcount_t` with saturation; concurrent `pde_remove` cannot underflow against unbounded /proc readers.
- PAX_MEMORY_SANITIZE: seq-buffer slabs (`seq_buf`, kmalloc-1k power-of-two) are poisoned on free so a subsequent /proc/meminfo read cannot recover a prior /proc/cpuinfo dump from a recycled page.
- PAX_UDEREF: `seq_read_iter`'s final user copy honors SMAP/PAN; a faulting userspace mapping cannot trick the kernel into a write-side oracle.
- PAX_RAP / kCFI: each misc PDE installs its `proc_ops` via signature-validated registration; an attacker cannot swap `loadavg_proc_show` for `version_proc_show` to bypass capability gating.
- GRKERNSEC_HIDESYM: `/proc/version` masks any kernel pointer (`%pK`) for callers without `CAP_SYSLOG`; build-id, GCC version string, and link timestamp are emitted, raw symbol addresses are not.
- GRKERNSEC_DMESG: rejected misc opens (lockdown, capability) are rate-limited so a fuzzer cannot flood console.
- Per-file CAP gating: /proc/kallsyms requires `CAP_SYSLOG` for non-zero address resolution; /proc/slabinfo requires `CAP_SYS_ADMIN` for write of `force_rcu` toggles; /proc/buddyinfo and /proc/zoneinfo expose only ns-visible zones under user-ns scoping.
- /proc/uptime jitter: idle-time accumulator is read with `seqcount_t` retry under PaX-validated barriers so wraparound during a torn read yields `-EAGAIN` rather than negative deltas.
- /proc/version HIDESYM enforcement: UTS namespace strings are bounded by `__NEW_UTS_LEN`; a forged `setdomainname()` cannot inject newline-smuggled console-command sequences into a /proc/version read.
- Audit: misc-file opens by non-`CAP_SYS_ADMIN` callers under lockdown-confidentiality are logged with the rejected pathname and capability set, independent of dmesg suppression.

## Open Questions

- /proc/swaps formatting under suspended-swap (swap-area frozen during hibernate): is the row hidden or shown stale?
- /proc/buddyinfo: per-`MAX_PAGE_ORDER` rename + columns count under future MM-tuning.
- /proc/zoneinfo: per-`per_node_stat` ordering vs upstream — pinned by Documentation but not test-asserted.
- /proc/cpuinfo per-arch: ARM64 / RISC-V layouts vs x86 layouts — should the Rookery cpuinfo_op be per-arch (mirror upstream) or unified?

## Out of Scope

- `fs/proc/proc-generic.md` Tier-3 — owns `proc_create_*`, `pde_make_permanent`, `single_open_size`, `seq_read_iter` plumbing.
- `kernel/printk/00-overview.md` — owns `linux_proc_banner` definition.
- `kernel/sched/00-overview.md` — owns `nr_running`, `nr_iowait`, `total_forks`, `get_avenrun`, `kstat_cpu_irqs_sum`, `kcpustat_cpu_fetch` semantics.
- `mm/00-overview.md` — owns `si_meminfo`, `si_swapinfo`, `global_node_page_state`, `si_mem_available`, `vm_memory_committed`, `vm_commit_limit`.
- `mm/page-allocator.md` — owns `fragmentation_op` / `pagetypeinfo_op` content.
- `mm/reclaim.md` — owns vmstat counter semantics consumed by /proc/vmstat.
- `mm/vmalloc.md` — owns `vmalloc_info_show` and the vmap_area structure walked.
- `mm/swapfile.md` — owns `swaps_proc_ops` content and `swap_info` structures.
- `block/00-overview.md` — owns `diskstats_op` / `partitions_op` content.
- `kernel/irq/00-overview.md` — owns `show_interrupts` per-arch + `irq_get_nr_irqs`.
- Per-arch cpuinfo formatting (`arch/x86/kernel/cpu/proc.c`, etc.) — covered in per-arch Tier-3 docs.
- `/proc/<pid>/...` per-task files — covered in `proc-pid.md` / `proc-task.md` / `array.md`.
- `/proc/sys/...` sysctl tree — covered in `proc-sysctl.md`.
- `/proc/kcore` — covered in `proc-kcore.md`.
- `/proc/kmsg` — covered in `proc-kmsg.md`.
- Implementation code.
