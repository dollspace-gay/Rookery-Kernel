---
title: "Tier-3: kernel/audit.c — auditd subsystem (per-task audit_context + per-rule filter + netlink dispatch + watch/tree)"
tags: ["tier-3", "kernel-audit", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

The audit subsystem records security-relevant kernel events (syscall entry/exit, file access, mount, sysctl, capability use, etc.) for compliance + intrusion-detection. Per-task `audit_context` tracks current syscall arguments + return; per-rule (audit_filter_rule) configured by userspace `auditctl` filters which events generate audit-records; matching events are pushed to userspace auditd via netlink (NETLINK_AUDIT). audit_tree.c implements per-directory subtree filtering using fsnotify; audit_watch.c tracks per-path file-open auditing. Critical for SELinux/AppArmor + DISA-STIG compliance + post-breach forensics.

This Tier-3 covers `audit.c` (~2841) + `auditsc.c` (~2983) + `audit_tree.c` (~1093) + `audit_watch.c` (~547).

### Acceptance Criteria

- [ ] AC-1: Boot with auditctl -e 1; configure rule "auditctl -a always,exit -F arch=b64 -S openat"; touch /tmp/x; verify SYSCALL record emitted.
- [ ] AC-2: Filter by uid: rule "-F uid=1000"; user-1000 syscalls audited; root not.
- [ ] AC-3: Watch test: auditctl -w /etc/passwd; cat /etc/passwd; verify PATH record.
- [ ] AC-4: Tree test: auditctl -w /etc; touch /etc/foo; verify subtree-rule fires.
- [ ] AC-5: Backlog overflow: synthetic flood; backlog hits limit; configured failure mode (drop/printk/panic) honored.
- [ ] AC-6: Multi-CPU: 32 CPUs concurrent syscalls; all audited records arrive at auditd.
- [ ] AC-7: Live rule update: auditctl -D + add new rules; takes effect immediately.
- [ ] AC-8: Per-rule LSM context (SELinux): rule "-F obj_user=httpd_t"; only httpd-context syscalls audited.
- [ ] AC-9: Linux Test Project audit suite passes.

### Architecture

`AuditContext` per-task:

```
struct AuditContext {
  state: AuditState,                          // BuildContext, RecordContext, Dummy
  serial: u32,
  ctime: u64,
  major: u32,                                  // syscall number
  arch: u32,
  argv: [u64; 4],
  return_code: i64,
  success: bool,
  names_list: ListHead,
  aux_list: ListHead,
  loginuid: KuidT,
  sessionid: u32,
  pid: i32,
  ppid: i32,
  ...
}

struct AuditKrule {
  pflags: u32,
  flags: u32,
  listnr: u32,
  action: u32,
  mask: KBitmap,                               // syscall bitmap
  buflen: u32,
  field_count: u32,
  fields: KVec<AuditField>,
  tree: Option<KArc<AuditTree>>,
  watch: Option<KArc<AuditWatch>>,
  prio: u32,
}
```

`Auditsc::syscall_entry(major, a0, a1, a2, a3)`:
1. ctx := current.audit_context.
2. If !ctx: return (audit disabled).
3. ctx.major = major; ctx.argv[..] = [a0, a1, a2, a3].
4. ctx.state = BuildContext.
5. audit_filter_syscall(current, ctx).
6. If matched rule action == ALWAYS: ctx.state = RecordContext.

`Auditsc::syscall_exit(success, ret)`:
1. ctx := current.audit_context.
2. ctx.success = success; ctx.return_code = ret.
3. If ctx.state == RecordContext:
   - audit_log_exit(ctx).
4. Free per-syscall names_list + aux_list.
5. ctx.state = BuildContext.

`AuditLog::start(ctx, gfp, type)`:
1. Alloc skbuff with audit-header (type + len + ts + serial + uid).
2. Return audit_buffer wrapping skbuff.

`AuditLog::end(ab)`:
1. Append final newline.
2. nlmsg_end(ab.skb, ab.nlh).
3. queue_work(audit_wq, &send_work) — async netlink dispatch.

`AuditFilter::syscall(task, ctx)`:
1. Walk rule-list AUDIT_FILTER_EXIT.
2. Per-rule:
   - If !rule.mask[ctx.major]: skip.
   - For each rule.field: evaluate predicate against ctx.
   - If all-match:
     - If action == NEVER: stop matching; ctx.state stays.
     - If action == ALWAYS: ctx.state = RecordContext; break.

`AuditWatch::add(rule, path)`:
1. Resolve path → inode.
2. Allocate watch struct.
3. Register fsnotify watcher for path.
4. Link to rule.

### Out of Scope

- Per-LSM specifics (SELinux audit covered in `security/selinux/audit.md` future Tier-3)
- audit-userspace (userspace concern)
- inotify / fanotify (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct audit_context` | per-task syscall-context | `kernel::audit::AuditContext` |
| `struct audit_buffer` | per-record alloc'd skbuff | `AuditBuffer` |
| `struct audit_filter_list` | per-list filter rule chain | `AuditFilterList` |
| `struct audit_krule` | per-rule kernel-side | `AuditKrule` |
| `struct audit_field` | per-field rule predicate | `AuditField` |
| `audit_alloc(task)` | per-task ctx alloc | `AuditContext::alloc` |
| `audit_free(task)` | per-task ctx free | `AuditContext::free` |
| `audit_syscall_entry(major, a0, a1, a2, a3)` (auditsc.c) | syscall-entry hook | `Auditsc::syscall_entry` |
| `audit_syscall_exit(success, ret)` | syscall-exit hook | `Auditsc::syscall_exit` |
| `__audit_filesystem(...)` | per-FS-op record | `Auditsc::filesystem` |
| `__audit_inode(...)` / `__audit_inode_child(...)` | per-inode access | `Auditsc::inode` / `_inode_child` |
| `__audit_mq_open(...)` / `_mq_sendrecv(...)` | per-mqueue ops | `Auditsc::mq_*` |
| `__audit_socketcall(...)` | per-socket ops | `Auditsc::socketcall` |
| `audit_log_start(ctx, gfp, type)` | per-record begin | `AuditLog::start` |
| `audit_log_format(ab, fmt, ...)` | per-record append | `AuditLog::format` |
| `audit_log_end(ab)` | per-record commit + send | `AuditLog::end` |
| `audit_filter_syscall(task, ctx)` | per-rule eval at syscall | `AuditFilter::syscall` |
| `audit_filter_inode_name(...)` | per-inode rule eval | `AuditFilter::inode_name` |
| `audit_send_reply(...)` | netlink reply send | `Audit::send_reply` |
| `audit_receive(skb)` | netlink receive entry | `Audit::receive` |
| `audit_add_rule(...)` (auditfilter.c) | AUDIT_ADD_RULE op | `AuditFilter::add_rule` |
| `audit_del_rule(...)` | AUDIT_DEL_RULE op | `AuditFilter::del_rule` |
| `audit_make_rule(...)` | parse user-rule to krule | `AuditFilter::make_rule` |
| `audit_log_user_message(...)` | userspace-generated record | `AuditLog::user_message` |
| `audit_kill_trees(...)` (audit_tree.c) | per-tree teardown | `AuditTree::kill` |
| `audit_add_watch(...)` (audit_watch.c) | per-watch register | `AuditWatch::add` |

### compatibility contract

REQ-1: Per-task `audit_context`:
- `state` (AUDIT_BUILD_CONTEXT / _RECORD_CONTEXT / _DUMMY).
- `serial` (per-task event-id sequence).
- `arch` (syscall arch).
- `major` (syscall number).
- `argv[0..3]` (syscall args).
- `return_code` (return value).
- `success` (bool: !error).
- `names_list` (per-syscall files-touched list).
- `aux` (per-syscall aux-data list).
- `pid`, `loginuid`, `sessionid`.
- `current_state` (running, dummy).

REQ-2: Per-rule `audit_krule`:
- `pflags` (rule type: PER_TASK / FS / EXIT / EXCLUDE / FILTER_FS).
- `flags` (rule list: SYSCALL / TASK / FS / EXCLUDE).
- `listnr` (filter-list number 0-7).
- `action` (NEVER / POSSIBLE / ALWAYS).
- `mask[]` (syscall bitmap; one bit per syscall).
- `field_count`.
- `fields[]` (predicate field array).
- `tree` (audit_tree if FS rule).
- `watch` (audit_watch if WATCH rule).
- `prio` (priority).

REQ-3: Per-field `audit_field`:
- `type` (AUDIT_PID / _UID / _GID / _ARCH / _SUCCESS / _OBJ_USER / etc.).
- `op` (AUDIT_EQUAL / _LESS_THAN / _GREATER_THAN / etc.).
- `val` (compare value).
- `lsm_str` (LSM-context string for SELinux match).
- `lsm_rule` (compiled LSM-rule).

REQ-4: Filter lists:
- AUDIT_FILTER_TASK (0): task-creation rules.
- AUDIT_FILTER_ENTRY (1): syscall entry (deprecated).
- AUDIT_FILTER_WATCH (2): watch rules.
- AUDIT_FILTER_EXIT (3): syscall exit.
- AUDIT_FILTER_TYPE (4): user-record exclusions.
- AUDIT_FILTER_FS (5): filesystem-op rules.

REQ-5: syscall-entry hook flow (`audit_syscall_entry`):
1. ctx := current.audit_context.
2. ctx.major = major; ctx.arch = arch; ctx.argv[] = a0..a3.
3. ctx.state = AUDIT_BUILD_CONTEXT.
4. audit_filter_syscall(task, ctx) → may transition to AUDIT_RECORD_CONTEXT if rule matches.

REQ-6: syscall-exit hook flow (`audit_syscall_exit`):
1. ctx := current.audit_context.
2. ctx.return_code = ret; ctx.success = success.
3. If ctx.state == AUDIT_RECORD_CONTEXT:
   - audit_log_exit(ctx) → emit AUDIT_SYSCALL records.
4. ctx.state = AUDIT_BUILD_CONTEXT.

REQ-7: Per-record netlink dispatch:
- Records sent to userspace via NETLINK_AUDIT broadcast.
- Per-record type: AUDIT_SYSCALL / _PATH / _CWD / _ARGV / _CONFIG_CHANGE / _USER / _LOGIN / _SOCKADDR / _AVC / _MAC_*.
- Userspace auditd subscribes; persists to /var/log/audit/audit.log.

REQ-8: audit_buffer construction:
- audit_log_start(ctx, gfp, type) → alloc skbuff with header.
- audit_log_format(ab, fmt, ...) → append fields.
- audit_log_end(ab) → commit (queue for netlink send + free).

REQ-9: Watch (per-path inode watch):
- audit_watch_path is path-string.
- On open(2) of matched path: hook fires; audit-record emitted.
- audit_watch.c uses fsnotify to detect path-target changes (rename, unlink).

REQ-10: Tree (per-subtree watch):
- audit_tree_path is directory; recursive subtree.
- Per-path tracked in tree; new files within subtree audited.
- audit_tree.c uses fsnotify for in-tree-creation events.

REQ-11: Per-task loginuid + sessionid:
- Set at session-establishment (PAM module via /proc/self/loginuid).
- Inherited by child processes.
- Used by rule field AUDIT_LOGINUID for per-session policy.

REQ-12: Backlog limit:
- Per-VM `audit_backlog_limit` (default 64); records exceeding queued in audit_skb_queue.
- Beyond limit: drop or panic per failure-mode config.

REQ-13: Failure modes:
- `audit_failure` ∈ {silent, printk, panic}.
- Configurable via auditctl -f.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `audit_context_no_uaf` | UAF | per-task ctx allocated at task_init; freed at exit. |
| `rule_field_count_bounded` | INVARIANT | per-rule.field_count ≤ AUDIT_MAX_FIELDS (typically 64). |
| `syscall_mask_no_oob` | OOB | per-rule.mask indexed by syscall number ≤ NR_SYSCALLS. |
| `backlog_limit_enforced` | INVARIANT | per-VM audit_skb_queue size ≤ audit_backlog_limit. |
| `loginuid_immutable_after_set` | INVARIANT | per-task loginuid set-once via /proc/self/loginuid; defense against tampering. |

### Layer 2: TLA+

`kernel/audit/audit_state.tla`:
- Per-task ctx state ∈ {Dummy, BuildContext, RecordContext}.
- Transitions per syscall_entry, syscall_exit, filter_syscall.
- Properties:
  - `safety_record_only_after_match` — RecordContext implies a matching rule has action == ALWAYS.
  - `liveness_eventual_emit_or_drop` — every RecordContext eventually emits or drops at backlog limit.

`kernel/audit/netlink_dispatch.tla`:
- States: Pending, Queued, Sent.
- Properties:
  - `safety_in_order_per_serial` — per-task per-syscall records emitted in serial order.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Auditsc::syscall_entry` post: ctx.state ∈ {BuildContext, RecordContext} per filter | `Auditsc::syscall_entry` |
| `Auditsc::syscall_exit` post: ctx.state == BuildContext | `Auditsc::syscall_exit` |
| `AuditLog::end` post: skbuff queued for netlink send | `AuditLog::end` |
| Per-rule mask[syscall_n] checked before field-eval | `AuditFilter::syscall` |
| `AuditWatch::add` post: watch registered with fsnotify; ref held | `AuditWatch::add` |

### Layer 4: Verus/Creusot functional

`Per-syscall: matched audit-rule → audit-record emitted` semantic equivalence: per-syscall the audit-buffer fields match the syscall-context (uid, pid, args, return-code, file-paths) at exit time.

### hardening

(Inherits row-1 features from `kernel/audit/00-overview.md` § Hardening.)

audit-specific reinforcement:

- **Per-rule field-count bound** — defense against malformed rule causing OOB walk.
- **Per-VM backlog limit** — defense against audit-flood causing memory exhaustion.
- **Per-task loginuid immutable-after-set** — defense against process tampering with audit identity.
- **Per-record netlink reply privileged** — defense against unauthorized rule modification.
- **fsnotify integration via inode-mark refcount** — defense against UAF on path-rename + unlink.
- **Per-tree subtree path-recursion bounded** — defense against unbounded depth on pathological hierarchy.
- **Per-LSM-rule context-string validated** — defense against attacker-supplied bogus LSM tag causing crash on match.
- **failure-mode panic-on-loss** option for compliance environments — defense against silent audit-data loss.
- **Per-syscall-entry filter eval bounded** — defense against rule-list iterating per-syscall causing performance collapse.
- **Audit netlink netns-aware** — defense against cross-namespace audit-record leak.
- **Per-record fields sanitized for control-chars** — defense against attacker-input embedding fake-record-boundary in audit log.
- **Per-rule LSM compile-time validate** — defense against LSM rule referencing non-existent label.

