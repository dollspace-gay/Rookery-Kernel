# Tier-5 syscall: open_tree(2) — syscall 428

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/namespace.c (SYSCALL_DEFINE3(open_tree), open_tree_attr, can_open_root)
  - fs/internal.h (open_detached_copy)
  - include/uapi/linux/mount.h (OPEN_TREE_CLONE, OPEN_TREE_CLOEXEC, AT_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (428  common  open_tree)
-->

## Summary

`open_tree(2)` returns a file descriptor referring to a mount object — either the existing mount at the given path or a freshly **cloned, detached** copy of that mount sub-tree (when `OPEN_TREE_CLONE` is set). It is one of the four "new mount API" calls (`open_tree`, `move_mount`, `fsopen`, `fspick`) introduced in Linux 5.2 to replace `mount(2)`'s overloaded string-blob interface with typed, fd-bearing primitives. The returned fd participates in `fstat`, `move_mount`, `fsmount`, `statmount`, etc., and survives changes to its origin path.

Critical for: container runtimes that need atomic mount handoff, observability tools that want a stable handle on a mount, and any privileged tool that constructs mount trees out-of-band before splicing them into the namespace.

## Signature

```c
int open_tree(int dirfd, const char *pathname, unsigned int flags);
```

```c
#define OPEN_TREE_CLONE      0x00000001   /* clone the subtree into a detached mount */
#define OPEN_TREE_CLOEXEC    O_CLOEXEC    /* 0o2000000 — apply close-on-exec to the fd */
/* AT_* flags accepted in flags: */
#define AT_EMPTY_PATH        0x1000
#define AT_NO_AUTOMOUNT      0x800
#define AT_SYMLINK_NOFOLLOW  0x100
#define AT_RECURSIVE         0x8000       /* with OPEN_TREE_CLONE: clone whole subtree */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Resolution base: `AT_FDCWD`, an open dirfd, or with `AT_EMPTY_PATH` any fd. |
| `pathname` | `const char *` | in | Path resolved relative to `dirfd`. May be `""` with `AT_EMPTY_PATH`. |
| `flags` | `unsigned int` | in | Bitmask of `OPEN_TREE_*` and `AT_*` flags. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Success: new mount fd. If `OPEN_TREE_CLONE`, fd refers to a detached clone of the subtree. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Caller lacks `CAP_SYS_ADMIN` in the mount's user namespace. |
| `EINVAL` | Unknown flag bits, `AT_RECURSIVE` without `OPEN_TREE_CLONE`, conflicting flags. |
| `EFAULT` | `pathname` pointer faults. |
| `ENOENT` | Path component missing. |
| `ENOTDIR` | Non-final path component is not a directory. |
| `ELOOP` | Symlink loop / depth exceeded. |
| `ENAMETOOLONG` | Path too long. |
| `EBADF` | `dirfd` invalid. |
| `EMFILE` / `ENFILE` | Per-process or system fd table full. |
| `ENOMEM` | Out of memory cloning the mount. |
| `EACCES` | Path traversal denied by DAC / MAC / chroot policy. |

## ABI surface

```text
__NR_open_tree (x86_64) = 428
__NR_open_tree (arm64)  = 428
__NR_open_tree (riscv)  = 428
__NR_open_tree (i386)   = 428

/* fd refers to a struct mount; usable with move_mount, fsmount,
   statmount, listmount, fstat (returns mount-fs info), and close. */
```

## Compatibility contract

REQ-1: Syscall number is **428** on x86_64. ABI-stable since Linux 5.2.

REQ-2: `flags` validation: any unknown bit other than the documented `OPEN_TREE_*` and `AT_*` set returns `-EINVAL`. Reserved bits MUST be zero.

REQ-3: `OPEN_TREE_CLOEXEC` (== `O_CLOEXEC`) marks the returned fd close-on-exec. Recommended for hardened containers.

REQ-4: `OPEN_TREE_CLONE` produces a **detached** mount: clone of the source mount's superblock-attachment, with no parent in any namespace, until `move_mount(2)` splices it back. Detached clones are private to the calling process and disappear on fd close.

REQ-5: `AT_RECURSIVE` is valid only with `OPEN_TREE_CLONE`; clones the entire subtree (all child mounts) rather than only the head mount. `AT_RECURSIVE` without `OPEN_TREE_CLONE` returns `-EINVAL`.

REQ-6: Capability gate: cloning requires `CAP_SYS_ADMIN` in the mount's user namespace (peer ns of the mount's owning mountns). Opening without clone follows normal path-traversal DAC.

REQ-7: Mount propagation type: cloned subtree inherits per-mount propagation (private/shared/slave/unbindable) per `Documentation/filesystems/sharedsubtree.rst`. Unbindable mounts cannot be `OPEN_TREE_CLONE`d; returns `-EINVAL`.

REQ-8: Path resolution uses LOOKUP_FOLLOW unless `AT_SYMLINK_NOFOLLOW`; uses LOOKUP_AUTOMOUNT unless `AT_NO_AUTOMOUNT`.

REQ-9: The returned fd is a `struct file` over an anon-inode of type `mnt:*`; `fstat` returns `S_IFDIR` for the mountpoint root.

REQ-10: `open_tree` is namespaced: it can only resolve paths visible in the caller's current mount namespace.

REQ-11: Pidfd-style lifecycle: closing the fd of a detached clone unmounts it; closing the fd of a non-detached open is just a normal close.

## Acceptance Criteria

- [ ] AC-1: `open_tree(AT_FDCWD, "/proc", OPEN_TREE_CLOEXEC)` returns positive fd; `fstat` succeeds.
- [ ] AC-2: `open_tree(AT_FDCWD, "/", OPEN_TREE_CLONE|AT_RECURSIVE|OPEN_TREE_CLOEXEC)` returns fd to a detached clone; closing it tears the clone down.
- [ ] AC-3: Without `CAP_SYS_ADMIN`, `OPEN_TREE_CLONE` returns `-EPERM`.
- [ ] AC-4: `AT_RECURSIVE` without `OPEN_TREE_CLONE` returns `-EINVAL`.
- [ ] AC-5: Unknown bit `0x80000000` returns `-EINVAL`.
- [ ] AC-6: Cloning an unbindable mount returns `-EINVAL`.
- [ ] AC-7: `move_mount` on the returned detached fd splices it into the namespace.

## Architecture

```rust
#[syscall(nr = 428, abi = "sysv")]
pub fn sys_open_tree(dirfd: i32, pathname: UserPtr<u8>, flags: u32) -> isize {
    OpenTree::do_open_tree(dirfd, pathname, flags)
}
```

`OpenTree::do_open_tree(dirfd, pathname, flags) -> isize`:
1. /* Flag validation */
2. if flags & !(OPEN_TREE_CLONE | OPEN_TREE_CLOEXEC | AT_EMPTY_PATH | AT_NO_AUTOMOUNT | AT_SYMLINK_NOFOLLOW | AT_RECURSIVE) != 0 { return -EINVAL; }
3. if (flags & AT_RECURSIVE) ∧ !(flags & OPEN_TREE_CLONE) { return -EINVAL; }
4. /* Path resolution */
5. let lookup_flags = LookupFlags::from_at(flags);
6. let path = Fs::resolve_at(dirfd, pathname, lookup_flags)?;
7. /* Capability check (clone only) */
8. if flags & OPEN_TREE_CLONE != 0 {
9.   Capability::check_ns(CAP_SYS_ADMIN, path.mnt.user_ns)?;     // EPERM
10. }
11. /* Build the mount fd */
12. let mnt_fd = if flags & OPEN_TREE_CLONE != 0 {
13.   let recursive = flags & AT_RECURSIVE != 0;
14.   Mount::clone_subtree_detached(&path, recursive)?            // ENOMEM, EINVAL on unbindable
15. } else {
16.   Mount::ref_existing_fd(&path)?
17. };
18. /* Install */
19. let oflags = if flags & OPEN_TREE_CLOEXEC != 0 { O_CLOEXEC } else { 0 };
20. FdTable::install(mnt_fd, oflags) as isize

`Mount::clone_subtree_detached(path, recursive) -> Result<MountFd>`:
1. if path.mnt.propagation == Unbindable { return Err(EINVAL); }
2. let new_mnt = mount_clone(path.mnt, path.dentry, recursive)?;
3. new_mnt.set_detached();
4. Ok(MountFd::new(new_mnt))

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_bits_validated` | INVARIANT | unknown flag bit ⟹ EINVAL. |
| `at_recursive_requires_clone` | INVARIANT | AT_RECURSIVE ∧ !OPEN_TREE_CLONE ⟹ EINVAL. |
| `clone_requires_cap` | INVARIANT | OPEN_TREE_CLONE ∧ !CAP_SYS_ADMIN ⟹ EPERM. |
| `detached_clone_lifetime` | INVARIANT | detached clone fd close ⟹ subtree torn down. |
| `unbindable_rejected` | INVARIANT | clone of unbindable mount ⟹ EINVAL. |

### Layer 2: TLA+

`fs/open-tree.tla`:
- States: flag-validate, resolve, cap-check, clone-or-ref, fd-install.
- Properties:
  - `safety_clone_implies_cap` — clone path only when capability held.
  - `safety_detached_clone_isolated` — detached clone not visible in any namespace until move_mount.
  - `safety_recursive_without_clone_rejected`.
  - `liveness_returns` — every open_tree returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_open_tree` post: success ⟹ fd installed | `OpenTree::do_open_tree` |
| `clone_subtree_detached` post: new_mnt.parent == None | `Mount::clone_subtree_detached` |
| `do_open_tree` post: !flag bits unknown | `OpenTree::do_open_tree` |

### Layer 4: Verus / Creusot functional

Per-`open_tree(2)` man-page semantic equivalence with util-linux `mount --bind` / `mount --rbind` migration tests.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`open_tree(2)` reinforcement:

- **Per-flag whitelist** — defense against per-extension-flag smuggling.
- **Per-clone capability gate** — defense against unprivileged subtree cloning.
- **Per-detached lifecycle tied to fd** — defense against per-leak detached clones.
- **Per-unbindable rejection** — defense against propagation-policy bypass.
- **Per-CLOEXEC strongly recommended** — defense against per-fd-inheritance leak across exec.

## Grsecurity / PaX surface

- **PaX UDEREF on `pathname` copy_from_user** — defense against per-pointer kernel-deref bug; SMAP enforced.
- **GRKERNSEC_CHROOT_MOUNT** — chroot processes denied open_tree entirely (cannot construct mount trees out of band).
- **GRKERNSEC_CHROOT_NOMOUNT** — even with `CAP_SYS_ADMIN`, chrooted callers cannot clone or splice mounts.
- **GRKERNSEC_DENYUSB / GRKERNSEC_LOCK_MOUNT** — once locked, open_tree(CLONE) returns -EPERM regardless of caps.
- **PAX_USERCOPY_HARDEN on path lookup buffer** — bounded copy uses whitelisted slab.
- **Per-user-namespace CAP_SYS_ADMIN strict** — non-init userns cannot clone host-namespace mounts; grsec further requires init_user_ns for any cross-namespace splice.
- **GRKERNSEC_PROC_RESTRICT for mountinfo** — detached clones not exposed via /proc/self/mountinfo to non-owner.
- **PAX_REFCOUNT on mount refcount** — defense against per-mount-ref overflow UAF.
- **Audit log on every OPEN_TREE_CLONE** — defense against silent mount-tree manipulation.
- **GRKERNSEC_HARDEN_PTRACE on mount fd inherited across ptrace** — defense against per-mount-fd-leak via ptrace.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Mount propagation algorithms (covered in Tier-3 `fs/namespace.md`).
- `move_mount(2)` splice semantics (covered in `move_mount.md`).
- `fsmount(2)` / `fsopen(2)` superblock construction.
- Implementation code.
