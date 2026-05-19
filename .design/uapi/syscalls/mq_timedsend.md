# Tier-5: syscall 242 — mq_timedsend(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`242  common  mq_timedsend  sys_mq_timedsend`)
  - ipc/mqueue.c (`SYSCALL_DEFINE5(mq_timedsend, ...)`, `do_mq_timedsend`, `pipelined_send`, `wq_add`)
  - include/uapi/linux/mqueue.h (`MQ_PRIO_MAX`)
  - include/linux/time64.h (`struct __kernel_timespec`)
  - kernel/sysctl.c (`fs.mqueue.msg_max`, `fs.mqueue.msgsize_max`)
-->

## Summary

`mq_timedsend(2)` is **x86_64 syscall 242**, the POSIX message-queue sender with a deadline. It places one priority-tagged message of length `msg_len` (bounded by the queue's `mq_msgsize`) into the queue referenced by `mqdes`. If the queue is full and `O_NONBLOCK` is not set, the call blocks the calling thread on the queue's `wait_send` waitqueue until space becomes available, the absolute deadline `abs_timeout` (CLOCK_REALTIME) elapses, or a signal arrives.

Messages are inserted in *strict descending priority order* (FIFO within equal priority) into the kernel-managed `msg_msg` ring backing the queue. On a successful send the kernel may *pipeline-deliver* the message to a thread already blocked in `mq_timedreceive` on the same queue, bypassing the ring; otherwise it links the message into the per-queue list and wakes any registered `mq_notify` recipient.

Critical for: every POSIX-RT message producer, every real-time IPC channel under deadline, every Rookery test for priority ordering, blocking semantics, signal cancellation, and `abs_timeout` precision.

## Signature

C (POSIX / man-pages):

```c
int mq_timedsend(mqd_t mqdes, const char *msg_ptr, size_t msg_len,
                 unsigned int msg_prio,
                 const struct timespec *abs_timeout);
```

glibc wrapper: `__mq_timedsend` → `INLINE_SYSCALL(mq_timedsend, 5, mqdes, msg_ptr, msg_len, msg_prio, abs_timeout)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE5(mq_timedsend,
                mqd_t,                          mqdes,
                const char __user *,            u_msg_ptr,
                size_t,                         msg_len,
                unsigned int,                   msg_prio,
                const struct __kernel_timespec __user *, u_abs_timeout);
```

Rookery dispatch:

```rust
pub fn sys_mq_timedsend(
    mqdes: i32,
    msg_ptr: UserPtr<u8>,
    msg_len: usize,
    msg_prio: u32,
    abs_timeout: UserPtr<Timespec>,
) -> SyscallResult<i32>;
```

## Parameters

| name        | type                                    | constraints                                                                                       | errno-on-bad           |
|-------------|-----------------------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| mqdes       | `mqd_t` (`int`)                         | Open mqueue fd with write access (`O_WRONLY`/`O_RDWR`).                                           | `EBADF`/`EBADMSG`      |
| msg_ptr     | `const char __user *`                   | Readable for `msg_len` bytes; may be `NULL` only if `msg_len == 0`.                               | `EFAULT`               |
| msg_len     | `size_t`                                | `0 ≤ msg_len ≤ queue->attr.mq_msgsize`.                                                            | `EMSGSIZE`             |
| msg_prio    | `unsigned int`                          | `0 ≤ msg_prio < MQ_PRIO_MAX (32768)`.                                                              | `EINVAL`               |
| abs_timeout | `const struct __kernel_timespec __user *` | If non-NULL, valid absolute `CLOCK_REALTIME` time: `tv_nsec ∈ [0, 1_000_000_000)`.                | `EINVAL`/`EFAULT`      |

## Return value

- Success: `0` (message enqueued or pipelined).
- Failure: `-1` with `errno`; in-kernel: a negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EBADF`        | `mqdes` is not a valid descriptor, or not opened for write.                                     |
| `EBADMSG`      | `mqdes` refers to a non-mqueue file.                                                            |
| `EFAULT`       | `msg_ptr` (for non-zero `msg_len`) or `abs_timeout` crosses the user/kernel boundary illegally. |
| `EMSGSIZE`     | `msg_len > queue->attr.mq_msgsize`.                                                              |
| `EINVAL`       | `msg_prio ≥ MQ_PRIO_MAX`, or `abs_timeout->tv_nsec` out of range, or `tv_sec < 0`.               |
| `EAGAIN`       | Queue full and `O_NONBLOCK` set on `mqdes`.                                                     |
| `ETIMEDOUT`    | Blocking send and `abs_timeout` passed before space was available.                              |
| `EINTR`        | Signal interrupted the wait; no partial message enqueued.                                       |

## ABI surface (constants + flags)

From `include/uapi/linux/mqueue.h`:

- `MQ_PRIO_MAX = 32768` — exclusive upper bound on `msg_prio`.
- `O_NONBLOCK` — checked from `mqdes->f_flags`; flips send to non-blocking.

`abs_timeout` clock domain: **`CLOCK_REALTIME`** (per POSIX). `NULL` indicates infinite wait.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=mqdes`, `%rsi=msg_ptr`, `%rdx=msg_len`, `%r10=msg_prio`, `%r8=abs_timeout`.
- REQ-2: `fget(mqdes)`; if `f->f_op != &mqueue_file_operations` ⟹ `-EBADMSG`. If `(f->f_mode & FMODE_WRITE) == 0` ⟹ `-EBADF`.
- REQ-3: `msg_prio ≥ MQ_PRIO_MAX ⟹ -EINVAL`.
- REQ-4: `msg_len > info->attr.mq_msgsize ⟹ -EMSGSIZE`.
- REQ-5: `abs_timeout` (if non-NULL) copied via `get_timespec64` and validated; `tv_nsec ∉ [0, NSEC_PER_SEC)` or `tv_sec < 0` ⟹ `-EINVAL`.
- REQ-6: `msg = load_msg(msg_ptr, msg_len)`; allocates a `struct msg_msg` + payload via slab; failure ⟹ `-ENOMEM`.
- REQ-7: Acquires `info->lock` (spinlock) before queue inspection.
- REQ-8: If a thread is blocked in `mq_timedreceive` on this queue, pipeline-deliver: copy message directly to receiver, wake receiver, return `0` without ring insertion.
- REQ-9: Otherwise, if `info->attr.mq_curmsgs == info->attr.mq_maxmsg`:
  - If `O_NONBLOCK` set on `mqdes`: `-EAGAIN`.
  - Else: insert wait-entry on `info->wait[SEND]`, drop lock, `schedule_hrtimeout_range_clock(abs_timeout, ..., CLOCK_REALTIME)`.
  - On wakeup, re-acquire lock and recheck; on timeout ⟹ `-ETIMEDOUT`; on signal ⟹ `-EINTR`.
- REQ-10: On enqueue success: insert `msg` into per-priority list (descending priority); `info->attr.mq_curmsgs++`.
- REQ-11: If `info->notify.sigev_notify != SIGEV_NONE` and queue was empty before insert and no receiver pipelined: deliver `mq_notify` (signal/thread) and clear notify.
- REQ-12: Lockstep ordering: per-priority FIFO; `pipelined_send` honors absolute receiver wakeup ordering.
- REQ-13: Audit hook: optional record on enqueue (POSIX_MQ_SEND).

## Acceptance Criteria

- [ ] AC-1: `mq_timedsend(fd, "x", 1, 0, NULL)` to non-full queue ⟹ `0`; `mq_getattr.mq_curmsgs` increments.
- [ ] AC-2: `msg_prio = MQ_PRIO_MAX` ⟹ `-EINVAL`.
- [ ] AC-3: `msg_len > mq_msgsize` ⟹ `-EMSGSIZE`.
- [ ] AC-4: Queue full + `O_NONBLOCK` ⟹ `-EAGAIN`.
- [ ] AC-5: Queue full + blocking + deadline past ⟹ `-ETIMEDOUT`.
- [ ] AC-6: Signal during block ⟹ `-EINTR`; no partial enqueue.
- [ ] AC-7: `mqdes` opened `O_RDONLY` ⟹ `-EBADF`.
- [ ] AC-8: Non-mqueue fd ⟹ `-EBADMSG`.
- [ ] AC-9: Concurrent sends with different priorities ⟹ receiver dequeues high-priority first; FIFO within equal priority.
- [ ] AC-10: Receiver already blocked ⟹ pipelined delivery; `mq_curmsgs` unchanged.
- [ ] AC-11: `mq_notify` registered + queue empty + send ⟹ notification delivered exactly once.
- [ ] AC-12: `abs_timeout->tv_nsec >= 1e9` ⟹ `-EINVAL` (validated before any allocation).

## Architecture

```
struct MqTimedsendArgs {
    mqdes: i32,
    msg_ptr: UserPtr<u8>,
    msg_len: usize,
    msg_prio: u32,
    abs_timeout: UserPtr<Timespec>,
}
```

`Mqueue::sys_mq_timedsend(args) -> i32`:

1. /* Eager validation */ `if args.msg_prio >= MQ_PRIO_MAX { return -EINVAL; }`
2. `let timeout = if !args.abs_timeout.is_null() { Some(Timespec::copy_from_user_validated(args.abs_timeout)?) } else { None };`
3. `let file = current().files.fget_write(args.mqdes)?;`
4. `if !Mqueue::is_mqueue_file(&file) { return -EBADMSG; }`
5. `let info = mqueue_inode_info(file.inode);`
6. `if args.msg_len > info.attr.mq_msgsize { return -EMSGSIZE; }`
7. `let msg = MsgMsg::load_from_user(args.msg_ptr, args.msg_len, args.msg_prio)?;`
8. `return Mqueue::do_send(&file, info, msg, timeout);`

`Mqueue::do_send(file, info, msg, timeout) -> i32`:

1. `let nonblock = (file.f_flags & O_NONBLOCK) != 0;`
2. `info.lock.lock();`
3. `if let Some(recv) = info.wait[RECV].pop_first() { Mqueue::pipelined_send(recv, msg); info.lock.unlock(); return 0; }`
4. `if info.attr.mq_curmsgs == info.attr.mq_maxmsg {`
5. `    if nonblock { info.lock.unlock(); MsgMsg::free(msg); return -EAGAIN; }`
6. `    let waiter = WaitEntry::for_send(current(), msg);`
7. `    info.wait[SEND].push(&waiter);`
8. `    info.lock.unlock();`
9. `    match schedule_hrtimeout_until(timeout, CLOCK_REALTIME) { TimedOut => return -ETIMEDOUT, Interrupted => return -EINTR, Resumed => () }`
10. `    /* Resumed: pipelined_send already moved message; success */ return 0;`
11. `}`
12. `Mqueue::insert_message(info, msg);`
13. `Mqueue::notify_if_was_empty(info);`
14. `info.lock.unlock();`
15. `return 0;`

`Mqueue::insert_message(info, msg)`:

1. `let head = &mut info.messages[msg.prio];`
2. `head.push_back(msg);`
3. `info.attr.mq_curmsgs += 1;`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prio_bounded` | INVARIANT | `msg_prio < MQ_PRIO_MAX`. |
| `msg_len_bounded` | INVARIANT | `msg_len ≤ mq_msgsize`. |
| `timespec_valid` | INVARIANT | `tv_nsec ∈ [0, NSEC_PER_SEC)`, `tv_sec ≥ 0`. |
| `pipelined_or_inserted_exactly_once` | INVARIANT | Each message either pipelined to receiver OR inserted; never both, never neither on success. |
| `priority_ordering_preserved` | INVARIANT | Higher-priority messages dequeued before lower; FIFO within prio. |
| `no_partial_enqueue_on_interrupt` | INVARIANT | `-EINTR`/`-ETIMEDOUT` ⟹ message freed, queue unchanged. |

### Layer 2: TLA+

`uapi/syscalls/mq_timedsend.tla`:
- Per-call → validate → fget → load_msg → lock → pipeline-or-insert-or-wait.
- Properties:
  - `safety_priority_total_order`,
  - `safety_no_lost_message_on_error`,
  - `safety_notify_at_most_once_per_send`,
  - `liveness_under_consumer_progress_send_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ `mq_curmsgs` increased by 1 OR receiver woken with message | `Mqueue::do_send` |
| Post: `-EAGAIN` ⟹ queue state unchanged | `Mqueue::do_send` |
| Post: `-ETIMEDOUT` ⟹ message freed, no waiter remaining | `Mqueue::do_send` |
| Post: `mq_notify` fired iff queue transitioned 0→1 messages and notify registered | `Mqueue::notify_if_was_empty` |

### Layer 4: Verus/Creusot functional

`mq_timedsend(mqdes, msg_ptr, msg_len, msg_prio, abs_timeout)` ≡ POSIX.1-2017 `mq_timedsend` per `man 3 mq_timedsend`; priority ordering + `CLOCK_REALTIME` deadline + pipelined delivery.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mq_timedsend(2)` reinforcement:

- **Per-`MQ_PRIO_MAX` strict bound** — defense against priority overflow / heap corruption.
- **Per-`mq_msgsize` cap** — defense against unbounded slab allocation.
- **Per-`abs_timeout` validation pre-allocation** — defense against malicious-deadline DoS.
- **Per-`O_NONBLOCK` strict separation** — defense against blocking RT-task deadlock.
- **Per-pipelined-send atomic transfer** — defense against double-delivery.
- **Per-`mq_notify` once-fire** — defense against notification flooding.
- **Per-`f_op` identity check** — defense against fd-confusion (sending to non-mqueue fd).
- **Per-`info->lock` strict scope** — defense against TOCTOU between `mq_curmsgs` read and write.
- **Per-`-EINTR`/`-ETIMEDOUT` message-free** — defense against slab leak on cancellation.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `msg_ptr` and `abs_timeout` SMAP-guarded; `copy_from_user` bounded by `msg_len` and `sizeof(timespec)`.
- **GRKERNSEC_CHROOT_MQUEUE** — chrooted task can only send to queues created in its IPC namespace and chroot-visible mqueue mount.
- **GRKERNSEC_RLIMIT** — message slab allocation accounted against `RLIMIT_MSGQUEUE` (cumulative); send fails with `-EAGAIN` if exceeded even with space in queue.
- **GRKERNSEC_AUDIT_IPC** — every `mq_timedsend` (success/fail) logged with mqdes/priority/length/uid for forensics.
- **GRKERNSEC_HIDESYM** — printks on `-EBADMSG`/`-EBADF` redact kernel pointers.
- **PAX_REFCOUNT** — `file` and `mqueue_inode_info` refcounts saturating.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — `msg_msg` payload zeroed on free, preventing leak of trailing payload bytes through slab reuse.
- **GRKERNSEC_RESLOG** — exceeded `RLIMIT_MSGQUEUE` due to send-allocation logged per real-uid.
- **GRKERNSEC_PROC_IPC** — `/proc/<pid>/fdinfo/<fd>` for mqueue redacted (no in-kernel addresses).
- **GRKERNSEC_TIME** — `abs_timeout` against `CLOCK_REALTIME` jumps mitigated; sender wake re-checked on `clock_was_set`.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `mq_timedreceive(2)` — separate Tier-5 (`mq_timedreceive.md`).
- `mq_notify(2)` / `mq_getsetattr(2)` — separate Tier-5.
- `mq_open(2)` / `mq_unlink(2)` — separate Tier-5.
- POSIX RT signal delivery — covered in `kernel/signal.md` Tier-3.
- High-resolution timer subsystem — covered in `kernel/hrtimer.md` Tier-3.
- Implementation code.
