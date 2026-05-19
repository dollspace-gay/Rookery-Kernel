---
title: "Tier-5 syscall: semctl(2) — syscall 66"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`semctl(2)` performs control operations on a System V semaphore set. The fourth argument `arg` is a `union semun` whose interpretation depends on `op`:

- `IPC_STAT` / `IPC_SET`: `arg.buf` is `struct semid_ds *`.
- `IPC_INFO` / `SEM_INFO`: `arg.__buf` is `struct seminfo *`.
- `IPC_RMID`: `arg` ignored.
- `GETVAL`: returns single semval as `int`.
- `SETVAL`: `arg.val` is the new value for `sem_num` semaphore.
- `GETALL` / `SETALL`: `arg.array` is `unsigned short *`.
- `GETPID` / `GETNCNT` / `GETZCNT`: return scalar metadata for `sem_num`.
- `SEM_STAT` / `SEM_STAT_ANY`: treat `semid` as table index.

Critical for: semaphore lifecycle, observability, and per-sem value queries.

### Acceptance Criteria

- [ ] AC-1: `IPC_STAT` copies set metadata correctly.
- [ ] AC-2: `SETVAL` to value 5: subsequent `GETVAL` returns 5.
- [ ] AC-3: `SETVAL` to SEMVMX+1 ⟹ `-ERANGE`.
- [ ] AC-4: `SETVAL` on `sem_num >= sem_nsems` ⟹ `-EINVAL`.
- [ ] AC-5: `SETALL` updates entire vector; `GETALL` reflects it.
- [ ] AC-6: `SETALL` with one element > SEMVMX ⟹ `-ERANGE`, no partial update.
- [ ] AC-7: `IPC_RMID` removes set; subsequent `semop` ⟹ `-EINVAL`.
- [ ] AC-8: `IPC_RMID` wakes blocked semop with `-EIDRM`.
- [ ] AC-9: `IPC_SET` by non-owner without CAP ⟹ `-EPERM`.
- [ ] AC-10: `GETPID` returns last sempid.
- [ ] AC-11: `GETNCNT` matches active waiters with sem_op<0.
- [ ] AC-12: `SEM_INFO` `semaem` / `semusz` counters track live state.
- [ ] AC-13: SETVAL resets undo entries to 0 for that sem.
- [ ] AC-14: Unknown `op` ⟹ `-EINVAL`.

### Architecture

```rust
#[syscall(nr = 66, abi = "sysv")]
pub fn sys_semctl(semid: i32, sem_num: i32, op: i32, arg: usize) -> isize {
    Ipc::semctl(semid, sem_num, op, arg)
}
```

`Ipc::semctl(semid, sem_num, op, arg) -> isize`:
1. let ns = current_ipc_ns();
2. let cmd = op & !IPC_64;
3. match cmd {
4.   IPC_INFO | SEM_INFO        => return Ipc::semctl_info(ns, cmd, arg as UserPtr<Seminfo>),
5.   SEM_STAT | SEM_STAT_ANY    => return Ipc::semctl_stat(ns, semid, cmd, arg as UserPtr<SemidDs>),
6.   IPC_STAT                   => return Ipc::semctl_stat(ns, semid, cmd, arg as UserPtr<SemidDs>),
7.   GETVAL|GETPID|GETNCNT|GETZCNT|GETALL|SETVAL|SETALL
8.                              => return Ipc::semctl_main(ns, semid, sem_num, cmd, arg),
9.   IPC_SET | IPC_RMID         => return Ipc::semctl_down(ns, semid, cmd, arg as UserPtr<SemidDs>),
10.  _                          => return -EINVAL,
11. }

`Ipc::semctl_main(ns, semid, sem_num, op, arg) -> isize`:
1. let mut sma = Ipc::sma_lock(ns, semid)?;
2. let alter = matches!(op, SETVAL | SETALL);
3. Ipc::ipcperms(&sma.sem_perm, if alter { S_IWUGO } else { S_IRUGO })?;
4. match op {
5.   GETVAL  => { check_sem_num(); Ok(sma.sem_base[sem_num].semval as isize) }
6.   GETPID  => { check_sem_num(); Ok(sma.sem_base[sem_num].sempid as isize) }
7.   GETNCNT => { check_sem_num(); Ok(sma.sem_base[sem_num].semncnt as isize) }
8.   GETZCNT => { check_sem_num(); Ok(sma.sem_base[sem_num].semzcnt as isize) }
9.   GETALL  => {
10.    let mut buf = vec![0u16; sma.sem_nsems as usize];
11.    for i in 0..sma.sem_nsems { buf[i] = sma.sem_base[i].semval; }
12.    unsafe { (arg as UserPtr<u16>).copy_out_slice(&buf)?; }
13.    Ok(0)
14.  }
15.  SETVAL => {
16.    check_sem_num();
17.    let v = arg as i32;
18.    if v < 0 || v > SEMVMX as i32 { return -ERANGE; }
19.    sma.sem_base[sem_num].semval = v as u16;
20.    sma.sem_base[sem_num].sempid = current().tgid;
21.    Ipc::reset_undo_for_sem(&mut sma, sem_num);
22.    sma.sem_ctime = now_seconds();
23.    Ipc::update_queue(&mut sma);
24.    Ok(0)
25.  }
26.  SETALL => {
27.    let mut buf = vec![0u16; sma.sem_nsems as usize];
28.    unsafe { (arg as UserPtr<u16>).copy_in_slice(&mut buf)?; }
29.    for v in &buf { if *v > SEMVMX { return -ERANGE; } }
30.    for i in 0..sma.sem_nsems { sma.sem_base[i].semval = buf[i]; sma.sem_base[i].sempid = 0; }
31.    Ipc::reset_undo_all(&mut sma);
32.    sma.sem_ctime = now_seconds();
33.    Ipc::update_queue(&mut sma);
34.    Ok(0)
35.  }
36. }

`Ipc::semctl_down(ns, semid, op, arg) -> isize`:
1. let mut sma = Ipc::sma_lock(ns, semid)?;
2. Ipc::ipc_check_perms(&sma.sem_perm, IPC_OWNER)?;
3. Ipc::security_sem_semctl(&sma, op)?;
4. match op {
5.   IPC_SET => {
6.     let mut new = SemidDs::default();
7.     unsafe { arg.copy_in(&mut new)?; }
8.     sma.sem_perm.uid  = new.sem_perm.uid;
9.     sma.sem_perm.gid  = new.sem_perm.gid;
10.    sma.sem_perm.mode = new.sem_perm.mode & 0o777;
11.    sma.sem_ctime     = now_seconds();
12.    Ok(0)
13.  }
14.  IPC_RMID => {
15.    Ipc::freeary(ns, &mut sma);
16.    Ok(0)
17.  }
18. }

### Out of Scope

- semop / semtimedop (covered in `semop.md`, `semtimedop.md`).
- Allocation (covered in `semget.md`).
- Implementation code.
- Pre-2.4 i386 ipc(2) trampoline.

### signature

```c
int semctl(int semid, int sem_num, int op, ...);

union semun {
    int                 val;
    struct semid_ds    *buf;
    unsigned short     *array;
    struct seminfo     *__buf;
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `semid`   | `int` | in | Set id or table index (SEM_STAT). |
| `sem_num` | `int` | in | Semaphore index (used by GETVAL/SETVAL/GETPID/GETNCNT/GETZCNT). |
| `op`      | `int` | in | Control op (see Summary). |
| `arg`     | `union semun` (varargs) | in/out | Per-op interpretation. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success for `IPC_SET`, `IPC_RMID`, `SETVAL`, `SETALL`. |
| `>= 0` | `GETVAL` returns semval; `GETPID` returns sempid; `GETNCNT`/`GETZCNT` return counters; `IPC_INFO`/`SEM_INFO` return highest index; `SEM_STAT`/`SEM_STAT_ANY` return id. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Insufficient permission. |
| `EFAULT` | `arg.buf`/`array`/`__buf` not addressable. |
| `EIDRM` | Set removed during operation. |
| `EINVAL` | Bad `op`, bad `semid`, bad `sem_num`, SETALL value > SEMVMX. |
| `EPERM` | `IPC_SET`/`IPC_RMID` without `CAP_IPC_OWNER` and euid != cuid != uid. |
| `ERANGE` | `SETVAL`/`SETALL` value exceeds `SEMVMX`. |

### abi surface

```text
__NR_semctl  (x86_64)  = 66
__NR_semctl  (arm64)   = 191
__NR_semctl  (riscv)   = 191
__NR_semctl  (i386)    = via ipc(2) multiplexer (call 3)

IPC_64 bit OR'd into op selects modern semid_ds layout.
seminfo layout: { semmap, semmni, semmns, semmnu, semmsl, semopm, semume, semusz, semvmx, semaem }.
```

### compatibility contract

REQ-1: Syscall number is **66** on x86_64. ABI-stable.

REQ-2: Op dispatch:
- `IPC_STAT` / `SEM_STAT` / `SEM_STAT_ANY`: read; copy_to_user `semid_ds`. SEM_STAT_ANY skips per-set perm check (needs CAP_IPC_OWNER in user-ns).
- `IPC_SET`: requires owner / cap; mutable fields are `sem_perm.uid`, `gid`, `mode`. Updates `sem_ctime`.
- `IPC_RMID`: requires owner / cap; frees set, wakes all waiters with EIDRM.
- `IPC_INFO`: copy static `seminfo` (ns ceilings).
- `SEM_INFO`: copy live `seminfo` (counts).
- `GETVAL`: read; require read-perm. Returns `sma->sem_base[sem_num].semval`.
- `GETPID`: read; returns `sempid`.
- `GETNCNT`: read; count of waiters whose `sem_op < 0` on `sem_num`.
- `GETZCNT`: read; count of waiters whose `sem_op == 0` on `sem_num`.
- `GETALL`: read; copy `sem_nsems` shorts.
- `SETVAL`: write; set `semval = arg.val` (range 0..=SEMVMX). Resets all `semadj` undo entries for that sem across all procs to 0. Wakes waiters.
- `SETALL`: write; copy array of `sem_nsems` shorts; range 0..=SEMVMX. Resets all sempid to 0; resets undo entries. Wakes waiters. Updates `sem_ctime`.

REQ-3: Permission ladder: read ops require `S_IRUGO`; write/set ops require `S_IWUGO`; admin ops (IPC_SET, IPC_RMID, SEM_STAT_ANY) require ownership or `CAP_IPC_OWNER`.

REQ-4: `sem_num` must be `< sma->sem_nsems` for GETVAL / SETVAL / GETPID / GETNCNT / GETZCNT; else `EINVAL`.

REQ-5: `SETVAL` / `SETALL` value > `SEMVMX` (32767): `-ERANGE`.

REQ-6: `IPC_RMID`: marks set deleted, drains `pending_alter` + `pending_const` with `EIDRM`, frees `sem_undo` linkages, RCU-deferred free.

REQ-7: LSM hook `security_sem_semctl(sma, op)`.

REQ-8: After SETVAL/SETALL: `update_queue(sma)` re-tests all blocked waiters for satisfiability.

REQ-9: `IPC_64` bit OR'd into `op` selects modern 64-bit-time layout.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `op_dispatch_total` | INVARIANT | every op handled exactly once. |
| `sem_num_bound` | INVARIANT | sem_num<sem_nsems for per-sem ops. |
| `setval_range` | INVARIANT | SETVAL/SETALL value <= SEMVMX. |
| `setall_no_partial` | INVARIANT | any value > SEMVMX in SETALL ⟹ no update applied. |
| `rmid_wakes_all_eidrm` | INVARIANT | RMID frees set + waiters all get EIDRM. |
| `setval_resets_undo` | INVARIANT | SETVAL resets undo entries for that sem to 0. |

### Layer 2: TLA+

`ipc/semctl.tla`:
- States: dispatch, perm-check, mutate, update_queue, rmid drain.
- Properties:
  - `safety_rmid_eidrm_to_all` — RMID terminates every waiter with EIDRM.
  - `safety_setall_atomicity` — either all values applied or none.
  - `safety_undo_reset_on_setval` — semadj[i] = 0 across all procs on SETVAL of sem i.
  - `liveness_update_queue_wakes_satisfiable` — waiters whose vectors now satisfy wake.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `semctl_main` post: GETALL ⟹ user buf = sma snapshot | `Ipc::semctl_main` |
| `semctl_down` post: IPC_SET ⟹ sem_ctime updated | `Ipc::semctl_down` |
| `freeary` post: deleted=true, waiters drained, undo unlinked | `Ipc::freeary` |

### Layer 4: Verus / Creusot functional

Per-`semctl(2)` man-page; LTP `ipc/semctl*`; SUSv4 conformance.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`semctl(2)` reinforcement:

- **Per-op dispatch** — defense against per-op-confusion.
- **Per-SETALL atomicity** — defense against per-partial-update.
- **Per-IPC_RMID EIDRM-broadcast** — defense against per-stuck-waiter.
- **Per-SETVAL undo-reset** — defense against per-stale-undo replay.
- **Per-LSM hook** — defense against per-policy bypass.

### grsecurity / pax surface

- **PaX UDEREF on copy_from_user / copy_to_user (buf, array, __buf)** — defense against per-arg kernel-deref bug; SMAP forced.
- **PAX_USERCOPY_HARDEN** — semid_ds / seminfo staged through stack buffers; sem_array slab is usercopy-deny.
- **GRKERNSEC_HARDEN_IPC** — IPC_SET / IPC_RMID / SETVAL / SETALL require euid match cuid OR `CAP_IPC_OWNER` in set's user-ns; world-writable mode 0666 does NOT grant cross-uid mutate.
- **CAP_IPC_OWNER strict in set's user-ns** — defense against cross-userns admin-bypass. SEM_STAT_ANY rejected outside init userns.
- **GRKERNSEC_PROC_IPC info-leak** — sempid / cuid / cgid in IPC_STAT zeroed for non-owner. SEM_STAT_ANY restricted to root/init-ns.
- **PAX_REFCOUNT on sem_perm + sem_undo** — defense against per-RMID race UAF and undo-list UAF.
- **GRKERNSEC_AUDIT_IPC** — IPC_SET, IPC_RMID, SETVAL, SETALL all emit audit records with (uid, key, id, op, success/fail).
- **IPC_RMID RCU-drain + audit** — defense against silent IPC set teardown post-priv-acquire.
- **IPC namespace strict** — SEM_INFO counters per-ns; SEM_STAT scans only owning ns's id-array.
- **GRKERNSEC_RANDSTRUCT on semid_ds + sem** — randomized layout defeats per-offset info-leak.
- **PaX KERNEXEC on update_queue traversal** — defense against per-wake-callback corruption.
- **PAX_RAP** — update_queue indirect-call CFI.
- **Undo-list cleanup on SETALL** — defense against per-leaked-undo-replay (resetting semval without resetting undo would otherwise create phantom adjusts on exit).
- **PAX_RANDKSTACK + STACKLEAK** — kstack randomized and erased on syscall return.

