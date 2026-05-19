---
title: "Tier-5: syscall 243 — mq_timedreceive(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`mq_timedreceive(2)` is **x86_64 syscall 243**, the POSIX message-queue receiver with a deadline. It removes the highest-priority message from the queue referenced by `mqdes`, copying its payload (at most `msg_len` bytes — the buffer must be ≥ `mq_msgsize`) into `msg_ptr` and the originating priority into `*msg_prio` (if non-NULL). If the queue is empty and `O_NONBLOCK` is not set, the caller blocks on the queue's `wait_recv` waitqueue until a message arrives, the absolute deadline `abs_timeout` (`CLOCK_REALTIME`) elapses, or a signal is delivered.

The receive path is the symmetric peer of `mq_timedsend(2)`: it consumes from the per-priority FIFO list backed by `struct mqueue_inode_info`. If a sender is blocked because the queue was full, the kernel performs a *pipelined receive* — the receiver wakes that sender's stashed message, allowing one queue slot to be reused without taking the ring slow-path.

Critical for: every POSIX-RT consumer, every real-time task dispatching IPC events under deadline, every Rookery test asserting priority dequeue order, blocking semantics, and signal cancellation.

### Acceptance Criteria

- [ ] AC-1: Send "x" prio 5, then `mq_timedreceive` ⟹ returns `1`, `msg_ptr = "x"`, `*msg_prio = 5`.
- [ ] AC-2: Multiple priorities: dequeue order 31000 → 5 → 0 regardless of send order.
- [ ] AC-3: `msg_len < mq_msgsize` ⟹ `-EMSGSIZE`.
- [ ] AC-4: Empty queue + `O_NONBLOCK` ⟹ `-EAGAIN`.
- [ ] AC-5: Empty queue + blocking + deadline past ⟹ `-ETIMEDOUT`.
- [ ] AC-6: Signal during block ⟹ `-EINTR`; no message consumed.
- [ ] AC-7: `mqdes` opened `O_WRONLY` ⟹ `-EBADF`.
- [ ] AC-8: Non-mqueue fd ⟹ `-EBADMSG`.
- [ ] AC-9: Sender blocked on full queue; receiver consumes ⟹ pipelined-receive wakes sender with `0`, queue state advances by one.
- [ ] AC-10: `msg_prio = NULL` honored: receive succeeds without priority write.
- [ ] AC-11: `abs_timeout->tv_nsec >= 1e9` ⟹ `-EINVAL` (pre-validation).
- [ ] AC-12: Successful receive frees `msg_msg` slab object (no leak detected by KMSAN).

### Architecture

```
struct MqTimedreceiveArgs {
    mqdes: i32,
    msg_ptr: UserPtr<u8>,
    msg_len: usize,
    msg_prio: UserPtr<u32>,
    abs_timeout: UserPtr<Timespec>,
}
```

`Mqueue::sys_mq_timedreceive(args) -> isize`:

1. `let timeout = if !args.abs_timeout.is_null() { Some(Timespec::copy_from_user_validated(args.abs_timeout)?) } else { None };`
2. `let file = current().files.fget_read(args.mqdes)?;`
3. `if !Mqueue::is_mqueue_file(&file) { return -EBADMSG; }`
4. `let info = mqueue_inode_info(file.inode);`
5. `if args.msg_len < info.attr.mq_msgsize { return -EMSGSIZE; }`
6. `return Mqueue::do_receive(&file, info, args.msg_ptr, args.msg_len, args.msg_prio, timeout);`

`Mqueue::do_receive(file, info, ubuf, ulen, uprio, timeout) -> isize`:

1. `let nonblock = (file.f_flags & O_NONBLOCK) != 0;`
2. `info.lock.lock();`
3. `if info.attr.mq_curmsgs == 0 {`
4. `    if nonblock { info.lock.unlock(); return -EAGAIN; }`
5. `    let waiter = WaitEntry::for_recv(current());`
6. `    info.wait[RECV].push(&waiter);`
7. `    info.lock.unlock();`
8. `    match schedule_hrtimeout_until(timeout, CLOCK_REALTIME) { TimedOut => return -ETIMEDOUT, Interrupted => return -EINTR, Resumed => () }`
9. `    /* sender's pipelined_send placed message into waiter.msg */`
10. `    return Mqueue::deliver(waiter.msg, ubuf, ulen, uprio);`
11. `}`
12. `let msg = Mqueue::pop_highest(info);`
13. `if let Some(sender) = info.wait[SEND].pop_first() { Mqueue::pipelined_receive(info, sender); }`
14. `info.lock.unlock();`
15. `return Mqueue::deliver(msg, ubuf, ulen, uprio);`

`Mqueue::deliver(msg, ubuf, ulen, uprio) -> isize`:

1. `let bytes = msg.payload_len;`
2. `copy_to_user(ubuf, msg.payload(), bytes)?;`
3. `if !uprio.is_null() { copy_to_user(uprio, &msg.prio, sizeof(u32))?; }`
4. `let len = bytes as isize;`
5. `MsgMsg::free(msg);`
6. `return len;`

### Out of Scope

- `mq_timedsend(2)` — separate Tier-5 (`mq_timedsend.md`).
- `mq_notify(2)` / `mq_getsetattr(2)` — separate Tier-5.
- `mq_open(2)` / `mq_unlink(2)` — separate Tier-5.
- POSIX RT signal delivery — covered in `kernel/signal.md` Tier-3.
- High-resolution timer subsystem — covered in `kernel/hrtimer.md` Tier-3.
- Implementation code.

### signature

C (POSIX / man-pages):

```c
ssize_t mq_timedreceive(mqd_t mqdes, char *msg_ptr, size_t msg_len,
                        unsigned int *msg_prio,
                        const struct timespec *abs_timeout);
```

glibc wrapper: `__mq_timedreceive` → `INLINE_SYSCALL(mq_timedreceive, 5, mqdes, msg_ptr, msg_len, msg_prio, abs_timeout)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(mq_timedreceive,
                mqd_t,                          mqdes,
                char __user *,                  u_msg_ptr,
                size_t,                         msg_len,
                unsigned int __user *,          u_msg_prio,
                const struct __kernel_timespec __user *, u_abs_timeout);
```

Rookery dispatch:

```rust
pub fn sys_mq_timedreceive(
    mqdes: i32,
    msg_ptr: UserPtr<u8>,
    msg_len: usize,
    msg_prio: UserPtr<u32>,
    abs_timeout: UserPtr<Timespec>,
) -> SyscallResult<isize>;
```

### parameters

| name        | type                                    | constraints                                                                                       | errno-on-bad           |
|-------------|-----------------------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| mqdes       | `mqd_t` (`int`)                         | Open mqueue fd with read access (`O_RDONLY`/`O_RDWR`).                                            | `EBADF`/`EBADMSG`      |
| msg_ptr     | `char __user *`                         | Writable for `msg_len` bytes.                                                                     | `EFAULT`               |
| msg_len     | `size_t`                                | `msg_len ≥ queue->attr.mq_msgsize`.                                                                | `EMSGSIZE`             |
| msg_prio    | `unsigned int __user *`                 | May be `NULL`; if non-NULL, writable for `sizeof(unsigned int)`.                                  | `EFAULT`               |
| abs_timeout | `const struct __kernel_timespec __user *` | If non-NULL, valid absolute `CLOCK_REALTIME` time: `tv_nsec ∈ [0, 1_000_000_000)`.                | `EINVAL`/`EFAULT`      |

### return value

- Success: number of bytes copied to `msg_ptr` (the message's actual length, `0 ≤ ret ≤ mq_msgsize`).
- Failure: `-1` with `errno`; in-kernel: a negated errno.

### errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EBADF`        | `mqdes` is not a valid descriptor, or not opened for read.                                      |
| `EBADMSG`      | `mqdes` refers to a non-mqueue file.                                                            |
| `EFAULT`       | `msg_ptr`, `msg_prio` (if non-NULL), or `abs_timeout` crosses the user/kernel boundary illegally. |
| `EMSGSIZE`     | `msg_len < queue->attr.mq_msgsize`.                                                              |
| `EINVAL`       | `abs_timeout->tv_nsec` out of range, or `tv_sec < 0`.                                            |
| `EAGAIN`       | Queue empty and `O_NONBLOCK` set on `mqdes`.                                                    |
| `ETIMEDOUT`    | Blocking receive and `abs_timeout` passed before a message arrived.                              |
| `EINTR`        | Signal interrupted the wait; no message consumed.                                                |

### abi surface (constants + flags)

From `include/uapi/linux/mqueue.h`:

- `MQ_PRIO_MAX = 32768` — receiver returns priorities in `[0, MQ_PRIO_MAX)`.
- `O_NONBLOCK` — checked from `mqdes->f_flags`; flips receive to non-blocking.

`abs_timeout` clock domain: **`CLOCK_REALTIME`** (per POSIX). `NULL` indicates infinite wait.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=mqdes`, `%rsi=msg_ptr`, `%rdx=msg_len`, `%r10=msg_prio`, `%r8=abs_timeout`.
- REQ-2: `fget(mqdes)`; `f->f_op != &mqueue_file_operations ⟹ -EBADMSG`. `(f->f_mode & FMODE_READ) == 0 ⟹ -EBADF`.
- REQ-3: `msg_len < info->attr.mq_msgsize ⟹ -EMSGSIZE`.
- REQ-4: `abs_timeout` (if non-NULL) copied via `get_timespec64` and validated; `tv_nsec ∉ [0, NSEC_PER_SEC)` or `tv_sec < 0` ⟹ `-EINVAL`.
- REQ-5: Acquires `info->lock` before queue inspection.
- REQ-6: If `info->attr.mq_curmsgs > 0`: pick highest-priority list head; unlink message; decrement `mq_curmsgs`.
- REQ-7: If a sender is blocked on `info->wait[SEND]`, *pipelined-receive*: pop sender waiter, take its message, requeue sender's slot, wake sender. Sender wakes with `0` return.
- REQ-8: If `info->attr.mq_curmsgs == 0`:
  - If `O_NONBLOCK` set: `-EAGAIN`.
  - Else: insert wait-entry on `info->wait[RECV]`, drop lock, `schedule_hrtimeout_range_clock(abs_timeout, ..., CLOCK_REALTIME)`.
  - On wakeup, re-acquire lock; on timeout ⟹ `-ETIMEDOUT`; on signal ⟹ `-EINTR`; else proceed with the message planted by sender's `pipelined_send`.
- REQ-9: Copy payload to `msg_ptr` (size = `msg->m_ts`); if `msg_prio != NULL`, write priority.
- REQ-10: Free `msg_msg` slab object after copy-out.
- REQ-11: Return value = bytes copied (`msg->m_ts`).
- REQ-12: Audit hook: optional record on receive (POSIX_MQ_RECV).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `buf_size_ge_msgsize` | INVARIANT | `msg_len ≥ mq_msgsize`. |
| `timespec_valid` | INVARIANT | `tv_nsec ∈ [0, NSEC_PER_SEC)`, `tv_sec ≥ 0`. |
| `priority_total_order_dequeue` | INVARIANT | Receiver dequeues highest priority first; FIFO within prio. |
| `no_double_free_msg` | INVARIANT | Each `msg_msg` freed exactly once (either after delivery or sender free on `-EINTR`). |
| `pipelined_receive_advances_sender` | INVARIANT | Pipelined receive moves exactly one sender from `wait[SEND]` to enqueued. |
| `prio_write_only_when_non_null` | INVARIANT | `msg_prio` written iff non-NULL. |

### Layer 2: TLA+

`uapi/syscalls/mq_timedreceive.tla`:
- Per-call → validate → fget → lock → consume-or-wait → deliver.
- Properties:
  - `safety_priority_dequeue_total_order`,
  - `safety_no_lost_message_on_interrupt`,
  - `safety_pipelined_receive_at_most_one_sender_wake`,
  - `liveness_under_sender_progress_recv_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ returns `m_ts` bytes; `mq_curmsgs` decreased by 1 OR sender pipelined slot opened | `Mqueue::do_receive` |
| Post: `-EAGAIN` ⟹ queue state unchanged | `Mqueue::do_receive` |
| Post: `-ETIMEDOUT` ⟹ receiver removed from waitqueue, no message consumed | `Mqueue::do_receive` |
| Post: `-EINTR` ⟹ no copy_to_user performed; no slab free | `Mqueue::do_receive` |

### Layer 4: Verus/Creusot functional

`mq_timedreceive(mqdes, msg_ptr, msg_len, msg_prio, abs_timeout)` ≡ POSIX.1-2017 `mq_timedreceive` per `man 3 mq_timedreceive`; priority dequeue + `CLOCK_REALTIME` deadline + pipelined receive.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mq_timedreceive(2)` reinforcement:

- **Per-`msg_len ≥ mq_msgsize` requirement** — defense against truncated copy / partial message loss.
- **Per-`abs_timeout` validation pre-block** — defense against malicious-deadline DoS.
- **Per-`O_NONBLOCK` strict separation** — defense against blocking RT-task starvation.
- **Per-pipelined-receive atomic transfer** — defense against double-consume.
- **Per-`f_op` identity check** — defense against fd-confusion.
- **Per-`info->lock` strict scope** — defense against TOCTOU between empty-check and dequeue.
- **Per-`-EINTR`/`-ETIMEDOUT` no-consume** — defense against message loss on cancellation.
- **Per-`copy_to_user` post-dequeue strict ordering** — defense against torn copy mid-deliver.

### grsecurity/pax-style reinforcement

- **PAX_UDEREF** — `msg_ptr`, `msg_prio`, `abs_timeout` SMAP-guarded; `copy_to_user` bounded by `m_ts`.
- **GRKERNSEC_CHROOT_MQUEUE** — chrooted task can only receive from queues created in its IPC namespace and chroot-visible mqueue mount.
- **GRKERNSEC_RLIMIT** — receive does not return RLIMIT_MSGQUEUE bytes until slab is actually freed, preventing accounting underflow.
- **GRKERNSEC_AUDIT_IPC** — every `mq_timedreceive` (success/fail) logged with mqdes/priority/length/uid.
- **GRKERNSEC_HIDESYM** — `-EBADMSG`/`-EBADF` printks redact kernel pointers.
- **PAX_REFCOUNT** — `file` and `mqueue_inode_info` refcounts saturating.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — `msg_msg` payload zeroed on free after copy-out (defense against later slab reuse leak).
- **GRKERNSEC_TIME** — `abs_timeout` against `CLOCK_REALTIME` jumps mitigated; receiver wake re-checked on `clock_was_set`.
- **GRKERNSEC_RESLOG** — pipelined-receive sender uncharge logged for forensic balance.
- **GRKERNSEC_PROC_IPC** — `/proc/<pid>/fdinfo/<fd>` for mqueue redacted.

