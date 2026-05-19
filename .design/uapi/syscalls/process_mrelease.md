# Tier-5 syscall: process_mrelease(2) — syscall 448

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/oom_kill.c (SYSCALL_DEFINE2(process_mrelease), __oom_reap_task_mm)
  - kernel/exit.c (do_exit interaction)
  - kernel/pid.c (pidfd_get_task)
  - arch/x86/entry/syscalls/syscall_64.tbl (448  common  process_mrelease)
-->

## Summary

`process_mrelease(2)` is a userspace-driven OOM-reaper hook: it requests the kernel to immediately reap the memory of a dying process identified by a `pidfd`. The target MUST already have been sent `SIGKILL` (or otherwise be in the exit path with `PF_EXITING`); `process_mrelease` then walks the target's mm and unmaps its anonymous and shmem pages without waiting for normal exit teardown. Designed for low-memory-killer userspace daemons (Android LMKD, systemd-oomd) that kill a memory hog and want the pages back fast so the system does not collapse before the killed task gets scheduled to clean up. Critical for: Android OOM-recovery latency, systemd-oomd responsiveness, container-OOM bounded latency.

## Signature

```c
int process_mrelease(int pidfd, unsigned int flags);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pidfd` | `int` | in | pidfd referencing the target process (from `pidfd_open` or `clone3(CLONE_PIDFD)`). |
| `flags` | `unsigned int` | in | Reserved; must be `0`. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; reap initiated (mm walked, anon/shmem unmapped). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EBADF` | `pidfd` is not a valid open fd. |
| `EINVAL` | `pidfd` is not a pidfd, or `flags != 0`. |
| `ESRCH` | Target task is gone (zombie reaped) or invalid. |
| `EINTR` | Reaper retry-loop interrupted by signal. |
| `EAGAIN` | Target not yet in `PF_EXITING`; caller should send SIGKILL first. |
| `EPERM` | Caller cannot signal target (distinct userns without CAP_KILL / CAP_SYS_PTRACE). |

## ABI surface

```text
__NR_process_mrelease (x86_64) = 448
__NR_process_mrelease (arm64)  = 448
__NR_process_mrelease (riscv)  = 448
__NR_process_mrelease (i386)   = 448

/* Available since Linux 5.15. */
/* flags reserved; pass 0. */
/* Target must be PF_EXITING when reaper attempts mmap_read_lock. */
```

## Compatibility contract

REQ-1: Syscall number is **448** on all architectures (new-syscall convention). ABI-stable.

REQ-2: `flags` MUST be `0`; non-zero returns `-EINVAL`.

REQ-3: `pidfd` MUST be a valid pidfd:
- `fget(pidfd)` returns a struct file with `f_op == &pidfd_fops`, else `-EINVAL`.

REQ-4: Target task lookup uses `pidfd_get_task(pidfd_file)`:
- Resolves to `struct task_struct *`; if NULL → `-ESRCH`.

REQ-5: Permission check: caller must be able to send signals to target — `check_kill_permission(SIGKILL, target)`, equivalent to `kill(target, SIGKILL)` being allowed. If denied → `-EPERM`.

REQ-6: Reaper precondition: target MUST be in `PF_EXITING` (post-SIGKILL, in do_exit path) OR have `MMF_OOM_VICTIM` set on its mm. Otherwise → `-EAGAIN`.

REQ-7: Reaper enters `__oom_reap_task_mm(target)`:
- Acquires `mmap_read_lock` (try-lock with retry; defers if contended).
- Walks `mm->mm_vmas()`, skipping VM_LOCKED, VM_PFNMAP, VM_HUGETLB.
- For each remaining VMA: `unmap_page_range(&tlb, vma, vma->vm_start, vma->vm_end)`.
- Sets `MMF_OOM_SKIP` on completion.

REQ-8: Concurrency: only one reaper active per mm at a time; idempotent (re-call returns 0 with no work if MMF_OOM_SKIP set).

REQ-9: Per-VM_LOCKED: mlocked VMAs are NOT reaped (preserves real-time guarantees).

REQ-10: Per-VM_HUGETLB: hugetlbfs VMAs are NOT reaped via this path (hugepages tear down at exit).

REQ-11: Per-VM_PFNMAP / VM_MIXEDMAP: device/driver mappings are NOT reaped (could disturb device DMA).

REQ-12: Per-`MMF_OOM_SKIP` idempotency: second `process_mrelease` for same target after first success is a no-op returning 0.

REQ-13: Per-namespace: pidfd resolves task across PID namespaces if visible; permission check uses caller's userns CAP_KILL/CAP_SYS_PTRACE.

REQ-14: Per-`PR_SET_DUMPABLE`: target's dumpable state does NOT relax permission; the check is signal-send equivalence (not ptrace).

REQ-15: Per-audit: every call audit-logged with target pid, mm bytes reaped (approx), caller real-uid.

## Acceptance Criteria

- [ ] AC-1: SIGKILL target then `process_mrelease(pidfd, 0)`: returns 0; RSS of target drops to ~0 before exit completes.
- [ ] AC-2: `flags = 1` returns `-EINVAL`.
- [ ] AC-3: `pidfd = 999` (invalid fd) returns `-EBADF`.
- [ ] AC-4: `pidfd` is a regular file fd: returns `-EINVAL`.
- [ ] AC-5: Target not yet SIGKILLed: returns `-EAGAIN`.
- [ ] AC-6: Target gone (zombie reaped): returns `-ESRCH`.
- [ ] AC-7: Caller in distinct userns without CAP: `-EPERM`.
- [ ] AC-8: Second call on same target: returns 0 (idempotent).
- [ ] AC-9: VM_LOCKED VMA not reaped; RSS for that VMA preserved.
- [ ] AC-10: VM_HUGETLB VMA not reaped via this path.
- [ ] AC-11: Audit record per call.

## Architecture

```rust
#[syscall(nr = 448, abi = "sysv")]
pub fn sys_process_mrelease(pidfd: i32, flags: u32) -> isize {
    ProcessMrelease::do_mrelease(pidfd, flags)
}
```

`ProcessMrelease::do_mrelease(pidfd, flags) -> isize`:
1. if flags != 0 { return -EINVAL; }
2. let f = FdTable::fget(pidfd).ok_or(-EBADF)?;
3. if !f.is_pidfd() { return -EINVAL; }
4. let target = Pid::from_pidfd(&f).ok_or(-ESRCH)?;
5. ProcessMrelease::check_kill_permission(&target)?;   // EPERM
6. let mm = get_task_mm(&target).ok_or(-ESRCH)?;
7. if !(target.flags.contains(PF_EXITING) || mm.flags.contains(MMF_OOM_VICTIM)) {
8.   mmput(mm); return -EAGAIN;
9. }
10. let bytes = ProcessMrelease::reap_mm(&target, &mm)?;
11. AuditLog::record_mrelease(target.pid, bytes);
12. mmput(mm);
13. 0

`ProcessMrelease::check_kill_permission(target) -> Result<()>`:
1. /* Equivalent to allow-kill check */
2. if !check_kill_permission(SIGKILL, /*si=NULL=*/None, target) { return Err(EPERM); }
3. Ok(())

`ProcessMrelease::reap_mm(target, mm) -> Result<usize>`:
1. if mm.flags.contains(MMF_OOM_SKIP) { return Ok(0); }
2. let mut retries = 0u32;
3. loop {
4.   if mmap_read_trylock(mm) { break; }
5.   if signal_pending(current) { return Err(EINTR); }
6.   if retries >= REAP_MAX_RETRIES { return Err(EINTR); }
7.   retries += 1; cond_resched();
8. }
9. let mut tlb = tlb_gather_mmu(mm);
10. let mut bytes = 0usize;
11. for vma in mm.mm_vmas() {
12.   if vma.flags & (VM_LOCKED | VM_PFNMAP | VM_HUGETLB | VM_MIXEDMAP) != 0 { continue; }
13.   unmap_page_range(&mut tlb, vma, vma.vm_start, vma.vm_end);
14.   bytes += (vma.vm_end - vma.vm_start) as usize;
15. }
16. tlb_finish_mmu(&mut tlb);
17. mmap_read_unlock(mm);
18. set_bit(MMF_OOM_SKIP, &mm.flags);
19. Ok(bytes)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `flags_reserved_zero` | INVARIANT | flags != 0 ⟹ EINVAL. |
| `pidfd_strict` | INVARIANT | non-pidfd fd ⟹ EINVAL. |
| `kill_permission_required` | INVARIANT | check_kill_permission false ⟹ EPERM. |
| `pf_exiting_or_oom_victim` | INVARIANT | target not exiting ⟹ EAGAIN. |
| `mmap_read_lock_held_during_reap` | INVARIANT | reaper holds mmap_read_lock. |
| `skip_vm_locked` | INVARIANT | VM_LOCKED VMAs not unmapped. |
| `idempotent_skip` | INVARIANT | MMF_OOM_SKIP set ⟹ second call no-op. |

### Layer 2: TLA+

`mm/process-mrelease.tla`:
- States: per-arg-validate, per-pidfd-resolve, per-permission, per-precondition, per-reap, per-set-skip, per-return.
- Properties:
  - `safety_no_reap_without_kill_perm` — never reap without signal-send permission.
  - `safety_no_reap_unless_dying` — reap only if PF_EXITING or MMF_OOM_VICTIM.
  - `safety_skip_locked_vma` — VM_LOCKED VMAs preserved.
  - `safety_idempotent` — second call no-op.
  - `liveness_returns` — every call terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_mrelease` post: success ⟹ MMF_OOM_SKIP set | `ProcessMrelease::do_mrelease` |
| `reap_mm` post: VM_LOCKED VMAs untouched | `ProcessMrelease::reap_mm` |
| `check_kill_permission` post: ok ⟹ kill allowed | `ProcessMrelease::check_kill_permission` |

### Layer 4: Verus / Creusot functional

Per-`process_mrelease(2)` man-page semantic equivalence. LMKD selftest + LTP `process_mrelease01..03` pass.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`process_mrelease(2)` reinforcement:

- **Per-flags reserved-zero** — defense against per-flag smuggling.
- **Per-pidfd strict type check** — defense against per-fd-confusion.
- **Per-`check_kill_permission` (signal-send equivalence)** — defense against per-unrelated-process reap.
- **Per-PF_EXITING precondition** — defense against per-reap-of-running-task (data corruption).
- **Per-VM_LOCKED preserved** — defense against per-RT-disturbance.
- **Per-VM_PFNMAP / VM_HUGETLB preserved** — defense against per-driver-DMA disturbance.
- **Per-MMF_OOM_SKIP idempotent** — defense against per-double-reap UAF.
- **Per-mmap_read_trylock with retry bound** — defense against per-deadlock vs writer.

## Grsecurity / PaX surface

- **PaX UDEREF on pidfd struct file access** — defense against per-file-pointer kernel deref; SMAP forced.
- **GRKERNSEC_HARDEN_PTRACE not applicable** (this is kill-permission, not ptrace) — but **GRKERNSEC_HARDEN_KILL** governs signal-send across userns boundaries; same gating reused: caller must dominate target uid OR have CAP_KILL in target userns.
- **CAP_SYS_PTRACE strict** for sibling-userns: distinct userns CAP refused; only init-userns CAP_SYS_PTRACE (or CAP_KILL with matching uid) is honored.
- **PAX_REFCOUNT on target mm_users** — defense against per-mm refcount overflow UAF during long reap.
- **GRKERNSEC_PROC restrictions** on /proc/[pid] visibility translate to `-ESRCH` (not EPERM) for invisible targets to avoid existence-leak.
- **GRKERNSEC_AUDIT_MRELEASE** — kernel.audit emits ANOM_PROCESS_MRELEASE record per call with caller real-uid + target pid + bytes reaped.
- **PaX KERNEXEC on `__oom_reap_task_mm` walker** — defense against per-W^X violation in page-table walk.
- **PAX_RAP on tlb_finish_mmu indirect call** — RAP-tagged; defense against per-ROP gadget.
- **GRKERNSEC_DENY_PROCESS_MRELEASE toggle** — restricts the syscall to processes with `CAP_SYS_ADMIN` in init_userns (recommended on hosts without an LMKD).
- **GRKERNSEC_LOG_RATE_LIMIT on EPERM/EAGAIN** — defense against per-log-flood probing reachability.
- **PaX MEMORY_SANITIZE on freed reaped pages** — defense against per-page-leak-of-victim data; reaped pages overwritten with poison before return to allocator.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Per-`__oom_reap_task_mm` walker details (covered in Tier-3 `mm/oom-kill.md`).
- Per-pidfd lifecycle (covered in Tier-3 `kernel/pid.md`).
- Android LMKD policy (out of kernel scope).
- Implementation code.
