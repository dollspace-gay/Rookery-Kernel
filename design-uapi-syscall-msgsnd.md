---
title: "Tier-5 syscall: msgsnd(2) — syscall 69"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`msgsnd(2)` appends a message to the System V message queue identified by `msqid`. The message is laid out as `struct msgbuf { long mtype; char mtext[]; }`; `mtype` must be strictly positive. `msgsz` is the size of `mtext` only, not including `mtype`. The kernel copies the full message in (allocating a kernel-side `struct msg_msg` chained from multi-page segments if `mtext` exceeds one page).

If the queue is full (cumulative byte count `q_cbytes + msgsz > q_qbytes`, or message-count > `q_qbytes/2` for tiny messages), the call blocks unless `IPC_NOWAIT` is set, in which case it returns `EAGAIN`. Critical for: System V IPC interop, deterministic batch handoff, and SUSv4 conformance.

### Acceptance Criteria

- [ ] AC-1: msgsnd of `(mtype=1, "hi")` then msgrcv returns the same payload.
- [ ] AC-2: `mtype=0` returns `-EINVAL`.
- [ ] AC-3: `mtype=-1` returns `-EINVAL`.
- [ ] AC-4: `msgsz = MSGMAX + 1` returns `-EINVAL`.
- [ ] AC-5: `msgp = NULL` returns `-EFAULT`.
- [ ] AC-6: Filling queue to `q_qbytes` then `IPC_NOWAIT` send returns `-EAGAIN`.
- [ ] AC-7: Blocking sender wakes when receiver drains and message is correctly enqueued.
- [ ] AC-8: RMID on queue while sender blocks returns `-EIDRM`.
- [ ] AC-9: SIGUSR1 to blocked sender returns `-EINTR`.
- [ ] AC-10: Without write-perm caller gets `-EACCES`.
- [ ] AC-11: Multi-page message (msgsz > PAGE_SIZE) round-trips intact.
- [ ] AC-12: q_lspid updated to sender tgid.

### Architecture

```rust
#[syscall(nr = 69, abi = "sysv")]
pub fn sys_msgsnd(msqid: i32, msgp: UserPtr<u8>, msgsz: usize, msgflg: i32) -> isize {
    Ipc::msgsnd(msqid, msgp, msgsz, msgflg)
}
```

`Ipc::msgsnd(msqid, uptr, msgsz, msgflg) -> isize`:
1. let ns = current_ipc_ns();
2. if msgsz > ns.msg_ctlmax { return -EINVAL; }
3. let mtype: i64 = unsafe { uptr.read_at::<i64>(0)? };       // EFAULT
4. if mtype <= 0 { return -EINVAL; }
5. let msg = Ipc::load_msg(uptr.add(size_of::<i64>()), msgsz)?; // ENOMEM/EFAULT
6. msg.mtype = mtype;
7. let msq = Ipc::msq_lock(ns, msqid)?;                       // EINVAL
8. Ipc::ipcperms(&msq.q_perm, S_IWUGO)?;                      // EACCES
9. Ipc::security_msg_queue_msgsnd(&msq, &msg)?;
10. loop {
11.   if Ipc::pipelined_send(&mut msq, &msg) { break; }       // direct handoff
12.   if msq.q_cbytes + msgsz as u64 <= msq.q_qbytes
13.      && msq.q_qnum < msq.q_qbytes {
14.     Ipc::enqueue_msg(&mut msq, msg);
15.     break;
16.   }
17.   if msgflg & IPC_NOWAIT != 0 { Ipc::free_msg(msg); return -EAGAIN; }
18.   Ipc::sleep_on_senders(&mut msq)?;                       // EIDRM/EINTR
19. }
20. msq.q_lspid = current().tgid;
21. msq.q_stime = ktime_get_real_seconds();
22. Ipc::wake_one_receiver(&mut msq);
23. return 0;

`Ipc::load_msg(uptr, n) -> Result<Box<MsgMsg>>`:
1. let mut head = MsgMsg::alloc()?;
2. let mut remaining = n;
3. let mut off = 0usize;
4. let first_chunk = min(remaining, DATALEN_MSG);
5. unsafe { uptr.add(off).copy_in_partial(&mut head.body[..first_chunk])?; }
6. remaining -= first_chunk; off += first_chunk;
7. while remaining > 0 {
8.   let seg = MsgSeg::alloc()?;
9.   let chunk = min(remaining, DATALEN_SEG);
10.  unsafe { uptr.add(off).copy_in_partial(&mut seg.body[..chunk])?; }
11.  head.append_seg(seg);
12.  remaining -= chunk; off += chunk;
13. }
14. Ok(head)

### Out of Scope

- Message receive (covered in `msgrcv.md`).
- Queue creation/administration (covered in `msgget.md`, `msgctl.md`).
- Implementation code.
- Pre-2.4 i386 ipc(2) trampoline.

### signature

```c
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
```

```c
struct msgbuf {
    long mtype;        /* > 0 */
    char mtext[];      /* msgsz bytes */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `msqid` | `int` | in | Identifier returned by `msgget(2)`. |
| `msgp`  | `const void *` | in | User pointer to `struct msgbuf` header followed by `msgsz` bytes. |
| `msgsz` | `size_t` | in | Size of `mtext`. Must be `<= MSGMAX` (default 8192 bytes; per-ns sysctl). |
| `msgflg`| `int` | in | `IPC_NOWAIT` only meaningful bit. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success; message enqueued. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EACCES` | Caller lacks write permission on queue. |
| `EAGAIN` | Queue full and `IPC_NOWAIT` set. |
| `EFAULT` | `msgp` not addressable. |
| `EIDRM` | Queue was removed (`IPC_RMID`) while waiter blocked. |
| `EINTR` | Signal caught while blocked. |
| `EINVAL` | `mtype < 1`, `msgsz > MSGMAX`, or bad msqid. |
| `ENOMEM` | Out of memory allocating `msg_msg` segments. |

### abi surface

```text
__NR_msgsnd  (x86_64)  = 69
__NR_msgsnd  (arm64)   = 189
__NR_msgsnd  (riscv)   = 189
__NR_msgsnd  (i386)    = via ipc(2) multiplexer (call 11)

Layout: copy_from_user reads sizeof(long) for mtype, then msgsz bytes for mtext.
Total user payload = sizeof(long) + msgsz; verified against access_ok.
```

### compatibility contract

REQ-1: Syscall number is **69** on x86_64. ABI-stable.

REQ-2: `mtype` is read as a `long` (8 bytes on LP64, 4 on ILP32) from `msgp`. Must be `> 0`; else `EINVAL`.

REQ-3: `msgsz <= ns->msg_ctlmax` (MSGMAX, default 8192). Larger ⟹ `EINVAL`.

REQ-4: Permission: write-permission per `ipcperms(msq, S_IWUGO)`; `CAP_IPC_OWNER` in queue's user-ns bypasses.

REQ-5: Storage: kernel allocates `struct msg_msg` (header) + chained `msg_msgseg` segments (one per additional page). Allocation uses `GFP_KERNEL_ACCOUNT` so cost is charged to the memcg.

REQ-6: Queue-full predicate: `q_cbytes + msgsz > q_qbytes` OR `q_qnum >= q_qbytes` (when message size is zero, count limit applies).

REQ-7: Blocking semantics: if queue full and `IPC_NOWAIT` clear, caller is put on `q_senders` wait list, queue lock dropped, `schedule()`; reawakened by `msgrcv` (or RMID/signal). On wake re-test predicate, retry copy.

REQ-8: `IPC_RMID` while blocked ⟹ `EIDRM`. Signal ⟹ `EINTR` (no ERESTARTSYS — POSIX requires the partial-copy invariant).

REQ-9: On success the message is appended to `q_messages` (FIFO), `q_qnum`++, `q_cbytes += msgsz`, `q_lspid = current.tgid`, `q_stime = now`.

REQ-10: Wake one reader from `q_receivers` whose `r_msgtype` matches (`pipelined_send` fast path bypasses queue insertion in that case).

REQ-11: LSM hook: `security_msg_queue_msgsnd(msq, msg)` may deny `-EACCES`.

REQ-12: User pointer copy must use `copy_from_user`; partial-page faults must roll back the allocation cleanly (no half-written message visible).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mtype_positive` | INVARIANT | mtype <= 0 ⟹ EINVAL, no allocation. |
| `msgsz_bounded` | INVARIANT | msgsz <= MSGMAX always; else EINVAL. |
| `efault_rollback` | INVARIANT | EFAULT mid-copy frees all allocated segs. |
| `eidrm_on_rmid` | INVARIANT | blocked sender + RMID ⟹ EIDRM. |
| `pipelined_send_fast` | INVARIANT | matched receiver ⟹ direct handoff, q_cbytes unchanged. |
| `q_lspid_set` | INVARIANT | success ⟹ q_lspid = current.tgid. |

### Layer 2: TLA+

`ipc/msgsnd.tla`:
- States: copy-in, perm-check, queue-full check, block/sleep, wake, enqueue, return.
- Properties:
  - `safety_no_partial_enqueue` — EFAULT/EIDRM/EINTR ⟹ q_qnum unchanged.
  - `safety_q_cbytes_bound` — invariant: q_cbytes <= q_qbytes.
  - `safety_pipelined_handoff_atomic` — direct send to receiver does not change cbytes.
  - `liveness_unblock_on_drain` — drain or RMID eventually unblocks senders.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `load_msg` post: total bytes copied = msgsz | `Ipc::load_msg` |
| `enqueue_msg` post: q_qnum += 1, q_cbytes += msg.size | `Ipc::enqueue_msg` |
| `msgsnd` post: returns 0 ⟹ message reachable from q_messages | `Ipc::msgsnd` |

### Layer 4: Verus / Creusot functional

Per-`msgsnd(2)` man-page; LTP `ipc/msgsnd*`; SUSv4 conformance.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`msgsnd(2)` reinforcement:

- **Per-MSGMAX upper bound enforced before allocation** — defense against per-allocation-DoS.
- **Per-queue MSGMNB byte budget** — defense against per-queue memory blow-up.
- **Per-memcg accounting via GFP_KERNEL_ACCOUNT** — defense against per-cgroup escape.
- **Per-mtype positivity check** — defense against per-receiver type-confusion (mtype 0 reserved as "any").
- **Per-EFAULT rollback frees segs** — defense against per-partial-message leak.

### grsecurity / pax surface

- **PaX UDEREF on copy_from_user(msgp)** — defense against per-msgp kernel-deref bug; SMAP forced. Each msg_msgseg copy uses access_ok + uaccess primitives only.
- **PAX_USERCOPY_HARDEN whitelist on msg_msg / msg_msgseg slabs** — these slabs are NOT user-copy whitelisted; any future copy_to_user against them is trapped (only msgrcv may release contents, and it explicitly stages through a stack buffer).
- **GRKERNSEC_HARDEN_IPC** — write permission requires euid == cuid OR explicit `CAP_IPC_OWNER` in queue's user-ns; mode bits alone do not grant cross-uid write.
- **MSGMNB (per-queue) and MSGMAX (per-message) enforced under namespace** — sysctls are per-ipc-namespace and cannot be raised above init-ns ceiling from a child userns.
- **PAX_REFCOUNT on q_perm refcount + q_senders list** — defense against per-refcount-overflow UAF during RMID race.
- **GRKERNSEC_PROC_IPC info-leak suppression** — `q_lspid` (last sender pid) exposed only to root or queue cuid via /proc/sysvipc/msg; q_stime rounded to seconds (already), and finer-grain timing leaks suppressed.
- **CAP_IPC_OWNER strict in queue's user-ns** — defense against cross-userns write-bypass.
- **IPC namespace strict** — message body, q_senders, q_receivers all per-ns; no cross-ns wake or peek.
- **SIGNAL_INTERRUPT semantics POSIX-strict** — no ERESTARTSYS path; defense against per-restart-replay (would re-copy_from_user, potentially reading attacker-modified payload).
- **GRKERNSEC_RANDSTRUCT on msg_msg layout** — slab layout randomization, defeating offset-based heap-spray exploits.
- **Per-pipelined_send still validates receiver permission** — defense against per-fast-path policy-bypass.
- **PaX KERNEXEC** — msg payload pages are NX; defense against any per-message-buffer code-execution attempt.

