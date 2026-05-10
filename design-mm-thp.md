---
title: "Tier-3: mm/huge_memory.c — Transparent Huge Pages (THP) — anonymous + file/shmem PMD-mapped huge folios"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

Transparent Huge Pages (THP) backs anonymous + shmem + file-tmpfs VMAs with **PMD-sized (2MB) huge folios** transparently — no userspace API changes. Per-fault allocation: try PMD-fold via `do_huge_pmd_anonymous_page` (anon) / `do_huge_pmd_pagecache` (file/shmem). Per-VMA `THP_FLAG`: `always` / `madvise` / `defer` / `never`. Per-khugepaged background daemon scans VMAs to **collapse** 4KB-mapped runs into PMD-mapped huge folios. Per-split: `split_huge_page` for swap/migration/COW. Critical for: TLB-pressure reduction; large-page benefit on multi-GB workloads (databases, HPC); part of multi-size THP (mTHP) for any folio order.

This Tier-3 covers `huge_memory.c` (~5093 lines).

### Acceptance Criteria

- [ ] AC-1: malloc 2MB + read: do_huge_pmd_anonymous_page allocs PMD-folio.
- [ ] AC-2: madvise(MADV_HUGEPAGE) + populated: VMA huge-promoted via khugepaged.
- [ ] AC-3: COW on huge: do_huge_pmd_wp_page → either keep huge or split.
- [ ] AC-4: swap-out huge: split_huge_page → 512 × 4KB swap entries.
- [ ] AC-5: migrate huge: split_huge_pmd → migrate per-page → re-collapse.
- [ ] AC-6: /sys/.../enabled = madvise: only MADV_HUGEPAGE VMAs get THP.
- [ ] AC-7: khugepaged_collapse: 4KB-mapped run → PMD-folio.
- [ ] AC-8: shmem THP: tmpfs-backed VMA gets PMD folios.
- [ ] AC-9: pmd_trans_huge(pmd): returns true for huge-PMD.
- [ ] AC-10: /proc/<pid>/smaps THPeligible / AnonHugePages reported.

### Architecture

Per-VMA state (subset):

```
struct VmAreaStruct {
  ...
  vm_flags: { VM_HUGEPAGE, VM_NOHUGEPAGE, ... },
}
```

Per-mm:

```
struct MmStruct {
  ...
  def_flags: u64,                                // VM_HUGEPAGE in mm-default
  flags: MmFlags,                                // MMF_DISABLE_THP, ...
}
```

Globals:

```
static THP_FLAGS: AtomicU64;                      // transparent_hugepage_flags
const HPAGE_PMD_ORDER: u8 = 9;                    // 2MB on x86_64 4K pages
const HPAGE_PMD_SIZE: u64 = 1 << 21;
```

`Thp::vma_allowed(vma) -> bool`:
1. If !(vma.vm_flags & VM_HUGEPAGE) ∧ !(thp_flags & ALWAYS): return false.
2. If vma.vm_flags & VM_NOHUGEPAGE: return false.
3. If !is_anon_or_shmem(vma): return false.
4. If vma.vm_start | vma.vm_end & ~HPAGE_PMD_MASK: return false.
5. Return true.

`Thp::do_huge_pmd_anonymous_page(vmf) -> VmFault`:
1. vma = vmf.vma.
2. if !Thp::vma_allowed(vma): return VM_FAULT_FALLBACK.
3. gfp = vma_thp_gfp_mask(vma).
4. folio = vma_alloc_folio(gfp, HPAGE_PMD_ORDER, vma, vmf.address & HPAGE_PMD_MASK).
5. if !folio: return VM_FAULT_FALLBACK.
6. mem_cgroup_charge(folio, vma.vm_mm, gfp).
7. clear_huge_page(folio, vmf.address & HPAGE_PMD_MASK, HPAGE_PMD_NR).
8. /* Take per-PMD pgtable lock */
9. ptl = pmd_lock(vma.vm_mm, vmf.pmd).
10. set_huge_pmd(vma, vmf.address, vmf.pmd, folio).
11. folio_add_lru.
12. spin_unlock(ptl).
13. Return 0.

`Thp::set_huge_pmd(vma, addr, pmd, folio)`:
1. entry = mk_huge_pmd(folio, vma.vm_page_prot).
2. /* Per-anon: pmd_dirty(entry) */
3. set_pmd_at(vma.vm_mm, addr & HPAGE_PMD_MASK, pmd, entry).

`Thp::split_huge_pmd(vma, pmd, addr)`:
1. /* Convert PMD-huge to PTE-table */
2. pgtable = pte_alloc_one(vma.vm_mm).
3. for i in 0..512:
   - subpage = folio_page(folio, i).
   - pte = mk_pte(subpage, vma.vm_page_prot).
   - pgtable[i] = pte.
4. set_pmd_at(vma.vm_mm, addr & HPAGE_PMD_MASK, pmd, mk_pmd_table(pgtable)).

`Thp::split_huge_page(folio) -> Result<()>`:
1. Per-folio split into 512 4KB folios.
2. Update per-mapping (anon_vma / shmem-mapping).
3. Per-PMD using folio: split_huge_pmd.
4. Per-folio.lru: re-add to per-page LRU.

`KhugePaged::collapse(mm, vma, hstart, hend)`:
1. /* Scan VMA for collapsable 4KB-mapped runs */
2. for addr in [hstart, hend) step HPAGE_PMD_SIZE:
   - if 512 contig 4KB-PTEs ∧ all-young ∧ !shared:
     - Alloc huge folio.
     - Copy 512 × 4KB into huge.
     - Set huge PMD; free old folios.

### Out of Scope

- mm/00-overview (Tier-2)
- hugetlb (covered in `hugetlb.md` Tier-3)
- Khugepaged detail (covered separately if expanded)
- Page allocator buddy (covered in `page-allocator.md` Tier-3)
- Reclaim (covered in `reclaim.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `transparent_hugepage_flags` | global flags | `Thp::flags` |
| `do_huge_pmd_anonymous_page()` | per-anon-VMA fault → huge | `Thp::do_huge_pmd_anonymous_page` |
| `do_huge_pmd_wp_page()` | COW on huge PMD | `Thp::do_huge_pmd_wp_page` |
| `huge_pmd_set_accessed()` | per-PMD A-bit set on access | `Thp::huge_pmd_set_accessed` |
| `set_huge_pmd()` | per-PMD install | `Thp::set_huge_pmd` |
| `split_huge_page()` | per-folio split into 4KB pages | `Thp::split_huge_page` |
| `split_huge_pmd()` | per-PMD split + 4KB-PMD-table install | `Thp::split_huge_pmd` |
| `pmd_trans_huge()` / `pmd_devmap()` | per-PMD huge-flag check | helpers |
| `khugepaged_init()` | per-bg daemon init | `KhugePaged::init` |
| `khugepaged_collapse()` | per-VMA scan + collapse | `KhugePaged::collapse` |
| `khugepaged_register()` | per-mm register for scan | `KhugePaged::register` |
| `vma_thp_allowed()` | per-VMA THP-eligible | `Thp::vma_allowed` |
| `transparent_hugepage_active()` | per-feature-active | `Thp::active` |
| `THP_FLAG_*` | per-feature bitmap | shared |
| `MADV_HUGEPAGE` / `MADV_NOHUGEPAGE` | per-VMA madvise | UAPI |
| `mm->def_flags` | per-mm THP default | `MmStruct::def_flags` |

### compatibility contract

REQ-1: Per-flag transparent_hugepage_flags:
- TRANSPARENT_HUGEPAGE_FLAG: enable/disable.
- TRANSPARENT_HUGEPAGE_REQ_MADV_FLAG: only on MADV_HUGEPAGE.
- TRANSPARENT_HUGEPAGE_DEFRAG_DIRECT_FLAG: per-fault sync-defrag.
- TRANSPARENT_HUGEPAGE_DEFRAG_KSWAPD_FLAG: kswapd-only defrag.
- TRANSPARENT_HUGEPAGE_DEFRAG_KSWAPD_OR_MADV_FLAG.
- TRANSPARENT_HUGEPAGE_USE_ZERO_PAGE_FLAG.
- TRANSPARENT_HUGEPAGE_DEFRAG_REQ_MADV_FLAG.

REQ-2: Per-VMA THP eligibility:
- vma_thp_allowed(vma):
  - vma is anon ∨ file/shmem.
  - vma not VM_NOHUGEPAGE.
  - vma size aligned to PMD_SIZE.
  - mm.def_flags + vma.vm_flags consistent.

REQ-3: do_huge_pmd_anonymous_page (page-fault path):
- /* Allocate folio of PMD_ORDER (typically 9 = 2MB). */
- folio = vma_alloc_folio(gfp, HPAGE_PMD_ORDER, vma, addr).
- mem_cgroup_charge_folio.
- /* Set up PMD-mapped huge: */
- set_huge_pmd(vma, addr, folio).
- folio_add_lru.

REQ-4: do_huge_pmd_wp_page:
- COW on huge PMD: split + COW per 4KB OR keep huge + alloc new huge folio.

REQ-5: split_huge_pmd(vma, pmd, addr):
- Convert PMD → page-table of 512 PTEs.
- Each PTE points to 4KB sub-page of original folio.
- Per-folio still a single struct folio (not split-folio).

REQ-6: split_huge_page(folio):
- Per-folio: split into 512 individual 4KB folios.
- Per-page mapping fixed.
- Used per-swap / migration.

REQ-7: khugepaged daemon:
- Per-mm registered via khugepaged_enter or khugepaged_register.
- Background scan per-VMA: identifies 4KB-mapped runs of 512 contiguous pages.
- per-collapse: alloc PMD-folio + copy + install + free old.

REQ-8: Per-MADV_HUGEPAGE:
- vma.vm_flags |= VM_HUGEPAGE.
- Per-fault prefer huge.

REQ-9: Per-MADV_NOHUGEPAGE:
- vma.vm_flags |= VM_NOHUGEPAGE.
- Per-fault skip huge.

REQ-10: Per-userspace ABI:
- /sys/kernel/mm/transparent_hugepage/enabled: always | madvise | never.
- /sys/kernel/mm/transparent_hugepage/defrag: always | defer | defer+madvise | madvise | never.
- /sys/kernel/mm/transparent_hugepage/khugepaged/*.

REQ-11: Per-shmem THP:
- /sys/kernel/mm/transparent_hugepage/shmem_enabled.
- shmem_alloc_folio + page-cache install at PMD-aligned offset.

REQ-12: mTHP (multi-size THP):
- Per-VMA folio order ∈ {0, ..., 9}.
- /sys/kernel/mm/transparent_hugepage/hugepages-{16,32,...,2048}kB/enabled.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `pmd_aligned_vma_only` | INVARIANT | huge alloc only when vma start/end PMD-aligned. |
| `huge_pmd_implies_huge_folio` | INVARIANT | pmd_trans_huge(pmd) ⟹ folio_order >= HPAGE_PMD_ORDER. |
| `split_preserves_data` | INVARIANT | post-split: per-4KB-page data unchanged from huge-folio. |
| `mem_cgroup_charged_pre_install` | INVARIANT | per-set_huge_pmd: folio.memcg charged. |
| `khugepaged_skip_shared` | INVARIANT | khugepaged collapse skips shared/COW pages. |

### Layer 2: TLA+

`mm/thp.tla`:
- Per-fault huge-alloc + per-collapse + per-split.
- Properties:
  - `safety_no_split_during_walk` — per-PMD walked under lock; split serialized.
  - `safety_huge_promotes_correctly` — per-collapse: 4KB pages copied; refs dropped.
  - `liveness_madvise_eventual_promotion` — MADV_HUGEPAGE + sufficient memory ⟹ khugepaged collapses.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Thp::do_huge_pmd_anonymous_page` post: PMD-mapped + folio in LRU + memcg-charged | `Thp::do_huge_pmd_anonymous_page` |
| `Thp::split_huge_pmd` post: PMD becomes PTE-table; data preserved | `Thp::split_huge_pmd` |
| `Thp::split_huge_page` post: per-page distinct folio; mapping updated | `Thp::split_huge_page` |
| `KhugePaged::collapse` post: per-collapsed-range PMD-mapped | `KhugePaged::collapse` |

### Layer 4: Verus/Creusot functional

`Per-VMA THP-eligible + memory-pressure-favorable → huge-PMD-mapped folio reduces TLB pressure` semantic equivalence: per-Linux Documentation/admin-guide/mm/transhuge.rst.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

THP-specific reinforcement:

- **Per-VMA-PMD alignment enforced** — defense against per-misaligned huge-fold causing fault.
- **Per-mm khugepaged registration** — defense against per-mm uncontrolled collapse storm.
- **Per-collapse refcount + lock** — defense against per-collapse race-with-fault UAF.
- **Per-split atomicity** — defense against per-split torn-folio.
- **Per-shared/COW skip** — defense against per-collapse spurious COW-break.
- **Per-MADV_NOHUGEPAGE honored** — defense against per-VMA unwanted huge-promotion.
- **Per-mem_cgroup_charge before install** — defense against per-charge-fail leaving huge-PMD.
- **Per-defrag policy honored** — defense against per-system unbounded compaction during fault.
- **Per-pmd-lock around set/split** — defense against per-PMD torn install.
- **Per-page-fault fallback to 4KB** — defense against per-OOM hard-failure.
- **Per-shmem THP gated by tmpfs mount-options** — defense against per-FS unwanted promotion.

