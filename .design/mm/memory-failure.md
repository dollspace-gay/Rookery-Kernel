# Tier-3: mm/memory-failure.c — Hardware memory-failure recovery (HWPoison)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/memory-failure.c (~2970 lines)
  - mm/hwpoison-inject.c
  - include/linux/mm.h (HWPoison flags + me_*)
-->

## Summary

HWPoison handles per-page hardware memory-failures (uncorrectable ECC errors). Per-`memory_failure(pfn, ...)` invoked from MCE/EDAC handler: marks struct page PG_hwpoison; per-page-state-machine routes to per-error-class handler (anon, file-page, swap-cache, slab, hugetlb). Per-handler: `me_swapcache_dirty` / `me_pagecache_clean` / `me_pagecache_dirty` / `me_huge_page` / `me_unknown` / etc. Per-process holding poisoned page: SIGBUS via per-action-required SIGBUS-AR or queued. Per-poisoned page: removed from buddy/lru/page-cache; future allocs skip. Per-/sys/kernel/mm/hwpoison: poison-injection for testing. Critical for: server-grade RAS (reliability/availability/serviceability).

This Tier-3 covers `memory-failure.c` (~2970 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `memory_failure()` | per-PFN entry from MCE | `MemFail::memory_failure` |
| `memory_failure_queue()` | async-queue version | `MemFail::queue` |
| `me_swapcache_dirty()` / `me_swapcache_clean()` | per-swap-cache handlers | `MemFail::me_swapcache_*` |
| `me_pagecache_clean()` / `me_pagecache_dirty()` | per-page-cache handlers | `MemFail::me_pagecache_*` |
| `me_anon_dirty()` / `me_anon_clean()` | per-anon handlers | `MemFail::me_anon_*` |
| `me_huge_page()` | per-hugetlb handler | `MemFail::me_huge_page` |
| `me_unknown()` | per-default | `MemFail::me_unknown` |
| `me_kernel()` | per-kernel-page (panic) | `MemFail::me_kernel` |
| `unpoison_memory()` | per-test-recovery | `MemFail::unpoison_memory` |
| `kill_procs()` | per-poisoned-page kill holders | `MemFail::kill_procs` |
| `add_to_kill()` | per-process tracking | `MemFail::add_to_kill` |
| `task_early_kill()` / `task_action_required()` | per-policy classify | `MemFail::task_early_kill` |
| `hwpoison_filter()` | per-filter inject | `MemFail::hwpoison_filter` |
| `MF_ACTION_REQUIRED` / `MF_MUST_KILL` / `MF_COUNT_INCREASED` flags | per-flag | shared |
| `PG_hwpoison` | per-page poison flag | shared |

## Compatibility contract

REQ-1: Per-MCE entry:
- arch/x86/kernel/mce.c: per-uncorrectable error → memory_failure(pfn, MF_ACTION_REQUIRED).
- Per-page in DRAM region; per-MCE bank decoded.

REQ-2: memory_failure(pfn, flags):
- folio = pfn_folio(pfn).
- /* Page-state classify */
- if PageReserved: kernel-page; possibly panic.
- if PageHuge: me_huge_page.
- if PageSlab: me_kernel (slab-failure → panic).
- elif PageSwapCache: me_swapcache_*.
- elif PageAnon: me_anon_*.
- elif PageMlocked: me_unknown ∨ kill (mlocked can't migrate).
- elif page_mapping(folio): me_pagecache_*.
- else: me_unknown.

REQ-3: Per-handler decision:
- DELAYED: page can't be acted on now (transient state).
- IGNORED: already-handled.
- FAILED: action failed; system unstable.
- RECOVERED: page poisoned + freed.

REQ-4: Per-poisoned page:
- folio.flags |= PG_hwpoison.
- folio_set_dirty (if needed for page-cache writeback failure).
- Remove from LRU / buddy / page-cache.
- get_page_unless_zero held to keep struct page alive.

REQ-5: kill_procs (action-required):
- Per-PFN walk all rmap (anon_vma + i_mmap) → list of (task, addr).
- For each task: send SIGBUS with siginfo(BUS_MCEERR_AR or AO).
- AR (Action-Required): per-current-MCE; kill before-resume.
- AO (Action-Optional): async; may race-with-task.

REQ-6: Per-MF_ACTION_REQUIRED:
- per-MCE delivered to faulting CPU; must SIGBUS BEFORE resume.
- Per-MCE-AR: kill faulting task synchronously.

REQ-7: Per-userspace ABI:
- /sys/kernel/mm/hwpoison/{corrupt-pfn, unpoison-pfn}: testing.
- /proc/sys/vm/memory_failure_early_kill: vs late-kill.
- /proc/sys/vm/memory_failure_recovery: 0=panic, 1=recover.

REQ-8: Per-HWPoison filter:
- hwpoison_filter: per-process / per-cgroup / per-mem-policy filter at inject.
- Used by /sys/kernel/mm/hwpoison/corrupt-pfn for selective testing.

REQ-9: Per-anonymous-page recovery:
- me_anon_dirty: lose data; SIGBUS.
- me_anon_clean: drop page; remap may succeed.

REQ-10: Per-pagecache recovery:
- me_pagecache_clean: drop from page-cache.
- me_pagecache_dirty: writeback fails → keep poison; SIGBUS reader.

REQ-11: Per-swapcache recovery:
- me_swapcache_clean: drop from swap-cache.
- me_swapcache_dirty: writeback fails; SIGBUS user.

REQ-12: Per-hugetlb recovery:
- me_huge_page: per-huge-folio: drop from hugetlb-pool; SIGBUS users.

REQ-13: Per-stats:
- /proc/vmstat: pgpoisoned / pgsoftpoisoned.
- per-zone hwpoisoned counter.

## Acceptance Criteria

- [ ] AC-1: memory_failure on anon-page: PG_hwpoison set; SIGBUS to user.
- [ ] AC-2: memory_failure on swap-cache: dropped; subsequent swap-in fails with SIGBUS.
- [ ] AC-3: memory_failure on slab-page: panic (or warn).
- [ ] AC-4: memory_failure on pagecache-clean: dropped; readers get SIGBUS.
- [ ] AC-5: memory_failure on huge-page: hugetlb-pool reduced; users SIGBUS'd.
- [ ] AC-6: HWPoison-filter: per-test only PFN matching filter poisoned.
- [ ] AC-7: /sys/kernel/mm/hwpoison/corrupt-pfn write: triggers memory_failure.
- [ ] AC-8: /sys/kernel/mm/hwpoison/unpoison-pfn: clears PG_hwpoison.
- [ ] AC-9: Per-MF_ACTION_REQUIRED: faulting task SIGBUS-AR.
- [ ] AC-10: Per-poisoned PFN: alloc_pages skips.

## Architecture

Per-PFN handler dispatch table:

```
struct PageState {
  mask: u64,                                     // page-flag mask
  res: u64,                                      // expected match
  msg: &'static str,
  action: fn(folio: &Folio, pfn: u64, flags: i32) -> i32,
}

static PAGE_STATES: &[PageState] = &[
  { mask: 1<<PG_reserved, res: 1<<PG_reserved, msg: "reserved", action: me_kernel },
  { mask: 1<<PG_slab, res: 1<<PG_slab, msg: "kernel slab", action: me_kernel },
  { mask: 1<<PG_swapcache | dirty_mask, res: 1<<PG_swapcache | dirty, msg: "dirty swap-cache", action: me_swapcache_dirty },
  ...
];
```

Per-task list:

```
struct ToKill {
  list: ListLink,
  tsk: *TaskStruct,
  addr: u64,
  size_shift: u8,
}
```

`MemFail::memory_failure(pfn, flags) -> i32`:
1. /* Per-PFN validate */
2. if !pfn_valid(pfn): return -ENXIO.
3. folio = pfn_folio(pfn).
4. /* Per-flag MF_COUNT_INCREASED: caller already holds ref */
5. /* Per-class dispatch */
6. for ps in PAGE_STATES:
   - if (folio.flags & ps.mask) == ps.res:
     - return ps.action(folio, pfn, flags).
7. return MemFail::me_unknown(folio, pfn, flags).

`MemFail::me_anon_dirty(folio, pfn, flags) -> i32`:
1. /* Walk rmap; per-VMA per-task: add_to_kill */
2. unmap_success = try_to_unmap_one(folio, ...).
3. /* Lose data */
4. SetPageHWPoison(folio).
5. /* Kill all holders */
6. MemFail::kill_procs(&to_kill_list, flags).
7. return RECOVERED.

`MemFail::me_pagecache_clean(folio, pfn, flags) -> i32`:
1. truncate_inode_folio(folio.mapping, folio).
2. SetPageHWPoison(folio).
3. unmap_success = try_to_unmap_one(folio, ...).
4. return RECOVERED.

`MemFail::me_huge_page(folio, pfn, flags) -> i32`:
1. /* Hugetlb-specific cleanup */
2. SetPageHWPoison(folio).
3. dissolve_free_hugetlb_folio(folio).
4. /* Kill all huge-page holders */
5. MemFail::kill_procs.
6. return RECOVERED.

`MemFail::kill_procs(to_kill_list, flags)`:
1. for tk in to_kill_list:
   - sig = (flags & MF_ACTION_REQUIRED) ? SIGBUS_AR : SIGBUS_AO.
   - send_sig_info(SIGBUS, &siginfo, tk.tsk).

`MemFail::add_to_kill(tsk, vma, addr, &to_kill_list)`:
1. tk = kmalloc(sizeof(ToKill)).
2. tk.tsk = get_task_struct(tsk); tk.addr = addr.
3. list_add_tail(&tk.list, &to_kill_list).

`MemFail::me_kernel(folio, pfn, flags) -> i32`:
1. /* Kernel-page poisoning is fatal */
2. if !sysctl_memory_failure_recovery: panic("Hardware memory error").
3. return FAILED.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pfn_valid_pre_dispatch` | INVARIANT | per-memory_failure: pfn_valid checked. |
| `state_dispatch_terminates` | INVARIANT | per-PAGE_STATES walk: terminates with at-most-one match. |
| `pg_hwpoison_set_per_recover` | INVARIANT | per-handler success ⟹ folio.PG_hwpoison set. |
| `kill_procs_iterates_to_kill_list` | INVARIANT | per-poisoned page: all rmap-mappers killed. |
| `me_kernel_panic_or_fail` | INVARIANT | per-kernel-page poison: panic OR FAILED. |

### Layer 2: TLA+

`mm/memory_failure.tla`:
- Per-PFN poison + per-page-state-handler + per-task-kill flow.
- Properties:
  - `safety_no_user_access_post_poison` — per-PG_hwpoison ⟹ user access SIGBUSes.
  - `safety_buddy_skips_poisoned` — per-alloc: poisoned PFN never returned.
  - `liveness_action_required_killed_pre_resume` — per-MF_ACTION_REQUIRED ⟹ SIGBUS-AR before resume.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `MemFail::memory_failure` post: per-PG_hwpoison set; per-class handler dispatched | `MemFail::memory_failure` |
| `MemFail::me_anon_dirty` post: rmap-walked; kill-list populated; SIGBUS sent | `MemFail::me_anon_dirty` |
| `MemFail::me_huge_page` post: huge-folio dissolved; users SIGBUSed | `MemFail::me_huge_page` |
| `MemFail::kill_procs` post: per-tk task signaled with SIGBUS+siginfo | `MemFail::kill_procs` |

### Layer 4: Verus/Creusot functional

`Per-MCE-uncorrectable → memory_failure → per-page-class handler → poison + SIGBUS holders + drop from buddy/LRU/page-cache` semantic equivalence: per-Documentation/admin-guide/mm/hwpoison.rst.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

HWPoison-specific reinforcement:

- **Per-PG_hwpoison locked once-set** — defense against per-poison repeat-handling.
- **Per-rmap_walk before SIGBUS** — defense against per-incomplete kill leaving live mapping.
- **Per-buddy hwpoison-skip** — defense against per-alloc returning poisoned PFN.
- **Per-userspace SIGBUS_AR before resume** — defense against per-bad-data continuing.
- **Per-kernel-page panic-or-FAILED** — defense against per-kernel-corruption silent-recovery.
- **Per-action-required-flag synchronous** — defense against per-MCE async-race.
- **Per-poisoned-zone counter** — defense against per-stat invisibility.
- **Per-hwpoison_filter scoped to corrupt-pfn-test** — defense against per-test-leak.
- **Per-recovery sysctl gates panic** — defense against per-system policy mismatch.
- **Per-MF_COUNT_INCREASED ref-balance** — defense against per-PFN double-decref.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/00-overview (Tier-2)
- arch/x86/kernel/cpu/mce/ (covered separately)
- EDAC drivers (covered separately)
- Implementation code
