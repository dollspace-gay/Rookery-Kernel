# Tier-5 syscall: getgroups(2) — syscall 115

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - kernel/groups.c (SYSCALL_DEFINE2(getgroups), groups_to_user, group_info)
  - include/linux/cred.h (struct cred::group_info)
  - include/linux/user_namespace.h (from_kgid_munged)
  - arch/x86/entry/syscalls/syscall_64.tbl (115 common getgroups)
-->

## Summary

`getgroups(2)` retrieves the supplementary group IDs (in addition to the primary group ID) of the calling process. With `size == 0`, it returns the count without writing the array, enabling a two-call probe pattern. With `size > 0`, the kernel copies up to `size` GIDs into the caller-provided buffer; if the kernel-side count exceeds `size`, the call returns `-EFAINVAL`.

GIDs are returned through `from_kgid_munged()`, which maps in-kernel `kgid_t` (always in `init_user_ns`) to the **caller's** user-namespace view; unmapped GIDs are reported as the overflow GID (`/proc/sys/kernel/overflowgid`, default 65534). Critical for: POSIX permission checks in userspace, `chown`-helper tools, container-ns GID mapping.

## Signature

```c
int getgroups(int size, gid_t list[]);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `size` | `int` | in | Array capacity in entries (`gid_t`). `0` ⟹ probe mode (return count, no write). |
| `list` | `gid_t []` | out | User-space buffer to receive supplementary GIDs. Untouched when `size == 0`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of supplementary group IDs (when `size == 0`, the true count; when `size > 0`, also the number written). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `size < 0`, OR `size > 0 && size < current.ngroups`. |
| `EFAULT` | `list` not writable for `size * sizeof(gid_t)` bytes. |

## ABI surface

```text
__NR_getgroups  (x86_64) = 115
__NR_getgroups  (arm64)  = 158
__NR_getgroups  (riscv)  = 158
__NR_getgroups  (i386)   = 80

/* gid_t is u32 on Linux (POSIX-compliant). */
/* NGROUPS_MAX = 65536 (kernel-side hard cap). */
```

## Compatibility contract

REQ-1: Syscall number is **115** on x86_64. ABI-stable.

REQ-2: `size < 0` ⟹ `-EINVAL`.

REQ-3: `size == 0`: return `current_cred()->group_info->ngroups`; do NOT touch `list` (callable with `list == NULL`).

REQ-4: `size > 0`:
- If `size < current.ngroups` ⟹ `-EINVAL`.
- Else copy `ngroups` GIDs via `groups_to_user(list, group_info)`:
  - For each kgid in `group_info->gid[i]`, translate via `from_kgid_munged(current_user_ns(), kgid)`.
  - Write to `list[i]`.
- Return `ngroups`.

REQ-5: Primary group (real/effective GID) is NOT included in `getgroups(2)`. Get via `getegid(2)`/`getgid(2)`.

REQ-6: `group_info` is reference-counted; held during the call via `get_group_info(current_cred()->group_info)` and `put_group_info` at exit.

REQ-7: When a kgid has no mapping in caller's user_ns, `from_kgid_munged` returns the overflow GID (`/proc/sys/kernel/overflowgid`); not an error.

REQ-8: Order is stable: `groups_sort()` ensures `group_info->gid[]` is sorted ascending; userspace can binary-search.

REQ-9: Hard kernel cap: `NGROUPS_MAX = 65536`. `current.ngroups` is therefore ≤ 65536. `getgroups` may need up to 256 KiB of user buffer.

REQ-10: 32-bit compat: glibc historically used `__kernel_old_gid_t` (16-bit) via `getgroups16(2)` (`__NR_getgroups16 = 80` on i386). Modern code uses the 32-bit form via `__NR_getgroups32` (= 205 on i386). Kernel implements both.

REQ-11: NPTL: glibc maintains a per-thread groups cache; setgroups invalidates; getgroups is generally a syscall (no caching, since other tasks can modify via setgroups on same cred via setuid_keep_caps etc.).

REQ-12: `current_cred()` is read under RCU; cred swap during execve does not cause torn reads (cred is immutable once published).

## Acceptance Criteria

- [ ] AC-1: `getgroups(0, NULL)` returns ngroups (≥ 0) and writes nothing.
- [ ] AC-2: `getgroups(64, buf)` with ngroups=3 returns 3, writes 3 GIDs to buf[0..3].
- [ ] AC-3: `getgroups(2, buf)` with ngroups=3 returns -EINVAL.
- [ ] AC-4: `getgroups(-1, buf)`: -EINVAL.
- [ ] AC-5: `getgroups(64, NULL)` when ngroups > 0: -EFAULT.
- [ ] AC-6: Inside userns with unmapped GIDs: returns overflow GID for unmapped entries.
- [ ] AC-7: List sorted ascending.
- [ ] AC-8: Primary GID (getegid) not present in returned list (unless explicitly set as supplementary).
- [ ] AC-9: Cred change via setresgid: subsequent getgroups reflects new cred.
- [ ] AC-10: `getgroups` count matches `setgroups` count just performed.

## Architecture

```rust
#[syscall(nr = 115, abi = "sysv")]
pub fn sys_getgroups(size: i32, list: UserPtr<u32>) -> isize {
    GetGroups::do_getgroups(size, list)
}
```

`GetGroups::do_getgroups(size, list) -> isize`:
1. if size < 0 { return -EINVAL; }
2. let cred = current_cred();
3. let gi = get_group_info(&cred.group_info);
4. let n = gi.ngroups;
5. if size == 0 { put_group_info(gi); return n as isize; }
6. if (size as usize) < n {
7.   put_group_info(gi);
8.   return -EINVAL;
9. }
10. /* Translate and copy. */
11. let mut kbuf = SmallVec::<[u32; 32]>::with_capacity(n);
12. for i in 0..n {
13.   let kgid = gi.gid[i];
14.   let gid_view = from_kgid_munged(current_user_ns(), kgid);
15.   kbuf.push(gid_view);
16. }
17. let r = list.copy_out_slice(&kbuf);
18. put_group_info(gi);
19. r.map(|_| n as isize).unwrap_or(-EFAULT)

`from_kgid_munged(ns, kgid) -> u32`:
1. match kgid_to_gid_in(ns, kgid) {
2.   Some(g) => g,
3.   None => OVERFLOWGID,
4. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `negative_size_einval` | INVARIANT | size < 0 ⟹ EINVAL. |
| `probe_no_write` | INVARIANT | size == 0 ⟹ list untouched. |
| `size_too_small_einval` | INVARIANT | 0 < size < ngroups ⟹ EINVAL. |
| `efault_on_bad_list` | INVARIANT | copy_to_user failure ⟹ EFAULT. |
| `group_info_refcount_balanced` | INVARIANT | get_group_info paired with put_group_info on every path. |
| `unmapped_overflowgid` | INVARIANT | kgid without mapping ⟹ overflow gid value. |

### Layer 2: TLA+

`kernel/getgroups.tla`:
- States: per-cred group_info, per-userns gid map, per-buffer write.
- Properties:
  - `safety_size_zero_probe` — probe returns count, no write.
  - `safety_size_too_small_einval` — short buffer fails before write.
  - `safety_userns_mapped_or_overflow` — every output GID either mapped or == overflow.
  - `liveness_returns` — bounded steps.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_getgroups` post: ret >= 0 ⟹ ret == cred.ngroups | `GetGroups::do_getgroups` |
| `do_getgroups` post: size == 0 ⟹ no put_user calls | `GetGroups::do_getgroups` |
| `do_getgroups` post: ret > 0 ⟹ list[0..n] reflects from_kgid_munged | `GetGroups::do_getgroups` |
| `from_kgid_munged` post: unmapped ⟹ OVERFLOWGID | `from_kgid_munged` |

### Layer 4: Verus/Creusot functional

Per-`getgroups(2)` man page + POSIX getgroups semantic equivalence. glibc `getgroups(3)` wrapper passes.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`getgroups(2)` reinforcement:

- **Per-size signed-zero / negative check** — defense against per-negative-overflow oracle.
- **Per-group_info refcount balanced** — defense against per-cred UAF during call.
- **Per-from_kgid_munged unmapped-overflow** — defense against per-userns-info-leak of host gids.
- **Per-copy_to_user via slice** — defense against per-pointer-arith bug in groups_to_user.
- **Per-ngroups ≤ NGROUPS_MAX** — defense against per-ngroups-overflow.
- **Per-cred snapshot under RCU** — defense against per-cred-swap torn read.

## Grsecurity / PaX surface

- **PaX UDEREF on copy_to_user** — defense against per-list kernel-deref bug; SMAP forced.
- **PAX_USERCOPY_HARDEN on group list copy** — bounded `ngroups * sizeof(u32)`; whitelisted slab.
- **GRKERNSEC_PROC_ADD** — when GRKERNSEC_PROC_USER hides group info in /proc/$pid/status, getgroups on another task via ptrace mirror is also restricted.
- **PAX_REFCOUNT on group_info refcount** — defense against per-refcount-overflow UAF.
- **GRKERNSEC_HIDESYM** — internal group_info pointer not exposed.
- **Per-userns strict unmapped→overflow** — defense against per-cross-userns gid information leak; even ptrace-allowed callers see overflow gid for unmapped entries.
- **Per-cred snapshot under RCU** — defense against per-cred-swap during getgroups (set*gid mid-call).
- **GRKERNSEC_AUDIT_GROUPS** — optional audit of every getgroups call by uid range; defense against group-enumeration recon.
- **Per-NGROUPS_MAX strict** — grsec lowers default cap to 65536 and rejects setgroups beyond it (mirror gate ensures getgroups never sees larger ngroups).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `setgroups(2)` writer (covered in `setgroups.md`).
- `getgid(2)`/`getegid(2)` primary GID (covered separately).
- User-namespace gid_map setup (covered in `userns.md`).
- Implementation code.
