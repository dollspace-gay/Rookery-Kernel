# Tier-5 syscall: quotactl(2) — syscall 179

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/quota/quota.c (SYSCALL_DEFINE4(quotactl), do_quotactl)
  - fs/quota/dquot.c (dquot_*)
  - include/uapi/linux/quota.h (Q_*, QFMT_*)
  - arch/x86/entry/syscalls/syscall_64.tbl (179  common  quotactl)
-->

## Summary

`quotactl(2)` manipulates disk quotas on a filesystem. It multiplexes ~20 commands selected by `cmd`, each operating on either a quota subsystem global state (turn on/off), a per-uid/gid/projectid quota record (get/set), or filesystem-quota statistics. Quotas constrain blocks-used and inodes-used per user/group/project, enforced at filesystem-write time.

Critical for: multi-tenant fileservers, build farms, container per-project quota enforcement, and storage administration.

## Signature

```c
int quotactl(int cmd, const char *special, int id, caddr_t addr);
```

```c
/* cmd = ((subcommand << SUBCMDSHIFT) | type) */
#define Q_QUOTAON     0x800002  /* enable quota on fs */
#define Q_QUOTAOFF    0x800003  /* disable */
#define Q_GETQUOTA    0x800007  /* read per-id quota */
#define Q_SETQUOTA    0x800008  /* set per-id quota */
#define Q_GETINFO     0x800005  /* read fs-wide quota info */
#define Q_SETINFO     0x800006  /* set fs-wide quota info */
#define Q_GETFMT      0x800004  /* return active format */
#define Q_SYNC        0x800001
#define Q_GETSTATS    0x800009  /* deprecated; use proc */
#define Q_XQUOTAON    0x584F4E01  /* XFS variants */
#define Q_XQUOTAOFF   0x584F4F46
#define Q_XGETQUOTA   0x584F4753
#define Q_XSETQLIM    0x584F5351
#define Q_XGETQSTAT   0x584F5354
/* quota types */
#define USRQUOTA  0
#define GRPQUOTA  1
#define PRJQUOTA  2
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `cmd` | `int` | in | Encoded subcommand + quota type. |
| `special` | `const char *` | in | Path of block device hosting the filesystem (e.g. `/dev/sda1`); `NULL` for cmds that do not need a fs (`Q_SYNC` with global scope). |
| `id` | `int` | in | uid/gid/projectid (for per-id cmds); ignored for fs-wide cmds. |
| `addr` | `caddr_t` | in/out | Pointer to a struct whose type depends on cmd (`struct dqblk`, `struct dqinfo`, `struct fs_disk_quota`, ...). |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success. |
| `>= 0` | For `Q_GETFMT`: the format code in `*addr`. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Caller lacks `CAP_SYS_ADMIN` for privileged subcmds. |
| `EACCES` | DAC denial on quota file. |
| `EFAULT` | `addr` or `special` faults. |
| `EINVAL` | Unknown cmd; bad type; bad id; quota format mismatch. |
| `ENOENT` | `special` does not name a block device with quota support. |
| `ENOTBLK` | `special` is not a block device. |
| `ENODEV` | Block device has no mounted fs. |
| `ENOSYS` | Kernel built without quota support. |
| `ENOTSUP` | Filesystem does not support requested quota type. |
| `ESRCH` | Quota disabled for the specified type. |
| `EIO` | Quota file IO error. |
| `EMFILE` | Quota subsystem internal limit hit. |
| `ERANGE` | Quota limit out of range. |

## ABI surface

```text
__NR_quotactl (x86_64) = 179
__NR_quotactl (arm64)  = 60
__NR_quotactl (riscv)  = 60
__NR_quotactl (i386)   = 131

/* struct if_dqblk for VFS quotas; struct fs_disk_quota for XFS variants. */
```

## Compatibility contract

REQ-1: Syscall number is **179** on x86_64. ABI-stable.

REQ-2: `cmd` is a packed (subcmd, type) pair; unknown subcmd or unknown type → `-EINVAL`.

REQ-3: Capability gate:
- `Q_QUOTAON`, `Q_QUOTAOFF`, `Q_SETQUOTA`, `Q_SETINFO`, `Q_XQUOTAON`, `Q_XQUOTAOFF`, `Q_XSETQLIM`: require `CAP_SYS_ADMIN` in the filesystem's user namespace.
- `Q_GETQUOTA`, `Q_XGETQUOTA`: caller may read **own** uid/gid quota without privilege; reading another id requires `CAP_SYS_ADMIN`.
- `Q_GETINFO`, `Q_GETFMT`, `Q_GETSTATS`, `Q_SYNC`: no special capability.

REQ-4: `special` resolution: looked up via `getname` + `lookup_bdev`. Must refer to a block device backing a mounted fs (else `-ENOTBLK` / `-ENODEV`).

REQ-5: Type validation: `type ∈ {USRQUOTA, GRPQUOTA, PRJQUOTA}`. Filesystem may support only a subset (e.g. ext4 supports all three with `quota=ext4`; some older fs only USR/GRP). Unsupported type → `-ENOTSUP`.

REQ-6: `Q_QUOTAON`: enables quotas using format specified in lower 8 bits of `id` (e.g. `QFMT_VFS_V0`, `QFMT_VFS_V1`) and quota-file path in `addr`. The format must already be loaded (modular `quota_v2`).

REQ-7: `Q_SETQUOTA` writes a `struct if_dqblk` containing block/inode hard+soft limits and current usage; the kernel rejects negative usage and limits exceeding `S64_MAX/2`.

REQ-8: `Q_GETQUOTA` returns `struct if_dqblk` with usage and limits. Per-user grace times included.

REQ-9: XFS variants (`Q_X*`) bypass the dquot subsystem and call into XFS's native quota; same capability rules.

REQ-10: `Q_SYNC` with non-NULL `special` syncs just that fs; with NULL syncs all quota-enabled fs.

REQ-11: `addr` may be NULL only for cmds that do not need it (`Q_QUOTAOFF`, `Q_SYNC`); else `-EFAULT`.

## Acceptance Criteria

- [ ] AC-1: `quotactl(Q_GETQUOTA, "/dev/sda1", uid, &dq)` for own uid returns 0 and `dq` populated.
- [ ] AC-2: `quotactl(Q_GETQUOTA, "/dev/sda1", other_uid, &dq)` without CAP_SYS_ADMIN returns `-EPERM`.
- [ ] AC-3: `quotactl(Q_QUOTAON, ...)` without CAP_SYS_ADMIN returns `-EPERM`.
- [ ] AC-4: Unknown subcmd returns `-EINVAL`.
- [ ] AC-5: Non-block-device `special` returns `-ENOTBLK`.
- [ ] AC-6: `Q_SETQUOTA` with usage > limit-allowed: returns `-ERANGE`.
- [ ] AC-7: XFS-mounted fs accepts `Q_XGETQUOTA`; ext4-mounted fs accepts `Q_GETQUOTA`.

## Architecture

```rust
#[syscall(nr = 179, abi = "sysv")]
pub fn sys_quotactl(cmd: i32, special: UserPtr<u8>, id: i32, addr: UserPtr<u8>) -> isize {
    Quotactl::do_quotactl(cmd, special, id, addr)
}
```

`Quotactl::do_quotactl(cmd, special_ptr, id, addr_ptr) -> isize`:
1. let (subcmd, qtype) = Quotactl::decode_cmd(cmd)?;             // EINVAL
2. /* Capability check (subcmd-specific) */
3. Quotactl::check_cap(subcmd, qtype, id)?;                       // EPERM
4. /* Resolve fs (most cmds need it) */
5. let sb = if Quotactl::needs_fs(subcmd) {
6.   let path = if special_ptr.is_null() { return -EFAULT; } else { Fs::lookup_bdev(special_ptr)? };
7.   Vfs::get_mounted_sb(&path).ok_or(ENODEV)?
8. } else { None };
9. /* Per-cmd dispatch */
10. match subcmd {
11.   Q_QUOTAON   => Quotactl::quota_on(sb, qtype, id, addr_ptr),
12.   Q_QUOTAOFF  => Quotactl::quota_off(sb, qtype),
13.   Q_GETQUOTA  => Quotactl::get_dquot(sb, qtype, id, addr_ptr),
14.   Q_SETQUOTA  => Quotactl::set_dquot(sb, qtype, id, addr_ptr),
15.   Q_GETINFO   => Quotactl::get_info(sb, qtype, addr_ptr),
16.   Q_SETINFO   => Quotactl::set_info(sb, qtype, addr_ptr),
17.   Q_GETFMT    => Quotactl::get_format(sb, qtype, addr_ptr),
18.   Q_SYNC      => Quotactl::sync(sb),
19.   Q_X*        => Quotactl::xfs_dispatch(sb, subcmd, qtype, id, addr_ptr),
20.   _           => Err(EINVAL),
21. }

`Quotactl::check_cap(subcmd, qtype, id) -> Result<()>`:
1. let priv_subcmds = [Q_QUOTAON, Q_QUOTAOFF, Q_SETQUOTA, Q_SETINFO,
                       Q_XQUOTAON, Q_XQUOTAOFF, Q_XSETQLIM];
2. if priv_subcmds.contains(&subcmd) {
3.   return Capability::check(CAP_SYS_ADMIN);
4. }
5. if subcmd == Q_GETQUOTA || subcmd == Q_XGETQUOTA {
6.   let own = match qtype { USRQUOTA => current_uid() == id, GRPQUOTA => in_group(id), _ => false };
7.   if !own { return Capability::check(CAP_SYS_ADMIN); }
8. }
9. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `subcmd_decoded` | INVARIANT | unknown subcmd or type ⟹ EINVAL. |
| `cap_for_set` | INVARIANT | Q_SET*/Q_QUOTAON without CAP_SYS_ADMIN ⟹ EPERM. |
| `getquota_self_or_cap` | INVARIANT | Q_GETQUOTA on other id requires CAP_SYS_ADMIN. |
| `fs_required_for_data_cmds` | INVARIANT | data-cmd without valid fs ⟹ ENODEV. |
| `range_validation` | INVARIANT | Q_SETQUOTA values clamped to S64_MAX/2. |

### Layer 2: TLA+

`fs/quotactl.tla`:
- States: decode-cmd, cap-check, resolve-fs, dispatch, copy-to/from-user.
- Properties:
  - `safety_cap_first` — capability checked before any user buffer read.
  - `safety_no_cross_uid_read` — non-priv caller cannot read other id.
  - `safety_set_atomic` — Q_SETQUOTA atomic against concurrent reads.
  - `liveness_returns` — every quotactl returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_quotactl` post: success ⟹ cap held for the subcmd | `Quotactl::do_quotactl` |
| `set_dquot` post: stored values ≤ S64_MAX/2 | `Quotactl::set_dquot` |
| `get_dquot` post: addr populated with current limits | `Quotactl::get_dquot` |

### Layer 4: Verus / Creusot functional

Per-`quotactl(2)` man-page semantic equivalence with `quota-tools` regression tests on ext4 and xfs.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`quotactl(2)` reinforcement:

- **Per-subcmd capability check** — defense against per-uid-quota-bypass.
- **Per-self-vs-other GETQUOTA gate** — defense against per-cross-uid reconnaissance.
- **Per-range clamp on SETQUOTA** — defense against per-overflow-wrap.
- **Per-bdev validation** — defense against per-arbitrary-path attempt.
- **Per-quota-file IO error EIO not crash** — defense against per-fs-quota-table corruption.

## Grsecurity / PaX surface

- **PaX UDEREF on `special` and `addr` user pointers** — defense against per-pointer kernel-deref bug; SMAP enforced.
- **CAP_SYS_ADMIN STRICT** — grsec enforces `CAP_SYS_ADMIN` in **init_user_ns** for all Q_*ON/OFF/SET* commands, not just owning user_ns. Closes container-escape via misconfigured user-namespaced quotas.
- **GRKERNSEC_CHROOT_QUOTA** — chrooted process is denied `quotactl` entirely (cannot manipulate quotas on host fs).
- **GRKERNSEC_PROC_RESTRICT** — quota-file paths and stats not visible to non-CAP_SYS_ADMIN callers via /proc.
- **PAX_USERCOPY_HARDEN on if_dqblk / fs_disk_quota copy_to/from_user** — bounded copy uses whitelisted slab.
- **GRKERNSEC_AUDIT_QUOTACTL** — every quotactl call audited with (cmd, special, id, uid, pid, exe).
- **PAX_REFCOUNT on sb / dquot refcount** — defense against per-quota UAF.
- **GRKERNSEC_HIDESYM** — Q_GETFMT format codes well-defined; no kernel pointers in output.
- **GRKERNSEC_DMESG-style rate limit on EPERM** — defense against per-cmd brute-force probe.
- **Per-init_user_ns gate on Q_QUOTAON** — non-init userns may never enable quotas on a host-namespace fs.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-fs dquot internals (Tier-3 `fs/quota/dquot.md`).
- XFS quota internals (Tier-3 XFS doc).
- `quotactl_fd(2)` (own Tier-5 doc).
- Implementation code.
