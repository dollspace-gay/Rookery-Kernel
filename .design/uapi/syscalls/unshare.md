# Tier-5: syscall 272 — unshare(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/fork.c (ksys_unshare, sys_unshare, unshare_userns, unshare_nsproxy_namespaces)
  - kernel/nsproxy.c
  - include/uapi/linux/sched.h (CLONE_NEW*)
-->

## Summary

`unshare(2)` disassociates parts of the current task's execution context that were previously shared (typically via `clone`/`fork`). Per-CLONE_NEW* flag creates a fresh namespace of the corresponding kind without spawning a new task. Per-non-NEW flags (CLONE_FILES, CLONE_FS, CLONE_SYSVSEM) un-share other shared state. Per-namespace creation requires CAP_SYS_ADMIN in the parent userns — except CLONE_NEWUSER, which is the canonical privilege-bootstrap (allowed to unprivileged), after which the user becomes "root" in the new userns and may create the other namespaces. Critical for: containers, sandboxes, rootless Docker, namespace-based privilege boundaries.

This Tier-5 covers `syscall 272 unshare`.

## Signature

```c
int unshare(int flags);
```

Per-x86_64-syscall-table: `__NR_unshare = 272`. Per-glibc: `sched.h`. Per-vDSO: not vDSO-vectored.

## Parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `flags` | `int` | in | OR of CLONE_FILES, CLONE_FS, CLONE_SYSVSEM, CLONE_NEWNS, CLONE_NEWUTS, CLONE_NEWIPC, CLONE_NEWUSER, CLONE_NEWPID, CLONE_NEWNET, CLONE_NEWCGROUP, CLONE_NEWTIME, CLONE_THREAD-disallowed, CLONE_SIGHAND-disallowed, CLONE_VM-disallowed. |

## Return

- 0 on success.
- -1 with `errno` on failure.

## Errors

| errno | Cause |
|---|---|
| EPERM | Lacks CAP_SYS_ADMIN for one of CLONE_NEW{NS,UTS,IPC,PID,NET,CGROUP,TIME}; or unprivileged_userns_clone sysctl disabled and CLONE_NEWUSER requested. |
| EINVAL | Unknown flag bit; CLONE_THREAD/CLONE_SIGHAND/CLONE_VM requested; or unsupported combination. |
| ENOMEM | Allocation failure. |
| ENOSPC | Namespace nesting limit (sysctl `user.max_user_namespaces`, etc.) reached. |
| EUSERS | Too many user namespaces nested. |

## ABI surface

- Syscall number: `__NR_unshare = 272`.
- Flags identical to `clone(2)`'s CLONE_*: CLONE_FILES=0x400, CLONE_FS=0x200, CLONE_SYSVSEM=0x40000, CLONE_NEWNS=0x20000, CLONE_NEWUTS=0x04000000, CLONE_NEWIPC=0x08000000, CLONE_NEWUSER=0x10000000, CLONE_NEWPID=0x20000000, CLONE_NEWNET=0x40000000, CLONE_NEWCGROUP=0x02000000, CLONE_NEWTIME=0x00000080.
- Effects persist for the calling task (and its CLONE_FS/FILES sharers if those are unshared first).
- Order-of-operations: per-upstream the namespaces are created in a fixed order (user → ipc → uts → pid → net → time → cgroup → mnt) so userns must be set up first if it's part of the request.

## Compatibility contract

REQ-1: Flag validation:
- flags & ~CLONE_UNSHARE_VALID_MASK ⟹ EINVAL.
- CLONE_UNSHARE_VALID_MASK = CLONE_FILES|FS|SYSVSEM|NEWNS|NEWUTS|NEWIPC|NEWUSER|NEWPID|NEWNET|NEWCGROUP|NEWTIME.
- Explicit reject: CLONE_THREAD, CLONE_SIGHAND, CLONE_VM, CLONE_DETACHED ⟹ EINVAL.

REQ-2: Per-CLONE_NEWUSER:
- May be performed unprivileged unless sysctl `kernel.unprivileged_userns_clone = 0`.
- Creates new user_namespace as child of current's user_ns; sets up uid/gid map (empty until configured via /proc/self/uid_map).
- Caller becomes effective "root" in new userns (with all capabilities), but only against children of that userns.

REQ-3: Per-CLONE_NEWNS:
- Requires CAP_SYS_ADMIN in current user_ns (or in the newly-created userns if combined with CLONE_NEWUSER).
- Creates new mount_namespace as a copy-on-write clone of current's.
- All existing mounts cloned with MNT_LOCKED on entries the userns owner didn't have rights to manipulate.

REQ-4: Per-CLONE_NEWUTS / CLONE_NEWIPC / CLONE_NEWNET / CLONE_NEWCGROUP / CLONE_NEWTIME:
- Each requires CAP_SYS_ADMIN in current user_ns (or new userns).
- Creates fresh namespace of that type with default state.

REQ-5: Per-CLONE_NEWPID:
- Requires CAP_SYS_ADMIN.
- Important: takes effect on **next fork** — the calling task remains in its old PID namespace; its children enter the new one. This is by design (a task cannot change its PID).
- Subsequent setns into a different PID namespace is forbidden after CLONE_NEWPID has been activated in this way.

REQ-6: Per-CLONE_FILES:
- Un-share file descriptor table: dup current.files into a private copy.

REQ-7: Per-CLONE_FS:
- Un-share fs_struct (root, pwd, umask): private copy.

REQ-8: Per-CLONE_SYSVSEM:
- Un-share SysV semaphore undo list.

REQ-9: Per-order-of-operations:
- 1. Validate flags.
- 2. unshare_userns(flags) if CLONE_NEWUSER set.
- 3. unshare_fs(flags) / unshare_files(flags) / unshare_sysvsem(flags).
- 4. unshare_nsproxy_namespaces(flags) — creates remaining namespaces.
- 5. switch_task_namespaces.

REQ-10: Per-namespace-stacking limits:
- user.max_user_namespaces (default 32768).
- user.max_mnt_namespaces.
- user.max_pid_namespaces.
- user.max_net_namespaces.
- user.max_ipc_namespaces.
- user.max_uts_namespaces.
- user.max_cgroup_namespaces.
- user.max_time_namespaces.
- Exceeded ⟹ ENOSPC / EUSERS.

REQ-11: Per-userns chained capabilities:
- After CLONE_NEWUSER, caller has full capability set in new userns. This allows subsequent CLONE_NEWNS|NEWUTS|... within the new userns even though caller was unprivileged in parent userns.

REQ-12: Per-current.ns refcount management:
- get_*_ns on new namespaces; put_*_ns on old (deferred via task_work where needed).

REQ-13: Per-MNT_LOCKED installation:
- New mnt_ns from unprivileged-userns: all mounts get MNT_LOCKED on (NOSUID, NODEV, NOEXEC, ATIME-policy) so they cannot be relaxed by the unprivileged owner.

REQ-14: Per-PID-namespace pid_max:
- new pidns inherits pid_max from parent until explicit /proc/sys/kernel/pid_max write.

## Acceptance Criteria

- [ ] AC-1: unshare(CLONE_NEWUSER): succeeds even unprivileged; new userns created.
- [ ] AC-2: unshare(CLONE_NEWUSER|CLONE_NEWNS): succeeds; mount-ns cloned with MNT_LOCKED on all inherited mounts.
- [ ] AC-3: unshare(CLONE_NEWPID): succeeds; next fork()'s child has pid 1 in new pidns; current still has old pid.
- [ ] AC-4: unshare(CLONE_NEWNET) as non-root: EPERM (unless inside fresh userns).
- [ ] AC-5: unshare(CLONE_THREAD): EINVAL.
- [ ] AC-6: unshare(CLONE_VM): EINVAL.
- [ ] AC-7: unshare(0): no-op success.
- [ ] AC-8: unshare(CLONE_FILES): /proc/self/fd is now a private table.
- [ ] AC-9: unshare(CLONE_FS): subsequent chdir affects only self even if previously CLONE_FS-shared.
- [ ] AC-10: kernel.unprivileged_userns_clone = 0 + unshare(CLONE_NEWUSER) as non-root: EPERM.
- [ ] AC-11: max_user_namespaces exceeded: ENOSPC.
- [ ] AC-12: unshare(CLONE_NEWUSER|CLONE_NEWUTS): both succeed; capability path via new userns.
- [ ] AC-13: unshare(CLONE_NEWTIME): subsequent clock_gettime(CLOCK_BOOTTIME) starts from offset 0.

## Architecture

```
Unshare::sys_unshare(flags: i32) -> Result<i32, Errno>
```

1. /* Flag validation */
2. let flags = flags as u64;
3. if flags & !CLONE_UNSHARE_VALID_MASK != 0: return Err(EINVAL).
4. if flags & (CLONE_THREAD | CLONE_SIGHAND | CLONE_VM | CLONE_DETACHED) != 0: return Err(EINVAL).
5. /* Per-newuser policy gate */
6. if flags & CLONE_NEWUSER != 0 && !ns_capable(current().cred.user_ns, CAP_SYS_ADMIN):
   - if !config.unprivileged_userns_clone: return Err(EPERM).
7. /* Step 1: unshare userns first (creates capability foothold) */
8. let new_cred = if flags & CLONE_NEWUSER != 0 {
     Some(unshare_userns(flags)?)
   } else { None };
9. /* Apply new_cred temporarily so subsequent ns_capable checks see new userns */
10. let cred_guard = override_creds_if(new_cred);
11. /* Step 2: non-namespace shares */
12. let new_fs = if flags & CLONE_FS != 0 { Some(copy_fs_struct(current().fs)?) } else { None };
13. let new_files = if flags & CLONE_FILES != 0 { Some(dup_fd(current().files)?) } else { None };
14. if flags & CLONE_SYSVSEM != 0 { exit_sem(current()); }
15. /* Step 3: nsproxy namespaces */
16. let new_nsproxy = unshare_nsproxy_namespaces(flags, current().nsproxy, &current().cred)?;
17. /* Step 4: install */
18. switch_task_namespaces(current(), new_nsproxy);
19. if let Some(f) = new_fs    { current().fs    = f; }
20. if let Some(d) = new_files { current().files = d; }
21. if let Some(c) = new_cred  { commit_creds(c); }
22. /* drop cred_guard */
23. Ok(0).
```

`unshare_userns(flags) -> Result<Cred, Errno>`:
1. Check per-user-namespace count limit (max_user_namespaces).
2. new_ns = create_user_ns(current().cred)?.
3. new_cred = prepare_creds().
4. new_cred.user_ns = new_ns.
5. new_cred.cap_effective = CAP_FULL_SET (in new_ns).
6. new_cred.cap_permitted = CAP_FULL_SET.
7. new_cred.cap_inheritable = current().cred.cap_inheritable.
8. new_cred.cap_bset = current().cred.cap_bset.
9. Ok(new_cred).

`unshare_nsproxy_namespaces(flags, old, cred) -> Result<NsProxy, Errno>`:
1. Required namespaces: NS_MNT|UTS|IPC|PID|NET|CGROUP|TIME.
2. if flags & required == 0: return Ok(old) — no-op nsproxy clone.
3. /* CAP_SYS_ADMIN in cred's user_ns required for these */
4. if !ns_capable(cred.user_ns, CAP_SYS_ADMIN): return Err(EPERM).
5. /* For each kind, create new ns (subject to per-kind limits) */
6. let nsp = create_new_namespaces(flags, current(), cred.user_ns)?;
7. Ok(nsp).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_valid_mask` | INVARIANT | per-sys_unshare: bits outside mask ⟹ EINVAL. |
| `thread_vm_sighand_rejected` | INVARIANT | per-sys_unshare: CLONE_THREAD/VM/SIGHAND ⟹ EINVAL. |
| `userns_order_first` | INVARIANT | per-sys_unshare: CLONE_NEWUSER processed before other CLONE_NEW*. |
| `caps_via_new_userns` | INVARIANT | per-sys_unshare: CLONE_NEWUSER + others uses new userns capabilities. |
| `pidns_takes_effect_on_fork` | INVARIANT | per-sys_unshare: CLONE_NEWPID does not change current's pid; child fork enters new pidns. |
| `ns_refcount_balanced` | INVARIANT | per-sys_unshare: every new ns gets +1 ref; old ns gets -1 (deferred). |
| `stacking_limits_enforced` | INVARIANT | per-create_*_ns: respective max checked. |

### Layer 2: TLA+

`uapi/unshare.tla`:
- States: task.nsproxy, task.cred, namespace-tree depth.
- Properties:
  - `safety_userns_first` — per-call: userns created before other nses.
  - `safety_atomic_install` — per-call: either all requested unshares apply, or none.
  - `safety_pidns_defer` — per-CLONE_NEWPID: current.pidns unchanged; child.pidns = new.
  - `safety_locked_mounts_on_unpriv` — per-CLONE_NEWUSER+CLONE_NEWNS unprivileged: MNT_LOCKED on inherited mounts.
  - `safety_max_namespaces` — per-create: counter ≤ sysctl limit.
  - `liveness_terminates` — per-call: returns success or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Unshare::sys_unshare` post: success ⟹ task.nsproxy reflects requested unshares | `Unshare::sys_unshare` |
| `unshare_userns` post: new_cred.user_ns is child of caller's user_ns | `unshare_userns` |
| `unshare_nsproxy_namespaces` post: ∀ NEW flag in set: new ns of that kind installed | `unshare_nsproxy_namespaces` |
| `create_user_ns` post: new_ns.level == parent.level + 1 ≤ MAX_NS_LEVEL | `create_user_ns` |

### Layer 4: Verus/Creusot functional

Per-unshare semantic equivalence with upstream `ksys_unshare`: per-Documentation/admin-guide/namespaces/.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

unshare-syscall reinforcement:

- **Per-CLONE_THREAD/VM/SIGHAND rejected** — defense against per-process-state confusion.
- **Per-sysctl kernel.unprivileged_userns_clone gate** — defense against per-userns-attack-surface exposure.
- **Per-max-namespaces-{user,mnt,pid,net,ipc,uts,cgroup,time}** — defense against per-namespace-bomb DoS.
- **Per-userns-level ≤ MAX_NS_LEVEL (32)** — defense against per-deep-stacking exhaustion.
- **Per-MNT_LOCKED installed on unprivileged-CLONE_NEWNS** — defense against per-pivot-and-relax escape.
- **Per-CAP_SYS_ADMIN required for non-USER namespaces** — defense against per-unprivileged-namespace-creation.
- **Per-userns-first ordering** — defense against per-capability-gap during multi-flag unshare.
- **Per-PID-ns deferred-to-fork semantics** — defense against per-current-pid-change UB.
- **Per-GRKERNSEC_CHROOT_FINDTASK: chrooted task cannot see tasks across new nsproxy** — defense against per-chroot-escape via shared procfs.
- **Per-PAX UDEREF on flags read** — defense against per-kernel-pointer fixup race.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-unshare.
- **Per-fs_struct copy under write_lock** — defense against per-CLONE_FS race.
- **Per-files_struct dup atomic** — defense against per-fd-table inconsistency.
- **Per-namespace-stacking bounded** — defense against per-recursive-unshare runaway.
- **Per-new-cred install via commit_creds atomic** — defense against per-half-commit credential race.

## Grsecurity/PaX-style Reinforcement

Baseline kernel-wide mitigations relied upon by `unshare(2)`:

- **PAX_USERCOPY** — bounds-checks the flags read and any subsequent per-ns prepare paths that copy from user (mnt_ns flags inheritance).
- **PAX_KERNEXEC** — per-subsystem `*_clone` and `commit_nsset` paths live in RX/RO text; unshare cannot rewrite ns_operations during prepare.
- **PAX_RANDKSTACK** — per-syscall stack-offset randomization on `__do_sys_unshare` defeats stack-grooming chains that ride the unshare/setns combo.
- **PAX_REFCOUNT** — saturating refs on every namespace being created (`user_ns->count`, `mnt_ns->count`, `pid_ns_for_children`, `net->count`, `ipc_ns->count`, `uts_ns->count`, `cgroup_ns->count`, `time_ns->count`); refcount underflow on prepare-fail rollback traps.
- **PAX_MEMORY_SANITIZE** — zeroes failed prepare allocations (new `struct nsproxy`, new `struct cred`, copied fs_struct/files_struct) before kfree.
- **PAX_UDEREF** — strict user/kernel pointer separation for the flags argument and any indirect ns lookup.
- **PAX_RAP / kCFI** — forward-edge CFI on `create_new_namespaces`, `commit_creds`, and per-ns `copy_*_ns` callbacks.
- **GRKERNSEC_HIDESYM** — `ksys_unshare`, `create_new_namespaces`, `unshare_userns` hidden from unprivileged kallsyms.
- **GRKERNSEC_DMESG** — restricts dmesg so per-ns creation failure traces are root-only.

Namespace-family-specific reinforcement:

- **CAP_SYS_ADMIN required for non-USER namespaces** — paired with userns-first ordering so the cap check is honored after a fresh USER_NS install.
- **GRKERNSEC_CHROOT_FINDTASK** — chrooted task cannot use unshare(CLONE_NEWPID|CLONE_NEWUSER) to escape via shared procfs.
- **GRKERNSEC_CHROOT_NICE / CHROOT_CAPS** — bounds the capability set a chroot'd task can synthesize in a freshly-unshared userns.
- **kernel.unprivileged_userns_clone sysctl gate** — global policy switch; paired with PAX_REFCOUNT on user_ns->count for accurate level enforcement.
- **MAX_NS_LEVEL=32 + max-namespaces-per-user sysctls** — caps recursive unshare bombs; PAX_REFCOUNT keeps the level counter UAF-safe.

Rationale: `unshare(2)` is the primary attack surface for userns-elevation chains; the combination of grsec chroot/userns gates and PaX refcount/memory-sanitize on every per-ns allocation collapses both the privilege-bracket-escape and the namespace-bomb DoS classes.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `clone(2)` / `clone3(2)` task creation (covered in `clone.md` / `clone3.md`).
- `setns(2)` namespace attach (covered in `setns.md`).
- Per-namespace-kind internals (covered in `kernel/<ns>.md` Tier-3 docs).
- Implementation code.
