# Tier-3: drivers/android/{binder,binder_alloc}.c — Android Binder IPC core (/dev/binder + transactions + async-buf)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
upstream-paths:
  - drivers/android/binder.c
  - drivers/android/binder_alloc.c
  - drivers/android/binder_alloc.h
  - drivers/android/binder_internal.h
  - drivers/android/binder_netlink.c
  - drivers/android/binder_trace.h
  - include/uapi/linux/android/binder.h
-->

## Summary

Android Binder is the kernel IPC primitive that backs the Android Framework Service Manager + AIDL — every cross-process call between `system_server`, `mediaserver`, `surfaceflinger`, `hwservicemanager`, `vndservicemanager`, and HALs flows through a binder transaction. The driver exposes three character devices (`/dev/binder`, `/dev/hwbinder`, `/dev/vndbinder`) — or, on a binderfs-enabled system, an unlimited set of devices under `/dev/binderfs/` (cross-ref `binderfs.md`) — each with an independent `binder_context` carrying a context-manager (the AOSP `servicemanager`/`hwservicemanager`/`vndservicemanager` daemon), an `hlist_head` of `binder_proc`, and a per-context node id space.

This Tier-3 covers `binder.c` (~7170 lines: ioctl dispatch, transaction engine, ref/node mgmt, thread pool, freezer, death-notification, debugfs, netlink notifications) plus `binder_alloc.c` (~1410 lines: per-`binder_proc` async-buf VMA + per-allocation freelist + LRU shrinker integration).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct binder_proc` | per-open(/dev/binder) state — thread pool, nodes, refs, todo queue | `drivers::android::binder::Proc` |
| `struct binder_thread` | per-task in-flight transaction stack + return-status | `drivers::android::binder::Thread` |
| `struct binder_node` | exported object owned by a `Proc` (server-side reference target) | `drivers::android::binder::Node` |
| `struct binder_ref` | per-`Proc` handle into a remote `Node` (client-side proxy) | `drivers::android::binder::Ref` |
| `struct binder_transaction` | in-flight transaction record (work item on todo queues) | `drivers::android::binder::Transaction` |
| `struct binder_buffer` | per-transaction async-buf allocation in the receiver's mmap region | `drivers::android::binder_alloc::Buffer` |
| `struct binder_alloc` | per-`Proc` allocator — vm_area, freelist, allocated rbtree, lru | `drivers::android::binder_alloc::Allocator` |
| `binder_ioctl(file, cmd, arg)` | ioctl entry (`BINDER_WRITE_READ`, `BINDER_SET_CONTEXT_MGR_EXT`, `BINDER_FREEZE`, …) | `Proc::ioctl` |
| `binder_thread_write(proc, thread, ubuf, sz, *consumed)` | drain BC_* opcodes from userland (TRANSACTION/REPLY/INCREFS/ACQUIRE/RELEASE/…) | `Thread::process_bc` |
| `binder_thread_read(proc, thread, ubuf, sz, *consumed, non_block)` | block on todo queue + emit BR_* opcodes (TRANSACTION/REPLY/SPAWN_LOOPER/…) | `Thread::process_br` |
| `binder_transaction(proc, thread, tr, reply, extra_buf_size)` | core: lookup target, build `Transaction`, alloc buf, fix-up offsets, queue on receiver | `Proc::transaction` |
| `binder_translate_handle / _binder / _fd / _fd_array` | per-object translation between sender/receiver address spaces | `Transaction::translate_*` |
| `binder_alloc_new_buf(alloc, data_sz, off_sz, extra_sz, is_async, pid)` | allocate per-transaction buffer in receiver's mmap | `Allocator::new_buf` |
| `binder_alloc_free_buf(alloc, buf)` | release buffer + return pages to LRU freelist | `Allocator::free_buf` |
| `binder_install_buffer_pages(alloc, buf, sz)` | per-page demand install (`pin_user_pages` semantics inside kernel mmap) | `Allocator::install_pages` |
| `binder_mmap(file, vma)` | per-proc 4 MB mmap install; one-shot VMA per `Proc` | `Proc::mmap` |
| `binder_open(inode, file)` / `binder_release(inode, file)` | per-open lifecycle hooked to fd close | `Proc::open` / `Proc::release` |
| `binder_ioctl_set_ctx_mgr(...)` / `_ext` | promote the calling `Proc` to context-manager (handle 0) | `Context::set_manager` |
| `binder_ioctl_freeze(...)` / `_get_freezer_info(...)` | per-`Proc` IPC-freezer (used by frozen-app cgroup) | `Proc::freeze` |
| `binder_set_thread_priority(...)` / `binder_inherit_priority(...)` | per-transaction sched-policy inheritance | `Thread::priority_inherit` |
| `binder_fops` | `file_operations` for `/dev/{binder,hwbinder,vndbinder}` | `BinderFileOps` |

## Compatibility contract

REQ-1: ioctl ABI compatible — `BINDER_WRITE_READ`, `BINDER_SET_CONTEXT_MGR{,_EXT}`, `BINDER_VERSION` (= 8 on 64-bit), `BINDER_GET_NODE_INFO_FOR_REF`, `BINDER_FREEZE`, `BINDER_GET_FROZEN_INFO`, `BINDER_GET_EXTENDED_ERROR`, `BINDER_ENABLE_ONEWAY_SPAM_DETECTION`.

REQ-2: BC_/BR_ opcode set complete — BC_TRANSACTION/BC_REPLY (sync), BC_TRANSACTION_SG/BC_REPLY_SG (scatter-gather), BC_FREE_BUFFER, BC_INCREFS/BC_ACQUIRE/BC_RELEASE/BC_DECREFS, BC_REQUEST_DEATH_NOTIFICATION/BC_CLEAR_DEATH_NOTIFICATION/BC_DEAD_BINDER_DONE, BR_TRANSACTION/BR_REPLY/BR_DEAD_BINDER/BR_FROZEN_REPLY/BR_FAILED_REPLY_*.

REQ-3: Object translation — `flat_binder_object` (BINDER / WEAK_BINDER / HANDLE / WEAK_HANDLE), `binder_fd_object` (FD), `binder_fd_array_object` (FDA), `binder_buffer_object` (PTR with optional HAS_PARENT chain) — each translated, ref-counted, fd-installed in receiver as appropriate.

REQ-4: Async (oneway) transaction queued on `binder_node::async_todo` — receiver dequeues one at a time so that oneway calls from one client cannot starve another node. Async-buf bytes accounted against `proc->free_async_space` (=`buffer_size/2`).

REQ-5: Per-`Proc` mmap is a kernel-managed VMA (no userspace write); receiver reads transaction data via the returned `binder_transaction_data::data.ptr.buffer` cookie. Mmap is `PROT_READ`-only at user level; kernel side writes the translated payload.

REQ-6: Death-notification — when a `Node`'s owner `Proc` releases, a `BR_DEAD_BINDER` work item is queued on every `Ref` that registered for notifications; client must reply `BC_DEAD_BINDER_DONE` so the ref can be cleared.

REQ-7: Context-manager — exactly one `Proc` may hold handle 0 per context; gate via `binder_set_context_mgr_ext` with `BINDER_SET_CONTEXT_MGR_EXT`; LSM hook `security_binder_set_context_mgr` called.

REQ-8: Sched-policy + nice inheritance per transaction — caller's policy/priority (within `min_priority` of target node) propagated to the receiving thread; reset on reply.

REQ-9: pid-namespace + euid translation — `binder_transaction_data::sender_pid` reflects the target proc's pid-namespace view; sender_euid is the caller's euid (or `INVALID_UID` if filtered).

REQ-10: Per-context `binderfs` integration — each context lives under its own `binderfs` super-block in a user namespace (cross-ref `binderfs.md`); per-namespace device count gated.

REQ-11: KUnit interface — `EXPORT_SYMBOL_IF_KUNIT` exports allocator internals (`binder_alloc_new_buf`, `_free_buf`, `_mmap_handler`, `_deferred_release`, `_free_page`, `_vma_close`) for `drivers/android/tests/`.

REQ-12: Netlink notification surface — `binder_netlink.c` emits per-event multicast notifications consumed by `vendor_init` and DropBox; the genl family is `BINDER_GENL_NAME`.

## Acceptance Criteria

- [ ] AC-1: cuttlefish boot + AOSP CTS `BinderTest` + `BinderProxyTest` passes.
- [ ] AC-2: `servicemanager` registration via `BINDER_SET_CONTEXT_MGR_EXT` succeeds exactly once per context.
- [ ] AC-3: AIDL sync call (BC_TRANSACTION → BR_TRANSACTION → BC_REPLY → BR_REPLY) round-trip < 50 µs on the reference device.
- [ ] AC-4: oneway storm test does not exceed `free_async_space` budget; offending sender gets `BR_FAILED_REPLY` not OOM.
- [ ] AC-5: fd-passing test through BINDER_TYPE_FD/FDA correctly installs fd in receiver under receiver's `files_struct` with capability check.
- [ ] AC-6: death-notification fires within one scheduling tick of `BC_DEAD_BINDER_DONE` consumer process exit.
- [ ] AC-7: `BINDER_FREEZE` blocks new sync transactions to the frozen proc; pending replies return `BR_FROZEN_REPLY`.
- [ ] AC-8: KUnit `drivers/android/tests/` allocator tests pass under KASAN + KMSAN.
- [ ] AC-9: kselftest `tools/testing/selftests/filesystems/binderfs/` (parts touching `/dev/binder`) passes.

## Architecture

`Proc` lives at `drivers::android::binder::Proc`:

```
struct Proc {
  inner_lock: Spinlock,
  outer_lock: Spinlock,
  context: Arc<Context>,        // owning /dev/binder | /dev/hwbinder | /dev/vndbinder | binderfs node
  pid: Pid,                     // task_tgid(current) at open()
  cred: Arc<Cred>,              // captured at open() — *not* the caller's current cred
  nodes: RbTree<u64, Arc<Node>>,
  refs_by_desc: RbTree<u32, Arc<Ref>>,
  refs_by_node: RbTree<u64, Arc<Ref>>,
  threads: RbTree<Pid, Arc<Thread>>,
  waiting_threads: List<WaitingThread>,
  todo: VecDeque<Arc<Work>>,    // proc-level work queue
  delivered_death: List<Arc<Death>>,
  alloc: KBox<Allocator>,       // see below
  max_threads: AtomicU32,
  requested_threads: AtomicU32,
  default_priority: BinderPriority,
  is_frozen: AtomicBool,
  sync_recv: AtomicBool,
  async_recv: AtomicBool,
  oneway_spam_detection_enabled: AtomicBool,
}

struct Allocator {
  vma: Option<NonNull<VmAreaStruct>>,
  mm: Arc<MmStruct>,
  buffer: NonNull<u8>,          // user-space address of the mmap region
  buffer_size: usize,           // typically 1 MB or 4 MB
  free_buffers: RbTree<usize, Arc<Buffer>>,   // by size (best-fit)
  allocated_buffers: RbTree<usize, Arc<Buffer>>, // by user_data ptr
  free_async_space: AtomicUsize, // budget for is_async=true allocations
  pages: KBox<[Page; N]>,       // demand-installed via list_lru shrinker
  freelist: ListLruRef<Page>,   // LRU of unused pages for shrinker reclaim
  vma_addr: AtomicUsize,
  mapped: AtomicBool,
  pid: Pid,
}
```

Per-open lifecycle `Proc::open`:
1. `binder_get_thread(proc)` lazily creates `Thread` for `current->pid`.
2. `Allocator::init(proc)` registers shrinker entry; defers mmap until `Proc::mmap` is invoked.
3. Hash insert into `binder_procs` list (RCU read by debugfs).

Per-`Proc::mmap`:
1. Validate `vma->vm_flags`: `FORBIDDEN_MMAP_FLAGS == VM_WRITE` (write disallowed).
2. Compute `buffer_size = min(vma_size, SZ_4M)`.
3. Install `binder_vm_ops` (vma_close, fault → returns SIGBUS — receiver must not fault).
4. Reserve `pages` array + register on `binder_freelist` LRU.

Per-`Thread::process_bc(BC_TRANSACTION{,_SG})`:
1. Copy `binder_transaction_data{,_sg}` from user; security_binder_transaction(sender, target).
2. `Proc::transaction(target_handle, code, flags, data, offsets, extra_buf_size, is_reply=false)`.
3. Inside `transaction`:
   a. Resolve `target_node` (`refs_by_desc[target_handle]` or context-manager's node 0).
   b. `target_proc = target_node.owner.upgrade()`; if frozen + sync → `BR_FROZEN_REPLY`.
   c. `target_alloc.new_buf(data_size, off_size, extra, is_async, sender_pid)` — best-fit on `free_buffers`.
   d. Copy `data` + `offsets` arrays from sender into the new buffer (cross-address-space, validated against `binder_validate_object`).
   e. Walk `offsets` array, translate each object (`translate_handle/_binder/_fd/_fd_array/_buffer_object`); per-fd: `__receive_fd` into receiver's `files_struct` with `security_binder_transfer_file`.
   f. Build `Transaction` work item; for oneway, queue on `target_node.async_todo`; for sync, queue on `target_proc.todo` or specific waiting `Thread::todo`.
4. Sender returns to user with `BR_TRANSACTION_COMPLETE` queued on sender thread.

Per-`Thread::process_br`:
1. Block on `proc.wait` / `thread.wait` until todo non-empty (subject to non-block flag).
2. Pop top work item → emit corresponding BR_* + transaction-data with buffer ptr in user mmap.
3. On TRANSACTION delivery, push `Transaction` onto `Thread::transaction_stack`; reply will pop it.

Per-async-buf accounting `Allocator::new_buf(is_async=true)`:
1. Check `free_async_space >= size + sizeof(struct binder_buffer)`. If not → `-ENOSPC`; `debug_low_async_space_locked` may set `oneway_spam_detection`.
2. Subtract from `free_async_space`; on `free_buf`, add back.

LRU shrinker `binder_shrink_count` / `_scan`:
- `binder_freelist` (`list_lru`) tracks pages unmapped from active buffers. Under memory pressure, shrinker free-page-walks and unmaps via `binder_free_page` (`free_unref_page`).

## Hardening

(Inherits row-1 features from `drivers/00-overview.md` § Hardening.)

binder-specific reinforcement:

- **Per-context-manager singleton** — exactly one Proc holds handle 0; `set_ctx_mgr` is non-rebindable absent context-manager death; defense against rogue daemon hijack.
- **`security_binder_*` LSM hooks on every privileged path** — `set_context_mgr`, `transaction`, `transfer_binder`, `transfer_file`; SELinux/AppArmor policy mandatory in Rookery.
- **`FORBIDDEN_MMAP_FLAGS == VM_WRITE`** — user mmap is read-only; defense against forged transaction-data injection.
- **`copy_from_user` bounded by `binder_validate_object`** — each translated object (offset, ptr_size, type) validated against buffer bounds; defense against integer overflow in offset arithmetic.
- **fd-passing capability check** — `security_binder_transfer_file` + `CAP_SYS_PTRACE`-equivalent boundary; cross-uid fd-transfer audited.
- **oneway queue size cap** — `free_async_space = buffer_size/2`; defense against oneway-flood DoS on async-todo.
- **`ENABLE_OOM_REAPER` allowlist** — binder-receiver pages excluded from OOM-reaper to avoid double-free with in-flight transaction.
- **`binder_thread` `kfree_sensitive`** — thread struct holds caller-pid/cred; zeroed on free.
- **Death-notification one-shot** — `BR_DEAD_BINDER` requires `BC_DEAD_BINDER_DONE` ack; ref refcount underflow trapped via PAX_REFCOUNT.

## Grsecurity/PaX-style Reinforcement

- **PAX_USERCOPY** — whitelisted slab caches for `binder_proc`, `binder_thread`, `binder_node`, `binder_ref`, `binder_transaction`, and `binder_buffer`; `binder_transaction_data{,_sg,_secctx}` copies bounded by `sizeof(struct)` with `check_object_size`.
- **PAX_KERNEXEC** — binder ioctl/mmap/release entries in W^X kernel text; `binder_fops` and `binder_vm_ops` live in `__ro_after_init` text.
- **PAX_RANDKSTACK** — randomize kernel-stack offset across `binder_ioctl`, `binder_thread_write`, `binder_thread_read`, and `binder_transaction` entries — every Android IPC syscall benefits.
- **PAX_REFCOUNT** — saturating `refcount_t` on `binder_proc`, `binder_node`, `binder_ref`, `binder_transaction`, and `binder_buffer`; overflow trap defeats ref-leak UAF and the historical CVE-2019-2215 class.
- **PAX_MEMORY_SANITIZE** — zero-on-free for `binder_thread` (caller pid/cred), `binder_transaction` (sender_euid/secctx), async-buf pages on `binder_alloc_free_buf`, and the `binder_buffer` slab.
- **PAX_UDEREF** — SMAP/PAN enforced on every `binder_ioctl`, `binder_thread_write`, `binder_thread_read`, and `binder_translate_*` user-pointer deref; reject mis-sized `BINDER_WRITE_READ` payloads.
- **PAX_RAP / kCFI** — `binder_fops`, `binder_vm_ops`, `binder_alloc_ops`, and the BR_/BC_ opcode dispatch tables marked `__ro_after_init` with kCFI-typed indirect dispatch.
- **GRKERNSEC_HIDESYM** — gate kallsyms exposure of `binder_proc`/`binder_node` pointers in `/sys/kernel/debug/binder/` behind CAP_SYSLOG; suppress `%p` in transaction-log seq files.
- **GRKERNSEC_DMESG** — restrict `binder: %d:%d` user-error banners and oneway-spam warnings to CAP_SYSLOG so attackers cannot probe IPC topology via dmesg.
- **fd-passing CAP_SYS_PTRACE-equivalent gating** — `security_binder_transfer_file` cross-uid transfers require an explicit allow-listed binder context; default-deny.
- **oneway-queue hard cap** — `free_async_space = buffer_size/2`; spam-detection forces `BR_FAILED_REPLY` and an audit record on offending sender.
- **`ENABLE_OOM_REAPER` allowlist** — binder-receiver mmap pages excluded from OOM-reaper while a transaction holds the buffer.
- **`binder_thread` / `binder_transaction` `kfree_sensitive`** — caller cred + secctx zeroed on free; defeats use-after-free secctx leak.

Rationale: Binder is the largest user-reachable kernel-attack surface in any Android device — historically the source of CVE-2019-2215, CVE-2020-0041, CVE-2022-20421 and many more. RAP/kCFI on `binder_fops`/`binder_vm_ops`, PAX_REFCOUNT on every ref/node/transaction, SMAP/PAN on every user-pointer deref, mandatory SELinux LSM gating, oneway-queue caps, and zero-on-free of cred/secctx convert Binder from "fast IPC with a security hook" into a structurally hardened IPC boundary.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- binderfs control-device + dynamic-device allocation (covered in `binderfs.md`)
- KUnit test crate (covered in `binder-tests.md` future Tier-3)
- Vendor-specific binder extensions (out of upstream scope)
- 32-bit binder (`BINDER_IPC_32BIT`) — Rookery is 64-bit only
- Implementation code
