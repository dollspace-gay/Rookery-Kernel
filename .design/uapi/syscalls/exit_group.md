# Tier-5 syscall: exit_group(2) — syscall 231

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/exit.c (sys_exit_group, do_group_exit)
  - include/linux/sched.h (struct signal_struct.group_exit_code, .flags & SIGNAL_GROUP_EXIT)
  - arch/x86/entry/syscalls/syscall_64.tbl (231  common  exit_group)
-->

## Summary

`exit_group(2)` terminates the **entire thread group** (i.e. the whole
process from POSIX's perspective). Every thread in the calling task's
`tgid` is sent `SIGKILL` (or rather, has `SIGNAL_GROUP_EXIT` flagged and
is woken), they all run through `do_exit`, and finally the thread group
leader is reaped (or becomes a zombie awaiting parent reap). This is the
syscall that `libc::_exit(3)`, `libc::exit(3)`, and `return main()` all
ultimately invoke — it implements the POSIX process-exit semantic.

Behavior:
- Set `signal->group_exit_code = (status & 0xff) << 8`.
- Set `signal->flags |= SIGNAL_GROUP_EXIT`.
- For every thread `t` in current task's thread group:
  - `set_tsk_thread_flag(t, TIF_SIGPENDING)` — they pick up the exit at
    next syscall return / preemption.
  - The threads on entering `get_signal()` see `SIGNAL_GROUP_EXIT` and
    call `do_group_exit` (which re-enters via `do_exit`).
- The CALLING task itself enters `do_exit(group_exit_code)`.
- After all threads exit, the parent is notified once via `SIGCHLD`.

The distinction from per-thread `exit(2)`:
- `exit(2)` (syscall 60): only the calling thread terminates. The
  process continues running with one fewer thread.
- `exit_group(2)` (syscall 231): all threads terminate. The process ends.

Critical for: every program's normal termination (libc emits this), every
runtime crash handler that wants the whole process down, every
container-init handling SIGTERM, every test framework cleaning up
worker pools.

This Tier-5 covers `kernel/exit.c::SYSCALL_DEFINE1(exit_group, ...)` plus
its immediate `do_group_exit` workhorse (~50 lines of entry surface).

## Signature

```c
[[noreturn]] void exit_group(int status);
```

This IS the syscall that libc `_exit(3)` and `exit(3)` map to.

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `status` | `int` | in | Low 8 bits = exit status byte stored in `signal->group_exit_code`. Upper bits ignored by wait*() macros. |

## Return value

This syscall does **not return**. The calling task is the one that
records the group_exit_code; it then runs `do_exit` itself (which never
returns).

## Errors

None directly. Internal allocation failures during reaping are handled by
the kernel and not propagated.

## ABI surface

```text
__NR_exit_group (x86_64)   = 231
__NR_exit_group (i386)     = 252
__NR_exit_group (generic)  = 94      /* arm64/riscv/loongarch */
__NR_exit_group (powerpc)  = 234
__NR_exit_group (s390x)    = 248
__NR_exit_group (sparc)    = 188
__NR_exit_group (mips O32) = 4246
__NR_exit_group (mips N64) = 5205
```

### Exit-status encoding

Same as `exit(2)` (see `exit.md` REQ-2 / ABI surface). For
`exit_group(status)`: signal byte = 0, status byte = `status & 0xff`,
`WEXITSTATUS = status & 0xff`, `WIFEXITED = 1`, `WIFSIGNALED = 0`.

### `signal_struct.flags`

```text
SIGNAL_STOP_STOPPED   = 0x00000001
SIGNAL_STOP_CONTINUED = 0x00000002
SIGNAL_GROUP_EXIT     = 0x00000004    /* set by sys_exit_group; consulted by get_signal() */
SIGNAL_CLD_STOPPED    = 0x00000010
SIGNAL_CLD_CONTINUED  = 0x00000020
SIGNAL_UNKILLABLE     = 0x00000040    /* init in pid_ns root */
```

## Compatibility contract

REQ-1: The syscall number is **231** on x86_64, **94** on generic-syscall
archs. ABI-stable forever.

REQ-2: `exit_group(status)` terminates ALL threads in the calling
`task->signal->thread_head` list. This is the POSIX-process-exit
primitive.

REQ-3: Side-effect ordering inside `do_group_exit`:
1. Acquire `task->sighand->siglock`.
2. If `signal->flags & SIGNAL_GROUP_EXIT` already set:
   - Use the already-recorded `group_exit_code` (don't overwrite).
   - Release siglock.
3. Else:
   - Set `signal->group_exit_code = (status & 0xff) << 8`.
   - Set `signal->flags |= SIGNAL_GROUP_EXIT`.
   - Walk `signal->thread_head`: for each other thread `t` (not current),
     `sigaddset(&t->pending.signal, SIGKILL); signal_wake_up(t, 1);`
     — this guarantees they enter `get_signal()` ASAP.
4. Release siglock.
5. Call `do_exit(signal->group_exit_code)` for the calling thread.

REQ-4: Concurrent `exit_group` calls from multiple threads: only the
FIRST caller's status is recorded (`SIGNAL_GROUP_EXIT` check). Subsequent
callers observe the flag and use the existing code. This is how the
"first exit code wins" rule is implemented.

REQ-5: A fatal signal that triggers `do_group_exit` (e.g. SIGSEGV
delivered with no handler) ALSO sets `SIGNAL_GROUP_EXIT` and uses the
signal-encoded exit status. If a thread later calls `exit_group()`, its
status is ignored — the signal-induced exit wins. The reverse is also
true: `exit_group()` followed by a kill-the-process signal — the
already-set `group_exit_code` wins.

REQ-6: Parent notification: a single `SIGCHLD` is queued to the parent
when the LAST thread of the group exits (the "zombie group leader" path
in `exit_notify`). Reapable via `wait*(2)`.

REQ-7: `wait*(2)` returns:
- `WIFEXITED(s) == 1` (group exit via `exit_group`).
- `WEXITSTATUS(s) == status & 0xff`.

REQ-8: If the calling task is the thread-group leader AND the only
thread: `exit_group(s)` semantically equals `exit(s)` plus the group-exit
flag set, but observable behavior to the parent is the same.

REQ-9: Pending signals on each thread are flushed during their
`do_exit`. The SIGKILL-style wakeup is private (not delivered to
userspace) — it merely unblocks any pending wait/sleep so the thread can
run `do_exit` promptly.

REQ-10: Threads in `TASK_UNINTERRUPTIBLE` state are NOT awoken by the
exit_group fast-path — they finish their uninterruptible operation
first, then check `TIF_SIGPENDING` and run `do_group_exit`. This can
delay exit-completion arbitrarily under pathological I/O hangs.

REQ-11: Threads in `TASK_INTERRUPTIBLE` / `TASK_KILLABLE` state are
awoken immediately. All threads on `wait_event_killable` come out with
`-ERESTARTNOINTR` and then take the exit path.

REQ-12: Threads currently executing in user space are preempted at the
next entry into the kernel (timer tick, syscall, page fault); the
preempt path checks `TIF_SIGPENDING` and routes to `get_signal` →
`do_group_exit`.

REQ-13: The group-exit-leader (the original caller of `exit_group`) is
NOT necessarily the thread that gets reaped last. The "thread group
leader" (the task with `pid == tgid`) is the one that becomes the zombie
visible to the parent; if the group leader exited earlier via `exit(2)`,
it's already a "zombie leader" waiting for the rest of the group.

REQ-14: After `SIGNAL_GROUP_EXIT` is set, no new threads may be created
in the group: `clone(CLONE_THREAD, ...)` from a thread that's about to
exit returns `-EAGAIN` (because `copy_signal` rejects shared sighand
with SIGNAL_GROUP_EXIT set).

REQ-15: `prctl(PR_SET_SUBREAPER)` does NOT affect `exit_group` semantics
for the calling process; subreaper rules apply to ORPHANED children of
the exiting process.

REQ-16: ptrace: each thread's `PTRACE_EVENT_EXIT` is delivered to its
tracer (if any) during its `do_exit`. The group-exit code is visible to
the tracer.

REQ-17: cgroup membership: each thread is removed from its cgroup as it
exits. The cgroup's `cgroup.procs` and `cgroup.threads` files reflect
the group dying.

REQ-18: Audit: a single `AUDIT_SYSCALL` record for the `exit_group`
call, plus per-thread `AUDIT_SYSCALL` records from their own `do_exit`
paths.

REQ-19: All `exit(2)` REQ-3 to REQ-16 (per-thread `do_exit` side effects)
apply to EACH thread as it exits.

REQ-20: Status preservation: `signal->group_exit_code` is read by
parent's `wait*` and is NOT overwritten by any subsequent thread's exit.

## Acceptance Criteria

- [ ] AC-1: Syscall number is 231 on x86_64; 94 on generic.
- [ ] AC-2: `exit_group(0)` from a single-thread process: process exits with `WEXITSTATUS = 0`.
- [ ] AC-3: `exit_group(42)` from a 4-thread process: all 4 threads terminate; parent's `wait*` returns `WEXITSTATUS = 42`.
- [ ] AC-4: Two threads call `exit_group(7)` and `exit_group(13)` racing: parent observes `WEXITSTATUS = 7` (first-call wins).
- [ ] AC-5: Thread calls `exit_group(99)` while another thread is in `uninterruptible read()`: process eventually exits with 99 once the I/O completes.
- [ ] AC-6: SIGSEGV delivered to one thread with no handler: ALL threads die; `WIFSIGNALED = 1`, `WTERMSIG = SIGSEGV`.
- [ ] AC-7: After SIGNAL_GROUP_EXIT set, `clone(CLONE_THREAD)` in another thread → `-EAGAIN`.
- [ ] AC-8: ptrace tracer of each thread receives `PTRACE_EVENT_EXIT` with `group_exit_code`.
- [ ] AC-9: cgroup `cgroup.procs` shows the process gone after exit_group.
- [ ] AC-10: Parent receives EXACTLY ONE `SIGCHLD`.
- [ ] AC-11: `libc::_exit(15)` → `exit_group(15)` syscall observed (strace).
- [ ] AC-12: Auditd records the exit_group syscall + per-thread exit records.
- [ ] AC-13: After `exit_group`, no thread of the group is observable via `/proc/<tgid>`.
- [ ] AC-14: Session leader exit_group: foreground process group of the session receives `SIGHUP` + `SIGCONT`.
- [ ] AC-15: `exit_group` does not return.

## Architecture

```rust
#[syscall(nr = 231, abi = "sysv")]
pub fn sys_exit_group(error_code: i32) -> ! {
    Exit::do_group_exit((error_code as u32 & 0xff) << 8)
}
```

`Exit::do_group_exit(exit_code) -> !`:
1. let task    = Task::current();
2. let sig     = task.signal;
3. let sighand = task.sighand;
4. /* Take siglock for thread-group state mutations */
5. let _g = SpinLockGuard::lock(&sighand.siglock);
6. /* Already exiting? */
7. if sig.flags & SIGNAL_GROUP_EXIT != 0 {
8.   /* Use existing code; race-loser */
9.   drop(_g);
10.  Exit::do_exit(sig.group_exit_code);
11. }
12. /* First to call exit_group */
13. sig.group_exit_code = exit_code;
14. sig.flags |= SIGNAL_GROUP_EXIT;
15. /* SIGKILL-wake every other thread */
16. for t in sig.thread_head.iter() {
17.   if t != task {
18.     sigaddset(&t.pending.signal, SIGKILL);
19.     Signal::wake_up_state(t, TASK_INTERRUPTIBLE | TASK_KILLABLE);
20.   }
21. }
22. drop(_g);
23. /* Calling task exits */
24. Exit::do_exit(exit_code);
25. unreachable!();

`Signal::get_signal(...)` (relevant path for other threads):
1. If `task.signal.flags & SIGNAL_GROUP_EXIT != 0`:
2.   /* This thread is being reaped as part of group exit */
3.   Exit::do_exit(task.signal.group_exit_code);
4.   unreachable!();

`Signal::wake_up_state(t, mask)`:
1. spin_lock_irqsave(t.pi_lock).
2. set_tsk_thread_flag(t, TIF_SIGPENDING).
3. if t.state & mask: ttwu_queue(t).
4. spin_unlock_irqrestore.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `group_exit_idempotent` | INVARIANT | Second call with `SIGNAL_GROUP_EXIT` set does not overwrite group_exit_code. |
| `all_threads_signalled` | INVARIANT | Post-call: every thread in `signal->thread_head` either is_current or has TIF_SIGPENDING set. |
| `siglock_held_during_mutation` | INVARIANT | group_exit_code + flags mutation under sighand->siglock. |
| `exit_group_does_not_return` | INVARIANT | `sys_exit_group` is `!`. |

### Layer 2: TLA+

`kernel/exit-group.tla`:
- States: ENTER → TAKE_LOCK → (SET_CODE | OBSERVE_RACE) → WAKE_THREADS → RELEASE_LOCK → DO_EXIT.
- Concurrent agents: thread1 calling exit_group(7), thread2 calling exit_group(13), thread3 sleeping.
- Properties:
  - `safety_first_caller_wins` — final group_exit_code is the first SET_CODE write.
  - `safety_all_threads_reach_do_exit` — every thread eventually executes do_exit.
  - `safety_single_sigchld` — parent receives at most one SIGCHLD for the group.
  - `liveness_eventual_reap` — after exit_group, the tgid eventually becomes a zombie.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_group_exit` post: `signal.flags & SIGNAL_GROUP_EXIT != 0` | `Exit::do_group_exit` |
| `do_group_exit` post: ∀t ∈ group: t.exit_state ⟶ ZOMBIE/DEAD (eventually) | TLA-side |
| `get_signal` SIGNAL_GROUP_EXIT branch: calls do_exit(group_exit_code) | `Signal::get_signal` |

### Layer 4: Verus / Creusot functional

POSIX `_exit(2)` (process-level) semantic equivalence. LTP `exit_group01..exit_group03` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`exit_group(2)` reinforcement:

- **Per-`SIGNAL_GROUP_EXIT` idempotent** — defense against per-exit-code-overwrite race.
- **Per-siglock-held mutation** — defense against per-thread-list torn-read.
- **Per-thread SIGKILL-style wake** — defense against per-thread-stranded-in-sleep.
- **Per-no-new-threads after exit-flag** — defense against per-clone-during-exit zombie inflation.
- **Per-cgroup membership cleared per thread** — defense against per-cgroup-stale entry.
- **Per-single SIGCHLD** — defense against per-parent-flooding.
- **Per-uninterruptible-wait honored** — defense against per-data-loss (in-flight I/O completes before exit).
- **Per-tracer `PTRACE_EVENT_EXIT` per thread** — defense against per-tracer-miss.
- **Per-audit record per syscall + per do_exit** — defense against per-mass-exit elision.

## Grsecurity / PaX surface

- **PAX_RANDKSTACK at syscall entry** — `sys_exit_group` randomizes
  kernel stack offset; the per-thread `do_exit` paths each get a fresh
  randomization too (each is a separate syscall-entry equivalent on
  preemption).
- **PaX UDEREF** — no user buffers on this syscall; `status` is a
  register-passed integer.
- **GRKERNSEC_HARDEN_PTRACE** — `PTRACE_EVENT_EXIT` per thread honors
  yama+grsec; cross-credential tracers are denied the final exit_code
  view.
- **GRKERNSEC_PROC restrictions** — `/proc/<tgid>` and `/proc/<tgid>/task/`
  entries vanish atomically when the group reaches `EXIT_ZOMBIE`; per-uid
  hide-filter prevents readback during the zombie window from non-owner.
- **GRKERNSEC_CHROOT** — exit_group inside chroot: parent (outside chroot)
  receives `SIGCHLD` from kernel-credentialed source.
- **GRKERNSEC_BRUTE** — high-rate `exit_group(SIGSEGV-style)` patterns
  from a UID (e.g., a forking SUID program ROP-probing) increment the
  per-uid forkbomb / brute-force counter; sustained pattern triggers
  per-uid exec/fork lockout for 30 seconds.
- **GRKERNSEC_SIGNALS spoof prevention** — `SIGCHLD` to parent carries
  `si_pid`, `si_uid`, and `si_code = CLD_EXITED` from kernel-credentialed
  source; un-spoofable.
- **Per-grsec `chroot_findtask`** — once group is zombie, parent outside
  chroot cannot enumerate the dying threads' state via `/proc`.
- **Per-grsec `task_size_max`** — final RSS / VM accounting recorded per
  uid; sustained pattern of "grow-then-exit" detected.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `exit(2)` syscall (separate Tier-5 doc).
- `do_exit` per-thread internals (Tier-3 `kernel/exit.c`).
- `wait4` / `waitid` reaping (separate Tier-5 docs).
- Robust-futex walker (`kernel/futex/core.c` — Tier-3).
- Signal delivery to interrupted threads (`get_signal` — Tier-3).
- Implementation code.
