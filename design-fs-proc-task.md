---
title: "Tier-3: fs/proc/proc-task — per-task /proc/<pid>/* infrastructure"
tags: ["design-doc", "tier-3", "fs", "procfs"]
sources: []
contributors: ["gb9t"]
created: 2026-05-09
updated: 2026-05-09
---


## Design Specification

### Summary

Tier-3 design for the per-task `/proc/<pid>/*` and `/proc/<pid>/task/<tid>/*` directory infrastructure — every utility from `ps` to `htop` to `top` to `strace -p` to `gdb attach` to systemd's per-service introspection depends on these entries' format being byte-identical. Implements:

- Per-pid directory dynamic generation (`base.c` — the file-list registration via `pid_entry[]` array)
- Per-task formatted text output (`array.c` — `proc_pid_status`, `proc_tgid_stat`, `proc_pid_stat`, `do_task_stat`)
- Per-VMA memory mapping output (`task_mmu.c` — `/proc/<pid>/maps`, `/smaps`, `/numa_maps`, `/pagemap`, `/clear_refs`)
- Per-fd directory + per-fd info (`fd.c` — `/proc/<pid>/fd/<n>` + `fdinfo/<n>`)
- Per-thread `/proc/<pid>/task/<tid>/*` mirror (`base.c`)
- `/proc/self` + `/proc/thread-self` symlinks (`self.c`, `thread_self.c`)

Sub-tier-3 of `fs/proc/00-overview.md`. The single largest compatibility surface in procfs.

### Requirements

- REQ-1: Per-pid directory + per-thread `task/<tid>/` directory generation; identical entry-list per upstream's `pid_entry[]`/`tgid_entry[]` arrays.
- REQ-2: `/proc/<pid>/stat` 52-field format byte-identical to `do_task_stat`.
- REQ-3: `/proc/<pid>/status` multi-line key:value format byte-identical (per the field list above).
- REQ-4: `/proc/<pid>/cmdline` argv-NUL-joined format byte-identical (empty for kthreads).
- REQ-5: `/proc/<pid>/comm` r/w semantics; 15-char limit + NUL.
- REQ-6: `/proc/<pid>/maps` per-VMA single-line format byte-identical.
- REQ-7: `/proc/<pid>/smaps` + `smaps_rollup` per-VMA detail-fields byte-identical.
- REQ-8: `/proc/<pid>/numa_maps` per-VMA NUMA stats byte-identical.
- REQ-9: `/proc/<pid>/pagemap` binary 8-bytes-per-page format byte-identical.
- REQ-10: `/proc/<pid>/fd/<n>` symlink scheme dispatch (socket/pipe/eventfd/etc.) byte-identical.
- REQ-11: `/proc/<pid>/fdinfo/<n>` per-fd-type detail byte-identical.
- REQ-12: `/proc/<pid>/task/<tid>/*` per-thread mirror with tgid-vs-tid scope distinction.
- REQ-13: `/proc/self` + `/proc/thread-self` symlinks resolve identically.
- REQ-14: `/proc/<pid>/clear_refs`, `coredump_filter`, `oom_score_adj` writable interfaces byte-identical.
- REQ-15: `/proc/<pid>/uid_map` / `gid_map` / `setgroups` user-ns interface byte-identical.
- REQ-16: `/proc/<pid>/auxv` + `environ` binary content byte-identical.
- REQ-17: `/proc/<pid>/wchan` resolution to kernel symbol name byte-identical.
- REQ-18: `/proc/<pid>/cgroup` + `/mountinfo` / `mounts` / `mountstats` per-task view byte-identical.
- REQ-19: `/proc/<pid>/ns/*` namespace symlinks; `open(2)` returns valid nsfd for `setns(2)`.
- REQ-20: `/proc/<pid>/attr/*` LSM-attribute interfaces routed to per-LSM hook.
- REQ-21: Hardening section per `00-security-principles.md` template.

### Acceptance Criteria

- [ ] AC-1: `cat /proc/$$/stat` byte-identical to upstream's; `ps -ef` parses correctly. (covers REQ-2)
- [ ] AC-2: `cat /proc/$$/status` byte-identical; field-list complete. (covers REQ-3)
- [ ] AC-3: `cat /proc/$$/maps` byte-identical; pmap parses. (covers REQ-6)
- [ ] AC-4: `cat /proc/$$/smaps_rollup` byte-identical; smem-style tools work. (covers REQ-7)
- [ ] AC-5: pagemap test: `head -c 8 /proc/self/pagemap` from VPN-0 returns 8-byte LE entry with bit 63 = present-flag. (covers REQ-9)
- [ ] AC-6: `ls -l /proc/$$/fd/0` symlink resolves to `/dev/pts/X` or `pipe:[<ino>]` etc. correctly. (covers REQ-10)
- [ ] AC-7: `cat /proc/$$/fdinfo/0` byte-identical: `pos:`, `flags:`, `mnt_id:`, `ino:`. (covers REQ-11)
- [ ] AC-8: per-thread test: pthread-spawn 3 threads; `ls /proc/$$/task` shows 4 directories (main + 3 threads); each task/<tid>/stat byte-identical. (covers REQ-12)
- [ ] AC-9: `readlink /proc/self` returns calling pid. (covers REQ-13)
- [ ] AC-10: clear_refs test: write `1`; subsequent `cat /proc/$$/smaps` shows soft-dirty bits cleared. (covers REQ-14)
- [ ] AC-11: user-ns test: write to `/proc/<child>/uid_map` from parent; child sees mapped UIDs. (covers REQ-15)
- [ ] AC-12: `cat /proc/$$/wchan` returns kernel symbol name (e.g., `do_select` for `select(2)` blocked). (covers REQ-17)
- [ ] AC-13: `cat /proc/$$/cgroup` byte-identical; cgroup-v2 single-line format. (covers REQ-18)
- [ ] AC-14: `setns(2)` after opening `/proc/<pid>/ns/mnt` switches caller's mount namespace. (covers REQ-19)
- [ ] AC-15: SELinux test: `cat /proc/$$/attr/current` returns SELinux label; write transitions if policy allows. (covers REQ-20)
- [ ] AC-16: Hardening section present and follows template. (covers REQ-21)

### Architecture

### Rust module organization

- `kernel::fs::proc::base::ProcPid` — per-pid directory generator
- `kernel::fs::proc::base::ProcTaskDir` — per-thread `task/<tid>/`
- `kernel::fs::proc::base::PidEntries` — `pid_entry[]` registration
- `kernel::fs::proc::array::TaskStat` — `/proc/<pid>/stat`
- `kernel::fs::proc::array::TaskStatus` — `/proc/<pid>/status`
- `kernel::fs::proc::array::TaskCmdline` — `/proc/<pid>/cmdline`
- `kernel::fs::proc::array::TaskComm` — `/proc/<pid>/comm`
- `kernel::fs::proc::task_mmu::TaskMaps` — `/proc/<pid>/maps`
- `kernel::fs::proc::task_mmu::TaskSmaps` + `SmapsRollup`
- `kernel::fs::proc::task_mmu::TaskNumaMaps`
- `kernel::fs::proc::task_mmu::TaskPagemap`
- `kernel::fs::proc::task_mmu::ClearRefs`
- `kernel::fs::proc::fd::ProcFd` — `/proc/<pid>/fd/`
- `kernel::fs::proc::fd::FdInfo` — `/proc/<pid>/fdinfo/`
- `kernel::fs::proc::self::ProcSelf` — `/proc/self`
- `kernel::fs::proc::thread_self::ProcThreadSelf` — `/proc/thread-self`
- `kernel::fs::proc::namespaces::ProcNs` — `/proc/<pid>/ns/*`

### Locking and concurrency

- **Per-task `task->lock`** (rwlock): held during stat/status snapshot read
- **Per-mm `mmap_lock`** (rwsem): held during maps/smaps walk
- **Per-task RCU**: pid-directory lookup RCU-side
- **Per-fd-table `task->files->file_lock`** (spinlock): held during fd dir read

### Error handling

- `Err(ESRCH)` — task exited mid-read
- `Err(EACCES)` — permission denied per LSM / hidepid
- `Err(EFAULT)` — userspace buffer fault
- `Err(ENOMEM)` — alloc fail
- `Err(EAGAIN)` — task being torn down

### Out of Scope

- System-wide entries (cross-ref `proc-system.md`)
- /proc/sys (cross-ref `proc-sysctl.md`)
- /proc/net (cross-ref `proc-net.md`)
- 32-bit-only paths
- Implementation code

### upstream references in scope

| Subsystem | Upstream paths |
|---|---|
| Per-pid + per-thread directory + entries | `fs/proc/base.c` |
| Per-task formatted output (stat, status, statm, schedstat) | `fs/proc/array.c` |
| Per-VMA memory mapping (maps, smaps, numa_maps, pagemap, clear_refs) | `fs/proc/task_mmu.c` |
| nommu variant | `fs/proc/task_nommu.c` |
| Per-fd subdirectory | `fs/proc/fd.c` |
| `/proc/self` symlink | `fs/proc/self.c` |
| `/proc/thread-self` symlink | `fs/proc/thread_self.c` |

### compatibility contract

### `/proc/<pid>/stat` (single-line, 52 fields)

Per `fs/proc/array.c` `do_task_stat`:
```
pid (comm) state ppid pgrp session tty_nr tpgid flags minflt cminflt majflt cmajflt utime stime cutime cstime priority nice num_threads itrealvalue starttime vsize rss rsslim startcode endcode startstack kstkesp kstkeip signal blocked sigignore sigcatch wchan nswap cnswap exit_signal processor rt_priority policy delayacct_blkio_ticks guest_time cguest_time start_data end_data start_brk arg_start arg_end env_start env_end exit_code
```

`comm` enclosed in parens; spaces inside comm preserved. All fields decimal except (legacy nswap=cnswap=0 always). Format byte-identical so `ps` / `top` / `htop` parse correctly.

### `/proc/<pid>/status` (multi-line key:value)

Per `proc_pid_status`:
```
Name:	<comm>
Umask:	0022
State:	S (sleeping)
Tgid:	<pid>
Ngid:	0
Pid:	<pid>
PPid:	<ppid>
TracerPid:	0
Uid:	<uid> <euid> <suid> <fsuid>
Gid:	<gid> <egid> <sgid> <fsgid>
FDSize:	<fd_table_size>
Groups:	<grp1> <grp2> ...
NStgid:	<ns_tgid_chain>
NSpid:	<ns_pid_chain>
NSpgid:	<ns_pgid>
NSsid:	<ns_sid>
VmPeak:	<kB>
VmSize:	<kB>
VmLck:	<kB>
VmPin:	<kB>
VmHWM:	<kB>
VmRSS:	<kB>
RssAnon:	<kB>
RssFile:	<kB>
RssShmem:	<kB>
VmData:	<kB>
VmStk:	<kB>
VmExe:	<kB>
VmLib:	<kB>
VmPTE:	<kB>
VmSwap:	<kB>
HugetlbPages:	<kB>
CoreDumping:	0
THP_enabled:	1
Threads:	<count>
SigQ:	<curr>/<max>
SigPnd:	<bitmap>
ShdPnd:	<bitmap>
SigBlk:	<bitmap>
SigIgn:	<bitmap>
SigCgt:	<bitmap>
CapInh:	<bitmap>
CapPrm:	<bitmap>
CapEff:	<bitmap>
CapBnd:	<bitmap>
CapAmb:	<bitmap>
NoNewPrivs:	0/1
Seccomp:	0/1/2
Speculation_Store_Bypass:	thread vulnerable / not vulnerable / disabled / etc.
Cpus_allowed:	<bitmap>
Cpus_allowed_list:	<list>
Mems_allowed:	<bitmap>
Mems_allowed_list:	<list>
voluntary_ctxt_switches:	<count>
nonvoluntary_ctxt_switches:	<count>
```

Format byte-identical (every field name + format).

### `/proc/<pid>/cmdline` (NUL-delimited argv)

Per `get_task_cmdline`: argv joined with NULs (no terminator). For kthreads, output is empty. Format byte-identical so `ps -ef` works.

### `/proc/<pid>/comm` (16-byte task name + newline)

Read returns task's `comm[]` field; write modifies (constrained to length 15 + NUL). Used by setproctitle-style tooling.

### `/proc/<pid>/maps` (per-VMA single-line)

Per `task_mmu.c` `show_map`:
```
<start>-<end> <perms> <offset> <dev_major>:<dev_minor> <inode> <pathname>
```

Where `<perms>` is `rwxp`/`rwxs` 4-char string (perms with private/shared flag). `<pathname>` is empty for anonymous, `[heap]`/`[stack]`/`[vdso]`/`[vsyscall]`/`[vvar]`/`[uprobes]` for special, or actual file path. Format byte-identical so `pmap` works.

### `/proc/<pid>/smaps` (per-VMA detailed)

Per `show_smap`: per-VMA single-line header (same as `maps`) + ~25 detail-fields:
```
<header line>
Size:	<kB>
KernelPageSize:	<kB>
MMUPageSize:	<kB>
Rss:	<kB>
Pss:	<kB>
Pss_Anon:	<kB>
Pss_File:	<kB>
Pss_Shmem:	<kB>
Shared_Clean:	<kB>
Shared_Dirty:	<kB>
Private_Clean:	<kB>
Private_Dirty:	<kB>
Referenced:	<kB>
Anonymous:	<kB>
LazyFree:	<kB>
AnonHugePages:	<kB>
ShmemPmdMapped:	<kB>
FilePmdMapped:	<kB>
Shared_Hugetlb:	<kB>
Private_Hugetlb:	<kB>
Swap:	<kB>
SwapPss:	<kB>
Locked:	<kB>
THPeligible:	<count>
ProtectionKey:	<key>
VmFlags:	<list>
```

Format byte-identical.

### `/proc/<pid>/smaps_rollup` (single-aggregate)

Like smaps but single-VMA-equivalent aggregate over all VMAs. Used by tools like `smem` for per-process memory summary.

### `/proc/<pid>/numa_maps` (per-VMA NUMA)

Per-VMA single-line with per-NUMA-node page count + policy info. Format byte-identical.

### `/proc/<pid>/pagemap` (binary VPN→PFN+flags)

Binary 8-bytes-per-page mapping: bit 0-54 PFN, bit 55 PTE soft-dirty, bit 56 page-exclusive-mapped, bit 60-61 zero, bit 62 swapped, bit 63 present. Used by `vmtouch` / `page-types`.

### `/proc/<pid>/fd/<n>` symlinks

Per-fd symlink resolving to `<scheme>:[<id>]` where scheme is one of:
- `socket:[<inode>]`
- `pipe:[<inode>]`
- `eventfd:[<id>]`
- `signalfd`
- `timerfd`
- `epoll`
- `inotify`
- `fanotify`
- `bpf-prog`
- `bpf-map`
- `bpf-link`
- `pidfd`
- regular `<file path>` for files

Format byte-identical.

### `/proc/<pid>/fdinfo/<n>` per-fd detail

Per-fd multi-line:
```
pos:	<offset>
flags:	<O_*>
mnt_id:	<mount id>
ino:	<inode number>
```

Plus per-fd-type extras:
- eventfd: `eventfd-count: <N>` + `eventfd-id: <id>` + flags
- signalfd: `sigmask: <bitmap>`
- epoll: per-fd `tfd: <fd> events: <events> data: <data> pos: <pos> ino: <ino> sdev: <sdev>` lines
- inotify: per-watch `inotify wd:<wd> ino:<ino> sdev:<sdev> mask:<mask> ignored_mask:<mask> fhandle-bytes:<n> fhandle-type:<n> f_handle:<hex>`
- fanotify: similar
- pidfd: `Pid: <pid>` + `NSpid: <chain>`
- BPF: `prog_type: <n>`, `prog_id: <n>`, `prog_tag: <hex>`, etc.

Format byte-identical for each fd-type.

### `/proc/<pid>/task/<tid>/*` per-thread mirror

Per-thread directory mirroring most pid-level entries (stat, status, comm, maps, etc. — but not duplicating unique-per-process entries like `cmdline`). Format byte-identical.

### `/proc/self` + `/proc/thread-self` symlinks

Resolve to caller's pid + tid respectively. Per-task-special semantics (stat() returns lstat'd state, but readlink() returns calling task's pid).

### `/proc/<pid>/clear_refs` (write-only)

Write `1` clears soft-dirty bits on all VMAs (anon+file mapped); `2` for anon-only; `3` for file-only; `4` for clear soft-dirty + reset reference. Used by working-set probing tools.

### `/proc/<pid>/coredump_filter` (read/write)

Bitmask controlling which VMA types are included in core dumps. Per-bit: anon-private / anon-shared / file-private / file-shared / elf-headers / hugetlb-shared / hugetlb-private / dax-private / dax-shared. Default `0x33`.

### `/proc/<pid>/oom_score`, `/oom_score_adj`

oom_score: read-only computed score (~0..1000); oom_score_adj: ±1000 adjustment writable by privileged callers. Identical UAPI.

### `/proc/<pid>/uid_map`, `/gid_map`, `/setgroups`

Per-user-namespace UID/GID mappings. setgroups: write `allow`/`deny` (default `allow` if user-ns has CAP_SETGID, else `deny`). Cross-ref `kernel/user_namespace.md` Tier-3.

### `/proc/<pid>/auxv`, `/environ`

Binary auxv ELF info; environ as NUL-delimited env. Used by ldd / debugger.

### `/proc/<pid>/personality` (write/read 32-bit hex)

Process personality flags (per `personality(2)`). Read-only via `/proc/<pid>/personality`.

### `/proc/<pid>/loginuid`, `/sessionid`

Audit subsystem session tracking. Loginuid set once-per-login.

### `/proc/<pid>/attr/{current,exec,fscreate,keycreate,sockcreate,prev}`

LSM (SELinux/Smack/AppArmor) attribute interfaces. Read returns label; write changes label (subject to LSM policy).

### `/proc/<pid>/wchan` (kernel symbol where blocked)

Returns kernel-symbol-name where task is currently blocked (e.g., `do_select`, `pipe_read`). Used by ps/top.

### `/proc/<pid>/sched` (read-only scheduler stats)

Per-task scheduler-class-specific stats. Format follows scheduler internals (cross-ref `kernel/sched/cfs.md`).

### `/proc/<pid>/cgroup`

Per-controller cgroup membership: `<id>:<controller>:<cgroup-path>`.

### `/proc/<pid>/mountinfo`, `/mounts`, `/mountstats`

Per-task view of mounted filesystems (per-mount-namespace).

### `/proc/<pid>/ns/{mnt,uts,ipc,pid,net,user,cgroup,time,all}`

Per-namespace symlinks; opening returns nsfd. Used by `setns(2)` + `nsenter`.

### verification

### Layer 1: Kani SAFETY proofs

| Surface | Harness |
|---|---|
| Per-pid directory generation (refcount-acquire on task; release on dir-close) | `kani::proofs::fs::proc::task::pid_dir_safety` |
| `do_task_stat` formatting (sprintf-style bound-checked) | `kani::proofs::fs::proc::task::stat_safety` |
| Per-VMA walk under mmap_lock (no use-after-free during snapshot) | `kani::proofs::fs::proc::task::vma_walk_safety` |
| Per-fd directory read under file_lock | `kani::proofs::fs::proc::task::fd_dir_safety` |

### Layer 2: TLA+ models

(none new owned at this granularity; per-task snapshot consistency declared at Tier-2)

### Layer 3: Kani harnesses for data-structure invariants

| Data structure | Invariant | Harness |
|---|---|---|
| Per-pid directory entries | every entry has unique `name`; entries match `pid_entry[]` array contents | `kani::proofs::fs::proc::task::dir_invariants` |
| Per-fd directory | every fd in directory matches `task->files->fdt` bitmap | `kani::proofs::fs::proc::task::fd_invariants` |

### Layer 4: Functional correctness (declared in `fs/proc/00-overview.md` Layer 4)

- **Per-task /proc/<pid>/stat snapshot consistency theorem** — proves: per-task stat output is a consistent snapshot (no half-updated counter values), even under concurrent task-state mutation. Owned here.

### hardening

(Cites `00-security-principles.md` § Locked default-policy table.)

### Row-1 features owned by this component

| Feature | Responsibility | Default inherited from |
|---|---|---|
| **REFCOUNT** | per-task ref + per-mm ref + per-fd ref refcounts use `Refcount` (saturating) | § Mandatory |
| **MEMORY_SANITIZE** | freed seq_file buffers cleared (carry sensitive task state — environ, auxv, secctx) | § Default-on configurable off |
| **HIDEPID** | per-mount default `hidepid=2` for non-root pids | § Default-on configurable off |

### Row-1 features consumed by this component

- **REFCOUNT, AUTOSLAB, MEMORY_SANITIZE**: see above
- **CONSTIFY**: `pid_entry[]` array `static const`
- **USERCOPY**: per-file read uses `simple_read_from_buffer` / `seq_*` bound-checked accessors
- **SIZE_OVERFLOW**: per-stat field arithmetic uses checked operators
- **KERNEXEC**: per-file dispatch via `static const fn-ptr` array

### Row-2 / GR-RBAC integration

- LSM hook `security_inode_permission` — per-file access gate.
- LSM hook `security_task_to_inode` — per-task /proc inode labeling.
- Default useful GR-RBAC policy: deny `/proc/<pid>/mem` writes outside gradm-marked `debugger` role; deny `/proc/<pid>/personality` writes outside `prctl_admin`.

### Userspace-visible behavior changes

None beyond upstream defaults.

### Verification

(See § Verification above.)

