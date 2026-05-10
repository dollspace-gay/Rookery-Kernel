# Tier-3: virt/kvm/kvm_main.c (memslot subset) — KVM memory slots (RCU mgmt + HVA→GPA translation + per-slot dirty bitmap + KVM_SET_USER_MEMORY_REGION)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/00-overview.md
upstream-paths:
  - virt/kvm/kvm_main.c (memslot portions)
  - include/linux/kvm_host.h (struct kvm_memslots, struct kvm_memory_slot)
  - include/uapi/linux/kvm.h (KVM_USER_MEM_REGION)
-->

## Summary

KVM memory slots are how userspace VMM (qemu, cloud-hypervisor, firecracker) tells KVM "this guest physical address range maps to this user-virtual range backed by anonymous OR file-backed OR device-backed OR guest_memfd-backed memory". Per-VM `kvm_memslots` array (per-address-space — there are 2 on x86: SMM vs non-SMM) RCU-swapped on every `KVM_SET_USER_MEMORY_REGION[2]` ioctl. Critical KVM hot path: every guest page-fault looks up the matching memslot via per-VM RCU memslots; every dirty-page-tracking call walks per-slot bitmaps; every memory-encrypted-guest's mapping has per-slot guest_memfd info.

This Tier-3 covers the memslot subset of `virt/kvm/kvm_main.c` (~1500 lines of memslot-related logic — `kvm_set_memory_region`, `kvm_set_internal_memslot`, `kvm_get_dirty_log`, `kvm_clear_dirty_log`, `kvm_swap_active_memslots`, `kvm_arch_prepare_memory_region`, `kvm_arch_commit_memory_region`, `__gfn_to_pfn_memslot`, `gfn_to_hva_memslot_prot`, etc.).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_memslots` | per-address-space memslots array | `kernel::kvm::Memslots` |
| `struct kvm_memory_slot` | per-slot control block | `kernel::kvm::MemorySlot` |
| `kvm_memslots(kvm, as_id)` | per-VM RCU-protected memslots accessor | `Vm::memslots` |
| `kvm_get_active_memslots(kvm, as_id)` | locked variant | `Vm::active_memslots` |
| `kvm_set_memory_region(kvm, mem)` | top-level memslot install (called from KVM_SET_USER_MEMORY_REGION ioctl) | `Vm::set_memory_region` |
| `kvm_set_internal_memslot(kvm, mem)` | install internal slot (e.g., per-arch APIC-access page) | `Vm::set_internal_memslot` |
| `kvm_swap_active_memslots(kvm, as_id, &slot, ...)` | RCU-swap memslots ptrs | `Vm::swap_active_memslots` |
| `kvm_arch_prepare_memory_region(kvm, old, new, change)` | per-arch pre-commit hook | `arch::kvm::prepare_memory_region` |
| `kvm_arch_commit_memory_region(kvm, old, new, change)` | per-arch post-commit hook | `arch::kvm::commit_memory_region` |
| `kvm_arch_flush_shadow_memslot(kvm, slot)` | per-arch flush after delete | `arch::kvm::flush_shadow_memslot` |
| `kvm_arch_flush_shadow_all(kvm)` | flush all shadows (vm destroy) | `arch::kvm::flush_shadow_all` |
| `__gfn_to_pfn_memslot(slot, gfn, atomic, async, write_fault, &writable)` | per-slot gfn → host pfn | `MemorySlot::gfn_to_pfn` |
| `gfn_to_hva_memslot(slot, gfn)` | per-slot gfn → user-virtual address | `MemorySlot::gfn_to_hva` |
| `gfn_to_hva_memslot_prot(slot, gfn, &writable)` | with R/W check | `MemorySlot::gfn_to_hva_prot` |
| `gfn_to_memslot(kvm, gfn)` | per-VM gfn → slot lookup (binary-search) | `Vm::gfn_to_memslot` |
| `kvm_get_dirty_log(kvm, log)` | KVM_GET_DIRTY_LOG ioctl handler (per-slot bitmap) | `Vm::get_dirty_log` |
| `kvm_clear_dirty_log(kvm, log)` | KVM_CLEAR_DIRTY_LOG ioctl handler | `Vm::clear_dirty_log` |
| `kvm_get_dirty_log_protect(kvm, log)` / `_clear_*_protect(...)` | dirty-tracking + write-protect variants | `Vm::get_dirty_log_protect` / `_clear_dirty_log_protect` |
| `kvm_dirty_ring_alloc(ring, count)` | per-vCPU dirty ring alloc | `DirtyRing::alloc` |
| `kvm_alloc_dirty_bitmap(slot)` / `_free_dirty_bitmap(slot)` | per-slot bitmap alloc/free | `MemorySlot::alloc_dirty_bitmap` / `_free_dirty_bitmap` |
| `kvm_arch_create_memslot(kvm, slot, npages)` | per-arch alloc per-slot bookkeeping (e.g., per-page link headed by lpage_info) | `arch::kvm::create_memslot` |

## Compatibility contract

REQ-1: `KVM_SET_USER_MEMORY_REGION` UAPI byte-identical (struct kvm_userspace_memory_region):
- `slot`: 16-bit index + as_id encoded; up to 32K slots per VM.
- `flags`: KVM_MEM_LOG_DIRTY_PAGES, KVM_MEM_READONLY, KVM_MEM_GUEST_MEMFD.
- `guest_phys_addr` (gpa).
- `memory_size` (bytes).
- `userspace_addr` (hva backing).

REQ-2: `KVM_SET_USER_MEMORY_REGION2` (newer): adds `guest_memfd` + `guest_memfd_offset` for SEV-SNP / TDX private-memory guest_memfd backing.

REQ-3: Validation:
- `gpa + memory_size` doesn't overflow.
- `userspace_addr` (hva) is page-aligned + valid for current's mm.
- `memory_size` is page-aligned + non-zero (or zero to delete).
- No overlap with existing slot in same as_id.
- For KVM_MEM_GUEST_MEMFD: guest_memfd is valid + guest_memfd_offset bounded.

REQ-4: Operations: CREATE (slot didn't exist) / DELETE (memory_size == 0) / MOVE (gpa changed for existing) / FLAGS_ONLY (only flags changed, e.g., enable dirty tracking).

REQ-5: Per-VM 2 address spaces (x86: SMM + non-SMM); each gets independent `kvm_memslots`. as_id encoded in upper bits of `slot` field.

REQ-6: RCU semantics: `kvm_memslots(kvm, as_id)` returns RCU-protected snapshot; readers see either old or new slot table fully (never partial). Writers via `kvm_swap_active_memslots` allocate new array, populate, RCU-swap pointers; old array RCU-freed after grace period.

REQ-7: Per-slot bitmap (KVM_MEM_LOG_DIRTY_PAGES enabled): one bit per 4K page indicating dirty since last reset; double-buffered in KVM_GET_DIRTY_LOG_PROTECT mode (read-and-clear via second bitmap).

REQ-8: Dirty-ring (per-vCPU mode, set via KVM_CAP_DIRTY_LOG_RING_ACQ_REL): per-vCPU acq-rel-ordered ring buffer of dirty GFNs; producer (vCPU thread on page-table-walk) appends; consumer (userspace migration thread) drains.

REQ-9: `gfn_to_pfn_memslot` translation:
- For normal anon-backed: walk userspace mm pgtables → host pfn.
- For file-backed: file's a_ops->fault() → host pfn.
- For guest_memfd: per-guest_memfd page lookup (cross-ref `guest-memfd.md` future Tier-3).
- Per-vCPU mmu_lock taken to prevent concurrent host pte change racing with KVM SPTE install.

REQ-10: Per-arch pre/post-commit hooks: arch can install/remove per-slot bookkeeping (e.g., x86 flushes prior shadow ptes for deleted slots).

## Acceptance Criteria

- [ ] AC-1: qemu boot Linux guest: KVM_SET_USER_MEMORY_REGION called for each per-region (RAM, MMIO, BIOS, etc.); slots installed correctly.
- [ ] AC-2: SEV-SNP / TDX guest: KVM_SET_USER_MEMORY_REGION2 with guest_memfd backing for private memory; guest boots.
- [ ] AC-3: Dirty-tracking test: enable via KVM_MEM_LOG_DIRTY_PAGES on a slot; sustained vCPU writes; KVM_GET_DIRTY_LOG returns correct bitmap.
- [ ] AC-4: Dirty-ring test: KVM_CAP_DIRTY_LOG_RING_ACQ_REL bound; vCPU writes append entries; userspace consumer drains.
- [ ] AC-5: Slot delete test: delete a slot; subsequent guest access to that gpa generates EXIT_MMIO (via shadow-flush).
- [ ] AC-6: Slot move test: move existing slot to different gpa; in-flight guest access racing-with-move handled via mmu_lock.
- [ ] AC-7: Stress: 32K slots installed; lookup performance via binary-search (log N).
- [ ] AC-8: kselftest `tools/testing/selftests/kvm/set_memory_region_test` passes.

## Architecture

`Memslots` lives in `kernel::kvm::Memslots`:

```
struct Memslots {
  generation: u64,                    // monotonic generation counter
  slots: VarLenArray<MemorySlot>,      // sorted by gpa (binary-search lookup)
  hva_tree: RBTree<MemorySlot>,         // hva-keyed for hva→slot lookup (used by mmu-notifier)
  gfn_tree: RBTree<MemorySlot>,         // gfn-keyed for gfn→slot lookup
  node_idx: usize,                     // index of LRU-cached slot
}

struct MemorySlot {
  base_gfn: u64,
  npages: u64,
  dirty_bitmap: AtomicPtr<u64>,         // per-page dirty bit (None if KVM_MEM_LOG_DIRTY_PAGES disabled)
  arch: arch::kvm::ArchMemslot,         // per-arch per-slot data (e.g., x86 lpage_info for huge-page tracking)
  userspace_addr: u64,                  // hva backing
  flags: u32,                            // KVM_MEM_*
  id: u16,                               // slot id
  as_id: u16,                            // address-space id
  guest_memfd: Option<Arc<File>>,        // KVM_MEM_GUEST_MEMFD-only
  guest_memfd_offset: u64,
  hva_node: RBNode,
  gfn_node: RBNode,
}
```

Per-VM memslot install `Vm::set_memory_region(kvm, mem)`:
1. Validate `mem` per REQ-3.
2. Determine operation (CREATE / DELETE / MOVE / FLAGS_ONLY).
3. Take `kvm.slots_lock`.
4. Allocate new MemorySlot OR look up existing.
5. Determine per-arch prep: `arch::kvm::prepare_memory_region(kvm, old, &new, change)`:
   - x86: alloc per-slot lpage_info[] for huge-page tracking.
6. `Vm::swap_active_memslots(kvm, as_id, &new_slot, ...)`:
   - Allocate new `kvm_memslots` array (CoW from current).
   - Insert/replace/delete slot.
   - Sort by gpa (re-insert into rb-trees).
   - `rcu_assign_pointer(kvm.memslots[as_id], new_array)`.
   - Bump generation counter.
   - Schedule old array for `synchronize_srcu` + free.
7. Per-arch commit: `arch::kvm::commit_memory_region(kvm, old, new, change)`:
   - x86: `kvm_arch_flush_shadow_memslot(kvm, old)` — for DELETE/MOVE, flush all SPTEs for old gpa range (cross-ref `x86-mmu-tdp.md`).
8. Drop `kvm.slots_lock`.

Per-VM gfn lookup `Vm::gfn_to_memslot(kvm, gfn)`:
1. `srcu_read_lock(&kvm.srcu)`.
2. `slots = rcu_dereference(kvm.memslots[active_as_id])`.
3. Check LRU-cached slot first: `slots.slots[slots.node_idx]`; if gfn in [base_gfn, base_gfn+npages), return it.
4. Else: binary-search `slots.slots` for gfn → matching slot.
5. Update LRU cache.
6. `srcu_read_unlock`.

Per-slot `MemorySlot::gfn_to_hva(slot, gfn)`:
1. `offset = (gfn - slot.base_gfn) * PAGE_SIZE`.
2. Return `slot.userspace_addr + offset`.

Per-slot `MemorySlot::gfn_to_pfn(slot, gfn, atomic, async, write_fault, &writable)`:
1. If `slot.flags & KVM_MEM_GUEST_MEMFD` AND gfn is private:
   - `guest_memfd_get_pfn(slot.guest_memfd, slot.guest_memfd_offset + offset)` → host pfn.
2. Else (normal hva-backed):
   - `hva = MemorySlot::gfn_to_hva(slot, gfn)`.
   - If `atomic`: `hva_to_pfn_fast(hva)` (no fault, just check if mapped).
   - Else: `get_user_pages(current.mm, hva, 1, ...)` → pin page → return pfn.

Dirty bitmap operations:
- `KVM_GET_DIRTY_LOG`: read per-slot bitmap (snapshot copy to userspace).
- `KVM_CLEAR_DIRTY_LOG`: per-page clear of bits in the bitmap (after userspace acks the corresponding pages have been migrated).
- `KVM_GET_DIRTY_LOG_PROTECT`: atomic read + write-protect — atomically copy current bitmap to userspace + reset internal bitmap to zero + write-protect all pages so subsequent writes re-fault + re-mark dirty.

Dirty-ring (per-vCPU mode):
- Per-vCPU `dirty_ring` allocated at `KVM_CAP_DIRTY_LOG_RING_ACQ_REL` bind.
- vCPU thread on page-table-walk (e.g., from `tdp_mmu_set_spte` setting dirty bit): append GFN entry to ring's tail.
- Userspace migration thread: poll ring's head; for each entry, copy associated page; advance head.
- Acq-rel ordering between producer (tail update) + consumer (head update) ensures no torn reads.

Per-arch shadow-flush after slot delete: `kvm_arch_flush_shadow_memslot(kvm, slot)`:
- x86: walk SPTEs in [slot.base_gfn, slot.base_gfn+slot.npages) for both shadow + TDP MMUs; clear SPTE entries; flush remote TLBs.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `memslot_no_uaf` | UAF | `kvm_memslots` array RCU-freed after grace period; concurrent reader's snapshot remains valid until srcu_read_unlock. |
| `slot_overflow_no_oob` | OVERFLOW | `gpa + memory_size` checked against u64 overflow before validation. |
| `slot_id_no_oob` | OOB | slot id < KVM_USER_MEM_SLOTS (32K); per-as slot count bounded. |
| `dirty_bitmap_no_oob` | OOB | per-bitmap bit indexed by `(gfn - base_gfn)` validated against `npages`. |

### Layer 2: TLA+

`models/kvm/dirty_ring.tla` (parent-declared): proves producer-side enqueue from vCPU thread + consumer-side dequeue from userspace migration thread observe acquire-release with no torn or duplicated GFN entries.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Vm::set_memory_region` post: new slot installed iff validation passed; old slot invalidated; shadow flushed for DELETE/MOVE | `Vm::set_memory_region` |
| `Vm::gfn_to_memslot` invariant: returned slot (if Some) covers gfn (gfn in [slot.base_gfn, slot.base_gfn+slot.npages)) | `Vm::gfn_to_memslot` |
| `MemorySlot::gfn_to_hva` post: returned hva is in [slot.userspace_addr, slot.userspace_addr + slot.npages * PAGE_SIZE) | `MemorySlot::gfn_to_hva` |
| Per-slot dirty bit invariant: bit set iff TDP/shadow SPTE for this gfn has D-bit set since last reset | dirty bitmap maintenance |

### Layer 4: Verus/Creusot functional

`Vm::set_memory_region(create) → Vm::gfn_to_memslot(gfn) → MemorySlot::gfn_to_hva(slot, gfn) → MemorySlot::gfn_to_pfn(slot, gfn) → guest reads/writes pfn` round-trip equivalence: post-install, every gfn in [base_gfn, base_gfn+npages) translates to corresponding hva offset + corresponding host pfn.

## Hardening

(Inherits row-1 features from `virt/kvm/00-overview.md` § Hardening.)

memslot-specific reinforcement:

- **Memslot install validation strict** — every gpa+size+hva field validated before commit; defense against malformed userspace memslot causing kernel-side OOB.
- **No overlap with existing slot in same as_id** — defense against double-mapping causing inconsistent guest view.
- **RCU-protected memslot snapshot** — concurrent vCPU access during install never sees partial state; reader holds srcu lock during access.
- **Per-arch shadow flush mandatory on DELETE/MOVE** — defense against stale SPTE pointing at freed userspace page.
- **dirty-bitmap allocator validated against per-slot bitmap_npages** — defense against bitmap-OOB.
- **dirty-ring per-vCPU with acq-rel ordering** — Layer-2 dirty_ring.tla is the proof.
- **MAX slots per VM = 32K** in v0 (configurable); defense against per-VM slot-flood DoS.
- **guest_memfd-backed slot validation** — guest_memfd_offset + memory_size ≤ guest_memfd file size.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- KVM_SET_USER_MEMORY_REGION ioctl dispatch (covered in `kvm-core.md` Tier-3)
- Per-arch pgtable walk (covered in `x86-mmu-tdp.md` Tier-3)
- guest_memfd internals (covered in `virt/kvm/guest-memfd.md` future Tier-3)
- Dirty-tracking detail per-arch (covered in per-arch MMU Tier-3s)
- 32-bit-only paths
- Implementation code
