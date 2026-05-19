# Tier-5 syscall: ustat(2) — syscall 136 (deprecated)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/statfs.c (SYSCALL_DEFINE2(ustat))
  - include/uapi/linux/ustat.h (struct ustat)
  - arch/x86/entry/syscalls/syscall_64.tbl (136  common  ustat)
-->

## Summary

`ustat(2)` is a **deprecated** Unix V7-era syscall that returns extremely minimal filesystem statistics for a filesystem identified by a `dev_t` device number. It returns only two useful fields: the count of free blocks and the count of free inodes; it does NOT return total size, block size, or filesystem type. Superseded by `statfs(2)`/`fstatfs(2)` decades ago and retained only for ancient SysV ABI compatibility. Glibc's `ustat()` wrapper has been deprecated since glibc 2.28. Hardened kernels return `-ENOSYS`. Critical for: ABI stability with V7-era binaries (rare), audit-trail of deprecated-syscall use, hardened ENOSYS contract.

## Signature

```c
int ustat(dev_t dev, struct ustat *ubuf);
```

```c
struct ustat {
    __kernel_daddr_t  f_tfree;    /* free blocks (32-bit on most arches) */
    __kernel_ino_t    f_tinode;   /* free inodes */
    char              f_fname[6]; /* legacy: filesystem name (unused) */
    char              f_fpack[6]; /* legacy: filesystem pack name (unused) */
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `dev` | `dev_t` | in | Device-number identifier of the filesystem (as returned by `stat(2).st_dev`). |
| `ubuf` | `struct ustat *` | out | Kernel fills `f_tfree` and `f_tinode`; the name fields are zeroed. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; `ubuf` filled. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `dev` does not correspond to a currently mounted filesystem; or FS does not implement `statfs`. |
| `EFAULT` | `ubuf` userptr fault. |
| `ENOSYS` | Hardened build / grsec returns ENOSYS unconditionally. |

## ABI surface

```text
__NR_ustat (x86_64) = 136
__NR_ustat (arm64)  = N/A (not implemented; ENOSYS)
__NR_ustat (riscv)  = N/A (not implemented; ENOSYS)
__NR_ustat (i386)   = 62

/* Deprecated: prefer statfs(2)/fstatfs(2). */
/* f_tfree and f_tinode are __kernel_daddr_t (32-bit signed) — silently truncates for >2^31 blocks. */
/* f_fname and f_fpack are LEGACY zero-filled; never carried real data in Linux. */
```

## Compatibility contract

REQ-1: Syscall number is **136** on x86_64. ABI-stable (but body may return ENOSYS in hardened builds).

REQ-2: `dev` MUST identify a currently mounted filesystem; otherwise `-EINVAL`.

REQ-3: Per-`CONFIG_USTAT_SYSCALL` (or equivalent hardened toggle): when disabled, returns `-ENOSYS`.

REQ-4: Implementation walks the super-block list (`user_get_super(dev, false)`):
- Iterates `super_blocks` under `sb_lock`.
- Matches `sb->s_dev == dev`.

REQ-5: Once `sb` found, calls `sb->s_op->statfs(sb->s_root, &kstatfs)`; copies `kstatfs.f_bfree` to `ubuf.f_tfree` and `kstatfs.f_ffree` to `ubuf.f_tinode`.

REQ-6: Per-32-bit-truncation: `f_tfree` is `__kernel_daddr_t` (32-bit signed); kernel silently truncates `kstatfs.f_bfree` to `INT32_MAX` if it overflows. Documented quirk (do NOT EOVERFLOW; ABI requires truncate).

REQ-7: `f_fname` and `f_fpack` MUST be zeroed (never carried real data in Linux; preserved for struct layout).

REQ-8: Permission: no permission check; any user can call. (Like sysfs(2), this is part of why hardened builds disable it.)

REQ-9: Per-namespace: no mount-namespace filtering — searches GLOBAL `super_blocks` list. Defense via hardened toggle.

REQ-10: Per-audit: every call audit-logged with dev, caller real-uid.

REQ-11: Locking: `sb_lock` for super_blocks walk; `super->s_umount` read-lock during `s_op->statfs` call.

REQ-12: Userspace SHOULD use `statfs(2)`/`fstatfs(2)`. ustat(2) is preserved only for ABI completeness.

## Acceptance Criteria

- [ ] AC-1: (Legacy build) `ustat(root_dev, &u)`: returns 0; `u.f_tfree > 0`.
- [ ] AC-2: (Legacy build) `ustat(invalid_dev, &u)`: `-EINVAL`.
- [ ] AC-3: NULL ubuf: `-EFAULT`.
- [ ] AC-4: `u.f_fname[0..6]` all zero; `u.f_fpack[0..6]` all zero.
- [ ] AC-5: (Legacy build) FS with > 2^31 free blocks: `u.f_tfree == INT32_MAX` (silent truncate, ABI).
- [ ] AC-6: (Hardened build) any `ustat(...)`: `-ENOSYS`.
- [ ] AC-7: Audit record per call regardless of build mode.

## Architecture

```rust
#[syscall(nr = 136, abi = "sysv")]
pub fn sys_ustat(dev: u32, ubuf: UserPtr<Ustat>) -> isize {
    if !cfg_enabled!(USTAT_SYSCALL) {
        AuditLog::record_deprecated_syscall("ustat");
        return -ENOSYS as isize;
    }
    Ustat::do_ustat(dev, ubuf)
}
```

`Ustat::do_ustat(dev, ubuf_ptr) -> isize`:
1. let sb = SuperBlocks::user_get_super(dev).ok_or(-EINVAL)?;
2. let mut kst = KernelStatfs::default();
3. if sb.s_op.statfs.is_none() {
4.   SuperBlocks::put(sb);
5.   return -EINVAL;
6. }
7. (sb.s_op.statfs)(&sb.s_root, &mut kst)?;
8. let mut u = Ustat::default();   /* zeroed including f_fname/f_fpack */
9. u.f_tfree = clamp_to_i32(kst.f_bfree);
10. u.f_tinode = clamp_to_i32(kst.f_ffree) as u32;
11. SuperBlocks::put(sb);
12. ubuf_ptr.write(&u)?;            // EFAULT
13. AuditLog::record_ustat(dev);
14. 0

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `hardened_returns_enosys` | INVARIANT | hardened build ⟹ ENOSYS always. |
| `f_fname_fpack_zero` | INVARIANT | output f_fname/f_fpack always zero. |
| `truncate_no_eoverflow` | INVARIANT | >INT32_MAX f_bfree truncates to INT32_MAX, never EOVERFLOW. |
| `dev_match_required` | INVARIANT | no matching sb ⟹ EINVAL. |
| `s_op_statfs_required` | INVARIANT | sb without statfs ⟹ EINVAL (not ENOSYS). |
| `sb_refcount_balanced` | INVARIANT | super_block ref get/put balanced across all paths. |

### Layer 2: TLA+

`fs/ustat.tla`:
- States: per-config-gate, per-sb-lookup, per-statfs-call, per-truncate, per-write-user.
- Properties:
  - `safety_hardened_enosys` — hardened build ⟹ ENOSYS unconditionally.
  - `safety_name_fields_zero` — output name fields always zero.
  - `safety_silent_truncate` — overflow truncates rather than errors (ABI quirk).
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_ustat` post: success ⟹ ubuf filled with truncated counters | `Ustat::do_ustat` |
| `user_get_super` post: returned sb has matching s_dev | `SuperBlocks::user_get_super` |

### Layer 4: Verus / Creusot functional

Per-`ustat(2)` legacy man-page semantic equivalence. Hardened build: ENOSYS contract.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`ustat(2)` reinforcement:

- **Per-hardened-build ENOSYS strict** — defense against per-deprecated-syscall reconnaissance.
- **Per-deprecated-syscall audit** — defense against per-legacy-vector use.
- **Per-name-field zero** — defense against per-info-disclosure via legacy struct fields.
- **Per-sb-refcount balanced** — defense against per-sb UAF.
- **Per-mount-ns-global walk** — flagged by audit; hardened builds disable.

## Grsecurity / PaX surface

- **GRKERNSEC_DEPRECATED_SYSCALL_ENOSYS_STRICT** — ustat(2) returns `-ENOSYS` unconditionally; the call path emits an audit record (ANOM_DEPRECATED_SYSCALL) with caller comm/uid. Defense against per-legacy-vector reconnaissance and per-cross-mount-ns superblock enumeration.
- **PaX UDEREF on `ubuf` copy_to_user** — defense against per-ubuf-pointer kernel deref (relevant if syscall is enabled in a legacy build); SMAP forced.
- **GRKERNSEC_HIDESYM on superblock dev_t enumeration** — ustat(2) walks the global super_blocks list across all mount namespaces; in hardened builds this enumeration is denied for non-root callers regardless of the ENOSYS toggle.
- **PAX_USERCOPY_HARDEN on Ustat struct copy** — bounded into whitelisted slab/stack; rejects oversized writes mapped to wrong slab.
- **GRKERNSEC_AUDIT_LEGACY_SYSCALL** — kernel.audit emits ANOM_LEGACY_USTAT record per call attempt (even when ENOSYS returned) with caller uid/comm/dev argument.
- **PAX_REFCOUNT on superblock refcount during walk** — defense against per-sb refcount overflow UAF.
- **GRKERNSEC_LOG_RATE_LIMIT** — defense against per-ENOSYS log flood by repeated legacy probing.
- **PaX KERNEXEC on `s_op->statfs` indirect call** — RAP-tagged; defense against per-ROP gadget; routes are disabled in hardened builds.
- **GRKERNSEC_MOUNT_NS_STRICT** — ustat(2) walks GLOBAL super_blocks list (no namespace filtering); hardened builds enforce ENOSYS partly to close this cross-namespace info leak.
- **GRKERNSEC_NEUTRALIZE_LEGACY_SYSCALLS family policy** — ustat(2) is co-neutralized with sysfs(2), uselib(2), getpmsg(2)/putpmsg(2), and unimplemented stubs; shared SIEM record-type for correlation.
- **PaX MEMORY_SANITIZE on local `Ustat` struct** — defense against per-stack-leak of kernel data into uninitialized fields; struct is zero-initialized before partial fill.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-`statfs(2)` / `fstatfs(2)` modern replacement (covered in their docs).
- Per-FS `s_op->statfs` implementations (covered in per-FS Tier-3 docs).
- Per-super_blocks list management (covered in Tier-3 `fs/super.md`).
- Implementation code.
