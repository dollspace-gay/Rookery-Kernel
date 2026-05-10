---
title: "Tier-3: fs/eventpoll.c — epoll (epoll_create / epoll_ctl / epoll_wait core implementation)"
tags: ["tier-3", "fs", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

`epoll(7)` is Linux's high-throughput level-triggered + edge-triggered IO multiplexing primitive (replacement for poll/select). User creates an epoll-fd via `epoll_create1`, registers per-monitored-fd events via `epoll_ctl(EPOLL_CTL_ADD/MOD/DEL)`, then calls `epoll_wait` to retrieve ready events. Internally each per-monitored-fd is wrapped in `struct epitem`; per-`epi` has a wait_queue_entry attached to the underlying file's poll-waitqueue; on file readiness change the file's waitqueue wake fires `ep_poll_callback` which moves epi to per-eventpoll ready-list + wakes any task in epoll_wait.

This Tier-3 covers `fs/eventpoll.c` (~2621 lines).

### Acceptance Criteria

- [ ] AC-1: Basic EPOLL_CTL_ADD + EPOLL_WAIT: socket pair; wait blocks until peer writes; returns ready-event matching written-fd.
- [ ] AC-2: EPOLLET: edge-triggered fd; first read returns data; second wait does not return ready (until further data).
- [ ] AC-3: EPOLLONESHOT: ready-event delivered once; subsequent waits do not return until ctl_mod re-arm.
- [ ] AC-4: EPOLLEXCLUSIVE: 100 threads wait on epoll-fd; one event arrives → only one thread wakes.
- [ ] AC-5: Cycle-detection: EPOLL_CTL_ADD epfd_a to monitor epfd_b; then add epfd_b to monitor epfd_a → ELOOP returned.
- [ ] AC-6: file-close cleanup: monitored fd closed; epitem auto-removed; subsequent epoll_wait does not deliver stale entry.
- [ ] AC-7: 1M-fd stress: register 1M sockets; intermediate readiness; epoll_wait consistently returns batches.
- [ ] AC-8: Mass concurrency: 32 threads doing epoll_ctl + epoll_wait on shared epfd; no UAF; no missed event.

### Architecture

`EventPoll`:

```
struct EventPoll {
  mtx: Mutex<()>,
  wq: WaitQueueHead,
  poll_wait: WaitQueueHead,
  rdllist: ListHead,                          // ready epitem's
  lock: RawSpinLock<()>,                       // protects rdllist + ovflist
  rbr: RbRoot<Epitem>,                         // all registered epi's
  ovflist: AtomicPtr<Epitem>,                  // overflow during ep_send_events
  user: KArc<UserStruct>,
  ws: Option<KArc<WakeupSource>>,
  file: KWeak<File>,
  nests: PerCpu<i32>,                          // recursion-depth
}

struct Epitem {
  rbn: RbNode,
  rdllink: ListNode,
  next: KAtomicPtr<Epitem>,                    // ovflist link
  ffd: EpollFileFd,                            // (file*, fd) key
  nwait: u32,
  ep: KArc<EventPoll>,
  event: EpollEvent,
  pwqlist: ListHead,
  fllink: ListNode,                            // f_ep list
}

struct EppollEntry {
  llink: ListNode,                              // epitem.pwqlist
  whead: *const WaitQueueHead,                  // file's poll waitqueue
  wait: WaitQueueEntry,                         // hooked into whead
  base: KWeak<Epitem>,
}
```

`EventPoll::insert(ep, &event, file, fd, full_check)`:
1. lock ep.mtx.
2. cycle-check via ep_loop_check_proc.
3. Allocate epi := KArc::new(Epitem).
4. epi.ffd = EpollFileFd { file: file.clone_weak(), fd }.
5. epi.event = event.
6. epi.ep = ep.clone().
7. Init pwqlist via init_poll_funcptr(&pt, ep_ptable_queue_proc).
8. Call file.f_op.poll(file, &pt) — for each call into ep_ptable_queue_proc:
   - Allocate eppoll_entry; hook into per-target waitqueue (with WQ_FLAG_EXCLUSIVE if EPOLLEXCLUSIVE).
   - Increment epi.nwait.
9. rb_insert by (file*, fd) key.
10. Add epi to file.f_ep list (so close cascade can find).
11. Get initial readiness: revents = ep_item_poll(epi).
12. If revents & event.events: ep_done(epi, revents) — add to rdllist + wake ep.wq.
13. unlock ep.mtx.

`EventPoll::poll_callback(wait, mode, sync, key)`:
1. epi := container_of(wait, EppollEntry, wait).base.
2. ep := epi.ep.
3. revents := key as u32.
4. If !(revents & epi.event.events): return 0 (not interesting).
5. lock ep.lock (irqsave; called from waker context).
6. If ep_send_events in progress (ovflist != EP_UNACTIVE_PTR):
   - If epi.next == EP_UNACTIVE_PTR: epi.next = ep.ovflist; ep.ovflist = epi.
7. Else if !list_empty(&epi.rdllink): /* already in ready-list */.
   Else: list_add_tail(&epi.rdllink, &ep.rdllist).
8. waiters-handle:
   - If !waitqueue_active(&ep.wq): nothing to wake.
   - Else: wake_up(&ep.wq); if poll_wait: wake_up(&ep.poll_wait).
9. unlock ep.lock.
10. Return 1.

`EventPoll::wait(ep, &events[], maxevents, &timeout)`:
1. Compute end-time (jiffies+timeout) for hrtimeout.
2. Loop:
   - lock ep.lock.
   - If !rdllist.empty: break.
   - prepare_to_wait_exclusive(&ep.wq, &wqe, TASK_INTERRUPTIBLE).
   - unlock.
   - schedule_hrtimeout_range(timeout, slack).
   - lock.
   - finish_wait.
   - If timeout reached: break.
   - If signal_pending: -EINTR.
3. ep_send_events(ep, events, maxevents).
4. unlock.

`EventPoll::send_events(ep, &events[], maxevents)`:
1. ep.ovflist = NULL (mark "draining").
2. unlock ep.lock.
3. Move ep.rdllist → txlist.
4. Per-epi in txlist:
   - revents := ep_item_poll(epi).
   - If revents & epi.event.events:
     - copy_to_user(events[count], &epi.event); count++.
     - If !(epi.event.events & EPOLLET):
       - re-add to rdllist (level-triggered).
     - If epi.event.events & EPOLLONESHOT:
       - Disable epi.
   - Else: re-arm OK; epi removed from rdllist.
5. lock ep.lock.
6. Move ovflist → rdllist.
7. ep.ovflist = EP_UNACTIVE_PTR.
8. Return count.

### Out of Scope

- VFS core (covered in `fs/vfs/00-overview.md`)
- Per-driver f_op->poll implementations
- io_uring (covered in `io_uring/00-overview.md`)
- aio (older async-IO)
- timerfd / signalfd / eventfd (covered in `fs/eventfd.md` future Tier-3)
- inotify / fanotify (covered separately)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct eventpoll` | per-epoll-fd state | `fs::eventpoll::EventPoll` |
| `struct epitem` | per-monitored-fd entry | `Epitem` |
| `struct ep_pqueue` | per-poll request adaptor | `EpPqueue` |
| `struct epoll_filefd` | (file*, fd) pair key | `EpollFileFd` |
| `do_epoll_create(flags)` | EPOLL_CREATE syscall impl | `EventPoll::create` |
| `do_epoll_ctl(epfd, op, fd, &event, nonblock)` | EPOLL_CTL syscall impl | `EventPoll::ctl` |
| `do_epoll_wait(epfd, &events[], maxevents, &to)` | EPOLL_WAIT syscall impl | `EventPoll::wait` |
| `do_epoll_pwait(...)` | _PWAIT variant | `EventPoll::pwait` |
| `ep_insert(ep, &epds, file, fd, full_check)` | EPOLL_CTL_ADD impl | `EventPoll::insert` |
| `ep_modify(ep, epi, &epds)` | EPOLL_CTL_MOD impl | `EventPoll::modify` |
| `ep_remove(ep, epi)` | EPOLL_CTL_DEL impl | `EventPoll::remove` |
| `ep_poll_callback(wait, mode, sync, key)` | per-fd waitqueue callback | `EventPoll::poll_callback` |
| `ep_send_events(ep, &events[], maxevents)` | drain ready-list to user | `EventPoll::send_events` |
| `ep_done(epi, eventmask)` | per-epi readiness latch | `Epitem::set_ready` |
| `ep_loop_check_proc(...)` | EPOLL_CTL_ADD cycle-detect | `EventPoll::loop_check_proc` |
| `ep_eventpoll_release(file)` | per-fd close cleanup | `EventPoll::release` |
| `reverse_path_check(...)` | EPOLLEXCLUSIVE consistency | `EventPoll::reverse_path_check` |
| `ep_busy_loop(ep, nonblock)` | NAPI-busy-poll integration | `EventPoll::busy_loop` |

### compatibility contract

REQ-1: Per-epoll-fd `eventpoll`:
- `mtx` (mutex; protects rb-tree mutation).
- `wq` (wait_queue_head; epoll_wait sleepers).
- `poll_wait` (wait_queue_head; for "epoll-fd is ready" notification when nested).
- `rdllist` (ready-list; epi's whose readiness fired).
- `lock` (raw_spinlock; protects ready-list).
- `rbr` (rb-root of all registered epi's).
- `rdllist_lock` (ratelimit; ovflist for nested wakes).
- `ovflist` (overflow-list during ep_send_events).
- `ws` (PM wakeup_source).
- `user` (user_struct refcount for memcg accounting).
- `file` (back-ref to anon-file).
- `nests` (per-CPU recursion-depth).

REQ-2: Per-monitored-fd `epitem`:
- `rbn` (rb_node in eventpoll.rbr).
- `rdllink` (list_node in eventpoll.rdllist or ovflist).
- `next` (pwq-list if multiple wait queue heads on same file).
- `ffd` (epoll_filefd: file* + fd).
- `nwait` (count of pwq's hooked into file's poll-waitqueues).
- `ep` (back-ref).
- `event` (epoll_event {events, data} mask).
- `pwqlist` (list of `eppoll_entry` for the file's poll-waitqueues).
- `fllink` (link in file.f_ep list — for finding all epoll-fds monitoring this file at file-close).

REQ-3: Per-pwq `eppoll_entry`:
- Wraps wait_queue_entry hooked into target file's poll-waitqueue.
- `whead` = target waitqueue.
- `base` = back-ref to epitem.

REQ-4: epoll_create1 flow:
1. Allocate eventpoll struct.
2. Anon-file via anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep, ...).
3. Return fd.

REQ-5: epoll_ctl(EPOLL_CTL_ADD) flow (`ep_insert`):
1. Lock ep.mtx.
2. cycle-check (ep_loop_check) for nested eventpoll-fd's.
3. Allocate epitem.
4. Init pwqs:
   - poll_wait_proc gets called for each waitqueue the file's f_op->poll uses.
   - Per waitqueue: alloc eppoll_entry; hook into target waitqueue.
5. rb_insert(&ep.rbr, &epi.rbn) keyed by (file*, fd).
6. Initial poll: call f_op->poll to get current readiness; if matching event mask: ep_done.
7. Unlock.

REQ-6: ep_poll_callback (called from target file's wake-up):
1. Read epi from waitqueue-entry private field.
2. ep := epi.ep.
3. Acquire ep.lock.
4. If epi already in rdllist: drop (already ready).
5. Else: list_add_tail(&epi.rdllink, &ep.rdllist).
6. If sleepers in ep.wq: wake_up_one (or wake_up_all for EPOLLEXCLUSIVE).
7. If poll_wait: wake (cascade for nested epoll).
8. Release ep.lock.
9. Return wakeup-disposition (1 if awake, 0 if not).

REQ-7: epoll_wait flow (`do_epoll_wait` → `ep_poll`):
1. Lock ep.mtx (or just lock).
2. If !rdllist.empty: ep_send_events(events[], maxevents).
3. Else: prepare_to_wait_exclusive(&ep.wq, &wqe, TASK_INTERRUPTIBLE).
4. schedule_hrtimeout (or hrtimeout_range for slack).
5. On wake: re-check rdllist + signal + timeout.
6. ep_send_events:
   - Move rdllist → txlist.
   - Per-epi in txlist: re-poll file's f_op->poll; if not ready: drop (level-triggered re-arm); else copy event to user + (if EPOLLET cleared) re-add to rdllist.
   - Set ovflist during walk so concurrent ep_poll_callback uses ovflist (no rdllist).

REQ-8: EPOLLET (edge-triggered):
- Per-epi event mask bit EPOLLET.
- After level-triggered drain, epi NOT re-added to rdllist; user must re-arm by re-reading.

REQ-9: EPOLLONESHOT:
- Single-fire; epi auto-disabled after first delivery.
- User must EPOLL_CTL_MOD to re-arm.

REQ-10: EPOLLEXCLUSIVE:
- Only one waiter woken per event (thundering-herd defense).
- pwq registered with WQ_FLAG_EXCLUSIVE.

REQ-11: Cycle-detection (nested epoll):
- EPOLL_CTL_ADD adding ep-fd to monitor in another ep-fd would create a graph; cycle would cause unbounded recursion in ep_poll_callback.
- ep_loop_check walks ep-graph DFS; rejects with ELOOP if cycle.

REQ-12: NAPI busy-poll integration:
- For network-heavy workloads, ep_busy_loop spins polling NICs directly bypassing wakeup overhead.
- Configured per-ep via SO_INCOMING_NAPI_ID + busy-poll-budget.

REQ-13: file-close cascade:
- f_op->release walks file.f_ep list; per-ep: ep_remove(ep, epi).

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rb_tree_invariants` | INVARIANT | per-ep.rbr keyed by (file*, fd); insertion + lookup ordered. |
| `pwq_list_no_uaf` | UAF | per-eppoll_entry held under whead.lock during walk; defense against close-during-wake. |
| `cycle_detect_terminates` | INVARIANT | EPOLL_CTL_ADD ep_loop_check bounded by EP_MAX_NESTS (default 4); defense against runaway recursion. |
| `ovflist_atomic_swap` | INVARIANT | per-ep.ovflist atomic ptr-swap; defense against torn-update during send_events. |
| `f_ep_list_synchronized` | INVARIANT | per-file f_ep list mutated under file_lock + ep.mtx. |

### Layer 2: TLA+

`fs/eventpoll/level_vs_edge.tla`:
- Per-epi state ∈ {Idle, Ready, Delivered}.
- EPOLLET vs level-triggered transitions.
- Properties:
  - `safety_level_re_arms` — level-triggered: Delivered → Ready if condition still holds.
  - `safety_edge_no_redelivery` — EPOLLET: Delivered → Idle until next condition-edge.
  - `liveness_event_eventually_observed` — assuming epoll_wait runs, Ready → Delivered.

`fs/eventpoll/cycle_check.tla`:
- DAG of nested ep-fd's.
- Properties:
  - `safety_no_cycle_after_check` — ep_loop_check rejects edge that would create cycle.
  - `safety_max_depth` — DFS bounded by EP_MAX_NESTS.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `EventPoll::insert` post: epi in rb-tree; pwq's hooked into per-file waitqueues | `EventPoll::insert` |
| `EventPoll::poll_callback` post: epi in rdllist xor ovflist (during send) xor stale; wakers notified | `EventPoll::poll_callback` |
| `EventPoll::send_events` post: per-epi event copied to user iff revents & event-mask; level-triggered re-armed | `EventPoll::send_events` |
| Per-ep.rdllist and ovflist disjoint | invariants on dispatch |
| Per-file.f_ep list contains all epi's referencing that file | `EventPoll::insert` / `EventPoll::release` |

### Layer 4: Verus/Creusot functional

`File-fd readiness change → ep_poll_callback → epitem in rdllist → epoll_wait returns event` semantic equivalence: per-readiness-change of monitored fd, ≤ 1 epitem-readiness-event delivered to one user-side ep_wait call (or all calls for non-EPOLLEXCLUSIVE).

### hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

eventpoll-specific reinforcement:

- **EP_MAX_NESTS bound (typically 4)** — defense against unbounded epoll-fd nesting causing stack overflow in poll_callback recursion.
- **Cycle detection via ep_loop_check** — defense against monitor-loop causing infinite-wakeup feedback.
- **ep.ovflist atomic-pointer-swap** — defense against rdllist torn-update during send_events.
- **Per-ep.user refcount via UserStruct** — defense against epoll DoS via unbounded epi-allocation per-user.
- **Per-epi nwait counter** capped — defense against single file with thousands of waitqueues exhausting memory.
- **EPOLLEXCLUSIVE thundering-herd defense** — defense against waking 1000 threads on single event.
- **f_ep list maintained at file-close** — defense against ep_poll_callback firing on freed file.
- **Per-pwq waitqueue-entry private field validated** — defense against attacker-controlled wait-queue entry pointing to crafted struct.
- **Per-ep.mtx + ep.lock split-lock discipline** — defense against deadlock between mutating-rb-tree (mtx) and waker (lock).
- **EPOLLONESHOT auto-disable atomic** — defense against double-delivery on rapid close-then-fire.
- **Per-eventpoll PM wakeup_source registration** — defense against suspend-while-pending events causing missed-wake.
- **NAPI busy-poll budget bounded** — defense against busy-loop spinning per-CPU > softirq-budget.

