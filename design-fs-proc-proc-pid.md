---
title: "Tier-3: fs/proc/base.c — Per-PID procfs entries (`/proc/<pid>/`)"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Per-PID procfs is the largest namespace in proc — `/proc/<pid>/{cmdline, comm, exe, root, cwd, environ, maps, smaps, status, stat, statm, oom_adj, oom_score, oom_score_adj, pagemap, mem, mounts, mountinfo, mountstats, fdinfo, fd, attr, gid_map, uid_map, projid_map, setgroups, latency, sched, schedstat, syscall, timens_offsets, wchan, ...}`. Per-base.c builds the per-pid dentry-tree on lookup; per-task_struct → per-entry → per-callback formatter. Critical for: ps/top/htop, OOM-killer userspace policy, gdb/strace inspection, container init.

This Tier-3 covers `fs/proc/base.c` (~4009 lines).

### Acceptance Criteria

- [ ] AC-1: ps aux: per-PID /proc/<pid>/stat enumerated.
- [ ] AC-2: cat /proc/self/cmdline: per-current-task cmdline.
- [ ] AC-3: readlink /proc/self/exe: per-current binary path.
- [ ] AC-4: cat /proc/self/maps: per-current VMA list.
- [ ] AC-5: cat /proc/self/status: per-current status verbose.
- [ ] AC-6: echo "newcomm" > /proc/self/comm: per-task->comm updated.
- [ ] AC-7: cat /proc/<other>/maps: PTRACE_MODE_READ_FSCREDS check; -EACCES if denied.
- [ ] AC-8: cat /proc/<pid>/oom_score: per-oom_badness output.
- [ ] AC-9: echo 1000 > /proc/self/oom_score_adj: oom_score_adj updated.
- [ ] AC-10: ls /proc/<pid>/fd/: per-fd symlinks.
- [ ] AC-11: ls /proc/<pid>/task/: per-thread TIDs.
- [ ] AC-12: cat /proc/self/wchan: per-current wait-symbol or "0".
- [ ] AC-13: cat /proc/<pid>/stack as non-root: -EPERM.
- [ ] AC-14: cat /proc/<pid>/syscall: per-task syscall args.
- [ ] AC-15: PID-namespace /proc: pids translated to ns.

### Architecture

Per-PID dir:

```
struct ProcPid;
impl ProcPid {
  fn lookup(parent: &Inode, dentry: &mut Dentry) -> Result<()>;
  fn readdir(file: &File, ctx: &mut DirContext) -> Result<()>;
}
```

`ProcPid::lookup(parent, dentry)`:
1. /* Parse /proc/<pid> */
2. tgid = parse_pid(dentry.name).
3. task = find_task_by_vpid(tgid).
4. if !task: return -ENOENT.
5. inode = proc_pid_make_inode(parent.sb, task, S_IFDIR | 0555).
6. inode.i_op = &proc_tgid_base_inode_operations.
7. inode.i_fop = &proc_tgid_base_operations.
8. d_add(dentry, inode).

`ProcPid::pident_lookup(dir, dentry, ents, nents) -> Inode`:
1. /* Linear scan for <name> in ents */
2. for ent in ents: if ent.name == dentry.name:
   - inode = proc_pident_inode(dir.sb, task, ent).
   - return inode.
3. -ENOENT.

`proc_pid_cmdline_read(file, buf, count, pos) -> ssize_t`:
1. mm = get_task_mm(task).
2. if !mm: return 0.
3. arg_start = mm.arg_start.
4. arg_end = mm.arg_end.
5. /* Read [arg_start..arg_end) from mm */
6. n = access_remote_vm(mm, arg_start + *pos, buf, len).
7. mmput(mm).
8. return n.

`proc_pid_get_link(dentry, inode, done) -> *const char`:
1. /* For exe/cwd/root */
2. if !ptrace_may_access(task, PTRACE_MODE_READ_FSCREDS): -EACCES.
3. switch ent_type:
   - exe: path = mm.exe_file.f_path.
   - cwd: path = task.fs.pwd.
   - root: path = task.fs.root.
4. /* Build link string */
5. return d_path(&path, buf, sz).

`proc_oom_score_adj_write(file, buf, count, pos) -> ssize_t`:
1. /* Parse int */
2. v = kstrtoint(buf).
3. if v < OOM_SCORE_ADJ_MIN || v > OOM_SCORE_ADJ_MAX: -EINVAL.
4. /* Permission */
5. if v < task.signal.oom_score_adj_min ∧ !capable(CAP_SYS_RESOURCE): -EACCES.
6. task.signal.oom_score_adj = v.
7. task.signal.oom_score_adj_min = min(v, task.signal.oom_score_adj_min).
8. return count.

`proc_pid_status(seq, ns, pid, task)`:
1. /* Build verbose status */
2. seq_printf(seq, "Name:\t%s\n", task.comm).
3. seq_printf(seq, "State:\t%s\n", task_state_string(task)).
4. seq_printf(seq, "Tgid:\t%d\n", task.tgid).
5. seq_printf(seq, "Pid:\t%d\n", task.pid).
6. seq_printf(seq, "VmRSS:\t%lu kB\n", task_rss_kb(task)).
7. /* ... 50+ more fields */

`proc_tid_base_lookup(parent, dentry)` (similar but tid_base_stuff).

`proc_pident_readdir(file, ctx, ents, nents)`:
1. for i = ctx.pos - 2 .. nents:
   - dir_emit(ctx, ents[i].name, ents[i].len, dynamic_dyn_ino(...), ents[i].mode >> 12).
2. ctx.pos = 2 + nents.

`proc_pid_permission(idmap, inode, mask) -> Result<()>`:
1. task = get_proc_task(inode).
2. if !task: return -ESRCH.
3. /* Per-base perm */
4. if !ptrace_may_access(task, PTRACE_MODE_READ_FSCREDS): return -EACCES.
5. return generic_permission(idmap, inode, mask).

### Out of Scope

- fs/proc core (covered in `fs/proc/00-overview.md` Tier-3)
- fs/proc/array.c (covered if expanded)
- fs/proc/task_mmu.c maps/smaps/pagemap (covered in `proc-task.md`)
- fs/proc/proc_net.c (covered in `proc-net.md`)
- fs/proc/proc_sysctl.c (covered in `proc-sysctl.md`)
- fs/proc/namespaces.c (covered in `proc-namespaces.md`)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `proc_pid_lookup()` | per-PID dentry lookup | `ProcPid::lookup` |
| `proc_pid_readdir()` | per-/proc readdir for PIDs | `ProcPid::readdir` |
| `tgid_base_stuff[]` | per-PID entry table | `ProcPid::TGID_BASE` |
| `tid_base_stuff[]` | per-TID entry table | `ProcPid::TID_BASE` |
| `proc_pident_lookup()` | per-pident inner lookup | `ProcPid::pident_lookup` |
| `proc_pident_readdir()` | per-pident readdir | `ProcPid::pident_readdir` |
| `proc_pid_get_link()` | per-symlink read (exe/cwd/root) | `ProcPid::pid_get_link` |
| `proc_pid_readlink()` | per-readlink | `ProcPid::pid_readlink` |
| `proc_pid_cmdline_read()` | per-cmdline read | `ProcPid::cmdline_read` |
| `proc_environ_read()` | per-environ read | `ProcPid::environ_read` |
| `comm_show()` / `comm_write()` | per-comm | `ProcPid::comm_*` |
| `proc_oom_score()` / `proc_oom_score_adj_*()` | per-OOM scores | `ProcPid::oom_*` |
| `proc_loginuid_*()` | per-loginuid | `ProcPid::loginuid_*` |
| `proc_pid_attr_*()` | per-LSM attr | `ProcPid::attr_*` |
| `proc_pid_personality()` | per-personality | `ProcPid::personality` |
| `proc_pid_limits()` | per-limits | `ProcPid::limits` |
| `proc_pid_syscall()` | per-syscall info | `ProcPid::syscall` |
| `proc_pid_wchan()` | per-wait-channel | `ProcPid::wchan` |
| `proc_pid_stack()` | per-kernel-stack trace | `ProcPid::stack` |
| `proc_pid_schedstat()` | per-schedstat | `ProcPid::schedstat` |
| `proc_pid_io_accounting()` | per-IO accounting | `ProcPid::io_accounting` |
| `proc_pid_status()` | per-status | `ProcPid::status` |
| `proc_pid_stat()` | per-stat | `ProcPid::stat` |
| `proc_pid_statm()` | per-statm | `ProcPid::statm` |

### compatibility contract

REQ-1: /proc/<pid> dentry:
- Per-/proc readdir lists per-task PIDs (PID namespace-scoped).
- Per-PID dentry created on lookup (lazy).
- Per-task_struct->pid → /proc/<pid>.

REQ-2: tgid_base_stuff[] entries (per-/proc/<pid>/...):
- task: dir → /proc/<pid>/task/.
- fd: dir → /proc/<pid>/fd/.
- fdinfo: dir → /proc/<pid>/fdinfo/.
- ns: dir → /proc/<pid>/ns/ (namespace fds).
- net: dir → /proc/<pid>/net/.
- attr: dir → /proc/<pid>/attr/ (LSM).
- map_files: dir → /proc/<pid>/map_files/.
- maps: file → /proc/<pid>/maps (VMA list).
- numa_maps: file → NUMA-VMA.
- smaps: file → smap-detailed.
- smaps_rollup: file → aggregate.
- pagemap: file → per-VPN PFN-info.
- mem: file → per-task mem read/write.
- environ: file → per-task envp.
- cmdline: file → per-task argv.
- comm: file → per-task->comm.
- exe: symlink → mm.exe_file.
- cwd: symlink → fs.pwd.
- root: symlink → fs.root.
- mounts: file → per-mountns mounts.
- mountinfo: file → per-mountns mountinfo.
- mountstats: file → per-mountns mountstats.
- stat: file → per-task stat (proc_pid_stat).
- statm: file → per-task statm.
- status: file → per-task status (verbose).
- limits: file → per-task->signal->rlim.
- personality: file → per-task->personality.
- syscall: file → per-task in-syscall.
- wchan: file → per-task wait-channel symbol.
- stack: file → per-task kernel-stack-trace (CAP_SYS_ADMIN).
- schedstat: file → per-task sched stats.
- sched: file → per-task sched-class info.
- io: file → per-task io-accounting.
- oom_score: file → per-task oom_badness.
- oom_score_adj: file → per-task->signal->oom_score_adj (rw).
- gid_map: file → per-task user-ns gid_map (rw).
- uid_map: file → per-task user-ns uid_map (rw).
- projid_map: file → per-task user-ns projid_map (rw).
- setgroups: file → per-task user-ns setgroups (rw).
- timens_offsets: file → per-task timens-offsets (rw).
- loginuid: file → per-task->loginuid (rw, CAP_AUDIT_CONTROL).
- sessionid: file → per-task->sessionid.
- coredump_filter: file → per-coredump-filter mask (rw).
- patch_state: file → per-livepatch-state.

REQ-3: tid_base_stuff[] (per-/proc/<pid>/task/<tid>/...):
- Subset of tgid_base; thread-specific entries.

REQ-4: Per-permission:
- proc_pid_permission: per-task ptrace_may_access(PTRACE_MODE_READ).
- Per-mem/maps/smaps/pagemap/syscall: PTRACE_MODE_ATTACH_FSCREDS or CAP_SYS_PTRACE.
- Per-stack: CAP_SYS_ADMIN.
- Per-cmdline: PTRACE_MODE_READ_FSCREDS for non-self.

REQ-5: Per-symlink (exe/cwd/root):
- get_link returns mm.exe_file or fs.pwd or fs.root path.
- Per-readlink for legacy.

REQ-6: Per-cmdline format:
- argv \0-separated, \0\0 terminator.
- Per-prctl(PR_SET_MM_ARG_*) custom range.

REQ-7: Per-environ format:
- envp \0-separated.
- PTRACE_MODE_READ.

REQ-8: Per-comm:
- Read: per-task->comm (max 16 bytes).
- Write: per-task->comm (truncated to 15+\0).

REQ-9: Per-stat fields (52 fields):
- pid, comm, state, ppid, pgrp, session, tty_nr, tpgid, flags, minflt, cminflt, majflt, cmajflt, utime, stime, cutime, cstime, priority, nice, num_threads, itrealvalue, starttime, vsize, rss, rsslim, startcode, endcode, startstack, kstkesp, kstkeip, signal, blocked, sigignore, sigcatch, wchan, nswap, cnswap, exit_signal, processor, rt_priority, policy, delayacct_blkio_ticks, guest_time, cguest_time, start_data, end_data, start_brk, arg_start, arg_end, env_start, env_end, exit_code.

REQ-10: Per-status (verbose):
- Name, Umask, State, Tgid, Ngid, Pid, PPid, TracerPid, Uid, Gid, FDSize, Groups, NStgid, NSpid, NSpgid, NSsid, VmPeak, VmSize, VmLck, VmPin, VmHWM, VmRSS, RssAnon, RssFile, RssShmem, VmData, VmStk, VmExe, VmLib, VmPTE, VmSwap, HugetlbPages, CoreDumping, THP_enabled, Threads, SigQ, SigPnd, ShdPnd, SigBlk, SigIgn, SigCgt, CapInh, CapPrm, CapEff, CapBnd, CapAmb, NoNewPrivs, Seccomp, Speculation_Store_Bypass, SpeculationIndirectBranch, Cpus_allowed, Cpus_allowed_list, Mems_allowed, Mems_allowed_list, voluntary_ctxt_switches, nonvoluntary_ctxt_switches.

REQ-11: Per-statm:
- size, resident, shared, text, data+stack, library, dt.

REQ-12: Per-oom_score:
- oom_badness(task) (read-only).

REQ-13: Per-oom_score_adj:
- Range -1000..1000.
- Per-CAP_SYS_RESOURCE for adjusting below current.

REQ-14: Per-fd:
- /proc/<pid>/fd/N: symlink → file path.
- /proc/<pid>/fdinfo/N: file → fd flags + per-fd-specific (e.g., epoll).
- Per-EXEC unlink.

REQ-15: Per-task subdir:
- /proc/<pid>/task/<tid>/ for each thread.

REQ-16: Per-PID-namespace:
- /proc views per-mount's pid_ns; pids translated.

REQ-17: Per-/proc/<pid>/ns/:
- ipc, mnt, net, pid, pid_for_children, time, time_for_children, user, uts, cgroup.
- setns(2) target.

REQ-18: Per-/proc/<pid>/net/:
- Per-netns: routes, tcp, udp, etc. (covered in net Tier-3).

REQ-19: Per-/proc/<pid>/wchan:
- kallsyms_lookup(get_wchan(task)).
- Per-task in-kernel wait-symbol.

REQ-20: Per-/proc/<pid>/stack:
- save_stack_trace_tsk(task).
- CAP_SYS_ADMIN required.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `task_get_put_balanced` | INVARIANT | per-lookup get_task_struct paired with put_task_struct. |
| `mm_get_put_balanced` | INVARIANT | per-cmdline/maps get_task_mm paired with mmput. |
| `oom_score_adj_in_range` | INVARIANT | per-oom_score_adj in [OOM_SCORE_ADJ_MIN, OOM_SCORE_ADJ_MAX]. |
| `ptrace_may_access_checked` | INVARIANT | per-mem/maps/smaps/syscall: ptrace_may_access checked. |
| `permission_propagated` | INVARIANT | per-pident inherit per-task permission. |
| `arg_range_within_mm` | INVARIANT | per-cmdline read: [arg_start, arg_end) ⊆ mm vma. |

### Layer 2: TLA+

`fs/proc/proc-pid.tla`:
- Per-lookup → per-pident-lookup → per-read.
- Per-task-exit during read.
- Properties:
  - `safety_task_alive_during_read` — per-read-callback: task ref held.
  - `safety_ptrace_check_before_read` — per-mem: PTRACE_MODE_ATTACH checked.
  - `liveness_per_lookup_eventually_returns` — per-lookup terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ProcPid::lookup` post: dentry has inode for valid pid | `ProcPid::lookup` |
| `proc_oom_score_adj_write` post: oom_score_adj in range | `oom_score_adj_write` |
| `proc_pid_cmdline_read` post: bytes ≤ count; mm ref balanced | `cmdline_read` |
| `proc_pid_status` post: per-50-fields emitted | `proc_pid_status` |
| `proc_pid_get_link` post: per-permission check; path well-formed | `proc_pid_get_link` |

### Layer 4: Verus/Creusot functional

`Per-task_struct → /proc/<pid>/{cmdline, maps, status, stat, fd, ...} per-callback formatter` semantic equivalence: per-Documentation/filesystems/proc.rst.

### hardening

(Inherits row-1 features from `fs/proc/00-overview.md` § Hardening.)

PID-procfs reinforcement:

- **Per-ptrace_may_access for mem/maps/smaps/syscall** — defense against per-cross-task info-leak.
- **Per-CAP_SYS_ADMIN for stack/sched** — defense against unprivileged kernel-stack expose.
- **Per-CAP_SYS_RESOURCE for oom_score_adj-decrease** — defense against unprivileged OOM-kill-evade.
- **Per-task ref-count get_task_struct/put** — defense against per-task UAF.
- **Per-mm_struct mmget/mmput** — defense against per-mm UAF.
- **Per-comm 16-byte truncation** — defense against per-overflow.
- **Per-cmdline arg_start/end bounded** — defense against per-OOR mm-read.
- **Per-PID-namespace translation** — defense against per-cross-ns PID-leak.
- **Per-/proc/<pid>/exe symlink read-only** — defense against per-exe-spoof.
- **Per-fdinfo per-fd type-specific format** — defense against per-fd-info-mismatch.
- **Per-attr LSM-mediated** — defense against per-cred-bypass.
- **Per-uid_map/gid_map setgroups-gate** — defense against per-userns-priv-elevate.
- **Per-loginuid CAP_AUDIT_CONTROL once** — defense against per-loginuid-spoof.
- **Per-/proc/<pid>/coredump_filter mask-only** — defense against per-coredump-overflow.

