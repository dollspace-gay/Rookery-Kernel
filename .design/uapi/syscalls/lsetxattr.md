# Tier-5: syscall 189 — lsetxattr(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`189  common  lsetxattr  sys_lsetxattr`)
  - fs/xattr.c (`SYSCALL_DEFINE5(lsetxattr, ...)`, `path_setxattr`, `setxattr`)
  - fs/namei.c (`user_path_at` with lookup flags `0` — no `LOOKUP_FOLLOW`)
  - include/uapi/linux/xattr.h
  - security/security.c (`security_inode_setxattr`)
-->

## Summary

`lsetxattr(2)` is **x86_64 syscall 189**, the symlink-no-follow variant of `setxattr(2)`. It is identical to `setxattr(2)` except the terminal component of `path` is **not** followed if it is a symbolic link: the xattr is written to the symlink inode itself rather than to its target. Non-terminal components are still traversed normally (symlinks encountered before the last component are still followed).

Because xattrs on symlinks are filesystem-policy-restricted (most filesystems disallow `user.*` xattrs on symlinks, since the size limit interacts badly with the inline-symlink optimization), `lsetxattr` is primarily used for `trusted.*` and `security.*` namespaces on symlink inodes (e.g., SELinux labels on symlinks, IMA on symlinks). Internally it lowers to the same `path_setxattr` helper as `setxattr(2)`, differing only in the `LOOKUP_FOLLOW` bit passed to `user_path_at`.

Critical for: every libc `lsetxattr`, every SELinux label on symlinks, every backup/restore tool preserving xattrs on symlinks (`rsync -X`, `tar --xattrs`), every Rookery xattr-vfs symlink test.

## Signature

C (POSIX-ish / man-pages):

```c
int lsetxattr(const char *path, const char *name,
              const void *value, size_t size, int flags);
```

glibc wrapper: `__lsetxattr` → `INLINE_SYSCALL(lsetxattr, 5, path, name, value, size, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(lsetxattr,
                const char __user *, pathname,
                const char __user *, name,
                const void __user *, value,
                size_t,              size,
                int,                 flags);
```

Rookery dispatch:

```rust
pub fn sys_lsetxattr(
    path: UserPtr<u8>,
    name: UserPtr<u8>,
    value: UserPtr<u8>,
    size: usize,
    flags: i32,
) -> SyscallResult<i32>;
```

## Parameters

| name    | type                  | constraints                                                                                                | errno-on-bad           |
|---------|-----------------------|------------------------------------------------------------------------------------------------------------|------------------------|
| path    | `const char __user *` | NUL-terminated; length `< PATH_MAX`. **Terminal symlink is NOT followed.**                                  | `EFAULT` / `ENAMETOOLONG` / `ENOENT` |
| name    | `const char __user *` | NUL-terminated; `1 ≤ strlen(name) ≤ XATTR_NAME_MAX (255)`; must begin with a known namespace prefix.        | `EFAULT` / `ERANGE` / `ENOTSUP` |
| value   | `const void __user *` | Readable for `size` bytes; may be `NULL` only if `size == 0`.                                              | `EFAULT`               |
| size    | `size_t`              | `0 ≤ size ≤ XATTR_SIZE_MAX (65536)`; filesystem may impose smaller cap on symlink inodes.                  | `E2BIG`                |
| flags   | `int`                 | `0`, `XATTR_CREATE (0x1)`, or `XATTR_REPLACE (0x2)`; never both.                                            | `EINVAL`               |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.

## Errors

Same as `setxattr(2)` plus:

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EPERM`        | Filesystem disallows xattrs on the terminal symlink (most fs reject `user.*` on symlinks).      |
| `EOPNOTSUPP`   | Namespace not supported on symlink inodes for this fs.                                          |
| All of `setxattr(2)` | `EFAULT`, `ENAMETOOLONG`, `ENOENT`, `EACCES`, `ELOOP` (non-terminal), `ENOTDIR`, `EINVAL`, `EEXIST`, `ENODATA`, `ENOTSUP`, `E2BIG`, `EROFS`, `EDQUOT`, `ENOSPC`. |

Note: `ELOOP` applies only to non-terminal symlink chains (terminal symlink is the operand, so it does not contribute to the loop counter).

## ABI surface (constants + flags)

Identical to `setxattr(2)`: `XATTR_CREATE`, `XATTR_REPLACE`, `XATTR_NAME_MAX = 255`, `XATTR_SIZE_MAX = 65536`, the four namespace prefixes (`user.`, `trusted.`, `security.`, `system.`).

The only ABI difference is the lookup-flag passed internally: `lsetxattr` uses lookup flag `0` (no `LOOKUP_FOLLOW`), whereas `setxattr` uses `LOOKUP_FOLLOW`.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=path`, `%rsi=name`, `%rdx=value`, `%r10=size`, `%r8=flags`.
- REQ-2: `flags` validation identical to `setxattr(2)`: `flags & ~(XATTR_CREATE | XATTR_REPLACE) ⟹ -EINVAL`; mutually exclusive.
- REQ-3: `user_path_at(AT_FDCWD, path, 0, &p)` — lookup flag is `0`, so terminal symlink is **not** followed.
- REQ-4: Non-terminal symlinks in `path` are still followed (POSIX `lstat`-style semantics).
- REQ-5: After lookup, `path_setxattr` body is identical to `setxattr(2)`: `mnt_want_write`, name/value copy-in, LSM hook, `__vfs_setxattr_locked`, post-update.
- REQ-6: `XATTR_NAME_MAX` and `XATTR_SIZE_MAX` enforced identically.
- REQ-7: Filesystem may reject xattrs on symlinks at the handler level (`xattr_handler.set` returns `-EPERM` or `-EOPNOTSUPP`); kernel does not pre-empt this — the handler is authoritative.
- REQ-8: Capability gating per namespace identical to `setxattr(2)`: `trusted.* ⟹ CAP_SYS_ADMIN`, `security.capability ⟹ CAP_SETFCAP`, etc.
- REQ-9: LSM hook `security_inode_setxattr` runs against the symlink inode itself (not the target).
- REQ-10: On success: `fsnotify_xattr(dentry)` and `i_ctime` update apply to the symlink inode.
- REQ-11: Immutable / append-only symlink inodes reject the write with `-EPERM`.
- REQ-12: `delegated_inode` break-loop applies (rare on symlinks but supported).
- REQ-13: RO mount ⟹ `-EROFS`.
- REQ-14: Idmap is `mnt_idmap(p.mnt)`, identical to `setxattr(2)`.

## Acceptance Criteria

- [ ] AC-1: `lsetxattr("symlink", "trusted.x", "v", 1, 0)` writes xattr to symlink inode (verifiable via `lgetxattr`).
- [ ] AC-2: `setxattr("symlink", ...)` writes to the target — `lsetxattr` writes to the link — both verifiable independently.
- [ ] AC-3: `lsetxattr("symlink", "user.foo", ...) → -EPERM` on filesystems disallowing `user.*` on symlinks.
- [ ] AC-4: Non-terminal symlinks in `path` are followed normally.
- [ ] AC-5: `XATTR_CREATE` on existing symlink xattr → `-EEXIST`.
- [ ] AC-6: `XATTR_REPLACE` on missing symlink xattr → `-ENODATA`.
- [ ] AC-7: Unprivileged `trusted.*` write on symlink → `-EPERM`.
- [ ] AC-8: Name length > `XATTR_NAME_MAX` → `-ERANGE`.
- [ ] AC-9: Value size > `XATTR_SIZE_MAX` → `-E2BIG`.
- [ ] AC-10: RO mount → `-EROFS`.
- [ ] AC-11: Symlink with `S_IMMUTABLE` → `-EPERM`.

## Architecture

```
struct LsetxattrArgs {
    path: UserPtr<u8>,
    name: UserPtr<u8>,
    value: UserPtr<u8>,
    size: usize,
    flags: i32,
}
```

`Xattr::sys_lsetxattr(args) -> i32`:

1. `if args.flags & !(XATTR_CREATE | XATTR_REPLACE) != 0 { return -EINVAL; }`
2. `if args.flags == (XATTR_CREATE | XATTR_REPLACE) { return -EINVAL; }`
3. `let path = Path::user_path_at(AT_FDCWD, args.path, 0 /* no LOOKUP_FOLLOW */)?;`
4. `return Xattr::path_setxattr(&path, args.name, args.value, args.size, args.flags);`

(`Xattr::path_setxattr` is the shared helper from `setxattr.md`. The only behavioral difference between `setxattr` and `lsetxattr` is the lookup flag in step 3.)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `terminal_symlink_not_followed` | INVARIANT | `lsetxattr` ⟹ lookup flag `0`; target of terminal symlink is never dereferenced. |
| `nonterminal_symlinks_followed` | INVARIANT | Non-terminal components still walked through symlinks. |
| `flags_strict` | INVARIANT | Any bit outside `{XATTR_CREATE, XATTR_REPLACE}` ⟹ `-EINVAL`; mutually exclusive. |
| `name_length_bounded` | INVARIANT | `1 ≤ len(name) ≤ XATTR_NAME_MAX`. |
| `value_size_bounded` | INVARIANT | `0 ≤ size ≤ XATTR_SIZE_MAX`. |
| `lsm_hook_on_symlink_inode` | INVARIANT | `security_inode_setxattr` evaluated against symlink inode, not target. |

### Layer 2: TLA+

`uapi/syscalls/lsetxattr.tla`:
- Per-call → validate → lookup-no-follow → mnt_want_write → LSM → handler → fsnotify → mnt_drop_write → path_put.
- Properties:
  - `safety_no_follow_terminal`,
  - `safety_flags_validated`,
  - `safety_namespace_capability_gated`,
  - `liveness_lsetxattr_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ xattr installed on symlink inode (not target) | `Xattr::vfs_setxattr_locked` |
| Post: any error ⟹ symlink inode unchanged | `Xattr::vfs_setxattr_locked` |
| Post: success ⟹ symlink `i_ctime` advanced; target's `i_ctime` untouched | `Xattr::vfs_setxattr_locked` |

### Layer 4: Verus/Creusot functional

`lsetxattr(path, name, value, size, flags)` ≡ Linux lsetxattr(2) per `man 2 lsetxattr`: identical to `setxattr(2)` except terminal symlinks are not dereferenced.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening; inherits all `setxattr(2)` hardening.)

`lsetxattr(2)` reinforcement:

- **Per-no-follow-terminal** — defense against TOCTOU between caller's intended symlink target and a racing target-swap.
- **Per-symlink-namespace-policy** — filesystems disallow `user.*` on symlinks to prevent inline-symlink corruption.
- **Per-symlink immutable/append-only honored** — defense against attribute-laundering through symlinks.
- **Per-symlink LSM evaluation** — defense against labeling-bypass via symlink-as-proxy.
- **Per-`mnt_want_write` discipline** — identical to `setxattr(2)`.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `path`, `name`, `value` SMAP-guarded; copy bounded by validated sizes.
- **GRKERNSEC_LINK** — symlink-write protection: even if the caller owns the symlink, writes to xattrs of symlinks in world-writable / sticky directories are gated to match owner-or-CAP_FOWNER.
- **GRKERNSEC_SYMLINKOWN** — only the symlink's owner (or CAP_FOWNER) may modify its xattrs in tagged directories.
- **GRKERNSEC_FIFO** — FIFO + symlink combinations gated identically; xattrs cannot bypass FIFO write protection.
- **GRKERNSEC_TRUSTED** — `trusted.*` writes on symlinks restricted to root (or container-root with `CAP_SYS_ADMIN` over namespace).
- **GRKERNSEC_SECURITY_XATTR** — `security.*` on symlinks LSM-gated; `security.capability` on symlinks rejected outright (file capabilities are not inheritable through symlinks).
- **GRKERNSEC_SYSTEM_XATTR** — `system.*` on symlinks generally rejected by fs (no ACLs on symlinks); kernel honors the rejection.
- **GRKERNSEC_PROC** — xattr listings on symlinks restricted to owner/root.
- **GRKERNSEC_CHROOT_FCHDIR** — chroot'd `lsetxattr` cannot traverse pre-chroot paths.
- **GRKERNSEC_HIDESYM** — LSM denial printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — kvalue buffer zeroed on free.
- **PAX_REFCOUNT** — `struct path`/`struct dentry` refcounts saturating.
- **GRKERNSEC_AUDIT_XATTR** — every `trusted.*` / `security.*` symlink-write logged.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `setxattr(2)` — separate Tier-5 (`setxattr.md`).
- `fsetxattr(2)` — separate Tier-5 (`fsetxattr.md`).
- `setxattrat(2)` — separate Tier-5 (`setxattrat.md`).
- POSIX ACL semantics — covered in `fs/posix_acl.md` Tier-3 if expanded.
- LSM handler internals — covered in `security/00-overview.md`.
- Filesystem-specific symlink xattr policy (ext4/xfs/btrfs/tmpfs) — covered in per-fs Tier-3.
- Implementation code.
