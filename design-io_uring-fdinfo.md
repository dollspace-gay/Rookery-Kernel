---
title: "Tier-3: io_uring/fdinfo.c — /proc/<pid>/fdinfo for io_uring fds"
tags: ["tier-3", "io_uring", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`io_uring_show_fdinfo()` is the `file_operations.show_fdinfo` hook for every io_uring fd. The VFS invokes it when a process reads `/proc/<pid>/fdinfo/<n>` for an io_uring fd. It produces a human-readable, line-oriented dump of the ring's runtime state: SQ/CQ head and tail pointers (both userspace-visible *and* kernel-cached), the entries-in-flight counts, the actual SQE and CQE arrays decoded by opcode, the SQPOLL thread identity and CPU accounting, the registered file table and buffer table, the polling hash list, the CQ overflow list, and (under `CONFIG_NET_RX_BUSY_POLL`) NAPI tracking parameters. The decode is intentionally *imprecise* — `cached_sq_head` and `cached_cq_tail` are read without holding `uring_lock`, and `sq_tail`/`cq_head` are written by userspace — because fdinfo is a diagnostic tool primarily consulted "when it is stuck." The top-level entry point uses `mutex_trylock(&ctx->uring_lock)` to avoid the seq_file-vs-uring-mutex ABBA deadlock: if the trylock fails, fdinfo prints nothing rather than risk a deadlock. SQE-128 (large SQEs) and CQE-32 (large CQEs) are handled with shift/stride adjustments. `IORING_SETUP_NO_SQARRAY` bypasses the indirect sq_array. `IORING_SETUP_SQE_MIXED` permits per-SQE 128-byte entries flagged via opdef `is_128`. Critical for: post-mortem ring diagnosis, container observability, debugging SQPOLL stalls, hung-task triage.

This Tier-3 covers `io_uring/fdinfo.c` (~271 lines).

### Acceptance Criteria

- [ ] AC-1: `cat /proc/$$/fdinfo/N` (N = io_uring fd) emits the documented header lines in the documented order: SqMask, SqHead, SqTail, CachedSqHead, CqMask, CqHead, CqTail, CachedCqTail, SQEs.
- [ ] AC-2: SQE-array section: every in-flight SQE between (sq_head, sq_tail) prints exactly one row containing opcode name (via `io_uring_get_opcode`), fd, flags, off, addr, rw_flags, buf_index, user_data.
- [ ] AC-3: IORING_SETUP_SQE128 ring: each SQE row also includes `e0..e7` extra-data fields.
- [ ] AC-4: IORING_SETUP_NO_SQARRAY ring: sq_idx derived directly from `entry & sq_mask` (no indirection through sq_array).
- [ ] AC-5: IORING_SETUP_SQE_MIXED ring with `is_128`-flagged opcode at sq_mask boundary: prints "corrupted sqe, wrapping 128B entry" and breaks loop.
- [ ] AC-6: Opcode ≥ IORING_OP_LAST in SQE: row skipped (not printed).
- [ ] AC-7: CQE-array section: prints CQEs count, then one row per (cq_head, cq_tail) CQE. IORING_SETUP_CQE32 or per-CQE F_32 ⟹ row includes `extra1`/`extra2`.
- [ ] AC-8: SQPOLL ring with live thread: emits SqThread (positive pid), SqThreadCpu (≥ 0 if pinned, else -1), SqTotalTime, SqWorkTime.
- [ ] AC-9: Non-SQPOLL ring: SqThread=-1, SqThreadCpu=-1, SqTotalTime=0, SqWorkTime=0.
- [ ] AC-10: SQPOLL ring during thread termination race (sq->thread == NULL): no crash; still emits sentinels.
- [ ] AC-11: UserFiles list shows path for each registered file (escaped via `seq_file_path`); UserBufs list shows ubuf/len or `<none>` for empty slot.
- [ ] AC-12: PollList walks all hash buckets and emits one line per in-flight poll/cancel-table entry.
- [ ] AC-13: CqOverflowList walks under completion_lock and emits one line per overflowed CQE.
- [ ] AC-14: NAPI section: present when CONFIG_NET_RX_BUSY_POLL=y; absent (no output) otherwise.
- [ ] AC-15: uring_lock contended: fdinfo returns empty output rather than blocking (no partial output, no deadlock).

### Architecture

```
struct FdInfoCtx<'a> {
  ctx:        &'a IoRingCtx,
  m:          &'a mut SeqFile,
}

// VFS dispatch
fn FdInfo::show(m: &mut SeqFile, file: &File) {
  let ctx = file.private_data::<IoRingCtx>();
  if ctx.uring_lock.try_lock() {
    FdInfo::show_inner(ctx, m);
    ctx.uring_lock.unlock();
  }
}
```

`FdInfo::show_inner(ctx, m)`:
1. Compute sq_mask = ctx.sq_entries - 1; cq_mask = ctx.cq_entries - 1.
2. sq_shift = if ctx.flags.contains(SETUP_SQE128) { 1 } else { 0 }.
3. Read userspace-visible head/tail via READ_ONCE: sq_head, sq_tail, cq_head, cq_tail.
4. Print top section (SqMask, SqHead, SqTail, CachedSqHead via data_race(ctx.cached_sq_head), CqMask, CqHead, CqTail, CachedCqTail via data_race(ctx.cached_cq_tail), SQEs = sq_tail - sq_head).
5. sq_entries = min(sq_tail - sq_head, ctx.sq_entries).
6. For i in 0..sq_entries:
   - entry = i + sq_head.
   - If SETUP_NO_SQARRAY: sq_idx = entry & sq_mask.
   - Else: sq_idx = READ_ONCE(ctx.sq_array[entry & sq_mask]).
   - If sq_idx > sq_mask: continue.
   - sqe = &ctx.sq_sqes[sq_idx << sq_shift].
   - opcode = READ_ONCE(sqe.opcode).
   - If opcode ≥ IORING_OP_LAST: continue.
   - opcode = array_index_nospec(opcode, IORING_OP_LAST).
   - sqe128 = false.
   - If sq_shift == 1: sqe128 = true.
   - Else if io_issue_defs[opcode].is_128:
     - If !ctx.flags.contains(SETUP_SQE_MIXED): print "invalid sqe, 128B entry on non-mixed sq" and break.
     - If sq_idx == sq_mask: print "corrupted sqe, wrapping 128B entry" and break.
     - sq_head += 1; i += 1; sqe128 = true.
   - Print SQE row (opcode-name, fd, flags, off, addr, rw_flags, buf_index, user_data).
   - If sqe128: print e0..e7 from (sqe + 1).
   - Newline; cond_resched.
7. Print "CQEs:" count.
8. cq_entries = min(cq_tail - cq_head, ctx.cq_entries).
9. For i in 0..cq_entries:
   - cqe = &r.cqes[cq_head & cq_mask].
   - cqe32 = (cqe.flags & CQE_F_32) || ctx.flags.contains(SETUP_CQE32).
   - Print CQE row (idx, user_data, res, flags).
   - If cqe32: print extra1, extra2.
   - Newline; cq_head += 1; if cqe32 { cq_head += 1; i += 1; }; cond_resched.
10. SQPOLL section (defaults sq_pid=-1, sq_cpu=-1, totals=0):
    - If SETUP_SQPOLL: rcu_read_lock; tsk = rcu_dereference(ctx.sq_data.thread); if tsk { get_task_struct(tsk); rcu_read_unlock; usec = io_sq_cpu_usec(tsk); put_task_struct(tsk); sq_pid = sq.task_pid; sq_cpu = sq.sq_cpu; total = usec; work = sq.work_time; } else { rcu_read_unlock; }.
    - Print SqThread, SqThreadCpu, SqTotalTime, SqWorkTime.
11. Print UserFiles count and iterate file_table.data.nodes[i] → io_slot_file → seq_file_path(m, f, " \t\n\\").
12. Print UserBufs count and iterate buf_table.nodes[i].buf → "%5u: 0x%llx/%u" or "<none>".
13. Print "PollList:"; for each bucket in ctx.cancel_table.hbs walk hash_node list and print "  op=%d, task_works=%d".
14. Print "CqOverflowList:"; spin_lock(completion_lock); iterate cq_overflow_list printing user_data/res/flags; spin_unlock.
15. FdInfo::napi_show(ctx, m).

`FdInfo::napi_show(ctx, m)` (CONFIG_NET_RX_BUSY_POLL):
1. mode = READ_ONCE(ctx.napi_track_mode).
2. INACTIVE ⟹ "NAPI:\tdisabled".
3. DYNAMIC ⟹ FdInfo::napi_common(ctx, m, "dynamic").
4. STATIC  ⟹ FdInfo::napi_common(ctx, m, "static").
5. default ⟹ "NAPI:\tunknown mode (%u)".

`FdInfo::napi_common(ctx, m, strategy)`:
1. "NAPI:\tenabled".
2. "napi tracking:\t%s" (strategy).
3. "napi_busy_poll_dt:\t%llu" (ctx.napi_busy_poll_dt).
4. "napi_prefer_busy_poll:\ttrue|false".

`FdInfo::show(m, file)`:
1. ctx = file.private_data.
2. If ctx.uring_lock.try_lock():
   - FdInfo::show_inner(ctx, m).
   - ctx.uring_lock.unlock().

### Out of Scope

- io_uring core SQE/CQE submission/completion (covered in `io_uring-core.md` Tier-3)
- SQPOLL thread lifecycle + io_sq_cpu_usec implementation (covered in `sqpoll.md` Tier-3)
- File-table and buffer-table registration mechanics (covered in `rsrc.md` Tier-3)
- NAPI busy-poll tracking implementation (covered in `napi.md` if expanded; net-side internals out of scope)
- /proc filesystem and seq_file infrastructure (kernel-fs-side, out of io_uring scope)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `io_uring_show_fdinfo()` | per-show_fdinfo VFS hook (trylocks uring_lock) | `FdInfo::show` |
| `__io_uring_show_fdinfo()` | per-decode under uring_lock | `FdInfo::show_inner` |
| `napi_show_fdinfo()` | per-NAPI-tracking-mode dispatch | `FdInfo::napi_show` |
| `common_tracking_show_fdinfo()` | per-NAPI static/dynamic common section | `FdInfo::napi_common` |
| `io_uring_get_opcode()` | per-opcode → string name | shared (`io_uring/opdef`) |
| `io_issue_defs[].is_128` | per-opcode 128-byte-SQE marker | shared (`io_uring/opdef`) |
| `array_index_nospec()` | per-Spectre-v1 mitigation when indexing opdef[] | shared (kernel-generic) |
| `io_slot_file()` | per-file-table-node → struct file | shared (`io_uring/filetable`) |
| `seq_file_path()` | per-file path serialization (escapes `\ \t\n\\`) | shared (`fs/seq_file`) |
| `io_sq_cpu_usec()` | per-SQPOLL CPU usage accounting | shared (`io_uring/sqpoll`) |
| `task_work_pending()` | per-PollList task-work flag | shared (kernel-generic) |
| `data_race()` | per-cached_sq_head / cached_cq_tail unlocked read | shared (kernel-generic) |
| `mutex_trylock()` | per-uring_lock non-blocking acquisition | shared (kernel-generic) |
| `cond_resched()` | per-iter scheduler yield in SQE/CQE walks | shared (kernel-generic) |
| `struct io_ring_ctx` | per-ring context (consumed read-only) | shared |
| `struct io_rings` | per-userspace-mmaped sq/cq head/tail/cqes | shared |
| `struct io_overflow_cqe` | per-CqOverflowList entry | shared |
| `struct io_hash_bucket` | per-cancel_table hash bucket walked for PollList | shared |
| `struct io_mapped_ubuf` | per-registered-buffer descriptor (ubuf, len) | shared |
| `struct io_uring_sqe` | per-SQE wire format (`opcode`, `fd`, `flags`, `off`, `addr`, `rw_flags`, `buf_index`, `user_data`) | shared |
| `struct io_uring_cqe` | per-CQE wire format (`user_data`, `res`, `flags`, `big_cqe[2]`) | shared |

### compatibility contract

REQ-1: Top-level entry `io_uring_show_fdinfo(m, file)`:
- ctx = file->private_data (io_uring fd's `private_data` always points at io_ring_ctx).
- Acquire `mutex_trylock(&ctx->uring_lock)`:
  - Acquired ⟹ call `__io_uring_show_fdinfo(ctx, m)`; `mutex_unlock(&ctx->uring_lock)`.
  - Failed ⟹ return silently (print nothing); the ABBA-vs-seq_file rule is "trylock or skip."

REQ-2: SQE-128 mode detection:
- sq_shift = 1 if (ctx->flags & IORING_SETUP_SQE128); else 0.
- All sqe-array indexing uses `sq_idx << sq_shift` byte offset into ctx->sq_sqes.

REQ-3: Mask + head/tail read:
- sq_mask = ctx->sq_entries - 1.
- cq_mask = ctx->cq_entries - 1.
- sq_head = READ_ONCE(r->sq.head); sq_tail = READ_ONCE(r->sq.tail).
- cq_head = READ_ONCE(r->cq.head); cq_tail = READ_ONCE(r->cq.tail).
- `cached_sq_head` and `cached_cq_tail` read via `data_race()` (no lock vs the writers).

REQ-4: Output fields (top section, in order):
- `SqMask:\t0x%x` (sq_mask).
- `SqHead:\t%u` (sq_head — userspace-visible).
- `SqTail:\t%u` (sq_tail — userspace-visible).
- `CachedSqHead:\t%u` (ctx->cached_sq_head, kernel-side).
- `CqMask:\t0x%x` (cq_mask).
- `CqHead:\t%u` (cq_head — userspace-visible).
- `CqTail:\t%u` (cq_tail — userspace-visible).
- `CachedCqTail:\t%u` (ctx->cached_cq_tail, kernel-side).
- `SQEs:\t%u` (sq_tail - sq_head — outstanding SQEs).

REQ-5: SQE-array decode loop:
- sq_entries = min(sq_tail - sq_head, ctx->sq_entries).
- For i in 0..sq_entries:
  - entry = i + sq_head.
  - IORING_SETUP_NO_SQARRAY ⟹ sq_idx = entry & sq_mask (direct).
  - else ⟹ sq_idx = READ_ONCE(ctx->sq_array[entry & sq_mask]) (indirect).
  - sq_idx > sq_mask ⟹ skip this entry (corrupt / userspace-mutated).
  - sqe = &ctx->sq_sqes[sq_idx << sq_shift].
  - opcode = READ_ONCE(sqe->opcode); opcode ≥ IORING_OP_LAST ⟹ skip.
  - opcode = array_index_nospec(opcode, IORING_OP_LAST).
  - sqe128 decision:
    - sq_shift == 1 ⟹ sqe128 = true (whole ring is 128B).
    - else io_issue_defs[opcode].is_128 ⟹ require IORING_SETUP_SQE_MIXED; print "invalid sqe, 128B entry on non-mixed sq" and break if not mixed; require sq_idx ≠ sq_mask (cannot wrap); else sq_head++, i++, sqe128 = true.
  - Print one row:
    - `%5u: opcode:%s, fd:%d, flags:%x, off:%llu, addr:0x%llx, rw_flags:0x%x, buf_index:%d user_data:%llu`.
  - If sqe128 ⟹ append `, e0:0x%llx, e1:0x%llx, … e7:0x%llx` (all 8 u64s).
  - Newline; `cond_resched()`.

REQ-6: CQE-array decode loop:
- Print `CQEs:\t%u` (cq_tail - cq_head).
- cq_entries = min(cq_tail - cq_head, ctx->cq_entries).
- For i in 0..cq_entries:
  - cqe = &r->cqes[cq_head & cq_mask].
  - cqe32 = (cqe->flags & IORING_CQE_F_32) ∨ (ctx->flags & IORING_SETUP_CQE32).
  - Print `%5u: user_data:%llu, res:%d, flags:%x` with idx = cq_head & cq_mask.
  - cqe32 ⟹ append `, extra1:%llu, extra2:%llu` (cqe->big_cqe[0], [1]).
  - Newline; cq_head++; cqe32 ⟹ cq_head++, i++ (stride-of-two for 32-byte CQEs).
  - `cond_resched()`.

REQ-7: SQPOLL section:
- sq_pid = -1; sq_cpu = -1; sq_total_time = 0; sq_work_time = 0 (defaults).
- ctx->flags & IORING_SETUP_SQPOLL ⟹
  - sq = ctx->sq_data.
  - rcu_read_lock().
  - tsk = rcu_dereference(sq->thread).
  - tsk ≠ NULL ⟹ get_task_struct(tsk); rcu_read_unlock(); usec = io_sq_cpu_usec(tsk); put_task_struct(tsk); sq_pid = sq->task_pid; sq_cpu = sq->sq_cpu; sq_total_time = usec; sq_work_time = sq->work_time.
  - tsk == NULL ⟹ rcu_read_unlock() (raced with SQPOLL termination).
- Print:
  - `SqThread:\t%d` (sq_pid).
  - `SqThreadCpu:\t%d` (sq_cpu).
  - `SqTotalTime:\t%llu`.
  - `SqWorkTime:\t%llu`.

REQ-8: Registered file table:
- Print `UserFiles:\t%u` (ctx->file_table.data.nr).
- For i in 0..ctx->file_table.data.nr:
  - f = (ctx->file_table.data.nodes[i]) ? io_slot_file(...) : NULL.
  - f ≠ NULL ⟹ `%5u: ` then `seq_file_path(m, f, " \t\n\\")` then newline.

REQ-9: Registered buffer table:
- Print `UserBufs:\t%u` (ctx->buf_table.nr).
- For i in 0..ctx->buf_table.nr:
  - buf = (ctx->buf_table.nodes[i]) ? ctx->buf_table.nodes[i]->buf : NULL.
  - buf ≠ NULL ⟹ `%5u: 0x%llx/%u` (buf->ubuf, buf->len).
  - buf == NULL ⟹ `%5u: <none>`.

REQ-10: PollList walk:
- `seq_puts(m, "PollList:\n")`.
- For i in 0..(1 << ctx->cancel_table.hash_bits):
  - hb = &ctx->cancel_table.hbs[i].
  - hlist_for_each_entry(req, &hb->list, hash_node):
    - Print `  op=%d, task_works=%d` (req->opcode, task_work_pending(req->tctx->task) → 0 or 1).

REQ-11: CQ overflow list:
- `seq_puts(m, "CqOverflowList:\n")`.
- spin_lock(&ctx->completion_lock).
- list_for_each_entry(ocqe, &ctx->cq_overflow_list, list):
  - cqe = &ocqe->cqe.
  - Print `  user_data=%llu, res=%d, flags=%x`.
- spin_unlock(&ctx->completion_lock).

REQ-12: NAPI section (CONFIG_NET_RX_BUSY_POLL):
- mode = READ_ONCE(ctx->napi_track_mode).
- IO_URING_NAPI_TRACKING_INACTIVE ⟹ `NAPI:\tdisabled`.
- IO_URING_NAPI_TRACKING_DYNAMIC ⟹ common section with `napi tracking:\tdynamic`.
- IO_URING_NAPI_TRACKING_STATIC ⟹ common section with `napi tracking:\tstatic`.
- default ⟹ `NAPI:\tunknown mode (%u)`.

REQ-13: NAPI common section:
- `NAPI:\tenabled`.
- `napi tracking:\t<strategy>`.
- `napi_busy_poll_dt:\t%llu` (ctx->napi_busy_poll_dt).
- ctx->napi_prefer_busy_poll ⟹ `napi_prefer_busy_poll:\ttrue`; else `napi_prefer_busy_poll:\tfalse`.

REQ-14: Without CONFIG_NET_RX_BUSY_POLL:
- napi_show_fdinfo is the empty inline (no output).

REQ-15: Locking:
- Outer (io_uring_show_fdinfo) holds `mutex_trylock(&ctx->uring_lock)`; release after inner returns.
- Inner uses spin_lock(&ctx->completion_lock) only for the CqOverflowList traversal.
- Inner reads `cached_sq_head` and `cached_cq_tail` with `data_race()` macros — explicit "imprecise OK" reads.
- SQPOLL thread access is RCU-guarded; tsk lifetime extended via get_task_struct over RCU drop boundary.

REQ-16: Imprecision contract:
- Documentation in source: "we may get imprecise sqe and cqe info if uring is actively running … But it's ok since we usually use these info when it is stuck."
- Output is informational; no operational decisions should rely on consistency.

REQ-17: cond_resched cadence:
- Once per SQE row printed (inside the SQE loop).
- Once per CQE row printed (inside the CQE loop).

REQ-18: `__cold` hot-path discipline:
- `io_uring_show_fdinfo`, `__io_uring_show_fdinfo` (file-scope local), `napi_show_fdinfo`, `common_tracking_show_fdinfo` all `__cold` — never inlined into hot dispatch.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `trylock_or_silent` | INVARIANT | per-io_uring_show_fdinfo: !trylock(uring_lock) ⟹ zero output, no recursion. |
| `opcode_bounds_via_nospec` | INVARIANT | per-SQE-loop: opcode < IORING_OP_LAST after `array_index_nospec`. |
| `sq_idx_mask_bounded` | INVARIANT | per-SQE-loop: sq_idx > sq_mask ⟹ skip (no out-of-bounds index of sq_sqes). |
| `sqe128_requires_mixed_or_full` | INVARIANT | per-SQE-loop: `is_128` opcode requires SETUP_SQE128 ∨ SETUP_SQE_MIXED. |
| `sqe128_no_wrap` | INVARIANT | per-SQE-loop: 128B entry rejected at sq_idx == sq_mask boundary. |
| `cqe_stride_two_for_32` | INVARIANT | per-CQE-loop: cqe32 ⟹ cq_head advances by 2; index uses original head & mask. |
| `sqpoll_tsk_lifetime` | INVARIANT | per-SQPOLL: get_task_struct before rcu_read_unlock, put_task_struct after read. |
| `sqpoll_thread_null_tolerated` | INVARIANT | per-SQPOLL: tsk == NULL ⟹ default sentinels (-1, 0); no deref. |
| `cqoverflow_under_completion_lock` | INVARIANT | per-CqOverflowList: spin_lock(completion_lock) held during walk. |
| `cond_resched_per_entry` | INVARIANT | per-SQE/CQE loops: cond_resched called every iter (no infinite loop without yield). |
| `imprecise_read_via_data_race` | INVARIANT | per-cached_sq_head / cached_cq_tail: read via data_race() macro (suppresses KCSAN). |
| `file_table_node_optional` | INVARIANT | per-file-list: nodes[i] may be NULL; only deref if non-NULL. |
| `buf_table_node_optional` | INVARIANT | per-buf-list: same NULL-tolerance. |
| `napi_disabled_under_no_busy_poll` | INVARIANT | per-CONFIG_NET_RX_BUSY_POLL=n: napi_show is no-op. |

### Layer 2: TLA+

`io_uring/fdinfo.tla`:
- Per-show_fdinfo state machine: trylock → decode → unlock; concurrent SQE/CQE producers + consumers.
- Properties:
  - `safety_no_deadlock` — fdinfo never blocks on uring_lock (trylock-only).
  - `safety_no_partial_output_on_skip` — failed trylock prints zero bytes (not partial).
  - `safety_opcode_in_range` — every printed opcode satisfies opcode < IORING_OP_LAST.
  - `safety_index_in_range` — every sq_sqes[i] access uses i ≤ sq_mask (post-shift fits within ring).
  - `safety_cqoverflow_walk_locked` — completion_lock held for entire cq_overflow_list traversal.
  - `safety_sqpoll_ref_balanced` — get_task_struct exactly balanced by put_task_struct on every path through SQPOLL block.
  - `liveness_show_terminates` — show_fdinfo always returns (bounded by ctx.sq_entries + ctx.cq_entries + table sizes + hash_buckets).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `FdInfo::show` post: ctx.uring_lock unlocked iff it was locked | `FdInfo::show` |
| `FdInfo::show_inner` post: m written with header + SQE + CQE + SQPOLL + Files + Bufs + PollList + CqOverflow + NAPI sections in order | `FdInfo::show_inner` |
| `FdInfo::show_inner` SQE invariant: ∀ printed rows, sq_idx ≤ sq_mask ∧ opcode < IORING_OP_LAST | `FdInfo::show_inner` |
| `FdInfo::show_inner` CQE invariant: ∀ printed rows, idx = (head_at_iter) & cq_mask | `FdInfo::show_inner` |
| `FdInfo::show_inner` SQPOLL post: tsk refcount net-zero | `FdInfo::show_inner` |
| `FdInfo::show_inner` CqOverflow post: completion_lock locked-then-unlocked | `FdInfo::show_inner` |
| `FdInfo::napi_show` post: produces ≤ 4 lines or "NAPI:\tdisabled" / "NAPI:\tunknown mode" | `FdInfo::napi_show` |

### Layer 4: Verus/Creusot functional

`Per-/proc/<pid>/fdinfo/<n> read → seq_file callback → io_uring_show_fdinfo → (trylock OR skip) → __io_uring_show_fdinfo (headers + SQE walk + CQE walk + SQPOLL + Files + Bufs + PollList + CqOverflowList + NAPI)` semantic equivalence: per the byte-for-byte text format consumed by liburing-test-tools and io_uring(7) diagnostic conventions. Field names and order match `Documentation/filesystems/proc.rst` § io_uring fdinfo.

### hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

io_uring/fdinfo reinforcement:

- **Per-mutex_trylock(uring_lock)** — defense against per-seq_file-vs-uring-mutex ABBA deadlock when reader holds seq lock.
- **Per-array_index_nospec on opcode** — defense against per-Spectre-v1 speculation through opdef[].
- **Per-opcode ≥ IORING_OP_LAST skipped** — defense against per-malicious-userspace OOB read of opdef[].
- **Per-sq_idx > sq_mask skipped** — defense against per-userspace-mutated sq_array poison.
- **Per-data_race on cached_sq_head / cached_cq_tail** — defense against per-KCSAN false-positive flood.
- **Per-cond_resched in SQE/CQE loops** — defense against per-large-ring soft-lockup.
- **Per-get_task_struct over rcu_read_unlock boundary** — defense against per-SQPOLL-thread freed-while-printed UAF.
- **Per-spin_lock(completion_lock) only for CqOverflowList** — defense against per-overlong critical section starving I/O.
- **Per-NULL-tolerant file_table.data.nodes[i] / buf_table.nodes[i]** — defense against per-partially-registered table walk.
- **Per-seq_file_path escape mask " \t\n\\"** — defense against per-malicious-path-injection into fdinfo line format.
- **Per-128B-SQE wrap-at-sq_mask rejected** — defense against per-corrupted-ring half-read.
- **Per-`__cold` annotation** — defense against per-icache-pollution of hot dispatch by diagnostic code.

