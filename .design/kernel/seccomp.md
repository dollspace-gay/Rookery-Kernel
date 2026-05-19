# Tier-3: kernel/seccomp.c — Secure computing (BPF syscall filter)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/seccomp.c (~2569 lines)
  - include/linux/seccomp.h
  - include/uapi/linux/seccomp.h (SECCOMP_MODE_*, SECCOMP_SET_MODE_*, SECCOMP_RET_*, SECCOMP_FILTER_FLAG_*, SECCOMP_IOCTL_NOTIF_*, SECCOMP_USER_NOTIF_*)
  - include/uapi/linux/filter.h (struct sock_filter, struct sock_fprog)
  - include/uapi/linux/prctl.h (PR_SET_SECCOMP, PR_GET_SECCOMP, PR_SET_NO_NEW_PRIVS)
-->

## Summary

**seccomp** ("secure computing") is the kernel's per-task syscall-filter sandbox. Each task carries `current->seccomp` (mode + filter chain). Per-syscall-entry the kernel invokes `__secure_computing` which, for `SECCOMP_MODE_STRICT`, allows only `read`/`write`/`exit`/`sigreturn`, and for `SECCOMP_MODE_FILTER` evaluates a stack of classic-BPF programs (`struct seccomp_filter`) against a `struct seccomp_data` (syscall nr, arch, args[0..5], instruction pointer). Per-filter the lowest BPF return value wins, and the bottom 16 bits (`SECCOMP_RET_DATA`) supply payload (errno number, ptrace tag, etc.) for the action. The action set is `SECCOMP_RET_KILL_PROCESS / KILL_THREAD / TRAP / ERRNO / USER_NOTIF / TRACE / LOG / ALLOW`. Filter chains are linked through `seccomp_filter.prev`, COW-shared across `fork()` and refcounted; on `exec` they survive iff the task either holds `CAP_SYS_ADMIN` or has `PR_SET_NO_NEW_PRIVS` set. `SECCOMP_FILTER_FLAG_TSYNC` synchronises the filter to all sibling threads; `_NEW_LISTENER` returns an `anon_inode` fd implementing `SECCOMP_RET_USER_NOTIF` (read = pending notification, write = userspace decision, poll = readiness, ioctl = `NOTIF_RECV` / `NOTIF_SEND` / `NOTIF_ID_VALID` / `NOTIF_ADDFD` / `NOTIF_SET_FLAGS`). Critical for: containers / runtimes / sandboxes / Chrome-class renderer isolation.

This Tier-3 covers `kernel/seccomp.c` (~2569 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct seccomp_filter` | per-filter (BPF prog + chain + cache + notif) | `SeccompFilter` |
| `struct seccomp_knotif` | per-USER_NOTIF in-flight notification | `SeccompKnotif` |
| `struct seccomp_kaddfd` | per-ADDFD ioctl request | `SeccompKaddfd` |
| `struct notification` | per-listener-side queue | `SeccompNotification` |
| `struct action_cache` | per-filter native/compat const-allow bitmap | `SeccompActionCache` |
| `populate_seccomp_data()` | per-entry sd snapshot | `Seccomp::populate_data` |
| `seccomp_run_filters()` | per-entry classic-BPF eval | `Seccomp::run_filters` |
| `seccomp_check_filter()` | per-install opcode validator | `Seccomp::check_filter` |
| `seccomp_prepare_filter()` | per-install kernel cBPF assembly | `Seccomp::prepare_filter` |
| `seccomp_prepare_user_filter()` | per-install user-copyin | `Seccomp::prepare_user_filter` |
| `seccomp_attach_filter()` | per-install chain link | `Seccomp::attach_filter` |
| `seccomp_assign_mode()` | per-install mode commit | `Seccomp::assign_mode` |
| `seccomp_may_assign_mode()` | per-install monotonicity gate | `Seccomp::may_assign_mode` |
| `seccomp_can_sync_threads()` / `seccomp_sync_threads()` | per-TSYNC | `Seccomp::can_sync_threads` / `sync_threads` |
| `is_ancestor()` | per-TSYNC chain check | `Seccomp::is_ancestor` |
| `seccomp_cache_prepare()` / `_bitmap()` / `seccomp_is_const_allow()` | per-filter syscall fast-path cache | `Seccomp::cache_prepare` / `is_const_allow` |
| `__secure_computing()` / `secure_computing_strict()` / `__secure_computing_strict()` / `__seccomp_filter()` | per-syscall entry | `Seccomp::secure_computing` / `_strict` / `_filter` |
| `seccomp_do_user_notification()` | per-USER_NOTIF wait | `Seccomp::do_user_notification` |
| `seccomp_handle_addfd()` | per-ADDFD install | `Seccomp::handle_addfd` |
| `seccomp_log()` | per-action audit emit | `Seccomp::log` |
| `seccomp_set_mode_strict()` | per-mode-1 install | `Seccomp::set_mode_strict` |
| `seccomp_set_mode_filter()` | per-mode-2 install | `Seccomp::set_mode_filter` |
| `seccomp_get_action_avail()` | per-introspection action probe | `Seccomp::get_action_avail` |
| `seccomp_get_notif_sizes()` | per-introspection struct sizes | `Seccomp::get_notif_sizes` |
| `do_seccomp()` / `SYSCALL_DEFINE3(seccomp,...)` | per-syscall dispatch | `Seccomp::do_op` / `sys_seccomp` |
| `prctl_set_seccomp()` / `prctl_get_seccomp()` | per-prctl entry | `Seccomp::prctl_set` / `prctl_get` |
| `seccomp_notify_ioctl()` / `_recv()` / `_send()` / `_id_valid()` / `_set_flags()` / `_addfd()` / `_poll()` / `_release()` | per-fd interface | `Seccomp::Notify::*` |
| `init_listener()` / `has_duplicate_listener()` | per-listener-fd creation | `Seccomp::init_listener` / `has_dup_listener` |
| `seccomp_filter_release()` / `__seccomp_filter_release()` / `__seccomp_filter_orphan()` / `__put_seccomp_filter()` / `__get_seccomp_filter()` / `get_seccomp_filter()` | per-lifecycle | `Seccomp::filter_release` / `put` / `get` |
| `seccomp_filter_free()` | per-free | `Seccomp::filter_free` |
| `seccomp_get_filter()` / `seccomp_get_metadata()` / `get_nth_filter()` | per-CHECKPOINT_RESTORE introspection | `Seccomp::get_filter` / `get_metadata` |
| `arch_seccomp_spec_mitigate()` | per-Spectre v2/v4 mitigation hook | `Seccomp::arch_spec_mitigate` |
| `seccomp_actions_logged` / `seccomp_log_names[]` | per-sysctl audit gate | `Seccomp::actions_logged` |

## Compatibility contract

REQ-1: struct seccomp_filter:
- refs: refcount_t — per direct attachment + per dependent filter + (optionally) per notifier fd.
- users: refcount_t — per direct attachment + per dependent filter; reaching 0 implies no more tasks can become attached (drives `EPOLLHUP` for notifier).
- log: per-FILTER_FLAG_LOG flag.
- wait_killable_recv: per-FILTER_FLAG_WAIT_KILLABLE_RECV flag.
- cache: struct action_cache — per-arch syscall-nr `allow_native[]` / `allow_compat[]` const-allow bitmap.
- prev: previously installed / inherited filter (forms a per-task forest via fork()).
- prog: `struct bpf_prog *` — classic-BPF program.
- notif: optional `struct notification *` for `SECCOMP_RET_USER_NOTIF`.
- notify_lock: mutex protecting `notif`.
- wqh: waitqueue for poll() on the notifier fd.

REQ-2: struct seccomp_data (UAPI, populated per-syscall-entry):
- nr: i32 syscall number.
- arch: u32 `AUDIT_ARCH_*`.
- instruction_pointer: u64.
- args[0..5]: u64.

REQ-3: SECCOMP_MODE_* (current->seccomp.mode monotonicity):
- `SECCOMP_MODE_DISABLED` (0) — default; may transition to STRICT or FILTER once.
- `SECCOMP_MODE_STRICT` (1) — only `read`/`write`/`exit`/`sigreturn` allowed (and `uretprobe`/`uprobe` if defined).
- `SECCOMP_MODE_FILTER` (2) — BPF chain consulted per-syscall.
- `SECCOMP_MODE_DEAD` (internal, MODE_FILTER+1) — task hit a fatal action and is on its way out; subsequent `__secure_computing` re-entries `do_exit(SIGKILL)` after `WARN_ON_ONCE`.
- Once non-zero, `seccomp_may_assign_mode` rejects any change to a different mode.

REQ-4: do_seccomp(op, flags, uargs):
- op = `SECCOMP_SET_MODE_STRICT` → require `flags == 0 ∧ uargs == NULL` → `seccomp_set_mode_strict`.
- op = `SECCOMP_SET_MODE_FILTER` → `seccomp_set_mode_filter(flags, uargs)`.
- op = `SECCOMP_GET_ACTION_AVAIL` → `seccomp_get_action_avail(uargs)` (returns 0 iff action is one of `SECCOMP_RET_{KILL_PROCESS,KILL_THREAD,TRAP,ERRNO,USER_NOTIF,TRACE,LOG,ALLOW}`).
- op = `SECCOMP_GET_NOTIF_SIZES` → `seccomp_get_notif_sizes(uargs)` (writes `sizeof(seccomp_notif)`, `sizeof(seccomp_notif_resp)`, `sizeof(seccomp_data)`).
- default → -EINVAL.

REQ-5: SYSCALL_DEFINE3(seccomp, op, flags, uargs) ⇒ `do_seccomp(op, flags, uargs)`.

REQ-6: prctl_set_seccomp(seccomp_mode, filter):
- `SECCOMP_MODE_STRICT` → `do_seccomp(SET_MODE_STRICT, 0, NULL)` (filter ignored).
- `SECCOMP_MODE_FILTER` → `do_seccomp(SET_MODE_FILTER, 0, filter)`.
- else → -EINVAL.

REQ-7: prctl_get_seccomp() ⇒ `current->seccomp.mode`.

REQ-8: seccomp_set_mode_strict():
- spin_lock_irq(siglock).
- if !seccomp_may_assign_mode(STRICT) → -EINVAL.
- if TIF_NOTSC defined → `disable_TSC()`.
- `seccomp_assign_mode(current, STRICT, 0)` → sets `mode`, smp_mb__before_atomic, arch_seccomp_spec_mitigate if `!SPEC_ALLOW`, `set_task_syscall_work(SECCOMP)`.

REQ-9: seccomp_set_mode_filter(flags, filter):
- flag mask = `SECCOMP_FILTER_FLAG_MASK`; bits include `TSYNC`, `LOG`, `SPEC_ALLOW`, `NEW_LISTENER`, `TSYNC_ESRCH`, `WAIT_KILLABLE_RECV`.
- Reject `TSYNC + NEW_LISTENER` unless `TSYNC_ESRCH` is also set.
- Reject `WAIT_KILLABLE_RECV` without `NEW_LISTENER`.
- prepared = `seccomp_prepare_user_filter(filter)` (cBPF validation, refs=1, users=1).
- if `NEW_LISTENER`: allocate `O_CLOEXEC` fd, `init_listener(prepared)` (anon_inode `seccomp notify`).
- if `TSYNC`: take `cred_guard_mutex` (interleaves with `exec`).
- spin_lock_irq(siglock).
- if !`seccomp_may_assign_mode(FILTER)` → -EINVAL.
- if `has_duplicate_listener(prepared)` → -EBUSY (no nested notifiers).
- `seccomp_attach_filter(flags, prepared)`:
  - sum existing path length + 4-instr penalty per ancestor; reject if > `MAX_INSNS_PER_PATH` ((1<<18)/sizeof(sock_filter)).
  - if `TSYNC`: `seccomp_can_sync_threads()` → 0 or pid-of-bad-thread (or -ESRCH if `TSYNC_ESRCH`).
  - set `filter->log` from `FILTER_FLAG_LOG`; set `wait_killable_recv` from flag.
  - `filter->prev = current->seccomp.filter`; `seccomp_cache_prepare(filter)`; `current->seccomp.filter = filter`; `atomic_inc(filter_count)`.
  - if `TSYNC`: `seccomp_sync_threads(flags)`.
- `seccomp_assign_mode(current, FILTER, flags)`.
- if `NEW_LISTENER`: on success `fd_install(listener_fd, listener_file)` and return fd; on failure detach + put.

REQ-10: seccomp_prepare_filter(fprog):
- Per `task_no_new_privs(current) || ns_capable_noaudit(current_user_ns(), CAP_SYS_ADMIN)`; else -EACCES.
- Per `fprog->len in (0, BPF_MAXINSNS]`; else -EINVAL.
- `bpf_prog_create_from_user(&sfilter->prog, fprog, seccomp_check_filter, save_orig)`.
- `refcount_set(refs,1)`; `refcount_set(users,1)`; `init_waitqueue_head(wqh)`; `mutex_init(notify_lock)`.

REQ-11: seccomp_check_filter():
- Per-insn cBPF validator: rewrites loads of `struct seccomp_data` fields with `BPF_LDX_ABS` / `BPF_LD_*` shapes, rejects unsupported opcodes / out-of-range field offsets / `BPF_S_*` legacy escapes.

REQ-12: __secure_computing() entry (CONFIG_SECCOMP_FILTER):
- If `CONFIG_CHECKPOINT_RESTORE ∧ current->ptrace & PT_SUSPEND_SECCOMP` → 0 (bypass).
- this_syscall = `syscall_get_nr(current, current_pt_regs())`.
- switch on `current->seccomp.mode`:
  - `MODE_STRICT` → `__secure_computing_strict(this_syscall)` (may `do_exit(SIGKILL)`); return 0.
  - `MODE_FILTER` → `__seccomp_filter(this_syscall, false)`.
  - `MODE_DEAD` → WARN_ON_ONCE + `do_exit(SIGKILL)`; return -1.
  - default → BUG().

REQ-13: __secure_computing_strict(this_syscall):
- allowed_syscalls = `mode1_syscalls[] = { __NR_read, __NR_write, __NR_exit, __NR_sigreturn, [__NR_uretprobe], [__NR_uprobe], -1 }` (per-arch `get_compat_mode1_syscalls()` under compat).
- Per linear scan; if hit, return. Else: set mode=DEAD, `seccomp_log(_, SIGKILL, RET_KILL_THREAD, true)`, `do_exit(SIGKILL)`.

REQ-14: __seccomp_filter(this_syscall, recheck_after_trace):
- smp_rmb (pair with `seccomp_assign_mode`).
- `populate_seccomp_data(&sd)` (snapshots nr, arch, IP, args[0..5]).
- `filter_ret = seccomp_run_filters(&sd, &match)`.
- data = `filter_ret & SECCOMP_RET_DATA` (low 16 bits).
- action = `filter_ret & SECCOMP_RET_ACTION_FULL`.
- switch action:
  - `RET_ERRNO`: data = min(data, MAX_ERRNO); `syscall_set_return_value(current, regs, -data, 0)`; skip.
  - `RET_TRAP`: `syscall_rollback(current, regs)`; `force_sig_seccomp(this_syscall, data, false)` (SIGSYS, no coredump); skip.
  - `RET_TRACE`: if `recheck_after_trace` → 0. If !`ptrace_event_enabled(current, PTRACE_EVENT_SECCOMP)` → return -ENOSYS to caller; skip. Else `ptrace_event(PTRACE_EVENT_SECCOMP, data)`; if fatal_signal_pending → skip; reload nr; recurse with recheck=true.
  - `RET_USER_NOTIF`: `seccomp_do_user_notification(this_syscall, match, &sd)`; if returned non-zero → skip.
  - `RET_LOG`: `seccomp_log(_, 0, action, true)`; return 0 (allow).
  - `RET_ALLOW`: return 0 (no logging — ALLOW is the starting state and `match` is NULL).
  - `RET_KILL_THREAD` / `RET_KILL_PROCESS` / default: set mode=DEAD; `seccomp_log(_, SIGSYS, action, true)`; if last live thread or `KILL_PROCESS` → `syscall_rollback` + `force_sig_seccomp(this_syscall, data, true)` (SIGSYS coredump); else `do_exit(SIGSYS)`; return -1.
- skip label: `seccomp_log(_, 0, action, match ? match->log : false)`; return -1.

REQ-15: seccomp_run_filters(sd, *match):
- ret = `SECCOMP_RET_ALLOW`. f = `READ_ONCE(current->seccomp.filter)`.
- if f == NULL → WARN + return `SECCOMP_RET_KILL_PROCESS` (defense against fail-open).
- if `seccomp_cache_check_allow(f, sd)` (per-syscall-nr cached const-allow on native/compat arch) → return ALLOW (no BPF run).
- for f; f; f = f->prev: cur_ret = `bpf_prog_run_pin_on_cpu(f->prog, sd)`; if `ACTION_ONLY(cur_ret) < ACTION_ONLY(ret)` → ret = cur_ret; *match = f.
- return ret. /* Lowest action value wins. */

REQ-16: SECCOMP_RET_* numeric priority (lowest wins; `ACTION_ONLY = ret & SECCOMP_RET_ACTION_FULL`):
- `KILL_PROCESS` (0x80000000) ≺ `KILL_THREAD` (0x00000000) ≺ `TRAP` (0x00030000) ≺ `ERRNO` (0x00050000) ≺ `USER_NOTIF` (0x7fc00000) ≺ `TRACE` (0x7ff00000) ≺ `LOG` (0x7ffc0000) ≺ `ALLOW` (0x7fff0000).
- Bottom 16 bits = `SECCOMP_RET_DATA` payload.

REQ-17: seccomp_cache_prepare / seccomp_is_const_allow:
- Per-filter symbolic eval of cBPF when only `(nr, arch)` are read; if program collapses to `RET ALLOW` for a `(nr, arch)` pair, bitmap bit is left set; else cleared.
- New filter must be at least as restrictive as `prev` (`bitmap_copy` from previous bitmap, then `__clear_bit` for non-cacheable nrs).
- `__NR_uretprobe` / `__NR_uprobe` always count as allow (uprobe exception).

REQ-18: seccomp_sync_threads (TSYNC, post-`seccomp_can_sync_threads` success):
- Skip if `SIGNAL_GROUP_EXIT` (process dying).
- for_each_thread(caller, thread) != caller and !PF_EXITING:
  - `get_seccomp_filter(caller)`; `__seccomp_filter_release(thread->seccomp.filter)`; `smp_store_release(&thread->seccomp.filter, caller->seccomp.filter)`; copy `filter_count`.
  - if `task_no_new_privs(caller)` → `task_set_no_new_privs(thread)` (closes thread-NNP escape).
  - if `thread->seccomp.mode == DISABLED` → `seccomp_assign_mode(thread, FILTER, flags)`.

REQ-19: USER_NOTIF semantics — listener fd:
- Filter returns `SECCOMP_RET_USER_NOTIF` → tracee calls `seccomp_do_user_notification`:
  - allocate `struct seccomp_knotif n = { task=current, state=INIT, data=&sd, id=seccomp_next_notify_id(match), ready=COMPLETION }`.
  - list_add_tail to `match->notif->notifications`; `atomic_inc(requests)`; wake `match->wqh` (`EPOLLIN | EPOLLRDNORM`).
  - wait loop: `wait_for_completion_{interruptible,killable}(n.ready)` (killable iff `wait_killable_recv && state >= SENT`); on completion process `addfd` queue then check `state == REPLIED`.
  - on REPLIED: ret = `n.val`; err = `n.error`; flags = `n.flags`.
  - if `flags & SECCOMP_USER_NOTIF_FLAG_CONTINUE` → return 0 (let syscall run).
  - else `syscall_set_return_value(current, regs, err, ret)` and return -1.

REQ-20: seccomp_notify_ioctl (file ops on listener fd):
- `SECCOMP_IOCTL_NOTIF_RECV` → `seccomp_notify_recv`: wait until a knotif with state==INIT; copy into `struct seccomp_notif { id, pid, data }`; flip state=SENT; wake `wqh` with `EPOLLOUT|EPOLLWRNORM`. On copy_to_user failure, reset state=INIT.
- `SECCOMP_IOCTL_NOTIF_SEND` → `seccomp_notify_send`: copy `struct seccomp_notif_resp { id, val, error, flags }`. Reject unknown flags ≠ `SECCOMP_USER_NOTIF_FLAG_CONTINUE`. Reject `CONTINUE` with non-zero `error`/`val`. Find by id; require state==SENT (else -EINPROGRESS); set state=REPLIED + payload + `complete(ready)` (per `SECCOMP_USER_NOTIF_FD_SYNC_WAKE_UP` use `complete_on_current_cpu`).
- `SECCOMP_IOCTL_NOTIF_ID_VALID` (and broken-direction alias `_WRONG_DIR`) → returns 0 iff id matches a knotif in state==SENT.
- `SECCOMP_IOCTL_NOTIF_SET_FLAGS` → mutates `notif->flags` (only `SECCOMP_USER_NOTIF_FD_SYNC_WAKE_UP` accepted).
- `SECCOMP_IOCTL_NOTIF_ADDFD` (variable-size via `EA_IOCTL`) → `seccomp_notify_addfd`: validates `newfd_flags & ~O_CLOEXEC`, `flags & ~(SECCOMP_ADDFD_FLAG_SETFD|SEND)`; `fget(srcfd)`; require state==SENT; `list_add(&kaddfd.list, &knotif->addfd)`; `complete(&knotif->ready)`; wait on `kaddfd.completion`; install via `receive_fd` or `receive_fd_replace`; if `ADDFD_FLAG_SEND` → also concludes the reply.

REQ-21: seccomp_notify_poll(file, poll_tab):
- `poll_wait(file, &filter->wqh, poll_tab)`.
- For each notification: state==INIT → `EPOLLIN|EPOLLRDNORM`; state==SENT → `EPOLLOUT|EPOLLWRNORM`.
- `refcount_read(&filter->users) == 0` → `EPOLLHUP` (all tracees gone).

REQ-22: seccomp_notify_release(inode, file):
- Wake all pending knotifs with `error=-ENOSYS, val=0, state=REPLIED, complete(ready)` so tracees unblock.
- `seccomp_notify_free` (kfree notif); `__put_seccomp_filter(filter)`.

REQ-23: init_listener(filter):
- `filter->notif = kzalloc(*notif)`; `next_id = get_random_u64()`; `INIT_LIST_HEAD(notifications)`.
- `anon_inode_getfile("seccomp notify", &seccomp_notify_ops, filter, O_RDWR)`.
- `__get_seccomp_filter(filter)` (file owns a ref).

REQ-24: has_duplicate_listener(new_child):
- If `new_child->notif` and any ancestor in `current->seccomp.filter` chain already has a `notif` → reject (no nested listeners on the same task).

REQ-25: Filter inheritance (fork / clone):
- `get_seccomp_filter(child)` from `copy_process` increments per-filter `refs` and `users`.
- Filter chain is a forest: forked task shares the same `prev` pointer tree; new filters on child install new leaves while ancestors stay shared.
- `task_no_new_privs(parent)` ⇒ child inherits NNP.

REQ-26: Filter lifecycle on task exit:
- `seccomp_filter_release(tsk)` (PF_EXITING required): detach `tsk->seccomp.filter = NULL`; `__seccomp_filter_release(orig)` = `__seccomp_filter_orphan` (per-filter decrement `users` and wake `wqh` with `EPOLLHUP` when it hits 0) + `__put_seccomp_filter` (decrement `refs`; iterate up to free single-ref ancestors).

REQ-27: arch_seccomp_spec_mitigate(task):
- Default __weak no-op. x86 applies STIBP / SSBD per `current->seccomp.mode == FILTER` unless `SECCOMP_FILTER_FLAG_SPEC_ALLOW` is set.

REQ-28: PR_SET_NO_NEW_PRIVS interaction:
- `seccomp_prepare_filter` requires `task_no_new_privs(current) || ns_capable_noaudit(user_ns, CAP_SYS_ADMIN)`.
- TSYNC propagates NNP to siblings so an unprivileged task cannot drop NNP by spawning a helper.

REQ-29: CHECKPOINT_RESTORE introspection (ptrace-only path):
- `PTRACE_SECCOMP_GET_FILTER` → `seccomp_get_filter(task, filter_off, data)` → `get_nth_filter` walks `prev` chain. Requires `capable(CAP_SYS_ADMIN) && current->seccomp.mode == DISABLED`.
- `PTRACE_SECCOMP_GET_METADATA` → returns `{ filter_off, flags = (filter->log ? FILTER_FLAG_LOG : 0) }`.

REQ-30: seccomp_log audit policy:
- `seccomp_actions_logged` bitmap of `SECCOMP_LOG_*` controls which actions emit audit messages.
- Defaults: `KILL_PROCESS | KILL_THREAD | TRAP | ERRNO | USER_NOTIF | TRACE | LOG` (i.e. all except `ALLOW`).
- `RET_KILL_*` and `RET_LOG` always log when their bit is set; `TRAP/ERRNO/TRACE/USER_NOTIF` log only when `requested` (i.e. their filter has `FILTER_FLAG_LOG`).
- sysctl `kernel.seccomp.actions_logged` exposes the mask via `seccomp_log_names[]`.

## Acceptance Criteria

- [ ] AC-1: `prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT)` on a clean task → `current->seccomp.mode == STRICT`; subsequent syscall not in `mode1_syscalls[]` ⇒ `do_exit(SIGKILL)`.
- [ ] AC-2: `seccomp(SECCOMP_SET_MODE_FILTER, 0, fprog)` without `PR_SET_NO_NEW_PRIVS` and without `CAP_SYS_ADMIN` → -EACCES.
- [ ] AC-3: A filter returning `SECCOMP_RET_ERRNO | EPERM` causes the syscall to return -EPERM without entering the handler.
- [ ] AC-4: A filter returning `SECCOMP_RET_TRAP | sig_data` delivers SIGSYS via `force_sig_seccomp` with `si_errno == sig_data`, no coredump.
- [ ] AC-5: A filter returning `SECCOMP_RET_KILL_PROCESS` kills the entire thread group (SIGSYS + coredump); `SECCOMP_RET_KILL_THREAD` kills only the calling thread unless it is the last live thread.
- [ ] AC-6: A filter returning `SECCOMP_RET_USER_NOTIF` blocks the tracee until the listener writes `SECCOMP_IOCTL_NOTIF_SEND { id, val, error }`; tracee resumes with that result; `FLAG_CONTINUE` causes the original syscall to proceed.
- [ ] AC-7: `SECCOMP_RET_TRACE` without an attached ptracer expecting `PTRACE_EVENT_SECCOMP` returns -ENOSYS to userspace.
- [ ] AC-8: Multiple filters: lowest action value wins; ties resolved by lowest filter (chain order).
- [ ] AC-9: `SECCOMP_FILTER_FLAG_TSYNC` succeeds iff every sibling is `DISABLED` or has the calling task's filter as an ancestor; failure returns the offending pid (or -ESRCH with `TSYNC_ESRCH`).
- [ ] AC-10: After `fork()`, child inherits parent's filter chain (`prev` pointer shared, refs bumped); independent filter installs on child do not affect parent.
- [ ] AC-11: `seccomp(SECCOMP_GET_ACTION_AVAIL, 0, &action)` returns 0 for every action listed in REQ-16; returns -EOPNOTSUPP for an unknown value.
- [ ] AC-12: `seccomp(SECCOMP_GET_NOTIF_SIZES, 0, &sizes)` writes exactly `sizeof(seccomp_notif)`, `sizeof(seccomp_notif_resp)`, `sizeof(seccomp_data)`.
- [ ] AC-13: Listener fd: `read`/`ioctl(NOTIF_RECV)` blocks until a notification is INIT, then returns it; `poll()` returns `EPOLLIN` when pending and `EPOLLHUP` once all tracees have exited.
- [ ] AC-14: `ioctl(NOTIF_ADDFD)` installs an fd in the tracee at `newfd` (or next free if `newfd==-1` and `!SETFD`); `ADDFD_FLAG_SEND` doubles as a reply.
- [ ] AC-15: Mode-transition monotonicity: once `current->seccomp.mode != 0`, attempts to change to a *different* non-zero mode return -EINVAL.

## Architecture

```
struct SeccompFilter {
  refs: AtomicU32,                      // refcount_t — total references
  users: AtomicU32,                     // refcount_t — direct task attachments
  log: bool,                            // FILTER_FLAG_LOG
  wait_killable_recv: bool,             // FILTER_FLAG_WAIT_KILLABLE_RECV
  cache: SeccompActionCache,            // per-arch const-allow bitmaps
  prev: Option<Arc<SeccompFilter>>,     // ancestor in tree
  prog: BpfProg,                        // classic-BPF program
  notif: Option<Box<SeccompNotification>>,
  notify_lock: Mutex<()>,
  wqh: WaitQueueHead,
}

struct SeccompKnotif {
  task: *TaskStruct,
  id: u64,
  data: *const SeccompData,             // valid for whole wait
  state: NotifyState,                   // INIT / SENT / REPLIED
  error: i32,
  val: i64,
  flags: u32,
  ready: Completion,
  list: ListHead,
  addfd: ListHead,                      // pending SeccompKaddfd
}

struct SeccompKaddfd {
  file: *File,
  fd: i32,                              // -1 = allocate
  flags: u32,                           // O_CLOEXEC only
  ioctl_flags: u32,                     // ADDFD_FLAG_SETFD | SEND
  setfd_or_ret: union { setfd: bool, ret: i32 },
  completion: Completion,
  list: ListHead,
}

struct SeccompData {                    // UAPI, populated per-syscall
  nr: i32,
  arch: u32,
  instruction_pointer: u64,
  args: [u64; 6],
}
```

`Seccomp::do_op(op, flags, uargs) -> i64`:
1. match op:
   - `SET_MODE_STRICT`: require flags == 0 ∧ uargs == NULL → `set_mode_strict()`.
   - `SET_MODE_FILTER`: `set_mode_filter(flags, uargs)`.
   - `GET_ACTION_AVAIL`: `get_action_avail(uargs)`.
   - `GET_NOTIF_SIZES`: `get_notif_sizes(uargs)`.
   - else: -EINVAL.

`Seccomp::set_mode_strict()`:
1. spin_lock_irq(current.sighand.siglock).
2. if !`may_assign_mode(STRICT)` → -EINVAL.
3. arch-conditional `disable_TSC()`.
4. `assign_mode(current, STRICT, 0)`:
   - current.seccomp.mode = STRICT.
   - smp_mb__before_atomic.
   - if !(flags & SPEC_ALLOW) → `arch_spec_mitigate(current)`.
   - `set_task_syscall_work(SECCOMP)`.
5. spin_unlock_irq.

`Seccomp::set_mode_filter(flags, user_fprog)`:
1. validate flags ⊆ `SECCOMP_FILTER_FLAG_MASK`; reject `TSYNC|NEW_LISTENER` without `TSYNC_ESRCH`; reject `WAIT_KILLABLE_RECV` without `NEW_LISTENER`.
2. prepared = `prepare_user_filter(user_fprog)`.
3. if `NEW_LISTENER`: listener_fd = get_unused_fd(`O_CLOEXEC`); listener_file = `init_listener(prepared)`.
4. if `TSYNC`: take `cred_guard_mutex` (interruptible).
5. spin_lock_irq(sighand.siglock).
6. if !`may_assign_mode(FILTER)` → -EINVAL.
7. if `has_duplicate_listener(prepared)` → -EBUSY.
8. ret = `attach_filter(flags, prepared)`:
   - per-path instruction sum + 4 penalty/ancestor; ≤ `MAX_INSNS_PER_PATH`.
   - if `TSYNC`: ret = `can_sync_threads()`; non-zero → -ESRCH (TSYNC_ESRCH) or ret.
   - filter.log = flags & LOG; filter.wait_killable_recv = flags & WAIT_KILLABLE_RECV.
   - filter.prev = current.seccomp.filter.
   - `cache_prepare(filter)`.
   - current.seccomp.filter = filter; `atomic_inc(filter_count)`.
   - if `TSYNC`: `sync_threads(flags)`.
9. on success: `assign_mode(current, FILTER, flags)`; prepared = None (don't free).
10. spin_unlock_irq + `cred_guard_mutex.unlock()` if TSYNC.
11. if `NEW_LISTENER`: on success `fd_install(listener_fd, listener_file)` and return listener_fd; on failure put fd + detach.
12. `filter_free(prepared)` (no-op if attached).

`Seccomp::run_filters(sd) -> (action_full_u32, match: Option<&SeccompFilter>)`:
1. ret = `SECCOMP_RET_ALLOW`; match = None.
2. f = `READ_ONCE(current.seccomp.filter)`.
3. if f == None: WARN; return (`KILL_PROCESS`, None).
4. if `cache_check_allow(f, sd)` → return (ALLOW, None).
5. for cur in chain(f):
   - cur_ret = `bpf_prog_run_pin_on_cpu(cur.prog, sd)`.
   - if `ACTION_ONLY(cur_ret) < ACTION_ONLY(ret)`: ret = cur_ret; match = Some(cur).
6. return (ret, match).

`Seccomp::secure_computing() -> i32`:
1. if `current.ptrace & PT_SUSPEND_SECCOMP` → 0.
2. nr = `syscall_get_nr(current, regs)`.
3. match current.seccomp.mode:
   - STRICT → `secure_computing_strict(nr)` (may not return); 0.
   - FILTER → `__seccomp_filter(nr, false)`.
   - DEAD → WARN_ON_ONCE + `do_exit(SIGKILL)`.
   - DISABLED → 0.
   - else → BUG().

`Seccomp::__seccomp_filter(nr, recheck) -> i32`:
1. smp_rmb.
2. populate sd.
3. (ret_full, match) = `run_filters(sd)`.
4. data = ret_full & `RET_DATA`; action = ret_full & `RET_ACTION_FULL`.
5. dispatch per REQ-14 (ERRNO / TRAP / TRACE / USER_NOTIF / LOG / ALLOW / KILL_*).
6. on skip: `log(nr, 0, action, match?.log)`; return -1.

`Seccomp::do_user_notification(nr, match, sd) -> i32`:
1. mutex_lock(match.notify_lock).
2. if !match.notif → -ENOSYS.
3. n = {task=current, state=INIT, data=sd, id=`next_notify_id(match)`, ready=new Completion()}.
4. list_add_tail(&match.notif.notifications, &n.list).
5. atomic_inc(match.notif.requests).
6. wake match.wqh with EPOLLIN|EPOLLRDNORM (`_SYNC_WAKE_UP` → on current CPU).
7. loop:
   a. wait_killable = match.wait_killable_recv ∧ n.state ≥ SENT.
   b. mutex_unlock; wait_for_completion[_killable|_interruptible](n.ready); mutex_lock.
   c. on EINTR: if not eligible for killable upgrade → goto interrupted.
   d. addfd = list_first_entry_or_null(n.addfd); if present → `handle_addfd(addfd, &n)`.
   e. until n.state == REPLIED.
8. ret = n.val; err = n.error; flags = n.flags.
9. interrupted: drain n.addfd with ret=-ESRCH.
10. if match.notif still present → list_del(&n.list).
11. mutex_unlock.
12. if flags & `USER_NOTIF_FLAG_CONTINUE` → return 0.
13. else `syscall_set_return_value(current, regs, err, ret)`; return -1.

`Seccomp::Notify::recv(filter, buf) -> i64`:
1. `check_zeroed_user(buf, sizeof(seccomp_notif))`.
2. `recv_wait_event(filter)` — sleeps until requests > 0 (or users == 0).
3. mutex_lock(notify_lock).
4. find first knotif with state == INIT; if none → -ENOENT.
5. unotif = {id, pid=`task_pid_vnr(knotif.task)`, data=*knotif.data}.
6. knotif.state = SENT; wake wqh EPOLLOUT.
7. mutex_unlock.
8. on copy_to_user fail → relock, reset state=INIT, atomic_inc(requests), wake EPOLLIN.

`Seccomp::Notify::send(filter, buf) -> i64`:
1. copy_from_user `seccomp_notif_resp { id, val, error, flags }`.
2. validate flags ⊆ `USER_NOTIF_FLAG_CONTINUE`; CONTINUE ⇒ error==0 ∧ val==0.
3. mutex_lock(notify_lock); find knotif by id; require state==SENT.
4. knotif.{state=REPLIED, error, val, flags}; `complete(knotif.ready)` (or `complete_on_current_cpu`).

`Seccomp::Notify::addfd(filter, uaddfd, size)`:
1. copy `struct seccomp_notif_addfd { id, srcfd, newfd, newfd_flags, flags }` via `copy_struct_from_user`.
2. validate `newfd_flags ⊆ O_CLOEXEC`, `flags ⊆ (SETFD | SEND)`, `(newfd ≠ 0) ⇒ SETFD`.
3. kaddfd.file = fget(srcfd); else -EBADF.
4. mutex_lock; find knotif by id; require SENT.
5. if `ADDFD_FLAG_SEND` ∧ knotif.addfd non-empty → -EBUSY.
6. if `ADDFD_FLAG_SEND` → knotif.state = REPLIED.
7. list_add(&kaddfd.list, &knotif.addfd); `complete(knotif.ready)`.
8. mutex_unlock; wait_for_completion_interruptible(kaddfd.completion); return kaddfd.ret.

`Seccomp::Notify::poll(file, poll_tab) -> u32`:
1. poll_wait(file, &filter.wqh, poll_tab).
2. for each knotif: INIT → EPOLLIN|EPOLLRDNORM; SENT → EPOLLOUT|EPOLLWRNORM.
3. users == 0 → EPOLLHUP.

`Seccomp::filter_release(task)`:
1. require PF_EXITING.
2. spin_lock_irq(task.sighand.siglock); orig = task.seccomp.filter; task.seccomp.filter = None; spin_unlock_irq.
3. `__filter_orphan(orig)` — walk decrementing `users`; wake `wqh` with EPOLLHUP at 0.
4. `__put_filter(orig)` — walk decrementing `refs`; `filter_free` at 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mode_monotonic` | INVARIANT | per-`may_assign_mode`: once mode != DISABLED, no transition to a different non-zero mode. |
| `filter_path_bounded` | INVARIANT | per-`attach_filter`: total chain length ≤ MAX_INSNS_PER_PATH. |
| `lowest_action_wins` | INVARIANT | per-`run_filters`: returned action_full satisfies ACTION_ONLY(ret) ≤ ACTION_ONLY(any cur_ret). |
| `dead_mode_kills` | INVARIANT | per-`secure_computing`: MODE_DEAD ⇒ do_exit(SIGKILL). |
| `nnp_filter_install_gated` | INVARIANT | per-`prepare_filter`: !task_no_new_privs ∧ !CAP_SYS_ADMIN ⇒ -EACCES. |
| `tsync_or_listener_exclusive` | INVARIANT | per-`set_mode_filter`: (TSYNC ∧ NEW_LISTENER) ⇒ TSYNC_ESRCH. |
| `notif_state_machine` | INVARIANT | per-knotif: state ∈ {INIT → SENT → REPLIED}, no backward transitions except recv-fail reset SENT → INIT. |
| `addfd_only_when_sent` | INVARIANT | per-`Notify::addfd`: knotif.state == SENT at insertion. |
| `errno_capped` | INVARIANT | per-RET_ERRNO: data ← min(data, MAX_ERRNO). |
| `refs_balanced` | INVARIANT | per-filter: `refs` ≥ `users` at all observable points; both reach 0 only after `filter_release`. |

### Layer 2: TLA+

`kernel/seccomp.tla`:
- Per-install + per-fork-inherit + per-syscall-eval + per-USER_NOTIF + per-release.
- Properties:
  - `safety_mode_monotonic` — per-task mode transitions form a DAG `DISABLED → STRICT|FILTER → DEAD`.
  - `safety_filter_chain_acyclic` — per-task filter `prev` chain is finite and rooted at None.
  - `safety_tsync_atomic` — TSYNC either updates all sibling filters or none (cred_guard_mutex held).
  - `safety_notif_no_double_reply` — per-knotif transition INIT → SENT → REPLIED occurs at most once.
  - `safety_notif_listener_releases_unblock_tracees` — `notify_release` ⇒ every queued knotif eventually reaches REPLIED with -ENOSYS.
  - `safety_filter_release_orphan_wakes_poll` — per-filter `users → 0` ⇒ `wqh` wake EPOLLHUP.
  - `liveness_per_syscall_terminates` — per-`__seccomp_filter` returns within a bounded number of chain links.
  - `liveness_per_notif_replied_or_listener_died` — per-USER_NOTIF tracee unblocks within finite time iff listener responds or closes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Seccomp::do_op` post: ret ∈ {≥0, -EINVAL, -EFAULT, -EACCES, -ENOMEM, -EBUSY, -EOPNOTSUPP, listener_fd ≥ 0} | `Seccomp::do_op` |
| `Seccomp::set_mode_filter` post: success ⇒ mode == FILTER ∧ filter chain extended by one ∧ (NEW_LISTENER ⇒ ret = fd) | `Seccomp::set_mode_filter` |
| `Seccomp::run_filters` post: action_full == min over chain (under ACTION_ONLY ordering); ALLOW ⇒ match == None | `Seccomp::run_filters` |
| `Seccomp::__seccomp_filter` post: skip ⇒ syscall return value set via syscall_set_return_value | `Seccomp::__seccomp_filter` |
| `Seccomp::Notify::recv` post: knotif state INIT → SENT; on copy fail rolled back | `Seccomp::Notify::recv` |
| `Seccomp::Notify::send` post: knotif.state == SENT precondition; on success REPLIED + complete(ready) | `Seccomp::Notify::send` |
| `Seccomp::filter_release` post: task.seccomp.filter == None; refs/users walked to ancestors | `Seccomp::filter_release` |

### Layer 4: Verus/Creusot functional

`Per-syscall: populate_seccomp_data → run_filters → action dispatch (ERRNO/TRAP/TRACE/USER_NOTIF/LOG/ALLOW/KILL_*)` semantic equivalence: per-`Documentation/userspace-api/seccomp_filter.rst` and per-`tools/testing/selftests/seccomp/` (`seccomp_bpf.c`) reference behavior.

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

Seccomp reinforcement:

- **Per-mode monotonicity (`may_assign_mode`)** — defense against per-task mode-downgrade.
- **Per-PR_SET_NO_NEW_PRIVS gate on `prepare_filter`** — defense against per-unprivileged-suid-bypass via injected filter.
- **Per-TSYNC propagates NNP to siblings (`task_set_no_new_privs`)** — defense against per-thread-NNP-escape.
- **Per-`MAX_INSNS_PER_PATH` cap (256 KiB / sizeof(sock_filter))** — defense against per-chain-DoS.
- **Per-`bpf_prog_create_from_user` opcode validation (`seccomp_check_filter`)** — defense against per-malformed-cBPF.
- **Per-`seccomp_run_filters` fail-closed on NULL filter (WARN + RET_KILL_PROCESS)** — defense against per-fail-open.
- **Per-RET_ERRNO `min(data, MAX_ERRNO)`** — defense against per-syscall returning bogus errno.
- **Per-MODE_DEAD `do_exit(SIGKILL)` on re-entry** — defense against per-survived-fatal-action.
- **Per-listener fd: `has_duplicate_listener` rejects nested notifiers** — defense against per-double-listener confusion.
- **Per-USER_NOTIF `NOTIF_FLAG_CONTINUE` rejects non-zero error/val** — defense against per-listener-injects-result-on-continue ambiguity.
- **Per-USER_NOTIF `check_zeroed_user` on recv buffer** — defense against per-stale-struct-fields-leak.
- **Per-notif id `get_random_u64` seed** — defense against per-id-prediction by other listeners on the same filter.
- **Per-`cred_guard_mutex` held during TSYNC** — defense against per-mid-exec NNP/mode race.
- **Per-`PT_SUSPEND_SECCOMP` requires CAP_SYS_ADMIN + CONFIG_CHECKPOINT_RESTORE** — defense against per-trace-bypass abuse.
- **Per-`arch_seccomp_spec_mitigate` unless `SPEC_ALLOW`** — defense against per-Spectre v2/v4 in sandboxed code.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- BPF interpreter / JIT internals (covered in `kernel/bpf/` Tier-3)
- `audit_seccomp` audit framework (covered in `kernel/audit/` Tier-3)
- ptrace tracee-stop and `PTRACE_EVENT_SECCOMP` plumbing (covered in `ptrace.md` Tier-3)
- POSIX capability semantics (covered in `capability.md` Tier-3)
- `prctl` other than `PR_SET_SECCOMP` / `PR_SET_NO_NEW_PRIVS` (covered in `sys.md` Tier-3)
- `seccomp_user_dispatch` / syscall_user_dispatch (covered separately)
- Implementation code
