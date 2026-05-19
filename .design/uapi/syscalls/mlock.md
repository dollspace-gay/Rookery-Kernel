# Tier-5: syscall 149 — mlock(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`149  common  mlock  sys_mlock`)
  - mm/mlock.c (`SYSCALL_DEFINE2(mlock, ...)`, `do_mlock`, `apply_vma_lock_flags`, `mlock_fixup`)
  - include/linux/mm.h (`VM_LOCKED`, `VM_LOCKONFAULT`, `VM_LOCKED_MASK`)
  - kernel/sys.c (`prlimit64`, `RLIMIT_MEMLOCK`)
  - mm/mmap.c (`mm->locked_vm`)
-->

## Summary

`mlock(2)` is **x86_64 syscall 149**, the primitive that pins virtual memory pages in physical RAM. It sets `VM_LOCKED` on each VMA intersecting `[addr, addr+len)` and faults every page in, guaranteeing no demand-paging and no swap-out for the affected range. `mlock` is bounded by `RLIMIT_MEMLOCK` (per-process locked-memory budget); over-budget requests fail with `-ENOMEM`. The pinned pages are still subject to migration (e.g., NUMA balancing, hugepage collapse) and to explicit unmap. Unprivileged processes share the system-wide locked-memory budget; only `CAP_IPC_LOCK` allows exceeding the rlimit. The semantic guarantee is: between successful `mlock` and `munlock` (or unmap), no major page fault will occur on the locked range.

Critical for: cryptographic key buffers (defense against swap exposure), real-time / low-latency code paths (defense against pf-latency), `gpg-agent`, `ssh-agent`, password managers, signal-handling-safe data structures, deterministic-latency embedded workloads.

## Signature

C (POSIX / man-pages):

```c
int mlock(const void *addr, size_t len);
```

glibc wrapper: `__mlock` → `INLINE_SYSCALL(mlock, 2, addr, len)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE2(mlock, unsigned long, start, size_t, len);
```

Rookery dispatch:

```rust
pub fn sys_mlock(start: u64, len: usize) -> SyscallResult<i32>;
```

## Parameters

| name  | type             | constraints                                                       | errno-on-bad |
|-------|------------------|-------------------------------------------------------------------|--------------|
| start | `const void *`   | rounded *down* to page boundary (NOT rejected when unaligned)     | n/a          |
| len   | `size_t`         | `>= 0`; combined with mis-aligned start, rounded up; sum no overflow | `EINVAL` / `ENOMEM` |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.
- Partial application is **not visible** to the caller: on a per-VMA failure, the kernel rolls back `VM_LOCKED` changes on previously-processed VMAs.

## Errors

| errno    | condition                                                                                              |
|----------|--------------------------------------------------------------------------------------------------------|
| `EINVAL` | `addr + len` overflows; computed `len == 0` after page-down/page-up adjustments.                       |
| `ENOMEM` | Some address in range is not mapped; `RLIMIT_MEMLOCK` exceeded and `!CAP_IPC_LOCK`; allocator failure during fault. |
| `EAGAIN` | Some pages cannot be locked (e.g., `VM_IO`, `VM_PFNMAP` ranges that disallow lock).                    |
| `EPERM`  | LSM denial; `CAP_IPC_LOCK` required and not held in mlock-strict mode (rare; legacy 2.6 mode only).    |
| `EINTR`  | `mmap_write_lock_killable` interrupted by fatal signal.                                                |

## ABI surface

No flag constants for `mlock(2)` itself. Internally consults / sets:

- `VM_LOCKED` (`0x00002000`) — VMA flag indicating locked status.
- `VM_LOCKONFAULT` (`0x00200000`) — `mlock2(MLOCK_ONFAULT)` only; `mlock(2)` always pre-faults so this bit is cleared.
- `VM_LOCKED_MASK` = `VM_LOCKED | VM_LOCKONFAULT`.

Resource limits:

- `RLIMIT_MEMLOCK` — default `64 KiB` for unprivileged; controllable via `/proc/PID/limits` or `prlimit`.
- `mm->locked_vm` — process-wide locked-page counter (in pages).
- `current->user->locked_shm` — per-user accounting for shm.

Capability:

- `CAP_IPC_LOCK` (`14`) — bypasses `RLIMIT_MEMLOCK`.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=start`, `%rsi=len`.
- REQ-2: `start = untagged_addr(start);` strip TBI/LAM tags.
- REQ-3: `len = (len + (start & ~PAGE_MASK))` — adjust for mis-aligned start. (Unlike most VMA syscalls, `mlock(2)` does **not** require page-aligned `start`; it rounds down.)
- REQ-4: `start &= PAGE_MASK;` round start *down*.
- REQ-5: `len = PAGE_ALIGN(len);` if rounded-up overflows ⟹ `-EINVAL`.
- REQ-6: `start + len < start` overflow ⟹ `-EINVAL`.
- REQ-7: `len == 0` after rounding ⟹ `0` (no-op success).
- REQ-8: `locked = len >> PAGE_SHIFT;` Compute would-be `mm->locked_vm + locked`. If `(mm->locked_vm + locked) << PAGE_SHIFT > rlimit(RLIMIT_MEMLOCK)` and `!capable(CAP_IPC_LOCK)` ⟹ `-ENOMEM`.
- REQ-9: `mmap_write_lock_killable(mm)` — fatal signal ⟹ `-EINTR`.
- REQ-10: `apply_vma_lock_flags(start, len, VM_LOCKED)`:
   - Walks VMAs in `[start, start+len)`.
   - For each, splits at endpoints if needed; sets `newflags = vma->vm_flags | VM_LOCKED; newflags &= ~VM_LOCKONFAULT;`.
   - `mlock_fixup(&vmi, vma, &prev, start, end, newflags)` updates `vma->vm_flags`, accounts `mm->locked_vm`, and populates pages via `__mm_populate(start, len, ignore_errors=true)`.
- REQ-11: Each affected VMA must be capable of locking: `VM_IO` / `VM_PFNMAP` not generally lockable ⟹ `-EAGAIN`.
- REQ-12: After lock+fault, pages are marked `PG_mlocked`; placed on the unevictable LRU; not reclaimable until `munlock` or unmap.
- REQ-13: Populate failures (e.g., file-backed page that yields `SIGBUS` due to truncate): the `ignore_errors=true` path in `__mm_populate` records but continues. Caller still sees `0` overall if the VMA accounting succeeded; the unfaulted page remains tracked as locked-but-not-resident (will fault on first access and be re-pinned).
- REQ-14: If `mm->locked_vm` post-call exceeds `RLIMIT_MEMLOCK` and `!CAP_IPC_LOCK` ⟹ pre-call rollback ensures we never partial-commit; return `-ENOMEM`.
- REQ-15: `mlock_fixup` may merge with adjacent VMAs of matching flags, reducing `mm->map_count`.
- REQ-16: On success, all pages in the range are resident. Subsequent `read`/`write`/exec accesses to the range incur no major page faults until `munlock(2)` or `munmap(2)`.
- REQ-17: Locked pages can still be migrated by NUMA balancing, hotplug, THP collapse — these are kernel-internal page relocations and do not violate the "no major fault" contract.
- REQ-18: `mlock` is idempotent: calling on a range already `VM_LOCKED` is a no-op success (no double-accounting of `mm->locked_vm`).

## Acceptance Criteria

- [ ] AC-1: `mlock(p, 4096)` after `p = mmap(NULL, 4096, ...)` succeeds; `/proc/PID/smaps` shows `Locked: 4 kB` for the VMA.
- [ ] AC-2: `mlock(p, very_large)` exceeding `RLIMIT_MEMLOCK` ⟹ `-ENOMEM` for unprivileged.
- [ ] AC-3: `mlock(p, very_large)` exceeding `RLIMIT_MEMLOCK` with `CAP_IPC_LOCK` ⟹ `0` (caps bypass).
- [ ] AC-4: `mlock(unmapped, 4096) ⟹ -ENOMEM`.
- [ ] AC-5: `mlock(p+1, 4095)` rounds down to page-aligned start and rounds up `len`; total locked pages equals the VMA pages covered.
- [ ] AC-6: After `mlock(p, len)`, reading from each page does not increment major-fault counters in `/proc/PID/stat`.
- [ ] AC-7: `mlock` then `munlock` then `mlock` is idempotent in accounting (`mm->locked_vm` not double-counted).
- [ ] AC-8: `mlock` on a `VM_PFNMAP` (e.g., `/dev/mem` mapping) ⟹ `-EAGAIN`.
- [ ] AC-9: `mlock` then `fork(2)` — the child inherits the VMAs as `VM_LOCKED` only if `mlockall(MCL_FUTURE)`/`MCL_CURRENT` semantics apply; otherwise child's pages are not locked (per POSIX).
- [ ] AC-10: After successful `mlock`, the locked pages survive a `cgroup memory.high` reclaim attempt (unevictable LRU placement verified).

## Architecture

```
struct MlockArgs { start: u64, len: usize }
```

`sys_mlock(args) -> i32`:

1. Strip address tag.
2. Adjust for unaligned start (round down; bump `len`).
3. Page-align `len` up; overflow ⟹ `-EINVAL`.
4. `len == 0` after rounding ⟹ `0`.
5. `let locked_pages = len >> PAGE_SHIFT;`
6. `mmap_write_lock_killable(mm)?;`
7. If `(mm.locked_vm + locked_pages) << PAGE_SHIFT > rlimit(RLIMIT_MEMLOCK)` and `!capable(CAP_IPC_LOCK)` ⟹ unlock; `-ENOMEM`.
8. `ret = apply_vma_lock_flags(start, len, VM_LOCKED);`
9. `mmap_write_unlock(mm);`
10. If `ret == 0` ⟹ `__mm_populate(start, len, ignore_errors=true)`.
11. Return `ret`.

`Mm::apply_vma_lock_flags(start, len, flags)`:

1. `end = start + len`.
2. `vma_iter_init(&vmi, mm, start);`
3. `vma = vma_find(&vmi, end);` — if no VMA in range ⟹ `-ENOMEM`.
4. Loop:
   - If `vma->vm_start > nstart` ⟹ gap detected ⟹ `-ENOMEM` (with rollback).
   - `newflags = (vma->vm_flags & ~VM_LOCKED_MASK) | flags;`
   - `mlock_fixup(&vmi, vma, &prev, nstart, min(vma->vm_end, end), newflags);`
   - Advance `nstart = vma->vm_end`.
   - If `nstart >= end` ⟹ done.

`Mm::mlock_fixup(vmi, vma, prev_out, start, end, newflags)`:

1. Split front/back if needed (`vma_split_at`).
2. Update accounting: `mm->locked_vm += (end - start) >> PAGE_SHIFT;` (only if `flags & VM_LOCKED` newly set).
3. Set `vma->vm_flags = newflags;` and call `vma_set_page_prot(vma)` for PTE protection update.
4. Try merge with neighbors via `vma_merge`.
5. Return `0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `range_overflow_rejected` | INVARIANT | `start + PAGE_ALIGN(len) >= start` and `<= TASK_SIZE`. |
| `rlimit_honored` | INVARIANT | Post-success, `mm.locked_vm * PAGE_SIZE <= rlimit(RLIMIT_MEMLOCK)` or `CAP_IPC_LOCK`. |
| `idempotent` | INVARIANT | `mlock(r) ; mlock(r)` ⟹ `mm.locked_vm` does not double-count. |
| `lock_held_during_flag_update` | INVARIANT | `vma->vm_flags` updated under `mmap_write_lock`. |
| `no_partial_commit` | INVARIANT | On error, no VMA in range has `VM_LOCKED` set newly. |
| `populate_after_unlock` | INVARIANT | `__mm_populate` runs after `mmap_write_unlock`. |

### Layer 2: TLA+

`uapi/syscalls/mlock.tla`:
- States: `{vmas, locked_vm, ptes, resident_pages, rlimit_memlock}`.
- Actions: `MlockOk`, `MlockOverRlimit`, `MlockGap`, `Idempotent`, `Rollback`.
- Properties:
  - `safety_rlimit_memlock_never_exceeded`,
  - `safety_locked_pages_are_resident`,
  - `liveness_mlock_terminates_under_capacity`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `apply_vma_lock_flags` either updates every VMA in range or none | `Mm::apply_vma_lock_flags` |
| `mlock_fixup` increments `locked_vm` exactly once per new lock | `Mm::mlock_fixup` |
| `__mm_populate` ignores errors only with explicit `ignore_errors=true` | `Mm::sys_mlock` |

### Layer 4: Verus/Creusot functional

`mlock(addr, len) ≡ POSIX.1-2024 mlock` with Linux extension of mid-page rounding-down on `addr`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mlock(2)` reinforcement:

- **Per `RLIMIT_MEMLOCK` enforcement** — pre-check before any VMA mutation; defense against unprivileged memory-pin DoS.
- **Per `CAP_IPC_LOCK` gate** — privileged bypass requires capability, not effective UID.
- **Per per-VMA rollback** — partial application is observable only as no-op on error; defense against accounting-skew attacks.
- **Per `mmap_write_lock_killable`** — interruptible by fatal signal; defense against unkillable processes blocked in mlock.
- **Per `VM_PFNMAP` denial** — non-faultable VMAs cannot be mlocked; defense against locking I/O ranges.
- **Per `__mm_populate` post-unlock** — population happens outside the mmap lock so concurrent VMA reads are not starved.
- **Per `PG_mlocked` flag** — pages placed on unevictable LRU; defense against reclaim-driven swap-out of locked pages.
- **Per-cgroup memlock accounting** — when configured, `mm->locked_vm` charges to the memcg's `memory.kmem.tcp.limit_in_bytes` (in modern kernels: cgroup v2 `memory.swap.max` interplay).

## Grsecurity/PaX-style Reinforcement

- **PAX_MPROTECT** — `mlock` does not affect protections; PaX W^X invariant unaffected.
- **PAX_NOEXEC** — irrelevant for mlock per se, but mlock'd ranges remain subject to PaX exec policy.
- **PAX_RANDMMAP** — irrelevant (no address selection).
- **PAX_RANDEXEC** — irrelevant.
- **GRKERNSEC_LOCKEDMEM_LOG** — grsec audit log entry for any process that lock'd memory above N pages (configurable threshold). Defense against silent memory-pin attacks.
- **PAX_REFCOUNT** — `mm->locked_vm` and `mm->mm_users` saturating counters; defense against integer-overflow accounting attacks (modern Linux uses `atomic_long_t`; PaX REFCOUNT additionally guards against wrap).
- **PAX_UDEREF** — `addr` is numeric; SMAP enforced; PaX user-deref check confirms no kernel dereference of these pointers.
- **PAX_MEMORY_SANITIZE** — irrelevant (no page free in mlock; only on munlock/munmap).
- **GRKERNSEC_BRUTE** — repeated mlock-driven OOM crashes detected as brute and throttled.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset rerolled at mlock syscall entry.
- **GRKERNSEC_RESLOG** — `RLIMIT_MEMLOCK` exceeded events logged with audit context (caller UID, PID, exe path).
- **Per RLIMIT_MEMLOCK strict mode** — grsec configurable strict mode that ignores `CAP_IPC_LOCK` for daemons running under specific roles, forcing them to the limit even when capable.
- **Defence against memlock-overcommit-DoS** — combined with `mlockall(MCL_FUTURE)`, grsec adds a kernel-side cap on total system-wide locked-vm percent.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `mlock2(2)` (separate Tier-5 — `mlock2.md`).
- `mlockall(2)` (separate Tier-5 — `mlockall.md`).
- `munlock(2)` / `munlockall(2)` (separate Tier-5 if expanded).
- `shmctl(SHM_LOCK)` (System V shared-memory lock; separate path).
- cgroup v2 memlock controllers.
- Implementation code.
