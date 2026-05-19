# Tier-5: syscall 151 — mlockall(2)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - arch/x86/entry/syscalls/syscall_64.tbl (`151  common  mlockall  sys_mlockall`)
  - mm/mlock.c (`SYSCALL_DEFINE1(mlockall, ...)`, `apply_mlockall_flags`)
  - include/uapi/asm-generic/mman.h (`MCL_CURRENT`, `MCL_FUTURE`, `MCL_ONFAULT`)
  - include/linux/mm.h (`VM_LOCKED`, `VM_LOCKONFAULT`)
-->

## Summary

`mlockall(2)` is **x86_64 syscall 151**, the whole-process variant of `mlock(2)`. It locks **all** of the calling process's currently-mapped pages (`MCL_CURRENT`) and/or causes **all future** mappings to be locked (`MCL_FUTURE`), with optional on-fault semantics (`MCL_ONFAULT`, added Linux 4.4). When `MCL_FUTURE` is set, the kernel records `VM_LOCKED` into `mm->def_flags`; every subsequent `mmap`, `brk`, stack-extension, etc. will produce VMAs already flagged `VM_LOCKED`, so newly faulted-in pages are immediately pinned. This is the right primitive for real-time processes that want a global no-swap, no-fault guarantee for everything they touch over their lifetime.

Critical for: real-time audio and video servers, robotics control loops, safety-critical embedded software, cryptographic daemons that should never page out, low-latency trading systems.

## Signature

C (POSIX / man-pages):

```c
int mlockall(int flags);
```

glibc wrapper: `__mlockall` → `INLINE_SYSCALL(mlockall, 1, flags)`.

Kernel SYSCALL_DEFINE:

```c
SYSCALL_DEFINE1(mlockall, int, flags);
```

Rookery dispatch:

```rust
pub fn sys_mlockall(flags: i32) -> SyscallResult<i32>;
```

## Parameters

| name  | type  | constraints                                                              | errno-on-bad |
|-------|-------|--------------------------------------------------------------------------|--------------|
| flags | `int` | bitmask of `MCL_CURRENT | MCL_FUTURE | MCL_ONFAULT`; at least one of CURRENT/FUTURE required | `EINVAL`     |

## Return value

- Success: `0`.
- Failure: `< 0` — negated errno.

## Errors

| errno    | condition                                                                                              |
|----------|--------------------------------------------------------------------------------------------------------|
| `EINVAL` | `flags == 0`; `flags & ~(MCL_CURRENT|MCL_FUTURE|MCL_ONFAULT) != 0`; `MCL_ONFAULT` set without either `MCL_CURRENT` or `MCL_FUTURE`. |
| `ENOMEM` | `MCL_CURRENT` requested and total VM exceeds `RLIMIT_MEMLOCK` and `!CAP_IPC_LOCK`; allocator failure during pre-fault. |
| `EAGAIN` | Some pages cannot be locked (e.g., `VM_IO`, `VM_PFNMAP`).                                              |
| `EPERM`  | LSM denial; in pre-2.6.9 kernels, `CAP_IPC_LOCK` was required unconditionally — modern kernels allow unprivileged within `RLIMIT_MEMLOCK`. |
| `EINTR`  | `mmap_write_lock_killable` interrupted by fatal signal.                                                |

## ABI surface

`flags` (bitwise OR):

- `MCL_CURRENT`  `0x01` — lock all pages currently mapped in the address space.
- `MCL_FUTURE`   `0x02` — lock all future mappings (set `VM_LOCKED` in `mm->def_flags`).
- `MCL_ONFAULT`  `0x04` — lock pages on-fault (not pre-faulted). Must be combined with `MCL_CURRENT` and/or `MCL_FUTURE`.

Combinations and effect on `mm->def_flags`:

| flags                                | def_flags effect                                  | current-VMA effect                         |
|--------------------------------------|---------------------------------------------------|--------------------------------------------|
| `MCL_CURRENT`                        | unchanged                                         | lock all VMAs; pre-fault                   |
| `MCL_FUTURE`                         | `|= VM_LOCKED;` `&= ~VM_LOCKONFAULT;`             | unchanged                                  |
| `MCL_CURRENT|MCL_FUTURE`             | `|= VM_LOCKED;` `&= ~VM_LOCKONFAULT;`             | lock all VMAs; pre-fault                   |
| `MCL_CURRENT|MCL_ONFAULT`            | unchanged                                         | lock all VMAs with `VM_LOCKONFAULT`; no pre-fault |
| `MCL_FUTURE|MCL_ONFAULT`             | `|= VM_LOCKED|VM_LOCKONFAULT;`                    | unchanged                                  |
| `MCL_CURRENT|MCL_FUTURE|MCL_ONFAULT` | `|= VM_LOCKED|VM_LOCKONFAULT;`                    | lock all VMAs with `VM_LOCKONFAULT`; no pre-fault |

Resource:

- `RLIMIT_MEMLOCK` — checked against `mm->total_vm` (in pages) when `MCL_CURRENT` is set.
- `CAP_IPC_LOCK` — bypasses `RLIMIT_MEMLOCK`.

## Compatibility contract

- REQ-1: Argument lowering: `%rdi=flags`.
- REQ-2: `flags == 0` ⟹ `-EINVAL`.
- REQ-3: `flags & ~(MCL_CURRENT|MCL_FUTURE|MCL_ONFAULT) != 0` ⟹ `-EINVAL`.
- REQ-4: `(flags & MCL_ONFAULT) && !(flags & (MCL_CURRENT|MCL_FUTURE))` ⟹ `-EINVAL`.
- REQ-5: If `flags & MCL_CURRENT`:
   - `if (mm->total_vm << PAGE_SHIFT) > rlimit(RLIMIT_MEMLOCK) && !capable(CAP_IPC_LOCK)` ⟹ `-ENOMEM`.
- REQ-6: `mmap_write_lock_killable(mm)` — fatal signal ⟹ `-EINTR`.
- REQ-7: Compute `to_add = 0; if flags & MCL_FUTURE { to_add |= VM_LOCKED; if flags & MCL_ONFAULT { to_add |= VM_LOCKONFAULT; } }`.
- REQ-8: Compute `to_clear = 0; if (flags & MCL_FUTURE) && !(flags & MCL_ONFAULT) { to_clear |= VM_LOCKONFAULT; }`.
- REQ-9: Update `mm->def_flags = (mm->def_flags & ~(VM_LOCKED|VM_LOCKONFAULT)) | to_add;` (clearing previous lock bits first).
- REQ-10: If `flags & MCL_CURRENT`:
   - Compute per-VMA flag mask `vma_flags = if flags & MCL_ONFAULT { VM_LOCKED|VM_LOCKONFAULT } else { VM_LOCKED }`.
   - Walk every VMA: `newflags = (vma->vm_flags & ~VM_LOCKED_MASK) | vma_flags;` then `mlock_fixup(...)`.
   - Account `mm->locked_vm += (vma_end - vma_start) >> PAGE_SHIFT` (skipping already-locked portions).
- REQ-11: `mmap_write_unlock(mm)`.
- REQ-12: If `flags & MCL_CURRENT && !(flags & MCL_ONFAULT)` ⟹ for each VMA in process, `__mm_populate(vma_start, vma_len, ignore_errors=true)`.
- REQ-13: Subsequent `mmap` / `brk` / `mremap` / stack-extension under `MCL_FUTURE` get `mm->def_flags` ORed into `vma->vm_flags` at VMA-creation time, so new pages are locked.
- REQ-14: `MCL_FUTURE` is cleared by `munlockall(2)` (which clears `VM_LOCKED|VM_LOCKONFAULT` from `mm->def_flags`).
- REQ-15: Idempotence: `mlockall(MCL_CURRENT|MCL_FUTURE)` twice is a no-op in accounting.
- REQ-16: `mlockall(MCL_CURRENT)` does *not* set `MCL_FUTURE`; previously-set `mm->def_flags` is left as-is.
   - Counter-example: `mlockall(MCL_CURRENT|MCL_FUTURE)` followed by `mlockall(MCL_CURRENT)` does **clear** `VM_LOCKED|VM_LOCKONFAULT` from `def_flags` per REQ-9 (because we always rewrite def_flags from the current flag-set). Callers must combine flags explicitly to preserve FUTURE.
- REQ-17: `fork(2)` inheritance: child inherits `mm->def_flags`, so `MCL_FUTURE` carries over. `MCL_CURRENT` already-locked VMAs are inherited as `VM_LOCKED` per the standard COW semantic.

## Acceptance Criteria

- [ ] AC-1: `mlockall(0) == -EINVAL`.
- [ ] AC-2: `mlockall(MCL_CURRENT)` succeeds; `/proc/PID/smaps` shows all VMAs as `Locked: ...`.
- [ ] AC-3: `mlockall(MCL_FUTURE)`, then `mmap(NULL, 4096, ..., MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)` ⟹ resulting VMA has `VM_LOCKED`; pages are resident and pinned after first access.
- [ ] AC-4: `mlockall(MCL_FUTURE|MCL_ONFAULT)`, then `mmap` ⟹ VMA has `VM_LOCKED|VM_LOCKONFAULT`; pages locked on fault.
- [ ] AC-5: `mlockall(MCL_ONFAULT) == -EINVAL` (no CURRENT or FUTURE).
- [ ] AC-6: `mlockall(MCL_CURRENT)` exceeding `RLIMIT_MEMLOCK` unprivileged ⟹ `-ENOMEM`.
- [ ] AC-7: `mlockall(MCL_CURRENT)` with `CAP_IPC_LOCK` ⟹ succeeds regardless of `mm->total_vm`.
- [ ] AC-8: `mlockall(MCL_FUTURE)` + `fork(2)`: child inherits `MCL_FUTURE` semantics.
- [ ] AC-9: `mlockall(MCL_CURRENT|MCL_FUTURE)` then `munlockall()` clears both `mm->def_flags` lock bits and all VMA `VM_LOCKED` bits.
- [ ] AC-10: `mlockall(MCL_FUTURE)` followed by `brk(grow)` ⟹ new heap pages are immediately pinned.

## Architecture

```
struct MlockallArgs { flags: i32 }
```

`sys_mlockall(args) -> i32`:

1. Validate `flags` (REQ-2, REQ-3, REQ-4).
2. If `flags & MCL_CURRENT` and `(mm.total_vm << PAGE_SHIFT) > rlimit(RLIMIT_MEMLOCK)` and `!CAP_IPC_LOCK` ⟹ `-ENOMEM`.
3. `mmap_write_lock_killable(mm)?;`
4. Compute new `def_flags`:
   - `let mut new_def = mm.def_flags & !(VM_LOCKED | VM_LOCKONFAULT);`
   - `if flags & MCL_FUTURE { new_def |= VM_LOCKED; if flags & MCL_ONFAULT { new_def |= VM_LOCKONFAULT; } }`
   - `mm.def_flags = new_def;`
5. If `flags & MCL_CURRENT`:
   - `let vma_flags = VM_LOCKED | (if flags & MCL_ONFAULT { VM_LOCKONFAULT } else { 0 });`
   - `apply_mlockall_flags(mm, vma_flags);` walks every VMA, splits as needed, sets new flags.
6. `mmap_write_unlock(mm);`
7. If `flags & MCL_CURRENT && !(flags & MCL_ONFAULT)`:
   - For each VMA, `__mm_populate(vma.vm_start, vma.vm_end - vma.vm_start, ignore_errors=true)`.
8. Return `0`.

`Mm::apply_mlockall_flags(mm, vm_flags)`:

1. `vma_iter_init(&vmi, mm, 0);`
2. For each `vma` in mm:
   - `newflags = (vma->vm_flags & ~VM_LOCKED_MASK) | vm_flags;`
   - `mlock_fixup(&vmi, vma, &prev, vma->vm_start, vma->vm_end, newflags);`
3. Return `0`.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_valid` | INVARIANT | `flags & ~(MCL_CURRENT|MCL_FUTURE|MCL_ONFAULT) == 0`. |
| `combinations_legal` | INVARIANT | `MCL_ONFAULT` requires `MCL_CURRENT` or `MCL_FUTURE`. |
| `def_flags_overwrite_correct` | INVARIANT | `mm.def_flags` lock bits exactly reflect `flags`. |
| `current_path_fault` | INVARIANT | `MCL_CURRENT && !MCL_ONFAULT` ⟹ all VMA pages resident post-call. |
| `rlimit_honored` | INVARIANT | Pre-call check against `RLIMIT_MEMLOCK` and `CAP_IPC_LOCK`. |

### Layer 2: TLA+

`uapi/syscalls/mlockall.tla`:
- States: `{def_flags, vmas, locked_vm, rlimit_memlock}`.
- Actions: `LockallCurrent`, `LockallFuture`, `LockallBoth`, `OnFault*`, `Munlockall`, `MmapAfter`.
- Properties:
  - `safety_future_propagates_to_new_vmas`,
  - `safety_current_locks_existing_vmas`,
  - `liveness_mlockall_terminates`.

### Layer 3: Verus invariants

| Invariant | Component |
|---|---|
| `apply_mlockall_flags` updates every VMA exactly once | `Mm::apply_mlockall_flags` |
| Post-`MCL_FUTURE`, `mm->def_flags` contains `VM_LOCKED` (and optionally `VM_LOCKONFAULT`) | `Mm::sys_mlockall` |
| Post-`MCL_CURRENT && !MCL_ONFAULT`, `__mm_populate` runs for every VMA outside the mmap lock | `Mm::sys_mlockall` |

### Layer 4: Verus/Creusot functional

`mlockall(flags) ≡ POSIX.1-2024 mlockall` for `MCL_CURRENT|MCL_FUTURE`; Linux extensions per `man 2 mlockall` for `MCL_ONFAULT`.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`mlockall(2)` reinforcement:

- **Per-flag-validation** — invalid combinations rejected before any state mutation.
- **Per `RLIMIT_MEMLOCK` enforcement** — checked against `mm->total_vm` for `MCL_CURRENT`.
- **Per `CAP_IPC_LOCK` gate** — privileged bypass requires capability.
- **Per `mmap_write_lock_killable`** — interruptible by fatal signal.
- **Per `__mm_populate` post-unlock** — population happens outside the mmap lock; per-VMA loop allows concurrent reader progress.
- **Per `def_flags` is overwritten not OR'd** — caller must combine flags explicitly; defense against accidental persistence of stale lock-mode after partial reconfiguration.
- **Per per-VMA `mlock_fixup`** — flag changes go through the same path as `mlock(2)`/`mlock2(2)`, ensuring identical accounting and merge behavior.

## Grsecurity/PaX-style Reinforcement

- **PAX_MPROTECT** — irrelevant for mlockall (no protection change).
- **PAX_NOEXEC** — irrelevant for mlockall itself, but `MCL_FUTURE` interacts with PaX: future VMAs still subject to PaX exec policy.
- **PAX_RANDMMAP** — irrelevant (no address selection).
- **PAX_RANDEXEC** — irrelevant.
- **GRKERNSEC_LOCKEDMEM_LOG** — grsec audit log entry for `mlockall(MCL_CURRENT)` regardless of size, given the broad scope.
- **PAX_REFCOUNT** — `mm->locked_vm` saturating atomic counter; defense against accounting wrap.
- **PAX_UDEREF** — no user pointer dereferenced.
- **PAX_MEMORY_SANITIZE** — irrelevant (no page free).
- **GRKERNSEC_BRUTE** — repeated mlockall OOM crashes detected as brute and throttled.
- **GRKERNSEC_HIDESYM** — error printks redact kernel pointers.
- **PAX_RANDKSTACK** — kstack offset rerolled at mlockall syscall entry.
- **GRKERNSEC_RESLOG** — `RLIMIT_MEMLOCK` exceeded events logged with audit context (caller UID, PID, exe path).
- **RLIMIT_MEMLOCK strict mode** — grsec configurable mode that ignores `CAP_IPC_LOCK` for daemons.
- **System-wide locked-vm cap** — grsec optional global ceiling on aggregate locked memory across all processes; defense against mlockall-fork-bomb DoS.
- **`MCL_FUTURE` propagation audit** — grsec records cases where `MCL_FUTURE` is set by a process whose RLIMIT_MEMLOCK is unlimited, since this is a sign of trusted privileged daemons (audit log integrity).
- **Per `def_flags` integrity** — grsec ensures `mm->def_flags` cannot be set via debug interfaces (e.g., `/proc/PID/syscall`) to bypass capability checks.

## Open Questions

(none at this Tier-5 level)

## Out of Scope

- `mlock(2)` (separate Tier-5 — `mlock.md`).
- `mlock2(2)` (separate Tier-5 — `mlock2.md`).
- `munlockall(2)` (separate Tier-5 if expanded).
- VMA-creation path's consultation of `mm->def_flags` — described in `mmap.md` and `brk.md`.
- cgroup memlock controllers.
- Implementation code.
