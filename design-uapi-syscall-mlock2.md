---
title: "Tier-5: syscall 325 — mlock2(2)"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`mlock2(2)` is **x86_64 syscall 325** (added in Linux 4.4), the flag-aware variant of `mlock(2)`. It pins memory in the same way as `mlock(2)`, but accepts a `flags` argument that currently selects between **pre-fault locking** (default; identical to `mlock(2)`) and **on-fault locking** (`MLOCK_ONFAULT`). With `MLOCK_ONFAULT`, the VMA gets `VM_LOCKED | VM_LOCKONFAULT`: pages are not immediately faulted in, but any page that *is* (or becomes) resident is pinned and cannot be reclaimed. `mlock2(MLOCK_ONFAULT)` is the right primitive for sparse / lazy-touched buffers where the caller wants the no-swap guarantee only for pages it actually uses.

Critical for: large guard-page-laden arenas (jemalloc), real-time audio buffers that rarely-but-deterministically touch ends, sandboxes that allocate huge VAs but populate sparsely.

### Acceptance Criteria

- [ ] AC-1: `mlock2(p, 4096, 0)` is semantically equivalent to `mlock(p, 4096)`: pages faulted and locked.
- [ ] AC-2: `mlock2(p, 4096, MLOCK_ONFAULT)` sets `VM_LOCKONFAULT` on the VMA; no pre-fault occurs; touching the page pins it.
- [ ] AC-3: `mlock2(p, len, MLOCK_ONFAULT|0x02) == -EINVAL` (unknown flag).
- [ ] AC-4: `mlock2(p, 0, 0) == 0` (no-op).
- [ ] AC-5: `mlock2(p, very_large, MLOCK_ONFAULT)` exceeding `RLIMIT_MEMLOCK` ⟹ `-ENOMEM` (accounting is conservative).
- [ ] AC-6: `mlock2(p, len, MLOCK_ONFAULT)` then touching a page ⟹ page now has `PG_mlocked` and is on unevictable LRU.
- [ ] AC-7: `mlock2(p, len, MLOCK_ONFAULT)` then `munlock(p, len)` clears both bits and `mm->locked_vm` accounting.
- [ ] AC-8: `mlock2(p, len, 0)` then `mlock2(p, len, MLOCK_ONFAULT)` switches mode; pre-faulted pages remain resident and pinned.
- [ ] AC-9: `mlock2(p, len, 0)` on an unmapped range ⟹ `-ENOMEM`.
- [ ] AC-10: `mlock2(p, len, MLOCK_ONFAULT)` then `fork(2)` — child VMA inherits `VM_LOCKONFAULT` only via `mlockall(MCL_FUTURE)`; otherwise child sees unlocked VMA (per Linux semantic).

### Architecture

```
struct Mlock2Args { start: u64, len: usize, flags: i32 }
```

`sys_mlock2(args) -> i32`:

1. Validate `flags & ~MLOCK_ONFAULT == 0` ⟹ else `-EINVAL`.
2. Strip address tag.
3. Adjust start/len (round start down; bump len; page-align len up).
4. Overflow check ⟹ `-EINVAL`.
5. If `len == 0` ⟹ `0`.
6. `let vm_flags = VM_LOCKED | (if flags & MLOCK_ONFAULT { VM_LOCKONFAULT } else { 0 });`
7. `mmap_write_lock_killable(mm)?;`
8. Resource check (`mm->locked_vm` projected + `RLIMIT_MEMLOCK`).
9. `ret = apply_vma_lock_flags(start, len, vm_flags);`
10. `mmap_write_unlock(mm);`
11. If `ret == 0 && !(flags & MLOCK_ONFAULT)` ⟹ `__mm_populate(start, len, ignore_errors=true);`.
12. Return `ret`.

Compared to `sys_mlock`:

- Same `apply_vma_lock_flags` / `mlock_fixup` plumbing.
- Difference: `vm_flags` parameter includes `VM_LOCKONFAULT` when `MLOCK_ONFAULT` set.
- Difference: pre-fault step (`__mm_populate`) skipped under `MLOCK_ONFAULT`.

### Out of Scope

- `mlock(2)` (separate Tier-5 — `mlock.md`).
- `mlockall(2)` (separate Tier-5 — `mlockall.md`).
- `munlock(2)` / `munlockall(2)` (separate Tier-5 if expanded).
- Page-fault internals (`do_fault` / `handle_mm_fault`) — described in fault subsystem doc.
- cgroup memlock controllers.
- Implementation code.

### signature

C (POSIX / man-pages):

```c
int mlock2(const void *addr, size_t len, unsigned int flags);
```

glibc wrapper: `__mlock2` → `INLINE_SYSCALL(mlock2, 3, addr, len, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE3(mlock2, unsigned long, start, size_t, len, int, flags);
```

Rookery dispatch:

```rust
pub fn sys_mlock2(start: u64, len: usize, flags: i32) -> SyscallResult<i32>;
```

### parameters

| name  | type             | constraints                                                       | errno-on-bad |
|-------|------------------|-------------------------------------------------------------------|--------------|
| start | `const void *`   | rounded down to page boundary                                     | n/a          |
| len   | `size_t`         | `>= 0`; rounded up; sum no overflow                               | `EINVAL`     |
| flags | `unsigned int`   | `0` or `MLOCK_ONFAULT` (`0x01`)                                   | `EINVAL`     |

### return value

- Success: `0`.
- Failure: `< 0` — negated errno.

### errors

| errno    | condition                                                                                              |
|----------|--------------------------------------------------------------------------------------------------------|
| `EINVAL` | `flags & ~MLOCK_ONFAULT != 0` (unknown flag bits); `addr + len` overflows; `len == 0` after rounding (no, this is `0` success); arithmetic overflow during page-up. |
| `ENOMEM` | Some address in range is not mapped; `RLIMIT_MEMLOCK` exceeded and `!CAP_IPC_LOCK`; allocator failure during fault (only for non-`MLOCK_ONFAULT` path). |
| `EAGAIN` | Some pages cannot be locked (e.g., `VM_IO`, `VM_PFNMAP` ranges).                                       |
| `EPERM`  | LSM denial; `CAP_IPC_LOCK` strict-mode requirement.                                                    |
| `EINTR`  | `mmap_write_lock_killable` interrupted by fatal signal.                                                |

### abi surface

`flags`:

- `MLOCK_ONFAULT` `0x01` — lock pages on fault only; do not pre-fault. The VMA is marked `VM_LOCKED | VM_LOCKONFAULT`. Subsequent faults inside the VMA bring pages in and pin them.

Reserved future-flag bits: any unused bit set ⟹ `-EINVAL`. Kernel uses `flags & ~MLOCK_ONFAULT == 0` as the validity check.

`VM_*` flags:

- `VM_LOCKED` (`0x00002000`) — pages locked.
- `VM_LOCKONFAULT` (`0x00200000`) — lock-on-fault mode.
- `VM_LOCKED_MASK = VM_LOCKED | VM_LOCKONFAULT`.

Capability and resource:

- `CAP_IPC_LOCK` — bypasses `RLIMIT_MEMLOCK`.
- `RLIMIT_MEMLOCK` — per-process locked-memory budget.

Accounting: `mm->locked_vm` increments by the VMA size in pages **regardless of MLOCK_ONFAULT** — the budget tracks would-be-locked, not actually-resident, because each page that does fault in *will* be locked. This is conservative-but-fair: the same accounting model whether you pre-fault or not.

### compatibility contract

- REQ-1: Argument lowering: `%rdi=start`, `%rsi=len`, `%rdx=flags`.
- REQ-2: `start = untagged_addr(start);` strip TBI/LAM tags.
- REQ-3: `flags & ~MLOCK_ONFAULT != 0` ⟹ `-EINVAL`. (Kernel validates flag bits first.)
- REQ-4: `len = len + (start & ~PAGE_MASK); start &= PAGE_MASK; len = PAGE_ALIGN(len);` if overflow ⟹ `-EINVAL`.
- REQ-5: `start + len < start` ⟹ `-EINVAL`.
- REQ-6: `len == 0` after rounding ⟹ `0` success.
- REQ-7: Compute `locked_pages = len >> PAGE_SHIFT;`. If `(mm->locked_vm + locked_pages) << PAGE_SHIFT > rlimit(RLIMIT_MEMLOCK)` and `!CAP_IPC_LOCK` ⟹ `-ENOMEM`.
- REQ-8: `mmap_write_lock_killable(mm)` — fatal signal ⟹ `-EINTR`.
- REQ-9: Compute `vm_flags = VM_LOCKED;` if `flags & MLOCK_ONFAULT` ⟹ `vm_flags |= VM_LOCKONFAULT`.
- REQ-10: `apply_vma_lock_flags(start, len, vm_flags)`:
   - Walks intersecting VMAs; splits at endpoints; sets `newflags = (vma->vm_flags & ~VM_LOCKED_MASK) | vm_flags;`.
   - Calls `mlock_fixup(...)` to update `vma->vm_flags`, account `mm->locked_vm`, and update PTE protections.
- REQ-11: If `!(flags & MLOCK_ONFAULT)` ⟹ post-unlock `__mm_populate(start, len, ignore_errors=true)` faults pages in.
- REQ-12: If `flags & MLOCK_ONFAULT` ⟹ no pre-fault. Resident pages in the range are already pinned (via `mlock_drain_remote_pvec` and per-page `mlock_folio`); non-resident pages will be pinned on first fault.
- REQ-13: `mlock2(MLOCK_ONFAULT)` over a range already `VM_LOCKED` (no `VM_LOCKONFAULT`) ⟹ flag is *changed* to LOCKONFAULT for the entire range; previously pre-faulted resident pages stay pinned; non-resident pages are no longer scheduled for pre-fault by *this* call but may already be resident from previous activity.
- REQ-14: Conversely, `mlock2(0)` (no flag) over a `VM_LOCKED | VM_LOCKONFAULT` range clears `VM_LOCKONFAULT` and triggers `__mm_populate` to fault remaining non-resident pages.
- REQ-15: `mm->locked_vm` does **not** double-count: idempotent calls on already-locked ranges (in either mode) result in zero accounting change.
- REQ-16: Per-VMA-rollback on per-VMA failure: any partial VMA mutation reverted; caller sees the error, no `VM_LOCKED` set.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flag_bits_validated` | INVARIANT | `flags & ~MLOCK_ONFAULT == 0`. |
| `mode_set_atomically` | INVARIANT | All VMAs in range have identical `VM_LOCKED|VM_LOCKONFAULT` state post-success. |
| `populate_iff_eager_mode` | INVARIANT | `__mm_populate` runs ⟺ `MLOCK_ONFAULT` is not set. |
| `rlimit_honored_conservatively` | INVARIANT | `mm.locked_vm * PAGE_SIZE <= RLIMIT_MEMLOCK` or `CAP_IPC_LOCK` post-success. |
| `lock_held_during_flag_update` | INVARIANT | `vma->vm_flags` updated under `mmap_write_lock`. |

### Layer 2: TLA+

`uapi/syscalls/mlock2.tla`:
- States: `{vmas, locked_vm, vm_lockonfault_set, resident_pages, rlimit_memlock}`.
- Actions: `Mlock2Eager`, `Mlock2OnFault`, `ModeSwitch`, `Rollback`.
- Properties:
  - `safety_onfault_no_eager_populate`,
  - `safety_locked_vm_accounting_conservative`,
  - `liveness_mlock2_terminates`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `apply_vma_lock_flags` either commits across all VMAs in range or none | `Mm::apply_vma_lock_flags` |
| `MLOCK_ONFAULT` ⟹ `vma->vm_flags & VM_LOCKONFAULT` set post-call | `Mm::sys_mlock2` |
| Mode-switch from LOCKONFAULT to eager triggers `__mm_populate` to catch residue | `Mm::sys_mlock2` |

### Layer 4: Verus/Creusot functional

`mlock2(addr, len, flags) ≡ Linux man-page semantics (no POSIX standard exists for `mlock2`).`

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mlock2(2)` reinforcement:

- **Per-flag-bit validation** — unknown bits rejected before any state mutation.
- **Per conservative `RLIMIT_MEMLOCK` accounting** — `MLOCK_ONFAULT` charges as if all pages would be locked; defense against allocate-huge-then-touch-piecemeal-bypass of memlock budget.
- **Per per-VMA rollback** — partial-application invisible to caller.
- **Per `mmap_write_lock_killable`** — interruptible by fatal signal.
- **Per `VM_PFNMAP` / `VM_IO` denial** — non-faultable VMAs cannot be locked.
- **Per `__mm_populate` post-unlock** — population happens outside the mmap lock.
- **Per `PG_mlocked` flag** — pages placed on unevictable LRU.
- **Per page-fault path integration** — `MLOCK_ONFAULT` pages are pinned via the standard fault handler (`do_anonymous_page` / `do_pte_missing` / `do_fault`) checking `VM_LOCKONFAULT`.

### grsecurity/pax-style reinforcement

- **PAX_MPROTECT** — `mlock2` does not affect protections; W^X invariant unaffected.
- **PAX_NOEXEC** — irrelevant for mlock2 per se.
- **PAX_RANDMMAP** — irrelevant (no address selection).
- **PAX_RANDEXEC** — irrelevant.
- **GRKERNSEC_LOCKEDMEM_LOG** — grsec audit log entry on lock above threshold; LOCKONFAULT counted as if eager.
- **PAX_REFCOUNT** — `mm->locked_vm` saturating counter via atomic_long_t (PaX REFCOUNT additionally guards against wrap on 32-bit hosts).
- **PAX_UDEREF** — `addr` numeric; SMAP enforced; no kernel-side dereference of these pointers.
- **PAX_MEMORY_SANITIZE** — irrelevant (no page free in mlock2).
- **GRKERNSEC_BRUTE** — repeated mlock2 OOM crashes detected as brute and throttled.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset rerolled at mlock2 syscall entry.
- **GRKERNSEC_RESLOG** — `RLIMIT_MEMLOCK` exceeded events logged with audit context.
- **RLIMIT_MEMLOCK strict mode** — grsec configurable mode that ignores `CAP_IPC_LOCK` for daemons.
- **Defence against MLOCK_ONFAULT-budget-evasion** — grsec extends to require that any process invoking `MLOCK_ONFAULT` for ranges larger than configurable threshold be audited; without that, a malicious process could reserve huge VA and stage gradual lock-up attack.
- **PAX_NORANDOM_LOCKONFAULT bypass guard** — grsec ensures that LOCKONFAULT mode does not interact pathologically with `PAX_RANDMMAP` (no leak of unmap timing through reuse delays).

