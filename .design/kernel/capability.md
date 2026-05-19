# Tier-3: kernel/capability.c — POSIX capabilities (capget/capset/capable)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: kernel/00-overview.md
upstream-paths:
  - kernel/capability.c (~503 lines)
  - include/linux/capability.h (kernel_cap_t, CAP_*, capable() / ns_capable() / *_noaudit)
  - include/uapi/linux/capability.h (_LINUX_CAPABILITY_VERSION_{1,2,3}, struct __user_cap_header_struct, __user_cap_data_struct, CAP_LAST_CAP, CAP_VALID_MASK)
  - include/linux/cred.h (struct cred — cap_effective/permitted/inheritable/bset/ambient)
  - include/uapi/linux/xattr.h (XATTR_NAME_CAPS = "security.capability"; struct vfs_cap_data)
  - include/uapi/linux/prctl.h (PR_SET_NO_NEW_PRIVS, PR_CAP_AMBIENT, PR_CAPBSET_*)
  - security/commoncap.c (file caps, exec transition, vfs_caps_from_xattr; **out-of-scope here** but referenced)
-->

## Summary

POSIX-1003.1e capabilities partition the historical superuser power into ~41 distinct privileges (`CAP_CHOWN`, `CAP_NET_ADMIN`, `CAP_SYS_ADMIN`, `CAP_SYS_PTRACE`, ...; through `CAP_LAST_CAP`). Each task carries five 64-bit sets in `current->cred`: **permitted (P)** = ceiling of what the task may have; **effective (E)** = currently exercisable; **inheritable (I)** = preserved across `execve` per file-caps rules; **bset** (bounding set) = ceiling on P at `execve`; **ambient (A)** = subset of P∩I that is added to P (and E if no file caps) of any program the task `execve`s, so a non-suid binary can inherit privileges from an ambient-set ancestor. `kernel/capability.c` implements the per-task get/set syscalls and the kernel-side checks: `capget(2)` / `capset(2)` (versioned via `_LINUX_CAPABILITY_VERSION_{1,2,3}` headers — v1 = 32-bit single set, v2/v3 = two-32-bit halves to fit 64-bit), `capable(int cap)` (current task, init_user_ns), `ns_capable(ns, cap)` (current task, target user-ns), `has_capability(_noaudit)` (arbitrary task), `capable_wrt_inode_uidgid`, `ptracer_capable`, `file_ns_capable`. The actual enforcement of capability bits at `execve` (file caps via `security.capability` xattr, `vfs_cap_data` parsing, suid emulation, no-new-privs gating) lives in `security/commoncap.c`; this file is solely the syscall surface + kernel-side `cap_*` query helpers that thunk through LSM `security_capable` / `security_capget` / `security_capset`. Critical for: every privileged kernel operation, containers, `systemd-nspawn`/`runc`/`crun`, ambient-cap shell pipelines, sandbox layering.

This Tier-3 covers `kernel/capability.c` (~503 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `file_caps_enabled` | per-boot file-caps switch (`no_file_caps`) | `Capability::file_caps_enabled` |
| `file_caps_disable()` (__setup) | per-boot kernel-arg parser | `Capability::file_caps_disable` |
| `warn_legacy_capability_use()` | per-v1 caller warning | `Capability::warn_legacy` |
| `warn_deprecated_v2()` | per-v2 caller warning | `Capability::warn_v2` |
| `cap_validate_magic()` | per-header version check | `Capability::validate_magic` |
| `cap_get_target_pid()` | per-capget target locate | `Capability::get_target_pid` |
| `SYSCALL_DEFINE2(capget, ...)` | per-capget(2) | `Capability::sys_capget` |
| `mk_kernel_cap()` | per-(low,high) → kernel_cap_t | `Capability::mk_kernel_cap` |
| `SYSCALL_DEFINE2(capset, ...)` | per-capset(2) | `Capability::sys_capset` |
| `has_ns_capability()` | per-task per-ns query (audited) | `Capability::has_ns_capability` |
| `has_ns_capability_noaudit()` | per-task per-ns query (silent) | `Capability::has_ns_capability_noaudit` |
| `has_capability_noaudit()` | per-task init_user_ns query (silent) | `Capability::has_capability_noaudit` |
| `ns_capable_common()` | per-current per-ns inner | `Capability::ns_capable_common` |
| `ns_capable()` | per-current per-ns audited | `Capability::ns_capable` |
| `ns_capable_noaudit()` | per-current per-ns silent | `Capability::ns_capable_noaudit` |
| `ns_capable_setid()` | per-current per-ns + INSETID flag | `Capability::ns_capable_setid` |
| `capable()` | per-current init_user_ns audited | `Capability::capable` |
| `file_ns_capable()` | per-file-opener cap query | `Capability::file_ns_capable` |
| `privileged_wrt_inode_uidgid()` | per-inode uid/gid in ns | `Capability::privileged_wrt_inode_uidgid` |
| `capable_wrt_inode_uidgid()` | per-current cap + inode uid/gid in ns | `Capability::capable_wrt_inode_uidgid` |
| `ptracer_capable()` | per-tracer CAP_SYS_PTRACE in ns | `Capability::ptracer_capable` |

## Compatibility contract

REQ-1: kernel_cap_t internal representation:
- `struct kernel_cap_t { u64 val; }` — 64-bit single word holding bits `[0 .. CAP_LAST_CAP]` (CAP_LAST_CAP currently 40).
- `CAP_VALID_MASK = (1ULL << (CAP_LAST_CAP + 1)) - 1` — bits outside masked off by `mk_kernel_cap`.

REQ-2: UAPI versioning (`struct __user_cap_header_struct`):
- Fields: `__u32 version; int pid;` (pid=0 means caller).
- Supported versions:
  - `_LINUX_CAPABILITY_VERSION_1` (0x19980330): `_LINUX_CAPABILITY_U32S_1 = 1` (single u32; legacy 32-bit set).
  - `_LINUX_CAPABILITY_VERSION_2` (0x20071026): `_LINUX_CAPABILITY_U32S_3 = 2` (two u32 halves; deprecated v2 header — emits one-shot warning).
  - `_LINUX_CAPABILITY_VERSION_3` (0x20080522): same wire layout as v2; current `_KERNEL_CAPABILITY_VERSION`.

REQ-3: cap_validate_magic(header, *tocopy) -> i32:
- `get_user(version, &header->version)` (-EFAULT on fault).
- switch version:
  - v1 → `warn_legacy_capability_use`; `*tocopy = U32S_1` (=1).
  - v2 → `warn_deprecated_v2`; fallthrough to v3.
  - v3 → `*tocopy = U32S_3` (=2).
  - default → `put_user((u32)_KERNEL_CAPABILITY_VERSION, &header->version)` (informs caller of current version) + -EINVAL.
- return 0.

REQ-4: `SYSCALL_DEFINE2(capget, header, dataptr)`:
- `cap_validate_magic(header, &tocopy)`; if `dataptr == NULL` ∧ ret == -EINVAL → return 0 (probe pattern: app calls with NULL data to discover version). Else if ret → return ret.
- `get_user(pid, &header->pid)` → -EFAULT.
- pid < 0 → -EINVAL.
- `cap_get_target_pid(pid, &pE, &pI, &pP)`:
  - if pid != 0 ∧ pid != `task_pid_vnr(current)`: rcu_read_lock; target = `find_task_by_vpid(pid)`; if !target → -ESRCH; else `security_capget(target, &pE, &pI, &pP)`; rcu_read_unlock.
  - else `security_capget(current, &pE, &pI, &pP)`.
- Split into kdata[0]/kdata[1] (low/high u32 halves of each of effective/permitted/inheritable).
- `copy_to_user(dataptr, kdata, tocopy * sizeof(kdata[0]))` (silently drops upper bits if `tocopy < U32S_3`).
- return 0.

REQ-5: mk_kernel_cap(low, high) -> kernel_cap_t:
- `kernel_cap_t { val = (low | ((u64)high << 32)) & CAP_VALID_MASK }`.
- Bits above CAP_LAST_CAP are silently dropped.

REQ-6: `SYSCALL_DEFINE2(capset, header, data)`:
- `cap_validate_magic(header, &tocopy)`; ret → return ret.
- `get_user(pid, &header->pid)` → -EFAULT.
- pid != 0 ∧ pid != `task_pid_vnr(current)` → -EPERM (no cross-process set since linux historical deprecation).
- `copybytes = tocopy * sizeof(__user_cap_data_struct)`; if `copybytes > sizeof(kdata)` → -EFAULT.
- `copy_from_user(kdata, data, copybytes)` → -EFAULT.
- (effective, permitted, inheritable) = three `mk_kernel_cap(kdata[0].x, kdata[1].x)` triples.
- `new = prepare_creds()`; if !new → -ENOMEM.
- `security_capset(new, current_cred(), &effective, &inheritable, &permitted)` → on error `abort_creds(new)`; return ret.
- `audit_log_capset(new, current_cred())`.
- `commit_creds(new)`.

REQ-7: capset(2) per-set rules (enforced inside `security_capset` via `cap_capset` in commoncap.c, summarised for clarity):
- New I ⊆ old I ∪ old P (any raised inheritable must come from old permitted).
- New P ⊆ old P (can never raise permitted).
- New E ⊆ new P (effective subset of permitted post-update).
- bset is unaffected; only `prctl(PR_CAPBSET_DROP, cap)` may shrink it (out-of-scope for this file).
- ambient is unaffected; only `prctl(PR_CAP_AMBIENT, ...)` may modify it (out-of-scope).
- file_caps_enabled == 0 disables file-cap pickup at exec (out-of-scope), but capset rules themselves still apply.

REQ-8: capable / ns_capable / has_capability semantics:
- Five entry shapes:
  - `capable(cap)` — current task in `&init_user_ns`, audited.
  - `ns_capable(ns, cap)` — current task in `ns`, audited, sets `PF_SUPERPRIV` on success.
  - `ns_capable_noaudit(ns, cap)` — same, silent.
  - `ns_capable_setid(ns, cap)` — same as `ns_capable`, with `CAP_OPT_INSETID` flag (signals to LSM the check is inside a setid/setgroups syscall).
  - `has_ns_capability[_noaudit](task, ns, cap)` — arbitrary task; does **not** set `PF_SUPERPRIV`.
  - `has_capability_noaudit(task, cap)` — arbitrary task in init_user_ns; silent; exported.
- All ultimately call `security_capable(cred, ns, cap, opts)` returning 0 iff allowed.

REQ-9: ns_capable_common(ns, cap, opts):
- `cap_valid(cap)` (i.e. `0 ≤ cap ≤ CAP_LAST_CAP`); else `pr_crit("capable() called with invalid cap=%u\n", cap); BUG();` (kernel bug — caller mis-spelled / used out-of-range constant).
- `security_capable(current_cred(), ns, cap, opts)`.
- on 0 → `current->flags |= PF_SUPERPRIV` (marks the task as having used a superuser-class privilege, surfaced via `/proc/$pid/status`); return true.
- else return false.

REQ-10: has_ns_capability(t, ns, cap):
- rcu_read_lock; ret = `security_capable(__task_cred(t), ns, cap, CAP_OPT_NONE)`; rcu_read_unlock; return ret == 0.

REQ-11: has_ns_capability_noaudit(t, ns, cap):
- rcu_read_lock; ret = `security_capable(__task_cred(t), ns, cap, CAP_OPT_NOAUDIT)`; rcu_read_unlock; return ret == 0.

REQ-12: has_capability_noaudit(t, cap):
- `has_ns_capability_noaudit(t, &init_user_ns, cap)`.
- Exported as `EXPORT_SYMBOL(has_capability_noaudit)`.

REQ-13: ns_capable(ns, cap):
- `ns_capable_common(ns, cap, CAP_OPT_NONE)`.
- Exported as `EXPORT_SYMBOL(ns_capable)`.

REQ-14: ns_capable_noaudit(ns, cap):
- `ns_capable_common(ns, cap, CAP_OPT_NOAUDIT)`.
- Exported as `EXPORT_SYMBOL(ns_capable_noaudit)`.

REQ-15: ns_capable_setid(ns, cap):
- `ns_capable_common(ns, cap, CAP_OPT_INSETID)`.
- Exported as `EXPORT_SYMBOL(ns_capable_setid)`.

REQ-16: capable(cap):
- `ns_capable(&init_user_ns, cap)`.
- Exported as `EXPORT_SYMBOL(capable)`.
- Compiled only when CONFIG_MULTIUSER.

REQ-17: file_ns_capable(file, ns, cap):
- WARN_ON_ONCE(!cap_valid(cap)) → false.
- `security_capable(file->f_cred, ns, cap, CAP_OPT_NONE) == 0` → true; else false.
- Does **not** set `PF_SUPERPRIV` (caller may not actually be the one exercising the privilege).
- Exported as `EXPORT_SYMBOL(file_ns_capable)`.

REQ-18: privileged_wrt_inode_uidgid(ns, idmap, inode):
- `vfsuid_has_mapping(ns, i_uid_into_vfsuid(idmap, inode))` ∧ `vfsgid_has_mapping(ns, i_gid_into_vfsgid(idmap, inode))`.

REQ-19: capable_wrt_inode_uidgid(idmap, inode, cap):
- `ns = current_user_ns()`.
- `ns_capable(ns, cap) ∧ privileged_wrt_inode_uidgid(ns, idmap, inode)`.
- Exported as `EXPORT_SYMBOL(capable_wrt_inode_uidgid)`.

REQ-20: ptracer_capable(tsk, ns):
- rcu_read_lock; `cred = rcu_dereference(tsk->ptracer_cred)`; if cred → ret = `security_capable(cred, ns, CAP_SYS_PTRACE, CAP_OPT_NOAUDIT)`; else ret = 0 (no tracer ⇒ no extra restriction).
- rcu_read_unlock; return ret == 0.

REQ-21: file_caps_enabled boot-time toggle:
- Default `int file_caps_enabled = 1;`.
- `__setup("no_file_caps", file_caps_disable)` sets to 0; consumers in `security/commoncap.c` (out-of-scope here) gate file-cap pickup on this flag.

REQ-22: cred-snapshot semantics:
- Capability checks always read the **current** `task->cred` (RCU-protected for cross-task `has_ns_capability*`; direct `current_cred()` for self).
- `capset` always commits via `prepare_creds()` → `security_capset` → `commit_creds()` — never mutates in place.
- After `commit_creds`, all subsequent `capable()` checks observe the new sets.

REQ-23: CONFIG_MULTIUSER:
- When disabled (constrained embedded build), `has_capability_noaudit`, `ns_capable*`, `capable`, `ns_capable_setid` are stubbed out by the `#endif /* CONFIG_MULTIUSER */` block ending after `capable()`; the remaining helpers (`file_ns_capable`, `privileged_wrt_inode_uidgid`, `capable_wrt_inode_uidgid`, `ptracer_capable`) remain unconditionally compiled.
- `SYSCALL_DEFINE2(capget,...)` / `capset` are always compiled (so userspace probing the version still works).

REQ-24: Probing pattern (legacy libcap discovery):
- App calls `capget({version=garbage, pid=0}, NULL)`:
  - `cap_validate_magic` writes `_KERNEL_CAPABILITY_VERSION` back into `header->version` and returns -EINVAL.
  - syscall body sees `(dataptr == NULL) && (ret == -EINVAL)` → returns 0.
- App now knows the kernel's preferred version and retries with the correct header.

REQ-25: CAP_OPT_* flag values passed to LSM `security_capable`:
- `CAP_OPT_NONE` = 0 — normal audited check.
- `CAP_OPT_NOAUDIT` — suppress audit message.
- `CAP_OPT_INSETID` — caller is inside `set*uid` / `set*gid` / `setgroups` syscall (allows LSMs like SELinux to apply finer-grained policy without rejecting legitimate setid transitions).

## Acceptance Criteria

- [ ] AC-1: `capget({version=_LINUX_CAPABILITY_VERSION_3, pid=0}, &data)` returns 0 and `data[0..1]` holds the low/high halves of current's E/P/I sets exactly matching `current_cred()`.
- [ ] AC-2: `capget({version=garbage, pid=0}, NULL)` returns 0 after writing `_KERNEL_CAPABILITY_VERSION` into `header->version`.
- [ ] AC-3: `capget({version=_LINUX_CAPABILITY_VERSION_1, pid=0}, &data1)` returns the low 32 bits of E/P/I (upper bits silently dropped), with a one-shot warning printed.
- [ ] AC-4: `capget` with pid != 0 ∧ pid != current's vpid returns the target's caps via `find_task_by_vpid` + `security_capget`; missing pid → -ESRCH.
- [ ] AC-5: `capset({pid=other_pid, ...}, &data)` returns -EPERM (no cross-process set).
- [ ] AC-6: `capset` raising a bit in inheritable that was not in old permitted → -EPERM via `security_capset` ∘ `cap_capset`.
- [ ] AC-7: `capset` raising a bit in permitted → -EPERM.
- [ ] AC-8: `capset` with effective ⊄ new permitted → -EPERM.
- [ ] AC-9: `capable(CAP_NET_ADMIN)` from a task with the bit set in effective for init_user_ns returns true and sets `PF_SUPERPRIV`.
- [ ] AC-10: `capable(out_of_range)` (e.g. `CAP_LAST_CAP+1`) calls `pr_crit` + `BUG()`.
- [ ] AC-11: `ns_capable_noaudit(ns, cap)` and `has_ns_capability_noaudit(t, ns, cap)` do not emit an audit record on denial.
- [ ] AC-12: `file_ns_capable(file, ns, cap)` returns true iff the cred snapshot at file open had `cap` in `ns`; does **not** set `PF_SUPERPRIV` on current.
- [ ] AC-13: `ptracer_capable(tsk, ns)`: returns true if no tracer; otherwise iff `tsk->ptracer_cred` had `CAP_SYS_PTRACE` in `ns`.
- [ ] AC-14: `capable_wrt_inode_uidgid(idmap, inode, cap)` requires both `ns_capable(current_user_ns(), cap)` and inode uid/gid mapped into `current_user_ns()`.
- [ ] AC-15: Boot with `no_file_caps` cmdline sets `file_caps_enabled = 0`; capability syscalls still function (only file-cap pickup at exec is disabled).

## Architecture

```
struct KernelCap {                      // kernel_cap_t
  val: u64,                             // bits [0..CAP_LAST_CAP]; masked by CAP_VALID_MASK
}

struct UserCapHeader {                  // UAPI __user_cap_header_struct
  version: u32,                         // _LINUX_CAPABILITY_VERSION_{1,2,3}
  pid: i32,                             // 0 = current
}

struct UserCapData {                    // UAPI __user_cap_data_struct
  effective: u32,
  permitted: u32,
  inheritable: u32,
}

struct CredCaps {                       // embedded in struct cred
  cap_effective: KernelCap,             // E — currently exercisable
  cap_permitted: KernelCap,             // P — ceiling on E
  cap_inheritable: KernelCap,           // I — preserved across exec per file caps
  cap_bset: KernelCap,                  // bounding set — ceiling on P at exec
  cap_ambient: KernelCap,               // A — added to P (and E w/o file caps) at exec
}

enum CapOpt {                           // bits passed to security_capable
  None = 0,
  NoAudit = 1 << 1,
  InSetid = 1 << 2,
}
```

`Capability::sys_capget(header, dataptr) -> i32`:
1. ret = `validate_magic(header, &tocopy)`.
2. if dataptr == NULL ∨ ret != 0:
   - if dataptr == NULL ∧ ret == -EINVAL → return 0 (probe).
   - else return ret.
3. `get_user(pid, &header.pid)` → -EFAULT.
4. pid < 0 → -EINVAL.
5. ret = `get_target_pid(pid, &pE, &pI, &pP)`; ret → return ret.
6. Split each KernelCap into two u32:
   - kdata[0].effective = pE.val as u32; kdata[1].effective = (pE.val >> 32) as u32.
   - same for permitted (pP) and inheritable (pI).
7. `copy_to_user(dataptr, kdata, tocopy * sizeof(UserCapData))` → -EFAULT.
8. return 0.

`Capability::get_target_pid(pid, &pE, &pI, &pP) -> i32`:
1. if pid != 0 ∧ pid != `task_pid_vnr(current)`:
   - rcu_read_lock.
   - target = `find_task_by_vpid(pid)`.
   - if !target → ret = -ESRCH.
   - else ret = `security_capget(target, &pE, &pI, &pP)`.
   - rcu_read_unlock.
2. else ret = `security_capget(current, &pE, &pI, &pP)`.
3. return ret.

`Capability::sys_capset(header, data) -> i32`:
1. ret = `validate_magic(header, &tocopy)`; ret → return ret.
2. `get_user(pid, &header.pid)` → -EFAULT.
3. pid != 0 ∧ pid != `task_pid_vnr(current)` → -EPERM.
4. copybytes = tocopy * sizeof(UserCapData); copybytes > sizeof(kdata) → -EFAULT.
5. `copy_from_user(&kdata, data, copybytes)` → -EFAULT.
6. effective = `mk_kernel_cap(kdata[0].effective, kdata[1].effective)`.
7. permitted = `mk_kernel_cap(kdata[0].permitted, kdata[1].permitted)`.
8. inheritable = `mk_kernel_cap(kdata[0].inheritable, kdata[1].inheritable)`.
9. new = `prepare_creds()`; !new → -ENOMEM.
10. ret = `security_capset(new, current_cred(), &effective, &inheritable, &permitted)`.
11. ret < 0 → `abort_creds(new)`; return ret.
12. `audit_log_capset(new, current_cred())`.
13. return `commit_creds(new)`.

`Capability::ns_capable_common(ns, cap, opts) -> bool`:
1. !`cap_valid(cap)` → `pr_crit + BUG()`.
2. capable = `security_capable(current_cred(), ns, cap, opts)`.
3. if capable == 0 → `current.flags |= PF_SUPERPRIV`; return true.
4. else return false.

`Capability::has_ns_capability_inner(t, ns, cap, opts) -> bool`:
1. rcu_read_lock.
2. ret = `security_capable(__task_cred(t), ns, cap, opts)`.
3. rcu_read_unlock.
4. return ret == 0.

`Capability::file_ns_capable(file, ns, cap) -> bool`:
1. WARN_ON_ONCE(!cap_valid(cap)) → false.
2. return `security_capable(file.f_cred, ns, cap, CAP_OPT_NONE) == 0`.
3. /* Note: PF_SUPERPRIV NOT set. */

`Capability::capable_wrt_inode_uidgid(idmap, inode, cap) -> bool`:
1. ns = `current_user_ns()`.
2. return `ns_capable(ns, cap) ∧ privileged_wrt_inode_uidgid(ns, idmap, inode)`.

`Capability::ptracer_capable(tsk, ns) -> bool`:
1. ret = 0.  /* absent tracer ⇒ no restriction */
2. rcu_read_lock.
3. cred = `rcu_dereference(tsk.ptracer_cred)`.
4. if cred → ret = `security_capable(cred, ns, CAP_SYS_PTRACE, CAP_OPT_NOAUDIT)`.
5. rcu_read_unlock.
6. return ret == 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `magic_versions_only` | INVARIANT | per-`validate_magic`: version ∈ {v1, v2, v3} ⇒ ret = 0 ∧ tocopy ∈ {1, 2}; else version field rewritten + -EINVAL. |
| `caps_masked` | INVARIANT | per-`mk_kernel_cap`: result.val & ~CAP_VALID_MASK == 0. |
| `capset_self_only` | INVARIANT | per-`sys_capset`: pid != 0 ∧ pid != vpid(current) ⇒ -EPERM before any copy. |
| `cred_commit_atomic` | INVARIANT | per-`sys_capset`: success path is exactly `prepare_creds → security_capset → audit_log_capset → commit_creds`; failure ⇒ `abort_creds`. |
| `cap_valid_or_bug` | INVARIANT | per-`ns_capable_common`: !cap_valid(cap) ⇒ pr_crit + BUG (no silent allow). |
| `pf_superpriv_on_success` | INVARIANT | per-`ns_capable_common`: success ⇒ current.flags |= PF_SUPERPRIV; failure ⇒ unchanged. |
| `file_ns_capable_no_pf_superpriv` | INVARIANT | per-`file_ns_capable`: current.flags unchanged regardless of result. |
| `has_capability_rcu_protected` | INVARIANT | per-`has_ns_capability*`: `__task_cred(t)` accessed under rcu_read_lock. |
| `ptracer_capable_absent_tracer_true` | INVARIANT | per-`ptracer_capable`: `tsk.ptracer_cred == NULL` ⇒ true. |
| `copy_size_bounded` | INVARIANT | per-`sys_capset`: `copybytes = tocopy * sizeof(UserCapData) ≤ sizeof(kdata)`. |

### Layer 2: TLA+

`kernel/capability.tla`:
- Per-capset (P/I/E transition) + per-capget + per-capable + per-LSM hook outcome.
- Properties:
  - `safety_permitted_monotone_decreasing` — per-capset: new_P ⊆ old_P always (no raise).
  - `safety_inheritable_subset_old_PI` — per-capset: new_I ⊆ old_I ∪ old_P.
  - `safety_effective_subset_new_permitted` — per-capset post: new_E ⊆ new_P.
  - `safety_bset_unchanged_by_capset` — per-capset: bset and ambient unchanged.
  - `safety_capable_in_init_ns` — per-`capable(cap)`: equivalent to `ns_capable(init_user_ns, cap)`.
  - `safety_cap_check_uses_target_ns_cred` — per-`has_ns_capability*`: cred is `__task_cred(t)` not `current_cred()`.
  - `liveness_per_capset_terminates` — per-`sys_capset`: returns in bounded steps regardless of cred-prep outcome.
  - `liveness_per_capget_target_eventually_resolved` — per-`get_target_pid`: pid lookup terminates (find_task_by_vpid is finite-time).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Capability::sys_capget` post: ret ∈ {0, -EFAULT, -EINVAL, -ESRCH} | `Capability::sys_capget` |
| `Capability::sys_capset` post: ret ∈ {0, -EFAULT, -EINVAL, -EPERM, -ENOMEM, security_capset return} | `Capability::sys_capset` |
| `Capability::sys_capset` invariant: failure path always reaches `abort_creds(new)` | `Capability::sys_capset` |
| `Capability::sys_capset` post: success ⇒ `commit_creds(new)` executed and audited via `audit_log_capset` | `Capability::sys_capset` |
| `Capability::ns_capable_common` post: success ⇔ `security_capable(current_cred(), ns, cap, opts) == 0` ⇒ PF_SUPERPRIV set | `Capability::ns_capable_common` |
| `Capability::has_ns_capability_*` post: result ⇔ `security_capable(__task_cred(t), ns, cap, opts) == 0` | `Capability::has_ns_capability` |
| `Capability::file_ns_capable` post: result ⇔ `security_capable(file.f_cred, ns, cap, NONE) == 0`; no PF_SUPERPRIV mutation | `Capability::file_ns_capable` |
| `Capability::ptracer_capable` post: NULL tracer ⇒ true; otherwise iff `security_capable(ptracer_cred, ns, CAP_SYS_PTRACE, NOAUDIT) == 0` | `Capability::ptracer_capable` |

### Layer 4: Verus/Creusot functional

`Per-capset: validate_magic → copy_from_user → mk_kernel_cap × 3 → prepare_creds → security_capset → audit_log_capset → commit_creds` semantic equivalence: per-`man 2 capset`, per-`man 7 capabilities`, and `tools/testing/selftests/capabilities/test_execve.c` reference behavior.

`Per-`ns_capable_common` ∘ `security_capable` ⇒ PF_SUPERPRIV bookkeeping` semantic equivalence: per-`/proc/$pid/status` `CapEff/CapPrm/CapInh/CapBnd/CapAmb` exposure (handled in `fs/proc/array.c` which reads these sets).

## Hardening

(Inherits row-1 features from `kernel/00-overview.md` § Hardening.)

Capability reinforcement:

- **Per-cap masking via `CAP_VALID_MASK` in `mk_kernel_cap`** — defense against per-undefined-bits in user input.
- **Per-`cap_valid(cap)` check + BUG()** — defense against per-call-site typo silently allowing.
- **Per-`prepare_creds` / `commit_creds` atomic transition** — defense against per-mid-update inconsistent cap view.
- **Per-`security_capset` LSM hook gating** — defense against per-policy bypass (SELinux, AppArmor, Smack).
- **Per-`audit_log_capset` mandatory on success** — defense against per-stealth privilege change.
- **Per-`capset` self-only (-EPERM on cross-pid)** — defense against per-historical setuid-everyone abuse.
- **Per-`I ⊆ old I ∪ old P` rule (in `cap_capset`)** — defense against per-inheritable laundering.
- **Per-`P` monotone-decreasing rule** — defense against per-self-uplift.
- **Per-RCU-protected `__task_cred(t)` in cross-task queries** — defense against per-cred-UAF.
- **Per-`PF_SUPERPRIV` exposure via /proc** — defense (observability) against per-silent-privilege-usage.
- **Per-`CAP_OPT_NOAUDIT` reserved to inner kernel queries (`ns_capable_noaudit`, `has_*_noaudit`, `ptracer_capable`)** — defense against per-audit-log flood.
- **Per-`CAP_OPT_INSETID` plumbing** — defense against per-LSM mis-deny on legitimate setid transitions.
- **Per-`file_ns_capable` skips `PF_SUPERPRIV` set** — defense against per-cred-of-record/cred-of-use confusion.
- **Per-`ptracer_capable` `NOAUDIT` + ptracer-cred snapshot** — defense against per-tracer-cred-uplift between attach and access.
- **Per-`no_file_caps` boot option** — defense (operator escape hatch) against per-file-cap-misconfiguration on legacy installs.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — `capget` / `capset` copy `__user_cap_data_struct` arrays of `_LINUX_CAPABILITY_U32S_3 * 2` u32s; whitelist the exact per-version size (V1=1, V2=2, V3=2) so an attacker cannot use a forged `header.version` to drag adjacent task_struct fields across the user boundary.
- **PAX_KERNEXEC** — `security_capable` / `security_capget` / `security_capset` LSM hook dispatch via `call_int_hook` is indirect; enforce W^X on the LSM hook list page so a kernel-write primitive cannot rewrite the hook chain to skip the commoncap gate.
- **PAX_RANDKSTACK** — `cap_*` checks fire from every privileged syscall entry at attacker-controlled depth; randomized kstack offset disrupts ROP through the giant `ns_capable` → `security_capable` → `cap_capable` chain.
- **PAX_REFCOUNT** — `struct cred->usage` and `struct user_namespace->ns.count` are refcount_t with saturating overflow; defense against fd-storm + setuid-loop underflow racing `commit_creds` against `__put_cred`.
- **PAX_MEMORY_SANITIZE** — `put_cred_rcu` callback must zero the `struct cred` (cap_inheritable/permitted/effective/bset/ambient bitfields) before slab return; defense against UAF residual exposing a freed task's full capability set to a new cred allocation.
- **PAX_UDEREF** — `capget`/`capset` user pointer dereferences must engage SMAP/PAN across the whole `cap_validate_magic` → `copy_to/from_user` window; never enable AC for more than the single copy.
- **PAX_RAP/kCFI** — LSM `capable` / `capget` / `capset` hooks are indirect-called from `security_*` wrappers; require matching kCFI tag so a corrupted LSM hook list cannot route capability checks through an attacker-chosen stub.
- **GRKERNSEC_HIDESYM** — `/proc/<pid>/status` `Cap*:` lines and `getauxval(AT_SECURE)` must not leak kernel pointers; existing capability bitfields are user-facing but the surrounding cred struct address must stay hidden from non-CAP_SYSLOG readers.
- **GRKERNSEC_DMESG** — `cap_validate_magic`'s `-EINVAL` path on bad version must not splat raw user pointers or kernel cred addresses into dmesg readable by non-CAP_SYSLOG.
- **CAP_SETPCAP strict** — `cap_capset` must refuse expansion of `permitted` outside the caller's own bset even when CAP_SETPCAP is held; Rookery enforces upstream's "CAP_SETPCAP allows setting the bset of children, not raising the caller's own permitted set" rule with no exceptions.
- **`file_caps_enabled` boot toggle** — `no_file_caps` cmdline must hard-disable `get_vfs_caps_from_disk` and force the setuid-root pre-fcaps semantics for legacy mounts; defense against per-mount fcap parsing on untrusted media.
- **`no_new_privs` propagation** — `PR_SET_NO_NEW_PRIVS` must be enforced through `cap_bprm_creds_from_file` so an exec into a setuid/fcap binary cannot regain caps even with valid xattr; `nnp` flag is inherited unconditionally to children.
- **`ptracer_capable` cred snapshot** — `ptracer_capable` already snapshots tracer creds; harden by requiring `same-user-ns OR CAP_SYS_PTRACE-in-init-ns` to defeat cred-uplift between attach and access.
- **Rationale** — capabilities are the kernel's primary post-uid privilege model; a single bypass (cred UAF, hook list corruption, bset confusion) collapses the whole gate. The grsec regime forces an attacker to defeat refcount saturation + LSM kCFI + cred sanitize + nnp propagation + setpcap strict-check simultaneously before reaching arbitrary cap uplift.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- File capability xattr parsing and `cap_inode_getsecurity` / `cap_inode_killpriv` (covered in `fs/xattr` + `security/commoncap.c` Tier-3)
- `vfs_cap_data` struct layout and `XATTR_NAME_CAPS` ("security.capability") wire format (covered in commoncap Tier-3)
- `execve` cap transitions (P', I', E', A' calculation; suid emulation) — `cap_bprm_creds_from_file` in `security/commoncap.c`
- `prctl(PR_CAPBSET_*)`, `prctl(PR_CAP_AMBIENT, ...)`, `prctl(PR_SET_KEEPCAPS, ...)`, `prctl(PR_SET_NO_NEW_PRIVS, ...)` — covered in `sys.md` Tier-3
- LSM `security_capable` / `security_capget` / `security_capset` plumbing — covered in `security/security.c` Tier-3
- User-namespace creation and cap-uplift inside child ns — covered in `kernel/user_namespace.c` Tier-3
- `/proc/$pid/status` cap exposure — covered in `fs/proc/array.c` Tier-3
- Implementation code
