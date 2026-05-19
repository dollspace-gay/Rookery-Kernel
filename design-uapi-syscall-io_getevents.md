---
title: "Tier-5 syscall: io_getevents(2) — syscall 208"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`io_getevents(2)` is the original Linux AIO completion-reaping syscall introduced with `io_setup(2)`/`io_submit(2)`. Given a context handle `ctx_id` previously returned by `io_setup(2)`, it blocks (or polls, with timeout) until between `min_nr` and `nr` completed I/O events are available in the per-context completion ring (`aio_ring`), copies their `struct io_event` records to userspace, and returns the number copied. It is the read half of the asynchronous-IO submit/reap loop and the per-syscall mechanism by which userspace observes that previously submitted `iocb`s have finished.

Critical for: legacy AIO consumers (databases — PostgreSQL, MySQL — and storage middleware that predate io_uring), and any in-tree selftest that exercises `fs/aio.c`. New code is expected to use `io_uring`, but the syscall remains ABI-stable.

### Acceptance Criteria

- [ ] AC-1: Submit one iocb via `io_submit`, then `io_getevents(ctx,1,1,buf,NULL)` returns 1 with `buf[0].obj == iocb_ptr`.
- [ ] AC-2: `io_getevents(invalid_ctx,...)` returns `-EINVAL`.
- [ ] AC-3: `io_getevents(ctx,0,1,buf,{0,0})` polls and returns 0 when nothing ready.
- [ ] AC-4: `io_getevents(ctx,1,1,buf,{0,500000000})` returns 0 after ~500 ms when no completion arrives.
- [ ] AC-5: Signal delivered during wait → `-EINTR`.
- [ ] AC-6: `events` pointer that faults → `-EFAULT` (no events lost — undelivered events remain in ring).
- [ ] AC-7: `min_nr < 0` or `nr < min_nr` → `-EINVAL`.
- [ ] AC-8: Two threads racing `io_getevents` on same ctx with two completions ready: each thread returns 1; no duplicate delivery.
- [ ] AC-9: Concurrent `io_destroy(ctx)`: pending getevents returns `-EAGAIN`.
- [ ] AC-10: `res` field matches the byte count or `-errno` of the underlying I/O.

### Architecture

```rust
#[syscall(nr = 208, abi = "sysv")]
pub fn sys_io_getevents(
    ctx_id: AioContextT,
    min_nr: i64,
    nr: i64,
    events: UserPtrMut<IoEvent>,
    timeout: UserPtr<KernelTimespec>,
) -> isize {
    Aio::do_io_getevents(ctx_id, min_nr, nr, events, timeout)
}
```

`Aio::do_io_getevents(ctx_id, min_nr, nr, events, timeout) -> isize`:
1. /* Validate bounds */
2. if min_nr < 0 || nr < min_nr { return -EINVAL; }
3. if nr.saturating_mul(size_of::<IoEvent>() as i64) > isize::MAX as i64 { return -EINVAL; }
4. /* Resolve context */
5. let ctx = Aio::lookup_ioctx(ctx_id).ok_or(-EINVAL)?;
6. /* Convert timeout */
7. let until: Option<KTime> = match timeout.is_null() {
   - true  => None,
   - false => Some(Aio::copy_timespec(timeout)?.to_ktime_or(EFAULT)),
   };
8. /* Reap loop */
9. let copied = Aio::read_events(&ctx, min_nr, nr, events, until)?;
10. Aio::put_ioctx(ctx);
11. copied as isize

`Aio::read_events(ctx, min_nr, nr, events, until) -> Result<i64>`:
1. let mut total: i64 = 0;
2. loop {
   - /* Drain whatever the ring holds now */
   - let n = Aio::read_events_ring(ctx, events.add(total as usize), nr - total)?;
   - total += n;
   - if total >= min_nr || ctx.dying.load() { break; }
   - /* Wait for more */
   - match Aio::wait_for_completion(ctx, until)? {
     - WaitResult::Woken   => continue,
     - WaitResult::Timeout => break,
     - WaitResult::Signal  => return Err(EINTR),
     - WaitResult::Dying   => return Err(EAGAIN),
     }
   };
3. Ok(total)

`Aio::read_events_ring(ctx, events, max) -> Result<i64>`:
1. let ring = ctx.ring_mapping.kernel_view();
2. let head = ring.head; let tail = ring.tail;
3. let avail = ring_avail(head, tail, ctx.nr);
4. let n = min(avail, max);
5. for i in 0..n {
   - let slot = (head + i) % ctx.nr;
   - let ev = ring.events[slot];
   - events.add(i as usize).copy_out(&ev)?;       // EFAULT
   };
6. /* Atomically advance head */
7. ring.head = (head + n) % ctx.nr;
8. Ok(n)

### Out of Scope

- `io_setup(2)` context creation (covered in its own Tier-5 doc).
- `io_submit(2)` iocb submission (covered in its own Tier-5 doc).
- `io_destroy(2)` context teardown (covered separately).
- `io_uring` (covered under `io_uring_*` Tier-5 docs).
- `aio_ring` page-allocation policy (covered in Tier-3 `fs/aio.md`).
- Implementation code.

### signature

```c
int io_getevents(aio_context_t ctx_id,
                 long min_nr,
                 long nr,
                 struct io_event *events,
                 struct timespec *timeout);
```

```c
typedef __kernel_ulong_t aio_context_t;

struct io_event {
    __u64 data;       /* iocb->aio_data echoed back */
    __u64 obj;        /* pointer to original iocb (user vaddr) */
    __s64 res;        /* result code (>=0 byte count, <0 -errno) */
    __s64 res2;       /* secondary result (unused on modern paths) */
};
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `ctx_id` | `aio_context_t` | in | AIO context handle from `io_setup`. Kernel maps via radix lookup to a `kioctx`. |
| `min_nr` | `long` | in | Minimum events to wait for; `0` makes the call non-blocking past initially-available events. Must be `>= 0` and `<= nr`. |
| `nr` | `long` | in | Maximum events to copy. Bounded by kernel-side aio_max_nr per context. |
| `events` | `struct io_event *` | out | Caller buffer for completed events; must hold `nr` records (`nr * sizeof(struct io_event)`). |
| `timeout` | `struct timespec *` | in (optional) | Maximum wait. `NULL` waits indefinitely. `{0,0}` polls. |

### return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of events copied into `events[]`; may be less than `min_nr` if `timeout` expired (returning `0` is legal on timeout when `min_nr > 0`). |
| `-1` + `errno` | Failure (see below). |

### errors

| errno | Trigger |
|---|---|
| `EINVAL` | Invalid `ctx_id` (no live `kioctx` lookup), `min_nr < 0`, `nr < min_nr`, or `timeout` has invalid fields. |
| `EFAULT` | `events` or `timeout` user pointer faults during copy_from_user / copy_to_user. |
| `EINTR` | Wait interrupted by signal before `min_nr` events were available; events copied so far are returned via the success path only when count >= min_nr. |
| `EAGAIN` | Context was being torn down (`exit_aio`) concurrently. |
| `ENOSYS` | Built without `CONFIG_AIO`. |

### abi surface

```text
__NR_io_getevents (x86_64) = 208
__NR_io_getevents (arm64)  =   4
__NR_io_getevents (riscv)  =   4
__NR_io_getevents (i386)   = 247

/* struct io_event is 32 bytes on every ABI; ABI-stable since 2.5. */
/* struct timespec is the legacy 32-bit-tv_sec form on 32-bit ABIs;
   the y2038-safe variant is io_pgetevents_time64 (syscall 416 on i386). */
```

### compatibility contract

REQ-1: Syscall number is **208** on x86_64. ABI-stable.

REQ-2: `ctx_id` must resolve to a live `kioctx`; `lookup_ioctx(ctx_id)` returns `NULL` for invalid/destroyed contexts → `-EINVAL`.

REQ-3: `min_nr` and `nr` are checked: `min_nr >= 0`, `nr >= min_nr`, `nr <= LONG_MAX / sizeof(struct io_event)` (overflow guard on the user buffer size).

REQ-4: When `min_nr <= aio_ring->head_to_tail`, the kernel returns immediately without scheduling.

REQ-5: When events are not yet available, the kernel parks on the context's `wait` waitqueue and is woken by `aio_complete()` from completion IRQ context (or workqueue context for completion handlers).

REQ-6: `timeout` is converted with `timespec64_to_ktime`; `NULL` means infinite (`KTIME_MAX`); `{0,0}` returns whatever is presently available (may be 0).

REQ-7: `timeout` is updated on return on some legacy paths? **No** — Linux does not update `timeout` (unlike `select(2)`). Callers must reset their own timer.

REQ-8: Events are copied via `aio_read_events_ring()`, which advances `aio_ring->head` (per-context shared mmap). Copying respects ring-buffer wrap.

REQ-9: Completion order is FIFO with respect to `aio_complete()` calls; not necessarily submission order.

REQ-10: Concurrent `io_getevents()` from multiple threads on the same `ctx_id` is supported; each event is delivered to exactly one caller (per-event ring-slot reservation via `cmpxchg` of head).

REQ-11: Per-EINTR semantics: if a signal arrives before `min_nr` events accrue and the kernel has already copied some, the kernel still returns `-EINTR` (the partial-event count is forfeited; caller must re-poll to re-collect). This matches the upstream kernel's `read_events()` behavior.

REQ-12: Per-`exit_aio()` race: if the context is destroyed mid-wait, the waitqueue is woken with `-EAGAIN`.

REQ-13: Per-userspace ring access: userspace MAY directly read `aio_ring` (mmap'd) to peek at events without a syscall, but MUST use `io_getevents` for blocking semantics. (`aio_ring->magic == AIO_RING_MAGIC`.)

REQ-14: Per-compat ABI: on 32-bit compat, `compat_io_getevents` translates `struct compat_io_event` and `compat_timespec`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `bounds_check_total` | INVARIANT | min_nr<0 || nr<min_nr ⟹ EINVAL before any work. |
| `ctx_refcount_balanced` | INVARIANT | every lookup_ioctx paired with put_ioctx. |
| `copy_within_bounds` | INVARIANT | copy_out called <= nr times. |
| `signal_returns_eintr` | INVARIANT | wait interrupted ⟹ EINTR, no partial count. |
| `eagain_on_dying_ctx` | INVARIANT | exit_aio race ⟹ EAGAIN. |
| `head_advance_atomic` | INVARIANT | ring head advanced via cmpxchg under ctx.lock. |

### Layer 2: TLA+

`fs/aio-getevents.tla`:
- States: per-call lookup, copy-loop, wait-park, completion-wake, copy-out.
- Properties:
  - `safety_no_duplicate_event_delivery` — each ring slot consumed by exactly one caller.
  - `safety_head_monotonic` — ring.head only advances.
  - `safety_no_event_leak_on_efault` — EFAULT path leaves slot un-consumed.
  - `liveness_progress_under_completion` — completion eventually wakes a parked getevents.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_io_getevents` post: returns -EINVAL for invalid ctx | `Aio::do_io_getevents` |
| `read_events` post: total <= nr ∧ total >= min_nr (on success) | `Aio::read_events` |
| `read_events_ring` post: head advanced by n; events copied | `Aio::read_events_ring` |
| `wait_for_completion` post: terminates with Woken/Timeout/Signal/Dying | `Aio::wait_for_completion` |

### Layer 4: Verus / Creusot functional

Per-`io_getevents(2)` man-page semantics + libaio semantic equivalence. Selftests: `tools/testing/selftests/aio/` pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`io_getevents(2)` reinforcement:

- **Per-context lookup table refcounted** — defense against per-ctx UAF after `io_destroy`.
- **Per-ring head advance atomic** — defense against per-double-delivery race.
- **Per-`nr` overflow guard** — defense against per-user-buffer-size overflow.
- **Per-EFAULT keeps slot unconsumed** — defense against per-completion loss on user-page fault.
- **Per-`aio_max_nr` system-wide cap (`sysctl_aio_max_nr`)** — defense against per-context exhaustion DoS.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `events` and `timeout` copy** — defense against per-user-pointer kernel-deref. SMAP forced; user accesses gated by stac/clac.
- **GRKERNSEC_HIDESYM on sched_clock-based wait timings** — defense against per-timer-jitter side-channel revealing kernel scheduler internals via wait-resolution.
- **AIO RLIMIT_MEMLOCK accounting on `aio_ring` mmap** — the per-context ring is mlocked-equivalent; grsec accounts its pages against `RLIMIT_MEMLOCK` of the issuing user so a malicious userspace cannot pin arbitrary memory by spinning up contexts.
- **`sysctl_aio_max_nr` enforced per-userns** — defense against per-userns aio exhaustion; grsec narrows the upstream global counter to a per-user-namespace counter.
- **PAX_REFCOUNT on `kioctx` refcount** — defense against per-refcount-overflow UAF when malicious userspace races getevents against io_destroy.
- **GRKERNSEC_PROC_GETPID** — per-context info (per-task `/proc/<pid>/io`, `/proc/<pid>/aio`) hidden from non-owner; defense against per-AIO-fingerprinting info-leak.
- **PAX_USERCOPY_HARDEN on `io_event` copy_to_user** — bounded copy uses whitelisted slab region; defense against per-overlong-copy.
- **Per-`aio_ring` mmap PaX MPROTECT** — userspace's view of the ring is read-only-mapped; defense against per-userspace tampering with `head`/`tail` to forge completions.
- **GRKERNSEC_DENYUSB / GRKERNSEC_KMEM-style restriction on direct `aio_ring` peek** — hardened policy requires `io_getevents()` syscall path and disallows the optimization of userspace reading the ring directly when the trust boundary is hostile.
- **Per-wait waitqueue bounded by ctx->nr** — defense against per-waitqueue-pile-up DoS.

