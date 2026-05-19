---
title: "Tier-5 syscall: shmctl(2) — syscall 31"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`shmctl(2)` performs control operations on a System V shared-memory segment. `op` selects:

- `IPC_STAT`: copy `shmid_ds` to `buf`.
- `IPC_SET`: apply selected fields from `buf` (`shm_perm.uid/gid/mode`).
- `IPC_RMID`: mark segment for destruction; actual teardown deferred until `shm_nattch == 0`.
- `IPC_INFO`: copy `struct shminfo` (system-wide tunables).
- `SHM_INFO`: copy `struct shm_info` (live counters).
- `SHM_STAT`: treat `shmid` as table index; return id + copy `shmid_ds`.
- `SHM_STAT_ANY`: like SHM_STAT but bypass per-seg permission check.
- `SHM_LOCK` / `SHM_UNLOCK`: pin backing pages in RAM, blocking swap-out.

Critical for: segment lifecycle, memory-pinning, observability (`ipcs -m`).

### Acceptance Criteria

- [ ] AC-1: `IPC_STAT` returns correct metadata.
- [ ] AC-2: `IPC_SET` changes mode bits; subsequent IPC_STAT reflects.
- [ ] AC-3: `IPC_RMID` with attachers: defers destroy; shmat blocked.
- [ ] AC-4: `IPC_RMID` with zero attach: destroy immediate.
- [ ] AC-5: `SHM_LOCK` without CAP_IPC_LOCK ⟹ `-EPERM`.
- [ ] AC-6: `SHM_LOCK` with CAP and within RLIMIT_MEMLOCK: pins pages; observable in /proc/PID/status VmLck after attach.
- [ ] AC-7: `SHM_LOCK` exceeding RLIMIT_MEMLOCK: `-ENOMEM`.
- [ ] AC-8: `SHM_UNLOCK` releases pinning.
- [ ] AC-9: Unknown op ⟹ `-EINVAL`.
- [ ] AC-10: `SHM_STAT_ANY` without CAP ⟹ `-EPERM`.
- [ ] AC-11: `IPC_INFO` matches sysctl.
- [ ] AC-12: `SHM_INFO` shm_tot matches sum of seg pages.
- [ ] AC-13: SHM_LOCK on hugetlbfs seg: no-op success.
- [ ] AC-14: `buf == NULL` for IPC_STAT ⟹ `-EFAULT`.

### Architecture

```rust
#[syscall(nr = 31, abi = "sysv")]
pub fn sys_shmctl(shmid: i32, op: i32, buf: UserPtr<u8>) -> isize {
    Ipc::shmctl(shmid, op, buf)
}
```

`Ipc::shmctl(shmid, op, buf) -> isize`:
1. let ns = current_ipc_ns();
2. let cmd = op & !IPC_64;
3. match cmd {
4.   IPC_INFO            => Ipc::shmctl_ipc_info(ns, buf as UserPtr<Shminfo>),
5.   SHM_INFO            => Ipc::shmctl_shm_info(ns, buf as UserPtr<ShmInfo>),
6.   IPC_STAT | SHM_STAT | SHM_STAT_ANY
7.                       => Ipc::shmctl_stat(ns, shmid, cmd, buf as UserPtr<ShmidDs>),
8.   IPC_SET | IPC_RMID  => Ipc::shmctl_down(ns, shmid, cmd, buf as UserPtr<ShmidDs>),
9.   SHM_LOCK | SHM_UNLOCK
10.                      => Ipc::shmctl_do_lock(ns, shmid, cmd),
11.  _                   => -EINVAL,
12. }

`Ipc::shmctl_down(ns, shmid, op, buf) -> isize`:
1. let mut new = ShmidDs::default();
2. if op == IPC_SET { unsafe { buf.copy_in(&mut new)?; } }
3. let mut shp = Ipc::shp_lock(ns, shmid)?;                   // EINVAL/EIDRM
4. Ipc::ipc_check_perms(&shp.shm_perm, IPC_OWNER)?;           // EPERM
5. Ipc::security_shm_shmctl(&shp, op)?;
6. match op {
7.   IPC_SET => {
8.     shp.shm_perm.uid  = new.shm_perm.uid;
9.     shp.shm_perm.gid  = new.shm_perm.gid;
10.    shp.shm_perm.mode = (shp.shm_perm.mode & !0o777) | (new.shm_perm.mode & 0o777);
11.    shp.shm_ctim      = now_seconds();
12.    Ok(0)
13.  }
14.  IPC_RMID => {
15.    shp.shm_perm.mode |= SHM_DEST;
16.    shp.shm_perm.deleted = true;
17.    ipc_rmid(&ns.shm_ids, &mut shp.shm_perm);
18.    if shp.shm_nattch == 0 { Ipc::shm_destroy(ns, &mut shp); }
19.    Ok(0)
20.  }
21. }

`Ipc::shmctl_do_lock(ns, shmid, op) -> isize`:
1. let mut shp = Ipc::shp_lock(ns, shmid)?;
2. match op {
3.   SHM_LOCK => {
4.     if !ns_capable(seg_user_ns(&shp), CAP_IPC_LOCK) { return -EPERM; }
5.     if shp.shm_file.is_hugetlb() { /* already pinned */ shp.shm_perm.mode |= SHM_LOCKED; return Ok(0); }
6.     let n = shp.shm_segsz / PAGE_SIZE;
7.     let lim = rlimit(RLIMIT_MEMLOCK, &seg_owner_creds(&shp));
8.     if locked_pages_for_uid(seg_cuid(&shp)) + n > lim/PAGE_SIZE { return -ENOMEM; }
9.     shmem_lock(&shp.shm_file, 1, seg_owner_creds(&shp))?;
10.    shp.shm_perm.mode |= SHM_LOCKED;
11.    Ok(0)
12.  }
13.  SHM_UNLOCK => {
14.    /* Either CAP_IPC_LOCK in seg's userns or seg-owner; */
15.    if !ns_capable(seg_user_ns(&shp), CAP_IPC_LOCK)
16.       && !ipc_check_perms(&shp.shm_perm, IPC_OWNER).is_ok() { return -EPERM; }
17.    if shp.shm_file.is_hugetlb() { shp.shm_perm.mode &= !SHM_LOCKED; return Ok(0); }
18.    shmem_lock(&shp.shm_file, 0, seg_owner_creds(&shp))?;
19.    shp.shm_perm.mode &= !SHM_LOCKED;
20.    Ok(0)
21.  }
22. }

`Ipc::shm_destroy(ns, shp)`:
1. ns.shm_tot -= shp.shm_segsz / PAGE_SIZE;
2. if (shp.shm_perm.mode & SHM_LOCKED) != 0 { shmem_lock(&shp.shm_file, 0, ...); }
3. fput(shp.shm_file);
4. security_shm_free(shp);
5. call_rcu(|| Ipc::free_shp(shp));

### Out of Scope

- shmat / shmdt (attach/detach) — covered in separate per-syscall docs.
- Allocation (covered in `shmget.md`).
- Implementation code.
- Pre-2.4 i386 ipc(2) trampoline.

### signature

```c
int shmctl(int shmid, int op, struct shmid_ds *buf);
```

```c
struct shmid_ds {
    struct ipc_perm shm_perm;
    size_t          shm_segsz;
    __kernel_time_t shm_atime, shm_dtime, shm_ctime;
    __kernel_pid_t  shm_cpid, shm_lpid;
    unsigned long   shm_nattch;
};
struct shminfo  { ulong shmmax, shmmin, shmmni, shmseg, shmall; };
struct shm_info { int used_ids; ulong shm_tot, shm_rss, shm_swp, swap_attempts, swap_successes; };
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `shmid` | `int` | in | Seg id or table index (SHM_STAT). |
| `op`    | `int` | in | One of `IPC_STAT`, `IPC_SET`, `IPC_RMID`, `IPC_INFO`, `SHM_INFO`, `SHM_STAT`, `SHM_STAT_ANY`, `SHM_LOCK`, `SHM_UNLOCK`. |
| `buf`   | `struct shmid_ds *` / `struct shminfo *` / `struct shm_info *` / NULL | in/out | Per-op interpretation. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success for `IPC_STAT`, `IPC_SET`, `IPC_RMID`, `SHM_LOCK`, `SHM_UNLOCK`. |
| `>= 0` | `IPC_INFO`/`SHM_INFO`: highest used index. `SHM_STAT`/`SHM_STAT_ANY`: seg id. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | `IPC_STAT` / `SHM_STAT` without read permission. |
| `EFAULT` | `buf` not accessible. |
| `EIDRM` | Segment marked destroyed during operation. |
| `EINVAL` | Bad `op`, bad `shmid`, bad `SHM_STAT` index. |
| `ENOMEM` | `SHM_LOCK` cannot pin all pages within rlimit. |
| `EOVERFLOW` | `IPC_STAT` field would overflow user struct size. |
| `EPERM` | `IPC_SET`/`IPC_RMID` without `CAP_IPC_OWNER` and euid != owner; `SHM_LOCK` without `CAP_IPC_LOCK`. |

### abi surface

```text
__NR_shmctl  (x86_64)  = 31
__NR_shmctl  (arm64)   = 195
__NR_shmctl  (riscv)   = 195
__NR_shmctl  (i386)    = via ipc(2) multiplexer (call 24)

IPC_64 bit OR'd into op selects modern 64-bit-time shmid_ds layout.
SHM_LOCK is the only op that pins pages; it uses RLIMIT_MEMLOCK budget per UID.
```

### compatibility contract

REQ-1: Syscall number is **31** on x86_64. ABI-stable.

REQ-2: `IPC_STAT`: requires read permission. RCU-snapshot of `shmid_ds`, `copy_to_user`.

REQ-3: `IPC_SET`: requires `(euid in {uid, cuid}) ∨ CAP_IPC_OWNER` in seg's user-ns. Mutates only `shm_perm.uid`, `gid`, `mode`. Updates `shm_ctime`.

REQ-4: `IPC_RMID`: same caller requirement as `IPC_SET`. Marks segment with `SHM_DEST` (`shm_perm.deleted = true`). If `shm_nattch == 0`, calls `shm_destroy` immediately; else destruction deferred until last detach. Subsequent `shmat` requests fail.

REQ-5: `IPC_INFO`: copy static `shminfo` of ns ceilings. Returns highest in-use id.

REQ-6: `SHM_INFO`: copy live `shm_info` (used_ids, total pages, RSS, swap).

REQ-7: `SHM_STAT`: `shmid` as index `[0, SHMMNI)`. Returns encoded id (`seq << 16 | index`) + copies `shmid_ds`. Requires read-perm.

REQ-8: `SHM_STAT_ANY`: like SHM_STAT, skips per-seg perm; requires `CAP_IPC_OWNER` (or `CAP_SYS_ADMIN` legacy) in init userns. Used by `ipcs`.

REQ-9: `SHM_LOCK`: requires `CAP_IPC_LOCK` (in seg's user-ns). Sets `shp->shm_perm.mode |= SHM_LOCKED`. Calls `shmem_lock(file, 1)` which iterates backing pages, applying `mlock` accounting (counts against `RLIMIT_MEMLOCK` of seg's `cuid`). On `ENOMEM` rolls back.

REQ-10: `SHM_UNLOCK`: requires `CAP_IPC_LOCK` (in seg's user-ns) OR ownership. Clears `SHM_LOCKED`; calls `shmem_lock(file, 0)` releasing accounting.

REQ-11: LSM hook `security_shm_shmctl(shp, op)`.

REQ-12: `IPC_64` bit selects modern 64-bit-time layout (kernel internal).

REQ-13: `shm_destroy`: drops shm_file ref; if last, tmpfs/hugetlbfs file is freed; pages released to allocator.

REQ-14: `SHM_LOCK` only sensible on shmem-backed segs; hugetlbfs segs always pinned, so SHM_LOCK is a no-op there.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `op_dispatch_total` | INVARIANT | every op handled exactly once. |
| `rmid_defer_until_zero_nattch` | INVARIANT | RMID + nattch>0 ⟹ defer destroy; SHM_DEST mark set. |
| `shm_lock_cap` | INVARIANT | SHM_LOCK requires CAP_IPC_LOCK in seg user-ns. |
| `shm_lock_rlimit` | INVARIANT | SHM_LOCK pages + currently-locked-for-uid <= RLIMIT_MEMLOCK. |
| `shm_lock_rollback` | INVARIANT | ENOMEM mid-lock ⟹ no pages remain pinned. |
| `unknown_op_einval` | INVARIANT | op ∉ table ⟹ EINVAL. |

### Layer 2: TLA+

`ipc/shmctl.tla`:
- States: dispatch, perm-check, mutate, lock/unlock, rmid, deferred-destroy.
- Properties:
  - `safety_rmid_defer_to_zero` — destroy only when last detach.
  - `safety_lock_caps` — pinning requires CAP_IPC_LOCK.
  - `safety_lock_rlimit_bound` — never exceed RLIMIT_MEMLOCK.
  - `liveness_destroy_eventually` — once SHM_DEST + last detach ⟹ memory freed.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `shmctl_down` post: IPC_SET ⟹ shm_ctim updated; mode masked to 0o777 | `Ipc::shmctl_down` |
| `shmctl_do_lock` post: SHM_LOCK ⟹ shm_perm.mode | SHM_LOCKED on success | `Ipc::shmctl_do_lock` |
| `shm_destroy` post: shm_file ref dropped; ns.shm_tot decremented | `Ipc::shm_destroy` |

### Layer 4: Verus / Creusot functional

Per-`shmctl(2)` man-page; LTP `ipc/shmctl*`; SUSv4 conformance.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`shmctl(2)` reinforcement:

- **Per-op dispatch** — defense against per-op-confusion.
- **Per-RMID deferred destroy** — defense against per-UAF on still-attached seg.
- **Per-SHM_LOCK CAP gate + RLIMIT_MEMLOCK** — defense against per-pin DoS.
- **Per-LSM hook** — defense against per-policy bypass.

### grsecurity / pax surface

- **PaX UDEREF on copy_from_user(buf) for IPC_SET + copy_to_user for IPC_STAT** — defense against per-buf kernel-deref bug; SMAP forced. shmid_ds / shminfo / shm_info staged via stack buffers; the slab is usercopy-deny.
- **GRKERNSEC_HARDEN_IPC** — IPC_SET / IPC_RMID / SHM_LOCK require euid match cuid OR `CAP_IPC_OWNER` (or `CAP_IPC_LOCK` for lock) in seg's user-ns. World-writable mode 0666 does NOT grant cross-uid mutate.
- **CAP_IPC_OWNER strict in seg's user-ns** — defense against cross-userns admin-bypass.
- **CAP_IPC_LOCK strict in seg's user-ns for SHM_LOCK** — defense against unprivileged pin / DoS. Even with the cap, RLIMIT_MEMLOCK of the seg's cuid bounds total pinning.
- **SHM_STAT_ANY restricted to init userns** — even with CAP_IPC_OWNER, SHM_STAT_ANY in a child userns is rejected (would expose parent-ns seg metadata).
- **GRKERNSEC_PROC_IPC info-leak** — IPC_STAT / SHM_STAT fields shm_cpid / shm_lpid / shm_atime hidden from non-owner; SHM_INFO swap_attempts / swap_successes leaked only to root.
- **GRKERNSEC_AUDIT_IPC** — IPC_SET, IPC_RMID, SHM_LOCK, SHM_UNLOCK emit audit records (uid, key, id, op, success/fail, pages).
- **IPC_RMID drains via RCU + SHM_DEST mark + audit** — defense against silent SysV-shm teardown.
- **IPC namespace strict** — SHM_INFO counts per-ns; SHM_STAT scans only owning ns's id-array.
- **PAX_REFCOUNT on shm_perm + shm_file refcounts** — defense against per-RMID race UAF and per-file-leak.
- **GRKERNSEC_RANDSTRUCT on shmid_ds** — randomized layout defeats per-offset info-leak.
- **PAX_USERCOPY_HARDEN** — per-op buf copy uses bounded length; defense against per-overcopy.
- **SHM_LOCK rollback on ENOMEM** — defense against per-partial-pin leak. shmem_lock iterates pages with strict accounting and unwinds on failure.
- **SHM_LOCK page-aging override audited** — SHM_LOCKED pages do not migrate; grsec emits a record if a long-lived large pinned seg crosses a configurable threshold.
- **PaX KERNEXEC + W^X** — backing pages NX in kernel mapping.
- **PAX_RANDKSTACK + STACKLEAK** — kstack randomized and erased on return.
- **No `CAP_SYS_ADMIN` shortcut** — grsec removes the legacy "CAP_SYS_ADMIN bypasses CAP_IPC_LOCK" path; only CAP_IPC_LOCK satisfies SHM_LOCK.

