# Tier-5 UAPI: include/uapi/linux/userfaultfd.h — userfaultfd(2) ABI

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - include/uapi/linux/userfaultfd.h (~386 lines)
  - fs/userfaultfd.c
  - Documentation/admin-guide/mm/userfaultfd.rst
-->

## Summary

`userfaultfd(2)` returns a file descriptor that delivers page-fault and VMA-lifecycle events for ranges of the calling process's address space, enabling user-space page-fault handlers (live migration, CRIU, hot-patching, sandboxed managed runtimes). The fd answers `UFFDIO_API` (handshake), `UFFDIO_REGISTER` / `UFFDIO_UNREGISTER` (per-range registration in MISSING / WP / MINOR modes), and per-range fix-up ioctls `UFFDIO_COPY` / `UFFDIO_ZEROPAGE` / `UFFDIO_CONTINUE` / `UFFDIO_POISON` / `UFFDIO_MOVE` / `UFFDIO_WRITEPROTECT` / `UFFDIO_WAKE`. Reading the fd yields a stream of `struct uffd_msg` events: `UFFD_EVENT_PAGEFAULT` / `_FORK` / `_REMAP` / `_REMOVE` / `_UNMAP`. Per-`UFFD_USER_MODE_ONLY` restricts handling to user-mode faults (refused for kernel-mode faults, preventing privileged-page-fault gadgets). Per-`/dev/userfaultfd` (ioctl `USERFAULTFD_IOC_NEW`) is the privileged path when sysctl `vm.unprivileged_userfaultfd=0`. Critical for: managed-memory runtimes, post-copy live migration, CRIU restore, write-protect-based GC, hugetlbfs/shmem MINOR-fault handling.

This Tier-5 covers `include/uapi/linux/userfaultfd.h` (~386 lines).

## ABI surface

| Constant / Type | Value | Purpose |
|---|---|---|
| `USERFAULTFD_IOC` | `0xAA` | ioctl group for `/dev/userfaultfd` |
| `USERFAULTFD_IOC_NEW` | `_IO(0xAA, 0x00)` | allocate uffd fd via chardev |
| `UFFD_API` | `0xAA` (u64) | API version handshake constant |
| `UFFDIO` | `0xAA` | ioctl group for uffd fd |
| `_UFFDIO_REGISTER` | `0x00` | register range |
| `_UFFDIO_UNREGISTER` | `0x01` | unregister range |
| `_UFFDIO_WAKE` | `0x02` | wake faulting threads |
| `_UFFDIO_COPY` | `0x03` | resolve missing fault with user data |
| `_UFFDIO_ZEROPAGE` | `0x04` | resolve missing fault with zero page |
| `_UFFDIO_MOVE` | `0x05` | atomic anon-page move into uffd range |
| `_UFFDIO_WRITEPROTECT` | `0x06` | set/clear WP on range |
| `_UFFDIO_CONTINUE` | `0x07` | resolve MINOR fault (existing page) |
| `_UFFDIO_POISON` | `0x08` | install HWPOISON marker pte |
| `_UFFDIO_API` | `0x3F` | handshake ioctl number |
| `UFFDIO_API` | `_IOWR(0xAA, 0x3F, uffdio_api)` | handshake |
| `UFFDIO_REGISTER` | `_IOWR(0xAA, 0x00, uffdio_register)` | register range |
| `UFFDIO_UNREGISTER` | `_IOR(0xAA, 0x01, uffdio_range)` | unregister range |
| `UFFDIO_WAKE` | `_IOR(0xAA, 0x02, uffdio_range)` | wake range |
| `UFFDIO_COPY` | `_IOWR(0xAA, 0x03, uffdio_copy)` | copy resolve |
| `UFFDIO_ZEROPAGE` | `_IOWR(0xAA, 0x04, uffdio_zeropage)` | zero resolve |
| `UFFDIO_MOVE` | `_IOWR(0xAA, 0x05, uffdio_move)` | move resolve |
| `UFFDIO_WRITEPROTECT` | `_IOWR(0xAA, 0x06, uffdio_writeprotect)` | WP toggle |
| `UFFDIO_CONTINUE` | `_IOWR(0xAA, 0x07, uffdio_continue)` | MINOR resolve |
| `UFFDIO_POISON` | `_IOWR(0xAA, 0x08, uffdio_poison)` | poison install |
| `UFFDIO_REGISTER_MODE_MISSING` | `1<<0` | trap missing-page faults |
| `UFFDIO_REGISTER_MODE_WP` | `1<<1` | trap write-protect faults |
| `UFFDIO_REGISTER_MODE_MINOR` | `1<<2` | trap minor (already-present) faults |
| `UFFDIO_COPY_MODE_DONTWAKE` | `1<<0` | suppress waiter wake-up |
| `UFFDIO_COPY_MODE_WP` | `1<<1` | install pte as write-protected |
| `UFFDIO_ZEROPAGE_MODE_DONTWAKE` | `1<<0` | suppress wake |
| `UFFDIO_WRITEPROTECT_MODE_WP` | `1<<0` | set (1) / clear (0) WP |
| `UFFDIO_WRITEPROTECT_MODE_DONTWAKE` | `1<<1` | suppress wake on WP=0 |
| `UFFDIO_CONTINUE_MODE_DONTWAKE` | `1<<0` | suppress wake |
| `UFFDIO_CONTINUE_MODE_WP` | `1<<1` | install WP pte |
| `UFFDIO_POISON_MODE_DONTWAKE` | `1<<0` | suppress wake |
| `UFFDIO_MOVE_MODE_DONTWAKE` | `1<<0` | suppress wake on dst |
| `UFFDIO_MOVE_MODE_ALLOW_SRC_HOLES` | `1<<1` | tolerate unmapped src ptes |
| `UFFD_EVENT_PAGEFAULT` | `0x12` | uffd_msg event: page fault |
| `UFFD_EVENT_FORK` | `0x13` | uffd_msg event: fork inherited uffd |
| `UFFD_EVENT_REMAP` | `0x14` | uffd_msg event: mremap |
| `UFFD_EVENT_REMOVE` | `0x15` | uffd_msg event: madvise REMOVE/DONTNEED |
| `UFFD_EVENT_UNMAP` | `0x16` | uffd_msg event: munmap |
| `UFFD_PAGEFAULT_FLAG_WRITE` | `1<<0` | fault was a write |
| `UFFD_PAGEFAULT_FLAG_WP` | `1<<1` | fault was a WP violation |
| `UFFD_PAGEFAULT_FLAG_MINOR` | `1<<2` | fault was a MINOR (page-present) fault |
| `UFFD_FEATURE_PAGEFAULT_FLAG_WP` | `1<<0` | userspace handles WP faults |
| `UFFD_FEATURE_EVENT_FORK` | `1<<1` | deliver FORK events |
| `UFFD_FEATURE_EVENT_REMAP` | `1<<2` | deliver REMAP events |
| `UFFD_FEATURE_EVENT_REMOVE` | `1<<3` | deliver REMOVE events |
| `UFFD_FEATURE_MISSING_HUGETLBFS` | `1<<4` | MISSING on hugetlbfs |
| `UFFD_FEATURE_MISSING_SHMEM` | `1<<5` | MISSING on shmem/tmpfs |
| `UFFD_FEATURE_EVENT_UNMAP` | `1<<6` | deliver UNMAP events |
| `UFFD_FEATURE_SIGBUS` | `1<<7` | deliver SIGBUS instead of fault |
| `UFFD_FEATURE_THREAD_ID` | `1<<8` | report faulting tid in pagefault.feat.ptid |
| `UFFD_FEATURE_MINOR_HUGETLBFS` | `1<<9` | MINOR on hugetlbfs |
| `UFFD_FEATURE_MINOR_SHMEM` | `1<<10` | MINOR on shmem |
| `UFFD_FEATURE_EXACT_ADDRESS` | `1<<11` | unmasked fault address |
| `UFFD_FEATURE_WP_HUGETLBFS_SHMEM` | `1<<12` | WP on hugetlbfs/shmem |
| `UFFD_FEATURE_WP_UNPOPULATED` | `1<<13` | WP applies to empty ptes (anon) |
| `UFFD_FEATURE_POISON` | `1<<14` | UFFDIO_POISON available |
| `UFFD_FEATURE_WP_ASYNC` | `1<<15` | async WP-fault auto-resolve |
| `UFFD_FEATURE_MOVE` | `1<<16` | UFFDIO_MOVE available |
| `UFFD_USER_MODE_ONLY` | `1` | userfaultfd(2) flag: only user-mode faults |
| `struct uffdio_api` | — | `{api, features, ioctls}` |
| `struct uffdio_range` | — | `{start, len}` |
| `struct uffdio_register` | — | `{range, mode, ioctls}` |
| `struct uffdio_copy` | — | `{dst, src, len, mode, copy}` |
| `struct uffdio_zeropage` | — | `{range, mode, zeropage}` |
| `struct uffdio_writeprotect` | — | `{range, mode}` |
| `struct uffdio_continue` | — | `{range, mode, mapped}` |
| `struct uffdio_poison` | — | `{range, mode, updated}` |
| `struct uffdio_move` | — | `{dst, src, len, mode, move}` |
| `struct uffd_msg` | packed | event header + per-event union |

`struct uffd_msg` (packed):
- `__u8 event;` — `UFFD_EVENT_*`.
- `__u8 reserved1; __u16 reserved2; __u32 reserved3;` — padding, must be zero on read.
- `union arg`:
  - `pagefault { __u64 flags; __u64 address; union feat { __u32 ptid; }; }`.
  - `fork { __u32 ufd; }`.
  - `remap { __u64 from; __u64 to; __u64 len; }`.
  - `remove { __u64 start; __u64 end; }`.
  - `reserved { __u64 reserved1, reserved2, reserved3; }`.

API-set bitmasks:
- `UFFD_API_REGISTER_MODES` = MISSING | WP | MINOR.
- `UFFD_API_FEATURES` = OR of all `UFFD_FEATURE_*` bits the kernel knows about.
- `UFFD_API_IOCTLS` = bits for `_UFFDIO_REGISTER | _UFFDIO_UNREGISTER | _UFFDIO_API`.
- `UFFD_API_RANGE_IOCTLS` = bits for WAKE | COPY | ZEROPAGE | MOVE | WRITEPROTECT | CONTINUE | POISON.
- `UFFD_API_RANGE_IOCTLS_BASIC` = WAKE | COPY | WRITEPROTECT | CONTINUE | POISON (hugetlbfs subset; no ZEROPAGE/MOVE).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `userfaultfd(2)` syscall | per-fd allocation | `Uffd::syscall_create` |
| `USERFAULTFD_IOC_NEW` | per-/dev/userfaultfd path | `Uffd::chardev_new` |
| `UFFDIO_API` | per-handshake | `Uffd::ioctl_api` |
| `UFFDIO_REGISTER` | per-range register | `Uffd::ioctl_register` |
| `UFFDIO_UNREGISTER` | per-range unregister | `Uffd::ioctl_unregister` |
| `UFFDIO_WAKE` | per-range wake | `Uffd::ioctl_wake` |
| `UFFDIO_COPY` | per-MISSING copy | `Uffd::ioctl_copy` |
| `UFFDIO_ZEROPAGE` | per-MISSING zero | `Uffd::ioctl_zeropage` |
| `UFFDIO_CONTINUE` | per-MINOR continue | `Uffd::ioctl_continue` |
| `UFFDIO_POISON` | per-poison pte | `Uffd::ioctl_poison` |
| `UFFDIO_MOVE` | per-anon move | `Uffd::ioctl_move` |
| `UFFDIO_WRITEPROTECT` | per-WP toggle | `Uffd::ioctl_writeprotect` |
| `struct uffd_msg` | per-event read frame | `UffdMsg` |
| `UFFD_USER_MODE_ONLY` | per-create flag | `UffdCreateFlags::USER_MODE_ONLY` |

## Compatibility contract

REQ-1: `userfaultfd(int flags)` — flags from `{ O_CLOEXEC, O_NONBLOCK, UFFD_USER_MODE_ONLY }`. Returns non-negative fd or `-1/errno`. EINVAL on unknown flags. EPERM if unprivileged and `vm.unprivileged_userfaultfd=0` (alt-path: `open("/dev/userfaultfd")` + `USERFAULTFD_IOC_NEW`).

REQ-2: `UFFDIO_API` MUST be the first ioctl. Userland writes `{api=UFFD_API, features=requested}`; kernel writes back `{api=UFFD_API, features=available_subset, ioctls=UFFD_API_IOCTLS}`. EINVAL if api != UFFD_API or requested-features ⊄ supported.

REQ-3: `UFFDIO_REGISTER` — `mode` is OR of `UFFDIO_REGISTER_MODE_{MISSING,WP,MINOR}`. Range MUST be page-aligned and within a single VMA-class set; kernel writes back `ioctls` bitmask of permitted range-ioctls (`UFFD_API_RANGE_IOCTLS` or `_BASIC` subset for hugetlbfs).

REQ-4: `UFFDIO_UNREGISTER` — `uffdio_range` only; range MUST match a previously-registered span.

REQ-5: `UFFDIO_COPY` — atomically install `len` bytes from `src` into `[dst, dst+len)`. `copy` field returns bytes copied or `-errno`. `MODE_WP` installs WP pte. `MODE_DONTWAKE` suppresses waiter wake-up.

REQ-6: `UFFDIO_ZEROPAGE` — install zero page over `range`. `zeropage` field returns bytes installed or `-errno`. `MODE_DONTWAKE` honored.

REQ-7: `UFFDIO_CONTINUE` — resolve a MINOR fault by attaching the already-existing pagecache page. `mapped` returns bytes mapped. `MODE_WP` installs WP. `MODE_DONTWAKE` honored.

REQ-8: `UFFDIO_POISON` — install HWPOISON ptes over range so any future access raises SIGBUS BUS_MCEERR. `updated` returns bytes processed. Requires `UFFD_FEATURE_POISON`.

REQ-9: `UFFDIO_MOVE` — atomically move anon pages from `[src, src+len)` to `[dst, dst+len)`. `move` returns bytes moved or `-errno`. `MODE_ALLOW_SRC_HOLES` permits empty src ptes (installed as zero on dst). Requires `UFFD_FEATURE_MOVE`.

REQ-10: `UFFDIO_WRITEPROTECT` — `MODE_WP=1` sets WP on range; `MODE_WP=0` clears WP (resolves WP-faulting waiters). `MODE_DONTWAKE` meaningful only with `WP=0`.

REQ-11: `UFFDIO_WAKE` — wake all threads blocked on pagefaults in `range` without supplying a fix-up (used after lazy resolution).

REQ-12: `read(uffd, buf, n*sizeof(struct uffd_msg))` returns one or more `uffd_msg` records. `event` field is one of `UFFD_EVENT_*`. Per-`UFFD_EVENT_PAGEFAULT`: `pagefault.flags` encodes `WRITE | WP | MINOR`; `pagefault.address` is the faulting VA (masked to page boundary unless `UFFD_FEATURE_EXACT_ADDRESS`); `pagefault.feat.ptid` is the faulting tid if `UFFD_FEATURE_THREAD_ID`.

REQ-13: Per-`UFFD_EVENT_FORK`: child inherits uffd; `fork.ufd` is the child's fd (must be closed when manager finishes handling). Requires `UFFD_FEATURE_EVENT_FORK`.

REQ-14: Per-`UFFD_EVENT_REMAP`: emitted for `mremap` of a registered range; `remap.{from,to,len}` describes the move. Requires `UFFD_FEATURE_EVENT_REMAP`.

REQ-15: Per-`UFFD_EVENT_REMOVE`: emitted for `madvise(MADV_REMOVE | MADV_DONTNEED)` over a registered range; `remove.{start,end}`. Requires `UFFD_FEATURE_EVENT_REMOVE`.

REQ-16: Per-`UFFD_EVENT_UNMAP`: emitted for `munmap` over a registered range. Requires `UFFD_FEATURE_EVENT_UNMAP`.

REQ-17: Per-`UFFD_FEATURE_SIGBUS`: kernel sends SIGBUS to faulting thread instead of producing PAGEFAULT events; no manager needed.

REQ-18: Per-`UFFD_USER_MODE_ONLY=1`: kernel rejects (with SIGBUS) any kernel-mode page fault that would otherwise wait on this uffd. Default (0) allows kernel faults — restricted by `vm.unprivileged_userfaultfd` policy.

REQ-19: Per-ABI tail-field rule: `copy`/`zeropage`/`mapped`/`updated`/`move` are last 8 bytes; `copy_from_user` skips them so userland MAY leave them uninitialized on call.

REQ-20: Per-poll(POLLIN): readable when ≥ 1 event queued. EAGAIN on read of empty queue with O_NONBLOCK.

## Acceptance Criteria

- [ ] AC-1: `userfaultfd(0)` returns fd; flags { CLOEXEC, NONBLOCK, USER_MODE_ONLY } accepted.
- [ ] AC-2: `UFFDIO_API` rejects api != UFFD_API with EINVAL.
- [ ] AC-3: `UFFDIO_API` rejects unsupported feature bits with EINVAL.
- [ ] AC-4: `UFFDIO_REGISTER` returns range-ioctls bitmask matching VMA class (BASIC for hugetlbfs).
- [ ] AC-5: `UFFDIO_COPY` resolves MISSING fault; faulting thread wakes if not DONTWAKE.
- [ ] AC-6: `UFFDIO_ZEROPAGE` rejects with EINVAL on hugetlbfs (not in BASIC set).
- [ ] AC-7: `UFFDIO_WRITEPROTECT MODE_WP=1` causes subsequent stores to fault with `UFFD_PAGEFAULT_FLAG_WP`.
- [ ] AC-8: `UFFDIO_CONTINUE` resolves MINOR fault on shmem when MINOR_SHMEM feature negotiated.
- [ ] AC-9: `UFFDIO_POISON` causes subsequent loads to deliver SIGBUS BUS_MCEERR.
- [ ] AC-10: `UFFDIO_MOVE` returns bytes moved; src ptes cleared (or zero-filled with ALLOW_SRC_HOLES).
- [ ] AC-11: `read(uffd)` yields packed uffd_msg with event in 0x12..=0x16.
- [ ] AC-12: `UFFD_FEATURE_THREAD_ID` populates pagefault.feat.ptid.
- [ ] AC-13: `UFFD_FEATURE_SIGBUS` suppresses PAGEFAULT events; SIGBUS delivered to faulter.
- [ ] AC-14: `UFFD_USER_MODE_ONLY` rejects kernel-mode faults with SIGBUS, never blocks kernel.
- [ ] AC-15: `UFFDIO_UNREGISTER` on unregistered range returns EINVAL.
- [ ] AC-16: fork inherits uffd; FORK event emits child ufd (requires EVENT_FORK).

## Architecture

Rookery surface in `kernel/uffd/uapi.rs`:

```rust
#[repr(C)]
pub struct UffdioApi {
    pub api: u64,
    pub features: u64,
    pub ioctls: u64,
}

#[repr(C)]
pub struct UffdioRange {
    pub start: u64,
    pub len: u64,
}

#[repr(C)]
pub struct UffdioRegister {
    pub range: UffdioRange,
    pub mode: u64,
    pub ioctls: u64,
}

#[repr(C)]
pub struct UffdioCopy {
    pub dst: u64,
    pub src: u64,
    pub len: u64,
    pub mode: u64,
    pub copy: i64,
}

#[repr(C)]
pub struct UffdioZeropage {
    pub range: UffdioRange,
    pub mode: u64,
    pub zeropage: i64,
}

#[repr(C)]
pub struct UffdioWriteprotect {
    pub range: UffdioRange,
    pub mode: u64,
}

#[repr(C)]
pub struct UffdioContinue {
    pub range: UffdioRange,
    pub mode: u64,
    pub mapped: i64,
}

#[repr(C)]
pub struct UffdioPoison {
    pub range: UffdioRange,
    pub mode: u64,
    pub updated: i64,
}

#[repr(C)]
pub struct UffdioMove {
    pub dst: u64,
    pub src: u64,
    pub len: u64,
    pub mode: u64,
    pub move_: i64,
}

#[repr(C, packed)]
pub struct UffdMsg {
    pub event: u8,
    pub reserved1: u8,
    pub reserved2: u16,
    pub reserved3: u32,
    pub arg: UffdMsgArg, // tagged union by event
}
```

`Uffd::ioctl_api(uffd, &mut UffdioApi)`:
1. /* Reject if api != UFFD_API */
2. if api.api != UFFD_API: return -EINVAL.
3. /* Reject unsupported features */
4. if api.features & !UFFD_API_FEATURES: return -EINVAL.
5. uffd.features = api.features.
6. api.ioctls = UFFD_API_IOCTLS.
7. uffd.state = UFFD_STATE_RUNNING.
8. return 0.

`Uffd::ioctl_register(uffd, &mut UffdioRegister)`:
1. /* Validate alignment */
2. if reg.range.start % PAGE_SIZE != 0 ∨ reg.range.len % PAGE_SIZE != 0: return -EINVAL.
3. /* Validate mode */
4. if reg.mode & !UFFD_API_REGISTER_MODES: return -EINVAL.
5. /* Walk VMAs over range */
6. for vma in mm.vmas_overlapping(reg.range):
   - if !vma.supports_uffd(reg.mode): return -EINVAL.
   - vma.uffd_ctx = uffd; vma.vm_flags |= per-mode VM_UFFD_*.
7. /* Pick range-ioctls subset */
8. reg.ioctls = if hugetlbfs { UFFD_API_RANGE_IOCTLS_BASIC } else { UFFD_API_RANGE_IOCTLS }.
9. return 0.

`Uffd::ioctl_copy(uffd, &mut UffdioCopy)`:
1. /* Validate alignment + non-overlap */
2. if copy.dst%PAGE ∨ copy.len%PAGE: return -EINVAL.
3. n = mfill_atomic_copy(mm, copy.dst, copy.src, copy.len, copy.mode).
4. copy.copy = n as i64.
5. if !(copy.mode & UFFDIO_COPY_MODE_DONTWAKE): Uffd::wake_range(uffd, copy.dst, n).
6. return 0.

`Uffd::read_msg(uffd, buf, n)`:
1. /* Block until events queued (or EAGAIN if NONBLOCK) */
2. ev = uffd.event_queue.pop_or_block().
3. msg = UffdMsg::from(ev).
4. copy_to_user(buf, &msg, sizeof(UffdMsg)).
5. return bytes_copied.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `api_handshake_first` | INVARIANT | non-API ioctls fail before UFFDIO_API succeeds |
| `register_range_page_aligned` | INVARIANT | start, len multiples of PAGE_SIZE |
| `register_mode_bits_valid` | INVARIANT | mode ⊆ UFFD_API_REGISTER_MODES |
| `features_subset_supported` | INVARIANT | accepted features ⊆ UFFD_API_FEATURES |
| `tail_field_writeback_only` | INVARIANT | copy/zeropage/mapped/updated/move written, not read |
| `user_mode_only_rejects_kernel_fault` | INVARIANT | with USER_MODE_ONLY=1, kernel-mode fault → SIGBUS, not block |
| `msg_event_in_valid_range` | INVARIANT | event ∈ {0x12,..,0x16} on read |
| `register_ioctls_basic_iff_hugetlbfs` | INVARIANT | hugetlbfs ⟹ BASIC subset only |

### Layer 2: TLA+

`uapi/userfaultfd.tla`:
- States: UNINIT → APIed → REGISTERED → (faulting | resolving) → UNREGISTERED → CLOSED.
- Properties:
  - `safety_handshake_first` — UFFDIO_API precedes UFFDIO_REGISTER.
  - `safety_feature_monotone` — features set only at handshake; never expanded.
  - `safety_range_ioctls_match_vma` — issued ioctl ∈ register.ioctls.
  - `safety_no_kernel_block_user_mode_only` — USER_MODE_ONLY ⟹ kernel-fault SIGBUS.
  - `liveness_pagefault_resolved` — every PAGEFAULT event eventually COPY/ZEROPAGE/CONTINUE/MOVE/WAKE or fault-task killed.
  - `liveness_unregister_drains` — UNREGISTER eventually quiesces pending events.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `ioctl_api` post: features = requested ∩ supported | `Uffd::ioctl_api` |
| `ioctl_register` post: vma.uffd_ctx = uffd ∀ vma in range | `Uffd::ioctl_register` |
| `ioctl_copy` post: copy.copy ∈ {0..len, -errno} | `Uffd::ioctl_copy` |
| `ioctl_writeprotect` post: WP=1 → ptes WP; WP=0 → ptes RW + wake | `Uffd::ioctl_writeprotect` |
| `ioctl_poison` post: ptes installed with HWPOISON marker | `Uffd::ioctl_poison` |
| `ioctl_move` post: src ptes cleared; dst ptes mapped | `Uffd::ioctl_move` |
| `read_msg` post: msg.event ∈ UFFD_EVENT_* enum | `Uffd::read_msg` |

### Layer 4: Verus/Creusot functional

Per-Documentation/admin-guide/mm/userfaultfd.rst semantic equivalence: handshake → register → fault → fix-up → unregister. Per-LTP `testcases/kernel/syscalls/userfaultfd/*` conformance.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

## Grsecurity/PaX-style Reinforcement

- **GRKERNSEC_HARDEN_USERFAULTFD** — `userfaultfd(2)` and `USERFAULTFD_IOC_NEW` require `CAP_SYS_PTRACE` (in addition to `vm.unprivileged_userfaultfd`) whenever `UFFD_USER_MODE_ONLY` is **not** set; refuses kernel-mode-fault handling to unprivileged tasks, eliminating the well-known UAF / use-after-write race-window-extension primitive abused in CVE-2016-3672-style exploits.
- **PaX UDEREF on uffdio_* buffers** — every `copy_from_user`/`copy_to_user` of `uffdio_api`/`uffdio_register`/`uffdio_copy`/etc. crosses the user/kernel boundary with explicit per-field validation; reject any pointer whose target maps into kernel mode (defense against split-UAF / TOCTOU on the tail-field writeback regions).
- **PAX_RANDKSTACK at UFFD entry** — randomize the kernel stack on every ioctl entry to `Uffd::ioctl_*`, including the read-event path; prevents speculative-stack-layout disclosure via uffd-triggered stalls.
- **MAX_REGISTER_LIMIT** — enforce a per-uid ceiling on registered uffd ranges and total queued events; refuses with `-EMFILE` past quota, breaking event-flood DoS against the manager kthread.
- **GRKERNSEC_NO_FORK_FROM_USERFAULTFD** — block `UFFD_FEATURE_EVENT_FORK` whenever the manager process is `suid`/`sgid` or has elevated caps; prevents a privileged child from inheriting a uffd whose handler runs as the unprivileged parent (privilege-laundering vector).
- **CAP_SYS_PTRACE for UFFDIO_MOVE / UFFDIO_POISON** — moving anonymous pages across the address space and installing HWPOISON ptes are diagnostic-class operations; gate them behind ptrace capability instead of default-allowing them based on `UFFD_API_FEATURES`.
- **Per-`UFFD_USER_MODE_ONLY` default-on** — Rookery boots with `vm.userfaultfd_user_mode_only_default=1`, requiring an explicit sysctl override to allow kernel-mode-fault handling; aligned with PaX defence-in-depth defaults (deny first, allow second).
- **GRKERNSEC_LOG_UFFD** — audit-log every ioctl whose result has cross-mm effects (REGISTER / UNREGISTER / MOVE / POISON) with uid + range; deters silent in-kernel TOCTOU primitive farming.
- **Per-event-queue bounded** — hard cap on `uffd.event_queue.len()` per fd; refuses with `-EAGAIN` instead of unbounded kmalloc growth (defense against memory-pressure-DoS exposure of `select_bad_process`).

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `fs/userfaultfd.c` event-queue + fault-blocking implementation (covered in `mm/userfaultfd-impl.md` Tier-3)
- `mm/userfaultfd.c` mfill_atomic_* helpers (covered in `mm/userfaultfd-impl.md`)
- `vm.unprivileged_userfaultfd` sysctl (covered in `uapi/sysctl-vm.md`)
- `/dev/userfaultfd` chardev driver wiring (covered separately if expanded)
- userfaultfd-WP integration with shadow stack / CET (covered in `mm/wp-cet.md` if expanded)
- Implementation code
