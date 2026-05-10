---
title: "Tier-3: kernel/fork.c — Process / thread creation (fork / vfork / clone / clone3)"
tags: ["tier-3", "kernel", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`kernel/fork.c` implements **process and thread creation** for Linux — the kernel side of `fork(2)`, `vfork(2)`, `clone(2)`, `clone3(2)`, and the in-kernel `kernel_thread()` / `kernel_clone()` / `create_io_thread()` / `fork_idle()` paths. Userspace requests are funnelled through `copy_process()` which incrementally duplicates the components of `struct task_struct`: stack + arch state (`dup_task_struct`), credentials (`copy_creds`), memory map (`copy_mm`, copy-on-write when `!CLONE_VM`), open files (`copy_files` → `dup_fd`), filesystem state (`copy_fs`), signal handlers (`copy_sighand`), signal-delivery state (`copy_signal`), namespaces (`copy_namespaces`), I/O context (`copy_io`), seccomp filters (`copy_seccomp`), and the arch-specific thread state (`copy_thread`). The `CLONE_*` flag set selects, per component, whether the child *shares* the parent's structure (refcount-inc) or *duplicates* it (deep-copy / COW). New PIDs are allocated via `alloc_pid()` from the calling task's `nsproxy->pid_ns_for_children`. `CLONE_PIDFD` reserves a pidfs file descriptor via `pidfd_prepare()` so the parent can supervise the child without racing reap. `clone3(2)` accepts a versioned `struct clone_args` (sizes `CLONE_ARGS_SIZE_VER0`/`VER1`/`VER2`) parsed into `struct kernel_clone_args` by `copy_clone_args_from_user()`. Critical for: process creation, container/sandbox runtimes, threading libraries (NPTL/musl), service supervisors (systemd, runit), io_uring SQ workers, and kernel-thread bring-up.

This Tier-3 covers `kernel/fork.c` (~3383 lines).

### Acceptance Criteria

- [ ] AC-1: `fork()` returns child PID in parent, 0 in child; both observe distinct mm post-write (CoW separation).
- [ ] AC-2: `vfork()` parent blocks in `wait_for_vfork_done()` until child's `mm_release` (execve or exit).
- [ ] AC-3: `clone(CLONE_VM|CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD, ...)` creates thread in same tgid, same mm/fs/files, distinct kernel stack.
- [ ] AC-4: `clone(CLONE_THREAD)` without `CLONE_SIGHAND`: returns -EINVAL.
- [ ] AC-5: `clone(CLONE_SIGHAND)` without `CLONE_VM`: returns -EINVAL.
- [ ] AC-6: `clone(CLONE_NEWNS|CLONE_FS)`: returns -EINVAL (mutex).
- [ ] AC-7: `clone3(struct clone_args { flags = CLONE_PIDFD, pidfd = &fd, exit_signal = SIGCHLD })`: returns child PID; *fd is a valid pidfs file descriptor; `pidfd_send_signal(fd, SIGTERM, ...)` reaches child.
- [ ] AC-8: `clone3` with `usize < CLONE_ARGS_SIZE_VER0`: returns -EINVAL.
- [ ] AC-9: `clone3` with `usize > PAGE_SIZE`: returns -E2BIG.
- [ ] AC-10: `clone3` with unknown flag bit: returns -EINVAL.
- [ ] AC-11: `clone3(flags = CLONE_INTO_CGROUP, cgroup = fd)`: child appears in target cgroup at first `cgroup.procs` read.
- [ ] AC-12: `clone3(flags = CLONE_PIDFD_AUTOKILL)` without `CLONE_NNP` from unprivileged user: returns -EPERM.
- [ ] AC-13: `clone3(flags = CLONE_NEWPID|CLONE_THREAD)`: returns -EINVAL.
- [ ] AC-14: RLIMIT_NPROC exceeded for non-INIT_USER user without CAP_SYS_RESOURCE/CAP_SYS_ADMIN: returns -EAGAIN.
- [ ] AC-15: `nr_threads >= max_threads`: returns -EAGAIN.
- [ ] AC-16: `set_tid_address(tidptr)`: subsequent task exit clears *tidptr and futex-wakes (CLONE_CHILD_CLEARTID semantics).
- [ ] AC-17: `copy_process` failure: zero leaked task_struct / pid / pidfile / fd / mm / signal / sighand on all 17 unwind labels.

### Architecture

```
struct KernelCloneArgs {
  flags: u64,                       // CLONE_*
  pidfd: Option<UserPtr<i32>>,      // CLONE_PIDFD out
  child_tid: Option<UserPtr<i32>>,  // CLONE_CHILD_{SET,CLEAR}TID
  parent_tid: Option<UserPtr<i32>>, // CLONE_PARENT_SETTID
  name: Option<&'static str>,       // kthread comm
  exit_signal: i32,                 // (low 8 bits)
  kthread: bool,
  io_thread: bool,
  user_worker: bool,
  no_files: bool,
  stack: usize,
  stack_size: usize,
  tls: usize,
  set_tid: Option<&[Pid]>,          // ≤ MAX_PID_NS_LEVEL
  cgroup: i32,                      // CLONE_INTO_CGROUP fd
  idle: bool,
  fn_: Option<extern "C" fn(*mut c_void) -> i32>,
  fn_arg: *mut c_void,
  cgrp: Option<*Cgroup>,            // pinned target
  cset: Option<*CssSet>,            // pinned css_set
  kill_seq: u32,
}
```

`Fork::kernel_clone(args) -> Result<Pid>`:
1. /* CLONE_EMPTY_MNTNS implies CLONE_NEWNS */
2. if args.flags & CLONE_EMPTY_MNTNS: args.flags |= CLONE_NEWNS.
3. /* pidfd / parent_tid alias check */
4. if (args.flags & CLONE_PIDFD) ∧ (args.flags & CLONE_PARENT_SETTID) ∧ args.pidfd == args.parent_tid: return Err(EINVAL).
5. /* ptrace-event selection */
6. let trace = match { CLONE_UNTRACED: 0; CLONE_VFORK: PTRACE_EVENT_VFORK; exit_signal != SIGCHLD: PTRACE_EVENT_CLONE; else PTRACE_EVENT_FORK } gated on ptrace_event_enabled.
7. let p = Fork::copy_process(None, trace, NUMA_NO_NODE, args)?.
8. add_latent_entropy().
9. trace_sched_process_fork(current, p).
10. let pid = get_task_pid(p, PIDTYPE_PID); let nr = pid_vnr(pid).
11. if CLONE_PARENT_SETTID: put_user(nr, args.parent_tid).
12. if CLONE_VFORK: p.vfork_done = Some(&vfork); init_completion(&vfork); get_task_struct(p).
13. if LRU_GEN_WALKS_MMU ∧ !CLONE_VM: lru_gen_add_mm(p.mm).
14. Sched::wake_up_new_task(p).
15. if trace: ptrace_event_pid(trace, pid).
16. if CLONE_VFORK: Fork::wait_vfork(p, &vfork); on success ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid).
17. put_pid(pid). Ok(nr).

`Fork::copy_process(pid, trace, node, args) -> Result<*TaskStruct>`:
1. Run flag-mutex matrix (REQ-6) — return early Err on any rule violation.
2. Set up multiprocess_signals delayed collector under siglock; -ERESTARTNOINTR if pending fatal.
3. p = Fork::dup_task_struct(current, node).
4. Apply per-args flags: kthread/io_thread/user_worker; comm via strscpy_pad.
5. p.set_child_tid / clear_child_tid per CLONE_CHILD_*TID.
6. Init RT mutex / blocked_lock / lockdep / softirq sanity.
7. Fork::copy_creds(p, flags) — RLIMIT_NPROC check via is_rlimit_overlimit.
8. Reject if nr_threads ≥ max_threads.
9. delayacct_tsk_init; mempolicy dup; sched_fork; perf_event_init_task; audit_alloc; security_task_alloc.
10. Fork::copy_semundo / copy_files / copy_fs / copy_sighand / copy_signal / copy_mm / copy_namespaces / copy_io / copy_thread (in this exact order — error unwind depends on it).
11. stackleak_task_init.
12. if pid is None: pid = Pid::alloc(p.nsproxy.pid_ns_for_children, args.set_tid).
13. if CLONE_PIDFD: Pidfs::prepare(pid, flags{|PIDFD_THREAD|PIDFD_AUTOKILL}, &pidfile); put_user(pidfd, args.pidfd).
14. Clear single-step / syscall trace / latency bits in child.
15. p.pid = pid_nr(pid). CLONE_THREAD: group_leader = current.group_leader; tgid = current.tgid. else: group_leader = p; tgid = p.pid.
16. cgroup_can_fork(p, args). sched_cgroup_fork(p, args). Maybe futex_hash_allocate_default.
17. p.start_time = now_ns(); p.start_boottime = boot_ns().
18. write_lock_irq(tasklist_lock). Set real_parent / parent_exec_id / exit_signal per CLONE_PARENT|CLONE_THREAD.
19. klp_copy_process; sched_core_fork; spin_lock(siglock); rv_task_fork; rseq_fork.
20. Recheck pid_ns adding bit + fatal_signal_pending — last fail points.
21. Fork::copy_seccomp(p). if CLONE_NNP: task_set_no_new_privs.
22. Publish: init_task_pid_links; per-PID-type attach_pid (PIDTYPE_PID always; if thread-group-leader also TGID/PGID/SID and is_child_reaper handling).
23. CLONE_AUTOREAP → signal.autoreap = 1. Thread (non-leader) → signal.nr_threads++ + thread_node link.
24. total_forks++. Drop locks. fd_install(pidfd, pidfile) if any.
25. proc_fork_connector; cgroup_post_fork; sched_post_fork; perf_event_fork; trace_task_newtask; uprobe_copy_process; user_events_fork; copy_oom_score_adj.
26. Ok(p).

`Fork::copy_mm(flags, tsk) -> Result<()>`:
1. tsk.mm = None; tsk.active_mm = None.
2. let oldmm = current.mm; if oldmm.is_none(): return Ok(()) (kthread).
3. let mm = if flags & CLONE_VM { mmget(oldmm); oldmm } else { Fork::dup_mm(tsk, oldmm)? }.
4. tsk.mm = Some(mm); tsk.active_mm = Some(mm).

`Fork::dup_mm(tsk, oldmm) -> Result<*MmStruct>`:
1. mm = allocate_mm()?.
2. memcpy(mm, oldmm, sizeof MmStruct).
3. mm_init(mm, tsk, oldmm.user_ns)?.
4. uprobe_start_dup_mmap().
5. Mm::dup_mmap(mm, oldmm)? — walks each VMA; allocates child vma; sets VM_WRITE→PTE-write-protect on private mappings for CoW; anon_vma_fork.
6. uprobe_end_dup_mmap().
7. mm.hiwater_rss = mm_rss(mm); mm.hiwater_vm = mm.total_vm.
8. binfmt module-pin if mm.binfmt.
9. Ok(mm).

`Fork::copy_files(flags, tsk, no_files) -> Result<()>`:
1. oldf = current.files; if oldf.is_none(): return Ok(()).
2. if no_files: tsk.files = None; return Ok(()).
3. if flags & CLONE_FILES: atomic_inc(&oldf.count); return Ok(()).
4. tsk.files = Some(Fdtable::dup_fd(oldf, None)?).

`Fork::copy_fs(flags, tsk) -> Result<()>`:
1. let fs = current.fs.
2. if flags & CLONE_FS: read_seqlock_excl(&fs.seq). if fs.in_exec { unlock; return Err(EAGAIN). } fs.users += 1; unlock; return Ok(()).
3. tsk.fs = Some(FsStruct::copy(fs)?).

`Fork::copy_sighand(flags, tsk) -> Result<()>`:
1. if flags & CLONE_SIGHAND: refcount_inc(&current.sighand.count); return Ok(()).
2. sig = sighand_cachep.alloc(GFP_KERNEL)?.
3. RCU_INIT_POINTER(tsk.sighand, sig).
4. refcount_set(&sig.count, 1).
5. spin_lock_irq(&current.sighand.siglock); memcpy(sig.action, current.sighand.action, ...); spin_unlock_irq.
6. if flags & CLONE_CLEAR_SIGHAND: flush_signal_handlers(tsk, 0).

`Fork::copy_signal(flags, tsk) -> Result<()>`:
1. if flags & CLONE_THREAD: return Ok(()) (siblings share).
2. sig = signal_cachep.zalloc(GFP_KERNEL)?; tsk.signal = sig.
3. nr_threads = 1; quick_threads = 1; live = 1; sigcnt = 1.
4. thread_head <-> tsk.thread_node circular-list-init.
5. init waitqueue / sigpending / multiprocess / stats_lock.
6. POSIX_TIMERS: hrtimer_setup real_timer; posix_cpu_timers_init_group.
7. memcpy rlim from current.signal.rlim.
8. tty_audit_fork; sched_autogroup_fork; oom_score_adj inherit; cred_guard_mutex init.

`Fork::copy_seccomp(p)` — must hold current.sighand.siglock:
1. get_seccomp_filter(current); p.seccomp = current.seccomp.
2. if task_no_new_privs(current): task_set_no_new_privs(p).
3. if p.seccomp.mode != SECCOMP_MODE_DISABLED: set_task_syscall_work(p, SECCOMP).

`Pidfs::prepare(pid, flags, ret_file) -> Result<i32>`:
1. if !(flags & PIDFD_STALE):
   - lock pid.wait_pidfd.
   - if !pid_has_task(pid, PIDTYPE_PID): return Err(ESRCH).
   - if !(flags & PIDFD_THREAD) ∧ !pid_has_task(pid, PIDTYPE_TGID): return Err(ENOENT).
2. pidfd = get_unused_fd(O_CLOEXEC)?.
3. file = pidfs_alloc_file(pid, flags | O_RDWR)?.
4. *ret_file = file; Ok(pidfd) — fd reserved, file not installed yet (caller installs after last failure point).

### Out of Scope

- `fs/exec.c` execve / binfmt loading (covered separately in `fs/exec.md` Tier-3).
- `kernel/exit.c` exit/wait/zombie reaping (covered in `kernel/task-lifecycle.md` Tier-3).
- `kernel/nsproxy.c` `create_new_namespaces` per-namespace internals (covered in respective per-ns Tier-3s).
- `kernel/cred.c` `copy_creds` cred-lifecycle internals (Tier-3 `kernel/cred.md`).
- `fs/file.c` `dup_fd` fdtable internals (covered in `fs/file.md` Tier-3).
- `fs/pidfs.c` pidfs filesystem internals (Tier-3 `fs/pidfs.md`).
- `arch/*/kernel/process.c` `copy_thread` per-arch register-frame setup.
- `kernel/sched/core.c` `sched_fork` / `wake_up_new_task` scheduler integration (Tier-3 `kernel/sched/00-overview.md`).
- io_uring `io_uring_fork` worker-thread integration (Tier-3 `fs/io_uring.md`).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kernel_clone_args` | in-kernel clone descriptor | `KernelCloneArgs` |
| `struct clone_args` | uapi clone3 descriptor | `CloneArgs` (uapi) |
| `kernel_clone()` | per-fork main entry | `Fork::kernel_clone` |
| `copy_process()` | per-fork core duplicator | `Fork::copy_process` |
| `dup_task_struct()` | per-task-stack alloc | `Fork::dup_task_struct` |
| `copy_creds()` | per-cred dup or share | `Fork::copy_creds` |
| `copy_mm()` | per-mm CLONE_VM or COW | `Fork::copy_mm` |
| `dup_mm()` | per-mm deep clone (CoW VMAs) | `Fork::dup_mm` |
| `copy_files()` | per-files CLONE_FILES or dup_fd | `Fork::copy_files` |
| `copy_fs()` | per-fs CLONE_FS or copy_fs_struct | `Fork::copy_fs` |
| `copy_sighand()` | per-sighand CLONE_SIGHAND or alloc | `Fork::copy_sighand` |
| `copy_signal()` | per-signal CLONE_THREAD or alloc | `Fork::copy_signal` |
| `copy_namespaces()` | per-ns CLONE_NEW* or share | `Fork::copy_namespaces` |
| `copy_io()` | per-io_context CLONE_IO | `Fork::copy_io` |
| `copy_thread()` | arch entry-frame + regs init | `Fork::copy_thread` (arch) |
| `copy_seccomp()` | per-seccomp filter inherit | `Fork::copy_seccomp` |
| `alloc_pid()` | per-pid_ns PID allocation | `Pid::alloc` |
| `pidfd_prepare()` | per-CLONE_PIDFD reserve fd+file | `Pidfs::prepare` |
| `wake_up_new_task()` | per-fork runnable | `Sched::wake_up_new_task` |
| `wait_for_vfork_done()` | per-vfork parent-block | `Fork::wait_vfork` |
| `copy_clone_args_from_user()` | per-clone3 uapi → kernel | `Fork::copy_clone_args_from_user` |
| `clone3_args_valid()` | per-clone3 sanity | `Fork::clone3_args_valid` |
| `SYSCALL_DEFINE0(fork)` | per-fork syscall | `Sys::fork` |
| `SYSCALL_DEFINE0(vfork)` | per-vfork syscall | `Sys::vfork` |
| `SYSCALL_DEFINE5(clone)` | per-clone syscall | `Sys::clone` |
| `SYSCALL_DEFINE2(clone3)` | per-clone3 syscall | `Sys::clone3` |
| `SYSCALL_DEFINE1(set_tid_address)` | per-clear_child_tid set | `Sys::set_tid_address` |
| `SYSCALL_DEFINE1(unshare)` | per-namespace detach | `Sys::unshare` |
| `kernel_thread()` | in-kernel thread spawn | `Fork::kernel_thread` |
| `user_mode_thread()` | in-kernel-launched usermode thread | `Fork::user_mode_thread` |
| `create_io_thread()` | io_uring SQ worker spawn | `Fork::create_io_thread` |
| `fork_idle()` | per-cpu idle task spawn | `Fork::fork_idle` |

### compatibility contract

REQ-1: struct kernel_clone_args (per-include/linux/sched/task.h):
- flags: u64 CLONE_* bitmask (low 32 bits legacy + high 32 bits clone3).
- pidfd: int __user * — out parameter for CLONE_PIDFD.
- child_tid: int __user * — out for CLONE_CHILD_SETTID; cleared on exit if CLONE_CHILD_CLEARTID.
- parent_tid: int __user * — out for CLONE_PARENT_SETTID.
- name: const char * — comm string for in-kernel threads.
- exit_signal: int — signal sent to parent on child exit (typically SIGCHLD).
- kthread, io_thread, user_worker, no_files: u32:1 bitfields — internal-only.
- stack, stack_size: unsigned long — child stack pointer + size (clone3).
- tls: unsigned long — CLONE_SETTLS TLS descriptor.
- set_tid, set_tid_size: pid_t * + size_t — request specific PIDs in nested PID namespaces (CAP_SYS_ADMIN in target ns).
- cgroup: int — fd of target cgroup directory for CLONE_INTO_CGROUP.
- idle: int — fork_idle path only.
- fn, fn_arg: kernel-thread function pointer + arg.
- cgrp, cset: per-CLONE_INTO_CGROUP pinned targets.
- kill_seq: per-pidfs autokill sequence.

REQ-2: struct clone_args (per-include/uapi/linux/sched.h):
- All fields __aligned_u64 for 32/64-bit binary compatibility.
- Versioned by size: CLONE_ARGS_SIZE_VER0 = 64 (.flags..tls), VER1 = 80 (+set_tid, set_tid_size), VER2 = 88 (+cgroup).
- usize > PAGE_SIZE: -E2BIG. usize < VER0: -EINVAL. usize ∈ (VER0..sizeof): tail must be zero (copy_struct_from_user).

REQ-3: CLONE_* low-32 flags (per-include/uapi/linux/sched.h):
- CSIGNAL = 0x000000ff — exit-signal mask (legacy clone, ignored by clone3).
- CLONE_VM = 0x00000100 — share mm_struct; both processes see same VMAs.
- CLONE_FS = 0x00000200 — share fs_struct (root, cwd, umask).
- CLONE_FILES = 0x00000400 — share files_struct (fd table).
- CLONE_SIGHAND = 0x00000800 — share sighand_struct (signal-handler vector).
- CLONE_PIDFD = 0x00001000 — return pidfd in *pidfd (incompatible with CLONE_PARENT_SETTID at same address; not allowed with CLONE_DETACHED).
- CLONE_PTRACE = 0x00002000 — let ptracer auto-attach to child.
- CLONE_VFORK = 0x00004000 — parent blocks until child execve/exit; CLONE_VM also implied.
- CLONE_PARENT = 0x00008000 — child reparented to parent's real_parent (siblings).
- CLONE_THREAD = 0x00010000 — same thread group (tgid); requires CLONE_SIGHAND ⊃ CLONE_VM.
- CLONE_NEWNS = 0x00020000 — new mount namespace; mutex with CLONE_FS / CLONE_NEWUSER.
- CLONE_SYSVSEM = 0x00040000 — share System V SEM_UNDO list.
- CLONE_SETTLS = 0x00080000 — set TLS to args->tls.
- CLONE_PARENT_SETTID = 0x00100000 — store child TID into *parent_tid.
- CLONE_CHILD_CLEARTID = 0x00200000 — on mm_release, clear *child_tid + futex-wake.
- CLONE_DETACHED = 0x00400000 — historical; ignored / reserved (clone3 rejects).
- CLONE_UNTRACED = 0x00800000 — parent's ptracer cannot auto-attach.
- CLONE_CHILD_SETTID = 0x01000000 — child writes its TID into *child_tid.
- CLONE_NEWCGROUP = 0x02000000 — new cgroup namespace.
- CLONE_NEWUTS = 0x04000000 — new UTS namespace.
- CLONE_NEWIPC = 0x08000000 — new SysV IPC namespace.
- CLONE_NEWUSER = 0x10000000 — new user namespace; mutex with CLONE_FS.
- CLONE_NEWPID = 0x20000000 — new PID namespace (mutex with CLONE_THREAD).
- CLONE_NEWNET = 0x40000000 — new network namespace.
- CLONE_IO = 0x80000000 — share io_context (CFQ block-layer scheduling).

REQ-4: CLONE_* high-32 (clone3-only) flags:
- CLONE_CLEAR_SIGHAND = 1ULL<<32 — child's signal handlers reset to SIG_DFL (mutex with CLONE_SIGHAND).
- CLONE_INTO_CGROUP = 1ULL<<33 — attach child to cgroup fd args->cgroup (requires VER2; cgroup ≤ INT_MAX).
- CLONE_AUTOREAP = 1ULL<<34 — child auto-reaped on exit; mutex with CLONE_THREAD/CLONE_PARENT; exit_signal must be 0.
- CLONE_NNP = 1ULL<<35 — child gets PR_SET_NO_NEW_PRIVS; mutex with CLONE_THREAD.
- CLONE_PIDFD_AUTOKILL = 1ULL<<36 — child killed when pidfd closes; requires CLONE_PIDFD ∧ CLONE_AUTOREAP; mutex with CLONE_THREAD; CAP_SYS_ADMIN unless CLONE_NNP.
- CLONE_EMPTY_MNTNS = 1ULL<<37 — child gets empty mnt-ns (implies CLONE_NEWNS).
- CLONE_NEWTIME = 0x00000080 — new time namespace (clone3/unshare only, intersects CSIGNAL).

REQ-5: kernel_clone(args) — main fork driver:
- /* Pre-CLONE_PIDFD/CLONE_PARENT_SETTID alias check */
- if (clone_flags & CLONE_PIDFD) ∧ (clone_flags & CLONE_PARENT_SETTID) ∧ args->pidfd == args->parent_tid: return -EINVAL.
- /* Promote CLONE_EMPTY_MNTNS to CLONE_NEWNS */
- if clone_flags & CLONE_EMPTY_MNTNS: clone_flags |= CLONE_NEWNS.
- /* Determine ptrace event */
- if !(clone_flags & CLONE_UNTRACED):
  - if clone_flags & CLONE_VFORK: trace = PTRACE_EVENT_VFORK.
  - else if args->exit_signal != SIGCHLD: trace = PTRACE_EVENT_CLONE.
  - else: trace = PTRACE_EVENT_FORK.
  - if !ptrace_event_enabled(current, trace): trace = 0.
- p = copy_process(NULL, trace, NUMA_NO_NODE, args).
- if IS_ERR(p): return PTR_ERR(p).
- add_latent_entropy().
- trace_sched_process_fork(current, p).
- pid = get_task_pid(p, PIDTYPE_PID); nr = pid_vnr(pid).
- if clone_flags & CLONE_PARENT_SETTID: put_user(nr, args->parent_tid).
- if clone_flags & CLONE_VFORK: p->vfork_done = &vfork; init_completion(&vfork); get_task_struct(p).
- if LRU_GEN_WALKS_MMU ∧ !CLONE_VM: lru_gen_add_mm(p->mm).
- wake_up_new_task(p).
- if trace: ptrace_event_pid(trace, pid).
- if clone_flags & CLONE_VFORK: wait_for_vfork_done(p, &vfork); on success ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid).
- put_pid(pid); return nr.

REQ-6: copy_process(pid, trace, node, args) — duplicator (returns task_struct *):
- /* Mutex matrix */
- (CLONE_NEWNS | CLONE_FS) == both: -EINVAL.
- (CLONE_NEWUSER | CLONE_FS) == both: -EINVAL.
- CLONE_THREAD ∧ !CLONE_SIGHAND: -EINVAL.
- CLONE_SIGHAND ∧ !CLONE_VM: -EINVAL.
- CLONE_PARENT ∧ current.signal.flags & SIGNAL_UNKILLABLE: -EINVAL (init/container-init forbidden to sibling).
- CLONE_THREAD ∧ (CLONE_NEWUSER | CLONE_NEWPID): -EINVAL.
- CLONE_THREAD ∧ task_active_pid_ns(current) != nsp->pid_ns_for_children: -EINVAL.
- CLONE_PIDFD ∧ CLONE_DETACHED: -EINVAL.
- CLONE_AUTOREAP ∧ (CLONE_THREAD | CLONE_PARENT): -EINVAL.
- CLONE_AUTOREAP ∧ args->exit_signal != 0: -EINVAL.
- CLONE_PARENT ∧ current.signal.autoreap: -EINVAL.
- CLONE_NNP ∧ CLONE_THREAD: -EINVAL.
- CLONE_PIDFD_AUTOKILL: requires CLONE_PIDFD ∧ CLONE_AUTOREAP ∧ !CLONE_THREAD; if !CLONE_NNP requires ns_capable(CAP_SYS_ADMIN).
- /* Delayed-signal collector */
- sigemptyset(&delayed.signal); INIT_HLIST_NODE(&delayed.node).
- spin_lock_irq(&current.sighand.siglock).
- if !CLONE_THREAD: hlist_add_head(&delayed.node, &current.signal.multiprocess).
- recalc_sigpending(); spin_unlock_irq.
- if task_sigpending(current): -ERESTARTNOINTR (kill before fork).
- /* Stage 1: dup_task_struct (kernel stack + arch state + canary) */
- p = dup_task_struct(current, node).
- p->flags clears PF_KTHREAD then |= per-args.kthread/user_worker/io_thread.
- p->set_child_tid = (CLONE_CHILD_SETTID ? args->child_tid : NULL).
- p->clear_child_tid = (CLONE_CHILD_CLEARTID ? args->child_tid : NULL).
- /* Stage 2: subsystem inits + copy_* in fixed order with goto-error unwind */
- ftrace_graph_init_task; rt_mutex_init_task; raw_spin_lock_init(&p->blocked_lock).
- copy_creds(p, clone_flags) — RLIMIT_NPROC enforcement + cred clone or share.
- check is_rlimit_overlimit(UCOUNT_RLIMIT_NPROC, RLIMIT_NPROC).
- check data_race(nr_threads >= max_threads): -EAGAIN.
- delayacct_tsk_init; p->flags |= PF_FORKNOEXEC; INIT_LIST_HEAD children/sibling.
- rcu_copy_process(p); init_sigpending(&p->pending).
- p->utime/stime/gtime = 0; prev_cputime_init(&p->prev_cputime).
- io_uring_fork(p) if CONFIG_IO_URING.
- task_io_accounting_init; acct_clear_integrals; posix_cputimers_init; tick_dep_init_task.
- p->io_context = NULL initially (set later by copy_io).
- cgroup_fork(p).
- if args->kthread: set_kthread_struct(p).
- NUMA: mpol_dup(p->mempolicy).
- unwind_task_init(p).
- sched_fork(clone_flags, p) — assigns CPU; vrun-time init.
- perf_event_init_task(p, clone_flags).
- audit_alloc(p).
- shm_init_task; security_task_alloc(p, clone_flags).
- copy_semundo(clone_flags, p) — System V SEM_UNDO inherit.
- copy_files(clone_flags, p, args->no_files).
- copy_fs(clone_flags, p).
- copy_sighand(clone_flags, p).
- copy_signal(clone_flags, p).
- copy_mm(clone_flags, p).
- copy_namespaces(clone_flags, p).
- copy_io(clone_flags, p).
- copy_thread(p, args) — arch state (regs / pt_regs frame / stack pointer / TLS).
- stackleak_task_init(p).
- /* Stage 3: PID allocation */
- if pid != &init_struct_pid: pid = alloc_pid(p->nsproxy->pid_ns_for_children, args->set_tid, args->set_tid_size). On error: PTR_ERR.
- /* Stage 4: CLONE_PIDFD */
- if CLONE_PIDFD:
  - flags = PIDFD_STALE | (CLONE_THREAD ? PIDFD_THREAD : 0) | (CLONE_PIDFD_AUTOKILL ? PIDFD_AUTOKILL : 0).
  - pidfd_prepare(pid, flags, &pidfile) — reserves fd + alloc pidfs file (not installed).
  - put_user(pidfd, args->pidfd).
- /* Stage 5: tracing/syscall clear */
- user_disable_single_step(p); clear_task_syscall_work(p, SYSCALL_TRACE | SYSCALL_EMU); clear_tsk_latency_tracing.
- /* Stage 6: PID + thread-group linkage */
- p->pid = pid_nr(pid).
- if CLONE_THREAD: p->group_leader = current->group_leader; p->tgid = current->tgid.
- else: p->group_leader = p; p->tgid = p->pid.
- p->task_works = NULL; clear_posix_cputimers_work.
- /* Stage 7: cgroup can_fork + sched_cgroup_fork */
- cgroup_can_fork(p, args) — runs each cgroup subsys's can_fork hook + CLONE_INTO_CGROUP attach.
- sched_cgroup_fork(p, args) — re-place task on the correct runqueue under final cgroup.
- need_futex_hash_allocate_default(clone_flags) → futex_hash_allocate_default.
- /* Stage 8: tasklist publication */
- p->start_time = ktime_get_ns(); p->start_boottime = ktime_get_boottime_ns().
- write_lock_irq(&tasklist_lock).
- if CLONE_PARENT|CLONE_THREAD: p->real_parent = current->real_parent; exit_signal = -1 if THREAD else inherited.
- else: p->real_parent = current; exit_signal = args->exit_signal.
- klp_copy_process(p); sched_core_fork(p).
- spin_lock(&current->sighand->siglock); rv_task_fork(p); rseq_fork(p, clone_flags).
- if !(ns_of_pid(pid)->pid_allocated & PIDNS_ADDING): -ENOMEM (zap_pid_ns_processes raced).
- if fatal_signal_pending(current): -EINTR.
- /* Past this point no failure paths */
- copy_seccomp(p) — under siglock to sync with set_no_new_privs.
- if CLONE_NNP: task_set_no_new_privs(p).
- init_task_pid_links(p); if p->pid:
  - ptrace_init_task(p, CLONE_PTRACE | trace).
  - init_task_pid(p, PIDTYPE_PID, pid).
  - if thread_group_leader(p):
    - init_task_pid(p, PIDTYPE_TGID, pid).
    - init_task_pid(p, PIDTYPE_PGID, task_pgrp(current)).
    - init_task_pid(p, PIDTYPE_SID, task_session(current)).
    - if is_child_reaper(pid): WRITE_ONCE(ns->child_reaper, p); p->signal->flags |= SIGNAL_UNKILLABLE.
    - p->signal->shared_pending.signal = delayed.signal; tty inherit.
    - has_child_subreaper inheritance.
    - if CLONE_AUTOREAP: p->signal->autoreap = 1.
    - list_add_tail(&p->sibling, &p->real_parent->children).
    - list_add_tail_rcu(&p->tasks, &init_task.tasks).
    - attach_pid PIDTYPE_TGID/PGID/SID; __this_cpu_inc(process_counts).
  - else (thread):
    - current->signal->nr_threads++; quick_threads++; atomic_inc(&signal.live); sigcnt++.
    - task_join_group_stop(p); list_add_tail_rcu(&p->thread_node, &p->signal->thread_head).
  - attach_pid(p, PIDTYPE_PID); nr_threads++.
- total_forks++; hlist_del_init(&delayed.node).
- spin_unlock(&current->sighand->siglock); syscall_tracepoint_update(p); write_unlock_irq(&tasklist_lock).
- if pidfile: fd_install(pidfd, pidfile).
- proc_fork_connector(p); cgroup_post_fork(p, args); sched_post_fork(p); perf_event_fork(p).
- trace_task_newtask(p, clone_flags); uprobe_copy_process(p, clone_flags); user_events_fork(p, clone_flags).
- copy_oom_score_adj(clone_flags, p).
- return p.

REQ-7: dup_task_struct(orig, node):
- if node == NUMA_NO_NODE: node = tsk_fork_get_node(orig).
- tsk = alloc_task_struct_node(node) (slab).
- arch_dup_task_struct(tsk, orig) — arch-specific copy (e.g. FPU state holders).
- alloc_thread_stack_node(tsk, node) — vmap'd or page-allocated kernel stack.
- THREAD_INFO_IN_TASK: refcount_set(&tsk->stack_refcount, 1).
- account_kernel_stack(tsk, 1); scs_prepare(tsk, node) — shadow-call-stack.
- SECCOMP: tsk->seccomp.filter = NULL (filled later under siglock).
- setup_thread_stack; clear_user_return_notifier; clear_tsk_need_resched; set_task_stack_end_magic; clear_syscall_work_syscall_user_dispatch.
- STACKPROTECTOR: tsk->stack_canary = get_random_canary().
- cpus_ptr / cpus_mask + dup_user_cpus_ptr.
- refcount_set(&tsk->rcu_users, 2); refcount_set(&tsk->usage, 1).
- splice_pipe, task_frag.page, wake_q.next, worker_private = NULL.
- kcov_task_init; kmsan_task_create; kmap_local_fork; MEMCG: active_memcg = NULL; SCHED_MM_CID: cid = MM_CID_UNSET.

REQ-8: copy_mm(clone_flags, tsk):
- tsk->mm = NULL; tsk->active_mm = NULL.
- oldmm = current->mm.
- if !oldmm: return 0 (kernel thread).
- if clone_flags & CLONE_VM: mmget(oldmm); mm = oldmm (refcount-inc + share).
- else: mm = dup_mm(tsk, current->mm) — deep clone:
  - allocate_mm(); memcpy(mm, oldmm); mm_init(mm, tsk, mm->user_ns).
  - uprobe_start_dup_mmap; dup_mmap(mm, oldmm) — per-VMA clone with CoW write-protect.
  - uprobe_end_dup_mmap; hiwater_rss / hiwater_vm set.
  - binfmt module-pin.
- tsk->mm = mm; tsk->active_mm = mm.

REQ-9: copy_files(clone_flags, tsk, no_files):
- oldf = current->files; if !oldf: return 0.
- if no_files (kthread/user_worker): tsk->files = NULL.
- if clone_flags & CLONE_FILES: atomic_inc(&oldf->count); share.
- else: newf = dup_fd(oldf, NULL) — clones fdtable (file refcounts incremented).
- tsk->files = newf.

REQ-10: copy_fs(clone_flags, tsk):
- fs = current->fs.
- if CLONE_FS:
  - read_seqlock_excl(&fs->seq).
  - if fs->in_exec: read_sequnlock_excl; return -EAGAIN (racing exec).
  - fs->users++; read_sequnlock_excl; return 0.
- else: tsk->fs = copy_fs_struct(fs) — deep clone of root/pwd path refs.

REQ-11: copy_sighand(clone_flags, tsk):
- if CLONE_SIGHAND: refcount_inc(&current->sighand->count); share.
- else: sig = kmem_cache_alloc(sighand_cachep, GFP_KERNEL); RCU_INIT_POINTER(tsk->sighand, sig).
- refcount_set(&sig->count, 1).
- spin_lock_irq(&current->sighand->siglock); memcpy(sig->action, current->sighand->action, ...); spin_unlock_irq.
- if CLONE_CLEAR_SIGHAND: flush_signal_handlers(tsk, 0) (reset !SIG_IGN handlers to SIG_DFL).

REQ-12: copy_signal(clone_flags, tsk):
- if CLONE_THREAD: return 0 (siblings share signal_struct).
- sig = kmem_cache_zalloc(signal_cachep, GFP_KERNEL); tsk->signal = sig.
- nr_threads = quick_threads = 1; atomic_set(&sig->live, 1); refcount_set(&sig->sigcnt, 1).
- thread_head <-> tsk->thread_node circular list-init.
- init_waitqueue_head(&sig->wait_chldexit).
- curr_target = tsk; init_sigpending(&shared_pending); INIT_HLIST_HEAD(&sig->multiprocess); seqlock_init(&sig->stats_lock).
- POSIX_TIMERS: hrtimer_setup(&sig->real_timer, ...); posix_cpu_timers_init_group.
- memcpy(sig->rlim, current->signal->rlim, ...).
- tty_audit_fork; sched_autogroup_fork.
- oom_score_adj / oom_score_adj_min inherit from current.
- cred_guard_mutex / exec_update_lock init.

REQ-13: copy_seccomp(p) — must hold current->sighand->siglock:
- get_seccomp_filter(current); p->seccomp = current->seccomp.
- if task_no_new_privs(current): task_set_no_new_privs(p) (sync nnp).
- if p->seccomp.mode != SECCOMP_MODE_DISABLED: set_task_syscall_work(p, SECCOMP).

REQ-14: copy_namespaces (per-kernel/nsproxy.c):
- If no CLONE_NEW{NS,UTS,IPC,PID,NET,USER,CGROUP,TIME} bits set: refcount-inc nsproxy; share.
- else: create_new_namespaces() — allocates fresh nsproxy and per-ns objects gated on user_ns capabilities.

REQ-15: copy_io(clone_flags, tsk):
- BLOCK + IO scheduling: if CLONE_IO inherits parent's io_context (refcount-inc); else fresh per-task io_context allocated on first I/O.

REQ-16: copy_thread(tsk, args) — arch hook (per-arch/x86/kernel/process.c, arch/arm64/kernel/process.c, ...):
- Sets up pt_regs at top-of-stack; clears callee-saved.
- if args->kthread / args->fn: child entry = ret_from_kernel_thread → fn(fn_arg).
- else: copy parent's pt_regs (user mode); if args->stack != 0: regs->sp = stack + stack_size (or stack on grow-up archs).
- if CLONE_SETTLS: regs->fs_base / regs->gs_base / TLS-segment-update per arch.
- CLONE_VM ∧ stack==0 (vfork): leave parent's stack pointer.

REQ-17: alloc_pid(ns, set_tid, set_tid_size):
- Allocates struct pid + idr_alloc per level from outermost ns through inner-most.
- set_tid_size ≤ MAX_PID_NS_LEVEL: caller may request specific PID per level (requires CAP_SYS_ADMIN in target user_ns; -EEXIST on collision).
- On failure: free already-allocated levels; return ERR_PTR(-errno).

REQ-18: pidfd_prepare(pid, flags, &pidfile):
- If !PIDFD_STALE: guard(spinlock_irq)(&pid->wait_pidfd.lock).
  - !pid_has_task(pid, PIDTYPE_PID): -ESRCH (reaped already).
  - !PIDFD_THREAD ∧ !pid_has_task(pid, PIDTYPE_TGID): -ENOENT (not a tg leader).
- CLASS(get_unused_fd)(O_CLOEXEC) — reserve fd slot.
- pidfs_file = pidfs_alloc_file(pid, flags | O_RDWR) — pidfs inode.
- *ret_file = pidfs_file; return take_fd(pidfd) (fd is reserved, file not installed yet — caller fd_installs after no-more-failure point).

REQ-19: clone3 entry SYSCALL_DEFINE2(clone3, uargs, size):
- ARCH_BROKEN_SYS_CLONE3: -ENOSYS.
- kargs.set_tid = on-stack array (size MAX_PID_NS_LEVEL).
- copy_clone_args_from_user(&kargs, uargs, size).
- !clone3_args_valid(&kargs): -EINVAL.
- return kernel_clone(&kargs).

REQ-20: copy_clone_args_from_user(kargs, uargs, usize):
- BUILD_BUG_ON offset checks for VER0/VER1/VER2 sizes.
- usize > PAGE_SIZE: -E2BIG. usize < VER0: -EINVAL.
- copy_struct_from_user(&args, sizeof(args), uargs, usize) — zero-extends short, rejects non-zero tail beyond known size.
- args.set_tid_size > MAX_PID_NS_LEVEL: -EINVAL.
- (set_tid_size > 0) XOR (set_tid != NULL): -EINVAL.
- args.exit_signal & ~CSIGNAL: -EINVAL.
- !valid_signal(args.exit_signal): -EINVAL.
- CLONE_INTO_CGROUP: args.cgroup ≤ INT_MAX ∧ usize ≥ VER2.
- Populate kargs from args; copy_from_user set_tid array.

REQ-21: clone3_args_valid(kargs):
- flags ⊂ (CLONE_LEGACY_FLAGS | CLONE_CLEAR_SIGHAND | CLONE_INTO_CGROUP | CLONE_AUTOREAP | CLONE_NNP | CLONE_PIDFD_AUTOKILL | CLONE_EMPTY_MNTNS) — unknown flag → false.
- (CLONE_DETACHED | (CSIGNAL & ~CLONE_NEWTIME)) ∩ flags: false (reused bits).
- (CLONE_SIGHAND | CLONE_CLEAR_SIGHAND) == both: false.
- (CLONE_THREAD | CLONE_PARENT) ∧ exit_signal != 0: false.
- clone3_stack_valid(kargs) — stack==0 ⟹ stack_size==0; stack!=0 ⟹ access_ok(stack, stack_size); !STACK_GROWSUP: stack += stack_size.

REQ-22: SYSCALL_DEFINE0(fork):
- CONFIG_MMU: args = { exit_signal = SIGCHLD }; kernel_clone(&args).
- !CONFIG_MMU: -EINVAL (fork unsupported on nommu; vfork only).

REQ-23: SYSCALL_DEFINE0(vfork):
- args = { flags = CLONE_VFORK | CLONE_VM, exit_signal = SIGCHLD }; kernel_clone(&args).

REQ-24: SYSCALL_DEFINE5(clone, clone_flags, newsp, parent_tidptr, child_tidptr, tls) (and CONFIG_CLONE_BACKWARDS{,2,3} arg-order variants):
- args = { flags = lower_32_bits(clone_flags) & ~CSIGNAL, pidfd = parent_tidptr, child_tid = child_tidptr, parent_tid = parent_tidptr, exit_signal = clone_flags & CSIGNAL, stack = newsp, tls }.
- kernel_clone(&args).
- Note: legacy clone aliases pidfd onto parent_tidptr — copy_process rejects collision with CLONE_PARENT_SETTID at same address.

REQ-25: SYSCALL_DEFINE1(set_tid_address, tidptr):
- current->clear_child_tid = tidptr.
- return task_pid_vnr(current).

REQ-26: fork_idle(cpu) (init-time per-CPU idle):
- args = { flags = CLONE_VM, fn = idle_dummy, kthread = 1, idle = 1 }.
- copy_process(&init_struct_pid, 0, cpu_to_node(cpu), &args) — reuses init_struct_pid (pid 0).
- init_idle_pids(task); init_idle(task, cpu).

REQ-27: create_io_thread(fn, arg, node) (io_uring SQ workers):
- flags = CLONE_FS|CLONE_FILES|CLONE_SIGHAND|CLONE_THREAD|CLONE_IO|CLONE_VM|CLONE_UNTRACED.
- args = { flags, fn, fn_arg = arg, io_thread = 1, user_worker = 1 }.
- copy_process(NULL, 0, node, &args) — returns inactive task; caller wake_up_new_task.
- All signals blocked except SIGKILL/SIGSTOP.

REQ-28: kernel_thread / user_mode_thread:
- args.flags = (flags | CLONE_VM | CLONE_UNTRACED) & ~CSIGNAL.
- args.exit_signal = flags & CSIGNAL.
- kernel_thread also sets args.kthread = 1; passes args.name for comm.

REQ-29: Error-path unwind labels (bad_fork_* goto chain) — strict reverse-order undo:
- bad_fork_core_free → sched_core_free + drop siglock + tasklist write_unlock.
- bad_fork_cancel_cgroup → cgroup_cancel_fork.
- bad_fork_put_pidfd → fput(pidfile) + put_unused_fd(pidfd).
- bad_fork_free_pid → free_pid(pid) if not init_struct_pid.
- bad_fork_cleanup_thread → exit_thread.
- bad_fork_cleanup_io → exit_io_context if p->io_context.
- bad_fork_cleanup_namespaces → exit_nsproxy_namespaces.
- bad_fork_cleanup_mm → mm_clear_owner + mmput.
- bad_fork_cleanup_signal → free_signal_struct if !CLONE_THREAD.
- bad_fork_cleanup_sighand → __cleanup_sighand.
- bad_fork_cleanup_fs / files / semundo / security / audit / perf.
- bad_fork_sched_cancel_fork → sched_cancel_fork.
- bad_fork_cleanup_policy → lockdep_free_task + mpol_put.
- bad_fork_cleanup_delayacct → io_uring_free + delayacct_tsk_free.
- bad_fork_cleanup_count → dec_rlimit_ucounts + exit_creds.
- bad_fork_free → WRITE_ONCE(p->__state, TASK_DEAD) + exit_task_stack_account + put_task_stack + delayed_free_task.
- fork_out → siglock + hlist_del_init(&delayed.node) + return ERR_PTR(retval).

REQ-30: unshare(2) (SYSCALL_DEFINE1(unshare, unshare_flags)) — re-enters per-component "copy_*" logic against current to drop sharing without forking; not covered in depth here (orthogonal Tier-3 in `kernel/task-lifecycle.md`).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_mutex_matrix_enforced` | INVARIANT | per-copy_process: every (CLONE_THREAD/!CLONE_SIGHAND), (CLONE_SIGHAND/!CLONE_VM), (CLONE_NEWNS/CLONE_FS), (CLONE_NEWUSER/CLONE_FS), (CLONE_AUTOREAP/CLONE_THREAD) violation rejected with -EINVAL before resource alloc. |
| `error_unwind_no_leak` | INVARIANT | per-copy_process: every bad_fork_* label releases exactly the subset of resources acquired before that point — no double-put, no missed put. |
| `pidfile_fd_atomicity` | INVARIANT | per-CLONE_PIDFD: fd is fd_install'd iff copy_process succeeded; on failure path: fput(pidfile) + put_unused_fd(pidfd). |
| `tasklist_lock_ordering` | INVARIANT | per-copy_process: tasklist_lock taken in write mode before siglock and dropped after; never recursive. |
| `siglock_held_for_seccomp_copy` | INVARIANT | per-copy_seccomp: current.sighand.siglock asserted held (assert_spin_locked). |
| `clone3_size_bounds` | INVARIANT | per-clone3: VER0 ≤ usize ≤ PAGE_SIZE; copy_struct_from_user enforces zero-extension. |
| `set_tid_size_bounded` | INVARIANT | per-copy_clone_args_from_user: set_tid_size ≤ MAX_PID_NS_LEVEL. |
| `pidfd_parent_tid_alias_rejected` | INVARIANT | per-kernel_clone: same address for pidfd and parent_tid with both flags → -EINVAL. |

### Layer 2: TLA+

`kernel/fork.tla`:
- States: idle → validate → dup_task → copy_creds → copy_files → copy_fs → copy_sighand → copy_signal → copy_mm → copy_namespaces → copy_io → copy_thread → alloc_pid → pidfd_prepare → publish → wake → vfork_wait? → done.
- Per-stage failure transition to corresponding unwind state.
- Properties:
  - `safety_no_partial_publication` — per-publish: if write_lock_irq(tasklist_lock) acquired then either full publication completes or full bad_fork_core_free rollback (no half-attached PID).
  - `safety_pid_unique` — per-alloc_pid: every successful copy_process yields a pid_nr unique within its pid_ns_for_children.
  - `safety_thread_group_consistent` — per-CLONE_THREAD: tgid == current.tgid, group_leader == current.group_leader, sighand shared, signal_struct shared.
  - `safety_vfork_completion` — per-CLONE_VFORK: parent.vfork_done == completion-pointer until child mm_release; then complete + put_task_struct.
  - `liveness_fork_terminates` — per-kernel_clone: every legal args either returns nr ≥ 0 or returns -errno; no hangs except the explicit CLONE_VFORK wait.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Fork::kernel_clone` post: returns Ok(nr) ∨ Err(errno); on Ok, child is on a runqueue. | `Fork::kernel_clone` |
| `Fork::copy_process` post: on Ok, p has refcount usage=1 + rcu_users=2 + stack_refcount=1. | `Fork::copy_process` |
| `Fork::dup_task_struct` post: stack canary fresh; SCS prepared; mm_cid unset. | `Fork::dup_task_struct` |
| `Fork::copy_mm` post: CLONE_VM ⟹ tsk.mm == current.mm ∧ mm.users >= 2; !CLONE_VM ⟹ new mm with same VMA layout, all private-writable PTEs write-protected. | `Fork::copy_mm` |
| `Fork::copy_files` post: CLONE_FILES ⟹ tsk.files == current.files ∧ files.count >= 2; !CLONE_FILES ⟹ new fdtable with file refcounts +1 each. | `Fork::copy_files` |
| `Fork::copy_sighand` post: CLONE_SIGHAND ⟹ tsk.sighand == current.sighand ∧ count >= 2; CLONE_CLEAR_SIGHAND ⟹ all !SIG_IGN reset to SIG_DFL. | `Fork::copy_sighand` |
| `Fork::copy_signal` post: !CLONE_THREAD ⟹ fresh signal_struct with nr_threads=1; CLONE_THREAD ⟹ shared signal.live+= 1. | `Fork::copy_signal` |
| `Fork::copy_seccomp` post: seccomp filter refcount incremented; NNP synced from parent. | `Fork::copy_seccomp` |
| `Pidfs::prepare` post: on Ok, pidfd reserved + pidfs file allocated; on Err, no fd slot consumed. | `Pidfs::prepare` |

### Layer 4: Verus/Creusot functional

`Per-fork: validate-flags → dup_task_struct → copy_creds → copy_files → copy_fs → copy_sighand → copy_signal → copy_mm → copy_namespaces → copy_io → copy_thread → alloc_pid → pidfd_prepare → cgroup_can_fork → write_lock_irq → publish (attach_pid PID/TGID/PGID/SID) → wake_up_new_task → vfork_wait (if CLONE_VFORK)` semantic equivalence to upstream — per-Documentation/admin-guide/clone3.rst + man clone(2)/clone3(2)/fork(2)/vfork(2).

### hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

Fork reinforcement:

- **Per-flag-mutex matrix rejected pre-alloc** — defense against per-impossible-task creation (CLONE_THREAD without CLONE_SIGHAND, etc.).
- **Per-strict goto-unwind labels** — defense against per-partial-task leak on any failure path.
- **Per-CLONE_PIDFD_AUTOKILL gated on CLONE_NNP or CAP_SYS_ADMIN** — defense against unprivileged privilege-escalation-via-autokill.
- **Per-PID-namespace adding-bit recheck before publish** — defense against per-zap_pid_ns_processes-race.
- **Per-fatal_signal_pending recheck before publish** — defense against per-mid-fork-SIGKILL miss.
- **Per-RLIMIT_NPROC + max_threads enforced early** — defense against per-fork-bomb.
- **Per-clone3 size + tail-zero validation (copy_struct_from_user)** — defense against per-uninit-args + per-ABI-skew.
- **Per-set_tid_size ≤ MAX_PID_NS_LEVEL** — defense against per-namespace-stack-overflow.
- **Per-CLONE_PIDFD reserves fd without install** — defense against per-mid-fork-leak into child's fdtable.
- **Per-tasklist_lock write-mode publication** — defense against per-half-visible-task (proc/ps races).
- **Per-stack canary refreshed in dup_task_struct** — defense against per-canary-replay across processes.
- **Per-SCS (shadow call stack) re-prepared** — defense against per-return-address corruption inheritance.
- **Per-seccomp filter ref + NNP sync under siglock** — defense against per-set_no_new_privs / fork race.

