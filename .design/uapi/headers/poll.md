# Tier-5 UAPI: include/uapi/asm-generic/poll.h — poll(2) / epoll(2) Bit ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/asm-generic/poll.h (~42 lines)
  - include/uapi/linux/eventpoll.h (EPOLL_CTL_* and EPOLL* bits, ~135 lines)
-->

## Summary

The poll/epoll UAPI is the syscall-edge contract for `poll(2)`, `ppoll(2)`, `select(2)`, `pselect(2)`, `epoll_create(2)`, `epoll_create1(2)`, `epoll_ctl(2)`, `epoll_wait(2)`, and `epoll_pwait(2)`/`epoll_pwait2(2)`. It defines:

- **Per-fd event-mask** `struct pollfd` (`fd`, `events`, `revents`) — the array element passed to `poll(2)` and embedded in epoll's per-fd state. 8 bytes per entry.
- **iBCS2-standard event bits** `POLLIN` (0x0001), `POLLPRI` (0x0002), `POLLOUT` (0x0004), `POLLERR` (0x0008), `POLLHUP` (0x0010), `POLLNVAL` (0x0020) — the bits userspace passes in `events` and the kernel returns in `revents`.
- **Extended (non-iBCS2) bits** `POLLRDNORM` (0x0040), `POLLRDBAND` (0x0080), `POLLWRNORM` (0x0100), `POLLWRBAND` (0x0200), `POLLMSG` (0x0400), `POLLREMOVE` (0x1000), `POLLRDHUP` (0x2000). Some are `#ifndef`'d per-arch (POLLWRNORM/POLLWRBAND/POLLMSG/POLLREMOVE/POLLRDHUP may be overridden by `<asm/poll.h>`).
- **Kernel-internal-only bits** `POLLFREE` (0x4000) and `POLL_BUSY_LOOP` (0x8000) — marked `__force __poll_t`; never expected from userspace.
- **`epoll_ctl(2)` opcodes** `EPOLL_CTL_ADD` (1), `EPOLL_CTL_DEL` (2), `EPOLL_CTL_MOD` (3) — operation argument to `epoll_ctl`.
- **Epoll-specific behavior bits** ORed into `epoll_event.events`: `EPOLLET` (1u << 31, edge-triggered), `EPOLLONESHOT` (1u << 30, fire-once and disarm), `EPOLLWAKEUP` (1u << 29, hold a wakeup_source while pending — needs `CAP_BLOCK_SUSPEND`), `EPOLLEXCLUSIVE` (1u << 28, exclusive wake-one semantics), `EPOLLURING_WAKE` (1u << 27, io_uring wake-fast-path marker, kernel-internal).
- **`struct epoll_event`** carrying `__poll_t events` (u32) + `__u64 data` (opaque cookie) — packed-attributed on x86 (8 + 8 = 12 bytes packed; 8 + 8 + 4 pad = 16 bytes naturally aligned).

Critical for: every reactor (`libevent`, `libev`, `libuv`), every async runtime (tokio, asyncio's selector backend), every network server (nginx, haproxy, envoy), every event-driven daemon (systemd, dbus), every userspace driver doing fd-multiplexing. The bit-table is **frozen ABI** — programs from 1993 with `POLLIN | POLLOUT` still work today because these constants have never changed.

This Tier-5 covers `include/uapi/asm-generic/poll.h` (~42 lines) and the directly-related `EPOLL_CTL_*` / `EPOLL*` constants from `include/uapi/linux/eventpoll.h`.

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct pollfd` | per-fd event-mask in poll(2) | `Pollfd` |
| `__poll_t` | per-event-mask typedef | `PollT` (`u32`) |
| `POLLIN..POLLBUSY_LOOP` | per-event bits | `Poll` constants |
| `POLLFREE` | per-internal "vma being freed" marker | `Poll::FREE` |
| `POLL_BUSY_LOOP` | per-internal "busy-poll requested" | `Poll::BUSY_LOOP` |
| `struct epoll_event` | per-fd state in epoll | `EpollEvent` |
| `EPOLL_CTL_ADD/MOD/DEL` | per-epoll_ctl cmd | `EpollCtlCmd` |
| `EPOLLIN..EPOLLET/...` | per-epoll_event.events bits | `Epoll` constants |
| `do_sys_poll` | per-poll(2) core | `Poll::do_sys_poll` |
| `do_pollfd` | per-fd evaluate in poll | `Poll::do_pollfd` |
| `ep_insert` / `ep_modify` / `ep_remove` | per-epoll_ctl arms | `Epoll::{insert,modify,remove}` |
| `ep_send_events` | per-ready-list drain to user | `Epoll::send_events` |
| `ep_poll` | per-epoll_wait core | `Epoll::poll` |
| `ep_poll_callback` | per-fd wakeup callback | `Epoll::poll_callback` |

## ABI surface (constants + structs)

### `struct pollfd` (`poll.h:36-40`)

```text
struct pollfd {
    int   fd;       /* file descriptor; negative = ignore */
    short events;   /* requested events (POLL* bits) */
    short revents;  /* returned events; kernel writes */
};
```

Size: 8 bytes (4 + 2 + 2). Naturally 4-byte-aligned. An entry with `fd < 0` is **ignored** by `poll(2)` (revents zeroed and entry skipped). The kernel writes only `revents`; userspace may read `events` to remember its request.

### iBCS2 event bits (`poll.h:6-11`)

```text
POLLIN    = 0x0001   /* data may be read without blocking */
POLLPRI   = 0x0002   /* urgent / out-of-band data available */
POLLOUT   = 0x0004   /* writable without blocking */
POLLERR   = 0x0008   /* error condition; always reported in revents */
POLLHUP   = 0x0010   /* hangup; always reported in revents */
POLLNVAL  = 0x0020   /* invalid fd; always reported in revents */
```

`POLLERR` / `POLLHUP` / `POLLNVAL` are **return-only**: userspace SHOULD NOT set them in `events` (kernel ignores them on input but always evaluates them on output regardless of `events`).

### Extended event bits (`poll.h:14-30`)

```text
POLLRDNORM = 0x0040   /* normal data may be read */
POLLRDBAND = 0x0080   /* priority data may be read */
POLLWRNORM = 0x0100   /* writing normal data won't block */
POLLWRBAND = 0x0200   /* writing priority data won't block */
POLLMSG    = 0x0400   /* SIGPOLL message available */
POLLREMOVE = 0x1000   /* request to remove (used internally by some drivers) */
POLLRDHUP  = 0x2000   /* peer closed read end (TCP half-close) */
```

`POLLWRNORM`/`POLLWRBAND`/`POLLMSG`/`POLLREMOVE`/`POLLRDHUP` are `#ifndef`'d so that arches with non-canonical values (e.g., sparc, alpha) may override them in `<asm/poll.h>`. The asm-generic file is the default.

### Kernel-internal bits (`poll.h:32-34`)

```text
POLLFREE       = 0x4000   /* __force __poll_t — never set by userspace */
POLL_BUSY_LOOP = 0x8000   /* __force __poll_t — busy-poll request marker */
```

`POLLFREE` is set by the kernel when an internal poll-wait-queue is being torn down (`__pollfd_check`) to signal upper-layer waiters. `POLL_BUSY_LOOP` is set by select/poll when `SO_BUSY_POLL` is enabled on a socket. Both bits are masked out of any user-supplied `events` word at syscall entry.

### `epoll_ctl(2)` opcodes (from `<linux/eventpoll.h>`)

```text
EPOLL_CTL_ADD = 1   /* add fd to epset */
EPOLL_CTL_DEL = 2   /* remove fd from epset */
EPOLL_CTL_MOD = 3   /* modify event mask for existing fd */
```

### Epoll-specific behavior bits ORed into `epoll_event.events`

```text
EPOLLEXCLUSIVE = 1u << 28   /* wake-one semantics; mutually exclusive with EPOLLONESHOT, EPOLLET on add */
EPOLLWAKEUP    = 1u << 29   /* hold wakeup_source while pending (CAP_BLOCK_SUSPEND) */
EPOLLONESHOT   = 1u << 30   /* fire once, then disarm; rearm via EPOLL_CTL_MOD */
EPOLLET        = 1u << 31   /* edge-triggered; default is level-triggered */
EPOLLURING_WAKE = 1u << 27  /* kernel-internal: io_uring wakeup-coalescing marker */
```

`EPOLLIN`/`OUT`/`PRI`/`ERR`/`HUP`/`RDHUP`/etc. use the same low-bit values as the corresponding `POLL*` constants — `epoll_event.events` is a `u32` superset.

### `struct epoll_event` (`<linux/eventpoll.h>`)

```text
struct epoll_event {
    __poll_t events;     /* EPOLL* bits | POLL* bits */
    __u64    data;       /* opaque user cookie */
}
__attribute__ ((__packed__))     /* on x86 only, to match historical glibc layout */
```

On x86 the struct is `__packed__` so `sizeof == 12`. On most other arches it is naturally aligned, `sizeof == 16`. This is the one place in the poll/epoll UAPI where Rookery MUST match Linux's per-arch packing decision.

### `epoll_create1(2)` flags

```text
EPOLL_CLOEXEC = O_CLOEXEC   /* set FD_CLOEXEC on the epoll fd */
```

## Compatibility contract

REQ-1: `struct pollfd` MUST be exactly 8 bytes: `int fd; short events; short revents;`. Field offsets MUST be 0/4/6. Endianness: native.

REQ-2: `POLLIN..POLLNVAL` (bits 0x01..0x20) MUST match iBCS2 exactly. Cannot change without ABI break of every networked program written since 1989.

REQ-3: `POLLERR | POLLHUP | POLLNVAL` are return-only. The kernel MUST report these in `revents` whenever the underlying condition holds, regardless of whether they were set in `events`. Userspace MUST be prepared to observe them on every poll-able fd.

REQ-4: `POLLRDNORM`/`POLLRDBAND`/`POLLWRNORM`/`POLLWRBAND`/`POLLMSG`/`POLLREMOVE`/`POLLRDHUP` use the asm-generic values unless `<asm/poll.h>` overrides for a given arch. Rookery on x86_64 follows asm-generic.

REQ-5: `POLLFREE = 0x4000` and `POLL_BUSY_LOOP = 0x8000` MUST be masked out of any user-supplied `events` word at syscall entry to prevent userspace from forging these internal-only signals.

REQ-6: `poll(2)` with `fd < 0` MUST silently ignore the entry (revents = 0) and not consume it. `fd == -1` is the canonical "skip" sentinel used by libc poll-set builders.

REQ-7: `poll(2)` MUST treat closed/invalid `fd` (where `fd >= 0` but not in the caller's fd table) by setting `POLLNVAL` in `revents` and **continuing** evaluation of the rest of the array — it MUST NOT return `-EBADF` for the whole poll set.

REQ-8: `poll(2)` `timeout_ms == 0` MUST poll-and-return-immediately. `timeout_ms < 0` MUST block indefinitely. `ppoll(2)` accepts a `struct timespec` and an optional `sigset_t` mask for atomic signal-mask install.

REQ-9: `epoll_create(2)` `size` argument is **ignored** since 2.6.8 (kept for ABI). `size <= 0` returns `-EINVAL`. `epoll_create1(0)` is the modern entry point; `epoll_create1(EPOLL_CLOEXEC)` sets FD_CLOEXEC.

REQ-10: `EPOLL_CTL_ADD` returns `-EEXIST` if `fd` is already in the set. `EPOLL_CTL_MOD` returns `-ENOENT` if `fd` is not in the set. `EPOLL_CTL_DEL` returns `-ENOENT` if `fd` is not in the set; `event` argument is unused (may be NULL since 2.6.9).

REQ-11: `EPOLLEXCLUSIVE` MAY be set only on `EPOLL_CTL_ADD`, not on `EPOLL_CTL_MOD`. Attempting to set it via MOD returns `-EINVAL`. It is mutually exclusive with `EPOLLEXCLUSIVE` already set on a sibling watching the same source.

REQ-12: `EPOLLWAKEUP` requires `CAP_BLOCK_SUSPEND`. Without the capability, the bit is silently cleared (not an error) for backwards compat.

REQ-13: `EPOLLURING_WAKE` is kernel-internal — userspace setting it in `epoll_event.events` MUST be masked off at syscall entry. It is reserved for io_uring's epoll-poll-multishot integration.

REQ-14: `struct epoll_event` MUST be `__packed__` on x86 (12 bytes) and naturally aligned (16 bytes) on most other arches. Rookery follows Linux per-arch.

REQ-15: `epoll_wait(2)` MUST return immediately if any fd is already ready. `maxevents <= 0` returns `-EINVAL`. `timeout_ms` semantics match `poll(2)`.

REQ-16: An fd MAY be added to multiple epoll sets simultaneously (epoll instances form a graph). The kernel MUST detect cycles via `ep_loop_check` and reject `EPOLL_CTL_ADD` with `-ELOOP` if adding the fd would create a cycle in the dependency graph.

REQ-17: Adding an epoll fd to itself MUST return `-EINVAL` (a depth-1 cycle).

REQ-18: The dependency graph depth MUST be bounded — exceeding `EP_MAX_NESTS` (currently 4) returns `-EINVAL` on add.

REQ-19: `EPOLL_CTL_MOD` MUST atomically swap the events mask and `data` payload (no partial-update race). Concurrent `epoll_wait` MUST either see the old or new (event, data) pair, never a torn read.

REQ-20: An fd closed via `close(2)` is **automatically removed** from every epoll set referencing it (the kernel ties this to the underlying `struct file` lifetime via the eventpoll waitqueue). Userspace need not call `EPOLL_CTL_DEL` before close.

## Acceptance Criteria

- [ ] AC-1: `sizeof(struct pollfd) == 8`; offsets: `fd@0`, `events@4`, `revents@6`.
- [ ] AC-2: `POLLIN==0x01, POLLPRI==0x02, POLLOUT==0x04, POLLERR==0x08, POLLHUP==0x10, POLLNVAL==0x20`.
- [ ] AC-3: `POLLRDNORM==0x40, POLLRDBAND==0x80, POLLWRNORM==0x100, POLLWRBAND==0x200, POLLMSG==0x400, POLLREMOVE==0x1000, POLLRDHUP==0x2000`.
- [ ] AC-4: `POLLFREE==0x4000, POLL_BUSY_LOOP==0x8000` masked out of user `events`.
- [ ] AC-5: `EPOLL_CTL_ADD==1, EPOLL_CTL_DEL==2, EPOLL_CTL_MOD==3`.
- [ ] AC-6: `EPOLLET == 1u<<31`, `EPOLLONESHOT == 1u<<30`, `EPOLLWAKEUP == 1u<<29`, `EPOLLEXCLUSIVE == 1u<<28`, `EPOLLURING_WAKE == 1u<<27`.
- [ ] AC-7: `poll(2)` with `fd == -1` entry: `revents == 0`, entry not counted in return value.
- [ ] AC-8: `poll(2)` with stale (closed) `fd >= 0`: `POLLNVAL` in `revents`, rest of array still evaluated.
- [ ] AC-9: `poll(2)` `timeout == 0` returns immediately; `timeout < 0` blocks; `timeout > 0` blocks ≤ that long.
- [ ] AC-10: `epoll_create1(EPOLL_CLOEXEC)` returns fd with `FD_CLOEXEC` set.
- [ ] AC-11: `EPOLL_CTL_ADD` of fd already in set returns `-EEXIST`.
- [ ] AC-12: `EPOLL_CTL_MOD` of fd not in set returns `-ENOENT`.
- [ ] AC-13: `EPOLL_CTL_DEL` of fd not in set returns `-ENOENT`.
- [ ] AC-14: `EPOLL_CTL_ADD` of epoll fd to itself returns `-EINVAL`.
- [ ] AC-15: Cycle in epoll dependency graph (A→B→A) returns `-ELOOP`.
- [ ] AC-16: Depth > `EP_MAX_NESTS` returns `-EINVAL`.
- [ ] AC-17: `EPOLLEXCLUSIVE` via `EPOLL_CTL_MOD` returns `-EINVAL`.
- [ ] AC-18: `EPOLLWAKEUP` without `CAP_BLOCK_SUSPEND` is silently cleared (no error).
- [ ] AC-19: Userspace-set `EPOLLURING_WAKE` is masked off at syscall entry.
- [ ] AC-20: `close(2)` of fd automatically removes it from every epoll set; subsequent `epoll_wait` does not report it.

## Architecture

```
pub const POLLIN:    u16 = 0x0001;
pub const POLLPRI:   u16 = 0x0002;
pub const POLLOUT:   u16 = 0x0004;
pub const POLLERR:   u16 = 0x0008;
pub const POLLHUP:   u16 = 0x0010;
pub const POLLNVAL:  u16 = 0x0020;
pub const POLLRDNORM:u16 = 0x0040;
pub const POLLRDBAND:u16 = 0x0080;
pub const POLLWRNORM:u16 = 0x0100;
pub const POLLWRBAND:u16 = 0x0200;
pub const POLLMSG:   u16 = 0x0400;
pub const POLLREMOVE:u16 = 0x1000;
pub const POLLRDHUP: u16 = 0x2000;
pub const POLLFREE:       u32 = 0x4000;   /* internal */
pub const POLL_BUSY_LOOP: u32 = 0x8000;   /* internal */

pub const POLL_USER_MASK: u16 = 0x3FFF;   /* userspace cannot set FREE/BUSY_LOOP */

#[repr(C)]
pub struct Pollfd {
    pub fd:      i32,
    pub events:  i16,
    pub revents: i16,
}

pub const EPOLL_CTL_ADD: i32 = 1;
pub const EPOLL_CTL_DEL: i32 = 2;
pub const EPOLL_CTL_MOD: i32 = 3;

pub const EPOLLURING_WAKE: u32 = 1 << 27;   /* kernel-internal */
pub const EPOLLEXCLUSIVE:  u32 = 1 << 28;
pub const EPOLLWAKEUP:     u32 = 1 << 29;
pub const EPOLLONESHOT:    u32 = 1 << 30;
pub const EPOLLET:         u32 = 1 << 31;

#[repr(C, packed)]   /* x86 */
pub struct EpollEvent {
    pub events: u32,    /* EPOLL* | POLL* */
    pub data:   u64,    /* opaque user cookie */
}
```

`Poll::do_pollfd(pfd: &mut Pollfd, wait: &PollTable) -> i16`:
1. /* Skip negative fd */
2. if pfd.fd < 0: pfd.revents = 0; return 0.
3. /* Resolve file; report POLLNVAL on bad fd */
4. let file = match fdget(pfd.fd) {
   Some(f) => f,
   None    => { pfd.revents = POLLNVAL as i16; return POLLNVAL as i16; }
}.
5. /* User-mask: strip kernel-internal bits */
6. let mask = (pfd.events as u32) & (POLL_USER_MASK as u32);
7. /* Always evaluate ERR/HUP/NVAL even if not requested */
8. let req = mask | (POLLERR | POLLHUP | POLLNVAL) as u32;
9. let r = file.poll(wait) & req;
10. pfd.revents = r as i16;
11. return r as i16.

`Poll::do_sys_poll(ufds: UserPtr<Pollfd>, nfds: u32, timeout_ns: Option<u64>) -> Result<i32>`:
1. if nfds > RLIMIT_NOFILE: return Err(EINVAL).
2. let fds = copy_pollfds_from_user(ufds, nfds)?.
3. let table = PollTable::new(timeout_ns).
4. loop {
     let mut count = 0;
     for pfd in &mut fds:
       if Poll::do_pollfd(pfd, &table) != 0: count += 1;
     if count > 0 || table.expired() || pending_signal(): break;
     table.wait();
   }.
5. copy_pollfds_to_user(ufds, &fds)?.
6. return Ok(count).

`Epoll::insert(ep: &Epoll, fd: i32, event: &EpollEvent) -> Result<()>`:
1. /* Mask kernel-internal bits */
2. let events = event.events & !EPOLLURING_WAKE.
3. /* CAP_BLOCK_SUSPEND for EPOLLWAKEUP */
4. let events = if (events & EPOLLWAKEUP) != 0 && !capable(CAP_BLOCK_SUSPEND) {
     events & !EPOLLWAKEUP
   } else { events }.
5. /* Disallow adding ep to itself */
6. if fd == ep.fd: return Err(EINVAL).
7. /* Cycle check */
8. if Epoll::loop_check(ep, fd)?.cycle: return Err(ELOOP).
9. if Epoll::depth(ep) >= EP_MAX_NESTS: return Err(EINVAL).
10. /* Duplicate check */
11. if ep.contains(fd): return Err(EEXIST).
12. /* Insert into RB-tree + register waitqueue callback */
13. ep.rb_insert(EpollItem { fd, events, data: event.data })?.
14. return Ok(()).

`Epoll::modify(ep: &Epoll, fd: i32, event: &EpollEvent) -> Result<()>`:
1. let item = ep.find(fd).ok_or(Err(ENOENT))?.
2. /* EPOLLEXCLUSIVE is add-only */
3. if event.events & EPOLLEXCLUSIVE != 0: return Err(EINVAL).
4. /* Mask kernel-internal */
5. let events = event.events & !EPOLLURING_WAKE.
6. /* Atomic swap */
7. WRITE_ONCE(&item.events, events).
8. WRITE_ONCE(&item.data,   event.data).
9. /* Re-check readiness */
10. Epoll::poll_callback(&item).
11. return Ok(()).

`Epoll::remove(ep: &Epoll, fd: i32) -> Result<()>`:
1. let item = ep.find(fd).ok_or(Err(ENOENT))?.
2. ep.rb_remove(&item).
3. waitqueue_unregister(&item.wq).
4. return Ok(()).

`Epoll::poll(ep: &Epoll, out: UserPtr<EpollEvent>, maxevents: i32, timeout_ns: Option<u64>) -> Result<i32>`:
1. if maxevents <= 0: return Err(EINVAL).
2. /* Wait for ready list to populate */
3. while ep.ready_list.is_empty():
   - if timeout_ns == Some(0): return Ok(0).
   - wait_event_timeout(ep.wq, !ep.ready_list.is_empty(), timeout_ns)?.
   - if pending_signal(): return Err(EINTR).
4. /* Drain ready list */
5. let count = Epoll::send_events(ep, out, maxevents)?.
6. return Ok(count).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pollfd_is_8` | INVARIANT | `sizeof::<Pollfd>() == 8`; offsets fd@0, events@4, revents@6. |
| `poll_bit_table` | INVARIANT | Every POLL* matches the canonical iBCS2 + Linux extension table. |
| `internal_bits_masked` | INVARIANT | per-do_pollfd: POLLFREE and POLL_BUSY_LOOP cleared from user events. |
| `err_hup_nval_always_evaluated` | INVARIANT | per-do_pollfd: result ANDs (mask | POLLERR | POLLHUP | POLLNVAL). |
| `negative_fd_skipped` | INVARIANT | per-do_pollfd: fd < 0 ⟹ revents = 0, count not incremented. |
| `bad_fd_yields_pollnval` | INVARIANT | per-do_pollfd: fdget None ⟹ revents has POLLNVAL bit. |
| `epoll_uring_wake_masked` | INVARIANT | per-insert/modify: EPOLLURING_WAKE cleared from user events. |
| `epoll_exclusive_add_only` | INVARIANT | per-modify: EPOLLEXCLUSIVE present ⟹ EINVAL. |
| `epoll_self_add_rejected` | INVARIANT | per-insert: fd == ep.fd ⟹ EINVAL. |
| `epoll_cycle_rejected` | INVARIANT | per-insert: would-cycle ⟹ ELOOP. |
| `epoll_depth_bounded` | INVARIANT | per-insert: depth(ep) >= EP_MAX_NESTS ⟹ EINVAL. |
| `epoll_event_packed_x86` | INVARIANT | on x86: `sizeof::<EpollEvent>() == 12` and field offsets 0, 4. |

### Layer 2: TLA+

`uapi/headers/poll.tla`:
- Per-poll(2): build fds → enter waitqueue → wake → drain → return count.
- Per-epoll: create → CTL_ADD → wait → MOD/DEL → close-fd auto-removes.
- Properties:
  - `safety_negative_fd_skipped` — per-pollfd: fd < 0 entries never report ready.
  - `safety_poll_nval_on_bad_fd` — per-pollfd: closed/invalid fd ⟹ POLLNVAL.
  - `safety_internal_bits_unforgeable` — userspace cannot set POLLFREE or POLL_BUSY_LOOP.
  - `safety_epoll_no_cycle` — epoll dependency graph is acyclic at all times.
  - `safety_epoll_remove_on_close` — when a watched fd's last reference is closed, it is removed from every epoll set referencing it.
  - `safety_epoll_modify_atomic` — concurrent epoll_wait never observes a torn (events, data) pair.
  - `liveness_epoll_wait_returns` — per-epoll_wait: eventually returns (ready, timeout, or signal).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Pollfd` layout post: 8 bytes, fd@0, events@4, revents@6 | `Pollfd` |
| `EpollEvent` layout post: x86 packed (12), other-arch natural (16) | `EpollEvent` |
| `Poll::do_pollfd` post: revents subset of (events | POLLERR | POLLHUP | POLLNVAL) | `Poll::do_pollfd` |
| `Poll::do_sys_poll` post: return count ∈ [0, nfds] | `Poll::do_sys_poll` |
| `Epoll::insert` post: ep contains fd; no cycle introduced | `Epoll::insert` |
| `Epoll::modify` post: events/data atomically swapped; no EPOLLEXCLUSIVE added | `Epoll::modify` |
| `Epoll::remove` post: ep no longer contains fd; waitqueue unregistered | `Epoll::remove` |
| `Epoll::poll` post: returned count ∈ [0, maxevents]; copied count events to user | `Epoll::poll` |

### Layer 4: Verus/Creusot functional

`Per poll(2) → fdget → file->poll(wait_table) → mask iBCS2 + Linux extensions → POLLERR/HUP/NVAL always-evaluated → drain or block` semantic equivalence: per-`Documentation/filesystems/locking.rst` (->poll method contract), per-POSIX `poll(2)`, per-Linux `poll(2)` man page §POLLERR notes.

`Per epoll_create1 → epoll_ctl(ADD|MOD|DEL) → epoll_wait → close-fd auto-remove` semantic equivalence: per-`Documentation/userspace-api/epoll.rst`, per-`fs/eventpoll.c` contract.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

poll/epoll UAPI reinforcement:

- **`POLLFREE` / `POLL_BUSY_LOOP` masked from user events** — defense against per-internal-signal forgery (the historical "vma teardown poll race" CVE-2022-0185-style class).
- **`EPOLLURING_WAKE` masked from user events** — defense against per-io_uring-fast-path-confusion.
- **`EPOLLWAKEUP` requires `CAP_BLOCK_SUSPEND`** — defense against per-unprivileged suspend-blocker.
- **`EPOLLEXCLUSIVE` add-only** — defense against per-MOD-induced thundering-herd-bypass.
- **Cycle check on `EPOLL_CTL_ADD`** — defense against per-recursive-epoll DoS / kernel-stack-exhaustion.
- **Depth bounded by `EP_MAX_NESTS`** — defense against per-deep-graph kernel-stack overrun.
- **Auto-remove on `close(2)`** — defense against per-stale-fd UAF in epoll's RB-tree.
- **`epoll_event.events` mask checked against known-bits per release** — defense against per-future-bit ambiguity.
- **`POLLNVAL` non-fatal** — defense against per-array-truncation in legacy polled multiplexers.
- **`epoll_event` x86-packed validated** — defense against per-userspace-padding-poisoning.

## Grsecurity/PaX-style Reinforcement

- **PaX UDEREF/USERCOPY** on every `copy_from_user` of `struct pollfd[]` and `struct epoll_event` and every `copy_to_user` of the same — defense against per-kernel-pointer dereference and per-OOB write through attacker-influenced `nfds` or `maxevents`. The pollfd-array copy is variable-length and is one of the historical hot-spots for `__copy_*_user` audit findings.
- **PAX_RANDKSTACK on syscall entry** — randomize kernel-stack offset on every `poll(2)`, `ppoll(2)`, `select(2)`, `pselect(2)`, `epoll_wait(2)`, `epoll_pwait(2)`, `epoll_pwait2(2)` entry so heap/stack grooming of the per-call `poll_wqueues` and `poll_table` structures (which live on-stack) is infeasible.
- **File-descriptor lifetime strictly refcounted under epoll** — every `EPOLL_CTL_ADD` holds a `struct file` reference via `fget`; the reference is released only when `EPOLL_CTL_DEL` runs or the underlying file's last user closes. Defense against per-epoll-UAF (CVE-2020-0423-class on aio/eventpoll fd-lifetime confusion).
- **`close(2)` → auto-remove from all epoll sets** — backed by the eventpoll waitqueue list attached to `struct file`. The kernel walks every `ep_pwq` on `__fput` and removes the corresponding `epitem`. Defense against per-dangling-epitem reference after fd reuse.
- **Cycle detection (`ep_loop_check`) under `epnested_mutex`** — strict, no time-bound bypass; rejects with `-ELOOP` rather than crashing. Defense against per-recursive-epoll kernel-stack exhaustion (CVE-2010-3850-style).
- **Depth cap `EP_MAX_NESTS == 4`** — bounded recursion through nested epoll instances. Defense against per-pathological-graph kernel-stack overrun.
- **`epoll_ctl` event mask sanitized** — kernel-internal bits (`EPOLLURING_WAKE`, `POLLFREE`, `POLL_BUSY_LOOP`) are cleared from user-supplied `events` before storage. Defense against per-userspace-forging-internal-events to confuse upper-layer dispatch.
- **`POLLERR`/`POLLHUP`/`POLLNVAL` always evaluated regardless of `events`** — userspace cannot mask out error conditions to ignore them. Defense against per-app-induced infinite-loop on POLLHUP/POLLNVAL where a malicious file-handle would otherwise be silently retried.
- **`EPOLLWAKEUP` requires `CAP_BLOCK_SUSPEND`** — silently cleared if missing the capability (no error to preserve ABI), preventing per-unprivileged-suspend-blocker DoS that would drain battery on mobile devices.
- **Per-pid `RLIMIT_NOFILE` enforced on poll `nfds`** — `poll(2)` with `nfds > RLIMIT_NOFILE` returns `-EINVAL`; defense against per-unbounded-stack-buffer allocation in the per-call poll-table.
- **`epoll_create1` flags validated** — only `EPOLL_CLOEXEC` permitted; any other bit returns `-EINVAL`. Defense against per-future-bit ambiguity and per-fd-leak-via-non-cloexec on suid binaries.
- **GRKERNSEC_PROC restrictions** on `/proc/<pid>/fdinfo/<fd>` (which exposes the per-epoll `tfd` watchlist) — defense against per-unprivileged enumeration of which fds a privileged process is multiplexing.
- **`EPOLL_CTL_ADD` audit logging** on suid-binary callers — record (target fd kind, events mask) per add; defense against per-stealth-monitoring of privileged daemon's fd set.

## Open Questions

(none at this Tier-5 level — `fs/eventpoll.c` and `fs/select.c` cover Tier-3 implementation semantics)

## Out of Scope

- `include/uapi/linux/eventpoll.h` full surface (covered here for `EPOLL_CTL_*` / `EPOLL*` bits only; the remainder lives in `uapi/headers/eventpoll.md`).
- `<asm/poll.h>` per-arch overrides (covered in per-arch UAPI docs).
- `select(2)` / `pselect(2)` fd_set ABI (covered in `uapi/headers/select.md`).
- `fs/eventpoll.c` implementation (covered in `fs/eventpoll.md` Tier-3).
- `fs/select.c` implementation (covered in `fs/select.md` Tier-3).
- `io_uring` IORING_OP_POLL_ADD/REMOVE (covered in `uapi/headers/io_uring.md`).
- Implementation code.
