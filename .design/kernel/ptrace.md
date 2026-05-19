# Tier-3: kernel/ptrace.c — Process tracing (ptrace syscall)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/ptrace.c (~1562 lines)
  - include/linux/ptrace.h (PT_*, PTRACE_MODE_*, PTRACE_O_*)
  - include/uapi/linux/ptrace.h (PTRACE_TRACEME/ATTACH/SEIZE/DETACH/INTERRUPT/KILL/PEEK*/POKE*/CONT/SINGLESTEP/SYSCALL/SYSEMU/GETREGS*/SETREGS*/GETSIGINFO/SETSIGINFO/PEEKSIGINFO/GETSIGMASK/SETSIGMASK/GETEVENTMSG/GETREGSET/SETREGSET/SETOPTIONS/SECCOMP_GET_FILTER/GET_SYSCALL_INFO/SET_SYSCALL_INFO)
  - include/uapi/linux/regset.h (struct iovec, struct user_regset)
  - arch/*/include/asm/ptrace.h (arch_ptrace, pt_regs)
-->

## Summary

**ptrace** is the kernel's universal debugger / tracer interface. One task ("tracer") attaches to another ("tracee"); the tracee's `task->parent` is rewritten to the tracer and a per-tracee `ptrace` flag mask (`PT_PTRACED`, `PT_SEIZED`, `PT_PTRACE_CAP`, `PT_EXITKILL`, `PT_SUSPEND_SECCOMP`, and `PTRACE_O_*` options left-shifted into the upper bits) records the relationship. Per-tracee state machine: the tracee transitions between `TASK_RUNNING`, `TASK_STOPPED`, and `TASK_TRACED`, with `JOBCTL_*` flags (`STOP_PENDING`, `TRAP_STOP`, `TRAP_NOTIFY`, `LISTENING`, `TRAPPING`, `PTRACE_FROZEN`) gating transitions. Per-syscall the tracee may trap on entry / exit (`SYSCALL_WORK_SYSCALL_TRACE` / `_EMU`) and at events (`PTRACE_EVENT_FORK`, `_VFORK`, `_CLONE`, `_EXEC`, `_EXIT`, `_SECCOMP`, `_STOP`). The tracer issues `ptrace(request, pid, addr, data)` syscalls dispatched through `ptrace_request` / arch-specific `arch_ptrace`: introspection (`PEEKTEXT`/`PEEKDATA`/`PEEKUSER`/`GETREGS`/`GETFPREGS`/`GETREGSET`/`GETSIGINFO`/`PEEKSIGINFO`/`GETEVENTMSG`/`GETSIGMASK`/`GET_SYSCALL_INFO`/`SECCOMP_GET_FILTER`), mutation (`POKETEXT`/`POKEDATA`/`POKEUSER`/`SETREGS`/`SETFPREGS`/`SETREGSET`/`SETSIGINFO`/`SETSIGMASK`/`SETOPTIONS`/`SET_SYSCALL_INFO`), execution control (`CONT`/`SYSCALL`/`SYSEMU`/`SYSEMU_SINGLESTEP`/`SINGLESTEP`/`SINGLEBLOCK`/`INTERRUPT`/`LISTEN`/`KILL`/`DETACH`). `PTRACE_SEIZE` is the modern attach (no auto-SIGSTOP, supports `PTRACE_INTERRUPT`/`LISTEN`). Access-control: `ptrace_may_access` (REALCREDS for `ATTACH`, FSCREDS for `/proc` inspection) combines UID/GID equality, `CAP_SYS_PTRACE` in tracee's user-ns, dumpable (`SUID_DUMP_USER`), LSM `security_ptrace_access_check`. Critical for: gdb / strace / lldb / debuggers / fuzzers / CRIU / sandbox observers.

This Tier-3 covers `kernel/ptrace.c` (~1562 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `ptrace_access_vm()` | per-PEEK/POKE remote VM IO | `Ptrace::access_vm` |
| `__ptrace_link()` / `ptrace_link()` | per-attach reparent + cred-pin | `Ptrace::link` |
| `__ptrace_unlink()` | per-detach reparent + clear jobctl | `Ptrace::unlink` |
| `looks_like_a_spurious_pid()` | per-EVENT_EXEC tid-change quirk | `Ptrace::looks_spurious_pid` |
| `ptrace_freeze_traced()` / `ptrace_unfreeze_traced()` | per-request uninterruptible-tracee window | `Ptrace::freeze_traced` / `unfreeze_traced` |
| `ptrace_check_attach()` | per-request precondition gate | `Ptrace::check_attach` |
| `ptrace_has_cap()` | per-CAP_SYS_PTRACE check | `Ptrace::has_cap` |
| `__ptrace_may_access()` | per-introspection access policy | `Ptrace::may_access_inner` |
| `ptrace_may_access()` | per-introspection wrapper | `Ptrace::may_access` |
| `check_ptrace_options()` | per-SETOPTIONS / per-SEIZE validation | `Ptrace::check_options` |
| `ptrace_set_stopped()` | per-attach forced-stop transition | `Ptrace::set_stopped` |
| `ptrace_attach()` | per-ATTACH / per-SEIZE entry | `Ptrace::attach` |
| `ptrace_traceme()` | per-TRACEME entry | `Ptrace::traceme` |
| `__ptrace_detach()` / `ptrace_detach()` | per-DETACH and on tracer exit | `Ptrace::detach` |
| `exit_ptrace()` | per-tracer-exit detach-all + EXITKILL | `Ptrace::exit_ptrace` |
| `ptrace_readdata()` / `ptrace_writedata()` | per-large-IO block-copy | `Ptrace::readdata` / `writedata` |
| `ptrace_setoptions()` | per-SETOPTIONS apply | `Ptrace::set_options` |
| `ptrace_getsiginfo()` / `ptrace_setsiginfo()` / `ptrace_peek_siginfo()` | per-signal introspection / mutation | `Ptrace::*siginfo` |
| `ptrace_resume()` | per-CONT/SYSCALL/SYSEMU/SINGLESTEP/SINGLEBLOCK | `Ptrace::resume` |
| `find_regset()` / `ptrace_regset()` | per-GETREGSET / SETREGSET | `Ptrace::regset` |
| `ptrace_get_syscall_info()` / `ptrace_set_syscall_info()` / `_entry()` / `_exit()` / `_seccomp()` / `_op()` | per-GET/SET_SYSCALL_INFO | `Ptrace::syscall_info_*` |
| `ptrace_get_rseq_configuration()` | per-CRIU rseq snapshot | `Ptrace::rseq_config` |
| `ptrace_request()` | per-request dispatch | `Ptrace::request` |
| `SYSCALL_DEFINE4(ptrace,...)` | per-syscall entry | `Ptrace::sys_ptrace` |
| `generic_ptrace_peekdata()` / `generic_ptrace_pokedata()` | per-PEEK/POKE word IO | `Ptrace::peekdata` / `pokedata` |
| `compat_ptrace_request()` | per-32-on-64 dispatch | `Ptrace::compat_request` |
| `ptracer_capable()` | per-MM access cred check | `Ptrace::ptracer_capable` |

## Compatibility contract

REQ-1: Per-tracee ptrace state (`task_struct` fields):
- `task->ptrace`: bitmask of `PT_PTRACED | PT_SEIZED | PT_PTRACE_CAP | PT_EXITKILL | PT_SUSPEND_SECCOMP | (PTRACE_O_* << PT_OPT_FLAG_SHIFT)`.
- `task->parent`: per-attach reparented to tracer; per-detach restored to `real_parent`.
- `task->ptracer_cred`: tracer's `struct cred` snapshot pinned at attach (used by `ptracer_capable`).
- `task->ptrace_entry`: list_head linked into `tracer->ptraced`.
- `task->jobctl`: includes `JOBCTL_STOP_PENDING`, `JOBCTL_TRAP_STOP`, `JOBCTL_TRAP_NOTIFY`, `JOBCTL_LISTENING`, `JOBCTL_TRAPPING`, `JOBCTL_PTRACE_FROZEN`, `JOBCTL_STOPPED`, `JOBCTL_TRACED`.
- `task->last_siginfo`: pointer to the `kernel_siginfo_t` currently being delivered to tracer (peek/set targets).
- `task->ptrace_message`: per-event payload (e.g. event nr for `PTRACE_EVENT_*`).
- `task->exit_code`: signal number used to resume (set by `ptrace_resume` / `ptrace_detach`).

REQ-2: Per-PTRACE_MODE_* in `__ptrace_may_access`:
- `PTRACE_MODE_READ` ↔ `PTRACE_MODE_ATTACH` (privilege level).
- `PTRACE_MODE_NOAUDIT` (suppresses audit logging).
- `PTRACE_MODE_FSCREDS` (use fsuid/fsgid; for `/proc` introspection).
- `PTRACE_MODE_REALCREDS` (use uid/gid; for syscall-driven inspection).
- Exactly one of FSCREDS / REALCREDS must be set (else WARN + -EPERM).
- Same thread group → 0 (always allowed).
- Otherwise rcu-locked: caller's (fs|uid, fs|gid) must equal each of tracee's (euid, suid, uid) and (egid, sgid, gid); OR `ptrace_has_cap(tcred->user_ns, mode)` (CAP_SYS_PTRACE in tracee's user ns).
- After cred-equal/cap-grant: smp_rmb; if `get_dumpable(mm) != SUID_DUMP_USER` and tracee's mm user-ns does not grant cap → -EPERM.
- Finally `security_ptrace_access_check(task, mode)`.

REQ-3: ptrace_may_access(task, mode) wrapper:
- task_lock(task); err = `__ptrace_may_access(task, mode)`; task_unlock(task); return !err.

REQ-4: check_ptrace_options(data):
- `data & ~PTRACE_O_MASK` → -EINVAL.
- `data & PTRACE_O_SUSPEND_SECCOMP` requires `CONFIG_CHECKPOINT_RESTORE && CONFIG_SECCOMP` (else -EINVAL), `capable(CAP_SYS_ADMIN)` (else -EPERM), `seccomp_mode(current) == DISABLED && !(current->ptrace & PT_SUSPEND_SECCOMP)` (else -EPERM).

REQ-5: PTRACE_O_* (recognised options encoded in upper bits of `task->ptrace`):
- `PTRACE_O_TRACESYSGOOD` — sets bit 7 of `si_code` (0x80) on syscall trap so tracer can distinguish from other SIGTRAPs.
- `PTRACE_O_TRACEFORK`, `_TRACEVFORK`, `_TRACECLONE`, `_TRACEEXEC`, `_TRACEEXIT`, `_TRACEVFORKDONE` — stop at corresponding `PTRACE_EVENT_*`.
- `PTRACE_O_TRACESECCOMP` — stop at seccomp `RET_TRACE` action.
- `PTRACE_O_EXITKILL` — `SIGKILL` sent to tracee on tracer exit.
- `PTRACE_O_SUSPEND_SECCOMP` — disable seccomp checks while traced (CHECKPOINT_RESTORE).

REQ-6: ptrace_attach(task, request, addr, flags):
- seize = (request == `PTRACE_SEIZE`).
- if seize: addr must be 0 (-EIO); flags ⊆ PTRACE_O_MASK (-EIO); `check_ptrace_options(flags)`; `task->ptrace = PT_PTRACED | PT_SEIZED | (flags << PT_OPT_FLAG_SHIFT)`.
- else: flags = `PT_PTRACED`.
- `audit_ptrace(task)`.
- Reject `task->flags & PF_KTHREAD` (-EPERM).
- Reject `same_thread_group(task, current)` (-EPERM).
- Under `task->signal->cred_guard_mutex` (interruptible; -ERESTARTNOINTR on signal):
  - task_lock(task); `__ptrace_may_access(task, PTRACE_MODE_ATTACH_REALCREDS)`; task_unlock.
  - write_lock_irq(tasklist_lock):
    - reject `task->exit_state != 0` (-EPERM).
    - reject `task->ptrace` already set (-EPERM).
    - `task->ptrace = flags`; `ptrace_link(task, current)` (links into `current->ptraced`, pins `ptracer_cred = current_cred()`); `ptrace_set_stopped(task, seize)`.
- `wait_on_bit(&task->jobctl, JOBCTL_TRAPPING_BIT, TASK_KILLABLE)` (wait for tracee to reach TRACED).
- `proc_ptrace_connector(task, PTRACE_ATTACH)`.

REQ-7: ptrace_set_stopped(task, seize):
- guard(spinlock)(task.sighand.siglock).
- if !seize: `send_signal_locked(SIGSTOP, SEND_SIG_PRIV, task, PIDTYPE_PID)` (force-stop on classic ATTACH).
- if task is stopped and `task_set_jobctl_pending(JOBCTL_TRAP_STOP | JOBCTL_TRAPPING)`: clear `JOBCTL_STOPPED`; `signal_wake_up_state(task, __TASK_STOPPED)` (drives STOPPED → RUNNING → TRACED transition).

REQ-8: ptrace_traceme():
- write_lock_irq(tasklist_lock).
- If `!current->ptrace`: `security_ptrace_traceme(current->parent)`; if 0 and `!(current->real_parent->flags & PF_EXITING)`: `current->ptrace = PT_PTRACED`; `ptrace_link(current, current->real_parent)`.
- write_unlock_irq.

REQ-9: ptrace_detach(child, data):
- if `!valid_signal(data)` → -EIO.
- `ptrace_disable(child)` (arch-specific hardware disable).
- write_lock_irq(tasklist_lock).
- `child->exit_code = data`; `__ptrace_detach(current, child)`:
  - `__ptrace_unlink(child)` — clears `SYSCALL_TRACE`/`SYSCALL_EMU` work, restores `child->parent = child->real_parent`, list_del `ptrace_entry`, drops `ptracer_cred`, clears `child->ptrace = 0`, clears `JOBCTL_TRAP_MASK` and `JOBCTL_TRAPPING`, re-asserts `JOBCTL_STOP_PENDING` if group stopped, `ptrace_signal_wake_up(child, true)`.
  - If `child->exit_state == EXIT_ZOMBIE`: redirect notification (`do_notify_parent` or autoreap) and possibly mark `EXIT_DEAD`.

REQ-10: exit_ptrace(tracer, dead):
- For each `p` in `tracer->ptraced`:
  - if `p->ptrace & PT_EXITKILL` → `send_sig_info(SIGKILL, SEND_SIG_PRIV, p)`.
  - if `__ptrace_detach(tracer, p)` returns true → add to `dead` list (caller calls `release_task`).

REQ-11: ptrace_check_attach(child, ignore_state):
- read_lock(tasklist_lock).
- Require `child->ptrace` set and `child->parent == current`.
- If `ignore_state` (PTRACE_KILL / PTRACE_INTERRUPT) → 0; else `ptrace_freeze_traced(child)` must return true (sets `JOBCTL_PTRACE_FROZEN` under siglock iff task is TRACED, not spurious-pid, no fatal signal).
- read_unlock.
- If non-ignore and 0: WARN if `!wait_task_inactive(child, __TASK_TRACED|TASK_FROZEN)` → -ESRCH.

REQ-12: ptrace_access_vm(tsk, addr, buf, len, gup_flags):
- mm = `get_task_mm(tsk)`; if !mm → 0.
- Require `tsk->ptrace`, `current == tsk->parent`, and (dumpable == SUID_DUMP_USER or `ptracer_capable(tsk, mm->user_ns)`); else mmput + 0.
- `access_remote_vm(mm, addr, buf, len, gup_flags)`.

REQ-13: ptrace_readdata / ptrace_writedata:
- Block-copy loop with 128-byte kernel buffer through `ptrace_access_vm` + `copy_{to,from}_user`. Used by arch `PTRACE_PEEKDATA` / `POKEDATA` byte-stream variants.

REQ-14: generic_ptrace_peekdata(tsk, addr, data):
- tmp = 0; `ptrace_access_vm(tsk, addr, &tmp, sizeof(tmp), FOLL_FORCE)`; require full `sizeof(unsigned long)` copied (-EIO else); `put_user(tmp, (unsigned long __user *)data)`.

REQ-15: generic_ptrace_pokedata(tsk, addr, data):
- `ptrace_access_vm(tsk, addr, &data, sizeof(data), FOLL_FORCE | FOLL_WRITE)`; require full word; 0 / -EIO.

REQ-16: ptrace_setoptions(child, data):
- `check_ptrace_options(data)`.
- `child->ptrace = (child->ptrace & ~(PTRACE_O_MASK << PT_OPT_FLAG_SHIFT)) | (data << PT_OPT_FLAG_SHIFT)` (atomic via single assignment of computed flags).

REQ-17: ptrace_getsiginfo / setsiginfo:
- `lock_task_sighand(child, &flags)`; require `child->last_siginfo != NULL` (else -EINVAL / -ESRCH); copy via `copy_siginfo`.

REQ-18: ptrace_peek_siginfo(child, addr, data):
- copy `struct ptrace_peeksiginfo_args { off, flags, nr }` from user.
- Reject `flags & ~PTRACE_PEEKSIGINFO_SHARED`; reject `nr < 0`; `off > ULONG_MAX` → 0.
- pending = `arg.flags & SHARED ? &child->signal->shared_pending : &child->pending`.
- Iterate at most `nr` entries, copy via `copy_siginfo_to_user[32]`.

REQ-19: ptrace_resume(child, request, data):
- Reject `!valid_signal(data)` → -EIO.
- If `PTRACE_SYSCALL`: `set_task_syscall_work(child, SYSCALL_TRACE)`; else clear.
- If `PTRACE_SYSEMU` / `SYSEMU_SINGLESTEP`: `set_task_syscall_work(child, SYSCALL_EMU)`; else clear (per CONFIG_GENERIC_ENTRY / TIF_SYSCALL_EMU).
- If `PTRACE_SINGLEBLOCK`: require `arch_has_block_step()`; `user_enable_block_step(child)`.
- elif `SINGLESTEP` / `SYSEMU_SINGLESTEP`: require `arch_has_single_step()`; `user_enable_single_step(child)`.
- else: `user_disable_single_step(child)`.
- spin_lock_irq(child.sighand.siglock); `child->exit_code = data`; `child->jobctl &= ~JOBCTL_TRACED`; `wake_up_state(child, __TASK_TRACED)`; spin_unlock_irq.

REQ-20: ptrace_regset(task, req, type, kiov):
- view = `task_user_regset_view(task)`; regset = `find_regset(view, type)`.
- Require regset and `kiov->iov_len % regset->size == 0`.
- Clamp `kiov->iov_len` to `regset->n * regset->size`.
- req == `PTRACE_GETREGSET` → `copy_regset_to_user`; else `copy_regset_from_user`.

REQ-21: ptrace_get_syscall_info / set_syscall_info:
- `struct ptrace_syscall_info` carries `op`, `arch`, `instruction_pointer`, `stack_pointer`, and a union of `entry` / `exit` / `seccomp`.
- `_op(child)` derives op from `last_siginfo->si_code`:
  - `SIGTRAP | 0x80` + `ptrace_message == PTRACE_EVENTMSG_SYSCALL_ENTRY` → `INFO_ENTRY`.
  - `SIGTRAP | 0x80` + `_SYSCALL_EXIT` → `INFO_EXIT`.
  - `SIGTRAP | (PTRACE_EVENT_SECCOMP << 8)` → `INFO_SECCOMP`.
  - else `INFO_NONE`.
- `_entry`: nr + args[0..5] from `syscall_get_nr` / `syscall_get_arguments`.
- `_exit`: rval = `syscall_get_error(child, regs)`; is_error = !!rval; if !is_error rval = `syscall_get_return_value(child, regs)`.
- `_seccomp`: same as entry plus `ret_data = child->ptrace_message`.
- Copies `min(actual_size, user_size)` to userspace; returns actual_size.

REQ-22: ptrace_get_rseq_configuration(task, size, data):
- Snapshot `{ rseq_abi_pointer = (u64)task->rseq.usrptr, rseq_abi_size = task->rseq.len, signature = task->rseq.sig, flags = 0 }` and copy `min(size, sizeof(conf))`.

REQ-23: ptrace_request(child, request, addr, data) dispatch:
- `PTRACE_PEEKTEXT` / `PTRACE_PEEKDATA` → `generic_ptrace_peekdata(child, addr, data)`.
- `PTRACE_POKETEXT` / `PTRACE_POKEDATA` → `generic_ptrace_pokedata(child, addr, data)`.
- `PTRACE_SETOPTIONS` (and `PTRACE_OLDSETOPTIONS`) → `ptrace_setoptions`.
- `PTRACE_GETEVENTMSG` → `put_user(child->ptrace_message, datalp)`.
- `PTRACE_PEEKSIGINFO` → `ptrace_peek_siginfo`.
- `PTRACE_GETSIGINFO` → `ptrace_getsiginfo` + `copy_siginfo_to_user`.
- `PTRACE_SETSIGINFO` → `copy_siginfo_from_user` + `ptrace_setsiginfo`.
- `PTRACE_GETSIGMASK` / `PTRACE_SETSIGMASK` → require `addr == sizeof(sigset_t)`; getsigmask reads `child->saved_sigmask` if `test_tsk_restore_sigmask` else `child->blocked`; setsigmask removes SIGKILL/SIGSTOP from new mask, writes under siglock, clears restore-sigmask flag.
- `PTRACE_INTERRUPT` — SEIZE-only; under siglock if `task_set_jobctl_pending(JOBCTL_TRAP_STOP)`: `ptrace_signal_wake_up(child, jobctl & JOBCTL_LISTENING)`.
- `PTRACE_LISTEN` — SEIZE-only; under siglock if `last_siginfo` indicates `PTRACE_EVENT_STOP`: set `JOBCTL_LISTENING`; if `JOBCTL_TRAP_NOTIFY` already set, `ptrace_signal_wake_up`.
- `PTRACE_DETACH` → `ptrace_detach`.
- `PTRACE_SINGLESTEP` / `SINGLEBLOCK` / `SYSEMU` / `SYSEMU_SINGLESTEP` / `SYSCALL` / `CONT` → `ptrace_resume`.
- `PTRACE_KILL` → `send_sig_info(SIGKILL, SEND_SIG_NOINFO, child)` (legacy; equivalent to `kill(pid, SIGKILL)`).
- `PTRACE_GETREGSET` / `SETREGSET` → copy user `struct iovec`, `ptrace_regset`, write back updated `iov_len`.
- `PTRACE_GET_SYSCALL_INFO` / `SET_SYSCALL_INFO` → `ptrace_get_syscall_info` / `ptrace_set_syscall_info`.
- `PTRACE_SECCOMP_GET_FILTER` → `seccomp_get_filter(child, addr, datavp)`.
- `PTRACE_SECCOMP_GET_METADATA` → `seccomp_get_metadata(child, addr, datavp)`.
- `PTRACE_GET_RSEQ_CONFIGURATION` → `ptrace_get_rseq_configuration`.
- `PTRACE_GET_SYSCALL_USER_DISPATCH_CONFIG` / `SET_..._CONFIG` → `syscall_user_dispatch_{get,set}_config`.
- `PTRACE_GETFDPIC` (CONFIG_BINFMT_ELF_FDPIC) → returns `mm->context.exec_fdpic_loadmap` / `interp_fdpic_loadmap`.
- default → -EIO.

REQ-24: SYSCALL_DEFINE4(ptrace, request, pid, addr, data):
- If request == `PTRACE_TRACEME` → `ptrace_traceme()`.
- Else `child = find_get_task_by_vpid(pid)`; -ESRCH if none.
- If request ∈ {`PTRACE_ATTACH`, `PTRACE_SEIZE`} → `ptrace_attach(child, request, addr, data)`.
- Else `ptrace_check_attach(child, request ∈ {PTRACE_KILL, PTRACE_INTERRUPT})`; on success `arch_ptrace(child, request, addr, data)`; if ret or request != `PTRACE_DETACH` → `ptrace_unfreeze_traced(child)`.
- `put_task_struct(child)`.

REQ-25: arch_ptrace contract:
- Per-arch wrapper that handles `PTRACE_GETREGS` / `SETREGS` / `GETFPREGS` / `SETFPREGS` / `PEEKUSER` / `POKEUSER` / arch-specific requests, then falls through to `ptrace_request` for generic ones.

REQ-26: PTRACE_EVENT_* and trap reporting:
- Tracee uses `ptrace_event(event, message)` (defined in `linux/ptrace.h`) to set `last_siginfo` with `SIGTRAP | (event << 8)`, store `message` in `ptrace_message`, and `ptrace_notify` to enter `TASK_TRACED`.
- Events: `_FORK`, `_VFORK`, `_VFORK_DONE`, `_CLONE`, `_EXEC`, `_EXIT`, `_SECCOMP`, `_STOP` (SEIZE-only).

REQ-27: ptrace_message UAPI:
- For `_FORK/_VFORK/_CLONE`: child pid (vnr).
- For `_EXIT`: exit status (`task->exit_code`).
- For `_SECCOMP`: low 16 bits of the `SECCOMP_RET_TRACE | data` filter result.
- For `_EXEC`: leader thread pid (old).
- For `_STOP`: 0.
- For syscall-trace via `PTRACE_O_TRACESYSGOOD`: 0; `si_code = SIGTRAP | 0x80`.

REQ-28: PT_SUSPEND_SECCOMP semantics (CHECKPOINT_RESTORE):
- When set on tracee by tracer (via `PTRACE_O_SUSPEND_SECCOMP`), `__secure_computing` short-circuits returning 0 without consulting filters.
- Set requires `CAP_SYS_ADMIN`, tracee must currently be `SECCOMP_MODE_DISABLED`, and the bit must not already be set.

REQ-29: PT_EXITKILL semantics (`PTRACE_O_EXITKILL`):
- Per-tracer-exit `exit_ptrace` iterates `ptraced` list and sends SIGKILL to each tracee with `PT_EXITKILL` set before detaching.

REQ-30: PT_PTRACED vs PT_SEIZED distinction:
- `PTRACE_ATTACH` (classic) — sends SIGSTOP, tracee stops naturally on next signal handling; `PTRACE_INTERRUPT` / `LISTEN` rejected (-EIO).
- `PTRACE_SEIZE` — no auto-SIGSTOP; tracee continues running until `PTRACE_INTERRUPT`; supports `PTRACE_EVENT_STOP`; options are passed at attach.

## Acceptance Criteria

- [ ] AC-1: `ptrace(PTRACE_TRACEME)` from child sets `current->ptrace = PT_PTRACED` and reparents to `real_parent`.
- [ ] AC-2: `ptrace(PTRACE_ATTACH, pid, 0, 0)`: tracee `parent` becomes tracer, `task->ptrace` gains `PT_PTRACED`, SIGSTOP delivered, `JOBCTL_TRAP_STOP` set; subsequent wait returns the SIGSTOP report.
- [ ] AC-3: `ptrace(PTRACE_SEIZE, pid, 0, options)`: tracee `task->ptrace = PT_PTRACED | PT_SEIZED | (options << shift)`, no SIGSTOP delivered, tracee keeps running.
- [ ] AC-4: `ptrace(PTRACE_ATTACH, pid)` on same-thread-group → -EPERM; on PF_KTHREAD task → -EPERM; on already-traced task → -EPERM.
- [ ] AC-5: `__ptrace_may_access` denies unprivileged tracer on different-uid tracee unless tracer has CAP_SYS_PTRACE in tracee's user-ns.
- [ ] AC-6: `PTRACE_PEEKDATA` reads one machine word from tracee VM via `ptrace_access_vm` + `FOLL_FORCE`; `PTRACE_POKEDATA` writes one machine word via `FOLL_FORCE|FOLL_WRITE`.
- [ ] AC-7: `PTRACE_GETREGS` / `GETFPREGS` / `GETREGSET` return tracee register state; `SETREGS` / `SETFPREGS` / `SETREGSET` write it (round-tripped via `task_pt_regs`).
- [ ] AC-8: `PTRACE_GETSIGINFO` returns the siginfo currently being delivered; `PTRACE_SETSIGINFO` overwrites it; both require `child->last_siginfo != NULL`.
- [ ] AC-9: `PTRACE_SETOPTIONS` with `PTRACE_O_TRACESYSGOOD` causes subsequent syscall-trace stops to set `si_code = SIGTRAP | 0x80`.
- [ ] AC-10: `PTRACE_CONT` resumes tracee with given signal; `PTRACE_SYSCALL` arms `SYSCALL_WORK_SYSCALL_TRACE`; `PTRACE_SINGLESTEP` enables single-step via `user_enable_single_step` (or -EIO if arch lacks `arch_has_single_step`).
- [ ] AC-11: `PTRACE_DETACH` reparents tracee to `real_parent`, clears `task->ptrace`, and resumes the tracee with the signal in `data` (validated by `valid_signal`).
- [ ] AC-12: `PTRACE_KILL` delivers SIGKILL via `send_sig_info`; `PTRACE_INTERRUPT` on SEIZE'd tracee sets `JOBCTL_TRAP_STOP` and wakes the tracee.
- [ ] AC-13: Tracer exit: tracees with `PT_EXITKILL` receive SIGKILL; all are detached via `exit_ptrace`.
- [ ] AC-14: `PTRACE_O_SUSPEND_SECCOMP` requires `CAP_SYS_ADMIN`, `CONFIG_CHECKPOINT_RESTORE && CONFIG_SECCOMP`, and tracee `seccomp.mode == DISABLED`; once set, `__secure_computing` skips filter evaluation.
- [ ] AC-15: `PTRACE_GET_SYSCALL_INFO` returns `op = ENTRY|EXIT|SECCOMP|NONE` derived from `last_siginfo->si_code`, plus matching arch / IP / SP / args / rval.

## Architecture

```
struct PtraceFlags {                    // packed into task_struct.ptrace
  pt_ptraced: bool,                     // PT_PTRACED
  pt_seized: bool,                      // PT_SEIZED
  pt_ptrace_cap: bool,                  // PT_PTRACE_CAP (CAP_SYS_PTRACE at attach)
  pt_exitkill: bool,                    // PT_EXITKILL
  pt_suspend_seccomp: bool,             // PT_SUSPEND_SECCOMP
  options: u32,                         // PTRACE_O_* in PT_OPT_FLAG_SHIFT range
}

struct PtraceLink {                     // wired into task_struct
  parent: *TaskStruct,                  // overridden during attach
  real_parent: *TaskStruct,             // preserved for detach
  ptracer_cred: *Cred,                  // RCU-protected snapshot
  ptrace_entry: ListNode,               // in tracer.ptraced
  ptraced: ListHead,                    // list of tracees (when tracer)
  jobctl: u64,                          // JOBCTL_*
  last_siginfo: *KernelSiginfo,
  ptrace_message: u64,
  exit_code: i32,
}

struct PtraceSyscallInfo {              // UAPI
  op: u8,                               // ENTRY=1, EXIT=2, SECCOMP=3, NONE=0
  arch: u32,                            // AUDIT_ARCH_*
  instruction_pointer: u64,
  stack_pointer: u64,
  variant: union {
    entry: { nr: i64, args: [u64; 6] },
    exit: { rval: i64, is_error: u8 },
    seccomp: { nr: i64, args: [u64; 6], ret_data: u32 },
  },
}
```

`Ptrace::sys_ptrace(request, pid, addr, data) -> i64`:
1. if request == TRACEME → `traceme()`.
2. child = `find_get_task_by_vpid(pid)`; -ESRCH if None.
3. if request ∈ {ATTACH, SEIZE} → `attach(child, request, addr, data)`; goto put.
4. ignore_state = request ∈ {KILL, INTERRUPT}; `check_attach(child, ignore_state)`.
5. ret = `arch_ptrace(child, request, addr, data)` (arch wrapper that handles arch-specific then falls through to `request(child, request, addr, data)`).
6. if ret != 0 ∨ request != DETACH → `unfreeze_traced(child)`.
7. put_task_struct(child); return ret.

`Ptrace::attach(task, request, addr, flags) -> i32`:
1. seize = request == SEIZE.
2. if seize: addr != 0 → -EIO; flags & ~PTRACE_O_MASK → -EIO; `check_options(flags)`; flags = `PT_PTRACED | PT_SEIZED | (flags << PT_OPT_FLAG_SHIFT)`.
3. else: flags = PT_PTRACED.
4. audit_ptrace(task).
5. task.flags & PF_KTHREAD → -EPERM.
6. same_thread_group(task, current) → -EPERM.
7. scoped(cred_guard_mutex, interruptible → -ERESTARTNOINTR):
   a. scoped(task_lock(task)): `may_access_inner(task, PTRACE_MODE_ATTACH_REALCREDS)`.
   b. scoped(write_lock_irq(tasklist_lock)):
      - task.exit_state != 0 → -EPERM.
      - task.ptrace != 0 → -EPERM.
      - task.ptrace = flags.
      - `link(task, current)`: `task.ptrace_entry → current.ptraced`; `task.parent = current`; `task.ptracer_cred = get_cred(current_cred())`.
      - `set_stopped(task, seize)`.
8. `wait_on_bit(&task.jobctl, JOBCTL_TRAPPING_BIT, TASK_KILLABLE)`.
9. `proc_ptrace_connector(task, PTRACE_ATTACH)`.

`Ptrace::detach(child, data) -> i32`:
1. !valid_signal(data) → -EIO.
2. `arch_ptrace_disable(child)`.
3. write_lock_irq(tasklist_lock).
4. child.exit_code = data; `detach_inner(current, child)`:
   - `unlink(child)`: clear SYSCALL_TRACE/EMU work; child.parent = child.real_parent; list_del(ptrace_entry); put_cred(ptracer_cred); spin_lock(sighand.siglock); child.ptrace = 0; task_clear_jobctl_pending(JOBCTL_TRAP_MASK); task_clear_jobctl_trapping; if !PF_EXITING ∧ (signal.flags & STOP_STOPPED ∨ group_stop_count): jobctl |= JOBCTL_STOP_PENDING; if JOBCTL_STOP_PENDING ∨ task_is_traced(child): ptrace_signal_wake_up(child, true); spin_unlock.
   - if child.exit_state == EXIT_ZOMBIE: redirect parent notification or autoreap; possibly mark EXIT_DEAD.
5. write_unlock_irq; `proc_ptrace_connector(child, PTRACE_DETACH)`.

`Ptrace::may_access_inner(task, mode) -> i32`:
1. require exactly one of {FSCREDS, REALCREDS}.
2. same_thread_group(task, current) → 0.
3. rcu_read_lock(); (caller_uid, caller_gid) = mode-dependent selection.
4. tcred = `__task_cred(task)`.
5. UID-equal across {euid, suid, uid} ∧ GID-equal across {egid, sgid, gid} → ok.
6. else `has_cap(tcred.user_ns, mode)` → ok.
7. else rcu_read_unlock; return -EPERM.
8. ok: rcu_read_unlock; smp_rmb.
9. mm = task.mm; if mm ∧ `get_dumpable(mm) != SUID_DUMP_USER` ∧ !`has_cap(mm.user_ns, mode)` → -EPERM.
10. return `security_ptrace_access_check(task, mode)`.

`Ptrace::request(child, request, addr, data) -> i32`:
1. seized = child.ptrace & PT_SEIZED.
2. match request: dispatch per REQ-23.
3. Default → -EIO.

`Ptrace::resume(child, request, data) -> i32`:
1. !valid_signal(data) → -EIO.
2. arm/clear SYSCALL_TRACE / SYSCALL_EMU per request.
3. SINGLEBLOCK → require arch_has_block_step; user_enable_block_step.
4. SINGLESTEP / SYSEMU_SINGLESTEP → require arch_has_single_step; user_enable_single_step.
5. else user_disable_single_step.
6. spin_lock_irq(child.sighand.siglock); child.exit_code = data; child.jobctl &= ~JOBCTL_TRACED; `wake_up_state(child, __TASK_TRACED)`; spin_unlock_irq.

`Ptrace::peek_siginfo(child, addr, data) -> i32`:
1. copy_from_user(arg, addr).
2. arg.flags & ~PEEKSIGINFO_SHARED → -EINVAL; arg.nr < 0 → -EINVAL; arg.off > ULONG_MAX → 0.
3. pending = SHARED ? &child.signal.shared_pending : &child.pending.
4. for i in 0..arg.nr: spin_lock_irq(sighand.siglock); walk pending.list to entry off+i; spin_unlock_irq; copy_siginfo_to_user[32]; data += sizeof(siginfo_t); cond_resched.
5. return i (or first errno).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `attach_needs_REALCREDS` | INVARIANT | per-`attach`: `may_access(task, PTRACE_MODE_ATTACH_REALCREDS)` invoked before link. |
| `no_self_attach` | INVARIANT | per-`attach`: same_thread_group(task, current) ⇒ -EPERM. |
| `no_kthread_attach` | INVARIANT | per-`attach`: PF_KTHREAD ⇒ -EPERM. |
| `no_double_attach` | INVARIANT | per-`attach`: task.ptrace already set ⇒ -EPERM. |
| `unlink_clears_ptrace` | INVARIANT | per-`unlink`: child.ptrace == 0 ∧ child.parent == child.real_parent ∧ child.ptracer_cred == NULL on exit. |
| `freeze_under_siglock` | INVARIANT | per-`freeze_traced`: task.sighand.siglock held; sets JOBCTL_PTRACE_FROZEN only iff task_is_traced ∧ !looks_spurious ∧ !fatal_signal_pending. |
| `cap_check_uses_tracee_ns` | INVARIANT | per-`may_access_inner`: `has_cap(tcred.user_ns, mode)` (not init_user_ns). |
| `peekdata_full_word_or_eio` | INVARIANT | per-`generic_ptrace_peekdata`: copied != sizeof(unsigned long) ⇒ -EIO. |
| `resume_signal_valid` | INVARIANT | per-`resume`: !valid_signal(data) ⇒ -EIO before any state change. |
| `suspend_seccomp_gated` | INVARIANT | per-`check_options`: PTRACE_O_SUSPEND_SECCOMP requires CAP_SYS_ADMIN ∧ seccomp_mode == DISABLED ∧ !already-set. |

### Layer 2: TLA+

`kernel/ptrace.tla`:
- Per-attach + per-stop / per-trap + per-request + per-detach + per-tracer-exit.
- Properties:
  - `safety_unique_tracer` — at any time, a task has at most one tracer (`task.parent` reparenting is exclusive).
  - `safety_detach_restores_parent` — per-`detach`: child.parent == child.real_parent post-condition.
  - `safety_freeze_excludes_kill` — per-`freeze_traced`: fatal_signal_pending ⇒ no freeze.
  - `safety_check_attach_requires_parent` — per-request: `child.parent == current` enforced.
  - `safety_seize_vs_attach_disjoint` — per-`attach`: PT_PTRACED ⇒ exactly one of PT_SEIZED present, never both flipped after install.
  - `safety_exitkill_kills_before_detach` — per-tracer-exit: `PT_EXITKILL` ⇒ SIGKILL precedes `__ptrace_detach`.
  - `liveness_per_request_terminates` — per-`request`: every dispatch returns or transfers control to `arch_ptrace` within bounded steps.
  - `liveness_per_attach_completes` — per-`attach`: `wait_on_bit(JOBCTL_TRAPPING_BIT)` resolves within finite time (tracee reaches TRACED or dies).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ptrace::sys_ptrace` post: ret ∈ {0, ≥0, -ESRCH, -EPERM, -EIO, -EINVAL, -EFAULT, -ERESTARTNOINTR} | `Ptrace::sys_ptrace` |
| `Ptrace::attach` post: success ⇒ task.ptrace ≠ 0 ∧ task.parent == current ∧ task linked into current.ptraced | `Ptrace::attach` |
| `Ptrace::detach` post: child.ptrace == 0 ∧ child.parent == child.real_parent | `Ptrace::detach` |
| `Ptrace::may_access_inner` post: PERM ⇒ (cred-equal ∨ has_cap(tcred.user_ns, mode)) ∧ dumpable-check ∧ LSM-check | `Ptrace::may_access_inner` |
| `Ptrace::resume` post: child.exit_code == data ∧ ¬JOBCTL_TRACED | `Ptrace::resume` |
| `Ptrace::get_syscall_info` post: op derived solely from last_siginfo.si_code + ptrace_message | `Ptrace::get_syscall_info` |
| `Ptrace::regset` post: req == GETREGSET ⇒ copy_regset_to_user; req == SETREGSET ⇒ copy_regset_from_user | `Ptrace::regset` |

### Layer 4: Verus/Creusot functional

`Per-attach: REALCREDS may_access → tasklist_lock_irq → link + set_stopped → wait_on_bit JOBCTL_TRAPPING → connector notify` semantic equivalence: per-`man 2 ptrace`, `Documentation/admin-guide/cgroup-v2.rst` (for cgroup-aware ptrace), and `tools/testing/selftests/ptrace/*` reference behavior.

`Per-request dispatch (ptrace_request switch)` semantic equivalence: per-UAPI `<linux/ptrace.h>` and `man 2 ptrace`.

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

ptrace reinforcement:

- **Per-`PTRACE_MODE_FSCREDS` vs `_REALCREDS` enforced exclusivity** — defense against per-cred-confusion in /proc readers.
- **Per-cred snapshot at attach (`ptracer_cred`)** — defense against per-cred-uplift between attach and access.
- **Per-`get_dumpable` SUID_DUMP_USER gate** — defense against per-non-dumpable-mm leak via setuid execution.
- **Per-`security_ptrace_access_check` LSM hook** — defense against per-policy bypass.
- **Per-`PF_KTHREAD` attach refusal** — defense against per-kthread-tampering.
- **Per-same-thread-group attach refusal** — defense against per-self-trace deadlock and CVE-class privilege confusion.
- **Per-double-attach refusal under tasklist_lock** — defense against per-multiple-tracer race.
- **Per-`cred_guard_mutex` taken during attach** — defense against per-mid-exec attach setuid-bypass.
- **Per-`PT_SUSPEND_SECCOMP` CAP_SYS_ADMIN + DISABLED + once-only** — defense against per-seccomp-bypass via tracer.
- **Per-`PTRACE_KILL` retained but legacy; `tkill(SIGKILL)` preferred** — defense against per-historical-quirk reliance.
- **Per-`FOLL_FORCE` reserved to ptrace_access_vm and gated by parent identity** — defense against per-arbitrary-VM-write via `/proc/$pid/mem`.
- **Per-`valid_signal(data)` gate on resume / detach** — defense against per-bogus-signal injection.
- **Per-Yama `kernel.yama.ptrace_scope` (LSM-layer)** — defense against per-cross-process inspection in default-deny distros.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- arch_ptrace and `pt_regs` / FPU register layout (covered in arch Tier-3 docs)
- Yama LSM `ptrace_scope` policy (covered in `security/yama/` Tier-3 if expanded)
- `/proc/$pid/mem` and other procfs ptrace surfaces (covered in `proc.md` Tier-3)
- `coredump.c` and `binfmt_*` ptrace-aware paths (covered separately)
- seccomp filter introspection (covered in `seccomp.md` Tier-3)
- syscall_user_dispatch infrastructure (covered separately)
- Implementation code
