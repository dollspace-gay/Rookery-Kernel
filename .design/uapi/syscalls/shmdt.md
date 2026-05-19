# Tier-5 syscall: shmdt(2) — syscall 67

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - ipc/shm.c (SYSCALL_DEFINE1(shmdt), ksys_shmdt, do_shmdt)
  - ipc/shm.c (shm_close VMA close hook)
  - mm/mmap.c (do_munmap)
  - include/linux/shm.h (shm_vm_ops)
  - arch/x86/entry/syscalls/syscall_64.tbl (67  common  shmdt)
-->

## Summary

`shmdt(2)` detaches the System V shared-memory segment whose attachment begins at `shmaddr` from the calling process's address space. The kernel locates the VMA whose start equals `shmaddr` and whose vm_ops is `shm_vm_ops`, unmaps it (and any contiguous VMAs that share the same underlying shm file as a chain, the legacy "partial-attach" range case), decrements the segment's `shm_nattch`, updates `shm_dtime` / `shm_lpid`, and — if the segment is `SHM_DEST` and `shm_nattch` reaches zero — destroys the segment.

Critical for: every `shmat`-using consumer to release its share at exit / cleanup, container teardown completeness, and segment reaping in `IPC_RMID`-then-detach flows. A missed `shmdt` keeps a `IPC_RMID`'d segment alive indefinitely (visible as `(deleted)` in `ipcs`).

## Signature

```c
int shmdt(const void *shmaddr);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `shmaddr` | `const void *` | in | The address previously returned by `shmat(2)` for this attachment. Must be the exact start of the attachment. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Success; attachment removed. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EINVAL` | No SysV-shm VMA starts at `shmaddr`, or `shmaddr` is unaligned, or `shmaddr` is outside `TASK_SIZE`. |

(`shmdt` is famously errno-poor: any malformed or non-matching address returns `EINVAL`.)

## ABI surface

```text
__NR_shmdt (x86_64)   = 67
__NR_shmdt (i386)     = 398 (post 5.1 direct; pre-5.1 via ipc(SHMDT, ...))
__NR_shmdt (arm64)    = 197
__NR_shmdt (generic)  = 197
```

## Compatibility contract

REQ-1: Syscall number is **67** on x86_64; **197** on arm64 / generic.

REQ-2: `shmaddr` must equal the start of a VMA whose `vm_ops == &shm_vm_ops`. Any other address (including mid-VMA, or a non-shm VMA) returns `EINVAL`.

REQ-3: Implementation walks `mm->mm_rb` (or maple-tree equivalent) finding the first VMA at-or-after `shmaddr`. If the found VMA's `vm_start != shmaddr` or its ops are not `shm_vm_ops`, return `EINVAL`.

REQ-4: Legacy "partial-attach" handling: when a single `shmat` originally produced multiple contiguous VMAs (rare, for very large segments interleaved with `SHM_REMAP`), `shmdt` removes all VMAs whose start-offsets lie within `[shmaddr, shmaddr + segsz)` and that map the same `shm_file`. Single-call best-effort.

REQ-5: Permission check: none. The caller's own attachment is freely detachable; no LSM hook, no cap check. (LSM hook `security_shm_shmat` only fired on attach.)

REQ-6: Unmapping uses `do_munmap`, which triggers `shm_close` on every removed shm VMA. `shm_close` is the path that decrements `shm_nattch`.

REQ-7: After the last attachment is removed:
  - `shp->shm_nattch.dec` reaches 0.
  - `shp->shm_dtime = ktime_get_real_seconds()`.
  - `shp->shm_lpid = current_tgid()`.
  - If `shp->shm_perm.mode & SHM_DEST`: `shm_destroy` is called, segment freed, tmpfs inode dropped.

REQ-8: `mmap_write_lock(current->mm)` is held during detach. The IPC `ids_rwsem` is NOT taken on the detach path; only the per-segment lock guards `shm_nattch` decrement.

REQ-9: `shmdt` is async-signal-safe per POSIX, but is NOT reentrant against `shmat` of the same segment in another thread (the mmap write-lock serializes).

REQ-10: After `shmdt`, dereferencing `shmaddr` from userspace raises `SIGSEGV`.

REQ-11: `shmdt(NULL)` returns `EINVAL` (no VMA starts at 0 in modern systems; even if one did, it would have to be shm).

REQ-12: `shmdt` does NOT modify `shm_perm.mode` bits other than the implicit `SHM_DEST` clearance via `shm_destroy` if last-attach.

REQ-13: PID-namespace and IPC-namespace lookup are not needed — the VMA itself carries the `shm_file` and `shp` back-pointer, so namespace traversal is bypassed.

REQ-14: `shmdt` after `execve(2)`: `execve` itself unmaps all VMAs, calling `shm_close` per VMA; explicit `shmdt` after `execve` is unnecessary.

## Acceptance Criteria

- [ ] AC-1: `shmdt(p)` where `p = shmat(id, NULL, 0)` returns 0.
- [ ] AC-2: After AC-1, dereferencing `p` raises `SIGSEGV`.
- [ ] AC-3: `shmdt(p + 1)` (mid-VMA): `EINVAL`.
- [ ] AC-4: `shmdt(NULL)`: `EINVAL`.
- [ ] AC-5: `shmdt(non_shm_vma_start)`: `EINVAL`.
- [ ] AC-6: `shm_nattch` decrements by 1 per successful call.
- [ ] AC-7: After last detach of a `SHM_DEST` segment, `ipcs -m` shows the id gone.
- [ ] AC-8: After last detach of a non-`SHM_DEST` segment, segment persists.
- [ ] AC-9: `shm_dtime` updated to wall-clock at last successful call.
- [ ] AC-10: `shm_lpid` updated to caller's TGID.
- [ ] AC-11: Concurrent `shmdt` on the same VMA from two threads: one succeeds (0), one fails (`EINVAL`).
- [ ] AC-12: After `execve`, no explicit `shmdt` needed; `shm_nattch` consistent.

## Architecture

```rust
#[syscall(nr = 67, abi = "sysv")]
pub fn sys_shmdt(shmaddr: UserPtr<u8>) -> isize {
    Shm::do_shmdt(shmaddr)
}
```

`Shm::do_shmdt(shmaddr) -> isize`:
1. let addr = shmaddr.addr();
2. if (addr & (PAGE_SIZE - 1)) != 0 { return Err(EINVAL).into(); }
3. let mm = current_mm();
4. let _guard = mm.mmap_write_lock();
5. /* Locate VMA whose start == addr and is a shm VMA */
6. let vma = mm.find_vma(addr).ok_or(EINVAL)?;
7. if vma.vm_start != addr { return Err(EINVAL).into(); }
8. if !Shm::is_shm_vma(&vma) { return Err(EINVAL).into(); }
9. /* Compute detach range. For multi-VMA legacy segments, walk forward
10.    while contiguous VMAs share the same shm_file. */
11. let mut end = vma.vm_end;
12. let shm_file = vma.vm_file.clone();
13. while let Some(next) = mm.next_vma(end) {
14.   if next.vm_start != end { break; }
15.   if !Arc::ptr_eq(&next.vm_file, &shm_file) { break; }
16.   if !Shm::is_shm_vma(&next) { break; }
17.   end = next.vm_end;
18. }
19. /* do_munmap triggers shm_close on each removed shm VMA */
20. let r = mm.do_munmap(addr, end - addr);
21. r.map(|_| 0).into()

`Shm::is_shm_vma(vma) -> bool`:
1. vma.vm_ops as *const _ == &shm_vm_ops as *const _

`shm_close(vma)`:                                /* VMA close hook */
1. let shp = vma.vm_file.private_data as *mut ShmIdKernel;
2. let was = shp.shm_nattch.fetch_sub(1, Ordering::SeqCst);
3. if was == 1 {
4.   shp.shm_dtime = ktime_get_real_seconds();
5.   shp.shm_lpid = current_tgid();
6.   if shp.shm_perm.mode & SHM_DEST != 0 {
7.     Shm::shm_destroy(shp);            /* free segment, drop tmpfs inode */
8.   }
9. }
```

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `addr_alignment_enforced` | INVARIANT | per-shmdt: misaligned addr ⟹ EINVAL, no mutation. |
| `vma_match_required` | INVARIANT | per-shmdt: success requires vm_start == addr ∧ vm_ops == shm_vm_ops. |
| `nattch_decrement_exactly` | INVARIANT | per-shmdt success: shm_nattch decremented by exactly the number of VMAs removed. |
| `dest_zero_destroys` | INVARIANT | per-last-detach of SHM_DEST: shm_destroy called exactly once. |
| `non_shm_vma_untouched` | INVARIANT | per-shmdt: VMAs that are not shm or have different shm_file are not removed. |
| `mmap_write_lock_held` | INVARIANT | per-shmdt: address-space mutation under mmap_write_lock. |

### Layer 2: TLA+

`ipc/shmdt.tla`:
- States: per-mm VMA list, per-segment shm_nattch, SHM_DEST flag, destruction set.
- Properties:
  - `safety_no_detach_non_shm` — `shmdt(non_shm_addr)` cannot remove any VMA.
  - `safety_nattch_matches_vmas` — `shm_nattch` always equals system-wide attach count.
  - `safety_destroy_iff_last_dest` — `shm_destroy` runs ⟺ last attach removed ∧ SHM_DEST set.
  - `safety_concurrent_winner` — concurrent `shmdt` of same VMA: exactly one returns 0.
  - `liveness_detach_terminates` — every `shmdt` returns.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_shmdt` post: success ⟹ no shm VMA starts at `addr` after call | `Shm::do_shmdt` |
| `shm_close` post: shm_nattch decremented; if 0 ∧ SHM_DEST ⟹ segment freed | `shm_close` |
| `is_shm_vma` pure: depends only on vm_ops pointer equality | `Shm::is_shm_vma` |
| `do_munmap` post (limited to shm): each removed shm VMA fires shm_close exactly once | `mm.do_munmap` |

### Layer 4: Verus / Creusot functional

Per-`shmdt(2)` man-page equivalence; LTP `shmdt01..02` pass; PostgreSQL postmaster shutdown cleanup exercises `IPC_RMID` + final `shmdt` last-detach destruction path.

## Hardening

(Inherits row-1 features from `uapi/00-overview.md` § Hardening.)

`shmdt(2)` reinforcement:

- **Per-vm_ops pointer comparison** — defense against per-spoofed-VMA detach of non-shm mapping.
- **Per-start-address strict match** — defense against per-mid-VMA partial unmap exploit.
- **Per-mmap_write_lock serialization** — defense against per-race with concurrent `shmat` / `mmap`.
- **Per-shm_nattch saturating refcount** — defense against per-underflow when over-detach is attempted concurrently.
- **Per-SHM_DEST last-attach destroy** — defense against per-leaked-segment after `IPC_RMID`.
- **Per-no LSM (by design)** — caller can always detach own attachment; no policy can deadlock teardown.

## Grsecurity / PaX surface

- **GRKERNSEC_CHROOT_SHMAT detach symmetry** — `shmdt` of an attachment installed before chroot entry is always permitted (no orphan attachments), but `shmdt` does not become a side-channel for cross-jail probing: the VMA test is pointer equality on `shm_vm_ops`, which is per-kernel-image, not per-namespace.
- **PaX UDEREF on `shmaddr`** — although `shmdt` does not copy from the address, the alignment + range checks run under UDEREF rules to ensure a kernel-range `shmaddr` cannot reach `find_vma`.
- **PAX_REFCOUNT on `shm_nattch` decrement** — saturating refcount ensures underflow panics. A doubled-detach race that would otherwise yield `shm_nattch == -1` and free-on-next-detach is caught.
- **GRKERNSEC_AUDIT_IPC** — every `shmdt` is recorded with task, uid, cap, segment id (resolved via VMA back-pointer), and detach-time-delta vs attach.
- **PaX KERNEXEC during shm_destroy** — when the last detach drops a `SHM_DEST` segment, the tmpfs inode-drop path runs under KERNEXEC W^X to defeat any post-free exec window.
- **No syscall-cap relax** — `shmdt` is unprivileged by POSIX, and grsec does not gate it; doing so could deadlock cleanup. Instead the hardening is on the create / attach side.
- **GRKERNSEC_RESLOG on last-detach destroy** — when shm_destroy fires from `shmdt`, an info-level log records the segment's lifetime, max attachers, and total bytes consumed; useful for shm-leak triage.
- **Per-`PAX_USERCOPY_HARDEN` neutral** — no userspace copies on this path; nothing to slab-whitelist.
- **No_new_privs neutral** — read-mostly path; no cred change.
- **Anti-fingerprint** — `shmdt` runs in O(log V) over the VMA tree; consistent timing not exploitable.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `shmat(2)` (Tier-5 separate doc — attachment).
- `shmget(2)` (Tier-5 separate doc — segment creation).
- `shmctl(2)` (Tier-5 separate doc — control / IPC_RMID).
- `do_munmap` general semantics (Tier-3 in `mm/mmap.md`).
- tmpfs inode lifecycle (Tier-3 in `mm/shmem.md`).
- Implementation code.
