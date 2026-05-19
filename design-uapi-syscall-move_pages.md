---
title: "Tier-5 syscall: move_pages(2) — syscall 279"
tags: ["tier-5", "uapi", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-19
updated: 2026-05-19
---


## Design Specification

### Summary

`move_pages(2)` is the per-page, address-driven NUMA migration interface. Given a target PID, an array of `count` user-space virtual addresses (`pages[]`), and a parallel array of destination node ids (`nodes[]`), the kernel migrates each addressed page to its target node. If `nodes == NULL`, the syscall is a *query* — it writes the current resident node (or negative errno) for each address into `status[]` without migrating.

Critical for: numactl page-by-page placement, NUMA introspection tools, hot-page promotion in user-driven CXL tiering, tcmalloc/jemalloc page-locality tooling.

### Acceptance Criteria

- [ ] AC-1: `move_pages(0, 1, [page], [1], status, MPOL_MF_MOVE)`: page moves to node 1; `status[0] == 1`.
- [ ] AC-2: `move_pages(0, 1, [page], NULL, status, 0)`: query; `status[0]` is the resident node.
- [ ] AC-3: `nodes[0] = MAX_NUMNODES`: `status[0] = -EINVAL`.
- [ ] AC-4: `nodes[0] = offline node`: `status[0] = -EACCES`.
- [ ] AC-5: Cross-uid target without CAP_SYS_NICE: `-EPERM`.
- [ ] AC-6: `MPOL_MF_MOVE_ALL` without CAP_SYS_NICE: `-EPERM`.
- [ ] AC-7: Page not present (never touched): `status[i] = -ENOENT`.
- [ ] AC-8: Pinned page (GUP-pinned, mlock'd): `status[i] = -EBUSY`.
- [ ] AC-9: `count = 0`: returns 0, nothing written.
- [ ] AC-10: `count > PAGE_LIST_MAX`: `-E2BIG`.
- [ ] AC-11: `pages` ptr faults at index `i`: `-EFAULT` overall.
- [ ] AC-12: `nodes[i] = -1`: entry skipped, status[i] = resident node (query) or unchanged.

### Architecture

```rust
#[syscall(nr = 279, abi = "sysv")]
pub fn sys_move_pages(pid: i32, count: usize,
                      pages: UserPtr<UserPtr<u8>>,
                      nodes: UserPtr<i32>,
                      status: UserPtr<i32>,
                      flags: i32) -> isize {
    Migrate::do_move_pages(pid, count, pages, nodes, status, flags)
}
```

`Migrate::do_move_pages(pid, count, pages_p, nodes_p, status_p, flags) -> isize`:
1. /* Bounds */
2. if flags & !(MPOL_MF_MOVE | MPOL_MF_MOVE_ALL) != 0: return -EINVAL;
3. if (flags & MPOL_MF_MOVE_ALL) && !cap(CAP_SYS_NICE): return -EPERM;
4. if count == 0: return 0;
5. if count > PAGE_LIST_MAX: return -E2BIG;
6. /* Resolve target */
7. let target = resolve_pid(pid)?;
8. if !same_thread_group(target, current()) && !check_creds_or_cap(target, CAP_SYS_NICE)? {
9.   return -EPERM;
10. }
11. /* Query vs migrate */
12. if nodes_p.is_null() {
13.   return Migrate::do_pages_stat(target, count, pages_p, status_p);
14. }
15. /* Migrate path: chunked to avoid huge kmalloc */
16. const CHUNK: usize = 16 * PAGE_SIZE / size_of::<MigratePage>();
17. let mut start = 0;
18. while start < count {
19.   let n = min(CHUNK, count - start);
20.   let result = Migrate::move_chunk(target, start, n, pages_p, nodes_p, status_p, flags)?;
21.   start += n;
22. }
23. Ok(0)

`Migrate::move_chunk(target, off, n, pages_p, nodes_p, status_p, flags)`:
1. /* Copy chunk */
2. let pages = pages_p.add(off).copy_array_in(n)?;
3. let nodes = nodes_p.add(off).copy_array_in(n)?;
4. let mut status_out: Vec<i32> = vec![0; n];
5. mmap_read_lock(target.mm);
6. for i in 0..n {
7.   if nodes[i] == -1 { status_out[i] = lookup_node(target.mm, pages[i]); continue; }
8.   if nodes[i] < 0 || nodes[i] as usize >= MAX_NUMNODES {
9.     status_out[i] = -EINVAL; continue;
10.   }
11.   if !node_state(nodes[i] as usize, N_MEMORY) {
12.     status_out[i] = -EACCES; continue;
13.   }
14.   add_page_for_migration(target.mm, pages[i], nodes[i], &mut migration_list, flags, &mut status_out[i]);
15. }
16. mmap_read_unlock(target.mm);
17. /* Perform migration */
18. for nid in 0..MAX_NUMNODES { migrate_pages(&list[nid], alloc, nid); }
19. status_p.add(off).copy_array_out(&status_out)?;

### Out of Scope

- Page-migration mechanics (Tier-3 `mm/migrate.md`).
- Hugetlb migration (Tier-3 `mm/hugetlb.md`).
- Implementation code.

### signature

```c
long move_pages(int pid, unsigned long count, void **pages,
                const int *nodes, int *status, int flags);
```

### parameters

| Name | Type | Direction | Description |
|---|---|---|---|
| `pid` | `int` | in | Target task; `0` = self. |
| `count` | `unsigned long` | in | Number of entries in each array. |
| `pages` | `void **` | in | Array of user-space VAs (page-aligned recommended; kernel rounds down). |
| `nodes` | `const int *` | in | Array of target node ids; `nodes[i] == -1` skips entry; `NULL` ⟹ query mode. |
| `status` | `int *` | out | Array receiving per-page result: target node id on success, negative errno on failure. |
| `flags` | `int` | in | `MPOL_MF_MOVE` (default) / `MPOL_MF_MOVE_ALL` (requires `CAP_SYS_NICE`). |

### return value

| Value | Meaning |
|---|---|
| `0` | All entries processed (per-entry success/failure in `status[]`). |
| `-1` + `errno` | Whole-syscall failure (validation, OOM, ESRCH, etc.). |

### errors

| errno (top-level) | Trigger |
|---|---|
| `EFAULT` | `pages` / `nodes` / `status` user pointer faults. |
| `EINVAL` | `flags` invalid; `count == 0` with non-NULL nodes; node id out of range. |
| `EPERM` | Cross-task with no `CAP_SYS_NICE`; `MOVE_ALL` without cap. |
| `ESRCH` | `pid` not found. |
| `ENOMEM` | Internal allocation; per-page ENOMEM goes to `status[i]`. |
| `E2BIG` | `count` exceeds kernel arg cap. |

| Per-entry status[i] | Trigger |
|---|---|
| `>= 0` | Resident node id (query) or new node id (migration). |
| `-EACCES` | Page mapped from a non-movable region. |
| `-EFAULT` | `pages[i]` not in a VMA. |
| `-ENOENT` | Page not present (not faulted in). |
| `-EBUSY` | Page locked / pinned / under writeback. |
| `-EINVAL` | `nodes[i]` invalid. |
| `-ENOMEM` | Target node OOM. |

### abi surface

```text
__NR_move_pages  (x86_64)  = 279
__NR_move_pages  (arm64)   = 239
__NR_move_pages  (riscv)   = 239
__NR_move_pages  (i386)    = 317

/* `int nodes[]` is signed 32-bit; -1 means "skip"; valid range [0, MAX_NUMNODES). */
/* `void *pages[]` width follows the calling ABI (8 on x86_64, 4 on i386). */
```

### compatibility contract

REQ-1: Syscall number is **279** on x86_64. ABI-stable.

REQ-2: `pid == 0` ⟹ self; `pid > 0` ⟹ target task.

REQ-3: Capability:
- Self: no extra cap.
- Other task, same realuid: no extra cap, target's mems_allowed governs.
- Other task, different uid: `CAP_SYS_NICE` in target's user_ns.
- `MPOL_MF_MOVE_ALL`: always requires `CAP_SYS_NICE`.

REQ-4: `count` upper bound: implementation-defined (`PAGE_LIST_MAX = 65536` typical); above ⟹ `-E2BIG`.

REQ-5: If `nodes == NULL`: status query path (`do_pages_stat`). No migration occurs.

REQ-6: If `nodes != NULL`: per-entry migration via `add_page_for_migration` + `migrate_pages` batch.

REQ-7: Per-`nodes[i]`: must be `< MAX_NUMNODES`, online with `N_MEMORY`, and ∈ target's mems_allowed (else `status[i] = -EACCES`).

REQ-8: Per-page lookup: `find_vma` + `follow_page`; page must be present (not on swap). Swapped-out ⟹ `status[i] = -ENOENT`.

REQ-9: Per-page eligibility: must be movable (LRU, anon, or page-cache); shared kernel pages ⟹ `-EACCES`.

REQ-10: Per-page result is written to `status[i]` before returning.

REQ-11: Atomicity: per-page migration is per-entry; partial success allowed. The syscall returns 0 if all entries were *processed*, regardless of per-entry success.

REQ-12: Page addresses rounded down to `PAGE_SIZE`; THP page treated as the head page.

REQ-13: `mmap_read_lock` on target for VMA lookup; released before migration phase.

REQ-14: Migration phase is preemptible; can block.

REQ-15: hugetlb pages: only if `pmd_huge_movable` for the hstate; else `-EBUSY`.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `chunk_arrays_bounded` | INVARIANT | every chunk ≤ count - off. |
| `cap_gate_cross_uid` | INVARIANT | cross-uid target ⟹ CAP_SYS_NICE. |
| `status_written_for_each_processed` | INVARIANT | status[i] written for 0..count (if no top-level EFAULT). |
| `mmap_read_lock_balanced` | INVARIANT | balanced on all paths. |
| `nodes_null_no_migration` | INVARIANT | nodes==NULL ⟹ no page migrated. |

### Layer 2: TLA+

`mm/move-pages.tla`:
- States: per-validate, per-resolve-target, per-chunk-copy, per-page-classify, per-migrate.
- Properties:
  - `safety_no_migration_in_query`.
  - `safety_per_page_classification_total`.
  - `safety_cross_uid_requires_cap`.
  - `liveness_move_pages_terminates`.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `do_move_pages` post: status[0..count] populated; 0 returned ∨ -errno | `Migrate::do_move_pages` |
| `move_chunk` post: status_out[i] in expected enum | `Migrate::move_chunk` |
| `do_pages_stat` post: no VMA mutation | `Migrate::do_pages_stat` |

### Layer 4: Verus / Creusot functional

Per-`move_pages(2)` man-page semantic equivalence. numactl test-suite parity.

### hardening

(Inherits row-1 from `uapi/00-overview.md` § Hardening.)

`move_pages(2)` reinforcement:

- **Per-`MOVE_ALL` CAP_SYS_NICE gate** — defense against per-cross-process migration abuse.
- **Per-chunked array copy** — defense against per-huge-kmalloc DoS.
- **Per-page status-always-written** — defense against per-stale-status info-leak.
- **Per-pinned-page reported as EBUSY** — defense against per-GUP-induced kernel block.
- **Per-target mems_allowed intersection** — defense against per-policy bypass.

### grsecurity / pax surface

- **PaX UDEREF on `pages` / `nodes` / `status` copy_*_user** — defense against per-user-pointer kernel-deref bug; SMAP forced.
- **GRKERNSEC_PROC_GETPID against move_pages info-leak** — query mode (`nodes == NULL`) can be used to discover NUMA placement of another process's address space, providing a probe of ASLR layout via reachable VMAs. Grsec gates query mode to either the same task, same realuid, or `PTRACE_MODE_READ_REALCREDS`; hidden processes return `-ESRCH` before any status[] write to avoid an existence oracle.
- **CAP_SYS_NICE in target's user_ns** — required for cross-uid migration; root-in-non-init-userns insufficient.
- **PAX_USERCOPY_HARDEN on all three arrays** — whitelisted slab; per-chunk size capped.
- **GRKERNSEC_MEM block on offline/forbidden nodes** — refuses node ids in boot blacklist.
- **PAX_REFCOUNT on target task during the syscall** — defense against per-target-UAF when target exits mid-call.
- **PaX KERNSEAL on `migrate_target_node` callback table** — read-only post-init.
- **GRKERNSEC_RBAC move_pages ACL** — subjects can be denied move_pages entirely (sandboxed renderers).
- **Per-`PAGE_LIST_MAX` lower in hardened mode** — grsec lowers the cap to reduce burst-allocation DoS surface.
- **Per-query-mode rate limit** — defense against per-probing leak of resident-node info from other tasks.

