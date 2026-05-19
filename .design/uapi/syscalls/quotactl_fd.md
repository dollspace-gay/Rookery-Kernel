# Tier-5 syscall: quotactl_fd(2) — syscall 443

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/quota/quota.c (SYSCALL_DEFINE4(quotactl_fd), kernel_quotactl)
  - include/uapi/linux/quota.h
  - arch/x86/entry/syscalls/syscall_64.tbl (443  common  quotactl_fd)
-->

## Summary

`quotactl_fd(2)` is the fd-based variant of `quotactl(2)`. Instead of identifying the target filesystem by a `/dev/sdaN` block-device path (`special`), it takes a regular file descriptor referring to any object on the target mount and uses that mount's superblock. This avoids needing access to `/dev` (useful in chroot, user-namespace, and rootless container environments) and removes the path-based race window in the legacy interface.

Critical for: container runtimes managing per-project quotas, rootless tools that lack `/dev/sda1` visibility, and quota libraries migrating to the fd API.

## Signature

```c
int quotactl_fd(unsigned int fd, unsigned int cmd, qid_t id, void *addr);
```

```c
typedef __kernel_uid32_t qid_t;
/* cmd encoding identical to quotactl(2); see Q_* constants. */
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `fd` | `unsigned int` | in | File descriptor for any object on the target filesystem (file, dir, mount fd, O_PATH fd). Kernel uses `fd->f_path.mnt->mnt_sb`. |
| `cmd` | `unsigned int` | in | Encoded subcmd + quota-type, same as `quotactl(2)`. |
| `id` | `qid_t` | in | uid/gid/projectid (for per-id cmds); ignored for fs-wide cmds. |
| `addr` | `void *` | in/out | Pointer to a struct whose type depends on cmd (`struct if_dqblk`, `struct fs_disk_quota`, ...). |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `>= 0` | For `Q_GETFMT`: format code. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Caller lacks `CAP_SYS_ADMIN` for privileged subcmds. |
| `EBADF` | `fd` invalid. |
| `EFAULT` | `addr` faults. |
| `EINVAL` | Unknown cmd; bad type; bad id; quota format mismatch. |
| `ENOSYS` | Kernel built without quota. |
| `ENOTSUP` | Filesystem does not support requested quota type. |
| `ESRCH` | Quota disabled for the type. |
| `EIO` | Quota file IO error. |
| `ERANGE` | Quota limit out of range. |

## ABI surface

```text
__NR_quotactl_fd (x86_64) = 443
__NR_quotactl_fd (arm64)  = 443
__NR_quotactl_fd (riscv)  = 443
__NR_quotactl_fd (i386)   = 443
```

## Compatibility contract

REQ-1: Syscall number is **443** on x86_64. ABI-stable since Linux 5.14.

REQ-2: Behaviour identical to `quotactl(2)` except for filesystem identification: superblock derived from `fd->f_path.mnt->mnt_sb` rather than looked up from a `/dev` path.

REQ-3: `fd` may be of any type — regular file, directory, symlink (`O_PATH`), block device, or mount fd from `open_tree(2)`. The only requirement is that it has an `f_path.mnt`.

REQ-4: Capability gates identical to `quotactl(2)` (REQ-3 in its doc). All privileged subcmds require `CAP_SYS_ADMIN`; `Q_GETQUOTA` allows self-id only without privilege.

REQ-5: Type validation, range clamping, and per-fs `.quota_*` op presence identical to `quotactl(2)`.

REQ-6: Race-free: no path lookup is performed at syscall time; the mount is held via the file's mount refcount, so the target cannot be unmounted underneath.

REQ-7: `addr` validation: the type depends on the cmd; size assumed by the kernel per-cmd. Buffer must reside in the caller's address space (PaX UDEREF enforced).

REQ-8: `id == -1` for fs-wide cmds (e.g. `Q_QUOTAOFF`, `Q_SYNC`) is allowed; otherwise the kernel validates `id ∈ [0, UID_MAX]`.

REQ-9: Mount-namespace scope: the fs targeted is whatever the caller's mount fd references; this can be a mount the caller does not normally have `/dev` visibility into, which is precisely the use case in containers.

REQ-10: `fd` is not consumed; caller retains ownership.

REQ-11: Block-device-fd compatibility: if `fd` refers directly to a block device, the kernel resolves the device's mounted superblock (if any) — making `quotactl_fd(open("/dev/sda1"), ...)` behave like the legacy `quotactl(..., "/dev/sda1", ...)` while still benefiting from race-free fd-based identification.

REQ-12: Mount-fd compatibility: an `open_tree(2)` mount fd is acceptable as `fd`; the kernel uses the mount's superblock directly without needing any in-tree file lookup.

## Acceptance Criteria

- [ ] AC-1: `quotactl_fd(fd_of_file_on_fs, Q_GETQUOTA, own_uid, &dq)` returns 0.
- [ ] AC-2: `quotactl_fd(fd, Q_GETQUOTA, other_uid, &dq)` without CAP_SYS_ADMIN returns `-EPERM`.
- [ ] AC-3: Invalid `fd` returns `-EBADF`.
- [ ] AC-4: Unknown cmd returns `-EINVAL`.
- [ ] AC-5: Filesystem without quota support returns `-ENOTSUP`.
- [ ] AC-6: Q_SETQUOTA with overflow values returns `-ERANGE`.
- [ ] AC-7: Q_QUOTAOFF without CAP_SYS_ADMIN returns `-EPERM`.

## Architecture

```rust
#[syscall(nr = 443, abi = "sysv")]
pub fn sys_quotactl_fd(fd: u32, cmd: u32, id: u32, addr: UserPtr<u8>) -> isize {
    QuotactlFd::do_quotactl_fd(fd, cmd, id, addr)
}
```

`QuotactlFd::do_quotactl_fd(fd, cmd, id, addr_ptr) -> isize`:
1. let (subcmd, qtype) = Quotactl::decode_cmd(cmd as i32)?;       // EINVAL
2. let file = FdTable::get(fd as i32).ok_or(EBADF)?;
3. let sb = file.f_path.mnt.mnt_sb;
4. /* Shared capability check */
5. Quotactl::check_cap(subcmd, qtype, id as i32)?;                 // EPERM
6. /* Per-cmd dispatch (shared with quotactl(2)) */
7. match subcmd {
8.   Q_QUOTAON   => Quotactl::quota_on(Some(sb), qtype, id as i32, addr_ptr),
9.   Q_QUOTAOFF  => Quotactl::quota_off(Some(sb), qtype),
10.  Q_GETQUOTA  => Quotactl::get_dquot(Some(sb), qtype, id as i32, addr_ptr),
11.  Q_SETQUOTA  => Quotactl::set_dquot(Some(sb), qtype, id as i32, addr_ptr),
12.  Q_GETINFO   => Quotactl::get_info(Some(sb), qtype, addr_ptr),
13.  Q_SETINFO   => Quotactl::set_info(Some(sb), qtype, addr_ptr),
14.  Q_GETFMT    => Quotactl::get_format(Some(sb), qtype, addr_ptr),
15.  Q_SYNC      => Quotactl::sync(Some(sb)),
16.  Q_X*        => Quotactl::xfs_dispatch(sb, subcmd, qtype, id as i32, addr_ptr),
17.  _           => Err(EINVAL),
18. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `subcmd_decoded` | INVARIANT | unknown subcmd ⟹ EINVAL. |
| `fd_validated` | INVARIANT | invalid fd ⟹ EBADF (no kernel-side fs deref). |
| `cap_for_set` | INVARIANT | privileged subcmd ∧ !CAP_SYS_ADMIN ⟹ EPERM. |
| `range_validation` | INVARIANT | Q_SETQUOTA values clamped to S64_MAX/2. |
| `no_path_lookup` | INVARIANT | sb derived from fd only; no /dev resolution. |

### Layer 2: TLA+

`fs/quotactl-fd.tla`:
- States: decode-cmd, fd-fetch, cap-check, dispatch, copy-to/from-user.
- Properties:
  - `safety_no_unmount_race` — fd's mount ref pins sb for the call.
  - `safety_cap_first` — capability checked before fs IO.
  - `liveness_returns` — every quotactl_fd returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_quotactl_fd` post: success ⟹ cap held for the subcmd | `QuotactlFd::do_quotactl_fd` |
| Identical post-conditions to `quotactl(2)` per cmd | `Quotactl::*` (shared) |

### Layer 4: Verus / Creusot functional

Per-`quotactl_fd(2)` man-page semantic equivalence: behaviour identical to `quotactl(2)` for the same (cmd, id, addr) tuple given an fd whose mount.sb equals the device's sb.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`quotactl_fd(2)` reinforcement:

- **Per-subcmd capability check** — same gates as `quotactl(2)`; defense against per-uid-quota-bypass.
- **Per-fd race-free target identification** — defense against per-path-race (`/dev/sda1` rename underneath).
- **Per-self-vs-other GETQUOTA gate** — defense against per-cross-uid reconnaissance.
- **Per-range clamp on SETQUOTA** — defense against per-overflow-wrap.
- **Per-mount-ref hold** — defense against per-unmount race.

## Grsecurity / PaX surface

- **PaX UDEREF on `addr` user pointer** — defense against per-pointer kernel-deref bug; SMAP enforced.
- **CAP_SYS_ADMIN STRICT** — grsec enforces `CAP_SYS_ADMIN` in **init_user_ns** for all Q_*ON/OFF/SET* commands. Container quota delegation must be explicit.
- **GRKERNSEC_CHROOT_QUOTA** — chrooted process denied quotactl_fd entirely (matches quotactl(2) policy; closes /dev-less escape vector).
- **GRKERNSEC_PROC_RESTRICT** — quota file paths and stats not visible to non-CAP_SYS_ADMIN callers via /proc.
- **PAX_USERCOPY_HARDEN on if_dqblk / fs_disk_quota copy_to/from_user** — bounded copy uses whitelisted slab.
- **GRKERNSEC_AUDIT_QUOTACTL** — every quotactl_fd call audited with (fd, cmd, id, uid, pid, exe).
- **PAX_REFCOUNT on sb / dquot / file refcount** — defense against per-quota UAF.
- **GRKERNSEC_HIDESYM** — Q_GETFMT format codes well-defined; no kernel pointers in output.
- **GRKERNSEC_DMESG-style rate limit on EPERM** — defense against per-cmd brute-force probe.
- **Per-init_user_ns gate on Q_QUOTAON** — non-init userns may never enable quotas on a host-namespace fs, even given an fd that mounts it.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-fs dquot internals (Tier-3 `fs/quota/dquot.md`).
- XFS quota internals (Tier-3 XFS doc).
- `quotactl(2)` legacy path-based variant (own Tier-5 doc).
- Implementation code.
