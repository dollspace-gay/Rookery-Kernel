# Tier-5: syscall 308 — setns(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/nsproxy.c (sys_setns, ns_install)
  - fs/nsfs.c (ns_get_path, ns_get_owner)
  - include/uapi/linux/sched.h (CLONE_NEW*)
-->

## Summary

`setns(2)` attaches the calling task to an existing namespace referenced by `fd`. The fd is an `nsfs` handle (obtained via `open("/proc/<pid>/ns/<kind>")`, `pidfd_open`, or `pidfd_getfd`). Per-`nstype = 0` accepts any namespace kind; nonzero `nstype` must match the fd's kind. Per-CAP_SYS_ADMIN required in the target namespace's owning user_ns (with userns nuances). Per-CLONE_NEWPID changes only the namespace of subsequent children (not the current task's PID). Critical for: `nsenter`, container exec, sidecar agents joining a workload's namespaces.

This Tier-5 covers `syscall 308 setns`.

## Signature

```c
int setns(int fd, int nstype);
```

Per-x86_64-syscall-table: `__NR_setns = 308`. Per-glibc: `sched.h`. Per-vDSO: not vDSO-vectored.

## Parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `fd` | `int` | in | nsfs file descriptor; OR pidfd (since 5.8) — uses target task's nsproxy. |
| `nstype` | `int` | in | 0 (accept any) or one of CLONE_NEWNS, CLONE_NEWUTS, CLONE_NEWIPC, CLONE_NEWUSER, CLONE_NEWPID, CLONE_NEWNET, CLONE_NEWCGROUP, CLONE_NEWTIME. With a pidfd: bitmask of namespaces to join. |

## Return

- 0 on success.
- -1 with `errno` on failure.

## Errors

| errno | Cause |
|---|---|
| EBADF | `fd` invalid. |
| EINVAL | `fd` not an nsfs/pidfd file; `nstype` does not match fd's kind; `nstype` has unsupported bits with pidfd; attempt to setns CLONE_NEWPID into ancestor/non-descendant pidns; or attempt by multi-threaded task to enter NEWUSER/NEWMNT. |
| EPERM | Lacks CAP_SYS_ADMIN in target namespace's owning userns; or fd's userns is not equal/ancestor of caller's; or task is shared (CLONE_THREAD) for restricted kinds. |
| ENOMEM | Kernel allocation failed. |
| ESRCH | pidfd: target task no longer exists. |
| EUCLEAN | nsfs ref dead (rare). |

## ABI surface

- Syscall number: `__NR_setns = 308`.
- Two flavors:
  - **nsfs fd**: fd from /proc/<pid>/ns/<kind> — single namespace; nstype ∈ {0, CLONE_NEW<KIND>}.
  - **pidfd**: fd from pidfd_open / pidfd_getfd — joins target task's namespaces; nstype is a CLONE_NEW* bitmask (since 5.8).
- nstype == 0 means "infer from fd" (single-ns variant only).
- pidfd variant with nstype == 0 joins **all** namespaces of target.

## Compatibility contract

REQ-1: fd validation:
- fget(fd) returns file*.
- file->f_op == &ns_file_operations  (nsfs)  OR
- file->f_op == &pidfd_fops          (pidfd)
- Otherwise EINVAL.

REQ-2: Per-nsfs variant:
- ns = file->private_data = struct ns_common.
- If nstype != 0 and nstype != ns->ops->type: EINVAL.
- Per-ns->ops->install(nsproxy, ns) hook performs the kind-specific install.

REQ-3: Per-pidfd variant (since 5.8):
- target = pidfd_get_task(file).
- target_nsproxy = read snapshot of target.nsproxy under rcu.
- For each CLONE_NEW* bit in nstype: install corresponding ns from target.

REQ-4: Per-CAP_SYS_ADMIN:
- ns_capable(target_ns.user_ns, CAP_SYS_ADMIN) required for non-userns kinds.
- Per-CLONE_NEWUSER: caller's user_ns must equal or be a parent of target user_ns; otherwise EPERM.

REQ-5: Per-CLONE_NEWPID:
- Cannot setns into ancestor pidns of caller's current pidns: EINVAL.
- Cannot setns into a non-descendant: EINVAL.
- Effects: subsequent children created via fork enter the new pidns; current task's pid unchanged in its own pidns.

REQ-6: Per-CLONE_NEWUSER:
- Task must be single-threaded (atomic_read(&current->signal->live) == 1): EINVAL otherwise.
- Replaces task.cred.user_ns with the new userns; capability set adjusted per-target-userns membership.

REQ-7: Per-CLONE_NEWNS:
- Task must be single-threaded (CLONE_FS sharing): EINVAL otherwise.
- Cannot enter mnt_ns that is being destroyed.

REQ-8: Per-CLONE_NEWCGROUP / CLONE_NEWUTS / CLONE_NEWIPC / CLONE_NEWNET / CLONE_NEWTIME:
- May be multi-threaded.
- Replaces corresponding namespace pointer in nsproxy.

REQ-9: Per-nsproxy copy-on-write:
- Replace current.nsproxy with a private copy containing the new ns(es); release old refs.
- switch_task_namespaces atomic.

REQ-10: Per-pidfd-with-nstype validation:
- nstype & ~(NS_VALID_MASK) ⟹ EINVAL.
- NS_VALID_MASK = CLONE_NEWNS|UTS|IPC|USER|PID|NET|CGROUP|TIME.

REQ-11: Per-userns-stacking:
- Joining a userns deeper than MAX_USER_NAMESPACE_LEVEL would not occur because the userns already exists; level is fixed.

REQ-12: Per-time-namespace:
- After setns CLONE_NEWTIME: subsequent clock_gettime(CLOCK_MONOTONIC) sees the new offsets.

REQ-13: Per-target-namespace-must-be-alive:
- ns->ops->install fails if ns is zombie (e.g. pidns of dying init).

REQ-14: Per-failure-atomic:
- Either all requested namespaces joined or none. Failure rolls back partial installs.

## Acceptance Criteria

- [ ] AC-1: open("/proc/123/ns/net"); setns(fd, CLONE_NEWNET): joins target net-ns.
- [ ] AC-2: setns(fd, 0) with single-kind nsfs fd: joins that ns.
- [ ] AC-3: setns(fd, CLONE_NEWUTS) where fd is IPC ns: EINVAL.
- [ ] AC-4: pidfd_open(123); setns(pidfd, CLONE_NEWNET|CLONE_NEWPID): joins both.
- [ ] AC-5: setns(pidfd, 0): joins all of target's namespaces.
- [ ] AC-6: Multi-threaded task setns CLONE_NEWUSER: EINVAL.
- [ ] AC-7: Multi-threaded task setns CLONE_NEWNS: EINVAL.
- [ ] AC-8: setns into ancestor pidns: EINVAL.
- [ ] AC-9: Non-CAP_SYS_ADMIN: EPERM.
- [ ] AC-10: setns CLONE_NEWUSER then capability set reflects new userns.
- [ ] AC-11: setns CLONE_NEWPID + fork(): child has pid 1 (or next-pid) in new pidns.
- [ ] AC-12: nstype with unsupported bit (pidfd): EINVAL.
- [ ] AC-13: pidfd to dead task: ESRCH.

## Architecture

```
Setns::sys_setns(fd: i32, nstype: i32) -> Result<i32, Errno>
```

1. let file = fget(fd).ok_or(EBADF)?;
2. let result = if is_pidfd(&file) {
     setns_pidfd(&file, nstype as u32)
   } else if is_nsfs(&file) {
     setns_single(&file, nstype as u32)
   } else {
     Err(EINVAL)
   };
3. fput(file).
4. result.
```

`setns_single(file, nstype)`:
1. let ns = file.private_data::<NsCommon>();
2. if nstype != 0 && nstype != ns.ops.kind_flag(): return Err(EINVAL).
3. /* Capability + relation checks */
4. ns.ops.may_install(current(), &ns)?;          // per-kind: cap + relation
5. /* Threaded-task restriction */
6. if ns.ops.requires_single_threaded() && !is_single_threaded(current()): return Err(EINVAL).
7. /* Build new nsproxy */
8. let new_nsproxy = clone_nsproxy(current().nsproxy);
9. ns.ops.install(&mut new_nsproxy, ns)?;
10. /* Atomic install */
11. switch_task_namespaces(current(), new_nsproxy);
12. Ok(0).
```

`setns_pidfd(pidfd_file, nstype)`:
1. if nstype & !NS_VALID_MASK != 0: return Err(EINVAL).
2. let target = pidfd_get_task(pidfd_file).ok_or(ESRCH)?;
3. let bits = if nstype == 0 { NS_VALID_MASK } else { nstype };
4. let new_nsproxy = clone_nsproxy(current().nsproxy);
5. /* For each kind in bits */
6. for kind in iter_ns_kinds(bits) {
     let target_ns = target.nsproxy.get(kind);
     target_ns.ops.may_install(current(), &target_ns)?;
     if target_ns.ops.requires_single_threaded() && !is_single_threaded(current()): return Err(EINVAL).
     target_ns.ops.install(&mut new_nsproxy, &target_ns)?;
   };
7. /* Userns special-case (separate cred install) */
8. if bits & CLONE_NEWUSER != 0 {
     let new_cred = install_userns_cred(target.cred.user_ns.clone())?;
     commit_creds(new_cred);
   };
9. switch_task_namespaces(current(), new_nsproxy);
10. put_task_struct(target).
11. Ok(0).
```

`ns.ops.may_install(task, ns)`:
- Each kind implements: ns_capable(ns.owning_user_ns, CAP_SYS_ADMIN); for CLONE_NEWPID: check ancestor relation; for CLONE_NEWUSER: ancestry check.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `fd_type_validated` | INVARIANT | per-sys_setns: file must be nsfs or pidfd. |
| `nstype_match` | INVARIANT | per-setns_single: nstype either 0 or matches ns.ops.kind_flag. |
| `caps_required` | INVARIANT | per-may_install: CAP_SYS_ADMIN in target userns. |
| `single_threaded_for_user_mnt` | INVARIANT | per-CLONE_NEWUSER|CLONE_NEWNS: caller single-threaded. |
| `pidns_descendant_only` | INVARIANT | per-CLONE_NEWPID: target pidns is descendant of caller's. |
| `nsproxy_install_atomic` | INVARIANT | per-sys_setns: switch_task_namespaces under task lock. |
| `failure_rollback` | INVARIANT | per-pidfd-multi: any kind-install fail ⟹ no commit. |

### Layer 2: TLA+

`uapi/setns.tla`:
- States: task.nsproxy, task.cred, target-task.nsproxy.
- Properties:
  - `safety_caps` — per-call: CAP_SYS_ADMIN in target userns.
  - `safety_pidns_relation` — per-CLONE_NEWPID: target is descendant.
  - `safety_atomicity` — per-call: success ⟹ all requested ns installed; fail ⟹ none.
  - `safety_single_threaded` — per-CLONE_NEWUSER/NEWNS: live thread count == 1.
  - `liveness_terminates` — per-call: returns success or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Setns::sys_setns` post: success ⟹ ∀ requested kind: task.nsproxy.<kind> == target ns | `Setns::sys_setns` |
| `setns_single` post: ns_refcount += 1 for new; -= 1 for old | `setns_single` |
| `setns_pidfd` post: target task ref taken+released balanced | `setns_pidfd` |
| `may_install` post: CAP_SYS_ADMIN held ∧ relation ok | `may_install` |

### Layer 4: Verus/Creusot functional

Per-setns semantic equivalence with upstream `sys_setns` + per-kind `ns_install` hooks: per-Documentation/admin-guide/namespaces/.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

setns-syscall reinforcement:

- **Per-CAP_SYS_ADMIN in target userns** — defense against per-cross-userns privilege misuse.
- **Per-CLONE_NEWPID descendant-only** — defense against per-pidns escape.
- **Per-CLONE_NEWUSER ancestor relation** — defense against per-foreign-userns join.
- **Per-CLONE_NEWUSER|CLONE_NEWNS single-threaded restriction** — defense against per-thread-divergence vulnerability.
- **Per-pidfd valid-target check** — defense against per-fd-type confusion.
- **Per-nsfs fd kind validation against nstype** — defense against per-kind mismatch.
- **Per-atomic-rollback on multi-ns failure** — defense against per-partial-join inconsistency.
- **Per-MNT_LOCKED unchanged on setns(NEWMNT)** — defense against per-locked-mount-relax.
- **Per-GRKERNSEC_CHROOT_FINDTASK: chrooted task cannot pidfd to outside tasks** — defense against per-chroot-escape via setns.
- **Per-PAX UDEREF on syscall args** — defense against per-kernel-pointer fixup race.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-setns.
- **Per-namespace-refcount strict get/put** — defense against per-UAF on ns lifetime.
- **Per-stacking-bounded** — defense against per-recursive-namespace-join runaway.
- **Per-cred install via commit_creds atomic** — defense against per-half-credential race.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `unshare(2)` namespace creation (covered in `unshare.md`).
- `pidfd_open(2)` (covered in its own Tier-5).
- Per-namespace-kind install semantics (covered in `kernel/<ns>.md` Tier-3).
- Implementation code.
