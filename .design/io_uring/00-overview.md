# Subsystem: io_uring/ — io_uring async I/O framework

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: in-v0
upstream-paths:
  - io_uring/
  - include/uapi/linux/io_uring.h
  - include/linux/io_uring.h
  - include/linux/io_uring_types.h
-->

## Summary
Tier-2 overview for `io_uring/` — the high-performance async I/O interface using two shared-memory rings (Submission + Completion) between userspace and kernel. Owns the syscall trio (`io_uring_setup`, `io_uring_enter`, `io_uring_register`), the per-context state machine, the per-operation handlers (read/write/openat/close/fsync/recvmsg/sendmsg/poll/epoll/timeout/cancel/futex/openat2/recv-zerocopy/send-zerocopy/io-wq/sqpoll/etc.), the file-descriptor table, the buffer-ring (provided buffers), and the memory-mapped queue handling.

io_uring is sibling to (not under) fs/, net/, mm/ — it consumes those subsystems' operations behind a fast async ring interface. Originally lived in `fs/io_uring.c`; promoted to its own top-level directory in mainline.

## Upstream references in scope

`io_uring/` (~85 .c files, all top-level — no subdirs). Mapping to Tier-3 docs:

| Category | Upstream paths | Planned Tier-3 doc |
|---|---|---|
| Core ring + lifecycle | `io_uring/io_uring.{c,h}`, `io_uring/refs.h`, `io_uring/slist.h` | `core.md` |
| Op dispatcher | `io_uring/opdef.{c,h}` | `op-dispatch.md` |
| File table + registered files | `io_uring/filetable.{c,h}`, `io_uring/rsrc.{c,h}` | `filetable-rsrc.md` |
| Submission queue polling thread | `io_uring/sqpoll.{c,h}`, `io_uring/io-wq.{c,h}` | `sqpoll-iowq.md` |
| File-system ops | `io_uring/fs.{c,h}`, `io_uring/openclose.{c,h}`, `io_uring/rw.{c,h}`, `io_uring/sync.{c,h}`, `io_uring/statx.{c,h}`, `io_uring/truncate.{c,h}`, `io_uring/xattr.{c,h}`, `io_uring/advise.{c,h}`, `io_uring/splice.{c,h}` | `fs-ops.md` |
| Network ops | `io_uring/net.{c,h}`, `io_uring/notif.{c,h}` (zerocopy notifications), `io_uring/cmd_net.c`, `io_uring/zcrx.{c,h}` (zero-copy receive) | `net-ops.md` |
| Polling | `io_uring/poll.{c,h}`, `io_uring/epoll.{c,h}` | `poll-ops.md` |
| Timeouts + cancel + waitid + wait | `io_uring/timeout.{c,h}`, `io_uring/cancel.{c,h}`, `io_uring/waitid.{c,h}`, `io_uring/wait.{c,h}` | `wait-cancel.md` |
| Futex ops | `io_uring/futex.{c,h}` | folded into `wait-cancel.md` |
| Eventfd + signal-fd-via-uring | `io_uring/eventfd.{c,h}` | folded into `wait-cancel.md` |
| Buffers (provided buffer rings + ring buffers) | `io_uring/kbuf.{c,h}`, `io_uring/memmap.{c,h}` | `buffers.md` |
| Per-task context | `io_uring/tctx.{c,h}`, `io_uring/tw.{c,h}` (task-work runner) | `task-context.md` |
| Registration + capabilities | `io_uring/register.{c,h}`, `io_uring/query.{c,h}` | `register.md` |
| FDInfo | `io_uring/fdinfo.{c,h}` | folded into `core.md` |
| Cross-system ops | `io_uring/uring_cmd.{c,h}` (URING_CMD generic dispatcher used by NVMe, ublk, etc.) | `uring-cmd.md` |
| Alloc cache + msg ring | `io_uring/alloc_cache.{c,h}`, `io_uring/msg_ring.{c,h}` | folded into `core.md` |
| Loop op + napi-poll integration | `io_uring/loop.{c,h}`, `io_uring/napi.{c,h}` | folded into `net-ops.md` |
| Nop op (for benchmarking + testing) | `io_uring/nop.{c,h}` | folded into `op-dispatch.md` |
| BPF integration | `io_uring/bpf-ops.{c,h}`, `io_uring/bpf_filter.{c,h}` | `bpf-integration.md` |
| Mock file (for testing) | `io_uring/mock_file.c` | (not separately documented) |

## Compatibility contract

### Syscall surface

`io_uring_setup`, `io_uring_enter`, `io_uring_register` — full ABI per `include/uapi/linux/io_uring.h`.

(Each gets a Tier-5 `uapi/syscalls/<name>.md` in Phase D.)

### Memory-mapped ring layout

The `io_uring_setup(2)` call returns a file descriptor that userspace maps with `mmap(2)` to obtain three regions:
- **Submission Queue Ring**: `IORING_OFF_SQ_RING`
- **Completion Queue Ring**: `IORING_OFF_CQ_RING`
- **Submission Queue Entries**: `IORING_OFF_SQES`

The struct layouts (`struct io_uring_sqe`, `struct io_uring_cqe`, `struct io_sqring_offsets`, `struct io_cqring_offsets`) are byte-identical to upstream — every field at the same offset.

### Operation set (`IORING_OP_*`)

Every `IORING_OP_*` value upstream supports must be implemented identically. The list is large (~70 ops at baseline) and growing.

### Registration commands (`IORING_REGISTER_*`)

All `IORING_REGISTER_*` commands (BUFFERS, FILES, EVENTFD, PROBE, PERSONALITY, FILES_UPDATE, BUFFERS2, IOWQ_AFF, IOWQ_MAX_WORKERS, RING_FDS, RING_FDS_SPARSE, MAP_HUGE, MAP_BUFFERS, MAP_BUFFERS2, RESTRICTIONS, ENABLE_RINGS, IORING_REGISTER_FILE_ALLOC_RANGE, ...) preserve numeric values and semantics.

### `/proc/<pid>/fdinfo/<fd>` for io_uring fds

Format-identical: lists ring sizes, head/tail pointers, registered files, registered buffers.

## Requirements

- REQ-1: `io_uring_setup`, `io_uring_enter`, `io_uring_register` are byte-identical with upstream — register/struct layouts/errno semantics.
- REQ-2: Memory-mapped ring layout (SQ ring, CQ ring, SQEs) is byte-identical: every offset in `io_sqring_offsets` / `io_cqring_offsets` matches upstream so existing liburing-userspace works unmodified.
- REQ-3: Every `IORING_OP_*` operation has a Rookery handler with byte-identical observable semantics.
- REQ-4: SQPOLL (kernel polling thread) preserves its CPU-affinity, idle-timeout, and shared-thread semantics.
- REQ-5: io-wq (offload worker pool) preserves its bounded/unbounded categorization and `IOWQ_MAX_WORKERS` registration semantics.
- REQ-6: Provided buffers (regular + ring) preserve their selection algorithms.
- REQ-7: `IORING_OP_URING_CMD` preserves the generic command-passthrough ABI (consumed by NVMe-passthrough, ublk, etc.).
- REQ-8: io_uring + cgroup integration: cgroup-tagged work runs in the right cgroup context.
- REQ-9: Existing liburing test suite (`liburing/test/`) passes against Rookery with the same pass/fail set as upstream.
- REQ-10: All Tier-3 docs declare their unsafe-block clusters, TLA+ models (the SQ/CQ ring concurrency contract is a strong TLA+ candidate), and Kani harnesses for ring-state invariants.

## Acceptance Criteria

- [ ] AC-1: liburing test suite under `tools/io_uring-test/` (or upstream liburing repo's test/) passes with the same set as upstream. (covers REQ-1, REQ-3, REQ-9)
- [ ] AC-2: A user-space program mmaps the rings on Rookery, computes offsets via `io_sqring_offsets`, writes SQEs at the offsets, and observes the kernel completing them. (covers REQ-2)
- [ ] AC-3: SQPOLL selftests pass: a thread shares an io_uring across multiple submitters; idle timeout fires correctly. (covers REQ-4)
- [ ] AC-4: io-wq selftests pass: bounded (e.g., openat) and unbounded (e.g., recv) work distribute correctly across workers. (covers REQ-5)
- [ ] AC-5: Provided-buffers selftest exercises both buffer modes (regular + ring) and asserts the right buffer is selected. (covers REQ-6)
- [ ] AC-6: NVMe-passthrough through `IORING_OP_URING_CMD` succeeds against a virtio-nvme device. (covers REQ-7)
- [ ] AC-7: A cgroup-confined io_uring submitter sees its work charged to the right memcg/blkcg. (covers REQ-8)
- [ ] AC-8: `make verify` passes io_uring/ Kani harnesses; `make tla` passes io_uring/ models. (covers REQ-10)

## Architecture

### Layout map

```
.design/io_uring/
  00-overview.md           ← this document
  core.md                  ← lifecycle + setup + enter + msg_ring + alloc_cache + fdinfo
  op-dispatch.md           ← opdef + nop op
  filetable-rsrc.md        ← registered files + resource update
  sqpoll-iowq.md           ← SQ polling thread + io-wq worker pool
  fs-ops.md                ← read/write/openat/openat2/close/fsync/sync_file_range/statx/truncate/xattr/advise/splice
  net-ops.md               ← send/recv/sendmsg/recvmsg/connect/accept/sockopts + zerocopy + napi-poll + cmd_net
  poll-ops.md              ← poll + epoll
  wait-cancel.md           ← timeout + cancel + waitid + wait + futex + eventfd
  buffers.md               ← provided buffers (regular + ring) + memmap
  task-context.md          ← per-task tctx + task-work runner
  register.md              ← registration commands + query + restrictions
  uring-cmd.md             ← URING_CMD generic dispatcher
  bpf-integration.md       ← bpf-ops + bpf_filter
```

### Cross-references

- `fs/00-overview.md` — every fs syscall has an io_uring analog; cross-reference per-op docs
- `net/00-overview.md` — net-ops consumes the socket layer
- `mm/00-overview.md` — registered buffers and the memmap setup interact with mm
- `kernel/00-overview.md` — task-work runner consumes the kernel/ task lifecycle
- `kernel/bpf/00-overview.md` — bpf-integration consumes BPF
- `00-glossary.md` — `io_uring`

### Rust module organization (informative)

- `kernel::io_uring::core` — ring lifecycle
- `kernel::io_uring::ops` — op dispatch
- `kernel::io_uring::sqpoll`, `kernel::io_uring::iowq`
- … (mirrors Tier-3 layout)

### Locking and concurrency

io_uring is heavily concurrent:
- SQ + CQ rings: lock-free with memory ordering between userspace and kernel
- Per-context `ctx->completion_lock` (spinlock) — protects the CQ
- Per-context `ctx->uring_lock` (mutex) — held during many submission paths
- Task-work runner: per-task linked list with atomic enqueue
- io-wq: per-pool spinlock + per-cpu state

### Error handling

io_uring uses standard errno for syscall returns; per-op errors land in the CQE's `res` field.

## Verification

### Layer 1: Kani SAFETY proofs
- SQE / CQE field reads (against the userspace-mapped ring)
- Buffer-ring entry validation
- File-table lookup with refcount

### Layer 2: TLA+ models
- `models/io_uring/sq_cq_ring.tla` — proves SQ/CQ ring's lock-free producer/consumer ordering: kernel reads SQE only after userspace publishes; userspace reads CQE only after kernel publishes; head/tail pointers never desync.
- `models/io_uring/iowq.tla` — proves io-wq worker dispatch correctness.
- `models/io_uring/task_work.tla` — proves task-work list integrity under concurrent enqueue + dequeue.

### Layer 3: Kani harnesses
- Ring head/tail invariants (tail - head ≤ ring size; head ≤ tail)
- Provided-buffer ring invariants

### Layer 4: opt-in
- SQ/CQ ring memory-ordering correctness via Verus — strong candidate (well-bounded scope, known-tricky property).

## Hardening

Placeholder per `00-overview.md` D6. io_uring's RESTRICTIONS feature provides registration-time op restrictions; preserved.

## Resolved Decisions

### D1 (2026-05-09): BPF integration IN v0
`io_uring/bpf-ops.c` + `bpf_filter.c` (BPF program attachment for filtering ops) is in v0 scope. Small addition consuming `kernel/bpf/` infrastructure. `bpf-integration.md` Tier 3 covers it.

## Open Questions

(none — all open questions for this subsystem document are resolved above)

## Out of Scope

- AIO (`fs/aio.c`) — owned by `fs/00-overview.md` § aio.md
- 32-bit-only paths
- Implementation code
