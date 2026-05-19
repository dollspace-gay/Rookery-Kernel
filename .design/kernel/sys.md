# Tier-3: kernel/sys.c — Miscellaneous syscalls (id/pgrp/uname/prctl/rlimit/rusage)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/sys.c (~3074 lines)
  - include/linux/syscalls.h
  - include/uapi/linux/prctl.h
  - include/uapi/linux/utsname.h
  - include/uapi/linux/resource.h
  - include/uapi/linux/sysinfo.h
-->

## Summary

`kernel/sys.c` is the catch-all home for POSIX/Linux identity, scheduling-priority, resource-limit, host/uts, prctl, and self-introspection syscalls. Per-syscall families: identity (`getuid` / `geteuid` / `getgid` / `getegid` / `getpid` / `getppid` / `gettid` / `getsid` / `setsid` / `setpgid` / `getpgid` / `getpgrp`); cred-mutation (`setregid` / `setgid` / `setreuid` / `setuid` / `setresuid` / `setresgid` / `setfsuid` / `setfsgid`); host (`sethostname` / `gethostname` / `setdomainname` / `newuname` / `uname` / `olduname`); resource (`getrlimit` / `setrlimit` / `prlimit64` / `getrusage` / `getpriority` / `setpriority` / `sysinfo` / `getcpu` / `umask` / `times`); per-task control (`prctl(2)` dispatch — ~60 PR_* options including `PR_SET_DUMPABLE`, `PR_SET_NO_NEW_PRIVS`, `PR_SET_NAME`, `PR_SET_CHILD_SUBREAPER`, `PR_SET_SECCOMP`, `PR_SET_PDEATHSIG`, `PR_SET_MM`, `PR_SET_MDWE`, `PR_SET_VMA`, `PR_SET_TIMERSLACK`, ...). Critical for: POSIX compliance, container identity translation through user-ns, security-policy enforcement, ABI stability for libc/runtimes.

This Tier-3 covers `kernel/sys.c` (~3074 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `SYSCALL_DEFINE0(getpid)` | per-task pid in active pid-ns | `Sys::getpid` |
| `SYSCALL_DEFINE0(gettid)` | per-task tid in active pid-ns | `Sys::gettid` |
| `SYSCALL_DEFINE0(getppid)` | per-task parent pid in active pid-ns | `Sys::getppid` |
| `SYSCALL_DEFINE0(getuid)` / `geteuid` | per-cred real/eff uid mapped via user-ns | `Sys::getuid` / `Sys::geteuid` |
| `SYSCALL_DEFINE0(getgid)` / `getegid` | per-cred real/eff gid mapped via user-ns | `Sys::getgid` / `Sys::getegid` |
| `SYSCALL_DEFINE2(setpgid, pid, pgid)` | per-process group reparent | `Sys::setpgid` |
| `SYSCALL_DEFINE1(getpgid, pid)` / `getpgrp` | per-process group lookup | `Sys::getpgid` / `Sys::getpgrp` |
| `SYSCALL_DEFINE1(getsid, pid)` | per-session id lookup | `Sys::getsid` |
| `SYSCALL_DEFINE0(setsid)` / `ksys_setsid` | per-task become session leader | `Sys::setsid` |
| `SYSCALL_DEFINE1(newuname, name)` | per-uts struct copyout | `Sys::newuname` |
| `SYSCALL_DEFINE1(uname, name)` / `olduname` | per-legacy uts copyout | `Sys::uname` / `Sys::olduname` |
| `SYSCALL_DEFINE2(sethostname, name, len)` | per-uts nodename mutator | `Sys::sethostname` |
| `SYSCALL_DEFINE2(gethostname, name, len)` | per-uts nodename reader | `Sys::gethostname` |
| `SYSCALL_DEFINE2(setdomainname, name, len)` | per-uts domainname mutator | `Sys::setdomainname` |
| `SYSCALL_DEFINE3(setpriority, which, who, niceval)` | per-task/pgrp/user nice setter | `Sys::setpriority` |
| `SYSCALL_DEFINE2(getpriority, which, who)` | per-task/pgrp/user nice reader | `Sys::getpriority` |
| `SYSCALL_DEFINE2(getrlimit, res, rlim)` | per-task rlimit copyout | `Sys::getrlimit` |
| `SYSCALL_DEFINE2(setrlimit, res, rlim)` | per-task rlimit mutator | `Sys::setrlimit` |
| `SYSCALL_DEFINE4(prlimit64, pid, res, new, old)` | per-pid rlimit get/set | `Sys::prlimit64` |
| `do_prlimit()` | per-set/get rlimit core | `Sys::do_prlimit` |
| `SYSCALL_DEFINE2(getrusage, who, ru)` | per-task/process rusage copyout | `Sys::getrusage` |
| `void getrusage(p, who, r)` | per-task rusage accumulator | `Sys::getrusage_inner` |
| `SYSCALL_DEFINE1(sysinfo, info)` / `do_sysinfo` | per-system mem/uptime/loads | `Sys::sysinfo` / `Sys::do_sysinfo` |
| `SYSCALL_DEFINE3(getcpu, cpup, nodep, _)` | per-task CPU/node id | `Sys::getcpu` |
| `SYSCALL_DEFINE1(umask, mask)` | per-fs creation mask | `Sys::umask` |
| `SYSCALL_DEFINE1(times, tbuf)` / `do_sys_times` | per-task tms copyout | `Sys::times` |
| `SYSCALL_DEFINE5(prctl, option, a2, a3, a4, a5)` | per-PR_* dispatch | `Sys::prctl` |
| `prctl_set_mm` / `prctl_set_mm_map` | per-PR_SET_MM | `Sys::prctl_set_mm` |
| `prctl_set_vma` | per-PR_SET_VMA_ANON_NAME | `Sys::prctl_set_vma` |
| `prctl_set_mdwe` / `prctl_get_mdwe` | per-PR_SET_MDWE / GET | `Sys::prctl_mdwe` |
| `prctl_set_thp_disable` / `prctl_get_thp_disable` | per-PR_*_THP_DISABLE | `Sys::prctl_thp` |
| `prctl_set_auxv` / `prctl_get_auxv` | per-PR_GET_AUXV | `Sys::prctl_auxv` |
| `propagate_has_child_subreaper` | per-PR_SET_CHILD_SUBREAPER walker | `Sys::propagate_subreaper` |
| `set_one_prio` / `set_one_prio_perm` | per-setpriority leaf | `Sys::set_one_prio` |
| `check_prlimit_permission` | per-prlimit64 caller-check | `Sys::check_prlimit_permission` |
| `rlim64_to_rlim` / `rlim_to_rlim64` | per-uapi rlimit convert | `Sys::rlim_convert` |
| `rlim64_is_infinity` | per-arch RLIM_INFINITY check | `Sys::rlim64_is_infinity` |
| `override_release` | per-personality UNAME26 quirk | `Sys::override_release` |
| `uts_sem` (`DECLARE_RWSEM`) | per-uts read/write lock | `Sys::uts_sem` |

## Compatibility contract

REQ-1: Identity syscall returns (`getuid` / `geteuid` / `getgid` / `getegid` / `getpid` / `getppid` / `gettid` / `getsid` / `getpgid` / `getpgrp`):
- Each is `SYSCALL_DEFINE0` or `SYSCALL_DEFINE1`.
- All read-only; never fail except for `getpgid(pid)` and `getsid(pid)` with bad pid (`-ESRCH`).
- IDs are translated **through the caller's current user/pid-namespace**: `from_kuid_munged(current_user_ns(), task.cred.uid)`, `task_tgid_vnr(current)`, `pid_vnr(...)`. Per-init-ns IDs are never leaked to a child ns.

REQ-2: `getpid()` / `gettid()` / `getppid()`:
- `getpid` = `task_tgid_vnr(current)`.
- `gettid` = `task_pid_vnr(current)` (per-thread id, distinct from tgid for non-leader threads).
- `getppid` = `task_tgid_vnr(rcu_dereference(current.real_parent))` under `rcu_read_lock`.

REQ-3: `getuid` / `geteuid` / `getgid` / `getegid`:
- `getuid` = `from_kuid_munged(current_user_ns(), current_uid())`.
- `geteuid` = `from_kuid_munged(current_user_ns(), current_euid())`.
- `getgid` / `getegid` analogously over `current_gid()` / `current_egid()`.
- `_munged` form: unmapped IDs return `overflowuid` / `overflowgid` (default `(uid_t)-2`).

REQ-4: `setpgid(pid, pgid)`:
- pid==0 ⟹ pid = task_pid_vnr(group_leader).
- pgid==0 ⟹ pgid = pid.
- pgid < 0 ⟹ `-EINVAL`.
- `write_lock_irq(&tasklist_lock)`.
- `p = find_task_by_vpid(pid)`; !p ⟹ `-ESRCH`.
- !thread_group_leader(p) ⟹ `-EINVAL`.
- if `same_thread_group(p.real_parent, current.group_leader)`:
  - task_session(p) != task_session(group_leader) ⟹ `-EPERM`.
  - !(p.flags & PF_FORKNOEXEC) ⟹ `-EACCES` (no execve'd children may be moved).
- else: p != group_leader ⟹ `-ESRCH`.
- p.signal.leader ⟹ `-EPERM` (session leaders cannot change pgrp).
- if pgid != pid: pgrp = find_vpid(pgid); pid_task(pgrp, PIDTYPE_PGID) must exist in same session.
- security_task_setpgid(p, pgid) hook.
- `change_pid(pids, p, PIDTYPE_PGID, pgrp)` if needed.

REQ-5: `getpgid(pid)` / `getpgrp()` / `getsid(pid)`:
- pid==0 ⟹ self.
- else find_task_by_vpid(pid); !p ⟹ `-ESRCH`.
- `task_pgrp(p)` / `task_session(p)`.
- security_task_getpgid(p) / security_task_getsid(p) hook.
- Return `pid_vnr(grp)` / `pid_vnr(sid)` (translated via active pid-ns).
- `getpgrp()` ≡ `getpgid(0)` (legacy alias under `__ARCH_WANT_SYS_GETPGRP`).

REQ-6: `setsid()` / `ksys_setsid()`:
- write_lock_irq(&tasklist_lock).
- group_leader.signal.leader == 1 ⟹ `-EPERM` (already a session leader).
- pid_task(task_pid(group_leader), PIDTYPE_PGID) ⟹ `-EPERM` (proposed sid collides with existing pgrp).
- group_leader.signal.leader = 1.
- `set_special_pids(pids, sid)`: change_pid(SID, sid) + change_pid(PGID, sid).
- `proc_clear_tty(group_leader)` (detach controlling tty).
- on success: `proc_sid_connector(group_leader)` + `sched_autogroup_create_attach(group_leader)`.
- Return new session id (== task_pid_vnr(group_leader)).

REQ-7: `newuname(name)` / `uname(name)` / `olduname(name)`:
- `down_read(&uts_sem)`.
- Copy current `utsname()` (per-uts-ns) → user buffer.
- `override_release()` applied to `release` field for `PER_LINUX32` personality (`UNAME26`: maps 4.x → 2.6.60+x — back-compat for ancient binaries).
- `override_architecture(name)` substitutes `machine` for PER_LINUX32 (e.g. x86_64 → i686).
- `up_read(&uts_sem)`.
- `copy_to_user` failure ⟹ `-EFAULT`.

REQ-8: `sethostname(name, len)` / `setdomainname(name, len)`:
- `!ns_capable(current.nsproxy.uts_ns.user_ns, CAP_SYS_ADMIN)` ⟹ `-EPERM`.
- len < 0 ∨ len > `__NEW_UTS_LEN` (64) ⟹ `-EINVAL`.
- copy_from_user(tmp, name, len); failure ⟹ `-EFAULT`.
- `add_device_randomness(tmp, len)` (feed entropy pool).
- `down_write(&uts_sem)`.
- u = utsname(); memcpy(u.nodename / u.domainname, tmp, len); zero-fill the remainder.
- `uts_proc_notify(UTS_PROC_HOSTNAME / _DOMAINNAME)` (notify `/proc/sys/kernel/hostname` waiters).
- `up_write(&uts_sem)`.

REQ-9: `gethostname(name, len)` (legacy alias of `uname().nodename`):
- len < 0 ⟹ `-EINVAL`.
- `down_read(&uts_sem)`; i = min(1 + strlen(u.nodename), len); memcpy(tmp, u.nodename, i); up_read.
- copy_to_user(name, tmp, i).

REQ-10: `getpriority(which, who)` / `setpriority(which, who, niceval)`:
- which ∈ { `PRIO_PROCESS`, `PRIO_PGRP`, `PRIO_USER` }; else `-EINVAL`.
- `set`: niceval clamped to [`MIN_NICE` (-20), `MAX_NICE` (19)].
- `PRIO_PROCESS`: who==0 ⟹ current; else find_task_by_vpid(who).
- `PRIO_PGRP`: who==0 ⟹ task_pgrp(current); else find_vpid(who); iterate via `do_each_pid_thread(pgrp, PIDTYPE_PGID, p)` under read_lock(&tasklist_lock).
- `PRIO_USER`: who==0 ⟹ cred.uid; else `make_kuid(cred.user_ns, who)` + find_user(uid); iterate `for_each_process_thread(g, p)` matching `task_uid(p) == uid ∧ task_pid_vnr(p) != 0` (excludes pid-ns 0).
- per-task: `set_one_prio_perm`: `uid_eq(pcred.uid, cred.euid) ∨ uid_eq(pcred.euid, cred.euid) ∨ ns_capable(pcred.user_ns, CAP_SYS_NICE)`.
- `niceval < task_nice(p) ∧ !can_nice(p, niceval)` ⟹ `-EACCES`.
- security_task_setnice / security hook checks; set_user_nice(p, niceval).
- get returns `nice_to_rlimit(task_nice(p))` (40..1 instead of -20..19, per legacy ABI).

REQ-11: `getrlimit(resource, rlim)` / `setrlimit(resource, rlim)` / `prlimit64(pid, resource, new, old)`:
- resource ≥ `RLIM_NLIMITS` (16) ⟹ `-EINVAL`. `array_index_nospec()` applied (Spectre v1 hardening).
- new_rlim.rlim_cur > new_rlim.rlim_max ⟹ `-EINVAL`.
- resource == `RLIMIT_NOFILE` ∧ new.rlim_max > `sysctl_nr_open` ⟹ `-EPERM`.
- new.rlim_max > old.rlim_max ∧ !`capable(CAP_SYS_RESOURCE)` ⟹ `-EPERM`.
- security_task_setrlimit hook.
- task_lock(tsk.group_leader); copy old; store new; task_unlock.
- RLIMIT_CPU + non-INFINITY ⟹ `update_rlimit_cpu(group_leader, rlim_cur)` (re-arms POSIX CPU timer).
- `prlimit64`: read `LSM_PRLIMIT_READ`/`_WRITE` check via `check_prlimit_permission` (id match across uid/euid/suid/gid/egid/sgid or ns_capable(CAP_SYS_RESOURCE)); needs tasklist_lock if target is in different thread-group.
- `setrlimit` ≡ `do_prlimit(current, resource, &new, NULL)`.

REQ-12: `getrusage(who, ru)` / kernel `void getrusage(p, who, r)`:
- who ∈ { `RUSAGE_SELF`, `RUSAGE_CHILDREN`, `RUSAGE_BOTH`, `RUSAGE_THREAD` }.
- `RUSAGE_THREAD`: per-thread; accumulate_thread_rusage(current, r); maxrss = sig.maxrss.
- `RUSAGE_SELF` / `_CHILDREN` / `_BOTH`: `read_seqbegin_or_lock_irqsave(&sig.stats_lock, &seq)`; sum cutime/cstime/cnvcsw/cnivcsw/cmin_flt/cmaj_flt/cinblock/coublock from sig + walk all threads via `__for_each_thread(sig, t)` accumulating live nvcsw/nivcsw/min_flt/maj_flt/io counters.
- task_cputime_adjusted(p, &tgutime, &tgstime) (per-virt-cycle scaling).
- maxrss = max(maxrss, get_mm_hiwater_rss(mm)) if mm.
- r.ru_utime / r.ru_stime ← ns_to_timeval(scaled cputime).
- r.ru_maxrss = maxrss (KB).

REQ-13: `sysinfo(info)` / `do_sysinfo(info)`:
- memset(info, 0).
- `ktime_get_boottime_ts64(&tp)` + `timens_add_boottime(&tp)` (per-time-ns offset).
- info.uptime = tp.tv_sec + (tp.tv_nsec ? 1 : 0).
- `get_avenrun(info.loads, 0, SI_LOAD_SHIFT - FSHIFT)` (load averages — fixed-point).
- info.procs = `nr_threads`.
- `si_meminfo(info)` (totalram/freeram/sharedram/bufferram/totalhigh/freehigh/mem_unit).
- `si_swapinfo(info)` (totalswap/freeswap).
- 32-bit overflow rescale: while totalram+totalswap > ULONG_MAX, mem_unit <<= 1 and rescale 8 fields; info.mem_unit = 1 (2.2.x compat).

REQ-14: `umask(mask)` / `getcpu(cpup, nodep, _)` / `times(tbuf)`:
- `umask`: returns prior `fs.umask`; sets new (mask & 0777).
- `getcpu`: cpu = smp_processor_id(); node = cpu_to_node(cpu); put_user(cpu, cpup) / put_user(node, nodep); ignores `unused` arg (per-vDSO ABI: was tcache slot).
- `times`: do_sys_times(&tms): tms_utime/stms_stime = scaled current.signal totals; tms_cutime/cstime = cutime/cstime; ret = jiffies_64_to_clock_t(get_jiffies_64()).

REQ-15: `prctl(option, arg2, arg3, arg4, arg5)` dispatch:
- security_task_prctl(option, ...) — if returns != -ENOSYS, return that error early (LSM short-circuit).
- switch (option):
  - `PR_SET_PDEATHSIG` / `PR_GET_PDEATHSIG`: valid_signal(arg2); store/read under read_lock(&tasklist_lock).
  - `PR_SET_DUMPABLE` / `PR_GET_DUMPABLE`: arg2 ∈ { `SUID_DUMP_DISABLE` (0), `SUID_DUMP_USER` (1) } only; set/get via `set_dumpable(mm, val)` / `get_dumpable(mm)`.
  - `PR_SET_NAME` / `PR_GET_NAME`: copy ≤ `TASK_COMM_LEN-1` (15) bytes; `set_task_comm` / `get_task_comm` + `proc_comm_connector`.
  - `PR_SET_SECCOMP` / `PR_GET_SECCOMP`: delegate to `prctl_set_seccomp` / `prctl_get_seccomp`.
  - `PR_GET_TIMING` = `PR_TIMING_STATISTICAL`; SET only accepts STATISTICAL.
  - `PR_SET_TSC` / `PR_GET_TSC`: arch-specific (x86 — RDTSC trap).
  - `PR_SET_UNALIGN` / `_FPEMU` / `_FPEXC` / `_ENDIAN`: arch macros (no-op on x86_64).
  - `PR_TASK_PERF_EVENTS_DISABLE` / `_ENABLE`: per-task perf gating.
  - `PR_SET_TIMERSLACK` / `_GET_TIMERSLACK`: per-task hrtimer slack ns; clamps to default if arg2==0.
  - `PR_MCE_KILL` / `_GET`: per-task hwpoison policy (EARLY_KILL vs LATE_KILL).
  - `PR_SET_MM`: subops PR_SET_MM_{START_CODE,END_CODE,START_DATA,END_DATA,START_STACK,START_BRK,BRK,ARG_START,ARG_END,ENV_START,ENV_END,AUXV,EXE_FILE,MAP,MAP_SIZE} — CAP_SYS_RESOURCE required for most; `prctl_set_mm_map` for atomic update via `struct prctl_mm_map`.
  - `PR_GET_TID_ADDRESS`: copy current.clear_child_tid to user.
  - `PR_SET_CHILD_SUBREAPER` / `_GET`: signal.is_child_subreaper bit; on SET walk_process_tree(propagate_has_child_subreaper).
  - `PR_SET_NO_NEW_PRIVS`: arg2==1, arg3/4/5==0 only; `task_set_no_new_privs(current)` — sticky, never cleared.
  - `PR_GET_NO_NEW_PRIVS`: returns `task_no_new_privs(current) ? 1 : 0`.
  - `PR_SET_THP_DISABLE` / `_GET`: per-mm MMF_DISABLE_THP flag (mmap_write_lock'd).
  - `PR_SET_FP_MODE` / `_GET`: per-task FP register mode (MIPS).
  - `PR_SVE_SET_VL` / `_GET_VL`, `PR_SME_SET_VL` / `_GET_VL`: arm64 SVE/SME vector length.
  - `PR_GET_SPECULATION_CTRL` / `PR_SET_SPECULATION_CTRL`: arch_prctl_spec_ctrl_{get,set} — STIBP / SSBD / IBPB control.
  - `PR_PAC_RESET_KEYS` / `_SET_ENABLED_KEYS` / `_GET_ENABLED_KEYS`: arm64 pointer-authentication.
  - `PR_SET_TAGGED_ADDR_CTRL` / `_GET`: arm64 TBI / MTE control.
  - `PR_SET_IO_FLUSHER` / `_GET`: CAP_SYS_RESOURCE; per-task PR_IO_FLUSHER flag (PF_LESS_THROTTLE-equivalent).
  - `PR_SET_SYSCALL_USER_DISPATCH`: `set_syscall_user_dispatch(arg2, arg3, arg4, (char __user *)arg5)` (allows userspace to intercept its own syscalls).
  - `PR_SCHED_CORE` (if CONFIG_SCHED_CORE): `sched_core_share_pid(arg2, arg3, arg4, arg5)` (core-scheduling cookie sharing).
  - `PR_SET_MDWE` / `_GET`: prctl_set_mdwe / prctl_get_mdwe — Memory-Deny-Write-Execute sticky bit.
  - `PR_PPC_GET_DEXCR` / `PR_PPC_SET_DEXCR`: POWER10 DEXCR aspect.
  - `PR_SET_VMA` (arg2 == `PR_SET_VMA_ANON_NAME`): rename anon VMA range.
  - `PR_GET_AUXV`: copy_to_user current.mm.saved_auxv (capped at arg3 bytes).
  - `PR_SET_MEMORY_MERGE` / `_GET` (if CONFIG_KSM): ksm_enable_merge_any(mm) / ksm_disable; mmap_write_lock_killable.
  - `PR_RISCV_V_SET_CONTROL` / `_GET`, `PR_RISCV_SET_ICACHE_FLUSH_CTX`: RISC-V V extension / ifence control.
  - `PR_GET_SHADOW_STACK_STATUS` / `_SET` / `_LOCK_*`: arm64/x86 CET shadow-stack.
  - `PR_TIMER_CREATE_RESTORE_IDS`: `posixtimer_create_prctl(arg2)` (CRIU-restore-time timer-id selection).
  - `PR_FUTEX_HASH`: `futex_hash_prctl(arg2, arg3, arg4)` (immediate-hash policy).
  - `PR_RSEQ_SLICE_EXTENSION`: rseq scheduling-slice extension.
  - `PR_GET_CFI` / `PR_SET_CFI` (arg2==`PR_CFI_BRANCH_LANDING_PADS`): arch_prctl_get/set/lock branch-landing-pad state (IBT / BTI).
  - `PR_MPX_ENABLE_MANAGEMENT` / `_DISABLE_MANAGEMENT`: always `-EINVAL` (MPX removed).
  - default: `trace_task_prctl_unknown` + `-EINVAL`.

REQ-16: prctl `PR_SET_DUMPABLE` semantics:
- Only `SUID_DUMP_DISABLE` (0) or `SUID_DUMP_USER` (1) writable; `SUID_DUMP_ROOT` (2) is kernel-set only (set on suid-exec).
- Affects `/proc/<pid>/{maps,mem,environ,...}` readability and core_pattern triggering.

REQ-17: prctl `PR_SET_NO_NEW_PRIVS` semantics:
- Sticky bit on `task_struct.atomic_flags & PFA_NO_NEW_PRIVS`.
- Once set, `execve` of suid/sgid binary or with file-capabilities silently drops the privilege bits.
- Inherited across fork; copied across exec.
- Required precondition for unprivileged `seccomp(SECCOMP_SET_MODE_FILTER)`.
- Cannot be cleared.

REQ-18: prctl `PR_SET_MM` requires `CAP_SYS_RESOURCE`; mutates per-mm address-range hints that influence `/proc/<pid>/stat`, coredump format, and AT_RANDOM/AT_BASE auxv slots. PR_SET_MM_MAP copies a fully-checked `struct prctl_mm_map` atomically under mmap_write_lock.

## Acceptance Criteria

- [ ] AC-1: `getuid()` / `geteuid()` in a user-ns return mapped (not host) uids; unmapped returns `overflowuid`.
- [ ] AC-2: `getpid()` in a pid-ns returns the pid-ns-local id; pid 1 in the inner ns is not pid 1 in init_pid_ns.
- [ ] AC-3: `gettid()` returns the per-thread id (== pid for single-threaded; distinct for non-leader threads).
- [ ] AC-4: `setpgid(0, 0)` makes the caller the leader of a new pgrp == its pid.
- [ ] AC-5: `setsid()` from session leader returns `-EPERM`; from non-leader makes caller the session+pgrp leader and detaches controlling tty.
- [ ] AC-6: `sethostname` without `CAP_SYS_ADMIN` over the uts-ns's user-ns returns `-EPERM`.
- [ ] AC-7: `setrlimit` raising rlim_max above its prior value without `CAP_SYS_RESOURCE` returns `-EPERM`.
- [ ] AC-8: `prlimit64` against another task with mismatched ids and no `CAP_SYS_RESOURCE` over target's user-ns returns `-EPERM`.
- [ ] AC-9: `setpriority(PRIO_PROCESS, 0, n)` with `n < 0` and no `CAP_SYS_NICE` returns `-EACCES` if it lowers below current.
- [ ] AC-10: `prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0)` succeeds and is observed via `PR_GET_NO_NEW_PRIVS`; subsequent execve of a suid binary drops the suid bit.
- [ ] AC-11: `prctl(PR_SET_DUMPABLE, 2, ...)` returns `-EINVAL` (only 0 or 1 allowed from userspace).
- [ ] AC-12: `prctl(PR_SET_NAME, p)` truncates to 15 bytes + NUL.
- [ ] AC-13: `prctl(PR_GET_CHILD_SUBREAPER, &out)` returns the value set by a prior `PR_SET_CHILD_SUBREAPER`.
- [ ] AC-14: `sysinfo()` populates `uptime`, `loads`, `procs`, and memory fields; 32-bit overflow rescale leaves `mem_unit` == 1 + 64-bit values shifted.
- [ ] AC-15: `getrusage(RUSAGE_THREAD)` returns only the current thread's counters.
- [ ] AC-16: `uname()` with `PER_LINUX32` personality rewrites `release` to "2.6.6X.y" format.

## Architecture

```
/* Per-syscall front ends live in Sys; per-uts-ns and per-user-ns
 * translation flow through Ns helpers documented in
 * kernel/pid-namespace.md and kernel/user-namespace.md. */

pub struct Sys;

impl Sys {
  pub fn getpid()  -> Pid       { task_tgid_vnr(current()) }
  pub fn gettid()  -> Pid       { task_pid_vnr(current()) }
  pub fn getppid() -> Pid       { rcu_read_lock(); let p = task_tgid_vnr(current().real_parent); rcu_read_unlock(); p }
  pub fn getuid()  -> Uid       { from_kuid_munged(current_user_ns(), current_uid()) }
  pub fn geteuid() -> Uid       { from_kuid_munged(current_user_ns(), current_euid()) }
  pub fn getgid()  -> Gid       { from_kgid_munged(current_user_ns(), current_gid()) }
  pub fn getegid() -> Gid       { from_kgid_munged(current_user_ns(), current_egid()) }
  /* ... see REQ-1..18 ... */
}
```

`Sys::setpgid(pid, pgid) -> Result<(), Errno>`:
1. pid = if pid == 0 { task_pid_vnr(current.group_leader) } else { pid }.
2. pgid = if pgid == 0 { pid } else { pgid }.
3. if pgid < 0 { return Err(EINVAL) }.
4. rcu_read_lock(); write_lock_irq(&tasklist_lock).
5. p = find_task_by_vpid(pid).ok_or(ESRCH)?
6. if !thread_group_leader(p) { return Err(EINVAL) }.
7. if same_thread_group(p.real_parent, current.group_leader):
   - if task_session(p) != task_session(current.group_leader) { return Err(EPERM) }.
   - if !(p.flags & PF_FORKNOEXEC) { return Err(EACCES) }.
8. else if p != current.group_leader { return Err(ESRCH) }.
9. if p.signal.leader { return Err(EPERM) }.
10. let pgrp = if pgid == pid { task_pid(p) } else { let pgrp = find_vpid(pgid); let g = pid_task(pgrp, PIDTYPE_PGID); require(g && task_session(g) == task_session(current.group_leader), EPERM); pgrp }.
11. security_task_setpgid(p, pgid)?
12. if task_pgrp(p) != pgrp { change_pid(&pids, p, PIDTYPE_PGID, pgrp) }.
13. write_unlock_irq + rcu_read_unlock + free_pids(&pids).

`Sys::setsid() -> Result<Pid, Errno>`:
1. let gl = current.group_leader.
2. write_lock_irq(&tasklist_lock).
3. if gl.signal.leader { return Err(EPERM) }.
4. let sid = task_pid(gl).
5. if pid_task(sid, PIDTYPE_PGID).is_some() { return Err(EPERM) }.
6. gl.signal.leader = 1.
7. set_special_pids(&pids, sid).
8. proc_clear_tty(gl).
9. write_unlock_irq(&tasklist_lock); free_pids(&pids).
10. proc_sid_connector(gl); sched_autogroup_create_attach(gl).
11. Ok(pid_vnr(sid)).

`Sys::sethostname(name: *const u8, len: i32) -> Result<(), Errno>`:
1. if !ns_capable(current.nsproxy.uts_ns.user_ns, CAP_SYS_ADMIN) { return Err(EPERM) }.
2. if len < 0 || len > __NEW_UTS_LEN { return Err(EINVAL) }.
3. let mut tmp = [0u8; __NEW_UTS_LEN].
4. copy_from_user(&mut tmp, name, len)?
5. add_device_randomness(&tmp[..len], len).
6. down_write(&uts_sem).
7. let u = utsname(); copy(&mut u.nodename[..len], &tmp[..len]); zero(&mut u.nodename[len..]).
8. uts_proc_notify(UTS_PROC_HOSTNAME).
9. up_write(&uts_sem); Ok(()).

`Sys::do_prlimit(tsk, resource, new, old) -> Result<(), Errno>`:
1. if resource >= RLIM_NLIMITS { return Err(EINVAL) }; resource = array_index_nospec(resource, RLIM_NLIMITS).
2. if let Some(n) = new { if n.rlim_cur > n.rlim_max { return Err(EINVAL) }; if resource == RLIMIT_NOFILE && n.rlim_max > sysctl_nr_open { return Err(EPERM) } }.
3. let rlim = &tsk.signal.rlim[resource]; task_lock(tsk.group_leader).
4. if let Some(n) = new { if n.rlim_max > rlim.rlim_max && !capable(CAP_SYS_RESOURCE) { task_unlock; return Err(EPERM) }; security_task_setrlimit(tsk, resource, &n)? }.
5. if let Some(o) = old { *o = *rlim }.
6. if let Some(n) = new { *rlim = n }.
7. task_unlock(tsk.group_leader).
8. if let Some(n) = new { if resource == RLIMIT_CPU && n.rlim_cur != RLIM_INFINITY { update_rlimit_cpu(tsk.group_leader, n.rlim_cur) } }.
9. Ok(()).

`Sys::prctl(option, a2, a3, a4, a5) -> Result<i64, Errno>`:
1. let r = security_task_prctl(option, a2, a3, a4, a5); if r != -ENOSYS { return r }.
2. match option {
   - PR_SET_PDEATHSIG => { require(valid_signal(a2), EINVAL); read_lock(&tasklist_lock); current.pdeath_signal = a2; read_unlock; Ok(0) },
   - PR_GET_PDEATHSIG => put_user(current.pdeath_signal, a2 as *mut i32),
   - PR_GET_DUMPABLE  => Ok(get_dumpable(current.mm) as i64),
   - PR_SET_DUMPABLE  => { require(a2 == SUID_DUMP_DISABLE || a2 == SUID_DUMP_USER, EINVAL); set_dumpable(current.mm, a2); Ok(0) },
   - PR_SET_NAME      => { let mut c = [0u8; 16]; strncpy_from_user(&mut c[..15], a2 as *const u8)?; set_task_comm(current, &c); proc_comm_connector(current); Ok(0) },
   - PR_GET_NAME      => { let c = get_task_comm(current); copy_to_user(a2 as *mut u8, &c, 16)?; Ok(0) },
   - PR_SET_NO_NEW_PRIVS => { require(a2 == 1 && a3 == 0 && a4 == 0 && a5 == 0, EINVAL); task_set_no_new_privs(current); Ok(0) },
   - PR_GET_NO_NEW_PRIVS => { require(a2 == 0 && a3 == 0 && a4 == 0 && a5 == 0, EINVAL); Ok(task_no_new_privs(current) as i64) },
   - PR_SET_CHILD_SUBREAPER => { current.signal.is_child_subreaper = (a2 != 0); walk_process_tree(current, propagate_has_child_subreaper, None); Ok(0) },
   - PR_GET_CHILD_SUBREAPER => put_user(current.signal.is_child_subreaper, a2 as *mut i32),
   - PR_SET_THP_DISABLE => prctl_set_thp_disable(a2, a3, a4, a5),
   - PR_GET_THP_DISABLE => prctl_get_thp_disable(a2, a3, a4, a5),
   - PR_SET_SECCOMP     => prctl_set_seccomp(a2, a3 as *const u8),
   - PR_GET_SECCOMP     => prctl_get_seccomp(),
   - PR_SET_MM          => prctl_set_mm(a2, a3, a4, a5),
   - PR_SET_MDWE        => prctl_set_mdwe(a2, a3, a4, a5),
   - PR_GET_MDWE        => prctl_get_mdwe(a2, a3, a4, a5),
   - PR_SET_VMA         => prctl_set_vma(a2, a3, a4, a5),
   - PR_GET_AUXV        => prctl_get_auxv(a2 as *mut u8, a3),
   - PR_SET_SPECULATION_CTRL => arch_prctl_spec_ctrl_set(current, a2, a3),
   - PR_GET_SPECULATION_CTRL => arch_prctl_spec_ctrl_get(current, a2),
   - /* ... all ~60 cases ... */
   - _ => { trace_task_prctl_unknown(option, a2, a3, a4, a5); Err(EINVAL) },
   }.

`Sys::sysinfo(info: *mut SysinfoStruct) -> Result<(), Errno>`:
1. let mut val = SysinfoStruct::zero().
2. Sys::do_sysinfo(&mut val).
3. copy_to_user(info, &val, sizeof::<SysinfoStruct>()).

`Sys::do_sysinfo(info)`:
1. info.zero().
2. let mut tp = TimeSpec64::zero(); ktime_get_boottime_ts64(&mut tp); timens_add_boottime(&mut tp).
3. info.uptime = tp.tv_sec + if tp.tv_nsec != 0 { 1 } else { 0 }.
4. get_avenrun(&mut info.loads, 0, SI_LOAD_SHIFT - FSHIFT).
5. info.procs = nr_threads().
6. si_meminfo(info); si_swapinfo(info).
7. /* 32-bit rescale loop: while overflow, mem_unit <<= 1 and info.totalram/freeram/sharedram/bufferram/totalswap/freeswap/totalhigh/freehigh <<= 1 */.
8. info.mem_unit = 1.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `id_translation_via_active_ns` | INVARIANT | per-get{uid,gid,pid,ppid}: always via current_user_ns / task_active_pid_ns; never raw kuid/kgid. |
| `setpgid_locks_tasklist_write` | INVARIANT | per-setpgid: write_lock_irq(&tasklist_lock) held during mutation. |
| `setsid_session_leader_check` | INVARIANT | per-setsid: signal.leader && pid_task(sid, PIDTYPE_PGID).is_some() ⟹ -EPERM. |
| `uts_write_under_uts_sem` | INVARIANT | per-sethostname/setdomainname: down_write(&uts_sem) held. |
| `setrlimit_max_raise_requires_capability` | INVARIANT | per-do_prlimit: rlim_max raise without CAP_SYS_RESOURCE ⟹ EPERM. |
| `rlimit_resource_bounded` | INVARIANT | per-do_prlimit: resource < RLIM_NLIMITS; array_index_nospec applied. |
| `prctl_no_new_privs_sticky` | INVARIANT | per-PR_SET_NO_NEW_PRIVS: once set, never cleared. |
| `prctl_dumpable_only_0_or_1` | INVARIANT | per-PR_SET_DUMPABLE: arg2 ∉ {0,1} ⟹ -EINVAL. |
| `getrusage_seqlock_consistent` | INVARIANT | per-getrusage: read_seqbegin_or_lock_irqsave retried on inconsistent read. |
| `setpriority_clamps_nice` | INVARIANT | per-setpriority: niceval clamped to [MIN_NICE, MAX_NICE]. |

### Layer 2: TLA+

`kernel/sys.tla`:
- Per-syscall ∈ { getid, setpgid, setsid, sethostname, setrlimit, prctl, prlimit64, sysinfo, getrusage }.
- Per-state: id-namespace map, uts_sem mode (read/write/free), prctl-flags-set, rlim[], cred lattice.
- Properties:
  - `safety_setsid_unique` — per-setsid: at most one session leader per session pid.
  - `safety_setpgid_session_local` — per-setpgid: new pgrp ⊆ caller's session.
  - `safety_uts_writer_exclusive` — per-uts_sem: writer ⊻ readers.
  - `safety_no_new_privs_monotonic` — per-PR_SET_NO_NEW_PRIVS: PFA_NO_NEW_PRIVS only ever set, never cleared.
  - `safety_rlimit_cur_le_max` — per-rlim: rlim_cur ≤ rlim_max in all states.
  - `safety_dumpable_kernel_only_root` — per-set_dumpable: SUID_DUMP_ROOT (2) only set by kernel paths (suid-exec), never userspace.
  - `liveness_setpgid_eventually_completes` — per-setpgid: under bounded contention, completes or returns error.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sys::getuid` post: ret == from_kuid_munged(current_user_ns(), current_uid()) | `Sys::getuid` |
| `Sys::getpid` post: ret == task_tgid_vnr(current()) | `Sys::getpid` |
| `Sys::setpgid` post: !err ⟹ task_pgrp(p) == pgrp ∧ task_session(p) unchanged | `Sys::setpgid` |
| `Sys::setsid` post: !err ⟹ task_session(gl) == task_pid(gl) ∧ gl.signal.leader == true ∧ ret == pid_vnr(task_pid(gl)) | `Sys::setsid` |
| `Sys::sethostname` post: !err ⟹ utsname().nodename[..len] == tmp[..len] ∧ utsname().nodename[len..] == 0 | `Sys::sethostname` |
| `Sys::do_prlimit` post: !err ⟹ ∀ resource. new ⟹ rlim[resource] == new ∧ old ⟹ *old == prior_rlim | `Sys::do_prlimit` |
| `Sys::prctl(PR_SET_NO_NEW_PRIVS,1,0,0,0)` post: task_no_new_privs(current) | `Sys::prctl` |
| `Sys::prctl(PR_SET_DUMPABLE,v,0,0,0)` post: v ∉ {0,1} ⟹ Err(EINVAL); else get_dumpable(mm) == v | `Sys::prctl` |
| `Sys::sysinfo` post: ret.mem_unit ≥ 1 ∧ ret.uptime ≥ 0 ∧ ret.procs == nr_threads() | `Sys::sysinfo` |
| `Sys::getrusage` post: ret.ru_utime + ret.ru_stime monotonic non-decreasing per-task between calls | `Sys::getrusage` |

### Layer 4: Verus/Creusot functional

`Per-syscall input → POSIX-defined output / errno` semantic equivalence per `man 2 getpid` / `man 2 setpgid` / `man 2 setsid` / `man 2 setrlimit` / `man 2 prlimit` / `man 2 getrusage` / `man 2 prctl` / `man 2 uname` / `man 2 sethostname` / `man 2 sysinfo` and `Documentation/admin-guide/cgroup-v2.rst` (cgroup-aware rlimit/nproc accounting).

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

Miscellaneous-syscall reinforcement:

- **Per-`array_index_nospec(resource, RLIM_NLIMITS)`** — defense against per-Spectre-v1 rlim[] bounds-bypass.
- **Per-`ns_capable(uts_ns.user_ns, CAP_SYS_ADMIN)`** — defense against per-container hostname-escape.
- **Per-`add_device_randomness(hostname)`** — opportunistic entropy harvest from setup-time hostname.
- **Per-`uts_sem` writer-exclusive** — defense against per-torn-read of uts during gethostname.
- **Per-`PFA_NO_NEW_PRIVS` sticky** — defense against per-exec privilege regain.
- **Per-`SUID_DUMP_ROOT` kernel-only** — defense against per-coredump-leak of suid-binary memory.
- **Per-`security_task_prctl` LSM early-return** — defense against per-LSM-bypass via prctl.
- **Per-`set_one_prio_perm`: uid match ∨ CAP_SYS_NICE over target.user_ns** — defense against per-cross-user nice abuse.
- **Per-`check_prlimit_permission`: ns_capable(target.user_ns, CAP_SYS_RESOURCE)** — defense against per-cross-container rlimit-poke.
- **Per-`update_rlimit_cpu` re-arm on RLIMIT_CPU change** — defense against per-stale-cpu-timer.
- **Per-`override_release` PER_LINUX32 fixed table** — defense against per-version-string leak / format-confusion.
- **Per-`mmap_write_lock_killable` for PR_SET_MEMORY_MERGE / PR_SET_THP_DISABLE** — defense against per-mm corruption during prctl.
- **Per-`task_set_no_new_privs` required for unprivileged seccomp** — chain-of-trust enforcement.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `prctl`, `setrlimit`, `sysinfo`, `uname`, `gethostname` user-buffer copies bounds-checked; no `task_struct` / `rlimit` slab leakage.
- **PAX_KERNEXEC** — `do_prctl`, `__sys_setresuid`, `setrlimit`, and per-personality entrypoints reside in W^X kernel text.
- **PAX_RANDKSTACK** — per-syscall kstack offset so repeated `prctl(PR_SET_*)` flooding cannot rely on predictable kstack layout.
- **PAX_REFCOUNT** — `cred->usage`, `task_struct->signal->rlim`, and personality refs saturating-refcounted; underflow on `commit_creds` oopses.
- **PAX_MEMORY_SANITIZE** — freed `cred` / `signal_struct` slabs scrubbed so reused task slots cannot inherit stale uid/gid/rlimit.
- **PAX_UDEREF** — `prctl` and `setrlimit` arg pointers dereferenced via `copy_from_user` only; no implicit user-page deref during ID transitions.
- **PAX_RAP / kCFI** — `prctl` dispatch table and per-personality `set_personality` callbacks type-signatured; mismatched signature is a hard CFI fault.
- **GRKERNSEC_HIDESYM** — `init_task`, `init_cred`, and per-task `cred` addresses hidden from `/proc/kallsyms` and `uname` leaks.
- **GRKERNSEC_DMESG** — sys-call WARN splats (bad rlimit, PR_SET refused) gated to CAP_SYSLOG.
- **prctl(PR_SET_*) audit** — each `PR_SET_*` op logged via the audit subsystem and gated by capability checks (CAP_SYS_RESOURCE for limits, CAP_SETPCAP for caps, CAP_SYS_ADMIN for memory-merge/THP-disable).
- **sys_personality bounded** — only `PER_LINUX`, `PER_LINUX32`, and a known-safe subset accepted; unknown personality bits rejected so an attacker cannot flip `READ_IMPLIES_EXEC` to weaken NX/PAX_MPROTECT.
- **uname HIDESYM** — `uname()` and `/proc/version` redacted under GRKERNSEC_HIDESYM to deny attackers the exact kernel version+config string used for exploit selection.
- **Rationale** — `sys.c` is the catch-all for "modify the calling task's identity, limits, and personality" — every PR_SET, setresuid, setrlimit and personality switch can directly relax later mitigations. Grsec/PaX hardening forces all of them through capability + refcount + audit gates.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- kernel/cred.c credential lifecycle (covered separately if expanded)
- kernel/exit.c, kernel/fork.c (covered in `exit.md`, `fork.md`)
- kernel/seccomp.c (covered in `seccomp.md`)
- kernel/capability.c (covered in `capability.md`)
- kernel/pid.c / kernel/pid_namespace.c (covered in `pid-namespace.md`)
- kernel/user_namespace.c (covered in `user-namespace.md`)
- kernel/utsname.c uts namespace (covered separately if expanded)
- kernel/sysctl.c / fs/proc/proc_sysctl.c (covered separately)
- kernel/sys_ni.c (stub-not-implemented table)
- Architecture-specific prctl backends (arch_prctl_spec_ctrl_*, SVE_*, PAC_*, RISCV_*)
- Implementation code
