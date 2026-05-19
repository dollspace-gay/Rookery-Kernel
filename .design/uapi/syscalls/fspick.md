# Tier-5 syscall: fspick(2) — syscall 433

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/fsopen.c (SYSCALL_DEFINE3(fspick), vfs_fspick)
  - fs/fs_context.c (fs_context_for_reconfigure)
  - include/uapi/linux/mount.h (FSPICK_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (433  common  fspick)
-->

## Summary

`fspick(2)` returns a fs-context fd referencing the superblock of an **existing** mount, so the caller can subsequently `fsconfig(2)` parameters on it and `fsconfig(FSCONFIG_CMD_RECONFIGURE)` to apply them — i.e. the new-API replacement for `mount(..., MS_REMOUNT, ...)`. It is the symmetric reconfigure counterpart to `fsopen(2)` (which builds a fresh superblock context) and pairs with `open_tree(2)` (which references the mount object, not the superblock).

Critical for: structured remount of established mounts, userspace tools that want typed param errors instead of a single `-EINVAL` from a `mount` call, and container runtimes performing live policy updates on mounted filesystems.

## Signature

```c
int fspick(int dirfd, const char *path, unsigned int flags);
```

```c
#define FSPICK_CLOEXEC          0x00000001
#define FSPICK_SYMLINK_NOFOLLOW 0x00000002
#define FSPICK_NO_AUTOMOUNT     0x00000004
#define FSPICK_EMPTY_PATH       0x00000008
#define FSPICK__MASK            0x0000000f
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dirfd` | `int` | in | Resolution base for `path`; `AT_FDCWD`, dirfd, or with `FSPICK_EMPTY_PATH` any fd. |
| `path` | `const char *` | in | Path identifying any object on the target mount; `""` permitted with `_EMPTY_PATH`. |
| `flags` | `unsigned int` | in | Bitmask of `FSPICK_*`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Success: new fs-context fd usable with `fsconfig(2)`. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Caller lacks `CAP_SYS_ADMIN` in the mount's user namespace. |
| `EINVAL` | Unknown flag bits, or filesystem does not support reconfigure. |
| `EBADF` | `dirfd` invalid. |
| `ENOENT` | Path missing. |
| `ENOTDIR` | Non-final path component not a directory. |
| `EFAULT` | `path` pointer faulted. |
| `EACCES` | DAC / MAC denial on path traversal. |
| `EBUSY` | Filesystem in transient state (e.g. another reconfigure in progress). |
| `ENOMEM` | Allocator failed for fs_context. |
| `EOPNOTSUPP` | Filesystem type lacks `reconfigure` operation. |

## ABI surface

```text
__NR_fspick (x86_64) = 433
__NR_fspick (arm64)  = 433
__NR_fspick (riscv)  = 433
__NR_fspick (i386)   = 433

/* The returned fd is an anon-inode of type "fscontext". */
```

## Compatibility contract

REQ-1: Syscall number is **433** on x86_64. ABI-stable since Linux 5.2.

REQ-2: `flags` validation: any bit outside `FSPICK__MASK` returns `-EINVAL`.

REQ-3: `FSPICK_CLOEXEC` marks the returned fd close-on-exec. Strongly recommended in hardened mode.

REQ-4: `FSPICK_SYMLINK_NOFOLLOW`: identical to `AT_SYMLINK_NOFOLLOW`. Final component of `path` must not be followed if a symlink.

REQ-5: `FSPICK_NO_AUTOMOUNT`: identical to `AT_NO_AUTOMOUNT`. Resolver does not trigger automounts.

REQ-6: `FSPICK_EMPTY_PATH`: identical to `AT_EMPTY_PATH`. With `path == ""`, `dirfd` itself names the target.

REQ-7: Capability gate: requires `CAP_SYS_ADMIN` in the mount's owning user namespace. Reconfiguring is privileged because it can change mount-flags and per-fs security policy (e.g. `nodev`/`noexec`).

REQ-8: The returned fs-context starts in `purpose = FS_CONTEXT_FOR_RECONFIGURE`. Subsequent `fsconfig(FSCONFIG_SET_STRING/FLAG/...)` accumulate parameter changes; `fsconfig(FSCONFIG_CMD_RECONFIGURE)` commits.

REQ-9: Filesystem must implement `.reconfigure` in its `super_operations` or `fs_context_operations`; otherwise `-EOPNOTSUPP`.

REQ-10: A successful `fspick` does not yet reconfigure — it merely binds the context to the existing superblock. Closing the fd without commit is a no-op (no rollback needed).

REQ-11: Locking: per-superblock `s_umount` is taken in write mode at commit (during fsconfig RECONFIGURE), not at pick. Concurrent picks against the same sb are permitted; only one may commit at a time.

## Acceptance Criteria

- [ ] AC-1: `fspick(AT_FDCWD, "/", FSPICK_CLOEXEC)` returns positive fd on a reconfigurable rootfs.
- [ ] AC-2: Unknown bit `0x80000000` returns `-EINVAL`.
- [ ] AC-3: Without `CAP_SYS_ADMIN`, returns `-EPERM`.
- [ ] AC-4: Target fs lacking `.reconfigure` returns `-EOPNOTSUPP`.
- [ ] AC-5: Returned fd accepts `fsconfig(FSCONFIG_SET_STRING)` and `fsconfig(FSCONFIG_CMD_RECONFIGURE)`.
- [ ] AC-6: Two concurrent commits on the same sb: one succeeds, one returns `-EBUSY`.
- [ ] AC-7: `FSPICK_EMPTY_PATH` with `path = ""` and `dirfd` of an open file resolves to that file's mount.

## Architecture

```rust
#[syscall(nr = 433, abi = "sysv")]
pub fn sys_fspick(dirfd: i32, path: UserPtr<u8>, flags: u32) -> isize {
    Fspick::do_fspick(dirfd, path, flags)
}
```

`Fspick::do_fspick(dirfd, path, flags) -> isize`:
1. /* Flag validation */
2. if flags & !FSPICK__MASK != 0 { return -EINVAL; }
3. let lookup = LookupFlags::from_fspick(flags);
4. let target = Fs::resolve_at(dirfd, path, lookup)?;
5. /* Capability */
6. Capability::check_ns(CAP_SYS_ADMIN, target.mnt.user_ns)?;       // EPERM
7. /* Reconfigure context */
8. if target.mnt.sb.fs_type.reconfigure.is_none() ∧ target.mnt.sb.s_op.reconfigure.is_none() {
9.   return -EOPNOTSUPP;
10. }
11. let ctx = FsContext::for_reconfigure(target.mnt.sb)?;           // ENOMEM
12. /* Install */
13. let oflags = if flags & FSPICK_CLOEXEC != 0 { O_CLOEXEC } else { 0 };
14. let fd = AnonInode::install("fscontext", &fscontext_fops, ctx, oflags)?;
15. fd as isize

`FsContext::for_reconfigure(sb) -> Result<FsContext>`:
1. let mut ctx = FsContext::new(sb.fs_type, FS_CONTEXT_FOR_RECONFIGURE);
2. ctx.root = sb.s_root.clone();
3. ctx.sb_flags_mask = sb.s_flags & RECONFIGURE_MASK;
4. Ok(ctx)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_mask_validated` | INVARIANT | unknown bit ⟹ EINVAL. |
| `cap_required` | INVARIANT | !CAP_SYS_ADMIN in target user_ns ⟹ EPERM. |
| `reconfigure_method_required` | INVARIANT | fs lacking .reconfigure ⟹ EOPNOTSUPP. |
| `pick_does_not_lock_sb` | INVARIANT | s_umount not held at fspick return. |

### Layer 2: TLA+

`fs/fspick.tla`:
- States: flag-validate, resolve, cap, reconfigure-support, ctx-alloc, fd-install.
- Properties:
  - `safety_cap_first` — capability checked before any allocation.
  - `safety_no_commit_at_pick` — pick alone does not change sb flags.
  - `liveness_returns` — every fspick returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_fspick` post: success ⟹ fd is an fs-context, purpose=RECONFIGURE | `Fspick::do_fspick` |
| `for_reconfigure` post: ctx.root = sb.s_root | `FsContext::for_reconfigure` |

### Layer 4: Verus / Creusot functional

Per-`fspick(2)` man-page semantic equivalence with util-linux remount tests (`mount -o remount,ro`).

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`fspick(2)` reinforcement:

- **Per-flag whitelist** — defense against per-extension-flag smuggling.
- **Per-CAP_SYS_ADMIN check** — defense against unprivileged remount attempts.
- **Per-EOPNOTSUPP for non-reconfigurable fs** — defense against silent no-op.
- **Per-CLOEXEC strongly recommended** — defense against per-fd-inheritance leak across exec.
- **Per-commit-time s_umount serialization** — defense against per-concurrent-remount race.

## Grsecurity / PaX surface

- **PaX UDEREF on `path` copy_from_user** — defense against per-pointer kernel-deref bug; SMAP enforced.
- **GRKERNSEC_CHROOT_NOMOUNT** — chrooted process cannot fspick (cannot get a reconfigure handle).
- **GRKERNSEC_LOCK_MOUNT** — locked mount tree: fspick returns -EPERM regardless of caps.
- **PAX_USERCOPY_HARDEN on path buffer** — bounded copy uses whitelisted slab.
- **Per-init_user_ns required for security-relevant flag changes** — non-init userns fspick cannot clear `nodev`/`noexec`/`nosuid` via subsequent fsconfig.
- **GRKERNSEC_AUDIT_MOUNT** — fspick logged with (path, uid, pid, exe).
- **PAX_REFCOUNT on fs_context refcount** — defense against per-refcount overflow UAF.
- **GRKERNSEC_PROC_RESTRICT** — fscontext fds not exposed via /proc/PID/fd to non-owner.
- **Per-fs reconfigure callback under PAX_USERCOPY_HARDEN** — defense against per-parameter kernel-buffer overflow.
- **GRKERNSEC_HARDEN_PTRACE** — fscontext fds not inheritable across ptrace to lower-priv tracer.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `fsconfig(2)` parameter setting (own Tier-5 doc).
- `fsmount(2)` superblock-to-mount conversion (own Tier-5 doc).
- Per-filesystem `.reconfigure` semantics (Tier-3 per-fs docs).
- Implementation code.
