# Tier-3: arch/x86/kvm/mmu/mmu.c — legacy shadow paging MMU (non-TDP fallback + page-tracker + per-SP rmap)

<!--
baseline-commit: 27a26ccfd528da725a999ea1e3102503c61eb655
baseline-version: 7.1.0-rc2
status: draft
parent: virt/kvm/x86-mmu-tdp.md
upstream-paths:
  - arch/x86/kvm/mmu/mmu.c
  - arch/x86/kvm/mmu/paging_tmpl.h
  - arch/x86/kvm/mmu/spte.c
  - arch/x86/kvm/mmu/spte.h
  - arch/x86/kvm/mmu/mmu_internal.h
  - arch/x86/kvm/mmu/page_track.c
  - arch/x86/kvm/mmu/page_track.h
  - arch/x86/kvm/mmu/mmutrace.h
-->

## Summary

Legacy KVM shadow MMU is the fallback for hosts without EPT/NPT (or when guest enables CR0.WP-trap-required + paged-real-mode quirks). KVM walks guest page tables on every guest-#PF + builds shadow SPTEs that mirror guest mappings; per-shadow-page (SP) state tracks parent links, child links, and rmap (reverse-map gfn → list-of-SPTE) so single-page-invalidation can sweep all SPTEs touching that gfn. `paging_tmpl.h` provides per-paging-mode walkers (32-bit / PAE / 64-bit). `spte.c/spte.h` defines per-SPTE bit layout. Page tracker hooks notify external clients (KVMGT, nested-virt) on guest write to monitored gfn.

Critical for: pre-Nehalem hosts (no EPT) — qemu still uses shadow; nested-virt with shadow-on-shadow; KVMGT (Intel-GVT-g) page-write monitoring; Intel-PT shadow-CR3 remapping. This Tier-3 covers `mmu.c` (~8104 lines, the largest single file in KVM).

## Upstream references

| Upstream symbol | Purpose | Rookery owner |
|---|---|---|
| `struct kvm_mmu_page` | per-shadow-page (SP) | `kernel::kvm::x86::mmu::Sp` |
| `struct kvm_mmu` | per-vCPU MMU context | `Vcpu::Mmu` |
| `struct kvm_rmap_head` | per-gfn rmap chain head | `RmapHead` |
| `kvm_mmu_get_page(...)` / `_get_shadow_page` | per-gfn SP lookup-or-alloc | `Mmu::get_shadow_page` |
| `mmu_alloc_root` / `mmu_alloc_direct_roots` | per-vCPU root SP alloc | `Mmu::alloc_root` |
| `kvm_mmu_free_roots(...)` | free per-vCPU root list | `Mmu::free_roots` |
| `__direct_map(...)` / `direct_page_fault(...)` | direct-mode (TDP-like) installer | `Mmu::direct_map` |
| `FNAME(page_fault)` / `FNAME(walk_addr)` (paging_tmpl.h) | per-mode shadow-walk + #PF handler | `ShadowWalker::<Mode>::page_fault` |
| `FNAME(fetch)` | per-mode shadow-page-table install on miss | `ShadowWalker::<Mode>::fetch` |
| `FNAME(invlpg)` | per-mode INVLPG emulation | `ShadowWalker::<Mode>::invlpg` |
| `FNAME(prefetch_page)` | per-mode opportunistic-prefetch | `ShadowWalker::<Mode>::prefetch_page` |
| `kvm_mmu_pte_write(...)` | guest-write to shadowed PTE | `Mmu::pte_write` |
| `kvm_mmu_unprotect_page(...)` | clear write-protect on guest-PTE-page | `Mmu::unprotect_page` |
| `kvm_mmu_zap_all(...)` | invalidate all SPs (e.g., memslot delete) | `Mmu::zap_all` |
| `kvm_mmu_zap_oldest_mmu_pages(...)` | LRU-evict SPs when over-budget | `Mmu::zap_oldest` |
| `mmu_set_spte(...)` / `make_spte(...)` | per-SPTE construct + install | `Mmu::set_spte` / `Spte::make` |
| `gfn_to_rmap(...)` | gfn → rmap-head lookup | `Mmu::gfn_to_rmap` |
| `rmap_add(...)` / `rmap_remove(...)` | per-gfn rmap chain mgmt | `RmapHead::add` / `_remove` |
| `kvm_test_age_gfn(...)` / `kvm_unmap_gfn_range(...)` | mmu-notifier callbacks via shadow | `Mmu::test_age_gfn` / `_unmap_gfn_range` |
| `kvm_page_track_register_notifier(...)` (page_track.c) | external client subscribe | `PageTracker::register_notifier` |
| `kvm_page_track_unregister_notifier(...)` | unsubscribe | `PageTracker::unregister_notifier` |
| `kvm_page_track_write(...)` | per-write notify dispatch | `PageTracker::write` |
| `make_spte_executable(...)` | per-SPTE NX clear | `Spte::make_executable` |
| `is_shadow_present_pte` / `is_writable_pte` | per-SPTE state queries | `Spte::is_present` / `_is_writable` |

## Compatibility contract

REQ-1: Legacy shadow paging path engaged when `!tdp_enabled` (host lacks EPT/NPT) OR guest in mode requiring shadow (e.g., real-mode with paged virt, smm with specific quirks).

REQ-2: Per-vCPU MMU context (`struct kvm_mmu`):
- root_role (per-paging-mode + per-feature config of root SP).
- root[0..] (up to 4 roots for PAE 4 PDPTEs + nested cases).
- pae_root (PAE legacy 4-PDPTE buffer).
- gva_to_gpa (function pointer per-paging-mode walker).
- shadow_root_level / root_level / direct_map / lm_root.

REQ-3: Per-shadow-page (SP) struct (`struct kvm_mmu_page`):
- gfn (guest-frame mapped by this SP).
- role (sp_level + direct-bit + access + cr0_wp + cr4_smap + cr4_smep + ...).
- spt[] (512 SPTEs in the table page).
- parent_ptes (rmap-style list of SPTEs in parent SPs that point to this SP).
- root_count (refcount for root SPs).
- mmu_valid_gen (generation; bumped to invalidate all on memslot change).

REQ-4: Per-mode shadow walker (paging_tmpl.h templated for PT_GUEST_LEVEL ∈ {2,3,4}):
- `walk_addr`: per-walk-iteration: read guest PTE; check P/RW/US/A/D bits; permission-check vs CR0.WP+CR4.SMAP+CR4.SMEP+u/s; on miss: return error.
- `fetch`: per-walk after walk_addr success: traverse shadow-tree; per-level: get-or-alloc child SP; install SPTE pointing to child.
- `invlpg`: per-walk: zap leaf SPTE matching guest VA; mark SP need-sync if intermediate.

REQ-5: rmap (reverse-map) per-gfn:
- `kvm_rmap_head` per-gfn in memslot.rmap[].
- Head encodes either single-SPTE-pointer (low-bit-clear) or chain-of-SPTE-pointers (low-bit-set, full chain in PtePteList).
- Used to find all SPTEs touching gfn for: write-protect, host-page-migration, mmu-notifier-invalidate.

REQ-6: Page tracker (page_track.c):
- External client (KVMGT, nested-virt) subscribes to write-events on monitored gfn-list.
- `kvm_page_track_write(vcpu, gpa, new_data, bytes)` called from `kvm_mmu_pte_write` per-guest-write to shadowed page.
- Per-vmm: write-protect monitored gfn so shadow-fault triggers on guest write → tracker dispatches to clients.

REQ-7: SP cache: `mmu_page_hash[KVM_NUM_MMU_PAGES_PER_HASH=512]` per-VM hash by gfn (with role) for SP lookup.

REQ-8: Per-VM `arch.n_used_mmu_pages` budget: capped at `KVM_PERMILLE_MMU_PAGES * total_mem` (default ~2% of RAM); over-budget → zap LRU SPs.

REQ-9: SPTE bit layout (`spte.h`): per-SPTE encodes:
- Present bit (bit 0).
- Writable bit (bit 1).
- User bit (bit 2).
- PWT/PCD/A/D/PAT/Global (host MMU bits).
- Bit 63 = NX.
- KVM-reserved bits 8-10 (sw-only): MMU_WRITABLE / HOST_WRITABLE / SPECIAL_MASK.
- Physical addr bits 12..maxphyaddr.

REQ-10: MMU notifier hookup: `kvm_unmap_gfn_range` / `kvm_test_age_gfn` / `kvm_set_pte_gfn` walk rmap to find SPTEs to zap or update access-bit; called from host MM via mmu-notifier on host-page-migrate/swap/cow.

REQ-11: Per-vCPU shadow walk on guest #PF:
- vmexit reason = EPT-violation (TDP) OR #PF (shadow): shadow path takes #PF.
- Read guest CR3 + walk_addr to find guest PTE.
- Build/update shadow SPTE chain from root down to leaf.
- Install final leaf SPTE pointing to host PFN of guest physical addr.
- Resume guest.

## Acceptance Criteria

- [ ] AC-1: Boot Linux guest under qemu with `-cpu kvm64 -machine accel=kvm,tdp=off`: guest boots; shadow path active.
- [ ] AC-2: SMP guest stable under shadow: 4-vCPU 4GB Linux guest stress test 1h without crash.
- [ ] AC-3: Nested-virt with shadow: L1 KVM running L2 guest with shadow-on-shadow boots + survives 30s stress.
- [ ] AC-4: Per-mode walk: 32-bit non-PAE / 32-bit PAE / 64-bit Linux guests all boot under shadow.
- [ ] AC-5: KVMGT plug-in test: page-tracker callback fires on guest write to monitored gfn; observed via debugfs counter.
- [ ] AC-6: mmu-notifier integration: host SwapOut on guest-pinned page → SPTE invalidated; guest re-faults + re-pins via shadow.
- [ ] AC-7: SP-budget enforcement: 8GB guest with 4MB SP-budget triggers LRU-evict; counter monotonically increases under stress.
- [ ] AC-8: kvm-unit-tests `mmu` test passes under shadow mode.

## Architecture

`Vcpu::Mmu` is the per-vCPU MMU context:

```
struct Mmu {
  root_role: Role,
  cpu_role: Role,
  root: [Root; PT64_ROOT_MAX_LEVEL+1],     // up to 4 roots for PAE
  pae_root: KBox<[u64; 4]>,
  shadow_root_level: u8,
  root_level: u8,
  direct_map: bool,
  lm_root: Option<KBox<[u64; 512]>>,        // long-mode shim for paged-protected-mode
  gva_to_gpa: GvaToGpaFn,
  page_fault: PageFaultFn,
  walk_addr: WalkAddrFn,
  invlpg: InvlpgFn,
  ...
}

struct Sp {
  gfn: u64,
  role: Role,                                // {sp_level, direct, access, cr0_wp, cr4_smap, cr4_smep, ...}
  spt: KBox<[u64; 512]>,                     // SPTE table page
  parent_ptes: RmapHead,                     // SPTE list referencing this SP
  link: ListNode,                            // VM-wide SP list
  hash_link: HlistNode,                      // hash by gfn
  unsync: bool,                              // unsync (write-permitted; needs sync on next vmenter)
  unsync_children: u32,                      // counter for unsync descendants
  root_count: u32,                           // ref-count for root SPs
  mmu_valid_gen: u64,                        // gen-counter; bumped to invalidate all
}

struct RmapHead(u64);                        // single-spte (clear-bit) or chain-head (set-bit) encoding
```

`Mmu::page_fault` (shadow path on guest #PF):
1. Read guest CR3; build root if needed (alloc_root).
2. `walk_addr(vcpu, gva, error_code)`:
   - Iterate from level=root_level down to 1:
     - Read guest PTE at offset.
     - Validate present + permissions (US/RW vs error_code).
     - Track A/D bits to set.
   - Return walked-PTE info or fault.
3. `fetch(vcpu, gva, walked, write_fault)`:
   - Iterate root → leaf:
     - At each level: lookup-or-alloc child SP via `get_shadow_page(gfn, role)`.
     - Construct SPTE pointing to child SP page.
   - At leaf: build SPTE pointing to host PFN of guest PA.
   - `set_spte(vcpu, sp, sptep, gfn, pfn, level, ...)`:
     - Allocate rmap entry; add to per-gfn rmap chain.
     - Install SPTE atomically (cmpxchg).
4. Resume guest.

`Mmu::pte_write` (guest writes shadowed PTE):
1. Find SP containing the written gfn (via mmu_page_hash).
2. If SP found: per-byte parse to find affected SPTE indexes.
3. Per-affected SPTE: invalidate (zap_one_rmap_spte) + queue invlpg for guest.
4. Notify page-tracker: `kvm_page_track_write(vcpu, gpa, new_data, bytes)`.

`Mmu::zap_all` (memslot delete or VM destroy):
1. Bump `arch.mmu_valid_gen` (next vmenter sees stale-gen check + flushes).
2. List-walk SP-list per VM; per-SP free spt page + free SP struct.
3. Per-vCPU root_count clears.

`PageTracker::register_notifier(vm, slot, gfn_range, ops)`:
1. Per-gfn-in-range: increment `kvm_memory_slot.arch.gfn_track[mode][gfn]` counter.
2. If counter goes 0→1: write-protect SPTE for that gfn (force shadow-fault on write).
3. Add notifier to per-vm callback list.

## Verification

### Layer 1: Kani SAFETY

| Property | Kind | Statement |
|---|---|---|
| `sp_spt_no_oob` | OOB | per-SP.spt[i] indexed by SPTE-offset bounded by 512. |
| `rmap_chain_no_uaf` | UAF | per-RmapHead chain entries managed via PteListDesc; defense against per-spte-zap-during-rmap-walk. |
| `walk_addr_bounded` | OOB | per-walk iteration count bounded by walk_root_level (≤ 4). |
| `pae_root_4_pdpte` | INVARIANT | per-vCPU pae_root buffer always exactly 4 PDPTE slots. |
| `mmu_valid_gen_monotonic` | INVARIANT | per-VM mmu_valid_gen monotonically increasing; rollover safe at u64. |

### Layer 2: TLA+

`virt/kvm/mmu_shadow_walk.tla` models per-vCPU shadow-walk: per-iteration state ∈ {WalkAddr, Fetch, SpAlloc, SpteInstall, Done, Fault}; transitions per spec.

Properties:
- `safety_no_install_on_fault` — once Fault, no SpteInstall.
- `safety_per_level_progress` — Fetch advances level monotonically toward leaf.
- `liveness_walk_terminates` — every WalkAddr eventually reaches Done or Fault.

`virt/kvm/spte_lifecycle.tla` (already exists from earlier Tier-3) models per-SPTE state: NotPresent → Present(ro) → Present(rw) → NotPresent transitions; safety: no Present-rw if write-protect bit set.

### Layer 3: Kani + Verus invariants

| Invariant | Component |
|---|---|
| `Mmu::get_shadow_page` post: returned SP has `gfn == requested && role == requested && cached in hash` | `Mmu::get_shadow_page` |
| `Mmu::page_fault` post: leaf SPTE installed iff walk_addr succeeded and host pfn obtained | `Mmu::page_fault` |
| `Mmu::pte_write` post: all affected SPTEs invalidated + page-tracker notified | `Mmu::pte_write` |
| Per-VM SP count ≤ `n_max_mmu_pages` budget | `Mmu::zap_oldest` |
| Per-RmapHead chain consistent: every entry points to a present SPTE referencing this gfn | `RmapHead::add` / `_remove` |

### Layer 4: Verus/Creusot functional

`Mmu::page_fault(gva, error_code) → guest re-execute @ gva` semantic equivalence: post-fault execution of guest instruction at gva succeeds iff guest-PTE walk succeeds + host-pfn available + permissions match error_code.

## Hardening

(Inherits row-1 features from `virt/kvm/x86-mmu-tdp.md` § Hardening.)

shadow-MMU-specific reinforcement:

- **Per-VM SP-budget cap** — defense against guest-induced SP-overgrowth DoS.
- **Per-walk iteration count cap** (≤ root_level=4) — defense against guest constructing PTE-cycle.
- **Per-SPTE rmap-chain-len cap** — defense against pathological guest gfn aliasing.
- **Per-SP root_count refcount** — defense against root-SP-free-while-active causing UAF.
- **Per-SP unsync state machine** — unsync ↔ sync transitions atomic; defense against torn-spte-update on write.
- **mmu_valid_gen check on vmenter** — detect stale per-vCPU root after memslot change → force re-init.
- **Page-tracker write-protect ref-count** — per-gfn counter; SPTE write-protected iff any subscriber; defense against drop-then-resub races.
- **Per-vCPU root[i] init under mmu_lock** — defense against concurrent root-alloc with vCPU-thread-vmenter.
- **MMU notifier integration** — host MM operations (swap, migrate, COW) invalidate SPTEs via rmap; defense against host-page-reuse-while-mapped.
- **PAE root quirk validated** — exactly 4 PDPTE slots; defense against PAE walker reading out-of-range PDPTE index.

## Open Questions

(none at this Tier-3 level)

## Out of Scope

- TDP MMU (covered in `x86-mmu-tdp.md` Tier-3)
- KVM core (covered in `kvm-core.md` Tier-3)
- Per-vCPU memslots (covered in `memslots.md` Tier-3)
- LAPIC (covered in `x86-lapic.md` Tier-3)
- Nested-virt VMCS-shadowing (covered in `nested-vmx.md` future Tier-3)
- KVMGT specifics (covered in `gpu/i915-gvt.md` future Tier-3)
- Implementation code
