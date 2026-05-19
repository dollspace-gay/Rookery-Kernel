---
title: "Tier-5 syscall: shmget(2) — syscall 29"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`shmget(2)` returns a System V shared-memory segment identifier for `key`. If `key == IPC_PRIVATE`, or no segment exists for `key` and `IPC_CREAT` is set, a new segment of `size` bytes (rounded up to PAGE_SIZE, or to the hugepage size if `SHM_HUGETLB` is set) is created. Backing store is by default tmpfs (`shmem_kernel_file_setup`); with `SHM_HUGETLB`, hugetlbfs is used (caller chooses pool via `SHM_HUGE_*` flags).

Initial segment contents are zero. The segment must subsequently be mapped into the caller's address space via `shmat(2)`. Critical for: System V IPC interop, oracle / db engines, postgres shared buffers, container init.

### Acceptance Criteria

- [ ] AC-1: `shmget(IPC_PRIVATE, PAGE_SIZE, 0600)` returns non-negative id.
- [ ] AC-2: `shmget(K, PAGE_SIZE, IPC_CREAT|IPC_EXCL|0600)` then repeat ⟹ `-EEXIST`.
- [ ] AC-3: `shmget(K, PAGE_SIZE, 0)` when none exists ⟹ `-ENOENT`.
- [ ] AC-4: `size = 0` on create ⟹ `-EINVAL`.
- [ ] AC-5: `size > SHMMAX` ⟹ `-EINVAL`.
- [ ] AC-6: `SHM_HUGETLB` without CAP_IPC_LOCK + not in hugetlb_shm_group ⟹ `-EPERM`.
- [ ] AC-7: Open existing with `size > shm_segsz` ⟹ `-EINVAL`.
- [ ] AC-8: SHMMNI segs ∧ one more ⟹ `-ENOSPC`.
- [ ] AC-9: Cross-CLONE_NEWIPC isolation.
- [ ] AC-10: New seg shm_nattch == 0, shm_segsz == PAGE_ALIGN(size).
- [ ] AC-11: ID composition: index < SHMMNI.

### Architecture

```rust
#[syscall(nr = 29, abi = "sysv")]
pub fn sys_shmget(key: i32, size: usize, shmflg: i32) -> isize {
    Ipc::shmget(key, size, shmflg)
}
```

`Ipc::shmget(key, size, shmflg) -> isize`:
1. let ns = current_ipc_ns();
2. if key == IPC_PRIVATE {
3.   if size < SHMMIN || size > ns.shm_ctlmax { return -EINVAL; }
4.   return Ipc::newseg(ns, key, size, shmflg);
5. }
6. let slot = ipc_findkey(&ns.shm_ids, key);
7. match slot {
8.   None => {
9.     if (shmflg & IPC_CREAT) == 0 { return -ENOENT; }
10.    if size < SHMMIN || size > ns.shm_ctlmax { return -EINVAL; }
11.    return Ipc::newseg(ns, key, size, shmflg);
12.  }
13.  Some(shp) => {
14.    if (shmflg & (IPC_CREAT|IPC_EXCL)) == (IPC_CREAT|IPC_EXCL) { return -EEXIST; }
15.    if size > 0 && size > shp.shm_segsz { return -EINVAL; }
16.    Ipc::security_shm_associate(shp, shmflg)?;
17.    Ipc::ipcperms(&shp.shm_perm, shmflg & 0o777)?;
18.    return shp.shm_perm.id as isize;
19.  }
20. }

`Ipc::newseg(ns, key, size, shmflg) -> isize`:
1. let huge  = (shmflg & SHM_HUGETLB) != 0;
2. let nores = (shmflg & SHM_NORESERVE) != 0;
3. let aligned = if huge { hugepage_align(size, shmflg) } else { page_align(size) };
4. let pages = aligned / PAGE_SIZE;
5. if huge && !capable(CAP_IPC_LOCK) && !in_hugetlb_shm_group() { return -EPERM; }
6. if ns.shm_ids.in_use >= ns.shm_ctlmni as usize { return -ENOSPC; }
7. if ns.shm_tot + pages > ns.shm_ctlall { return -ENOSPC; }
8. let shp = ShmSegment::alloc()?;                            // ENOMEM
9. shp.shm_perm.mode = (shmflg & 0o777) as u16;
10. shp.shm_perm.key  = key;
11. shp.shm_perm.uid  = current_euid(); shp.shm_perm.cuid = shp.shm_perm.uid;
12. shp.shm_perm.gid  = current_egid(); shp.shm_perm.cgid = shp.shm_perm.gid;
13. shp.shm_segsz     = aligned;
14. shp.shm_cprid     = current().tgid;
15. shp.shm_ctim      = now_seconds();
16. shp.shm_file = if huge {
17.   hugetlb_file_setup(name, aligned, ACCT_DEFAULT, huge_size_shift(shmflg))?
18. } else {
19.   shmem_kernel_file_setup(name, aligned, VM_SHARED | if nores { VM_NORESERVE } else { 0 })?
20. };
21. Ipc::security_shm_alloc(&mut shp)?;
22. let id = ipc_addid(&ns.shm_ids, &mut shp.shm_perm, ns.shm_ctlmni)?;
23. ns.shm_tot += pages;
24. return shp.shm_perm.id as isize;

### Out of Scope

- shmat / shmdt (attach/detach) — covered in separate per-syscall docs.
- shmctl admin (covered in `shmctl.md`).
- Implementation code.
- Pre-2.4 i386 ipc(2) trampoline.

### signature

```c
int shmget(key_t key, size_t size, int shmflg);
```

```c
#define SHM_HUGETLB    04000
#define SHM_NORESERVE  010000
#define SHM_HUGE_SHIFT 26
#define SHM_HUGE_2MB   (21 << SHM_HUGE_SHIFT)
#define SHM_HUGE_1GB   (30 << SHM_HUGE_SHIFT)
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `key`    | `key_t` | in | IPC key; `IPC_PRIVATE` requests a fresh segment. |
| `size`   | `size_t` | in | Segment size in bytes; on create, `SHMMIN <= size <= SHMMAX`. |
| `shmflg` | `int` | in | `IPC_CREAT|IPC_EXCL`, 0777 mode, `SHM_HUGETLB`, `SHM_NORESERVE`, `SHM_HUGE_*`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Shared-memory identifier. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Segment exists but caller lacks permission per mode. |
| `EEXIST` | `IPC_CREAT|IPC_EXCL` and segment exists. |
| `EINVAL` | On create: `size < SHMMIN` or `size > SHMMAX`. On open: `size > existing seg size`. |
| `ENOENT` | No segment exists and `IPC_CREAT` clear. |
| `ENOMEM` | Out of memory. |
| `ENOSPC` | `SHMMNI` exceeded, or total pages > `SHMALL`. |
| `EPERM` | `SHM_HUGETLB` without `CAP_IPC_LOCK` (and not in hugetlb_shm_group). |

### abi surface

```text
__NR_shmget  (x86_64)  = 29
__NR_shmget  (arm64)   = 194
__NR_shmget  (riscv)   = 194
__NR_shmget  (i386)    = via ipc(2) multiplexer (call 23)

SHMMIN = 1.
SHMMAX = ULONG_MAX (per-ns sysctl, default unlimited modulo PAGE_SIZE).
SHMALL = max total pages of SysV shm (default unlimited).
SHMMNI = max segs per-ns (default 4096).
```

### compatibility contract

REQ-1: Syscall number is **29** on x86_64. ABI-stable.

REQ-2: `key == IPC_PRIVATE`: always create new segment; ignore `IPC_EXCL`.

REQ-3: `key != IPC_PRIVATE`: lookup in current `ipc_namespace`. Not found ∧ `IPC_CREAT` ⟹ create. `IPC_CREAT|IPC_EXCL` ∧ found ⟹ `-EEXIST`. Not found ∧ `!IPC_CREAT` ⟹ `-ENOENT`.

REQ-4: Create: `SHMMIN <= size <= ns->shm_ctlmax` else `-EINVAL`. Page-rounded internally; with `SHM_HUGETLB`, rounded to hugepage size.

REQ-5: Open existing: if `size > shp->shm_segsz` ⟹ `-EINVAL`. If `size == 0` on open, no check.

REQ-6: Permission on existing: `ipcperms(shp, shmflg & 0777)`; `CAP_IPC_OWNER` in user-ns bypasses.

REQ-7: `SHM_HUGETLB`: requires `CAP_IPC_LOCK` or membership in `vm.hugetlb_shm_group`. Hugepage pool selected by `SHM_HUGE_*` (encoded shift in bits 26-31 of shmflg). Default to `SHM_HUGE_SHIFT == 0` ⟹ default huge size.

REQ-8: `SHM_NORESERVE`: no swap reservation up front (sparse).

REQ-9: Per-ns quotas: `in_use + 1 > ns->shm_ctlmni (SHMMNI)` ⟹ `-ENOSPC`. `shm_tot + pages > ns->shm_ctlall (SHMALL)` ⟹ `-ENOSPC`.

REQ-10: New segment: `shm_perm.uid = cuid = euid(current)`, `gid = cgid = egid(current)`, `mode = shmflg & 0o777`, `shm_segsz = size`, `shm_cprid = current.tgid`, `shm_ctim = now`, `shm_atim = shm_dtim = 0`, `shm_nattch = 0`.

REQ-11: Backing file: `shmem_kernel_file_setup(name, size, VM_NORESERVE?, VM_ACCT?)`. With `SHM_HUGETLB`, `hugetlb_file_setup(...)`. File reference kept in `shp->shm_file`.

REQ-12: LSM hooks: `security_shm_alloc(shp)` on create; `security_shm_associate(shp, shmflg)` on open.

REQ-13: ID composition: `(seq << 16) | index`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `size_bounds_create` | INVARIANT | SHMMIN<=size<=SHMMAX on create. |
| `size_bounds_open` | INVARIANT | open: size<=shp.shm_segsz. |
| `excl_rejects_existing` | INVARIANT | IPC_CREAT|IPC_EXCL on existing ⟹ EEXIST. |
| `quota_shmmni_shmall` | INVARIANT | in_use+1<=SHMMNI; shm_tot+pages<=SHMALL. |
| `huge_cap_required` | INVARIANT | SHM_HUGETLB ⟹ CAP_IPC_LOCK or in hugetlb_shm_group. |
| `init_zero_contents` | INVARIANT | new seg backing pages are zero-initialized. |

### Layer 2: TLA+

`ipc/shmget.tla`:
- States: key-lookup, ENOENT/CREAT, EEXIST, newseg backing, ID-install.
- Properties:
  - `safety_per_ns_quota` — SHMMNI / SHMALL never exceeded.
  - `safety_creds_capture` — cuid/cgid captured.
  - `safety_namespace_isolation` — seg invisible across CLONE_NEWIPC.
  - `safety_huge_cap` — SHM_HUGETLB requires cap.
  - `liveness_terminates` — shmget returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `shmget` post: success ⟹ id ∈ ns.shm_ids active set | `Ipc::shmget` |
| `newseg` post: shm_file backed by shmem or hugetlbfs | `Ipc::newseg` |
| `ipc_findkey` post: returns Some iff key in ns | `Ipc::ipc_findkey` |

### Layer 4: Verus / Creusot functional

Per-`shmget(2)` man-page; LTP `ipc/shmget*`; SUSv4 conformance.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`shmget(2)` reinforcement:

- **Per-SHMMNI / SHMALL / SHMMAX quotas per-ns** — defense against per-seg / per-page exhaustion.
- **Per-cuid capture** — defense against per-ownership spoof.
- **Per-LSM alloc/associate** — defense against per-policy bypass.
- **Per-SHM_HUGETLB cap gate** — defense against per-hugepage-DoS.
- **Per-zero-initialized backing** — defense against per-stale-contents disclosure.

### grsecurity / pax surface

- **PaX UDEREF irrelevant (scalar args)** — SMAP/SMEP active.
- **GRKERNSEC_HARDEN_IPC** — opening existing shm seg requires euid match cuid OR `CAP_IPC_OWNER` in seg's user-ns. World-readable mode 0666 does NOT grant cross-uid attach.
- **CAP_IPC_OWNER strict in seg's user-ns** — defense against cross-userns priv-raise.
- **CAP_IPC_LOCK strict in seg's user-ns for SHM_HUGETLB** — defense against unprivileged hugepage-pool DoS. `vm.hugetlb_shm_group` mapping is per-ns and never escapes init.
- **SHMMAX / SHMALL / SHMMNI per-namespace cap** — sysctls per-ipc-ns; child userns cannot raise above init-ns ceiling.
- **GRKERNSEC_PROC_IPC** — `/proc/sysvipc/shm` listing restricted to root or seg cuid; shm_cprid / shm_lprid / shm_atim hidden from non-owner.
- **GRKERNSEC_RAND_IDS** — option to randomize the `seq` component of shm ids defeating predictability.
- **PAX_REFCOUNT on shm_perm + shm_file refcounts** — defense against per-RMID race UAF and per-file-leak.
- **IPC namespace strict** — even root in child userns cannot reach parent-ns segs; backing files in tmpfs/hugetlbfs are per-ns mounts.
- **SHM_NORESERVE policy** — if vm.overcommit_memory == 2 (strict accounting), SHM_NORESERVE rejected without `CAP_IPC_LOCK`. Defense against per-overcommit DoS.
- **GRKERNSEC_AUDIT_IPC** — emits audit record on every create (key, size, uid, mode, hugetlb-flag) + every EACCES.
- **PAX_USERCOPY_HARDEN whitelist** — shm_segment slab usercopy-deny; shmget does no copy, but accidental copy_to_user traps.
- **Initial zero-init enforced** — tmpfs / hugetlbfs zeroes pages on first fault; defense against per-stale-content disclosure from reused pages.
- **PaX KERNEXEC + W^X** — shm-backed pages mapped RW only via shmat; never executable in kernel mapping.
- **PAX_RANDKSTACK** — kstack randomized for shmget entry point.
- **GRKERNSEC_DENYSHMUSER** — option to disable SysV shm entirely in unprivileged userns (force POSIX shm_open instead).

