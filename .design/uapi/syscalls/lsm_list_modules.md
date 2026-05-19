# Tier-5 syscall: lsm_list_modules(2) — syscall 461

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - security/lsm_syscalls.c (SYSCALL_DEFINE3(lsm_list_modules))
  - security/security.c (lsm_active_count, lsm_id_iterate)
  - include/uapi/linux/lsm.h (LSM_ID_SELINUX, LSM_ID_APPARMOR, LSM_ID_SMACK, LSM_ID_LANDLOCK, LSM_ID_BPF, ...)
  - include/linux/security.h (security_list_modules)
  - arch/x86/entry/syscalls/syscall_64.tbl (461  common  lsm_list_modules)
-->

## Summary

`lsm_list_modules(2)` enumerates the LSMs currently active on the running kernel by returning their numeric IDs. The caller supplies a `__u64` array and its capacity (in entries); the kernel writes one ID per active LSM in stacking order (the order in which they're invoked for each hook) and returns the count written. As with `lsm_get_self_attr(2)`, the same out-parameter pattern is used: a `size` slot that, on `-E2BIG`, is updated to the required-size in **entries** (not bytes) so the caller can re-allocate.

This is the userspace-discoverable inventory of LSM stacking, replacing the inferred-from-`/sys/kernel/security/lsm` text file with a strict, kernel-defined integer-ID list.

Critical for: container runtimes building per-LSM label-set masks, systemd-condition checks, libselinux/libapparmor coexistence, debugger and audit-tool capability negotiation.

## Signature

```c
int lsm_list_modules(
    __u64 *ids,
    __u32 *size,    /* in: capacity in entries; out: number of entries written */
    __u32 flags     /* reserved; must be 0 */
);
```

```c
/* Known LSM IDs (subset; full list in linux/lsm.h) */
#define LSM_ID_CAPABILITY    100
#define LSM_ID_SELINUX       101
#define LSM_ID_SMACK         102
#define LSM_ID_TOMOYO        103
#define LSM_ID_APPARMOR      104
#define LSM_ID_YAMA          105
#define LSM_ID_LOADPIN       106
#define LSM_ID_SAFESETID     107
#define LSM_ID_LOCKDOWN      108
#define LSM_ID_BPF           109
#define LSM_ID_LANDLOCK      110
#define LSM_ID_IMA           111
#define LSM_ID_EVM           112
#define LSM_ID_IPE           113
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ids` | `__u64 *` | out | Buffer to receive LSM IDs. Each entry is `__u64` to leave room for vendor IDs. |
| `size` | `__u32 *` | in/out | In: capacity in entries. Out: number written, or required-size on `-E2BIG`. |
| `flags` | `__u32` | in | Reserved; MUST be `0`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of IDs written. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | `flags != 0`, `ids == NULL && *size != 0`. |
| `EFAULT` | `ids` or `size` user pointer invalid. |
| `E2BIG` | `*size` smaller than active LSM count; `*size` set to required, nothing written into `ids`. |
| `ENOSYS` | Kernel built without `CONFIG_SECURITY`. |

## ABI surface

```text
__NR_lsm_list_modules  (x86_64) = 461
__NR_lsm_list_modules  (arm64)  = 461
__NR_lsm_list_modules  (riscv)  = 461
__NR_lsm_list_modules  (i386)   = 461

/* Order is stacking-order: the order in which LSMs are invoked for any
   hook. Caller MUST treat this order as significant if it needs to know
   which LSM gets the first say on a decision. */
```

## Compatibility contract

REQ-1: Syscall number is **461** on x86_64. ABI-stable.

REQ-2: `flags` MUST be `0`. No flags defined.

REQ-3: `*size` is the IN capacity (entries, not bytes). Capacity less than active count ⟹ `-E2BIG` with `*size` updated to required.

REQ-4: `*size == 0` is legal; returns `-E2BIG` with `*size` set to required. `ids` is NOT dereferenced.

REQ-5: `ids == NULL` is legal only with `*size == 0`.

REQ-6: Output is in stable, kernel-defined stacking order — identical between calls on the same boot.

REQ-7: The "capability" LSM (`LSM_ID_CAPABILITY`) is always present and always first when listed.

REQ-8: BPF-LSM (`LSM_ID_BPF`) is reported as a single LSM regardless of how many BPF programs are attached.

REQ-9: Vendor / out-of-tree LSMs MAY appear with IDs in the vendor-reserved range; userspace MUST tolerate unknown IDs.

REQ-10: No capability is required to call this syscall.

REQ-11: Active LSM set is fixed at boot; this syscall's output never changes during a single boot.

REQ-12: Returned count fits in `__u32`; the active LSM count is bounded by `CONFIG_LSM_MAX` (small constant, typically < 64).

REQ-13: This syscall is the recommended way to detect specific-LSM presence; `/sys/kernel/security/lsm` is parser-fragile and remains for compatibility only.

REQ-14: Even if `*size` is sufficient, the kernel writes ONLY `count` entries; bytes beyond are NOT zeroed.

REQ-15: Successful return ⟹ `*size` equals the count actually written.

## Acceptance Criteria

- [ ] AC-1: Kernel with SELinux + capability: returns 2 entries, `[LSM_ID_CAPABILITY, LSM_ID_SELINUX]`.
- [ ] AC-2: Kernel with full stack: returns ≤ CONFIG_LSM_MAX entries.
- [ ] AC-3: `*size = 0` returns `-E2BIG` with `*size` set to active count.
- [ ] AC-4: `flags = 1` returns `-EINVAL`.
- [ ] AC-5: `ids == NULL && *size > 0` returns `-EINVAL`.
- [ ] AC-6: Capability LSM is always present and always first.
- [ ] AC-7: BPF-LSM (if enabled) appears exactly once regardless of attached prog count.
- [ ] AC-8: Repeated calls on same boot yield identical output.
- [ ] AC-9: Buffer exactly equal to count: succeeds and writes all entries.
- [ ] AC-10: No capability required to call.

## ABI surface (extended)

```rust
const CONFIG_LSM_MAX: usize = 64; /* Rookery bound; matches CONFIG_LSM_MAX */

#[repr(C)]
pub struct LsmIdList {
    pub ids: [u64; CONFIG_LSM_MAX],
    pub n: u32,
}
```

## Architecture

```rust
#[syscall(nr = 461, abi = "sysv")]
pub fn sys_lsm_list_modules(
    ids: UserPtr<u64>,
    size: UserPtr<u32>,
    flags: u32,
) -> isize {
    Lsm::list_modules(ids, size, flags)
}
```

`Lsm::list_modules(ids_p, size_p, flags) -> isize`:
1. if flags != 0 { return Err(EINVAL); }
2. let cap = size_p.read()?;                                   // EFAULT
3. if cap > 0 && ids_p.is_null() { return Err(EINVAL); }
4. let actives = Lsm::active_iter().map(|l| l.id()).collect::<Vec<u64>>();
5. let n = actives.len() as u32;
6. if cap < n {
7.    size_p.write(n)?;
8.    return Err(E2BIG);
9. }
10. for (i, id) in actives.iter().enumerate() {
11.    ids_p.add(i).write(*id)?;
12. }
13. size_p.write(n)?;
14. Ok(n as isize)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_strict_zero` | INVARIANT | `flags != 0` ⟹ `-EINVAL`. |
| `null_ids_with_capacity` | INVARIANT | `ids == NULL && cap > 0` ⟹ `-EINVAL`. |
| `undersize_returns_e2big` | INVARIANT | `cap < n` ⟹ `-E2BIG` with `*size` set to `n`. |
| `stacking_order_stable` | INVARIANT | consecutive calls produce identical output. |
| `cap_first` | INVARIANT | `LSM_ID_CAPABILITY` always at index 0. |
| `bounded_count` | INVARIANT | `n <= CONFIG_LSM_MAX`. |

### Layer 2: TLA+

`security/lsm-list-modules.tla`:
- Per-call transitions: parse, count, fit-check, write.
- Properties:
  - `safety_no_partial_write` — buffer either fully populated with `n` entries or untouched on error.
  - `safety_size_truthful` — `*size` after return equals entries written.
  - `safety_order_deterministic` — output order = boot-time stacking order.
  - `liveness_terminates` — every call returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Lsm::active_iter` post: stable order per-boot | `Lsm::active_iter` |
| `Lsm::list_modules` post: ret == *size | `Lsm::list_modules` |
| `Lsm::list_modules` post: -E2BIG ⟹ no entries written | `Lsm::list_modules` |
| `Lsm::list_modules` post: cap >= n ⟹ all entries written | `Lsm::list_modules` |

### Layer 4: Verus / Creusot functional

Per-`lsm_list_modules(2)` man page; per-`Documentation/userspace-api/lsm.rst`; replaces `/sys/kernel/security/lsm` text-parse path.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`lsm_list_modules(2)` reinforcement:

- **Per-`flags == 0` strict** — defense against per-future-flag smuggle.
- **Per-undersize buffer atomic E2BIG** — defense against per-info-leak via partial write.
- **Per-stacking-order stable per-boot** — defense against per-nondeterminism for caller logic.
- **Per-capability-LSM-always-first** — defense against per-misordering attack on caller's checks.

## Grsecurity / PaX surface

- **PaX UDEREF on `ids` and `size`** — defense against per-pointer smuggling; SMAP forced for the inevitable `copy_to_user`.
- **LSM stacking surface restricted** — `GRKERNSEC_LSM_STACK_LOCKDOWN` ensures the active LSM set is frozen at boot; this syscall reflects only the boot-time set.
- **PAX_USERCOPY on id array write** — bounded `copy_to_user` of `n * 8` bytes uses whitelisted slab; rejects oversize attempts.
- **Per-`E2BIG` size-only path** — required-size is the **only** information returned on undersize; no IDs leaked.
- **GRKERNSEC_PROC_GETPID identity** — even though syscall is unrestricted, audit subsystem records caller for forensic correlation when `GRKERNSEC_AUDIT_LSM_LIST` is set.
- **Per-vendor-LSM ID range allow-list** — `GRKERNSEC_LSM_VENDOR_DENY` strips vendor-range IDs from output unless the caller is `CAP_SYS_ADMIN` in init userns (defense against fingerprinting third-party LSMs from unprivileged userspace).
- **PAX_REFCOUNT on LSM module refcount during iteration** — defense against per-LSM-rmmod race.
- **GRKERNSEC_HIDESYM** — output is integer IDs only; no kernel-pointer cookies ever leak.
- **Per-id stability across `lsm_get_self_attr` calls** — defense against per-TOCTOU between list and per-LSM query.
- **Per-active-iter snapshotted** — defense against per-mid-iteration LSM rmmod via stale pointer.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `lsm_get_self_attr(2)` (see `lsm_get_self_attr.md`).
- `lsm_set_self_attr(2)` (see `lsm_set_self_attr.md`).
- Per-LSM internal data (Tier-3 `security/<lsm>.md`).
- Implementation code.
