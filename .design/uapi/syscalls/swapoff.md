# Tier-5 syscall: swapoff(2) — syscall 168

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/swapfile.c (SYSCALL_DEFINE1(swapoff), __do_sys_swapoff, try_to_unuse)
  - mm/swapfile.c (unuse_mm, unuse_pte_range, swap_remove_full_pages)
  - include/uapi/linux/swap.h (struct swap_info_struct flags)
  - arch/x86/entry/syscalls/syscall_64.tbl (168  common  swapoff)
-->

## Summary

`swapoff(2)` deactivates a swap area. The kernel first marks the `swap_info_struct` as draining (no new allocations), then walks every process's mm and every page-cache anonymous page that references slots in this swap area and **swaps them back in** (`try_to_unuse`). Only after all references are gone is the device closed and the slot freed. This can take minutes on a busy machine and may fail with `-ENOMEM` if there isn't enough RAM to absorb the swapped-out pages.

Critical for: clean shutdown, swap-device replacement, hibernation finalization, container teardown.

## Signature

```c
int swapoff(const char *path);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `path` | `const char *` | in | Path used previously with `swapon`; resolved in current mount namespace. |

## Return value

| Value | Meaning |
|---|---|
| `0` | Swap area drained and offlined. |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EPERM` | Caller lacks `CAP_SYS_ADMIN`. |
| `EFAULT` | `path` user pointer faults. |
| `ENOENT` | Path not currently active as a swap area. |
| `EINVAL` | `path` resolves but is not registered. |
| `ENOMEM` | Insufficient RAM to swap-in pages; partial drain rolled back. |
| `EBUSY` | Another swapoff already draining this device (race). |

## ABI surface

```text
__NR_swapoff  (x86_64)  = 168
__NR_swapoff  (arm64)   = 225
__NR_swapoff  (riscv)   = 225
__NR_swapoff  (i386)    = 115
```

## Compatibility contract

REQ-1: Syscall number is **168** on x86_64. ABI-stable.

REQ-2: Capability: `CAP_SYS_ADMIN` in init userns.

REQ-3: `path` resolved via standard `namei`. Matches against `swap_info[i].swap_file` (inode-equality and bdev-equality both checked).

REQ-4: No matching entry ⟹ `-EINVAL` (or `-ENOENT` if path doesn't resolve).

REQ-5: Per-`swap_info.flags |= SWP_DRAIN`: marks not-allocatable; removed from `swap_avail_heads`.

REQ-6: `try_to_unuse(type)` loop:
- For each used slot, find every mm referencing it via `swap_info.swap_map[]` (per-slot refcount).
- Page-fault-style swap-in via `read_swap_cache_async` + `do_swap_page`.
- Update PTEs to point to anonymous page; drop swap slot ref.
- Repeat until `swap_info.inuse_pages == 0` or `-ENOMEM`.

REQ-7: On `-ENOMEM`: swap area returned to `SWP_USED` state (re-enabled); caller must retry.

REQ-8: Per-`MMF_VM_HUGEPAGE` / `MMF_HAS_PINNED` / locked-task signals respected during walk (yields).

REQ-9: On success: close file (`filp_close`), release `bd_claim`, free `swap_map[]`, free `swap_extents`, return slot to free pool.

REQ-10: `/proc/swaps` entry removed only after successful completion.

REQ-11: Operation is interruptible by `SIGKILL` to the calling thread; partial swap-in already done is committed (pages re-resident).

REQ-12: Concurrent `swapoff` of same path: `-EBUSY`.

REQ-13: Pages re-swap-in obey existing mempolicy of the receiving VMA — they may land on a specific node.

REQ-14: Hibernation: special early-shutdown path bypasses `try_to_unuse` and assumes mm already frozen.

REQ-15: After successful return, the file/device can be re-used (e.g. `mkswap` again).

## Acceptance Criteria

- [ ] AC-1: `swapoff("/dev/sdb1")` after `swapon`: returns 0; `/proc/swaps` no longer lists it.
- [ ] AC-2: Non-root caller: `-EPERM`.
- [ ] AC-3: Path not active: `-EINVAL`.
- [ ] AC-4: Path nonexistent: `-ENOENT`.
- [ ] AC-5: Swap-out of 1 GiB, swapoff with only 256 MiB free RAM: `-ENOMEM`, area re-enabled.
- [ ] AC-6: Concurrent swapoff on same path: one succeeds, the other `-EBUSY` or `-EINVAL` once first wins.
- [ ] AC-7: Process killed during swapoff: pages already swapped-in remain resident; caller observes signal-based exit.
- [ ] AC-8: After successful swapoff, `swapon` of the same path succeeds again.
- [ ] AC-9: All anonymous pages re-resident on success (verified via `/proc/<pid>/smaps` Swap=0 for affected mappings).
- [ ] AC-10: hibernation finalization path bypasses normal drain.

## Architecture

```rust
#[syscall(nr = 168, abi = "sysv")]
pub fn sys_swapoff(path: UserPtr<u8>) -> isize {
    Swap::do_swapoff(path)
}
```

`Swap::do_swapoff(path_uptr) -> isize`:
1. if !cap(CAP_SYS_ADMIN, init_user_ns()): return -EPERM;
2. let pathname = Fs::getname(path_uptr)?;
3. let inode = Fs::lookup_inode(&pathname).ok_or(-ENOENT)?;
4. let p = Swap::find_swap_info_by_inode_or_bdev(inode).ok_or(-EINVAL)?;
5. /* Race guard */
6. swap_lock();
7. if p.flags & SWP_DRAIN != 0 { swap_unlock(); return -EBUSY; }
8. p.flags |= SWP_DRAIN;
9. Swap::remove_from_avail_heads(p);
10. swap_unlock();
11. /* Drain */
12. let err = Swap::try_to_unuse(p.type_id);
13. if let Err(e) = err {
14.    /* Rollback: restore as active */
15.    swap_lock();
16.    p.flags &= !SWP_DRAIN;
17.    Swap::reinsert_avail_heads(p);
18.    swap_unlock();
19.    return e;
20. }
21. /* Final teardown */
22. Swap::release_swap_info(p);
23. Ok(0)

`Swap::try_to_unuse(type) -> Result<()>`:
1. while p.inuse_pages > 0 {
2.   /* Iterate processes */
3.   for_each_process(task) {
4.     mmap_read_lock(task.mm);
5.     Swap::unuse_mm(task.mm, type, &p.swap_map)?;
6.     mmap_read_unlock(task.mm);
7.     /* Yield if reschedule needed */
8.     if cond_resched() { /* ... */ }
9.   }
10.  /* Also page-cache anonymous (shmem) */
11.  Swap::unuse_shmem(type)?;
12.  if signal_pending(current()) { return Err(EINTR_OR_PARTIAL); }
13.  if out_of_memory_check()? { return Err(ENOMEM); }
14. }
15. Ok(())

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_sys_admin_init_ns` | INVARIANT | non-init userns root rejected. |
| `drain_flag_idempotent` | INVARIANT | concurrent swapoff observes EBUSY. |
| `rollback_on_oom` | INVARIANT | ENOMEM ⟹ swap_info restored to active. |
| `inuse_zero_before_release` | INVARIANT | release_swap_info called only when inuse_pages == 0. |

### Layer 2: TLA+

`mm/swapoff.tla`:
- States: per-cap, per-find, per-drain-mark, per-try-to-unuse, per-rollback-or-release.
- Properties:
  - `safety_no_release_with_inuse`.
  - `safety_rollback_restores_state`.
  - `safety_concurrent_swapoff_serializes`.
  - `liveness_swapoff_terminates` (under memory availability).

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_swapoff` post: success ⟹ swap_info removed from arrays | `Swap::do_swapoff` |
| `try_to_unuse` post: success ⟹ inuse_pages == 0 | `Swap::try_to_unuse` |
| `release_swap_info` pre: inuse_pages == 0 | `Swap::release_swap_info` |

### Layer 4: Verus / Creusot functional

Per-`swapoff(2)` man-page semantic equivalence. util-linux `swapoff` parity.

## Hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`swapoff(2)` reinforcement:

- **Per-CAP_SYS_ADMIN init-userns gate** — defense against per-userns swap-control escape.
- **Per-SWP_DRAIN exclusive flag** — defense against per-concurrent-swapoff UAF.
- **Per-ENOMEM rollback** — defense against per-half-offlined swap state.
- **Per-inuse_pages-zero precondition** — defense against per-premature-release UAF.
- **Per-signal_pending observed** — defense against per-uninterruptible-swapoff DoS.

## Grsecurity / PaX surface

- **PaX UDEREF on `path` getname** — defense against per-user-pointer kernel-deref bug; SMAP forced.
- **CAP_SYS_ADMIN in init_user_ns mandatory** — grsec forbids `swapoff` from non-init user namespaces.
- **GRKERNSEC_CHROOT_NO_SWAP** — chrooted processes cannot call `swapoff` either, even with `CAP_SYS_ADMIN`.
- **GRKERNSEC_AUDIT_SWAPON** — every `swapoff` emits an audit record (path, drain duration, success).
- **PAX_REFCOUNT on swap_info refcount** — defense against per-double-swapoff UAF and per-stale-pointer release.
- **PaX KERNSEAL on `swap_info` dispatch table** — read-only post-init.
- **GRKERNSEC_RBAC swapoff ACL** — subjects can be denied this syscall even with caps held.
- **Per-throttling of repeated swapoff/swapon cycles** — grsec rate-limits transitions to prevent a per-thrash DoS (forcing repeated try_to_unuse + setup_swap_extents).
- **Per-OOM hardening interaction** — grsec enforces that a swapoff returning `-ENOMEM` does not cause the OOM-killer to target the calling process via inflated heuristics.
- **Per-hibernation-bypass guard** — the hibernation early-shutdown bypass of `try_to_unuse` only permitted when `pm_freezing` is genuinely set; otherwise normal drain.

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- `try_to_unuse` inner mechanics (Tier-3 `mm/swap.md`).
- Hibernation suspend-to-disk (Tier-3 `kernel/power/hibernate.md`).
- zswap interaction during drain (Tier-3 `mm/zswap.md`).
- Implementation code.
