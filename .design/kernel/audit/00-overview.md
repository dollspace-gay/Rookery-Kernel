# Tier-2: kernel/audit — Linux audit (audit + auditd netlink + filter + watch + tree + fsnotify)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - kernel/audit*.c
  - include/linux/audit.h
  - include/uapi/linux/audit.h
-->

## Summary

Tier-2 wrapper for the Linux audit subsystem — kernel-side compliance + intrusion-detection logging required by PCI-DSS, HIPAA, FISMA, Common Criteria. Components: `audit.c` (NETLINK_AUDIT socket impl + per-record format + per-task audit-context + syscall audit hooks invoked from `kernel/seccomp.c` and arch syscall entry), `auditfilter.c` (per-rule filter matching: arch / pid / uid / gid / euid / egid / suid / sgid / fsuid / fsgid / loginuid / pers / arch / msgtype / ppid / comm / exe / subj_user / subj_role / subj_type / subj_sen / subj_clr / saddr_fam / inode / dir / path / fstype / fsmagic / objtype / inode_uid / inode_gid / msgtype operators), `audit_fsnotify.c` (fsnotify integration for path-watch rules), `audit_tree.c` (recursive directory-tree watch — `auditctl -w /etc -p wa`), `audit_watch.c` (per-watch fsnotify mark management), and headers `audit.h`.

Audit consumes from: every `__audit_syscall_entry`/`exit` from arch entry code, every LSM hook for `security_audit_*`, every `audit_log_*` from across kernel.

## Compatibility contract — outline

- NETLINK_AUDIT messages byte-identical: `AUDIT_GET`, `_SET`, `_LIST`, `_LIST_RULES`, `_ADD`, `_DEL`, `_ADD_RULE`, `_DEL_RULE`, `_USER`, `_USER_AVC`, `_USER_TTY`, `_SIGNAL_INFO`, `_GET_FEATURE`, `_SET_FEATURE`, `_FIRST_USER_MSG..AUDIT_LAST_USER_MSG`, `_FIRST_KERN_MSG..AUDIT_LAST_KERN_MSG` ranges (~200 message types).
- Per-record text format `type=NAME msg=audit(<ts>:<seq>): <fields>` byte-identical (auditd + ausearch + aureport consume unchanged).
- Per-rule struct (audit_rule_data) byte-identical.
- Watch + tree rules via `auditctl -w <path> -p <perms>` parse + match identical.
- `/proc/<pid>/loginuid` + `/proc/<pid>/sessionid` byte-identical (loginuid sets at PAM-login time, immutable post-set unless CAP_AUDIT_CONTROL).

## Tier-3 docs governed

| Tier-3 | Scope |
|---|---|
| `kernel/audit/audit-core.md` | `audit.c`: NETLINK_AUDIT socket + per-task audit-context + syscall hooks |
| `kernel/audit/auditfilter.md` | `auditfilter.c`: per-rule filter matching |
| `kernel/audit/audit-fsnotify.md` | `audit_fsnotify.c`: fsnotify integration |
| `kernel/audit/audit-tree.md` | `audit_tree.c`: recursive directory-tree watch |
| `kernel/audit/audit-watch.md` | `audit_watch.c`: per-watch fsnotify mark |

## Compatibility outline / AC / Verification / Hardening

- REQ-O1: NETLINK_AUDIT messages byte-identical (auditd + ausearch + aureport + lib audit consume).
- REQ-O2: Per-rule struct + watch/tree behavior byte-identical (`auditctl -l/-w/-D` works).
- REQ-O3: Per-task audit-context propagated correctly (loginuid + sessionid + selinux-context).
- REQ-O4: TLA+ models (per-task audit context propagation across exec/fork; audit-rule list lock + concurrent rule update + match; audit-buffer overflow rate-limit).
- REQ-O5: AC: `auditd` running, `auditctl -w /etc/passwd -p wa -k passwd_changes` watches; modify file → audit record matches.
- REQ-O6: AC: STIG-compliance audit ruleset loads + boots successfully.
- Hardening: NETLINK_AUDIT requires CAP_AUDIT_CONTROL for write + CAP_AUDIT_READ for read; loginuid immutable post-set; audit-rate-limit enforced (`/proc/sys/kernel/audit_backlog_limit`); audit-buffer drop count exposed for monitoring.

## Out of Scope
- Implementation code; 32-bit-only paths; audit userspace
