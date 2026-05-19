# Tier-5 syscall: semop(2) — syscall 65

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - ipc/sem.c (SYSCALL_DEFINE3(semop), do_semtimedop, perform_atomic_semop, update_queue)
  - include/uapi/linux/sem.h (struct sembuf, SEM_UNDO, SEMOPM, SEMVMX)
  - arch/x86/entry/syscalls/syscall_64.tbl (65  common  semop)
-->

## Summary

`semop(2)` performs a vector of operations on the System V semaphore set `semid`. Each `struct sembuf` element specifies one operation: a semaphore number (`sem_num`), a signed delta (`sem_op`), and a flag set (`sem_flg`: `IPC_NOWAIT`, `SEM_UNDO`).

The kernel applies the entire vector **atomically**: either all `nsops` operations complete or none do. If any operation cannot proceed (sub-zero on `sem_op < 0`, non-zero on `sem_op == 0`), the call blocks until all become possible (or `IPC_NOWAIT` returns `EAGAIN`). `SEM_UNDO` records a reverse delta in the process's `sem_undo` list so adjustments are reverted automatically on process exit.

`semop` is equivalent to `semtimedop(semid, sops, nsops, NULL)`. Critical for: classic SysV synchronization, oracle / db engines, init systems.

## Signature

```c
int semop(int semid, struct sembuf *sops, size_t nsops);

struct sembuf {
    unsigned short sem_num;
    short          sem_op;
    short          sem_flg;
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `semid` | `int` | in | Set id from `semget(2)`. |
| `sops`  | `struct sembuf *` | in | Vector of operations. |
| `nsops` | `size_t` | in | Vector length, `1..=SEMOPM` (default 500). |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; all operations applied. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `E2BIG` | `nsops > SEMOPM`. |
| `EACCES` | Insufficient permission (alter requires write, P0 requires read). |
| `EAGAIN` | `IPC_NOWAIT` and at least one op would block. |
| `EFAULT` | `sops` not addressable. |
| `EFBIG` | `sem_num >= sma->sem_nsems`. |
| `EIDRM` | Set removed while blocked. |
| `EINTR` | Signal caught while blocked. |
| `EINVAL` | Bad `semid`, `nsops == 0`. |
| `ENOMEM` | Out of memory for `SEM_UNDO` allocation. |
| `ERANGE` | Resulting semval would exceed `SEMVMX` (32767) or go below 0 for SEM_UNDO entry. |

## ABI surface

```text
__NR_semop  (x86_64)  = 65
__NR_semop  (arm64)   = via __NR_semtimedop = 192 (semop deprecated on new ports)
__NR_semop  (riscv)   = via semtimedop_time64
__NR_semop  (i386)    = via ipc(2) multiplexer (call 3)

semop is internally do_semtimedop with timeout=NULL.
SEMVMX = 32767 (max semval).
SEMAEM = 32767 (max adjust-on-exit value).
```

## Compatibility contract

REQ-1: Syscall number is **65** on x86_64. ABI-stable.

REQ-2: `1 <= nsops <= ns->sem_ctlopm (SEMOPM)`, else `E2BIG`. `nsops == 0` ⟹ `EINVAL`.

REQ-3: Atomicity: kernel attempts the entire op vector while holding the set's spinlock (or `sem_base[i].lock` if all ops touch a single sem — fastpath). If any op blocks, all prior side-effects are rolled back before sleeping.

REQ-4: Per-op semantics:
- `sem_op > 0`: `semval += sem_op`; never blocks (unless overflow > SEMVMX). Requires write permission.
- `sem_op == 0`: wait for `semval == 0`. Requires read permission.
- `sem_op < 0`: requires `semval >= |sem_op|`; if so, `semval -= |sem_op|`; else block. Requires write permission.

REQ-5: `sem_flg & SEM_UNDO`: kernel allocates / locates a `sem_undo` for (current.signal, semid); reverse delta is added to `semadj[sem_num]`. On `do_exit()`, the kernel applies `+semadj[*]` to each semaphore to revert. Clamped by `SEMAEM`.

REQ-6: `sem_flg & IPC_NOWAIT` set: if vector cannot complete immediately, returns `-EAGAIN`.

REQ-7: Blocking semantics: queue this caller on `sma->pending_alter` (if any sem_op != 0) or `sma->pending_const` (if all sem_op == 0). Drop spinlock, schedule. Re-enter `perform_atomic_semop` on wake.

REQ-8: `IPC_RMID` on set while blocked ⟹ `-EIDRM`. Signal ⟹ `-EINTR`.

REQ-9: On success: per-op semaphore `sempid = current.tgid`. `sma->sem_otime = now`.

REQ-10: `copy_from_user(sops, nsops * sizeof(struct sembuf))`; on stack if `nsops <= SEMOPM_FAST` (~64), heap otherwise (kmalloc, `ENOMEM` possible).

REQ-11: LSM hook: `security_sem_semop(sma, sops, nsops, alter_bool)`.

REQ-12: Permission check ladder: any sem_op > 0 OR < 0 ⟹ require write (S_IWUGO). All ops zero ⟹ read (S_IRUGO).

## Acceptance Criteria

- [ ] AC-1: `semop(id, [{0,+1,0}], 1)` increments sem 0.
- [ ] AC-2: `semop(id, [{0,-1,0}], 1)` decrements sem 0 if >= 1, else blocks.
- [ ] AC-3: `nsops = 0` ⟹ `-EINVAL`.
- [ ] AC-4: `nsops > SEMOPM` ⟹ `-E2BIG`.
- [ ] AC-5: Atomicity: 2-op vector where 2nd would block: 1st is not visibly applied during sleep.
- [ ] AC-6: `IPC_NOWAIT` on blocked op: `-EAGAIN`.
- [ ] AC-7: SIGUSR1 on blocked op: `-EINTR`.
- [ ] AC-8: RMID while blocked: `-EIDRM`.
- [ ] AC-9: `SEM_UNDO`: process exit reverts changes.
- [ ] AC-10: Overflow past SEMVMX: `-ERANGE`.
- [ ] AC-11: `sem_num >= sma->sem_nsems`: `-EFBIG`.
- [ ] AC-12: sempid updated to current.tgid.

## Architecture

```rust
#[syscall(nr = 65, abi = "sysv")]
pub fn sys_semop(semid: i32, sops: UserPtr<Sembuf>, nsops: usize) -> isize {
    Ipc::do_semtimedop(semid, sops, nsops, None)
}
```

`Ipc::do_semtimedop(semid, uptr, nsops, timeout) -> isize`:
1. let ns = current_ipc_ns();
2. if nsops == 0 { return -EINVAL; }
3. if nsops > ns.sem_ctlopm as usize { return -E2BIG; }
4. let sops = Ipc::copy_sops_from_user(uptr, nsops)?;        // EFAULT
5. let alter = sops.iter().any(|s| s.sem_op != 0);
6. let undo = sops.iter().any(|s| (s.sem_flg & SEM_UNDO) != 0);
7. let mut sma = Ipc::sma_lock(ns, semid)?;                  // EINVAL/EIDRM
8. Ipc::ipcperms(&sma.sem_perm, if alter { S_IWUGO } else { S_IRUGO })?;
9. for s in &sops { if s.sem_num as u32 >= sma.sem_nsems { return -EFBIG; } }
10. Ipc::security_sem_semop(&sma, &sops, alter)?;
11. let su = if undo { Some(Ipc::find_alloc_undo(ns, &sma, current())?) } else { None };
12. loop {
13.   let r = Ipc::perform_atomic_semop(&mut sma, &sops, su.as_deref_mut());
14.   match r {
15.     Ok(()) => break,
16.     Err(EAgain) => {
17.       if (sops[0].sem_flg & IPC_NOWAIT) != 0 || sops.iter().any(|s| s.sem_flg & IPC_NOWAIT != 0) {
18.         return -EAGAIN;
19.       }
20.       Ipc::queue_and_sleep(&mut sma, &sops, alter, timeout)?;  // EIDRM/EINTR/ETIMEDOUT
21.     }
22.     Err(e) => return -e,
23.   }
24. }
25. for s in &sops { sma.sem_base[s.sem_num as usize].sempid = current().tgid; }
26. sma.sem_otime = now_seconds();
27. Ipc::update_queue(&mut sma);     // wake other waiters whose vectors now succeed
28. return 0;

`Ipc::perform_atomic_semop(sma, sops, undo) -> Result<(), Errno>`:
1. /* Tentative apply; rollback on failure */
2. let mut applied: Vec<(usize, i16)> = Vec::with_capacity(sops.len());
3. for s in sops {
4.   let cur = sma.sem_base[s.sem_num as usize].semval;
5.   let next = cur as i32 + s.sem_op as i32;
6.   if s.sem_op == 0 && cur != 0 { rollback(&mut sma, &applied); return Err(EAgain); }
7.   if s.sem_op < 0 && cur < (-s.sem_op as i32) { rollback(); return Err(EAgain); }
8.   if next > SEMVMX as i32 { rollback(); return Err(ERANGE); }
9.   sma.sem_base[s.sem_num as usize].semval = next as u16;
10.  applied.push((s.sem_num as usize, s.sem_op));
11. }
12. if let Some(u) = undo {
13.   for s in sops {
14.     let adj = u.semadj[s.sem_num as usize] as i32 - s.sem_op as i32;
15.     if adj > SEMAEM as i32 || adj < -(SEMAEM as i32) { /* undo rollback + sma rollback */ return Err(ERANGE); }
16.     u.semadj[s.sem_num as usize] = adj as i16;
17.   }
18. }
19. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `atomicity_all_or_none` | INVARIANT | failure ⟹ no semval change visible. |
| `sem_num_bound` | INVARIANT | sem_num < sma.sem_nsems else EFBIG. |
| `nsops_in_semopm` | INVARIANT | 1<=nsops<=SEMOPM. |
| `semvmx_bound` | INVARIANT | next semval <= SEMVMX else ERANGE. |
| `undo_balance` | INVARIANT | SEM_UNDO records reverse delta; exit applies +semadj. |
| `eagain_no_partial` | INVARIANT | block path: applied vector rolled back before sleep. |

### Layer 2: TLA+

`ipc/semop.tla`:
- States: copy-in, perm-check, tentative-apply, rollback, queue+sleep, wake, finalize.
- Properties:
  - `safety_atomic_vector` — vector applied atomically with respect to other semop callers.
  - `safety_semval_bounded` — 0 <= semval <= SEMVMX.
  - `safety_eidrm_kills_waiter` — RMID terminates each waiter with EIDRM.
  - `safety_undo_inverse` — SEM_UNDO ⟹ semadj[i] += -sem_op.
  - `liveness_wake_when_possible` — when condition becomes satisfiable, some waiter wakes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `perform_atomic_semop` post: success ⟹ all semvals updated | `Ipc::perform_atomic_semop` |
| `update_queue` post: only callers whose vectors succeed are woken | `Ipc::update_queue` |
| `do_semtimedop` post: success ⟹ sempid[*touched*] = current.tgid | `Ipc::do_semtimedop` |

### Layer 4: Verus / Creusot functional

Per-`semop(2)` man-page; LTP `ipc/semop*`; SUSv4 conformance.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`semop(2)` reinforcement:

- **Per-vector atomicity** — defense against per-partial-update race.
- **Per-SEMOPM cap** — defense against per-vector-DoS.
- **Per-SEMVMX cap** — defense against per-semval-overflow.
- **Per-SEM_UNDO bounded by SEMAEM** — defense against per-undo overflow.
- **Per-LSM hook** — defense against per-policy bypass.

## Grsecurity / PaX surface

- **PaX UDEREF on copy_from_user(sops)** — defense against per-sops kernel-deref bug; SMAP forced. SOPS staging buffer is stack-allocated when small, kmalloc'd otherwise; never aliased to user.
- **PAX_USERCOPY whitelist** — sem_array, sem_undo, sem slabs are usercopy-deny; semop does no copy_to_user.
- **GRKERNSEC_HARDEN_IPC** — write/read permission requires euid match cuid OR `CAP_IPC_OWNER` in set's user-ns; world-writable mode bits do not grant cross-uid alter.
- **CAP_IPC_OWNER in set's user-ns strict** — defense against cross-userns synchronization-bypass.
- **SEMOPM / SEMVMX / SEMAEM per-namespace cap** — sysctls per-ipc-ns; child userns cannot raise above init-ns ceiling.
- **PAX_REFCOUNT on sem_perm + sem_undo refcounts** — defense against per-RMID race UAF and per-undo-list lifetime UAF.
- **IPC namespace strict** — pending_alter, pending_const, sem_undo lists per-ns.
- **GRKERNSEC_PROC_IPC info-leak** — sempid (last-PID-to-op-this-sem) exposed only to root or sem cuid; defense against per-IPC pid-discovery channel.
- **SEM_UNDO list audited on do_exit** — defense against per-undo-list corruption silently leaking semval.
- **GRKERNSEC_AUDIT_IPC** — every semop on a set whose cuid != current emits audit record.
- **SIGNAL_INTERRUPT POSIX-strict** — no ERESTARTSYS; defense against per-restart-replay (would re-copy_from_user, possibly altered sops).
- **GRKERNSEC_RANDSTRUCT on sem_array** — defense against per-heap-spray on sem_array fields.
- **PaX KERNEXEC** — pending_alter / pending_const list pages NX.
- **PAX_RAP / Control Flow Integrity** — update_queue uses pinned indirect calls; defense against per-wake-callback-corruption.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Timed variant (covered in `semtimedop.md`).
- Set admin (covered in `semctl.md`).
- Allocation (covered in `semget.md`).
- Implementation code.
- Pre-2.4 i386 ipc(2) trampoline.
