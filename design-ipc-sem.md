---
title: "Tier-3: ipc/sem.c — System V semaphores"
tags: ["tier-3", "ipc", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

System V **semaphores** are arrays of counting semaphores (1..SEMMSL per set) addressable by `key_t`/`semid`. Per-array: `struct sem_array` (sem_perm, sem_nsems, sems[], pending_alter, pending_const, list_id of undos, complex_count, use_global_lock). Per-semaphore: `struct sem` (semval, sempid, per-sem spinlock, per-sem pending_alter/_const, sem_otime; `____cacheline_aligned_in_smp`). Per-syscall: `semget(key, nsems, semflg)` creates / opens; `semop(semid, sops, nsops)` and `semtimedop(semid, sops, nsops, timeout)` apply an array of `struct sembuf` operations atomically (all-or-nothing; either all sops succeed without blocking, or the whole syscall blocks until they all can); `semctl(semid, semnum, cmd, arg)` GETVAL / SETVAL / GETALL / SETALL / GETPID / GETNCNT / GETZCNT / IPC_STAT / IPC_SET / IPC_RMID / IPC_INFO / SEM_INFO / SEM_STAT(_ANY). Per-process `struct sem_undo_list` (`task->sysvsem.undo_list`) tracks per-array `struct sem_undo` with `semadj[nsems]` short array; on exit, exit_sem reverses adjustments to prevent dead-process deadlocks. SEM_UNDO requires per-op `semadj` to stay within [-SEMAEM-1, SEMAEM]. Per-array two-tier locking: per-semaphore spinlock for simple ops (`nsops == 1`), global sem_perm.lock for complex ops (`nsops > 1` ∨ contention) with `use_global_lock` hysteresis (USE_GLOBAL_LOCK_HYSTERESIS = 10). Critical for: SUS semop semantics, FIFO ordering, lockless wake-up, undo-on-exit deadlock-prevention.

This Tier-3 covers `ipc/sem.c` (~2484 lines).

### Acceptance Criteria

- [ ] AC-1: semget(IPC_PRIVATE, 4, 0666) creates 4-semaphore array; sem_ctime set.
- [ ] AC-2: semget with nsems > ns.sc_semmsl → -EINVAL.
- [ ] AC-3: semget when used_sems + nsems > ns.sc_semmns → -ENOSPC.
- [ ] AC-4: semop with single +1 op: succeeds, increments semval, updates sempid.
- [ ] AC-5: semop with single -1 op on semval==0 + IPC_NOWAIT → -EAGAIN.
- [ ] AC-6: semop with single -1 op on semval==0 + !IPC_NOWAIT blocks until increment.
- [ ] AC-7: semop with two ops failing mid-array: array unchanged (atomic).
- [ ] AC-8: semop pushing semval > SEMVMX → -ERANGE.
- [ ] AC-9: semop with SEM_UNDO pushing semadj > SEMAEM → -ERANGE.
- [ ] AC-10: process exit with pending semadj: exit_sem reverses adjustments; clamps to [0, SEMVMX].
- [ ] AC-11: CLONE_SYSVSEM: child shares parent's undo_list (refcnt++).
- [ ] AC-12: !CLONE_SYSVSEM: child gets NULL undo_list (fresh).
- [ ] AC-13: semtimedop with timeout expires → -EAGAIN; queue.list unlinked.
- [ ] AC-14: semop with nsops > ns.sc_semopm → -E2BIG.
- [ ] AC-15: semctl(GETVAL) returns sems[semnum].semval.
- [ ] AC-16: semctl(SETVAL) clears all undos' semadj[semnum] for this array.
- [ ] AC-17: semctl(IPC_RMID) wakes all pending with -EIDRM; invalidates undos (semid=-1).
- [ ] AC-18: wait-for-zero (sem_op==0) wakes when semval reaches 0.

### Architecture

```
struct SemArray {
  sem_perm: KernIpcPerm,
  sem_ctime: i64,
  pending_alter: ListHead<SemQueue>,
  pending_const: ListHead<SemQueue>,
  list_id: ListHead<SemUndo>,
  sem_nsems: i32,
  complex_count: i32,
  use_global_lock: u32,                // hysteresis
  sems: [Sem; sem_nsems],              // flex
}

struct Sem {
  semval: i32,                         // 0..SEMVMX
  sempid: Option<*Pid>,
  lock: SpinLock,                      // ____cacheline_aligned_in_smp
  pending_alter: ListHead<SemQueue>,
  pending_const: ListHead<SemQueue>,
  sem_otime: i64,
}

struct SemQueue {
  list: ListLink,
  sleeper: *TaskStruct,
  undo: Option<*SemUndo>,
  pid: *Pid,
  status: AtomicI32,                   // -EINTR initial; smp_store_release(result)
  sops: *SemBuf,
  blocking: *SemBuf,
  nsops: i32,
  alter: bool,
  dupsop: bool,
}

struct SemUndo {
  list_proc: ListLink,                 // in ulp.list_proc (rcu)
  rcu: RcuHead,
  ulp: *SemUndoList,
  list_id: ListLink,                   // in sma.list_id
  semid: i32,                          // -1 on RMID
  semadj: [i16; nsems],                // bounded [-SEMAEM-1, SEMAEM]
}

struct SemUndoList {
  refcnt: RefCount,                    // CLONE_SYSVSEM
  lock: SpinLock,
  list_proc: ListHead<SemUndo>,
}
```

`Sem::sys_semget(key, nsems, semflg) -> Result<i32>`:
1. ns = current.nsproxy.ipc_ns.
2. if nsems < 0 ∨ nsems > ns.sc_semmsl: -EINVAL.
3. sem_ops = {.getnew = Sem::newary, .associate, .more_checks}.
4. return ipcget(ns, &sem_ids(ns), &sem_ops, ¶ms).

`Sem::do_semtimedop_inner(semid, sops, nsops, timeout, ns) -> Result<()>`:
1. Pre-scan sops: compute max, undos, dupsop, alter.
2. if undos: un = find_alloc_undo(ns, semid).
3. sma = sem_obtain_object_check.
4. ipcperms + security_sem_semop.
5. locknum = sem_lock(sma, sops, nsops).
6. queue = build SemQueue.
7. error = Sem::perform_atomic(sma, &queue).
8. if error == 0: do_smart_update or set_semotime; unlock; wake_up_q; return Ok.
9. if error < 0: unlock; return Err.
10. /* Block */
11. Enqueue queue into per-sem or per-array pending list (alter/const).
12. Loop: WRITE_ONCE(queue.status, -EINTR); TASK_INTERRUPTIBLE; unlock; schedule_hrtimeout_range; re-lock; READ_ONCE(queue.status).
13. On wake: unlink_queue; return status.

`Sem::perform_atomic(sma, q) -> i32`:
1. /* Validate */
2. For each sop in q.sops:
   - idx = array_index_nospec(sop.sem_num, sma.sem_nsems).
   - result = sems[idx].semval + sop.sem_op.
   - if sop.sem_op == 0 ∧ sems[idx].semval != 0: would_block.
   - if result < 0: would_block.
   - if result > SEMVMX: return -ERANGE.
   - if sop.sem_flg & SEM_UNDO: bounds-check un.semadj[sop.sem_num] - sop.sem_op ∈ [-SEMAEM-1, SEMAEM].
3. /* Apply */
4. For each sop: update semval, semadj, sempid.
5. return 0.

`Sem::find_alloc_undo(ns, semid) -> Result<*SemUndo>`:
1. get_undo_list(&ulp). /* allocates task.sysvsem.undo_list if NULL */
2. rcu_read_lock; lookup_undo(ulp, semid) — return if found.
3. sma = sem_obtain_object_check; nsems = sma.sem_nsems.
4. ipc_rcu_getref(&sma.sem_perm); rcu_read_unlock.
5. new = kvzalloc_flex(*new, semadj, nsems, GFP_KERNEL_ACCOUNT).
6. rcu_read_lock; sem_lock_and_putref(sma).
7. spin_lock(&ulp.lock); re-check lookup_undo (race).
8. If new: new.ulp = ulp; new.semid = semid; list_add_rcu(&new.list_proc, &ulp.list_proc); list_add(&new.list_id, &sma.list_id).
9. sem_unlock; return un.

`Sem::exit_sem(tsk)`:
1. ulp = tsk.sysvsem.undo_list; if !ulp: return.
2. tsk.sysvsem.undo_list = NULL.
3. if !refcount_dec_and_test(&ulp.refcnt): return. /* still shared */
4. For each un in ulp.list_proc:
   - rcu_read_lock; sma = sem_obtain_object_check(ns, un.semid).
   - sem_lock(sma, NULL, -1).
   - For i in 0..sma.sem_nsems:
     - if un.semadj[i]: sems[i].semval += un.semadj[i]; clamp to [0, SEMVMX]; update sempid.
   - do_smart_update(sma, NULL, 0, 1, &wake_q).
   - list_del(&un.list_id).
   - sem_unlock; wake_up_q.
   - kvfree_rcu(un, rcu).

`Sem::freeary(ns, ipcp)`:
1. For each un in sma.list_id: list_del; un.semid = -1; list_del_rcu(&un.list_proc); kvfree_rcu(un).
2. For each q in sma.pending_const / pending_alter: unlink_queue; wake_up_sem_queue_prepare(q, -EIDRM, wake_q).
3. For i in 0..nsems: same on sems[i].pending_*.
4. sem_rmid; sem_unlock; wake_up_q.
5. ns.used_sems -= sma.sem_nsems.
6. ipc_rcu_putref(&sma.sem_perm, sem_rcu_free).

`Sem::init_ns(ns)`:
1. ns.sc_semmsl = SEMMSL (32000).
2. ns.sc_semmns = SEMMNS (SEMMNI * SEMMSL).
3. ns.sc_semopm = SEMOPM (500).
4. ns.sc_semmni = SEMMNI (32000).
5. ns.used_sems = 0.
6. ipc_init_ids(&ns.ids[IPC_SEM_IDS]).

### Out of Scope

- `ipc/util.c` ipc_addid / ipcget / ipcctl_obtain_check / ipcperms — covered in `ipc/util.md` Tier-3.
- `ipc/namespace.c` ipc_namespace lifecycle — covered in `ipc/namespace.md`.
- POSIX semaphores (`sem_open` in glibc + futex-backed) — separate, not in ipc/sem.c.
- LSM security_sem_* implementations (`security/selinux/`, `security/smack/`).
- /proc/sys/kernel/sem sysctl write path (handled in `kernel/sysctl.c`).
- compat (32-bit) ABI marshaling beyond what semctl_compat does inline.
- Implementation code.

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct sem` | per-individual-semaphore state | `Sem` |
| `struct sem_array` | per-semaphore-set | `SemArray` |
| `struct sem_queue` | per-pending-operation | `SemQueue` |
| `struct sem_undo` | per-process per-array undo record | `SemUndo` |
| `struct sem_undo_list` | per-task undo list (task.sysvsem.undo_list) | `SemUndoList` |
| `struct sembuf` (UAPI) | per-op { sem_num, sem_op, sem_flg } | `SemBuf` |
| `ksys_semget()` / `SYSCALL_DEFINE3(semget, ...)` | per-syscall create/open | `Sem::sys_semget` |
| `SYSCALL_DEFINE3(semop, ...)` | per-syscall op (wraps semtimedop) | `Sem::sys_semop` |
| `ksys_semtimedop()` / `SYSCALL_DEFINE4(semtimedop, ...)` | per-syscall timed op | `Sem::sys_semtimedop` |
| `ksys_semctl()` / `SYSCALL_DEFINE4(semctl, ...)` | per-syscall control | `Sem::sys_semctl` |
| `newary()` | per-newseg constructor | `Sem::newary` |
| `freeary()` | per-IPC_RMID destructor | `Sem::freeary` |
| `__do_semtimedop()` | per-op core | `Sem::do_semtimedop_inner` |
| `perform_atomic_semop()` / `_slow()` | per-array atomic apply | `Sem::perform_atomic` / `_slow` |
| `update_queue()` / `do_smart_update()` | per-wake pending alter ops | `Sem::update_queue` / `do_smart_update` |
| `wake_const_ops()` / `do_smart_wakeup_zero()` | per-wake wait-for-zero | `Sem::wake_const_ops` / `do_smart_wakeup_zero` |
| `wake_up_sem_queue_prepare()` | per-set-status-and-wake | `Sem::wake_queue_prepare` |
| `find_alloc_undo()` | per-lookup-or-create undo | `Sem::find_alloc_undo` |
| `get_undo_list()` | per-task-undo-list lookup/alloc | `Sem::get_undo_list` |
| `lookup_undo()` / `__lookup_undo()` | per-undo lookup | `Sem::lookup_undo` |
| `copy_semundo()` | per-CLONE_SYSVSEM fork copy | `Sem::copy_semundo` |
| `exit_sem()` | per-task-exit undo replay | `Sem::exit_sem` |
| `sem_lock()` / `sem_unlock()` / `complexmode_enter()` / `complexmode_tryleave()` | per-array two-tier lock | `Sem::lock` / `unlock` / `complex_enter` / `complex_tryleave` |
| `semctl_setval` / `semctl_main` / `semctl_stat` / `semctl_info` / `semctl_down` | per-cmd dispatch | `Sem::ctl_setval` / `ctl_main` / `ctl_stat` / `ctl_info` / `ctl_down` |
| `count_semcnt()` | per-GETNCNT/GETZCNT count | `Sem::count_semcnt` |
| `check_qop()` | per-complex-op blocker test | `Sem::check_qop` |
| `merge_queues()` / `unmerge_queues()` | per-complex-mode list merge | `Sem::merge_queues` / `unmerge_queues` |
| `set_semotime()` / `get_semotime()` | per-otime book-keeping | `Sem::set_semotime` / `get_semotime` |
| `sem_init_ns()` / `sem_exit_ns()` | per-namespace lifecycle | `Sem::init_ns` / `exit_ns` |

### compatibility contract

REQ-1: struct sem:
- semval: per-current-value (int, 0..SEMVMX).
- sempid: per-`struct pid` of last modifier (semop, semctl SETVAL/SETALL, exit_sem undo).
- lock: per-semaphore spinlock (fine-grained simple-op path).
- pending_alter: per-list of `struct sem_queue` (single-sop, alters this sem).
- pending_const: per-list of `struct sem_queue` (single-sop, wait-for-zero).
- sem_otime: per-replicated sem_otime candidate (cache-line trashing avoidance).
- ____cacheline_aligned_in_smp: per-cache-line alignment.

REQ-2: struct sem_array:
- sem_perm: per-`struct kern_ipc_perm`.
- sem_ctime: per-create / last semctl change time.
- pending_alter: per-list of complex sem_queues that alter (nsops > 1).
- pending_const: per-list of complex sem_queues wait-for-zero.
- list_id: per-list of all `struct sem_undo` referencing this array.
- sem_nsems: per-array size.
- complex_count: per-count of pending complex ops.
- use_global_lock: per-hysteresis counter (>0: global lock required for any op).
- sems[]: per-flex `struct sem` array.
- __randomize_layout.

REQ-3: struct sem_queue:
- list: per-list-link (per-sem.pending_alter, pending_const, or sma->pending_*).
- sleeper: per-task_struct of sleeper.
- undo: per-`struct sem_undo` pointer or NULL.
- pid: per-`struct pid` of submitter (task_tgid(current)).
- status: per-result (-EINTR initial, set via smp_store_release to result).
- sops: per-`struct sembuf *` (FAST stack-buffer up to SEMOPM_FAST=64 or kvmalloc).
- blocking: per-`struct sembuf *` (the op that caused block; used for GETNCNT/GETZCNT).
- nsops: per-op-count.
- alter: per-bool (does sops alter any sem?).
- dupsop: per-bool (sops on more than one sem_num — uses slow path).

REQ-4: struct sem_undo:
- list_proc: per-process list-link (rcu-protected; in ulp->list_proc).
- rcu: per-rcu_head for kvfree_rcu.
- ulp: per-back-pointer to sem_undo_list.
- list_id: per-array list-link (in sma->list_id).
- semid: per-array id (set to -1 on RMID).
- semadj[]: per-flex short array, one adjustment per semaphore.

REQ-5: struct sem_undo_list (task->sysvsem.undo_list):
- refcnt: per-refcount_t (CLONE_SYSVSEM share via copy_semundo).
- lock: per-spinlock for list_proc.
- list_proc: per-list of sem_undo for this task-group.

REQ-6: semget(key, nsems, semflg) (ksys_semget):
- ns = current.nsproxy.ipc_ns.
- if nsems < 0 ∨ nsems > ns.sc_semmsl: return -EINVAL.
- sem_ops = {.getnew = newary, .associate = security_sem_associate, .more_checks = sem_more_checks}.
- return ipcget(ns, &sem_ids(ns), &sem_ops, ¶ms).

REQ-7: newary(ns, params):
- if !nsems: return -EINVAL.
- if ns.used_sems + nsems > ns.sc_semmns: return -ENOSPC.
- sma = sem_alloc(nsems). /* kvzalloc_flex *sma + sems[nsems] */
- sma.sem_perm.mode = semflg & S_IRWXUGO.
- security_sem_alloc(&sma.sem_perm).
- for i in 0..nsems: INIT_LIST_HEAD(pending_alter); INIT_LIST_HEAD(pending_const); spin_lock_init(&sems[i].lock).
- sma.complex_count = 0; sma.use_global_lock = USE_GLOBAL_LOCK_HYSTERESIS (10).
- INIT_LIST_HEAD on pending_alter, pending_const, list_id.
- sma.sem_nsems = nsems.
- sma.sem_ctime = ktime_get_real_seconds.
- ipc_addid(&sem_ids(ns), &sma.sem_perm, ns.sc_semmni).
- ns.used_sems += nsems.
- sem_unlock(sma, -1); rcu_read_unlock.
- return sma.sem_perm.id.

REQ-8: sem_lock(sma, sops, nsops) → locknum:
- /* Per-fast-path: nsops == 1 and no contention */
- if nsops == 1 ∧ !sma.complex_count:
  - spin_lock(&sem->lock). /* per-sem fine-grained */
  - if !use_global_lock: return semnum.
  - /* fallback: lost race with complex op; drop and retry global */
- /* Global lock path */
- ipc_lock_object(&sma.sem_perm); /* global rwsem-protected spinlock */
- complexmode_enter(sma); /* per-sem.lock acquired for all sems */
- return -1.

REQ-9: perform_atomic_semop(sma, q) (fast path, q.dupsop == false):
- /* Two-pass: validate then apply */
- for sop in q.sops:
  - idx = array_index_nospec(sop.sem_num, sma.sem_nsems).
  - sem_op = sop.sem_op; result = sems[idx].semval.
  - if !sem_op ∧ result: goto would_block. /* wait-for-zero */
  - result += sem_op.
  - if result < 0: goto would_block. /* try-to-decrement-below-0 */
  - if result > SEMVMX: return -ERANGE.
  - if sop.sem_flg & SEM_UNDO:
    - undo = un.semadj[sop.sem_num] - sem_op.
    - if undo < -SEMAEM-1 ∨ undo > SEMAEM: return -ERANGE.
- /* Apply */
- for sop in q.sops:
  - if sop.sem_flg & SEM_UNDO: un.semadj[sop.sem_num] -= sem_op.
  - sems[sop.sem_num].semval += sem_op.
  - ipc_update_pid(&sems[sop.sem_num].sempid, q.pid).
- return 0.
- would_block: q.blocking = sop; return (sop.sem_flg & IPC_NOWAIT ? -EAGAIN : 1).

REQ-10: perform_atomic_semop_slow (q.dupsop == true): same logic but iterative with rollback for partial apply if any op fails mid-array.

REQ-11: __do_semtimedop(semid, sops, nsops, timeout, ns):
- if nsops < 1 ∨ semid < 0: return -EINVAL.
- if nsops > ns.sc_semopm: return -E2BIG.
- if timeout: expires = ktime_add_safe(ktime_get(), timespec64_to_ktime(*timeout)); exp = &expires.
- /* Scan sops */
- for sop in sops:
  - if sop.sem_num >= max: max = sop.sem_num.
  - if sop.sem_flg & SEM_UNDO: undos = true.
  - mask = 1ULL << (sop.sem_num % BITS_PER_LONG).
  - if dup & mask: dupsop = true.
  - if sop.sem_op != 0: alter = true; dup |= mask.
- /* Allocate undo if needed */
- if undos: un = find_alloc_undo(ns, semid). else: un = NULL; rcu_read_lock.
- sma = sem_obtain_object_check(ns, semid).
- if max >= sma.sem_nsems: return -EFBIG.
- ipcperms(ns, &sma.sem_perm, alter ? S_IWUGO : S_IRUGO).
- security_sem_semop(&sma.sem_perm, sops, nsops, alter).
- locknum = sem_lock(sma, sops, nsops).
- if !ipc_valid_object: return -EIDRM.
- if un ∧ un.semid == -1: return -EIDRM. /* concurrent RMID + id reuse */
- queue = {.sops, .nsops, .undo = un, .pid = task_tgid(current), .alter, .dupsop}.
- error = perform_atomic_semop(sma, &queue).
- if error == 0:
  - if alter: do_smart_update(sma, sops, nsops, 1, &wake_q). /* wake other pending ops */
  - else: set_semotime(sma, sops).
  - sem_unlock; rcu_read_unlock; wake_up_q.
  - return 0.
- if error < 0: return error. /* -EAGAIN, -ERANGE */
- /* error == 1: block */
- /* Enqueue based on nsops */
- if nsops == 1:
  - curr = &sems[idx].
  - if alter:
    - if sma.complex_count: list_add_tail(&queue.list, &sma.pending_alter).
    - else: list_add_tail(&queue.list, &curr.pending_alter).
  - else: list_add_tail(&queue.list, &curr.pending_const).
- else (complex):
  - if !sma.complex_count: merge_queues(sma). /* fold per-sem queues into sma->pending */
  - if alter: list_add_tail(&queue.list, &sma.pending_alter).
  - else: list_add_tail(&queue.list, &sma.pending_const).
  - sma.complex_count++.
- /* Sleep */
- do:
  - WRITE_ONCE(queue.status, -EINTR).
  - queue.sleeper = current.
  - __set_current_state(TASK_INTERRUPTIBLE).
  - sem_unlock; rcu_read_unlock.
  - timed_out = !schedule_hrtimeout_range(exp, current.timer_slack_ns, HRTIMER_MODE_ABS).
  - rcu_read_lock.
  - error = READ_ONCE(queue.status).
  - if error != -EINTR: smp_acquire__after_ctrl_dep; return error.
  - sem_lock; if !ipc_valid_object: return -EIDRM.
  - error = READ_ONCE(queue.status).
  - if error != -EINTR: return error.
  - if timed_out: error = -EAGAIN.
- while error == -EINTR ∧ !signal_pending(current). /* spurious wake */
- unlink_queue(sma, &queue).
- return error.

REQ-12: do_smart_update(sma, sops, nsops, otime, wake_q):
- /* Wake waiters whose ops may now succeed */
- otime |= do_smart_wakeup_zero(sma, sops, nsops, wake_q). /* wait-for-zero on incremented sems */
- if !list_empty(&sma.pending_alter):
  - update_queue(sma, -1, wake_q). /* global complex queue */
- else if !sops:
  - for i in 0..nsems: update_queue(sma, i, wake_q).
- else:
  - /* Only sems that were incremented could allow waiters to proceed */
  - for sop in sops: if sop.sem_op > 0: update_queue(sma, sop.sem_num, wake_q).
- if otime: set_semotime(sma, sops).

REQ-13: update_queue(sma, semnum, wake_q):
- pending_list = (semnum == -1 ? &sma.pending_alter : &sems[semnum].pending_alter).
- again: list_for_each_entry_safe(q, tmp, pending_list, list):
  - if semnum != -1 ∧ sems[semnum].semval == 0: break. /* decrements blocked */
  - error = perform_atomic_semop(sma, q).
  - if error > 0: continue. /* still blocked */
  - unlink_queue(sma, q).
  - if !error:
    - do_smart_wakeup_zero(sma, q.sops, q.nsops, wake_q).
    - restart = check_restart(sma, q).
  - wake_up_sem_queue_prepare(q, error, wake_q).
  - if restart: goto again.

REQ-14: wake_up_sem_queue_prepare(q, error, wake_q):
- sleeper = get_task_struct(q.sleeper).
- smp_store_release(&q.status, error). /* SEM_BARRIER_2 */
- wake_q_add_safe(wake_q, sleeper).

REQ-15: SEM_UNDO mechanics:
- find_alloc_undo(ns, semid):
  - get_undo_list(&ulp). /* allocates task.sysvsem.undo_list if NULL */
  - lookup_undo(ulp, semid). /* RCU-protected list_proc walk */
  - if not found: kvzalloc_flex(*new, semadj, nsems, GFP_KERNEL_ACCOUNT).
  - sem_lock_and_putref + ipc_valid_object check; race retry.
  - new.ulp = ulp; new.semid = semid.
  - list_add_rcu(&new.list_proc, &ulp.list_proc).
  - list_add(&new.list_id, &sma.list_id).
- perform_atomic_semop with SEM_UNDO: un.semadj[sop.sem_num] -= sop.sem_op.
- Bound: |semadj[i]| ≤ SEMAEM (SEMAEM = SEMVMX = 32767).

REQ-16: copy_semundo(clone_flags, tsk):
- if CLONE_SYSVSEM: get_undo_list(&undo_list); refcount_inc(&undo_list.refcnt); tsk.sysvsem.undo_list = undo_list.
- else: tsk.sysvsem.undo_list = NULL. /* fresh on first semop */

REQ-17: exit_sem(tsk):
- ulp = tsk.sysvsem.undo_list. if !ulp: return.
- tsk.sysvsem.undo_list = NULL.
- if !refcount_dec_and_test(&ulp.refcnt): return.
- for each un in ulp.list_proc:
  - sma = sem_obtain_object_check(ns, un.semid).
  - /* Apply semadj[] adjustments to sems[] */
  - for i in 0..sma.sem_nsems:
    - delta = un.semadj[i].
    - if delta:
      - sems[i].semval += delta.
      - if sems[i].semval < 0: sems[i].semval = 0. /* clamp */
      - if sems[i].semval > SEMVMX: sems[i].semval = SEMVMX. /* clamp */
      - ipc_update_pid(&sems[i].sempid, task_tgid(tsk)).
  - do_smart_update(sma, NULL, 0, 1, &wake_q). /* wake waiters */
  - list_del(&un.list_id).
  - kvfree_rcu(un, rcu).

REQ-18: semctl(semid, semnum, cmd, arg):
- IPC_INFO / SEM_INFO: semctl_info → copy_to_user struct seminfo.
- IPC_STAT / SEM_STAT / SEM_STAT_ANY: semctl_stat → copy_semid_to_user.
- GETVAL: read sems[semnum].semval (with ipc_valid_object check).
- SETVAL: semctl_setval (val ≤ SEMVMX); resets all undos' semadj[semnum] = 0; do_smart_update.
- GETALL / SETALL: semctl_main (kvmalloc_array if nsems > SEMMSL_FAST=256).
- GETPID: pid_vnr(sems[semnum].sempid).
- GETNCNT / GETZCNT: count_semcnt(sma, semnum, false/true).
- IPC_SET: ipcctl_obtain_check; ipc_update_perm; sem_ctime updated.
- IPC_RMID: freeary(ns, ipcp).

REQ-19: freeary(ns, ipcp):
- /* Free undos for this array */
- list_for_each_entry_safe(un, tu, &sma.list_id, list_id):
  - list_del(&un.list_id).
  - spin_lock(&un.ulp.lock); un.semid = -1; list_del_rcu(&un.list_proc); spin_unlock.
  - kvfree_rcu(un, rcu).
- /* Wake all pending with -EIDRM */
- list_for_each_entry_safe(q, ..., &sma.pending_const + sma.pending_alter): unlink_queue; wake_up_sem_queue_prepare(q, -EIDRM, wake_q).
- for i in 0..nsems: same on sems[i].pending_const + pending_alter.
- ipc_update_pid(&sems[i].sempid, NULL).
- sem_rmid(ns, sma); sem_unlock; rcu_read_unlock.
- wake_up_q; ns.used_sems -= sma.sem_nsems; ipc_rcu_putref(&sma.sem_perm, sem_rcu_free).

REQ-20: sem_init_ns / sem_exit_ns:
- init: ns.sc_semmsl = SEMMSL (32000); ns.sc_semmns = SEMMNS (SEMMNI*SEMMSL); ns.sc_semopm = SEMOPM (500); ns.sc_semmni = SEMMNI (32000); ns.used_sems = 0; ipc_init_ids.
- exit: free_ipcs(ns, &sem_ids(ns), freeary); idr_destroy; rhashtable_destroy.

REQ-21: Memory ordering (SEM_BARRIER_1, _2):
- use_global_lock 0→non-zero: smp_load_acquire ↔ spin_lock+spin_unlock pair.
- queue.status: smp_store_release(&q.status, error) ↔ READ_ONCE + smp_acquire__after_ctrl_dep.
- queue.sleeper handled by wake_q_add_safe.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `semval_bounded` | INVARIANT | per-perform_atomic: 0 ≤ semval ≤ SEMVMX. |
| `semadj_bounded` | INVARIANT | per-SEM_UNDO: -SEMAEM-1 ≤ semadj[i] ≤ SEMAEM. |
| `atomic_all_or_nothing` | INVARIANT | per-perform_atomic: success ⟹ all sops applied; failure ⟹ none applied (slow path rolls back). |
| `array_index_nospec_used` | INVARIANT | per-sop.sem_num: bounded via array_index_nospec. |
| `nsops_le_semopm` | INVARIANT | per-do_semtimedop: nsops ≤ ns.sc_semopm. |
| `undo_list_refcount_balanced` | INVARIANT | per-copy_semundo / exit_sem: refcnt inc on CLONE_SYSVSEM, dec_and_test on exit. |
| `undo_semid_invalidated_on_rmid` | INVARIANT | per-freeary: un.semid = -1 before kvfree_rcu. |
| `sem_barrier_status_release_acquire` | INVARIANT | per-SEM_BARRIER_2: smp_store_release ↔ READ_ONCE + smp_acquire__after_ctrl_dep. |

### Layer 2: TLA+

`ipc/sem.tla`:
- States: SemGet, SemOp{Validate, Apply, Block, Wake}, SemCtlRmid, ExitSemReplay, CloneSysvSem.
- Properties:
  - `safety_atomicity` — perform_atomic either applies all or applies none.
  - `safety_semval_bound` — for all sems: 0 ≤ semval ≤ SEMVMX after every transition.
  - `safety_semadj_bound` — |semadj[i]| ≤ SEMAEM after every undo update.
  - `safety_rmid_invalidates_undos` — IPC_RMID ⟹ ∀ un ∈ sma.list_id: un.semid = -1.
  - `safety_fifo_alter` — pending_alter ops processed in FIFO order (no starvation guarantee beyond FIFO).
  - `safety_wait_for_zero_wakeup` — sem.semval == 0 ⟹ pending_const on this sem evaluated.
  - `liveness_semop_terminates` — semop returns or signal_pending eventually.
  - `liveness_exit_sem_completes` — task exit eventually completes undo replay for all undos.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Sem::newary` post: sem_nsems == params.nsems ∧ used_sems += nsems | `Sem::newary` |
| `Sem::perform_atomic` post: ret==0 ⟹ all sops applied; ret==1 ⟹ no state change | `Sem::perform_atomic` |
| `Sem::do_semtimedop_inner` post: queue removed from pending on return (unless still queued waiting) | `Sem::do_semtimedop_inner` |
| `Sem::find_alloc_undo` post: returned un has un.semid == semid ∧ semadj sized to sma.sem_nsems | `Sem::find_alloc_undo` |
| `Sem::exit_sem` post: tsk.sysvsem.undo_list == NULL; clamped semvals; undos freed | `Sem::exit_sem` |
| `Sem::freeary` post: sma removed; all undos invalidated; all sleepers woken with -EIDRM | `Sem::freeary` |
| `Sem::copy_semundo` post: CLONE_SYSVSEM ⟹ refcnt++; else child undo_list == NULL | `Sem::copy_semundo` |

### Layer 4: Verus/Creusot functional

`semop(sops) → perform_atomic → apply-or-block → wake-via-update_queue → status-via-smp_store_release` semantic equivalence: per-SUS semop(2) + per-Linux SEM_UNDO + per-Manfred-Spraul lockless-wake. `exit_sem` replay equivalence: per-SUS-Linux semop(2) "set of adjustments are limited to 0..SEMVMX" with clamp semantics.

### hardening

(Inherits row-1 features from `ipc/00-overview.md` § Hardening.)

Sem-array reinforcement:

- **Per-sop.sem_num via array_index_nospec** — defense against per-Spectre-v1 speculative OOB.
- **Per-SEMVMX upper bound on semval** — defense against per-integer-wrap / per-stale-semval DoS.
- **Per-SEMAEM bound on semadj** — defense against per-malicious-undo overflow.
- **Per-SEMOPM bound on nsops** — defense against per-massive-sops stack/heap exhaust.
- **Per-SEMMSL bound on nsems at create** — defense against per-huge-array memory exhaust.
- **Per-SEMMNS bound on global semaphore count** — defense against per-namespace exhaustion.
- **Per-SEM_UNDO mandatory bounds revalidation each op** — defense against per-undo-drift across exits.
- **Per-RCU sem_rcu_free + kvfree_rcu(un)** — defense against per-UAF on concurrent RMID + exit_sem.
- **Per-un.semid == -1 race detection** — defense against per-id-reuse undo-on-wrong-array.
- **Per-sem_lock two-tier (fine-grained + global)** — defense against per-O(N) lock-contention DoS.
- **Per-use_global_lock hysteresis** — defense against per-mode-thrash on mixed ops.
- **Per-CAP_IPC_OWNER for IPC_RMID/IPC_SET** — defense against per-cross-user destruction.
- **Per-LSM hook security_sem_{alloc,semop,semctl,associate}** — defense via SELinux/AppArmor mediation.
- **Per-`__randomize_layout`** — defense against per-known-offset attacks.
- **Per-ipc_namespace isolation** — defense against per-cross-ns semaphore access.

