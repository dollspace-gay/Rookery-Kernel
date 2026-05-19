# Tier-5: syscall 244 — mq_notify(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`244  common  mq_notify  sys_mq_notify`)
  - ipc/mqueue.c (`SYSCALL_DEFINE2(mq_notify, ...)`, `do_mq_notify`, `remove_notification`)
  - include/uapi/asm-generic/siginfo.h (`union sigval`, `struct sigevent`)
  - kernel/signal.c (`sigqueue_alloc`, `send_sigqueue`)
  - net/netlink/af_netlink.c (`SIGEV_THREAD` userland helper)
-->

## Summary

`mq_notify(2)` is **x86_64 syscall 244**, the POSIX message-queue async-notification registrar. It associates the calling process with the queue referenced by `mqdes` so that the next message arriving on a previously-empty queue triggers a one-shot notification. The notification kind is selected by `sevp->sigev_notify`:

- `SIGEV_NONE` — no notification (used to clear).
- `SIGEV_SIGNAL` — kernel sends real-time signal `sevp->sigev_signo` with `sigval = sevp->sigev_value` (queued via `sigqueue`).
- `SIGEV_THREAD` — kernel routes notification through a netlink socket `sevp->sigev_value.sival_int` (an fd) so a glibc helper can wake a worker thread in userspace.

Notification is **one-shot** and **edge-triggered**: it fires exactly once on a 0→1 transition of `mq_curmsgs` while no receiver is blocked. After firing, the registration is cleared; the process must call `mq_notify` again to re-arm. Only one process may be registered at a time per queue; subsequent registrations from another process while one is active return `-EBUSY`. Passing `sevp == NULL` unregisters the caller's notification.

Critical for: every POSIX async-IO-on-queue pattern, every libc `mq_notify(SIGEV_THREAD)` user-thread wakeup, every Rookery test for the one-shot edge-trigger semantics and `-EBUSY` contention.

## Signature

C (POSIX / man-pages):

```c
int mq_notify(mqd_t mqdes, const struct sigevent *sevp);
```

glibc wrapper: `mq_notify` (handles `SIGEV_THREAD` via internal `netlink` helper) → `INLINE_SYSCALL(mq_notify, 2, mqdes, sevp)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(mq_notify,
                mqd_t,                       mqdes,
                const struct sigevent __user *, u_notification);
```

Rookery dispatch:

```rust
pub fn sys_mq_notify(
    mqdes: i32,
    notification: UserPtr<Sigevent>,
) -> SyscallResult<i32>;
```

## Parameters

| name         | type                          | constraints                                                                                       | errno-on-bad           |
|--------------|-------------------------------|---------------------------------------------------------------------------------------------------|------------------------|
| mqdes        | `mqd_t` (`int`)               | Open mqueue fd.                                                                                   | `EBADF`/`EBADMSG`      |
| notification | `const struct sigevent *`     | Either `NULL` (clear) or readable for `sizeof(struct sigevent)`; `sigev_notify ∈ {SIGEV_NONE, SIGEV_SIGNAL, SIGEV_THREAD}`. | `EFAULT`/`EINVAL` |

`struct sigevent` layout (UAPI, simplified):

```c
struct sigevent {
    sigval_t sigev_value;        /* opaque value passed to handler/thread */
    int      sigev_signo;        /* signal number for SIGEV_SIGNAL */
    int      sigev_notify;       /* SIGEV_NONE | SIGEV_SIGNAL | SIGEV_THREAD */
    union { ... } _sigev_un;     /* SIGEV_THREAD attributes; opaque to kernel */
};
```

## Return value

- Success: `0`.
- Failure: `-1` with `errno`; in-kernel: a negated errno.

## Errors

| errno          | condition                                                                                       |
|----------------|-------------------------------------------------------------------------------------------------|
| `EBADF`        | `mqdes` is not a valid descriptor.                                                              |
| `EBADMSG`      | `mqdes` refers to a non-mqueue file.                                                            |
| `EFAULT`       | `notification` is non-NULL and crosses the user/kernel boundary illegally.                      |
| `EINVAL`       | `sigev_notify` not one of `SIGEV_NONE/SIGEV_SIGNAL/SIGEV_THREAD`; or `SIGEV_SIGNAL` with out-of-range `sigev_signo`. |
| `EBUSY`        | Another process is already registered for notification on this queue.                            |
| `ENOMEM`       | Kernel could not allocate `sigqueue` / notification slot.                                       |

## ABI surface (constants + flags)

From `include/uapi/asm-generic/siginfo.h`:

- `SIGEV_SIGNAL = 0` — send a signal.
- `SIGEV_NONE = 1` — no notification (register cleared).
- `SIGEV_THREAD = 2` — kernel writes a netlink message on `sigev_value.sival_int` fd; userland glibc helper consumes and wakes thread.
- `SIGEV_THREAD_ID = 4` — Linux extension; deliver to specific tid (not used by `mq_notify`'s POSIX surface but possible).
- `SIGRTMIN ≤ sigev_signo ≤ SIGRTMAX` recommended (queued semantics); `SIGRTMIN..._NSIG-1` valid.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=mqdes`, `%rsi=notification`.
- REQ-2: `notification == NULL` ⟹ unregister: if `info->notify_owner == current->nsproxy->pid_ns`, clear registration; else (nobody registered for caller) silently `0`.
- REQ-3: `notification != NULL` ⟹ `copy_from_user(&sigev, notification, sizeof(sigevent))?`.
- REQ-4: `sigev.sigev_notify ∉ {SIGEV_NONE, SIGEV_SIGNAL, SIGEV_THREAD} ⟹ -EINVAL`.
- REQ-5: `SIGEV_SIGNAL` requires `sigev.sigev_signo ∈ [1, _NSIG)`; else `-EINVAL`.
- REQ-6: `SIGEV_THREAD` requires `sigev.sigev_value.sival_int` to be a valid netlink fd (`AF_NETLINK`, `NETLINK_USERSOCK`); kernel sends a one-shot netlink message on first message arrival.
- REQ-7: `fget(mqdes)`; if non-mqueue ⟹ `-EBADMSG`.
- REQ-8: Acquire `info->lock`; if `info->notify_owner` is set and not the calling process ⟹ `-EBUSY` (allow self-replace).
- REQ-9: For `SIGEV_SIGNAL`: pre-allocate `sigqueue` via `sigqueue_alloc()` so delivery cannot fail at notify-time; store on `info->notify_cookie`.
- REQ-10: For `SIGEV_THREAD`: pin netlink socket via `netlink_attachskb`; store on `info->notify_sock`.
- REQ-11: Store `info->notify = sigev`, `info->notify_owner = task_active_pid_ns(current)`, `info->notify_user_ns = current_user_ns()`.
- REQ-12: On first message arrival to previously-empty queue (no receiver pipelined): kernel calls `__do_notify(info)` which:
  - For `SIGEV_SIGNAL`: `send_sigqueue(info->notify_cookie, info->notify_owner, 0)`.
  - For `SIGEV_THREAD`: `netlink_sendskb(info->notify_sock, skb)`.
  - Clears `info->notify_owner` and frees `notify_cookie`/`notify_sock`.
- REQ-13: One-shot: registration cleared after delivery; caller must re-register.
- REQ-14: Process exit auto-unregisters (`remove_notification` in `mqueue_release`).

## Acceptance Criteria

- [ ] AC-1: `mq_notify(fd, &sevp_signal)` then send to empty queue ⟹ caller receives `sevp_signal.sigev_signo` with `si_value = sevp_signal.sigev_value`.
- [ ] AC-2: After delivery, registration cleared; second send does not deliver again until re-arm.
- [ ] AC-3: Two processes contending: second `mq_notify` ⟹ `-EBUSY`.
- [ ] AC-4: Same process replacing own registration is allowed (no `-EBUSY`).
- [ ] AC-5: `mq_notify(fd, NULL)` while registered ⟹ unregister; `0` returned.
- [ ] AC-6: `sigev_notify = 99` ⟹ `-EINVAL`.
- [ ] AC-7: `SIGEV_SIGNAL` with `sigev_signo = 0` or `≥ _NSIG` ⟹ `-EINVAL`.
- [ ] AC-8: `SIGEV_THREAD` with non-netlink fd ⟹ `-EINVAL`/`-EBADF`.
- [ ] AC-9: Non-mqueue fd ⟹ `-EBADMSG`.
- [ ] AC-10: Notification fires only on 0→1 transition (sends into non-empty queue do not re-trigger).
- [ ] AC-11: Receiver blocked in `mq_timedreceive` ⟹ pipelined delivery; **no** notification fired.
- [ ] AC-12: Caller exit auto-unregisters: subsequent `mq_notify` from another process succeeds.

## Architecture

```
struct MqNotifyArgs {
    mqdes: i32,
    notification: UserPtr<Sigevent>,
}
```

`Mqueue::sys_mq_notify(args) -> i32`:

1. `let file = current().files.fget(args.mqdes)?;`
2. `if !Mqueue::is_mqueue_file(&file) { return -EBADMSG; }`
3. `let info = mqueue_inode_info(file.inode);`
4. `if args.notification.is_null() { return Mqueue::clear_notify(info); }`
5. `let sigev = Sigevent::copy_from_user(args.notification)?;`
6. `match sigev.sigev_notify { SIGEV_NONE | SIGEV_SIGNAL | SIGEV_THREAD => (), _ => return -EINVAL }`
7. `Mqueue::register_notify(info, sigev)`

`Mqueue::register_notify(info, sigev) -> i32`:

1. /* Validate per-kind */
2. `match sigev.sigev_notify {`
3. `    SIGEV_SIGNAL => if sigev.sigev_signo < 1 || sigev.sigev_signo >= _NSIG { return -EINVAL; },`
4. `    SIGEV_THREAD => Mqueue::prepare_netlink(sigev)?,`
5. `    SIGEV_NONE   => (),`
6. `}`
7. `let cookie = if sigev.sigev_notify == SIGEV_SIGNAL { Some(SigQueue::alloc()?) } else { None };`
8. `info.lock.lock();`
9. `if let Some(owner) = info.notify_owner { if owner != current_pid_ns() { info.lock.unlock(); SigQueue::free(cookie); return -EBUSY; } }`
10. `Mqueue::clear_notify_locked(info); /* idempotent replace */`
11. `info.notify = sigev;`
12. `info.notify_owner = current_pid_ns();`
13. `info.notify_user_ns = current_user_ns();`
14. `info.notify_cookie = cookie;`
15. `info.lock.unlock();`
16. `return 0;`

`Mqueue::do_notify(info)` (called from `mq_timedsend` on 0→1):

1. `if info.notify_owner.is_none() { return; }`
2. `match info.notify.sigev_notify {`
3. `    SIGEV_SIGNAL => SigQueue::send(info.notify_cookie.take().unwrap(), info.notify_owner.take().unwrap(), info.notify.sigev_signo, info.notify.sigev_value),`
4. `    SIGEV_THREAD => Netlink::send_one_shot(info.notify_sock.take().unwrap()),`
5. `    SIGEV_NONE   => (),`
6. `}`
7. `info.notify_owner = None;`

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `notify_kind_validated` | INVARIANT | `sigev_notify ∈ {NONE, SIGNAL, THREAD}` else `-EINVAL`. |
| `signal_range_validated` | INVARIANT | `SIGEV_SIGNAL` ⟹ `1 ≤ signo < _NSIG`. |
| `single_owner_per_queue` | INVARIANT | At most one `notify_owner` set per queue; cross-process replace ⟹ `-EBUSY`. |
| `one_shot_strict` | INVARIANT | `notify_owner` cleared on delivery. |
| `cookie_lifecycle_balanced` | INVARIANT | Every `SigQueue::alloc` matched by `SigQueue::send` or `SigQueue::free`. |
| `edge_trigger` | INVARIANT | Notification fires only on 0→1 `mq_curmsgs` transition. |

### Layer 2: TLA+

`uapi/syscalls/mq_notify.tla`:
- Per-call → validate → register-or-clear → edge-fire-on-send.
- Properties:
  - `safety_at_most_one_active_registration`,
  - `safety_one_shot_clears_after_fire`,
  - `safety_no_fire_when_receiver_pipelined`,
  - `liveness_under_send_progress_notify_fires`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| Post: success ⟹ `notify_owner = caller_pid_ns` and kind matches sigev | `Mqueue::register_notify` |
| Post: `-EBUSY` ⟹ existing registration untouched | `Mqueue::register_notify` |
| Post: caller exit ⟹ `notify_owner` cleared if it was caller | `Mqueue::release` |
| Post: edge fire ⟹ `notify_owner` cleared and cookie consumed | `Mqueue::do_notify` |

### Layer 4: Verus/Creusot functional

`mq_notify(mqdes, sevp)` ≡ POSIX.1-2017 `mq_notify` per `man 3 mq_notify`; one-shot edge-triggered, single-owner, three notification kinds.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mq_notify(2)` reinforcement:

- **Per-`sigev_notify` whitelist** — defense against silent acceptance of unknown kinds.
- **Per-`sigev_signo` range check** — defense against bogus signal injection.
- **Per-single-owner enforcement** — defense against silent override by another process.
- **Per-one-shot guarantee** — defense against notification flooding.
- **Per-pre-allocated `sigqueue`** — defense against notify-time `-ENOMEM` (delivery cannot fail).
- **Per-`SIGEV_THREAD` netlink fd validation** — defense against fd-confusion.
- **Per-process-exit auto-unregister** — defense against stale registration leaking past exit.
- **Per-edge-trigger semantics** — defense against repeated wakeups on every send.

## Grsecurity/PaX-style Reinforcement

- **PAX_UDEREF** — `sigevent` SMAP-guarded copy; size strictly `sizeof(struct sigevent)`.
- **GRKERNSEC_CHROOT_MQUEUE** — chrooted task can only register notification on queues in its IPC namespace and chroot-visible mqueue mount.
- **GRKERNSEC_SIGNAL** — signal delivery from `mq_notify` honors `RLIMIT_SIGPENDING` of recipient real uid.
- **GRKERNSEC_AUDIT_IPC** — every `mq_notify` register/unregister/fire logged with mqdes/sigev_notify/signo/uid/exe.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_REFCOUNT** — `file`, `mqueue_inode_info`, and `sigqueue` refcounts saturating.
- **PAX_RANDKSTACK** — kstack offset randomized at syscall entry.
- **PAX_MEMORY_SANITIZE** — `sigqueue` payload zeroed on free (defense against leak of `sival_ptr` to subsequent allocations).
- **GRKERNSEC_PROC_IPC** — `/proc/<pid>/fdinfo/<fd>` for mqueue notification fd redacted.
- **GRKERNSEC_NETLINK** — `SIGEV_THREAD` netlink fd validated to belong to same pid_namespace; cross-ns netlink notification denied.
- **GRKERNSEC_RESLOG** — `-EBUSY` contention logged with both contender uids for forensic correlation.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `mq_timedsend(2)` / `mq_timedreceive(2)` — separate Tier-5.
- `mq_getsetattr(2)` — separate Tier-5.
- `mq_open(2)` / `mq_unlink(2)` — separate Tier-5.
- RT-signal delivery (`sigqueue`) — covered in `kernel/signal.md` Tier-3.
- Netlink delivery internals — covered in `net/netlink.md` Tier-3.
- Implementation code.
