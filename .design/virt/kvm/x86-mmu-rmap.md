# Tier-3: arch/x86/kvm/mmu/mmu.c (rmap subset) — KVM MMU reverse-mapping (sptes-pointing-to-gfn)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-mmu-shadow.md
upstream-paths:
  - arch/x86/kvm/mmu/mmu.c (rmap subset: ~880..1100, ~1500..1900)
  - include/linux/kvm_host.h (kvm_rmap_head)
  - arch/x86/kvm/mmu/mmu_internal.h
-->

## Summary

KVM MMU rmap (reverse-mapping) tracks **all SPTEs that map a given GFN**, enabling efficient per-GFN write-protect / zap / age-bit operations during dirty-tracking, MMU-notifier, and memslot-mutation. Per-memslot has rmap-arrays indexed by [level][gfn-offset]; each entry is `kvm_rmap_head` — a smart pointer that stores either a single SPTE address (compact) or a list of pte_list_desc descriptors (when ≥ 2 SPTEs map same GFN). Per-shadow-MMU uses; per-TDP-MMU does NOT (TDP iterates page-tables directly). Critical for: dirty-bitmap walking + per-GFN MMU-notifier invalidate.

This Tier-3 covers rmap subset of `mmu.c` (~400 lines).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_rmap_head` | per-(memslot, level, gfn) rmap-head | `KvmRmapHead` |
| `struct pte_list_desc` | per-list-of-spte block | `PteListDesc` |
| `kvm_rmap_lock()` / `kvm_rmap_unlock()` | per-rmap_head lock | `Rmap::lock` / `unlock` |
| `pte_list_add()` | per-(spte, gfn) link | `Rmap::pte_list_add` |
| `pte_list_remove()` | per-(spte, gfn) unlink | `Rmap::pte_list_remove` |
| `gfn_to_rmap()` | per-(gfn, level, slot) → rmap_head | `Rmap::gfn_to_rmap` |
| `rmap_get_first()` / `rmap_get_next()` | per-rmap iteration | `Rmap::iter` |
| `kvm_zap_rmap()` | per-rmap zap all sptes | `Rmap::zap_rmap` |
| `kvm_unmap_rmap()` | per-rmap unmap (release ref) | `Rmap::unmap_rmap` |
| `kvm_age_rmap()` | per-rmap age bit | `Rmap::age_rmap` |
| `kvm_test_age_rmap()` | per-rmap test age | `Rmap::test_age_rmap` |
| `__rmap_set_dirty()` | per-rmap mark dirty | `Rmap::set_dirty` |
| `__rmap_clear_dirty()` | per-rmap clear dirty | `Rmap::clear_dirty` |
| `__rmap_write_protect()` | per-rmap WP | `Rmap::write_protect` |
| `slot_handle_level_range()` | per-(slot, level) walker | `Rmap::slot_handle_level_range` |

## Compatibility contract

REQ-1: kvm_rmap_head encoding:
- `val: AtomicULong`
- Bit[0] (LSB) = 1: pointer to `pte_list_desc` (multi-SPTE).
- Bit[0] = 0: direct SPTE pointer (single-SPTE) OR 0 (empty).
- High bit (KVM_RMAP_LOCKED, bit-0 of val with desc): per-rmap lock-bit during cmpxchg.

REQ-2: pte_list_desc structure:
- `more: *PteListDesc` (next descriptor).
- `spte_count: u16`.
- `sptes: [*u64; PTE_LIST_EXT]` (e.g. 14 entries).

REQ-3: pte_list_add(kvm, cache, spte, rmap_head):
- val = kvm_rmap_lock(rmap_head).
- if val == 0: rmap_head.val = (unsigned long)spte (no LSB set).
- elif val & 1 (multi-list):
  - desc = head; iterate to find free slot or alloc new desc.
- else (single):
  - alloc desc; desc.sptes[0] = (val); desc.sptes[1] = spte; desc.spte_count = 2.
  - rmap_head.val = (desc | 1).
- kvm_rmap_unlock(rmap_head, new_val).

REQ-4: pte_list_remove(kvm, spte, rmap_head):
- val = kvm_rmap_lock.
- if !val: BUG.
- if !(val & 1):
  - assert val == spte; rmap_head.val = 0.
- else (multi-list):
  - walk descs; find spte; remove (memmove tail forward).
  - if last desc empty: free desc; possibly collapse to single.
- kvm_rmap_unlock.

REQ-5: gfn_to_rmap(gfn, level, slot):
- idx = gfn_to_index(gfn, slot.base_gfn, level).
- Return slot.arch.rmap[level - PG_LEVEL_4K] + idx.

REQ-6: rmap_get_first / rmap_get_next:
- Iterator over rmap_head's spte list.

REQ-7: kvm_zap_rmap(kvm, slot, rmap_head):
- For each spte in rmap_head: drop_spte(spte); pte_list_remove.
- Returns whether anything was zapped.

REQ-8: __rmap_write_protect(kvm, slot, rmap_head, pt_protect):
- For each spte in rmap_head: clear writable-bit (sets W=0 in PTE).
- Per-pt_protect: also clear A/D bits.

REQ-9: __rmap_clear_dirty(kvm, slot, rmap_head):
- For each spte in rmap_head: clear dirty-bit (D=0); set writable=1.

REQ-10: kvm_age_rmap(kvm, slot, rmap_head, gfn):
- For each spte: clear access-bit (A=0); flush per-CPU TLB.

REQ-11: slot_handle_level_range(kvm, slot, fn, start_level, end_level, start_gfn, end_gfn):
- For level in start_level..=end_level:
  - For gfn in start_gfn..=end_gfn step (1 << (level-1)*9):
    - rmap_head = gfn_to_rmap(gfn, level, slot).
    - fn(kvm, slot, rmap_head).

REQ-12: Per-TDP-MMU not used:
- TDP MMU iterates page-tables via SPTE walks (no rmap).
- Shadow MMU uses rmap.

## Acceptance Criteria

- [ ] AC-1: Single SPTE for GFN: rmap_head.val == spte_addr (LSB=0).
- [ ] AC-2: Two SPTEs for GFN: rmap_head.val = desc-pointer | 1.
- [ ] AC-3: pte_list_add(spte_2, rmap): converts to multi-list.
- [ ] AC-4: pte_list_remove(spte_1, rmap): collapses back to single.
- [ ] AC-5: kvm_zap_rmap(rmap): all sptes zapped; rmap_head.val == 0.
- [ ] AC-6: __rmap_write_protect: per-spte W-bit cleared.
- [ ] AC-7: __rmap_clear_dirty: per-spte D-bit cleared.
- [ ] AC-8: kvm_age_rmap: per-spte A-bit cleared.
- [ ] AC-9: slot_handle_level_range: walks per-level.
- [ ] AC-10: Per-multi-list collision: desc.spte_count == #sptes.
- [ ] AC-11: Per-rmap_lock cmpxchg: concurrent add/remove serialized.

## Architecture

Per-memslot rmap arrays:

```
struct KvmArchMemslot {
  rmap: [Vec<KvmRmapHead>; KVM_NR_PAGE_SIZES],   // [PG_LEVEL_4K..1G]
  ...
}
```

Per-rmap-head:

```
#[repr(transparent)]
struct KvmRmapHead {
  val: AtomicUSize,                              // bit[0]=is_desc; bit[0]=lock
}

const KVM_RMAP_LOCKED: usize = 1 << 0;            // for cmpxchg lock
const KVM_RMAP_DESC: usize = 1 << 0;              // for is-desc tag (separate context)
```

Per-list-of-sptes:

```
struct PteListDesc {
  more: Option<&PteListDesc>,
  spte_count: u16,
  sptes: [Option<*mut u64>; PTE_LIST_EXT],
}
const PTE_LIST_EXT: usize = 14;
```

`Rmap::lock(rmap_head) -> u64`:
1. for {
   - old = rmap_head.val.load();
   - if !(old & KVM_RMAP_LOCKED):
     - if cmpxchg_acquire(&rmap_head.val, old, old | KVM_RMAP_LOCKED): return old.
   - cpu_relax();
2. }.

`Rmap::unlock(rmap_head, new_val)`:
1. rmap_head.val.store_release(new_val).

`Rmap::pte_list_add(kvm, cache, spte, rmap_head) -> usize` (returns count):
1. old_val = Rmap::lock(rmap_head).
2. if old_val == 0: new_val = spte as USize; count=1.
3. elif old_val & KVM_RMAP_DESC:
   - desc = ptr_from_val(old_val).
   - find last desc; if !full: append spte; count++.
   - else: alloc new desc from cache; chain.
4. else (single):
   - desc = alloc_from_cache.
   - desc.sptes[0] = ptr_from_val(old_val).
   - desc.sptes[1] = spte.
   - desc.spte_count = 2.
   - new_val = (desc as USize) | KVM_RMAP_DESC.
5. Rmap::unlock(rmap_head, new_val).
6. Return count.

`Rmap::pte_list_remove(kvm, spte, rmap_head)`:
1. old_val = Rmap::lock(rmap_head).
2. if !(old_val & KVM_RMAP_DESC):
   - assert old_val == spte.
   - new_val = 0.
3. else:
   - walk desc-chain; find spte; remove (compact remaining).
   - if last desc has 1 entry + no more-link: collapse to single (desc.sptes[0] as USize).
   - else: keep desc-chain.
4. Rmap::unlock(rmap_head, new_val).

`Rmap::gfn_to_rmap(gfn, level, slot) -> &KvmRmapHead`:
1. idx = gfn_to_index(gfn, slot.base_gfn, level).
2. Return &slot.arch.rmap[level - PG_LEVEL_4K][idx].

`Rmap::zap_rmap(kvm, slot, rmap_head) -> bool`:
1. flush = false.
2. while spte = rmap_get_first(rmap_head):
   - drop_spte(kvm, spte).
   - flush = true.
3. Return flush.

`Rmap::write_protect(kvm, slot, rmap_head, pt_protect) -> bool`:
1. flush = false.
2. iter over rmap_head.sptes:
   - if pt_protect: spte_set_writable(spte, false); spte_set_dirty(spte, false).
   - else: spte_set_dirty(spte, false) only.
   - flush = true.
3. Return flush.

`Rmap::slot_handle_level_range(kvm, slot, handler, start_level, end_level, start_gfn, end_gfn)`:
1. for level in start_level..=end_level:
   - for gfn in start_gfn..end_gfn step (KVM_PAGES_PER_HPAGE(level)):
     - rmap_head = Rmap::gfn_to_rmap(gfn, level, slot).
     - handler(kvm, slot, rmap_head, level).

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `rmap_lock_held_during_mut` | INVARIANT | per-pte_list_add/remove: rmap_head locked. |
| `desc_count_le_pte_list_ext` | INVARIANT | per-desc.spte_count ≤ PTE_LIST_EXT. |
| `single_or_multi_consistent` | INVARIANT | rmap_head.val LSB consistent with single-vs-desc state. |
| `gfn_idx_in_rmap_array` | INVARIANT | per-gfn_to_rmap: idx < slot.npages. |
| `level_in_range` | INVARIANT | per-level ∈ [PG_LEVEL_4K, PG_LEVEL_1G]. |

### Layer 2: TLA+

`virt/kvm/mmu_rmap.tla`:
- Per-rmap add/remove + per-walker zap/write_protect.
- Properties:
  - `safety_concurrent_add_serialized` — per-cmpxchg-lock serializes.
  - `safety_zap_clears_all` — post-zap_rmap: rmap_head.val == 0.
  - `safety_no_double_remove` — per-pte_list_remove called per-(spte, rmap) once.
  - `liveness_lock_eventually_acquired` — per-cmpxchg loop terminates.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Rmap::pte_list_add` post: spte in rmap; count incremented | `Rmap::pte_list_add` |
| `Rmap::pte_list_remove` post: spte not in rmap; count decremented | `Rmap::pte_list_remove` |
| `Rmap::zap_rmap` post: all sptes zapped; rmap_head.val=0 | `Rmap::zap_rmap` |
| `Rmap::write_protect` post: per-spte W-bit (and D-bit if pt_protect) cleared | `Rmap::write_protect` |

### Layer 4: Verus/Creusot functional

`Per-(slot, level, gfn) rmap → all sptes mapping that GFN tracked → per-MMU-notifier invalidate or per-dirty-track-WP applies to all` semantic equivalence: per-rmap matches shadow-MMU contract.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-mmu-shadow.md` § Hardening.)

Rmap-specific reinforcement:

- **Per-rmap_head atomic-cmpxchg lock** — defense against concurrent add/remove corruption.
- **Per-pte_list_desc memcache pre-alloc** — defense against per-fault alloc-failure.
- **Per-desc.spte_count overflow detected** — defense against per-spte_count > PTE_LIST_EXT.
- **Per-rmap_head LSB tagging** — defense against per-pointer-misinterpret single-vs-desc.
- **Per-zap_rmap iterates all sptes** — defense against per-leaked spte after zap.
- **Per-walker walks per-level** — defense against per-huge-page miss.
- **Per-rmap_walk in MMU-notifier ctx** — defense against per-mm-page-mover miss.
- **Per-TDP-MMU bypasses rmap** — defense against double-tracking cost.
- **Per-rmap_head zeroed at memslot-create** — defense against per-stale-spte after slot reuse.
- **Per-collapse single-from-multi atomicity** — defense against per-collapse race-with-add.

## Grsecurity/PaX-style Reinforcement

Baseline hardening (always applied):

- **PAX_USERCOPY** — rmap chains never copied to/from user; accessor bounds-checked.
- **PAX_KERNEXEC** — rmap walker callbacks RO after init.
- **PAX_RANDKSTACK** — randomized kstack per fault / MMU-notifier callback.
- **PAX_REFCOUNT** — pte_list_desc.spte_count saturating; panic on overflow.
- **PAX_MEMORY_SANITIZE** — pte_list_desc kmem_cache GFP_ZERO; freed-back chunks zeroed.
- **PAX_UDEREF** — no user pointer in rmap; assert hva→gfn translation safe.
- **PAX_RAP / kCFI** — rmap_walk handler indirect-call type-checked.
- **GRKERNSEC_HIDESYM** — spte pointer / pte_list_desc address redacted.
- **GRKERNSEC_DMESG** — overflow / unrecoverable rmap warnings rate-limited.

Rmap-specific:

- **CAP_SYS_ADMIN on KVM_RUN / memslot ops that pin rmap chains** — defense against unprivileged rmap mutation.
- **rmap_head LSB tag bit guarded by cmpxchg** — TOCTOU-safe.
- **Memslot delete zaps rmap before slot free** — closes per-slot-reuse stale-spte UAF.

Rationale: rmap is the reverse-map index from gfn→spte; sanitize + saturating refcount close stale-spte UAF that would otherwise let a torn-down memslot's mapping survive into a new one.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM core MMU (covered in `x86-mmu-shadow.md` Tier-3)
- KVM TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- KVM SPTE format (covered in `x86-mmu-spte.md` Tier-3)
- KVM page-track (covered in `x86-mmu-page-track.md` Tier-3)
- Implementation code
