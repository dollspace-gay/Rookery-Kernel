# Tier-3: kernel/auditsc.c — Syscall audit core (per-task audit_context + per-rule filter + per-record AUDIT_PATH/CWD/EXECVE/EOE/SOCKADDR/IPC/FD_PAIR/CONFIG_CHANGE)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/audit/00-overview.md
upstream-paths:
  - kernel/auditsc.c (~2983 lines)
  - kernel/audit.h
  - kernel/audit.c (audit_log_*, audit_serial — shared with this file)
  - include/linux/audit.h
  - include/uapi/linux/audit.h
-->

## Summary

`kernel/auditsc.c` is the **system-call audit core**: the per-task `struct audit_context` that accumulates every syscall's arguments, opened paths, inodes, sockaddrs, IPC ops, capabilities, mqueue ops, BPF/module loads, fanotify decisions, ptrace targets, signal targets, and netfilter config changes — then emits a multi-record audit event at syscall exit gated by per-rule filter evaluation. Entry / exit are wired in via `__audit_syscall_entry(major, a1..a4)` and `__audit_syscall_exit(success, return_code)`, called from arch syscall trampolines + SYSCALL_AUDIT taskwork. Records emitted: `AUDIT_SYSCALL` (mandatory header — `arch=`, `syscall=`, `success=`, `exit=`, `a0..a3=`, `items=`, `ppid=`, `pid=`, `auid=`, `uid=`, `gid=`, `euid=`, `suid=`, `fsuid=`, `egid=`, `sgid=`, `fsgid=`, `tty=`, `ses=`, `comm=`, `exe=`, `key=`), `AUDIT_PATH` (one per audited filename, `item=N name= inode= dev= mode= ouid= ogid= rdev= nametype=NORMAL/PARENT/DELETE/CREATE cap_*`), `AUDIT_CWD` (per-syscall pwd at entry), `AUDIT_EXECVE` (`argc=N a0= a1= a2= ...` — split across multiple records if cumulative size > MAX_EXECVE_AUDIT_LEN=7500), `AUDIT_EOE` (per-event terminator — `auditd` uses to know multi-record event boundary), `AUDIT_SOCKADDR` (`saddr=<hex>`), `AUDIT_IPC` / `AUDIT_IPC_SET_PERM`, `AUDIT_FD_PAIR` (`fd0= fd1=`), `AUDIT_BPRM_FCAPS`, `AUDIT_CAPSET`, `AUDIT_MMAP`, `AUDIT_OPENAT2`, `AUDIT_KERN_MODULE`, `AUDIT_MQ_OPEN` / `_SENDRECV` / `_NOTIFY` / `_GETSETATTR`, `AUDIT_OBJ_PID` (per-signal-target), `AUDIT_FANOTIFY`, `AUDIT_NETFILTER_CFG`, `AUDIT_ANOM_ABEND` (per-coredump), `AUDIT_SECCOMP`, `AUDIT_CONFIG_CHANGE`, `AUDIT_PROCTITLE`, `AUDIT_TIME_INJOFFSET` / `_ADJNTPVAL`, `AUDIT_URINGOP` (io_uring). Per-record sequence number `stamp.serial` (assigned by `audit_serial()` in audit.c on first auditable call to `auditsc_get_stamp` for each event) ties all records of a single event together in the `audit(<ts>:<seq>)` prefix. Per-rule filter evaluation `audit_filter_rules(tsk, rule, ctx, name, &state, task_creation)` evaluates ~40 field operators across 7 filter lists: `AUDIT_FILTER_TASK` (at copy_process), `AUDIT_FILTER_ENTRY` (deprecated → rejected), `AUDIT_FILTER_EXIT` (at syscall exit), `AUDIT_FILTER_FS` (per-fstype reject), `AUDIT_FILTER_URING_EXIT` (io_uring exit), `AUDIT_FILTER_USER` (in audit.c), `AUDIT_FILTER_EXCLUDE` (in audit.c). Critical for: PCI-DSS, HIPAA, FISMA, Common Criteria, DISA-STIG compliance — every userspace boundary syscall on production servers logged byte-identical to upstream auditd consumption.

This Tier-3 covers `kernel/auditsc.c` (~2983 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct audit_context` | per-task syscall accumulator | `kernel::audit::sc::AuditContext` |
| `struct audit_aux_data` | per-aux-record linked-list header | `AuditAuxData` |
| `struct audit_aux_data_pids` | per-signal-target list | `AuditAuxDataPids` |
| `struct audit_aux_data_bprm_fcaps` | per-execve fcap escalation | `AuditAuxDataBprmFcaps` |
| `struct audit_tree_refs` | per-31-entry chunk-ref array | `AuditTreeRefs` |
| `struct audit_names` | per-audited-filename entry | `AuditNames` |
| `struct audit_stamp` | per-event timestamp + serial | `AuditStamp` |
| `enum audit_state` | `_DISABLED` / `_BUILD` / `_RECORD` | `AuditState` |
| `audit_alloc(tsk)` | per-task ctx alloc on fork | `Auditsc::alloc` |
| `audit_alloc_context(state)` | alloc helper | `Auditsc::alloc_context` |
| `audit_reset_context(ctx)` | per-event reset | `Auditsc::reset_context` |
| `audit_free_context(ctx)` | per-task ctx free | `Auditsc::free_context` |
| `__audit_free(tsk)` | per-task ctx free on exit | `Auditsc::free` |
| `__audit_syscall_entry(major, a1, a2, a3, a4)` | per-syscall entry hook | `Auditsc::syscall_entry` |
| `__audit_syscall_exit(success, ret)` | per-syscall exit hook | `Auditsc::syscall_exit` |
| `__audit_uring_entry(op)` | per-io_uring entry hook | `Auditsc::uring_entry` |
| `__audit_uring_exit(success, ret)` | per-io_uring exit hook | `Auditsc::uring_exit` |
| `audit_return_fixup(ctx, success, code)` | per-ERESTART* fixup | `Auditsc::return_fixup` |
| `audit_filter_task(tsk, key)` | per-fork TASK-filter eval | `Auditsc::filter_task` |
| `audit_filter_syscall(tsk, ctx)` | per-exit EXIT-filter eval | `Auditsc::filter_syscall` |
| `audit_filter_uring(tsk, ctx)` | per-exit URING-filter eval | `Auditsc::filter_uring` |
| `audit_filter_inodes(tsk, ctx)` | per-collected-inode rule eval | `Auditsc::filter_inodes` |
| `audit_filter_inode_name(tsk, n, ctx)` | per-inode hash-bucket eval | `Auditsc::filter_inode_name` |
| `audit_filter_rules(tsk, rule, ctx, name, state, task_creation)` | per-rule predicate eval | `Auditsc::filter_rules` |
| `__audit_filter_op(tsk, ctx, list, name, op)` | per-list+op eval helper | `Auditsc::filter_op` |
| `audit_in_mask(rule, val)` | per-syscall-mask check | `Auditsc::in_mask` |
| `audit_match_perm(ctx, mask)` | per-perm-class match | `Auditsc::match_perm` |
| `audit_match_filetype(ctx, val)` | per-filetype S_IFMT match | `Auditsc::match_filetype` |
| `audit_field_compare(...)` | per-cross-field UID/GID compare | `Auditsc::field_compare` |
| `audit_log_pid_context(...)` | per-OBJ_PID record emit | `Auditsc::log_pid_context` |
| `audit_log_execve_info(ctx, ab)` | per-EXECVE record emit (chunked) | `Auditsc::log_execve_info` |
| `audit_log_name(ctx, n, path, num, panic)` | per-PATH record emit | `Auditsc::log_name` |
| `audit_log_proctitle()` | per-PROCTITLE record emit | `Auditsc::log_proctitle` |
| `audit_log_uring(ctx)` | per-URINGOP record emit | `Auditsc::log_uring` |
| `audit_log_exit()` | per-event multi-record emit | `Auditsc::log_exit` |
| `audit_log_cap(ab, prefix, cap)` | per-cap-mask field | `Auditsc::log_cap` |
| `audit_log_fcaps(ab, name)` | per-fcap field | `Auditsc::log_fcaps` |
| `audit_log_time(ctx, ab)` | per-NTP/TK time-injection record | `Auditsc::log_time` |
| `show_special(ctx, call_panic)` | per-typed-record dispatcher | `Auditsc::show_special` |
| `auditsc_get_stamp(ctx, stamp)` | per-event serial alloc + auditable-set | `Auditsc::get_stamp` |
| `audit_alloc_name(ctx, type)` | per-name slot alloc | `Auditsc::alloc_name` |
| `__audit_getname(name)` | per-getname() hook | `Auditsc::getname` |
| `__audit_inode(name, dentry, flags)` | per-inode-lookup hook | `Auditsc::inode` |
| `__audit_inode_child(parent, dentry, type)` | per-create/delete hook | `Auditsc::inode_child` |
| `__audit_file(file)` | per-already-open file hook | `Auditsc::file` |
| `__audit_mq_open(oflag, mode, attr)` | per-mq_open | `Auditsc::mq_open` |
| `__audit_mq_sendrecv(mqdes, len, prio, abs_to)` | per-mq send/recv | `Auditsc::mq_sendrecv` |
| `__audit_mq_notify(mqdes, n)` | per-mq_notify | `Auditsc::mq_notify` |
| `__audit_mq_getsetattr(mqdes, mqstat)` | per-mq_get/setattr | `Auditsc::mq_getsetattr` |
| `__audit_ipc_obj(ipcp)` | per-IPC perms snapshot | `Auditsc::ipc_obj` |
| `__audit_ipc_set_perm(qbytes, uid, gid, mode)` | per-IPC perms-set | `Auditsc::ipc_set_perm` |
| `__audit_bprm(bprm)` | per-execve start | `Auditsc::bprm` |
| `__audit_socketcall(nargs, args)` | per-sys_socketcall | `Auditsc::socketcall` |
| `__audit_fd_pair(fd1, fd2)` | per-pipe2 / socketpair | `Auditsc::fd_pair` |
| `__audit_sockaddr(len, addr)` | per-bind/connect/sendto | `Auditsc::sockaddr` |
| `__audit_ptrace(t)` | per-ptrace target | `Auditsc::ptrace` |
| `audit_signal_info_syscall(t)` | per-signal target | `Auditsc::signal_info` |
| `__audit_log_bprm_fcaps(bprm, new, old)` | per-execve fcap | `Auditsc::log_bprm_fcaps` |
| `__audit_log_capset(new, old)` | per-capset | `Auditsc::log_capset` |
| `__audit_mmap_fd(fd, flags)` | per-mmap of fd | `Auditsc::mmap_fd` |
| `__audit_openat2_how(how)` | per-openat2 | `Auditsc::openat2_how` |
| `__audit_log_kern_module(name)` | per-finit_module | `Auditsc::log_kern_module` |
| `__audit_fanotify(resp, friar)` | per-fanotify decision | `Auditsc::fanotify` |
| `__audit_tk_injoffset(off)` | per-timekeeping offset | `Auditsc::tk_injoffset` |
| `__audit_ntp_log(ad)` | per-NTP adjust | `Auditsc::ntp_log` |
| `__audit_log_nfcfg(name, af, n, op, gfp)` | per-netfilter cfg | `Auditsc::log_nfcfg` |
| `audit_core_dumps(signr)` | per-coredump AUDIT_ANOM_ABEND | `Auditsc::core_dumps` |
| `audit_seccomp(syscall, signr, code)` | per-seccomp action | `Auditsc::seccomp` |
| `audit_seccomp_actions_logged(...)` | per-config-change for seccomp | `Auditsc::seccomp_actions_logged` |
| `audit_killed_trees()` | per-killed-tree list accessor | `Auditsc::killed_trees` |
| `handle_one(inode)` / `handle_path(dentry)` | per-tree-watch chunk-ref tracking | `Auditsc::handle_one` / `handle_path` |
| `put_tree_ref / grow_tree_refs / unroll_tree_refs / free_tree_refs / match_tree_refs` | per-tree-ref management | `Auditsc::TreeRefs::*` |
| `audit_n_rules`, `audit_signals` | global counters | shared |

## Compatibility contract

REQ-1: `struct audit_context` per-task fields (consumed via `tsk->audit_context`, accessed by `audit_context()` macro):
- `state` (AUDIT_STATE_DISABLED/_BUILD/_RECORD) — sticky per-task baseline from TASK-filter at fork.
- `context` (AUDIT_CTX_UNUSED / _SYSCALL / _URING) — current-event lifecycle.
- `current_state` — per-event mutable (filter may upgrade `_BUILD` → `_RECORD`).
- `prio` (u64) — rule-priority slot (~0ULL when state == _RECORD, 0 otherwise).
- `stamp` (struct audit_stamp): `.ctime` (struct timespec64 from `ktime_get_coarse_real_ts64`) + `.serial` (u32 from `audit_serial()` on demand).
- `arch` (u32 from `syscall_get_arch(current)`), `major` (int — syscall number), `argv[4]` (a0..a3 raw register values).
- `return_code` (long), `return_valid` (AUDITSC_INVALID / _SUCCESS / _FAILURE).
- `ppid` (lazy via `task_ppid_nr`), `personality` (snapshotted from `current->personality` at log_exit).
- `uid` / `euid` / `suid` / `fsuid` / `gid` / `egid` / `sgid` / `fsgid` (kuid_t / kgid_t).
- `names_list` (struct list_head of audit_names), `name_count` (int), `preallocated_names[AUDIT_NAMES]` (inline pool).
- `pwd` (struct path — captured by `get_fs_pwd` on first audit_alloc_name).
- `aux` (linked list of typed aux records — e.g., BPRM_FCAPS), `aux_pids` (signal-target aux).
- `trees` / `first_trees` / `tree_count` — `audit_tree_refs` chain (31-pointer arrays of chunk*).
- `killed_trees` (list_head — chunks killed during this event → CONFIG_CHANGE records).
- `sockaddr` (kmalloc'd, sockaddr_storage-sized) + `sockaddr_len`.
- `fds[2]` (-1 sentinel + pair).
- `type` (record-type tag for the typed payload — AUDIT_SOCKETCALL/IPC/MQ_*/MMAP/OPENAT2/EXECVE/KERN_MODULE/TIME_*/CAPSET).
- Union of typed payload structs: `socketcall`, `ipc`, `mq_open`, `mq_sendrecv`, `mq_notify`, `mq_getsetattr`, `capset`, `mmap`, `openat2`, `execve`, `module`, `time`, `proctitle`.
- `target_pid` / `target_auid` / `target_uid` / `target_sessionid` / `target_comm` / `target_ref` — first signal target slot.
- `uring_op` (u8), `dummy` (bool — fast-path: state==BUILD ∧ audit_n_rules==0).
- `filterkey` (char* — duplicated from matching rule).

REQ-2: `audit_alloc(tsk)` (called from copy_process, no lock):
- if !audit_ever_enabled: return 0.
- state = audit_filter_task(tsk, &key).
- if state == AUDIT_STATE_DISABLED: clear_task_syscall_work(tsk, SYSCALL_AUDIT); return 0.
- context = audit_alloc_context(state).
- if !context: kfree(key); audit_log_lost("out of memory in audit_alloc"); return -ENOMEM.
- context->filterkey = key.
- audit_set_context(tsk, context).
- set_task_syscall_work(tsk, SYSCALL_AUDIT).
- return 0.

REQ-3: `audit_alloc_context(state)`:
- context = kzalloc_obj(*context).
- context->context = AUDIT_CTX_UNUSED.
- context->state = state.
- context->prio = (state == AUDIT_STATE_RECORD ? ~0ULL : 0).
- INIT_LIST_HEAD(&context->killed_trees).
- INIT_LIST_HEAD(&context->names_list).
- context->fds[0] = -1.
- context->return_valid = AUDITSC_INVALID.

REQ-4: `__audit_syscall_entry(major, a1, a2, a3, a4)`:
- context = audit_context().
- if !audit_enabled || !context: return.
- WARN_ON(context->context != AUDIT_CTX_UNUSED || context->name_count).
- if invalid state: audit_panic("unrecoverable error in audit_syscall_entry()"); return.
- if context->state == AUDIT_STATE_DISABLED: return.
- context->dummy = !audit_n_rules.
- if !dummy && state == AUDIT_STATE_BUILD:
  - prio = 0.
  - if auditd_test_task(current): return (auditd itself bypasses syscall audit).
- context->arch = syscall_get_arch(current).
- context->major = major; argv[0..3] = a1..a4.
- context->context = AUDIT_CTX_SYSCALL.
- context->current_state = state.
- ktime_get_coarse_real_ts64(&context->stamp.ctime).

REQ-5: `__audit_syscall_exit(success, return_code)`:
- context = audit_context().
- if !context || context->dummy || context->context != AUDIT_CTX_SYSCALL: goto out.
- if !list_empty(&context->killed_trees): audit_kill_trees(context) (emits CONFIG_CHANGE for each killed tree).
- audit_return_fixup(context, success, return_code).
- audit_filter_syscall(current, context) (EXIT-filter list).
- audit_filter_inodes(current, context) (per-name hash buckets).
- if context->current_state != AUDIT_STATE_RECORD: goto out.
- audit_log_exit().
- out: audit_reset_context(context).

REQ-6: `audit_return_fixup(ctx, success, code)`:
- if code in [-ERESTART_RESTARTBLOCK, -ERESTARTSYS] ∧ code != -ENOIOCTLCMD: ctx->return_code = -EINTR.
- else: ctx->return_code = code.
- ctx->return_valid = success ? AUDITSC_SUCCESS : AUDITSC_FAILURE.

REQ-7: `audit_filter_task(tsk, &key) -> enum audit_state`:
- rcu_read_lock.
- list_for_each_entry_rcu(e, &audit_filter_list[AUDIT_FILTER_TASK], list):
  - if audit_filter_rules(tsk, &e->rule, NULL, NULL, &state, true):
    - if state == AUDIT_STATE_RECORD: *key = kstrdup(e->rule.filterkey, GFP_ATOMIC).
    - rcu_read_unlock; return state.
- rcu_read_unlock.
- return AUDIT_STATE_BUILD.

REQ-8: `audit_filter_rules(tsk, rule, ctx, name, &state, task_creation) -> int`:
- if ctx ∧ rule->prio <= ctx->prio: return 0 (already a higher-priority match).
- cred = rcu_dereference_check(tsk->cred, tsk == current || task_creation).
- for i in 0..rule->field_count:
  - f = &rule->fields[i].
  - switch f->type:
    - AUDIT_PID / _PPID / _UID / _EUID / _SUID / _FSUID / _GID / _EGID / _SGID / _FSGID / _LOGINUID / _LOGINUID_SET / _SESSIONID / _PERS / _ARCH / _EXIT / _SUCCESS / _DEVMAJOR / _DEVMINOR / _INODE / _OBJ_UID / _OBJ_GID / _WATCH / _DIR / _EXE / _SADDR_FAM / _SUBJ_USER..CLR / _OBJ_USER..LEV_HIGH / _ARG0..ARG3 / _FILTERKEY / _PERM / _FILETYPE / _FIELD_COMPARE / _LSM-rule operators (security_audit_rule_match) — each computes `result` (0/1) via audit_comparator / audit_uid_comparator / audit_gid_comparator / audit_watch_compare / match_tree_refs / audit_match_perm / audit_match_filetype / audit_field_compare.
  - if !result: return 0 (rule fails — all fields must match).
- if ctx:
  - if rule->filterkey: kfree(ctx->filterkey); ctx->filterkey = kstrdup(rule->filterkey, GFP_ATOMIC).
  - ctx->prio = rule->prio.
- switch rule->action:
  - AUDIT_NEVER: *state = AUDIT_STATE_DISABLED.
  - AUDIT_ALWAYS: *state = AUDIT_STATE_RECORD.
- return 1.

REQ-9: `audit_filter_syscall(tsk, ctx)`:
- if auditd_test_task(tsk): return.
- rcu_read_lock.
- __audit_filter_op(tsk, ctx, &audit_filter_list[AUDIT_FILTER_EXIT], NULL, ctx->major).
- rcu_read_unlock.

REQ-10: `audit_filter_uring(tsk, ctx)`:
- if auditd_test_task(tsk): return.
- rcu_read_lock.
- __audit_filter_op(tsk, ctx, &audit_filter_list[AUDIT_FILTER_URING_EXIT], NULL, ctx->uring_op).
- rcu_read_unlock.

REQ-11: `__audit_filter_op(tsk, ctx, list, name, op)`:
- list_for_each_entry_rcu(e, list, list):
  - if audit_in_mask(&e->rule, op) ∧ audit_filter_rules(tsk, &e->rule, ctx, name, &state, false):
    - ctx->current_state = state; return 1.
- return 0.

REQ-12: `audit_in_mask(rule, val)`:
- if val > 0xffffffff: return false.
- word = AUDIT_WORD(val); bit = AUDIT_BIT(val).
- if word >= AUDIT_BITMASK_SIZE: return false.
- return rule->mask[word] & bit.

REQ-13: per-name slot allocation `audit_alloc_name(ctx, type)`:
- if name_count < AUDIT_NAMES: aname = &preallocated_names[name_count]; memset(aname, 0, sizeof(*aname)).
- else: aname = kzalloc_obj(*aname, GFP_NOFS); if !aname return NULL; aname->should_free = true.
- aname->ino = AUDIT_INO_UNSET; aname->type = type.
- list_add_tail(&aname->list, &ctx->names_list); name_count++.
- if !ctx->pwd.dentry: get_fs_pwd(current->fs, &ctx->pwd).

REQ-14: `__audit_getname(name)` (called from fs/namei.c:getname):
- if context->context == AUDIT_CTX_UNUSED: return.
- n = audit_alloc_name(context, AUDIT_TYPE_UNKNOWN).
- n->name = name; n->name_len = AUDIT_NAME_FULL; name->aname = n; name->refcnt++.

REQ-15: `__audit_inode(name, dentry, flags)`:
- /* AUDIT_FILTER_FS list checked for AUDIT_NEVER per fsmagic — fs-exempt fast-exit */
- rcu_read_lock, scan audit_filter_list[AUDIT_FILTER_FS]; if AUDIT_FSTYPE match ∧ AUDIT_NEVER: rcu_read_unlock; return.
- rcu_read_unlock.
- locate or alloc audit_names entry (matching by inode || name).
- type = AUDIT_TYPE_PARENT if (flags & AUDIT_INODE_PARENT), else AUDIT_TYPE_NORMAL.
- handle_path(dentry) — chunk-ref capture for tree-watch.
- audit_copy_inode(n, dentry, inode, flags & AUDIT_INODE_NOEVAL).

REQ-16: `__audit_inode_child(parent, dentry, type)`:
- AUDIT_FILTER_FS fast-exit per parent fsmagic.
- if inode: handle_one(inode).
- locate matching parent + child slots in names_list.
- create anonymous parent record if none; create child record if none.
- copy inode info for child.
- EXPORT_SYMBOL_GPL.

REQ-17: `auditsc_get_stamp(ctx, stamp) -> int`:
- if ctx->context == AUDIT_CTX_UNUSED: return 0.
- if !ctx->stamp.serial: ctx->stamp.serial = audit_serial() (audit_serial defined in kernel/audit.c — monotonic per-record-event counter, wraps).
- *stamp = ctx->stamp.
- if !ctx->prio: ctx->prio = 1; ctx->current_state = AUDIT_STATE_RECORD (auto-promote to recordable).
- return 1.

REQ-18: `audit_log_exit()` (the multi-record emit on syscall exit, AUDIT_CTX_SYSCALL):
- context->personality = current->personality.
- ab = audit_log_start(context, GFP_KERNEL, AUDIT_SYSCALL).
- audit_log_format(ab, "arch=%x syscall=%d", context->arch, context->major).
- if personality != PER_LINUX: " per=%lx".
- if return_valid != AUDITSC_INVALID: " success=%s exit=%ld".
- " a0=%lx a1=%lx a2=%lx a3=%lx items=%d".
- audit_log_task_info(ab) — appends ppid=, pid=, auid=, uid=, gid=, euid=, suid=, fsuid=, egid=, sgid=, fsgid=, tty=, ses=, comm=, exe=, subj= (from kernel/audit.c).
- audit_log_key(ab, context->filterkey).
- audit_log_end(ab) (final commit + netlink send via audit.c).
- for aux in context->aux: emit per-type record (AUDIT_BPRM_FCAPS payload formatted here).
- if context->type: show_special(context, &call_panic) — emit typed payload record (SOCKETCALL / IPC / IPC_SET_PERM / MQ_* / CAPSET / MMAP / OPENAT2 / EXECVE / KERN_MODULE / TIME_*).
- if fds[0] >= 0: emit AUDIT_FD_PAIR "fd0=%d fd1=%d".
- if sockaddr_len: emit AUDIT_SOCKADDR "saddr=<hex>".
- for aux_pids: emit AUDIT_OBJ_PID per target (audit_log_pid_context).
- if target_pid: emit AUDIT_OBJ_PID for primary target.
- if pwd.dentry ∧ pwd.mnt: emit AUDIT_CWD "cwd=<path>".
- for n in names_list: if !n->hidden: audit_log_name(ctx, n, NULL, i++, &call_panic) → AUDIT_PATH per name.
- if context == AUDIT_CTX_SYSCALL: audit_log_proctitle() → AUDIT_PROCTITLE.
- emit AUDIT_EOE — empty record, marks end-of-event for auditd reassembly.
- if call_panic: audit_panic("error in audit_log_exit()").

REQ-19: `audit_log_execve_info(ctx, ab)`:
- buf_head = kmalloc(MAX_EXECVE_AUDIT_LEN + 1 = 7501, GFP_KERNEL); if !buf_head: audit_panic("out of memory for argv string"); return.
- audit_log_format(*ab, "argc=%d", context->execve.argc).
- For each arg (0..argc):
  - strnlen_user-determined len_full.
  - strncpy_from_user up to len_max from current->mm->arg_start.
  - on -EFAULT: send_sig(SIGKILL, current, 0); goto out.
  - Encode if arg contains control char (audit_string_contains_control) — emit hex.
  - else emit quoted string.
  - chunk record across multiple AUDIT_EXECVE records if needed: audit_log_end old; audit_log_start new AUDIT_EXECVE.
  - emit per-arg field: " a%d_len=%lu" (first iter only, if multi-part) + " a%d[%d]=" or " a%d=".

REQ-20: `audit_log_name(ctx, n, path, record_num, call_panic)` — emits AUDIT_PATH:
- ab = audit_log_start(context, GFP_KERNEL, AUDIT_PATH).
- audit_log_format(ab, "item=%d", record_num).
- if path: audit_log_d_path(ab, " name=", path).
- else if n->name: emit based on n->name_len (AUDIT_NAME_FULL = full untrustedstring; 0 = cwd-relative → log pwd; default = directory component only).
- if ino != AUDIT_INO_UNSET: " inode=%llu dev=%02x:%02x mode=%#ho ouid=%u ogid=%u rdev=%02x:%02x".
- if lsmprop_set: audit_log_obj_ctx (LSM context).
- nametype= NORMAL / PARENT / DELETE / CREATE / UNKNOWN.
- audit_log_fcaps — cap_fp / _fi / _fe / _fver / _frootid.
- audit_log_end(ab).

REQ-21: `audit_log_proctitle()`:
- ab = audit_log_start(context, GFP_KERNEL, AUDIT_PROCTITLE).
- audit_log_format(ab, "proctitle=").
- if context->proctitle.value not cached: get_cmdline(current, buf, MAX_PROCTITLE_AUDIT_LEN=128); audit_proctitle_rtrim non-printable; cache.
- audit_log_n_untrustedstring(ab, value, len).
- audit_log_end.

REQ-22: `audit_reset_context(ctx)` (per-event teardown):
- ctx->context = AUDIT_CTX_UNUSED.
- if dummy: return (early-exit).
- ctx->current_state = ctx->state; stamp.serial = 0; stamp.ctime zeroed; major = 0; uring_op = 0; argv zeroed.
- return_code = 0; prio = (state == _RECORD ? ~0ULL : 0); return_valid = AUDITSC_INVALID.
- audit_free_names(ctx) — list + put filenames + path_put pwd.
- if state != _RECORD: kfree(filterkey).
- audit_free_aux(ctx) — both aux and aux_pids chains.
- kfree(sockaddr); sockaddr_len = 0.
- ppid, uid/euid/suid/fsuid, gid/egid/sgid/fsgid, personality, arch, target_*, lsmprop reset.
- unroll_tree_refs(ctx, NULL, 0) — drop chunk refs.
- WARN_ON(!list_empty(&ctx->killed_trees)).
- audit_free_module — kfree module.name.
- fds[0] = -1.
- type = 0 (reset last so audit_free_* see prior type).

REQ-23: `__audit_free(tsk)` (from do_exit / fork-error):
- context = tsk->audit_context.
- if !context: return.
- if !list_empty(&killed_trees): audit_kill_trees(context).
- if tsk == current ∧ !context->dummy:
  - return_valid = AUDITSC_INVALID; return_code = 0.
  - if AUDIT_CTX_SYSCALL: audit_filter_syscall + audit_filter_inodes; if state == _RECORD: audit_log_exit().
  - else if AUDIT_CTX_URING: audit_filter_uring + audit_filter_inodes; if state == _RECORD: audit_log_uring(context).
- audit_set_context(tsk, NULL).
- audit_free_context(context).

REQ-24: io_uring entry/exit:
- `__audit_uring_entry(op)`: if state == _DISABLED: return; ctx->uring_op = op; if already in AUDIT_CTX_SYSCALL: return; ctx->dummy = !audit_n_rules; if !dummy ∧ state == _BUILD: prio = 0; ctx->context = AUDIT_CTX_URING; ctx->current_state = ctx->state; capture stamp.ctime.
- `__audit_uring_exit(success, code)`: audit_return_fixup; if AUDIT_CTX_SYSCALL: filter syscall, then uring as needed; emit AUDIT_URINGOP via audit_log_uring; bail (syscall path will handle full record). Else: kill_trees if any; filter_uring + filter_inodes; if state == _RECORD: audit_log_exit. Always audit_reset_context.

REQ-25: per-event typed-payload `show_special(ctx, &call_panic)` — single ab record dispatched by ctx->type:
- AUDIT_SOCKETCALL: "nargs=%d a0=%lx ..."
- AUDIT_IPC: "ouid=%u ogid=%u mode=%#ho" + LSM ctx; if has_perm → second AUDIT_IPC_SET_PERM "qbytes= ouid= ogid= mode=".
- AUDIT_MQ_OPEN: "oflag= mode= mq_flags= mq_maxmsg= mq_msgsize= mq_curmsgs=".
- AUDIT_MQ_SENDRECV: "mqdes= msg_len= msg_prio= abs_timeout_sec= abs_timeout_nsec=".
- AUDIT_MQ_NOTIFY: "mqdes= sigev_signo=".
- AUDIT_MQ_GETSETATTR: "mqdes= mq_flags= mq_maxmsg= mq_msgsize= mq_curmsgs=".
- AUDIT_CAPSET: "pid=" + cap_pi/pp/pe/pa.
- AUDIT_MMAP: "fd= flags=".
- AUDIT_OPENAT2: "oflag= mode= resolve=".
- AUDIT_EXECVE: audit_log_execve_info (multi-record).
- AUDIT_KERN_MODULE: "name=<untrustedstring>".
- AUDIT_TIME_ADJNTPVAL / AUDIT_TIME_INJOFFSET: audit_log_time.

REQ-26: tree-watch chunk references (`audit_tree_refs`):
- 31-pointer arrays of `audit_chunk *`, chained via ->next.
- handle_one(inode): if inode has fsnotify_marks, audit_tree_lookup; put_tree_ref or grow_tree_refs(31) or unroll on OOM + audit_set_auditable + audit_put_chunk.
- handle_path(dentry): walks up dentry chain under rename_lock seq + RCU; for each backing inode with marks → lookup chunk → put_tree_ref. Retry on rename_lock seq-retry or chunk-drop.
- match_tree_refs(ctx, tree): scan full arrays then partial last array.
- free_tree_refs frees chain on context destroy.

REQ-27: signal-info recording (`audit_signal_info_syscall(t)`):
- if !audit_signals || audit_dummy_context: return 0.
- First target slot: ctx->target_pid/auid/uid/sessionid/comm/ref direct in ctx.
- Subsequent: kzalloc aux_pids (AUDIT_OBJ_PID type), AUDIT_AUX_PIDS=16 slot capacity; chain via aux_pids head.

REQ-28: `audit_core_dumps(signr)`:
- if !audit_enabled: return.
- if signr == SIGQUIT: return (user-initiated; ignore).
- emit AUDIT_ANOM_ABEND: audit_log_task + " sig=%ld res=1".

REQ-29: `audit_seccomp(syscall, signr, code)`:
- emit AUDIT_SECCOMP: audit_log_task + " sig= arch= syscall= compat= ip= code=".
- Bypasses audit_enabled / dummy gate — seccomp actions always log.
- `audit_seccomp_actions_logged(names, old_names, res)`: AUDIT_CONFIG_CHANGE "op=seccomp-logging actions= old-actions= res=".

REQ-30: per-record format / API used in this file (defined in kernel/audit.c):
- `audit_log_start(ctx, gfp, type)` → struct audit_buffer * (skb-backed) — also writes timestamp prefix "audit(<sec>.<ms>:<seq>): ".
- `audit_log_format(ab, fmt, ...)` → vsnprintf into skb tail.
- `audit_log_n_string(ab, str, len)` → quoted ASCII (escape control chars).
- `audit_log_n_untrustedstring(ab, str, len)` → hex-encode if any unprintable.
- `audit_log_n_hex(ab, buf, len)` → ASCII hex.
- `audit_log_end(ab)` → finalize skb + queue to netlink AUDIT_NLGRP_READLOG / direct unicast to auditd.
- `audit_log_d_path(ab, prefix, path)` → resolve and quote `<prefix><path>`.

REQ-31: per-record sequence number `audit_serial()`:
- Defined in kernel/audit.c; monotonic per-event counter; allocated lazily on first auditsc_get_stamp call per event; same value shared across all records of one event (so auditd reassembly works on `audit(<ts>:<seq>)` match).
- Cleared on audit_reset_context.

REQ-32: filter-list registry (`audit_filter_list` indexed by AUDIT_FILTER_*):
- AUDIT_FILTER_TASK (0): consulted at audit_alloc (fork).
- AUDIT_FILTER_ENTRY (1): deprecated/rejected; old kernels evaluated at syscall entry.
- AUDIT_FILTER_EXIT (2): at __audit_syscall_exit + __audit_free.
- AUDIT_FILTER_FS (5): at __audit_inode / _inode_child for fstype-based skip.
- AUDIT_FILTER_URING_EXIT (7): at __audit_uring_exit.
- Plus AUDIT_FILTER_USER / EXCLUDE handled in kernel/audit.c.

## Acceptance Criteria

- [ ] AC-1: After audit_alloc, tsk->audit_context allocated; SYSCALL_AUDIT task-work set unless TASK-filter says DISABLED.
- [ ] AC-2: __audit_syscall_entry sets context->context = AUDIT_CTX_SYSCALL and records arch + argv[0..3].
- [ ] AC-3: __audit_syscall_exit with ctx->current_state == _RECORD emits AUDIT_SYSCALL + zero-or-more AUDIT_PATH + AUDIT_CWD + AUDIT_PROCTITLE + AUDIT_EOE — all with same audit(<ts>:<serial>) prefix.
- [ ] AC-4: audit_filter_rules with rule->action == AUDIT_NEVER and all-fields-match returns 1 and sets *state = AUDIT_STATE_DISABLED.
- [ ] AC-5: audit_in_mask rejects syscall numbers > 0xffffffff or word >= AUDIT_BITMASK_SIZE.
- [ ] AC-6: ERESTARTSYS / ERESTARTNOINTR / ERESTARTNOHAND / ERESTART_RESTARTBLOCK (excluding ENOIOCTLCMD) rewritten to -EINTR by audit_return_fixup.
- [ ] AC-7: audit_log_execve_info emits "argc=N" then chunked AUDIT_EXECVE records when cumulative arg-length exceeds MAX_EXECVE_AUDIT_LEN=7500 bytes; -EFAULT from userspace strncpy_from_user triggers SIGKILL.
- [ ] AC-8: __audit_socketcall rejects nargs <= 0 or > AUDITSC_ARGS with -EINVAL.
- [ ] AC-9: __audit_sockaddr stores up to sizeof(sockaddr_storage) bytes; emits AUDIT_SOCKADDR "saddr=<hex>" at log_exit if sockaddr_len > 0.
- [ ] AC-10: __audit_fd_pair fills fds[0..1]; AUDIT_FD_PAIR record "fd0=%d fd1=%d" emitted iff fds[0] >= 0.
- [ ] AC-11: audit_signal_info_syscall first call writes into ctx->target_*; subsequent into aux_pids list (AUDIT_AUX_PIDS=16 per chunk); produces one AUDIT_OBJ_PID record per target at log_exit.
- [ ] AC-12: audit_core_dumps with SIGQUIT does nothing; other signal produces AUDIT_ANOM_ABEND with task info + sig=N res=1.
- [ ] AC-13: audit_seccomp emits AUDIT_SECCOMP regardless of audit_enabled or dummy state (seccomp bypasses both gates).
- [ ] AC-14: auditsc_get_stamp lazily allocates ctx->stamp.serial via audit_serial() and auto-promotes prio == 0 to ctx->prio = 1 + current_state = AUDIT_STATE_RECORD.
- [ ] AC-15: handle_path retries on rename_lock seq-retry; OOM in grow_tree_refs sets ctx auditable + drops chunk + unrolls.

## Architecture

```
pub struct AuditContext {
    state: AuditState,
    context: AuditCtxKind,           // UNUSED / SYSCALL / URING
    current_state: AuditState,
    prio: u64,
    dummy: bool,
    stamp: AuditStamp,                // .ctime, .serial
    arch: u32,
    major: i32,                       // syscall nr
    argv: [u64; 4],
    return_code: i64,
    return_valid: i32,                // INVALID / SUCCESS / FAILURE
    ppid: PidT,
    personality: u64,
    uid: KUidT,
    euid: KUidT,
    suid: KUidT,
    fsuid: KUidT,
    gid: KGidT,
    egid: KGidT,
    sgid: KGidT,
    fsgid: KGidT,
    names_list: ListHead,             // AuditNames
    name_count: i32,
    preallocated_names: [AuditNames; AUDIT_NAMES],
    pwd: Path,
    aux: Option<NonNull<AuditAuxData>>,
    aux_pids: Option<NonNull<AuditAuxData>>,
    trees: Option<NonNull<AuditTreeRefs>>,
    first_trees: Option<NonNull<AuditTreeRefs>>,
    tree_count: i32,
    killed_trees: ListHead,
    sockaddr: Option<KBox<SockaddrStorage>>,
    sockaddr_len: i32,
    fds: [i32; 2],
    type_: u32,                       // record-type tag for typed payload
    payload: AuditPayload,            // union over socketcall/ipc/mq_*/capset/mmap/openat2/execve/module/time/proctitle
    target_pid: PidT,
    target_auid: KUidT,
    target_uid: KUidT,
    target_sessionid: u32,
    target_comm: [u8; TASK_COMM_LEN],
    target_ref: LsmProp,
    uring_op: u8,
    filterkey: Option<KString>,
}
```

`Auditsc::syscall_entry(major, a1, a2, a3, a4)`:
1. ctx = audit_context().
2. if !audit_enabled || !ctx: return.
3. WARN if ctx.context != UNUSED || ctx.name_count != 0.
4. on inconsistency: audit_panic; return.
5. if ctx.state == AuditState::Disabled: return.
6. ctx.dummy = (audit_n_rules == 0).
7. if !ctx.dummy && ctx.state == AuditState::Build:
   - ctx.prio = 0.
   - if auditd_test_task(current): return.
8. ctx.arch = syscall_get_arch(current).
9. ctx.major = major.
10. ctx.argv = [a1, a2, a3, a4].
11. ctx.context = AuditCtxKind::Syscall.
12. ctx.current_state = ctx.state.
13. ktime_get_coarse_real_ts64(&mut ctx.stamp.ctime).

`Auditsc::syscall_exit(success, return_code)`:
1. ctx = audit_context().
2. if !ctx || ctx.dummy || ctx.context != AuditCtxKind::Syscall: goto out.
3. if !ctx.killed_trees.is_empty(): audit_kill_trees(ctx) (emits CONFIG_CHANGE per tree).
4. Self::return_fixup(ctx, success, return_code).
5. Self::filter_syscall(current, ctx).
6. Self::filter_inodes(current, ctx).
7. if ctx.current_state != AuditState::Record: goto out.
8. Self::log_exit().
9. out: Self::reset_context(ctx).

`Auditsc::filter_rules(tsk, rule, ctx, name, &mut state, task_creation) -> i32`:
1. if ctx && rule.prio <= ctx.prio: return 0.
2. cred = rcu_dereference_check(tsk.cred, tsk == current || task_creation).
3. for i in 0..rule.field_count:
   - f = &rule.fields[i].
   - match f.r#type: /* 40+ AUDIT_* field types */
     - Self::eval_field(tsk, cred, ctx, name, f) → result.
   - if result == 0: return 0.
4. if ctx:
   - if rule.filterkey.is_some(): drop(ctx.filterkey); ctx.filterkey = kstrdup(rule.filterkey, GFP_ATOMIC).
   - ctx.prio = rule.prio.
5. match rule.action:
   - AUDIT_NEVER: state = AuditState::Disabled.
   - AUDIT_ALWAYS: state = AuditState::Record.
6. return 1.

`Auditsc::log_exit()`:
1. ctx.personality = current.personality.
2. match ctx.context:
   - Syscall:
     - ab = audit_log_start(ctx, GFP_KERNEL, AUDIT_SYSCALL).
     - audit_log_format(ab, "arch=%x syscall=%d", ctx.arch, ctx.major).
     - if personality != PER_LINUX: " per=%lx".
     - if return_valid != INVALID: " success=%s exit=%ld".
     - " a0=%lx a1=%lx a2=%lx a3=%lx items=%d".
     - audit_log_task_info(ab).
     - audit_log_key(ab, ctx.filterkey).
     - audit_log_end(ab).
   - Uring: Self::log_uring(ctx).
   - else BUG().
3. for aux in ctx.aux.iter(): emit per-type record (BPRM_FCAPS).
4. if ctx.type_: Self::show_special(ctx).
5. if ctx.fds[0] >= 0: emit AUDIT_FD_PAIR.
6. if ctx.sockaddr_len != 0: emit AUDIT_SOCKADDR.
7. for aux_pids in ctx.aux_pids.iter(): emit AUDIT_OBJ_PID per slot.
8. if ctx.target_pid: emit primary AUDIT_OBJ_PID.
9. if ctx.pwd.dentry: emit AUDIT_CWD.
10. let mut i = 0;
11. for n in ctx.names_list.iter():
    - if n.hidden: continue.
    - Self::log_name(ctx, n, None, i, &mut call_panic); i += 1.
12. if ctx.context == Syscall: Self::log_proctitle().
13. ab = audit_log_start(ctx, GFP_KERNEL, AUDIT_EOE); audit_log_end(ab).
14. if call_panic: audit_panic("error in audit_log_exit()").

`Auditsc::log_execve_info(ctx, ab)`:
1. buf_head = kmalloc(MAX_EXECVE_AUDIT_LEN + 1, GFP_KERNEL).
2. if buf_head.is_null(): audit_panic("out of memory for argv string"); return.
3. audit_log_format(ab, "argc=%d", ctx.execve.argc).
4. p = current.mm.arg_start.
5. for arg in 0..ctx.execve.argc:
   - len_full = strnlen_user(p, MAX_ARG_STRLEN) - 1 if needed.
   - len_tmp = strncpy_from_user(buf_head, p, len_max).
   - if len_tmp == -EFAULT: send_sig(SIGKILL, current, 0); goto out.
   - decide encode (control-char detect or buffer-spanning forced).
   - if remaining < sizeof(abuf)+8: audit_log_end old AUDIT_EXECVE; ab = audit_log_start new AUDIT_EXECVE.
   - emit "a%d_len=%lu" or "a%d[%d]=" or "a%d=".
   - audit_log_n_hex (encoded) or audit_log_n_string (quoted).
6. out: kfree(buf_head).

`Auditsc::get_stamp(ctx, stamp) -> i32`:
1. if ctx.context == AuditCtxKind::Unused: return 0.
2. if ctx.stamp.serial == 0: ctx.stamp.serial = audit_serial().
3. *stamp = ctx.stamp.
4. if ctx.prio == 0:
   - ctx.prio = 1.
   - ctx.current_state = AuditState::Record.
5. return 1.

`Auditsc::reset_context(ctx)`:
1. ctx.context = AuditCtxKind::Unused.
2. if ctx.dummy: return.
3. ctx.current_state = ctx.state.
4. ctx.stamp.serial = 0; ctx.stamp.ctime = TimeSpec64::zero().
5. ctx.major = 0; ctx.uring_op = 0; ctx.argv = [0; 4].
6. ctx.return_code = 0; ctx.return_valid = AUDITSC_INVALID.
7. ctx.prio = if ctx.state == Record { !0u64 } else { 0 }.
8. Self::free_names(ctx).
9. if ctx.state != Record: drop(ctx.filterkey).
10. Self::free_aux(ctx).
11. drop(ctx.sockaddr); ctx.sockaddr_len = 0.
12. ctx.ppid = 0; reset uid/gid sets; personality = 0; arch = 0; target_* = 0.
13. lsmprop_init(&mut ctx.target_ref); ctx.target_comm[0] = 0.
14. Self::unroll_tree_refs(ctx, None, 0).
15. WARN_ON(!ctx.killed_trees.is_empty()).
16. Self::free_module(ctx); ctx.fds[0] = -1.
17. ctx.type_ = 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `context_lifecycle_unused_or_syscall_or_uring` | INVARIANT | per-ctx.context ∈ {UNUSED, SYSCALL, URING}. |
| `entry_requires_unused` | INVARIANT | per-syscall_entry: assert ctx.context == UNUSED on entry. |
| `exit_requires_syscall` | INVARIANT | per-syscall_exit: ctx.context == SYSCALL gates log_exit. |
| `serial_lazy_allocate` | INVARIANT | per-stamp.serial: assigned exactly once per event (zero on reset). |
| `record_state_promotes_prio` | INVARIANT | per-get_stamp ∧ prio == 0: post-condition prio == 1 ∧ current_state == RECORD. |
| `return_fixup_erestart_to_eintr` | INVARIANT | per-return_fixup: code in [-ERESTART_RESTARTBLOCK, -ERESTARTSYS] ∧ != -ENOIOCTLCMD ⟹ return_code = -EINTR. |
| `filter_rules_short_circuit_zero` | INVARIANT | per-filter_rules: any field result == 0 ⟹ return 0. |
| `filter_rules_action_to_state` | INVARIANT | per-filter_rules match: AUDIT_NEVER → DISABLED, AUDIT_ALWAYS → RECORD. |
| `in_mask_bounds` | INVARIANT | per-audit_in_mask: val > 0xffffffff || word >= AUDIT_BITMASK_SIZE ⟹ false. |
| `execve_buffer_bounded` | INVARIANT | per-log_execve_info: kmalloc <= MAX_EXECVE_AUDIT_LEN+1; -EFAULT ⟹ SIGKILL current. |
| `socketcall_arg_bounds` | INVARIANT | per-socketcall: nargs ∈ [1, AUDITSC_ARGS] else -EINVAL. |
| `tree_refs_31_per_array` | INVARIANT | per-tree_refs: tree_count ∈ [0, 31]; chunks past tree_count are NULL. |
| `fd_pair_sentinel` | INVARIANT | per-fds[0] == -1 ⟹ no AUDIT_FD_PAIR emitted. |
| `aux_pids_chunk_capped` | INVARIANT | per-aux_pids: pid_count ≤ AUDIT_AUX_PIDS=16; overflow ⟹ new chunk. |
| `reset_context_no_killed_trees` | INVARIANT | per-reset_context: WARN if killed_trees non-empty. |

### Layer 2: TLA+

`kernel/audit/auditsc.tla`:
- Variables: ctx_context (per task), names_list_size, aux_chain_len, aux_pids_count, killed_trees_pending, serial_assigned, current_state, prio.
- Actions: AuditAlloc, SyscallEntry, GetName, Inode, InodeChild, Sockaddr, FdPair, Bprm, MqOpen, IpcObj, SyscallExit, UringEntry, UringExit, Free.
- Properties:
  - `safety_entry_exit_paired` — per-task: every SyscallEntry on a task is followed by SyscallExit before next SyscallEntry (or by Free).
  - `safety_context_kind_progression` — UNUSED → (SYSCALL | URING) → UNUSED transitions only.
  - `safety_serial_assigned_once_per_event` — serial_assigned increments per event, reset on context-reset.
  - `safety_killed_trees_drained_before_unused` — context cannot return to UNUSED with non-empty killed_trees.
  - `safety_filter_priority_monotonic` — per-event: ctx.prio is non-decreasing through filter_rules.
  - `liveness_syscall_exit_emits_if_record` — current_state == RECORD ∧ SyscallExit ⟹ audit_log_exit ⟹ EOE record.
  - `liveness_eventually_reset` — every SyscallExit eventually reaches audit_reset_context.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Auditsc::syscall_entry` post: ctx.context == SYSCALL, argv recorded, stamp.ctime set | `Auditsc::syscall_entry` |
| `Auditsc::syscall_exit` post: ctx.context == UNUSED post-reset | `Auditsc::syscall_exit` |
| `Auditsc::return_fixup` post: ERESTART* mapped to -EINTR; return_valid in {SUCCESS, FAILURE} | `Auditsc::return_fixup` |
| `Auditsc::filter_rules` post: action ⟹ state; prio updated only on full match | `Auditsc::filter_rules` |
| `Auditsc::filter_op` post: state = updated only on (in_mask ∧ filter_rules) | `Auditsc::filter_op` |
| `Auditsc::log_exit` post: AUDIT_SYSCALL + (zero+) typed-payload + AUDIT_PATH-per-name + AUDIT_CWD + AUDIT_PROCTITLE + AUDIT_EOE, all with same serial | `Auditsc::log_exit` |
| `Auditsc::log_execve_info` post: argc=N + chunked AUDIT_EXECVE; -EFAULT ⟹ SIGKILL | `Auditsc::log_execve_info` |
| `Auditsc::get_stamp` post: stamp.serial nonzero; prio ≥ 1; current_state = RECORD | `Auditsc::get_stamp` |
| `Auditsc::reset_context` post: all fields zeroed/freed; type_ last reset | `Auditsc::reset_context` |
| `Auditsc::free` post: tsk.audit_context = NULL | `Auditsc::free` |

### Layer 4: Verus/Creusot functional

`Per-syscall-audit lifecycle (audit_alloc on fork → syscall_entry → 0..N collect-hooks → filter_syscall → filter_inodes → audit_log_exit → audit_reset_context → eventual audit_free)` semantic equivalence: per Documentation/audit/* + linux-audit user-space ABI (auditctl + auditd + ausearch consume record format byte-identical).

`Per-rule filter evaluation (audit_filter_rules across 40+ field types: PID/UID/GID/LOGINUID/PERS/ARCH/EXIT/SUCCESS/DEVMAJOR/MINOR/INODE/OBJ_UID/OBJ_GID/WATCH/DIR/EXE/SADDR_FAM/SUBJ_*/OBJ_*/ARG0-3/FILTERKEY/PERM/FILETYPE/FIELD_COMPARE)` semantic equivalence: per uapi/linux/audit.h + auditctl(8).

## Hardening

(Inherits row-1 features from `kernel/audit/00-overview.md` § Hardening.)

Auditsc reinforcement:

- **Per-state machine enforce (UNUSED → SYSCALL/URING → UNUSED)** — defense against per-double-entry / per-uring-from-syscall confusion.
- **Per-WARN_ON in syscall_entry on non-UNUSED context** — defense against per-recursive-syscall audit double-init.
- **Per-audit_panic("unrecoverable error in audit_syscall_entry()")** — defense against per-state-corruption silent log loss.
- **Per-context.dummy fast-path when audit_n_rules == 0** — defense against per-zero-rule full-record overhead.
- **Per-AUDIT_PERMIT auditd_test_task self-exempt** — defense against per-auditd self-recursion (auditd reading audit netlink itself).
- **Per-rule field-AND short-circuit** — defense against per-misconfigured rule emitting on partial match.
- **Per-prio rule supersession (rule.prio > ctx.prio gates re-eval)** — defense against per-low-priority rule overriding high.
- **Per-AUDIT_BITMASK_SIZE bounded audit_in_mask** — defense against per-OOB rule->mask access.
- **Per-MAX_EXECVE_AUDIT_LEN=7500 cap with chunked records** — defense against per-very-large-argv kernel allocation blowup; -EFAULT ⟹ SIGKILL.
- **Per-MAX_PROCTITLE_AUDIT_LEN=128 cap** — defense against per-arg_start-walk OOM.
- **Per-AUDITSC_ARGS clamp in socketcall** — defense against per-socketcall arg-array overflow.
- **Per-AUDIT_AUX_PIDS=16 per chunk in aux_pids** — defense against per-signal-flood single-alloc growth.
- **Per-sockaddr kmalloc sockaddr_storage size** — defense against per-saddr struct-size-confusion (AF_UNIX 108 vs AF_INET 16).
- **Per-rename_lock seq-retry in handle_path** — defense against per-rename TOCTOU on chunk capture.
- **Per-grow_tree_refs OOM ⟹ audit_set_auditable (record despite missing chunk)** — defense against per-partial-tree-watch-loss silent.
- **Per-AUDIT_FILTER_FS AUDIT_NEVER fast-exit** — defense against per-tmpfs / procfs flood.
- **Per-AUDIT_INODE_NOEVAL** — defense against per-d_path race on shadow-mount.
- **Per-AUDITSC_INVALID return_valid sentinel** — defense against per-stale-return-code emission.
- **Per-ERESTART* mapped to -EINTR** — defense against per-userspace seeing internal kernel codes.
- **Per-audit_serial monotonic, per-event** — defense against per-record reassembly-ambiguity in auditd.
- **Per-AUDIT_EOE terminator** — defense against per-auditd partial-event-flush (event boundary signal).
- **Per-audit_log_n_untrustedstring hex-escape** — defense against per-userspace-content terminal-injection in log.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- `kernel/audit.c` netlink dispatch + audit_log_start/_format/_end implementation + audit_serial (covered in `kernel/audit/audit.md` Tier-3)
- `kernel/auditfilter.c` rule add/delete/list netlink ops (covered in `kernel/audit/audit.md` Tier-3 — same parent overview)
- `kernel/audit_tree.c` audit_chunk + fsnotify mark mgmt (covered separately)
- `kernel/audit_watch.c` audit_watch + path-watch fsnotify (covered separately)
- `kernel/audit_fsnotify.c` fsnotify integration (covered separately)
- userspace `auditctl` / `auditd` / `ausearch` / `aureport`
- LSM `security_audit_rule_match` per-LSM bridge (covered in respective LSM Tier-3)
- io_uring `__io_audit_*` site-wiring (covered in `io_uring/` Tier-3)
- Implementation code
