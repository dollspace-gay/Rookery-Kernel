# Tier-3: io_uring/timeout.c — io_uring timeouts

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/timeout.c (~741 lines)
  - io_uring/timeout.h
  - include/uapi/linux/io_uring.h (IORING_OP_TIMEOUT, IORING_OP_TIMEOUT_REMOVE, IORING_OP_LINK_TIMEOUT, IORING_TIMEOUT_*)
-->

## Summary

`io_uring` exposes three timer opcodes — `IORING_OP_TIMEOUT`, `IORING_OP_TIMEOUT_REMOVE` and `IORING_OP_LINK_TIMEOUT` — that all share one core record: `struct io_timeout` (req-private) backed by `struct io_timeout_data` (async-data: hrtimer + clock + mode + flags). A bare `TIMEOUT` is either a pure countdown (no `off` ⟹ fire after `data->time` elapses) or a CQ-event-count gate (`off > 0` ⟹ fire when `off` non-timeout CQEs have posted since submit). `LINK_TIMEOUT` arms a per-link guardian: the timer fires only if the head of the link has not yet completed; on expiry it cancels that head with `-ECANCELED` and emits `-ETIME` for itself. `TIMEOUT_REMOVE` doubles as cancel-by-user-data ('UPDATE not set) and update-in-place ('UPDATE set with new `addr2`-supplied time + mode). Clock domains are selected by `IORING_TIMEOUT_BOOTTIME` / `_REALTIME` (default `CLOCK_MONOTONIC`) and pinned through `time_namespace`. `IORING_TIMEOUT_ABS` switches to `HRTIMER_MODE_ABS`. `IORING_TIMEOUT_ETIME_SUCCESS` reports `-ETIME` without setting `REQ_F_FAIL` (so the link does NOT abort). `IORING_TIMEOUT_MULTISHOT` re-arms on each expiry up to `repeats` (= `off` at prep time) with `IORING_CQE_F_MORE`. Two list anchors live on `io_ring_ctx`: `timeout_list` (regular timeouts, kept insertion-sorted by `target_seq` for the seq-gate variant) and `ltimeout_list` (in-flight linked timeouts). Locking: hrtimer side uses `timeout_lock` (raw spinlock IRQ); user-side updates and removes acquire `completion_lock` then `timeout_lock`. Critical for: bounded waits, link-failure recovery, deadline-driven workloads, cancel races (timer ↔ completion).

This Tier-3 covers `io_uring/timeout.c` (~741 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_timeout` | per-req timer record (off, target_seq, repeats, list, head, prev) | `IoTimeout` |
| `struct io_timeout_rem` | per-remove-or-update arg record | `IoTimeoutRem` |
| `struct io_timeout_data` | per-async-data (hrtimer + clock + mode + flags + time) | `IoTimeoutData` |
| `io_flags_to_clock()` | per-flag clockid_t selector | `Timeout::flags_to_clock` |
| `io_parse_user_time()` | copy_from_user timespec64 + ns_to_ktime + timens conversion | `Timeout::parse_user_time` |
| `io_is_timeout_noseq()` | per-`!off ∨ MULTISHOT`: bypass insertion sort | `Timeout::is_noseq` |
| `io_timeout_finish()` | per-multishot: decide rearm vs final | `Timeout::finish` |
| `io_timeout_fn()` | hrtimer expiry callback (regular timeout) | `Timeout::expire` |
| `io_link_timeout_fn()` | hrtimer expiry callback (linked timeout) | `Timeout::link_expire` |
| `io_timeout_complete()` | task-work: post -ETIME CQE, rearm if MULTISHOT | `Timeout::complete_tw` |
| `io_req_task_link_timeout()` | task-work: cancel link head, complete self | `Timeout::link_timeout_tw` |
| `io_flush_killed_timeouts()` | drain a local list → queue tw_complete | `Timeout::flush_killed` |
| `io_kill_timeout()` | hrtimer_try_to_cancel + move to local list (must_hold timeout_lock) | `Timeout::kill_one` |
| `io_flush_timeouts()` | per-CQE-post: walk timeout_list, fire seq-gated ones | `Timeout::flush_seq` |
| `io_disarm_next()` | per-completion: tear down linked timeout / fail links | `Timeout::disarm_next` |
| `__io_disarm_linked_timeout()` | per-link-head completion path | `Timeout::disarm_linked` |
| `io_timeout_extract()` | per-cancel: locate req by user_data, cancel hrtimer | `Timeout::extract` |
| `io_timeout_cancel()` | per-IORING_OP_TIMEOUT_REMOVE (no UPDATE) | `Timeout::cancel` |
| `io_timeout_update()` | per-IORING_OP_TIMEOUT_REMOVE (UPDATE, non-link) | `Timeout::update` |
| `io_linked_timeout_update()` | per-IORING_LINK_TIMEOUT_UPDATE | `Timeout::update_linked` |
| `io_translate_timeout_mode()` | per-IORING_TIMEOUT_ABS → HRTIMER_MODE_ABS / _REL | `Timeout::translate_mode` |
| `io_timeout_remove_prep()` | per-OP_TIMEOUT_REMOVE prep | `Timeout::remove_prep` |
| `io_timeout_remove()` | per-OP_TIMEOUT_REMOVE issue | `Timeout::remove_issue` |
| `__io_timeout_prep()` | per-OP_TIMEOUT / OP_LINK_TIMEOUT prep core | `Timeout::prep_inner` |
| `io_timeout_prep()` | per-OP_TIMEOUT prep | `Timeout::prep` |
| `io_link_timeout_prep()` | per-OP_LINK_TIMEOUT prep | `Timeout::link_prep` |
| `io_timeout()` | per-OP_TIMEOUT issue | `Timeout::issue` |
| `io_queue_linked_timeout()` | per-link-head-issued: arm guard timer | `Timeout::queue_linked` |
| `io_match_task()` | per-tctx / cancel_all match (timeout_lock) | `Timeout::match_task` |
| `io_kill_timeouts()` | per-ring-teardown / per-task-exit | `Timeout::kill_all` |
| `io_put_req()` | local ref drop helper | shared (`io_uring/core`) |
| `io_fail_links()` | propagate `REQ_F_FAIL` along link | `Timeout::fail_links` |
| `io_req_tw_fail_links()` | task-work: cancel/fail chained reqs | `Timeout::tw_fail_links` |
| `io_remove_next_linked()` | unlink-and-skip helper | `Timeout::remove_next_linked` |

## Compatibility contract

REQ-1: struct io_timeout layout (req-private command area):
- file: per-cmd file (unused for timeout, retained for shape).
- off: per-CQE-gate count (or 0 = pure timeout; or initial multishot repeat count).
- target_seq: per-(tail + off) snapshot at issue (seq-gated only).
- repeats: per-MULTISHOT remaining-arm count (= off at prep, decremented each rearm).
- list: per-anchor on `ctx->timeout_list` or `ctx->ltimeout_list`.
- head: per-linked-timeout back-ref to guarded head req (LINK_TIMEOUT only).
- prev: per-link-expiry: snapshot of head at expiry so task-work can cancel it.

REQ-2: struct io_timeout_data layout (`req->async_data`):
- req: per-back-ref to owning io_kiocb.
- timer: hrtimer.
- time: per-ktime_t deadline (REL = delta; ABS = absolute on chosen clock).
- mode: HRTIMER_MODE_ABS or HRTIMER_MODE_REL (pinned via `io_translate_timeout_mode`).
- flags: per-IORING_TIMEOUT_* mask.

REQ-3: io_flags_to_clock(flags):
- flags & IORING_TIMEOUT_CLOCK_MASK == IORING_TIMEOUT_BOOTTIME ⟹ CLOCK_BOOTTIME.
- flags & IORING_TIMEOUT_CLOCK_MASK == IORING_TIMEOUT_REALTIME ⟹ CLOCK_REALTIME.
- default 0 ⟹ CLOCK_MONOTONIC.
- More than one clock bit ⟹ rejected at prep (`hweight32(... CLOCK_MASK) > 1` is -EINVAL).

REQ-4: io_parse_user_time(time, arg, flags):
- IORING_TIMEOUT_IMMEDIATE_ARG ⟹ time = ns_to_ktime(arg); reject if *time < 0.
- Otherwise: get_timespec64(&ts, u64_to_user_ptr(arg)) — -EFAULT on copy failure; -EINVAL if tv_sec < 0 ∨ tv_nsec < 0.
- IORING_TIMEOUT_ABS ⟹ apply `timens_ktime_to_host(io_flags_to_clock(flags), *time)` (time-namespace offset).

REQ-5: __io_timeout_prep(req, sqe, is_timeout_link):
- sqe->addr3 ≠ 0 ∨ sqe->__pad2[0] ≠ 0 ⟹ -EINVAL.
- sqe->buf_index ≠ 0 ∨ sqe->len ≠ 1 ∨ sqe->splice_fd_in ≠ 0 ⟹ -EINVAL.
- off = sqe->off; off ≠ 0 ∧ is_timeout_link ⟹ -EINVAL (linked timeout cannot be seq-gated).
- flags = sqe->timeout_flags.
- flags ∉ (ABS | CLOCK_MASK | ETIME_SUCCESS | MULTISHOT | IMMEDIATE_ARG) ⟹ -EINVAL.
- hweight32(flags & CLOCK_MASK) > 1 ⟹ -EINVAL.
- MULTISHOT ∧ ABS both set ⟹ -EINVAL (multishot is rel-only).
- INIT_LIST_HEAD(&timeout->list); timeout->off = off.
- if off ≠ 0 ∧ !(ctx->int_flags & IO_RING_F_OFF_TIMEOUT_USED): set IO_RING_F_OFF_TIMEOUT_USED.
- repeats = 0; (MULTISHOT ∧ off > 0) ⟹ repeats = off.
- req_has_async_data(req) ⟹ WARN_ON_ONCE + -EFAULT.
- io_uring_alloc_async_data(NULL, req) ⟹ data; -ENOMEM on failure.
- data->req = req; data->flags = flags; parse time into data->time; data->mode = translate_mode(flags).
- is_timeout_link ⟹ link->head must exist; link->last->opcode ≠ IORING_OP_LINK_TIMEOUT (cannot chain two); timeout->head = link->last; link->last->flags |= REQ_F_ARM_LTIMEOUT; hrtimer_setup(&data->timer, io_link_timeout_fn, clock, mode).
- else: hrtimer_setup(&data->timer, io_timeout_fn, clock, mode).
- return 0.

REQ-6: io_timeout (regular issue):
- raw_spin_lock_irq(&ctx->timeout_lock).
- io_is_timeout_noseq(req) ⟹ entry = ctx->timeout_list.prev; goto add.
- Otherwise: tail = data_race(ctx->cached_cq_tail) - atomic_read(&ctx->cq_timeouts); target_seq = tail + off; ctx->cq_last_tm_flush = tail.
- Insertion-sort: walk timeout_list backwards; skip noseq entries; stop when `off >= nextt->target_seq - tail`.
- add: list_add(&timeout->list, entry); hrtimer_start(&data->timer, data->time, data->mode).
- raw_spin_unlock_irq.
- return IOU_ISSUE_SKIP_COMPLETE (timer drives completion).

REQ-7: io_timeout_fn (hrtimer callback, regular):
- container_of(timer, io_timeout_data, timer).
- raw_spin_lock_irqsave(&ctx->timeout_lock, flags).
- list_del_init(&timeout->list).
- atomic_inc-equivalent: cq_timeouts += 1 (atomic_set(read+1)).
- raw_spin_unlock_irqrestore.
- !(data->flags & IORING_TIMEOUT_ETIME_SUCCESS) ⟹ req_set_fail(req).
- io_req_set_res(req, -ETIME, 0); req->io_task_work.func = io_timeout_complete; io_req_task_work_add(req).
- return HRTIMER_NORESTART.

REQ-8: io_timeout_complete (task-work):
- !io_timeout_finish(timeout, data) ⟹ try io_req_post_cqe(req, -ETIME, IORING_CQE_F_MORE); on success, raw_spin_lock_irq(&ctx->timeout_lock); list_add(&timeout->list, ctx->timeout_list.prev); hrtimer_start(&data->timer, data->time, data->mode); raw_spin_unlock_irq; return.
- Otherwise io_req_task_complete(tw_req, tw) — final CQE, free.

REQ-9: io_timeout_finish(timeout, data):
- !(data->flags & MULTISHOT) ⟹ true (final).
- !timeout->off ⟹ false (infinite multishot).
- (timeout->repeats && --timeout->repeats) ⟹ false (still arming).
- ⟹ true (last shot consumed).

REQ-10: io_flush_timeouts (called on CQE post):
- raw_spin_lock_irq(&ctx->timeout_lock).
- seq = READ_ONCE(cached_cq_tail) - atomic_read(&cq_timeouts).
- list_for_each_entry_safe(timeout, tmp, &ctx->timeout_list, list):
  - io_is_timeout_noseq(req) ⟹ break (sort-order guarantee).
  - events_needed = timeout->target_seq - ctx->cq_last_tm_flush; events_got = seq - ctx->cq_last_tm_flush.
  - events_got < events_needed ⟹ break.
  - io_kill_timeout(req, &list).
- ctx->cq_last_tm_flush = seq.
- raw_spin_unlock_irq.
- io_flush_killed_timeouts(&list, 0).

REQ-11: LINK_TIMEOUT issue path (io_queue_linked_timeout):
- raw_spin_lock_irq(&ctx->timeout_lock).
- timeout->head ≠ NULL (head not yet completed) ⟹ hrtimer_start(&data->timer, data->time, data->mode); list_add_tail(&timeout->list, &ctx->ltimeout_list).
- raw_spin_unlock_irq.
- io_put_req(req) — drop submission ref (timer holds remaining ref).

REQ-12: io_link_timeout_fn (linked-timer expiry):
- raw_spin_lock_irqsave(&ctx->timeout_lock, flags).
- prev = timeout->head; timeout->head = NULL.
- prev ≠ NULL ⟹ io_remove_next_linked(prev); !req_ref_inc_not_zero(prev) ⟹ prev = NULL.
- list_del(&timeout->list); timeout->prev = prev.
- raw_spin_unlock_irqrestore.
- req->io_task_work.func = io_req_task_link_timeout; io_req_task_work_add.
- return HRTIMER_NORESTART.

REQ-13: io_req_task_link_timeout (link-expiry task-work):
- prev ≠ NULL:
  - !tw.cancel ⟹ io_try_cancel(req->tctx, &cd { ctx, data = prev->cqe.user_data }, 0); ret holds result.
  - tw.cancel ⟹ ret = -ECANCELED.
  - io_req_set_res(req, ret ?: -ETIME, 0); io_req_task_complete(tw_req, tw); io_put_req(prev).
- prev == NULL ⟹ io_req_set_res(req, -ETIME, 0); io_req_task_complete(tw_req, tw).

REQ-14: io_disarm_next (called on every completion that may have linked-timeout):
- req->flags & REQ_F_ARM_LTIMEOUT ⟹ link = req->link; clear REQ_F_ARM_LTIMEOUT; if link ∧ link->opcode == LINK_TIMEOUT ⟹ io_remove_next_linked(req); io_req_queue_tw_complete(link, -ECANCELED) (timer was set up but never started).
- req->flags & REQ_F_LINK_TIMEOUT ⟹ raw_spin_lock_irq(&ctx->timeout_lock); link = (req->link->opcode == LINK_TIMEOUT) ? __io_disarm_linked_timeout(req, req->link) : NULL; raw_spin_unlock_irq; if link ⟹ io_req_queue_tw_complete(link, -ECANCELED).
- (req->flags & REQ_F_FAIL) ∧ !(req->flags & REQ_F_HARDLINK) ⟹ io_fail_links(req).

REQ-15: __io_disarm_linked_timeout(req, link) (timeout_lock + completion_lock held):
- data = link->async_data; timeout = link_cmd.
- io_remove_next_linked(req); timeout->head = NULL.
- hrtimer_try_to_cancel(&io->timer) ≠ -1 ⟹ list_del(&timeout->list); return link.
- ⟹ return NULL (timer already firing — task-work will complete it).

REQ-16: io_timeout_remove_prep:
- flags & REQ_F_FIXED_FILE ∨ flags & REQ_F_BUFFER_SELECT ⟹ -EINVAL.
- sqe->addr3 ∨ sqe->__pad2[0] ⟹ -EINVAL.
- sqe->buf_index ∨ sqe->len ∨ sqe->splice_fd_in ⟹ -EINVAL.
- tr->ltimeout = false; tr->addr = sqe->addr; tr->flags = sqe->timeout_flags.
- (flags & IORING_TIMEOUT_UPDATE_MASK) ⟹ hweight32(flags & CLOCK_MASK) > 1 = -EINVAL; flags & IORING_LINK_TIMEOUT_UPDATE ⟹ tr->ltimeout = true; reject anything outside (UPDATE_MASK | ABS | IMMEDIATE_ARG); parse time into tr->time.
- else if flags ≠ 0 ⟹ -EINVAL (plain remove cannot carry flags).

REQ-17: io_timeout_remove (issue):
- !(tr->flags & IORING_TIMEOUT_UPDATE) ⟹ spin_lock(&ctx->completion_lock); io_timeout_cancel(ctx, &cd { ctx, data = tr->addr }); spin_unlock.
- IORING_TIMEOUT_UPDATE ⟹ raw_spin_lock_irq(&ctx->timeout_lock); tr->ltimeout ? io_linked_timeout_update(...) : io_timeout_update(...); raw_spin_unlock_irq.
- ret < 0 ⟹ req_set_fail.
- io_req_set_res(req, ret, 0); return IOU_COMPLETE.

REQ-18: io_timeout_cancel(ctx, cd) (completion_lock held):
- raw_spin_lock_irq(&ctx->timeout_lock); req = io_timeout_extract(ctx, cd); raw_spin_unlock_irq.
- IS_ERR(req) ⟹ return PTR_ERR(req) (-ENOENT or -EALREADY).
- io_req_task_queue_fail(req, -ECANCELED); return 0.

REQ-19: io_timeout_extract (must_hold timeout_lock):
- Walk ctx->timeout_list; first req where io_cancel_req_match(tmp, cd).
- Not found ⟹ ERR_PTR(-ENOENT).
- hrtimer_try_to_cancel(&io->timer) == -1 ⟹ ERR_PTR(-EALREADY) (timer already firing).
- list_del_init(&timeout->list); return req.

REQ-20: io_timeout_update (in-place, non-link):
- io_timeout_extract → req; IS_ERR ⟹ return.
- timeout->off = 0 (force noseq going forward); data->time = time; list_add_tail(&timeout->list, &ctx->timeout_list); hrtimer_setup(&data->timer, io_timeout_fn, clock, mode); hrtimer_start.

REQ-21: io_linked_timeout_update:
- Walk ctx->ltimeout_list by user_data.
- Not found ⟹ -ENOENT.
- hrtimer_try_to_cancel == -1 ⟹ -EALREADY.
- hrtimer_setup(&io->timer, io_link_timeout_fn, clock, mode); hrtimer_start.

REQ-22: io_kill_timeouts (ring teardown / task exit):
- spin_lock(&ctx->completion_lock); raw_spin_lock_irq(&ctx->timeout_lock).
- Walk ctx->timeout_list; for each, io_match_task(req, tctx, cancel_all) ⟹ io_kill_timeout(req, &list).
- raw_spin_unlock_irq; spin_unlock(&ctx->completion_lock).
- return io_flush_killed_timeouts(&list, -ECANCELED).

REQ-23: Lock order:
- completion_lock outer; timeout_lock inner (raw, IRQ-safe).
- hrtimer callbacks acquire timeout_lock only.
- Update / remove paths acquire timeout_lock; cancel-by-data path additionally acquires completion_lock first.

REQ-24: Multishot constraints:
- MULTISHOT may not be combined with ABS at prep (`(~flags & (MULTISHOT|ABS)) == 0` ⟹ -EINVAL).
- MULTISHOT with off > 0 sets repeats = off; each expiry decrements; CQE posts -ETIME with IORING_CQE_F_MORE while !finish.
- MULTISHOT with off == 0 ⟹ infinite-multi until cancelled / linked-completion.

REQ-25: ETIME_SUCCESS semantics:
- !(flags & ETIME_SUCCESS) ⟹ req_set_fail(req) on expiry; downstream linked reqs are cancelled per io_disarm_next.
- (flags & ETIME_SUCCESS) ⟹ no fail flag; -ETIME treated like a successful completion; downstream link continues.

REQ-26: Sequence-gated semantics:
- off > 0 ⟹ timer fires *or* `off` non-timeout CQEs post — whichever first.
- io_timeout itself never blocks; timer arms and the issue returns IOU_ISSUE_SKIP_COMPLETE.
- io_flush_timeouts called from CQE-post path drives the seq side.

REQ-27: Time-namespace handling:
- ABS + non-MONOTONIC clock ⟹ timens_ktime_to_host converts container-relative time into host-clock time at prep.

## Acceptance Criteria

- [ ] AC-1: `OP_TIMEOUT` with off=0 + REL time: fires after `data->time` elapses ⟹ single -ETIME CQE; subsequent linked reqs cancelled unless ETIME_SUCCESS.
- [ ] AC-2: `OP_TIMEOUT` with off=N: fires when N non-timeout CQEs have posted since submit OR when timer expires, whichever first; cq_timeouts incremented exactly once per fire.
- [ ] AC-3: `OP_TIMEOUT` with ABS + REALTIME flags: hrtimer armed with HRTIMER_MODE_ABS on CLOCK_REALTIME; timens conversion applied at prep.
- [ ] AC-4: ETIME_SUCCESS: -ETIME CQE posted *without* REQ_F_FAIL; following linked SQEs continue to issue.
- [ ] AC-5: MULTISHOT + off=K: K -ETIME|F_MORE CQEs then one final -ETIME (no F_MORE); rearm uses the same REL time each interval.
- [ ] AC-6: MULTISHOT + ABS ⟹ -EINVAL at prep.
- [ ] AC-7: `OP_LINK_TIMEOUT` head completes before timer fires: timer cancelled via __io_disarm_linked_timeout; link timeout CQE = -ECANCELED.
- [ ] AC-8: `OP_LINK_TIMEOUT` timer fires first: head cancelled via io_try_cancel; head CQE = -ECANCELED (or actual completion if race); link CQE = -ETIME.
- [ ] AC-9: `OP_LINK_TIMEOUT` chained twice (link of LINK_TIMEOUT) ⟹ -EINVAL at prep.
- [ ] AC-10: `OP_LINK_TIMEOUT` as first SQE of a chain (no head before it) ⟹ -EINVAL at prep.
- [ ] AC-11: `OP_TIMEOUT_REMOVE` (no UPDATE) with addr=user_data: cancels matching timeout ⟹ -ECANCELED CQE on victim, 0 CQE on remover. Missing target ⟹ remover CQE = -ENOENT. Timer already firing ⟹ -EALREADY.
- [ ] AC-12: `OP_TIMEOUT_REMOVE` with TIMEOUT_UPDATE: re-arms the named timer with new time + mode; timeout->off forced to 0; remover CQE = 0.
- [ ] AC-13: `OP_TIMEOUT_REMOVE` with LINK_TIMEOUT_UPDATE: re-arms named linked timer via io_linked_timeout_update.
- [ ] AC-14: Multiple clock bits set ⟹ -EINVAL at prep.
- [ ] AC-15: Ring teardown / task cancellation: io_kill_timeouts cancels all matching, posts -ECANCELED for each.

## Architecture

```
struct IoTimeout {
  file: *File,                              // unused, retained for cmd shape
  off: u32,                                 // 0 = pure / multishot-infinite
  target_seq: u32,                          // tail+off snapshot
  repeats: u32,                             // multishot remaining
  list: ListHead,                           // anchor in ctx.timeout_list / ltimeout_list
  head: Option<*IoKiocb>,                   // LINK_TIMEOUT guarded head
  prev: Option<*IoKiocb>,                   // task-work snapshot of head
}

struct IoTimeoutRem {
  file: *File,
  addr: u64,                                // target user_data
  time: KTime,                              // UPDATE only
  flags: u32,
  ltimeout: bool,                           // LINK_TIMEOUT_UPDATE selector
}

struct IoTimeoutData {                      // req.async_data
  req: *IoKiocb,
  timer: HrTimer,
  time: KTime,
  mode: HrTimerMode,                        // ABS or REL
  flags: u32,                               // IORING_TIMEOUT_*
}
```

`Timeout::prep_inner(req, sqe, is_link)`:
1. Validate sqe shape (addr3, __pad2[0], buf_index, len==1, splice_fd_in).
2. off = sqe.off; reject (off && is_link).
3. flags = sqe.timeout_flags; reject unknown bits; hweight CLOCK_MASK ≤ 1.
4. Reject MULTISHOT | ABS combination.
5. Init list head; record off + repeats (= off if MULTISHOT && off > 0).
6. Set ctx.int_flags |= IO_RING_F_OFF_TIMEOUT_USED if off ≠ 0.
7. Reject double async_data; allocate IoTimeoutData.
8. data.req = req; data.flags = flags; parse user time; data.mode = translate_mode(flags).
9. If is_link: validate link head; reject if last is LINK_TIMEOUT; set head; mark head REQ_F_ARM_LTIMEOUT; hrtimer_setup with io_link_timeout_fn.
10. Else: hrtimer_setup with io_timeout_fn.

`Timeout::issue(req)`:
1. raw_spin_lock_irq(&ctx.timeout_lock).
2. If is_noseq: entry = ctx.timeout_list.prev; goto add.
3. tail = data_race(ctx.cached_cq_tail) - ctx.cq_timeouts; target_seq = tail + off; ctx.cq_last_tm_flush = tail.
4. Insertion-sort: walk timeout_list backwards; skip noseq; stop when off ≥ next.target_seq - tail.
5. add: list_add(&timeout.list, entry); hrtimer_start(&data.timer, data.time, data.mode).
6. raw_spin_unlock_irq.
7. return IOU_ISSUE_SKIP_COMPLETE.

`Timeout::expire(timer) -> HrtimerRestart` (regular):
1. data = container_of(timer); req = data.req; ctx = req.ctx.
2. raw_spin_lock_irqsave(&ctx.timeout_lock, flags).
3. list_del_init(&timeout.list); ctx.cq_timeouts += 1.
4. raw_spin_unlock_irqrestore.
5. If !(data.flags & ETIME_SUCCESS): req_set_fail(req).
6. io_req_set_res(req, -ETIME, 0); io_req_task_work_add(req, Timeout::complete_tw).
7. return HRTIMER_NORESTART.

`Timeout::complete_tw(tw_req, tw)`:
1. If !Timeout::finish(timeout, data):
   - If io_req_post_cqe(req, -ETIME, IORING_CQE_F_MORE):
     - raw_spin_lock_irq(&ctx.timeout_lock); list_add(&timeout.list, ctx.timeout_list.prev); hrtimer_start(&data.timer, data.time, data.mode); raw_spin_unlock_irq.
     - return.
2. io_req_task_complete(tw_req, tw).

`Timeout::link_expire(timer) -> HrtimerRestart`:
1. data = container_of(timer); req = data.req; ctx = req.ctx.
2. raw_spin_lock_irqsave(&ctx.timeout_lock, flags).
3. prev = timeout.head; timeout.head = None.
4. If prev:
   - io_remove_next_linked(prev).
   - If !req_ref_inc_not_zero(prev): prev = None.
5. list_del(&timeout.list); timeout.prev = prev.
6. raw_spin_unlock_irqrestore.
7. io_req_task_work_add(req, Timeout::link_timeout_tw).
8. return HRTIMER_NORESTART.

`Timeout::link_timeout_tw(tw_req, tw)`:
1. prev = timeout.prev.
2. If prev:
   - If !tw.cancel: ret = io_try_cancel(req.tctx, &{ctx, data = prev.cqe.user_data}, 0).
   - Else: ret = -ECANCELED.
   - io_req_set_res(req, ret ?: -ETIME, 0); io_req_task_complete(tw_req, tw); io_put_req(prev).
3. Else: io_req_set_res(req, -ETIME, 0); io_req_task_complete(tw_req, tw).

`Timeout::queue_linked(req)`:
1. raw_spin_lock_irq(&ctx.timeout_lock).
2. If timeout.head:
   - hrtimer_start(&data.timer, data.time, data.mode).
   - list_add_tail(&timeout.list, &ctx.ltimeout_list).
3. raw_spin_unlock_irq.
4. io_put_req(req).

`Timeout::disarm_next(req)`:
1. If req.flags & REQ_F_ARM_LTIMEOUT:
   - link = req.link; clear REQ_F_ARM_LTIMEOUT.
   - If link && link.opcode == IORING_OP_LINK_TIMEOUT:
     - io_remove_next_linked(req); io_req_queue_tw_complete(link, -ECANCELED).
2. Else if req.flags & REQ_F_LINK_TIMEOUT:
   - raw_spin_lock_irq(&ctx.timeout_lock).
   - link = (req.link && req.link.opcode == LINK_TIMEOUT) ? Timeout::disarm_linked(req, req.link) : None.
   - raw_spin_unlock_irq.
   - If link: io_req_queue_tw_complete(link, -ECANCELED).
3. If (req.flags & REQ_F_FAIL) && !(req.flags & REQ_F_HARDLINK):
   - Timeout::fail_links(req).

`Timeout::disarm_linked(req, link)` (both completion_lock + timeout_lock held):
1. io_remove_next_linked(req); timeout.head = None.
2. If hrtimer_try_to_cancel(&io.timer) ≠ -1:
   - list_del(&timeout.list).
   - return Some(link).
3. return None.

`Timeout::flush_seq(ctx)` (called on CQE-post):
1. raw_spin_lock_irq(&ctx.timeout_lock).
2. seq = READ_ONCE(ctx.cached_cq_tail) - ctx.cq_timeouts.
3. Walk timeout_list:
   - If is_noseq(req): break.
   - events_needed = timeout.target_seq - ctx.cq_last_tm_flush.
   - events_got = seq - ctx.cq_last_tm_flush.
   - If events_got < events_needed: break.
   - Timeout::kill_one(req, &local).
4. ctx.cq_last_tm_flush = seq.
5. raw_spin_unlock_irq.
6. Timeout::flush_killed(&local, 0).

`Timeout::kill_one(req, list)` (must_hold timeout_lock):
1. If hrtimer_try_to_cancel(&io.timer) ≠ -1:
   - ctx.cq_timeouts += 1.
   - list_move_tail(&timeout.list, list).

`Timeout::flush_killed(list, err) -> bool`:
1. While !list_empty(list):
   - timeout = list_first_entry(list).
   - list_del_init(&timeout.list).
   - req = cmd_to_io_kiocb(timeout).
   - If err: req_set_fail(req).
   - io_req_queue_tw_complete(req, err).
2. return true (if any).

`Timeout::remove_issue(req)`:
1. If !(tr.flags & IORING_TIMEOUT_UPDATE):
   - spin_lock(&ctx.completion_lock).
   - ret = Timeout::cancel(ctx, &{ctx, data = tr.addr}).
   - spin_unlock(&ctx.completion_lock).
2. Else:
   - mode = translate_mode(tr.flags).
   - raw_spin_lock_irq(&ctx.timeout_lock).
   - ret = tr.ltimeout ? Timeout::update_linked(ctx, tr.addr, tr.time, mode) : Timeout::update(ctx, tr.addr, tr.time, mode).
   - raw_spin_unlock_irq.
3. If ret < 0: req_set_fail(req).
4. io_req_set_res(req, ret, 0); return IOU_COMPLETE.

`Timeout::extract(ctx, cd) -> Result<*IoKiocb>` (must_hold timeout_lock):
1. Walk ctx.timeout_list; first match via io_cancel_req_match.
2. Not found ⟹ Err(-ENOENT).
3. If hrtimer_try_to_cancel(&io.timer) == -1: Err(-EALREADY).
4. list_del_init(&timeout.list); Ok(req).

`Timeout::kill_all(ctx, tctx, cancel_all) -> bool`:
1. spin_lock(&ctx.completion_lock); raw_spin_lock_irq(&ctx.timeout_lock).
2. Walk ctx.timeout_list:
   - If io_match_task(req, tctx, cancel_all): Timeout::kill_one(req, &local).
3. raw_spin_unlock_irq; spin_unlock(&ctx.completion_lock).
4. return Timeout::flush_killed(&local, -ECANCELED).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `prep_rejects_addr3_pad` | INVARIANT | per-__io_timeout_prep: sqe.addr3 ≠ 0 ∨ __pad2[0] ≠ 0 ⟹ -EINVAL. |
| `prep_rejects_unknown_flags` | INVARIANT | per-__io_timeout_prep: any flag outside (ABS|CLOCK_MASK|ETIME_SUCCESS|MULTISHOT|IMMEDIATE_ARG) ⟹ -EINVAL. |
| `prep_rejects_multi_clock` | INVARIANT | per-__io_timeout_prep: hweight32(flags & CLOCK_MASK) > 1 ⟹ -EINVAL. |
| `prep_rejects_multishot_abs` | INVARIANT | per-__io_timeout_prep: MULTISHOT ∧ ABS ⟹ -EINVAL. |
| `prep_link_rejects_seq_off` | INVARIANT | per-__io_timeout_prep is_link: off ≠ 0 ⟹ -EINVAL. |
| `prep_link_rejects_first_sqe` | INVARIANT | per-__io_timeout_prep is_link: link->head must be set. |
| `prep_link_rejects_chained_ltimeout` | INVARIANT | per-__io_timeout_prep is_link: link->last->opcode == LINK_TIMEOUT ⟹ -EINVAL. |
| `prep_rejects_double_async_data` | INVARIANT | per-__io_timeout_prep: req_has_async_data ⟹ WARN + -EFAULT. |
| `parse_user_time_negative` | INVARIANT | per-io_parse_user_time: tv_sec < 0 ∨ tv_nsec < 0 ⟹ -EINVAL. |
| `expire_increments_cq_timeouts_once` | INVARIANT | per-io_timeout_fn: cq_timeouts atomically incremented exactly once per fire. |
| `expire_sets_fail_unless_etime_success` | INVARIANT | per-io_timeout_fn: !(flags & ETIME_SUCCESS) ⟹ req_set_fail. |
| `multishot_rearm_under_lock` | INVARIANT | per-io_timeout_complete: rearm path holds timeout_lock during list_add + hrtimer_start. |
| `cancel_extract_returns_ealready_if_firing` | INVARIANT | per-io_timeout_extract: hrtimer_try_to_cancel == -1 ⟹ -EALREADY. |
| `remove_prep_plain_rejects_flags` | INVARIANT | per-io_timeout_remove_prep: !UPDATE ∧ flags ≠ 0 ⟹ -EINVAL. |
| `link_disarm_unlinks_under_both_locks` | INVARIANT | per-__io_disarm_linked_timeout: must_hold completion_lock ∧ timeout_lock. |
| `flush_timeouts_break_on_noseq` | INVARIANT | per-io_flush_timeouts: noseq entry ends scan (sorted-order). |
| `kill_one_holds_timeout_lock` | INVARIANT | per-io_kill_timeout: __must_hold(&ctx->timeout_lock). |
| `queue_linked_drops_submission_ref` | INVARIANT | per-io_queue_linked_timeout: io_put_req called exactly once. |

### Layer 2: TLA+

`io_uring/timeout.tla`:
- Per-issue + per-expiry + per-cancel + per-update + per-link-disarm + per-flush-seq state machine.
- Properties:
  - `safety_cq_timeouts_monotonic` — cq_timeouts never decreases.
  - `safety_one_etime_per_timeout` — single-shot path posts exactly one -ETIME.
  - `safety_multishot_F_MORE_then_final` — MULTISHOT path emits k F_MORE CQEs followed by exactly one final.
  - `safety_no_double_kill` — a timeout req is either on `timeout_list` xor `ltimeout_list` xor local-killed-list xor completed; never two simultaneously.
  - `safety_link_pair_exclusive` — for any link, only one of {head, link-timeout} drives the other's cancellation.
  - `safety_extract_or_already_firing` — extract returns -ENOENT (not found), -EALREADY (timer firing), or req with list unlinked.
  - `safety_lock_order_completion_then_timeout` — completion_lock acquired strictly before timeout_lock when both are taken.
  - `liveness_pure_timeout_eventually_fires` — bare countdown with no cancel: eventually io_timeout_complete runs.
  - `liveness_seq_gated_eventually_completes` — off > 0: timer fires OR `off` non-timeout CQEs post; both paths complete.
  - `liveness_link_timeout_pair_terminates` — guarded head + link timer: exactly one cancels the other; both complete.
  - `liveness_ring_teardown_clears_lists` — io_kill_timeouts empties both timeout_list and ltimeout_list.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Timeout::prep_inner` post: ret == 0 ⟹ async_data alloc'd ∧ timer set up with chosen clock/mode | `Timeout::prep_inner` |
| `Timeout::issue` post: req on timeout_list ∧ timer started ∧ IOU_ISSUE_SKIP_COMPLETE | `Timeout::issue` |
| `Timeout::expire` post: cq_timeouts += 1 ∧ task-work enqueued ∧ list unlinked | `Timeout::expire` |
| `Timeout::complete_tw` rearm: req re-linked at tail-prev with timer restarted under timeout_lock | `Timeout::complete_tw` |
| `Timeout::link_expire` post: head ref bumped or prev = None ∧ task-work enqueued | `Timeout::link_expire` |
| `Timeout::link_timeout_tw` post: prev cancelled or -ETIME emitted; io_put_req(prev) iff prev was non-None | `Timeout::link_timeout_tw` |
| `Timeout::queue_linked` post: timer armed ∧ on ltimeout_list ∧ submission ref dropped | `Timeout::queue_linked` |
| `Timeout::disarm_next` post: link-timeout cancelled with -ECANCELED or already firing | `Timeout::disarm_next` |
| `Timeout::flush_seq` post: cq_last_tm_flush advanced; sorted-list invariant preserved | `Timeout::flush_seq` |
| `Timeout::extract` post: req unlinked ∧ timer cancelled, or -ENOENT, or -EALREADY | `Timeout::extract` |
| `Timeout::update` post: timeout.off = 0 ∧ timer re-armed with new mode/clock | `Timeout::update` |
| `Timeout::kill_all` post: every matching req queued for tw_complete(-ECANCELED) | `Timeout::kill_all` |

### Layer 4: Verus/Creusot functional

`Per-OP_TIMEOUT prep → issue → (timer fire ∨ seq-gate ∨ cancel) → task-work → CQE`, `Per-OP_LINK_TIMEOUT prep → arm-on-head-issue → (head-completes-first ⟹ disarm; timer-fires-first ⟹ cancel-head + -ETIME)`, `Per-OP_TIMEOUT_REMOVE prep → (cancel | update | linked-update)` semantic equivalence: per Documentation/io_uring/* and `include/uapi/linux/io_uring.h` flag semantics. cq_timeouts counter increments and IORING_CQE_F_MORE multishot framing per io_uring(7) and CQE flag definitions.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

io_uring/timeout reinforcement:

- **Per-prep flag whitelist strict** — defense against per-undefined-flag silent acceptance.
- **Per-prep MULTISHOT ∧ ABS rejected at prep** — defense against per-monotonically-impossible rearm.
- **Per-prep CLOCK_MASK hweight ≤ 1** — defense against per-ambiguous clock domain.
- **Per-prep `len == 1` enforced** — defense against per-sqe-shape drift.
- **Per-prep linked-of-linked rejected** — defense against per-runaway chained guards.
- **Per-extract -EALREADY when timer firing** — defense against per-cancel-vs-expiry double completion.
- **Per-completion-then-timeout lock order** — defense against per-ABBA deadlock with cancel paths.
- **Per-raw_spin_irq for hrtimer side** — defense against per-IRQ-context lock-class mismatch.
- **Per-req_ref_inc_not_zero on link head snapshot** — defense against per-head-freed-during-link-expire UAF.
- **Per-cq_timeouts atomic increment** — defense against per-seq-gate divergence with concurrent CQE post.
- **Per-IO_RING_F_OFF_TIMEOUT_USED sticky flag** — defense against per-flush_timeouts spurious cost when no seq timer ever used.
- **Per-timeout_list insertion-sort + break-on-noseq** — defense against per-O(n²) flush.
- **Per-data->req back-ref set before hrtimer_start** — defense against per-callback-on-uninitialized.
- **Per-cancel-by-user_data scoped to req.cqe.user_data + ctx** — defense against per-cross-ring cancel.

## Grsecurity/PaX-style Reinforcement

io_uring/timeout backs every IORING_OP_TIMEOUT/IORING_OP_LINK_TIMEOUT with an hrtimer that runs in IRQ context; the timer-vs-cancel race is the highest-attacked surface. Rookery grsec/PaX floor:

- **PAX_USERCOPY** — SQE fields read from kernel-owned SQ mmap region; `__kernel_timespec` deserialized via `get_timespec64`/`get_old_timespec64` with bounds checked at prep.
- **PAX_KERNEXEC** — `io_op_def[IORING_OP_TIMEOUT*]` entries and `io_timeout_fn` hrtimer callback are `static const`/`__ro_after_init`.
- **PAX_RANDKSTACK** — both submit and hrtimer-callback paths run under randomized kstack offsets where applicable.
- **PAX_REFCOUNT** — `io_kiocb.refs` and per-`io_timeout_data.req` back-pointer use saturating refcount; `req_ref_inc_not_zero` audited at every linked-timeout snapshot.
- **PAX_MEMORY_SANITIZE** — `io_timeout_data` slab zeroed on free; `io_kiocb.timeout` union cleared so stale `target_seq`/`flags` cannot bleed into a later submission.
- **PAX_UDEREF** — `get_timespec64` is the sole boundary; no raw `__user` deref in issue/cancel paths.
- **PAX_RAP/kCFI** — `io_timeout_fn`, `io_link_timeout_fn`, and `io_op_def.{prep,issue,cleanup}` are CFI-typed; hrtimer callback function pointer in `hrtimer.function` is checked at every `hrtimer_start`.
- **GRKERNSEC_HIDESYM** — timeout-fired/cancel diagnostics emit only `req->cqe.user_data` (caller-supplied) and `req->cqe.res`; no kernel pointers.
- **GRKERNSEC_DMESG** — WARN_ON paths (e.g., callback on uninitialized req) rate-limited and scrubbed of req identity.
- **hrtimer-backed PAX_REFCOUNT** — `io_timeout_data.req` back-pointer increments refcount before `hrtimer_start`; hrtimer callback runs `io_req_task_complete` only if the saturating ref-inc-not-zero succeeds; defeats UAF where cancel completes while callback is mid-flight.
- **MULTISHOT validation** — MULTISHOT requires REL clock domain (rejected with ABS at prep); MULTISHOT rearm checks `req->ctx` still references the same ring; cross-ring MULTISHOT rearm denied.
- **CLOCK_MASK hweight = 1** — clock domain bits validated at prep to exactly one of {MONOTONIC, BOOTTIME, REALTIME}; CLOCK_MASK hweight > 1 rejected; defeats clock-confusion oracles.
- **Per-`completion-then-timeout` lock order** — `ctx->timeout_lock` then `ctx->completion_lock`; reverse order is build-time-asserted via lockdep; ABBA deadlock with cancel path impossible.
- **Linked-of-linked rejected** — linked-timeout cannot itself be the head of a link chain; cycle prevention at prep time.

Rationale: io_uring/timeout is the standard exploit primitive for UAF-via-hrtimer-callback (CVE-2022-3910-class); the grsec-equivalent floor pins the req back-ref via saturating PAX_REFCOUNT and forbids MULTISHOT cross-domain semantics that historically smuggled cancellation races into the callback path.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- io_uring core SQE/CQE infrastructure (covered in `io_uring-core.md` Tier-3)
- io_cancel_req_match semantics + cancel-by-fd/op (covered in `cancel.md` Tier-3)
- Generic hrtimer / time_namespace internals (kernel-time subsystem)
- IORING_OP_TIMEOUT polling-mode interactions (covered in `poll.md` Tier-3)
- Wait-CQE syscall-side timeout-arg (covered in `io_uring-core.md` Tier-3)
- Implementation code
