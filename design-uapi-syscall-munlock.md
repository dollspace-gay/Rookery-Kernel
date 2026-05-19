---
title: "Tier-5 syscall: munlock(2) — syscall 150"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`munlock(2)` unpins the address range `[addr, addr + length)` from RAM, undoing a prior `mlock(2)` or the locked-on-fault portion of `mlock2(2)`. After the call, the pages may be paged out, migrated, or unmapped as if they had never been locked. The kernel re-enables LRU eviction by clearing `VM_LOCKED` on every VMA in the range and by removing the per-folio mlock count contribution so that the global `RLIMIT_MEMLOCK` accounting drops.

`munlock(2)` is the dual of `mlock(2)`. It does not free memory and does not require the pages to be resident — it simply releases the lock. Critical for: long-running services that lock secret material (gpg-agent, ssh-agent, browsers) and need to release secrets, real-time applications that lock only during a deadline-bound section, and any caller that needs to drop the `RLIMIT_MEMLOCK` charge to lock a different region.

### Acceptance Criteria

- [ ] AC-1: After `mlock(addr, len)` then `munlock(addr, len)`, `/proc/self/status` `VmLck` decreases by `len`.
- [ ] AC-2: `munlock` on a region that was never locked returns 0 (no-op).
- [ ] AC-3: `munlock(addr, len)` where `addr + len` overflows `ULONG_MAX` returns `-EINVAL`.
- [ ] AC-4: `munlock` on an unmapped range returns `-ENOMEM`.
- [ ] AC-5: `munlock` partially covering a VMA splits the VMA and clears `VM_LOCKED` on only the requested portion.
- [ ] AC-6: After unlock, pages can be swapped out by reclaim.
- [ ] AC-7: `mlock2(MLOCK_ONFAULT)` then `munlock`: `VM_LOCKONFAULT` cleared even if no page was ever faulted in.
- [ ] AC-8: `munlock` returns before any per-cpu drained folio reverts to the LRU (guaranteed via `mlock_drain_local`).
- [ ] AC-9: Without `CAP_IPC_LOCK`: `munlock` succeeds (no capability required).
- [ ] AC-10: `RLIMIT_MEMLOCK` accounting decreases atomically with the unlock; concurrent `mlock` sees the freed budget.

### Architecture

```rust
#[syscall(nr = 150, abi = "sysv")]
pub fn sys_munlock(addr: UserAddr, length: usize) -> isize {
    Mlock::do_munlock(addr, length)
}
```

`Mlock::do_munlock(addr, length) -> isize`:
1. let start = addr & !(PAGE_SIZE - 1);
2. let end = match addr.checked_add(length).map(page_align) {
3.    Some(e) if e <= TASK_SIZE => e,
4.    _                          => return -EINVAL,
5. };
6. if end == start { return 0; }
7. let mm = current().mm();
8. mm.mmap_write_lock_killable()?;
9. let res = Mlock::apply_vma_lock_flags(mm, start, end, 0);
10. mm.mmap_write_unlock();
11. mlock_drain_local();
12. res

`Mlock::apply_vma_lock_flags(mm, start, end, newflags) -> isize`:
1. let mut prev = None;
2. let mut cursor = start;
3. let mut last_err: isize = 0;
4. while cursor < end {
5.    let Some(vma) = mm.find_vma_intersection(cursor, end) else {
6.        last_err = -ENOMEM; break;
7.    };
8.    if vma.start() > cursor { last_err = -ENOMEM; break; }
9.    let nstart = max(vma.start(), cursor);
10.   let nend = min(vma.end(), end);
11.   let nflags = (vma.flags() & !(VM_LOCKED | VM_LOCKONFAULT)) | newflags;
12.   match mlock_fixup(vma, &mut prev, nstart, nend, nflags) {
13.     Ok(())  => {}
14.     Err(e)  => { last_err = e; break; }
15.   }
16.   cursor = nend;
17. }
18. last_err

### Out of Scope

- `mlock(2)` and `mlock2(2)` lock-side semantics (covered in `mlock.md`, `mlock2.md`).
- `mlockall(2)` / `munlockall(2)` (covered in their own Tier-5 docs).
- mm/mlock.c LRU re-introduction logic (covered in Tier-3 `mm/mlock.md`).
- Implementation code.

### signature

```c
int munlock(const void *addr, size_t length);
```

The kernel rounds `addr` down to a page boundary and `length` up so that the affected range covers every page even partially overlapped by `[addr, addr + length)`.

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `addr` | `const void *` | in | Start of the range; rounded down to the page. |
| `length` | `size_t` | in | Byte length of the range; rounded up to the next page boundary. |

### return value

| Value | Meaning |
|---|---|
| `0` | Success — entire range had `VM_LOCKED` cleared. |
| `-1` + `errno` | Failure. |

### errors

| errno | Trigger |
|---|---|
| `EAGAIN` | A transient memory-management failure prevented updating a folio's mlock state (rare; tier-3 reaper interaction). |
| `EINVAL` | Range arithmetic overflows; `addr + length` wraps. |
| `ENOMEM` | Some portion of the range is not mapped. |
| `EPERM` | The range crosses into a VMA the caller has no rights to mutate (rare; primarily applies to attached SysV-shm segments with restricted ACLs). |

### abi surface

```text
__NR_munlock  (x86_64)  = 150
__NR_munlock  (arm64)   = 229
__NR_munlock  (riscv)   = 229
__NR_munlock  (i386)    = 151

/* No flag word; complements mlock2(2)'s MLOCK_ONFAULT by clearing the
   on-fault lock just like a regular lock — i.e. munlock removes any
   lock policy, not just the eager-resident one. */
```

### compatibility contract

REQ-1: Syscall number is **150** on x86_64. ABI-stable.

REQ-2: Address normalization: `start = addr & ~(PAGE_SIZE - 1)` and `end = PAGE_ALIGN(addr + length)`. If `end` overflows the user address space, return `-EINVAL`.

REQ-3: For every VMA covering `[start, end)`, the kernel runs `mlock_fixup(vma, prev, start, end, newflags)` where `newflags = vma->vm_flags & ~(VM_LOCKED | VM_LOCKONFAULT)`.

REQ-4: `mlock_fixup` performs a VMA split if the range partially covers a VMA, ensuring exactly the requested pages have their lock cleared.

REQ-5: After per-VMA clearing, the kernel walks the affected pages via `folio_referenced` / `munlock_vma_folio` and decrements the per-folio mlock count. When the count reaches zero, the folio is returned to the LRU list.

REQ-6: Accounting: each successfully unlocked page reduces `current->mm->locked_vm` by 1; combined with shared-memory tracking it lowers the `RLIMIT_MEMLOCK` charge.

REQ-7: Gaps: if any portion of the range lacks a VMA, the kernel processes the prefix it could find and returns `-ENOMEM`. The contract is: best-effort partial-completion with error indication; callers MUST treat the entire range as in an indeterminate state.

REQ-8: `munlock` on a range with no `VM_LOCKED` set is a no-op success — POSIX requires idempotence.

REQ-9: Hugepage VMAs: lock clearing is at the huge-page granularity; partial-huge-page boundaries cause VMA split.

REQ-10: Per-cpu mlock pagevec drain: `munlock_vma_folio` defers the LRU return until either the per-cpu vec is full or the syscall path triggers `mlock_drain_local`, which `munlock` does on its exit path to ensure semantic completion before return.

REQ-11: `munlock` does NOT require `CAP_IPC_LOCK`. Locking memory was the privileged operation; releasing it is unprivileged.

REQ-12: `MLOCK_ONFAULT` lock policy (set by `mlock2(2)`) is also cleared, regardless of whether the corresponding pages have been faulted in.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `range_overflow_rejected` | INVARIANT | `addr + length > TASK_SIZE` ⟹ EINVAL. |
| `mmap_write_lock_balanced` | INVARIANT | locked iff dropped on every exit edge. |
| `vm_locked_cleared` | INVARIANT | success ⟹ every covered VMA has `VM_LOCKED == 0`. |
| `lockonfault_cleared` | INVARIANT | success ⟹ `VM_LOCKONFAULT == 0` too. |
| `idempotent_on_unlocked` | INVARIANT | range with no VM_LOCKED ⟹ rc = 0. |
| `mlock_drain_on_exit` | INVARIANT | per-cpu drain called before return. |

### Layer 2: TLA+

`mm/munlock.tla`:
- States: per-arg-check, per-mmap-lock, per-vma-walk, per-fixup, per-drain.
- Properties:
  - `safety_partial_progress_with_error` — gap inside range ⟹ ENOMEM + partial unlock.
  - `safety_no_priv_required` — no capability gate.
  - `safety_lock_cleared_on_success` — all VM_LOCKED cleared.
  - `liveness_munlock_terminates` — every call returns (or signal-killable).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_munlock` post: returns -EINVAL on overflow | `Mlock::do_munlock` |
| `apply_vma_lock_flags` post: cursor ≤ end | `Mlock::apply_vma_lock_flags` |
| `mlock_fixup` post: VM_LOCKED ⊕ newflags consistent | `Mlock::mlock_fixup` |
| `do_munlock` post: mmap_write lock balanced | `Mlock::do_munlock` |

### Layer 4: Verus / Creusot functional

Per-`munlock(2)` man-page and POSIX `munlock` semantics; LTP `kernel/syscalls/munlock/` selftests pass.

### hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`munlock(2)` reinforcement:

- **Per-overflow EINVAL** — defense against per-arithmetic-overflow into kernel address space.
- **Per-VMA-fixup atomicity** — defense against per-partial-state UAF on split failure.
- **Per-cpu drain on syscall exit** — defense against per-deferred-reclaim leak of the lock budget.
- **Per-killable mmap lock** — defense against per-DoS via fault storms blocking munlock.
- **Per-MLOCK_ONFAULT clearing complete** — defense against per-lingering-residency.
- **Per-RLIMIT_MEMLOCK atomic decrease** — defense against per-budget-double-charge.

### grsecurity / pax surface

- **PaX UDEREF on `addr` validation** — defense against per-user-addr kernel-deref bug; SMAP forced during VMA lookup.
- **GRKERNSEC_HARDEN_RLIMIT_MEMLOCK** — re-evaluate `RLIMIT_MEMLOCK` after munlock so that races between munlock-and-relock from different threads cannot temporarily exceed the user's accumulated lock budget.
- **PAX_REFCOUNT on mm reference during mlock_drain_local** — defense against per-refcount-underflow UAF on background drain worker.
- **GRKERNSEC_HARDEN_USERFAULTFD interaction** — munlock on a userfaultfd-registered range checks that the caller is the same task that registered the uffd, blocking a peer-userfaultfd-driven page-pinning bypass.
- **PaX KERNEXEC on mlock_fixup function pointers** — VMA-ops dispatch goes through the trampoline so an attacker cannot redirect mlock_fixup to a controlled callback.
- **GRKERNSEC_AUDIT_IPC on unlock of CAP_IPC_LOCK-acquired range** — emits an audit record when a previously privileged lock is released, so post-mortem analysis can see when secret-bearing pages were released.
- **GRKERNSEC_TRUSTED_PATH constraints on locked SysV-shm** — munlock on a SysV-shm attached from a non-trusted mount denied unless caller still owns the segment.
- **PAX_USERCOPY no copy_to_user** — munlock has no user output; PAX_USERCOPY confirms this via the syscall-shape annotation.
- **Per-RLIMIT_MEMLOCK enforcement on re-lock** — releasing memory does not bypass the enforced cap on subsequent locks even within the same syscall window.
- **Per-VM_LOCKED clearing under mmap_write_lock** — defense against per-concurrent-mlock race that could re-arm VM_LOCKED mid-walk.
- **GRKERNSEC_HIDESYM on backtrace if reaper hits mlock-cleared folio** — kernel-mode WARN does not leak the folio pointer to dmesg.

