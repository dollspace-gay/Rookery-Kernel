---
title: "Tier-5: syscall 155 — pivot_root(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`pivot_root(2)` swaps the root mount of the calling process's mount namespace: `new_root` becomes the new `/`, and the old root is reattached at `put_old`. Per-precondition: `new_root` and `put_old` must be mounts (mountpoints with mnt.mnt_root == dentry), `put_old` must be at or under `new_root`, and the current root must not be MS_SHARED (or its peer-group blocks reroot). Per-result: every `chdir("/")` after pivot sees the new root; the old root is accessible via `put_old` for cleanup (typically `umount2 -l`). Critical for: container runtime root-switch, initramfs-to-real-root transition, namespace isolation.

This Tier-5 covers `syscall 155 pivot_root`.

### Acceptance Criteria

- [ ] AC-1: chdir("/new"); pivot_root(".", "./old"); chroot("."): "/" now reflects /new contents.
- [ ] AC-2: After pivot, /proc/self/mountinfo shows new_root at "/" and old_root at /old.
- [ ] AC-3: pivot_root with cwd not under new_root: EINVAL.
- [ ] AC-4: pivot_root when new_root is not a mount: EINVAL.
- [ ] AC-5: pivot_root when put_old not under new_root: EINVAL.
- [ ] AC-6: pivot_root when current root is MS_SHARED: EINVAL.
- [ ] AC-7: Non-CAP_SYS_ADMIN: EPERM.
- [ ] AC-8: Caller chrooted (fs.root != mnt_ns.root): EINVAL.
- [ ] AC-9: Other tasks in same mnt_ns: pwd/root unaltered.
- [ ] AC-10: pivot_root then umount2("/old", MNT_DETACH): old root unmounted.
- [ ] AC-11: pivot_root then re-pivot_root back: succeeds if symmetric (no MS_SHARED).

### Architecture

```
PivotRoot::sys_pivot_root(new_root: UserPtr<u8>, put_old: UserPtr<u8>) -> Result<i32, Errno>
```

1. /* Permission */
2. let mnt_ns = current().nsproxy.mnt_ns;
3. if !ns_capable(mnt_ns.user_ns, CAP_SYS_ADMIN): return Err(EPERM).
4. /* Lookup */
5. let new = user_path_at(AT_FDCWD, new_root, LOOKUP_FOLLOW | LOOKUP_DIRECTORY)?;
6. let old = user_path_at(AT_FDCWD, put_old, LOOKUP_FOLLOW | LOOKUP_DIRECTORY)?;
7. /* Permission to manipulate */
8. let result = do_pivot_root(&new, &old);
9. path_put(&old); path_put(&new).
10. result.
```

`PivotRoot::do_pivot_root(new, old) -> Result<(), Errno>`:
1. /* Resolve current root */
2. let root = get_fs_root(current().fs);
3. /* Class checks */
4. if !is_root_mount(&new): return Err(EINVAL).               // new.dentry == new.mnt.mnt_root
5. if is_root_mount(&root_relative(&new)): return Err(EINVAL). // new != root
6. if !is_path_under(&old, &new): return Err(EINVAL).
7. if &old == &new: return Err(EINVAL).
8. if real_mount(new.mnt).mnt_ns != mnt_ns: return Err(EINVAL).
9. if current().fs.root != mnt_ns.root: return Err(EINVAL).   // chrooted
10. if !is_path_reachable(real_mount(new.mnt), new.dentry, &root): return Err(EINVAL).
11. /* Acquire locks */
12. namespace_lock();
13. lock_mount_hash();
14. /* Propagation guard */
15. if propagation_would_escape_ns(&root, &new): { unlock; return Err(EINVAL); }
16. /* Atomic swap */
17. let new_mp = lookup_mountpoint(&old);                      // creates if absent
18. detach_mounts_at(&new_mp).
19. attach_mnt(real_mount(root.mnt), new_mp);                 // old root → put_old
20. attach_mnt(real_mount(new.mnt), mnt_ns.root_mp);          // new root → namespace root
21. mnt_ns.root = real_mount(new.mnt);
22. /* Rewrite caller fs_struct */
23. set_fs_root(current().fs, new);
24. /* Unlock */
25. unlock_mount_hash(); namespace_unlock().
26. Ok(()).
```

`PivotRoot::is_path_under(p, parent) -> bool`:
- Walk p.dentry's parents up to parent.mnt.mnt_root within parent.mnt.

`PivotRoot::propagation_would_escape_ns(root, new) -> bool`:
- For each peer of root in shared subtree: peer.mnt_ns != mnt_ns ⟹ true.
- For each peer of new: same check.

### Out of Scope

- chroot per-task root change (covered in `chroot.md`).
- mount-namespace creation (covered in `unshare.md` and `setns.md`).
- Propagation theory (covered in `fs/namespace.md` Tier-3).
- Implementation code.

### signature

```c
int pivot_root(const char *new_root, const char *put_old);
```

Per-x86_64-syscall-table: `__NR_pivot_root = 155`. Per-glibc: not exposed — call via `syscall(SYS_pivot_root, ...)`. Per-vDSO: not vDSO-vectored.

### parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `new_root` | `const char *` | in | Path to mount that will become the new root; must be a mount (mountpoint root dentry). |
| `put_old` | `const char *` | in | Path under `new_root` where the old root will be re-attached; must be a directory under `new_root` (a mount itself is fine). |

### return

- 0 on success.
- -1 with `errno` on failure.

### errors

| errno | Cause |
|---|---|
| EPERM | Caller lacks CAP_SYS_ADMIN in mnt_ns.user_ns; or in chroot (GRKERNSEC_CHROOT_PIVOT). |
| EINVAL | `new_root` is not a mount; `put_old` is not under `new_root`; current root is shared and would propagate; current cwd not under new_root; new_root == current root; put_old == new_root. |
| EBUSY | `new_root` or `put_old` is the current root or has children that prevent reroot; or the new_root is being moved. |
| ENOENT | Either path does not exist. |
| ENOTDIR | Component not a directory. |
| ENAMETOOLONG | Path > PATH_MAX. |
| EFAULT | Either user pointer invalid. |
| ELOOP | Symlink loop on lookup. |

### abi surface

- Syscall number: `__NR_pivot_root = 155`.
- No flags argument (only path pair). All semantics implicit.
- Effects are observable through /proc/self/mountinfo (root rotation) and getcwd (no auto-rewrite — caller typically chdir("/") before/after).

### compatibility contract

REQ-1: Lookup:
- new_path = user_path_at(AT_FDCWD, new_root, LOOKUP_FOLLOW | LOOKUP_DIRECTORY).
- old_path = user_path_at(AT_FDCWD, put_old, LOOKUP_FOLLOW | LOOKUP_DIRECTORY).

REQ-2: Permissions:
- ns_capable(mnt_ns.user_ns, CAP_SYS_ADMIN).

REQ-3: Path-class invariants:
- new_path.dentry == new_path.mnt.mnt_root (new_root is a mount).
- old_path is a directory under new_path (vfs_path_parent_test).
- new_path != current root.
- old_path != new_path.
- new_path != current root's parent's mount.
- Current root's mount and new_path's mount belong to same mnt_ns.

REQ-4: Sharing constraint:
- The propagation type of mounts involved must not require propagation to a peer that does not have new_root accessible.
- Specifically: do_pivot_root returns EINVAL if current root or new_root is MS_SHARED in a way that would propagate the reroot outside the caller's namespace.
- Practical guard: must mount --make-private / --make-rslave the root subtree before pivot.

REQ-5: Mount-state transitions:
- Detach current root from its mountpoint.
- Re-attach current root to `put_old` (which must reside under new_root post-pivot).
- Attach new_root to "/" of the namespace.
- All mnt_parent pointers across the namespace updated atomically.

REQ-6: Per-fs_struct rewriting:
- Caller's fs_struct.root rewritten to new_path.
- Caller's fs_struct.pwd: if it pointed under old root, **not** auto-rewritten (caller should chdir).
- Other tasks sharing the mnt_ns: their fs_struct.root and pwd are **not** modified (they still point to inodes under the now-relocated old root via put_old).

REQ-7: Per-namespace_lock + mount_lock acquired around the swap. Single atomic operation.

REQ-8: Per-chroot interaction:
- If caller is chrooted (fs.root != mnt_ns.root): EINVAL (cannot pivot inside chroot).

REQ-9: Per-mountpoint validity:
- new_root.mnt must have a parent (i.e. it must itself be mounted on something).
- new_root.mnt.mnt_parent != mnt_ns.root.mnt (cannot pivot to current root).

REQ-10: Per-MNT_LOCKED:
- Locked mounts in the subtree may prevent pivot (EBUSY) when the caller lacks the userns rights to manipulate them.

REQ-11: Per-symlink-final: LOOKUP_FOLLOW is default. There is no NO_FOLLOW variant.

REQ-12: Per-cwd-not-under-new-root: returns EINVAL (POSIX-tradition: caller must be inside new_root). Upstream `do_pivot_root` checks `is_path_reachable(real_mount(new_path.mnt), new_path.dentry, &root)`.

REQ-13: Per-success: returns 0; old_root path now reflects the former root; subsequent chdir("/") sees new_root.

REQ-14: Per-userspace pattern: chdir(new_root); pivot_root(".", "./put_old"); chroot("."); umount2("./put_old", MNT_DETACH).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `caps_required` | INVARIANT | per-sys_pivot_root: CAP_SYS_ADMIN precedes lookup. |
| `chroot_rejected` | INVARIANT | per-do_pivot_root: chrooted caller (fs.root != mnt_ns.root) returns EINVAL. |
| `new_is_mount_root` | INVARIANT | per-do_pivot_root: new.dentry == new.mnt.mnt_root. |
| `old_under_new` | INVARIANT | per-do_pivot_root: old reachable under new. |
| `same_mnt_ns` | INVARIANT | per-do_pivot_root: new.mnt.mnt_ns == current.mnt_ns. |
| `propagation_safe` | INVARIANT | per-do_pivot_root: no cross-ns propagation occurs. |
| `atomic_swap` | INVARIANT | per-do_pivot_root: namespace_lock held across attach/detach pair. |

### Layer 2: TLA+

`uapi/pivot_root.tla`:
- States: mnt_ns.root, mount-parent graph, per-task fs_struct.
- Properties:
  - `safety_atomic` — per-call: success ⟹ mnt_ns.root and all parents updated; failure ⟹ no change.
  - `safety_other_tasks_untouched` — per-task t != current: t.fs unchanged.
  - `safety_no_cross_ns_leak` — per-pivot: no mount detaches from another ns.
  - `safety_chroot_blocked` — per-chrooted-caller: EINVAL.
  - `liveness_terminates` — per-call: returns success or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `PivotRoot::sys_pivot_root` post: success ⟹ mnt_ns.root == real_mount(new.mnt) | `PivotRoot::sys_pivot_root` |
| `PivotRoot::do_pivot_root` post: success ⟹ old_root accessible at put_old path | `PivotRoot::do_pivot_root` |
| `PivotRoot::do_pivot_root` post: fail ⟹ mount-graph unchanged | `PivotRoot::do_pivot_root` |
| `is_path_reachable` post: true ⟹ ancestor chain valid | `is_path_reachable` |

### Layer 4: Verus/Creusot functional

Per-pivot_root semantic equivalence with upstream `sys_pivot_root` + `do_pivot_root`: per-Documentation/admin-guide/namespaces/.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

pivot_root-syscall reinforcement:

- **Per-GRKERNSEC_CHROOT_PIVOT: chrooted caller blocked** — defense against per-chroot-escape via pivot.
- **Per-chroot detection (fs.root != mnt_ns.root)** — defense against per-double-chroot trick.
- **Per-propagation-escape guard** — defense against per-shared-subtree cross-ns leak.
- **Per-same-mnt_ns enforcement** — defense against per-mount-stealing across namespaces.
- **Per-CAP_SYS_ADMIN in mnt_ns.user_ns** — defense against per-unprivileged reroot.
- **Per-atomic namespace_lock + mount_lock** — defense against per-half-pivot inconsistency.
- **Per-cwd-under-new-root requirement** — defense against per-orphaned-cwd path-walk surprises.
- **Per-MNT_LOCKED honored** — defense against per-unshare-pivot escape.
- **Per-PAX UDEREF on path copy_from_user** — defense against per-kernel-pointer-fixup race.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-pivot.
- **Per-GRKERNSEC_CHROOT_DOUBLE: chroot inside chroot blocked when combined with pivot** — defense against per-double-chroot escape.
- **Per-GRKERNSEC_CHROOT_MOUNT: chrooted task forbidden from mount-family** — defense against per-bind-then-pivot escape.
- **Per-PaX UDEREF + KERNEXEC across pivot critical section** — defense against per-kernel-execute-user-page.
- **Per-fs_struct rewrite local to current task** — defense against per-cross-task fs hijack.

