---
title: "Tier-5 syscall: msgrcv(2) — syscall 70"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`msgrcv(2)` removes a message from the System V message queue `msqid` and copies it to a user buffer `msgp` of capacity `msgsz` bytes (mtext only). Selection is driven by `msgtyp`: `0` = first message; `> 0` = first message with `mtype == msgtyp`; `< 0` = lowest-`mtype` message with `mtype <= |msgtyp|`. Flags include `MSG_NOERROR` (truncate to `msgsz` instead of `E2BIG`), `MSG_EXCEPT` (with positive `msgtyp`, take first message with `mtype != msgtyp`), `MSG_COPY` (peek by sequence, used with `IPC_NOWAIT`, leaves message in queue), and `IPC_NOWAIT`.

Critical for: System V IPC interop, priority demultiplexing via mtype, and SUSv4 conformance.

### Acceptance Criteria

- [ ] AC-1: Empty queue + IPC_NOWAIT ⟹ `-ENOMSG`.
- [ ] AC-2: `msgtyp = 0` returns oldest message regardless of mtype.
- [ ] AC-3: `msgtyp = 5`: returns first message with mtype==5.
- [ ] AC-4: `msgtyp = -5`: returns lowest mtype among messages with mtype<=5.
- [ ] AC-5: `MSG_EXCEPT` with `msgtyp=5`: returns first mtype != 5.
- [ ] AC-6: Oversized message + `!MSG_NOERROR` ⟹ `-E2BIG`, message stays.
- [ ] AC-7: Oversized message + `MSG_NOERROR` ⟹ truncate, return msgsz.
- [ ] AC-8: RMID on queue while blocked ⟹ `-EIDRM`.
- [ ] AC-9: SIGUSR1 on blocked receiver ⟹ `-EINTR`.
- [ ] AC-10: `MSG_COPY` peeks at index, leaves count unchanged.
- [ ] AC-11: q_lrpid updated to receiver tgid.
- [ ] AC-12: msgp NULL ⟹ `-EFAULT` (message remains).

### Architecture

```rust
#[syscall(nr = 70, abi = "sysv")]
pub fn sys_msgrcv(
    msqid: i32, msgp: UserPtr<u8>, msgsz: usize,
    msgtyp: i64, msgflg: i32,
) -> isize {
    Ipc::msgrcv(msqid, msgp, msgsz, msgtyp, msgflg)
}
```

`Ipc::msgrcv(msqid, uptr, msgsz, msgtyp, msgflg) -> isize`:
1. let ns = current_ipc_ns();
2. if msgsz > MSGMAX_BUFFER { return -EINVAL; }
3. if (msgflg & MSG_COPY) != 0 {
4.   if (msgflg & MSG_EXCEPT) != 0 { return -EINVAL; }
5.   if !capable(CAP_CHECKPOINT_RESTORE) { return -EINVAL; }
6.   if (msgflg & IPC_NOWAIT) == 0 { return -EINVAL; }
7. }
8. let mut msq = Ipc::msq_lock(ns, msqid)?;                   // EINVAL
9. Ipc::ipcperms(&msq.q_perm, S_IRUGO)?;                      // EACCES
10. loop {
11.   let pick = Ipc::find_msg(&msq, msgtyp, msgflg)?;
12.   match pick {
13.     Some(msg) => {
14.       Ipc::security_msg_queue_msgrcv(&msq, &msg, current(), msgtyp, msgflg)?;
15.       if msg.size > msgsz && (msgflg & MSG_NOERROR) == 0 { return -E2BIG; }
16.       if (msgflg & MSG_COPY) == 0 { Ipc::dequeue_msg(&mut msq, &msg); }
17.       let n = min(msg.size, msgsz);
18.       Ipc::store_msg(uptr, &msg, n)?;                      // EFAULT
19.       if (msgflg & MSG_COPY) == 0 { Ipc::free_msg(msg); Ipc::wake_one_sender(&mut msq); }
20.       msq.q_lrpid = current().tgid;
21.       msq.q_rtime = ktime_get_real_seconds();
22.       return n as isize;
23.     }
24.     None => {
25.       if (msgflg & IPC_NOWAIT) != 0 { return -ENOMSG; }
26.       Ipc::sleep_on_receivers(&mut msq, msgtyp, msgflg)?;  // EIDRM/EINTR
27.     }
28.   }
29. }

`Ipc::find_msg(msq, msgtyp, flg) -> Option<&MsgMsg>`:
1. for m in msq.q_messages.iter() {
2.   if msgtyp == 0 { return Some(m); }
3.   if msgtyp > 0 {
4.     if (flg & MSG_EXCEPT) == 0 && m.mtype == msgtyp { return Some(m); }
5.     if (flg & MSG_EXCEPT) != 0 && m.mtype != msgtyp { return Some(m); }
6.   } else {
7.     /* msgtyp < 0 */
8.     /* track minimum so far */
9.   }
10. }
11. /* For msgtyp < 0: return tracked lowest-mtype <= |msgtyp| or None. */

### Out of Scope

- Sending (covered in `msgsnd.md`).
- Queue admin (covered in `msgctl.md`).
- Implementation code.
- Pre-2.4 i386 ipc(2) trampoline.

### signature

```c
ssize_t msgrcv(int msqid, void *msgp, size_t msgsz,
               long msgtyp, int msgflg);
```

```c
#define MSG_NOERROR  010000
#define MSG_EXCEPT   020000
#define MSG_COPY     040000
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `msqid`  | `int` | in | Queue identifier from `msgget(2)`. |
| `msgp`   | `void *` | out | User buffer; receives `mtype` (long) followed by up to `msgsz` payload bytes. |
| `msgsz`  | `size_t` | in | Max `mtext` bytes the buffer can hold. |
| `msgtyp` | `long` | in | Selection criterion (see Summary). |
| `msgflg` | `int` | in | `IPC_NOWAIT` | `MSG_NOERROR` | `MSG_EXCEPT` | `MSG_COPY`. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of `mtext` bytes copied (does not include `mtype`). |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `E2BIG` | Message bigger than `msgsz` and `MSG_NOERROR` not set. |
| `EACCES` | Caller lacks read permission. |
| `EAGAIN` | `IPC_NOWAIT` and no matching message. |
| `EFAULT` | `msgp` not writable. |
| `EIDRM` | Queue removed while blocked. |
| `EINTR` | Signal caught while blocked. |
| `EINVAL` | Bad `msqid`, `msgsz < 0`, conflicting `MSG_COPY` + `MSG_EXCEPT`, MSG_COPY without `CHECKPOINT_RESTORE`. |
| `ENOMSG` | `IPC_NOWAIT` and no matching message (when EAGAIN not applicable). |

### abi surface

```text
__NR_msgrcv  (x86_64)  = 70
__NR_msgrcv  (arm64)   = 188
__NR_msgrcv  (riscv)   = 188
__NR_msgrcv  (i386)    = via ipc(2) multiplexer (call 12)

Layout written to msgp: [long mtype][min(msg.size, msgsz) bytes of mtext].
MSG_COPY requires CONFIG_CHECKPOINT_RESTORE and CAP_SYS_ADMIN (or root).
```

### compatibility contract

REQ-1: Syscall number is **70** on x86_64. ABI-stable.

REQ-2: Selection rule:
- `msgtyp == 0`: take first (oldest) message regardless of mtype.
- `msgtyp > 0` and `!(msgflg & MSG_EXCEPT)`: first message with `mtype == msgtyp`.
- `msgtyp > 0` and `(msgflg & MSG_EXCEPT)`: first message with `mtype != msgtyp`.
- `msgtyp < 0`: of messages with `mtype <= |msgtyp|`, take lowest-`mtype` (oldest if tied).

REQ-3: `MSG_COPY` (with `IPC_NOWAIT`): treat `msgtyp` as a 0-based sequence index into the queue; copy without removal. Used by CRIU. Requires `CHECKPOINT_RESTORE` capability and `IPC_NOWAIT`. Conflicts with `MSG_EXCEPT` ⟹ `EINVAL`.

REQ-4: If message's `mtext` size `> msgsz`:
- `MSG_NOERROR` set: truncate to `msgsz`, return `msgsz`.
- Otherwise: do not remove, return `-E2BIG`.

REQ-5: Read permission: `ipcperms(msq, S_IRUGO)`; `CAP_IPC_OWNER` in user-ns bypasses.

REQ-6: Blocking semantics: no matching message + `IPC_NOWAIT` ⟹ `-ENOMSG` (note: not `-EAGAIN`). No matching message + blocking: register on `q_receivers` with `r_msgtype` set, drop lock, `schedule()`. Pipelined-send by a writer may directly hand off a message and wake us.

REQ-7: `IPC_RMID` while blocked ⟹ `-EIDRM`. Signal ⟹ `-EINTR`.

REQ-8: On success: `q_lrpid = current.tgid`, `q_rtime = now`, `q_qnum--`, `q_cbytes -= msg.size`. Wake one blocked sender if any.

REQ-9: `copy_to_user` writes mtype first (sizeof(long) bytes), then `min(msg.size, msgsz)` bytes of mtext. EFAULT mid-copy: message is lost (removed from queue) — caller is expected to validate buffer before the call.

REQ-10: LSM hook: `security_msg_queue_msgrcv(msq, msg, target, msgtyp, mode)` may deny.

REQ-11: `msgsz` is `size_t` (unsigned); but negative cast from int returns -EINVAL after upper-bound check (`msgsz > MSGMAX_BUFFER`).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `select_obeys_msgtyp` | INVARIANT | selection matches msgtyp sign + MSG_EXCEPT bit. |
| `e2big_no_remove` | INVARIANT | E2BIG path leaves msg in queue. |
| `efault_after_dequeue_lost` | INVARIANT | EFAULT post-dequeue marks message as freed (no leak). |
| `msg_copy_no_remove` | INVARIANT | MSG_COPY does not decrement q_qnum. |
| `cap_checkpoint_restore_for_copy` | INVARIANT | MSG_COPY without CAP_CHECKPOINT_RESTORE ⟹ EINVAL. |
| `q_lrpid_set` | INVARIANT | success ⟹ q_lrpid = current.tgid. |

### Layer 2: TLA+

`ipc/msgrcv.tla`:
- States: select, perm-check, copy-out, block, wake, return.
- Properties:
  - `safety_select_correct_for_msgtyp_sign` — selection rule per REQ-2.
  - `safety_msg_copy_immutable` — MSG_COPY peek is read-only.
  - `safety_no_double_free` — message freed exactly once on remove path.
  - `liveness_unblock_on_send` — pending matching send eventually wakes receiver.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `find_msg` post: returned msg satisfies selection rule | `Ipc::find_msg` |
| `dequeue_msg` post: q_qnum -= 1, q_cbytes -= msg.size | `Ipc::dequeue_msg` |
| `msgrcv` post: success n = min(msg.size, msgsz); msg consumed unless MSG_COPY | `Ipc::msgrcv` |

### Layer 4: Verus / Creusot functional

Per-`msgrcv(2)` man-page; LTP `ipc/msgrcv*`; SUSv4 conformance.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`msgrcv(2)` reinforcement:

- **Per-mtype selection deterministic** — defense against per-priority-bypass.
- **Per-E2BIG leaves message intact** — defense against per-truncation-information-loss.
- **Per-CAP_CHECKPOINT_RESTORE for MSG_COPY** — defense against per-criu-misuse.
- **Per-wake-one-sender on dequeue** — defense against per-thundering-herd.

### grsecurity / pax surface

- **PaX UDEREF on copy_to_user(msgp)** — defense against per-msgp kernel-deref bug. SMAP forced.
- **PAX_USERCOPY_HARDEN on store_msg path** — defense against per-overcopy past msgsz; bounded length is the only acceptable copy.
- **GRKERNSEC_HARDEN_IPC** — read permission requires euid == cuid OR `CAP_IPC_OWNER` in queue's user-ns; world-readable mode bits do not grant cross-uid read.
- **GRKERNSEC_PROC_IPC info-leak suppression** — q_lrpid (last receiver pid) exposed only to root or queue cuid; defense against per-IPC pid-discovery side channel.
- **MSG_COPY restricted to init userns** — even with `CAP_CHECKPOINT_RESTORE`, MSG_COPY is rejected from non-init user namespaces (grsec: a child userns cannot snapshot parent-ns IPC).
- **IPC namespace strict** — receivers see only their ns; defense against per-cross-ns peek.
- **PAX_REFCOUNT on msg_msg + q_perm refcounts** — defense against per-RMID race UAF.
- **GRKERNSEC_RANDSTRUCT on msg_msg layout** — defense against per-heap-spray exploit.
- **SIGNAL_INTERRUPT POSIX-strict** — no ERESTARTSYS; defense against per-restart-double-receive.
- **CAP_IPC_OWNER in queue's user-ns strict** — defense against cross-userns bypass.
- **GRKERNSEC_HIDESYM on msg payload pages** — payload pages are anon kernel pages, never aliased to userland outside of explicit copy_to_user.
- **MSG_NOERROR-truncation audited** — when truncation occurs, an audit record is emitted (info-leak surface monitored).

