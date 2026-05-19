# Tier-5: syscall 432 — fsmount(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/namespace.c (sys_fsmount, do_new_mount_fc, vfs_create_mount)
  - fs/fs_context.c (vfs_get_tree)
  - include/uapi/linux/mount.h (FSMOUNT_CLOEXEC, MOUNT_ATTR_*)
-->

## Summary

`fsmount(2)` converts a configured fs_context (built via `fsopen` + `fsconfig`) into a **detached mount** referenced by a new fd. The mount is not yet attached to any mountpoint — `move_mount(2)` performs that step. Per-FSMOUNT_CLOEXEC sets O_CLOEXEC on the returned fd. Per-`attr_flags` carries MOUNT_ATTR_* (RDONLY, NOSUID, NODEV, NOEXEC, atime variants, NOSYMFOLLOW) for the resulting mount. Per-fsmount is the materialization point at which the superblock has been instantiated (via FSCONFIG_CMD_CREATE in fsconfig); the returned fd represents an `fsmount` file holding a struct vfsmount. Critical for: programmatic mount composition, fd-based mount handoff, atomic mount-attribute selection.

This Tier-5 covers `syscall 432 fsmount`.

## Signature

```c
int fsmount(int fs_fd, unsigned int flags, unsigned int attr_flags);
```

Per-x86_64-syscall-table: `__NR_fsmount = 432`. Per-glibc: direct syscall wrapper. Per-vDSO: not vDSO-vectored.

## Parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `fs_fd` | `int` | in | fscontext fd from fsopen(2) that has been configured and `fsconfig(FSCONFIG_CMD_CREATE)`'d. |
| `flags` | `unsigned int` | in | 0 or FSMOUNT_CLOEXEC (0x1). |
| `attr_flags` | `unsigned int` | in | MOUNT_ATTR_* mask: RDONLY, NOSUID, NODEV, NOEXEC, _ATIME (NOATIME/RELATIME/STRICTATIME), NODIRATIME, NOSYMFOLLOW. |

## Return

- ≥ 0: new fsmount fd (anon-inode) representing a detached mount.
- -1 with `errno` on failure.

## Errors

| errno | Cause |
|---|---|
| EBADF | `fs_fd` invalid. |
| EINVAL | `fs_fd` is not an fscontext fd; fc not yet at FS_CONTEXT_AWAITING_MOUNT state (CMD_CREATE not called); `flags` has bits other than FSMOUNT_CLOEXEC; `attr_flags` has unknown bits or conflicting atime bits. |
| EPERM | Caller lacks CAP_SYS_ADMIN in fc.user_ns. |
| ENOMEM | Allocation failed. |
| EMFILE / ENFILE | Fd table limits. |
| EBUSY | fsmount already called on this fc (one-shot). |

## ABI surface

- Syscall number: `__NR_fsmount = 432` (since 5.2).
- Flags: FSMOUNT_CLOEXEC = 0x1.
- attr_flags bit layout: identical to `struct mount_attr.attr_set` MOUNT_ATTR_* (low 24 bits used).
- Returned fd: anon-inode "fsmount"; `f_op = fsmount_fops` (release only); private_data holds `struct vfsmount *`.
- The fsmount fd is consumed (by reference) via `move_mount(2)` to attach the mount to a path.

## Compatibility contract

REQ-1: fs_fd validation:
- fget(fs_fd) returns file*.
- file.f_op == &fscontext_fops; else EINVAL.
- fc = file.private_data.

REQ-2: fc state precondition:
- fc.phase == FS_CONTEXT_AWAITING_MOUNT (i.e. FSCONFIG_CMD_CREATE has been issued, vfs_get_tree succeeded, fc.root is set).
- Otherwise EINVAL.

REQ-3: Flag validation:
- flags & !FSMOUNT_CLOEXEC ⟹ EINVAL.
- attr_flags & !MOUNT_ATTR_VALID_MASK ⟹ EINVAL.

REQ-4: atime mutex:
- popcount(attr_flags & MOUNT_ATTR__ATIME) ≤ 1; otherwise EINVAL.

REQ-5: Capability:
- ns_capable(fc.user_ns, CAP_SYS_ADMIN). Use fc.cred saved at fsopen-time as the authorizing identity.

REQ-6: vfsmount creation:
- mnt = vfs_create_mount(fc) — allocates struct mount, sets mnt.mnt_sb = fc.root.dentry.d_sb, mnt.mnt_root = fc.root.dentry, increments super ref.
- mnt.mnt_userns = fc.user_ns.

REQ-7: Apply attr_flags to mnt.mnt_flags:
- MOUNT_ATTR_RDONLY     ⟹ MNT_READONLY.
- MOUNT_ATTR_NOSUID     ⟹ MNT_NOSUID.
- MOUNT_ATTR_NODEV      ⟹ MNT_NODEV.
- MOUNT_ATTR_NOEXEC     ⟹ MNT_NOEXEC.
- MOUNT_ATTR_NOATIME    ⟹ MNT_NOATIME (clears MNT_RELATIME).
- MOUNT_ATTR_RELATIME   ⟹ MNT_RELATIME.
- MOUNT_ATTR_STRICTATIME⟹ clear MNT_NOATIME|MNT_RELATIME.
- MOUNT_ATTR_NODIRATIME ⟹ MNT_NODIRATIME.
- MOUNT_ATTR_NOSYMFOLLOW⟹ MNT_NOSYMFOLLOW.

REQ-8: One-shot:
- After successful fsmount, fc.phase = FS_CONTEXT_DEACTIVATED (or similar terminal state). Subsequent fsmount on same fc ⟹ EBUSY.

REQ-9: fd allocation:
- get_unused_fd_flags(O_RDWR | (flags & FSMOUNT_CLOEXEC ? O_CLOEXEC : 0)).

REQ-10: file allocation:
- anon_inode_getfile("fsmount", &fsmount_fops, mnt, O_RDWR).
- fsmount fops have only release (mntput on close) and llseek=no_llseek; no read/write/poll/ioctl.

REQ-11: fd install:
- fd_install(fd, file).

REQ-12: Detached mount:
- mnt belongs to no mnt_ns yet. mnt.mnt_ns = NULL.
- attach happens via move_mount(2).

REQ-13: Per-fc.cred preserves opener identity:
- ns_capable check uses fc.cred->user_ns (the saved one), not current's. Allows fsopen-as-root + fd-passing + fsmount-as-nobody patterns when fc.cred dictates.

REQ-14: Per-error rollback:
- If anon_inode_getfile fails: mntput(mnt); fc.phase restored to FS_CONTEXT_AWAITING_MOUNT.

## Acceptance Criteria

- [ ] AC-1: fsopen→fsconfig(set source/options)→fsconfig(CMD_CREATE)→fsmount: returns valid fsmount fd.
- [ ] AC-2: fsmount with FSMOUNT_CLOEXEC: fd has FD_CLOEXEC.
- [ ] AC-3: fsmount before CMD_CREATE: EINVAL.
- [ ] AC-4: fsmount called twice on same fc: second call EBUSY.
- [ ] AC-5: fsmount with attr_flags = MOUNT_ATTR_RDONLY|NOSUID: mount has MNT_READONLY|MNT_NOSUID.
- [ ] AC-6: attr_flags with NOATIME|RELATIME both set: EINVAL.
- [ ] AC-7: Unknown bit in attr_flags: EINVAL.
- [ ] AC-8: flags = 0x2: EINVAL.
- [ ] AC-9: fs_fd is not fscontext: EINVAL.
- [ ] AC-10: fs_fd invalid: EBADF.
- [ ] AC-11: Non-CAP_SYS_ADMIN in fc.user_ns: EPERM.
- [ ] AC-12: Read/write on fsmount fd: EINVAL/ESPIPE per fsmount_fops.
- [ ] AC-13: close(fsmount_fd) before move_mount: mnt freed (mntput).

## Architecture

```
Fsmount::sys_fsmount(fs_fd: i32, flags: u32, attr_flags: u32) -> Result<i32, Errno>
```

1. /* Flag validation */
2. if flags & !FSMOUNT_CLOEXEC != 0: return Err(EINVAL).
3. if attr_flags & !MOUNT_ATTR_VALID_MASK != 0: return Err(EINVAL).
4. if popcount(attr_flags & MOUNT_ATTR__ATIME) > 1: return Err(EINVAL).
5. /* Resolve fc */
6. let file = fget(fs_fd).ok_or(EBADF)?;
7. let fc = match fscontext_from_file(&file) {
     Some(fc) => fc,
     None => { fput(file); return Err(EINVAL); }
   };
8. /* Capability via fc.cred */
9. if !ns_capable_cred(&fc.cred, fc.user_ns, CAP_SYS_ADMIN):
     { fput(file); return Err(EPERM); }
10. /* Phase precondition */
11. lock_fc(&fc);
12. if fc.phase != FsContextPhase::AwaitingMount: { unlock_fc; fput; return Err(EINVAL); }
13. /* Materialize vfsmount */
14. let mnt = match vfs_create_mount(&fc) {
      Ok(m) => m,
      Err(e) => { unlock_fc; fput; return Err(e); }
    };
15. /* Apply attr_flags */
16. apply_mount_attr(&mut mnt, attr_flags);
17. /* Phase transition (one-shot) */
18. fc.phase = FsContextPhase::AwaitingMove;
19. unlock_fc(&fc);
20. fput(file);
21. /* fd + file */
22. let cloexec = flags & FSMOUNT_CLOEXEC != 0;
23. let fd = match get_unused_fd_flags(O_RDWR | if cloexec { O_CLOEXEC } else { 0 }) {
      Ok(fd) => fd,
      Err(e) => { mntput(mnt); return Err(e); }
    };
24. let mnt_file = match anon_inode_getfile("fsmount", &FSMOUNT_FOPS, mnt.clone(), O_RDWR) {
      Ok(f) => f,
      Err(e) => { put_unused_fd(fd); mntput(mnt); return Err(e); }
    };
25. fd_install(fd, mnt_file);
26. Ok(fd as i32).
```

`apply_mount_attr(mnt, attr_flags)`:
- if attr_flags & MOUNT_ATTR_RDONLY:     mnt.mnt_flags |= MNT_READONLY.
- if attr_flags & MOUNT_ATTR_NOSUID:     mnt.mnt_flags |= MNT_NOSUID.
- if attr_flags & MOUNT_ATTR_NODEV:      mnt.mnt_flags |= MNT_NODEV.
- if attr_flags & MOUNT_ATTR_NOEXEC:     mnt.mnt_flags |= MNT_NOEXEC.
- if attr_flags & MOUNT_ATTR_NOATIME:    mnt.mnt_flags = (mnt.mnt_flags & !MNT_RELATIME) | MNT_NOATIME.
- if attr_flags & MOUNT_ATTR_RELATIME:   mnt.mnt_flags = (mnt.mnt_flags & !MNT_NOATIME) | MNT_RELATIME.
- if attr_flags & MOUNT_ATTR_STRICTATIME:mnt.mnt_flags &= !(MNT_NOATIME|MNT_RELATIME).
- if attr_flags & MOUNT_ATTR_NODIRATIME: mnt.mnt_flags |= MNT_NODIRATIME.
- if attr_flags & MOUNT_ATTR_NOSYMFOLLOW:mnt.mnt_flags |= MNT_NOSYMFOLLOW.

`FsmountFops`:
- release = fsmount_release  // mntput
- llseek = no_llseek
- read/write/poll/ioctl = NULL (return EINVAL/ESPIPE)

`vfs_create_mount(fc)`:
1. mnt = alloc_vfsmount(fc.fs_type)?;
2. mnt.mnt_sb = fc.root.dentry.d_sb;          // already created
3. mnt.mnt_root = fc.root.dentry;
4. mnt.mnt_userns = fc.user_ns.clone();
5. atomic_inc(&fc.root.dentry.d_sb.s_active);
6. dget(fc.root.dentry);
7. Ok(mnt).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_validated` | INVARIANT | per-sys_fsmount: only FSMOUNT_CLOEXEC accepted. |
| `attr_flags_validated` | INVARIANT | per-sys_fsmount: bits ⊆ MOUNT_ATTR_VALID_MASK. |
| `atime_mutex` | INVARIANT | per-sys_fsmount: ≤ 1 atime bit. |
| `phase_awaiting_mount` | INVARIANT | per-sys_fsmount: fc.phase == AwaitingMount on entry. |
| `one_shot` | INVARIANT | per-sys_fsmount: phase transition to AwaitingMove; repeat ⟹ EBUSY. |
| `caps_via_fc_cred` | INVARIANT | per-sys_fsmount: ns_capable uses fc.cred->user_ns. |
| `mntput_on_error` | INVARIANT | per-sys_fsmount: error after vfs_create_mount drops mnt ref. |
| `fd_install_after_anon_file` | INVARIANT | per-sys_fsmount: anon_inode_getfile precedes fd_install; either both succeed or fd reservation released. |

### Layer 2: TLA+

`uapi/fsmount.tla`:
- States: fc.phase, mnt detached/attached, fd-table.
- Properties:
  - `safety_phase_progression` — per-fc: AwaitingMount → AwaitingMove monotonic.
  - `safety_mnt_ref_balance` — per-call: success ⟹ mnt ref held by file; failure ⟹ mnt ref dropped or never taken.
  - `safety_attr_applied` — per-call: mnt.mnt_flags matches attr_flags translation.
  - `safety_cloexec_fidelity` — per-FSMOUNT_CLOEXEC.
  - `liveness_terminates` — per-call: returns fd or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Fsmount::sys_fsmount` post: success ⟹ current.files.fdt[fd] points to fsmount file holding mnt | `Fsmount::sys_fsmount` |
| `vfs_create_mount` post: mnt.mnt_sb == fc.root.d_sb; mnt.mnt_root == fc.root.dentry; sb.s_active += 1 | `vfs_create_mount` |
| `apply_mount_attr` post: ∀ bit b in attr_flags ⟹ corresponding MNT_* bit set | `apply_mount_attr` |
| `fc.phase_transition` post: AwaitingMount → AwaitingMove | fc phase machine |

### Layer 4: Verus/Creusot functional

Per-fsmount semantic equivalence with upstream `sys_fsmount` + `vfs_create_mount`: per-Documentation/filesystems/mount_api.rst.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

fsmount-syscall reinforcement:

- **Per-fc.cred capability source** — defense against per-credential-stripping attack via fd-passing.
- **Per-one-shot phase transition** — defense against per-double-materialization race.
- **Per-attr_flags strict mask validation** — defense against per-future-flag-conflation.
- **Per-atime-bit mutex** — defense against per-undefined-behavior on conflicting attrs.
- **Per-MOUNT_ATTR_NOSUID|NODEV|NOEXEC|NOSYMFOLLOW applied at materialization** — defense against per-attack-fs-on-mount; the receiver decides policy.
- **Per-idmap left to mount_setattr** — defense against per-attribute-channel-confusion (idmap not via attr_flags).
- **Per-mnt_userns = fc.user_ns** — defense against per-cross-userns mount confusion.
- **Per-anon_inode_getfile pairs with fd reservation** — defense against per-half-open file leak.
- **Per-FSMOUNT_CLOEXEC honored** — defense against per-exec-leak of detached mount fd.
- **Per-fsmount_fops minimal (no read/write/poll)** — defense against per-fd-data-leak.
- **Per-PAX UDEREF on syscall args** — defense against per-kernel-pointer fixup race.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-fsmount.
- **Per-GRKERNSEC_CHROOT_MOUNT: chrooted caller blocked** — defense against per-chroot-escape via detached-mount synthesis.
- **Per-mnt detached until move_mount** — defense against per-premature-visibility in mnt_ns.
- **Per-sb.s_active counter atomic** — defense against per-superblock UAF on fc destruction.

## Grsecurity/PaX-style Reinforcement

This syscall inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — `fsmount(2)` argument vector (`fd`, `flags`, `attr_flags`) is fixed-width; no variable-length copy_from_user — attack surface restricted to enum/flag validation.
- **PAX_KERNEXEC** — `do_fsmount`, `vfs_create_mount`, `mnt_alloc_id`, and `init_mount_idmapped` reside in RX `.text`; per-superblock fs_type vtable resolved via kCFI-signed indirection.
- **PAX_RANDKSTACK** — kstack-offset randomization on syscall entry; defense against ROP-via-fsmount (REQ-PAX_RANDKSTACK).
- **PAX_REFCOUNT** — saturating refcount on `struct mount` (`mnt_count`, `mnt_writers`), `super_block.s_active`, and the originating `fs_context`; defense against detached-mount UAF on dup-and-drop.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `struct mount`, `mnt_idmap`, and detached `mount_kattr` scratch on fsmount failure path.
- **PAX_UDEREF (SMAP/SMEP)** — ASM_CLAC on syscall args copy; `attr_flags` mask validated against canonical `MOUNT_ATTR_*` set; unknown bits rejected with -EINVAL.
- **PAX_RAP / kCFI** — `fs_context_operations` and `super_operations` callbacks reachable from fsmount dispatched via kCFI-signed indirect calls.
- **GRKERNSEC_HIDESYM** — `mount_lock`, `mnt_id_ida`, `fs_context` symbols stripped from /proc/kallsyms for non-CAP_SYSLOG.
- **GRKERNSEC_DMESG** — fsmount failure / superblock-mismatch / attr-validation diagnostics restricted to CAP_SYSLOG.
- **CAP_SYS_ADMIN strict** — `fsmount(2)` gated on caller's CAP_SYS_ADMIN in init_user_ns or per-mount-ns equivalent; capable mount only after `fc->users` validation.
- **MOUNT_ATTR_* mask strict-check** — out-of-range attribute bits rejected before mount-instance materialization; defense against future-flag smuggling.
- **GRKERNSEC_CHROOT_MOUNT** — chrooted caller blocked from fsmount; defense against chroot-escape via detached-mount synthesis.
- **Detached-until-move_mount(2)** — fsmount-produced fd refers to a detached mount not in any mnt_ns until `move_mount(2)` attaches it; defense against premature visibility.
- **`sb.s_active` counter atomic** — defense against superblock UAF on fs_context destruction race.

Per-syscall rationale: `fsmount(2)` materializes the configured `fs_context` into a kernel mount object — the moment of trust where parser-validated state becomes a live mount. A REFCOUNT wrap on `mount` or a stale `fs_context` pointer would let an attacker resurrect a freed superblock. Detached-until-move_mount + saturating refcount + CAP_SYS_ADMIN gate + GRKERNSEC_CHROOT_MOUNT is the layered defense against the mount-API privilege-boundary class.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `fsopen(2)` context creation (covered in `fsopen.md`).
- `fsconfig(2)` parameter setting (covered in `fsconfig.md`).
- `move_mount(2)` attach to mountpoint (covered separately).
- `mount_setattr(2)` post-mount attribute change incl. idmap (covered in `mount_setattr.md`).
- Implementation code.
