---
title: "Tier-3: ipc/msg.c — System V message queues"
tags: ["tier-3", "ipc", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

System V **message queues** are kernel-resident FIFO byte-stream queues addressable by `key_t` and `msqid`. Per-queue: `struct msg_queue` (q_perm, q_qnum, q_cbytes, q_qbytes, list of `struct msg_msg` linked via `m_list`, sleeping-sender list, sleeping-receiver list). Per-syscall: `msgget(key, msgflg)` creates / opens; `msgsnd(msqid, msgp, msgsz, msgflg)` appends (blocks if `q_cbytes + msgsz > q_qbytes` ∧ !IPC_NOWAIT); `msgrcv(msqid, msgp, msgsz, msgtyp, msgflg)` removes matching message (blocks if no match ∧ !IPC_NOWAIT); `msgctl(msqid, cmd, buf)` IPC_STAT / IPC_SET / IPC_RMID / IPC_INFO / MSG_INFO / MSG_STAT(_ANY). Per-queue `q_qbytes` defaults to `ns->msg_ctlmnb` (MSGMNB=16384) ; per-message limit `ns->msg_ctlmax` (MSGMAX=8192) ; per-namespace `ns->msg_ctlmni` (MSGMNI=32000). Per-msgrcv with `MSG_NOERROR` truncates oversize, without it returns -E2BIG. Per-`pipelined_send` directly hands message to a matching sleeping receiver without queue insertion. Per-ipc_namespace `ns->percpu_msg_bytes` + `ns->percpu_msg_hdrs` track global bytes/headers. Critical for: legacy IPC ABI, SUS msgsnd/msgrcv semantics, namespace isolation.

This Tier-3 covers `ipc/msg.c` (~1376 lines).

### Acceptance Criteria

- [ ] AC-1: msgget(IPC_PRIVATE, 0666) creates new queue with q_qbytes = MSGMNB.
- [ ] AC-2: msgget(K, IPC_CREAT|IPC_EXCL, 0666) twice → second returns -EEXIST.
- [ ] AC-3: msgsnd of msgsz > ns->msg_ctlmax → -EINVAL.
- [ ] AC-4: msgsnd of mtype < 1 → -EINVAL.
- [ ] AC-5: msgsnd to full queue with IPC_NOWAIT → -EAGAIN.
- [ ] AC-6: msgsnd to full queue without IPC_NOWAIT blocks until space; resumes after msgrcv.
- [ ] AC-7: msgrcv with empty queue + IPC_NOWAIT → -ENOMSG.
- [ ] AC-8: msgrcv with bufsz < msg.m_ts + no MSG_NOERROR → -E2BIG, message left in queue.
- [ ] AC-9: msgrcv with bufsz < msg.m_ts + MSG_NOERROR → truncated message returned, removed from queue.
- [ ] AC-10: msgrcv with msgtyp > 0 → first message of matching type.
- [ ] AC-11: msgrcv with msgtyp < 0 → first message with m_type ≤ |msgtyp|.
- [ ] AC-12: msgrcv with MSG_EXCEPT + msgtyp > 0 → first message NOT of that type.
- [ ] AC-13: msgctl(IPC_RMID) wakes blocked senders and receivers with -EIDRM.
- [ ] AC-14: msgctl(IPC_SET) raising q_qbytes above ns->msg_ctlmnb requires CAP_SYS_RESOURCE.
- [ ] AC-15: pipelined_send delivers directly to a matching sleeper without enqueue.
- [ ] AC-16: ipc_namespace isolation: queue created in ns A invisible in ns B.

### Architecture

```
struct MsgQueue {
  q_perm: KernIpcPerm,
  q_stime: i64,
  q_rtime: i64,
  q_ctime: i64,
  q_cbytes: u64,                      // current bytes on queue
  q_qnum: u64,                        // current message count
  q_qbytes: u64,                      // max bytes (ns->msg_ctlmnb at create)
  q_lspid: Option<*Pid>,              // last sender pid
  q_lrpid: Option<*Pid>,              // last receiver pid
  q_messages: ListHead<MsgMsg>,
  q_receivers: ListHead<MsgReceiver>,
  q_senders: ListHead<MsgSender>,
}

struct MsgMsg {
  m_list: ListLink,
  m_type: i64,
  m_ts: usize,
  next: Option<*MsgMsgseg>,
  security: Option<*LsmBlob>,
}

struct MsgReceiver {
  r_list: ListLink,
  r_tsk: *TaskStruct,
  r_mode: u8,                         // SEARCH_*
  r_msgtype: i64,
  r_maxsize: i64,
  r_msg: AtomicPtr<MsgMsg>,           // -EAGAIN / msg / -E2BIG / -EIDRM
}

struct MsgSender {
  list: ListLink,
  tsk: *TaskStruct,
  msgsz: usize,
}
```

`Msg::sys_msgget(key, msgflg) -> Result<i32>`:
1. ns = current.nsproxy.ipc_ns.
2. msg_ops = {.getnew = Msg::newque, .associate = security_msg_queue_associate}.
3. return ipcget(ns, &msg_ids(ns), &msg_ops, ¶ms{key, msgflg}).

`Msg::newque(ns, params) -> Result<i32>`:
1. msq = kmalloc_obj(GFP_KERNEL_ACCOUNT).
2. msq.q_perm.mode = msgflg & 0o777; msq.q_perm.key = key.
3. security_msg_queue_alloc(&msq.q_perm).
4. msq.q_stime = q_rtime = 0; q_ctime = real_seconds.
5. msq.q_qbytes = ns.msg_ctlmnb.
6. INIT_LIST_HEAD on q_messages / q_receivers / q_senders.
7. ipc_addid(&msg_ids(ns), &msq.q_perm, ns.msg_ctlmni).
8. ipc_unlock_object; rcu_read_unlock.
9. return msq.q_perm.id.

`Msg::do_msgsnd(msqid, mtype, mtext, msgsz, msgflg) -> Result<()>`:
1. Bounds: msgsz ≤ ns.msg_ctlmax, mtype ≥ 1, msqid ≥ 0.
2. msg = load_msg(mtext, msgsz).
3. msg.m_type = mtype; msg.m_ts = msgsz.
4. rcu_read_lock; msq = msq_obtain_object_check(ns, msqid).
5. ipc_lock_object.
6. Loop:
   - ipcperms(S_IWUGO); ipc_valid_object; security_msg_queue_msgsnd.
   - if msg_fits_inqueue(msq, msgsz): break.
   - if msgflg & IPC_NOWAIT: return -EAGAIN.
   - ss_add(msq, &s, msgsz); ipc_rcu_getref; unlock + schedule.
   - re-lock; ipc_rcu_putref; check valid_object; ss_del.
   - if signal_pending: return -ERESTARTNOHAND.
7. ipc_update_pid(&msq.q_lspid, task_tgid(current)).
8. msq.q_stime = real_seconds.
9. if !Msg::pipelined_send(msq, msg, wake_q):
   - list_add_tail(&msg.m_list, &msq.q_messages).
   - msq.q_cbytes += msgsz; msq.q_qnum += 1.
   - percpu_counter_add_local(...).
10. ipc_unlock_object; wake_up_q; rcu_read_unlock.

`Msg::do_msgrcv(msqid, buf, bufsz, msgtyp, msgflg, msg_handler) -> Result<isize>`:
1. Bounds + MSG_COPY constraints (MSG_EXCEPT forbidden, IPC_NOWAIT required).
2. mode = Msg::convert_mode(&msgtyp, msgflg).
3. rcu_read_lock; msq = msq_obtain_object_check.
4. Loop:
   - ipcperms(S_IRUGO); ipc_lock_object; valid_object check.
   - msg = Msg::find_msg(msq, &msgtyp, mode).
   - if !IS_ERR(msg):
     - if bufsz < msg.m_ts ∧ !(msgflg & MSG_NOERROR): -E2BIG.
     - if msgflg & MSG_COPY: copy_msg(msg, copy); leave-in-queue.
     - else: list_del + dec counters + ss_wakeup.
     - break.
   - if msgflg & IPC_NOWAIT: -ENOMSG; break.
   - enqueue msr_d on q_receivers; WRITE_ONCE(r_msg, -EAGAIN); TASK_INTERRUPTIBLE.
   - schedule; lockless READ_ONCE(r_msg); smp_acquire__after_ctrl_dep.
   - re-lock retry; if signal_pending: -ERESTARTNOHAND.
5. unlock + wake_up_q + rcu_read_unlock.
6. if !IS_ERR(msg): bufsz = msg_handler(buf, msg, bufsz); free_msg(msg).

`Msg::pipelined_send(msq, msg, wake_q) -> i32`:
1. For each msr in msq.q_receivers:
   - if testmsg(msg, msr.r_msgtype, msr.r_mode) ∧ !security_msg_queue_msgrcv(...):
     - list_del.
     - if msr.r_maxsize < msg.m_ts:
       - wake_q_add; smp_store_release(&msr.r_msg, ERR_PTR(-E2BIG)).
     - else:
       - update lrpid + rtime; wake_q_add; smp_store_release(&msr.r_msg, msg).
       - return 1.
2. return 0.

`Msg::freeque(ns, ipcp)`:
1. expunge_all(msq, -EIDRM, wake_q). /* receivers fail with -EIDRM */
2. ss_wakeup(msq, wake_q, true). /* kill flag clears list.next on senders */
3. msg_rmid(ns, msq).
4. ipc_unlock_object; wake_up_q; rcu_read_unlock.
5. For each msg in q_messages: percpu_counter_sub; free_msg.
6. percpu_counter_sub_local(&ns.percpu_msg_bytes, q_cbytes).
7. ipc_update_pid(NULL) on lspid + lrpid.
8. ipc_rcu_putref(&msq.q_perm, msg_rcu_free). /* deferred via RCU */

`Msg::init_ns(ns) -> Result<()>`:
1. ns.msg_ctlmax = MSGMAX; ns.msg_ctlmnb = MSGMNB; ns.msg_ctlmni = MSGMNI.
2. percpu_counter_init(&ns.percpu_msg_bytes).
3. percpu_counter_init(&ns.percpu_msg_hdrs).
4. ipc_init_ids(&ns.ids[IPC_MSG_IDS]).

### Out of Scope

- `ipc/msgutil.c` payload allocation (load_msg / store_msg / free_msg / copy_msg overflow-chain) — covered separately.
- `ipc/util.c` ipc_addid / ipcget / ipcctl_obtain_check / ipcperms — covered in `ipc/util.md` Tier-3.
- `ipc/namespace.c` ipc_namespace lifecycle — covered in `ipc/namespace.md`.
- POSIX message queues (`ipc/mqueue.c`) — separate Tier-3.
- LSM security_msg_queue_* implementations (`security/selinux/`, `security/smack/`).
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct msg_queue` | per-queue object | `MsgQueue` |
| `struct msg_msg` | per-message header (defined in `ipc/msgutil.c`) | `MsgMsg` |
| `struct msg_receiver` | per-sleeping receiver | `MsgReceiver` |
| `struct msg_sender` | per-sleeping sender | `MsgSender` |
| `ksys_msgget()` / `SYSCALL_DEFINE2(msgget, ...)` | per-syscall create/open | `Msg::sys_msgget` |
| `ksys_msgsnd()` / `SYSCALL_DEFINE4(msgsnd, ...)` | per-syscall send | `Msg::sys_msgsnd` |
| `ksys_msgrcv()` / `SYSCALL_DEFINE5(msgrcv, ...)` | per-syscall receive | `Msg::sys_msgrcv` |
| `ksys_msgctl()` / `SYSCALL_DEFINE3(msgctl, ...)` | per-syscall control | `Msg::sys_msgctl` |
| `newque()` | per-`newq` constructor (ipc_ops.getnew) | `Msg::newque` |
| `freeque()` | per-RMID destructor (wakes waiters) | `Msg::freeque` |
| `do_msgsnd()` | per-send core | `Msg::do_msgsnd` |
| `do_msgrcv()` | per-receive core | `Msg::do_msgrcv` |
| `pipelined_send()` | per-direct sender-to-receiver handoff | `Msg::pipelined_send` |
| `expunge_all()` | per-RMID/IPC_SET wake-with-error | `Msg::expunge_all` |
| `ss_add()` / `ss_del()` / `ss_wakeup()` | per-sleeping-sender queue mgmt | `Msg::ss_add` / `ss_del` / `ss_wakeup` |
| `find_msg()` | per-msgtyp scan | `Msg::find_msg` |
| `testmsg()` | per-message match predicate | `Msg::testmsg` |
| `convert_mode()` | per-msgtyp → SEARCH_* | `Msg::convert_mode` |
| `msg_fits_inqueue()` | per-fit check | `Msg::fits_inqueue` |
| `msgctl_down()` | per-IPC_RMID/IPC_SET | `Msg::ctl_down` |
| `msgctl_info()` / `msgctl_stat()` | per-IPC_INFO/IPC_STAT | `Msg::ctl_info` / `ctl_stat` |
| `msg_init_ns()` / `msg_exit_ns()` | per-namespace lifecycle | `Msg::init_ns` / `exit_ns` |
| `load_msg()` / `store_msg()` / `free_msg()` / `copy_msg()` | per-msg-payload helpers (msgutil) | `MsgUtil::*` |

### compatibility contract

REQ-1: struct msg_queue:
- q_perm: per-`struct kern_ipc_perm` (key, uid, gid, mode, id, seq, security, rcu).
- q_stime / q_rtime / q_ctime: per-last msgsnd / msgrcv / change time.
- q_cbytes: per-current bytes on queue.
- q_qnum: per-current message count.
- q_qbytes: per-max bytes (default `ns->msg_ctlmnb` = MSGMNB).
- q_lspid / q_lrpid: per-`struct pid` of last sender / receiver.
- q_messages: per-list of `struct msg_msg` (FIFO order, linked via `m_list`).
- q_receivers: per-list of sleeping `struct msg_receiver`.
- q_senders: per-list of sleeping `struct msg_sender`.
- __randomize_layout: per-RANDSTRUCT layout obfuscation.

REQ-2: struct msg_msg (from `include/linux/msg.h`):
- m_list: per-list-link in q_messages.
- m_type: per-message type (long; ≥ 1 enforced by do_msgsnd).
- m_ts: per-message text size (in bytes).
- next: per-overflow segment chain (msg payload > DATALEN_MSG).
- security: per-LSM blob.

REQ-3: struct msg_receiver:
- r_list: per-list-link in q_receivers.
- r_tsk: per-sleeping task_struct.
- r_mode: per-SEARCH_ANY / _EQUAL / _NOTEQUAL / _LESSEQUAL / _NUMBER.
- r_msgtype: per-msgtyp passed by receiver.
- r_maxsize: per-buf-size (INT_MAX if MSG_NOERROR).
- r_msg: per-result (smp_store_release'd by pipelined_send / expunge_all; READ_ONCE'd lockless).

REQ-4: struct msg_sender:
- list: per-list-link in q_senders.
- tsk: per-sleeping task.
- msgsz: per-message size requested.

REQ-5: msgget(key, msgflg) (ksys_msgget):
- /* Per-ipc_namespace lookup via ipcget */
- ns = current->nsproxy->ipc_ns.
- msg_ops = {.getnew = newque, .associate = security_msg_queue_associate}.
- params = {.key = key, .flg = msgflg}.
- return ipcget(ns, &msg_ids(ns), &msg_ops, &params).
- /* Per-IPC_PRIVATE always new */
- /* Per-IPC_CREAT|IPC_EXCL create-or-EEXIST */
- /* Per-IPC_CREAT create-or-existing */

REQ-6: newque(ns, params):
- msq = kmalloc_obj(*msq, GFP_KERNEL_ACCOUNT).
- msq.q_perm.mode = msgflg & S_IRWXUGO.
- msq.q_perm.key = key.
- security_msg_queue_alloc(&msq.q_perm).
- msq.q_stime = q_rtime = 0; q_ctime = ktime_get_real_seconds().
- msq.q_cbytes = q_qnum = 0.
- msq.q_qbytes = ns->msg_ctlmnb.
- INIT_LIST_HEAD on q_messages / q_receivers / q_senders.
- ipc_addid(&msg_ids(ns), &msq.q_perm, ns->msg_ctlmni). /* may return -ENOSPC */
- ipc_unlock_object; rcu_read_unlock.
- return msq.q_perm.id.

REQ-7: do_msgsnd(msqid, mtype, mtext, msgsz, msgflg):
- /* Bounds */
- if msgsz > ns->msg_ctlmax ∨ (long)msgsz < 0 ∨ msqid < 0: return -EINVAL.
- if mtype < 1: return -EINVAL.
- /* Allocate */
- msg = load_msg(mtext, msgsz); /* copies user buf into msg_msg with possible overflow chain */
- if IS_ERR(msg): return PTR_ERR(msg).
- msg.m_type = mtype; msg.m_ts = msgsz.
- /* Lookup */
- rcu_read_lock; msq = msq_obtain_object_check(ns, msqid).
- ipc_lock_object(&msq.q_perm).
- /* Retry loop */
- for (;;):
  - if ipcperms(ns, &msq.q_perm, S_IWUGO): err = -EACCES; goto out.
  - if !ipc_valid_object(&msq.q_perm): err = -EIDRM; goto out.
  - security_msg_queue_msgsnd(&msq.q_perm, msg, msgflg).
  - if msg_fits_inqueue(msq, msgsz): break.
  - if msgflg & IPC_NOWAIT: err = -EAGAIN; goto out.
  - ss_add(msq, &s, msgsz); /* enqueue self; __set_current_state(TASK_INTERRUPTIBLE) */
  - ipc_rcu_getref(&msq.q_perm).
  - ipc_unlock_object; rcu_read_unlock; schedule().
  - rcu_read_lock; ipc_lock_object; ipc_rcu_putref.
  - if !ipc_valid_object: err = -EIDRM; goto out.
  - ss_del(&s).
  - if signal_pending(current): err = -ERESTARTNOHAND; goto out.
- /* Deliver */
- ipc_update_pid(&msq.q_lspid, task_tgid(current)).
- msq.q_stime = ktime_get_real_seconds.
- if !pipelined_send(msq, msg, &wake_q):
  - list_add_tail(&msg.m_list, &msq.q_messages).
  - msq.q_cbytes += msgsz; msq.q_qnum++.
  - percpu_counter_add_local(&ns->percpu_msg_bytes, msgsz).
  - percpu_counter_add_local(&ns->percpu_msg_hdrs, 1).
- err = 0; msg = NULL.
- ipc_unlock_object; wake_up_q(&wake_q); rcu_read_unlock.
- if msg: free_msg(msg).
- return err.

REQ-8: pipelined_send(msq, msg, wake_q):
- /* Hand msg directly to first compatible sleeping receiver */
- list_for_each_entry_safe(msr, t, &msq.q_receivers, r_list):
  - if testmsg(msg, msr.r_msgtype, msr.r_mode) ∧ !security_msg_queue_msgrcv(...):
    - list_del(&msr.r_list).
    - if msr.r_maxsize < msg.m_ts:
      - wake_q_add(wake_q, msr.r_tsk).
      - smp_store_release(&msr.r_msg, ERR_PTR(-E2BIG)).
    - else:
      - ipc_update_pid(&msq.q_lrpid, task_pid(msr.r_tsk)).
      - msq.q_rtime = ktime_get_real_seconds.
      - wake_q_add(wake_q, msr.r_tsk).
      - smp_store_release(&msr.r_msg, msg).
      - return 1.
- return 0.

REQ-9: do_msgrcv(msqid, buf, bufsz, msgtyp, msgflg, msg_handler):
- /* Bounds */
- if msqid < 0 ∨ (long)bufsz < 0: return -EINVAL.
- if msgflg & MSG_COPY:
  - if (msgflg & MSG_EXCEPT) ∨ !(msgflg & IPC_NOWAIT): return -EINVAL.
  - copy = prepare_copy(buf, min(bufsz, ns->msg_ctlmax)). /* CHECKPOINT_RESTORE */
- mode = convert_mode(&msgtyp, msgflg).
- /* Lookup + lock + retry */
- rcu_read_lock; msq = msq_obtain_object_check(ns, msqid).
- for (;;):
  - if ipcperms(ns, &msq.q_perm, S_IRUGO): msg = ERR_PTR(-EACCES); goto out.
  - ipc_lock_object(&msq.q_perm).
  - if !ipc_valid_object: msg = ERR_PTR(-EIDRM); goto out.
  - msg = find_msg(msq, &msgtyp, mode).
  - if !IS_ERR(msg):
    - /* Found */
    - if bufsz < msg.m_ts ∧ !(msgflg & MSG_NOERROR): msg = ERR_PTR(-E2BIG); goto out.
    - if msgflg & MSG_COPY: msg = copy_msg(msg, copy); goto out. /* leave in queue */
    - list_del(&msg.m_list); msq.q_qnum--.
    - msq.q_rtime = ktime_get_real_seconds.
    - ipc_update_pid(&msq.q_lrpid, task_tgid(current)).
    - msq.q_cbytes -= msg.m_ts.
    - percpu_counter_sub_local(...); percpu_counter_sub_local(...).
    - ss_wakeup(msq, &wake_q, false). /* wake potential senders */
    - goto out.
  - /* No match */
  - if msgflg & IPC_NOWAIT: msg = ERR_PTR(-ENOMSG); goto out.
  - /* Block */
  - list_add_tail(&msr_d.r_list, &msq.q_receivers).
  - msr_d = {.r_tsk = current, .r_msgtype = msgtyp, .r_mode = mode, .r_maxsize = (msgflg & MSG_NOERROR ? INT_MAX : bufsz)}.
  - WRITE_ONCE(msr_d.r_msg, ERR_PTR(-EAGAIN)).
  - __set_current_state(TASK_INTERRUPTIBLE).
  - ipc_unlock_object; rcu_read_unlock; schedule().
  - /* Lockless wake check */
  - rcu_read_lock; msg = READ_ONCE(msr_d.r_msg).
  - if msg != ERR_PTR(-EAGAIN): smp_acquire__after_ctrl_dep; goto out.
  - ipc_lock_object; msg = READ_ONCE(msr_d.r_msg).
  - if msg != ERR_PTR(-EAGAIN): goto out.
  - list_del(&msr_d.r_list).
  - if signal_pending(current): msg = ERR_PTR(-ERESTARTNOHAND); goto out.
- /* Copy out */
- ipc_unlock_object; wake_up_q; rcu_read_unlock.
- if IS_ERR(msg): free_copy(copy); return PTR_ERR(msg).
- bufsz = msg_handler(buf, msg, bufsz). /* do_msg_fill: put_user(m_type) + store_msg(mtext) */
- free_msg(msg).
- return bufsz.

REQ-10: convert_mode(msgtyp, msgflg):
- if msgflg & MSG_COPY: return SEARCH_NUMBER.
- if *msgtyp == 0: return SEARCH_ANY.
- if *msgtyp < 0:
  - if *msgtyp == LONG_MIN: *msgtyp = LONG_MAX. /* per-undefined-behavior guard */
  - else: *msgtyp = -*msgtyp.
  - return SEARCH_LESSEQUAL.
- if msgflg & MSG_EXCEPT: return SEARCH_NOTEQUAL.
- return SEARCH_EQUAL.

REQ-11: testmsg(msg, type, mode):
- SEARCH_ANY → 1.
- SEARCH_LESSEQUAL → m_type ≤ type.
- SEARCH_EQUAL → m_type == type.
- SEARCH_NOTEQUAL → m_type != type.
- SEARCH_NUMBER → caller-counted (returns 1 unconditionally, count enforced in find_msg).

REQ-12: msgctl(msqid, cmd, buf):
- if msqid < 0 ∨ cmd < 0: return -EINVAL.
- switch cmd:
  - IPC_INFO / MSG_INFO: msgctl_info → copy_to_user struct msginfo.
  - MSG_STAT / MSG_STAT_ANY / IPC_STAT: msgctl_stat → copy_msqid_to_user.
  - IPC_SET: copy_msqid_from_user → msgctl_down(IPC_SET).
  - IPC_RMID: msgctl_down(IPC_RMID).
  - default: -EINVAL.

REQ-13: msgctl_down (IPC_SET / IPC_RMID):
- down_write(&msg_ids(ns).rwsem); rcu_read_lock.
- ipcctl_obtain_check(ns, &msg_ids(ns), msqid, cmd, perm, msg_qbytes). /* permission + capability */
- IPC_RMID:
  - ipc_lock_object; freeque(ns, ipcp). /* unlocks ipc + rcu */
- IPC_SET:
  - if msg_qbytes > ns->msg_ctlmnb ∧ !capable(CAP_SYS_RESOURCE): -EPERM.
  - ipc_lock_object; ipc_update_perm(perm, ipcp).
  - msq.q_qbytes = msg_qbytes; msq.q_ctime = ktime_get_real_seconds.
  - expunge_all(msq, -EAGAIN, &wake_q). /* receivers may lose perm */
  - ss_wakeup(msq, &wake_q, false). /* senders may now fit */
  - ipc_unlock_object; wake_up_q.
- up_write.

REQ-14: freeque(ns, ipcp):
- /* Called with msg_ids.rwsem (writer) + q_perm spinlock held; expunges and frees */
- expunge_all(msq, -EIDRM, &wake_q). /* sleeping receivers get -EIDRM */
- ss_wakeup(msq, &wake_q, true). /* sleeping senders zeroed list.next, wake */
- msg_rmid(ns, msq). /* ipc_rmid(&msg_ids(ns), &msq.q_perm) */
- ipc_unlock_object; wake_up_q; rcu_read_unlock.
- list_for_each_entry_safe(msg, t, &msq.q_messages, m_list):
  - percpu_counter_sub_local(&ns->percpu_msg_hdrs, 1).
  - free_msg(msg).
- percpu_counter_sub_local(&ns->percpu_msg_bytes, msq.q_cbytes).
- ipc_update_pid(&msq.q_lspid, NULL).
- ipc_update_pid(&msq.q_lrpid, NULL).
- ipc_rcu_putref(&msq.q_perm, msg_rcu_free). /* deferred kfree via RCU */

REQ-15: msg_init_ns / msg_exit_ns (ipc_namespace lifecycle):
- init: ns.msg_ctlmax = MSGMAX (8192); ns.msg_ctlmnb = MSGMNB (16384); ns.msg_ctlmni = MSGMNI (32000); percpu_counter_init(&ns.percpu_msg_bytes); percpu_counter_init(&ns.percpu_msg_hdrs); ipc_init_ids.
- exit: free_ipcs(ns, &msg_ids(ns), freeque); idr_destroy; rhashtable_destroy; percpu_counter_destroy.

REQ-16: Limits + ns-isolation:
- MSGMAX = 8192 bytes (per-message max; ns->msg_ctlmax overrides).
- MSGMNB = 16384 bytes (per-queue max bytes; ns->msg_ctlmnb).
- MSGMNI = 32000 queues / namespace (ns->msg_ctlmni).
- All limits live on `struct ipc_namespace`; sysctl writes to /proc/sys/kernel/{msgmax,msgmnb,msgmni} mutate ns fields.

REQ-17: Lockless-receive memory barriers (MSG_BARRIER):
- Sender path: `smp_store_release(&msr->r_msg, msg)` in pipelined_send.
- Receiver path: `READ_ONCE(msr_d.r_msg)` + `smp_acquire__after_ctrl_dep()` after observing non-`-EAGAIN`.
- Pairs with mqueue locking idiom; preserves `r_msg` initial value `-EAGAIN`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `msgsnd_size_bound` | INVARIANT | per-do_msgsnd: msgsz ≤ ns.msg_ctlmax. |
| `msgsnd_qbytes_bound` | INVARIANT | per-msg_fits_inqueue: cbytes + msgsz ≤ qbytes ∧ qnum + 1 ≤ qbytes. |
| `msgrcv_e2big_bounds` | INVARIANT | per-do_msgrcv: bufsz < m_ts ∧ !MSG_NOERROR ⟹ -E2BIG and message remains. |
| `pipelined_send_at_most_one` | INVARIANT | per-pipelined_send: returns 1 ⟹ exactly one receiver consumed. |
| `freeque_drains_senders_receivers` | INVARIANT | per-freeque: all senders woken with cleared list.next, all receivers stored -EIDRM. |
| `r_msg_release_acquire_paired` | INVARIANT | per-MSG_BARRIER: smp_store_release(&r_msg) ↔ smp_acquire__after_ctrl_dep(). |
| `ipcperms_checked_each_iter` | INVARIANT | per-do_msgsnd / do_msgrcv loop: ipcperms re-checked after wake. |

### Layer 2: TLA+

`ipc/msg.tla`:
- States: MsgGet, MsgSnd, MsgRcv, MsgCtlRmid, Sleeper, PipelinedDeliver.
- Properties:
  - `safety_qbytes_invariant` — q_cbytes + pending_send.msgsz never exceeds q_qbytes (only enqueue when fits).
  - `safety_qnum_le_qbytes` — qnum ≤ qbytes always.
  - `safety_no_msg_lost` — every msgsnd success ⟹ either pipelined-delivered or list_add_tail to q_messages.
  - `safety_rmid_wakes_all` — IPC_RMID ⟹ all sleeping receivers see -EIDRM, all sleeping senders woken.
  - `liveness_unblocked_after_rcv` — sender blocked because !fits eventually progresses after a msgrcv frees bytes.
  - `liveness_msgrcv_finds_msg` — msgrcv without IPC_NOWAIT eventually returns or signal_pending.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Msg::newque` post: q_qbytes == ns.msg_ctlmnb ∧ all lists empty | `Msg::newque` |
| `Msg::do_msgsnd` post: on success either pipelined or in q_messages, never both | `Msg::do_msgsnd` |
| `Msg::do_msgrcv` post: on success message removed from q_messages (unless MSG_COPY) | `Msg::do_msgrcv` |
| `Msg::pipelined_send` post: ret==1 ⟹ caller skips list_add_tail | `Msg::pipelined_send` |
| `Msg::convert_mode` post: SEARCH_* ∈ {ANY, EQUAL, NOTEQUAL, LESSEQUAL, NUMBER} | `Msg::convert_mode` |
| `Msg::testmsg` post: pure predicate | `Msg::testmsg` |
| `Msg::freeque` post: msq removed from ids; pending msgs freed; pid refs dropped | `Msg::freeque` |

### Layer 4: Verus/Creusot functional

`msgsnd → fit-check or block → pipelined_send-or-list_add → ksys_msgsnd return 0` semantic equivalence: per-SUS msgsnd(2) + per-Linux MSG_BARRIER lockless-wake; `msgrcv → find_msg → list_del or block → handler copy_to_user` per-SUS msgrcv(2) + per-Linux MSG_COPY (CHECKPOINT_RESTORE) / MSG_EXCEPT / MSG_NOERROR extensions.

### hardening

(Inherits row-1 features from `ipc/00-overview.md` § Hardening.)

Msg-queue reinforcement:

- **Per-msgsz ≤ ns.msg_ctlmax** — defense against per-userland oversize-message OOM.
- **Per-msgsz signed check `(long)msgsz < 0`** — defense against per-negative-size integer wraparound.
- **Per-mtype ≥ 1** — defense against per-broken-msgrcv-semantics (mtype=0 is "any").
- **Per-q_qbytes IPC_SET ≤ ns.msg_ctlmnb without CAP_SYS_RESOURCE** — defense against per-unprivileged DoS via huge queues.
- **Per-percpu_counter for global msg_bytes / msg_hdrs** — defense against per-counter contention DoS.
- **Per-CAP_IPC_OWNER for IPC_RMID / IPC_SET** — defense against cross-user destruction (enforced in ipcctl_obtain_check).
- **Per-LSM hook security_msg_queue_{alloc,msgsnd,msgrcv,msgctl,associate}** — defense via SELinux/AppArmor mediation.
- **Per-RCU rcu_free + ipc_rcu_putref** — defense against per-UAF on concurrent RMID.
- **Per-ipc_valid_object re-check after wake** — defense against per-stale-msq UAF (RMID race).
- **Per-MSG_BARRIER smp_store_release/acquire** — defense against per-receiver-reads-stale-r_msg.
- **Per-signal_pending ERESTARTNOHAND** — defense against per-uninterruptible-hang.
- **Per-`__randomize_layout`** — defense against per-known-offset KASLR-bypass attacks.
- **Per-MSG_COPY only with CHECKPOINT_RESTORE** — defense against per-leak (otherwise -ENOSYS).
- **Per-ipc_namespace isolation** — defense against per-cross-ns queue access.

