# Tier-5 syscall: munlockall(2) — syscall 152

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/mlock.c (SYSCALL_DEFINE0(munlockall), apply_mlockall_flags)
  - mm/internal.h (mlock_drain_local)
  - include/linux/mman.h (VM_LOCKED, VM_LOCKONFAULT)
  - arch/x86/entry/syscalls/syscall_64.tbl (152  common  munlockall)
-->

## Summary

`munlockall(2)` unlocks every page currently mapped into the calling process's address space, undoing any prior `mlockall(2)` and any per-region `mlock(2)` / `mlock2(2)` calls. After the syscall returns, no VMA has `VM_LOCKED` or `VM_LOCKONFAULT` set, no future allocations (heap, stack growth, file maps, anonymous maps) will be locked-on-fault, and the entire `RLIMIT_MEMLOCK` charge previously contributed by this process is released.

The call is the dual of `mlockall(2)`. It is unprivileged: any process can release locks it holds. It is also the recommended emergency-release primitive — for example, a service handling `SIGTERM` may call `munlockall()` to allow pages to be paged out promptly so that a successor process can lock its own working set. Critical for: real-time daemons exiting deadline mode, security daemons rotating secret material, container-orchestration shutdown sequences.

## Signature

```c
int munlockall(void);
```

The syscall takes no arguments. There is no flag word.

## Parameters

(none)

## Return value

| Value | Meaning |
|---|---|
| `0` | Success — every VMA in the process now has `VM_LOCKED` and `VM_LOCKONFAULT` clear. |
| `-1` + `errno` | Failure (rare). |

## Errors

| errno | Trigger |
|---|---|
| `EINTR` | A signal was delivered while waiting on `mmap_write_lock_killable`. |
| `ENOMEM` | (theoretical) A VMA split for partial unlock failed; in practice `munlockall` does not split because it spans whole VMAs. |

## ABI surface

```text
__NR_munlockall  (x86_64)  = 152
__NR_munlockall  (arm64)   = 231
__NR_munlockall  (riscv)   = 231
__NR_munlockall  (i386)    = 153

/* No argument register is consumed; the syscall takes zero arguments. */
/* Process-global effect: every VMA, including future ones, lose VM_LOCKED. */
```

## Compatibility contract

REQ-1: Syscall number is **152** on x86_64. ABI-stable.

REQ-2: Process-global clearing: the kernel walks `current->mm` and applies `apply_mlockall_flags(0)` so that every VMA's flags become `vma->vm_flags & ~(VM_LOCKED | VM_LOCKONFAULT)`.

REQ-3: Future-allocation policy reset: `current->mm->def_flags &= ~(VM_LOCKED | VM_LOCKONFAULT)`. After `munlockall()`, neither `mmap(2)` nor `brk(2)` produces a locked VMA unless an explicit per-call lock is requested.

REQ-4: Whole-mm walk under `mmap_write_lock_killable`. The hold time is `O(N)` in VMA count; the kernel checks `fatal_signal_pending` between VMAs to allow `EINTR`.

REQ-5: After per-VMA clearing, the kernel returns affected folios to the LRU via `munlock_vma_folio` and drains the per-cpu mlock pagevec via `mlock_drain_local` before returning to user space.

REQ-6: `RLIMIT_MEMLOCK` accounting: `mm->locked_vm` is set to 0; SysV-shm-locked counts are released; the user's `user_struct->locked_shm` is decremented for SHM segments owned by this process.

REQ-7: Idempotence: if no VMA had `VM_LOCKED` set, `munlockall` succeeds as a no-op.

REQ-8: `munlockall` does NOT require `CAP_IPC_LOCK`. The lock-acquisition syscalls were the privileged primitive; release is unprivileged.

REQ-9: Concurrency: a parallel `mlockall(MCL_FUTURE)` call may set the future-allocation flag immediately after `munlockall` returns; the semantics are last-writer-wins under the mm write lock.

REQ-10: Threading: `munlockall` is per-mm; all threads sharing the mm observe the cleared state. The kernel does not iterate threads.

REQ-11: HugeTLB and DAX mappings are also unlocked. There is no carve-out.

REQ-12: PROT_NONE VMAs created by `mlockall(MCL_FUTURE)` still exist after `munlockall`; only the lock bits are cleared, not the mapping policy.

## Acceptance Criteria

- [ ] AC-1: After `mlockall(MCL_CURRENT)` then `munlockall()`, `VmLck` in `/proc/self/status` is 0.
- [ ] AC-2: After `mlockall(MCL_FUTURE)` then `munlockall()`, `mmap` produces a non-locked VMA.
- [ ] AC-3: `munlockall()` on a process with no locks succeeds.
- [ ] AC-4: `munlockall()` returns `-EINTR` if a fatal signal is pending during the walk.
- [ ] AC-5: `munlockall()` clears both `VM_LOCKED` and `VM_LOCKONFAULT`.
- [ ] AC-6: Per-cpu mlock pagevec drained before return.
- [ ] AC-7: SysV-shm locks attributed to this process released.
- [ ] AC-8: Without `CAP_IPC_LOCK`: `munlockall()` succeeds.
- [ ] AC-9: `RLIMIT_MEMLOCK` budget released; concurrent thread can `mlock()` the freed budget.
- [ ] AC-10: HugeTLB-only mm: `munlockall` clears huge VMA locks without splitting.

## Architecture

```rust
#[syscall(nr = 152, abi = "sysv")]
pub fn sys_munlockall() -> isize {
    Mlock::do_munlockall()
}
```

`Mlock::do_munlockall() -> isize`:
1. let mm = current().mm();
2. mm.mmap_write_lock_killable()?;
3. /* Per-VMA walk */
4. let mut prev = None;
5. for vma in mm.vmas_mut() {
6.    let nflags = vma.flags() & !(VM_LOCKED | VM_LOCKONFAULT);
7.    match mlock_fixup(vma, &mut prev, vma.start(), vma.end(), nflags) {
8.        Ok(())  => {}
9.        Err(e)  => { mm.mmap_write_unlock(); return e; }
10.   }
11.   if signal_pending(current()) {
12.       mm.mmap_write_unlock();
13.       return -EINTR;
14.   }
15. }
16. /* Process-global future-allocation policy */
17. mm.def_flags &= !(VM_LOCKED | VM_LOCKONFAULT);
18. /* Account */
19. mm.locked_vm.store(0, Ordering::Relaxed);
20. mm.mmap_write_unlock();
21. mlock_drain_local();
22. user_struct::release_locked_shm(current_user());
23. 0

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `mmap_write_lock_balanced` | INVARIANT | locked iff dropped on every exit edge. |
| `all_vmas_cleared` | INVARIANT | success ⟹ no VMA has VM_LOCKED. |
| `def_flags_cleared` | INVARIANT | success ⟹ mm.def_flags has no VM_LOCKED/VM_LOCKONFAULT. |
| `pagevec_drained_on_exit` | INVARIANT | per-cpu drain called before return. |
| `locked_vm_zeroed` | INVARIANT | success ⟹ mm.locked_vm == 0. |
| `eintr_on_signal` | INVARIANT | killable signal ⟹ EINTR (write lock released). |

### Layer 2: TLA+

`mm/munlockall.tla`:
- States: per-mmap-lock, per-vma-walk, per-def-flags-clear, per-shm-release, per-drain.
- Properties:
  - `safety_global_clear` — after success, no per-process VMA has VM_LOCKED.
  - `safety_no_priv_required` — call always allowed regardless of CAP_IPC_LOCK.
  - `safety_future_policy_dropped` — MCL_FUTURE bit cleared.
  - `liveness_munlockall_terminates` — every call returns under signal or success.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_munlockall` post: every vma.flags & VM_LOCKED == 0 | `Mlock::do_munlockall` |
| `do_munlockall` post: mm.def_flags clean | `Mlock::do_munlockall` |
| `do_munlockall` post: locked_vm == 0 | `Mlock::do_munlockall` |
| `do_munlockall` post: mmap lock released on every exit | `Mlock::do_munlockall` |

### Layer 4: Verus / Creusot functional

Per-`munlockall(2)` man-page and POSIX `munlockall` semantics; LTP `kernel/syscalls/munlockall/` selftests pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`munlockall(2)` reinforcement:

- **Per-walk killable lock** — defense against per-DoS via fault storms blocking the walk.
- **Per-cpu drain on exit** — defense against per-deferred-reclaim leak.
- **Per-def_flags reset atomic** — defense against per-future-allocation lock-leak.
- **Per-RLIMIT_MEMLOCK release atomic** — defense against per-budget double-account.
- **Per-EINTR on fatal signal** — defense against per-shutdown stall.
- **Per-process scope, never per-mm escalation** — defense against per-thread-tampering with another mm.

## Grsecurity / PaX surface

- **PaX UDEREF inapplicable** — `munlockall` takes no user pointer; no copy_from_user; SMAP/SMEP-irrelevant on entry.
- **GRKERNSEC_HARDEN_RLIMIT_MEMLOCK** — releases `user_struct->locked_shm` under a per-user spinlock so that races with concurrent SysV-shm-locking can't double-charge.
- **PAX_REFCOUNT on user_struct->locked_shm** — saturating decrement; defense against per-refcount-underflow when munlockall races with shmctl(IPC_RMID).
- **GRKERNSEC_AUDIT_IPC** — emits an audit_log record when a process with previously CAP_IPC_LOCK-locked memory releases its locks, so privileged release events are traceable.
- **PaX KERNEXEC on mlock_fixup function-pointer indirection** — defense against per-VMA-ops hijack during the walk.
- **GRKERNSEC_HIDESYM on warning paths** — WARN if `locked_vm` underflows; the warning sanitizes mm pointer.
- **GRKERNSEC_HARDEN_USERFAULTFD interaction** — if any VMA is uffd-registered, munlockall confirms uffd owner == current thread group; defense against peer-uffd lock-state tampering.
- **GRKERNSEC_TRUSTED_PATH on locked SysV-shm** — release of a SHM lock acquired on a non-trusted-path segment denied unless ownership unchanged.
- **PAX_USERCOPY no user copy** — confirmed; the syscall is pure mm state change.
- **Per-thread-group consistency** — defense against per-thread-race: munlockall is serialized via mmap_write_lock so partial states are invisible to peer threads.
- **Per-MCL_FUTURE reset under mm lock** — defense against per-policy stale read by a parallel mmap.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `mlockall(2)` (covered in `mlockall.md`).
- `mlock(2)` / `mlock2(2)` / `munlock(2)` (covered in their own Tier-5 docs).
- mm/mlock.c LRU re-introduction logic (covered in Tier-3 `mm/mlock.md`).
- Implementation code.
