---
title: "Tier-3: io_uring/io_uring.c — io_uring core (SQ/CQ rings + per-fd ctx + opcode dispatch + worker pool)"
tags: ["tier-3", "io_uring", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

io_uring is Linux's async-IO submission interface that scales to millions of requests per second by sharing two ring buffers (Submission Queue + Completion Queue) between userspace + kernel via mmap. User submits SQEs (Submission Queue Entries) describing IO ops; kernel processes async via either inline (if non-blocking) or per-context worker thread (`io-wq`); kernel writes CQEs (Completion Queue Entries) back to CQ ring. Eliminates per-syscall overhead (one io_uring_enter syscall amortizes thousands of IOs); supports linked operations + buffer rings + FIXED_FD/buffer-pre-registration. Backs modern high-perf services: nginx, fio, rust-async-runtimes, qemu virtio-fs, etc.

This Tier-3 covers `io_uring/io_uring.c` (~3259 lines) + `io_uring.h` (~584) + `register.c` (~1038) + `sqpoll.c` (~569).

### Acceptance Criteria

- [ ] AC-1: Basic io_uring_setup + simple READ + io_uring_enter: 1 SQE submitted; 1 CQE returned.
- [ ] AC-2: Multi-op batch: 1000 IOs submitted in one io_uring_enter; all complete; no SQE/CQE loss.
- [ ] AC-3: Linked ops: read1 -> write1 -> read2 chain; chain breaks on read1 error.
- [ ] AC-4: Pre-registered fixed buffers: zero-copy read into pre-registered buffer.
- [ ] AC-5: SQPOLL: kernel kthread polls SQ ring; no io_uring_enter needed; per-IO latency reduced ~1us.
- [ ] AC-6: Cancel: ASYNC_CANCEL by user_data; pending op completes with -ECANCELED.
- [ ] AC-7: io_uring fio benchmark: 1M IOPS sustained on NVMe; matches bare-metal-baseline.
- [ ] AC-8: liburing test suite: full pass.
- [ ] AC-9: Per-fd close: pending ops canceled; ring teardown clean (no UAF).

### Architecture

`IoRingCtx`:

```
struct IoRingCtx {
  flags: u32,
  sq_ring: KArc<IoSqRingMmap>,                // mapped to user
  cq_ring: KArc<IoCqRingMmap>,
  sqes: KBox<[IoUringSqe; MAX_ENTRIES]>,
  sq_array: KBox<[u32; MAX_ENTRIES]>,         // index into sqes
  
  task: KArc<TaskStruct>,                      // submitting task
  user_files: Option<KBox<[KArc<File>; MAX_FILES]>>,  // FIXED_FILE pre-reg
  user_bufs: Option<KBox<[IoFixedBuf; MAX_BUFS]>>,
  
  io_wq: KArc<IoWq>,
  sq_thread: Option<KArc<TaskStruct>>,         // SQPOLL
  
  cq_wait: WaitQueueHead,
  cq_overflow_list: ListHead,
  cq_extra: AtomicU32,                         // overflow count
  
  refs: AtomicU64,
  ...
}

struct IoKiocb {
  ctx: KArc<IoRingCtx>,
  file: KArc<File>,
  task: KArc<TaskStruct>,
  user_data: u64,
  result: i32,
  cqe_flags: u32,
  flags: u32,                                  // io_kiocb internal flags
  link: Option<KArc<IoKiocb>>,                 // IOSQE_IO_LINK chain
  ...
}
```

`IoRingCtx::submit_one_sqe(ctx, req, sqe)`:
1. opcode := sqe.opcode.
2. If !io_op_defs[opcode].prep: return -EINVAL.
3. io_op_defs[opcode].prep(req, sqe).
4. If (sqe.flags & IOSQE_FIXED_FILE):
   - file := ctx.user_files[sqe.fd].
5. Else: file := fget(sqe.fd).
6. req.file = file; req.user_data = sqe.user_data; req.flags = sqe.flags.
7. If sqe.flags & IOSQE_IO_LINK: queue req in current chain.
8. Else: io_issue_sqe(req, IO_URING_F_NONBLOCK).

`IoKiocb::issue(req, issue_flags)`:
1. opcode := req.opcode.
2. ret := io_op_defs[opcode].issue(req, issue_flags).
3. If ret == -EAGAIN AND issue_flags & IO_URING_F_NONBLOCK:
   - io_queue_iowq(req) — defer to io-wq worker pool.
4. Else: io_req_complete(req, ret).

`IoRingCtx::post_cqe(ctx, user_data, res, flags)`:
1. tail := READ_ONCE(ctx.cq_ring.tail).
2. cqe_ptr := &ctx.cq_ring.cqes[tail & ctx.cq_ring.mask].
3. WRITE_ONCE(cqe_ptr.user_data, user_data).
4. WRITE_ONCE(cqe_ptr.res, res).
5. WRITE_ONCE(cqe_ptr.flags, flags).
6. smp_store_release(&ctx.cq_ring.tail, tail + 1).
7. If io_uring_enter waiters: cqring_wakeup.

### Out of Scope

- Per-op handlers (covered in `io_uring/per-op.md` future Tier-3s)
- io-wq worker pool details (covered in `io_uring/io-wq.md` future Tier-3)
- Buffer ring (covered in `io_uring/buffer-ring.md` future Tier-3)
- io_uring + cgroup integration (covered separately)
- io_uring tracepoints (covered in `io_uring/trace.md` future Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct io_ring_ctx` | per-io_uring-fd context | `io_uring::IoRingCtx` |
| `struct io_kiocb` | per-request kernel-side state | `IoKiocb` |
| `struct io_sq_ring` / `io_cq_ring` (UAPI) | per-shared-ring metadata | `IoSqRing` / `IoCqRing` |
| `struct io_uring_sqe` | submission entry (UAPI) | `IoUringSqe` |
| `struct io_uring_cqe` | completion entry (UAPI) | `IoUringCqe` |
| `io_uring_setup(entries, params)` | syscall: create ring + alloc backing | `IoRing::setup` |
| `io_uring_register(fd, op, arg, nr_args)` | syscall: per-fd register ops | `IoRing::register` |
| `io_uring_enter(fd, to_submit, min_complete, flags, ...)` | syscall: submit + wait | `IoRing::enter` |
| `io_submit_sqes(ctx, to_submit)` | per-call submit-N path | `IoRingCtx::submit_sqes` |
| `io_submit_sqe(ctx, req, sqe)` | per-SQE process | `IoRingCtx::submit_one_sqe` |
| `io_init_req(ctx, req, sqe)` | per-SQE → io_kiocb populate | `IoKiocb::init` |
| `io_issue_sqe(req, issue_flags)` | per-op dispatch via opcode-table | `IoKiocb::issue` |
| `io_req_complete(req, ret)` | per-req mark complete + post CQE | `IoKiocb::complete` |
| `io_post_cqe(ctx, user_data, res, flags)` | post CQE to ring | `IoRingCtx::post_cqe` |
| `io_cqring_wakeup(ctx)` | wake io_uring_enter waiters | `IoRingCtx::cqring_wakeup` |
| `io_wq_submit_work(work)` (io_wq.c) | worker-pool fallback path | `IoWq::submit_work` |
| `io_sqpoll_wait_sq(ctx)` (sqpoll.c) | SQPOLL kthread main loop | `IoSqpoll::wait_sq` |
| `io_register_files(ctx, args)` (register.c) | IORING_REGISTER_FILES | `IoRing::register_files` |
| `io_register_buffers(ctx, args)` | IORING_REGISTER_BUFFERS | `IoRing::register_buffers` |
| `io_apoll_task_func(req)` | async-poll fallback when can't issue inline | `IoKiocb::apoll_task_func` |
| `io_uring_try_cancel_requests(...)` | cancel pending on close | `IoRingCtx::try_cancel` |
| `io_match_task_safe(req, tctx)` | per-task req filter for cancel | `IoKiocb::match_task` |

### compatibility contract

REQ-1: Per-`io_ring_ctx`:
- 4KiB SQ-ring + 4KiB CQ-ring + 8KiB sqes-array (per-N entries × sqe-size).
- Pre-registered files array.
- Pre-registered buffers array.
- Per-task `io_uring_task` ref-count.
- Per-context worker pool (`io-wq`).
- Submit-state: `cq_overflow_list` for CQEs that didn't fit.

REQ-2: SQE (64 bytes):
- opcode (u8): IORING_OP_NOP / _READV / _WRITEV / _READ / _WRITE / _ACCEPT / _CONNECT / _RECV / _SEND / _OPENAT / _CLOSE / _STATX / _FALLOCATE / _ASYNC_CANCEL / _LINK_TIMEOUT / _MULTISHOT_*.
- flags (u8): IOSQE_FIXED_FILE / _IO_DRAIN / _IO_LINK / _IO_HARDLINK / _ASYNC / _BUFFER_SELECT / _CQE_SKIP_SUCCESS.
- ioprio (u16).
- fd (s32).
- off (u64).
- addr (u64).
- len (u32).
- per-op-specific union.
- user_data (u64; opaque to kernel).

REQ-3: CQE (16 bytes; or 32 bytes with IORING_SETUP_CQE32):
- user_data (u64; copied from SQE).
- res (s32; return value or -errno).
- flags (u32; IORING_CQE_F_BUFFER + buffer-id, etc.).

REQ-4: Per-`io_kiocb` (per-request):
- ctx (back-ref).
- file (Resolves fd to file).
- result (kernel ret value).
- task (submitting task).
- linked-list (for IO_LINK chains).
- flags + io-op-specific data.

REQ-5: io_uring_setup flow:
1. Validate `entries` (must be ≤ IORING_MAX_ENTRIES = 32768).
2. Allocate `io_ring_ctx`.
3. Allocate SQ ring (entries-rounded-up-pow2 × sqe-pointer-size + ring-metadata).
4. Allocate sqes array (entries × sizeof(io_uring_sqe)).
5. Allocate CQ ring (2 × entries entries × sizeof(io_uring_cqe) + ring-metadata).
6. Init worker pool.
7. Anon-file via anon_inode_getfd("[io_uring]", &io_uring_fops, ctx).
8. Return fd.

REQ-6: io_uring_enter(fd, to_submit, min_complete, flags) flow:
1. Resolve fd → io_ring_ctx.
2. If to_submit > 0: io_submit_sqes(ctx, to_submit).
3. If min_complete > 0 OR (flags & IORING_ENTER_GETEVENTS):
   - Wait on ctx.cq_wait until at least min_complete CQEs available.
4. Return submit count.

REQ-7: io_submit_sqes(ctx, to_submit) flow:
1. Read SQ-ring head.
2. For i = 0 to to_submit:
   - sqe := ctx.sq_array[(head + i) & sq_ring.mask].
   - Allocate io_kiocb req from per-CPU pool.
   - io_init_req(ctx, req, sqe): populate from SQE.
   - io_issue_sqe(req, IO_URING_F_NONBLOCK):
     - Dispatch via opcode-table to per-op handler.
     - If returns -EAGAIN: queue for async (io-wq).
     - Else: io_req_complete(req, ret).
3. Update SQ ring head.

REQ-8: io_issue_sqe via opcode-table:
- `io_op_defs[opcode]`: per-op metadata + prep + issue handlers.
- Per-op `prep` (validate sqe fields).
- Per-op `issue` (do the IO; may sleep or not depending on file).

REQ-9: Per-CQE post:
1. tail := ctx.cq_ring.tail.
2. cqe := ctx.cq_ring.cqes[(tail) & cq_ring.mask].
3. cqe.user_data = req.user_data; cqe.res = ret; cqe.flags = req.cqe_flags.
4. smp_store_release(&ctx.cq_ring.tail, tail + 1).
5. Wake io_uring_enter sleepers.

REQ-10: io-wq worker pool:
- Per-context worker_pool with bounded + unbounded workers.
- Bounded: workers count == concurrent-IO count cap.
- Unbounded: workers grow on demand.
- Per-worker thread runs queue-of-pending io_kiocb.

REQ-11: SQPOLL mode (IORING_SETUP_SQPOLL):
- Kernel kthread polls SQ ring for new entries; eliminates io_uring_enter syscall per submission.
- Used for ultra-low-latency.

REQ-12: Linked operations (IOSQE_IO_LINK / _HARDLINK):
- Subsequent SQEs in chain depend on prior; chain breaks on error (LINK) or continues (HARDLINK).

REQ-13: IORING_REGISTER_* ops (register.c):
- _BUFFERS: pre-register user-buffer for FIXED_BUF reads.
- _FILES: pre-register file table (avoid fdget per-op).
- _EVENTFD: bind eventfd to ring for async wakeup notification.
- _PROBE: query supported opcodes.
- _PERSONALITY: register cred for per-personality ops.
- _RESTRICTIONS: lock down which ops allowed.
- _RING_FDS: nested io_uring fd registration.

REQ-14: Per-buffer-ring (IORING_REGISTER_PBUF_RING):
- Pre-allocated buffer ring; per-recv consumes head buffer; per-send commits tail.
- Used by recvmsg multi-shot + zero-copy.

REQ-15: Cancel ops:
- IORING_OP_ASYNC_CANCEL: cancel pending op by user_data or by fd.
- io_uring_try_cancel_requests on fd close.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sq_ring_indices_bounded` | OOB | per-SQE index bounded by sq_ring.mask + 1; defense against OOB read. |
| `cq_ring_indices_bounded` | OOB | per-CQE index bounded by cq_ring.mask + 1. |
| `kiocb_refcount_no_underflow` | INVARIANT | per-IoKiocb refcount ≥ 0. |
| `linked_chain_no_cycles` | INVARIANT | per-chain detected via prep-time validation. |
| `fixed_file_index_validated` | INVARIANT | sqe.fd < ctx.user_files.len when IOSQE_FIXED_FILE. |

### Layer 2: TLA+

`io_uring/sq_cq_ordering.tla`:
- Per-SQE state ∈ {Free, Submitted, Issuing, Completing, Done}.
- Properties:
  - `safety_cqe_ordering` — CQE.user_data matches SQE.user_data.
  - `safety_no_double_post` — per-IoKiocb post_cqe called exactly once.
  - `liveness_eventually_complete` — every Submitted eventually Done.

`io_uring/linked_chain.tla`:
- Per-chain state: linear chain of req's.
- Properties:
  - `safety_chain_break_on_error` (LINK) — if req[i] fails, req[i+1..] canceled.
  - `safety_hardlink_continues` — HARDLINK ignores per-link error.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `IoRing::setup` post: ctx allocated; rings mmap'd; anon-fd created | `IoRing::setup` |
| `IoRingCtx::submit_sqes` post: each consumed SQE issued or queued; SQ head advanced by consumed-count | `IoRingCtx::submit_sqes` |
| `IoKiocb::complete` post: CQE posted; req refcount dropped | `IoKiocb::complete` |
| `IoRingCtx::post_cqe` post: CQ tail advanced by 1; field-write order = user_data → res → flags → release-store-tail | `IoRingCtx::post_cqe` |
| Per-IoRingCtx.user_files indexing bounded | `IoRing::register_files` setup |

### Layer 4: Verus/Creusot functional

`io_uring submit + complete = equivalent to native syscall result for each op`: per-IO the user observes the same final outcome (return value, side effects on file/socket) as if they had called the corresponding native syscall directly.

### hardening

(Inherits row-1 features from `io_uring/00-overview.md` § Hardening.)

io_uring-specific reinforcement:

- **Per-context refcount tracking** — defense against close-during-submit UAF.
- **IORING_MAX_ENTRIES cap (32768)** — defense against unbounded ring-size memory exhaustion.
- **Per-task io_uring_task ref** — defense against task-exit-with-active-ring leaving orphaned context.
- **CQ overflow_list** — defense against CQE-loss when CQ ring full; spillover to overflow list.
- **Per-op opcode validated** at prep — defense against unsupported opcode reaching issue path.
- **IOSQE_FIXED_FILE bounds check** — defense against attacker-supplied fd-index OOB.
- **IORING_SETUP_R_DISABLED + ENABLE_RINGS** — defense against ring being usable before full setup.
- **IORING_REGISTER_RESTRICTIONS** — defense against ops being usable beyond declared scope.
- **CQE field-write release ordering** — defense against guest-reading half-formed CQE.
- **Per-task io-wq worker bounded** — defense against unbounded worker thread proliferation.
- **Per-context cancel on fd-close** — defense against post-close worker accessing freed file.
- **SQPOLL kthread bounded by per-context CPU mask** — defense against SQPOLL hogging cpus.
- **Per-buffer-ring head/tail validated** — defense against malicious user advancing tail past valid range.
- **IO_LINK chain depth bounded** — defense against pathological chain causing stack overflow on per-link complete.

