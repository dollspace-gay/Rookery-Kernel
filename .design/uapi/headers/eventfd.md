# Tier-5 UAPI: include/uapi/linux/eventfd.h — eventfd(2) ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/eventfd.h (~11 lines)
  - fs/eventfd.c
  - Documentation/admin-guide/eventfd.rst (man eventfd(2))
-->

## Summary

`eventfd(2)` (and `eventfd2(2)`) returns a file descriptor backed by a kernel-resident 64-bit unsigned counter, used as a lightweight kernel-to-userspace and userspace-to-userspace signaling primitive (replaces pipe(2)-based wakeups in libuv/glib/io_uring/KVM/vhost). Per-default mode: `read()` returns the entire counter and resets it to zero; in `EFD_SEMAPHORE` mode `read()` returns `1` and decrements the counter by one (semaphore semantics). Per-`write()` adds an arbitrary u64 to the counter (saturating at `0xfffffffffffffffe`; `ULLONG_MAX` reserved as the "would-overflow" sentinel). Per-`EFD_NONBLOCK`: `read()` of zero counter returns EAGAIN. Per-`EFD_CLOEXEC`: close-on-exec semantics. Critical for: epoll-based event loops, io_uring completion notifications, KVM irqfd, vhost worker wake-up.

This Tier-5 covers `include/uapi/linux/eventfd.h` (~11 lines).

## ABI surface

| Constant / Type | Value | Purpose |
|---|---|---|
| `EFD_SEMAPHORE` | `1 << 0` (0x1) | semaphore-mode counter (read decrements by 1) |
| `EFD_CLOEXEC` | `O_CLOEXEC` (0x80000) | close-on-exec |
| `EFD_NONBLOCK` | `O_NONBLOCK` (0x800) | non-blocking I/O |
| (syscall) `eventfd(unsigned int initval)` | legacy | flags=0 implied |
| (syscall) `eventfd2(unsigned int initval, int flags)` | extended | flags from above |

Read frame: `__u64 counter` (host-endian, 8 bytes).
Write frame: `__u64 inc` (host-endian, 8 bytes).
Counter range: `[0, 0xfffffffffffffffe]`. `0xffffffffffffffff` is the would-overflow sentinel — writes that would push counter to this value or higher return `-EAGAIN` (or block until reader drains, unless `EFD_NONBLOCK`).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `eventfd(2)` / `eventfd2(2)` | per-fd allocation | `Eventfd::create` |
| `eventfd_read` / `read()` op | per-counter consume | `Eventfd::read` |
| `eventfd_write` / `write()` op | per-counter add | `Eventfd::write` |
| `eventfd_signal()` (in-kernel) | per-kernel-source signal | `Eventfd::signal` |
| `EFD_SEMAPHORE` | per-mode | `EventfdFlags::SEMAPHORE` |

## Compatibility contract

REQ-1: `eventfd2(initval, flags)`:
- `flags` MUST be subset of `{ EFD_SEMAPHORE, EFD_CLOEXEC, EFD_NONBLOCK }` else EINVAL.
- Counter initialized to `initval` (u32 widened to u64).
- Returns fd ≥ 0 or -1/errno.

REQ-2: Legacy `eventfd(initval)` == `eventfd2(initval, 0)`.

REQ-3: `read(fd, buf, count)`:
- `count ≥ 8` else EINVAL.
- Blocks until counter > 0, unless `EFD_NONBLOCK` (then EAGAIN).
- If `EFD_SEMAPHORE` set: writes `1` to `buf`; counter decremented by 1.
- Else: writes current counter value to `buf`; counter reset to 0.
- Returns 8 on success.

REQ-4: `write(fd, buf, count)`:
- `count ≥ 8` else EINVAL.
- `*(u64*)buf` MUST NOT equal `0xffffffffffffffff` (reserved overflow sentinel) — else EINVAL.
- counter_new = counter + inc.
- If counter_new > `0xfffffffffffffffe`:
  - block until reader drains (counter goes to 0) unless `EFD_NONBLOCK`, then EAGAIN.
- Else: counter = counter_new.
- Returns 8 on success.

REQ-5: `poll(fd, POLLIN)`:
- POLLIN set iff counter > 0.
- POLLOUT set iff counter + 1 ≤ `0xfffffffffffffffe` (a `write(1)` would succeed).
- POLLERR set iff counter == `0xffffffffffffffff` (kernel-side overflow detected; shouldn't be userspace-visible but reserved).

REQ-6: Per-fork: fd inherited; counter is shared across all opens of the same fd (per-file struct).

REQ-7: Per-exec: cleared iff `EFD_CLOEXEC` set at create time.

REQ-8: In-kernel API `eventfd_signal(ctx, n)` adds `n` to counter atomically with no userspace overhead — used by io_uring (CQ-tail wakeups), KVM (irqfd assertion), vhost-net (RX-ready), aio.

REQ-9: Per-counter monotonicity within a single read/write cycle is atomic: each read consumes a coherent snapshot; concurrent writes serialize.

## Acceptance Criteria

- [ ] AC-1: `eventfd2(0, EFD_CLOEXEC|EFD_NONBLOCK)` returns fd.
- [ ] AC-2: Invalid flag bit → EINVAL.
- [ ] AC-3: `write(fd, &1, 8)` then `read(fd, &v, 8)` → v=1 (non-semaphore).
- [ ] AC-4: With `EFD_SEMAPHORE`: write 5; five reads each return 1; sixth blocks/EAGAIN.
- [ ] AC-5: `read` with count < 8 → EINVAL.
- [ ] AC-6: `write` with count < 8 → EINVAL.
- [ ] AC-7: `write(fd, &0xffffffffffffffff, 8)` → EINVAL.
- [ ] AC-8: Write that would overflow counter past 0xfffffffffffffffe → blocks or EAGAIN.
- [ ] AC-9: `poll` reports POLLIN iff counter > 0; POLLOUT iff write(1) would succeed.
- [ ] AC-10: After `read` of nonzero counter (non-semaphore mode), subsequent reads block/EAGAIN.
- [ ] AC-11: `EFD_CLOEXEC` propagates `O_CLOEXEC` to fd; survives `dup` but not `exec`.

## Architecture

Rookery surface in `kernel/eventfd/uapi.rs`:

```rust
bitflags! {
    pub struct EventfdFlags: i32 {
        const SEMAPHORE = 1 << 0;
        const CLOEXEC   = 0x80000;
        const NONBLOCK  = 0x800;
    }
}

pub struct EventfdCtx {
    pub count:     AtomicU64,
    pub flags:     EventfdFlags,
    pub wq:        WaitQueue,
}

pub const EFD_MAX: u64 = 0xfffffffffffffffe;
```

`Eventfd::create(initval, flags)`:
1. if flags & !EventfdFlags::all().bits(): return -EINVAL.
2. ctx = EventfdCtx { count: AtomicU64::new(initval as u64), flags, wq: WaitQueue::new() }.
3. fd = anon_inode_getfd("[eventfd]", &Eventfd::fops, ctx, flags_to_open(flags)).
4. return fd.

`Eventfd::read(file, buf, count)`:
1. if count < 8: return -EINVAL.
2. loop:
   - cur = ctx.count.load(Acquire).
   - if cur > 0:
     - new = if SEMAPHORE { cur - 1 } else { 0 }.
     - if ctx.count.compare_exchange(cur, new, Release, Relaxed).is_ok(): break.
   - else:
     - if NONBLOCK: return -EAGAIN.
     - wq.wait_for(|| ctx.count.load() > 0).
3. v: u64 = if SEMAPHORE { 1 } else { cur }.
4. copy_to_user(buf, &v, 8).
5. ctx.wq.wake_all_writers().
6. return 8.

`Eventfd::write(file, buf, count)`:
1. if count < 8: return -EINVAL.
2. inc: u64 = copy_from_user(buf, 8).
3. if inc == u64::MAX: return -EINVAL.
4. loop:
   - cur = ctx.count.load(Acquire).
   - new = cur.checked_add(inc).
   - if new.is_none() ∨ new.unwrap() > EFD_MAX:
     - if NONBLOCK: return -EAGAIN.
     - wq.wait_for(|| ctx.count.load().saturating_add(inc) ≤ EFD_MAX).
     - continue.
   - if ctx.count.compare_exchange(cur, new.unwrap(), Release, Relaxed).is_ok(): break.
5. ctx.wq.wake_all_readers().
6. return 8.

`Eventfd::signal(ctx, n)` (in-kernel, no userspace fault):
1. ctx.count.fetch_add(n, AcqRel) under overflow check.
2. ctx.wq.wake_all_readers().

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `create_flags_subset` | INVARIANT | flags ⊆ {SEMAPHORE, CLOEXEC, NONBLOCK} |
| `read_count_ge_8` | INVARIANT | read len < 8 ⟹ EINVAL |
| `write_count_ge_8` | INVARIANT | write len < 8 ⟹ EINVAL |
| `write_max_sentinel_rejected` | INVARIANT | inc == u64::MAX ⟹ EINVAL |
| `count_bounded` | INVARIANT | post-write: count ≤ EFD_MAX |
| `semaphore_decrement_atomic` | INVARIANT | SEMAPHORE read: count' = count − 1 |
| `non_semaphore_drain_atomic` | INVARIANT | non-SEMAPHORE read: count' = 0 |
| `overflow_blocks_or_eagain` | INVARIANT | inc + count > EFD_MAX ⟹ block or EAGAIN |

### Layer 2: TLA+

`uapi/eventfd.tla`:
- Variables: `count`, `pending_readers`, `pending_writers`.
- Properties:
  - `safety_count_bounded` — count ∈ [0, EFD_MAX].
  - `safety_semaphore_drain` — N writes of 1 ⟹ exactly N successful semaphore reads.
  - `liveness_writer_eventually_progresses` — overflow-blocked writer wakes after reader drains.
  - `liveness_reader_eventually_reads` — count > 0 ⟹ blocked reader wakes.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `create` post: ctx.count == initval | `Eventfd::create` |
| `read` post (non-sem): ctx.count == 0 | `Eventfd::read` |
| `read` post (sem): ctx.count' == ctx.count − 1 | `Eventfd::read` |
| `write` post: ctx.count' = ctx.count + inc ∧ ctx.count' ≤ EFD_MAX | `Eventfd::write` |
| `signal` post: ctx.count' = ctx.count + n (saturating) | `Eventfd::signal` |

### Layer 4: Verus/Creusot functional

Per-`eventfd(2)` man page semantic equivalence. Per-LTP `testcases/kernel/syscalls/eventfd/*` and `glibc` `sysdeps/unix/sysv/linux/eventfd_*.c` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

## Grsecurity/PaX-style Reinforcement

- **GRKERNSEC_FIFO scaled to eventfd flood** — per-uid quota on simultaneous open eventfds and per-uid rate-limit on `Eventfd::signal` events (covers `eventfd2` the same way grsec's GRKERNSEC_FIFO covers pipes); refuses creation with `-EMFILE` past quota to break event-flood DoS against single-thread reactors.
- **EFD_CLOEXEC mandatory under restrictive policy** — when calling task has `PR_SET_NO_NEW_PRIVS=1` or runs under crosslink seccomp lockdown, refuse `eventfd2` without `EFD_CLOEXEC`; eliminates cross-exec covert-channel where parent leaks counter state to a less-privileged child.
- **Counter overflow detection mandatory** — the `0xffffffffffffffff` sentinel write is rejected with `-EINVAL` *and* logged at warn level; refuses any in-kernel `eventfd_signal(ctx, n)` whose `n` would push count past `EFD_MAX` (instead of silent saturation), preventing torn-counter primitive farming.
- **PAX_RANDKSTACK on eventfd read/write** — randomize kernel stack on entry; prevents stack-layout disclosure via eventfd polling cadence.
- **EFD_SEMAPHORE accounting per-uid** — semaphore-mode eventfds consume more wake-up overhead than plain counters; cap per-uid count and refuse creation past threshold.
- **GRKERNSEC_LOG_EVENTFD_OVERFLOW** — audit-log every blocked-overflow write and every saturated `eventfd_signal`; deters covert-channel use of overflow timing as side-channel.
- **In-kernel `eventfd_signal` from interrupt context bounded** — refuse `eventfd_signal` calls from IRQ handlers that exceed per-IRQ rate-limit; prevents lock-up via signal-flooding from misbehaving drivers (vhost / KVM irqfd).

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `fs/eventfd.c` implementation (covered in `fs/eventfd-impl.md` Tier-3 if expanded)
- `io_uring` SQ/CQ wake integration via `IORING_REGISTER_EVENTFD` (covered in `uapi/headers/io_uring.md`)
- KVM `irqfd` and `ioeventfd` (covered in `uapi/headers/kvm.md`)
- vhost worker eventfd wake-up (covered separately if expanded)
- Implementation code
