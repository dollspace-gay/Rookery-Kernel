---
title: "Tier-5 syscall: exit(2) — syscall 60"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`exit(2)` terminates the **calling thread** (NOT the thread group). This is
the per-thread exit primitive — when a multithreaded program wants to kill
itself, it calls `exit_group(2)` (syscall 231); the libc-visible
`_exit(3)` and `exit(3)` map to `exit_group`, not to this syscall. The
syscall named `exit` in the Linux ABI is the **pthread-internal**
termination called by `pthread_exit(3)` once a thread function returns.

Behavior:
- The current `task_struct` enters `EXIT_ZOMBIE` (or `EXIT_DEAD` if
  auto-reaped — `CLONE_THREAD` non-detached thread, or `CLONE_AUTOREAP`
  set, or parent's `SIGCHLD` is `SIG_IGN`/`SA_NOCLDWAIT`).
- The exit status byte is recorded in `task_struct.exit_code`.
- `CLONE_CHILD_CLEARTID`: zero the recorded user `child_tidptr` and emit
  `FUTEX_WAKE(1)` — this is the kernel-side primitive `pthread_join`
  relies on.
- The robust-futex list is walked and each robust futex's PI-state is
  released / `FUTEX_OWNER_DIED` flag set.
- The signal disposition table (`sighand_struct`) refcount is decremented.
- If this is the LAST thread of the thread group AND the group is being
  reaped via `do_group_exit`, the parent process gets `SIGCHLD`.
- For an ordinary single-thread program: behavior is effectively
  equivalent to `exit_group(2)` because the only thread IS the thread
  group leader.

This Tier-5 covers `kernel/exit.c::SYSCALL_DEFINE1(exit, int error_code)`
plus its immediate `do_exit` workhorse (~30 lines of entry-code surface;
the full `do_exit` is a Tier-3 covered separately in `kernel/exit.c`).

### Acceptance Criteria

- [ ] AC-1: Syscall number is 60 on x86_64; 93 on generic.
- [ ] AC-2: `exit(0)`: task transitions to `EXIT_ZOMBIE` (or `EXIT_DEAD` if auto-reaped). Status byte = 0.
- [ ] AC-3: `exit(42)`: parent's `wait`-returned status has `WEXITSTATUS(s) == 42`.
- [ ] AC-4: `exit(-1)`: stored exit_code low 8 bits = `0xff`; `WEXITSTATUS = 255`.
- [ ] AC-5: Multi-threaded program: `exit(2)` in one thread leaves the other threads running.
- [ ] AC-6: `pthread_join` on the exited thread returns immediately.
- [ ] AC-7: `CLONE_CHILD_CLEARTID` registered: zero is written to `*clear_child_tid` and FUTEX_WAKE(1) issued.
- [ ] AC-8: Robust-futex list: each entry marked `FUTEX_OWNER_DIED`; waiters wake.
- [ ] AC-9: Task held by tracer: `PTRACE_EVENT_EXIT` delivered before zombification.
- [ ] AC-10: `SIG_IGN` for `SIGCHLD` on parent + this is the last thread: task auto-reaped (`EXIT_DEAD`), no zombie left.
- [ ] AC-11: Thread (`task.exit_signal == 0`): auto-reaped; no `SIGCHLD` sent.
- [ ] AC-12: Process (`task.exit_signal == SIGCHLD`): parent receives `SIGCHLD`; task left as zombie until reap.
- [ ] AC-13: Auditd records exit_code and pid.
- [ ] AC-14: Session leader exit: foreground process group receives `SIGHUP` then `SIGCONT`.
- [ ] AC-15: After `exit(2)`, the syscall does not return to userspace (return register undefined).

### Architecture

```rust
#[syscall(nr = 60, abi = "sysv")]
pub fn sys_exit(error_code: i32) -> ! {
    Exit::do_exit((error_code as u32 & 0xff) << 8)
}
```

`Exit::do_exit(code) -> !`:
1. let task = Task::current();
2. /* PTRACE detach */
3. if task.ptraced { Ptrace::release(task); }
4. /* Robust-futex walk */
5. Futex::exit_robust_list(task);
6. /* CLONE_CHILD_CLEARTID */
7. if !task.clear_child_tid.is_null() {
8.   user_access::put_user(0u32, task.clear_child_tid).ok();
9.   Futex::wake(task.clear_child_tid, 1);
10. }
11. /* mm release — wakes vfork parent if any */
12. Mm::mm_release(task);
13. /* Files / fs / sighand / signal release */
14. Files::exit_files(task);
15. Fs::exit_fs(task);
16. Signal::exit_sighand(task);
17. Signal::exit_signal(task);
18. /* cgroup */
19. Cgroup::exit(task);
20. /* Timers */
21. Timer::exit_itimers(task);
22. Timer::exit_posix_timers(task);
23. /* Record exit_code */
24. task.exit_code = code;
25. /* Session leader: SIGHUP fg pgrp */
26. if task.is_session_leader() && task.has_ctty {
27.   Session::disassociate_ctty(task, /*on_exit=*/true);
28. }
29. /* Move to zombie / dead */
30. task.exit_state = if Exit::should_autoreap(task) { EXIT_DEAD } else { EXIT_ZOMBIE };
31. /* Notify */
32. if task.exit_state == EXIT_ZOMBIE {
33.   Exit::notify_parent(task);
34. } else {
35.   Task::release(task);  // free task_struct
36. }
37. /* Schedule away — never returns */
38. Scheduler::schedule_dead();
39. unreachable!();

`Exit::should_autoreap(task) -> bool`:
1. if task.exit_signal == 0 { return true; }                      // thread
2. if task.flags & PF_AUTOREAP != 0 { return true; }              // CLONE_AUTOREAP
3. let parent_sighand = task.real_parent.sighand;
4. let sigaction = parent_sighand.action[SIGCHLD - 1];
5. if sigaction.handler == SIG_IGN { return !task.ptraced; }
6. if sigaction.flags & SA_NOCLDWAIT != 0 { return !task.ptraced; }
7. false

### Out of Scope

- `exit_group(2)` syscall (separate Tier-5 doc).
- `do_exit` internals (Tier-3 `kernel/exit.c`).
- `wait4` / `waitid` reaping (separate Tier-5 docs).
- Robust-futex walker (`kernel/futex/core.c::exit_robust_list` — Tier-3).
- Session/tty disassociation (`drivers/tty/tty_jobctrl.c` — Tier-3).
- Implementation code.

### signature

```c
[[noreturn]] void exit(int status);
```

The libc `_exit(3)` wrapper does NOT call this — it calls `exit_group(2)`.
The only canonical caller of this syscall is `pthread_exit(3)` after
running the thread's destructors.

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `status` | `int` | in | Low 8 bits = exit status byte stored in `task.exit_code`. Upper bits are reserved (kernel preserves them in `exit_code` but `wait*(2)` macros only expose the low 8 bits). |

### return value

This syscall does **not return**. Userspace control is never resumed in
the calling thread. (The kernel does not even attempt to populate the
return register.)

### errors

None. `exit(2)` cannot fail in any way visible to userspace because it does
not return. Internal kernel allocation failures (e.g. allocating the
zombie-tracking data) are handled by panic / OOM-kill paths and do not
propagate to a caller register.

### abi surface

```text
__NR_exit (x86_64)    = 60
__NR_exit (i386)      = 1
__NR_exit (generic)   = 93     /* arm64/riscv/loongarch */
__NR_exit (powerpc)   = 1
__NR_exit (s390x)     = 1
__NR_exit (sparc)     = 1
__NR_exit (mips O32)  = 4001
__NR_exit (mips N64)  = 5058
__NR_exit (alpha)     = 1
__NR_exit (parisc)    = 1
```

### Exit-status encoding (`uapi/linux/wait.h`)

```text
/* As stored in task.exit_code: */
exit_code = (status & 0xff) << 8       /* shifted into the high byte of the 16-bit field */
          | (signo & 0x7f)             /* signal that killed; 0 for normal exit */
          | (core_dumped ? 0x80 : 0);  /* core-dump bit */

/* Reconstructed by wait*() macros: */
WIFEXITED(s)      = ((s & 0x7f) == 0)
WEXITSTATUS(s)    = (s >> 8) & 0xff       /* the 8-bit value passed to exit(2) */
WIFSIGNALED(s)    = (((s & 0x7f) + 1) >> 1) > 0
WTERMSIG(s)       = (s & 0x7f)
WCOREDUMP(s)      = (s & 0x80) != 0
```

For `exit(2)`: signo byte = 0, status byte = `status & 0xff`. So
`WEXITSTATUS = status & 0xff`.

### compatibility contract

REQ-1: The syscall number is **60** on x86_64, **93** on generic-syscall
archs. ABI-stable forever.

REQ-2: `exit(2)` terminates ONLY the calling thread (the `task_struct`
with `current_thread`'s TID). Other threads in the same `tgid` continue
running. This is the central distinction from `exit_group(2)`.

REQ-3: Side-effect ordering (executed in this order inside `do_exit`):
1. PTRACE detach (if traced) — `ptrace_release` clears tracer link.
2. Robust-futex walk — every entry in `task.robust_list` has
   `FUTEX_OWNER_DIED` set and a `FUTEX_WAKE(1)` issued.
3. `CLONE_CHILD_CLEARTID`: zero `task.clear_child_tid` in userspace and
   emit `FUTEX_WAKE(1)`.
4. Close all open files in `files_struct` (only if `files->count == 1`;
   otherwise just drop a reference — other threads share it).
5. Release `mm_struct` reference (`mmput`); if last user, mm is freed.
6. Release `fs_struct`, `sighand_struct`, `signal_struct` references.
7. Detach from cgroup (`cgroup_exit`).
8. Move task to `EXIT_ZOMBIE` (or `EXIT_DEAD` if auto-reaped).
9. Run task exit-notifier list (perf-event, taskstats, audit).
10. Notify parent via `do_notify_parent` (or auto-reap).
11. `schedule()` — never returns.

REQ-4: If the current task is the thread-group leader AND there are still
other threads in the tgid, this task's `exit_state` becomes `EXIT_ZOMBIE`
but it does NOT notify parent — notification is deferred to
`exit_notify` invoked from the LAST thread's `do_exit`.

REQ-5: If `CLONE_VFORK` is set on the task and the parent is suspended on
`vfork_done`, completion is signaled here (in `mm_release`, step 5).

REQ-6: `CLONE_CHILD_CLEARTID` userspace memory write: if the address is
no longer mapped (mm-release happened first via `exec_mmap` for an
inter-leaving execve), the write silently fails — no error to caller (the
caller cannot observe errors). The corresponding `FUTEX_WAKE` may still
be issued from the (now-irrelevant) old address; this is benign.

REQ-7: Pending signals on this task are flushed (the queue is freed when
sighand reference drops).

REQ-8: Per-CPU work assigned to this task (workqueue items in flight,
RCU callbacks queued by this task) is preserved — `do_exit` does not
cancel ongoing kernel work; it merely terminates user-mode execution.
The task struct itself persists until reaped.

REQ-9: Parent notification:
- Default: `SIGCHLD` queued to real_parent (or to debugger if traced).
- Auto-reap conditions (no signal, task becomes `EXIT_DEAD` immediately):
  - `task.exit_signal == 0` (thread, not process).
  - Parent has `SIG_IGN` for `SIGCHLD` AND task is not traced.
  - Parent has `SA_NOCLDWAIT` set on `SIGCHLD`.
  - `task.flags & PF_AUTOREAP` (from `CLONE_AUTOREAP`).

REQ-10: After zombie state, the task is reapable via `wait4(2)`,
`waitpid(2)`, `waitid(2)`, or `wait3(2)` by the parent (or any thread of
the parent's tgid). Reap frees the `task_struct`.

REQ-11: `exit_code` field is overwritten by any subsequent signal-kill
event ONLY if the task's `exit_state` is still `EXIT_ZOMBIE` AND the
signal arrives AFTER notification but BEFORE reap; this race is benign
because the parent always sees the FIRST recorded exit_code.

REQ-12: Audit: `AUDIT_SYSCALL` record emitted with `exit_code` and pid.

REQ-13: Thread-group reduction: if the calling thread is the
thread-group leader but other threads remain, the leader's
`thread_group_empty()` check is false; the leader becomes a "zombie
leader" — it stays in `EXIT_ZOMBIE` and is NOT reaped until ALL its
threads exit. This guarantees the tgid (= PID) remains valid for the
process's lifetime.

REQ-14: A task with `PT_AUTOREAP` flag (set via `clone3(CLONE_AUTOREAP)`)
skips parent notification entirely — task transitions directly to
`EXIT_DEAD` and is freed immediately.

REQ-15: If the task holds open POSIX timers, they are deleted.

REQ-16: If the task holds the controlling tty of its session AND is the
session leader, all processes in the session's foreground group receive
`SIGHUP` followed by `SIGCONT` (job-control terminal hangup). NOTE: this
applies to `do_exit` for ANY exiting session leader, including via
`exit(2)`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `exit_does_not_return` | INVARIANT | `sys_exit` control-flow is `!` (never returns). |
| `robust_list_processed` | INVARIANT | Pre-zombie: robust-list walked to completion. |
| `clear_child_tid_wake` | INVARIANT | If clear_child_tid set: write+wake performed before zombification. |
| `mm_release_before_zombie` | INVARIANT | mm reference dropped before exit_state set. |
| `autoreap_iff_threaded_or_ignored` | INVARIANT | EXIT_DEAD ⟺ should_autoreap(task) is true. |

### Layer 2: TLA+

`kernel/exit.tla`:
- States: RUNNING → PTRACE_DETACH → ROBUST → CLEAR_TID → MM_RELEASE → FILES_RELEASE → SIGHAND_RELEASE → CGROUP_EXIT → SET_EXIT_CODE → NOTIFY → SCHEDULE_DEAD.
- Properties:
  - `safety_no_double_release` — each resource refcount decrement matches one prior get.
  - `safety_parent_observes_exit_code` — wait*() reads the exit_code set in SET_EXIT_CODE.
  - `liveness_eventually_dead` — every entry to do_exit terminates in SCHEDULE_DEAD.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_exit` total: arbitrary i32 → control transfers to do_exit | `sys_exit` |
| `do_exit` post: task.exit_state ∈ {EXIT_ZOMBIE, EXIT_DEAD} | `Exit::do_exit` |
| `exit_robust_list` post: each entry FUTEX_OWNER_DIED | `Futex::exit_robust_list` |
| `should_autoreap` total + boolean | `Exit::should_autoreap` |

### Layer 4: Verus / Creusot functional

POSIX `_exit(2)` semantic equivalence (per-thread variant). Linux man
`exit(2)` quirks (per-thread vs per-process distinction) preserved. LTP
`exit01..exit02` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`exit(2)` reinforcement:

- **Per-robust-futex walk mandatory** — defense against per-thread-death PI inversion.
- **Per-CLONE_CHILD_CLEARTID write+wake idempotent** — defense against per-pthread_join stuck (pthread_join waits on this futex).
- **Per-mm_release before zombification** — defense against per-vfork-stranded parent.
- **Per-session-leader SIGHUP fg pgrp** — defense against per-orphaned-foreground-job.
- **Per-EXIT_ZOMBIE strict** — defense against per-task-struct-UAF (zombie persists until reap).
- **Per-auto-reap conditions narrow** — defense against per-zombie-leak (only auto-reap when policy says so).
- **Per-exit_code preserved across notification** — defense against per-status-overwrite race.
- **Per-audit record emitted** — defense against per-thread-death elision.

### grsecurity / pax surface

- **PAX_RANDKSTACK at syscall entry** — `sys_exit` is the LAST opportunity
  to randomize kernel stack offset for this task; while the task is
  exiting, no further userspace return occurs, so randomization here is
  a no-op except for any RCU-callback context running on this CPU
  immediately after.
- **PaX UDEREF** — `CLONE_CHILD_CLEARTID` user write performed via
  UDEREF / PAN-gated `put_user`; a kernel bug cannot silently scribble
  outside the task's mm via this path.
- **GRKERNSEC_HARDEN_PTRACE** — `PTRACE_EVENT_EXIT` delivered to tracer
  honors yama+grsec policy; cross-credential tracer is denied access to
  the exiting task's final state.
- **GRKERNSEC_PROC restrictions** — exiting task's `/proc/<pid>/status`,
  `/proc/<pid>/exit_code` (if any) are subject to per-uid hide-filter for
  the zombie window.
- **GRKERNSEC_CHROOT** — exit inside chroot: parent (which may be outside
  the chroot) receives `SIGCHLD`; `si_code = CLD_EXITED` and the signal
  source is kernel-credentialed.
- **GRKERNSEC_BRUTE** — repeated `exit(crash_code)` patterns where the
  task is dying from a memory-fault are recorded; sustained exit-pattern
  (e.g., ROP probing causing SIGSEGV exits) increments the per-uid
  brute-force counter.
- **GRKERNSEC_SIGNALS spoof prevention** — `SIGCHLD` notification carries
  `si_pid = task.pid` and `si_uid = task.real_cred.uid`; un-spoofable.
- **Per-grsec `chroot_findtask`** — parent outside chroot of an exiting
  task inside chroot is restricted in its ability to read
  `/proc/<pid>/exe`, `cmdline`, `environ` of the zombie.
- **Per-grsec `task_size_max`** — exit accounting (max RSS, max VM size)
  is recorded against per-uid quotas; sustained pattern of
  `mm-grow-then-exit` is detected.

