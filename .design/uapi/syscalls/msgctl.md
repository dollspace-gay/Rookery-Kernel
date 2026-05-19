# Tier-5 syscall: msgctl(2) — syscall 71

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - ipc/msg.c (SYSCALL_DEFINE3(msgctl), msgctl_down, msgctl_info, msgctl_stat, freeque)
  - ipc/util.c (ipc_get, ipc_rmid)
  - include/uapi/linux/msg.h (struct msqid_ds, struct msginfo, MSG_INFO, MSG_STAT, MSG_STAT_ANY)
  - include/uapi/linux/ipc.h (IPC_STAT, IPC_SET, IPC_RMID, IPC_INFO)
  - arch/x86/entry/syscalls/syscall_64.tbl (71  common  msgctl)
-->

## Summary

`msgctl(2)` performs control operations on a System V message queue. `op` selects one of:
`IPC_STAT` (copy `struct msqid_ds` to `buf`), `IPC_SET` (apply selected fields from `buf` — `msg_perm.uid/gid/mode`, `msg_qbytes`), `IPC_RMID` (mark queue for deletion and wake all waiters), `IPC_INFO` (copy system-wide `struct msginfo` limits to `buf`), `MSG_INFO` (like IPC_INFO but with live counters), `MSG_STAT` (treat `msqid` as an index into the kernel's id table, return id in retval), `MSG_STAT_ANY` (like MSG_STAT but skip per-queue permission check — requires `CAP_IPC_OWNER`).

Critical for: queue lifecycle management, sysadmin observability (`ipcs(1)`), and quota tuning.

## Signature

```c
int msgctl(int msqid, int op, struct msqid_ds *buf);
```

```c
struct msqid_ds {
    struct ipc_perm msg_perm;
    __kernel_time_t  msg_stime;
    __kernel_time_t  msg_rtime;
    __kernel_time_t  msg_ctime;
    unsigned long    msg_cbytes;
    unsigned long    msg_qnum;
    unsigned long    msg_qbytes;
    __kernel_pid_t   msg_lspid;
    __kernel_pid_t   msg_lrpid;
};
struct msginfo {
    int msgpool, msgmap, msgmax, msgmnb, msgmni,
        msgssz, msgtql; unsigned short msgseg;
};
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `msqid` | `int` | in | Queue id (or table index for `MSG_STAT`/`MSG_STAT_ANY`). |
| `op`    | `int` | in | One of `IPC_STAT`, `IPC_SET`, `IPC_RMID`, `IPC_INFO`, `MSG_INFO`, `MSG_STAT`, `MSG_STAT_ANY`. |
| `buf`   | `struct msqid_ds *` / `struct msginfo *` | in/out | Per-op interpretation. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success for `IPC_STAT`, `IPC_SET`, `IPC_RMID`. |
| `>= 0` | For `IPC_INFO`/`MSG_INFO`: highest used index. For `MSG_STAT`/`MSG_STAT_ANY`: queue id. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EACCES` | `IPC_STAT` without read permission; `MSG_STAT` index entry not readable. |
| `EFAULT` | `buf` not accessible. |
| `EIDRM` | Queue removed during operation. |
| `EINVAL` | Bad `op`, bad `msqid`, `IPC_SET`'s `msg_qbytes` > `MSGMNB` and not `CAP_SYS_RESOURCE`. |
| `EPERM` | `IPC_SET`/`IPC_RMID` without `CAP_IPC_OWNER` (in user-ns), and euid != cuid != uid of queue. |

## ABI surface

```text
__NR_msgctl  (x86_64)  = 71
__NR_msgctl  (arm64)   = 187
__NR_msgctl  (riscv)   = 187
__NR_msgctl  (i386)    = via ipc(2) multiplexer (call 14)

Compat: 32-bit/64-bit msqid_ds variants (msqid_ds_v1, msqid_ds_v2).
IPC_64 bit OR'd into op selects modern layout (kernel-internal).
```

## Compatibility contract

REQ-1: Syscall number is **71** on x86_64. ABI-stable.

REQ-2: `IPC_STAT`: requires read permission. Kernel snapshots `msqid_ds` under RCU and `copy_to_user`. Returns `0`.

REQ-3: `IPC_SET`: requires (euid in {uid, cuid}) ∨ `CAP_IPC_OWNER` in queue's user-ns. Reads `msqid_ds` from user. Only `msg_perm.uid`, `msg_perm.gid`, `msg_perm.mode` (low 9 bits), and `msg_qbytes` are mutable. Raising `msg_qbytes` above `MSGMNB` requires `CAP_SYS_RESOURCE`. Updates `msg_ctime`.

REQ-4: `IPC_RMID`: same caller requirement as `IPC_SET`. Marks queue as deleted (sets `msq_perm.deleted = true`), wakes all `q_senders` and `q_receivers` with `-EIDRM`, frees all enqueued messages, drops table slot. Subsequent operations on the id return `-EINVAL`.

REQ-5: `IPC_INFO`: copies static `struct msginfo` of namespace ceilings. Returns highest-in-use id (per-ns index).

REQ-6: `MSG_INFO`: like `IPC_INFO` but `msgpool` = current bytes used, `msgmap` = current message count, `msgtql` = total messages ever.

REQ-7: `MSG_STAT`: `msqid` treated as index in [0, MSGMNI). Returns the encoded id (`seq << 16 | index`) and copies the `msqid_ds`. Requires per-queue read permission.

REQ-8: `MSG_STAT_ANY`: like `MSG_STAT` but skips per-queue mode check. Requires `CAP_IPC_OWNER` in the queue's user-ns (or `CAP_SYS_ADMIN` legacy). Used by `ipcs -m`.

REQ-9: `IPC_64`: ABI bit OR'd into `op` for the modern 64-bit-time `msqid_ds`. Kernel splits dispatch internally; userland libc handles transparently.

REQ-10: Concurrent `IPC_RMID` and any other op: RCU-protected; loser sees `-EIDRM`.

REQ-11: LSM hooks: `security_msg_queue_msgctl(msq, op)` may deny.

## Acceptance Criteria

- [ ] AC-1: `IPC_STAT` copies queue metadata; mode and uid/gid match creation.
- [ ] AC-2: `IPC_SET` changes mode 0600 -> 0660 visible by next `IPC_STAT`.
- [ ] AC-3: `IPC_SET` raising `msg_qbytes` past `MSGMNB` without CAP_SYS_RESOURCE returns `-EINVAL`.
- [ ] AC-4: `IPC_RMID` removes queue; subsequent `msgsnd` returns `-EINVAL`.
- [ ] AC-5: `IPC_RMID` wakes blocked sender/receiver with `-EIDRM`.
- [ ] AC-6: `IPC_INFO` returns `msginfo` matching sysctl values.
- [ ] AC-7: `MSG_INFO` `msgpool` matches sum of `q_cbytes`.
- [ ] AC-8: `MSG_STAT` on index without read-perm returns `-EACCES`.
- [ ] AC-9: `MSG_STAT_ANY` without `CAP_IPC_OWNER` returns `-EPERM`.
- [ ] AC-10: Unknown `op` returns `-EINVAL`.
- [ ] AC-11: `buf == NULL` for `IPC_STAT` returns `-EFAULT`.
- [ ] AC-12: `IPC_SET` by non-owner without CAP returns `-EPERM`.

## Architecture

```rust
#[syscall(nr = 71, abi = "sysv")]
pub fn sys_msgctl(msqid: i32, op: i32, buf: UserPtr<u8>) -> isize {
    Ipc::msgctl(msqid, op, buf)
}
```

`Ipc::msgctl(msqid, op, buf) -> isize`:
1. let ns = current_ipc_ns();
2. let cmd = op & !IPC_64;
3. match cmd {
4.   IPC_INFO | MSG_INFO => return Ipc::msgctl_info(ns, cmd, buf),
5.   MSG_STAT | MSG_STAT_ANY => return Ipc::msgctl_stat(ns, msqid, cmd, buf),
6.   IPC_STAT => return Ipc::msgctl_stat(ns, msqid, cmd, buf),
7.   IPC_SET | IPC_RMID => return Ipc::msgctl_down(ns, msqid, cmd, buf),
8.   _ => return -EINVAL,
9. }

`Ipc::msgctl_down(ns, msqid, op, buf) -> isize`:
1. let mut new = MsqidDs::default();
2. if op == IPC_SET { unsafe { buf.copy_in(&mut new)?; } }
3. let mut msq = Ipc::msq_lock(ns, msqid)?;                   // EINVAL/EIDRM
4. Ipc::ipc_check_perms(&msq.q_perm, IPC_OWNER)?;             // EPERM
5. Ipc::security_msg_queue_msgctl(&msq, op)?;
6. match op {
7.   IPC_SET => {
8.     if new.msg_qbytes > ns.msg_ctlmnb && !capable(CAP_SYS_RESOURCE) { return -EINVAL; }
9.     msq.q_perm.uid  = new.msg_perm.uid;
10.    msq.q_perm.gid  = new.msg_perm.gid;
11.    msq.q_perm.mode = new.msg_perm.mode & 0o777;
12.    msq.q_qbytes    = new.msg_qbytes;
13.    msq.q_ctime     = now_seconds();
14.    return 0;
15.  }
16.  IPC_RMID => {
17.    Ipc::freeque(ns, &mut msq);
18.    return 0;
19.  }
20.  _ => unreachable!(),
21. }

`Ipc::freeque(ns, msq)`:
1. msq.q_perm.deleted = true;
2. ipc_rmid(&ns.msg_ids, &mut msq.q_perm);
3. /* Wake all senders + receivers with EIDRM */
4. expunge_all(&mut msq.q_senders, EIDRM);
5. expunge_all(&mut msq.q_receivers, EIDRM);
6. for m in msq.q_messages.drain(..) { Ipc::free_msg(m); }
7. ipc_unlock_object(&msq.q_perm);
8. call_rcu(|| Ipc::free_msq(msq));

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `set_owner_or_cap` | INVARIANT | IPC_SET requires euid match or CAP_IPC_OWNER. |
| `set_qbytes_caprule` | INVARIANT | msg_qbytes > MSGMNB ⟹ requires CAP_SYS_RESOURCE. |
| `rmid_idempotent_on_concurrent` | INVARIANT | RCU: only one RMID succeeds; rest EIDRM. |
| `rmid_wakes_all_waiters_eidrm` | INVARIANT | RMID expunges senders/receivers with EIDRM. |
| `unknown_op_einval` | INVARIANT | op ∉ {STAT,SET,RMID,INFO,MSG_INFO,MSG_STAT,MSG_STAT_ANY} ⟹ EINVAL. |
| `msg_stat_any_requires_cap` | INVARIANT | MSG_STAT_ANY ⟹ CAP_IPC_OWNER required. |

### Layer 2: TLA+

`ipc/msgctl.tla`:
- States: dispatch by op, owner check, mutate, rmid, wake-all-waiters.
- Properties:
  - `safety_rmid_eidrm_to_all` — RMID terminates every waiter with EIDRM.
  - `safety_no_partial_set` — IPC_SET either fully applies or none.
  - `safety_msg_stat_index_bounded` — index < MSGMNI.
  - `liveness_rmid_terminates` — RCU drain converges.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `msgctl_down` post: IPC_SET ⟹ msq.q_ctime updated | `Ipc::msgctl_down` |
| `freeque` post: q_qnum = 0, deleted=true, waiters drained | `Ipc::freeque` |
| `msgctl_stat` post: buf out has consistent snapshot | `Ipc::msgctl_stat` |

### Layer 4: Verus / Creusot functional

Per-`msgctl(2)` man-page; LTP `ipc/msgctl*`; SUSv4 conformance.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`msgctl(2)` reinforcement:

- **Per-op dispatch table** — defense against per-op-confusion.
- **Per-IPC_RMID wakes-all-EIDRM** — defense against per-blocked-waiter zombie.
- **Per-msg_qbytes raise gated by CAP_SYS_RESOURCE** — defense against per-quota-evasion.
- **Per-RCU teardown** — defense against per-UAF on concurrent RMID + STAT.

## Grsecurity / PaX surface

- **PaX UDEREF on copy_from_user(buf) for IPC_SET and copy_to_user for IPC_STAT** — defense against per-buf kernel-deref bug; SMAP forced. msqid_ds slab not user-copy whitelisted; copies are stack-buffered.
- **GRKERNSEC_HARDEN_IPC** — IPC_SET / IPC_RMID require euid match cuid OR explicit `CAP_IPC_OWNER` in queue's user-ns; world-writable mode bits (0666) do not grant cross-uid SET. The "anyone with mode write can RMID" loophole is closed: RMID always requires ownership/cap.
- **CAP_IPC_OWNER strict in queue's user-ns** — never init_user_ns; defense against cross-userns admin-bypass.
- **CAP_SYS_RESOURCE in queue's user-ns** — for raising msg_qbytes above MSGMNB; per-namespace ceiling cannot exceed init-ns ceiling.
- **GRKERNSEC_PROC_IPC** — IPC_STAT/MSG_STAT lspid/lrpid/cuid/cgid exposed only to root or queue cuid; otherwise zeroed; MSG_STAT_ANY information limited to admin.
- **MSG_STAT_ANY only in init userns** — even with CAP_IPC_OWNER, MSG_STAT_ANY in a child userns is rejected (it would expose parent-ns queue metadata).
- **PAX_REFCOUNT on q_perm** — defense against per-refcount-overflow UAF during RMID racing with STAT/SET.
- **IPC_RMID drains via RCU + audit emits IPC_REMOVED record** — defense against silent IPC teardown by attacker post-priv-acquire.
- **GRKERNSEC_AUDIT_IPC** — IPC_SET, IPC_RMID emit audit messages with (uid, queue-key, queue-id, op, success/fail).
- **IPC namespace strict** — MSG_INFO counters are per-ns; child-ns cannot see parent-ns aggregate.
- **PaX KERNEXEC during freeque** — message payload pages NX during teardown; defense against per-UAF-into-payload-exec.
- **GRKERNSEC_RANDSTRUCT on msqid_ds** — randomized layout; defense against per-info-leak via predictable field offset.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Send / receive data path (covered in `msgsnd.md`, `msgrcv.md`).
- Initial allocation (covered in `msgget.md`).
- Implementation code.
- Pre-2.4 i386 ipc(2) trampoline.
