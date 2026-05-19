# Tier-5 syscall: eventfd(2) — syscall 284

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/eventfd.c (SYSCALL_DEFINE1(eventfd))
  - fs/eventfd.c (SYSCALL_DEFINE2(eventfd2))
  - fs/eventfd.c (struct eventfd_ctx, eventfd_signal)
  - include/uapi/linux/eventfd.h
  - arch/x86/entry/syscalls/syscall_64.tbl (284 64 eventfd)
-->

## Summary

`eventfd(2)` is the **legacy** single-argument creator for an eventfd file descriptor — a kernel-managed 64-bit counter that can be incremented (write) and observed/consumed (read) as a userspace synchronization primitive. The legacy variant lacks a `flags` argument: the resulting fd has neither `O_CLOEXEC` nor `O_NONBLOCK` (eventfd2 introduces both). Like `epoll_create(2)`/`signalfd(2)`, this is part of the legacy-fd hazard class motivating uniform `*2` variants. Used as: lightweight cross-thread/cross-process wakeup (KVM ioeventfd, io_uring CQE wakeup, container init signal forwarding, async runtime task-wakers, sd-event userspace wakers, dbus pipe replacement).

The counter semantics: `write(fd, &u64, 8)` increments counter by the supplied value (saturating at `UINT64_MAX - 1`); `read(fd, &u64, 8)` returns the counter and zeros it (or, in semaphore mode, returns 1 and decrements). Blocking read sleeps until counter > 0.

Critical for: KVM `kvm_irqfd` / `kvm_ioeventfd`, io_uring `IORING_REGISTER_EVENTFD`, every event-loop "wake-me-up" cross-thread primitive, async-runtime task notifications (tokio mio, async-std), GPU command submission completion signaling, container PID-1 shutdown coordination.

## Signature

```c
int eventfd(unsigned int initval, int flags);   /* glibc(3) — wraps eventfd2(2) */
int eventfd(unsigned int initval);              /* raw syscall 284 — legacy */
```

Note: The raw legacy syscall takes ONE argument; glibc's `eventfd(3)` is a TWO-arg wrapper calling `eventfd2(2)` (syscall 290). This doc covers the raw single-arg syscall 284.

## Parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `initval` | `unsigned int` | in | Initial counter value (kernel-extended to `u64`). |

## Return value

| Value | Meaning |
|---|---|
| ≥ 0 | New eventfd file descriptor. |
| `-1` | Error; `errno` set. |

## Errors

| `errno` | Cause |
|---|---|
| `EMFILE` | Per-process fd limit reached. |
| `ENFILE` | System-wide fd table exhausted. |
| `ENOMEM` | Kernel could not allocate `eventfd_ctx`. |

## ABI surface

```text
__NR_eventfd  (x86_64) = 284
__NR_eventfd  (i386)   = 323
__NR_eventfd  (arm64)  = NOT IMPLEMENTED   (arm64 only provides eventfd2)
__NR_eventfd  (generic)= NOT IMPLEMENTED

/* eventfd2 flags (not available via legacy eventfd): */
#define EFD_CLOEXEC     O_CLOEXEC      /* 0x080000 */
#define EFD_NONBLOCK    O_NONBLOCK     /* 0x000800 */
#define EFD_SEMAPHORE   00000001       /* 1 — read decrements by 1, not zero */

/* Internal counter limit (saturation): */
#define ULLONG_MAX_MINUS_ONE   0xFFFFFFFFFFFFFFFEULL

struct eventfd_ctx {
    refcount_t   kref;
    wait_queue_head_t wqh;
    __u64        count;
    unsigned int flags;          /* EFD_SEMAPHORE | EFD_NONBLOCK | EFD_CLOEXEC */
    int          id;             /* for fdinfo display */
};
```

## Compatibility contract

REQ-1: Syscall number is **284** on x86_64; **323** on i386; **NOT exposed** on arm64/generic (modern arches only provide `eventfd2`). Kernel uses `__ARCH_WANT_SYS_EVENTFD` to compile-in legacy entry.

REQ-2: Counter is 64-bit unsigned (`__u64`), initialized to `initval` (zero-extended from `unsigned int`).

REQ-3: Returned fd has flags `0`: NO `O_CLOEXEC`, NO `O_NONBLOCK`, NO `EFD_SEMAPHORE`. Survives `execve` unless `FD_CLOEXEC` is later set.

REQ-4: `read(fd, buf, len)`: `len` MUST be ≥ 8 bytes; otherwise `EINVAL`. Reads exactly 8 bytes in host byte-order. Default mode: returns the counter, then atomically zeros it. Semaphore mode: returns `1`, decrements counter by 1.

REQ-5: Blocking read with counter == 0: sleeps on `wqh` waitqueue until `write` is invoked or signal arrives. `EINTR` on signal. `O_NONBLOCK`: returns `EAGAIN`.

REQ-6: `write(fd, buf, len)`: `len` MUST be ≥ 8 bytes (writes exactly 8). Adds the supplied `u64` to counter, saturating at `ULLONG_MAX - 1` (kernel reserves `~0ULL` as overflow sentinel). Write of `0xFFFFFFFFFFFFFFFF` returns `EINVAL`. Successful write wakes all readers.

REQ-7: Blocking write that would exceed `ULLONG_MAX - 1`: sleeps until reader drains. `O_NONBLOCK`: returns `EAGAIN`.

REQ-8: `poll(2)`/`epoll(2)`/`select(2)`: `POLLIN`/`EPOLLIN` asserted when counter > 0; `POLLOUT`/`EPOLLOUT` asserted when counter < `ULLONG_MAX - 1`.

REQ-9: `dup(2)`/`dup2(2)`/`fcntl(F_DUPFD)`: dup'd fd shares the same `eventfd_ctx` (and counter). Multiple writers/readers cooperate on one counter.

REQ-10: Returned fd refers to an anonymous inode "[eventfd]". `lseek(2)` returns `ESPIPE`.

REQ-11: Per-process accounting: eventfd is not per-uid quota-limited in upstream; bounded only by fd table limits.

REQ-12: PaX UDEREF: read/write buffer pointers pre-validated. Not directly applicable to creator syscall.

REQ-13: LSM hook `security_file_alloc` fires on anon_inode creation.

REQ-14: Kernel-internal users (`eventfd_signal()`, `eventfd_ctx_fdget()`) hold an in-kernel reference; userspace close releases the userspace half but kernel-held ref keeps `ctx` alive until kernel releases it.

REQ-15: `seccomp` filters see `initval`; SHOULD be unrestricted but MAY block in favor of `eventfd2`.

REQ-16: Closing the last reference (kernel + userspace) frees `eventfd_ctx` and wakes any pending writers/readers with the file-close event.

## Acceptance Criteria

- [ ] AC-1: `eventfd(0)` returns a valid fd with initial counter == 0.
- [ ] AC-2: `eventfd(5)`: first read returns 5, counter then 0.
- [ ] AC-3: Returned fd has `FD_CLOEXEC == 0` (verified via `fcntl(F_GETFD)`).
- [ ] AC-4: After `write(fd, &v, 8)` with `v = 10`, counter == 10.
- [ ] AC-5: Read of size < 8 returns `EINVAL`.
- [ ] AC-6: Write of size < 8 returns `EINVAL`.
- [ ] AC-7: Write of `0xFFFFFFFFFFFFFFFF` returns `EINVAL`.
- [ ] AC-8: Blocking read on counter==0 sleeps until write or signal.
- [ ] AC-9: Non-blocking read on counter==0 returns `EAGAIN` only if `O_NONBLOCK` set (legacy eventfd: cannot be set at create; user must `fcntl`).
- [ ] AC-10: `epoll_wait` on eventfd with counter > 0 returns `EPOLLIN`.
- [ ] AC-11: Multiple writers' increments accumulate atomically.
- [ ] AC-12: After `execve` without `FD_CLOEXEC`, eventfd survives (legacy semantics).

## Architecture

```rust
#[syscall(nr = 284, abi = "sysv")]
pub fn sys_eventfd(initval: u32) -> KResult<i32> {
    /* Legacy: forward to eventfd2 with flags = 0. */
    EventFd::do_eventfd2(initval, 0)
}
```

`EventFd::do_eventfd2(initval, flags) -> KResult<i32>`:
1. /* Validate flags */
2. if flags & !(EFD_CLOEXEC | EFD_NONBLOCK | EFD_SEMAPHORE) != 0 { return Err(EINVAL); }
3. /* Allocate context */
4. let ctx = Arc::new(EventfdCtx {
   - kref: AtomicU32::new(1),
   - wqh: WaitQueueHead::new(),
   - count: AtomicU64::new(initval as u64),
   - flags: AtomicU32::new(flags),
   - id: alloc_eventfd_id(),
5. });
6. /* Anon inode */
7. let file_flags = O_RDWR | (flags & EFD_NONBLOCK);
8. let file = anon_inode_getfile("[eventfd]", &EVENTFD_FOPS, ctx, file_flags)?;
9. let fd_flags = if flags & EFD_CLOEXEC != 0 { O_CLOEXEC } else { 0 };
10. let fd = get_unused_fd_flags(fd_flags)?;
11. fd_install(fd, file);
12. Ok(fd)

`EventfdCtx::read(buf: &mut [u8], file_flags: u32) -> KResult<usize>`:
1. if buf.len() < 8 { return Err(EINVAL); }
2. loop {
   - let cur = self.count.load(Ordering::Acquire);
   - if cur > 0 {
     - let val = if self.flags.load(Ordering::Relaxed) & EFD_SEMAPHORE != 0 {
       - self.count.fetch_sub(1, Ordering::Release);
       - 1u64
     - } else {
       - self.count.store(0, Ordering::Release);
       - cur
     - };
     - self.wqh.wake_all();   /* wake blocked writers */
     - let bytes = val.to_ne_bytes();
     - copy_to_user_slice(buf, &bytes)?;
     - return Ok(8);
   - }
   - if file_flags & O_NONBLOCK != 0 { return Err(EAGAIN); }
   - wait_event_interruptible(self.wqh, self.count.load(Ordering::Acquire) > 0)?;
3. }

`EventfdCtx::write(buf: &[u8], file_flags: u32) -> KResult<usize>`:
1. if buf.len() < 8 { return Err(EINVAL); }
2. let mut bytes = [0u8; 8];
3. copy_from_user_slice(&mut bytes, &buf[..8])?;
4. let add = u64::from_ne_bytes(bytes);
5. if add == ULLONG_MAX { return Err(EINVAL); }
6. loop {
   - let cur = self.count.load(Ordering::Acquire);
   - if let Some(new) = cur.checked_add(add) {
     - if new > ULLONG_MAX - 1 { /* would saturate */ } else {
       - match self.count.compare_exchange(cur, new, Ordering::Release, Ordering::Acquire) {
         - Ok(_) => { self.wqh.wake_all(); return Ok(8); }
         - Err(_) => continue,
       - }
     - }
   - }
   - /* Cannot add without overflow */
   - if file_flags & O_NONBLOCK != 0 { return Err(EAGAIN); }
   - wait_event_interruptible(self.wqh, self.count.load(Ordering::Acquire) <= ULLONG_MAX - 1 - add)?;
7. }

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `read_size_min_8` | INVARIANT | per-read: len ≥ 8 mandatory. |
| `write_size_min_8` | INVARIANT | per-write: len ≥ 8 mandatory. |
| `counter_no_overflow` | INVARIANT | per-write: counter ≤ ULLONG_MAX - 1. |
| `write_max_rejected` | INVARIANT | per-write: value 0xFFFFFFFFFFFFFFFF returns EINVAL. |
| `semaphore_decrement_one` | INVARIANT | per-EFD_SEMAPHORE read: counter -= 1, returns 1. |
| `legacy_no_cloexec` | INVARIANT | per-legacy: returned fd has FD_CLOEXEC unset. |
| `refcount_balanced` | INVARIANT | per-create/close: ctx ref balanced. |

### Layer 2: TLA+

`fs/eventfd.tla`:
- States: per-ctx counter, per-ctx waitqueue.
- Properties:
  - `safety_counter_in_range` — per-write: counter ∈ [0, ULLONG_MAX - 1].
  - `safety_semaphore_step` — per-semaphore-mode: each read decrements by exactly 1.
  - `safety_read_consumes` — per-default-mode: read returns counter and zeros it.
  - `liveness_eventual_drain` — per-blocking-write: eventually completes once reader drains.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_eventfd2` post: fd refers to anon_inode[eventfd] | `EventFd::do_eventfd2` |
| `read` post: counter == 0 (default) or counter -= 1 (semaphore) | `EventfdCtx::read` |
| `write` post: counter += add atomically | `EventfdCtx::write` |

### Layer 4: Verus / Creusot functional

Per-`eventfd(2)` man-page equivalence. LTP `eventfd01..02` pass. `eventfd(N)` and `eventfd2(N, 0)` produce equivalent fds.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`eventfd(2)` legacy reinforcement:

- **Per-counter saturation** — defense against per-counter-overflow wrap-to-zero.
- **Per-read/write size ≥ 8** — defense against per-partial-IO corruption.
- **Per-ctx refcount strict** — defense against per-UAF on close-during-IO.
- **Per-waitqueue wake-all atomic** — defense against per-lost-wakeup.
- **Per-anon_inode anonymity** — defense against per-procfs eventfd enumeration.

## Grsecurity / PaX-style Reinforcement

- **eventfd counter quota** — grsec adds a per-uid quota on total eventfd counter sum to prevent counter-flood DoS where a malicious task writes large values to many eventfds, pinning kernel memory in waitqueues. sysctl `kernel.eventfd_counter_max_per_uid` (default unlimited; hardened systems set 1<<30).
- **signalfd CLOEXEC mandatory analog** — grsec REWRITES legacy `eventfd(2)` to forcibly set `FD_CLOEXEC`, eliminating post-execve eventfd-leak. Userspace that depends on the legacy "fd survives execve" semantics MUST use `eventfd2(flags & EFD_CLOEXEC)` explicitly to opt-out (sysctl `kernel.eventfd_legacy_cloexec_default = 1`).
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/fdinfo/<efd>` counter field is filtered for non-owners; you cannot enumerate another process's eventfd counter.
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — peer ptracers cannot inspect another's eventfd counter without CAP_SYS_PTRACE.
- **PIDFD_SELF safe self-target** — eventfd is per-fd, not pidfd-targetable; grsec adds no PIDFD_SELF interaction.
- **PaX UDEREF on read/write user buffers** — copy_to_user/copy_from_user paths pre-validated via UDEREF; rejects kernel-half pointers.
- **EPOLL fd-lifetime refcount strict** — when eventfd is watched by epoll (very common pattern: io_uring + eventfd), the eventfd file refcount is held by the epitem; grsec asserts the watched-file ref is properly released on `EPOLL_CTL_DEL` and on epfd close.
- **GRKERNSEC_AUDIT_GROUP** — burst-rate eventfd creation from a marked group is rate-limited / audited (potential resource exhaustion).
- **No_new_privs neutral** — NNP does not change eventfd semantics.
- **Anti-fingerprint hardening** — eventfd counter is opaque; never echo kernel-side pointers/addresses. The `id` field used in /proc/fdinfo is namespaced per-pidns.
- **kvm_irqfd / kvm_ioeventfd binding** — grsec audits when eventfd is bound to KVM (CAP_SYS_ADMIN required for some KVM ioctls); cross-VM eventfd sharing is denied without explicit CAP_SYS_RAWIO.
- **Capability gate for EFD_SEMAPHORE counter steal** — semaphore mode is unprivileged; grsec audits high-rate semaphore writes from untrusted users.
- **Seccomp interaction** — seccomp filters may explicitly block syscall 284 in favor of 290 (`eventfd2`); default-deny hardening recommends forbidding legacy eventfd entirely.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `eventfd2(2)` (Tier-5 separate doc — modern variant with flags).
- KVM ioeventfd / irqfd bindings (Tier-3 in `virt/kvm/eventfd.md`).
- io_uring `IORING_REGISTER_EVENTFD` (Tier-3 in `io_uring/register.md`).
- `pipe2(2)` analogue patterns (Tier-5 in `pipe2.md`).
- waitqueue internals (Tier-3 in `kernel/sched/wait.md`).
- Implementation code.
