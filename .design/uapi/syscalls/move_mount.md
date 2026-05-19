# Tier-5 syscall: move_mount(2) — syscall 429

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/namespace.c (SYSCALL_DEFINE5(move_mount), do_move_mount, can_move_mount_beneath)
  - fs/internal.h (struct mount, prepare_path)
  - include/uapi/linux/mount.h (MOVE_MOUNT_F_*, MOVE_MOUNT_T_*, MOVE_MOUNT_SET_GROUP, MOVE_MOUNT_BENEATH)
  - arch/x86/entry/syscalls/syscall_64.tbl (429  common  move_mount)
-->

## Summary

`move_mount(2)` is the "splice" half of the new mount API. It moves a mount (or a previously detached subtree from `open_tree(OPEN_TREE_CLONE)` or `fsmount`) from a "from" location to a "to" location, where each location is the (dirfd, path) pair familiar from the `*at` family. Compared to the legacy `mount("",..., MS_MOVE, ...)` it accepts fd-relative resolution on both ends, supports `MOVE_MOUNT_BENEATH` (insert under an existing mount), and supports `MOVE_MOUNT_SET_GROUP` (recreate sharing groups for a subtree).

Critical for: container runtimes performing atomic rootfs handoff, snapshotting tools that need to graft a prepared subtree, and userspace volume managers building mount stacks without exposing intermediates.

## Signature

```c
int move_mount(int from_dirfd, const char *from_pathname,
               int to_dirfd,   const char *to_pathname,
               unsigned int flags);
```

```c
/* "from" resolution flags */
#define MOVE_MOUNT_F_SYMLINKS      0x00000001
#define MOVE_MOUNT_F_AUTOMOUNTS    0x00000002
#define MOVE_MOUNT_F_EMPTY_PATH    0x00000004
/* "to" resolution flags */
#define MOVE_MOUNT_T_SYMLINKS      0x00000010
#define MOVE_MOUNT_T_AUTOMOUNTS    0x00000020
#define MOVE_MOUNT_T_EMPTY_PATH    0x00000040
/* operation modifiers */
#define MOVE_MOUNT_SET_GROUP       0x00000100
#define MOVE_MOUNT_BENEATH         0x00000200
#define MOVE_MOUNT__MASK           0x00000377
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `from_dirfd` | `int` | in | Resolution base for source. `AT_FDCWD`, an open dirfd, or with `MOVE_MOUNT_F_EMPTY_PATH` any fd (incl. an `open_tree` fd). |
| `from_pathname` | `const char *` | in | Source path; `""` permitted with `_F_EMPTY_PATH`. |
| `to_dirfd` | `int` | in | Resolution base for destination. |
| `to_pathname` | `const char *` | in | Destination path. |
| `flags` | `unsigned int` | in | Bitmask of `MOVE_MOUNT_*`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success: source mount now appears at destination; no fd is returned (source fd, if any, still references the same mount object). |
| `-1` + `errno` | Failure; namespace unchanged. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Caller lacks `CAP_SYS_ADMIN` in mountns. |
| `EINVAL` | Unknown flag bits; both `_SET_GROUP` and `_BENEATH` set; cross-namespace move not permitted. |
| `EBUSY` | Destination mount busy or already has mount with conflicting propagation. |
| `EXDEV` | Source and destination cross mount namespaces. |
| `ENOENT` | Path resolution failed. |
| `ENOTDIR` | Non-final path component not a directory. |
| `ELOOP` | Symlink loop. |
| `ENAMETOOLONG` | Path too long. |
| `EFAULT` | Pathname pointer faulted. |
| `EACCES` | DAC / MAC denial during traversal. |

## ABI surface

```text
__NR_move_mount (x86_64) = 429
__NR_move_mount (arm64)  = 429
__NR_move_mount (riscv)  = 429
__NR_move_mount (i386)   = 429
```

## Compatibility contract

REQ-1: Syscall number is **429** on x86_64. ABI-stable since Linux 5.2.

REQ-2: `flags` validation: any bit outside `MOVE_MOUNT__MASK` returns `-EINVAL`. `MOVE_MOUNT_SET_GROUP | MOVE_MOUNT_BENEATH` combined returns `-EINVAL`.

REQ-3: `MOVE_MOUNT_F_EMPTY_PATH`: if set and `from_pathname == ""`, the source is `from_dirfd` itself — typically a mount fd from `open_tree`. Equivalent semantics for `_T_EMPTY_PATH`.

REQ-4: Capability gate: requires `CAP_SYS_ADMIN` in the mount namespace owning **both** the source and the destination. Cross-namespace moves return `-EXDEV`.

REQ-5: `MOVE_MOUNT_BENEATH`: inserts the source mount **under** the existing mount at the destination (the destination becomes a child of the inserted mount). The destination mount must be the "head" mount on its mountpoint and must be locked-mount-compatible. Designed for transparent layer insertion without unmounting upper layers.

REQ-6: `MOVE_MOUNT_SET_GROUP`: copies the mount-propagation group (peer-group id) of source to destination — effectively splicing the destination into the source's shared-subtree group. Requires both source and destination to be mounts (not just dirs).

REQ-7: Propagation rules (per `Documentation/filesystems/sharedsubtree.rst`) are honored: cannot move a shared mount under a mount that is private and unsplicable; unbindable mounts cannot be moved into shared subtrees without `MOVE_MOUNT_SET_GROUP`.

REQ-8: Atomicity: move is performed under `namespace_sem` and per-mount locks; either both ends commit or both revert.

REQ-9: A `move_mount` consuming a detached clone from `open_tree(OPEN_TREE_CLONE)` re-attaches the detached mount; closing the source fd after move no longer tears down the mount (it now has a parent).

REQ-10: `MOVE_MOUNT_BENEATH` requires the new under-mount to not already have a parent in any namespace; using a non-detached source returns `-EINVAL`.

## Acceptance Criteria

- [ ] AC-1: `move_mount(tree_fd, "", AT_FDCWD, "/mnt/dst", MOVE_MOUNT_F_EMPTY_PATH)` splices a detached clone at `/mnt/dst`.
- [ ] AC-2: `move_mount` across mount namespaces returns `-EXDEV`.
- [ ] AC-3: Without `CAP_SYS_ADMIN`, returns `-EPERM`.
- [ ] AC-4: `MOVE_MOUNT_SET_GROUP | MOVE_MOUNT_BENEATH` returns `-EINVAL`.
- [ ] AC-5: Unknown flag bit `0x80000000` returns `-EINVAL`.
- [ ] AC-6: `MOVE_MOUNT_BENEATH` with non-detached source returns `-EINVAL`.
- [ ] AC-7: Move succeeded ⟹ subsequent `/proc/self/mountinfo` shows new mountpoint.

## Architecture

```rust
#[syscall(nr = 429, abi = "sysv")]
pub fn sys_move_mount(
    from_dirfd: i32, from_pathname: UserPtr<u8>,
    to_dirfd:   i32, to_pathname:   UserPtr<u8>,
    flags: u32,
) -> isize {
    MoveMount::do_move_mount(from_dirfd, from_pathname, to_dirfd, to_pathname, flags)
}
```

`MoveMount::do_move_mount(from_dirfd, from_path, to_dirfd, to_path, flags) -> isize`:
1. /* Flag validation */
2. if flags & !MOVE_MOUNT__MASK != 0 { return -EINVAL; }
3. if (flags & MOVE_MOUNT_SET_GROUP) ∧ (flags & MOVE_MOUNT_BENEATH) { return -EINVAL; }
4. let from = Fs::resolve_at_for_move(from_dirfd, from_path, flags & 0x07)?;
5. let to   = Fs::resolve_at_for_move(to_dirfd,   to_path,   (flags >> 4) & 0x07)?;
6. /* Capability */
7. Capability::check_ns(CAP_SYS_ADMIN, from.mnt.mnt_ns)?;        // EPERM
8. Capability::check_ns(CAP_SYS_ADMIN, to.mnt.mnt_ns)?;
9. /* Cross-ns rejection */
10. if from.mnt.mnt_ns != to.mnt.mnt_ns { return -EXDEV; }
11. /* Per-mode dispatch */
12. namespace_sem.write_lock();
13. let r = if flags & MOVE_MOUNT_SET_GROUP != 0 {
14.   Mount::set_group(&from, &to)
15. } else if flags & MOVE_MOUNT_BENEATH != 0 {
16.   Mount::splice_beneath(&from, &to)                          // requires detached source
17. } else {
18.   Mount::splice(&from, &to)
19. };
20. namespace_sem.write_unlock();
21. r as isize

`Mount::splice_beneath(src, dst) -> Result<()>`:
1. if !src.is_detached() { return Err(EINVAL); }
2. if dst.mnt.is_locked() { return Err(EBUSY); }
3. propagation_check(src, dst)?;
4. mount_insert_under(src, dst);
5. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_mask_validated` | INVARIANT | bits outside MASK ⟹ EINVAL. |
| `set_group_xor_beneath` | INVARIANT | both set ⟹ EINVAL. |
| `cap_both_namespaces` | INVARIANT | move requires CAP_SYS_ADMIN in both src and dst mountns. |
| `cross_ns_rejected` | INVARIANT | src.mnt_ns != dst.mnt_ns ⟹ EXDEV. |
| `beneath_requires_detached` | INVARIANT | BENEATH ∧ !detached ⟹ EINVAL. |
| `atomicity` | INVARIANT | failure path leaves namespace state unchanged. |

### Layer 2: TLA+

`fs/move-mount.tla`:
- States: flag-validate, resolve-from, resolve-to, cap-check, lock, splice, unlock.
- Properties:
  - `safety_cross_ns_rejected`.
  - `safety_atomic_splice` — partial state never observable.
  - `safety_propagation_invariant_preserved`.
  - `liveness_returns` — every move_mount returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_move_mount` post: success ⟹ namespace contains new mount at dst | `MoveMount::do_move_mount` |
| `splice` post: source.parent = dst.mnt | `Mount::splice` |
| `splice_beneath` post: dst.parent = src | `Mount::splice_beneath` |

### Layer 4: Verus / Creusot functional

Per-`move_mount(2)` man-page semantic equivalence with util-linux `mount --move` and `mount --bind` migration tests.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`move_mount(2)` reinforcement:

- **Per-flag mask whitelist** — defense against per-extension-flag smuggling.
- **Per-cross-ns rejection** — defense against per-mount-ns escape.
- **Per-atomic splice** — defense against per-half-move race.
- **Per-cap check in BOTH namespaces** — defense against per-target-ns priv-escalation.
- **Per-BENEATH detached source rule** — defense against per-double-parent mount UAF.

## Grsecurity / PaX surface

- **PaX UDEREF on both pathnames copy_from_user** — defense against per-pointer kernel-deref bug; SMAP enforced.
- **GRKERNSEC_CHROOT_MOUNT** — chrooted process denied move_mount entirely.
- **GRKERNSEC_CHROOT_NOMOUNT** — even with CAP_SYS_ADMIN, chrooted callers cannot splice mounts.
- **GRKERNSEC_LOCK_MOUNT** — once mount-tree locked, move_mount returns -EPERM unconditionally.
- **PAX_USERCOPY_HARDEN on pathname buffers** — bounded copies use whitelisted slab.
- **Per-init_user_ns enforcement for BENEATH** — non-init userns cannot use BENEATH to overlay host mounts.
- **GRKERNSEC_AUDIT_MOUNT** — every move_mount audited with (src,dst,flags,uid,pid,exe).
- **PAX_REFCOUNT on mount refcount** — defense against per-mount-ref overflow UAF during splice.
- **GRKERNSEC_HARDEN_PTRACE** — move_mount-derived mount fds not inheritable via ptrace to lower-priv tracer.
- **GRKERNSEC_PROC_RESTRICT** — splice transitions hidden from non-owner /proc/mountinfo views.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Shared-subtree propagation algorithm (Tier-3 `fs/namespace.md`).
- Mount lifecycle and refcount management (Tier-3).
- `fsmount` / `open_tree` construction (own Tier-5 docs).
- Implementation code.
