---
title: "Tier-5: syscall 442 — mount_setattr(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`mount_setattr(2)` updates per-mount attributes (MNT_NOSUID, MNT_NODEV, MNT_NOEXEC, MNT_NOATIME variants, MNT_NOSYMFOLLOW) and installs a user-namespace **id-mapped** mount in a single atomic operation. Per-`struct mount_attr` carries `attr_set` (bits to set), `attr_clr` (bits to clear), `propagation` (MS_SHARED/PRIVATE/SLAVE/UNBINDABLE — 0 to leave), and `userns_fd` (for MOUNT_ATTR_IDMAP). Per-AT_RECURSIVE applies the change to the whole subtree. Per-prepare/commit pattern ensures atomicity: all-or-nothing across the subtree. Critical for: container id-mapped mounts, atomic policy tightening, recursive flag flips.

This Tier-5 covers `syscall 442 mount_setattr`.

### Acceptance Criteria

- [ ] AC-1: mount_setattr(fd, "", AT_EMPTY_PATH, {.attr_set=RDONLY}, sizeof) flips RO.
- [ ] AC-2: AT_RECURSIVE applies to entire subtree atomically.
- [ ] AC-3: Mid-subtree validation failure: no mount changes (atomicity).
- [ ] AC-4: MOUNT_ATTR_IDMAP with userns_fd: mnt_userns installed; immutable thereafter.
- [ ] AC-5: Repeated MOUNT_ATTR_IDMAP on same mount: EBUSY.
- [ ] AC-6: MOUNT_ATTR_IDMAP without userns_fd: EINVAL.
- [ ] AC-7: MOUNT_ATTR_IDMAP on FS without FS_ALLOW_IDMAP: EINVAL.
- [ ] AC-8: Conflicting atime bits in attr_set: EINVAL.
- [ ] AC-9: Unknown bit in attr_set: EINVAL.
- [ ] AC-10: size < MOUNT_ATTR_SIZE_VER0: EINVAL.
- [ ] AC-11: size > kernel-known + non-zero trailing: E2BIG.
- [ ] AC-12: MNT_LOCKED bit clear request: EPERM.
- [ ] AC-13: Non-CAP_SYS_ADMIN caller: EPERM.
- [ ] AC-14: propagation == MS_SHARED|MS_PRIVATE (both): EINVAL.

### Architecture

```
MountSetattr::sys_mount_setattr(dirfd: i32, path: UserPtr<u8>,
                                flags: u32, attr_user: UserPtr<MountAttr>,
                                size: usize) -> Result<i32, Errno>
```

1. /* Flag validation */
2. if flags & !(AT_EMPTY_PATH|AT_RECURSIVE|AT_SYMLINK_NOFOLLOW|AT_NO_AUTOMOUNT) != 0: return Err(EINVAL).
3. /* Copy extensible struct */
4. let mut attr = MountAttr::default();
5. copy_struct_from_user(&mut attr, attr_user, size, MOUNT_ATTR_SIZE_VER0)?;
6. /* Validate attr */
7. let kattr = build_mount_kattr(&attr)?;        // bits + propagation + idmap fd
8. /* Resolve path */
9. let lookup_flags = lookup_flags_from(flags);
10. let path = user_path_at(dirfd, path, lookup_flags)?;
11. /* Permission */
12. let mnt_ns = current().nsproxy.mnt_ns;
13. if !ns_capable(mnt_ns.user_ns, CAP_SYS_ADMIN): return Err(EPERM).
14. /* Prepare (validate subtree) */
15. namespace_lock();
16. let target_root = real_mount(path.mnt);
17. let recursive = flags & AT_RECURSIVE != 0;
18. mount_setattr_prepare(target_root, &kattr, recursive)?;
19. /* Commit */
20. mount_setattr_commit(target_root, &kattr, recursive);
21. namespace_unlock();
22. drop(kattr);                                   // releases userns ref if not consumed
23. path_put(&path).
24. Ok(0).
```

`MountSetattr::build_mount_kattr(attr)`:
1. if attr.propagation & !(MS_SHARED|MS_PRIVATE|MS_SLAVE|MS_UNBINDABLE) != 0: Err(EINVAL).
2. if popcount(attr.propagation & PROP_MASK) > 1: Err(EINVAL).
3. let set = attr.attr_set, clr = attr.attr_clr.
4. if (set|clr) & !MOUNT_ATTR_VALID_MASK != 0: Err(EINVAL).
5. let atime_set = set & MOUNT_ATTR__ATIME.
6. if popcount(atime_set) > 1: Err(EINVAL).
7. if (clr & MOUNT_ATTR__ATIME) != 0 && (clr & MOUNT_ATTR__ATIME) != MOUNT_ATTR__ATIME: Err(EINVAL).
8. /* IDMAP */
9. let mnt_userns = if set & MOUNT_ATTR_IDMAP != 0 {
     let fd = attr.userns_fd as i32;
     let ns = userns_from_fd(fd)?;
     ns_check_parent_or_self(current().cred.user_ns, ns)?;
     Some(ns)
   } else if attr.userns_fd != 0 {
     return Err(EINVAL);
   } else { None };
10. /* IDMAP cannot be in attr_clr */
11. if clr & MOUNT_ATTR_IDMAP != 0: Err(EINVAL).
12. Ok(MountKattr { set, clr, propagation: attr.propagation, mnt_userns }).
```

`MountSetattr::prepare(mount, kattr, recursive)`:
- for m in iter_subtree(mount, recursive):
  - if m.mnt_ns != current_mnt_ns: Err(EINVAL).
  - if !can_change_locked(m, kattr): Err(EPERM).
  - if kattr.set & MOUNT_ATTR_IDMAP != 0:
    - if m.mnt_sb.s_type.flags & FS_ALLOW_IDMAP == 0: Err(EINVAL).
    - if m.mnt_userns != initial_user_ns: Err(EBUSY).
- Ok.

`MountSetattr::commit(mount, kattr, recursive)`:
- for m in iter_subtree(mount, recursive):
  - apply_mnt_flag_changes(m, kattr.set, kattr.clr).
  - if let Some(ns) = kattr.mnt_userns: m.mnt_userns = ns (acquire ref); m.flags |= MNT_IDMAPPED.
  - if kattr.propagation != 0: change_mnt_propagation(m, kattr.propagation).

### Out of Scope

- Per-FS implementation of FS_ALLOW_IDMAP (covered in fs/<name>/idmap.md).
- nsfs fd creation (covered in `kernel/nsfs.md`).
- pivot_root / unshare / setns interactions (covered in their own Tier-5 docs).
- mount() classic syscall (covered in `mount.md`).
- Implementation code.

### signature

```c
int mount_setattr(int dirfd,
                  const char *path,
                  unsigned int flags,
                  struct mount_attr *attr,
                  size_t size);

struct mount_attr {
    __u64 attr_set;       /* MOUNT_ATTR_* to set */
    __u64 attr_clr;       /* MOUNT_ATTR_* to clear */
    __u64 propagation;    /* MS_SHARED/PRIVATE/SLAVE/UNBINDABLE, 0 = leave */
    __u64 userns_fd;      /* fd of userns for MOUNT_ATTR_IDMAP */
};
```

Per-x86_64-syscall-table: `__NR_mount_setattr = 442`. Per-glibc: thin wrapper since 2.36. Per-vDSO: not vDSO-vectored.

### parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `dirfd` | `int` | in | Base fd for `path` lookup; AT_FDCWD for cwd-relative. |
| `path` | `const char *` | in | Path to mount; relative to `dirfd` unless absolute. |
| `flags` | `unsigned int` | in | AT_EMPTY_PATH, AT_RECURSIVE, AT_SYMLINK_NOFOLLOW, AT_NO_AUTOMOUNT. |
| `attr` | `struct mount_attr *` | in | Attribute change descriptor. |
| `size` | `size_t` | in | sizeof(struct mount_attr) the caller built; extensible struct (per-copy_struct_from_user). |

### return

- 0 on success (entire subtree updated when AT_RECURSIVE).
- -1 with `errno` on failure (no partial changes).

### errors

| errno | Cause |
|---|---|
| EPERM | Caller lacks CAP_SYS_ADMIN in mnt_ns.user_ns; or MOUNT_ATTR_IDMAP without rights over userns_fd. |
| EBADF | `dirfd` invalid; or `userns_fd` invalid for MOUNT_ATTR_IDMAP. |
| EINVAL | Unknown flag bit; conflicting attr_set/attr_clr; unknown propagation value; size invalid; idmap on already-idmapped mount; idmap on non-supporting fs; attr_set has reserved bits. |
| ENOENT | `path` does not resolve. |
| ENOTDIR | Component not a directory; or AT_RECURSIVE on non-directory mountpoint. |
| ENAMETOOLONG | path > PATH_MAX. |
| EFAULT | `path` or `attr` outside accessible address space. |
| E2BIG | `size` larger than kernel knows and trailing bytes non-zero. |
| ENOSPC | Out of internal idmap reference slots. |
| EBUSY | userns_fd refers to userns already in use as idmap on this mount. |
| ENOMEM | Kernel allocation failed. |

### abi surface

- Syscall number: `__NR_mount_setattr = 442` (since Linux 5.12).
- `struct mount_attr` is **extensible** (copy_struct_from_user): unknown trailing bytes must be zero; zero-filled if caller's size < kernel's size.
- MOUNT_ATTR_* bits (u64): RDONLY=0x1, NOSUID=0x2, NODEV=0x4, NOEXEC=0x8, _ATIME=0x70 (mask), RELATIME=0x0, NOATIME=0x10, STRICTATIME=0x20, NODIRATIME=0x80, IDMAP=0x100000, NOSYMFOLLOW=0x200000.
- `propagation` reuses MS_SHARED/MS_PRIVATE/MS_SLAVE/MS_UNBINDABLE numeric values.
- `userns_fd` valid only when MOUNT_ATTR_IDMAP in attr_set.

### compatibility contract

REQ-1: `flags` accepted bits: AT_EMPTY_PATH (path == ""), AT_RECURSIVE, AT_SYMLINK_NOFOLLOW, AT_NO_AUTOMOUNT. Any other bit ⟹ EINVAL.

REQ-2: `size` handling via copy_struct_from_user(attr, MOUNT_ATTR_SIZE_VER0, user_attr, size):
- size < MOUNT_ATTR_SIZE_VER0 ⟹ EINVAL.
- size > sizeof(struct mount_attr): trailing bytes must be zero ⟹ otherwise E2BIG.
- short copy: zero-extend the kernel-side struct.

REQ-3: attr_set ∩ attr_clr per-_ATIME group:
- _ATIME mask validation: at most one of NOATIME/RELATIME/STRICTATIME set; cleared bits subset of mask.
- attr_set & MOUNT_ATTR__ATIME and attr_clr & MOUNT_ATTR__ATIME may not overlap (EINVAL).

REQ-4: Reserved bits in attr_set or attr_clr: ~(MOUNT_ATTR_RDONLY|NOSUID|NODEV|NOEXEC|_ATIME|NODIRATIME|IDMAP|NOSYMFOLLOW) ⟹ EINVAL.

REQ-5: `propagation` must be 0 or exactly one of MS_SHARED, MS_PRIVATE, MS_SLAVE, MS_UNBINDABLE. Other ⟹ EINVAL.

REQ-6: Per-MOUNT_ATTR_IDMAP:
- attr_set & MOUNT_ATTR_IDMAP requires userns_fd >= 0.
- userns_fd must be an `nsfs` fd referencing a user namespace (FD_TYPE_USER_NS).
- Caller's user_ns must be a parent of or equal to the target userns (capable check).
- mount must not already be idmapped (one-shot; EBUSY otherwise).
- super_block must opt-in via FS_ALLOW_IDMAP; otherwise EINVAL.
- idmap is **immutable** after install (no MOUNT_ATTR_IDMAP in attr_clr).

REQ-7: Per-prepare/commit:
- mount_setattr_prepare: walk subtree (if AT_RECURSIVE), validate every mount against attr; if any fails, abort.
- mount_setattr_commit: apply changes under namespace_lock.
- Atomic: subtree-wide single critical section.

REQ-8: Per-attribute-to-MNT mapping:
- MOUNT_ATTR_RDONLY ⟹ MNT_READONLY (note: SB_RDONLY unchanged; only mount visibility flipped).
- MOUNT_ATTR_NOSUID ⟹ MNT_NOSUID.
- MOUNT_ATTR_NODEV ⟹ MNT_NODEV.
- MOUNT_ATTR_NOEXEC ⟹ MNT_NOEXEC.
- MOUNT_ATTR_NOATIME ⟹ MNT_NOATIME (clears RELATIME).
- MOUNT_ATTR_RELATIME ⟹ MNT_RELATIME (clears NOATIME).
- MOUNT_ATTR_STRICTATIME ⟹ clear MNT_NOATIME, clear MNT_RELATIME.
- MOUNT_ATTR_NODIRATIME ⟹ MNT_NODIRATIME.
- MOUNT_ATTR_NOSYMFOLLOW ⟹ MNT_NOSYMFOLLOW.
- MOUNT_ATTR_IDMAP ⟹ mount.mnt_userns set + mnt_idmapped flag.

REQ-9: Per-CAP_SYS_ADMIN required in mnt_ns.user_ns (not target mount's mnt_userns).

REQ-10: Per-MNT_LOCKED interaction:
- Mount cloned by less-privileged unshare carries MNT_LOCKED for select bits; attr_clr that would clear a locked bit returns EPERM.

REQ-11: Per-AT_RECURSIVE on locked subtree: EPERM if any descendant locked against requested change.

REQ-12: Per-validate idmap fd consumed exactly once across the call; fd remains open after syscall (semantics like SCM_RIGHTS — kernel takes its own ref on the user_ns).

REQ-13: Per-mount belonging to mount-namespace other than caller's (e.g. bind from foreign ns): EINVAL.

REQ-14: Per-pseudofs (procfs, sysfs, etc) typically rejects MOUNT_ATTR_IDMAP (FS_ALLOW_IDMAP absent).

REQ-15: Per-empty path: AT_EMPTY_PATH ∧ path == "" ⟹ dirfd is the mount.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | per-sys_mount_setattr: only known AT_* bits accepted. |
| `attr_extensible_zero_tail` | INVARIANT | per-copy_struct_from_user: trailing bytes zero or E2BIG. |
| `idmap_oneshot` | INVARIANT | per-prepare: mount.mnt_userns must be initial before IDMAP. |
| `idmap_immutable` | INVARIANT | per-build_mount_kattr: MOUNT_ATTR_IDMAP forbidden in attr_clr. |
| `atomic_subtree` | INVARIANT | per-prepare-fail: no commit performed. |
| `caps_required` | INVARIANT | per-sys_mount_setattr: CAP_SYS_ADMIN precedes any state change. |
| `locked_bits_respected` | INVARIANT | per-can_change_locked: MNT_LOCKED bit clear ⟹ EPERM. |

### Layer 2: TLA+

`uapi/mount_setattr.tla`:
- States: subtree mount-flag-tuples, idmap-state, propagation-state.
- Properties:
  - `safety_atomic` — per-call: every subtree mount changed iff none failed.
  - `safety_idmap_immutable` — per-mount.mnt_userns: monotone (initial → set; never cleared).
  - `safety_propagation_single` — per-call: at most one propagation type applied.
  - `safety_locked_bits` — per-MNT_LOCKED: never relaxed.
  - `liveness_terminates` — per-call: returns success or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MountSetattr::build_mount_kattr` post: kattr fields each within accepted mask | `MountSetattr::build_mount_kattr` |
| `MountSetattr::prepare` post: either Ok with no commit prerequisite violated, or Err leaving state untouched | `MountSetattr::prepare` |
| `MountSetattr::commit` post: ∀ m in subtree: mnt_flags reflects (set ∪ (orig − clr)) | `MountSetattr::commit` |
| `userns_from_fd` post: returned ns has +1 refcount; released by Drop on error path | `userns_from_fd` |

### Layer 4: Verus/Creusot functional

Per-mount_setattr semantic equivalence with upstream `do_mount_setattr` + `mount_setattr_prepare`/`commit`: per-Documentation/filesystems/idmappings.rst.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

mount_setattr-syscall reinforcement:

- **Per-extensible-struct trailing-zero rule (E2BIG)** — defense against per-uninitialized-stack leak into kernel.
- **Per-IDMAP one-shot install** — defense against per-userns-swap TOCTOU.
- **Per-IDMAP forbidden in attr_clr (immutability)** — defense against per-idmap-rollback attack.
- **Per-FS_ALLOW_IDMAP gate** — defense against per-procfs/sysfs idmap confusion.
- **Per-MNT_LOCKED respected** — defense against per-unprivileged-flag-relax via unshare-pivot.
- **Per-subtree atomicity (prepare/commit)** — defense against per-partial-update inconsistency.
- **Per-CAP_SYS_ADMIN in mnt_ns.user_ns** — defense against per-cross-namespace privilege misuse.
- **Per-userns_fd validated FD_TYPE_USER_NS** — defense against per-fd-type confusion.
- **Per-PAX UDEREF on attr copy_from_user** — defense against per-kernel-pointer fixup race.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-attr.
- **Per-GRKERNSEC_CHROOT_MOUNT applies to mount_setattr** — chroot'd tasks blocked from changing mount attributes.
- **Per-atime-bit mutex (NOATIME⊕RELATIME⊕STRICTATIME)** — defense against per-undefined-behavior on conflicting attrs.
- **Per-reserved-bit reject on attr_set/clr** — defense against per-future-flag-conflation.
- **Per-namespace-lock around prepare+commit** — defense against per-concurrent-mount-mutation races.

