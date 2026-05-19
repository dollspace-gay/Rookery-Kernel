# Tier-3: fs/eventpoll.c — epoll internals (data structures, concurrency, wake-stampede prevention)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: fs/00-overview.md
upstream-paths:
  - fs/eventpoll.c (~2621 lines)
  - include/linux/eventpoll.h
  - include/uapi/linux/eventpoll.h
  - include/linux/poll.h
  - include/linux/wait.h
-->

## Summary

This Tier-3 is the **internals** companion to `fs/eventpoll.md` (which sketches the syscall surface). Here we dissect the in-kernel objects (`struct eventpoll`, `struct epitem`, `struct eppoll_entry`, `struct epitems_head`), the three-level lock hierarchy (`epnested_mutex` ⊐ `ep->mtx` ⊐ `ep->lock`), the **dual ready-list discipline** (`rdllist` + `ovflist` LIFO) that lets `ep_poll_callback` enqueue work from IRQ context while `ep_send_events` drains a stolen list lock-free, the wake-stampede mitigations (`EPOLLEXCLUSIVE` + `add_wait_queue_exclusive` + `ep_autoremove_wake_function`), the **edge-trigger / level-trigger / oneshot** state machine in `ep_send_events`, the loop-detection algorithm (`ep_loop_check_proc` + `ep_get_upwards_depth_proc` + reverse-path budget), and the destruction protocol (`ep_clear_and_put` + `eventpoll_release_file` + `kfree_rcu`). Critical for: correctness under high-fanout TCP servers, thundering-herd suppression on accept-sockets, EPOLLET semantics, and freedom from epoll-fd cycles.

This Tier-3 covers `fs/eventpoll.c` (~2621 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct eventpoll` | per-epoll-fd ctx (rb-tree root, rdllist, ovflist, wq, mtx, lock, refs, gen) | `EventPoll` |
| `struct epitem` | per-monitored-fd entry (rbn, rdllink, next, ffd, pwqlist, ws, fllink, event) | `Epitem` |
| `struct eppoll_entry` | per-wq-attachment (wait, whead, base, next) | `EpPollEntry` |
| `struct epitems_head` | per-non-epoll-file fanout anchor (epitems hlist + next) | `EpitemsHead` |
| `struct epoll_filefd` | rb-tree key = (file*, fd) | `EpollFileFd` |
| `struct ep_pqueue` | poll_table wrapper carrying owning epi | `EpPQueue` |
| `ep_poll_callback(wait, mode, sync, key)` | IRQ-safe wakeup hook | `EventPoll::poll_callback` |
| `ep_ptable_queue_proc(file, whead, pt)` | per-EPOLL_CTL_ADD wq attach | `EventPoll::ptable_queue_proc` |
| `ep_start_scan(ep, txlist)` | steal-rdllist + freeze-ovflist | `EventPoll::start_scan` |
| `ep_done_scan(ep, txlist)` | drain-ovflist + re-inject-residual | `EventPoll::done_scan` |
| `ep_send_events(ep, evs, max)` | EPOLLET/LT/ONESHOT state machine | `EventPoll::send_events` |
| `ep_poll(ep, evs, max, to)` | wait-and-harvest core | `EventPoll::poll` |
| `ep_autoremove_wake_function(wq, mode, sync, key)` | self-removing wake function | `EventPoll::autoremove_wake_fn` |
| `ep_loop_check(ep, to)` | cycle / depth guard for nested epoll | `EventPoll::loop_check` |
| `ep_loop_check_proc(ep, depth)` | per-rbtree DFS cycle visit | `EventPoll::loop_check_proc` |
| `ep_get_upwards_depth_proc(ep, depth)` | per-`refs` DFS upward depth | `EventPoll::upward_depth_proc` |
| `reverse_path_check()` / `reverse_path_check_proc(refs, depth)` | per-`tfile_check_list` wakeup-path budget | `EventPoll::reverse_path_check` |
| `attach_epitem(file, epi)` / `list_file` / `unlist_file` | per-target-file fanout list | `EventPoll::attach_epitem` |
| `ep_poll_safewake(ep, epi, pollflags)` | nested-poll-wait safe wake | `EventPoll::poll_safewake` |
| `eventpoll_release_file(file)` | per-`__fput` cleanup hook | `EventPoll::release_file` |
| `ep_clear_and_put(ep)` | per-`epoll_fd` close drain | `EventPoll::clear_and_put` |
| `ep_remove(ep, epi)` / `ep_remove_epi` / `ep_remove_file` | per-CTL_DEL tear-down | `EventPoll::remove` |
| `ep_free(ep)` | per-rcu reclaim | `EventPoll::free` |
| `ep_busy_loop(ep)` / `ep_set_busy_poll_napi_id(epi)` | per-NAPI busy-poll | `EventPoll::busy_loop` |
| `ep_eventpoll_bp_ioctl(file, cmd, arg)` | EPIOCS/GPARAMS busy-poll knobs | `EventPoll::bp_ioctl` |
| `epnested_mutex` (global) | per-cycle-check serialisation | `EventPoll::NESTED_MUTEX` |
| `loop_check_gen` (global u64) | per-walk generation tag | `EventPoll::LOOP_CHECK_GEN` |
| `tfile_check_list` (global) | per-walk file-fanout chain | `EventPoll::TFILE_CHECK_LIST` |
| `epi_cache` / `pwq_cache` / `ephead_cache` | slab caches | `EventPoll::{EPI_CACHE,PWQ_CACHE,EPHEAD_CACHE}` |
| `EP_UNACTIVE_PTR` `((void *)-1L)` | sentinel for inactive ovflist / next | const `EP_UNACTIVE_PTR` |
| `EP_PRIVATE_BITS` `EPOLLWAKEUP\|EPOLLONESHOT\|EPOLLET\|EPOLLEXCLUSIVE` | mask of non-deliverable bits | const `EP_PRIVATE_BITS` |
| `EP_MAX_NESTS` `4` | per-nesting depth cap | const `EP_MAX_NESTS` |
| `path_limits[5] = {1000, 500, 100, 50, 10}` | per-depth wakeup-path budget | const `PATH_LIMITS` |

## Compatibility contract

REQ-1: `struct eventpoll` layout:
- `mtx`: mutex — protects rbr (RB-tree), pwqlist mutation, EPOLL_CTL_*, send-events scan, release path. Acquired in nested order via `mutex_lock_nested` keyed on epoll-tree depth (lockdep subkey).
- `wq`: wait_queue_head_t — `epoll_wait` sleepers.
- `poll_wait`: wait_queue_head_t — wakers waiting on the *epoll-fd itself* (nested-epoll case).
- `rdllist`: list_head — primary ready-list of epi; FIFO.
- `lock`: spinlock_t (IRQ-safe via spin_lock_irq / irqsave) — protects rdllist AND ovflist.
- `rbr`: rb_root_cached — keyed by (file*, fd) via `ep_cmp_ffd`.
- `ovflist`: struct epitem * — LIFO singly-linked overflow list active only while ep_send_events / __ep_eventpoll_poll holds `mtx` and has stolen rdllist; otherwise = `EP_UNACTIVE_PTR`.
- `ws`: wakeup_source * — top-level epoll wakeup source for `EPOLLWAKEUP`.
- `user`: user_struct * — per-user epoll-watches accounting.
- `file`: struct file * — back-pointer (epoll-fd's struct file).
- `gen`: u64 — last `loop_check_gen` this ep was visited in (memoise loop-check).
- `refs`: hlist_head — list of epitems that reference *this* ep as their target file (upward graph; used for reverse-path / upward-depth).
- `loop_check_depth`: u8 — memoised result of `ep_loop_check_proc(ep, _)` for current `loop_check_gen`.
- `refcount`: refcount_t — ep is alive while any epitem references it (refs >= 1) + the epoll-fd's file holds 1 + initial 1.
- `rcu`: rcu_head — defer free past `ep_get_upwards_depth_proc` RCU walk.
- `nests`: u8 (CONFIG_DEBUG_LOCK_ALLOC) — recorded nesting depth for `spin_lock_irqsave_nested`.
- `napi_id`, `busy_poll_usecs`, `busy_poll_budget`, `prefer_busy_poll` (CONFIG_NET_RX_BUSY_POLL) — busy-poll knobs.

REQ-2: `struct epitem` layout (cache-line-budgeted):
- `rbn` / `rcu` union — rbtree node before destruction, RCU head after.
- `rdllink`: list_head — link into `ep->rdllist`; `list_empty(&epi->rdllink)` ⇔ not ready.
- `next`: struct epitem * — link into `ep->ovflist`; `EP_UNACTIVE_PTR` ⇔ not on ovflist.
- `ffd`: epoll_filefd — (file, fd) rbtree key.
- `pwqlist`: eppoll_entry * — head of attached wq entries (one per file's wakeup queue).
- `ep`: struct eventpoll * — owning ep.
- `fllink`: hlist_node — link into the target file's `f_ep` fanout (`epitems_head::epitems` or `eventpoll::refs`).
- `ws`: wakeup_source * (rcu) — per-epi wakeup source for `EPOLLWAKEUP`.
- `event`: epoll_event — user-supplied {events, data}; `events` includes EP_PRIVATE_BITS.

REQ-3: `struct eppoll_entry`:
- `next`: per-epi singly-linked chain.
- `base`: back-pointer to owning epi.
- `wait`: wait_queue_entry_t with `wait.func = ep_poll_callback`.
- `whead`: wait_queue_head_t * (load via smp_load_acquire; `POLLFREE` path stores NULL via smp_store_release).

REQ-4: Lock hierarchy (acquire top-down only):
- L1 `epnested_mutex` (global) — for `EPOLL_CTL_ADD` of an epoll-fd into an epoll-fd: serialises cycle checks; held across ep_loop_check + reverse_path_check.
- L2 `ep->mtx` (per-ep) — protects rbtree mutation, send-events scan, release path. May be acquired in pairs in nested-epoll order via `mutex_lock_nested(..., depth)`.
- L3 `ep->lock` (per-ep spinlock) — IRQ-safe; protects rdllist + ovflist; held by `ep_poll_callback` (possibly from IRQ) and by start_scan / done_scan / ep_insert / ep_modify when they touch rdllist.
- `wait_queue_head_t::lock` (per-wq) — held inside ep_poll_safewake nested via `spin_lock_irqsave_nested(..., nests)`.
- `file->f_lock` — protects file->f_ep singleton mutation.

REQ-5: `ep_poll_callback(wait, mode, sync, key)` — IRQ-safe wake hook:
- epi = ep_item_from_wait(wait); ep = epi->ep; pollflags = key_to_poll(key).
- spin_lock_irqsave(&ep->lock).
- ep_set_busy_poll_napi_id(epi).
- /* EPOLLONESHOT-quiesced: events mask cleared except EP_PRIVATE_BITS */
- if !(epi->event.events & ~EP_PRIVATE_BITS): goto out_unlock.
- if pollflags ∧ !(pollflags & epi->event.events): goto out_unlock.
- /* Per-ovflist switch: send-events in progress */
- if READ_ONCE(ep->ovflist) != EP_UNACTIVE_PTR:
  - if epi->next == EP_UNACTIVE_PTR:                  /* not yet on ovflist */
    - epi->next = READ_ONCE(ep->ovflist).
    - WRITE_ONCE(ep->ovflist, epi).                   /* LIFO push */
    - ep_pm_stay_awake_rcu(epi).
- else if !ep_is_linked(epi):
  - list_add_tail(&epi->rdllink, &ep->rdllist).       /* FIFO append */
  - ep_pm_stay_awake_rcu(epi).
- /* Wake — EPOLLEXCLUSIVE filter */
- if waitqueue_active(&ep->wq):
  - if (epi->event.events & EPOLLEXCLUSIVE) ∧ !(pollflags & POLLFREE):
    - select ewake by pollflags ∩ EPOLLINOUT_BITS vs epi.event.events.
  - if sync: wake_up_sync(&ep->wq) else wake_up(&ep->wq).
- if waitqueue_active(&ep->poll_wait): pwake++.
- spin_unlock_irqrestore(&ep->lock).
- if pwake: ep_poll_safewake(ep, epi, pollflags & EPOLL_URING_WAKE).
- if !(epi->event.events & EPOLLEXCLUSIVE): ewake = 1.
- /* POLLFREE: target file is being freed; detach safely */
- if pollflags & POLLFREE:
  - list_del_init(&wait->entry).
  - smp_store_release(&ep_pwq_from_wait(wait)->whead, NULL).
- return ewake.

REQ-6: `ep_start_scan(ep, txlist)`:
- lockdep_assert_irqs_enabled.
- spin_lock_irq(&ep->lock).
- list_splice_init(&ep->rdllist, txlist).               /* steal ready list */
- WRITE_ONCE(ep->ovflist, NULL).                        /* activate ovflist */
- spin_unlock_irq(&ep->lock).

REQ-7: `ep_done_scan(ep, txlist)`:
- spin_lock_irq(&ep->lock).
- /* Drain LIFO ovflist back into rdllist; deduplicate against txlist */
- for nepi = READ_ONCE(ep->ovflist); (epi = nepi) != NULL; nepi = epi->next, epi->next = EP_UNACTIVE_PTR:
  - if !ep_is_linked(epi):
    - list_add(&epi->rdllink, &ep->rdllist).            /* reverse to FIFO */
    - ep_pm_stay_awake(epi).
- WRITE_ONCE(ep->ovflist, EP_UNACTIVE_PTR).             /* deactivate */
- list_splice(txlist, &ep->rdllist).                    /* residual */
- __pm_relax(ep->ws).
- if !list_empty(&ep->rdllist) ∧ waitqueue_active(&ep->wq): wake_up(&ep->wq).
- spin_unlock_irq(&ep->lock).

REQ-8: `ep_send_events(ep, events, maxevents)` — EPOLLET/LT/ONESHOT state machine:
- if fatal_signal_pending(current): return -EINTR.
- mutex_lock(&ep->mtx).
- ep_start_scan(ep, &txlist).
- list_for_each_entry_safe(epi, tmp, &txlist, rdllink):
  - if res >= maxevents: break.
  - /* Wakeup-source PM dance */
  - ws = ep_wakeup_source(epi).
  - if ws ∧ ws->active: __pm_stay_awake(ep->ws); __pm_relax(ws).
  - list_del_init(&epi->rdllink).                       /* remove from txlist */
  - revents = ep_item_poll(epi, &pt, 1) & epi->event.events.
  - if !revents: continue.                              /* spurious — drop, don't requeue */
  - events = epoll_put_uevent(revents, epi->event.data, events).
  - if !events:                                         /* copy_to_user failed */
    - list_add(&epi->rdllink, &txlist).                 /* re-inject for next call */
    - ep_pm_stay_awake(epi).
    - if !res: res = -EFAULT.
    - break.
  - res++.
  - /* ONESHOT: clear interest mask except private bits */
  - if epi->event.events & EPOLLONESHOT: epi->event.events &= EP_PRIVATE_BITS.
  - /* LT: re-queue for next epoll_wait so it re-polls */
  - else if !(epi->event.events & EPOLLET):
    - list_add_tail(&epi->rdllink, &ep->rdllist).
    - ep_pm_stay_awake(epi).
- ep_done_scan(ep, &txlist).
- mutex_unlock(&ep->mtx).
- return res.

REQ-9: `ep_poll(ep, events, maxevents, timeout)` — wait-and-harvest core:
- eavail = ep_events_available(ep).
- loop:
  - if eavail:
    - res = ep_try_send_events(ep, events, maxevents).
    - if res: return res.
  - if timed_out: return 0.
  - eavail = ep_busy_loop(ep).
  - if eavail: continue.
  - if signal_pending(current): return -EINTR.
  - init_wait(&wait); wait.func = ep_autoremove_wake_function.
  - spin_lock_irq(&ep->lock).
  - __set_current_state(TASK_INTERRUPTIBLE).
  - eavail = ep_events_available(ep).                   /* recheck under lock */
  - if !eavail: __add_wait_queue_exclusive(&ep->wq, &wait).
  - spin_unlock_irq(&ep->lock).
  - if !eavail: timed_out = !ep_schedule_timeout(to) ∨ !schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS).
  - __set_current_state(TASK_RUNNING).
  - eavail = 1.
  - if !list_empty_careful(&wait.entry):
    - spin_lock_irq(&ep->lock).
    - if timed_out: eavail = list_empty(&wait.entry).
    - __remove_wait_queue(&ep->wq, &wait).
    - spin_unlock_irq(&ep->lock).

REQ-10: `ep_autoremove_wake_function`:
- Calls `default_wake_function`; then `list_del_init_careful(&wq_entry->entry)` unconditionally — so a wakeup hands the event off to the *next* exclusive waiter without bouncing back. Crucial against thundering-herd on accept-sockets when N threads epoll_wait the same ep.

REQ-11: EPOLLEXCLUSIVE semantics (`EPOLL_CTL_ADD` only):
- Validated in `do_epoll_ctl`: rejected on `EPOLL_CTL_MOD`; rejected if target is itself an epoll-fd; rejected if events contain bits outside `EPOLLEXCLUSIVE_OK_BITS = EPOLLIN|EPOLLOUT|EPOLLERR|EPOLLHUP|EPOLLWAKEUP|EPOLLET|EPOLLEXCLUSIVE`.
- In `ep_ptable_queue_proc`: `add_wait_queue_exclusive(whead, &pwq->wait)` (vs `add_wait_queue` non-exclusive default) so the source file's `wake_up()` only wakes a single matching epitem.
- In `ep_poll_callback`: ewake flag is set per matched EPOLLIN/EPOLLOUT case; returned to the wq-walker so non-matching epitems can continue to receive the same wake (per `__wake_up_common` exclusive logic).
- In `ep_poll`: `__add_wait_queue_exclusive(&ep->wq, &wait)` — only one task is woken per `wake_up(&ep->wq)`.

REQ-12: EPOLLET (edge-triggered) semantics:
- In `ep_send_events`: after delivery, the epi is NOT re-queued onto rdllist; it can only be re-queued by a fresh `ep_poll_callback` (a new edge from the target file).
- In `ep_insert` and `ep_modify`: an `ep_item_poll` returning non-zero unconditionally queues the epi onto rdllist (initial edge).
- Consequence: an EPOLLET caller MUST drain the fd (read until EAGAIN); failing to do so is silently observable as "no more events", since no new edge fires until the file transitions empty→ready again.

REQ-13: EPOLLONESHOT semantics:
- In `ep_send_events`: after first delivery, `epi->event.events &= EP_PRIVATE_BITS` — effective interest mask becomes empty.
- In `ep_poll_callback`: gated on `!(epi->event.events & ~EP_PRIVATE_BITS)` — drops further wakeups.
- Re-armed only by an explicit `EPOLL_CTL_MOD` from userspace.

REQ-14: `ep_loop_check(ep, to)` — cycle and depth guard for `EPOLL_CTL_ADD` of one epoll-fd into another:
- Caller holds `epnested_mutex`. inserting_into = ep; loop_check_gen++ (in caller, do_epoll_ctl).
- depth = ep_loop_check_proc(to, 0).
- if depth > EP_MAX_NESTS: return -1.
- rcu_read_lock; upwards_depth = ep_get_upwards_depth_proc(ep, 0); rcu_read_unlock.
- if depth + 1 + upwards_depth > EP_MAX_NESTS: return -1.
- return 0.

REQ-15: `ep_loop_check_proc(ep, depth)` — DFS down through rbtree:
- if ep->gen == loop_check_gen: return ep->loop_check_depth.  /* memoise */
- mutex_lock_nested(&ep->mtx, depth + 1).
- ep->gen = loop_check_gen.
- for rbp in rb-tree:
  - epi = rb_entry(rbp).
  - if is_file_epoll(epi->ffd.file):
    - ep_tovisit = epi->ffd.file->private_data.
    - if ep_tovisit == inserting_into ∨ depth > EP_MAX_NESTS: result = EP_MAX_NESTS+1.
    - else: result = max(result, ep_loop_check_proc(ep_tovisit, depth+1) + 1).
    - if result > EP_MAX_NESTS: break.
  - else: list_file(epi->ffd.file).         /* register for reverse-path budget */
- ep->loop_check_depth = result.
- mutex_unlock(&ep->mtx).

REQ-16: `reverse_path_check()` — wakeup-path budget enforcement:
- For each epitems_head on the global `tfile_check_list` (registered by `list_file` during the downward DFS), recurse `reverse_path_check_proc(refs, 0)` walking the `ep->refs` upward graph; at each depth d, count emitting paths and verify `path_count[d] <= path_limits[d]` where `path_limits = {1000, 500, 100, 50, 10}`.
- Return -1 (translated to -EINVAL by ep_insert) if any depth exceeds budget.

REQ-17: `ep_insert(ep, &event, tfile, fd, full_check)`:
- Per-user watches accounting via `percpu_counter_compare(&ep->user->epoll_watches, max_user_watches)` — bail -ENOSPC.
- kmem_cache_zalloc(epi_cache).
- ep_set_ffd; epi->next = EP_UNACTIVE_PTR.
- attach_epitem(tfile, epi) — link onto file->f_ep (either tep->refs for an epoll-target or a freshly allocated epitems_head for a regular file).
- ep_rbtree_insert(ep, epi).
- ep_get(ep)                                        /* refcount += 1 for the epi */.
- if full_check ∧ reverse_path_check(): ep_remove; return -EINVAL.
- if EPOLLWAKEUP: ep_create_wakeup_source(epi).
- init_poll_funcptr(&epq.pt, ep_ptable_queue_proc); revents = ep_item_poll(epi, &epq.pt, 1).
- spin_lock_irq(&ep->lock).
- if revents ∧ !ep_is_linked(epi): list_add_tail(&epi->rdllink, &ep->rdllist); wake_up(&ep->wq) etc.
- spin_unlock_irq(&ep->lock).
- if nested poll_wait active: ep_poll_safewake(ep, NULL, 0).

REQ-18: `ep_modify(ep, epi, &event)`:
- epi->event.events = event->events (smp_mb afterwards); epi->event.data = event->data.
- Recompute wakeup-source if EPOLLWAKEUP toggled.
- smp_mb to ensure ep_poll_callback / f_op->poll see the new mask before we sample readiness.
- if ep_item_poll(epi, &pt, 1) ∧ !ep_is_linked(epi): enqueue + wake.

REQ-19: `ep_remove(ep, epi)` / `ep_remove_epi` / `ep_remove_file`:
- ep_unregister_pollwait — detach all pwq via `ep_remove_wait_queue` (smp_load_acquire(whead); remove_wait_queue if non-NULL).
- epi_fget(epi) — `file_ref_get` on the target file; if zero, skip (eventpoll_release_file will clean up).
- ep_remove_file — clear file->f_ep singleton, hlist_del_rcu.
- ep_remove_epi — rb_erase_cached, list_del_init under ep->lock, wakeup_source_unregister(ws), kfree_rcu(epi).
- ep_refcount_dec_and_test(ep) — warn if rb-root not empty.

REQ-20: `eventpoll_release_file(file)` — invoked from `__fput` of any monitored file:
- Loops: take file->f_lock, dequeue an epi from file->f_ep, drop f_lock, ep_unregister_pollwait, ep_remove_file, ep_remove_epi under ep->mtx; if ref drops to zero, ep_free.

REQ-21: `ep_clear_and_put(ep)` — invoked from release of the epoll-fd itself:
- ep_poll_safewake(ep, NULL, 0) — wake any poll_wait sleepers.
- mutex_lock(&ep->mtx).
- Pass 1: walk rb-tree, ep_unregister_pollwait, cond_resched.
- Pass 2: walk rb-tree, ep_remove(ep, epi), cond_resched.
- mutex_unlock.
- if ep_refcount_dec_and_test: ep_free.

REQ-22: `ep_free(ep)` — RCU-deferred reclaim:
- ep_resume_napi_irqs(ep). mutex_destroy(&ep->mtx). free_uid(ep->user). wakeup_source_unregister(ep->ws). kfree_rcu(ep, rcu).

REQ-23: Per-busy-poll (`CONFIG_NET_RX_BUSY_POLL`):
- `ep_busy_loop(ep)` — if `napi_id_valid(ep->napi_id)` ∧ `ep_busy_loop_on(ep)`, call `napi_busy_loop(napi_id, ep_busy_loop_end, ep, prefer_busy_poll, budget)`; returns true iff events appeared during poll.
- `ep_set_busy_poll_napi_id(epi)` — records the NAPI id from a newly-ready socket into `ep->napi_id` (called by ep_poll_callback and ep_insert).
- `EPIOCSPARAMS` / `EPIOCGPARAMS` ioctls set `busy_poll_usecs`, `busy_poll_budget`, `prefer_busy_poll`; budgets above NAPI_POLL_WEIGHT require CAP_NET_ADMIN.

REQ-24: Per-`EPOLL_URING_WAKE` (urpoll):
- A wake-source bit (in pollflags) indicating the wake originated from io_uring. Propagated through `ep_poll_safewake(ep, epi, pollflags & EPOLL_URING_WAKE)` to the nested poll_wait so the upstream waiter can distinguish io_uring-originated readiness for sched accounting.

REQ-25: Per-POLLFREE handling:
- When a target file is being freed and the f_op->poll layer signals POLLFREE through `key`, `ep_poll_callback` MUST detach the wait_queue_entry from the source whead *without* relying on `whead` being valid afterwards: `list_del_init(&wait->entry)` directly, then `smp_store_release(&pwq->whead, NULL)`. The matching `ep_remove_wait_queue` performs `smp_load_acquire(&pwq->whead)` to observe the release.

REQ-26: Per-`refs` upward graph:
- For an epi whose target is itself an epoll-fd, `attach_epitem` links the epi onto the *target* ep's `refs` hlist. This is the upward edge used by `ep_get_upwards_depth_proc` and by `reverse_path_check_proc`.

REQ-27: Per-`tfile_check_list`:
- Singly-linked chain of epitems_head structures registered (via `list_file`) when `ep_loop_check_proc` walks down through non-epoll target files. Used by `reverse_path_check` to enforce per-depth wakeup-path budgets. Cleared by `clear_tfile_check_list` at the end of `do_epoll_ctl`. Protected by `epnested_mutex`.

REQ-28: Per-`ep_poll_safewake` (CONFIG_DEBUG_LOCK_ALLOC variant):
- Uses per-ep `nests` counter to drive `spin_lock_irqsave_nested(&ep->poll_wait.lock, flags, nests)` so lockdep can see the actual recursion depth instead of falsely complaining about same-class lock recursion.

## Acceptance Criteria

- [ ] AC-1: A monitored fd that becomes readable while an `epoll_wait` thread is sleeping causes exactly one `wake_up(&ep->wq)` and the thread harvests the event.
- [ ] AC-2: With `EPOLLEXCLUSIVE`, N threads in `epoll_wait` on the same ep, M source-fd wakes deliver each event to exactly one thread (no thundering herd).
- [ ] AC-3: With `EPOLLET`, two back-to-back wakes on the *same* readiness state deliver exactly one event; subsequent `epoll_wait` returns 0 / blocks until a new edge fires.
- [ ] AC-4: With `EPOLLONESHOT`, after first delivery a second source-fd wake produces no event until userspace issues `EPOLL_CTL_MOD` re-arming the mask.
- [ ] AC-5: `ep_send_events` setting `ep->ovflist = NULL` causes a concurrent `ep_poll_callback` to LIFO-push onto ovflist; `ep_done_scan` re-injects in reversed FIFO order.
- [ ] AC-6: `EPOLL_CTL_ADD` of ep_A into ep_B then ep_B into ep_A returns -ELOOP, no state leaked.
- [ ] AC-7: `EPOLL_CTL_ADD` creating a wakeup path graph exceeding `path_limits` at any depth returns -EINVAL.
- [ ] AC-8: `EPOLL_CTL_ADD` of self (epfd == fd) returns -EINVAL.
- [ ] AC-9: `EPOLL_CTL_MOD` with `EPOLLEXCLUSIVE` bit returns -EINVAL.
- [ ] AC-10: Closing the epoll-fd while other threads are mid-`epoll_wait` causes those threads to harvest 0 events and return cleanly; no UAF on `epi` / `ep`.
- [ ] AC-11: Closing a monitored fd triggers `eventpoll_release_file` which removes the associated epi from every ep that referenced it; no dangling pwq.
- [ ] AC-12: `POLLFREE` from a target file detaches the wait_queue_entry without dereferencing the freed whead afterwards.
- [ ] AC-13: An exclusive waiter woken via `ep_autoremove_wake_function` is removed from `ep->wq` even on default_wake_function failure (e.g., already woken).
- [ ] AC-14: `max_user_watches` accounting via `percpu_counter` is decremented on every successful `ep_remove`, even via `eventpoll_release_file`.
- [ ] AC-15: A signal pending on `epoll_wait` returns -EINTR before the `__add_wait_queue_exclusive` step.

## Architecture

```
struct EventPoll {
  mtx: Mutex,
  wq: WaitQueueHead,                       // epoll_wait sleepers (exclusive)
  poll_wait: WaitQueueHead,                // nested-epoll "epoll-fd is ready" waiters
  rdllist: ListHead<Epitem, rdllink>,      // FIFO ready list
  lock: SpinLock,                          // protects rdllist + ovflist; IRQ-safe
  rbr: RbRootCached<Epitem, rbn, ffd>,
  ovflist: AtomicPtr<Epitem>,              // EP_UNACTIVE_PTR | NULL | LIFO chain
  ws: Option<*WakeupSource>,
  user: *UserStruct,
  file: *File,
  gen: u64,                                // memo of last loop_check_gen
  refs: HListHead<Epitem, fllink>,         // upward edges from nested epoll
  loop_check_depth: u8,
  refcount: RefCount,
  rcu: RcuHead,
  #[cfg(NET_RX_BUSY_POLL)] napi_id: u32,
  #[cfg(NET_RX_BUSY_POLL)] busy_poll_usecs: u32,
  #[cfg(NET_RX_BUSY_POLL)] busy_poll_budget: u16,
  #[cfg(NET_RX_BUSY_POLL)] prefer_busy_poll: bool,
  #[cfg(DEBUG_LOCK_ALLOC)] nests: u8,
}

struct Epitem {
  rbn_or_rcu: union { RbNode, RcuHead },
  rdllink: ListHead,                       // ep->rdllist link
  next: AtomicPtr<Epitem>,                 // ep->ovflist link; EP_UNACTIVE_PTR if not on
  ffd: EpollFileFd,                        // (file*, fd) key
  pwqlist: Option<*EpPollEntry>,           // attached wq entries (1 per source whead)
  ep: *EventPoll,
  fllink: HListNode,                       // file->f_ep fanout link
  ws: Rcu<Option<*WakeupSource>>,
  event: EpollEvent,                       // {events incl EP_PRIVATE_BITS, data}
}

struct EpPollEntry {
  next: Option<*EpPollEntry>,
  base: *Epitem,
  wait: WaitQueueEntry,                    // wait.func = ep_poll_callback
  whead: AtomicPtr<WaitQueueHead>,         // release/acquire pair w/ POLLFREE path
}

const EP_UNACTIVE_PTR: *mut Epitem = -1isize as *mut Epitem;
const EP_PRIVATE_BITS: u32 = EPOLLWAKEUP | EPOLLONESHOT | EPOLLET | EPOLLEXCLUSIVE;
const EP_MAX_NESTS: usize = 4;
const PATH_LIMITS: [u32; 5] = [1000, 500, 100, 50, 10];
```

`EventPoll::poll_callback(wait, mode, sync, key) -> i32`:
1. epi = container_of_wait(wait); ep = epi.ep; pollflags = key_to_poll(key); ewake = 0.
2. ep.lock.lock_irqsave(&flags).
3. ep.set_busy_poll_napi_id(epi).
4. if epi.event.events & !EP_PRIVATE_BITS == 0: goto out_unlock.    /* ONESHOT-quiesced */
5. if pollflags != 0 ∧ (pollflags & epi.event.events) == 0: goto out_unlock.
6. if ep.ovflist.load(Relaxed) != EP_UNACTIVE_PTR:
   - if epi.next.load(Relaxed) == EP_UNACTIVE_PTR:
     - epi.next.store(ep.ovflist.load(Relaxed), Relaxed).
     - ep.ovflist.store(epi, Release).
     - ep.pm_stay_awake_rcu(epi).
7. else if !epi.is_linked():
   - ep.rdllist.list_add_tail(&epi.rdllink).
   - ep.pm_stay_awake_rcu(epi).
8. if ep.wq.is_active():
   - if (epi.event.events & EPOLLEXCLUSIVE != 0) ∧ (pollflags & POLLFREE == 0):
     - match pollflags & (EPOLLIN | EPOLLOUT):
       - EPOLLIN  if epi.event.events & EPOLLIN  != 0 => ewake = 1,
       - EPOLLOUT if epi.event.events & EPOLLOUT != 0 => ewake = 1,
       - 0 => ewake = 1,
       - _ => {}.
   - if sync != 0 { ep.wq.wake_up_sync() } else { ep.wq.wake_up() }.
9. let pwake = ep.poll_wait.is_active() as i32.
10. ep.lock.unlock_irqrestore(flags).
11. if pwake != 0: ep.poll_safewake(epi, pollflags & EPOLL_URING_WAKE).
12. if epi.event.events & EPOLLEXCLUSIVE == 0: ewake = 1.
13. if pollflags & POLLFREE != 0:
    - wait.entry.list_del_init().
    - container_of(wait, EpPollEntry, wait).whead.store(null_mut(), Release).
14. return ewake.

`EventPoll::send_events(ep, events, maxevents) -> i32`:
1. if fatal_signal_pending(current): return -EINTR.
2. let mut txlist = ListHead::new(); let mut res = 0i32; let mut pt = PollTable::null().
3. ep.mtx.lock().
4. EventPoll::start_scan(ep, &mut txlist).
5. for epi in txlist.iter_safe():
   - if res >= maxevents { break }.
   - /* PM dance */
   - if let Some(ws) = ep.wakeup_source(epi): if ws.active { ep.ws.stay_awake(); ws.relax(); }.
   - epi.rdllink.list_del_init().
   - revents = EventPoll::item_poll(epi, &mut pt, 1) & epi.event.events.
   - if revents == 0 { continue }.
   - events = epoll_put_uevent(revents, epi.event.data, events);
   - if events.is_null():
     - epi.rdllink.list_add(&txlist).
     - ep.pm_stay_awake(epi).
     - if res == 0 { res = -EFAULT }.
     - break.
   - res += 1.
   - if epi.event.events & EPOLLONESHOT != 0:
     - epi.event.events &= EP_PRIVATE_BITS.
   - else if epi.event.events & EPOLLET == 0:
     - epi.rdllink.list_add_tail(&ep.rdllist).
     - ep.pm_stay_awake(epi).
6. EventPoll::done_scan(ep, &mut txlist).
7. ep.mtx.unlock().
8. return res.

`EventPoll::start_scan(ep, txlist)`:
1. lockdep_assert_irqs_enabled().
2. ep.lock.lock_irq().
3. ep.rdllist.list_splice_init(txlist).
4. ep.ovflist.store(null_mut(), Release).                     /* activate */
5. ep.lock.unlock_irq().

`EventPoll::done_scan(ep, txlist)`:
1. ep.lock.lock_irq().
2. let mut cur = ep.ovflist.load(Acquire).
3. while !cur.is_null():
   - let epi = &*cur; let next = epi.next.load(Relaxed);
   - epi.next.store(EP_UNACTIVE_PTR, Relaxed).
   - if !epi.is_linked():
     - epi.rdllink.list_add(&ep.rdllist).                     /* reverse LIFO ⇒ FIFO */
     - ep.pm_stay_awake(epi).
   - cur = next.
4. ep.ovflist.store(EP_UNACTIVE_PTR, Release).
5. txlist.list_splice(&ep.rdllist).                           /* residual */
6. ep.ws.relax().
7. if !ep.rdllist.is_empty() ∧ ep.wq.is_active(): ep.wq.wake_up().
8. ep.lock.unlock_irq().

`EventPoll::poll(ep, events, maxevents, timeout) -> i32`:
1. lockdep_assert_irqs_enabled.
2. let (slack, to) = resolve_timeout(timeout); let mut timed_out = false; let mut eavail = ep.events_available().
3. loop:
   - if eavail:
     - res = ep.try_send_events(events, maxevents).
     - if res != 0 { return res }.
   - if timed_out { return 0 }.
   - eavail = ep.busy_loop(); if eavail { continue }.
   - if signal_pending(current) { return -EINTR }.
   - let mut wait = WaitQueueEntry::init(); wait.func = ep_autoremove_wake_function.
   - ep.lock.lock_irq().
   - set_current_state(TASK_INTERRUPTIBLE).
   - eavail = ep.events_available().
   - if !eavail { ep.wq.add_exclusive(&mut wait); }.
   - ep.lock.unlock_irq().
   - if !eavail:
     - timed_out = !schedule_timeout_check(to) ∨ !schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS).
   - set_current_state(TASK_RUNNING).
   - eavail = true.
   - if !wait.entry.is_empty_careful():
     - ep.lock.lock_irq().
     - if timed_out { eavail = wait.entry.is_empty(); }.
     - ep.wq.remove(&mut wait).
     - ep.lock.unlock_irq().

`EventPoll::loop_check(ep, to) -> Result<(), Loop>`:
1. INSERTING_INTO.store(ep, Relaxed).
2. let depth = EventPoll::loop_check_proc(to, 0).
3. if depth > EP_MAX_NESTS { return Err(Loop) }.
4. rcu_read_lock(); let up = EventPoll::upward_depth_proc(ep, 0); rcu_read_unlock().
5. if depth + 1 + up > EP_MAX_NESTS { return Err(Loop) }.
6. return Ok(()).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `lock_hierarchy_respected` | INVARIANT | epnested_mutex acquired before any ep->mtx; ep->mtx before ep->lock. |
| `ovflist_sentinel_disjoint` | INVARIANT | per-callback: epi.next ∈ {EP_UNACTIVE_PTR, valid Epitem*, NULL}; mutually exclusive with rdllink linkage. |
| `rdllink_disjoint_from_ovflist` | INVARIANT | per-callback: ep_is_linked(epi) ⊻ (epi.next != EP_UNACTIVE_PTR). |
| `start_scan_freezes_ovflist` | INVARIANT | post-start_scan: ovflist == NULL; rdllist empty (txlist owns it). |
| `done_scan_restores_unactive` | INVARIANT | post-done_scan: ovflist == EP_UNACTIVE_PTR. |
| `irq_safe_path` | INVARIANT | ep_poll_callback acquires ep->lock with spin_lock_irqsave; never blocks. |
| `epollexclusive_addqueue_exclusive` | INVARIANT | per-CTL_ADD with EPOLLEXCLUSIVE: add_wait_queue_exclusive (not add_wait_queue). |
| `epollexclusive_blocked_on_mod` | INVARIANT | per-CTL_MOD with EPOLLEXCLUSIVE bit: -EINVAL. |
| `epollet_no_relist` | INVARIANT | per-send_events: epi with EPOLLET ∧ !EPOLLONESHOT does not re-enter rdllist. |
| `epolloneshot_mask_zeroed` | INVARIANT | per-send_events: after delivery, epi.event.events & ~EP_PRIVATE_BITS == 0. |
| `loop_check_caps_depth` | INVARIANT | per-CTL_ADD: depth + 1 + upwards_depth <= EP_MAX_NESTS. |
| `reverse_path_budget` | INVARIANT | per-CTL_ADD: at every depth d, path_count[d] <= path_limits[d]. |
| `pwq_whead_release_acquire` | INVARIANT | POLLFREE detach: smp_store_release(whead, NULL) pairs with smp_load_acquire in ep_remove_wait_queue. |
| `eventpoll_release_file_drains_all_epi` | INVARIANT | per-fput: file->f_ep empty on exit; every previously-linked epi removed from its owning ep. |
| `ep_refcount_balanced` | INVARIANT | every ep_get matched by ep_refcount_dec_and_test on the corresponding removal path. |
| `kfree_rcu_for_ep_and_epi` | INVARIANT | both ep and epi free via kfree_rcu, never plain kfree. |

### Layer 2: TLA+

`fs/eventpoll.tla`:
- Models: per-ep, per-epi, per-source-file readiness; ep_poll_callback (IRQ), ep_send_events (mtx), ep_poll (waiter), ep_insert/modify/remove (mtx), ep_clear_and_put (release).
- Properties:
  - `safety_no_lost_event` — per-source readiness transition: an epi observed ready at any point is either delivered or remains on rdllist/ovflist until next epoll_wait.
  - `safety_no_duplicate_event_et` — per-EPOLLET: between two source-readiness edges, at most one delivery per epi.
  - `safety_oneshot_quiesce` — per-EPOLLONESHOT: after first delivery, no further delivery until CTL_MOD.
  - `safety_exclusive_single_wake` — per-EPOLLEXCLUSIVE: per source wake-up event, exactly one epoll_wait waiter is woken (modulo waiter race).
  - `safety_no_loop` — per-CTL_ADD: graph remains acyclic and depth-bounded.
  - `safety_ovflist_consistent` — per-send_events: every epi pushed onto ovflist is re-injected into rdllist by done_scan.
  - `liveness_eventually_delivered` — per-pending-event with at least one epoll_wait: event delivered in finite steps.
  - `liveness_clear_and_put_terminates` — per-ep close: ep_clear_and_put finishes regardless of concurrent eventpoll_release_file.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `EventPoll::poll_callback` post: ovflist push XOR rdllist push (mutually exclusive) | `EventPoll::poll_callback` |
| `EventPoll::start_scan` post: ovflist == NULL ∧ rdllist.is_empty() | `EventPoll::start_scan` |
| `EventPoll::done_scan` post: ovflist == EP_UNACTIVE_PTR ∧ all ovflist members on rdllist | `EventPoll::done_scan` |
| `EventPoll::send_events` post: ret ∈ {0, -EFAULT, -EINTR, [1, maxevents]} | `EventPoll::send_events` |
| `EventPoll::poll` post: ret ∈ {0, -EINTR, [1, maxevents]} | `EventPoll::poll` |
| `EventPoll::loop_check` post: Ok(()) ⟹ resulting graph depth <= EP_MAX_NESTS | `EventPoll::loop_check` |
| `EventPoll::insert` post: epi on rb-tree ∧ ep.refcount += 1 ∧ percpu_counter += 1 | `EventPoll::insert` |
| `EventPoll::remove` post: epi off rb-tree, off rdllist, off ovflist, off file f_ep | `EventPoll::remove` |
| `EventPoll::release_file` post: file.f_ep is null ∨ first == null | `EventPoll::release_file` |
| `EventPoll::clear_and_put` post: rb-tree empty; ep freed iff refcount hits zero | `EventPoll::clear_and_put` |

### Layer 4: Verus/Creusot functional

`Per-epoll-fd state machine: epoll_create → N × epoll_ctl(ADD|MOD|DEL) → M × epoll_wait → close` is the trace semantics of `Documentation/filesystems/epoll.rst` and `man 7 epoll`. The ovflist + rdllist pair is functionally equivalent to a single FIFO queue with concurrent producer (ep_poll_callback) and consumer (ep_send_events); the LIFO push + reversal in done_scan preserves arrival order modulo concurrent inserts during the scan window.

## Hardening

(Inherits row-1 features from `fs/00-overview.md` § Hardening.)

eventpoll-internals reinforcement:

- **Per-three-level lock hierarchy** (`epnested_mutex` ⊐ `ep->mtx` ⊐ `ep->lock`) — defense against per-deadlock under nested-epoll.
- **Per-ovflist sentinel `EP_UNACTIVE_PTR`** — defense against per-double-enqueue + per-spurious-IRQ-context list traversal.
- **Per-IRQ-safe `spin_lock_irqsave` on ep->lock** — defense against per-IRQ-recursion deadlock when ep_poll_callback fires from device IRQ.
- **Per-EPOLLEXCLUSIVE `add_wait_queue_exclusive`** — defense against per-thundering-herd on accept-sockets.
- **Per-`ep_autoremove_wake_function`** — defense against per-wakeup-loss (woken-but-not-removed exclusive waiter racing a kill).
- **Per-`EP_MAX_NESTS = 4` recursion cap** — defense against per-stack-blast in cascaded poll wakes.
- **Per-`path_limits = {1000,500,100,50,10}`** — defense against per-wakeup-storm via maliciously-shaped epoll graphs.
- **Per-`ep_loop_check` cycle guard** — defense against per-deadlock from CTL_ADD cycles.
- **Per-`loop_check_gen` memoisation** — defense against per-exponential walk in deep DAGs.
- **Per-`POLLFREE` release-acquire on `pwq->whead`** — defense against per-UAF when a target file races free with ep_poll_callback.
- **Per-`kfree_rcu` for both ep and epi** — defense against per-UAF from `ep_get_upwards_depth_proc` RCU walkers.
- **Per-`percpu_counter` max_user_watches** — defense against per-user epoll-watch DoS.
- **Per-`f_ep` singleton under `file->f_lock`** — defense against per-double-attach race.
- **Per-`smp_mb` in `ep_modify`** — defense against per-missed-event on interest-mask narrowing.
- **Per-`EPIOCSPARAMS` CAP_NET_ADMIN gate on oversize budget** — defense against per-unprivileged busy-poll-storm.
- **Per-`ep_poll_safewake` `spin_lock_irqsave_nested`** — defense against per-lockdep false-positive blocking valid nested wakes.

## Grsecurity/PaX-style Reinforcement

This subsystem inherits the standard PaX/Grsecurity surface and reinforces it with:

- **PAX_USERCOPY** — bounded user-buffer copy on `ep_send_events` → `__put_user(struct epoll_event)` so a malicious `events` array cannot escape the user iov bound.
- **PAX_KERNEXEC** — W^X for any executable mapping touched by the wakeup chain.
- **PAX_RANDKSTACK** — per-syscall kernel-stack randomization across `epoll_wait` / `epoll_pwait2` so polling cadence cannot leak stack layout.
- **PAX_REFCOUNT** — saturating refcount on `struct eventpoll`, on every `struct epitem`, and on the per-task wakeup-chain depth counter; `ep_loop_check` cannot wrap.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `struct epitem` and `struct eppoll_entry` so stale `f_ep`/`fllink` pointers do not survive into reallocation.
- **PAX_UDEREF** — SMAP/SMEP strict user-pointer access; `ep_send_events` defers all user-side stores to a single bounded region.
- **PAX_RAP / kCFI** — indirect-call signature enforcement on `wait_queue_func_t` (`ep_poll_callback`) and on `wait_queue_entry.func` so an attacker who plants a function pointer in a free'd `eppoll_entry` cannot redirect control flow on wake.
- **GRKERNSEC_HIDESYM** — kernel pointer hiding for `/proc/<pid>/fdinfo/N` epoll entries.
- **GRKERNSEC_DMESG** — syslog restriction on epoll watch-limit and loop-check diagnostics.
- **EPOLL nesting depth bound (`EP_MAX_NESTS = 4`)** — `ep_loop_check_proc` rejects deeper epoll-on-epoll graphs so an attacker cannot blow the kernel stack via recursive wakeups.
- **EPOLLEXCLUSIVE wake-stampede prevention** — `ep_poll_callback` honors `EPOLLEXCLUSIVE` and `WQ_FLAG_EXCLUSIVE` so a single ready fd never wakes the entire watcher cohort on shared-listen-socket workloads.
- **POLLFREE release/acquire fence** — `ep_remove_wait_queue` pairs `smp_store_release(&epi->ws, NULL)` with the wakeup-side `smp_load_acquire` so a racing `close()` cannot leave a wait_queue head pointing into freed memory.
- **`percpu_counter` `max_user_watches`** — per-uid epitem cap enforced atomically; rlimit-like quota survives namespace traversal.
- **`f_ep` singleton under `file->f_lock`** — a given (epfd, target_fd) pair admits exactly one epitem, blocking double-attach UAF.
- **`EPIOCSPARAMS` CAP_NET_ADMIN gate** — oversize busy-poll budgets require capability so unprivileged tasks cannot pin a CPU spinning.

Per-doc rationale: the epoll internals own a wait-queue graph that crosses fd/file/task boundaries and runs callbacks from softirq, so a single stale pointer or missed barrier becomes a kernel-side UAF on a hot wakeup path; PaX/grsec reinforcement closes the gap between "POLLFREE was sent" and "the last softirq callback retired" with explicit refcount, fencing, and CFI rather than relying on the file table's release order.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- Syscall surface and userspace ABI (covered in `eventpoll.md` Tier-3)
- io_uring integration beyond the `EPOLL_URING_WAKE` flag (covered separately if expanded)
- NAPI / network busy-poll core (covered in `net/core/busy-poll.md` Tier-3 if expanded)
- `fs/select.c` / `fs/poll.c` legacy multiplexers (covered separately)
- `f_op->poll` per-driver implementations
- Implementation code
