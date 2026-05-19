# Tier-5: syscall 165 — mount(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/namespace.c (do_mount, path_mount, do_new_mount, do_remount, do_loopback, do_move_mount, do_change_type)
  - include/uapi/linux/mount.h (MS_*)
  - include/linux/fs_context.h
  - Documentation/filesystems/mount_api.rst
-->

## Summary

`mount(2)` attaches a filesystem (identified by `source` + `fstype` + optional `data`) onto the directory tree at `target`. Per-MS_* flag selects mode: per-new-mount (default), per-bind (MS_BIND), per-remount (MS_REMOUNT), per-move (MS_MOVE), per-type-change (MS_SHARED / MS_PRIVATE / MS_SLAVE / MS_UNBINDABLE). Per-fstype dispatches to `file_system_type.mount` (legacy) or the new `init_fs_context` API (fsopen/fsconfig/fsmount). Per-caller must hold CAP_SYS_ADMIN in the mount namespace's owning user namespace. Critical for: filesystem composition, container root assembly, bind-mount semantics, propagation trees.

This Tier-5 covers `syscall 165 mount`.

## Signature

```c
int mount(const char *source,
          const char *target,
          const char *fstype,
          unsigned long flags,
          const void *data);
```

Per-x86_64-syscall-table: `__NR_mount = 165`. Per-glibc: `sys/mount.h`. Per-vDSO: not vDSO-vectored.

## Parameters

| Name | Type | Direction | Meaning |
|---|---|---|---|
| `source` | `const char *` | in | Device path (e.g. `/dev/sda1`), source path (bind), `none`/`tmpfs`/etc per-fs, or NULL when ignored. |
| `target` | `const char *` | in | Mountpoint path; must exist and be a directory (or file for bind-mount-on-file). |
| `fstype` | `const char *` | in | Filesystem type (e.g. `ext4`, `tmpfs`, `proc`); ignored for MS_BIND / MS_MOVE / MS_REMOUNT / propagation-only. |
| `flags` | `unsigned long` | in | MS_* bitfield: MS_RDONLY, MS_NOSUID, MS_NODEV, MS_NOEXEC, MS_SYNCHRONOUS, MS_REMOUNT, MS_MANDLOCK, MS_DIRSYNC, MS_NOSYMFOLLOW, MS_NOATIME, MS_NODIRATIME, MS_BIND, MS_MOVE, MS_REC, MS_SILENT, MS_POSIXACL, MS_UNBINDABLE, MS_PRIVATE, MS_SLAVE, MS_SHARED, MS_RELATIME, MS_KERNMOUNT, MS_I_VERSION, MS_STRICTATIME, MS_LAZYTIME, MS_NOREMOTELOCK, MS_NOSEC, MS_BORN, MS_ACTIVE, MS_NOUSER. |
| `data` | `const void *` | in | FS-specific options (typically a comma-separated key=value string); upper-bounded by PAGE_SIZE; may be NULL. |

## Return

- 0 on success.
- -1 with `errno` set on failure.

## Errors

| errno | Cause |
|---|---|
| EPERM | Caller lacks CAP_SYS_ADMIN in the owning user namespace. |
| EACCES | Component of path search-denied, or RO-fs and MS_RDONLY not requested where required. |
| ENOENT | `target` or `source` does not exist. |
| ENOTDIR | Component of path not a directory; or `target` not directory for non-bind mount. |
| EBUSY | Already mounted, or source is in use, or remount conflict. |
| EINVAL | Invalid flag combination (e.g. MS_BIND|MS_REMOUNT misuse), unknown fstype option, or `data` malformed. |
| ENODEV | `fstype` not registered. |
| ENOMEM | Out of kernel memory. |
| ENAMETOOLONG | Path > PATH_MAX. |
| ELOOP | Symlink loop on lookup. |
| EROFS | Underlying device is read-only and MS_RDONLY missing for new mount. |
| EFAULT | `source`, `target`, `fstype`, or `data` outside accessible address space. |
| ENXIO | Device major number out of range. |
| EMFILE | Per-namespace mount count exceeded sysctl `fs.mount-max`. |

## ABI surface

- Syscall number: `__NR_mount = 165` (x86_64, arm64, riscv: per-arch table preserves number).
- Argument register order per System V AMD64 ABI: rdi=source, rsi=target, rdx=fstype, r10=flags, r8=data.
- All pointer arguments are `__user`; strings copied via `strndup_user` bounded by PATH_MAX (PAGE_SIZE for data).
- MS_* bit layout fixed since 2.4; new flags added at high bits (MS_NOSYMFOLLOW = 1<<8 since 5.10, MS_LAZYTIME = 1<<25).
- Internal MS_KERNMOUNT, MS_BORN, MS_ACTIVE never accepted from userspace (rejected pre-VFS).

## Compatibility contract

REQ-1: Flag dispatch precedence (mutually-exclusive groups):
- MS_REMOUNT ⟹ do_remount (no fstype lookup; reuse existing super_block).
- MS_BIND ⟹ do_loopback (clones existing mount; fstype ignored).
- MS_MOVE ⟹ do_move_mount (relocates an existing mount).
- MS_SHARED|MS_PRIVATE|MS_SLAVE|MS_UNBINDABLE ⟹ do_change_type (propagation only).
- Otherwise ⟹ do_new_mount (fstype required).

REQ-2: Combining flags rejected with EINVAL:
- MS_REMOUNT + MS_BIND requires MS_REMOUNT|MS_BIND only with per-mount-attr (atime/RO) reflag.
- MS_BIND + MS_MOVE.
- MS_SHARED + MS_SLAVE.
- Per-type-change + (MS_BIND | MS_MOVE | MS_REMOUNT).

REQ-3: Per-MS_REC applies recursively to:
- MS_BIND (rbind).
- MS_PRIVATE/MS_SHARED/MS_SLAVE/MS_UNBINDABLE (subtree propagation change).

REQ-4: Per-CAP_SYS_ADMIN check:
- In mnt_ns->user_ns: must hold capability.
- Initial user-ns: ns_capable(init_user_ns, CAP_SYS_ADMIN).
- Per-MS_REMOUNT|MS_BIND for existing-mount attribute-only-change: still CAP_SYS_ADMIN.

REQ-5: Per-new-mount path (do_new_mount):
- /* Lookup fs_type */
- type = get_fs_type(fstype). EINVAL ⟹ ENODEV.
- /* fs_context init (new API) or legacy mount */
- if type->init_fs_context:
  - fc = fs_context_for_mount(type, sb_flags).
  - vfs_parse_fs_string(fc, "source", source).
  - vfs_parse_fs_string(fc, key, val) for each `data` kv.
  - vfs_get_tree(fc).
  - do_new_mount_fc(fc, path, mnt_flags).
- else (legacy):
  - mount_fs(type, sb_flags, source, data).
- put_filesystem(type).

REQ-6: Per-remount (do_remount):
- /* Cannot change fstype */
- Reuses existing mount->mnt.mnt_sb.
- Calls fc->ops->reconfigure(fc) (new API) or super_block.s_op->remount_fs (legacy).
- Per-MS_RDONLY toggle: must not have open-rw files when going RO (EBUSY).
- Per-MS_NOSUID|MS_NODEV|MS_NOEXEC|MS_NOATIME|MS_NODIRATIME|MS_RELATIME|MS_STRICTATIME|MS_NOSYMFOLLOW: mount-attribute change only.

REQ-7: Per-bind (do_loopback):
- /* source resolves to an existing mount path */
- old = user_path_at(AT_FDCWD, source).
- /* Clone */
- mnt = clone_mnt(old.mnt, old.dentry, CL_*).
- /* MS_REC ⟹ clone subtree */
- if MS_REC: copy_tree(old, dentry, CL_*).
- /* Attach */
- attach_recursive_mnt(mnt, path, NULL, false).

REQ-8: Per-move (do_move_mount):
- /* source is existing mount root */
- old = user_path_at(...).
- /* Must be a mount root */
- if old.dentry != old.mnt.mnt_root: EINVAL.
- /* Cannot move root mount */
- if old.mnt == old.mnt.mnt_parent: EINVAL.
- /* Detach + reattach */
- attach_recursive_mnt(old.mnt, path, NULL, false).

REQ-9: Per-type-change (do_change_type):
- /* Propagation-only on existing mount */
- For each mount in subtree (per-MS_REC): change_mnt_propagation(m, type).
- types: MS_SHARED ⟹ MNT_SHARED, MS_PRIVATE ⟹ 0, MS_SLAVE ⟹ MNT_SLAVE, MS_UNBINDABLE ⟹ MNT_UNBINDABLE.

REQ-10: Per-mount-attribute extraction (legacy MS_* → MNT_*):
- MS_NOSUID ⟹ MNT_NOSUID.
- MS_NODEV ⟹ MNT_NODEV.
- MS_NOEXEC ⟹ MNT_NOEXEC.
- MS_NOATIME ⟹ MNT_NOATIME.
- MS_NODIRATIME ⟹ MNT_NODIRATIME.
- MS_RELATIME ⟹ MNT_RELATIME.
- MS_STRICTATIME ⟹ clear MNT_RELATIME|MNT_NOATIME.
- MS_NOSYMFOLLOW ⟹ MNT_NOSYMFOLLOW.

REQ-11: Per-sb_flags extraction (legacy MS_* → SB_*):
- MS_RDONLY ⟹ SB_RDONLY.
- MS_SYNCHRONOUS ⟹ SB_SYNCHRONOUS.
- MS_MANDLOCK ⟹ SB_MANDLOCK.
- MS_DIRSYNC ⟹ SB_DIRSYNC.
- MS_SILENT ⟹ SB_SILENT.
- MS_POSIXACL ⟹ SB_POSIXACL.
- MS_KERNMOUNT ⟹ REJECT (internal).
- MS_I_VERSION ⟹ SB_I_VERSION.
- MS_LAZYTIME ⟹ SB_LAZYTIME.

REQ-12: Per-mount namespace bookkeeping:
- Mount added to current->nsproxy->mnt_ns->list.
- mnt_ns->mounts++, mnt_ns->event++.
- Reject if mnt_ns->mounts >= sysctl_mount_max (EMFILE).

REQ-13: Per-data argument bound:
- copy_mount_options(data) — at most PAGE_SIZE-1; NUL-padded.
- May be NULL.

REQ-14: Per-string copy:
- source, target, fstype, data each via strndup_user / copy_mount_string (PATH_MAX / PAGE_SIZE).
- EFAULT on bad pointer; ENAMETOOLONG on overlong path.

REQ-15: Per-MS_NOUSER: rejected from userspace (EINVAL); internal-only.

## Acceptance Criteria

- [ ] AC-1: mount("/dev/sda1", "/mnt", "ext4", 0, "") succeeds; /proc/self/mountinfo shows it.
- [ ] AC-2: mount(NULL, "/mnt", NULL, MS_REMOUNT|MS_RDONLY, NULL) flips RO; EBUSY if rw-files open.
- [ ] AC-3: mount("/a", "/b", NULL, MS_BIND, NULL) creates bind clone; same sb_dev.
- [ ] AC-4: mount("/a", "/b", NULL, MS_BIND|MS_REC, NULL) clones subtree.
- [ ] AC-5: mount(NULL, "/mnt", NULL, MS_PRIVATE, NULL) sets MNT_NO propagation; readable via /proc/self/mountinfo "shared:" absence.
- [ ] AC-6: Non-CAP_SYS_ADMIN caller: EPERM.
- [ ] AC-7: Unknown fstype "no-such-fs": ENODEV.
- [ ] AC-8: data > PAGE_SIZE: EINVAL.
- [ ] AC-9: MS_REMOUNT|MS_BIND alone with new fstype: EINVAL (cannot replace).
- [ ] AC-10: mnt_ns->mounts == fs.mount-max: EMFILE.
- [ ] AC-11: MS_KERNMOUNT from user: EINVAL.
- [ ] AC-12: Per-MS_NOSYMFOLLOW respected: open(O_NOFOLLOW alike) on subsequent path walks.

## Architecture

```
Mount::sys_mount(source: UserPtr<u8>, target: UserPtr<u8>,
                 fstype: UserPtr<u8>, flags: u64, data: UserPtr<u8>) -> Result<i32, Errno>
```

1. /* Permissions */
2. cred = current().cred.
3. mnt_ns = current().nsproxy.mnt_ns.
4. if !ns_capable(mnt_ns.user_ns, CAP_SYS_ADMIN): return Err(EPERM).
5. /* Copy strings */
6. let kernel_source = copy_mount_string(source)?;        // PATH_MAX
7. let kernel_target = copy_mount_string(target)?;
8. let kernel_fstype = copy_mount_string(fstype)?;
9. let kernel_data   = copy_mount_options(data)?;          // PAGE_SIZE - 1
10. /* Reject internal flags */
11. if flags & (MS_KERNMOUNT | MS_NOUSER | MS_BORN | MS_ACTIVE) != 0: return Err(EINVAL).
12. /* Lookup target */
13. let path = user_path_at(AT_FDCWD, &kernel_target, LOOKUP_FOLLOW)?;
14. /* Dispatch */
15. let result = match select_mode(flags) {
      Mode::Remount        => path_mount_remount(&path, flags, &kernel_data),
      Mode::Bind { rec }   => path_mount_bind(&kernel_source, &path, rec, flags),
      Mode::Move           => path_mount_move(&kernel_source, &path),
      Mode::ChangeType(t)  => path_mount_change_type(&path, t, flags & MS_REC != 0),
      Mode::New            => path_mount_new(&kernel_fstype, &kernel_source, &path, flags, &kernel_data),
    };
16. path_put(&path).
17. result.
```

`Mount::select_mode(flags) -> Mode`:
- if flags & MS_REMOUNT: Mode::Remount.
- elif flags & MS_BIND: Mode::Bind { rec: flags & MS_REC != 0 }.
- elif flags & MS_MOVE: Mode::Move.
- elif flags & (MS_SHARED|MS_PRIVATE|MS_SLAVE|MS_UNBINDABLE):
  - Mode::ChangeType(prop_type(flags)).
- else: Mode::New.

`Mount::path_mount_new`:
1. Reject conflicting flags.
2. type = FsRegistry::get_fs_type(fstype) or return Err(ENODEV).
3. if type.init_fs_context.is_some():
   - fc = FsContext::for_mount(type, sb_flags_from_ms(flags))?;
   - fc.parse_string("source", source)?;
   - for (k, v) in parse_kv(data): fc.parse_string(k, v)?;
   - fc.get_tree()?;
   - do_new_mount_fc(fc, path, mnt_flags_from_ms(flags))?;
4. else:
   - mnt = mount_fs(type, sb_flags, source, data)?;
   - do_new_mount(type, mnt, path, mnt_flags_from_ms(flags))?;
5. FsRegistry::put_fs_type(type).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mount_caps_checked` | INVARIANT | per-sys_mount: CAP_SYS_ADMIN check precedes any mutation. |
| `internal_flags_rejected` | INVARIANT | per-sys_mount: MS_KERNMOUNT|MS_BORN|MS_ACTIVE|MS_NOUSER ⟹ EINVAL. |
| `data_bound_by_page` | INVARIANT | per-sys_mount: copy_mount_options ≤ PAGE_SIZE-1. |
| `mode_dispatch_disjoint` | INVARIANT | per-select_mode: at most one branch taken; remount/bind/move/change/new mutually-exclusive. |
| `mount_max_enforced` | INVARIANT | per-do_new_mount: mnt_ns.mounts < sysctl_mount_max post-insert. |
| `fc_put_on_error` | INVARIANT | per-fs_context error paths: fc dropped, no leak. |

### Layer 2: TLA+

`uapi/mount.tla`:
- Per-mount-namespace state machine: mount/unmount/propagate.
- Properties:
  - `safety_caps_required` — per-mount: EPERM iff ¬CAP_SYS_ADMIN.
  - `safety_mode_disjoint` — per-flags: exactly one of {Remount,Bind,Move,ChangeType,New}.
  - `safety_mount_max` — per-namespace: mounts ≤ sysctl_mount_max.
  - `safety_remount_rdonly_blocked_when_writers` — per-MS_REMOUNT|MS_RDONLY: EBUSY when ∃ rw-fd.
  - `liveness_mount_terminates` — per-call: returns success or definite errno.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mount::sys_mount` post: success ⟹ mnt_ns.mounts += 1 (new mode) | `Mount::sys_mount` |
| `Mount::path_mount_new` post: new mount inserted into mnt_ns list | `Mount::path_mount_new` |
| `Mount::path_mount_bind` post: cloned mount shares super_block | `Mount::path_mount_bind` |
| `Mount::path_mount_remount` post: super_block fstype unchanged | `Mount::path_mount_remount` |
| `Mount::path_mount_change_type` post: only mnt_flags propagation bits change | `Mount::path_mount_change_type` |

### Layer 4: Verus/Creusot functional

Per-mount syscall semantic equivalence to upstream `do_mount` / `path_mount` for each Mode: per-Documentation/filesystems/mount_api.rst.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

mount-syscall reinforcement:

- **Per-CAP_SYS_ADMIN in mnt_ns user_ns** — defense against per-unprivileged-mount escalation.
- **Per-MS_KERNMOUNT|MS_BORN|MS_ACTIVE rejected from userspace** — defense against per-internal-flag injection.
- **Per-data bounded to PAGE_SIZE-1** — defense against per-options DoS.
- **Per-strndup_user PATH_MAX bound on source/target/fstype** — defense against per-overlong-path OOM.
- **Per-mode-dispatch disjoint** — defense against per-flag-combination logic flaw.
- **Per-sysctl fs.mount-max enforced** — defense against per-namespace mount flooding.
- **Per-MS_NOSYMFOLLOW propagated to MNT_NOSYMFOLLOW** — defense against per-symlink-mountpoint escape.
- **Per-MS_NOSUID|MS_NODEV|MS_NOEXEC carried to MNT_*** — defense against per-suid-binary on untrusted FS.
- **Per-MS_REMOUNT cannot change fstype** — defense against per-superblock-type-confusion.
- **Per-mount-propagation type change without MS_REC bounded** — defense against per-runaway-propagation.
- **Per-PAX UDEREF on copy_from_user of source/target/fstype/data** — defense against per-kernel-pointer-fixup races.
- **Per-GRKERNSEC_CHROOT_MOUNT: chroot'd task forbidden from mount** — defense against per-chroot-escape via bind-mount.
- **Per-GRKERNSEC_NOMOUNT for non-root with sysctl gate** — defense against per-policy-bypass.
- **Per-PAX_RANDKSTACK on syscall entry** — defense against per-ROP-via-mount-arg.
- **Per-idmap immutability after mount established** — defense against per-uid-map-races (see mount_setattr).

## Grsecurity/PaX-style Reinforcement

Baseline kernel-wide mitigations relied upon by the legacy `mount(2)` syscall:

- **PAX_USERCOPY** — bounds-checks every `strndup_user` of source/target/fstype/data; aborts on slab-boundary crossings before any vfs_parse_monolithic step.
- **PAX_KERNEXEC** — `file_system_type` and `super_operations` vtables are RX/RO; rules out post-init rewrites of method tables during a remount race.
- **PAX_RANDKSTACK** — per-syscall kernel-stack offset randomization on `__do_sys_mount` defeats stack-pivot exploits routed through long `data=` strings.
- **PAX_REFCOUNT** — saturating refcount on `mnt->mnt_count`, `sb->s_active`, and `file_system_type->fs_supers` references; mount-vs-umount UAF races trap rather than underflow.
- **PAX_MEMORY_SANITIZE** — auto-zeroes `data` page and `kfree(source/target/fstype)` to remove tenant-supplied options from reused slab.
- **PAX_UDEREF** — strict user/kernel split on the five user pointers; a hostile process cannot smuggle kernel addresses through `data`.
- **PAX_RAP / kCFI** — forward-edge CFI on `file_system_type->mount` and `super_operations->remount_fs`, blocking JOP via tampered vtables.
- **GRKERNSEC_HIDESYM** — `do_mount`, `path_mount`, and per-fs `get_tree_*` symbols hidden from unprivileged kallsyms.
- **GRKERNSEC_DMESG** — restricts dmesg so per-fs mount failure traces (which often leak slab/module layout) are root-only.

Mount-family-specific reinforcement:

- **CAP_SYS_ADMIN-in-mnt_ns->user_ns strict** — every mode (new, bind, remount, propagation, move) re-checks owning-userns capability rather than init_user_ns alone.
- **GRKERNSEC_CHROOT_MOUNT** — chrooted task is unconditionally denied `mount(2)`; closes the classic chroot-escape-via-bind-mount path.
- **GRKERNSEC_CHROOT_PIVOT / CHROOT_DOUBLE** — blocks bind-mount-then-pivot sequences from inside a chroot even with CAP_SYS_ADMIN spoofed via userns.
- **GRKERNSEC_NOMOUNT** — sysctl gate forbidding any non-root mount, even with userns capabilities, for hard multi-tenant policy.
- **GRKERNSEC_SYSFS_RESTRICT** — limits leakage of `/sys/fs/<type>` state observed via mount errors during fuzzing.

Rationale: legacy `mount(2)` is the highest-risk capable syscall in the VFS surface; combining USERCOPY/UDEREF on the five user pointers with the grsec chroot/nomount gates collapses an entire historical class of container-escape and DoS bugs without disturbing the syscall ABI.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `fsopen`/`fsconfig`/`fsmount` new-API path (covered in their own Tier-5 docs).
- `mount_setattr` per-mount attribute change (covered in `mount_setattr.md`).
- `pivot_root` namespace re-rooting (covered in `pivot_root.md`).
- `umount2` (covered in `umount2.md`).
- Per-fs `init_fs_context` implementations (covered in fs/* Tier-3).
- Per-fs option parsing (covered in fs/<name>/options.md).
- Implementation code.
