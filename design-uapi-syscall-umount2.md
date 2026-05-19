---
title: "Tier-5: syscall 166 — umount2(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`umount2(2)` detaches the filesystem mounted at `target`. Per-MNT_FORCE forces dirty-superblock disconnect for hung remote filesystems. Per-MNT_DETACH performs a lazy umount: detach now, free when last ref drops. Per-MNT_EXPIRE marks an unused mount for expiration (a second call after idleness completes it). Per-UMOUNT_NOFOLLOW refuses to traverse a final-component symlink at `target`. The legacy `umount(2)` (syscall 22 — not on x86_64) implicit-flag=0 form folds into this. Critical for: container teardown, NFS-server-loss recovery, namespace cleanup.

This Tier-5 covers `syscall 166 umount2`.

### Acceptance Criteria

- [ ] AC-1: umount2("/mnt", 0) on idle mount: succeeds; /proc/self/mountinfo no longer lists.
- [ ] AC-2: Busy mount with no flags: EBUSY.
- [ ] AC-3: umount2("/mnt", MNT_DETACH) on busy: succeeds; mount disappears from ns but inode-ref kept.
- [ ] AC-4: umount2("/mnt", MNT_FORCE) on NFS with dead server: invokes umount_begin and unblocks.
- [ ] AC-5: First MNT_EXPIRE on idle: returns 0, MNT_EXPIRE flag set; second MNT_EXPIRE: umounts.
- [ ] AC-6: MNT_EXPIRE between which access occurred: EAGAIN.
- [ ] AC-7: MNT_EXPIRE | MNT_FORCE: EINVAL.
- [ ] AC-8: UMOUNT_NOFOLLOW on symlink target: ELOOP/EINVAL (depending on lookup stage).
- [ ] AC-9: Non-mountpoint path: EINVAL.
- [ ] AC-10: mnt_ns rootfs without MNT_DETACH: EINVAL.
- [ ] AC-11: Non-CAP_SYS_ADMIN: EPERM.
- [ ] AC-12: MNT_LOCKED less-privileged: EPERM.
- [ ] AC-13: Shared subtree: peers umounted via propagation.

### Architecture

```
Umount::sys_umount2(target: UserPtr<u8>, flags: i32) -> Result<i32, Errno>
```

1. /* Flag validation */
2. if flags & !UMOUNT_FLAG_MASK != 0: return Err(EINVAL).
3. if (flags & MNT_EXPIRE) && (flags & (MNT_FORCE|MNT_DETACH)) != 0: return Err(EINVAL).
4. /* Lookup target */
5. let lookup = LOOKUP_MOUNTPOINT | if flags & UMOUNT_NOFOLLOW == 0 { LOOKUP_FOLLOW } else { 0 };
6. let path = user_path_at(AT_FDCWD, target, lookup)?;
7. /* Must be mountpoint */
8. if path.dentry != path.mnt.mnt_root: { path_put(&path); return Err(EINVAL); }
9. /* Permission */
10. let mnt_ns = current().nsproxy.mnt_ns;
11. if !ns_capable(mnt_ns.user_ns, CAP_SYS_ADMIN): { path_put(&path); return Err(EPERM); }
12. /* Core */
13. let mnt = real_mount(path.mnt);
14. let result = do_umount(mnt, flags);
15. path_put(&path).
16. result.
```

`Umount::do_umount(mnt, flags)`:
1. /* Cannot umount rootfs without MNT_DETACH */
2. if mnt == mnt.mnt_ns.root && flags & MNT_DETACH == 0: return Err(EINVAL).
3. /* Locked-mount checks */
4. if mnt.mnt_flags & MNT_LOCKED && !ns_can_unmount_locked(): return Err(EPERM).
5. /* Force */
6. if flags & MNT_FORCE != 0 && mnt.mnt_sb.s_op.umount_begin.is_some():
   - mnt.mnt_sb.s_op.umount_begin(mnt.mnt_sb).
7. /* Acquire namespace lock + mount_lock */
8. namespace_lock();
9. lock_mount_hash();
10. /* Expire */
11. if flags & MNT_EXPIRE != 0:
    - if !can_umount(mnt): { unlock; return Err(EAGAIN); }
    - if mnt.mnt_flags & MNT_EXPIRE != 0 && !accessed_since_mark(mnt):
      - umount_tree(mnt, UMOUNT_PROPAGATE).
      - unlock; return Ok(0).
    - mnt.mnt_flags |= MNT_EXPIRE.
    - unlock; return Ok(0).
12. /* Detach / busy / regular */
13. if flags & MNT_DETACH == 0:
    - if !can_umount(mnt): { unlock; return Err(EBUSY); }
14. /* Perform tree umount */
15. umount_tree(mnt, UMOUNT_PROPAGATE | if flags & MNT_DETACH != 0 { UMOUNT_SYNC } else { 0 }).
16. unlock_mount_hash(); namespace_unlock().
17. Ok(0).
```

`Umount::can_umount(mnt) -> bool`:
- mnt.mnt_count == 2 (one caller, one ns) AND no child mounts AND no open files referencing it.

`Umount::umount_tree(mnt, how)`:
- Recursive walk over mnt + propagation peers.
- For each m: detach from parent + ns list, schedule mntput.

### Out of Scope

- `move_mount` (covered separately).
- `pivot_root` (covered in `pivot_root.md`).
- Per-fs `umount_begin` implementations (covered in fs/<name>/Tier-3).
- Implementation code.

### signature

```c
int umount2(const char *target, int flags);
```

Per-x86_64-syscall-table: `__NR_umount2 = 166`. Per-glibc: `sys/mount.h`. Per-vDSO: not vDSO-vectored.

### parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `target` | `const char *` | in | Path to mountpoint. |
| `flags` | `int` | in | MNT_FORCE (1) | MNT_DETACH (2) | MNT_EXPIRE (4) | UMOUNT_NOFOLLOW (8). |

### return

- 0 on success.
- -1 with `errno` on failure.

### errors

| errno | Cause |
|---|---|
| EPERM | Caller lacks CAP_SYS_ADMIN in mnt_ns.user_ns; or MNT_LOCKED on mount and clearing not permitted. |
| EBUSY | Mount in use (open files, child mounts) and neither MNT_DETACH nor MNT_FORCE supplied. |
| EAGAIN | MNT_EXPIRE set on a not-yet-expired mount that became unused (second call needed); or MNT_EXPIRE on a busy mount. |
| EINVAL | Unknown flag; conflicting MNT_EXPIRE with MNT_FORCE or MNT_DETACH; `target` not a mountpoint; `target` is root of `/`. |
| ENOENT | `target` does not exist. |
| ENOTDIR | Component not a directory. |
| ELOOP | Symlink loop, or UMOUNT_NOFOLLOW and final component is symlink. |
| ENAMETOOLONG | Path > PATH_MAX. |
| EFAULT | `target` outside accessible address space. |

### abi surface

- Syscall number: `__NR_umount2 = 166` (x86_64; arch number stable).
- Flag layout: MNT_FORCE=0x1, MNT_DETACH=0x2, MNT_EXPIRE=0x4, UMOUNT_NOFOLLOW=0x8.
- Mutual exclusion: MNT_EXPIRE incompatible with MNT_FORCE|MNT_DETACH.
- The legacy `umount(target)` (no flags arg, x86 only on some 32-bit archs) is not exposed on x86_64; userspace uses `umount2(target, 0)`.

### compatibility contract

REQ-1: Flag validation:
- flags & ~(MNT_FORCE|MNT_DETACH|MNT_EXPIRE|UMOUNT_NOFOLLOW) ⟹ EINVAL.
- (flags & MNT_EXPIRE) ∧ (flags & (MNT_FORCE|MNT_DETACH)) ⟹ EINVAL.

REQ-2: Path lookup:
- Lookup flag LOOKUP_FOLLOW unless UMOUNT_NOFOLLOW.
- LOOKUP_AUTOMOUNT cleared (do not trigger automount on umount).
- Final component must be the mountpoint dentry (path.mnt.mnt_root == path.dentry).
- Otherwise EINVAL.

REQ-3: Permission:
- ns_capable(mnt_ns.user_ns, CAP_SYS_ADMIN) required.
- Special-case: mount with MNT_USER_DELETED bit set may permit calling user (rare; not exposed in v7.1).

REQ-4: Cannot umount the rootfs of the mnt_ns:
- if mount == mnt_ns.root and !MNT_DETACH: EINVAL.
- mnt_ns rootfs detach requires MNT_DETACH and CAP_SYS_ADMIN.

REQ-5: Per-MNT_LOCKED:
- Mount cloned by less-privileged user-ns carries MNT_LOCKED.
- umount2 on locked mount from less-privileged context: EPERM.

REQ-6: Per-busy logic (do_umount):
- mount.mnt_count: total references.
- can_umount(m) checks count == 2 (caller + mount-table) and no child-mounts.
- Busy ⟹ EBUSY unless MNT_DETACH.

REQ-7: Per-MNT_DETACH (lazy):
- Mount removed from namespace immediately (so subsequent path lookups don't see it).
- Final put when last open-file/cwd-ref drops.
- propagate_umount applies to peer/slave mounts.

REQ-8: Per-MNT_FORCE:
- super_block.s_op->umount_begin called (FS-specific: NFS sets nfs_killall_sigtask).
- After force-begin, normal umount proceeds; may still EBUSY if local-fs ignores force.

REQ-9: Per-MNT_EXPIRE:
- First call on idle mount: marks MNT_EXPIRE on the mount, returns 0.
- Second call (while still idle): performs umount.
- If accessed between calls: MNT_EXPIRE bit cleared on access; second call returns EAGAIN.
- Mount is busy: EAGAIN.

REQ-10: Per-propagation:
- umount_tree walks shared/slave subtree.
- Peer mounts under MNT_SHARED also umounted (propagation).
- MNT_UNBINDABLE / MNT_SLAVE: propagation up but not down.

REQ-11: Per-superblock-shutdown:
- If last mount of super_block detached: deactivate_super, sync, kill super.
- Async if MNT_DETACH (in task_work).

REQ-12: Per-RCU + namespace_lock:
- All structural changes under namespace_lock + mount_lock.
- Mount free deferred via call_rcu.

REQ-13: Per-EFAULT precedes EPERM precedes ENOENT in upstream behavior — match order for capability-confused-deputy resistance.

REQ-14: Per-target == "/" with no MNT_DETACH from init mnt_ns: EBUSY (root-mount sentinel; matches POSIX umount).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | per-sys_umount2: only known bits accepted. |
| `expire_mutex` | INVARIANT | per-sys_umount2: MNT_EXPIRE ⊕ {MNT_FORCE, MNT_DETACH}. |
| `mountpoint_required` | INVARIANT | per-sys_umount2: path.dentry == path.mnt.mnt_root. |
| `rootfs_protected` | INVARIANT | per-do_umount: ns.root + !MNT_DETACH ⟹ EINVAL. |
| `locked_respect` | INVARIANT | per-do_umount: MNT_LOCKED unprivileged ⟹ EPERM. |
| `busy_eqcheck` | INVARIANT | per-can_umount: no removal if mnt_count > 2 ∨ has-children, unless MNT_DETACH. |
| `expire_idempotent` | INVARIANT | per-MNT_EXPIRE first call: idempotent flag set; second call commits. |

### Layer 2: TLA+

`uapi/umount2.tla`:
- States: mount-presence-in-ns, MNT_EXPIRE flag, busy-counter, propagation-set.
- Properties:
  - `safety_no_partial` — per-umount_tree: either whole propagation set detached or none.
  - `safety_rootfs_protect` — per-mnt_ns.root: protected from undetached umount.
  - `safety_force_calls_umount_begin` — per-MNT_FORCE: super_block.umount_begin invoked exactly once.
  - `safety_expire_resettable_on_access` — per-MNT_EXPIRE: cleared by access.
  - `liveness_terminates` — per-call: returns success or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Umount::sys_umount2` post: success ⟹ mnt removed from mnt_ns.list (regular) or ns.list change deferred (detach) | `Umount::sys_umount2` |
| `Umount::do_umount` post: MNT_DETACH ⟹ async free; non-DETACH busy ⟹ EBUSY no-state-change | `Umount::do_umount` |
| `Umount::umount_tree` post: ∀ peer in propagation set: detached | `Umount::umount_tree` |

### Layer 4: Verus/Creusot functional

Per-umount2 semantic equivalence with upstream `path_umount` + `do_umount` + `umount_tree`: per-Documentation/filesystems/sharedsubtree.txt.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

umount2-syscall reinforcement:

- **Per-MNT_LOCKED enforced** — defense against per-unprivileged unshare-pivot escape via umount.
- **Per-rootfs detach guarded** — defense against per-namespace integrity loss.
- **Per-busy-check unless MNT_DETACH** — defense against per-mount-table inconsistency.
- **Per-MNT_FORCE limited to NFS-style super_block** — defense against per-misuse on local FS.
- **Per-MNT_EXPIRE access-resets flag** — defense against per-active-fs eviction race.
- **Per-EXPIRE ⊕ FORCE/DETACH** — defense against per-undefined-state.
- **Per-CAP_SYS_ADMIN in mnt_ns.user_ns** — defense against per-cross-namespace privilege abuse.
- **Per-UMOUNT_NOFOLLOW respected** — defense against per-symlink-mountpoint trickery.
- **Per-PAX UDEREF on target copy_from_user** — defense against per-kernel-pointer-fixup race.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-umount.
- **Per-GRKERNSEC_CHROOT_MOUNT: chroot'd cannot umount** — defense against per-chroot-escape via umount-and-rebind.
- **Per-propagation-set bounded** — defense against per-runaway-detach.
- **Per-RCU-deferred-free of mounts** — defense against per-UAF on dentry/mnt.
- **Per-namespace_lock + mount_lock around state mutation** — defense against per-concurrent-umount races.

