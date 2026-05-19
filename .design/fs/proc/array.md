# Tier-3: fs/proc/array.c — /proc/<pid>/{status,stat,statm} formatters

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/proc/00-overview.md
upstream-paths:
  - fs/proc/array.c (~816 lines)
  - fs/proc/task_mmu.c (task_mem, task_statm definitions)
  - fs/proc/task_nommu.c (nommu task_mem, task_statm)
  - fs/proc/internal.h
  - include/linux/sched.h (task_struct)
  - include/linux/sched/signal.h (signal_struct, sighand_struct)
  - include/linux/cred.h (cred)
  - Documentation/filesystems/proc.rst (per-field /proc/<pid>/stat schema)
-->

## Summary

`fs/proc/array.c` formats the three byte-stable per-task ASCII reports that every distro tool depends on: **status**, **stat**, **statm**. Per-`/proc/<pid>/status` is the human-readable, multi-line `Key:\tvalue\n` form (`Name`, `Umask`, `State`, `Tgid`, `Ngid`, `Pid`, `PPid`, `TracerPid`, `Uid`, `Gid`, `FDSize`, `Groups`, `NStgid` / `NSpid` / `NSpgid` / `NSsid`, `Kthread`, `VmPeak`...`VmSwap`, `CoreDumping`, `THP_enabled`, `untag_mask`, `Threads`, `SigQ`, `SigPnd` / `ShdPnd` / `SigBlk` / `SigIgn` / `SigCgt`, `CapInh` / `CapPrm` / `CapEff` / `CapBnd` / `CapAmb`, `NoNewPrivs`, `Seccomp`, `Seccomp_filters`, `Speculation_Store_Bypass`, `SpeculationIndirectBranch`, `Cpus_allowed[_list]`, `Mems_allowed[_list]`, `voluntary_ctxt_switches`, `nonvoluntary_ctxt_switches`). Per-`/proc/<pid>/stat` is the single-line, fixed-positional 52-field record consumed by `procps`, `top`, `htop`, `ps`, `glibc-getpriority`, language runtimes — its on-disk order is a frozen ABI. Per-`/proc/<pid>/statm` is the seven-field `size resident shared text lib data dt` line (with `lib` and `dt` historically zero). Critical for: procps compatibility, container runtime introspection, performance tools, OOM observability.

This Tier-3 covers `fs/proc/array.c` (~816 lines). Per-`task_mem` / `task_statm` accounting (Vm* lines, statm derivation) lives in `task_mmu.c` and is referenced here only as a sub-call.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `task_state_array[]` | per-state ASCII names ("R (running)" ... "I (idle)") | `procfs::array::TASK_STATE_ARRAY` |
| `get_task_state(tsk)` | per-task index → state-string | `procfs::array::task_state_str` |
| `proc_task_name(m, p, escape)` | per-task `comm` writer (wq / kthread / user) | `procfs::array::write_task_name` |
| `render_sigset_t(m, hdr, set)` | per-sigset hex-nibble formatter | `procfs::array::render_sigset` |
| `collect_sigign_sigcatch(p, ign, cat)` | per-sa_handler partition | `procfs::array::collect_sigign_sigcatch` |
| `render_cap_t(m, hdr, cap)` | per-kernel_cap_t hex formatter | `procfs::array::render_cap` |
| `task_state(m, ns, pid, p)` | per-status header (Umask...Kthread) | `procfs::array::write_task_state` |
| `task_mem(m, mm)` | per-status Vm* block | `procfs::task_mmu::write_task_mem` |
| `task_sig(m, p)` | per-status Threads / SigQ / SigPnd / ShdPnd / SigBlk / SigIgn / SigCgt | `procfs::array::write_task_sig` |
| `task_cap(m, p)` | per-status CapInh / CapPrm / CapEff / CapBnd / CapAmb | `procfs::array::write_task_cap` |
| `task_seccomp(m, p)` | per-status NoNewPrivs / Seccomp[_filters] / Speculation_* | `procfs::array::write_task_seccomp` |
| `task_cpus_allowed(m, p)` | per-status Cpus_allowed[_list] | `procfs::array::write_task_cpus_allowed` |
| `task_core_dumping(m, p)` | per-status CoreDumping | `procfs::array::write_task_core_dumping` |
| `task_thp_status(m, mm)` | per-status THP_enabled | `procfs::array::write_task_thp_status` |
| `task_untag_mask(m, mm)` | per-status untag_mask | `procfs::array::write_task_untag_mask` |
| `task_context_switch_counts(m, p)` | per-status voluntary / nonvoluntary | `procfs::array::write_task_ctxsw_counts` |
| `arch_proc_pid_thread_features(m, p)` | per-arch hook (weak) | per-arch crate |
| `proc_pid_status(m, ns, pid, task)` | per-status entrypoint | `procfs::array::pid_status` |
| `do_task_stat(m, ns, pid, task, whole)` | per-stat core (52 fields) | `procfs::array::do_task_stat` |
| `proc_tid_stat(m, ns, pid, task)` | per-thread stat (whole=0) | `procfs::array::tid_stat` |
| `proc_tgid_stat(m, ns, pid, task)` | per-process stat (whole=1) | `procfs::array::tgid_stat` |
| `proc_pid_statm(m, ns, pid, task)` | per-statm formatter | `procfs::array::pid_statm` |
| `task_statm(mm, ...)` | per-mm statm-counter | `procfs::task_mmu::task_statm` |
| `get_children_pid(...)` | per-CONFIG_PROC_CHILDREN sibling walk | `procfs::array::children_get_pid` |
| `children_seq_{start,next,stop,show}` | per-CONFIG_PROC_CHILDREN seq_ops | `procfs::array::children_seq_ops` |
| `proc_tid_children_operations` | per-CONFIG_PROC_CHILDREN file_ops | `procfs::array::CHILDREN_FOPS` |

## Compatibility contract

REQ-1: Per-task_state_array — exactly the nine entries in this order; index = `task_state_index(tsk)`:
- 0x00 `R (running)`.
- 0x01 `S (sleeping)`.
- 0x02 `D (disk sleep)`.
- 0x04 `T (stopped)`.
- 0x08 `t (tracing stop)`.
- 0x10 `X (dead)`.
- 0x20 `Z (zombie)`.
- 0x40 `P (parked)`.
- 0x80 `I (idle)` — beyond TASK_REPORT, idle kthreads.
- BUILD_BUG_ON: `1 + ilog2(TASK_REPORT_MAX) == ARRAY_SIZE(task_state_array)`.

REQ-2: proc_task_name(m, p, escape):
- If `p->flags & PF_WQ_WORKER`: `wq_worker_comm(tcomm, 64, p)` — workqueue-decorated name.
- Else if `p->flags & PF_KTHREAD`: `get_kthread_comm(tcomm, 64, p)`.
- Else: `get_task_comm(tcomm, p)`.
- If escape: `seq_escape_str(m, tcomm, ESCAPE_SPACE | ESCAPE_SPECIAL, "\n\\")`.
- Else: `seq_printf(m, "%.64s", tcomm)`.
- /* Escape mode is used by /status (Name:), unescaped by /stat (...) — parentheses are sentinel. */

REQ-3: render_sigset_t(m, header, set):
- Emit header (e.g. `"\nSigPnd:\t"`).
- For `i = _NSIG; i >= 4; i -= 4`:
  - Build 4-bit nibble `x = sigismember(i+1)|...|sigismember(i+4)<<3`.
  - Emit `hex_asc[x]`.
- Newline.
- /* Result: most-significant 8 chars + ... MSB-first hex; exactly _NSIG/4 nibbles (16 for 64 signals). */

REQ-4: collect_sigign_sigcatch(p, sigign, sigcatch):
- For `i = 1; i <= _NSIG; i++`:
  - k = `&p->sighand->action[i-1]`.
  - If `k->sa.sa_handler == SIG_IGN`: `sigaddset(sigign, i)`.
  - Else if `k->sa.sa_handler != SIG_DFL`: `sigaddset(sigcatch, i)`.

REQ-5: render_cap_t(m, header, cap):
- Emit header.
- `seq_put_hex_ll(m, NULL, cap->val, 16)` — exactly 16 hex chars, big-endian.
- Newline.

REQ-6: task_state(m, ns, pid, p):
- /* RCU + ptracer + cred + fs/files snapshot. */
- rcu_read_lock.
- tracer = ptrace_parent(p); if tracer: tpid = task_pid_nr_ns(tracer, ns).
- ppid = task_ppid_nr_ns(p, ns); tgid = task_tgid_nr_ns(p, ns); ngid = task_numa_group_id(p).
- cred = get_task_cred(p).
- task_lock(p).
  - if p->fs: umask = p->fs->umask.
  - if p->files: max_fds = files_fdtable(p->files)->max_fds.
- task_unlock(p); rcu_read_unlock.
- Emit if umask>=0: `Umask:\t%#04o\n`.
- Emit `State:\t<get_task_state(p)>`.
- Emit `\nTgid:\t<tgid>\nNgid:\t<ngid>\nPid:\t<pid_nr_ns>\nPPid:\t<ppid>\nTracerPid:\t<tpid>`.
- Emit `\nUid:\t<uid>\t<euid>\t<suid>\t<fsuid>` — all `from_kuid_munged(user_ns, ...)`.
- Emit `\nGid:\t<gid>\t<egid>\t<sgid>\t<fsgid>`.
- Emit `\nFDSize:\t<max_fds>`.
- Emit `\nGroups:\t<gid0> <gid1> ...` over `cred->group_info->gid[0..ngroups]`.
- put_cred(cred).
- Emit trailing ' ' (historical wart — must keep).
- If CONFIG_PID_NS:
  - `\nNStgid:` then for `g=ns->level..pid->level`: `\t<task_tgid_nr_ns(p, pid->numbers[g].ns)>`.
  - `\nNSpid:` per-level `task_pid_nr_ns`.
  - `\nNSpgid:` per-level `task_pgrp_nr_ns`.
  - `\nNSsid:` per-level `task_session_nr_ns`.
- Newline.
- Emit `Kthread:\t<1|0>\n` from `p->flags & PF_KTHREAD`.

REQ-7: task_sig(m, p):
- Initialize empty pending, shpending, blocked, ignored, caught sigsets; num_threads=0, qsize=0, qlim=0.
- if `lock_task_sighand(p, &flags)`:
  - pending = p->pending.signal.
  - shpending = p->signal->shared_pending.signal.
  - blocked = p->blocked.
  - collect_sigign_sigcatch(p, &ignored, &caught).
  - num_threads = get_nr_threads(p).
  - rcu_read_lock; qsize = get_rlimit_value(task_ucounts(p), UCOUNT_RLIMIT_SIGPENDING); rcu_read_unlock.
  - qlim = task_rlimit(p, RLIMIT_SIGPENDING).
  - unlock_task_sighand(p, &flags).
- Emit `Threads:\t<num_threads>\nSigQ:\t<qsize>/<qlim>`.
- render_sigset_t(`\nSigPnd:\t`, pending) → render_sigset_t(`ShdPnd:\t`, shpending) → SigBlk → SigIgn → SigCgt.

REQ-8: task_cap(m, p):
- rcu_read_lock; cred = __task_cred(p); snapshot the five cap_t; rcu_read_unlock.
- render_cap_t `CapInh:`, `CapPrm:`, `CapEff:`, `CapBnd:`, `CapAmb:`.

REQ-9: task_seccomp(m, p):
- Emit `NoNewPrivs:\t<task_no_new_privs(p)>`.
- if CONFIG_SECCOMP:
  - `\nSeccomp:\t<p->seccomp.mode>`.
  - if CONFIG_SECCOMP_FILTER: `\nSeccomp_filters:\t<atomic_read(&p->seccomp.filter_count)>`.
- Emit `\nSpeculation_Store_Bypass:\t<token>` via `arch_prctl_spec_ctrl_get(p, PR_SPEC_STORE_BYPASS)` switched to: `unknown` / `not vulnerable` / `thread force mitigated` / `thread mitigated` / `thread vulnerable` / `globally mitigated` / `vulnerable`.
- Emit `\nSpeculationIndirectBranch:\t<token>` via `PR_SPEC_INDIRECT_BRANCH` switched to: `unsupported` / `not affected` / `conditional force disabled` / `conditional disabled` / `conditional enabled` / `always enabled` / `always disabled` / `unknown`.
- Trailing newline.

REQ-10: task_cpus_allowed(m, task):
- Emit `Cpus_allowed:\t%*pb\n` then `Cpus_allowed_list:\t%*pbl\n` over `&task->cpus_mask`.
- /* `cpuset_task_status_allowed(m, task)` immediately follows for Mems_allowed[_list]. */

REQ-11: task_core_dumping(m, task):
- Emit `CoreDumping:\t<!!task->signal->core_state>\n`.

REQ-12: task_thp_status(m, mm):
- `thp_enabled = IS_ENABLED(CONFIG_TRANSPARENT_HUGEPAGE)`.
- If true: `thp_enabled &= !mm_flags_test(MMF_DISABLE_THP_COMPLETELY, mm)`.
- Emit `THP_enabled:\t<0|1>\n`.

REQ-13: task_untag_mask(m, mm):
- Emit `untag_mask:\t%#lx\n` over `mm_untag_mask(mm)`.

REQ-14: task_context_switch_counts(m, p):
- Emit `voluntary_ctxt_switches:\t<p->nvcsw>\nnonvoluntary_ctxt_switches:\t<p->nivcsw>\n`.

REQ-15: arch_proc_pid_thread_features(m, task):
- `__weak` no-op default; per-arch override appends arch-private lines (x86: appends nothing; arm64: MTE/SVE state; etc.).

REQ-16: proc_pid_status(m, ns, pid, task) — entrypoint order is the wire-format order:
1. `Name:\t<proc_task_name(escape=true)>\n`.
2. task_state(...).
3. if `mm = get_task_mm(task)`:
   - task_mem(m, mm) — emits `VmPeak`, `VmSize`, `VmLck`, `VmPin`, `VmHWM`, `VmRSS`, `RssAnon`, `RssFile`, `RssShmem`, `VmData`, `VmStk`, `VmExe`, `VmLib`, `VmPTE`, `VmSwap`, `HugetlbPages` — defined in task_mmu.c (nommu: task_nommu.c with reduced set).
   - task_core_dumping(...).
   - task_thp_status(m, mm).
   - task_untag_mask(m, mm).
   - mmput(mm).
4. task_sig(...).
5. task_cap(...).
6. task_seccomp(...).
7. task_cpus_allowed(...).
8. cpuset_task_status_allowed(m, task) — `Mems_allowed` and `Mems_allowed_list`.
9. task_context_switch_counts(...).
10. arch_proc_pid_thread_features(...).
- Returns 0.

REQ-17: do_task_stat(m, ns, pid, task, whole) — emits exactly these space-separated fields, in this order, terminated by `\n`:
1. `pid` — `pid_nr_ns(pid, ns)`.
2. ` (<comm>) ` — `proc_task_name(m, task, escape=false)` wrapped in literal `(`...`)`.
3. `<state>` — single char from `*get_task_state(task)`.
4. `ppid` — `task_ppid_nr_ns` under sighand lock.
5. `pgid` — `task_pgrp_nr_ns`.
6. `sid` — `task_session_nr_ns`.
7. `tty_nr` — `new_encode_dev(tty_devnum(sig->tty))` or 0.
8. `tty_pgrp` — `pid_nr_ns(tty_get_pgrp(sig->tty), ns)` or -1.
9. `flags` — `task->flags`.
10. `min_flt` — per-thread or aggregated.
11. `cmin_flt` — `sig->cmin_flt`.
12. `maj_flt` — per-thread or aggregated.
13. `cmaj_flt` — `sig->cmaj_flt`.
14. `utime` — `nsec_to_clock_t(utime)` after `task_cputime_adjusted` or `thread_group_cputime_adjusted`.
15. `stime` — `nsec_to_clock_t(stime)`.
16. `cutime` — `nsec_to_clock_t(sig->cutime)`.
17. `cstime` — `nsec_to_clock_t(sig->cstime)`.
18. `priority` — `task_prio(task)`.
19. `nice` — `task_nice(task)`.
20. `num_threads` — `get_nr_threads(task)`.
21. itrealvalue=0 (zeroed since 2.6 — emit literal `0`).
22. `start_time` — `nsec_to_clock_t(timens_add_boottime_ns(task->start_boottime))`.
23. `vsize` — `task_vsize(mm)` or 0.
24. `rss` — `get_mm_rss(mm)` or 0.
25. `rsslim` — `READ_ONCE(sig->rlim[RLIMIT_RSS].rlim_cur)`.
26. `start_code` — `mm->start_code` (or 1 if not permitted, 0 if no mm).
27. `end_code` — `mm->end_code` (or 1 / 0).
28. `start_stack` — `mm->start_stack` (or 0).
29. `esp` — 0 unless permitted ∧ (PF_EXITING|PF_DUMPCORE|PF_POSTCOREDUMP) ∧ `try_get_task_stack`: `KSTK_ESP(task)`.
30. `eip` — 0 unless same conditions: `KSTK_EIP(task)`.
31. `signal` — `task->pending.signal.sig[0] & 0x7fffffffUL` (obsolete-but-decimal; user must consult /status for RT signals).
32. `blocked` — `task->blocked.sig[0] & 0x7fffffffUL`.
33. `sigignore` — `sigign.sig[0] & 0x7fffffffUL` from collect_sigign_sigcatch.
34. `sigcatch` — `sigcatch.sig[0] & 0x7fffffffUL`.
35. `wchan` — 0/1 flag: `!task_is_running(task)` when permitted ∧ (!whole ∨ num_threads<2); otherwise 0. Address never leaked here — actual symbol-name via /proc/<pid>/wchan.
36. nswap=0 (literal).
37. cnswap=0 (literal).
38. `exit_signal` — `task->exit_signal`.
39. `processor` — `task_cpu(task)`.
40. `rt_priority` — `task->rt_priority`.
41. `policy` — `task->policy`.
42. `delayacct_blkio_ticks` — `delayacct_blkio_ticks(task)`.
43. `guest_time` — `nsec_to_clock_t(gtime)`.
44. `cguest_time` — `nsec_to_clock_t(sig->cgtime)`.
45. `start_data` — `mm->start_data` (only if mm ∧ permitted; else literal block of seven `0`).
46. `end_data`.
47. `start_brk`.
48. `arg_start`.
49. `arg_end`.
50. `env_start`.
51. `env_end`.
52. `exit_code` — `task->exit_code` (or `sig->group_exit_code` when whole ∧ SIGNAL_GROUP_EXIT|SIGNAL_STOP_STOPPED), literal `0` if not permitted.

REQ-18: do_task_stat permission gating:
- `permitted = ptrace_may_access(task, PTRACE_MODE_READ_FSCREDS | PTRACE_MODE_NOAUDIT)`.
- esp/eip leak only under permitted ∧ exit-state ∧ `try_get_task_stack(task)` (paired with `put_task_stack`).
- mm address fields (start_code, end_code, start_stack, start_data...env_end, exit_code) are zeroed for non-permitted callers — defense against ASLR-leak via /stat.

REQ-19: do_task_stat aggregation locking (whole=1, tgid_stat):
- Under `scoped_guard(rcu)` + `scoped_seqlock_read(&sig->stats_lock, ss_lock_irqsave)`:
  - Read cmin_flt, cmaj_flt, cutime, cstime, cgtime.
  - If whole: min_flt = sig->min_flt; maj_flt = sig->maj_flt; gtime = sig->gtime.
  - `__for_each_thread(sig, t)`: min_flt += t->min_flt; maj_flt += t->maj_flt; gtime += task_gtime(t).
- After seqlock:
  - if whole: `thread_group_cputime_adjusted(task, &utime, &stime)`.
  - else: `task_cputime_adjusted(task, &utime, &stime)`; min_flt = task->min_flt; maj_flt = task->maj_flt; gtime = task_gtime(task).

REQ-20: proc_tid_stat = do_task_stat(whole=0); proc_tgid_stat = do_task_stat(whole=1).

REQ-21: proc_pid_statm(m, ns, pid, task):
- mm = get_task_mm(task).
- If mm:
  - `size = task_statm(mm, &shared, &text, &data, &resident)`.
  - mmput(mm).
  - Emit `<size> <resident> <shared> <text> 0 <data> 0\n` — fields 5 (lib) and 7 (dt) are historically zero.
- Else: `seq_write(m, "0 0 0 0 0 0 0\n", 14)`.

REQ-22: CONFIG_PROC_CHILDREN — /proc/<tid>/task/<tid>/children:
- get_children_pid(inode, pid_prev, pos): under tasklist_lock read:
  - start = pid_task(proc_pid(inode), PIDTYPE_PID).
  - Fast path: if pid_prev's task->real_parent == start ∧ has sibling: pick next sibling.
  - Slow path: list_for_each_entry(task, &start->children, sibling) until `pos-- == 0`.
  - Returns get_pid(task_pid(task)) or NULL.
- seq_ops: start = `get_children_pid(NULL, *pos)`; next = `get_children_pid(v, *pos+1)` then `put_pid(v)`; stop = `put_pid(v)`; show = `seq_printf(seq, "%d ", pid_nr_ns(v, proc_pid_ns(sb)))`.
- `proc_tid_children_operations` = `{open=children_seq_open, read=seq_read, llseek=seq_lseek, release=seq_release}`.

REQ-23: All decimal emission goes through `seq_put_decimal_ull` / `seq_put_decimal_ll` — no `seq_printf("%llu", ...)` in hot path — defense against per-locale and per-format-leak.

REQ-24: All sigsets / capability sets must emit MSB-first hex with exactly `_NSIG/4` (16) and 16-char widths respectively — byte-for-byte procps compatibility.

## Acceptance Criteria

- [ ] AC-1: `/proc/<pid>/status` first line is `Name:\t<comm>\n` with comm escaped via `ESCAPE_SPACE|ESCAPE_SPECIAL`; second block always starts `State:\t` followed by one of the nine documented `X (label)` strings.
- [ ] AC-2: `/proc/<pid>/status` `Uid:` line contains four kuids in order (uid, euid, suid, fsuid) tab-separated; same for `Gid:`.
- [ ] AC-3: `/proc/<pid>/status` `SigPnd`, `ShdPnd`, `SigBlk`, `SigIgn`, `SigCgt` are exactly 16 hex chars, MSB-first.
- [ ] AC-4: `/proc/<pid>/status` `CapInh`, `CapPrm`, `CapEff`, `CapBnd`, `CapAmb` are exactly 16 hex chars.
- [ ] AC-5: `/proc/<pid>/status` includes `Kthread:\t1\n` iff `PF_KTHREAD` set, else `0`.
- [ ] AC-6: `/proc/<pid>/stat` has exactly 52 whitespace-separated fields and ends with `\n`.
- [ ] AC-7: `/proc/<pid>/stat` field 2 is `(comm)` with comm not escaped — parentheses are the only delimiter parsers may rely on.
- [ ] AC-8: `/proc/<pid>/stat` field 3 (state) is a single char (R/S/D/T/t/X/Z/P/I).
- [ ] AC-9: `/proc/<pid>/stat` fields 22 (start_time) and 23 (vsize) match procps' `top` time-on-screen + RES expectations.
- [ ] AC-10: `/proc/<pid>/stat` field 30 (eip) and 29 (esp) are 0 except for permitted reader of exiting / coredumping task.
- [ ] AC-11: `/proc/<pid>/stat` field 35 (wchan) is 0 for running tasks, 1 for sleeping tasks (never an address since 7.0).
- [ ] AC-12: `/proc/<pid>/stat` fields 45-51 (start_data..env_end) are 0 for non-permitted readers.
- [ ] AC-13: `/proc/<tid>/stat` (whole=0) and `/proc/<tgid>/task/<tgid>/stat` (whole=0) emit per-thread min_flt/maj_flt/gtime; `/proc/<tgid>/stat` (whole=1) emits process-summed values.
- [ ] AC-14: `/proc/<pid>/statm` is exactly `"<size> <resident> <shared> <text> 0 <data> 0\n"`; for mm-less task → exactly `"0 0 0 0 0 0 0\n"` (14 bytes).
- [ ] AC-15: PID-namespace correctness: every PID in `/status` and `/stat` is translated to the seq-reader's namespace via `*_nr_ns(p, ns)`; `NStgid` etc. enumerate from `ns->level` to `pid->level`.

## Architecture

```rust
struct TaskStateArray {
    entries: [&'static str; 9],   // R / S / D / T / t / X / Z / P / I
}

const TASK_STATE_ARRAY: TaskStateArray = TaskStateArray {
    entries: [
        "R (running)", "S (sleeping)", "D (disk sleep)", "T (stopped)",
        "t (tracing stop)", "X (dead)", "Z (zombie)", "P (parked)",
        "I (idle)",
    ],
};

struct StatSnapshot {              // captured atomically per do_task_stat invocation
    pid: i32, comm: [u8; 64],
    state: u8, ppid: i32, pgid: i32, sid: i32,
    tty_nr: i32, tty_pgrp: i32,
    flags: u32,
    min_flt: u64, cmin_flt: u64, maj_flt: u64, cmaj_flt: u64,
    utime_ns: u64, stime_ns: u64, cutime_ns: i64, cstime_ns: i64,
    priority: i32, nice: i32,
    num_threads: i32,
    start_time_ticks: u64,
    vsize: u64, rss: u64, rsslim: u64,
    start_code: u64, end_code: u64, start_stack: u64,
    esp: u64, eip: u64,
    pending_lo: u32, blocked_lo: u32, sigign_lo: u32, sigcatch_lo: u32,
    wchan: u8,
    exit_signal: i32, processor: i32,
    rt_priority: u32, policy: u32,
    blkio_ticks: u64,
    gtime_ns: u64, cgtime_ns: i64,
    mm_perm_block: Option<MmRangeBlock>,  // start_data..env_end
    exit_code: i32,
    permitted: bool,
}

struct MmRangeBlock {
    start_data: u64, end_data: u64,
    start_brk: u64,
    arg_start: u64, arg_end: u64,
    env_start: u64, env_end: u64,
}
```

`procfs::array::pid_status(m, ns, pid, task) -> i32`:
1. seq_puts "Name:\t" + write_task_name(m, task, escape=true) + "\n".
2. write_task_state(m, ns, pid, task).
3. mm = get_task_mm(task); if mm:
   - task_mmu::write_task_mem(m, mm).
   - write_task_core_dumping(m, task).
   - write_task_thp_status(m, mm).
   - write_task_untag_mask(m, mm).
   - mmput(mm).
4. write_task_sig(m, task).
5. write_task_cap(m, task).
6. write_task_seccomp(m, task).
7. write_task_cpus_allowed(m, task).
8. cpuset::task_status_allowed(m, task).
9. write_task_ctxsw_counts(m, task).
10. arch::proc_pid_thread_features(m, task).
11. return 0.

`procfs::array::do_task_stat(m, ns, pid, task, whole) -> i32`:
1. state_char = `*get_task_state(task)` (first byte of "R (running)" etc.).
2. vsize=eip=esp=0.
3. permitted = ptrace_may_access(task, READ_FSCREDS|NOAUDIT).
4. mm = get_task_mm(task); if mm:
   - vsize = task_vsize(mm).
   - if permitted ∧ task->flags & (PF_EXITING|PF_DUMPCORE|PF_POSTCOREDUMP):
     - if try_get_task_stack(task): eip = KSTK_EIP; esp = KSTK_ESP; put_task_stack.
5. sigemptyset(sigign); sigemptyset(sigcatch).
6. if lock_task_sighand(task, &flags):
   - tty: tty_pgrp = pid_nr_ns(tty_get_pgrp(sig->tty)); tty_nr = new_encode_dev(tty_devnum(sig->tty)).
   - num_threads = get_nr_threads.
   - collect_sigign_sigcatch.
   - rsslim = READ_ONCE(sig->rlim[RLIMIT_RSS].rlim_cur).
   - if whole ∧ sig->flags & (SIGNAL_GROUP_EXIT|SIGNAL_STOP_STOPPED): exit_code = sig->group_exit_code.
   - sid = task_session_nr_ns; ppid = task_ppid_nr_ns; pgid = task_pgrp_nr_ns.
   - unlock_task_sighand.
7. if permitted ∧ (!whole ∨ num_threads<2): wchan = !task_is_running(task).
8. RCU + seqlock_read(&sig->stats_lock):
   - cmin_flt, cmaj_flt, cutime, cstime, cgtime = sig->c*.
   - if whole: min_flt = sig->min_flt + Σ t->min_flt; same for maj_flt and gtime.
9. if whole: thread_group_cputime_adjusted(&utime, &stime); else task_cputime_adjusted + per-task min_flt/maj_flt/gtime.
10. priority = task_prio; nice = task_nice.
11. start_time = nsec_to_clock_t(timens_add_boottime_ns(task->start_boottime)).
12. Emit field-by-field via seq_put_decimal_ull / seq_put_decimal_ll per REQ-17.
13. if mm: mmput(mm); return 0.

`procfs::array::pid_statm(m, ns, pid, task) -> i32`:
1. mm = get_task_mm(task).
2. if mm:
   - size = task_statm(mm, &shared, &text, &data, &resident).
   - mmput(mm).
   - Emit `<size> <resident> <shared> <text> 0 <data> 0\n`.
3. Else: seq_write(b"0 0 0 0 0 0 0\n").
4. return 0.

`procfs::array::render_sigset(m, header, set)`:
1. Emit header.
2. i = _NSIG; loop:
   - i -= 4.
   - x = bit(set, i+1)|bit(set,i+2)<<1|bit(set,i+3)<<2|bit(set,i+4)<<3.
   - emit HEX_LOWER[x].
   - while i >= 4.
3. Emit '\n'.

`procfs::array::write_task_sig(m, p)`:
1. Take stack sigsets; num_threads=qsize=qlim=0.
2. if lock_task_sighand(p, &flags):
   - snapshot pending = p->pending.signal; shpending = p->signal->shared_pending.signal; blocked = p->blocked.
   - collect_sigign_sigcatch(p, &ignored, &caught).
   - num_threads = get_nr_threads(p).
   - rcu { qsize = get_rlimit_value(task_ucounts(p), UCOUNT_RLIMIT_SIGPENDING) }.
   - qlim = task_rlimit(p, RLIMIT_SIGPENDING).
   - unlock_task_sighand.
3. Emit `Threads:\t<num_threads>\nSigQ:\t<qsize>/<qlim>`.
4. render_sigset for SigPnd, ShdPnd, SigBlk, SigIgn, SigCgt.

`procfs::array::write_task_cap(m, p)`:
1. rcu_read_lock.
2. cred = __task_cred(p).
3. snapshot 5 cap_t into stack.
4. rcu_read_unlock.
5. render_cap for CapInh, CapPrm, CapEff, CapBnd, CapAmb.

`procfs::array::children_seq_ops` for CONFIG_PROC_CHILDREN:
1. start: get_children_pid(inode, NULL, *pos).
2. next: pid = get_children_pid(inode, prev, *pos+1); put_pid(prev); ++*pos.
3. stop: put_pid(v).
4. show: seq_printf("%d ", pid_nr_ns(v, proc_pid_ns(sb))).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `stat_emits_52_fields` | INVARIANT | per-do_task_stat: exactly 52 separator-delimited fields. |
| `statm_seven_fields` | INVARIANT | per-pid_statm: exactly 7 fields; mm-less branch exactly `"0 0 0 0 0 0 0\n"`. |
| `comm_is_at_most_64_bytes` | INVARIANT | per-proc_task_name: tcomm[64] never overflows; unescaped emit uses `%.64s`. |
| `state_index_in_bounds` | INVARIANT | per-get_task_state: `task_state_index(t) < 9` (ARRAY_SIZE of task_state_array). |
| `sigset_16_hex_nibbles` | INVARIANT | per-render_sigset_t: exactly `_NSIG/4` nibbles + newline. |
| `cap_16_hex_chars` | INVARIANT | per-render_cap_t: 16 hex chars + newline. |
| `permitted_gates_address_leak` | INVARIANT | per-do_task_stat: !permitted ⟹ esp=eip=0 ∧ mm-block emitted as `" 0 0 0 0 0 0 0"`. |
| `pid_namespace_translated` | INVARIANT | per-do_task_stat / pid_status: every emitted PID came from `*_nr_ns(p, ns)`. |
| `try_get_task_stack_balanced` | INVARIANT | per-eip/esp leak path: every try_get_task_stack paired with put_task_stack. |
| `cred_get_put_balanced` | INVARIANT | per-task_state: every get_task_cred paired with put_cred. |
| `mm_get_put_balanced` | INVARIANT | per-pid_status / do_task_stat / pid_statm: every get_task_mm paired with mmput. |

### Layer 2: TLA+

`fs/proc/array.tla`:
- States per /proc/<pid>/{status,stat,statm} read: open → grab task → snapshot under locks → emit → release.
- Properties:
  - `safety_lock_order` — per-task_lock + sighand lock + rcu acquired in declared order, never inverted.
  - `safety_sighand_lock_required_for_signal_state` — per-task_sig: pending/shpending/blocked read only after `lock_task_sighand` returned non-zero.
  - `safety_rcu_held_for_cred` — per-task_cap: `__task_cred(p)` deref only between rcu_read_lock/_unlock.
  - `safety_no_address_leak_to_unpermitted` — per-do_task_stat: !permitted ⟹ no kernel-virtual-address emitted.
  - `liveness_status_terminates` — per-pid_status: returns 0 in finite steps for any task.
  - `liveness_stat_terminates` — per-do_task_stat: terminates regardless of num_threads.
  - `liveness_statm_terminates` — per-pid_statm: terminates regardless of mm presence.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pid_status` post: returns 0 ∧ output contains `Name:`, `State:`, `Tgid:`, `Pid:`, `Uid:`, `Gid:`, `Threads:`, `SigPnd:`, `CapEff:`, `Cpus_allowed:`, `voluntary_ctxt_switches:` | `procfs::array::pid_status` |
| `do_task_stat` post: returns 0 ∧ output is `<pid> (<comm>) <state> ...` 52-field line | `procfs::array::do_task_stat` |
| `pid_statm` post: returns 0 ∧ 7-field statm line | `procfs::array::pid_statm` |
| `render_sigset` post: writes header + 16 hex chars + newline | `procfs::array::render_sigset` |
| `render_cap` post: writes header + 16 hex chars + newline | `procfs::array::render_cap` |
| `task_state` post: emits Umask?, State, Tgid, Ngid, Pid, PPid, TracerPid, Uid, Gid, FDSize, Groups, [NStgid/NSpid/NSpgid/NSsid], Kthread | `procfs::array::write_task_state` |
| `collect_sigign_sigcatch` post: SIG_IGN ⟹ sigign; non-SIG_DFL ∧ non-SIG_IGN ⟹ sigcatch; SIG_DFL ⟹ neither | `procfs::array::collect_sigign_sigcatch` |
| `proc_task_name` post: comm ≤ 64 bytes; escape ⟹ no raw `\n` or `\\` in output | `procfs::array::write_task_name` |
| `tgid_stat` post: whole=1; per-process aggregated min_flt/maj_flt/gtime | `procfs::array::tgid_stat` |
| `tid_stat` post: whole=0; per-thread values | `procfs::array::tid_stat` |

### Layer 4: Verus/Creusot functional

`Per-/proc/<pid>/status` → byte-stream equivalence with reference golden corpus (procps-ng test suite + util-linux `ps` regression vectors): same key names, same key order, same value formats, same trailing-space artifacts.

`Per-/proc/<pid>/stat` → byte-stream equivalence with `man 5 proc` documented 52-field schema and procps `pcp-proc-pid-stat` parser: per-field type, per-field width range, per-field semantics.

`Per-/proc/<pid>/statm` → byte-stream equivalence with `man 5 proc` (7 fields, fields 5 and 7 always 0).

## Hardening

(Inherits row-1 features from `fs/proc/00-overview.md` § Hardening.)

array.c reinforcement:

- **Per-comm bounded copy (64 bytes)** — defense against per-`task_struct.comm` overflow into user-visible output; mirrors `TASK_COMM_LEN=64`.
- **Per-comm escape on /status** — defense against per-procps line-parsing break via `\n` or `\\` in comm.
- **Per-comm unescaped on /stat with `(`...`)` sentinel** — defense against per-parser-divergence; matches frozen ABI.
- **Per-permitted gating of esp/eip/start_*/end_*/arg_*/env_*** — defense against per-unprivileged ASLR-leak via /stat.
- **Per-wchan as 0/1 flag (never address)** — defense against per-kernel-pointer leak; address-form retired pre-7.0.
- **Per-`ptrace_may_access(... | NOAUDIT)`** — defense against per-audit-flood from /proc readers.
- **Per-sighand lock for signal state** — defense against per-torn-read of pending/blocked sigsets.
- **Per-RCU + seqlock for sig->c* aggregates** — defense against per-torn-read of process-wide cputime / fault counters.
- **Per-RCU for cred snapshot** — defense against per-stale cred after `commit_creds`.
- **Per-task_lock for mm / fs / files** — defense against per-mm-reaper race.
- **Per-mmput symmetric to get_task_mm** — defense against per-mm-leak on early-return paths.
- **Per-`try_get_task_stack` / `put_task_stack` symmetry** — defense against per-kstk_eip-after-free.
- **Per-`seq_put_decimal_ull` numeric emit** — defense against per-format-string injection; no `%s` of task-controlled data.

## Grsecurity/PaX-style Reinforcement

array.c is one of the densest information-disclosure surfaces in /proc; every Tier-1 PaX/grsec feature must be honored when emitting /proc/<pid>/{stat,status,statm}:

- **PAX_USERCOPY** — `seq_buf`/`seq_printf` copy_to_user paths bounded against the seq_file slab object size; no whole-`task_struct` copy can ever leak.
- **PAX_KERNEXEC** — `proc_pid_status` and per-field emitters live in `__read_only` ops tables; no runtime patching of the status emitter chain.
- **PAX_RANDKSTACK** — kernel-stack offsets randomized so `wchan`/`kstk_eip` leaks (when permitted) do not anchor a per-build attack.
- **PAX_REFCOUNT** — `get_task_struct` / `put_task_struct`, `get_task_mm` / `mmput`, `try_get_task_stack` / `put_task_stack` are saturating-refcount audited.
- **PAX_MEMORY_SANITIZE** — `task_struct.comm[]` and signal/cred snapshots zeroed on free so stale comm bytes never bleed into a later /proc reader.
- **PAX_UDEREF** — every `proc_pid_status` field path treats the seq_file buffer as user-domain memory; no kernel-pointer dereference smuggled into the format string.
- **PAX_RAP/kCFI** — per-pid_entry `proc_show` function pointers are CFI-typed; `tgid_base_stuff` table is `__ro_after_init` to defeat hijack.
- **GRKERNSEC_HIDESYM** — `/proc/<pid>/{stat,status,statm}` strip raw kernel addresses (`wchan`, `kstk_eip`, `start_code`, `start_stack`) for non-CAP_SYSLOG callers regardless of `kptr_restrict`.
- **GRKERNSEC_DMESG** — array.c never logs task identifiers to dmesg on /proc read; ptrace-deny diagnostics route to audit via `ptrace_may_access(... | NOAUDIT)` only.
- **/proc/<pid>/syscall CAP_SYS_PTRACE** — emitter rejects readers that fail `ptrace_may_access(target, PTRACE_MODE_READ_FSCREDS)`, mirroring grsec `GRKERNSEC_PROC_USER`.
- **/proc/<pid>/{stat,status,statm} HIDESYM** — esp/eip/arg_start..env_end gated behind the same ptrace check, even when `kptr_restrict=0`, so unprivileged readers never observe live ASLR addresses.

Rationale: array.c is the canonical source the grsec patch series targeted (the original `pax_useroffset` and `GRKERNSEC_PROC_ADD` chains land here); the upstream `ptrace_may_access` gates encode a strict subset of that policy, and the contract above commits Rookery to the stricter grsec-equivalent behavior by default.

## Open Questions

- Per-arch `arch_proc_pid_thread_features` content (x86 emits nothing; arm64 emits MTE/SVE) — defer per-arch fields to per-arch crate when arm64/risc-v added.
- Per-CONFIG_NUMA `task_numa_group_id` always-zero on non-NUMA — matches upstream; revisit if NUMA-balancing is gated separately.
- Per-`Mems_allowed[_list]` is emitted by `cpuset_task_status_allowed` (kernel/cgroup/cpuset.c), not by array.c — covered in cpuset Tier-3.

## Out of Scope

- `task_mem` / `task_statm` definitions (fs/proc/task_mmu.c, fs/proc/task_nommu.c — emit VmPeak..VmSwap, derive statm fields) — covered in `task-mmu.md` Tier-3.
- `cpuset_task_status_allowed` for Mems_allowed (kernel/cgroup/cpuset.c) — covered in cpuset Tier-3.
- `/proc/<pid>/wchan` symbol formatter (fs/proc/base.c) — covered in `proc-pid.md`.
- `/proc/<pid>/cmdline`, `/environ`, `/maps`, `/smaps`, `/auxv`, `/io`, `/limits`, `/oom_score*` — covered separately in `proc-pid.md` / `task-mmu.md`.
- `/proc/<pid>/task/<tid>` enumeration logic — covered in `proc-task.md`.
- Userspace `procps` regression suite — referenced as golden corpus, but tests live outside the kernel crate.
- Implementation code.
