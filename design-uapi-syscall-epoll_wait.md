---
title: "Tier-5 syscall: epoll_wait(2) — syscall 232"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`epoll_wait(2)` is the **event-reap** half of the epoll interface: it blocks (or polls) the calling thread on the per-epoll-fd ready list, returning up to `maxevents` `struct epoll_event` records describing fds that have become ready since the last call. The kernel maintains a per-epoll `rdllist` (ready linked-list) populated by callback from each watched fd's poll-table wake-up; `epoll_wait` walks this list with `ep_send_events`, copying matched entries to userspace and (for level-triggered watches) re-arming. Timeout is in milliseconds: `-1` blocks indefinitely, `0` polls, positive values bound the wait. Modern userspace SHOULD use `epoll_pwait(2)` or `epoll_pwait2(2)` for atomic signal-mask + nanosecond timeouts; `epoll_wait` is the original 2.6.0 variant.

Critical for: hot-path event-loop reap in all multiplexed I/O servers; tokio epoll-driver `poll`; libuv `uv__io_poll`; libevent `event_base_loop`; Go runtime `netpoll`; nginx `ngx_epoll_module`; haproxy `_do_poll`; every async-I/O Linux runtime.

### Acceptance Criteria

- [ ] AC-1: `epoll_wait` with `timeout = 0` and empty ready-list returns 0 immediately.
- [ ] AC-2: `epoll_wait` with `timeout = -1` blocks until a watched fd becomes readable; returns 1 with `EPOLLIN`.
- [ ] AC-3: `epoll_wait` with `timeout = 100` returns 0 after ~100ms if no events.
- [ ] AC-4: `maxevents = 0` returns `EINVAL`.
- [ ] AC-5: `maxevents = -1` returns `EINVAL`.
- [ ] AC-6: `events = bad-pointer` returns `EFAULT`.
- [ ] AC-7: `epfd = non-epoll-fd` returns `EINVAL`.
- [ ] AC-8: Level-triggered fd remains ready across calls until drained.
- [ ] AC-9: Edge-triggered fd reports once per state transition; not again until cleared.
- [ ] AC-10: `EPOLLONESHOT` fd is auto-disarmed after one delivery.
- [ ] AC-11: Signal during blocking wait returns `EINTR`.
- [ ] AC-12: `EPOLLEXCLUSIVE` watch wakes at most one of N waiters per event.

### Architecture

```rust
#[syscall(nr = 232, abi = "sysv")]
pub fn sys_epoll_wait(
    epfd: i32,
    events: UserPtr<EpollEvent>,
    maxevents: i32,
    timeout: i32,
) -> KResult<i32> {
    /* Legacy: forward to epoll_pwait with sigmask = NULL. */
    EpollWait::do_epoll_wait(epfd, events, maxevents, timeout, None)
}
```

`EpollWait::do_epoll_wait(epfd, events, maxevents, timeout_ms, sigmask) -> KResult<i32>`:
1. /* Validate maxevents */
2. if maxevents <= 0 || maxevents > EP_MAX_EVENTS { return Err(EINVAL); }
3. /* UDEREF userspace events buffer */
4. let nbytes = (maxevents as usize) * core::mem::size_of::<EpollEvent>();
5. access_ok_uderef(events, nbytes)?;
6. /* Resolve epfd to eventpoll */
7. let file = fdget(epfd)?;
8. let ep = EventPoll::from_file(&file).ok_or(EINVAL)?;
9. /* Install sigmask if provided (epoll_pwait path) */
10. let saved_mask = sigmask.map(|m| set_sigmask_atomic(m));
11. /* Core wait */
12. let timeout_ns = ms_to_ns(timeout_ms);
13. let n = EpollWait::ep_poll(&ep, events, maxevents, timeout_ns)?;
14. /* Restore sigmask */
15. if let Some(m) = saved_mask { restore_sigmask(m); }
16. Ok(n)

`EpollWait::ep_poll(ep, events, maxevents, timeout_ns) -> KResult<i32>`:
1. /* Try fast path: ready list non-empty */
2. if !ep.rdllist.is_empty() {
   - return EpollWait::ep_send_events(ep, events, maxevents);
3. }
4. /* timeout = 0: no wait */
5. if timeout_ns == 0 { return Ok(0); }
6. /* Block until event or timeout */
7. let mut wait_entry = WaitEntry::new(current());
8. ep.wq.add_exclusive(&mut wait_entry);
9. loop {
   - if !ep.rdllist.is_empty() { break; }
   - if signal_pending(current()) { return Err(ERESTARTSYS); }
   - let res = schedule_hrtimeout(timeout_ns, HRTIMER_MODE_REL);
   - if res == 0 { /* timeout */ break; }
10. }
11. ep.wq.remove(&mut wait_entry);
12. /* Drain ready list */
13. EpollWait::ep_send_events(ep, events, maxevents)

`EpollWait::ep_send_events(ep, events, maxevents) -> KResult<i32>`:
1. let mut sent = 0;
2. let _lock = ep.mtx.lock();
3. let mut txlist = ep.rdllist.take();   /* move out for processing */
4. while let Some(epi) = txlist.pop_front() {
   - if sent >= maxevents { /* re-queue rest */ ep.rdllist.prepend(epi); break; }
   - let revents = ep_item_poll(epi, ep.poll_table);
   - if revents == 0 { continue; }   /* spurious */
   - let ev = EpollEvent { events: revents & epi.event.events, data: epi.event.data };
   - if copy_to_user_idx(events, sent, &ev).is_err() {
     - /* rollback: re-queue this epi */
     - ep.rdllist.prepend(epi);
     - if sent == 0 { return Err(EFAULT); } else { break; }
   - }
   - sent += 1;
   - /* Level-triggered: re-queue if still ready */
   - if !(epi.event.events & EPOLLET) && (revents & epi.event.events) != 0 {
     - ep.rdllist.push_back(epi);
   - }
   - /* ONESHOT: disarm */
   - if epi.event.events & EPOLLONESHOT != 0 {
     - epi.event.events &= EP_PRIVATE_BITS;
   - }
5. }
6. Ok(sent as i32)

### Out of Scope

- `epoll_create(2)` / `epoll_create1(2)` (Tier-5 separate docs).
- `epoll_ctl(2)` (Tier-5 separate doc — watch add/mod/del).
- `epoll_pwait(2)` (Tier-5 separate doc — modern variant with atomic sigmask).
- `epoll_pwait2(2)` (Tier-5 separate doc — nanosecond timespec timeout).
- RB-tree internals (Tier-3 in `lib/rbtree.md`).
- Implementation code.

### signature

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

### parameters

| Param | Type | Direction | Semantics |
|---|---|---|---|
| `epfd` | `int` | in | epoll file descriptor from `epoll_create(2)` / `epoll_create1(2)`. |
| `events` | `struct epoll_event __user *` | out | Buffer to receive ready events. Writable user memory. |
| `maxevents` | `int` | in | Max events to return; MUST be > 0 (typically 64..1024). |
| `timeout` | `int` | in | Milliseconds: `-1` = block indefinitely, `0` = poll, `>0` = bounded wait. |

### return value

| Value | Meaning |
|---|---|
| ≥ 0 | Number of events written to `events`. `0` ⇒ timeout expired with no events. |
| `-1` | Error; `errno` set. |

### errors

| `errno` | Cause |
|---|---|
| `EBADF` | `epfd` is not a valid open fd. |
| `EINVAL` | `epfd` is not an epoll fd; `maxevents <= 0`; `maxevents > EP_MAX_EVENTS`. |
| `EFAULT` | `events` buffer is not in caller's writable address space. |
| `EINTR` | Caller was interrupted by an unmasked signal before any event arrived. |

### abi surface

```text
__NR_epoll_wait  (x86_64) = 232
__NR_epoll_wait  (i386)   = 256
__NR_epoll_wait  (arm64)  = NOT IMPLEMENTED   (arm64 only provides epoll_pwait)
__NR_epoll_wait  (generic)= NOT IMPLEMENTED

struct epoll_event {
    __poll_t   events;   /* bitmask: EPOLLIN/OUT/PRI/ERR/HUP/RDHUP/ET/ONESHOT/WAKEUP/EXCLUSIVE */
    epoll_data_t data;   /* opaque user cookie */
} __attribute__((__packed__));    /* x86_64 only; aligned elsewhere */

#define EP_MAX_EVENTS    (INT_MAX / sizeof(struct epoll_event))   /* ~268M */

/* Event bits returned: */
EPOLLIN        0x00000001
EPOLLPRI       0x00000002
EPOLLOUT       0x00000004
EPOLLERR       0x00000008
EPOLLHUP       0x00000010
EPOLLRDNORM    0x00000040
EPOLLRDBAND    0x00000080
EPOLLWRNORM    0x00000100
EPOLLWRBAND    0x00000200
EPOLLMSG       0x00000400
EPOLLRDHUP     0x00002000
EPOLLET        0x80000000   /* set at ADD; returned for caller convenience */
EPOLLONESHOT   0x40000000
EPOLLEXCLUSIVE 0x10000000
EPOLLWAKEUP    0x20000000   /* prevents suspend; CAP_BLOCK_SUSPEND required */
```

### compatibility contract

REQ-1: Syscall number is **232** on x86_64; **256** on i386; **NOT exposed** on arm64/generic (modern arches only provide `epoll_pwait`). Kernel uses `__ARCH_WANT_SYS_EPOLL_WAIT`.

REQ-2: `maxevents` MUST be > 0 and ≤ `EP_MAX_EVENTS` (`INT_MAX / sizeof(struct epoll_event)`). Otherwise `EINVAL`.

REQ-3: `events` MUST be a writable user buffer of at least `maxevents * sizeof(struct epoll_event)` bytes. Pre-validate via PaX UDEREF. `EFAULT` on bad pointer.

REQ-4: `timeout = -1`: block until at least one event or signal; `EINTR` on signal.

REQ-5: `timeout = 0`: poll the ready list, return immediately. `0` events ⇒ return `0`, NOT `EAGAIN`. NO blocking.

REQ-6: `timeout > 0`: bounded wait in milliseconds; jiffies-rounded UP. Returns ≥ 0 events on event arrival, `0` on timeout expiry, `-EINTR` on signal.

REQ-7: Caller's epoll fd MUST be `EFD_RDWR`-equivalent (anon_inode epoll); other fd types ⇒ `EINVAL`.

REQ-8: Level-triggered (no `EPOLLET` set on watch): the fd remains in `rdllist` while the condition persists; subsequent `epoll_wait` re-returns it until the condition clears or watch is removed.

REQ-9: Edge-triggered (`EPOLLET` on watch): event reported exactly once per state-transition (e.g. non-readable → readable). Userspace MUST drain the fd to `EAGAIN` before next wait.

REQ-10: `EPOLLONESHOT`: after delivery, watch is auto-disarmed (`events &= ~EPOLLALL_ACTIVE`); userspace MUST `epoll_ctl(EPOLL_CTL_MOD)` to re-arm. Useful for thread-pools.

REQ-11: `EPOLLEXCLUSIVE`: at most one waiter on the same fd is woken per event (thundering-herd prevention). Set at `EPOLL_CTL_ADD`, MUST NOT be combined with `EPOLLET` or `EPOLLONESHOT` for some kernel versions (relaxed in 5.x).

REQ-12: Returned `epoll_event` records: `.events` carries the OR of currently-active mask bits intersected with the watched mask; `.data` is the cookie supplied at `EPOLL_CTL_ADD`/`MOD`.

REQ-13: `EFAULT` on partial copy_to_user: the kernel rolls back the event from `rdllist` re-arm logic so it is re-deliverable on next call. Atomicity is per-event (not per-batch).

REQ-14: Concurrent `epoll_wait` from multiple threads on the same epfd is safe; without `EPOLLEXCLUSIVE`, all waiters may be woken (thundering herd). With `EPOLLEXCLUSIVE`, exactly one.

REQ-15: Signal handling: blocked thread interrupted by an unmasked, non-SA_RESTART signal returns `EINTR`. With `SA_RESTART`, the kernel restarts via `ERESTARTSYS`. NOTE: legacy `epoll_wait` does NOT atomically install a sigmask; use `epoll_pwait` for atomic signal-mask + wait.

REQ-16: PaX UDEREF on `events` before `ep_send_events_proc` copies to userspace.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `maxevents_in_range` | INVARIANT | per-wait: 0 < maxevents ≤ EP_MAX_EVENTS. |
| `uderef_pre_copy` | INVARIANT | per-send_events: UDEREF passes before copy_to_user. |
| `rollback_on_fault` | INVARIANT | per-EFAULT: epi re-queued; no event lost. |
| `et_one_shot_per_edge` | INVARIANT | per-ET: event delivered once per state-transition. |
| `oneshot_disarms` | INVARIANT | per-ONESHOT: events &= EP_PRIVATE_BITS post-delivery. |
| `exclusive_one_waiter` | INVARIANT | per-EXCLUSIVE: at most one waiter woken. |

### Layer 2: TLA+

`fs/epoll-wait.tla`:
- States: per-epfd rdllist, per-watch event-mask, per-waiter wait-queue.
- Properties:
  - `safety_no_lost_events` — per-EFAULT: event remains observable on next wait.
  - `safety_level_triggered_persists` — per-LT: event re-returned while condition holds.
  - `safety_edge_triggered_unique` — per-ET: at most one delivery per transition.
  - `liveness_eventual_wake` — per-ready-fd: waiter eventually returned.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_epoll_wait` post: ret ≥ 0 events written | `EpollWait::do_epoll_wait` |
| `ep_poll` post: timeout honored | `EpollWait::ep_poll` |
| `ep_send_events` post: sent ≤ maxevents | `EpollWait::ep_send_events` |

### Layer 4: Verus / Creusot functional

Per-`epoll_wait(2)` man-page equivalence. LTP `epoll_wait01..05` pass. POSIX `poll(2)` analogous semantics for level-triggered cases.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`epoll_wait(2)` reinforcement:

- **Per-maxevents bounded** — defense against per-INT_MAX kernel-alloc DoS.
- **Per-rollback on copy_to_user fault** — defense against per-lost-event silent drop.
- **Per-rdllist mutex** — defense against per-concurrent-corruption.
- **Per-ET unique delivery** — defense against per-spurious-wakeup-storm.
- **Per-EXCLUSIVE one-waiter** — defense against per-thundering-herd.

### grsecurity / pax-style reinforcement

- **PaX UDEREF on `struct epoll_event *events`** — every copy_to_user(events) MUST traverse UDEREF SMAP/SMEP gate; rejects partial writes that would leak kernel residue in unwritten event slots. UDEREF is checked BEFORE any event is dequeued from rdllist to guarantee atomic rollback on fault.
- **EPOLL fd-lifetime refcount strict** — file refcount on epfd verified before each wait; any race that would close epfd mid-wait is caught and returns EBADF rather than UAF.
- **GRKERNSEC_HARDEN_PTRACE process-isolation** — peer ptracers cannot block waitqueue inspection of another task's epoll waitlist.
- **GRKERNSEC_PROC restrictions** — `/proc/<pid>/fdinfo/<epfd>` ready-list state is filtered for non-owners.
- **PIDFD_SELF safe self-target** — watching `pidfd_open(getpid(), 0)` for self-exit is supported; grsec ensures the resulting watch cannot create unbreakable shutdown loops.
- **signalfd CLOEXEC mandatory analog** — not applicable to `epoll_wait` itself, but the epfd's CLOEXEC default is enforced by grsec at create time so that exec'd children do not inherit epoll watching of the parent's fds.
- **GRKERNSEC_AUDIT_GROUP** — burst-rate epoll_wait calls with `maxevents = INT_MAX` from a marked group are rate-limited / audited to detect DoS attempts.
- **No_new_privs neutral** — NNP does not change epoll_wait semantics.
- **Anti-fingerprint hardening** — `epoll_event.data` is opaque user-cookie; never echo kernel pointers. The kernel guarantees `data` is the exact 64-bit value supplied by userspace at `EPOLL_CTL_ADD`/`MOD`.
- **Per-waiter EPOLLWAKEUP CAP_BLOCK_SUSPEND** — the EPOLLWAKEUP bit (preventing system suspend) requires CAP_BLOCK_SUSPEND or it is silently cleared; grsec audits attempts.
- **Seccomp interaction** — seccomp filters may explicitly block syscall 232 in favor of 281 (`epoll_pwait`); default-deny hardening recommends forbidding legacy variant entirely on arches that expose both.

