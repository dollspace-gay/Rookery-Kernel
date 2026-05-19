---
title: "Tier-5 syscall: semget(2) — syscall 64"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`semget(2)` returns the System V semaphore-set identifier associated with `key`. If `key == IPC_PRIVATE`, or no set exists for `key` and `IPC_CREAT` is set, a new set of `nsems` semaphores is created. Initial semaphore values are `0`; they must be set via `semctl(SETALL)` or `semctl(SETVAL)` before use.

`nsems` must be `> 0` on creation; for opens of an existing set, `nsems` may be `0` (means "any size matches") or must be `<=` the set's actual size. Low nine bits of `semflg` form the access mode. Critical for: classic SysV synchronization, legacy daemons, container init systems.

### Acceptance Criteria

- [ ] AC-1: `semget(IPC_PRIVATE, 1, 0600)` returns non-negative id.
- [ ] AC-2: `semget(K, 0, IPC_CREAT|0600)` ⟹ `-EINVAL` (nsems must be > 0 on create).
- [ ] AC-3: `semget(K, SEMMSL+1, IPC_CREAT|0600)` ⟹ `-EINVAL`.
- [ ] AC-4: Open existing set with `nsems = 0`: returns id.
- [ ] AC-5: Open existing set with `nsems > sma->sem_nsems`: `-EINVAL`.
- [ ] AC-6: `IPC_CREAT|IPC_EXCL` on existing key: `-EEXIST`.
- [ ] AC-7: SEMMNI sets created; one more: `-ENOSPC`.
- [ ] AC-8: SEMMNS total sems reached: `-ENOSPC`.
- [ ] AC-9: Cross-namespace isolation: invisible across CLONE_NEWIPC.
- [ ] AC-10: All sems initialized to 0; sempid = 0.

### Architecture

```rust
#[syscall(nr = 64, abi = "sysv")]
pub fn sys_semget(key: i32, nsems: i32, semflg: i32) -> isize {
    Ipc::semget(key, nsems, semflg)
}
```

`Ipc::semget(key, nsems, semflg) -> isize`:
1. let ns = current_ipc_ns();
2. if nsems < 0 || nsems as usize > ns.sem_ctlmsl as usize { /* deferred to newary on create path */ }
3. if key == IPC_PRIVATE {
4.   if nsems <= 0 || nsems as u32 > ns.sem_ctlmsl { return -EINVAL; }
5.   return Ipc::newary(ns, key, nsems, semflg);
6. }
7. /* Find by key */
8. let slot = ipc_findkey(&ns.sem_ids, key);
9. match slot {
10.  None => {
11.    if (semflg & IPC_CREAT) == 0 { return -ENOENT; }
12.    if nsems <= 0 || nsems as u32 > ns.sem_ctlmsl { return -EINVAL; }
13.    return Ipc::newary(ns, key, nsems, semflg);
14.  }
15.  Some(sma) => {
16.    if (semflg & (IPC_CREAT|IPC_EXCL)) == (IPC_CREAT|IPC_EXCL) { return -EEXIST; }
17.    if nsems > 0 && (nsems as u32) > sma.sem_nsems { return -EINVAL; }
18.    Ipc::security_sem_associate(sma, semflg)?;
19.    Ipc::ipcperms(&sma.sem_perm, semflg & 0o777)?;
20.    return sma.sem_perm.id as isize;
21.  }
22. }

`Ipc::newary(ns, key, nsems, semflg) -> isize`:
1. if ns.used_sems + nsems as u64 > ns.sem_ctlmns { return -ENOSPC; }
2. if ns.sem_ids.in_use >= ns.sem_ctlmni { return -ENOSPC; }
3. let sma = SemArray::alloc(nsems as usize)?;               // ENOMEM
4. sma.sem_perm.mode = (semflg & 0o777) as u16;
5. sma.sem_perm.key  = key;
6. sma.sem_perm.uid  = current_euid(); sma.sem_perm.cuid = sma.sem_perm.uid;
7. sma.sem_perm.gid  = current_egid(); sma.sem_perm.cgid = sma.sem_perm.gid;
8. sma.sem_nsems     = nsems as u32;
9. sma.sem_ctime     = now_seconds();
10. for i in 0..nsems { sma.sem_base[i] = Semaphore::zero(); }
11. Ipc::security_sem_alloc(&mut sma)?;
12. let id = ipc_addid(&ns.sem_ids, &mut sma.sem_perm, ns.sem_ctlmni)?;
13. ns.used_sems += nsems as u64;
14. return sma.sem_perm.id as isize;

### Out of Scope

- Semaphore operations (covered in `semop.md`, `semtimedop.md`).
- Semaphore administration (covered in `semctl.md`).
- Implementation code.
- Pre-2.4 i386 ipc(2) trampoline.

### signature

```c
int semget(key_t key, int nsems, int semflg);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `key`    | `key_t` | in | IPC key; `IPC_PRIVATE` requests a fresh set. |
| `nsems`  | `int` | in | Semaphores in the set; `1..=SEMMSL` on create. |
| `semflg` | `int` | in | `IPC_CREAT|IPC_EXCL` and 0777 mode. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Semaphore-set identifier. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Set exists but caller lacks permission per mode bits. |
| `EEXIST` | `IPC_CREAT|IPC_EXCL` and set exists. |
| `EINVAL` | On create: `nsems < 1` or `nsems > SEMMSL`. On open: `nsems > existing size`. |
| `ENOENT` | No set exists and `IPC_CREAT` clear. |
| `ENOMEM` | Out of memory. |
| `ENOSPC` | System-wide `SEMMNI` or `SEMMNS` (total sems) exceeded. |

### abi surface

```text
__NR_semget  (x86_64)  = 64
__NR_semget  (arm64)   = 190
__NR_semget  (riscv)   = 190
__NR_semget  (i386)    = via ipc(2) multiplexer (call 2)

SEMMNI: per-ns max sem sets (default 32000).
SEMMSL: max sems per set (default 32000).
SEMMNS: per-ns max total sems = SEMMNI * SEMMSL by default.
SEMOPM: max ops per semop call (default 500).
SEMVMX: max semval (32767).
```

### compatibility contract

REQ-1: Syscall number is **64** on x86_64. ABI-stable.

REQ-2: `key == IPC_PRIVATE`: always create new set; ignore `IPC_EXCL`.

REQ-3: `key != IPC_PRIVATE`: lookup in current `ipc_namespace`. If not found ∧ `IPC_CREAT`: create. Both `IPC_CREAT|IPC_EXCL` ∧ found: `-EEXIST`. Not found ∧ `!IPC_CREAT`: `-ENOENT`.

REQ-4: On create: `1 <= nsems <= ns->sem_ctlmsl (SEMMSL)`; else `-EINVAL`.

REQ-5: On open: if `nsems > 0` and `nsems > sma->sem_nsems`, `-EINVAL`. If `nsems == 0`, no check.

REQ-6: Permission on existing set: `ipcperms(sma, semflg & 0777)`; `CAP_IPC_OWNER` in user-ns bypasses.

REQ-7: New set: `sem_perm.uid = cuid = euid(current)`, `gid = cgid = egid(current)`, `mode = semflg & 0777`. All `sem_base[i] = { semval=0, sempid=0, semzcnt=0, semncnt=0 }`. `sem_otime = 0`, `sem_ctime = now`.

REQ-8: Quota: total sems in namespace `+ nsems > ns->sem_ctlmns (SEMMNS)` ⟹ `-ENOSPC`. Set count `+ 1 > ns->sem_ctlmni (SEMMNI)` ⟹ `-ENOSPC`.

REQ-9: Per-ipc-namespace; CLONE_NEWIPC isolates.

REQ-10: LSM hooks: `security_sem_alloc(sma)` on create; `security_sem_associate(sma, semflg)` on open.

REQ-11: ID composition: `(seq << 16) | index`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `nsems_bounds_create` | INVARIANT | create path: 1<=nsems<=SEMMSL. |
| `nsems_check_open` | INVARIANT | open with nsems>0: nsems<=sma.sem_nsems. |
| `quota_semmns_total` | INVARIANT | used_sems+nsems<=SEMMNS. |
| `quota_semmni_sets` | INVARIANT | in_use+1<=SEMMNI. |
| `excl_rejects_existing` | INVARIANT | IPC_CREAT|IPC_EXCL on existing ⟹ EEXIST. |
| `init_zero_values` | INVARIANT | new set: every semval=0, sempid=0. |

### Layer 2: TLA+

`ipc/semget.tla`:
- States: key-lookup, ENOENT/CREAT, EEXIST, newary, ID-install.
- Properties:
  - `safety_per_ns_quota` — SEMMNI/SEMMNS never exceeded.
  - `safety_creds_capture` — cuid/cgid captured.
  - `safety_namespace_isolation` — set invisible across CLONE_NEWIPC.
  - `liveness_terminates` — semget returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `semget` post: success ⟹ id ∈ ns.sem_ids active set | `Ipc::semget` |
| `newary` post: sem_nsems = nsems, all semval=0 | `Ipc::newary` |
| `ipc_findkey` post: returns Some iff key matches in ns | `Ipc::ipc_findkey` |

### Layer 4: Verus / Creusot functional

Per-`semget(2)` man-page; LTP `ipc/semget*`; SUSv4 conformance.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`semget(2)` reinforcement:

- **Per-SEMMNI / SEMMNS / SEMMSL quotas per-ns** — defense against per-set / per-sem exhaustion.
- **Per-cuid capture** — defense against per-ownership spoofing.
- **Per-LSM alloc/associate** — defense against per-policy bypass.
- **Per-zero-init semaphores** — defense against per-stale-value disclosure.

### grsecurity / pax surface

- **PaX UDEREF irrelevant (scalar args)** — SMAP/SMEP still enforced.
- **GRKERNSEC_HARDEN_IPC** — opening an existing semaphore set requires euid match cuid OR `CAP_IPC_OWNER` in set's user-ns. Mode 0666 does NOT grant cross-uid access.
- **CAP_IPC_OWNER in set's user-ns strict** — defense against cross-userns priv-raise; the check uses `ns_capable(sma->ns->user_ns, CAP_IPC_OWNER)`, not init_user_ns.
- **SEMMSL / SEMMNS / SEMMNI per-namespace cap** — sysctls are per-ipc-namespace; child userns cannot raise above init-ns ceiling.
- **GRKERNSEC_PROC_IPC** — `/proc/sysvipc/sem` listing restricted to root or set's cuid; cross-uid info-leak (sempid etc.) hidden.
- **GRKERNSEC_RAND_IDS** — option to randomize the `seq` component of sem ids defeating predictability.
- **PAX_REFCOUNT on sem_perm refcount** — defense against per-RMID race UAF.
- **IPC namespace strict** — even root in child userns cannot reach parent-ns sem sets.
- **LSM `security_sem_alloc` mandatory** — defense against per-creation bypass; built-in stub still records auditable record.
- **GRKERNSEC_AUDIT_IPC** — emits audit record on every successful create + every EACCES denial (per-uid, per-key, per-mode).
- **PAX_USERCOPY whitelist denial on sem_array slab** — semget does no copy, but slab is registered usercopy-deny so accidental copy_to_user against it traps.
- **Initial semval = 0 enforced** — defense against per-leaked-state from prior owner of reused slab.
- **PAX_RANDKSTACK** — semget runs under randomized kstack, defeating ROP gadget chains against this entry point.

