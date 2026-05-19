# Tier-5 syscall: migrate_pages(2) — syscall 256

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: uapi/00-overview.md
upstream-paths:
  - mm/mempolicy.c (SYSCALL_DEFINE4(migrate_pages), kernel_migrate_pages, do_migrate_pages)
  - mm/migrate.c (migrate_pages, unmap_and_move, putback_movable_pages)
  - include/uapi/linux/mempolicy.h (MPOL_MF_MOVE, MPOL_MF_MOVE_ALL)
  - arch/x86/entry/syscalls/syscall_64.tbl (256  common  migrate_pages)
-->

## Summary

`migrate_pages(2)` migrates all pages of a target process that currently reside on the nodes in `old_nodes` to the nodes in `new_nodes`. Per-page selection is policy-agnostic: every present, movable page anywhere in the target's address space is candidate. The mapping (1-to-1, paired) between old and new nodes is determined by ordering of set bits; if `|old| > |new|` excess sources are unioned, if `|new| > |old|` excess targets are unused. Returns the number of pages that could not be moved.

Critical for: `numactl --migrate`, cluster manager rebalancing, CXL-tier promotion / demotion, hot-page sweep before node maintenance.

## Signature

```c
long migrate_pages(int pid, unsigned long maxnode,
                   const unsigned long *old_nodes,
                   const unsigned long *new_nodes);
```

## Parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pid` | `int` | in | Target PID; `0` = self. Must be in the caller's PID namespace. |
| `maxnode` | `unsigned long` | in | Width of both bitmaps; one more than highest valid bit. |
| `old_nodes` | `const unsigned long *` | in | Bitmap of source nodes — pages on these nodes are migration candidates. |
| `new_nodes` | `const unsigned long *` | in | Bitmap of destination nodes; per-bit-position pair maps `old_i` to `new_i`. |

## Return value

| Value | Meaning |
|---|---|
| `>= 0` | Number of pages that **failed** to migrate (0 = full success). |
| `-1` + `errno` | Failure. |

## Errors

| errno | Trigger |
|---|---|
| `EFAULT` | `old_nodes` / `new_nodes` user pointer faults. |
| `EINVAL` | `maxnode > MAX_NUMNODES + 1`; either mask empty after intersection with `mems_allowed`. |
| `EPERM` | Target `pid` is not owned by caller and caller lacks `CAP_SYS_NICE`; or target is across user-namespace boundary. |
| `ESRCH` | `pid` does not name a valid task. |
| `ENOMEM` | No memory on any destination node; out-of-vector allocation. |
| `E2BIG` | `maxnode` exceeds compile-time bound. |
| `ENODEV` | Destination node offline / not in target's mems_allowed. |

## ABI surface

```text
__NR_migrate_pages  (x86_64)  = 256
__NR_migrate_pages  (arm64)   = 238
__NR_migrate_pages  (riscv)   = 238
__NR_migrate_pages  (i386)    = 294

/* Both bitmaps are unsigned long arrays of identical width. */
```

## Compatibility contract

REQ-1: Syscall number is **256** on x86_64. ABI-stable.

REQ-2: `pid == 0` ⟹ self-migration; `pid > 0` ⟹ migration of named task.

REQ-3: Capability gate:
- Self ⟹ no extra cap.
- Other-task in same uid ⟹ no extra cap, but **target's** mems_allowed governs destinations.
- Other-task across uid ⟹ `CAP_SYS_NICE` in target's user_ns.

REQ-4: Bitmap copy: `get_nodes` validates `maxnode`, copies, then intersects with `target->cpuset.mems_allowed`. If either intersection empty ⟹ `-EINVAL`.

REQ-5: Per-page candidacy: page must be `PageLRU` or movable; pinned, locked, kernel-bound, hugetlb-non-migratable pages skipped (counted toward "failed").

REQ-6: Mapping: bit-position `i` in old_nodes maps to bit-position `i` in new_nodes after both are compacted (set-bit-by-set-bit). If `|old| > |new|`, extra old bits map to last new bit (or are skipped per upstream); spec: kernel-defined, must be deterministic.

REQ-7: Migration uses internal `migrate_pages()` (mm/migrate.c); calls `unmap_and_move` per page; respects `PageTransHuge` (THP split if needed).

REQ-8: `MPOL_MF_MOVE_ALL` semantics applied implicitly when `CAP_SYS_NICE` held against another task (else `MPOL_MF_MOVE`).

REQ-9: Returns count of un-moveable pages (positive integer); 0 ⟹ all moved.

REQ-10: Target task is **not** stopped during migration. Migration races with running mappings: PTE-write locked per-page.

REQ-11: Per-`mmap_read_lock` held on target while walking VMAs.

REQ-12: hugetlb pages: only those whose hstate supports migration are eligible.

REQ-13: Shared anon pages: only moved when `MPOL_MF_MOVE_ALL` (cross-uid scope) honored.

REQ-14: Per-node N_MEMORY check on destinations; offline nodes refused.

REQ-15: Result accumulates per-thread; multi-threaded task migrated whole-mm (shared mm).

## Acceptance Criteria

- [ ] AC-1: `migrate_pages(0, 64, {0}, {1})` after touching pages on node 0: pages relocated to node 1 (verified via `/proc/<pid>/numa_maps`).
- [ ] AC-2: `migrate_pages(pid_other_uid, 64, {0}, {1})` without `CAP_SYS_NICE`: `-EPERM`.
- [ ] AC-3: `pid = -1`: `-EINVAL`.
- [ ] AC-4: `pid` not found: `-ESRCH`.
- [ ] AC-5: Offline destination node in mask: `-ENODEV`.
- [ ] AC-6: `maxnode = MAX_NUMNODES + 2`: `-EINVAL` / `-E2BIG`.
- [ ] AC-7: Self-migration with pinned page (mlock + GUP-pin): page counted as failed, return > 0.
- [ ] AC-8: THP migration: huge page split if dest is single-page, else moved as huge.
- [ ] AC-9: Empty new_nodes after intersection: `-EINVAL`.
- [ ] AC-10: Migration of a task across nodes succeeds with both single-bit masks; verify `move_pages(MPOL_F_NODE | MPOL_F_ADDR)` after.
- [ ] AC-11: Concurrent munmap during migration: races handled, no UAF.

## Architecture

```rust
#[syscall(nr = 256, abi = "sysv")]
pub fn sys_migrate_pages(pid: i32, maxnode: usize,
                         old: UserPtr<u64>, new: UserPtr<u64>) -> isize {
    Mempolicy::do_migrate_pages(pid, maxnode, old, new)
}
```

`Mempolicy::do_migrate_pages(pid, maxnode, old_p, new_p) -> isize`:
1. /* Resolve target task */
2. let target = if pid == 0 { current() } else {
3.   find_task_by_vpid(pid).ok_or(-ESRCH)?
4. };
5. /* Capability gate */
6. if !same_thread_group(target, current()) &&
7.    !check_same_creds_or_cap(target, CAP_SYS_NICE)? {
8.   return -EPERM;
9. }
10. /* Copy node masks */
11. let old_mask = Mempolicy::get_nodes(old_p, maxnode)?;
12. let new_mask = Mempolicy::get_nodes(new_p, maxnode)?;
13. /* Intersect with target's mems_allowed */
14. let target_mems = task_cpuset_mems(target);
15. let old = old_mask.intersect(&target_mems);
16. let new = new_mask.intersect(&target_mems);
17. if old.is_empty() || new.is_empty() { return -EINVAL; }
18. /* Build node→node mapping */
19. let map = Mempolicy::build_node_pair(&old, &new);
20. /* Walk VMAs and queue migration */
21. let mm = target.mm.clone();
22. mmap_read_lock(mm);
23. let mut failed: usize = 0;
24. for vma in mm.vmas() {
25.   failed += Mempolicy::queue_pages_for_migrate(vma, &map, MPOL_MF_MOVE | extra_cap_move_all)?;
26. }
27. mmap_read_unlock(mm);
28. /* Perform migration */
29. failed += Migrate::migrate_pages(&migration_list, alloc_dest_page, ...)?;
30. Ok(failed as isize)

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `cap_gate_cross_uid` | INVARIANT | cross-uid target ⟹ CAP_SYS_NICE in target userns. |
| `intersection_nonempty` | INVARIANT | empty intersection ⟹ EINVAL before walking pages. |
| `mmap_read_lock_balanced` | INVARIANT | balanced on all paths. |
| `failed_count_monotonic` | INVARIANT | failed count never decreases. |

### Layer 2: TLA+

`mm/mempolicy-migrate.tla`:
- States: per-resolve-task, per-cap, per-build-pair, per-queue, per-migrate.
- Properties:
  - `safety_no_cross_uid_without_cap`.
  - `safety_target_mems_respected`.
  - `safety_offline_node_refused`.
  - `liveness_migrate_pages_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_migrate_pages` post: every successfully migrated page resides on a `new` node | `Mempolicy::do_migrate_pages` |
| `build_node_pair` post: deterministic, total mapping | `Mempolicy::build_node_pair` |
| `queue_pages_for_migrate` post: ineligible pages counted | `Mempolicy::queue_pages_for_migrate` |

### Layer 4: Verus / Creusot functional

Per-`migrate_pages(2)` man-page semantic equivalence. numactl --migrate parity.

## Hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`migrate_pages(2)` reinforcement:

- **Per-cross-task CAP_SYS_NICE gate** — defense against per-cross-uid migration abuse.
- **Per-target mems_allowed intersection** — defense against per-policy bypass.
- **Per-offline-node rejection** — defense against per-stranded allocation.
- **Per-pinned-page count-as-failed** — defense against per-GUP-induced lockup.
- **Per-mmap_read_lock balanced** — defense against per-stuck-lock DoS.

## Grsecurity / PaX surface

- **PaX UDEREF on `old_nodes` / `new_nodes` copy_from_user** — defense against per-user-pointer kernel-deref bug; SMAP forced.
- **CAP_SYS_NICE in target's user_ns** — grsec requires the capability in the target's namespace, not the caller's, blocking userns-priv-escalation against init-uid processes.
- **GRKERNSEC_PROC_GETPID parity on target lookup** — `find_task_by_vpid` honors process-hiding; hidden targets return `-ESRCH` even before the cap check, eliminating the existence oracle.
- **PAX_USERCOPY_HARDEN on both bitmaps** — whitelisted slab; size strictly `maxnode`-bounded.
- **GRKERNSEC_MEM block on offline / forbidden nodes** — boot-blacklisted nodes refuse migration.
- **PAX_REFCOUNT on task refcount during migration** — defense against per-target-UAF when target exits mid-migration.
- **PaX KERNSEAL on migration-alloc callback table** — read-only post-init.
- **GRKERNSEC_RBAC migrate ACL** — subjects can be denied migrate_pages entirely.
- **Per-pinned/locked-page audit log** — failed migrations of pinned pages emit a grsec audit record (helps detect pin-based migration-blocking DoS).

## Open Questions

- (none at this Tier-5 level)

## Out of Scope

- Page-migration mechanics (Tier-3 `mm/migrate.md`).
- THP split logic (Tier-3 `mm/huge_memory.md`).
- Cpuset semantics (Tier-3 `kernel/cgroup/cpuset.md`).
- Implementation code.
