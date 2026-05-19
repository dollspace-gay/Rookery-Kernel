# Tier-3: io_uring/notif.c — Zerocopy-send notification

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: io_uring/00-overview.md
upstream-paths:
  - io_uring/notif.c (~141 lines)
  - io_uring/notif.h
  - include/linux/skbuff.h (ubuf_info, ubuf_info_ops, skb_zcopy_*)
  - include/uapi/linux/io_uring.h (IORING_NOTIF_USAGE_ZC_COPIED, IORING_OP_SEND_ZC, IORING_OP_SENDMSG_ZC)
-->

## Summary

The **io_uring zerocopy-send notification** is a completion-pair CQE delivered to userspace once the network stack is done with the user pages handed to `IORING_OP_SEND_ZC` / `IORING_OP_SENDMSG_ZC`. Per-zc-send, two CQEs are posted: the "request" CQE (carrying `MORE` while the kernel still references the payload) and the "notification" CQE (posted when `refcnt` of the `ubuf_info` drops to zero — i.e. all `sk_buff`s carrying the user pages have been freed). Per-notif allocation: `io_alloc_notif()` reuses the io_uring request slab and embeds a `struct io_notif_data` inside the `io_kiocb` cmd payload — `nd->uarg` is the `struct ubuf_info` that the network stack reference-counts. Per-skb attach: `io_link_skb()` chains additional skbs to an existing notif when a single send fragments across many skbs, or fuses two notifs into one ubuf-chain. Per-completion: `io_tx_ubuf_complete()` is invoked from softirq / network-stack release path; when `refcount_dec_and_test(&uarg->refcnt)` succeeds it task-work-queues `io_notif_tw_complete()` which posts the notification CQE on the originating ring, decoded with optional `IORING_NOTIF_USAGE_ZC_COPIED` flag (set when the stack had to copy because the device DMA hardware could not directly send the user pages). Critical for: TCP/UDP transmit zerocopy, MSG_ZEROCOPY equivalent, throughput-bound services that want to avoid bcopy on the send path.

This Tier-3 covers `io_uring/notif.c` (~141 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_notif_data` | per-notif state (uarg, chain, account_pages, zc_report flags) | `IoNotifData` |
| `IO_NOTIF_UBUF_FLAGS` | per-skbuff zcopy flags (`SKBFL_ZEROCOPY_FRAG \| SKBFL_DONT_ORPHAN`) | `IO_NOTIF_UBUF_FLAGS` |
| `IO_NOTIF_SPLICE_BATCH` | per-batch flush cap (32) | `IO_NOTIF_SPLICE_BATCH` |
| `io_alloc_notif()` | per-zc-send allocator (request slab + ubuf init) | `IoNotif::alloc` |
| `io_tx_ubuf_complete()` | per-net-stack release callback | `IoNotif::tx_ubuf_complete` |
| `io_link_skb()` | per-skb-attach: chain new skb to notif's uarg | `IoNotif::link_skb` |
| `io_notif_tw_complete()` | per-task-work CQE post + chain walk | `IoNotif::tw_complete` |
| `io_notif_to_data()` | per-kiocb downcast to io_notif_data | `IoNotif::to_data` |
| `io_notif_flush()` | per-userspace flush (drop kernel ref) | `IoNotif::flush` |
| `io_notif_account_mem()` | per-mlock accounting on user pages | `IoNotif::account_mem` |
| `io_ubuf_ops` (`.complete`, `.link_skb`) | per-ubuf_info vtable for io_uring zc | `IoNotif::UBUF_OPS` |
| `IORING_NOTIF_USAGE_ZC_COPIED` | per-CQE-res flag: stack copied | shared UAPI bit |

## Compatibility contract

REQ-1: struct io_notif_data layout (cmd-payload of io_kiocb):
- file: per-syscall-required first field (`io_kiocb_to_cmd` invariant).
- uarg: per-`struct ubuf_info` (the network stack's reference handle).
- next: per-chain successor (NULL for tail or single-notif).
- head: per-chain anchor (== self for unchained).
- account_pages: per-mlock-accounted page count (0 if unaccounted).
- zc_report: per-flag: report `IORING_NOTIF_USAGE_ZC_COPIED` on completion.
- zc_used: per-flag: at least one skb made it to the device zerocopy-eligible.
- zc_copied: per-flag: stack copied at least once.

REQ-2: io_alloc_notif(ctx):
- `__must_hold(&ctx->uring_lock)`.
- `io_alloc_req(ctx, &notif)` — borrow request slab; per-NULL: caller returns -ENOMEM.
- notif.ctx = ctx.
- notif.opcode = IORING_OP_NOP (the notif rides the request infra but is not a real op).
- notif.flags = 0.
- notif.file = NULL.
- notif.tctx = current.io_uring.
- `io_get_task_refs(1)` — take user-task ref so completion can rendezvous on owning task.
- notif.file_node = NULL; notif.buf_node = NULL.
- nd = io_notif_to_data(notif).
- nd.zc_report = false; nd.account_pages = 0; nd.next = NULL; nd.head = nd.
- nd.uarg.flags = IO_NOTIF_UBUF_FLAGS  (`SKBFL_ZEROCOPY_FRAG | SKBFL_DONT_ORPHAN`).
- nd.uarg.ops = &io_ubuf_ops.
- refcount_set(&nd.uarg.refcnt, 1).
- return notif.

REQ-3: io_ubuf_ops vtable:
- .complete = io_tx_ubuf_complete  (skb-finalize / freed-from-stack callback).
- .link_skb = io_link_skb  (per-additional-skb attach).
- No `.recv_link_skb` (TX-only).

REQ-4: io_tx_ubuf_complete(skb, uarg, success):
- nd = container_of(uarg, struct io_notif_data, uarg).
- notif = cmd_to_io_kiocb(nd).
- /* Capture zc-usage stats */
- if nd.zc_report:
  - if success ∧ !nd.zc_used ∧ skb: WRITE_ONCE(nd.zc_used, true).
  - else if !success ∧ !nd.zc_copied: WRITE_ONCE(nd.zc_copied, true).
- /* Drop ref; bail if not the last */
- if !refcount_dec_and_test(&uarg.refcnt): return.
- /* Bubble up to chain head */
- if nd.head != nd: io_tx_ubuf_complete(skb, &nd.head.uarg, success); return.
- /* Last ref of head: queue task-work to post CQE */
- tw_flags = nd.next ? 0 : IOU_F_TWQ_LAZY_WAKE.
- notif.io_task_work.func = io_notif_tw_complete.
- __io_req_task_work_add(notif, tw_flags).

REQ-5: io_notif_tw_complete(tw_req, tw):
- `lockdep_assert_held(&ctx->uring_lock)`.
- Iterates the singly-linked chain (head → next → ...):
  - WARN_ON_ONCE(ctx != notif.ctx)  /* per-chain-ctx invariant */.
  - lockdep_assert(refcount_read(&nd.uarg.refcnt) == 0)  /* per-only-on-final */.
  - if unlikely(nd.zc_report) ∧ (nd.zc_copied ∨ !nd.zc_used):
    - notif.cqe.res |= IORING_NOTIF_USAGE_ZC_COPIED.
  - if nd.account_pages ∧ notif.ctx.user:
    - __io_unaccount_mem(notif.ctx.user, nd.account_pages).
    - nd.account_pages = 0.
  - nd = nd.next; io_req_task_complete({notif}, tw); /* posts CQE */.
- Loop terminates at NULL successor.

REQ-6: io_link_skb(skb, uarg):
- nd = container_of(uarg, struct io_notif_data, uarg).
- notif = cmd_to_io_kiocb(nd).
- prev_uarg = skb_zcopy(skb).
- if !prev_uarg:
  - /* First zc handle for this skb: attach */
  - net_zcopy_get(&nd.uarg).
  - skb_zcopy_init(skb, &nd.uarg).
  - return 0.
- /* Self-link is a no-op */
- if prev_uarg == &nd.uarg: return 0.
- /* Reject joining two already-linked chains */
- if nd.head != nd ∨ nd.next: return -EEXIST.
- /* Reject mixing zc providers (e.g. MSG_ZEROCOPY + io_uring zc) */
- if prev_uarg.ops != &io_ubuf_ops: return -EEXIST.
- prev_nd = container_of(prev_uarg, struct io_notif_data, uarg).
- prev_notif = cmd_to_io_kiocb(prev_nd).
- /* Reject cross-ring / cross-task chaining (task_work must hit one tctx) */
- if notif.ctx != prev_notif.ctx ∨ notif.tctx != prev_notif.tctx: return -EEXIST.
- /* Splice nd into chain headed by prev_nd.head */
- nd.head = prev_nd.head.
- nd.next = prev_nd.next.
- prev_nd.next = nd.
- net_zcopy_get(&nd.head.uarg)  /* head holds the live ref to keep chain alive */.
- return 0.

REQ-7: io_notif_flush(notif):
- `__must_hold(&notif->ctx->uring_lock)`.
- nd = io_notif_to_data(notif).
- io_tx_ubuf_complete(NULL, &nd.uarg, true).
- Drops the synthetic ref set at allocation so the completion path can fire once the stack has also dropped its refs.

REQ-8: io_notif_account_mem(notif, len):
- nr_pages = (len >> PAGE_SHIFT) + 2  /* over-approximation: per-edge-page-slack */.
- if ctx.user:
  - ret = __io_account_mem(ctx.user, nr_pages); if ret: return ret.
  - nd.account_pages += nr_pages.
- return 0.
- Reverse-charged in io_notif_tw_complete via __io_unaccount_mem.

REQ-9: Notification CQE encoding:
- res = bytes sent (set by IORING_OP_SEND_ZC/SENDMSG_ZC issue path; the notif borrows this when delivered).
- res |= IORING_NOTIF_USAGE_ZC_COPIED if `zc_report ∧ (zc_copied ∨ !zc_used)`.
- user_data = SQE user_data of the originating zc-send.
- flags: no IORING_CQE_F_MORE (notification is terminal).

REQ-10: Chain semantics:
- Per-send: one head notif; additional skbs attach via io_link_skb.
- All chained notifs share a single deferred task-work hop (the head's).
- Per io_notif_tw_complete: walks chain, posts one CQE per notif (so userspace still sees user_data of each individual send if the stack split them — but each was its own io_alloc_notif call).

REQ-11: Reference-count model:
- `refcount_set(uarg.refcnt, 1)` at allocation (kernel "issuer" ref).
- Each `skb_zcopy_init` / `net_zcopy_get` takes +1.
- Each `skb` releasing the ubuf decrements via `skb_zcopy_clear()`.
- Final `refcount_dec_and_test()` fires `io_tx_ubuf_complete` → task-work.

REQ-12: IO_NOTIF_UBUF_FLAGS:
- `SKBFL_ZEROCOPY_FRAG`: skb pages may not be coalesced/copied for delivery.
- `SKBFL_DONT_ORPHAN`: skb owner must not be cleared on transmit (so completion can find the ubuf_info).

REQ-13: Lock discipline:
- io_alloc_notif: caller holds `ctx->uring_lock`.
- io_notif_flush: caller holds `ctx->uring_lock`.
- io_tx_ubuf_complete: may run in softirq; lock-free (refcount + task-work).
- io_notif_tw_complete: `ctx->uring_lock` held (task-work invariant).

REQ-14: zc_report semantics:
- Set by io_send_zc_prep when SQE flag requests reporting.
- io_tx_ubuf_complete latches zc_used / zc_copied as skbs flow.
- io_notif_tw_complete folds them into one CQE `IORING_NOTIF_USAGE_ZC_COPIED` bit.

REQ-15: Error paths:
- io_alloc_notif failure: caller returns -ENOMEM at SQE submit; no notif allocated; no ubuf_info; no CQE pair.
- io_link_skb -EEXIST: caller (io_send_zc) requests a fresh skb (avoids mixing zc providers / chain saturation).

## Acceptance Criteria

- [ ] AC-1: IORING_OP_SEND_ZC SQE submitted → io_alloc_notif returns non-NULL when slab has space; ubuf.refcnt == 1.
- [ ] AC-2: First skb of zc-send has skb_zcopy(skb) == &nd.uarg after io_link_skb attaches.
- [ ] AC-3: Second skb (same send) attaches via io_link_skb without -EEXIST; chain head == first nd; head's refcnt incremented.
- [ ] AC-4: Stack-copied skb (NIC cannot DMA user pages): nd.zc_copied = true; CQE.res has IORING_NOTIF_USAGE_ZC_COPIED iff zc_report set.
- [ ] AC-5: All skbs freed: refcount drops to 0; task-work queued; CQE posted with user_data of originating send.
- [ ] AC-6: io_notif_flush called from io_send_zc completion drops kernel-issuer ref; CQE not posted until stack releases too.
- [ ] AC-7: account_pages > 0 at allocation → __io_unaccount_mem called for same nr_pages at io_notif_tw_complete.
- [ ] AC-8: Cross-ring chain attempt (notif.ctx != prev_notif.ctx) → io_link_skb returns -EEXIST.
- [ ] AC-9: MSG_ZEROCOPY ubuf prev attached → io_link_skb returns -EEXIST (mixed providers rejected).
- [ ] AC-10: nd.head != nd: completion cascades to head via recursive io_tx_ubuf_complete; only head queues task-work.
- [ ] AC-11: io_notif_tw_complete walks chain in order; one CQE per notif; ctx-mismatch trips WARN_ON_ONCE and aborts.
- [ ] AC-12: Last in chain: tw_flags == IOU_F_TWQ_LAZY_WAKE; non-tail: 0.
- [ ] AC-13: SKBFL_ZEROCOPY_FRAG + SKBFL_DONT_ORPHAN set on uarg.flags at allocation.

## Architecture

```
struct IoNotifData {
  file: Option<*File>,           // first-field invariant (io_kiocb_to_cmd)
  uarg: UbufInfo,                // refcounted handle held by network stack
  next: Option<*IoNotifData>,    // chain successor
  head: *IoNotifData,            // chain anchor (self for unchained)
  account_pages: u32,            // mlock-charged pages
  zc_report: bool,               // emit IORING_NOTIF_USAGE_ZC_COPIED
  zc_used: bool,                 // any skb truly went zerocopy
  zc_copied: bool,               // any skb fell back to copy
}

const IO_NOTIF_UBUF_FLAGS: u32 = SKBFL_ZEROCOPY_FRAG | SKBFL_DONT_ORPHAN;
const IO_NOTIF_SPLICE_BATCH: u32 = 32;

static UBUF_OPS: UbufInfoOps = UbufInfoOps {
  complete: IoNotif::tx_ubuf_complete,
  link_skb: IoNotif::link_skb,
};
```

`IoNotif::alloc(ctx: *IoRingCtx) -> Option<*IoKiocb>`:
1. /* Caller holds ctx.uring_lock */
2. notif = io_alloc_req(ctx)?  /* slab borrow; None ⟹ -ENOMEM */
3. notif.ctx = ctx; notif.opcode = IORING_OP_NOP; notif.flags = 0.
4. notif.file = None; notif.tctx = current.io_uring.
5. io_get_task_refs(1)  /* user-task ref for completion rendezvous */
6. notif.file_node = None; notif.buf_node = None.
7. nd = io_notif_to_data(notif).
8. nd.zc_report = false; nd.account_pages = 0; nd.next = None; nd.head = nd.
9. nd.uarg.flags = IO_NOTIF_UBUF_FLAGS.
10. nd.uarg.ops = &UBUF_OPS.
11. refcount_set(&nd.uarg.refcnt, 1).
12. Some(notif).

`IoNotif::tx_ubuf_complete(skb, uarg, success)`:
1. nd = container_of(uarg, IoNotifData, uarg).
2. notif = cmd_to_io_kiocb(nd).
3. if nd.zc_report:
   - if success ∧ !nd.zc_used ∧ skb.is_some(): WRITE_ONCE(nd.zc_used, true).
   - else if !success ∧ !nd.zc_copied: WRITE_ONCE(nd.zc_copied, true).
4. if !refcount_dec_and_test(&uarg.refcnt): return.
5. if nd.head != nd:
   - tx_ubuf_complete(skb, &nd.head.uarg, success).
   - return.
6. tw_flags = if nd.next.is_some() { 0 } else { IOU_F_TWQ_LAZY_WAKE }.
7. notif.io_task_work.func = IoNotif::tw_complete.
8. __io_req_task_work_add(notif, tw_flags).

`IoNotif::tw_complete(tw_req, tw)`:
1. notif = tw_req.req; nd = io_notif_to_data(notif); ctx = notif.ctx.
2. lockdep_assert_held(&ctx.uring_lock).
3. loop:
   - notif = cmd_to_io_kiocb(nd).
   - if WARN_ON_ONCE(ctx != notif.ctx): return.
   - lockdep_assert(refcount_read(&nd.uarg.refcnt) == 0).
   - if unlikely(nd.zc_report) ∧ (nd.zc_copied ∨ !nd.zc_used):
     - notif.cqe.res |= IORING_NOTIF_USAGE_ZC_COPIED.
   - if nd.account_pages > 0 ∧ notif.ctx.user.is_some():
     - __io_unaccount_mem(notif.ctx.user, nd.account_pages).
     - nd.account_pages = 0.
   - next = nd.next.
   - io_req_task_complete({notif}, tw).
   - nd = next; if nd.is_none(): break.

`IoNotif::link_skb(skb, uarg) -> i32`:
1. nd = container_of(uarg, IoNotifData, uarg).
2. notif = cmd_to_io_kiocb(nd).
3. prev_uarg = skb_zcopy(skb).
4. if prev_uarg.is_none():
   - net_zcopy_get(&nd.uarg).
   - skb_zcopy_init(skb, &nd.uarg).
   - return 0.
5. if prev_uarg == &nd.uarg: return 0.
6. if nd.head != nd ∨ nd.next.is_some(): return -EEXIST.
7. if prev_uarg.ops != &UBUF_OPS: return -EEXIST.
8. prev_nd = container_of(prev_uarg, IoNotifData, uarg).
9. prev_notif = cmd_to_io_kiocb(prev_nd).
10. if notif.ctx != prev_notif.ctx ∨ notif.tctx != prev_notif.tctx: return -EEXIST.
11. nd.head = prev_nd.head.
12. nd.next = prev_nd.next.
13. prev_nd.next = nd.
14. net_zcopy_get(&nd.head.uarg).
15. return 0.

`IoNotif::flush(notif)`:
1. /* Caller holds notif.ctx.uring_lock */
2. nd = io_notif_to_data(notif).
3. tx_ubuf_complete(None, &nd.uarg, true).

`IoNotif::account_mem(notif, len) -> i32`:
1. nr_pages = (len >> PAGE_SHIFT) + 2.
2. if ctx.user.is_some():
   - ret = __io_account_mem(ctx.user, nr_pages); if ret != 0: return ret.
   - nd.account_pages += nr_pages.
3. return 0.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `uarg_refcnt_eq_one_post_alloc` | INVARIANT | per-alloc: uarg.refcnt == 1 on exit. |
| `chain_head_self_post_alloc` | INVARIANT | per-alloc: nd.head == nd, nd.next == None. |
| `tw_complete_walks_to_null` | INVARIANT | per-tw: loop terminates at nd.next == None. |
| `tw_complete_holds_uring_lock` | INVARIANT | per-tw: ctx.uring_lock held. |
| `complete_chains_to_head` | INVARIANT | per-tx_ubuf_complete: nd.head != nd ⟹ recurse to head. |
| `link_skb_no_self_link` | INVARIANT | per-link_skb: prev_uarg == &nd.uarg ⟹ no chain mutation. |
| `link_skb_no_mixed_ops` | INVARIANT | per-link_skb: prev_uarg.ops != &UBUF_OPS ⟹ -EEXIST. |
| `link_skb_no_cross_ctx` | INVARIANT | per-link_skb: ctx / tctx mismatch ⟹ -EEXIST. |
| `account_pages_unaccount_match` | INVARIANT | per-tw: __io_unaccount_mem called for exactly nd.account_pages. |
| `zc_copied_only_if_zc_report` | INVARIANT | per-tw: IORING_NOTIF_USAGE_ZC_COPIED set ⟹ nd.zc_report was true. |
| `ubuf_flags_correct` | INVARIANT | per-alloc: uarg.flags == SKBFL_ZEROCOPY_FRAG \| SKBFL_DONT_ORPHAN. |

### Layer 2: TLA+

`io_uring/notif.tla`:
- Per-alloc + per-attach (link_skb) + per-skb-release (complete) + per-tw + per-flush.
- Properties:
  - `safety_refcnt_nonneg` — per-step: uarg.refcnt >= 0.
  - `safety_cqe_once_per_notif` — per-notif: at most one task-work-driven CQE.
  - `safety_no_cqe_before_zero_refs` — per-tw_complete: nd.uarg.refcnt == 0 at entry.
  - `safety_chain_acyclic` — per-link_skb: chain is a singly-linked list with unique head.
  - `safety_no_cross_ring_chain` — per-link_skb: chained notifs share ctx and tctx.
  - `liveness_eventually_cqe` — per-alloc: after flush + all skbs released, CQE eventually posted.
  - `liveness_account_pages_returned` — per-alloc with account_pages > 0: __io_unaccount_mem eventually called.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IoNotif::alloc` post: returned notif has uarg.refcnt == 1, head == self | `IoNotif::alloc` |
| `IoNotif::tx_ubuf_complete` post: refcnt' == refcnt - 1; refcnt' == 0 ⟹ task-work queued (on head) | `IoNotif::tx_ubuf_complete` |
| `IoNotif::link_skb` post: success ⟹ skb_zcopy(skb) refers to a notif in same ctx/tctx with .ops == UBUF_OPS | `IoNotif::link_skb` |
| `IoNotif::tw_complete` post: for each nd in chain, one io_req_task_complete; account_pages cleared | `IoNotif::tw_complete` |
| `IoNotif::flush` post: emulates one ref-drop on head uarg | `IoNotif::flush` |
| `IoNotif::account_mem` post: account_pages monotone non-decreasing across calls before unaccount | `IoNotif::account_mem` |

### Layer 4: Verus/Creusot functional

`Per-zc-send issue → io_alloc_notif → io_link_skb (per skb) → kernel stack consumes pages → io_tx_ubuf_complete (per skb release) → final refcount_dec_and_test → io_notif_tw_complete → CQE posted with optional IORING_NOTIF_USAGE_ZC_COPIED` semantic equivalence: per-Documentation/networking/msg_zerocopy.rst + per-io_uring(7) IORING_OP_SEND_ZC manual page.

## Hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

Notification reinforcement:

- **Per-uarg refcnt monotonic-to-zero** — defense against per-double-complete UAF.
- **Per-link_skb mixed-provider rejection** — defense against per-zc-provider confusion (MSG_ZEROCOPY ↔ io_uring zc).
- **Per-link_skb cross-ring rejection** — defense against per-task-work delivering on wrong ring.
- **Per-link_skb cross-tctx rejection** — defense against per-task-work wrong-owner CQE post.
- **Per-link_skb self-link no-op** — defense against per-self-recursive uarg chain.
- **Per-link_skb chain-already-linked rejection** — defense against per-rebraided-chain ordering corruption.
- **Per-chain head ref keeps chain alive** — defense against per-mid-chain-free UAF.
- **Per-WARN_ON_ONCE ctx mismatch in tw_complete** — defense against per-cross-ctx task-work injection.
- **Per-lockdep_assert refcnt == 0 entering tw_complete** — defense against per-stale-tw firing.
- **Per-__io_account_mem strict pair with __io_unaccount_mem** — defense against per-mlock-leak.
- **Per-IOU_F_TWQ_LAZY_WAKE only on chain tail** — defense against per-userspace-priority-inversion (mid-chain must not throttle).
- **Per-IO_NOTIF_UBUF_FLAGS includes SKBFL_DONT_ORPHAN** — defense against per-orphan-strip losing notification.
- **Per-IO_NOTIF_UBUF_FLAGS includes SKBFL_ZEROCOPY_FRAG** — defense against per-skb-copy-elision miss.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — bounds-checked copy on the user-supplied `notif_idx` / `flags` fields that index into the per-context notif slab; SEND_ZC completion `res`/`flags` rendered into the CQE is write-bounded.
- **PAX_KERNEXEC** — write-protects `io_notif_complete_tw_ext`, `io_tx_ubuf_complete`, and the `ubuf_info_msgzc.ops` callback vector after init.
- **PAX_RANDKSTACK** — per-call stack-offset randomization on every `io_alloc_notif`, `io_notif_flush`, `io_tx_ubuf_complete` entry.
- **PAX_REFCOUNT** — saturating wraparound trap on `ubuf_info.refcnt` and on `req->refs` taken across the notif task-work chain; on `mm->pinned_vm` accounting via `io_notif_account_mem`.
- **PAX_MEMORY_SANITIZE** — zeroes `struct io_notif_data` slab on free via `io_cache_free`; scrubs the `ubuf_info_msgzc` allocation on `__io_notif_complete_tw`.
- **PAX_UDEREF** — notif state is kernel-internal; defends any user-pointer aliasing by trapping the SEND_ZC `notif_idx` user-payload lane at prep.
- **PAX_RAP / kCFI** — forward-edge CFI on `ubuf_info.ops->complete` (`io_tx_ubuf_complete`) dispatch from the network stack.
- **GRKERNSEC_HIDESYM** — strips notif helpers and `ubuf_info` pointers from kallsyms-leaking debug paths.
- **GRKERNSEC_DMESG** — restricts `pr_debug` SEND_ZC and notif-flush traces to CAP_SYSLOG.
- **Zerocopy notify PAX_REFCOUNT** — `ubuf_info.refcnt` increments on every skb that references the notif fragment and decrements on `skb_zcopy_clear`/`skb_orphan_frags`/skb-free; saturating trap defends against per-skb-clone refcount overflow leading to use-after-free on the user-locked pinned page.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- io_uring/net.c SEND_ZC / SENDMSG_ZC issue paths (covered in `net.md` Tier-3)
- io_uring/zcrx.c RX zerocopy pool (covered in `zcrx.md` Tier-3 when expanded)
- net/core skbuff zerocopy infrastructure (`ubuf_info`, `skb_zcopy_*`) (covered separately under net/core)
- TCP / UDP transmit fast-path tx zerocopy handling (covered under net/ipv4 Tier-3 docs)
- io_uring/rsrc.c page accounting primitives `__io_account_mem` / `__io_unaccount_mem` (covered in `rsrc.md` Tier-3)
- Implementation code
