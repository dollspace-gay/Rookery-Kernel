# Tier-5 syscall: epoll_pwait2(2) — syscall 441

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - fs/eventpoll.c (sys_epoll_pwait2, do_epoll_wait, ep_poll)
  - include/uapi/linux/eventpoll.h
  - arch/*/include/generated/uapi/asm/unistd_64.h (441)
  - Documentation/admin-guide/epoll.rst, man epoll_pwait2(2), man epoll(7)
-->

## Summary

`epoll_pwait2(2)` is the **nanosecond-precision, sigmask-aware** completion-wait of the epoll family. It supersedes `epoll_pwait(2)` (syscall 73 / 113 / 346 depending on arch) by accepting a `struct __kernel_timespec *` timeout instead of an `int milliseconds`, enabling sub-millisecond wait deadlines required by modern latency-sensitive runtimes (DPDK fallback, Aeron, ScyllaDB shard reactors, NIC-RX timestamping, real-time audio). Like `epoll_pwait`, the syscall atomically substitutes the calling task's signal mask for the duration of the wait, restoring on return — closing the classic `signal-before-wait` race window unsolvable with separate `sigprocmask + epoll_wait`. Critical for: high-frequency reactors, signal-correct shutdown, deadline-respecting event loops.

This Tier-5 covers the userspace ABI of syscall 441.

## Signature

```c
int epoll_pwait2(int epfd,
                 struct epoll_event *events,
                 int maxevents,
                 const struct __kernel_timespec *timeout,
                 const sigset_t *sigmask,
                 size_t sigsetsize);
```

Rust ABI shim:

```rust
pub fn sys_epoll_pwait2(epfd: i32,
                        events: *mut EpollEvent,
                        maxevents: i32,
                        timeout: *const __kernel_timespec,
                        sigmask: *const SigSet,
                        sigsetsize: usize) -> isize;
```

Syscall number: **441**.

## Parameters

| Param | Type | Direction | Purpose |
|---|---|---|---|
| `epfd` | `i32` | IN | epoll instance fd from `epoll_create1(2)` |
| `events` | `struct epoll_event *` | OUT | array to receive ready events; written `0..ret` |
| `maxevents` | `i32` | IN | capacity of `events`; `1 ≤ maxevents ≤ EP_MAX_EVENTS = INT_MAX/sizeof(epoll_event)` |
| `timeout` | `const __kernel_timespec *` | IN | relative wait deadline; `NULL` ⟹ block forever; `{0,0}` ⟹ poll-only |
| `sigmask` | `const sigset_t *` | IN | sigmask to install for the duration of wait; `NULL` ⟹ unchanged |
| `sigsetsize` | `size_t` | IN | size of `sigmask`; MUST equal `_NSIG/8` (typically `8`) |

## Return

- **Success**: non-negative count of events written to `events[]` (`0` on timeout with no ready events).
- **Failure**: `-1` and `errno`; Rust internal returns negated errno.

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `epfd` not a valid fd |
| `EINTR` | wait interrupted by signal not blocked by `sigmask` |
| `EINVAL` | `epfd` not an epoll fd; `maxevents ≤ 0`; `sigmask != NULL` and `sigsetsize != _NSIG/8`; `timeout->tv_nsec` ∉ `[0, 999_999_999]`; `timeout->tv_sec < 0` |
| `EFAULT` | `events` not writable; `timeout` not readable; `sigmask` not readable |
| `ECANCELED` | epoll instance being torn down concurrently |

## ABI surface

Constants reused from `epoll_create1.md` / `epoll_ctl.md`:

- `struct epoll_event` (12 bytes packed on x86, 16 elsewhere).
- `EPOLLIN | EPOLLOUT | EPOLLERR | EPOLLHUP | EPOLLRDHUP | EPOLLPRI | EPOLLEXCLUSIVE | EPOLLET | EPOLLONESHOT | EPOLLWAKEUP`.
- `EP_MAX_NESTS = 4`.
- `EP_MAX_EVENTS = INT_MAX / sizeof(struct epoll_event)`.

`struct __kernel_timespec`:

```c
struct __kernel_timespec {
    __kernel_time64_t tv_sec;   /* always 64-bit */
    long long         tv_nsec;
};
```

`sigsetsize` is the size in bytes of a `sigset_t`; on Linux this is `_NSIG/8` = `8` on x86_64 / arm64 / riscv64.

Companion / predecessors:

| Syscall | Timeout type | Sigmask | Note |
|---|---|---|---|
| `epoll_wait(2)` | `int ms` | none | legacy |
| `epoll_pwait(2)` | `int ms` | yes | adds sigmask |
| `epoll_pwait2(2)` | `__kernel_timespec *` | yes | adds ns-precision |

## Compatibility contract

REQ-1: `maxevents` validation:
- `maxevents ≤ 0` → `-EINVAL`.
- `maxevents > EP_MAX_EVENTS` → `-EINVAL`.

REQ-2: `epfd` validation:
- `epfd` must be open and be an epoll fd; else `-EBADF` / `-EINVAL`.

REQ-3: `sigsetsize` validation:
- `sigmask != NULL`: `sigsetsize` MUST equal `_NSIG/8` else `-EINVAL`.
- `sigmask == NULL`: `sigsetsize` ignored.

REQ-4: `timeout` semantics:
- `timeout == NULL` ⟹ block indefinitely until event / signal / cancellation.
- `timeout->tv_sec == 0 && timeout->tv_nsec == 0` ⟹ poll-only; immediate return.
- `timeout->tv_sec > 0 || timeout->tv_nsec > 0` ⟹ relative deadline on `CLOCK_MONOTONIC`.
- `tv_nsec` MUST be `[0, 999_999_999]` else `-EINVAL`.
- `tv_sec < 0` ⟹ `-EINVAL`.

REQ-5: Sigmask install/restore:
- If `sigmask != NULL`: atomic install before any wait happens.
- Sigmask masked from observation: `SIGKILL`, `SIGSTOP` cannot be masked.
- On any return path (success / signal / timeout / fault), original sigmask restored.

REQ-6: Wait semantics:
- Block until at least one fd is ready, or timeout, or signal.
- On ready: copy up to `maxevents` `EpollEvent`s from the ready list into `events[]`.
- For each level-triggered ready entry: stays on ready list (re-checked next call).
- For each edge-triggered ready entry (`EPOLLET`): removed from ready list — userspace must drain target until `EAGAIN`.
- For each oneshot entry (`EPOLLONESHOT`): mask cleared after copy; re-arm via `EPOLL_CTL_MOD`.
- For exclusive entry (`EPOLLEXCLUSIVE`): only one waker per event.

REQ-7: Returned count:
- `ret == 0` ⟹ timeout (or empty poll-only).
- `ret > 0` ⟹ `events[0..ret]` populated.
- `ret ≤ maxevents`.

REQ-8: `events` is OUT only:
- Kernel does not read prior contents.
- Faulting page-on-write on partial copy: `-EFAULT`.

REQ-9: Concurrent `epoll_ctl`:
- Allowed concurrently.
- Entries added during wait become candidates for wake.
- Entries removed during wait dequeued atomically.

REQ-10: Concurrent `close(epfd)`:
- Wakes waiter with `-ECANCELED`.

REQ-11: Spurious wakes:
- Permitted under `EPOLLET` semantics if target state oscillates; userspace must recheck via `EAGAIN`.

REQ-12: Multiple concurrent waiters on same epoll:
- All wake on event (default) unless `EPOLLEXCLUSIVE` interest entries (then one waker only).

REQ-13: `EPOLLPRI` propagation:
- TCP OOB / chrdev priority data: reported even if not subscribed (implicit per REQ-7 in `epoll_ctl.md`).

REQ-14: `EPOLLERR` / `EPOLLHUP` propagation:
- Always reported, even if not subscribed.

REQ-15: Timeout precision:
- Resolution governed by kernel tick (`CONFIG_HZ`) for non-`HRTIMER` builds; `HRTIMER` builds achieve sub-ms.
- `epoll_pwait2` requests but does not guarantee ns-precision; bound by underlying timer driver.

## Acceptance Criteria

- [ ] AC-1: `epoll_pwait2(epfd, ev, 8, NULL, NULL, 0)` with no events ready: blocks until event or signal.
- [ ] AC-2: `epoll_pwait2(epfd, ev, 8, &ts{0,0}, NULL, 0)`: returns `0` immediately if no events.
- [ ] AC-3: `epoll_pwait2(epfd, ev, 8, &ts{0,500_000}, NULL, 0)` (500us): returns `0` after ~500us; `ret == 0`.
- [ ] AC-4: `maxevents == 0` → `-EINVAL`.
- [ ] AC-5: `maxevents == -1` → `-EINVAL`.
- [ ] AC-6: `sigmask != NULL` and `sigsetsize != 8` → `-EINVAL`.
- [ ] AC-7: `ts.tv_nsec == 1_000_000_000` → `-EINVAL`.
- [ ] AC-8: `ts.tv_sec == -1` → `-EINVAL`.
- [ ] AC-9: Signal arrives during wait: `-EINTR`; sigmask restored.
- [ ] AC-10: Concurrent `close(epfd)` mid-wait → `-ECANCELED`.
- [ ] AC-11: `EPOLLET` entry: returned once; subsequent waits don't repeat until drain + new edge.
- [ ] AC-12: `EPOLLONESHOT` entry: returned once; mask cleared; `EPOLL_CTL_MOD` re-arms.
- [ ] AC-13: `EPOLLERR` reported even without subscription.
- [ ] AC-14: Multiple ready: up to `maxevents` reported per call; rest reported next call.
- [ ] AC-15: `events == NULL` → `-EFAULT`.

## Architecture

Rookery surface in `kernel/eventpoll/pwait2.rs`:

```rust
pub const EP_MAX_EVENTS: i32 = (i32::MAX as usize / mem::size_of::<EpollEvent>()) as i32;
```

`Epoll::pwait2(epfd, ev_uptr, max, ts_uptr, sig_uptr, sigsize) -> isize`:
1. /* maxevents */
2. if max <= 0 || max > EP_MAX_EVENTS { return -EINVAL; }
3. /* resolve ctx */
4. let ep_file = current.fdtable.get(epfd).ok_or(-EBADF)?;
5. let ctx = ep_file.as_eventpoll().ok_or(-EINVAL)?;
6. /* sigmask validation */
7. let saved_mask = if !sig_uptr.is_null() {
     - if sigsize != size_of::<SigSet>() { return -EINVAL; }
     - let m = copy_from_user::<SigSet>(sig_uptr).map_err(|_| -EFAULT)?;
     - Some(current.swap_sigmask(m & !KILL_STOP_MASK))
   } else {
     - None
   };
8. /* timeout */
9. let deadline = if ts_uptr.is_null() {
     - Deadline::Forever
   } else {
     - let ts = copy_from_user::<KernelTimespec>(ts_uptr).map_err(|_| -EFAULT)?;
     - if ts.tv_sec < 0 || ts.tv_nsec < 0 || ts.tv_nsec >= 1_000_000_000 {
       - restore_mask_if(saved_mask, current);
       - return -EINVAL;
     }
     - if ts.tv_sec == 0 && ts.tv_nsec == 0 { Deadline::Poll }
     - else { Deadline::Relative(ts) }
   };
10. /* validate events writable */
11. if !user_writable(ev_uptr, max as usize * size_of::<EpollEvent>()) {
      - restore_mask_if(saved_mask, current);
      - return -EFAULT;
    }
12. /* wait loop */
13. let res = ep_poll(&ctx, ev_uptr, max, deadline);
14. /* restore sigmask */
15. restore_mask_if(saved_mask, current);
16. res.map_or_else(|e| e as isize, |n| n as isize);

`ep_poll(ctx, ev_uptr, max, deadline)`:
1. let start = clock_monotonic();
2. loop {
   - /* fast path: scan ready list */
   - let scanned = scan_rdllist_into_user(&ctx, ev_uptr, max)?;
   - if scanned > 0 { return Ok(scanned); }
   - /* check deadline */
   - match deadline {
     - Poll => return Ok(0),
     - Relative(ts) => {
       - let elapsed = clock_monotonic() - start;
       - if elapsed >= ts { return Ok(0); }
     },
     - Forever => (),
   }
   - /* check cancellation */
   - if ctx.is_torn_down() { return Err(-ECANCELED); }
   - /* check signal */
   - if signal_pending() { return Err(-EINTR); }
   - /* park */
   - ctx.wq.wait_with_deadline(remaining(deadline, start));
 }

`scan_rdllist_into_user(ctx, ev_uptr, max)`:
1. let mut rd = ctx.rdllist.lock();
2. let mut written = 0;
3. while written < max && let Some(epi) = rd.pop_front() {
     - let ev = *epi.event.read();
     - let state = epi.poll_state() & ev.events;
     - if state == 0 { continue; }  /* spurious — drop */
     - let out_ev = EpollEvent { events: state | implicit_err_hup(epi), data: ev.data };
     - copy_to_user(ev_uptr.add(written), &out_ev)?;
     - /* re-arm logic */
     - let events_flags = EpollEvents::from_bits_truncate(ev.events);
     - if events_flags.contains(EpollEvents::ET) {
       - /* ET: stay off rdllist until next edge */
     } else if events_flags.contains(EpollEvents::ONESHOT) {
       - epi.event.write().events &= !ALL_EVENT_BITS;
     } else {
       - /* LT: re-queue if still ready after callback drain */
       - if epi.poll_state() != 0 { rd.push_back(epi); }
     }
     - written += 1;
   }
4. Ok(written).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `maxevents_positive` | INVARIANT | `max ≤ 0 ⟹ -EINVAL` |
| `sigsetsize_strict` | INVARIANT | `sigmask != NULL ∧ sigsetsize != _NSIG/8 ⟹ -EINVAL` |
| `ts_normalized` | INVARIANT | `tv_nsec ∉ [0, 999_999_999] ⟹ -EINVAL` |
| `ts_nonneg` | INVARIANT | `tv_sec < 0 ⟹ -EINVAL` |
| `sigmask_restored` | INVARIANT | sigmask restored on every return path |
| `ret_bounded` | INVARIANT | `ret ≤ max` |
| `events_user_writable` | INVARIANT | `EFAULT` if any of `[ev, ev+max)` not writable |
| `et_dequeue` | INVARIANT | ET entry returned once until next edge |

### Layer 2: TLA+

`uapi/epoll_pwait2.tla`:
- Variables: `rdllist`, `sigmask`, `deadline`, `signal_pending`, `events_array`.
- Properties:
  - `safety_sigmask_restored` — sigmask restored on every termination.
  - `safety_ret_bounded` — `ret ≤ max`.
  - `safety_et_once` — ET entry leaves rdllist post-copy.
  - `safety_lt_persistent` — LT entry stays on rdllist while ready.
  - `liveness_terminate` — call terminates on event / signal / timeout / cancel.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `pwait2` post(ok): `0 ≤ ret ≤ max` | `Epoll::pwait2` |
| `pwait2` post: sigmask unchanged on entry/return | `Epoll::pwait2` |
| `scan_rdllist_into_user` post: `written ≤ max` | `scan_rdllist_into_user` |
| `ep_poll` post(EINTR): signal_pending was true | `ep_poll` |
| `ep_poll` post(Ok(0)): poll-only or relative-deadline expired | `ep_poll` |

### Layer 4: Verus/Creusot functional

Per-`epoll_pwait2(2)` man page, `man epoll(7)`, `Documentation/admin-guide/epoll.rst` semantic equivalence. Per-LTP `testcases/kernel/syscalls/epoll_pwait2/*.c` and glibc `sysdeps/unix/sysv/linux/epoll_pwait2.c` round-trip.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

epoll_pwait2 reinforcement:

- **Sigmask install/restore strictly paired** — defense against per-leak across syscall boundary.
- **Timeout normalization strict** — defense against per-negative-tv-nsec hang.
- **Per-call user-writable prevalidate** — defense against per-partial-copy fault.
- **Per-ET dequeue invariant** — defense against per-busy-loop on stuck edge.
- **Per-`ECANCELED` on tear-down** — defense against per-stale-wait UAF.

## Grsecurity/PaX-style Reinforcement

- **PAX_RANDKSTACK on epoll_pwait2 entry** — randomize kernel stack at the highest-frequency epoll syscall; epoll_pwait2 is the hot path for modern reactors and a known target for layout disclosure via wait-latency timing.
- **PaX UDEREF on `copy_from_user` of `timeout`, `sigmask`, and `copy_to_user` of `events[]`** — enforces unsigned-only userspace deref across all four pointer surfaces; the `events` OUT pointer is particularly attractive because it's a kernel-side write into user memory with attacker-shaped size (`max * sizeof(EpollEvent)`).
- **GRKERNSEC_BPF_HARDEN considerations** — though epoll is not BPF, the per-instance poll-callback chain forms a small VM that runs in the wakeup path of every subscribed fd. Apply BPF-style discipline: bound the per-instance ready-list traversal under wakeup at policy-configurable limit; audit-log per-call `ret > N` (where N is policy-configurable).
- **CAP_BPF gating not appropriate** — pwait2 is a baseline reactor primitive.
- **GRKERNSEC_FIFO scaled to epoll-wait flood** — per-uid rate-limit on `epoll_pwait2` calls per second (poll-only and 0-timeout variants are the abuse vector); under flood refuse with `-EAGAIN` and audit-log.
- **PAX_USERCOPY hardening on `events[]` write** — validate exact-bound copy of `max * sizeof(EpollEvent)`; refuse copies whose destination spans a SLAB cache boundary that crosses into a sensitive allocation (epoll wait is a known SLAB-grooming primitive against neighboring allocator caches).
- **GRKERNSEC_HIDESYM on event reflection** — under `kernel.kptr_restrict ≥ 2` the kernel does not log `data.u64` payloads (which are attacker-controlled 64-bit blobs reflected back, frequently pointers in legitimate use).
- **Sigmask strict honoring** — refuse `sigmask` swap that would unmask `SIGKILL` or `SIGSTOP` (canonical Linux behavior — make explicit, not implicit); audit any attempt as policy probe.
- **Strict honoring of `EPOLLEXCLUSIVE` from prior `epoll_ctl(ADD)`** — refuse to wake non-exclusive waiters when an exclusive waiter is queued; defense against per-thundering-herd reintroduction via wait-path bug.
- **Audit log on repeated `EINTR` cycles with custom sigmask** — abnormal mask + EINTR cycles are characteristic of race-window probing on signal-versus-wake; rate-limited audit at `LOGLEVEL_INFO`.
- **Refuse `epoll_pwait2` from a task whose `dumpable == 0`-history mismatches the epoll instance's creator UID** — defense against fd-inheritance after setuid, where a setuid-dropped task continues to observe events on an instance created with elevated privilege.
- **Strict `__kernel_timespec` bounds enforced even on 64-bit kernels** — `tv_sec` overflow into negative interpretation is rejected with `-EINVAL`; defense against per-clock-arithmetic confusion.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `epoll_create1.md` sibling — instance creation
- `epoll_ctl.md` sibling — interest list manipulation
- `fs/eventpoll.md` Tier-3 — RB-tree, ready-list, callback path, wakeup mechanics
- `epoll_wait(2)` / `epoll_pwait(2)` legacy variants — covered as compatibility shims in `fs/eventpoll.md`
- Implementation code
