---
title: "Tier-3: mm/ksm.c — Kernel Same-page Merging (KSM) — content-based dedup"
tags: ["tier-3", "mm", "design-doc"]
sources: []
contributors: ["gb9t"]
created: 2026-05-10
updated: 2026-05-10
---


## Design Specification

### Summary

KSM (Kernel Same-page Merging) deduplicates identical anonymous pages across multiple processes. Per-task `madvise(MADV_MERGEABLE)` opts a VMA in. Per-ksmd-thread periodically scans MERGEABLE pages: per-page hash + content-compare against **stable tree** (previously-merged immutable pages) and **unstable tree** (candidates not yet stable). On match: replace per-PTE with COW-mapped reference to single shared page; reclaim duplicate. Critical for: VM dedup (KVM-host running 100s of similar guests); container images sharing libraries; database servers.

This Tier-3 covers `ksm.c` (~4021 lines).

### Acceptance Criteria

- [ ] AC-1: madvise MADV_MERGEABLE on anon range: VMA flagged.
- [ ] AC-2: ksmd scanning detects identical pages across two processes.
- [ ] AC-3: Per-merge: PTE points to shared kpage; counter pages_sharing++.
- [ ] AC-4: Subsequent write to ksm-page: break_ksm allocates fresh page.
- [ ] AC-5: madvise MADV_UNMERGEABLE: VMA un-flagged; per-page un-merged.
- [ ] AC-6: /sys/kernel/mm/ksm/run = 2: bulk un-merge runs.
- [ ] AC-7: stable_tree_search hits known-merged page: instant merge.
- [ ] AC-8: NUMA: per-node trees keep pages per-node.
- [ ] AC-9: pages_volatile counter: tracks pages-changed-during-merge-attempt.
- [ ] AC-10: KVM running 10 identical guests: massive memory savings.

### Architecture

Per-rmap_item:

```
struct KsmRmapItem {
  rmap_list: ListLink,                            // mm.rmap_list
  anon_vma: *AnonVma,
  nid: i32,                                       // NUMA node
  mm: *MmStruct,
  address: u64,                                   // virtual addr
  oldchecksum: u32,                               // last checksum
  rmap_node: RbNode,                              // in stable / unstable tree
  head: u8,                                       // STABLE / UNSTABLE tree
  next: *KsmRmapItem,                             // chain (multi rmap → kpage)
}

struct KsmStableNode {
  rb_node: RbNode,
  hlist: HListHead,                               // ksm_rmap_items
  kpfn: u64,                                      // page-frame
  rmap_hlist_len: u16,
  nid: u16,
}
```

Per-mm slot:

```
struct KsmMmSlot {
  mm_node: HListNode,                             // ksm_mm_slot_hash
  mm: *MmStruct,
  link: ListLink,                                 // ksm_mm_head list
  rmap_list: ListHead<KsmRmapItem>,
  flags: u32,                                     // KSM_MM_LISTED / ...
}
```

Globals:

```
static KSM_RUN: AtomicU32;                        // 0=STOP / 1=RUN / 2=UNMERGE
static KSM_THREAD_PAGES_TO_SCAN: AtomicU32;       // 100 default
static KSM_THREAD_SLEEP_MILLISECS: AtomicU32;     // 20 default
static MM_HEAD: ListHead<KsmMmSlot>;
static STABLE_TREE: [RbRoot; MAX_NUMNODES];       // per-node
static UNSTABLE_TREE: [RbRoot; MAX_NUMNODES];
static KSM_PAGES_SHARED: AtomicU64;
static KSM_PAGES_SHARING: AtomicU64;
```

`Ksm::madvise(vma, behavior, addr, end) -> Result<()>`:
1. switch behavior:
   - MADV_MERGEABLE: vma.vm_flags |= VM_MERGEABLE; if !mm.flags & MMF_VM_MERGEABLE: __ksm_enter(mm).
   - MADV_UNMERGEABLE: unmerge_ksm_pages(vma, addr, end); vma.vm_flags &= !VM_MERGEABLE.

`Ksm::do_scan(scan_npages)`:
1. mm = current_scan_mm.
2. For i in 0..scan_npages:
   - rmap_item = scan_get_next_rmap_item(&page).
   - Ksm::cmp_and_merge_page(page, rmap_item).
3. /* Move to next mm */
4. queue_delayed_work(ksm_thread_work, msecs_to_jiffies(sleep_millisecs)).

`Ksm::cmp_and_merge_page(page, rmap_item)`:
1. checksum = jhash2(page_data, PAGE_SIZE/4, seed).
2. if checksum == rmap_item.oldchecksum: pages_volatile--; return.
3. /* Stable-tree first */
4. kpage = Ksm::stable_tree_search(page).
5. if kpage:
   - err = Ksm::try_to_merge_with_ksm_page(rmap_item, page, kpage).
   - if !err: pages_sharing++; return.
6. /* Unstable-tree */
7. tree_item = Ksm::unstable_tree_search_insert(rmap_item, page, &tree_page).
8. if tree_item:
   - kpage = Ksm::try_to_merge_two_pages(rmap_item, page, tree_item, tree_page).
   - if kpage: stable_tree_insert(kpage); pages_shared++.

`Ksm::try_to_merge_one_page(vma, page, kpage) -> i32`:
1. folio_lock(page).
2. write_protect_page(vma, page, &orig_pte).
3. if !pages_identical(page, kpage): folio_unlock; return -EBUSY.
4. Ksm::replace_page(vma, page, kpage, orig_pte).
5. folio_unlock(page).

`Ksm::replace_page(vma, page, kpage, orig_pte)`:
1. get_page(kpage).
2. pte = mk_pte(kpage, vma.vm_page_prot).
3. pte = pte_wrprotect(pte_mkclean(pte)).
4. flush_cache_page(vma, addr, page_to_pfn(page)).
5. set_pte_at_notify(mm, addr, pte_addr, pte).
6. page_remove_rmap(page, vma, false).
7. dec_mm_counter(MM_ANONPAGES).
8. put_page(page).

`Ksm::break_ksm(vma, addr) -> Result<()>`:
1. /* Triggered on write-fault to ksm-page */
2. handle_mm_fault(vma, addr, FAULT_FLAG_WRITE | FAULT_FLAG_KSM_BREAK).
3. fresh_folio = vma_alloc_folio(GFP_HIGHUSER, 0, vma, addr).
4. copy_user_highpage(fresh_folio, kpage, addr, vma).
5. set_pte_at(mm, addr, pte_addr, mk_pte(fresh_folio, vma.vm_page_prot)).

### Out of Scope

- mm/00-overview (Tier-2)
- madvise (covered in `mmap.md` Tier-3 if mature)
- VMA core (covered in `virtual-memory.md` Tier-3)
- COW handling (covered in `virtual-memory.md` Tier-3)
- Implementation code

### upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct ksm_rmap_item` | per-page rmap-item linked into stable/unstable tree | `KsmRmapItem` |
| `struct ksm_stable_node` | per-stable-tree-node | `KsmStableNode` |
| `struct ksm_mm_slot` | per-mm scan-state | `KsmMmSlot` |
| `ksm_init()` | per-module init | `Ksm::init` |
| `ksm_madvise()` | per-VMA opt-in | `Ksm::madvise` |
| `__ksm_enter()` / `__ksm_exit()` | per-mm enter/exit | `Ksm::__enter` / `__exit` |
| `ksm_do_scan()` | per-iter scan + merge | `Ksm::do_scan` |
| `cmp_and_merge_page()` | per-page try-merge | `Ksm::cmp_and_merge_page` |
| `try_to_merge_one_page()` | per-page replace PTE | `Ksm::try_to_merge_one_page` |
| `try_to_merge_two_pages()` | per-page-pair stable-create | `Ksm::try_to_merge_two_pages` |
| `try_to_merge_with_ksm_page()` | per-page merge with stable | `Ksm::try_to_merge_with_ksm_page` |
| `stable_tree_search()` | per-page lookup in stable | `Ksm::stable_tree_search` |
| `unstable_tree_search_insert()` | per-page lookup-or-insert in unstable | `Ksm::unstable_tree_search_insert` |
| `replace_page()` | per-PTE swap to ksm-page | `Ksm::replace_page` |
| `break_ksm()` | per-VMA un-merge (CoW) | `Ksm::break_ksm` |
| `ksm_might_need_to_copy()` | per-fault un-merge check | `Ksm::might_need_to_copy` |
| `MADV_MERGEABLE` / `MADV_UNMERGEABLE` | per-VMA madvise | UAPI |

### compatibility contract

REQ-1: Per-VMA opt-in:
- madvise(addr, len, MADV_MERGEABLE): vma.vm_flags |= VM_MERGEABLE.
- Per-mm: __ksm_enter(mm) registers in ksm_mm_slot.

REQ-2: Per-mm scan-state:
- ksm_mm_slot: list_head (LRU of mms), mm pointer, mm_slot_link.
- Per-mm scan_addr: cursor per-mm.

REQ-3: ksm_do_scan(scan_npages):
- Each iter scans at most `pages_to_scan` pages.
- For each MERGEABLE VMA in current mm:
  - For each anon page at scan_addr in [vma.vm_start, vma.vm_end):
    - cmp_and_merge_page(page, rmap_item).
    - scan_addr += PAGE_SIZE.
- Move to next mm at iter end.

REQ-4: cmp_and_merge_page(page, rmap_item):
- /* Phase 1: Try stable tree */
- kpage = stable_tree_search(page).
- If kpage: try_to_merge_with_ksm_page(rmap_item, page, kpage); return.
- /* Phase 2: Try unstable tree */
- tree_rmap_item = unstable_tree_search_insert(rmap_item, page, &tree_page).
- If tree_rmap_item:
  - kpage = try_to_merge_two_pages(rmap_item, page, tree_rmap_item, tree_page).
  - If kpage: stable_tree_insert(kpage); promote both to stable.

REQ-5: try_to_merge_one_page(vma, page, kpage):
- Take folio_lock(page).
- write_protect_page(vma, page).
- /* Verify no concurrent write */
- if pages_identical(page, kpage):
  - replace_page(vma, page, kpage, orig_pte).
  - return 0.
- folio_unlock; return -EBUSY.

REQ-6: replace_page(vma, page, kpage, orig_pte):
- get_page(kpage).
- pte = mk_pte(kpage, vma_page_prot).
- pte = pte_mkclean(pte_wrprotect(pte))  // shared + write-protected.
- set_pte_at(vma.vm_mm, addr, pte_addr, pte).
- inc_mm_counter(MM_ANONPAGES) for kpage; dec for old.
- page_remove_rmap(page, vma, ...).
- put_page(page).

REQ-7: stable_tree_search(page):
- Per-rb-tree walk by page-content hash + bytewise compare.
- Per-NUMA: per-node stable trees (NUMA-aware).
- Returns kpage (already merged) or NULL.

REQ-8: unstable_tree_search_insert(rmap_item, page):
- Walk per-NUMA unstable rb-tree.
- If match: return matching rmap_item.
- Else: insert as candidate.

REQ-9: Per-CoW: break_ksm:
- Per-write-fault on ksm-page: break_ksm(vma, addr).
- Allocates fresh page; copies content; replaces PTE.
- Decrements ksm-page rmap.

REQ-10: ksm_might_need_to_copy(folio, vma, addr):
- Per-fork: child gets ksm-page; on first write, break.

REQ-11: Per-userspace ABI:
- /sys/kernel/mm/ksm/run: 0=stop / 1=run / 2=unmerge.
- /sys/kernel/mm/ksm/pages_to_scan: per-iter cap.
- /sys/kernel/mm/ksm/sleep_millisecs: between iters.
- /sys/kernel/mm/ksm/pages_shared / pages_sharing / pages_unshared / pages_volatile / full_scans / merge_across_nodes / use_zero_pages.

REQ-12: Per-NUMA-aware:
- merge_across_nodes=1 (default 1): allow cross-node merging.
- merge_across_nodes=0: per-node trees + merging.

### verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `kpage_refcount_eq_sharing_count` | INVARIANT | per-kpage refcount == #PTEs pointing to it. |
| `merge_only_identical` | INVARIANT | per-merge: pages_identical pre-replace. |
| `ksm_pte_wrprotected` | INVARIANT | per-ksm-PTE: pte_write(pte) == false. |
| `mm_in_slot_iff_enter_called` | INVARIANT | mm in ksm_mm_head ⟺ __ksm_enter called. |
| `unstable_tree_walked_under_rcu_or_lock` | INVARIANT | per-walk: ksm_thread_mutex held. |

### Layer 2: TLA+

`mm/ksm.tla`:
- Per-page candidate → unstable → stable lifecycle.
- Per-write-fault break_ksm.
- Properties:
  - `safety_no_data_loss_on_merge` — per-merge: kpage content == src content.
  - `safety_no_share_after_break` — per-break_ksm: PTE points to fresh page, not kpage.
  - `liveness_volatile_eventually_stable_or_dropped` — per-page either stabilizes or dropped from candidate.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Ksm::cmp_and_merge_page` post: per-match merged; per-no-match in unstable | `Ksm::cmp_and_merge_page` |
| `Ksm::try_to_merge_one_page` post: per-success: PTE → kpage; refcount++ | `Ksm::try_to_merge_one_page` |
| `Ksm::replace_page` post: PTE wrprotected + clean; rmap updated | `Ksm::replace_page` |
| `Ksm::break_ksm` post: per-PTE → fresh-folio; refcount kpage-- | `Ksm::break_ksm` |

### Layer 4: Verus/Creusot functional

`Per-MERGEABLE VMA → KSM identifies identical pages → merges via shared write-protected PTE → savings = (N-1) × page_count` semantic equivalence: per-Documentation/admin-guide/mm/ksm.rst.

### hardening

(Inherits row-1 features from `mm/00-overview.md` § Hardening.)

KSM-specific reinforcement:

- **Per-VMA opt-in via madvise** — defense against per-system unintended merging.
- **Per-pages_identical check pre-merge** — defense against per-content-mismatch corruption.
- **Per-PTE write-protected on merge** — defense against per-write to shared page.
- **Per-CoW break_ksm on write-fault** — defense against per-shared-page silent corruption.
- **Per-ksm_thread_mutex** — defense against per-tree concurrent-access UAF.
- **Per-NUMA per-node trees option** — defense against cross-node migration cost.
- **Per-checksum prefilter** — defense against per-O(N) byte-compare DoS.
- **Per-mm slot LRU** — defense against per-stalled-mm starving others.
- **Per-merge_across_nodes off** — defense against cross-node sharing causing remote-RAM thrash.
- **Per-/sys/kernel/mm/ksm/run = 2 unmerge bulk** — defense against per-system-wide unmerge stall.
- **Per-volatile counter tracks fluctuating pages** — defense against per-pages_to_scan budget waste.

