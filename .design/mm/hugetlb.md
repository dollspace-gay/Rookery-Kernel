# Tier-3: mm/hugetlb.c — Explicit hugetlbfs huge pages (2MB / 1GB pre-allocated reserve)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: mm/00-overview.md
upstream-paths:
  - mm/hugetlb.c (~7324 lines)
  - mm/hugetlb_cgroup.c
  - mm/hugetlb_vmemmap.c
  - fs/hugetlbfs/
  - include/linux/hugetlb.h
  - include/uapi/linux/mman.h (MAP_HUGETLB / MAP_HUGE_*)
-->

## Summary

hugetlb (vs THP transparent) provides **explicit, pre-allocated** huge-page reserves (2MB / 1GB). Per-system has multiple **hstates** (per-page-size). Per-`/proc/sys/vm/nr_hugepages` reserves N at boot; per-`hugepagesz=` cmdline. Userspace allocates via `mmap(MAP_HUGETLB)` or `hugetlbfs`. Per-VMA: `vm_flags & VM_HUGETLB`. Per-fault: `hugetlb_fault` allocates pre-reserved huge page (no on-demand alloc). Per-memcg: hugetlb_cgroup separately tracks. Per-vmemmap-optimization: per-huge-page metadata reduced from 64 4KB pages to 1. Critical for: HugePage-aware databases (PostgreSQL, MySQL), gigabyte-page DPDK / SPDK, predictable TLB behavior.

This Tier-3 covers `hugetlb.c` (~7324 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct hstate` | per-(page-size) state | `Hstate` |
| `hugetlb_init()` | per-module init | `Hugetlb::init` |
| `hugetlb_hstate_alloc_pages()` | per-hstate boot reserve | `Hugetlb::hstate_alloc_pages` |
| `alloc_huge_page()` | per-fault alloc | `Hugetlb::alloc_huge_page` |
| `free_huge_page()` | per-folio free | `Hugetlb::free_huge_page` |
| `hugetlb_fault()` | per-VMA-fault entry | `Hugetlb::hugetlb_fault` |
| `hugetlb_no_page()` | per-no-page-fault | `Hugetlb::no_page` |
| `hugetlb_reserve_pages()` | per-mmap reserve | `Hugetlb::reserve_pages` |
| `hugetlb_unreserve_pages()` | per-munmap unreserve | `Hugetlb::unreserve_pages` |
| `huge_pte_alloc()` | per-page-table-entry alloc | `Hugetlb::huge_pte_alloc` |
| `huge_pmd_share()` | per-VMA shared-PMD optimization | `Hugetlb::huge_pmd_share` |
| `__unmap_hugepage_range()` | per-VMA unmap | `Hugetlb::__unmap_hugepage_range` |
| `try_to_unmap_one_huge()` | per-folio unmap helper | `Hugetlb::try_to_unmap_one` |
| `MAP_HUGETLB` / `MAP_HUGE_*SHIFT` | per-mmap flag | UAPI |
| `MADV_HUGEPAGE` / `MADV_NOHUGEPAGE` | madvise (THP-only) | UAPI |
| `nr_hugepages` / `surplus_hugepages` / `nr_overcommit_hugepages` | sysfs/sysctl | UAPI |

## Compatibility contract

REQ-1: Per-hstate:
- order: page-order (e.g. 9 = 2MB, 18 = 1GB on 4K pages).
- nr_huge_pages: reserved.
- free_huge_pages: free in pool.
- surplus_huge_pages: above-reserve.
- nr_overcommit_huge_pages: max above-reserve.
- next_nid_to_alloc: per-NUMA-RR cursor.
- hugepage_freelists[N_NUMA_NODES]: per-node free.
- name: e.g. "hugepages-2048kB".

REQ-2: Per-boot reserve:
- `hugepagesz=2M default_hugepagesz=2M hugepages=512`: reserve 512 × 2MB at boot.
- `hugetlb_hstate_alloc_pages_specific_nodes`: per-NUMA-spread.

REQ-3: hugetlbfs filesystem:
- /dev/hugepages mount.
- per-file: shmem-style; backed by hugetlb pages.
- per-mmap: vma.vm_flags |= VM_HUGETLB.

REQ-4: mmap(MAP_HUGETLB):
- anon huge-mmap; default hstate or MAP_HUGE_2MB / MAP_HUGE_1GB.
- per-vma_alloc_huge_folio at fault.

REQ-5: hugetlb_fault flow:
- page-fault on VM_HUGETLB VMA.
- huge_pte_alloc: alloc PMD/PUD table entry (or shared via huge_pmd_share).
- if !huge_pte_present: hugetlb_no_page.
- alloc_huge_page from hstate.free_huge_pages.
- set_huge_pte_at.

REQ-6: alloc_huge_page:
- gfp = htlb_alloc_mask(hstate).
- folio = dequeue_hugetlb_folio_node_exact (per-NUMA).
- if !folio: dequeue_hugetlb_folio_vma (per-VMA).
- hstate.free_huge_pages--.
- mem_cgroup_charge_hugetlb (separate from regular memcg).

REQ-7: hugetlb_reserve_pages:
- Per-VMA mmap: reserve range from hstate.
- per-resv_map: track reservations per-VMA.
- Per-fork: child inherits resv_map.

REQ-8: hugetlb_unreserve_pages:
- Per-munmap: free reserved range back to hstate.

REQ-9: huge_pmd_share (PMD-share for shmem):
- Per-shared-mapping: multiple VMAs share same PMD-table → 1 PMD covers 512×2MB = 1GB.
- mm.nr_pmds reduced.

REQ-10: hugetlb_vmemmap optimization:
- Per-huge-page has 64 (4KB) struct page metadata.
- Optimization: collapse identical metadata pages → 1 page reference.
- Per-CONFIG_HUGETLB_PAGE_OPTIMIZE_VMEMMAP.

REQ-11: hugetlb_cgroup:
- Per-cgroup-v1 hugetlb.usage_in_bytes / hugetlb.limit_in_bytes.
- Per-cgroup-v2 hugetlb.<size>.{current, max, events}.

REQ-12: Per-userspace ABI:
- /proc/sys/vm/nr_hugepages_mempolicy.
- /sys/kernel/mm/hugepages/hugepages-NkB/{nr_hugepages, free_hugepages, resv_hugepages, surplus_hugepages, nr_overcommit_hugepages}.
- /proc/meminfo: HugePages_*, Hugetlb.

## Acceptance Criteria

- [ ] AC-1: Boot with hugepages=512: 512 huge-pages reserved.
- [ ] AC-2: mmap(MAP_HUGETLB | MAP_ANONYMOUS, 4MB): 2 × 2MB pages allocated.
- [ ] AC-3: hugetlbfs mount + write 4MB file: huge-pages allocated.
- [ ] AC-4: free_huge_pages decreases on fault.
- [ ] AC-5: munmap returns pages: free_huge_pages increases.
- [ ] AC-6: Pool exhausted: alloc returns -ENOMEM.
- [ ] AC-7: NUMA: per-node freelist; alloc preferred-node.
- [ ] AC-8: 1GB pages: MAP_HUGE_1GB allocates 1GB folios.
- [ ] AC-9: hugetlb_cgroup limits: per-cgroup huge-page count enforced.
- [ ] AC-10: vmemmap optimization: per-huge-page metadata reduced.
- [ ] AC-11: surplus_hugepages: dynamic alloc above nr_hugepages.

## Architecture

Per-hstate:

```
struct Hstate {
  next_nid_to_alloc: i32,
  next_nid_to_free: i32,
  order: u32,                                    // page-order
  mask: u64,
  max_huge_pages: u64,
  nr_huge_pages: u64,
  free_huge_pages: u64,
  resv_huge_pages: u64,
  surplus_huge_pages: u64,
  nr_overcommit_huge_pages: u64,
  hugepage_freelists: [ListHead<Folio>; MAX_NUMNODES],
  hugepage_activelist: ListHead<Folio>,
  free_huge_pages_node: [u64; MAX_NUMNODES],
  surplus_huge_pages_node: [u64; MAX_NUMNODES],
  cgroup_files: ...,
  name: String,                                  // e.g. "hugepages-2048kB"
}
```

Per-VMA reservation map:

```
struct ResvMap {
  refs: AtomicI64,
  region_cache_count: i32,
  region_cache: ListHead<FileRegion>,
  regions: ListHead<FileRegion>,
  ...
}

struct FileRegion {
  link: ListLink,
  from: u64,                                     // start (in pages)
  to: u64,                                        // end
}
```

`Hugetlb::hstate_alloc_pages(h)`:
1. /* Per-NUMA round-robin */
2. for nid in 0..MAX_NUMNODES:
   - allocated += hugetlb_hstate_alloc_pages_onenode(h, nid).
3. /* Update freelists */

`Hugetlb::alloc_huge_page(vma, addr, avoid_reserve) -> Folio`:
1. h = hstate_vma(vma).
2. ret = vma_needs_reservation(h, vma, addr).
3. /* Try per-NUMA first */
4. folio = dequeue_hugetlb_folio_vma(h, vma, addr, avoid_reserve, &gbl_chg).
5. If !folio: alloc_buddy_hugetlb_folio_with_mpol (surplus).
6. mem_cgroup_charge_hugetlb (separate cgroup).
7. h.free_huge_pages--.
8. Return folio.

`Hugetlb::hugetlb_fault(mm, vma, addr, flags) -> VmFault`:
1. h = hstate_vma(vma).
2. /* Per-fault mutex_lock for serialization */
3. mutex_lock(&hugetlb_fault_mutex_table[hash]).
4. ptep = huge_pte_alloc(mm, vma, addr & h.mask, sz).
5. if !huge_pte_present(*ptep):
   - vmf = hugetlb_no_page(mm, vma, mapping, idx, addr, ptep, flags).
6. else if pte_write_protected ∧ flags & FAULT_FLAG_WRITE:
   - vmf = hugetlb_wp(mm, vma, addr, ptep, flags, pagecache_folio, ...).
7. mutex_unlock.

`Hugetlb::hugetlb_no_page(mm, vma, mapping, idx, addr, ptep, flags) -> VmFault`:
1. /* Try page cache */
2. folio = filemap_lock_folio(mapping, idx).
3. if !folio:
   - folio = Hugetlb::alloc_huge_page(vma, addr, 0).
   - if !folio: return VM_FAULT_OOM.
   - clear_huge_page(folio, addr, hstate_size_in_pages(h)).
4. set_huge_pte_at(mm, addr, ptep, mk_huge_pte(folio, vma.vm_page_prot), sz).

`Hugetlb::reserve_pages(inode, from, to, vma, vm_flags) -> i64`:
1. resv_map = inode_resv_map(inode) ∨ vma_resv_map(vma).
2. chg = region_chg(resv_map, from, to).
3. if chg < 0: return -ENOMEM.
4. /* Reserve from h.free_huge_pages */
5. h.resv_huge_pages += chg.

`Hugetlb::free_huge_page(folio)`:
1. h = folio_hstate(folio).
2. enqueue_hugetlb_folio(h, folio).
3. h.free_huge_pages++.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `free_pages_le_total` | INVARIANT | h.free_huge_pages ≤ h.nr_huge_pages + h.surplus_huge_pages. |
| `resv_le_free_or_surplus` | INVARIANT | h.resv_huge_pages ≤ h.free_huge_pages. |
| `surplus_le_overcommit` | INVARIANT | h.surplus_huge_pages ≤ h.nr_overcommit_huge_pages. |
| `vma_VM_HUGETLB_per_hugetlb_fault` | INVARIANT | hugetlb_fault only called for VM_HUGETLB. |
| `huge_pte_at_aligned` | INVARIANT | per-set_huge_pte_at: addr aligned to hstate-page-size. |

### Layer 2: TLA+

`mm/hugetlb.tla`:
- Per-hstate alloc/free + per-VMA reserve + per-fault no_page.
- Properties:
  - `safety_no_double_alloc` — per-folio alloc'd ⟹ on activelist; not on freelist.
  - `safety_reserve_consumes_free` — per-mmap reserve_pages: free_huge_pages-resv == surplus.
  - `liveness_munmap_eventually_returns` — per-munmap ⟹ free_huge_pages++.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Hugetlb::alloc_huge_page` post: folio on h.hugepage_activelist; freelist count-- | `Hugetlb::alloc_huge_page` |
| `Hugetlb::free_huge_page` post: folio on freelist[nid]; count++ | `Hugetlb::free_huge_page` |
| `Hugetlb::hugetlb_fault` post: PMD/PTE installed iff alloc succeeded | `Hugetlb::hugetlb_fault` |
| `Hugetlb::reserve_pages` post: resv_map updated; resv_huge_pages incremented | `Hugetlb::reserve_pages` |

### Layer 4: Verus/Creusot functional

`Per-hugetlbfs / MAP_HUGETLB userspace alloc → pre-reserved huge folio installed → giant TLB-friendly mapping` semantic equivalence: per-Documentation/admin-guide/mm/hugetlbpage.rst.

## Hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

Hugetlb-specific reinforcement:

- **Per-hstate freelist atomic** — defense against per-alloc race.
- **Per-fault hugetlb_fault_mutex_table[hash]** — defense against per-page concurrent-fault races.
- **Per-VMA resv_map refcount** — defense against per-VMA UAF.
- **Per-mem-cgroup-hugetlb separate** — defense against per-memcg double-account.
- **Per-vmemmap optimization gated** — defense against per-metadata-reduce torn mapping.
- **Per-surplus capped by overcommit** — defense against per-system unbounded huge-page alloc.
- **Per-NUMA freelist preserves locality** — defense against per-cross-node thrash.
- **Per-1GB alloc requires CMA or contig** — defense against per-fragmentation alloc-fail without infrastructure.
- **Per-huge_pmd_share for shmem** — defense against per-VMA-table memory amplification.
- **Per-MAP_HUGETLB with non-2MB-aligned size: rejected** — defense against per-config invalid.
- **Per-cgroup hugetlb.max enforces** — defense against per-cgroup unbounded.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- mm/00-overview (Tier-2)
- THP (covered in `thp.md` Tier-3)
- hugetlbfs filesystem (covered separately if expanded)
- hugetlb_cgroup (covered with kernel/cgroup/ separately)
- hugetlb_vmemmap (covered separately if expanded)
- Implementation code
