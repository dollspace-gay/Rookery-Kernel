# Tier-5 syscall: sysfs(2) — syscall 139 (deprecated)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/filesystems.c (SYSCALL_DEFINE3(sysfs), find_filesystem)
  - include/uapi/linux/unistd.h (__NR_sysfs)
  - arch/x86/entry/syscalls/syscall_64.tbl (139  common  sysfs)
-->

## Summary

`sysfs(2)` is a **deprecated** syscall that predates the `/sys` virtual filesystem (which confusingly bears the same name). It returns information about the kernel's table of registered filesystem types — convert a filesystem name to its index in the table, convert an index back to a name, or report the total number of registered filesystems. It was the pre-procfs way to enumerate filesystems and is preserved only for ABI stability. Modern code MUST use `/proc/filesystems` instead. Some hardened kernels (and grsecurity) wire this syscall to return `-ENOSYS` unconditionally. Critical for: ABI stability with pre-2.4 binaries, audit-trail of legacy-syscall usage, hardened-kernel ENOSYS contract.

## Signature

```c
int sysfs(int option, ... /* args depend on option */);

/* Per option: */
int sysfs(int option /* = 1 */, const char *fsname);          /* name -> index   */
int sysfs(int option /* = 2 */, unsigned int fs_index,
          char *buf);                                          /* index -> name   */
int sysfs(int option /* = 3 */);                              /* count           */
```

## Parameters

| option | Args | Description |
|---|---|---|
| `1` | `const char *fsname` | Look up filesystem by name; return its index in the registered table. |
| `2` | `unsigned int fs_index, char *buf` | Look up filesystem by index; copy its name (NUL-terminated) into `buf` (must be at least `PAGE_SIZE`). |
| `3` | (none) | Return the count of currently registered filesystems. |

## Return value

| Value | Meaning |
|---|---|
| `option=1` | `>= 0`: index of named FS in registered table. |
| `option=2` | `0`: success; `buf` filled with NUL-terminated name. |
| `option=3` | `>= 0`: count of registered filesystems. |
| `-1` + `errno` | Failure (any option). |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `option` not in {1,2,3}. |
| `ENOSYS` | Hardened build / grsec disables this syscall; returns `-ENOSYS`. |
| `ENOENT` | `option=1`: `fsname` not in registered table. |
| `EFAULT` | `option=1`: `fsname` userptr fault. `option=2`: `buf` userptr fault. |
| `EOVERFLOW` | `option=2`: `fs_index >= count`. |

## ABI surface

```text
__NR_sysfs (x86_64) = 139
__NR_sysfs (arm64)  = N/A (not implemented; ENOSYS)
__NR_sysfs (riscv)  = N/A (not implemented; ENOSYS)
__NR_sysfs (i386)   = 135

/* Deprecated: prefer reading /proc/filesystems instead. */
/* Hardened builds (CONFIG_SYSFS_SYSCALL=n by default since 4.5) return -ENOSYS. */
/* Glibc removed the libc wrapper; callers must invoke via syscall(2). */
```

## Compatibility contract

REQ-1: Syscall number is **139** on x86_64. ABI-stable (but body may return ENOSYS in hardened builds).

REQ-2: `option` MUST be `1`, `2`, or `3`; otherwise `-EINVAL`.

REQ-3: Per-`CONFIG_SYSFS_SYSCALL`: when disabled (default in mainline since 4.5), the entire syscall returns `-ENOSYS`. UAPI doc preserves the contract for ABI stability.

REQ-4: Per-`option=1`:
- `strndup_user(fsname, PAGE_SIZE)` — EFAULT on user-pointer fault, ENAMETOOLONG on > PAGE_SIZE.
- Walk `file_systems` linked list under `file_systems_lock`.
- Return 0-based index of matching `struct file_system_type`.
- Not found → `-ENOENT`.

REQ-5: Per-`option=2`:
- Walk `file_systems` list to `fs_index`-th entry.
- `copy_to_user(buf, fs->name, strlen(fs->name) + 1)`.
- `buf` MUST be at least 64 bytes (NAME_MAX-ish); kernel does NOT validate the size from caller — caller-buffer overrun is caller's responsibility (legacy ABI).
- Index out-of-range → `-EOVERFLOW`.

REQ-6: Per-`option=3`:
- Walk `file_systems` list counting entries; return count.

REQ-7: Per-namespace: no namespace filtering — returns the GLOBAL kernel filesystem-type registration table.

REQ-8: Per-permission: no permission check; any user can call. (This is part of why hardened builds disable it: it leaks the loaded-FS-module list.)

REQ-9: Locking: `read_lock(&file_systems_lock)` during list walk; concurrent `register_filesystem`/`unregister_filesystem` serialized.

REQ-10: Per-audit: every call audit-logged with option, caller real-uid.

REQ-11: ABI-deprecation policy: kernel returns ENOSYS in hardened builds; userspace MUST be prepared for ENOSYS and fall back to `/proc/filesystems`.

REQ-12: No new code SHOULD use this syscall; included in Tier-5 docs only for ABI completeness and to specify the strict ENOSYS contract.

## Acceptance Criteria

- [ ] AC-1: (Legacy build) `sysfs(1, "ext4")`: returns >= 0.
- [ ] AC-2: (Legacy build) `sysfs(2, 0, buf)`: returns 0; buf NUL-terminated.
- [ ] AC-3: (Legacy build) `sysfs(3)`: returns count > 0.
- [ ] AC-4: `sysfs(4)`: `-EINVAL`.
- [ ] AC-5: `sysfs(1, NULL)`: `-EFAULT`.
- [ ] AC-6: `sysfs(2, 999999, buf)`: `-EOVERFLOW`.
- [ ] AC-7: `sysfs(1, "nonexistent_fs")`: `-ENOENT`.
- [ ] AC-8: (Hardened build) any `sysfs(...)`: `-ENOSYS`.
- [ ] AC-9: Audit record per call regardless of build mode.

## Architecture

```rust
#[syscall(nr = 139, abi = "sysv")]
pub fn sys_sysfs(option: i32, arg1: u64, arg2: u64) -> isize {
    if !cfg_enabled!(SYSFS_SYSCALL) {
        AuditLog::record_deprecated_syscall("sysfs");
        return -ENOSYS as isize;
    }
    Sysfs::do_sysfs(option, arg1, arg2)
}
```

`Sysfs::do_sysfs(option, a1, a2) -> isize`:
1. match option {
2.   1 => Sysfs::name_to_index(UserPtr::from(a1)),
3.   2 => Sysfs::index_to_name(a1 as u32, UserPtr::from(a2)),
4.   3 => Sysfs::count(),
5.   _ => Err(-EINVAL),
6. }

`Sysfs::name_to_index(fsname_ptr) -> isize`:
1. let name = strndup_user(fsname_ptr, PAGE_SIZE)?;     // EFAULT/ENAMETOOLONG
2. read_lock(&file_systems_lock);
3. for (i, fs) in file_systems.iter().enumerate() {
4.   if fs.name == name { read_unlock; AuditLog::record_sysfs(1); return i as isize; }
5. }
6. read_unlock;
7. Err(-ENOENT)

`Sysfs::index_to_name(idx, buf_ptr) -> isize`:
1. read_lock(&file_systems_lock);
2. let fs = file_systems.iter().nth(idx as usize).ok_or_else(|| { read_unlock; -EOVERFLOW })?;
3. let n = fs.name.len() + 1;
4. buf_ptr.write_bytes(fs.name.as_bytes(), n)?;        // EFAULT
5. read_unlock;
6. AuditLog::record_sysfs(2);
7. 0

`Sysfs::count() -> isize`:
1. read_lock(&file_systems_lock);
2. let n = file_systems.iter().count();
3. read_unlock;
4. AuditLog::record_sysfs(3);
5. n as isize

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `option_in_range` | INVARIANT | option ∉ {1,2,3} ⟹ EINVAL. |
| `hardened_returns_enosys` | INVARIANT | CONFIG_SYSFS_SYSCALL=n ⟹ ENOSYS always. |
| `strict_no_buf_size_in_kernel` | INVARIANT | option=2: kernel does NOT receive caller buf-size; documented as ABI quirk. |
| `lock_held_during_walk` | INVARIANT | file_systems_lock read-held during enumeration. |
| `audit_per_call` | INVARIANT | per-call audit emitted. |

### Layer 2: TLA+

`fs/sysfs-syscall.tla`:
- States: per-config-gate, per-option-dispatch, per-list-walk, per-copy-out, per-return.
- Properties:
  - `safety_hardened_enosys` — CONFIG_SYSFS_SYSCALL=n ⟹ ENOSYS for all options.
  - `safety_invalid_option_einval` — option ∉ {1,2,3} ⟹ EINVAL.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_sysfs` post: option dispatch total | `Sysfs::do_sysfs` |
| `name_to_index` post: returns index of matching fs in walk order | `Sysfs::name_to_index` |
| `index_to_name` post: writes NUL-terminated name to user buf | `Sysfs::index_to_name` |
| `count` post: returns size of file_systems list | `Sysfs::count` |

### Layer 4: Verus / Creusot functional

Per-`sysfs(2)` man-page semantic equivalence (legacy contract). Hardened build: ENOSYS contract.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`sysfs(2)` reinforcement:

- **Per-`CONFIG_SYSFS_SYSCALL=n` default** — defense against per-FS-module-list leak.
- **Per-deprecated-syscall audit** — defense against per-legacy-vector reconnaissance.
- **Per-option strict {1,2,3}** — defense against per-table OOB dispatch.
- **Per-PAGE_SIZE strndup_user bound** — defense against per-user-pointer DoS.
- **Per-list-walk read-lock** — defense against per-concurrent-register UAF.

## Grsecurity / PaX surface

- **GRKERNSEC_DEPRECATED_SYSCALL_ENOSYS_STRICT** — sysfs(2) returns `-ENOSYS` unconditionally; the call path emits an audit record (ANOM_DEPRECATED_SYSCALL) recording caller comm/uid/syscall-nr. Defense against per-legacy-vector reconnaissance and FS-module-list disclosure.
- **PaX UDEREF on `fsname`/`buf` user-pointer copy** — defense against per-user-pointer kernel deref (relevant on builds that retain the legacy syscall behavior); SMAP forced.
- **GRKERNSEC_HIDESYM on file_systems list** — list of loaded FS modules is not exposed to non-root callers in any path (sysfs(2), /proc/filesystems, /proc/modules); defense against per-attack-surface fingerprinting.
- **PAX_USERCOPY_HARDEN on `buf` copy_to_user** — bounded into whitelisted slab/stack; legacy syscall does NOT receive caller-supplied buf-size, so a conservative 64-byte limit is enforced even if the syscall is enabled.
- **GRKERNSEC_AUDIT_LEGACY_SYSCALL** — kernel.audit emits ANOM_LEGACY_SYSFS record per call attempt (even when ENOSYS returned) with caller uid/comm/option.
- **PAX_REFCOUNT on file_systems list entry refcount during walk** — defense against per-module-unregister UAF during enumeration.
- **GRKERNSEC_LOG_RATE_LIMIT** — defense against per-ENOSYS log flood by repeated legacy probing.
- **PaX KERNEXEC on file_systems list walk** — defense against per-W^X violation in legacy code path; bypass via this syscall blocked.
- **GRKERNSEC_PROC_GETPID consistency** — caller identity recorded in audit even though the syscall doesn't otherwise expose pid context.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family policy** — sysfs(2) is one of a class (ustat, uselib, getpmsg, putpmsg, fdatasync-on-deprecated-fd) that is collectively neutralized; audit records use the same record-type so SIEM correlation works.
- **PaX MEMORY_SANITIZE on local `name` buffer** — defense against per-stack-leak of kernel `fs->name` string into uninitialized adjacent stack.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- The unrelated `/sys` virtual filesystem (covered in Tier-3 `fs/sysfs.md`).
- `/proc/filesystems` (covered in Tier-3 `fs/proc.md`).
- Filesystem registration mechanism (covered in Tier-3 `fs/filesystems.md`).
- Implementation code.
