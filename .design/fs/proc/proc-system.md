# Tier-3: fs/proc/proc-system — system-wide procfs entries (cpuinfo / meminfo / loadavg / etc.)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - fs/proc/cpuinfo.c
  - fs/proc/meminfo.c
  - fs/proc/loadavg.c
  - fs/proc/uptime.c
  - fs/proc/stat.c
  - fs/proc/cmdline.c
  - fs/proc/version.c
  - fs/proc/devices.c
  - fs/proc/interrupts.c
  - fs/proc/softirqs.c
  - fs/proc/consoles.c
  - fs/proc/page.c
-->

## Summary
Tier-3 design for the system-wide (non-per-task) `/proc/*` entries — the scalar / aggregate kernel-state files consumed by every system-monitoring tool. Format byte-identical compatibility is critical: `top` / `htop` / `vmstat` / `free` / `mpstat` / `dstat` / `sar` / `tload` / `uptime` / `who` / `last` / `lscpu` all parse these files.

Owns the bulk of `/proc/*` non-pid files:
- `/proc/cpuinfo` (per-CPU info, format per-arch via `arch/x86/kernel-platform.md`)
- `/proc/meminfo` (system-wide memory)
- `/proc/loadavg` (1/5/15-min load averages)
- `/proc/uptime` (uptime + idle time)
- `/proc/stat` (per-CPU + system stats)
- `/proc/cmdline` (kernel boot cmdline)
- `/proc/version` (kernel version string)
- `/proc/devices` (registered char + block major numbers)
- `/proc/interrupts` (per-CPU per-IRQ counts)
- `/proc/softirqs` (per-CPU per-softirq counts)
- `/proc/consoles` (registered console drivers)
- `/proc/kpagecount`, `/proc/kpageflags`, `/proc/kpagecgroup` (per-page state — `page.c`)

Sub-tier-3 of `fs/proc/00-overview.md`. Sibling of `fs/proc/proc-task.md`. The "consult the kernel for system state" surface.

## Upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| /proc/cpuinfo (per-arch hooks) | `fs/proc/cpuinfo.c` |
| /proc/meminfo | `fs/proc/meminfo.c` |
| /proc/loadavg | `fs/proc/loadavg.c` |
| /proc/uptime | `fs/proc/uptime.c` |
| /proc/stat (per-CPU + system) | `fs/proc/stat.c` |
| /proc/cmdline | `fs/proc/cmdline.c` |
| /proc/version | `fs/proc/version.c` |
| /proc/devices | `fs/proc/devices.c` |
| /proc/interrupts (per-IRQ per-CPU) | `fs/proc/interrupts.c` |
| /proc/softirqs (per-softirq per-CPU) | `fs/proc/softirqs.c` |
| /proc/consoles | `fs/proc/consoles.c` |
| /proc/kpagecount + /proc/kpageflags + /proc/kpagecgroup | `fs/proc/page.c` |

## Compatibility contract

### `/proc/cpuinfo`

Per-CPU multi-line key:value record per CPU, terminated by blank line. x86_64 fields:
```
processor	: <logical-CPU-index>
vendor_id	: GenuineIntel | AuthenticAMD | ...
cpu family	: <decimal>
model		: <decimal>
model name	: <full string>
stepping	: <decimal>
microcode	: 0x<hex>
cpu MHz		: <freq>
cache size	: <size> KB
physical id	: <socket-id>
siblings	: <thread-count>
core id		: <core-index>
cpu cores	: <core-count>
apicid		: <decimal>
initial apicid	: <decimal>
fpu		: yes
fpu_exception	: yes
cpuid level	: <decimal>
wp		: yes
flags		: <space-separated CPUID feature names>
vmx flags	: <space-separated VMX features>  [if Intel + VMX]
bugs		: <space-separated CVE/erratum names>
bogomips	: <decimal>
clflush size	: <decimal>
cache_alignment	: <decimal>
address sizes	: <virt> bits virtual, <phys> bits physical
power management: <space-separated>
```

Format per `arch/x86/kernel/cpu/proc.c` `show_cpuinfo`; `fs/proc/cpuinfo.c` provides only generic dispatch hook.

ARM64 / RISC-V / etc. have different fields. Per-arch identical to upstream's per-arch implementation.

### `/proc/meminfo`

Multi-line key:value (always with `kB` units except where noted):
```
MemTotal:	<kB>
MemFree:	<kB>
MemAvailable:	<kB>
Buffers:	<kB>
Cached:		<kB>
SwapCached:	<kB>
Active:		<kB>
Inactive:	<kB>
Active(anon):	<kB>
Inactive(anon):	<kB>
Active(file):	<kB>
Inactive(file):	<kB>
Unevictable:	<kB>
Mlocked:	<kB>
SwapTotal:	<kB>
SwapFree:	<kB>
Zswap:		<kB>
Zswapped:	<kB>
Dirty:		<kB>
Writeback:	<kB>
AnonPages:	<kB>
Mapped:		<kB>
Shmem:		<kB>
KReclaimable:	<kB>
Slab:		<kB>
SReclaimable:	<kB>
SUnreclaim:	<kB>
KernelStack:	<kB>
PageTables:	<kB>
SecPageTables:	<kB>
NFS_Unstable:	<kB>
Bounce:		<kB>
WritebackTmp:	<kB>
CommitLimit:	<kB>
Committed_AS:	<kB>
VmallocTotal:	<kB>
VmallocUsed:	<kB>
VmallocChunk:	<kB>
Percpu:		<kB>
HardwareCorrupted:	<kB>
AnonHugePages:	<kB>
ShmemHugePages:	<kB>
ShmemPmdMapped:	<kB>
FileHugePages:	<kB>
FilePmdMapped:	<kB>
CmaTotal:	<kB>
CmaFree:	<kB>
HugePages_Total:	<count>
HugePages_Free:		<count>
HugePages_Rsvd:		<count>
HugePages_Surp:		<count>
Hugepagesize:	<kB>
Hugetlb:	<kB>
DirectMap4k:	<kB>
DirectMap2M:	<kB>
DirectMap1G:	<kB>
```

Format byte-identical (every field name + tab + value-justified-right). Used by `free`/`top`/`htop`/`vmstat -s`.

### `/proc/loadavg`

Single line:
```
<1min> <5min> <15min> <running>/<total> <last_pid>
```

Format byte-identical (3 floats with 2 decimal places, then "N/M", then last-allocated pid). Used by `uptime`.

### `/proc/uptime`

Single line:
```
<uptime_sec>.<uptime_centisec> <idle_sec>.<idle_centisec>
```

Format byte-identical (two floats with 2 decimal places).

### `/proc/stat`

System-wide + per-CPU stats:
```
cpu  user nice system idle iowait irq softirq steal guest guest_nice
cpu0 user0 nice0 system0 idle0 iowait0 irq0 softirq0 steal0 guest0 guest_nice0
cpu1 ...
...
intr <total> <per-IRQ counts>
ctxt <total>
btime <boot_unix_time>
processes <total fork count>
procs_running <current>
procs_blocked <current>
softirq <total> <per-softirq counts>
```

Times in USER_HZ ticks (typically jiffies). Format byte-identical (used by `mpstat`, `top`, `htop`, `vmstat`).

### `/proc/cmdline`

Single line: kernel boot command-line as passed by bootloader. Format byte-identical.

### `/proc/version`

Single line:
```
Linux version <kver> (<builder>@<host>) (gcc version <gcc_ver>) #<n> SMP <date>
```

Format byte-identical.

### `/proc/devices`

Two-section list of registered char/block major numbers:
```
Character devices:
  1 mem
  2 pty
  ...
  
Block devices:
  1 ramdisk
  7 loop
  8 sd
  ...
```

Used by `mknod` / `lsblk` / debugging.

### `/proc/interrupts`

Per-IRQ per-CPU table:
```
            CPU0       CPU1       CPU2       CPU3
   0:        125          0          0          0   IO-APIC   2-edge      timer
   1:          0          0          0          5   IO-APIC   1-edge      i8042
   ...
 NMI:          0          0          0          0   Non-maskable interrupts
 LOC:          1234       5678       9012       3456 Local timer interrupts
 ...
```

Per-CPU column count + alignment + per-IRQ row per upstream's `show_interrupts`. Used by `lscpu` / `mpstat -I`.

### `/proc/softirqs`

Per-softirq per-CPU table similar to /proc/interrupts:
```
                    CPU0       CPU1
          HI:          0          0
       TIMER:        100        200
      NET_TX:          0          0
      NET_RX:        500       1000
       BLOCK:         50        100
   IRQ_POLL:          0          0
     TASKLET:         10         20
       SCHED:        100        200
     HRTIMER:          0          0
         RCU:        500       1000
```

10 standard softirqs. Format byte-identical.

### `/proc/consoles`

Per-console-driver multi-line:
```
<name><index>     <flags>     (<status>)    <type>:<minor>
ttyS0            -W- (EC      )    4:64
tty0             -WU (E       )    4:1
```

Used by `dmesg --console`-aware tools.

### `/proc/kpagecount`, `/proc/kpageflags`, `/proc/kpagecgroup`

Binary 8-bytes-per-page mappings:
- `kpagecount`: per-PFN refcount
- `kpageflags`: per-PFN flag bitmap (PG_locked / PG_referenced / PG_uptodate / PG_dirty / PG_lru / PG_active / PG_slab / PG_writeback / etc.)
- `kpagecgroup`: per-PFN owning cgroup-id

Used by `page-types` / VM-debugging tools. Gated by CAP_SYS_ADMIN.

## Requirements

- REQ-1: `/proc/cpuinfo` per-CPU format byte-identical to per-arch upstream (x86_64 / ARM64 / RISC-V).
- REQ-2: `/proc/meminfo` field-list + format byte-identical.
- REQ-3: `/proc/loadavg` 5-field format byte-identical.
- REQ-4: `/proc/uptime` 2-field format byte-identical.
- REQ-5: `/proc/stat` per-CPU + system + intr + ctxt + btime + processes + procs_running + procs_blocked + softirq format byte-identical.
- REQ-6: `/proc/cmdline` byte-identical to boot-time cmdline.
- REQ-7: `/proc/version` format byte-identical.
- REQ-8: `/proc/devices` two-section format byte-identical.
- REQ-9: `/proc/interrupts` per-IRQ per-CPU table format byte-identical.
- REQ-10: `/proc/softirqs` 10-softirq table format byte-identical.
- REQ-11: `/proc/consoles` per-driver format byte-identical.
- REQ-12: `/proc/kpagecount` / `/kpageflags` / `/kpagecgroup` 8-bytes-per-page binary format byte-identical; CAP_SYS_ADMIN gated.
- REQ-13: All entries are seq_file-based, one-shot snapshot per read; concurrent readers see consistent view per-call.
- REQ-14: Hardening section per `00-security-principles.md` template.

## Acceptance Criteria

- [ ] AC-1: `cat /proc/cpuinfo` byte-identical to upstream's on x86_64; `lscpu` output identical. (covers REQ-1)
- [ ] AC-2: `cat /proc/meminfo` byte-identical; `free -m` output identical. (covers REQ-2)
- [ ] AC-3: `cat /proc/loadavg` byte-identical; `uptime` output identical. (covers REQ-3)
- [ ] AC-4: `cat /proc/uptime` byte-identical. (covers REQ-4)
- [ ] AC-5: `cat /proc/stat` byte-identical; `mpstat` parses correctly. (covers REQ-5)
- [ ] AC-6: `cat /proc/cmdline` byte-identical to bootloader-provided cmdline. (covers REQ-6)
- [ ] AC-7: `uname -a` output (which reads /proc/version) identical. (covers REQ-7)
- [ ] AC-8: `cat /proc/devices` shows current char + block majors. (covers REQ-8)
- [ ] AC-9: `cat /proc/interrupts` byte-identical; `mpstat -I CPU` parses. (covers REQ-9)
- [ ] AC-10: `cat /proc/softirqs` byte-identical with 10 softirq rows + per-CPU columns. (covers REQ-10)
- [ ] AC-11: `cat /proc/consoles` byte-identical content. (covers REQ-11)
- [ ] AC-12: kpagecount test: `head -c 8 /proc/kpagecount` from PFN 0 returns LE-u64 refcount; non-CAP_SYS_ADMIN read returns 0 bytes (gated). (covers REQ-12)
- [ ] AC-13: Snapshot consistency test: 1000 concurrent reads of /proc/meminfo; no half-updated values. (covers REQ-13)
- [ ] AC-14: Hardening section present and follows template. (covers REQ-14)

## Architecture

### Rust module organization

- `kernel::fs::proc::system::cpuinfo::CpuInfo` — `/proc/cpuinfo` (per-arch hook)
- `kernel::fs::proc::system::meminfo::MemInfo`
- `kernel::fs::proc::system::loadavg::LoadAvg`
- `kernel::fs::proc::system::uptime::Uptime`
- `kernel::fs::proc::system::stat::Stat`
- `kernel::fs::proc::system::cmdline::Cmdline`
- `kernel::fs::proc::system::version::Version`
- `kernel::fs::proc::system::devices::Devices`
- `kernel::fs::proc::system::interrupts::Interrupts`
- `kernel::fs::proc::system::softirqs::Softirqs`
- `kernel::fs::proc::system::consoles::Consoles`
- `kernel::fs::proc::system::page::Kpagecount`, `Kpageflags`, `Kpagecgroup`

### Locking and concurrency

- **Per-file `seq_file` infrastructure**: per-read snapshot via `seq_*` accumulators
- **Per-cpu stats reads** lockless via per-CPU atomics; aggregated into seq_file output
- **`/proc/interrupts` per-IRQ-per-CPU**: per-IRQ refcount + per-CPU stat snapshot

### Error handling

- `Err(EACCES)` — kpagecount/flags without CAP_SYS_ADMIN
- `Err(EFAULT)` — userspace buffer fault
- `Err(EINVAL)` — bad offset for binary kpagecount/flags

## Verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-file seq_file output (no buffer-overrun on long lines) | `kani::proofs::fs::proc::system::seq_safety` |
| Per-CPU stat aggregation arithmetic (no overflow on 64-bit jiffies sum) | `kani::proofs::fs::proc::system::stat_aggregate_safety` |
| kpagecount/flags binary read (offset+size bounds-check) | `kani::proofs::fs::proc::system::page_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-file output | line-count + format matches expected per-entry; no garbage chars | `kani::proofs::fs::proc::system::format_invariants` |
| Per-CPU stat snapshot | sum of per-CPU values = system aggregate at each snapshot | `kani::proofs::fs::proc::system::aggregate_invariants` |

### Layer 4: Functional correctness (opt-in)

- **Format-stability theorem** via Verus — proves: per-file output adheres to upstream-documented format for any system state.

## Hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **MEMORY_SANITIZE** | freed seq_file buffers cleared (some carry sensitive data — kallsyms partial output, kcore content) | § Default-on configurable off |
| **CONSTIFY** | per-file `proc_ops` vtables `static const` | § Mandatory |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE, CONSTIFY**: see above + per-seq_file slab cache
- **USERCOPY**: seq_file reads use `simple_read_from_buffer` / `seq_*` bound-checked
- **SIZE_OVERFLOW**: per-file aggregator arithmetic uses checked operators
- **KERNEXEC**: per-file dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_inode_permission` — per-file access gate.
- Default useful GR-RBAC policy: deny `/proc/kcore` read outside gradm-marked `kernel_debugger` role; deny `/proc/kpagecount` outside `vm_admin`.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

## Grsecurity/PaX-style Reinforcement

/proc/{hostname,version,uptime,loadavg,meminfo,cmdline,kallsyms,kcore,kpagecount,kpageflags,...} are system-wide oracles; grsec floors most of them at `CAP_SYSLOG` or `kernel_admin`. Rookery contract:

- **PAX_USERCOPY** — every seq_file emitter copies via `seq_*` APIs that track buffer size; `simple_read_from_buffer` is bounded by the source allocation.
- **PAX_KERNEXEC** — per-file `proc_ops` vtables are `static const`; the registration table in `proc_root_init` is `__ro_after_init`.
- **PAX_RANDKSTACK** — emitter paths run under randomized kstack offset; per-build kstack-shape primitives are denied.
- **PAX_REFCOUNT** — per-pde refcount and per-seq_file ref saturate; aggregator paths use atomic_t with overflow trap.
- **PAX_MEMORY_SANITIZE** — `/proc/kcore`, `/proc/kallsyms`, `/proc/kpagecount`, `/proc/kpageflags` scratch and seq buffers zeroed on free; partial kallsyms output and per-page metadata cannot bleed.
- **PAX_UDEREF** — emitters write through seq_buf user-domain APIs; `/proc/kcore` ELF generation never deref's an unchecked target.
- **PAX_RAP/kCFI** — `proc_ops.proc_read`/`proc_read_iter`/`proc_lseek` are CFI-typed; per-file vtable identity is `__ro_after_init`.
- **GRKERNSEC_HIDESYM** — `/proc/kallsyms` zeroes all addresses for non-CAP_SYSLOG readers; `/proc/version` strips compile-host/path; `/proc/modules` zeroes addresses; `kptr_restrict` is treated as a floor.
- **GRKERNSEC_DMESG** — `/proc/kmsg` and ring buffer access require CAP_SYSLOG; cross-ref `proc-kmsg.md` for ringbuffer policy.
- **/proc/hostname PAX_USERCOPY** — read emits hostname via `seq_escape_str` with bounded length (`__NEW_UTS_LEN`); write (`/proc/sys/kernel/hostname` via sysctl path) is CAP_SYS_ADMIN gated.
- **/proc/version PAX_USERCOPY** — emitted as a single constant string from `linux_proc_banner`; no format-string consumer of user data.
- **/proc/kcore CAP_SYS_RAWIO + CAP_SYSLOG** — both required for any read; `mmap` denied; LSM `security_locked_down(LOCKDOWN_KCORE)` enforced as floor.
- **/proc/kallsyms CAP_SYSLOG** — non-CAP_SYSLOG readers see `0000000000000000` regardless of `kptr_restrict`, matching grsec hide-sym default.

Rationale: system-wide procfs is the original target of `GRKERNSEC_HIDESYM` — kallsyms + version + kcore + module list compose a single-reader full-kASLR-break primitive; the Rookery contract pins every emitter to CAP_SYSLOG (or stricter) at the upstream code path.

## Open Questions

(none — system-wide procfs entries exhaustively specified by upstream + decades of distro tooling)

## Out of Scope

- Per-task entries (cross-ref `proc-task.md`)
- /proc/sys (cross-ref `proc-sysctl.md`)
- /proc/net (cross-ref `proc-net.md`)
- 32-bit-only paths
- Implementation code
