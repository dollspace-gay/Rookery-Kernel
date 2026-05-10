# Tier-3: mm/userfaultfd.c + fs/userfaultfd.c — userfaultfd userspace page-fault handling

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/userfaultfd.c (~2241 lines)
  - fs/userfaultfd.c (~2231 lines)
  - include/linux/userfaultfd_k.h
  - include/uapi/linux/userfaultfd.h
-->

## Summary

userfaultfd (UFFD) lets userspace handle page-faults on registered VMA ranges. `userfaultfd(2)` returns an fd. `UFFDIO_REGISTER` ioctl marks per-VMA range with mode {MISSING, WP (write-protect), MINOR}. Per-fault on registered range: kernel blocks the faulting task on `userfaultfd_ctx.fault_wqh`; pushes `uffd_msg` (fault address + thread + flags) to fd-readable queue. Userspace reads msg + responds via `UFFDIO_COPY` (zero-copy install), `UFFDIO_ZEROPAGE`, `UFFDIO_CONTINUE` (for MINOR mode), or `UFFDIO_WRITEPROTECT`. Critical for: live migration (post-copy), userland CRIU/checkpoint, language-runtime guard pages, hardware-assisted compaction.

This Tier-3 covers `mm/userfaultfd.c` (~2241 lines) + `fs/userfaultfd.c` (~2231 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct userfaultfd_ctx` | per-fd state | `UserfaultfdCtx` |
| `userfaultfd_create()` | userfaultfd(2) syscall | `Uffd::create` |
| `userfaultfd_ctx_alloc()` | per-ctx alloc | `Uffd::ctx_alloc` |
| `userfaultfd_register()` | UFFDIO_REGISTER ioctl | `Uffd::register` |
| `userfaultfd_register_range()` | per-VMA register | `Uffd::register_range` |
| `userfaultfd_unregister()` | UFFDIO_UNREGISTER | `Uffd::unregister` |
| `handle_userfault()` | per-fault wait+signal | `Uffd::handle_userfault` |
| `mfill_atomic_copy()` | UFFDIO_COPY | `Uffd::mfill_atomic_copy` |
| `mfill_atomic_zeropage()` | UFFDIO_ZEROPAGE | `Uffd::mfill_atomic_zeropage` |
| `mfill_atomic_continue()` | UFFDIO_CONTINUE | `Uffd::mfill_atomic_continue` |
| `mwriteprotect_range()` | UFFDIO_WRITEPROTECT | `Uffd::mwriteprotect_range` |
| `userfaultfd_msg_init()` | per-msg builder | `Uffd::msg_init` |
| `userfaultfd_event_wait_completion()` | per-event remap/remove | `Uffd::event_wait` |
| `UFFD_FEATURE_*` | per-feature bits | UAPI |
| `UFFDIO_REGISTER_MODE_MISSING` / `_WP` / `_MINOR` | per-register mode | UAPI |

## Compatibility contract

REQ-1: userfaultfd(2) syscall:
- flags: O_CLOEXEC | O_NONBLOCK | UFFD_USER_MODE_ONLY.
- Returns fd backed by anon-inode with `userfaultfd_fops`.
- ctx initialized; mm reference taken.

REQ-2: UFFDIO_API ioctl:
- userspace passes desired feature bits.
- kernel echoes back supported subset.
- ctx.features = (user_features & supported_features) | UFFD_FEATURE_INITIALIZED.

REQ-3: UFFD_FEATURE_*:
- PAGEFAULT_FLAG_WP: write-protect mode.
- EVENT_FORK / REMAP / REMOVE / UNMAP.
- MISSING_HUGETLBFS / SHMEM.
- SIGBUS: SIGBUS instead of fault-handler-block.
- THREAD_ID: include faulting tid in msg.
- MINOR_HUGETLBFS / MINOR_SHMEM: minor-fault mode.
- WP_ASYNC: async write-protect.
- WP_UNPOPULATED: WP without alloc.
- EXACT_ADDRESS: exact-vs-page-aligned msg.address.

REQ-4: UFFDIO_REGISTER:
- struct uffdio_register: { range: {start, len}, mode: {MISSING|WP|MINOR}, ioctls: [out] }.
- Per-VMA in range: VMA.vm_userfaultfd_ctx = ctx; VMA flags |= VM_UFFD_*.
- Returns supported ioctl-bitmap.

REQ-5: handle_userfault flow (per-page-fault):
- vma = find_vma(mm, addr).
- if !(vma.vm_flags & VM_UFFD_*): regular fault path.
- ctx = vma.vm_userfaultfd_ctx.ctx.
- Build uffd_msg: pagefault.address = addr; pagefault.feat = current.tid (if THREAD_ID).
- Add to ctx.fault_pending_wqh.
- If ctx.features & SIGBUS: send SIGBUS to faulting task.
- Else: schedule out task; wait on ctx.fault_wqh.

REQ-6: read(uffd) returns uffd_msg:
- Pops earliest msg from ctx.event_wqh / ctx.fault_pending_wqh.
- If !has_msg ∧ !O_NONBLOCK: block.

REQ-7: UFFDIO_COPY:
- Per-(dst, src, len, mode):
  - mfill_atomic_copy: copy src→guest-mem at dst; install PTE; mark VMA-range-uffd-resolved.
  - Wake faulting task.

REQ-8: UFFDIO_ZEROPAGE:
- Per-dst-range: install PTE pointing to per-zero-page (no copy).

REQ-9: UFFDIO_CONTINUE (MINOR mode):
- Per-already-allocated-page: just install PTE without alloc.

REQ-10: UFFDIO_WRITEPROTECT:
- Per-range mark/unmark write-protected.
- Subsequent guest writes will MINOR-fault.

REQ-11: Per-event:
- EVENT_FORK: per-fork-of-uffd-registered-task, child gets new uffd-ctx.
- EVENT_REMAP: per-mremap; userspace notified.
- EVENT_REMOVE: per-madvise(REMOVE); userspace notified.
- EVENT_UNMAP: per-munmap; userspace notified.

REQ-12: ctx lifetime:
- Per-fd refcount.
- Per-VMA refs ctx; ctx releases on VMA-final-unref.

## Acceptance Criteria

- [ ] AC-1: userfaultfd(O_CLOEXEC): returns fd; ctx allocated.
- [ ] AC-2: UFFDIO_API: feature-bits negotiated.
- [ ] AC-3: UFFDIO_REGISTER mode=MISSING: VMA marked.
- [ ] AC-4: Page-fault on registered range: handle_userfault called; task blocks.
- [ ] AC-5: read(uffd): returns uffd_msg with addr.
- [ ] AC-6: UFFDIO_COPY src→dst: PTE installed; faulting task resumes.
- [ ] AC-7: UFFDIO_ZEROPAGE: zero-page installed.
- [ ] AC-8: UFFDIO_WRITEPROTECT then guest write: MINOR-fault delivered.
- [ ] AC-9: UFFD_FEATURE_SIGBUS: SIGBUS instead of block.
- [ ] AC-10: UFFDIO_UNREGISTER: VMA flags cleared; ctx released.
- [ ] AC-11: Per-fork with EVENT_FORK: child uffd-ctx created.
- [ ] AC-12: Userspace-CRIU live-migration uses post-copy via UFFD.

## Architecture

Per-fd state:

```
struct UserfaultfdCtx {
  fault_pending_wqh: WaitQueueHead<&UffdMsg>,
  fault_wqh: WaitQueueHead<&FaultingTask>,
  fd_wqh: WaitQueueHead<&Reader>,
  event_wqh: WaitQueueHead<&UffdEventMsg>,
  refcount: AtomicI64,
  flags: u32,
  features: u64,                                  // UFFD_FEATURE_*
  state: UffdState,
  released: bool,
  mmap_changing: AtomicBool,
  mm: *MmStruct,
}

enum UffdState { Init, Running, Released }
```

Per-VMA:

```
struct VmAreaStruct {
  ...
  vm_userfaultfd_ctx: VmUffdCtx,
  vm_flags: { VM_UFFD_MISSING, VM_UFFD_WP, VM_UFFD_MINOR },
}

struct VmUffdCtx {
  ctx: Option<*UserfaultfdCtx>,
}
```

Per-msg (uffd_msg):

```
#[repr(C)]
struct UffdMsg {
  event: u8,                                     // UFFD_EVENT_*
  reserved1: u8,
  reserved2: u16,
  reserved3: u32,
  arg: union {
    pagefault: { flags: u64, address: u64, feat: { ptid: u32 } },
    fork: { ufd: u32 },
    remap: { from: u64, to: u64, len: u64 },
    remove: { start: u64, end: u64 },
  },
}
```

`Uffd::create(flags) -> Result<i32>`:
1. Validate flags: O_CLOEXEC | O_NONBLOCK | UFFD_USER_MODE_ONLY.
2. ctx = userfaultfd_ctx_alloc().
3. ctx.mm = current.mm; mmgrab(ctx.mm).
4. fd = anon_inode_getfd("[userfaultfd]", &userfaultfd_fops, ctx, ...).
5. Return fd.

`Uffd::register(ctx, args) -> Result<()>`:
1. Validate args.range; mode must include MISSING | WP | MINOR.
2. mmap_write_lock(ctx.mm).
3. for each VMA in [start, end):
   - validate vma.vm_flags compatible with mode.
   - vma.vm_userfaultfd_ctx.ctx = ctx; vma.vm_flags |= VM_UFFD_*.
4. mmap_write_unlock.
5. Return supported ioctl-bitmap.

`Uffd::handle_userfault(vmf, reason)`:
1. vma = vmf.vma; ctx = vma.vm_userfaultfd_ctx.ctx.
2. msg = Uffd::msg_init(addr=vmf.address, reason=reason, features=ctx.features, faulting_tid=current.tid).
3. If ctx.features & SIGBUS: force_sig(SIGBUS); return VM_FAULT_SIGBUS.
4. spin_lock(&ctx.fault_pending_wqh.lock).
5. add_wait_queue(&ctx.fault_pending_wqh, &msg).
6. spin_unlock.
7. wake_up_poll(&ctx.fd_wqh, EPOLLIN).
8. /* Block faulting task */
9. wait_on(&ctx.fault_wqh, until response).
10. Return VM_FAULT_RETRY (let arch retry).

`Uffd::mfill_atomic_copy(ctx, dst, src, len, mode) -> Result<usize>`:
1. mmap_read_lock(ctx.mm).
2. err = mfill_atomic(ctx.mm, dst, src, len, MFILL_ATOMIC_COPY).
3. mmap_read_unlock.
4. /* Wake fault-blocked tasks at dst-range */
5. wake_up_pool(&ctx.fault_wqh).
6. Return copied bytes.

`Uffd::mfill_atomic_zeropage(ctx, dst, len) -> Result<usize>`:
1. mmap_read_lock; mfill_atomic ZEROPAGE; unlock; wake; return.

`Uffd::mfill_atomic_continue(ctx, dst, len) -> Result<usize>`:
1. mmap_read_lock; mfill_atomic CONTINUE; unlock; wake; return.

`Uffd::mwriteprotect_range(ctx, start, len, wp) -> Result<()>`:
1. mmap_write_lock.
2. For each PTE in [start, start+len):
   - if wp: pte = pte_wrprotect(pte) | _PAGE_UFFD_WP.
   - else: pte = pte_mkwrite(pte) & ~_PAGE_UFFD_WP.
3. mmap_write_unlock.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `ctx_features_within_supported` | INVARIANT | per-API: ctx.features ⊆ supported_features. |
| `vma_uffd_only_with_ctx` | INVARIANT | vma.vm_flags & VM_UFFD_* ⟺ vma.vm_userfaultfd_ctx.ctx != NULL. |
| `register_under_mmap_write_lock` | INVARIANT | per-register: mmap_write_lock held. |
| `mfill_atomic_under_mmap_read_lock` | INVARIANT | per-mfill: mmap_read_lock held. |
| `released_no_new_register` | INVARIANT | post-released: no new register accepted. |

### Layer 2: TLA+

`mm/userfaultfd.tla`:
- Per-ctx lifecycle + per-fault wait/wake + per-mfill resolve.
- Properties:
  - `safety_no_orphan_blocked_task` — per-blocked task eventually woken or SIGBUS.
  - `safety_per_fault_msg_delivered_once` — per-fault: msg in queue at most once.
  - `liveness_pending_eventually_consumed` — per-msg + reader ⟹ eventually read.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Uffd::register` post: per-VMA flags + ctx set; ioctls returned | `Uffd::register` |
| `Uffd::handle_userfault` post: msg in fault_pending_wqh; task wait OR SIGBUS | `Uffd::handle_userfault` |
| `Uffd::mfill_atomic_copy` post: per-page PTE installed; faulters woken | `Uffd::mfill_atomic_copy` |
| `Uffd::mwriteprotect_range` post: per-PTE _PAGE_UFFD_WP toggled per `wp` | `Uffd::mwriteprotect_range` |

### Layer 4: Verus/Creusot functional

`Per-fault on UFFD-registered range → userspace receives msg → resolves via UFFDIO_COPY/ZEROPAGE/CONTINUE → faulting task resumes` semantic equivalence: per-userfaultfd(2) man page.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

UFFD-specific reinforcement:

- **UFFD_USER_MODE_ONLY mandatory in v0** — defense against per-userspace fd accessing kernel-mode ranges.
- **Per-VMA flag locked under mmap_write_lock** — defense against per-register race.
- **Per-mfill_atomic under mmap_read_lock** — defense against per-mfill VMA-mutation race.
- **Per-fault SIGBUS option** — defense against per-blocked task on uncooperative userspace.
- **Per-ctx refcount via per-VMA + per-fd** — defense against UAF on close.
- **Per-EVENT_FORK proper child-ctx** — defense against per-fork shared-ctx confusion.
- **Per-EVENT_REMAP/UNMAP/REMOVE notify** — defense against per-userspace stale-mapping ignorance.
- **Per-mode validated against VMA type** — defense against per-MINOR on non-shmem/hugetlbfs.
- **Per-WP_ASYNC opt-in** — defense against per-config sync WP causing live-migration stall.
- **Per-msg pagefault.address aligned per-EXACT_ADDRESS feature** — defense against per-feature ABI confusion.
- **Per-fault retry-fault model** — defense against per-fault sync-block in arch handler.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/00-overview (Tier-2)
- VMA core (covered in `virtual-memory.md` Tier-3)
- mmap (covered in `mmap.md` Tier-3)
- Live migration (out-of-tree consumer)
- CRIU (out-of-tree consumer)
- Implementation code
