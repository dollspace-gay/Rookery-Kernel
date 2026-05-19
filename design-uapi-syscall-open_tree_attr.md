---
title: "Tier-5 syscall: open_tree_attr(2) — forward-looking (ENOSYS in 7.1.0-rc2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`open_tree_attr(2)` is a **proposed syscall** that combines `open_tree(2)` (detach-or-clone a subtree of the mount namespace into a fresh, FD-owned VFS view) with an immediate `mount_setattr(2)` call applied to the cloned subtree before the file descriptor is returned. The motivation is to eliminate the small but security-significant window in which a freshly-cloned subtree exists with the default attributes (e.g. without `MOUNT_ATTR_NOSUID`, without an idmap), where a racing thread holding the same mount-namespace can observe / pin / re-clone the subtree with its un-restricted attributes. Wiring atomic clone-and-attrs into a single syscall closes that window.

The current 7.1.0-rc2 baseline **does not** ship this syscall: `__NR_open_tree_attr = 471` is reserved in the Rookery design and the entry trampoline returns `-ENOSYS`. Userspace MUST continue to call `open_tree(2)` followed by `mount_setattr(2)` (and accept the race window) until the syscall is wired in a later phase.

Critical for: container-runtime cloning of `/proc`, `/sys`, `/dev` subtrees with `MOUNT_ATTR_NOSUID | MOUNT_ATTR_NODEV | MOUNT_ATTR_NOEXEC` atomically; ID-mapped mounts (`MOUNT_ATTR_IDMAP`) applied without exposing the un-mapped tree even momentarily; rootless container engines that cannot afford a TOCTOU between clone and setattr.

### Acceptance Criteria

- [ ] AC-1: In 7.1.0-rc2, `syscall(__NR_open_tree_attr, ...)` returns `-1` with `errno == ENOSYS`.
- [ ] AC-2: After implementation: clone + nosuid in one call: fd has `MOUNT_ATTR_NOSUID` set as observed via `statx`.
- [ ] AC-3: After implementation: concurrent reader thread cannot observe the cloned subtree without nosuid.
- [ ] AC-4: After implementation: bad attr_set/clr conflict returns `EINVAL`; fd not created.
- [ ] AC-5: After implementation: `usize = sizeof + 4` with zero tail succeeds.
- [ ] AC-6: After implementation: `usize = sizeof + 4` with non-zero tail returns `E2BIG`.
- [ ] AC-7: After implementation: `MOUNT_ATTR_IDMAP` with invalid `userns_fd` returns `EBADF` and no fd.
- [ ] AC-8: After implementation: caller without `CAP_SYS_ADMIN` returns `EPERM`.
- [ ] AC-9: After implementation: setattr-step failure closes the cloned fd and leaves ns unchanged.
- [ ] AC-10: After implementation: `attr_set == 0` no-op behaves identical to `open_tree(2)`.

### Architecture

```rust
#[syscall(nr = 471, abi = "sysv")]
pub fn sys_open_tree_attr(
    dfd: i32,
    filename: UserPtr<u8>,
    flags: u32,
    uattr: UserPtr<MountAttr>,
    usize_: usize,
) -> isize {
    /* Forward-looking: not implemented in 7.1.0-rc2. */
    -ENOSYS as isize
}
```

When implemented:

`OpenTreeAttr::do_open_tree_attr(dfd, filename, flags, uattr, usize) -> isize`:
1. /* Validate flags */
2. let known_flags = OPEN_TREE_CLONE | OPEN_TREE_CLOEXEC | AT_RECURSIVE | AT_EMPTY_PATH | AT_SYMLINK_NOFOLLOW;
3. if (flags & !known_flags) != 0 { return -EINVAL; }
4. /* Copy attrs with forward-compat tail probe */
5. let kattr = OpenTreeAttr::copy_attr(uattr, usize)?;            // EFAULT / EINVAL / E2BIG
6. if !OpenTreeAttr::attrs_valid(&kattr) { return -EINVAL; }
7. /* Resolve path */
8. let path = path_lookupat(dfd, filename, flags)?;               // ENOENT / ELOOP / EACCES
9. /* Capability */
10. if !ns_capable(path.mnt.mnt_ns.user_ns, CAP_SYS_ADMIN) { return -EPERM; }
11. /* Atomic clone + setattr under namespace_sem */
12. let _g = namespace_sem.write();
13. let clone = clone_mount(&path, flags & (OPEN_TREE_CLONE | AT_RECURSIVE))?;  // EBUSY
14. let r = mount_setattr_locked(&clone, flags & AT_RECURSIVE, &kattr);
15. if let Err(e) = r {
16.   /* Detached clone is dropped by put_mount on Drop */
17.   drop(clone);
18.   return -e as isize;
19. }
20. /* Install fd */
21. let cloexec = flags & OPEN_TREE_CLOEXEC != 0;
22. let fd = install_mount_fd(clone, cloexec)?;                   // EMFILE
23. audit_log(AUDIT_OPEN_TREE_ATTR, &kattr, fd);
24. fd as isize

`OpenTreeAttr::copy_attr(uattr, usize) -> Result<MountAttr>`:
1. const K: usize = size_of::<MountAttr>();
2. if usize == 0 { return Err(EINVAL); }
3. let n = min(usize, K);
4. let mut k = MountAttr::zeroed();
5. unsafe { uattr.copy_in_partial(&mut k, n)?; }
6. if usize > K {
7.   let tail = usize - K;
8.   let mut probe = [0u8; 64];
9.   for off in (0..tail).step_by(64) {
10.    let n = min(64, tail - off);
11.    uattr.add(K + off).copy_in_partial(&mut probe[..n])?;
12.    if probe[..n].iter().any(|b| *b != 0) { return Err(E2BIG); }
13.  }
14. }
15. Ok(k)
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `enosys_in_baseline` | INVARIANT | per-7.1.0-rc2: sys_open_tree_attr always returns -ENOSYS. |
| `atomic_clone_setattr` | INVARIANT | per-implemented: clone+setattr happen under namespace_sem.write OR detached state. |
| `setattr_fail_rollback` | INVARIANT | per-implemented: setattr failure ⟹ no fd installed, no ns observer can see clone. |
| `tail_zero_check` | INVARIANT | per-implemented: usize > K & non-zero tail ⟹ E2BIG. |
| `cap_check_before_clone` | INVARIANT | per-implemented: CAP_SYS_ADMIN check precedes clone_mount. |
| `no_widen_caps` | INVARIANT | per-implemented: never grants capability beyond clone+setattr composition. |

### Layer 2: TLA+

`fs/open-tree-attr.tla`:
- States: per-mount-ns mount tree, detached-clone state, fd table.
- Properties:
  - `safety_enosys_baseline` — in 7.1.0-rc2, every call returns ENOSYS.
  - `safety_observer_sees_either_old_or_attrd` — any ns observer sees the pre-clone state or the post-attrs state, never an intermediate.
  - `safety_rollback_on_setattr_fail` — setattr fail ⟹ no fd, no ns mutation.
  - `safety_no_widen_caps` — equivalent to open_tree;mount_setattr composition under same creds.
  - `liveness_terminates` — each call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `sys_open_tree_attr` post (baseline): returns -ENOSYS unconditionally | `sys_open_tree_attr` (baseline) |
| `do_open_tree_attr` post (implemented): success ⟹ fd refers to clone with kattr applied | `OpenTreeAttr::do_open_tree_attr` |
| `copy_attr` post: returned MountAttr matches uattr (zero-padded) | `OpenTreeAttr::copy_attr` |
| `mount_setattr_locked` post: attrs visible on clone before namespace_sem release | `mount_setattr_locked` |

### Layer 4: Verus / Creusot functional

- Baseline: probe call equals `errno == ENOSYS`.
- Implemented: semantic equivalence to `open_tree(2)` + `mount_setattr(2)` composition under atomicity; LTP / xfstests mount tests; `crun --idmapped-mount` exercises path.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`open_tree_attr(2)` reinforcement (forward-looking):

- **Per-atomic clone+setattr** — defense against per-TOCTOU pre-attrs observation race.
- **Per-detached state until attrs applied** — defense against per-cross-ns observer probing un-attrs'd clone.
- **Per-`usize` tail zero-probe** — defense against per-extension-field smuggling.
- **Per-CAP_SYS_ADMIN check pre-clone** — defense against per-cred-race UAF.
- **Per-rollback on setattr fail** — defense against per-leaked-cloned-mount.
- **Per-IDMAP userns_fd validated at syscall time** — defense against per-userns-swap TOCTOU.
- **Per-`ENOSYS` baseline semantics** — defense against per-EINVAL-confusion in libc probes.

## Grsecurity / PaX surface

- **GRKERNSEC_CHROOT_MOUNT denial** — inside a grsec chroot, `open_tree_attr` is rejected (returns `EPERM`) unless `GRKERNSEC_CHROOT_NSENTER` plus explicit allow-list is set. Mount-tree manipulation is one of the strongest jail-escape primitives.
- **PaX UDEREF on `uattr` copy_from_user** — SMAP-guarded; trailing-byte probe is bounded.
- **PAX_USERCOPY_HARDEN on `struct mount_attr` copy** — bounded copy via whitelisted slab; oversize `usize` is rejected before allocation.
- **GRKERNSEC_HARDEN_MOUNT** — globally disables all mount-namespace mutation (`open_tree`, `move_mount`, `mount_setattr`, `fsmount`, and `open_tree_attr`) for unprivileged tasks; only `CAP_SYS_ADMIN` in `init_user_ns` retains access.
- **CAP_SYS_ADMIN in init_user_ns required for IDMAP** — grsec elevates `MOUNT_ATTR_IDMAP` to demand init-userns admin even when the target ns owner would otherwise suffice. Closes container-side idmap-injection.
- **GRKERNSEC_AUDIT_MOUNT** — every successful `open_tree_attr` audited with full `mount_attr` payload, target path, caller creds, and resulting fd. Forensic completeness.
- **PaX KERNEXEC + PAX_NOEXEC for new mount points** — when `MOUNT_ATTR_NOEXEC` is set on the clone, the cloned vfsmount inherits PAX_NOEXEC enforcement on its underlying inode pages immediately.
- **GRKERNSEC_LINK + AT_SYMLINK_NOFOLLOW interaction** — when GRKERNSEC_LINK is enabled, `open_tree_attr` traversing user-owned symlinks fails per grsec sticky-symlink restrictions.
- **PAX_REFCOUNT on cloned vfsmount refcount** — saturating; setattr-fail rollback cannot underflow.
- **No_new_privs interaction** — NNP does not block this syscall (it does not exec), but the cloned subtree retains NNP-honoring nosuid semantics when set.
- **Anti-fingerprint** — `ENOSYS` in baseline means no version oracle from partial-implementation differences.
- **Per-`-ENOSYS` is the only baseline outcome** — slot 471 sealed to `sys_ni_syscall` thunk under grsec until the implementation lands.

## Open Questions

- Q1: Should `attr_set == 0` no-op path require `OPEN_TREE_CLONE`? Leaning yes (per REQ-4); bare attach without clone should not pass through this syscall. Add a dedicated LSM hook for SELinux/AppArmor distinction from raw `open_tree;mount_setattr`? Punted.
- Q2: Should `MOUNT_ATTR_IDMAP` failure mid-recursive-walk roll back partial attrs? Yes, atomically — defer to implementation.

## Out of Scope

- `open_tree(2)` (Tier-5 separate doc — bare clone path).
- `mount_setattr(2)` (Tier-5 separate doc — bare setattr path).
- `move_mount(2)` (Tier-5 separate doc — attach to ns).
- `fsmount(2)` (Tier-5 separate doc — new-mount-fd).
- ID-mapped mounts internals (Tier-3 in `fs/idmapped-mounts.md`).
- Implementation code.

### signature

```c
int open_tree_attr(int dfd, const char *filename,
                   unsigned int flags,
                   struct mount_attr *uattr,
                   size_t usize);

/* same struct as mount_setattr(2): */
struct mount_attr {
    __u64 attr_set;      /* MOUNT_ATTR_* bits to set */
    __u64 attr_clr;      /* MOUNT_ATTR_* bits to clear */
    __u64 propagation;   /* MS_SHARED|PRIVATE|SLAVE|UNBINDABLE */
    __u64 userns_fd;     /* for MOUNT_ATTR_IDMAP */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dfd` | `int` | in | Directory fd or `AT_FDCWD`; relative anchor for `filename`. |
| `filename` | `const char *` | in | Path to the mount-subtree root (empty string + `AT_EMPTY_PATH` reuses `dfd`). |
| `flags` | `unsigned int` | in | `OPEN_TREE_CLONE`, `OPEN_TREE_CLOEXEC`, `AT_RECURSIVE`, `AT_EMPTY_PATH`, `AT_SYMLINK_NOFOLLOW`. |
| `uattr` | `struct mount_attr *` | in | Attribute payload (forward-compat sized struct; see `usize`). |
| `usize` | `size_t` | in | `sizeof(*uattr)`; supports forward-compat extension. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | New file descriptor referencing the (possibly cloned) subtree with attributes applied. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `ENOSYS` | **Always, in 7.1.0-rc2.** The syscall is reserved but not implemented. |
| `EINVAL` | Bad `flags` combination, conflicting `attr_set` / `attr_clr`, bad `usize` tail. |
| `E2BIG` | `usize` exceeds kernel-known struct size and trailing bytes are non-zero. |
| `EFAULT` | `uattr` or `filename` is an invalid user pointer. |
| `EPERM` | Caller lacks `CAP_SYS_ADMIN` in the mount namespace owner. |
| `EACCES` | Path traversal denied. |
| `ENOENT` | Path component missing. |
| `EBUSY` | Subtree is locked / pinned by another mount op. |
| `EBADF` | Bad `dfd` or bad `userns_fd` for `MOUNT_ATTR_IDMAP`. |
| `ELOOP` | Excessive symlink traversal. |

### abi surface

```text
__NR_open_tree_attr (x86_64)   = 471   /* reserved; ENOSYS in 7.1.0-rc2 */
__NR_open_tree_attr (arm64)    = 471   /* reserved; ENOSYS in 7.1.0-rc2 */
__NR_open_tree_attr (generic)  = 471

/* Inherits flag space from open_tree(2): */
#define OPEN_TREE_CLONE     1
#define OPEN_TREE_CLOEXEC   O_CLOEXEC
#define AT_RECURSIVE        0x8000

/* Inherits attribute space from mount_setattr(2): */
#define MOUNT_ATTR_RDONLY        0x00000001
#define MOUNT_ATTR_NOSUID        0x00000002
#define MOUNT_ATTR_NODEV         0x00000004
#define MOUNT_ATTR_NOEXEC        0x00000008
#define MOUNT_ATTR__ATIME        0x00000070
#define MOUNT_ATTR_NOATIME       0x00000010
#define MOUNT_ATTR_RELATIME      0x00000000
#define MOUNT_ATTR_STRICTATIME   0x00000020
#define MOUNT_ATTR_NOSYMFOLLOW   0x00000100
#define MOUNT_ATTR_IDMAP         0x00100000
```

### compatibility contract

REQ-1: In 7.1.0-rc2, `open_tree_attr` is wired into the syscall table at slot 471 but the implementation body returns `-ENOSYS`. Userspace probing the syscall MUST see `ENOSYS`, NOT `EINVAL` or `EFAULT`.

REQ-2: `__NR_open_tree_attr = 471` slot is reserved cross-arch.

REQ-3: When implemented (target: phase-E or later), the syscall MUST be **semantically equivalent** to:
```c
int fd = open_tree(dfd, filename, flags);
if (fd >= 0 && (uattr->attr_set | uattr->attr_clr | uattr->propagation | uattr->userns_fd)) {
    if (mount_setattr(fd, "", AT_EMPTY_PATH | (flags & AT_RECURSIVE),
                      uattr, usize) < 0) {
        int saved = errno; close(fd); errno = saved; return -1;
    }
}
return fd;
```
…but with the two operations as an **atomic VFS step** — no other mount-namespace observer sees the cloned subtree in its pre-attr state.

REQ-4: `flags` validation:
  - At least `OPEN_TREE_CLONE` is mandatory if any `attr_set`/`attr_clr` would modify properties of a non-detached tree. Bare attach-only (no clone) attribute writes are rejected (`EINVAL`) so the syscall never inadvertently mutates the live ns.
  - `AT_RECURSIVE` applies to both the clone scope AND the attribute application — exactly matches `mount_setattr` semantics.

REQ-5: `usize` validation:
  - `usize < sizeof(struct mount_attr)`: missing tail treated as zero; success is permitted (backward compat).
  - `usize > sizeof(struct mount_attr)`: trailing bytes MUST be zero, else `E2BIG` (forward-compat probe).
  - `usize == 0`: `EINVAL`.

REQ-6: Atomicity guarantee: between the `clone_mount` step and the `mount_setattr` step, NO other task in the same mount namespace can observe / clone / move the new subtree. Implementation holds `namespace_sem` write-locked for the full operation OR creates the subtree in a detached state (no path in any ns) until attributes are applied.

REQ-7: ID-mapped mounts: `MOUNT_ATTR_IDMAP` with `userns_fd` resolved at syscall time; user namespace MUST be already initialized AND the caller MUST own (or `CAP_SYS_ADMIN` over) the target user namespace.

REQ-8: If `mount_setattr` step fails, the cloned fd is closed and the namespace is unaffected — there is no "partial" outcome. Errno reflects the setattr failure.

REQ-9: Capability gate: `CAP_SYS_ADMIN` in the mount namespace owner (or in `init_user_ns` for global mounts), same as `mount_setattr(2)`.

REQ-10: No new LSM hook is added; existing `security_move_mount`, `security_sb_set_mnt_opts`, and per-attr LSM hooks run unchanged on the wrapped sequence.

REQ-11: Audit emits a single `AUDIT_OPEN_TREE_ATTR` record with `clone_flags`, `attr_set`, `attr_clr`, `propagation`, `userns_fd` reference.

REQ-12: Forward compat: `struct mount_attr` may grow; `usize` is the negotiated size, identical to existing `mount_setattr` semantics.

REQ-13: Backward compat: kernels without `open_tree_attr` MUST return `-ENOSYS` from slot 471; libc probes rely on this.

REQ-14: The syscall MUST NOT widen any capability surface; it only combines two existing operations.

REQ-15: A caller passing `attr_set == 0 && attr_clr == 0 && propagation == 0 && userns_fd == 0` MUST succeed with semantics identical to `open_tree(2)` (no-op attributes).

